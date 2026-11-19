# Java ThreadLocal 深度解析 · 底层原理、延伸知识与大厂面试题全解

> 整理日期：2026-11-19　|　适用：Java 后端社招/校招面试、源码复习、线上问题排查
> 阅读建议：先看「第二章内存模型」和「第四章内存泄漏」，这两块是面试 80% 的分值所在；第五章到第七章是拉开差距的加分项；第八章按公司看。

---

## 目录

- [第〇章　先搞懂：ThreadLocal 到底解决什么问题](#第〇章先搞懂threadlocal-到底解决什么问题)
- [第一章　API 与语义边界](#第一章api-与语义边界)
- [第二章　底层内存模型（最重要的一张图）](#第二章底层内存模型最重要的一张图)
- [第三章　源码级剖析：set / get / remove / 扩容](#第三章源码级剖析set--get--remove--扩容)
- [第四章　内存泄漏：完整因果链与反证](#第四章内存泄漏完整因果链与反证)
- [第五章　InheritableThreadLocal：父子线程传递](#第五章inheritablethreadlocal父子线程传递)
- [第六章　TransmittableThreadLocal：线程池传递的正确答案](#第六章transmittablethreadlocal线程池传递的正确答案)
- [第七章　延伸知识（拉开差距的部分）](#第七章延伸知识拉开差距的部分)
- [第八章　国内大厂面试题全解（含考核程度）](#第八章国内大厂面试题全解含考核程度)
- [第九章　最佳实践 Checklist](#第九章最佳实践-checklist)
- [第十章　面试前 5 分钟速记卡](#第十章面试前-5-分钟速记卡)

---

# 第〇章　先搞懂：ThreadLocal 到底解决什么问题

## 0.1 问题的起点：共享可变状态

一个变量如果被多个线程同时读写，就要处理两件事：**原子性**和**可见性**。传统解法是「抢锁」：

```
方案 A：加锁串行化
   线程1 ──┐
   线程2 ──┼──▶ [ synchronized 临界区 ] ──▶ 共享变量
   线程3 ──┘         （同一时刻只放一个进去）
```

加锁的本质是「让多个线程轮流用同一个变量」。但很多场景下，我们根本**不需要共享**——每个线程想要的是「自己独有的一份」，比如：

- `SimpleDateFormat` 不是线程安全的，但每个线程各用一个实例就没问题；
- 数据库连接、`SqlSession` 不能跨线程混用；
- 一次 HTTP 请求的 `UserContext`、`traceId`，只属于处理这条请求的线程；
- Spring 事务管理器要把「当前线程绑定的连接」存起来。

这时加锁是**杀鸡用牛刀**：锁竞争会带来上下文切换和串行化开销，而真正的诉求是「隔离」。

## 0.2 核心思想：线程封闭（Thread Confinement）

> **不要共享，每个线程自己存一份。** 数据被封闭在单线程内部，天然没有并发问题——不需要锁、不需要 CAS、不需要 volatile。

线程封闭有三种常见实现：

| 方式 | 做法 | 特点 | 典型代表 |
|---|---|---|---|
| Ad-hoc 封闭 | 靠程序员自觉，只在单线程内传递 | 无保障，易被误用 | 方法内局部变量 |
| 栈封闭（Stack Confinement） | 变量只存在于方法栈帧里 | 编译器/内存模型天然保证 | 方法内 `int count` |
| **ThreadLocal 封闭** | 变量挂在线程对象上，按线程隔离 | **可跨方法、跨类传递**，仍保持隔离 | `ThreadLocal`、`RequestContextHolder` |

**关键洞察**：局部变量已经能做到栈封闭，但它只能在**一个方法的调用链**里传递。一旦要跨方法、跨多层调用（Controller → Service → DAO → 工具类）传上下文，就只能一路加参数（丑陋且侵入），或者塞进一个全局 Map（要加锁）。ThreadLocal 的价值就是：**既能跨方法传递，又不需要加锁**。

## 0.3 一句话定义

> ThreadLocal 提供**线程本地变量**：同一个 ThreadLocal 对象，在不同线程里 `get()` 到的值互不相同；值实际存放在**每个 Thread 对象自己的 `ThreadLocalMap` 字段**里，ThreadLocal 本身只是访问这份数据的「钥匙」（key）。

⚠️ 注意最后半句——这是本章最容易被答错的地方：

- ❌ 错误认知：「ThreadLocal 内部有一个 Map，key 是线程」
- ✅ 正确认知：「**Thread 内部有一个 Map，key 是 ThreadLocal**」

方向正好相反。这个「谁持有谁」的倒置，直接决定了内存泄漏的形态，也是面试官最爱追问的点。

---

# 第一章　API 与语义边界

## 1.1 四个核心方法

| 方法 | 语义 | 底层动作 |
|---|---|---|
| `T get()` | 取当前线程的值 | 拿当前线程的 map → 按 hash 查 Entry → 没有则调用 `setInitialValue()` |
| `void set(T value)` | 设置当前线程的值 | 拿 map，没有就 `createMap`；有就 `map.set(this, value)` |
| `void remove()` | 删除当前线程的值 | `map.remove(this)`，会把 Entry 的 key、value、数组槽全部置 null |
| `protected T initialValue()` | 初始值（延迟调用） | 首次 `get()` 时若无值才调用，默认返回 `null` |

辅助 API：

```java
// 工厂方法：等价于 new ThreadLocal<>() + 重写 initialValue()
ThreadLocal<UserInfo> tl = ThreadLocal.withInitial(UserInfo::new);

// JDK 8+ 的函数式初始化（内部类 SuppliedThreadLocal）
```

## 1.2 三个必须明确的语义边界

**① 「线程本地」不等于「线程安全」**

ThreadLocal 保证的是「每个线程有自己的副本」，但如果这个副本本身是**可变对象且被多线程共享引用**，依然不安全：

```java
// 反例：value 是一个 List，父线程把它塞进去，子线程通过 InheritableThreadLocal 拿到同一个引用
private static final ThreadLocal<List<String>> HOLDER =
        new InheritableThreadLocal<>();
HOLDER.set(new ArrayList<>());   // 主线程创建
// 子线程 get() 到的是【同一个 ArrayList 对象】，两个线程同时 add → 不安全
```

**② 不是「拷贝」，是「每线程各自 set」**

ThreadLocal 不会自动克隆值。如果默认 `initialValue()` 返回同一个静态对象，那所有线程拿到的也是同一个对象。

**③ `set(null)` 不等于 `remove()`**

```java
tl.set(null);    // Entry 还在，只是 value 变成 null，占位仍然存在，不会触发清理
tl.remove();     // 把 Entry 从 table 里摘掉，size--，是真正的清理
```

别把 `set(null)` 当 remove 用——面试里这么答会被追问「那 stale entry 怎么清」。

## 1.3 典型应用场景（面试要能张口就来 3~5 个）

| 场景 | 用的什么 | 为什么必须用 ThreadLocal |
|---|---|---|
| `SimpleDateFormat` 复用 | `ThreadLocal<SimpleDateFormat>` | 该类内部 `Calendar` 非线程安全，加锁性能差 |
| Spring `RequestContextHolder` | `NamedThreadLocal` + 可选 `InheritableThreadLocal` | 让任意层拿到当前 `HttpServletRequest` |
| Spring 事务 `TransactionSynchronizationManager` | `ThreadLocal<Map<Object,Object>>` | 把「当前线程绑定的 Connection」与事务状态绑定，保证同一事务用同一连接 |
| MyBatis `SqlSession` / 一级缓存 | `ThreadLocal` | 会话不能跨线程 |
| 日志 MDC（`traceId`、`userId`） | `ThreadLocal<Map<String,String>>` | 全链路日志打标，零参数侵入 |
| 多数据源动态切换 | `ThreadLocal<String>` 存 dsKey + `AbstractRoutingDataSource` | 一次请求内数据源不变 |
| 全链路压测「影子表」标记 | `ThreadLocal<Boolean>` | 请求级标记，随线程走 |
| 用户上下文 `UserContext` | `ThreadLocal` | 避免每个方法都传 `userId` |

---

# 第二章　底层内存模型（最重要的一张图）

## 2.1 三个关键字段

`java.lang.Thread` 里藏了两个字段（JDK 8）：

```java
public class Thread implements Runnable {
    // ...
    ThreadLocal.ThreadLocalMap threadLocals = null;            // 普通 ThreadLocal 用这个
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;  // InheritableThreadLocal 用这个
}
```

`ThreadLocal` 自己则只有一个不可变的 hash 值：

```java
public class ThreadLocal<T> {
    private final long threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode = new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
    private static int nextHashCode() { return nextHashCode.getAndAdd(HASH_INCREMENT); }
    // 注意：没有 value 字段！value 不在 ThreadLocal 里
}
```

## 2.2 整体内存模型图

```
                      JVM 堆
  ┌───────────────────────────────────────────────────────────────────────────┐
  │                                                                           │
  │   ┌────────────────────┐         ┌──────────────────────────────────┐     │
  │   │ Thread 对象 (A)     │         │  Thread 对象 (B)  ← 线程池里的   │     │
  │   │ ─────────────────  │         │  ──────────────────────────────  │     │
  │   │ threadLocals ──────┼──┐      │  threadLocals ──────┐           │     │
  │   └────────────────────┘  │      └─────────────────────┼───────────┘     │
  │                           │                            │                 │
  │                           ▼                            ▼                 │
  │            ┌──────────────────────────┐   ┌──────────────────────────┐   │
  │            │ ThreadLocalMap (A 私有)   │   │ ThreadLocalMap (B 私有)  │   │
  │            │  table : Entry[16]        │   │  table : Entry[16]       │   │
  │            │  size  : 2                │   │  size  : 2               │   │
  │            │  threshold : 10           │   │  threshold : 10          │   │
  │            └────────────┬─────────────┘   └────────────┬─────────────┘   │
  │                         │                              │                │
  │              ┌──────────┴──────────┐        ┌──────────┴──────────┐      │
  │              ▼                     ▼        ▼                     ▼      │
  │     ┌────────────────┐   ┌────────────────┐ ...                        │
  │     │ Entry[h1]      │   │ Entry[h2]      │                            │
  │     │ key  ───弱──┐   │   │ key  ───弱──┐  │                            │
  │     │ value = 张三 │   │   │ value = 李四 │  │                            │
  │     └────────────────┘   └────────────────┘                            │
  │              │                     │                                    │
  │              ▼                     ▼                                    │
  │     ┌──────────────────┐  ┌──────────────────┐                          │
  │     │ ThreadLocal 对象1 │  │ ThreadLocal 对象2 │  ← 通常 static final    │
  │     │ hash = 0x...      │  │ hash = 0x...      │     (GC Root 可达)      │
  │     └──────────────────┘  └──────────────────┘                          │
  │                                                                         │
  └───────────────────────────────────────────────────────────────────────────┘

  同一个 ThreadLocal 对象1，同时是 A 和 B 两个 Map 的 key。
  但各自 Entry 里的 value 完全独立 —— 这就是"同一个钥匙，开不同的锁"。
```

**从这张图能直接读出 4 个结论**（面试时可以主动讲，非常加分）：

1. **数据跟着线程走**：线程销毁，`Thread` 对象不可达，`ThreadLocalMap` 和它所有 Entry/value 一起被回收 → 不会泄漏。
2. **ThreadLocal 只是 key**：它不持有 value，所以「ThreadLocal 被回收」并不意味着「value 被回收」。
3. **每个 Thread 最多一个 Map**：一个 ThreadLocal 对应一个 Entry，多个 ThreadLocal 竞争同一个 table 的槽位 → **有哈希冲突**，所以 ThreadLocalMap 需要处理冲突（这就是开放地址法的由来）。
4. **线程池是前提改变者**：线程长期存活 → Thread 长期强可达 → Map 长期存活 → 里面残留的 value 就「活」着，泄漏由此产生。

## 2.3 为什么设计成「Thread 持有 Map」而不是「ThreadLocal 持有 Map」

这是必考的设计题，标准答案分三层：

| 维度 | Thread 持有 Map（JDK 实际方案） | ThreadLocal 持有 Map（错误方案） |
|---|---|---|
| 生命周期 | Map 与线程同生共死，线程死了自动回收 | Map 全局常驻，线程销毁后**没有任何机制能回收其中的 Entry**（ThreadLocal 是静态的，永远可达） |
| 数量 | 每个线程一个 Map，线程数×Entry 数 | 一个全局 Map，key 是 Thread，所有线程的 value 挤在一起 → 需要并发控制 |
| 并发 | 每个 Map 只被单线程访问，**完全无锁** | 多线程写同一个 Map，必须加锁或 CAS → 退化成 `ConcurrentHashMap`，性能骤降 |
| 定位效率 | 直接 `t.threadLocals` 字段读，O(1) | 每次都要 hash 查 Map |

一句话总结：**把 Map 挂在线程上，就同时拿到了「无锁」「自动回收」「O(1) 定位」三样好处**；反过来把 Map 挂在 ThreadLocal 上，三样全丢，还得自己处理并发。

## 2.4 一次 `get()` 的完整寻址链（先有个印象，第三章拆源码）

```
Thread.currentThread()
        │
        ├─▶ 读字段 t.threadLocals            (O(1)，无锁)
        │
        ├─▶ i = threadLocalHashCode & (len-1)  (len 是 2 的幂 → 等价于取模)
        │
        ├─▶ 看 table[i]
        │     ├─ 命中（e.get() == this）──▶ 返回 e.value
        │     ├─ 槽为 null ───────────────▶ 没找到，返回 null
        │     └─ 槽非 null 但 key 不同 ───▶ 线性探测 i+1, i+2 ...（同时顺手清 stale）
        │
        └─▶ 若 map 为 null 或没找到 ─────▶ setInitialValue() → 调 initialValue() → 建 Entry
```

---

# 第三章　源码级剖析：set / get / remove / 扩容

## 3.1 ThreadLocalMap 的类结构

`ThreadLocalMap` 是 `ThreadLocal` 的**静态内部类**（注意：不是 `Map` 接口的实现类，也不是 `HashMap` 的子类）。

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** Entry 自己强引用 value */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);          // key 交给 WeakReference 持有 → 弱引用
            value = v;         // value 是普通字段 → 强引用
        }
    }

    private static final int INITIAL_CAPACITY = 16;   // 必须是 2 的幂

    private Entry[] table;          // 数组，不是链表 + 数组
    private int size = 0;
    private int threshold;          // 扩容阈值

    private void setThreshold(int len) {
        threshold = len * 2 / 3;    // 负载因子 2/3，比 HashMap 的 0.75 低
    }
}
```

**与 `HashMap` 的结构差异一览：**

| 维度 | `HashMap` | `ThreadLocalMap` |
|---|---|---|
| 冲突解决 | 链地址法（数组 + 链表 / 红黑树） | **开放地址法（线性探测）** |
| Entry 结构 | `Node(hash, key, value, next)` | `Entry extends WeakReference<ThreadLocal>`，无 next |
| key 引用 | 强引用 | **弱引用** |
| 初始容量 | 16 | 16 |
| 负载因子 | 0.75 | **2/3 ≈ 0.667** |
| 扩容 | 2 倍，需要 rehash 所有节点 | 2 倍，`expungeStaleEntries` + 重新插入 |
| 线程安全 | 否 | 是（靠「每线程私有」实现的线程封闭，不是靠锁） |

## 3.2 为什么 Entry 要继承 `WeakReference`（先说结论，第四章展开）

- key 用弱引用：让 `ThreadLocal` 对象在**外部没有强引用时**能被 GC 回收，回收后 `Entry.get()` 返回 `null` → 这个槽位变成一个可以被识别的「墓碑（stale entry）」，后续 `set/get/remove` 就能**顺带**把它清掉。
- value 用强引用：因为业务正在用它，不能莫名其妙消失（否则每次 GC 都可能让业务数据凭空消失）。

这是「**牺牲 key 保 value**」的设计：key 可以被回收（因为它是我们自己创建的工具对象），value 不能（因为它是业务数据）。

## 3.3 哈希值：`0x61c88647` 的秘密

```java
private static final int HASH_INCREMENT = 0x61c88647;      // = 1640531527
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);          // 每个 ThreadLocal 递增这个值
}
```

**这个魔数的来历**：`0x61c88647 = 2^32 × (√5 − 1) / 2`，即**黄金分割数**在 32 位无符号整数下的补码表示（`2^32 * 0.6180339887 ≈ 2654435769`，其有符号表示为 `-1640531527`，无符号即 `0x9E3779B9`；ThreadLocal 用的是它的等价形式 `0x61c88647`）。

它对应一种叫 **Fibonacci Hashing** 的技巧：用黄金分割数做步长去「散列」一个连续递增的序列，得到的分布**极其均匀**，相邻值落在数组上的位置会均匀散开，而不是扎堆。

验证一下（用一个 16 长度的数组模拟）：

```java
public static void main(String[] args) {
    int len = 16, inc = 0x61c88647, h = 0;
    for (int i = 0; i < 16; i++) {
        h += inc;
        System.out.print((h & (len - 1)) + " ");
    }
}
// 输出：3 6 9 12 15 2 5 8 11 14 1 4 7 10 13 0
// 16 个 ThreadLocal 恰好铺满 16 个槽位，且间隔均匀 —— 完美的散列
```

**为什么能做到「完美铺满」**：因为 `0x61c88647` 与 2 的幂互素，累加取模等价于在环上以固定步长跳跃，必然遍历所有槽位（类似全排列）。这样在同一个线程里创建 N 个 ThreadLocal（N < 容量），**几乎不会冲突**。

**面试表述**：
> ThreadLocal 的 hash 不是按对象地址算的，而是一个 `AtomicInteger` 每次自增 `0x61c88647`。这个数是黄金分割数在 32 位下的表示，用它做步长能让 hash 在 2 的幂长度的数组上分布均匀，大幅降低线性探测的冲突概率。

⚠️ 注意：`threadLocalHashCode` 是 `final` 字段，**在 ThreadLocal 对象创建时就固定**，之后不会再变。所以它不会像 `HashMap` 那样因为 key 的 hash 变化而找不到数据。同时也意味着——**不要重写 `ThreadLocal` 的 `equals/hashCode`**，因为定位用的是引用相等 `k == key`，不是 `equals`。

## 3.4 `set()` 源码逐行

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);        // return t.threadLocals;
    if (map != null)
        map.set(this, value);              // 已有 map，走 ThreadLocalMap.set
    else
        createMap(t, value);               // 首次 set：new ThreadLocalMap(this, value)
}
```

`createMap` 会以「懒加载」的方式建一个容量 16 的 Map：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);        // threshold = 16 * 2 / 3 = 10
}
```

核心逻辑在 `ThreadLocalMap.set`：

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);        // ① 定位初始槽

    // ② 线性探测
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();                     // 取弱引用指向的 key（可能为 null）

        if (k == key) {                                 // ③ 命中同一个 key → 覆盖 value
            e.value = value;
            return;
        }
        if (k == null) {                                // ④ 碰到 stale entry（key 已被回收）
            replaceStaleEntry(key, value, i);           //    用新值占据这个位置，并顺手清理
            return;
        }
    }

    // ⑤ 一路探测到空槽 → 直接放进去
    tab[i] = new Entry(key, value);
    int sz = ++size;

    // ⑥ 没有清理出空间 且 达到阈值 → rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);                  // 环形前进，走到末尾回到 0
}
```

**三个要点**（这是面试官爱挖的细节）：

1. `set` **不是**先算位置再插入，而是「**边探测边判断**」：探测过程中一旦发现 key 已被回收的槽，就调用 `replaceStaleEntry` 复用这个槽——这是**写路径上的主要清理入口**。
2. 只有探测到 `null` 槽才是「真的没有」，才会新建 Entry。注意：**这里能用 null 判断「没有」，正是因为线性探测法保证了「从 hash 位置到第一个 null 之间」的所有 key 都探测过**（这也是开放地址法删除时不能直接置 null 的根本原因，见 3.7）。
3. 触发 `rehash` 的条件是**双重的**：`cleanSomeSlots` 没清理出空间 **且** size 到阈值。意思是「如果已经在探测过程中清理掉了垃圾，就先不扩容」，能省则省。

## 3.5 `get()` 与 `getEntryAfterMiss`

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue();        // map 为 null 或没找到 → 走初始化
}

private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)     // 一次命中（绝大多数情况）
        return e;
    else
        return getEntryAfterMiss(key, i, e);   // 冲突，线性探测
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);        // ★ 读路径上的清理入口：碰到墓碑就清
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;                          // 探测到 null 槽 → 确实不存在
}
```

`setInitialValue()`：

```java
private T setInitialValue() {
    T value = initialValue();             // 默认返回 null，子类可重写
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

**注意 `initialValue()` 是「懒」的**：只有真正 `get()` 且没有值时才会执行。这是 `ThreadLocal.withInitial(...)` 能省内存的原因——线程没用过就不会创建对象。

## 3.6 `remove()`

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();                 // Reference.clear()：把弱引用指向的对象置 null
            expungeStaleEntry(i);      // 清理槽位，并修复后面被"断链"的元素
            return;
        }
    }
}
```

`remove` 也走线性探测（因为 key 可能被挤到了别的槽）。找到后：
- `e.clear()` 把 key 的弱引用清空；
- `expungeStaleEntry(i)` 把 value、Entry 置 null，size--，并**重排后续元素**。

## 3.7 关键函数 `expungeStaleEntry`：不只是清理，还要「修补断链」

这是整个 ThreadLocalMap 最精妙的一段代码，面试说到它能直接体现源码功底。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // ① 清理当前墓碑
    tab[staleSlot].value = null;      // 断开 value 的强引用（解决内存泄漏的关键一行）
    tab[staleSlot] = null;
    size--;

    // ② 继续向后扫描，直到遇到 null 槽
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {

        ThreadLocal<?> k = e.get();
        if (k == null) {              // 又碰到一个墓碑 → 一起清
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // ③ 关键：重算这个元素的理想位置 h
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {             // 它不在理想位置（当初是被冲突挤过来的）
                tab[i] = null;
                // 从理想位置开始找第一个空槽，把它挪回去（也叫"增量式 rehash"）
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;                         // 返回遇到的 null 槽位置
}
```

**为什么必须「挪元素」（这段的核心原因）**：

线性探测法依赖一条不变量——**从 key 的理想位置到它实际所在位置，中间不能出现 null 槽**，否则查找会在 null 处提前停止，导致「明明存在却查不到」。

```
删除前：
  index:  0     1     2     3     4     5
          [A]   [B]   [C]   [ ]   [D]   [ ]
  假设 B、C、D 的理想位置都是 0（冲突后依次后移）

直接删 B（置 null）：
  index:  0     1     2     3     4     5
          [A]   [ ]   [C]   [ ]   [D]   [ ]
  → 查 C 时，从 0 开始，到 1 遇到 null 就停 → 判定"不存在"，但 C 明明在！

正确做法（expungeStaleEntry 的挪动）：
  index:  0     1     2     3     4     5
          [A]   [C]   [D]   [ ]   [ ]   [ ]
  → 链条重新连续，查找正确
```

**这就是开放地址法与链地址法的本质区别**：链表删除只要改指针，O(1)；开放地址法删除要修复「探测链」，代价是 O(len) 的扫描。ThreadLocalMap 用「一次扫描顺带清掉后续所有墓碑 + 顺便重排」的方式摊薄了这个成本。

**另一个细节：`expungeStaleEntry` 的返回值是「遇到的第一个 null 槽位置」**，上层拿它继续做 `cleanSomeSlots` 的起点——一次遍历，多处复用。

## 3.8 `cleanSomeSlots`：启发式清理

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {     // 遇到墓碑
            n = len;                            // 重置扫描次数：发现垃圾说明"值得再扫一轮"
            removed = true;
            i = expungeStaleEntry(i);           // 清理，并拿到新的起点
        }
    } while ((n >>>= 1) != 0);                  // 每次右移一位，最多扫描 log2(n) 次
    return removed;
}
```

**逻辑解读**：
- 默认只扫描 `log₂(n)` 个槽位（n=16 时扫 4 个），是**采样式**的「启发式清理」，不是全量扫描；
- 一旦扫到墓碑，就把 `n` 重置为 `len`，相当于「发现异常就加大扫描力度」，再扫 log₂(len) 个；
- 返回值表示「这一轮有没有清出东西」。`set` 里用它来决定要不要真的扩容：**清出了空间就不用扩容**。

**这是一种「均摊成本」的设计哲学**：每次 set/get/remove 顺手做一点点清理，让内存泄漏在多数场景下被自动消化，而不是每次都做 O(n) 全量扫描。

## 3.9 扩容：`rehash` → `resize`

```java
private void rehash() {
    expungeStaleEntries();                       // ① 先全量清理一遍所有墓碑
    // ② 清理后如果 size 仍然接近阈值（>= 3/4 threshold），才真正扩容
    if (size >= threshold - threshold / 4)
        resize();
}

private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;                     // ③ 容量翻倍
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null;                  // 顺手再清一次（双保险）
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);   // ④ 按新长度重算位置
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);                   // 线性探测找空位
                newTab[h] = e;
                count++;
            }
        }
    }
    setThreshold(newLen);                        // 新 threshold = newLen * 2/3
    size = count;
    table = newTab;
}
```

**扩容路径完整版（以 16 → 32 为例）**：

```
初始：len=16, threshold=10
   ↓ size 增长
size == 10 且 cleanSomeSlots 没清出东西
   ↓ rehash()
expungeStaleEntries() 全量清理 → 假设 size 降到 8
   ↓ 判断 8 >= 10 - 2 = 8 ?  是 → resize
len=32, threshold = 32*2/3 = 21

注意：如果全量清理后 size 降到了 8 以下，就【不扩容】，继续用 16 的数组 —— 
这就是"能复用就不扩"的策略，避免 ThreadLocalMap 无脑膨胀。
```

**为什么负载因子是 2/3 而不是 0.75？**
开放地址法对「装填因子」非常敏感：装得越满，线性探测的平均探测长度增长越快（接近 0.7 以后性能急剧恶化）。2/3 是这个 Java 实现留给探测链的余量。

**为什么容量必须是 2 的幂？**
`hash & (len - 1)` 等价于 `hash % len`，但位运算比取模快一个量级；同时保证 `resize` 时每个元素的新位置只有两种可能（原位置 或 原位置+oldLen），分布均匀。

## 3.10 为什么不用 `HashMap` 而要自己写 `ThreadLocalMap`

面试标准答法（四条，缺一条扣分）：

| # | 原因 |
|---|---|
| 1 | **无需并发控制**：Map 是线程私有的，彻底单线程访问，`HashMap` 的并发设计（Node 的锁、CAS、扩容协作）全是浪费。 |
| 2 | **需要弱引用 key**：`HashMap` 的 key 是强引用，做不到「key 被回收后留下可识别的墓碑」。 |
| 3 | **生命周期管理**：需要在 set/get/remove 时「顺带」清理 stale entry，`HashMap` 没有这种语义，也没有 `expungeStaleEntry` 这种修补探测链的能力。 |
| 4 | **内存与性能**：ThreadLocal 数量通常很少（个位数），`HashMap` 的 Node 对象头 + next 指针开销更大；开放地址法数组紧凑、cache 友好、无额外对象分配，且省掉了每个 Node 的对象头开销。 |

补充一句加分的话：**ThreadLocalMap 是一个「特化的、以删除/清理为核心诉求的」哈希表**，它的设计目标不是通用性，而是「无锁 + 可清理 + 省内存」。

---

# 第四章　内存泄漏：完整因果链与反证

> 这一章是 ThreadLocal 面试的**绝对核心**，能完整讲清楚「为什么弱引用还会泄漏」的人不超过三成。

## 4.1 先把引用链拆成 4 段

```
  [GC Root]
      │  ① 强引用
      ▼
   Thread 对象
      │  ② 强引用（字段 threadLocals）
      ▼
  ThreadLocalMap
      │  ③ 强引用（字段 table 数组）
      ▼
   Entry[] table
      │  ④ 强引用（数组元素）
      ▼
   Entry
    ├── ⑤ 弱引用 ──▶ ThreadLocal 对象（key）
    └── ⑥ 强引用 ──▶ value（业务数据）

  另外：ThreadLocal 对象通常还有一条来自 static 字段的强引用（⑦）。
```

- ① ~ ④ 全是强引用链，**只要 Thread 活着，整条链就都活着**；
- ⑤ 是弱引用，**ThreadLocal 只有在没有 ⑦ 这类强引用时才会被回收**；
- ⑥ 是强引用，**只要 Entry 活着，value 就活着**。

## 4.2 四个阶段的生命周期图

```
【阶段 1】正常使用

  Thread ──▶ Map ──▶ Entry ──弱──▶ ThreadLocal(静态变量持有)
                          └──强──▶ value ✅ 都在


【阶段 2】ThreadLocal 的强引用被断开
（典型场景：非 static 的 ThreadLocal 定义在方法内，方法返回后没人引用它）

  Thread ──▶ Map ──▶ Entry ──弱──▶ (已被 GC 回收, key = null)
                          └──强──▶ value ❌ 还活着！没人用了，但引用链没断

  这就是「stale entry / 墓碑」：key 为 null，但 value 仍被 Entry 强引用，
  而 Entry 仍被 table 强引用，table 仍被 Map 强引用，Map 仍被 Thread 强引用。
  → GC 无法回收 value。这就是内存泄漏。


【阶段 3】后续操作触发清理（运气好的情况）

  下一次对同一个 Map 执行 set / get / remove 时，
  只要探测路径经过这个墓碑，expungeStaleEntry 就会把 value 置 null → 泄漏被消除。

  但这是【机会性】的：
   - 如果之后这个线程再也不用 ThreadLocal，永远不会有探测经过那里；
   - 如果线程池里的线程一直存活但不碰这个 Map，value 就永久滞留。


【阶段 4】线程池放大（最危险）

  线程池 corePoolSize=200 的线程长期存活（甚至与 JVM 同寿命）
  → 每个线程的 Map 都在
  → 每个 Map 里残留的 value 都在
  → 如果 value 是一个 5MB 的 byte[]/查询结果集/大缓存对象
  → 200 × N × 5MB，直接 OOM，而且 heap dump 里看不出"谁在引用"，只能看到一堆
     ThreadLocalMap$Entry 的 value 巨大 —— 排查成本极高
```

## 4.3 核心问题一：为什么 key 要用弱引用？

先明确一个前提：**「弱引用」不是为了「防止 value 泄漏」**，它解决的是另一个问题——**让 Map 有能力识别出「这个 key 已经没用了」**。

**反证法：假设 key 用强引用会怎样？**

```
如果 Entry.key 是强引用：

  Thread ──▶ Map ──▶ Entry ──强──▶ ThreadLocal

  即使业务代码早已不再使用这个 ThreadLocal（方法结束、字段置 null），
  因为 Entry 强引用了它，ThreadLocal 对象【永远无法被回收】。
  → 这个 Entry 永远无法被判定为"过期"，永远清不掉。
  → ThreadLocal 对象 + value 双重泄漏，而且【连清理的机会都没有】。
```

而用弱引用之后：key 一旦被回收，`e.get() == null` 就成了一个**明确的、可判定的"过期信号"**，程序据此可以主动清理。所以：

> **弱引用 = 给 ThreadLocalMap 一个「自我诊断」的能力**，让它能发现「这个槽已经废了」。真正解决泄漏的是后面的 `expungeStaleEntry`，而不是弱引用本身。

面试时如果只答「用弱引用是为了防止内存泄漏」，会被追问「那为什么还会泄漏」，答不上来就崩了。正确表述是：
> 弱引用让 key 可被回收，从而产生「key == null」这一**可判定的过期标记**，为后续的探测式清理提供了依据；但它只解决了 key 一侧，value 一侧仍是强引用，所以仍然需要 `expungeStaleEntry` 主动清理，以及业务侧 `remove()` 兜底。

## 4.4 核心问题二：为什么 value 不能用弱引用？

反过来想：如果 value 也是弱引用：

- **只要发生一次 GC（尤其是 Young GC），value 就可能在业务仍在使用的过程中被回收**；
- 你 `set` 进去一个对象，下一毫秒 Full GC，`get()` 就返回 `null` —— 这在语义上完全不可接受；
- ThreadLocal 的定位是「线程级别的变量容器」，其语义等价于「线程的一个字段」，字段值当然不能随便被 GC 掉。

所以设计上的取舍是明确的：**key 是「工具」可以丢，value 是「数据」不能丢**。

## 4.5 到底什么情况下会真的泄漏？一张判定表

| 场景 | ThreadLocal 是否 static | 是否 remove | 是否泄漏 | 说明 |
|---|---|---|---|---|
| A. 普通线程，用完线程销毁 | 任意 | 否 | ❌ 不泄漏 | 线程死亡 → Thread 对象不可达 → 整条链被回收 |
| B. 线程池 + static ThreadLocal + 不 remove | static | 否 | ⚠️ **key 不回收，value 滞留** | key 有静态强引用永不为 null，槽位不会被判为 stale；value 一直占着，**必须 remove** |
| C. 线程池 + 非 static ThreadLocal + 不 remove | 非 static | 否 | ⚠️ **典型泄漏** | key 被回收成 null，value 滞留；只能靠后续探测「碰巧」清掉 |
| D. 任意场景 + 正确 remove | 任意 | 是 | ✅ 不泄漏 | 标准做法 |
| E. 单例/长生命周期的 ThreadLocal + 大对象 value | static | 否 | ⚠️ 泄漏 | 最隐蔽，因为「看着像正常使用」 |

**注意 B 行这个反直觉的结论**：很多人以为「static 的 ThreadLocal 因为 key 永远可达，所以反而安全」—— 恰恰相反。**static 的 ThreadLocal 的 key 永远不为 null，所以永远不会被判定为 stale，永远不会被自动清理**。它的 value 只能靠 `remove()` 手动清。这是面试里的高级追问点。

## 4.6 结论：必须 remove，而且必须用 `try-finally`

```java
// ✅ 正确姿势：finally 保证异常路径也能清理
private static final ThreadLocal<UserContext> CTX = new ThreadLocal<>();

public void handle() {
    CTX.set(new UserContext(...));
    try {
        doBusiness();
    } finally {
        CTX.remove();          // 不要写 CTX.set(null)
    }
}

// ❌ 错误姿势 1：没有 finally，抛异常就泄漏
CTX.set(ctx);
doBusiness();                  // 抛异常 → 后面永远不执行
CTX.remove();

// ❌ 错误姿势 2：用 set(null) 假装清理
CTX.set(null);                 // Entry 还在，槽位仍被占用，不会触发 expunge

// ❌ 错误姿势 3：在拦截器里 set，却在另一个拦截器里忘了 remove
//    （典型：preHandle set，afterCompletion 忘写，或异步请求走了另一条链路）
```

**在线程池场景下，「用完 remove」不是可选项，是硬性要求**。原因：线程不会被销毁，value 会一直挂在这个工作线程上，直到下一次有人 `set` 覆盖它，或者恰好有探测扫到它。

## 4.7 一个容易被忽略的「伪泄漏」：`ThreadLocal` 数量爆炸

如果代码里动态 `new ThreadLocal<>()`（比如每次请求 new 一个），会产生大量 ThreadLocal 对象，每个都会在 Map 里占一个槽：

- 槽位很快被填满 → 频繁 rehash / resize → 性能下降；
- 大量墓碑 → 探测链变长 → `get` 从 O(1) 退化到 O(n)。

**最佳实践：ThreadLocal 一律 `private static final`**。这样：
- 全局只有一个实例，不会疯狂占槽；
- 生命周期与类一致（通常与 JVM 一致），语义清晰；
- 但注意 4.5 的 B 行——它**不能**免除 remove 的义务。

## 4.8 泄漏的量化感受（面试可以举的例子）

```
假设线程池 200 个线程，每个线程 Map 里残留 3 个 Entry，
每个 value 是一个 2MB 的查询结果 List：

  200 × 3 × 2MB = 1.2GB  ← 全部是"看不到归谁"的滞留内存

heap dump 里的表征：
  java.lang.ThreadLocal$ThreadLocalMap$Entry   数量 600
  每个 Entry 的 value 指向 ArrayList → Object[] → 大量业务对象
  且这些 Entry 的 referent（key）为 null
```

排查手段（详见第八章 Q25）：`jmap -histo` 看 `ThreadLocalMap$Entry` 数量异常 → `jmap -dump` 用 MAT 看 dominator tree → 找到 `java.lang.Thread` 的 `threadLocals` 字段 → 顺着 table 找非 null 的 value。

---

# 第五章　InheritableThreadLocal：父子线程传递

## 5.1 问题的提出

普通 `ThreadLocal` **完全没有跨线程能力**：

```java
private static final ThreadLocal<String> TL = new ThreadLocal<>();

public static void main(String[] args) throws Exception {
    TL.set("parent-value");
    new Thread(() -> System.out.println(TL.get())).start();
    // 输出：null —— 子线程有自己的 ThreadLocalMap，什么都没有
}
```

`InheritableThreadLocal` 的目标：**父线程创建子线程时，把可继承的 ThreadLocal 复制一份给子线程。**

## 5.2 原理：发生在 `Thread` 的构造过程里

关键在 `Thread` 的构造器（`Thread.init`）：

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    // ...
    Thread parent = currentThread();          // ★ 注意：是"创建者"线程，不是"父类"
    // ...
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
                ThreadLocalMap.createInheritedMap(parent.inheritableThreadLocals);
}
```

也就是说：**继承发生在 `new Thread()` 的那一刻，由「创建该线程的线程」拷贝给「新线程」**。

`InheritableThreadLocal` 只是重写了三个方法，把「用哪个字段」换成 `inheritableThreadLocals`：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    // ① 子线程复制时如何生成自己的值（默认直接返回父值）
    protected T childValue(T parentValue) {
        return parentValue;
    }

    // ② 用哪个字段当 map
    ThreadLocalMap getMap(Thread t) {
        return t.inheritableThreadLocals;
    }

    // ③ 首次 set 时初始化哪个字段
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

`createInheritedMap` 的复制逻辑：

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];
    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);   // ★ 可重写 childValue 实现深拷贝
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

**复制过程的图解**：

```
   父线程 (main)
   ┌───────────────────────────────┐
   │ inheritableThreadLocals       │
   │   Entry[3] key=TL1 value=A ───┼──┐  ① childValue(A) 默认返回 A（浅拷贝！）
   └───────────────────────────────┘  │
                                      │  ② new Thread() 时复制
                                      ▼
   子线程 (thread-1)                   复制出的新 Entry，但 value 指向【同一个对象】
   ┌───────────────────────────────┐
   │ inheritableThreadLocals       │
   │   Entry[h] key=TL1 value=A ───┼──▶ 同一个 A 对象
   └───────────────────────────────┘
```

## 5.3 三个致命缺陷

### 缺陷 1：线程池里彻底失效（最致命）

```java
ExecutorService pool = Executors.newFixedThreadPool(2);

InheritableThreadLocal<String> ITL = new InheritableThreadLocal<>();
ITL.set("task-1-context");
pool.submit(() -> System.out.println(ITL.get()));  // 第一次：可能拿到 task-1-context
                                                    // （因为线程是此刻 new 的，复制了当时的上下文）
ITL.set("task-2-context");
pool.submit(() -> System.out.println(ITL.get()));  // 拿到的还是 task-1-context！
```

**原因**：线程池的 `corePoolSize` 个线程是**在第一次提交任务时才创建**，之后**永远复用**。继承只发生在创建那一刻：

```
   pool.submit(task1) ──▶ 创建 thread-0，复制 main 的 ITL（此时值 = "task-1-context"）
   pool.submit(task2) ──▶ 复用 thread-0，【不会再复制】，值还是 "task-1-context"
   ...
   pool.submit(taskN) ──▶ 复用 thread-0，值永远是创建时的那个快照

   → 上下文"卡死"在第一次创建时的状态，这就是线上经典的「数据串号」问题
```

### 缺陷 2：浅拷贝导致共享可变对象

`childValue` 默认直接 `return parentValue`，两个线程拿到**同一个对象引用**：

```java
InheritableThreadLocal<List<String>> ITL = new InheritableThreadLocal<>();
ITL.set(new ArrayList<>());
new Thread(() -> ITL.get().add("child")).start();   // 子线程往里加
ITL.get().add("parent");                            // 父线程也往里加
// → 两个线程操作同一个 ArrayList，非线程安全 → ConcurrentModificationException / 数据错乱
```

**如果确实要隔离，必须重写 `childValue` 做深拷贝**：

```java
new InheritableThreadLocal<Map<String, String>>() {
    @Override
    protected Map<String, String> childValue(Map<String, String> parent) {
        return parent == null ? null : new HashMap<>(parent);   // 深拷贝
    }
};
```

### 缺陷 3：单向、一次性的「继承」

- 只有「创建时」复制，之后父线程修改，子线程**看不到**（不是共享，只是快照）；
- 反向不成立：子线程改了，父线程看不到；
- 孙线程会继续继承子线程的快照（逐代复制，逐代偏离）。

## 5.4 适用场景

只适合**「一次性创建、生命周期明确的子线程」**：

- 主线程启动一个后台常驻线程，需要带上启动时的配置；
- 批处理任务里 fork 少量工作线程，父线程上下文在 fork 后不再变化。

**不适合**：线程池、异步任务（`@Async`）、定时任务、任何复用线程的场景。

→ 这就是为什么必须有 TTL。

---

# 第六章　TransmittableThreadLocal：线程池传递的正确答案

> 出自阿里巴巴开源的 `transmittable-thread-local`（TTL），是「线程池 + 上下文传递」的标准方案，也是阿里系面试的高频题。

## 6.1 核心思想：从「继承」改为「传递」

`InheritableThreadLocal` 的思路是「**创建时复制**」（被动、一次性）；
TTL 的思路是「**执行前注入、执行后还原**」（主动、每次任务都做）：

```
  提交任务时            任务执行前              任务执行后
  ─────────            ──────────              ──────────
  捕获快照    ──▶      把快照"回放"到     ──▶  恢复成原来的值
  (capture)            执行线程身上             (restore)
                        (replay)
```

对应三个方法：`TransmittableThreadLocal.Transmitter.capture()` / `replay(snapshot)` / `restore(snapshot)`。

## 6.2 源码级原理：装饰器模式

TTL 不改变线程池本身，而是**装饰 `Runnable` / `Callable`**：

```java
// 使用方式
Runnable task = TtlRunnable.get(originalRunnable);   // 包装一次
executor.execute(task);
```

`TtlRunnable` 的核心逻辑（简化版）：

```java
public final class TtlRunnable implements Runnable {
    private final AtomicReference<Captured> capturedRef;   // 提交线程的上下文快照
    private final Runnable runnable;
    private final boolean releaseTtlValueReferenceAfterRun;

    private TtlRunnable(Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
        // ★ 在【提交线程】里执行 capture，拿到提交时刻的上下文快照
        this.capturedRef = new AtomicReference<>(capture());
        this.runnable = runnable;
        this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
    }

    @Override
    public void run() {
        Object captured = capturedRef.get();
        if (captured == null || releaseTtlValueReferenceAfterRun && !capturedRef.compareAndSet(captured, null)) {
            throw new IllegalStateException("TTL value reference is released after run!");
        }

        Object backup = replay(captured);      // ① 把提交者的快照"回放"到当前执行线程，并备份原值
        try {
            runnable.run();                    // ② 执行业务
        } finally {
            restore(backup);                   // ③ 无论成败，恢复执行线程原来的值
        }
    }
}
```

**流程图**：

```
  提交线程 (main)                        线程池工作线程 (pool-1-thread-0)
  ─────────────                          ─────────────────────────────
  TTL.set("ctx-A")
       │
       ├─▶ TtlRunnable.get(task)
       │      └─ capture()  ←── 快照: {TTL1: "ctx-A"}
       │
       └─▶ executor.execute(wrapped)
                     │
                     ▼
              ┌──────────────────────────────────────────────┐
              │ run():                                       │
              │   backup = replay(captured)                  │
              │      ├─ 保存工作线程线程原有 TTL 值 → backup  │
              │      └─ 把 "ctx-A" 写入工作线程的 TTL         │
              │                                              │
              │   try { task.run() }   ← 业务里 TTL.get() 拿到 "ctx-A" ✅
              │                                              │
              │   finally { restore(backup) }                │
              │      └─ 把工作线程还原成执行前的值（通常是空）  │
              └──────────────────────────────────────────────┘
                     │
                     ▼
              线程回到池中，上下文"干净" → 下一个任务不会串号 ✅
```

## 6.3 为什么必须 `restore`？

这是 TTL 设计里最关键的一步，也是面试追问点：

- 工作线程是**复用的**。如果不还原，任务 A 的上下文会残留在工作线程上，任务 B 执行时就会读到 A 的数据 → **数据串号**；
- `restore` 本质是「给线程池的线程做一次现场保护与恢复」，等价于把 `ThreadLocal` 当成**可嵌套的栈**来用；
- 即使任务抛异常，`finally` 也保证还原。

**这三点合起来构成了「隔离性」**：capture 保证「取的是提交者的」，replay 保证「执行者能看到」，restore 保证「执行完不污染」。

## 6.4 完整方案：还要包装线程池

只包装 `Runnable` 有个漏洞：**任务内部再往线程池提交子任务时，子任务没有包装**。TTL 提供了线程池装饰器来递归解决：

```java
// ① 装饰线程池（提交时自动包装所有任务）
ExecutorService executor = TtlExecutors.getTtlExecutorService(originalExecutor);

// ② 或用 TTL agent 无侵入增强（javaagent 方式，推荐生产使用）
//    -javaagent:path/to/transmittable-thread-local-2.x.x.jar

// ③ 或手动包装单个任务
executor.execute(TtlRunnable.get(() -> {
    System.out.println("ctx = " + CTX.get());
}));
```

**线程池装饰器的作用**：`getTtlExecutorService` 会把 `execute/submit/invokeAll` 等所有入口的任务都自动 `TtlRunnable.get()` 包一层，并且是**幂等包装**（重复 wrap 不会套娃）。

## 6.5 TTL 的增强点（比 ITL 多做了什么）

| 能力 | `InheritableThreadLocal` | `TransmittableThreadLocal` |
|---|---|---|
| 传递时机 | 仅 `new Thread()` 时，一次性 | **每次任务提交时**，动态 |
| 线程池 | ❌ 完全失效 | ✅ 核心场景 |
| 快照/还原 | 无 | ✅ capture / replay / restore |
| 组件扩展 | 只能继承 `ThreadLocal` | ✅ 提供 `TtlCopier`、`Transmittee` 接口，可注册自定义捕获/还原逻辑 |
| 嵌套任务 | 不支持 | ✅ 支持（backup 机制可嵌套） |
| 与 `remove` 的关系 | 需手动 | `restore` 自动清理 |
| 性能开销 | 仅创建线程时一次 | 每次任务提交 + 执行各一次快照复制（可接受，但要注意 value 别太大） |

**加分点**：TTL 还提供了 `Transmitter.clear()` 用于清理，以及 `TtlCallable`、`TtlRunnable`、`TtlExecutors`、`TtlAgent` 等完整生态；`TransmittableThreadLocal` 继承了 `InheritableThreadLocal`，所以**它同时具备「new Thread 继承」和「线程池传递」两种能力**。

## 6.6 TTL 的坑（面试可以主动说，显得有实战）

1. **快照是浅拷贝**：value 若是可变对象，多线程共享同一引用，仍需注意并发；
2. **value 太大会拖慢提交**：每次 `capture` 都要复制整条链，大对象会放大开销；
3. **必须包装到「所有」提交入口**：`new Thread`、`CompletableFuture`（默认 ForkJoinPool）、`@Async`、中间件自带线程池都可能漏；
4. **`CompletableFuture` 默认用 `ForkJoinPool.commonPool()`**，需要显式传 `TtlExecutors` 装饰过的 executor；
5. **agent 方式有启动参数要求**，容器化部署时注意 `JAVA_TOOL_OPTIONS` 注入。

---

# 第七章　延伸知识（拉开差距的部分）

## 7.1 Java 四种引用与 `ReferenceQueue`

ThreadLocalMap 的 Entry 用到弱引用，所以要能讲清四种引用：

| 类型 | 类 | GC 行为 | 典型用途 |
|---|---|---|---|
| 强引用 | 普通赋值 `Object o = new Object()` | **永不回收**（除非不可达） | 常规对象 |
| 软引用 | `SoftReference` | **内存不足时才回收** | 缓存（如图片缓存） |
| 弱引用 | `WeakReference` | **下一次 GC 一定回收**（只要没有强引用） | `ThreadLocalMap` 的 key、`WeakHashMap` |
| 虚引用 | `PhantomReference` | 随时可能被回收，**必须配合 ReferenceQueue**，仅用于「回收前收到通知」 | 直接内存（`DirectByteBuffer`）释放、资源清理 |

**`ReferenceQueue` 的作用**：引用对象被回收后，JVM 会把这个 `Reference` 对象本身**入队**，程序可以 `queue.poll()` 拿到它，做后续清理（如 `WeakHashMap` 用它来删除对应的 Entry）。

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
WeakReference<Object> ref = new WeakReference<>(new Object(), queue);
System.gc();
Reference<?> cleared = queue.remove(1000);   // 拿到已被回收的引用，做后续清理
```

**ThreadLocalMap 为什么不用 ReferenceQueue？**
它选择了「**轮询式**」而非「通知式」：在 `set/get/remove` 时自己扫数组找 `e.get() == null` 的槽。原因：
- 数组访问本身就是 O(1) 定位，顺手扫一下成本低；
- 如果引入队列，就需要额外的线程或时机去 `poll`，还得处理并发；
- ThreadLocalMap 是线程私有的，**没有别的线程能来"通知"它**——队列的消费者只能是它自己，那就等于白绕一圈。

这是一个很好的「设计权衡」回答。

## 7.2 开放地址法 vs 链地址法

| 维度 | 链地址法（HashMap） | 开放地址法（ThreadLocalMap） |
|---|---|---|
| 结构 | 数组 + 链表/树 | 纯数组 |
| 冲突 | 挂到同一个桶的链表上 | 往后找下一个空槽（线性探测） |
| 删除 | 改指针，O(1) | **必须修补探测链**，O(n) |
| 内存 | 每个节点有对象头 + next 指针 | 只有数组，紧凑、cache 友好 |
| 装填因子 | 可以到 0.75 甚至更高 | 必须低（≤ 2/3），否则探测链变长 |
| 适用 | 数据量大、频繁增删 | 数据量小、读多、要省内存 |

**线性探测的性能曲线**（为什么 2/3 是红线）：

```
平均探测次数（成功查找）≈ 0.5 * (1 + 1/(1-α))
   α = 0.5  → 约 1.5 次
   α = 0.667 → 约 2.0 次
   α = 0.8  → 约 3.0 次
   α = 0.9  → 约 5.5 次   ← 急剧恶化
```

ThreadLocal 数量一般很少（个位数到几十），装填因子极低，所以线性探测在这里是**性价比最高的选择**。

还有另一个方案：**二次探测 / 双重散列**，能缓解「一次聚集」，实现更复杂。ThreadLocalMap 用最简单的线性探测 + 优秀 hash 函数（黄金分割数）规避了聚集问题。

## 7.3 为什么 `ThreadLocalMap` 的字段不需要 `volatile`？

这是一个很好的深入题。答案是：**因为单线程访问，可见性问题根本不存在。**

- 可见性问题只在**一个线程写、另一个线程读**时才出现（需要 happens-before 保证）；
- `ThreadLocalMap` 只被它所属的那个 Thread 访问，读写都在同一个线程内，天然满足 as-if-serial 语义，JIT 也不会做「跨线程」的错误优化；
- 所以 `table`、`size`、`threshold` 全是普通字段，`ThreadLocal.threadLocalHashCode` 也是普通 `final` 字段。

**但要注意边界**：一旦跨线程传递（TTL / ITL），可见性保证来自**别的地方**：

- `new Thread().start()`：JVM 规范规定 `start()` 之前的写对子线程可见（happens-before）；
- 线程池提交任务：`BlockingQueue.put/take` 建立了 happens-before，所以 `capture` 的快照对工作线程可见；
- `CompletableFuture` 的完成回调、`ExecutorService` 的 `Future.get()` 也都有 HB 保证。

**加分表述**：
> ThreadLocal 本身不依赖任何同步原语，是因为它把并发问题**转化成"不共享"**，而不是"正确地共享"。一旦重新引入共享（跨线程传递），就必须靠 JMM 提供的 happens-before 边（start、队列、锁、volatile）来保证可见性。

## 7.4 `ThreadLocal` vs `synchronized` vs `volatile`：三种哲学

| 方案 | 解决的问题 | 代价 | 适用 |
|---|---|---|---|
| `synchronized` / `Lock` | 多个线程**共享同一份**可变状态时的**原子性 + 可见性** | 阻塞、上下文切换、可能死锁 | 必须共享且写多（计数器、库存） |
| `volatile` | **可见性 + 有序性**，不保证原子性 | 禁止重排序、每次读写主存 | 状态标志位、单写多读 |
| `ThreadLocal` | **根本不给别人看**（隔离） | 每线程一份内存、泄漏风险 | 上下文传递、非线程安全工具的复用 |

**一句话**：`synchronized` 是「让大家排队用同一个」，`ThreadLocal` 是「每人发一个」。能用后者解决的问题，就不该用前者。

## 7.5 Spring 中的 ThreadLocal 应用（面试常问「举几个例子」）

**① `RequestContextHolder`**

```java
public abstract class RequestContextHolder {
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
            new NamedThreadLocal<>("Request attributes");
    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =
            new NamedThreadLocal<>("Inheritable Request attributes");

    public static void setRequestAttributes(RequestAttributes attributes, boolean inheritable) { ... }
}
```

> 注意：Spring 用了 `NamedThreadLocal`（继承 `ThreadLocal` 并重写 `toString`），只是为了**调试时能看出是谁**。`inheritable` 为 true 时用了另一套可继承的实现（`NamedInheritableThreadLocal`）。

在 `DispatcherServlet` 的 `processRequest` 里 `set`，在 `finally` 里 `resetRequestAttributes()` —— **标准的 try-finally 兜底**，可以直接当范例讲。

**② `TransactionSynchronizationManager`**

```java
private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");
```

存的是「当前线程绑定的事务资源」（如 `DataSource` → `ConnectionHolder`），保证同一事务内多次 DAO 调用拿到**同一个 Connection**，也保证事务的传播（`PROPAGATION_REQUIRED` 时复用）。

**③ `LocaleContextHolder`、`SimpleAsyncTaskExecutor` 的 `TaskDecorator`**

**④ 面试延伸题：`@Async` 为什么会丢上下文？**

`@Async` 把方法扔到 `ThreadPoolTaskExecutor` 执行，工作线程是另一个线程，`requestAttributesHolder` 是普通 `ThreadLocal` → 拿不到。解决方案：

```java
@Bean
public TaskDecorator contextCopyingDecorator() {
    return runnable -> {
        // 捕获当前线程的上下文
        RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
        Map<String, String> mdc = MDC.getCopyOfContextMap();
        String traceId = TraceContext.get();
        return () -> {
            try {
                RequestContextHolder.setRequestAttributes(attrs);
                if (mdc != null) MDC.setContextMap(mdc);
                TraceContext.set(traceId);
                runnable.run();
            } finally {
                RequestContextHolder.resetRequestAttributes();
                MDC.clear();
                TraceContext.remove();          // ★ 必须清理
            }
        };
    };
}

@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
    e.setTaskDecorator(contextCopyingDecorator());
    return e;
}
```

> `TaskDecorator` 是 Spring 提供的官方钩子，本质就是「手写版的 TTL」：capture → replay → restore。生产里也可以直接接 TTL。

## 7.6 日志 MDC 与全链路 traceId

SLF4J 的 `MDC` 是一个门面，真实实现在 logback 的 `LogbackMDCAdapter`：

```java
// logback 实现（简化）
private final ThreadLocal<Map<String, String>> copyOnThreadLocal = new ThreadLocal<>();
```

所以：

- `MDC.put("traceId", id)` 是写当前线程的 `ThreadLocal`；
- 日志 pattern `%X{traceId}` 是从当前线程读；
- **异步日志（AsyncAppender）是另一个线程**，`%X{traceId}` 会变成空 —— 需要 `MDC.getCopyOfContextMap()` 传递；
- **线程池里必须清理**，否则下一个请求会打出上一个请求的 traceId（这是线上最常见的「traceId 串号」事故）。

```java
// 拦截器里的标准写法
String traceId = UUID.randomUUID().toString().replace("-", "");
MDC.put("traceId", traceId);
try {
    chain.doFilter(req, resp);
} finally {
    MDC.clear();      // 必须 clear
}
```

## 7.7 Netty 的 `FastThreadLocal`：为什么能快 3 倍

Netty 认为 `ThreadLocalMap` 有两个缺陷：**哈希冲突**（线性探测 O(n)）和**墓碑清理**（`expungeStaleEntry` 要扫描、要修链）。于是它自己造了一个：

```
FastThreadLocal 方案：

  每个 FastThreadLocal 在创建时，从全局 AtomicInteger 申请一个【自增 index】
  （0, 1, 2, 3 ...），这个 index 就是它在数组里的【下标】。

  InternalThreadLocalMap（挂在 FastThreadLocalThread 上）
  ┌─────┬─────┬─────┬─────┬─────┬─────┐
  │ idx0│ idx1│ idx2│ idx3│ idx4│ ... │   Object[] indexedVariables
  │  A  │  B  │  C  │  D  │  E  │     │
  └─────┴─────┴─────┴─────┴─────┴─────┘
        ↑
        └── FTL_B 的 index = 1 → 直接 array[1]，O(1)，【零哈希、零冲突、零探测】

  删除时：置为 UNSET 哨兵值，并回调 onRemoval()，不需要修链
  清理时：FastThreadLocalRunnable 在 finally 里 removeAll()，整表清理
```

| 对比项 | `ThreadLocal` | `FastThreadLocal` |
|---|---|---|
| 定位 | hash + 线性探测，冲突时 O(n) | 固定 index 直接数组访问，恒 O(1) |
| 冲突 | 有（依赖黄金分割数缓解） | **完全没有** |
| 删除成本 | 要修探测链 | 置哨兵值 |
| 内存 | Entry 对象 + WeakReference | 纯 `Object[]`，无 Entry 对象 |
| 泄漏风险 | 有 | 无（`removeAll` 兜底） |
| 使用前提 | 无 | **必须运行在 `FastThreadLocalThread` 上**，普通 Thread 会退化成慢速实现 |

**面试答法**：
> Netty 的优化本质是「**把哈希寻址换成索引寻址**」——用空间换时间，用「每个 ThreadLocal 占一个固定下标」消灭哈希冲突和探测，同时用「整表清理」消灭墓碑。

**延伸思考**：为什么 JDK 不这么做？因为 JDK 的 `ThreadLocal` 数量不可控（可能成千上万、动态创建销毁），固定下标会导致数组无限膨胀；Netty 的 `FastThreadLocal` 数量是**编译期/启动期就确定的少数几个**，所以 index 方案成立。**这是一个「约束换性能」的经典案例**。

## 7.8 虚拟线程（JDK 21+）与 ThreadLocal 的冲击

**背景**：JDK 21 正式发布虚拟线程（JEP 444）。一个 JVM 里可以轻松跑几十万到上百万虚拟线程。

**问题**：

- 每个虚拟线程仍然是一个 `Thread` 对象，仍然有 `threadLocals` 字段；
- 一万个平台线程的 ThreadLocalMap 是 1 万份，**一百万个虚拟线程就是 100 万份**；
- 如果每份里放一个 1KB 的上下文 → 1GB 内存，直接压垮堆；
- 而且虚拟线程的 ThreadLocal **无法像平台线程那样"复用得久"**，频繁创建/销毁会带来更多分配压力。

**官方的应对：`ScopedValue`（作用域值）**

```java
// JDK 21 起以预览 API 引入（JEP 446，之后多轮预览演进）
static final ScopedValue<UserInfo> USER = ScopedValue.newInstance();

ScopedValue.where(USER, userInfo).run(() -> {
    // 在这个作用域内（含所有被调用的方法）可以读
    System.out.println(USER.get());
});
// 出了 run()，绑定自动失效 —— 不可变、自动释放、结构化并发友好
```

| 维度 | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| 可变性 | 可变（set/get） | **不可变**（只能读，绑定即定值） |
| 生命周期 | 手动 `remove`，易泄漏 | **作用域结束自动失效**，无泄漏 |
| 传递性 | 需 ITL / TTL | **结构化并发天然继承**（fork 的子任务自动可见） |
| 内存 | 每线程一份 | 共享不可变绑定，**几乎零额外内存** |
| 状态 | 正式 | 预览（尚未转正） |

**面试表述**：
> 虚拟线程放大了 ThreadLocal 的两个老问题：内存放大和清理遗漏。JDK 给出的方向是 `ScopedValue`——用「不可变 + 作用域」替代「可变 + 手动清理」，从设计上消灭泄漏。但目前 `ScopedValue` 仍是预览特性，主流框架（Spring、TTL）还依赖 ThreadLocal，短期内不会替换。

## 7.9 响应式 / 协程场景的坑（Reactor、Kotlin 协程）

- **Reactor**：一个 `Mono/Flux` 链路会在不同线程上执行（`publishOn`/`subscribeOn` 切换），`ThreadLocal` 完全不适用。Reactor 用 `Context`（`contextWrite`/`deferContextual`）传递。强行用 ThreadLocal 会得到 `null` 或错乱。
- **Kotlin 协程**：协程可以在不同线程上恢复执行，`ThreadLocal` 会丢。解决用 `ThreadContextElement`（如 `MDCContext`），它会在协程恢复时把上下文重新放回当前线程。
- **共同本质**：**ThreadLocal 的假设是「一个逻辑任务绑定一个线程」**。一旦「逻辑任务可以在多线程间迁移」，这个假设就崩了。

## 7.10 `ThreadLocalRandom`：一个「神似而形不似」的延伸

```java
ThreadLocalRandom.current().nextInt(100);
```

它**不是** `ThreadLocal` 的子类，但用的是同一套思想：把状态放在 `Thread` 对象自己的字段里。

```java
// java.lang.Thread 里的字段
long threadLocalRandomSeed;      // 每个线程自己的种子
int  threadLocalRandomProbe;     // 探测值，用于避免伪共享
int  threadLocalSecondarySeed;
```

**为什么不用 `ThreadLocal<Random>`？** 因为：
1. `ThreadLocal<Random>` 需要维护一个 Map 和 Entry 对象，开销更大；
2. `ThreadLocalRandom` 通过 `Unsafe` 直接读线程字段，**零 Map 开销**；
3. 还处理了**伪共享**（`@Contended` / probe 填充），性能远超 `Random` 的 CAS 竞争。

**这是一个绝佳的收尾论点**：ThreadLocal 的价值不在「这个类」，而在「**状态跟着线程走**」这套思路。凡是能这么干的，都能获得无锁的性能。

---

# 第八章　国内大厂面试题全解（含考核程度）

## 8.0 先看评分规则

**考核程度（星级）说明**

| 星级 | 含义 | 面试中的定位 |
|---|---|---|
| ★ | 送分题，答不出直接挂 | 开场暖场、确认基础 |
| ★★ | 常规题，校招必答 | 确认「你知道这个东西」 |
| ★★★ | 有区分度，能刷掉 50% | 确认「你用过、想过」 |
| ★★★★ | 高分题，社招分水岭 | 确认「你读过源码/踩过坑」 |
| ★★★★★ | 定级题，决定 P6/P7 | 确认「你能设计方案、能救火」 |
| ★★★★★+ | 压力测试，答得好可直接加分 | 面试官故意挖坑，看边界意识 |

**各厂风格速查（帮你调整回答重心）**

| 公司 | 风格特点 | 应对策略 |
|---|---|---|
| 阿里 / 蚂蚁 | 原理 + 场景 + 线上问题三连；爱追问「为什么这样设计」 | 讲设计权衡，别只背结论；准备一个真实踩坑故事 |
| 美团 | 源码细节最深：hash、扩容、探测链、阈值；爱问数字 | 记住 16 / 2-3 / 0x61c88647 / log₂n 这些具体值 |
| 字节 | 场景驱动 + 边界条件；爱问「线程池里怎么办」 | 一定要能讲 ITL 缺陷 + TTL 三步 |
| 腾讯 | 概念辨析 + 工程规范；爱问和锁/局部变量的取舍 | 讲清「隔离 vs 共享」的哲学区别 |
| 京东 / 拼多多 | 实战导向；爱问工具类写法、数据源切换 | 能写出可运行的代码示例 |
| 百度 / 快手 / 网易 | 综合，有时挖到 JMM、引用类型、Netty | 延伸知识要能接得上 |

> ⚠️ 说明：以下题目是**按高频题型归纳整理**，不同年份、不同部门的表述会有差异（有的问「讲一下原理」，有的问「为什么这么设计」），但考察的内核一致。面试时请以面试官实际表述为准，不要硬套。

---

## 8.1 阿里系（阿里 / 蚂蚁 / 淘天）

### Q1　【阿里·一面】讲一下 ThreadLocal 的实现原理，变量到底存在哪里？

**考核程度：★★★★☆**　**考点：内存模型是否真的搞懂**

**答题框架**：先纠正方向 → 再讲结构 → 最后讲一次 get 的寻址

**详解**

> 先说最容易答错的地方：**不是 ThreadLocal 里有个 Map 存各线程的值，而是每个 Thread 对象里有个 `ThreadLocalMap` 字段存各 ThreadLocal 的值。方向是反的。**

具体结构三层：

1. `Thread` 类里有两个字段：`threadLocals`（普通）和 `inheritableThreadLocals`（可继承），类型都是 `ThreadLocal.ThreadLocalMap`，**默认 null，懒加载**；
2. `ThreadLocalMap` 内部是一个 `Entry[] table`，`Entry` 的 **key 是 ThreadLocal 对象的弱引用**，**value 是业务值的强引用**；
3. `ThreadLocal` 自己**不存 value**，只有一个 `final long threadLocalHashCode` 作为定位依据。

一次 `get()` 的流程：

```
t = Thread.currentThread()        → t.threadLocals（O(1) 读字段）
i = hash & (len - 1)               → 定位槽
table[i].get() == this ?           → 命中返回 value
否则线性探测 i+1, i+2 ...          → 碰到 key==null 顺手 expunge
map 为 null 或全没找到              → setInitialValue() → initialValue() → 建 Entry
```

**为什么这么设计（主动补一句，加分）**：Map 挂在线程上 → 数据随线程生命周期自动回收，且**每个 Map 只被单线程访问，完全无锁**。如果反过来挂在 ThreadLocal 上，就变成全局 Map，既要加锁又无法随线程回收。

**延伸**：`ThreadLocal.withInitial()` 内部是 `SuppliedThreadLocal`，重写了 `initialValue()` 调用你传的 `Supplier`；`initialValue()` 是**懒执行**的，只有第一次 `get()` 且没值时才调。

---

### Q2　【阿里·二面】为什么 ThreadLocalMap 要定义在 Thread 类里，而不是 ThreadLocal 里？如果让你设计，你会怎么选？

**考核程度：★★★★★**　**考点：设计权衡能力，P6/P7 分水岭**

**详解**

这是「设计题」，必须从三个维度对比（**答两点及格，答三点优秀**）：

**① 生命周期（决定会不会泄漏）**

- 挂在 `Thread` 上：Map 是线程的「附属物」，线程销毁 → Thread 不可达 → Map、Entry、value 全部一起回收，**天然自净**；
- 挂在 `ThreadLocal` 上：ThreadLocal 通常是 `static final`，与 JVM 同寿命 → 这个全局 Map 永远活着 → 线程死了以后，**Map 里以 Thread 为 key 的条目没有任何机制能回收**（因为没人知道那个线程已经死了），泄漏比现在严重得多。

**② 并发模型**

- 挂在 `Thread` 上：每个线程访问自己的 Map，**单线程访问，零同步**，可以用最朴素的数组 + 线性探测；
- 挂在 `ThreadLocal` 上：所有线程往同一个 Map 写，必须 `ConcurrentHashMap` 或加锁，**每次 get/set 都要 CAS 或锁**，性能差一个数量级。

**③ 定位效率**

- 挂在 `Thread` 上：`t.threadLocals` 是一次**字段读取**，O(1)，不需要哈希；
- 挂在 `ThreadLocal` 上：每次都要 `map.get(currentThread())`，多一次哈希 + 可能的冲突探测。

**一句话收尾**：
> ThreadLocalMap 挂在 Thread 上，同时拿到了「无锁」「自动回收」「O(1) 定位」三个好处；挂在 ThreadLocal 上三个全丢。这是一个典型的「**数据结构跟着生命周期走**」的设计。

**可能的追问：那 ThreadLocal 会不会有很多个 Map（每个线程一个），内存不是更浪费？**
> 不会。Map 是懒加载的，线程不用 ThreadLocal 就永远是 null，一个对象都不多。而如果真的挂在 ThreadLocal 上，全局 Map 要为每个用过的线程保留一个条目，同样占内存，而且无法回收。

---

### Q3　【阿里·二面·高频】ThreadLocal 会内存泄漏吗？Entry 为什么用弱引用？既然用了弱引用，为什么还会泄漏？

**考核程度：★★★★★**　**考点：本题是全章最高频，答不完整基本就止步二面**

**答题框架**：会 → 讲引用链 → 讲弱引用解决什么 → 讲弱引用没解决什么 → 讲清理机制 → 讲业务侧兜底

**详解（分五步，一步不能少）**

**第一步：结论**
> 会。准确说不是「永久泄漏」，而是「**value 在长生命周期线程上滞留**」——只要线程不死、又没人 `remove`，value 就一直被强引用着，GC 回收不掉。

**第二步：引用链**
```
Thread(强) → ThreadLocalMap(强) → Entry[](强) → Entry
                                              ├─ 弱 → ThreadLocal(key)
                                              └─ 强 → value
```
整条链除了 key 是弱引用，其余全是强引用。**只要 Thread 活着，value 就活着。**

**第三步：弱引用解决了什么**
> 弱引用解决的是「**让 Map 有能力识别过期槽位**」。当 ThreadLocal 对象在外部没有强引用时，GC 会回收它，`Entry.get()` 变成 null。这个 null 就是一个明确、可判定的「墓碑」标记，后续 `set/get/remove` 探测到这个槽时就能把它清掉（`expungeStaleEntry`）。
>
> 如果 key 是强引用：Entry → 强 → ThreadLocal，即使业务早就不用了，ThreadLocal 也永远回收不掉，**连"这个槽已经废了"都无法判定，一次清理的机会都没有**。

**第四步：弱引用没解决什么（关键！）**
> 弱引用只管 key。value 仍然是 Entry 的强引用字段，Entry 又被 table 强引用，table 被 Map 强引用，Map 被 Thread 强引用。所以「key 死了，value 还活着」——这就是墓碑。
>
> value 不能也做成弱引用，因为那会导致**业务正在用的数据被下一次 GC 直接抹掉**，语义上不可接受。所以只能「牺牲 key，保住 value，然后靠主动清理补漏洞」。

**第五步：清理机制 + 业务兜底**
> 清理是**机会性**的，只在三个时机发生：
> - `set`：探测遇到墓碑 → `replaceStaleEntry`
> - `get`：探测遇到墓碑 → `expungeStaleEntry`
> - `remove` → `expungeStaleEntry`
>
> 这些都要求「**有操作经过那个槽**」。如果一个线程池线程之后再也不碰 ThreadLocal，就永远清不掉。所以**业务侧必须在 `finally` 里 `remove()`**，这不是优化，是硬性要求。

**一句话总结（可以直接背）**：
> 弱引用给了 ThreadLocalMap「自我诊断」的能力，让它能把「key 已死、value 还活」的槽标记出来并清理；但诊断不等于治疗——清理只在有操作经过时发生，所以线程池场景必须靠 `remove()` 兜底。

---

### Q4　【阿里·三面·挖坑】有人说「用 static 修饰 ThreadLocal 就不会内存泄漏了」，对吗？

**考核程度：★★★★★+**　**考点：边界意识，这是故意挖的坑**

**详解**

**答：错，而且恰恰相反——`static` 反而让自动清理彻底失效。**

推导：

- `static final ThreadLocal` 被 GC Root 强引用，**永远不为 null**；
- 因此 `Entry.get()` **永远不会返回 null**，这个槽**永远不会被判定为 stale**；
- `expungeStaleEntry` / `cleanSomeSlots` 只清理「key == null」的槽 → **永远不会清理它**；
- 结果：value 只能靠 `set` 覆盖或 `remove()` 显式清掉，否则在线程池线程上**永久滞留**。

**对比一下两种情况：**

| 场景 | key 会被回收吗 | 会自动清理吗 | 泄漏风险 |
|---|---|---|---|
| 非 static，方法内 new，方法结束无引用 | ✅ 会（成为墓碑） | ✅ 有机会被探测清掉 | 中（靠运气） |
| `static final` | ❌ 不会 | ❌ 完全不会 | **高（必须 remove）** |

**正确的表述**：
> `static` 的价值在于「保证 ThreadLocal 只有一个实例、不疯狂占槽、语义清晰」，它**不能**免除 `remove` 的义务。反而是「非 static + 线程池」的组合更危险，因为 key 被回收后 value 滞留，还可能因为反复 new ThreadLocal 把 table 撑爆。

**加分回答**：
> 严格说，ThreadLocal 引起的通常不是「不可达内存泄漏」，而是「**可达但无用**的内存滞留」（memory retention）。从 heap dump 看，这些对象**是有 GC Root 的**，所以 MAT 的 Leak Suspects 报告不一定能命中，需要人肉顺引用链排查——这也是它排查成本高的原因。

---

### Q5　【阿里·二面】ThreadLocalMap 的清理机制具体是怎么实现的？`expungeStaleEntry` 为什么还要"挪动"后面的元素？

**考核程度：★★★★★**　**考点：真正读过源码**

**详解**

见 [3.7 节](#37-关键函数-expungestaleentry不只是清理还要修补断链)，这里给面试口径：

**清理的三个入口**

| 入口 | 触发条件 | 调用链 |
|---|---|---|
| 写路径 | `set` 探测到 `key==null` | `set` → `replaceStaleEntry` → `expungeStaleEntry` + `cleanSomeSlots` |
| 读路径 | `get` 探测到 `key==null` | `get` → `getEntryAfterMiss` → `expungeStaleEntry` |
| 显式 | 业务调 `remove()` | `remove` → `clear()` + `expungeStaleEntry` |

**"挪动元素"的原因（核心）**

线性探测法依赖一个不变量：**从 key 的理想位置到它的实际位置之间，不能出现 null 槽**。删除时如果只把槽置 null，就会出现"断链"，导致本来存在的元素查不到：

```
  [A][B][C][ ][D][ ]   ← B、C、D 都冲突后移了
  删 B → [A][ ][C][ ][D][ ]
  查 C：从 0 开始，i=1 遇 null 就停 → 报"不存在"，但 C 明明在！
```

所以 `expungeStaleEntry` 在清理完墓碑后，会**继续向后扫描到第一个 null 槽**，把途中所有元素重新计算理想位置 `h = hash & (len-1)`，`h != i` 的就**搬回离理想位置最近的空槽**（又叫"增量式 rehash"）。这样探测链重新连续，查找恢复正确。

**代价**：一次清理是 O(连续探测段长度)，不是 O(1)。这就是开放地址法「删除贵」的本质，也是 ThreadLocalMap 只在必要时才清理的原因。

---

### Q6　【阿里·三面】线上一个服务 Full GC 频繁但老年代回收不掉，怀疑和 ThreadLocal 有关，你怎么排查？

**考核程度：★★★★★**　**考点：实战能力，区分"会背"和"会救火"**

**详解（按顺序讲，体现排查方法论）**

**① 先确认现象，别一上来就看 ThreadLocal**

```bash
jstat -gcutil <pid> 1000 20        # 看 FGC 次数、老年代占用是否只涨不跌
jmap -histo:live <pid> | head -30  # 看对象数量 TOP
```

重点看：`java.lang.ThreadLocal$ThreadLocalMap$Entry` 数量是否异常（正常是个位数到几十/线程）。

**② dump 堆，用 MAT 找引用链**

```bash
jmap -dump:format=b,file=heap.hprof <pid>
# 或启动时加 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump
```

在 MAT 里：
- 打开 **Dominator Tree**，看哪个对象 Retained Size 最大；
- 找 `java.lang.Thread` → 展开 `threadLocals` → 展开 `table` → 看哪些 `Entry` 的 `value` 特别大、且 `referent`（key）为 null 或指向某个类；
- 用 **Path to GC Roots（exclude weak references）** 确认引用链是 `Thread → ThreadLocalMap → table → Entry → value`。

**③ 定位是哪个业务对象**

拿到 value 的类名 → 全局搜索哪里 `set` 了这个对象 → 检查是否 `remove`。

**④ 检查线程池配置**

```bash
jstack <pid> | grep "pool-" | wc -l     # 看池里线程数量
```

线程数 × 每线程残留 value 大小 = 滞留总量，这个账要能算出来给面试官。

**⑤ 临时止血 + 根治**

- 止血：滚动重启（注意先摘流量）；或临时调小线程池、加 `-XX:MaxRAMPercentage` 限制；
- 根治：所有 `set` 的地方补 `try-finally remove`；拦截器/过滤器里统一兜底；异步链路用 `TaskDecorator` 或 TTL 传递 + 清理。

**⑥ 加防护（体现工程思维）**

- 自定义 `ThreadPoolExecutor` 重写 `afterExecute`，统一清理本线程 ThreadLocal；
- 或统一封装 `ThreadLocalUtil`，强制 `execute(Runnable)` 必须带清理；
- 加监控：定期采样 ThreadLocalMap 的 size，超阈值告警。

**加分收尾**：
> ThreadLocal 泄漏的特点是「**内存增长平缓、Full GC 后不降、heap dump 里对象都有 GC Root**」，所以容易被误判成"业务缓存没清"。排查时必须先看 `ThreadLocalMap$Entry` 的数量和 value 大小，这个特征比看 GC 日志更快。

---

## 8.2 美团

> 美团风格：**极爱源码细节和具体数字**。「threshold 是多少」「初始容量」「hash 怎么算」「扩容几倍」这类问题一定会问。

### Q7　【美团·一面】ThreadLocal 的 hash 值是怎么算的？`0x61c88647` 这个数有什么讲究？

**考核程度：★★★★**　**考点：是否真的看过源码，而不是背博客**

**详解**

```java
private static final int HASH_INCREMENT = 0x61c88647;
private static AtomicInteger nextHashCode = new AtomicInteger();
private final long threadLocalHashCode = nextHashCode();   // 创建时固定
private static int nextHashCode() { return nextHashCode.getAndAdd(HASH_INCREMENT); }
```

**三个要点**：

1. **不是按对象地址算的**（没有用 `System.identityHashCode`），而是全局 `AtomicInteger` **每次自增一个固定步长**。所以第 1 个 ThreadLocal 的 hash 是 0，第 2 个是 `0x61c88647`，第 3 个是 `2×0x61c88647`……
2. **`0x61c88647` 是黄金分割数**：`2^32 × (√5−1)/2 ≈ 2654435769`，其 32 位有符号表示为 `-1640531527`（无符号 `0x9E3779B9`），JDK 取的是它的等价补码形式 `0x61c88647`（十进制 1640531527）。
3. **为什么要用它**：这叫 **Fibonacci Hashing**。用黄金分割数做步长去散列一个递增序列，结果在 2 的幂长度的数组上**分布极其均匀**，且能"铺满"所有槽位。

**当场验证（面试可以直接口述这个实验）**：

```java
int len = 16, inc = 0x61c88647, h = 0;
for (int i = 0; i < 16; i++) { h += inc; System.out.print((h & 15) + " "); }
// 3 6 9 12 15 2 5 8 11 14 1 4 7 10 13 0  ← 16 个值恰好占满 16 个槽，无冲突
```

**延伸**：
- `threadLocalHashCode` 是 `final` 字段，**创建后不可变** → 不会出现 HashMap 那种「key 的 hash 变了导致找不到」的问题；
- 定位用的是**引用相等** `e.get() == this`，不是 `equals`；所以**重写 ThreadLocal 的 `equals/hashCode` 没有任何意义**（`threadLocalHashCode` 是 final 字段，且定位不走 `equals`）。

---

### Q8　【美团·二面】ThreadLocalMap 的哈希冲突怎么解决？为什么不用链表？

**考核程度：★★★★**　**考点：数据结构选型理解**

**详解**

**冲突解决：开放地址法中的「线性探测」（Linear Probing）**

```java
int i = hash & (len - 1);                        // 理想位置
for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) { ... }
private static int nextIndex(int i, int len) { return (i + 1 < len) ? i + 1 : 0; }  // 环形
```

探测到 `null` 就停（说明真的没有），探测到 `key == null` 就清理并复用槽位，探测到相同 key 就覆盖。

**为什么不用链表（链地址法）？四个理由**：

| # | 理由 |
|---|---|
| 1 | **没有 `next` 指针的对象开销**。`Entry` 直接继承 `WeakReference`，如果再加 `next` 字段，每个 Entry 更大；ThreadLocal 数量少但线程多，乘以线程数后开销可观。 |
| 2 | **天然支持"墓碑"语义**。开放地址法里"空槽"和"墓碑"是两种不同状态，能表达"这里曾经有东西、现在废了"；链表法里删除就是摘链，无法保留"曾经存在"的信息，也就没法做 `cleanSomeSlots` 这种启发式清理。 |
| 3 | **Cache 友好**。连续数组访问，探测几个槽通常在同一 cache line 内；链表要跳指针，cache miss 更多。 |
| 4 | **实现简单、无额外对象分配**。链表法要为每个冲突节点分配 Node 对象（`HashMap` 就是这样），开放地址法零分配。 |

**代价（要主动说，体现全面）**：删除贵（要修探测链）、装填因子必须低（≤2/3）、极端冲突时探测长度退化。但 ThreadLocal 数量少（几个到几十个），这些代价在实际场景中几乎不出现。

**追问：那如果线程里有 1000 个 ThreadLocal 呢？**
> 会出现性能问题：装填因子升高 → 探测链变长 → get 从 O(1) 退化；同时频繁 rehash。这也是为什么最佳实践要求 ThreadLocal 必须 `static final`、数量可控。真需要成百上千个线程本地变量，应该考虑 Netty 的 `FastThreadLocal`（索引寻址）或自己维护一个 Map。

---

### Q9　【美团·二面·爱问数字】ThreadLocalMap 的容量、阈值、扩容规则具体是多少？

**考核程度：★★★★☆**　**考点：数字型细节，答错显得没读过源码**

**详解（把这些数字背下来）**

| 项目 | 值 | 说明 |
|---|---|---|
| 初始容量 `INITIAL_CAPACITY` | **16** | 必须是 2 的幂，才能用 `hash & (len-1)` 取模 |
| 负载因子 | **2/3** | `threshold = len * 2 / 3`，比 HashMap 的 0.75 低 |
| 初始 threshold | **10** | 16 × 2 / 3 |
| 扩容倍数 | **2 倍** | `newLen = oldLen * 2` |
| 扩容后 threshold | **newLen × 2/3** | 32 → 21 |

**扩容的完整判定链（这是重点）**：

```java
// set 末尾
if (!cleanSomeSlots(i, sz) && sz >= threshold)   // ① 没清出空间 且 ② 达到阈值
    rehash();

private void rehash() {
    expungeStaleEntries();                        // ③ 全量清理一遍
    if (size >= threshold - threshold / 4)        // ④ 清理后仍 >= 3/4 阈值（即 >= 7.5，取 7）
        resize();                                 // ⑤ 真正扩容
}
```

**以 16 为例走一遍**：
```
size 到 10，且 cleanSomeSlots 返回 false（没清出东西）
  → rehash()
  → expungeStaleEntries() 全量清，假设 size 从 10 降到 6
  → 6 >= 10 - 2(=8)?  否 → 【不扩容】，继续用 16 的数组
  → 如果清理后 size 还有 8 → 扩容到 32，threshold = 21
```

**这个设计的意图**：**先清理，清理不够再扩容**。很多 ThreadLocal 泄漏其实能通过清理解决，没必要浪费内存扩容。这是「**延后扩容**」策略，和 `HashMap` 的「到 0.75 就扩」不同。

**resize 时的 rehash 方式**：对每个非 null 元素重算 `hash & (newLen-1)`，线性探测找空位插入；key 为 null 的直接丢弃（顺手再清一次）。

**追问：为什么负载因子是 2/3 而不是 0.75？**
> 开放地址法对装填因子极其敏感。线性探测的成功查找平均次数约 `0.5(1 + 1/(1-α))`：α=0.667 时约 2 次，α=0.75 时约 2.5 次，α=0.8 时约 3 次，α=0.9 时约 5.5 次。2/3 是留给探测链的安全余量，代价是更早扩容。

---

### Q10　【美团·三面】`remove` 的时候为什么不能简单把 `table[i] = null`？

**考核程度：★★★★★**　**考点：开放地址法的核心不变量，很多人答不出来**

**详解**

因为会**破坏线性探测的"连续链"不变量**，导致原本存在的数据"消失"。

```
不变量：从 key 的理想位置 h 到它的实际位置 i 之间，所有槽都非 null。
       查找时探测到 null 就停止，认为"不存在"。
```

**图示**：

```
  数组：  [0]A   [1]B   [2]C   [3]   空   [4]D   [5]空
  A 的 hash→0，B、C、D 都 hash→0（冲突后依次后移）

  ❌ 直接置空 B：
        [0]A   [1]空   [2]C   [3]空   [4]D
        查 C：从 0 开始 → 1 是空 → 停止 → 返回"不存在"。但 C 明明在 [2]！
        （这种 bug 只在特定 hash 组合下出现，极难复现，是最恶心的一类问题）

  ✅ 正确做法（expungeStaleEntry）：
        清 [1]，然后从 [2] 开始向后扫描到第一个空槽，
        对每个元素重算理想位置 h：
          C 的 h=0，当前在 2 → 搬到离 0 最近的空槽 [1]
          D 的 h=0，当前在 4 → 搬到 [2]
        结果：[0]A  [1]C  [2]D  [3]空  [4]空
        链条连续，查找正确。
```

**顺带一个细节**：`remove` 里先 `e.clear()`（清掉弱引用的 referent）再 `expungeStaleEntry(i)`，这样 `expungeStaleEntry` 内部的循环就能把当前槽当普通墓碑一起处理，代码路径统一。

**加分表述**：
> 这就是「链地址法删除 O(1)、开放地址法删除 O(n)」的根源。ThreadLocalMap 用「一次扫描顺带清掉后续所有墓碑 + 顺便搬移元素」把这次 O(n) 摊薄成了「一次遍历做两件事」，这是它在删除上做的工程优化。

---

## 8.3 字节跳动

> 字节风格：**场景驱动，爱问边界和"如果……怎么办"**。TTL 几乎必问。

### Q11　【字节·一面】ThreadLocal 和 synchronized 有什么区别？什么时候用哪个？

**考核程度：★★★**　**考点：基础辨析，但答得浅会被追问**

**详解**

**核心区别一句话**：
> `synchronized` 解决「**多个线程共享同一份数据**」时的并发安全问题，靠**串行化**；`ThreadLocal` 解决「**每个线程需要独立一份数据**」的问题，靠**隔离**。

| 维度 | `synchronized` | `ThreadLocal` |
|---|---|---|
| 解决的问题 | 原子性 + 可见性（共享数据） | 数据隔离（线程私有） |
| 实现机制 | 监视器锁（monitorenter/exit），阻塞 | 每个 Thread 一份副本，无锁 |
| 性能 | 有竞争时上下文切换，可能死锁 | 几乎零开销（一次字段读 + 数组访问） |
| 内存 | 一份数据 | N 份数据（N = 线程数） |
| 适用 | 计数器、库存扣减、单例初始化 | 上下文传递、非线程安全工具复用 |
| 副作用 | 死锁、优先级反转、吞吐下降 | 内存滞留、跨线程传递困难 |

**什么时候用哪个（举三个具体判断）**：

1. 数据**必须全局一致**（如库存、余额）→ 只能 `synchronized` / CAS，ThreadLocal 会导致每个线程看到不同库存，业务直接错；
2. 数据**只需要在本次调用链内可见**（如 `userId`、`traceId`）→ `ThreadLocal`，加锁纯属浪费；
3. 工具类**非线程安全但可以多实例**（`SimpleDateFormat`、`StringBuilder`）→ `ThreadLocal` 复用实例，比每次 new 省，比加锁快。

**加分延伸**：
> 两者不是互斥的，可以叠加。比如「用 ThreadLocal 缓存连接，但连接的创建过程用双重检查锁」；再比如 TTL 的 `capture` 内部就用了 `synchronized` 保护快照结构。

---

### Q12　【字节·二面·必问】线程池里怎么把上下文（比如 traceId、用户信息）传给异步任务？

**考核程度：★★★★**　**考点：这是字节最爱问的实战题**

**详解（必须答出三层：不能用什么 → 用什么 → 怎么保证不串号）**

**第一层：普通 ThreadLocal 不行**
工作线程是复用的，与提交线程是两个线程，`get()` 直接是 null。

**第二层：InheritableThreadLocal 也不行（关键，容易漏）**
`ITL` 只在 `new Thread()` 时复制一次，而线程池的线程**创建后就复用**。第一次提交任务时复制了当时的上下文快照，之后所有任务拿到的**都是那个旧快照**——这就是经典的「数据串号」。

```java
ITL.set("req-1");
pool.submit(() -> print(ITL.get()));   // 第一次：req-1（线程此刻创建，复制了）
ITL.set("req-2");
pool.submit(() -> print(ITL.get()));   // 还是 req-1 ← 灾难
```

**第三层：正确方案有三种，按推荐度排序**

**方案 A（首选）：阿里 TransmittableThreadLocal（TTL）**

```java
private static final TransmittableThreadLocal<String> CTX = new TransmittableThreadLocal<>();

// ① 包装线程池（提交时自动 capture，执行时 replay，完成后 restore）
ExecutorService pool = TtlExecutors.getTtlExecutorService(rawPool);
// ② 或包装单个任务
pool.execute(TtlRunnable.get(() -> System.out.println(CTX.get())));
// ③ 生产推荐：agent 方式无侵入
//    -javaagent:transmittable-thread-local-2.x.jar
```

原理三步：`capture`（提交时把提交线程的 TTL 值做快照）→ `replay`（执行前把快照写进工作线程，并备份原值）→ `restore`（执行后还原，防止污染下一个任务）。

**方案 B：Spring `TaskDecorator`（手写版 TTL）**

```java
@Bean
public TaskDecorator decorator() {
    return task -> {
        String traceId = TraceContext.get();
        Map<String,String> mdc = MDC.getCopyOfContextMap();
        return () -> {
            try {
                TraceContext.set(traceId);
                if (mdc != null) MDC.setContextMap(mdc);
                task.run();
            } finally {
                TraceContext.remove();
                MDC.clear();
            }
        };
    };
}
```

**方案 C：显式传参（最土但最可靠）**
把上下文作为方法参数或任务对象的字段显式传入。没有隐式依赖，没有泄漏风险，可测试性好。**如果不是上下文特别多，这反而是最推荐的工程实践。**

**必须补的一句（加分）**：
> 不管用哪种方案，**`finally` 里的清理都不能省**。因为工作线程复用，不清理就会把 A 请求的 traceId 带到 B 请求，日志排查时会出现「一个 traceId 对应两条不相干的请求」，比 OOM 更难查。

---

### Q13　【字节·二面】`InheritableThreadLocal` 的实现原理是什么？为什么说它"不可靠"？

**考核程度：★★★★**　**考点：源码 + 缺陷理解**

**详解**

**原理（一句话 + 一行源码）**：
> 继承发生在 `new Thread()` 的构造过程里。`Thread.init` 中判断 `inheritThreadLocals && parent.inheritableThreadLocals != null`，就调用 `ThreadLocalMap.createInheritedMap(parentMap)` 复制一份。

`InheritableThreadLocal` 只是重写了 `getMap` / `createMap` 让它用 `inheritableThreadLocals` 字段，并提供了 `childValue()` 钩子（默认 `return parentValue`，是**浅拷贝**）。

**三个"不可靠"**：

1. **时机不可靠**：只在创建线程那一刻复制，之后父线程改值，子线程看不到（是快照，不是共享）；
2. **线程池场景完全失效**：线程复用 → 只在第一次创建时复制 → 上下文"卡死"在旧值；
3. **对象不安全**：`childValue` 默认返回同一个引用，父子线程共享可变对象 → 并发问题。

**加分：`new Thread` 里的 `parent` 是谁？**
> 是**执行 `new Thread()` 那个语句的线程**（`Thread.currentThread()`），不是"线程类层面的父线程"。所以「谁创建我，我继承谁」，不是「谁是我的父线程」。

**追问：那它还有什么用？**
> 适合「一次性创建、生命周期固定的子线程」——比如主线程启动一个后台常驻线程，带上启动配置；或者 fork 一批短期工作线程做批处理。**任何线程池、异步框架、定时任务场景都不适用。**

---

### Q14　【字节·三面·TTL 深挖】TTL 为什么必须在 `finally` 里 `restore`？如果不 restore 会怎样？

**考核程度：★★★★★**　**考点：TTL 设计意图，很多人只会用不会讲**

**详解**

**先讲三步各自的职责**：

| 步骤 | 在哪执行 | 做什么 | 不做会怎样 |
|---|---|---|---|
| `capture` | **提交线程** | 把提交线程当前的 TTL 值做成快照 | 工作线程拿不到任何上下文 |
| `replay` | **工作线程**（任务开始前） | 把快照写进工作线程，并**备份**工作线程原值 | 任务里 `get()` 是 null 或旧值 |
| `restore` | **工作线程**（任务结束后） | 把工作线程还原成备份值 | **上一个任务的上下文残留，污染下一个任务** |

**不 restore 的后果（必须能具体描述）**：

```
  请求 A（traceId=A）→ 提交任务 → replay 后工作线程 CTX = A
  任务执行完，没 restore → 工作线程 CTX 仍然是 A
  请求 B（traceId=B）→ 提交任务 → replay 把 CTX 改成 B ✅（这个场景看着没事）

  但如果是这样：
  请求 A → 任务 X（replay A）
          任务 X 内部提交子任务 Y（没包装 → 没 replay）
                  → 子任务 Y 读到的是残留的 A，而它应该属于别的请求
          任务 X 结束 → restore 没做 → 工作线程 CTX = A 残留

  下一个请求 C（没有设置 CTX，期望是 null）→ 直接执行（没有 replay，因为某些路径漏了包装）
                  → 读到 A ❌ 张冠李戴
```

**核心原因**：
> 线程池的线程是**共享资源**，每次使用都应该「**借时清理、还时还原**」。`restore` 等价于「用完把桌面收拾干净」，是 `try-finally` 语义在线程池里的体现。没有它，ThreadLocal 就从"每任务隔离"退化成"每线程遗留"。

**顺带答一个延伸**：`restore` 之后，工作线程的 TTL 值回到执行前状态（通常是空），因此**不需要额外 `remove`** —— 这正是 TTL 比"手动 set/remove"更可靠的地方：它把清理做进了框架。

---

### Q15　【字节·三面】如果不用 TTL，让你自己设计一个「线程池上下文传递」方案，你会怎么做？要考虑哪些点？

**考核程度：★★★★★**　**考点：设计能力，开放题**

**详解（按"目标—约束—方案—边界"四段回答）**

**目标**：任务在任意线程执行时，都能看到「提交者」的上下文；执行完不影响后续任务。

**约束（先列清楚，体现思维严谨）**：
1. 工作线程复用 → 必须"每次任务前注入、任务后还原"；
2. 提交点分散（`execute`/`submit`/`invokeAll`/`CompletableFuture`/`@Async`）→ 必须覆盖所有入口；
3. 任务内部可能再提交子任务 → 必须可嵌套；
4. 上下文可能很大 → 快照成本要可控；
5. 不能污染业务代码 → 尽量用装饰器/agent，避免"每个任务都手写 try-finally"。

**方案（分三层）**：

**① 数据结构层**：定义 `ContextHolder`，内部是一个 `ThreadLocal<Map<String,Object>>`（或用 TTL）。提供 `capture()` 返回不可变快照、`replay(snapshot)` 返回 backup、`restore(backup)`。

**② 任务包装层**：`ContextRunnable implements Runnable`，构造时 `capture()`（在提交线程），`run()` 里 `backup = replay(); try { target.run(); } finally { restore(backup); }`。`Callable` 同理。

**③ 提交入口层**：装饰 `ExecutorService`（用 `AbstractExecutorService` 的子类或动态代理），重写 `execute/submit/invokeAll/invokeAny`，全部走 `ContextRunnable.wrap()`。嵌套提交因为包装是幂等的，会被自动处理。

**边界处理（加分项，面试官最想听这段）**：
- **幂等包装**：已经包过的任务不再包（用标记字段或 `instanceof` 判断），避免套娃导致快照被覆盖；
- **快照不可变**：`capture` 返回的值要做防御性拷贝，防止提交后又被修改；
- **空上下文短路**：没有上下文时直接返回原任务，零开销；
- **`CompletableFuture` 默认 `ForkJoinPool.commonPool()`**：必须显式传装饰过的 executor，否则漏掉；
- **`@Async`**：用 `TaskDecorator` 接进来；
- **清理兜底**：线程池 `afterExecute` 里再清一次，防止业务代码自己 set 了没清。

**收尾一句**：
> 这套方案的本质是「**把 ThreadLocal 从"线程级状态"改造成"任务级状态"**」——用一次快照 + 一次还原，把「线程复用」这个前提重新对齐到「任务隔离」的语义。

---

## 8.4 腾讯

> 腾讯风格：**概念辨析 + 工程规范**，喜欢问"你会怎么写"、"团队规范怎么定"。

### Q16　【腾讯·一面】ThreadLocal 在什么情况下不会导致内存泄漏？

**考核程度：★★★★**　**考点：反向思维，考边界**

**详解（四种不会泄漏的情况）**

| # | 情况 | 原因 |
|---|---|---|
| 1 | **线程用完即销毁**（非线程池、`new Thread` 后结束） | 线程终止 → `Thread` 对象不可达 → `threadLocals` 整条链被回收 |
| 2 | **正确 `remove`**（`finally` 里调） | Entry 从 table 摘除，value 引用断开 |
| 3 | **value 是小对象且会被覆盖** | 同一线程反复 `set` 同一个 key，旧 value 被新 value 覆盖，旧对象可回收（但注意最后一次的 value 仍滞留） |
| 4 | **value 是 interned 常量 / 无状态单例** | 滞留也无所谓，不占额外内存（如 `Boolean.TRUE`、空集合 `Collections.emptyList()`） |

**但要说清"严格意义"**：
> 第 3、4 条其实只是"泄漏的量可以忽略"，不是"没有泄漏"。真正干净的只有 1 和 2。

**加分**：
> 面试常问"是不是只要 remove 就绝对安全？"——是的，`remove` 会把 `table[i].value = null`、`table[i] = null`、`size--`，并修补探测链，是唯一彻底的清理方式。`set(null)` 不算。

---

### Q17　【腾讯·二面】既然 ThreadLocal 能传上下文，为什么很多团队还禁止用它传业务参数？

**考核程度：★★★**　**考点：工程规范意识**

**详解（讲"隐式依赖"的危害）**

1. **隐式依赖，破坏可测试性**：方法签名里看不出它依赖上下文，单测时忘了 set 就 NPE，而且报错位置离根因很远；
2. **调用链不透明**：一个 `UserContext.getUserId()` 可能来自 3 层之外的拦截器，重构时不敢动；
3. **跨线程/跨框架容易丢**：异步、MQ 消费、定时任务、RPC 线程池，每一处都要单独处理；
4. **生命周期难管理**：忘记 remove 就串号，而且难以在 code review 里发现；
5. **不适合"大量参数"**：上下文塞十几个字段后，就变成了"隐式全局变量"，本质是反模式。

**正确姿势**：
> ThreadLocal 只用于「**跨切面的基础设施信息**」——traceId、租户 ID、语言/时区、数据源路由标记、事务资源。**业务参数必须显式传参**。判断标准：如果这个值只被某一层业务逻辑使用，就应该走参数；如果每一层都可能用到且与业务无关，才考虑 ThreadLocal。

---

### Q18　【腾讯·二面】Spring 里哪些地方用了 ThreadLocal？`@Async` 为什么会丢上下文？

**考核程度：★★★★**　**考点：框架理解**

**详解**

**三个典型**：

1. **`RequestContextHolder`**：`NamedThreadLocal<RequestAttributes>`，让任意层拿到 `HttpServletRequest`。在 `DispatcherServlet.processRequest` 里 set，`finally` 里 `resetRequestAttributes`；
2. **`TransactionSynchronizationManager`**：`ThreadLocal<Map<Object,Object>>` 存事务资源（`ConnectionHolder`），保证同一事务内所有 DAO 用同一个 `Connection`，也支撑事务传播行为；
3. **`LocaleContextHolder`**：存 `Locale` / `TimeZone`。

**`@Async` 丢上下文的原因**：

`@Async` 由 `AsyncExecutionInterceptor` 拦截，把方法调用封装成 `Callable` 提交给 `TaskExecutor`（默认 `SimpleAsyncTaskExecutor` 或配置的 `ThreadPoolTaskExecutor`）。**执行在另一个线程**，而 `RequestContextHolder` 用的是普通 `ThreadLocal`（非 inheritable），所以：

- `RequestContextHolder.getRequestAttributes()` → `null`；
- `MDC` 的 traceId → 空；
- 任何自定义 ThreadLocal → 空。

**解决**：`TaskDecorator`（见 Q12 方案 B），或改用 TTL，或在 `@Async` 方法上显式传参。

**加分**：Spring 6 之后提供了 `ContextPropagatingTaskDecorator`（在 Spring Cloud Sleuth / Micrometer 体系里），思路同样是「装饰任务 + 复制上下文 + 清理」。

---

## 8.5 京东 / 拼多多 / 携程

> 风格：**实战导向**，爱让你写代码。

### Q19　【京东·一面·手写】用 ThreadLocal 写一个线程安全的 `SimpleDateFormat` 工具类

**考核程度：★★★**　**考点：能否写出生产可用的代码（含清理）**

**参考答案**

```java
public final class DateUtils {

    private static final ThreadLocal<SimpleDateFormat> SDF_HOLDER =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    private DateUtils() {}

    public static String format(Date date) {
        return SDF_HOLDER.get().format(date);
    }

    public static Date parse(String text) throws ParseException {
        return SDF_HOLDER.get().parse(text);
    }

    /** 线程池场景必须显式清理，否则 SDF 对象会滞留在线程上 */
    public static void clear() {
        SDF_HOLDER.remove();
    }
}
```

**必须主动说的三个点（体现水平）**：

1. **`SimpleDateFormat` 为什么线程不安全**：内部持有 `Calendar` 实例，`format/parse` 会修改它的 `fields` 数组；多线程并发调用会互相覆盖中间状态，导致日期错乱（甚至抛 `NumberFormatException`）。`parse` 还会因为 `Calendar` 状态污染返回错误日期——这类 bug 极其隐蔽。
2. **JDK 8+ 更好的选择**：`DateTimeFormatter` 是**不可变、线程安全**的，直接 `static final` 单例即可，**根本不需要 ThreadLocal**：

```java
private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
```
> 面试时这么答是加分项——说明你知道「ThreadLocal 不是万能药，能不用就不用」。

3. **`ThreadLocal.withInitial` 的价值**：`SimpleDateFormat` 创建成本不低（解析 pattern），懒初始化保证「不用的线程不付代价」。

**追问：`ThreadLocal` 里的 SDF 会不会因为线程池而泄漏？**
> 会。`SDF_HOLDER` 是 static，key 永远是 null 不了，所以不会被自动清理。必须在请求结束时 `clear()`，或者在拦截器 `afterCompletion` 里统一清。

---

### Q20　【拼多多·二面·设计】用 ThreadLocal 实现动态数据源切换，说说完整方案

**考核程度：★★★★**　**考点：Spring 扩展点 + ThreadLocal 实战**

**详解**

**① 定义上下文持有者**

```java
public final class DataSourceContextHolder {
    private static final ThreadLocal<String> DS_KEY = new ThreadLocal<>();

    public static void set(String dsKey) { DS_KEY.set(dsKey); }
    public static String get() { return DS_KEY.get(); }
    public static void clear() { DS_KEY.remove(); }     // ★ 必须清理
}
```

**② 实现 `AbstractRoutingDataSource`**

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // 每次获取连接时调用，返回当前线程要用的数据源 key
        return DataSourceContextHolder.get();
    }
}
```

**③ 配置**：`targetDataSources` 里放多个真实 `DataSource`，`defaultTargetDataSource` 兜底。

**④ AOP 切换 + 清理**

```java
@Aspect
@Component
public class DsAspect {
    @Around("@annotation(ds)")
    public Object around(ProceedingJoinPoint pjp, DS ds) throws Throwable {
        String origin = DataSourceContextHolder.get();   // 备份（支持嵌套）
        try {
            DataSourceContextHolder.set(ds.value());
            return pjp.proceed();
        } finally {
            if (origin == null) DataSourceContextHolder.clear();
            else DataSourceContextHolder.set(origin);     // 还原，不是清空
        }
    }
}
```

**⑤ 三个坑（必须讲，这是加分点）**

| 坑 | 说明 | 解法 |
|---|---|---|
| **事务与切换的顺序** | `@Transactional` 在 `DataSource` 获取连接**之前**就绑定了连接 → 如果 AOP 顺序错了，切换不生效 | `@DS` 的切面要**优先级高于**事务切面（`@Order` 更小），保证数据源先确定，事务再绑定连接 |
| **嵌套切换** | A 方法切到从库，内部调 B 方法切到另一个库，B 结束必须**还原成 A 的值**（不是清空） | 备份 + 还原（如上代码） |
| **异步/线程池失效** | 上下文在线程 A 设置，任务在线程 B 执行 → 拿不到 → 走到默认数据源（**可能是主库，造成误判**） | 用 TTL 传递，或显式把 dsKey 作为参数 |

**为什么必须 ThreadLocal**：数据源的选择是「**一次调用链内的决策**」，影响整条链上所有 DAO。如果显式传参，每个 Service/Dao 方法都要带一个 `dsKey` 参数，侵入性极强。ThreadLocal 让这个决策「随线程走」，零侵入。

---

### Q21　【携程·二面】线上出现"数据串号"（A 用户的请求拿到了 B 用户的信息），你怀疑是 ThreadLocal，怎么定位？

**考核程度：★★★★**　**考点：真实线上问题排查**

**详解**

**① 先确认现象特征（ThreadLocal 串号的三个典型信号）**

| 信号 | 说明 |
|---|---|
| 现象**概率性、偶发**，与并发量正相关 | 因为要"恰好复用到某个带残留上下文的工作线程" |
| 同一个 traceId 下出现**两个不同用户**的数据 | 上下文被覆盖或残留 |
| 重启后**暂时消失**，压测时必现 | 线程池是新建的，干净；跑久了就脏了 |

**② 代码侧排查（90% 的问题在这里）**

按这个清单逐条查：

```bash
# 找出所有 set 的地方
grep -rn "\.set(" --include="*.java" src/ | grep -i "context\|holder\|threadlocal"
# 找出所有 remove 的地方，数量必须 >= set 的数量
grep -rn "\.remove()" --include="*.java" src/
```

重点看四个地方：
1. **拦截器/过滤器**：`preHandle` 里 set，`afterCompletion` 里有没有 remove？（异步请求走的是 `AsyncHandlerInterceptor` 的 `afterConcurrentHandlingStarted`，容易漏）
2. **异常路径**：`set` 之后抛异常，`remove` 没在 `finally` 里；
3. **异步/MQ 消费**：主线程 set 了，异步线程没传也没清；
4. **`@Async` / 线程池**：没用 TTL / TaskDecorator。

**③ 加日志验证**

```java
// 在 set 时打印线程名 + 值 + 请求 ID
log.info("set ctx, thread={}, traceId={}, userId={}", Thread.currentThread().getName(), traceId, userId);
// 在 get 时打印
log.info("get ctx, thread={}, traceId={}, userId={}", Thread.currentThread().getName(), traceId, userId);
```

如果看到「**同一个线程名，先出现 traceId=A 的 set，然后出现 traceId=B 的 get**」→ 确认是残留污染。

**④ 运行期验证（不重启）**

```bash
# Arthas：直接看某个线程的 ThreadLocalMap
arthas-boot> vmtool --action getInstances --className java.lang.ThreadLocal --limit 10
# 或 watch 某个 ThreadLocal 的 get 调用
arthas-boot> watch com.xxx.ContextHolder get '{params, returnObj, throwException}' -x 3
```

**⑤ 修复 + 防复发**

- 所有 `set` 统一封装成 `try-finally`；
- 在线程池 `afterExecute` 里做**全局兜底清理**；
- 用 ArchUnit / 自定义 Checkstyle 规则，禁止裸调 `ThreadLocal.set`（必须走封装工具）；
- 压测时专门跑「长稳压测」而不是短时压测——短压测线程池刚建，掩盖问题。

**加分收尾**：
> ThreadLocal 串号的本质是「**线程复用 + 清理缺失**」，所以排查的核心不是"看 ThreadLocal 哪里写错了"，而是"**穷举所有 set 的入口，检查每个入口的清理是否在 finally 里、是否覆盖了异步路径**"。

---

## 8.6 百度 / 快手 / 网易 / 通用高频

### Q22　【百度·三面】如果让你实现一个「比 ThreadLocal 更快的线程本地变量」，你会怎么做？

**考核程度：★★★★**　**考点：能否讲出 Netty 的索引寻址思路**

**详解**

**先定位 ThreadLocal 的三个性能损耗点**：

1. **哈希 + 线性探测**：冲突时 O(n)，且探测过程中要读多个 Entry；
2. **墓碑扫描**：`expungeStaleEntry` 要扫、要搬元素；
3. **对象开销**：每个 Entry 是一个对象（含对象头、弱引用结构），cache 不友好。

**对应三个优化方向**：

| 优化 | 做法 | 收益 |
|---|---|---|
| **索引寻址替代哈希** | 每个 ThreadLocal 创建时从 `AtomicInteger` 申请一个**固定下标**，直接 `array[index]` 访问 | 消除哈希冲突，恒 O(1) |
| **纯数组替代 Entry 对象** | 用 `Object[]` 直接存 value，不再有 `Entry` 对象 | 省对象头、cache 友好 |
| **整表清理替代墓碑扫描** | 提供 `removeAll()`，任务结束一次性清空 | 无探测链修补，O(n) 但只做一次 |

这就是 Netty `FastThreadLocal` 的思路：

```
每个 FastThreadLocal 分配 index = 0,1,2,3...
InternalThreadLocalMap.indexedVariables : Object[]
        [0]    [1]    [2]    [3]
         A      B      C      D
                       ↑
               FTL_C.index = 2 → array[2]，直接下标访问
```

**但要讲清代价（体现深度）**：
- **数组会随 ThreadLocal 数量线性增长**，且**不能回收下标**（回收会导致复用冲突），所以不适合"动态创建/销毁大量 ThreadLocal"的场景；
- 需要**专用线程类型**（`FastThreadLocalThread`），普通线程上会退化成慢速实现（`SlowThreadLocalMap`）；
- 需要**框架级兜底**（`FastThreadLocalRunnable` 在 finally 里 `removeAll`），否则一样泄漏。

**结论式收尾**：
> ThreadLocal 是「**通用**」方案（任意数量、任意线程、自动回收），FastThreadLocal 是「**受约束的高性能**」方案（数量固定、专用线程、显式清理）。JDK 不采用索引方案，正是因为 ThreadLocal 的数量不可控。

---

### Q23　【快手·二面】ThreadLocal、InheritableThreadLocal、TransmittableThreadLocal 三者怎么选？

**考核程度：★★★★**　**考点：一张表讲清**

**详解**

| 场景 | 选择 | 理由 |
|---|---|---|
| 单线程内传递上下文（Web 请求、同步调用链） | **ThreadLocal** | 最简单、零额外开销 |
| `new Thread()` 创建的短生命周期子线程，且父线程值不再变 | **InheritableThreadLocal** | 创建时自动复制，无需框架 |
| 线程池 / `@Async` / `CompletableFuture` / MQ 消费线程 | **TransmittableThreadLocal** | 唯一能"每次任务都传递并还原"的方案 |
| 有框架钩子（Spring `TaskDecorator`、Dubbo Filter） | **TaskDecorator + ThreadLocal** | 不引第三方依赖，可控 |
| 上下文极多、跨线程频繁 | **显式传参 / 上下文对象** | 避免隐式依赖，可测试 |
| 虚拟线程 + 只读上下文（未来） | **ScopedValue**（预览） | 不可变、自动释放、无泄漏 |

**一句话记忆**：
> **同线程用 TL，new Thread 用 ITL，线程池用 TTL，能传参就别用 TL。**

---

### Q24　【网易·三面·新题】虚拟线程（JDK 21）会对 ThreadLocal 造成什么影响？`ScopedValue` 是什么，能替代它吗？

**考核程度：★★★（但会答的人很少，是加分题）**　**考点：技术前瞻**

**详解**

**影响一：内存放大**

虚拟线程可以轻松创建上百万个，每个仍然是一个 `Thread` 对象、仍然有 `threadLocals` 字段。如果每个虚拟线程都塞一份上下文：

```
100 万虚拟线程 × 每个 1KB 上下文 = 1GB
```

这在平台线程时代是不可想象的（平台线程本身就要 1MB 栈），但虚拟线程的栈是"按需增长"的，**ThreadLocal 反而成了新的内存大头**。

**影响二：清理更难保证**

虚拟线程生命周期短、创建频繁、由 `StructuredTaskScope` 或框架管理，业务代码很难保证每个都 `remove`。

**影响三：`ThreadLocal` 在虚拟线程里仍有语义，但不再是最佳实践**

JDK 明确建议：**虚拟线程场景优先用 `ScopedValue`**。

**`ScopedValue` 的核心差异**

```java
static final ScopedValue<UserInfo> USER = ScopedValue.newInstance();

ScopedValue.where(USER, userInfo).run(() -> {
    // 作用域内可读，且【不可变】
    service.doWork();
});
// 出了 run()，绑定自动失效，无需 remove
```

| 维度 | ThreadLocal | ScopedValue |
|---|---|---|
| 可变性 | 可变 | **不可变**（绑定即定值） |
| 释放 | 手动 remove | **作用域结束自动释放** |
| 传递 | 需 ITL/TTL | **结构化并发自动继承**（fork 子任务可见） |
| 内存 | 每线程一份 | 绑定共享，几乎零额外内存 |
| 状态 | 正式（JDK 1.2 起） | **预览**（JDK 21 起 JEP 446，多轮预览演进中） |

**为什么 ScopedValue 天然不泄漏**：它是"**栈式绑定**"——`where(...).run(...)` 进出作用域时自动 push/pop，不需要业务代码记得清理。**这把"清理"从"人的责任"变成了"语言的责任"**。

**但要客观（体现判断力）**：
> `ScopedValue` 目前还是预览特性，且语义是**只读**的——像"动态数据源切换"这种需要中途改值的场景，它替代不了 ThreadLocal。主流框架（Spring、TTL、日志 MDC）短期内仍依赖 ThreadLocal。所以现在是"**新场景用 ScopedValue，存量继续 ThreadLocal + 严格 remove**"的过渡期。

---

### Q25　【通用·一面】ThreadLocal 能存基本类型吗？会有什么问题？

**考核程度：★★**　**考点：细节，容易答漏**

**详解**

**语法上不能**：`ThreadLocal<int>` 编译不过，泛型必须是引用类型。但可以用包装类：

```java
ThreadLocal<Integer> tl = ThreadLocal.withInitial(() -> 0);
tl.set(1);
```

**问题一：自动装箱/拆箱的性能与内存开销**

每次 `set(1)` 会 `Integer.valueOf(1)`，`get()` 返回 `Integer` 再拆箱。小整数走 `IntegerCache`（-128~127）没事，但频繁 set 大数值会创建大量短命对象，加重 Young GC。

**问题二：`get()` 返回 null 导致拆箱 NPE（最容易踩的坑）**

```java
ThreadLocal<Integer> tl = new ThreadLocal<>();     // 没重写 initialValue
int n = tl.get();                                  // NPE！null 拆箱
```

**解法**：一定要用 `withInitial()` 给默认值，或者先判空：

```java
ThreadLocal<Integer> tl = ThreadLocal.withInitial(() -> 0);
```

**问题三：`get()` 后自增不是原子的（虽然单线程内安全，但容易误解）**

```java
tl.set(tl.get() + 1);        // 单线程内没问题，但如果有跨线程传递就是坑
```

**加分回答**：
> 如果只是需要一个线程级的计数器，用 `ThreadLocal<Integer>` 不如用「局部变量 + 返回值」；如果确实需要跨方法，考虑用 `AtomicInteger` 放进 ThreadLocal，或者直接用 `LongAdder`（如果确实需要全局计数）。

---

### Q26　【通用·压轴】ThreadLocal 能不能替代锁实现事务？说说 Spring 事务和 ThreadLocal 的关系

**考核程度：★★★★★**　**考点：把知识点串成体系，这是"高级"和"中级"的分界线**

**详解**

**先纠正提问**：ThreadLocal **不能**替代锁，它和事务的关系是"**协作**"而非"替代"。

**Spring 事务的三件事，ThreadLocal 各扮演什么角色**：

```
  ① 连接绑定（ThreadLocal 干的）
     TransactionSynchronizationManager.resources
       = ThreadLocal<Map<Object, Object>>
     存 { DataSource → ConnectionHolder }
     ↓ 作用：保证同一事务内 N 次 DAO 调用拿到【同一个 Connection】

  ② 事务边界控制（AOP 干的，不是 ThreadLocal）
     @Transactional → TransactionInterceptor
       → PlatformTransactionManager.getTransaction()  (开启/挂起/加入)
       → 业务方法
       → commit / rollback

  ③ 并发安全（数据库 + 锁干的，不是 ThreadLocal）
     行锁/间隙锁 → 保证多个事务并发时的隔离性
     ThreadLocal 在这里【什么都不做】
```

**关键结论（三句）**：

1. **ThreadLocal 解决的是"同一线程内资源共享"**——让 `Service` 里调的两个 `Dao` 用的是同一个 `Connection`，而不是各自去连接池借一个（那样就不是同一个事务了）；
2. **事务的原子性靠数据库的锁和日志**，不是靠 ThreadLocal 或 Java 锁；
3. **`ThreadLocal` 一旦跨线程就断链**——这就是为什么 `@Transactional` 方法里开新线程做数据库操作，**那个操作不在事务里**（它拿不到绑定的 Connection，会从连接池另借一个，自动提交）。

**这个"跨线程断链"是极经典的坑，面试可以展开**：

```java
@Transactional
public void transfer() {
    accountDao.debit(from, 100);         // ✅ 在事务里
    new Thread(() -> {
        accountDao.credit(to, 100);      // ❌ 不在事务里！拿的是新连接，自动提交
    }).start();
    // 主线程事务回滚，但子线程已经提交了 → 钱凭空多出来
}
```

**正确做法**：不要在线程内开新线程做同事务的 DB 操作；必须异步就改用「本地消息表 / 事务消息 / TCC」等分布式事务方案；如果只是查询，可以异步但要接受"读不到未提交数据"。

**收尾（体现体系化认知）**：
> ThreadLocal 在整个事务体系里承担的是「**把"当前事务的物理连接"这个资源，绑定到执行线程上**」这一件事。它解决的是"资源复用与一致"，不解决"原子性"和"并发隔离"。把这三者混为一谈，是最典型的理解偏差。

---

## 8.7 灵魂追问清单（面试官可能连续挖的 10 个坑）

> 这一节是「压力测试」预设。每题给「一句话答案」，用来在被追问时不慌。

| # | 追问 | 一句话答案 |
|---|---|---|
| 1 | ThreadLocal 的 key 是强引用会怎样？ | 永远无法判定过期，连清理机会都没有，ThreadLocal 对象和 value 双重泄漏 |
| 2 | 为什么不用 `ReferenceQueue` 清理？ | Map 是线程私有的，没有其他线程能通知它；队列消费者只能是自己，不如探测时顺手扫 |
| 3 | `set(null)` 和 `remove()` 一样吗？ | 不一样。`set(null)` 保留 Entry 占位，`remove()` 摘除 Entry 并 size-- |
| 4 | ThreadLocalMap 会死循环吗？ | 扩容时是**单线程**顺序 rehash，无并发，不会像 JDK7 HashMap 那样成环 |
| 5 | 一个线程能有多少个 ThreadLocal？ | 理论上无限，但超过 2/3 容量会扩容；实践上应控制在个位数到几十 |
| 6 | 子线程能改父线程的 ThreadLocal 吗？ | 不能。ITL 是快照复制，双向都不通 |
| 7 | 线程池的线程死了，ThreadLocal 会清吗？ | 会。线程对象不可达 → 整条引用链被回收（这也是"非池化线程不泄漏"的原因） |
| 8 | ThreadLocal 的 value 什么时候会被 GC？ | 只有三种：`remove`、被新值覆盖且旧值无其他引用、Entry 所在的 Thread 被回收 |
| 9 | 为什么 ThreadLocal 不用 `volatile`？ | 单线程访问，不需要跨线程可见性；跨线程传递时的可见性由 `start()`/队列的 happens-before 保证 |
| 10 | ThreadLocal 能提升性能吗？ | 能——用"每线程一份"消除锁竞争。典型例子是 `SimpleDateFormat` 复用和 `ThreadLocalRandom` |

---

# 第九章　最佳实践 Checklist

## 9.1 编码规范（10 条，可直接贴进团队规范）

- [ ] **1. 一律 `private static final`**。避免每次 new 导致 table 膨胀、墓碑增多。
- [ ] **2. 必须用 `withInitial()` 或重写 `initialValue()` 给默认值**。否则 `get()` 返回 null，拆箱或 `.length()` 直接 NPE。
- [ ] **3. 所有 `set` 必须配 `try-finally remove()`**。异常路径是最常见的泄漏来源。
  ```java
  CTX.set(v);
  try { doWork(); } finally { CTX.remove(); }
  ```
- [ ] **4. 禁止用 `set(null)` 代替 `remove()`**。前者不释放槽位、不触发清理。
- [ ] **5. 不要存大对象**。ThreadLocal 的 value 会跟着线程活很久，大对象（大 List、结果集、图片 byte[]）会造成成倍放大。
- [ ] **6. value 尽量用不可变对象**。避免跨线程传递时出现共享可变状态问题。
- [ ] **7. 不要把业务参数塞进 ThreadLocal**。只放「跨切面的基础设施信息」（traceId、租户、数据源标记）。
- [ ] **8. 异步/线程池场景必须用 TTL 或 `TaskDecorator`**，并且清理不能省。
- [ ] **9. 拦截器/过滤器里 set 的，必须在 `afterCompletion` 里清**（注意异步请求走 `afterConcurrentHandlingStarted`）。
- [ ] **10. 统一封装工具类**，禁止业务代码裸调 `ThreadLocal.set/get`。

## 9.2 推荐的工具类封装

```java
public final class TraceContext {

    private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();
    private static final ThreadLocal<String> TENANT_ID = new ThreadLocal<>();

    private TraceContext() {}

    public static void setTraceId(String id) { TRACE_ID.set(id); }
    public static String getTraceId() { return TRACE_ID.get(); }
    public static void setTenantId(String id) { TENANT_ID.set(id); }
    public static String getTenantId() { return TENANT_ID.get(); }

    /** 一次性清理所有，避免漏清 */
    public static void clear() {
        TRACE_ID.remove();
        TENANT_ID.remove();
    }

    /** 模板方法：保证 set/remove 成对出现，业务代码无法忘记清理 */
    public static <T> T callWithTrace(String traceId, Supplier<T> action) {
        String origin = TRACE_ID.get();
        TRACE_ID.set(traceId);
        try {
            return action.get();
        } finally {
            if (origin == null) TRACE_ID.remove();
            else TRACE_ID.set(origin);     // 支持嵌套
        }
    }
}
```

## 9.3 线程池专项兜底（推荐加到基础设施层）

```java
public class ContextAwareThreadPool extends ThreadPoolExecutor {

    public ContextAwareThreadPool(int core, int max, long keepAlive, TimeUnit unit,
                                  BlockingQueue<Runnable> queue) {
        super(core, max, keepAlive, unit, queue);
    }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        // 兜底清理：不管业务代码有没有清，任务结束一律清干净
        TraceContext.clear();
        MDC.clear();
        super.afterExecute(r, t);
    }
}
```

> ⚠️ 注意：`afterExecute` 只能清「当前线程」的上下文，无法清「任务内部创建的 ThreadLocal」——所以业务侧的 `finally remove` 依然是第一道防线，这里只是**最后一道保险**。

## 9.4 反模式清单（Code Review 看到就直接打回）

| 反模式 | 危害 |
|---|---|
| `new ThreadLocal<>()` 写在方法体内 | 每次调用 new 一个，table 被塞满，墓碑堆积 |
| `CTX.set(x); ... CTX.remove();` 中间没 `finally` | 异常即泄漏 |
| 用 ThreadLocal 在 `Controller` 里存整个 `HttpServletRequest` | 大对象滞留，且 `request` 本身有生命周期管理 |
| 用 ThreadLocal 传「10 个业务字段」 | 隐式全局变量，重构噩梦 |
| 在线程池任务里 `set` 但没清，指望 GC | 线程不死，永远不清 |
| `InheritableThreadLocal` 用于线程池 | 上下文卡在第一次创建时的快照 |
| 把 `ThreadLocal<SimpleDateFormat>` 当全局日期工具但从不清理 | static 导致自动清理失效 |

---

# 第十章　面试前 5 分钟速记卡

## 10.1 全流程一张图

```
        ThreadLocal.set(v)
                │
                ▼
   t = currentThread()
                │
                ▼
   map = t.threadLocals ──(null?)──▶ createMap(t, v)   ← 懒加载，容量 16，阈值 10
                │
                ▼
   i = hash & (len-1)        ← hash = 累加 0x61c88647（黄金分割数）
                │
                ▼
   ┌────────────────────────────────────────────────┐
   │ 线性探测（nextIndex 环形前进）                    │
   │   tab[i] == null ────────────▶ 新建 Entry，size++ │
   │   key == this ───────────────▶ 覆盖 value        │
   │   key == null（墓碑）─────────▶ replaceStaleEntry │
   │                                  （复用槽 + 清理） │
   └────────────────────────────────────────────────┘
                │
                ▼
   !cleanSomeSlots(i, size) && size >= threshold
                │
                ▼
   rehash() → expungeStaleEntries() → size >= 3/4*threshold ?
                │
                ▼
   resize()：len *= 2，重算位置，threshold = len*2/3
```

## 10.2 必须记住的 8 个数字

| 数字 | 含义 |
|---|---|
| **16** | 初始容量（必须是 2 的幂） |
| **2/3** | 负载因子（threshold = len × 2/3，初始为 10） |
| **0x61c88647** | hash 步长（黄金分割数） |
| **2 倍** | 扩容倍数 |
| **log₂(n)** | `cleanSomeSlots` 的默认扫描次数 |
| **3/4** | rehash 后触发 resize 的判定系数（`threshold - threshold/4`） |
| **2** | Thread 里的 Map 字段个数（`threadLocals` / `inheritableThreadLocals`） |
| **1** | 每个 ThreadLocal 在每个线程里最多一个 Entry |

## 10.3 三句话总结（能背下来就够应付 60% 的追问）

1. **数据存在 Thread 里，不在 ThreadLocal 里**：`Thread.threadLocals` → `ThreadLocalMap` → `Entry[]`，key 是 ThreadLocal 的弱引用，value 是强引用。
2. **弱引用让 key 可被回收，从而产生可判定的墓碑；但 value 仍是强引用，所以仍需探测式清理 + 业务侧 `remove`**。
3. **线程池把「线程用完即死」变成了「线程长期存活」，所以必须显式 `remove`；跨线程传递需要 TTL（capture / replay / restore）**。

## 10.4 答题模板（30 秒版 vs 3 分钟版）

**30 秒版（一面开场）**
> ThreadLocal 是线程本地变量。值存在每个 Thread 对象的 `threadLocals` 字段里，也就是 `ThreadLocalMap`。Map 的 key 是 ThreadLocal 的弱引用，value 是强引用。定位用 `threadLocalHashCode & (len-1)`，冲突用线性探测。因为 Map 是线程私有的，所以完全无锁。风险是线程池场景下 value 会随线程滞留，必须在 `finally` 里 `remove`。

**3 分钟版（二面深入）**
> 在 30 秒版基础上补三块：
> ① **设计权衡**：为什么 Map 挂在 Thread 上（无锁 + 自动回收 + O(1)），以及为什么不用 HashMap（弱引用、墓碑清理、省内存）；
> ② **泄漏完整链条**：引用链四段 → 弱引用解决"可诊断" → value 强引用导致滞留 → 三类清理入口（set/get/remove）→ 机会性清理的局限 → static 反而让自动清理失效；
> ③ **跨线程方案**：ITL 只在 `new Thread` 时复制，线程池失效 → TTL 的 capture/replay/restore，以及为什么 restore 必须存在。

**5 分钟版（三面/定级）**
> 再加：一个真实踩坑案例（重点说"现象 → 排查手段 → 根因 → 修复 → 防复发"）、Netty `FastThreadLocal` 的索引寻址优化、虚拟线程 / `ScopedValue` 的方向判断。

## 10.5 自测 10 题（能全对说明掌握到位）

| # | 题目 | 答案 |
|---|---|---|
| 1 | ThreadLocalMap 定义在哪个类里？ | `java.lang.Thread`（作为静态内部类定义在 ThreadLocal，但**字段**在 Thread） |
| 2 | 初始容量和阈值？ | 16 和 10 |
| 3 | hash 怎么算？ | `AtomicInteger` 累加 `0x61c88647`，创建时固定 |
| 4 | 冲突怎么解决？ | 线性探测（开放地址法） |
| 5 | key 是什么引用？value 呢？ | 都是强引用，只有 key 被 `WeakReference` 包装 |
| 6 | `remove` 除了清 value 还做什么？ | `e.clear()`、`table[i] = null`、`size--`，并 `expungeStaleEntry` 修复探测链 |
| 7 | `cleanSomeSlots` 扫几次？ | 默认 `log₂(n)` 次，发现墓碑则重置为 `len` 再扫 |
| 8 | rehash 后一定扩容吗？ | 不一定，要 `size >= threshold - threshold/4` 才 resize |
| 9 | ITL 在哪一步复制？ | `Thread.init` 里，`new Thread()` 时由创建者复制 |
| 10 | TTL 的三步？ | capture（提交线程）/ replay（执行前）/ restore（执行后，`finally`） |

---

## 附录 A　版本差异速查

| 版本 | 变化 |
|---|---|
| JDK 1.2 | `ThreadLocal` 引入，内部是 `Map<Thread, Object>`（早期实现不同） |
| JDK 5.0 | 改为 `Thread` 持有 `ThreadLocalMap` 的结构（现代版本的基础） |
| JDK 8 | 新增 `ThreadLocal.withInitial(Supplier)`、`SuppliedThreadLocal` |
| JDK 8+ | `ThreadLocalMap` 核心结构（弱引用 Entry + 线性探测 + 探测式清理）基本稳定至今 |
| JDK 21 | 虚拟线程正式发布（JEP 444），ThreadLocal 语义不变但内存特性改变；`ScopedValue` 以预览形式引入（JEP 446），后续多轮预览演进 |

## 附录 B　关键源码位置（想深挖可以按这个索引去看）

| 内容 | 位置 |
|---|---|
| `Thread.threadLocals` 字段 | `java.lang.Thread` 类头部 |
| `Thread.init` 中的继承逻辑 | `java.lang.Thread#init` |
| `ThreadLocal.set/get/remove` | `java.lang.ThreadLocal` |
| `ThreadLocalMap` 全部实现 | `java.lang.ThreadLocal.ThreadLocalMap` |
| `Entry` 定义 | `java.lang.ThreadLocal.ThreadLocalMap.Entry` |
| `expungeStaleEntry` | 同上，核心清理函数 |
| `cleanSomeSlots` / `rehash` / `resize` | 同上 |
| `InheritableThreadLocal` | `java.lang.InheritableThreadLocal` |
| `FastThreadLocal` | `io.netty.util.concurrent.FastThreadLocal` / `InternalThreadLocalMap` |
| TTL | `com.alibaba.ttl.TransmittableThreadLocal` / `TtlRunnable` / `TtlExecutors` |

## 附录 C　一句话收尾（面试结束时可以说）

> ThreadLocal 不是一个"缓存工具"，而是一套「**把并发问题转成隔离问题**」的设计范式。它的全部复杂性都来自两个约束：**要能自动回收（弱引用 + 墓碑）**，**又要能跨线程使用（ITL → TTL）**。理解了这两个约束，源码里的每个细节就都有了解释。




