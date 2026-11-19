# Java 线程池底层原理与大厂面试题全解

> 一条线索贯穿到底：**一个 `Runnable` 从 `execute()` 进去，到 `pthread_create` 出来，中间经历了 JVM、libc、内核三层。**
> 本文按「操作系统 → JVM → 线程池源码 → 场景实践 → 面试题」自底向上组织，每一层都回答同一个问题：这一层到底在干什么、为什么必须这样干。

---

## 目录

- [第 0 章 全景地图](#第-0-章-全景地图)
- [第 1 章 为什么需要线程池：从"创建一个线程"讲起](#第-1-章-为什么需要线程池从创建一个线程讲起)
- [第 2 章 操作系统层：Java 线程在内核里到底是个什么东西](#第-2-章-操作系统层java-线程在内核里到底是个什么东西)
- [第 3 章 JVM 层：从 `Thread.start()` 到 `pthread_create`](#第-3-章-jvm-层从-threadstart-到-pthread_create)
- [第 4 章 ThreadPoolExecutor 的骨架：ctl、状态机、Worker](#第-4-章-threadpoolexecutor-的骨架ctl状态机worker)
- [第 5 章 执行流程逐行走读：execute → addWorker → runWorker → getTask → exit](#第-5-章-执行流程逐行走读)
- [第 6 章 队列与拒绝策略：线程池的"缓冲"与"泄压阀"](#第-6-章-队列与拒绝策略线程池的缓冲与泄压阀)
- [第 7 章 容量规划：线程数到底该设多少](#第-7-章-容量规划线程数到底该设多少)
- [第 8 章 生命周期与关闭：shutdown / shutdownNow / 优雅停机](#第-8-章-生命周期与关闭)
- [第 9 章 生产实践：线程池的 12 个真实坑](#第-9-章-生产实践线程池的-12-个真实坑)
- [第 10 章 国内大厂面试题全解（45 题）](#第-10-章-国内大厂面试题全解45-题)
- [附录 A 一页速查表](#附录-a-一页速查表)
- [附录 B 自测清单](#附录-b-自测清单)

---

## 第 0 章 全景地图

先看一张"上帝视角"的图。**你在代码里写的 `pool.execute(task)`，其实压在下面这五层之上。**

```
┌──────────────────────────────────────────────────────────────────────┐
│ 业务层     pool.execute(task) / submit(task) / invokeAll(tasks)      │
│            ↑ 你自己写的 Runnable / Callable                          │
├──────────────────────────────────────────────────────────────────────┤
│ 线程池层   ThreadPoolExecutor                                       │
│            ├─ ctl (AtomicInteger)  ← 状态 + 线程数的原子合体          │
│            ├─ BlockingQueue       ← 任务的"蓄水池"                    │
│            ├─ Worker (extends AQS)← 线程的"包装壳"，一壳一线程         │
│            └─ RejectedExecutionHandler ← 满员后的"泄压阀"             │
├──────────────────────────────────────────────────────────────────────┤
│ JVM / HotSpot 层                                                    │
│            java.lang.Thread → JavaThread(VM) → OSThread(VM)         │
│            栈由 JVM mmap 出来，带 guard page；栈帧在栈上压/弹          │
├──────────────────────────────────────────────────────────────────────┤
│ libc 层    pthread_create / pthread_cond_wait / pthread_mutex_lock   │
│            （NPTL：1:1 模型，一个 pthread = 一个内核线程）             │
├──────────────────────────────────────────────────────────────────────┤
│ 内核层     clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_THREAD|...)      │
│            → 生成一个 task_struct，与父进程共享 mm_struct             │
│            futex(FUTEX_WAIT/FUTEX_WAKE) ← 所有 Java 阻塞的原点        │
│            CFS 调度器：vruntime + 红黑树，决定谁上 CPU                 │
├──────────────────────────────────────────────────────────────────────┤
│ 硬件层     CPU 核、寄存器、TLB、Cache、NUMA                          │
└──────────────────────────────────────────────────────────────────────┘
```

**这张图要记住三个结论，后面所有内容都是这三条的展开：**

| 结论 | 含义 | 对应面试高频点 |
|---|---|---|
| ① Java 线程 ≠ 用户态协程 | 每个 Java 线程都是真实内核线程，创建要走系统调用，调度由内核决定 | "Java 线程模型是 1:1，不是 M:N" |
| ② 线程池的本质是**复用内核线程** | 省掉 `clone()` + 栈分配 + 销毁，而不是"省掉 new 对象" | "线程池为什么快" |
| ③ 线程池同时是**并发度控制器** | 它限制的是"同时在跑的线程数"，从而保护下游资源 | "为什么要自定义线程池" |

---

## 第 1 章 为什么需要线程池：从"创建一个线程"讲起

### 1.1 一个线程的"开箱成本"清单

很多人背过"线程创建销毁开销大"，但说不清大在哪。把它拆成 6 笔账：

```
new Thread(r).start()
   │
   ├─① Java 对象分配        Thread 对象 + ThreadGroup + 栈上的 ThreadLocalMap 引用
   │                       ≈ 几百字节，Eden 区分配，可忽略（TLAB 指针碰撞）
   │
   ├─② JVM 内部结构         JavaThread + OSThread 两个 C++ 对象，含大量字段
   │                       ≈ 1~2 KB，且要注册进全局线程链表（有锁）
   │
   ├─③ 栈内存 mmap          默认 -Xss 1MB（x64），内核建 VMA，用户态建页表项（惰性）
   │                       物理内存按需缺页分配，但 VMA + 守卫页是立刻的
   │
   ├─④ 系统调用 clone()    陷入内核，拷贝 task_struct（含 cgroup、sched_entity、
   │                       signal、files_struct 引用计数等），≈ 2~4 KB 内核对象
   │                       单次 ≈ 10~30 μs（含调度器注册）
   │
   ├─⑤ 首次被调度           新线程要等 CFS 给它一个时间片，首次上 CPU 有排队延迟
   │
   └─⑥ 退出与回收           exit() → 释放 task_struct、栈、VMA；等待父线程 join
                           或 JDK 的线程回收机制收尸（zombie 清理）
```

> **量化参考**（x64 Linux，非严谨 benchmark，只用于建立量感）：
> - 创建一个线程并让它跑一个空任务然后结束：**约 30~80 μs**
> - 一次上下文切换（同核）：**约 1~3 μs**，若跨核 + Cache 冷启动，实际损失可达 **5~10 μs 等效算力**
> - 两者相差一个数量级——**这就是"复用"比"重造"划算的根本原因**

### 1.2 三种"省钱"思路，线程池选了哪一种

```
问题：一次请求要执行一段代码，怎么让它和 CPU 核挂上钩？

方案 A：来一个请求 → 创建一个线程 → 干完销毁
        成本 O(创建) 每次，10000 QPS = 每秒 1 万次 clone
        ✗ 系统调用风暴、内存碎片、调度器压力

方案 B：预先创建固定数量线程 → 循环取任务
        成本 O(创建) 一次性，且数量可控
        ✓ 这就是线程池

方案 C：一个线程 + 事件循环（Reactor / Netty）
        成本 O(1)，但阻塞操作会拖死整个循环
        △ 适合 IO 多路复用，不适合有阻塞调用的业务逻辑
```

**线程池 = 方案 B，它的两个身份必须同时记住：**

1. **资源池（复用器）**：线程建一次，反复用，摊销创建/销毁成本。
2. **并发阀门（限流器）**：`maximumPoolSize` 就是"同时最多多少个线程能碰下游系统"。它顺手解决了"保护数据库连接数、保护下游 RPC"的问题——这也是它比"复用"更重要的价值。

> 💡 **延伸到虚拟线程**：JDK 21 的虚拟线程（Loom）走的是方案 C 的思路——M:N 调度，阻塞时自动 `unmount` 让出载体线程（carrier thread）。所以官方文档明说：**IO 密集型任务优先用虚拟线程，不要再用大线程池**。但 CPU 密集型、以及需要限流的场景，`ThreadPoolExecutor` 依然不可替代。这一点在面试里是加分项，见第 10 章 Q35。

### 1.3 线程池省下的到底是什么（一句话版本）

> 线程池省下的不是"对象分配"，而是 **`clone()` 系统调用 + 内核对象构造 + 栈 VMA 建立 + 调度器注册/注销** 这一整条内核路径，以及由此带来的**调度器压力与内存抖动**。

---
## 第 2 章 操作系统层：Java 线程在内核里到底是个什么东西

### 2.1 Linux 根本没有"线程"这个概念

这是理解一切的地基。Linux 内核源码里**只有 `task_struct`，没有 thread_struct 这一层"线程对象"**（`thread_struct` 只是 task_struct 里的一个字段，存寄存器上下文）。

```
传统认知（教科书）               Linux 的实际实现
─────────────────────           ─────────────────────────────────
进程 = 资源分配单位    ────────→  task_struct（也叫"任务"）
线程 = 调度执行单位    ────────→  另一个 task_struct
两者是父子/包含关系               两者是"兄弟"，差别只在共享了哪些资源
```

**一个 task_struct 由两部分资源构成：**

| 资源 | 结构体 | 进程 | 线程 |
|---|---|---|---|
| 地址空间（代码/数据/堆/栈映射） | `mm_struct` | 独占 | **共享**（`CLONE_VM`） |
| 打开的文件描述符表 | `files_struct` | 独占 | **共享**（`CLONE_FILES`） |
| 文件系统信息（cwd、umask） | `fs_struct` | 独占 | **共享**（`CLONE_FS`） |
| 信号处理函数表 | `sighand_struct` | 独占 | **共享**（`CLONE_SIGHAND`） |
| 待处理信号队列 | `signal_struct` | 独占 | **共享**（`CLONE_THREAD`） |
| 页表基址 `CR3` | — | 各自一份 | **同一份** |
| 内核栈 | 每 task 一份 | 每 task 一份 | 每 task 一份 |
| 寄存器上下文 | `thread_struct` | 每 task 一份 | 每 task 一份 |
| `sched_entity`（调度实体） | — | 每 task 一份 | 每 task 一份 |

> **关键洞察**：因为"线程"共享 `mm_struct`，所以**线程间切换不需要换页表**（不用写 CR3），这是线程切换比进程切换便宜一半的根本原因。而因为每个 task 都有独立的 `sched_entity`，所以**内核可以对每个线程独立调度**——这就是"线程是调度单位"的物理含义。

### 2.2 `clone()` 系统调用：一个函数造出进程和线程

`pthread_create()` 在 glibc 里的最终归宿是 `clone()`。区别进程还是线程，全靠 flag：

```c
/* 创建一个"线程"（其实是共享地址空间的新 task） */
clone(thread_fn,          /* 子任务入口函数 */
      stack_top,          /* 子任务自己的栈顶（用户态栈，必须由调用者提供） */
      CLONE_VM          | /* 共享地址空间 → 同一个 mm_struct，refcount++ */
      CLONE_FS          | /* 共享 cwd/umask */
      CLONE_FILES       | /* 共享 fd 表 */
      CLONE_SIGHAND     | /* 共享信号处理函数 */
      CLONE_THREAD      | /* 归入同一个线程组（同一个 tgid = 进程号） */
      CLONE_SYSVSEM     | /* 共享 System V 信号量 undo */
      CLONE_SETTLS      | /* 设置 TLS（FS 段基址），Java 的 ThreadLocal 靠它 */
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID, /* 线程 tid 与退出通知 */
      arg);
```

**`CLONE_THREAD` 是"看起来像线程"的关键**：它让新 task 与父 task 共享 `tgid`（thread group id）。`ps` 看到的 PID 其实是 tgid，而内核调度看到的 PID 是每个 task 自己的 `pid`。所以：

```
$ ps -eLf | grep java
java  10086  10086  10087  ...     ← PID=10086(tgid)  LWP=10087(内核调度 id)
       ↑      ↑      ↑
       tgid   ppid   lwp(tid)

$ top -H -p 10086       ← -H 就是按 LWP 展示，能看出哪个线程吃 CPU
```

> 🔧 **生产技能点**：排查"CPU 100%"，`top -H` 拿到占用最高的 LWP（十进制）→ 转十六进制 → 去 `jstack` 里搜 `nid=0x...`，才能定位到 Java 线程栈。这是面试"线上 CPU 飙高怎么排查"的标准答案第一步，而这第一步的底层原理就是 **LWP ≠ PID**。

### 2.3 Java 线程模型：1:1（而不是 M:N）

```
        Java 层                JVM 层                内核层
   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
   │ Thread obj 1 │─────→│ JavaThread 1 │─────→│ task_struct 1│ → CPU0
   ├──────────────┤      ├──────────────┤      ├──────────────┤
   │ Thread obj 2 │─────→│ JavaThread 2 │─────→│ task_struct 2│ → CPU1
   ├──────────────┤      ├──────────────┤      ├──────────────┤
   │ Thread obj 3 │─────→│ JavaThread 3 │─────→│ task_struct 3│ → CPU2
   └──────────────┘      └──────────────┘      └──────────────┘
        1 : 1 : 1        （JDK 21 之前的"平台线程"模型）
```

**1:1 模型的三条硬约束（也是面试常问的"Java 线程的代价"）：**

1. **线程数上限 = 内核能承受的 task 数上限**。你在 Java 里 `new Thread()` 一万次，内核里就真有一万个 task_struct 在争抢 CPU，不是"轻量对象"。
2. **阻塞 = 内核态睡眠 = 占着一个内核线程名额**。你在 Java 里等一次网络 IO 5 秒，那个内核线程就在 `S` 状态躺 5 秒，什么也干不了——但它仍然占着 1MB 栈 + 内核对象。
3. **调度完全由 CFS 决定**，Java 的 `setPriority()` 基本是安慰剂（见 2.7）。

> 对比：Go 的 goroutine、Java 21 的虚拟线程是 **M:N**，N 个用户态协程映射到 M 个内核线程，阻塞时把协程"搬下"载体线程。这就是为什么它们能开百万个。

### 2.4 线程的栈：三层结构，缺一不可

```
高地址
┌────────────────────────────────────────────┐
│  用户态栈（Java 栈 / -Xss，默认 1MB）        │  ← HotSpot 自己 mmap，不是 glibc 给的
│  ├─ 栈帧(frame)：局部变量表 + 操作数栈 + ... │
│  ├─ ...                                     │
│  └─ 栈顶（当前帧）                          │
├────────────────────────────────────────────┤
│  Guard Page（守卫页，PROT_NONE）            │  ← 越界访问触发 SIGSEGV
│  ↓ 越过它 → JVM 捕获并抛 StackOverflowError │
├────────────────────────────────────────────┤
│  内核栈（每 task 一份，x64 默认 16KB）       │  ← 系统调用/中断时压这里
└────────────────────────────────────────────┘
低地址
```

**三个必须知道的点：**

- **JVM 的栈是自己 `mmap` 的**，不走 glibc 的 `pthread_attr_setstacksize`。所以 `-Xss` 调大不会影响 pthread 默认栈，Java 栈溢出可控性好。默认值：x64 Linux/Windows 均约 1MB（64 位默认 1M，部分版本 512K~1M，以 `-XX:+PrintFlagsFinal | grep ThreadStackSize` 为准）。
- **栈是"虚拟内存一次给足，物理内存按需缺页"**。开 1000 个线程，虚拟地址吃掉 1GB，但物理内存可能只用了 100MB——**所以"线程数受内存限制"这件事，受限的是 VMA 数量而不是物理内存**（见 2.9 的 `max_map_count`）。
- **内核栈不可换出到用户态**。深递归 + 系统调用嵌套，可能先耗尽内核栈（这就是所谓 kernel stack overflow，会 panic 而不是抛 Java 异常）。

**栈帧结构（`-Xss` 溢出为什么发生在"方法调用太深"）：**

```
每个 Java 方法调用 = 一个栈帧压栈，帧大小 ≈ 
    局部变量表(4/8 字节 × slot 数) 
  + 操作数栈(编译期算出的最大深度) 
  + 动态链接/返回地址/锁记录等固定开销

栈深度上限 ≈ (-Xss) / (平均帧大小)
  -Xss=1M, 帧=100B  → 约 1 万层
  -Xss=1M, 帧=1KB   → 约 1000 层
```

### 2.5 上下文切换：到底切了什么，贵在哪

**切换一次线程，内核做了 7 件事：**

```
线程 A 正在 CPU 上跑
   │
   │ ① 触发切换：时间片用完（CFS tick）/ 主动让出（futex 睡眠）/ 被抢占（更高优先级）
   ▼
② 保存 A 的"现场"到 A 的内核栈：
     - 通用寄存器（callee-saved：rbx, rbp, r12-r15）
     - 栈指针 rsp、指令指针 rip
     - FPU/SIMD 状态（xsave，AVX 有 256/512 位，这块越来越贵）
     - TLS 基址（FS 段寄存器）
   ▼
③ 更新 A 的 task_struct：state = TASK_INTERRUPTIBLE，记账 vruntime
   ▼
④ CFS 从红黑树取最左节点 B（vruntime 最小者）
   ▼
⑤ 切 rsp 到 B 的内核栈，从 B 的内核栈恢复"现场"
   （线程切换不写 CR3，因为 mm_struct 相同 → 省掉 TLB flush）
   ▼
⑥ 切换完成，B 从上次被切走的地方继续
   ▼
⑦ [隐形成本] 分支预测器历史被污染、L1/L2 Cache 里是 A 的热数据、
   TLB 里是 A 的页表项、内存预取队列作废
```

**成本量化与两条结论：**

| 切换类型 | 直接开销 | 真实影响（含 Cache 冷启动） |
|---|---|---|
| 同进程线程切换 | 1~3 μs | 有效算力损失 5~10 μs 等效 |
| 跨进程切换 | 3~10 μs（多一次 CR3 写 + TLB flush，PCID 可缓解） | 更差 |
| 同核切换 | 相对便宜（Cache 部分复用） | — |
| 跨 NUMA 节点切换 | 最贵（远端内存访问延迟 ×2~3） | — |

> **所以：线程数不是越多越好。** 当可运行线程数 >> 核数时，大量 CPU 时间被"切换"吃掉（这叫 thrashing），吞吐反而下降。CFS 的 `sched_latency`（默认 6ms）会在任务很多时把时间片压到最小粒度 `min_granularity`（默认 0.75ms），**低于这个粒度就不再压缩了，多出来的线程只能排队等更久**——这就是"线程开太多，每个都变得很慢"的内核级解释。

### 2.6 CFS：内核怎么决定"下一个谁跑"

```
每个 task 都有一个 sched_entity：
   vruntime = 实际运行时间 × (NICE_0_LOAD / weight)
              ↑ nice 值越高 → weight 越小 → vruntime 涨得越快 → 越早被换下

调度器维护一棵红黑树（cfs_rq->tasks_timeline），按 vruntime 排序：

            [vruntime=100]
           /              \
     [80]                  [150]
     /  \                  /   \
  [72]  [90]          [130]   [200]
   ↑
  最左节点 = 最小 vruntime = 最该被运行

每次 pick_next_task_fair() 就取最左节点 → O(log n)，且缓存了 leftmost 指针 → 实际 O(1)
```

**三个调度参数（面试里说得出名字就加分）：**

| 参数 | 默认值 | 含义 |
|---|---|---|
| `sched_latency_ns` | 6 ms | 一个"调度周期"的目标时长，所有可运行任务在这段时间内至少各跑一次 |
| `min_granularity_ns` | 0.75 ms | 单个任务的最小时间片。**任务多到"6ms / N < 0.75ms"时，不再压缩，改为排队轮转** |
| `sched_min_vruntime` | — | 单调递增的"虚拟时钟下界"，防止睡眠久的线程醒来后靠极小 vruntime 霸占 CPU |

> **实操含义**：4 核机器，若有 64 个可运行线程，每个时间片 = 6ms 上限 / 64 ≈ 0.09ms < 0.75ms，于是实际按 0.75ms 轮转，一个线程要等 64 × 0.75ms ≈ 48ms 才轮到一次。**这就是"线程池开太大，任务延迟反而暴涨"的数学原因**，比任何口头经验都硬。

### 2.7 优先级、亲和性：Java 里哪些是有效的

| 手段 | Java API | 内核是否真的听 | 说明 |
|---|---|---|---|
| 线程优先级 | `setPriority(1..10)` | **基本无效** | HotSpot 在 Linux 上把 1~10 映射为 nice 值，但 nice 只影响权重比例，且映射粗糙；只有在极低负载下才有可感知差异。**结论：不要用优先级做业务隔离** |
| 守护线程 | `setDaemon(true)` | 有效（JVM 语义） | 守护线程不阻止 JVM 退出，非守护线程会 |
| CPU 亲和性 | 无标准 API | 有效但需 native | 要用 `taskset` 或 JNA 调 `sched_setaffinity`，Java 层拿不到 |
| 实时调度 | 无 | 需 root + `chrt` | 金融低延迟场景会做，但要小心饿死其他进程 |

> 面试陷阱题：**"用 `Thread.setPriority` 能保证高优先级线程先执行吗？"** 标准答：**不能**。跨平台语义不一致（Windows 有一定效果，Linux 几乎无效），且 CFS 的设计哲学是公平而非优先。要"优先"应该换队列（`PriorityBlockingQueue`）而不是换优先级。

### 2.8 futex：所有 Java 阻塞的原点

这是"线程池里线程怎么等待任务"的最终答案。Java 里**所有**让线程"停下来"的操作，最终都落到 futex：

```
Object.wait()      ┐
Thread.sleep()     │
Thread.join()      ├──→ JVM/liblc ──→ futex(FUTEX_WAIT) / nanosleep ──→ 内核 S 状态
LockSupport.park() │
synchronized 竞争  ┘   (轻量级锁自旋失败、膨胀为重量级锁后)
BlockingQueue.take() ──→ Condition.await() ──→ LockSupport.park() ──→ futex
```

**futex 的精妙在于"无竞争时零系统调用"：**

```
用户态共享一个 32 位整数（futex word）

加锁快路径（无竞争）：
   CAS 0 → 1 成功  → 结束！全程在用户态，不进内核，约 20ns

加锁慢路径（有竞争）：
   CAS 失败 → 调 futex(FUTEX_WAIT, addr, expected=1)
             内核：if (*(u32*)addr != expected) return EAGAIN;  ← 二次确认，防丢唤醒
                   否则把自己挂到 addr 对应的 wait queue，state = S，让出 CPU

解锁：
   写 0 → 若发现有等待者 → futex(FUTEX_WAKE, addr, 1) → 唤醒一个
```

**这套机制解释了三个"为什么"：**

1. **为什么 `synchronized` 在无竞争时和 `ReentrantLock` 差不多快**：两者在快路径都不进内核。
2. **为什么"线程池里线程空闲时不消耗 CPU"**：它睡在 futex wait queue 上，CPU 上根本没有它。1000 个空闲线程 ≈ 0 CPU。
3. **为什么"唤醒"有延迟**：从 `FUTEX_WAKE` 到目标线程真的跑起来，要走"从 wait queue 摘下 → 塞进 CFS 红黑树 → 等调度 → 上 CPU"，**典型 5~50 μs**（负载高时更久）。这解释了 RPC 框架为什么热衷"异步 + 批量"，因为每次阻塞唤醒都在付这个税。

### 2.9 硬上限：一个进程最多能开多少线程

这是"线程池 maximumPoolSize 能不能设 10000"的物理答案。**六个天花板，先撞到哪个算哪个：**

| 限制项 | 查看方式 | 典型默认值 | 撞上时的报错 |
|---|---|---|---|
| **物理内存** | `free -h` | — | `OutOfMemoryError: unable to create native thread` |
| **`vm.max_map_count`** | `sysctl vm.max_map_count` | 65530 | 同上。**每个线程的栈区通常占 1~2 个 VMA（栈本体 + guard page）**，因此上限意味着"数万线程量级"就会撞墙——这是最容易被忽略的那个 |
| **`kernel.pid_max`** | `sysctl kernel.pid_max` | 32768（老）/ 4194304（新） | `fork: Cannot allocate memory` |
| **`kernel.threads-max`** | `sysctl kernel.threads-max` | ≈ 物理页数/8 | 同上 |
| **cgroup `pids.max`** | 容器里 `/sys/fs/cgroup/pids.max` | 常被 k8s 设为 1000~4000 | **K8s 里最常见的坑**：Pod 里线程数超了，整个容器被 OOM/限制 |
| **`ulimit -u`** | `ulimit -u` | 由 `/etc/security/limits.conf` 定 | `Resource temporarily unavailable` |

> 🔧 **一条能落地的经验值**：单 JVM 实例的**活跃线程总数控制在 500 以内**（含框架线程、GC 线程、定时任务线程）。这不是玄学，而是由 `500 × 2 VMA × ...` 加上 1MB 栈虚拟地址叠加出来的安全区。**线程池的 `maximumPoolSize` 千万不要设成 `Integer.MAX_VALUE`**（`Executors.newCachedThreadPool` 就是这么干的）。

### 2.10 Java 线程状态 ↔ 内核状态：一张必背的映射表

这张表是"jstack 看不出问题"系列困惑的解药。

| Java `Thread.State` | 触发场景 | 内核 `task_struct.state` | `jstack` 显示 | 是否吃 CPU |
|---|---|---|---|---|
| `NEW` | `new Thread()` 未 `start()` | **不存在**（还没 clone） | 不出现 | — |
| `RUNNABLE` | 正在跑 / 在 runqueue 排队 / **阻塞在 socket read** | `TASK_RUNNING`(R) 或 `S` | `java.lang.Thread.State: RUNNABLE` | **可能 0%，也可能 100%** |
| `BLOCKED` | 等 `synchronized` 的 monitor 锁 | `S`（futex wait） | `- waiting to lock <0x...>` | 0% |
| `WAITING` | `wait()`/`join()`/`park()` | `S` | `- waiting on condition` / `parking to wait for` | 0% |
| `TIMED_WAITING` | `sleep(t)`/`wait(t)`/`parkNanos` | `S` + 定时器 | `- sleeping` / `parking to wait for` | 0% |
| `TERMINATED` | run 结束 | task 已回收（`TASK_DEAD`） | 不出现 | — |

> ⚠️ **最重要的认知纠正**：**Java 的 `RUNNABLE` 包含了"OS 层的休眠"**。HotSpot 只要线程没在等锁、没在 `park`、没在 `sleep`，就一律标 `RUNNABLE`——哪怕它正阻塞在 `read()` 上等网络数据。
>
> 所以：
> - 看到大量 `RUNNABLE` 就断定"CPU 高" → **错**。要看 `top -H` 谁在真耗 CPU。
> - 排查"线程卡住"时，`RUNNABLE` 且 `top` 里 CPU 为 0 的线程，八成在等 IO（`strace` 能看到 `read(fd,...)` 挂住）。

---
## 第 3 章 JVM 层：从 `Thread.start()` 到 `pthread_create`

### 3.1 调用链：五个跳转，三次跨界

```
java.lang.Thread.start()                     [Java 层]
   │  if (started) throw IllegalThreadStateException();   ← 只能 start 一次
   │  start0();
   ▼
private native void start0();                [JNI 边界：用户态 → JVM C++]
   ▼
JVM_StartThread()                            [HotSpot: jvm.cpp]
   │  ① 检查 java_lang_Thread 的 status 必须仍是 NEW
   │  ② new JavaThread(&thread_entry, sz)   ← 分配 C++ 侧线程对象，栈大小 = -Xss
   │  ③ 把 java_lang_Thread 对象塞进 JavaThread（eetop 字段做双向绑定）
   │  ④ Thread::start(native_thread)
   ▼
JavaThread::run() { ... }                    [HotSpot: thread.cpp]
   │  os::create_thread(this, thr_type, stack_sz)   ← OS 抽象层，跨平台
   ▼
os::create_thread()                          [Linux: os_linux.cpp]
   │  pthread_attr_init / pthread_attr_setstacksize(1M)
   │  pthread_attr_setdetachstate(PTHREAD_CREATE_DETACHED)
   │  pthread_create(&tid, &attr, thread_native_entry, this)  ← ★ 真正的系统调用在这
   │  等待子线程把 _state 置为 INITIALIZED（信号量同步，防止 start 返回时线程还没准备好）
   ▼
thread_native_entry(void* thread)            [内核已经跑起来了]
   │  设置 TLS、设置 Java 线程状态、Thread::call_run()
   ▼
thread_entry() → JavaCalls::call_virtual("run")  ← 反过来调回 Java 的 Thread.run()
```

**这张调用链的意义**：`start0()` 是"用户态 → 内核态 → 用户态"的一次完整往返。**`pthread_create` 返回时，新线程可能已经在另一个 CPU 核上跑了一段**，所以 `start()` 返回不代表 `run()` 已执行完，甚至不代表已经开始执行。

### 3.2 三个"线程对象"的三角关系（必背）

```
   Java 堆                                C++ 堆（元空间/原生内存）
┌────────────────────┐      eetop      ┌──────────────────────────┐
│ java.lang.Thread   │ ──────────────→ │ JavaThread               │
│  ├ name            │   (指针字段)     │  ├ _threadObj ←──────────┼─┐
│  ├ priority        │                 │  ├ _osthread             │ │
│  ├ threadStatus    │ ←───────────────│  ├ _stack_base / _stack_size│
│  ├ threadLocals    │   (状态回写)     │  ├ _pending_interrupt    │ │
│  └ runnable        │                 │  └ _anchor (栈顶帧)      │ │
└────────────────────┘                 └──────────┬───────────────┘ │
                                                  │                  │
                                                  ▼                  │
                                       ┌──────────────────────────┐ │
                                       │ OSThread                 │ │
                                       │  ├ pthread_t             │ │
                                       │  ├ 内核线程 id           │ │
                                       │  └ 线程栈/状态           │ │
                                       └──────────────────────────┘ │
                                                                     │
   三者互指（Java↔JavaThread↔OSThread），任一环节断裂都会造成            │
   "看起来线程还在、其实已经废了" 的诡异问题 ──────────────────────────┘
```

**面试里这个三角关系的用处**：当出现 `OutOfMemoryError: unable to create native thread` 时，说明**Java 对象创建成功了、但 `pthread_create` 失败**——即"第 3 步成功了，第 5 步挂了"。所以这类 OOM 不能靠调大堆解决，必须减线程数或调内核参数（见 2.9）。

### 3.3 栈映射与守卫页：为什么栈溢出是 `StackOverflowError` 而不是崩溃

```
HotSpot 在 Linux 上创建线程栈时（JavaThread::create_stack_guard_pages）：

  mmap(addr, stack_size, RW, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK)
  └─ 在栈底再 mmap 一页 PROT_NONE 的 Guard Page（黄色区/红色区）

  Java 栈向下生长，溢出时访问到 Guard Page
     ↓
  触发 SIGSEGV
     ↓
  JVM 信号处理器判断"是不是踩了 guard page"
     ↓
  是 → 转成 java.lang.StackOverflowError 抛给 Java 层（线程可继续存活）
  否 → 说明是野指针，直接崩溃（hs_err_pid.log）
```

> **延伸**：`-Xss` 调太小会 `StackOverflowError`，调太大会让"单线程内存占用"上升，且**未必是好事**（一个泄漏的线程吃 10MB 栈，100 个就是 1GB）。JDK 21 虚拟线程的栈是**堆上的 StackChunk 链表**，可以动态伸缩，这才是它敢开百万线程的原因之一。

### 3.4 线程中断：一个标志位 + 一次唤醒

`Thread.interrupt()` 在 JVM 层只做两件事：

```
Thread.interrupt()
   ├─ ① 设置 JavaThread::_interrupted = true（Java 层可见的标志）
   ├─ ② 如果线程正阻塞在可中断的等待上
   │     （park / sleep / wait / join / InterruptibleChannel）
   │     → 唤醒它（futex wake），并让它抛 InterruptedException
   └─ ③ 如果线程在跑，什么都不做 ← 只是"打个招呼"，线程必须自己检查
```

**关键语义（线程池大量依赖它们）：**

| 方法 | 行为 | 副作用 |
|---|---|---|
| `interrupt()` | 置标志 + 唤醒 | 不强制终止任何东西 |
| `isInterrupted()` | 只读，**不清除**标志 | — |
| `Thread.interrupted()` | 读并**清除**标志 | ⚠️ 容易误清，导致外层收不到中断 |
| 阻塞方法抛出 `InterruptedException` | 抛出时**自动清除**标志 | 所以 catch 后要 `Thread.currentThread().interrupt()` 恢复现场 |

> **这就是线程池 `runWorker` 里那段绕口代码的原因**：
> ```java
> if ((runStateAtLeast(ctl.get(), STOP) ||
>      (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
>     && !wt.isInterrupted())
>     wt.interrupt();
> ```
> 翻译成人话：**"如果池已经是 STOP 状态；或者刚收到中断信号且池也已经进入 STOP/之后的状态——那就确保当前线程的中断位被重新置上；否则把可能残留在中断位上的脏标记清掉，避免污染下一个任务。"** 因为线程是复用的，**上个任务留下的中断位如果不清，下个任务会莫名其妙地立刻抛异常**——这是线程池的经典坑之一。

### 3.5 平台线程 vs 虚拟线程：线程池的未来

```
【平台线程 Platform Thread】= 1:1，一个 Java 线程钉死一个内核线程
   开 10000 线程 → 10000 个 task_struct → 20GB 虚拟地址 + 调度器爆炸

【虚拟线程 Virtual Thread】(JDK 21+) = M:N
   ┌─ Continuation（用户态可续执行栈，可"暂停/恢复"，栈在堆上）
   ├─ Scheduler = ForkJoinPool（载体线程 carrier，数量 = CPU 核数）
   └─ mount / unmount：
        虚拟线程跑到阻塞点（socket read / queue take / sleep）
          → 把当前栈帧"快照"进堆（Continuation.yield）
          → unmount，载体线程立刻去跑别的虚拟线程
        数据到了
          → 调度器把 Continuation 重新 mount 到某个载体线程 → 继续执行

   效果：100 万个虚拟线程只需 ~核数 个内核线程
```

| 维度 | 平台线程 + 线程池 | 虚拟线程 |
|---|---|---|
| 每线程内存 | ~1MB 栈（虚拟） | 几百字节起，堆上按需增长 |
| 建/销成本 | 30~80 μs | 纳秒~微秒级（堆对象） |
| 阻塞代价 | 占死一个内核线程 | unmount，载体线程不浪费 |
| 并发上限 | 数百~数千 | 百万级 |
| 适用场景 | **CPU 密集、需要限流** | **IO 密集、高并发短任务** |
| 坑 | 线程数爆炸 | **`synchronized` 块内阻塞会 pin 住载体线程**（JDK 21 中；JDK 24 已修复 JEP 491）；`ThreadLocal` 百万份会吃内存 |

> **面试标准答法**："虚拟线程解决的是**阻塞的代价**，不是**并发的上限**。它会让你把'一个请求一个线程'的写法重新变得合理，但**它不替代线程池的两个能力：CPU 密集型任务的并发度控制、以及对下游资源的限流保护**。所以线程池会长期存在，只是 IO 池可以退化成 'ThreadPerTaskExecutor'。"

### 3.6 至此，桥接到线程池

把上面所有内容压缩成一句话：

> **`ThreadPoolExecutor` 做的事情，就是"预先 clone 出 N 个内核线程，让它们各自阻塞在队列的 futex 上，然后把 `Runnable` 一个个喂给它们"。**

下一章开始，拆解这个"喂"的精确过程。

---

## 第 4 章 ThreadPoolExecutor 的骨架：ctl、状态机、Worker

### 4.1 类图：谁继承了谁

```
java.util.concurrent.Executor
        │
        ├── ExecutorService
        │      │
        │      ├── AbstractExecutorService      （提供 submit/invokeAll/invokeAny 模板）
        │      │      │
        │      │      └── ThreadPoolExecutor    ★ 本文主角
        │      │             │
        │      │             ├── ScheduledThreadPoolExecutor（继承 + 加延时队列）
        │      │             │
        │      │             └── ForkJoinPool（另一套体系，work-stealing）
        │      │
        │      └── (自定义实现)
        │
        └── Executors（工厂类 —— 阿里规约明令禁止直接用，见 Q3）
```

**四个关键成员变量：**

```java
public class ThreadPoolExecutor extends AbstractExecutorService {

    // ① 状态 + 线程数的"原子合体"，整个类的核心锁
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    // ② 任务蓄水池
    private final BlockingQueue<Runnable> workQueue;

    // ③ 所有工作线程的集合（**只在持有 mainLock 时访问**）
    private final HashSet<Worker> workers = new HashSet<>();

    // ④ 主锁：保护 workers 集合、largestPoolSize、completedTaskCount 等统计量
    private final ReentrantLock mainLock = new ReentrantLock();

    // ⑤ 终止条件：让 awaitTermination() 能阻塞等待
    private final Condition termination = mainLock.newCondition();
}
```

> **为什么 `workers` 要用 `HashSet` + `mainLock` 保护，而不是 `ConcurrentHashMap`？**
> 因为对 `workers` 的操作都是"低频、需要快照语义"的（`shutdown` 遍历中断、统计大小、终止时判空）。低频场景下，一把 `ReentrantLock` 比并发容器更简单更省内存，且能顺便保护 `largestPoolSize` 等一组统计字段的一致性。

### 4.2 `ctl`：一个 int 装下两个东西

这是整个线程池最漂亮的设计，也是面试必问。

```java
private static final int COUNT_BITS = Integer.SIZE - 3;   // 32 - 3 = 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 2^29 - 1 = 536870911

// 状态值：高 3 位，低 29 位全 0
private static final int RUNNING    = -1 << COUNT_BITS;  // 111 000...0
private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 000 000...0
private static final int STOP       =  1 << COUNT_BITS;  // 001 000...0
private static final int TIDYING    =  2 << COUNT_BITS;  // 010 000...0
private static final int TERMINATED =  3 << COUNT_BITS;  // 011 000...0

// 三个拆包函数 —— 全部是位运算，零开销
private static int runStateOf(int c)     { return c & ~CAPACITY; }  // 取高 3 位
private static int workerCountOf(int c)  { return c &  CAPACITY;  }  // 取低 29 位
private static int ctlOf(int rs, int wc) { return rs | wc;        }  // 合体

// 大小比较可以用直接的 < > —— 因为状态值按"生命周期顺序"单调递增
private static boolean runStateLessThan(int c, int s) { return c < s;  }
private static boolean runStateAtLeast(int c, int s) { return c >= s; }
```

**内存布局（一张图看懂）：**

```
 31 30 29 │ 28 27 ... 3 2 1 0
┌──┬──┬──┼──────────────────────┐
│  │  │  │                      │
└──┴──┴──┴──────────────────────┘
 └──┬──┘  └────────┬───────────┘
  状态(3位)      线程数(29位, 最大 5.36 亿)

RUNNING    = 111 0000000... → int 值为负数（-536870912）
SHUTDOWN   = 000 0000000... → 0
STOP       = 001 0000000... → 536870912
TIDYING    = 010 0000000... → 1073741824
TERMINATED = 011 0000000... → 1610612736
```

**这个设计回答了三个"为什么"（面试高频追问）：**

| 追问 | 答案 |
|---|---|
| **为什么要合成一个 `int`，不拆成两个字段？** | 因为**状态与线程数必须原子地一起变**。拆成两个 `AtomicInteger`，在"减少线程数 + 检查是否该终止"这类复合操作上会出现中间态（如：线程数已减到 0，但状态还没改成 TIDYING，别的线程看到 ctl 会做出错误判断）。合体后**一次 CAS 同时完成两件事**，天然无中间态 |
| **为什么状态值要设计成"负数在前、正数在后"的单调顺序？** | 这样 `c < SHUTDOWN` 就等于 `isRunning`，`c >= STOP` 就等于"至少 STOP"。**省掉查表/多条件判断**，且能用一条 `if` 表达"RUNNING/SHUTDOWN/STOP/TIDYING/TERMINATED"的层级关系 |
| **为什么是 3 位状态 + 29 位线程数？** | 状态只有 5 个，2 位不够、3 位有余；29 位能给到 5 亿线程数（远超任何真实需要），**用 int 而不是 long 是因为 `AtomicInteger` 的 CAS 在 32 位和 64 位上都是原子的单条指令，而 `AtomicLong` 在 32 位机上需要 `cmpxchg8b`**，性能更差 |

> ⚠️ **注意 `RUNNING` 是负数**。这带来一个反直觉的结果：`ctl.get()` 得到的 int **不能当"线程数"看**，必须 `workerCountOf()`。调试时打印 `ctl` 看到 `-536870912` 别以为是错——那是 0 线程的 RUNNING。

### 4.3 五种状态与转换（含"为什么需要 TIDYING"）

```
                  shutdown()                   队列空 & 线程数=0
   ┌─────────┐  ──────────────→  ┌──────────┐  ─────────────→  ┌────────┐
   │ RUNNING │                    │ SHUTDOWN │                   │        │
   │         │  ──────────────→  │          │                   │TIDYING │
   └─────────┘   shutdownNow()    └──────────┘                   │        │
        │              │                │                       └────────┘
        │              ▼                │                            │
        │        ┌──────────┐           │ 队列空 & 线程数=0           │ terminated()
        │        │   STOP   │ ──────────┴──────────────────────────→│ 返回后
        │        └──────────┘                                        ▼
        │              │                                    ┌────────────┐
        └──────────────┴──────────────────────────────────→ │ TERMINATED │
                                                            └────────────┘
   各状态语义：
   RUNNING    ：接受新任务，也处理队列里的任务
   SHUTDOWN   ：**不接受**新任务，但**会**处理完队列中已有任务（"温和关门"）
   STOP       ：不接受新任务，不处理队列任务，且**中断正在执行的任务**
   TIDYING    ：所有任务已终止、workerCount=0，即将执行 terminated() 钩子
   TERMINATED ：terminated() 已执行完毕
```

**语义差异的一句话对照（面试最爱考）：**

| 状态 | `execute()` 新任务 | 队列中已有任务 | 正在执行的任务 |
|---|---|---|---|
| RUNNING | ✅ 接受 | ✅ 执行 | ✅ 不打断 |
| SHUTDOWN | ❌ 拒绝 | ✅ **执行完** | ✅ 不打断 |
| STOP | ❌ 拒绝 | ❌ **丢弃** | ❌ **中断** |

> **"为什么必须有 TIDYING 这个中间态？"**
> 因为 `terminated()` 是留给子类重写的钩子（`ScheduledThreadPoolExecutor` 用它取消定时任务、清空队列）。如果没有 TIDYING，就没有一个"线程数已归零、但钩子还没跑"的稳定瞬间来安全执行钩子。**TIDYING 是"宣告死亡"与"实际死亡"之间的手术室**。

### 4.4 七个构造参数：每个都有物理含义

```java
public ThreadPoolExecutor(
    int corePoolSize,      // ① 常驻线程数（即使空闲也不销毁，除非 allowCoreThreadTimeOut）
    int maximumPoolSize,   // ② 线程数硬上限（只在队列满时才会往这里扩）
    long keepAliveTime,    // ③ 非核心线程空闲多久后销毁（allowCoreThreadTimeOut 时对核心线程也生效）
    TimeUnit unit,         // ④ ③的单位
    BlockingQueue<Runnable> workQueue, // ⑤ 任务蓄水池
    ThreadFactory threadFactory,       // ⑥ 线程工厂（命名、优先级、守护、异常处理器）
    RejectedExecutionHandler handler   // ⑦ 满员时的泄压阀
)
```

**每个参数的"物理/工程含义"：**

| 参数 | 不只是"数字"，它实际控制的是 | 设错的后果 |
|---|---|---|
| `corePoolSize` | **常态并发度**。这些线程的栈 1MB×N 是常驻虚拟内存 | 设太大：内存浪费 + 空闲线程多（虽然不吃 CPU）；设太小：常态吞吐不足 |
| `maximumPoolSize` | **突发并发度上限**，也是对下游的"最大压力" | 设太大：突发时把下游打挂（DB 连接池耗尽）；设 `Integer.MAX_VALUE`：线程爆炸 |
| `keepAliveTime` | **弹性回收速度**。突发扩容后多久缩回来 | 设太小：频繁创建销毁，抖动；设太大：突发后资源长期占着 |
| `workQueue` | **缓冲能力 = 允许的"排队延迟"**。有界队列本质是"背压阀" | 无界队列：`maximumPoolSize` 永久失效 + OOM 风险（见 6.2） |
| `threadFactory` | **可观测性 + 线程属性** | 用默认工厂：线程名全是 `pool-N-thread-M`，**jstack 里根本认不出是谁**，线上排查灾难 |
| `handler` | **过载时的行为策略** | 默认 `AbortPolicy` 抛异常，业务侧没 catch 就丢任务 |

**构造函数的参数校验（源码里会抛 `IllegalArgumentException`）：**

```java
if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize ||
    keepAliveTime < 0) throw new IllegalArgumentException();
if (workQueue == null || threadFactory == null || handler == null)
    throw new NullPointerException();
```

> ⚠️ **注意源码不会校验"`corePoolSize > maximumPoolSize`"以外的语义问题**：比如"无界队列 + 大 `maximumPoolSize`"这种逻辑上矛盾但语法合法的组合，它照单全收。**参数之间的语义一致性要靠人来保证**——这正是很多线上事故的成因。

### 4.5 `Worker`：一个线程 + 一个锁 + 一个任务

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;        // 这个 Worker 独占的线程（由 ThreadFactory 创建）
    Runnable firstTask;         // 首次任务（可以非空，用于"新建即干活"）
    volatile long completedTasks; // 这个 Worker 累计完成的任务数（用于统计）

    Worker(Runnable firstTask) {
        setState(-1);           // ★ 关键：初始 AQS 状态为 -1，表示"禁止中断"
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);  // ★ 把自己当作 Runnable
    }

    public void run() { runWorker(this); }   // 线程真正执行的入口

    // —— 一个极简的、不可重入的互斥锁 ——
    protected boolean isHeldExclusively() { return getState() == 1; }
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {          // 0 → 1 才算拿到
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    public void lock()   { acquire(1); }
    public void unlock() { release(1); }
    public boolean tryLock() { return tryAcquire(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {  // ★ state>=0 才中断
            try { t.interrupt(); } catch (SecurityException ignore) { }
        }
    }
}
```

**这里有三个"神来之笔"，逐个拆开：**

#### ① 为什么 `Worker` 要 `implements Runnable`，把自己传给 `ThreadFactory`？

因为它需要一个**"线程 + 任务"的绑定载体**。如果让线程直接跑业务任务，线程跑完就废了；`Worker` 的存在让"线程"和"当前任务"解耦：

```
Worker.thread  的生命周期 ──────────────────────────────────→
         │                                    
         └─ Worker.run() ──→ runWorker(Worker) ──→ while(getTask()) { task.run() }
                             ↑
                    业务任务只是这个 while 循环里的一次迭代，
                    任务结束 ≠ 线程结束，线程回到 getTask() 继续等下一个
```

**这一行就是"线程复用"的全部秘密**——把"任务"降级为循环体的一次迭代。

#### ② `setState(-1)` 是什么意思？为什么初始值是 -1？

`AQS.state` 在这里被借用来表示三种意义：

| state | 含义 | 能否被 `interruptIfStarted` 中断 |
|---|---|---|
| `-1` | Worker **刚创建，线程还没 `start()`** | ❌ 跳过（中断一个还没跑的线程没意义且危险） |
| `0` | **空闲**（阻塞在 `getTask()` 上，等队列） | ✅ 可以中断（正是 `shutdownNow` 要做的） |
| `1` | **正在执行任务**（持锁） | ✅ 可以中断（但 `shutdown` 的"温和中断"会跳过它） |

`runWorker` 的第一件事就是 `w.unlock()`，把 `-1` 改成 `0`，注释写着 `// allow interrupts`。**这是一个"生命开关"**：从这一刻起，这个线程才是"可被中断的合法工作线程"。

#### ③ 为什么用 AQS 而不是 `ReentrantLock`？

这是**经典高压面试题**，标准答案有三层：

**第一层（表面）**：性能。`ReentrantLock` 有自己的 `Sync` 内部类、`Condition` 对象、`owner` 字段、重入计数。而 `Worker` 只需要"0/1 的互斥"这一件事，用 `AQS` 直接覆写 `tryAcquire`/`tryRelease` 是最小实现，**每个 Worker 省下几十字节 × 数百线程，且少一层对象**。

**第二层（语义）**：**不可重入性是刻意的**。

```
Worker 的锁的真正用途不是"保护临界区数据"，而是"标记线程忙/闲"：

  isLocked() == true  ⇔  该线程正在执行任务  ⇔  shutdown() 不该中断它
  isLocked() == false ⇔  该线程空闲阻塞在 getTask() ⇔ 可以安全中断它

如果锁可重入会怎样？
  线程在执行 task 期间，task 内部若再调用 pool 的方法（如提交任务、
  甚至触发 shutdown），同一线程可以重入拿到锁 → 状态判断混乱。
不可重入 → "锁在谁手里"这件事永远清晰、外部可判定。
```

**第三层（Doug Lea 的真实注释）**：

> *"We implement a simple non-reentrant mutual exclusion lock rather than use ReentrantLock because we do not want user code to be able to call `unlock()`... and because it avoids needing to track a lock count."*
> —— 不用 `ReentrantLock` 是因为**不想让用户代码（`beforeExecute`/`afterExecute`/`task.run()` 里）有机会调用 `unlock()` 破坏"忙/闲"标记**，同时省掉重入计数的开销。

**这三层合起来答，才是满分。** 只答"性能好"是及格线。

#### ④ `tryLock()` 是"如何判断线程空闲"的答案

```
shutdown() / shutdownNow() 需要"中断空闲线程"
     ↓
interruptIdleWorkers() 遍历 workers，对每个调 w.tryLock()
     ├─ tryLock 成功（拿到锁）→ 说明它现在**没在跑任务** → w.interrupt() 唤醒它
     │   └─ 拿到后立刻 unlock()，因为我们只是"探测"，不是要占着
     └─ tryLock 失败 → 说明它**持着锁 = 正在跑任务** → 跳过（shutdown 场景）
                                              → 仍中断（shutdownNow 场景，见 8.1）
```

**这是"用锁状态承载线程状态"的经典技巧**，比额外维护一个 `volatile boolean busy` 更可靠——因为锁的获取/释放与"执行任务"严格同生命周期，不可能出现"标记了 busy 但实际空闲"的不一致。

---
## 第 5 章 执行流程逐行走读

> 这一章是全文最硬的部分。**建议对照 JDK 8/11 的 `ThreadPoolExecutor.java` 源码逐段读**，下面每一段都标了行号区间附近的内容梗概（不同 JDK 版本行号有偏移，但逻辑一致）。

### 5.1 `execute()`：三步走，三次"回读 ctl"

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    // ① 读取当前快照（后面可能过期，所以后面会重新读）
    int c = ctl.get();

    // 【第 1 步】核心线程还没满 → 直接开一个新核心线程来跑这个任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))    // true = 以 corePoolSize 为上限
            return;                      // 成功即返回，任务已经交给新线程了
        c = ctl.get();                   // 失败（并发下已被别人占满 / 池已关闭）→ 重读
    }

    // 【第 2 步】核心满了 或 上一步失败 → 尝试入队（前提：池还活着）
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();         // ★ 入队成功后必须"重新检查"
        // 2a. 池在入队后被关闭了 → 把刚入队的任务摘回来，走拒绝
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 2b. 池还活着，但线程数变成了 0（极端：所有线程恰好在此之前退出）
        //     → 必须补一个线程，否则任务永远没人取
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);      // 补一个非核心线程，firstTask=null
        // 否则：正常情况，什么都不做，等着空闲线程来取

    // 【第 3 步】入队失败（队列满了，或池不是 RUNNING）→ 扩容到 max
    } else if (!addWorker(command, false))   // false = 以 maximumPoolSize 为上限
        reject(command);                      // 扩不上去了 → 拒绝策略
}
```

**流程图（这张图必须能背下来）：**

```
                        execute(command)
                               │
                               ▼
                 ┌──────────────────────────┐
                 │ workerCount < corePoolSize│
                 └──────┬───────────────────┘
                    是  │            否
                        ▼             │
        ┌───────────────────────┐      │
        │ addWorker(task, true) │      │
        └──────┬────────────────┘      │
           成功 │      失败（并发竞争/已关闭）
               ▼        │              │
             【开新线程   └──────────────┤
              直接执行】                 │
                                        ▼
                            ┌───────────────────────┐
                            │ isRunning && queue.offer│
                            └──────┬────────────────┘
                              成功  │        失败（队列满 / 非RUNNING）
                                    ▼                 │
                         ┌──────────────────┐         │
                         │ recheck ctl      │         │
                         └──┬────────┬──────┘         │
                    已关闭  │        │ wc==0           │
                            ▼        ▼                 │
                    ┌──────────┐  ┌────────────┐       │
                    │remove()  │  │addWorker   │       │
                    │→reject() │  │(null,false)│       │
                    └──────────┘  │补一个线程   │       │
                                  └────────────┘       │
                                                       ▼
                                          ┌────────────────────────┐
                                          │ addWorker(task, false) │
                                          └──────┬─────────────────┘
                                             成功 │      失败（已达max）
                                                 ▼              ▼
                                            【开新线程       ┌──────────┐
                                             直接执行】      │ reject() │
                                                            └──────────┘
```

#### 为什么"先开线程"再"入队"，而不是反过来？

**这是面试深水区。** 反过来（先 `offer` 再 `addWorker(command, true)`）会造成两个问题：

**问题 1：任务可能"困死"在队列里，没有线程能取。**
设 `corePoolSize=5, max=5, queue容量=1`。若先入队：任务 A 入队成功、准备开线程时，`addWorker` 因并发或已关闭而失败 → **A 在队列里，但没有任何线程会来取它**（队列里只有一个元素，也不会触发扩容路径）。而"先开线程"的写法，失败就直接落到 `else if` 分支去扩容或拒绝，任务永远有明确归宿。

**问题 2：无法区分"扩容"和"拒绝"的时机。**
`execute` 的三步实际上是一条**优先级链**：核心线程 → 排队 → 扩容 → 拒绝。先开线程，才能让"核心未满"这个判断点的失败可回退（回退到入队）；如果先入队，"核心未满"这个信息就丢了。

> **Doug Lea 的设计原则可以总结为一句话**：**宁可先起线程，也不要先把任务塞进一个可能无人消费的队列。** 因为"线程多余"是可回收的资源浪费，"任务无人处理"是正确性事故。

#### 为什么第 2 步入队成功后还要 `recheck`？

因为 `offer` 与"池状态变化 / 线程全部退出"之间没有原子性。两个具体竞态：

| 竞态 | 时间线 | `recheck` 如何补救 |
|---|---|---|
| **入队后池被关闭** | T1: `isRunning` 为 true → T2: `offer` 成功 → T3: 另一个线程 `shutdown()` 且所有 worker 取完队列后退出了 → T4: 这个任务永远留着 | 检查到 `!isRunning` → `remove(command)` 摘回 → `reject()`。**注意：只有 `remove` 成功才 reject，否则说明任务已被某个 worker 取走，就正常执行** |
| **入队后线程数归零** | `corePoolSize=0`（`allowCoreThreadTimeOut` 也可能导致），或所有线程恰好超时退出 → 队列里有任务但没有线程 | 检查到 `workerCountOf == 0` → `addWorker(null, false)` 补一个"空任务"线程，它会立刻去 `getTask()` 取到那个任务 |

> ⚠️ **第二类竞态非常真实**：`new ThreadPoolExecutor(0, N, 60s, ..., new SynchronousQueue<>())`（即 `newCachedThreadPool` 的参数）时，`corePoolSize=0`，**每个任务都必须走"入队 + 补线程"路径**，`recheck` 的 `wc==0` 分支是常态而非异常。

#### 为什么第 1 步的 `workerCountOf < corePoolSize` 和 `addWorker` 内部的检查不冲突？

因为外层判断是**乐观的、非原子的**，`addWorker` 内部才是**权威的**：

```
外层：workerCountOf(c) < corePoolSize   ← 可能同时有 10 个线程都判断为 true
内层 addWorker：CAS 递增 + 再次检查上限 ← 只有真正成功的才会超过，且不会超过 core
结果：可能多创建几个线程，但绝不会突破上限；多余的线程会因 CAS 失败而快速返回 false
```

**这就是"乐观检查 + 权威 CAS"的经典并发模式**，代价是多几次无用的 CAS 尝试，收益是避免在外层加锁。

### 5.2 `addWorker()`：双层自旋 + 三道闸门

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:                                   // ★ 外层循环：处理"状态变化"
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 【闸门 1：状态检查 —— 混合条件，最容易读错的一行】
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
            return false;
        // 拆解：
        //   rs >= SHUTDOWN                      → 已不允许接新任务
        //   && !( 三个条件同时成立才算例外 )      → 唯一例外：
        //        rs == SHUTDOWN   （只允许 SHUTDOWN，不允许 STOP 及之后）
        //        && firstTask == null （必须是"补位线程"，不能带新任务）
        //        && !queue.isEmpty()   （队列里确实还有活儿要干）
        // 换句话说：**SHUTDOWN 状态下，只允许"为了清空队列而补线程"**

        // 【内层循环：线程数 CAS 递增】
        for (;;) {
            int wc = workerCountOf(c);
            // 【闸门 2：容量检查】
            if (wc >= CAPACITY ||                       // 29 位装不下了（5.36 亿）
                wc >= (core ? corePoolSize : maximumPoolSize))  // 超过目标上限
                return false;
            if (compareAndIncrementWorkerCount(c))      // ★ 原子递增，成功即跳出
                break retry;
            c = ctl.get();                              // CAS 失败 → 重读
            if (runStateOf(c) != rs)                    // 状态变了 → 回外层重新做状态检查
                continue retry;
            // 状态没变，只是线程数被别人改了 → 继续内层循环，重新检查上限
        }
    }

    // 【闸门 3：加锁后"三度确认"，然后才真正 start】
    boolean workerStarted = false, workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);                 // 创建 Worker（此时 state = -1）
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN ||               // 还是 RUNNING
                    (rs == SHUTDOWN && firstTask == null)) {  // 或是"补位线程"
                    if (t.isAlive()) throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize) largestPoolSize = s;
                    workerAdded = true;
                }
            } finally { mainLock.unlock(); }
            if (workerAdded) {
                t.start();                             // ★ 这里才真正 clone 内核线程
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);   // 回滚：从 workers 移除 + 递减 ctl + tryTerminate
    }
    return workerStarted;
}
```

**三个细节值得单独说：**

**① 为什么"先递增 ctl，后创建线程"？**
因为**创建线程可能失败（OOM），而 ctl 递增是"预留名额"**。先占名额再做重活，失败就回滚（`addWorkerFailed`）。反过来"先建线程再递增"会导致线程已存在但计数没加，统计与状态不一致。

**② 为什么"t.start()"放在 mainLock 外面？**
因为 `start()` 内部会走 `pthread_create`，是一个**几十微秒的系统调用**。放在锁里会把 `mainLock` 的临界区拉长几十倍，阻塞 `shutdown`、`getPoolSize` 等所有要拿 `mainLock` 的操作。**"锁内只做内存操作，锁外做慢操作"**——这是并发编码的基本原则。

**③ `if (t.isAlive()) throw new IllegalThreadStateException();` 是防御什么？**
防御 `ThreadFactory` 返回一个**已经 start 过的线程**（错误的自定义工厂）。JDK 用这个检查让错误尽早暴露，而不是等到诡异的状态错乱。

### 5.3 `runWorker()`：线程的"主循环"，也是复用的核心

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;          // 清空，为下一轮做准备
    w.unlock();                  // ★ state: -1 → 0，"允许中断"，线程正式上班
    boolean completedAbruptly = true;   // 是否"异常退出"（决定要不要补线程）
    try {
        // ★★★ 复用就发生在这里：while 循环不断取新任务 ★★★
        while (task != null || (task = getTask()) != null) {
            w.lock();            // ★ 标记"我在忙"（shutdown 就不会中断我）

            // 【中断状态清理 —— 全文最绕的一段，但必须理解】
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
                && !wt.isInterrupted())
                wt.interrupt();
            // 目的：线程是复用的，上一个任务可能留下"中断位"或"未处理的中断"
            //  - 若池已是 STOP：无条件重新置中断位（提醒当前/下一个任务该收工）
            //  - 若池不是 STOP：用 Thread.interrupted() 把残留的中断位清掉，
            //    避免污染下一个任务（注意这个调用本身就清除了标志）
            //  - !wt.isInterrupted() 是防止重复置位

            try {
                beforeExecute(wt, task);      // 钩子：任务执行前（可用于埋点/统计）
                Throwable thrown = null;
                try {
                    task.run();               // ★ 执行真正的业务代码
                } catch (RuntimeException x) { thrown = x; throw x; }
                  catch (Error x)           { thrown = x; throw x; }
                  catch (Throwable x)       { thrown = x; throw new Error(x); }
                  // 注意：异常会被抛出，导致 while 循环终止 → completedAbruptly = true
                  //      → processWorkerExit 会补一个新线程
                finally {
                    afterExecute(task, thrown);  // 钩子：无论成败都执行（可用于监控耗时）
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();               // ★ 标记"我空闲了"
            }
        }
        completedAbruptly = false;        // 正常退出（getTask 返回 null）
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

**三个必须理解的要点：**

**① 为什么 `task.run()` 抛异常会让线程死掉？**
`runWorker` 没有 catch 住 `while` 循环里的异常（只在 `task.run()` 外面 catch 了又 rethrow）。异常穿出 `runWorker` → 穿出 `Worker.run()` → 到达 `Thread.run()` 的顶层 → 线程进入 `TERMINATED`。**但 `finally` 里的 `processWorkerExit` 会补一个新线程**，所以线程池的"容量"不会缩水。

**这也解释了 `execute()` 与 `submit()` 的差异（见 Q2）：**
- `execute(r)` 直接跑 r，异常无人接，**依赖 `processWorkerExit` 补线程**，异常本身通过 `UncaughtExceptionHandler` 打印或静默（默认打到 stderr）。
- `submit(r)` 把 r 包进 `FutureTask`，**异常被 `FutureTask` 捕获并存进 `outcome`**，线程不死，但**如果你不 `get()`，异常就永久沉默**——这就是"submit 吞异常"的根因。

**② `beforeExecute` / `afterExecute` 是官方留的监控钩子。**
但要记住：`afterExecute` 拿到的 `Throwable` **对 `submit` 提交的任务是 `null`**（异常被 `FutureTask` 吃了），只有 `execute` 提交的才能拿到。这是很多监控组件统计不准的原因。

**③ `w.lock()` 与 `w.unlock()` 包围的范围，正好等于"这个线程忙"的时间。**
所以 `getActiveCount()` ≈ 处于 lock 状态的 Worker 数（实际实现是遍历 `workers` 数 `isLocked()`），**不是"正在跑任务的线程数"的近似，而是精确值**。

### 5.4 `getTask()`：线程从哪里"取活"，什么时候"下班"

```java
private Runnable getTask() {
    boolean timedOut = false;         // 上一轮 poll 是否超时
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 【退出条件 1：池状态】
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();   // ★ 自己负责减计数（所以调用方不用减）
            return null;              // 返回 null → runWorker 的 while 结束 → 线程退出
        }
        // 拆解：
        //   STOP/TIDYING/TERMINATED → 无条件退出（队列里的任务也不要了）
        //   SHUTDOWN               → 只有队列空了才退出（这就是"温和关闭会跑完队列"的实现点）

        int wc = workerCountOf(c);
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // ↑ 这个线程"是否属于可超时回收"的那一类

        // 【退出条件 2：该收缩了】
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;                 // CAS 失败 → 重试
        }
        // 拆解两个 && ：
        //   前面：线程数超了 max（发生在 setMaximumPoolSize 调小时）
        //        或 超时了（timed && timedOut）
        //   后面：还有别的线程 或 队列空了 → 才能安全退出
        //        ★ 关键：如果只剩我这一个线程且队列非空，我不能走（否则没人干活）

        // 【取任务：两条路径，对应"阻塞"和"限时阻塞"】
        try {
            Runnable r = timed
                ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)  // 限时等，超时返回 null
                : workQueue.take();                                    // 死等，直到有任务或被中断
            if (r != null) return r;
            timedOut = true;           // poll 超时了，下一轮循环会判"该退出"
        } catch (InterruptedException retry) {
            timedOut = false;          // ★ 被中断不算超时，重置后重新循环（重新判断池状态）
        }
    }
}
```

**这段代码的四个"坑点"（面试常挖）：**

| 坑点 | 解释 |
|---|---|
| **`take()` vs `poll(timeout)` 的选择** | `timed == false` 时用 `take()` 无限阻塞（核心线程常驻）；`timed == true` 时用 `poll()` 限时等（非核心线程/允许超时的核心线程）。**两者底层都是 futex，但 `poll` 带超时 = 内核定时器，会有额外的定时 wheel 开销** |
| **`timedOut` 为什么要用变量"跨循环携带"** | 因为 `poll` 返回 null 的瞬间不能直接 `return null`——可能此刻池状态刚变成 SHUTDOWN，或者需要再确认一次容量。所以标记 `timedOut = true`，**下一轮循环在统一的地方做退出判断**，保证判断的一致性 |
| **`catch (InterruptedException) { timedOut = false; }` 为什么"吞掉"中断** | 因为中断在这个位置的含义是"有人在叫我（`shutdown`/`shutdownNow`/缩容）"。**不能直接退出，必须回到循环顶部重新判断池状态**：如果是 SHUTDOWN 且队列非空，我应该继续干活而不是退出。**这就是"温和关闭不打断正在排队的任务"的实现细节** |
| **`decrementWorkerCount()` 在这条路径，而 `compareAndDecrement` 在另一条** | 前者是"状态已定，必然退出"的确定性路径，直接减；后者是"需要抢着减，抢不到就重试"的竞争路径。**同一个函数里两种减法，是 Doug Lea 对"确定 vs 竞争"的区分** |

**`getTask()` 返回值汇总：**

```
返回 Runnable  → 有活干，回到 runWorker 的 while 继续
返回 null      → 三种原因，都会让线程退出：
                  ① 池已 STOP/之后，或 SHUTDOWN 且队列空   → 正常退休
                  ② 线程数超过 max（被缩容）                → 被裁
                  ③ poll 超时（闲太久）                     → 超时退休
```

### 5.5 `processWorkerExit()` + `tryTerminate()`：退出与终止的级联

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly)
        decrementWorkerCount();      // ★ 异常退出：getTask 没来得及减，这里补减
                                     //   正常退出时 getTask 已经减过了，不能重复减

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;   // 汇总统计
        workers.remove(w);                        // 从活跃集合摘除
    } finally { mainLock.unlock(); }

    tryTerminate();                  // ★ 每次都尝试推进终止状态机

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {             // 池还在 RUNNING/SHUTDOWN
        if (!completedAbruptly) {                // 正常退休 → 看要不要保底
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;                         // 队列还有活，至少留一个线程
            if (workerCountOf(c) >= min)
                return;                          // 不需要补
        }
        addWorker(null, false);                  // 补一个"空任务"线程顶上
    }
}
```

**`tryTerminate()`：状态机的"最后一公里"**

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 【三重返回条件 —— 任何一个成立就不能终止】
        if (isRunning(c) ||                                   // ① 还在跑
            runStateAtLeast(c, TIDYING) ||                    // ② 已经进入终止流程（别人在办）
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))  // ③ 温和关闭但队列没清空
            return;

        // 【关键：线程数不为 0，但有资格终止了（队列已空）】
        if (workerCountOf(c) != 0) {
            interruptIdleWorkers(ONLY_ONE);   // ★ 只中断"一个"空闲线程
            return;
        }
        // 为什么要"只中断一个"？
        //   因为每个线程退出时都会调用 tryTerminate()。
        //   中断一个 → 它醒来 → 判断状态 → 退出 → 再次调用 tryTerminate() → 再中断一个
        //   → 形成"接力式级联唤醒"，把剩下所有空闲线程逐个叫醒并送走。
        //   若这里一次中断全部，会有大量线程同时抢 mainLock、竞争激烈；
        //   逐个传播既省锁竞争，又保证"最后那个线程一定会把状态推进到 TERMINATED"。

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {    // ★ CAS 抢"宣告死亡"权
                try {
                    terminated();                              // 钩子（子类可重写）
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));              // 正式死亡
                    termination.signalAll();                    // 唤醒 awaitTermination() 的等待者
                }
                return;
            }
        } finally { mainLock.unlock(); }
        // CAS 失败 → 别人已经推进了 → 重试循环（下一轮会因 TIDYING 而返回）
    }
}
```

> **`interruptIdleWorkers(ONLY_ONE)` 这个"只叫醒一个"的设计，是理解线程池优雅终止的关键，也是面试区分"看过源码"和"背过八股"的分水岭。** 见 Q17。

### 5.6 一个完整时序：从提交到终止

```
时刻   提交线程 / 空闲 Worker / 池状态
────────────────────────────────────────────────────────────────────────
T0     execute(task) → wc(2) < core(4) → addWorker(task,true)
T1       ├─ ctl: wc 2→3 (CAS)
T2       ├─ new Worker(task) [state=-1] → workers.add → t.start()
T3       └─ 返回 true，提交线程走了
T4     [新线程] runWorker: firstTask=task, unlock(state 0), w.lock()
T5       ├─ task.run() → 业务代码
T6       └─ unlock(state 0), completedTasks++
T7     [线程进入循环] getTask(): timed=false → workQueue.take() → futex 睡眠 ★
       ... 空闲 N 秒，CPU 占用 0 ...
T8     新任务入队 → 队列 signal → futex wake → 该线程被唤醒
T9     getTask 返回 task → runWorker 再跑一轮 → 回到 T4
       ...
T10    shutdown() → ctl: RUNNING→SHUTDOWN → interruptIdleWorkers()
T11      └─ 遍历 workers，tryLock() 成功的（空闲线程）→ interrupt() → 从 take() 抛出 IE
T12    getTask: catch IE → timedOut=false → 重新循环 → 判 SHUTDOWN 且队列空 → 减计数 → return null
T13    runWorker: while 结束 → completedAbruptly=false
T14    processWorkerExit: workers.remove → tryTerminate()
T15      └─ wc != 0 → interruptIdleWorkers(ONLY_ONE) → 再叫醒一个 → 级联
T16    ...最后一个线程：wc==0 → CAS 到 TIDYING → terminated() → TERMINATED → signalAll
T17    main 线程的 awaitTermination() 返回 true
```

**关键观察点：**

- **T7 是线程池"不占 CPU"的原因**——线程真的睡在 futex 上，不是"while 轮询"。
- **T8 的唤醒延迟**决定了一个任务从"入队"到"真正开始执行"的最小延迟（**典型 10~100 μs**，这就是为什么线程池不能替代"直接调用"，也是为什么"跨线程传递"有成本）。
- **T12 的 `catch` 不退出** 是"温和关闭"的核心：中断只是"提醒"，线程回到循环顶部重新判断，从而保证队列任务被跑完。

### 5.7 三张"反直觉"的边界图

**边界 1：核心线程是"懒创建"的**

```
new ThreadPoolExecutor(4, 4, 60s, ..., queue) 之后，workers.size() == 0
                                                     ↑ 一个线程都没起！

只有 execute() 被调用时，才按需逐个创建核心线程。
想预热 → prestartCoreThread()（起 1 个）/ prestartAllCoreThreads()（起满 core）
```

**边界 2：任务是"三条路"进去的，延迟差异巨大**

```
路径 A：直接新线程执行    延迟 ≈ 线程创建 30~80 μs（首次才有）
路径 B：进队列等空闲线程   延迟 ≈ 唤醒 10~100 μs（若队列积压则 = 排队长度 × 单任务耗时）
路径 C：被拒绝            延迟 = 0，但业务失败

★ 队列容量越大 → 越多任务走 B → 延迟越高、越容易"雪崩式积压"
★ 所以"有界队列 + 合理拒绝 + 监控队列长度"是生产必须
```

**边界 3：`maximumPoolSize` 在"无界队列"下是死参数**

```
workQueue = new LinkedBlockingQueue<>()   // 容量 = Integer.MAX_VALUE
     ↓
execute 的第 3 步（入队失败 → 扩容）永远走不到，因为 offer 永远成功
     ↓
maximumPoolSize 永远不会被用上，实际最大线程数 = corePoolSize
     ↓
任务无限堆积 → 内存涨 → 最终 OOM 或 GC 崩溃
（这正是 Executors.newFixedThreadPool 的致命缺陷）
```

---
## 第 6 章 队列与拒绝策略：线程池的"缓冲"与"泄压阀"

### 6.1 六种队列的底层结构与行为差异

`BlockingQueue` 不是一个"容器"那么简单，**它的数据结构决定了线程池的吞吐、延迟形态和公平性**。

```
┌───────────────────────────────────────────────────────────────────────────┐
│ 队列              数据结构        锁模型           阻塞实现      容量      │
├───────────────────────────────────────────────────────────────────────────┤
│ ArrayBlockingQueue  环形数组      单锁 ReentrantLock  两个Condition  有界必填 │
│                                  + notEmpty/notFull   (futex)             │
├───────────────────────────────────────────────────────────────────────────┤
│ LinkedBlockingQueue 单向链表(Node) 双锁 putLock/takeLock 两个Condition 默认无界 │
│                                  + AtomicInteger count (futex)   (可指定)  │
├───────────────────────────────────────────────────────────────────────────┤
│ SynchronousQueue   无存储        CAS 无锁(Transfer)  park/unpark   容量=0  │
│                    (手递手)      Stack(非公平)/Queue(公平)                │
├───────────────────────────────────────────────────────────────────────────┤
│ PriorityBlockingQueue 二叉堆数组  单锁 + notEmpty      非空才阻塞  无界(默认11)│
│                                  (put 永不阻塞)                   扩容    │
├───────────────────────────────────────────────────────────────────────────┤
│ DelayQueue         优先级队列   同 PriorityBlocking  + 定时器     无界     │
│                    + 到期时间    take 需队首到期                    │
├───────────────────────────────────────────────────────────────────────────┤
│ LinkedBlockingDeque 双向链表     单锁 + notEmpty/notFull 两个Condition 默认无界│
│                    (可从两端取)                                          │
└───────────────────────────────────────────────────────────────────────────┘
```

**逐个看底层实现要点：**

#### ① `ArrayBlockingQueue`——环形数组 + 一把锁两个条件

```
内部字段：
  final Object[] items;        // 环形数组，容量创建时定死
  int takeIndex, putIndex;     // 两个游标，到末尾就回绕（取模 / 位运算）
  int count;                   // 当前元素数
  final ReentrantLock lock;    // ★ 只有一把锁：入队和出队互斥
  final Condition notEmpty;    // 出队者在这里等
  final Condition notFull;     // 入队者在这里等

入队：
  lock.lock();
  while (count == items.length) notFull.await();   // 满了就等（注意是 while 不是 if！）
  items[putIndex] = e;
  if (++putIndex == items.length) putIndex = 0;    // 环形回绕
  count++;
  notEmpty.signal();                                // 叫醒一个出队者
  lock.unlock();
```

> **`while` 而不是 `if` 的原因**：`await()` 返回不代表条件成立（可能被"虚假唤醒"spurious wakeup，或被别的线程先抢了名额）。**所有 `Condition.await()` 必须包在 `while` 里重查条件**——这是 `Object.wait()` 和 `Condition.await()` 的通用铁律，面试常考。

**特点**：数组预分配，**内存布局紧凑、无 Node 对象开销、GC 压力小**；但**入队出队共用一把锁**，高并发下是瓶颈。

#### ② `LinkedBlockingQueue`——双锁分离，吞吐更高

```
内部字段：
  final ReentrantLock takeLock = new ReentrantLock();
  final Condition notEmpty = takeLock.newCondition();
  final ReentrantLock putLock = new ReentrantLock();
  final Condition notFull  = putLock.newCondition();
  final AtomicInteger count = new AtomicInteger();   // ★ 计数用原子类，跨两把锁共享

★ 关键：入队只拿 putLock，出队只拿 takeLock → **生产者与消费者可以真并行**
★ 代价：每个元素一个 Node 对象（额外 ~ 24~40 字节），GC 压力大
★ 默认容量 = Integer.MAX_VALUE → 这就是"无界队列"的万恶之源
```

**为什么 `count` 用 `AtomicInteger` 而不是普通 int？** 因为它被两把锁同时访问（入队时 putLock 下改，出队时 takeLock 下读），普通 int 会有可见性问题。`AtomicInteger` 同时提供**可见性 + 原子性**（虽然增减本身不需要原子，但需要 volatile 语义）。

#### ③ `SynchronousQueue`——容量为 0 的"手递手"

这是最反直觉的一个：**它不存元素**。`put` 必须等一个 `take` 来接手，反之亦然。

```
两种实现模式：

【非公平模式（默认，TransferStack）】
   后进先出（LIFO）栈
   put 压栈 → 自旋/CAS 找匹配 → 匹配上直接交换数据 → 无匹配则 park
   优点：吞吐高（后到的消费者更可能还在 CPU 上，命中率高）
   缺点：饥饿（先来的可能一直匹配不上）

【公平模式：new SynchronousQueue<>(true)（TransferQueue）】
   先进先出（FIFO）队列
   优点：公平，先到先服务
   缺点：吞吐低一些

★ 底层是"无锁 + park/unpark"，不走队列数组/链表，因此极快
★ 但它把"缓冲"能力降为 0 —— 没有空闲线程时，offer 直接失败！
```

**`newCachedThreadPool` 为什么配 `SynchronousQueue`？**

```java
new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>());
```

语义是：**"有闲线程就交给它（transfer 成功）；没有就立刻新建一个线程（offer 失败 → 走扩容）；线程闲 60 秒就销毁"**。这套组合是"纯弹性、零排队"的模型，**适合大量短小的异步任务**，但 `max=Integer.MAX_VALUE` 让它能创建无限线程——**一旦任务变慢，线程数会爆到内核扛不住**，这就是它被阿里规约禁用的原因。

#### ④ `PriorityBlockingQueue`——二叉堆 + 无界

```
底层是 Object[] queue + 二叉堆（小顶堆），put 用 siftUp，take 用 siftDown
★ put 永不阻塞（无界），只有 take 会阻塞
★ 元素必须实现 Comparable，或构造时传 Comparator
★ 注意：PriorityBlockingQueue 只保证"取出时有序"，迭代时不保证
```

**用它做"优先级线程池"的坑**：线程池的 `getTask()` 用的是 `poll`/`take`，**不感知优先级**；优先级只在"同一时刻有多个任务排队、且消费者来取"时体现。更致命的是**低优先级任务可能饿死**（高优先级源源不断进来）。所以"优先级线程池"通常要配合**多个线程池 + 按优先级路由**，或自己实现。

#### ⑤ `DelayQueue`——`ScheduledThreadPoolExecutor` 的心脏

```
DelayQueue = PriorityBlockingQueue<Delayed> + 到期判断

take() 的逻辑：
  lock.lock();
  for (;;) {
      first = queue.peek();          // 队首（最早到期）
      if (first == null) notEmpty.await();
      else {
          long delay = first.getDelay(NANOSECONDS);
          if (delay <= 0) return queue.poll();         // 到期了，取走
          available.awaitNanos(delay);                  // 没到期，睡到那个时刻
      }
  }
★ 所以 DelayQueue 的 take 是"精确睡到队首到期时间"，而不是轮询
```

> **`ScheduledThreadPoolExecutor` 的关键改造**：它把 `workQueue` 换成 `DelayedWorkQueue`（自己实现的变体），并把 `getTask()` 重写为"取到期任务 + 若无任务则阻塞到下一个到期点"。**它的 `maximumPoolSize` 无意义（固定 `Integer.MAX_VALUE`），`corePoolSize` 才决定并行度**——这是常考细节，见 Q31。

### 6.2 无界队列：线程池最经典的"死亡陷阱"

**问题链条（这是阿里规约那条规则的完整推理）：**

```
无界队列（LinkedBlockingQueue 默认容量）
   ↓
execute() 第 3 步"入队失败"永远不成立
   ↓
maximumPoolSize 永久失效，实际线程数封顶在 corePoolSize
   ↓
任务到达速率 > 处理速率时，队列无限增长
   ↓
① 内存持续上涨 → GC 压力剧增 → Full GC 频繁 → STW 变长 → 处理更慢 → 队列涨得更快（正反馈雪崩）
② 最终 OutOfMemoryError: Java heap space，整个 JVM 挂掉
③ 即使没 OOM，队列里任务的"等待时间"已经远超业务超时，执行业务逻辑毫无意义（算了也白算）
```

**从"排队论"看这件事（Little's Law）：**

```
利特尔法则： L = λ × W
   L = 系统中平均任务数（= 队列长度 + 正在执行的任务数）
   λ = 任务到达速率（QPS）
   W = 任务在系统中的平均停留时间（等待 + 执行）

假设 λ = 1000 QPS，W = 2 s（因为排队），则 L = 2000 个任务同时在系统里。
若每个任务对象及其上下文占 5 KB → 2000 × 5KB = 10 MB，看似不多；
但 W 会因为排队而自我放大（任务越多 → 每个越慢 → W 更大 → L 更大）
→ W 从 2s 涨到 60s，L 就涨到 60000 → 300 MB
→ 再涨到 600s，就是 3 GB → OOM
```

> **结论：队列容量不是"越大越安全"，而是"越大越晚出事、出事时越惨"。**
> 正确做法是**有界队列（容量 = 你能容忍的排队延迟 × 峰值 QPS） + 明确的拒绝策略 + 对队列长度的监控告警**。

**队列容量的经验算法：**

```
队列容量 ≈ 峰值 QPS × 可容忍排队延迟(s)

例：峰值 2000 QPS，业务 SLA 允许 P99 延迟 500ms，单任务平均耗时 50ms
    → 可容忍排队 ≈ 500 - 50 = 450ms
    → 容量 ≈ 2000 × 0.45 ≈ 900，取整 1000

若容量 1000 仍然天天满 → 说明不是队列小，而是**处理能力不足**，
应该扩线程/扩实例/优化任务耗时，而不是把队列调大。
```

### 6.3 四种拒绝策略 + 自定义

```java
// ① AbortPolicy（默认）—— 直接抛异常
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    throw new RejectedExecutionException("Task " + r.toString() +
                                         " rejected from " + e.toString());
}
// 特点：不丢任务（异常抛给提交者），但**要求调用方必须处理**，否则业务静默失败
// 适用：能明确感知失败并重试/降级的场景

// ② CallerRunsPolicy —— 让提交任务的线程自己跑
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {      // ★ 关键：如果池已关闭，就直接丢弃
        r.run();                 // 在调用者的线程里同步执行！
    }
}
// 特点：**天然的负反馈/背压机制**——提交者被拖去干活，它就没法继续提交，自然降速
// 适用：允许阻塞调用者的场景（如内部批量任务），不适合 Web 请求线程（会把 Tomcat 线程占满）
// 坑：如果调用者是 Netty/Reactor 的 IO 线程，会把整个事件循环卡住！

// ③ DiscardPolicy —— 静默丢弃
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) { }
// 特点：什么都不做，**任务凭空消失，无任何痕迹**
// 适用：可容忍丢失的"尽力而为"通知类任务（如非核心埋点）
// 风险：排查问题时完全无声，生产慎用

// ④ DiscardOldestPolicy —— 丢掉最老的，再试一次
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();     // 踢掉队首（最老的）
        e.execute(r);            // 重新提交（可能再次失败，但不会无限递归，因为队列腾出位置）
    }
}
// 特点："保新弃旧"，适合**只关心最新状态**的场景（如实时行情快照、位置上报）
// 坑：如果队列是 PriorityBlockingQueue，poll 掉的是"优先级最高"的，语义错位
```

**自定义策略的典型实现（生产常用）：**

```java
public class BlockingAwarePolicy implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        // 1) 先上报指标（最重要的动作，否则永远不知道在拒）
        Metrics.counter("threadpool.reject", e.getPoolSize()).inc();
        // 2) 落盘/落库做补偿，避免任务真丢
        RetryStore.save(r);
        // 3) 再决定是抛异常还是降级
        if (isCritical(r)) throw new RejectedExecutionException("critical task dropped");
        // else 走降级逻辑
    }
}
```

**四种策略对比表（背这张表）：**

| 策略 | 是否丢任务 | 是否阻塞调用者 | 是否有反馈信号 | 典型场景 |
|---|---|---|---|---|
| `AbortPolicy` | 不丢（抛异常） | 否 | 异常 | 默认、能感知失败的核心任务 |
| `CallerRunsPolicy` | 不丢 | **是** | 调用者变慢 | 内部批处理、需要背压 |
| `DiscardPolicy` | **丢** | 否 | **无** | 可丢的埋点/通知 |
| `DiscardOldestPolicy` | 丢"最老的" | 否 | 无 | 只关心最新值的实时场景 |

### 6.4 "队列满" vs "线程满"：两个完全不同的语义

很多人把这两件事混为一谈，实际上它们是**两个不同阶段的信号**：

```
任务进来
   │
   ├─ 阶段 1：线程数 < core        → 直接开线程（未满，无排队）
   │
   ├─ 阶段 2：线程 = core，且队列未满 → 排队（**等待**，还没满）
   │
   ├─ 阶段 3：线程 = core，队列满   → 扩容（往 max 方向加线程）
   │             ↑ 此时是"队列满但线程没满"，说明**缓冲耗尽，但还有弹性**，
   │               线程池正在"加速"，这是最需要关注但最容易被忽略的时刻
   │
   └─ 阶段 4：线程 = max，队列也满   → 拒绝（**真正打满**）
```

> **运维含义**：
> - 如果监控只看到"拒绝数上升"，说明已打满（阶段 4），要立刻降级。
> - 如果看到"线程数在 core 和 max 之间波动"，说明在阶段 3，是**设计内的弹性行为**，不必惊慌。
> - **最怕的是"线程恒等于 core、队列持续增长、拒绝数恒为 0"**——这说明你用了无界队列，正在"安静地走向 OOM"（见 6.2）。

---

## 第 7 章 容量规划：线程数到底该设多少

### 7.1 三个层次的方法，从"口诀"到"推导"

**① 口诀层（面试及格线）**

```
CPU 密集型：N = CPU 核数 + 1
   → "+1" 是为了在偶发缺页/中断时仍有线程可用，避免 CPU 空转

IO 密集型：N = CPU 核数 × 2 （经验值）
   → 或 N × (1 + 阻塞时间/计算时间)
```

**② 公式层（Brain Goetz《Java 并发编程实践》的经典公式）**

```
            ┌        W ┐
   N = N_cpu × │ 1 + ─── │  × U_cpu
            └        C ┘

   N_cpu：核数
   U_cpu：目标 CPU 利用率（0~1，如 0.8）
   W   ：单任务的平均**等待时间**（IO、锁、RPC 等待）
   C   ：单任务的平均**计算时间**
```

> **为什么这个公式是对的**：要让 CPU 利用率达到 `U_cpu`，就需要"在 CPU 干活的时间内，总能有线程可跑"。每个线程的生命周期是 `W + C`，其中只有 `C` 在吃 CPU。所以需要的线程数是 `(W+C)/C` 倍于"纯计算所需线程数"。

**举例：**

```
8 核，目标利用率 0.8
任务平均耗时 100ms，其中 CPU 计算 20ms，IO 等待 80ms
   → W/C = 80/20 = 4
   → N = 8 × (1 + 4) × 0.8 = 32

若 IO 等待增长到 380ms（下游变慢）：
   → W/C = 19 → N = 8 × 20 × 0.8 = 128  ← 线程数需求暴涨
   → 但 128 线程 × 1MB 栈 = 128MB，且可能压垮下游 DB
   → 这时正确做法不是"调大线程池"，而是"优化下游/加缓存/降级"
```

**③ 推导层（Little's Law，工程上最实用）**

```
Little's Law: L = λ × W

反过来求解线程数：
   在稳定状态下，同时在系统中的任务数 = 到达速率 × 停留时间
   而"同时执行的任务数"上限就是线程数 N

   N ≥ 峰值QPS × 单任务平均耗时(含IO)
   —— 这是**下限**（保证不排队累积）
   再乘以安全系数 1.2~1.5，并受下游资源约束封顶

例：峰值 500 QPS，单任务 80ms
   → N ≥ 500 × 0.08 = 40
   → 取 40 × 1.3 ≈ 52
   
但有下游约束：DB 连接池只有 20 个
   → 线程超过 20 只会在连接池上排队，毫无收益
   → 实际 N = 20（或 20 + 少量用于非 DB 任务）
```

> **这条"从下游倒推"的推理，是面试里最有区分度的答法**。绝大多数人只会背 `CPU+1` 和 `2N`，能说出"线程数上限应由下游资源（连接池、下游 QPS 配额、文件句柄）决定"的人，一看就是有真实生产经验。

### 7.2 四类任务的推荐配置

| 任务类型 | 特征 | `corePoolSize` | `maximumPoolSize` | 队列 | 拒绝策略 |
|---|---|---|---|---|---|
| **CPU 密集**（计算、序列化、加解密、图像处理） | `W/C` 很小 | `N_cpu + 1` | 同 core（不让它涨） | **有界小队列**（如 100~1000） | `Abort`（快失败）+ 上游限流 |
| **IO 密集**（RPC、DB、Redis、文件） | `W/C` 大 | `N_cpu × 2` ~ `N_cpu × 4` | core × 2（给突发余量） | 有界（1000~5000） | `CallerRuns`（背压）或 `Abort` + 降级 |
| **混合型** | 两者都有 | **拆分**成两个池 | — | — | — |
| **定时/延迟任务** | 时间驱动 | `N_cpu`（并行度） | 无意义（`Scheduled` 固定 MAX） | `DelayedWorkQueue` | `Abort` |

> **混合型一定要拆池**：这是最重要的一条实践。IO 任务把线程占满时，CPU 任务会被饿死；两者共用一个池，等于让快任务陪慢任务一起排队（**队头阻塞**）。

### 7.3 一组可直接照抄的"生产级"线程池定义

```java
@Bean("orderQueryPool")
public ThreadPoolExecutor orderQueryPool() {
    int cpu = Runtime.getRuntime().availableProcessors();
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
        cpu * 2,                              // core：IO 型起点
        cpu * 4,                              // max：突发上限（受 DB 连接池封顶，见下）
        60L, TimeUnit.SECONDS,                // 非核心线程 60s 回收
        new LinkedBlockingQueue<>(2000),      // ★ 有界！容量 = 峰值QPS × 可容忍排队
        new NamedThreadFactory("order-query"),// ★ 有意义的线程名
        new ThreadPoolExecutor.CallerRunsPolicy() // 背压
    );
    pool.allowCoreThreadTimeOut(true);        // 低峰期彻底缩容（可选）
    pool.prestartAllCoreThreads();            // 预热，避免流量突刺时"边建线程边处理"（可选）
    return pool;
}

// 关键：max 必须 ≤ 下游连接池大小，否则线程只是在连接池上排队
// 例：HikariCP maximumPoolSize=20 → 线程池 max 不应超过 20（对 DB 操作部分）
```

**`ThreadFactory` 一定要自定义的理由（生产必做）：**

```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger seq = new AtomicInteger(1);
    private final String prefix;
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + seq.getAndIncrement());
        t.setDaemon(true);                     // 不阻止 JVM 退出
        t.setUncaughtExceptionHandler((th, e) -> log.error("thread {} died", th.getName(), e));
        return t;
        // 进阶：用 InheritableThreadLocal 传递 traceId 时，注意池化会导致
        // "创建时继承一次"，后续任务拿不到新 traceId → 需要 TTL 包装（见 9.6）
    }
}
```

> **为什么线程名如此重要**：`jstack` 输出里线程名是唯一能定位"这个线程属于哪个业务池"的信息。默认名 `pool-1-thread-3` 在几十个池的环境里等于没有信息量。

### 7.4 一个"反常识"的结论

> **线程池调优，90% 的收益不来自"调参数"，而来自"改任务本身"**：
> - 把串行 RPC 改成并行/批量 → `W` 变小 → 所需线程数骤降
> - 加缓存 → `λ` 打到下游的部分变小
> - 缩短锁持有时间 → `W` 里的锁等待消失
>
> **调参只能在"任务特性不变"的前提下做局部最优；改任务形态则是换一条曲线。** 这句话在面试里说出来，比背公式值钱。

---
## 第 8 章 生命周期与关闭

### 8.1 `shutdown()` vs `shutdownNow()`：源码级差异

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();                    // 安全检查（SecurityManager 时代）
        advanceRunState(SHUTDOWN);                // ★ 状态推进到 SHUTDOWN
        interruptIdleWorkers();                   // ★ 只中断"空闲"线程
        onShutdown();                             // 钩子（ScheduledThreadPoolExecutor 用来取消定时任务）
    } finally { mainLock.unlock(); }
    tryTerminate();                               // 尝试推进终止
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);                    // ★ 直接推进到 STOP
        interruptIdleWorkers();                   // ★ 中断"所有"线程（含正在跑任务的）
        tasks = drainQueue();                     // ★ 把队列里剩余任务全部取出并返回
    } finally { mainLock.unlock(); }
    tryTerminate();
    return tasks;
}
```

**`advanceRunState` 的实现（一个"只许前进"的 CAS 循环）：**

```java
private void advanceRunState(int targetState) {
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||           // 已经达到或超过 → 不动
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))  // 否则 CAS 推进（保留线程数）
            break;
    }
}
// ★ 注意：只改高 3 位状态，低 29 位线程数原样保留（ctlOf 的组合）
// ★ "只许前进"保证不会从 STOP 退回 RUNNING
```

**`interruptIdleWorkers()` 的两种模式：**

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {   // ★ tryLock 成功 = 该线程空闲
                try { t.interrupt(); }
                catch (SecurityException ignore) { }
                finally { w.unlock(); }
            }
            if (onlyOne) break;                        // tryTerminate 用的"只叫一个"
        }
    } finally { mainLock.unlock(); }
}
```

> ⚠️ **注意 `shutdown()` 也调用了 `interruptIdleWorkers()`（不是 `shutdownNow` 才中断）**。区别在于：
> - `shutdown()`：`tryLock` **失败**（正在跑任务）的线程**不中断** → 正在执行的任务跑完再退出。
> - `shutdownNow()`：**也是**用 `interruptIdleWorkers()`……那"中断正在执行的任务"怎么实现的？
>
> **答案是：`shutdownNow` 把状态设成 STOP 后，`runWorker` 在"任务边界"重新检查状态并置中断位**（5.3 里那段绕口代码）。也就是说：
> - `shutdownNow` 对"正在跑的任务"的中断，**依赖于：①任务本身可响应中断（阻塞在可中断点）；②`runWorker` 在下一轮循环把中断位补上**。
> - **如果任务是个纯 CPU 死循环、完全不检查中断标志，`shutdownNow` 也停不掉它**。这是必须知道的语义边界——"Java 中断是协作式的，不是抢占式的"。

### 8.2 各种关闭方式的语义对照

| 方式 | 新任务 | 队列任务 | 执行中任务 | 返回值 | 是否阻塞等待 |
|---|---|---|---|---|---|
| `shutdown()` | 拒绝 | **跑完** | **跑完** | void | 否 |
| `shutdownNow()` | 拒绝 | **丢弃**（返回给你） | **尝试中断** | `List<Runnable>` 未执行任务 | 否 |
| `awaitTermination(t, unit)` | — | — | — | boolean | **是**（最多 t） |
| `shutdown() + awaitTermination(...)` | 拒绝 | 跑完 | 跑完 | boolean | **是** ← 优雅关闭标准写法 |
| `shutdownNow() + awaitTermination(...)` | 拒绝 | 丢弃 | 中断 | — | 是（强制关闭） |

**优雅停机标准写法（可直接用）：**

```java
public void gracefulShutdown(ExecutorService pool, long timeout, TimeUnit unit) {
    pool.shutdown();                                  // 1. 停止接新任务，通知空闲线程
    try {
        if (!pool.awaitTermination(timeout, unit)) {  // 2. 等待队列跑完
            List<Runnable> dropped = pool.shutdownNow(); // 3. 超时则强制中断
            log.warn("force shutdown, {} tasks dropped", dropped.size());
            if (!pool.awaitTermination(timeout, unit))    // 4. 再等一次
                log.error("pool did not terminate");      // 5. 仍然不退 → 只能记录告警
        }
    } catch (InterruptedException ie) {
        pool.shutdownNow();                           // 6. 自己被中断 → 立即强制
        Thread.currentThread().interrupt();            // ★ 恢复中断位（重要！）
    }
}
```

**Spring 环境下的落地方式：**

```java
@Component
public class PoolLifecycle implements DisposableBean {
    @Override
    public void destroy() throws Exception {
        // Spring 容器关闭时调用 → 在这里优雅关闭所有自建池
        gracefulShutdown(orderPool, 30, TimeUnit.SECONDS);
    }
}
// 注意：@PreDestroy 也可以，但要注意与 @Async 的 Executor 顺序
// 常见事故：容器先停了 DB 连接池，线程池里的任务再跑就报"连接已关闭"
```

**关闭时最容易被忽略的三件事：**

1. **关闭顺序**：先停"生产者"（MQ 消费者、HTTP 端口），再停线程池，最后停资源（DB/Redis）。**顺序错了会出现"任务还在跑，但依赖已经没了"的报错。**
2. **`awaitTermination` 必须给超时**：不给超时的 `awaitTermination(Long.MAX_VALUE, ...)` 会让停机流程永久挂住（任务里有死循环时常见）。
3. **`shutdown()` 之后再 `submit()` 会抛 `RejectedExecutionException`**——这是正确行为，但业务代码若在 finally 里提交"收尾任务"，停机时就会炸。**要用 `isShutdown()` 判断或 try-catch。**

---

## 第 9 章 生产实践：线程池的 12 个真实坑

> 这一章每一条都是"上线后真的会炸"的级别。按"现象 → 根因 → 修复"组织，可直接当 checklist 用。

### 9.1 坑 1：`Executors` 的四个工厂方法各有致命伤

| 方法 | 内部实现 | 致命伤 |
|---|---|---|
| `newFixedThreadPool(n)` | `core=max=n`, `LinkedBlockingQueue()` **无界** | 任务无限堆积 → **OOM**（`maximumPoolSize` 失效） |
| `newSingleThreadExecutor()` | `core=max=1`, `LinkedBlockingQueue()` **无界** | 同上，且**单线程**意味着一个任务卡死就全线卡死 |
| `newCachedThreadPool()` | `core=0`, **`max=Integer.MAX_VALUE`**, `SynchronousQueue` | 任务变慢时**无限建线程** → `unable to create native thread` / 打爆内核 |
| `newScheduledThreadPool(n)` | `max=Integer.MAX_VALUE`, `DelayedWorkQueue` | 队列**无界**（`DelayedWorkQueue` 可无限增长）+ 任务异常会**取消后续调度** |

**修复**：全部手写 `ThreadPoolExecutor`，显式指定**有界队列 + 线程名 + 拒绝策略**。

### 9.2 坑 2：`submit()` 静默吞掉异常（最高频事故）

```java
// 现象：任务失败了，日志里什么都没有，业务数据对不上
pool.submit(() -> {
    throw new RuntimeException("biz failed");   // ★ 完全静默
});

// 根因：submit 把任务包成 FutureTask，
//       FutureTask.run() 内部 catch Throwable 存进 outcome 字段，不 rethrow
//       异常要等你调用 future.get() 才会以 ExecutionException 抛出
//       如果你不 get（绝大多数人都不 get），异常就永久沉默

// 修复 1：用 execute() 而不是 submit()（异常会到 UncaughtExceptionHandler）
// 修复 2：submit 后必须 get()，或至少 .exceptionally(...) / whenComplete(...)
// 修复 3：自定义 ThreadPoolExecutor 重写 afterExecute，检测 Future 并 get
protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    if (t == null && r instanceof Future<?> && ((Future<?>) r).isDone()) {
        try { ((Future<?>) r).get(); }
        catch (CancellationException ce) { t = ce; }
        catch (ExecutionException ee) { t = ee.getCause(); }   // ★ 拿到真实异常
        catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
    }
    if (t != null) log.error("task failed", t);
}
```

### 9.3 坑 3：`ThreadLocal` 在线程池里不清理 → 内存泄漏 + 串数据

```
根因：
  ThreadLocal 的值存在 Thread.threadLocals（ThreadLocalMap）里，
  key 是 ThreadLocal 的弱引用，value 是强引用。

  线程池里线程不死 → Thread 对象不死 → threadLocals 表不死
   → 业务代码用完没 remove() → value 一直被强引用 → **内存泄漏**
   → 更糟：下一个任务复用同一线程，读到上一个任务的值 → **串数据（最可怕）**

现象：
  - 用户 A 的请求里读到了用户 B 的数据（权限、身份、租户 ID）
  - Full GC 后内存不降，dump 里一堆业务对象挂在 ThreadLocalMap 上

修复：
  - 在 finally 里 ThreadLocal.remove()（不是 set(null)！set(null) 仍保留 Entry）
  - 用线程池包装器统一清理（如 Spring 的 TaskDecorator）
  - 用阿里巴巴 TTL（TransmittableThreadLocal）并显式 TtlExecutors 包装
```

### 9.4 坑 4：`InheritableThreadLocal` 在池化之后失效

```
InheritableThreadLocal 的传递时机 = **子线程被创建的那一刻**（init() 时复制父的 map）

  任务 A（线程 T1 创建时继承 traceId=A）→ T1 被复用跑任务 B
   → 任务 B 拿到的仍是 traceId=A ❌（不是 B 的）

修复：
  TransmittableThreadLocal (TTL) + TtlRunnable/TtlExecutors 包装，
  在任务提交时"捕获"、执行前"回放"、执行后"恢复"。
  或改用 MDC + 显式传参，不依赖线程继承。
```

### 9.5 坑 5：线程池导致的 ClassLoader 泄漏（Web 容器热部署）

```
根因：线程池的 Worker 线程的 contextClassLoader 指向 Web 应用的 ClassLoader，
      容器热部署时旧应用的 ClassLoader 无法被回收（线程还活着并持有它）
   → Metaspace OOM / 每次热部署泄漏几十 MB

修复：容器关闭钩子里 shutdown 线程池；线程创建时显式设置 contextClassLoader。
```

### 9.6 坑 6：父子任务提交同一个池 → 死锁

```java
// 经典死锁：池大小 = 4，任务 A 提交 4 个任务 B 并等结果
pool = new ThreadPoolExecutor(4, 4, ..., queue(100));
pool.execute(() -> {
    Future<?> f1 = pool.submit(child);  // 4 个父任务同时各提交子任务
    Future<?> f2 = pool.submit(child);
    f1.get();  // ★ 4 个线程全在等子任务，子任务在队列里没人取 → 永久死锁
    f2.get();
});
根因：线程池没有"线程可以等待自己提交的任务"这种语义（不同于 ForkJoinPool 的
      work-stealing 有 join 特殊处理）

修复：
  - 父子任务用不同的池（父池小、子池大）
  - 用 ForkJoinPool（它实现了 work-stealing + 补偿线程）
  - 或者彻底异步化，不阻塞等待
```

### 9.7 坑 7：队列积压到"算了也白算"

```
现象：任务都执行成功了，但业务方投诉"数据延迟 10 分钟"

根因：队列长度 5000，单任务 50ms，8 线程 → 队列排满时最坏等待
      5000 / 8 × 50ms ≈ 31 秒（还只是队列，不含执行）
      实际因为 GC/下游抖动，可能变成几分钟

修复：
  - 队列容量按"可容忍延迟"倒推（见 6.2）
  - 任务入队前打时间戳，执行时检查"已排队多久"，超过 SLA 直接丢弃并记录
    if (System.currentTimeMillis() - submitTime > 5000) { drop(); return; }
  - 这就是所谓的"过期任务快速失败"（fail-fast on stale）
```

### 9.8 坑 8：共用线程池导致故障扩散

```
现象：一个非核心业务（如报表导出）慢查询，把整个系统的下单接口拖垮

根因：所有业务共用一个池 → 慢任务占满线程 → 快任务排队 → 全线雪崩
      （"故障隔离"缺失）

修复：**按业务/按依赖分组，物理隔离线程池**
  - 核心交易池、非核心异步池、定时任务池 各自独立
  - 每个池独立配置 + 独立监控 + 独立告警
  - 这就是"舱壁模式（bulkhead pattern）"
```

### 9.9 坑 9：`setCorePoolSize` 动态调参的语义陷阱

```java
// 源码逻辑（简化）：
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0) throw new IllegalArgumentException();
    int delta = corePoolSize - this.corePoolSize;
    this.corePoolSize = corePoolSize;          // ★ 先赋值
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();                // 调小 → 中断空闲线程让它退出
    else if (delta > 0) {
        int k = Math.min(delta, workQueue.size());
        while (k-- > 0 && addWorker(null, true)) {   // 调大 → 立刻补线程（最多补到队列长度）
            if (workQueue.isEmpty()) break;
        }
    }
}
```

**三个陷阱：**

1. **调小不会立刻减线程**：只是"中断空闲线程"，正在跑任务的线程要等任务结束、`getTask` 重新判断时才退出。所以**线程数是"渐进下降"的**。
2. **调大只在队列有任务时才立刻补线程**（`k = min(delta, queue.size())`），队列空时不会预建。想立刻生效要配合 `prestartAllCoreThreads()`。
3. **`setMaximumPoolSize` 调小同样不会立刻杀线程**，靠 `getTask` 里 `wc > maximumPoolSize` 那条分支自然淘汰。

**动态调参的正确用法（配合配置中心）：**

```
Nacos/Apollo 推送 params → listener 回调 → 依次调用
  setCorePoolSize / setMaximumPoolSize / setKeepAliveTime / setRejectedExecutionHandler
★ 注意顺序：**必须先调大 max，再调大 core**；缩小时反过来。
  否则中间态会出现 core > max 的非法组合（构造时会抛异常，setter 不校验但语义错误）
```

### 9.10 坑 10：`CallerRunsPolicy` 卡死 IO 线程

```
现象：拒绝策略设成 CallerRunsPolicy，结果 Netty 的 EventLoop 线程被拖去跑
      50ms 的同步任务 → 整个连接的读写都停滞 → 超时雪崩

根因：CallerRunsPolicy 在"提交者的线程"里同步执行任务。
      如果提交者是 IO 线程 / 定时器线程 / 单线程调度器 → 直接把它堵死

修复：IO 线程上永远不要用 CallerRunsPolicy；改用 Abort + 降级，
      或把任务投递到另一个"可阻塞"的队列（如 Disruptor / MQ）。
```

### 9.11 坑 11：没有监控 = 事故时瞎猜

**必须采集的 6 个指标：**

| 指标 | 取法 | 告警阈值建议 |
|---|---|---|
| `poolSize` | `getPoolSize()` | 持续 == max → 已打满 |
| `activeCount` | `getActiveCount()` | 持续 == max → 线程全忙 |
| `queueSize` | `getQueue().size()` | 持续增长 / > 容量 80% → 处理不过来 |
| `completedTaskCount` | `getCompletedTaskCount()` | 斜率下降 → 吞吐退化 |
| **拒绝次数** | 自定义 handler 计数 | 任何非 0 → 立即告警 |
| **任务耗时分布** | `beforeExecute/afterExecute` 埋点 | P99 > SLA → 告警 |

**扩展性监控（用 `ThreadPoolExecutor` 的钩子零成本实现）：**

```java
public class MonitorablePool extends ThreadPoolExecutor {
    private final Timer taskTimer = ...;   // 如 Micrometer Timer

    @Override protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        startTime.set(System.nanoTime());   // 用 ThreadLocal 存更严谨
    }
    @Override protected void afterExecute(Runnable r, Throwable t) {
        taskTimer.record(System.nanoTime() - startTime.get());
        if (t != null) failureCounter.increment();
        super.afterExecute(r, t);
    }
}
```

### 9.12 坑 12：`ScheduledThreadPoolExecutor` 的三个隐藏行为

```
① 任务抛异常 → 该任务被取消，后续调度不再发生（静默消失！）
   修复：任务内部 try-catch 全部异常

② scheduleAtFixedRate 是"固定频率"，若任务耗时 > 周期，会连续补跑（堆积）
   scheduleWithFixedDelay 是"固定延迟"，上一次结束才计时 → 更安全
   ★ 千万别在 scheduleAtFixedRate 里放慢任务

③ 它的 maximumPoolSize 被硬编码为 Integer.MAX_VALUE，
   corePoolSize 才是真正的并行度（默认 1！newScheduledThreadPool(1) 就是单线程）
```

---
## 第 10 章 国内大厂面试题全解（45 题）

### 说明：考核程度标注规则

| 标记 | 含义 | 面试中的定位 |
|---|---|---|
| ★★★★★ | **必考**。说不出来直接扣分 | 一线大厂 P6/2-1/3-1 起步题 |
| ★★★★ | **高频**。80% 的面试会问到 | 区分"会用"和"懂原理" |
| ★★★ | **常考**。中高级岗位常见 | 区分"看过源码" |
| ★★ | **进阶**。架构/专家岗会问 | 加分项 |
| ★ | **冷门/深水**。答对很亮眼 | 拉开差距 |

**厂标签说明**：阿=阿里巴巴，字=字节跳动，腾=腾讯，美=美团，京=京东，快=快手，拼=拼多多，百=百度，网=网易，滴=滴滴，华=华为，蚂=蚂蚁。

---

## A 组 · 基础与参数（Q1–Q12）

### Q1. 线程池的七大核心参数是什么？请完整描述一个任务提交后的执行流程。
> **考核程度：★★★★★ 必考** · 高频：阿 / 字 / 腾 / 美 / 京 / 拼

**考点**：参数记忆 + 三步流程 + 边界条件。

**参考答案（分两层作答）**：

**第一层：参数（必须一字不差地说出 7 个）**

| 参数 | 一句话定义 |
|---|---|
| `corePoolSize` | 常驻核心线程数（默认不销毁） |
| `maximumPoolSize` | 线程总数硬上限（只在队列满时才会向这里扩） |
| `keepAliveTime` | 超出 core 的线程空闲多久被回收（`allowCoreThreadTimeOut` 为 true 时对 core 也生效） |
| `unit` | 时间单位 |
| `workQueue` | 任务阻塞队列（缓冲能力 = 允许的排队延迟） |
| `threadFactory` | 线程工厂（命名、守护、异常处理器） |
| `handler` | 饱和/拒绝策略 |

**第二层：执行流程（三步 + 三处边界）**

```
1) 线程数 < corePoolSize
   → addWorker(task, true) 新建核心线程直接执行
   → 失败则重读 ctl 继续往下

2) 线程数 >= corePoolSize
   → isRunning && workQueue.offer(task) 入队
   → 入队成功后 recheck：
       a. 池已关闭 → remove(task) 成功则 reject
       b. 线程数为 0 → addWorker(null, false) 补一个线程
       c. 否则正常等待空闲线程消费

3) 入队失败（队列满 / 非 RUNNING）
   → addWorker(task, false) 扩容至 maximumPoolSize
   → 仍失败 → reject(task) 走拒绝策略
```

**加分项（说出这三条，就和普通候选人拉开了距离）**：

1. **核心线程默认是"懒创建"的**，`new` 完池子一个线程都没有，要 `prestartCoreThread`/`prestartAllCoreThreads` 才预热。
2. **`maximumPoolSize` 在无界队列下永远不会生效**，因为第 3 步的 `offer` 永远成功。
3. **第 2 步的 `recheck` 不是防御性冗余**，它真实修复两类竞态（入队后被关闭、入队后线程归零），尤其 `corePoolSize=0`（缓存型池）时是常态路径。

**延伸**：`execute` 的三步本质是一条**优先级链**：`核心线程 → 排队 → 扩容 → 拒绝`。Doug Lea 的设计原则是**"宁可多起线程，也不把任务塞进可能无人消费的队列"**，因为前者是可回收的资源浪费，后者是正确性事故。

---

### Q2. `execute()` 和 `submit()` 有什么区别？为什么说 `submit` 会"吞异常"？
> **考核程度：★★★★★ 必考** · 高频：字 / 美 / 京 / 滴

**考点**：`AbstractExecutorService` 的包装逻辑 + `FutureTask` 的异常处理 + 线程存活语义。

**参考答案**：

```java
// AbstractExecutorService.submit 的本质
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);   // → new FutureTask<>(task)
    execute(ftask);                                // ★ 最终还是走 execute
    return ftask;
}
```

**差异表（这是答案的骨架）：**

| 维度 | `execute(Runnable)` | `submit(Callable/Runnable)` |
|---|---|---|
| 返回值 | `void` | `Future<T>` |
| 任务包装 | 不包装，直接跑 | 包成 `FutureTask` |
| 异常去向 | **抛出到线程顶层** → 线程 `TERMINATED` → `UncaughtExceptionHandler` 处理 | **被 `FutureTask` 捕获存进 `outcome`** → 只有 `get()` 才抛出 |
| 对线程的影响 | 异常 → **线程死掉**（但池会补一个新线程） | 异常 → **线程不死**，正常回到循环 |
| 能否取消 | 不能 | 能（`future.cancel(true)`） |
| 能否拿结果 | 不能 | 能 |
| 能否感知完成 | 不能 | 能（`isDone` / `get` / `whenComplete`） |

**"吞异常"的源码级原因**：

```java
// FutureTask.run()
public void run() {
    if (state != NEW || !RUNNER.compareAndSet(this, null, Thread.currentThread())) return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {          // ★★★ 所有异常在这里被捕获
                result = null;
                ran = false;
                setException(ex);              // ★ 存进 outcome 字段，不 rethrow
            }
            if (ran) set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING) handlePossibleCancellationInterrupt(s);
    }
}
```

**所以"吞异常"不是 Bug，是 `Future` 语义的必然结果**——异常要等到你去 `get()` 时才以 `ExecutionException` 抛出（`getCause()` 拿真实异常）。问题在于**绝大多数业务代码 `submit` 完从不 `get()`**，异常就永久沉默。

**修复方案（说出三种以上算优秀）：**

1. 非必需返回值 → 用 `execute()`。
2. `submit` 后必须消费：`future.get()` / `.exceptionally()` / `.whenComplete()`。
3. 全局兜底：自定义 `ThreadPoolExecutor` 重写 `afterExecute`，若 `r instanceof Future` 且已完成则 `get()` 并记日志。
4. 用 `CompletableFuture` 并统一挂 `.exceptionally(log::error)`。

**延伸（面试官最爱追问）**："`execute` 的异常会让线程死掉，那线程池的线程数会不会越来越少？"
→ **不会**。`runWorker` 的 `finally` 里 `processWorkerExit(w, completedAbruptly=true)` 会 `decrementWorkerCount()` 后**补一个 `addWorker(null,false)`**。所以线程会"换一个身子"继续服务，池容量不缩水。**这个细节能答出来，说明真读过源码。**

---

### Q3. 阿里巴巴 Java 开发手册为什么禁止用 `Executors` 创建线程池？
> **考核程度：★★★★★ 必考** · 高频：阿 / 字 / 美 / 京 / 快

**考点**：工厂方法背后的隐藏参数 + OOM 成因 + 生产约束。

**参考答案（用一张表直接击穿）：**

| 工厂方法 | 真实参数 | 隐患 |
|---|---|---|
| `newFixedThreadPool(n)` | core=max=n，**`LinkedBlockingQueue()` 无界（容量 `Integer.MAX_VALUE`）** | 任务堆积到内存耗尽 → `OutOfMemoryError: Java heap space`。且 `max` 形同虚设 |
| `newSingleThreadExecutor()` | core=max=1，**无界队列** | 同上；且单线程 → 一个任务卡死全线阻塞，毫无隔离 |
| `newCachedThreadPool()` | core=0，**max=`Integer.MAX_VALUE`**，`SynchronousQueue` | 任务变慢时无限建线程 → `OutOfMemoryError: unable to create native thread`（撞 `max_map_count` / `pids.max`） |
| `newScheduledThreadPool(n)` | max=`Integer.MAX_VALUE`，`DelayedWorkQueue`（**无界**） | 队列无限增长；且**任务抛异常会导致后续调度被静默取消** |

**根因总结（三句话）：**

1. **队列无界** → 失去"背压"，把过载变成"内存增长"，最终 OOM 而不是快速失败。
2. **线程数无界** → 失去"并发保护"，把过载转嫁给内核（`clone` 风暴 + 调度爆炸）。
3. **参数不可控** → 无法指定线程名、无法定制拒绝策略 → **出事故时既查不到也兜不住**。

**正面回答手册要求**："必须用 `ThreadPoolExecutor` 显式指定，并**推荐使用有界队列 + 明确的拒绝策略 + 可识别的线程名**。"

**延伸追问："那 `CachedThreadPool` 在什么场景下反而是对的？"**
→ 大量**极短、无阻塞、可丢失**的异步任务（如日志写入、指标上报），且**下游能承受瞬时线程数增长**。但即便如此，也应该自己写 `new ThreadPoolExecutor(0, 200, 60s, new SynchronousQueue<>())`，把 `max` 封顶。

---

### Q4. 线程池有哪几种状态？状态之间怎么转换？为什么需要 `TIDYING`？
> **考核程度：★★★★ 高频** · 高频：阿 / 腾 / 美

**参考答案**：

```
RUNNING → SHUTDOWN → TIDYING → TERMINATED      （温和关闭路径）
RUNNING → STOP      → TIDYING → TERMINATED      （强制关闭路径）
```

| 状态 | 高3位 | 语义 | 新任务 | 队列任务 | 执行中任务 |
|---|---|---|---|---|---|
| `RUNNING` | 111 | 正常 | 接受 | 执行 | 不打断 |
| `SHUTDOWN` | 000 | 温和关门 | 拒绝 | **跑完** | 不打断 |
| `STOP` | 001 | 强制关门 | 拒绝 | **丢弃** | **中断** |
| `TIDYING` | 010 | 收尾中（workerCount=0） | 拒绝 | 空 | 无 |
| `TERMINATED` | 011 | 已终止（`terminated()` 已执行） | 拒绝 | 空 | 无 |

**转换触发点：**

- `RUNNING → SHUTDOWN`：`shutdown()`
- `RUNNING → STOP`：`shutdownNow()`
- `SHUTDOWN/STOP → TIDYING`：`tryTerminate()` 中，**队列空 && workerCount == 0** 时 CAS 成功
- `TIDYING → TERMINATED`：执行完 `terminated()` 钩子后 `ctl.set(...)`

**为什么必须有 `TIDYING`（这是本题的核心）**：

`terminated()` 是留给子类重写的钩子（`ScheduledThreadPoolExecutor` 在里面取消所有定时任务、清空队列）。**必须有一个"线程已全部退出、但钩子还没跑"的稳定窗口**，钩子才能在无并发干扰的情况下安全执行，并且它抛异常也不会影响状态机的最终收敛（`finally` 里无条件置 `TERMINATED`）。

**没有 TIDYING 会怎样**：
- 无法区分"还有线程在收尾"和"真的结束了"，`awaitTermination` 可能提前返回；
- 钩子可能在仍有多线程活动的状态下执行 → 竞态；
- 状态机也无法表达"正在执行收尾动作"这一中间语义。

**状态值设计的巧思**：`RUNNING=-1<<29` 是**负数**，而 `SHUTDOWN/STOP/TIDYING/TERMINATED` 递增为正数。因此 `c < SHUTDOWN` 即"在 RUNNING"，`c >= STOP` 即"至少 STOP"。**用整数的自然大小关系编码了状态的层级关系**，省去枚举比较。

---

### Q5. `ctl` 为什么用一个 `AtomicInteger` 同时存"状态"和"线程数"？拆成两个变量不行吗？
> **考核程度：★★★★★ 必考（源码深度题）** · 高频：阿 / 字 / 腾

**参考答案**：

**为什么合体（这是唯一正确的理由）——原子性：**

线程池有两类复合操作必须"同时看到两个值"：

```
① "线程数减到 0 了，现在该不该进入 TIDYING？"
      —— 减计数 与 改状态 必须一起决定
② "状态是 RUNNING 且线程数 < core，可以加线程"
      —— 判状态 与 判计数 必须基于同一快照
```

如果拆成两个 `AtomicInteger`：

```
线程 A：wc.decrementAndGet() → 0        ┐ 中间态：wc=0 但 state 仍是 RUNNING
线程 B：读 state=RUNNING, wc=0          ┘ → 误判：既不是"该终止"，也不是"该加线程"
        → 出现"卡在 RUNNING 且无任何线程"的僵死状态
```

合体后，**一次 CAS 同时改变"状态 + 计数"**，不存在中间态。

**为什么用 `int` 而不是 `long`：**
`AtomicInteger` 的 CAS 在 32/64 位平台都是**单条指令**（`lock cmpxchg`）；`AtomicLong` 在 32 位平台需要 `cmpxchg8b`（总线锁，更慢）。29 位线程数上限 5.36 亿，远超实际需求，**没必要为容量付 64 位 CAS 的代价**。

**位运算的三个函数（现场必须能写出来）：**

```java
private static final int COUNT_BITS = Integer.SIZE - 3;   // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

static int runStateOf(int c)    { return c & ~CAPACITY; }  // 高 3 位
static int workerCountOf(int c) { return c &  CAPACITY; }  // 低 29 位
static int ctlOf(int rs, int wc){ return rs | wc; }        // 合并
```

**延伸追问："29 位真的够吗？如果线程数超了会怎样？"**
→ `addWorker` 里有 `if (wc >= CAPACITY) return false;`，即**超限时拒绝创建**，不会溢出污染状态位。实际上撞到 5 亿线程前，早就撞到 `vm.max_map_count` 了（见第 2.9 节）。

---

### Q6. 线程池是怎么实现"线程复用"的？一个任务执行完，线程去哪了？
> **考核程度：★★★★★ 必考（最本质的一题）** · 高频：阿 / 字 / 腾 / 美 / 京 / 拼 / 快

**参考答案（答案的主体是 4 行代码）**：

```java
// runWorker(Worker w) 的核心：while 循环
while (task != null || (task = getTask()) != null) {
    w.lock();
    try { task.run(); }        // ← 业务任务只是循环体的一次迭代
    finally { task = null; w.completedTasks++; w.unlock(); }
}
```

**完整链路（这才是"复用"的全貌）：**

```
① ThreadFactory.newThread(worker) —— 线程只被创建一次
         ↓
② Worker implements Runnable，thread.run() → runWorker(this)
         ↓
③ runWorker 进入 while 循环
         ↓
④ task = firstTask（首次），非空 → task.run() 执行业务代码
         ↓
⑤ firstTask 置 null；unlock()（标记空闲）
         ↓
⑥ 下一轮：getTask() → workQueue.take() → **阻塞在 futex 上，不占 CPU**
         ↓
⑦ 新任务到达 → 队列 signal → 线程被唤醒 → 拿到新 task
         ↓
⑧ 回到 ④，但这次 task 来自队列，不是 firstTask
         ↓
⑨ getTask() 返回 null（池关闭 / 超时 / 缩容）→ while 结束 → processWorkerExit → 线程才真正死亡
```

**三条要点（能说出这三条才算理解）：**

1. **复用的是"内核线程"**，不是"Java 对象"。线程的 `pthread_t`、内核栈、用户栈在整个生命周期内都是同一份（见第 2、3 章）。
2. **任务与线程解耦的关键是 `Worker`**：`Worker.thread` 的 run 方法是固定的 `runWorker`，业务任务只是"被喂进去的数据"。
3. **任务结束 ≠ 线程结束**：只有 `getTask()` 返回 null 才终结。这个条件由"池状态 / 超时 / 缩容"三者之一触发。

**加分延伸**：为什么不是"线程池自己维护一个任务队列 + 用信号量唤醒"？
→ 实际上就是这样，只是用了 `BlockingQueue` 而不是裸的信号量：**队列本身就承担了"任务存储 + 阻塞/唤醒"两件事**，复用了 `BlockingQueue` 的成熟实现（futex 级的 park/unpark）。如果自己用 `Semaphore + 数组`，要处理可见性、公平性、超时，成本更高。

---

### Q7. 核心线程会一直存活吗？`allowCoreThreadTimeOut` 做了什么？
> **考核程度：★★★★ 高频** · 高频：美 / 京 / 滴 / 网

**参考答案**：

**默认行为：核心线程"几乎"永久存活**，但有三个例外：

| 例外 | 触发条件 |
|---|---|
| ① 池被关闭 | `shutdown()` / `shutdownNow()` |
| ② 任务抛异常 | 该线程死掉（但会被补一个新线程，**线程身份变了**） |
| ③ `allowCoreThreadTimeOut(true)` | 核心线程也按 `keepAliveTime` 回收 |

**`allowCoreThreadTimeOut` 的源码联动（关键在两处）：**

```java
public void allowCoreThreadTimeOut(boolean value) {
    if (value && keepAliveTime <= 0)
        throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
    if (value != allowCoreThreadTimeOut) {
        allowCoreThreadTimeOut = value;
        if (value) interruptIdleWorkers();   // ★ 立刻让空闲核心线程开始计时/退出
    }
}
```

真正的生效点在 `getTask()`：

```java
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
//       ↑ 为 true 时，即使 wc <= core，也会走 poll(keepAliveTime) 而不是 take()
Runnable r = timed ? workQueue.poll(keepAliveTime, NANOSECONDS) : workQueue.take();
```

**副作用（面试加分点）：**

1. **`allowCoreThreadTimeOut(true)` 后，池可能缩到 0 线程**。此时 `execute` 的第 2 步 `recheck` 的 `wc == 0` 分支变成常态——**每次任务都要"入队 + 新建线程"**，吞吐下降（多一次 `clone`）。
2. 因此官方建议：**要么完全预热（`prestartAllCoreThreads`）+ 不允许超时**，要么**core=0 + 允许超时**（纯弹性模型），**别混着用**。

**延伸**：为什么 `keepAliveTime` 必须 > 0 才允许超时？因为 `poll(0)` 会立刻返回 null，线程会瞬间全部退出，池子功能失效——构造函数直接禁止这个组合，属于"用校验代替运行时踩坑"。

---

### Q8. 线程池里的线程是在 `new ThreadPoolExecutor` 时就创建好的吗？`prestartAllCoreThreads` 有什么用？
> **考核程度：★★★ 常考** · 高频：京 / 滴 / 网

**参考答案**：

**不是。核心线程是"懒创建"（lazy）的。**

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(8, 8, 60, SECONDS, queue);
// 此刻 pool.getPoolSize() == 0，一个线程都没起
// 第一次 execute() 才会按需创建，且是"来一个任务建一个线程"（最多到 core）
```

**两个预热方法：**

```java
public boolean prestartCoreThread() {
    return workerCountOf(ctl.get()) < corePoolSize && addWorker(null, true);
}
public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true)) ++n;   // 循环直到加满 core
    return n;
}
```

**注意：预热创建的是 `firstTask == null` 的 Worker**，它们启动后立刻进 `getTask()`，用 `take()` 阻塞等任务——**不占 CPU，但栈和内核对象已就位**。

**为什么需要预热（三条真实理由）：**

| 理由 | 说明 |
|---|---|
| **消除首波延迟** | 不预热时，第一批请求要承担 `clone()` 的 30~80 μs，还要排队等调度，P99 会出现明显尖刺 |
| **失败前置** | 线程创建失败（OOM / `max_map_count` 满）在启动时就暴露，而不是在流量高峰时炸 |
| **`allowCoreThreadTimeOut` 的配合** | 若开启超时，池会缩到 0；配合定时"巡检式预热"能保持弹性与常驻的平衡 |

**代价**：`core × 1MB` 的虚拟地址常驻 + `core` 份内核对象。**8~32 个核心线程的池，预热收益远大于成本；64 个以上的池要谨慎。**

**延伸追问："预热之后线程会一直占用 CPU 吗？"**
→ 不会。它们 `take()` 阻塞在 futex 上，**CPU 占用为 0**。这就是线程池"容量大但不吃 CPU"的物理基础（见 2.8 节）。

---

### Q9. `execute()` 里为什么是"先加线程、再入队"，而不是"先入队、再考虑加线程"？
> **考核程度：★★★★ 高频（源码理解分水岭）** · 高频：阿 / 字 / 腾

**参考答案（三层）**：

**第一层：如果先入队会怎样（反证法）**

```
假设顺序改为：workQueue.offer(task) 成功 → 再 addWorker(task, true)

场景：core=4, max=4, queue容量=1
  T0  队列空
  T1  offer(taskA) 成功
  T2  addWorker(A, true) —— 此刻恰好有 4 个核心线程（被别人并发了）
      → 返回 false，任务 A 已经在队列里，但 A 没人执行
  T3  外层逻辑拿不到"A 已入队"这个信息 → 不会走扩容/拒绝
      → **A 永久卡在队列里（除非后续有任务触发 recheck 的 wc==0 分支）**
```

**先加线程的顺序则天然安全**：`addWorker` 失败 → 直接落到"入队"分支；入队失败 → 落到"扩容"分支；扩容失败 → 拒绝。**每一步失败都有明确的下一步，任务永远有归宿。**

**第二层：设计哲学**

> Doug Lea 的原则：**"宁可多创建线程（可回收的资源浪费），也绝不把任务塞进一个可能无人消费的队列（正确性事故）。"**

线程多余 → 最终会被 `keepAliveTime` 回收，或立即在 `addWorker` 的 CAS 容量检查里被拒绝；任务滞留 → 业务超时、数据不一致，是事故。

**第三层：性能考量**

`addWorker` 成功时，任务**立即开始执行**，无需经过队列的入队/出队（两次锁操作 + 两次 `Condition.signal`）。**核心路径上少一次队列往返**，对低延迟场景有意义。

**延伸追问："那为什么第 2 步入队后还要 `recheck`？这不就绕回去了吗？"**
→ 不绕。`recheck` 处理的是**另一类竞态**（入队与"池关闭/线程归零"之间的时间窗），不是为"先加线程"的设计做补丁。两者的关注点不同：前者保证"入队动作是原子有效的"，后者保证"任务有线程消费"。

---

### Q10. `getTask()` 什么时候返回 null？返回 null 之后发生了什么？
> **考核程度：★★★★ 高频（源码必问）** · 高频：阿 / 字 / 美

**参考答案**：

**三条路径，全部返回 `null`：**

```java
// ① 池状态不允许继续
if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
    decrementWorkerCount(); return null;
}
//   STOP 及之后 → 无论队列是否有任务都退出（队列任务被丢弃）
//   SHUTDOWN     → 只有队列空了才退出（"温和关闭要跑完队列"就在这里实现）

// ② 该收缩了
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
if ((wc > maximumPoolSize || (timed && timedOut))
    && (wc > 1 || workQueue.isEmpty())) {
    if (compareAndDecrementWorkerCount(c)) return null;
    continue;
}
//   wc > maximumPoolSize  → 池被 setMaximumPoolSize 调小了，需要裁线程
//   timed && timedOut     → poll 超时了，这个线程该退休
//   ★ (wc > 1 || queue.isEmpty()) 是保底：只剩我一个且队列有活时，我不能走

// ③ poll 超时（这是 ② 的输入）
Runnable r = timed ? workQueue.poll(keepAliveTime, NANOSECONDS) : workQueue.take();
if (r != null) return r;
timedOut = true;   // ← 不立刻退出，标记后回到循环顶部统一判断
```

**返回 null 之后（线程退休的完整动作）：**

```
getTask() == null
   ↓ runWorker 的 while 结束，completedAbruptly = false
   ↓ finally → processWorkerExit(w, false)
        ├─ （不递减计数，因为 getTask 已经减了）★ 这是最容易搞错的地方
        ├─ mainLock.lock() → completedTaskCount += w.completedTasks; workers.remove(w)
        ├─ tryTerminate()            ← 推进状态机
        └─ 判断是否要"补线程"（正常退休时通常不补）
   ↓ 线程从 runWorker 返回 → Thread.run() 结束 → 线程 TERMINATED → 内核 task 回收
```

**加分延伸（两个坑）：**

1. **为什么 `timedOut` 要用变量跨循环，而不是 `poll` 返回 null 就直接 `return null`？**
   因为 poll 超时的瞬间，池状态可能刚变成 SHUTDOWN，或队列刚进来任务（虽然 `poll` 会返回 null 但下一轮 `take` 前又有了）。**统一在循环顶部做判断，才能保证"状态、容量、队列"三者被一致地评估**。
2. **`catch (InterruptedException) { timedOut = false; }` 为什么要"吞掉"异常？**
   中断在这里表示"有人叫我（shutdown/缩容）"，但我**不能直接退出**——如果是 SHUTDOWN 且队列非空，我应该继续干活。所以要回到循环顶部**重新判断状态**。`timedOut=false` 是防止把"被中断"误判成"超时"。

---

### Q11. `Worker` 为什么继承 `AbstractQueuedSynchronizer`？为什么不用 `ReentrantLock`？
> **考核程度：★★★★★ 必考（深度区分题）** · 高频：阿 / 字 / 腾 / 美

**参考答案（三层递进，缺一层就不算满分）：**

**第一层：它需要一个"忙/闲标记"，而不是"临界区保护"**

```java
// Worker 里的锁，真正的用途：
w.lock();      // ← 标记"我正在执行任务"
task.run();
w.unlock();    // ← 标记"我空闲了"

// 外部通过 tryLock 判定：
shutdown() → interruptIdleWorkers() → if (w.tryLock()) { /* 空闲，可以中断 */ }
```

**这是"用锁状态承载线程状态"**：`isLocked()` 与 `正在执行任务` 严格同生命周期，天然不会出现"标记了 busy 但其实空闲"的不一致（而额外维护一个 `volatile boolean busy` 就可能漏改、错改）。

**第二层：不可重入是刻意的**

`ReentrantLock` 的语义是"同一线程可重复获取"，会**累积 hold count**。而 `Worker` 需要的是"**只有 0 和 1 两个状态**"：

| 需求 | 可重入会怎样 |
|---|---|
| 精确判断"是否正在执行任务" | 重入后 `isLocked()` 仍为 true，但 hold count > 1，语义模糊 |
| `shutdown` 时区分"忙/闲" | 外部线程 `tryLock` 本来就拿不到（跨线程不可重入），但**代码内部的可重入会让状态维护更复杂** |
| 不允许用户代码解锁 | 用户若在 `beforeExecute`/`afterExecute`/`task` 里误调 `unlock()`，`ReentrantLock` 会真的释放锁 → 忙闲标记被破坏 |

**第三层：Doug Lea 的原始注释（说得出这段，直接封神）**

> *"We implement a simple non-reentrant mutual exclusion lock rather than use `ReentrantLock` because we do not want user code to be able to call `unlock()` ... and because it avoids needing to track a lock count."*

翻成中文：**① 不希望用户代码有机会调用 `unlock()` 破坏忙闲标记；② 省掉重入计数的开销（每个 Worker 少几字节 × 数百线程，且少一层 `Sync` 对象）。**

**第四层（附加加分）：`setState(-1)` 的意义**

```java
Worker(Runnable firstTask) {
    setState(-1);      // ★ 初始为 -1：禁止中断
    ...
}
// runWorker 第一步：w.unlock() → state 变为 0 → "允许中断"
```

三种 state 语义：

| state | 含义 | `interruptIfStarted()` 是否中断 |
|---|---|---|
| `-1` | 刚创建，线程还没 `start()` | ❌ 跳过（中断尚未启动的线程无意义且危险） |
| `0` | 空闲（阻塞在 `getTask`） | ✅ |
| `1` | 正在执行任务 | ✅（`shutdownNow` 会，`shutdown` 不会） |

**`-1` 是一个"生命开关"**：它挡住了"`shutdownNow` 遍历 workers 时，误中断一个还没 `t.start()` 的线程"这个真实竞态（因为 `workers.add(w)` 在 `t.start()` 之前）。

---

### Q12. `shutdown()` 怎么知道哪些线程"空闲"、哪些"在忙"？
> **考核程度：★★★★ 高频（Q11 的实战化）** · 高频：阿 / 腾 / 美

**参考答案**：

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {   // ★ 判定点
                try { t.interrupt(); }
                catch (SecurityException ignore) { }
                finally { w.unlock(); }                 // 探测完立刻还回去
            }
            if (onlyOne) break;
        }
    } finally { mainLock.unlock(); }
}
```

**判定逻辑：**

```
w.tryLock() 成功  ⇔ state 从 0 CAS 到 1 成功 ⇔ 该 Worker 没有持锁
                                     ⇔ 它不在 task.run() 里 ⇔ **空闲**
                  → 它在 workQueue.take()/poll() 上阻塞 → interrupt() 能立刻让它醒来

w.tryLock() 失败  ⇔ state 已是 1 ⇔ 它正在执行任务 ⇔ **忙**
                  → shutdown() 跳过它（让它跑完）
                  → shutdownNow() 依赖 runWorker 在"任务边界"补置中断位
```

**三个精妙处：**

1. **探测完立刻 `unlock()`**：`tryLock` 的目的是"读状态"，不是"占住"。占住会让紧接着的任务执行卡住，反而制造延迟。
2. **`!t.isInterrupted()` 前置检查**：避免重复中断（`interrupt()` 对已中断线程是无害的，但少一次内核相关操作）。
3. **`mainLock` 保护 `workers` 集合**：遍历时集合可能被 `processWorkerExit` 修改，必须持锁（`HashSet` 非并发安全）。

**延伸追问**："如果线程恰好在这两个瞬间之间切换状态怎么办？"
→ 存在**极窄的竞态窗口**：`tryLock` 成功 → 该线程刚拿到任务（`state` 尚未置 1）→ `interrupt()` 打在了"刚进入 `task.run()`"的线程上 → **任务会被中断**。
缓解：`runWorker` 在 `task.run()` 之前有"中断位清理"逻辑（见 Q13），且业务任务本身应当是可响应中断的。**这是设计上接受的权衡：宁可偶尔多中断一个刚开始的任务，也不能漏掉一个空闲线程导致 `shutdown` 卡住。**

---
## B 组 · 源码与并发设计（Q13–Q24）

### Q13. `runWorker()` 里那段"清理中断位"的代码到底在干什么？为什么要写得这么绕？
> **考核程度：★★★★★ 必考（源码最难点）** · 高频：阿 / 字 / 腾

```java
if ((runStateAtLeast(ctl.get(), STOP) ||
     (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
    && !wt.isInterrupted())
    wt.interrupt();
```

**参考答案**：

**先翻译成人话：**

> "如果池已经进入 STOP（或更后）状态；**或者**我刚收到一个中断信号、**并且**池也已经是 STOP 之后——那就确保我自己的中断位被重新置上。否则，把可能残留在中断位上的脏标记清掉，别污染下一个任务。"

**拆成四种组合（这张表是答案的核心）：**

| `runStateAtLeast(ctl, STOP)` | `Thread.interrupted()` | 结果 | 语义 |
|---|---|---|---|
| **true**（池已 STOP） | 短路不执行（但注意 `Thread.interrupted()` 被短路了，未清除标志） | `!wt.isInterrupted()` 成立时 → **重新置中断位** | 池要强制关闭，提醒当前/下一个任务收工 |
| **false**（池还 RUNNING/SHUTDOWN） | 执行 → 若之前有中断位则**清除**并返回 true，然后又因 `STOP` 判断为 false，整体为 false | **不置中断位** | ★ 关键：**清掉残留中断位**，防止上个任务的中断"毒害"下个任务 |
| false | false | 整体 false | 正常情况，什么都不做 |
| true | true | 整体 true → 置中断位 | 双重确认，确保中断传递 |

**为什么必须这么写（三个真实原因）：**

1. **线程是被复用的，中断位会"跨任务"残留。**
   ```
   任务 A 被 shutdownNow 中断 → A 没处理干净 → 中断位仍为 true
   → 线程回到 getTask() 拿到任务 B → B 一开始就处于"已中断"状态
   → B 里的任何 sleep/wait/IO 立刻抛 InterruptedException → 业务莫名失败
   ```
   `Thread.interrupted()` 的副作用（**读取并清除**）正好用来"擦掉上一轮的痕迹"。

2. **不能无条件清中断位。** 因为如果池已经 STOP，那"中断"是**合法且必须保留**的信号（它告诉后续代码"该退出了"）。所以要先判状态，再决定是"清"还是"重设"。

3. **`!wt.isInterrupted()` 是幂等保护**：避免对已中断的线程反复 `interrupt()`（虽然无害，但会多一次系统调用级别的唤醒尝试）。

**⚠️ 陷阱警告（这是本题的"坑"）：`Thread.interrupted()` 会清除标志！** 所以这段代码的执行顺序**不能调整**——必须"先清除、再按状态决定是否重设"。如果写成先判断状态再清除，就会出现"STOP 状态下中断位被清掉"的严重 Bug。

**延伸**：这一段代码也解释了为什么 `shutdownNow()` **不一定能立刻停掉任务**——它只在"任务边界"生效。如果任务本身是个 100% CPU 的 `for(;;){}`，不检查 `isInterrupted()`，也不调用任何阻塞方法，**线程永远不会退出**（见 Q16）。

---

### Q14. `completedAbruptly` 这个变量是干什么的？为什么它决定"要不要补线程"？
> **考核程度：★★★ 常考（细节题）** · 高频：美 / 滴 / 网

**参考答案**：

```java
boolean completedAbruptly = true;          // 默认"异常退出"
try {
    while (task != null || (task = getTask()) != null) {
        ... task.run() ...
    }
    completedAbruptly = false;             // 只有正常走到 while 结束才是 false
} finally {
    processWorkerExit(w, completedAbruptly);
}
```

**两个分支的含义：**

| `completedAbruptly` | 含义 | `processWorkerExit` 的处理 |
|---|---|---|
| `true` | `task.run()` 抛了异常，`while` 被异常打断 | **① 必须 `decrementWorkerCount()`**（因为 `getTask` 没来得及减，是异常路径）<br>② **必须补一个线程**（否则池容量永久缩水） |
| `false` | `getTask()` 返回 null，正常退休 | ① **不能减计数**（`getTask` 已经减过，重复减会把 ctl 减坏）<br>② 只在 `workerCount < min` 时才补 |

**为什么"补线程"是必须的（否则会发生什么）：**

```
场景：core=8，任务持续抛 RuntimeException
  线程 1 抛异常死 → 不补 → 池剩 7
  线程 2 抛异常死 → 不补 → 池剩 6
  ... 8 次异常后 → 池剩 0，但仍是 RUNNING 状态
  → 后续所有任务只能"入队 + recheck wc==0 → addWorker(null,false)"，
    且每次都是一次"新建线程 → 立刻死"的循环
  → 实际表现：吞吐崩溃 + 空转 + 线程创建风暴
```

**正常退休时为什么"可能补、也可能不补"：**

```java
if (runStateLessThan(c, STOP)) {
    if (!completedAbruptly) {
        int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
        if (min == 0 && ! workQueue.isEmpty()) min = 1;   // 队列有活，至少留 1 个
        if (workerCountOf(c) >= min) return;              // 够数 → 不补
    }
    addWorker(null, false);                               // 不够 → 补
}
```

三种"补"的情形要能说清：
1. **异常退出** → 无条件补 1 个。
2. **正常退休但池子小于 min**（如 `core` 被调大后线程陆续超时退出）→ 补。
3. **`allowCoreThreadTimeOut=true` 且队列还有任务但线程已归零** → 至少补 1 个（否则任务无人处理）。

---

### Q15. `tryTerminate()` 里为什么是 `interruptIdleWorkers(ONLY_ONE)`，只中断一个？中断全部不行吗？
> **考核程度：★★★★ 深水区（极少人能答对）** · 高频：阿 / 字

**参考答案（这题的价值在于展示"锁竞争 + 状态机推进"的双重考量）：**

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) || runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) return;   // 三重返回
        if (workerCountOf(c) != 0) {
            interruptIdleWorkers(ONLY_ONE);      // ★ 只叫醒一个
            return;
        }
        // CAS 到 TIDYING → terminated() → TERMINATED → signalAll
    }
}
```

**核心机制：级联传播（chain propagation）**

```
每个线程退出时都会调用 processWorkerExit → tryTerminate()
     ↓
tryTerminate 发现 wc != 0 → 中断【一个】空闲线程
     ↓
那个线程被唤醒 → getTask 判状态 → 返回 null → 退出 → 再次调用 tryTerminate()
     ↓
再次中断【一个】……（接力）
     ↓
直到某个线程发现 wc == 0 → CAS 到 TIDYING → 执行 terminated() → TERMINATED
```

**为什么不能一次中断全部（三个理由，层层递进）：**

**① 锁竞争（性能）**
`interruptIdleWorkers()` 需要持有 `mainLock` 并遍历整个 `workers` 集合。如果有 200 个线程，一次"全部中断"会产生 200 次 `tryLock` + 200 次 `interrupt()`（每次都是 futex wake 系统调用），且所有被唤醒的线程**同时**去抢 `mainLock` 和 `ctl` 的 CAS → **惊群效应（thundering herd）**，CPU 瞬间打满却只是互相踩踏。

**② 正确性（更关键）**
如果一次性全部唤醒，**没有任何一个线程能保证"最后一个"一定会把状态推进到 TIDYING**。`ctl.compareAndSet(c, ctlOf(TIDYING, 0))` 要求此时 `workerCount == 0`，而多个线程同时进入 `wc == 0` 的判断会产生竞态（都以为自己是最后一个，CAS 只有一个成功，其余的会在循环里重试——虽然最终能收敛，但**是繁忙重试而非安静等待**）。

**③ 状态机的"自收敛"设计**
级联传播让**每个线程的退出都成为一次"推进状态机"的机会**，最终由"最后一个退出者"完成 TIDYING 的 CAS。这是**分布式收敛**（类似 Gossip / 环状唤醒）的思路：不依赖某个"协调者"，而是每个参与者都承担一次推动职责。

**延伸追问："如果某个线程卡在任务里一直不退出怎么办？"**
→ `tryTerminate` 会一直停在 `workerCountOf(c) != 0` 那个分支，**永远到不了 TIDYING**。这就是为什么 `awaitTermination` 必须给超时，超时后再 `shutdownNow` 强推。**"状态机收敛依赖于所有线程最终退出"是这套设计的前提假设。**

---

### Q16. `shutdown()` 和 `shutdownNow()` 的底层差异是什么？`shutdownNow()` 能把一个纯 CPU 死循环任务停掉吗？
> **考核程度：★★★★★ 必考** · 高频：阿 / 字 / 腾 / 美 / 京

**参考答案**：

**差异表：**

| 维度 | `shutdown()` | `shutdownNow()` |
|---|---|---|
| 状态推进 | `RUNNING → SHUTDOWN` | `RUNNING → **STOP**` |
| 新任务 | 拒绝 | 拒绝 |
| 队列任务 | **跑完**（`getTask` 判 `SHUTDOWN && !queue.isEmpty()` 时继续取） | **丢弃**，`drainQueue()` 返回给你 |
| 执行中任务 | **不打断**（`interruptIdleWorkers` 只中断空闲的） | **尝试中断** |
| 返回值 | `void` | `List<Runnable>`（未执行的任务，可用于补偿/落库） |
| 幂等 | 是（`advanceRunState` 只许前进） | 是 |

**"停不掉死循环"—— 这是本题的灵魂：**

```
shutdownNow() 的中断路径：
    advanceRunState(STOP)
    → interruptIdleWorkers()  ← 注意：还是只中断"空闲"的（tryLock 成功）
    → 正在跑任务的线程 tryLock 失败 → 不会被 interrupt()
    → 那么"正在执行的任务"怎么被中断？
        ↓
    runWorker 在【任务边界】检查状态：
        if (runStateAtLeast(ctl, STOP)) → wt.interrupt()
        ↓
    也就是说：只有当前任务跑完、回到循环顶部时，才会给线程置中断位

结论：
  ✓ 能停掉的：阻塞型任务（sleep / wait / park / IO）—— 因为它本来就阻塞在
             可中断点上，STOP 后 runWorker 置中断位，它下次进入阻塞立刻抛出
  ✗ 停不掉的：纯 CPU 死循环（for(;;){} 里不 check isInterrupted()）
             —— 它永远不回到"任务边界"，中断位根本不会被设置，
               或者设置了它也不看
```

**这就是"Java 中断是协作式（cooperative）而非抢占式（preemptive）"的物理事实。** JVM 没有"强制杀死线程"的能力（`Thread.stop()` 已被废弃，因为它会留下不一致的共享状态）。

**正确的工程做法：**

```java
// 任务内部必须配合检查中断
while (!Thread.currentThread().isInterrupted()) {
    doWorkChunk();          // 分块执行，每块都检查
}
// 阻塞点天然可中断：
socket.read()  → 抛出 InterruptedException / ClosedByInterruptException
queue.take()   → 抛出 InterruptedException
```

**延伸（面试官可能追的第二刀）**："`shutdownNow()` 返回的 `List<Runnable>` 能用来做什么？"
→ **补偿/重建**。但要注意：这些 `Runnable` 可能是已经被包装过的 `FutureTask`（用 `submit` 提交的），**你只能重新 `execute` 它们，无法再拿到原来的 `Future` 引用**（除非你在提交时自己保存了）。所以生产上更常用"落库 + 重试框架"，而不是直接重投。

---

### Q17. 线程池内部有哪几把"锁"？`mainLock` 保护什么？为什么 `workers` 用 `HashSet` 而不是 `ConcurrentHashMap`？
> **考核程度：★★★★ 高频** · 高频：阿 / 腾 / 美

**参考答案：**

**四层锁，粒度从细到粗：**

```
第 1 层：ctl 的 CAS（无锁）
   AtomicInteger.compareAndSet —— 保护"状态 + 线程数"
   频次最高、临界区最短（一条 CAS），冲突时自旋重试

第 2 层：Worker 的 AQS（每个线程一把）
   保护"忙/闲"标记，只在 runWorker 里 lock/unlock
   ★ 冲突率极低（每把锁只被"自己 + 探测者"访问）

第 3 层：BlockingQueue 内部的锁
   保护队列结构（ArrayBlockingQueue 单锁 / LinkedBlockingQueue 双锁）
   这是实际的热点锁（高并发下入队出队都在这）

第 4 层：mainLock（ReentrantLock，全池一把）
   保护 workers 集合、largestPoolSize、completedTaskCount 等统计量
   只在 addWorker / processWorkerExit / shutdown / 查询池大小时持有
```

**`mainLock` 保护的清单：**

| 字段 | 为什么必须保护 |
|---|---|
| `workers`（HashSet） | 非线程安全；遍历时会被增删 |
| `largestPoolSize` | 需要"读-比较-写"，不是单次原子操作 |
| `completedTaskCount` | `+=` 非原子 |
| `termination`（Condition） | 与 `awaitTermination` 配对使用 |

**为什么不用 `ConcurrentHashMap`（三个理由）：**

1. **操作频率低**。`workers` 的增删只发生在"线程创建/销毁"时（秒级甚至分钟级），**不是每任务一次**。低频场景下，一把 `ReentrantLock` 的吞吐完全够，且实现简单。
2. **需要"多字段原子快照"**。`largestPoolSize`、`completedTaskCount`、`workers.size()` 三者要在同一个临界区里一致更新。并发容器只保证单元素操作原子，**组合操作仍需外部锁**——那就不如直接用一把锁。
3. **省内存 + 省复杂度**。`ConcurrentHashMap` 每个实例有 Node、table 数组、sizeCtl 等开销；`HashSet` 只是薄薄一层。数百个 Worker 的池子里，这种"基础设施开销"是要计入内存预算的。

**延伸（对比思维）**：这也解释了 `ConcurrentHashMap` 的适用边界——**高并发 + 单元素操作 + 无需组合一致性**。`workers` 三条都不满足，所以用锁。

---

### Q18. 一个任务从 `execute()` 进去，到执行完，可能有哪些"归宿"？各自的延迟量级是多少？
> **考核程度：★★★ 常考（工程感题）** · 高频：京 / 滴 / 网

**参考答案：**

**五种归宿（前三种是正常，后两种是异常）：**

| # | 归宿 | 触发条件 | 延迟量级 | 风险 |
|---|---|---|---|---|
| ① | **新建线程立即执行** | `wc < core` | 首次：`clone` 30~80 μs；之后：调度 1~10 μs | 首次有建线程抖动 |
| ② | **空中线程立即接手** | `SynchronousQueue` 或线程刚空闲 | 唤醒 10~100 μs（futex wake → CFS 调度） | 无 |
| ③ | **排队后执行** | 入队成功，等空闲线程 | **排队长度 × 单任务耗时 + 唤醒延迟** | ★ 最危险：延迟 = 队列积压的直接函数 |
| ④ | **补线程后执行** | 入队后 `recheck` 发现 `wc==0` | 同上 ① | `core=0` 时是常态路径 |
| ⑤ | **被拒绝** | 队列满 + 线程满 / 池已关闭 | 0（但业务失败） | 必须有明确处理 |

**归宿 ③ 的延迟公式（这是最有价值的部分）：**

```
任务在第 k 位的排队等待 ≈ k × (平均单任务耗时 / 并发线程数)
                        = k × 单任务耗时 / N_threads

例：队列 2000，8 线程，单任务 50ms
   队尾任务等待 ≈ 2000 × 50ms / 8 ≈ 12.5 秒
   → 业务超时 3 秒 → 这个任务"算出来也是废的"
```

**结论：**
> **"队列长度"不是"缓冲能力"，而是"延迟承诺"。** 队列每长一节，你就在向业务承诺"可以等更久"。所以有界队列的容量应该按 **SLA 倒推**（见 6.2 节），并且**任务入队时打时间戳，执行前检查是否已过期，过期直接丢弃**——这叫 **stale-task fast-fail**。

**加分延伸：三处"隐形成本"**
1. **提交线程可能被阻塞**（`CallerRunsPolicy`、`SynchronousQueue` 的 transfer）。
2. **`Future.get()` 的阻塞**（把线程池的问题反向传导给调用者，见 Q24）。
3. **第一次任务触发的类加载 + JIT 编译**（前几百毫秒吞吐只有稳态的 1/10，这是"预热"存在的另一个理由）。

---

### Q19. 用线程池能保证任务按提交顺序执行吗？
> **考核程度：★★★ 常考** · 高频：美 / 京 / 网

**参考答案：**

**分场景回答（能分层就是满分）：**

| 场景 | 是否有序 | 原因 |
|---|---|---|
| **单线程池**（`core=max=1`，FIFO 队列） | ✅ 严格有序 | 只有一个线程，队列 FIFO，取一个跑一个 |
| **多线程 + FIFO 队列** | ❌ **不保证** | 取任务有序，但**执行完成顺序无序**（线程 A 拿到任务 1、线程 B 拿到任务 2，谁先跑完不确定） |
| **多线程 + `SynchronousQueue`** | ❌ 更不保证 | 优先级由"哪个线程先来抢"决定，且非公平模式是 LIFO |
| **多线程 + `PriorityBlockingQueue`** | ❌ 只保证"取出有序" | 出队按优先级，但执行仍并行 |
| **`Executors.newSingleThreadExecutor()`** | ✅ 但**无界队列**危险 | 顺序有了，内存没了 |

**关键区分（面试官最想听的）：**

```
"取任务的顺序"  ≠  "任务完成的顺序"  ≠  "副作用生效的顺序"
```

多线程池只保证**取任务时有 FIFO 语义**（队列本身是 FIFO），但**执行期间的相对进度完全由调度决定**。所以"任务 1 先提交、任务 2 后提交"，但任务 1 的 DB 写入可能晚于任务 2。

**要真正保证顺序，四种做法：**

1. **单线程池**（性能差，但最简单）。
2. **按 key 分片**：`hash(key) % N` 路由到 N 个单线程池 → **同 key 有序，不同 key 并行**（这是 MQ 顺序消费的标准做法）。
3. **串行化的"有序队列 + 单消费者"**：`LinkedBlockingQueue` + 一个专属消费线程，配合批量提交。
4. **业务层用版本号/状态机**保证幂等与顺序（根本方案）。

**延伸**：`ForkJoinPool` 的 `invokeAll` 甚至**不保证"提交顺序=完成顺序"**，因为它有 work-stealing（任务会被别的线程偷走执行）。所以"依赖顺序"的场景**绝对不能用并行流/ForkJoin**。

---

### Q20. 线程池里的线程是守护线程吗？JVM 要退出时，线程池会怎样？
> **考核程度：★★★ 常考** · 高频：京 / 滴 / 网 / 华

**参考答案：**

**默认：不是守护线程（`daemon=false`）。**

```java
// ThreadPoolExecutor 的默认线程工厂
DefaultThreadFactory:
    Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
    if (t.isDaemon()) t.setDaemon(false);       // ★ 强制非守护
    if (t.getPriority() != Thread.NORM_PRIORITY)
        t.setPriority(Thread.NORM_PRIORITY);   // ★ 强制普通优先级
```

**JVM 退出的规则：**

```
JVM 在所有【非守护线程】结束后才会退出。
    ↓
主线程结束 ≠ JVM 退出（只要还有非守护线程活着）
    ↓
线程池的 Worker 是非守护线程 → **JVM 不会退出，会挂在那里等**
    ↓
表现：main 方法跑完了，进程还在（`jps` 看得到），CPU 为 0
```

**这就是"忘记 shutdown 线程池"的经典现象：单元测试跑完不退出、Spring Boot 应用 Ctrl+C 后卡住、命令行工具执行完不返回。**

**三种正确处理：**

| 做法 | 效果 | 适用 |
|---|---|---|
| `threadFactory` 里 `setDaemon(true)` | JVM 退出时不等待，**任务可能被丢弃** | 非关键的异步任务（日志、埋点） |
| 显式 `shutdown()` + `awaitTermination()` | 优雅，任务跑完 JVM 才退 | 生产服务（推荐） |
| Spring 的 `ThreadPoolTaskExecutor` + `setWaitForTasksToCompleteOnShutdown(true)` | 框架托管 | Spring 项目 |

**⚠️ 一个反直觉的点**：即使设了 `daemon=true`，**JVM 退出时也不会"中断"这些线程**，而是直接终止进程。所以"守护线程里的任务"可能在任何时刻被硬切——**涉及数据一致性的任务不能用守护线程池**。

**延伸追问**："`shutdown` 之后 JVM 会等多久？"
→ `shutdown()` 只是置状态 + 中断空闲线程，**JVM 会等所有 Worker 真正退出**（包括正在跑的任务）。如果任务卡死，JVM 就永久挂住 → 所以必须 `awaitTermination(timeout)` 后补 `shutdownNow()`。

---

### Q21. `corePoolSize` 和 `maximumPoolSize` 是什么关系？`core=0` 的线程池是什么行为？
> **考核程度：★★★ 常考** · 高频：京 / 滴

**参考答案：**

**关系约束：**

```java
if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
    throw new IllegalArgumentException();
// 构造时强校验 core <= max，运行时可调但不校验（setter 允许越界，语义自己负责）
```

**语义分工（一句话）：**

- `corePoolSize` = **常态并发度**（这些线程不会因空闲被回收）。
- `maximumPoolSize` = **突发并发度**（只在"队列已满"时才被用到）。
- 二者之间的差值 = **弹性区间**，靠 `keepAliveTime` 回收。

**`core=0` 的行为（`newCachedThreadPool` 就是这么配的）：**

```
execute(task)
  → wc(0) < core(0)? 不成立（0 < 0 为 false）
  → isRunning && queue.offer(task)
      ├─ SynchronousQueue：若此刻有线程在 take() → transfer 成功 → 直接交给它
      └─ 若无等待者 → offer 返回 false → **走第 3 步扩容**
  → addWorker(task, false) → 新建非核心线程直接执行
  → 线程空闲 keepAliveTime 后销毁，池回到 0
```

**`core=0` 的三个后果：**

| 后果 | 说明 |
|---|---|
| **每个任务都要么"手递手"、要么"新建线程"** | 队列（如果是 SynchronousQueue）完全无缓冲，任何抖动都会触发建线程 |
| **`recheck` 的 `wc==0` 分支变常态** | 每次入队成功后都可能要补线程 |
| **无预热可言** | `prestartAllCoreThreads` 在 `core=0` 时什么都不做（循环立即失败） |

**所以 `core=0` 只适合"任务极短 + 可容忍瞬时建线程"的场景，且 `max` 必须封顶。**

**延伸（一个高阶陷阱）**："`core=0` + `LinkedBlockingQueue`（无界）会发生什么？"
→ **灾难**。因为 `SynchronousQueue` 的 `offer` 会失败从而触发扩容；而无界队列的 `offer` 永远成功 → **永远不会建非核心线程** → **池子里永远 0 个线程**（除了 `recheck` 补的那个），任务全部堆积，队列涨到 OOM。**这是一个"合法参数组合导致完全失效"的经典案例。**

---

### Q22. Tomcat / Netty 的线程模型和线程池有什么本质区别？为什么它们不直接用 `ThreadPoolExecutor`？
> **考核程度：★★★ 常考（架构视野题）** · 高频：阿 / 腾 / 字节

**参考答案：**

**三方对比（一张表打穿）：**

| 维度 | `ThreadPoolExecutor` | Tomcat（`ThreadPoolExecutor` 变体） | Netty（`EventLoopGroup`） |
|---|---|---|---|
| 基本单位 | 线程 | 线程 | `EventLoop`（1 线程绑 1 个 Selector） |
| 任务来源 | 队列 | 队列（`TaskQueue` 定制） | **IO 事件 + 任务队列** |
| 阻塞模型 | 线程阻塞在队列上 | 线程阻塞在队列（或 `poll` 带超时） | **非阻塞 IO 多路复用，一个线程管 N 个连接** |
| 线程数 | 按任务调整 | `maxThreads` 默认 200 | `workerGroup` 默认 `2 × CPU` |
| 关键定制 | — | **`TaskQueue` 重写 `offer`：线程数 < max 时返回 false，强制扩容** | 自己实现 `SingleThreadEventExecutor`，用 `LinkedBlockingQueue` + `wakeup` |

**最精彩的一点：Tomcat 那版 `TaskQueue` 是对 JDK 线程池缺陷的"打补丁"。**

```java
// org.apache.tomcat.util.threads.TaskQueue（简化）
@Override
public boolean offer(Runnable o) {
    if (parent.isShutdown()) return super.offer(o);
    // 如果线程数已达 max → 正常入队
    if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
    // 如果提交数 < 当前线程数 → 有空闲线程，入队即可
    if (parent.getSubmittedCount() <= parent.getPoolSize()) return super.offer(o);
    // ★ 关键：线程数 < max → 返回 false，**强制走扩容路径**，而不是先排队
    if (parent.getPoolSize() < parent.getMaximumPoolSize()) return false;
    return super.offer(o);
}
```

**为什么需要这个补丁：** JDK 的语义是"**先排队，队列满了才扩容**"。而 Tomcat 想要的是"**先扩容到 max，再排队**"——因为 Web 请求不允许在队列里等太久（延迟敏感）。于是它让 `offer` 主动返回 false 来"欺骗" `execute` 走第 3 步。

**Netty 的区别更本质：**

```
ThreadPoolExecutor：**线程等任务**（阻塞在队列上）
Netty EventLoop：  **线程等 IO 事件**（阻塞在 epoll_wait 上），
                    业务任务被塞进 eventLoop 自己的 taskQueue，
                    由 select 循环在"两次 IO 事件之间"批量执行

→ Netty 的瓶颈不是线程池大小，而是"单个连接的处理必须快"，
  因为一个 EventLoop 上挂了成百上千个连接，任何一个阻塞都会影响全部
```

**结论（这是本题的落点）：**
> 线程池解决的是"**任务 → 线程**的分配问题"；Netty 解决的是"**连接 → 线程**的分配问题"。前者以任务为中心，后者以 IO 事件为中心。**所以 Netty 里绝不能用 `CallerRunsPolicy` 或任何阻塞调用**——那会把整个 EventLoop 的成百上千连接一起卡死（见 9.10 坑）。

---

### Q23. `Worker` 的 `firstTask` 为什么允许为 `null`？这带来了什么能力？
> **考核程度：★★★ 常考（细节题）** · 高频：美 / 滴

**参考答案：**

**`firstTask` 有两种用法：**

| 值 | 场景 | 行为 |
|---|---|---|
| **非 null** | `execute` 的第 1 步 / 第 3 步扩容 | 新线程启动后**立刻执行这个任务**，无需先经过队列（省一次入队/出队） |
| **`null`** | ① `addWorker(null, false)` 补位<br>② `prestartCoreThread()` 预热 | 线程启动后**直接进 `getTask()` 等队列任务** |

**"补位"是 `firstTask=null` 最重要的用途：**

```
场景：core=0 或所有线程已退出，但队列里还有任务
      execute 第 2 步 recheck 发现 workerCount == 0
      → addWorker(null, false)
      → 新线程启动 → 不进 while 的 firstTask 分支 → 直接 getTask() → 拿到队列里的任务

如果这里用 addWorker(task, false) 会怎样？
      → 新线程去跑 task，但队列里那个任务**依然无人处理**（因为新线程没去取）
      → 任务"漏掉"了
```

**因此 `firstTask == null` 在 `addWorker` 的状态闸门里也被特殊对待：**

```java
if (rs >= SHUTDOWN &&
    !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
    return false;
//   SHUTDOWN 状态下，**只允许创建"不带任务的补位线程"**，
//   因为 SHUTDOWN 的语义是"不再接新任务，但要清空队列"。
```

**能力总结：** `firstTask=null` 让 **"创建线程"这件事与"提交任务"解耦**。线程可以先建好等着（预热），也可以在需要时"只补线程不塞任务"（补位）。**如果没有这个设计，`prestartCoreThread` 和 `recheck` 补位都无从实现。**

---

### Q24. `Future.get()` 阻塞会反噬线程池——什么是"线程饥饿"？怎么避免？
> **考核程度：★★★ 常考（场景题）** · 高频：阿 / 字 / 美

**参考答案：**

**问题的标准形态：**

```
池大小 = 10，线程 10 个
来了 10 个"父任务"，每个都：
    1. submit(子任务) → 子任务进队列
    2. future.get()   ← **阻塞等待子任务完成**

    ↓
10 个线程全部阻塞在 get() 上
子任务在队列里，没有任何线程能取
    ↓
【永久死锁】，且 CPU 占用 0，jstack 里全是 WAITING
```

**根因（一句话）：** **线程池不知道"线程正在等待自己池里的任务"，无法做补偿。** 这是它与 `ForkJoinPool` 的根本差异——FJP 遇到 `join` 时会**创建补偿线程或窃取任务**（`tryCompensate`）。

**五种解法（按推荐度排序）：**

| 方案 | 做法 | 评价 |
|---|---|---|
| **① 隔离池** | 父任务用池 A，子任务用池 B（B 的容量 ≥ A 的并发 × 子任务数） | ✅ 最常用、最可靠 |
| **② 异步化** | `CompletableFuture.allOf(...).thenApply(...)`，全程不阻塞 | ✅ 最优，但要求全链路改造 |
| **③ 改 `ForkJoinPool`** | 用 `RecursiveTask` + `fork/join`，它自带补偿 | ✅ 适合分治型计算 |
| **④ 加大池** | 池大小 ≥ 父任务数 × (1 + 子任务数) | ⚠️ 治标，且线程数爆炸 |
| **⑤ 限流** | 用信号量控制"同时在等的父任务数" | △ 缓解，不根治 |

**通用诊断特征（背下来，面试能用）：**

```
jstack 里出现大量：
   "pool-1-thread-N" ... java.lang.Thread.State: WAITING
        at java.util.concurrent.FutureTask.get(...)
        at ...你的业务代码...
   ← 且 CPU 为 0，且这些线程数 == corePoolSize
→ 立刻怀疑"同池父子任务 + Future.get"
```

**延伸（这是加分点）：为什么 `CompletableFuture` 的默认 `commonPool` 也有同样问题？**
→ `ForkJoinPool.commonPool()` 的并行度是 `CPU - 1`，**它同样会被阻塞型任务拖垮**（阻塞占用工作线程，而 commonPool 没有无限补偿能力）。所以官方文档明确指出：**`commonPool` 只适合无阻塞的纯计算任务**，IO 任务必须自定义 Executor（见 Q38）。

---
## C 组 · 队列、拒绝与容量（Q25–Q35）

### Q25. 线程池的 `workQueue` 该怎么选？六种 `BlockingQueue` 的底层差异是什么？
> **考核程度：★★★★ 高频** · 高频：阿 / 字 / 美 / 京 / 快

**参考答案：**

**六种队列的底层结构与行为：**

| 队列 | 数据结构 | 锁模型 | 容量 | 线程池中的效果 |
|---|---|---|---|---|
| `ArrayBlockingQueue` | 环形数组 | **单锁** + `notEmpty`/`notFull` 两条件 | 有界（必填） | 内存紧凑、GC 友好；**入队出队互斥，高并发是瓶颈** |
| `LinkedBlockingQueue` | 单向链表 `Node` | **双锁**（putLock/takeLock）+ `AtomicInteger count` | 默认 `Integer.MAX_VALUE` | **生产消费可真并行，吞吐高**；每元素一份 Node，GC 压力大；★**默认无界 = 陷阱** |
| `SynchronousQueue` | 无存储 | CAS 无锁 transfer，`park/unpark` | **0** | 不缓冲，**无空闲线程则 offer 失败** → 强制扩容。配 `max=MAX_VALUE` 会爆线程 |
| `PriorityBlockingQueue` | 二叉堆数组 | 单锁 + `notEmpty` | 无界（默认 11，扩容） | **put 永不阻塞**；可按优先级取；低优先级可能饿死 |
| `DelayQueue` | 优先级队列 + 到期时间 | 同 PBQ + `available` 条件 | 无界 | `take` 精确睡到队首到期；`ScheduledThreadPoolExecutor` 的基础 |
| `LinkedBlockingDeque` | 双向链表 | 单锁 + 两条件 | 默认无界 | 可从两端取，适合 work-stealing 类场景 |

**决策树（面试时画出来最直观）：**

```
任务需要"排队缓冲"吗？
├── 不需要（要求任务立即执行或立即扩容）
│      → SynchronousQueue（配一个有上限的 max）
│
├── 需要，且要"背压"（不能无限堆积）
│      → ArrayBlockingQueue(容量)    ← 小容量、低延迟、GC 友好
│      → LinkedBlockingQueue(容量)  ← 大容量、高吞吐（务必传容量！）
│
├── 需要按优先级
│      → PriorityBlockingQueue（注意饿死问题，最好配多池分优先级）
│
└── 需要延迟/定时
       → DelayQueue（一般直接用 ScheduledThreadPoolExecutor）
```

**三个必答的细节：**

1. **`ArrayBlockingQueue` 只有一把锁**，所以"生产者"和"消费者"会互斥；`LinkedBlockingQueue` 用两把锁，入队和出队可并行，**吞吐约高一倍**（代价是 Node 对象和 GC）。
2. **`LinkedBlockingQueue` 不传容量就是 `Integer.MAX_VALUE`**，这正是 `newFixedThreadPool` 的坑源。**生产代码里必须显式 `new LinkedBlockingQueue<>(2000)`。**
3. **`Condition.await()` 必须放在 `while` 里重查条件**（防虚假唤醒 + 防被抢先）：`while (count == items.length) notFull.await();`

**延伸**："为什么不用 `ConcurrentLinkedQueue`？"
→ 它**不阻塞**，`getTask()` 的 `take()` 就没有等待语义了，线程只能自旋（烧 CPU）或 sleep（延迟高）。**线程池的空闲模型依赖"阻塞队列的 futex 睡眠"，这是它能"0 CPU 待机"的根本。**

---

### Q26. `SynchronousQueue` 没有容量，它是怎么工作的？公平模式和非公平模式差在哪？
> **考核程度：★★★ 常考（深水细节）** · 高频：阿 / 字

**参考答案：**

**核心语义：手递手（rendezvous），不存储。**

```
生产者 put(x)：
   必须找到一个正在等待的消费者 take()，直接把 x 交过去
   找不到 → 自己 park 等待（或被拒绝）

消费者 take()：
   必须找到一个正在等待的生产者，直接取走
   找不到 → 自己 park 等待

★ 中间没有任何缓冲，因此"容量 = 0"
```

**两种实现（`Transferer` 的两个子类）：**

| 模式 | 实现类 | 结构 | 匹配顺序 | 特点 |
|---|---|---|---|---|
| **非公平（默认）** | `TransferStack` | **LIFO 栈** | 后进先出 | 吞吐高（后来的线程还在 CPU 上，缓存热、CAS 易成功）；**可能饥饿**（先来的等很久） |
| **公平**（`new SynchronousQueue<>(true)`） | `TransferQueue` | **FIFO 队列**（`QNode` 双向链表） | 先进先出 | 公平、延迟可预期；**吞吐略低** |

**底层实现要点（这是"深水"部分）：**

```java
// TransferStack 的核心：无锁 CAS 三元匹配
// 节点有 3 种模式：
//   REQUEST  = 消费者（要数据）
//   DATA     = 生产者（给数据）
//   FULFILLING = 正在匹配中（占位标记，避免 ABA）
//
// 匹配流程（简化）：
//   1. 头节点是"相反模式" → 尝试 CAS 压入一个 FULFILLING 节点
//   2. CAS 成功 → 交换数据 → 两个节点一起弹出
//   3. CAS 失败 → 自旋重试若干次
//   4. 自旋失败 → park()，等被匹配者 unpark

★ 全程无锁（纯 CAS + park/unpark），因此在低竞争下极快（~100ns 级）
★ 但它是"队列"里唯一一个 *不用 AQS/Condition* 的实现
```

**在线程池里的实际含义（这才是重点）：**

```
pool = new ThreadPoolExecutor(0, 200, 60s, SECONDS, new SynchronousQueue<>());

场景：8 个线程都空闲在 take()
    → 提交任务 → 某个线程被匹配 → offer 成功 → 直接执行（延迟极低）

场景：8 个线程都在忙
    → 提交任务 → 没有等待者 → offer 失败 → 走 execute 第 3 步 → 新建线程（到 200 为止）
    → 第 201 个任务 → 拒绝

★ 所以 SynchronousQueue 配线程池的语义是："**有闲线程就给它，没有就建，建不了就拒**"
★ 它把"排队"这个缓冲彻底取消了 → 任务延迟最低，但抗突发能力最差
```

**延伸坑**：`SynchronousQueue` 的 `offer()` 在**没有等待者时立即返回 false**，所以 `execute` 的第 2 步 `recheck` 逻辑对它几乎不起作用；它每次都是"入队失败 → 扩容"的路径，**这也是 `newCachedThreadPool` 能无限建线程的机制来源**。

---

### Q27. 四种拒绝策略各自的真实影响是什么？`CallerRunsPolicy` 有什么风险？如何自定义？
> **考核程度：★★★★★ 必考** · 高频：阿 / 字 / 腾 / 美 / 京 / 拼

**参考答案：**

**四种策略的源码（必须能默写 `CallerRunsPolicy`）：**

```java
// ① AbortPolicy（默认）：抛异常
throw new RejectedExecutionException("Task " + r + " rejected from " + e);

// ② CallerRunsPolicy：调用者线程直接跑
if (!e.isShutdown()) { r.run(); }    // ★ isShutdown 判断不能漏

// ③ DiscardPolicy：静默丢弃
// 空实现

// ④ DiscardOldestPolicy：踢掉队首，重投
if (!e.isShutdown()) { e.getQueue().poll(); e.execute(r); }
```

**影响对照表：**

| 策略 | 任务是否丢失 | 提交者是否被阻塞 | 有无可观测信号 | 核心风险 |
|---|---|---|---|---|
| `AbortPolicy` | 不丢（抛异常） | 否 | 异常 | **业务侧不 catch → 任务静默失败**（异常被上层吞掉） |
| `CallerRunsPolicy` | 不丢 | **是（同步执行）** | 提交者变慢 | **把提交者线程拖死**（IO 线程/EventLoop 上会引发雪崩） |
| `DiscardPolicy` | **丢** | 否 | ⚠️ **完全无声** | 排查时"任务凭空消失"，最危险 |
| `DiscardOldestPolicy` | 丢最老的 | 否 | 无 | 若队列是 `PriorityBlockingQueue`，`poll` 掉的是**优先级最高**的（语义完全错位） |

**`CallerRunsPolicy` 的双面性（这是本题的核心考点）：**

**正面：它是天然的"背压（backpressure）"机制。**

```
提交者被拉去同步执行 → 提交速率自然下降 → 队列不再增长
这是"负反馈"：系统压力越大，输入越慢
（这也是它在批处理场景被推荐的原因）
```

**负面：它把"线程池过载"转化为"调用方阻塞"，风险转嫁。**

```
❌ 在 Tomcat 的 worker 线程里用 → 请求线程被占死 → 连接堆积 → 整个服务不可用
❌ 在 Netty EventLoop 里用 → 一个连接池的过载卡死全部连接
❌ 在定时任务线程里用 → 调度全线延迟
✅ 在"专门的数据搬运/批处理线程"里用 → 合理
```

**自定义策略的模板（生产必备）：**

```java
public class MetricsRejectPolicy implements RejectedExecutionHandler {
    private final Counter rejected = ...;
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        rejected.increment();                                     // ① 打点（必须）
        log.warn("rejected: poolSize={}, queueSize={}, active={}",
                 e.getPoolSize(), e.getQueue().size(), e.getActiveCount());  // ② 现场快照
        RetryStore.save(r);                                       // ③ 落库补偿（可选）
        throw new RejectedExecutionException("pool saturated");    // ④ 明确失败
    }
}
```

**延伸追问："`DiscardPolicy` 真的完全不能用吗？"**
→ 可以用，但**必须同时满足两条**：① 任务可丢（如非核心埋点）；② **在别处有独立的计数**（否则你永远不知道丢了多少）。华为/阿里的规范里通常要求"任何丢弃都必须可观测"。

---

### Q28. 线程池的核心线程数到底该怎么设置？有没有计算公式？
> **考核程度：★★★★★ 必考** · 高频：全部大厂

**参考答案（分四层作答，层次越高分越高）：**

**第一层：口诀**

```
CPU 密集型：N = CPU 核数 + 1
IO 密集型：N = CPU 核数 × 2（经验值，或 2N+1）
```

**第二层：Brian Goetz 公式**

```
        ┌        W ┐
N = N_cpu × │ 1 + ─── │ × U_cpu
        └        C ┘

W = 平均等待时间（IO、锁、RPC）
C = 平均计算时间
U_cpu = 目标 CPU 利用率（0~1）
```

**推导（能讲出推导过程才是高手）：**
每个线程的生命周期是 `W + C`，但只有 `C` 在吃 CPU。要让 CPU 利用率达到 `U`，就需要线程数 = "纯计算所需数量 × (W+C)/C × U"。其中"纯计算所需"就是 `N_cpu`。

**第三层：Little's Law（工程上最实用）**

```
L = λ × W    →    N ≥ 峰值QPS × 单任务平均耗时 × 安全系数(1.2~1.5)

例：峰值 500 QPS，单任务 80ms → N ≥ 500 × 0.08 = 40 → 取 50
```

**第四层（最有区分度）：从下游资源倒推封顶**

```
线程数的真正上限不是 CPU，而是【下游能承受多少并发】：
   - DB 连接池 maximumPoolSize = 20 → DB 类任务线程数不应超过 20
   - Redis 集群 QPS 配额 = 5 万 → 任务耗时 × 线程数 ≤ 配额
   - 下游 RPC 的并发限流 = 100 → 线程数超过 100 只会在限流器上排队

结论：**N = min( 公式算出来的, 下游资源上限 )**
超过上限的线程不是"更强"，而是"更长的排队队列"。**
```

**一个完整的落地例子：**

```
场景：8 核机器，接口平均 100ms（CPU 计算 15ms，RPC+DB 等待 85ms），
     峰值 300 QPS，DB 连接池 20。

① Goetz：N = 8 × (1 + 85/15) × 0.8 ≈ 8 × 6.67 × 0.8 ≈ 42
② Little：N ≥ 300 × 0.1 × 1.3 ≈ 39
③ 下游封顶：DB 连接池 20 → 但只有 70% 任务走 DB → 20/0.7 ≈ 28
④ 折中取值：core=32, max=48, queue=1000（约 300×2s 缓冲）

★ 最后必须【压测验证】：从 16 开始逐步加到"吞吐不再上升、P99 开始恶化"的拐点。
```

**加分结论（面试官会眼前一亮）：**
> **线程池调参只能榨取"当前任务形态下的局部最优"。真正的提升来自改任务形态**——串行 RPC 改并行/批量（`W` 下降）、加缓存（`λ` 打到下游变小）、缩短锁持有时间。**"调参"和"改架构"是两个数量级的收益。**

---

### Q29. 线程数是不是越多越好？从操作系统层面看，线程开太多会发生什么？
> **考核程度：★★★★ 高频（OS 结合题）** · 高频：字 / 腾 / 华 / 阿

**参考答案（这题必须往内核答，否则就是背八股）：**

**四个层级的代价，从下往上：**

**① 硬件层：Cache 与 TLB 被反复冲刷**

```
线程切换 → 新线程的代码/数据不在 L1/L2
        → 大量 Cache Miss → 内存带宽成为瓶颈
        → 分支预测器历史被污染（BTB 里的表项全部作废）
实测：跨线程切换的"直接开销"1~3μs，但"有效算力损失"可达 5~10μs 等效
```

**② 内核层：CFS 的时间片被稀释（有精确数学）**

```
Linux CFS 调度参数：
   sched_latency_ns   = 6 ms      （一个调度周期的目标）
   min_granularity_ns = 0.75 ms   （单任务最小时间片）

4 核机器，64 个可运行线程：
   理想时间片 = 6ms / 64 = 0.09ms
   但最小粒度是 0.75ms → 不再压缩 → 每个线程按 0.75ms 轮转
   → 一个线程要等 64 × 0.75 = 48ms 才轮到下一次
   → **延迟放大 8 倍（相比 8 线程时）**

★ 这就是"线程开太多，每个任务都变慢"的内核级解释
```

**③ 内存层：不是"1MB × N"那么简单**

```
每个线程的成本：
   - 用户栈：-Xss（默认 1MB）虚拟地址，物理按需
   - 内核栈：16KB（THREAD_SIZE），**实际物理**
   - task_struct + 各类 struct：2~4KB
   - JVM 侧 JavaThread/OSThread：1~2KB
   - VMA：约 1~2 个（栈区 + guard page）→ 会撞 vm.max_map_count（默认 65530）
   → 数万线程量级即到上限，报 unable to create native thread
```

**④ 系统层：调度器、GC、监控全线受压**

```
- GC 线程要遍历/挂起更多线程（safepoint 变慢，STW 变长）
- jstack 采样成本上升（几百线程还行，几千线程 dump 一次要几秒）
- 内核 task 链表、负载统计、cgroup 记账全部变慢
```

**结论（一句话）：**

```
最优线程数 = min( CPU 能饱和利用的数量 , 下游能承受的并发 )
              ↑ 前者由 Goetz 公式/Little 定律给出
                            ↑ 后者由连接池/限流配额给出
超过这个数的线程，**收益为负**：只会增加上下文切换、延迟和内存，不增加吞吐。
```

---

### Q30. 一个 Java 线程从创建到销毁，对应哪些系统调用？线程池的"取任务"最终阻塞在哪里？
> **考核程度：★★★★ 深水加分题** · 高频：阿（中间件/基础架构岗）、字节基础架构

**参考答案（这题考的是"能不能打通到内核"，多数候选人答不出）：**

**创建路径（一次"用户态-内核-用户态"往返）：**

```
Thread.start()                [用户态]
  → start0()                  [JNI]
  → JVM_StartThread           [HotSpot]
  → new JavaThread            [malloc + 栈 mmap]
  → os::create_thread
      → pthread_attr_setstacksize(1M)
      → pthread_create(..., thread_native_entry, ...)
          → **clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|
                   CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|...)**   ★ 系统调用 #1
          ← 返回新 task_struct，两个 task 共享 mm_struct
  ← 等子线程把状态置为 INITIALIZED（内部信号量同步）
```

**执行路径：**

```
task 被 CFS 选中 → 上 CPU → thread_native_entry
  → Thread::call_run → thread_entry → JavaCalls::call_virtual("run")
  → Worker.run() → runWorker()
```

**"取任务"的阻塞点（线程池待机 0 CPU 的秘密）：**

```
getTask()
  → workQueue.take() 或 poll(timeout)
      → ArrayBlockingQueue/LinkedBlockingQueue
          → lock.lock()            → **futex(FUTEX_WAIT)** ★ 系统调用 #2（有竞争时）
          → notEmpty.await()      → Condition.await → LockSupport.park
              → Unsafe.park → pthread_cond_wait → **futex(FUTEX_WAIT)** ★
          ← 进入 TASK_INTERRUPTIBLE (S) 状态，**从 runqueue 摘下，不吃 CPU**
```

**唤醒路径：**

```
新任务到达 → offer()
  → notEmpty.signal() → unpark(workerThread)
      → **futex(FUTEX_WAKE, 1)** ★ 系统调用 #3
  → 目标线程从 wait queue 被摘下 → 塞进 CFS 红黑树 → 等调度
  → 被调度后从 take() 返回，拿到任务
```

**销毁路径：**

```
runWorker 返回 → Thread.run() 结束 → JavaThread::exit
  → 关闭该线程的 VMA（munmap 栈）
  → **exit() / futex 唤醒等待者（join）** ★
  → pthread 是 detached（JVM 用 PTHREAD_CREATE_DETACHED）→ 内核自动回收 task_struct
```

**一句话总结（可以直接背）：**

> **线程池的"高效"来自把 `clone()` 变成一次性成本；它的"低耗"来自把轮询变成 `futex` 上的睡眠。一个 `execute()` + 一次 `take()`，本质上是"一次 `clone` 的摊销 + 一次 `futex` 的等待"。**

**延伸追问："那 `LockSupport.park` 和 `Object.wait` 在内核层有区别吗？"**
→ 都落到 `futex`，但实现层级不同：
- `Object.wait()` → `ObjectMonitor::wait` → `Thread::_ParkEvent` → `pthread_cond_wait` → `futex`
- `LockSupport.park()` → `Parker::park` → 直接操作 `_counter` + `pthread_cond_wait`/`futex`
- **`park` 更轻（少一层 monitor 记账），这也是 `AQS` 体系比 `synchronized` 在高竞争下更可控的原因之一**

---

### Q31. Java 线程是 1:1 模型，那虚拟线程是什么？它能让线程池消失吗？
> **考核程度：★★★★ 高频（新趋势题）** · 高频：字 / 阿 / 腾 / 华

**参考答案：**

**模型对比：**

| 维度 | 平台线程（平台线程 / 1:1） | 虚拟线程（JDK 21 / M:N） |
|---|---|---|
| Java 线程 ↔ 内核线程 | 1 : 1 | M : N（M 可达百万，N ≈ 核数） |
| 栈 | `-Xss`（默认 1MB），原生栈 | **堆上的 StackChunk 链表**，可动态伸缩 |
| 创建成本 | `clone()`，30~80 μs | 堆对象，纳秒~微秒级 |
| 阻塞时 | 占住内核线程，`S` 状态 | **`unmount`，载体线程（carrier）立刻去跑别的虚拟线程** |
| 调度 | CFS（内核） | `ForkJoinPool`（用户态，M:N 调度） |

**关键机制：`Continuation` + `mount/unmount`**

```
虚拟线程执行中遇到阻塞点（socket read / queue take / sleep）
   → 把它的栈帧"快照"进堆（Continuation.yield）
   → unmount：从载体线程上摘下
   → 载体线程立刻去跑队列里的下一个虚拟线程（**不浪费**）
数据到达
   → 调度器把 Continuation 重新 mount 到某个载体线程
   → 从 yield 处继续执行（**栈恢复**）
```

**它能让线程池消失吗？—— 不能，但会让线程池"降级为限流器"。**

| 能力 | 线程池的不可替代性 |
|---|---|
| **复用内核线程** | 虚拟线程接管了这件事 → 线程池的"复用"价值下降 |
| **控制并发度（限流）** | ✅ **完全不可替代**。虚拟线程没有"最大并发数"概念，你必须用 `Semaphore` 或**一个小线程池/信号量**来限制对下游的压力 |
| **CPU 密集型的并行度控制** | ✅ 不可替代（虚拟线程跑 CPU 密集任务毫无优势，反而多一层调度） |
| **任务队列/背压** | ✅ 需要显式的有界队列 |

**正确的未来架构（这是满分答案）：**

```
IO 密集型短任务 → 虚拟线程（Executors.newVirtualThreadPerTaskExecutor()）
CPU 密集型       → 小线程池，大小 = N_cpu + 1
需要限流的地方    → Semaphore / 有界线程池（保护下游）
```

**必须提的三个坑（能说出来说明真研究过）：**

1. **`synchronized` 会 pin 住载体线程**（JDK 21 中；JEP 491 在 JDK 24 修复）。在 `synchronized` 块内阻塞会让虚拟线程无法 unmount，M:N 退化为 1:1。
2. **`ThreadLocal` 百万份会吃爆堆**（每个虚拟线程一份 ThreadLocalMap）。
3. **CPU 密集任务放虚拟线程池 → 载体线程被占满，其他虚拟线程全部饿死**（因为 carrier 数量 = 核数）。

---

### Q32. 线程池里用 `ThreadLocal` 会有什么问题？为什么比单线程更危险？
> **考核程度：★★★★ 高频（生产事故题）** · 高频：阿 / 字 / 美 / 京 / 滴

**参考答案：**

**两个独立问题（必须分开讲）：**

**问题 A：内存泄漏（线程不死 → ThreadLocalMap 不死）**

```
Thread 对象持有 threadLocals（ThreadLocalMap）
  Entry 的 key = WeakReference<ThreadLocal>（弱引用，可被回收）
  Entry 的 value = **强引用** 指向业务对象

单线程/短命线程：线程结束 → Thread 对象可回收 → map 一起回收 → 没问题
线程池：线程**永不结束** → map 永不回收
        → 业务代码忘记 remove() → **value 一直挂在弱引用 entry 上**
        → 弱引用 key 被回收后形成 key=null 的"僵尸 entry"，value 仍强引用
        → 内存泄漏（直到 ThreadLocal 被 replace/cleanSomeSlots 偶然清理）
```

**问题 B：串数据（比泄漏更可怕）**

```
线程 T1 跑任务 A：tl.set("用户A的上下文") → 忘了 remove
线程 T1 跑任务 B（复用同一线程）：tl.get() → **拿到"用户A的上下文"**
→ 越权、串号、脏数据 —— 这是安全事故级别的 Bug
```

**为什么在线程池里更危险：**
> **单线程场景下，线程结束就是天然的"清理"。线程池把"线程的生命周期"和"任务的生命周期"解耦了，于是"任务级的作用域"失去了自动边界。** 这是"复用"的必然代价。

**四种修复方式：**

| 方案 | 做法 |
|---|---|
| ① **finally remove（标准）** | `try { tl.set(x); ... } finally { tl.remove(); }`（**注意是 `remove()` 不是 `set(null)`**，后者仍保留 Entry） |
| ② **包装器统一清理** | Spring 的 `TaskDecorator`；自己写 `Runnable` 包装，在执行前后清理 |
| ③ **TTL（阿里 TransmittableThreadLocal）** | 解决 `InheritableThreadLocal` 在池化下失效的问题（见 9.4），并通过 `TtlRunnable` 在"提交时捕获、执行前回放、执行后恢复" |
| ④ **显式传参** | 彻底不用 ThreadLocal，把上下文作为任务参数传递（最安全，改造成本最高） |

**延伸（面试官可能追问）**："为什么 `ThreadLocalMap` 的 key 要用弱引用？"
→ 为了让"业务代码丢掉了 ThreadLocal 引用"时，key 能被 GC 回收，从而通过 `cleanSomeSlots`/`expungeStaleEntries` 顺手清掉 value。**但这是个"尽力而为"的补丁——如果你不 `remove()`，value 在 `get/set` 之前都可能一直挂着。所以弱引用不能替代 `remove()`。**

---

### Q33. 如何优雅关闭一个线程池？关闭时有哪些坑？
> **考核程度：★★★★ 高频** · 高频：阿 / 腾 / 美 / 京

**参考答案：**

**标准四步（可直接背的代码）：**

```java
pool.shutdown();                                        // 1. 停止接新任务
try {
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) { // 2. 等队列跑完
        List<Runnable> dropped = pool.shutdownNow();    // 3. 超时强制中断
        if (!pool.awaitTermination(10, TimeUnit.SECONDS)) // 4. 再等一次
            log.error("pool cannot terminate, size={}", pool.getPoolSize());
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();                 // ★ 恢复中断位
}
```

**五个真实坑：**

| # | 坑 | 说明 |
|---|---|---|
| 1 | **关闭顺序错** | 先停 DB/Redis 再停线程池 → 任务跑到一半依赖已断，报一堆"连接已关闭"。**顺序：生产者 → 线程池 → 资源** |
| 2 | **`awaitTermination` 不给超时** | `Long.MAX_VALUE` 会让停机永久挂住（任务卡死时）。**必须给超时 + 兜底 `shutdownNow`** |
| 3 | **`shutdown` 后还在提交** | `finally` 块里"收尾任务"会在停机时抛 `RejectedExecutionException` → 用 `if (!pool.isShutdown())` 或 try-catch |
| 4 | **`shutdownNow` 打断正在跑的 DB 事务** | 中断会让连接变成"脏连接"，事务状态不确定 → **对事务型任务要设"允许跑完"标记，或在业务层捕获中断做回滚** |
| 5 | **Spring 容器关闭时线程池还在跑** | `@PreDestroy` 里的顺序问题；`@Async` 用的 `ThreadPoolTaskExecutor` 要 `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(30)` |

**Spring Boot 的正确配置：**

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(8); e.setMaxPoolSize(32);
    e.setQueueCapacity(1000);
    e.setThreadNamePrefix("biz-");
    e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    e.setWaitForTasksToCompleteOnShutdown(true);   // ★ 停机等待
    e.setAwaitTerminationSeconds(30);              // ★ 最长等 30s
    return e;
}
```

**延伸**："K8s 里 pod 被 SIGTERM 时怎么办？"
→ 需要把"优雅停机"接到 K8s 的 `terminationGracePeriodSeconds`（默认 30s）上：**先摘流量（改成 Not Ready）→ 等 in-flight 请求结束 → 关线程池 → 退出**。顺序错了会出现"请求已摘流量但线程还在跑，pod 被 SIGKILL 硬杀"。

---

### Q34. 你会怎么监控一个线程池？有哪些必须采集的指标？
> **考核程度：★★★ 常考（工程能力题）** · 高频：阿 / 美 / 京 / 滴 / 拼

**参考答案：**

**六个核心指标 + 三个衍生指标：**

| 指标 | API | 告警信号 |
|---|---|---|
| `poolSize` | `getPoolSize()` | 持续 == `max` → 弹性已耗尽 |
| `activeCount` | `getActiveCount()` | 持续 == `poolSize` → 全部在忙 |
| `queueSize` | `getQueue().size()` | 持续增长 / > 80% 容量 → 处理不过来 |
| `completedTaskCount` | `getCompletedTaskCount()` | 斜率下降 → 吞吐退化 |
| **拒绝次数** | 自定义 handler 计数 | **任何非 0 都该告警** |
| **任务耗时** | `beforeExecute`/`afterExecute` | P99 > SLA |
| `largestPoolSize` | `getLargestPoolSize()` | 历史峰值，用于容量评估 |
| 线程创建/销毁速率 | 自定义 factory 计数 | 频繁抖动 → `keepAliveTime` 太短或流量抖动 |
| **队列等待时间** | 任务自带时间戳 | 直接反映"用户会等多久" |

**零成本埋点实现（重写钩子）：**

```java
public class InstrumentedPool extends ThreadPoolExecutor {
    private final LongAdder rejected = new LongAdder();
    private final Timer latency;

    @Override protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        ((AttrRunnable) r).setStartNanos(System.nanoTime());   // 或用 ThreadLocal
    }
    @Override protected void afterExecute(Runnable r, Throwable t) {
        latency.record(System.nanoTime() - startNanos);
        if (t != null) failures.increment();
        super.afterExecute(r, t);       // ★ 注意 submit 的异常在 Future 里（见 9.2）
    }
}
```

**三个"必须有"的告警规则（这是落点）：**

```
① rejected > 0（1 分钟内）     → P1，说明已经打满，业务在丢任务
② queueSize > 容量 × 0.8 持续 5 分钟   → P2，说明处理能力不足
③ 任务 P99 耗时 > SLA          → P2，说明任务本身或下游变慢
```

**延伸（高阶）**："怎么发现'线程池吃掉了异常'？"
→ 三个信号源：① 拒绝计数 ② `afterExecute` 的 `Throwable`（仅 `execute` 有效）③ **如果用了 `submit`，必须额外检查 `Future`**（见 9.2 的 `afterExecute` 补丁）。很多团队的事故复盘结论都是"任务静默失败，监控里只有 0 拒绝、0 积压，看不出任何异常"。

---

### Q35. 线程池"打满"有哪两种？分别怎么应急处理？
> **考核程度：★★★ 常考（应急能力题）** · 高频：阿 / 京 / 拼 / 滴

**参考答案：**

**两种"满"，语义完全不同：**

| 类型 | 现象 | 含义 | 应急动作 |
|---|---|---|---|
| **队列满但线程未满** | `queueSize == 容量`，`poolSize < max` | 缓冲耗尽，但**弹性还在**，池正在"加速"（扩容） | **不用慌**，这是设计内的行为。观察是否很快扩到 `max`；若长期卡在这里，说明 `max` 设得太低 |
| **线程满且队列也满** | `activeCount == max`，`queueSize == 容量`，`rejected > 0` | **真正饱和**，新任务被拒 | 立即降级/限流 |

**真正饱和时的五步应急（按顺序）：**

```
1. 【止血】上游限流：网关/队列层把流量砍到当前处理能力的 70%
2. 【扩容】水平扩容实例（若瓶颈不在单机 CPU）
3. 【降级】把非核心任务切到"丢弃/异步补偿"，保核心链路
4. 【定位】看 activeCount 高的线程在干什么 → jstack → 是慢在哪（DB? RPC? 锁?）
5. 【复盘】如果是"任务本身变慢"，调大线程池是错的方向（会打垮下游）
```

**一条重要的反直觉原则：**

```
线程池打满时，**第一反应绝不能是"调大线程数"**。
因为线程池是"下游的代言人"——它满了，通常说明下游（DB/RPC/Redis）已经到极限了。
调大线程池 = 给下游加压 = 加速雪崩。

正确顺序：先限流 → 再看是不是某类慢任务占满了池 → 再考虑隔离/扩容。
```

---
## D 组 · 场景、排查与工程实践（Q36–Q45）

### Q36. `ScheduledThreadPoolExecutor` 是怎么实现"定时"的？它有哪些隐藏行为？
> **考核程度：★★★★ 高频** · 高频：阿 / 京 / 滴 / 美

**参考答案：**

**改造点（它继承 `ThreadPoolExecutor`，但改了四处）：**

```
① workQueue 换成 DelayedWorkQueue（自己实现的、基于堆的延迟队列）
② maximumPoolSize 硬编码为 Integer.MAX_VALUE（因为队列无界，max 无意义）
③ corePoolSize 才是真正的并行度（默认 1！）
④ 重写 execute/submit，把任务包成 ScheduledFutureTask（带 triggerTime）
⑤ 重写 onShutdown，取消所有已排队的任务
```

**`DelayedWorkQueue.take()` 的精妙（这是核心）：**

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    for (;;) {
        RunnableScheduledFuture<?> first = queue[0];      // 堆顶 = 最早到期
        if (first == null) available.await();
        else {
            long delay = first.getDelay(NANOSECONDS);
            if (delay <= 0) return finishPoll(first);      // 到期 → 取走
            // ★ 没到期 → 睡到"到期那一刻"，而不是睡固定时间后重查
            available.awaitNanos(delay);
        }
    }
}
```

> **关键差异**：普通 `BlockingQueue.take()` 是"有元素就返回"；`DelayedWorkQueue` 是"**睡到堆顶元素到期的那一刻**"。这避免了轮询（`Thread.sleep(1)` 那种烧 CPU 的写法），精度由 `Condition.awaitNanos` 保证。

**周期任务的"重排"逻辑（容易被忽略）：**

```
scheduleAtFixedRate：
   任务执行完 → 计算下次执行时间 = 上次**计划时间** + period
   → 若已过期（任务耗时 > period）→ **立刻再次执行**（会堆积！）
   → 重新入堆

scheduleWithFixedDelay：
   任务执行完 → 下次执行时间 = **本次执行结束时间** + delay
   → 永远不会堆积
```

**三个隐藏行为（面试加分点）：**

| 隐藏行为 | 后果 | 修复 |
|---|---|---|
| **任务抛异常 → 该任务被取消，后续调度不再发生** | 定时任务"静默消失"，日志里可能只有一条异常 | 任务内部 `try-catch` 全包 |
| **`scheduleAtFixedRate` 在耗时 > 周期时会连续补跑** | CPU 飙高、任务重叠 | 用 `scheduleWithFixedDelay`，或加"上一次未完成则跳过"的标记 |
| **`corePoolSize` 默认 1** | `newScheduledThreadPool(1)` 是**单线程**，一个慢任务会拖住所有定时任务 | 显式指定 core（如 `N_cpu`） |

**延伸（高级）**："分布式环境下定时任务会重复执行吗？"
→ 会。`ScheduledThreadPoolExecutor` 是**单机**语义，多实例部署时每个实例都会跑。**必须用分布式调度（XXL-Job / ElasticJob / Quartz 集群模式）或加分布式锁。**

---

### Q37. `ForkJoinPool` 和 `ThreadPoolExecutor` 的本质区别是什么？work-stealing 是怎么工作的？
> **考核程度：★★★★ 高频** · 高频：字 / 阿 / 腾 / 华

**参考答案：**

**五个本质区别：**

| 维度 | `ThreadPoolExecutor` | `ForkJoinPool` |
|---|---|---|
| 任务模型 | 外部提交（任务独立） | **可分解**（`RecursiveTask` 自我 fork） |
| 队列结构 | **所有线程共享一个队列** | **每个线程一个双端队列（deque）** |
| 取任务方式 | 队列 FIFO，任意线程抢 | **自己从队头 LIFO 取**，其他线程从**队尾偷** |
| 阻塞/等待 | 线程阻塞即浪费 | `join` 时**创建补偿线程**或去偷别的任务 |
| 适用 | 通用异步 | **分治型 CPU 密集计算** |

**work-stealing 的完整机制（这部分是答案的核心）：**

```
每个 worker 线程维护一个 deque：

    worker-1 的 deque：  [新任务][任务][旧任务]
                          ↑头            ↑尾
    自己干活：从【队头】pop（LIFO）→ 缓存热、无竞争
    别人偷活：worker-2 从 worker-1 的【队尾】steal（FIFO）→ 减少锁冲突

★ 为什么"自己 pop 头、别人偷尾"？
   - 头尾分离 → 两者几乎不碰同一元素 → 无需重锁
   - LIFO 让刚 fork 的子任务优先执行 → 数据还在缓存里（局部性好）
   - FIFO 让偷走的任务是"最早 fork 的、粒度最大的" → 偷一次能偷到很多活

任务 fork 的树形结构：
        task
       /    \
    left    right      ← fork() 把子任务压入自己 deque
                      ← 然后自己处理 left，right 留在 deque 里等被偷
    join() → 若 right 还没被偷走 → 自己顺手做完（省一次调度）
           → 若被偷走了 → 等它完成（此时可 tryCompensate 创建补偿线程）
```

**为什么 `join` 不会死锁（关键差异）：**

```java
// ForkJoinTask.join() 遇到自己等待的任务时：
//   ① 如果等待的任务就在自己 deque 里 → 直接做掉，不用等
//   ② 如果被别的线程偷走了 → 自己"帮忙干别的"（去偷别人的任务）
//   ③ 如果实在没有可偷的 → 尝试创建补偿线程（tryCompensate）
// 所以 FJP 的并行度是"动态的"，不会因为阻塞而死锁
```

**适用边界（这是落点）：**

```
✅ 分治递归：归并排序、大数组并行处理、树遍历、并行流（parallelStream）
❌ IO 阻塞任务：FJP 会用"补偿线程"应对阻塞，但补偿是有限度的；
   大量阻塞会把 commonPool 拖垮（并行度 = CPU-1，见 Q38）
❌ 需要"全局 FIFO 顺序"的任务：work-stealing 打乱了顺序
```

**延伸（对比思维）**："为什么 `parallelStream()` 用 `commonPool`，而很多公司禁用？"
→ 因为它用 `ForkJoinPool.commonPool()`，**整个 JVM 共享**（并行度 = `CPU - 1`）。一个业务里的慢任务会拖垮所有用并行流的地方（**跨业务故障传染**）。且它无法指定线程名，出问题时 jstack 里全是 `ForkJoinPool.commonPool-worker-N`，**完全无法定位**。

---

### Q38. `CompletableFuture` 默认用哪个线程池？为什么生产上必须自定义？
> **考核程度：★★★ 常考（现代 Java 必问）** · 高频：字 / 阿 / 美 / 京

**参考答案：**

**默认行为：**

```java
CompletableFuture.supplyAsync(() -> doWork());
//                       ↑ 不传 executor 时 → ForkJoinPool.commonPool()

// 并行度（JDK 9+ 可配）：
//   默认 = Runtime.availableProcessors() - 1
//   可通过 -Djava.util.concurrent.ForkJoinPool.common.parallelism=N 调整
```

**为什么必须自定义（四条硬理由）：**

| 理由 | 说明 |
|---|---|
| **① 并行度太小** | `CPU - 1` 是给"纯计算"设计的。IO 任务（等 DB/RPC）会占满这些线程，**第 N 个任务只能排队** |
| **② 全局共享 → 故障传染** | 全 JVM 一个池。一个业务的慢任务会让所有业务一起卡死（**没有舱壁隔离**） |
| **③ 无法观测** | 线程名固定为 `ForkJoinPool.commonPool-worker-N`，无法按业务打点、无法按业务限流 |
| **④ 阻塞补偿有限** | FJP 遇到阻塞会尝试补偿线程，但补偿有上限（`maxSpares`），大量阻塞仍会饿死 |

**正确用法（生产模板）：**

```java
// 显式传入业务线程池
CompletableFuture<User> f = CompletableFuture
    .supplyAsync(() -> userService.get(id), bizPool)          // ★ 传池
    .thenApplyAsync(u -> enrich(u), bizPool)                   // ★ 每一步都传
    .exceptionally(ex -> { log.error("fail", ex); return null; });

// 注意：不传 executor 的 thenApply / thenAccept 会在"上一个任务完成的线程"里执行，
//       可能是 commonPool，也可能是 bizPool —— **行为不确定**，必须显式传
```

**三个常见的坑：**

1. **`thenApply`（不带 Async）** 在"完成它的那个线程"里同步执行 → 如果那个是 IO 线程，会阻塞它。
2. **`allOf` 之后的 `thenApply`** 通常在**最后一个完成的 future 的线程**里执行——可能是任意线程。
3. **`commonPool` 的任务抛异常** 会被包装进 `CompletionException`，`exceptionally` 能捕获，但**如果整个链一个都没挂异常处理，异常会被静默丢弃**。

**延伸（一句话总结）**：
> `CompletableFuture` 的默认池是"给你写 demo 用的"，生产环境**每一个异步阶段都必须显式指定线程池**，且**按业务隔离**。这条规则和"禁止用 Executors"是同一层级的工程底线。

---

### Q39. 线程池的 `corePoolSize` / `maximumPoolSize` 能动态调整吗？源码上有什么风险？
> **考核程度：★★★ 常考** · 高频：阿 / 美 / 滴

**参考答案：**

**可以，三个 setter：**

```java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0) throw new IllegalArgumentException();
    int delta = corePoolSize - this.corePoolSize;
    this.corePoolSize = corePoolSize;                       // ① 先赋值
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();                             // ② 缩小：中断空闲线程
    else if (delta > 0) {
        int k = Math.min(delta, workQueue.size());          // ③ 扩大：最多补到"队列长度"
        while (k-- > 0 && addWorker(null, true)) { if (workQueue.isEmpty()) break; }
    }
}

public void setMaximumPoolSize(int maximumPoolSize) { ... } // 调小不立刻杀线程，靠 getTask 淘汰
public void setKeepAliveTime(long time, TimeUnit unit) { ... }
```

**四个风险点：**

| # | 风险 | 说明 |
|---|---|---|
| ① | **调小不会立刻减线程** | 只是"中断空闲线程"，正在跑任务的要等任务结束 + `getTask` 重判才退出。**线程数是渐进下降的，不是瞬时** |
| ② | **调大只在队列有任务时才补** | `k = min(delta, queue.size())`，队列空时不预建。要立刻生效需配合 `prestartAllCoreThreads()` |
| ③ | **setter 不校验 `core <= max`** | 构造时有校验，setter 没有。**动态调参顺序错会出现 `core > max` 的非法中间态** |
| ④ | **无锁保护** | `corePoolSize` 不是 `volatile`，多线程同时 set 会读到脏值。**只应有一个"配置中心监听器"单线程调它** |

**正确的动态调参顺序（这是本题的落点）：**

```
调大：先 setMaximumPoolSize(大) → 再 setCorePoolSize(大)
调小：先 setCorePoolSize(小)   → 再 setMaximumPoolSize(小)
       ↑ 保证任何时刻都满足 core <= max

再配 keepAliveTime 收缩，最终由 getTask 自然收敛
```

**一个完整的"可动态调参"实现：**

```java
@Component
public class PoolTuner {
    @ApolloConfigChangeListener("threadpool")
    public void onChange(ConfigChangeEvent e) {
        // 串行执行，避免并发 set
        synchronized (this) {
            pool.setMaximumPoolSize(cfg.getMax());
            pool.setCorePoolSize(cfg.getCore());
            pool.setKeepAliveTime(cfg.getKeepAlive(), TimeUnit.SECONDS);
            pool.setRejectedExecutionHandler(cfg.getPolicy());
        }
    }
}
```

**延伸（警惕）**："动态调参是解药还是安慰剂？"
→ 它是**短期的止血工具**（大促临时扩线程），**不是根因解法**。长期看，反复调整说明"容量规划没做对"，应该回头算 QPS × 耗时 + 下游约束（见 Q28）。

---

### Q40. 出现 `OutOfMemoryError: unable to create native thread` 怎么排查？
> **考核程度：★★★★ 高频（故障诊断题）** · 高频：阿 / 华 / 京 / 滴

**参考答案：**

**先理解这条报错的含义：**

```
它不是"堆不够"，而是"第 5 步 pthread_create 失败"（见 3.1 的调用链）：
   Java Thread 对象创建成功 → JavaThread 创建成功 → **pthread_create 失败**

★ 所以调大 -Xmx 完全没用，反而可能更糟（内存更紧张）
```

**五个可能原因，按概率排序：**

| # | 原因 | 检查命令 |
|---|---|---|
| ① | **线程数真的太多**（`max=Integer.MAX_VALUE` 的池） | `jstack | grep 'pool-' | wc -l`；`ps -eLf | wc -l` |
| ② | **`vm.max_map_count` 不够**（每线程约 1~2 个 VMA） | `sysctl vm.max_map_count`（默认 65530）；`cat /proc/<pid>/maps | wc -l` |
| ③ | **cgroup 的 pids 限制**（K8s 容器） | `cat /sys/fs/cgroup/pids.max`；`cat /sys/fs/cgroup/pids.current` |
| ④ | **`ulimit -u` / `kernel.threads-max` / `pid_max`** | `ulimit -u`；`sysctl kernel.threads-max kernel.pid_max` |
| ⑤ | **内存真的不够**（栈的物理内存 + 内核对象） | `free -h`；`cat /proc/<pid>/status | grep -i threads` |

**排查步骤（可直接背的流程）：**

```
1. 先看"到底有多少线程"：
      jstack <pid> | grep -c '^"'          ← Java 线程数
      ls /proc/<pid>/task | wc -l           ← 内核 task 数（应基本一致）
2. 按线程名前缀分组统计，找出"谁开的"：
      jstack <pid> | grep '^"' | sed 's/"//' | awk -F'-' '{print $1}' | sort | uniq -c | sort -rn
3. 检查内核限制（上表 ②③④）
4. 检查是否有"线程池未 shutdown"导致的泄漏（定时任务 / 每次请求新建池）
5. 检查是否用了 newCachedThreadPool / max=MAX_VALUE
```

**修复（按性价比排序）：**

```
① 给所有线程池设 max 上限（最直接）
② 用有界队列 + 合理拒绝，避免任务堆积触发"扩容到 max"
③ 检查是否有"每次请求都 new 一个线程池且不 shutdown"的代码（泄漏）
④ K8s 场景调大 pids.max（但优先减线程）
⑤ 极端情况调大 vm.max_map_count（治标）
⑥ 长期方案：IO 密集任务改用虚拟线程（不占内核线程）
```

**延伸（一句话点题）**：
> **这条 OOM 是"线程数超标"的症状，不是"内存不足"的症状。** 看到它就想到"某处线程池失控"，而不是"该加内存"。

---

### Q41. 线上 CPU 100%，你怎么定位到具体是线程池的哪个任务？
> **考核程度：★★★★★ 必考（排查经典题）** · 高频：全部大厂

**参考答案（这是有标准动作的题，必须答得出命令序列）：**

**第一步：找到"哪个线程"在烧 CPU（关键：PID 与 LWP 的区别）**

```bash
top -H -p <pid>            # -H = 按线程（LWP）展示
# 找到 CPU 最高的那个 LWP，比如 12345
```

**第二步：LWP → 十六进制 → jstack 里的 nid**

```bash
printf '%x\n' 12345        # → 3039
jstack <pid> | grep -A 30 'nid=0x3039'
```

> **为什么必须转十六进制**：`jstack` 输出里的 `nid=0x...` 是十六进制，而 `top -H` 显示的是十进制。**这一步是"CPU 高排查"的核心动作，背后的原理是"LWP ≠ PID"（见 2.2）**。

**第三步：看栈判断在干什么**

```
"pool-1-thread-3" #23 prio=5 os_prio=0 tid=0x... nid=0x3039 runnable
   java.lang.Thread.State: RUNNABLE
        at com.xxx.BizService.calc(BizService.java:88)      ← 定位到代码行
        at java.util.concurrent.ThreadPoolExecutor.runWorker(...)
   ← RUNNABLE + 高 CPU = 真在算（死循环、复杂计算、正则回溯、JSON 大对象序列化）
```

**四种典型结论（这张表是加分项）：**

| 栈特征 | 结论 |
|---|---|
| `RUNNABLE` + 业务代码 + CPU 高 | **真在计算**：死循环 / 正则回溯 / 大量序列化 / 大对象 hash |
| `RUNNABLE` + `SocketInputStream.read` + CPU 低 | **在等 IO**（Java 把 IO 阻塞也标 RUNNABLE，见 2.10）→ 不是 CPU 问题 |
| 大量线程 `BLOCKED` 在同一个锁 | **锁竞争**（不是 CPU 高，而是"CPU 空转 + 线程堆积"） |
| 大量线程 `WAITING` 在 `Future.get()` | **线程饥饿死锁**（见 Q24） |

**第四步：多次采样确认（单次 jstack 会骗人）**

```bash
for i in {1..5}; do jstack <pid> > /tmp/js_$i.txt; sleep 1; done
# 对比 5 次的栈，同一个方法反复出现 → 就是热点
# （这就是"采样分析"的基本原理，arthas 的 profiler 也是这个思路）
```

**arthas 一键版（生产首选）：**

```bash
thread -n 3          # 最忙的 3 个线程的栈
thread -b            # 找出阻塞其他线程的"元凶"线程
profiler start / profiler stop --format html    # 火焰图
```

**延伸（线程池视角的独特价值）**：
> 如果热点线程名是 `pool-1-thread-N`，说明**问题出在线程池里的任务**。这时要追问三件事：① 这个池属于哪个业务？② 池的 `max` 是多少、任务是否被"超量提交"？③ 任务是"CPU 密集型"却被配置在大 IO 池里？（池类型与任务类型不匹配是常见事故。）

---

### Q42. 在线程池里用 `@Transactional` / Spring 上下文为什么会失效？
> **考核程度：★★★★ 高频（Spring + 并发交叉题）** · 高频：阿 / 美 / 京 / 拼

**参考答案：**

**失效的根因：Spring 的"上下文"都挂在 `ThreadLocal` 上。**

```
@Transactional 依赖：
   TransactionSynchronizationManager.threadLocals
     ├─ resources：DataSource → ConnectionHolder（**当前线程绑定的连接**）
     └─ synchronizations

RequestContextHolder：请求上下文（request/response）
SecurityContextHolder：认证信息
MDC：日志 traceId

★ 全部是 ThreadLocal！

主线程开启事务 → 事务上下文绑在【主线程】的 ThreadLocal 上
          ↓
        提交异步任务 → 任务在【池中另一个线程】执行
          ↓
        那个线程的 ThreadLocal 是空的
          ↓
        @Transactional 注解在异步方法上 → 它开启的是一个**全新的、独立的连接和事务**
        （如果方法上有 @Transactional，而且是通过代理调用的）
          ↓
        ❌ 主线程的事务回滚，异步任务的事务不会回滚（反之亦然）
        ❌ 异步任务读不到主线程未提交的数据（不同连接、不同事务）
```

**四种典型表现：**

| 表现 | 原因 |
|---|---|
| 主事务回滚，异步任务的数据已落库 | 异步任务用了**自己的连接 + 自己的事务**（已自动提交） |
| 异步任务报"连接已关闭" | 主线程提交任务后事务结束，连接归还，异步线程还在用（如果传了 `Connection` 引用） |
| 异步方法里 `@Transactional` 完全不生效 | **同类内部调用不走代理**（`this.method()` 绕过了 AOP），或异步方法不是 Spring Bean 的方法 |
| 日志里 traceId 丢了 | MDC 是 ThreadLocal（且非 `InheritableThreadLocal`）→ 池化线程拿不到 |

**四条正确做法：**

```
① 【最推荐】异步任务里独立开启事务：把异步方法放到独立的 @Service Bean，
   通过注入的代理调用（确保走 AOP），让它自己 new 一个事务。
   → 语义清晰：主事务和异步事务本就应该是独立的（分布式事务问题）

② 需要"主事务提交后才执行"→ 用 TransactionSynchronizationManager
   .registerSynchronization(afterCommit 里再提交异步任务)
   ★ 这是最常被忽略的关键点：**不要在事务未提交时就把任务扔进线程池**

③ 上下文传递：用 TTL（TransmittableThreadLocal）或显式把 userId/traceId
   作为任务参数传进去，不要依赖线程继承

④ MDC：任务包装器里在 before/after 手动 put/clear
```

**延伸（这才是面试官想听的判断）**：
> **"异步 + 事务"本身就是一对矛盾**：事务要"全有或全无"，异步要"解耦、不等待"。所以正确的问题不是"怎么让异步任务参与主事务"，而是"**这个操作到底该不该异步**"。
> - 必须一致的 → **不能异步**，用本地事务或事务消息（RocketMQ 半消息）
> - 可以最终一致的 → 异步 + 补偿/对账（这正是分布式事务的解决思路）

---

### Q43. 如何设计一个"同 key 有序、不同 key 并行"的线程池？
> **考核程度：★★★ 常考（设计题）** · 高频：阿 / 美 / 京 / 拼

**参考答案：**

**问题定义：**
消息/任务按 `orderId` 分区，同一个 `orderId` 必须串行（保序），不同 `orderId` 之间要并行（吞吐）。

**方案一：按 key 取模路由到 N 个单线程池（最常用）**

```java
public class OrderedExecutor {
    private final ThreadPoolExecutor[] pools;

    public OrderedExecutor(int shards, int queuePerShard) {
        pools = new ThreadPoolExecutor[shards];
        for (int i = 0; i < shards; i++) {
            pools[i] = new ThreadPoolExecutor(
                1, 1, 0, TimeUnit.MILLISECONDS,              // ★ 单线程 = 天然串行
                new LinkedBlockingQueue<>(queuePerShard),
                new NamedThreadFactory("ordered-" + i),
                new ThreadPoolExecutor.CallerRunsPolicy());
        }
    }

    public void execute(String key, Runnable task) {
        int idx = Math.floorMod(key.hashCode(), pools.length);   // ★ 同 key 同池
        pools[idx].execute(task);
    }
}
```

| 优点 | 缺点 |
|---|---|
| 实现简单、语义清晰 | **分片不均匀**（热点 key 会把某个分片打满，其他分片空闲） |
| 天然保序 | 分片数 = 并行度上限，无法动态伸缩 |
| 无锁 | 需要保证 `hashCode` 稳定（**不要用 `String.hashCode` 对可变对象**） |

**方案二：`key → 单线程队列` 的"串行执行器"（精准保序）**

```java
// 同 key 的任务进同一个队列，由同一个"虚拟执行器"按序取出
// 用 ConcurrentHashMap<key, SerialQueue> + 一个共享线程池驱动
// SerialQueue 用 AtomicBoolean 标记"是否已有线程在处理"（避免并发执行）
// 关键：处理完后要"重新提交自己"（链式驱动）
```
优点：不分片、无热点问题；缺点：实现复杂，容易出"漏提交"导致任务卡死。

**方案三：MQ 顺序消息（生产最主流）**

```
Kafka / RocketMQ 的顺序消息：
   producer 按 key 选分区 → 同 key 落同一分区 → 消费端单线程消费该分区
   ★ 本质和"方案一"一样，只是把"分片"放到 MQ 层做了
```

**面试加分点（三条）：**

1. **"队列有界 + 热点 key 会导致队头阻塞"**：同 key 的慢任务会阻塞该 key 的所有后续任务（这是保序的必然代价），所以**保序粒度越细越好**（能按 `orderId` 就别按 `userId`）。
2. **"保序 + 重试"会打架**：重试会打乱顺序，需要"失败即阻塞该 key 的队列"（顺序消费的标准做法）。
3. **`Math.floorMod` 而不是 `%`**：负数 hash 用 `%` 会得到负索引，直接 `ArrayIndexOutOfBoundsException`。

---

### Q44. 如果让你负责"公司级线程池治理"，你会怎么做？（开放设计题）
> **考核程度：★★★ 常考（架构开放题）** · 高频：阿 / 美 / 京（P7+/3-2 以上）

**参考答案（分五层，每层都要有"为什么"）：**

**第一层：准入（统一 SDK，禁止裸用）**

```
① 提供 ThreadPoolBuilder（强制：有界队列、线程名、拒绝策略、监控）
② 静态扫描拦截 Executors.*（ArchUnit / SonarQube 自定义规则）
③ Code Review checklist：新增线程池必须在"线程池台账"里登记（业务、QPS、耗时、下游）
```

**第二层：隔离（舱壁模式）**

```
按"依赖"分组，而不是按"业务"分组：
  - 每个下游（DB-A / Redis-B / RPC-C）一个池 → 一个下游挂了只影响自己的池
  - 核心链路与非核心链路物理隔离
  - 定时任务池独立（避免拖累在线请求）

★ 依据：线程池是"下游的代言人"，按下游隔离才是真正的故障隔离
```

**第三层：容量（可计算、可验证）**

```
每个池必须有：
  - 容量推导表：峰值QPS × 平均耗时 × 安全系数，并注明下游约束
  - 压测报告：拐点在哪里（吞吐不再涨、P99 开始恶化）
  - 定期复核：业务变化后重算（每季度）
```

**第四层：可观测（三类指标 + 三条告警）**

```
指标：poolSize / activeCount / queueSize / rejectCount / taskLatency / queueWaitTime
告警：拒绝>0（P1）；队列>80%持续5min（P2）；P99>SLA（P2）
看板：按业务/按池展示，能看到"哪个池在恶化"
```

**第五层：治理闭环（动态 + 演练）**

```
- 配置中心动态调参（带顺序约束）
- 定期"打满演练"：验证拒绝策略是否真的按预期降级（很多团队从没验证过）
- 事后复盘模板：是"任务变慢"还是"流量超预期"还是"池配置错"？
```

**收尾判断（能让面试官点头的一句）：**
> **线程池治理的本质不是"调参数"，而是"让'并发度'成为架构里的一等公民"**：谁在并发、并发多少、下游能承受多少、超了怎么办——这四件事必须在设计阶段就写清楚，而不是等 OOM 了再来看 `jstack`。

---

### Q45. 虚拟线程来了，线程池会被淘汰吗？说说你的技术判断。
> **考核程度：★★★ 常考（趋势判断题）** · 高频：字 / 阿 / 腾 / 华

**参考答案（要有明确观点，不要骑墙）：**

**观点：线程池不会消失，但会"降级"——从"复用器"退化为"限流器"。**

**分三类场景判断：**

| 场景 | 未来方案 | 线程池的作用 |
|---|---|---|
| **IO 密集型**（RPC、DB、HTTP 调用） | **虚拟线程**（`newVirtualThreadPerTaskExecutor()`） | 不再需要"几百个线程"，但**仍需 `Semaphore` 或有界池限制对下游的并发** |
| **CPU 密集型**（计算、编解码） | **小线程池**（`N_cpu + 1`） | ✅ 完全不可替代（虚拟线程跑 CPU 任务毫无优势） |
| **需要排队语义/背压**（批量任务、削峰） | **有界队列 + 线程池** | ✅ 不可替代（虚拟线程没有"队列"概念） |

**三条支撑理由：**

1. **虚拟线程解决的是"阻塞的代价"，不是"并发的上限"。** 它让"一个请求一个线程"重新变得合理（不用再为了省线程而写复杂的异步回调），但**它不提供"最多 20 个任务同时打 DB"这种能力**——那必须靠信号量或有界池。
2. **CPU 资源仍然有限。** 100 万个虚拟线程同时做计算，载体线程还是只有 `N_cpu` 个，只是把"排队"从内核搬到了用户态调度器。**并行度控制依然必要。**
3. **线程池是"可观测、可治理"的载体。** 指标（queueSize / rejectCount / poolSize）、拒绝策略、动态调参，这套治理体系是虚拟线程暂时没有的。**企业级系统需要"看得见、控得住"。**

**一个精准的预测（能说出这句，档次就上去了）：**

> **未来 3~5 年，"线程池"这个词的含义会从"ThreadPoolExecutor"泛化为"ConcurrencyLimiter"**：IO 任务用虚拟线程承载，用信号量/有界队列限流；CPU 任务用小 `ThreadPoolExecutor`。**`ThreadPoolExecutor` 会瘦身为"CPU 任务的专用执行器 + 限流器"，而不再是"所有异步任务的默认归宿"。**

**必须提的两个迁移坑：**

```
① synchronized 会 pin 载体线程（JDK 21；JDK 24 的 JEP 491 已修复）
   → 迁移前先排查代码里的 synchronized 块内是否有阻塞调用
② ThreadLocal 成本被放大：百万虚拟线程 = 百万份 ThreadLocalMap
   → 改用 ScopedValue（JDK 21 预览）或减少 ThreadLocal 使用
```

---
## 附录 A 一页速查表

### A.1 参数速查

```java
new ThreadPoolExecutor(
    corePoolSize,        // 常态并发；懒创建；默认不回收
    maximumPoolSize,     // 突发上限；★ 无界队列时永远不生效
    keepAliveTime,       // 非核心线程空闲回收时间（allowCoreThreadTimeOut 时对 core 生效）
    unit,
    workQueue,           // ★ 必须有界（LinkedBlockingQueue 默认 Integer.MAX_VALUE）
    threadFactory,       // ★ 必须命名（jstack 可读性）
    handler              // ★ 必须有明确语义（默认 AbortPolicy）
);
```

### A.2 状态速查

| 状态 | 高3位 | 新任务 | 队列任务 | 执行中 | 触发 |
|---|---|---|---|---|---|
| `RUNNING` | 111 | 接受 | 执行 | 不打断 | 初始 |
| `SHUTDOWN` | 000 | 拒绝 | **跑完** | 不打断 | `shutdown()` |
| `STOP` | 001 | 拒绝 | **丢弃** | **中断** | `shutdownNow()` |
| `TIDYING` | 010 | — | 空 | 无（wc=0） | `tryTerminate()` CAS |
| `TERMINATED` | 011 | — | 空 | 无 | `terminated()` 执行完 |

### A.3 关键方法速查

| 方法 | 语义 | 注意 |
|---|---|---|
| `execute(R)` | 执行，异常抛到线程顶层 | 异常 → 线程死 → **池会补线程** |
| `submit(R/C)` | 执行并返回 `Future` | ★ **异常被 `Future` 吞**，必须 `get()` |
| `shutdown()` | 温和关闭，跑完队列 | 也要 `awaitTermination` |
| `shutdownNow()` | 强制关闭，丢弃队列并中断 | 返回未执行任务列表；**停不掉 CPU 死循环** |
| `awaitTermination(t,u)` | 阻塞等待终止 | ★ **必须给超时** |
| `prestartAllCoreThreads()` | 预热 | 消除首波延迟 |
| `allowCoreThreadTimeOut(true)` | 核心线程也可回收 | 需 `keepAliveTime > 0`；会缩到 0 |
| `purge()` | 清理已取消的任务（队列里的 `FutureTask`） | 配合 `Future.cancel()` 防内存泄漏 |
| `getActiveCount()` | 忙线程数 | 遍历 `workers` 数 `isLocked()` |

### A.4 源码关键五段（能默写等级 ★★★）

```java
// ① 执行三步
if (workerCountOf(c) < corePoolSize) { if (addWorker(command, true)) return; c = ctl.get(); }
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (!isRunning(recheck) && remove(command)) reject(command);
    else if (workerCountOf(recheck) == 0) addWorker(null, false);
} else if (!addWorker(command, false)) reject(command);

// ② 复用循环
while (task != null || (task = getTask()) != null) { w.lock(); task.run(); w.unlock(); }

// ③ 收缩判断
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) { ... return null; }

// ④ 级联终止
if (workerCountOf(c) != 0) { interruptIdleWorkers(ONLY_ONE); return; }

// ⑤ 忙闲探测
if (!t.isInterrupted() && w.tryLock()) { t.interrupt(); w.unlock(); }
```

### A.5 延迟量级速查（建立工程直觉）

| 操作 | 量级 |
|---|---|
| `clone()` 创建一个线程 | 30~80 μs |
| 同核线程上下文切换（直接） | 1~3 μs |
| 上下文切换（含 Cache 冷启动的实际损失） | 5~10 μs 等效 |
| `futex` wake → 线程真正跑起来 | 10~100 μs（负载高时更久） |
| 无竞争 CAS | ~20 ns |
| 无竞争 `synchronized`（偏向/轻量锁） | ~20~50 ns |
| 一次 RPC（同机房） | 0.5~2 ms |
| 一次 SSD 随机读 | 50~100 μs |
| 一次内存访问 / Cache Miss / 缺页 | 1 ns / 10~100 ns / 10 μs 级 |

> **用法**：面试被问"线程池为什么快"，可以答"**创建线程 30~80 μs，而唤醒一个空闲线程只要 10~100 μs 且可复用，同时省掉内核对象构造与栈 VMA 建立**"——有数字的回答，可信度高一个量级。

### A.6 内核参数速查

| 参数 | 命令 | 默认 | 与线程的关系 |
|---|---|---|---|
| `vm.max_map_count` | `sysctl vm.max_map_count` | 65530 | **每线程 2 个 VMA** → 约 3 万线程撞墙 |
| `kernel.pid_max` | `sysctl kernel.pid_max` | 32768 / 4194304 | 线程数上限 |
| `kernel.threads-max` | `sysctl kernel.threads-max` | ≈ 内存页/8 | 系统级线程上限 |
| `pids.max`（cgroup） | `cat /sys/fs/cgroup/pids.max` | 因集群而异 | **K8s 里最常见的隐形限制** |
| `ulimit -u` | `ulimit -u` | limits.conf | 单用户进程/线程数 |
| `sched_latency_ns` | `sysctl kernel.sched_latency_ns` | 6 ms | 调度周期 |
| `min_granularity_ns` | `sysctl kernel.sched_min_granularity_ns` | 0.75 ms | 最小时间片 |

### A.7 "错误做法 → 正确做法"对照（可直接贴到团队规范里）

| ❌ 错误 | ✅ 正确 |
|---|---|
| `Executors.newFixedThreadPool(n)` | `new ThreadPoolExecutor(n, n, 0, MS, new LinkedBlockingQueue<>(cap), namedFactory, AbortPolicy)` |
| `new LinkedBlockingQueue<>()`（不传容量） | `new LinkedBlockingQueue<>(2000)` |
| `submit(task)` 从不 `get()` | 用 `execute`，或 `get()` / 统一异常兜底 |
| 线程名用默认 `pool-N-thread-M` | `new NamedThreadFactory("order-query")` |
| 所有业务共用一个池 | 按下游/按核心非核心**物理隔离** |
| `shutdown()` 后不 `awaitTermination` | `shutdown()` + `awaitTermination(30s)` + 兜底 `shutdownNow()` |
| 打满时调大 `maximumPoolSize` | 先限流，再定位慢任务/慢下游 |
| `ThreadLocal` 不 `remove()` | `finally { tl.remove(); }` |
| 用 `setPriority` 做业务优先级 | 用独立池 / `PriorityBlockingQueue` / 分片 |
| 认为 `RUNNABLE` = 吃 CPU | 看 `top -H`，结合 `nid` 定位 |

---

## 附录 B 自测清单（按层级打分）

> 用法：**能不看答案说出 ≥ 80%，说明这一层掌握了**。面试前逐层过一遍。

### L1 · 会用（初级 / 校招）

- [ ] 说出七大参数及其含义
- [ ] 画出"核心 → 队列 → 扩容 → 拒绝"的三步流程
- [ ] 说清四种拒绝策略的差异，以及 `CallerRunsPolicy` 何时危险
- [ ] 知道 `execute` 与 `submit` 的异常差异
- [ ] 知道为什么禁 `Executors`
- [ ] 会给线程池起有意义的名字
- [ ] 知道 `shutdown` 与 `shutdownNow` 的语义差异

### L2 · 懂原理（中级 / P6）

- [ ] 说清 `ctl` 为什么合体、29 位/3 位的来历、状态值为何是负数在前
- [ ] 说清"线程复用"的机制（`Worker` + `while` + `getTask`）
- [ ] 说清核心线程会不会销毁、`allowCoreThreadTimeOut` 的联动点
- [ ] 说清 `keepAliveTime` 在 `getTask` 里怎么生效
- [ ] 会算线程数（Goetz 公式 / Little 定律 / 下游倒推三重）
- [ ] 说清队列容量与"延迟承诺"的关系
- [ ] 说清 `ThreadLocal` 在线程池里的两类问题
- [ ] 会写优雅关闭代码

### L3 · 懂源码（高级 / P7）

- [ ] `execute` 第 2 步 `recheck` 修复的两个竞态
- [ ] 为什么"先加线程再入队"而不是反过来
- [ ] `addWorker` 的双层循环 + 状态闸门的混合条件
- [ ] `Worker` 为什么用 AQS 而非 `ReentrantLock`（三层理由 + `setState(-1)`）
- [ ] `shutdown` 如何判定"忙/闲"（`tryLock`）
- [ ] `runWorker` 里"清理中断位"那段代码的四种组合
- [ ] `getTask` 返回 null 的三条路径 + `timedOut` 跨循环的意义
- [ ] `completedAbruptly` 的两种语义与"补线程"
- [ ] `tryTerminate` 为什么 `ONLY_ONE`（级联传播）
- [ ] `firstTask=null` 的两个用途（预热 + 补位）

### L4 · 懂内核 / 懂架构（专家 / P8+）

- [ ] 说清 `task_struct` / `clone` 的 flag / 1:1 模型
- [ ] 说清线程切换为什么不换 CR3、贵在哪（Cache/TLB/分支预测）
- [ ] 用 CFS 参数算出"线程太多导致延迟放大"的具体倍数
- [ ] 说清 `futex` 的快慢路径，以及"Java 阻塞的原点"
- [ ] 说清 `max_map_count` 与"每线程 2 个 VMA"的关系
- [ ] 说清 `Java RUNNABLE` 包含了 OS 层的 `S`（IO 阻塞）
- [ ] 说清 `nid`/`LWP`/`tgid` 的区别，能给出排查命令序列
- [ ] 说清虚拟线程的 mount/unmount、pinning、以及"为什么仍需要限流"
- [ ] 能设计"公司级线程池治理方案"（准入/隔离/容量/监控/闭环）
- [ ] 能判断"保序 vs 并行"的设计取舍，并给出分片方案

---

## 附录 C 源码阅读路线（想变强就按这个顺序读）

```
第 1 遍：只读 execute() + addWorker()        ← 建立"三步流程"的骨架
第 2 遍：读 runWorker() + getTask()          ← 建立"复用 + 回收"的骨架
第 3 遍：读 processWorkerExit + tryTerminate  ← 建立"状态机收敛"的骨架
第 4 遍：读 shutdown/shutdownNow + interruptIdleWorkers  ← 建立"关闭"的骨架
第 5 遍：通读全文，重点看注释              ← Doug Lea 的注释里藏着所有设计意图
第 6 遍：读 Worker 的 AQS 部分 + setState(-1) ← 最后读，因为最难

配套阅读：
  - AbstractExecutorService（submit/invokeAll 的模板方法）
  - FutureTask（为什么能吞异常、能取消）
  - LinkedBlockingQueue / ArrayBlockingQueue（队列的锁模型）
  - ScheduledThreadPoolExecutor（延迟队列的变体）
  - ForkJoinPool（另一套完全不同的并发哲学）
  - JDK 21 的 VirtualThread / Continuation（未来）
HotSpot 侧（可选，但读了就高人一等）：
  - thread.cpp（JavaThread 生命周期）
  - os_linux.cpp（os::create_thread / futex 封装）
  - synchronizer.cpp（ObjectMonitor，理解 synchronized 的重量级路径）
```

---

## 结语：把这张"一句话地图"记住

```
业务：execute(task)
   ↓
线程池：ctl(CAS) 决定"开线程 / 排队 / 扩容 / 拒绝"
        Worker(AQS) 标记"忙 / 闲"，while 循环实现复用
        BlockingQueue(futex) 提供"缓冲 + 睡眠"
   ↓
JVM：Thread.start0 → JavaThread → OSThread，栈 mmap + guard page
   ↓
libc：pthread_create / pthread_cond_wait
   ↓
内核：clone(CLONE_VM|CLONE_THREAD) 造 task_struct
     CFS 用 vruntime 红黑树决定谁上 CPU
     futex 让空闲线程真正睡下（0 CPU）

★ 记住这条链，你就能回答关于线程池的任何一个"为什么"。
```

---

*本文档基于 JDK 8/11/17/21 的 `ThreadPoolExecutor` 实现与 Linux 内核（CFS / futex）通用行为整理。具体行号与参数默认值可能随 JDK 版本与内核版本略有差异，核心逻辑一致。*










