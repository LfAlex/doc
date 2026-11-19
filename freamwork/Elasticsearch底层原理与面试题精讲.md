# Elasticsearch 底层原理与面试题精讲

> 面向「想真正搞懂 ES 为什么这么设计」的读者：从倒排索引的字节级结构，到写入链路的每一次 fsync，到分布式查询的两阶段合并，最后落到国内一线大厂的高频面试题与逐步推导的答案。
>
> 全文按 **由浅入深** 组织：每章先给"一句话结论"，再拆结构，再讲代价与取舍。

---

## 0. 阅读指南

### 0.1 三条主线

理解 ES 只需要抓住三条主线，其余都是枝叶：

| 主线 | 核心问题 | 对应章节 |
|---|---|---|
| **存**：数据以什么形态躺在磁盘上 | 倒排索引 + 不可变段（segment） | 第 2、3 章 |
| **写**：一条文档如何变成可搜索、可恢复的数据 | buffer → refresh → translog → flush → merge | 第 4 章 |
| **查**：一次搜索如何跨越 N 个分片 | 路由 → query 阶段 → fetch 阶段 → 归并 | 第 5 章 |

把这三条线串起来，ES 里 90% 的"奇怪行为"都能自己解释出来。比如：

- 为什么写入后要等 1 秒才能搜到？→ 因为 refresh 是"存"与"查"之间的可见性开关。
- 为什么更新一条文档比插入慢？→ 因为段不可变，更新=删除+插入。
- 为什么删除数据磁盘不降？→ 因为 .liv 只是标记，空间要等 merge 才回收。
- 为什么分片数建好就不能改？→ 因为路由公式 `hash % 分片数` 依赖分片数，改了路由全错。

### 0.2 难度标记

本文面试题部分使用三档标记：

- **★☆☆ 基础必答**：答不上来直接减分，要求"准确 + 有结构"。
- **★★☆ 进阶区分**：能答出"是什么"，还要答出"为什么这样设计"和"代价是什么"。
- **★★★ 压轴深水区**：能讲到源码级结构、线上故障现象、量化取舍，通常决定 P6/P7 定级。

---

## 1. 认知地图：ES 到底是什么

### 1.1 一句话结论

> **Elasticsearch = Apache Lucene（单机索引引擎）+ 一层自己写的分布式外壳（路由、分片、副本、集群状态、REST、聚合、DSL）。**

这句结论决定了后面所有内容的归属：

- 凡是"倒排索引怎么压缩、FST 怎么省内存、BKD 树怎么查范围"——**这是 Lucene 的事**。
- 凡是"分片怎么路由、副本怎么同步、主节点怎么选、深分页为什么受限"——**这是 ES 外壳的事**。

面试里最常见的失分点，就是把两层混为一谈。比如被问"ES 写入流程"，很多人上来就讲 refresh 和 translog；但完整的链路其实是 **协调节点路由 → 主分片写入 → 副本同步 → translog/buffer → refresh → flush**，前四步属于 ES 外壳，后三步属于 Lucene。

### 1.2 三层 "index" 名词辨析（面试高频陷阱）

"index" 这个词在 ES 体系里有三个完全不同的含义，答错就暴露没读过底层：

| 层级 | 名称 | 本质 | 由谁管理 | 能否独立搜索 |
|---|---|---|---|---|
| 逻辑层 | **ES Index**（如 `products`） | 一批文档的逻辑集合，是 mapping/设置的容器 | ES 集群 | 是（但靠分片实现） |
| 分片层 | **Shard**（如 `products` 的 shard 0） | **一个完整的 Lucene Index 实例** | ES 分配、Lucene 执行 | 是，可独立读写 |
| 存储层 | **Segment**（如 `_0.cfs`） | 一个**不可变的迷你倒排索引** | Lucene 内部 | 是（读时对全部段逐一执行再归并） |

再补一句常被追问的：**ES 7.0 起彻底移除了 type（映射类型）**。原因是 Lucene 里没有"表"的概念，6.x 的"一个 index 多个 type"是靠在一个 Lucene index 里给每个文档加 `_type` 字段硬模拟的，导致同一字段名在不同 type 里必须类型一致、且全局 ordinals 空间被污染。7.x 通过"一个 index 一个 type `_doc`"，8.x 连 `_doc` 都只是 URL 兼容占位。这题问的是"你为什么这么设计"，答"官方让这么干"是不够的。

### 1.3 层次结构图

```
                     ┌──────────────────────────────────────────────┐
   Client ──REST/HTTP──▶│  Coordinating Node（任意节点都可充当）        │
                     │  解析 DSL → 改写 → 路由 → 聚合结果 → 返回      │
                     └───────────────┬──────────────────────────────┘
                                     │  按 routing 分发
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
 ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
 │ Shard 0(P)   │            │ Shard 1(P)   │    ...     │ Shard N(P)   │
 │ = Lucene Idx │            │ = Lucene Idx │            │ = Lucene Idx │
 └──────┬───────┘            └──────────────┘            └──────────────┘
        │ 同步复制
        ▼
 ┌──────────────┐
 │ Shard 0(R)   │  ← 副本：承担读流量 + 故障接管
 └──────────────┘

 单个 Shard(=Lucene Index) 内部：
 ┌─────────────────────────────────────────────────────────────────┐
 │ 内存 IndexWriter Buffer ──refresh──▶ Segment _0  Segment _1 ...   │
 │        │                                   │                     │
 │        └─────── 同时写 ──────▶ translog（WAL，用于崩溃恢复）        │
 │                                            │                     │
 │                                     后台 merge 合并小段            │
 └─────────────────────────────────────────────────────────────────┘
```

### 1.4 关键量化基线（记住这些数字，面试很有用）

| 参数 | 默认值 | 含义 / 影响 |
|---|---|---|
| `index.refresh_interval` | **1s** | 写入到可搜索的延迟上限（近实时的来源） |
| `index.translog.durability` | **request** | 每个写请求都 fsync translog，宕机最多丢"当前这条未完成的" |
| `index.translog.sync_interval` | **5s** | 仅当 durability=async 时生效，**最多丢 5 秒数据** |
| `index.translog.flush_threshold_size` | **512MB** | translog 涨到此值触发 Lucene commit（flush） |
| `index.translog.retention.age` | **12h** | translog 保留时长，影响恢复窗口 |
| `indices.memory.index_buffer_size` | **10% 堆** | 写入缓冲区，满了也会触发段生成 |
| `index.max_result_window` | **10000** | `from + size` 的硬上限 |
| `index.merge.policy.segments_per_tier` | **10** | 每层最多 10 个段，超过就合并 |
| `index.merge.policy.max_merged_segment` | **5GB** | 单段上限，超过不再参与普通合并 |
| 堆内存 | **≤50% 物理内存，且 ≤ ~31GB** | 超过 32GB 失去压缩指针（compressed oops） |
| 磁盘水位线 | **85% / 90% / 95%** | low / high / flood（flood 会强制索引只读） |

---

## 2. 倒排索引：ES 快的根本原因

### 2.1 一句话结论

> **正排索引是"文档 → 词"，倒排索引是"词 → 文档列表"。全文检索要的是后者：拿到查询词，直接跳到包含它的文档集合，而不是把每个文档读一遍。**

类比：正排像一本书的正文，你只能一页页翻着找关键词；倒排像书末的索引页（`索引 → 第 12, 45, 88 页`），你查到词就直接翻到对应页码。

举例，三篇文档：

```
doc1: "小米手机 快充 67W"
doc2: "华为手机 快充 66W"
doc3: "小米 笔记本 快充 100W"
```

分词后的倒排表（简化，省略位置信息）：

```
快充  → [1, 2, 3]
手机  → [1, 2]
小米  → [1, 3]
67W   → [1]
100W  → [3]
```

查询 `手机 AND 快充`：取 `手机=[1,2]` 与 `快充=[1,2,3]` 求交集 → `[1,2]`。**扫描量与文档总数无关，只与倒排列表长度有关。**

这里已经埋下两个关键工程问题，也是面试的深水区：

1. 词表可能有上亿个 term，**怎么在磁盘上快速定位某个 term 的倒排列表**？（→ 词典 + FST）
2. 倒排列表可能上千万个 docId，**怎么省空间、怎么快速求交**？（→ 压缩编码 + 跳表）

### 2.2 倒排索引的三层结构

Lucene 的倒排索引不是"一个词一个链表"这么朴素，它被拆成三层，各自解决不同的问题：

```
┌──────────────────────────────────────────────────────────────────┐
│ ① Term Index（词项索引）  ← 常驻内存（堆），体积极小              │
│    本质：FST（有限状态转换机），key = term 的前缀路径             │
│    value = 该 term 所在 .tim 文件的块偏移（block fp）             │
├──────────────────────────────────────────────────────────────────┤
│ ② Term Dictionary（词典 .tim）  ← 磁盘 + page cache               │
│    按字典序分块存储（默认每块 128 个 term），块内前缀共享压缩      │
│    每个 term 后跟它的 posting 元数据（docFreq、位置指针）          │
├──────────────────────────────────────────────────────────────────┤
│ ③ Posting List（倒排列表 .doc/.pos）  ← 磁盘 + page cache          │
│    文档号升序 + delta 编码 + 位宽打包；可选 positions/freqs        │
└──────────────────────────────────────────────────────────────────┘

查询 "elasticsearch" 的真实路径：
FST 中走完 e-l-a-s-t-i-c-s-e-a-r-c-h 的路径
   → 拿到 block fp（比如 .tim 第 3 块的偏移）
   → 读该块（128 个 term，通常已在 page cache 里）
   → 块内二分/线性定位到 "elasticsearch"
   → 取出 docFreq 和 posting 起始指针
   → 顺序读 .doc 中的倒排列表（按 128 个 docId 一组跳读）
```

**为什么要三层而不是直接一个 `HashMap<term, posting>`？**

- 词表太大：上亿 term，string 本身 + 指针，纯堆内存几个 GB 起，且 JVM GC 会很痛（大对象、长生命周期、无法回收）。
- 前缀重复度高：英文词、URL、商品标题里的品牌词有大量公共前缀，压缩收益巨大。
- 冷热分离：**热数据是"词的路径"，冷数据是"倒排列表"**。前者小到能全量进堆，后者大到只能靠 OS page cache 命中。

### 2.3 FST：为什么用有限状态转换机

FST（Finite State Transducer）本质是一个**带输出值的、确定性、无环的最小自动机**，它同时具备两个压缩特性：

1. **前缀共享**：`cat` / `car` / `care` 共享 `ca` 路径。
2. **后缀共享（这是 Trie 做不到的）**：`mop` / `top` / `pop` 的尾部 `op` 合并到同一组状态节点；`internal` / `eternal` 这类词也能共享尾部结构。

因此 FST 的体积可以显著小于原始词表，常见量级是**词表的几十分之一**。这也是"几千万 term 的索引，term index 只有几十 MB，能整块放堆里"的原因。

FST 的几个关键性质，面试常问：

| 性质 | 说明 |
|---|---|
| 查询复杂度 | **O(term 长度)**，与词表大小无关；走一次状态转移，最后检查该状态是否接受（final） |
| 支持能力 | 前缀查询（`prefix`）、范围查询（term range）都是 FST 的天然能力，因为它是按字典序构建的 |
| 不支持能力 | 通配符 `*abc`（前导通配）无法走 FST，因为路径方向是从前向后；这也是"前导 `*` 查询会全表扫描"的底层原因 |
| 构建代价 | 需要**全量 term 有序**才能构建，所以只能"写完再建"；这也是段不可变的原因之一 |
| 内存归属 | FST 读入堆内存（不是 mmap），因此 **term 数暴涨会直接抬升堆占用**——这是"字段基数过高导致堆压力"的一个隐藏来源 |

> **延伸：为什么不用 B 树 / 哈希？**
> 哈希不支持前缀和范围（`prefix`、`range`、`wildcard` 都要扫）；B 树能范围但每条边要存完整 key 片段，体积大得多。FST 在"只读、静态、需要字典序"的前提下是近乎最优解——而这恰好就是不可变段的前提。**结构选择永远跟着"数据是否可变"走。**

> **延伸：ES 的 term 数为什么要注意？**
> 每个 segment 都有一份自己的 term dictionary 和 term index。如果某字段是高基数字段（UUID、sessionId、日志原文）却用了 `text` 类型，term 数会随文档线性爆炸，导致：段生成变慢、FST 变大、merge 变慢、堆压力上升。正确做法是 `index: false`（不索引）或改成 `keyword` + 不通配查询，甚至只留 `_source`。这是面试题"如何优化 ES 存储/写入"的高分答案之一。

### 2.4 Posting List：三个压缩技巧

倒排列表的核心矛盾是**大**：一个热门词可能命中几百万文档。Lucene 用了三招：

**① delta 编码（差分）**
docId 是有序的（段内按写入顺序递增），存 `[3, 7, 9, 11]` 不如存 `[3, 4, 2, 2]`。数值变小后，需要的二进制位变少。

**② 定长块 + 位宽打包（bit packing / FOR 思路）**
把列表切成固定长度块（Lucene 典型为 128 个 docId 一块），每块只记录"本块最大值需要多少 bit"，块内所有数按这个位宽紧凑排列。

```
docId:   1000, 1002, 1003, 1009   (delta: 1000, 2, 1, 6)
块头: 需要 10 bit（因为最大 1000 < 2^10）
块体: 每个 delta 用 10 bit 存 → 4 * 10 = 40 bit 代替 4 * 32 bit
```

代价是块内"一个特别大的数"会抬高整块的位宽，所以分块本身也是一种自适应。

**③ 稀疏用跳表，稠密用位图**

- 稀疏（命中文档占少数）：有序数组 + **跳表（skip list）**。每 128 个 docId 抽一个"跳跃指针"，求交时可以整块跳过。
- 稠密（命中比例很高）：**bitmap / Roaring Bitmap**。Roaring 按高 16 位分容器：稀疏容器用有序数组，稠密容器用 2^16 位图，极稠密用 RLE 行程编码——一个结构适配三种分布。

**求交为什么快（这是"多词查询"的性能核心）**

```
查询 A AND B：
1) 先按 docFreq 升序排序 → 最短的列表当"驱动列表"（driving list）
2) 用跳表在长列表上做"跳跃定位"：当前 docId=57，跳 128 到 185，若 185 > 57，
   则回退到 128 附近的块内二分
3) 两个列表多路归并，只有 docId 相等才输出
4) 若某列表 docFreq > 阈值（默认与其它列表长度比），改用 bitset 直接按位与
```

**量化直觉**：两个各 100 万 docId 的列表求交，若纯线性比对是 100 万次比较；有跳表后，实际比较次数约 `100万 / 128 * 常数`，量级降到 1/100 左右。这就是"为什么 ES 多关键词查询也不慢"的底层答案。

### 2.5 位置信息（positions）与短语查询

`match_phrase` 要求"词相邻且顺序一致"，只靠 docId 不够，必须存**每个 term 在每个文档中的出现位置（position）**。

- 有 positions → 可以做 `match_phrase`、`span_near`、邻近度（slop）、高亮。
- 无 positions（`index_options: docs` / `freqs`）→ 只能做 `term`/布尔查询，但**索引体积可小 20%~40%**，写入更快。
- 这是典型的"功能换空间"取舍。日志类字段如果只做 `term` 精确过滤，把 `index_options` 设成 `docs` 是常见优化。

`index_options` 四档：`docs`（只存 docId） < `freqs`（+词频） < `positions`（+位置，默认） < `offsets`（+字符偏移，供高亮/`fast vector highlighter`）。

### 2.6 打分：从 TF-IDF 到 BM25

倒排列表给出"谁包含"，打分解决"谁更相关"。ES 5.0 起默认 **BM25**（此前是 TF-IDF）。

**TF-IDF 的直觉与缺陷**

```
score = tf * idf * boost * norm
tf   = 词在文档中出现次数（越多越相关）
idf  = log(总文档数 / 含该词的文档数)（越罕见越有区分度）
norm = 1 / sqrt(字段长度)（文档越长越"稀释"）
```

缺陷：`tf` 是线性的。一个词出现 100 次就比出现 10 次相关 10 倍？现实中早就饱和了。

**BM25 的修正（三段式）**

```
                tf × (k1 + 1)
score = IDF × ─────────────────────────────── × (1 + ...) 
              tf + k1 × (1 - b + b × dl / avgdl)

IDF   = ln(1 + (N - n + 0.5) / (n + 0.5))     ← 平滑，避免负值
k1    = 1.2 （默认） 控制 tf 饱和速度：tf 越大，边际收益越小，上限趋近 (k1+1)
b     = 0.75（默认） 控制长度归一强度：b=0 完全不管长度，b=1 完全按比例惩罚
dl    = 当前文档该字段长度（词数）
avgdl = 全段该字段平均长度
```

三个可调旋钮的工程含义：

- `k1` 调小 → 更看重"是否命中"而不是"命中几次"，适合短文本（标题）。
- `b` 调小 → 长文档不吃亏，适合正文型/FAQ 型内容。
- 多字段时用 `multi_match` 的 `best_fields`（取最高分字段）或 `most_fields`（各字段分相加）+ 字段 `boost`。

> **延伸：BM25 在分布式下的坑**
> 打分依赖 `N`（总文档数）、`n`（含该词文档数）、`avgdl`——这些是**跨分片的全局统计量**。默认 query-then-fetch 下，各分片用"本分片的局部统计"打分，同一文档在不同分片布局下分数可能不同，甚至分片间分数不可比。这就是 **DFS query-then-fetch** 存在的理由（见 5.3）。
>
> **延伸：什么时候该放弃打分**
> 电商/内容检索里，相关性常被"业务权重 + 排序模型"接管。此时应把过滤条件放 `filter`（不参与打分、可缓存），排序用 `sort`，`track_scores: false`，甚至用 `function_score`/`rank_feature` 把 BM25 分数降为一路特征。这能把 CPU 从打分里省出来。

### 2.7 数值/日期/地理为什么不用倒排：BKD 树

`int`、`long`、`date`、`ip`、`geo_point`、`geo_shape` 走的是 **BKD 树（Block k-d Tree）**，不是倒排索引。

**为什么？** 倒排是"精确值 → 文档"，适合"等值匹配"。但 `price between 100 and 200` 是**范围**查询：倒排得枚举区间内所有具体数值（数值域可能是 2^64），不可行。

**BKD 树的原理**

```
把 N 个 (维度 d) 点递归地用中位数切分：
  第 1 层按维度 1 切 → 左子树值都小，右子树值都大
  第 2 层按维度 2 切 → ...
  叶子节点存最多 leaf_max_size（默认 1024）个点，压成紧凑块
范围查询 = 递归下探 + 子树剪枝（整块在区间外直接跳过）
```

- 1 维时退化为**有序数组 + 二分**（所以单个数值范围查询极快）。
- 多维（`geo_point` 的 lat/lon）时剪枝效率依然很好，因为每层换一个维度切，边界框能快速排除大量子树。
- 叶子块（1024 点）是"块存储"，因此叫 **Block** k-d tree，能顺序读、压缩好。

> 历史延伸：Lucene 6.0 之前数值索引用的是 **Trie 树（前缀树，按位切分）**，每个精度层一个 term，term 数随精度爆炸。BKD 是 6.0 引入的替代方案。面试里"ES 数值索引结构"答"倒排"是错的，答"BKD 树"才对，能补一句"6.0 之前是 Trie"就是加分项。

### 2.8 doc values：排序和聚合的"另一半"

倒排擅长"由词找文档"，但不擅长"由文档取某字段的值并按它排序"。原因很直接：倒排里同一个字段的值散落在各个 term 的 posting 里，要遍历"所有 term → 所有文档"才能拿到一列值。

**doc values = 列式存储（columnar）**，在**索引时**就随段一起生成，每个字段一个"列"：

```
docId:      0    1    2    3
price 列:  199  299  99   499     ← 连续存放，顺序扫描，压缩好（delta + GCD）
```

- **排序/聚合/脚本访问**全部走 doc values，不做 fielddata（fielddata 是"从倒排临时构建列"，全部堆内存，是 ES 1.x/2.x 的事故源，`text` 字段至今禁止开 fielddata，必须用 `keyword`）。
- **字符串列**会先做 **ordinal 映射**：`"北京"→0, "上海"→1`，列里只存 `int`，聚合时按 ordinal 分组，最后再映射回字符串。这就是为什么 `terms` 聚合比 `LIKE` 快得多。
- **global ordinals**：跨段统一编号。聚合需要"同一个词在不同段里是同一个桶"，所以要么用 global ordinals（额外内存 + 构建成本，适合少字段多查询），要么在协调节点做 map 合并（省内存但耗 CPU/网络）。`execution_hint: map` / `global_ordinals` / `bytes_hash` 就是在做这个取舍。

> **延伸：为什么"给 text 字段排序"会报错？**
> `text` 字段经过分词，一个文档对应多个 term，**不存在"一个文档一个值"的列**，无法构建 doc values。所以要么加一个 `keyword` 子字段（`fields: {raw: {type: keyword}}`），要么用 `fielddata`（不推荐，堆杀手）。

### 2.9 五种数据结构一表对比（这张表能答掉一半"底层结构"题）

| 结构 | 服务的字段类型 | 解决什么查询 | 存在哪 | 关键代价 |
|---|---|---|---|---|
| 倒排索引（.tim/.doc/.pos） | text / keyword | 全文检索、term、phrase | 磁盘 + page cache | 体积大、term 多时构建慢 |
| FST（term index） | 所有倒排字段 | 快速定位 term | **堆内存** | term 基数高则堆压力 |
| BKD 树（.kdi/.kdd） | 数值、日期、IP、geo | 范围、距离、多边形 | 磁盘 + page cache | 更新需重建（段不可变） |
| doc values（.dvd/.dvm） | 几乎所有字段（除 text） | 排序、聚合、脚本、`docvalue_fields` | 磁盘 + page cache | 索引体积增加、写入变慢 |
| stored fields / `_source`（.fdt） | 全部 | 取回原文（fetch 阶段、reindex） | 磁盘（可压缩） | 占空间；`_source: false` 后无法 reindex/更新 |

---

## 3. 段（Segment）：不可变带来的全部设计

### 3.1 一句话结论

> **Lucene 选择"永远不改已写好的文件"，把"修改"翻译成"新增 + 标记删除"，用后台 merge 慢慢收拾。这一条决定了 refresh、translog、merge、删除不回收空间等几乎所有现象。**

为什么必须不可变？三个硬理由：

1. **倒排索引无法原地更新**：往一个已排序的 term dictionary 中间插入一个 term，等于重写整个词典；往 posting list 中间插 docId，等于重写整条列表。代价不可接受。
2. **并发模型极简**：段一旦写完就只读，读者（search）与写者（index）不再争同一份数据，只需引用计数管理"什么时候能删旧段"。这避免了读写锁，直接换来高并发。
3. **构建依赖全量有序**：FST 需要 term 全局有序才能建，有序只能"写完再建"。

### 3.2 一个段里到底有哪些文件

| 文件 | 内容 | 谁用 |
|---|---|---|
| `.tim` / `.tip` | term dictionary / term index（FST 在 .tip） | 全文检索定位 term |
| `.doc` | posting list 的 docId + 词频 | 布尔查询、打分 |
| `.pos` | posting 的位置信息 | phrase、高亮、邻近 |
| `.fdt` / `.fdx` | stored fields（`_source` 压缩后存这里） | fetch 阶段取原文 |
| `.dvd` / `.dvm` | doc values 数据 / 元数据 | 排序、聚合 |
| `.kdd` / `.kdi` | BKD 树数据 / 索引 | 数值/geo 范围查询 |
| `.nvd` / `.nvm` | norms（字段长度归一化因子，1 byte/字段/文档） | BM25 打分 |
| `.liv` | **活文档位图**（哪些 docId 已删） | 所有查询都先过它 |
| `.si` / `segments_N` | 段元信息 / **提交点**（当前有哪些段是"已提交"的） | 启动恢复、flush 落盘 |

一个常见追问："`_source` 和 `store: true` 有什么区别？"——`_source` 是整条文档的原始 JSON（默认压缩存 `.fdt`），用于 fetch 返回和 reindex；`store: true` 是**单字段**独立存一份，用于"只取某几个字段时不必解压整条 `_source`"（`stored_fields`）。多数场景 `_source` 够用，`store` 只在"文档巨大但只查少数小字段"时才有收益。

### 3.3 删除与更新：全是"标记"

```
删除 doc 5：
  .liv 里把 docId=5 的位从 1 改成 0     ← 磁盘上一个 bit
  文件一个字节都没少                    ← 所以删数据磁盘不降

更新 doc 7 的内容：
  .liv 里 docId=7 置 0（旧版本失效）
  新版本写入当前正在写的段（新 docId）
  → 查询时旧版本可能仍被匹配到，但结果输出前会被 .liv 过滤掉
  → 空间回收要等 merge
  → 所以"更新" = 一次删除 + 一次插入，成本高于插入
```

**由此导出三个高频面试结论**：

1. **频繁 update 同一文档 = 写放大灾难**。每次更新都新增一份完整文档（含 `_source`、doc values、倒排条目），旧版本只标删。日志/状态类数据如果高频覆盖写，应改成"追加新文档 + 外部版本号过滤"，或用 `_update_by_query` 谨慎处理，或干脆换存储（如 ClickHouse/HBase）。
2. **`delete by query` 不等于释放空间**，只是批量标删 + 可选 refresh；之后仍需 merge。
3. **`_forcemerge?only_expunge_deletes=true`** 是"只清理已删文档"的轻量 merge，比全量 forcemerge 安全得多，适合定期清理大量删除后的索引。

### 3.4 合并（merge）：TieredMergePolicy

merge 是"把小段合成大段"的后台任务，收益有三：

- 减少段数 → 查询要遍历的段变少（每个段都是一次完整倒排查找）。
- 回收已删文档空间。
- 提升压缩率（大段里 docId 更连续，delta 更小；词典更大，FST 共享更多）。

**TieredMergePolicy 的分层策略**（ES 默认策略）：

```
把段按大小分层（tier），每层最多 segments_per_tier=10 个
若某层段数 > 10，挑出"大小相近"的若干个（考虑删除率 expunge_deletes_allowed=10%）
合并成一个新段；新段进入更大一层
单段超过 max_merged_segment=5GB 后不再参与常规合并（避免"大雪球"越滚越慢）
```

关键细节：

- **merge 是"读旧段 + 写新段"**，写放大 1 倍；同时 merge 期间旧段仍要服务查询（引用计数），所以峰值磁盘占用可能接近 2 倍。
- **merge 会限流**（`indices.store.throttle.max_bytes_per_sec`，7.x 起默认 auto/不限但由外部配额控制），但大 merge 仍可能打满 IO 导致查询变慢——这是"夜间写入突增导致白天查询抖动"的典型原因。
- **`_forcemerge` 无节流**，会吃满 IO。官方明确建议：只对**只读索引**（按天/周滚动的历史索引）做 `max_num_segments=1`，且最好先把索引迁到空闲节点。对活跃写入索引 force merge 是反模式：合并出的巨大段会让后续 merge 更痛苦，且新写入产生的小段又要重新堆叠。

> **延伸：`index.merge.policy.*` 什么时候该动？**
> 默认值对大多数场景够用。只有两类场景值得调：① 大量顺序追加 + 少量删除的时序数据，调大 `max_merged_segment`（如 10GB）减少段数；② 写入极频繁的小文档，调小 `segments_per_tier` 反而会更频繁合并（通常不要动，而是调大 refresh_interval 或用 bulk 降低段生成速率）。

### 3.5 段数量与查询性能的量化关系

一次搜索在每个分片上要对**该分片的所有段**各执行一遍倒排查找，然后归并：

```
单分片查询成本 ≈ Σ(每个段：词典定位 + posting 扫描) + 归并开销
→ 段数从 100 降到 1，通常能有数倍查询提升（尤其 term 查询和聚合）
→ 但段太少（超大段）会让"新增文档"只能追加到极小段，段数又会回升
```

**工程结论**：段数的健康区间是"每分片十几到几十个"。监控 `_cat/segments`、`_nodes/stats/indices/segments`。段数长期上百，说明 refresh 太频繁或写入过碎（该调 `refresh_interval`、上 bulk、或做 rollover 分索引）。

---
## 4. 写入链路：一条文档的完整旅程

### 4.1 一句话结论

> **写 = 路由到主分片 → 写 translog（保证不丢）→ 写内存 buffer（等 refresh 变可见）→ 同步副本 → 等待全部 ack。搜索可见性与持久化是两件被刻意拆开的事。**

### 4.2 端到端 8 步（含每一步的失败影响）

```
① Client 发起 index/bulk 请求
        │  （任意节点接收，该节点成为本次请求的 Coordinating Node）
        ▼
② Coordinating Node 解析文档、按 mapping 校验字段类型
        │  计算路由：shard = floorMod(murmur3(_routing or _id), num_primary_shards)
        ▼
③ 转发到目标 primary shard 所在节点
        ▼
④ Primary shard 执行写入：
        ├─ 4.1 写 translog（durability=request 时立即 fsync）  ← 落盘，可恢复
        └─ 4.2 写 Lucene IndexWriter 内存 buffer              ← 尚未可搜索
        ▼
⑤ Primary 把这份 change 广播给该分片的所有 in-sync replica
        （每条副本独立执行 4.1/4.2）
        ▼
⑥ 等待所有 in-sync replica ack（受 wait_for_active_shards 约束）
        ▼
⑦ Coordinating Node 汇总，返回 {"result":"created","_seq_no":..,"_primary_term":..}
        ▼
⑧ 之后的某个时刻：
        ├─ refresh（默认 1s）：buffer → 新 segment → 打开新 reader → 可搜索
        └─ flush（translog 512MB / 30min）：Lucene commit + fsync 段文件 + 清空 translog
```

**每一步"挂了会怎样"，这是面试的加分点**：

| 失败点 | 后果 |
|---|---|
| ② 路由节点收到请求但转发失败 | 客户端收到失败/超时；若 `_id` 已写入主分片，可能产生"客户端认为失败、实际成功"的不一致（需用 `_id` 幂等重试 + `_version` 校验） |
| ④ translog 未 fsync 就宕机 | durability=request 时极端情况下丢"正在写的这一条"；async 时最多丢 `sync_interval`（默认 5s）内的所有写 |
| ④ buffer 已写、refresh 未发生就宕机 | 数据在 translog 里，重启时 **replay translog** 恢复到 buffer/段，不丢 |
| ⑤ 某副本 ack 失败 | 该副本被标记为 stale/失效，主分片降级处理；写入仍成功（只要满足 `wait_for_active_shards`） |
| ⑥ 副本数不足 | 若 `wait_for_active_shards` 不满足，请求被拒绝；否则成功但**冗余度下降**（集群 yellow） |

### 4.3 三个最容易被混淆的概念

这是面试里"你到底懂不懂"的分水岭，必须能用一句话切开：

| 概念 | 干了什么 | 解决什么问题 | 触发条件 | 数据在哪 |
|---|---|---|---|---|
| **refresh** | buffer → 新 segment，重开 IndexReader | **可见性**（能不能搜到） | 默认 1s / `?refresh=true` / buffer 满 | Filesystem cache（**未 fsync**） |
| **flush**（Lucene commit） | refresh + fsync 段文件 + 写新 commit point + 清空 translog | **持久化的干净起点**（缩短恢复时间） | translog 512MB / 30min / `_flush` API | 磁盘（fsync 完成） |
| **merge** | 多个小段 → 一个大段，丢弃已删文档 | **查询效率 + 空间回收** | 后台策略触发 | 磁盘 |

关键洞察：**refresh 之后数据已经"能搜到"，但还没"防断电"**。此时断电，数据靠 translog 重放恢复；只有 flush 之后才真正不依赖 translog。所以：

- 把 `refresh_interval` 调成 30s，**只降低可见性频率，不降低持久性**（translog 仍是每条 fsync）。
- 把 `translog.durability` 调成 async，**降低的是持久性，不是可见性**。

这两个开关正交，答错方向会被判定"没实操过"。

### 4.4 translog 细节

**结构**：每个 shard 一个 translog，由多个 `translog-N.log` 文件 + 一个 `translog-N.tst` 组成。写入是 append-only，因此极快（顺序写）；每 512MB 滚动一个新文件。

**两阶段写入**：每条操作先写 translog（`add`），refresh 时把该批操作的 translog 位置记入"uncommitted operations"；commit（flush）时把这一批标记为已提交，然后删除已提交的 translog 文件。

**恢复（recovery）**：分片启动/节点重启时，Lucene 先打开**最后一次 commit point**（`segments_N`）里的段 → 得到"已提交"的数据 → 再 replay 该 commit 之后所有 translog 记录 → 恢复到崩溃前的状态。这就是"即使没 flush 也不丢已 ack 数据"的完整机制。

**translog 还有一个隐藏用途：实时 GET**
`GET /index/_doc/1` 在文档尚未 refresh 时，主分片会**直接从 translog 里读出这条文档**（realtime get）。所以"按 id 取"是实时的，"按条件搜"是近实时的。这个差异在"用户下单后立刻查订单详情"的场景里非常有用。

**调参取舍**：

```yaml
# 追吞吐，可容忍极端情况丢几秒（如日志采集）
index.translog.durability: async
index.translog.sync_interval: 5s

# 追可靠性（默认，金融/订单）
index.translog.durability: request   # 每条 fsync，写入延迟受磁盘 fsync 能力约束
```

> **延伸：为什么 fsync 是写入吞吐的隐形天花板？**
> `request` 模式下每条写请求都要等一次 fsync（机械盘 ~5-10ms，SSD ~0.1-1ms）。单分片因此被限制在百级到千级 TPS。解法不是"改 async 赌命"，而是：**bulk 批量**（一次 fsync 覆盖上千文档）、**加主分片数**（并行 fsync）、**换 NVMe**、**把 `_id` 改为不自带索引**（见 4.7）。这也是"大促前为什么要扩分片"的真正原因。

### 4.5 写放大：三层叠加

一个文档从客户端到磁盘，实际写入的字节可能是原始大小的数倍：

```
第 1 层：副本放大      → 1 主 + 1 副本 = 2 倍
第 2 层：段内多结构    → 倒排 + doc values + _source + norms ≈ 2~4 倍
第 3 层：merge 重写    → 每个段平均被重写 log_10(N) 次（分层合并），约 1.5~3 倍
────────────────────────────────────────────────
总写放大常见 6~20 倍
```

所以"每天写入 100GB 原始数据"的集群，磁盘 IO 与容量不能只按 100GB 算。这也是为什么日志场景要用 `index_options: docs`、关掉不需要的 doc values、压缩 `_source`（`best_compression`）、并按 ILM 定期删除/归档。

### 4.6 bulk 与背压：为什么"一次发 10 万条"反而更慢

- **bulk 不是事务**：每条形如 `{"index":{"_id":1}}` 的 action 独立生效，**部分失败不影响其它条**，必须检查 response 里的 `errors` 和 `items[].status`。这是高频面试题。
- **单批过大**：协调节点要在内存里构造整批请求、序列化、分发；主分片要在一个 refresh 周期内吞下巨量文档，可能触发 `es_rejected_execution_exception`（队列满，默认 `thread_pool.write.queue_size=1000`）。
- **推荐批量**：单批 5MB~15MB 或 1000~5000 条（按文档大小调），并发 2~4 个线程即可打满；用 `_bulk` 的 `filter_path=errors,items.*.error` 减少返回体。
- **导入期优化组合**：`refresh_interval: -1`（关 refresh）+ `number_of_replicas: 0`（关副本）+ 大 bulk → 导入完成后恢复设置 + `_forcemerge`（只读索引）。
- **写入拒绝的处理**：客户端必须**指数退避重试**，只重试 429/503 与 `rejected` 类错误，不要无脑重试 400（mapping 冲突类错误重试无穷次也没用）。

### 4.7 并发控制：ES 的"乐观锁"是什么

ES 没有悲观锁，只有**乐观并发控制**，两套机制并存（旧版 `_version`，7.x 推荐 `_seq_no` + `_primary_term`）：

```
_seq_no      ：分片级单调递增的操作序号（每写一次 +1）
_primary_term：主分片"任期"，主分片切换（failover）时 +1

更新时带上 if_seq_no=5 & if_primary_term=2：
  若当前分片 seq_no 已是 6 → 冲突，返回 409 version_conflict_engine_exception
  若主分片已换代（term 变了）→ 也拒绝，避免"对着旧主分片的老视图"写入
```

**为什么必须带 `_primary_term`？** 只有 `_seq_no` 不够：主分片 failover 后，新主分片的 `_seq_no` 可能和旧主分片"看起来相同但含义不同"（旧主分片上的 5 与新主分片上的 5 是不同的写）。`_primary_term` 相当于 Raft 的 term，用来区分"哪个任期的序号"。

**`retry_on_conflict`**：`_update` API 支持自动重试（默认 0），在"读-改-写"模式下设成 3~5 可以吸收偶发冲突。但注意它只对 `_update` 生效，对 `index` 覆盖写无效。

**原子性边界（必须答准）**：

- **单条文档的写是原子的**（一条 index/update 要么整条生效要么不生效）。
- **bulk 不是原子的**，没有跨文档事务，没有回滚。
- **没有跨分片/跨索引事务**（ES 不是 OLTP 数据库）。需要事务的场景应该把"业务真相"放在 MySQL，ES 只做检索副本，用 binlog/CDC 同步 + 幂等 `_id` 保证最终一致。
- **`_update_by_query` / `_delete_by_query` 用快照 + 乐观锁**：执行期间新写入的文档不在快照内，可能被漏处理；冲突可用 `conflicts=proceed` 跳过。

### 4.8 写一致性参数

| 参数 | 语义 | 典型用法 |
|---|---|---|
| `wait_for_active_shards`（默认 1） | 写之前要求至少 N 个分片副本活跃（`all` 或数字） | 强一致场景设 `all`/副本数+1，代价是可用性下降 |
| `refresh=false`（默认） | 写完不等待可见 | 高吞吐写入 |
| `refresh=wait_for` | 等待下一次 refresh 完成后返回（**不强制立刻 refresh**） | 写入后立刻要能搜到，但不想打乱 refresh 节奏 |
| `refresh=true` | 强制立即 refresh（**昂贵**，制造大量小段） | 测试、极低频的"写完必须立刻搜" |
| `timeout` / `?master_timeout` | 请求级超时 | 避免长尾拖垮客户端线程池 |

重点区分 `wait_for` 与 `true`：前者是"等下个周期"，后者是"现在就来一次"。批量写用 `refresh=true` 会产生大量 1 文档小段，直接拖垮 merge 与查询。

### 4.9 mapping 在写路径上的作用

写入时会**逐字段按 mapping 校验并转换**：类型不匹配会 `mapper_parsing_exception`；`dynamic: true`（默认）会自动为新字段推断类型并**更新集群状态（cluster state）**——这意味着"未知字段写入"会触发元数据变更，是隐蔽的性能与一致性风险（多节点并发推断可能产生 mapping 冲突）。生产建议：显式 mapping + `dynamic: strict`（或 `false`），并在写入侧做强类型校验。

### 4.10 一次写入的耗时构成

```
总延迟 ≈ 网络RTT(2~4次) + mapping解析 + translog fsync + buffer写入
        + 副本并行写(与主分片并行，取最慢的) + 队列等待(线程池满时线性增长)
```

优化优先级：**bulk 化（减少 RTT 与 fsync 次数）> 加副本/主分片并行度 > 换更快磁盘 > 关掉不必要的索引结构 > 调大 refresh_interval（只影响可见性，不影响写延迟）**。

---

## 5. 查询链路：一次搜索如何跨越 N 个分片

### 5.1 一句话结论

> **默认是 query-then-fetch 两阶段：每个分片先在本地算出 top-K（只回传 docId + 排序值），协调节点全局归并出 top-N，再按 docId 回源取 `_source` 拼装结果。**

### 5.2 两阶段图解

```
Client ──"查 name=手机，from=0 size=10，按 price 降序"──▶ Coordinating Node
                                                              │
   ┌───────────────────────────── query 阶段 ────────────────┐│
   │  广播给 index 的所有分片（主或副本，按 ARS 选一个）        ││
   │  每个分片本地执行：倒排查找 → 过滤 → 打分 → 局部排序        ││
   │  只返回 [docId, sortValue] 的 top 10 + totalHits        ││
   └──────────────────────────────────────────────────────────┘│
                                                              ▼
   协调节点归并：按 sortValue 全局排序，取前 10 个 docId
                                                              │
   ┌───────────────────────────── fetch 阶段 ────────────────┐│
   │  用 docId 列表向对应分片发起 multi-get（只取这 10 条）    ││
   │  各分片读 _source（可能还要高亮、脚本字段、nested 展开）   ││
   └──────────────────────────────────────────────────────────┘│
                                                              ▼
                              组装 _source → 返回 Client
```

**为什么必须两阶段？** 因为 `_source` 可能很大（一条商品文档几十 KB）。若 query 阶段就让每个分片返回完整文档，10 个分片各返回 1000 条候选，网络传输会爆炸。**只传 docId（4 字节）和排序值，把"搬运大对象"推迟到确定最终结果之后**——这是 ES 最重要的分布式设计权衡之一。

### 5.3 DFS query-then-fetch：什么时候需要它

| 模式 | 阶段 | 打分/聚合准确性 | 代价 |
|---|---|---|---|
| **query-then-fetch**（默认） | 2 轮 | 用分片局部统计量，**近似但不精确** | 最快、网络最省 |
| **DFS query-then-fetch**（`search_type=dfs_query_then_fetch`） | 3 轮（先发 `_search_shards` 预查询收集全局 df/avgdl） | **全局统计量，打分精确、分片间可比** | 多一轮 RTT + 协调节点聚合统计 |

什么时候真的要开 DFS？

- 索引**分片很少、文档极少**（如测试环境、小索引），局部统计偏差极大导致排序诡异。
- 对**相关性排序的一致性有强要求**（同一查询换分片布局结果不能变）。
- 大量分片 + 打分是核心业务（学术检索）。

反过来说：**绝大多数互联网搜索不需要 DFS**，因为排序权重由业务/模型主导，BM25 只是其中一路特征；开了反而多一轮网络开销。

### 5.4 协调节点还能"跳过"分片

ES 有一系列 **can-match / 剪枝** 优化，让协调节点不必真的把查询发到所有分片：

- **时间范围剪枝**：`range` 查询在 `date` 字段上，若分片内该字段的 min/max 落在范围外（利用 BKD 的边界元数据），整片跳过。
- **索引排序剪枝**（`index.sort.field`）：段内按某字段有序，`sort` + `range` 组合时可提前终止扫描（early termination），甚至跳过整段。
- **路由过滤**：查询里带 `routing` 参数 → 只打对应分片（常见于"按用户 id 路由、按租户隔离"）。
- **`preference`**：把同一用户的查询固定到同一组分片/副本，提升 query cache 命中率（用户翻页时尤其明显）。
- **filter 缓存复用**：命中的 filter 结果可在多个请求间复用（见 5.5）。

### 5.5 filter vs query：不只是"算不算分"

| 维度 | query context | filter context |
|---|---|---|
| 是否打分 | 是（BM25） | **否**（只判断 true/false） |
| 是否缓存 | 否 | **是**（node 级 query cache） |
| 典型用途 | `match`、`multi_match`、`function_score` | `term`、`range`、`exists`、`bool.filter` |
| 性能 | 较慢（要算 tf/idf/norm） | 快（位图运算 + 缓存） |

**query cache 的真实生效条件**（面试常挖）：

- 只缓存 **filter context** 的查询，且查询作用于**单个 segment**。
- 只对**"足够大"的段**生效（小段缓存不划算，实现上要求段内文档数超过阈值）。
- 缓存的是 **docId 位图（bitset）**，因此命中后是纯位运算，极快。
- 段被 merge 后缓存失效（段对象变了）。
- 上限 `indices.queries.cache.size`（默认堆的 10%），LRU 淘汰。

**request cache**：另一个层级——缓存**整个 search 请求的结果**（`indices.requests.cache.size` 默认 1%）。key 是规范化后的请求体，只对 `size=0` 的聚合类请求或不依赖 `now` 的请求有稳定收益；滚动/实时排序场景几乎不命中。很多"聚合慢"的优化其实是"给聚合加 `request_cache=true`"。

> **实践准则**：把**所有纯过滤条件**放进 `bool.filter`，只让真正影响相关性的条件进 `bool.must`。这一条能同时降低 CPU、提升缓存命中、稳定延迟——是查询优化的第一优先级，比任何参数调优都有效。

### 5.6 深分页：三种方案与它们的代价

**问题本质**：`from=990, size=10` 时，每个分片都必须返回**前 1000 条**（因为全局第 990~1000 名可能来自任意分片），协调节点要归并 `分片数 × 1000` 条记录。

```
内存/CPU ≈ O(shards × (from + size))        网络 ≈ O(shards × (from + size))
```

所以：分片越多、翻得越深，成本线性上升；`from+size > 10000` 直接被 `max_result_window` 拒绝（不是怕慢，是怕 OOM）。

| 方案 | 原理 | 适用 | 代价/陷阱 |
|---|---|---|---|
| `from + size` | 每片取 from+size 归并 | 浅分页（前几页） | 深翻成本爆炸；上限 10000 |
| `scroll` | 首次查询生成**快照** + scroll_id，后续按批游标拉取 | 全量导出、离线任务 | 快照期间段无法回收/merge，占资源；**不能反映新写入**；不适合用户翻页 |
| `search_after` | 用上一页最后一条的**排序值**作为游标（无状态） | 用户连续翻页、深分页 | 必须**唯一且稳定**的排序（如 `sort: [price, _id]`），不能跳页 |
| `PIT + search_after`（推荐） | 先 `_pit` 固定一个时间点视图，再 `search_after` 翻页 | 生产深分页标准解法 | 需管理 PIT 生命周期（`keep_alive`），否则泄漏 |

**`search_after` 的坑（高分点）**：

- 排序键必须**全局唯一**。只按 `create_time desc` 排序，时间相同的文档顺序不稳定 → 翻页会重复或漏数据。必须补 `_id` 或 `_shard_doc` 作为 tiebreaker。
- 排序值要从 response 的 `sort` 数组里**原样回传**，自己重新计算（如把 `long` 转成 int）会精度丢失。
- 与 `from` **互斥**（同时用会报错）。
- 没有 PIT 时，翻页期间若有删除/merge，可能出现"游标指向的文档消失" → 用 PIT 固定视图。

### 5.7 聚合的分布式原理

聚合是"map-reduce"式的：**分片本地聚合 → 协调节点二次合并**。

```
terms 聚合（按 brand 分桶，取 top 10）：
  ├ 每个分片本地算 top 10 → 回传 10 个桶（含 doc_count、子聚合值）
  └ 协调节点把同名桶相加 → 全局排序 → 取 top 10

问题：某品牌在分片 A 排第 11、分片 B 排第 12，全局可能进前 10
     → 本地只取 10 会漏掉它（"丢失的桶"）
解法：shard_size 比 size 大（默认 size*1.5+10），让分片多回传一些候选
     → 精度换网络/内存；shard_size 越大越准，也越慢
```

**常见聚合的性能陷阱与解法**：

| 陷阱 | 原因 | 解法 |
|---|---|---|
| 高基数 `terms`（如按 userId 聚合） | 桶数爆炸，协调节点 OOM | 确认业务是否真要全量；用 `composite` 分页聚合；加 `filter` 缩小范围；`shard_size` 别乱调大 |
| `cardinality` 精确去重 | 精确去重需全量集合 | 用默认 HLL（近似，误差 ~1-2%，`precision_threshold` 提升精度但耗内存） |
| 嵌套聚合 + `nested` | 需展开 nested 文档，join 成本高 | 减少 nested 层级；考虑把需要聚合的维度平铺成独立字段 |
| 大范围 `date_histogram` | 每秒一个桶，桶数巨大 | 用 `calendar_interval`（1d/1h）而非 `fixed_interval` 过细粒度；限制 `extended_bounds` |
| 聚合卡在 merge | 深分页 + 大聚合同时跑 | 拆成两次请求；聚合请求设 `size: 0`；给聚合开 `request_cache` |
| 脚本聚合 | `painless` 逐文档执行 | 能用 doc values 字段就用字段；必须脚本时确认 `doc_values` 已开 |

**`global ordinals` 的取舍**（进阶）：跨段聚合需要把"同名 term"映射到同一个桶。ES 有两条路：加载 global ordinals（每字段一份，常驻堆，构建有成本，但聚合快），或在协调节点按字符串 map 合并（省堆但慢）。对"少字段、高频聚合"用前者，对"高基数、偶尔聚合"用后者（`execution_hint: map`）。

### 5.8 用 profile API 定位慢查询

不要靠猜。`GET /index/_search?pretty` 加 `"profile": true` 能看到每个分片、每个 Lucene 重写阶段、每个 `term`/`postings`/`doc_values` 的耗时与命中数：

```json
"profile": {
  "shards": [{
    "searches": [{
      "query": [{"type": "BooleanQuery", "time_in_nanos": 812345, "breakdown": {...}}],
      "rewrite_time": 12000,
      "collector": [{"name": "SimpleTopScoreDocCollector", "time_in_nanos": 500000}]
    }]
  }]
}
```

典型诊断路径：

- `next_doc` 耗时高 → 大量文档被扫描后丢弃，说明**过滤条件顺序/选择性差**（把高选择性的 `term` 放前面，或用 `filter`）。
- `advance` 耗时高 → 跳表跳得太多，通常是多条件 AND 中的长列表在驱动（把最短列表放前面）。
- `build_scorer` 耗时高 → 每次查询都在重建打分器，检查是否可用 `filter`（可缓存）。
- `doc_values` 耗时高 → 聚合/排序字段基数过高，或 `global_ordinals` 反复构建。
- `rewrite` 耗时高 → `prefix`/`wildcard`/`regexp` 展开成大量 term（前导通配的典型症状）。

### 5.9 自适应副本选择（ARS）

同一分片有多个副本时，协调节点选哪个？早期是**轮询**，问题很明显：如果某个副本所在节点正在做 merge、GC、或磁盘抖动，轮询会把 1/N 的请求打到最慢的节点，形成长尾。

ES 6.4+ 默认开启 **adaptive replica selection**：按 **EWMA 指数加权移动平均的服务时间 + 队列中的未完成请求数** 打分，选"预计最快"的副本（思路借鉴 Cassandra 的 c3）。这对"集群整体健康但个别节点抖动"的长尾问题效果显著。

---

## 6. 分布式与集群：ES 外壳的核心

### 6.1 一句话结论

> **ES 的分布式不是"数据库分库分表"，而是"把一批独立 Lucene 索引（分片）拼成一个逻辑索引"，靠路由决定写入落点，靠副本提供读扩展与故障接管，靠一个轻量主节点维护集群状态。**

### 6.2 路由：一个文档去哪

```java
// 实际实现（OperationRouting）：先取 routing 值算 murmur3 哈希，再取模
int hash = Murmur3HashFunction.hash(routingValue);   // routingValue 默认是 _id
int shardId = Math.floorMod(hash, numberOfPrimaryShards);
```

三个必须记住的推论：

1. **主分片数创建后不可改**。改了 `hash % N` 的 N，所有文档的路由全变，已有数据全部"找错家"。要改只能：`_reindex` 重建 / `_shrink`（减半，要求副本为 0 且只读）/ `_split`（倍增）/ rollover 建新索引 + alias 切换。
2. **自定义 `routing` 可以控制数据聚集**。多租户场景用 `routing=tenant_id`，让同一租户数据落同一分片，查询只打一片（省广播）。代价是**热点**：大租户会把单个分片打死。所以大厂常见做法是 `routing = tenant_id + 分桶后缀`（如 `tenant_id#0..9`），把大租户打散成 10 片。
3. **`_id` 会自动建索引**（`.id` 字段），用于"按 id 查/去重"。如果业务不需要按 id 查（纯日志、append-only），把 `_id` 显式给一个值并设置 `mapping._id` 索引关闭？——实际操作是**用 `index.mapping._source` 无关**，正解是"给 `_id` 赋值但不依赖它查询"或改用 `_index`+自增序号，能省一份索引开销（写入提升 10%~30%）。

### 6.3 分片分配与再平衡

**分片分配的约束条件**（任何一条不满足，分片就 unassigned）：

```
① 该分片没有"正在进行的其它副本/分配"（同一分片同一时刻只能有一个初始化）
② 节点有足够磁盘空间（受水位线约束，还要预留"未来可能增长"的空间）
③ 满足 allocation awareness（机架/可用区反亲和）
④ 满足 filter 规则（如 hot/warm 分层：冷索引不许落热节点）
⑤ 节点角色匹配（data 角色才能放分片；8.x 起角色划分更严格）
⑥ 不违反 same_shard.host（同一分片的主副本不能在同一物理机）
```

**关键机制**：

- **再平衡（rebalance）**：集群节点数变化或分配不均时，主节点计算"迁移哪些分片最划算"，然后下发。`cluster.routing.rebalance.enable` 可设为 `none/primaries/replicas/all`。
- **水位线是硬约束**：`low=85%` 停止往该节点分配新分片；`high=90%` 主动把分片迁走；`flood=95%`（`flood_stage`）强制索引**只读**（`index.blocks.read_only_allow_delete`）。这是生产事故"磁盘突然 95%，整个集群只读"的直接原因。
- **恢复（recovery）有节流**：`indices.recovery.max_bytes_per_sec`（默认 40MB/s 左右，8.x 按节点/集群区分）。大副本恢复时如果打满带宽会拖垮查询，需要按网络能力调。
- **排查神器**：`GET /_cluster/allocation/explain` 会逐条告诉你"这个分片为什么没分配"（"no valid node", "disk watermark exceeded", "node does not match filter"…）。这是线上排障第一个该敲的命令。

### 6.4 主节点选举：Zen2 到底做了什么

ES 的选主机制演进：**Zen（1.x-6.x，基于 `minimum_master_nodes` 的 Bully 式）→ Zen2（7.x 起，基于"版本比较 + 法定票数"）**。

**Zen2 的选举逻辑（面试要能讲清"它为什么不会脑裂"）**：

```
每个 master-eligible 节点持续广播自己的 cluster state version
  → 节点收集到足够多的"活跃 master-eligible 节点"响应（达到法定数）
  → 认为当前无主，发起选举
  → 候选节点必须获得【(N/2)+1】票才能当选（N = master-eligible 节点数）
  → 当选后成为 master，开始发布集群状态

关键：少数派永远凑不齐法定票
  → 网络分区时，少数派那一侧无法选出 master，也无法形成可写集群
  → 因此"两边各有一个 master 同时写"（传统脑裂）不会发生
```

再补两个 7.x/8.x 的重要细节：

- **`cluster.initial_master_nodes`**：只在**首次启动**引导时使用（告诉节点"初始候选名单"），之后应该从配置里删掉。如果集群重启时还留着它，可能引发"选举竞争/节点反复重启"的经典事故。
- **`discovery.type=single-node`**：单节点开发模式，跳过选主与引导，不能用于生产。
- **投票配置排除（voting config exclusions）**：下线节点前用 `_cluster/voting_config_exclusions` 把它排除，否则它会一直参与投票导致"master 选不出来"。

**与 Raft 的区别（高分点）**：ES 用"投票 + 版本比较"解决**选主**，但**集群状态复制不是 Raft 日志复制**——master 计算新状态后**广播（publish）给所有节点，节点 ack 版本号**，是"推送 + 确认"模型。所以 ES 的集群状态一致性是"版本单调 + 多数确认"，而不是 Raft 的日志顺序提交。这解释了为什么"集群状态更新"在节点多时会有秒级延迟。

### 6.5 集群状态（cluster state）里存什么

```
cluster state（二进制，压缩存储，每个节点常驻内存）
├── 元数据：索引名、mapping、setting、别名、模板、ILM 策略
├── 路由表：每个分片在哪个节点（routing table）
├── 节点信息：谁活着、角色、属性（机架、可用区）
└── 其它：snapshot 仓库、license、ingest pipeline
```

三个重要事实：

1. **主节点不是数据瓶颈**。它只维护元数据与分配决策，不处理文档读写。所以"master 节点挂了搜索不受影响"，只是**元数据变更（建索引、改 mapping、分片迁移）暂停**。
2. **集群状态变更 = 一次广播 + 全节点 ack**。因此**频繁的元数据变更**（每秒建一个索引、动态 mapping 不断新增字段、频繁 rollover）会推高 master 压力与全网延迟。这是"日志场景为什么必须用模板固定 mapping + ILM 而不是手建索引"的底层原因。
3. **`cluster.state` 体积随"分片数 + 索引数 + mapping 数"增长**。分片数上万、索引数上千时，每次 publish 都要序列化几 MB，master 的 CPU 与网络成为瓶颈——这是"为什么不能无脑建几千个索引/几万个分片"的硬理由之一。

### 6.6 节点角色（8.x 强制显式划分）

| 角色 | 职责 | 生产建议 |
|---|---|---|
| `master` | 集群状态、分片分配 | **专用**，3 个（奇数），不承担数据 |
| `data_hot/warm/cold/frozen` | 存数据，按温度分层 | 热层 SSD，冷层大容量 HDD/可搜索快照 |
| `data_content` / `data_search` | 8.x 细分的内容型/搜索型节点 | 按业务隔离，避免"日志聚合"和"核心检索"互相打 |
| `ingest` | 写入前 pipeline 处理（解析、富化、脱敏） | 与 data 分离，避免 CPU 抢占 |
| `coordinating`（仅协调） | 只做查询归并，不存数据 | 大集群可加，缓解 data 节点的归并压力 |
| `remote_cluster_client` | 跨集群搜索（CCS）/CCR | 按需 |

**为什么 master 要专用且奇数？** 专用避免 GC/merge 干扰导致"master 频繁失联 → 集群状态反复重发布"；奇数是"多数派"的必要条件（2 个节点的集群反而比 1 个更脆弱，因为任何 1 个挂掉都无法凑齐 2 票中的多数…实际上 2 节点时法定票是 2，挂一个就选不出）。

### 6.7 故障恢复：副本如何接管

```
主分片所在节点失联（超过 fault_detection 超时，默认 30s 级）
  → master 从该分片的 in-sync 副本中选一个，提升为 primary（promotion）
  → 旧 primary 若恢复回来，发现自己已不是 primary，会**丢弃本地数据**，
     重新作为副本从新 primary 做 peer recovery（全量/增量）
  → 期间该分片短暂 red/unassigned，客户端写入可能失败（需要重试）
```

**peer recovery 的两种模式**（面试常问"副本恢复是拷贝还是重放"）：

- **全量（file-based）**：直接拷 segment 文件（快，但吃网络/磁盘 IO）。
- **增量（translog-based）**：只重放缺失的 translog 区间（快，但要求两边的 commit point 有共同历史）。

### 6.8 脑裂：现象、成因、修复

**现象**：集群出现两个 master，各自接受写入，同一分片在两个分区里被独立修改 → 数据分叉（同一条文档两个版本），恢复后只能丢弃一侧。

**成因**：网络分区 + 配置错误（6.x 的 `minimum_master_nodes` 设错，如 3 节点集群设成 1）；或运维操作（同时重启两个 master）。7.x 后 Zen2 从机制上避免了"少数派选主"，但仍有其它坑：

- `cluster.initial_master_nodes` 残留 + 整集群重启 → 多节点同时自认为可以引导。
- 一个节点被手工用 `--master` 之类方式强行改变角色（版本差异）。
- 云环境网络抖动 + master 数不足（1 个 master 挂了无人可替）。

**修复**：先停写、隔离一个分区、用 `_cluster/state` 比对两个视图、选数据更全的一侧、另一侧重建。**ES 没有自动"合并分叉"能力**，所以"防"比"治"重要：3 个专用 master + 机架/可用区反亲和 + 正确的投票排除。

---

## 7. Mapping 与分词：决定"准不准"

### 7.1 一句话结论

> **mapping 决定一个字段"怎么被索引"（能不能搜、怎么搜、能不能聚合排序），分词器决定"文本被切成什么 term"。这两件事在写入时就固化，事后基本只能重建索引。**

### 7.2 text vs keyword：最基础也最容易错

| 维度 | `text` | `keyword` |
|---|---|---|
| 是否分词 | 是（走 analyzer） | 否（整串一个 term） |
| 可全文检索 | 是（`match`） | 否（只能 `term` 精确匹配） |
| 可排序/聚合 | **否**（无 doc values） | 是 |
| 典型字段 | 标题、正文、描述 | 状态、类型、标签、地区、SKU、email |
| 常见错误 | 用 `text` 做 `terms` 聚合 → 报错或结果错乱 | 用 `keyword` 做全文搜 → 搜"手机"搜不到"小米手机" |

**标准做法（双字段）**：

```json
"title": {
  "type": "text",
  "analyzer": "ik_max_word",
  "fields": { "raw": { "type": "keyword", "ignore_above": 256 } }
}
```

`title` 负责搜，`title.raw` 负责排序/聚合/精确匹配。`ignore_above` 防止超长字符串撑爆 doc values。

### 7.3 分词器三件套与中文分词

```
char_filter（字符级预处理：去 HTML、& 转 and、繁简转换）
    → tokenizer（切词：standard/whitespace/ik/jieba/ngram/pattern）
    → token_filter（词的加工：小写、去停用词、同义词、stemming、edge_ngram、pinyin）
```

**中文分词的现实选择**：

| 方案 | 特点 | 场景 |
|---|---|---|
| `standard` | 单字切分（中文按字） | 召回高、精度差，一般不单独用 |
| `ik_max_word` / `ik_smart` | 细粒度/粗粒度，词典可扩展 | 主流电商/内容检索 |
| `jieba` / `smartcn` | 社区插件 | 轻量场景 |
| `ngram`（2~3） | 无需词典，暴力切 | 兜底召回（品牌名、错别字） |
| `pinyin` 插件 | 拼音首字母/全拼 | 搜索框联想、人名容错 |

**搜索端 vs 索引端 analyzer**（高分点）：索引时用细粒度（`ik_max_word`，多切词提召回），查询时用粗粒度（`ik_smart`，少切词提精度），通过 `search_analyzer` 区分。这样"小米手机"索引成 `小/米/小米/手机`，查询"小米手机"按 `小米/手机` 匹配，召回与精度兼顾。

### 7.4 object vs nested vs join：关联数据的三条路

| 方案 | 存储形态 | 能否保持"数组内元素间的关联" | 代价 |
|---|---|---|---|
| `object`（默认） | **扁平化**成 `a.b`、`a.c` 多值字段 | **不能**（交叉匹配误命中） | 最省 |
| `nested` | 每个子对象变成**独立隐藏文档**，父子用 BlockJoin 关联 | 能 | 索引膨胀、查询要 `nested` 包裹、排序/聚合受限、写入更慢 |
| `join`（parent/child） | 同分片内父子文档 + `join` 字段 | 能（且子文档可独立更新） | 只能同分片（`routing` 必须一致）、查询要 `has_child/has_parent`、性能更差 |

**经典陷阱**：`{"orders":[{"sku":"A","num":1},{"sku":"B","num":2}]}` 用 `object` 存，查询 `sku=A AND num=2` 会**误命中**（因为扁平化后是 `sku:[A,B]`, `num:[1,2]`，条件分别命中不同数组元素）。这是"数组里带条件"必须 `nested` 的原因。

### 7.5 runtime fields 与字段设计

- **runtime fields**（7.11+）：不占索引空间，查询时用 `painless` 现算。适合"偶尔用、改了不用重建索引"的派生字段（如 `day_of_week`）。代价是每次查询都要算，**不能用于高频过滤/sort**（除非用 `emitted` 缓存，仍不如索引）。
- **`fields` API**（7.10+）替代 `docvalue_fields`/`stored_fields` 的组合，统一取"索引值或 runtime 值"。
- **字段设计 checklist**：只搜不聚合 → `text`；只过滤 → `keyword`/数值；不搜不聚合只要返回 → `index: false, doc_values: false`（省一半空间）；超大文本 → 考虑 `_source: false` + `store` 或干脆不入 ES。

### 7.6 向量检索（ES 8.x 的能力边界）

```
dense_vector(dims=768, index=true, similarity=cosine)  → HNSW 图索引（Lucene 9）
knn 查询：先过滤（filter）→ 在候选集上做 HNSW 近似最近邻 → 返回 top k
num_candidates 控制"每分片探索多少候选"，越大越准越慢
```

工程要点：HNSW 索引**吃内存**（图结构常驻）；**过滤 + 向量混合**时如果过滤后候选太少会退化为暴力扫描；分片数影响召回（每分片独立 HNSW，需 `num_candidates` 补偿）。所以"纯向量检索"场景，专用向量库（Milvus/Qdrant）在超大规模下仍有优势；ES 的价值是"全文 + 向量 + 过滤 + 聚合"一站式。

---

## 8. 内存与存储：容量决策的物理边界

### 8.1 堆内存：为什么是"50% 且不超过 31GB"

```
物理内存
├── JVM 堆（≤50%，且 ≤ ~31GB）
│    ├── segment 元数据、FST（term index）
│    ├── IndexWriter buffer（10% 堆）
│    ├── query cache（10% 堆）、request cache（1% 堆）
│    ├── fielddata（禁止 text，keyword 聚合时可能用）
│    └── 各种 breaker 监控的临时对象（聚合桶、排序队列）
└── 剩余 50%+ → OS page cache（Lucene 读 .tim/.doc/.dvd/.fdt 全靠它）
```

两条硬理由：

1. **Lucene 用 mmap 读索引文件**，热数据靠 page cache。堆给太多，page cache 就少，读放大变多，查询反而变慢。**"堆越大越好"是 ES 最经典的错误直觉。**
2. **31GB 是压缩指针（compressed oops）的边界**。超过 32GB 后，对象引用从 4 字节变 8 字节，实际可用堆"倒退"。所以要么 ≤31GB 单节点多实例，要么 >32GB 时接受指针膨胀（一般不建议跨越）。

**堆压力从哪来（面试常问"ES 为什么 OOM"）**：

| 来源 | 症状 | 对策 |
|---|---|---|
| FST / term 基数爆炸 | 堆缓慢上涨，GC 后不回落 | 高基数字段 `index:false`；别把日志原文建成 `text` |
| 聚合桶（高基数 terms） | 请求级 OOM，`circuit_breaking_exception` | 加 `filter` 缩范围；`composite` 分页；`search.max_buckets` 限制 |
| 深分页 | 排序队列占堆 | `search_after`/PIT |
| fielddata（误用） | 堆瞬间涨满 | 用 `keyword`；开 `fielddata` 必须监控 |
| 大 bulk 请求 | 请求体在堆里构造 | 控制单批体积；`http.max_content_length` |
| 分片过多 | 每分片都有 buffer/元数据开销 | 控制分片数（经验：**每 GB 堆 20~30 个分片上限**） |

### 8.2 断路器（circuit breaker）：ES 的"主动拒绝"

```
parent breaker（总闸，默认 95% 堆）
 ├── fielddata breaker（默认 40% 堆）
 ├── request breaker（默认 60% 堆，聚合/排序/分页的临时内存）
 └── in-flight requests breaker（并发请求体总量）
```

命中时抛 `circuit_breaking_exception`（HTTP 429），并明确告诉"estimated bytes / limit"。**这是"设计上的自我保护"，不是 bug**——遇到它应该减少单请求的数据量，而不是调大阈值（调大只是把"可控拒绝"换成"JVM OOM 崩节点"）。

### 8.3 磁盘：不只存数据，还存"余量"

```
磁盘需求 ≈ 原始数据 × (1 + 副本数) × 索引放大(1.5~3x) × merge 峰值(≤2x) × 水位余量(1/0.85)
例：100GB 原始、1 副本、索引放大 2x、水位 85%
  → 100 × 2 × 2 = 400GB，再除以 0.85 ≈ 470GB 可用空间
```

**分层存储（ILM + data tiers）**：

```
hot（7 天内，SSD，1 副本，频繁写查）
 → warm（30 天内，HDD，force merge 到 1 段，只读，可降副本）
 → cold（90 天内，大容量 HDD，可关闭）
 → frozen / delete（可搜索快照：只把元数据放本地，数据按需从对象存储拉取）
```

**可搜索快照（searchable snapshot）**是"冷数据成本杀手"：索引本体在 S3/OSS，本地只留少量结构，查询时按需加载。代价是首次查询延迟高（要拉取）。适合"合规要求保留 1 年但几乎不查"的数据。

---

## 9. 容量规划与调优手册

### 9.1 分片怎么定（最常被问的"你怎么规划"）

**核心原则：先按数据量定分片大小，再按节点数定副本，最后用索引数控制总量。**

```
① 单分片目标 20~50GB（业务检索）；日志类 30~50GB（写多查少）
② 主分片数 = 预估数据总量 / 单分片大小（向上取整，且尽量是节点数的整数倍，
   让分片均匀落盘；6 分片 3 节点是 2:2:2，7 分片就会 3:2:2 不均）
③ 副本数：核心检索 ≥1；日志可 0~1；读多写少可 2
④ 分片总数硬约束：
   - 每 GB 堆 ≤ 20~30 个分片（经验值，官方文档给过 20）
   - cluster.max_shards_per_node 默认 1000（7.x），超了建索引直接失败
⑤ 时间序列数据用 data stream + ILM rollover，按"天/周"建索引，
   绝不用"一个索引存一年"
```

**反模式**：默认 `number_of_shards: 5` 一把梭。10GB 数据建 5 分片，每个分片 2GB，段小、merge 频繁、协调开销占比高；而 1TB 数据建 5 分片，单分片 200GB，恢复要几十分钟，一台机器挂了影响面巨大。

### 9.2 写入调优清单

| 动作 | 收益 | 风险 |
|---|---|---|
| `_bulk` 批量（5-15MB/批） | 吞吐提升一个数量级 | 单批过大触发 429 |
| 导入期 `refresh_interval: -1` | 减少段生成与 merge | 导入期间不可搜索 |
| 导入期 `number_of_replicas: 0` | 写入减半 IO | 期间无冗余 |
| 显式 `_id` 而非自动生成？**反过来**：不需要按 id 查时，避免额外 `.id` 索引 | 写入 +10~30% | 失去按 id 直查能力 |
| `index_options: docs`（只过滤的文本字段） | 索引体积 -20~40% | 失去 phrase/高亮 |
| 不需要的字段 `index: false` | 体积与写入双降 | 不能搜 |
| `_source` 用 `best_compression` | 存储降 30%+ | 解压 CPU 略增 |
| 加大主分片数（并行 fsync） | 写入线性扩展 | 分片过多反噬查询 |
| SSD/NVMe + 独立磁盘（data 与 translog 分开） | fsync 更快 | 成本 |

### 9.3 查询调优清单

| 动作 | 说明 |
|---|---|
| 过滤放 `bool.filter` | 不打分 + 可缓存，最有效的一条 |
| 避免前导通配 `*abc` | 走不了 FST，全 term 扫描；改 `ngram` 或 `match_phrase_prefix` |
| `size: 0` 做聚合 | 不取 `_source`，省 fetch |
| `track_total_hits: false` | 不需要精确总数时省掉计数开销 |
| `_source: false` / `_source_includes` | 只取需要的字段 |
| 深分页用 `search_after` + PIT | 避免 O(shards × from) |
| 聚合加 `filter` 缩范围、用 `composite` 分页 | 控制桶数 |
| 高频相同查询用 `preference` 固定副本 | 提升 query/request cache 命中 |
| 用 `profile` + 慢日志定位 | `index.search.slowlog.threshold.query.warn: 10s` |

### 9.4 生产故障速查表（面试"你遇到过什么问题"的答案库）

| 现象 | 常见原因 | 排查/处置 |
|---|---|---|
| 集群 red | 主分片未分配（磁盘满/节点掉/规则冲突） | `_cluster/allocation/explain` → 看原因逐条解 |
| 集群 yellow | 副本未分配（只剩 1 节点/磁盘不够） | 同上，或临时降副本 |
| 写入 429 `rejected` | write 线程池队列满（默认 1000） | 降并发/加 bulk 大小/扩分片；客户端指数退避 |
| 查询变慢但 CPU 不高 | page cache 不足、段太多、merge 打 IO | 看 `_nodes/stats` 的 `segments`、`merges`、`io`；查 force merge/大 bulk |
| 堆持续高位、GC 频繁 | 分片过多、FST 太大、聚合桶泄漏 | `_nodes/stats/jvm` + `hot_threads`；查 fielddata/breaker 指标 |
| 磁盘突然 95% 只读 | flood 水位线 | 立即清理/扩容（`_cat/allocation`）；临时调水位线只是缓兵 |
| master 频繁切换 | master 专用性不足/GC 停顿/网络抖动 | 3 专用 master、调大 `discovery` 超时、查 GC 日志 |
| 副本恢复慢 | `recovery.max_bytes_per_sec` 限流 | 按网络能力调；避免同时恢复大量分片 |
| 搜索结果抖动/排序不稳 | 分片局部统计（BM25）+ 排序键不唯一 | `search_after` 加 `_id` tiebreaker；必要时 DFS |
| 同一条数据"更新后还能搜到旧值" | refresh 未生效 / 查的是副本上的旧 reader | 用 `refresh=wait_for`；确认 `_seq_no` |
| delete 后空间没降 | .liv 标删，未 merge | `_forcemerge?only_expunge_deletes=true`（只读索引安全） |

---

## 10. 选型对比：ES 的边界在哪

| 对比项 | Elasticsearch | Solr | ClickHouse | MongoDB | 专用向量库（Milvus 等） |
|---|---|---|---|---|---|
| 底层 | Lucene | Lucene | 自研列存 | WiredTiger | HNSW/IVF |
| 强项 | 全文检索 + 聚合 + 生态 | 全文检索、成熟稳定 | 大宽表扫描、聚合极快 | 文档 CRUD、事务 | 十亿级向量召回 |
| 弱项 | 写入放大、深分页、JOIN 弱 | 分布式需 ZooKeeper、运维重 | 全文弱、更新代价高 | 全文与聚合弱 | 无全文/聚合 |
| 实时性 | 近实时（1s） | 近实时 | 准实时（批次/微批） | 实时 | 实时 |
| 典型用途 | 搜索、日志、可观测、聚合分析 | 企业检索、电商 | OLAP、日志分析、指标 | 业务库 | 以图搜图、语义检索 |

**一句话选型结论**：

- "既要搜索、又要多维过滤和聚合、还要高可用" → ES。
- "只做海量日志/指标的聚合分析，不要求全文相关性" → ClickHouse 更省。
- "以向量召回为主（十亿级）" → 专用向量库 + ES 做过滤与展示，或 ES 8.x kNN（千万级以内更合适）。
- "事务/强一致是核心" → 数据库为准，ES 只做异步检索副本（CDC/binlog 同步 + 幂等 `_id`）。

---
# 第三部分：国内大厂 Elasticsearch 面试题精讲

> **说明**：以下题目按"公开面经 + 岗位技术要求"归类整理，按**考察方向**而非"某公司某年原题"组织。同一道题在阿里、字节、美团、腾讯的表述会略有差异，但考的都是同一个底层点，所以按"能力档位"分三档，并在每题标注**最常出现在哪类团队**。
>
> 答题建议统一用四段式：**结论 → 结构/流程 → 为什么这样设计（代价） → 工程取舍/线上现象**。只答第一段的人，通常会被判定"背过八股但没落地"。

## 各厂考察风格对照（用来决定"往哪个方向深挖"）

| 团队/公司画像 | 常考重点 | 典型追问风格 |
|---|---|---|
| 阿里 / 蚂蚁（搜索、电商中台，P6-P7） | 倒排与段结构、写链路、mapping 治理、线上故障定级 | "你说的这一层在源码里是哪个类？线上出过什么问题？" |
| 字节（数据平台、搜索中台） | 写放大、merge 策略、分片规划、压测与容量、性能量化 | "给你 500 亿文档，怎么定分片？瓶颈在哪？" |
| 腾讯（TEG/PCG 后台） | 集群运维、监控告警、故障恢复、多租户隔离 | "集群 red 了，你按什么顺序处理？" |
| 美团 / 京东 / 拼多多（商品与订单搜索） | 深分页、聚合、别名与零停机重建、大促写入 | "怎么在不影响线上搜索的情况下改 mapping？" |
| 滴滴 / 高德（时空数据） | geo 查询、BKD 树、轨迹检索、routing 设计 | "10 亿个 geo_point 怎么查附近司机？" |
| 快手 / 小红书 / B站 / 知乎（内容检索） | 相关性、分词与召回、向量检索、多路融合 | "全文 + 向量混合召回怎么设计？" |
| 百度 / 华为（引擎与基础软件） | 倒排压缩、FST、内存模型、段文件格式 | "FST 为什么能后缀共享？体积怎么估？" |
| 网易 / 携程 / 贝壳（日志与可观测） | ILM、data stream、冷热分层、成本治理 | "一年 5PB 日志，存储成本怎么降一半？" |

---

## 第一档 ★☆☆ 基础必答（答错直接减分）

### A1. Elasticsearch 是什么？它和 Lucene 是什么关系？

**考核程度**：★☆☆ ｜ **高频团队**：全部

**答题框架**：

1. **定位**：ES 是一个基于 Lucene 的**分布式、近实时、RESTful** 搜索与分析引擎。它不只做全文检索，还承担聚合分析、日志存储、向量检索。
2. **分层**：Lucene 是**单机索引库**（Java 库，无分布式、无 REST、无副本、无集群），提供了倒排索引、段存储、打分、查询执行这些"引擎能力"；ES 在它之上加了**分布式层**（分片/副本/路由）、**集群管理层**（master/状态/分配）、**接口层**（REST/DSL/聚合）、**生态层**（Kibana/Logstash/Beats/ILM）。
3. **一句话分界**：凡是"索引结构、段、FST、BKD、doc values、BM25"属于 Lucene；凡是"分片、副本、集群状态、深分页限制、routing、reindex"属于 ES。

**延伸追问**：*"那 OpenSearch 呢？"* —— 2021 年从 ES 7.10.2 fork 出来的 AWS 主导分支，底层仍是 Lucene，架构同源；差异在治理模式、默认安全策略、部分功能（如异步搜索、跨集群复制）的演进节奏。回答时强调"同源不同治理"，比背版本号更显得懂。

---

### A2. 什么是倒排索引？为什么它比数据库的 LIKE 快？

**考核程度**：★☆☆ ｜ **高频团队**：全部（百度/华为会继续深挖到压缩算法）

**答题框架**：

1. **定义**：正排是"文档 id → 文档内容"，倒排是"**词项 term → 包含它的文档 id 列表（posting list）**"。搜索时按 term 直接定位候选集，再对候选集做布尔运算（AND/OR/NOT）与打分。
2. **为什么快（三层原因，答全才满分）**：
   - **不扫全表**：`LIKE '%手机%'` 必须逐行读字符串做匹配，成本 O(行数 × 字段长度)；倒排是"查词表 → 取列表"，成本与"命中量"相关，与"总行数"基本无关。
   - **结构极紧凑**：docId 升序 + delta 编码 + 位宽打包，一个 docId 平均只占几 bit；`LIKE` 要读完整字符串。
   - **求交有算法**：跳表跳跃 + 短列表驱动 + 位图按位与，让"多词 AND"从 O(n) 降到近似 O(n/128)。
3. **代价**：索引体积显著大于原始数据（常见 1.5~3 倍），写入要分词建索引，且**不支持"任意子串/正则"的高效匹配**（前导通配退化为全扫描）。

**追问**：*"那 LIKE '手机%' 前缀匹配呢？"* —— 数据库可以走 B 树前缀；ES 里 `prefix` 查询也能走 FST（因为 FST 按字典序），但 `*手机` 不行。这个对比能体现你真理解"为什么是字典序结构"。

---

### A3. text 和 keyword 有什么区别？什么时候用哪个？

**考核程度**：★☆☆ ｜ **高频团队**：全部（美团/京东必问，因为直接影响聚合正确性）

**标准答案要点**：

| | `text` | `keyword` |
|---|---|---|
| 分词 | 是 | 否 |
| 检索方式 | `match`（全文） | `term`（精确） |
| 排序/聚合 | 不支持（无 doc values） | 支持 |
| 用途 | 标题、正文 | 状态、标签、枚举、SKU |

**必须补的一句**：生产上最常用的是**同一字段建双索引**（`title` 为 text + `title.raw` 为 keyword），一个负责搜、一个负责排序聚合。

**追问**：*"给 text 字段排序为什么会报错？"* —— 因为 text 分词后一个文档对应多个 term，**不存在"一文档一值"的列**，无法生成 doc values；`fielddata` 虽能强行构建列，但会把所有 term 全量驻留堆内存，是 OOM 常客，官方已限制 text 开启。

---

### A4. 描述一下 ES 写入一条文档的完整流程。

**考核程度**：★☆☆（但答全了就是 ★★☆）｜ **高频团队**：全部

**答题框架（8 步，缺一步就显浅）**：

```
1. 客户端请求到任意节点 → 该节点成为 coordinating node
2. 协调节点按 mapping 校验文档，计算路由 hash(_routing/_id) % 主分片数
3. 请求转发到 primary shard 所在节点
4. primary 写 translog（durability=request 时立即 fsync）
5. primary 写 Lucene IndexWriter 的内存 buffer（此时不可搜索）
6. primary 将变更同步给该分片的所有 in-sync 副本（副本各自做 4、5 两步）
7. 等待副本 ack（受 wait_for_active_shards 约束）→ 返回客户端
8. 之后异步发生：refresh（1s）→ 生成新段、可搜索；translog 到 512MB/30min → flush（Lucene commit）
```

**加分点**：明确区分"**第 4-5 步保证不丢，第 8 步才决定能不能搜到**"，并指出"副本同步是**分片内**的同步复制，不是集群级事务"。

**追问**：*"如果第 7 步时某个副本挂了？"* —— 该副本被标记为失效，主分片继续服务（只要 `wait_for_active_shards` 满足）；集群变 yellow；后续该副本恢复时通过 peer recovery（全量拷文件或增量重放 translog）追平。

---

### A5. refresh、flush、merge 分别是什么？有什么区别？

**考核程度**：★☆☆ ｜ **高频团队**：字节、腾讯（这三个概念混淆是最常见的"假懂"）

**一句话回答**：

- **refresh**：把 buffer 里的文档写成**新段**并重开 reader → 解决"**能不能搜到**"，默认 1s。写完**没有 fsync**。
- **flush**：refresh + **fsync 段文件** + 写新 commit point + 清空 translog → 解决"**恢复要多久**"（缩短 replay），由 translog 512MB/30min 触发。
- **merge**：把多个小段合成一个大段，丢弃已删文档 → 解决"**查询效率与空间回收**"，后台策略触发。

**关键洞察（这句最能体现水平）**：
> "refresh 和 flush 是两个正交的开关：调大 `refresh_interval` 只降低可见性频率、**不影响持久性**；把 `translog.durability` 改成 async 只降低持久性、**不影响可见性**。"

**追问**：*"为什么 refresh 不顺手 fsync？"* —— fsync 是磁盘级操作，代价比写 page cache 高一到两个数量级；每秒对全部分片 fsync 会让写入吞吐崩掉。所以 ES 用 translog 承担"耐久性"，让 refresh 只承担"可见性"，这是把两个目标**解耦**的经典设计。

---

### A6. ES 为什么叫"近实时"？1 秒延迟是怎么来的？

**考核程度**：★☆☆ ｜ **高频团队**：全部

**答案**：因为文档写入后先在内存 buffer 里，只有 `refresh` 把它落成段并打开新的 IndexReader 后才对搜索可见，默认 `refresh_interval=1s`，所以最坏延迟约 1 秒，平均 0.5 秒。这就是"near real-time（NRT）"。

**为什么不做成实时**：每次写入都立刻建段会产生海量微小段（每秒每分片一个段），段数爆炸导致查询遍历成本剧增、merge 压力巨大。**用 1 秒的可见性延迟换写入吞吐与查询效率**，是刻意的权衡。

**追问**：*"有办法做到写完立刻能查吗？"* —— ① `refresh=wait_for`（等下个周期，不打乱节奏，推荐）；② `refresh=true`（强制立即 refresh，昂贵，只适合低频场景）；③ **按 `_id` 的 GET 本身就是实时的**（主分片直接从 translog 读，无需 refresh）——这一点很多人不知道，是高分细节。

---

### A7. 集群 yellow 和 red 有什么区别？分别怎么处理？

**考核程度**：★☆☆ ｜ **高频团队**：腾讯、网易（运维向必问）

**答案**：

| 状态 | 含义 | 影响 |
|---|---|---|
| green | 主分片与副本全部分配 | 正常 |
| yellow | **主分片全部分配，但有副本未分配** | 读写正常，**无冗余**（再挂一个节点就丢数据） |
| red | **至少一个主分片未分配** | 该分片的数据**不可读写**，搜索结果缺片 |

**处理顺序（体现工程素养）**：

1. `GET /_cluster/health?level=indices` 定位到具体索引；
2. `GET /_cluster/allocation/explain` 逐条看"为什么没分配"（磁盘水位/节点角色/filter 规则/同一分片已有一份）；
3. 常见处置：扩容或清理磁盘（水位线）、检查 `cluster.routing.allocation.*` 规则、`POST /_cluster/reroute` 手动分配（**高风险，仅在明确知道副本完整时**）、临时降副本数（yellow 场景最快的止血）。
4. red 且副本也没了 → 只能从 snapshot 恢复，或用 `_recovery` 的 `allow_partial` 类手段抢救（要接受数据不完整）。

**延伸追问**："red 时能不能直接 reroute 到空节点？" —— 不能瞎来：如果该分片的所有副本都丢失，reroute 会把分片分配成"空的"，之后再用旧数据覆盖就会**丢失真实数据**。正确动作是先 snapshot 校验、确认损失范围，再决定"接受空分片"还是"从备份恢复"。

---

### A8. 分片和副本各解决什么问题？副本能接收写请求吗？

**考核程度**：★☆☆ ｜ **高频团队**：全部

**答案**：

- **分片（primary shard）**：解决**水平扩展**——把数据与写入/查询负载分散到多节点。分片数是"单索引的并行度上限"，创建后**不可修改**（改则路由失效）。
- **副本（replica shard）**：解决**高可用**（主挂了升主）+ **读扩展**（副本可分担搜索）+ **容错**（单节点故障不丢数据）。
- **副本不接收写请求**：所有写入必须打到主分片，主分片再**同步转发**给副本。副本只服务读（search/get），且 ES 会用 ARS 选择"预计最快"的副本来响应。

**追问**：*"副本越多，写入越快还是越慢？"* —— 越**慢**。每次写都要等所有 in-sync 副本 ack，副本越多，写延迟的尾部风险越大、网络与磁盘 IO 成倍增加。副本买到的是"读吞吐 + 可靠性"，不是"写能力"。这也是"导入数据时先关副本"的原因。

---

### A9. 为什么 `from + size` 超过 10000 就被拒绝？深分页该怎么解决？

**考核程度**：★☆☆（方案部分是 ★★☆）｜ **高频团队**：美团、京东、字节

**答案结构**：

1. **原因**：`from=9990, size=10` 时，**每个分片都必须返回前 10000 条候选**（因为全局 9990~10000 名可能来自任意分片），协调节点要归并 `分片数 × 10000` 条记录。内存与耗时是 O(shards × (from+size))，深翻几页就能把协调节点打爆。`index.max_result_window=10000` 是**保护性硬限制**，不是性能调优点。

2. **三种解法（必须说清代价）**：

| 方案 | 机制 | 适用 | 代价 |
|---|---|---|---|
| `scroll` | 拉快照 + scroll_id 游标 | 全量导出/离线 | 快照占资源、看不到新数据、不适合用户翻页 |
| `search_after` | 用上一页最后一条的**排序值**当游标（无状态） | 连续翻页 | 排序键必须唯一稳定、不能跳页 |
| **PIT + `search_after`** | 先固定视图再翻页 | **生产标准方案** | 需管理 PIT 生命周期 |

3. **必答的坑**：只按 `create_time desc` 排序会因"时间相同"导致重复/漏数据，必须补 `_id` 或 `_shard_doc` 做 tiebreaker；`sort` 值要**原样回传**，自己转换会精度丢失；`search_after` 与 `from` 互斥。

---

### A10. 分词器由哪几部分组成？中文分词你怎么选？

**考核程度**：★☆☆ ｜ **高频团队**：快手、小红书、京东（内容与商品搜索）

**答案**：

- 三件套：**char_filter**（字符级预处理：去 HTML、繁简转换、`&`→`and`）→ **tokenizer**（切词）→ **token_filter**（小写、停用词、同义词、词干、`edge_ngram`、拼音）。
- 中文选择：`ik_max_word`（细粒度、高召回，索引端）/ `ik_smart`（粗粒度、高精度，查询端）；品牌/型号/错别字靠 `ngram` 兜底；搜索框联想用 `edge_ngram` 或 `pinyin`。
- **关键设计**：`analyzer`（索引）与 `search_analyzer`（查询）**分开配置**，索引端多切词提召回、查询端少切词提精度。

**追问**：*"同义词怎么处理？"* —— 用 `synonym` token filter（索引时扩展 vs 查询时扩展的取舍：索引时扩展体积大但查询快，查询时扩展灵活但 CPU 高；同义词词典更新需要 `_reload_search_analyzers`）。

---

### A11. `_bulk` 是原子的吗？部分失败怎么办？

**考核程度**：★☆☆ ｜ **高频团队**：全部（容易被当成"送分题"却答错）

**答案**：

- **不是原子的**。bulk 只是一次"打包传输"，内部每条 action 独立生效、独立返回状态；没有跨文档事务、没有回滚。
- 必须检查响应里的 `errors` 字段，以及 `items[].index.status`（201 成功 / 409 版本冲突 / 400 mapping 错误 / 429 队列满）。
- 处理策略：**429/503 用指数退避重试**；400 类（mapping 冲突、字段类型错）**不要重试**，要修数据或改 mapping；409 视业务决定是否重试。
- 单批建议 5~15MB 或 1000~5000 条，过大反而因队列满/长尾而变慢。

---

## 第二档 ★★☆ 进阶区分（答出"为什么"和"代价"）

### B1. 从 buffer 到磁盘，一条文档经历了哪些数据结构变化？写放大从哪来？

**考核程度**：★★☆ ｜ **高频团队**：字节、阿里、华为

**完整链路**：

```
JSON 文档
 → mapping 解析（类型转换、日期格式化、ignore_above 判断）
 → 分词（text 字段）→ 得到 term 流
 → 写入内存 buffer 中的"倒排结构"（每个字段的 term→docId 增量表）
 → refresh：排序、编码、写盘
     ├ .tim/.tip：term dictionary + FST（term 必须有序才能建）
     ├ .doc/.pos：posting list（delta + 位宽打包）
     ├ .dvd/.dvm：doc values（列存 + ordinal）
     ├ .fdt/.fdx：_source（默认 DEFLATE 压缩）
     ├ .nvd/.nvm：norms（1 byte/字段/文档）
     └ .kdd/.kdi：数值/geo 的 BKD 树
 → merge：读旧段 → 写新段 → 丢弃 .liv 中的死文档
```

**写放大三层**（这是"为什么磁盘要按 3~5 倍估"的答案）：

1. **副本放大**：1 主 1 副本 = 2 倍。
2. **结构放大**：倒排 + doc values + `_source` + norms ≈ 1.5~3 倍。
3. **merge 重写**：分层合并使每个字节平均被重写 1.5~3 次。

**追问**：*"为什么 term 必须有序才能建 FST？"* —— FST 是"按字典序逐条插入构建最小自动机"的，无序插入需要反复重建状态图。所以 Lucene 只能在段写完后、term 已全量可知时构建——**这就是段不可变的又一个根因**。

---

### B2. 更新和删除在底层是怎么实现的？为什么说 update 比 index 贵？

**考核程度**：★★☆ ｜ **高频团队**：阿里、美团、拼多多

**答案**：

- **删除**：不删字节，只在 `.liv`（live docs 位图）里把对应 docId 位清 0。查询时命中但**输出前过滤**，空间要等 merge 才回收。
- **更新**：**旧版标删 + 新版写入当前段**（两条完整记录并存直到 merge）。所以一次 update ≈ 一次 delete + 一次 insert，成本高于 insert，而且**短期磁盘占用会上升**。
- **由此导出的工程结论**：
  1. 高频覆盖同一文档（如"库存状态每秒刷一次"）会造成严重写放大与段膨胀，应改为"追加 + 外部版本号过滤"或换存储。
  2. `delete_by_query` 之后磁盘不降是正常的，需 `_forcemerge?only_expunge_deletes=true`。
  3. `_update` 的"读-改-写"需要乐观锁（`if_seq_no`）或 `retry_on_conflict`，否则并发更新会互相覆盖。

**追问**：*"能不能只更新一个字段而不重写整条文档？"* —— 不能。Lucene 的段不可变，`_source` 也是整条存储，所以"改一个字段"必然整条重写。这也是"宽文档（几百字段）在 ES 里更新代价很高"的原因。

---

### B3. FST 是什么？为什么能省内存？为什么不用 HashMap 或 B 树？

**考核程度**：★★☆ ｜ **高频团队**：百度、华为、阿里（P7 常追问到实现）

**答案**：

1. **是什么**：有限状态转换机（Finite State Transducer），把"term → block 偏移"建成一个**确定性、无环、带输出**的状态图。查询时沿字符走状态转移，走完看是否为接受状态并取输出值。
2. **为什么省**：① **前缀共享**（`cat/car/care` 共享 `ca`）；② **后缀共享**（多个词共享尾部状态，这是 Trie 做不到的）；③ 状态转移用紧凑数组/位打包存储，边不重复存字符。
3. **为什么不用 HashMap**：上亿 term 的字符串 + 对象头 + 指针会在堆里吃掉几个 GB，且长生命周期大对象让 GC 极痛；HashMap 也不支持 `prefix`/`range`。
4. **为什么不用 B 树**：B 树每条边要存完整 key 片段，体积远大于 FST；FST 在"只读 + 字典序 + 需要前缀/范围"场景是近乎最优。
5. **能力边界（关键）**：FST 只能"从前向后"走，所以 `prefix`、`range`、`fuzzy`（有限）可以，**前导通配 `*abc` 不行**——这就是"别用前导通配"的底层原因。
6. **体积量级**：常见为词表的几十分之一，几千万 term 的索引 FST 常在几十~几百 MB，因此能常驻堆；但**基数失控时它会成为堆杀手**。

**追问**：*"FST 在堆里还是 mmap？"* —— ES/Lucene 把 FST 读入**堆内存**（`ByteArrayIndexInput`），因此 term 数与堆占用强相关。这是"高基数字段为什么不能随便 `text`"的隐藏原因。

---

### B4. 倒排列表怎么压缩？多个词的 AND 为什么快？

**考核程度**：★★☆ ｜ **高频团队**：百度、字节、华为

**答案**：

1. **docId 升序** → **delta 编码**（存差值，数值变小、位宽变窄）。
2. **定长块 + 位宽打包**：每块（Lucene 典型 128 个 docId）只记录"本块最大值需多少 bit"，块内按该位宽紧凑排列；块头还存"块内最后一个 docId"，便于跳跃。
3. **稀疏用数组 + 跳表，稠密用位图**：命中率高的列表转成 bitset（甚至 Roaring：数组/位图/RLE 三种容器自适应），布尔运算变成按位与/或，极快。
4. **求交算法**：按 docFreq 升序，**最短列表驱动**；在长列表上用跳表"跳块 + 块内二分"；只有 docId 相等才输出。
5. **量化**：两个 100 万 docId 的列表，纯线性比对约 100 万次；跳表分块后约降到 1/100 量级。

**追问**：*"为什么不用压缩率更高的通用压缩（如 gzip）？"* —— 因为查询需要**随机跳读**（skip to docId），通用压缩必须从头解压，无法随机访问。Lucene 的编码是"可随机访问的轻量压缩"，这是"存储格式必须服务于访问模式"的典型例子。

---

### B5. doc values 是什么？为什么排序/聚合不用倒排索引？

**考核程度**：★★☆ ｜ **高频团队**：全部（聚合题的前置）

**答案**：

- **doc values 是列式存储**，索引时随段生成，每个字段一列，按 docId 顺序连续存放，便于顺序扫描与压缩。
- **倒排不适合"按字段取值"**：倒排是 term → docId，同一个字段的值散落在各个 term 的 posting 里；要取"一列值"必须遍历该字段所有 term 的所有 posting，再做归并，成本高几个数量级。
- **字符串列先转 ordinal**：`"北京"→0`，列里只存 int，聚合按 ordinal 分组，最后映射回字符串——这是 `terms` 聚合快的核心。
- **跨段聚合需要 global ordinals**：保证同名 term 在各段是同一编号；要么加载 global ordinals（吃堆、构建有成本、聚合快），要么协调节点按字符串 merge（省堆、慢）。`execution_hint` 就是在做这个取舍。
- **代价**：索引体积增加、写入变慢。不需要排序聚合的字段应 `doc_values: false`（可省 30%+ 空间）。

**追问**：*"fielddata 和 doc values 什么关系？"* —— 同一目标（列式访问）的两种实现：doc values 是**索引时落盘**的正规方案；fielddata 是**查询时从倒排临时构建**并全量驻留堆，是 ES 1.x 时代的遗留，现在只对 text 可用且被强烈不建议。

---

### B6. 数值和地理字段为什么用 BKD 树，而不是倒排？

**考核程度**：★★☆ ｜ **高频团队**：滴滴、高德、美团（geo 场景必考）

**答案**：

1. **倒排不擅长范围**：`price between 100 and 200` 若用倒排，需要枚举区间内**每一个具体数值**对应的 posting（数值域可达 2^64，不可能枚举）。
2. **BKD 树 = 块存储的 k-d 树**：递归用中位数按维度切分，叶子存最多 1024 个点的紧凑块；范围查询靠"子树边界与查询区间不相交就整块剪枝"。
3. **1 维退化为有序数组 + 二分**，所以单数值范围极快；多维（lat/lon）时逐层换维切分，能快速排除大量子树。
4. **geo 能力建立在它之上**：`geo_bounding_box`（矩形与 BKD 边界求交）、`geo_distance`（外接矩形先粗筛，再算 haversine 精筛）、`geo_shape`（前缀树 + 空间关系判定，更复杂）。
5. **历史**：Lucene 6.0 之前是 Trie（按位切分、每精度层一个 term，term 数爆炸），6.0 起改用 Points/BKD。

**追问**：*"10 亿个司机位置，怎么查附近 3 公里的司机？"* —— ① `geo_point` 字段走 BKD；② 查询用 `geo_distance` + `filter`（不打分）；③ 高频更新位置是灾难（每次都重写整条文档），正解是把"位置"拆成独立小索引/短 TTL 索引，或用 Redis GEO 做实时位置、ES 做检索与回放；④ 分片按地理网格 routing 可显著减少广播。

---

### B7. 为什么查询要分两阶段？只用一阶段行不行？

**考核程度**：★★☆ ｜ **高频团队**：字节、阿里、腾讯

**答案**：

1. **query 阶段**：广播到各分片，各自本地查询 → 打分 → 局部排序 → 只返回 `[docId, sortValue]` 的 top-K 与 totalHits。
2. **fetch 阶段**：协调节点全局归并出 top-N 的 docId，再向对应分片做 multi-get 取 `_source`（含高亮、脚本字段、nested 展开）。
3. **为什么不能一阶段**：`_source` 可能几十 KB，若 query 阶段就回传完整文档，`分片数 × 候选数 × 文档大小` 的网络传输会爆炸；而最终只需要 N 条。**先传"指针"（4 字节 docId）再传"内容"，是最基本的分布式省流设计。**
4. **代价**：多一轮 RTT；`_source` 在 fetch 阶段可能已经不在 page cache（需要回盘读）。
5. **优化开关**：`_source: false` / `_source_includes` 只取需要字段；`size: 0` 做聚合时直接跳过 fetch。

**追问**：*"fetch 阶段会不会取到已经被删除的文档？"* —— 会存在这种窗口（query 与 fetch 之间发生删除/merge），实现上会校验并丢弃失效 docId，结果是"本次返回条数可能少于 size"，这是"近实时搜索"的固有权衡。

---

### B8. filter 和 query 有什么区别？query cache 什么时候生效？

**考核程度**：★★☆ ｜ **高频团队**：美团、京东、阿里

**答案**：

| | query context | filter context |
|---|---|---|
| 打分 | 是 | 否 |
| 缓存 | 否 | 是（node 级 query cache） |
| 用途 | `match`/`multi_match` | `term`/`range`/`exists`/`bool.filter` |

**query cache 生效条件（面试常挖细节）**：

- 只缓存 **filter context** 的查询；
- 作用对象是**单个 segment 的 docId 位图**，不是整个请求；
- 只对"足够大的段"缓存（小段缓存不划算）；
- 段被 merge 后缓存自动失效；
- 容量 `indices.queries.cache.size`（默认堆 10%），LRU。

**request cache** 是另一层：缓存**整个 search 响应**（默认堆 1%），key 是规范化请求体，对 `size:0` 的稳定聚合有效，对含 `now`/随机排序的请求几乎不命中。

**实践结论**：把纯过滤条件全部放进 `bool.filter` 是**收益最高的一条优化**（省 CPU + 提升缓存命中 + 稳定 P99），胜过大多数参数调优。

---

### B9. 聚合是怎么在分布式上执行的？`terms` 聚合为什么会"不准"？

**考核程度**：★★☆ ｜ **高频团队**：美团、京东、字节（订单/商品分析必考）

**答案**：

1. **map-reduce 模式**：各分片本地聚合（利用 doc values + ordinals 顺序扫描）→ 回传部分结果 → 协调节点按 key 合并（文档数相加、子聚合合并/再计算）→ 全局排序取 top N。
2. **"丢失的桶"问题**：某品牌在分片 A 排第 11、分片 B 排第 12，全局可能进前 10，但本地各只取 10 就漏了。解法是 `shard_size`（默认 `size*1.5+10`），让分片多回传候选；`shard_size` 越大越准越慢。
3. **近似聚合**：`cardinality` 默认用 **HLL**（误差 ~1-2%），`precision_threshold` 提精度但吃内存；要精确去重得用 `terms` + `composite` 分页自己算，代价高。
4. **高基数是灾难**：按 `userId` 聚合会生成百万桶，直接触发 `circuit_breaking_exception` 或 `search.max_buckets`（默认 65535）。解法：`filter` 缩范围、`composite` 分页、`sampler` 抽样、或把聚合搬到 ClickHouse/OLAP。
5. **global ordinals 取舍**：跨段统一编号（快、吃堆）vs 协调节点字符串 merge（省堆、慢）。

**追问**：*"为什么 `terms` 聚合不能像 SQL 的 GROUP BY 那样精确？"* —— SQL 在单机上有全量数据，能精确分组；ES 是"分片本地 top-N + 协调节点合并"，为了省网络只回传候选，本质是**用近似换性能**。要精确就得让分片回传全量桶（`shard_size` 极大），网络与内存不可接受。

---

### B10. 路由是怎么算的？为什么主分片数不能改？

**考核程度**：★★☆ ｜ **高频团队**：全部

**答案**：

```java
shard = Math.floorMod(Murmur3.hash(_routing 或 _id), number_of_primary_shards)
```

- **为什么不能改**：路由公式里 `N` 是主分片数，改了 N 则 `hash % N` 全变，**所有历史文档都"找错家"**（查不到、写重复）。所以 ES 只提供：`_reindex` 重建 / `_shrink`（减半，需副本 0 + 只读）/ `_split`（倍增）/ rollover 新建索引 + alias 切换。
- **自定义 routing 的价值**：多租户用 `routing=tenant_id` 让同租户落一片，查询只打一片（省广播、提缓存命中）。**风险是热点**：大租户打爆单分片 → 大厂做法是 `tenant_id#分桶号` 把大租户打散。
- **`_id` 自带索引**：`.id` 字段会建索引供按 id 查；纯 append 场景若不按 id 查，可考虑不自带（或改用业务字段作 `_id` 并避免额外查询需求），能省 10~30% 写入开销。

**追问**：*"routing 用错会怎样？"* —— ① 写入与查询必须用**同一个 routing 值**，否则查不到（这是最常见的线上 bug）；② `_reindex` 换 routing 会导致所有文档重新分布，等同重建。

---

### B11. mapping 建好后能改字段类型吗？怎么做到零停机重建索引？

**考核程度**：★★☆ ｜ **高频团队**：京东、拼多多、美团（商品搜索必问）

**答案**：

1. **不能改**。字段类型决定了磁盘上的索引结构（倒排 vs BKD vs doc values），改类型等于重建。ES 只允许"新增字段"和部分安全变更（如 `ignore_above`）。
2. **零停机重建（alias 切换）标准流程**：

```
① 新建索引 products_v2（新 mapping）
② 双写：应用同时写 v1 和 v2（或先写 v1 + 用 CDC 同步到 v2）
③ 用 _reindex（可加 ?wait_for_completion=false 走后台任务 + slices 并行）回灌历史数据
④ 校验：doc 数、抽样对比、关键查询结果 diff
⑤ 原子切换：POST /_aliases {"actions":[{"remove":{"index":"products_v1","alias":"products"}},
                                        {"add":{"index":"products_v2","alias":"products"}}]}
   —— 别名切换是原子的，客户端无感
⑥ 观察稳定后停双写，删除 v1（或留作回滚）
```

3. **配套技巧**：用 `index templates` 固定 mapping 防止动态推断；用 `dynamic: strict` 让未知字段写入直接失败（尽早暴露问题）；`_reindex` 用 `slices=auto` + `size` 调大批次提速；重建期间用 `?refresh=false`。

**追问**：*"reindex 期间新写入的数据怎么办？"* —— 三种组合：双写（最稳）、CDC 追平（binlog 位点对齐）、或"先 reindex 再短暂停写补差"。只做一次 reindex 必然会漏掉增量，这是面试官最爱抓的点。

---

### B12. ES 的并发控制是怎么做的？`_seq_no` 和 `_primary_term` 为什么都要？

**考核程度**：★★☆ ｜ **高频团队**：阿里、蚂蚁（订单/库存场景）

**答案**：

1. **乐观锁，无悲观锁**。旧机制是 `_version`（文档级），7.x 起主推 `_seq_no`（**分片级**单调序号）+ `_primary_term`（主分片任期）。
2. **更新时带 `if_seq_no` + `if_primary_term`**，不一致返回 409 `version_conflict_engine_exception`。
3. **为什么必须带 term**：主分片 failover 后，新主分片的 `_seq_no` 可能与旧主"数值相同但含义不同"（旧主写到 5，新主也从 5 开始）。term 相当于 Raft 的任期，用来区分"哪个任期里的 5"，避免对着过期视图写入。
4. **原子性边界**：单文档写原子；bulk 不原子；**无跨文档/跨分片事务**。所以"库存扣减"这类强一致操作必须在数据库做，ES 只做检索副本。
5. **`retry_on_conflict`**：只对 `_update` 生效（读-改-写模式），`index` 覆盖写不重试。

---

### B13. 段合并策略是什么？`force merge` 为什么危险？

**考核程度**：★★☆ ｜ **高频团队**：字节、腾讯、华为

**答案**：

1. **TieredMergePolicy（分层）**：把段按大小分层，每层最多 `segments_per_tier`（默认 10）个；超过就挑"大小相近"的若干个合并；单段超过 `max_merged_segment`（默认 5GB）不再参与常规合并；`expunge_deletes_allowed`（默认 10%）决定"删除率高时优先回收"。
2. **收益**：段数↓ → 每次查询要遍历的段↓；回收 `.liv` 死文档；大段压缩率更好（docId 更连续、词典更大、FST 共享更多）。
3. **代价**：merge 是"读旧段 + 写新段"，**1 倍写放大**；期间旧段仍服务查询（引用计数），**峰值磁盘可达 2 倍**；大 merge 吃满 IO 会让查询抖动（"夜间写入、白天查询变慢"的常见根因）。
4. **`_forcemerge` 危险在哪**：① **不节流**，能吃满 IO 让集群失去响应；② 对被写入的活跃索引做 force merge 是反模式——合成巨大段后新写入只能进极小段，段结构再次劣化，且后续合并更痛苦；③ 只应对**只读索引**（按天滚动的历史索引）做 `max_num_segments=1`，且最好先迁到空闲节点。
5. **轻量替代**：`_forcemerge?only_expunge_deletes=true` 只清理死文档，比全量 force merge 安全得多。

---
## 第三档 ★★★ 压轴深水区（决定 P6/P7 定级）

### C1. 三个 master 节点，网络分成 2:1，会发生脑裂吗？Zen2 的选举到底怎么做的？

**考核程度**：★★★ ｜ **高频团队**：腾讯、阿里、华为（架构向必问）

**答题框架**：

1. **结论**：不会脑裂（不会出现两个 master 同时接受元数据变更），但**少数派那一侧会失去 master**，其上的分片变为不可用/需重新分配。
2. **机制**：
   - 每个 master-eligible 节点持续广播自己的 **cluster state version**；
   - 节点向其它 master-eligible 节点发起"ping + 版本交换"，收集足够多的响应后**判断当前是否有活跃 master**；
   - 若判断无主，进入选举：候选节点必须获得**法定票数 `(N/2)+1`**（N = master-eligible 节点数，且以"投票配置"为准）；
   - 2:1 分区时，多数派侧（2 个）能凑齐 2 票选出 master；少数派侧（1 个）永远凑不齐 → **无法成为 master，也无法形成可写集群**。
3. **关键洞察**：Zen2 是用"**法定票数 + 版本单调**"从机制上消除脑裂，而不是靠"人工配置 minimum_master_nodes"（那是 6.x 的做法，7.x 已移除）。
4. **与 Raft 的区别（区分高低分）**：选主用投票，但**集群状态复制不是 Raft 日志复制**——master 计算新状态后**publish 广播给所有节点，节点 ack 版本**，是"推送 + 确认"模型。这解释了为什么节点多时"改 mapping"会有秒级延迟。

**追问链**：
- *"两个 master 的集群行不行？"* → 不行。2 节点的法定票是 2，挂一个就选不出 master，可用性比 1 节点更差。**必须奇数，且≥3。**
- *"下线一个 master 要注意什么？"* → 先用 `_cluster/voting_config_exclusions` 把它排除出投票配置，再停机；否则它会持续参与投票导致"选不出主"。
- *"`cluster.initial_master_nodes` 什么时候该删？"* → 只在**首次引导**用；集群稳定后必须从配置移除，否则整集群重启时可能触发多节点同时"自我引导"的经典事故。

---

### C2. 主节点会不会成为瓶颈？什么操作会把 master 打挂？

**考核程度**：★★★ ｜ **高频团队**：字节、阿里、腾讯

**答案**：

1. **正常情况下不是瓶颈**：master 只处理元数据（集群状态、分片分配、模板/别名/ILM）与分配决策，**不碰文档读写**。所以 master 挂了搜索不受影响，只是"元数据变更暂停"。
2. **什么时候会成为瓶颈**：
   - **集群状态体积大**：分片数上万、索引数千、mapping 巨大 → 每次 publish 都要序列化几 MB 并广播到所有节点，master CPU/网络吃紧，全网 ack 延迟上升。
   - **元数据变更过于频繁**：每秒建索引、动态 mapping 不断新增字段、频繁 rollover、频繁改副本数 → 每次变更都是一次"全集群 publish"。
   - **master 与 data 混部**：data 节点的 GC/merge 停顿会让 master 心跳超时，触发"master 切换 → 状态重发布 → 又是负载"，形成恶性循环。
   - **分片分配风暴**：大量节点同时上下线，master 反复计算再平衡。
3. **治理手段**：**3 个专用 master**（不存数据、只做元数据）；固定 mapping + index template（禁止动态推断）；用 data stream + ILM 代替手工建索引；控制分片总数（每 GB 堆 ≤20~30 分片）；监控 `cluster_state` 的 publish 耗时（`_nodes/stats` 的 `indices.cluster_state` 指标）。

**追问**：*"为什么动态 mapping 是隐患？"* → 未知字段写入会触发**集群状态变更**（新增字段映射），且多节点并发推断可能产生类型冲突（一个节点推成 `long`，另一个推成 `text`）。生产应 `dynamic: strict` + 显式 mapping。

---

### C3. 线上 ES 堆内存持续高位、频繁 Old GC，你怎么定位和治理？

**考核程度**：★★★ ｜ **高频团队**：阿里、美团、腾讯（"你踩过什么坑"必问）

**排查路径（分四层，层层收窄）**：

```
第 1 层：确认是"堆问题"还是"页面缓存问题"
  _nodes/stats/jvm → heap_used_percent、old_gc_count、old_gc_time
  _nodes/hot_threads → 看 CPU 热点是 merge / search / indexing 哪一类

第 2 层：确认堆被谁占了
  _nodes/stats/indices/segments → 段数、内存（terms 的 FST 占用）
  _nodes/stats/indices/fielddata → 是否有 fielddata 泄漏
  _nodes/stats/breaker → fielddata / request / parent 的 tripped 次数
  _cat/nodes?v&h=heap.*,segments.*,fielddata.memory_size

第 3 层：定位到具体索引/字段
  分片数是否过多（每分片都有 buffer + FST）
  是否有高基数字段被建成 text（term 数爆炸 → FST 巨大）
  是否有高基数 terms 聚合 / 大 scroll / 深分页（request breaker）

第 4 层：GC 日志与堆 dump
  jstat / gc log 看 Old 区是否"回收后仍不降"（真实泄漏）
  heap dump 找占比最大的对象（多为 FST、聚合桶、缓存）
```

**常见根因与处置（按出现频率排序）**：

| 根因 | 现象 | 处置 |
|---|---|---|
| 分片过多 | 堆随数据量线性上涨，GC 后不降 | 合并索引、减少分片、rollover 控制总量 |
| 高基数字段建成 `text` | FST/词典巨大 | 改 `keyword` 或 `index: false`，重建索引 |
| 聚合桶爆炸 | 请求时堆瞬时飙高 + 429 | `filter` 缩范围、`composite` 分页、`search.max_buckets` |
| fielddata 误用 | 堆缓慢上涨 | 关 fielddata，改 keyword 双字段 |
| 深分页/大 scroll | 并发高时堆不稳 | PIT + `search_after`；限制单次 scroll 大小 |
| 缓存配置过大 | 堆吃满 | 调 `indices.queries.cache.size` / `requests.cache.size` |
| 真的内存泄漏 | 回收后仍增长 | 升级版本（历史上有过 known leak）、dump 分析 |

**一句话总结**：ES 的堆问题 80% 不是"堆给少了"，而是"分片太多、字段建错、聚合太大"。**加堆只是把 OOM 推迟，且越过 31GB 还会因指针膨胀反而更差。**

---

### C4. 设计一个支持"无限下拉"的搜索分页方案，说清边界条件。

**考核程度**：★★★ ｜ **高频团队**：美团、京东、字节（业务设计题）

**答案要点**：

1. **方案选型**：`PIT + search_after`。
   - `POST /index/_pit?keep_alive=1m` 拿 `pit_id`；
   - 每页请求带 `pit.id` + `sort`（业务排序键 + `_shard_doc`）+ `search_after`（上一页 `sort` 数组原样回传）+ `size`；
   - 会话结束 `DELETE /_pit`（或靠 `keep_alive` 自动过期）。
2. **边界条件（考官真正想听的）**：
   - **排序键必须全局唯一且稳定**：`sort: [{price: desc}, {_id: asc}]`；否则同价商品会重复/漏出。
   - **不能跳页**：`search_after` 只有"下一页"，若产品要求页码跳转，只能"限制最大页数"或改用"按业务分区（如按品类/时间桶）再分页"。
   - **深翻的性能上限**：单页 O(shards × size)，与页深**无关**（这是它相对 `from+size` 的核心优势），但每页仍要打全部分片 → 分片多时用 `preference` 固定副本提升缓存命中。
   - **数据一致性**：无 PIT 时翻页过程中删除/merge 会导致游标指向的文档消失；有 PIT 则"视图固定"，但**看不到新写入**（可接受，需在 UI 上说明"结果基于开始搜索时"）。
   - **PIT 资源**：`keep_alive` 太长会占用段（阻止 merge），必须设上限并在客户端"取消/离开页面"时主动释放。
3. **对比 `scroll`**：`scroll` 适合"一次性全量导出"，不适合"用户翻页"（快照占资源久、无法中途改查询、结果陈旧）。

---

### C5. 大促前如何做写入容量评估与压测？（工程方案题）

**考核程度**：★★★ ｜ **高频团队**：京东、拼多多、阿里（大促团队常问）

**答案结构（评估 → 压测 → 调优 → 兜底）**：

**① 评估**

```
目标写入 TPS × 平均文档大小 = 带宽需求
单分片可持续写入 ≈ f(磁盘 fsync 能力, bulk 大小, 段生成速率)
主分片数 ≥ 目标 TPS / 单分片能力
分片总数 ≤ 节点数 × 每 GB 堆 20~30 × 堆GB
磁盘 = 原始量 × (1+副本) × 索引放大 × merge 峰值 × 1/0.85
```

**② 压测（关键：压"写入 + 后台 merge + 并发查询"三者叠加）**

- 用 `esrally` 或自研 bulk 压测，**必须同时跑查询流量**，否则测不出"merge 与查询争 IO"的真实瓶颈；
- 观察指标：`indexing` 线程池队列、`rejected` 计数、`translog` 大小增速、`segments` 数量、`merge` 当前数、`refresh` 耗时、`fsync` 延迟（`_nodes/stats/fs`）；
- 找**拐点**：并发从 N 翻倍时，P99 是否跳变、是否开始 429。

**③ 调优（导入期与在线期两套配置）**

| 阶段 | 配置 |
|---|---|
| 批量导入/大促预热 | `refresh_interval: -1`（或 30s）、`replicas: 0`、bulk 5~15MB、`translog.durability: async`（可容忍时） |
| 在线服务 | 恢复 `refresh_interval: 1s`、`replicas: 1+`、`translog.durability: request` |
| 老索引 | `_forcemerge` 到 1 段（只读索引）、降副本、迁冷节点 |

**④ 兜底**

- 客户端**指数退避 + 有界队列 + 降级写入**（写不进就先落 Kafka/MQ，回放补写）；
- **读写分离到不同副本**、核心索引与日志索引**物理隔离**；
- 大促前**冻结 mapping**（禁止动态字段）、**固定分片**（禁止 reindex/rollover 冲突）；
- 准备好"限流开关"（`search.max_concurrent_searches`、`action.destructive_requires_name`）。

**追问**：*"为什么压测要跑 merge？"* → 因为 merge 是写路径的**隐藏成本**：写入越快，段越多，merge 越凶，IO 越紧张。只压"纯写入"会得到一个虚假的高 TPS，上生产后一开查询就崩。这是"压测做没做过真的"的照妖镜。

---

### C6. 数据从 100GB 涨到 10TB，索引怎么演进？分片怎么扩？

**考核程度**：★★★ ｜ **高频团队**：字节、腾讯、贝壳

**答案**：

1. **先判断是"同一索引变大"还是"时间序列累积"**：
   - **时间序列（日志/订单/埋点）** → 用 **data stream + ILM rollover**，按天/周建新索引，永不"一个索引存一年"；冷数据降副本、force merge、迁 warm/cold；更冷的上**可搜索快照**（数据在对象存储，本地只留元数据）。
   - **单一大索引（商品/内容）** → 需要"按业务维度拆分"（品类/租户/区域 routing）或**按数据量重分片**。
2. **扩分片的三种手段**：

| 手段 | 机制 | 限制 |
|---|---|---|
| `_split` | 1 分片 → N 分片（复制数据再切） | 要求**只读 + 副本 0**，磁盘要 2 倍空间，耗时与数据量成正比 |
| `_shrink` | N 分片 → 1/2…（合并） | 同上，用于"分片过多" |
| `_reindex` + 新索引 | 重建，可换 mapping/分片数/routing | 最灵活，但要处理增量（双写/CDC） |
| rollover + alias | 新建索引接管新数据 | 老数据仍在旧索引，需按时间窗口淘汰 |

3. **推荐的"无痛"路径**：新老并存 → `alias` 指向"当前写索引" → 双写/CDC 回灌 → 校验 → 原子切别名 → 老索引只读、逐步下线。
4. **量化红线**：单分片 20~50GB；每 GB 堆 ≤20~30 分片；`cluster.max_shards_per_node` 默认 1000 —— 10TB / 40GB ≈ 250 个主分片，加 1 副本 = 500 个分片，**至少需要 5~6 个数据节点（按每节点 100 分片以内算更稳）**。

---

### C7. 一个节点宕机后恢复，集群发生了什么？为什么恢复很久？

**考核程度**：★★★ ｜ **高频团队**：腾讯、华为、网易

**答案**：

1. **故障检测**：节点间心跳（`cluster.fault_detection` 的 ping 间隔/超时，默认秒级）失败 → master 判定节点失联（不是立刻，要给足网络抖动余量）。
2. **分片重分配**：master 把该节点上的**主分片**从其它节点的 in-sync 副本中**提升为 primary**（promotion）；副本缺失的分片标记 unassigned，随后在其它节点上重建副本。
3. **副本重建（peer recovery）**：
   - **全量（file-based）**：直接从主分片拷 segment 文件（快，但吃网络/磁盘 IO）；
   - **增量（translog-based）**：若两边 commit point 有共同历史，只重放缺失 translog 区间。
   - 限流参数 `indices.recovery.max_bytes_per_sec`（默认几十 MB/s 量级），**这是"恢复很久"的直接原因**——限流是为了不把查询打死。
4. **旧节点回来**：发现自己已不是 primary，**丢弃本地数据**，作为副本重新 recovery（不能"直接接着用"，否则数据分叉）。
5. **恢复慢的常见原因**：① 限流配置保守；② 同时恢复的分片太多（并发打满网络）；③ 分片太大（单分片 200GB 的重拷）；④ 磁盘慢/共享网络；⑤ 副本数过多（恢复量 = 数据量 × 副本数）。

**追问**：*"为什么建议分片别太大？"* → 恢复时间与分片大小成正比；单分片 200GB 意味着"一次故障要拷 200GB"，期间该分片一直 red/yellow，恢复窗口长、影响面大。**"分片大小"其实是"故障恢复时间"的直接决定因素**，很多人只记得"20~50GB"却不知道原因。

---

### C8. 同一个文档在不同分片布局下打分不同，正常吗？怎么保证排序一致？

**考核程度**：★★★ ｜ **高频团队**：阿里、百度、快手

**答案**：

1. **正常，且是设计使然**。BM25 依赖 `N`（总文档数）、`n`（含该词的文档数）、`avgdl`（平均字段长度），这些是**分片局部统计量**。默认 query-then-fetch 下每个分片用自己的局部统计打分，因此：
   - 同一文档在不同分片布局下分数不同；
   - 不同分片的分数**不完全可比**（这是"分布式检索打分近似"的固有问题）。
2. **解法**：
   - `search_type=dfs_query_then_fetch`：先做 `_search_shards` 预查询收集全局 df/avgdl，再执行正式查询 → 打分精确、跨分片可比，代价是多一轮 RTT。
   - **更实用的解法**：让打分不参与最终排序——用 `sort`（业务字段）+ `function_score` 的确定性权重，或 `track_scores: false`；相关性只作为一路特征在应用层融合。
   - 单分片/小索引场景偏差更明显，可考虑"减少分片"或干脆用 DFS。
3. **延伸**：这也是"ES 排序结果不可复现"的一类根因——**排序键不唯一**（如只按 `_score` 或只按时间）时，同分文档顺序由内部 docId 决定，跨查询/跨分片会变。要稳定必须加 tiebreaker。

---

### C9. 设计"全文检索 + 向量语义召回"的混合检索方案。

**考核程度**：★★★ ｜ **高频团队**：快手、小红书、B站、字节（2024 年后高频）

**答案结构**：

1. **为什么混合**：全文（BM25）擅长精确词/品牌/型号；向量（kNN）擅长语义/同义/口语化。单用任一方都有明显短板，工业界主流是**双路召回 + 融合排序**。
2. **两条实现路径**：

| 路径 | 做法 | 优缺点 |
|---|---|---|
| **ES 单集群内混合** | 同一索引存 `text` + `dense_vector`；查询同时发 `match` 与 `knn`，用 `rank`/`rrf`（倒数排名融合）或 `function_score` 合并 | 一站式、过滤条件可共享、运维简单；向量规模受限（HNSW 吃内存，千万级以内更合适） |
| **召回分离** | 向量库（Milvus/Qdrant）出候选 id → 回查 ES 取详情与过滤；或 ES 出文本候选 → 向量库重排 | 各自可扩到十亿级；但**两套系统的一致性、延迟、双写**是主要成本 |

3. **关键工程细节（拉开差距的部分）**：
   - `num_candidates`：每分片探索的候选数，太小召回差、太大变慢；分片越多越要调大。
   - **过滤 + 向量**：先过滤再 kNN 时，若过滤后候选极少，HNSW 会退化为近似暴力扫描，延迟飙升；可考虑"分区建索引（按品类分索引）"。
   - **向量维度与内存**：768 维 float32，1000 万文档 ≈ 30GB 原始向量 + HNSW 图开销，必须算清内存再选规模。
   - **召回融合**：RRF（`1/(k+rank)` 相加）比"分数直接加权"更鲁棒，因为两路分数量纲不同。
   - **embedding 更新**：模型升级意味着**全量重算向量**，必须设计"双向量字段 + 灰度切换"。

---

### C10. MySQL 是主库，ES 做检索，怎么保证数据一致？

**考核程度**：★★★ ｜ **高频团队**：阿里、蚂蚁、美团、京东（业务架构必问）

**答案（这是"ES 不能当主库"的落地版）**：

1. **定位先说清**：ES **不是事务型存储**，无跨文档事务、无强一致、近实时（1s 可见）。所以正确答案永远是"**以 MySQL 为唯一真相源，ES 是最终一致的检索副本**"。
2. **三种同步方案对比**：

| 方案 | 一致性 | 侵入性 | 风险 |
|---|---|---|---|
| 双写（应用同时写 DB 和 ES） | 弱（两次写有一致性窗口） | 高（业务代码耦合） | 一边失败就不一致；重试/顺序难保证 |
| **异步消息（binlog → MQ → 消费写 ES）** | 最终一致 | 低 | 消息重复/乱序/丢失需处理 |
| **Canal/Flink CDC 直读 binlog** | 最终一致 + 有序（按主键分区） | 最低 | 运维复杂、schema 变更需适配 |

3. **必须解决的四个问题**：
   - **幂等**：`_id` 用业务主键（`order_id`），重复消费等价于覆盖写。
   - **乱序**：同一主键的 binlog 必须按顺序消费（Kafka 按 key 分区）；或用**版本号/更新时间**字段做"新版本才覆盖"（`_external` 版本或 `if_seq_no` 乐观锁）。
   - **删除**：binlog 只有 delete 事件没有内容，需"标记删除"（`is_deleted` 字段）或先查 DB 再删（有竞态）。
   - **对账**：定期跑"DB 与 ES 的 count/抽样 diff/checksum"，用 `_update_by_query` 或重灌修复 —— 这是"一致性"的最后一道防线，也是面试里最能体现工程经验的答案。
4. **读写路径建议**：写走 DB，读走 ES；"写完立刻查"的场景（如下单后看订单）走 DB 或 ES 的 **realtime GET**（按 `_id` 直查，走 translog，无需 refresh）。

---

### C11. 一个查询从 20ms 变成 2s，你的完整排查路径是什么？

**考核程度**：★★★ ｜ **高频团队**：全部（P7 常以"现场故障"形式出）

**分层排查（从"是不是 ES 的锅"开始）**：

```
① 确认范围：单查询慢 / 某类查询慢 / 全集群慢
   → 全集群慢：先看 _cluster/health、节点负载、GC、磁盘 IO（可能不是查询的问题）
   → 单查询慢：抓具体 DSL

② 客户端/网络层
   连接池耗尽？协调节点 CPU 打满？响应体过大（_source 太大）？超时重试放大流量？

③ ES 指标层
   _nodes/stats：search 线程池队列、rejected、query_cache 命中率、segments 数、merges 当前
   _cat/nodes?v&h=cpu,heap.*,search.*,segments.count
   慢日志：index.search.slowlog.threshold.query.warn（如 10s/warn、5s/info）

④ 查询本身
   profile: true → 看是 rewrite 慢、next_doc 慢、advance 慢、doc_values 慢、collector 慢
   - rewrite 慢 → 前导通配/正则/大量 term 展开
   - next_doc 慢 → 过滤选择性差（把 filter 放前面、加 term 约束）
   - advance 慢 → 长列表做驱动（应最短列表驱动）
   - doc_values 慢 → 高基数聚合/排序、global ordinals 反复构建
   - fetch 慢 → _source 太大、高亮、脚本字段

⑤ 数据与结构层
   段数是否上百（merge 跟不上）？分片数是否过多（广播成本）？
   字段是否建错（text 做聚合）？是否 deep paging / 大 scroll / 大聚合？

⑥ 系统层
   page cache 命中率、磁盘 util 100%（merge 打满）、NUMA/CPU steal、容器内存限制
```

**典型结论映射**（能背出这张表，说明真处理过）：

| 症状 | 结论 |
|---|---|
| 只有翻到第 50 页才慢 | 深分页 → `search_after` |
| 只有带 `*关键词` 的查询慢 | 前导通配 → ngram 或改查询 |
| 每天凌晨查询抖动 | merge 打满 IO → 调 merge 策略/错峰/换盘 |
| 并发一高就 429 | search 队列满 → 降并发/加副本/调 `max_concurrent_searches` |
| 某些用户特别慢 | 大租户数据集中 → 检查 routing 是否造成热点分片 |
| 慢但 CPU 不高 | page cache 不足/回盘读 → 堆别给太大、加内存、force merge 冷索引 |

---

### C12. ES 为什么不能替代数据库？和 ClickHouse 怎么选？

**考核程度**：★★★ ｜ **高频团队**：字节、阿里、蚂蚁（选型题，考"边界感"）

**答案**：

1. **ES 的天生短板**：
   - **无事务**（无跨文档/跨分片 ACID），bulk 不原子；
   - **近实时而非实时**（1s 可见）；
   - **写放大严重**（副本 + 多结构 + merge 重写，6~20 倍），不适合高频更新；
   - **JOIN 极弱**（nested/join 都是"文档内模拟"，代价高）；
   - **深分页与高基数聚合有硬限制**；
   - **存储成本高**（列存能力仅 doc values，压缩不如专用列存）。
2. **与 ClickHouse 的边界**：

| 维度 | ES | ClickHouse |
|---|---|---|
| 全文相关性 | 强（BM25/分词/FST） | 弱（倒排/`tokenbf_v1` 有限） |
| 大宽表聚合 | 中（受桶数与 JVM 堆约束） | **极强**（向量化执行、列存压缩） |
| 点查/更新 | 中（写放大） | 弱（`ReplacingMergeTree` 异步去重、更新代价高） |
| 实时性 | 近实时 1s | 微批（可做到秒级） |
| 运维复杂度 | 中（JVM + 分片副本） | 中（无 JVM，但副本/ZooKeeper 体系） |
| 典型场景 | 搜索、日志检索、可观测 | 指标分析、漏斗、BI、大宽表 |

3. **一句话结论**：**"要搜（相关性、模糊、多维过滤混合）选 ES；要算（大规模聚合、明细扫描）选 ClickHouse；两者常并存——ES 做检索入口，ClickHouse 做分析下钻。"**

---

# 附录

## 附录 A：一页速查表（面试前 5 分钟看这个）

```
【写】
index/bulk → 协调节点路由 floorMod(murmur3(routing), N)
 → 主分片：translog(fsync) + buffer
 → 副本同步 → 全 ack → 返回
 → refresh(1s)：buffer→segment，可搜索（未 fsync）
 → flush(512MB/30min)：commit+fsync+清translog
 → merge：小段合大段，回收 .liv 死文档

【读】
query 阶段：各分片本地 topK（只回 docId+sortValue）
fetch 阶段：协调节点归并 topN → multi-get 取 _source
DFS：多一轮预查询拿全局 df/avgdl（打分精确但慢）

【结构】
term index(FST, 堆) → term dict(.tim) → posting(.doc/.pos)
数值/geo → BKD；排序聚合 → doc values(列存+ordinal)
原文 → _source(.fdt)；打分长度 → norms；删除 → .liv

【分布式】
主分片数不可改；副本只读、升主用；Zen2 法定票数防脑裂
cluster state = 元数据+路由表，master 只广播不搬数据

【红线】
from+size ≤ 10000；堆 ≤50% 且 ≤31GB；每GB堆 ≤20~30 分片
磁盘 85/90/95% 水位；单分片 20~50GB；force merge 只对只读索引

【一句话优化】
过滤进 bool.filter（不打分+可缓存）＞ 批量写 ＞ 少字段少结构 ＞ 深分页换 search_after
```

## 附录 B：最容易答错的 12 个点

1. ❌ "ES 实时搜索" → ✅ **近实时**，默认 1s 可见；但按 `_id` 的 GET 是实时的（走 translog）。
2. ❌ "refresh 就是刷盘" → ✅ refresh 只写 Filesystem cache **不 fsync**；fsync 是 flush 干的。
3. ❌ "调大 refresh_interval 会丢数据" → ✅ 只降可见性，**不影响持久性**（translog 仍每条 fsync）。
4. ❌ "副本也能写" → ✅ 写入只打主分片，副本只读。
5. ❌ "删除后磁盘就降" → ✅ `.liv` 只是标记，要 merge 才回收。
6. ❌ "数值索引用倒排" → ✅ **BKD 树**（6.0 前是 Trie）。
7. ❌ "text 能做聚合" → ✅ 无 doc values，必须 keyword 双字段。
8. ❌ "主分片数可以改" → ✅ 不可改，只能 reindex/shrink/split/rollover。
9. ❌ "bulk 是原子的" → ✅ 每条独立，部分失败必须检查 `items`。
10. ❌ "堆越大越快" → ✅ 超过 50%/31GB 反而更差（page cache 少 + 指针膨胀）。
11. ❌ "深分页调大 max_result_window 就行" → ✅ 那只是把 OOM 边界推后，正解是 search_after/PIT。
12. ❌ "ES 能做事务" → ✅ 单文档原子，无跨文档/跨分片事务，必须由 DB 做真相源。

## 附录 C：版本演进关键节点（答题时提一句显得"跟得上"）

| 版本 | 关键变化 | 面试价值 |
|---|---|---|
| 5.x | 默认打分改 **BM25**；引入 `text`/`keyword` 拆分 | 解释"为什么现在默认 BM25" |
| 6.x | 移除 `_all`；单 index 单 type 限制；**BKD 成熟**；`index.sort` | 解释"多 type 为什么被废弃" |
| 7.x | **彻底移除 type**；Zen2 选主取代 `minimum_master_nodes`；AR​S 默认开启；`_seq_no` 体系；data stream / ILM / 可搜索快照（7.10+） | 高频：type 移除原因、防脑裂机制 |
| 8.x | **默认开启安全（TLS/认证）**；向量 kNN（HNSW）成熟；节点角色强制显式；frozen tier | 2024 后高频：向量检索、零信任默认 |

## 附录 D：学习路径（从"会用"到"懂底层"）

```
阶段 1（会用）
  装单节点 → 建 index/mapping → bulk 导入 → match/term/bool/range → terms/date_histogram 聚合

阶段 2（会调）
  _cat APIs 看集群 → slowlog + profile 定位 → 分片/副本规划 → ILM + rollover

阶段 3（懂原理）
  读《Elasticsearch: The Definitive Guide》的 "Inside a Shard" 章节
  → 看 Lucene 的 segment/FST/BKD 文档
  → 自己用 Lucene 写 20 行 demo（建索引/查询/看 .tim 文件大小），理解会陡增

阶段 4（懂取舍）
  做一次"亿级数据分片规划 + 零停机重建 + 压测"的完整演练
  → 读 ES 官方 "Tune for indexing speed / search speed" 两篇
  → 复现一次 red 集群 + merge 打满 IO + 堆 OOM，练排查肌肉记忆
```

## 附录 E：免责与使用说明

- 本文的技术细节以 **Elasticsearch 7.x / 8.x + Lucene 9.x** 为主，版本间参数默认值存在差异（如断路器阈值、recovery 限流、线程池队列），落地时请以**你所部署版本的官方文档**为准。
- 面试题按"公开面经 + 岗位技术要求"归类，用于**自测能力档位**，不代表任何公司的正式题库或原题。
- 文中所有"经验值"（如单分片 20~50GB、每 GB 堆 20~30 分片）是行业实践区间，不是硬性规范；真实规划必须结合你的文档大小、查询模式、SLA 与硬件实测。
- 涉及生产变更（force merge、reroute、改副本数、删索引、切别名）前，请先在预发环境验证并保留回滚方案。

---

**（全文完）**
