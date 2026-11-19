# synchronized 底层原理深度解析 & 大厂面试题库

> 面向人群：想真正搞懂 `synchronized` 而不是背结论的 Java 工程师 / 面试冲刺者
> 阅读建议：先读「第 0 章 学习地图」，按四层结构自下而上理解；面试题部分按 ★ 等级挑着刷。
> 重要前提：文中所有热点实现细节都以 **HotSpot（JDK 8 ~ JDK 21）** 为准，OpenJ9 / GraalVM 的实现路径不同，面试时如果被追问要说明前提。

---

## 目录

- [第 0 章 学习地图：把 synchronized 拆成四层](#第-0-章-学习地图把-synchronized-拆成四层)
- [第 1 章 语法层与字节码层](#第-1-章-语法层与字节码层)
  - [1.1 三种用法与锁对象](#11-三种用法与锁对象)
  - [1.2 同步代码块：monitorenter / monitorexit](#12-同步代码块monitorenter--monitorexit)
  - [1.3 同步方法：ACC_SYNCHRONIZED](#13-同步方法acc_synchronized)
  - [1.4 异常路径：为什么必须有两个 monitorexit](#14-异常路径为什么必须有两个-monitorexit)
- [第 2 章 对象内存布局与 Mark Word](#第-2-章-对象内存布局与-mark-word)
  - [2.1 对象在堆里的样子](#21-对象在堆里的样子)
  - [2.2 Mark Word 的位布局（锁状态的"寄存器"）](#22-mark-word-的位布局锁状态的寄存器)
  - [2.3 identityHashCode 在哪个位置？为什么它和偏向锁互斥](#23-identityhashcode-在哪个位置为什么它和偏向锁互斥)
- [第 3 章 锁的四种状态与升级全链路](#第-3-章-锁的四种状态与升级全链路)
  - [3.1 全景图](#31-全景图)
  - [3.2 偏向锁：原理、撤销、批量重偏向与 epoch](#32-偏向锁原理撤销批量重偏向与-epoch)
  - [3.3 轻量级锁：Lock Record 与 CAS](#33-轻量级锁lock-record-与-cas)
  - [3.4 重量级锁：ObjectMonitor 全解析](#34-重量级锁objectmonitor-全解析)
  - [3.5 三种锁的对比表](#35-三种锁的对比表)
  - [3.6 为什么锁只能升级不能降级](#36-为什么锁只能升级不能降级)
- [第 4 章 编译期优化：自旋、锁消除、锁粗化](#第-4-章-编译期优化自旋锁消除锁粗化)
- [第 5 章 内存语义：happens-before 与内存屏障](#第-5-章-内存语义happens-before-与内存屏障)
- [第 6 章 硬件层：CAS、缓存一致性、park/unpark](#第-6-章-硬件层cas缓存一致性parkunpark)
- [第 7 章 wait / notify 机制](#第-7-章-wait--notify-机制)
- [第 8 章 JDK 版本演进与虚拟线程的坑](#第-8-章-jdk-版本演进与虚拟线程的坑)
- [第 9 章 大厂面试题库（按难度分级）](#第-9-章-大厂面试题库按难度分级)
  - [L1 基础必答（★~★★）](#l1-基础必答)
  - [L2 中级高频（★★★）](#l2-中级高频)
  - [L3 高级深挖（★★★★）](#l3-高级深挖)
  - [L4 专家级 / 细节杀手（★★★★★）](#l4-专家级--细节杀手)
  - [L5 场景设计与开放题](#l5-场景设计与开放题)
- [第 10 章 面试现场：追问链路与答题模板](#第-10-章-面试现场追问链路与答题模板)
- [第 11 章 速记卡与自测清单](#第-11-章-速记卡与自测清单)

---

## 第 0 章 学习地图：把 synchronized 拆成四层

很多人背了"偏向锁→轻量级锁→重量级锁"就以为懂了，但面试官只要往下问一层就会露馅。原因是 `synchronized` 从来不是一个东西，它是**四层机制叠起来的**：

```
 +---------------------------------------------------------------------+
 |  L1  Java syntax   : synchronized method / synchronized block        |
 +---------------------------------------------------------------------+
 |  L2  Bytecode      : monitorenter / monitorexit / ACC_SYNCHRONIZED   |
 +---------------------------------------------------------------------+
 |  L3  JVM runtime   : Mark Word state machine + ObjectMonitor         |
 +---------------------------------------------------------------------+
 |  L4  Hardware/OS   : CAS(lock cmpxchg) / MESI / futex(park)          |
 +---------------------------------------------------------------------+
```

| 层 | 它负责回答的面试问题 | 本章位置 |
|---|---|---|
| L1 Java 语法 | 锁的是哪个对象？实例锁和类锁互斥吗？可重入吗？ | 第 1.1 节 |
| L2 字节码 | `monitorenter` 做了什么？异常时锁怎么释放？ | 第 1.2 ~ 1.4 节 |
| L3 JVM 运行时 | Mark Word 长什么样？锁怎么升级？Monitor 里有什么？ | 第 2 ~ 4、7 章 |
| L4 硬件 / OS | CAS 为什么原子？park 为什么慢？内存屏障在哪？ | 第 5 ~ 6 章 |

一次 `synchronized(obj){...}` 的完整生命周期（后面每章都在填这张图的细节）：

```
 source          javac           class file             interpreter/JIT        CPU
 ------------------------------------------------------------------------------
 synchronized -> monitorenter -> InterpreterRuntime   -> lock cmpxchg
 (obj) { ... }    ...             ::monitorenter()        / mfence (release)
                  monitorexit    ObjectSynchronizer      / park (contended)
                  (+exception    ::fast_enter
                    table)       ::slow_enter
```

**一条必须刻进脑子的主线**：`synchronized` 的所有魔法都围绕一件事——**在对象头的 Mark Word 里用 2 个 bit 记录"这个对象当前被谁、以什么方式锁着"**。锁升级 = 改写这 2+ 位的过程；锁膨胀 = 从"Mark Word 直接存线程指针/LockRecord 指针"退化成"Mark Word 只存一个 ObjectMonitor 指针"。

---

## 第 1 章 语法层与字节码层

### 1.1 三种用法与锁对象

```java
// 用法一：同步实例方法 —— 锁的是 this（当前实例对象）
public synchronized void m1() { ... }

// 用法二：同步静态方法 —— 锁的是 Class 对象（XXX.class，是"类锁"）
public static synchronized void m2() { ... }

// 用法三：同步代码块 —— 锁的是括号里那个对象的引用
public void m3() {
    Object o = this.lock;          // 常见：锁一个专用 final 对象
    synchronized (o) { ... }
}
// 变体：synchronized(this) / synchronized(XXX.class) / synchronized("常量")
```

| 写法 | 锁对象 | 影响范围 |
|---|---|---|
| `synchronized` 修饰实例方法 | `this` 实例 | 同一个实例的**所有**同步实例方法互斥 |
| `synchronized` 修饰静态方法 | `Class` 对象（`Xxx.class`） | 整个 JVM 内该类**所有**同步静态方法互斥 |
| `synchronized(this)` | 当前实例 | 与实例同步方法**共用一个锁** |
| `synchronized(Xxx.class)` | Class 对象 | 与同步静态方法**共用一个锁** |
| `synchronized(new Object())` | 每次都是新对象 | **等于没锁**（锁对象不共享，最常见错误） |

> ⚠️ 两个必须记住的结论：
> 1. **锁是对象身上的，不是代码块身上的。** 所以"给方法加 synchronized 就安全"是错的——只有拿到同一个对象锁的线程才互斥。
> 2. **实例锁和类锁是两个不同的锁，彼此不互斥。** 一个线程执行 `synchronized` 静态方法、另一个线程执行同一类的 `synchronized` 实例方法，可以同时进行。

### 1.2 同步代码块：monitorenter / monitorexit

```java
public void syncBlock() {
    synchronized (this) {
        count++;
    }
}
```

`javap -v` 出来的字节码（去掉常量池部分）：

```
  public void syncBlock();
    Code:
       0: aload_0                // 把 this 压栈
       1: dup                    // 复制一份，monitorenter 会"消耗"一个引用
       2: astore_1               // 存到局部变量槽 1，给 monitorexit 用
       3: monitorenter           // ★ 加锁
       4: aload_0
       5: dup
       6: getfield      #2       // count
       9: iconst_1
      10: iadd
      11: putfield      #2
      14: aload_1
      15: monitorexit            // ★ 正常路径解锁
      16: goto          24
      19: astore_2               // ← 异常处理器入口
      20: aload_1
      21: monitorexit            // ★ 异常路径解锁
      22: aload_2
      23: athrow                 // 把异常重新抛出
      24: return
    Exception table:
       from    to  target type
           4    16    19   any   // ★ 任何异常都跳到 19，保证 unlock
```

三个关键点：

1. **`monitorenter` 是"弹出一个引用并对其加锁"**。所以编译器要先 `dup` 一份：一份给 enter，一份留给 exit（存在局部变量里）。
2. **`monitorenter` 和 `monitorexit` 必须成对**，且 JVM 规范要求**任何路径**上 enter 与 exit 数量相等，否则抛出 `IllegalMonitorStateException`（在方法异常返回时由 JVM 检查）。
3. **异常表类型是 `any`**——这不是 Java 里的 `Exception`，而是"任意 Throwable"，用来保证死锁/锁泄漏不会因为业务异常发生。

### 1.3 同步方法：ACC_SYNCHRONIZED

同步方法**没有** `monitorenter` 指令，而是靠方法访问标志：

```
  public static synchronized void m2();
    flags: (0x0029) ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
```

调用时的处理是隐式的：JVM 在 `Method::invoke` 时检测到 `ACC_SYNCHRONIZED`，就自动执行 `monitorenter`（相当于隐含的锁动作），返回时自动 `monitorexit`（含异常返回）。热点的实现是 `InterpreterRuntime::monitorenter` 被调用（`JVM_ENTRY` 里 `synchronized` 检查），等价于：

```cpp
// 简化示意
if (method->is_synchronized()) {
    Handle h_obj = is_static ? method->method_holder()->java_mirror() : h_recv;
    ObjectSynchronizer::fast_enter(h_obj, &lock, false, CHECK);
}
```

> 💡 面试里可以这么说：**"修饰代码块是显式指令，修饰方法是访问标志 + JVM 隐式加锁，二者最终都收敛到 `ObjectSynchronizer::fast_enter`"**——一句话就说明了你懂字节码层和运行时层是接得上的。

### 1.4 异常路径：为什么必须有两个 monitorexit

因为一个 monitor 的 enter/exit 计数必须平衡。如果只在正常路径 exit，一旦 `count++` 抛异常，锁就永久泄漏，其它线程永远阻塞。

JVM 规范对此有明确要求（`monitorenter` 的语义 + 结构化锁），并且**字节码校验器**会检查：对于每个 `monitorenter`，在方法每一条退出路径上都必须有对应的 `monitorexit`。这也是为什么 javac 会生成"正常一条 + 异常一条 + athrow"的三段式结构。

延伸：这也是 `synchronized` 相比手写 `lock.lock() / finally unlock()` 更"安全"的地方——**它不可能忘记释放**。

---

## 第 2 章 对象内存布局与 Mark Word

### 2.1 对象在堆里的样子

一个 Java 对象在堆里由**三段**构成：

| 区段 | 内容 | 大小（64 位） |
|---|---|---|
| **对象头 · Mark Word** | 哈希码 / GC 分代年龄 / 锁状态 / 偏向线程 ID（**复用同一块 8 字节**） | 8 字节 |
| **对象头 · Klass Pointer** | 指向方法区的 `InstanceKlass`，用来找到类元信息 | 4 字节（开压缩指针）/ 8 字节 |
| **对象头 · 数组长度** | 仅数组对象有 | 4 字节 |
| **实例数据 Instance Data** | 业务字段，按 long/double → int/float → short/char → byte/boolean → 引用 顺序排列；**父类字段在前，子类在后** | 不定 |
| **对齐填充 Padding** | 补齐到 8 字节整数倍（HotSpot 要求对象起始地址 8 字节对齐） | 0~7 字节 |

```
             Java 对象在堆中的布局（从低地址到高地址）
+---------------------------+  <- 对象起始地址（8 字节对齐）
|  Mark Word  (8 bytes)     |   <- 锁的一切状态都在这 8 个字节里
+---------------------------+
|  Klass Ptr  (4 or 8)      |   <- 指向 InstanceKlass
+---------------------------+
|  [array length (4)]       |   <- 仅数组对象存在
+---------------------------+
|  Instance Fields          |   <- 字段，含父类继承来的
|  ...                      |
+---------------------------+
|  Padding (0~7 bytes)      |   <- 补齐 8 字节倍数
+---------------------------+
```

- 开启指针压缩（`-XX:+UseCompressedOops`，64 位 JVM 默认开、堆 < 32G 时生效）后，对象头 = **12 字节**，不开启 = **16 字节**。
- 常用工具验证：`org.openjdk.jol:jol-core` 的 `ClassLayout.parseInstance(obj).toPrintable()`。
- `-XX:+UseCompressedClassPointers` 控制 Klass Pointer 是否压缩（与 Oops 压缩是两个开关）。

一个 `new Object()` 在 64 位 + 压缩指针下占 **16 字节**（12 字节头 + 4 字节填充），这就是为什么"大量小对象很吃内存"。

### 2.2 Mark Word 的位布局（锁状态的"寄存器"）

Mark Word 是一个 **8 字节（64 bit）的复用字段**——同一块内存，在不同锁状态下含义完全不同。这是理解 `synchronized` 的钥匙。

**核心直觉（64 位 JVM / JDK 8，带偏向锁）：同一块 8 字节，"预算"是固定的（64 bit），装了这个就装不下那个。**

```
NORMAL : [ unused 25 ][          hash 31          ][u1][age4][b0][01]
BIASED : [           thread ID 54           ][ep 2][u1][age4][b1][01]
THIN   : [                  LockRecord ptr 62                   ][00]
FAT    : [                 ObjectMonitor ptr 62                 ][10]
```
> NORMAL=无锁，BIASED=偏向锁，THIN=轻量级锁，FAT=重量级锁；`u1`=1 bit 未用，`age4`=GC 分代年龄，`b?`=biased_lock 位。
> 从这张图能直接读出两个"反直觉"结论：
> 1. **hash 与偏向锁的 thread ID 抢同一片空间 → 二者互斥**（详见 2.3 与 Q23）；
> 2. **指针只用 62 bit**（低 2 bit 被 lock 位占用）→ 对象地址必须 8 字节对齐。

**四种状态对照（64 位 JVM / JDK 8，按 bit 从高到低）：**

| 状态 | lock 位 | biased_lock | 其余 62 bit 装什么 |
|---|---|---|---|
| 无锁 Normal | `01` | 0 | unused(25) + **identity_hash(31)** + unused(1) + age(4) |
| 偏向锁 Biased | `01` | 1 | **thread ID(54)** + **epoch(2)** + unused(1) + age(4) |
| 轻量级锁 Thin Lock | `00` | — | 指向线程栈 **Lock Record** 的指针(62) |
| 重量级锁 Fat Lock | `10` | — | 指向堆中 **ObjectMonitor** 的指针(62) |
| GC 标记 | `11` | — | 空 / 转发指针 |

> 从这张图能直接读出两个"反直觉"结论：
> 1. **hash 和偏向锁共用同 31 bit 空间 → 二者互斥**（见 2.3 与 Q23）；
> 2. **轻量级锁/重量级锁的指针只用 62 bit**（低 2 bit 被 lock 位占了），所以对象地址也要求 8 字节对齐。

**32 位 JVM 的差异**（只记差异，面试常追问）：

| 状态 | 32 位布局 |
|---|---|
| 普通 | unused(1) + hash(25) + age(4) + biased_lock(1) + lock(2) |
| 偏向 | thread(23) + epoch(2) + age(4) + biased_lock(1) + lock(2) |
| 轻量级 | ptr_to_lock_record(30) + lock(2) |
| 重量级 | ptr_to_heavyweight_monitor(30) + lock(2) |

**判断口诀**：看最低 2 bit。

| lock 位 | biased_lock | 状态 |
|---|---|---|
| `01` | 0 | 无锁（Normal） |
| `01` | 1 | 偏向锁（Biased） |
| `00` | - | 轻量级锁（Thin Lock） |
| `10` | - | 重量级锁（Fat Lock / Inflated） |
| `11` | - | GC 标记 |

> 🔍 实测小技巧：`hashCode()` 默认是"懒生成"的，第一次调用才写进 Mark Word。所以 `print(ClassLayout)` 时如果看到 `hashcode: 0` 且 `biased_lock: 1`，说明还是可偏向状态；调用 `System.identityHashCode(obj)` 之后会变成 `biased_lock: 0`。

### 2.3 identityHashCode 在哪个位置？为什么它和偏向锁互斥

这是**最容易被问倒的细节**：

- 哈希码占 31 bit，而偏向锁状态下这 31 bit 被 `thread ID(54)` 覆盖了 → **一块内存不能同时装两个东西**。
- 因此：**一个对象一旦计算过 identity hash code，就永久失去偏向能力**（`markOop::has_bias_pattern()` 判定失败），后续加锁直接从"无锁/轻量级锁"路径走。
- 反过来，如果对象正在偏向状态，此时调用 `System.identityHashCode()`，会**撤销偏向锁**（`BiasedLocking::revoke`，需要 safepoint），然后把 hash 写进 Mark Word。注意 `Object.hashCode()` 如果是被重写的业务 hashCode 则不影响，只有 `System.identityHashCode` / 默认 `Object.hashCode` 才用到这个字段。
- 所以"用了 HashMap/HashSet 存对象"在极端情况下会影响锁性能，因为 `hashCode()` 让对象不可偏向了。这是很反直觉但真实存在的关联。

延伸：GC 分代年龄也是从 Mark Word 的 4 bit 读的，最大 15，这就是 `-XX:MaxTenuringThreshold` 上限是 15 的原因（CMS 之后改用另外的计数，但默认 Parallel/Serial 仍走这里）。

---

## 第 3 章 锁的四种状态与升级全链路

### 3.1 全景图

```
         +----------------------+
         | UNLOCKED (Normal)    |
         | new Object()         |
         +----------+-----------+
                    |
                    |  1st thread enters (biased locking on, no identityHashCode yet)
                    v
+------->+----------------------+
|        | BIASED               |        lock=01, bias=1
|        | MW = thread ID       |        zero CAS on reentry, no monitor
|        +----------+-----------+
|                   |
|                   |  2nd thread contends / identityHashCode / wait() / revoke
|                   v
|        +----------------------+
|        | THIN (Lightweight)   |        lock=00
|        | MW = LockRecord ptr  |        CAS only, no block, no kernel
|        +----------+-----------+
|                   |
|                   |  CAS restore failed / spin failed / waiters present
|                   v
|        +----------------------+
+--------| FAT (Heavyweight)    |        lock=10
  bulk   | MW = ObjectMonitor   |        park/unpark, kernel, thread suspended
 rebias  +----------------------+
```

> 图例：`UNLOCKED` = 无锁，`BIASED` = 偏向锁，`THIN` = 轻量级锁，`FAT` = 重量级锁；`MW` = Mark Word，`bulk rebias` = 批量重偏向（只回退到"可重新偏向"，不是真正降级）。

**四条升级触发线，按版本和场景各不同，务必分类记忆**：

| 从 | 到 | 触发条件 |
|---|---|---|
| 无锁 | 偏向锁 | 第一个线程进入同步块（JDK 6~14 默认开启；JDK 15+ 默认关闭） |
| 偏向锁 | 轻量级锁 | 另一个线程尝试获取；调用 `identityHashCode`；调用 `wait()`（wait 强制要求 monitor）；偏向线程已退出/已死需要撤销 |
| 偏向锁 | 重量级锁 | 撤销时原线程仍在同步块内，且新线程 CAS 抢不到 / 达 safepoint 后仍冲突 |
| 轻量级锁 | 重量级锁 | 解锁时 CAS 恢复 Mark Word 失败（说明有第三方线程进来过）；或加锁时 CAS 失败且自旋仍失败 |

### 3.2 偏向锁：原理、撤销、批量重偏向与 epoch

#### 3.2.1 它要解决什么问题

统计事实：绝大多数 `synchronized` 块**同一时刻只有一个线程访问**（例如单线程环境、或锁的临界区被同一个线程反复进出）。这种情况下用 CAS 加锁/解锁本身就是浪费——每次都要一条 `lock cmpxchg`（会锁缓存行）。

偏向锁的思路极端偷懒：**"这对象我承包了，把线程 ID 写进对象头，以后我进来连 CAS 都不做。"**

#### 3.2.2 加锁流程（`BiasedLocking::revoke_and_rebias`）

```cpp
// 简化示意，对应 hotspot/src/share/vm/runtime/synchronizer.cpp
if (mark->has_bias_pattern()) {
    if (mark->biased_locker() == Self) {
        // 情况1：就是我自己的偏向 → 直接成功，什么都不用做（零成本！）
        return true;
    }
    if (mark->biased_locker() == NULL && !mark->is_biased_anonymously()) {
        // 情况2：匿名偏向（刚创建、还没偏向任何人）→ CAS 写入自己的线程 ID
        if (CAS(mark, biased_to_self)) return true;
    }
    // 情况3：别人偏向了 / epoch 不匹配 → 走撤销流程（需要 safepoint）
    return revoke_and_rebias(...);
}
// 不可偏向的对象 → 走 fast_enter / slow_enter
```

关键：**第一次偏向也只需要一次 CAS**，之后同线程重入是 **0 次原子操作**、0 次内核调用。这是偏向锁最诱人的地方。

#### 3.2.3 撤销（Revocation）：为什么代价高

撤销分两种情况：

| 场景 | 处理 |
|---|---|
| 偏向线程**已死亡**，或已不在该同步块内 | 直接把 Mark Word 恢复成无锁（`markOop::set_unbiased`），代价低 |
| 偏向线程**仍持有**（还在同步块里） | 必须**在该线程栈上伪造一条 Lock Record**（`biased_locker` 的栈帧里塞一个 displaced header = 原 Mark Word），把 Mark Word 改成指向它 → 变成轻量级锁；然后新线程再去 CAS 抢 |

而**"别人偏向了"这个事实是被动发现的**：原线程不可能主动通知 JVM。所以 JVM 采取的策略是：

> **在安全点（safepoint）批量处理撤销请求**——所有线程停在安全点，JVM 遍历被标记的 JavaThread，找到持有偏向的对象并就地撤销。这就是为什么文档里说"偏向锁撤销需要 STW"。

这里有个**必须说清的细节**（面试加分点）：撤销本身是通过 `VM_RevokeBias` 这个 VM operation 在 safepoint 执行的，但它只让**被撤销的那个线程**停顿处理，而不是"整个 JVM 长时间 STW"。不过在高并发撤销场景下，频繁进入 safepoint 会造成明显的 STW 抖动——这才是"偏向锁在高竞争场景是负优化"的真实原因。

#### 3.2.4 批量重偏向（Bulk Rebias）与批量撤销（Bulk Revoke）

前提概念：**epoch（2 bit）** 是 Mark Word 里的"偏向有效期时间戳"，同时**每个类（Klass）也存一份 epoch**（`_prototype_header`）。

撤销计数记在类上（`BiasedLockingCounters`）：

| 阈值（默认值） | 行为 | 效果 |
|---|---|---|
| 单个类撤销 ≥ **20** 次（`BiasedLockingBulkRebiasThreshold`） | **批量重偏向**：把该类的 epoch +1，并遍历该类的所有实例把 Mark Word 的 epoch 更新；旧偏向的 epoch 与新 epoch 不匹配 → 之前的偏向全部作废，但**偏向能力保留** | 新线程可以重新偏向，避免每个对象都撤销 |
| 单个类撤销 ≥ **40** 次（`BiasedLockingBulkRevokeThreshold`） | **批量撤销**：把该类标记为**不可偏向**（`_prototype_header` 去掉 bias 位），该类今后所有对象都从无锁走起；同时把所有实例的 Mark Word 直接恢复成无锁 | 彻底放弃这类对象上的偏向 |
| 全局撤销 ≥ `BiasedLockingDecayTime`(25s) 内超过阈值 | 同样触发批量撤销（针对整体） | 防止全 JVM 抖动 |

典型触发场景：**一个类被大量创建，然后被不同线程依次锁一下（比如线程池里每个任务 new 一个对象加锁）**——这时偏向锁几乎 100% 会被撤销，JVM 会在 20 次后批量重偏向，40 次后彻底禁用该类偏向。这是"偏向锁在某些场景下纯亏"的量化解释。

#### 3.2.5 为什么 JDK 15 默认关闭、JDK 18 移除

1. **撤销依赖 safepoint**，在高并发下造成 STW 抖动，收益 < 成本。
2. **现代应用几乎都是多线程**（线程池、并发框架），"单线程独占"的假设在真实负载里很难成立。
3. **偏向锁让对象头语义复杂**，阻碍了其它优化（如 Lilliput 项目要把对象头压缩到 64 bit 甚至更小、Valhalla 值类型）。
4. 维护成本高：偏向锁与 `identityHashCode`、GC、JIT 的交互极其难测。

JEP 374（JDK 15）标记为 Deprecated 并默认关闭（`-XX:-UseBiasedLocking` 成为默认，需要显式 `+UseBiasedLocking` 才开）；JDK 18 起**代码被移除**（相关 `-XX:BiasedLocking*` 参数直接报"Unrecognized VM option"）。

> 🎯 面试应答："现在（JDK 17/21）默认已经没有偏向锁了，对象从无锁直接走轻量级锁。但面试官问原理，我依然要说清它，因为它是理解 Mark Word 复用的最好例子。" —— 这句话能直接证明你不是背 2015 年的博客。

### 3.3 轻量级锁：Lock Record 与 CAS

#### 3.3.1 核心数据结构：BasicLock / Lock Record

```
== Java 线程栈（某个栈帧内）==========================================
   ...
   BasicObjectLock            <- 这就是 "Lock Record"
     |-- _displaced_header --> 原本的 8 字节 Mark Word
     |                         (若为 NULL，表示这一层是"重入层")
     |-- _obj              --> 被锁对象的 oop（用来知道给谁解锁）
   ...

== 堆上的对象 =======================================================
   Object
     |-- Mark Word (lock=00) --> 指向上面那个 Lock Record 的地址
     +-- Klass Pointer
```

> 关键：**"锁"的信息被拆成两半**——对象头只留一个"指针"，真正的原值（displaced header）存在线程栈上。所以轻量级锁是"**栈上备份 + 对象头指针互换**"。

要点：**Lock Record 分配在线程自己的栈帧上**（不是堆），每个 `synchronized` 块最多 N 个（N = 该线程最大嵌套重入深度，JIT 会静态算出来 `max_monitors`）。

#### 3.3.2 加锁（`ObjectSynchronizer::slow_enter` 的 trivial 分支）

```cpp
markOop mark = obj->mark();
if (mark->is_neutral()) {                       // 无锁
    lock->set_displaced_header(mark);            // ① 先把原 Mark Word 备份到 Lock Record
    if (mark == CAS(&obj->mark, mark, lock)) {   // ② CAS：Mark Word ← Lock Record 地址 + 00
        return;                                  //    成功 = 拿到轻量级锁
    }
    // CAS 失败 → 有竞争 → 膨胀
}
```

注意：这里保存的是 **markOop 指针值**（在 HotSpot 里 Mark Word 用 `markOop` 表示，实际是 8 字节内容拷贝），不是"复制到堆"。所以轻量级锁的"加锁"本质是**把对象头的 8 字节挪到线程栈上，再用一个指针指回来**。

#### 3.3.3 重入怎么办？——displaced_header 置 NULL 的巧思

```java
synchronized (this) {      // 第 1 层：LockRecord1.displaced_header = 原MarkWord
    synchronized (this) {  // 第 2 层：LockRecord2.displaced_header = NULL  ← 关键！
        ...
    }
}
```

- 重入时 **Mark Word 已经是 `lock=00` 了**，不能再 CAS（否则会把"指向 LockRecord1 的指针"覆盖成"指向 LockRecord2 的指针"，破坏结构）。
- 实现方式：再加一个 Lock Record，但 `_displaced_header` 填 **NULL**，**不做 CAS**。
- 解锁时读 `_displaced_header`：**NULL ⇒ 这是重入层，直接弹栈返回，不动 Mark Word**；**非 NULL ⇒ 最外层，用 CAS 把 Mark Word 恢复成备份值**。

一句话：**用"displaced_header 是否为 NULL"来区分"最外层加锁"和"重入层"**。这是轻量级锁实现里最优雅的一笔，也是面试高频追问点。

#### 3.3.4 解锁与膨胀

```cpp
void ObjectSynchronizer::fast_exit(...) {
    // 若 displaced_header 是 NULL（重入层）→ 直接返回
    if (CAS(&obj->mark, lock, displaced_header)) return;   // 恢复成功 = 无人竞争
    // CAS 失败：说明期间有人把 Mark Word 改成了 Monitor 指针，已膨胀
    monitor->exit(true);   // 唤醒 EntryList / cxq 上的线程
}
```

CAS 失败的原因只有一个：**在我持有期间，另一个线程来过**（它 CAS 失败 → 触发 inflate → Mark Word 变成 monitor 指针）。这时原持有者必须"接着"去释放重量级锁，并把 `_owner` 置空、唤醒后继。

#### 3.3.5 轻量级锁的定位

| 优点 | 缺点 |
|---|---|
| 不阻塞、不进内核态（纯用户态 CAS） | 有竞争就退化成自旋，自旋浪费 CPU |
| 解锁 CAS 成功即 0 成本 | 需要栈上分配 Lock Record（JIT 可优化） |
| 适合"交替执行、临界区极短" | 临界区长 / 有阻塞 IO → 升级重量级 |

> ⚠️ **一个"教科书 vs 实现"的差异，说出来能惊艳面试官**：
> 教科书说"轻量级锁 CAS 失败后**自适应自旋**，自旋若干次仍失败才升级重量级锁"。在现代 HotSpot（JDK 8+）的实际代码里，`slow_enter` 的 trivial CAS 一旦失败就**立即 `inflate()` 成 ObjectMonitor**，然后在 `ObjectMonitor::enter()` 里做 `TrySpin`（自适应自旋），自旋失败才 `park()`。
> 也就是说：**自旋发生在 monitor 内部（重量级锁的实现里），不是发生在轻量级锁阶段**。锁状态从 `00` 变成 `10` 的那一刻，自旋才真正开始。这是版本演进导致的概念漂移，很多老博客的描述已经过时。

### 3.4 重量级锁：ObjectMonitor 全解析

#### 3.4.1 结构（`hotspot/src/share/vm/runtime/objectMonitor.hpp`）

`ObjectMonitor` 是一个 **C++ 对象**（不是 Java 对象），分配在 C heap 上，字段如下（`hotspot/src/share/vm/runtime/objectMonitor.hpp`）：

| 字段 | 类型 | 含义 | 关键度 |
|---|---|---|---|
| `_header` | `markOop` | 被"顶替"下来的**原始 Mark Word**（含 hash/age），解锁时要还原 | 高 |
| `_owner` | `volatile intptr_t` | **当前持有锁的线程指针，NULL = 空闲**——这是整个互斥的核心 | ★★★ |
| `_recursions` | `intptr_t` | 重入次数（owner 再次进入时 +1） | 高 |
| `_count` | `intptr_t` | 粗略的竞争者计数（近似值，仅用于启发式决策） | 中 |
| `_waiters` | `intptr_t` | 调用 `wait()` 的线程数量 | 中 |
| `_cxq` | `volatile` 单向链表 | **竞争队列**：新来竞争失败的线程，CAS **头插**（LIFO） | ★★★ |
| `_EntryList` | `ObjectWaiter*` | **待唤醒队列**：从 `_cxq` 转移过来的竞争者（FIFO） | ★★★ |
| `_WaitSet` | `ObjectWaiter*` | **条件等待队列**：调用 `wait()` 被挂起的线程（双向链表） | ★★★ |
| `_succ` | `intptr_t` | "假定继承者"：启发式预测下一个持锁线程，用于减少无效唤醒 | 高 |
| `_SpinDuration` | `intptr_t` | 自适应自旋的状态（记录该 monitor 的自旋预算/成功率） | 中 |
| `_QMode` | `intptr_t` | 队列处理策略（0~4，见 3.4.3） | 中 |
| `_obj` | `oop`（弱语义） | 反向指向被锁对象（用于调试/deflate 判断） | 低 |

**三个队列的职责划分（面试必问）：**

```
              调用 monitorenter / 竞争失败
                        │
                        │  CAS 头插 _cxq（新来的线程）
                        ▼
                ┌──────────────┐
                │    _cxq      │  单向链表，LIFO（后进先出）
                └──────┬───────┘
                       │   owner 释放时按 _QMode 策略转移
                       ▼
                ┌──────────────┐
                │  _EntryList  │  等待被唤醒后"重新竞争"的队列，FIFO
                └──────┬───────┘
                       │  unpark 选中一个 → 它去 CAS _owner
                       ▼
                ┌──────────────┐   获取到锁 → 成为 _owner → 执行临界区
                │   _owner     │
                └──────┬───────┘
                       │  执行中调用 wait()
                       ▼
                ┌──────────────┐
                │   _WaitSet   │  双向链表，调用 wait() 被挂起的线程
                └──────────────┘
                       │  notify / notifyAll / 被中断 / 超时
                       ▼  移回 _EntryList（或 cxq），重新竞争锁
```

记忆口诀：
- **`_cxq` = "门口排队"（刚来的，头插，LIFO）**
- **`_EntryList` = "候客区"（被唤醒/转移过来，等叫号）**
- **`_WaitSet` = "等候室"（已经拿到过锁、主动 wait 放弃锁的）**

`_cxq` 用头插（LIFO）是有意为之：新来的线程大概率"热"（缓存/分支预测都在状态），让它先试一次；同时头插是 O(1)，无需遍历。

#### 3.4.2 加锁：`ObjectMonitor::enter`

```cpp
void ObjectMonitor::enter(TRAPS) {
    // ① 快速路径：_owner 为空，直接 CAS 抢（这是唯一无锁的分支）
    if (Atomic::cmpxchg(Self, &_owner, NULL) == NULL) return;

    // ② 重入：owner 就是自己 → _recursions++
    if (Self == _owner) { _recursions++; return; }

    // ③ 自适应自旋：按 _SpinDuration 决定自旋多久（C2 版还会先看 owner 的线程状态）
    if (TrySpin(Self) > 0) return;

    // ④ 真正阻塞：
    //    构造 ObjectWaiter 节点 → CAS 头插 _cxq → 再试一次 _owner
    //    → 失败则 Atomic::inc(&_count) 并 park()（底层 pthread_mutex+condvar）
    //    → 被 unpark 后醒来，从 _cxq/_EntryList 中把自己摘掉，再 CAS _owner
}
```

关键点 ①：**重量级锁的"获取"第一步也是 CAS**，不是"天然就慢"。慢在 **park 进内核 + 唤醒的上下文切换**（Linux 上典型 1~10 μs 级别，取决于调度器）。

关键点 ③：自适应自旋的数据来源是 `_SpinDuration`，由 `TrySpin` 每次的成败反馈调整（成功 → 变长；失败 → 变短，下限可能降到 0 直接 park）。C2 里更激进：如果发现 monitor 很快会被释放（owner 在跑），会用 `cmp` 循环自旋；否则直接 park。

#### 3.4.3 解锁：`ObjectMonitor::exit`

```cpp
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
    if (_owner != Self) { /* 未持有 → 抛 IllegalMonitorStateException */ }
    if (_recursions != 0) { _recursions--; return; }   // 还有重入层，先减计数

    OrderAccess::release_store(&_owner, NULL);          // ★ 释放语义的内存屏障在这里
    // 然后按 QMode 决定把 _cxq 整体挪到 _EntryList，还是直接"交接"给 _succ
    // 再用 unpark 唤醒一个（或按策略唤醒多个）
}
```

**QMode 的四种策略（说明 HotSpot 在"延迟/吞吐"之间的取舍）：**

| QMode | 行为 | 特点 |
|---|---|---|
| 0（默认） | 把 `_cxq` 全部转移进 `_EntryList`，然后唤醒 EntryList 头部 | 简单，竞争中可能反复唤醒 |
| 1 | 只把 `_cxq` 的首节点摘出来唤醒 | 减少无效唤醒 |
| 2 | 若 `_succ` 有效则直接 unpark `_succ`（"交接"） | 避免惊群 |
| 3 | `_cxq` 和 `_EntryList` 混合唤醒 | 折中 |
| 4 | 直接 unpark EntryList 头 | 最省 |

> 记住结论：**默认非公平**。`_owner` 置空后，新来的线程可以立刻 CAS 抢走（barging），从而"插队"到已经在 `_cxq` 里排队很久的线程前面。这就是为什么 `synchronized` 不公平、为什么在高竞争下可能有**线程饥饿**风险（不过 HotSpot 有 `_succ` 防御性机制缓解：被唤醒的线程发现抢不到时会自旋等待，尽量不让"接班"失败）。

#### 3.4.4 一个 monitor 什么时候创建

- **懒创建**：只有发生竞争、需要膨胀时才 `inflate()` 出一个 ObjectMonitor（`ObjectSynchronizer::inflate`，从全局 monitor 列表 `gFreeList` 分配，用完归还但不销毁 → 复用池）。
- 注意 JDK 15 之后偏向锁移除后，**"无竞争 → 轻量级锁"**，才需要 monitor。
- 所以"每个对象都自带一个 monitor"的说法是**错的**——monitor 是竞争时才出现的。

#### 3.4.5 锁与 GC 的关系（延伸）

`ObjectMonitor` 里的 `_obj` 弱引用被锁对象；在 STW 期间，如果 monitor 属于某个死线程，JVM 需要做 `deflate_idle_monitors`（`MonitorDeflationThread`，JDK 9+ 独立线程按 `GuaranteedSafepointInterval` 定期收缩），把空闲 monitor 归还池子并**把 Mark Word 从 monitor 指针还原成无锁**。这也解释了：为什么长期持有的 monitor 会占用内存；以及为什么会看到 `-XX:MonitorDeflationMax`／`-XX:GuaranteedSafepointInterval` 这类参数。

### 3.5 三种锁的对比表

| 维度 | 偏向锁（JDK≤17） | 轻量级锁 | 重量级锁 |
|---|---|---|---|
| Mark Word | thread ID + epoch, lock=01,bias=1 | 指向 Lock Record, lock=00 | 指向 ObjectMonitor, lock=10 |
| 加锁开销 | 首次 1 次 CAS，之后 **0** | 1 次 CAS | CAS + 可能 park（内核） |
| 适用场景 | 单线程反复进出 | 多线程交替、临界区短 | 高竞争、临界区长、有阻塞 |
| 阻塞线程 | 无 | 无（自旋，不挂起） | 有（进 wait/entry 队列，park） |
| 是否进内核 | 否 | 否 | 是（futex/pthread） |
| 竞争表现 | 撤销需 safepoint，高竞争下极差 | CAS 失败 → 膨胀 | 上下文切换开销 |
| 是否公平 | - | - | 非公平（可插队） |
| 额外内存 | 0 | 线程栈 Lock Record | 堆上 ObjectMonitor |

### 3.6 为什么锁只能升级不能降级

这是**设计选择，不是物理限制**（技术上完全可以做降级，代码里也有 `deflate`）。HotSpot 不做的理由：

1. **降级的判定成本 > 收益**。要知道"现在没竞争了"，需要持续采样 monitor 的状态/冲突率，判定逻辑本身要加锁。
2. **状态机的"旧持有者"问题**。降级时如果发现 monitor 队列里还有线程，就不能降；而队列是否为空需要 O(n) 或加锁检查。
3. **膨胀后信息已迁移**。Mark Word 里原来存的东西（hash、epoch）已经被挪到 monitor 的 `_header`，降级要按"回滚顺序"精确还原，任何一步出错就是数据竞争/崩溃。
4. **一次性风险收益不对称**。锁膨胀通常意味着"这个锁确实有竞争"，回退到轻量级很可能马上又膨胀——来回震荡成本更高。
5. 唯一"半降级"是 **monitor 回收（deflation）**：空闲 monitor 被回收、Mark Word 还原成无锁（`lock=01`）。但这是"回收资源"，不是"主动退化"，且由后台线程周期性处理，不是热路径决策。

**面试标准答案**："HotSpot 的设计假设是——有竞争就大概率持续有竞争，因此升级后靠膨胀/收缩的采样统计（`MonitorInUseLists`/deflation）来间接判断，而不是逐次降级；代价最小、路径最确定。"

---

## 第 4 章 编译期优化：自旋、锁消除、锁粗化

这些优化是 **JIT（C1/C2）在编译期对字节码的重写**，跟解释器执行时的锁升级是两条并行的路径。面试如果只答"锁升级"而不提这几项，会被判定为"只读过一篇博客"。

### 4.1 自适应自旋（Adaptive Spinning）

**为什么自旋**：线程挂起/唤醒的代价（两次上下文切换 + 内核调度）远大于"原地空转等一会儿"。如果锁只会被占用极短时间（几十条指令），自旋比 park 划算。

**为什么"自适应"**：固定自旋次数（`-XX:PreBlockSpin`，JDK 6 早期是 10 次）在两种场景下都是错的：
- 锁持有时间极短 → 自旋次数不够，白自旋后还是要 park；
- 锁持有时间很长（或被抛到磁盘 IO、被调度出去了）→ 自旋次数太多，纯粹烧 CPU，还可能加剧"锁 convoy"。

**自适应的判定来源**（`ObjectMonitor::TrySpin`）：
- 该 monitor 的 `_SpinDuration`：历史自旋的成功率越高，允许自旋的轮次越多（`Knob_SpinLimit` 为上限）；
- C2 还会看 owner 的状态：如果 owner 正在运行（`_succ` 未设置且 owner 处于 running），倾向于自旋；如果 owner 在 `_cxq` 上等待（即"二级竞争"），直接放弃自旋去排队，避免无效自旋。

> 一句话回答：**"自旋次数不是常数，而是由该 monitor 的历史成功率动态调节；C2 还会结合 owner 的线程状态做判断。"**

### 4.2 锁消除（Lock Elision）

前提是 **逃逸分析（Escape Analysis）**：

```java
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();  // sb 未逃逸（只是方法内局部使用）
    sb.append(a).append(b);                // append 内部有 synchronized
    return sb.toString();
}
```

- `StringBuffer.append` 是 `synchronized` 方法，但 `sb` 只在本方法内被访问，**不可能被其它线程看到** → 锁无意义 → C2 把整个 `monitorenter/monitorexit` 消除掉。
- 开关：`-XX:+EliminateLocks`（默认开，需 `-XX:+DoEscapeAnalysis`，默认开）。**注意：只有 C2 做，C1 不做**；解释执行时仍会加锁。所以"我测出来 StringBuffer 不加锁也很快"必须在 JIT 编译后才成立。
- 局限：如果对象被写入字段、被传入未知方法、被 lambda 捕获并逃逸 → 分析失败，不消除。

### 4.3 锁粗化（Lock Coarsening）

```java
for (int i = 0; i < 1000; i++) {
    synchronized (lock) { count++; }   // 1000 次加锁/解锁
}
// C2 可能优化成：
synchronized (lock) { for (...) count++; }   // 1 次
```

判定条件：**相邻的、锁对象相同的、中间没有逃逸/异常边界的临界区**，且"合并后的临界区不太大"。反之如果两个临界区之间隔着很长的计算，粗化会让锁持有时间过长、反而降低并发 → 不粗化。

> 这两项优化的面试价值在于：说明 **`synchronized` 不是"运行时行为"，而是"JIT 可重写的行为"**。同一个方法，解释执行、C1、C2 编译后的锁行为可能完全不同。

### 4.4 其它相关优化

| 优化 | 说明 |
|---|---|
| 栈上分配 / 标量替换 | 逃逸分析副产品，减少对象创建（间接减少锁） |
| 消除冗余 load/store | 依赖 monitor 的 acquire/release 语义做"锁内不可见外部写" |
| 消除重复的 monitor | 嵌套同一 monitor 的 enter/exit 可合并（见锁粗化） |
| `-XX:+UseHeavyMonitors` | 强制所有锁走重量级（调试用）；`-XX:-UseBiasedLocking` 关偏向 |

---

## 第 5 章 内存语义：happens-before 与内存屏障

`synchronized` 的三性里，"原子性"容易理解（临界区互斥），"可见性/有序性"靠的是**JMM 的锁规则 + 内存屏障**，这才是真正需要说清的部分。

### 5.1 JMM 中的两条规则

**规则一（happens-before）**：对一个 monitor 的 **unlock happens-before** 后续对**同一 monitor** 的 lock。

```
  time
   |
   |  [Thread A]  x = 1;
   |  [Thread A]  monitorenter(lock)      <- acquire
   |  [Thread A]      y = 2;
   |  [Thread A]  monitorexit(lock)       <- release
   |                    |
   |                    +-- happens-before (same monitor) --+
   |                                                         |
   |  [Thread B]                      monitorenter(lock) <---+
   |  [Thread B]                          read y  ==> 2      (visible: A's write)
   |  [Thread B]                          read x  ==> 1      (visible: A's write)
   |  [Thread B]                      monitorexit(lock)
   v
```

> 注意 `x = 1` 是**普通字段**（非 volatile），它之所以对 B 可见，**只是因为"monitor 的 unlock → lock"这一对 happens-before 把它"顺带"发布了**。这就是 `synchronized` 兼具 volatile 语义的机制来源。

只要拿到的是**同一个锁对象**，A 在 unlock 之前的所有写，对 B 在 lock 之后的读**全部可见**，包括普通（非 volatile）字段。这就是 `synchronized` 兼具 volatile 语义的原因。

**规则二（原子性）**：`monitorenter` 到 `monitorexit` 之间的代码对其它线程表现为不可分割（对同一把锁而言）。

### 5.2 实现：acquire / release 屏障

JSR-133 定义的锁语义 → 编译器插入屏障：

| 位置 | 屏障 | 作用 |
|---|---|---|
| `monitorenter` 之后 | **LoadLoad + LoadStore（acquire 语义）** | 禁止"后面的读/写"被重排到锁之前 |
| `monitorexit` 之前 | **StoreStore + StoreLoad（release 语义）** | 保证临界区内所有写，在解锁前对其他 CPU 可见 |

具体在 **x86（TSO 内存模型）** 上的落地：

- **acquire 侧：不需要任何指令**。x86 的 TSO 保证 load 不会越过更早的 load/store，所以"进入锁后的读"天然有序。这也是 x86 上 `synchronized` 进出成本很低的重要原因。
- **release 侧：必须一条 `lock` 前缀指令或 `mfence`**。HotSpot 在 x86 上用 `lock addl $0, (%rsp)`（"锁总线/锁缓存行的一次空加"）来实现 full fence，因为 `mfence` 在部分老 CPU 上更慢。C2 的 `membar` 节点最终生成这条指令。
- **CAS 自带的屏障**：`lock cmpxchg` 本身就是 full barrier（既 acquire 又 release），所以**轻量级锁的加锁/解锁天然带内存语义**，不需要额外插桩。

在 ARM（AArch64）上，则分别需要 `ldar`（acquire）/ `stlr`（release）或 `dmb ish`，成本明显高于 x86 → 这是"同一段并发代码在 ARM 服务器上性能不一样"的底层原因之一。

### 5.3 synchronized 能保证有序性吗？

**能，但不是"禁止所有重排"**，而是"禁止跨锁边界的重排"：

```java
synchronized (lock) {
    a = 1;      // ①
    b = 2;      // ②
}
// ①② 之间允许重排（都在锁内，对外不可见）
// 但 ①② 都不允许被重排到 monitorexit 之后
```

这正是 **DCL（双重检查锁）单例**必须加 `volatile` 的原因——**只加 synchronized 是不够的**：

```java
// ❌ 错误写法：instance 没有 volatile
public static Singleton getInstance() {
    if (instance == null) {                    // 第一次检查（锁外读）
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();    // ← new 不是原子的！
                // 实际分三步：1.分配内存 2.构造/初始化 3.把引用赋给 instance
                // 2 和 3 可以重排 → 别的线程看到"非 null 但未初始化"的对象
            }
        }
    }
    return instance;
}
```

**为什么 synchronized 救不了它**：因为第一次 `if (instance == null)` 的读**在锁外面**，它和锁内的写之间没有 happens-before（没有共享锁的 unlock→lock 配对），所以可以读到重排后的中间状态。加 `volatile` 后，赋值那一步有 release 语义，锁外的读有 acquire 语义，才形成正确的发布（safe publication）。

> 这个点是"DCL 为什么必须 volatile"的标准答案，也是 synchronized 语义边界最好的例子。

### 5.4 synchronized vs volatile 对比

| 维度 | synchronized | volatile |
|---|---|---|
| 原子性 | ✅ 临界区整体原子 | ❌ 只保证单次读/写（i++ 不行） |
| 可见性 | ✅ unlock→lock 传递 | ✅ 每次读都从主存/缓存一致性协议拿 |
| 有序性 | ✅ 锁边界禁止重排 | ✅ 前后插屏障禁止重排 |
| 阻塞 | ✅ 会阻塞 | ❌ 不阻塞 |
| 适用 | 复合操作、需要互斥 | 状态标志、一次写入多次读 |

---

## 第 6 章 硬件层：CAS、缓存一致性、park/unpark

### 6.1 CAS 到底是什么

```java
// AtomicInteger.compareAndSet 的底层：Unsafe.compareAndSwapInt
// x86-64 上的实现（hotspot/src/cpu/x86/vm/assembler_x86.cpp，macroAssembler::cmpxchg）
lock cmpxchg [addr], reg
```

- `cmpxchg`：比较并交换（比较 `[addr]` 与 eax，相等则写入 reg，否则把 `[addr]` 加载进 eax）。
- **`lock` 前缀**：让这条指令对**所有 CPU 核**原子。它有两种实现方式：
  - **总线锁**：拉低 `LOCK#` 信号，锁住整个内存总线（老 CPU、跨 cache line 时使用）；
  - **缓存锁**：通过 MESI 协议把该 cache line 置为 **Exclusive/Modified**，其它核的访问被挡住（现代 CPU 的主流方式，成本低得多）。
- `lock` 前缀同时具有 **full memory barrier** 效果（禁止指令重排、刷 store buffer）。

**MESI 简述**（面试常顺带问）：
| 状态 | 含义 |
|---|---|
| M (Modified) | 本核独占且已修改，其他核无副本，写回内存由总线 snoop 保证 |
| E (Exclusive) | 本核独占且干净 |
| S (Shared) | 多核共享，只读 |
| I (Invalid) | 无效，需重新加载 |

一个核要写共享行 → 发 `RFO`（Request For Ownership）让其它核置 I → 若其它核也有该行，还需写回 → **这就是 CAS 的性能上限来源**（cache line 在核间"乒乓"）。所以：**高竞争 CAS 的瓶颈是缓存一致性流量，不是指令本身**。

### 6.2 伪共享（False Sharing）——延伸但极有用

两个独立变量落在**同一条 64 字节 cache line** 上，不同核分别写 → 虽然逻辑无冲突，但物理上互相使对方 cache line 失效，性能暴跌数倍。

```java
// 典型修复：用填充把两个热点变量隔开（JDK 8 的 @Contended，需 -XX:-RestrictContended）
@jdk.internal.vm.annotation.Contended
volatile long value;
```

`LongAdder`、`ConcurrentHashMap` 的 counter cells、Disruptor 的 Sequence 都用这个思路。虽然跟 `synchronized` 不直接相关，但它是"锁竞争为什么会慢"的物理层解释——**多核环境下，任何"共享写"都在付出缓存一致性成本，锁只是把它显式化了**。

### 6.3 park / unpark 底层

`LockSupport.park()`（重量级锁挂起线程用的就是它，`ObjectMonitor::park` → `PlatformEvent::park`）：

```
Java:  LockSupport.park()
  ↓
JVM:   Parker::park()  →  PlatformEvent::park()
  ↓
Linux: pthread_mutex_lock(&_mutex) + pthread_cond_wait(&_cond, &_mutex)
  ↓
glibc: futex(FUTEX_WAIT) 系统调用 → 进入内核，线程状态 TASK_INTERRUPTIBLE
  ↓
调度器: 从 runqueue 摘下，等 unpark 时 futex(FUTEX_WAKE) → TASK_RUNNING → 等 CPU
```

要点：
- **park 是"许可（permit）"语义**，不是"信号量"：`unpark` 可以先于 `park` 调用（permit 被记下来，最多 1 个），所以不会"丢失唤醒"。这是它比 `wait/notify` 更可靠的原因（`notify` 先于 `wait` 会丢失）。
- **不是每次 park 都进内核**：futex 会先在用户态检查（fast path），只有真要睡眠才 syscall。竞争不激烈时，park/unpark 可能完全在用户态完成（这是 JDK 9+ 的优化方向）。
- 相比 `Thread.sleep` / `wait`：park 可以被 `unpark`、`interrupt` 精确唤醒，粒度更细。

---

## 第 7 章 wait / notify 机制

这部分是"锁 + 条件变量"的组合，是把 `ObjectMonitor` 的三个队列全部用上的地方。

### 7.1 语义三条铁律

1. **`wait()` / `notify()` / `notifyAll()` 必须在持有该对象 monitor 时调用**，否则抛 `IllegalMonitorStateException`。
2. **`wait()` 会释放锁**（`_owner` 置空、`_recursions` 清零暂存），并进入 `_WaitSet`，被唤醒后**需要重新竞争锁**才能从 `wait()` 返回。
3. **`notify()` 不释放锁**——被唤醒的线程只是从 `_WaitSet` 移到 `_EntryList`，必须等 notify 所在临界区**执行完并 monitorexit** 之后，才可能真正拿到锁。

### 7.2 完整时序图

```
[1] A: synchronized(lock)                     ->  _owner = A
[2] A: while(queue.isEmpty()) lock.wait()
        ->  A 挂到 _WaitSet；_owner = NULL（释放锁）；A park 挂起
[3] B: synchronized(lock)                     ->  _owner = B（B 拿到锁）
[4] B: queue.add(x); lock.notify()
        ->  把 A 从 _WaitSet 移到 _EntryList（B 仍持有锁，A 还不能执行）
[5] B: } 即 monitorexit                       ->  _owner = NULL，unpark 一个后继
[6] A: 从 park 返回 -> 重新 CAS 抢 _owner -> 抢到后才从 wait() 返回
[7] A: 回到 while 条件判断 -> 成立 -> consume()
```

> 把 [4] 和 [5] 分两步看，就理解了"`notify` 不释放锁"和"被唤醒线程何时才真正执行"这两个问题的全部答案。

注意 ③ 这个"醒来后还要重新抢锁"的点，决定了 `notify` 之后被唤醒线程**不可能立刻执行**，也决定了为什么必须用 `while` 而不是 `if`。

### 7.3 为什么必须用 while 包裹 wait（虚假唤醒）

三种"醒来但条件不成立"的情况：

1. **虚假唤醒（spurious wakeup）**：操作系统/硬件允许 `wait` 在没收到 notify 的情况下返回（POSIX 明确允许）。根源是 `pthread_cond_wait` 的语义就是"可能虚假唤醒"，必须由调用方循环复查。
2. **被 notify 但条件被别人先消费**（多个消费者，一个生产信号）。
3. **被 `notifyAll` 唤醒但不是目标线程**。

所以：

```java
// ❌ 错误：if + wait
if (queue.isEmpty()) lock.wait();   // 醒来直接往下走，可能队列还是空 → NPE / 逻辑错误
consume();

// ✅ 正确：while + wait
while (queue.isEmpty()) lock.wait();
consume();
```

### 7.4 wait 与 sleep 的区别（面试送分题，但要答全）

| 维度 | `Object.wait()` | `Thread.sleep()` |
|---|---|---|
| 所属 | Object 实例方法 | Thread 静态方法 |
| 是否释放锁 | ✅ 释放 monitor | ❌ 不释放（抱着锁睡） |
| 是否需持锁 | ✅ 必须（否则 IMSE） | ❌ 不需要 |
| 唤醒方式 | notify/notifyAll/超时/中断 | 超时/中断 |
| 底层 | `_WaitSet` + park | 单纯 `Thread.sleep` → 定时器 + park |
| 典型误用 | 在非同步块里调用 | 在同步块里 sleep（导致别人全部等） |

### 7.5 一个高频陷阱：notify 唤醒哪个线程？

`notify` 是**任选一个**（JVM 实现通常是 `_WaitSet` 头部，但规范不保证），所以：

- 多个条件不同的等待者共用一把锁时，**必须用 `notifyAll`**，否则可能唤醒"错误的那一个"导致永久卡死（丢失信号）。
- 典型错误：生产者-消费者共用一把锁，队列满时生产者 wait，队列空时消费者 wait。用 `notify` 可能唤醒另一个生产者（队列还是满）→ 所有线程互相等待 → **假死**（不是死锁，因为没形成环，但永久停滞）。`jstack` 上看是 `WAITING (on object monitor)` 却没有进展。

---

## 第 8 章 JDK 版本演进与虚拟线程的坑

### 8.1 版本时间线

| 版本 | 变化 | 面试意义 |
|---|---|---|
| JDK 1.0 ~ 1.5 | 只有重量级锁（mutex），`synchronized` 极慢，社区主推 `ReentrantLock` | 老面试题"为什么 ReentrantLock 比 synchronized 快"的出处 |
| JDK 6（mustang） | 引入**偏向锁、轻量级锁、自适应自旋、锁消除、锁粗化** | 锁升级方案的诞生，绝大多数中文博客讲的就是这版 |
| JDK 9 | `MonitorDeflationThread` 独立做 monitor 收缩 | 长期持有 monitor 的内存问题被缓解 |
| JDK 15（JEP 374） | **偏向锁默认禁用**（`-XX:+UseBiasedLocking` 变成 opt-in） | 面试新分水岭 |
| JDK 18 | **偏向锁实现被移除**（相关参数失效） | 再说"默认有偏向锁"就是错的 |
| JDK 21（JEP 444） | 虚拟线程 GA，`synchronized` 会 **pin 住载体线程** | 新热点问题 |
| JDK 24（JEP 491） | 虚拟线程在 `synchronized` 上不再 pinning | 前沿加分项 |

### 8.2 虚拟线程 + synchronized 的 pinning 问题（高价值延伸）

**现象**：在 JDK 21 的虚拟线程里，如果阻塞发生在 `synchronized` 块内部（例如块里做了阻塞 IO 或 `Thread.sleep`），虚拟线程**无法从载体线程（carrier）上卸载**，会把它所在的平台线程一起卡住 → 并发能力退化到平台线程数量级别，甚至因为载体线程池耗尽而死锁。

**原因**：虚拟线程的挂起/恢复依赖 `LockSupport.park` 与 continuation 的 yield；而 `synchronized` 的 monitor 是**绑定在操作系统线程（平台线程）身份上的**（`_owner` 记录的是 OS 线程/JavaThread），一旦虚拟线程在持有 monitor 期间被卸载到另一个载体线程，monitor 的持有者身份就无法正确表达。因此 JVM 选择**保守策略：持有 monitor 期间不允许 unmount（pin）**。

**规避方式（JDK 21 时代）**：
- 用 `ReentrantLock` 替代 `synchronized`（AQS 的 park 是可让出的）；
- 缩短同步块，把阻塞操作（IO/sleep/网络）移出 `synchronized`；
- 用 `-Djdk.tracePinnedThreads=full` 打印 pinning 栈定位现场。

**JDK 24 的 JEP 491**：把 monitor 的持有者从"平台线程"改成"虚拟线程"，使 `synchronized` 也能安全卸载。所以标准的表述是：**"JDK 21~23 有 pinning 问题，JDK 24 修复；新项目在 21 上慎用 synchronized 包阻塞调用"**。

> 这一段是 2024 年后大厂面试的"新题"，能答出来直接区分"读过 2020 年博客"和"持续跟进 JEP"的候选人。

### 8.3 顺带澄清一个老谣言

> "JDK 6 之后 synchronized 和 ReentrantLock 性能一样，所以优先用 synchronized。"

这句话**在 JDK 6~14、低竞争场景下大致成立**，但要补充三点：
1. **高竞争**下 `ReentrantLock` 的 `tryLock(timeout)` / 公平模式 / `Condition` 多条件队列仍是 `synchronized` 给不了的；
2. **JDK 15+ 没有偏向锁**后，单线程反复进出 `synchronized` 的成本略高于 JDK 8，但已被 JIT 的锁消除/粗化大量抵消；
3. **虚拟线程（JDK 21）**下 `synchronized` 有 pinning，`ReentrantLock` 更适合。

---

## 第 9 章 大厂面试题库（按难度分级）

### 评级说明

| 等级 | 标记 | 含义 | 出现轮次 |
|---|---|---|---|
| L1 基础必答 | ★ ~ ★★ | 不会就直接挂，必须脱口而出 | 一面 / 笔试 |
| L2 中级高频 | ★★★ | 区分"用过"和"懂了" | 一面~二面 |
| L3 高级深挖 | ★★★★ | 区分"读过博客"和"读过源码" | 二面 / 三面 |
| L4 专家细节 | ★★★★★ | 区分"读过源码"和"真的看过 HotSpot" | 三面 / 交叉面 / 专家面 |
| L5 场景设计 | ★★★~★★★★★ | 考察工程判断，往往无标准答案 | 三面 / leader 面 |

> 说明：题目来源为各家公开面经中反复出现的原始表述，同一考点在不同公司只是问法不同。`★` 数量同时代表"被追问的深度"，不代表"重要性"——L4 的题在 L1 面前往往也只需要一句话。

---

### L1 基础必答

#### Q1. synchronized 有哪几种用法？分别锁的是什么对象？
**来源**：阿里/腾讯/京东 一面 ·**考核程度**：★☆☆☆☆（必答，答错即淘汰）

**参考答案**

| 用法 | 锁对象 | 备注 |
|---|---|---|
| 修饰实例方法 | `this` | 与 `synchronized(this)` 同锁 |
| 修饰静态方法 | `当前类的 Class 对象` | JVM 内全局唯一，与 `synchronized(Xxx.class)` 同锁 |
| 修饰代码块 | 括号里的对象引用 | 灵活控制粒度，**推荐写法** |

**三条必须补的结论**：
1. 锁在**对象**上，不在代码上。两个线程锁不同对象 → 不互斥。
2. **实例锁与类锁互不影响**：`synchronized` 静态方法 与 `synchronized` 实例方法可并发执行。
3. `synchronized(new Object())`（每次 new）等价于无锁——这是最常见的线上事故写法。

**延伸追问**："`synchronized` 修饰构造方法可以吗？"——语法上不允许（构造方法不能有 `synchronized` 修饰符，编译报错），因为对象还没构造完，锁没有意义（外部拿不到引用）。

---

#### Q2. synchronized 的底层实现原理是什么？
**来源**：几乎所有公司一面 · **考核程度**：★★☆☆☆（标准开场题）

**参考答案（分两层说，不要只答一半）**

**① 字节码层**
- 同步代码块：编译器生成 `monitorenter` + 2 个 `monitorexit`（正常路径 1 个，异常路径 1 个，异常表类型为 `any`）。
- 同步方法：不加指令，而是在方法访问标志里置 `ACC_SYNCHRONIZED`，由 JVM 在方法调用/返回时隐式加锁解锁。

**② JVM 运行时层**
- `monitorenter` 的执行入口是 `InterpreterRuntime::monitorenter`，最终进入 `ObjectSynchronizer::fast_enter`：
  ```
  fast_enter
   ├─ 偏向锁可用 → BiasedLocking::revoke_and_rebias（CAS 写线程ID，零成本重入）
   └─ 否则 → slow_enter
        ├─ Mark Word 无锁 → CAS 换成本线程 Lock Record（轻量级锁）
        └─ CAS 失败 → inflate() 生成 ObjectMonitor → monitor->enter()
                        （自适应自旋 → 失败则 park 阻塞）
  ```

**③ 最终依赖**
- 对象头 Mark Word 的 2 bit 锁标志位记录状态；
- 原子操作靠 CPU 的 `lock cmpxchg`（CAS）；
- 阻塞靠 `park/unpark`（Linux 上是 `pthread_mutex` + `pthread_cond_wait` → `futex`）。

**答题结构建议**：字节码 → JVM 实现 → CPU 指令 → 操作系统，四层说完就是满分。

---

#### Q3. 为什么同步方法没有 monitorenter/monitorexit 指令？
**来源**：字节跳动 二面 · **考核程度**：★★★☆☆（考察是否真看过字节码）

**参考答案**

- 同步方法的锁动作是**方法调用协议的一部分**，不需要在方法体里重复表达。JVM 在执行 `invoke*` 时，如果 `method->is_synchronized()` 为真，会先取锁对象（实例方法取 `recv` 即 `this`，静态方法取 `method_holder()->java_mirror()` 即 Class 对象），执行隐含的 `monitorenter`；方法返回（含异常返回）时执行 `monitorexit`。
- 好处是：**锁的范围严格等于方法体**，不存在"忘记 unlock"的可能，也少两条指令，且 JIT 可以把"方法入口 = 加锁"当作内联时的已知条件。
- 代价是：**粒度不可控**（整个方法都锁住，包括不需要同步的代码），所以最佳实践是"同步方法只用于确实整个方法都要互斥的场景，否则改代码块"。

**追问**："单例的 `getInstance()` 方法加 `synchronized` 有什么问题？"——整个方法都锁，包括返回值的处理；且每次调用都要抢锁，正确做法是 DCL + volatile，只在 `instance == null` 时进入同步块。

---

#### Q4. 对象头里有什么？Mark Word 的结构是怎样的？
**来源**：美团/京东/滴滴 一面 · **考核程度**：★★★☆☆

**参考答案**

**对象头组成（64 位 JVM）**：
- Mark Word：8 字节，存 hash / GC 年龄 / 锁状态 / 偏向线程 ID 等（**复用**）；
- Klass Pointer：4 字节（开压缩指针，`-XX:+UseCompressedClassPointers`）或 8 字节，指向方法区里的 `InstanceKlass`；
- 数组对象额外 4 字节存长度。

**Mark Word 关键位（最低 2 bit = 锁标志 + biased_lock 位）**：

| lock | biased_lock | 状态 | Mark Word 里装什么 |
|---|---|---|---|
| 01 | 0 | 无锁 | hash(31) + age(4) |
| 01 | 1 | 偏向锁 | thread ID(54) + epoch(2) |
| 00 | — | 轻量级锁 | 指向线程栈 Lock Record 的指针 |
| 10 | — | 重量级锁 | 指向 ObjectMonitor 的指针 |
| 11 | — | GC 标记 | — |

**加分点**：主动说"这是**复用**字段，所以 `identityHashCode` 和偏向锁互斥"（详见 Q23），面试官通常会顺着这个点深入。也可以说"用 JOL 的 `ClassLayout.parseInstance().toPrintable()` 可以实测这些位"。

---

#### Q5. synchronized 是可重入的吗？底层怎么实现重入？
**来源**：百度/网易 · **考核程度**：★★☆☆☆

**参考答案**

**是可重入的**，否则一个同步方法调用另一个同锁同步方法就会自死锁。

**实现按锁状态分三种**：
1. **偏向锁**：Mark Word 里的线程 ID 就是自己 → 直接返回，**什么都不做**（连计数都没有，这是它最省钱的地方）。
2. **轻量级锁**：每重入一层就在栈上再压一个 Lock Record，但 `displaced_header` 填 **NULL** 表示"重入层"，且不做 CAS。解锁时见到 NULL 直接弹栈（详见 Q16）。
3. **重量级锁**：`ObjectMonitor._owner == Self` → `_recursions++`；解锁时 `_recursions--`，减到 0 才真正释放 `_owner`。

**延伸**：重入次数理论上受栈大小限制（`StackOverflowError`），JIT 会静态计算 `max_monitors` 一次性在栈帧里预留 Lock Record 空间。

---

#### Q6. 同步代码块里抛出异常，锁会释放吗？
**来源**：京东/携程 · **考核程度**：★★★☆☆

**参考答案**

**会释放**。因为 javac 为 `synchronized` 块生成了异常表项（`type = any`），把 4~16 的字节码范围映射到"19: aload_1; monitorexit; athrow"这一段处理器，任何 Throwable 都会走异常路径的 `monitorexit`。

**关键对比**：
```java
// ✅ 自动释放
synchronized (lock) { throw new RuntimeException(); }

// ❌ 手动锁必须 finally，否则泄漏
lock.lock();
try { throw new RuntimeException(); } finally { lock.unlock(); }
```

**延伸**：如果 `monitorexit` 执行时发现 enter/exit 不平衡（比如字节码被篡改），JVM 会抛 `IllegalMonitorStateException`；这也是字节码校验器要检查 enter/exit 配对的原因。

---

### L2 中级高频

#### Q7. 说说锁升级的完整过程
**来源**：阿里/字节/美团/快手 二面（**出现率最高的一题**）· **考核程度**：★★★★☆

**参考答案（按顺序讲，每个状态给"进入条件 + Mark Word + 代价"）**

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

**① 无锁（lock=01, biased=0）**：`new Object()` 后就是这个状态。

**② 偏向锁（lock=01, biased=1）**
- 进入条件：第一个线程进入同步块，且偏向锁开启（JDK 6~14 默认）。
- 动作：CAS 把线程 ID 写进 Mark Word。
- 代价：首次 1 次 CAS，之后同线程重入 **0 次原子操作**。

**③ 轻量级锁（lock=00）**
- 进入条件：另一个线程来竞争 / 调用了 `identityHashCode` / 偏向撤销。
- 动作：在当前线程栈帧分配 Lock Record，把原 Mark Word 存进去（displaced header），CAS 让 Mark Word 指向该 Lock Record。
- 代价：1 次 CAS，不阻塞、不进内核。

**④ 重量级锁（lock=10）**
- 进入条件：CAS 反复失败（有真实竞争）/ 有线程已在等待 / 调用了 `wait()`。
- 动作：`inflate()` 在堆里创建 `ObjectMonitor`，Mark Word 改为指向它；竞争失败线程进 `_cxq`/`_EntryList` 并 `park` 挂起。
- 代价：上下文切换（内核态），μs 级。

**必须补的两个进阶点**：
1. **升级不可逆**（详见 Q17）。
2. **JDK 15+ 没有偏向锁**，链路变成"无锁 → 轻量级 → 重量级"；回答时提一句"我按 JDK 8 讲，JDK 15 之后偏向锁默认关闭"，立刻区分于照抄博客的候选人。

---

#### Q8. 偏向锁、轻量级锁分别适合什么场景？为什么？
**来源**：腾讯/小米 · **考核程度**：★★★☆☆

**参考答案**

| | 偏向锁 | 轻量级锁 | 重量级锁 |
|---|---|---|---|
| 适合 | **同一线程**反复进出同一锁，几乎无第二个线程 | 多线程**交替**执行（有时间错开），临界区短 | 真并发、临界区长、含阻塞 IO |
| 原理依据 | 加锁退化为"比较线程 ID" | 用 CAS 代替互斥量，把"互斥"降级为"乐观重试" | 只能靠挂起，让出 CPU |
| 不适合 | 多线程轮流锁同一对象（撤销成本极高，触发 safepoint） | 高竞争（CAS 失败 + 自旋烧 CPU） | 低竞争短临界区（切换成本 > 收益） |

**一句话总结**：**偏向锁赌"没人抢"，轻量级锁赌"抢的人少且很快放手"，重量级锁承认"确实在抢"。**

**延伸**：这也解释了为什么 JDK 15 敢废弃偏向锁——现代应用里"没人抢"的概率被线程池和并发框架大幅拉低，赌输的代价（safepoint 撤销）却很高。

---

#### Q9. synchronized 和 ReentrantLock 有什么区别？怎么选？
**来源**：阿里/腾讯/美团（几乎必问）· **考核程度**：★★★☆☆

**参考答案**

**相同**：都可重入、都保证互斥 + 可见性 + 有序性、都非公平（默认）。

**差异**：

| 维度 | synchronized | ReentrantLock |
|---|---|---|
| 实现层 | JVM 内置（monitor） | JDK 层 AQS（Java 代码，可改可扩） |
| 释放 | 自动（异常安全） | 必须 `finally` 手动 |
| 可中断 | ❌ 获取锁不可中断 | ✅ `lockInterruptibly()` |
| 超时 | ❌ 死等 | ✅ `tryLock(timeout)` |
| 公平 | ❌ 强制非公平 | ✅ 可选公平（`new ReentrantLock(true)`） |
| 条件队列 | 1 个（`_WaitSet`），`notify` 随机唤醒 | ✅ 多个 `Condition`，精确唤醒 |
| 读写分离 | ❌ | `ReentrantReadWriteLock` |
| 锁状态查询 | ❌ | `isLocked()`、`getQueueLength()` |
| 性能 | JDK 6 后接近，低竞争更优 | 高竞争/复杂场景更优 |

**选型建议**：
- 简单互斥、无超时/中断需求 → `synchronized`（不可能忘记释放，JIT 优化更彻底）；
- 需要超时防死锁、可中断、多条件队列、公平锁、读写分离 → `ReentrantLock`；
- **JDK 21 虚拟线程**下，阻塞调用优先用 `ReentrantLock`（`synchronized` 会 pin 载体线程）。

**追问陷阱**："synchronized 底层也用 CAS，那它和 AQS 的 CAS 有什么区别？"——`synchronized` 的 CAS 在 **Mark Word** 上（对象头），AQS 的 CAS 在 **`volatile int state`** 上（普通字段），后者是纯 Java 层，前者要改对象头且需要 JIT/解释器配合。AQS 的等待队列是显式的 CLH 双向链表（`Node`），`synchronized` 的队列藏在 C++ 的 `ObjectMonitor` 里（见上一章对比）。

---

#### Q10. synchronized 能保证可见性吗？靠什么保证？
**来源**：阿里/百度 二面 · **考核程度**：★★★★☆

**参考答案**

**能**。JMM 规定：**对同一 monitor 的 unlock happens-before 后续的 lock**。

```
线程A:  x=1;  synchronized(lock){ y=2; }   ← unlock
线程B:                        synchronized(lock){ read x,y }  ← lock
```
A 在 unlock 前的所有写（包括普通字段 `x`），对 B 在 lock 后的读**全部可见**。

**实现机制**：
- `monitorenter` 后插 **acquire 屏障**（LoadLoad/LoadStore）：禁止后续读写被重排到锁之前；
- `monitorexit` 前插 **release 屏障**（StoreStore/StoreLoad）：保证临界区内写全部刷出、禁止被重排到锁之后；
- x86 上 acquire 侧不需要指令（TSO 保证），release 侧用 `lock addl $0,(%rsp)`；
- 轻量级锁的 CAS 本身是 `lock cmpxchg`，自带 full barrier，额外语义免费获得。

**边界（高频追问）**：
- **只在"同一把锁"上成立**。A 锁 lock1、B 锁 lock2 → 无 happens-before。
- **`synchronized` 不能替代 `volatile` 做"锁外读"**，例如 DCL 第一次判空在锁外（见 Q26）→ 必须 `volatile`。

---

#### Q11. wait/notify 为什么必须在 synchronized 块里调用？
**来源**：京东/网易 一面 · **考核程度**：★★☆☆☆

**参考答案**

**三个层面**：

1. **语义层（防丢失唤醒）**：`wait` 和 `notify` 是**条件变量**操作，必须与"条件判断"原子化。如果不持锁，可能出现：
   ```
   A: if(!ready)                    B: ready=true; notify();  ← 此时 A 还没 wait
      // 被切走                        （notify 找不到等待者，信号丢失）
      wait();                        ← 永久等待
   ```
   持锁后，`判断条件` + `wait()` 在同一临界区，B 无法插进来，信号不会丢。

2. **实现层**：`notify` 需要从 `_WaitSet` 里摘节点并移到 `_EntryList`，`wait` 需要改 `_owner`/`_count` 并 park——**都是对 monitor 内部状态的修改，必须在持锁时进行**，否则并发修改链表会崩。

3. **规范层**：`Object.wait/notify/notifyAll` 在 `ObjectSynchronizer` 里第一件事就是检查 `_owner == Self`，不满足直接 `IllegalMonitorStateException`。

**延伸对比**：`LockSupport.park/unpark` 不需要持锁，因为它是**许可（permit）模型**——`unpark` 可以先于 `park` 执行（许可最多累积 1 个），所以不存在信号丢失问题。这就是 AQS 用 park 而不用 wait 的原因之一。

---

#### Q12. notify 之后，被唤醒的线程什么时候执行？为什么不释放锁？
**来源**：美团/字节 二面 · **考核程度**：★★★☆☆

**参考答案**

**完整时序**：

```
notify() 调用瞬间：
  ① 从 _WaitSet 摘一个节点 → 插入 _EntryList（或 _cxq）
  ② notify 所在线程【继续持有锁】，_owner 不变
  ③ 被唤醒的线程从 park 返回，但立刻发现 _owner 还是别人 → 继续阻塞/自旋
  ④ notify 所在临界区执行完 → monitorexit → _owner=NULL → unpark 后继
  ⑤ 被唤醒线程 CAS 抢到 _owner → 才真正从 wait() 返回
```

**为什么 notify 不释放锁**：
- 设计上，`notify` 之后通常还要继续访问共享状态（例如"取走一个元素后调整 size"），如果立刻释放锁，被唤醒线程可能看到**中间状态**，破坏临界区原子性。
- 所以语义是：**`notify` 只是"给个信号/把票发出去"，真正的交接点在 `monitorexit`。** 与之相对，`wait` 必须立刻释放锁——因为它接下来要睡很久，抱着锁睡会阻塞所有人（甚至死锁）。

**易错点**：`wait` 是**唯一**会释放锁的 monitor 操作（`notify`/`notifyAll` 都不会）。

---

#### Q13. 什么是虚假唤醒？怎么避免？
**来源**：百度/携程 · **考核程度**：★★★☆☆

**参考答案**

**定义**：线程在没有被 `notify`/`notifyAll`/中断/超时的情况下，`wait()` 自行返回。

**根因**：POSIX 的 `pthread_cond_wait` 语义明确允许虚假唤醒（底层 futex/信号实现和调度都可能造成），JVM 透传了这个行为。JMM/JavaDoc 也明确写了"`wait` 应该总是在循环中调用"。

**三种"醒来但条件不成立"**：
1. 真正的 spurious wakeup；
2. 被 `notify` 了，但条件已被其它线程抢先消费；
3. 被 `notifyAll` 唤醒但不是自己的条件。

**避免方式**：**用 `while` 而不是 `if`**：

```java
synchronized (lock) {
    while (!condition) {          // ✅ 醒来必须重新检查
        lock.wait();
    }
    // 到这里条件一定成立
}
```

**延伸**：`ReentrantLock` 的 `Condition.await()` 同样要求 while（AQS 也不保证不虚假唤醒）。凡是"条件等待"都必须循环复查，这是并发编程的通用规则，不是 Java 的怪癖。

---

#### Q14. 用 synchronized 写一个死锁，并说明如何排查
**来源**：阿里/美团（现场手写）· **考核程度**：★★★☆☆

**参考答案**

**手写死锁**：
```java
Object A = new Object(), B = new Object();

// 线程1：先 A 后 B
new Thread(() -> { synchronized (A) { sleep(100); synchronized (B) {} } }).start();
// 线程2：先 B 后 A
new Thread(() -> { synchronized (B) { sleep(100); synchronized (A) {} } }).start();
```

**四个必要条件**：互斥、请求并保持、不可剥夺（`synchronized` 无法被抢占）、循环等待。

**排查手段**：
1. `jps -l` 找 pid → `jstack <pid>`：
   ```
   Found one Java-level deadlock:
   "Thread-0": waiting to lock monitor 0x00007f...(object 0x0000000..., a java.lang.Object),
              which is held by "Thread-1"
   "Thread-1": waiting to lock monitor ..., which is held by "Thread-0"
   ```
   会直接打印 `Found one Java-level deadlock`，无需自己分析。
2. `jconsole` / `VisualVM` → "检测死锁"按钮；
3. `jcmd <pid> Thread.print`（JDK 9+ 等价于 jstack）；
4. `ThreadMXBean.findDeadlockedThreads()` 做线上自动检测告警。

**预防**：**固定加锁顺序**（按对象 hash 或业务 ID 排序）、用 `tryLock(timeout)` 替代（`ReentrantLock` 特有）、缩小锁粒度、避免在持锁时调用外部方法（可扩展性死锁）。

---

### L3 高级深挖

#### Q15. 轻量级锁的加锁过程？重入是怎么处理的？
**来源**：字节跳动/蚂蚁 二面 · **考核程度**：★★★★☆（区分"背过"和"真懂"的分水岭）

**参考答案**

**加锁（三步）**：
```cpp
// ObjectSynchronizer::slow_enter 的 trivial 分支
markOop mark = obj->mark();
if (mark->is_neutral()) {                       // ① 确认无锁
    lock->set_displaced_header(mark);            // ② 备份原 Mark Word 到栈上 Lock Record
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, &obj->mark, mark)) {  // ③ CAS
        return;   // 成功 → Mark Word = LockRecord 地址(low2=00)
    }
    // CAS 失败 → inflate 成 ObjectMonitor
}
```

**关键点：Lock Record 在线程栈上**（`BasicObjectLock`，含 `_displaced_header` + `_obj`），每个 `synchronized` 块对应一个，JIT 会静态算 `max_monitors` 预留栈空间。

**重入处理（精华）**：
```java
synchronized (this) {          // 层1: LockRecord1.displaced_header = 原 MarkWord
    synchronized (this) {      // 层2: LockRecord2.displaced_header = NULL, 且【不做 CAS】
        ...
    }
}
```
- 第 2 层为什么不能 CAS：Mark Word 现在是 `lock=00` 指向 LockRecord1，若再 CAS 成 LockRecord2，**外层解锁时就找不到还原值了**（原来的 Mark Word 只存在 LockRecord1 里）。
- 于是用 **`displaced_header == NULL` 当作"重入标记"**：解锁时见到 NULL 就直接弹栈、不碰 Mark Word；见到非 NULL 才说明是最外层，执行 `CAS(MarkWord, lock, displaced_header)` 还原。

**解锁失败的含义**：
```cpp
if (!CAS(&obj->mark, lock, displaced_header)) {
    // 说明我持有期间有人来过：它 CAS 失败 → 已经把 Mark Word 膨胀成 monitor 指针
    monitor->exit(true);   // 接手重量级锁的释放逻辑，唤醒后继
}
```

**一句话总结**："轻量级锁本质是**把对象头的 8 字节搬到线程栈上，再用一个指针把它指回来**；重入靠 displaced_header 置 NULL 做标记。"

---

#### Q16. 重量级锁的 ObjectMonitor 里，_cxq、_EntryList、_WaitSet 分别是什么？
**来源**：阿里/字节/美团 三面 · **考核程度**：★★★★★（深水区，能答全的人不多）

**参考答案**

| 字段 | 类型 | 存谁 | 进出方式 |
|---|---|---|---|
| `_cxq` | 单向链表 | **刚来竞争失败的线程**（contention queue） | CAS **头插**（LIFO），O(1) |
| `_EntryList` | 双向链表 | 被选中待唤醒的竞争者 | owner 释放时从 `_cxq` 转移（按 QMode） |
| `_WaitSet` | 双向链表 | **调用过 `wait()` 的线程** | `wait` 入、`notify`/`notifyAll`/中断/超时出 |

**它们描述的是一条线程的一生**：
```
新线程 → CAS _owner 失败 → 头插 _cxq → (owner 释放，_cxq 转移) → _EntryList
       → unpark 唤醒 → 再 CAS _owner → 成为 owner → 执行临界区
       → 若调 wait() → 释放 owner → 挂到 _WaitSet → park
       → notify → 移到 _EntryList → 重新竞争 → 拿到锁 → 从 wait 返回
```

**为什么 `_cxq` 用头插（LIFO）**：新到的线程"最热"（缓存行、分支预测都是热的），让它先试一把成功率更高；头插 O(1)，不用遍历；而**公平性交给 EntryList 的 FIFO** 来补。这个 LIFO + FIFO 混合是 HotSpot 有意为之的"吞吐优先"设计。

**QMode 策略**（owner 释放时怎么把 cxq 转过去）：

| QMode | 动作 | 效果 |
|---|---|---|
| 0（默认） | cxq 全量转移至 EntryList，唤醒头部 | 简单，但可能产生无效唤醒 |
| 1 | 只摘 cxq 头唤醒 | 少唤醒 |
| 2 | 优先唤醒 `_succ`（假定继承者） | 避免惊群，近似"直接交接" |
| 4 | 直接唤醒 EntryList 头 | 最省 |

**延伸（`_succ` 的作用）**：`_succ` 是"假定继承者"字段。owner 释放时如果认定某线程会接手，就把它记为 `_succ` 并直接 unpark；被唤醒的线程如果抢不到锁，会**自旋等待**（防守型自旋），尽量不让"票"浪费。这是 HotSpot 缓解"非公平导致的饥饿 + 无效唤醒"的机制。

---

#### Q17. 为什么锁升级是不可逆的？不能降级吗？
**来源**：蚂蚁/字节/腾讯 三面 · **考核程度**：★★★★☆

**参考答案**

先纠正一个前提：**"不可逆"是 HotSpot 的设计选择，不是物理限制**（代码里有 `deflate` 能把 monitor 收回、Mark Word 还原成无锁）。

**不做的 5 个原因**：
1. **判定成本**：要判断"现在没竞争了"，需要持续采样冲突率/队列长度，采样本身要原子操作或加锁，比省下的开销更贵。
2. **状态还原风险**：轻量级锁的信息（Lock Record 指针）和重量级锁的信息（monitor 的 `_header`）在不同位置，降级要按严格顺序还原 Mark Word；任何一步被其它线程的 CAS 插进来就会破坏一致性（Mark Word 是 8 字节，**只有一个字的原子性保证**）。
3. **震荡成本**：高竞争场景下降级→马上又升级，来回抖动比"一直重量级"更慢。
4. **信息丢失**：偏向锁的 epoch、hash 已在膨胀过程中被搬走/覆盖，回退要知道"原来是不是偏向的"，成本高。
5. **收益不确定**：大量实测表明，"升过级的锁"绝大多数会继续有竞争——这是经验性假设，也是 HotSpot 优化资源的分配依据。

**唯一的"半降级"**：`MonitorDeflationThread`（JDK 9+）周期性调用 `deflate_idle_monitors`，把**空闲** monitor 归还 `gFreeList` 池，并把 Mark Word 从 monitor 指针还原为 `lock=01` 的无锁态。注意这是**资源回收**，由后台线程按 `GuaranteedSafepointInterval`（默认 1000ms 量级）触发，不在热路径上。

**答题加分**："严格说不是不可逆，而是不做主动降级；`deflate` 会把它变回无锁，但那是空闲回收，不改变'竞争过就大概率继续竞争'的判断。"

---

#### Q18. 锁消除和锁粗化是什么？说说逃逸分析
**来源**：阿里/美团/百度 二面 · **考核程度**：★★★★☆

**参考答案**

**锁消除（Lock Elision）**
```java
StringBuffer sb = new StringBuffer();   // 局部变量，未逃逸
sb.append(a).append(b);                  // append 是 synchronized 方法
```
C2 的逃逸分析判定 `sb` **不逃逸**（没被写入字段、没传给未知方法、没被 lambda/内部类捕获、没被 return），于是**整个 monitorentor/monitorexit 被删除**。
- 开关：`-XX:+DoEscapeAnalysis`（默认开）+ `-XX:+EliminateLocks`（默认开，**仅 C2**）。
- 注意：**解释执行和 C1 编译时仍会加锁**，所以小样本压测（还没 JIT 完）会看到"锁还在"。

**逃逸分析的三种结论**：
| 结论 | 含义 | 优化 |
|---|---|---|
| NoEscape | 完全不出方法 | 栈上分配、标量替换、**锁消除** |
| ArgEscape | 作为参数传出但方法内不改 | 部分优化 |
| GlobalEscape | 逃到堆/全局/其它线程 | 无优化 |

**锁粗化（Lock Coarsening）**
```java
for (int i = 0; i < 1000; i++) synchronized (lock) { count++; }
// C2 可能重写为：
synchronized (lock) { for (int i = 0; i < 1000; i++) count++; }
```
条件：**锁对象相同 + 相邻 + 中间无逃逸/异常边界 + 粗化后临界区不过大**。如果两个临界区之间隔着重计算，粗化会让持锁时间过长 → 不做。

**延伸对比**：
- 锁消除回答"**没必要加的锁**"，锁粗化回答"**加得太碎的锁**"；
- 它们都是 **JIT 编译期行为**，是"同一段代码在解释器/C1/C2 下锁行为不同"的根源，也是为什么"基准测试必须用 JMH 并预热"。

---

#### Q19. synchronized 底层是怎么保证原子性和有序性的？内存屏障插在哪？
**来源**：阿里/字节 三面 · **考核程度**：★★★★☆

**参考答案**

**原子性**：靠互斥（同一时刻只有一个线程能成为 `_owner`）。注意它保证的是"**临界区整体**对同一把锁的其它线程原子"，不是"任意代码原子"。

**有序性 + 可见性**：靠 JSR-133 的锁规则 + 编译期插入的屏障。

```
        monitorenter
        ┌─────────────┐
        │ LoadLoad    │  ← acquire: 禁止后面的 load/store 被重排到锁之前
        │ LoadStore   │
        └─────────────┘
           临界区代码
        ┌─────────────┐
        │ StoreStore  │  ← release: 保证临界区内的写对其它核可见
        │ StoreLoad   │  ← 禁止临界区内的写被重排到 unlock 之后
        └─────────────┘
        monitorexit
```

**在 x86（TSO）上的实际落地**：
- acquire 侧**不需要指令**（TSO 保证 load 不越过更早的 load/store）；
- release 侧需要一条 `lock` 前缀（HotSpot 用 `lock addl $0,(%rsp)` 做 full fence）；
- 轻量级锁的 `lock cmpxchg` 自带 full barrier，所以这一层"免费"。

**在 ARM/AArch64 上**：需要 `ldar`（acquire）/`stlr`（release）或 `dmb ish`，成本明显高于 x86——这是"同一段并发代码在 ARM 服务器上更慢"的物理原因。

**必须强调的边界**：屏障只"锁边界"，不"锁内部"。锁内 `a=1; b=2;` 之间仍可重排，但它们不会被提出锁外。

---

#### Q20. synchronized 是非公平锁吗？为什么？能改公平吗？
**来源**：美团/快手 三面 · **考核程度**：★★★★☆

**参考答案**

**是非公平的，且不可配置。**

**非公平在哪**：`ObjectMonitor::exit` 里 `release_store(&_owner, NULL)` 之后，**新来的线程可以直接 CAS 抢到 `_owner`**（这叫 barging，插队），而 `_cxq` / `_EntryList` 里排队的线程还在等 unpark。所以排队者可能被持续插队（极端情况下的**饥饿**）。

**为什么设计成非公平**：
1. **吞吐优先**：插队线程通常"热"（刚被调度、缓存热、代码热），让它直接跑比唤醒一个冷线程快；唤醒 + 上下文切换的成本可能超过"短暂的不公平"。
2. **避免 convoy（护航效应）**：严格 FIFO 下，每次释放都要唤醒队首、等它调度上来，锁的实际空闲时间被拉长。
3. **实现简单**：公平需要维护有序队列 + 每次释放都精确唤醒，成本高。
4. **有防御机制**：`_succ` "假定继承者"+ 被唤醒者的防守性自旋，缓解严重饥饿。

**能不能改公平**：`synchronized` **没有公平模式**（无参数可调）。需要公平用 `new ReentrantLock(true)`（AQS 的 `hasQueuedPredecessors()` 检查队列）。

**延伸**：这是一种典型的"**吞吐 vs 延迟**"取舍——非公平的平均吞吐更高，但**尾延迟（P99）更差**，这也是为什么延迟敏感的场景（如交易撮合）倾向 `ReentrantLock(true)` 或自研排队锁。

---

#### Q21. 自旋锁和自适应自旋是怎么回事？自旋次数是多少？
**来源**：京东/滴滴 二面 · **考核程度**：★★★☆☆

**参考答案**

**为什么自旋**：park 挂起/唤醒 = 两次上下文切换 + 内核调度（μs 级），若锁只被占用几十条指令，**自旋比挂起划算**。

**自旋多少次——答案是"没有固定次数"**：
- JDK 6 早期有 `-XX:PreBlockSpin=10`（固定 10 次），**已废弃**；
- 现代 HotSpot 的 `ObjectMonitor::TrySpin` 用**自适应**：由该 monitor 的 `_SpinDuration` 控制，**历史自旋成功率高 → 允许更长**，失败率高 → 缩短（可降到 0 直接 park）；
- C2 版本（`C2i`/`C2` 的 `MacroAssembler` 优化）更激进：会检查 owner 是否"正在运行"（`_succ` 未设 + owner running 才自旋），否则直接放弃自旋去排队，避免"锁 convoy"。

**关键实现细节**：现代 HotSpot 里，**自旋发生在 `ObjectMonitor::enter()` 内部**，而 Mark Word 早在 `slow_enter` 的 CAS 失败时就已膨胀成 `lock=10`。所以"轻量级锁自旋后升级重量级"的说法，在实现上其实是"升级后自旋，失败才 park"（见第 3.3.5 节的辨析）。

**延伸**：`-XX:+UseSpinning` 参数早已废弃（自适应自旋默认开启）。自旋的前提是**多核**——单核上自旋毫无意义（owner 根本没机会跑）。

---

#### Q22. synchronized 会造成线程饥饿吗？什么情况下会发生？
**来源**：字节/蚂蚁 三面 · **考核程度**：★★★★☆

**参考答案**

**会，但概率低于 `ReentrantLock` 的非公平模式极端场景**，因为 HotSpot 有 `_succ` + 防守型自旋的缓解机制。

**饥饿的三条成因**：
1. **barging（插队）**：owner 置空瞬间，新线程直接 CAS 抢走，排队的线程收不到 unpark 或收到后抢不到。
2. **唤醒丢失**：owner 释放时 `_succ` 失效 → unpark 了一个已经被中断/超时的线程 → 队列空转。
3. **长临界区 + 高频短临界区混跑**：短任务不断插队，长任务永远排在队尾。

**表现**：线程处于 `BLOCKED (on object monitor)` 长时间不动，吞吐看起来正常但某些请求超时——**用 `jstack` 连续采样，看某个线程的 `waiting to lock` 一直不变**。

**缓解/规避**：
- 缩短临界区、把 IO/锁外计算移出去；
- 拆锁（锁分段，如 `LongAdder` 的 cell 思路）、无锁化（CAS/`Atomic*`）；
- 需要强公平 → `ReentrantLock(true)`；
- JDK 21 虚拟线程下用 `ReentrantLock` 避免 pinning。

---

### L4 专家级 / 细节杀手

#### Q23. 一个对象调用 System.identityHashCode() 之后，还能用偏向锁吗？
**来源**：阿里/蚂蚁 专家面 · **考核程度**：★★★★★（几乎没人答对的"神来一问"）

**参考答案**

**不能，永久失去偏向能力。**

**原因**：Mark Word 是 8 字节的**复用**字段：

| 状态 | bit 布局 |
|---|---|
| 可偏向 | `thread ID(54) + epoch(2) + unused(1) + age(4) + biased_lock(1) + lock(2)` |
| 无锁 + hash | `unused(25) + hash(31) + unused(1) + age(4) + biased_lock(1) + lock(2)` |

`thread ID` 占 54 bit，`hash` 占 31 bit，**同一块内存装不下两个**。所以 HotSpot 规定：**对象一旦有了 identity hash，`has_bias_pattern()` 恒为 false**，之后所有加锁都从"无锁 → 轻量级"路径走。

**反向也成立**：如果对象当前正偏向某线程，此时调用 `System.identityHashCode()`，JVM 会先 `revoke` 偏向（可能有 safepoint），再把 hash 写进 Mark Word。

**注意区分**：
- `Object.hashCode()`：如果被业务重写 → 用的是自己的算法，**不占 Mark Word**，不影响偏向；
- 默认 `Object.hashCode()` / `System.identityHashCode()` → 用 Mark Word 的 31 bit，影响偏向；
- `hashCode()` 是**懒生成**的（第一次调用才写），所以 `new Object()` 刚创建时 hash 位是 0。

**延伸（这个例子最精彩）**：`HashSet`/`HashMap` 的 key 在插入时会调用 `hashCode()` → 一旦用了 identity hash，这些对象**永久不可偏向**。这就是"为什么拿业务对象当锁有时比拿专用 `Object` 当锁慢"的一种可能原因（虽然通常可忽略，但这个因果链非常能说明"底层细节如何影响上层"）。

**版本补充**：JDK 15+ 没有偏向锁，此问题退化为"hash 存在哪"，但答案的历史意义仍在。

---

#### Q24. 用 String / Integer 当锁对象有什么坑？
**来源**：腾讯/京东/携程 二面 · **考核程度**：★★★★☆（经典事故题）

**参考答案**

**三个坑，本质都是"锁对象被意外共享"**：

**坑 1：字符串常量池**
```java
// A 模块
synchronized ("order") { ... }
// B 模块（完全无关的业务）
synchronized ("order") { ... }   // ← 同一个 String 常量！全局互斥
```
字面量在常量池里**全局唯一**，两个不相关的代码会共享同一把锁，导致莫名其妙的互相阻塞。

更危险的是 **`String.intern()`**：
```java
synchronized (userId.intern()) { ... }   // 大量不同 userId → 常量池膨胀；
                                          // 相同值 → 意外串行；还有 Metaspace OOM 风险
```

**坑 2：Integer 缓存（-128~127）**
```java
Integer a = 127, b = 127;  // 同一对象（IntegerCache）
Integer c = 128, d = 128;  // 不同对象！
synchronized (a) {...}     // 与 synchronized(b) 互斥
synchronized (c) {...}     // 与 synchronized(d) 不互斥 → 锁失效
```
`Integer.valueOf()` 走缓存，`new Integer()` 不走；`Long/Short/Byte/Character` 同样。

**坑 3：自动装箱的隐形换对象**
```java
private Integer count = 0;
synchronized (count) { count++; }   // count++ 会重新装箱 → 锁对象变了！
                                    // 每次都锁在"旧的" Integer 对象上，形同无锁
```
同理 `Boolean`、`Long` 的 `++`/`+=`。

**正确做法**：
```java
private final Object lock = new Object();   // 专用、private、final、不可变引用
```
三条铁律：**private（防外部抢锁）、final（防指向被改）、专用（不与业务对象混用）**。
另外：**不要锁 `this`**（外部能拿到，第三方框架可能也锁它），**不要锁 `Class`**（全局唯一，容易跨模块互斥）。

---

#### Q25. 批量重偏向（bulk rebias）和批量撤销（bulk revoke）的触发条件与 epoch 机制
**来源**：蚂蚁/阿里 专家面 · **考核程度**：★★★★★

**参考答案**

**问题背景**：偏向锁撤销要进 safepoint，如果一个类的对象被反复"锁一下就走"，逐个撤销会让 JVM 不停进 safepoint → 抖动。所以要**批量处理**。

**计数记在哪**：记在**类（Klass）**上（`BiasedLockingCounters`），不是记在对象上。

| 阈值 | 默认值 | 行为 |
|---|---|---|
| `BiasedLockingBulkRebiasThreshold` | **20** | 该类累计撤销 ≥20 次 → **批量重偏向** |
| `BiasedLockingBulkRevokeThreshold` | **40** | 该类累计撤销 ≥40 次 → **批量撤销**（该类永久不可偏向） |
| `BiasedLockingDecayTime` | 25000 ms | 计数衰减窗口：超过这个时间没有新撤销，计数清零（防止慢速累积误判） |

**epoch 机制（核心）**：
- 每个类有一份 `_prototype_header`（原型对象头），里面有 2 bit 的 **`epoch` 值**；
- 每个实例的 Mark Word 也存了一份 epoch；
- **判定偏向有效**的条件是：`实例的 epoch == 类的 epoch`，且线程 ID 匹配。
- **批量重偏向** = 把类的 epoch +1，然后遍历该类所有实例把实例 epoch 也 +1（惰性：部分走 `markOop` 扫描，`bulk_rebias` 实际是遍历所有实例）。之后**老的偏向全部失效**（epoch 不匹配），但对象仍"可偏向"，新线程可以重新偏向。
- **批量撤销** = 把类的 `_prototype_header` 去掉 bias 位（`clear_bias_pattern()`），该类**所有**对象（含之后 new 的）都不可偏向；同时把现有实例的 Mark Word 直接恢复无锁。

**为什么要 epoch 而不是给每个对象打标记**：2 bit 的 epoch 是一个"廉价的时间戳"，让"全班作废"变成 O(1) 的类级操作 + 一次实例遍历；否则要维护对象级的撤销计数表，内存和复杂度都不可接受。

**典型触发场景**（面试可以举例）：
> 线程池中每个任务 `new` 一个对象然后 `synchronized(obj)`：任务1 由线程A 偏向，任务2 由线程B 来抢 → 撤销；累积到 20 次后批量重偏向，40 次后该类彻底不可偏向。**这正是"偏向锁在线程池场景下纯亏"的量化机制。**

---

#### Q26. 偏向锁撤销真的会导致 STW 吗？
**来源**：字节/阿里 专家面 · **考核程度**：★★★★★（最容易说错的一题）

**参考答案**

**要分精度说，不能简单答"会"或"不会"**：

1. **撤销动作必须在安全点执行**。因为要判断"偏向线程是否还在同步块内"，只能在该线程处于已知状态（safepoint）时读它的栈帧——否则栈帧可能正在变化。
2. 执行者是 `VM_RevokeBias` 这个 **VM Operation**，在 safepoint 下运行。但它的粒度是**逐个被标记的 JavaThread**（"handshake"式处理），只让**相关线程**停下来处理，而不是"整个 JVM 长时间停住"。
3. **真正的问题在高频撤销**：如果很多类都在持续撤销，JVM 会**反复进出 safepoint**（每次 safepoint 都是全局停顿，通常几十 μs，但频繁发生会显著拉高 P99）。同时 `bulk_rebias` 需要遍历该类的**所有实例**，对象多时是 O(n) 的停顿。
4. **所以合理的表述是**：偏向锁撤销 = "**在 safepoint 下执行的一次小范围停顿**"，成本取决于撤销频率和实例数量，高频时表现为**全局 STW 抖动**，不是"每次撤销都 STW 很久"。

**加分句**："这也是 JDK 15 默认关闭偏向锁（JEP 374）的直接理由之一：撤销依赖 safepoint 的机制与高并发、低延迟的目标冲突。"

---

#### Q27. 同样是阻塞，synchronized 的 park 和 Object.wait 有什么区别？
**来源**：美团/滴滴 专家面 · **考核程度**：★★★★★

**参考答案**

两者底层都落到 `ParkEvent::park`（Linux 上 `pthread_mutex_lock` + `pthread_cond_wait` → `futex`），但**语义层完全不同**：

| 维度 | `monitorenter` 竞争失败 → park | `Object.wait()` → park |
|---|---|---|
| 挂到哪 | `_cxq` / `_EntryList`（**竞争**队列） | `_WaitSet`（**条件**队列） |
| 唤醒者 | owner 释放锁时（`exit`） | `notify` / `notifyAll` / 中断 / 超时 |
| 是否需持锁 | 不需要（就是想拿锁） | **必须**持锁 |
| 醒来后 | 直接尝试 CAS `_owner` | 从 `wait` 返回前**必须重新竞争锁** |
| 是否可中断 | ❌ 不可中断（等锁不吃 interrupt） | ✅ 抛 `InterruptedException` |
| 语义 | "我要锁，让一下" | "条件不满足，我先让锁，等通知" |

**为什么 weight 锁用 park 而不是 wait**：`park/unpark` 是**许可模型**（permit 可提前累积 1 个），不会丢唤醒；`wait/notify` 是**必须持锁 + 随时可能丢信号**的条件变量，不适合"精确唤醒某个竞争线程"的场景。

**延伸（面试官爱追）**："为什么 `synchronized` 等锁不可中断，而 `ReentrantLock.lockInterruptibly` 可以？"——因为 JVM 的 `ObjectMonitor::enter` 走的是 `park` 的**不可中断模式**（`ParkEvent` 的 interrupt 标志被 `Thread.interrupted` 清掉后忽略），设计上认为"内置锁的语义应当简单、不可打断"；而 AQS 在 `park` 外层循环里检查 `Thread.interrupted()`，主动抛 `InterruptedException`。

---

#### Q28. 一个对象什么时候才真正创建 ObjectMonitor？它什么时候被销毁？
**来源**：阿里/字节 三面 · **考核程度**：★★★★☆

**参考答案**

**创建：懒膨胀（lazy inflate）**
- `new Object()` **不创建** monitor，Mark Word 只是无锁。
- 只有 `ObjectSynchronizer::inflate()` 被调用时才创建（CAS 失败/需要阻塞/调用 wait）。实现上从全局空闲链表 `gFreeList` 取一个复用（`omAlloc`/`omRelease`），所以**创建开销通常只是"从池里摘一个 + 复位字段"**。
- 对象 → monitor 的映射由 Mark Word 存指针，**monitor 不回指对象**（`_obj` 有，但用 weak 语义）。

**销毁：Deflation（收缩）**
- JDK 9+ 有专门的 `MonitorDeflationThread`（不再挂在 VMThread 上），周期性在 safepoint 下执行 `deflate_idle_monitors`：
  - 条件：`_owner == NULL` 且队列为空且没有 `wait` 线程（"idle"）；
  - 动作：把 Mark Word 从 monitor 指针**还原为无锁**（`lock=01`），monitor 归还 `gFreeList`；
  - 参数：`-XX:GuaranteedSafepointInterval`（默认 1000ms，控制收缩周期）、`-XX:MonitorDeflationMax`（单次最多收缩个数）。
- **不销毁的后果**：monitor 是 C++ 堆对象，长期持有会占用 native memory；大量膨胀过的对象会让 native 内存上涨（这也是 `jcmd VM.native_memory` 里能看到的一类）。

**延伸**：这解释了"为什么 `synchronized` 的对象多了会看到 native 内存增长"、"为什么 `-XX:GuaranteedSafepointInterval` 调大会增加 native 内存"。也说明"每个对象都自带 monitor"是错的——monitor 是**竞争时才出现、空闲时被回收**的。

---

#### Q29. 为什么"在方法里写 synchronized"可能压根不加锁？
**来源**：字节/快手 三面 · **考核程度**：★★★★☆

**参考答案**

因为**解释器、C1、C2 三者的锁行为不同**，而 `synchronized` 的消除/粗化**只有 C2 做**：

| 执行方式 | 锁行为 |
|---|---|
| 解释器（`-Xint`） | 每条 `monitorenter` 都真实执行，走 `fast_enter` |
| C1（Client，`-XX:TieredStopAtLevel=1`） | 基本保留锁，可能有简单优化 |
| C2（Server，默认最终层级） | **逃逸分析 → 锁消除**；**相邻合并 → 锁粗化**；内联后可能整块消失 |

所以：

```java
public String m(String a) {
    StringBuffer sb = new StringBuffer();  // 未逃逸
    sb.append(a);                           // 解释执行时真的加锁
    return sb.toString();                   // C2 编译后：锁没了
}
```

**工程含义（这才是面试官想听的）**：
1. **基准测试必须预热 + 用 JMH**，否则测的是解释器/C1 的锁行为，和线上不一致；
2. **"我看到锁还在"多数是没 JIT 完**，可以用 `-XX:+PrintCompilation`、`-XX:+PrintInlining`、`-XX:+UnlockDiagnosticVMOptions -XX:+PrintEliminateLocks`（部分版本）验证；
3. 反过来说，**不要指望锁消除救你**——一旦对象逃逸（存进字段、传给别人、被 lambda 捕获），锁就老老实实存在。正确做法是**用 `StringBuilder` 而不是 `StringBuffer`**，而不是指望 JIT。

---

#### Q30. synchronized 在高并发下到底慢在哪？给一个能量化的解释
**来源**：阿里/美团 三面 · **考核程度**：★★★★☆

**参考答案**

按代价从低到高排序（这是回答的骨架）：

| 场景 | 代价量级 | 瓶颈 |
|---|---|---|
| 偏向锁命中（JDK≤14） | ~ns（一次比较） | 无 |
| 轻量级锁 CAS 成功 | ~10-20 ns | cache line 独占（RFO） |
| CAS 自旋失败（竞争） | ~100 ns ~ μs | **缓存一致性流量**（line 在核间乒乓） |
| park/unpark（阻塞） | ~1-10 μs | **两次上下文切换 + 内核调度** |
| 严重竞争 + 队列 convoy | 数十 μs ~ ms | 调度延迟累积、CPU 空转、虚假唤醒 |
| 跨 NUMA / 伪共享 | 可放大 5-10 倍 | 跨 socket 内存访问 |

**三个可量化的机制**：
1. **缓存乒乓**：`lock cmpxchg` 要让 cache line 处于独占，每次换核都要 RFO + 写回 → 高竞争下**吞吐随核数增加而下降**（负扩展）。
2. **上下文切换**：Linux 一次 `futex_wait`/`wake` 约 1-5 μs，若每秒百万次竞争 → 光切换就吃掉几个核。
3. **convoy 效应**：每次释放只唤醒一个、被唤醒者要等调度 → 临界区利用率下降，队列越排越长。

**优化手段（对应上面的瓶颈）**：
- 减少共享写（`LongAdder` 分段计数 > `AtomicLong` > `synchronized`）；
- 缩短临界区，把 IO/日志/序列化移出锁外；
- 降低锁粒度（锁分段、`ConcurrentHashMap` 的桶级 CAS）；
- 无锁化（CAS + 重试）、避免伪共享（`@Contended`）；
- 读多写少 → `StampedLock` / `ReentrantReadWriteLock`。

---

### L5 场景设计与开放题

#### Q31. 双重检查锁单例为什么必须加 volatile？只加 synchronized 不行吗？
**来源**：阿里/腾讯/京东（**经典必考**）· **考核程度**：★★★★☆

**参考答案**

**先看 `new` 不是原子操作**。`instance = new Singleton()` 在字节码里是三步：
```
① new           # 分配内存，得到引用（此时字段全是默认值 0/null）
② invokespecial # 调用构造方法，初始化字段
③ putstatic     # 把引用写入静态字段 instance
```
**② 和 ③ 之间允许重排序**（JMM 只要求"线程内 as-if-serial"，而这两步之间没有数据依赖……严格说是有依赖的，但 JIT/CPU 在"构造未完成的对象引用可以先发布"这一点上被允许，因为其它线程看不到中间状态除非引用被发布了）。

于是：
```
线程A: ① ②(重排后) ③ → A 看来对象已完整
线程B: 第一次判空 if (instance == null) → 读到【非 null 但未初始化】的对象
       → 直接 return instance → 使用未初始化对象 → NPE / 脏数据
```

**为什么 synchronized 救不了**：因为第一次判空**在锁外**。锁内 unlock → 锁外 lock 之间没有 happens-before（不是同一把锁的配对），所以锁外的读可以看见锁内的重排中间状态。

**正确写法**：
```java
private static volatile Singleton instance;   // ★ volatile 不可省

public static Singleton getInstance() {
    if (instance == null) {                    // 第一次检查（锁外，靠 volatile 的 acquire 语义）
        synchronized (Singleton.class) {
            if (instance == null) {            // 第二次检查（锁内）
                instance = new Singleton();    // volatile 写 = release，保证 ①② 不会排到 ③ 之后
            }
        }
    }
    return instance;
}
```

**更优雅的方案**（面试可以顺带说，显示视野）：
- **静态内部类（Holder）**：靠类加载的初始化锁（`<clinit>` 由 JVM 保证线程安全 + happens-before），**天然无锁且无需 volatile**：
  ```java
  private static class Holder { static final Singleton INSTANCE = new Singleton(); }
  public static Singleton getInstance() { return Holder.INSTANCE; }
  ```
- **枚举单例**：防反射、防序列化破坏（《Effective Java》推荐）。

---

#### Q32. 单机多线程累加 1000 万次，用 synchronized、AtomicLong 还是 LongAdder？
**来源**：阿里/字节 场景题 · **考核程度**：★★★★☆

**参考答案**

**先给结论**：`LongAdder > AtomicLong >> synchronized`（在**高竞争**下），但**低竞争**时 `AtomicLong` 可能反超 `LongAdder`。

**逐个分析**：

| 方案 | 原理 | 高竞争瓶颈 | 适用 |
|---|---|---|---|
| `synchronized` | monitor 互斥，串行化 | park/unpark + 缓存乒乓，吞吐近似"单线程" | 需要复合操作/多变量一致性 |
| `AtomicLong` | 一个 `volatile long` 上 CAS 自旋 | **单点 CAS 失败风暴**：N 个线程抢同一 cache line，重试率随核数上升 | 低~中竞争、需要精确读写 |
| `LongAdder` | **分段累加**：`base + Cell[]`，线程按 probe 散列到不同 cell | 几乎无（不同 cell 在不同 cache line，`@Contended` 防伪共享），sum 时合并 | 高并发计数、只写不读/最终一致 |

**LongAdder 的实现要点**（能讲出来就是加分）：
```java
// 写：先试 base（无竞争时）；失败则按 hash 找 Cell，CAS 到自己的槽
// Cell 用 @Contended 填充，避免伪共享
// sum()：base + 遍历 cells 累加 —— 弱一致（并发时可能不准）
```
- **代价**：`sum()` 是 O(n cells) 的，且**并发下不精确**；内存更大。
- 所以"需要精确实时读"→ `AtomicLong`；"只要最终汇总"（监控计数、QPS 统计）→ `LongAdder`。

**实测经验值**（供面试时给量级感，不同机器差异大）：
- 8 线程 1000 万次：`synchronized` 数百 ms ~ 秒级；`AtomicLong` 数十~百 ms；`LongAdder` 几~十几 ms。

**答题加分**："如果还要'每 1000 次批量刷一次'，可以用 `LongAdder` + 定时 `sumThenReset`，比每次读 `AtomicLong` 更省。"

---

#### Q33. 秒杀扣库存，synchronized 怎么用？怎么优化？
**来源**：阿里/美团/拼多多 场景题 · **考核程度**：★★★★★（开放题，考察工程判断）

**参考答案**

**分层给方案，从"能用"到"好用"**：

**① 单机版（最小可用）**
```java
private final Object lock = new Object();      // 专用锁对象，不用 this/String
public boolean deduct(Long skuId) {
    synchronized (lock) {                       // 全库一把锁 → 吞吐极低
        int stock = stockMap.get(skuId);
        if (stock <= 0) return false;
        stockMap.put(skuId, stock - 1);
        return true;
    }
}
```
问题：锁粒度 = 全库，不同 SKU 互相阻塞。

**② 锁分段 / 按 key 分锁（分段锁思想）**
```java
private final Object[] locks = new Object[64];   // 固定桶
private Object lockFor(Long skuId) { return locks[(int)(skuId ^ (skuId >>> 32)) & 63]; }

public boolean deduct(Long skuId) {
    synchronized (lockFor(skuId)) {               // 同一 SKU 互斥，不同 SKU 并发
        ...
    }
}
```
注意：**哈希冲突时不同 SKU 会共锁**（可接受）；`ConcurrentHashMap.computeIfAbsent`/`compute` 本身就是"桶级 synchronized + CAS"，等价但更成熟。

**③ 无锁（CAS）**
```java
// AtomicInteger 或 LongAdder 不适合"判断+扣减"，用 CAS 循环
while (true) {
    int cur = stock.get();
    if (cur <= 0) return false;
    if (stock.compareAndSet(cur, cur - 1)) return true;   // 失败重试
}
```
优点：无阻塞，低竞争极快；缺点：高竞争下 CAS 空转、**无法保证 FIFO**（可能有人一直抢不到）。

**④ 数据库层（最终防线）**
```sql
UPDATE stock SET count = count - 1 WHERE sku_id = ? AND count > 0;
-- 影响行数 = 0 即扣减失败（乐观锁 + 原子判断）
```
用 DB 的行锁/条件更新做"最终一致"，前面用 Redis（`DECR` 原子操作）+ 本地缓存挡流量。

**⑤ 分布式场景**
`synchronized` **只在本 JVM 内有效**——多实例部署时完全无效，必须用 Redis 分布式锁 / DB 行锁 / 消息队列串行化。**这是本题最重要的考核点**，一定要主动说出来。

**完整架构（满分答案的骨架）**：
```
请求 → 网关限流 → Redis 预扣（DECR，原子）→ 本地缓存库存（减少 Redis 压力）
     → MQ 异步下单（削峰，串行化到单分区保证顺序）
     → DB 条件更新（最终校验）→ 失败则回滚 Redis
```
**核心思路**：`synchronized` 只能做"单机临界区"，秒杀真正的解法是"**把并发削成串行 + 多层校验**"，而不是"把锁调得更快"。

---

#### Q34. 线上 CPU 100% 或接口突然变慢，怀疑是锁竞争，你怎么排查？
**来源**：阿里/美团 现场排查题 · **考核程度**：★★★★☆

**参考答案**

**第一步：确认是锁而不是别的原因**
```bash
top -Hp <pid>                  # 找最耗 CPU 的线程（tid）
printf '%x\n' <tid>            # 转十六进制
jstack <pid> | grep -A20 <hex> # 看该线程在干什么
```
- 若栈里是 `Object.wait()` / `park` 且 CPU 不高 → 不是 CPU 问题；
- 若栈顶是 `ObjectSynchronizer::enter` / `ObjectMonitor::TrySpin` / `Unsafe_Park` → 锁相关；
- 若栈顶是业务代码且在 `RUNNABLE` 反复出现 → **自旋**（轻量级锁反复 CAS 失败）。

**第二步：看锁的统计**
```bash
jstack -l <pid>        # 看 locked/ waiting to lock 的对象
jcmd <pid> Thread.print
```
或开 JFR：
```bash
jcmd <pid> JFR.start duration=60s filename=/tmp/rec.jfr
jcmd <pid> JFR.stop
# 用 JMC 看 "Java Monitor Blocked" / "Java Monitor Enter" 事件
```
JFR 的 `jdk.JavaMonitorEnter` 事件能直接给出**每个锁的阻塞次数和总阻塞时长排行**——这是最快的定位方式。

**第三步：对照原因**
| 现象 | 可能原因 |
|---|---|
| 大量线程 `BLOCKED (on object monitor)` 同一个对象 | 热点锁，粒度过大 |
| 大量 `WAITING (on object monitor)` 且无进展 | `notify` 误用（应用 `notifyAll`）/ 丢失信号 |
| `Found one Java-level deadlock` | 死锁，看 jstack 报告 |
| CPU 高但线程都是 RUNNABLE 且栈在 CAS 循环 | 自旋/活锁，高竞争 |
| 线程数暴涨 + 每个都等锁 | 线程池 + 阻塞锁 → convoy |

**第四步：修复方向**
- 缩小临界区（把 IO/日志/序列化/远程调用移出去）；
- 换锁（`ReentrantLock.tryLock(timeout)` 防死等、`StampedLock` 读优化、`LongAdder` 分段）；
- 降竞争（分片、无锁、批量合并）；
- 加监控（`ThreadMXBean` 检测死锁、JFR 常开、锁等待耗时打点）。

**加分**："如果是 `synchronized` 且发现是 JDK 8 且锁对象被大量撤销，可以试 `-XX:-UseBiasedLocking` 关掉偏向锁对比（这在 JDK 15+ 已经默认关闭）。"

---

#### Q35. 能用 synchronized 做分布式锁吗？
**来源**：腾讯/字节/美团 · **考核程度**：★★★★☆（送分题，但必须说清边界）

**参考答案**

**不能，`synchronized` 的作用域严格限定在单个 JVM 进程内。**

**原因**：
- `synchronized` 的互斥依赖**同一个堆里的同一个对象的 Mark Word / ObjectMonitor**；不同 JVM 有各自的堆，对象互不相干；
- 多实例部署（K8s 多副本、集群）时，每个实例都能"拿到锁"，等于没锁。

**分布式锁的三条硬性要求**（顺带答出来显得专业）：
1. **互斥**：跨进程生效；
2. **防死锁**：持锁者崩溃要能自动释放（TTL / 租约）；
3. **防误删**：只有持有者能释放（唯一 token + Lua 原子校验）。

**正确做法**：
| 方案 | 实现 | 特点 |
|---|---|---|
| Redis（SETNX + EXPIRE，或用 Redisson） | `SET key uuid NX PX 30000` + Lua 释放 | 性能高；需处理主从切换丢锁（Redlock 有争议） |
| ZooKeeper | 临时顺序节点 + watch | CP，强一致，性能较低 |
| 数据库 | 唯一索引 / `SELECT ... FOR UPDATE` | 简单可靠，性能差，适合低频 |
| 框架 | `ShedLock`、`Curator` | 封装好，注意 TTL 与重入 |

**延伸（面试官爱追）**：Redis 分布式锁的经典坑：
- **锁超时 vs 业务超时**：业务执行超过 TTL → 锁被释放 → 两个线程同时进临界区（需"看门狗"续期，Redisson 的 `ReentrantLock` 就是这么做的）；
- **主从切换丢锁**：master 写入后未同步就宕机，slave 升主后锁丢失 → 需 Redlock 或"fencing token"（在存储层校验 token 单调递增）；
- **释放误删**：必须 Lua 脚本"比较 token 再删除"，不能先 get 再 del。

**一句话总结**："`synchronized` 管进程内，分布式锁要跨进程；两者不是替代关系，而是不同层级——通常先做进程内 `synchronized`/`ReentrantLock` 保护本地状态，再做分布式锁保护跨实例资源。"

---

## 第 10 章 面试现场：追问链路与答题模板

### 10.1 最常见的追问链路（一条链问到底）

面试官通常顺着一条链往下挖，**每一层的答案决定下一层的问题**：

```
Q: synchronized 原理？
   → A: 字节码 monitorenter/monitorexit + JVM 层面 Monitor
      ↓ 追问 1
Q: 那 Monitor 到底是个什么结构？
   → A: ObjectMonitor（_owner/_recursions/_cxq/_EntryList/_WaitSet）
      ↓ 追问 2
Q: 为什么要有这么多队列？
   → A: 竞争排队（cxq/EntryList）与条件等待（WaitSet）语义不同
      ↓ 追问 3
Q: 那对象头里的 Mark Word 怎么和 Monitor 对应？
   → A: lock 位 10 + 指针指向 ObjectMonitor
      ↓ 追问 4
Q: 从无锁到重量级，中间经历了什么？为什么要有偏向锁/轻量级锁？
   → A: 锁升级链 + "没人抢/抢得少/真在抢"的三种假设
      ↓ 追问 5（杀手）
Q: 那为什么 JDK 15 把偏向锁废弃了？
   → A: 撤销依赖 safepoint、高并发下抖动、现代场景假设不成立、阻碍对象头压缩
```

**准备建议**：把这条链背成"一条故事线"，而不是散点知识。任何一环断了，面试官会换一条链（比如转向"内存语义"或"虚拟线程"）。

### 10.2 答题模板（把散点组织成结构）

**三层法**（推荐用于"原理题"）：

```
第 1 层：语法/字节码 —— 一句话说清"代码变成了什么"
第 2 层：JVM 运行时 —— 说清"状态怎么存、怎么变"（Mark Word + Monitor）
第 3 层：硬件/OS —— 说清"最终靠什么原子操作、什么系统调用"
```

**对比法**（推荐用于"区别题"）：**相同点 → 差异维度表 → 选型结论 → 边界/坑**。只答差异不答"什么时候用哪个"，会被判定为背书。

**边界法**（推荐用于任何"保证 XX 吗"的问题）：**先答"能"或"不能" → 再说"在什么前提下" → 举一个反例**。例如"能保证可见性，但仅限同一把锁；DCL 的锁外读就是反例，所以还要 volatile"。

### 10.3 高频翻车点（背下来避免踩）

| 说法 | 为什么错 |
|---|---|
| "每个对象都自带一个 monitor" | monitor 是**懒创建**的，竞争时才 inflate，空闲会被 deflate 回收 |
| "偏向锁就是没有锁" | 它有真实的锁语义和撤销成本，只是**加锁路径极短** |
| "锁在代码块上" | 锁在**对象**上（Mark Word），代码块只是决定锁哪个对象 |
| "轻量级锁自旋 N 次后升级重量级" | 现代 HotSpot 里自旋发生在 `ObjectMonitor::enter` 内部（已膨胀后），N 不是常数 |
| "synchronized 一定比 ReentrantLock 慢/快" | 取决于竞争程度、JDK 版本（有无偏向锁）、是否虚拟线程 |
| "锁可以降级" | 只有 monitor 的 deflate 回收，没有主动降级 |
| "JDK 8 默认有偏向锁，所以现在也有" | JDK 15 默认关闭、JDK 18 移除 |
| "synchronized 保证所有代码有序" | 只禁止**跨越锁边界**的重排，锁内仍可重排 |
| "wait 和 sleep 都是让线程等待" | `wait` 释放锁、需持锁；`sleep` 不释放、不需持锁 |

---

## 第 11 章 速记卡与自测清单

### 11.1 一页速记卡

```
【四层结构】  语法 → 字节码(monitorenter/ACC_SYNCHRONIZED) → JVM(MarkWord+Monitor) → 硬件(CAS/park)
【Mark Word】 lock 位: 01无锁/01+bias偏向/00轻量/10重量/11GC ；8字节复用
              所以 identityHashCode 与偏向锁互斥
【升级链】    无锁 → 偏向锁(写线程ID,0成本重入) → 轻量级(栈上LockRecord+CAS) → 重量级(ObjectMonitor+park)
              触发: 第二个线程来 / hashcode / wait() / CAS失败
【偏向锁】    撤销需 safepoint；类级计数 20→批量重偏向(epoch+1)，40→批量撤销(该类不可偏向)
              默认延迟 4s 启用；JDK15 默认关闭，JDK18 移除
【轻量级】    重入靠 displaced_header = NULL 标记（不做 CAS）；解锁 CAS 失败 = 已膨胀
【重量级】    _owner / _recursions / _cxq(头插LIFO) / _EntryList(FIFO) / _WaitSet(wait)
              exit: release_store(_owner,NULL) + 按 QMode 转移 + unpark
              非公平（可 barging），_succ 缓解饥饿
【内存语义】  unlock HB lock（同一把锁）；acquire 屏障在 enter 后，release 在 exit 前
              x86 acquire 免费，release 用 lock addl $0,(%rsp)；ARM 需 ldar/stlr
【wait/notify】必须持锁；wait 释放锁+进 _WaitSet；notify 不释放+移至 _EntryList；
              醒来必须重新抢锁；必须 while 防虚假唤醒
【JIT】       逃逸分析→锁消除(仅C2)；相邻合并→锁粗化；自适应自旋由 _SpinDuration 调节
【不变量】    锁不可主动降级（只有 deflate 回收）；锁在对象上不在代码上；类锁≠实例锁
【新变化】    JDK15 废弃偏向锁 / JDK21 虚拟线程 pinning / JDK24 修复 pinning
```

### 11.2 自测清单（能独立讲清 = 掌握）

**入门**
- [ ] 三种用法各自锁什么，实例锁与类锁是否互斥
- [ ] 字节码里 `monitorenter` 为什么配两条 `monitorexit`，异常表类型是什么
- [ ] 同步方法为什么没有指令，靠什么标志

**进阶**
- [ ] Mark Word 五个状态 + 64/32 位差异 + JOL 怎么验证
- [ ] 锁升级四步，每步的触发条件/Mark Word/代价
- [ ] 轻量级锁的重入怎么标记，解锁 CAS 失败代表什么
- [ ] ObjectMonitor 三个队列各自存谁、进出方向、为什么 cxq 头插
- [ ] wait/notify/notifyAll 的完整时序，为什么 notify 不释放锁

**深入**
- [ ] 偏向锁撤销为什么要 safepoint，批量重偏向/撤销的阈值与 epoch 机制
- [ ] identityHashCode 和偏向锁为什么互斥
- [ ] 为什么锁不可降级，deflate 又是什么
- [ ] 锁消除/锁粗化的条件，为什么只在 C2
- [ ] acquire/release 屏障在 x86 和 ARM 上的差别
- [ ] DCL 为什么必须 volatile（synchronized 为什么不够）
- [ ] 虚拟线程下 synchronized 的 pinning 问题与规避

**实战**
- [ ] 用 jstack/JFR 定位锁竞争与死锁
- [ ] 高并发计数的选型（synchronized/Atomic/LongAdder）
- [ ] 单机临界区 → 分段锁 → CAS → 分布式锁的演进路径

---

## 附录：参考资料与验证方式

**源码位置（HotSpot）**
- `src/share/vm/runtime/objectMonitor.hpp` / `.cpp` —— ObjectMonitor 结构与 enter/exit
- `src/share/vm/runtime/synchronizer.cpp` —— fast_enter / slow_enter / inflate / deflate
- `src/share/vm/runtime/biasedLocking.cpp` —— 偏向锁撤销、批量重偏向
- `src/share/vm/oops/markOop.hpp` —— Mark Word 位定义
- `src/share/vm/runtime/park.hpp` / `os_posix.cpp` —— park/unpark
- `src/share/vm/interpreter/interpreterRuntime.cpp` —— `monitorenter` 的入口

**规范**
- JVMS §3.11.10 / §6.5 `monitorenter` `monitorexit`
- JLS §17.4.5 happens-before（monitor lock/unlock 规则）
- JEP 374（Deprecated Biased Locking）、JEP 444（Virtual Threads）、JEP 491

**动手验证**
```bash
# 1. 看字节码
javap -v -p Target.class | grep -A30 "syncBlock"

# 2. 看对象头（引入 jol-core）
java -jar jol-cli.jar internals java.lang.Object
# 或代码：ClassLayout.parseInstance(obj).toPrintable()

# 3. 看锁相关 JVM 参数
java -XX:+PrintFlagsFinal -version | grep -iE "BiasedLocking|EliminateLocks|Spinning|MonitorDeflation"

# 4. 看 JIT 是否消除了锁
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining ...

# 5. 看锁竞争
jcmd <pid> JFR.start duration=30s filename=lock.jfr
```

**推荐延伸阅读**
- 《Java 并发编程的艺术》第 2 章（锁的实现，中文最系统）
- 《深入理解 Java 虚拟机》第 13 章（线程安全与锁优化）
- Doug Lea《The java.util.concurrent Synchronizer Framework》（AQS 论文，用于对比）
- Aleksey Shipilev 关于 biased locking / lock elision 的演讲与 JMH 实践

---

> **最后一句**：`synchronized` 的所有知识都能收敛到一句话——**"用对象头的 2 个 bit 记录锁状态，用 CAS 做无竞争的快路径，用 park 做有竞争的慢路径，用内存屏障保证语义"**。把这句话记住，再往上挂细节，就不会散。

