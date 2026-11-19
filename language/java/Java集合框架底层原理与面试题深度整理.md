# Java 集合框架底层原理 · 深度知识手册 + 大厂面试题库

> 适用版本：以 **JDK 8 / JDK 11 / JDK 17** 为主线，关键处对比 **JDK 7**（HashMap、ConcurrentHashMap 的演进是面试最大考点之一）。
> 整理日期：2026-11-19
> 阅读建议：第 1～4 章是「原理地基」，第 5～8 章是「逐类深挖」，第 9 章是「横向对比与选型」，第 10 章是「面试题库（带详解）」。每一章末尾都有 `⚠️ 易错点` 与 `💡 延伸`。

---

## 目录

| 章节 | 内容 | 重要度 |
|---|---|---|
| 第 1 章 | 集合框架全景：接口体系与设计思想 | ★★★★ |
| 第 2 章 | 底层数据结构地基：数组 / 链表 / 哈希表 / 红黑树 / 跳表 / 堆 | ★★★★★ |
| 第 3 章 | ArrayList、LinkedList、Vector、CopyOnWriteArrayList | ★★★★★ |
| 第 4 章 | HashMap 全链路拆解（含 JDK7 vs JDK8） | ★★★★★ |
| 第 5 章 | LinkedHashMap、TreeMap、Hashtable | ★★★★ |
| 第 6 章 | Set 家族与 Queue 家族（ArrayDeque、PriorityQueue、阻塞队列） | ★★★★ |
| 第 7 章 | ConcurrentHashMap 全链路拆解（JDK7 分段锁 → JDK8 CAS） | ★★★★★ |
| 第 8 章 | 迭代器、fail-fast / fail-safe、集合工具类与经典坑 | ★★★★ |
| 第 9 章 | 横向对比总表与选型决策 | ★★★★★ |
| 第 10 章 | 大厂面试题库（60+ 题，标注考核程度 + 详解 + 追问链） | ★★★★★ |

---

# 第 1 章 集合框架全景：接口体系与设计思想

## 1.1 一句话理解集合框架

> **集合 = 把「一堆元素 + 对元素的组织规则 + 对元素的操作算法」封装成一个对象。**
> 数组是最朴素的集合（只有「连续内存 + 下标」），而集合框架是在数组/链表这些物理结构之上，抽象出「逻辑结构（List/Set/Queue/Map）」和「算法（查找、排序、遍历、并发控制）」。

## 1.2 接口层次结构（必须背下来的那张图）

```
java.lang.Iterable                       ← 一切可 foreach 的根
  └── java.util.Collection               ← 集合（元素容器）的根接口
        ├── List        有序、可重复、有下标
        │     ├── ArrayList
        │     ├── LinkedList（同时实现 Deque）
        │     ├── Vector（遗留，synchronized）
        │     │     └── Stack（遗留，LIFO）
        │     └── CopyOnWriteArrayList（JUC）
        ├── Set         无序（或按某规则有序）、不可重复
        │     ├── HashSet        → 底层 HashMap
        │     ├── LinkedHashSet  → 底层 LinkedHashMap（插入序/访问序）
        │     ├── TreeSet        → 底层 TreeMap（排序）
        │     ├── CopyOnWriteArraySet（JUC，底层 CopyOnWriteArrayList）
        │     ├── ConcurrentSkipListSet（JUC，底层 ConcurrentSkipListMap）
        │     └── EnumSet（位向量实现，极快）
        └── Queue       队列（FIFO 为主）
              ├── Deque（双端队列）
              │     ├── ArrayDeque
              │     ├── LinkedList
              │     └── ConcurrentLinkedDeque（JUC）
              ├── PriorityQueue（优先队列/堆）
              ├── ConcurrentLinkedQueue（JUC，无锁链表）
              └── BlockingQueue（阻塞队列，JUC）
                    ├── ArrayBlockingQueue
                    ├── LinkedBlockingQueue
                    ├── PriorityBlockingQueue
                    ├── DelayQueue
                    ├── SynchronousQueue
                    └── LinkedTransferQueue

java.util.Map                              ← 不属于 Collection！是「键值映射」
  ├── HashMap
  │     └── LinkedHashMap（插入序 / 访问序，可做 LRU）
  ├── Hashtable（遗留，全表锁）
  ├── TreeMap（红黑树，有序）
  ├── WeakHashMap（弱键，GC 可回收）
  ├── IdentityHashMap（== 比较，不用 equals）
  ├── EnumMap（枚举专用，数组实现，最快）
  ├── Properties（Hashtable 子类，配置文件）
  ├── ConcurrentHashMap（JUC）
  └── ConcurrentSkipListMap（JUC，跳表，有序并发）
```

### 三个「反直觉」的设计点（面试常问）

1. **Map 不继承 Collection**。
   - 原因：Collection 的语义是「一组单独的元素的容器」，其 `add(E)`、`iterator()`、`contains(Object)` 都是**单元素语义**；而 Map 的元素是 `Entry<K,V>` 二元组，`keySet()/values()/entrySet()` 才是把 Map「投影」成 Collection。若 Map 继承 Collection，`add` 的语义会冲突（加一个 key 还是加一个 entry？），且 `toArray()` 也无意义。
2. **Set 底层几乎都是 Map**。
   - HashSet 的 `add(e)` 就是 `map.put(e, PRESENT)`；TreeSet 就是 `TreeMap` 的 key 集合视图。所以「Set 的原理」= 「Map 的原理 + value 恒为一个哨兵对象」。
3. **LinkedList 同时是 List 和 Deque**，但它作为 List 的性能极差（`get(i)` 是 O(n)），实际工程中几乎只当 Deque/Queue 用。

## 1.3 集合框架用了哪些设计模式

| 模式 | 体现 | 说明 |
|---|---|---|
| **迭代器模式** | `Iterator`、`ListIterator` | 把「遍历」从「容器结构」中解耦，容器只需暴露 `iterator()` |
| **适配器模式** | `Arrays.asList()`、`Collections.unmodifiableList()`、`subList()`、`keySet()` | 把一个对象包装成另一个接口形态；注意这些多为**视图（view）**，不是拷贝 |
| **装饰器模式** | `Collections.synchronizedList()`、`Collections.unmodifiableList()` | 在原集合外再包一层，拦截并增强方法 |
| **工厂/静态工厂** | `List.of()`、`Map.of()`、`Collections.emptyList()` | JDK9+ 的不可变集合工厂 |
| **模板方法** | `AbstractList`、`AbstractMap` | 抽象类定义骨架（如 `addAll` 循环调 `add`），子类实现原子方法 |
| **策略模式** | `Comparator`、`HashMap` 的 `loadFactor` | 把「比较/装载」策略外置 |
| **享元模式** | `Integer.valueOf` 缓存、`Collections.EMPTY_LIST` 共享空集合 | 复用不可变对象 |

> 💡 **延伸：为什么 JDK9 之后 List.of 不允许 null？**
> 因为不可变集合的设计目标是「可安全共享、可做常量、序列化语义确定」。允许 null 会让 `List.of(a, b)` 在序列化/反序列化、`equals`、`Stream` 的 `map` 链路里产生大量 NPE 分支；同时 JDK 想引导开发者用 `Optional` 表达「可能为空」。另外 `List.of()` 的空实现是单例，含 null 会破坏共享语义。

---

# 第 2 章 底层数据结构地基

> 这一章是全书的地基。**所有集合的「为什么」最终都能落到这几张图上。**

## 2.1 数组：唯一被 JVM 特殊对待的容器

### 2.1.1 内存模型

```
Java 数组对象在堆上的布局（64 位 JVM，开启压缩指针 UseCompressedOops）：
┌──────────────────────────────────────────────────────┐
│ Mark Word (8B)  │ Klass Pointer (4B) │ length (4B)   │  ← 对象头 16B
├──────────────────────────────────────────────────────┤
│ element[0] │ element[1] │ element[2] │ ... │ 对齐填充  │  ← 元素区
└──────────────────────────────────────────────────────┘
```

关键点：
- **length 存在对象头里**，所以 `.length` 是 O(1)，且数组越界能精确抛 `ArrayIndexOutOfBoundsException`；
- 元素是**引用（4B 压缩指针 / 8B）** 还是**基本类型值**，决定内存占用（`int[10]` 是 40B 元素区，`Integer[10]` 是 40B 引用 + 最多 10 个 Integer 对象）；
- 数组是**连续内存**，所以 `a[i]` 的地址 = `base + i * elementSize`，这是 O(1) 随机访问的物理原因，也是 **CPU 缓存行友好**（顺序遍历时预取器命中率高）的物理原因。

### 2.1.2 为什么 `System.arraycopy` 比 for 循环快

`System.arraycopy` 是 `native` 方法， HotSpot 内部有专门实现：
1. 先做**类型检查**（是否同为引用数组、是否基本类型匹配），不匹配抛 `ArrayStoreException`；
2. 对基本类型数组走 `CopyMemory`（memmove 语义，可向量化）；
3. 对引用数组走**写屏障友好**的批量拷贝（G1/ZGC 下有专门的 card marking 优化）；
4. 小数组会走**内联快速路径**，避免 native 调用开销。

> ⚠️ **易错点**：`System.arraycopy(src, srcPos, dest, destPos, length)` **允许源和目标是同一个数组**，且能正确处理重叠（`memmove` 语义）。而 `Arrays.copyOf` 是「新建数组 + arraycopy」，`Arrays.copyOfRange` 是区间版。

### 2.1.3 数组的三个先天缺陷（决定了为什么要链表/树）

| 缺陷 | 后果 | 补救结构 |
|---|---|---|
| 容量固定 | 必须扩容（拷贝 O(n)） | ArrayList 的 grow、ArrayDeque 的 doubleCapacity |
| 中间插入/删除要搬移 | O(n) | LinkedList（O(1) 改指针）、跳表 |
| 查找只能靠下标或遍历 | 无序时 O(n) | 哈希表（O(1)）、树/跳表（O(log n)） |

## 2.2 链表

### 2.2.1 单链表 / 双链表 / 循环链表

```
单链表：head → [A|next] → [B|next] → [C|null]
双链表：null ←[prev|A|next] ⇄ [prev|B|next] ⇄ [prev|C|next]→ null
循环双链表：head.prev = tail, tail.next = head（LinkedList 不是循环，ArrayDeque 是循环数组）
```

- **单链表**：删除某节点必须知道前驱 → 通常要「双指针」或哨兵头节点；
- **双链表**：`Node{item, next, prev}`，删除自身 O(1)（因为能拿到 prev），这是 `LinkedList.unlink()` 能 O(1) 的原因；
- **哨兵（sentinel/dummy）节点**：省掉 head/tail 的 null 判断，Redis 的 adlist、JDK 的 `LinkedHashMap` 头结点都是这个思路。

### 2.2.2 链表的物理代价（面试高频：为什么链表实际比数组慢）

在**理论上**链表插入 O(1)，但**工程上**：
1. **缓存不友好**：节点散落在堆上，每次 `next` 都可能是一次 cache miss（约 100 个时钟周期），而数组顺序访问是「一次 miss 预取一整行」；
2. **内存开销**：64 位 JVM 下每个 `Node` 对象头 16B + item 引用 4B + next 4B + prev 4B = 24B（可能对齐到 24B），即存储 4B 的引用要付 24B 代价；
3. **GC 压力**：节点多 → 标记阶段对象图遍历多 → 年轻代复制/标记成本上升。

> 实测经验：`ArrayList` 顺序遍历通常比 `LinkedList` 快 **2~10 倍**，即使在「随机插入」场景，元素量 < 10 万时 ArrayList 的搬移成本也常被缓存优势抵消。这是「**LinkedList 几乎不用**」的根本原因。

## 2.3 哈希表：集合框架最核心的物理结构

### 2.3.1 从「取模」到「位运算」

哈希表的核心两步：

```
index = hash(key) % tableLength          ← 理论
index = hash(key) & (tableLength - 1)    ← 工程（要求 tableLength 是 2 的幂）
```

**为什么等价？** 设 `n = 2^k`，则 `n - 1` 的二进制是 `0b00...0111...1`（低 k 位全 1，高位全 0）。与运算 `h & (n-1)` 恰好保留 `h` 的低 k 位，等价于 `h % 2^k`；而 `%` 在 JVM 上是 `idiv` 指令（约 20+ 周期），`&` 是 1 周期。

**推论（面试必答）**：只要容量是 2 的幂，索引计算就退化为「取 hash 的低 k 位」。这也意味着 **hash 的低位必须足够随机**，否则低位相同的 key 会全撞在一个桶里 —— 这正是 HashMap 扰动函数存在的原因（见 4.2）。

### 2.3.2 冲突解决的两大流派

| 流派 | 做法 | 代表 | 优缺点 |
|---|---|---|---|
| **链地址法（拉链 / separate chaining）** | 每个桶挂一个链表（JDK8 再升级为红黑树） | HashMap、ConcurrentHashMap、Hashtable | 实现简单、删除方便、装载因子可 > 1；指针跳转缓存不友好 |
| **开放定址法（open addressing）** | 冲突了就按规则找下一个空槽（线性探测 / 二次探测 / 双重散列） | ThreadLocalMap、Python dict、Redis dict（部分）、C++ unordered_map（部分实现） | 无指针、缓存友好；删除要「墓碑标记」、装载因子必须 < 1、易聚集（clustering） |

```
拉链法：
bucket[3] → [k1,v1] → [k2,v2] → null

线性探测（开放定址）：
bucket[3]=k1, bucket[4]=k2(冲突后顺延), bucket[5]=...
删除 k1 后不能直接置 null，要放"墓碑"，否则 k2 的查找链会断
```

> 💡 **延伸：ThreadLocalMap 为什么用开放定址？**
> 因为 ThreadLocalMap 通常只有几个 Entry，用开放定址可以避免每个 Entry 再挂 Node 对象（省内存），且 `Entry` 用 `WeakReference<ThreadLocal>` 做 key 便于 GC 回收；代价是必须处理「脏 Entry」的探测清理（`expungeStaleEntry`）。

### 2.3.3 装载因子（load factor）的本质

```
loadFactor = size / capacity
```

- 太小 → 空间浪费、扩容频繁；
- 太大 → 冲突率指数上升（拉链变长），查找退化；
- **0.75 是「空间 vs 时间」的经验最优点**：在泊松分布假设下，平均探测次数约 2~3 次即可命中；数学上 \(\log_2\) 哈希表的期望链长在 0.75 附近仍有良好常数。

## 2.4 红黑树（TreeMap / HashMap 树化桶）

### 2.4.1 五条性质（必须能背）

1. 每个节点是**红**或**黑**；
2. 根节点是**黑**；
3. 每个叶子（NIL 空节点）是**黑**；
4. **红节点的两个子节点必须是黑**（即不能有连续两个红）；
5. 从任一节点到其所有叶子的路径上，**黑节点数量相同**（黑高相等）。

**由性质 4+5 可推出**：最长路径 ≤ 2 × 最短路径，因此树高 ≤ `2*log2(n+1)`，所有操作 O(log n)。

### 2.4.2 插入后的修复：三种情况（面试加分项）

新插入节点**一律染红**（染红只可能破坏性质 4，代价最小）。

设 `z` 为新节点，`p` 为父，`g` 为祖父，`u` 为叔叔：

| 情况 | 条件 | 操作 |
|---|---|---|
| **Case 1** | 叔叔 `u` 是红 | `p`、`u` 染黑，`g` 染红，`z = g` **向上继续修复**（可能一路上溯到根，然后把根染黑，树高 +1） |
| **Case 2** | 叔叔黑，且 `z` 是「内侧」节点（LR / RL） | 对 `p` 旋转，转成 Case 3（**不改变颜色，只换形**） |
| **Case 3** | 叔叔黑，且 `z` 是「外侧」节点（LL / RR） | `p` 染黑、`g` 染红，对 `g` 旋转；**结束** |

```
Case 3 (LL)：                 Case 2 (LR) → 先左旋 p，变成 LL：
      g(黑)                          g(黑)
     /                              /
   p(红)                          p(红)
   /                              /
 z(红)                          z(红)
   ↓ 右旋 g + 变色
      p(黑)
     /     \
  z(红)    g(红)
```

> ⚠️ **易错点**：Case 1 是**唯一会向上传播**的情况，也是插入修复复杂度的来源（最坏 O(log n) 次变色 + 至多 2 次旋转）。删除修复比插入复杂（4 种情况 + 「双黑」上溯），面试一般只要求讲清「删除黑色节点会破坏黑高，需要兄弟节点借/旋/变色补偿」。

### 2.4.3 为什么用红黑树而不是 AVL

| 维度 | 红黑树 | AVL |
|---|---|---|
| 平衡严格度 | 近似平衡（黑高相等） | 严格平衡（左右高差 ≤ 1） |
| 查找 | O(log n)，常数略大 | O(log n)，更快（树更矮） |
| **插入/删除旋转次数** | **≤ 3 次（插入）/ ≤ 3 次（删除）** | O(log n) 次，可能一路旋转到根 |
| 适用场景 | **读写混合、写多**（HashMap 桶、TreeMap、Linux CFS、epoll） | 读多写少（数据库索引内存结构、需要极致查找） |

> 💡 **延伸：HashMap 的桶为什么树化后仍是「红黑树 + 链表」双结构？**
> `TreeNode` 继承 `LinkedHashMap.Entry` 继承 `HashMap.Node`，保留了 `next` 指针。原因：① 树化/退化时可以在树上做链表操作（`split` 扩容时按高低位拆成两条链）；② 删除时若节点数降到 6，可以快速 `untreeify` 回链表；③ `TreeNode` 还额外维护 `prev`，方便删除时找前驱（这是 `LinkedHashMap` 复用的结果）。

## 2.5 跳表（SkipList）

### 2.5.1 结构

```
Level 3: head ─────────────────────────────→ 30 ─────────→ null
Level 2: head ────────→ 10 ────────────────→ 30 ─────────→ null
Level 1: head ─→ 5 ───→ 10 ───→ 20 ────────→ 30 ─→ 40 ───→ null
Level 0: head → 5 → 8 → 10 → 15 → 20 → 25 → 30 → 35 → 40 → null
                 （每层都是一个有序链表，上层是下层的"抽样索引"）
```

查找 `25`：Level3 直接跳到 30 发现过了 → 下降 Level2 → 到 10 → Level1 到 20 → Level0 到 25。**期望 O(log n)**。

### 2.5.2 为什么跳表比红黑树更适合「并发」

| 维度 | 跳表 | 红黑树 |
|---|---|---|
| 插入/删除影响范围 | **局部**（只改前驱后继指针） | **全局**（旋转可能波及根） |
| 无锁化（CAS） | **容易**（每层用 CAS 改 next） | **极难**（旋转需要原子地改多个指针） |
| 范围查询 | **天然支持**（第 0 层就是有序链表） | 需要中序遍历 + 栈 |
| 实现复杂度 | 低 | 高 |
| 内存 | 每节点平均 1/(1-p) 个指针（p=0.25 时约 1.33 层） | 每节点固定 3 指针 + 颜色位 |

这就是 **Redis ZSet 用跳表、ConcurrentSkipListMap 用跳表、LevelDB/RocksDB MemTable 用跳表**，而 **TreeMap 用红黑树**的原因。

> ⚠️ **易错点**：跳表的层数是**随机**的（`ThreadLocalRandom` + `while (rnd.nextInt() & 1) level++`，上限 31/32），不是计算出来的。这保证了「期望平衡」而无需旋转。

## 2.6 堆（PriorityQueue）

**完全二叉树 + 数组存储**（因为完全二叉树可以用数组无空洞表示，父子关系靠下标算）：

```
parent(i) = (i - 1) >>> 1
left(i)   = (i << 1) + 1
right(i)  = (i << 1) + 2
```

- **小顶堆**：`queue[i] <= queue[left] && queue[i] <= queue[right]`；
- `siftUp`（上浮）：新元素放末尾，与父比较，比父小就交换，O(log n)；
- `siftDown`（下沉）：取末尾元素放堆顶，与「较小的孩子」比较并交换，O(log n)；
- **建堆是 O(n) 而不是 O(n log n)**：从 `n/2 - 1` 开始下沉，越靠下的节点越少且下沉高度越低，求和 \(\sum_{h} \frac{n}{2^{h+1}} \cdot h = O(n)\)。

## 2.7 集合框架里用到的位运算技巧汇总

| 技巧 | 代码 | 用途 |
|---|---|---|
| 判断 2 的幂 | `(n & (n-1)) == 0` | HashMap 容量校验 |
| 求 ≥n 的最小 2 的幂 | `tableSizeFor` 的五次 `n \|= n >>> 1/2/4/8/16` | HashMap 构造 |
| 取模代替 | `h & (n-1)` | 索引定位 |
| 高低位混合 | `h ^ (h >>> 16)` | 扰动函数 |
| 判断扩容后落在高位还是低位 | `(e.hash & oldCap) == 0` | JDK8 resize 拆链 |
| 环形数组移动 | `(i + 1) & (len - 1)`、`(i - 1) & (len - 1)` | ArrayDeque |
| 计算父/子下标 | `(i-1)>>>1`、`(i<<1)+1`、`(i<<1)+2` | PriorityQueue |
| 容量翻倍 | `oldCap << 1` | 多处 |
| 1.5 倍扩容 | `oldCap + (oldCap >> 1)` | ArrayList |

---

# 第 3 章 ArrayList、LinkedList、Vector、CopyOnWriteArrayList

## 3.1 ArrayList：90% 业务场景的默认选择

### 3.1.1 核心字段（JDK 8 源码）

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    private static final int DEFAULT_CAPACITY = 10;

    // 两个"空数组"哨兵，用于区分"显式指定容量0"和"默认构造"
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 真正存数据的数组。transient：不参与默认序列化
    transient Object[] elementData;

    private int size;   // 实际元素个数（≠ elementData.length）

    protected transient int modCount;  // 继承自 AbstractList，结构性修改计数
}
```

**两个空数组为什么要分开？** 这是 JDK 8 的一个经典细节：

| 构造方式 | elementData | 首次 add 时容量 |
|---|---|---|
| `new ArrayList<>()` | `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` | **10**（懒加载） |
| `new ArrayList<>(0)` | `EMPTY_ELEMENTDATA` | **1**（每次扩容 1.5 倍，从 0 开始 → 1） |
| `new ArrayList<>(100)` | `new Object[100]` | 100（不扩容） |

如果只有一个 `EMPTY_ELEMENTDATA`，就无法区分「用户要 0 容量」与「用户没指定容量」，首次扩容会错误地跳到 10 或错误地停在 1。

### 3.1.2 构造方法全解

```java
// 1. 默认构造：懒加载，数组先指向共享空数组
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 2. 指定初始容量
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}

// 3. 从集合构造
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;              // 同为 ArrayList 且 toArray 返回 Object[] 时直接复用
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);  // 防御性拷贝，修正数组类型
        }
    } else {
        elementData = EMPTY_ELEMENTDATA;
    }
}
```

> ⚠️ **易错点**：`new ArrayList<>(c)` 在 `c` 是 `ArrayList` 时会**复用 `c.toArray()` 返回的数组**，看起来像浅拷贝共享，但实际上 `ArrayList.toArray()` 每次都 `Arrays.copyOf` 新建数组，所以是安全的。但如果你传的是 `Arrays.asList()` 的集合，`toArray()` 可能返回内部数组，JDK 用 `c.getClass() == ArrayList.class` 这个精确判断规避了风险（`Arrays$ArrayList` 的 class 不同）。

### 3.1.3 add 与扩容全流程（**最高频考点**）

```java
public boolean add(E e) {
    modCount++;                                  // ① 结构性修改计数 +1
    add(e, elementData, size);                   // ② JDK17 拆出的私有方法
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)                 // ③ 满了才扩容
        elementData = grow();
    elementData[s] = e;                          // ④ 直接放在 size 位置
    size = s + 1;
}

private Object[] grow() {
    return grow(size + 1);
}

private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
                minCapacity - oldCapacity,   /* minimum growth */
                oldCapacity >> 1);           /* preferred growth：1.5 倍 */
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        // 默认构造的首次 add：直接给 DEFAULT_CAPACITY=10
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}
```

`ArraysSupport.newLength(oldLength, minGrowth, prefGrowth)` 的逻辑（JDK11+ 抽出）：

```
newLength = oldLength + max(minGrowth, prefGrowth)
若 newLength 溢出或 > Integer.MAX_VALUE - 8 → 退化为 oldLength + minGrowth
若仍溢出 → 直接 hugeCapacity（MAX_ARRAY_SIZE 或 Integer.MAX_VALUE）→ 否则 OutOfMemoryError
```

**JDK 6/7 的实现（老面试题答案）**：

```java
int newCapacity = oldCapacity + (oldCapacity >> 1);   // 1.5 倍
if (newCapacity - minCapacity < 0) newCapacity = minCapacity;
if (newCapacity - MAX_ARRAY_SIZE > 0) newCapacity = hugeCapacity(minCapacity);
elementData = Arrays.copyOf(elementData, newCapacity);
```

**扩容过程图解（默认构造）**：

```
new ArrayList<>()           elementData → 共享空数组 (length 0), size 0
add(1)                      → new Object[10]                (length 10, size 1)
add ... add(10)             (length 10, size 10)
add(11)                     → 10 + (10>>1) = 15             (length 15, size 11)
add(16)                     → 15 + 7 = 22
...                         总量增长曲线：10 → 15 → 22 → 33 → 49 → 73 → 109 ...
```

**均摊复杂度为什么是 O(1)？**
设从 0 增长到 n，总拷贝次数 ≈ n + n/1.5 + n/1.5² + ... ≈ 3n（几何级数，公比 2/3）。总代价 O(n)，分摊到 n 次 add 即 **O(1) 均摊**。

### 3.1.4 为什么是 1.5 倍而不是 2 倍？（**阿里/美团高频**）

这是一个**权衡题**，标准答案有四层：

1. **空间**：2 倍增长时，`new = 2*old`，而「已用空间」为 `old`（若刚好满），意味着**至少浪费 50% 的容量**；1.5 倍时浪费约 33%。对大 ArrayList（如百万级）差异是几十 MB。
2. **可复用性（关键）**：1.5 倍增长后释放的旧数组大小，恰好能被「后续更小的扩容请求」复用。经典场景：累计 `n` 次扩容后，总分配内存 ≈ `3n`，而其中「历史释放的块」总和 ≈ `n`，能被下一次较大的请求复用（HotSpot 的 TLAB/堆分配器对这种「渐进式增长 + 及时释放」的碎片更友好）。2 倍增长会导致「旧块永远不够用」（因为新需求总是 2 倍，旧块只有 1 倍），全部变为垃圾，且内存占用呈锯齿状。
3. **避免过早 OOM**：2 倍增长对内存峰值不友好（一次扩容临时需要「旧 + 新」= 3 倍空间）。
4. **与 HashMap 的区别**：HashMap 必须是 2 的幂（为了位运算取模），所以只能 2 倍；ArrayList 没有这个约束，可以自由选 1.5。

> 💡 **延伸：为什么 ArrayList 扩容上限是 `Integer.MAX_VALUE - 8`？**
> 数组对象头需要 8~16 字节，且某些 JVM 实现（尤其是 HotSpot 的 `arrayOopDesc`）需要预留 header 空间，`MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8` 是为了「预留 header + 避免某些 VM 的 `OutOfMemoryError: Requested array size exceeds VM limit`」。真正到 `Integer.MAX_VALUE` 时会直接抛 OOM。

### 3.1.5 add(int index, E e) 与 remove —— 为什么是 O(n)

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);
    modCount++;
    final int s = size;
    Object[] elementData;
    if ((elementData = this.elementData).length == s)
        elementData = grow();
    // 把 index 及之后的元素整体后移一格
    System.arraycopy(elementData, index, elementData, index + 1, s - index);
    elementData[index] = element;
    size = s + 1;
}

public E remove(int index) {
    Objects.checkIndex(index, size);
    final Object[] es = elementData;
    E oldValue = (E) es[index];
    fastRemove(es, index);
    return oldValue;
}

private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize = size - 1;
    if (newSize > i)
        System.arraycopy(es, i + 1, es, i, newSize - i);  // 前移
    es[newSize] = null;   // ← 关键：置 null 帮助 GC
    size = newSize;
}
```

> ⚠️ **易错点：`es[newSize] = null` 这行的意义**
> 如果不置 null，被删元素仍被数组强引用，**直到该槽被下次 add 覆盖前都不会被 GC**。在「大对象 + 长生命周期 ArrayList」场景会造成**内存泄漏**。这也是面试官爱问「ArrayList 删除元素后会立刻释放内存吗」的答案：**引用置 null，但对象本身要等 GC**。

**删除的均摊分析**：`remove(0)` 要搬 n-1 个元素，`remove(n-1)` 是 O(1)，平均搬 n/2 个 → **O(n)**。所以「ArrayList 不适合频繁头部删除」；需要频繁头尾操作请用 `ArrayDeque`。

### 3.1.6 序列化：为什么 elementData 是 transient

```java
private void writeObject(java.io.ObjectOutputStream s) throws IOException {
    int expectedModCount = modCount;
    s.defaultWriteObject();          // 写 size、capacity 之外的普通字段
    s.writeInt(size);                // 只写实际元素个数
    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);
    }
    if (modCount != expectedModCount) throw new ConcurrentModificationException();
}

private void readObject(java.io.ObjectInputStream s) ... {
    s.defaultReadObject();
    s.readInt();
    if (size > 0) {
        // 反序列化时直接按 size 分配，而不是按 capacity
        elementData = new Object[size];
        ...
    }
}
```

**核心原因**：`capacity` 可能远大于 `size`（例如 `new ArrayList<>(1000000)` 只放 1 个元素），若序列化 `elementData` 会把**大量 null 也写进流**，体积爆炸。自定义 `writeObject` 只写 `size` 个真实元素，**序列化体积 ∝ 元素数，而不是容量**。这是「自定义序列化」的经典范例。

> 💡 **延伸：`modCount` 检查在 writeObject 里的作用**
> 序列化过程中若被其他线程修改，会写出「不一致快照」，所以 JDK 在末尾校验 `modCount`。注意：**这是「尽力而为」，不是同步保证**（`writeObject` 不是 synchronized）。

### 3.1.7 迭代器与 fail-fast（**腾讯/美团必考**）

```java
private class Itr implements Iterator<E> {
    int cursor;        // 下一个返回的元素下标
    int lastRet = -1;  // 上一个返回的元素下标
    int expectedModCount = modCount;   // ← 快照

    public boolean hasNext() { return cursor != size; }

    public E next() {
        checkForComodification();      // 关键校验
        int i = cursor;
        if (i >= size) throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

    public void remove() {
        if (lastRet < 0) throw new IllegalStateException();
        checkForComodification();
        try {
            ArrayList.this.remove(lastRet);    // 会 modCount++
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;       // ← 重新同步，所以不抛异常
        } catch (IndexOutOfBoundsException ex) { throw new ConcurrentModificationException(); }
    }
}
```

**经典错误写法与正确写法**：

```java
// ❌ 错误：for-each（本质是 Iterator）里调 list.remove()
for (String s : list) {
    if (s.startsWith("a")) list.remove(s);   // next() 时抛 CME
}

// ✅ 正确 1：显式 Iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("a")) it.remove();
}

// ✅ 正确 2：removeIf（JDK8，内部用位图批量删除，比 iterator.remove 更快）
list.removeIf(s -> s.startsWith("a"));

// ✅ 正确 3：倒序遍历下标删除（不需要迭代器）
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).startsWith("a")) list.remove(i);
}
```

> 💡 **延伸：为什么 `removeIf` 更快？**
> JDK8 的 `ArrayList.removeIf` 用「**双指针 + 位图（BitSet）**」：一次遍历标记要删的位置，再一次遍历批量前移，避免每个元素删除都 `System.arraycopy` 一次（O(n²) → O(n)）。源码在 `ArrayList.removeIf` / `ArrayList.batchRemove`。

### 3.1.8 ArrayList 的其他高频细节

| 方法 | 细节 | 考点 |
|---|---|---|
| `subList(from, to)` | 返回 `SubList` **视图**，与原 list 共享 `elementData`；修改互相可见；原 list 结构性修改后访问 subList 会 CME | 视图 vs 拷贝 |
| `trimToSize()` | `elementData = Arrays.copyOf(elementData, size)`，**会 `modCount++`** | 会触发 CME |
| `clear()` | 遍历置 null + `size = 0`，**不缩容** | 内存不释放 |
| `ensureCapacity(n)` | 手动扩容，避免多次拷贝 | 性能优化 |
| `indexOf(o)` | 支持 null（`o == null ? for(e==null) : o.equals(e)`） | null 安全 |
| `contains(o)` | 线性扫描 O(n) | 不是哈希 |
| `toArray()` | 每次 `Arrays.copyOf`，返回新数组（安全） | 与 `Arrays.asList` 相反 |
| `clone()` | 浅拷贝，`elementData` 复制，元素引用共享 | 深浅拷贝 |
| `RandomAccess` 标记接口 | 告诉算法「随机访问快」，`Collections.binarySearch` 会据此选「二分」还是「迭代器遍历」 | 标记接口用途 |

## 3.2 LinkedList：双向链表 + Deque

### 3.2.1 结构

```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(Node<E> prev, E element, Node<E> next) { ... }
    }
}
```

```
LinkedList 内部（非循环双链表，带 first/last 指针）：

first → [null ← prev|A|next →] ⇄ [B] ⇄ [C] ← last
```

### 3.2.2 为什么 get(i) 是 O(n)：`node(index)` 的「折半优化」

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {          // 前半段：从头找
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x;
    } else {                            // 后半段：从尾找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x;
    }
}
```

**注意**：这是「常数优化」，复杂度仍是 O(n)。**因此 `for (int i=0; i<list.size(); i++) list.get(i)` 是 O(n²)**，这是 LinkedList 最经典的性能陷阱，必须用 foreach / iterator。

### 3.2.3 关键操作源码

```java
// 头插
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null) last = newNode;
    else f.prev = newNode;
    size++; modCount++;
}

// 尾插（add 默认调它）
void linkLast(E e) { ... }

// 在非空节点 succ 前插入
void linkBefore(E e, Node<E> succ) { ... }

// 删除节点（O(1)，因为能拿到 prev）
E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    if (prev == null) first = next; else { prev.next = next; x.prev = null; }
    if (next == null) last = prev;  else { next.prev = prev; x.next = null; }
    x.item = null; size--; modCount++;
    return element;
}
```

### 3.2.4 LinkedList 作为 Deque 的方法语义

| Deque 方法 | 失败时 | 特点 |
|---|---|---|
| `addFirst/addLast` | 抛异常 | 插入 |
| `offerFirst/offerLast` | 返回 false | 插入（容量受限时用） |
| `removeFirst/removeLast` | 抛 `NoSuchElementException` | 删除 |
| `pollFirst/pollLast` | 返回 null | 删除 |
| `getFirst/getLast` | 抛异常 | 查看 |
| `peekFirst/peekLast` | 返回 null | 查看 |
| `push` = `addFirst` / `pop` = `removeFirst` | | 栈语义（注意方向） |

### 3.2.5 ArrayList vs LinkedList 的真实对比（**必背表**）

| 维度 | ArrayList | LinkedList |
|---|---|---|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 `get(i)` | **O(1)** | O(n) |
| 头部插入/删除 | O(n)（搬移） | **O(1)** |
| 尾部插入 | 均摊 O(1) | O(1) |
| 中间插入 | O(n) 搬移 | O(1) 改指针（**但要先 O(n) 找到位置**）→ 实际 O(n) |
| 内存 | 紧凑（无额外指针），可能有容量冗余 | 每元素多 2 个引用（24B/节点），无冗余 |
| 缓存友好度 | **高**（连续内存） | 低（指针跳转） |
| 迭代性能 | 快 | 慢（cache miss） |
| 实现接口 | List, RandomAccess | List, Deque（无 RandomAccess） |
| 适用 | **绝大多数场景（默认选它）** | 频繁头尾增删、需要 Deque 语义 |

> 🎯 **面试金句**：「LinkedList 只在『已知节点位置』的 O(1) 插入删除上占优，而 Java 的 API 不暴露节点引用（`Node` 是 private），所以外部永远要先 O(n) 定位，综合性能几乎总是劣于 ArrayList。Linux 内核的链表之所以高效，是因为它用的是『侵入式链表』，节点内嵌在业务结构里，拿到业务对象就等于拿到节点。」

## 3.3 Vector 与 Stack：为什么被淘汰

```java
public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
    protected Object[] elementData;
    protected int elementCount;
    protected int capacityIncrement;   // 扩容增量，默认 0 表示翻倍

    public synchronized boolean add(E e) { ... }   // 每个方法都 synchronized
    public synchronized E get(int index) { ... }
}
```

三个问题：
1. **全表锁**：`synchronized` 加在方法上，锁的是 `this`，所有读写串行，并发性能极差；
2. **扩容 2 倍**：`newCapacity = oldCapacity + (capacityIncrement > 0 ? capacityIncrement : oldCapacity)`，浪费空间；
3. **`Stack` 继承 `Vector`**：把「栈」语义污染成「List + 同步」，还能 `add(index, e)` 任意插入，破坏 LIFO 封装。

**替代方案**：
- 需要线程安全 List → `CopyOnWriteArrayList`（读多写少）或 `Collections.synchronizedList` + 外部同步；
- 需要栈 → **`ArrayDeque`**（`push/pop/peek`），非线程安全但快一个数量级；需要并发栈 → `ConcurrentLinkedDeque`。

> ⚠️ **易错点**：JDK 官方文档明确写「`Stack` 类已过时，建议用 `Deque` 实现栈」。`ArrayDeque.push` 是 `addFirst`，`pop` 是 `removeFirst`，**方向与 Stack 一致**（栈顶在数组头部）。

## 3.4 CopyOnWriteArrayList：写时复制

### 3.4.1 核心源码（思想极简）

```java
public class CopyOnWriteArrayList<E> implements List<E>, RandomAccess, Cloneable, Serializable {
    private transient volatile Object[] array;      // volatile：保证可见性

    final Object[] getArray() { return array; }
    final void setArray(Object[] a) { array = a; }

    public boolean add(E e) {
        synchronized (lock) {                        // 写操作串行
            Object[] es = getArray();
            int len = es.length;
            es = Arrays.copyOf(es, len + 1);         // ① 复制整个数组（拷贝！）
            es[len] = e;                             // ② 在副本末尾追加
            setArray(es);                            // ③ 原子替换引用
            return true;
        }
    }

    public E get(int index) { return elementAt(getArray(), index); }  // 读：无锁、无拷贝

    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);   // 迭代器持有「快照引用」
    }
}
```

### 3.4.2 读为什么完全无锁

因为 `array` 是 `volatile` 的引用，读线程要么看到旧数组、要么看到新数组，**永远不会看到「半成品」**（新数组在副本上构建完才发布）。这是**「不可变对象 + volatile 发布」**的经典组合，本质是**写时复制 + 引用原子替换**。

### 3.4.3 优缺点与适用场景（**必背**）

| 优点 | 缺点 |
|---|---|
| 读完全无锁，读性能极高 | **写开销巨大**：每次 add 都 O(n) 拷贝 + 内存翻倍峰值 |
| 迭代器是**快照**，绝不抛 CME | **数据弱一致**：读可能读到旧数据（迭代期间的新增不可见） |
| 实现简单，无死锁风险 | 内存占用高（同一时刻两份数组） |
| 适合「读多写极少」 | 不适合频繁写 / 大集合 |

**典型场景**：事件监听器列表（`ListenerList`）、配置白名单、`CopyOnWriteArraySet`、Spring 的 `ApplicationListener` 集合、Netty 的 `ChannelHandler` 链。

> ⚠️ **易错点**：`CopyOnWriteArrayList` 的 `iterator()` **不支持 `remove()`**（抛 `UnsupportedOperationException`），因为快照不能反向修改。

> 💡 **延伸：为什么 COW 集合不能「元素本身不可变」？**
> COW 只保证「数组引用」的替换安全，**不保证元素对象的可变性**。如果元素是可变对象（如 `User`），线程 A 读到的 `User` 可能正被线程 B 修改字段，仍然有可见性问题。正确用法是「COW + 元素不可变（final 字段 / 值对象）」。

## 3.5 本章速查表

| 集合 | 底层 | 扩容 | 线程安全 | 迭代器 | 允许 null | 典型场景 |
|---|---|---|---|---|---|---|
| ArrayList | Object[] | 1.5 倍 | ❌ | fail-fast | ✅ | 默认 List |
| LinkedList | 双链表 | 无 | ❌ | fail-fast | ✅ | Deque、频繁头尾 |
| Vector | Object[] | 2 倍 | ✅（全表锁） | fail-fast | ✅ | 遗留代码 |
| Stack | Object[] | 2 倍 | ✅（全表锁） | fail-fast | ✅ | 遗留，用 ArrayDeque 替代 |
| CopyOnWriteArrayList | volatile Object[] | 每次 +1 拷贝 | ✅（写锁） | **快照，无 CME** | ✅ | 读多写极少 |

---

# 第 4 章 HashMap 全链路拆解

> 这一章是面试的「绝对主战场」。**HashMap 的每一行关键常量、每一次扩容、每一个位运算，都可能被追问三层。**

## 4.1 结构演进：JDK7 vs JDK8

| 维度 | JDK 7 | JDK 8 |
|---|---|---|
| 数据结构 | 数组 + **单向链表** | 数组 + 链表 + **红黑树** |
| 节点类 | `Entry<K,V>{hash,key,value,next}` | `Node<K,V>{hash,key,value,next}` + `TreeNode` |
| 插入方式 | **头插法**（新节点插到链头） | **尾插法**（遍历到链尾再插） |
| 扩容时节点顺序 | **会反转**（头插导致） | **保持原相对顺序**（高低位拆成两条链） |
| 索引计算 | `h & (length-1)`，hash 需再散列 | 同左，hash 扰动只做一次 |
| 扩容触发条件 | `size >= threshold && table[i] != null` | `++size > threshold` |
| 并发问题 | **可能形成环形链表 → CPU 100%** | 可能丢数据 / size 不准，但**不会死循环** |
| 树化 | 无 | 链表长度 ≥ 8 且容量 ≥ 64 → 红黑树 |

```
JDK 8 HashMap 内存结构：

table (Node[] 数组，长度 2^n)
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ ...
└─┬──┴────┴─┬──┴────┴────┴────┴────┴────┘
  │         │
  ▼         ▼
[Node]     [TreeNode]（红黑树，桶内 ≥8 且容量 ≥64）
  │
  ▼
[Node] → [Node] → null     （桶内 <8 时是链表）
```

## 4.2 核心常量与字段（背下来）

```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   // 16
    static final int MAXIMUM_CAPACITY = 1 << 30;          // 2^30
    static final float DEFAULT_LOAD_FACTOR = 0.75f;       // 负载因子
    static final int TREEIFY_THRESHOLD = 8;               // 链表→树 阈值
    static final int UNTREEIFY_THRESHOLD = 6;             // 树→链表 阈值
    static final int MIN_TREEIFY_CAPACITY = 64;           // 树化的最小表容量

    transient Node<K,V>[] table;      // 桶数组，长度必为 2 的幂
    transient Set<Map.Entry<K,V>> entrySet;
    transient int size;               // K-V 对个数
    transient int modCount;           // 结构性修改次数（fail-fast）
    int threshold;                    // 扩容阈值 = capacity * loadFactor
    final float loadFactor;           // 负载因子（构造后不可变）
}
```

**注意 `threshold` 的双重身份**（易错点）：

- 正常状态下：`threshold = capacity * loadFactor`，表示「size 超过它就扩容」；
- **在 `table` 尚未初始化时**：`threshold` 被临时用来存「初始容量」。构造方法 `HashMap(int initialCapacity, float loadFactor)` 里：

```java
this.loadFactor = loadFactor;
this.threshold = tableSizeFor(initialCapacity);   // 此时 threshold 存的是 capacity！
```

之后第一次 `putVal` 调 `resize()` 时，`newCap = threshold`，再重新计算真正的 `threshold = newCap * loadFactor`。这是 JDK8 的「懒加载 + 复用字段」技巧。

## 4.3 构造函数

```java
// 1. 无参：一切默认，table 懒加载（第一次 put 才 new）
public HashMap() { this.loadFactor = DEFAULT_LOAD_FACTOR; }

// 2. 指定容量
public HashMap(int initialCapacity) { this(initialCapacity, DEFAULT_LOAD_FACTOR); }

// 3. 指定容量 + 负载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0) throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY) initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);   // ← 见上
}

// 4. 从 Map 构造（JDK8 加入）
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

> ⚠️ **易错点：`new HashMap<>(100)` 的容量是多少？**
> **不是 100，而是 128**。`tableSizeFor(100)` 返回 ≥100 的最小 2 的幂 = 128。第一次 put 时 `newCap = 128`，`threshold = 128 * 0.75 = 96`。
> 所以「初始容量」应设为 `预期元素数 / 0.75 + 1`，例如要存 100 个元素，应 `new HashMap<>(128 + 1)` 或 `new HashMap<>(136)`（Guava 的 `Maps.capacity(100)` = 133 → tableSizeFor = 256? 注意 133 → 256 会浪费，实际 Guava 是 `(int)(expectedSize / 0.75F) + 1 = 134` → 256。所以更精确做法是直接算 2 的幂）。

**容量计算表（背下来，面试常让你算）**：

| 预期元素数 n | 推荐 initialCapacity | 实际 table 长度 | 阈值 |
|---|---|---|---|
| 10 | 16 | 16 | 12 |
| 50 | 64 | 64 | 48 |
| 100 | 128 | 128 | 96 |
| 1000 | 2048 | 2048 | 1536 |

（因为 `n/0.75 = 1.33n`，向上取 2 的幂）

## 4.4 hash 扰动函数（**美团/字节高频**）

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**为什么要 `h ^ (h >>> 16)`？**

1. `hashCode()` 返回 int（32 位），但 HashMap 的索引只用 `hash & (n-1)`，**只用了低位**（n 通常 ≤ 2^16 时只用到低 16 位）；
2. 很多类的 `hashCode` 实现「低位区分度差、高位区分度好」（例如 `Long.hashCode` 是 `(int)(value ^ (value >>> 32))`，某些字符串哈希的高位才随机）；
3. 把**高 16 位异或到低 16 位**，让高位信息参与索引计算，同时不损失低位信息（异或是可逆的，不会碰撞两个原本不同的 hash）。

```
h        = 0b 1010_1100_0011_1001_0111_1110_1010_0101
h >>> 16 = 0b 0000_0000_0000_0000_1010_1100_0011_1001
h ^ ...  = 0b 1010_1100_0011_1001_1101_0010_1001_1000
                             └──── 低16位被高位"污染"，随机性提升 ────┘
```

> 💡 **延伸：为什么是「异或」而不是「与/或」？**
> - `&` 会让结果偏向 0（高位 0 时低位保持原样，信息丢失但不扩散）；
> - `|` 会让结果偏向 1；
> - `^` 保留了两边的信息熵，是唯一「不偏向」的位运算，且 `h ^ (h>>>16)` 是双射（可逆），不会把不同的 h 映射成同一个。

> ⚠️ **易错点：为什么不用 `hashCode() % length`？**
> 因为 `%` 对负数会得到负数（`-5 % 3 = -2`），需要 `Math.abs` 或额外处理；而且 `%` 是 `idiv`，慢；且 `%` 对「低位相同的 hash」同样无能为力。位运算 + 2 的幂是性能与分布的最优解。

**`tableSizeFor`（求 ≥cap 的最小 2 的幂）**：

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;                 // 减 1：处理 cap 本身就是 2 的幂的情况
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

原理：`n` 的二进制最高位 1 之后全部「传染」成 1（1→2→4→8→16 的位移覆盖 32 位），再 +1 就进位到下一个 2 的幂。

```
cap = 100 → n = 99 = 0b1100011
n |= n>>>1  → 0b1110111
n |= n>>>2  → 0b1111111
... → 0b1111111 (127)
return 128
```

## 4.5 为什么容量必须是 2 的幂（**必答三层**）

1. **索引计算可以用 `&` 代替 `%`**：`hash & (n-1)` 仅当 `n = 2^k` 时等价于 `hash % n`。位运算比取模快一个数量级（HashMap 的 get 是最高频操作，这个优化收益巨大）。
2. **保证元素分布均匀**：`n-1` 的低位全 1，使得 hash 的每一位都能参与索引，不会出现「某些桶永远空着」的情况。若 `n = 15`（非 2 的幂），`n-1 = 14 = 0b1110`，最低位永远是 0 → **所有奇数 hash 的 key 全挤在偶数桶**，一半桶浪费，冲突率翻倍。
3. **扩容时可以「高低位拆分」而不重新 hash**：因为 `n` 每次翻倍，元素的新索引只有两种可能——「原索引」或「原索引 + oldCap」，只需判断 `(e.hash & oldCap) == 0`（见 4.7）。

```
n=16:  index = hash & 0b1111      （低 4 位）
n=32:  index = hash & 0b11111     （低 5 位）
→ 新索引 = 旧索引 或 旧索引+16，取决于低 5 位中的第 5 位（即 hash & 16）
```

## 4.6 putVal 全流程（**最高频考点，必须能默写**）

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    // ① 表为空 → 初始化（懒加载）
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // ② 目标桶为空 → 直接放新节点（CAS 无关，HashMap 非并发）
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    else {
        Node<K,V> e; K k;
        // ③ 桶首节点就是目标 key → 记录 e，准备覆盖 value
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

        // ④ 桶首是树节点 → 走红黑树插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        // ⑤ 桶是链表 → 遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);   // 尾插
                    if (binCount >= TREEIFY_THRESHOLD - 1)      // binCount >= 7
                        treeifyBin(tab, hash);                  // 尝试树化
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;                                       // 命中已有 key
                p = e;
            }
        }

        // ⑥ 命中已有 key → 覆盖 value，返回旧值（不增加 size）
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);       // LinkedHashMap 的回调（accessOrder 时移到尾）
            return oldValue;
        }
    }

    ++modCount;
    // ⑦ 容量超阈值 → 扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);    // LinkedHashMap 的回调（可 evict 最老元素 → LRU）
    return null;
}
```

**流程图**：

```
put(k, v)
  │
  ├─ hash = (k == null) ? 0 : h ^ (h>>>16)
  │
  ├─ table 为 null/空 ? ──是──→ resize() 初始化（默认 16）
  │
  ├─ i = (n-1) & hash
  │
  ├─ table[i] == null ? ──是──→ 新建 Node 放入，size++，检查扩容 ── 结束
  │
  └─ 否 → 桶首节点检查
        ├─ hash 相同 且 (== 或 equals) → 覆盖 value，返回 oldValue ── 结束
        ├─ 是 TreeNode → putTreeVal（红黑树插入 + 平衡）
        └─ 是链表 → 遍历到尾
              ├─ 找到相同 key → 覆盖 value ── 结束
              └─ 到尾仍没找到 → 尾插
                    ├─ 链长 ≥ 8 → treeifyBin
                    │     ├─ table.length < 64 → resize() 扩容（而不是树化）
                    │     └─ ≥ 64 → 链表转红黑树
                    └─ size++ > threshold → resize()
```

### 4.6.1 三个细节必须讲清

**(1) 为什么判断 `p.hash == hash` 再 `equals`？**
`hash` 比较是 `int` 比较（1 个指令），`equals` 是方法调用（可能很重）。先比 hash 可以过滤掉 99% 的不相等情况，这是「**短路优化**」。注意 `p.hash == hash` 已经是**扰动后**的 hash，不是原始 `hashCode`。

**(2) 为什么 `(k = p.key) == key || key.equals(k)` 先 `==`？**
引用相等（同一个对象）必然 `equals` 为 true（`equals` 契约要求自反），所以先 `==` 可以省一次方法调用；同时**避免了对 null key 调 `equals`**（null key 的 hash 是 0，会在 0 号桶里，`key.equals(k)` 会 NPE，所以有 `key != null` 判断）。

**(3) `treeifyBin` 的完整逻辑**

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)   // 64
        resize();                       // ← 关键：容量太小先扩容，不树化
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {                            // 先把链表节点全部转成 TreeNode，串成双向链
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null) hd = p; else { p.prev = tl; tl.next = p; }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);            // 再"建树"（不是逐个插入，而是按链表顺序 build）
    }
}
```

> ⚠️ **易错点**：`treeify` 不是「把节点一个个 `putTreeVal`」，而是**先按链表顺序构建一棵近似平衡的树，再调用 `balanceInsertion` 修正**。这是因为链表本身是「按 hash 排序」的（插入顺序 + 尾插），可以用「中位数选根」的方式快速建树（`TreeNode.treeify` 里用 `mid` 递归选根）。

## 4.7 resize 扩容全解（**最难的考点**）

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {         // 到顶了，不再扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;                 // 阈值也翻倍
    }
    else if (oldThr > 0)                          // 用了带容量构造，threshold 里存的是容量
        newCap = oldThr;
    else {                                        // 无参构造后的首次初始化
        newCap = DEFAULT_INITIAL_CAPACITY;        // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  // 12
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY) ? (int)ft : Integer.MAX_VALUE;
    }
    threshold = newThr;

    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;                 // 帮助 GC
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;         // 单节点：直接重算索引
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);   // 树拆分
                else {                             // ← 链表高低位拆分（JDK8 精华）
                    Node<K,V> loHead = null, loTail = null;   // 低位链
                    Node<K,V> hiHead = null, hiTail = null;   // 高位链
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {         // 判断「新增的那一位」
                            if (loTail == null) loHead = e; else loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null) hiHead = e; else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) { loTail.next = null; newTab[j] = loHead; }
                    if (hiTail != null) { hiTail.next = null; newTab[j + oldCap] = hiHead; }
                }
            }
        }
    }
    return newTab;
}
```

### 4.7.1 「高低位拆分」为什么成立（**面试官的杀手问**）

设 `oldCap = 16 = 0b10000`，`newCap = 32 = 0b100000`。

- 旧索引：`hash & 15`（低 4 位）
- 新索引：`hash & 31`（低 5 位）

新旧索引的差别**只在于第 5 位（即 `0b10000` = 16 = oldCap）**：

```
hash = ...b4 b3 b2 b1 b0   （旧索引取低 4 位 b3..b0）
新索引取低 5 位 b4..b0
→ 若 b4 == 0：新索引 = 旧索引
→ 若 b4 == 1：新索引 = 旧索引 + 16（即 + oldCap）
```

而 `b4` 就是 `hash & oldCap`（因为 `oldCap = 0b10000` 恰好只保留第 5 位）。所以**只需一次 `&` 判断，不需要重新算 `hash & (newCap-1)`**，也不需要重新 hash。

```
扩容前 (cap=16)：
bucket[5] → A(hash=…) → B → C

扩容后 (cap=32)：
bucket[5]  → A → C     （hash & 16 == 0 的留在原位）
bucket[21] → B         （hash & 16 != 0 的移到 5+16）
```

**这就是 JDK8 扩容比 JDK7 快、且「不会形成环」的根本原因**：JDK7 的 `transfer` 是「遍历旧桶，把每个节点**头插**到新桶」，头插会**反转链序**；JDK8 是「拆成两条链，保持原序，尾插」。反转本身不致命，致命的是**并发时两个线程同时 transfer 导致的环**（见 4.9）。

### 4.7.2 `TreeNode.split` 的树拆分

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    // 与链表拆分类似：按 (e.hash & bit) 分成 lo/hi 两条链
    // 若拆分后节点数 <= UNTREEIFY_THRESHOLD(6) → untreeify 回链表
    // 否则 → 用 treeify 重新建树
}
```
注意：**树的拆分同样保持相对顺序**，且当某条链 ≤ 6 个节点时退化为链表（避免「小树」维护成本）。

### 4.7.3 扩容的性能账

- 每次 resize 要**遍历整个旧表**（O(capacity)）+ 复制所有节点（O(size)）；
- 从 16 扩到 N，总代价 ≈ `16 + 32 + ... + N ≈ 2N`，均摊到每次 put 是 **O(1)**；
- 但**单次 resize 是 O(n) 的停顿**，对延迟敏感场景（如实时交易）要**预设容量**避免扩容。

## 4.8 treeify 的三个阈值：8 / 6 / 64（**必答**）

### 8 的来源：泊松分布

HashMap 源码注释（JDK8）原文：

> Because TreeNodes are about twice the size of regular nodes, we use them only when bins contain enough nodes... In usages with well-distributed user hashCodes, tree bins are rarely used. Ideally, under random hashCodes, the frequency of nodes in bins follows a Poisson distribution with a parameter of about 0.5 on average for the default resizing threshold of 0.75, although with a large variance because of resizing granularity. Ignoring variance, the expected occurrences of list size k are (exp(-0.5) * pow(0.5, k) / factorial(k)).
> 0: 0.60653066, 1: 0.30326533, 2: 0.07598537, ... 7: 0.00000012, **8: 0.00000006**

即：理想情况下，一个桶里有 8 个元素的概率是 **6×10⁻⁸**（约 1/1500 万）。所以「树化」是一个**几乎不会发生的兜底机制**，只在「hash 被恶意构造 / hashCode 实现极差」时才启用。

**为什么是 8 不是 4 或 16？**
- 太小（如 4）：正常分布下就会频繁树化，而 TreeNode 内存是 Node 的 2 倍（多了 parent/left/right/prev + color），得不偿失；
- 太大（如 16）：链表查找已经慢（平均 8 次比较），树化的收益被延迟；
- 8 是「概率极低 + 收益明显」的平衡点。

### 6 的来源：抗抖动

`UNTREEIFY_THRESHOLD = 6` 与 8 之间留了 **2 的缓冲带**，防止「元素数在 8 附近反复增删」导致**树↔链表频繁转换**（每次转换都要重新建树/建链，代价大）。这是典型的**滞后（hysteresis）设计**。

```
         插入到 8 → 树化
...  7  8  9 ...  6  5 ...
      ← 退化为链表（到 6）
   缓冲带 [7] 内既可以是树也可以是链表，不转换
```

### 64 的来源：小表优先扩容

`MIN_TREEIFY_CAPACITY = 64`：当 `table.length < 64` 时，**即使某个桶链表长度到 8，也不树化，而是扩容**。

**原因**：小表时冲突的根因往往是「表太小 + hash 低位分布不均」，扩容（翻倍）能让元素重新分散到新桶，成本远低于维护一棵红黑树；而且小表往往元素总数也少，树化的收益不明显。**这是「先扩容、后树化」的优先级设计**。

> ⚠️ **易错点：树化阈值 8 是「链表长度」，不是「桶内元素数」？**
> 严格说：`binCount >= TREEIFY_THRESHOLD - 1` 即 `binCount >= 7`，而 `binCount` 从 0 开始计数「遍历过的节点数（不含头节点）」。当遍历到第 8 个节点（`p.next == null`）时 `binCount == 7`，触发树化。所以是「**插入后桶内节点数达到 8**」。

## 4.9 并发下的 HashMap：JDK7 死循环（**阿里 P6+ 必考**）

### 4.9.1 JDK7 的 transfer 源码（头插）

```java
// JDK 7 HashMap.transfer
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while (e != null) {
            Entry<K,V> next = e.next;              // ① 记住下一个
            if (rehash) e.hash = (e.key == null) ? 0 : hash(e.key);
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];                  // ② 头插：新节点指向旧桶头
            newTable[i] = e;                       // ③ 新桶头指向新节点
            e = next;                              // ④ 前进
        }
    }
}
```

### 4.9.2 死循环的形成（逐步推演）

初始：`oldTable[1] → A → B → null`，两个线程 T1、T2 同时 put 触发扩容。

```
步骤 1：T1 执行到 ① 后（next = B），被挂起。此时 A.next = B。
步骤 2：T2 完整执行完 transfer。因为头插，链序反转：
        newTable[1] → B → A → null
        且此时 B.next = A，A.next = null。
步骤 3：T1 恢复，继续处理 A：
        e = A, next = B（T1 手里记的）
        e.next = newTable[i] → A.next = B（← 覆盖了 A.next = null）
        newTable[i] = A       → newTable[1] → A → B → A → ...
        e = next = B
步骤 4：T1 处理 B：
        e = B, next = B.next = A   ← 因为 T2 把 B.next 改成了 A
        B.next = newTable[i] = A   → B.next = A
        newTable[i] = B
        e = next = A
步骤 5：T1 处理 A（再次）：
        e = A, next = A.next = B
        A.next = newTable[i] = B
        ...  无限循环：A → B → A → B → ...
```

**结果**：`newTable[1]` 里形成**环形链表** `A ⇄ B`。之后任何 `get` 落到这个桶，`while (e != null)` 永远走不到 null → **CPU 100%**。这是 2011 年淘宝/支付宝线上事故的经典原因。

### 4.9.3 JDK8 为什么不会死循环，但仍然线程不安全

JDK8 用**尾插 + 高低位拆分**，`loTail.next = e; loTail = e;` 保证每条链严格递增，**不会出现「A.next = B 且 B.next = A」**，所以不会成环。

但仍然不安全，会丢数据：

```
T1、T2 同时 put 到同一个空桶：
  T1: tab[i] = newNode(A)        ← 读到 null，准备写
  T2: tab[i] = newNode(B)        ← 也读到 null，后写覆盖
  → A 丢失

T1、T2 同时 ++size：
  size 是普通 int，非原子 → 少算 → size 偏小
  → 后续 resize 阈值判断错误

T1 扩容、T2 同时 put：
  T2 可能把节点插到「已被 T1 迁移的空桶」→ 节点挂在旧表上，永久丢失
```

**结论（标准答法）**：
- **JDK7**：并发 put 可能形成**环形链表**导致 **get 死循环、CPU 打满**（这是「HashMap 线程不安全」最出名的表现）；
- **JDK8**：死循环问题被修复，但仍会**丢数据、size 不准、覆盖**，**依旧不能用于并发**；
- 并发场景用 **`ConcurrentHashMap`**。

> 💡 **延伸：为什么 JDK8 的「尾插」也带来了新问题？**
> 头插的优点是「插入快（O(1)），不用遍历链表」；尾插必须遍历到链尾，所以 JDK8 的 put 在长链场景略慢。但换来的是「扩容顺序稳定 + 不会成环」，且配合树化，长链场景已被红黑树兜底。

## 4.10 get / remove 源码

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(key)) == null ? null : e.value;
}

final Node<K,V> getNode(Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & (hash = hash(key))]) != null) {   // 一次定位
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;                                        // 桶首命中（最常见，O(1)）
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);   // 红黑树 O(log n)
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);                       // 链表 O(k)
        }
    }
    return null;
}

public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

`removeNode` 要点：
- 删除桶首节点：`tab[i] = first.next`；
- 删除链表中间：`p.next = e.next`（需要前驱 `p`）；
- 删除树节点：`((TreeNode)first).removeTreeNode(...)`，可能触发 **`untreeify`**（节点数 ≤ 6）；
- 删除后 `++modCount; --size;`。

## 4.11 key 的契约：hashCode 与 equals（**小米/京东高频**）

### 4.11.1 契约

1. `equals` 相等的两个对象，`hashCode` **必须**相等；
2. `hashCode` 相等的两个对象，`equals` **不一定**相等（允许碰撞）；
3. `equals` 自反、对称、传递、一致；`null.equals` 不存在（用 `Objects.equals` 安全）。

### 4.11.2 如果只重写 equals 不重写 hashCode 会怎样

```java
class Key {
    String id;
    @Override public boolean equals(Object o) { ... 按 id 比较 ... }
    // 没有重写 hashCode → 用的是 Object 的「对象地址哈希」
}

map.put(new Key("1"), "A");
map.get(new Key("1"));   // → null ！
```

因为两个 `new Key("1")` 的 `identityHashCode` 不同 → 落在**不同的桶** → `get` 找不到，即使 `equals` 说相等。**这是最经典的生产事故：自定义对象做 Map key 时只重写 equals。**

反过来，**只重写 hashCode 不重写 equals**：两个不同对象 hash 相同 → 落在同一个桶 → 链表里用 `equals`（默认引用比较）→ 都认为不相等 → 桶里堆积大量节点 → **性能退化，但不丢数据**。

### 4.11.3 可变对象做 key 的陷阱

```java
List<String> key = new ArrayList<>(List.of("a"));
map.put(key, 1);
key.add("b");             // 改变了 hashCode
map.get(key);             // → null ！因为落在错误桶
map.remove(key);          // 也删不掉 → 内存泄漏
```

> ⚠️ **生产建议**：**Map 的 key 必须是不可变对象**（String、Integer、枚举、`record`、final 字段的值对象）。这是《Effective Java》Item 11 的明确建议。

### 4.11.4 String 的 hashCode 为什么用 31

```java
public int hashCode() {
    int h = hash;                     // 缓存（String 不可变，可缓存）
    if (h == 0 && value.length > 0) {
        for (byte v : value) h = 31 * h + (v & 0xff);
        hash = h;
    }
    return h;
}
```

- 31 是**奇素数**：`31 * h` 可以用 `(h << 5) - h` 实现（移位 + 减法，比乘法快）；
- 31 不太大也不太小：太小易碰撞，太大易溢出；
- 实验证明 31 在英文单词集合上碰撞率最低（《Effective Java》引用）；
- **注意**：`String.hashCode` 是**确定且可预测的**，这带来一个安全问题——攻击者可以构造大量 hash 相同的字符串（**Hash 碰撞 DoS**）。Tomcat 曾因此限制 `maxParameterCount`；Java 8u 之后对 `String` 引入了 **`hash32`（随机种子）备用哈希**（`String.hashCode` 仍不变，但 HashMap 在检测到「树化」时会用 `String` 的特殊处理）。

## 4.12 HashMap 的遍历与顺序

| 方式 | 说明 | 性能 |
|---|---|---|
| `entrySet()` + foreach | 最常用，一次遍历拿到 k 和 v | **推荐** |
| `keySet()` + `get(k)` | 每次 get 要再定位一次 | ❌ 慢一倍 |
| `values()` | 只要 value | 快 |
| `forEach((k,v) -> ...)` | JDK8，内部仍走 `entrySet` | 好 |
| `iterator` | 可 `remove` | 好 |

**顺序**：`HashMap` **不保证任何顺序**，且顺序会随扩容改变。需要「插入序」→ `LinkedHashMap`；需要「排序」→ `TreeMap`；需要「稳定哈希序」→ 无（可用 `LinkedHashMap` 或自己排序）。

> 💡 **延伸：为什么遍历顺序看起来「有点规律」？**
> 因为遍历是 `for (i = 0; i < table.length; i++)` 按桶下标扫描，桶内按链表顺序。所以顺序 = 「按 (hash & (n-1)) 升序，同桶内按插入序」。这解释了为什么 `HashMap` 的遍历顺序「既不随机也不插入序」，且**扩容后会变**。

## 4.13 HashMap 常见追问清单（自测）

1. `HashMap` 的 `size` 为什么不是 `table.length`？—— 前者是 KV 对数，后者是桶数。
2. 为什么 `HashMap` 允许 null key？—— `hash(null)` 返回 0，放在 0 号桶；`Hashtable` 不允许是因为它调用 `key.hashCode()` 会 NPE。
3. `HashMap` 的 `capacity` 上限为什么是 `1 << 30`？—— `int` 最大 2^31-1，2^31 溢出成负数；且 2^30 是最大可用 2 的幂。
4. 为什么 `threshold` 在 `resize` 后要重算？—— 因为 `loadFactor` 可能不是 0.75（自定义）。
5. `HashMap` 的 `modCount` 在哪些操作中增加？—— `put`（新增）、`remove`、`clear`、`resize` 本身**不增加** modCount（注意！`resize` 不改 modCount，所以扩容不会触发 CME，但会改变遍历顺序）。
6. 为什么 `entrySet` 的迭代器删除是安全的？—— `HashIterator.remove()` 调 `removeNode` 后会 `expectedModCount = modCount`。
7. `HashMap` 能存多少个元素？—— `size` 是 int，理论上限 `Integer.MAX_VALUE`，但受堆内存限制；容量到 2^30 后不再扩容，链表/树会变长。

> ⚠️ **第 5 条的坑**：`resize()` **不修改 `modCount`**。所以「遍历过程中另一个线程触发扩容」不会抛 CME，只会导致「遍历到 null 或重复」——这正是 HashMap 并发下「弱一致 + 可能死循环」的另一个根源。

## 4.14 HashMap 与 Hashtable / ConcurrentHashMap 快速对比

| 维度 | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| 线程安全 | ❌ | ✅（全表 `synchronized`） | ✅（CAS + 桶级锁） |
| null key/value | ✅ | ❌ | ❌ |
| 初始容量 | 16（懒加载） | 11 | 16（懒加载） |
| 扩容 | 2 倍 | `2*old + 1` | 2 倍（多线程协助） |
| 索引计算 | `hash & (n-1)` | `(hash & 0x7FFFFFFF) % tab.length` | `(n-1) & spread(h)` |
| 迭代器 | fail-fast | fail-fast（`Enumeration` 不快） | **弱一致** |
| 性能 | 单线程最高 | 差 | 并发高 |

> 💡 **延伸：Hashtable 为什么 `2*old + 1`？**
> 为了保持容量是**奇数/素数**，用 `%` 取模时分布更均匀（素数取模能减少规律性碰撞）。这是「开放定址/取模哈希」时代的习惯；JDK8 的 HashMap 改用 2 的幂 + 位运算后就不再需要素数。

---

# 第 5 章 LinkedHashMap、TreeMap、Hashtable

## 5.1 LinkedHashMap：HashMap + 双向链表

### 5.1.1 结构

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;      // ← 额外两个指针，串成双向链表
        Entry(int hash, K key, V value, Node<K,V> next) { super(hash, key, value, next); }
    }

    transient LinkedHashMap.Entry<K,V> head;   // 链表头（最老/最早插入）
    transient LinkedHashMap.Entry<K,V> tail;   // 链表尾（最新）
    final boolean accessOrder;                 // false=插入序，true=访问序
}
```

```
HashMap 的桶数组（哈希索引）          维护顺序的双向链表（贯穿所有 Entry）
┌────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │                head ⇄ Entry ⇄ Entry ⇄ Entry ⇄ tail
└─┬──┴────┴─┬──┴────┘                  ↑                              ↑
  ▼         ▼                       最早插入                     最近插入/访问
Entry ──→ Entry
  （每个 Entry 同时属于「桶链表」和「顺序链表」）
```

**关键设计**：`LinkedHashMap` **完全复用 `HashMap` 的哈希结构**（`table`、`putVal`、`resize`、红黑树树化逻辑），只是：
1. 把 `Node` 换成带 `before/after` 的 `Entry`；
2. 重写 `newNode`、`afterNodeAccess`、`afterNodeInsertion`、`afterNodeRemoval` 四个**回调钩子**。

这正是 HashMap 里那三个 `afterNodeXxx` 空方法存在的意义——**模板方法模式，给子类留钩子**。

### 5.1.2 四个钩子

```java
// HashMap 中的空实现（供 LinkedHashMap 覆盖）
void afterNodeAccess(Node<K,V> p) { }        // 访问后（get/put 命中）
void afterNodeInsertion(boolean evict) { }   // 插入后
void afterNodeRemoval(Node<K,V> p) { }       // 删除后

// LinkedHashMap 的实现
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    // 只有 accessOrder=true 时才把访问的节点移到链尾
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p = (Entry)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null) head = a; else b.after = a;
        if (a != null) a.before = b; else last = b;
        if (last == null) head = p; else { p.before = last; last.after = p; }
        tail = p;
        ++modCount;                     // ← 访问序下的"结构变化"也计入 modCount
    }
}

void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    // removeEldestEntry 默认返回 false；子类覆盖它即可实现 LRU 淘汰
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) { return false; }
```

### 5.1.3 手写 LRU（**字节/阿里必考的手撕题**）

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // initialCapacity, loadFactor, accessOrder=true
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;      // 超过容量就淘汰链表头（最久未使用）
    }
}
```

**三个必须说清的点**：
1. `accessOrder = true`：`get`、`put`（命中已有 key）、`getOrDefault`、`compute` 等都会把节点移到链尾；
2. `removeEldestEntry` 在**插入之后**被调用（`afterNodeInsertion`），此时 `size` 已包含新元素，所以判断 `size() > capacity`；
3. **非线程安全**，并发场景需要 `Collections.synchronizedMap(new LRUCache<>(n))` 或自己加锁（但注意 `synchronizedMap` 的复合操作仍需外部同步）。

**手撕「不借助 LinkedHashMap」的版本（面试常要求）**：

```java
class LRUCache {
    class Node { int k, v; Node prev, next; Node(int k,int v){this.k=k;this.v=v;} }
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0,0), tail = new Node(0,0);  // 哨兵
    private final int cap;

    public LRUCache(int capacity) {
        cap = capacity;
        head.next = tail; tail.prev = head;
    }

    public int get(int key) {
        Node n = map.get(key);
        if (n == null) return -1;
        moveToTail(n);          // 访问即提权
        return n.v;
    }

    public void put(int key, int value) {
        Node n = map.get(key);
        if (n != null) { n.v = value; moveToTail(n); return; }
        if (map.size() == cap) {                 // 淘汰头结点（最久未用）
            Node old = head.next;
            remove(old);
            map.remove(old.k);
        }
        Node nn = new Node(key, value);
        map.put(key, nn);
        addToTail(nn);
    }
    // addToTail / remove / moveToTail 均为 O(1) 指针操作
}
```

> ⚠️ **易错点：为什么哨兵节点 `head/tail` 能让代码少一半 if 判断？**
> 因为 `head.next` 永远有效（即使空表也指向 `tail`），`node.prev` 永远非 null，所以 `remove` / `addToTail` 不需要判空。这是链表题的标准技巧。

### 5.1.4 插入序 vs 访问序

| accessOrder | 顺序含义 | 典型用途 |
|---|---|---|
| `false`（默认） | **插入顺序**（`put` 新 key 时追加到尾部；覆盖已有 key **不改变位置**） | 保持配置项顺序、JSON 序列化字段顺序、`LinkedHashSet` |
| `true` | **访问顺序**（每次 `get`/`put` 命中都移到尾部） | **LRU 缓存**、最近使用列表 |

> ⚠️ **易错点：`accessOrder = true` 时，`get` 会修改 `modCount`**，所以「遍历中 `get`」会抛 `ConcurrentModificationException`！这是很多人踩过的坑。另外 `accessOrder = true` 的 `LinkedHashMap` **不能并发读**（读也是写操作），比 `false` 更不适合共享。

---

## 5.2 TreeMap：红黑树的有序 Map

### 5.2.1 结构

```java
public class TreeMap<K,V> extends AbstractMap<K,V>
        implements NavigableMap<K,V>, Cloneable, Serializable {
    private final Comparator<? super K> comparator;   // 可为 null（用 Comparable）
    private transient Entry<K,V> root;                // 红黑树根
    private transient int size = 0;
    private transient int modCount = 0;

    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key; V value;
        Entry<K,V> left, right, parent;
        boolean color = BLACK;
    }
}
```

### 5.2.2 核心操作

```java
// 查找：标准 BST 查找，靠 compare 决定方向
final Entry<K,V> getEntry(Object key) {
    if (comparator != null) return getEntryUsingComparator(key);
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0) p = p.left;
        else if (cmp > 0) p = p.right;
        else return p;
    }
    return null;
}

// 插入：BST 插入 + fixAfterInsertion（红黑树平衡）
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) { compare(key, key); root = new Entry<>(key, value, null); size = 1; modCount++; return null; }
    int cmp; Entry<K,V> parent;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {                       // 有 comparator：用它
        do { parent = t; cmp = cpr.compare(key, t.key);
             if (cmp < 0) t = t.left; else if (cmp > 0) t = t.right; else return t.setValue(value);
        } while (t != null);
    } else {                                 // 无 comparator：用 Comparable，key 为 null 直接 NPE
        if (key == null) throw new NullPointerException();
        Comparable<? super K> k = (Comparable<? super K>) key;
        do { parent = t; cmp = k.compareTo(t.key); ... } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0) parent.left = e; else parent.right = e;
    fixAfterInsertion(e);                    // 红黑树修复（见 2.4.2）
    size++; modCount++;
    return null;
}
```

### 5.2.3 NavigableMap 的强大 API（**面试加分**）

| 方法 | 含义 | 复杂度 |
|---|---|---|
| `firstKey()` / `lastKey()` | 最小 / 最大 key | O(log n) |
| `ceilingKey(k)` | ≥ k 的最小 key | O(log n) |
| `floorKey(k)` | ≤ k 的最大 key | O(log n) |
| `higherKey(k)` / `lowerKey(k)` | 严格大于 / 小于 | O(log n) |
| `headMap(k)` / `tailMap(k)` / `subMap(a,b)` | **视图**，范围操作 | O(log n) 定位，遍历 O(m) |
| `descendingMap()` | 逆序视图 | O(1) |
| `pollFirstEntry()` / `pollLastEntry()` | 取出并删除极值 | O(log n) |
| `firstEntry()` / `lastEntry()` | 极值 Entry | O(log n) |

**典型用途**：排行榜区间查询、时间窗口、`ceilingKey` 找「≥ 目标值的最小配置」、`subMap` 做范围过滤。

### 5.2.4 使用约束与坑

| 约束 | 说明 |
|---|---|
| key 必须可比较 | 要么实现 `Comparable`，要么构造时传 `Comparator`；否则 `ClassCastException` |
| 不允许 null key | `compareTo` 会 NPE（除非 comparator 自己处理 null） |
| 允许 null value | ✅ |
| 非线程安全 | 并发有序 Map 用 `ConcurrentSkipListMap` |
| `equals`/`hashCode` 不参与 | **只用 `compare`** 判等。若 `compare` 返回 0 而 `equals` 为 false，会被**当成同一个 key 覆盖**（这是经典坑） |
| 视图与原 Map 联动 | `subMap` 是视图，改视图就是改原 Map；原 Map 结构性修改后视图操作可能 CME |
| 遍历有序 | `entrySet()` 是**中序遍历**，O(n) |

> ⚠️ **经典坑**：
> ```java
> Comparator<String> byLength = Comparator.comparingInt(String::length);
> TreeMap<String, Integer> map = new TreeMap<>(byLength);
> map.put("aa", 1);
> map.put("bb", 2);     // compare("aa","bb") == 0 → 覆盖！size 仍为 1
> ```
> 因为 `Comparator` 只比长度，`aa` 和 `bb` 被视为「相等」。**结论：`Comparator` 必须与 `equals` 语义一致（至少不能把不相等的东西判成相等）。**

### 5.2.5 TreeMap vs HashMap

| 维度 | TreeMap | HashMap |
|---|---|---|
| 底层 | 红黑树 | 数组 + 链表/红黑树 |
| 顺序 | **有序（按 key 排序）** | 无序 |
| 查找 | O(log n) | 平均 O(1)，最坏 O(log n)（树化后） |
| 插入/删除 | O(log n)，需要平衡 | 平均 O(1) |
| null key | ❌ | ✅ |
| 范围查询 | ✅（`subMap` 等） | ❌ |
| 内存 | 每节点 5 个字段 + 颜色 | 每节点 4 个字段 |
| 适用 | 需要排序/范围/极值 | 一般 KV 存储 |

> 💡 **延伸：为什么 `TreeMap` 不用跳表？**
> 单线程场景下红黑树内存更省（不需要多层索引），且 `TreeMap` 的历史早于并发需求。跳表的优势在**并发无锁**和**范围查询的缓存友好**，所以只在 JUC 里用（`ConcurrentSkipListMap`）。

---

## 5.3 Hashtable（遗留类，但面试常问）

```java
public class Hashtable<K,V> extends Dictionary<K,V> implements Map<K,V>, Cloneable, Serializable {
    private transient Entry<?,?>[] table;
    private transient int count;
    private int threshold;
    private float loadFactor;

    public synchronized V put(K key, V value) {
        if (value == null) throw new NullPointerException();   // 不允许 null value
        ...
        int hash = key.hashCode();                             // key 为 null → NPE
        int index = (hash & 0x7FFFFFFF) % tab.length;          // 去掉符号位后取模
        ...
    }
    public synchronized V get(Object key) { ... }              // 全部方法 synchronized
}
```

### 核心差异（对比 HashMap）

| 项 | Hashtable | HashMap |
|---|---|---|
| 线程安全 | ✅ 方法级 `synchronized`（锁整个对象） | ❌ |
| null | **key/value 都不允许** | 都允许 |
| 初始容量 | 11 | 16（懒加载） |
| 扩容 | `2*old + 1`（保持奇数） | 2 倍（保持 2 的幂） |
| 索引 | `(hash & 0x7FFFFFFF) % len` | `hash & (len-1)` |
| 遍历 | `Enumeration`（不抛 CME）/ `Iterator`（fail-fast） | `Iterator`（fail-fast） |
| 继承 | `Dictionary`（已废弃） | `AbstractMap` |
| 替代 | `ConcurrentHashMap` | — |

### 为什么 Hashtable 不允许 null

源码直接 `throw new NullPointerException`，设计动机是「**早期 JDK 的保守设计**」：允许 null 会让 `get` 无法区分「不存在」和「值为 null」，而 `Hashtable` 是给多线程共享用的（`contains` 需要额外调用，且并发下不可靠）。JDK 后来的 `ConcurrentHashMap` 沿用这个约束，理由更充分（见 7.6）。

> 💡 **延伸：`Collections.synchronizedMap` 与 `Hashtable` 的区别？**
> `synchronizedMap` 是**装饰器**，把任意 Map 包一层，**所有方法都 `synchronized (mutex)`**，包括 `get/put/size`；迭代时必须手动 `synchronized (map) { ... }`，否则遍历中其他线程修改会抛 CME。`Hashtable` 是**内置**的，`Enumeration` 遍历不强一致。两者性能都差，现代代码应使用 `ConcurrentHashMap`。

---

# 第 6 章 Set 家族与 Queue 家族

## 6.1 Set 家族：本质都是 Map

### 6.1.1 HashSet

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, Serializable {
    private transient HashMap<E,Object> map;          // ← 底层就是 HashMap
    private static final Object PRESENT = new Object();  // 哨兵 value（所有 key 共享）

    public HashSet() { map = new HashMap<>(); }

    public boolean add(E e) { return map.put(e, PRESENT) == null; }   // put 返回 null 表示新增

    public boolean remove(Object o) { return map.remove(o) == PRESENT; }

    public boolean contains(Object o) { return map.containsKey(o); }

    public Iterator<E> iterator() { return map.keySet().iterator(); }

    public int size() { return map.size(); }
}
```

**为什么用 `PRESENT` 静态哨兵而不是 `new Object()`？**
1. 省内存：所有 Entry 的 value 指向同一个对象，不额外分配；
2. `remove` 可以用 `== PRESENT` 快速判断（引用比较），但注意源码里其实用的是 `map.remove(o) == PRESENT`，如果 value 被替换过就失效——所以正确性来自「value 永远是 PRESENT」。

**HashSet 的核心语义**：
- `add` 依赖 **`hashCode` + `equals`**：`hashCode` 决定桶，`equals` 决定桶内是否重复；
- **允许一个 null**（`hash(null) = 0`）；
- 无序（顺序由 hash 决定，扩容后会变）；
- 非线程安全。

**经典问题：`HashSet` 存自定义对象去重失败**

```java
Set<Person> set = new HashSet<>();
set.add(new Person("张三", 18));
set.add(new Person("张三", 18));   // 期望去重，实际 size = 2
```
因为 `Person` 没重写 `hashCode`/`equals` → 地址不同 → 不同桶。**必须同时重写两者，且用相同的字段。**

### 6.1.2 LinkedHashSet

```java
public class LinkedHashSet<E> extends HashSet<E> implements Set<E>, Cloneable, Serializable {
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);      // ← 三个参数的"包可见"构造
    }
}
// HashSet 中的特殊构造（专供 LinkedHashSet）
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);   // 换成 LinkedHashMap
}
```

- 有序：默认**插入序**（`accessOrder = false`）；
- 迭代顺序稳定，适合「去重 + 保序」场景（如「保留首次出现的元素」）；
- 性能略低于 `HashSet`（多维护一条双向链表）。

### 6.1.3 TreeSet

```java
public class TreeSet<E> extends AbstractSet<E> implements NavigableSet<E>, Cloneable, Serializable {
    private transient NavigableMap<E,Object> m;
    private static final Object PRESENT = new Object();

    TreeSet(NavigableMap<E,Object> m) { this.m = m; }
    public TreeSet() { this(new TreeMap<>()); }

    public boolean add(E e) { return m.put(e, PRESENT) == null; }
    // 大量委托方法：ceiling、floor、headSet、tailSet、subSet、descendingSet、pollFirst...
}
```

- 底层 `TreeMap`（红黑树），**有序**；
- 元素必须可比较（`Comparable` 或 `Comparator`）；
- 不允许 null（`TreeMap` 的约束）；
- 支持 `NavigableSet` 的范围/极值操作；
- `Comparator` 与 `equals` 不一致时会「误去重」（同 5.2.4 坑）。

### 6.1.4 EnumSet（最被低估的高性能集合）

```java
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E> implements Cloneable, Serializable {
    final Class<E> elementType;
    final Enum<?>[] universe;
    // 内部用 long 位向量（RegularEnumSet）或 long[]（JumboEnumSet，枚举 > 64 个时）
}
```

- **位向量实现**：`RegularEnumSet` 用一个 `long` 的每一位表示一个枚举常量是否存在；
- `add`/`contains`/`remove` 都是 **O(1) 位运算**；
- 内存极小（一个 long + 两个引用）；
- 迭代按**枚举声明顺序**（天然有序）；
- **不是线程安全的**（但可 `Collections.unmodifiableSet`）；
- 典型用途：权限位、状态标志、配置开关集合。

```java
EnumSet<Permission> perms = EnumSet.of(READ, WRITE);
EnumSet<Permission> all = EnumSet.allOf(Permission.class);
EnumSet<Permission> none = EnumSet.noneOf(Permission.class);
```

> 💡 **延伸：为什么 Java 不用 `BitSet` 做权限？**
> `BitSet` 用下标（int）表示位，语义弱、易越界；`EnumSet` 用枚举类型做类型安全约束，且 `EnumSet` 的位运算已优化到极致（`RegularEnumSet` 的 `add` 就是 `elements |= (1L << ordinal)`）。

### 6.1.5 Set 家族对比

| | HashSet | LinkedHashSet | TreeSet | EnumSet | CopyOnWriteArraySet | ConcurrentSkipListSet |
|---|---|---|---|---|---|---|
| 底层 | HashMap | LinkedHashMap | TreeMap | long 位向量 | COWArrayList | CSLM（跳表） |
| 顺序 | 无序 | 插入序 | 排序 | 枚举声明序 | 插入序 | 排序 |
| 查找 | O(1) | O(1) | O(log n) | **O(1) 位运算** | O(n) | O(log n) |
| null | ✅ 一个 | ✅ 一个 | ❌ | ❌ | ✅ | ❌ |
| 线程安全 | ❌ | ❌ | ❌ | ❌ | ✅（写锁） | ✅（无锁） |
| 适用 | 通用去重 | 去重保序 | 去重+排序/范围 | 枚举标志 | 读多写少 | 并发有序 |

## 6.2 Queue 家族

### 6.2.1 ArrayDeque：循环数组，最快的双端队列

```java
public class ArrayDeque<E> extends AbstractCollection<E> implements Deque<E>, Cloneable, Serializable {
    transient Object[] elements;      // 循环数组
    transient int head;               // 头指针（指向队首元素）
    transient int tail;               // 尾指针（指向"下一个可插入位置"）
    private static final int MIN_INITIAL_CAPACITY = 8;

    public ArrayDeque() { elements = new Object[16]; }   // 默认 16

    public void addFirst(E e) {
        if (e == null) throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;   // 头指针左移（环形）
        if (head == tail) doubleCapacity();                        // 满 → 扩容
    }

    public void addLast(E e) {
        if (e == null) throw new NullPointerException();
        elements[tail] = e;
        if ((tail = (tail + 1) & (elements.length - 1)) == head) doubleCapacity();
    }

    public E pollFirst() {
        int h = head; E result = (E) elements[h];
        if (result == null) return null;
        elements[h] = null;                       // 置 null 帮助 GC
        head = (h + 1) & (elements.length - 1);
        return result;
    }

    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1);
        E result = (E) elements[t];
        if (result == null) return null;
        elements[t] = null;
        tail = t;
        return result;
    }
}
```

**循环数组图解**：

```
elements.length = 8，head = 5，tail = 2（已存 5 个元素，占 5,6,7,0,1）

索引:  0    1    2    3    4    5    6    7
     ┌────┬────┬────┬────┬────┬────┬────┬────┐
     │ D  │ E  │    │    │    │ A  │ B  │ C  │
     └────┴────┴────┴────┴────┴────┴────┴────┘
       ▲              ▲         ▲
       │              │         └── head（队首 A）
       └──────────────┴── tail（下一个写入位置）
       
逻辑顺序：A(5) → B(6) → C(7) → D(0) → E(1)
```

**关键设计**：
- **`tail` 指向「下一个空位」，`head == tail` 表示「空」**（因为数组里始终留一个空槽来区分空/满）；
- 扩容 `doubleCapacity()`：把 `head` 之后的部分先拷到新数组开头，再把 `head` 之前的部分拷到后面，`head = 0, tail = size`；
- `(i - 1) & (len - 1)` 实现「环形左移」：`len` 是 2 的幂，`-1` 在二进制补码下等价于「加 len-1 再取模」，所以 `(0 - 1) & 7 = 7`，天然绕回；
- **不允许 null**（因为 `null` 被用来表示「空槽」，`poll` 靠 `result == null` 判断空）。

**为什么 ArrayDeque 比 Stack / LinkedList 都好？**

| | ArrayDeque | Stack | LinkedList |
|---|---|---|---|
| 底层 | 循环数组 | 数组（Vector） | 双链表 |
| 同步 | 无锁 | **synchronized（全表）** | 无锁 |
| 内存 | 紧凑 | 紧凑 + 扩容冗余 | 每节点 24B |
| 缓存 | **好** | 好 | 差 |
| 头尾操作 | **O(1) 均摊** | 头操作 O(n)（Stack 用 push 在尾部，其实 O(1)，但 Vector 需同步） | O(1) |
| 结论 | **首选** | 遗留，慢 | 次选（大元素、频繁中间删） |

> ⚠️ **易错点：`ArrayDeque` 不能存 null**，但 `LinkedList` 可以。如果业务需要「队列里放 null」，只能用 `LinkedList` 或 `LinkedBlockingQueue`（后者也不允许 null，所以实际上必须换成哨兵对象）。

> 💡 **延伸：为什么 JDK 说「`ArrayDeque` 作为栈比 `Stack` 快」？**
> ① 无 `synchronized`；② 循环数组的头插是 `head--` 的位运算，而 `Stack.push` 调 `Vector.addElement` 需要 `ensureCapacityHelper` + 同步；③ `ArrayDeque` 的数组不会被 `Vector` 那种「2 倍扩容」浪费。JMH 实测约 **3~5 倍**差距。

### 6.2.2 PriorityQueue：二叉堆

```java
public class PriorityQueue<E> extends AbstractQueue<E> implements Serializable {
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    transient Object[] queue;
    private int size = 0;
    private final Comparator<? super E> comparator;
    transient int modCount;

    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();       // 不允许 null
        modCount++;
        int i = size;
        if (i >= queue.length) grow(i + 1);
        size = i + 1;
        if (i == 0) queue[0] = e;                              // 第一个元素直接放根
        else siftUp(i, e);                                     // 上浮
        return true;
    }

    private void siftUp(int k, E x) {
        if (comparator != null) siftUpUsingComparator(k, x);
        else siftUpComparable(k, x);
    }

    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;                        // 父节点下标
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0) break;              // 比父大 → 停
            queue[k] = e;                                      // 父下移
            k = parent;
        }
        queue[k] = key;
    }

    public E poll() {
        if (size == 0) return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];                               // 取出堆顶
        E x = (E) queue[s];
        queue[s] = null;                                       // 末尾置 null
        if (s != 0) siftDown(0, x);                            // 末尾元素放根后下沉
        return result;
    }

    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        int half = size >>> 1;                                  // 只需下沉到叶子层之上
        while (k < half) {
            int child = (k << 1) + 1;                           // 左孩子
            Object c = queue[child];
            int right = child + 1;
            if (right < size && ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];                       // 选较小的孩子
            if (key.compareTo((E) c) <= 0) break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
}
```

**复杂度表**：

| 操作 | 复杂度 | 说明 |
|---|---|---|
| `offer`（入堆） | **O(log n)** | 上浮，最多树高 |
| `poll`（出堆） | **O(log n)** | 下沉 |
| `peek` | **O(1)** | 看堆顶 |
| `remove(Object)` | O(n) 查找 + O(log n) 删除 | 先线性找，再删 |
| `contains` | O(n) | 堆不是哈希 |
| **建堆 `heapify`** | **O(n)** | 从 `n/2 - 1` 下沉，数学期望 O(n) |
| 迭代 | **不保证有序** | 只有反复 `poll` 才有序 |

**扩容策略**：

```java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // 小于 64 时翻倍，否则 1.5 倍
    int newCapacity = oldCapacity + ((oldCapacity < 64) ? (oldCapacity + 2) : (oldCapacity >> 1));
    if (newCapacity - MAX_ARRAY_SIZE > 0) newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}
```

**TopK 问题（**面试高频手撕**）**：

```java
// 求「最大的 K 个元素」→ 用小顶堆（堆顶是这 K 个里最小的，来了更大的就替换堆顶）
public int[] topK(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);   // 默认小顶堆
    for (int n : nums) {
        if (minHeap.size() < k) minHeap.offer(n);
        else if (n > minHeap.peek()) { minHeap.poll(); minHeap.offer(n); }
    }
    // 输出
    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = minHeap.poll();
    return res;
}
// 时间 O(n log k)，空间 O(k)。比"全排序 O(n log n)"更适合海量数据/流式数据。
```

> ⚠️ **易错点：求「最大 K 个」用**小顶堆**，求「最小 K 个」用**大顶堆**。
> 因为你要「淘汰」的是「当前 K 个里最不该留的那个」——求最大 K 个时，最不该留的是 K 个里最小的，所以堆顶要能快速拿到最小值 → 小顶堆。这是反直觉但必须记牢的点。

**典型用途**：任务调度（按优先级）、Dijkstra 最短路、Huffman 编码、合并 K 个有序链表、定时任务（配合 DelayQueue）、`ThreadPoolExecutor` 的优先队列。

### 6.2.3 阻塞队列（BlockingQueue 家族）

#### ArrayBlockingQueue：单锁双条件

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
    final Object[] items;
    int takeIndex, putIndex, count;
    final ReentrantLock lock;            // ← 一把锁
    private final Condition notEmpty;    // 消费者等待条件
    private final Condition notFull;     // 生产者等待条件

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);           // 可选公平锁
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
    }

    public void put(E e) throws InterruptedException {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)          // ← 必须 while，防虚假唤醒
                notFull.await();
            enqueue(e);
        } finally { lock.unlock(); }
    }

    private void enqueue(E e) {
        items[putIndex] = e;
        if (++putIndex == items.length) putIndex = 0;   // 环形
        count++;
        notEmpty.signal();                              // 唤醒一个消费者
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) notEmpty.await();
            return dequeue();
        } finally { lock.unlock(); }
    }
}
```

**为什么用「一把锁 + 两个 Condition」？**
因为底层是**同一个数组**，`putIndex` 和 `takeIndex` 共享 `count`，读写必须互斥（否则 `count++` 和 `count--` 会竞争）。用两个 `Condition` 是为了「精准唤醒」——生产者只唤醒消费者，消费者只唤醒生产者，减少无效唤醒。

**为什么 `await()` 必须包在 `while` 里？**
`Condition.await()` 可能**虚假唤醒（spurious wakeup）**（JVM 允许，尤其在某些信号处理/OS 调度下），唤醒后条件可能仍不满足，必须重新检查。**`if` 是错误写法。**

#### LinkedBlockingQueue：双锁 + 原子计数（吞吐更高）

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
    static class Node<E> { E item; Node<E> next; Node(E x) { item = x; } }

    private final int capacity;
    private final AtomicInteger count = new AtomicInteger();     // ← 原子计数
    private final ReentrantLock takeLock = new ReentrantLock();  // ← 出队锁
    private final Condition notEmpty = takeLock.newCondition();
    private final ReentrantLock putLock = new ReentrantLock();   // ← 入队锁
    private final Condition notFull = putLock.newCondition();
    transient Node<E> head, last;

    public LinkedBlockingQueue() { this(Integer.MAX_VALUE); }    // 默认无界！⚠️
}
```

**双锁为什么能并存？**
因为链表**头尾操作互不干扰**：`put` 只改 `last`，`take` 只改 `head`；`count` 用 `AtomicInteger` 保证可见性与原子性。所以「生产者」和「消费者」可以**真正并行**（`ArrayBlockingQueue` 则必须串行）。

**代价**：
1. 每个元素都要 `new Node`（内存 + GC 压力），`ArrayBlockingQueue` 是预分配数组；
2. `count` 用 CAS，高并发下有开销；
3. **默认 `Integer.MAX_VALUE` 是无界**，生产速度 > 消费速度会 **OOM**（必考坑，必须显式指定容量）。

**两者的经典对比（**拼多多/字节必答**）**：

| 维度 | ArrayBlockingQueue | LinkedBlockingQueue |
|---|---|---|
| 结构 | 数组（预分配） | 链表（动态分配节点） |
| 锁 | **一把** ReentrantLock（读写互斥） | **两把**（putLock / takeLock 可并行） |
| 计数 | `int count`（锁保护） | `AtomicInteger count` |
| 吞吐 | 较低（读写串行） | **较高**（读写并行） |
| 内存 | 无额外对象，GC 友好 | 每元素一个 Node，GC 压力大 |
| 容量 | **必须有界** | 可选有界（默认无界，危险） |
| 公平性 | 可选公平锁 | 不支持公平（非公平） |
| 适用 | 稳定、内存敏感、需要公平 | 高吞吐、生产消费速度接近 |

#### 其他阻塞队列

| 队列 | 结构 | 特点 | 典型场景 |
|---|---|---|---|
| `SynchronousQueue` | 无容量（公平模式 TransferQueue / 非公平 TransferStack） | **put 必须等到 take**，不存储元素，直接移交 | `Executors.newCachedThreadPool()` 的默认队列；高并发「手递手」 |
| `PriorityBlockingQueue` | `PriorityQueue` + `ReentrantLock` + `notEmpty` | **无界**（`Integer.MAX_VALUE - 8`），按优先级出队，**不允许 null** | 优先级任务调度 |
| `DelayQueue` | `PriorityQueue` + `ReentrantLock` + `leader` 优化 | 元素必须实现 `Delayed`，**到期才能 take**；无界 | 订单超时取消、缓存过期、定时任务 |
| `LinkedTransferQueue` | 链表 + `tryTransfer` | 支持「生产者直接移交 + 未消费可立即返回」 | 高吞吐传递、`ConcurrentLinkedQueue` 的增强 |
| `LinkedBlockingDeque` | 双向链表 + 单锁双条件 | 双端阻塞队列，支持 `putFirst/takeLast` | 工作窃取（ForkJoin 有自己的实现） |

**DelayQueue 的 `leader` 优化（面试加分）**：

```java
// 多个消费者等待时，只有一个成为 leader，负责"按时"唤醒自己并取元素；
// 其他消费者无限等待。避免"所有线程同时醒来抢同一个元素"的惊群效应。
private Thread leader;
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0) return q.poll();
                first = null;                     // 不持引用，避免 GC 泄漏
                if (leader != null) available.await();   // 已有 leader，自己睡
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try { available.awaitNanos(delay); }    // leader 定时醒
                    finally { if (leader == thisThread) leader = null; }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null) available.signal();
        lock.unlock();
    }
}
```

#### 阻塞队列的方法语义（**必背四级 API**）

| 操作 | 抛异常 | 返回特殊值 | 阻塞 | 超时 |
|---|---|---|---|---|
| 插入 | `add(e)` | `offer(e)` | `put(e)` | `offer(e, timeout, unit)` |
| 移除 | `remove()` | `poll()` | `take()` | `poll(timeout, unit)` |
| 检查 | `element()` | `peek()` | — | — |

#### 线程池的队列选择（**高频场景题**）

```java
// Executors 的四个工厂方法对应的队列：
newFixedThreadPool(n)      → LinkedBlockingQueue（无界 ⚠️）
newSingleThreadExecutor()  → LinkedBlockingQueue（无界 ⚠️）
newCachedThreadPool()      → SynchronousQueue（无缓冲，可能创建大量线程 ⚠️）
newScheduledThreadPool(n)  → DelayedWorkQueue（ScheduledThreadPoolExecutor 内部）
```

> ⚠️ **阿里规约明确禁止用 `Executors` 创建线程池**，原因是：
> ① `FixedThreadPool`/`SingleThreadPool` 用无界队列 → 任务堆积 → **OOM**；
> ② `CachedThreadPool`/`ScheduledThreadPool` 的 `maximumPoolSize = Integer.MAX_VALUE` → 线程爆炸 → **OOM**。
> 正确做法是 `new ThreadPoolExecutor(...)` 显式指定**有界队列 + 有界最大线程数 + 拒绝策略**。

### 6.2.4 ConcurrentLinkedQueue：无锁链表队列

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E> implements Queue<E>, Serializable {
    private static class Node<E> {
        volatile E item;
        volatile Node<E> next;      // ← volatile，配合 CAS
        // 内部用 Unsafe 的 casItem / casNext
    }
    private transient volatile Node<E> head;
    private transient volatile Node<E> tail;
}
```

- **Michael-Scott 算法**：`offer` 用 CAS 把新节点挂到 `tail.next`，失败就重试；
- `head/tail` 允许「滞后」（不总指向真正的头尾），减少 CAS 竞争；
- `size()` 是 **O(n) 遍历**（不维护 count），这是它的代价；
- 无界、非阻塞、**不允许 null**；
- 适用于「高并发生产消费，但不需要阻塞等待」的场景（如事件分发）。

> 💡 **延伸：为什么 `ConcurrentLinkedQueue.size()` 是 O(n)？**
> 因为要维护精确 `count` 就必须用 CAS 更新计数器，而每次 `offer/poll` 都会竞争这个计数器 → 成为性能瓶颈。所以设计上**牺牲 `size()` 的 O(1)**，换取高吞吐。如果业务频繁调 `size()`，应改用 `LinkedBlockingQueue`（`AtomicInteger` 计数）或 `ConcurrentHashMap` 的 `mappingCount`。

---

# 第 7 章 ConcurrentHashMap 全链路拆解

> 这是 JUC 面试的「第二主战场」。核心是**两次架构演进**：JDK7 的「分段锁」→ JDK8 的「CAS + synchronized 桶锁」。

## 7.1 为什么需要 ConcurrentHashMap

三个历史阶段：

| 方案 | 问题 |
|---|---|
| `HashMap` | 线程不安全（JDK7 死循环、JDK8 丢数据） |
| `Hashtable` / `Collections.synchronizedMap` | **全表锁**，所有读写串行，并发度 = 1 |
| **`ConcurrentHashMap`** | 锁粒度细化到「桶」，并发度 ≈ 桶数量 |

**核心目标**：在保证线程安全的前提下，把「锁的粒度」尽可能缩小，并让「读」尽可能**无锁**。

## 7.2 JDK7：分段锁（Segment）

### 7.2.1 结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable {
    final Segment<K,V>[] segments;          // Segment 数组，默认 16 个

    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        transient volatile HashEntry<K,V>[] table;   // 每个 Segment 内部是一个 HashMap 式数组
        transient int count;
        transient int modCount;
        transient int threshold;
        final float loadFactor;
    }

    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;             // value 用 volatile 保证读可见性
        final HashEntry<K,V> next;    // 链表，final → 只能头插，且删除要重建链
    }
}
```

**结构图**：

```
ConcurrentHashMap (JDK7)
┌──────────────────────────────────────────────────────────────┐
│ Segment[0]  ──→ HashEntry[] ──→ [HE] → [HE] → null            │
│ Segment[1]  ──→ HashEntry[] ──→ [HE] → null                    │
│ ...                                                           │
│ Segment[15] ──→ HashEntry[] ──→ [HE] → [HE] → [HE] → null     │
└──────────────────────────────────────────────────────────────┘
   每把锁保护一个 Segment（默认 16 段 → 理论并发度 16）
```

### 7.2.2 定位与加锁

```java
// 1. 先算 Segment 下标（用「segment 数量」的高位）
final Segment<K,V> segmentFor(int hash) {
    return segments[(hash >>> segmentShift) & segmentMask];
}

// 2. 再在 Segment 内部用 hash 定位 HashEntry 下标
int index = (tab.length - 1) & hash;
```

**为什么用「高位数」定位 Segment，用「低位数」定位 HashEntry？**
因为 `HashMap` 内部用低位数做索引；若 Segment 也用低位数，会导致「同一 Segment 内的元素 hash 低位相同」而「Segment 之间分布不均」。用高位做 Segment 索引可以让**两层索引互相独立**，分布更均匀。这也是为什么 JDK7 的 `hash` 需要再做一次扩散（`hash = hash ^ (hash >>> 16)` 之类）。

### 7.2.3 put 流程

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null) throw new NullPointerException();   // value 不允许 null
    int hash = hash(key);                                   // key 为 null 会 NPE
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);                               // 懒初始化 Segment
    return s.put(key, hash, value, false);
}

// Segment.put
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);  // ① 先 tryLock
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {                  // ② 遍历桶内链表
            if (e != null) {
                K k;
                if ((k = e.key) == key || (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) { e.value = value; ++modCount; }
                    break;
                }
                e = e.next;
            } else {
                if (node != null) node.setNext(first);       // ③ 头插
                else node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY) rehash(node);   // ④ 扩容
                else setEntryAt(tab, index, node);
                ++modCount; count = c;
                oldValue = null;
                break;
            }
        }
    } finally { unlock(); }                                  // ⑤ 解锁
    return oldValue;
}
```

**`scanAndLockForPut` 的巧妙设计**：

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1;
    while (!tryLock()) {
        HashEntry<K,V> f;
        if (retries < 0) {
            if (e == null) {
                if (node == null) node = new HashEntry<K,V>(hash, key, value, null);   // 预先创建节点
                retries = 0;
            } else if (key.equals(e.key)) retries = 0;
            else e = e.next;
        } else if (++retries > MAX_SCAN_RETRIES) {          // 超过阈值 → 阻塞
            lock();
            break;
        } else if ((retries & 1) == 0 && (f = entryForHash(this, hash)) != first) {
            e = first = f; retries = -1;                    // 桶头变了 → 重新扫描
        }
    }
    return node;
}
```

**设计精妙之处**：自旋期间**顺便遍历链表**（找 key、预建节点），把「等待时间」变成「有效工作时间」；同时检测「桶头是否变化」来决定是否重试。这是「**乐观自旋 + 悲观阻塞**」的经典组合。

### 7.2.4 size 的实现（JDK7 的「模棱两可」难题）

```java
public int size() {
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow;
    long sum;
    long check = 0;
    int retries = -1;
    try {
        for (;;) {
            check = 0; sum = 0;
            for (int i = 0; i < segments.length; ++i) {
                sum += segments[i].count;      // ① 先无锁累加，同时累加 modCount
                check += segments[i].modCount;
            }
            if (check == sumLast) break;       // ② 两次 modCount 一致 → 认为估算可信
            ...
            if (retries++ > 2) {               // ③ 重试 3 次仍不一致 → 锁全部 Segment
                for (int i = 0; i < segments.length; ++i) segments[i].lock();
                ... 逐个累加 ...
                finally 全部 unlock
                break;
            }
        }
    }
    ...
}
```

**核心思想**：`size()` 是**弱一致估算**。先无锁读，用 `modCount` 校验「读的过程中有没有突变」；如果反复不一致，才升级为「锁全表」的悲观统计。这体现了并发容器的通用设计哲学：**优先乐观，冲突时才悲观**。

## 7.3 JDK8：CAS + synchronized 桶锁

### 7.3.1 结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable {
    transient volatile Node<K,V>[] table;
    private transient volatile Node<K,V>[] nextTable;   // 扩容期间的过渡表
    private transient volatile long baseCount;          // 基础计数
    private transient volatile int sizeCtl;             // 控制标识（见下）
    private transient volatile int transferIndex;       // 扩容进度
    private transient volatile CounterCell[] counterCells;  // 分散计数（类似 LongAdder）

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; final K key;
        volatile V val;              // ← val 是 volatile，读无需加锁
        volatile Node<K,V> next;     // ← next 也是 volatile
    }

    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;  // 扩容时标记「本桶已迁移」，hash = MOVED(-1)
    }

    static final class TreeBin<K,V> extends Node<K,V> {  // 桶内红黑树的包装（含读写锁）
        TreeNode<K,V> root; volatile TreeNode<K,V> first; volatile Thread waiter; volatile int lockState;
    }
}
```

### 7.3.2 sizeCtl 的五种取值（**必背**）

| 值 | 含义 |
|---|---|
| `0` | 默认初始状态（表未初始化） |
| `-1` | **正在初始化**（有线程持有初始化权） |
| `-(1 + n)` | **正在扩容**，`n` 表示参与扩容的线程数（如 `-3` 表示 2 个线程在协助扩容） |
| `> 0` | 初始化完成后的**扩容阈值**（= capacity * 0.75） |
| `小于 -1` 的其他负值 | 同上，`-1 - n` 表示 n 个线程 |

```java
// 初始化
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield();                             // 其他线程在初始化 → 让出 CPU
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {   // CAS 抢初始化权
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;    // 默认 16
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);                       // 0.75 * n = n - n/4
                }
            } finally { sizeCtl = sc; }                    // 释放：设为阈值
            break;
        }
    }
    return tab;
}
```

**注意 `sc = n - (n >>> 2)`**：这是 `n * 0.75` 的位运算写法，比 `(int)(n * 0.75f)` 精确且快（`0.75f` 浮点可能有精度问题）。

### 7.3.3 put 全流程（**JDK8 的核心**）

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();   // ①② 都不允许 null
    int hash = spread(key.hashCode());      // ③ 扩散：(h ^ (h >>> 16)) & HASH_BITS
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        // ④ 表未初始化 → 初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        // ⑤ 目标桶为空 → CAS 无锁插入（核心！）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                          // CAS 成功 → 完成
            // CAS 失败 → 自旋重试（别的线程抢先放了）
        }

        // ⑥ 桶首是 ForwardingNode → 正在扩容 → 参与协助迁移
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);

        // ⑦ 桶首是普通节点 → synchronized 锁桶首
        else {
            V oldVal = null;
            synchronized (f) {                  // ← 锁的是「桶首节点」对象
                if (tabAt(tab, i) == f) {       // 双重检查：桶首没变
                    if (fh >= 0) {              // 链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) e.val = value;   // 覆盖
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {         // 尾插
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {   // 红黑树桶
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)        // 8
                    treeifyBin(tab, i);
                if (oldVal != null) return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);      // ⑧ 计数 + 可能扩容
    return null;
}
```

**流程图（务必能画）**：

```
put(k,v)
  │ null 检查（k/v 都不能 null → NPE）
  │ hash = spread(k.hashCode())
  │
  ├─ table 未初始化 ──→ initTable()（CAS sizeCtl=-1 抢权，其他线程 yield）
  │
  ├─ tabAt(i) == null ──→ casTabAt(i, null, new Node) ──成功──→ break
  │                        └─ 失败 → 自旋重试
  │
  ├─ 桶首 hash == MOVED(-1) ──→ helpTransfer()（协助扩容）→ 重试
  │
  └─ 否则 ──→ synchronized(桶首 f) {
                双重检查 tabAt(i)==f
                ├─ 链表：遍历，命中覆盖 / 尾部插入，binCount++
                └─ TreeBin：putTreeVal（树插入）
              }
              ├─ binCount >= 8 → treeifyBin
              └─ addCount(1, binCount) → 可能触发 transfer
```

### 7.3.4 为什么 JDK8 放弃分段锁（**阿里 P7 必答**）

| 角度 | JDK7 分段锁 | JDK8 CAS + synchronized |
|---|---|---|
| 锁粒度 | Segment（默认 16 个，并发度上限 16） | **单个桶头节点**（并发度 ≈ 桶数量，可达数千） |
| 内存 | 每个 Segment 是一个完整小 HashMap，额外开销大 | 只有一个 Node 数组，无额外结构 |
| 读 | `get` 无锁（靠 `volatile value`） | `get` 无锁（靠 `volatile val`） |
| 空桶插入 | 必须锁 Segment | **CAS 无锁**（最高频场景零锁） |
| 扩容 | 单线程 `rehash`，全程锁 Segment | **多线程协助 `transfer`**，分段迁移 |
| 计数 | `count`（锁内）+ 三次重试 | `baseCount` + `CounterCell[]` 分散热点（**LongAdder 思路**） |
| 锁实现 | `ReentrantLock`（AQS，重） | **`synchronized`**（偏向锁/轻量级锁优化，JVM 内建，内存更省） |

**为什么 JDK8 用 `synchronized` 而不是 `ReentrantLock`？**
1. JDK6 之后 `synchronized` 有**偏向锁、轻量级锁、锁消除、锁粗化**等优化，在「短临界区、低竞争」场景性能不输 `ReentrantLock`；
2. `synchronized` 是 JVM 原语，**不需要额外的 AQS 对象**（每个 `ReentrantLock` 是一个对象 + `Sync` 内部类），在「数千万个桶」的场景能省下大量内存；
3. `synchronized` 的锁信息在**对象头 Mark Word** 里，天然「跟着被锁对象走」，而每个桶锁一个 `ReentrantLock` 需要额外分配对象；
4. 临界区非常短（只是遍历一个桶），不需要 `ReentrantLock` 的高级功能（可中断、超时、公平、多 Condition）。

> ⚠️ **易错点：JDK8 的 `synchronized (f)` 锁的是「桶首节点对象」**，不是「桶下标」。如果桶首节点被删除/替换，新来的线程会锁到新对象上 → 所以需要 `if (tabAt(tab, i) == f)` 双重检查。这是「**锁对象会变**」的经典处理。

### 7.3.5 get 为什么完全无锁

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {         // ① volatile 读数组元素
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;                            // ② 桶首命中
        }
        else if (eh < 0)                                 // ③ hash<0：树或 ForwardingNode
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {                   // ④ 链表遍历
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

**无锁的三个保证**：
1. `table` 是 `volatile` → 数组引用可见；
2. `Node.val` 和 `Node.next` 是 `volatile` → 节点内容可见；
3. `Unsafe.getObjectVolatile` / `tabAt` 是**volatile 语义的数组元素读**（普通 `tab[i]` 读不具备 volatile 语义，因为数组元素的 volatile 需要 `getObjectVolatile`）。

> 💡 **延伸：为什么数组元素的读要用 `Unsafe.getObjectVolatile` 而不是 `tab[i]`？**
> Java 中 `volatile` 只能修饰**引用本身**（`Node[] table`），不能修饰「数组的元素」。`tab[i]` 的读是普通读，可能读到**旧引用或未初始化完成的节点**（构造函数的 final 字段虽有保证，但 `next`/`val` 的可见性靠 volatile 字段）。所以 JUC 容器统一用 `Unsafe.getObjectVolatile(tab, offset)` 来获得「元素级 volatile 读」。

### 7.3.6 addCount 与扩容（多线程协助 transfer）

```java
private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;
    // ① 先尝试 CAS 更新 baseCount（无竞争时最快）
    if ((cs = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell c; long v; int m;
        boolean uncontended = true;
        // ② CAS 失败 → 用 CounterCell[] 分散（LongAdder 思路）
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended = U.compareAndSwapLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);      // ③ 还失败 → 自旋/扩容 cells
            return;
        }
        if (check <= 1) return;
        s = sumCount();
    }
    // ④ 检查是否要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
            if (sc < 0) {                      // 已有线程在扩容
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    (nt = nextTable) == null || transferIndex <= 0) break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))   // 加入协助
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2))
                transfer(tab, null);           // 第一个扩容线程
            s = sumCount();
        }
    }
}
```

**`transfer` 的分段迁移**：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 每个线程负责的桶数：至少 16 个，避免线程过多时竞争
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;      // 16
    ...
    while (advance) {
        int nextIndex, nextBound;
        if (--i >= bound || finishing) advance = false;
        else if ((nextIndex = transferIndex) <= 0) { i = -1; advance = false; }
        else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,
                 nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
            bound = nextBound; i = nextIndex - 1; advance = false;   // 领取 [bound, i] 区间
        }
    }
    // 逐个桶迁移，迁移完的桶放 ForwardingNode（hash=MOVED）
}
```

**`ForwardingNode` 的作用**：
- `hash = MOVED = -1`，`get`/`put` 遇到它会走 `e.find(h, key)` 或 `helpTransfer`；
- 它是一个「**迁移完成的墓碑**」，让其他线程知道「这个桶的数据已经搬到 `nextTable` 了，去新表找」。

### 7.3.7 扩容的触发与协助（关键数字）

| 常量 | 值 | 含义 |
|---|---|---|
| `DEFAULT_CAPACITY` | 16 | 默认初始容量 |
| `MAXIMUM_CAPACITY` | `1 << 30` | 最大容量 |
| `DEFAULT_CONCURRENCY_LEVEL` | 16 | 仅用于兼容旧 API（JDK8 已不用） |
| `TREEIFY_THRESHOLD` | 8 | 树化阈值 |
| `UNTREEIFY_THRESHOLD` | 6 | 退化阈值 |
| `MIN_TREEIFY_CAPACITY` | 64 | 树化最小表容量 |
| `MIN_TRANSFER_STRIDE` | 16 | 协助扩容时每线程最少处理 16 个桶 |
| `RESIZE_STAMP_BITS` | 16 | 扩容戳位数（用于区分「不同轮扩容」） |

**扩容戳（resizeStamp）的精妙设计**：

```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

- `numberOfLeadingZeros(n)`：n 是 2 的幂时，返回「前导零个数」，唯一标识 n（如 n=16 → 27，n=32 → 26）；
- 最高位置 1（`1 << 15`）：占位标记；
- 这样 `sizeCtl = resizeStamp(n) << 16 | (2 + 协助线程数)` 就同时编码了「**哪一轮扩容（n）**」和「**有多少线程参与**」，`sc == rs + MAX_RESIZERS` 判断「协助线程已满」，`sc == rs + 1` 判断「扩容结束」。这是**用一个 int 编码多个状态**的经典技巧。

### 7.3.8 计数器：为什么需要 CounterCell（**美团/字节高频**）

```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) for (CounterCell c : cs) if (c != null) sum += c.value;
    return sum;
}
```

**问题**：`size` 的更新是「每次 put/remove 都要 +1/-1」，如果所有线程 CAS 同一个 `baseCount` 变量，在 16 核上会形成**严重的热点竞争**（CAS 失败重试 → 性能骤降）。

**解决（LongAdder 思路）**：
1. 每个线程有自己的 `CounterCell`（通过 `ThreadLocalRandom.getProbe()` 散列到不同 cell）；
2. 更新时先 CAS `baseCount`，失败就 CAS 自己那个 cell；
3. 读 `size` 时把 `baseCount + 所有 cell` 求和（`sumCount`）。

**代价**：`size()` 是**弱一致估算**（求和过程中可能有并发更新），`mappingCount()` 返回 `long` 更精确（`size()` 返回 `int`，可能溢出）。

```
baseCount (volatile long)
CounterCell[0]  ← 线程 A 更新
CounterCell[1]  ← 线程 B 更新
CounterCell[2]  ← 线程 C 更新
...
size = baseCount + Σ CounterCell[i]
```

> ⚠️ **易错点：`counterCells` 长度是 2 的幂，最多 `NCPU` 个（`fullAddCount` 里限制 `cs.length >= NCPU` 就不再扩容）**。因为 cell 太多会导致 `sumCount` 遍历变慢，收益递减。

### 7.3.9 为什么 CHM 不允许 null（**字节必考**）

```java
if (key == null || value == null) throw new NullPointerException();
```

**三个层次的理由**：

1. **无法区分「不存在」和「值为 null」**：`get(key)` 返回 null 有两种可能——key 不存在，或 key 存在但 value 是 null。单线程下可以用 `containsKey` 二次确认，但**并发下 `containsKey` 和 `get` 之间可能有其他线程修改**，二次确认不可靠（TOCTOU 问题）。`HashMap` 允许 null 是因为它非并发，可以用 `containsKey` 确认。
2. **`get` 的无锁实现依赖「null 表示不存在」**：CHM 的 `get` 是「找到节点就返回 `e.val`，找不到就返回 null」，若允许 null value，就无法用 `null` 表示「未找到」，必须改用「返回 sentinel 节点」的复杂协议，破坏无锁读的简洁性。
3. **Doug Lea 的设计哲学**：并发容器应该「**fail-fast 且语义明确**」，宁可让调用方显式用 `Optional` 或哨兵对象表达 null，也不要引入「歧义 + 额外同步」。

**如果业务真的需要 null value**：用 `ConcurrentHashMap<K, Optional<V>>`，或用一个静态 `NULL` 哨兵对象，或改用 `Collections.synchronizedMap(new HashMap<>())`（全表锁，但允许 null）。

### 7.3.10 弱一致性迭代器

```java
public Iterator<K> iterator() { return new KeyIterator(); }
// KeyIterator 继承 BaseIterator，遍历时：
// - 不检查 modCount（没有 fail-fast）
// - 拿到的是「遍历开始时的 table 引用 + 每个桶的当前状态」
// - 可能看到「已删除的元素」（如果删除发生在遍历到它之前）
// - 可能看不到「新增的元素」（如果新增在已遍历的桶）
// - 但绝不会抛 ConcurrentModificationException
```

**为什么不 fail-fast？**
因为并发容器**天生会被并发修改**，抛异常会让「遍历 + 并发更新」这种正常用法不可用。所以设计成「**弱一致**」：遍历反映「某个时刻的近似快照」，不保证看到所有元素，但保证不崩。

---

## 7.4 ConcurrentSkipListMap：跳表的并发实现

```java
public class ConcurrentSkipListMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentNavigableMap<K,V>, Cloneable, Serializable {

    static final class Node<K,V> {
        final K key; volatile Object value; volatile Node<K,V> next;
    }
    static final class Index<K,V> {          // 索引层节点
        final Node<K,V> node;
        final Index<K,V> down;
        volatile Index<K,V> right;
    }
    private transient Index<K,V> head;       // 顶层索引头
}
```

### 7.4.1 为什么并发有序 Map 只能用跳表

| | TreeMap（红黑树） | ConcurrentSkipListMap（跳表） |
|---|---|---|
| 插入/删除 | 需要**旋转**，可能波及根，涉及多个指针 | 只改**局部**前驱后继，每层独立 CAS |
| 无锁化 | **几乎不可能**（旋转要原子改多指针 + 颜色） | **可行**（每层 `right` 用 CAS） |
| 范围查询 | 中序遍历 + 栈 | **第 0 层有序链表，直接顺序扫** |
| 实现 | 复杂（4 种删除情况） | 相对简单 |

**结论**：所有「需要并发 + 有序」的 JDK 容器都是跳表（`ConcurrentSkipListMap` / `ConcurrentSkipListSet`），因为跳表的**局部性**天然适配 CAS。

### 7.4.2 核心操作

```java
// 查找：从顶层索引开始，向右走，走不动就下沉
private Node<K,V> findNode(K key) {
    Node<K,V> b = head.node;  Index<K,V> p = head;
    for (;;) {
        Index<K,V> r = p.right;
        if (r != null) {
            Node<K,V> n = r.node;
            int c = cpr(cmp, key, n.key);
            if (c > 0) { p = r; continue; }        // 目标在右边 → 向右
            if (c == 0) return n;
        }
        Index<K,V> d = p.down;
        if (d == null) break;                      // 到底层了
        p = d;                                     // 下沉
    }
    // 在第 0 层链表上顺序找
}
```

**层数随机**：

```java
private static final int MAX_LEVEL = 31;   // 2^31 足够
// 随机层数：ThreadLocalRandom
int randomLevel() {
    int r = ThreadLocalRandom.nextSecondarySeed();
    int level = 1;
    while (((r >>>= 1) & 1) == 1 && level < MAX_LEVEL) level++;
    return level;
}
// 期望层数 = 2（p=0.5），实际上限 31，概率 2^-30，几乎不可能
```

> ⚠️ **注意**：JDK 的 `ConcurrentSkipListMap` 的 `size()` 也是**遍历 O(n)**（同 `ConcurrentLinkedQueue`），因为维护精确计数会成为瓶颈。

### 7.4.3 使用场景

- 需要**并发 + 有序**：排行榜（按分数排序 + 实时更新）、延时任务（按时间排序）、范围查询；
- `ConcurrentSkipListSet` 用于「并发去重 + 有序」；
- 替代 `TreeMap` 的并发场景（`Collections.synchronizedSortedMap(new TreeMap<>())` 性能差得多）。

> 💡 **延伸：Redis 的 ZSet 用跳表，为什么不用红黑树？**
> 除了并发（Redis 单线程，不需要无锁），还有三个原因：① **范围查询**（`ZRANGE`）跳表在第 0 层顺序扫，缓存友好；② **实现简单**，容易调试；③ **内存可控**（`p=0.25` 时平均 1.33 层指针，而红黑树每节点固定 3 指针 + 颜色）。Redis 作者 antirez 在源码注释里明确说：「跳表比平衡树更容易实现、更容易做范围查询」。

---

# 第 8 章 迭代器、fail-fast / fail-safe、工具类与经典坑

## 8.1 迭代器模式与 fail-fast 机制

### 8.1.1 为什么需要 modCount

```java
// AbstractList
protected transient int modCount = 0;    // 结构性修改次数

// Itr（ArrayList 的迭代器）
int expectedModCount = modCount;
final void checkForComodification() {
    if (modCount != expectedModCount) throw new ConcurrentModificationException();
}
```

**核心思想**：迭代器在创建时「记住」当时的 `modCount`，每次 `next()` 校验「容器结构有没有被改过」。如果改过，迭代器的 `cursor`、`lastRet` 可能已经指向错误的元素，继续遍历会得到**不可预期的结果**，所以**主动抛异常**（fail-fast）。

### 8.1.2 什么操作算「结构性修改」

| 操作 | 是否 modCount++ | 备注 |
|---|---|---|
| `add(e)` / `add(index, e)` | ✅ | |
| `remove(index)` / `remove(o)` | ✅ | |
| `clear()` | ✅ | |
| `trimToSize()` | ✅ | **注意！** 它只是缩容，但会改 `elementData` 引用 |
| `set(index, e)` | ❌ | 只改元素，不改结构 |
| `ensureCapacity()` | ✅（JDK8 中会） | 扩容改 `elementData` |
| `resize()`（HashMap） | ❌ | **HashMap 扩容不改 modCount** |
| `get()` / `contains()` | ❌ | |

> ⚠️ **`set()` 不抛 CME**，因为迭代器不关心「值变了」，只关心「结构变了」。

### 8.1.3 fail-fast 的三个「不算保证」的点

JDK 文档明确写：

> "The fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis."

翻译：**fail-fast 是「尽力而为」的检测机制，不是并发安全保证**。原因是：
1. `modCount` 是普通 `int`，**没有 volatile**，多线程下可能读到旧值；
2. 即使检测到，也可能「刚好错过」（如修改发生在 `hasNext` 和 `next` 之间）；
3. **不能用它来做并发正确性判断**。

### 8.1.4 fail-safe（弱一致）集合

| 集合 | 迭代器行为 |
|---|---|
| `CopyOnWriteArrayList` / `CopyOnWriteArraySet` | **快照迭代**：迭代的是「创建迭代器时的数组引用」，遍历期间的新增/删除不可见，**不抛 CME** |
| `ConcurrentHashMap` | **弱一致**：可能看到部分新数据，**不抛 CME** |
| `ConcurrentLinkedQueue` | 弱一致，不抛 CME |
| `ConcurrentSkipListMap` | 弱一致，不抛 CME |
| `Hashtable` 的 `Enumeration` | 不强一致，不抛 CME |

**「fail-safe」这个术语要慎用**：严格说「fail-safe」在 JDK 文档里并不存在（是民间叫法），更准确的表述是「**弱一致性迭代器（weakly consistent iterator）**」。面试时说出这个术语会加分。

## 8.2 集合工具类

### 8.2.1 Collections

| 方法 | 作用 | 注意 |
|---|---|---|
| `synchronizedList(list)` | 装饰成线程安全（锁 mutex） | **迭代必须手动同步**；复合操作（`check-then-act`）仍需外部锁 |
| `unmodifiableList(list)` | 只读视图 | **视图**，原 list 改动仍可见；`add` 抛 UOE |
| `singletonList(o)` | 单元素不可变 List | JDK 内部类，轻量 |
| `emptyList()` / `emptyMap()` / `emptySet()` | 空集合单例 | 泛型安全，比 `new ArrayList<>()` 省对象 |
| `sort(list)` / `sort(list, cmp)` | 稳定排序（TimSort） | 要求元素可比较；`List.sort` 是 JDK8 新增 |
| `binarySearch(list, key)` | 二分查找 | **必须先排序**；`RandomAccess` 时用索引，否则用迭代器 |
| `reverse(list)` | 反转 | |
| `shuffle(list)` | 随机打乱 | 内部用 `Random` |
| `max/min(list)` | 极值 | O(n) |
| `frequency(collection, o)` | 计数 | O(n) |
| `disjoint(a, b)` | 是否无交集 | |
| `addAll(collection, elements...)` | 批量添加 | |
| `newSetFromMap(map)` | 把 Map 包装成 Set | 少见但有用 |
| `checkedList(list, type)` | 运行时类型检查 | 泛型擦除的补救 |

**TimSort**：JDK7 引入的归并+插入混合排序，**稳定**，最好 O(n)、最坏 O(n log n)。JDK7 曾因 TimSort 的 `mergeLo/mergeHi` 实现 bug 导致 `ArrayIndexOutOfBoundsException`（2013 年的著名 bug），JDK8 已修复。

> ⚠️ **易错点：`Collections.sort` 的「稳定」意味着什么？**
> 相等元素的相对顺序不变。这对「先按 A 排序，再按 B 排序」的链式排序至关重要（第二次排序不会打乱第一次的结果）。

### 8.2.2 Arrays

| 方法 | 作用 | 坑 |
|---|---|---|
| `asList(T... a)` | 返回**固定大小**的 List | ① 不支持 `add/remove`（抛 UOE）；② 与原数组**共享存储**（`set` 会改原数组）；③ **传基本类型数组会当成一个元素**（`Arrays.asList(new int[]{1,2})` → size=1） |
| `sort(array)` | 双轴快排（DualPivotQuicksort） | 对基本类型用快排（不稳定），对对象用 TimSort（稳定） |
| `copyOf(arr, len)` | 复制/截断/扩容 | 常用于扩容 |
| `copyOfRange(arr, from, to)` | 区间复制 | `to` 可越界（补 null/0） |
| `fill(arr, val)` | 填充 | |
| `stream(array)` | 转流 | |
| `equals(a, b)` | 逐元素比较 | 不是 `Object.equals`（引用比较） |
| `hashCode(array)` | 内容哈希 | 与 `Object.hashCode` 不同 |

**`Arrays.asList` 的三个经典坑（**必背**）**：

```java
// 坑 1：基本类型数组
int[] nums = {1, 2, 3};
List list = Arrays.asList(nums);       // size == 1！元素是 int[] 本身
// 原因：asList 的签名是 asList(T... a)，int[] 不能展开成 Integer[]，只能作为一个 T

// 坑 2：固定大小
List<String> list = Arrays.asList("a", "b");
list.add("c");      // UnsupportedOperationException
list.remove("a");   // UnsupportedOperationException
list.set(0, "x");   // ✅ 可以（set 不改结构）

// 坑 3：共享存储
String[] arr = {"a", "b"};
List<String> list = Arrays.asList(arr);
list.set(0, "x");
System.out.println(arr[0]);    // "x" —— 原数组被改了！
```

**正确转换**：`new ArrayList<>(Arrays.asList(...))` 或 `List.of(...)`（不可变）或 `Arrays.stream(...).collect(Collectors.toList())`。

> 💡 **延伸：`List.of` 与 `Arrays.asList` 的区别**
> | | `Arrays.asList` | `List.of`（JDK9+） |
> |---|---|---|
> | 可变性 | 固定大小（可 `set`，不可 `add/remove`） | **完全不可变** |
> | null | 允许 | **不允许**（抛 NPE） |
> | 与原数组 | **共享** | 不共享（拷贝引用） |
> | 序列化 | 可序列化 | 可序列化（但反序列化后是不同实现） |
> | 实现 | `Arrays$ArrayList` | `ImmutableCollections$ListN` 等 |

## 8.3 集合使用中的 15 个经典坑（生产级）

| # | 坑 | 原因 | 正确做法 |
|---|---|---|---|
| 1 | `for-each` 里 `list.remove()` | CME（fail-fast） | `iterator.remove()` / `removeIf` |
| 2 | 自定义对象做 `HashSet` 元素不去重 | 没重写 `hashCode`/`equals` | 同时重写（或用 `record`） |
| 3 | 可变对象做 `HashMap` key | hash 变化 → 落在错误桶 | key 用不可变对象 |
| 4 | `Arrays.asList` 后 `add` | 固定大小 | `new ArrayList<>(...)` |
| 5 | `Arrays.asList(int[])` | 基本类型不展开 | `Arrays.stream(int[]).boxed()` |
| 6 | `subList` 后改原 list | 视图共享 + CME | 拷贝一份或一次性使用 |
| 7 | `List.of` 里放 null | 不可变集合禁 null | 用 `Optional` 或 `Arrays.asList` |
| 8 | `HashMap` 初始容量设成「元素数」 | 会触发扩容（阈值 = 0.75 倍） | 设 `元素数/0.75 + 1` 或 `Maps.capacity()` |
| 9 | `ConcurrentHashMap` 用 `get` + `put` 做「先检查再更新」 | 非原子（TOCTOU） | `putIfAbsent` / `computeIfAbsent` / `merge` |
| 10 | `LinkedBlockingQueue` 不指定容量 | 默认 `Integer.MAX_VALUE` → OOM | 显式指定容量 |
| 11 | `PriorityQueue` 里存可变对象 | 修改字段破坏堆序 | 入堆后不要改参与比较的字段 |
| 12 | `TreeMap` 的 `Comparator` 与 `equals` 不一致 | 误判相等，覆盖数据 | `Comparator` 必须与 `equals` 语义一致 |
| 13 | `ArrayList` 删元素不缩容 | `remove` 只置 null，`elementData` 仍大 | 大批量删除后 `trimToSize()` |
| 14 | `Collections.synchronizedList` 遍历不加锁 | 装饰器的迭代器不加锁 | `synchronized (list) { for (...) }` |
| 15 | `HashMap` 并发写 | JDK7 死循环 / JDK8 丢数据 | `ConcurrentHashMap` |

### 8.3.1 坑 9 的正确写法（**CHM 的原子复合操作**）

```java
// ❌ 错误：先检查再更新，并发下会重复创建
if (!map.containsKey(k)) map.put(k, new ExpensiveObject());

// ✅ 正确 1：computeIfAbsent（原子，且只创建一次）
map.computeIfAbsent(k, key -> new ExpensiveObject());

// ✅ 正确 2：putIfAbsent
map.putIfAbsent(k, new ExpensiveObject());   // ⚠️ value 仍会被创建（即使已存在）

// ✅ 正确 3：merge（原子更新）
map.merge(k, 1, Integer::sum);               // 计数场景的标准写法

// ✅ 正确 4：compute（原子重算）
map.compute(k, (key, old) -> old == null ? 1 : old + 1);
```

> ⚠️ **`computeIfAbsent` 的两个坑**：
> ① **不能在 mapping function 里修改同一个 map**（会死锁或抛 `IllegalStateException: Recursive update`）；
> ② `computeIfAbsent` 在 `HashMap` 上**不是线程安全**的，只有 `ConcurrentHashMap` 才原子。

### 8.3.2 坑 7 的延伸：`Map.of` 的限制

```java
Map.of("a", 1, "b", 2);       // 最多 10 个键值对（超过用 Map.ofEntries）
Map.of("a", null);            // ❌ NPE
Map.ofEntries(entry("a", 1), entry("b", 2));   // 任意数量
```

**为什么 `Map.of` 最多 10 对？** 因为 JDK 为每个数量（0~10）生成了**专用的不可变实现类**（`ImmutableCollections.Map1` ... `MapN`），避免用 `数组 + 循环` 的通用结构，从而获得更好的性能和更小的内存。超过 10 对用 `MapN`（内部是 `Object[] alternatingKeysAndValues`）。

---

# 第 9 章 横向对比总表与选型决策

## 9.1 全家族复杂度总表（**面试前必背**）

| 集合 | 底层结构 | `get` | `add` | `remove` | `contains` | 有序 | 线程安全 | null |
|---|---|---|---|---|---|---|---|---|
| **ArrayList** | 动态数组 | **O(1)** | 尾部均摊 O(1)，中间 O(n) | O(n) | O(n) | 插入序 | ❌ | ✅ |
| **LinkedList** | 双向链表 | O(n) | 头尾 O(1)，中间 O(n) | 头尾 O(1)，中间 O(n) | O(n) | 插入序 | ❌ | ✅ |
| **Vector** | 动态数组 | O(1) | 均摊 O(1) | O(n) | O(n) | 插入序 | ✅ | ✅ |
| **CopyOnWriteArrayList** | volatile 数组 | O(1) | **O(n)**（拷贝） | O(n) | O(n) | 插入序 | ✅ | ✅ |
| **HashMap** | 数组+链表/红黑树 | 平均 **O(1)**，最坏 O(log n) | 平均 O(1) | 平均 O(1) | 平均 O(1) | 无序 | ❌ | ✅ |
| **LinkedHashMap** | HashMap + 双链表 | O(1) | O(1) | O(1) | O(1) | **插入序/访问序** | ❌ | ✅ |
| **TreeMap** | 红黑树 | **O(log n)** | O(log n) | O(log n) | O(log n) | **排序** | ❌ | ❌ |
| **Hashtable** | 数组+链表 | O(1) | O(1) | O(1) | O(1) | 无序 | ✅（全表锁） | ❌ |
| **ConcurrentHashMap** | 数组+链表/树 + CAS | **O(1) 无锁** | O(1) | O(1) | O(1) | 无序 | ✅ | ❌ |
| **ConcurrentSkipListMap** | 跳表 | O(log n) | O(log n) | O(log n) | O(log n) | **排序** | ✅（无锁） | ❌ |
| **HashSet** | HashMap | — | O(1) | O(1) | O(1) | 无序 | ❌ | ✅（1 个） |
| **LinkedHashSet** | LinkedHashMap | — | O(1) | O(1) | O(1) | 插入序 | ❌ | ✅（1 个） |
| **TreeSet** | TreeMap | — | O(log n) | O(log n) | O(log n) | 排序 | ❌ | ❌ |
| **EnumSet** | long 位向量 | — | **O(1) 位运算** | O(1) | O(1) | 枚举序 | ❌ | ❌ |
| **ArrayDeque** | 循环数组 | 两端 O(1) | 两端均摊 O(1) | 两端 O(1) | O(n) | 双端 | ❌ | ❌ |
| **PriorityQueue** | 二叉堆（数组） | 堆顶 O(1) | O(log n) | O(log n)（堆顶） | O(n) | 堆序 | ❌ | ❌ |
| **ArrayBlockingQueue** | 数组 + 单锁 | — | O(1) | O(1) | O(n) | FIFO | ✅ | ❌ |
| **LinkedBlockingQueue** | 链表 + 双锁 | — | O(1) | O(1) | O(n) | FIFO | ✅ | ❌ |
| **ConcurrentLinkedQueue** | 无锁链表 | — | O(1) CAS | O(1) CAS | **O(n)** | FIFO | ✅ | ❌ |
| **SynchronousQueue** | 无容量 | — | 阻塞 | 阻塞 | — | — | ✅ | ❌ |

## 9.2 Map 家族全景对比

| 维度 | HashMap | LinkedHashMap | TreeMap | Hashtable | CHM | CSLM | WeakHashMap | IdentityHashMap | EnumMap |
|---|---|---|---|---|---|---|---|---|---|
| 顺序 | 无 | 插入/访问 | 排序 | 无 | 无 | 排序 | 无 | 无（但按 hash 布局） | 枚举序 |
| 线程安全 | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| null key | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| 查找 | O(1) | O(1) | O(log n) | O(1) | O(1) | O(log n) | O(1) | O(1) | **O(1) 数组下标** |
| 特殊能力 | 通用 | LRU | 范围/有序 | 遗留 | 高并发 | 并发有序 | **弱引用，GC 自动删** | **`==` 比较** | **最快、最省内存** |
| 典型场景 | 缓存、索引 | LRU、保序 | 排行榜、范围 | 遗留代码 | 并发缓存 | 并发排行榜 | 对象附加数据 | 序列化/图算法（需 identity） | 配置/状态机 |

**几个冷门但高价值的 Map（面试加分）**：

```java
// 1. EnumMap：内部就是一个 Object[]，下标 = 枚举 ordinal，最快
EnumMap<Status, String> m = new EnumMap<>(Status.class);

// 2. IdentityHashMap：用 System.identityHashCode + == 比较，不用 equals
//    适合「对象图遍历去重」（如序列化、深拷贝时避免循环引用）
IdentityHashMap<Object, Boolean> visited = new IdentityHashMap<>();

// 3. WeakHashMap：key 是弱引用，key 被 GC 后 Entry 自动移除
//    适合「给对象挂附加信息」，且不希望阻止对象回收
WeakHashMap<Object, Meta> cache = new WeakHashMap<>();

// 4. Properties：Hashtable<Object,Object> 子类，专用于配置文件（键值都是 String）
```

> ⚠️ **`WeakHashMap` 的坑**：如果 key 被**强引用**（比如在另一个集合里），就永远不会被回收，`WeakHashMap` 也不会清理。且 `WeakHashMap` 的 `size()` 是「估算」（要等 GC + expunge）。

## 9.3 选型决策树（**场景题必备**）

```
需要存「键值对」吗？
├── 是 → 需要线程安全吗？
│        ├── 否 → 需要顺序吗？
│        │        ├── 否 → 【HashMap】（默认，指定初始容量）
│        │        ├── 插入序 → 【LinkedHashMap】
│        │        └── 排序/范围 → 【TreeMap】
│        └── 是 → 需要顺序吗？
│                 ├── 否 → 【ConcurrentHashMap】
│                 └── 是 → 【ConcurrentSkipListMap】
│
└── 否 → 存「单个元素」，需要去重吗？
         ├── 是（Set）→ 需要线程安全吗？
         │              ├── 否 → 需要顺序吗？
         │              │       ├── 否 → 【HashSet】
         │              │       ├── 插入序 → 【LinkedHashSet】
         │              │       └── 排序 → 【TreeSet】
         │              └── 是 → 读多写极少？【CopyOnWriteArraySet】；需要有序？【ConcurrentSkipListSet】
         │
         └── 否 → 需要下标访问吗？
                  ├── 是 → 读多写少？【ArrayList】（默认）
                  └── 否 → 什么语义？
                           ├── 栈/队列/双端 → 【ArrayDeque】
                           ├── 优先级 → 【PriorityQueue】
                           ├── 生产者消费者 → 【ArrayBlockingQueue / LinkedBlockingQueue】
                           ├── 延时任务 → 【DelayQueue】
                           ├── 高并发无阻塞队列 → 【ConcurrentLinkedQueue】
                           └── 读多写极少（监听器列表）→ 【CopyOnWriteArrayList】
```

## 9.4 性能优化实践清单

### 9.4.1 预设容量（最有效的优化）

```java
// ❌ 默认 16，存 10000 个元素要扩容 10 次（10 → 15 → 22 → ... → 10000+）
List<Order> list = new ArrayList<>();

// ✅ 一次到位
List<Order> list = new ArrayList<>(10000);

// HashMap 要看 0.75
Map<String, Order> map = new HashMap<>(10000 / 0.75 + 1);   // ≈ 13334 → tableSizeFor → 16384
// 或用 Guava：Maps.newHashMapWithExpectedSize(10000)（内部算好）
```

### 9.4.2 遍历方式选择

| 集合 | 推荐 | 避免 |
|---|---|---|
| `ArrayList` | 下标 `for` 或 foreach | — |
| `LinkedList` | **foreach / iterator** | `for (i=0; i<size; i++) get(i)`（**O(n²)**） |
| `HashMap` | `entrySet()` | `keySet()` + `get(k)`（多一次查找） |
| `Set` | foreach | — |
| 大集合 + 需要过滤 | `removeIf` / `Stream`（注意 Stream 有额外开销） | 手写 `iterator.remove` |

```java
// LinkedList 的错误遍历（性能灾难）
for (int i = 0; i < list.size(); i++) list.get(i);   // O(n²)，10 万元素 ≈ 50 亿次指针跳转

// 正确
for (String s : list) { ... }
```

### 9.4.3 避免「装箱」开销

```java
// ❌ 每个 int 都装箱成 Integer（小值有缓存 -128~127，大值每次都 new）
Map<Integer, Integer> map = new HashMap<>();

// ✅ 用专用库（Eclipse Collections / fastutil / HPPC）
IntIntMap map = new IntIntHashMap();   // 无装箱，内存减半，快 2~5 倍
```

**`Integer` 缓存陷阱**：

```java
Integer a = 127, b = 127;
System.out.println(a == b);    // true（缓存）
Integer c = 128, d = 128;
System.out.println(c == d);    // false（超出缓存，各自 new）
// → 永远用 equals 比较包装类型
```

### 9.4.4 不可变集合的价值

```java
// 1. 线程安全（无需同步）
// 2. 可安全共享（不用防御性拷贝）
// 3. 可缓存 hashCode（List.of 的实现会缓存？实际上不会，但 String/record 会）
// 4. 语义明确（调用方知道不可改）

List<String> ROLES = List.of("ADMIN", "USER");       // JDK9+
Map<String, Integer> LEVELS = Map.of("L1", 1, "L2", 2);
Set<String> TAGS = Set.of("java", "juc");
```

> ⚠️ **`List.of` 的坑**：`List.of(arr)` 中 `arr` 是 `String[]` 时会**展开成多个元素**（因为可变参数）；而 `List.of(int[])` 会把 `int[]` 当成**一个元素**（因为 `int[]` 不是 `Object[]`，不能展开）。这与 `Arrays.asList` 行为一致，但更容易出错。

### 9.4.5 内存占用估算（64 位 JVM，压缩指针开启）

| 结构 | 每元素开销 | 说明 |
|---|---|---|
| `int[]` | 4B | 最紧凑 |
| `Integer[]` | 4B 引用 + 16B 对象（=20B） | 装箱代价 |
| `ArrayList` | 4B 引用 + 容量冗余（最多 50%） | 数组本身 |
| `LinkedList` | **24B/Node**（对象头 16 + item 4 + next 4 + prev 4，对齐） | 最贵 |
| `HashMap.Node` | 32B/Node（对象头 16 + hash 4 + key 4 + value 4 + next 4） | 加数组 4B/桶 |
| `HashMap.TreeNode` | 约 56B/Node | 树化后内存翻倍 |
| `HashSet` | 同 HashMap（value 共享 PRESENT） | |
| `ArrayDeque` | 4B 引用 + 1 个空槽 | 紧凑 |
| `PriorityQueue` | 4B 引用 + 数组 | 紧凑 |

> 💡 **延伸：为什么 `LinkedList` 的一个节点要 24B 而不是 12B？**
> 对象头 16B（Mark Word 8 + Klass Pointer 4 + 对齐 4？实际压缩指针下是 12B，对齐到 16B）+ item 4B + next 4B + prev 4B = 24B。若元素是 `Integer`，还要再加 16B 的 Integer 对象。所以「LinkedList 存 100 万个 Integer」约需 24MB 链表 + 16MB Integer = **40MB**，而 `ArrayList` 只需 4MB 数组 + 16MB Integer（若不在缓存范围）= 20MB。

---

# 第 10 章 大厂面试题库（60+ 题，带详解）

> **考核程度图例**：
> - ★★★★★ = 几乎必考，必须能默写源码级细节
> - ★★★★ = 高频，要求讲清原理 + 能对比
> - ★★★ = 中频，要求答对核心，细节可略
> - ★★ = 低频/深水区，答对加分
>
> 公司标注为「该题在哪些厂的面试中出现频率最高」，不代表只在这些厂考。

---

## A. HashMap 专题（15 题）

### A1. 讲讲 HashMap 的底层数据结构？★★★★★
**出现频率最高**：阿里、字节、腾讯、美团、京东

**参考作答（分四层说）**：

1. **整体**：`HashMap` 是「数组 + 链表 + 红黑树」的组合。数组叫 `table`（`Node<K,V>[]`），长度必须是 2 的幂；每个数组元素是一个「桶（bucket）」，桶里挂链表；当链表长度 ≥ 8 且数组长度 ≥ 64 时，链表转成红黑树。
2. **节点**：`Node` 有 `hash`、`key`、`value`、`next` 四个字段，`hash` 是**扰动后**的 hash（不是原始 `hashCode`）。
3. **定位**：`index = (n - 1) & hash(key)`，`hash(key) = (h = key.hashCode()) ^ (h >>> 16)`。
4. **扩容**：`size > threshold`（`threshold = capacity * loadFactor`，默认 0.75）时扩容为 2 倍，元素按 `(hash & oldCap)` 拆成「低位链」和「高位链」，分别留在原索引和 `原索引 + oldCap`。

**追问链**：
- 「为什么用红黑树不用 AVL？」→ 见 2.4.3。
- 「红黑树的节点是什么？」→ `TreeNode`，比 `Node` 多 `parent/left/right/prev/red`，且继承了 `LinkedHashMap.Entry`（保留 `next`，便于树化/退化和 `split`）。
- 「链表转树的阈值为什么是 8？」→ 见 4.8。

---

### A2. HashMap 的 put 过程？★★★★★
**出现频率最高**：所有大厂

**参考作答**（按 4.6 的 7 步说，必须提到「空桶直接放、桶首命中覆盖、树节点走 putTreeVal、链表遍历尾插、树化、size++ 超阈值扩容」）。

**加分点**：
- 指出 `onlyIfAbsent` 参数（`putIfAbsent` 复用 `putVal`）；
- 指出 `afterNodeAccess` / `afterNodeInsertion` 是给 `LinkedHashMap` 的钩子；
- 指出「命中已有 key 时**不增加 size**、不触发扩容，只 `return oldValue`」。

---

### A3. HashMap 的容量为什么必须是 2 的幂？★★★★★
**出现频率最高**：阿里、美团、字节

**参考作答**（三层，见 4.5）：
1. `(n-1) & hash` 等价于 `hash % n`，位运算远快于取模；
2. `n-1` 低位全 1，保证 hash 的每一位都参与索引，分布均匀；若 n 非 2 的幂（如 15），`n-1 = 14 = 0b1110`，最低位恒 0，一半桶浪费；
3. 扩容时可用 `(hash & oldCap)` 判断「新索引是原索引还是原索引 + oldCap」，无需重新 hash。

**追问**：「那 `new HashMap<>(15)` 的容量是多少？」→ 16（`tableSizeFor(15)` = 16）。「`new HashMap<>(17)`？」→ 32。

---

### A4. HashMap 的负载因子为什么是 0.75？★★★★
**出现频率最高**：阿里、美团

**参考作答**：
- `loadFactor` 是「空间」与「冲突率」的权衡。太小 → 空间浪费、扩容频繁；太大 → 冲突率上升，链表变长，查找退化。
- 0.75 是**经验值 + 数学支撑**：源码注释给出泊松分布（λ≈0.5）下，桶内元素数为 8 的概率约 `6e-8`。0.75 时，哈希表的「平均探测长度」约 2~3，仍在 O(1) 量级。
- 另外 0.75 = 3/4，`capacity * 0.75` 对 2 的幂容量可以用 `n - (n >>> 2)` 快速计算（`ConcurrentHashMap` 的 `initTable` 就是这么写的）。
- 可调：内存紧张可设 0.9（省空间，冲突多），追求速度可设 0.5（冲突少，费空间）。

---

### A5. HashMap 的 hash 扰动函数为什么这么设计？★★★★
**出现频率最高**：美团、字节

见 4.4。核心三点：只用到低位 → 高位信息浪费 → 用 `^` 把高 16 位混入低 16 位；异或是唯一「不偏向、可逆」的位运算；一次扰动就够了（不需要像 JDK7 那样多次）。

---

### A6. JDK7 和 JDK8 的 HashMap 有什么区别？★★★★★
**出现频率最高**：腾讯、阿里、字节、快手

**参考作答（五条对照）**：

| 维度 | JDK7 | JDK8 |
|---|---|---|
| 结构 | 数组 + 链表 | 数组 + 链表 + **红黑树** |
| 插入 | **头插** | **尾插** |
| 扩容迁移 | `transfer` 逐个重新计算索引 + 头插（**链序反转**） | 高低位拆分（`hash & oldCap`），**保持原序** |
| hash | 需要 `hash()` 多次扰动（`h ^= h>>>7; h ^= h>>>17...`） | 一次 `h ^ (h>>>16)` |
| 并发 | **可能形成环 → 死循环** | 不会成环，但会丢数据 |
| 扩容条件 | `size >= threshold && table[i] != null`（先判断再插） | `++size > threshold` |

**追问**：「尾插有什么代价？」→ 必须遍历到链尾，插入从 O(1) 变 O(链长)；但配合树化，长链被红黑树兜底，且解决了成环问题。

---

### A7. JDK7 的 HashMap 并发为什么会死循环？★★★★★
**出现频率最高**：阿里（P6+）、京东

**参考作答**（按 4.9.2 的五步推演，重点说清「T1 记住 next=B → T2 完成 transfer 使 B.next=A → T1 恢复后 A.next=B，形成 A⇄B 环 → get 时 while 永不结束 → CPU 100%」）。

**必须提到的关键词**：头插法、`transfer`、`Entry next = e.next` 被挂起、环形链表、`get` 的 `while (e != null)` 死循环。

**追问**：「JDK8 修复了吗？」→ 尾插 + 高低位拆分，不会成环，但仍有 `size` 不准、覆盖丢失、扩容与 put 竞争等问题，所以仍然线程不安全。

---

### A8. HashMap 的扩容过程？扩容后元素位置怎么变？★★★★★
**出现频率最高**：字节、阿里、美团

**参考作答**：
1. 新容量 = 旧容量 × 2，新阈值 = 旧阈值 × 2（若旧容量 ≥ 16）；
2. 遍历旧表的每个桶：
   - 单节点：直接 `newTab[e.hash & (newCap-1)] = e`；
   - 红黑树：`split`，按 `hash & oldCap` 拆成两条链，≤6 个则退化成链表；
   - 链表：用 `loHead/loTail` 和 `hiHead/hiTail` 拆成两条链，**保持原相对顺序**（尾插），低位链放 `newTab[j]`，高位链放 `newTab[j + oldCap]`。
3. **元素位置只有两种可能**：留在原索引，或移到「原索引 + oldCap」，因为 `n` 翻倍只多用了 hash 的一位。

**加分**：说出 `(e.hash & oldCap) == 0` 这个判断的含义（`oldCap` 的二进制只有一位 1，恰好是「新索引多出来的那一位」）。

---

### A9. HashMap 的 size 和 capacity 有什么区别？table 的 length 和 size 呢？★★★★
**出现频率最高**：美团、滴滴

- `size`：KV 对的数量；
- `capacity`：`table.length`，桶的数量；
- `threshold = capacity * loadFactor`；
- 关系：`size <= threshold <= capacity`，`size` 永远 ≤ `capacity`（每个桶至少能放 1 个）。

---

### A10. HashMap 允许 null key 吗？放在哪个桶？★★★★
**出现频率最高**：京东、小米

- 允许（`HashMap`）；`Hashtable`/`ConcurrentHashMap` 不允许。
- `hash(null)` 返回 **0**，所以 `null` key 永远放在 **0 号桶**。
- `putVal` 里 `(k = p.key) == key || (key != null && key.equals(k))` 的 `key != null` 判断就是为了避免 null key 调 `equals` 时 NPE。

**追问**：「一个 HashMap 能存几个 null key？」→ **1 个**（key 唯一）；「null value 呢？」→ 任意多个。

---

### A11. 为什么重写 equals 一定要重写 hashCode？★★★★★
**出现频率最高**：所有大厂（基础题）

见 4.11。**核心一句话**：「`hashCode` 决定落在哪个桶，`equals` 决定桶内是否重复。只重写 equals → 两个『相等』的对象落在不同桶 → `get` 找不到、`HashSet` 去重失败」。

**追问**：「只重写 hashCode 不重写 equals 呢？」→ 落在同一个桶，但 `equals` 用引用比较 → 都认为不相等 → 桶内堆积，**性能退化但不丢数据**。

---

### A12. 为什么 String 适合做 HashMap 的 key？★★★★
**出现频率最高**：阿里、美团

1. **不可变**：`hashCode` 不会变，不会「放进去后找不到」；
2. **`hashCode` 缓存**：`String` 内部 `private int hash` 缓存，第二次调用 O(1)；
3. **`equals` 高效**：先比长度/引用，再逐字符比较；
4. **`hashCode` 分布好**：31 乘子 + 字符累加，碰撞率低。

**追问**：「String 的 hashCode 有什么安全风险？」→ 可预测 → **Hash 碰撞 DoS**（构造大量同 hash 的字符串）→ JDK 8u 之后对 `String` 有额外的 `hash32` 随机化备用哈希。

---

### A13. HashMap 的遍历方式有几种？哪种最快？★★★
**出现频率最高**：京东、滴滴

见 4.12。**结论**：`entrySet()` + foreach **最快**；`keySet()` + `get(k)` 慢一倍（每次 get 要重新定位）；`values()` 只要 value 时最快。

---

### A14. HashMap 线程不安全的表现有哪些？★★★★★
**出现频率最高**：拼多多、字节、阿里

见 4.9.3 / 7.1。四条：
1. JDK7 扩容成环 → 死循环；
2. 并发 put 同桶空位 → **覆盖丢失**；
3. `++size` 非原子 → size 偏小 → 阈值判断错；
4. 并发 put + resize → 节点挂到旧表，**永久丢失**；
5. 遍历时其他线程修改 → CME（或弱一致读到脏数据）。

---

### A15. 如何让 HashMap 变线程安全？★★★★
**出现频率最高**：阿里、腾讯

| 方案 | 实现 | 并发度 | 推荐 |
|---|---|---|---|
| `Hashtable` | 全表 synchronized | 1 | ❌ 遗留 |
| `Collections.synchronizedMap` | 装饰器 + mutex | 1 | 只在「元素极少、几乎无并发」时 |
| **`ConcurrentHashMap`** | CAS + 桶锁 | ≈ 桶数 | ✅ **首选** |
| 自己加锁（`ReentrantReadWriteLock`） | 读写分离 | 读并发 | 特定场景 |

> ⚠️ 注意 `Collections.synchronizedMap` 的**复合操作仍需外部同步**（`if (!m.containsKey(k)) m.put(k,v)` 不是原子的）。

---

## B. ConcurrentHashMap 专题（10 题）

### B1. ConcurrentHashMap 的 JDK7 和 JDK8 有什么区别？★★★★★
**出现频率最高**：阿里、字节、美团、快手

**参考作答（六条）**：

| 维度 | JDK7 | JDK8 |
|---|---|---|
| 结构 | `Segment[]`（每个 Segment 继承 ReentrantLock，内含 HashEntry[]） | `Node[]` + 链表/红黑树（与 HashMap 类似） |
| 锁粒度 | Segment（默认 16，并发度上限 16） | **桶头节点**（并发度 ≈ 桶数） |
| 锁实现 | `ReentrantLock` | **`synchronized` + CAS** |
| 空桶插入 | 必须锁 Segment | **CAS 无锁** |
| 计数 | `count` + `modCount` 三次重试 | `baseCount` + `CounterCell[]`（LongAdder 思路） |
| 扩容 | 单线程 `rehash`（锁 Segment） | **多线程协助 `transfer`** + ForwardingNode |
| hash | 多次扰动 | `spread(h) = (h ^ (h>>>16)) & HASH_BITS`（保证正数） |

**追问**：「为什么放弃分段锁？」→ 见 7.3.4（并发度、内存、CAS 空桶、协助扩容、synchronized 优化）。

---

### B2. CHM 的 get 为什么不需要加锁？★★★★★
**出现频率最高**：字节、阿里

见 7.3.5。三个保证：`table` 是 volatile、`Node.val`/`next` 是 volatile、`tabAt` 用 `Unsafe.getObjectVolatile`。

**追问**：「volatile 能保证数组元素可见吗？」→ **不能**！`volatile` 只修饰引用本身，数组元素要用 `Unsafe.getObjectVolatile` 或 `VarHandle` 才有元素级 volatile 语义。这是 JUC 容器的通用做法。

---

### B3. CHM 的 put 过程？为什么用 synchronized 而不是 ReentrantLock？★★★★★
**出现频率最高**：阿里、字节、美团

见 7.3.3 + 7.3.4。**核心四点**：
1. 空桶 CAS 无锁插入（最高频路径）；
2. 非空桶 `synchronized(桶首)` + 双重检查；
3. `ForwardingNode` 触发协助扩容；
4. `synchronized` 比 `ReentrantLock` 省内存（无需额外 AQS 对象）+ JDK6 后的锁优化（偏向锁/轻量级锁）。

---

### B4. CHM 的 size 是怎么实现的？为什么不准？★★★★
**出现频率最高**：美团、字节

见 7.3.8。
- `baseCount`（volatile long）+ `CounterCell[]`（`@Contended` 防伪共享）；
- `put/remove` 先 CAS `baseCount`，失败则 CAS 当前线程对应的 cell；
- `size()` = `sumCount()`（遍历求和），是**估算值**；
- 原因：求和过程中可能有并发更新；且为了性能不愿对计数加锁。
- `mappingCount()` 返回 `long`，避免 `int` 溢出，**推荐用 `mappingCount`**。

**追问**：「为什么不能像 JDK7 那样『重试 3 次 + 锁全表』？」→ JDK8 追求「完全无锁的高并发」，锁全表会让 size 成为瓶颈（尤其在高频 put 场景）。

---

### B5. CHM 的扩容是怎么做的？多线程如何协助？★★★★
**出现频率最高**：阿里、字节

见 7.3.6。
1. 触发：`addCount` 里 `s >= sizeCtl` 时，CAS 把 `sizeCtl` 设为 `resizeStamp(n) << 16 | 2`，第一个线程开始 `transfer`；
2. 协助：其他线程 put 时遇到 `ForwardingNode`（hash = MOVED = -1）→ `helpTransfer` → CAS 把 `sizeCtl` 加 1（表示多一个线程）→ 加入 `transfer`；
3. 分工：`transferIndex` 从高位往低位分配，每个线程领 `stride = max(16, n/8/NCPU)` 个桶；
4. 迁移完成的桶放 `ForwardingNode`，`get` 遇到它去 `nextTable` 找；
5. 全部迁完，最后一个线程把 `table = nextTable`，`sizeCtl = 新阈值`。

**加分**：`resizeStamp(n)` 用 `Integer.numberOfLeadingZeros(n)` 编码「哪一轮扩容」，防止「不同轮扩容的线程互相干扰」。

---

### B6. CHM 为什么不允许 null？★★★★
**出现频率最高**：字节、阿里

见 7.3.9。三层理由：无法区分「不存在」与「null 值」+ 并发下 `containsKey` 二次确认不可靠（TOCTOU）+ 破坏无锁 get 的「null 表示不存在」协议。

---

### B7. CHM 的迭代器是 fail-fast 吗？★★★★
**出现频率最高**：美团、京东

**不是**。是**弱一致（weakly consistent）**：遍历时基于「遍历开始时的 table 引用 + 每个桶的当前状态」，可能看到旧数据/看不到新数据，但**绝不抛 CME**。

**追问**：「那怎么保证遍历安全？」→ 不需要保证「一致性」，只需要保证「不崩溃 + 每个元素最多被访问一次」。这是并发容器的设计取舍。

---

### B8. CHM 的 key 和 value 哪个是 volatile？为什么？★★★★
**出现频率最高**：阿里（P7）

- `Node.key` 是 **final**（不可变，不需要 volatile）；
- `Node.val` 是 **volatile**（可能被其他线程覆盖，`get` 要看到最新值）；
- `Node.next` 是 **volatile**（链表结构可能变化，`get` 遍历要看到最新后继）。
- `Node.hash` 是 `final`。

**追问**：「为什么 `key` 不用 volatile？」→ key 一旦放入就不允许修改（`final`），所以不需要可见性保证；`val` 会被 `put` 覆盖，必须 volatile。

---

### B9. CHM 的 `computeIfAbsent` 为什么不能用递归调用？★★★★
**出现频率最高**：字节

因为 `computeIfAbsent` 在 `ConcurrentHashMap` 里会**锁住目标桶**（`synchronized(f)`），如果 mapping function 内部又对**同一个桶**做 `put/computeIfAbsent`，会**同一线程重入同一把 `synchronized` 锁**（可重入，不死锁），但会导致**结构不一致或无限递归**，JDK 会检测并抛 `IllegalStateException: Recursive update`（源码里有 `ReservationNode` 标记）。

**正确做法**：在 mapping function 里只做「纯计算」，不要操作同一个 map。

---

### B10. 高并发计数场景，CHM 和 LongAdder 怎么选？★★★★
**出现频率最高**：阿里、美团

| | `ConcurrentHashMap` | `LongAdder` |
|---|---|---|
| 用途 | 需要「按 key 计数」（多维度） | 单一计数器 |
| 实现 | `baseCount + CounterCell`（同 LongAdder 思路） | `base + Cell[]` |
| 读取 | `size()` 是估算 | `sum()` 是估算 |
| 内存 | 每个 key 一个 Node | 只有 cells |
| 推荐 | `map.merge(k, 1L, Long::sum)` | 单指标统计 |

> 💡 **`LongAdder` 为什么比 `AtomicLong` 快？** `AtomicLong` 所有线程 CAS 同一个变量（热点），`LongAdder` 分散到多个 `Cell`（类似分段），求和时才合并。高并发下吞吐可高 5~10 倍。

---

## C. ArrayList / LinkedList 专题（10 题）

### C1. ArrayList 的扩容机制？为什么是 1.5 倍？★★★★★
**出现频率最高**：阿里、美团、字节、京东

见 3.1.3 + 3.1.4。**必须说清**：
- 默认 10（懒加载，首次 add 才分配）；
- `newCapacity = oldCapacity + (oldCapacity >> 1)`；
- 用 `Arrays.copyOf`（底层 `System.arraycopy`）；
- 1.5 倍的三个理由：**空间利用率**（2 倍浪费 50%）、**内存复用**（旧块能被后续请求复用）、**避免峰值 OOM**（2 倍需要「旧 + 新」= 3 倍临时空间）。

---

### C2. `new ArrayList<>()` 的初始容量是多少？什么时候分配数组？★★★★★
**出现频率最高**：字节、阿里

- **构造时容量是 0**（`elementData` 指向共享的 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`）；
- **第一次 add 时才分配 10**（懒加载）；
- `new ArrayList<>(0)` 与 `new ArrayList<>()` 的区别：前者用 `EMPTY_ELEMENTDATA`，首次扩容是 **1**；后者首次是 **10**。这是两个不同哨兵数组的意义。

**追问**：「为什么要懒加载？」→ ① 大量「创建了但没放元素」的 ArrayList 不占 10 个槽的内存；② 与 `new ArrayList<>(0)` 区分语义。

---

### C3. ArrayList 和 LinkedList 的区别？怎么选？★★★★★
**出现频率最高**：所有大厂

见 3.2.5。**必答金句**：「LinkedList 的 O(1) 插入只在『已知节点』时成立，而 Java 不暴露节点，外部定位仍要 O(n)，加上 24B/节点的内存和缓存不友好，**实际几乎总是劣于 ArrayList**。LinkedList 的正确用途是 **Deque**（`offerFirst/pollLast`），不是 List。」

---

### C4. ArrayList 的 fail-fast 是怎么实现的？怎么安全删除？★★★★★
**出现频率最高**：腾讯、美团、阿里

见 3.1.7。**必须能说出**：
- `modCount`（继承自 `AbstractList`）+ `expectedModCount`；
- `next()` 里 `checkForComodification()`；
- 正确删除：`iterator.remove()`（会同步 `expectedModCount`）/ `removeIf` / 倒序下标；
- 反例：foreach 里 `list.remove()` → **第一次 `next()` 就抛 CME**。

**追问**：「JDK8 的 `removeIf` 为什么快？」→ 双指针 + BitSet 批量前移，避免每个元素都 `arraycopy` 一次（O(n²) → O(n)）。

---

### C5. ArrayList 为什么用 transient 修饰 elementData？如何序列化？★★★★
**出现频率最高**：美团、字节

见 3.1.6。核心：**capacity 可能远大于 size，序列化 null 槽会体积爆炸**；自定义 `writeObject/readObject` 只写 `size` 个真实元素；且 `writeObject` 末尾校验 `modCount` 保证快照一致。

---

### C6. ArrayList 的 `remove` 会把内存还给操作系统吗？★★★★
**出现频率最高**：阿里（P7）

**不会**。`remove` 只是 `System.arraycopy` 前移 + `elementData[--size] = null`（帮助 GC 回收**元素对象**），`elementData` 数组本身的长度不变（不缩容）。要真正释放数组空间需 `trimToSize()`。

**追问**：「为什么不自动缩容？」→ 缩容要拷贝 O(n)，且「删了又加」的场景会反复缩容/扩容，得不偿失。JDK 选择把决策权交给调用方（`trimToSize`）。

---

### C7. `ArrayList.subList()` 返回的是视图还是拷贝？有什么坑？★★★★
**出现频率最高**：美团、京东

- **视图**（`SubList` 内部类），与原 list 共享 `elementData`；
- 改 subList 会影响原 list，反之亦然；
- **原 list 结构性修改后访问 subList → CME**（`SubList` 也检查 `modCount`）；
- `subList` 的 `clear()` 会**删除原 list 的对应区间**（这是很多人误以为「清空视图」的坑）；
- 需要独立副本：`new ArrayList<>(list.subList(a, b))`。

---

### C8. ArrayList 和数组的转换有哪些坑？★★★★
**出现频率最高**：京东、滴滴

```java
// 1. 数组 → List
String[] arr = {"a", "b"};
List<String> fixed = Arrays.asList(arr);        // 固定大小，共享存储
List<String> mutable = new ArrayList<>(Arrays.asList(arr));   // ✅ 独立可变
List<String> immutable = List.of(arr);          // ✅ 不可变

// 2. List → 数组
Object[] oa = list.toArray();                   // 返回 Object[]
String[] sa = list.toArray(new String[0]);      // ✅ 类型安全（JDK11 推荐 new String[0]）
// list.toArray(new String[list.size()]) 也常见，但 JDK11 后 new String[0] 更优（JVM 会优化）

// 3. 基本类型数组的坑
int[] nums = {1, 2, 3};
Arrays.asList(nums).size();     // 1 ！int[] 被当成一个元素
```

---

### C9. 如何用 ArrayList 实现一个「栈」？为什么不推荐？★★★
**出现频率最高**：京东、小米

- `add(e)` 入栈，`remove(size-1)` 出栈（O(1)）；
- 不推荐：API 语义不清晰（可以任意 index 插入）、`remove(size-1)` 要检查空、没有 `peek` 的便捷语义；
- 正确：`ArrayDeque`（`push/pop/peek`），或 `Deque` 接口。

---

### C10. `Vector` 和 `ArrayList` 的区别？为什么 `Vector` 被淘汰？★★★
**出现频率最高**：腾讯、百度

见 3.3。核心：`Vector` 全表 `synchronized`（并发度 1）、2 倍扩容（浪费）、`Stack` 继承它导致语义污染。替代：`CopyOnWriteArrayList`（读多写少）/ `ArrayDeque`（栈）。

---

## D. Set / Queue 专题（10 题）

### D1. HashSet 的原理？为什么 value 是 PRESENT？★★★★★
**出现频率最高**：所有大厂

见 6.1.1。**核心**：`HashSet` = `HashMap<E, Object>` + 静态哨兵 `PRESENT`。`add` 返回 `map.put(e, PRESENT) == null`（put 返回 null 表示「原来没有」）。`PRESENT` 是**所有 Entry 共享的同一个对象**，省内存。

**追问**：「HashSet 怎么判断重复？」→ `hashCode` 定位桶 + `equals` 桶内比较，**两者都必须正确**。

---

### D2. LinkedHashSet 是怎么保证顺序的？★★★★
**出现频率最高**：美团、字节

`HashSet` 有一个**包可见**的三参构造 `HashSet(int, float, boolean dummy)`，`LinkedHashSet` 通过它把内部的 `HashMap` 换成 `LinkedHashMap`（`accessOrder = false`，即**插入序**）。`LinkedHashMap` 用 `before/after` 双向链表维护顺序。

---

### D3. TreeSet 和 HashSet 的区别？为什么 TreeSet 不允许 null？★★★★
**出现频率最高**：京东、百度

- 底层：`TreeMap`（红黑树）vs `HashMap`；
- 顺序：排序 vs 无序；
- 复杂度：O(log n) vs O(1)；
- null：**`TreeSet` 不允许**，因为 `TreeMap.put` 里 `key.compareTo(...)` 会 NPE（`Comparable` 无法处理 null）；
- 要求：元素必须实现 `Comparable` 或传 `Comparator`。

---

### D4. 为什么说 ArrayDeque 比 Stack 和 LinkedList 都好？★★★★
**出现频率最高**：京东、阿里、美团

见 6.2.1。**四个理由**：
1. 无 `synchronized`（`Stack` 全表锁）；
2. 循环数组，**内存紧凑 + 缓存友好**（`LinkedList` 24B/节点）；
3. 头尾操作 O(1) 均摊（`Stack` 的头插是尾部追加，但 `Vector` 有同步开销）；
4. 位运算 `(i-1) & (len-1)` 实现环形，比取模快。

**追问**：「ArrayDeque 怎么区分空和满？」→ `head == tail` 表示空；**数组始终留一个空槽**，`tail` 是「下一个写入位置」，当 `(tail+1) & (len-1) == head` 时判定为满并扩容。

---

### D5. PriorityQueue 的底层？建堆为什么是 O(n)？★★★★
**出现频率最高**：字节、快手、美团

见 6.2.2。**要点**：
- 二叉小顶堆，数组存储，`queue[i]` 的父是 `(i-1)>>>1`，孩子是 `(i<<1)+1/2`；
- `offer` = 尾部追加 + `siftUp`（O(log n)）；
- `poll` = 取 `queue[0]`，末尾元素放根 + `siftDown`（O(log n)）；
- **建堆 O(n)**：从 `n/2 - 1` 开始向前 `siftDown`，深度为 h 的节点有 n/2^(h+1) 个，求和 Σ(n/2^(h+1) * h) = O(n)；
- 迭代器**不保证有序**（只有反复 poll 才有序）；
- 不允许 null。

---

### D6. 求 TopK 用什么数据结构？为什么？★★★★★
**出现频率最高**：字节、阿里、美团、快手

见 6.2.2。**核心**：「求最大 K 个 → **小顶堆**（容量 K）；求最小 K 个 → **大顶堆**」。时间 O(n log k)，空间 O(k)。比全排序 O(n log n) 更适合海量/流式数据。

**追问**：「1 亿个数找 Top 100，内存放不下怎么办？」→ **分治 + 堆**：分成若干块，每块求 Top 100（小顶堆），再对「所有块的 Top 100」做归并/再建堆。或者用「哈希分片 + 每片 Top K + 全局归并」。

---

### D7. ArrayBlockingQueue 和 LinkedBlockingQueue 的区别？★★★★★
**出现频率最高**：拼多多、字节、阿里

见 6.2.3。**核心对比**：单锁 vs 双锁（读写能否并行）、数组 vs 链表（内存/GC）、`int count` vs `AtomicInteger`、**有界 vs 默认无界**。

**追问**：「为什么 `LinkedBlockingQueue` 能双锁？」→ 链表头尾操作互不干扰（`head` 只被 take 改，`last` 只被 put 改），`count` 用原子类保证。

**追问**：「默认容量是多少？」→ `Integer.MAX_VALUE`，**生产环境必须显式指定**，否则 OOM。

---

### D8. SynchronousQueue 的原理？用在哪里？★★★★
**出现频率最高**：字节、阿里

- **无容量**：`put` 必须等到有 `take`（或反之），直接「手递手」移交；
- 公平模式用 `TransferQueue`（FIFO 队列），非公平用 `TransferStack`（LIFO 栈）；
- `Executors.newCachedThreadPool()` 用它：任务来了若有空闲线程直接移交，否则新建线程；
- 适合「生产与消费速度匹配、不想缓冲」的场景。

---

### D9. DelayQueue 的原理？怎么做订单超时取消？★★★★
**出现频率最高**：美团、阿里、京东

- `PriorityQueue<Delayed>` + `ReentrantLock` + `available Condition` + **`leader` 线程优化**；
- 元素必须实现 `Delayed`（`getDelay` + `compareTo`）；
- `take()` 时看堆顶的 `getDelay`，未到期则 `awaitNanos(delay)`；
- **`leader` 优化**：多个消费者中只有一个 `leader` 定时等待，其他无限等待，避免「惊群」；
- 订单超时：下单时 `offer(order, 30min)`，一个消费线程 `take()` 到期元素 → 检查订单状态 → 取消。

**追问**：「DelayQueue 是无界的吗？会不会 OOM？」→ 是，`Integer.MAX_VALUE`，需要业务层限流。

---

### D10. 线程池的队列怎么选？为什么阿里禁止用 Executors？★★★★★
**出现频率最高**：所有大厂（阿里规约）

见 6.2.3 末尾。**四个工厂方法的队列**：`Fixed/Single` → `LinkedBlockingQueue`（无界）、`Cached` → `SynchronousQueue`（可能线程爆炸）、`Scheduled` → `DelayedWorkQueue`。

**阿里规约原文**：「线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。」

---

## E. 综合 / 场景 / 手撕题（15 题）

### E1. 手写一个线程安全的 LRU 缓存 ★★★★★
**出现频率最高**：字节、阿里、腾讯、美团

见 5.1.3。**两种写法**：`LinkedHashMap(accessOrder=true)` + `removeEldestEntry`（最简）；或 `HashMap + 双向链表 + 哨兵`（手撕标准答案，必须能 15 分钟写完）。

**追问**：「怎么保证线程安全？」→ ① `synchronized` 所有方法（简单但吞吐低）；② `ReentrantReadWriteLock`（读并发，写互斥）；③ **`ConcurrentHashMap` 存节点 + 全局双向链表 + 分段锁**（工业级，如 Caffeine 的思路）。

**追问**：「`LinkedHashMap` 的 `accessOrder=true` 有什么坑？」→ `get` 会改 `modCount` → **遍历中 `get` 会抛 CME**；且并发读也是写操作，**不能并发**。

---

### E2. 手写一个 HashMap（拉链法）★★★★
**出现频率最高**：字节、阿里

```java
class MyHashMap<K, V> {
    static class Node<K, V> { int hash; K key; V value; Node<K, V> next; ... }
    private Node<K, V>[] table = new Node[16];
    private int size, threshold = 12;

    public V put(K key, V value) {
        int h = hash(key);
        int i = (table.length - 1) & h;
        for (Node<K, V> e = table[i]; e != null; e = e.next) {
            if (e.hash == h && (e.key == key || key.equals(e.key))) {
                V old = e.value; e.value = value; return old;
            }
        }
        table[i] = new Node<>(h, key, value, table[i]);   // 头插（演示用）
        if (++size > threshold) resize();
        return null;
    }
    private int hash(Object key) {
        int h = (key == null) ? 0 : key.hashCode();
        return h ^ (h >>> 16);
    }
    private void resize() { /* 2 倍 + rehash 所有节点 */ }
}
```
**面试官会追问的点**：hash 扰动、索引计算、扩容时 rehash、`equals` 与 `hashCode` 的顺序、null key 处理。

---

### E3. 海量数据去重，内存只有 1GB，怎么办？★★★★★
**出现频率最高**：字节、阿里、美团

**分层回答**：

| 数据规模 | 方案 |
|---|---|
| 几百万（能放进内存） | `HashSet` / `ConcurrentHashMap.newKeySet()` |
| 上亿、内存不够 | **布隆过滤器（BloomFilter）**：可能有误判（假阳性），但绝不漏判；Guava `BloomFilter` / Redis `BF.ADD` |
| 需要精确去重 | **分治**：按 hash 分片到多个文件，每片单独 `HashSet` 去重；或**外部排序 + 归并** |
| 超大规模、允许极小误差 | **HyperLogLog**（基数估计，误差 ~0.81%，Redis `PFADD`） |
| 需要精确 + 持久化 | **Redis Set / RocksDB** |

**追问**：「布隆过滤器的原理？」→ 多个 hash 函数 + 位数组；插入时把 k 个位置置 1；查询时若有一位为 0 则**一定不存在**，全为 1 则**可能存在**（假阳性率与位数组大小、hash 个数、元素数相关：`p ≈ (1 - e^(-kn/m))^k`）。

---

### E4. 两个大文件（各 10 亿 URL）找共同 URL？★★★★
**出现频率最高**：字节、阿里、百度

**方案**：**分治 + 哈希分片**
1. 把 A、B 各按 `hash(url) % N` 分成 N 个小文件（如 N=1000，每片约 100 万 URL）；
2. 对每一对「A_i, B_i」，把 A_i 装入 `HashSet`，遍历 B_i 判断是否存在；
3. 汇总结果。

**其他方案**：布隆过滤器（有误判）、外部排序 + 归并。

---

### E5. 如何设计一个「支持过期时间的本地缓存」？★★★★
**出现频率最高**：阿里、美团、京东

**要考虑**：
1. 数据结构：`ConcurrentHashMap<K, CacheEntry>`，`CacheEntry` 含 `value` + `expireAt`；
2. 过期策略：**惰性删除**（get 时检查 `expireAt`，过期则删）+ **定时清理**（`ScheduledExecutorService` 定期扫描，或用 `DelayQueue`）；
3. 容量限制：LRU（`LinkedHashMap accessOrder`）+ 分段锁，或用 Caffeine 的 W-TinyLFU；
4. 并发：`ConcurrentHashMap` + `computeIfAbsent`（避免缓存击穿）；
5. 防击穿：`synchronized` 单 key / `FutureTask`（`computeIfAbsent` 已经保证只算一次）。

> 💡 **推荐直接说「生产用 Caffeine」**：W-TinyLFU 淘汰策略（比 LRU 命中率高）、异步刷新、统计、`maximumSize`/`expireAfterWrite` 等，性能比 Guava Cache 高一个量级。

---

### E6. 一致性哈希是什么？和集合有什么关系？★★★
**出现频率最高**：阿里、字节（分布式岗）

- 场景：分布式缓存/分库分表，把 key 映射到节点；
- 普通哈希：`hash(key) % N`，节点增减时**几乎全部 key 要迁移**；
- 一致性哈希：把 `0 ~ 2^32-1` 组成哈希环，节点和 key 都映射到环上，key 顺时针找第一个节点；
- 节点增减只影响**相邻区间**（平均迁移 1/N）；
- **虚拟节点**解决「数据倾斜」（每个物理节点映射多个虚拟节点）；
- Java 实现：`TreeMap<Long, Node>` 或 `ConcurrentSkipListMap`（**跳表的有序 + 范围查询 `ceilingKey` 完美适配**）。

**加分**：「`TreeMap.ceilingKey(hash)` 就是『顺时针找第一个节点』的 O(log n) 实现」——这是集合知识与分布式设计的漂亮结合点。

---

### E7. 如何在遍历 Map 时安全地删除元素？★★★★
**出现频率最高**：腾讯、京东

```java
// 1. Iterator.remove
Iterator<Map.Entry<String,Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getValue() < 0) it.remove();   // ✅
}

// 2. removeIf（JDK8，推荐）
map.entrySet().removeIf(e -> e.getValue() < 0);

// 3. ConcurrentHashMap 的 forEach + 收集（弱一致，不能边遍历边删）
// 4. 收集要删的 key，遍历完再 remove（最简单）
```

**追问**：「`map.keySet().remove(k)` 可以吗？」→ `KeySet` 是**视图**，`remove` 会转发到 `map.remove`，会改 `modCount`，所以**遍历中调用仍会 CME**。

---

### E8. `HashMap` 为什么用「尾插」而 `LinkedHashMap` 用「双向链表」还能保持顺序？★★★★
**出现频率最高**：字节（P7 深挖）

- `HashMap` 的桶内链表只管「hash 冲突」，**顺序无意义**，尾插是为了「扩容时不反转、不成环」；
- `LinkedHashMap` 的 `before/after` 链表是**独立的顺序链**，与桶内 `next` 链**互不干扰**；`put` 时同时挂到两条链上，扩容只影响 `next` 链（`resize` 不改 `before/after`），所以顺序不变。

---

### E9. 为什么 `ConcurrentHashMap` 的 `keySet()` 是视图，而 `CopyOnWriteArrayList` 的迭代是快照？★★★★
**出现频率最高**：阿里、美团

- **视图**（`CHM.keySet()`、`map.entrySet()`、`list.subList()`）：包装原容器，**实时反映**容器变化，不复制；优点是省内存，缺点是「遍历时容器变了会看到不一致状态」；
- **快照**（`COWArrayList.iterator()`）：创建时**持有当时的数组引用**，之后容器怎么变都不影响这个迭代器；优点是「绝对安全」，缺点是要额外内存（写时复制）。
- 两种设计的取舍：**CHM 追求「零拷贝 + 高并发」→ 视图 + 弱一致；COW 追求「读完全无锁 + 迭代绝对安全」→ 快照**。

---

### E10. 集合相关的「内存泄漏」场景有哪些？★★★★
**出现频率最高**：阿里、美团（P7）

| 场景 | 原因 | 解法 |
|---|---|---|
| `HashMap` 的 key 是可变对象 | hash 变了 → 删不掉 | key 用不可变对象 |
| `ArrayList.remove` 后不缩容 | `elementData` 仍持有大数组 | `trimToSize()` |
| `subList` 持有原 list 引用 | 原 list 无法回收 | 拷贝 `new ArrayList<>(subList)` |
| `ThreadLocal` 的 `ThreadLocalMap` | key 弱引用被回收，value 强引用残留 | `remove()` |
| 静态集合不断 `add` | 无界增长 | 用有界队列/定期清理/`WeakHashMap` |
| `LinkedList` 节点被外部强引用 | 无法回收 | 及时 `clear()` |
| 监听器注册不注销 | 集合持有监听器 | `removeListener` / `WeakReference` |

---

### E11. 为什么 `Collections.unmodifiableList` 不复制？★★★
**出现频率最高**：京东、滴滴

- 它是**装饰器**，内部持有原 list 的引用，所有「修改方法」抛 `UnsupportedOperationException`，读方法转发；
- **不复制** → 省内存、O(1)；
- **代价**：如果原 list 被其他代码改，视图「看得见」（因为共享）。所以「不可变」只是「通过这个视图不能改」，不是「绝对不变」。
- 要绝对不可变：`List.copyOf(list)`（JDK10+，会**真正拷贝**）或 `List.of(...)`。

---

### E12. `HashMap` 和 `TreeMap` 分别适合什么场景？★★★★
**出现频率最高**：所有大厂

| 场景 | 选择 | 原因 |
|---|---|---|
| 普通 KV 缓存 | `HashMap` | O(1) |
| 需要按 key 排序输出 | `TreeMap` | 中序遍历有序 |
| 范围查询（时间窗口、分数区间） | `TreeMap` / `ConcurrentSkipListMap` | `subMap` |
| 找「≥ 某值的最小 key」 | `TreeMap.ceilingKey` | O(log n) |
| 排行榜（实时更新 + 排序） | `ConcurrentSkipListMap` | 并发 + 有序 |
| 需要 LRU | `LinkedHashMap(accessOrder=true)` | 双向链表 O(1) 移动 |

---

### E13. 一个类要作为 `HashMap` 的 key，需要满足什么条件？★★★★
**出现频率最高**：小米、京东

1. 实现 `hashCode`（且**稳定**，不能随内部状态变）；
2. 实现 `equals`（且与 `hashCode` 语义一致）；
3. **不可变**（或至少「参与 hash/equals 的字段」不可变）；
4. 建议 `final` 类 / `record`；
5. 若实现 `Comparable`，注意与 `equals` 一致（否则 `TreeMap` 会误判）。

---

### E14. 用集合实现一个「生产者-消费者」模型 ★★★★
**出现频率最高**：美团、京东、拼多多

```java
// 方案 1：BlockingQueue（最推荐）
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);
// 生产者：queue.put(task);
// 消费者：Task t = queue.take();

// 方案 2：wait/notify（考察基础）
// 方案 3：ReentrantLock + Condition
// 方案 4：Disruptor（极高吞吐，无锁环形队列）
```

**追问**：「`ArrayBlockingQueue` 内部用什么锁？」→ `ReentrantLock` + 两个 `Condition`（`notEmpty`/`notFull`），`await` 必须包在 `while` 里防虚假唤醒。

---

### E15. 你平时怎么评估一个集合选得对不对？★★★
**出现频率最高**：阿里（P7 软技能题）

**四个维度**：
1. **访问模式**：随机访问 / 顺序访问 / 头尾操作 / 范围查询 / 按 key 查找；
2. **数据规模与增长**：能否预估容量（决定是否预设容量）、是否需要无界；
3. **并发模型**：单线程 / 读多写少 / 写多读少 / 高并发（决定锁策略）；
4. **内存与 GC**：元素大小、对象数量、生命周期（决定用数组还是链表、是否需要 `WeakReference`）。

**加分**：「我会先用最简单的（`ArrayList` / `HashMap`），上线后用 JFR / async-profiler 看热点，**只在有证据时才换**。过早优化集合选型是典型的过度设计。」

---

## F. 高频「陷阱题」速答（10 题）

| # | 题目 | 答案 |
|---|---|---|
| F1 | `HashMap` 的 `size` 最大是多少？ | `Integer.MAX_VALUE`（约 21 亿），受堆内存限制 |
| F2 | `HashMap` 能存 `null` value 吗？`get` 返回 null 怎么区分？ | 能；用 `containsKey` 区分（非并发下可靠） |
| F3 | `List.of()` 和 `Arrays.asList()` 哪个能放 null？ | `Arrays.asList` 可以，`List.of` 不可以 |
| F4 | `new HashSet<>(100)` 的 `HashMap` 容量是多少？ | 128（`tableSizeFor(100)`） |
| F5 | `Iterator` 和 `ListIterator` 区别？ | 后者支持双向遍历 + `add/set`，仅 List 有 |
| F6 | `Enumeration` 和 `Iterator` 区别？ | `Enumeration` 只读、不支持 `remove`、不抛 CME（`Vector`/`Hashtable` 遗留） |
| F7 | `ArrayList` 的 `clone()` 是深拷贝吗？ | **浅拷贝**：数组复制，元素引用共享 |
| F8 | `HashMap` 扩容会改变元素的 `hashCode` 吗？ | 不会，只改「索引」；`hash` 字段也不变 |
| F9 | `ConcurrentHashMap` 的 `size()` 为什么返回值可能偏小？ | 求和期间并发更新，且 `baseCount` 与 cells 可能短暂不一致 |
| F10 | `PriorityQueue` 的 `iterator()` 有序吗？ | **不保证**，只有 `poll()` 出队才有序 |

---

## G. 面试答题方法论（拿满分的关键）

### G1. 四层答题法

```
第 1 层：一句话结论（是什么）
第 2 层：核心机制（怎么实现，2~3 个关键点）
第 3 层：源码细节（字段、常量、关键方法名）
第 4 层：设计权衡（为什么这么做，代价是什么，替代方案）
```

**例**（问「HashMap 为什么 2 的幂」）：
1. 一句话：为了让位运算代替取模，并保证分布均匀。
2. 机制：`(n-1) & hash` 等价 `hash % n`；`n-1` 低位全 1。
3. 源码：`tableSizeFor` 的 5 次位移、`resize` 的 `(e.hash & oldCap)`。
4. 权衡：代价是「容量不能任意设」，所以构造时会给到「≥ 目标的最小 2 的幂」，可能浪费最多近 2 倍空间。

### G2. 主动「埋钩子」

回答时**主动抛出一个你熟悉的深水区**，引导面试官问：
- 「这里用尾插而不是头插，主要是为了避免 JDK7 那种并发成环的问题……」→ 面试官大概率追问死循环；
- 「树化的阈值 8 其实来自泊松分布的一个概率计算……」→ 追问 8/6/64；
- 「CHM 的计数用了类似 LongAdder 的分散思想……」→ 追问伪共享 / `@Contended`。

### G3. 不会的题怎么答

> 「这块我了解得不够深。我的理解是 X（基于已有知识推理），但不确定 Y 这个细节。如果让我实现，我会先 Z。」

**不要说「不知道」就结束**，也不要在不知道的地方编造源码细节——面试官一追问就露馅。

### G4. 手撕题的评分点

| 评分项 | 权重 | 说明 |
|---|---|---|
| 边界条件 | 高 | 空集合、null、容量 0/1、重复 key |
| 复杂度 | 高 | 能否说出时间/空间复杂度，是否均摊 |
| 代码风格 | 中 | 命名、哨兵节点、`final` |
| 扩展性 | 中 | 能否加「线程安全」「过期」「统计」 |
| 测试 | 低～中 | 主动举 2~3 个用例验证 |

---

## 附录 A：一页速记卡（面试前 10 分钟看）

```
【HashMap】
capacity=16(2^n)  loadFactor=0.75  treeify=8  untreeify=6  minTreeifyCap=64
index = (n-1) & (h ^ (h>>>16))
扩容：2 倍；按 (hash & oldCap) 拆低位/高位；新索引 = 原 或 原+oldCap
JDK7 头插 → 并发成环 → 死循环；JDK8 尾插 → 不成环但丢数据

【ConcurrentHashMap】
JDK7：Segment[16] + ReentrantLock + HashEntry（final next，头插）
JDK8：Node[] + CAS(空桶) + synchronized(桶首) + ForwardingNode + TreeBin
sizeCtl：0 未初始化 / -1 初始化中 / -(1+n) 扩容中(n 线程) / >0 阈值
计数：baseCount + CounterCell[]（@Contended），size() 是估算
扩容：resizeStamp(n)<<16 | (2+协助数)；stride=max(16, n/8/NCPU)；transferIndex 领任务
null：key/value 都不允许

【ArrayList】
默认 10（懒加载）→ 1.5 倍扩容 → Arrays.copyOf
transient elementData + 自定义 writeObject（只写 size 个）
modCount / expectedModCount → fail-fast
remove 置 null 帮 GC，但不缩容（trimToSize）

【ArrayDeque】
循环数组 16，head/tail，tail 是「下一个写入位」，head==tail 为空
(i±1) & (len-1) 环形；满 → doubleCapacity；不允许 null

【PriorityQueue】
小顶堆，数组，默认 11；offer=siftUp(O(log n))，poll=siftDown(O(log n))
建堆 O(n)；迭代不保证有序；不允许 null
TopK：最大 K 个 → 小顶堆

【LinkedHashMap】
HashMap + before/after 双链表；accessOrder=false 插入序 / true 访问序
removeEldestEntry → LRU；accessOrder=true 时 get 会改 modCount → 遍历中 get 会 CME

【BlockingQueue】
ArrayBlockingQueue：数组 + 单锁 + 2 Condition（读写互斥）
LinkedBlockingQueue：链表 + 双锁 + AtomicInteger（读写并行，默认无界 ⚠️）
SynchronousQueue：无容量，手递手
DelayQueue：PriorityQueue + leader 优化
```

---

## 附录 B：推荐延伸阅读

| 主题 | 材料 |
|---|---|
| HashMap 源码 | JDK 源码 `java.util.HashMap`（重点 `putVal`/`resize`/`treeifyBin`） |
| CHM 源码 | `java.util.concurrent.ConcurrentHashMap`（重点 `putVal`/`addCount`/`transfer`） |
| 并发计数 | `LongAdder` / `Striped64`（理解 `CounterCell` 的前身） |
| 伪共享 | `@sun.misc.Contended`、缓存行 64B、Disruptor 的 padding |
| 红黑树 | 《算法导论》第 13 章；Sedgewick 的左倾红黑树 |
| 跳表 | `ConcurrentSkipListMap` 源码；Redis `t_zset.c` |
| 集合规范 | 《Effective Java》Item 11（equals/hashCode）、Item 47（Stream） |
| 线程池 | `ThreadPoolExecutor` 源码 + 阿里 Java 开发手册「并发处理」章节 |
| 性能工具 | JMH（微基准）、async-profiler（火焰图）、JFR（飞行记录） |

---

> **免责声明**：本文档为学习笔记，源码片段以 JDK 8/11/17 为准，不同 JDK 版本实现细节（尤其 JDK 17 的 `ArrayList`、JDK 21 的虚拟线程相关集合）可能有差异，请以实际使用的 JDK 源码为准。面试答案中的「公司标注」来自公开面经的归纳，不代表官方题库。
