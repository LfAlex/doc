# Java MQ 底层原理深度讲义 + 国内大厂面试题全解（RocketMQ / Kafka）

> 覆盖范围：MQ 通用理论 → 操作系统底层支撑 → RocketMQ 深度拆解 → Kafka 深度拆解 → 选型对比 → MQ 大厂面试题（标注考核程度 + 详解 + 延伸）。
>
> 阅读方式：由浅入深。每一章先讲"是什么、为什么"，再讲"底层怎么做的"，最后给"延伸追问"。面试题为国内大厂高频真题风格整理，难度标注：★ 基础 / ★★ 进阶 / ★★★ 深入 / ★★★★ 专家级（源码/架构设计）。

---

## 目录

- 第一部分：MQ 通用基础（为什么需要 MQ、消息语义、核心难题）
- 第二部分：操作系统底层支撑（PageCache、mmap、零拷贝、顺序写、刷盘、epoll/Reactor）
- 第三部分：RocketMQ 深度拆解（架构、存储引擎、读写链路、高可用、事务/延迟/顺序消息、5.x 演进）
- 第四部分：Kafka 深度拆解（架构、日志存储、ISR/HW、KRaft、幂等与事务、Rebalance）
- 第五部分：RocketMQ vs Kafka vs RabbitMQ 选型对比
- 第六部分：MQ 大厂面试题精讲（16 题）
- 附录：知识延伸学习地图

---

# 第一部分：MQ 通用基础

## 1.1 为什么需要 MQ：三大核心价值

```
【同步直调】                          【引入 MQ 后】
用户下单 ──> 订单服务                 用户下单 ──> 订单服务 ──> MQ ──┬──> 库存服务
              ├─> 库存服务(50ms)                                  ├──> 积分服务
              ├─> 积分服务(80ms)                                  └──> 通知服务
              ├─> 通知服务(60ms)
              └─> 返回(总耗时≈250ms)    订单服务写完 MQ 即返回(≈60ms)
```

1. **异步化**：把非核心链路从主流程剥离，缩短 RT（响应时间）。
2. **解耦**：生产者不需要知道谁消费、有几个消费者。新增下游只需订阅，不改上游代码——这是"发布订阅"对"直接调用"的本质优势。
3. **削峰填谷**：秒杀瞬时 10 万 QPS，下游数据库只能扛 5000 QPS，MQ 作为缓冲池让下游按自己的节奏消费。**代价是消息延迟上升**，这是一个典型的"用时间换空间"的权衡。

**引入 MQ 的代价（面试必答的反面）**：
- 系统复杂度上升：要处理消息丢失、重复、乱序、积压。
- 一致性变弱：从"同步强一致"退化为"最终一致"，业务要能容忍。
- 多一个故障点：MQ 集群本身要做高可用，否则 MQ 挂了全链路瘫痪。

## 1.2 两种消息模型

| 模型 | 语义 | 典型实现 |
|---|---|---|
| 点对点（Queue） | 一条消息只被一个消费者处理 | 任务分发 |
| 发布订阅（Topic） | 一条消息被每个订阅方各处理一次 | 事件广播 |

Kafka/RocketMQ 用 **Topic + Consumer Group** 统一了两种模型：
- **同一个 Group 内**的多个消费者分摊消息（集群消费 → 点对点语义）；
- **不同 Group 之间**各自独立消费全量消息（广播语义）。

```
Topic: order-created (4个分区/队列)
   P0  P1  P2  P3
   │   │   │   │
   ├───┴───┴───┴──> Group A (库存服务, 2实例)  ← 组内负载均衡
   └───┬───┬───┬──> Group B (积分服务, 4实例)  ← 组间广播
```

## 1.3 消息投递语义（Delivery Semantics）

这是所有 MQ 面试题的理论根基：

| 语义 | 含义 | 实现代价 |
|---|---|---|
| At Most Once | 最多一次，可能丢 | 发出去就不管（acks=0） |
| At Least Once | 至少一次，可能重 | 失败重试 + 消费确认后置（**工程主流**） |
| Exactly Once | 恰好一次 | 分布式事务/幂等，代价极高 |

**关键认知：网络世界里不存在真正的"恰好一次投递"**。因为"处理消息"和"确认消息"是两个动作，无法原子化——确认丢失就必然重发。所谓 Exactly Once 的工程实现只有两条路：

1. **At Least Once + 消费端幂等** = 业务效果上的 Exactly Once（最常用）。
2. **端到端事务**（如 Kafka 事务：消费-处理-生产-提交位移在一个事务内），但仅限 Kafka 内部闭环，出 Kafka 就失效。

## 1.4 MQ 的七大核心难题（贯穿全文的主线）

1. **消息不丢**：生产端 → 存储端 → 消费端，三段都要保证（详见面试题 2）。
2. **消息不重（幂等）**：去重表、唯一键、状态机、Redis setnx（详见面试题 3）。
3. **消息有序**：全局有序 vs 分区有序（详见面试题 4）。
4. **消息积压**：扩容、降级、旁路（详见面试题 12）。
5. **事务消息**：本地事务与发消息的原子性（详见面试题 5）。
6. **延迟消息**：定时投递（详见面试题 6）。
7. **高可用**：副本、选主、脑裂（RocketMQ DLedger / Kafka ISR+KRaft）。

## 1.5 Push vs Pull：一个被名字欺骗的经典问题

- **Push**（Broker 主动推）：实时性好，但 Broker 不知道消费者处理能力，容易把消费者打挂（缺乏背压）。
- **Pull**（消费者主动拉）：消费者自主控速，天然背压；但空轮询浪费资源、实时性差。

**工程最优解：长轮询（Long Polling）**——消费者发起拉请求，Broker 若无消息则 **hold 住请求不返回**（挂起），等新消息到达立刻唤醒返回；超时（如 RocketMQ 15s）则返回空。兼具 Push 的实时性与 Pull 的流控能力。

> RocketMQ 的 `DefaultMQPushConsumer` 名字叫 Push，**底层实际是 Pull + 长轮询**，这是高频面试陷阱。Kafka 是纯 Pull（fetch 请求也有 max.wait.ms 的类似挂起机制）。

---

# 第二部分：操作系统底层支撑（MQ 高性能的物理基础）

> 这一部分是 RocketMQ/Kafka "为什么快"的共同答案，也是深入面试的分水岭。不理解这一章，后面所有存储设计都是空中楼阁。

## 2.1 磁盘顺序写为什么能比内存随机写还快

机械磁盘随机写要经历：寻道（约 10ms）+ 旋转延迟 + 传输。顺序写省掉寻道和旋转，速度可达数百 MB/s。SSD 没有寻道，但随机写会触发 FTL 映射表更新、写放大和 GC，顺序写依然显著更快。

**实测数量级（经典数据）**：磁盘顺序写 ≈ 600MB/s，磁盘随机写 ≈ 100KB/s，内存随机写 ≈ 数 GB/s。但注意：**写"内存"如果指写 PageCache（异步刷盘），本质也是先写内存**——所以"顺序写磁盘 + PageCache"的组合，性能逼近纯内存操作。

这就是 Kafka 和 RocketMQ 都坚持 **Append-Only Log（只追加日志）** 的原因：
- 永远在文件末尾追加，绝不修改中间内容；
- 删除靠"整段文件删除"（segment/文件过期），不做行级删除；
- 这个思想源自数据库的 WAL（Write-Ahead Log），LSM-Tree（RocksDB/HBase）也是同源。

## 2.2 PageCache（页缓存）：MQ 其实是"写内存"的

Linux 写文件默认路径：

```
应用 write() ──拷贝──> PageCache(内核内存, 脏页) ──异步/主动──> 磁盘
应用 read()  <──拷贝── PageCache(命中则不读盘)
```

- `write()` 返回成功 ≠ 数据落盘，只是进了 PageCache（脏页）。内核 pdflush/flusher 线程按 `dirty_expire_centisecs`（默认 30s）等参数异步刷盘。
- **断电/内核崩溃会丢 PageCache 中的数据；进程崩溃不会丢**（数据已在内核）。这个区分是"刷盘策略"面试题的核心。
- 读消息时，消费者通常追尾读（tail read），数据大概率还在 PageCache 里，**根本不用读盘**。
- MQ 把"冷热分离"交给 OS：热数据在 PageCache，冷数据自动换出。这就是 Kafka 敢用 JVM 之外的内存做缓存的原因——**不占堆、不怕 GC、进程重启缓存还在**。

**延伸：为什么不自己维护内存缓存（像 Redis 那样）？**
1. JVM 堆内缓存大对象 → Full GC 灾难；堆外要自己管理生命周期。
2. 自己缓存 + OS PageCache = 双份内存浪费。
3. OS 的换页算法经过几十年打磨，比你写得好。
4. 进程重启后 PageCache 依然有效，预热成本为零。

## 2.3 传统 IO 的四次拷贝与两次零拷贝方案

**传统方式：把磁盘文件发到网卡（read + write）**

```
磁盘 ──DMA拷贝①──> PageCache ──CPU拷贝②──> 用户态Buffer
                                              │ CPU拷贝③
网卡 <──DMA拷贝④── SocketBuffer <─────────────┘
共 4 次拷贝（2 DMA + 2 CPU）+ 4 次用户态/内核态切换（read/write 各 2 次）
```

**方案一：mmap + write（RocketMQ 采用）**

```
磁盘 ──DMA──> PageCache <──共享映射──> 用户态可直接读写（无CPU拷贝）
                │ CPU拷贝
                └──> SocketBuffer ──DMA──> 网卡
3 次拷贝（1 次 CPU）+ 4 次上下文切换
```

- `mmap()` 把文件的 PageCache 直接映射进进程虚拟地址空间，用户态读写指针 = 读写 PageCache，**省掉"内核→用户"那次 CPU 拷贝**。
- 适合**小块、需要读写内容**的场景：RocketMQ 要按消息格式解析/构造消息，必须能"看到"内容，所以用 mmap。
- RocketMQ 映射粒度：CommitLog 单文件 1GB，用 `MappedByteBuffer`（Java 对 mmap 的封装）。

**方案二：sendfile（Kafka 消费链路采用）**

```
磁盘 ──DMA──> PageCache ──CPU拷贝──> SocketBuffer ──DMA──> 网卡   (Linux 2.1)
磁盘 ──DMA──> PageCache ──(只传描述符/偏移)──> 网卡 DMA 直接从 PageCache  gather  (Linux 2.4 + SG-DMA)
最终：2 次 DMA 拷贝 + 0 次 CPU 拷贝 + 2 次上下文切换
```

- `sendfile()` 一个系统调用完成"文件→socket"，数据**全程不进用户态**。
- Java 对应 `FileChannel.transferTo()`。Kafka 消费拉取走这条路。
- 代价：用户态碰不到数据，**无法加工内容**。所以 Kafka 的压缩/加密若要在 Broker 端做就会破坏零拷贝（这也是"Broker 尽量不解压、原样存储压缩批"的原因）。

**对比记忆表**：

| | mmap+write | sendfile |
|---|---|---|
| CPU 拷贝 | 1 次 | 0 次（SG-DMA） |
| 用户态可见数据 | 是（可修改） | 否 |
| 适合 | 小块读写、需解析（RocketMQ） | 大块透传（Kafka 消费） |
| Java API | MappedByteBuffer | FileChannel.transferTo |

**延伸追问：mmap 的缺点？**
1. 映射/解除映射本身有开销（页表建立、TLB 刷新），所以 RocketMQ 用 1GB 大文件减少映射次数，并做**文件预热**（提前逐页写 0 触发缺页 + `mlock` 锁页防止被 swap 出去 + `madvise`）。
2. 缺页中断（page fault）是同步阻塞的：第一次访问某页要从磁盘加载，冷读会卡住。
3. `MappedByteBuffer.force()` 才真正刷盘（对应 msync），close 不保证落盘。

## 2.4 刷盘（Flush）：性能与可靠性的旋钮

| 策略 | 做法 | 丢数据窗口 | 性能 |
|---|---|---|---|
| 异步刷盘 | write 进 PageCache 即返回，后台线程定期 force | 机器断电丢几秒 | 最高 |
| 同步刷盘 | 每条/每批消息 force() 落盘后才返回成功 | 不丢（单机） | 下降明显 |

- `fsync` 昂贵的原因：要等磁盘真正把数据写进介质（SSD 的掉电保护电容/机械盘落磁道），一次约几毫秒~几十毫秒。
- **组提交（Group Commit）**：同步刷盘不是每条消息 fsync 一次，而是攒一批一起 fsync，摊薄成本——RocketMQ 的 `GroupCommitService`、MySQL redo log 组提交、Kafka 依赖副本而非 fsync，都是同一思想。
- **可靠性更优的路线是"多副本"而非"同步刷盘"**：单机 fsync 防不了磁盘损坏；Kafka 用 `acks=all + min.insync.replicas=2` 把可靠性押在"多台机器同时断电概率极低"上，单机可以异步刷盘（甚至不主动 fsync），性能与可靠性兼得。这是两大 MQ 可靠性哲学的分野。

## 2.5 稀疏索引：用少量索引换取快速定位

Kafka 不给每条消息建索引（索引会膨胀、写入变慢），而是**每隔约 4KB 数据记一个索引项**（offset → 物理位置）。查找时：二分找到 ≤ 目标 offset 的最近索引项 → 从该物理位置顺序扫描少量数据。

- 索引文件小到可以整个 mmap 常驻内存；
- 写入路径几乎无索引维护成本；
- 查询多扫几 KB，PageCache 命中下几乎免费。

RocketMQ 的 ConsumeQueue 则是"稠密定长索引"（每条消息 20 字节），两者取舍不同，第三、四部分详细对比。

## 2.6 epoll 与 Reactor：网络层的底层逻辑

**IO 多路复用演进**：select（fd 上限 1024，每次全量拷贝+遍历）→ poll（链表无上限，仍全量遍历）→ **epoll**（内核红黑树管理 fd + 就绪链表回调，`epoll_wait` 只返回就绪 fd，O(就绪数)，fd 只注册一次）。

**Reactor 模式（Netty 主从 Reactor）**：

```
                    ┌── EventLoop1 (Thread + Selector + 任务队列) ── Channel A,B...
BossGroup(accept) ──┼── EventLoop2 ── Channel C,D...
                    └── EventLoop3 ── ...
每个 Channel 终生绑定一个 EventLoop → 单线程串行处理该连接所有事件 → 无锁化
耗时业务逻辑必须丢到独立业务线程池，否则阻塞 EventLoop 拖死同 loop 上所有连接
```

- 一个 EventLoop = 一个线程 + 一个 Selector + 一个任务队列，**线程亲和**避免了锁和上下文切换。
- RocketMQ Remoting、Dubbo、gRPC-Java 底层全是 Netty，这套模型是它们的共同地基。
- **延伸：为什么 Netty 快？** 除了 Reactor，还有：池化 ByteBuf（jemalloc 思想，减少 GC 与 malloc）、CompositeByteBuf/slice 逻辑零拷贝、FastThreadLocal、无锁串行化设计、`LengthFieldBasedFrameDecoder` 解决 TCP 粘包拆包（TCP 是字节流没有消息边界，应用层必须自定义"长度字段+消息体"或分隔符）。

---
# 第三部分：RocketMQ 深度拆解

## 3.1 整体架构：四大角色

```
                 ┌────────────┐  ┌────────────┐
                 │ NameServer1│  │ NameServer2│   ← 无状态、彼此不通信（AP）
                 └─────▲──────┘  └─────▲──────┘
        注册/心跳(30s)  │               │  拉取路由(30s)
   ┌───────────────────┼───────────────┼──────────────────┐
   │                   │               │                  │
┌──┴─────────┐   ┌─────┴──────┐   ┌────┴───────┐   ┌──────┴─────┐
│ Producer   │──>│ Broker     │<──│ Broker     │   │ Consumer   │
│ Group      │发送│ Master(a)  │复制│ Slave(a)   │拉取│ Group      │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
```

- **NameServer**：极简注册中心。每个节点保存全量路由（Topic→队列→Broker 地址），**节点间互不通信**。Broker 向**所有** NameServer 注册，每 30s 心跳；NameServer 120s 没收到心跳就剔除 Broker。客户端每 30s 拉一次路由。
  - **为什么不用 ZooKeeper？** 路由发现只需要 AP（可用性优先）：短暂的路由不一致可以靠客户端重试兜底，而 ZK 的 CP 在选主期间整个注册中心不可用，代价过大。这是"CAP 按场景裁剪"的经典案例（对比 Kafka 早期用 ZK 存元数据+选 Controller，后来干脆自研 KRaft）。
  - 注意：客户端感知 Broker 宕机最坏要 30s（路由刷新）+ 发送重试兜底，所以 Producer 的**故障延迟规避（latencyFaultTolerance）**很重要：某 Broker 发送超时后，一段时间内自动避开它。
- **Broker**：存储与转发核心。分 Master/Slave（或 DLedger 组），主从都可供读。
- **Producer/Consumer**：以 Group 为单位，Group 内实例分摊队列。

## 3.2 存储引擎：CommitLog + ConsumeQueue + IndexFile（核心中的核心）

### 3.2.1 三件套结构

```
store/
 ├── commitlog/                     ← 所有 Topic 的消息混合顺序写在一起
 │    ├── 00000000000000000000      ← 文件名 = 该文件第一条消息的全局物理偏移量(20位)
 │    ├── 00000000001073741824      ← 每个文件固定 1GB
 │    └── ...
 ├── consumequeue/                  ← 逻辑消费索引（按 Topic/Queue 分目录）
 │    └── TopicA/0/00000000000000000000
 │         每条目固定 20 字节：
 │         [8B CommitLog物理offset][4B 消息长度][8B Tag的hashcode]
 │         单文件 30万条 ≈ 5.72MB
 ├── index/                         ← 按 MessageKey/时间 查消息的哈希索引
 │    └── 20261119xxxx (单文件≈400MB: 40B头 + 500万slot×4B + 2000万条目×20B)
 └── config/consumerOffset.json 等
```

**设计精髓：一份数据，多份索引。**
- **CommitLog**：消息的唯一真实存储（Source of Truth），所有 Topic、所有 Queue 的消息**混写同一个文件序列**。写入永远是对"当前最后一个文件"的顺序追加 → 无论有多少 Topic/分区，磁盘上只有一条顺序写流。
- **ConsumeQueue**：消费视角的索引。消费者按 Topic+Queue 消费，但数据混在 CommitLog 里，怎么办？后台线程把每条消息的"位置指针"分发到对应队列的索引文件里。消费时先读 ConsumeQueue（20B 定长，`queueOffset × 20` 直接定位，O(1)），再按指针去 CommitLog 取消息体。
- **IndexFile**：支持按 Message Key / UniqKey / 时间范围查询（运维排查用）。结构是**磁盘上的 HashMap**：key 的 hash → slot（槽存最新条目编号）→ 条目内记录 next 指针形成链表解决哈希冲突（头插法）。

### 3.2.2 消息在 CommitLog 中的物理格式（源码级）

```
TOTALSIZE(4) MAGICCODE(4) BODYCRC(4) QUEUEID(4) FLAG(4)
QUEUEOFFSET(8) PHYSICALOFFSET(8) SYSFLAG(4) BORNTIMESTAMP(8) BORNHOST(8)
STORETIMESTAMP(8) STOREHOST(8) RECONSUMETIMES(4) PreparedTransactionOffset(8)
BODYLEN(4)+BODY  TOPICLEN(1)+TOPIC  PROPERTIESLEN(2)+PROPERTIES
```

延伸点：
- `QUEUEOFFSET`：该消息在逻辑队列中的序号（消费位点就是它）；`PHYSICALOFFSET`：在 CommitLog 的全局物理偏移。两套 offset 体系不要混淆。
- 消息属性（tags、keys、延迟级别、事务标记）序列化在 PROPERTIES 里，SQL92 过滤就是对它求值。
- 单条消息默认上限 4MB；大消息应走"对象存储 + 传引用"模式。

### 3.2.3 写入与分发全链路（异步构建索引）

```
Producer ──Netty──> Broker
  ① SendMessageProcessor 校验
  ② CommitLog.putMessage:
      获取写锁(putMessageLock: 自旋锁或ReentrantLock)
      → 找到当前 MappedFile(1GB, mmap)
      → 追加写入 MappedByteBuffer(即 PageCache)
      → 释放锁
  ③ 刷盘:
      同步刷盘: GroupCommitService 攒批 force(), 等待刷盘结果(默认超时5s)才返回SEND_OK
      异步刷盘: FlushRealTimeService 每500ms或脏页≥4页 force(), 写完PageCache即返回
  ④ HA复制(异步或同步等待slave ack)
  ⑤ ReputMessageService(每1ms): 从CommitLog重放新消息
      → DispatchRequest → 构建 ConsumeQueue 条目 + IndexFile 条目
      → 同时唤醒长轮询: NotifyMessageArrivingListener → PullRequestHoldService
```

**关键结论**：
1. ConsumeQueue 是**异步**构建的 → 消息写成功后有毫秒级"不可消费窗口"；
2. 写路径只有一条顺序写（CommitLog），ConsumeQueue 的写虽然按队列分散，但每个文件极小（5.72MB）且基本全在 PageCache，实际落盘也是准顺序；
3. Broker 宕机重启后靠 **Recover 流程**：根据 checkpoint/刷盘位点对 CommitLog 做 CRC 校验恢复，ConsumeQueue/IndexFile 可以**完全重建**（它们只是索引，丢了不丢数据）。

### 3.2.4 transientStorePool：读写分离的堆外写缓冲（高阶延伸）

开启 `transientStorePoolEnable` 后：

```
写: Producer消息 → DirectByteBuffer池(堆外, mlock锁页) ──commit──> FileChannel(PageCache) ──flush──> 磁盘
读: 消费者 ← mmap 的 PageCache
```

- 写入先进堆外内存池（默认 5 个 1GB buffer），后台 CommitRealTimeService 每 200ms 批量 commit 到 FileChannel，再走刷盘。
- 目的：**写走堆外、读走 PageCache，读写分离**，避免读写都挤在 PageCache 上互相争抢导致毛刺；mlock 保证写缓冲不被 swap。
- 代价：机器断电时，堆外 buffer 中未 commit 的数据丢失（进程崩溃不丢，因为堆外内存独立于 JVM？——注意：进程崩溃堆外 buffer 也没了，所以**此模式只允许异步刷盘 + 主从架构**，可靠性靠副本兜底）。

### 3.2.5 文件预热与 PageCache 治理（专家级延伸）

- **预热**：新建 1GB MappedFile 后逐页写 0（触发缺页中断，把页表建好、物理页分配好），可选 `mlock` 锁定 + `madvise(WILLNEED)`。避免消息写入时撞上缺页中断造成 RT 毛刺。
- **冷读保护**：消费者拉很久以前的消息（冷读）会触发磁盘 IO 并污染 PageCache。RocketMQ 用 `accessMessageInMemoryMaxRatio`（默认 40%）判断消息是否"在内存中"，冷读时建议客户端下次从 Slave 拉（`suggestWhichBrokerId`），保护 Master 的 PageCache。
- **文件删除**：默认保留 72 小时（`fileReservedTime`），每天凌晨 4 点（`deleteWhen`）删过期文件；磁盘水位超 85%/90% 触发强制清理/拒写。

## 3.3 高可用：三代方案演进

### 第一代：Master-Slave（HAService，4.x 默认）

- Slave 与 Master 建立连接，上报自己的 commitlog 最大偏移；Master 的 HAConnection 持续推送增量数据。
- `brokerRole=SYNC_MASTER`：发消息时 GroupTransferService 等待 Slave 确认已收到（注意：**是收到写入 PageCache，不是落盘**）才返回；`ASYNC_MASTER` 不等。
- **缺陷：Master 挂了 Slave 不能自动升主**（只能人工切换或 Slave 提供读），因为缺乏选主协议，贸然升主会脑裂/丢数据。

### 第二代：DLedger（Raft，4.5+）

- 用自研 DLedger（Raft 实现）接管 CommitLog：**一组至少 3 节点**，写入需**多数派（quorum）确认**，Leader 挂了自动选举新 Leader（秒级）。
- Raft 核心三件事（必须能讲清）：
  1. **Leader 选举**：任期 term + 随机超时投票，多数派同意即当选；随机超时避免瓜分选票。
  2. **日志复制**：Leader 追加日志 → 并行发给 Follower → 多数派持久化后 commit → 应用到状态机。
  3. **安全性**：只有日志最新（term/offset 最大）的节点能当选 → 已提交数据永不丢。
- 代价：3 副本起步（成本↑）、多数派确认延迟（性能比两副本异步略降）、DLedger 格式与原 CommitLog 不完全兼容（升级要迁移）。

### 第三代：Controller 模式（5.x，SyncStateSet）

- 引入可部署在 NameServer 内的 Controller（基于 DLedger 选主），Broker 组仍是**主从复制**（不改成 Raft 日志），Controller 维护 **SyncStateSet**（类似 Kafka ISR 的"跟得上的副本集合"），Master 挂了从 SyncStateSet 里选新主。
- 思想：**把"选主"与"数据复制"解耦**——复制走高效主从，选主走少数派 Raft 控制器。兼得性能与自动切换，且兼容旧存储格式。Kafka KRaft 与它异曲同工。

## 3.4 消费链路：Push 皮、Pull 骨

### 3.4.1 客户端内部结构

```
DefaultMQPushConsumer
  ├── RebalanceService(每20s): 决定本实例分到哪些 MessageQueue
  ├── PullMessageService: 单线程循环处理 PullRequest 队列
  │     每个队列一个 PullRequest → 异步拉取(默认批32条) → 回调
  │     成功 → 消息放入 ProcessQueue(TreeMap<queueOffset, Msg>) → 立即把下一个PullRequest塞回队列(流式拉取)
  ├── ConsumeMessageConcurrentlyService: 业务线程池(默认20~64线程)并发消费
  └── OffsetStore: 集群模式提交到Broker(RemoteBrokerOffsetStore, 每5s持久化consumerOffset.json)
                  广播模式存本地文件
```

**流控三道闸**（Pull 阶段检查 ProcessQueue）：
- 缓存消息数 > 1000 → 延迟 50ms 再拉；
- 缓存体积 > 100MB → 延迟；
- 并发消费时最大 offset 跨度 > 2000 → 延迟（防止一条慢消息卡住导致堆积爆炸）。

### 3.4.2 位点提交的秘密：提交的是 ProcessQueue 最小 offset

并发消费下，消息 5、6、7 完成但 4 还没完成 → 只能提交 4（最小未完成位点）。若此时消费者崩溃重启，5/6/7 会**重复消费** → 再次论证"幂等是消费端必修课"。

### 3.4.3 Broker 端长轮询实现

```
PullMessageProcessor 收到拉请求 → ConsumeQueue 无新消息
  → 不立即返回, 创建 PullRequest 挂到 PullRequestHoldService(按 topic@queueId 分组)
  → 两条唤醒路径:
     ① ReputMessageService 分发新消息时 → NotifyMessageArrivingListener → 立即唤醒对应挂起请求
     ② PullRequestHoldService 每5s兜底检查一次
  → 唤醒后重新查 ConsumeQueue, 有数据则读 CommitLog 返回
  → 挂起超时(长轮询15s / 未开启hold则1s)返回 NO_NEW_MSG
```

### 3.4.4 Rebalance（队列级独占分配）

- 触发：每 20s 定时 + 消费者上下线/订阅变化时 Broker 通知。
- 步骤：拿到 Topic 的全部队列 + Group 内全部消费者 ID → 双方排序 → 按策略分配（默认**平均分配 AllocateMessageQueueAveragely**，另有环形平均、一致性哈希、机房就近等）。
- **一个队列同一时刻只被组内一个消费者消费** → 消费者实例数 > 队列数时多出来的实例空转（扩容无效的经典原因）。
- Rebalance 瞬间队列易主，新旧消费者可能短暂同时持有 → 又一个小概率重复消费来源。

## 3.5 事务消息：二阶段提交 + 回查（半消息机制）

解决"**本地事务与发消息的原子性**"（如：下单落库 + 发扣库存消息，要么都成功要么都不发生）。

```
①Producer ──发半消息(half)──> Broker
     Broker 把它改写 Topic 为 RMQ_SYS_TRANS_HALF_TOPIC(真实topic/queue存进属性)
     → 对消费者不可见
②Broker 返回半消息写入成功
③Producer 执行本地事务(DB操作)
④根据本地事务结果发送 Commit / Rollback
     Commit: Broker 把半消息还原为真实Topic写入CommitLog → 消费者可见
             同时向 RMQ_SYS_TRANS_OP_HALF_TOPIC 写一条op消息标记"已处理"
     Rollback: 只写op标记, 半消息废弃
⑤若④丢失/Producer宕机: TransactionalCheckService 每60s扫描
     half topic 中没有对应op消息、且超过免疫期(6s)的半消息
     → 回查 Producer.checkLocalTransaction()(生产者组内任一存活实例)
     → 默认最多回查15次, 仍无结果则默认回滚
```

**深挖点**：
- 为什么半消息要换 Topic？——复用存储引擎，不改 CommitLog 结构就实现"写入但不可见"，靠 ConsumeQueue 不构建原 Topic 索引达成隔离。**这是"用索引换可见性"的巧妙设计**。
- 回查为什么发给整个 Producer Group？——原实例可能已宕机，同组任一实例都能查 DB 判断事务状态（所以本地事务状态必须可查，比如订单表有状态字段）。
- 事务消息只保证"上游本地事务与消息发送"的原子性，**不保证下游消费成功**；下游要靠重试+幂等+最终一致，必要时人工介入。它是"可靠消息最终一致性"方案，不是分布式强事务（对比 Seata AT/TCC）。

## 3.6 延迟消息：4.x 固定级别 vs 5.x 时间轮

**4.x：18 个固定级别**（1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h）

```
写入时: 消息Topic被改写为 SCHEDULE_TOPIC_XXXX, queueId = delayLevel - 1
        (真实topic/queue存属性) → 同级别消息天然按到期时间有序!
ScheduleMessageService: 每个level一个TimerTask, 只盯队首
        到期 → 还原真实Topic重新写CommitLog → 消费者可见
```

为什么只支持固定级别？——**同级别延迟相同 → 写入顺序 = 到期顺序 → 每个队列只需检查队首**，O(1) 调度，无需排序。若支持任意延迟，就要对海量消息按到期时间排序，代价大。

**5.x：任意精度定时（TimerWheel + TimerLog）**
- **时间轮**：秒级 slots 数组（类似钟表盘），消息按到期秒挂到对应 slot 链表；指针每秒走一格，处理当前格全部消息。跨多圈的记录圈数（round）。
- **TimerLog**：时间轮槽位持久化到磁盘日志（顺序写），重启可恢复；wheel 只存指针，海量定时消息不占内存。
- 同一思想在 Netty `HashedWheelTimer`、Kafka 时间轮（分层时间轮：秒轮→分轮→时轮，降级推进）中反复出现——**时间轮是"海量定时任务 O(1) 调度"的通用解**（对比 JDK DelayQueue 堆结构 O(logN)）。

## 3.7 顺序消息

- **全局有序**：Topic 只设 1 个队列 + 单线程消费（吞吐极低，很少用）。
- **分区有序（主流）**：同一业务键（订单号）hash 选队列 → 同键消息进同一队列；生产端对该队列**同步串行发送**；消费端用 `MessageListenerOrderly`：
  - 先向 Broker 申请**队列分布式锁**（Rebalance 后新消费者要等旧锁释放，默认锁 60s、每 20s 续期）+ 本地 ProcessQueue 锁；
  - 单线程顺序消费该队列，失败**本地重试并阻塞后续**（不像并发消费丢回重试队列——那样会破坏顺序）。
- **代价与坑**：热点 key 造成单队列积压；一条消息消费失败会阻塞整个队列；Broker 宕机队列数变化时 hash 路由改变，短暂乱序窗口。

## 3.8 重试与死信

```
并发消费失败(RECONSUME_LATER) → 消息发回Broker → %RETRY%{consumerGroup} 重试Topic
  → 利用延迟消息机制, 第n次重试延迟级别 = n+2 (10s,30s,1m,2m...2h)
  → 默认16次仍失败 → %DLQ%{consumerGroup} 死信队列(需人工/补偿任务处理)
```

消费组订阅时自动附加订阅自己的 %RETRY% Topic——重试消息本质是"延迟后重新投递给本组"。

## 3.9 通信层：Remoting 协议

Netty + 自定义二进制协议：

```
[4B 总长度][4B 头部长度|序列化类型][Header(JSON或RocketMQ私有编码)][Body]
RemotingCommand: code(请求码) / opaque(请求ID, 用于响应匹配) / flag(请求响应/单向)
```

- `opaque` 请求 ID + `ResponseFuture` 表实现**单连接多路复用**（与 Dubbo requestId、HTTP/2 streamId 同一思想）。
- 默认 JSON 序列化 Header（可读性好、性能一般），可切 RocketMQ 私有编码。Body 就是消息字节，不二次序列化。

## 3.10 RocketMQ 5.x 关键演进（面试加分项）

1. **Proxy 层**：计算存储分离，客户端极简化（gRPC 协议多语言 SDK），Proxy 可 Local（与 Broker 同进程）或 Cluster 部署。
2. **Pop 消费**：无队列绑定的新消费模式——消费者不独占队列，向 Broker "Pop"一批消息，Broker 记录 CheckPoint，超过 invisibleTime 未 Ack 则通过 Revive 机制重投。**消灭了 Rebalance 抖动与"实例数>队列数空转"问题**，向 SQS 式无状态消费靠拢（代价：Broker 端状态管理更重）。
3. **任意时间定时消息**（3.6 的时间轮）、**逻辑位点/RocksDB ConsumeQueue**（海量队列下索引膨胀治理）、**Controller 自动主从切换**（3.3 节）。

---
# 第四部分：Kafka 深度拆解

## 4.1 架构与核心概念

```
                    ┌───────────────────────────────┐
                    │ Controller (KRaft Quorum)     │ ← 元数据/选主
                    └──────────────▲────────────────┘
                                   │ metadata log 复制
Producer ──batch──> Broker1 [P0-Leader, P1-Follower]
                    Broker2 [P1-Leader, P0-Follower]
                    Broker3 [P0-Follower, P1-Follower]
Consumer Group ──fetch──> 各分区 Leader
```

- **Topic → Partition**：分区是并行与有序的基本单位。分区内有序，分区间无序。
- **副本**：每分区 1 Leader + N Follower，**读写都走 Leader**（Follower 只同步；2.4+ 支持 follower fetching 仅为就近读，不承担写）。与 RocketMQ "Slave 可读"不同。
- **AR / ISR / OSR**：AR=全部分配副本；ISR=与 Leader 保持同步的副本集合（含 Leader）；OSR=落后被踢出的。判定标准 `replica.lag.time.max.ms`（默认 30s）：Follower 只要在 30s 内追上过 Leader 的 LEO 就留在 ISR——**按时间而非条数**（0.9 之前按条数，突发流量会误踢，已废弃）。

## 4.2 日志存储：分区独立文件 + 稀疏索引

```
topicA-0/
 ├── 00000000000000000000.log        ← segment 数据文件(默认1GB或7天滚动)
 ├── 00000000000000000000.index      ← 位移索引(稀疏): [4B相对offset][4B物理位置]
 ├── 00000000000000000000.timeindex  ← 时间索引(稀疏): [8B时间戳][4B相对offset]
 ├── 00000000000003687691.log        ← 文件名 = 该segment第一条消息的offset(baseOffset)
 └── leader-epoch-checkpoint
```

**查找 offset=N 的消息（三步定位，必考）**：
1. **找 segment**：内存 ConcurrentSkipListMap 按 baseOffset `floorEntry(N)` → 二分/跳表定位文件；
2. **查稀疏索引**：.index 整个 mmap 进内存，二分找 ≤N 的最近索引项（相对 offset 省空间），得到物理位置；
3. **顺序扫描 .log**：从该位置向后扫到目标消息（最多扫 index.interval.bytes≈4KB）。

**消息格式 v2（RecordBatch）**：批为单位存储，批头含 baseOffset、lastOffsetDelta、producerId、producerEpoch、baseSequence（幂等去重三件套）、压缩标记；批内 record 用 **varint + 增量编码**（offsetDelta、timestampDelta），极致压缩空间。

**保留策略**：delete（按时间 168h / 大小，整段 segment 删除）或 compact（log cleaner 后台按 key 保留最新值，tombstone 表示删除——`__consumer_offsets` 就是 compact topic）。

## 4.3 Kafka 为什么快（系统性总结）

1. **分区级顺序追加写**（注意与 RocketMQ 的区别，4.6 详述）；
2. **PageCache 而非 JVM 缓存**（重启不失效、无 GC）；
3. **sendfile 零拷贝**（消费链路 FileChannel.transferTo，0 次 CPU 拷贝）；
4. **批量 + 压缩**：Producer 攒批（batch.size=16KB / linger.ms），按批压缩（lz4/zstd/snappy），Broker 原样存储压缩批（不解压 → 不破坏零拷贝），Consumer 解压。网络与磁盘传输的是压缩后字节；
5. **稀疏索引**：索引小、常驻内存、写入几乎零成本；
6. **分区水平扩展**：吞吐随分区/机器线性扩展。

## 4.4 副本一致性的基石：LEO、HW 与 Leader Epoch

### 4.4.1 基本定义

- **LEO（Log End Offset）**：每个副本下一条待写消息的 offset（自己日志末端）。
- **HW（High Watermark）**：**ISR 中最小的 LEO**。消费者只能读到 HW-1 的消息——保证读到的消息一定已被所有 ISR 副本复制，Leader 挂了换主也不会"读到又消失"。

### 4.4.2 HW 更新的两轮 Fetch 时序（细节考点）

```
Follower 发 FETCH(带自己的LEO) → Leader:
   ① 返回数据 + Leader当前HW
   ② 用Follower上报的LEO更新该Follower的远端LEO → 尝试推进Leader HW = min(ISR LEO)
Follower 写入数据提升自己LEO, 并把响应里的HW记为自己HW
→ HW 的推进天然滞后一轮 fetch: 这就是"只靠HW做截断"会出问题的根源
```

### 4.4.3 只靠 HW 截断的两大事故 → Leader Epoch（KIP-101/279）

**事故一（丢数据）**：Follower B 写入 m1 但 HW 还没更新（滞后一轮），B 重启 → 按旧 HW 截断丢掉 m1；此时 Leader A 也挂了，B 当选 → m1 永久丢失（尽管 A 曾 ack 过）。
**事故二（数据不一致）**：A、B 同时挂，B 先起来当选并写入新消息 m2'（同 offset 不同内容），A 起来按自己 HW 不截断 → 同 offset 两个副本内容分叉。

**解法：Leader Epoch**——每次选主 epoch+1，副本记录 `<epoch, endOffset>` 序列（leader-epoch-checkpoint）。恢复时不再盲目按 HW 截断，而是**向 Leader 询问"我这个 epoch 的日志应该到哪个 offset"**（OffsetsForLeaderEpoch 请求），按精确边界截断。一句话：**HW 决定"消费者能读到哪"，Leader Epoch 决定"副本恢复时截断到哪"**。

### 4.4.4 可靠性配置铁三角

```
acks=all(-1)                 ← Leader等ISR全部确认才ack
min.insync.replicas=2        ← ISR不足2个时直接拒写(宁可不可用也不裸奔)
unclean.leader.election.enable=false  ← 禁止OSR落后副本当选Leader(否则丢已确认数据)
replication.factor=3
```

注意 `acks=all` 等的是 **ISR 全体写入（PageCache）**，不是 fsync——Kafka 的可靠性哲学是"副本冗余 > 单机刷盘"（对照第二部分 2.4）。

## 4.5 Controller 与 KRaft：去 ZooKeeper 之路

**旧架构（ZK 时代）**：Broker 抢建 ZK 临时节点 `/controller` 成为 Controller；负责分区选主、副本状态机、监听 Broker 上下线，并把变更通过 LeaderAndIsr/UpdateMetadata 请求广播给所有 Broker。
痛点：元数据双写（ZK + 内存）不一致、Controller 切换要全量加载 ZK（分区多时分钟级）、ZK 运维复杂、watch 风暴。

**KRaft（KIP-500，3.3 生产可用，4.0 彻底移除 ZK）**：
- 元数据本身变成一个**内部 topic `__cluster_metadata`**，由 Controller Quorum（通常 3 个 controller 节点）用 **Raft** 复制；Active Controller 就是 Raft Leader。
- Broker 作为元数据日志的"订阅者"增量同步 → **事件溯源（Event Sourcing）**思想：状态 = 日志回放，天然一致。
- 收益：切换秒级、支持百万分区、部署少一个组件、元数据传播从"推全量"变"拉增量"。

## 4.6 Kafka 分区文件 vs RocketMQ 单一 CommitLog（架构级对比，必考）

| 维度 | Kafka（每分区独立日志） | RocketMQ（全局 CommitLog + 队列索引） |
|---|---|---|
| 写入流形 | 分区少时各文件顺序写；**分区多时（成百上千），多文件并发追加在磁盘视角退化为随机写** | 永远只有 1 条顺序写流，Topic/队列数不影响写模式 |
| 读取 | 直接读分区文件，**一次 IO**，天然适配 sendfile | 先读 ConsumeQueue 再跳 CommitLog，**两跳**，消息体分散 |
| 海量 Topic | 性能随分区数上涨明显劣化（数千分区拐点） | 支撑数万 Topic/队列仍平稳 |
| 功能扩展 | 结构简单，流处理生态强 | 索引化设计易加延迟/事务/重试等特性 |
| 适用 | 少 Topic 大吞吐（日志/流/埋点） | 多 Topic 业务消息（电商交易链路） |

一句话总结：**Kafka 为"吞吐"优化（直读直写），RocketMQ 为"功能与规模"优化（一份数据多份索引，读写两跳换写入恒定顺序）**。

## 4.7 Producer 深度：分区器、幂等、事务

### 4.7.1 发送链路与分区策略

```
send() → 拦截器 → 序列化 → Partitioner → RecordAccumulator(按分区攒批, BufferPool复用内存)
       → Sender线程取出就绪批 → NetworkClient → Broker
就绪条件: 批满(batch.size) 或 等待超时(linger.ms) 或 内存耗尽 或 flush/close
```

- 有 key：`murmur2(key) % 分区数` → 同 key 同分区（分区有序的根基）。**扩分区会改变 hash 路由 → 同 key 历史消息与新消息分家，顺序被破坏**（经典坑）。
- 无 key：2.4+ **粘性分区器（Sticky Partitioner）**——粘住一个分区直到批满再换，减少小批数量、提升压缩率与吞吐（对比老版轮询把消息摊薄到所有分区）。

### 4.7.2 幂等 Producer（解决"重试导致的单分区重复"）

```
enable.idempotence=true (3.0+默认开启)
Broker为Producer分配 PID; Producer对每个分区维护单调递增 sequence
每条批携带 <PID, ProducerEpoch, baseSequence>
Broker校验: 新seq == 缓存seq+1 → 接受
            seq <= 缓存seq → 重复, 直接ack但不再写(去重)
            seq > 缓存seq+1 → 乱序, 报OutOfOrderSequence
```

- 限制 1：**只保证单分区、单会话（Producer 重启 PID 变了）**去重；跨分区、跨重启不管。
- 限制 2：Broker 只缓存每分区最近 5 个批的元数据用于校验 → 要求 `max.in.flight.requests.per.connection ≤ 5`，否则失败重试可能乱序。
- ProducerEpoch 用于**僵尸隔离（fencing）**：同 transactional.id 的新实例 init 时 epoch+1，旧实例（假死复活）的请求被拒。

### 4.7.3 事务 Producer（跨分区原子写 + EOS）

```
①initTransactions(): 按transactional.id找TransactionCoordinator, 获取PID并bump epoch(隔离僵尸)
②beginTransaction()
③send()到多个分区(首次涉及某分区时向Coordinator注册该分区)
④(可选) sendOffsetsToTransaction(): 把消费位移也注册进事务
⑤commitTransaction():
   Coordinator把事务状态(含分区列表)写入 __transaction_state 并复制
   → 向各涉及分区写 COMMIT marker(控制消息)
   → 事务完成
read_committed 的消费者: 只能读到 LSO(Last Stable Offset, 最早未完成事务之前的位置)以下,
   且过滤 ABORT 事务的消息; 位移提交读到的也是事务化提交
```

**EOS（Exactly Once Semantics）边界**：Kafka→处理→Kafka 的闭环（Kafka Streams 的 exactly_once_v2）内成立；**一旦结果写到外部 DB/HTTP，事务管不到**，仍需幂等或 outbox 模式。面试说出这个边界是加分项。

## 4.8 Consumer Group 与 Rebalance 协议

### 4.8.1 位移管理

- 位移存内部 topic `__consumer_offsets`（50 分区，compact），分区路由 = `hash(groupId) % 50`；key = (group, topic, partition)。
- 提交的是"**下一条要消费的 offset**"（processed+1）。自动提交（enable.auto.commit=true，每 5s 在 poll 时提交上一批）→ 处理未完成就提交会丢；处理完再手动提交 → 崩溃窗口重复。**先处理后提交 + 幂等**是标准姿势。

### 4.8.2 Rebalance 全流程（组协调者协议）

```
GroupCoordinator = 该group的__consumer_offsets分区所在Leader Broker
① 所有成员发 JoinGroup → Coordinator选第一个成员为 Consumer Leader
② Leader 拿到全组成员订阅信息, 按分配策略计算分配方案
③ 各成员 SyncGroup → Coordinator把Leader算好的方案切片下发
④ 进入 Heartbeat 稳定态; generation.id 每轮+1(隔离旧世代成员的提交, fencing)
触发: 成员增减 / 订阅topic变化 / 分区数变化 / 心跳超时(session.timeout)
      / 处理超时(max.poll.interval.ms, 默认5min, 两次poll间隔过长被视为卡死)
```

**Eager vs Cooperative（KIP-429）**：
- Eager（旧）：Rebalance 开始**所有消费者先撤销全部分区**（Stop-The-World），重新分配后恢复 → 大组抖动剧烈。
- Cooperative Incremental（新）：只撤销"需要易主"的分区，其余继续消费；两轮 rebalance 完成迁移。`CooperativeStickyAssignor` 兼顾粘性与增量。
- **静态成员（group.instance.id，KIP-345）**：重启在 session.timeout 内回来不触发 rebalance（滚动发布友好）。

### 4.8.3 消费拉取细节

- `fetch.min.bytes` + `fetch.max.wait.ms`：Broker 攒够数据或超时才返回（服务端攒批，类似长轮询）。
- `max.poll.records`（默认 500）控制单批处理量，配合 max.poll.interval.ms 防"处理太慢被踢"。
- 一个分区同一时刻只被组内一个消费者消费 → 消费者数 > 分区数则空转（与 RocketMQ 同理）。

---

# 第五部分：RocketMQ vs Kafka vs RabbitMQ 选型对比

| 维度 | RocketMQ | Kafka | RabbitMQ |
|---|---|---|---|
| 语言/生态 | Java，阿里系业务消息 | Scala/Java，大数据流处理霸主 | Erlang，AMQP 传统企业 |
| 单机吞吐 | 十万级 | 百万级（批+压缩+少分区） | 万级 |
| 延迟 | ms 级 | ms 级（攒批可配） | 微秒~ms（低负载最低） |
| Topic 规模 | 数万 Topic 平稳 | 数千分区后劣化 | 队列多也劣化 |
| 顺序消息 | 分区有序完善（锁队列） | 分区内有序 | 单队列单消费者 |
| 事务消息 | 半消息+回查（业务事务） | 流内 EOS（跨分区原子写） | 有但弱 |
| 延迟消息 | 4.x 级别 / 5.x 任意 | 无原生（需自建） | 插件/TTL+死信 |
| 消息回溯 | 按时间/位点 | 按位点/时间，天然日志回放 | 弱（消费即删） |
| 过滤 | Tag / SQL92 Broker 端过滤 | 无（消费端自理） | routing key 灵活路由 |
| 高可用 | DLedger/Controller 自动切换 | ISR + KRaft | 镜像队列/Quorum 队列(Raft) |
| 典型场景 | 电商交易、业务解耦、事务最终一致 | 日志采集、埋点、流计算、CDC | 复杂路由、中小规模企业集成 |

**选型口诀**：大数据管道/流 → Kafka；业务消息（事务/延迟/多 Topic/过滤）→ RocketMQ；复杂路由小规模 → RabbitMQ；云上免运维 → Pulsar/SQS 等亦可评估（Pulsar 计算存储分离 + BookKeeper，是"下一代"对照系）。

---
# 第六部分：MQ 大厂面试题精讲

> 难度：★ 基础 / ★★ 进阶 / ★★★ 深入 / ★★★★ 专家级。题目为各大厂高频真题风格归纳（同一考点多家都问，标注最具代表性的出处风格）。每题按【考点 → 作答主干 → 深入原理 → 延伸追问】展开。

---

## Q1.（阿里/美团｜★）为什么要用消息队列？如何选型？引入后有什么问题？

**作答主干**：三大价值——异步（缩短主链路 RT）、解耦（发布订阅，新增下游零改动）、削峰（缓冲突发流量保护下游）。代价——最终一致性替代强一致、复杂度上升（丢/重/乱序/积压）、MQ 自身成为关键依赖。

**选型**：日志/埋点/流计算 → Kafka（吞吐与生态）；交易类业务消息（事务、延迟、海量 Topic、Tag 过滤）→ RocketMQ；复杂路由中小规模 → RabbitMQ。给出量化依据：Kafka 百万级吞吐但分区多会劣化；RocketMQ 十万级但数万 Topic 稳定。

**延伸追问**：
- "不用 MQ 怎么削峰？"→ 本地队列+线程池（进程内，重启丢、无法跨服务）、Redis List/Stream（轻量但可靠性/堆积能力弱）、限流直接拒绝（体验差）。MQ 的本质是**持久化的、可回放的、分布式缓冲区**。
- "MQ 挂了怎么办？"→ 生产端本地降级表（消息先落库，恢复后补偿投递）+ 发送失败重试 + 集群高可用，核心链路要有"无 MQ 兜底路径"。

---

## Q2.（字节/阿里/京东/美团｜★★，几乎必考）如何保证消息不丢失？

**作答框架：三段式，缺一不可。**

**① 生产端 → Broker（发送不丢）**
- RocketMQ：同步发送 + 检查 SendResult=SEND_OK + 失败重试（默认 2 次，自动换 Broker）；更强用事务消息或"本地消息表"。
- Kafka：`acks=all` + `retries=Integer.MAX_VALUE` + `delivery.timeout.ms` 控制总时限 + 回调确认；禁用"fire and forget"。
- 通用兜底：**本地消息表/Outbox 模式**——业务数据与"待发消息"在同一个本地事务落库，后台任务扫描投递，投递成功标记删除。把"发消息"变成"可重试的持久化任务"，这是不依赖 MQ 特性的终极方案。

**② Broker 存储不丢**
- RocketMQ：同步刷盘（SYNC_FLUSH，防单机断电）+ 同步复制（SYNC_MASTER 或 DLedger 多数派，防单机损毁）。**面试要能说清：异步刷盘时进程崩溃不丢（数据在 PageCache/内核），机器断电才丢**——这是第二部分 2.2 的直接应用。
- Kafka：`replication.factor=3` + `min.insync.replicas=2` + `acks=all` + `unclean.leader.election.enable=false`。Kafka 不靠 fsync 靠副本：三台机器同时断电概率可忽略。

**③ Broker → 消费端（消费不丢）**
- 铁律：**先处理业务，再提交位点/返回 CONSUME_SUCCESS**。
- RocketMQ：并发消费返回 RECONSUME_LATER 走重试；Kafka：关闭自动提交，处理完手动 commitSync。
- 反例：Kafka 自动提交 poll 后立刻提交、业务异步丢线程池就返回成功 → 崩溃丢消息。

**深入原理**：每一段的丢失本质都是"确认时机早于持久化/处理完成"。把确认动作后置必然引入重复（确认丢失→重发），所以**不丢与不重是同一枚硬币的两面**，工程解 = At Least Once + 幂等。

**延伸追问**："acks=all 就一定不丢吗？"→ 不一定：若 ISR 收缩到只剩 Leader（min.insync.replicas=1 时），all 退化为 1；Leader 随后挂且 OSR 当选（unclean=true）仍丢。所以三个参数必须配套。

---

## Q3.（字节/美团/拼多多｜★★）如何保证消息不重复消费（幂等）？

**为什么必然重复**：生产重试（Broker 已落盘但 ack 丢失）、消费位点提交滞后（RocketMQ 提交 ProcessQueue 最小 offset；Kafka 处理完未 commit 崩溃）、Rebalance 队列易主。

**幂等方案矩阵（按强度排序）**：
1. **数据库唯一约束**：业务单号做唯一键，插入冲突即忽略——最简单最强一致，适合"创建型"操作。
2. **去重表 + 本地事务**：`msgId/业务键` 插入去重表与业务操作同一事务；插入失败=已处理。注意用**业务键**而非 MQ 的 msgId（生产重试可能生成新 msgId，RocketMQ 重试消息 offsetMsgId 会变）。
3. **状态机幂等**：`UPDATE order SET status='PAID' WHERE id=? AND status='UNPAID'`，affected rows=0 即已处理——适合状态流转，天然防并发。
4. **Redis SETNX + 过期**：性能高，但"Redis 标记成功、DB 失败"或过期后重放会破防 → 只作前置快速拦截，最终一致靠 DB 层兜底。
5. **乐观锁版本号**：`WHERE version=期望值`，适合更新型。

**深入追问**："去重表无限膨胀？"→ 按业务窗口 TTL 清理（消息重试有最大时限，超过窗口的重复不可能出现）；分库分表跟随业务键路由。
"并发两条相同消息同时到？"→ 唯一键/状态机由 DB 行锁保证原子；Redis 方案需 SETNX 原子性 + 处理失败删 key 的补偿。
"Kafka 幂等 Producer 能替代消费幂等吗？"→ 不能，它只解决"生产者重试导致的单分区单会话重复"（4.7.2），消费端重复它管不了。

---

## Q4.（阿里/京东｜★★）如何保证消息顺序？

**拆概念**：全局有序（代价：单队列单线程，吞吐崩塌）vs **分区有序**（同一业务键有序，工程主流）。

**三关都要过**：
1. **发送有序**：同 key 路由到同一队列（RocketMQ MessageQueueSelector / Kafka key hash）+ 该队列**同步串行发送**。Kafka 若开重试且 max.in.flight>1 又没开幂等 → 批 1 失败重发排到批 2 后面，乱序；开幂等后 Broker 按 sequence 拒乱序，in.flight≤5 仍保序。
2. **存储有序**：队列/分区内天然 append 有序（两者都满足）。
3. **消费有序**：RocketMQ MessageListenerOrderly（Broker 队列锁 + 本地锁 + 单线程）；Kafka 分区绑定单消费者，**若消费者内部再丢线程池并发处理就破坏顺序**——要么单线程，要么按 key 再做内存队列二次分片（每片单线程）。

**深入坑点（面试官最爱）**：
- Kafka **扩分区**后 hash(key)%N 变化 → 同 key 新旧消息进不同分区，顺序断裂。方案：预留足够分区、或迁移期双写校验、或自定义一致性哈希分区器。
- RocketMQ 顺序消费失败会**阻塞整个队列**（不能像并发那样丢重试队列），要设最大重试+告警+人工介入，否则一条毒消息卡死全队列。
- Broker 宕机/Rebalance 瞬间的队列易主有短暂乱序窗口，强顺序业务需在消费端用版本号/状态机拒绝过期消息（"业务层保序"兜底）。

---

## Q5.（阿里｜★★★）RocketMQ 事务消息的实现原理？

**答**：见 3.5 全链路图。作答要点按序：半消息（改写 Topic 为 RMQ_SYS_TRANS_HALF_TOPIC，消费者不可见）→ 执行本地事务 → Commit（还原真实 Topic 重新落 CommitLog）/Rollback（写 OP 标记）→ 回查兜底（每 60s 扫描无 OP 对应的半消息，回调 Producer Group 任一实例查本地事务状态，默认 15 次上限后回滚）。

**深入原理三连**：
1. "为什么半消息不可见？"——CommitLog 里其实已写入，但 ConsumeQueue 不构建原 Topic 的索引；**可见性由索引决定而非数据本身**，这是对存储引擎的极致复用。
2. "OP 消息干什么？"——RMQ_SYS_TRANS_OP_HALF_TOPIC 记录半消息的处理结果标记，回查服务用 half 与 op 做"对账"找出悬而未决的半消息。本质是**两个内部 Topic 实现的轻量 2PC + 事务状态日志**。
3. "回查为什么发给整个 Group？"——原 Producer 可能宕机；本地事务状态必须落库可查（订单状态字段），任何同组实例都能代答。

**延伸**：与 Seata AT（全局锁+反向补偿 SQL，强一致偏 CP）、TCC（业务层三段接口）对比：事务消息是**可靠消息最终一致**，只保上游原子性，下游靠重试+幂等；适合"下单→扣库存"这类可接受秒级不一致的场景，不适合"必须同步知道结果"的场景。

---

## Q6.（美团/滴滴｜★★★）RocketMQ 延迟消息原理？为什么 4.x 只支持固定级别？5.x 怎么做到任意时间？

**4.x**：改写 Topic 为 SCHEDULE_TOPIC_XXXX、queueId=level-1 → **同级别延迟相同 → 队列内写入序=到期序** → 每级别一个定时任务只看队首，到期还原真实 Topic 重新投递。O(1) 调度、零排序，这就是只支持 18 级的原因：**任意延迟需要全局按到期时间排序，海量消息下代价不可接受**。

**5.x**：**时间轮（TimerWheel）+ 持久化 TimerLog**。秒级槽位数组，消息挂到"到期秒"槽的链表，指针每秒推进处理当前槽；超一圈记 round；槽内条目持久化在 TimerLog（顺序写日志），重启回放恢复，内存只存指针。默认支持最长若干天（timerMaxDelaySec 可配）的任意秒级定时。

**延伸（展示知识广度）**：
- 时间轮同族：Netty HashedWheelTimer（单层+round）、Kafka 分层时间轮（秒→分→时逐级降级，避免 round 扫描）、Linux 内核定时器。对比 DelayQueue/Timer 的堆结构 O(logN)：**海量定时任务 O(1) 的通用解就是时间轮**。
- 没有延迟消息的 Kafka 怎么做？——按延迟档位建多个中转 topic + 消费端暂停/时间判断，或外部调度（DB 扫描/Redis zset score=到期时间 + 轮询），都要自己处理堆积与精度。

---

## Q7.（阿里/字节｜★★★★）为什么 RocketMQ 用一个 CommitLog，Kafka 用分区独立文件？各自的代价？

**作答逻辑链**（这题考架构权衡，背结论没用，要推演）：
1. 两者都追求顺序写。Kafka 每分区一个日志：**分区少**时每个文件都是顺序追加，性能极佳且**读取一跳直达**（配合 sendfile 完美）。
2. 但磁盘只有一颗磁头/一组 IO 队列：**分区成百上千**时，同一时刻向数百个文件追加，宏观上磁头/IO 调度在多个文件区间跳跃 → **顺序写退化为随机写**，吞吐悬崖式下跌（实测数千分区拐点）。
3. RocketMQ 反其道：**所有 Topic 混写一个 CommitLog**，无论多少 Topic/队列，物理上永远一条顺序写流 → 写入性能与 Topic 数解耦，支撑数万 Topic 的电商场景。
4. 代价转移到**读**：消费要先读 ConsumeQueue 索引再跳 CommitLog（两跳、消息体物理分散），于是用"ConsumeQueue 极小可全驻 PageCache + 追尾读热数据"来弥补；也因此难以像 Kafka 那样对消息体直接 sendfile（要按索引挑消息）。
5. 附加收益：一份数据多份索引的架构，加"延迟/事务/重试"只需改写索引路由（换 Topic 重投递），功能扩展极优雅——3.5/3.6/3.8 全是同一招。

**一句话**：Kafka 用"写读都直达"换少分区极限吞吐；RocketMQ 用"读两跳"换写入恒定顺序与海量 Topic 及丰富消息语义。

**延伸追问**："RocketMQ 读放大怎么优化？"→ PageCache 预热、mlock、冷读走 Slave、5.x RocksDB ConsumeQueue；"Kafka 分区多为什么还有元数据压力？"→ Controller 状态机/副本同步任务随分区线性涨，KRaft 就是为此而生。

---

## Q8.（阿里｜★★★★）讲清 RocketMQ 从消息写入到被消费的完整底层流程（含 mmap/刷盘细节）

按 3.2.3 链路完整口述，关键得分点：
1. Netty 收包 → SendMessageProcessor → **putMessageLock（自旋/重入锁）串行化追加** → 定位 1GB MappedFile（文件名=起始物理 offset，二分/内存表定位）→ 写 MappedByteBuffer。
2. **mmap 本质**：把文件 PageCache 映射进用户态地址空间，写入=写 PageCache，省一次内核→用户 CPU 拷贝；缺页中断按需建页表 → 所以有**文件预热**（逐页写 0 + mlock + madvise）消除首写毛刺。
3. 刷盘分叉：SYNC_FLUSH 走 GroupCommitService **组提交**（攒批 force，等待条件变量唤醒，默认 5s 超时）；ASYNC_FLUSH 每 500ms/4 脏页后台 force。
4. HA：SYNC_MASTER 等 Slave 上报 offset 达标（GroupTransferService 唤醒）。
5. **ReputMessageService 每 1ms 重放** CommitLog 新增数据 → 构建 ConsumeQueue（20B 定长）与 IndexFile（哈希槽+链表）→ 唤醒长轮询挂起请求。
6. 消费：客户端 PullRequest → Broker 按 queueOffset×20 读 ConsumeQueue → 拿物理 offset+size 去 CommitLog 读消息（可 Tag hashcode 初筛）→ 返回；客户端 ProcessQueue 缓存 + 流控三闸 + 线程池消费 + 提交最小 offset。

**延伸追问**：
- "MappedByteBuffer 的脏页何时真正落盘？"→ force()（msync），close/unmap 不保证；JVM 里 MappedByteBuffer 释放靠 Cleaner（反射 invokeCleaner），这也是 RocketMQ 持有引用计数防 unmap 后访问崩溃（JVM crash）的原因。
- "为什么用 1GB 文件？"→ 减少 mmap 次数与文件数；文件名即 offset 让定位 O(1)。
- "transientStorePool 开了之后哪里变了？"→ 写路径改堆外 buffer→commit→FileChannel，读写分离，见 3.2.4。

---

## Q9.（字节/快手｜★★★）Kafka 为什么快？

六大点（4.3）：分区顺序追加、PageCache（无 GC/重启不失效）、sendfile 零拷贝（transferTo，SG-DMA 下 0 次 CPU 拷贝）、批量+端到端压缩（Broker 不解压保重零拷贝）、稀疏索引（4KB 一项，mmap 常驻）、分区水平扩展。

**必须能画出零拷贝对比图**（2.3）：传统 4 拷贝 4 切换 → sendfile 2 DMA 拷贝 2 切换。并说明 **Kafka 用 sendfile（透传）而 RocketMQ 用 mmap（要解析内容）** 的原因差异。

**延伸追问**：
- "零拷贝在什么情况下失效？"→ Broker 需要触碰数据时：消息格式向下转换（老客户端）、Broker 端解压/重压缩、SSL/TLS 加密（数据必须进用户态加密）→ Kafka 开 TLS 后吞吐明显下降的原因。
- "PageCache 会不会被冷读打爆？"→ 大量回溯消费污染缓存，隔离手段：独立回溯集群/follower fetch/限流。

---

## Q10.（字节｜★★★★）讲清 ISR、HW、LEO，以及为什么需要 Leader Epoch？

按 4.4 展开：
1. LEO=各副本日志末端；HW=min(ISR LEO)；**消费者只能读 HW 之前的数据**——保证读到的必然已被 ISR 全体复制，换主不"幻读"。
2. ISR 进出按 `replica.lag.time.max.ms`（时间维度，30s 内追上过就留）。
3. **HW 更新滞后一轮 fetch**（Follower 先拿数据、下一轮 fetch 才带上新 LEO，Leader 才推进 HW）→ 两个事故：副本重启按 HW 截断丢已确认数据；双主重启后同 offset 内容分叉（把 4.4.3 两个事故时序讲出来是满分标志）。
4. **Leader Epoch**：每次选主 epoch+1，记录 <epoch, endOffset>；恢复时向 Leader 发 OffsetsForLeaderEpoch 拿精确截断点，替代"按 HW 盲截"。**HW 管读边界，Epoch 管恢复截断**。

**延伸追问**：
- "min.insync.replicas=2 但 ISR 只剩 1 会怎样？"→ 拒写 NotEnoughReplicas，可用性换一致性（CAP 取舍题）。
- "Kafka 用的是 Paxos/Raft 吗？"→ 数据复制不是共识协议，是**主从+ISR 的类 PacificA** 模型；只有 KRaft 元数据层用 Raft。能分清"数据复制协议"与"选主共识协议"是深水区标志。

---

## Q11.（美团/快手｜★★★）Kafka Rebalance 流程、触发条件与生产事故规避？

流程（4.8.2）：JoinGroup（选 Consumer Leader）→ Leader 计算分配 → SyncGroup 下发 → Heartbeat 稳态；generation.id 递增隔离旧世代提交。

触发：成员增减、订阅/分区数变化、session.timeout 心跳超时、**max.poll.interval.ms 处理超时**（最常见生产事故：单批处理超 5min → 被踢 → rebalance → 该批白干 → 下一批又超时 → **rebalance 风暴死循环**）。

规避清单：
1. 减小 max.poll.records / 优化单条耗时 / 处理异步化+暂停分区（pause/resume）；
2. 升级 **CooperativeStickyAssignor**（增量协作式，只迁移必要分区，告别 Stop-The-World）；
3. **静态成员 group.instance.id**：滚动重启不触发 rebalance；
4. 合理 session.timeout（新版默认 45s）与 heartbeat.interval（3s）配比（心跳间隔 ≤ 1/3 超时）。

**延伸**：对比 RocketMQ 客户端自主 Rebalance（每 20s 各自算，无协调者，简单但短暂重复/抖动）与 Kafka 协调者协议（一致性强但组大时协议开销大）；再对比 RocketMQ 5.x Pop 消费直接绕开 rebalance——三代方案演进观是加分项。

---

## Q12.（所有大厂｜★★，场景必考）线上百万消息积压怎么处理？

**标准处置 SOP**：
1. **定位原因**：消费端故障（DB 慢/下游超时/线程池满/毒消息反复重试）还是流量突增？先看消费 TPS 与单条耗时曲线。
2. **修复消费端**：先解决慢的根因（下游扩容、批量写、异步化），否则扩容也白扩。
3. **扩容有上限意识**：消费者实例 ≤ 队列/分区数，超了空转。**RocketMQ/Kafka 队列数不够时的经典解法**：上线一个"搬运消费者"，只做转发——把积压消息快速搬到一个**分区数×10 的新 Topic**，再用 10 倍消费者消费新 Topic（搬运逻辑极轻，单机可达数万 TPS）。
4. **降级取舍**：非核心消息（通知/统计）可跳过或抽样消费（重置位点到最新+离线补数）；核心交易消息不可丢，只能扩。
5. **事后**：压测消费容量、积压告警（lag 阈值）、消费端限流保护下游、评估分区数预留。

**延伸追问**："能不能跳过积压直接消费最新？"→ 可以（reset offset），但必须回答"跳过的数据去哪了"——落库存档离线回放，否则就是丢消息事故。"RocketMQ 重试队列会不会造成积压放大？"→ 毒消息 16 次重试×延迟级别，监控 %RETRY% Topic lag，及时进死信人工处理。

---

## Q13.（字节｜★★★★）Kafka Exactly-Once 是怎么实现的？边界在哪？

三层递进（4.7）：
1. **幂等 Producer**：PID+epoch+sequence，Broker 按分区校验 seq 连续性去重 → 解决单分区单会话重试重复；epoch 隔离僵尸实例。
2. **事务 Producer**：transactional.id 绑定稳定身份 → TransactionCoordinator + __transaction_state 记录事务状态 → 跨分区写 + COMMIT/ABORT marker 原子可见；read_committed 消费者受 **LSO** 限制并过滤 abort 消息。
3. **EOS 闭环**：consume→process→produce 中把**消费位移也提交进事务**（sendOffsetsToTransaction）→ "处理结果+位移"原子提交，崩溃恢复后要么全生效要么全回滚（Kafka Streams exactly_once_v2）。

**边界（必答）**：仅 Kafka→Kafka 闭环成立；写外部 DB/调 HTTP 后事务失效 → 仍需消费幂等或 Outbox。**"Exactly-Once Delivery 不存在，存在的是 Exactly-Once Processing（状态效果恰好一次）"**——说出这句话基本锁定高分。

**延伸**：两阶段提交里 Coordinator 挂了怎么办？__transaction_state 多副本+事务超时（transaction.timeout.ms）自动 abort；LSO 卡住（长事务阻塞 read_committed 消费）是经典生产问题。

---

## Q14.（阿里｜★★★）RocketMQ 主从复制与自动主从切换（DLedger/Raft）原理？

三代演进（3.3）：HAService 主从（Slave 上报 offset 拉增量，SYNC_MASTER 等 ack，但**不能自动升主**）→ DLedger（Raft 接管 CommitLog，3 节点多数派写，自动选举）→ 5.x Controller（SyncStateSet + 主从复制不变，选主交给 Raft 控制器）。

**Raft 三要素必须能讲**：任期+随机超时选举（防瓜分选票）、多数派日志复制后 commit、**只有日志最新的节点能当选**（保证已提交数据不丢）。

**延伸追问**：
- "为什么 DLedger 要 3 节点而主从 2 节点就行？"→ 多数派需要 2/3；2 节点无法区分"对方挂了"和"网络分区"，会脑裂。
- "SYNC_MASTER 的 ack 是落盘吗？"→ 不是，Slave 写入（PageCache）即上报，双机同时断电仍可能丢——引出"同步刷盘+同步复制"组合与成本权衡。
- "Raft 与 ZAB/Paxos 区别？"→ Raft 强 Leader、日志连续不许空洞、工程可理解性优先；ZAB 类似但为 ZK 定制；Multi-Paxos 允许乱序确认更复杂。

---

## Q15.（阿里/腾讯｜★★★）RocketMQ 长轮询的实现？和 Push/Pull 什么关系？

答 1.5 + 3.4.3：PushConsumer 本质 Pull；无消息时 Broker hold 请求（PullRequestHoldService 按 topic@queueId 挂起），**ReputMessageService 构建 ConsumeQueue 的同时回调 NotifyMessageArrivingListener 精准唤醒**（消息到达→毫秒级返回），另有 5s 兜底轮询与 15s 挂起超时。

**深入点**：为什么不用 Netty 直接推？——Broker 无消费者处理能力信息，推模式缺背压；长轮询把"何时拉、拉多少"的主动权留在消费者（ProcessQueue 流控三闸），Broker 只负责"有货立刻应答"。**实时性≈Push，流控=Pull**。

**延伸**：Kafka fetch.min.bytes+fetch.max.wait.ms 是同一思想的"攒批版"；对比真正 Push 的 RabbitMQ（basic.qos prefetch 做背压）；再延伸 HTTP 长轮询/Comet→SSE→WebSocket 的实时通信演进。

---

## Q16.（拼多多/字节 架构题｜★★★★）让你设计一个分布式 MQ，你怎么设计？

**答题框架（展示架构推演能力，不是背 RocketMQ）**：
1. **明确目标**：吞吐/延迟/可靠性等级/消息语义（顺序?事务?延迟?）/规模（Topic 数、消息量）——先问清场景再动手，架构题最忌直接堆组件。
2. **存储**：Append-Only Log + PageCache + 顺序写；按 Q7 权衡选"分区文件"（少 Topic 大吞吐）或"CommitLog+索引"（多 Topic 业务）；稀疏/定长索引；segment 滚动+过期删除。
3. **网络**：Netty Reactor + 长度字段协议 + requestId 多路复用；消费用长轮询。
4. **高可用**：副本主从 + ISR 式同步集合 + 独立 Raft 控制器选主（RocketMQ 5.x/KRaft 同款思路：复制与选主解耦）；acks 可配 + min.insync 兜底。
5. **语义层**：At Least Once + 消费幂等指引；顺序=键路由+队列锁；事务=半消息+回查；延迟=时间轮+持久化日志。
6. **扩展性**：计算存储分离（Proxy 无状态，参考 5.x/Pulsar）、Pop 式无状态消费避免 rebalance、分区可迁移。
7. **运维面**：lag 监控、冷读隔离、磁盘水位保护、消息轨迹与回溯。

**评分关键**：每个决策都要带"为什么、代价是什么"，并能对照现存系统说出"我抄了谁、改了谁、为什么"。

---
# 附录：知识延伸学习地图

```
本文知识点 ──延伸──> 更深的底层与相邻领域
├─ Append-Only Log ──> WAL / LSM-Tree(RocksDB, HBase) / 事件溯源(Event Sourcing, KRaft元数据)
├─ PageCache/刷盘 ──> MySQL redo log组提交 / mmap vs DirectIO / io_uring
├─ 零拷贝 ──────────> DMA/SG-DMA / splice / Kafka TLS为何破坏零拷贝
├─ epoll/Reactor ───> Redis单线程Reactor / Nginx多进程 / Netty内存池jemalloc
├─ ISR/多数派 ──────> Raft(选举/复制/安全性) / ZAB / PacificA / Quorum NWR(Dynamo)
├─ 事务消息 ────────> 2PC/3PC / TCC / Saga / Seata AT / 本地消息表(Outbox) / CDC(Debezium)
├─ 时间轮 ──────────> Netty HashedWheelTimer / Kafka分层时间轮 / 内核定时器
├─ 幂等/去重 ───────> 分布式锁(Redis/ZK) / 唯一键+状态机 / 布隆过滤器
└─ 稳定性 ──────────> 熔断(Sentinel/Resilience4j) / 舱壁隔离 / 重试预算 / 亚稳态故障
```

**复习建议（由浅入深三轮法）**：
1. 第一轮：只看每章"是什么/为什么"与对比表，建立全景（1~2 天）；
2. 第二轮：对着面试题口述作答，卡壳处回到对应原理章节精读（3~5 天）；
3. 第三轮：★★★★ 题源码验证——RocketMQ 看 CommitLog/MappedFile/ReputMessageService/TransactionalMessageService，Kafka 看 LogSegment/ReplicaManager/ProducerIdempotence。

> 说明：文中面试题按各大厂公开面经的高频考点风格归纳整理，非某场面试原题；参数默认值以 RocketMQ 4.9/5.x、Kafka 3.x 为准，不同版本可能有差异，落地前请以官方文档核对。