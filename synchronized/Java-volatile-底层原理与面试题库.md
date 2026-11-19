# Java `volatile` 底层原理深度笔记 + 大厂面试题库

> 版本：2026-11 · 面向 HotSpot / JDK 8 ~ JDK 21
> 阅读建议：第 1~3 章建立直觉，第 4~6 章是"底层"主干（字节码 → JIT → CPU 指令），第 7~9 章是用法、对比与延伸陷阱，第 10 章是面试题库（30 题，分四档），第 11 章是考前速查表。

---

## 0. 一句话本质与认知地图

**一句话**：`volatile` 用**编译器无法消除的内存屏障**，把"Java 层面的变量读写"变成"CPU 层面必须与其他核心/主存同步的读写"，从而同时获得**可见性（visibility）**与**有序性（ordering / 禁止重排序）**，但**不提供原子性（atomicity）**。

**三层落地面貌**（记住这张图，后面全部展开都由它派生）：

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 层次 3：Java 语义层（JMM，Java Memory Model）                             │
│   volatile 写 ≈ monitor 的 release（释放锁）                              │
│   volatile 读 ≈ monitor 的 acquire（获取锁）                              │
│   结果：happens-before 边被建立 → 可见性 + 有序性                          │
└──────────────────────────────────────────────────────────────────────────┘
                                  ↕ （JMM 把语义翻译成屏障）
┌──────────────────────────────────────────────────────────────────────────┐
│ 层次 2：JVM / JIT 层（HotSpot C2 / 解释器）                               │
│   Access API: load_volatile / store_volatile                              │
│   中间表示：Volatile 标志的 LoadNode / StoreNode + MemBar 节点（PhaseCCP）  │
│   orderAccess 落地：acquire()/release()/fence() → 平台适配代码             │
└──────────────────────────────────────────────────────────────────────────┘
                                  ↕
┌──────────────────────────────────────────────────────────────────────────┐
│ 层次 1：硬件层（CPU 指令 + 缓存一致性协议）                                │
│   x86/amd64：volatile 写 → lock addl $0x0,(%rsp)（lock 前缀 = 隐性全屏障） │
│   x86/amd64：volatile 读 → 无额外指令（TSO 已保证 LoadLoad/LoadStore）     │
│   ARM/aarch64：volatile 写 → stlr / dmb ishst，读 → ldar / dmb ishld       │
│   底层协议：MESI 缓存行状态机 + store buffer / invalidate queue 冲刷       │
└──────────────────────────────────────────────────────────────────────────┘
```

**三句话记住 `volatile` 的能力边界**：

| 能力 | 是否具备 | 原因（关键词） |
|---|---|---|
| 可见性 | ✅ | 写后立即刷出 store buffer + 失效其他核心缓存行 + 屏障禁止后续读写"提前" |
| 有序性（禁止重排） | ✅ | 四个方向的内存屏障把 volatile 读写变成"栅栏" |
| 原子性 | ❌（对 `i++`、`check-then-act` 等复合操作） | 复合操作是 read-modify-write 三步，屏障只保证"每一步前后不越界"，不保证"三步不可分割" |
| 单个变量的读 / 写原子性 | ✅（含 64 位 long/double） | 屏障 + 平台实现的单次访存原子性 |

---

## 1. 三个经典"翻车现场"：为什么需要 volatile

先看现场，再看原理。这三个例子几乎覆盖了面试中 80% 的 `volatile` 提问起点。

### 1.1 现场一：主线程改标志位，工作线程永远看不见

```java
public class VisibilityDemo {
    // 去掉 volatile 试试
    private static boolean running = true;

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            int i = 0;
            while (running) {   // ① JIT 把 running 提升到循环外，或读到寄存器后缓存
                i++;
            }
            System.out.println("exit, i = " + i);
        });
        worker.start();
        Thread.sleep(100);
        running = false;        // ② 主线程写了，但写在自己的 store buffer / 本核缓存里
        worker.join();
    }
}
```

**不加 volatile 的两种失败路径（注意是两种，不是一种）**：

1. **编译器/JIT 层**：`running` 在循环体内只读不写、且方法内没有同步点，C2 判定它可以"循环不变量外提（loop-invariant hoisting）"，甚至直接判定 `while(true)` 永不退出 → 死循环。这是**有序性/可见性被编译器破坏**。
2. **硬件层**：即使 JIT 老实每次都读，读的也是本核 L1/L2 缓存里的副本；主线程的写先落进自己的 store buffer，再异步合并进缓存行，另一个核心的副本**没有被及时置为 Invalid** → 读到旧值。

> 一句话：`volatile` 要同时堵住**编译器的优化**和**硬件的乱序/缓存**这两条路。

### 1.2 现场二：DCL 单例拿到"半成品"对象

```java
public class Singleton {
    private static volatile Singleton instance;   // 这个 volatile 是必须的
    private int a = 1;
    private Singleton() { }

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查（无锁，必须能看见最新值）
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();    // ③ 这不是一个原子动作！
                }
            }
        }
        return instance;
    }
}
```

`new Singleton()` 在字节码/机器码层面是三步：

```
① 分配内存      new  → 分配对象空间，字段默认值（a=0）
② 调用构造器    invokespecial <init> → 真正赋值 a=1
③ 写引用        putstatic instance   ← 引用指向已初始化好的对象
```

**②与③之间没有数据依赖**，所以 JIT/CPU 完全可能重排为 ①→③→②：

```
时间线（无 volatile，③ 提前）
Thread A: ① 分配内存 → ③ instance = 引用（此时 a 还是 0） → ② 初始化 a=1
Thread B: 第一次检查 instance != null → 直接 return instance → 读到 a == 0 ← 拿到半成品
```

加了 `volatile` 后，`putstatic`（volatile 写）**前面会插入 StoreStore 屏障**，把 ② 的写"压"在它前面，禁止 ③ 上浮 → 安全发布。

> DCL 是"volatile 禁止重排序"最经典的面试载体，也是唯一一个"没它必错"的经典场景。

### 1.3 现场三：`i++` 加了 volatile 依然丢更新

```java
private static volatile int count = 0;
// 10 个线程各 ++ 10000 次
```

结果通常远小于 100000。因为 `count++` 是三步：

```
read count → 计算 count+1 → write count
```

`volatile` 只保证"read 拿到的一定是最新值"、"write 一定被别人看见"，但**中间那步计算允许别的线程插进来**：

```
T1: read=100           T2: read=100
T1: calc=101           T2: calc=101
T1: write=101          T2: write=101      ← 一次更新被吃掉
```

**正确的三种写法**：

```java
// ① 加锁
synchronized (lock) { count++; }

// ② 原子类（内部就是 volatile + CAS）
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();

// ③ LongAdder（高并发分段累加，更低争用）
LongAdder adder = new LongAdder();
adder.increment();
```

> 结论：`volatile` 是"**可见性 + 有序性**"工具，不是"**互斥**"工具。凡是"读-改-写"、"先检查后执行"，都必须靠锁或 CAS。

---

## 2. JMM：volatile 生长的土壤

不理解 JMM，`volatile` 就只是一句"保证可见性"的口诀。

### 2.1 为什么会有 Java 内存模型

现代 CPU 为了填平"CPU 与内存 100~300 倍的速度差"，堆了这些优化：

| 硬件/编译器优化 | 目的 | 带来的问题 |
|---|---|---|
| 多级缓存 L1/L2/L3 | 减少访存延迟 | **缓存不一致**（一个核改了，另一个核还读旧副本） |
| Store Buffer（写缓冲） | 写不阻塞，异步合并 | 写完但别人看不见 → **可见性问题** |
| Invalidate Queue（失效队列） | 收到失效消息先 ACK 再慢慢处理 | 失效延迟 → 读到过期数据 |
| 乱序执行 / 指令级并行 | 提高 IPC | **指令重排**（内存访问顺序被打乱） |
| 编译器重排、寄存器分配、循环优化 | 减少指令数 | 程序语义被"悄悄改变" |

JMM 的定位：**不改造硬件，而是给 Java 程序员一套"最小保证集"**，让程序在任何平台上表现一致；具体平台上的"更强保证"（如 x86 的 TSO）属于白送的福利，**不能依赖**。

**关键思想：JMM 是"语言级契约"，不是"硬件说明书"。** 它规定"什么重排允许、什么不允许、什么值可见"，至于用 `mfence` 还是 `lock` 还是 `dmb`，由 JVM 实现自己决定。

### 2.2 主内存 / 工作内存：一个必须理解成"抽象"的模型

```
        ┌─────────────────────────────────────────────┐
        │            主内存 Main Memory                │  ← 所有线程共享
        │   （对应堆中对象的实例字段、静态字段、数组元素）│
        └─────────────────────────────────────────────┘
              ↑ read/load        ↑ store/write
     ┌────────┴────────┐   ┌────┴─────────┐
     │  工作内存 T1     │   │  工作内存 T2  │   ← 线程私有"抽象副本"
     │ (寄存器/L1/L2/   │   │              │
     │  store buffer)  │   │              │
     └─────────────────┘   └──────────────┘
```

**三个必须纠正的误解**：

1. **工作内存不是"真的有一块内存"**。它是 CPU 寄存器 + 各级缓存 + store buffer + 编译器优化的**统称**，是逻辑抽象。
2. **"把变量从主内存拷贝到工作内存"也不存在**。真实情况是：CPU 按需从缓存行（cache line，64B）读，写先落 store buffer。
3. **局部变量（方法内基本类型）不在这个模型里**。它们只存在于栈帧/寄存器，天然线程私有，**不需要 volatile**。

> 所以 JMM 的"主内存/工作内存"是一套**为解释可见性而造的语言**，等价映射：
> 主内存 ≈ 物理内存 + 共享缓存；工作内存 ≈ 寄存器 + 私有缓存 + 写缓冲。

### 2.3 JMM 的八种原子操作（经典教科书模型）

| 操作 | 作用对象 | 含义 |
|---|---|---|
| `lock` | 主内存变量 | 标记为"某线程独占" |
| `unlock` | 主内存变量 | 解除独占 |
| `read` | 主内存 → 工作内存 | 把变量值传输过去 |
| `load` | 工作内存 | 把 read 到的值放入工作内存副本 |
| `use` | 工作内存 → 执行引擎 | 每次使用前必须 use，取最新副本 |
| `assign` | 执行引擎 → 工作内存 | 赋值写入副本 |
| `store` | 工作内存 → 主内存 | 把副本值传输过去 |
| `write` | 主内存 | 把 store 传来的值写入主内存变量 |

**必须成对、必须按序的约束**：`read+load`、`store+write` 必须成对出现；不允许出现 `assign` 后没 `store`、`store` 后没 `write`。

**这八条规则里，`volatile` 的特殊约束是**：

- `use` 前必须 `read + load`（每次都从主内存拿最新）→ **读语义**
- `assign` 后必须 `store + write`（每次改完立刻写回）→ **写语义**
- 同一线程对同一 volatile 变量的 `use` 必须与上一次 `use` 顺序一致（有序性）

> 教科书用这八条操作讲，是为了"可读"；JMM 正式规范（JSR-133）用的是 **happens-before + 同步动作（synchronization order）**。面试里两套都能讲，但要能说清"前者是通俗模型，后者是规范"。

### 2.4 happens-before：JMM 真正的"合同条款"

**定义**：若 A happens-before B，则 A 的执行结果对 B 可见，且 A 在 B 之前（在"可见性时序"意义上，不等于物理时间先后）。

**八条核心规则**：

| 规则 | 内容 | 对 volatile 的意义 |
|---|---|---|
| 程序顺序规则 | 单线程内，前面代码 hb 后面代码 | 保证 volatile 写前后的普通写不会"越过"屏障 |
| **volatile 变量规则** | 对同一 volatile 变量的**写** hb 后续对该变量的**读** | ⭐ 可见性的直接来源 |
| 监视器锁规则 | unlock hb 后续对同一锁的 lock | 与 volatile 同构（release/acquire） |
| 传递性 | A hb B，B hb C ⇒ A hb C | 保证 volatile 能把"一批写"整体传递出去 |
| start() 规则 | 线程 start() hb 该线程内任何动作 | 线程启动前的写可见 |
| join() 规则 | 线程内任何动作 hb 其他线程从 join() 返回 | 线程结束前的写可见 |
| 中断规则 | interrupt() hb 被中断线程检测到中断 | — |
| 对象终结规则 | 构造器结束 hb finalize() 开始 | — |

**volatile 的"传递性"是它最值钱的地方**（面试高频追问点）：

```java
// writer
a = 1;                    // 普通写
b = 2;                    // 普通写
flag = true;              // volatile 写（release）

// reader
if (flag) {               // volatile 读（acquire）
    int x = a;            // 一定看到 1
    int y = b;            // 一定看到 2  ← 靠的是传递性
}
```

推导：`a=1` hb `flag=true`（程序顺序规则）；`flag=true` hb `if(flag)`（volatile 规则）；`if(flag)` hb `x=a`（程序顺序规则）；**传递** ⇒ `a=1` hb `x=a`。

> 这就是"**volatile 写之前的普通写，对读到该 volatile 值的线程都可见**"这句话的严格来源。不是"volatile 有魔力把别的变量也刷出去"，而是**屏障 + hb 传递**的联合效果。

### 2.5 重排序：三种来源，一个统一敌人

```
源代码
  │  ① 编译器重排（Javac 几乎不重排，主要发生在 JIT 的 C1/C2：
  │    指令调度、循环优化、公共子表达式消除、死代码消除、条件推断）
  ▼
字节码
  │  ② CPU 指令级并行重排（乱序执行、分支预测、SMT）
  ▼
机器指令
  │  ③ 内存系统重排（store buffer 合并、invalidate queue 延迟、
  │    缓存行预取、多核写合并）
  ▼
最终内存可见顺序 ≠ 程序顺序
```

**JMM 的对策分层**：

- 对①②：靠**编译器屏障**（`LoadLoad`/`StoreStore` 等"伪指令"），JIT 看到就放弃跨越它的优化。
- 对③：靠**CPU 内存屏障指令**（`mfence`/`lfence`/`sfence`、`dmb`、`lock` 前缀）。

**三种重排的典型例子（必须能说出来）**：

| 类型 | 例子 | 后果 |
|---|---|---|
| 编译器 | `while(flag)` 中 flag 外提 / `if(x) {a=1;b=2;}` 重排为 `b=2;a=1` | 死循环、半初始化 |
| ILP | 两个无依赖 load 交换执行顺序 | 读到"未来"的值 |
| 内存系统 | 写进 store buffer 后顺序对外不可见 | 别核看到"旧值+新值"倒挂 |

**`as-if-serial` 的边界**：单线程语义不能被破坏（这是编译器重排的底线）。但**多线程下，两个线程各自的 as-if-serial 可以"互相刺穿"**，于是出现"理论上不可能的结果"——这就是著名的 **Dekker / Peterson / StoreLoad 重排**：

```java
// 初始 x=0, y=0。两个线程各自按序执行
Thread1: x = 1; r1 = y;
Thread2: y = 1; r2 = x;
// 结果可能是 r1 == 0 && r2 == 0  ← 两个读都"没看见"对方的写
```

原因：两个 `store` 都停在各自 store buffer 里，两个 `load` 都从本核缓存读到旧值。**这是 x86 TSO 都会允许的唯一一种重排（StoreLoad）**，也是 `volatile` 需要 `lock`/`mfence` 的唯一原因。

### 2.6 四种内存屏障（后面所有讨论的"原子单位"）

| 屏障 | 约束 | 通俗解释 |
|---|---|---|
| `LoadLoad` | Load1 ; **LoadLoad** ; Load2 → Load1 先于 Load2 完成 | "后面的读别跑到前面" |
| `StoreStore` | Store1 ; **StoreStore** ; Store2 → Store1 先于 Store2 对其他核可见 | "前面的写先落地，后面的写再出门" |
| `LoadStore` | Load1 ; **LoadStore** ; Store2 → Load1 先于 Store2 完成 | "读完了再写，别倒过来" |
| `StoreLoad` | Store1 ; **StoreLoad** ; Load2 → Store1 对其他核可见后才执行 Load2 | "写完让人看见，才能读"（**最重**，常由全屏障实现） |

**记忆口诀**：

- 两个同名（LoadLoad / StoreStore）= 同类操作排序；
- 读写交叉（LoadStore / StoreLoad）= 异类操作排序；
- **`StoreLoad` 是"全能屏障"**，x86 上唯一需要显式指令的，成本最高（≈ 几十到上百 cycle）。

---

## 3. `volatile` 的三大语义：逐条拆到"为什么能成立"

### 3.1 可见性：一条完整的因果链

```
① 线程 T1 执行 volatile write
        │
        ▼
② 编译器：在 write 前插 StoreStore、write 后插 StoreLoad（x86 落地为 lock 前缀）
        │
        ▼
③ CPU：lock 前缀触发三件事
   (a) 把本核 store buffer 中的该 cache line 立即刷到 L1/共享缓存
   (b) 通过总线/目录协议向所有其他核广播 "Invalidate"
   (c) 等待其他核 ACK（并清空自己的 invalidate queue 中已收到的失效消息）
        │
        ▼
④ 其他核：对应 cache line 从 S(Shared)/E(Exclusive) 变为 I(Invalid)
        │
        ▼
⑤ 线程 T2 执行 volatile read：cache miss → 向上层/主存取最新 → 拿到 T1 的值
```

**四个关键认知（面试常考的"并非常识"的点）**：

1. **可见性不是"写直接写进主内存"**。现代 CPU 极少直写内存，`volatile` 保证的是"写到**缓存一致性协议可见的层面**"，不是"写到 DRAM"。
2. **`volatile` 读在 x86 上没有额外指令**。因为 x86 是 TSO（Total Store Order）模型，`Load` 不会越过前面的 `Load`（LoadLoad 天然满足）、`Load` 不会越过后面的 `Store`（LoadStore 由 TSO 的"读优先"保证）。所以读只需普通 `mov`，**读的开销几乎为零**（这也是"读多写少用 volatile 很划算"的硬件依据）。
3. **写贵在读便宜**：`lock` 前缀 ≈ 全屏障，会让 store buffer 排空、流水线部分停顿，量级几十 cycle；读≈普通内存访问。
4. **可见性有"方向"**：`volatile` 保证"**写 → 后续读**"单向可见，不保证"读 → 之前写"（这毫无意义）。

### 3.2 有序性：JMM 规定的屏障插入策略（经典表）

JMM 在"保守策略"下要求的插入（来自 JSR-133 的 cookbook，HotSpot 会做减法）：

| 位置 | 屏障 | 作用 |
|---|---|---|
| 每个 volatile **写**之前 | `StoreStore` | 禁止上面的普通写与 volatile 写重排（DCL 靠它！） |
| 每个 volatile **写**之后 | `StoreLoad` | 禁止 volatile 写与下面的 volatile/普通读写重排 |
| 每个 volatile **读**之后 | `LoadLoad` | 禁止上面的 volatile 读与下面的普通读重排 |
| 每个 volatile **读**之后 | `LoadStore` | 禁止上面的 volatile 读与下面的普通写重排 |

**为什么"写前"是 StoreStore、"读后"是 LoadLoad —— 一图说清**：

```
        volatile 写（release 语义）              volatile 读（acquire 语义）
        ────────────────────────                ────────────────────────
   普通读 ─┐                                    volatile 读 ─┐
   普通写 ─┤ ← StoreStore 拦截（不许上浮）                    ├→ LoadLoad  ← 普通读
   ────────┘                                    ───────────┘
   volatile 写                                  LoadStore  ← 普通写
   ────────┐
           └→ StoreLoad  ← 后面的一切（不许下沉）
```

形式化表述（背下来，面试直接说）：

> **volatile 写 = release**：它之前的所有读写，都不能被重排到它之后。
> **volatile 读 = acquire**：它之后的所有读写，都不能被重排到它之前。
> 两条合起来 ⇒ 一对 volatile 写/读构成 **release-acquire 配对**，中间夹着的所有操作对读方全部可见。

### 3.3 不保证原子性：拆到字节码与 CPU 原子指令两个层面

**Java 层**：`count++` 编译为 4 条字节码：

```
getstatic  count      // 读
iconst_1
iadd                  // 算
putstatic  count      // 写
```

`volatile` 只让 `getstatic`/`putstatic` 各自变"贵"，**中间的 `iadd` 与两者之间可以被别的线程插入**。

**CPU 层**：即使 JIT 把三条合成一条 `inc [mem]`，x86 的 `inc` 对内存是 **read-modify-write**，**不是**单个原子指令；只有 **`lock inc`** 或 `lock xadd` 才原子。JVM 不会为 `count++` 生成 `lock inc`（因为 Java 规范没要求复合操作原子）。

**真正原子的是这些（单次访存级）**：

| 操作 | 原子性 |
|---|---|
| 引用赋值、`int`/`boolean`/`byte`/`char`/`short` 读写 | ✅ 天然 |
| `long`/`double` 的**非 volatile** 读写 | ⚠️ JLS 允许"拆成两个 32 位"（word tearing），HotSpot 实际多数平台原子，但**不能当规范依赖** |
| `long`/`double` 的 **volatile** 读写 | ✅ 规范保证原子（JLS §17.7） |
| `i++`、`a = a + b`、`if(x==null) x=new()` | ❌ 复合操作 |

> 面试中"volatile 保证 long 原子性吗"的标准答法：**普通 long/double 的读写在 JLS 层面不保证原子（可能撕裂）；加 volatile 后保证单次读/写原子；但 `long++` 依旧不原子。**

### 3.4 `volatile` 与 `synchronized` 的语义同构性（高频对比题的内核）

| 维度 | `volatile` | `synchronized` |
|---|---|---|
| 内存语义 | 写 = release，读 = acquire | unlock = release，lock = acquire |
| 可见性 | ✅ | ✅ |
| 有序性 | ✅（禁止重排） | ✅（临界区整体原子 + 屏障） |
| 原子性 | ❌（仅单次读写） | ✅（临界区互斥） |
| 阻塞 | 无（不挂起线程） | 有（竞争时膨胀为重量级锁，挂起） |
| 指令开销 | 写 ≈ `lock`（几十 cycle），读 ≈ 0 | 无竞争 ≈ CAS（十几 cycle）；有竞争 ≈ 系统调用级 |
| 适用 | 一写多读的状态标志、DCL、发布 | 复合操作、多变量不变式 |

**一句话总结**：`volatile` 是"轻量级的**单向**同步"，`synchronized` 是"重量级的**双向**同步"。`volatile` 能覆盖的场景极少，但覆盖的那几个（状态标志、一次性发布）用 `synchronized` 会明显变慢。

---

## 4. 第一层落地：字节码与 Class 文件里的 `volatile`

### 4.1 唯一痕迹：字段访问标志 `ACC_VOLATILE`

`volatile` **不改变指令**，只改变**字段的访问标志（access_flags）**。用 `javap -v` 看：

```java
public class V {
    private volatile int flag;
    public void set(int v) { flag = v; }
    public int get() { return flag; }
}
```

```
private volatile int flag;
  descriptor: I
  flags: (0x0042) ACC_PRIVATE, ACC_VOLATILE     ← 关键就是这一位
```

方法体字节码（**和普通字段完全一样**）：

```
public void set(int);
   0: aload_0
   1: iload_1
   2: putfield      #2   // Field flag:I     ← 指令本身没有 volatile 版本
   5: return

public int get();
   0: aload_0
   1: getfield      #2   // Field flag:I     ← 同上
   4: ireturn
```

**结论（要能一句话说清）**：

> `volatile` 在 Class 文件里只有 `ACC_VOLATILE` 这一个标志位，字节码指令与普通字段一致。**语义不靠字节码承载，靠 JVM/JIT 读到这个标志后的特殊处理承载。**

> 对比：`synchronized` 才是在字节码里留下 `monitorenter`/`monitorexit` + `ACC_SYNCHRONIZED`。这是 `volatile` 比 `synchronized` "轻"的第一层原因——**它根本没有指令级的存在感**。

### 4.2 解释器执行期的处理

解释器（C++ 实现的模板解释器）在执行 `getfield`/`putfield` 时会读字段的 `fieldDescriptor`（`FieldAccessors`），若 `is_volatile()`：

- `putfield` → 走 `store_volatile`（`Unsafe`/`Access` API），而不是普通 `store`；
- `getfield` → 走 `load_volatile`。

在 x86 上，`store_volatile` 的实现就是"**普通 mov + 一条 `lock addl`**"（详见第 6 章）。所以**即使不开 JIT，volatile 也是生效的**——这点很重要，很多人误以为 volatile 只是"JIT 的事"。

---

## 5. 第二层落地：HotSpot / JIT 如何实现 `volatile`

### 5.1 三段式访问 API：`Access` → `Unsafe` → `OrderAccess`

HotSpot 内部把字段访问抽象成 **Access API**（JDK 9 起正式化，替换原 `Unsafe` 的大部分用法）：

| Java 侧 API | HotSpot C++ 侧 | 平台语义 |
|---|---|---|
| `getfield`（普通） | `HeapAccess<>::load(...)` | plain |
| `getfield`（volatile） | `HeapAccess<>::load<volatile>(...)` (`load_volatile`) | acquire |
| `putfield`（volatile） | `store_volatile` | release |
| `VarHandle.getVolatile` | `load_volatile` | acquire |
| `VarHandle.setRelease` | `store_release`（`release_store`） | release（**不插 StoreLoad**，更便宜） |
| `VarHandle.getAcquire` | `load_acquire` | acquire |
| `VarHandle.getOpaque/setOpaque` | `load_opaque`/`store_opaque` | 仅禁止编译器优化，不保证跨核可见 |

**`OrderAccess` 的四个基本原语**（HotSpot 的平台抽象层，`src/hotspot/share/runtime/orderAccess.hpp`）：

```cpp
static void load_load_barrier();      // LoadLoad
static void store_store_barrier();    // StoreStore
static void acquire();                // 相当于 LoadLoad + LoadStore
static void release();                // 相当于 LoadStore + StoreStore
static void fence();                  // 全屏障，相当于 StoreLoad
```

### 5.2 x86-64 上 `OrderAccess` 的真实实现（关键！）

```cpp
// orderAccess_linux_x86.inline.hpp（示意）
inline void OrderAccess::load_load_barrier() { }            // 空！
inline void OrderAccess::store_store_barrier() { }          // 空！
inline void OrderAccess::acquire() { }                      // 空！
inline void OrderAccess::release() { }                      // 空！
inline void OrderAccess::fence() {
    // 全屏障 = lock 前缀 + 一条空 add
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
}
```

**为什么 5 个里有 4 个是空的？** 因为 **x86 是 TSO（Total Store Order，全存储序）模型**：

| 重排方向 | x86 是否允许 | 因此需要的屏障 |
|---|---|---|
| Load → Load（读读） | ❌ 不允许 | `LoadLoad` 不需要 → 空实现 |
| Load → Store（读写） | ❌ 不允许 | `LoadStore` 不需要 → 空实现 |
| Store → Store（写写） | ❌ 不允许（同地址；不同地址允许但由一致性保证） | `StoreStore` 不需要 → 空实现 |
| **Store → Load（写读）** | ✅ **允许** | `StoreLoad` **需要** → `lock`/`mfence` |

> **这是理解"volatile 读几乎免费、写要一条 lock"的最终答案。**
> 面试里能讲清"x86 的 TSO 让四种屏障里只有 StoreLoad 需要显式指令"，基本就是"读过源码"级别了。

### 5.3 volatile 写在 x86 上的汇编实证

```java
public class Bar {
    volatile int x;
    int y;
    public void write() {
        y = 1;      // 普通写
        x = 1;      // volatile 写
    }
}
```

`java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly` 的输出（HotSpot，示意但结构真实）：

```
movl    $0x1, 0x8(%rsi)          ; y = 1   （普通写，先落地）
lock addl $0x0, (%rsp)           ; ★ StoreStore + StoreLoad 的唯一实现：
                                 ;   lock 前缀本身既是全屏障，又是刷 store buffer 的信号
movl    $0x1, 0xc(%rsi)          ; x = 1   ← 注意：volatile 写本身也只是一条普通 mov！
```

**两个反直觉但必须掌握的结论**：

1. **volatile 的写不是特殊指令**，仍是 `mov`。真正干活的是插在它**前面**的 `lock addl $0x0,(%rsp)`。
   这条"对栈顶做 +0"的指令**没有任何业务含义**，纯粹是为了触发 `lock` 前缀 → 因为 `lock` 必须跟随一条读改写指令，`add $0` 是最便宜的选择。
2. **屏障插在 volatile 写之前**（对应 JMM 的 StoreStore 位置），但 `lock` 同时具备 `StoreLoad` 的全屏障效果，所以"写前的一条 lock"**一箭双雕**地同时满足了 StoreStore 与 StoreLoad。JDK 8 的 HotSpot 就是这么生成；JDK 9+ 有些平台会拆成 `mov` + `lock addl` 或使用 `mfence`，效果等价。

**JDK 9+ 的更精细版本（VarHandle `setRelease`）**：

```
movl  $0x1, 0x8(%rsi)      ; y = 1
movl  $0x1, 0xc(%rsi)      ; x = 1   ← 无 lock！只有编译屏障
```
因为 `release`（setRelease）**不要求 StoreLoad**（后面没有 load 需要在它之后），所以省掉了最贵的 `lock`。**这是"volatile 比 release 更贵"的实证。**

### 5.4 ARM/aarch64 上的实现（对比才有深度）

ARM 是弱内存模型（Weakly-ordered），**四种重排全部允许**，所以：

```cpp
// orderAccess_linux_aarch64.inline.hpp（示意）
inline void OrderAccess::load_load_barrier()  { __asm__("dmb ishld"); }   // LoadLoad
inline void OrderAccess::store_store_barrier(){ __asm__("dmb ishst"); }   // StoreStore
inline void OrderAccess::acquire()            { __asm__("dmb ishld"); }   // acquire
inline void OrderAccess::release()            { __asm__("dmb ish");   }   // release
inline void OrderAccess::fence()              { __asm__("dmb ish");   }   // 全屏障
```

JDK 9+ 会用更廉价的 `ldar`（load-acquire）/ `stlr`（store-release）单指令替代 `dmb` 屏障，因为 aarch64 硬件原生支持 acquire/release 语义。

**x86 vs ARM 的对照表（面试加分项）**：

| 维度 | x86-64（TSO） | ARM/aarch64（Weak） |
|---|---|---|
| volatile 读 | 普通 `mov`，**零额外开销** | `ldar` 或 `dmb ishld`，**有开销** |
| volatile 写 | `lock addl`（≈ 几十~上百 cycle） | `stlr` 或 `dmb ish`（≈ 几十 cycle） |
| 需要显式屏障的重排 | 仅 StoreLoad | 全部四种 |
| 结论 | "读免费、写贵" | "读写都贵" |

> 所以"volatile 读比写快很多"这句话**在 ARM 上不成立**。这也是为什么 JDK 在 ARM 上性能表现与 x86 不同——**JMM 只保证语义，不保证性能特征一致**。

### 5.5 JIT（C2）必须放弃的优化：`volatile` 的"编译屏障"作用

C2 在 `PhaseCCP`/`GraphKit` 中给 volatile 字段生成 `LoadNode`/`StoreNode` 时打上 `is_volatile` 标记，随后在**多个优化阶段主动跳过**：

| C2 优化 | 对普通字段 | 对 volatile 字段 |
|---|---|---|
| **循环不变量外提（hoisting）** | 可以提到循环外 | ❌ 禁止（每次迭代都必须重读） |
| **公共子表达式消除（CSE）** | 两次读可合并为一次 | ❌ 禁止（两次读可能不同） |
| **死存储消除（dead store elimination）** | 后面没读就删掉 | ❌ 禁止（必须让别人能看见） |
| **存储合并 / 寄存器缓存** | 值驻留寄存器 | ❌ 禁止（必须每次访存） |
| **指令调度跨越（scheduling across）** | 可自由移动 | ❌ 被 `MemBar*` 节点挡住 |
| **条件推断 / 分支剪枝** | `if(flag)` 可能被常量化 | ❌ 禁止（flag 可能被别的线程改） |

**"死循环"现场的技术复盘**（对应 1.1 节）：C2 的 `PhaseIdealLoop` 中有 `process_loop_invariants`，会把循环体里"不依赖循环变量、无副作用"的 `LoadI` 提到 preheader。**但若 `LoadNode` 带 volatile 标记，`can_hoist` 判定为 false**，于是每次迭代仍然 `mov` 从内存读 → 主线程一改就能看到。

**代码实证（JDK 源码语义）**：

```cpp
// hotspot/share/opto/memnode.cpp
bool LoadNode::can_remove_control() const { ... }
bool LoadNode::depends_only_on_test() const {
  if (is_volatile()) return false;          // ← 关键：volatile load 不能被优化掉
  ...
}
```

**编译屏障 vs CPU 屏障的区别（务必分清）**：

| | 编译屏障（Compiler Barrier） | CPU 屏障（Hardware Fence） |
|---|---|---|
| 作用对象 | 编译器的指令调度/优化 | CPU 的乱序执行/store buffer |
| 手段 | 特殊的"伪指令"节点、`asm volatile("" ::: "memory")` | `lock`、`mfence`、`dmb` |
| 成本 | 几乎为 0 | 几十~几百 cycle |
| 例子 | `StoreStore` 在 x86 是空实现，但**仍然禁止 C2 重排** | `fence()` 的 `lock addl` |

> **这是最容易答错的一题**："x86 上 StoreStore 屏障是空实现，那 volatile 怎么禁止重排？"——答：**空的是"硬件屏障"，编译屏障依然存在（C2 的 MemBar 节点 + volatile 标记让优化器不敢动）**。软件层的"不许重排"靠编译器，硬件层的"不许可见性乱序"靠 `lock`。

---

## 6. 第三层落地：CPU 与缓存一致性

### 6.1 `lock` 前缀到底做了什么（Intel SDM 的三条保证）

`lock` 前缀（配合 `add`/`xchg`/`cmpxchg` 等）保证：

1. **原子性**：该读改写指令在执行期间独占缓存行（或锁定总线），不被其他核打断。
2. **可见性/刷写**：把本处理器 **store buffer 中的所有待写数据**冲刷到缓存/内存（即 `lock` 隐含全屏障）。
3. **失效传播**：通过缓存一致性协议让其他核的对应副本失效（`Invalidate`），**且必须等到所有核 ACK**（不是发了就走）。

**第九条补充（很多人不知道）**：`lock` 还会阻止**后续指令的乱序执行跨越它**（相当于序列化点 serializing instruction 的弱化版），这就是它同时充当 `StoreLoad` 全屏障的原因。

**成本量级（经验值，用于回答"volatile 有多慢"）**：

| 操作 | 典型延迟 |
|---|---|
| 寄存器访问 | < 1 cycle |
| L1 命中 | ~4 cycle |
| L2 命中 | ~12 cycle |
| L3 命中 | ~30-70 cycle |
| 主内存 | ~150-300 cycle |
| `lock` 前缀（无竞争） | ~20-50 cycle 起，跨 NUMA/高竞争时可达数百 cycle |
| **volatile 写** | ≈ 普通写 + `lock` ≈ 几十 cycle |
| **volatile 读（x86）** | ≈ 普通读（0 额外） |

### 6.2 MESI：缓存行的四种状态（讲清"为什么需要协议"）

多核缓存必须保证"同一地址在不同核的副本不会各自为政"。MESI 用**状态机 + 总线嗅探（snooping）**做到：

| 状态 | 含义 | 本核可写？ | 其他核有副本？ |
|---|---|---|---|
| **M** (Modified) | 本核独占且已修改，与内存不一致 | ✅ | ❌ 无 |
| **E** (Exclusive) | 本核独占，与内存一致 | ✅（转 M） | ❌ 无 |
| **S** (Shared) | 多核共享，与内存一致 | ❌ 需先 RFO | ✅ 有 |
| **I** (Invalid) | 副本无效 | ❌ | 可能有也可能无 |

**一次 volatile 写在协议层的完整舞蹈**：

```
T1 想写变量 x（cache line 处于 S 状态，说明 T2 也有副本）
  │
  ├─① 发出 RFO (Request For Ownership) 总线事务
  ├─② T2 嗅探到，把自己的 cache line 置 I，返回 ACK
  ├─③ T1 收到全部 ACK → 该行转 M（独占）
  ├─④ T1 写入 → 但先落到 store buffer（此时还没"可见"！）
  └─⑤ volatile 写的 lock 前缀 → 强制排空 store buffer → 数据进入 M 状态的 cache line

T2 之后读 x
  │
  ├─① 发现 cache line 是 I → cache miss
  ├─② 发 BusRd → T1 嗅探到自己持有 M → 提供数据（cache-to-cache transfer）并降级为 S
  └─③ T2 拿到新值 → 从 I 转 S
```

**关键洞察**：MESI 保证的是"**读会拿到最新**"（因为它靠失效/共享），但它**不保证写立即生效**——写先落 store buffer。**`volatile` 的 `lock` 才是"把 store buffer 排空"的那一脚。** 所以：

```
缓存一致性协议  ≠  可见性
协议解决"副本不一致"，store buffer 造成"写未生效"，volatile 用 lock 一次性解决后者
```

> 面试标准答法："MESI 保证最终一致，但 store buffer 让写'迟到'，volatile 通过 lock 前缀排空 store buffer 并获得近似顺序一致的效果。"

### 6.3 store buffer 与 invalidate queue：两种"延迟"的病理

```
              ┌──────────── T1 核 ────────────┐      ┌──── T2 核 ────┐
  写 x=1 ───► │ Store Buffer (x=1 待发)        │      │               │
              │      ↓ 异步合并               │      │  L1: x=0      │
              │   L1: x=0 → x=1               │      │               │
              └───────────────────────────────┘      └───────────────┘
                          │  Invalidate(x) 广播
                          ▼
              ┌──── T2 核 ────┐
              │ Invalidate Queue: [x]  ← 收到了但"排队等待处理"
              │ L1: x=0（还没处理失效，所以仍能命中旧值！）
              └───────────────┘
```

- **store buffer 问题**：本核的后续读可以先读 store buffer（store-to-load forwarding），于是"本核看到的顺序"与"别的核看到的顺序"不同 → **StoreLoad 重排**。
- **invalidate queue 问题**：别的核收到失效消息后先入队 ACK，但 L1 里还是旧值 → **读到过期值**。

**屏障的本质就是"冲刷这两个队列"**：

- `StoreStore`/release：等 store buffer 排空（保证"我前面的写先被别人看见"）。
- `LoadLoad`/acquire：处理 invalidate queue（保证"我后面的读不会用旧副本"）。
- `StoreLoad`/fence：**两者都做**（排空 store buffer + 处理失效队列）→ 最贵。

### 6.4 伪共享（False Sharing）：volatile 的性能陷阱

**问题**：缓存行是 **64 字节**（x86）而非单个变量。若两个 volatile 变量落在同一行、被两个核各自频繁写，则每次写都会让对方的整行失效：

```
Cache Line (64B):
┌──────────────────────────────┬──────────────────────────┐
│  a (volatile, T1 写)          │  b (volatile, T2 写)      │  ← 同一 cache line！
└──────────────────────────────┴──────────────────────────┘
T1 写 a → 整行变 M → T2 的 b 副本变 I → T2 写 b 要 RFO → T1 的 a 变 I → …… ping-pong
```

实测可造成 **10 倍以上**性能下降。三种解法：

```java
// ① 手动填充（JDK 7 时代做法）
public class Padded {
    public volatile long value;
    public long p1,p2,p3,p4,p5,p6,p7;   // 填充到 64B 之外
}

// ② JDK 8+ @Contended（需 -XX:-RestrictContended）
@sun.misc.Contended
public class Padded2 { public volatile long value; }

// ③ 用 LongAdder / 线程本地累加后汇总（从根本上避免共享）
```

> 面试关联题："`LongAdder` 为什么比 `AtomicLong` 快？"——答：`AtomicLong` 的 `value` 是 volatile，所有线程 CAS 同一 cache line → 严重伪共享 + CAS 失败重试；`LongAdder` 用 `Cell[]` 数组把累加分散到不同 cache line（`@Contended` 填充），最终 `sum()` 汇总，把"写争用"转成"读汇总"。

### 6.5 一个必须澄清的"伪知识点"

网上常见说法："volatile 就是加了 `lock` 前缀，`lock` 会让缓存行失效并写回主内存，所以所有线程都从主内存读。"

**这句话有 3 处不严谨**：

1. `lock` **不一定**写回主内存（现代实现是"在缓存一致性域内可见"，通常停在 L3/共享缓存，不强制到 DRAM）。
2. `lock` 的失效范围是**具体 cache line**（通过目录/嗅探），不是"整个缓存"。
3. x86 上 volatile **读**没有 `lock`；只有**写**有。

**准确表述**：`lock` 前缀 = 原子性 + 排空 store buffer + 广播失效并等待 ACK + 指令序列化点。可见性由这四者共同保证。

---

## 7. volatile 典型用法矩阵（对错各半，面试最爱）

### 7.1 ✅ 正确用法 1：状态标志（一写多读）

```java
volatile boolean shutdownRequested;

public void shutdown() { shutdownRequested = true; }
public void doWork() {
    while (!shutdownRequested) {   // 每次必须重读，不能被外提
        // ... 干活
    }
}
```
**为什么对**：典型的"一写多读"，写方只有开关语义，不需要复合原子。`volatile` 读零成本，比加锁划算太多。这是 volatile 最纯粹、最安全的用法。

### 7.2 ✅ 正确用法 2：DCL 双重检查锁（见 1.2）

**唯一要点**：`volatile` 语义上防的是"**构造器执行与引用赋值之间的重排序**"，不是"缓存可见"。很多人答成"防止读到缓存旧值"，漏掉重点。

### 7.3 ✅ 正确用法 3：一次性安全发布（One-time safe publication）

```java
// 不可变对象 / 只初始化一次的大对象
private volatile Map<String, Config> cache;
public Map<String, Config> get() {
    Map<String, Config> m = cache;
    if (m == null) {                    // 局部变量承接，减少 volatile 读次数（性能小技巧）
        synchronized (this) {
            m = cache;
            if (m == null) {
                m = loadFromDisk();     // 耗时初始化
                cache = m;              // volatile 写 → 释放：m 内部所有字段对外可见
            }
        }
    }
    return m;
}
```
**原理**：volatile 写的 release 语义保证"`m` 内部所有字段的写"对读到 `cache` 的线程可见（靠 hb 传递性）。

### 7.4 ✅ 正确用法 4：开销较低的"读-写锁"策略

```java
// 用 volatile 优化读路径，用 synchronized 保证写路径
private volatile int value;
public int getValue() { return value; }          // 读：无锁
public synchronized void update(int v) { value = v; }   // 写：加锁
```
**限制**：只适用于"**读远多于写，且写操作之间需要互斥**"的场景。若读也需要复合操作（如 `getAndIncrement`），则失效。

### 7.5 ✅ 正确用法 5：volatile + CAS = 原子类的内核

```java
// AtomicInteger 的真实结构（JDK 8）
private volatile int value;                                  // ← 可见性
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  // ← 原子性
}
public final int incrementAndGet() {
    for (;;) {
        int current = get();        // volatile 读，保证拿到最新
        int next = current + 1;
        if (compareAndSet(current, next)) return next;   // CAS 失败就重试
    }
}
```
**这是"volatile 不保证原子性"的官方解法**：`volatile` 负责"读到最新"，`CAS` 负责"比较并替换不可分割"。两者缺一不可：
- 只有 volatile：`i++` 丢更新（1.3 节）；
- 只有 CAS 而字段非 volatile：CAS 指令本身原子，但**读到的 expect 可能是旧的**，导致 ABA 或大量无谓重试。

### 7.6 ❌ 错误用法 1：依赖 volatile 做计数

```java
volatile int count;
count++;                    // 错
```

### 7.7 ❌ 错误用法 2：volatile 数组 ≠ 数组元素 volatile（高频坑）

```java
volatile int[] arr = new int[10];
arr[0] = 1;                 // ← 这个写没有 volatile 语义！
```

**原因（必须讲清）**：`volatile` 修饰的是**引用变量 `arr` 本身**的读写，不是"数组元素"。`arr[0]=1` 等价于 `iaload/iastore`，走的是普通访存。

**三个正确替代**：

```java
// ① AtomicIntegerArray（内部用 VarHandle 的 volatile 语义 + 内存屏障，元素级 volatile）
AtomicIntegerArray arr = new AtomicIntegerArray(10);
arr.set(0, 1);

// ② 数组元素声明为 volatile 的包装（Java 不允许 volatile 数组元素，只能包一层对象）
volatile Node[] table;   // ConcurrentHashMap 的做法：只对 table 引用 volatile，
                         // 元素用 Unsafe/VarHandle 的 getObjectVolatile 读

// ③ VarHandle 对数组元素做 volatile 访问（JDK 9+）
static final VarHandle INT_ARR = MethodHandles.arrayElementVarHandle(int[].class);
INT_ARR.setVolatile(arr, 0, 1);
```

> `ConcurrentHashMap` 的经典实现就是这个坑的答案：`transient volatile Node<K,V>[] table;` 的 `volatile` 只保证"换表（resize）被看见"，**桶内节点的读写全靠 `Unsafe.getObjectVolatile`/`casTabAt`**。

### 7.8 ❌ 错误用法 3：以为 volatile 引用能保证对象内部字段的可见性

```java
volatile Holder h;
h = new Holder();      // ✅ 引用写入可见（且禁止 new 的三步重排）
h.field = 1;           // ❌ 之后对 field 的写，不因为 h 是 volatile 而自动"每次可见"
int x = h.field;       // 读同理：先读 h（volatile 语义），再读 field（普通语义）
```
**准确表述**：`volatile` 只作用于它修饰的那个变量的一次读/写。**它不会把"整条对象图"都变成 volatile。**

### 7.9 ❌ 错误用法 4：`check-then-act` 靠 volatile 保证互斥

```java
volatile boolean inited;
if (!inited) {          // 两个线程可能同时判断为 false
    init();
    inited = true;
}
```
必须换成 `synchronized` 或 `AtomicBoolean.compareAndSet(false, true)`。

### 7.10 汇总速查

| 场景 | 是否适合 volatile | 说明 |
|---|---|---|
| 状态开关（一写多读） | ✅ | 最经典 |
| DCL 单例 | ✅ 必须 | 防构造器/引用赋值重排 |
| 一次性安全发布 | ✅ | release 语义 + hb 传递 |
| 读多写少 + 写需互斥 | ✅ | 读写分离策略 |
| 计数器 `i++` | ❌ | 用 Atomic / LongAdder |
| check-then-act | ❌ | 用 CAS 或锁 |
| 多变量不变式（a+ b 需同时可见一致） | ❌ | 用锁 |
| 数组元素 | ❌（需包装） | AtomicXxxArray / VarHandle |
| 对象内部字段 | ❌ | volatile 只管一层 |

---

## 8. 横向对比：volatile 在整个 Java 同步体系中的坐标

### 8.1 五种"内存语义工具"对比

| 维度 | `volatile` | `synchronized` | `AtomicXxx`(CAS) | `VarHandle`（JDK 9+） | `final` |
|---|---|---|---|---|---|
| 可见性 | ✅ | ✅ | ✅（value 是 volatile） | ✅ 可按模式选择 | ✅（构造完成后） |
| 有序性 | ✅ | ✅ | ✅（CAS 是内存屏障级） | ✅ | ✅（禁止构造器内写外逸） |
| 原子性 | ❌ 复合操作 | ✅ | ✅ 单变量 RMW | ❌（除 CAS 方法） | — |
| 阻塞 | 无 | 有 | 无（自旋重试） | 无 | 无 |
| 粒度 | 单变量 | 代码块/方法 | 单变量 | 单变量/数组元素/字段 | 单字段 |
| 开销 | 写贵读免费 | 竞争时重 | 无竞争时轻 | 可精细控制（模式化） | 0 |
| 典型用途 | 标志、DCL、发布 | 复合逻辑 | 计数、无锁栈队列 | 替代 Unsafe、精细内存序 | 不可变对象安全发布 |

### 8.2 `volatile` 与 `final` 的组合：JSR-133 对 final 的"补丁"

**为什么 final 也需要内存语义**？因为 `new` 的三步重排问题，对 final 字段同样成立：

```java
class FinalField {
    final int x;
    FinalField() { x = 1; }     // 构造器里写 final
}
// 另一个线程可能看到 obj != null 但 obj.x == 0
```

**JSR-133 给 final 的两条规则**：

1. **构造器内**对 final 字段的写，**不能**被重排到构造器之外（编译器在 final 写后插 `StoreStore`）。
2. **首次**读到"引用指向该对象"的线程，一定能看到构造器内 final 字段的正确值（`LoadLoad` 屏障，配合"禁止 final 字段引用从构造器逸出"）。

**推论（面试常考）**：`final` 提供了**免锁的安全发布**——所以"不可变对象（所有字段 final）天生线程安全，无需 volatile"。这是 `volatile` 之外的第二种"零成本可见性"手段。

**但前提是"this 不能在构造器里逸出"**：

```java
class Bad {
    static Bad INSTANCE;
    final int x;
    Bad() { INSTANCE = this; x = 1; }   // ❌ this 逸出 → final 语义失效
}
```

### 8.3 `volatile` 与 `ThreadLocal`、`Thread` 的关系（一个易混淆点）

- `ThreadLocal` 解决的是"**共享**"，不是"**同步**"：它让每个线程有独立副本，从根本上避免竞争。
- `volatile` 解决的是"共享变量的**协调**"。
- 二者是"避免共享" vs "管理共享"的关系，**不在同一个问题维度上**，不能互相替代。

**例外（冷知识，面试深水区）**：`ThreadLocal` 的 `ThreadLocalMap` 存在 `Thread.threadLocals` 字段里，而 `Thread` 对象本身是共享的——`threadLocals` 字段声明为**普通字段**，但依赖"只有本线程访问自己的 threadLocals"这一约束来保证安全。这是"**用约定代替同步**"的典型。

---

## 9. 高级延伸（区分"会用"和"懂"的分水岭）

### 9.1 VarHandle 的四种访问模式（JDK 9 的内存序梯度）

这是现代 Java 并发编程的"精度旋钮"，**比 volatile 更细**：

| 模式 | 方法示例 | 保证 | 底层（x86） | 用途 |
|---|---|---|---|---|
| **Plain** | `get/set` | 无可见性、无有序性（等同普通字段） | `mov` | 不需要跨线程语义时 |
| **Opaque** | `getOpaque/setOpaque` | 保证**单变量访问不被编译器优化掉/不撕裂**，**不保证跨核顺序** | `mov` + 编译屏障 | 只需"每次真的读一次" |
| **Acquire/Release** | `getAcquire/setRelease` | 单向：acquire 后不重排、release 前不重排 | `mov`（x86 上 release 无 lock） | 生产者-消费者，比 volatile 便宜 |
| **Volatile** | `getVolatile/setVolatile` | 双向全序（= volatile） | `mov` + `lock` | 与 volatile 完全等价 |

**性能梯度（x86，经验值）**：`Plain` < `Opaque` < `Acquire/Release` < `Volatile`

**这是"为什么要发明 VarHandle"的核心答案**：`volatile` 只有一档，太粗；`Unsafe` 没有类型安全且不可持续；`VarHandle` 给了 4 档可调的内存序，并支持"字段、数组元素、ByteBuffer 位置"的原子访问，且**编译期类型检查 + 可作为 final 静态字段被 JIT 常量折叠**（性能优于反射）。

### 9.2 happens-before 的"最小保证"与"实现过度保证"

```java
// 这个程序在 x86 上很可能正常工作，但在 JMM 上是不安全的
int x = 0; boolean flag = false;   // 都不是 volatile
// T1: x = 1; flag = true;
// T2: if (flag) print(x);
```
**x86 上**：因为 TSO 禁止 StoreStore 重排，实践中常常"看起来没问题"。
**JMM 上**：`flag` 非 volatile，T2 可能永远看不到 `flag=true`（编译器外提、缓存不刷新），**代码是错的**。

> **这是最危险的一类认知**："在我机器上跑对了"不等于"符合 JMM"。面试时若被问"你遇到过的并发 bug"，答这个层次会很加分。

**反过来，JMM 也允许"过度保证"**：JVM 可以在 x86 上少插屏障（因为硬件已经保证），这叫"**利用平台的强一致性做减法**"，也是 5.2 节源码里 4 个空实现的原因。

### 9.3 64 位变量撕裂（Word Tearing）的真相

- **JLS §17.6**：非 volatile 的 `long`/`double` 读写**允许**被实现为两个 32 位操作（可能读到"半个新值"）。
- **JLS §17.7**：volatile 的 `long`/`double` **必须**原子。
- **HotSpot 实践**：所有主流 64 位平台（x86-64、aarch64）的 `mov` 本身就是 64 位原子，**实际上不会撕裂**；但在 32 位 JVM 上有真实风险。
- **JSR-133 的历史**：早期 JMM 曾提出"禁用 64 位非原子访问"，后因性能顾虑放弃，改为"允许但鼓励原子"。

> 面试标准答法："规范层面允许撕裂，实现层面几乎不发生；**要绝对安全就加 volatile**，加完规范强制原子。"

### 9.4 与 C++11 内存模型的映射（跨语言加分项）

| Java | C++11/14 | 说明 |
|---|---|---|
| `volatile`（Java） | `std::atomic<T>` 默认 seq_cst 或 `memory_order_seq_cst` | **注意：Java 的 volatile ≈ C++ 的 atomic seq_cst，不是 C++ 的 volatile！** |
| `VarHandle.setRelease` | `store(memory_order_release)` | 一一对应 |
| `VarHandle.getAcquire` | `load(memory_order_acquire)` | 一一对应 |
| `VarHandle.setOpaque` | `memory_order_relaxed`（近似） | 略有差异 |
| 普通字段 | 普通变量（有 UB 风险） | Java 无 UB，只有"数据竞争导致不确定值" |

> **最大的坑**：C++ 的 `volatile` **不提供线程间可见性/原子性**（它只禁止编译器优化，用于 MMIO）；Java 的 `volatile` 才是"同步原语"。**面试若被问"C++ volatile 和 Java volatile 区别"，这题就是送分/送命的分界线。**

### 9.5 `volatile` 与"指令级并行的边界"：为什么 volatile 会拖慢流水线

`lock` 前缀的实现细节（Intel 微架构）：

1. 让 store buffer 中所有条目**等待提交**（drain），触发 `mfence` 类行为；
2. 该指令成为**序列化点**（serializing-ish），后续指令不能提前取指执行；
3. 高竞争时触发**缓存行乒乓（cache line ping-pong）**，L1 miss 率飙升。

**实测经验（供回答"volatile 性能"用）**：
- 单线程无竞争 volatile 写：约 **10~30 ns** 级（比普通写慢一个数量级）；
- 高竞争 volatile 写：可能退化到 **100+ ns**，甚至不如 `synchronized`（后者有偏向锁/轻量级锁优化，且合并了多变量）。

> 结论：**volatile 的高频写入是性能杀手**。它适合"很少写、经常读"的模式。

### 9.6 `volatile` 在 JMM 中的"同步动作"地位（形式化视角）

JSR-133 把 volatile 读写定义为**同步动作（synchronization action）**，与 lock/unlock、线程 start/join 同级。这带来两个形式化结论：

1. **同步顺序（synchronization order）**：所有 volatile 读写 + 锁操作构成一个全序（每个线程内保序）；**volatile 变量规则**在此基础上建立 hb。
2. **因果性要求（causality）**：JMM 禁止"凭空出现"的值（out-of-thin-air）。这是 JSR-133 相对旧 JMM 的最大修正——旧模型允许 `r1=r2` 见 2.5 节的 Dekker 场景产生"两边都读到对方未来值"的循环依赖，新模型通过"提交/一致性"约束禁止。

**这段话的价值**：能说出"JSR-133 修正了旧 JMM 的 out-of-thin-air 问题"，基本可以横着走进任何一家大厂的二面。

### 9.7 常见误区总清单（自查表）

| # | 误区 | 正解 |
|---|---|---|
| 1 | volatile 保证原子性 | ❌ 只保证单次读/写；复合操作不安全 |
| 2 | volatile 让变量"存到主内存" | ❌ 现代 CPU 写到缓存一致性可见层，不强制 DRAM |
| 3 | volatile 会让所有缓存失效 | ❌ 只失效该 cache line |
| 4 | volatile 读也有 `lock` 前缀 | ❌ x86 上读无额外指令 |
| 5 | volatile 修饰数组 = 元素 volatile | ❌ 只修饰引用 |
| 6 | volatile 修饰引用 = 对象字段 volatile | ❌ 只一层 |
| 7 | volatile 能替代锁 | ❌ 只能替代"纯发布/标志"场景 |
| 8 | volatile 一定比 synchronized 快 | ⚠️ 高频写/高竞争时可能更慢 |
| 9 | volatile 只靠 JIT 实现 | ❌ 解释器执行也生效（Access API 层） |
| 10 | x86 上 StoreStore 是空的所以 volatile 无序性保证 | ❌ 编译屏障（MemBar 节点）仍在 |
| 11 | 非 volatile 的 long 一定撕裂 | ⚠️ 规范允许，实践几乎不发生 |
| 12 | C++ volatile ≈ Java volatile | ❌ 完全不同，C++ 的那个不保证可见性 |
| 13 | volatile 能保证多个变量的原子性 | ❌ 每个变量独立，无整体不变式 |
| 14 | volatile 读写之间的普通变量也"自动 volatile" | ❌ 靠 hb 传递一次性可见，不是持续可见 |
| 15 | 有了 volatile 就不需要 happens-before 分析 | ❌ hb 是判断正确性的唯一工具 |

---

## 10. 大厂面试题库（按考核程度分档）

### 10.0 使用说明：星级与职级映射

| 星级 | 含义 | 典型出现环节 | 对应职级锚点 |
|---|---|---|---|
| ★★★★★ | **必答**，答不上直接挂 | 一面/二面开场 | 所有级别（含校招） |
| ★★★★ | **有区分度**，二面主线 | 二面/交叉面 | 阿里 P6+ / 字节 2-1+ / 美团 L7+ |
| ★★★ | **深水区**，决定"优秀"与"合格" | 三面/技术终面 | 阿里 P7+ / 字节 2-2+ / 腾讯 T10+ |
| ★★ | **加分/造火箭**，能聊=惊喜 | 架构面/专家面 | P8+ / 专家岗 |
| ☆ | 冷知识，用来测"是否只会背八股" | 任意 | — |

> **重要提醒**：大厂面试的真实规律是——**同一道题会连续追问 3~5 层**。所以每题除了"参考答案"，我还给了"**追问链**"，这才是决定评级的部分。

---

### 一档：基础必答（★★★★★）

#### Q1. `volatile` 的作用是什么？它能保证原子性吗？

**出处**：几乎所有大厂一面（阿里、腾讯、字节、美团、京东、百度、快手、滴滴）
**考核程度**：★★★★★（答错"保证原子性"= 直接淘汰）

**参考答案（三段式）**：

1. **两个保证**：
   - **可见性**：一个线程写 volatile 变量后，后续读该变量的线程一定能看到最新值。
   - **有序性**：禁止 volatile 读写与周围其他内存操作重排序（提供 release/acquire 语义）。
2. **一个不保证**：**原子性**。`count++`、`if(x==null)x=new()` 这类复合操作依然不安全。
3. **例外**：单个 volatile 变量的**一次读或一次写**是原子的（含 `long`/`double`，JLS §17.7 强制）。

**为什么不能保证原子性（要能讲到硬件层）**：

```
Java 层：getstatic → iconst_1 → iadd → putstatic      （三步，中间可被插入）
硬件层：inc [mem] 是 read-modify-write，非 lock 前缀 → 不是单条原子指令
```

volatile 只能保证"`getstatic` 拿到最新"和"`putstatic` 立刻可见"，**不能把三步合成一步**。要原子就必须用 `lock` 前缀级的手段（CAS）或互斥（锁）。

**追问链**：
- 追问 1：那 `volatile int i = 1` 这条赋值语句是原子的吗？→ **是**（单次写）。
- 追问 2：那为什么 `i++` 不是？→ 因为它是"读-改-写"三步，不是单次访存。
- 追问 3：既然不保证原子性，那它存在的意义是什么？→ 覆盖"状态标志、一次性发布、DCL"这类**不需要复合原子**的场景，且比锁便宜。
- 追问 4：`long` 的 volatile 呢？→ 单次读写原子（规范强制），但 `long++` 照样不原子。

**延伸**：JLS §17.4.5 明确"volatile 只保证对该变量的单次读写原子"；JLS §17.7 单独规定 volatile long/double 的原子性。

---

#### Q2. `volatile` 和 `synchronized` 有什么区别？

**出处**：阿里、腾讯、美团（一面必问）
**考核程度**：★★★★★

**标准对比表（口头版，5 个维度就够拿分）**：

| 维度 | volatile | synchronized |
|---|---|---|
| 原子性 | ❌ 复合操作不安全 | ✅ 临界区互斥 |
| 可见性 | ✅ | ✅ |
| 有序性 | ✅ 禁止重排 | ✅ |
| 阻塞 | 不阻塞（无上下文切换） | 竞争时阻塞（重量级锁挂起） |
| 粒度/开销 | 单变量，读免费写有 `lock` | 代码块/方法，无竞争偏向锁≈0 |

**加分表述（这句能拉开差距）**：

> 二者在内存语义上是**同构**的：`volatile 写 ≈ unlock（release）`，`volatile 读 ≈ lock（acquire）`。区别只在于 `synchronized` 额外提供了**互斥（mutual exclusion）**，而 volatile 没有。

**再补一句（P7 级）**：`synchronized` 的 `monitorenter/monitorexit` 在字节码里**显式存在**，而 `volatile` 只在 `access_flags` 里有 `ACC_VOLATILE` 一位——**这是 volatile "轻"的结构性原因**。

**追问链**：
- 追问：`synchronized` 能完全替代 volatile 吗？→ **语义上可以**（它包含 volatile 的全部保证），但**性能上不划算**（热路径上的状态标志加锁会有 monitorenter 开销 + 可能的锁膨胀）。
- 追问：volatile 能替代 synchronized 吗？→ 只在"单变量发布/标志"这极窄场景可以。

---

#### Q3. `volatile` 的可见性在底层是怎么实现的？

**出处**：字节、阿里、美团（二面主线题）
**考核程度**：★★★★★

**必须按"三层"回答**（只答"内存屏障"是 P5 水平，答到汇编是 P7 水平）：

**第一层：JMM 语义（翻译层）**

```
volatile 写 = release：写之前的普通读写 → 插 StoreStore 屏障，禁止上浮
volatile 写之后       → 插 StoreLoad 屏障（x86 落地为 lock）
volatile 读之后       → 插 LoadLoad + LoadStore，禁止下沉
```

**第二层：JVM/JIT 层**
- 字节码只有 `ACC_VOLATILE` 标志，指令是普通 `getfield/putfield`；
- 解释器和 JIT 通过 `Access`/`Unsafe` 的 `load_volatile`/`store_volatile` 处理；
- JIT 生成 `MemBarStoreStore`/`MemBarStoreLoad` 等节点，**并禁止 CSE/hoisting/死存储消除**。

**第三层：硬件层（x86 实例）**

```
movl  $0x1, y_offset(%rsi)     ; 普通写先落地
lock addl $0x0, (%rsp)         ; ★ 全屏障：排空 store buffer + 广播失效 + 序列化点
movl  $0x1, x_offset(%rsi)     ; volatile 写本身只是普通 mov
```

`lock` 前缀的三重效果：**原子性 + 刷出 store buffer（可见性）+ 缓存行失效广播并等 ACK（一致性）**。配合 MESI 协议，其他核的副本从 S/E → I，下次读必然 cache miss 拿新值。

**追问链**：
- 追问：为什么是 `addl $0x0,(%rsp)` 而不是 `mfence`？→ `lock` 必须跟一条读改写指令，`add $0` 是最便宜的载体，**语义等价于全屏障**，且部分微架构上比 `mfence` 更省（JDK 8 的选择）。
- 追问：为什么插在写**前面**？→ 为了同时满足 StoreStore（禁止普通写上浮）和 StoreLoad（`lock` 自带）。
- 追问：读为什么没有屏障？→ x86 的 TSO 已保证 LoadLoad/LoadStore 顺序，`acquire()` 实现为空。

---

#### Q4. `volatile` 能保证有序性吗？它是怎么禁止重排序的？

**出处**：阿里、美团、拼多多
**考核程度**：★★★★★（很多人只会说"禁止重排"，说不出屏障位置）

**答案**：能。手段是**在两个层面同时插屏障**：

**（1）编译器屏障（软件层）**——让 C1/C2 不敢跨越：
- C2 在 volatile 字段的 `LoadNode`/`StoreNode` 前后插入 `MemBar` 节点；
- 优化器在 `PhaseIdealLoop`/`PhaseCCP` 检查到 `MemBar` 就停止调度/外提/合并。

**（2）CPU 屏障（硬件层）**——让乱序执行不越界：
- x86：`lock`（全屏障）；ARM：`dmb ish` / `ldar` / `stlr`。

**JMM 规定的四个位置**（能背下来就是加分）：

| 位置 | 插入屏障 | 拦的是哪种重排 |
|---|---|---|
| volatile 写**之前** | `StoreStore` | 前面的普通写不能跑到 volatile 写之后（**DCL 靠它**） |
| volatile 写**之后** | `StoreLoad` | volatile 写不能被后面的读写越过 |
| volatile 读**之后** | `LoadLoad` | 后面的普通读不能跑到 volatile 读之前 |
| volatile 读**之后** | `LoadStore` | 后面的普通写不能跑到 volatile 读之前 |

**一句话记忆**：**写像"关闸"，前面的一切必须先过闸；读像"开闸"，后面的一切必须等闸开。**

**追问链**（这题最容易被"追杀"）：
- 追问：为什么 volatile 写**之后**要 StoreLoad，而读之前不需要？→ 因为 StoreLoad 是"写→读"方向，只有可能发生在写之后；读之前只需要处理"读→读"（无）与"写→读"（由前一个写的 StoreLoad 覆盖）。
- 追问：既然是"禁止重排"，那 volatile 写之前的普通读会被拦住吗？→ **不会**。`StoreStore` 只排 Store；JMM 也不要求"读→写"方向（那是 LoadStore，发生在 volatile 读之后）。**这个细节能答对，说明真读过 JSR-133。**

---

#### Q5. 什么场景下应该用 `volatile`？举三个真实例子。

**出处**：腾讯、京东、快手（一面常规题）
**考核程度**：★★★★★

**三类必须能脱口而出**（见第 7 章）：

1. **状态标志（一写多读）**：`volatile boolean shutdownRequested`，避免 JIT 把判断外提造成死循环。
2. **DCL 单例 / 一次性安全发布**：防 `new` 的三步重排，保证别的线程不会拿到半初始化对象。
3. **volatile + CAS 的无锁结构**：`AtomicInteger` 的 `value` 是 volatile，靠 CAS 补原子性。

**加分项（说出"不该用"的场景）**：
- 计数、累加（`count++`）；
- check-then-act（需要互斥）；
- 多变量不变式（需要整体一致）；
- 数组元素、对象深层字段（volatile 只管一层）。

**追问链**：
- 追问：状态标志为什么不用 `AtomicBoolean`？→ 语义上可以，但 `volatile` 读免费、写少；`AtomicBoolean` 多一层 CAS 成本。**一写多读时 volatile 更轻。**
- 追问：如果状态是"关闭/运行/暂停"三态呢？→ 可以用 volatile + enum（单次写原子），但如果状态切换需要复合逻辑（暂停时要清空队列），就得上锁。

---

#### Q6. DCL 单例为什么必须加 `volatile`？

**出处**：阿里、字节、美团、华为（超高频）
**考核程度**：★★★★★（答"防止缓存"必被判错）

**必须答到"指令重排"而不是"缓存可见"**：

```java
instance = new Singleton();
```

在 JVM 层面是**三步**（`new` + `invokespecial <init>` + `putstatic`）：

```
① 分配内存（字段默认值，如 a = 0）
② 执行构造器（a = 1）
③ 引用赋值 instance = 内存地址
```

②与③**没有数据依赖**，可以重排为 ①→③→②。于是另一个线程在 ③ 之后、② 之前通过第一次检查，拿到"引用非 null 但字段还是默认值"的**半成品对象**。

**加 volatile 的效果**：`putstatic` 是 volatile 写 → 前面插 `StoreStore` 屏障 → ②的写不能被排到③之后 → 安全。

**追问链（这是分水岭，能走完 4 层的很少）**：
1. 追问：那用 `synchronized` 包住整个方法可以吗？→ **可以**（`monitorexit` 的 release 语义同样保证），但锁粒度变粗、每次 get() 都加锁，性能差。DCL 的目的就是"**第一次之后不走锁**"。
2. 追问：为什么用局部变量 `Singleton result = instance`？→ 减少 volatile 读次数，且避免"instance 被置 null"竞态（`final` 局部引用语义），是 JDK 里常见的 Double-Check 惯用法。
3. 追问：还有别的实现吗？→ **静态内部类（Holder）**（靠类初始化的 `clinit` 加锁 + JMM 保证）+ **enum 单例**（防反射/序列化）+ **`AtomicReference` + CAS**。
4. 追问：`volatile` 在这里能保证"构造器一定执行完"吗？→ **不能保证"构造器内所有逻辑不逸出"**。如果构造器里 `this` 逸出（注册监听、启动线程），volatile 也救不了——**final 语义 + 不逸出才是根本**。

**延伸**：`new` 的三步在 JLS 中没有明确规定顺序，是 HotSpot 的字节码顺序 + 编译器/硬件的重排自由度共同造成。所以这题在理论层面是"**JMM 允许的重排**"，实践层面是"**确实现象**"。

---

### 二档：进阶（★★★★）

#### Q7. 四种内存屏障分别是什么？`volatile` 用了哪些？

**出处**：字节、美团、蚂蚁（二面深挖）
**考核程度**：★★★★

**定义**（见 2.6 节表格），口头要能顺：
- `LoadLoad`：Load1 ; barrier ; Load2 → 读读有序；
- `StoreStore`：Store1 ; barrier ; Store2 → 写写有序；
- `LoadStore`：Load1 ; barrier ; Store2 → 读写有序；
- `StoreLoad`：Store1 ; barrier ; Load2 → 写读有序（**最重，唯一需要全屏障的**）。

**volatile 用到的**：
- volatile **写**：`StoreStore`（写前）+ `StoreLoad`（写后）；
- volatile **读**：`LoadLoad` + `LoadStore`（读后）。

**追问链（这里最能体现水平）**：
- 追问：x86 上这四种有几种是空实现？→ **三种**。x86 是 TSO，只有 StoreLoad 需要 `lock`/`mfence`；`LoadLoad`/`LoadStore`/`StoreStore` 的 `OrderAccess` 实现为空。
- 追问：空实现那 volatile 还有效吗？→ **有效**。空的是"硬件屏障"，**编译屏障（C2 的 MemBar 节点）依然存在**，编译器仍然不敢重排。**这是最容易答错的一题**（10 个候选人 8 个答"x86不需要屏障所以volatile只靠编译器"）。
- 追问：那 `StoreLoad` 在 x86 用什么实现？→ `lock addl $0x0,(%rsp)`（JDK8）或 `mfence`（部分实现/JDK9+ 选项）。

---

#### Q8. 为什么 `volatile` 读在 x86 上几乎没有开销，而写很贵？

**出处**：字节、快手、蚂蚁
**考核程度**：★★★★

**答案**：
1. **x86 是 TSO（Total Store Order）**：硬件保证 Load 不会越过前面的 Load、Load 不会越过后面的 Store。所以 volatile 读所需的 `LoadLoad`+`LoadStore` **天然满足**，`OrderAccess::acquire()` 是空实现 → 读 = 一条普通 `mov`。
2. **StoreLoad 是 x86 唯一允许的重排**（写进 store buffer 后，后面的读先执行）。volatile 写要 `lock` 版本的全屏障，会：
   - 排空 store buffer（等所有写提交）；
   - 广播 cache line 失效并**等所有核 ACK**；
   - 成为近似序列化点，阻塞流水线重叠。
3. **量级**：读 ≈ 普通访存（L1 命中 4 cycle）；写 ≈ 20~50 cycle 起，高竞争 ping-pong 时可达数百 cycle。

**对比 ARM（这句话能加很多分）**：
> ARM 是弱内存模型，四种重排全允许，所以 volatile 读也要 `ldar`/`dmb ishld`，写也要 `stlr`/`dmb ish`。**"volatile 读免费"这个结论只在 x86 成立**，在 ARM 上读写都贵。

**追问链**：
- 追问：`lock addl` 里的 `add $0` 有什么意义？→ 无业务意义，只是给 `lock` 一个合法的读改写载体（`lock` 只能修饰特定指令）。用 `add $0` 最便宜，不会触发异常/内存访问。
- 追问：JDK 9+ 有变化吗？→ `VarHandle` 引入后，`setRelease`/`getAcquire` 可以做"减法"：release 不要 StoreLoad，所以写更便宜；这给开发者一个"按需付费"的口子。

---

#### Q9. `volatile` 和 `AtomicInteger` 有什么区别？`AtomicInteger` 内部为什么还要 `volatile`？

**出处**：阿里、美团、京东、拼多多
**考核程度**：★★★★

**区别**：

| | volatile | AtomicInteger |
|---|---|---|
| 原子性 | ❌ | ✅（CAS） |
| `++` | 不安全 | 安全（`incrementAndGet`） |
| 读 | 直接读 | `get()` 内部也是 volatile 读 |
| 写 | 直接写 | CAS 循环（可能自旋） |
| 复合操作 | ❌ | ✅ `compareAndSet`、`addAndGet` |
| 开销 | 写一条 `lock` | 无竞争 ≈ 一条 `lock cmpxchg`；高竞争自旋/退化 |

**"为什么内部还要 volatile"的答案（这是这题的真正考点）**：

```java
private volatile int value;                            // ① 可见性：让循环里每次 get() 读到最新
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);   // ② 原子性
}
```

若 **没有 volatile**：CAS 指令本身仍原子，但 `incrementAndGet` 的 `for(;;)` 里 `get()` 可能一直读到**本核缓存的旧值** → CAS 永远失败 → **活锁（livelock）**。所以：

> **volatile 提供"最新值来源"，CAS 提供"提交的原子性"，两者是配套的，缺一不可。**

**追问链**：
- 追问：CAS 有什么问题？→ **ABA**（用 `AtomicStampedReference` 加版本号解决）、**自旋开销**（高竞争时 CPU 空转，用 `LongAdder` 缓解）、**只能保证单变量原子**（多变量用 `AtomicReference` 打包或加锁）。
- 追问：高并发计数用哪个？→ `LongAdder`（分段 Cell + `@Contended` 防伪共享），`sum()` 时汇总，牺牲"读的实时精确性"换写吞吐。
- 追问：`AtomicInteger` 的 `valueOffset` 怎么来的？→ `Unsafe.objectFieldOffset(Field)` 反射拿到字段偏移，JDK 9+ 改用 `VarHandle`。

---

#### Q10. `volatile` 和 happens-before 是什么关系？

**出处**：阿里 P7、字节 2-2、美团 L8（这题直接决定评级）
**考核程度**：★★★★（能讲清"传递性"就是 P7）

**答案结构**：

1. **JMM 的正式规范是 happens-before，不是"主内存/工作内存"那套八操作**（后者是通俗模型）。
2. **volatile 规则**：对同一 volatile 变量的**写** hb 后续的**读**。
3. **真正的价值在"传递性"**：

```
a = 1;   b = 2;   flag = true;      // writer
--------------------------------
if (flag) { x = a; y = b; }         // reader
```
   推导：`a=1` hb `flag=true`（程序顺序）→ `flag=true` hb `if(flag)`（volatile 规则）→ `if(flag)` hb `x=a`（程序顺序）→ **传递** ⇒ `a=1` hb `x=a`。

4. **所以正确的表述不是"volatile 把其他变量也刷出去了"**，而是：**volatile 建立了一条 hb 边，配合传递性，使"写之前的普通写"对"读到该值的线程"可见。**

**追问链（深水区）**：
- 追问：那 volatile 读之后的普通写，对 writer 可见吗？→ **不可见**。hb 是单向的，读方后续的写不会反向可见。
- 追问：两个 volatile 变量之间呢？→ 只有**同一个** volatile 变量的写读才建立 hb。
- 追问：hb 是"时间先后"吗？→ **不是物理时间**，是"可见性时序"。hb 不违反，但物理执行可以先做后发生的指令（只要结果符合 hb 约束）。
- 追问：那怎么判断我的代码有没有数据竞争？→ 用 hb 完备性：**任意两次冲突访问（至少一次写）之间必须有 hb 边**，否则就是 data race。

---

#### Q11. `volatile` 能保证 `long`/`double` 的原子性吗？

**出处**：腾讯、华为、滴滴（偏八股但有深度）
**考核程度**：★★★★

**三层回答**：

1. **JLS §17.6**：非 volatile 的 `long`/`double` 读**允许**被实现成两个 32 位操作 → **可能撕裂（word tearing）**，读到"高32位新、低32位旧"。
2. **JLS §17.7**：**volatile 的 `long`/`double` 读写必须原子**（规范强制）。这是 volatile 的一个**额外保证**，很多人不知道。
3. **实践**：x86-64/aarch64 的 `mov` 本身就是 64 位原子，**实际几乎不会撕裂**；32 位 JVM 才有真实风险。

**追问链**：
- 追问：为什么 JMM 当初不干脆禁止 64 位撕裂？→ JSR-133 讨论过（"atomicity for 64-bit"），但当时 32 位平台占主流，强制原子会让每次 long 访问都上锁，性能不可接受，最终**妥协为"允许但推荐 volatile"**。
- 追问：那 `long++` 加 volatile 就安全了？→ ❌ 仍然不安全（复合操作）。

---

#### Q12. `volatile` 修饰数组，能保证数组元素的可见性吗？

**出处**：阿里、美团、字节（很爱考这个坑）
**考核程度**：★★★★

**答案**：**不能**。`volatile` 修饰的是**引用变量本身**，`arr[0]=1` 走的是 `iastore`（普通写），**没有任何 volatile 语义**。

```java
volatile int[] arr = new int[10];
arr[0] = 1;    // ❌ 不是 volatile 写
arr = new int[10];   // ✅ 这个是 volatile 写（换引用）
```

**三种正确做法**：
1. `AtomicIntegerArray` / `AtomicLongArray` / `AtomicReferenceArray`；
2. JDK 9+ `VarHandle` 的 `arrayElementVarHandle(...).setVolatile(...)`；
3. 读用 `Unsafe.getObjectVolatile` + 写用 `casTabAt`（**ConcurrentHashMap 的做法**）。

**追问链**：
- 追问：那 `ConcurrentHashMap` 的 `transient volatile Node[] table` 有什么用？→ **只保证"换表（resize 时替换整个数组引用）"被其他线程看见**；桶内元素的读写全靠 `Unsafe.getObjectVolatile`/`casTabAt`。
- 追问：为什么数组元素不能直接声明 volatile？→ Java 语法层面 `volatile` 只能修饰**字段/变量**，不能修饰"元素"这种位置；元素级语义必须靠访问方法（VarHandle/Unsafe）注入。

---

#### Q13. `volatile` 的性能开销有多大？什么时候它会比 `synchronized` 更差？

**出处**：字节、快手、蚂蚁（性能敏感团队爱问）
**考核程度**：★★★★

**答案**：

| 操作 | x86 经验开销 |
|---|---|
| volatile 读 | ≈ 普通读（L1 4 cycle / L2 12 / L3 30+） |
| volatile 写（无竞争） | ≈ 20~50 cycle（`lock` 排空 store buffer） |
| volatile 写（高竞争） | 100+ cycle，触发 cache line ping-pong |
| synchronized（无竞争，偏向锁/轻量级） | ≈ CAS 10~20 cycle |
| synchronized（有竞争） | 膨胀为重量级锁，线程挂起/唤醒 ≈ 微秒级 |

**"比 synchronized 更差"的三种情况**：
1. **高频写 + 多核**：每次写都 ping-pong，开销可能超过"一次加锁保护多个变量"。
2. **需要保护多个变量的一致性**：用 3 个 volatile 分别写，线程可能看到"部分更新"；此时一个 `synchronized` 反而更省且更正确。
3. **伪共享**：两个 volatile 变量落在同一 cache line，被两个核各写 → 10 倍级下降（见 Q14）。

**追问链**：
- 追问：怎么测？→ JMH（`@Benchmark` + `Blackhole`），注意要防 JIT 消除、防常量折叠、用多线程 state。
- 追问：怎么优化？→ 减少写次数（批量/合并）、`@Contended` 填充、改用 `LongAdder`、改用 `VarHandle.setRelease`（省掉 StoreLoad）。

---

#### Q14. 什么是伪共享？它和 `volatile` 有什么关系？

**出处**：字节、美团、阿里（性能题高频）
**考核程度**：★★★★

**答案**：
- 缓存行是 **64 字节**，是缓存同步的最小单位。
- 两个**独立变量**落在同一 cache line，被两个核**各自写** → 每次写都让对方整行失效 → 来回 RFO（cache line ping-pong）→ **性能下降 10 倍以上**。这叫**伪共享**（变量本身不共享，但缓存行被迫共享）。
- 与 volatile 的关系：**volatile 写必须让其他副本失效，所以伪共享的惩罚被放大**——普通写还能在 store buffer 里"攒一攒"，volatile 写每次都被 `lock` 强制提交，ping-pong 更剧烈。

**解法**：
```java
// ① @Contended（JDK 8+，需 -XX:-RestrictContended）
@sun.misc.Contended
static final class Cell { volatile long value; }

// ② 手动 padding（老办法）
// ③ LongAdder 的 Cell[] 分段 + @Contended
```

**追问链**：
- 追问：`LongAdder` 为什么快？→ `Cell[]` 用 `@Contended` 填充到独立 cache line，各线程 CAS 各自的 Cell，把"单点争用"变"多点并行"，`sum()` 时再汇总。
- 追问：`Disruptor` 的 `Padding` 类？→ 就是伪共享填充的经典实现（`LEFT_PADDING`/`RIGHT_PADDING` 各 7 个 long）。

---

#### Q15. `volatile` 与 `final` 的内存语义有何异同？

**出处**：阿里 P7、字节 2-2
**考核程度**：★★★★

**答案**：

| | volatile | final |
|---|---|---|
| 写语义 | release（StoreStore + StoreLoad） | 只在构造器内插 `StoreStore`（禁止 final 写外逸） |
| 读语义 | acquire | 首次读"持有该对象的引用"时插 `LoadLoad` |
| 效果 | 每次读写都保证 | **只保证"构造完成后的首次发布"** |
| 成本 | 写有 `lock` | 0 |

**核心区别**：`final` 是"**一次性安全发布**"的零成本方案，`volatile` 是"**持续可见**"方案。

**关键前提（这题真正的考点）**：
> `final` 的内存语义只在"**构造器结束后对象才被其他线程看到**"的前提下成立。若 `this` 在构造器内逸出（注册回调、启动线程、赋给静态字段），final 保证**失效**——**此时必须换成 volatile 或加锁**。

**追问链**：
- 追问：所以不变对象还需要 volatile 吗？→ 不需要（所有字段 final + 不逸出 = 安全发布）。
- 追问：`final` 字段能反射改吗？→ 可以（`setAccessible`），但会破坏语义，且 JDK 会在 final 字段上做"禁止反射修改"的优化（`Field.set` 对 final 有特殊限制）。
- 追问：`final` 数组？→ 只保证引用 final，元素可变，跟 volatile 数组一个问题。

---

#### Q16. 为什么 `volatile` 写之前插 `StoreStore`，写之后插 `StoreLoad`？（位置为什么不能反）

**出处**：美团、蚂蚁、字节（二面能让 60% 人卡住）
**考核程度**：★★★★

**答案**：因为屏障的语义是"**约束某个方向的重排**"，方向由"谁是生产者/消费者"决定。

```
volatile 写（release）的职责是"把我之前的工作都交付出去"：
   之前的一切（读、写） ── 不许越过我 ──▶ 我
   ⇒ 需要拦住"前 → 后"方向越界：
      - 前面的**写**上浮 → StoreStore 拦
      - 前面的**读**上浮 → 也被 StoreStore? 不！读是被 LoadStore 拦
```

**精确解释**（这里必须严谨）：

- `StoreStore` 管的是"**写**与**写**"的顺序。volatile 写之前若有一个普通写，必须保证普通写先对外可见 → 需要 StoreStore。
- volatile 写之前的普通**读**，JMM **不要求**它先完成（读本来就不产生"对外可见"的效果，除非它读到别人写的数据并被后续使用——那种依赖由数据依赖保证）。
- volatile 写**之后**必须插 `StoreLoad`：因为"写 → 后续读"方向的重排（StoreLoad）在 x86 上**真实存在**（store buffer），必须用全屏障拦住。

**一句话**：**屏障只拦"会导致语义错误"的方向**，不是四面都围。

**追问链**：
- 追问：那 volatile 读之后的 `LoadStore` 拦的是什么？→ 拦"读 → 后面的普通写"重排。若不拦，`if(flag){ data = x; }` 里的 `data=x` 可能被提到 `if` 之前，读到未被保护的旧数据。
- 追问：能不能所有位置都插全屏障？→ 能，但性能崩了（每次读写都 `lock`）。JMM 的"最小充分"策略就是这样推导出来的。

---

#### Q17. 单例模式有几种写法？`volatile` 分别用在哪？

**出处**：阿里、腾讯、京东、小米（一面常见，要求手写）
**考核程度**：★★★★

| 写法 | 线程安全 | 懒加载 | volatile 位置 | 缺陷 |
|---|---|---|---|---|
| 饿汉式 | ✅ | ❌ | 不需要 | 类加载即初始化，可能浪费 |
| **DCL** | ✅ | ✅ | **`instance` 字段** | 必须 volatile，否则半初始化 |
| 静态内部类 Holder | ✅ | ✅ | 不需要 | 靠 `clinit` 的类初始化锁 + JMM |
| enum | ✅ | ✅ | 不需要 | 防反射、防序列化（JVM 保证） |
| 容器式（`ConcurrentHashMap`） | ✅ | ✅ | — | 灵活但有管理成本 |

**Holder 为什么不用 volatile（高分点）**：
```java
private static class Holder { static final Singleton INSTANCE = new Singleton(); }
public static Singleton get() { return Holder.INSTANCE; }
```
JVM 对类初始化（`<clinit>`）加**类级别的初始化锁**，且 JLS §12.4.2 明确规定"初始化完成后，其他线程看到的静态字段值必须是初始化写入的值"——**类初始化本身就是一次天然的同步**，所以不需要 volatile。

**追问链**：
- 追问：DCL 能不能用 `AtomicReference` 替代 volatile？→ 可以，但没必要（`get()` 也是 volatile 读，`set` 是 CAS，更贵）。
- 追问：enum 为什么防反射？→ `Constructor.newInstance` 对 enum 有硬编码检查（`ReflectionFactory` 抛 `IllegalArgumentException`）。
- 追问：序列化会破坏单例吗？→ 会（`readObject` 会 new 新对象），需 `readResolve()`；enum 天然免疫。

---

#### Q18. 用 `volatile` 实现"读多写少"的锁优化，怎么做？有什么限制？

**出处**：美团、阿里 P7、字节
**考核程度**：★★★★

**答案（读写分离策略）**：

```java
public class CheaperLock {
    private volatile int value;                 // 读路径无锁
    public int get() { return value; }          // 无锁读
    public synchronized void set(int v) { value = v; }   // 写路径互斥
}
```

**限制（必须说出来，否则是"会背不会用"）**：
1. 只适用于"**写操作之间需要互斥，读操作只读一个变量**"；
2. 若读也需要复合操作（`getAndIncrement`），失效；
3. 若"多个变量需要整体一致"，失效（volatile 是单变量）。

**更进一步（P8 加分）**：真正的"读写锁优化"是 `StampedLock`（乐观读 + 版本戳）。volatile 只适合"单一标量"的极简场景，`StampedLock` 才是"读多写少"的正解。

---

### 三档：深水区（★★★）

#### Q19. `volatile` 与 CPU 的 store buffer、invalidate queue 是什么关系？

**出处**：字节 2-2、阿里 P7、蚂蚁 P7（这题能过 = 真懂硬件）
**考核程度**：★★★

**答案（按因果链讲）**：

1. **store buffer**：CPU 写不直接进缓存，先入 store buffer 异步合并。后果：**本核已写，别核看不见**；且本核后续读可以 store-to-load forwarding，于是"本核顺序"≠"别核顺序" → **StoreLoad 重排**。
2. **invalidate queue**：收到别核的失效消息，先入队立即 ACK，但缓存行**仍未真正失效**。后果：**别核已写，本核仍读旧值**。
3. **屏障的作用就是冲刷这两个队列**：
   - `StoreStore`/release → 等 store buffer 排空（保证"我前面的写先被看见"）；
   - `LoadLoad`/acquire → 处理 invalidate queue（保证"我后面的读取最新"）；
   - `StoreLoad`/fence → **两者都做**（`lock` 前缀即如此）。
4. **volatile 的落地**：`lock` 前缀让 store buffer 排空 + 广播失效并**等 ACK** → 一次性解决两个延迟。

**追问链**：
- 追问：MESI 不是保证一致吗，为什么还需要 store buffer？→ MESI 保证"**副本最终一致**"，但写是**异步**提交的，store buffer 是"一致性之前的缓冲"。**协议解决副本，屏障解决时序。**
- 追问：`lock` 要等所有核 ACK 吗？→ Intel SDM：`lock` 期间该 cache line 被"锁定"，其他核的访问被延迟；失效传播必须完成。**这解释了为什么高竞争时 `lock` 极慢。**

---

#### Q20. 什么是 out-of-thin-air？JSR-133 修正了什么？

**出处**：阿里 P8、字节 2-2+、学术味重的团队
**考核程度**：★★★（答出来是惊喜）

**答案**：

**旧 JMM（JSR-133 之前）的问题**：允许"因果循环"，即两个线程互相读到对方"未来才写"的值：

```java
// 初始 x = y = 0
T1: r1 = y;  x = 1;
T2: r2 = x;  y = 1;
// 旧模型允许 r1 == 1 && r2 == 1（双方都读到了"对方还没写"的值）
```

这被称为 **out-of-thin-air（凭空出现）**——值仿佛"无中生有"。

**JSR-133 的修正**：
1. 用 **happens-before + 同步顺序 + 因果性要求（causality）** 替代旧的"八操作"模型；
2. 明确规定：**一个读只能看到"在 hb 意义下允许"的写**，禁止自洽但无来源的循环依赖；
3. 副作用：`volatile` 从此被正式定义为 **synchronization action**，与 lock/unlock 同级，从而获得 release/acquire 语义。

**为什么重要**：这是"volatile 语义"的法律依据。没有 JSR-133，volatile 只有"可见性"没有"有序性"的严格定义。

**追问链**：
- 追问：那现代 JMM 完全禁止 out-of-thin-air 了吗？→ **规范上禁止，但形式化上仍有争议**（"因果性"的精确定义在学术界讨论多年，POPL/CAV 有大量论文）。工程上：**不要写依赖这种场景的代码**。
- 追问：Dekker 算法在 Java 里能工作吗？→ 用 volatile 才行；不用 volatile 会失败（就是 Q19 的 StoreLoad 重排）。

---

#### Q21. `VarHandle` 的四种访问模式有什么区别？为什么要引入它？

**出处**：字节、阿里 P7+、蚂蚁（JDK 9+ 团队必问）
**考核程度**：★★★

**答案**（见 9.1 节表）：

| 模式 | 可见性 | 有序性 | x86 落地 | 典型场景 |
|---|---|---|---|---|
| Plain | ❌ | ❌ | `mov` | 无跨线程语义 |
| Opaque | ❌（不保证跨核顺序） | ❌（但禁止编译器优化/撕裂） | `mov` + 编译屏障 | 只需"每次真读一次" |
| Acquire/Release | ✅ 单向 | ✅ 单向 | release 无 `lock`（x86） | 生产者-消费者 |
| Volatile | ✅ | ✅ 双向 | `mov` + `lock` | = volatile |

**为什么引入**：
1. **`Unsafe` 不安全且不可持续**（JDK 9 起封装，JDK 17 强封装 `sun.misc.Unsafe` 的内存访问方法）；
2. **volatile 只有一档，太粗**：很多场景只需要 acquire/release，用 volatile 白白多付 StoreLoad 的代价；
3. **VarHandle 是 `final` 静态字段 + 编译期类型检查**，可被 JIT **常量折叠**（比反射快得多）；
4. 统一支持**字段、数组元素、`ByteBuffer` 位置**的原子访问。

**追问链**：
- 追问：`setRelease` 在 x86 上真的省了吗？→ 省了 `lock`（release 不需要 StoreLoad），只剩编译屏障 → 明显更快。
- 追问：`Opaque` 到底保证什么？→ 保证"**这次访问真实发生**"（不被 JIT 消除、不撕裂），但**不给跨线程可见性/顺序**。用于"性能计数器"这类"不需要严格同步但需要真读"的场景。
- 追问：`AtomicInteger` 现在还用 Unsafe 吗？→ JDK 9+ 内部改用 `VarHandle`（`U` 字段），对外 API 不变。

---

#### Q22. C++ 的 `volatile` 和 Java 的 `volatile` 有什么区别？

**出处**：字节、华为、小米（跨语言背景候选人常被问）
**考核程度**：★★★（答错很致命）

**答案（一句话定生死）**：

> **两者语义几乎完全不同。** C++ 的 `volatile` 只保证"**不被编译器优化掉、按序访问**"，**不提供线程间可见性、不提供原子性、不参与内存序**；Java 的 `volatile` 才是**线程同步原语**（≈ C++ 的 `std::atomic` + `memory_order_seq_cst`）。

**对照表**：

| 维度 | C++ `volatile` | Java `volatile` |
|---|---|---|
| 目的 | 禁止编译器优化（MMIO、信号处理、`setjmp`） | 线程间可见性 + 有序性 |
| 跨线程可见性 | ❌ | ✅ |
| 原子性 | ❌ | 单次读写 ✅ |
| 内存序 | ❌（不插屏障） | ✅（release/acquire + StoreLoad） |
| 等价物 | — | `std::atomic<T>`（seq_cst） |
| 编译器屏障 | ✅ | ✅ |
| CPU 屏障 | ❌ | ✅ |

**加分点**：
> C++11 之后，`volatile` 在并发语境下**基本被弃用**（标准明确"volatile 不用于线程同步"），正确做法是 `std::atomic` + `memory_order`。Java 因为历史原因（JDK 1.4 之前 volatile 语义不完整，JSR-133 才补全），`volatile` 从一开始就是同步工具。

---

#### Q23. C2（JIT）对 `volatile` 放弃了哪些优化？怎么阻止循环外提？

**出处**：阿里 JVM 岗、字节、美团（JVM 方向必问）
**考核程度**：★★★

**答案（按优化器分类）**：

| C2 优化阶段 | 对 volatile 的处理 |
|---|---|
| `PhaseIdealLoop`（循环优化） | volatile `LoadNode` 的 `can_hoist` = false → **禁止循环不变量外提** |
| `PhaseCCP`/`GVN` | 带 volatile 标记的 Load 视为"有副作用"，**禁止 CSE（公共子表达式消除）** |
| `PhaseIterGVN` | 禁止"值编号"合并两次 volatile 读 |
| `PhaseMacroExpand` | 在 `StoreNode` 前插 `MemBarStoreStore`，后插 `MemBarStoreLoad` |
| 调度（`Matcher`/`Scheduling`） | `MemBar` 节点是调度屏障，**禁止指令跨它移动** |
| 死代码消除 | volatile store 视为"可观察副作用"，**禁止删除** |
| 寄存器分配 | 每次 volatile 读必须重新访存，**不能驻留寄存器** |

**"死循环现场"的精确解释**（对应 1.1）：

```
不加 volatile：
  LoadI(running) 无副作用、循环不变量 → PhaseIdealLoop 提到 preheader
  → 循环内变成 if(false/true) 常量分支 → 死循环 或 立即退出

加 volatile：
  LoadI 带 is_volatile → can_hoist = false
  → 每次迭代仍有 mov 从内存读 → 主线程一写就能看到
```

**关键源码锚点**（说出来很加分）：
- `hotspot/share/opto/memnode.cpp`：`LoadNode::depends_only_on_test()` 对 volatile 返回 false；
- `hotspot/share/opto/library_call.cpp` / `graphKit.cpp`：`insert_mem_bar(Op_MemBarVolatile)`；
- `hotspot/share/opto/loopnode.cpp`：`can_hoist` 检查 `is_volatile`。

**追问链**：
- 追问：`MemBarVolatile` 和 `MemBarStoreStore` 有什么区别？→ `MemBarVolatile` 对应全屏障（StoreLoad），`MemBarStoreStore` 只拦写写。x86 上 `MemBarStoreStore` 展开为空指令，`MemBarVolatile` 展开为 `lock addl`。
- 追问：那解释器执行时呢？→ 解释器不优化，**每次都真的访存**，所以 volatile 在 `-Xint` 模式下也生效（只是慢）。**"volatile 只是 JIT 的事"是错的。**

---

#### Q24. `volatile` 引用能保证对象内部字段的可见性吗？`ConcurrentHashMap` 怎么处理？

**出处**：阿里 P7、字节 2-2、美团 L8（并发容器必问）
**考核程度**：★★★

**答案**：

**不能（持续可见性）**。`volatile Holder h` 只保证：
- "`h` 这个引用的一次读/写"有 volatile 语义；
- `h = new Holder()` 时，构造器内的写不会被重排到引用赋值之后 → **发布安全**。

但 `h.field = 1` 之后，**别的线程读 `h.field` 不保证看到 1**（除非中间又有 volatile 读写建立 hb，或是 final 字段）。

**`ConcurrentHashMap` 的两层设计（这是标准答案）**：

```java
transient volatile Node<K,V>[] table;        // ① 引用级 volatile：保证 resize 换表可见
// ② 桶内元素：用 Unsafe/VarHandle 注入 volatile 语义
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

**为什么这么设计**：
1. `table` 引用 volatile：**resize 是整个数组替换**，必须让所有线程看见新表；
2. 元素用 `getObjectVolatile`：数组元素无法声明 volatile，只能靠 VarHandle/Unsafe **逐元素**注入；
3. `casTabAt`：桶头插入用 CAS（无锁），失败则 `synchronized` 锁桶（JDK 8 的"CAS + 锁桶"混合策略）。

**追问链**：
- 追问：为什么不用 `volatile Node[]` 就够？→ 因为"换表"和"改元素"是两件事。换表要 volatile，改元素要元素级 volatile。
- 追问：JDK 8 为什么不用 Segment 分段锁了？→ 锁粒度从"段（16 个）"降到"桶（每个 bin）"，并发度大幅提升；且引入 CAS 无锁读路径。

---

#### Q25. 如何在不加锁的情况下用 `volatile` 实现"双缓冲/快照读"？

**出处**：字节、蚂蚁、架构面
**考核程度**：★★★（能讲思路即合格）

**答案（快照替换模式）**：

```java
public class SnapshotConfig {
    private volatile Config current = Config.DEFAULT;   // 不可变对象

    public Config read() { return current; }            // 无锁快照读

    public void update(Config next) {
        current = next;                                 // 原子替换引用（volatile 写）
    }
}
```

**三个前提（缺一不可，必须说出来）**：
1. `Config` **必须不可变**（所有字段 final，构造后不改）——否则读者会看到"半更新"；
2. 替换是**整体换引用**，不是"逐字段改"；
3. 写方需要**串行化**（通常单写线程，或用 `synchronized`/`AtomicReference` 保护写）。

**为什么有效**：`volatile` 写建立 release 边 → 读者拿到新引用时，新对象的**所有字段**（构造器内写的）都已可见（hb 传递）。旧对象被 GC 回收，天然"双缓冲"。

**延伸（COS/Disruptor 的做法）**：
- **RCU（Read-Copy-Update）思想的 Java 版**：读复制、写替换、旧版本延迟回收。
- **`CopyOnWriteArrayList`**：本质相同（`volatile Object[] array` + 写时全量复制 + 替换引用），代价是写放大。
- **`StampedLock` 的乐观读**：比"快照替换"更进一步，能检测"读到一半被改了"。

---

#### Q26. `volatile` 与"内存模型的形式化"：同步顺序、因果性、提交一致性

**出处**：阿里 P8、字节 3-1、JVM 专家岗
**考核程度**：★★★（纯加分题）

**答案要点**：

1. **同步顺序（synchronization order）**：所有同步动作（volatile 读写、lock/unlock、线程 start/join、默认值写入）构成一个**全序**；每个线程内保持程序顺序。
2. **hb 的定义**：`hb = (程序顺序 ∪ 同步顺序导出的边) 的传递闭包`。
3. **一致性（consistency）**：每个读 r 看到的写 w，必须满足 `w hb r`，且不存在 `w' hb r` 且 `w hb w'`（即"看到的是 hb 意义下最新的"）。
4. **因果性（causality）**：禁止"自洽但无来源"的执行（out-of-thin-air）。JSR-133 用"提交序列（commit sequence）"的形式化：任意前缀必须能被有限步"提交"构造出来。
5. **争议**：因果性的形式化（"well-behaved execution"）在 PL 学术界长期讨论（POPL'05 Hans Boehm、CAV 等），Java 规范的表述仍是"半形式化"。

**为什么值得说**：这说明你知道"JMM 不是完美公理系统"，而是"工程妥协的规范"。**面试官听到这层会认为你有研究能力，而不是背八股。**

---

#### Q27. 高并发下 `volatile` 写为什么会成为瓶颈？怎么优化？

**出处**：字节、快手、滴滴（性能优化岗）
**考核程度**：★★★

**答案（按开销来源拆解）**：

| 开销来源 | 机制 | 优化手段 |
|---|---|---|
| store buffer 排空 | 每次写等所有未提交的写落地 | 批量写、合并写、减少写次数 |
| cache line 失效广播 + 等 ACK | `lock` 要求所有核确认 | 减少共享变量的核数（线程本地聚合） |
| cache line ping-pong | 多核争同一行 | `@Contended` 填充、`LongAdder` 分段 |
| 流水线序列化点 | `lock` 阻碍指令重叠 | 改用 `setRelease`（省 StoreLoad） |
| 内存序过强 | 用了不需要的 volatile | 降级为 `Opaque`/`Acquire-Release` |

**实测建议**：先用 JMH 测出真实开销，再决定"降级内存序"还是"换数据结构"。

---

### 四档：加分/造火箭（★★ / ☆）

#### Q28. `volatile` 在 ARM 上是怎么实现的？和 x86 有什么本质差异？

**考核程度**：★★

**答案**：见 5.4 节。核心差异：

| | x86 (TSO) | ARM (Weak) |
|---|---|---|
| 硬件允许的重排 | 仅 StoreLoad | 全部四种 |
| volatile 读 | 普通 `mov`（0 开销） | `ldar` / `dmb ishld`（有开销） |
| volatile 写 | `lock addl` | `stlr` / `dmb ish` |
| `OrderAccess` 实现 | 4 个空 + 1 个 `lock` | 全部非空 |

**本质**：**JMM 只承诺语义，不承诺性能特征**。同一段代码在 ARM 上"volatile 读也要钱"，这是"弱内存模型 + JMM 强制 acquire"的必然结果。

**延伸**：这也是为什么 JDK 在 ARM 服务器上"看起来更保守"——它要为每次 volatile 读付 `ldar` 的成本。

---

#### Q29. 如果让你证明"volatile 真的生效了"，你会怎么做？

**考核程度**：★★（考察工程实证能力）

**三种手段**：

**① 汇编级验证**（最硬核）：
```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*Bar.write Bar
# 期望看到 lock addl $0x0,(%rsp)
```
或用 JMH 的 `-prof perfasm`。

**② 行为级验证**（写"死亡测试"）：
```java
// 不加 volatile：大概率死循环（-Xint 下不会，要看 JIT）
// 加 volatile：100% 退出
```

**③ 压力测试 + 断言**：
```java
// 用 JCStress（OpenJDK 官方并发测试框架）
@JCStressTest
@Outcome(id = "1, 1", expect = ACCEPTABLE, desc = "见对方写")
@Outcome(id = "0, 0", expect = FORBIDDEN, desc = "Dekker 场景，volatile 下不允许")
```

**追问**：为什么 `-Xint` 下不加 volatile 也可能退出？→ 解释器不优化、每次都真的访存，缓存还没"热"到坏的程度；**问题在 JIT 编译后爆发**。这解释了"为什么本地测不出来、线上偶发"。

---

#### Q30. `volatile` 能不能实现"无锁的生产者-消费者"？

**考核程度**：★★（架构思维题）

**答案**：**能，但只能实现"单生产者-单消费者（SPSC）"的简化版**，多生产者多消费者不行。

**SPSC 环形的核心**（Disruptor 的极简版）：

```java
// 单生产者写、单消费者读，靠 volatile 的 head/tail 建立 hb
private volatile long tail = 0;   // 生产者写
private volatile long head = 0;   // 消费者写
private final long[] buffer = new long[CAP];   // 元素本身不需要 volatile（靠 hb 保证）

// 生产者
void publish(long v) {
    long t = tail;
    buffer[(int)(t & (CAP-1))] = v;   // ① 先写数据（普通写）
    tail = t + 1;                     // ② 再发布（volatile 写 → release）
}
// 消费者
long consume() {
    if (head == tail) return EMPTY;   // volatile 读
    long v = buffer[(int)(head & (CAP-1))];  // ③ 读数据（靠 hb 传递看到 ①）
    head = head + 1;
    return v;
}
```

**为什么元素不用 volatile**：靠 `tail` 的 release + `tail` 读的 acquire 建立 hb 传递，**数据写在 hb 边之前 → 必然可见**。这就是"**用一条 volatile 变量保护一批普通变量**"的经典技巧（Disruptor/Netty 都这么干）。

**限制**：
- **单生产者单消费者**才安全（多生产者需要 CAS 抢序号）；
- 缓冲区容量要够，否则要处理"满/空"；
- 伪共享要用 padding 隔离 head/tail。

**追问**：多生产者怎么办？→ `AtomicLong`/CAS 抢 `tail` + `Sequence` 屏障（Disruptor 的做法），或者干脆用 `BlockingQueue`。

---

## 11. 一页速查（面试前 5 分钟看这个）

### 11.1 关键数字

| 项 | 值 |
|---|---|
| 缓存行大小 | 64 字节（x86/ARM 常见） |
| L1 / L2 / L3 / 内存延迟 | ~4 / ~12 / ~30-70 / ~150-300 cycle |
| `lock` 前缀开销 | ~20-50 cycle 起，高竞争可达数百 |
| volatile 读（x86） | ≈ 普通读，**0 额外** |
| volatile 写（x86） | ≈ 普通写 + `lock` |

### 11.2 关键指令

| 平台 | volatile 写 | volatile 读 | 全屏障 |
|---|---|---|---|
| x86-64 | `mov` + `lock addl $0x0,(%rsp)` | `mov` | `lock` / `mfence` |
| aarch64 | `stlr` / `dmb ish` | `ldar` / `dmb ishld` | `dmb ish` |

### 11.3 关键类/字段/方法

| 项 | 值 |
|---|---|
| Class 文件标志 | `ACC_VOLATILE`（0x0040） |
| 访问 API | `Access`/`Unsafe.getXxxVolatile`/`putXxxVolatile` |
| HotSpot 原语 | `OrderAccess::acquire/release/fence` |
| C2 节点 | `LoadNode`(is_volatile) / `StoreNode` / `MemBarStoreStore` / `MemBarVolatile` |
| JMM 规范 | JLS §17.4（JMM）、§17.4.5（hb）、§17.7（volatile long/double） |
| 历史文档 | JSR-133（JMM 修订，2004） |

### 11.4 三句话答题模板

1. **"volatile 保证可见性和有序性，不保证原子性"** + 补一句"单次读写原子，复合操作不安全"。
2. **"有序性靠内存屏障"** + 能说出四个位置（StoreStore / StoreLoad / LoadLoad / LoadStore）。
3. **"x86 上落地为 lock 前缀"** + 能说出"读无屏障因为 TSO，写要 lock 因为 StoreLoad"。

### 11.5 六个"能立刻拉开差距"的杀手句

1. "x86 的 `OrderAccess` 里四个屏障是空实现，但**编译屏障依然存在**。"
2. "`lock addl $0x0,(%rsp)` 里的 `+0` 没有业务含义，只是给 `lock` 找一个最便宜的读改写载体。"
3. "`volatile` 是 **release/acquire**，不是简单的'刷缓存'；它的价值在 **hb 传递性**。"
4. "`long` 的 volatile 读写规范强制原子（JLS §17.7），但 `long++` 依然不原子。"
5. "`final` 的零成本安全发布有个前提：**`this` 不能在构造器里逸出**。"
6. "JSR-133 修正了旧 JMM 的 **out-of-thin-air** 问题，volatile 从此成为 **synchronization action**。"

---

## 12. 结语：把 volatile 放回它的位置

`volatile` 不是"半个 synchronized"，而是**一套精确的、可组合的内存序原语**。它的能力边界非常清晰：

```
它解决的问题：      一个线程的写，如何被另一个线程可靠地看到；
                    编译器和 CPU 的重排，如何不破坏这种可靠性。
它不解决的问题：    多个操作的不可分割性；
                    多个变量的一致快照；
                    竞争下的性能（它甚至可能加剧竞争）。
```

**把这三层吃透，`volatile` 就不再是一句口诀**：

```
语义层（JMM）     →  写=release，读=acquire，靠 hb 传递
编译层（C2）      →  禁止 hoisting/CSE/死存储消除，插 MemBar 节点
硬件层（CPU）     →  x86 lock 前缀 / ARM dmb+stlr，配合 MESI 与 store buffer
```

> **最后一句**：能讲清"为什么 x86 上 volatile 读不要屏障、写要一条 `lock`，而 ARM 上两者都要"，并且能解释"这不是 JMM 的要求，而是平台的实现选择"——这个水平，足够应对国内绝大多数大厂的 Java 高级岗面试。

---

*本文档基于 HotSpot（JDK 8 ~ JDK 21）与 x86-64 / aarch64 平台整理；具体指令序列以 `-XX:+PrintAssembly` 实测为准（不同 JDK 版本与 C2 优化阶段可能有细微差异）。*
