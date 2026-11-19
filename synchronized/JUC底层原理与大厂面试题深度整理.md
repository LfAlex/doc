# JUC 并发编程底层原理与大厂面试题深度整理

> **定位**：从「怎么用」推进到「CPU 到底执行了什么指令、JVM 到底改了哪个字段」。
> **主线版本**：JDK 8 / 11 / 17（AQS、ConcurrentHashMap、ThreadPoolExecutor 在此区间实现基本一致）；JDK 19+ 虚拟线程相关差异单独标注。
> **读法建议**：第 1～3 章是地基（不懂屏障与对象头，后面全是背结论）；第 4～8 章是 JUC 主体；第 9 章是排障手段；第 10 章是面试题库，可按「公司 / 考核等级」跳读。

---

## 目录

| 章节 | 内容 | 重要度 |
|---|---|---|
| 0 | 全景图：JUC 到底由哪几块拼起来 | ★★★★★ |
| 1 | 硬件与操作系统底座：MESI、伪共享、内存屏障、CAS | ★★★★★ |
| 2 | JMM 与 happens-before：可见性/有序性的形式化定义 | ★★★★★ |
| 3 | volatile 端到端：字节码 → JIT → 汇编 → 硬件 | ★★★★★ |
| 4 | synchronized 全链路：对象头、锁升级、ObjectMonitor | ★★★★★ |
| 5 | AQS 深度拆解：CLH 变体、状态机、Condition、传播 | ★★★★★ |
| 6 | 基于 AQS 的组件：ReentrantLock / RWLock / StampedLock | ★★★★☆ |
| 7 | 并发容器：ConcurrentHashMap、COW、CLQ、BlockingQueue 家族 | ★★★★★ |
| 8 | 线程池：ThreadPoolExecutor、Scheduled、ForkJoin、CompletableFuture | ★★★★★ |
| 9 | 原子类与累加器：CAS 的代价、LongAdder 的分片哲学 | ★★★★☆ |
| 10 | 同步工具与线程基础设施：Latch/Barrier/Semaphore/Phaser/LockSupport/ThreadLocal | ★★★★☆ |
| 11 | 排障与调优：死锁、线程 dump、JMH、常见反模式 | ★★★☆☆ |
| 12 | 大厂面试题库：公司 × 考核等级 × 详解 × 延伸 | ★★★★★ |

---

# 第 0 章 全景图：JUC 到底由哪几块拼起来

## 0.1 一句话结论

**JUC 包（`java.util.concurrent`）99% 的同步能力，最终都收敛到三个原语：`volatile` 的读写、`CAS` 的原子修改、`park/unpark` 的线程阻塞。**

- `synchronized` / `ObjectMonitor` 是 JVM 内建的另一条路径（管程，monitor）。
- JUC 的作者 Doug Lea 在 `AbstractQueuedSynchronizer`（AQS）里把「一个 `volatile int state` + 一条双向等待队列 + `LockSupport.park`」组合成了通用的同步框架。
- 之后所有同步器（ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock、ThreadPoolExecutor 的 Worker、FutureTask）都是「换一个 `state` 的含义 + 换一段 `tryAcquire/tryRelease` 逻辑」。

## 0.2 分层视图（自顶向下）

```text
┌─────────────────────────────────────────────────────────────────────┐
│ L5 应用/工具层                                                       │
│   CompletableFuture  ForkJoinPool  Executors  ConcurrentHashMap     │
│   CopyOnWriteArrayList  DelayQueue  SynchronousQueue  LongAdder     │
├─────────────────────────────────────────────────────────────────────┤
│ L4 同步器层（策略层）                                                 │
│   AQS(state + CLH 变体队列 + ConditionObject)                        │
│   ├─ 独占：ReentrantLock / Mutex                                     │
│   ├─ 共享：Semaphore / CountDownLatch / ReadWriteLock 的读锁          │
│   └─ 自定义：FutureTask / ThreadPoolExecutor.Worker                  │
├─────────────────────────────────────────────────────────────────────┤
│ L3 原语层（机制层）                                                   │
│   CAS: Unsafe.compareAndSwapXxx → VarHandle(JDK9+)                   │
│   阻塞: LockSupport.park/unpark → Parker(信号量)                     │
│   可见性: volatile 字段读写                                          │
│   管程: synchronized → monitorenter/exit → ObjectMonitor             │
├─────────────────────────────────────────────────────────────────────┤
│ L2 JVM 层                                                            │
│   对象头 Mark Word / Klass Pointer、字节码、ACC_VOLATILE、            │
│   OrderAccess 内存屏障、偏向锁撤销、逃逸分析/锁消除、Safepoint         │
├─────────────────────────────────────────────────────────────────────┤
│ L1 OS 层                                                             │
│   pthread_mutex / condvar、futex、clone 线程、调度器、TLS            │
├─────────────────────────────────────────────────────────────────────┤
│ L0 硬件层                                                            │
│   多核、私有 L1/L2、共享 L3、MESI、Store Buffer/Invalidate Queue、    │
│   乱序执行、lock 前缀、cmpxchg、mfence/lfence/sfence、dmb ish         │
└─────────────────────────────────────────────────────────────────────┘
```

**关键认知**：一次 `lock.lock()` 的物理路径，是从 L5 一路打到 L0 的——`state` 的 CAS 是 `lock cmpxchg` 指令，失败后的阻塞是 `pthread_cond_wait`（Linux 上最终是 `futex(FUTEX_WAIT)` 系统调用）。面试里能把这四层串成一条线，就是「懂底层」和「会背 API」的分水岭。

## 0.3 组件地图（按「state 是什么」归类）

| 组件 | state 的含义 | 模式 | 队列作用 |
|---|---|---|---|
| `ReentrantLock` | 重入次数（0=空闲） | 独占 | 抢锁失败的线程排队 |
| `ReentrantReadWriteLock` 读锁 | 高 16 位：读锁持有数 | 共享 | 同上，但可批量唤醒 |
| `ReentrantReadWriteLock` 写锁 | 低 16 位：写重入数 | 独占 | 同上 |
| `Semaphore` | 剩余许可数 | 共享 | 拿不到许可排队 |
| `CountDownLatch` | 剩余计数 | 共享 | await 的线程排队 |
| `ThreadPoolExecutor.Worker` | 0（仅借用 AQS 的独占标志） | 独占 | 不排队，只用 `tryLock` 做「是否空闲」判断 |
| `FutureTask` | 任务运行状态 | 独占 | `get()` 的线程排队等待完成 |
| `SynchronousQueue` | 不用 AQS（自研栈/队列） | — | 生产者/消费者直接配对 |

> **延伸：为什么 ThreadPoolExecutor 的 Worker 要继承 AQS？**
> 它并不需要排队，只需要一个「线程是否正在执行任务」的**可中断的互斥标记**。`shutdown()` 只能中断空闲线程，判断「空闲」的方式就是 `tryLock()` 成功。用 AQS 是为了复用「不可重入的独占锁」这一现成语义（`tryAcquire` 无条件 CAS 0→1，`tryRelease` 无条件置 0，不允许重入，避免任务内部再次 lock 造成误判）。这是一个「把 AQS 当布尔标志用」的经典非典型用法。

---

# 第 1 章 硬件与操作系统底座

> 这一章回答一个问题：**「另一个 CPU 核写了一个变量，我这颗核凭什么能看见？」**
> 答案是：缓存一致性协议（MESI）+ 内存屏障（阻止重排与 Store Buffer 延迟）。这两个东西合起来，才是 Java 里 `volatile` 的全部物理本质。

## 1.1 存储层次与延迟量级

```text
寄存器             ~0.3 ns      1 cycle
L1 (32~64KB)       ~1 ns        ~4 cycles      每核私有
L2 (256KB~1MB)     ~4 ns        ~14 cycles     每核私有
L3 (8~64MB)        ~15 ns       ~50 cycles     多核共享（分片/NUMA）
主存 DRAM          ~80 ns       ~250 cycles    ← 一个 cache miss ≈ 200+ 条指令的时间
```

**为什么缓存行是 64 字节？** 因为要摊薄「行地址译码 + 突发传输」的固定开销。代价是：**你改 1 个字节，硬件搬 64 个字节**——这就是伪共享与「对象头/字段填充」优化的物理来源。

## 1.2 MESI：缓存行的四种状态

每个缓存行有 2 bit 的状态：

| 状态 | 含义 | 是否可写 | 其他核是否有副本 |
|---|---|---|---|
| **M** Modified | 本核独占且已修改，与内存不一致 | 可写 | 无（只有本核有） |
| **E** Exclusive | 本核独占，与内存一致 | 可写 | 无 |
| **S** Shared | 多核共享，与内存一致 | 不可直接写 | 有 |
| **I** Invalid | 失效，不可用 | 不可用 | — |

状态迁移的关键动作：

```text
Core0 读 x（x 不在任何缓存中）
   → Core0 发 BusRd → 内存返回 → 进入 E（独占），因为没别人要

Core1 读 x
   → Core1 发 BusRd → Core0 嗅探到，把 E 降为 S，回传数据
   → Core0=S, Core1=S

Core0 写 x（此时 Core0 为 S）
   → Core0 发 BusRFO(Read For Ownership) / BusUpgr，要求独占所有权
   → Core1 嗅探到，把自己那份置 I（Invalid），并把脏数据回写/转移给 Core0
   → Core0 变 M

Core1 再读 x（Core1 为 I）
   → BusRd → Core0 嗅探到自己是 M（有脏数据），先回写内存/直接转发给 Core1
   → Core0=M 转 S，Core1=S
```

**MESI 只能保证「最终一致」**，它保证不了顺序与原子性，因为：
1. **Store Buffer**（写缓冲）：写完先进缓冲区就继续执行，另一个核可能很久之后才看到。
2. **Invalidate Queue**（失效队列）：收到失效消息先入队、立即回 ACK，真正处理失效是延迟的。
3. 编译器/CPU 的乱序执行。

所以需要**内存屏障**来「清空这些缓冲区、等待确认」。

```text
         ┌──────── Core 0 ────────┐          ┌──────── Core 1 ────────┐
         │  ┌───────────┐         │          │  ┌───────────┐         │
         │  │ Store     │         │          │  │ Invalidate│         │
         │  │ Buffer    │         │          │  │ Queue     │         │
         │  └────┬──────┘         │          │  └────┬──────┘         │
         │       ↓                │          │       ↑                │
         │  ┌───────────┐   L1   │          │  ┌───────────┐   L1   │
         │  │Load Buffer│        │          │  │Load Buffer│        │
         │  └───────────┘         │          │  └───────────┘         │
         └───────────┬────────────┘          └───────────┬────────────┘
                     │          ┌───────────────┐        │
                     └──────────┤   L3 / 总线     ├────────┘
                                └───────────────┘
```

## 1.3 伪共享（False Sharing）

**现象**：两个线程各自修改**同一个缓存行内的不同变量**，硬件仍会按「缓存行」粒度让对方的副本失效，导致两个核交替抢夺该缓存行的所有权（持续 RFO），性能可能比单线程还差 2~5 倍。

```text
long[2] 相邻放在一行 64B 里：

地址偏移   0          8         16        24  ...  64
内容     ┌───────┬───────┬─────────┬─────────┐
         │ x = 0 │ y = 0 │  (填充) │  (填充) │  ← 同一个 cache line
         └───────┴───────┴─────────┴─────────┘
Thread-A 改 x  |  Thread-B 改 y
        ↓                ↓
   两个核不停地把这一行在自己 L1 里置 M，对方置 I
```

**Java 里的解法**：

| 手段 | 说明 | 使用者 |
|---|---|---|
| `@sun.misc.Contended` | JVM 在字段前后各补 64B（需 `-XX:-RestrictContended` 对用户类生效） | `LongAdder.Cell`、`Thread` 的字段、`ConcurrentHashMap.CounterCell` |
| 手工 pad 数组 | `long[8]` 填充，前后各 7 个废字段 | JDK6 时代的 `ConcurrentLinkedQueue` 节点填充、`Exchanger` 的 arena slot |
| 座位分散 | 每个线程用不同下标的槽，靠数组元素间隔 128B | `LongAdder` 的 `Cell[]`、`ThreadLocalRandom` 的 probe |
| 对象填充 | 对象头 12B + 字段 + padding 到 64B 倍数 | 老版本 `PaddedInt` 之类 |

> **延伸（面试高频追问）**：「LongAdder 的 Cell 数组为什么长度取 `NCPU` 的 2 的幂，而不是固定 8？」因为目标是把 N 个线程摊到 N 个独立缓存行上，冲突概率趋近 0；长度上限 `>= NCPU` 后即使再扩也没用（没有更多并发写者）。而 2 的幂是为了用 `(hash & (len-1))` 位运算定位，省掉除法。

## 1.4 内存屏障（Memory Barrier / Fence）

### 1.4.1 四种屏障语义

| 屏障 | 保证 | 典型用途 |
|---|---|---|
| **LoadLoad** | 屏障前的读 先于 屏障后的读 完成 | 防止读-读重排 |
| **StoreStore** | 屏障前的写 先于 屏障后的写 完成（且前面的写已对其他核可见） | 防止写-写重排，如 `final` 字段、volatile 写前 |
| **LoadStore** | 屏障前的读 先于 屏障后的写 | 防止读-写重排 |
| **StoreLoad** | 屏障前的写 先于 屏障后的读（**最强的屏障**，通常要清空 Store Buffer） | volatile 写、CAS、`synchronized` 退出 |

> **StoreLoad 为什么最贵？** 因为它必须把 Store Buffer 全部刷到对其它核可见，并等待 Invalidate Queue 处理完毕。x86 上它的实现就是 `mfence`，或用 `lock` 前缀指令（`lock addl $0,(%rsp)`）——`lock` 本身具有 full barrier 语义。

### 1.4.2 x86（TSO）与 ARM（弱一致）

**x86 是 TSO（Total Store Order）**：只允许「Store-Load」重排（写后读可以越过），Load-Load、Load-Store、Store-Store 都不重排。所以：

| 语义 | x86 实现 | ARM64 实现 |
|---|---|---|
| LoadLoad / LoadStore | 无（硬件保证） | `dmb ishld` / `dmb ish` |
| StoreStore | 无（硬件保证） | `dmb ishst` |
| StoreLoad | `mfence` 或 `lock ...` | `dmb ish` |
| 原子读改写 | `lock cmpxchg` | `ldaxr/stlxr` 循环 + `dmb` |
| volatile 读 | 普通 `mov`（有的版本用 `mov` 即可） | `ldar`（acquire 语义） |
| volatile 写 | `mov` + `lock addl $0,(%rsp)` | `stlr`（release 语义） |

**这解释了为什么「x86 上不加 volatile 好像也能跑对」**——那是 TSO 太强，掩盖了 bug；换到 ARM 服务器（很多国产/云原生机器是 ARM64）立刻翻车。这类问题在「本地 x86 复现不了、线上 ARM 必现」的故障里非常常见。

### 1.4.3 JVM 里的屏障落地

HotSpot 用一个跨平台封装 `OrderAccess`：

```cpp
// hotspot/src/share/vm/runtime/orderAccess.hpp (x86 实现)
inline void OrderAccess::loadload()   { acquire(); }
inline void OrderAccess::storestore() { release(); }
inline void OrderAccess::loadstore()  { acquire(); }
inline void OrderAccess::storeload()  { fence(); }   // → mfence 或 lock addl
```

JIT 编译器在遇到 volatile 字段访问时插入对应的 `membar` 节点（C2 的 `MemoBarNode` / `LoadLoadNode` 等），最终由 `x86_64.ad` / `aarch64.ad` 里的指令模板落地。

## 1.5 CAS 与「锁前缀」的物理含义

### 1.5.1 CAS 指令

`CAS(address, expected, new)` 的语义：**读改写三步不可分割**。

- x86：`lock cmpxchg [addr], reg`
  - `cmpxchg` 天然不是原子的（读-比较-写三步），必须加 `lock` 前缀。
- ARM64：`ldaxr`（Load-Acquire Exclusive）+ `stlxr`（Store-Release Exclusive），失败重试（LL/SC 架构）。

`lock` 前缀做了什么：

1. 使本核缓存行进入 **M 状态**（独占），阻止其他核在同一时刻操作该行 → **缓存锁**（cache locking）。
2. 把所有挂起的 Store Buffer 刷出、处理完 Invalidate Queue → **相当于 StoreLoad 屏障**。
3. 如果目标地址**跨缓存行**或位于不可缓存内存（UC），无法用缓存锁，则退化为**总线锁**（LOCK# 信号拉低，锁总线）——性能差一个数量级。

> **面试点**：「CAS 是不是一定比 synchronized 快？」—— 不一定。CAS 在**低竞争**下无系统调用、无上下文切换，极快；但在**高竞争**下，N 个线程不断重试 `lock cmpxchg`，缓存行在核间来回弹（cache line ping-pong），总线和功耗飙升，此时直接阻塞（重量级锁）反而更省 CPU。这就是 `LongAdder` 与 `AtomicLong` 在大压力下差距巨大的原因，也是 AQS 里「先 CAS 尝试、失败立即入队 park」而不是「死循环自旋」的设计理由。

### 1.5.2 ABA 问题

```text
线程1: 读 A → 准备 CAS(A → C)
线程2: 把 A 改成 B，又改回 A
线程1: CAS 成功（因为值还是 A）→ 但其实共享状态已被修改过两次
```

栈（Treiber 栈）上的 ABA 会导致「丢节点/悬挂指针」。解法：

| 方案 | 原理 | 组件 |
|---|---|---|
| 版本号/时间戳 | 把 (引用, 版本) 打包成一个原子值 | `AtomicStampedReference` |
| 标记位 | 只关心「是否被改过」 | `AtomicMarkableReference` |
| 不复用节点 | 出队即 GC，靠指针唯一性（+ `AtomicMarkableReference`） | `ConcurrentLinkedQueue`（不用 `AtomicReference` 直接 CAS 引用） |
| 延迟回收/epoch | 类似 RCU | 无锁数据结构通用方案 |

`AtomicStampedReference` 的实现是 `AtomicReference<Pair<T>>` + `compareAndSet(current, new Pair(stamp+1, newRef))`，先比引用再比 stamp，两者都相等才写。

## 1.6 上下文切换与系统调用成本

| 动作 | 量级 | 说明 |
|---|---|---|
| 一次 CAS | ~1~20 ns | 无竞争时就是一条指令 |
| `park()` 系统调用 | ~1~3 μs | futex 进内核、挂到等待队列 |
| 线程上下文切换 | ~3~10 μs | 保存/恢复寄存器、刷新 TLB、破坏缓存局部性 |
| 一次 64B cache miss | ~80 ns | 换线程后重新热身 |

**结论**：阻塞的代价约是一次 CAS 的 500~1000 倍。所以所有高性能并发结构的第一原则都是「**先自旋/重试，实在不行才阻塞**」，并且「**唤醒时只唤醒必要的人**」（AQS 的 `unparkSuccessor` 只唤醒一个）。

---

# 第 2 章 JMM 与 happens-before

## 2.1 JMM 是什么

JMM（Java Memory Model，JSR-133）不是「内存布局」，而是**一套规范，规定多线程读写共享变量时，哪些结果是被允许的**。它需要同时约束三方：

- **编译器**：不许把代码重排成违反规则的样子；
- **JVM/JIT**：不许把 `volatile` 优化掉、不许把非 volatile 的读提到屏障之后；
- **硬件**：通过屏障指令约束 CPU 重排。

### 2.1.1 JMM 的「8 种原子操作」（面试经典但需批判性地讲）

`lock / unlock / read / load / use / assign / store / write`，以及规则「read+load 必须连续、store+write 必须连续、不允许丢弃最近的 assign」。

> **重要补充（能加分）**：这 8 个操作是**早期（JSR-133 之前，即旧 JMM）的描述方式**，现行 JMM（JSR-133 之后）的正式定义是 **happens-before + 因果性（causality）+ 不允许「凭空出现」的值（out-of-thin-air）**。面试时先按 8 操作回答（大多数面试官按这个标准答案），然后补一句「现行规范用 happens-before 定义，8 操作是教学模型」，会明显区分层次。

### 2.1.2 三大性质

| 性质 | 定义 | 破坏它的原因 | 保证手段 |
|---|---|---|---|
| **原子性** | 操作不可分割 | 多步操作、64 位非原子（32 位机器上 long/double 的读写） | `synchronized`、CAS、`AtomicXxx`、`LongAdder` |
| **可见性** | 一个线程的写，另一个线程能读到 | 缓存、Store Buffer、编译器把变量放寄存器 | `volatile`、`synchronized`、`final`、`join` |
| **有序性** | 代码按程序顺序生效 | 编译器重排、CPU 乱序、内存系统重排 | `volatile`、`synchronized`、happens-before |

## 2.2 happens-before 规则（8 条 + 2 个补充）

`A happens-before B` 的含义：**A 的结果对 B 可见，且 A 在 B 之前「逻辑上」完成**（不是时间上，是可见性与顺序约束）。

| # | 规则 | 说明 |
|---|---|---|
| 1 | **程序次序** Program Order | 单线程内，前面的语句 hb 后面的语句（注意：这只保证「看起来」的顺序，允许重排但结果必须一致） |
| 2 | **监视器锁** Monitor Lock | 同一锁的 `unlock` hb 后续的 `lock` |
| 3 | **volatile 变量** | 对 volatile 的**写** hb 后续对同一变量的**读** |
| 4 | **线程启动** Thread Start | `t.start()` hb 线程 t 中的任何动作 |
| 5 | **线程终止** Thread Join | 线程 t 中所有动作 hb 其他线程从 `t.join()` 成功返回 |
| 6 | **线程中断** Interruption | 对线程 `interrupt()` hb 被中断线程检测到中断（`isInterrupted()`/`InterruptedException`） |
| 7 | **对象终结** Finalizer | 构造函数结束 hb `finalize()` 开始 |
| 8 | **传递性** Transitivity | A hb B，B hb C ⇒ A hb C |

**补充两条常被忽略但极其实用的**：

- **`CountDownLatch.await()` 返回** hb 每个 `countDown()` 之前的动作（因为内部 `releaseShared` → `acquireShared` 也是 AQS 的 volatile 读写链）。
- **`ConcurrentHashMap` 的 put/get**：`put` 写 `val` 后写 `tab[i]`（volatile），`get` 读 `tab[i]`（volatile）后读 `val` —— 靠一次 volatile 写读建立 hb。这是「用一次 volatile 建立整条链」的经典手法。

### 2.2.1 一个必须能手推的例子：为什么需要 happens-before 的「传递性」

```java
// 线程 A
x = 1;              // 普通写
volatileFlag = true; // volatile 写

// 线程 B
if (volatileFlag) {  // volatile 读
    assert x == 1;   // 一定成立吗？
}
```

推演：

1. `x = 1` hb `volatileFlag = true`（程序次序）
2. `volatileFlag = true` hb `if (volatileFlag)`（volatile 规则）
3. `if (volatileFlag)` hb `assert x == 1`（程序次序）
4. ⇒ 传递性：`x = 1` hb `assert x == 1` ⇒ 成立。

**物理上怎么保证的？** 编译器在 `volatileFlag = true` **之前**插入 `StoreStore` 屏障（把 `x=1` 刷出去），在 volatile 写**之后**插入 `StoreLoad`；读侧在 volatile 读**之后**插入 `LoadLoad/LoadStore`。这就是「一个 volatile 写 + 一个 volatile 读 = 一把轻量锁」的原理。

### 2.2.2 as-if-serial 与「重排的边界」

- **as-if-serial**：单线程语义不变，编译器和 CPU 可以任意重排。
- **数据依赖** 不会被重排（`a=1; b=a;` 中 `b=a` 不能提到 `a=1` 前）。
- **控制依赖** 可以被重排（推测执行）：`if (flag) x = 1;` 可能被提前执行 `x=1`——这就是 DCL 必须给 `instance` 加 `volatile` 的根本原因。

## 2.3 经典反例：DCL 双重检查锁

```java
public class Singleton {
    private static Singleton instance;          // ← 少 volatile 就出 bug
    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查（无锁，快）
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查（防重复创建）
                    instance = new Singleton(); // ← 这一行不是原子的！
                }
            }
        }
        return instance;
    }
}
```

`new Singleton()` 在字节码里是三步：

```text
1. new            → 在堆上分配内存（此时字段都是默认值 0/null）
2. invokespecial  → 调用 <init> 构造函数
3. putstatic      → 把引用赋给 instance
```

**2 和 3 可能被重排**（对单线程无影响，因为没人观察中间态）。于是线程 B 可能在 `instance` 已指向「未初始化完成的对象」时读到它，看到 `field == null`，炸。

加上 `volatile` 后：

- 写：`putstatic` 前插 `StoreStore`（保证构造函数内的写先完成），写后插 `StoreLoad`；
- 读：`getstatic` 后插 `LoadLoad`（保证后续读对象字段不会提到引用读取之前）。

> **延伸（JDK5 之后语义变化）**：JDK5 修复了「旧 JMM 里 final 字段可能被看到默认值」的问题（JSR-133 的 final 字段语义）。所以**「构造函数里写 final 字段 + 正确发布」是安全的**，这也是「不可变对象天然线程安全」的规范依据。但如果 final 字段指向可变对象，其内容仍需自己保证。

## 2.4 安全发布（Safe Publication）四种姿势

| 方式 | 原理 |
|---|---|
| 静态初始化（`static final`） | 类初始化由 JVM 保证 hb（`<clinit>` 的锁 + 内存屏障） |
| `final` 字段 | JSR-133 final 语义：构造函数内写 final 后自动 `StoreStore`，读 side 自动 `LoadLoad` |
| `volatile` 字段 / `AtomicReference` | volatile 写读建立 hb |
| 并发容器（`ConcurrentHashMap`、`BlockingQueue`） | 容器内部的 volatile/锁建立 hb |
| 正确加锁 | 监视器规则 |

**错误发布**：把对象放进非线程安全容器（`HashMap`、`ArrayList`）再交给其他线程，或者「在构造函数里把自己 `this` 注册到监听器/线程」，导致其他线程看到半初始化对象（`this` 逃逸）。

---

# 第 3 章 volatile 端到端：从字节码到 CPU 指令

> 这一章逐层拆解一个 `volatile` 字段的读写，究竟在哪一层被「特殊处理」了。**这是所有并发问题「为什么能/不能看见」的总答案**。

## 3.1 第一层：Java 语言与字节码

```java
public class VolatileDemo {
    private volatile int flag;
    public void writer() { flag = 1; }
    public int reader()  { return flag; }
}
```

`javap -v` 的关键输出：

```text
private volatile int flag;
  descriptor: I
  flags: ACC_PRIVATE, ACC_VOLATILE     ← 关键：字段级标记

public void writer();
   0: aload_0
   1: iconst_1
   2: putfield   #2  // Field flag:I
   ...
```

注意几点（面试很爱考）：

1. **`volatile` 是字段修饰符，不是方法/代码块修饰符**，所以它不会产生 `monitorenter`，只会在**字段访问的字节码**（`getfield/putfield/getstatic/putstatic`）上保留 `ACC_VOLATILE` 标记。
2. 字节码里 **看不到任何屏障指令**——屏障是 JVM 在解释执行/编译执行时**按字段的 ACC_VOLATILE 标记现场插入的**。这正说明「volatile 的语义不是靠字节码完成的」。
3. `ACC_VOLATILE` 与 `ACC_TRANSIENT` 复用同一个 bit 位（0x0040），因为一个是字段修饰、一个是序列化语义，JVM 内部区分使用。

## 3.2 第二层：JVM 访问字段的实现路径

`putfield` 在 HotSpot 里的执行路径（三条，取决于是否被 JIT 编译）：

```text
解释器：InterpreterRuntime::putfield
         ↓ (fieldDescriptor 的 is_volatile 为 true)
         → 在赋值前后调用 OrderAccess::release_store / storeload
模板解释器：TemplateTable::putfield → 生成带 membar 的机器码
C1（Client 编译器）：LIR 生成 store + 1_membar
C2（Server 编译器）：IR 里生成 StoreI/MemBar/StoreLoad 节点，
                     再由 x86_64.ad / aarch64.ad 选择具体指令
```

**关键的 JVM 抽象 `OrderAccess`**（跨平台屏障原语）：

```cpp
// 概念化的用法（HotSpot 源码风格）
void put_volatile_int(int* p, int v) {
    OrderAccess::storestore();   // 保证之前的所有写先完成（防止写-写重排到 volatile 写之后）
    *p = v;                      // 真正的写（x86 上是普通 mov）
    OrderAccess::storeload();    // 保证写对其他核可见（x86: mfence 或 lock addl）
}

int get_volatile_int(int* p) {
    int v = *p;                  // x86 上是普通 mov
    OrderAccess::acquire();      // 保证后续读写不会重排到 volatile 读之前
    return v;
}
```

**记住这张表（面试可直接说）**：

| 操作 | JVM 层要求 | x86_64 实际指令 | ARM64 实际指令 |
|---|---|---|---|
| volatile 写 | StoreStore + 写 + StoreLoad | `mov` + `lock addl $0, (%rsp)` | `stlr`（自带释放语义，无需额外 dmb） |
| volatile 读 | 读 + LoadLoad/LoadStore | `mov`（TSO 已保证） | `ldar` |
| CAS / `Atomic*` | 全屏障 | `lock cmpxchg` | `ldaxr`/`stlxr` + `dmb ish` |

> **为什么 x86 的 volatile 写用 `lock addl $0,(%rsp)` 而不是 `mfence`？**
> 两者都能实现 StoreLoad，但 `lock addl $0,(%rsp)`（对栈顶加 0，无实际数据变更）在某些微架构上比 `mfence` 更快，且不会与后续的 `mfence` 产生额外开销。HotSpot 的 `OrderAccess::fence()` 在 x86 上就是这一句（`lock(); addl(0, rsp)`）。这是 JIT 生成的机器码在反汇编里能直接看到的经典片段——**在 `hsdis` 反汇编里看到 `lock addl $0x0,(%rsp)` 就说明这里有一个 volatile 写或锁释放**。

## 3.3 第三层：编译器的重排边界

JIT（尤其 C2）会做大量优化，`volatile` 相当于给优化器画了红线：

| 优化 | 对 volatile 是否允许 | 原因 |
|---|---|---|
| 把变量提升到寄存器（循环内只读一次） | ❌ 禁止 | 每次都必须重新从内存读（否则其他线程的写永远看不见） |
| 死代码消除（写了没人读就删掉） | ❌ 禁止 | volatile 写有副作用（可见性/屏障） |
| 读写合并、常量传播 | ❌ 禁止 | 同上 |
| 把 volatile 写挪到循环外（循环内多次写同值） | ❌ 禁止 | 每次写都是一次发布 |
| 相邻 volatile 写合并 | ❌ 禁止 | — |

**这就是「volatile 的读一定读到最新值」的机制**：不是缓存被特殊处理，而是**编译器不允许把它缓存进寄存器**，每次都发 load，而 load 会走 MESI 取到最新的 M/所有者那份。

## 3.4 一个容易错的点：volatile 不保证原子性

```java
private volatile int count;
count++;   // ← 三个步骤：getfield → iadd → putfield
```

两次 `count++` 交错执行会丢更新，因为 `iadd` 不是原子的。修正方案（按性能从高到低）：

| 方案 | 实现 | 适用 |
|---|---|---|
| `AtomicInteger.incrementAndGet()` | `lock xadd` 或 CAS 循环 | 中低竞争 |
| `LongAdder.increment()` | 分片 Cell + base | 高竞争、只求最终求和 |
| `synchronized` / `ReentrantLock` | 管程 | 需要复合原子操作（读-改-写多个变量） |
| `VarHandle.getAndAdd`（JDK9+） | 与 AtomicInteger 同源 | 想摆脱 Unsafe 依赖 |

## 3.5 volatile 的正确用法清单

| 场景 | 示例 | 是否合适 |
|---|---|---|
| 状态标志位（一写多读） | `volatile boolean running` | ✅ 经典 |
| 一次性安全发布 | DCL 的 `instance` | ✅ |
| 独立观察（写不依赖当前值） | 心跳时间戳 `volatile long lastHeartbeat` | ✅ |
| 计数器 | `count++` | ❌ 用 `AtomicInteger` / `LongAdder` |
| 多个相关变量的一致性 | `volatile int x, y` 需要一起变 | ❌ 需加锁（volatile 只保证单个变量的 hb） |
| 替代锁保护复合不变式 | 转账（扣 A 加 B） | ❌ 必须锁 |

> **延伸：`volatile` 的「两次读」问题（DCL 之外的经典坑）**
> ```java
> if (flag) { doSomething(); }   // 只读一次 flag
> ```
> 这里没问题。但如果写成 `while (map.get(k) == null) { ... }`，`map` 若不是并发容器，`get` 内部的读可能被优化，或读到陈旧结构（如 `HashMap` 扩容中的状态），导致死循环——这是「非线程安全容器在多线程下不仅丢数据，还会结构损坏」的典型。

---

# 第 4 章 synchronized 全链路：对象头、锁升级、ObjectMonitor

> `synchronized` 不属于 JUC 包，但它是 JUC 的对立面与参照系，也是面试第一高频题。**不懂它，就说不清为什么 ReentrantLock 能中断、能超时、能公平。**

## 4.1 对象内存布局

```text
┌──────────────────────────────────────────────┐
│  Mark Word（8 字节 / 64 位）                   │  ← 锁状态、hashCode、GC 年龄、偏向线程 ID
├──────────────────────────────────────────────┤
│  Klass Pointer（4 字节 压缩 / 8 字节 不压缩） │  ← 指向方法区的 InstanceKlass
├──────────────────────────────────────────────┤
│  实例数据（instance data）                     │  ← 按字段类型紧凑排列（long/double 优先）
├──────────────────────────────────────────────┤
│  对齐填充（padding）                           │  ← 补到 8 字节倍数
└──────────────────────────────────────────────┘
```

- 空对象 16 字节（8 Mark + 4 Klass + 4 padding，开启 `-XX:+UseCompressedOops`）。
- 数组对象额外 4 字节 length。
- 这就是「对象头占 12/16 字节，所以小对象也要 16 字节起步」的来源。

## 4.2 Mark Word 的 5 种形态（64 位、开启压缩指针）

```text
① 无锁（normal）
 63        39 38          7 6   5   4     3  2 1 0
┌────────────┬───────────────┬─────┬─────┬─────┐
│  unused:25 │ hashCode:31   │unused│ age │0│01 │
└────────────┴───────────────┴─────┬─────┴──┴───┘
                                   biased_lock=0, lock=01

② 偏向锁（biased）
┌───────────────────────────┬────┬────┬─────┬─────┐
│  ThreadID:54  │ Epoch:2   │unused│age │ 1   │ 01  │
└───────────────────────────┴────┴────┴─────┴─────┘
                                       锁标志 101

③ 轻量级锁（thin lock）
┌──────────────────────────────────────┬─────┬─────┐
│  指向栈上 Lock Record 的指针 : 62      │     │ 00  │
└──────────────────────────────────────┴─────┴─────┘

④ 重量级锁（inflated）
┌──────────────────────────────────────┬─────┬─────┐
│  指向 ObjectMonitor 的指针 : 62        │     │ 10  │
└──────────────────────────────────────┴─────┴─────┘

⑤ GC 标记
┌──────────────────────────────────────┬─────┬─────┐
│  (空)                                 │     │ 11  │
└──────────────────────────────────────┴─────┴─────┘
```

**注意冲突**：`hashCode` 与 `偏向线程 ID` 共用同一块位域——所以**一旦调用过 `Object.hashCode()`（或 `System.identityHashCode`），偏向锁就再也不能建立**（需要把 ThreadID 位覆盖掉）。同理，对象进入重量级锁后，`hashCode` 会被挪到 `ObjectMonitor` 里保存（`_header` 字段），解锁时再恢复回 Mark Word。这个细节是高级面试题。

## 4.3 加锁的字节码层面

```java
public void syncBlock() {
    synchronized (this) { count++; }     // 显式同步块
}
public synchronized void syncMethod() { count++; }  // 同步方法
```

```text
// 同步块
2: monitorenter        ← 进入监视器
... 业务代码 ...
19: monitorexit        ← 正常退出
...
// 异常表：athrow 之前还有一个 monitorexit，保证异常也解锁
Exception table:
   from    to  target type
      5    16    19   any     ← 编译器自动补的 finally 式解锁

// 同步方法：没有 monitorenter/exit，
// 而是在方法访问标志里加 ACC_SYNCHRONIZED（0x0020）
public synchronized void syncMethod();
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```

> **考点**：`synchronized` 修饰方法与修饰代码块在字节码上不同（`ACC_SYNCHRONIZED` vs `monitorenter/exit`），但**运行时走同一个 `ObjectMonitor`**。方法级由 JVM 在方法调用/返回时隐式 enter/exit。

## 4.4 锁升级全流程（重点，务必能画出来）

```text
                   ┌──────────────────────────────────────────────┐
                   │  new Object()  →  无锁（lock=01, biased=0）  │
                   └───────────────────────┬──────────────────────┘
                                           │ 第一次 synchronized，且
                                           │ -XX:+UseBiasedLocking（JDK≤14 默认开）
                                           │ 且对象所属类未禁用偏向、且启动延迟(-XX:BiasedLockingStartupDelay=4s)已过
                                           ▼
                   ┌──────────────────────────────────────────────┐
            ┌─────▶│  偏向锁：Mark Word 写入 ThreadID + Epoch      │
            │      │  后续同一线程进入：只需比较 ThreadID，零 CAS  │
            │      └───────────────────┬──────────────────────────┘
            │                          │ 另一个线程来抢
            │                          ▼
            │      ┌──────────────────────────────────────────────┐
            │      │ 偏向撤销（revoke）：需要在 SafePoint 停线程     │
            │      │ ① 若原线程已死/不在同步块 → 直接改偏向新线程     │
            │      │ ② 若原线程仍活跃 → 升级为轻量级锁                │
            │      └───────────────────┬──────────────────────────┘
            │                          ▼
            │      ┌─────────────────────────────────────────────────┐
    重入时 │      │ 轻量级锁：CAS 把 Mark Word 换成指向当前栈帧          │
   displaced│     │ Lock Record 的指针；成功 = 加锁成功（lock=00）        │
   header= │      │ Lock Record 内保存原 Mark Word（displaced header）   │
   NULL    │      └───────────────────┬──────────────────────────────┘
            │                          │ CAS 失败（有竞争）→ 自适应自旋重试
            │                          │ 自旋仍失败 → 膨胀（inflate）
            │                          ▼
            │      ┌──────────────────────────────────────────────┐
            └──────│ 重量级锁：向 VM 申请 ObjectMonitor（lock=10） │
                   │ 未抢到 → 进入 cxq/EntryList 队列 → park       │
                   │ 释放 → 唤醒 EntryList/cxq 中的后继            │
                   └──────────────────────────────────────────────┘
```

### 4.4.1 各阶段的操作细节

**① 偏向锁（Biased Locking）**

- 目的：**消除「同一个线程反复进入同一把锁」的所有同步开销**（很多同步块实际无竞争，甚至是单线程程序的伪同步）。
- 加锁（`fast_enter`）：
  ```text
  读 Mark Word → 判断 biased_lock==1 && lock==01？
    → 是：比较 ThreadID 是否等于当前线程（指针比较，几乎零成本）
        → 相等：直接执行（连 CAS 都没有）
        → 不相等：进入 revoke 流程
    → 否（无锁态）：CAS 把 ThreadID 写入 Mark Word
        → 成功：获得偏向锁
        → 失败：说明有并发，撤销偏向 → 轻量级
  ```
- 释放：**偏向锁没有释放动作**（因为「一直偏向」，直到有别人来抢）。这也是它必须能被撤销的原因。
- **撤销（revoke）为什么必须 SafePoint？** 因为要判断「原偏向线程是否还在同步块内」，需要遍历它的栈帧（看有没有对应的 Lock Record / BasicObjectLock）；修改 Mark Word 时不能让对方在跑。所以偏向撤销是 **STW 级别的小停顿**，这正是「偏向锁在大量竞争场景下反而拖慢」的原因（JDK15 用 JEP 374 默认禁用，且已废弃）。
- **批量重偏向（bulk rebias）/ 批量撤销（bulk revoke）**：
  - 每个类维护一个 `epoch`（2 bit），对象 Mark Word 里也存一份 epoch。
  - 当某类的撤销次数达到 `BiasedLockingBulkRebiasThreshold = 20`（默认 20），JVM 认为「这类对象可能被多线程轮流使用」，于是把**类的 epoch +1**，并把该类所有对象的偏向标记为「可重偏向」——之后其他线程可以直接 CAS 改偏向，不必撤销。
  - 达到 `BiasedLockingBulkRevokeThreshold = 40`（默认 40），则**整个类禁用偏向**（`_prototype_header` 改成不可偏向），所有实例直接走轻量级。
  - **这就是为什么「先 A 线程用、再 B 线程用」的对象在 20 次之后行为会变**——面试里问到「偏向锁在什么情况下会被批量撤销」就答这个。

**② 轻量级锁（Thin Lock）**

- 目的：**在有少量竞争、且临界区很短时，用 CAS + 自旋代替系统调用**。
- 加锁（`slow_enter` 的无锁态分支）：
  ```text
  在当前栈帧里分配一个 BasicObjectLock（Lock Record）
  Lock Record._displaced_header = 当前 Mark Word（备份）
  CAS(obj->mark_word, 原值, 指向 Lock Record 的指针)
    → 成功：Mark Word 变为 lock=00，加锁完成
    → 失败：说明 Mark Word 已被别人改（有竞争）→ 自适应自旋
              自旋后仍失败 → inflate 膨胀为重量级
  ```
- **重入**：同一线程再次进入，会再分配一个 Lock Record，但 `_displaced_header = NULL` 作为重入标记；解锁时看到 NULL 就只 pop（不 CAS 回写）。
- 解锁：
  ```text
  monitorexit → 若 Lock Record 的 displaced_header == NULL → 重入，pop 即可
               否则 CAS(Mark Word, 当前指针, displaced_header)
                 → 成功：释放
                 → 失败：说明已膨胀为重量级，转 ObjectMonitor::exit
  ```
- **轻量级锁的 Mark Word 指针指向栈**，所以「锁在对象头上、栈上存旧值」是它和偏向锁最关键的区别。也因为指向栈帧，**锁记录的生命周期由栈帧决定**，方法返回就必须解锁（`synchronized` 是代码块，天然满足）。

**③ 自适应自旋（Adaptive Spinning）**

```text
自旋次数不是固定的 10 次（-XX:PreBlockSpin 只用于老式固定自旋），
而是由「上一次在该锁上的自旋成功率」动态决定：
  - 上次自旋成功等到了锁 → 认为这次也大概率成功 → 多自旋几次（甚至不 park）
  - 上次自旋白费 → 这次少转或不转，直接 park
JVM 还会在多核（>1 CPU）时才启用自旋，单核自旋毫无意义。
```
自旋的物理代价：**空转 CPU 周期**，但避免了「用户态→内核态→上下文切换」的 μs 级开销。临界区小于一次上下文切换（~10μs）时，自旋是划算的。

**④ 重量级锁 / ObjectMonitor**

膨胀后，对象头指向堆中的 `ObjectMonitor`（HotSpot 的 C++ 对象）：

```cpp
class ObjectMonitor {
  volatile markOop   _header;       // 保存原始 Mark Word（用于解锁恢复 hashCode 等）
  void*     volatile _object;       // 指向对象本身（back pointer）
  volatile intptr_t  _owner;        // 持有锁的线程（Thread*）
  volatile int       _recursions;   // 重入次数
  volatile int       _count;        // 近似 _waiters + _recursions 的优化计数
  volatile int       _waiters;      // wait 线程数
  ObjectWaiter* volatile _WaitSet;  // 调用 wait() 的线程（双向链表，队首 _WaitSet，队尾 _wait_set_tail）
  ObjectWaiter* volatile _cxq;      // 竞争锁的线程（单向 LIFO 栈，新来的插队头）
  ObjectWaiter* volatile _EntryList;// 被唤醒后等待重新竞争的线程（双向链表）
  volatile intptr_t  _succ;         // 指定继承人（heir presumptive），用于减少无效唤醒
  volatile int       _contentions;  // 竞争标记
  volatile int       _SpinDuration; // JDK19+ 为虚拟线程引入的自旋时长
};
```

**进入（`ObjectMonitor::enter`）**：

```text
1. 快速路径：Atomic::cmpxchg(_owner, NULL, Self)
     成功 → 直接返回（无锁竞争时的全部开销就是这一次 CAS）
2. 若 _owner == Self → _recursions++（可重入）
3. 否则进入 enterI：
   a. 自旋尝试（TrySpin / Speculative）：在多核下先转几圈，
      并用「启发式」把 _succ 设为自己（表示"我马上要拿到锁了"，
      让退出者直接把这个线程 unpark，而不是唤醒队列里其他线程）
   b. 自旋失败 → 把当前线程包装成 ObjectWaiter，
      用 CAS 头插到 _cxq（LIFO，后进先出，提高缓存局部性/减少饥饿）
   c. 再次尝试抢锁（可能刚好被释放）
   d. 仍然失败 → Atomic::park()（底层是 pthread_mutex_lock + cond_wait），
      状态变 BLOCKED
4. 被 unpark 后从 park 返回，重新走「抢锁 → 失败则把自己挂到 _EntryList」
```

**退出（`ObjectMonitor::exit`）**：

```text
1. _recursions > 0 → 减一返回（重入退出）
2. _owner = NULL
3. 唤醒策略（关键，考的是「为什么不是唤醒整个队列」）：
   ① 若 _succ 存在 → unpark(_succ)      ← 指定继承人，最省
   ② 否则若 _EntryList 非空 → 从 _EntryList 取一个 unpark
   ③ 否则若 _cxq 非空 → 把 _cxq 整条链搬进 _EntryList（并在搬运时反转顺序，
      这样 LIFO 的 _cxq 变成 FIFO 的 _EntryList，兼顾公平性），再唤醒一个
```

> **为什么需要 `_cxq` 和 `_EntryList` 两个队列？**
> `_cxq` 是「刚来抢锁失败」的 LIFO 栈（头插，无锁 CAS，快，且刚失败的线程数据还在缓存里，优先让它再试可能成功）；
> `_EntryList` 是「被 notify 唤醒、等待重新抢锁」的队列（也用于把 `_cxq` 搬过来后按 FIFO 处理，防止饥饿）。
> 这个设计是 HotSpot 在「吞吐」与「公平」之间的折中：**新线程插队（LIFO），老线程按序（FIFO）**。

**`wait/notify` 的流程（`_WaitSet`）**：

```text
wait():  ① 要求当前线程持有 monitor（否则 IllegalMonitorStateException）
         ② 把线程包装成 ObjectWaiter，加入 _WaitSet（双向链表）
         ③ 完全释放锁（_recursions 归零、_owner=NULL）
         ④ park
notify(): ① 从 _WaitSet 取一个（notifyAll 取全部）
          ② 转移到 _EntryList 或 _cxq（取决于策略 QMode）
          ③ 被转移的线程被 unpark，但要重新竞争锁（不保证立即执行）
```

**WAITING/BLOCKED 的区别**：`synchronized` 抢锁失败 → `BLOCKED`；`wait()` → `WAITING`。`jstack` 里能直接区分，这是线上定位「谁卡住了」的关键线索。

## 4.5 JVM 的两种锁优化：锁消除与锁粗化

**锁消除（Lock Elision）**：基于**逃逸分析（Escape Analysis）**。

```java
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();  // sb 不逃逸出方法
    sb.append(a).append(b);                // 内部 synchronized 全部被消除
    return sb.toString();
}
```

- 条件：对象**不逃逸**（不被返回、不被存入全局结构、不被其他线程访问），且 `-XX:+DoEscapeAnalysis`（默认开）、`-XX:+EliminateLocks`（默认开）。
- 实现：C2 的 `EliminateLockNodes` 优化阶段，把对应 `Lock/Unlock` 节点直接删掉。
- **面试陷阱**：「`StringBuffer` 是线程安全的，所以拼接肯定比 `StringBuilder` 慢」—— 在方法内局部变量场景下，JIT 会把 `StringBuffer` 的锁完全消掉，性能几乎一致；只有**逃逸**时才有真实锁开销。

**锁粗化（Lock Coarsening）**：把相邻的多次 `synchronized` 合并成一次。

```java
for (int i = 0; i < 100; i++) {
    synchronized (lock) { sum += i; }   // 100 次 enter/exit
}
// JIT 可能粗化成：
synchronized (lock) { for (int i = 0; i < 100; i++) sum += i; }
```

- 依据：在**同一个对象**上的连续加解锁，中间没有其他线程能观察到（因为在优化视角下临界区可合并），合并后减少重复的 CAS/唤醒成本。

## 4.6 synchronized 与 ReentrantLock 对比（面试必答表）

| 维度 | `synchronized` | `ReentrantLock` |
|---|---|---|
| 实现层 | JVM 管程 + 对象头 + ObjectMonitor（C++） | JDK 层 AQS（Java 代码 + CAS + park） |
| 加锁方式 | 自动（字节码 / 方法标志） | 手动 `lock()` / `unlock()`，必须 try-finally |
| 可重入 | 支持（`_recursions` 计数） | 支持（`state` 计数） |
| 公平性 | 只有非公平（`_cxq` 头插，允许插队） | 可选公平/非公平（`FairSync` 用 `hasQueuedPredecessors`） |
| 可中断 | ❌ 阻塞后不可中断 | ✅ `lockInterruptibly()` |
| 超时 | ❌ 不支持 | ✅ `tryLock(timeout)` |
| 尝试非阻塞 | ❌ | ✅ `tryLock()` |
| 条件变量 | 1 个（`wait/notify` 绑定在 monitor 上） | **多个** `Condition`（`newCondition()`），可精细唤醒 |
| 锁状态可查 | ❌ 无 API | ✅ `isLocked()`、`getQueueLength()`、`getHoldCount()` |
| 锁升级 | 有（偏向→轻量→重量） | 无（一直 CAS + park，但有自旋优化） |
| 性能 | 低竞争：偏向/轻量级下极快（接近无锁）；高竞争：内核态阻塞 | 低竞争：一次 CAS；高竞争：AQS 队列 + park，且可指定继承者减少无效唤醒 |
| 内存语义 | `monitorexit` 隐含 release（StoreLoad），`monitorenter` 隐含 acquire | 同样（AQS 的 volatile + CAS 提供） |
| 与虚拟线程 | **会导致载体线程 pinned（JDK19/21 早期）**，JDK 24 起 JEP 491 解决 | `park` 是虚拟线程友好的挂起 |

> **答题模板（一句话总结）**：「单方法、简单临界区用 `synchronized`（JVM 优化到位、不会忘解锁）；需要超时、可中断、公平、多条件变量、或需要在锁上做状态查询时用 `ReentrantLock`。」

> **延伸：为什么 `synchronized` 高竞争时性能不如 `ReentrantLock`？**
> 因为 ObjectMonitor 的唤醒策略虽然也有 `_succ` 继承，但它的队列管理与 `_EntryList/_cxq` 搬运、以及在超多线程下 `notifyAll` 的「惊群」效应（所有线程醒来抢一把锁，只有一个成功，其他重新 park）会显著浪费 CPU。而 AQS 的 `unparkSuccessor` 是**精确唤醒一个**，并在 `hasQueuedPredecessors` 公平模式下严格排队，队列操作也是纯 Java CAS，无内核对象开销。不过要注意：**低竞争时 `synchronized` 更快**（偏向锁/轻量级锁几乎没有额外对象分配，而 `ReentrantLock` 每次都要 `new` 一个 Node）。

---

# 第 5 章 AQS 深度拆解

> AQS = **`volatile int state` + 一条 CLH 变体的双向队列 + `LockSupport.park/unpark`**。
> 它的设计精髓不是「排队」，而是「**把同步器的策略（要不要给我锁）与阻塞机制（排队与唤醒）解耦**」——策略交给子类（`tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared/isHeldExclusively`），机制留给 AQS。

## 5.1 核心结构

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
    private volatile int state;              // 同步状态（含义由子类定义）

    static final class Node {
        static final Node SHARED = new Node();   // 共享模式标记
        static final Node EXCLUSIVE = null;      // 独占模式标记
        volatile int waitStatus;                 // 状态
        volatile Node prev;                      // 前驱（一定可靠）
        volatile Node next;                      // 后继（可能为 null / 不一致）
        volatile Thread thread;                  // 排队的线程
        Node nextWaiter;                         // 共享模式下标记 SHARED / Condition 队列的下一个
    }
    private transient volatile Node head;    // 懒初始化，永远指向一个「哨兵」（thread=null）
    private transient volatile Node tail;
}
```

### 5.1.1 Node 的 5 种 waitStatus（**必须能背**）

| 值 | 名称 | 含义 | 谁来设置 |
|---|---|---|---|
| `1` | `CANCELLED` | 该节点已取消（超时/中断/异常），需要被清除 | 线程自己（`cancelAcquire`） |
| `-1` | `SIGNAL` | **该节点的后继正在（或即将）park，所以当前节点释放时必须唤醒后继** | 后继线程（`shouldParkAfterFailedAcquire`） |
| `-2` | `CONDITION` | 该节点在 Condition 队列中等待（不在同步队列） | `addConditionWaiter` |
| `-3` | `PROPAGATE` | **共享模式下释放需要向后传播**（一次释放要唤醒一串） | `doReleaseShared` |
| `0` | — | 初始态：既是「新建节点」的值，也是「已释放但仍需处理」的中间态 | `new Node()` / CAS 改回 0 |

**这是 AQS 里最容易答错的地方**，务必理解：
- `SIGNAL` 挂在**前驱**身上，表示「我有后继在睡，我要负责叫醒它」。这是 AQS 的「**委托唤醒**」机制：**没有专门的唤醒线程，谁释放谁负责唤醒**。
- 因此 `head.waitStatus` 通常是 `SIGNAL` 或 `0`/`PROPAGATE`；`head` 节点本身不排队。

### 5.1.2 为什么叫「CLH 变体」

| | 原始 CLH | AQS 的变体 |
|---|---|---|
| 队列方向 | 单向（只 prev） | **双向**（prev 可靠用于从尾向前扫，next 用于向后唤醒） |
| 等待方式 | **自旋**在自己前驱的变量上 | **阻塞**（park），因为自旋在多线程下浪费 CPU |
| 节点归属 | 每个线程一个 node，只关心前驱状态 | node 保存 waitStatus，由后继修改前驱的 status |

> **为什么 `next` 可能不可靠而 `prev` 一定可靠？** 入队时是先 `node.prev = pred`（volatile 写），再 `CAS(tail, node)`；`pred.next = node` 是**之后的普通写**。所以从 `tail` 往回走一定能覆盖所有节点，而从 `head` 往后走可能漏掉「刚入队但 next 还没写」的节点。`unparkSuccessor` 因此**从 tail 反向扫描**（`for (Node t = tail; t != null && t != node; t = t.prev)`）。

## 5.2 入队：`addWaiter / enq`

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {                       // 快速路径：队列已初始化
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {  // 一次 CAS
            pred.next = node;
            return node;
        }
    }
    enq(node);                                // 慢路径：懒初始化 + 循环重试
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {                      // 第一次使用：创建哨兵 head
            if (compareAndSetHead(new Node()))  // 注意：head 是 dummy，thread = null
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;                     // 注意：返回的是旧的 tail（pred），这里其实没用返回值
            }
        }
    }
}
```

**三个面试要点**：

1. **head 是哨兵（dummy）**，`thread == null`，不入队、不抢锁。`setHead(node)` 时会把 node 的 `thread=null`、`prev=null`，把 `node` 转成新哨兵，同时 `head.next = null`（帮助 GC 旧节点）。
2. **入队不是原子的**：`prev` 写 + CAS tail + `next` 写，三步之间不一致是**允许**的（后面靠从 tail 反向扫兜底）。
3. **`enq` 返回旧 tail，`addWaiter` 忽略它** —— 这是历史残留代码，别被源码迷惑。

## 5.3 独占模式：`acquire` 全流程

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&                            // ① 先试一次（子类实现，必须无阻塞）
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // ② 失败则入队 + 排队自旋/阻塞
        selfInterrupt();                               // ③ 若在队列里被中断过，补一次中断
}
```

### 5.3.1 时序图

```text
Thread-A                AQS                    Thread-B(已排队)
   │                     │
   │ acquire(1)          │
   ├────────────────────▶│
   │   tryAcquire(1)     │  ① 子类实现：CAS state 0→1
   │   ├─ 成功 → return  │
   │   └─ 失败 ↓          │
   │   addWaiter(EXCL)   │  ② 入队（tail 插入）
   │   acquireQueued()   │
   │     ┌───────────────┴─────────────────────────────────┐
   │     │ for(;;) {                                        │
   │     │   p = node.predecessor();                        │
   │     │   if (p == head && tryAcquire(arg)) {            │  ← ③ 只有队首（head.next）才有资格抢
   │     │       setHead(node); p.next = null; return false;│
   │     │   }                                              │
   │     │   if (shouldParkAfterFailedAcquire(p, node)      │  ← ④ 把前驱设为 SIGNAL，再试一次
   │     │       && parkAndCheckInterrupt())                 │  ← ⑤ 真正 park
   │     │       interrupted = true;                        │
   │     │ }                                                │
   │     └──────────────────────────────────────────────────┘
   │                     │
   │   （被唤醒后回到 for 循环，重新 p==head 判断）
   ▼                     ▼
```

### 5.3.2 `acquireQueued` 逐行解读

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {   // 【关键 1】
                setHead(node);                    // 把自己变成哨兵
                p.next = null;                    // 断开旧 head，帮助 GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&  // 【关键 2】
                parkAndCheckInterrupt())                  // 【关键 3】
                interrupted = true;                       // 只记录，不立即抛出
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

**【关键 1】为什么只有 `p == head` 才 `tryAcquire`？**
保证**近似 FIFO**：只有队首（第一个真正排队的节点）有资格抢锁。但注意，**非公平模式下仍然允许新来的线程插队**（`acquire` 一进来就 `tryAcquire`），所以「非公平」体现在「新线程 vs 队首线程同时抢，新线程有概率赢」。公平模式则用 `hasQueuedPredecessors()` 把新线程挡住。

**【关键 2】`shouldParkAfterFailedAcquire`：三次机会才睡**

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) return true;              // 前驱已承诺唤醒我 → 可以放心睡
    if (ws > 0) {                                    // 前驱已取消 → 往前跳过所有取消节点
        do { node.prev = pred = pred.prev; } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);  // 0 或 PROPAGATE → 设为 SIGNAL
    }
    return false;                                    // 返回 false → 不睡，再试一次抢锁
}
```

设计意图：**「先设 SIGNAL，再回到主循环抢一次锁；抢不到才睡」**。这样大多数短临界区场景下，线程根本不需要 park（省一次系统调用）。

**【关键 3】`parkAndCheckInterrupt` 与中断处理**

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);                 // 阻塞；被 interrupt 会立即返回（不抛异常）
    return Thread.interrupted();            // 【重要】清除中断标志，返回「是否被中断过」
}
```

**为什么是清除标志而不是保留？** 因为如果不清除，park 会立刻返回，`for(;;)` 变成死循环空转（park 对已置位中断的线程立即返回）。所以 AQS 的策略是：**清除标志 → 记录到 `interrupted` → 继续排队 → 最终拿到锁后，`acquire` 里补一次 `selfInterrupt()` 把标志还回去**。这样既不会破坏排队，又不丢中断语义。

> **这就是「AQS 不响应中断、但会保留中断状态」的实现真相**：`lock()` 不可中断，但锁到手后你会看到中断标志；`lockInterruptibly()` 则用 `doAcquireInterruptibly`，在 `parkAndCheckInterrupt` 返回 true 时直接 `throw new InterruptedException()`。

## 5.4 取消：`cancelAcquire`（源码里最难的一段）

线程因中断/超时/异常退出排队时，必须把自己标记 `CANCELLED` 并「从链上摘掉」。难点是**不能简单摘链**（其他线程正在遍历、CAS），所以分三种情况：

```java
private void cancelAcquire(Node node) {
    if (node == null) return;
    node.thread = null;                       // ① 释放 thread 引用

    Node pred = node.prev;
    while (pred.waitStatus > 0) node.prev = pred = pred.prev;  // ② 跳过前面已取消的

    Node predNext = pred.next;
    node.waitStatus = Node.CANCELLED;         // ③ 标记取消（这是最重要的，让别人看到后跳过它）

    if (node == tail && compareAndSetTail(node, pred)) {        // 情况 A：自己是尾巴
        compareAndSetNext(pred, predNext, null);                //   直接把 pred.next 置 null
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {                              // 情况 B：pred 是有效节点且能负责唤醒
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);        //   把 pred.next 直接连到 next（摘掉自己）
        } else {
            unparkSuccessor(node);                              // 情况 C：pred 是 head（自己是队首），
                                                                //   需要立刻唤醒后继，否则队列会卡死
        }
        node.next = node;                                       // ④ 自引用，标记「已摘除」，帮助 GC 并防止遍历死循环
    }
}
```

**情况 C 是最关键的**：如果取消的节点正好是 `head.next`（即将要拿锁的那个），而它的前驱 head 的 `waitStatus` 可能已被改成 `0`，那「委托唤醒」就断了链——没人会唤醒它的后继。所以必须**主动唤醒后继**，让后继上位。

> 面试延伸：「AQS 里为什么要 `node.next = node`？」→ 自引用使该节点在遍历时被识别为「已出队」（`ConcurrentLinkedQueue` 里同样用这种标记），同时也是让 GC 能回收它（断开与其他节点的单向引用）。

## 5.5 释放：`release` 与精确唤醒

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {            // 子类：state 减到 0 才算真释放
        Node h = head;
        if (h != null && h.waitStatus != 0)   // 【关键】head.waitStatus != 0 才需要唤醒
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0) compareAndSetWaitStatus(node, ws, 0);   // 清 SIGNAL，允许失败

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {        // 后继为空或已取消 → 从 tail 反向找
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) s = t;
    }
    if (s != null) LockSupport.unpark(s.thread);  // 只唤醒一个
}
```

**为什么 `h.waitStatus != 0` 才唤醒？** 若为 `0`，说明**没有后继设置过 SIGNAL**（没人要睡/已有人处理过），此时唤醒是「无效唤醒」，纯浪费系统调用。这是 AQS 减少无谓 `unpark` 的细节。

**为什么 `tryRelease` 必须返回 boolean？** 因为可重入锁在 `state` 还没减到 0 时**不能唤醒后继**（锁还没真释放）。`ReentrantLock.tryRelease`：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { free = true; setExclusiveOwnerThread(null); }  // 只有归零才算释放
    setState(c);                                                  // volatile 写 → 隐含 release 语义
    return free;
}
```

## 5.6 共享模式：`acquireShared` 与 `PROPAGATE` 之谜

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)          // 负数=失败，0=成功但无剩余，正数=成功且有剩余
        doAcquireShared(arg);
}
```

**返回值语义是共享模式的灵魂**：`Semaphore` 的 `tryAcquireShared` 返回「剩余许可数」，`CountDownLatch` 返回 `state == 0 ? 1 : -1`。AQS 用这个值决定「**要不要继续传播唤醒**」。

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean interrupted = false;
    for (;;) {
        final Node p = node.predecessor();
        if (p == head) {
            int r = tryAcquireShared(arg);
            if (r >= 0) {
                setHeadAndPropagate(node, r);   // 【核心】成功后再传播
                p.next = null;
                if (interrupted) selfInterrupt();
                return;
            }
        }
        if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
            interrupted = true;
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    // 三个条件任一成立就继续唤醒后继：
    if (propagate > 0 || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())      // 后继是共享节点（或未知）→ 传播
            doReleaseShared();
    }
}
```

### 5.6.1 为什么需要 `PROPAGATE`？——一个著名的并发 Bug

**场景推演（这是 AQS 历史上最经典的修复，务必能讲）**：

```text
初始：state=1（只有 1 个许可/资源），共享模式
队列：head(dummy, ws=0) → A(ws=SIGNAL，因为A设过) → B(ws=0)

Thread-1: releaseShared() → 释放资源，state=0
          doReleaseShared 看到 head.ws == SIGNAL → CAS 改为 0 → unpark(A)
Thread-A: 醒来 → tryAcquireShared 成功 → setHeadAndPropagate(A, r=0)
           此时 propagate == 0（没有剩余资源）
           判断：h.waitStatus < 0 ? h 是旧 head，已被上面改成 0 → false
                 新 head（就是 A）的 waitStatus 是多少？ A 入队时 ws=0...
                 → 也不小于 0 → false
           ⇒ 【不传播】，B 永远不被唤醒！

与此同时 Thread-2 又来 releaseShared()：
           doReleaseShared 看到 head.ws == 0 → 尝试 CAS(0 → PROPAGATE)
           若刚才 A 的 setHead 还没执行完（head 还是旧的），
           Thread-2 成功把 ws 设成 PROPAGATE → 但此时 unparkSuccessor 已执行过？或已经判断过？
           ⇒ 各种交错下，最终可能没人再唤醒 B。
```

**Doug Lea 的修复方案**：

1. 新增 `PROPAGATE` 状态（`-3`），表示「**共享模式释放正在进行传播，必须继续向后传**」。
2. `doReleaseShared` 里：看到 `ws == 0` 就 `CAS(0 → PROPAGATE)` 并**再 unpark 一次**，而不是直接跳过。
3. `setHeadAndPropagate` 里：只要 `h.waitStatus < 0`（包括 `PROPAGATE`）就传播。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) continue;  // CAS 失败重来
                unparkSuccessor(h);                                        // 唤醒后继
            } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;   // 【关键】把 0 改成 PROPAGATE，避免「最后一次释放丢失」
        }
        if (h == head) break;      // head 变了说明有人抢到了，继续循环检查
    }
}
```

> **一句话总结（可直接背）**：`PROPAGATE` 是为了解决「**多个线程同时 release、且唤醒与入队交错时，共享唤醒可能丢失**」的问题——它把「已经有人负责唤醒但还没唤醒完」这个中间状态显式编码进了 `waitStatus`。

## 5.7 超时与中断变体

| 方法 | 实现 | 中断行为 | 超时行为 |
|---|---|---|---|
| `acquire` | `acquireQueued` | 不响应，但保留标志（`selfInterrupt`） | 无 |
| `acquireInterruptibly` | `doAcquireInterruptibly` | `parkAndCheckInterrupt()` 为 true → `throw InterruptedException` + `cancelAcquire` | 无 |
| `tryAcquireNanos` | `doAcquireNanos` | 同中断版 | `System.nanoTime()` 计算剩余，`parkNanos`，超时 → `cancelAcquire` + 返回 false |
| `acquireSharedInterruptibly` | `doAcquireSharedInterruptibly` | 同上 | 无 |
| `tryAcquireSharedNanos` | `doAcquireSharedNanos` | 同上 | 同上 |

`doAcquireNanos` 的关键细节（易错点）：

```java
long lastTime = System.nanoTime();
while (true) {
    if (p == head && tryAcquire(arg)) { ... }        // 每轮先尝试
    nanosTimeout = deadline - System.nanoTime();      // 剩余时间
    if (nanosTimeout <= 0L) return false;            // 超时
    if (shouldParkAfterFailedAcquire(p, node) &&
        nanosTimeout > spinForTimeoutThreshold)       // 【关键】< 1000ns(≈1μs) 就不 park 了，直接自旋
        LockSupport.parkNanos(this, nanosTimeout);
    ...
}
```

`spinForTimeoutThreshold = 1000L`（纳秒）：剩余时间太短时，**park 的开销（μs 级）比等完它还大**，所以干脆空转。这是「用测量数据指导优化」的典型例子。

## 5.8 Condition：AQS 的第二个队列

`ConditionObject` 实现了 `java.util.concurrent.locks.Condition`，是 `wait/notify` 的 JUC 版，但**一个 `ReentrantLock` 可以有任意多个 `Condition`**，且支持公平唤醒。

```java
public class ConditionObject implements Condition {
    private transient Node firstWaiter;   // 条件队列（单向，用 nextWaiter 串起来）
    private transient Node lastWaiter;
}
```

**两个队列的关系（必须画出来）**：

```text
                    ┌──────────────── 同步队列（双向，prev/next）───────────────┐
   lock() 失败 ───▶  │ head(dummy) ⇄ N1 ⇄ N2 ⇄ N3 ⇄ tail                      │
                    └───────────────▲──────────────────────────┬──────────────┘
                                    │                          │ await()：释放锁 + 移到条件队列
                                    │ signal()：移回同步队列     ▼
                    ┌──────────────── 条件队列（单向，nextWaiter）──────────────┐
                    │ firstWaiter → C1 → C2 → lastWaiter                      │
                    └────────────────────────────────────────────────────────┘
```

### 5.8.1 `await()` 全流程

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();

    Node node = addConditionWaiter();        // ① 加入条件队列（waitStatus = CONDITION）
    int savedState = fullyRelease(node);     // ② 【完全释放锁】返回释放前的 state（保存重入次数！）
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {           // ③ 只要还在条件队列（没被 signal 移到同步队列）
        LockSupport.park(this);              //    就睡（这里用 this 作为 blocker，便于 jstack 定位）
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)  // ④ 被中断
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)  // ⑤ 重新竞争锁，恢复 state
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) unlinkCancelledWaiters();            // ⑥ 清理已取消的等待者
    if (interruptMode != 0) reportInterruptAfterWait(interruptMode);  // ⑦ 抛异常或补中断
}
```

**五个必须理解的细节**：

1. **`fullyRelease` 释放的是「全部」重入次数**。`state` 可能是 3（重入 3 次），`await` 必须把它全部释放，否则其他线程永远进不来。
2. **`savedState` 保存下来，`acquireQueued(node, savedState)` 恢复**。所以 `await` 返回后，重入次数自动还原——这点 `Object.wait()` 做不到（`wait` 不保存重入计数，`notify` 后只重新抢一次锁）。
3. **只有 `LockSupport.park` 这一处阻塞**，且 `park` 被中断会立即返回（不抛异常），所以要看 `isOnSyncQueue` 判断是否已被 `signal`。
4. **`checkInterruptWhileWaiting`**：若中断发生在被 `signal` 之前（还在条件队列）→ 返回 `THROW_IE`（需自己把它转移到同步队列并抛异常）；若中断发生在被 signal 之后 → `REINTERRUPT`（只补标志，不抛）。这个区分就是为了**保证「先被 signal 拿锁成功、再抛中断」不丢信号**。
5. **`unlinkCancelledWaiters`**：条件队列的单向链表靠 `nextWaiter` 串，取消的节点要单独清理（**遍历时发现问题才清理，不是实时**）。

### 5.8.2 `signal()` 全流程

```java
public final void signal() {
    if (!isHeldExclusively()) throw new IllegalMonitorStateException();  // 必须持有锁
    Node first = firstWaiter;
    if (first != null) doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null) lastWaiter = null;
        first.nextWaiter = null;                       // 从条件队列摘掉
    } while (!transferForSignal(first) && (first = firstWaiter) != null);  // 转移失败就试下一个
}

final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))   // 已取消？→ 返回 false
        return false;
    Node p = enq(node);                                      // 转移到同步队列尾部
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);                     // 【关键】前驱已取消/SIGNAL设置失败 → 立即唤醒
    return true;
}
```

**【关键】为什么这里要立即 `unpark`？** 因为被转移的节点挂在 `p` 后面，而 `p` 可能**已经取消**（`ws > 0`），那没人会唤醒它 → 必须现在就叫醒它，让它自己重新排队/抢锁。这是 AQS 的容错设计。

### 5.8.3 `await` 与 `wait` 的差异（面试常考对照表）

| | `Object.wait()` | `Condition.await()` |
|---|---|---|
| 前置条件 | 必须持有 monitor | 必须持有 lock |
| 条件队列数量 | 1 个（monitor 上） | 每个 Condition 独立一个 |
| 重入恢复 | 不恢复（只重抢一次） | **完整恢复 `savedState`** |
| 可中断 | 抛 `InterruptedException` | 抛 `InterruptedException`，且区分 signal 前后 |
| 超时版本 | `wait(long)` | `awaitNanos` / `awaitUntil`（`awaitUntil` 用绝对时间，更精确） |
| 虚假唤醒 | 存在 | 存在（必须 `while` 循环判断条件） |
| 公平性 | 无 | `signal` 后仍需按 AQS 队列顺序竞争 |

## 5.9 公平 vs 非公平的真实差异

```java
// 非公平（默认）：直接 CAS 抢，不管队列
final boolean nonfairTryAcquire(int acquires) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
    ...
}

// 公平：先看队列里有没有人在排
protected final boolean tryAcquire(int acquires) {
    if (getState() == 0 && !hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
        ...
    }
}

public final boolean hasQueuedPredecessors() {
    Node t = tail; Node h = head;
    Node s;
    return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

> **为什么非公平吞吐更高？** 「**插队**」让刚来的线程直接抢到锁，避免了「唤醒-调度-上下文切换」的 μs 级开销——锁的持有者刚释放、线程还在 CPU 上，立即抢到相当于零延迟交接。代价是**队首线程可能长期饥饿**（但实际很难，因为释放瞬间队首也会被唤醒，插队者需要恰好在这个窗口抢到）。
> **为什么公平模式仍然不是「严格公平」？** `hasQueuedPredecessors` 只挡「队列非空」时的插队，`tryAcquire` 和队首的 `tryAcquire` 之间仍有竞争窗口；且 OS 调度本身不均等。

## 5.10 AQS 的模板方法一览（背下来能应付 80% 的追问）

| 类别 | 方法 | 默认实现 | 说明 |
|---|---|---|---|
| 独占获取 | `tryAcquire(int)` | `throw UnsupportedOperationException` | 必须自己实现（AQS 故意不提供） |
| 独占释放 | `tryRelease(int)` | 同上 | |
| 共享获取 | `tryAcquireShared(int)` | 同上 | 返回负=失败，0=成功无剩余，正=成功有剩余 |
| 共享释放 | `tryReleaseShared(int)` | 同上 | |
| 是否独占持有 | `isHeldExclusively()` | 同上 | Condition 用 |
| 状态 | `getState/setState/compareAndSetState` | — | `state` 是 `volatile`，`compareAndSetState` 用 `Unsafe` |
| 队列观测 | `hasQueuedPredecessors`、`getQueueLength`、`hasContended`、`getFirstQueuedThread` | — | 调试/公平性用 |
| 所有权 | `getExclusiveOwnerThread/setExclusiveOwnerThread` | — | 继承自 `AbstractOwnableSynchronizer` |

> **为什么 AQS 的 `tryAcquire` 不写成 `abstract`？** 因为 AQS 同时支持独占和共享两种模式，一个子类通常只用其中一种。如果声明为 `abstract`，`Semaphore` 就必须实现 `tryAcquire`（独占），这没意义。所以用「抛不支持异常」的模板方法，由子类按需覆盖——**这是模板方法模式在「多策略可选」场景下的经典用法**（对比：如果只有一种策略，就应该用 `abstract`）。

---

# 第 6 章 基于 AQS 的锁组件

## 6.1 ReentrantLock

### 6.1.1 结构

```java
public class ReentrantLock implements Lock {
    private final Sync sync;          // 内部抽象类 Sync extends AQS
    // 两个子类：NonfairSync / FairSync
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    // state = 0 → 空闲；> 0 → 持有锁且重入次数为 state
    // isHeldExclusively(): getExclusiveOwnerThread() == Thread.currentThread()
    // newCondition(): return new ConditionObject()
}
```

**注意 `Sync` 自己实现了 `nonfairTryAcquire`**，供 `NonfairSync` 和 `tryLock()`（无超时版本）共用——因为 `tryLock()` 不管是不是公平模式，**都必须是非公平的**（它就是在「试一下现在有没有机会」，公平排队会和「try」的语义冲突）。

### 6.1.2 加锁/解锁的完整链条

```text
lock()
  └─ sync.lock()
       ├─ NonfairSync: if (CAS(state,0,1))  → 成功：owner=current
       │                else acquire(1)    → 走 AQS 队列
       └─ FairSync:    acquire(1)          → 必定走 AQS（tryAcquire 里有 hasQueuedPredecessors）
            └─ acquire(1) → tryAcquire(1) 失败 → addWaiter → acquireQueued → park

unlock()
  └─ sync.release(1)
       └─ tryRelease(1): state-- ; 若 state==0 → owner=null 返回 true
            └─ true → unparkSuccessor(head)  → 唤醒队首
```

**为什么 `unlock()` 必须自己调用、且必须在 `finally` 里？**
AQS 没有「引用计数式自动解锁」，`unlock()` 少调一次就永久死锁；多调一次抛 `IllegalMonitorStateException`（由 `tryRelease` 里的 `owner` 检查抛出）。

### 6.1.3 常见误解纠正

| 误解 | 事实 |
|---|---|
| 「ReentrantLock 比 synchronized 快」 | 低竞争时 `synchronized`（偏向/轻量级）反而更快；`ReentrantLock` 每次排队都要 `new Node` |
| 「公平锁性能差是因为要排队」 | 主因是**每次释放都要唤醒队首 + 上下文切换**，且无法「就地交接」 |
| 「`tryLock()` 不会排队」 | 无超时的 `tryLock()` 不排队（一次 CAS）；带超时的会排队并 `parkNanos` |
| 「`lockInterruptibly` 能被中断，所以更安全」 | 它只影响**等锁阶段**；一旦拿到锁执行业务，中断仍要靠业务自己检查标志 |

## 6.2 ReentrantReadWriteLock（读写锁）

### 6.2.1 一个 int 装两种锁：`state` 的位域切分

```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);        // 0x00010000
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;    // 65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;    // 0x0000FFFF

static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }      // 高 16 位：读锁数量
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }      // 低 16 位：写锁重入数
```

```text
32 位 state：
┌─────────────────────────┬─────────────────────────┐
│  高 16 位 = 读锁持有总数   │  低 16 位 = 写锁重入次数   │
│  (sharedCount)           │  (exclusiveCount)        │
└─────────────────────────┴─────────────────────────┘
读锁加锁：state += 1 << 16
写锁加锁：state += 1
⇒ 读锁和写锁可以在同一个 CAS 里互不干扰地增减，这是「用一个变量同时表达两种锁」的关键。
```

### 6.2.2 每个线程的读锁重入计数：`HoldCounter` + `ThreadLocal`

`state` 只记录**全局读锁总数**，但「我这个线程重入了几次」必须单独记，否则无法判断「能不能释放」以及「最后一个读线程要不要唤醒写线程」。

```java
static final class HoldCounter {
    int count = 0;
    final long tid = getThreadId(Thread.currentThread());  // 用 tid 而不是 Thread 引用，避免内存泄漏
}
static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
    protected HoldCounter initialValue() { return new HoldCounter(); }
}
private transient ThreadLocalHoldCounter readHolds;      // 每个线程一份
private transient HoldCounter cachedHoldCounter;         // 【优化】缓存「最后一个成功获取读锁的线程」的计数
private transient Thread firstReader;                    // 第一个读线程（直接引用，省 ThreadLocal 查找）
private transient int firstReaderHoldCount;
```

**三层缓存设计（性能优化，面试加分点）**：

1. `firstReader / firstReaderHoldCount`：**只有一个读线程时**（最常见），直接用字段，零开销；
2. `cachedHoldCounter`：有多个读线程时，缓存「最近一个」的计数器（因为读锁获取/释放往往成对出现在同一线程）；
3. `readHolds`（ThreadLocal）：兜底，任意线程任意重入。

### 6.2.3 写锁与读锁的获取条件

```java
// 写锁 tryAcquire
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 有读锁(w==0 但 c!=0) 或 别的线程持有写锁 → 失败
        if (w == 0 || current != getExclusiveOwnerThread()) return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT) throw new Error("Maximum lock count exceeded");
    }
    if ((w == 0 && writerShouldBlock()) ||          // 公平模式：队列有前驱则阻塞
        !compareAndSetState(c, c + acquires)) return false;
    setExclusiveOwnerThread(current);
    return true;
}

// 读锁 tryAcquireShared
protected final int tryAcquireShared(int unused) {
    if (exclusiveCount(getState()) != 0 && holder != current) return -1; // 写锁被别人占 → 失败
    ...
    if (readerShouldBlock()) { /* 公平：排队；非公平：若队首是写线程也要让路，防写饥饿 */ }
    ...
}
```

> **非公平模式为什么也要判断「队首是写线程就阻塞读」？**
> 否则读锁源源不断地插队（读锁是共享的，一次能进一堆），写锁永远轮不到 → **写饥饿**。`readerShouldBlock()` 的非公平实现是 `apparentlyFirstQueuedIsExclusive()`（队首是独占节点就返回 true）。这是读写锁里最容易被问倒的细节。

### 6.2.4 锁降级（Lock Downgrade）与「不支持升级」

```java
// 锁降级：持有写锁 → 获取读锁 → 释放写锁（合法）
writeLock.lock();
try {
    readLock.lock();          // ① 在持有写锁时获取读锁（允许！）
} finally {
    writeLock.unlock();       // ② 释放写锁，此时仍持有读锁
}
// 之后可以继续读，其他读线程也能进
```

**为什么降级是必要的？** 为了保证「**释放写锁到获取读锁之间，没有其他线程修改数据**」这个原子性——否则你以为自己读的是刚写的数据，其实已经被别人改了（经典的「缓存 + 数据库」场景：写缓存后需要保证读到一致视图）。

**为什么不支持升级（读锁 → 写锁）？** 会死锁：两个线程都持有读锁，都想升级为写锁，但写锁要求「没有任何读锁」，于是互等。

> 面试延伸：「锁降级时，`readLock.lock()` 会不会被自己持有的写锁挡住？」→ 不会。`tryAcquireShared` 里 `exclusiveCount != 0 && holder != current` 的判断专门放行「当前线程持有写锁」的情况。

### 6.2.5 读写锁的适用边界

| 场景 | 建议 |
|---|---|
| 读多写极少（读:写 > 10:1），且读操作耗时 | ✅ 读写锁有价值 |
| 读多写少，但读操作极短（几条指令） | ❌ 读写锁比 `synchronized` 还慢（多了一次 CAS + 计数维护） |
| 读:写 ≈ 1:1 或写多 | ❌ 直接 `ReentrantLock` |
| 读多写少且能容忍「读到旧值」 | ✅✅ `StampedLock` 乐观读（性能最好） |

## 6.3 StampedLock（JDK8 引入，非 AQS 实现）

### 6.3.1 三种模式与 `stamp`

```java
long stamp = lock.tryOptimisticRead();   // ① 乐观读：不阻塞、不加锁，只返回一个版本号
int x = data;                            // ② 读数据（普通读，可能读到不一致）
if (!lock.validate(stamp)) {             // ③ 校验：期间有没有人写过？
    stamp = lock.readLock();             //    有 → 退化为悲观读锁
    try { x = data; } finally { lock.unlockRead(stamp); }
}
```

`state` 的位域：

```text
64 位 long state：
┌────┬──────────────┬──────────────┐
│  0 │ 写锁标记(1bit) │ 读锁计数(7bit) │ ... 其余为版本号(epoch)
└────┴──────────────┴──────────────┘
每次「写锁释放」都会让版本号 +1，所以 validate 本质上是在问「版本号有没有变过」。
```

### 6.3.2 核心特性与陷阱

| 特性 | 说明 |
|---|---|
| **乐观读** | 不加锁、不写 state，只读版本号；`validate` 失败才退化，**读多写少场景吞吐远超读写锁** |
| **不可重入** | 同一线程重复 `readLock()` 会死锁（没有重入计数） |
| **不支持 Condition** | 没有 `newCondition()` |
| **不支持中断** | `readLockInterruptibly` 有，但语义较弱 |
| **stamp 必须配对** | `unlockWrite(stamp)` 传错 stamp 会引发难以排查的问题 |
| **写锁饥饿** | 无公平策略，写锁可能长期抢不到（官方文档明说） |
| **不能替代读写锁做嵌套** | 因为不可重入，嵌套调用极易死锁 |

> **实践结论**：`StampedLock` 只适合「**结构简单、读多写少、无嵌套、能接受偶发重读**」的场景（如缓存快照、配置读取）。生产环境里 90% 的情况用 `ReentrantReadWriteLock` 或干脆 `ConcurrentHashMap` 更安全。

---

# 第 7 章 并发容器

## 7.1 ConcurrentHashMap：从分段锁到 CAS + synchronized

### 7.1.1 JDK7 与 JDK8 的结构对比

```text
JDK7：Segment 分段锁（继承 ReentrantLock）
┌──────────────────────────────────────────────────────────┐
│ ConcurrentHashMap                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐        ┌─────────┐  │
│  │Segment0 │ │Segment1 │ │Segment2 │  ...   │Segment15│  │
│  │(Lock)   │ │(Lock)   │ │(Lock)   │        │(Lock)   │  │
│  │ HashEntry[]  每个 Segment 是一个小 HashMap + 一把锁     │
│  └─────────┘ └─────────┘ └─────────┘        └─────────┘  │
│  并发度 = 16（默认），最多 16 个线程同时写                  │
└──────────────────────────────────────────────────────────┘

JDK8：Node 数组 + 桶级锁（CAS 空桶 + synchronized 头节点）
┌──────────────────────────────────────────────────────────┐
│ table[]  (volatile Node<K,V>[]，长度 2 的幂)              │
│  [0] → Node → Node → Node        （链表）                 │
│  [1] → null                      （空桶，put 用 CAS）    │
│  [2] → TreeBin（红黑树，hash 冲突 ≥ 8 且容量 ≥ 64）        │
│  [3] → ForwardingNode（扩容中，指向 nextTable）            │
│  ...                                                     │
│  锁粒度 = 一个桶（链表头 / 树根），并发度 ≈ 桶数量           │
└──────────────────────────────────────────────────────────┘
```

**为什么 JDK8 放弃分段锁？**

| 分段锁的问题 | JDK8 的改进 |
|---|---|
| 并发度被固定为 16（段数），桶多了也没用 | 并发度 ≈ 表长度（可到百万级） |
| 每个 Segment 是一个完整 HashMap，内存浪费 | 一个数组 + 节点，内存更紧凑 |
| 段内哈希冲突仍然退化为链表，锁住整段 | 只锁冲突的那条链/树 |
| `size()` 维护复杂（段内 count + 重试） | `baseCount + CounterCell[]` 分片计数 |
| 扩容只能段内扩（不同段长度可能不一致） | 全表扩容 + **多线程协助搬迁** |

### 7.1.2 关键常量与"特殊 hash 值"

```java
static final int MOVED     = -1;   // ForwardingNode：该桶已迁移，请去 nextTable 找
static final int TREEBIN   = -2;   // TreeBin 节点（红黑树根包装）
static final int RESERVED  = -3;   // ReservationNode：computeIfAbsent 占位
static final int HASH_BITS = 0x7fffffff;

// 计算 key 的 hash：高 16 位与低 16 位异或，让高位参与定位，减少冲突
static final int spread(int h) { return (h ^ (h >>> 16)) & HASH_BITS; }

static final int TREEIFY_THRESHOLD   = 8;   // 链表 → 树
static final int UNTREEIFY_THRESHOLD = 6;   // 树 → 链表
static final int MIN_TREEIFY_CAPACITY = 64; // 容量 < 64 时优先扩容而不是树化
```

> **为什么阈值是 8 和 6？** 泊松分布计算：在 0.75 负载因子下，一个桶里放 8 个元素的概率约 `0.00000006`（千万分之六）。也就是说链表长度到 8 时，**几乎必然是恶意哈希或极坏运气**，此时转树的收益大于成本。
> **为什么树化要求容量 ≥ 64？** 小表时哈希冲突多是因为表太短，**扩容比转树的收益更大**（扩容能直接分散冲突，且成本更低）。

### 7.1.3 `sizeCtl`：一个 int 的四种含义（超级容易答错）

| 值 | 含义 |
|---|---|
| `0` | 默认值，尚未初始化（首次 put 时按默认 16 初始化） |
| `-1` | **正在初始化**（`initTable` 的 CAS 占位，其他线程 `Thread.yield()` 等待） |
| `-(1 + n)` | **正在扩容**，`n` 是参与扩容的线程数；高 16 位是 `resizeStamp` |
| `> 0` | **扩容阈值**（= 容量 * 0.75），初始化完成后由它决定何时扩容 |

```java
// 扩容中的判断（关键位运算）
int rs = resizeStamp(n);           // 由 n 算出 stamp：高 16 位，第 16 位置 1
if (sc < 0) {
    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
        sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
        break;                     // 已有人在扩 / 扩完了 / 没活可领 → 不参与
    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))   // 领到一个"扩容工位"
        transfer(tab, nt);
}
```

```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    // numberOfLeadingZeros(16)=27（32位里前导0的个数），再与 0x8000 或 → 0x801B
}
```

**为什么用「前导零个数」当 stamp？** 因为表长度是 2 的幂，其二进制「1 的位置」唯一，`numberOfLeadingZeros` 恰好能编码这个位置（不同容量得到不同 stamp），且**能塞进 16 位**，与低位的线程计数拼在同一个 int 里。

### 7.1.4 `put` 全流程（**必须能默写流程**）

```java
public V put(K key, V value) { return putVal(key, value, false); }

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();  // ① 不允许 null！
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {                     // ② 死循环（CAS 失败重试）
        Node<K,V> f; int n, i, fh; K fk; V fv;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();                            // ③ 懒初始化（CAS sizeCtl=-1 抢初始化权）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                                    // ④ 空桶：CAS 直接放，无锁！
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);                   // ⑤ 正在扩容：帮一把，然后重来
        else {
            synchronized (f) {                            // ⑥ 只锁这一个桶的头节点
                if (tabAt(tab, i) == f) {                 //    双重检查（锁前可能被替换）
                    if (fh >= 0) {                        //    链表
                        for (Node<K,V> e = f; ; e = e.next) {
                            if (e.hash == hash && e.key.equals(key)) {  // 命中 → 更新
                                if (!onlyIfAbsent || e.val == null) e.val = value;
                                return e.val;
                            }
                            if (e.next == null) {         //    尾插
                                e.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { ... }  // 红黑树插入
                    else if (f instanceof ReservationNode) throw new IllegalStateException(...);
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) treeifyBin(tab, i);  // ⑦ 树化
                if (!onlyIfAbsent) ...;
                break;
            }
        }
    }
    addCount(1L, binCount);       // ⑧ 计数 + 判断是否扩容
    return null;
}
```

**六个关键点（面试逐条能讲）**：

1. **不允许 null key/value**：因为并发下无法用 `null` 表达「不存在」（`get` 返回 null 到底是没找到还是有 null 值？），而 `HashMap` 允许。
2. **首次初始化 `initTable`**：CAS 把 `sizeCtl` 从 0 改成 -1 抢到初始化权，其他线程 `Thread.yield()` 自旋等 `table` 可见（依靠 `sizeCtl` 的 volatile 读写建立 hb）。
3. **空桶用 CAS 写**：这是**无锁的快速路径**，绝大多数 put 落在这里。
4. **非空桶用 `synchronized(f)`**：JDK8 用内置锁而不是 `ReentrantLock`，因为**桶级锁临界区极短**（几个节点遍历），偏向/轻量级锁几乎零成本，且**省去了每个桶维护一个 Lock 对象的内存**（几十万个桶，每个一个 Lock 不可接受）。
5. **`synchronized` 内必须 `if (tabAt(tab, i) == f)` 二次检查**：因为在拿到锁之前，桶头可能已被扩容搬走/被树化替换。
6. **尾插而不是头插**：为了**迁移时的稳定性**（头插会导致迁移中链表顺序翻转，可能造成环或重复，JDK7 的 HashMap 头插并发扩容就出现过环）。

### 7.1.5 扩容 `transfer`：多线程协助搬迁

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // ① 每个线程负责的「桶区间」大小：至少 16，按 CPU 数均分
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;                     // 16
    if (nextTab == null) {                                // ② 第一个来的线程负责创建 nextTable
        nextTab = new Node[n << 1];
        nextTable = nextTab;
        transferIndex = n;                                // 从后往前分配任务
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<>(nextTab);
    boolean advance = true;
    for (int i = 0, bound = 0;;) {
        // ③ 领任务：CAS 把 transferIndex 减去 stride，[bound, i) 就是我的区间
        while (advance) { ... }
        if (i < bound) { ... } 
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);        // ④ 空桶直接放 ForwardingNode
        else if ((fh = f.hash) == MOVED) advance = true;  //    已处理过
        else {
            synchronized (f) {                            // ⑤ 锁住桶头，拆分链表
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;                     //    ln=低位链, hn=高位链
                    if (fh >= 0) {
                        // 用 hash & n 判断元素该去低位还是高位（因为容量翻倍，多了一位参与）
                        ...
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);            // ⑥ 旧桶标记 MOVED
                        advance = true;
                    } else if (f instanceof TreeBin) { ... split 红黑树 ... }
                }
            }
        }
    }
}
```

**核心机制总结**：

| 机制 | 作用 |
|---|---|
| `stride`（默认 16） | 每次领 16 个桶，减少 CAS 竞争 |
| `transferIndex`（volatile） | 从 `n` 递减，各线程 CAS 领任务 → **无锁任务分发** |
| `ForwardingNode` | 标记「已迁移」，让 `get` 去 `nextTable` 找，让 `put` 帮忙扩容 |
| `helpTransfer` | 一个线程发现桶是 MOVED，就加入扩容（`sizeCtl + 1`），做完再回来 put |
| 低/高位链 | `hash & n`（新多出的那一位）决定去 `i` 还是 `i+n`，**不需要重算 hash** |
| 树拆分 | `TreeBin` 用 `split` 按同样规则拆成两条链，若某条 ≤ 6 则退化为链表 |

> **为什么扩容是「翻倍」而不是「rehash 到质数」？** 因为翻倍后 `hash & (2n-1)` 只比 `hash & (n-1)` 多一位，元素要么留在原位、要么移到 `i+n`，**天然二分**，无需重新计算 hash。这就是「容量必须是 2 的幂」的原因。

### 7.1.6 `get`：完全无锁

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {          // volatile 读桶头
        if ((eh = e.hash) == h) {                         // 头节点命中
            if ((ek = e.key) == key || (ek != null && key.equals(ek))) return e.val;
        }
        else if (eh < 0)                                   // MOVED / TREEBIN / RESERVED
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {                     // 链表遍历
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

**为什么无锁也安全？** 三个理由：

1. `table` 与桶头 `tabAt` 都是 **volatile 读**，能看到最新的节点引用；
2. `Node.val` 是 `volatile`，节点一旦挂到链上就不会被原地改（更新是**新建节点**还是**改 val**？→ `e.val = value` 是直接改，靠 `val` 的 volatile 保证可见）；
3. 遍历中即使遇到扩容，`ForwardingNode.find` 会转到新表继续找；链表节点不会被「删除后断裂」（删除是 `e.next = e.next.next` 式短路，但遍历已经持有当前节点的引用，不会受影响）。

### 7.1.7 计数：`baseCount + CounterCell[]`

```java
private transient volatile long baseCount;
private transient volatile CounterCell[] counterCells;   // 类似 LongAdder

private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;
    if ((cs = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // ① CAS baseCount 失败 → 走高竞争路径，分片加
        CounterCell c; long v; int m;
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended = U.compareAndSwapLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);     // ② 初始化/扩容 cells 数组
            return;
        }
        if (check <= 1) return;
        s = sumCount();
    }
    if (check >= 0) {
        // ③ 判断是否需要扩容：s >= sizeCtl 且 table 未到最大 → transfer
        ...
    }
}
```

**要点**：`size()` 返回 `sumCount()`，需要**累加所有 Cell**，因此是**弱一致**的（`size()` 期间有新写入就不准）。这就是为什么 `ConcurrentHashMap.size()` 在并发场景下只能当「估算值」，不能当精确计数（精确计数要用 `mappingCount()` 也不精确，或加锁）。

### 7.1.8 CHM 的迭代器：弱一致性（fail-safe）

```java
// 迭代器创建时记录「当时的 table 引用」
final Node<K,V> nextNode() {
    Node<K,V>[] t; Node<K,V> e = next;
    if (e == null) throw new NoSuchElementException();
    if ((next = (e.next)...) == null) advance();
    // 遍历中不抛 ConcurrentModificationException
}
```

- **不抛 CME**：因为不使用 `modCount` 机制，允许并发修改。
- **可能看到「创建之后」的新增元素**，也可能看不到（取决于桶是否已经遍历过）。
- 因此**不保证**「遍历开始时刻的一致性快照」，只保证「每个元素最多被访问一次，不返回不存在的元素」。

### 7.1.9 与其他 Map 的对比（面试对照表）

| | `HashMap` | `Hashtable` | `ConcurrentHashMap` | `Collections.synchronizedMap` |
|---|---|---|---|---|
| 线程安全 | ❌ | ✅（全表锁，`synchronized` 方法） | ✅（桶级锁 + CAS） | ✅（全表锁，包装对象） |
| null key/value | ✅ 允许（key 一个 null） | ❌ | ❌ | 取决于内部 Map |
| 并发度 | — | 1 | ≈ 表长度 | 1 |
| 迭代器 | fail-fast（抛 CME） | fail-fast | **fail-safe（弱一致）** | 需手动 `synchronized(map)` 遍历 |
| 扩容 | 单线程 rehash | 单线程 rehash | 多线程协助 | 底层决定 |
| 读性能 | 快 | 需抢锁 | **无锁读** | 需抢锁 |
| 适用 | 单线程 | 已淘汰 | 高并发 | 低并发过渡方案 |

> **延伸易错点**：`ConcurrentHashMap` 的 `computeIfAbsent` 里**不要做递归写同一个 map**——`ReservationNode` 占位时会抛 `IllegalStateException`（"Recursive update"），这是 JDK9 加入的保护。

## 7.2 CopyOnWriteArrayList / CopyOnWriteArraySet

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();              // ① 读当前数组（volatile）
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);  // ② 全量复制
        newElements[len] = e;
        setArray(newElements);                       // ③ volatile 写，一次切换引用
        return true;
    } finally { lock.unlock(); }
}

public E get(int index) { return elementAt(getArray(), index); }  // 完全无锁
```

**设计哲学**：**「读的无锁」是用「写的全量复制」换来的**。

| 优点 | 代价 |
|---|---|
| 读零同步、零阻塞，甚至可以边读边写 | 写操作 O(n) 复制 + 内存翻倍 |
| 遍历天然快照一致（拿到的是旧数组，不会被改） | **内存占用高**，大 list 可能触发 Full GC |
| 实现极简，无锁 Bug | **弱一致**：读到的是「某一时刻的快照」，看不到后续 add |
| — | `iterator.remove()` 不支持（`UnsupportedOperationException`） |

**适用场景**：**读远多于写，且 list 很小**（监听器列表、白名单、配置项）。`size` 超过几千就基本不能用。

## 7.3 ConcurrentLinkedQueue：Michael-Scott 无锁队列

### 7.3.1 结构

```java
public class ConcurrentLinkedQueue<E> {
    private transient volatile Node<E> head;
    private transient volatile Node<E> tail;
    // Node: volatile E item; volatile Node<E> next;
}
```

**关键：`head`/`tail` 都不是精确的头/尾**，而是「**松弛（slack）**」的指针：

- `head` 可能指向一个已出队的节点（`item == null`），真正的队首是 `head.next`；
- `tail` 通常落后真正队尾一步（指向最后一个节点的前一个），这样**入队时不必每次都 CAS tail**。

```text
初始：head → [null]         tail → 同一个节点（都是哨兵）
       └──────────────────────┘

入队 A、B、C 后（tail 松弛）：
head → [null] → [A] → [B] → [C]
                        ↑tail（滞后一格）
真正队尾 = tail.next = [C]
```

### 7.3.2 `offer` 的循环逻辑

```java
public boolean offer(E e) {
    final Node<E> newNode = new Node<E>(e);
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {                              // ① p 是真正的尾
            if (casNext(p, null, newNode)) {          //     CAS 挂上
                if (p != t) casTail(t, newNode);      // ② 只有 p 落后一格才更新 tail
                return true;
            }
        }
        else if (p == q)                              // ③ p 指向自己 → 说明已被出队（自链接）
            p = (t != (t = tail)) ? t : head;         //    重新定位（可能 tail 已变，否则从头找）
        else                                          // ④ p 后面还有节点 → 往前挪
            p = (q != null) ? q : t;
    }
}
```

**三个关键设计**：

1. **两步走**（`p.next == null` 时 CAS；成功后**只在 `p != t` 时**才更新 tail）：这就是「松弛」——**tail 更新是可选的优化，不是正确性的必要**。减少了 CAS 次数（每次入队最多 1 次 CAS，而不是 2 次）。
2. **`p == q` 的自链接**：节点出队时会把 `next` 指向自己（`updateHead` 里 `h.next = h` 或类似），表示「这个节点已失效」，遍历时要跳过它并重新定位。
3. **`p = (t != (t = tail)) ? t : head`**：这行是「无锁编程里典型的 double-check」——先看 tail 有没有被别人改过（改过就用新 tail），没改就从 head 重新走（保证一定能找到尾巴）。

### 7.3.3 `poll` 与 `updateHead`

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            if (item != null && casItem(p, item, null)) {   // ① CAS 把 item 置 null（占位出队）
                if (!p.casNext(null, ...)) ...;              // ② 让 p 成为新 head
                updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) { updateHead(h, p); return null; }  // 空队列
            else if (p == q) continue restartFromHead;        // 自链接 → 重来
            else p = q;
        }
    }
}
```

**`poll` 用 `casItem(p, item, null)` 而不是 CAS 整个节点**：因为「**把 item 置 null 就是"逻辑出队"的标记**」，其他线程看到 `item == null` 就知道这个节点已被别人拿走了。

> **`size()` 为什么是 O(n)？** 因为无锁队列没法维护精确计数（维护计数本身就是个 CAS 热点）。所以 `size()` 只能遍历；文档明确说「返回的是**近似值**，遍历过程中可能变化」。

### 7.3.4 `ConcurrentLinkedDeque` 为什么更复杂

双向链表要同时维护 `prev` 和 `next` 两条链，且**头尾都可以操作**，因此：

- 节点有 `prev`，出队时要同时修 `prev.next` 和 `next.prev`（两次 CAS，中间态不一致）；
- 用「**哨兵 + 特殊标记值**」处理中间态：HEAD 节点、`PREV_TERMINATOR` / `NEXT_TERMINATOR`；
- 代码量是 CLQ 的 3 倍，性能却差不多（除非真的需要双端操作，**不要用 `ConcurrentLinkedDeque` 替代 CLQ**）。

## 7.4 阻塞队列家族总览

```text
BlockingQueue（接口）
├── ArrayBlockingQueue       有界，数组，1 把锁 + 2 个 Condition
├── LinkedBlockingQueue      可选界，链表，2 把锁 + 2 个 Condition
├── PriorityBlockingQueue    无界，堆，1 把锁 + 1 个 Condition
├── DelayQueue               无界，堆 + 延迟，1 锁 1 Condition + Leader
├── SynchronousQueue         容量 0，直接交接（自研栈/队列）
├── LinkedTransferQueue      无界，链表，xfer 四模式（最强的"交接"实现）
└── LinkedBlockingDeque      双端，1 把锁 + 2 个 Condition
```

### 7.4.1 ArrayBlockingQueue：为什么只用一把锁？

```java
public class ArrayBlockingQueue<E> {
    final Object[] items;
    int takeIndex;                       // 下一个出队位置（非 volatile）
    int putIndex;                        // 下一个入队位置（非 volatile）
    int count;
    final ReentrantLock lock;            // 一把锁同时保护 put/take
    private final Condition notEmpty;
    private final Condition notFull;
}
```

**为什么不能像 LinkedBlockingQueue 那样用两把锁？**
因为**循环数组的 `putIndex`/`takeIndex` 会相互纠缠**：当队列满时 `putIndex == takeIndex`，当队列空时也 `putIndex == takeIndex`（只靠 `count` 区分）。如果 put 和 take 并发修改这两个下标，必须保证二者**在同一个临界区内**看到一致的 `count/index` 组合，否则会出现「下标错位导致覆盖数据」。

**理论上可以拆两把锁**（用 `AtomicInteger` 维护 count，并让 put 只改 putIndex、take 只改 takeIndex），但**循环回绕时仍需同步**（`putIndex = (putIndex+1) % capacity` 跨过 `takeIndex` 时），复杂度剧增，收益有限。所以 Doug Lea 选择简单方案。

**对比：`LinkedBlockingQueue` 为什么能两把锁？**
链表节点的插入/删除只与**相邻节点指针**有关，`head` 和 `last` 是**两个独立的不相交引用**：

```java
// LinkedBlockingQueue
private final ReentrantLock takeLock = new ReentrantLock();
private final Condition notEmpty = takeLock.newCondition();
private final ReentrantLock putLock = new ReentrantLock();
private final Condition notFull  = putLock.newCondition();
private final AtomicInteger count = new AtomicInteger();   // ← 关键：计数用原子类，跨两把锁共享
```

**「级联唤醒（cascading notification）」的设计（面试高频细节）**：

```java
// put 之后
int c = count.getAndIncrement();
if (c + 1 < capacity) notFull.signal();    // ① 队列还没满 → 唤醒另一个生产者
if (c == 0) signalNotEmpty();              // ② 之前是空队列 → 唤醒消费者

// take 之后
int c = count.getAndDecrement();
if (c > 1) notEmpty.signal();              // ③ 还有元素 → 唤醒另一个消费者
if (c == capacity) signalNotFull();        // ④ 之前是满队列 → 唤醒生产者
```

**为什么 `c + 1 < capacity` 才 signal？**
如果已经满了，signal 一个生产者醒来发现进不去，又要 park——**无效唤醒**。所以只在「进队后仍有空位」时唤醒，避免惊群；且**只唤醒一个**（不是 `signalAll`），因为每个 put 只产生一个空位。

> **性能提示**：`LinkedBlockingQueue` 默认容量 `Integer.MAX_VALUE`（`Executors.newFixedThreadPool` 就用它），等于**无界**，生产速度 > 消费速度时会 OOM。这是「线程池必须用有界队列」的最直接理由。

### 7.4.2 SynchronousQueue：容量为 0 的「手递手」

```java
public class SynchronousQueue<E> implements BlockingQueue<E> {
    // 公平模式：TransferQueue（FIFO）
    // 非公平模式（默认）：TransferStack（LIFO）
}
```

**语义**：`put(e)` 会**阻塞直到有消费者 `take()` 接走**；反之 `take()` 阻塞直到有人 `put`。**没有任何缓冲**，所以 `size()` 恒为 0，`peek()` 恒为 null，`iterator()` 空。

**非公平模式 `TransferStack` 的内部结构（三种模式）**：

```java
static final int REQUEST = 0;      // 消费者在等数据
static final int DATA    = 1;      // 生产者有数据
static final int FULFILLING = 2;   // 正在匹配中（有线程正在完成配对）

static final class SNode {
    volatile SNode next;
    volatile SNode match;          // 匹配到的对端
    volatile Thread waiter;
    Object item;                   // DATA 模式下的数据
    int mode;
}
```

`transfer`（简化）的逻辑，每次看栈顶 `h`，分三种情况：

```text
① h == null 或 h.mode != mode：栈顶不匹配（比如我是生产者，栈顶是生产者）
   → 把自己的节点压栈，然后 park，等待别人来匹配

② h.mode == mode：栈顶类型和我一样（都是生产者或都是消费者）→ 继续压栈

③ h 是 FULFILLING（别人正在匹配）→ 帮忙推进（clean 栈）

④ h.mode != mode 且不是 FULFILLING → 我尝试匹配：
   CAS 把栈顶的 mode 设为 FULFILLING，然后 CAS h.match = 自己
   → 成功：唤醒 waiter，返回
   → 失败：说明被别人抢先，重试
```

**为什么默认非公平？** 因为 LIFO 栈能让「**刚阻塞的线程**」优先被匹配，它的数据还在缓存里，上下文切换代价最低（生产者-消费者交替时几乎不用真正 park）。

**Java 里为什么用它？** `Executors.newCachedThreadPool()` 用它做任务队列——**每个任务都直接交接给空闲线程，不排队**，配合 `maximumPoolSize = Integer.MAX_VALUE`，实现「来一个任务开一个线程，空闲 60s 回收」。

### 7.4.3 PriorityBlockingQueue：无界 + `tryGrow` 的「释放锁再扩容」

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);          // ← 需要扩容
    try { siftUp(n, e); } finally { lock.unlock(); }
    notEmpty.signal();
    return true;
}

private void tryGrow(Object[] array, int oldCap) {
    lock.unlock();                     // ① 先释放锁！（扩容很慢，不能堵住 put/take）
    Object[] newArray = null;
    if (allocationSpinLock == 0 &&
        U.compareAndSwapInt(this, ALLOCATIONSPINLOCK, 0, 1)) {  // ② 抢占扩容权
        try {
            int newCap = oldCap + ((oldCap < 64) ? (oldCap + 2) : (oldCap >> 1));
            ...
            newArray = new Object[newCap];
        } finally { allocationSpinLock = 0; }
    }
    if (newArray == null) Thread.yield();      // ③ 别人在扩，让出 CPU
    lock.lock();                                // ④ 重新加锁
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

**三个精妙点**（面试可以讲出「为什么不用 AQS 而用自旋锁」）：

1. **先 `unlock()` 再扩容**：扩容是 O(n) 的内存操作，如果持锁做，会把所有 put/take 堵住数十毫秒。这里**释放锁 → 扩容 → 重新加锁**，用 `queue == array` 的判断防止「别人已经扩好了」。
2. **用 `allocationSpinLock`（自旋 CAS）而不是 AQS**：因为只需要保证「只有一个线程在 new 数组」，临界区极短，且**扩容期间不能让等待者排队**（否则唤醒逻辑复杂）。
3. **`Thread.yield()`**：抢不到扩容权就等一下，避免忙等。

**为什么它是无界且 put 永不阻塞？** 因为它是**优先级队列**，容量理论上可以无限增长（`int` 上限），`put` 只需保证入堆有序。所以它**没有 `notFull` 条件**，只有 `notEmpty`。

### 7.4.4 DelayQueue：Leader-Follower 模式的教科书实现

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E> {
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();  // 按剩余延迟排序
    private Thread leader;                                      // 只有 leader 用带超时的等待
    private final Condition available = lock.newCondition();
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) available.await();           // ① 队列空 → 无限等
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0L) return q.poll();            // ② 到点了 → 直接取
                first = null;                                 // ③ 释放引用（防止 leader 持有导致内存泄漏）
                if (leader != null) available.await();       // ④ 已有人在等 → 我也无限等
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;                      // ⑤ 我当 leader
                    try { available.awaitNanos(delay); }      //    只等到「队首到期时刻」
                    finally { if (leader == thisThread) leader = null; }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null) available.signal();  // ⑥ 交接：唤醒下一个当 leader
        lock.unlock();
    }
}
```

**为什么需要 Leader-Follower？**
如果所有等待线程都用 `awaitNanos(队首delay)`，那么队首到期时会**同时醒来 N 个线程**抢一个元素，N-1 个白醒（惊群）。
Leader 模式让**只有一个线程**用带超时的等待，其他线程 `await()` 无限等——到期时 leader 取走元素，在 `finally` 里 `signal()` 唤醒一个新 leader。**「定时等待」的职责被收敛到一个人身上。**

> **注意 `first = null`**：这是为了在 leader 等待期间不持有 `first` 的强引用，否则元素无法被 GC（如果元素很大，会造成内存滞留）。这种「显式置 null 帮助 GC」在 JDK 源码里很常见。

### 7.4.5 LinkedTransferQueue：`xfer` 的四模式与「松弛」

```java
// xfer 的四个模式
private static final int NOW   = 0;  // 非阻塞，立即尝试（poll / tryTransfer 的立即版）
private static final int ASYNC = 1;  // 异步，入队即返回，不等待（add/offer/put）
private static final int SYNC  = 2;  // 同步，必须等到被消费（transfer）
private static final int TIMED = 3;  // 超时等待（poll(timeout) / tryTransfer(timeout)）

// 自旋与阻塞的阈值（按 CPU 数动态计算）
private static final int FRONT_SPINS   = 1 << 7;
private static final int CHAINED_SPINS = FRONT_SPINS >>> 1;
```

`xfer` 的核心思路：

```text
1. 检查 head/tail 是否「松弛」到位（对应 NOW/ASYNC 只需一次 CAS 即可完成）
2. 找「第一个未匹配的节点」：
   - 如果是同向节点（都是生产者 or 都是消费者）→ 我的节点挂到它后面
   - 如果是反向节点 → 尝试匹配（CAS 对方 item / 唤醒对方）
3. 匹配失败 → 根据 mode 决定：
   NOW   → 返回 null（不阻塞）
   ASYNC → 只要入队成功就返回（不等消费）
   SYNC  → 自旋若干次（FRONT_SPINS）后 park，直到被匹配或超时
   TIMED → 用 parkNanos，超时后清理自己的节点（可能需要 sweep）
```

**最强的特性：`transfer()` 的「零缓冲交接」既有阻塞语义、又不丢吞吐**，因此它被称为「SynchronousQueue 的升级版」（既能交接、又能缓冲）。`LinkedTransferQueue` 是**唯一同时支持「容量 0 语义」和「无界缓冲」**的队列。

**代价**：实现复杂度极高（`sweep` 清理、`dead node` 标记、松弛 tail 的 `slack` 机制），源码 2000+ 行，**面试只要讲清四种模式的区别和适用场景即可**。

| 模式 | 对应 API | 语义 |
|---|---|---|
| NOW | `poll()`、`tryTransfer(e)` | 立即返回，成功/失败 |
| ASYNC | `add/offer/put` | 入队即成功（不会等消费） |
| SYNC | `transfer(e)` | 必须等消费者取走 |
| TIMED | `poll(timeout)`、`tryTransfer(e, timeout)` | 等一段时间 |

### 7.4.6 阻塞队列选型决策树

```text
需要「容量为 0、直接交接」？
├─ 是 → SynchronousQueue（线程池 cached 场景）或 LinkedTransferQueue（需要 transfer 语义）
└─ 否 → 需要优先级？
        ├─ 是 → 需要延迟？ ─ 是 → DelayQueue
        │                └ 否 → PriorityBlockingQueue
        └─ 否 → 需要双端操作？
                ├─ 是 → LinkedBlockingDeque
                └─ 否 → 有界且元素数量固定可预估？
                        ├─ 是 → ArrayBlockingQueue（内存连续、无节点分配、GC 友好）
                        └─ 否 → LinkedBlockingQueue（吞吐高，注意必须设容量！）
```

| 队列 | 吞吐 | 内存 | 风险 |
|---|---|---|---|
| `ArrayBlockingQueue` | 中（单锁） | 预分配，稳定 | 容量固定，满了就阻塞 |
| `LinkedBlockingQueue` | 高（双锁） | 每元素一个 Node | **默认无界 → OOM** |
| `SynchronousQueue` | 最高（无缓冲） | 极小 | 无缓冲，生产者易被阻塞 |
| `LinkedTransferQueue` | 高 | 中 | 复杂、难调试 |
| `PriorityBlockingQueue` | 中 | 增长 | **无界 → OOM** |
| `DelayQueue` | 中 | 增长 | **无界 → OOM** |

---

# 第 8 章 线程池：ThreadPoolExecutor 全链路

## 8.1 `ctl`：一个 AtomicInteger 装下「状态 + 线程数」

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static final int COUNT_BITS = Integer.SIZE - 3;          // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;     // 536870911 ≈ 5 亿

// 5 种状态（高 3 位）
private static final int RUNNING    = -1 << COUNT_BITS;  // 111 000...0
private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 000 000...0
private static final int STOP       =  1 << COUNT_BITS;  // 001 000...0
private static final int TIDYING    =  2 << COUNT_BITS;  // 010 000...0
private static final int TERMINATED =  3 << COUNT_BITS;  // 011 000...0

private static int runStateOf(int c)     { return c & ~CAPACITY; }   // 高 3 位
private static int workerCountOf(int c)  { return c & CAPACITY; }     // 低 29 位
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

```text
32 位 ctl：
┌─────┬───────────────────────────────────────┐
│ 3bit│              29bit                     │
│状态  │            workerCount                  │
└─────┴───────────────────────────────────────┘
RUNNING=111 → 负数，所以 ctl < 0 恒表示"还在运行"
```

**为什么用一个 AtomicInteger 而不是两个 volatile 字段？** 因为「状态」和「线程数」必须**一起原子变更**：`addWorker` 必须在「状态仍是 RUNNING（或允许的状态）」的前提下增加线程数，否则会出现「状态已 SHUTDOWN 但又加了一个线程」的窗口。用 CAS 对 `ctl` 做一次操作，天然满足。

### 8.1.1 五种状态的流转（图必须会画）

```text
        ┌──────────┐  shutdown()   ┌──────────┐  队列空 & workerCount=0   ┌────────┐  terminated()   ┌────────────┐
        │ RUNNING  │ ────────────▶ │ SHUTDOWN │ ────────────────────────▶ │TIDYING │ ──────────────▶ │TERMINATED  │
        └──────────┘               └──────────┘                            └────────┘                  └────────────┘
             │                          │
             │ shutdownNow()            │ shutdownNow()
             └──────────┬───────────────┘
                        ▼
                   ┌──────────┐   workerCount=0
                   │   STOP   │ ───────────────▶ TIDYING → TERMINATED
                   └──────────┘
```

| 状态 | 含义 | 是否接受新任务 | 是否处理队列任务 | 是否中断线程 |
|---|---|---|---|---|
| `RUNNING` | 正常运行 | ✅ | ✅ | ❌ |
| `SHUTDOWN` | 不接受新任务，但**继续处理已入队的任务** | ❌ | ✅ | ❌ |
| `STOP` | 不接受新任务、不处理队列任务、**中断所有线程** | ❌ | ❌ | ✅ |
| `TIDYING` | 所有任务已终止，`workerCount == 0`，即将执行 `terminated()` | ❌ | ❌ | — |
| `TERMINATED` | `terminated()` 已执行完毕 | ❌ | ❌ | — |

## 8.2 `execute`：核心三步 + 二次检查

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();

    // ① workerCount < corePoolSize → 直接加一个核心线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) return;
        c = ctl.get();                                  // 加失败（并发/状态变化）→ 重读，走下一步
    }

    // ② 核心满了 → 尝试入队
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 【关键】入队后的二次检查
        if (!isRunning(recheck) && remove(command))
            reject(command);                            // 池已关闭 → 回滚入队 + 拒绝
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);                     // 池里已无线程（core=0 或全超时）→ 补一个
    }
    // ③ 入队失败（队列满）→ 尝试扩容到最大线程数
    else if (!addWorker(command, false))
        reject(command);
}
```

### 8.2.1 「线程池执行任务的顺序」标准答案 vs 真相

**标准答案**：core → queue → max → reject。

**真相（面试加分点）**：这个顺序有个著名的副作用——

> 当 core=5、queue=1000、max=50 时，**只有队列满了才会扩到 50**。也就是说，「队列没用满之前，线程数永远是 5」。这导致：**任务响应时间变长（排队），但线程不会被充分利用**。

这就是为什么很多团队用 `ThreadPoolExecutor` 时会**手动 hack**：把 `corePoolSize == maximumPoolSize`（不使用扩容路径），或者用 `SynchronousQueue` 配合大 max（`CachedThreadPool` 思路），或者干脆自定义 `Queue`（如 `LinkedBlockingQueue` 的容量设成很小）。美团/阿里的线程池规范都强调「**队列必须设容量**」。

### 8.2.2 `addWorker`：为什么 Worker 要继承 AQS

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // ① 状态检查：SHUTDOWN 且（有 firstTask 或队列空）→ 不允许加
        if (rs >= SHUTDOWN &&
            !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                return false;                          // ② 线程数上限
            if (compareAndIncrementWorkerCount(c))     // ③ CAS 加计数
                break retry;
            c = ctl.get();
            if (runStateOf(c) != rs) continue retry;   //    状态变了 → 重来
        }
    }

    boolean workerStarted = false, workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);                     // ④ 创建 Worker
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();                            // ⑤ 全局锁保护 workers 集合
            try {
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) throw new IllegalThreadStateException();
                    workers.add(w);
                    workerAdded = true;
                    int s = workers.size();
                    if (s > largestPoolSize) largestPoolSize = s;
                }
            } finally { mainLock.unlock(); }
            if (workerAdded) { t.start(); workerStarted = true; }   // ⑥ 启动线程
        }
    } finally {
        if (!workerStarted) addWorkerFailed(w);         // ⑦ 失败回滚：从集合移除 + 计数减一 + tryTerminate
    }
    return workerStarted;
}
```

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1);        // 【关键 1】初始 state = -1，禁止在 runWorker 之前被中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
    public void run() { runWorker(this); }

    protected boolean isHeldExclusively() { return getState() == 1; }
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) { setExclusiveOwnerThread(currentThread()); return true; }
        return false;
    }
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null); setState(0); return true;   // 【关键 2】无条件置 0，不可重入
    }
    public void lock()   { acquire(1); }
    public void unlock() { release(1); }
}
```

**`Worker` 继承 AQS 的两个关键设计**：

| 设计 | 目的 |
|---|---|
| `setState(-1)` 初始为 -1 | 因为 `tryAcquire` 只在 `state==0` 时成功，初始为 -1 时**任何 `tryLock()` 都失败**，从而防止 `shutdown()` 在 `runWorker` 启动前就中断它（`interruptIfStarted` 检查 `getState() >= 0`） |
| `tryRelease` 无条件置 0 | **不可重入**。如果可重入，任务内部再 `lock()` 一次，`isHeldExclusively` 就失真，`shutdown` 会误判线程「空闲」而中断正在执行任务的线程 |

**`shutdown()` 只中断空闲线程的实现**：

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {   // 【核心】tryLock 成功 == 线程当前没有在执行任务
                try { t.interrupt(); }
                catch (SecurityException ignore) { }
                finally { w.unlock(); }
            }
            if (onlyOne) break;
        }
    } finally { mainLock.unlock(); }
}
```

> **一句话**：`runWorker` 在**执行任务期间持有 `w.lock()`**，所以 `tryLock()` 成功 ⟺ 该 Worker 空闲（在队列上 `take/poll` 阻塞）。`shutdownNow()` 则用 `interruptWorkers()`，**不管有没有锁**，全部中断。

## 8.3 `runWorker`：任务执行与中断响应

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();                              // ① 允许中断（把 state 从 -1 恢复到 0）
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {   // ② 循环取任务
            w.lock();                                          // ③ 加锁，标记"正在执行"
            // ④ 【关键】如果池已 STOP，或（当前线程中断且池已 STOP）→ 确保中断标志置位
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
                && !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);       // ⑤ 钩子（可用来埋点、设置 MDC、ThreadLocal）
                Throwable thrown = null;
                try { task.run(); }            // ⑥ 真正执行
                catch (RuntimeException x) { thrown = x; throw x; }
                catch (Error x) { thrown = x; throw x; }
                catch (Throwable x) { thrown = x; throw new Error(x); }
                finally { afterExecute(task, thrown); }   // ⑦ 钩子（异常也会调用！）
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);   // ⑧ 线程退出处理
    }
}
```

**`Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)` 为什么要这样写？**
这是一个**双重保险**：`shutdownNow` 可能在「线程刚取到任务、还没加锁」的瞬间发起中断，此时 `interruptIfStarted` 的中断可能被 `getTask` 里的 `poll` 消费掉（`InterruptedException` 已抛出但标志被清）。所以在执行任务前**再检查一次**，并且用 `Thread.interrupted()` 顺带清标志（避免后续 park 立即返回）。

**`processWorkerExit` 的精髓**：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) decrementWorkerCount();   // ① 异常退出才减计数（正常退出已在 getTask 减过）

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);                            // ② 从集合移除
    } finally { mainLock.unlock(); }

    tryTerminate();                                   // ③ 尝试终止（若已是 SHUTDOWN/STOP 且 workerCount==0）

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {                  // ④ 还没 STOP → 可能需要补充线程
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && !workQueue.isEmpty()) min = 1;
            if (workerCountOf(c) >= min) return;      //    线程数够 → 不用补
        }
        addWorker(null, false);                       // ⑤ 补一个线程（关键！保证任务不被饿死）
    }
}
```

> **为什么 `processWorkerExit` 里要 `addWorker(null, false)`？**
> 考虑 `corePoolSize=1`：唯一的核心线程因异常死掉，此时若队列里还有任务，就再也没有线程来取 → **任务饿死**。所以必须补一个。这也是「`ThreadPoolExecutor` 的线程数是自愈的」的实现。

## 8.4 `getTask`：超时、SHUTDOWN 判断、动态调整

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // ① SHUTDOWN 且队列空 / STOP → 减少计数并返回 null（线程退出）
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);
        // ② 是否需要"带超时"地取任务：
        //    允许核心线程超时，或 当前线程数 > corePoolSize（非核心线程）
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) return null;   // ③ 超时/超编 → 退出
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :   // ④ 非核心：带超时
                workQueue.take();                                        //    核心：无限阻塞
            if (r != null) return r;
            timedOut = true;                  // ⑤ 超时没取到 → 下一轮退出
        } catch (InterruptedException retry) {
            timedOut = false;                 //    被中断 → 重试（不是退出！）
        }
    }
}
```

**三个关键点**：

1. **`wc > maximumPoolSize` 的处理**：支持**动态调小 `maximumPoolSize`** 后，多余线程要在超时后自然退出（否则 `setMaximumPoolSize` 就失效了）。这是「线程池动态调参」功能的核心支撑。
2. **`(wc > 1 || workQueue.isEmpty())`**：只有「当前是最后一个线程且队列非空」时才不能退出，否则最后一个线程退出就没人干活了。
3. **`catch (InterruptedException) { timedOut = false; }`**：`shutdownNow` 中断线程时，`take/poll` 抛异常，此时**不立刻退出**，而是回到循环顶部重新判断状态——保证「STOP 时能正确退出，SHUTDOWN 时能继续把队列吃完」。

## 8.5 参数配置与容量估算

### 8.5.1 经验公式

```text
N_threads = N_cpu × U_cpu × (1 + W/C)

N_cpu  : CPU 核数
U_cpu  : 目标 CPU 利用率（0~1）
W/C    : 等待时间 / 计算时间 的比值
```

| 任务类型 | W/C | 线程数（8 核） |
|---|---|---|
| CPU 密集（计算） | ≈ 0 | 8 ~ 9 |
| IO 密集（DB/HTTP，等待 10 倍于计算） | 10 | 88 |
| 混合 | 3 | 32 |

**但这只是起点**，真正的调参方式：

1. 用压测（JMeter / wrk / 自研）逐步加压，观察 **QPS、P99、CPU 使用率、线程池队列长度**；
2. 找到「QPS 不再上升而 P99 开始飙升」的拐点 → 那就是最优线程数；
3. 用**动态调参**（`setCorePoolSize` / `setMaximumPoolSize`）在不重启的情况下调整；
4. 配合**监控**（`getActiveCount()`、`getQueue().size()`、`getCompletedTaskCount()`）。

> **面试高频陷阱**：「线程数越多越好吗？」→ 不是。线程多了会发生：①上下文切换开销 ↑；②每线程栈内存（默认 1MB，可通过 `-Xss` 调小）；③缓存局部性变差；④锁竞争加剧；⑤GC 扫描的线程根变多。**最终是「吞吐」与「延迟」的权衡**。

### 8.5.2 `Executors` 的四个工厂方法与「阿里规约禁止使用」

| 方法 | core | max | queue | 问题 |
|---|---|---|---|---|
| `newFixedThreadPool(n)` | n | n | `LinkedBlockingQueue`（**无界**） | 任务堆积 → OOM |
| `newSingleThreadExecutor()` | 1 | 1 | `LinkedBlockingQueue`（**无界**） | 同上 |
| `newCachedThreadPool()` | 0 | `Integer.MAX_VALUE` | `SynchronousQueue` | 线程数无限 → OOM / 打爆 CPU |
| `newScheduledThreadPool(n)` | n | `Integer.MAX_VALUE` | `DelayedWorkQueue`（**无界**） | OOM |
| `newWorkStealingPool(n)` | ForkJoinPool | — | — | 适合分治任务，不适合阻塞 IO |

**《阿里巴巴 Java 开发手册》原话**：「【强制】线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。」

**正确姿势**：

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    8, 32, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),                          // 有界！
    new ThreadFactoryBuilder().setNameFormat("order-pool-%d").build(),  // 命名，便于 jstack
    new ThreadPoolExecutor.CallerRunsPolicy()                 // 或自定义：落库/降级/监控
);
pool.allowCoreThreadTimeOut(true);   // 可选：低峰期回收核心线程
```

**线程命名的价值**（实战经验）：`jstack` 里看到 `pool-1-thread-1` 根本没法定位业务；命名成 `order-pool-3` 就能直接找到对应服务。**这是排查线上问题的第一优先级**。

## 8.6 拒绝策略与「第五种策略」

| 策略 | 行为 | 适用 |
|---|---|---|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 需要让调用方感知失败 |
| `CallerRunsPolicy` | **由提交任务的线程自己执行** | 想要「自动降速」：调用方被占住，自然减少提交 → 反压 |
| `DiscardPolicy` | 静默丢弃 | 可丢的任务（日志、埋点） |
| `DiscardOldestPolicy` | 丢弃队列中最老的，再重试提交 | 只要最新的（行情推送） |

**自定义策略（实战中最常用）**：

```java
new RejectedExecutionHandler() {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        // ① 打点告警（必须！否则丢任务无感知）
        Metrics.counter("pool.rejected").increment();
        // ② 降级：落 DB / 写本地文件 / 转 MQ 重试
        // ③ 或者：抛异常让上游重试
        throw new RejectedExecutionException("pool full, queue=" + e.getQueue().size());
    }
}
```

> **`CallerRunsPolicy` 的一个隐蔽陷阱**：如果提交任务的线程本身就在这个线程池里（比如池内任务再提交子任务），会导致**线程池自我阻塞**（池内线程去执行子任务，占住自己，反而拖慢）。这种嵌套提交场景不能用 `CallerRunsPolicy`。

## 8.7 优雅关闭

```java
// 推荐姿势
pool.shutdown();                                        // ① 不接受新任务，处理完队列
if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {     // ② 等待
    pool.shutdownNow();                                 // ③ 超时则强制中断
    if (!pool.awaitTermination(60, TimeUnit.SECONDS))
        log.error("pool did not terminate");
}
```

**为什么要 `awaitTermination` 两次？**
`shutdownNow` 只是「发起中断」，被中断的任务可能还在 `finally` 里做清理（或忽略了中断），所以需要再等一次。**Spring 的 `ThreadPoolTaskExecutor` 就是这么做的**（`setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(60)`）。

**注意 `tryTerminate` 的传播机制**：

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) || runStateAtLeast(c, TIDYING) ||
            (runStateLessThan(c, STOP) && !workQueue.isEmpty())) return;  // 还有活 → 不终止
        if (workerCountOf(c) != 0) {                                  // 还有线程 → 中断一个空闲的
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        // 满足条件：CAS 到 TIDYING → 执行 terminated() → CAS 到 TERMINATED → condition 唤醒 awaitTermination
        if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
            try { terminated(); } finally { ctl.set(ctlOf(TERMINATED, 0)); }
            termination.signalAll();
        }
        return;
    }
}
```

**`interruptIdleWorkers(ONLY_ONE)` 的「链式唤醒」**：每个即将退出的线程都会调用 `tryTerminate`，若还有线程就只中断一个，被中断的那个醒来后发现该退出，退出时又调用 `tryTerminate` 再中断下一个 → **一个个收尾，直到全部退出**。这是「无锁级联收尾」的精巧设计。

## 8.8 线程池的常见故障（面试「你遇到过什么问题」的标准答案）

| 现象 | 原因 | 定位方法 | 解法 |
|---|---|---|---|
| 任务一直排队、不执行 | core 满了但队列没满 → 不扩容 | `getQueue().size()`、`getActiveCount()` | 调小队列容量 / 提高 core / 用 `SynchronousQueue` |
| 内存暴涨 OOM | 无界队列堆积 | `jmap -histo` 看 `LinkedBlockingQueue$Node` | 队列设容量 + 拒绝策略 |
| 线程数暴涨 | `CachedThreadPool` 或 max 过大 | `jstack` 数线程 | 限制 max + 有界队列 |
| 任务卡死不返回 | 任务里有阻塞 IO / 死锁 / 未设超时 | `jstack` 看线程栈 | 加超时、拆分线程池（IO/CPU 隔离） |
| ThreadLocal 串数据 | 线程复用，未 `remove()` | 逐个任务打印 | `try-finally { remove() }` |
| 上下文切换高、CPU 空转 | 线程过多 / 自旋 | `pidstat -w`、`vmstat` | 减少线程、降自旋 |
| 关闭后任务丢 | 只 `shutdown` 未 `awaitTermination` | 日志 | 优雅关闭流程 |

**线程池隔离原则（大厂实践）**：
> **不同业务、不同耗时、不同优先级的任务，必须用不同线程池。** 否则一个慢 SQL 打满池子，整个服务所有接口全部超时（「故障扩散」）。这是「线程池隔离」的核心价值，也是面试里「你们线上怎么用线程池」的标准答法。

---

## 8.9 ScheduledThreadPoolExecutor（定时任务）

### 8.9.1 结构

```java
public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor {
    // 构造时把 workQueue 换成 DelayedWorkQueue，并且 maximumPoolSize 设为 Integer.MAX_VALUE
    public ScheduledThreadPoolExecutor(int corePoolSize, ...) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), ...);
    }
}
```

**为什么 `maximumPoolSize = Integer.MAX_VALUE`？**
因为 `DelayedWorkQueue` 是**无界**的（`offer` 永远成功），`execute` 走到「入队成功」分支就不会再扩容，所以 max 设多大都无所谓。但注意：**如果任务提交速度 > 执行速度，队列会无限增长**（定时任务是「每次都重新入队」，更容易堆积）。

### 8.9.2 `DelayedWorkQueue` 的堆 + Leader

```java
static class DelayedWorkQueue extends AbstractQueue<Runnable> implements BlockingQueue<Runnable> {
    private static final int INITIAL_CAPACITY = 16;
    private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    private int size = 0;
    private Thread leader;              // ← 只有一个线程做"定时等待"
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
}
```

`take()` 的逻辑（和 `DelayQueue` 几乎一样）：

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null) available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0L) return finishPoll(first);
                first = null;
                if (leader != null) available.await();          // 别人在等 → 我无限等
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try { available.awaitNanos(delay); }        // 我只等到堆顶到期
                    finally { if (leader == thisThread) leader = null; }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null) available.signal();
        lock.unlock();
    }
}
```

**`offer` 里的「领导权交接」**：

```java
public boolean offer(Runnable x) {
    ...
    if (queue[0] == e) {          // 新任务成了堆顶（比原来更早执行）
        leader = null;            // 【关键】让出 leader，叫醒一个线程重新评估
        available.signal();
    }
}
```

> **为什么 `queue[0] == e` 时要 `leader = null`？**
> 因为原 leader 正按「旧堆顶的 delay」在 `awaitNanos`，现在来了一个更早的任务，如果不管它，新任务就会**晚执行**。所以清空 leader + signal，让等待的线程重新取堆顶、算出更短的等待时间。

### 8.9.3 三种调度方式的差异（必考）

```java
// ① 固定速率：按"计划的开始时间"排期，会追时间
scheduleAtFixedRate(task, initialDelay, period, unit)
   → 理论执行时刻：T0, T0+period, T0+2*period, ...
   → 若某次执行耗时 2*period，则后续会"追平"（可能连续执行多次）

// ② 固定延迟：按"上一次结束时间"排期，不会追
scheduleWithFixedDelay(task, initialDelay, delay, unit)
   → 下次执行 = 上次结束 + delay
   → 执行耗时不影响间隔，天然不会堆叠

// ③ 一次性
schedule(task, delay, unit)
```

```java
// ScheduledFutureTask.run 的关键
public void run() {
    if (!canRunInCurrentRunState(this)) cancel(false);   // 池已关闭 → 取消
    else if (!isPeriodic()) super.run();                 // 一次性：直接跑
    else if (super.runAndReset()) {                      // 周期：跑完且没异常
        setNextRunTime();                                //   算下次时间（fixedRate: time += period）
        reExecutePeriodic(outerTask);                    //   重新入队
    }
}
```

**关键差异**：

| | `scheduleAtFixedRate` | `scheduleWithFixedDelay` |
|---|---|---|
| 下次时间 | `time += period`（时间点累加） | `time = now + delay`（当前时间 + 延迟） |
| 耗时 > 周期 | 任务会「追赶」，可能并发度变高（但同一任务不会并发执行，只是不断堆积排队） | 不会追赶，间隔自动拉长 |
| 适用 | 心跳、指标上报（要求频率稳定） | 清理任务、扫描任务（要求不重叠） |

**两个必须知道的坑**：

1. **任务抛异常 → 周期任务自动终止，且静默！** `runAndReset` 返回 false（因为异常被 `FutureTask` 捕获存进 outcome），于是不再 `reExecutePeriodic`。**任务从此不再执行，也没有任何日志**。必须自己 `try-catch` 包住任务体。
2. **`Timer` 对比**：`Timer` 是**单线程**，一个任务抛异常会**杀掉整个 Timer**（所有任务全挂）；`ScheduledThreadPoolExecutor` 至少有多个线程，但单任务异常仍会让自己消失。

| | `Timer` | `ScheduledThreadPoolExecutor` |
|---|---|---|
| 线程数 | 1 | corePoolSize 可配 |
| 异常影响 | 整个 Timer 死掉 | 只影响该任务（该周期任务停止） |
| 时间精度 | `currentTimeMillis`（受系统时间调整影响） | `nanoTime`（单调递增） |
| 是否推荐 | ❌ 已淘汰 | ✅ |

---

## 8.10 ForkJoinPool 与工作窃取

### 8.10.1 核心思想

```text
                ┌──────────────── ForkJoinPool ────────────────┐
                │                                              │
  外部提交 ───▶  │  submission queue (偶数槽位，FIFO，全池共享)     │
                │                                              │
                │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
                │  │Worker 0 │  │Worker 1 │  │Worker 2 │  ...   │
                │  │ deque:  │  │ deque:  │  │ deque:  │        │
                │  │ [a,b,c] │  │ [d,e]   │  │ [f]     │        │
                │  │  ↑top   │  │  ↑top   │  │  ↑top   │        │
                │  │  ↓base  │  │  ↓base  │  │  ↓base  │        │
                │  └────┬────┘  └────┬────┘  └────┬────┘        │
                │       │  push/pop 自己的任务（LIFO）           │
                │       └────── 窃取别人的 base 端（FIFO）──────▶ │
                └──────────────────────────────────────────────┘
```

**两条黄金规则**：

1. **自己的任务用 LIFO**（`push`/`pop` 在 `top` 端）：刚 fork 的子任务还在缓存里，先执行它局部性最好；
2. **偷别人的任务用 FIFO**（从 `base` 端 `poll`）：偷「最老的」任务，减少与 owner 的冲突（owner 在 `top` 端操作，stealer 在 `base` 端操作，**两端分离，冲突概率最低**）。

### 8.10.2 `ForkJoinTask` 的 `status`

```java
volatile int status;    // 打包：完成状态 | SMASK
static final int DONE_MASK   = 0xf0000000;  // 高 4 位是完成状态
static final int NORMAL      = 0xf0000000;  // 正常完成（负数）
static final int CANCELLED   = 0xc0000000;
static final int EXCEPTIONAL = 0x80000000;
static final int SIGNAL      = 0x00010000;  // 需要在 join 时唤醒
static final int SMASK       = 0x0000ffff;
```

- `status` 是 **负数** ⟺ 已完成（`NORMAL/CANCELLED/EXCEPTIONAL` 的高位都是 1）；
- `SIGNAL` 位表示「有线程在 `join` 我，我完成时要唤醒它」。

### 8.10.3 `fork / join` 的真实行为

```java
// fork：把任务压入当前 worker 的 deque（或提交到 submission queue）
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);   // 自己是 worker → 压自己队列
    else
        ForkJoinPool.common.externalPush(this);           // 外部线程 → 进 submission queue
    return this;
}

// join：如果没完成，就"帮忙干活"，而不是傻等
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL) reportException(s);
    return getRawResult();
}

private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :         // ← 关键：awaitJoin 里会"帮助执行别人任务"
        externalAwaitDone();
}
```

**`awaitJoin` 的「帮助」机制（核心差异化）**：

```text
join 一个未完成的任务时：
  ① 若该任务正好在自己队列的 top → pop 出来自己执行（tryUnpush）
  ② 否则"帮助执行"：从自己队列取任务、或从别人队列窃取任务来跑
     （这样阻塞的线程不是白等，而是在干活 → CPU 利用率接近 100%）
  ③ 若实在没活可干 → 短暂自旋 + 挂起（有补偿机制）
```

### 8.10.4 `ctl` 与并行度

```java
// ForkJoinPool 的 ctl：高 16 位 active count，低 16 位 并行度+版本
volatile long ctl;
// 并行度默认 = Runtime.getRuntime().availableProcessors() - 1（commonPool）
// 可显式指定：new ForkJoinPool(parallelism)
```

### 8.10.5 适用与陷阱

| 适合 | 不适合 |
|---|---|
| 递归分治（归并排序、快排、矩阵乘） | **阻塞 IO**（会占住 worker，且 `ManagedBlocker` 补偿复杂） |
| 大数据集并行处理（`parallelStream`） | 任务粒度太小（fork/join 开销 > 收益） |
| CPU 密集、任务可均分 | 有共享状态竞争（拆了也没用） |

**必须知道的坑**：

1. **`commonPool` 是全局共享的**：`parallelStream()` 默认用它，**一个慢任务会拖累全 JVM 的并行流**。生产环境应该用自定义 `ForkJoinPool` 或在业务里改成 `CompletableFuture` + 自定义 Executor。
2. **`ManagedBlocker` 的用法**：任务要阻塞时，用 `ForkJoinPool.managedBlock(blocker)` 让池子「临时补偿一个线程」，否则并行度会因阻塞而下降。
3. **递归深度**：`fork` 太深会 OOM（每个任务都是对象），要设阈值（如 `THRESHOLD = 1000`）。
4. **`invoke()` 顺序**：`left.fork(); right.compute(); left.join();` —— **先 fork 一个，自己算另一个**，不要 `fork; fork; join; join`（那会丢失「帮助」的机会）。

### 8.10.6 `CountedCompleter`：更省内存的分治

```java
// 传统：fork 两个子任务 → 每个都是对象 + join 的同步开销
// CountedCompleter：只维护一个"未完成数"，完成时递减，到 0 触发 onCompletion
class SumTask extends CountedCompleter<Integer> {
    public void compute() {
        if (size <= THRESHOLD) { result = sum(); tryComplete(); return; }
        setPendingCount(1);
        new SumTask(this, left).fork();     // 只 fork 一个
        new SumTask(this, right).compute(); // 自己算另一个
    }
    public void onCompletion(CountedCompleter<?> caller) { ... 汇总 ... }
}
```

**优势**：不需要 `join` 的阻塞与唤醒，靠 `pendingCount` 递减触发完成；**内存占用和同步开销都更低**。`parallelStream` 内部有些场景就用它。

---

## 8.11 CompletableFuture 内部机制

### 8.11.1 数据结构

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    volatile Object result;              // 结果：null=未完成；AltResult=异常/null值；否则是值
    volatile Completion stack;           // 依赖动作的 Treiber 栈（无锁 LIFO）
}

static class AltResult {                 // 包装 null 和异常
    final Throwable ex;                  // null 表示"结果就是 null"
}
```

**为什么用 `AltResult` 包装 null？** 因为 `result == null` 被用来表示「未完成」，所以「已完成且值为 null」必须用一个哨兵对象表示（`new AltResult(null)`）。

### 8.11.2 `Completion` 的栈与 `postComplete`

```text
cf.stack ──▶ Completion C3 ──▶ Completion C2 ──▶ Completion C1 ──▶ null
             (最后注册的在栈顶，先执行 → LIFO，符合"依赖越晚注册越先触发"？)

其实 LIFO 是为了避免遍历长链：新注册的依赖在栈顶，能立刻被 postComplete 弹出执行。
```

```java
final void postComplete() {
    CompletableFuture<?> f = this;
    Completion h;
    while ((h = f.stack) != null || (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        if (f.casStack(h, t = h.next)) {       // ① 弹栈
            if (t != null) {
                if (f != this) { pushStack(h); continue; }  // 嵌套场景先压回去
                h.next = null;                 //    detach
            }
            f = (d = h.tryFire(NESTED)) == null ? this : d;  // ② 执行，返回"下一个要处理的 cf"
        }
    }
}
```

**关键设计**：每个 `Completion` 子类都有 `tryFire(mode)`，用 **CAS 抢所有权**（`result == null` 时把自己的 `cf.result` 设为已完成），保证**同一个依赖只执行一次**（多线程同时完成时只有一个能 CAS 成功）。

### 8.11.3 常见陷阱清单

| 陷阱 | 说明 | 正确做法 |
|---|---|---|
| 默认用 `ForkJoinPool.commonPool()` | 并行度 = CPU-1，阻塞任务会拖垮全局 | `xxxAsync(..., executor)` 显式传线程池 |
| `join()` 不响应中断 | `get()` 抛 `InterruptedException`，`join()` 抛 `CompletionException`（不中断） | 需中断用 `get` + 自己处理 |
| 异常被吞 | 只 `thenApply` 不 `exceptionally`，异常静默 | 始终加 `exceptionally` / `handle` / `whenComplete` |
| `thenApply` vs `thenApplyAsync` | 前者可能在**上一个完成的线程**（甚至调用者线程）执行 | 需要隔离线程时用 Async 版 |
| 嵌套 future | `thenCompose` 才是扁平化，`thenApply` 会得到 `CF<CF<T>>` | `thenCompose` |
| 忘记 `get` 超时 | 无限等待 | `orTimeout`（JDK9+）或 `get(timeout)` |
| 结果丢失 | `thenAccept` 无返回值，异常不会传播到外层 | 用 `handle` 记录 |

**`thenApply` 的执行线程到底是谁？**（高频追问）

```text
① 如果 cf 已经完成（complete() 之后才注册依赖）→ 由【调用 thenApply 的线程】直接执行
② 如果 cf 未完成 → 由【完成 cf 的那个线程】在 postComplete 里执行
③ 如果用 thenApplyAsync → 由 executor 执行
```

这意味着 **`thenApply` 里的代码可能跑在任何线程上**，如果里面有 ThreadLocal 依赖或阻塞操作，就是 bug 源头。

---

# 第 9 章 原子类与累加器

## 9.1 `AtomicInteger` 的实现

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
    private volatile int value;                       // ← 注意：字段本身是 volatile

    public final int get() { return value; }          // volatile 读

    public final int getAndIncrement() { return U.getAndAddInt(this, VALUE, 1); }
    // → 底层是 CAS 循环，或直接在支持 getAndAdd 的平台上用 LOCK XADD

    public final boolean compareAndSet(int expected, int update) {
        return U.compareAndSetInt(this, VALUE, expected, update);   // → lock cmpxchg
    }
}
```

**关键点**：

1. `value` 是 `volatile`——保证 `get()` 的可见性；
2. `objectFieldOffset` 在**类加载时计算一次**，之后 CAS 直接用偏移量（避免每次反射查找）；
3. `getAndIncrement` 在 x86 上被 intrinsify 为 `LOCK XADD`（**一条指令完成读-加-写**），比 CAS 循环快；
4. JDK9+ 用 `VarHandle` 替代 `Unsafe`（`MethodHandles.arrayElementVarHandle` 等），但底层还是同样的指令。

**`incrementAndGet` 的 CAS 循环实现（当没有 XADD 时）**：

```java
public final int incrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next)) return next;   // 失败就重试
    }
}
```

**高竞争下的问题**（一定要能讲）：

```text
N 个线程同时对同一个 AtomicInteger 做 incrementAndGet：
  每个线程都发 lock xadd → 该缓存行在 N 个核之间来回弹（cache line ping-pong）
  → 每次只有一个成功，其他重试 → 重试次数随 N 线性增长
  → 总耗时 ≈ O(N²)（N 线程 × N 次重试）

实测：8 线程下，LongAdder 的吞吐可达 AtomicLong 的 5~10 倍。
```

## 9.2 `LongAdder`：分片（Striped）思想

### 9.2.1 结构

```java
public class LongAdder extends Striped64 {
    public void add(long x) { ... }
    public void increment() { add(1L); }
    public long sum() { ... }         // 弱一致
}

// Striped64（LongAdder / LongAccumulator / DoubleAdder 的公共父类）
abstract class Striped64 extends Number {
    transient volatile Cell[] cells;      // 分片数组，长度 2 的幂
    transient volatile long base;          // 无竞争时的累加基准
    transient volatile int cellsBusy;      // 自旋锁（0=空闲，1=扩容/初始化中）

    @sun.misc.Contended
    static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) { return U.compareAndSwapLong(this, VALUE, cmp, val); }
    }
}
```

### 9.2.2 `add` 的完整流程

```java
public void add(long x) {
    Cell[] cs; long b, v; int m; Cell c;
    if ((cs = cells) != null || !casBase(b = base, b + x)) {
        // ① 如果 cells 已初始化，或 CAS base 失败（说明有竞争）→ 走分片路径
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[getProbe() & m]) == null ||                  // ② 用线程 probe 定位槽
            !(uncontended = c.cas(v = c.value, v + x))) {       // ③ 对该槽 CAS
            longAccumulate(x, null, uncontended);               // ④ 失败 → 初始化/扩容/重试
        }
    }
}

final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) { ThreadLocalRandom.current(); h = getProbe(); wasUncontended = true; }

    boolean collide = false;
    for (;;) {
        Cell[] cs; Cell c; int n; long v;
        if ((cs = cells) != null && (n = cs.length) > 0) {
            if ((c = cs[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {                            // ⑤ 空槽 → 抢 cellsBusy 建 Cell
                    Cell r = new Cell(x);
                    if (cellsBusy == 0 && casCellsBusy()) {
                        try {
                            Cell[] rs; int m; int j;
                            if ((rs = cells) != null && (m = (rs.length - 1)) >= 0 && rs[j = (m & h)] == null) {
                                rs[j] = r;
                                break;
                            }
                        } finally { cellsBusy = 0; }
                        continue;
                    }
                }
                collide = false;
            }
            else if (!wasUncontended) wasUncontended = true;      // ⑥ 上次 CAS 失败 → 换 probe 再试
            else if (c.cas(v = c.value, (fn == null) ? v + x : fn.applyAsLong(v, x))) break;
            else if (n >= NCPU || cells != cs) collide = false;   // ⑦ 到顶了/被扩容过 → 不再扩
            else if (!collide) collide = true;                    // ⑧ 第一次冲突 → 标记，下一轮扩容
            else if (cellsBusy == 0 && casCellsBusy()) {          // ⑨ 抢到锁 → 扩容（长度翻倍）
                try {
                    if (cells == cs) cells = Arrays.copyOf(cs, n << 1);
                } finally { cellsBusy = 0; }
                collide = false;
                continue;
            }
            h = advanceProbe(h);                                  // ⑩ 换一个 hash（xorshift）
        }
        else if (cellsBusy == 0 && cells == cs && casCellsBusy()) {
            // ⑪ 初次初始化：长度 2，放入线程自己的 Cell
            ...
        }
        else if (casBase(v = base, (fn == null) ? v + x : fn.applyWithLong(v, x)))
            break;
    }
}
```

**设计要点总结**：

| 机制 | 作用 |
|---|---|
| `base` 快速路径 | 无竞争时只需一次 CAS（和 AtomicLong 一样快） |
| `cells` 分片 | 有竞争时把写分散到 N 个不同缓存行 |
| `getProbe()` | 每个线程一个稳定的散列值（存在 `Thread` 的 `threadLocalRandomProbe` 字段里），保证同一线程尽量固定在一个槽 |
| `@Contended` | 保证每个 Cell 独占缓存行（否则相邻 Cell 仍会伪共享，优化全废） |
| `cellsBusy` 自旋锁 | 只用 CAS 保护「初始化/扩容」，临界区极短，不用 AQS（避免对象分配） |
| `collide` 双阶段 | 第一次冲突只换 probe（CAS 可能只是运气差）；连续两次才扩容（避免过早膨胀） |
| 长度上限 `>= NCPU` | 再大也没意义（没有更多并发写者），且浪费内存 |

### 9.2.3 `sum()` 为什么是弱一致的

```java
public long sum() {
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null) sum += c.value;
    }
    return sum;
}
```

遍历过程中，其他线程可能正在改某个 `Cell`（`c.value` 是 volatile 读，能读到「某一瞬间」的值），但**整个遍历不是原子的**，所以读到的是「各 Cell 在不同时刻的值的混合」——**不是任何真实时刻的精确总和**。

> **这就是为什么 `LongAdder` 不能用于「需要精确读」的场景**（如库存扣减、金额累加）。它只适合「**只求最终求和准确，中间读不精确也无所谓**」的**统计/监控**场景（QPS 计数、TPS、命中率）。
> 需要精确读数时：`LongAccumulator` 也不行（同样弱一致），必须用 `AtomicLong` 或加锁，或者「读的时候用 `sumThenReset` 也不精确」。

### 9.2.4 `DoubleAdder` 与 `LongAccumulator`

```java
// DoubleAdder：内部还是用 long 存 double 的位，避免 double 的 CAS（Double 没有原子 CAS）
// LongAccumulator：可自定义累加函数（如求最大值、最小值）
LongAccumulator max = new LongAccumulator(Math::max, Long.MIN_VALUE);
max.accumulate(10); max.accumulate(5);
long r = max.get();     // 10
```

**`LongAccumulator` 的限制**：累加函数必须是**满足结合律与交换律**的（因为分片顺序不确定）。所以 `max/min/sum` 可以，`减法` 不行。

## 9.3 字段更新器：`AtomicIntegerFieldUpdater`

```java
public class Account {
    private volatile int balance;                          // 必须是 volatile！
    private static final AtomicIntegerFieldUpdater<Account> BALANCE =
        AtomicIntegerFieldUpdater.newUpdater(Account.class, "balance");
}
```

**八个使用约束（面试常考）**：

1. 字段必须是 `volatile`；
2. 字段不能是 `private`（**JDK9+ 允许 private，但调用方必须能访问**；老版本要求调用方有访问权限）；
3. 字段不能是 `static`（用 `AtomicInteger` 代替）；
4. 字段类型必须匹配（`int`/`long`/`Object`）；
5. 必须是**可访问的**（反射能得到）；
6. 调用者必须能访问该字段（同包或 public）；
7. 不能是 `final`；
8. 只能通过 `newUpdater` 创建（构造器私有）。

**为什么它比 `AtomicInteger` 省内存？** 因为它**不包装对象**，只在原字段上做 CAS（用 `Unsafe.objectFieldOffset` 定位偏移量）。在「百万级对象每个都需要原子计数」的场景（如统计每个用户的点击数），能省掉大量 `AtomicInteger` 对象的 16 字节头。

## 9.4 原子类家族总览

| 类 | 用途 | 底层 |
|---|---|---|
| `AtomicBoolean/Integer/Long` | 单值原子操作 | `Unsafe`/`VarHandle` CAS |
| `AtomicReference<V>` | 引用原子操作 | CAS |
| `AtomicIntegerArray/LongArray/ReferenceArray` | 数组元素原子操作 | `arrayBaseOffset + i * indexScale` 算偏移 |
| `AtomicStampedReference` | 带版本号（解决 ABA） | `AtomicReference<Pair>` |
| `AtomicMarkableReference` | 带布尔标记 | 同上 |
| `LongAdder/DoubleAdder` | 高并发求和 | Striped64 分片 |
| `LongAccumulator/DoubleAccumulator` | 高并发自定义聚合 | 同上 |
| `AtomicIntegerFieldUpdater` / `...LongFieldUpdater` / `...ReferenceFieldUpdater` | 给已有字段加原子操作 | 字段偏移量 |

## 9.5 JDK9+ 的 `VarHandle`（面试加分项）

```java
private static final VarHandle VALUE;
static {
    try { VALUE = MethodHandles.lookup().findVarHandle(MyClass.class, "value", int.class); }
    catch (ReflectiveOperationException e) { throw new ExceptionInInitializerError(e); }
}
// 使用
VALUE.getAndAdd(this, 1);
VALUE.compareAndSet(this, expected, update);
```

**与 `Unsafe` 的区别**：

| | `Unsafe` | `VarHandle` |
|---|---|---|
| 可见性 | `sun.misc` 内部 API，JDK9 起被模块系统限制 | 标准 API |
| 类型安全 | 无（偏移量是 long，类型自己保证） | **有**（编译期签名校验） |
| 内存语义 | 只能全屏障 | **支持 5 种访问模式**（plain/opaque/acquire/release/volatile） |
| 性能 | 相同（都是 intrinsic） | 相同 |

```java
// VarHandle 的五种访问模式（可以精确控制内存语义，这是它最大的价值）
get() / set()                       → plain（无屏障，可能重排）
getOpaque() / setOpaque()           → 保证不重排，但不保证立即可见
getAcquire() / setRelease()         → acquire/release（单向屏障，比 volatile 轻）
getVolatile() / setVolatile()       → 完整 volatile 语义
compareAndSet() / compareAndExchange() → 全屏障
```

> **实战价值**：`acquire/release` 语义比 `volatile` 便宜（x86 上 release-store 不需要 `mfence`），在自研高性能并发结构时能省下可观的屏障开销。这是 JDK9 之后「无锁编程」的新武器。

---

# 第 10 章 同步工具与线程基础设施

## 10.1 `CountDownLatch`：一次性门闩

```java
public class CountDownLatch {
    private static class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) { setState(count); }
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;      // ① state==0 → 成功（1），否则失败（-1）
        }
        protected boolean tryReleaseShared(int releases) {
            for (;;) {                              // ② 循环 CAS 递减
                int c = getState();
                if (c == 0) return false;           //    已经是 0 了，不再减
                int nextc = c - 1;
                if (compareAndSetState(c, nextc)) return nextc == 0;   // ③ 减到 0 才返回 true
            }
        }
    }
}
```

**底层逻辑**：

```text
await():
  acquireSharedInterruptibly(1)
    → tryAcquireShared 返回 -1（state != 0）
    → doAcquireSharedInterruptibly → 入队 → park
    → 被唤醒后重试 tryAcquireShared，直到 state == 0 → 返回

countDown():
  releaseShared(1)
    → tryReleaseShared CAS state-1
    → 若减到 0 → return true → doReleaseShared → ① CAS head.SIGNAL→0
                                              ② unparkSuccessor
                                              ③ 循环检查 head，继续传播（PROPAGATE）
```

**为什么减到 0 时要「传播」？** 因为 `await` 的线程可能有多个（都在共享队列里），`state == 0` 是「一次性对所有人生效」的条件，所以必须**把队列里所有共享节点都唤醒**——这正是共享模式 + `PROPAGATE` 存在的意义。

**关键限制**：**计数不能重置**（`state` 无法回退到初始值），所以是「一次性」的。需要重复使用 → `CyclicBarrier`。

**经典用法**：

```java
// ① 一等多：主线程等 N 个子任务完成
CountDownLatch latch = new CountDownLatch(N);
for (...) executor.submit(() -> { try { work(); } finally { latch.countDown(); } });
latch.await(30, TimeUnit.SECONDS);   // 【必须带超时！】

// ② 多等一：N 个线程等一个"发令枪"
CountDownLatch start = new CountDownLatch(1);
for (...) new Thread(() -> { start.await(); run(); }).start();
start.countDown();   // 一次性放行所有人（模拟并发压测）
```

> **坑**：`countDown` 必须放在 `finally` 里，否则任务异常会导致主线程永远 `await`。

## 10.2 `CyclicBarrier`：可循环的栅栏

```java
public class CyclicBarrier {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();   // 当前"代"

    private static class Generation {
        boolean broken;           // 是否被破坏
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try { return dowait(false, 0L); }
        catch (TimeoutException toe) { throw new Error(toe); }
    }

    private int dowait(boolean timed, long nanos) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
            if (g.broken) throw new BrokenBarrierException();       // ① 已破坏 → 直接抛

            if (Thread.interrupted()) {                            // ② 中断 → 破坏栅栏
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;                                   // ③ 到达计数
            if (index == 0) {                                      // ④ 我是最后一个！
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null) command.run();            //    执行回调
                    ranAction = true;
                    nextGeneration();                              //    换代：重置 count + signalAll
                    return 0;
                } finally {
                    if (!ranAction) breakBarrier();                //    回调抛异常 → 破坏
                }
            }

            // ⑤ 不是最后一个 → 循环等待
            for (;;) {
                try {
                    if (!timed) trip.await();
                    else if (nanos > 0L) nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && !g.broken) { breakBarrier(); throw ie; }
                    else { Thread.currentThread().interrupt(); }   // 已经换代 → 只是补中断
                }
                if (g.broken) throw new BrokenBarrierException();  // ⑥ 醒来发现被破坏
                if (g != generation) return index;                 // ⑦ 醒来发现已换代 → 成功放行
                if (timed && nanos <= 0L) { breakBarrier(); throw new TimeoutException(); }
            }
        } finally { lock.unlock(); }
    }

    private void nextGeneration() {
        trip.signalAll();          // 唤醒所有等待者
        count = parties;           // 重置计数
        generation = new Generation();   // 【关键】换一个新的 Generation 对象
    }

    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
}
```

### 10.2.1 为什么需要 `Generation`？

因为**同一个 `Condition` 的等待队列会被重复使用**。如果只靠 `count` 判断，会有一个经典问题：

```text
第 1 轮：A、B 到达，C 最后到达 → nextGeneration → signalAll → count 重置为 3
此时 D 线程刚好在「被唤醒但还没检查 count」的窗口，看到 count 已经是新一轮的值，会误以为自己属于新一轮
⇒ 必须用"代"来区分「我是第几轮的等待者」
```

`generation` 是一个**每次换代都新建的对象**，线程醒来后比较 `g != generation` 就知道「我这轮已经结束了」，从而安全返回。这是「**ABA 问题的屏障版本**」——用对象身份而不是数值来做版本判断。

### 10.2.2 `broken` 状态的语义

`broken = true` 表示**这一轮栅栏被破坏**（有线程中断/超时/回调异常），所有等待者都会收到 `BrokenBarrierException`。**栅栏可以继续使用**（`breakBarrier` 里会把 count 重置），下一轮重新开始。

```text
中断/超时/回调异常 → breakBarrier() → broken=true + count=parties + signalAll
                  → 所有等待者抛 BrokenBarrierException
                  → 下一轮 await() 时 generation 已是新的（broken=false），可以正常用
```

### 10.2.3 `CountDownLatch` vs `CyclicBarrier`（必考对照）

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| 方向 | 一个/多个线程等 N 个事件 | N 个线程互相等（都到齐才继续） |
| 可重用 | ❌ 一次性 | ✅ 多轮（自动换代） |
| 计数方式 | `countDown` 递减，谁都能减 | `await` 递减，**只有参与者能等** |
| 触发动作 | 无 | 可指定 `barrierCommand`（最后到达者执行） |
| 一方出错 | 无影响（除非不 countDown） | 全体 `BrokenBarrierException` |
| 典型用途 | 主线程等子任务、并发压测发令枪 | 分阶段计算（MapReduce 式多轮迭代） |

## 10.3 `Semaphore`：许可与限流

```java
public class Semaphore {
    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) { setState(permits); }   // state = 剩余许可
    }
    static class NonfairSync extends Sync {
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
    // Sync 里的实现
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||                         // ① 不够 → 返回负数（失败）
                compareAndSetState(available, remaining))// ② CAS 扣减
                return remaining;
        }
    }

    // 公平模式
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors()) return -1;       // ③ 有人排队 → 不许插队
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 || compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

**为什么 `remaining < 0` 时就返回（不重试）？** 因为「许可不够」是**确定失败**，重试也没用（除非有人 release，但那要靠排队等待，不能在这里死转）。所以返回负数 → AQS 入队 park。

**`release` 的细节**：

```java
public void release() { sync.releaseShared(1); }

protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) throw new Error("Maximum permit count exceeded");  // 溢出保护
        if (compareAndSetState(current, next)) return true;
    }
}
```

> **注意**：`Semaphore.release()` 可以**无限增加许可**（没有上限校验，除了 int 溢出），所以「多 release」会造成许可膨胀。这是与 `CountDownLatch` 的重要区别。

**三种典型用法**：

```java
// ① 限流（最多 N 个并发访问）
Semaphore sem = new Semaphore(10);
sem.acquire();
try { callRemote(); } finally { sem.release(); }

// ② 资源池（连接池、对象池）
// ③ 控制「同时写」的线程数（保护下游 DB）
```

**与线程池限流的区别（面试追问）**：

| | `Semaphore` | 线程池 |
|---|---|---|
| 控制对象 | **执行中的并发数**（线程可以更多，但同时干活的最多 N 个） | **线程总数** |
| 排队在哪 | AQS 队列（占用线程） | 任务队列（不占线程） |
| 适用 | 保护下游资源（DB 连接数、第三方 QPS） | 控制计算资源、隔离 |

> **`Semaphore` 的「许可证被吞」问题**：如果 `acquire` 后代码抛异常而忘记 `release`，许可就永久丢失（最终 `Semaphore` 变成 0，全部阻塞）。必须 `try-finally`。

## 10.4 `Phaser`：可重用的多阶段同步（最强也最难）

```java
// state 是一个 long，分了 4 个位域
private volatile long state;
// unarrived  : 低 16 位（还差几个人到）
// parties    : 16~31 位（总参与者数）
// phase      : 32~62 位（当前阶段号）
// tiered     : 最高位（是否分层）
```

```text
63       62            32 31        16 15         0
┌────────┬──────────────┬────────────┬────────────┐
│tiered  │   phase      │  parties   │  unarrived │
│ 1bit   │   31bit      │   16bit    │   16bit    │
└────────┴──────────────┴────────────┴────────────┘
```

**核心 API**：

```java
Phaser phaser = new Phaser(parties);   // 注册 N 个参与者
phaser.register();                     // 动态增加
phaser.arriveAndDeregister();          // 到达并注销（不参与后续阶段）
phaser.arriveAndAwaitAdvance();        // 到达并等待其他人（核心）
phaser.awaitAdvance(phase);            // 等待指定阶段完成
protected boolean onAdvance(int phase, int registeredParties) { return true; }  // 可覆盖：返回 true 则终止
```

**与 `CyclicBarrier` 的对比**：

| | `CyclicBarrier` | `Phaser` |
|---|---|---|
| 参与者数量 | 固定 | **可动态注册/注销** |
| 阶段数 | 无限（自动换代） | 无限（phase 递增） |
| 能否「部分人先走」 | ❌ 必须全员到齐 | ✅ `arriveAndDeregister` |
| 层次化 | ❌ | ✅ 支持（多个 Phaser 组成树，减少竞争） |
| 实现 | `ReentrantLock` + `Condition` | **CAS + 自旋**（无锁） |
| 复杂度 | 低 | 高（容易绕晕） |

> **适用场景**：`Phaser` 适合「参与者数量会变化的多阶段任务」（如流式处理、可变分片）。**普通场景优先用 `CyclicBarrier`**（更简单、更不容易错）。

## 10.5 `Exchanger`：两个线程交换数据

```java
Exchanger<String> exchanger = new Exchanger<>();
// 线程 A: String a = exchanger.exchange("A的数据");   // 阻塞直到 B 也来
// 线程 B: String b = exchanger.exchange("B的数据");
// 结果：A 拿到 B 的，B 拿到 A 的
```

**实现**（`Exchanger` 是 JUC 里最精巧的结构之一）：

```text
arena[]：一个 Node 数组（长度 2 的幂，最多 32 个槽）
每个线程：
  ① 计算自己的 index（基于 probe & (arenaLength-1)）
  ② 在自己的 slot 上 CAS 放一个 Node（含 item + 等待者）
  ③ 若 CAS 失败（说明已有 Node，即对方已到）→ 尝试 CAS 对方的 item 来完成交换
  ④ 若 slot 空 → 进入三级等待：
     - SPIN（自旋 100 次以内，对应"对方马上就来"）
     - YIELD（让出 CPU）
     - PARK（带随机纳秒，避免两个线程反复同时醒来）
```

**为什么不需要锁？** 因为「交换」是**两个线程的点对点操作**，用 CAS + 数组槽位就够了（每个槽最多两个线程竞争）。arena 数组是为了**减少不同线程对的冲突**（多对线程同时交换时各用各的槽）。

> **"伪共享"细节**：`Exchanger` 的 `Slot` 里专门加了 `private volatile int padded` 之类的填充字段，避免相邻 slot 落在同一缓存行（老版本 JDK 显式 pad）。

## 10.6 `LockSupport`：所有阻塞的终点

```java
public class LockSupport {
    public static void park(Object blocker) { ... }
    public static void parkNanos(Object blocker, long nanos) { ... }
    public static void unpark(Thread thread) { ... }
}
```

**底层是 `Parker`**（每个 `Thread` 对象里有一个 `parker` 字段）：

```cpp
class Parker : public os::PlatformParker {
  volatile int _counter;       // 许可（0/1）
  ...
};
```

**核心语义：许可（permit）语义，不是「先来后到」语义**：

```text
unpark(thread)：
  ① CAS 把 _counter 设为 1（不管原来是多少）
  ② 若线程正 park 着 → 唤醒它（pthread_cond_signal / futex wake）

park()：
  ① 若 _counter == 1 → CAS 设为 0，立即返回（消费许可）
  ② 否则 → 阻塞（pthread_cond_wait + mutex）
```

**三个必须记住的性质**：

| 性质 | 说明 |
|---|---|
| **许可最多累积 1 个** | 连续 `unpark` 两次后再 `park`，仍然只会立即返回一次 |
| **park/unpark 顺序无关** | 先 `unpark` 后 `park`，`park` 立即返回（不会丢许可）——这是 AQS 能安全工作的前提 |
| **park 不响应中断抛异常** | 被 `interrupt` 时 `park` **立即返回**，但**不抛异常**，需自己检查 `Thread.interrupted()` |

**与 `synchronized` 的 `wait` 对比（为什么 `park` 更强）**：

| | `Object.wait()` | `LockSupport.park()` |
|---|---|---|
| 前置条件 | 必须持有 monitor | 无（随时可调用） |
| 顺序 | 必须先 wait 再 notify（否则丢信号） | 顺序无关（有许可机制） |
| 唤醒 | `notify` 必须持有 monitor | `unpark` 无需任何锁 |
| 中断 | 抛 `InterruptedException` | 静默返回 |
| 精准唤醒 | 不能指定线程（只能 `notify`/`notifyAll`） | **可以指定线程 `unpark(t)`** |
| blocker | 无 | 有（`park(blocker)` 让 `jstack` 显示阻塞位置，排障神器） |

**`park(blocker)` 的价值**：`jstack` 输出里会打印

```text
"pool-1-thread-1" #12 waiting on condition [0x00007f...]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000007d5c8a0b8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
```

括号里的 `ReentrantLock$NonfairSync` 就是 blocker 对象。**AQS 里所有 `park(this)` 传的都是当前节点所在的同步器**，所以 `jstack` 能显示「在等哪把锁」——这是线上定位死锁的关键信息。

## 10.7 线程中断机制

**三个方法**（极易混）：

| 方法 | 作用 | 是否清除标志 |
|---|---|---|
| `t.interrupt()` | 设置中断标志 + 唤醒（park/wait/sleep/join） | — |
| `t.isInterrupted()` | 查询标志 | ❌ 不清 |
| `Thread.interrupted()` | 查询并**清除**标志 | ✅ 清 |

**中断是否能生效，取决于线程状态**：

| 线程状态 | 中断的效果 |
|---|---|
| `RUNNABLE`（跑计算） | 只置标志，**不打断**（要靠代码自己检查 `isInterrupted`） |
| `BLOCKED`（等 `synchronized`） | 只置标志，**不打断**（`synchronized` 不可中断） |
| `WAITING`（`wait`/`join`/`park`） | `wait`/`join` **抛 `InterruptedException`** 并清标志；`park` **静默返回**并保留标志 |
| `TIMED_WAITING`（`sleep`/`parkNanos`） | `sleep` 抛异常；`parkNanos` 静默返回 |
| `NEW` / `TERMINATED` | 无效果 |

> **这就是「AQS 的 `lock()` 不可中断，但 `lockInterruptibly()` 可中断」的物理原因**：前者 `park` 返回后**忽略中断标志**继续排队；后者 `park` 返回后**检查标志并抛异常**。

**标准的中断处理范式**：

```java
try {
    while (!Thread.currentThread().isInterrupted() && hasWork()) {
        doWork();   // 内部可能抛 InterruptedException
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // 【关键】恢复中断标志，让上层知道
    // 清理资源
}
```

> **为什么捕获 `InterruptedException` 后要重新 `interrupt()`？** 因为抛出异常时 JVM 已经**清除了中断标志**，如果不恢复，上层（如线程池的 `runWorker`）就无法感知「这个任务被要求中断」——会破坏「中断逐层传播」的链条。

## 10.8 `ThreadLocal`：线程私有变量

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;              // 普通 ThreadLocal
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;   // 可继承
}
```

**结构**：

```text
Thread ──▶ ThreadLocalMap
             └── Entry[] table
                   ├── Entry(key=ThreadLocal@0x123 (弱引用), value=实际值 (强引用))
                   ├── Entry(key=ThreadLocal@0x456, value=...)
                   └── ...
```

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;                    // 强引用！
    Entry(ThreadLocal<?> k, Object v) {
        super(k);                    // key 是弱引用
        value = v;
    }
}
```

**内存泄漏的完整成因（必须能画）**：

```text
Thread（线程池里长期存活）
  └─ threadLocals（强引用）
       └─ Entry[] 
            └─ Entry
                 ├─ key: WeakReference → ThreadLocal 对象
                 │        （ThreadLocal 被外部置 null 后，key 变成 null，Entry 无法通过 key 访问）
                 └─ value: 强引用 → 大对象（永不释放！）

⇒ key 被 GC 了，但 value 还被 Entry（被 Thread 强引用）抓着 → 内存泄漏
⇒ ThreadLocalMap 的 get/set/remove 时会清理 key==null 的 Entry（expungeStaleEntry），
   但如果之后再也不用这个 map，就永远不会清理
```

**为什么 key 用弱引用、value 用强引用？**
- key 弱引用：让 ThreadLocal 对象在**外部不再引用时**可以被回收（否则 ThreadLocal 自己也会泄漏）；
- value 强引用：因为 value 是业务数据，JVM 无法判断它是否还有用，只能由业务负责清理。

**正确用法**：

```java
private static final ThreadLocal<SimpleDateFormat> FMT = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

try {
    FMT.get().format(date);
} finally {
    FMT.remove();     // 【必须】线程池场景下必须 remove
}
```

**`InheritableThreadLocal` 与「线程池传递失效」**：

- `InheritableThreadLocal` 在**创建线程时**（`Thread.init`）把父线程的 `inheritableThreadLocals` **复制**给子线程。
- 但线程池的线程是**复用**的：任务提交时并不会创建新线程，所以**子线程拿到的是「创建时」的旧值**，不是「提交任务时」的父线程值。
- 解法：阿里 `TransmittableThreadLocal`（TTL）：在任务提交时**快照**，在执行前**回放**，执行后**恢复**。

**为什么 `ThreadLocal` 的 `hash` 用「黄金分割数」`0x61c88647`？**

```java
private static final int HASH_INCREMENT = 0x61c88647;
private static int nextHashCode() { return nextHashCode.getAndAdd(HASH_INCREMENT); }
```

`0x61c88647` = `(√5 - 1)/2 × 2^32`（黄金分割比 × 2³²），用它做增量能让 `hashCode & (len-1)` 的分布**极其均匀**，冲突率最低。这是 `ThreadLocalMap` 用**开放地址法（线性探测）**而不是链地址法的前提——探测时要尽量少的冲突。

**`ThreadLocalMap` 与 `HashMap` 的差异**：

| | `ThreadLocalMap` | `HashMap` |
|---|---|---|
| 冲突解决 | **开放地址法（线性探测）** | 链地址法（链表/红黑树） |
| 扩容 | **只扩不缩**，阈值 2/3 | 0.75 |
| key | **弱引用** | 强引用 |
| 清理 | `expungeStaleEntry`（惰性） | 无 |
| 迭代器 | 不支持（`ThreadLocal` 不遍历） | 支持 |

---

# 第 11 章 排障与调优

## 11.1 死锁：四种必要条件与定位

**四个必要条件**：互斥、持有并等待、不可剥夺、循环等待。破坏任意一个即可。

**Java 死锁的三种类型**：

| 类型 | 例子 | 特点 |
|---|---|---|
| **锁顺序死锁** | A 先锁 x 再锁 y，B 先锁 y 再锁 x | 最常见，靠「统一加锁顺序」解决 |
| **动态锁顺序死锁** | 转账 `transfer(a,b)` 和 `transfer(b,a)` | 靠「按 id 排序再加锁」解决 |
| **协作对象间死锁** | 两个对象互相回调（如 `Taxi` 与 `Dispatcher`） | 靠「开放调用（open call）」解决：不要在持锁时调用外部方法 |
| **资源死锁** | 线程池 A 的任务等 B，B 的任务等 A | 靠「线程池隔离 + 有界队列」解决 |

**定位手段**：

```bash
jps -l                                  # 找到进程
jstack -l <pid> > dump.txt              # 打印线程栈
jcmd <pid> Thread.print -l              # 同上（新版推荐）

# jstack 会自动检测：
# "Found one Java-level deadlock:" 段落，
# 打印出互相等待的两个线程及各自持有的锁
```

```java
// 程序内检测（生产可用）
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlocked = bean.findDeadlockedThreads();   // 返回死锁线程 ID
for (long id : deadlocked) {
    ThreadInfo info = bean.getThreadInfo(id);
    log.error("deadlock: {} lockName={}", info.getThreadName(), info.getLockName());
}
```

> **注意**：`ThreadMXBean` 只能检测 `synchronized`/`ReentrantLock` 的**对象级死锁**，检测不到「等待信号量/条件变量」的**逻辑死锁**（那要靠业务监控）。

**避免死锁的 5 条工程原则**：

1. **统一加锁顺序**（按业务主键 id 排序）；
2. **加锁超时**（`tryLock(timeout)`，超时就回滚重试）；
3. **减小锁粒度**（不要在锁里做 RPC/IO）；
4. **避免嵌套锁**（能用一把锁就别用两把）；
5. **对外开放调用**（持锁时不调用外部回调）。

## 11.2 线程 dump 的读法（线上最常用技能）

```text
"order-pool-3" #23 prio=5 os_prio=0 tid=0x... nid=0x... WAITING (parking)
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000007d5c8a0b8> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(...)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(...)
        at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(...)
        at com.demo.Service.handle(Service.java:42)          ← 业务代码位置！
        ...
```

**读法四步**：

1. 看 `nid` → 对应 `top -H -p <pid>` 的线程 ID（16 进制），定位**CPU 最高的线程**；
2. 看 `Thread.State`：`BLOCKED` = 抢 `synchronized` 失败；`WAITING (parking)` = 在 `park`（等锁/条件）；`RUNNABLE` 但栈里有 socketRead = 真在跑 IO；
3. 看 `parking to wait for` 的**对象类型** → 判断在等哪把锁；
4. 看业务栈帧 → 定位到具体代码行。

**典型症状对照**：

| 现象 | dump 特征 |
|---|---|
| 线程池打满 | 大量同名线程处于 `WAITING (parking)` 在 `getTask` → 队列空、线程空闲 |
| 死锁 | 两个线程 `BLOCKED`，互相 `waiting to lock` |
| 慢 SQL | 大量线程 `RUNNABLE`，栈停在 `SocketInputStream.read` |
| 锁竞争 | 大量 `BLOCKED` 在同一个锁的 `monitorenter` |
| 任务堆积 | 线程 `RUNNABLE` 在业务代码，队列长度持续增长 |

## 11.3 JMH：测并发的唯一正确姿势

**为什么 `System.nanoTime()` 手写测不准**？

1. **JIT 预热**：前 1 万次执行是解释执行，慢 10~100 倍；
2. **死代码消除**：JIT 发现结果没用，直接删掉整个循环；
3. **常量折叠**：循环里算常量，JIT 直接算出结果；
4. **OS 优化**：短时间的测试受调度、频率调节影响。

**JMH 的正确用法**：

```java
@BenchmarkMode(Mode.Throughput)          // 吞吐
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 5, time = 1)        // 预热 5 轮
@Measurement(iterations = 10, time = 1)  // 测量 10 轮
@Fork(3)                                 // fork 3 个 JVM（避免同一个 JVM 的偏置）
@State(Scope.Thread)                     // 每个线程一个实例
public class CounterBench {
    private AtomicLong atomic = new AtomicLong();
    private LongAdder adder = new LongAdder();

    @Benchmark public void atomicInc() { atomic.incrementAndGet(); }
    @Benchmark public void adderInc()  { adder.increment(); }
}
```

**常用参数**：`-prof gc`（看分配速率）、`-prof stack`（看热点）、`-threads 8`（多线程）。

## 11.4 常见「看起来对、其实错」的代码

| 反模式 | 问题 | 修正 |
|---|---|---|
| `if (map.get(k) == null) map.put(k, v)` | check-then-act 竞态 | `putIfAbsent` / `computeIfAbsent` |
| `volatile` 修饰 `count++` | 不原子 | `LongAdder` / `AtomicLong` |
| `synchronized` 锁 `String` 常量 | 全局共享，可能锁到别人 | 锁专用对象 |
| 锁 `Integer` / 基本类型包装类 | 缓存对象复用（-128~127），锁错对象 | 锁专用对象 |
| `Collections.synchronizedList` 后遍历不加锁 | 遍历非原子，抛 CME | 手动 `synchronized` 或 `CopyOnWriteArrayList` |
| 线程池里 `ThreadLocal` 不 remove | 内存泄漏 + 数据串号 | `finally remove()` |
| `double-checked locking` 不加 `volatile` | 半初始化对象 | 加 `volatile` |
| `ConcurrentHashMap.size()` 当精确值 | 弱一致 | 用 `mappingCount()` 也是估算；精确要加锁 |
| `ExecutorService` 用 `submit` 忽略 `Future` | 异常被吞 | `execute` 或检查 `Future.get()` |
| `shutdownNow()` 后不等待 | 任务被中断，数据不一致 | `awaitTermination` |
| 在 `synchronized` 里调用 `Thread.sleep` | 长时间持锁 | 移到锁外 |
| 用 `notify` 而不是 `notifyAll` + `while` 判断 | 丢信号 / 虚假唤醒 | `while` 循环 + 视情况 `notifyAll` |
| `ForkJoinPool` 里做阻塞 IO | 并行度被打穿 | 用 `ManagedBlocker` 或换线程池 |
| `CompletableFuture` 默认池跑阻塞任务 | 拖垮 `commonPool` | 传自定义 Executor |

---

# 第 12 章 大厂面试题库（公司 × 考核等级 × 详解 × 延伸）

## 12.0 考核等级说明

| 等级 | 标记 | 含义 | 出现频率 |
|---|---|---|---|
| **L1 必考** | 🟥 | 几乎每场必问，答不出直接挂 | 90%+ |
| **L2 高频** | 🟧 | 大概率问到，答得浅会被追打 | 60%~80% |
| **L3 中频** | 🟨 | 中等概率，常作为「追问」出现 | 30%~60% |
| **L4 加分** | 🟩 | 答出来明显区分层级（P7+ / 3-1+） | 10%~30% |
| **L5 深水区** | 🟦 | 极少问，答对说明真读过源码 | <10% |

> **面试策略**：L1 必须「零停顿、结构化」；L2 要「有细节、有数字」；L3/L4 要「能主动延伸」；L5 答不上来不要硬编，说「这块我只了解主流程」比胡扯好。

---

## 12.1 通用必考 15 题（所有大厂都会问）

### Q1 🟥 `synchronized` 的实现原理？锁升级过程讲一下（阿里/腾讯/美团/京东必问）

**标准回答框架**（按四层展开）：

1. **字节码层**：同步代码块 → `monitorenter` / `monitorexit`（异常表保证异常也解锁）；同步方法 → `ACC_SYNCHRONIZED` 标志。
2. **对象头层**：锁状态存在 Mark Word（64 位，含 hash、GC age、偏向线程 ID、锁标志位）。
3. **JVM 层（锁升级）**：
   - **无锁**：`biased_lock=0, lock=01`；
   - **偏向锁**：CAS 把 ThreadID 写入 Mark Word，之后同线程进入只比较 ThreadID（零 CAS）。有竞争时需在 **SafePoint 撤销**；批量重偏向阈值 20、批量撤销阈值 40（按 epoch 判断）；
   - **轻量级锁**：CAS 把 Mark Word 换成指向栈上 Lock Record 的指针，失败则自适应自旋，仍失败则膨胀；
   - **重量级锁**：向 VM 申请 `ObjectMonitor`，Mark Word 指向它，抢不到进 `cxq/EntryList` 并 `park`。
4. **内核层**：`ObjectMonitor::enter` 最终是 `pthread_mutex_lock` + `cond_wait`（Linux 上 `futex`）。

**加分细节**：`_cxq` 是 LIFO 头插（新来的优先），`_EntryList` 是 FIFO（防饥饿）；退出时优先 `unpark(_succ)`（指定继承人），避免无效唤醒。

**延伸**：JDK15（JEP 374）默认禁用偏向锁并废弃，因为偏向撤销需要 STW，在「大量对象被多线程轮流使用」的场景下开销大于收益。

---

### Q2 🟥 `volatile` 原理？它保证什么、不保证什么？（字节/阿里必问）

**回答要点**：

- **保证**：可见性（写后立即对其他线程可见）、有序性（禁止指令重排，通过 4 种内存屏障）。
- **不保证**：原子性（`i++` 不是原子操作）。
- **实现**：
  - 字节码：字段 `ACC_VOLATILE` 标记，看不到屏障指令；
  - JVM：`OrderAccess::storestore + store + storeload`（写）、`load + acquire`（读）；
  - x86：写是 `mov` + **`lock addl $0,(%rsp)`**（StoreLoad 屏障的廉价实现）；读是普通 `mov`（TSO 已保证不重排）；
  - ARM：写 `stlr`、读 `ldar`（acquire/release 语义）。
- **为什么 x86 读不需要屏障**：TSO 只允许 Store-Load 重排，Load-Load/Load-Store 硬件保证。

**延伸（能讲出来加分）**：volatile 的「一个写 + 一个读」等价于「半个锁」——它建立了 happens-before 边，但不排除中间状态（多变量一致性仍要锁）。

---

### Q3 🟥 CAS 是什么？底层怎么实现？有什么问题？（字节/快手/拼多多高频）

**回答**：

1. **语义**：`CAS(addr, expected, new)` 读改写三步不可分割。
2. **底层**：x86 = `lock cmpxchg`（`lock` 前缀使缓存行独占 + 相当于 StoreLoad 屏障）；ARM = `ldaxr/stlxr` 循环（LL/SC）。
3. **三个问题**：
   - **ABA**：→ `AtomicStampedReference`（版本号）、`AtomicMarkableReference`（标记位）；
   - **自旋开销**：高竞争下 N 个线程反复重试，`cache line ping-pong`，总开销 O(N²) → `LongAdder` 分片、或直接阻塞（AQS 的策略）；
   - **只能保证单个变量**：多变量原子性 → 打包成对象 CAS 或加锁。
4. **CAS 与锁的选择**：低竞争用 CAS（无系统调用），高竞争用锁（避免空转）。

---

### Q4 🟥 AQS 的原理？`state` 和队列怎么工作？（阿里 P6+ 必问）

**四句话结构**：

1. **组成**：`volatile int state` + CLH 变体双向队列 + `LockSupport.park/unpark`；
2. **模板方法**：子类实现 `tryAcquire/tryRelease`（独占）或 `tryAcquireShared/tryReleaseShared`（共享），AQS 负责排队、阻塞、唤醒、中断、超时；
3. **独占流程**：`tryAcquire` 失败 → `addWaiter` 入队 → `acquireQueued` 循环（`p == head` 才 `tryAcquire`）→ `shouldParkAfterFailedAcquire`（把前驱设为 SIGNAL）→ `parkAndCheckInterrupt`；
4. **释放流程**：`tryRelease` 成功 → `unparkSuccessor(head)`（`head.waitStatus != 0` 才唤醒，且**从 tail 反向扫描**找有效后继）。

**加分**：Node 的 4 个 `waitStatus`（CANCELLED=1、SIGNAL=-1、CONDITION=-2、PROPAGATE=-3）；`next` 不可靠、`prev` 可靠的原因（入队三步非原子）。

---

### Q5 🟥 AQS 的共享模式为什么要 `PROPAGATE`？（字节/阿里 P7）

见 5.6 节的推演。**答题骨架**：

1. 共享模式一次释放可能唤醒**一串**节点（`CountDownLatch` 的 `state=0`、`Semaphore` 的多个许可）；
2. 在多线程同时 `releaseShared` 与 `setHeadAndPropagate` 交错时，会出现「**最后一次释放的唤醒丢失**」；
3. Doug Lea 用 `PROPAGATE(-3)` 显式记录「已有线程在传播但还没传完」这个中间态；
4. `doReleaseShared` 里把 `ws==0` CAS 成 `PROPAGATE` 并再 `unpark` 一次；`setHeadAndPropagate` 里看到 `h.waitStatus < 0`（含 PROPAGATE）就继续传播。

---

### Q6 🟥 `ReentrantLock` 与 `synchronized` 的区别？怎么选？（全公司必问）

见 4.6 表格。**必须补充的两句**：

- 「`synchronized` 在低竞争下更快，因为偏向锁/轻量级锁几乎零成本，而 `ReentrantLock` 每次排队都要 `new Node`」；
- 「选择标准：需要**超时、可中断、公平、多 Condition、锁状态查询** → `ReentrantLock`；否则 `synchronized`（不会忘记 unlock）」。

---

### Q7 🟥 `ConcurrentHashMap` 的 `put` 流程？JDK7 和 JDK8 有什么区别？（阿里/美团/京东必问）

见 7.1.4。**必答的六个节点**：`initTable`（CAS `sizeCtl=-1`）→ 空桶 `casTabAt` → `MOVED` 则 `helpTransfer` → `synchronized(f)` 桶级锁 → 链表尾插/红黑树插入 → `addCount` 判断扩容。

**JDK7 vs JDK8 对比**：分段锁（Segment 继承 ReentrantLock，并发度固定 16，内存浪费）→ 桶级锁（Node+CAS+synchronized，并发度≈桶数，多线程协助扩容，`baseCount+CounterCell` 计数）。

**为什么 JDK8 用 `synchronized` 而不是 `ReentrantLock`**：桶级临界区极短（几个节点遍历），偏向/轻量级锁几乎无开销，且**省掉几十万个 Lock 对象的内存**。

---

### Q8 🟥 `ConcurrentHashMap` 的 `get` 为什么不需要加锁？（阿里/字节高频）

三个理由：
1. `table`、桶头 `tabAt`、`Node.val` 都是 **volatile**，读到的是最新值；
2. 节点一旦挂到链上就**不会被原地删除**（删除是 `e.next = e.next.next` 的短路，遍历已持有的引用不受影响）；
3. 遇到扩容 `ForwardingNode`（`hash = MOVED = -1`），其 `find()` 会转到 `nextTable` 继续找。

**延伸**：`size()` 是弱一致（`baseCount + ΣCounterCell`，遍历期间有新写入就不准）；迭代器是 fail-safe（不抛 CME，但不保证快照）。

---

### Q9 🟥 线程池的核心参数？任务提交后的执行流程？（全公司必问）

七个参数 + 三步流程（core → queue → max → reject）+ `ctl` 高低位设计（高 3 位状态、低 29 位 workerCount）。

**必须补充的两个「真相」**：
1. 「只有队列满了才会扩到 max」→ 这导致「队列没用满时线程数永远等于 core」，响应变慢；
2. `processWorkerExit` 里会 `addWorker(null, false)` 补线程，避免「最后一个核心线程异常死亡导致任务饿死」。

---

### Q10 🟥 线程池的状态有哪些？`shutdown` 和 `shutdownNow` 的区别？（美团/京东高频）

五种状态（RUNNING/SHUTDOWN/STOP/TIDYING/TERMINATED）+ 流转图。

| | `shutdown()` | `shutdownNow()` |
|---|---|---|
| 状态 | → `SHUTDOWN` | → `STOP` |
| 新任务 | 拒绝 | 拒绝 |
| 队列任务 | **继续执行完** | **丢弃并返回未执行列表** |
| 中断 | 只中断**空闲**线程（`tryLock` 成功） | 中断**所有**线程 |

**加分**：`Worker` 继承 AQS 是为了用 `tryLock` 判断「是否空闲」；`tryRelease` 无条件置 0（不可重入，防止任务内部再 lock 导致误判）。

---

### Q11 🟥 线程池的线程数怎么设？IO 密集和 CPU 密集分别多少？（全公司必问）

公式：`N = N_cpu × U_cpu × (1 + W/C)`。

- CPU 密集：`N_cpu + 1`（+1 是为了在偶尔缺页/中断时仍能占满 CPU）；
- IO 密集：`2 × N_cpu` 起步，按 `W/C` 比例放大（如 DB 查询 W/C=10 → `8 × 1 × 11 = 88`）；
- **但公式只是起点**，真正方法是压测找拐点（QPS 不再上升而 P99 飙升）。

**加分**：还要考虑「线程栈内存」（默认 1MB × 线程数）、「上下文切换」（`pidstat -w`）、「锁竞争」。

---

### Q12 🟥 `ThreadLocal` 的原理？为什么会内存泄漏？（阿里/腾讯/美团必问）

- **结构**：`Thread → ThreadLocalMap → Entry[]`，`Entry.key` 是 `ThreadLocal` 的**弱引用**，`value` 是**强引用**。
- **泄漏链路**：ThreadLocal 外部引用置 null → key 被 GC → Entry 的 key 为 null，但 value 被 `Thread`（线程池中长期存活）强引用 → 无法回收。
- **为什么 key 用弱引用**：让 ThreadLocal 对象本身能被回收（否则连 key 都泄漏）。
- **为什么 value 不能弱引用**：业务数据无法用引用可达性判断是否还有用。
- **解决**：`finally { tl.remove() }`；`ThreadLocalMap` 在 `get/set/remove` 时会 `expungeStaleEntry` 惰性清理。
- **延伸**：`InheritableThreadLocal` 在线程池里失效（复制发生在**创建线程时**），生产用阿里 `TransmittableThreadLocal`。

---

### Q13 🟥 `LongAdder` 为什么比 `AtomicLong` 快？（字节/快手/拼多多高频）

- `AtomicLong`：所有线程 CAS 同一个 `value`，高竞争下缓存行 ping-pong，重试次数 O(N)，总耗时 O(N²)。
- `LongAdder`：`base + Cell[]` 分片，线程按 `probe` 散列到不同 Cell（每个 Cell 用 `@Contended` 独占缓存行），竞争被摊平。
- 代价：`sum()` 是**弱一致**（遍历各 Cell 累加，不是原子快照），所以**不能用于精确计数**（库存、金额）。
- 细节：`cellsBusy` 自旋锁保护扩容；`collide` 双阶段（先换 probe，再扩容）；数组上限 `>= NCPU` 就不再扩。

---

### Q14 🟥 死锁怎么排查？怎么避免？（美团/京东/滴滴高频）

**排查**：`jps` → `jstack -l <pid>`（自动检测 "Found one Java-level deadlock"）→ `ThreadMXBean.findDeadlockedThreads()`；看 `BLOCKED` + `waiting to lock` 的循环依赖。

**避免**：统一加锁顺序、加锁超时（`tryLock`）、减小锁粒度、避免嵌套锁、持锁时不调用外部方法（开放调用）、资源池隔离。

---

### Q15 🟥 线程的生命周期？`BLOCKED` 和 `WAITING` 的区别？（腾讯/网易高频）

六状态：NEW / RUNNABLE / BLOCKED / WAITING / TIMED_WAITING / TERMINATED。

- `BLOCKED`：**只在等 `synchronized` 的 monitor**；
- `WAITING`：`wait()`、`join()`、`park()`（无超时）；
- `TIMED_WAITING`：`sleep(ms)`、`wait(ms)`、`parkNanos`、`join(ms)`。

**加分**：`RUNNABLE` 在 JVM 里包含「正在跑」和「跑 IO 阻塞（内核态）」两种情况（因为 JVM 不区分），所以 dump 里看到 `RUNNABLE` 但栈在 `socketRead` 是正常的。

---

## 12.2 分厂特色题库

### 12.2.1 阿里（P5~P7，偏「体系化 + 源码细节」）

| # | 题目 | 等级 | 核心考点 |
|---|---|---|---|
| A1 | AQS 的 `Node` 有哪几种状态，分别由谁设置？ | 🟥 L1 | 状态机 |
| A2 | AQS 的 `cancelAcquire` 为什么要分三种情况？ | 🟩 L4 | 链表摘除的并发安全 |
| A3 | `Condition.await` 为什么要保存 `savedState`？ | 🟧 L2 | 重入恢复 |
| A4 | `ReentrantReadWriteLock` 的 `HoldCounter` 为什么要用 ThreadLocal？ | 🟩 L4 | 读锁重入计数 |
| A5 | 读写锁的非公平模式为什么还要阻止读锁插队？ | 🟩 L4 | 写饥饿 |
| A6 | `ConcurrentHashMap` 的 `sizeCtl` 有哪些取值？扩容 `stride` 怎么算？ | 🟧 L2 | 位域设计 |

**A1 详解**：`CANCELLED(1)` 由线程自己（`cancelAcquire`）设置；`SIGNAL(-1)` 由**后继线程**在 `shouldParkAfterFailedAcquire` 里对**前驱** CAS 设置；`CONDITION(-2)` 由 `addConditionWaiter` 设置（进入条件队列时）；`PROPAGATE(-3)` 由 `doReleaseShared` 设置。初始 `0`。
**延伸追问**：「为什么 SIGNAL 挂在别人身上？」→ 委托唤醒：谁释放谁负责唤醒后继，无需额外唤醒线程；且前驱一定存在（哨兵兜底），比「记录后继」更可靠（后继可能还没接上）。

**A2 详解**：三种情况分别是（1）自己是 tail → CAS tail 到 pred，`pred.next = null`；（2）pred 是有效节点（能负责唤醒）→ CAS `pred.next = node.next` 直接跨过自己；（3）pred 是 head（自己是队首）→ 必须 `unparkSuccessor(node)` 主动唤醒后继，否则「委托唤醒链」断裂，队列永久卡死。
**为什么不能简单摘链**：因为其他线程可能正在遍历 `next`，也可能正在 CAS 修改相邻指针；所有修改都必须用 CAS + 标记（`node.next = node` 自引用）来表达「已摘除」。

**A3 详解**：`await` 会 `fullyRelease` 把 `state` 全部释放（可能是重入 3 次），必须记下原值，`acquireQueued(node, savedState)` 时恢复。这样 `await` 返回后重入次数不变，语义与 `synchronized` 的「重入 + wait」一致（但 `Object.wait` 实际上不保存重入次数，`notify` 后只重新抢一次锁）。
**延伸**：`fullyRelease` 若失败（`tryRelease` 返回 false）会抛 `IllegalMonitorStateException`；`savedState` 还用于判断「是否还有未释放的重入」。

**A4 详解**：`state` 只记录**全局读锁总数**，无法表达「本线程重入几次」。`HoldCounter` 用 `ThreadLocal` 保存每线程计数；`cachedHoldCounter` 缓存「最近一个获取读锁的线程」（读写往往成对出现在同一线程）；`firstReader/firstReaderHoldCount` 在「只有一个读线程」这个最常见场景下用字段直接记录，完全绕开 ThreadLocal 查找。
**为什么不直接存 `Thread` 引用**：`HoldCounter` 里存 `tid`（long）而不是 `Thread` 对象，避免 ThreadLocalMap 里形成「Thread → Counter → Thread」的引用（虽然不会泄漏，但会增加 GC 扫描成本）。

**A5 详解**：非公平模式下 `readerShouldBlock()` 的实现是 `apparentlyFirstQueuedIsExclusive()`——**若队首是独占（写）节点，读锁也要让路**。否则读锁是共享的、一次能进一堆，源源不断插队，写锁永远饿死。
**延伸**：这也解释了「读写锁在写多场景下性能很差」——写锁既要等所有读锁释放，又要和读锁争抢队列。

**A6 详解**：`sizeCtl` 四种值：`0`（未初始化）、`-1`（初始化中）、`-(1+n)`（扩容中，高 16 位 `resizeStamp`，低 16 位参与线程数）、`>0`（扩容阈值 = 容量 × 0.75）。
`stride = (NCPU > 1) ? (n >>> 3) / NCPU : n`，且最小 `MIN_TRANSFER_STRIDE = 16`。含义：**把 n 个桶按 CPU 数均分，每线程一次领 16 个**；桶少时不切分（单线程搬完）。

---

### 12.2.2 字节跳动（偏「JMM + 硬件 + 无锁数据结构」）

| # | 题目 | 等级 | 核心考点 |
|---|---|---|---|
| B1 | 讲一下 JMM 的 happens-before 规则，举一个「重排导致 bug」的例子 | 🟥 L1 | JMM |
| B2 | 内存屏障有哪几种？x86 上 volatile 写对应什么指令？ | 🟧 L2 | 屏障落地 |
| B3 | 为什么要避免伪共享？Java 里怎么解决？ | 🟩 L4 | 缓存行 |
| B4 | `ConcurrentLinkedQueue` 为什么 `tail` 要「滞后」？ | 🟦 L5 | 无锁队列 |
| B5 | 无锁队列的 ABA 问题怎么解决？ | 🟧 L2 | ABA |
| B6 | `Disruptor` 为什么比 `ArrayBlockingQueue` 快？ | 🟩 L4 | 无锁 + 填充 |

**B1 详解**：八条规则（程序次序、监视器锁、volatile、start、join、interrupt、finalize、传递）。例子用 DCL：`instance = new Singleton()` 三步（分配内存 → 初始化 → 赋值），2 和 3 可重排，导致其他线程看到「非 null 但字段为默认值」的对象。加 `volatile` 后写前 `StoreStore`、写后 `StoreLoad`，读后 `LoadLoad`。
**延伸**：`final` 字段语义（JSR-133）——构造函数内写 final 后自动 `StoreStore`，读侧 `LoadLoad`，所以「正确构造的不可变对象」可安全发布。

**B2 详解**：LoadLoad / StoreStore / LoadStore / StoreLoad。StoreLoad 最强（清 Store Buffer），x86 用 `mfence` 或 `lock addl $0,(%rsp)`。
**追问：「为什么 volatile 读在 x86 上不需要屏障？」** → x86 是 TSO，只允许 Store-Load 重排，Load-Load/Load-Store 由硬件保证，所以读侧一条 `mov` 就够；但**换到 ARM 就必须 `ldar`**，所以「本地 x86 测不出问题」是经典的误判来源。

**B3 详解**：两个变量落在同一 64B 缓存行，两个核分别写 → 缓存行在核间反复失效（RFO），性能退化数倍。
解法：`@sun.misc.Contended`（JVM 前后各补 64B，用户类需 `-XX:-RestrictContended`）、手工 pad 字段/数组、把数据分散到不同槽（`LongAdder.Cell[]`）、对象填充到 64B 倍数。
**延伸**：`Thread` 类里 `threadLocalRandomProbe` 等字段就用 `@Contended` 分组，避免 false sharing 影响 `ThreadLocalRandom`。

**B4 详解**：`offer` 的快速路径只需「CAS 把新节点挂到 `p.next`」，如果每次都同时 `CAS tail`，就变成两次 CAS（成本翻倍）。滞后策略：**只有 `p != t`（tail 确实落后）时才更新 tail**，把「维护 tail 精确」降级为「维护 tail 大致正确」。
**安全性**：遍历时若 `p.next == null` 就认为到了队尾；若 `p == q`（自链接）说明节点已出队，需重新定位。`tail` 落后不影响正确性，只影响「下一次入队要多走一步」。

**B5 详解**：ABA 在 Treiber 栈里会导致「丢节点/悬挂」。`ConcurrentLinkedQueue` 的应对：**出队不真正删除节点，只把 `item` CAS 成 null**（逻辑删除），节点对象不复用（靠 GC），所以「引用唯一性」天然避免了「A 变 B 又变回 A」——队列里不会出现「同一个节点被重新放回」。
**通用解法**：`AtomicStampedReference`（版本号）、`AtomicMarkableReference`（标记位）、延迟回收（RCU/epoch）。

**B6 详解**：`Disruptor` 用**环形数组 + 序号（sequence）+ 内存屏障（`Unsafe`/`VarHandle`）**，无锁、无节点分配、可批量消费；并用 **cache line padding**（每个 `Sequence` 前后填充）避免伪共享。`ArrayBlockingQueue` 用一把 `ReentrantLock` + 两个 `Condition`，每次 put/take 都有锁竞争与线程唤醒。实测 `Disruptor` 吞吐可达 ABQ 的 3~10 倍。

---

### 12.2.3 腾讯（偏「synchronized 深挖 + 对象头」）

| # | 题目 | 等级 | 核心考点 |
|---|---|---|---|
| C1 | 对象头里 Mark Word 存了什么？调用 `hashCode()` 对锁有什么影响？ | 🟩 L4 | 位域冲突 |
| C2 | 偏向锁什么时候撤销？批量重偏向/批量撤销的阈值？ | 🟧 L2 | 偏向锁 |
| C3 | 轻量级锁的 Lock Record 里存什么？重入时怎么表示？ | 🟨 L3 | 轻量级锁 |
| C4 | 锁消除需要什么前提？举一个被消除的例子 | 🟨 L3 | 逃逸分析 |

**C1 详解**：Mark Word 复用同一块位域：无锁时存 `hashCode(31) + age(4) + biased_lock(1) + lock(2)`；偏向时存 `ThreadID(54) + epoch(2) + ...`。**两者冲突**——一旦调用 `Object.hashCode()`（或 `System.identityHashCode`），偏向锁无法建立（ThreadID 位被覆盖）；对象膨胀为重量级锁后，`hashCode` 被搬到 `ObjectMonitor._header` 保存，解锁时恢复。
**加分**：`-XX:hashCode=0~5` 控制 hash 生成策略（5 是自增，1 是地址相关），会影响是否与偏向冲突。

**C2 详解**：撤销发生在「另一个线程尝试获取该对象」或「调用 `wait/notify`」或「计算 `hashCode`」时，且**必须在 SafePoint**（要遍历原线程栈判断是否仍在同步块）。
阈值：`BiasedLockingBulkRebiasThreshold = 20`（某类撤销达 20 次 → 该类所有对象的 epoch +1，允许直接改偏向）；`BiasedLockingBulkRevokeThreshold = 40`（达 40 次 → 整个类禁用偏向）。
**延伸**：`-XX:BiasedLockingStartupDelay=4000`（默认延迟 4 秒，因为启动阶段竞争多、偏向没意义）；JDK15 起 `-XX:-UseBiasedLocking` 成为默认。

**C3 详解**：`BasicObjectLock`（Lock Record）里有 `_displaced_header`（原 Mark Word 备份）和 `_obj`（指向对象）。**重入时新 Lock Record 的 `_displaced_header = NULL`**，作为「这是重入」的标记；解锁时看到 NULL 就只 pop 不减计数（真正的计数隐含在栈帧层数里）。

**C4 详解**：前提是**逃逸分析**判定对象不逃逸（`-XX:+DoEscapeAnalysis`，默认开），且 `-XX:+EliminateLocks`（默认开）。例：方法内 `new StringBuffer()` 做拼接，其 `append` 里的 `synchronized` 会被 C2 的 `EliminateLockNodes` 阶段删掉。
**延伸**：这也是「`StringBuffer` 在局部变量场景并不比 `StringBuilder` 慢」的原因；但如果 `sb` 被 return 或存入字段（逃逸），锁就保留。

---

### 12.2.4 美团（偏「线程池实战 + 故障排查」）

| # | 题目 | 等级 | 核心考点 |
|---|---|---|---|
| M1 | 线程池为什么建议用有界队列？无界队列会怎样？ | 🟥 L1 | OOM |
| M2 | 线程池的 `CallerRunsPolicy` 有什么副作用？ | 🟩 L4 | 反压 |
| M3 | 怎么实现线程池的动态调参？调小 `max` 后多余线程怎么退出？ | 🟧 L2 | `getTask` |
| M4 | 定时任务的 `scheduleAtFixedRate` 和 `scheduleWithFixedDelay` 区别？任务抛异常会怎样？ | 🟧 L2 | 调度语义 |
| M5 | 线上接口超时，怀疑线程池打满，你怎么排查？ | 🟧 L2 | 排障 |

**M1 详解**：`LinkedBlockingQueue` 默认容量 `Integer.MAX_VALUE`，生产 > 消费时任务无限堆积 → `Node` 对象撑爆堆 → `OutOfMemoryError: Java heap space`。而且**队列用满之前不会扩到 max**，导致「线程数固定、响应时间线性上升」。所以必须 `new LinkedBlockingQueue<>(1000)` + 拒绝策略。

**M2 详解**：`CallerRunsPolicy` 让**提交任务的线程**自己执行任务，好处是「自动反压」（提交方被占住，自然降低提交速率）；副作用是：
① 如果提交方是 Web 容器线程（Tomcat），会**占用请求处理线程**，导致整体 RT 上升；
② 如果池内任务再向同池提交子任务，会**自我阻塞**（池内线程执行子任务，队列永远不满，形成活锁式拖慢）。

**M3 详解**：`setCorePoolSize` / `setMaximumPoolSize` 是线程安全的（内部用 `mainLock` + `ctl` CAS）。
调**大**：会立即 `addWorker` 补线程；
调**小**：不会立刻杀线程，而是靠 `getTask` 里的 `timed = allowCoreThreadTimeOut || wc > corePoolSize` 判断——多余线程在 `keepAliveTime` 超时后自然退出。`maximumPoolSize` 调小时，`getTask` 里 `wc > maximumPoolSize` 的条件会让多余线程退出。
**加分**：`allowCoreThreadTimeOut(true)` 能让核心线程也超时回收（低峰省资源），但会带来「高峰期线程反复创建」的开销。

**M4 详解**：`fixedRate` 按「计划时间点」累加（`time += period`），执行慢会「追平」（连续执行多次但不并发）；`fixedDelay` 按「上次结束 + delay」算，不会堆积。
**抛异常**：`ScheduledFutureTask.run` 里 `runAndReset` 返回 false（异常被 `FutureTask` 捕获存进 `outcome`），于是**不再 `reExecutePeriodic`** → 该周期任务**永久停止，且无任何日志**。必须自己 `try-catch`。
**延伸**：`Timer` 是单线程，一个任务抛异常会杀掉整个 `Timer`（所有任务全挂）；且 `Timer` 用 `currentTimeMillis`，受系统时钟调整影响。

**M5 详解**（标准排查路径）：
1. `jstack -l <pid>` → 数同名线程（如 `order-pool-*`），看状态分布；
2. 大量线程 `WAITING (parking)` 在 `getTask` → 队列空、线程空闲（不是打满）；
3. 大量 `RUNNABLE` 停在业务代码 → 任务慢（去看下游依赖）；
4. 大量 `BLOCKED` 在同一把锁 → 锁竞争；
5. 配合监控：`getActiveCount()`、`getQueue().size()`、`getCompletedTaskCount()`，以及 `RejectedExecutionException` 的计数；
6. 看队列长度是否持续增长 → 消费能力不足；
7. 结论 + 处置：临时扩 core/max、或降级（拒绝策略改落库/限流）。

---

### 12.2.5 京东 / 网易 / 快手 / 拼多多 / 滴滴 / 华为

| 公司 | 题目 | 等级 | 要点 |
|---|---|---|---|
| 京东 | `CountDownLatch` 和 `CyclicBarrier` 的区别？各自底层？ | 🟧 L2 | 共享 AQS vs 锁+Condition |
| 京东 | `Semaphore` 怎么实现限流？和线程池限流有什么区别？ | 🟨 L3 | 并发数 vs 线程数 |
| 网易 | `ThreadLocal` 的 `hash` 为什么用 `0x61c88647`？ | 🟦 L5 | 黄金分割 + 开放地址 |
| 快手 | `LongAdder` 的 `sum()` 为什么不精确？什么场景能用？ | 🟧 L2 | 弱一致 |
| 快手 | `CompletableFuture` 的 `thenApply` 在哪个线程执行？ | 🟩 L4 | 完成者线程 |
| 拼多多 | `ThreadPoolExecutor` 里 `Worker` 为什么要继承 AQS？ | 🟩 L4 | 空闲判定 |
| 滴滴 | `AQS` 的 `tryAcquire` 为什么不是 `abstract`？ | 🟨 L3 | 模板方法多策略 |
| 华为 | `StampedLock` 的乐观读原理？为什么不可重入？ | 🟩 L4 | 版本号校验 |

**京东-1 详解**：`CountDownLatch` = AQS 共享模式（`state` = 剩余计数，`tryAcquireShared` 返回 `state==0?1:-1`），一个/多个线程等 N 个事件，**一次性**；`CyclicBarrier` = `ReentrantLock` + `Condition` + `Generation`，N 个线程互相等齐，**可循环**（换代重置），且可指定「最后到达者执行的 `barrierCommand`」。
**关键差异**：`CountDownLatch.countDown` 谁都能调；`CyclicBarrier.await` 只有参与者能调，且**一方中断/超时会破坏整轮**（`BrokenBarrierException`）。

**京东-2 详解**：`Semaphore` 控制的是「**同时执行的数量**」（线程可以更多，但同时干活的最多 N 个），排队在 AQS 队列里（**占着线程**）；线程池控制的是「**线程总数**」，排队在任务队列里（**不占线程**）。所以 `Semaphore` 适合保护**下游资源**（DB 连接、第三方 QPS），线程池适合控制**计算资源**。
**陷阱**：`acquire` 后忘记 `release` → 许可永久丢失；`release` 可无限增加（无上限校验）。

**网易 详解**：`ThreadLocalMap` 用**开放地址法（线性探测）**，冲突就必须「顺延找空位」，所以 hash 的均匀性极其重要。`0x61c88647` = `(√5-1)/2 × 2³²`（黄金分割 × 2³²），用它做递增能让 `hash & (len-1)` 的分布几乎无冲突（Fibonacci hashing）。这是「为什么 `ThreadLocalMap` 不用链表」的底层原因。

**快手-1 详解**：`sum()` = `base + Σcells[i].value`，遍历过程中其他线程仍在改 Cell，读到的是**各 Cell 在不同时刻的值的混合**，不是任何真实瞬间的精确总和。
**能用的场景**：只求「最终结果准确」的**统计/监控**（QPS、TPS、命中率、日志计数）。
**不能用的场景**：库存扣减、金额累加、需要「读-改-写」原子性的逻辑。

**快手-2 详解**：三种情况：
① `cf` 已完成 → **调用 `thenApply` 的线程**直接执行（`uniApply` 里 `tryFire` 立即触发）；
② `cf` 未完成 → **完成 `cf` 的那个线程**在 `postComplete` 里执行；
③ `thenApplyAsync` → 提交到 Executor 执行。
**风险**：`thenApply` 里若有阻塞或依赖 ThreadLocal，可能跑在任意线程（包括 Tomcat 请求线程、FJ worker、甚至别的业务线程）。

**拼多多 详解**：`Worker` 不需要排队，只需要一个「**是否正在执行任务**」的可中断互斥标记：
- 执行任务期间持锁（`runWorker` 里 `w.lock()`）；
- `shutdown()` 用 `tryLock()` 判断空闲 → 只中断空闲线程（避免中断正在跑的任务）；
- `tryRelease` 无条件置 0（**不可重入**），防止任务内部再 `lock()` 导致「正在执行」被误判为空闲；
- 初始 `setState(-1)` 让 `tryAcquire` 失败，防止 `runWorker` 启动前被中断。

**滴滴 详解**：AQS 同时支持独占和共享两种模式，子类通常只用其中一种。若声明 `abstract`，`Semaphore` 就必须实现独占的 `tryAcquire`（无意义）。所以用「抛 `UnsupportedOperationException`」的模板方法，由子类按需覆盖——**多策略可选时用模板方法，单一策略时才用 abstract**。

**华为 详解**：`StampedLock` 的 `state` 是 64 位，含「写锁标记 + 读锁计数 + 版本号（epoch）」。乐观读 `tryOptimisticRead()` 只读取当前版本号（**不修改 state、不加锁**），读完用 `validate(stamp)` 检查版本号是否变过（每次写锁释放都让版本 +1）。变了就退化为悲观读锁重读。
**不可重入**：没有重入计数，同一线程再次 `readLock()` 会等自己释放 → 死锁；也因此**不能嵌套**、不能与 `Condition` 配合。

---

## 12.3 深水区 / 加分题（P7+、3-1+ 常问）

### D1 🟦 AQS 的 `unparkSuccessor` 为什么要从 `tail` 反向遍历？

因为 `next` 可能为 null（`pred.next = node` 是 CAS 之后的**普通写**，存在「node 已入队但 next 未接上」的窗口）。而 `prev` 在 CAS 之前就已赋值（volatile 写），**从 tail 反向走一定能覆盖所有节点**。
**延伸**：这也是为什么 `next` 只能做「优化提示」，不能做「正确性依赖」——所有需要遍历全部节点的地方（`unparkSuccessor`、`cancelAcquire`）都用 `prev`。

### D2 🟦 `AQS` 里 `parkAndCheckInterrupt` 为什么要清除中断标志？

若不清除，被中断过的线程调用 `park` 会**立即返回**，`for(;;)` 就变成 CPU 100% 的死循环（一直 park 一直立刻返回）。所以 AQS 的策略是：清除 → 记录 `interrupted` → 继续排队 → 最终拿到锁后在 `acquire` 里 `selfInterrupt()` 把标志补回来，做到「不破坏排队 + 不丢中断」。

### D3 🟦 `ConcurrentHashMap` 扩容时 `helpTransfer` 的线程数上限是多少？为什么？

`sizeCtl` 低 16 位记录参与线程数，上限是 `MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_SHIFT)) - 1 = 65535`。实际上限是「桶数 / stride」，因为每个线程至少领 16 个桶。判断条件 `sc == rs + MAX_RESIZERS` 只是防御性检查（正常到不了）。
**为什么限流**：线程太多会导致 CAS 竞争 `transferIndex` 的开销超过搬迁本身的收益。

### D4 🟦 `TreeBin` 为什么需要一个 `state` 读写锁？

`TreeBin` 是红黑树的**包装节点**（`hash = TREEBIN = -2`），内部有一个 `volatile int state` 做「读写锁」：
- `WRITER = 1`、`WAITER = 2`、`READER = 4`；
- 读线程（`find`）用 `CAS(state, s, s+4)` 登记读；
- 写线程（插入/删除/平衡）置 `WRITER`，等所有读者退出再改树；
- 若写时还有读者 → 设置 `WAITER` 让读者帮忙唤醒。
**为什么不用 `synchronized`**：因为 `get` 必须无锁；而红黑树的旋转会临时破坏树结构（读者可能看到不一致的树），所以需要轻量读写互斥。若结构正在变，读者退化为**遍历链表**（`TreeBin` 里同时维护了一条 `first` 链表），保证「读一定能读到」。

### D5 🟦 `SynchronousQueue` 的公平模式为什么比非公平慢？

公平模式用 `TransferQueue`（FIFO），每次 `put/take` 都要把节点挂到队尾（`QNode` 有 `next/prev`，需要维护双向链），且必须**严格按到达顺序匹配**——先来的生产者必须等先来的消费者，导致「缓存局部性差、上下文切换多」。
非公平模式用 `TransferStack`（LIFO），栈顶优先匹配：刚阻塞的线程数据还在寄存器/缓存里，匹配后几乎不用真正唤醒（`unpark` 只是置许可，对方可能还在自旋），**吞吐高 3~10 倍**。

### D6 🟦 虚拟线程（JDK19/21）与 JUC 的冲突点在哪？

| 冲突 | 原因 | 解决 |
|---|---|---|
| `synchronized` 会 **pin 住载体线程** | 虚拟线程在 `monitor` 上阻塞时无法卸载（JDK19~23） | JDK24（JEP 491）修复；此前用 `ReentrantLock` 替代 |
| `ThreadLocal` 成本高 | 虚拟线程数量可能是百万级，每个都带 `ThreadLocalMap` | 用 `ScopedValue`（JDK21 预览） |
| 线程池模型失效 | 虚拟线程不该池化（创建成本极低） | 用 `Thread.ofVirtual()` / `newVirtualThreadPerTaskExecutor` |
| `park` 友好 | `LockSupport.park` 是虚拟线程友好的（可卸载） | AQS 天然优于 `synchronized` |

**一句话总结**：虚拟线程时代，**`ReentrantLock` 优于 `synchronized`**（因为 `park` 可卸载），**`ThreadLocal` 需要被 `ScopedValue` 替代**，**`ThreadPoolExecutor` 需要被 `newVirtualThreadPerTaskExecutor` 替代**（但**限流仍需 `Semaphore`**，因为虚拟线程无限创建会打爆下游）。

### D7 🟦 为什么 `AQS` 的队列不用 `ConcurrentLinkedQueue`？

因为 AQS 需要「**节点状态可修改、可反向遍历、可精确唤醒**」：
1. `ConcurrentLinkedQueue` 是**逻辑删除**（`item=null`），AQS 需要把节点标记 `CANCELLED` 后**真正从链上摘除**（否则队列无限增长）；
2. AQS 需要**从 tail 反向找有效后继**（CLQ 没有 `prev`）；
3. AQS 的 `head` 是**哨兵且会被替换**（`setHead`），语义与 CLQ 的「head 指向第一个元素」不同；
4. AQS 的节点还要承载 `waitStatus` 与 `nextWaiter`（Condition 队列复用同一 Node 类型）。

### D8 🟦 `ForkJoinPool` 的 `ctl` 里为什么低 16 位既有「并行度」又有「版本」？

`ctl` 的低 16 位在 `addWorker`/`signalWork` 时会**递增**（当版本号用），用来检测「是否有并发修改」（类似乐观锁的版本号）。这样在 CAS 失败后能区分「是并行度不够」还是「状态已被别人改了」，避免无意义的重试。

---

## 12.4 答题模板与话术

### 12.4.1 四层展开法（万能结构）

任何「XX 的原理」题，都按这四层答，**从下往上**：

```text
① 硬件/OS 层：这条路径最终落到什么指令/系统调用？
② JVM/JDK 层：哪个类、哪个字段、哪个方法负责？状态怎么流转？
③ 数据结构层：用了什么队列/数组/位域？为什么这么设计？
④ 语义层：对外保证了什么（可见性/原子性/公平性/一致性）？代价是什么？
```

**示例（问 volatile）**：
> 「④ volatile 保证可见性和有序性，不保证原子性；② JDK 层是字段的 ACC_VOLATILE 标记，JIT 在字段访问处插 `membar` 节点；① 落到 x86 是 `mov` + `lock addl $0,(%rsp)`，ARM 是 `stlr/ldar`；④ 代价是每次写都要清 Store Buffer，比普通写贵约 10~20 倍。」

### 12.4.2 五个「能立刻加分」的表达

1. **给数字**：「偏向撤销阈值 20 / 40」「`stride` 最小 16」「`spinForTimeoutThreshold` 1000ns」「`TREEIFY_THRESHOLD` 8 / `UNTREEIFY` 6」。
2. **给反例**：「这个设计在 X 场景下会退化成 Y，所以 Z 场景不能用」。
3. **给对比**：「和 `synchronized` 比，它的优势是…，代价是…」。
4. **给演进**：「JDK7 是分段锁，JDK8 改成桶级锁，原因是…」。
5. **给边界**：「它保证 A，但不保证 B，所以 C 场景必须额外加锁」。

### 12.4.3 三个「千万不要」

| 不要 | 原因 |
|---|---|
| 不要背结论不给过程 | 面试官一追问「为什么」就露馅 |
| 不要把 `volatile` 说成「从主内存读、写主内存」 | 这是**过时且错误**的说法（真实是缓存一致性 + 屏障，没有「主内存」中转） |
| 不要硬编源码细节 | 记不清就说「主流程是…，细节我记不准」，比编造安全 |

> **关于「volatile 读写主内存」这个经典错误说法**：JMM 是**抽象模型**，所谓「主内存/工作内存」只是**概念映射**（主内存 ↔ 堆，工作内存 ↔ 寄存器/缓存）。真实硬件里没有「每次读写都过主内存」这回事——那样性能会掉 10 倍。正确表述是：**volatile 通过禁止编译器缓存 + 插入内存屏障，迫使 CPU 走 MESI 协议拿到最新值**。面试里主动纠正这一点，是很强的加分项。

---

## 附录 A：核心参数与常量速查

| 常量 | 值 | 出处 |
|---|---|---|
| `COUNT_BITS` / `CAPACITY` | 29 / 2²⁹-1 | `ThreadPoolExecutor` |
| `RUNNING/SHUTDOWN/STOP/TIDYING/TERMINATED` | -1<<29 / 0 / 1<<29 / 2<<29 / 3<<29 | 同上 |
| `MIN_TRANSFER_STRIDE` | 16 | `ConcurrentHashMap` |
| `TREEIFY_THRESHOLD` / `UNTREEIFY_THRESHOLD` | 8 / 6 | 同上 |
| `MIN_TREEIFY_CAPACITY` | 64 | 同上 |
| `MOVED / TREEBIN / RESERVED` | -1 / -2 / -3 | 同上 |
| `RESIZE_STAMP_BITS` | 16 | 同上 |
| `Node.SIGNAL / CANCELLED / CONDITION / PROPAGATE` | -1 / 1 / -2 / -3 | `AQS` |
| `spinForTimeoutThreshold` | 1000 ns | `AQS` |
| `SHARED_SHIFT` | 16 | `ReentrantReadWriteLock` |
| `BiasedLockingBulkRebiasThreshold` | 20 | HotSpot |
| `BiasedLockingBulkRevokeThreshold` | 40 | HotSpot |
| `BiasedLockingStartupDelay` | 4000 ms | HotSpot |
| `TREEIFY` 概率 | ≈ 6×10⁻⁸ | 泊松分布 |
| `HASH_INCREMENT` | 0x61C88647 | `ThreadLocal` |
| `SMASK / SIGNAL` | 0xFFFF / 0x10000 | `ForkJoinTask` |
| `FRONT_SPINS / CHAINED_SPINS` | 128 / 64 | `LinkedTransferQueue` |

## 附录 B：JVM 参数速查

```bash
-XX:+UseBiasedLocking / -XX:-UseBiasedLocking     # 偏向锁开关（JDK15 起默认关）
-XX:BiasedLockingStartupDelay=0                    # 偏向锁启动延迟（调试时设 0）
-XX:+DoEscapeAnalysis                              # 逃逸分析（默认开）
-XX:+EliminateLocks                                # 锁消除（默认开）
-XX:+EliminateAllocations                          # 标量替换（默认开）
-XX:PreBlockSpin=10                                # 老式固定自旋次数（自适应自旋下无效）
-XX:AutoBoxCacheMax=200                            # Integer 缓存上限（影响锁对象复用！）
-Xss256k                                           # 线程栈大小（线程多时调小）
-XX:-RestrictContended                             # 允许用户类使用 @Contended
```

## 附录 C：一页纸总结（面试前 5 分钟速览）

```text
① 一切同步的三原语：volatile（可见性）、CAS（原子性）、park/unpark（阻塞）。
② synchronized：对象头 Mark Word → 偏向（ThreadID）→ 轻量级（CAS 指向栈 Lock Record）
   → 重量级（ObjectMonitor：_owner/_cxq/_EntryList/_WaitSet/_succ）。
③ AQS：state + CLH 变体队列 + park。Node 四状态：SIGNAL(-1)/CANCELLED(1)/CONDITION(-2)/PROPAGATE(-3)。
   独占=tryAcquire+acquireQueued；共享=tryAcquireShared+PROPAGATE 传播。
④ CHM：CAS 空桶 + synchronized 桶头 + ForwardingNode 协助扩容 + baseCount/CounterCell。
⑤ 线程池：ctl(高3位状态+低29位线程数)；core→queue→max→reject；Worker 继承 AQS 做空闲判定。
⑥ LongAdder：base + Cell[] 分片 + @Contended，sum 弱一致。
⑦ 一切性能问题的根源：cache line（64B）、上下文切换（~5μs）、系统调用（~1μs）、CAS 重试（O(N²)）。
```

---

> **免责与使用说明**：本文档基于 JDK 8/11/17 的 HotSpot 实现与公开源码整理，其中涉及的具体指令、字段名、阈值以对应版本为准；不同 JDK 发行版（如 Zing、OpenJ9）实现可能不同。面试题库的「考核等级」为基于公开面经与常见技术栈的经验估计，仅供参考，不代表任何公司的官方标准。









