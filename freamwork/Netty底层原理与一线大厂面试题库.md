# Netty 底层原理全景 · 从内核 syscall 到 ChannelPipeline

> **适用版本约定**：Netty 4.1.x（主流生产版本，4.2 为 API 不兼容的迭代版）、JDK 8 / 11 / 17 / 21、Linux 4.x+（epoll）。
> **难度标记**：★ 基础认知 · ★★ 理解机制 · ★★★ 源码级 · ★★★★ 需要跨层（用户态+内核态）联合推理
> **文档定位**：不是"API 使用手册"，而是一条**自下而上的因果链**——内核怎么收包 → JDK 怎么暴露 → Netty 怎么封装 → 代码为什么这么写 → 面试为什么这么问。

---

## 目录

- [第一部分　全局定位：Netty 到底站在哪一层](#第一部分全局定位netty-到底站在哪一层)
- [第二部分　操作系统 I/O 底层：一次 read 到底发生了什么](#第二部分操作系统-io-底层一次-read-到底发生了什么)
- [第三部分　epoll 深挖与 JDK NIO 的实现真相](#第三部分epoll-深挖与-jdk-nio-的实现真相)
- [第四部分　零拷贝：两个完全不同世界的"零拷贝"](#第四部分零拷贝两个完全不同世界的零拷贝)
- [第五部分　TCP 底层：粘包、Nagle、TIME_WAIT 的物理根源](#第五部分tcp-底层粘包nagletime_wait-的物理根源)
- [第六部分　Reactor 模式与 Netty 线程模型](#第六部分reactor-模式与-netty-线程模型)
- [第七部分　EventLoop 源码级剖析](#第七部分eventloop-源码级剖析)
- [第八部分　Channel / Pipeline / Handler 事件传播机制](#第八部分channel--pipeline--handler-事件传播机制)
- [第九部分　ByteBuf 与内存池：Netty 最强的一块](#第九部分bytebuf-与内存池netty-最强的一块)
- [第十部分　服务端启动与一次连接的完整生命周期](#第十部分服务端启动与一次连接的完整生命周期)
- [第十一部分　写路径、水位线、编解码、心跳与优雅关闭](#第十一部分写路径水位线编解码心跳与优雅关闭)
- [第十二部分　性能调优、故障排查与常见认知误区](#第十二部分性能调优故障排查与常见认知误区)
- [第十三部分　一线大厂面试题库（含难度标注 + 深度解答 + 追问链）](#第十三部分一线大厂面试题库含难度标注--深度解答--追问链)
- [第十四部分　自测清单与学习路径](#第十四部分自测清单与学习路径)

---

# 第一部分　全局定位：Netty 到底站在哪一层

## 1.1 一句话定位

**Netty 是一个"把内核 I/O 多路复用能力，翻译成可组合的事件驱动编程模型"的中间件。**
它不是协议库、不是 RPC 框架，它是**网络编程的运行时（runtime）**。

```
┌──────────────────────────────────────────────────────────────┐
│ 业务层      Dubbo / gRPC / RocketMQ / Spring WebFlux / ES     │
├──────────────────────────────────────────────────────────────┤
│ 框架层      Netty  ← 编解码、连接管理、线程模型、内存管理      │
├──────────────────────────────────────────────────────────────┤
│ 抽象层      JDK NIO (Selector / Channel / ByteBuffer)         │
├──────────────────────────────────────────────────────────────┤
│ 系统调用    epoll_create / epoll_ctl / epoll_wait / read / write│
├──────────────────────────────────────────────────────────────┤
│ 内核层      VFS → Socket 层 → TCP/IP 协议栈 → 网卡驱动 → DMA  │
└──────────────────────────────────────────────────────────────┘
```

理解这张图的关键：**Netty 的几乎每一个"设计怪癖"，都能在下三层找到一个不得不这么做的原因。**

| Netty 的设计 | 往下追一层的原因 |
|---|---|
| EventLoop 单线程绑多个 Channel | epoll 是"一个线程轮询一批 fd"的模型，多线程轮询同一批 fd 只会增加锁竞争 |
| 所有 handler 在一个线程串行执行 | 免锁化：单线程串行 = 天然的 happens-before，业务代码可用普通变量 |
| ByteBuf 池化 + 引用计数 | 每次 read 都要一块缓冲区，若走 GC 分配，YGC 频率会被网络 IO 放大 |
| 必须处理半包/粘包 | TCP 是**字节流**协议，内核不保留"消息边界"，边界只能由应用层重建 |
| write 而不立即 flush | 每次 write 都是一次 syscall + 一次网卡 DMA，合并可把 N 次系统调用压成 1 次 |
| 空轮询检测 + 重建 Selector | JDK epoll 实现存在 select 返回 0 但 fd 就绪的 bug（JDK-6670302） |

## 1.2 JDK NIO 的五个原生缺陷（Netty 的存在理由）

> ★★ 这是面试第一道必考题"为什么不用原生 NIO"，但**答 API 复杂只是及格线**，真正的答案在后面 4 条。

1. **API 反人类**：`Selector` + `SelectionKey` + `ByteBuffer` 三者状态耦合，`interestOps` 修改、`key.remove()`、`selector.wakeup()` 的时序极易出错；`ByteBuffer` 只有 position/limit/capacity，读写共用一套指针，flip 忘记就是经典 bug。
2. **epoll 空轮询 Bug**：Linux 下 `selector.select()` 可能在无事件时立即返回 0，导致 `while(true)` 变成 100% CPU 死循环。Netty 用"计数 512 次重建 Selector"硬扛。
3. **ByteBuffer 能力缺失**：无池化、无引用计数、无读写指针分离、无复合缓冲区，堆外内存只能靠 `allocateDirect`（走 Cleaner，释放依赖 GC）。
4. **无连接生命周期管理**：没有心跳、没有断线重连、没有空闲检测、没有半包处理，全靠自己写状态机。
5. **无编码/解码抽象**：协议解析（长度字段、分隔符、定长）要自己拼，而这块恰是网络编程 80% 的 bug 来源。

**面试加分说法**：Netty 的核心价值不是"封装 NIO"，而是**把"不可靠的字节流"变成"可靠的消息对象流"**，并在此过程中把内存、线程、系统调用三者的成本做到接近理论下限。

---

# 第二部分　操作系统 I/O 底层：一次 read 到底发生了什么

> ★★★ 这一节是全部 Netty 知识的物理地基。跳过它，后面所有"为什么"都只能背结论。

## 2.1 用户态 / 内核态：为什么要切换

CPU 有特权级（Ring 0~3）。应用程序运行在 Ring 3（用户态），**不能直接访问硬件、不能直接改页表、不能操作网卡寄存器**。只有陷入内核（Ring 0）才能做这些事。

进入内核的唯一合法方式：**系统调用（syscall）**。过程：

```
用户态执行 read(fd, buf, len)
  ↓ 把 syscall 号放进 rax，参数放进 rdi/rsi/rdx
  ↓ 执行 syscall 指令（或 int 0x80）
CPU 保存用户态现场（SS/RSP/RFLAGS/CS/RIP 压栈到内核栈）
  ↓ 查 sys_call_table 跳转到 sys_read
内核态执行：校验 fd → 找到 file 结构 → 调 socket 层的 recv → 协议栈 → 若数据未到则睡眠
  ↓ 完成后 return
CPU 恢复用户态现场（sysret / iret）
用户态继续执行
```

**一次 syscall 的代价**（这是性能分析的基本单位，务必记住量级）：

| 成本项 | 典型量级 | 说明 |
|---|---|---|
| 模式切换（用户↔内核） | 约 100~1000ns | 需要保存/恢复寄存器、切换栈、TLB 部分失效 |
| 上下文切换（线程 A → 线程 B） | 约 2~10μs | 涉及调度器、页表基址切换、cache 冷启动 |
| 一次内存拷贝（1KB） | 约几十 ns ~ 数百 ns | 受 cache line、NUMA 影响 |
| 一次网络往返（同机房） | 约 100~500μs | 物理距离 + 协议栈处理 |

> **关键推论**：网络 IO 的性能优化，本质是**减少 syscall 次数**和**减少数据拷贝次数**。Netty 的"批量 flush"、"池化 buffer"、"零拷贝"全部服务于这两条。

## 2.2 五种 I/O 模型与"到底谁在阻塞"

| 模型 | 第一阶段（等数据到内核） | 第二阶段（内核→用户拷贝） | syscall 次数 | 线程占用 |
|---|---|---|---|---|
| 阻塞 I/O | 阻塞 | 阻塞 | 1 | 1 连接 1 线程，线程全睡 |
| 非阻塞 I/O | 轮询（返回 EAGAIN） | 阻塞 | N（轮询） | 忙等，CPU 空转 |
| **I/O 多路复用** | **select/epoll 阻塞** | 阻塞 | 3+N | **1 线程管 N 连接** |
| 信号驱动 I/O | 不阻塞（SIGIO 通知） | 阻塞 | 1 + 信号 | 信号处理难 |
| 异步 I/O（AIO） | 不阻塞 | **不阻塞**（内核完成后通知） | 1 | 真正的异步 |

**必须讲清的一个事实**：Linux 的 epoll 属于"**多路复用**"，不是"异步 I/O"。`epoll_wait` 只告诉你"**哪个 fd 可读了**"，真正的 `read` 仍然由你在用户态发起，而且**仍然要阻塞拷贝数据**。所以 Netty 的"异步"是**编程模型上的异步（回调/Future）**，不是内核层面的异步——这是面试高频陷阱题。

> 真正的内核异步是 io_uring（Linux 5.1+），Netty 生态里目前主要靠 `io_uring` 第三方实现（如 netty-incubator-transport-io_uring），主流仍是 epoll。

## 2.3 一次 TCP 收包的完整链路（自上而下 + 自下而上）

**网卡收到包（下半部）**：

```
① 网卡收到电信号 → DMA 把数据帧写入内核环形缓冲区（Ring Buffer，避免 CPU 搬运）
② 网卡发起硬中断 → 内核执行 ISR，只做最少的事：禁用中断、记录、触发软中断（NET_RX_SOFTIRQ）
③ 软中断（ksoftirqd 线程）执行 net_rx_action：
    - 从 ring buffer 取 skb（sk_buff）
    - 逐层剥头：以太网头 → IP 头 → TCP 头
    - 校验和、序列号检查、乱序重组（out-of-order queue）
    - 按四元组查找 socket（epoll 场景下用 hash 表 O(1)）
④ 数据放入 socket 的接收缓冲区（sk_receive_queue）
⑤ 关键一步：调用 sk->sk_data_ready() → sock_def_readable()
    → 唤醒等待队列上的进程（就是 epoll 的 ep_poll_callback）
```

**应用层拿到数据（上半部）**：

```
⑥ epoll_wait 被唤醒返回 → 就绪 fd 列表
⑦ 应用调用 read(fd, buf, n)：
    - 从 sk_receive_queue 取出 skb
    - 把数据从内核缓冲区拷贝到用户缓冲区（这一步是真拷贝，CPU 参与）
    - 更新 sk_rmem_alloc，若低于水位线则发 TCP window update
⑧ 返回读取字节数
```

从这里可以直接推出几条 Netty 设计：

- **为什么 epoll 比 select 快？** 因为 ③ 中的"查找就绪 fd"是**内核主动回调**（`ep_poll_callback`）完成的，而不是用户态每次遍历全部 fd。下面细讲。
- **为什么收到数据后必须"尽快读空"（ET 模式）？** 因为 `sk_data_ready` 只在"空→非空"的**状态跃迁**时触发一次，若没读干净，下一次数据到达时状态已是"非空"，不会再次唤醒。
- **为什么 SO_RCVBUF 太小会导致丢包？** 接收缓冲区满时，TCP 通告窗口为 0，对端停发；若对端仍发（如窗口探测后），内核直接丢弃，表现为"能连上但吞吐上不去/偶发超时"。

## 2.4 五个缓冲区：网络调优的所有秘密都在这里

```
发送端                                              接收端
┌─────────────┐   ┌──────────┐     网络     ┌──────────┐   ┌─────────────┐
│ 应用发送缓冲 │→ │ 内核发送  │ → ────────→ │ 内核接收  │→ │ 应用接收缓冲 │
│ (Netty Out- │   │ 缓冲区    │             │ 缓冲区    │   │ (ByteBuf)   │
│  boundBuffer)│   │SO_SNDBUF │             │SO_RCVBUF  │   │             │
└─────────────┘   └──────────┘             └──────────┘   └─────────────┘
```

| 缓冲区 | 归属 | 满/空的后果 | 调优参数 |
|---|---|---|---|
| Netty `ChannelOutboundBuffer` | 用户态堆外 | 触发 `isWritable()=false`，业务应做背压 | 高低水位线 |
| 内核发送缓冲区 | 内核 | `write` 返回 EAGAIN/阻塞（非阻塞下返回 EAGAIN） | `SO_SNDBUF` |
| 内核接收缓冲区 | 内核 | 通告窗口=0，对端停发 | `SO_RCVBUF` |
| Netty `ByteBuf`（cumulation） | 用户态 | 半包数据堆积，可能 OOM | `maxCumulationBufferComponents` |
| 网卡 Ring Buffer | 硬件/驱动 | 物理丢包（drop），`ifconfig` 的 RX dropped 增长 | `ethtool -G` |

**面试延伸点**：`SO_RCVBUF` / `SO_SNDBUF` 在 Linux 上设置后会被内核**翻倍**（因为内核要额外记账开销），且受 `net.core.rmem_max` 限制；Java 里 `setReceiveBufferSize` 只是 `setsockopt` 的封装，设得比 `rmem_max` 大无效。

---

# 第三部分　epoll 深挖与 JDK NIO 的实现真相

> ★★★★ 这一节是区分"会用 Netty"和"懂 Netty"的分水岭。面试里能讲清 epoll 数据结构 + JDK 映射关系的，基本可以直接定性为高级。

## 3.1 select / poll 的原罪

**select**：
```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
三个致命问题：
1. `fd_set` 是位图，**上限 1024**（`FD_SETSIZE`）。
2. 每次调用都要把 fd 集合**从用户态全量拷贝到内核态**（O(n) 拷贝）。
3. 返回后内核只告诉你"有几个就绪"，**不知道是哪几个**，用户态必须**再遍历一遍全部 fd** 做 `FD_ISSET`（O(n) 扫描）。即使只有 1 个连接活跃，也要扫 10000 次。

**poll**：用 `struct pollfd` 数组解决了 1024 上限，但**拷贝 + 全量扫描**两个问题没解决。

> 所以 select/poll 的复杂度是 **O(n)**，n = 关注的 fd 总数，而不是活跃 fd 数。这就是"**C10K 问题**"的技术本质——10000 个连接时，每轮事件处理要扫 10000 次，CPU 全花在扫描上。

## 3.2 epoll 的三个系统调用与内核数据结构

```c
int epoll_create1(int flags);                                  // 创建一个 epoll 实例
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev); // ADD / MOD / DEL
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

内核里的核心结构（`fs/eventpoll.c`）：

```c
struct eventpoll {
    spinlock_t        lock;
    struct rb_root_cached rbr;      // 红黑树：所有被监听的 fd（epitem）
    struct list_head  rdllist;      // 双向链表：已经就绪的 fd（"就绪链表"）
    struct wait_queue_head wq;      // epoll_wait 时睡眠在此，等待被唤醒
    struct file      *file;
    ...
};

struct epitem {                     // 红黑树节点 = 一个被监听的 fd
    struct rb_node    rbn;          // 挂到 rbr
    struct list_head  rdllink;      // 挂到 rdllist（就绪时）
    struct epoll_filefd ffd;        // {file, fd}
    struct eventpoll *ep;
    struct epoll_event event;       // 关注的事件（EPOLLIN/EPOLLOUT/EPOLLET...）
    struct list_head  pwqlist;      // 该 fd 上挂的等待队列项
};
```

**为什么用红黑树？** 增删改查都是 O(log n)，且**不需要连续内存**（对比 poll 的数组）。`epoll_ctl(ADD)` = 插入红黑树 + 在目标 socket 的等待队列上注册 `ep_poll_callback`。

**为什么用就绪链表？** 因为"就绪"本质是一个**事件流**，只需 O(1) 头插/取头；`epoll_wait` 只需遍历 rdllist 中**真正就绪**的项，复杂度 **O(k)**，k = 活跃 fd 数。这是 epoll 完胜 select/poll 的根本原因——**复杂度从"关注集大小"变成"活跃集大小"**。

## 3.3 回调机制：epoll 是"事件驱动"的内核实现

```
epoll_ctl(ADD, fd) 时：
    ep_insert() → 在 fd 对应的 socket 的 sk_wq 上注册一个 wait_queue_entry
                wait_queue_entry.func = ep_poll_callback

数据到来时（内核软中断里）：
    sk_data_ready() → wake_up(&sk->sk_wq)
        → 遍历该 socket 上所有 wait queue entry
        → 调用 ep_poll_callback()
            → 把该 fd 的 epitem 挂到 ep->rdllist（只挂一次，用 EPOLL_IS_LINKED 标记去重）
            → 唤醒 ep->wq 上睡眠的 epoll_wait 线程

epoll_wait 返回：
    把 rdllist 中的 epitem 拷贝到用户态 events 数组（只拷贝就绪的，所以拷贝量是 O(k)）
```

**这就是"回调代替轮询"**：内核在网络软中断的上下文里主动把 fd 标记为就绪，用户态只管取。**Netty 的 `EventLoop` 就是在这个机制上包了一层 for 循环**。

## 3.4 水平触发（LT） vs 边缘触发（ET）—— 最容易答错的一题

| 维度 | LT（Level Triggered，默认） | ET（Edge Triggered，`EPOLLET`） |
|---|---|---|
| 触发条件 | 只要缓冲区**还有数据**，每次 `epoll_wait` 都返回 | 只在**状态跃迁**（空→非空）时返回**一次** |
| 是否可能漏事件 | 不会（只要没读完就会再通知） | **会**（没读干净就再也不通知） |
| 读的要求 | 可以只读一部分 | **必须循环读到 `EAGAIN`** |
| 必须非阻塞 | 否 | **是**（否则最后一次 read 会永久阻塞） |
| 系统调用次数 | 多（重复唤醒） | 少（一次唤醒尽量读完） |
| 适用 | 简单、安全 | 高并发（Netty、Nginx、Redis 都用 ET） |

**ET 的一个真实坑**：如果一次事件里有 100KB 数据，但你只 read 了 10KB 就回去等下一次 `epoll_wait`，那么**剩下 90KB 永远不会触发新事件**，连接"看起来卡住"。这就是"ET 必须读到 EAGAIN"的物理原因——它源自 `sk_data_ready` 的**状态跃迁语义**。

**Netty 在 Linux 上默认用的是 LT（JDK NIO 只暴露 LT）**，`EpollEventLoop`（native transport）才可配 `EpollChannelOption.EPOLL_MODE` 选择 ET。但 Netty 通过"每次事件把 channel 的读循环跑到底（`read` 直到无数据）"来获得接近 ET 的效率。

**惊群（thundering herd）**：
- 多个进程/线程 `accept` 同一个 listen fd 时，一个连接到来会唤醒所有等待者，只有一个成功，其余 EAGAIN → 浪费 CPU。
- 解决：`EPOLLEXCLUSIVE`（4.5+，只唤醒一个）、`SO_REUSEPORT`（每进程独立 listen socket，内核做负载均衡）、或"主 Reactor 单线程 accept"（Netty boss 单线程天然规避）。

## 3.5 JDK 是怎么把 epoll 包起来的（源码映射）

Java NIO 的 `Selector` 在 Linux 上的实现链：

```
Selector.open()
  → SelectorProvider.provider()            // sun.nio.ch.DefaultSelectorProvider
  → EPollSelectorProvider.openSelector()
  → new EPollSelectorImpl(provider)
      ├─ fd1 = EPollArrayWrapper.epollCreate()      → epoll_create1(0)
      ├─ fd0 = eventfd(0, EFD_NONBLOCK)             → 用于 wakeup 的自唤醒 fd
      └─ pollWrapper = new EPollArrayWrapper()
```

`EPollArrayWrapper` 维护三样东西：

| 字段 | 作用 |
|---|---|
| `pollArray`（native 内存） | 传给 `epoll_wait` 的 `struct epoll_event` 数组 |
| `eventsLow[] / eventsHigh[]` | 记录每个 fd 的 `EPOLLIN` / `EPOLLOUT` 兴趣位（因为 epoll 事件是 int，JDK 用高低位分开存） |
| `updateDescriptors[] / updateEvents[]` | `epoll_ctl` 的**批量更新队列**，攒够一批（`MAX_UPDATE_ARRAY_SIZE=64`）一次性下发 |

关键 native 方法映射：

| Java 方法 | 系统调用 |
|---|---|
| `EPollArrayWrapper.epollCreate()` | `epoll_create1(0)` |
| `epollCtl(int opcode, int fd, int events)` | `epoll_ctl(epfd, op, fd, &ev)` |
| `epollWait(long pollAddress, int numfds, long timeout, int epfd)` | `epoll_wait(epfd, events, numfds, timeout)` |
| `interrupt()` / `wakeup()` | `write(eventfd, 1)` —— **往 fd0 写一个字节** |
| `EPollSelectorImpl.doSelect()` | `epollWait(...)` + 处理 fd0 |

**`Selector.wakeup()` 的本质**：不是信号，而是**往一个 eventfd 写 1 字节**，让 `epoll_wait` 因为 fd0 可读而立即返回。这就是为什么"另一个线程调用 `wakeup()` 能把阻塞在 select 的线程叫醒"——它借用了 epoll 自己的多路复用能力。

**为什么需要这个机制？** 因为 `epoll_wait` 是阻塞的，而"有新任务提交到 EventLoop"这件事，内核完全不知道，只能通过一个"假的 fd 就绪"去打断它。**这就是 Netty `NioEventLoop.wakeup()` 的底层。**

### 3.5.1 一个被忽略的优化：selectedKeys 的数组化

JDK 原生 `Selector.selectedKeys()` 返回 `HashSet<SelectionKey>`，每次 select 都要往 HashSet 里 add，有 hash 计算和扩容成本。Netty 通过反射把 `sun.nio.ch.SelectorImpl` 里的 `selectedKeys` / `publicSelectedKeys` 字段替换成自己实现的 `SelectedSelectionKeySet`（内部就是一个**数组 + size**，O(1) add，遍历时无迭代器开销）。

```java
// io.netty.channel.nio.SelectedSelectionKeySet
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {
    SelectionKey[] keys;
    int size;
    public boolean add(SelectionKey o) { keys[size++] = o; ... }  // 数组追加，无 hash
}
```

**代价**：JDK 9+ 模块化后反射 `sun.nio.ch` 需要 `--add-opens java.base/sun.nio.ch=ALL-UNNAMED`，否则失败并回退到原生 HashSet（可通过 `-Dio.netty.noKeySetOptimization=true` 显式关闭）。
**面试价值**：能说出"Netty 连 JDK 的 HashSet 都要替换成数组"这个细节，说明真的翻过源码。

## 3.6 epoll 空轮询 Bug 与 Netty 的 512 次重建

**现象**：`selector.select()` 本该阻塞，却不断立即返回 0，`while(true)` 循环变成 CPU 100%。
**根因**（JDK-6670302 / JDK-6403933）：Linux epoll 在某些边缘场景（如连接被 RST、`EPOLLHUP` 与 `EPOLLIN` 的处理时序）下，事件被"消费"掉了但 `epoll_wait` 依然返回 0；JDK 的 `EPollSelectorImpl` 没有做防御，直接返回。
**Netty 的修复**（`NioEventLoop.select()`）：

```java
// 简化版核心逻辑
int selectCnt = 0;
for (;;) {
    long timeoutMillis = (selectDeadLineNanos - currentTimeNanos) / 1000000 + 1;
    if (timeoutMillis <= 0) {
        if (selectCnt == 0) { selector.selectNow(); selectCnt = 1; }
        break;
    }
    if (hasTasks() && WakenUp.getAndSet(false)) {     // 有任务就先不阻塞
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    int selectedKeys = selector.select(timeoutMillis);  // ← 真正阻塞
    selectCnt++;
    if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
        break;                                          // 正常返回
    }
    if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {  // 默认 512
        // 重建 Selector：新建 selector，把旧的所有 key 重新注册过去
        rebuildSelector();
        selector = this.selector;
        selectCnt = 0;
        break;
    }
}
```

**注意三个细节（面试深挖点）**：
1. 判空轮询的条件是 `selectedKeys == 0 && 没被 wakeup && 没任务`——三者同时满足才认为是"空轮询"。
2. 阈值 512 可通过 `-Dio.netty.selectorAutoRebuildThreshold` 调整，设为 0 则关闭。
3. **重建 Selector 是有代价的**：要遍历旧 selector 的所有 key，重新 `register` 到新 selector，并迁移 attachment（`Channel` 引用）。这期间如果有事件，可能丢；所以 Netty 在重建后立即 `selectNow()` 补一次。

> **一句话总结这一节**：Netty 的 EventLoop 循环 = `epoll_wait` 阻塞 + 任务队列 + 512 次空轮询重建 + eventfd 唤醒。这四件事拼起来，才是一个生产可用的 Reactor 引擎。

---

# 第四部分　零拷贝：两个完全不同世界的"零拷贝"

> ★★★ 这是面试里**最容易张冠李戴**的知识点。Netty 文档里说的"zero-copy"和内核说的"zero-copy"**不是一回事**，混着答必被追问到崩。

## 4.1 先看没有零拷贝时有多惨

需求：把一个文件通过网络发出去（如静态资源服务、Kafka 日志段传输）。

**朴素做法**：

```java
FileInputStream in = ...;
byte[] buf = new byte[8192];
while ((len = in.read(buf)) > 0) { out.write(buf, 0, len); }
```

底层发生的事（一次 8KB）：

```
read():
  ① DMA: 磁盘 → 内核 page cache          （DMA 拷贝，CPU 不参与）
  ② CPU: page cache → 用户缓冲区          （真拷贝 1）
write():
  ③ CPU: 用户缓冲区 → socket 发送缓冲区    （真拷贝 2）
  ④ DMA: socket 缓冲区 → 网卡              （DMA 拷贝）
另外：4 次用户态/内核态切换（read 进出 + write 进出）
```

**总计：4 次拷贝（2 次 CPU 拷贝）+ 4 次上下文切换 + 2 次 syscall**，每 8KB 重复一遍。

## 4.2 内核零拷贝的四种形态

### ① mmap + write（减少一次 CPU 拷贝）

```
mmap(): 把内核 page cache 映射到用户虚拟地址空间（页表指向同一物理页，无拷贝）
write(): ② CPU: page cache → socket 缓冲区（仍有一次 CPU 拷贝）
```
- 拷贝降为 3 次（1 次 CPU），切换仍 4 次。
- 适合**大文件**；缺点是 mmap 的页表映射/缺页中断成本高，小文件反而更慢，且映射期间文件被截断会 SIGBUS。

### ② sendfile（真正的 2 次切换 + 0 次 CPU 拷贝）

```c
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
```
① DMA: 磁盘 → page cache
② CPU: page cache → socket 缓冲区（仅拷贝"数据位置+长度"这种描述符？—— 不是！）
③ DMA: socket 缓冲区 → 网卡
```
**关键**：`sendfile` 之后，内核发现"数据只是要转发"，于是**不再把数据拷进 socket 缓冲区，而是把 page cache 的页描述符（`skb` 的 `frag`）挂到 socket 的发送队列上**。所以：
- **CPU 拷贝 = 0**（数据始终在 page cache 的物理页里，网卡 DMA 直接按描述符从该页取数据）
- **切换 = 2 次**（一次 sendfile 调用进出）
- 这就是 "**零拷贝**" 的字面含义：**零次 CPU 数据拷贝**。

> 严格说还存在一次"页描述符传递"，但那是元数据（几十字节），不是数据本体。

### ③ sendfile + DMA gather（scatter-gather，硬件级）

新一代网卡支持 `SG-DMA`：DMA 引擎能按**分散的物理页地址列表**直接组装成帧，连"把 page cache 页挂到 socket 缓冲区"这一步的元数据搬运都省了。
- **拷贝 = 0，切换 = 2，CPU 参与 ≈ 0**。

### ④ splice（fd 到 fd 的管道式中转）

```c
splice(int fd_in, ..., int fd_out, ..., size_t len, unsigned int flags);
```
在两个 fd 之间建立**内核管道**直接搬运，不需要用户缓冲区，**也不需要 mmap**。适合"socket → socket"（代理、网关）。
- Nginx 的 `proxy_pass`、HAProxy 转发就用 splice。

**四者对比**：

| 方式 | CPU 拷贝 | 上下文切换 | 限制 |
|---|---|---|---|
| read+write | 2 | 4 | 无 |
| mmap+write | 1 | 4 | 大文件才划算 |
| **sendfile** | **0** | **2** | **输入必须是文件，不能是 socket** |
| splice | 0 | 2 | 需管道，部分场景语义受限 |

**Java 层的映射**：

| Java API | 底层 |
|---|---|
| `FileChannel.transferTo(pos, count, targetChannel)` | Linux 走 `sendfile64`（有 2GB 单次上限，循环调用） |
| `FileChannel.transferFrom(...)` | 可能退化为 `read+write`（取决于方向） |
| `FileChannel.map(...)` → `MappedByteBuffer` | `mmap64` |
| `SocketChannel.write(ByteBuffer)` | `write` / `writev`（若传入多个 buffer） |

## 4.3 Netty 的用户态"零拷贝"（这才是 Netty 文档说的）

Netty 的 zero-copy 指的是**在 JVM 堆内避免无谓的数据复制**，与内核无关。五个手段：

### ① `CompositeByteBuf`：逻辑合并，物理不合并

场景：HTTP 响应由 header + body 组成，传统做法 `ByteBufUtil.merge` 会把两块拷成一块。

```java
CompositeByteBuf composite = Unpooled.compositeBuffer();
composite.addComponents(true, headerBuf, bodyBuf);   // true = 自动增加 writerIndex
// 逻辑上是一个 buf，物理上是两个独立的 ByteBuf，写 socket 时用 writev 一次发出
```

**底层价值**：`writev`（scatter-gather write）让内核一次 syscall 写入多个不连续缓冲区，避免了"先 memcpy 拼接再 write"。

### ② `slice()` / `duplicate()` / `retainedSlice()`

```java
ByteBuf slice = buf.slice(0, 10);      // 共享底层 byte[]，只是 [offset, offset+len) 的视图
ByteBuf dup  = buf.duplicate();        // 共享内存，但独立维护读写指针
```
- `slice` 不拷贝数据，只新建一个"指针窗口"；**引用计数共享**（`retain()` 才安全持有）。
- 典型用途：协议解析时把 header 部分切出来传给下游，body 直接透传。

### ③ `wrappedBuffer()`

把已有的 `byte[]` / `ByteBuffer` 直接包装成 ByteBuf，**零拷贝**：

```java
ByteBuf buf = Unpooled.wrappedBuffer(existingByteArray);  // 不复制数组
```

### ④ `FileRegion` + `transferTo`

```java
FileRegion region = new DefaultFileRegion(fileChannel, 0, fileChannel.size());
channel.writeAndFlush(region);      // 底层 FileChannel.transferTo → sendfile
```
这是 Netty **真正对接内核零拷贝**的唯一入口。**注意**：若加了 `SslHandler`，`FileRegion` 必须降级为普通 ByteBuf（因为加密需要逐字节处理），Netty 会抛 `UnsupportedOperationException` 或自动转换 —— **这是"HTTPS 下大文件传输性能骤降"的经典原因**。

### ⑤ 堆外直接内存（DirectByteBuf）

严格说不是"零拷贝"，但它是"少一次拷贝"：JVM 堆内数组在 GC 时可能**移动**（压缩整理），所以 JVM 必须先把它 pin 住或复制到堆外才能交给内核；用 DirectByteBuffer 则物理地址固定，`write` 可直达。**这就是 Netty 默认用 DirectByteBuf 的根本原因。**

## 4.4 两种零拷贝的对照表（面试背诵版）

| | 内核零拷贝（sendfile/splice） | Netty 零拷贝（用户态） |
|---|---|---|
| 省掉的是 | **CPU 在内核里把数据从 page cache 拷到 socket 缓冲区** | **JVM 堆内的 `memcpy` / 对象复制** |
| 实现者 | 操作系统 | JVM 库 |
| 典型 API | `sendfile`, `splice`, `mmap` | `CompositeByteBuf`, `slice`, `wrap`, `FileRegion` |
| 收益 | 减少 CPU 占用、提升吞吐 | 减少 GC 压力、减少内存带宽占用 |
| 是否减少 syscall | 是（4 次切换 → 2 次） | 不一定（但 `writev` 会） |

**高分回答模板**：
> "零拷贝要分两层。内核层是 sendfile/splice，省掉 page cache 到 socket buffer 的 CPU 拷贝，Netty 里对应 FileRegion；用户层是避免 JVM 内数据复制，Netty 用 CompositeByteBuf、slice、wrap 实现，本质是把'拷贝'换成'视图 + 引用计数'。两者都叫零拷贝，但省的不是同一份成本。"

## 4.5 直接内存的分配、释放与 Netty 的 noCleaner 优化

`ByteBuffer.allocateDirect(n)` 在 JVM 里的真实路径：

```
DirectByteBuffer(int cap)
  → Bits.reserveMemory(size, cap)       // 记账，检查 -XX:MaxDirectMemorySize，不够则 System.gc() 后重试
  → Unsafe.allocateMemory(size)         // malloc，返回堆外地址
  → 创建 Cleaner（一个 PhantomReference 子类），注册到 ReferenceQueue
```

- **释放时机**：只有 DirectByteBuffer 对象被 GC 且 Cleaner 被处理时才 `free`。所以"Full GC 才能回收直接内存"——这是很多 OOM 的根源（堆内存充足，直接内存爆了）。
- 参数：`-XX:MaxDirectMemorySize`（未设置时默认 = 最大堆大小），`-XX:+DisableExplicitGC` 会让 `System.gc()` 失效，导致直接内存回收更慢。

**Netty 的做法（`PlatformDependent0`）**：

```java
// 通过 Unsafe 直接分配，并自己记账
long address = PlatformDependent0.allocateMemory(size);   // Unsafe.allocateMemory
// 释放时直接
PlatformDependent0.freeMemory(address);                   // Unsafe.freeMemory
```

- 这样就**绕开了 Cleaner 和 GC**，`ByteBuf.release()` 立即归还内存（池化场景即归还 PoolArena）。
- 代价：**必须保证 release 被调用**，否则内存泄漏，且 JVM 不知道这块内存，`jmap` 也看不到 → 只能看 `Native Memory Tracking`（`-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory`）。
- 相关参数：
  - `-Dio.netty.maxDirectMemory`：Netty 自己的直接内存上限（默认等于 JVM 的 MaxDirectMemorySize；设为 0 表示不限制，`-1` 表示用 JVM 的）
  - `-Dio.netty.noPreferDirect=true`：退化用堆内存
  - `-Dio.netty.allocator.numDirectArenas`：直接内存 PoolArena 数量

---

# 第五部分　TCP 底层：粘包、Nagle、TIME_WAIT 的物理根源

> ★★★ 这一部分回答的是："为什么 Netty 要写这么多看起来'多余'的代码（编解码器、心跳、半关闭）"。

## 5.1 粘包/半包的物理根源（不是 Netty 的锅）

**TCP 是"字节流（byte stream）"协议，不是"消息（message）"协议。** 它在设计上只保证：
1. 字节不丢、不重、按序；
2. **不保证任何消息边界**。

为什么？因为 TCP 段（segment）的大小由 **MSS / MTU / 拥塞窗口 / 发送缓冲区** 决定，与业务的消息大小**完全无关**：

```
你的业务写了 3 条消息： [100B] [100B] [100B]

可能变成：
  情况A（粘包）：一次收到 [300B]                    ← Nagle 合并 + 发送缓冲区合并
  情况B（半包）：收到 [50B]，下次收到 [250B]          ← MTU=1500 但缓冲区只攒了 50B 就发了
  情况C（混合）：收到 [150B][150B]，第二条被切成两半   ← 最真实的情况
```

**内核视角**：`write()` 只是把数据追加到 `sk_write_queue`，什么时候、切成几个段发出去，由**Nagle、拥塞窗口、发送缓冲区水位**决定，应用层完全不可控。

**唯一正确的解决思路**：**在应用层重建边界**。三种方案：

| 方案 | 适用 | 代价 |
|---|---|---|
| 固定长度 | 定长指令、二进制协议 | 不灵活，浪费带宽 |
| 分隔符（`\r\n`、`\0`） | 文本协议（HTTP header、Redis RESP） | 需处理"分隔符出现在正文"的转义 |
| **长度字段（Length-Prefixed）** | **绝大多数二进制 RPC 协议** | 需要 header 定义，最主流 |

Netty 的 `LengthFieldBasedFrameDecoder` 就是第三种的标准实现，5 个参数（**面试高频，必须能画图解释**）：

```java
new LengthFieldBasedFrameDecoder(
    maxFrameLength,          // 最大帧长，防御 OOM 攻击
    lengthFieldOffset,       // 长度字段在帧内的字节偏移
    lengthFieldLength,       // 长度字段本身占几字节（1/2/3/4/8）
    lengthAdjustment,        // 长度值需要修正多少（因为长度可能不含 header）
    initialBytesToStrip     // 解析成功后丢弃前几个字节（通常丢掉长度字段本身）
);
```

举例（Dubbo 协议头 16 字节，其中 offset=12 是 4 字节的 body 长度）：

```
+--------+--------+--------+--------+--------+--------+
| magic  | flag   | status |  reqId (8B)    | bodyLen(4B) | body...
+--------+--------+--------+--------+--------+--------+
                                        ↑ offset=12, length=4
lengthAdjustment = 0（bodyLen 就是纯 body 长度）
initialBytesToStrip = 16（丢掉整个 header 交给下游解码器）
```

**底层实现要点**（`ByteToMessageDecoder`）：
- 内部维护 `cumulation`（累积缓冲区），半包时把已读数据**累积**起来等下一批；
- `MERGE_CUMULATOR`：把新数据 `memcpy` 到已有 buffer（默认，省内存但有拷贝）；
- `COMPOSITE_CUMULATOR`：用 `CompositeByteBuf` 逻辑拼接（无拷贝，但碎片多，遍历慢）；
- **必须在 `while (in.isReadable())` 循环里解，一次可能解出多条**（这就是粘包的处理方式）；
- 解码出的对象交给下一个 handler 后，**原 ByteBuf 的引用计数要正确释放**（`ReferenceCountUtil.release`）。

## 5.2 三次握手、accept 队列与 `SO_BACKLOG`

```
Client                          Server
  |------ SYN (seq=x) --------->|   ① 服务器创建 request_sock，放入【半连接队列 SYN Queue】
  |<--- SYN+ACK (seq=y,ack=x+1)-|      状态 SYN_RCVD
  |------ ACK (ack=y+1) ------->|   ② 移出 SYN Queue，创建完整 socket，放入【全连接队列 Accept Queue】
                                 |      状态 ESTABLISHED
                                 |   ③ 应用调用 accept()，从 Accept Queue 取出，返回新 fd
```

**关键结论（面试常考）**：

| 队列 | 控制参数 | 满了会怎样 |
|---|---|---|
| 半连接队列（SYN Queue） | `net.ipv4.tcp_max_syn_backlog` | 丢弃新 SYN（客户端重试）；开启 `tcp_syncookies` 可缓解 SYN Flood |
| **全连接队列（Accept Queue）** | **`SO_BACKLOG` + `net.core.somaxconn`，实际长度 = min(backlog, somaxconn)** | 默认**丢弃** ACK（客户端认为已连接，服务端不知）→ 表现为"客户端连上了但请求超时"；若 `tcp_abort_on_overflow=1` 则直接发 RST |

**Netty 的 `SO_BACKLOG`**：
```java
new ServerBootstrap().option(ChannelOption.SO_BACKLOG, 1024);
```
Netty 4.1 的 `DefaultServerSocketChannelConfig` 默认取 `NetUtil.SOMAXCONN`（即读 `/proc/sys/net/core/somaxconn`），**不是固定 128**——很多人答 128 是错的（128 是 JDK `ServerSocket` 的默认值）。

**为什么 Netty 的 accept 队列不容易满？** 因为 boss EventLoop 在 `processSelectedKeys` 里对 `OP_ACCEPT` 做了**批量 accept**（`doReadMessages` 循环读到没有为止，最多 16 个一批），远快于"一次事件 accept 一个"的实现。

**排查命令**：
```bash
ss -lnt                                  # Recv-Q 列 = 当前全连接队列已排队数，Send-Q = backlog 上限
netstat -s | grep -i "listen queue"      # "times the listen queue of a socket overflowed"
```

## 5.3 Nagle、延迟 ACK 与 40ms 之谜（面试绝对高频）

**Nagle 算法（1984）**：为了减少小包（tinygram）对网络的冲击——
> 一个连接上，**最多只有一个未被 ACK 的小包**。若还有未确认数据，新写入的小数据就攒着，直到①收到 ACK 或②攒够 MSS 或③超时（通常 200ms）才发。

**延迟 ACK（Delayed ACK）**：接收方收到数据后不立即回 ACK，而是等一等（Linux 通常 **40ms**），期望"顺便"把 ACK 捎带在反向数据上，减少 ACK 数量。

**两者叠加 = 灾难**：

```
Client 发小包 A（1 字节） → 因为 Nagle，A 立即发出（此时无未确认数据）
Server 收到 A，延迟 ACK，等 40ms 想捎带数据
Client 想发小包 B → Nagle 发现 A 未被 ACK，于是把 B 攒住不发
……40ms 后 Server 的延迟 ACK 到了
Client 才发出 B
```

**结果**：每个小包都多等 40ms。这就是很多"RPC 单次调用偶发 40ms 抖动"的根因（在 request-response 交替的短连接或小包场景中最明显）。

**解决方案**：
1. **`TCP_NODELAY = true`**（关闭 Nagle）：**Netty 对 `NioSocketChannel` 默认就是 `true`**（`DefaultSocketChannelConfig` 构造时 `setTcpNoDelay(true)`）。这与 JDK `Socket` 默认 `false` 不同 —— **高频面试点**。
2. 减少小包：批量写、合并 flush（Netty 的 `write` 不 `flush` 的机制）。
3. 服务端 `TCP_QUICKACK`（Linux 特有，需用 `EpollChannelOption` 或原生 socket 设置，Netty 无直接 option，需自定义）。

**追问：那 NODELAY 会不会导致网络小包风暴？**
> 会。所以 Netty 的正确用法是"**应用层攒批**"：业务一次 `writeAndFlush` 写一个完整消息（而不是一个字段写一次），Netty 的 outbound buffer 会做合并；同时"服务端高频小响应"场景可以适度保留 Nagle。这是一种**用应用层控制替代内核控制**的权衡。

## 5.4 滑动窗口、拥塞控制与 Netty 的关系

| 机制 | 层次 | 作用 | 对 Netty 的影响 |
|---|---|---|---|
| 滑动窗口（接收窗口） | TCP | 接收方告诉发送方"我还能收多少" | `SO_RCVBUF` 决定窗口上限；窗口=0 → 对端停发 → 表现为"写成功但对方收不到" |
| 拥塞窗口 cwnd | TCP | 发送方根据丢包/时延自我限速 | 影响吞吐爬坡速度，与 Netty 无关但影响压测结论 |
| 慢启动 | TCP | 初始 cwnd 小，指数增长 | **短连接永远在慢启动**，这是"长连接比短连接快"的物理原因之一 |
| 快重传 / 快恢复 | TCP | 3 个重复 ACK 立即重传 | 影响 RTT 抖动 |
| BBR | TCP | 基于带宽×RTT 建模，不看丢包 | 高丢包链路（跨国）提升巨大，需内核 4.9+ |

**面试延伸**：
> "为什么我的服务 QPS 上不去，但 CPU 才 30%？"——可能是 `SO_RCVBUF` 太小导致窗口受限，或 `accept` 队列溢出丢连接，或客户端连接池太小导致 RTT 成为瓶颈（Little's Law：并发 = QPS × 延迟）。

## 5.5 TIME_WAIT 与 CLOSE_WAIT

**四次挥手**：

```
主动关闭方                     被动关闭方
FIN_WAIT_1  --- FIN --------->        CLOSE_WAIT
                                      （应用还没 close()，内核已收到 FIN）
FIN_WAIT_2  <-- ACK ----------
            <-- FIN ----------        LAST_ACK
TIME_WAIT   --- ACK --------->        CLOSED
（等 2MSL = Linux 固定 60s）
CLOSED
```

| 状态 | 谁产生 | 含义 | 常见成因 |
|---|---|---|---|
| **TIME_WAIT** | **主动关闭方** | 等 2MSL，保证最后一个 ACK 能重传、让旧连接的迷路报文过期 | 大量**短连接**（HTTP 无 keepalive、RPC 每次新建连接）；端口耗尽 → `Cannot assign requested address` |
| **CLOSE_WAIT** | 被动关闭方 | 内核收到 FIN 并回了 ACK，但**应用一直没调 close()** | **代码 bug**：连接泄漏、异常分支没关闭、Netty 里 `channel.close()` 漏调 |

**判断口诀**：**TIME_WAIT 多 = 架构问题（该用连接池/长连接）；CLOSE_WAIT 多 = 代码问题（忘了关闭）。**

**TIME_WAIT 的缓解**：

| 方案 | 说明 | 风险 |
|---|---|---|
| 连接池 / 长连接 | 治本 | 需处理连接失效（心跳） |
| `net.ipv4.tcp_tw_reuse=1` | 允许**主动发起**的新连接复用 TIME_WAIT 的端口（需 `tcp_timestamps=1`） | 仅对 outbound 有效，NAT 场景有极少概率串包 |
| 扩大 `ip_local_port_range` | 28232~60999 → 1024~65535 | 加剧与其他服务的端口冲突 |
| `SO_LINGER=0` | close 时直接发 RST，跳过 TIME_WAIT | **危险**：可能丢未发完的数据，且对端收到 RST 报错 |
| 让**服务端**主动关闭 | 把 TIME_WAIT 转移到服务端（服务端端口固定，不吃端口） | 需协议层配合（如 HTTP 的 `Connection: close` 由服务端决定） |

> `tcp_tw_recycle` 已在 Linux 4.12 被**移除**（NAT 场景下会误杀连接），面试若有人提它，可以直接指出这是过时/危险配置 —— 加分项。

## 5.6 TCP Keepalive vs 应用层心跳

| | TCP `SO_KEEPALIVE` | 应用层心跳（Netty `IdleStateHandler`） |
|---|---|---|
| 探测对象 | **连接是否还活着**（对端主机/进程） | **对端业务是否还活着**（进程卡死、GC 停顿、线程池满，TCP 层仍然"活着"） |
| 默认间隔 | Linux 7200s（2 小时）才开始第一次探测 | 秒级/分钟级，可自定 |
| 能否穿透代理/NAT | 不能（很多中间设备会拦截空 ACK） | 能（是真实业务数据） |
| 能否发现"假死" | 不能 | 能 |

**所以生产环境的标准做法**：`SO_KEEPALIVE=true`（兜底）+ **应用层心跳**（主手段）+ 空闲检测断开（`IdleStateHandler`）。
**代价**：心跳本身是流量。Netty 官方给的思路是"心跳间隔 = 业务可接受的最大发现故障延迟"，通常 30s~60s。

## 5.7 Netty 的 `ALLOW_HALF_CLOSURE`

TCP 支持**半关闭**（一方发 FIN 表示"我不再发数据了"，但仍可接收）。默认 JDK 行为是收到 FIN 就整个关闭。Netty 提供：
```java
.childOption(ChannelOption.ALLOW_HALF_CLOSURE, true);   // 默认 false
```
开启后，收到对端 FIN 只触发 `ChannelInputShutdownEvent`（userEvent），channel 仍可写出数据。用于"客户端发完请求就半关，服务端继续回包"的场景。

---

# 第六部分　Reactor 模式与 Netty 线程模型

> ★★ 概念不难，但**"为什么这么设计"**才是考点。

## 6.1 Reactor 模式的三种演进

**Reactor 的核心定义**：一个或多个线程监听事件（`select`），把事件**分发给对应的 handler** 处理。它是"**事件驱动 + 分发**"，本质是观察者模式的 IO 版本。

### ① 单 Reactor 单线程（Redis 模型）

```
        ┌──────────────────────────┐
        │  Reactor (1 thread)      │
        │  select → dispatch       │
        └───────┬──────────────────┘
                ↓
     ┌──────────┴──────────┐
  Acceptor              Handler(read→decode→process→encode→write)
```

- 优点：无线程切换、无锁、实现极简。
- 缺点：**一个 handler 阻塞，全部连接卡死**；无法利用多核。
- 代表：Redis（6.0 前）、Memcached（部分）、Node.js。

### ② 单 Reactor 多线程

```
Reactor(1) → select → dispatch → 把 handler 丢进线程池
```
- 优点：业务可并行，能利用多核。
- 缺点：**Reactor 仍是单点**（accept 和 read 都在一个线程），高并发下 IO 处理仍是瓶颈；且 handler 跨线程操作共享状态需加锁。

### ③ 主从 Reactor 多线程（Netty / Nginx / Memcached 生产版）

```
        MainReactor (boss, 只做 accept)
                │  accept 到连接后，注册到某个 SubReactor
    ┌───────────┼───────────┐
SubReactor1  SubReactor2  SubReactor3   （每个都是一条独立 epoll 线程）
  Channel A    Channel C    Channel D
  Channel B                 Channel E
```
- 优点：accept 与 read/write 解耦；每个 SubReactor 独立 epoll，无共享；一台 8 核机器可线性扩展 IO 吞吐。
- 现实：**Nginx 用"多进程 + SO_REUSEPORT"近似这个模型**，Netty 用"多线程 + 一个 boss selector"。

## 6.2 Netty 的线程模型精确描述

```java
EventLoopGroup bossGroup   = new NioEventLoopGroup(1);        // 默认 1？见下
EventLoopGroup workerGroup = new NioEventLoopGroup();         // 默认 2 * CPU 核数
```

**默认值来源**（`MultithreadEventLoopGroup.DEFAULT_EVENT_LOOP_THREADS`）：
```java
DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
    "io.netty.eventLoopThreads", NetRuntimeProcessorHolder.availableProcessors() * 2));
```
即 **CPU 核数 × 2**。为什么 ×2 而不是 ×1 或 ×8？
- IO 线程并不总是满负荷（`epoll_wait` 时是睡眠的），留一倍余量应对突发和少量阻塞；
- 再多的线程会带来可观的上下文切换和 cache 失效，收益递减；
- **最佳实践是可配置**，并根据压测调。有大量 CPU 密集编解码时可适当增加，纯转发可减少到核数。

**boss 线程数**：`new NioEventLoopGroup(1)` 是常见写法，但**默认（无参）也是 2×CPU**。为什么通常只给 1？
- 一个 listen socket 只注册在一个 boss EventLoop 上（**一个 Channel 一生只绑定一个 EventLoop**），多余的 boss 线程只是在"多个 ServerBootstrap 或 SO_REUSEPORT 多监听"时才有用。
- 想用多 boss 提升 accept 能力，正确方式是 `SO_REUSEPORT` 绑定多个 `ServerBootstrap`（每个 boss 一个独立 listen fd）。

**核心不变式（务必背下来）**：
1. 一个 `Channel` 从注册开始，**终生只绑定一个 `EventLoop`**；
2. 一个 `EventLoop` 是一个**单线程执行器**（内部就一个 Thread）；
3. 一个 `EventLoop` 可以绑定**多个 Channel**（成千上万）；
4. 一个 `Channel` 的**所有 handler 回调都在其 EventLoop 线程里串行执行**。

推导出的结果：**同一个 Channel 的 pipeline 里，永远不会有并发**（天然线程安全，无需加锁）；**不同 Channel 之间才可能并发**（所以 handler 若有共享状态必须加锁或用 `@Sharable` + 线程安全实现）。

## 6.3 为什么这么设计？—— "无锁串行化"（Serialized Execution）

如果改成"线程池处理所有连接的 IO"：
- 1000 个连接的事件被 16 个线程抢着处理 → 每个 Channel 的状态（读指针、解码器累积缓冲、pipeline 状态）都要加锁；
- 锁竞争 + 数据在核间迁移（cache line bouncing）= 性能塌陷；
- 每次都要重新"装载"Channel 的上下文，CPU cache 全废。

Netty 的选择：**用"亲和性"替代"锁"**。把 Channel 分片（round-robin 到 EventLoop），同一分片内串行，分片间无共享。这就是 **thread-per-channel 的流水线**，等价于"每个连接一个逻辑线程"，但物理线程只有 N 个。

> **类比**：Actor 模型（每个 Channel 是一个 Actor，EventLoop 是 Actor 的调度器）。这也是为什么 Netty 的 handler 里可以放心用普通字段而不用 `volatile`/`synchronized`。

## 6.4 业务阻塞的后果与三种解法

**一个问题**：如果在 `channelRead` 里做了一次 200ms 的数据库查询，会发生什么？

```
该 EventLoop 线程被占住 200ms
  → 它绑定的其它所有 Channel（可能是几千个）在这 200ms 内：
      - 无法 read（内核接收缓冲区堆积 → 窗口变小 → 对端发送受阻）
      - 无法 write（写队列堆积 → 内存上涨 → 触发水位线）
      - 无法执行定时任务（心跳超时误判）
  → 表现为"整体 RT 抖成狗"，但 CPU 不高（线程在等 DB）
```

**三种解法（按推荐度排序）**：

**① 保持异步（首选）**：用异步客户端（异步 DB/HTTP/Redis），handler 里只发请求 + 注册回调，立即返回。这是 Netty 官方推荐，也是 Reactor 的"正统用法"。

**② `EventExecutorGroup` 隔离（次选）**：
```java
EventExecutorGroup bizGroup = new DefaultEventExecutorGroup(16);
pipeline.addLast(bizGroup, "bizHandler", new MyBlockingHandler());
```
- 原理：`AbstractChannelHandlerContext.executor()` 返回的是**绑定到该 handler 的执行器**；pipeline 传播到该 handler 时，若不在其 executor 线程，就 `executor.execute(task)` 投递过去。
- **代价（面试深挖点）**：
  - 每次跨越 executor 都要**入队 + 线程切换**；
  - **内存可见性**：业务线程改了状态，回到 IO 线程可能看不到（需要 `volatile`/同步）；
  - **失去"同 Channel 串行"保证**：如果同一 Channel 的多个 read 被投递到同一个 `EventExecutorGroup`，`DefaultEventExecutorGroup` 按 Channel 做一致性哈希选择 executor，**同一 Channel 仍会落到同一个 executor**，所以串行性得以保留 —— 这是很多人不知道的细节。
- 何时用：**无法避免的阻塞调用**（老 JDBC、同步 RPC、磁盘 IO）。

**③ 扩大 worker 数**：治标不治本，只适合"偶尔短阻塞"。

## 6.5 Reactor 与 Netty 的对应关系总表

| Reactor 概念 | Netty 对应 | 说明 |
|---|---|---|
| Reactor | `EventLoop` / `NioEventLoop` | 一个 select 循环 + 任务队列 |
| MainReactor | `bossGroup` | 只关心 `OP_ACCEPT` |
| SubReactor | `workerGroup` | 关心 `OP_READ` / `OP_WRITE` |
| Acceptor | `ServerBootstrapAcceptor`（pipeline 里的一个 handler） | accept 后 `childGroup.register(child)` |
| Handler | `ChannelHandler` | 业务/编解码/心跳 |
| Dispatch | `ChannelPipeline` 事件传播 | `head → ... → tail` |
| 事件 | `ChannelInboundEvent` / `ChannelOutboundEvent` | read / write / close 等 |

**一句话总结**：**Netty = 主从 Reactor + 流水线（Pipeline）+ 可池化内存（ByteBuf）+ 异步结果（Future）**，四件套缺一不可。

---

# 第七部分　EventLoop 源码级剖析

> ★★★★ 面试的分水岭就在这一节。能把 `run()` 循环讲成"一条时间线"，而不是罗列方法名，就是"看过源码"和"背过八股"的区别。

## 7.1 类继承结构（先建立坐标系）

```
EventExecutorGroup
 └─ MultithreadEventExecutorGroup
     └─ MultithreadEventLoopGroup
         ├─ NioEventLoopGroup          （Java NIO transport）
         └─ EpollEventLoopGroup        （native epoll transport，Linux）
 └─ EventLoop (接口)
     └─ SingleThreadEventLoop
         ├─ NioEventLoop
         └─ EpollEventLoop
 └─ EventExecutorGroup 也实现了 ScheduledExecutorService 语义
```

三个必记字段：

| 字段 | 类型 | 作用 |
|---|---|---|
| `children[]` | `EventExecutor[]` | 线程池里的"槽位"，round-robin 分配给新 Channel |
| `chooser` / `chooserFactory` | `EventExecutorChooser` | 选择器，2 的幂时用 `idx & (len-1)` 位运算，否则 `%` |
| `SingleThreadEventExecutor.thread` | `Thread` | **懒启动**，第一次 `execute()` 才 `new Thread` 并启动 |

> 细节：EventLoop 的线程是**懒启动**的。`NioEventLoopGroup` 构造时不创建线程，直到第一个 Channel 注册或第一个任务提交。这也是"Netty 启动很快"的原因之一。

## 7.2 `NioEventLoop.run()` 主循环：一条时间线

```java
@Override
protected void run() {
    // 记录上一次循环"什么都没干"的次数，用于检测空轮询/饥饿
    int selectCnt = 0;
    for (;;) {                                   // ← 这就是"事件循环"
        try {
            int strategy;
            try {
                // ① 决策：要不要 select？selectNow 还是阻塞 select？
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                    case SelectStrategy.CONTINUE:    // 明确要求跳过本轮
                        continue;
                    case SelectStrategy.BUSY_WAIT:   // 忙等策略（有任务时的一种选择）
                    case SelectStrategy.SELECT:
                        // ② 计算 deadline：下一个定时任务还有多久到期
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE;         // Long.MAX_VALUE，表示"没有定时任务"
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                // ③ 阻塞等待 IO 事件
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                    default:
                }
            } catch (IOException e) {
                rebuildSelector0();                  // select 抛异常 → 重建 selector
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            // ④ 处理就绪的 IO 事件（读写 accept connect）
            processSelectedKeys();
            // ⑤ 处理普通任务队列 + 到期的定时任务
            runAllTasks();
            ...
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

**四个阶段对应的四类"事件源"**：

| 阶段 | 事件源 | 处理内容 |
|---|---|---|
| ②③ `select` | 内核 epoll | 阻塞等待，被 IO 事件或 `wakeup()` 唤醒 |
| ④ `processSelectedKeys` | 就绪 fd | `accept` / `read` / `write` / `connect` / `close` |
| ⑤ `runAllTasks` | 用户提交的任务 | `channel.writeAndFlush`（跨线程）、业务 `execute`、定时任务、`register` |
| 循环尾部 | 空闲/关闭 | `closeAll()`、`confirmShutdown()`、空轮询检测 |

**关键问题：为什么"没有定时任务时"可以无限阻塞？**

因为 `select(NONE)` 就是 `selector.select()`（无超时，永久阻塞）。它能被三种情况唤醒：
1. **fd 就绪**（正常 IO 事件）→ 内核 `ep_poll_callback`；
2. **`selector.wakeup()`**（别的线程往 eventfd 写 1 字节）→ 这是"任务投递"的唤醒路径；
3. **spurious wakeup**（极少，假唤醒，重新 select 即可）。

所以 Netty 的"无限阻塞"是安全的 —— **只要有任务提交，就一定会 `wakeup`**。这解释了 `SingleThreadEventExecutor.execute()` 里那行代码：

```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();                                   // 懒启动
        if (isShutdown()) { reject(); }
    }
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);                             // ★ 关键：唤醒 select
    }
}
```

**`wakeup()` 的优化（很多人不知道）**：

```java
protected void wakeup(boolean inEventLoop) {
    // 只有"当前不在 loop 线程" 且 "loop 确实在睡" 才真正 wakeup
    if (!inEventLoop && nextWakeupNanos.getAndSet(AWAKE) != AWAKE) {
        selector.wakeup();       // → write(eventfd, 1) → 一次 syscall
    }
}
```

`nextWakeupNanos` 是一个 `AtomicLong`，loop 在 select 前把"预期唤醒时间"写进去，`wakeup` 用 `getAndSet` 判断"现在是否真的在睡"：
- 如果 loop 已经醒了（值已是 `AWAKE`），**跳过 syscall**，因为下一次循环必然会看到 taskQueue 里的任务。
- 这个小优化在高并发写场景下非常关键：**避免每写一次都做一次 `eventfd` 系统调用**。

> **面试可以这样答**："Netty 用了两级唤醒——先是 AtomicLong 状态位过滤掉无效唤醒，再是 selector.wakeup 走 eventfd。因为 wakeup 是一次 syscall，在几十万 QPS 下这个优化能省掉大量内核态切换。"

## 7.3 任务队列体系：三队列 + 一个"偷懒"机制

| 队列 | 类型 | 用途 |
|---|---|---|
| `taskQueue` | MPSC 无锁队列（有 JCTools 时用 `MpscUnboundedArrayQueue`） | 普通任务（`execute` 提交） |
| `tailTasks` | MPSC 队列 | "本轮循环末尾"执行的任务（`executeAfterEventLoopIteration`，较少用） |
| `scheduledTaskQueue` | `PriorityQueue<ScheduledFutureTask>` | 定时任务（**按 deadline 排序的小顶堆**） |

**每个循环开头的动作**：
```java
private boolean fetchFromScheduledTaskQueue() {
    // 把 scheduledTaskQueue 里"已到期"的任务搬到 taskQueue
    // 对比堆顶任务的 deadline 与当前时间，逐个取出
}
```
这就是"定时任务最终变成普通任务被执行"的桥。

**`runAllTasks` 的执行预算**：

```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();                       // ① 到期定时任务下沉
    Runnable task = pollTask();
    if (task == null) { afterRunningAllTasks(); return false; }
    long deadline = timeoutNanos > 0 ? nanoTime() + timeoutNanos : 0;
    int i = 0;
    do {
        runTask(task);
        if ((i & 0x3F) == 0 && deadline != 0 && nanoTime() >= deadline) break;  // ★ 每 64 个检查一次时间
    } while ((task = pollTask()) != null);
    afterRunningAllTasks();                              // 处理 tailTasks
    return true;
}
```

**两个细节（面试加分）**：
1. **每 64 个任务才检查一次时间**（`i & 0x3F`），因为 `nanoTime()` 本身也有开销（虽然 vDSO 已经把 syscall 变成用户态读）。
2. **任务不是一次性跑完的**：有 `timeoutNanos` 预算，跑不完留到下轮，避免任务队列把 IO 饿死。

## 7.4 `ioRatio`：IO 与任务的 CPU 分配器

```java
long ioStartTime = System.nanoTime();
processSelectedKeys();                       // IO 耗时记为 ioTime
if (ioRatio == 100) {
    runAllTasks();                           // 不限制
} else {
    long ioTime = System.nanoTime() - ioStartTime;
    // ★ 核心公式：任务可用时间 = ioTime × (100 - ioRatio) / ioRatio
    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
}
```

- **默认 `ioRatio = 50`**：意味着"任务时间预算 = IO 时间预算"，即 IO 和任务各占 50% CPU。
- 调大（如 80）：更多时间给 IO，适合**纯转发/高吞吐**场景。
- 调小（如 30）：更多时间给任务，适合**写回频繁、定时任务多**的场景（如推送服务）。
- 设为 100：任务独占，**慎用**（任务队列大时会让 IO 饿死）。

> 这是一个非常"工程化"的设计：Netty 不试图精确调度，而是用"上次 IO 花了多久"来反推"这次该给任务多久"，简单且有效。

## 7.5 `processSelectedKeys`：事件分发的最后一公里

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();     // 用 Netty 的数组化 SelectedSelectionKeySet
    } else {
        processSelectedKeysPlain(selector.selectedKeys());   // 用 JDK 原生 HashSet
    }
}

private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; i++) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null;            // ★ 帮助 GC，防止 key 被长期持有
        final Object a = k.attachment();
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            // NioTask 场景（如注册自定义任务）
        }
        ...
    }
    selectedKeys.reset();                        // size = 0，数组复用
}
```

`processSelectedKey` 里的**关键分支**：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) { unsafe.close(...); return; }
    try {
        int readyOps = k.readyOps();
        // ① CONNECT 完成
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            unsafe.finishConnect();
        }
        // ② WRITE 可写
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.forceFlush();                 // 把堆积的 outbound buffer 继续写出去
        }
        // ③ READ / ACCEPT
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();                   // ★ 真正的读：Byte 用 doReadBytes，Message 用 doReadMessages
            if (!ch.isOpen()) return;
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(...);
    }
}
```

**三个隐蔽的细节**：

1. **`readyOps == 0` 也会 read**：这是为了兜住 epoll 的 spurious/异常返回，防止事件被漏掉。
2. **读完要检查 `ch.isOpen()`**：因为 `read` 过程中 handler 可能把 channel 关了（比如解码出非法包）。
3. **`processSelectedKey` 里有"清除无效 OP_WRITE"的防御**：某些 JDK 版本下 OP_WRITE 会一直就绪，导致 `forceFlush` 空转 → Netty 在 `flush0` 里做了"若写不出去就注销 OP_WRITE，可写了再注册"的配对逻辑。

## 7.6 读事件的两个实现分支（`NioByteUnsafe` vs `NioMessageUnsafe`）

**服务端（`NioServerSocketChannel`，`OP_ACCEPT`）→ `NioMessageUnsafe.read()`**：

```java
public void read() {
    int localRead = 0, totalRead = 0;
    try {
        for (;;) {
            int read = doReadMessages(readBuf);          // → accept() 系统调用，可能有多个
            if (read == 0) break;
            totalRead += read;
            if (read < config.getMaxMessagesPerRead()) break;   // 默认 16，公平性
        }
    } finally {
        int size = readBuf.size();
        for (int i = 0; i < size; i++) {
            pipeline.fireChannelRead(readBuf.get(i));     // ★ 每个新连接触发一次 channelRead
        }
        readBuf.clear();
        pipeline.fireChannelReadComplete();
    }
}
```

`doReadMessages` 在 `NioServerSocketChannel` 里就是：
```java
SocketChannel ch = SocketUtils.accept(javaChannel());   // → accept4() 系统调用
```
**注意 `accept4()`**：比 `accept()` 多一个 flags 参数，可以原子地设置 `SOCK_NONBLOCK`（避免"accept 后再 fcntl"的竞态）。

**客户端/连接（`NioSocketChannel`，`OP_READ`）→ `NioByteUnsafe.read()`**：

```java
public void read() {
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            byteBuf = allocHandle.allocate(allocator);           // ★ 自适应分配
            allocHandle.lastBytesRead(doReadBytes(byteBuf));     // → read() 系统调用
            if (allocHandle.lastBytesRead() <= 0) {
                byteBuf.release();
                byteBuf = null;
                if (lastBytesRead < 0) close = true;             // EOF → 关闭
                break;
            }
            allocHandle.incMessagesRead(1);
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;                                       // ★ 所有权转移给 pipeline
        } while (allocHandle.continueReading());                  // ★ 循环读，模拟 ET
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();
        if (close) closeOnRead(pipeline);
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();                                       // 非 autoRead 时读一次就摘掉 OP_READ
        }
    }
}
```

**`continueReading()` 的判断条件（重要）**：
```java
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead()
        && totalMessages < maxMessagePerRead                    // 默认 16，防单连接饿死其他连接
        && attemptedBytesRead == lastBytesRead;                 // ★ 上次读满了，说明可能还有数据
}
```
> **这就是 Netty 在 LT 模式下模拟 ET 的方式**：只要"上次一次把缓冲区读满"，就继续读，直到读不满或达到 16 次。既避免了 ET 漏事件的坑，又拿到了 ET 的 syscall 效率。

## 7.7 自适应接收缓冲区：`AdaptiveRecvByteBufAllocator`

```java
static final int[] SIZE_TABLE = {16, 32, 48, ..., 512, 1024, 2048, 4096, 8192, 16384, 32768, 65536};
```
- 初始分配 1024（`DEFAULT_INITIAL_BUFFER_SIZE`）。
- **若上次把 buffer 读满**（说明数据可能更多）→ 下次分配时**放大到下一档**；
- **若上次只读了不到一半** → **缩小到上一档**；
- 上限 `maxBytesPerRead`（默认 65536）。

**价值**：小包连接不浪费内存，大流量连接自动扩到 64KB，减少 syscall 次数。**这是"Netty 在大流量下 syscall 更少"的隐形功臣**，也是面试里少见的加分点。

## 7.8 OP_WRITE 什么时候注册？（高频面试题变体）

**错误回答**："一直监听 OP_WRITE，有可写事件就写。"

**为什么错**：Linux 下 socket 发送缓冲区**绝大多数时间是空的**，也就是说 `EPOLLOUT` **永远就绪**。若一直注册，`epoll_wait` 每次都会立即返回，`processSelectedKey` 每次调 `forceFlush`，**变成 CPU 100% 的空转**（LT 模式下尤其明显）。

**Netty 的正确做法**（`AbstractNioByteChannel` + `ChannelOutboundBuffer`）：

```
flush0():
    do {
        written = doWrite(outboundBuffer);     // → write / writev 系统调用
    } while (written > 0 && outboundBuffer.nonEmpty());
    if (outboundBuffer.isEmpty()) {         // 写完了
        clearOpWrite();                     // 注销 OP_WRITE
        return;
    }
    if (!isFlushPending()) {                // 没写完
        setOpWrite();                       // ★ 注册 OP_WRITE，等内核通知"可写了"
    }
```

**配对原则**：**只在"写不动"时注册 OP_WRITE，写完立刻注销**。这样：
- 正常情况：写一次 syscall 就完了，**根本不进入 epoll 的写事件路径**；
- 缓冲满：注册 OP_WRITE，等内核腾出空间，再 `forceFlush`。

**延伸追问："如果一直写不出去呢？"**
→ 数据堆积在 `ChannelOutboundBuffer`（堆外内存），超过 `writeBufferHighWaterMark`（64KB）时 `channel.isWritable()` 变 `false`，业务应做**背压**（停止读/降速）；超过 `WRITE_BUFFER_HIGH_WATER_MARK` 很多会 OOM。**这就是"背压（backpressure）"在 Netty 里的落地机制**。

## 7.9 `shutdownGracefully`：优雅关闭的三段时间

```java
EventLoopGroup group = ...;
group.shutdownGracefully(2, 15, TimeUnit.SECONDS);   // quietPeriod=2s, timeout=15s
```

| 参数 | 默认 | 含义 |
|---|---|---|
| `quietPeriod` | 2s | 关闭后**再等多久没有新任务**才真正退出（防止关闭瞬间还有新任务进来） |
| `timeout` | 15s | 最长等待上限，超时**强制关闭**所有 channel |

**内部状态机**：`ST_NOT_STARTED → ST_STARTED → ST_SHUTTING_DOWN → ST_SHUTDOWN → ST_TERMINATED`，`confirmShutdown()` 每轮检查"任务队列空 + 无定时任务 + quietPeriod 已过"。

**生产实践**：
- 关闭顺序：**先关 boss（停止接新连接）→ 再关 worker（处理完存量）→ 最后关 business group**；
- `channel.closeFuture().sync()` 或 `ChannelGroup.close().await()` 配合使用；
- 常见坑：`shutdownGracefully()` 后不 `awaitTermination` 就退出 main 线程，会导致部分 channel 未刷完数据（`SO_LINGER` 相关）。

## 7.10 `FastThreadLocal`：为什么比 JDK 的 ThreadLocal 快

JDK `ThreadLocal` 的问题：
1. `ThreadLocalMap` 用**开放寻址 + 线性探测**，hash 冲突时性能下降；
2. `Entry` 是 `WeakReference<ThreadLocal>`，需要定期 `expungeStaleEntry` 清理，否则**内存泄漏**；
3. 每次 `get` 要算 `threadLocalHashCode`（虽然是 magic number 递增，但仍有开销）。

Netty 的实现（`FastThreadLocal` + `InternalThreadLocalMap`）：

```java
// 每个 FastThreadLocal 在构造时分配一个全局递增的下标
private static final AtomicInteger NEXT_INDEX = new AtomicInteger();
private final int index = NEXT_INDEX.getAndIncrement();

// 线程本地存储就是一个 Object[] 数组
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    Object v = threadLocalMap.indexedVariable(index);      // 数组下标访问，O(1)
    ...
}
```

**核心差异**：

| | ThreadLocal | FastThreadLocal |
|---|---|---|
| 存储结构 | 哈希表（开放寻址） | **数组，按下标直接寻址** |
| 查找复杂度 | 平均 O(1)，冲突时退化 | **严格 O(1)** |
| 清理 | WeakReference + 惰性清理 | **显式 `removeAll()`**（由 `FastThreadLocalRunnable` 在任务结束时统一清理） |
| 前提 | 无 | **线程必须是 `FastThreadLocalThread`**（Netty 自己的线程都是；否则退化到 `SlowThreadLocalMap`） |

**为什么能严格 O(1)**：因为 index 是**编译期/构造期确定**的，不需要 hash，也就没有冲突。**代价**是数组可能稀疏（很多 FastThreadLocal 但只用少数），每个线程一个 `Object[]`，内存开销与"全局 FastThreadLocal 数量"成正比。

**延伸**：`InternalThreadLocalMap` 还缓存了 `ByteBufAllocator`、`charset`、`StringBuilder` 等常用对象，减少重复查找 —— 这也是 Netty 的"零碎优化"风格。

## 7.11 定时任务：`PriorityQueue` 而不是时间轮（一个常见误解）

**Netty 的 `EventLoop.schedule()` 用的是 `PriorityQueue<ScheduledFutureTask>`**（小顶堆），不是时间轮。
- 插入/删除：O(log n)
- 取最近到期：O(1) 看堆顶
- 每次循环 `fetchFromScheduledTaskQueue()` 把到期任务搬到 `taskQueue`

**那 `HashedWheelTimer` 用在哪？** —— 它是 Netty 提供的**独立工具类**，不用于 `EventLoop` 内部调度：

| 对比 | `EventLoop.schedule` | `HashedWheelTimer` |
|---|---|---|
| 数据结构 | 优先队列（小顶堆） | **时间轮**（512 个 bucket 的环形数组 + 链表） |
| 插入复杂度 | O(log n) | **O(1)** |
| 精度 | 纳秒级（受 select 唤醒影响） | `tickDuration`（默认 100ms） |
| 适用 | 少量、需高精度的定时任务 | **海量**（几十万）超时任务，精度要求低 |
| 典型使用者 | `IdleStateHandler`、`ReadTimeoutHandler` | Dubbo 的请求超时、连接空闲检测、自定义超时框架 |

**时间轮原理（一句话）**：把时间轴切成固定大小的 tick，任务按"到期 tick"挂到对应 bucket 的链表上；一个 worker 线程每 tick 前进一格，处理该 bucket。多圈任务用 `remainingRounds` 计数。**本质是用"精度换 O(1)"**。

> **面试陷阱**："Netty 的定时任务是不是用时间轮实现的？"——**不是**。EventLoop 用优先队列，`HashedWheelTimer` 是可选工具。答错会显得只看了博客。

---

# 第八部分　Channel / Pipeline / Handler 事件传播机制

> ★★★ Pipeline 是 Netty 最"反直觉"的部分：**同一个 `write`，从 `channel` 发起和从 `ctx` 发起，走的路径完全不同**。

## 8.1 数据结构：一个双向链表 + 两个哨兵

```
head ──► handlerA ──► handlerB ──► handlerC ──► tail
 ◄──      ◄──          ◄──          ◄──
```

- `head`（`HeadContext`）：既是 inbound 也是 outbound，**outbound 的终点**（真正调 `unsafe.write` / `unsafe.read`）；
- `tail`（`TailContext`）：inbound 的终点，**负责兜底释放 msg**（`ReferenceCountUtil.release(msg)`）和记录未处理异常；
- 每个节点是 `AbstractChannelHandlerContext`，持有 `handler`、`next`、`prev`、`executor`、`pipeline`。

```java
// DefaultChannelPipeline 的构造
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    tail = new TailContext(this);
    head = new HeadContext(this);
    head.next = tail;
    tail.prev = head;
}
```

**添加 handler 是线程安全的**：`addLast` 内部 `synchronized (this)`，并通过 `callHandlerAdded0` 保证 `handlerAdded` 回调在 handler 的 executor 里执行（不在的话投递过去）。

## 8.2 入站 vs 出站：方向完全相反

```
                     ┌──────────────────────────────────────────┐
   入站 (inbound)    │  head → h1 → h2 → h3 → tail              │ 顺着 next 走
   read/active      └──────────────────────────────────────────┘

                     ┌──────────────────────────────────────────┐
   出站 (outbound)   │  head ← h1 ← h2 ← h3 ← tail              │ 逆着 prev 走
   write/flush/close └──────────────────────────────────────────┘
```

**为什么相反？** 因为语义是"**从哪来，回哪去**"：
- 入站事件来自 socket，从 `head` 进入，**按添加顺序**经过业务 handler；
- 出站事件来自业务，从当前点出发，**按相反顺序**回到 `head`（socket）。

**这解释了一个经典现象**：
```java
pipeline.addLast(new LengthFieldBasedFrameDecoder(...));  // 解码：入站
pipeline.addLast(new LengthFieldPrepender(...));          // 编码：出站
pipeline.addLast(new BusinessHandler());                   // 业务

// 出站时 BusinessHandler.write() 的实际路径：
//   BusinessHandler → (prev) LengthFieldPrepender → (prev) LengthFieldBasedFrameDecoder(跳过，非出站) → head → socket
// 也就是说：编码器"看起来在后面"，实际上先于 socket 执行 ✔
```

## 8.3 `channel.write` vs `ctx.write` vs `writeAndFlush`（必考）

| 调用 | 起点 | 经过的 handler | 是否 flush | 线程安全 |
|---|---|---|---|---|
| `channel.writeAndFlush(msg)` | **tail** | 所有 outbound handler | 是 | 是（会检查 executor） |
| `channel.write(msg)` | **tail** | 所有 outbound handler | **否** | 是 |
| `ctx.write(msg)` | **当前 ctx 的前一个** | 只经过**当前节点之前**的 outbound handler | 否 | **否**（应在 eventLoop 内调用） |
| `ctx.writeAndFlush(msg)` | 同上 | 同上 | 是 | 否 |

源码依据（`AbstractChannelHandlerContext.write`）：
```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ObjectUtil.checkNotNull(msg, "msg");
    // ① 找到下一个 outbound context（从当前节点的 prev 开始找！）
    final AbstractChannelHandlerContext next = findContextOutbound(flush ? MASK_WRITE : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    // ② 线程判定：不在就投递
    if (executor.inEventLoop()) {
        if (flush) next.invokeWriteAndFlush(m, promise);
        else next.invokeWrite(m, promise);
    } else {
        executor.execute(() -> { ... });       // ★ 这就是"跨线程写线程安全"的实现
    }
}
```

**为什么 `ctx.write` 要"从 prev 开始找"**？因为 `ctx.write` 的语义是"**从我的下一个出站处理器开始往外发**"，**不包括自己**（避免 handler 自己处理自己写出的消息造成死循环）。这也是"在解码器里 `ctx.write` 会跳过自己"的原因。

**经典坑**：
```java
// 错误：在 handler 里用 ctx.writeAndFlush 想把响应经过 encoder
// 实际上 encoder 如果在当前 handler 之后（添加顺序上），它会被跳过！
```
正确做法：确保 encoder 在**添加顺序上位于业务 handler之前**（因为出站是逆序，越早添加越晚执行）。如果顺序搞反，消息会绕过编码器直接写出 → 对端收到裸数据。

## 8.4 `ChannelHandlerContext.executor()`：pipeline 里的"线程边界"

```java
pipeline.addLast(new MyHandler());                              // executor = channel 的 eventLoop
pipeline.addLast(bizGroup, "biz", new MyBlockingHandler());      // executor = bizGroup 里的某个 executor
```

- 传播时每个节点自己判断线程：
```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);              // 直接调用，零开销
    } else {
        executor.execute(() -> next.invokeChannelRead(m));   // 跨线程投递
    }
}
```
**这就是"handler 隔离"的实现原理**：不是特殊机制，只是**每个 context 自带一个 executor**。往前往后传播时，一旦 executor 变了，就自动做线程切换。

**代价（必须能说出来）**：
1. 每次切换 = 入队 + 可能的线程唤醒（`wakeup`）；
2. **消息所有权转移**：跨线程传递 `ByteBuf` 需要注意引用计数（Netty 用 `pipeline.touch()` 记录"当前在哪个 handler"用于泄漏排查）；
3. **顺序性**：同一 Channel 投递到同一 `EventExecutorGroup` 时，Netty 用 `channel.id()` 做 hash 选择 executor，保证**同一 Channel 落到同一 executor**，从而保持串行。

## 8.5 异常传播：`exceptionCaught` 只往后走

```java
// 默认实现（AbstractChannelHandlerContext）
static void invokeChannelRead(...) {
    try { ... } catch (Throwable t) { invokeExceptionCaught(t); }
}

private void invokeExceptionCaught(final Throwable cause) {
    if (executor.inEventLoop()) {
        handler().exceptionCaught(this, cause);
    } else { ... }
}
```

- `exceptionCaught` 是**入站事件**，从当前 ctx **向 tail 方向**传播；
- 若一路没人处理，`TailContext` 会打 WARN 日志（`An exceptionCaught() event was fired, and it reached at the tail of the pipeline`）；
- **注意**：在 `channelRead` 里抛异常，Netty 会自动 `exceptionCaught`，**但不会自动关闭连接**。生产中几乎总需要在 pipeline 末尾加一个统一的异常处理器：
```java
pipeline.addLast(new ChannelInboundHandlerAdapter() {
    @Override public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        log.error("channel error", cause);
        ctx.close();     // 视协议而定：是否关闭
    }
});
```

## 8.6 Handler 生命周期回调顺序（背下来）

```
Channel 注册到 EventLoop：
    handlerAdded → channelRegistered → (若已连接) channelActive
收到数据：
    channelRead → channelReadComplete
写出数据：
    write → flush（或 writeAndFlush）
对端关闭：
    channelInactive → channelUnregistered
异常：
    exceptionCaught
移除 handler：
    handlerRemoved
```

**顺序上的坑**：
- `handlerAdded` 在 `channelRegistered` **之前**；
- `channelActive` 在 TCP 连接**真正建立后**触发（客户端是 `finishConnect` 后，服务端是 accept 后）；
- `channelReadComplete` 是"一次读循环结束"，**不保证**对应一条消息；
- `channelInactive` 之后**不应再 write**（会抛 `ClosedChannelException`），但可以 `close()`（幂等）。

## 8.7 `@Sharable` 与有状态 handler

```java
@ChannelHandler.Sharable
public class MyStatelessHandler extends ChannelInboundHandlerAdapter { ... }
```
- Netty 在 `addLast` 时检查：若 handler 不是 `@Sharable` 且**已经被添加过**，抛 `ChannelPipelineException`；
- 判断依据是 `ChannelHandlerAdapter.added` 字段（第一次添加时置 true，`handlerRemoved` 时清 false）；
- **为什么要有这个注解**：因为 handler 实例可能被成千上万个 Channel 共享，**若有成员变量就是并发写**。`@Sharable` 是"我声明我是线程安全的"承诺，不是"帮我加锁"。

## 8.8 在 handler 中动态增删 handler（易错）

```java
// 错误：直接在 channelRead 里 remove 自己，可能导致当前事件传播断裂
// 正确：让它在本次事件处理完之后生效
ctx.pipeline().remove(this);        // Netty 内部会处理 next 指针，但要注意时序
ctx.pipeline().addLast(new XxxHandler());   // 若在 handlerAdded 里又 add，可能死循环
```

**Netty 的处理方式**：`remove` 时把 `prev.next = next`、`next.prev = prev`，并把被移除的 handler 的 `handlerRemoved` 投递到它的 executor；若当前正在遍历该节点，`next` 已经取到，所以不会 NPE，但**会跳过被移除节点的后续逻辑**。

**`ChannelInitializer` 的正确姿势**：它只执行一次 `initChannel`，执行完后**自动把自己从 pipeline 移除**（`remove(this)` in `initChannel` 的 finally）——这就是为什么它"看起来没占位置"。

---

# 第九部分　ByteBuf 与内存池：Netty 最强的一块

> ★★★★ 如果有人只问一个问题判断你是否真懂 Netty，就问 **"ByteBuf 为什么快？内存池怎么分配的？引用计数怎么保证不泄漏？"**

## 9.1 JDK `ByteBuffer` 的七个硬伤

| # | 问题 | 后果 |
|---|---|---|
| 1 | **读写共用 position/limit** | 每次切换读写都要 `flip()`，忘记就数据错乱 |
| 2 | 无扩容 | `capacity` 固定，不够只能新建 + 拷贝 |
| 3 | 无池化 | 每次 `allocateDirect` 都是一次 `malloc` + 系统调用 |
| 4 | 堆外内存靠 `Cleaner` 回收 | 只有 GC 才释放，容易 OOM 且不可控 |
| 5 | 无引用计数 | 谁该释放、何时释放完全靠约定，容易 double-free |
| 6 | 无复合缓冲 | 多段数据要手动拼接（多一次 memcpy） |
| 7 | 无视图语义 | 不能零拷贝地"切一段"出来共享内存 |

Netty 的 `ByteBuf` 逐条对治：读写指针分离、可扩容（`maxCapacity`）、池化、引用计数、`CompositeByteBuf`、`slice`。

## 9.2 `ByteBuf` 的内存布局：三个索引 + 四个区

```
 0                readerIndex         writerIndex          capacity      maxCapacity
 ├─────────────────┼───────────────────┼─────────────────────┼───────────────┤
 │  已读（可丢弃）  │   可读（未读）     │    可写（空闲）      │  可扩容空间    │
 └─────────────────┴───────────────────┴─────────────────────┴───────────────┘
     discarded         readableBytes        writableBytes      (capacity 未到 max)
```

| 索引 | 含义 | 变化规则 |
|---|---|---|
| `readerIndex` | 下一个**读**的位置 | `readXxx()` 后自增，**不会自动回退** |
| `writerIndex` | 下一个**写**的位置 | `writeXxx()` 后自增 |
| `capacity` | 当前已分配的容量 | 扩容时增加 |
| `maxCapacity` | 上限（默认 `Integer.MAX_VALUE`） | 超过抛 `IndexOutOfBoundsException` |

**四条不变式**：
```
0 <= readerIndex <= writerIndex <= capacity <= maxCapacity
readableBytes() = writerIndex - readerIndex
writableBytes() = capacity - writerIndex
```

## 9.3 读写 API 的语义差异（必须能区分 `get` 与 `read`）

| 类别 | 方法 | 是否移动索引 | 越界检查 |
|---|---|---|---|
| **读（消费）** | `readByte()` / `readInt()` / `readBytes(byte[])` | **移动 readerIndex** | 检查 `readableBytes` |
| **读（不消费）** | `getByte(i)` / `getInt(i)` | **不动** | 检查 `i < writerIndex`（注意不是 capacity） |
| **写** | `writeByte()` / `writeBytes()` | **移动 writerIndex**（必要时扩容） | 检查 `capacity`，不足则 `ensureWritable` |
| **改** | `setByte(i, v)` | **不动** | 检查 `i < capacity` |
| **标记** | `markReaderIndex()` / `resetReaderIndex()` | 保存/恢复 | — |
| **清理** | `clear()` | `readerIndex=writerIndex=0` | 数据不擦除，只是"逻辑清空" |
| **压缩** | `discardReadBytes()` | 把可读部分**前移**到 0 | 有一次 `memcpy`，慎用 |
| **扩容** | `ensureWritable(n)` / `capacity(newCap)` | 可能重新分配 + 拷贝 | — |

**高分细节**：`clear()` 不擦除内存（只是重置指针）—— 所以**敏感数据（密码、密钥）必须显式 `fill(0)` 或 `setZero` 后再释放**，否则池化复用可能把旧数据交给下一个使用者。这是**安全审计会查的点**。

**`discardReadBytes` 为什么不推荐**：它要 `System.arraycopy` 把 `[readerIndex, writerIndex)` 搬到 `0`，O(n)。如果只是为了"腾出前面空间"，更优的是**轮转式索引**（Netty 4.1 内部有 `PooledByteBuf` 的偏移技巧）或直接换新 buffer。

## 9.4 分类矩阵：一张表覆盖所有实现类

| 维度 | 实现类 | 说明 |
|---|---|---|
| **堆内 / 堆外** | `UnpooledHeapByteBuf` / `UnpooledDirectByteBuf` | 堆内是 `byte[]`，堆外是 `ByteBuffer.allocateDirect` |
| **池化 / 非池化** | `PooledByteBuf`（Heap/Direct） / `UnpooledByteBuf` | 池化由 `PooledByteBufAllocator` 分配 |
| **复合** | `CompositeByteBuf` | 逻辑合并多个 ByteBuf |
| **只读** | `readOnlyBuffer()` 返回的包装 | `isReadOnly()` |
| **视图** | `slice` / `duplicate` / `unwrap` | 共享内存，见 9.6 |

**默认分配器**：
```java
// PooledByteBufAllocator 是 4.1 的默认（4.0 是 UnpooledByteBufAllocator）
// 依据：ChannelConfig.DEFAULT_ALLOCATOR
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;
```
**调优参数**（都是系统属性，需在类加载前设置）：

| 参数 | 默认 | 作用 |
|---|---|---|
| `io.netty.allocator.type` | `pooled` | 改为 `unpooled` 可关闭池化（调试用） |
| `io.netty.allocator.numDirectArenas` | `min(2*核数, 8~)` | 直接内存 arena 数量 |
| `io.netty.allocator.numHeapArenas` | 同上 | 堆内存 arena 数量 |
| `io.netty.allocator.pageSize` | 8192 | chunk 内页大小（**必须 2 的幂**） |
| `io.netty.allocator.maxOrder` | 11 | chunkSize = pageSize × 2^maxOrder = 8KB × 2048 = **16MB** |
| `io.netty.allocator.tinyCacheSize` / `smallCacheSize` / `normalCacheSize` | 512 / 256 / 64 | 线程缓存条目数 |

## 9.5 引用计数：Netty 内存管理的"宪法"

### 9.5.1 规则

```java
ByteBuf buf = alloc.buffer();     // refCnt = 1
buf.retain();                     // refCnt = 2
buf.release();                    // refCnt = 1
buf.release();                    // refCnt = 0 → 归还池/释放内存
buf.release();                    // ✗ IllegalReferenceCountException
```

**三条铁律**：

1. **谁创建，谁持有**（refCnt 从 1 开始，创建者负责最终 release）；
2. **谁传递，谁转移所有权**（把 buf 交给 pipeline 的 `fireChannelRead` 后，**不能再用、不能再 release**）；
3. **谁 retain，谁 release**（多一次 retain 就要多一次 release）。

**底层实现**（`AbstractReferenceCountedByteBuf`）：
```java
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater = ...;
private volatile int refCnt = 1;                 // 用 int 而非 AtomicInteger，省对象头

public ByteBuf retain() { return retain0(1); }

private ByteBuf retain0(int increment) {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt == 0) throw new IllegalReferenceCountException(0, increment);
        if (refCntUpdater.compareAndSet(this, refCnt, refCnt + increment)) break;  // CAS 循环
    }
    return this;
}

public boolean release() { return handleRelease(refCntUpdater.getAndDecrement()); }
```
**两个设计细节**：
- 用 `AtomicIntegerFieldUpdater` + `volatile int` 而不是 `AtomicInteger`：**每个 ByteBuf 少一个对象头（16B）**，在百万级 buffer 下是几十 MB 的差别；
- `retain0` 是 CAS 循环而不是 `getAndAdd`：因为**要先检查是否为 0**，防止"释放后又 retain"的 use-after-free。

### 9.5.2 泄漏的典型场景（面试必问）

| 场景 | 原因 | 修复 |
|---|---|---|
| 入站消息没传给下一个 handler，也没 release | `channelRead` 里 `if (...) return;` 忘了 `ReferenceCountUtil.release(msg)` | 用 `SimpleChannelInboundHandler`（**它会自动 release**） |
| 异常分支跳出循环 | `try/catch` 里没 release | `finally { release }` |
| 把 buf 存到成员变量/缓存里，超出生命周期 | 所有权语义被破坏 | 存之前 `retain()`，用完 `release()` |
| 自己 new 的 buf 写完没 release | 以为 Netty 会管 | `writeAndFlush` 成功后 Netty 会 release（**失败也会**），但"没写出去"的要自己管 |
| `CompositeByteBuf` 内部组件的引用 | `addComponent` 转移所有权，`release` composite 会级联 release | 用 `addComponent(true, ...)` 明确语义 |
| 在非 eventLoop 线程持有 buf 太久 | 引用计数跨线程可见但生命周期难控 | 尽快交回 pipeline |

**`SimpleChannelInboundHandler` 的实现（值得记住）**：
```java
public final void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    boolean release = true;
    try {
        if (acceptInboundMessage(msg)) {
            I imsg = (I) msg;
            channelRead0(ctx, imsg);
        } else {
            release = false;
            ctx.fireChannelRead(msg);
        }
    } finally {
        if (autoRelease && release) {
            ReferenceCountUtil.release(msg);      // ★ 自动释放
        }
    }
}
```

### 9.5.3 泄漏检测器 `ResourceLeakDetector`

```bash
-Dio.netty.leakDetection.level=paranoid      # 生产慎用，性能损失大
-Dio.netty.leakDetection.targetRecords=8     # 记录最近几次访问栈
```

| 级别 | 采样率 | 开销 | 用途 |
|---|---|---|---|
`DISABLED` | 0 | 无 | 极致性能 |
`SIMPLE` | **1%**（默认） | 很低 | 生产常态 |
`ADVANCED` | 1%，但记录访问点 | 中 | 排查 |
`PARANOID` | **100%** | 高（每个 buffer 一个 Tracker） | 压测/定位 |

**原理**：
1. 分配时创建一个 `ResourceLeakTracker`（内部 `WeakReference<ByteBuf>` + 一个 `ReferenceQueue`）；
2. `release()` 到 0 时调用 `tracker.close()`，从队列移除；
3. 一个后台/触发式检查扫描 `ReferenceQueue`：**若 buffer 已被 GC 但 tracker 未被 close，说明泄漏** → 打印"最近一次访问的调用栈"。
4. 报错样例：`LEAK: ByteBuf.release() was not called before it's garbage-collected...`（会附带 `Recent access records`）。

> **面试加分**："泄漏检测不是实时监控，它是靠 GC + 弱引用发现的，所以**只有发生 GC 才可能报出来**；如果堆很大、GC 很少，泄漏可能积累到 OOM 才暴露。生产建议在压测阶段用 `PARANOID` 跑一遍。"

## 9.6 派生缓冲区：`slice` / `duplicate` / `retainedSlice`

| 方法 | 共享内存 | 独立读写指针 | 引用计数 | 用途 |
|---|---|---|---|---|
| `slice()` | ✔ | ✔ | **不增加**（共享） | 切一段视图，**不能独立于父 buf 存活** |
| `retainedSlice()` | ✔ | ✔ | **+1**（可独立存活） | 需要把切片传给别的 handler/线程 |
| `duplicate()` | ✔ | ✔ | 不增加 | 复制一个"同内容不同指针"的视图 |
| `readSlice()` | ✔ | ✔ | 不增加 | 读的同时切，父 buf 的 readerIndex 前移 |
| `copy()` | ✗（深拷贝） | ✔ | 新建（refCnt=1） | 需要真正独立的数据 |

**关键区别（面试爱考）**：
```java
ByteBuf a = alloc.buffer(16);
ByteBuf b = a.slice();          // 共享内存，refCnt 仍是 1
b.release();                    // ✗ 会把 a 也释放掉（refCnt 归零）
// 正确写法
ByteBuf c = a.retainedSlice();  // refCnt = 2
c.release();                    // refCnt = 1，a 仍可用
```

## 9.7 `CompositeByteBuf`：逻辑合并 + `writev`

```java
CompositeByteBuf comp = alloc.compositeBuffer();
comp.addComponents(true, header, body);   // true = 自动调整 writerIndex
// 内部：components[] 数组 + 每个 component 的 offset 表
```

**两个价值**：
1. **零拷贝拼接**（避免 `memcpy` 出一个大 buffer）；
2. **底层走 `writev`**：`NioSocketChannel.doWrite` 会把多个 `ByteBuffer` 攒进 `nioBuffers` 数组，一次 `writev` 系统调用发出 → **N 段数据一次 syscall**。

**代价（必须说）**：
- 每次 `get/read` 都要**二分查找**定位 component（`Component.indexOf`，O(log n)）；
- component 数量过多时，`calculateNewCapacity` 和遍历都变慢；
- Netty 提供 `consolidate()`（合并成一块）和 `maxNumComponents`（默认 16）作为保护。

> **实践建议**：header+body 两块用 Composite 很划算；几百个碎片不要用 Composite，直接 `consolidate` 或用 `ByteBufUtil.writeUtf8` 之类。

## 9.8 内存池架构：从 `PoolArena` 到 `PoolSubpage`

> 这是 Netty 里最像"操作系统内存管理"的部分，思想照抄 **jemalloc**。

### 9.8.1 五层结构总览

```
PooledByteBufAllocator
 ├─ PoolArena[heap]   × numHeapArenas     （堆内存竞技场）
 ├─ PoolArena[direct] × numDirectArenas   （堆外竞技场）
 └─ PoolThreadCache   （每个 EventLoop 线程一个，线程本地缓存）

PoolArena
 ├─ tinySubpagePools[32]    （<512B 的小块，按 16B 分级）
 ├─ smallSubpagePools[]     （512B ~ 8KB，按页内切分）
 └─ PoolChunkList × 6       （qInit → q000 → q025 → q050 → q075 → q100，
                              按 chunk 使用率分档，便于找到"最合适的 chunk"）

PoolChunk（默认 16MB = pageSize 8KB × 2^11）
 ├─ memoryMap / pageMap    （位图或完全二叉树，标记哪些 page 空闲）
 ├─ subpages[]             （每个 page 若被切成小块，对应一个 PoolSubpage）
 └─ PoolSubpage
     ├─ bitmap              （long[]，每一位代表一个 elem 是否被占用）
     └─ elemSize / maxNumElems
```

### 9.8.2 四类尺寸，四条分配路径

| 类别 | 大小范围 | 分配路径 |
|---|---|---|
| **Tiny** | < 512B（16B 递增，共 32 档） | `tinySubpagePools[i]` → `PoolSubpage` 位图分配 |
| **Small** | 512B ~ 8KB（pageSize） | `smallSubpagePools[i]` → `PoolSubpage` |
| **Normal** | 8KB ~ 16MB（chunkSize） | `PoolChunk` 直接按 page 数分配（连续 page） |
| **Huge** | > chunkSize | **不进池**，直接 `UnpooledUnsafeDirectByteBuf`（`malloc`） |

> 尺寸分档表由 `SizeClasses` 在类加载时**静态生成**，`PoolArena.size2idx` 查表 O(1)。不同 `pageSize`/`maxOrder` 配置下阈值会变，但"tiny/small/normal/huge"四分法的结构不变。

### 9.8.3 `PoolChunk` 的两代实现（面试能说出来是巨大加分）

**旧实现（完全二叉树 `memoryMap`）**：
- `memoryMap` 是一个 `byte[]`，逻辑上是一棵**完全二叉树**（数组下标 `i` 的左右子是 `2i`、`2i+1`）；
- 每个节点存 `depth`（该节点能分配的**最大层级**）：叶子（page）存 `maxOrder`，根存 0；
- 分配 n 个 page：从根开始，若 `left.depth <= targetDepth` 就往左走（左子树还有空），否则往右 → **O(log n)**；
- 分配成功后把路径上所有节点更新为"子树中最大可用 depth"，释放时反向合并。

**新实现（`pageMap` + `runs`，4.1.7x 之后）**：
- 用 `byte[] pageMap` 记录每个 page 的状态，加上一个 `LongArrayList runs` 记录"连续空闲区间"；
- 目的：**提升缓存局部性、降低内存占用**（memoryMap 每个 page 要一个 int 级别节点，pageMap 每个 page 只 1 字节）；
- 思想不变：**伙伴系统 / 位图 + 连续区间**。

**为什么用"完全二叉树"而不是链表**：因为要快速找到"**连续 n 个 page**"，树结构天然表达"某段是否整体空闲"，且父子关系让合并（释放）变成 O(log n) 的向上回溯。

### 9.8.4 `PoolSubpage`：页内再切（对付 <8KB 的小对象）

```
一个 page（8KB）被切成 maxNumElems = 8192/elemSize 个 elem
bitmap:  long[] ，第 i 位 = 1 表示第 i 个 elem 已被占用

分配：找到第一个为 0 的位 → 置 1 → 返回偏移
释放：清位；若 numAvail == maxNumElems → 整个 page 归还给 chunk
```

**关键优化**：`PoolSubpage` 本身挂在一个**双向链表**上（`prev/next`），同一个 `elemSize` 的所有 subpage 串成链（`tinySubpagePools[i].head`），分配时从链头取 → **O(1)**。

### 9.8.5 `PoolThreadCache`：无锁分配的关键

**问题**：如果每次都去 `PoolArena` 分配，`arena` 上需要 synchronized（虽然有 `PoolThreadCache` 后竞争大大降低）。

**解法**：每个 `EventLoop` 线程绑定一个 `PoolThreadCache`，缓存**最近释放的小块**：

```
MemoryRegionCache:
  tiny  → 每个 size class 一个队列（ArrayDeque<ByteBuf>），缓存 512 个
  small → 缓存 256 个
  normal→ 缓存 64 个
```

- **分配**：先查 `PoolThreadCache` 对应 size class 的队列，命中则**零锁**取出；
- **释放**：优先放回本线程的 cache，满了才归还 `PoolArena`；
- **好处**：**同一线程"分配-释放"闭环，完全无锁、无 syscall**，这是 Netty 高吞吐的核心；
- **代价**：内存"滞留"在线程缓存里，`numDirectArenas` 太多会导致**总内存峰值偏高**（每个 arena 都有自己的缓存池）。

### 9.8.6 一次分配的完整伪代码

```java
ByteBuf allocate(int reqCapacity) {
    // 1. 定位 arena：按当前线程 hash 到某个 arena，保证同一线程尽量用同一 arena
    PoolArena arena = threadArena();

    // 2. 尺寸分档
    int sizeIdx = arena.sizeClassIndex(reqCapacity);   // 查表 O(1)

    if (sizeIdx == -1) {                                // Huge：超过 chunkSize
        return newUnpooledDirect(reqCapacity);          // 不进池，直接 malloc
    }

    // 3. 线程缓存
    ByteBuf cached = threadCache.poll(sizeIdx);
    if (cached != null) return cached;

    // 4. arena 分配
    synchronized (arena) {
        PoolSubpage subpage = arena.tinySubpages[sizeIdx];   // 小块
        if (subpage != null && subpage.numAvail > 0) {
            long handle = subpage.allocate();                // 位图找 0
            return newPooledByteBuf(subpage, handle);
        }
        // 5. 从 chunkList 里找有空闲 page 的 chunk
        for (PoolChunkList list : arena.chunkLists) {
            for (PoolChunk chunk : list) {
                long handle = chunk.allocate(nPages);         // 二叉树/pageMap 查找
                if (handle != -1) {
                    // 6. 若请求小于一个 page，把该 page 变成 subpage 再切
                    PoolSubpage sp = chunk.newSubpage(handle, sizeIdx);
                    return newPooledByteBuf(sp, sp.allocate());
                }
            }
        }
        // 7. 全满 → 新建 chunk（malloc 16MB）
        PoolChunk chunk = newChunk(pageSize, maxOrder, pageShards, ...);
        arena.chunkLists.add(chunk);
        ...
    }
}
```

**面试可以这样总结这一整节**：
> "Netty 的内存池是 jemalloc 的 Java 版：**线程本地缓存消除锁竞争，Arena 分片降低竞争，Chunk 用页 + 伙伴系统管大块，Subpage 用位图管小块，四级尺寸分类保证每个请求都能落到 O(1) 或 O(log n) 的路径上，Huge 直接走 malloc 不进池。** 所以它在高并发下几乎没有分配开销，也不会给 GC 压力。"

## 9.9 ByteBuf 与 GC 的关系（为什么"零 GC"不等于"零内存"）

| 内存类型 | 谁分配 | 谁回收 | GC 是否参与 |
|---|---|---|---|
| 堆内 ByteBuf（池化） | `PoolChunk` 里的 `byte[]` | `release()` 归还池 | **不参与**（数组被池持有，永远可达） |
| 堆外 ByteBuf（池化） | `Unsafe.allocateMemory` | `release()` 归还池 | **不参与**（JVM 不知道这块内存） |
| 非池化堆内 | `new byte[]` | GC | 参与 |
| 非池化堆外 | `ByteBuffer.allocateDirect` + Cleaner | GC + Cleaner | 间接参与（Full GC 才可靠） |

**所以"Netty 零 GC"的准确说法是**：**在网络读写路径上不产生新的堆对象**（缓冲区复用 + 池化），但**不是"不占内存"**，池化内存是常驻的（默认 chunk 16MB × arena 数量，可能几百 MB）。

**压测时常见困惑**："我配了 -Xmx4g，为什么 RSS 到了 8g？" → 因为**堆外池化内存 + 线程栈 + Metaspace + JIT code cache** 都不在 `-Xmx` 里。排查用 `-XX:NativeMemoryTracking=detail` + `jcmd <pid> VM.native_memory summary`。

---

# 第十部分　服务端启动与一次连接的完整生命周期

> ★★★ 面试常问"Netty 启动做了哪些事"、"bind 的时候底层发生了什么 syscall"。能把代码路径和系统调用对齐，就是"真读过"。

## 10.1 一份标准服务端代码（后面所有路径都从这里展开）

```java
EventLoopGroup boss   = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();          // 默认 2*CPU
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(boss, worker)
     .channel(NioServerSocketChannel.class)
     .option(ChannelOption.SO_BACKLOG, 1024)              // 作用于 listen fd
     .option(ChannelOption.SO_REUSEADDR, true)
     .childOption(ChannelOption.TCP_NODELAY, true)        // 作用于 accept 出来的 fd
     .childOption(ChannelOption.SO_KEEPALIVE, true)
     .childHandler(new ChannelInitializer<SocketChannel>() {
         @Override protected void initChannel(SocketChannel ch) {
             ch.pipeline()
               .addLast(new LengthFieldBasedFrameDecoder(1 << 20, 0, 4, 0, 4))
               .addLast(new LengthFieldPrepender(4))
               .addLast(new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS))
               .addLast(new BusinessHandler());
         }
     });
    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    boss.shutdownGracefully();
    worker.shutdownGracefully();
}
```

**注意 `option` vs `childOption`**：前者作用于**监听 socket**（boss 的 channel），后者作用于**每个新连接**（worker 的 channel）。**这是超高频面试点**，答错说明没实际用过。

## 10.2 `bind()` 的完整链路（12 步 + syscall 对照）

```
b.bind(8080)
 │
 ├─① AbstractBootstrap.bind(localAddress)
 │      └─ doBind(localAddress)
 │
 ├─② initAndRegister()
 │      ├─ channelFactory.newChannel()               → 反射 new NioServerSocketChannel()
 │      │     └─ SelectorProvider.openServerSocketChannel()
 │      │           └─ 【syscall】socket(AF_INET, SOCK_STREAM, 0)  ← 得到 listen fd
 │      │
 │      ├─ initChannel(ch)                           → 装 pipeline（ChannelInitializer → initChannel）
 │      │
 │      └─ group.register(channel)                   → MultithreadEventLoopGroup.register
 │            ├─ chooser.next()                      → 选一个 boss EventLoop（round-robin/位运算）
 │            └─ SingleThreadEventLoop.register
 │                  ├─ AbstractChannel.this.eventLoop = eventLoop   ★ 终生绑定
 │                  ├─ 若不在 eventLoop 线程 → eventLoop.execute(() -> register0())
 │                  └─ register0()
 │                        ├─ doRegister()
 │                        │     └─ javaChannel().register(selector, 0, this)
 │                        │           └─ 【syscall】epoll_ctl(ADD, listenFd)  （interestOps=0！）
 │                        ├─ pipeline.fireChannelRegistered()
 │                        └─ (此时 isActive() 为 false，还不 fireChannelActive)
 │
 ├─③ doBind0()  （注册完成后在 eventLoop 线程执行）
 │      └─ pipeline.bind() → head → unsafe.bind()
 │            └─ NioServerSocketChannel.doBind
 │                  ├─ 【syscall】bind(listenFd, 0.0.0.0:8080)
 │                  └─ 【syscall】listen(listenFd, backlog)     ← 开始接受连接
 │
 ├─④ isActive() == true（ServerSocketChannel 已 bound）
 │      └─ pipeline.fireChannelActive()
 │            └─ doBeginRead()
 │                  └─ selectionKey.interestOps |= OP_ACCEPT
 │                        └─ 【syscall】epoll_ctl(MOD, listenFd, EPOLLIN)   ← 真正开始监听
 │
 └─⑤ 返回 ChannelFuture（此时 promise 已完成）
```

**为什么 ② 的 `register` 用 `interestOps=0`，而 ④ 才注册 `OP_ACCEPT`？**
> 因为**顺序问题**。如果注册时直接 `OP_ACCEPT`，可能在 `fireChannelRegistered` / `fireChannelActive` 还没执行完时，事件就已经进来，导致 pipeline 状态不一致（比如 handler 还没 `handlerAdded`）。Netty 的规则是：**先"注册"（结构就绪），再"激活"（开始收事件）**。这也解释了为什么 `doBeginRead` 是在 `channelActive` 里调用的。

## 10.3 accept 之后：新连接怎么进入 worker

```java
// ⑥ boss EventLoop 的 select 返回 OP_ACCEPT
processSelectedKey(k, ch) → unsafe.read() → NioMessageUnsafe.read()
    └─ doReadMessages(readBuf)
          └─ SocketUtils.accept(javaChannel())
                └─ 【syscall】accept4(listenFd, SOCK_NONBLOCK)   ← 一次可能接受多个
    └─ 对每个新 SocketChannel：
          pipeline.fireChannelRead(child)
              └─ ServerBootstrapAcceptor.channelRead()   ← 启动时就被加进 listen 的 pipeline
                    ├─ child.pipeline().addLast(childHandler)     ★ 给子 channel 装业务 pipeline
                    ├─ setChildOptions(child)                     ★ 应用 childOption
                    └─ childGroup.register(child).addListener(...) ★ 注册到 worker
                          └─ 同 register0 流程：doRegister(OP_READ=0) → channelRegistered
                              → 因为是已连接状态，直接 fireChannelActive → doBeginRead(OP_READ)
```

**关键点**：
- `ServerBootstrapAcceptor` 是 Netty 在 `ServerBootstrap.initChannel` 里**自动加到 listen pipeline 的一个内部 handler**；
- **每个新连接都执行一遍 `initChannel`**（`ChannelInitializer` 是靠"每个 channel 一份实例"保证的）；
- 新连接的 accept、pipeline 装配、注册都发生在 **boss 线程**里，只有 `register` 之后的 IO 才在 worker 线程 —— 所以 **boss 线程如果被阻塞（比如 initChannel 里有耗时操作），会拖慢 accept**。这也是"不要在 ChannelInitializer 里做重活"的原因。

## 10.4 客户端 `connect` 的差异（`OP_CONNECT`）

```java
b.connect(host, port) → doResolveAndConnect0
    → channel.connect(remote)
        → NioSocketChannel.doConnect
              → javaChannel().connect(remote)   // 【syscall】connect，非阻塞模式返回 false
    → 若返回 false（连接进行中）：
          doBeginRead() 之前先注册 OP_CONNECT
              → selectionKey.interestOps(OP_CONNECT)
    → select 报 OP_CONNECT
        → processSelectedKey → finishConnect()
              → javaChannel().finishConnect()    // 检查三次握手结果
              → 清除 OP_CONNECT
              → pipeline.fireChannelActive()
```

**面试细节**：非阻塞 `connect` 返回 `false` 表示"**正在连接**"，返回 `true` 表示"**立即成功**"（本机/同网段常见）；失败会抛 `IOException`。`finishConnect()` 才是判断最终成败的地方。

## 10.5 `ChannelOption` 全表（生产配置速查）

| Option | 默认值 | 作用 | 建议 |
|---|---|---|---|
| `SO_BACKLOG` | `NetUtil.SOMAXCONN` | 全连接队列上限 | 与 `somaxconn` 一起调大（如 4096） |
| `SO_REUSEADDR` | `true`（服务端） | 允许绑定 TIME_WAIT 端口、重启不报错 | 保持 true |
| `SO_REUSEPORT` | 需 native epoll | 多进程/多 boss 共享端口 | 多核 accept 扩展 |
| `TCP_NODELAY` | **`true`**（Netty 对 SocketChannel 默认开启） | 关闭 Nagle | 低延迟场景保持 true |
| `SO_KEEPALIVE` | `false` | TCP 层保活（默认 2h） | 建议 true 兜底 |
| `SO_RCVBUF` / `SO_SNDBUF` | 0（系统默认） | 内核收发缓冲区 | 高吞吐链路可显式设置（注意内核会翻倍） |
| `SO_LINGER` | -1（不设置） | close 行为（0=立即 RST） | 一般不要动 |
| `ALLOCATOR` | `PooledByteBufAllocator.DEFAULT` | 缓冲区分配器 | 保持默认 |
| `RCVBUF_ALLOCATOR` | `AdaptiveRecvByteBufAllocator` | 自适应读缓冲 | 保持默认 |
| `WRITE_BUFFER_HIGH_WATER_MARK` | **64KB** | 写队列高水位（触发 `isWritable=false`） | 按业务调整，别太大 |
| `WRITE_BUFFER_LOW_WATER_MARK` | **32KB** | 低水位（恢复可写） | 高低差建议 ≥ 2 倍 |
| `AUTO_READ` | `true` | 是否自动读 | 做背压时设 false |
| `CONNECT_TIMEOUT_MILLIS` | 30000 | 连接超时 | 内网可设 3~5s |
| `ALLOW_HALF_CLOSURE` | `false` | 允许半关闭 | 特殊协议才开 |

---

# 第十一部分　写路径、水位线、编解码、心跳与优雅关闭

## 11.1 `writeAndFlush` 的完整链路（一次写到底发生了什么）

```
channel.writeAndFlush(msg)
 │
 ├─① pipeline.writeAndFlush(msg)              → 从 tail 开始找 outbound
 │
 ├─② 逐个 outbound handler（逆序）
 │      ├─ 若是编码器（MessageToByteEncoder）→ 把 POJO 编成 ByteBuf
 │      └─ 若中途有人在 ctx.write 里拦截 → 可能不再往下走
 │
 ├─③ head.write → unsafe.write(msg, promise)
 │      └─ ChannelOutboundBuffer.addMessage(msg, size, promise)
 │            ├─ 追加到 unflushedEntry 链表尾部
 │            └─ incrementPendingOutboundBytes(size)   ← ★ 水位线判定
 │
 ├─④ head.flush → unsafe.flush() → flush0()
 │      ├─ addFlush()：unflushedEntry 整段变成 flushedEntry（O(1)，不是逐条移交）
 │      ├─ doWrite(outboundBuffer)
 │      │     └─ NioSocketChannel.doWrite0
 │      │           ├─ 把 flushedEntry 的每个 ByteBuf 转成 nioBuffer，攒进 nioBuffers[]
 │      │           ├─ 【syscall】writev(fd, nioBuffers, count)     ← 一次系统调用发多段
 │      │           └─ 写成功的 entry：remove() → ByteBuf.release() + promise.trySuccess()
 │      └─ 若没写完：
 │            ├─ 注册 OP_WRITE（等内核腾出空间）
 │            └─ 记录 totalPendingSize 继续累积
 │
 └─⑤ 返回 ChannelFuture（可能还没完成！）
```

**四个必须理解的点**：

1. **`write` 与 `flush` 分离的意义**：`write` 只是**入队**（用户态操作，无 syscall），`flush` 才**触发写**（syscall）。所以：
```java
// 一次 flush 发 100 条消息
for (...) { ctx.write(msg); }
ctx.flush();                  // 只 1 次 syscall（writev）
```
这是 Netty 最重要的性能技巧之一：**批量合并**。

2. **`addFlush` 是 O(1)**：只移动三个指针（`flushedEntry = unflushedEntry`），不是逐条搬。这让"大量小消息 + 一次 flush"非常高效。

3. **`writev` 依赖 `CompositeByteBuf` 或"多个 entry"**：`doWrite0` 会把多个 entry 的 buffer 攒成一个数组，交给 `writev`。**所以"多段数据"不一定要合并成一块**。

4. **`ChannelFuture` 的完成时机**：`promise` 在**数据真正写到内核发送缓冲区**（`write` 返回）时完成，**不是对端收到**（那需要应用层 ACK）。

## 11.2 `ChannelOutboundBuffer` 的结构（三个指针的链表）

```java
public final class ChannelOutboundBuffer {
    private Entry flushedEntry;      // 待写的第一个
    private Entry unflushedEntry;    // 未 flush 的第一个
    private Entry tailEntry;         // 链表尾

    static final class Entry {
        ByteBuf buf;                 // 消息（或 FileRegion）
        ChannelPromise promise;
        Entry next;
        int size;
    }
}
```

三种状态：
```
addMessage 后：
    flushedEntry = null
    unflushedEntry → [A] → [B] → [C] ← tailEntry

addFlush 后：
    flushedEntry → [A] → [B] → [C] ← tailEntry
    unflushedEntry = null

写完 [A] 后：
    flushedEntry → [B] → [C]
    （[A] 的 buf 被 release，promise 成功）
```

**为什么用链表 + 三个指针**：因为 `write`（入队）和 `flush`（切换）以及 `write`（消费）是三个不同时序的操作，用三个指针可以**各自 O(1)**。

## 11.3 水位线与背压（Netty 的"流控"）

```java
// 写队列累计字节数达到高水位 → isWritable = false
// 降到低水位以下 → isWritable = true
private void incrementPendingOutboundBytes(long size) {
    long newWriteBufferSize = totalPendingSize + size;
    if (newWriteBufferSize >= channel.config().getWriteBufferHighWaterMark()) {
        if (unwritable.compareAndSet(false, true)) {
            pipeline.fireChannelWritabilityChanged();     // ★ 通知业务
        }
    }
}
```

**正确的背压用法**：

```java
// 发送方（比如从 MQ 消费后推送到客户端）
private void push(Channel ch, ByteBuf data) {
    if (!ch.isWritable()) {
        // ① 停止读取上游数据（TCP 层背压）
        ch.config().setAutoRead(false);
        // ② 或者把数据暂存到有界队列，队列满了就阻塞上游
        return;
    }
    ch.writeAndFlush(data);
}

@Override
public void channelWritabilityChanged(ChannelHandlerContext ctx) {
    if (ctx.channel().isWritable()) {
        ctx.channel().config().setAutoRead(true);       // 恢复
    }
}
```

**三个层次的背压（面试可展开）**：

| 层次 | 机制 | 触发点 |
|---|---|---|
| 应用层 | `isWritable()` + `setAutoRead(false)` | 写队列 64KB |
| TCP 层 | 接收窗口 = 0 | 内核 `SO_RCVBUF` 满 |
| 上游 | 停止从 Kafka/RocketMQ 拉取 | 业务判定 |

**常见错误**：只判断 `isWritable` 但不恢复 `autoRead` → 连接永久卡死；或者把水位线设到 100MB → 内存爆了才反应过来。

## 11.4 编解码器家族（先分类，再记名字）

| 方向 | 抽象类 | 作用 |
|---|---|---|
| 入站 | `ByteToMessageDecoder` | **字节 → 消息**（处理半包/粘包） |
| 入站 | `MessageToMessageDecoder<I>` | 消息 → 消息（如 HTTP 对象 → 业务对象） |
| 出站 | `MessageToByteEncoder<I>` | 消息 → 字节 |
| 出站 | `MessageToMessageEncoder<I>` | 消息 → 消息 |
| 双向 | `ByteToMessageCodec` / `MessageToMessageCodec` | 上面两者的组合 |
| 特殊 | `ReplayingDecoder` | 可"重放"的解码器（内部靠异常回滚，性能差，**已不推荐**） |

**开箱即用的常用编解码器**：

| 类 | 用途 |
|---|---|
| `LengthFieldBasedFrameDecoder` / `LengthFieldPrepender` | 长度字段协议（**最通用**） |
| `DelimiterBasedFrameDecoder` | 分隔符（`\r\n`、`\0`） |
| `FixedLengthFrameDecoder` | 定长 |
| `LineBasedFrameDecoder` | 按行 |
| `ProtobufDecoder` / `ProtobufEncoder` / `ProtobufVarint32FrameDecoder` | protobuf |
| `HttpServerCodec` / `HttpObjectAggregator` | HTTP（**聚合器解决 chunked 半包**） |
| `WebSocketServerProtocolHandler` | WebSocket 握手 + 帧编解码 |
| `SslHandler` | TLS（**注意它会让 FileRegion 零拷贝失效**） |
| `ObjectEncoder` / `ObjectDecoder` | JDK 序列化（**禁用于生产**，有反序列化漏洞） |

## 11.5 半包处理的标准写法（可直接用）

```java
public class LengthFieldBasedFrameDecoderExample {
    // 协议：| 4B length (只含 body) | body |
    // 解码后：丢掉 length 字段，只把 body 交给下游
    ChannelPipeline p = ch.pipeline();
    p.addLast(new LengthFieldBasedFrameDecoder(
            1024 * 1024,   // maxFrameLength，防御"声明一个巨大长度"的攻击
            0,             // lengthFieldOffset：长度字段在开头
            4,             // lengthFieldLength：4 字节
            0,             // lengthAdjustment：长度值就是 body 长度
            4              // initialBytesToStrip：丢掉前 4 字节
    ));
    p.addLast(new LengthFieldPrepender(4));   // 出站自动加长度头
}
```

**三个安全要点（面试官很在意）**：
1. **`maxFrameLength` 必须设置**，否则恶意客户端发 `length=2GB` 会让服务端 OOM；
2. `ByteToMessageDecoder` 内部 `cumulation` 会累积直到能解出，攻击者可发大量半包慢慢撑爆 → 需要配合**连接读超时**（`ReadTimeoutHandler`）；
3. 解码出的对象必须**一路传到 tail 或显式 release**，否则泄漏。

**`cumulation` 的两个策略**：
```java
// 默认 MERGE_CUMULATOR：把新数据拷贝进已有 buffer
// 优点：内存连续，读取快；缺点：每次都要 memcpy
static final Cumulator MERGE_CUMULATOR = ...;

// COMPOSITE_CUMULATOR：用 CompositeByteBuf 追加
// 优点：零拷贝；缺点：碎片多时读取要二分查找，且长时间累积会退化
```
**选择建议**：**半包频率低（每次来的数据基本够一帧）用 MERGE；高频半包用 COMPOSITE**。Netty 默认 MERGE，是经过权衡的。

## 11.6 `IdleStateHandler`：心跳与空闲检测的底层

```java
pipeline.addLast(new IdleStateHandler(
    30,    // readerIdleTime：多久没读到数据 → 触发 READER_IDLE
    0,     // writerIdleTime：多久没写出数据 → 触发 WRITER_IDLE
    0,     // allIdleTime：读写都空闲 → 触发 ALL_IDLE
    TimeUnit.SECONDS
));
```

**原理**：

```java
// initialize() 时注册三个定时任务
readerIdleTimeout = schedule(readerIdleTime) → ReaderIdleTimeoutTask
writerIdleTimeout = schedule(writerIdleTime) → WriterIdleTimeoutTask
allIdleTimeout    = schedule(allIdleTime)    → AllIdleTimeoutTask

// 每次读事件（channelRead / channelReadComplete）更新 lastReadTime
// 定时任务 run() 时：
//     if (当前时间 - lastReadTime >= readerIdleTime) → fireUserEventTriggered(READER_IDLE)
//     else → 重新 schedule（剩余时间）
```

**关键行为（易错）**：
1. **它不关心消息内容**，任何读事件都会重置 `lastReadTime` —— 所以**心跳包必须由对端真实发出**，否则检测不到"对端已死"。
2. **触发的是 `userEventTriggered`**，不是异常，业务要自己处理：
```java
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent) {
        IdleStateEvent e = (IdleStateEvent) evt;
        if (e.state() == IdleState.READER_IDLE) {
            ctx.close();           // 或发送探测包 + 计数
        }
    }
}
```
3. **`writerIdleTime` 不能替代心跳发送**：它只是"你没写数据就通知你"，业务收到通知后要**主动写心跳**。
4. **不要设得太短**：GC 停顿、网络抖动都会造成误判；通常 30~60s，然后连续 2~3 次才断开。

## 11.7 客户端断线重连（一个正确的姿势）

```java
public class ReconnectHandler extends ChannelInboundHandlerAdapter {
    private final Bootstrap bootstrap;
    private final EventLoopGroup group;
    private final String host; private final int port;
    private volatile ChannelFuture future;

    @Override public void channelInactive(ChannelHandlerContext ctx) {
        // 指数退避 + 抖动，避免"雪崩式重连"
        long delay = Math.min(30, 1 << retryCount) * 1000L;
        group.schedule(() -> doConnect(), delay, TimeUnit.MILLISECONDS);
    }

    private void doConnect() {
        future = bootstrap.connect(host, port);
        future.addListener(f -> {
            if (!f.isSuccess()) {
                // 连接失败也要重连，否则首次失败就再也不试了
                scheduleRetry();
            }
        });
    }
}
```

**三个坑**：
1. **`channelInactive` 里不能用原来的 `ctx` 重连**（已关闭），要用 bootstrap 新建；
2. **必须加退避 + 随机抖动**，否则服务端故障恢复瞬间会被重连风暴打挂；
3. **重连期间要清理旧的 pipeline 状态**（比如未完成的请求），否则内存泄漏。

## 11.8 优雅关闭的正确顺序

```java
// 1. 停止接受新连接
bossGroup.shutdownGracefully(0, 10, TimeUnit.SECONDS).sync();
// 2. 等待存量请求处理完（业务自定义：等待在途请求计数归零）
awaitInflightRequestsZero(30_000);
// 3. 关闭 worker（会 close 所有 channel，触发 channelInactive）
workerGroup.shutdownGracefully(2, 15, TimeUnit.SECONDS).sync();
```

**为什么先关 boss**：如果先关 worker，listen fd 还在 accept，新连接会进来但没有 worker 处理 → 客户端连上就超时。**先关 boss（listen fd 关闭，内核回 RST/不响应），再关 worker，是标准顺序。**

---

# 第十二部分　性能调优、故障排查与常见认知误区

## 12.1 三层调优清单

### JVM 层

| 参数 | 建议 | 说明 |
|---|---|---|
| `-Xms = -Xmx` | 必须 | 避免堆动态扩缩引起停顿 |
| `-XX:+UseG1GC` / ZGC | 大堆用 G1/ZGC | 减少 STW，避免心跳误判 |
| `-XX:MaxDirectMemorySize` | 显式设置（如 2g） | 否则默认等于 Xmx，容易和堆一起把机器打爆 |
| `-XX:NativeMemoryTracking=detail` | 排查期开启 | 看堆外内存 |
| `-Dio.netty.leakDetection.level=simple` | 生产默认 | 定位泄漏时临时改 paranoid |
| `-Dio.netty.noKeySetOptimization=true` | JDK17+ 反射失败时 | 或加 `--add-opens` |
| `-XX:+AlwaysPreTouch` | 大内存服务可选 | 启动时预touch，减少运行时缺页 |

### Netty 层

| 项 | 建议 |
|---|---|
| worker 线程数 | 默认 2×CPU，IO 密集可保持；有阻塞任务要隔离而非加线程 |
| `SO_BACKLOG` | 与 `somaxconn` 对齐（如 4096/8192） |
| 写水位线 | 默认 64K/32K 起步，按消息大小调（**小于单条消息会频繁触发**） |
| `maxCumulationBufferComponents` | 默认 16，长连接小包多可调 1024 |
| 定时任务 | 海量超时用 `HashedWheelTimer`，不要都塞 EventLoop |
| 日志 | `channelRead` 里**禁止**打 debug 日志（每包一行会拖垮 IO 线程） |

### OS 层

```bash
# /etc/sysctl.conf
net.core.somaxconn = 8192                 # accept 队列上限
net.ipv4.tcp_max_syn_backlog = 8192       # 半连接队列
net.ipv4.ip_local_port_range = 1024 65535 # 客户端端口范围
net.ipv4.tcp_tw_reuse = 1                 # 允许复用 TIME_WAIT（outbound）
net.core.netdev_max_backlog = 5000        # 网卡 ring 溢出前的 backlog
net.ipv4.tcp_rmem = 4096 87380 6291456    # 接收缓冲区 min/default/max
net.ipv4.tcp_wmem = 4096 16384 4194304
fs.file-max = 1000000                     # 全局 fd 上限
# ulimit -n 1000000                       # 进程 fd 上限（必须和 file-max 一起调）
```

## 12.2 故障排查的"证据链"（按现象反推）

| 现象 | 最可能原因 | 验证命令 |
|---|---|---|
| 客户端"连上但无响应" | accept 队列满（`SO_BACKLOG` 太小） | `ss -lnt` 看 Recv-Q；`netstat -s \| grep overflow` |
| 偶发 40ms 延迟 | Nagle + 延迟 ACK | `tcpdump` 看小包间隔；确认 `TCP_NODELAY` |
| CPU 100% 但 QPS 很低 | epoll 空轮询 / 忙等 | `jstack` 看是否在 `select`；看 `selectCnt` 日志 |
| 内存持续上涨、GC 正常 | ByteBuf 泄漏 / 直接内存泄漏 | `-Dio.netty.leakDetection.level=paranoid`；`jcmd VM.native_memory` |
| `Too many open files` | fd 上限 | `ulimit -n`、`lsof -p` |
| `Cannot assign requested address` | 本地端口耗尽（TIME_WAIT 太多） | `ss -s` 看 timewait 数量 |
| 大量 CLOSE_WAIT | **代码没 close** | `ss -tan state close-wait` + 查代码 |
| 吞吐上不去、CPU 不高 | 窗口太小 / RTT 高 / 并发不足 | `ss -ti` 看 cwnd、rtt；算 Little's Law |
| 连接莫名被 RST | `SO_LINGER=0` / 对端异常关闭 / backlog 溢出 | `tcpdump` 看 RST 的 seq |
| 写操作越来越慢 | 对端不读，写队列堆积 | `channel.isWritable()` 监控 |

## 12.3 十二个高频认知误区（面试反问时会暴露）

| # | 误区 | 正解 |
|---|---|---|
| 1 | "Netty 是异步 I/O" | 编程模型异步；内核是**同步多路复用**（epoll 就绪后仍要自己 read） |
| 2 | "Netty 定时任务用时间轮" | EventLoop 用 **PriorityQueue**；`HashedWheelTimer` 是独立工具 |
| 3 | "ByteBuf 不用管，GC 会回收" | **必须 release**，否则泄漏；池化内存 GC 完全看不到 |
| 4 | "`writeAndFlush` 在调用线程执行" | 会 `executor.execute` **切到 eventLoop 线程** |
| 5 | "handler 里要加锁保护状态" | 同一 Channel 的事件**串行**，无需锁；跨 Channel 共享才需要 |
| 6 | "boss 线程越多 accept 越快" | 一个 listen fd 只绑一个 EventLoop；要扩展得用 `SO_REUSEPORT` |
| 7 | "关掉 Nagle 一定更快" | 可能造成小包风暴；应配合应用层批量写 |
| 8 | "池化内存不受 -Xmx 限制，可以随便用" | 受 `io.netty.maxDirectMemory` 约束，超了照样 OOM |
| 9 | "`SO_BACKLOG` 越大越好" | 受 `somaxconn` 限制，且**掩盖了 accept 慢的真实问题** |
| 10 | "`autoRead=false` 就是背压" | 必须配合 `isWritable()` 与恢复逻辑，否则连接卡死 |
| 11 | "零拷贝 = 不占内存" | 零拷贝是**不拷贝**，内存仍然占用（page cache / direct buffer） |
| 12 | "Netty 5 性能更好" | **Netty 5 已废弃**（ForkJoin 重构失败），4.1 仍是主流，4.2 是渐进升级 |

---

# 第十三部分　一线大厂面试题库（含难度标注 + 深度解答 + 追问链）

> **说明**：以下题目来自公开面经、技术社区讨论与作者对同类岗位面试的归纳，**不代表任何公司官方题库**；标注的"公司"是"该题在某类公司中高频出现"的归类，用于帮你按风格准备，而不是断言某公司必考。
>
> **难度定义**：
> - ★：**筛人题**——答不出直接挂（基础概念）
> - ★★：**及格题**——校招/初级必答
> - ★★★：**区分题**——中级/高级的分界线
> - ★★★★：**深挖题**——能答到源码/内核层，直接定级高级
> - ★★★★★：**架构题**——需要工程经验 + 跨层推理，通常是"专家岗/架构岗"的题

## 13.0 高频题总览（先自测，再去看详解）

| # | 题目 | 难度 | 高频公司 | 一句话答案锚点 |
|---|---|---|---|---|
| 1 | Netty 为什么快 / 解决了 NIO 什么问题 | ★★ | 全部 | 池化+无锁串行+批量syscall+零拷贝 |
| 2 | Netty 线程模型（主从 Reactor） | ★★★ | 阿里/字节/腾讯 | boss accept、worker 读写、单 Channel 单 EventLoop |
| 3 | EventLoop 的 run 循环做了什么 | ★★★★ | 阿里/美团 | select → processSelectedKeys → runAllTasks |
| 4 | select 阻塞时怎么被任务唤醒 | ★★★★ | 字节/阿里 | selector.wakeup → eventfd 写 1 字节 |
| 5 | epoll 原理，LT 与 ET | ★★★★ | 字节/百度/腾讯 | 红黑树 + 就绪链表 + 回调 |
| 6 | Netty 用 LT 还是 ET | ★★★★ | 字节 | JDK 是 LT；Netty 用"读满再读"模拟 ET |
| 7 | 粘包/半包的原因与解法 | ★★ | 全部 | TCP 字节流，无消息边界 |
| 8 | `LengthFieldBasedFrameDecoder` 五个参数 | ★★★ | 美团/滴滴 | offset/len/adjust/strip/max |
| 9 | 零拷贝（内核 vs Netty） | ★★★ | 字节/华为 | sendfile vs Composite/slice |
| 10 | ByteBuf 与 ByteBuffer 区别 | ★★ | 京东/腾讯 | 读写指针分离、池化、引用计数 |
| 11 | 内存池 PoolArena/Chunk/Subpage | ★★★★ | 阿里/字节 | jemalloc 思想，四级尺寸 |
| 12 | 引用计数与泄漏排查 | ★★★ | 阿里/字节 | refCnt CAS，ResourceLeakDetector |
| 13 | Pipeline 事件传播顺序 | ★★★★ | 腾讯/网易 | 入站 next、出站 prev |
| 14 | `ctx.write` 与 `channel.write` 区别 | ★★★★ | 腾讯/阿里 | 起点不同，ctx 跳过自己 |
| 15 | 什么时候注册 OP_WRITE | ★★★★ | 字节/快手 | 只在写不动时注册，写完注销 |
| 16 | 水位线与背压 | ★★★★ | 美团/拼多多 | 64K/32K，isWritable + autoRead |
| 17 | 服务端 bind 的完整流程 | ★★★★ | 阿里/华为 | socket→bind→listen→epoll_ctl |
| 18 | 为什么注册时 interestOps=0 | ★★★★ | 阿里 | 先注册后激活，避免事件早到 |
| 19 | 心跳与空闲检测 | ★★ | 全部 | IdleStateHandler + userEvent |
| 20 | 优雅关闭 | ★★★ | 腾讯/滴滴 | 先 boss 后 worker，quietPeriod |
| 21 | 单机百万连接怎么做到 | ★★★★★ | 字节/快手 | fd 上限+内存+多 EventLoop+协议精简 |
| 22 | 业务阻塞了 IO 线程怎么办 | ★★★★ | 阿里/美团 | EventExecutorGroup 隔离 |
| 23 | Netty 的 Future/Promise | ★★★ | 京东 | 异步结果 + 监听器 + 线程调度 |
| 24 | 如何保证消息有序 | ★★★ | 京东/滴滴 | 单 Channel 串行 + 序列号 |
| 25 | 定时任务用时间轮吗 | ★★★ | 京东 | 不是，用 PriorityQueue |
| 26 | FastThreadLocal 为什么快 | ★★★★ | 阿里/字节 | 数组下标 vs 哈希 |
| 27 | 空轮询 Bug 与 512 次重建 | ★★★★ | 阿里/百度 | JDK bug + rebuildSelector |
| 28 | Netty 4/5 演进与 native transport | ★★★ | 华为/百度 | 4.1 主流，5 废弃，epoll transport |

## 13.1 阿里巴巴 / 蚂蚁集团（偏"原理深度 + 源码细节"）

### 题 1.1　Netty 的线程模型是怎样的？为什么是"主从 Reactor"？
**难度**：★★★　**类型**：区分题　**考察点**：Reactor 演进、boss/worker 分工、单 Channel 绑单 EventLoop

**答题主线**（先结论，再展开，最后回到"为什么"）：
1. **结构**：`bossGroup` 只监听 `OP_ACCEPT`，`workerGroup` 负责 `OP_READ/OP_WRITE`；两者都是 `NioEventLoopGroup`，每个 `EventLoop` 是一条独立的 `epoll` 线程。
2. **绑定关系**：一个 `Channel` 注册时通过 `chooser` 选一个 `EventLoop`，**终生绑定**；一个 `EventLoop` 可以绑成千上万个 `Channel`。
3. **执行模型**：同一 Channel 的所有 handler 回调都在其 EventLoop 线程内**串行**执行，因此 pipeline 内无并发。
4. **为什么主从**：accept 与 read/write 的负载特征不同（accept 是突发、低频；read/write 是持续、高频），分离后互不干扰；且 accept 本身是"单点"，不需要多线程（一个 listen fd 只能在一个 selector 上）。
5. **线程数**：`DEFAULT_EVENT_LOOP_THREADS = max(1, CPU×2)`；boss 通常给 1。

**常见追问**：
- **Q：为什么是 CPU×2 而不是 CPU×1？**
  A：IO 线程在 `epoll_wait` 时是睡眠的，不是满负荷；留一倍余量应对突发、少量阻塞和 GC 停顿。再多会引入上下文切换与 cache 失效，收益递减。**注意这是默认值，不是最优值**，要按压测调。
- **Q：多个 boss 线程有用吗？**
  A：单 listen fd 只绑一个 EventLoop，多余 boss 线程在"单个 ServerBootstrap"下是浪费。要真正并行 accept，得用 `SO_REUSEPORT` 起多个 `ServerBootstrap`（每个有独立 listen fd）。
- **Q：一个 Channel 能不能换 EventLoop？**
  A：不能。`AbstractChannel.eventLoop` 在 `register` 时赋值，之后没有 setter；想"迁移"只能关掉重建。

**延伸**：这套模型对应 **Actor 模型**（Channel = Actor，EventLoop = 调度器），也是 **Nginx 多进程 + SO_REUSEPORT** 的线程版。理解"亲和性替代锁"这个思想，就能理解为什么 Netty 的 handler 可以随便用普通字段。

---

### 题 1.2　`NioEventLoop.run()` 到底做了什么？没有任务时为什么不会空转？
**难度**：★★★★　**类型**：深挖题（阿里最爱问这种"源码时间线"）

**答题主线**：按"一轮循环的四个阶段"讲，并强调**每个阶段的触发条件与代价**。

1. **决策阶段**：`selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())`
   - 有任务 → `selectNow()`（非阻塞，顺手取事件）
   - 无任务 → `SELECT`（走阻塞）
2. **计算 deadline**：`nextScheduledTaskDeadlineNanos()`，取最近一个定时任务的到期时间；没有则为 `NONE`（`Long.MAX_VALUE`）。
3. **阻塞 select**：
   - 有定时任务 → `select(timeoutMillis)`，超时即最近任务到期时间；
   - 无定时任务 → `selector.select()` **无限阻塞**。
4. **处理 IO**：`processSelectedKeys()` → 分发到 `unsafe.read()` / `forceFlush()` / `finishConnect()`。
5. **处理任务**：`runAllTasks()`，从 `taskQueue` 取，受 `ioRatio` 时间预算约束（每 64 个任务检查一次时间）。
6. **循环尾部**：关闭检查、空轮询检测。

**为什么不会空转**：
- 无任务时是**真阻塞在 epoll_wait**（不是 `while(true)` 忙等）；
- 有任务提交时会 `wakeup()`，往 eventfd 写 1 字节，`epoll_wait` 立即返回；
- **两级唤醒优化**：`nextWakeupNanos`（AtomicLong）先判断 loop 是否真的在睡，避免每次写都做一次 `eventfd` syscall。

**常见追问**：
- **Q：如果 select 一直不返回，定时任务怎么执行？**
  A：不会 —— deadline 就是"最近一个定时任务的到期时间"，select 超时后自然醒来执行。这是"用超时时间实现定时调度"的经典做法（**把定时任务变成 select 的超时**）。
- **Q：`runAllTasks` 会不会饿死 IO？**
  A：不会，因为有时间预算 `ioTime * (100 - ioRatio) / ioRatio`；但 `ioRatio=100` 时会（任务独占）。
- **Q：`ioRatio` 怎么调？**
  A：默认 50。纯转发调大（80），任务多（推送/定时）调小（30）。

**延伸**：把这个循环和 **Redis 的 `aeMain`** 对比：Redis 是 `aeApiPoll` + `processTimeEvents` + `beforeSleep`，结构几乎一样。**所有单线程事件循环本质都是"select + 任务队列"**。

---

### 题 1.3　一个 EventLoop 绑了几千个 Channel，怎么保证线程安全？
**难度**：★★★　**类型**：区分题（答"加锁"就输了）

**答题主线**：
1. **不用保证"同一个 Channel 内"的线程安全**——因为它的事件永远在同一个 EventLoop 线程串行执行，**天然的 happens-before**（同一线程的程序顺序）。
2. **需要保证的是"跨 Channel 共享状态"的安全**——比如一个 `@Sharable` 的计数器 handler 被 1000 个 Channel 共用，那必须用 `LongAdder`/`AtomicInteger`/锁。
3. **`Channel` 的公开 API 是线程安全的**：`writeAndFlush` 内部会判断 `executor.inEventLoop()`，不在就 `execute()` 投递，所以你可以从任意线程写。
4. **`ChannelHandlerContext` 的方法不是线程安全的**：`ctx.write` 假定你在 eventLoop 内（`invokeWrite` 直接操作 outbound buffer 的链表指针），跨线程调用可能破坏链表。

**常见追问**：
- **Q：那我在业务线程池里拿 `ctx.write` 会怎样？**
  A：可能破坏 `ChannelOutboundBuffer` 的链表结构（entry 的 next 指针竞争），轻则消息错乱，重则丢消息。正确做法是 `ctx.channel().writeAndFlush(...)`（走到 channel 级 API 会调度）或 `ctx.executor().execute(() -> ctx.write(...))`。
- **Q：pipeline 里 handler 的成员变量要不要 volatile？**
  A：同一 Channel 内不需要；被多 Channel 共享（`@Sharable`）则需要。

**延伸**：这就是 **"serialized execution"（串行化执行）** 设计模式，与 **Disruptor 的"单线程批处理"**、**Actor 的 mailbox** 同源。代价是"一个 handler 阻塞会拖住整条 EventLoop"，收益是"零锁"。

---

### 题 1.4　讲一下 Netty 的内存池设计，`PoolArena` / `PoolChunk` / `PoolSubpage` 分别解决什么问题？
**难度**：★★★★　**类型**：深挖题（蚂蚁/阿里 P7+ 高频）

**答题主线**（按"分层解决什么"讲，而不是背类名）：

| 组件 | 解决的问题 | 关键机制 |
|---|---|---|
| `PooledByteBufAllocator` | 全局入口 | 持有 N 个 Arena + 每个线程一个 ThreadCache |
| `PoolThreadCache` | **锁竞争** | 线程本地缓存 tiny/small/normal 三种队列，分配释放零锁 |
| `PoolArena` | **多线程竞争 + 尺寸分档** | 每个线程按 hash 固定到一个 arena；内部 tiny/small/normal/huge 四类，查表 O(1) |
| `PoolChunkList` | **碎片与利用率** | 6 条链（qInit/q000/q025/q050/q075/q100）按使用率分档，优先在"半满"的 chunk 里分配 |
| `PoolChunk` | **大块连续分配** | 16MB（pageSize 8K × 2^11）；旧版用完全二叉树 `memoryMap`，新版用 `pageMap`+`runs` |
| `PoolSubpage` | **小块内部碎片** | 一个 page 切成 N 个 elem，用 `long[] bitmap` 标记占用，同尺寸串成链，O(1) 分配 |

**讲清"一次分配的四条路径"**：
1. 查 ThreadCache → 命中即返回（**零锁**）；
2. 未命中 → 进 arena，查 tiny/small subpage 链；
3. 还没有 → 遍历 chunkList 找有空闲 page 的 chunk，分配 page，转成 subpage 再切；
4. 全满 → `newChunk()`（16MB malloc）；
5. **超过 chunkSize（Huge）→ 不进池，直接 unpooled 分配**。

**常见追问**：
- **Q：为什么用完全二叉树管 chunk？**
  A：因为要快速找到"连续 N 个 page"。树的每个节点存"该子树最大可分配层级"，从根往下判断左右子树能否满足 → O(log n)；释放时向上合并 → 天然的伙伴系统。
- **Q：新版为什么改成 `pageMap`+`runs`？**
  A：`memoryMap` 每个节点占空间大、访问跳跃，cache 不友好；`pageMap` 每 page 1 字节，配合 runs 记录连续区间，缓存局部性更好、内存更省。
- **Q：ThreadCache 会不会导致内存滞留？**
  A：会。每个线程缓存最多 512/256/64 个块，arena 多、线程多时"滞留内存"很可观。这就是"池化后 RSS 比预期高"的原因之一，可通过 `-Dio.netty.allocator.tinyCacheSize` 等调小。
- **Q：为什么不全部用 ThreadCache？**
  A：ThreadCache 只缓存"刚释放的小块"，大块（normal）缓存有限，且跨线程释放（A 线程分配、B 线程释放）会归还 arena，需要 arena 兜底。

**延伸**：这套设计几乎逐条对应 **jemalloc 的 arena / bin / run / slab** 概念（Netty 官方文档也明确说"借鉴 jemalloc"）。理解 jemalloc 的三个目标——**避免锁竞争、减少碎片、提高缓存局部性**——就能自己推导出 Netty 的每个设计。

---

### 题 1.5　ByteBuf 的引用计数是怎么实现的？线上内存泄漏怎么定位？
**难度**：★★★　**类型**：工程题（阿里喜欢"出问题你怎么查"）

**答题主线**：

**实现层**：
- `AbstractReferenceCountedByteBuf` 用 `volatile int refCnt = 1` + `AtomicIntegerFieldUpdater` 做 CAS；
- 不用 `AtomicInteger` 是为了**省对象头（每 buffer 省 16B）**；
- `retain0` 是 CAS 循环而非 `getAndAdd`，因为**必须先判断 refCnt 是否为 0**（防止 use-after-free）；
- `release()` 归零时调 `deallocate()`：池化 → 归还 arena/subpage；非池化堆外 → `Unsafe.freeMemory`。

**排查层**（这是加分部分）：
1. **开检测**：`-Dio.netty.leakDetection.level=paranoid`（压测环境），生产用 `simple`（1% 采样）；
2. **看日志**：`LEAK: ByteBuf.release() was not called before it's garbage-collected`，会打印**最近 8 次访问的调用栈**（`targetRecords`）；
3. **常见泄漏点**：
   - `channelRead` 里提前 `return` 没 `release` → 换 `SimpleChannelInboundHandler`（自动释放）；
   - 异常分支跳出；
   - 把 `ByteBuf` 存进 map/队列，生命周期超出；
   - 自己 `alloc.buffer()` 但写失败/没写；
   - `CompositeByteBuf.addComponent` 后重复 release。
4. **兜底手段**：`jcmd <pid> VM.native_memory summary`（看 direct memory 是否持续涨）；`-Dio.netty.maxDirectMemory` 设上限让它"早死早超生"，而不是拖垮整机。

**常见追问**：
- **Q：为什么泄漏检测要靠 GC？**
  A：因为它用 `WeakReference` + `ReferenceQueue` 判断"对象已被回收但 tracker 没 close"。**没有 GC 就不会报**，所以堆大、GC 少时泄漏可能积累很久。
- **Q：池化内存泄漏和堆内存泄漏怎么区分？**
  A：看 `jmap -heap`（堆正常）+ NMT 的 `Direct`/`Internal` 持续增长 → 池化/直接内存泄漏；堆也涨 → 普通对象泄漏。
- **Q：`retain()` 了但忘记 `release()`，检测能发现吗？**
  A：**不能**。因为对象仍被引用（refCnt>0），GC 不会回收它，tracker 不会触发。这种情况只能靠代码审查 + 监控"池化内存使用率"。

**延伸**：引用计数本质是**手动内存管理**（类似 C++ shared_ptr），它把"GC 的不确定性"换成了"代码正确性的要求"。这是 Netty 用"零 GC"换"必须 release"的**根本权衡**，也是面试里可以展开的架构思考。

---

### 题 1.6　服务端 `bind()` 的过程中，Netty 做了哪些事？底层有哪些系统调用？
**难度**：★★★★　**类型**：深挖题（阿里/华为都爱）

**答题主线**：按 10.2 的 12 步讲，重点是**三个"反直觉"的点**：

1. **`register` 时 `interestOps = 0`，不是 `OP_ACCEPT`** —— 因为要"先注册（结构就绪）后激活（开始收事件）"，否则事件可能在 pipeline 还没装好时就到达。
2. **`bind` 和 `listen` 是两次 syscall** —— Java 的 `ServerSocketChannel.bind()` 内部先 `bind` 再 `listen`，`listen` 的第二个参数就是 `SO_BACKLOG`。
3. **`channelActive` 之后才 `doBeginRead` 注册 `OP_ACCEPT`** —— 这是"结构完整后才对外服务"的体现。

**syscall 对照表**（必须能背）：

| 阶段 | syscall |
|---|---|
| newChannel | `socket(AF_INET, SOCK_STREAM, 0)` |
| register | `epoll_ctl(ADD, listenFd, interestOps=0)` → 实际可能延迟到首次 MOD |
| bind | `bind(fd, addr)` |
| listen | `listen(fd, backlog)` |
| 激活 | `epoll_ctl(MOD, fd, EPOLLIN)` |
| accept | `accept4(fd, SOCK_NONBLOCK)` |
| 读 | `read(fd, buf, n)` |
| 写 | `writev(fd, iov, n)` |

**常见追问**：
- **Q：为什么用 `accept4` 而不是 `accept`？**
  A：`accept4` 可以原子地设置 `SOCK_NONBLOCK`，避免"accept 后再 fcntl 设置非阻塞"之间的竞态窗口（那个窗口里如果对端发数据，可能阻塞）。
- **Q：`SO_BACKLOG` 满了会怎样？**
  A：Linux 默认丢弃新完成握手的连接（客户端以为连上了），如果 `tcp_abort_on_overflow=1` 则直接 RST。表现为"偶发连接超时"。
- **Q：`bind` 之前能不能先 `epoll_ctl`？**
  A：技术上可以（fd 已存在），但 Netty 故意不在注册阶段设 `OP_ACCEPT`，因为还没 `listen`，即使可读也无意义，且会让 pipeline 状态不一致。

**延伸**：把这套流程和 **JDK 原生 `ServerSocket`** 对比：JDK 的 `bind` 是阻塞的、`accept` 是阻塞的，Netty 把它们都改成了"注册事件 + 回调"，这是"同步 API → 异步模型"的转换成本所在。

---

### 题 1.7　百万连接/十万连接的场景，你会怎么设计和调优？
**难度**：★★★★★　**类型**：架构题

**答题主线**：分四层答（**这是架构题的标准答法：分层 + 量化**）。

**① 连接层（fd）**
- 每连接 1 个 fd（服务端 accept 出来的），10w 连接 = 10w fd；
- `ulimit -n 1000000` + `fs.file-max`；`/proc/sys/fs/nr_open`；
- 端口不是问题（服务端固定端口，客户端才吃端口）。

**② 内存层（每连接成本）**
- 内核：每个 socket 有 `struct sock`（约 1~2KB）+ 收发缓冲区（`SO_RCVBUF`/`SO_SNDBUF`，默认可能 128KB 起）；
- 用户态：`Channel` + `pipeline` + `ByteBuf`（**空连接不分配 buffer**，Netty 惰性分配）+ `SelectionKey`；
- **粗算**：10w 连接 × （2KB 内核结构 + 64KB 接收缓冲）= 6.4GB → **必须调小 `SO_RCVBUF`**（如 16KB）；
- 100w 连接必须用 **epoll + 极简协议 + 小缓冲**，且通常要"连接与业务分离"（网关只做转发）。

**③ 线程层**
- worker = 2×CPU（如 32 核 → 64 线程），每线程一个 epoll，绑 1~2w 连接；
- **不要用"一个连接一个线程"**（100w 线程 = 栈内存 100w×1MB = 1TB）。

**④ 事件层**
- 用 **native epoll transport**（`EpollEventLoopGroup`）减少 JDK 封装开销；
- `SO_REUSEPORT` + 多 boss 做 accept 扩展；
- 心跳要"分批+抖动"（避免同一秒 100w 个心跳包同时到）；
- 用 `AUTO_READ=false` + 背压，防止慢客户端拖垮内存。

**常见追问**：
- **Q：100w 连接时，select 一次要遍历 100w 吗？**
  A：**不用**。epoll 只返回就绪的（O(k)），这就是 epoll 相对 select 的核心优势。但要注意 `processSelectedKeys` 的数组遍历量和 `SelectionKey` 数量无关。
- **Q：`SO_RCVBUF` 设多小合适？**
  A：与业务有关。心跳型长连接（几乎没数据）可以 4~16KB；有实际流量的至少 64KB。**太小会导致窗口受限、吞吐下降**，要压测。
- **Q：怎么监控"连接数是否到瓶颈"？**
  A：`ss -s`（total sockets）、`cat /proc/net/sockstat`、应用层 `channelGroup.size()`、fd 使用率。

**延伸**：这就是 **C100K/C1000K** 问题。经典参考是 2012 年 Errata Security 的"1M connections on a single server"（用 epoll + 小内存），核心结论：**瓶颈不是 CPU，是内存（每连接的内核缓冲区）和 fd 上限**。

---

### 题 1.8　Netty 的 `ChannelFuture` / `Promise` 机制是怎么设计的？
**难度**：★★★　**类型**：区分题

**答题主线**：
1. **`Future` 是只读**（`isDone`/`isSuccess`/`get`/`addListener`），**`Promise` 是可写**（`setSuccess`/`setFailure`）；
2. `ChannelFuture` = `Channel` 的异步操作结果；`ChannelPromise` = 可写的 `ChannelFuture`；
3. **监听器机制**：`addListener` 时，若已完成则**立即在调用线程执行**（`notifyListeners`），否则在完成时执行；执行线程由 `executor` 决定（通常回到 eventLoop）；
4. **`sync()` 会阻塞调用线程**（用 `Object.wait`/`LockSupport`），**只能在非 eventLoop 线程用**，否则死锁（因为完成通知需要 eventLoop 线程跑）。
5. **`await()` 与 `sync()` 的区别**：`sync()` 失败会抛异常，`await()` 只返回 future。

**常见追问**：
- **Q：为什么不能在 eventLoop 线程里 `sync()`？**
  A：因为 `sync()` 等待的"完成事件"恰恰需要这个 eventLoop 线程去执行 → **自己等自己**，永久阻塞（Netty 会抛 `BlockingOperationException` 检测这种情况）。
- **Q：`addListener` 的回调在哪个线程跑？**
  A：如果在添加时 future 已完成 → **当前线程**；未完成 → 在 `setSuccess` 的线程（通常是 eventLoop），但 Netty 会用 `executor` 保证"在正确的 executor 里执行"。
- **Q：一个 promise 能设置两次结果吗？**
  A：不能，第二次抛 `IllegalStateException`。

**延伸**：这是 **Future/Promise 模式**（与 JS Promise、CompletableFuture 同族）。Netty 的 `DefaultPromise` 内部用 `AtomicReference` 存状态（`SUCCESS`/`UNCANCELLABLE` 等哨兵值），监听器存在 `GenericFutureListener` 数组或链表里，完成时 CAS 状态后逐个通知。

## 13.2 字节跳动（偏"底层机制 + 极端规模"）

### 题 2.1　从内核数据结构讲，epoll 为什么比 select/poll 快？
**难度**：★★★★　**类型**：深挖题（字节一面高频）

**答题主线**：**三个"不用做"**。

1. **不用每次把 fd 集合拷进内核**：select/poll 每次调用都要把整个 fd 集合从用户态拷到内核态（O(n) 拷贝）；epoll 只在 `epoll_ctl(ADD)` 时拷一次，之后 fd 常驻内核红黑树。
2. **不用遍历所有 fd 判断谁就绪**：select/poll 返回后必须遍历整个集合做 `FD_ISSET`（O(n)）；epoll 由内核在数据到达时通过 `ep_poll_callback` 把就绪 fd 挂到**就绪链表 `rdllist`**，`epoll_wait` 只拷贝就绪的（O(k)）。
3. **不用重复注册**：select 每次都要重新设置 `fd_set`；epoll 的"兴趣集合"（红黑树）和"就绪集合"（链表）是**分离**的。

**数据结构三件套（能画出来最好）**：
```
struct eventpoll {
    struct rb_root_cached rbr;    // 红黑树：所有被监听的 fd（epitem），O(log n) 增删
    struct list_head rdllist;     // 就绪链表：O(1) 插入/取出
    struct wait_queue_head wq;    // epoll_wait 睡眠队列
};
```
**流程**：`epoll_ctl(ADD)` → 插红黑树 + 在 socket 的 `sk_wq` 注册 `ep_poll_callback`；数据到达 → 软中断里 `sk_data_ready` → 回调把 epitem 挂 `rdllist` + 唤醒 `wq`；`epoll_wait` → 拷贝 `rdllist` 到用户态数组。

**常见追问**：
- **Q：为什么要红黑树而不是哈希表？**
  A：需要**有序**（便于范围查找？其实主要是 O(log n) 稳定 + 不需要连续内存 + 支持删除时快速重平衡）。哈希表虽然平均 O(1)，但扩容/冲突退化，且 epoll 的 fd 数量可达百万，红黑树的**最坏 O(log n)** 更可控。
- **Q：`rdllist` 会不会重复挂同一个 fd？**
  A：不会，用 `EPOLL_IS_LINKED` 标记去重（`ep_poll_callback` 里先检查）。
- **Q：epoll 的水平触发会不会让 `rdllist` 里堆满？**
  A：LT 模式下如果数据没读完，fd 会**再次被挂回 rdllist**（`ep_send_events` 里判断），所以是"重复通知"而不是"堆满"。

**延伸**：对比 **kqueue（BSD/macOS）**：kqueue 用"过滤器（filter）+ 事件队列"，比 epoll 更通用（支持文件、信号、定时器），Netty 的 `KQueueEventLoopGroup` 就是对应实现。

---

### 题 2.2　Linux 下 Netty 用的是 LT 还是 ET？为什么？
**难度**：★★★★　**类型**：深挖题（这题能刷掉 80% 的人）

**标准答案**：**JDK NIO 只暴露 LT，所以 `NioEventLoop` 是 LT；但 Netty 通过"一次事件读到读不动为止"来获得 ET 的效率。`EpollEventLoop`（native）可以通过 `EpollChannelOption.EPOLL_MODE` 显式选 ET。**

**三层展开**：

1. **JDK 层面**：`EPollSelectorImpl` 的 `epoll_ctl` 注册的是 `EPOLLIN`（没有 `EPOLLET`），所以是 LT。
2. **Netty 的补偿机制**：`NioByteUnsafe.read()` 里用 `do { ... } while (allocHandle.continueReading())`：
```java
public boolean continueReading() {
    return config.isAutoRead()
        && totalMessages < maxMessagePerRead          // 默认 16
        && attemptedBytesRead == lastBytesRead;       // 上次读满 → 可能还有
}
```
   **"上次读满就继续读"** —— 这等价于 ET 的"读到 EAGAIN"，但不会漏事件（LT 会再通知）。
3. **代价**：LT 下即使读完了，若内核又来了数据，会**再触发一次**事件（多一次 `epoll_wait` 返回 + 系统调用）。所以 LT 的 syscall 次数理论上多于 ET。

**常见追问**：
- **Q：ET 有什么坑？**
  A：① 必须读到 `EAGAIN`，否则剩余数据永远不触发；② 必须非阻塞，否则最后一次 read 永久阻塞；③ 代码里任何"提前 break"都会导致丢事件。
- **Q：为什么 Netty 不直接用 native ET？**
  A：① 默认 transport 要跨平台（Windows 用 `Nio`/`IoUring`，macOS 用 `KQueue`）；② native transport 需要额外的 JNI 依赖（`netty-transport-native-epoll`），部署复杂；③ LT 更安全，出错概率低。**性能差距在实际业务中通常不是瓶颈**（瓶颈常在业务逻辑）。
- **Q：`maxMessagePerRead=16` 为什么要有？**
  A：**公平性**。如果一个连接疯狂发数据，不能让它一直占用 EventLoop 把其他连接饿死。16 次后让出，下轮再处理。

**延伸**：ET 的"读空"要求，本质是因为 `sk_data_ready` 是**状态跃迁通知**（空→非空），不是"有数据就通知"。理解这一点，ET/LT 就不用背了。

---

### 题 2.3　Netty 的"零拷贝"具体指什么？和 sendfile 是一回事吗？
**难度**：★★★　**类型**：区分题（必考，且容易答错）

**答题主线**：**先分清两个世界，再各说三点。**

**内核零拷贝**：
- `sendfile`：`磁盘 → page cache →（仅传描述符）→ 网卡 DMA`，**0 次 CPU 拷贝，2 次切换**；
- `splice`：fd 间通过内核管道搬运，socket→socket 场景；
- `mmap`：映射 page cache，省一次拷贝但仍有 4 次切换。
- 限制：**输入必须是文件**，不能是 socket。

**Netty 用户态零拷贝**（5 个手段）：
1. `CompositeByteBuf`：逻辑合并，避免 `memcpy`；
2. `slice` / `duplicate`：共享内存视图；
3. `wrappedBuffer`：包装已有数组/ByteBuffer，不复制；
4. `FileRegion` + `transferTo`：**这才是真正对接 `sendfile`**；
5. `DirectByteBuf`：堆外内存，地址固定，避免"堆内数组被 GC 移动"的额外拷贝。

**关键陷阱（一定要主动说）**：
> `FileRegion` 一旦加上 `SslHandler` 就**必须降级**为普通 `ByteBuf`，因为 TLS 要逐字节加密，不能再让内核直接 DMA 原始文件内容。这就是"HTTPS 下大文件下载性能明显下降"的原因。Netty 在 `SslHandler` 里对 `FileRegion` 做了转换（否则抛 `UnsupportedOperationException`）。

**常见追问**：
- **Q：`CompositeByteBuf` 有什么代价？**
  A：`get/read` 要二分查找定位 component（O(log n)）；碎片过多时遍历慢，所以要 `maxNumComponents`（默认 16）限制，必要时 `consolidate()`。
- **Q：零拷贝是不是就不用内存了？**
  A：**不是**。零拷贝省的是"CPU 拷贝"，内存照样占（page cache / direct buffer）。**"零拷贝"≠"零内存"**。

---

### 题 2.4　Netty 什么时候注册 `OP_WRITE`？为什么不能一直注册？
**难度**：★★★★　**类型**：深挖题（字节/快手高频）

**答题主线**：

1. **默认不注册**：`writeAndFlush` 直接调 `doWrite` → `write`/`writev` syscall，一次写完就结束，**根本不经过 epoll 的写事件路径**。
2. **写不动才注册**：`doWrite` 返回"没写完"（socket 发送缓冲区满）→ `incompleteWrite()` → `setOpWrite()` → `epoll_ctl(MOD, EPOLLOUT)`。
3. **可写了再写**：`processSelectedKey` 收到 `OP_WRITE` → `forceFlush()` → 继续 `doWrite`；写完 `clearOpWrite()` 注销。
4. **为什么不能一直注册**：Linux 下 socket 发送缓冲区**绝大多数时候是空的**，`EPOLLOUT` **恒定就绪** → `epoll_wait` 每次都立即返回 → `forceFlush` 空跑 → **CPU 100% 空转，且饿死其他连接**。

**代码路径（能说出来就赢了）**：
```
flush0()
  → doWrite(in)               // write / writev
  → if (!in.isEmpty()) {      // 没写完
        incompleteWrite(true) // → setOpWrite()
    } else {
        clearOpWrite()
    }
```

**常见追问**：
- **Q：`OP_WRITE` 注册了但一直不可写怎么办？**
  A：数据堆在 `ChannelOutboundBuffer`（堆外），超过高水位（64KB）→ `isWritable()=false` → 业务应背压（停止读/降速）。**这本质是"用内存换时间"，必须有上限，否则 OOM。**
- **Q：为什么 `write` 不直接返回"写了多少"给业务？**
  A：因为 Netty 是异步模型，`ChannelFuture` 表示"是否写进了发送缓冲区"，真正的网络传输由内核负责，业务无需感知。
- **Q：`writeSpinCount` 是什么？**
  A：`doWrite` 里的自旋次数上限（默认 16），避免"一次 flush 在一个 channel 上写太久"，影响 EventLoop 上其他 channel 的公平性。

**延伸**：这是一个"**事件注册的经济学**"问题 —— 注册一个"几乎总是就绪"的事件，等于让多路复用退化成忙等。**所有基于 epoll 的框架（Redis、Nginx）都遵循同一原则：只在真正需要时才注册 OP_WRITE。**

---

### 题 2.5　"Netty 为什么快"这个问题，你会怎么答才有区分度？
**难度**：★★（题简单，但要答出层次）　**类型**：筛人题

**差答案**（会被打断）："因为它是异步非阻塞的、用了 NIO。"

**好答案**：**按"成本模型"分层**——Netty 快在**把四类成本都压到了接近下限**。

| 成本 | 朴素实现 | Netty 的做法 |
|---|---|---|
| **系统调用次数** | 每消息一次 `write` | `write` 入队 + `flush` 批量 → `writev` 一次发多段；读用 `maxMessagesPerRead` 批量 accept |
| **内存分配/拷贝** | 每包 `new byte[]` | `PooledByteBufAllocator` + ThreadCache，**网络路径零新对象**；`CompositeByteBuf`/`slice` 避免 `memcpy` |
| **线程/锁** | 线程池 + 锁 | 单 Channel 单 EventLoop 串行，**无锁**；跨线程只投递任务 |
| **上下文切换** | 每 IO 一次阻塞唤醒 | 一个线程 `epoll_wait` 管 N 连接，**切换次数与连接数解耦** |

**加一句"灵魂总结"**：
> "Netty 的性能不是靠某个黑科技，而是**把'每连接每消息'的固定成本（系统调用、内存分配、锁、线程切换）摊薄到接近零**。它的每一个设计——池化、批量 flush、无锁串行、自适应缓冲区——都对应一个具体的成本项。"

**常见追问**：
- **Q：那 Netty 有什么性能代价？**
  A：① 直接内存需要手动管理（泄漏风险）；② 池化内存常驻（RSS 高）；③ 学习成本高、调试难（异步栈断裂）；④ 单 EventLoop 内一个慢 handler 会拖累所有连接。
- **Q：什么场景不该用 Netty？**
  A：① 极低并发（几个连接，用 `HttpClient`/`RestTemplate` 更简单）；② 简单 HTTP 服务（Spring MVC 足够，Netty 是杀鸡用牛刀）；③ 团队没有网络编程能力（维护成本 > 收益）。

---

## 13.3 腾讯（偏"框架理解 + 长连接/游戏场景"）

### 题 3.1　Pipeline 的事件是怎么传播的？`ctx.write` 和 `channel.write` 有什么区别？
**难度**：★★★★　**类型**：深挖题（腾讯 IEG/CSIG 高频）

**答题主线**：

**结构**：`head ↔ handler1 ↔ handler2 ↔ tail`，双向链表，`head` 是 outbound 终点，`tail` 是 inbound 终点。

**传播方向**：
- **入站**（`channelRead`/`channelActive`/`exceptionCaught`）：从 `head` 沿 `next` 到 `tail`，**按添加顺序**；
- **出站**（`write`/`flush`/`close`）：从触发点沿 `prev` 到 `head`，**逆序**。

**为什么相反**：入站是"数据从 socket 进来往上走"，出站是"业务从里往外走到 socket"，**方向天然相反**。

**三个 API 的区别（表格化回答最清晰）**：

| API | 起点 | 跳过的 handler | 是否 flush |
|---|---|---|---|
| `channel.writeAndFlush` | `tail` | 无（经过所有 outbound） | 是 |
| `channel.write` | `tail` | 无 | 否 |
| `ctx.write` | **当前节点的前一个** | **自己** | 否 |
| `ctx.writeAndFlush` | 同上 | 自己 | 是 |

**源码依据**：`AbstractChannelHandlerContext.findContextOutbound()` 是从 `this.prev` 开始找的 —— 所以 `ctx.write` **不经过自己**。这带来一个经典 bug：

```java
// 假设 pipeline: [decoder] [encoder] [business]
// business 里用 ctx.writeAndFlush(obj)
// → 从 business 的 prev 开始找 outbound → encoder ✔ 正确
// 但如果顺序是 [decoder] [business] [encoder]
// → 从 business 的 prev 开始找 → 直接到 head → encoder 被跳过 ✘ 裸数据发出
```

**常见追问**：
- **Q：`@Sharable` 的作用？**
  A：声明 handler 可被多 Channel 共享。Netty 在 `addLast` 时检查：非 `@Sharable` 且已添加过 → 抛 `ChannelPipelineException`。**它是"我保证线程安全"的承诺，不是加锁。**
- **Q：在 handler 里 `pipeline.remove(this)` 会怎样？**
  A：Netty 会改 `prev.next`/`next.prev`，当前事件继续按已取到的 `next` 传播；但被移除节点后续的逻辑不会执行。**不要在 `channelRead` 里 `remove` 后又依赖 `ctx`**（可能已被置空）。
- **Q：`exceptionCaught` 往哪个方向传？**
  A：**入站方向**（往 tail 走）。一路没人处理 → `TailContext` 打 WARN。它**不会自动关闭连接**。

**延伸**：这套"双向链表 + 每节点带 executor"的设计，使 pipeline 可以**热插拔**（动态加解密、动态限流、协议切换）——这是 Netty 相比"固定处理链"的最大优势，也是游戏服务器"网关动态加协议"的基础。

---

### 题 3.2　TCP 三次握手、四次挥手，`TIME_WAIT` 和 `CLOSE_WAIT` 分别说明什么问题？
**难度**：★★★　**类型**：区分题

**答题主线**：

**三次握手**：`SYN → SYN+ACK → ACK`。为什么不是两次：防止"已失效的连接请求"到达服务端导致资源浪费（历史重复 SYN）。为什么不是四次：`SYN+ACK` 合并了。

**四次挥手**：`FIN → ACK → FIN → ACK`。为什么是四次：TCP 是全双工，一方不再发数据（FIN）不代表另一方也发完了，所以 ACK 和 FIN 要分开。

**两个状态的含义（这是重点）**：

| 状态 | 谁 | 本质 | 处置 |
|---|---|---|---|
| **TIME_WAIT** | 主动关闭方，持续 2MSL（Linux 60s） | **正常**。保证最后 ACK 可重传、旧连接报文过期 | 连接池/长连接治本；`tcp_tw_reuse` 缓解 |
| **CLOSE_WAIT** | 被动关闭方 | **异常**。内核已回 ACK，但应用没调 `close()` | **代码 bug**：连接泄漏、异常分支未关闭 |

**口诀**：`TIME_WAIT` 多 = 架构问题；`CLOSE_WAIT` 多 = 代码问题。

**常见追问**：
- **Q：为什么 TIME_WAIT 是 2MSL 而不是 1MSL？**
  A：MSL 是报文最大生存时间。要覆盖"我的 ACK 丢了 → 对方重传 FIN → 我再回 ACK"这个往返，所以是 2 倍。
- **Q：服务端能不能避免 TIME_WAIT？**
  A：让**服务端主动关闭**（TIME_WAIT 落在服务端，端口固定不吃端口）；或用 `SO_LINGER=0` 发 RST（**危险**，可能丢数据）；或干脆用长连接。
- **Q：`tcp_tw_recycle` 能用吗？**
  A：**不能**，Linux 4.12 已移除。它在 NAT 环境下会误判时间戳导致连接被拒。

**延伸**：Netty 里对应的是 `ALLOW_HALF_CLOSURE`（半关闭）和 `channel.close()` 的时机。**大量 CLOSE_WAIT 在 Netty 服务里通常意味着某个 handler 的 `channelInactive` 里忘了 `ctx.close()`**。

---

### 题 3.3　Netty 怎么优雅关闭？顺序为什么重要？
**难度**：★★★　**类型**：工程题

**答题主线**：
1. `shutdownGracefully(quietPeriod, timeout, unit)` —— 默认 2s / 15s；
2. **quietPeriod**：关闭后再等多久没有新任务才退出（防止关闭瞬间仍有任务进来）；
3. **timeout**：最长等待，超时强制关闭所有 channel；
4. **顺序**：先 `boss`（停止 accept）→ 等存量请求处理完 → 再 `worker`（关所有连接）→ 最后业务线程池；
5. **为什么**：如果先关 worker，listen fd 还在，新连接进来没人处理 → 客户端"连上即超时"。反之先关 boss，内核不再 accept，新连接会被拒绝/超时，但存量连接仍能正常收尾。

**常见追问**：
- **Q：`shutdownGracefully` 之后还要做什么？**
  A：`awaitTermination`/`sync()`，否则主线程退出会打断未完成的关闭流程（JVM 退出时 hook 也不保证）。
- **Q：怎么保证"存量请求处理完"？**
  A：业务层维护 `inflight` 计数（`LongAdder`），`shutdown` 后拒绝新请求，轮询计数归零再关 worker。
- **Q：关闭时还有未 flush 的写数据怎么办？**
  A：`shutdownGracefully` 会等任务队列清空，但**不保证内核缓冲区已发出**；严格场景要 `channel.closeFuture()` + `SO_LINGER` 配合，或业务层等 ACK。

---

### 题 3.4　设计一个百万在线的游戏/IM 长连接服务，你会怎么划分 Netty 的组件？
**难度**：★★★★　**类型**：架构题（腾讯 IEG 常问）

**答题主线**（分层 + 每层给数字）：

**① 接入层（Gateway）**
- `boss` 1 个（或 `SO_REUSEPORT` 多 boss），`worker` = 2×CPU；
- 只做：协议解析、鉴权、心跳、路由（**不做业务**，避免阻塞 IO 线程）；
- 每 worker 绑 1~2w 连接；单机 10w~50w 连接（受内存与 fd 限制）。

**② 会话层（Session）**
- 用 `ChannelGroup` 或自建 `Map<uid, Channel>`（**注意：`ChannelGroup` 主要做批量广播，不保证强一致**）；
- 多机时用 **Redis 存路由**（uid → gatewayId），或一致性哈希；
- `Channel.attr()`（`AttributeKey`）存用户上下文，**线程安全且生命周期随 channel**。

**③ 业务层**
- 独立 `EventExecutorGroup` 或独立服务（推荐独立服务）；
- 消息通过 MQ / RPC 转发，Gateway 只做"收 → 转发 → 回"。

**④ 可靠性**
- 心跳：`IdleStateHandler` 30~60s，连续 2 次未响应才断；
- 重连：指数退避 + 抖动；
- 消息可靠：客户端 ACK + 服务端重发（幂等 ID）。

**⑤ 关键数字（面试官爱听）**
- 10w 连接 ≈ 10w fd + 10w × (2KB 内核 + 16KB 缓冲) ≈ 1.8GB；
- 100w 连接必须"网关只转发 + 小缓冲 + native epoll"，且通常要多机。

**常见追问**：
- **Q：广播 10w 人怎么做才不卡？**
  A：**不要在一个线程里循环 10w 次 `writeAndFlush`**（会撑爆 outbound buffer 并阻塞 eventLoop）。做法：① 按 EventLoop 分组批量写（每个 loop 处理自己那批）；② 用 `ChannelGroup.writeAndFlush` 的分组优化；③ 超过水位线就暂停，分批投递。
- **Q：怎么防止"慢客户端"拖垮网关？**
  A：`isWritable()` 检查 + `setAutoRead(false)` + 超时踢下线 + 每连接写队列上限。

---

## 13.4 美团（偏"协议/编解码 + 稳定性 + 中间件"）

### 题 4.1　粘包和半包是怎么产生的？`LengthFieldBasedFrameDecoder` 五个参数怎么配？
**难度**：★★★　**类型**：区分题（必考）

**答题主线**：

**原因（一句话）**：TCP 是**字节流**，段的大小由 MSS/MTU/拥塞窗口/发送缓冲区决定，**与业务消息边界无关**。所以：
- 多个小消息被合并（Nagle / 发送缓冲区）→ **粘包**；
- 一个大消息被 MTU 切分 → **半包**；
- 两者混合是最常见情况。

**三种解法**：定长 / 分隔符 / **长度字段**（主流）。

**五个参数**（用一张图讲最清楚）：
```
|<-- lengthFieldOffset -->|<-- lengthFieldLength -->|<-- lengthAdjustment -->|
+------------------------+-------------------------+-------------------------+
|      header 其他字段    |       length 值          |        body ...         |
+------------------------+-------------------------+-------------------------+
|<- initialBytesToStrip ->|   （解码后丢弃这么多字节）
```

| 参数 | 含义 | 常见取值 |
|---|---|---|
| `maxFrameLength` | 最大帧长（**必须设，防 OOM**） | 1MB / 16MB |
| `lengthFieldOffset` | 长度字段的起始偏移 | 0（开头）或 12（Dubbo） |
| `lengthFieldLength` | 长度字段占几字节 | 2 / 4 |
| `lengthAdjustment` | 长度值需要修正的偏移 | 若 length 含 header 则填**负数 header 长度** |
| `initialBytesToStrip` | 解码成功后丢弃前几字节 | 通常 = offset+length（丢掉长度字段） |

**举例（最容易错的 `lengthAdjustment`）**：
- 协议：`| len(4B, 含自身+body) | body |`
- 则 `lengthAdjustment = -4`（因为长度值把 4 字节头也算进去了，要减回来）
- `initialBytesToStrip = 4`

**常见追问**：
- **Q：`maxFrameLength` 不设会怎样？**
  A：恶意客户端发 `len=2GB`，Netty 会尝试分配 2GB 的累积缓冲 → **OOM，服务被打挂**。这是真实的攻击面。
- **Q：`ByteToMessageDecoder` 的内部累积用什么？**
  A：`cumulation`，默认 `MERGE_CUMULATOR`（memcpy 追加），可换 `COMPOSITE_CUMULATOR`（零拷贝但碎片多）。
- **Q：一次收到 3 个完整帧怎么办？**
  A：`ByteToMessageDecoder` 的 `callDecode` 是 `while (in.isReadable())` 循环，**一次解出多个**，每个 `fireChannelRead`。这就是"粘包"的处理方式。

**延伸**：Dubbo 协议、RocketMQ 协议、gRPC（HTTP/2 frame）都是"长度字段 + 头部"的变体。**能画出自己公司协议的帧结构，是这道题的满分表现。**

---

### 题 4.2　Netty 的背压机制是什么？`isWritable` 和 `autoRead` 怎么配合？
**难度**：★★★★　**类型**：深挖题（美团/拼多多高频，因为外卖/电商流量大）

**答题主线**：

**问题**：如果下游消费慢（客户端不读、或处理慢），上游还在拼命写 → 数据堆在 `ChannelOutboundBuffer`（堆外）→ 内存上涨 → OOM / Full GC。

**Netty 的机制**：
1. **水位线**：`WRITE_BUFFER_HIGH_WATER_MARK`（默认 64KB）/ `LOW_WATER_MARK`（32KB）；
2. 写队列累计字节 ≥ 高水位 → `unwritable=true` → `fireChannelWritabilityChanged()`；
3. 写成功后累计降到 ≤ 低水位 → `unwritable=false` → 再触发一次；
4. **业务侧**必须监听这个事件，做"停止读上游 / 降低发送速率"。

**正确代码**：
```java
@Override
public void channelWritabilityChanged(ChannelHandlerContext ctx) {
    if (ctx.channel().isWritable()) {
        ctx.channel().config().setAutoRead(true);
    } else {
        ctx.channel().config().setAutoRead(false);   // 停止从 socket 读，TCP 窗口自然收缩
    }
    ctx.fireChannelWritabilityChanged();
}
```

**为什么 `autoRead=false` 能起作用**：停止注册 `OP_READ` → 不再调用 `read()` → 内核接收缓冲区堆满 → **TCP 通告窗口变小/为 0** → 发送方（对端）被内核限速。**这是从"用户态限流"传导到"TCP 层限流"的完整闭环。**

**常见追问**：
- **Q：水位线设多少合适？**
  A：要大于"单条消息大小"（否则每条都触发），且不能太大（失去意义）。默认 64KB 适合小消息；大消息（如图片）可设 1MB。
- **Q：`isWritable=false` 时 `write` 会失败吗？**
  A：**不会**，仍然入队（只是不推荐）。所以业务必须**自己检查**，否则等于没背压。
- **Q：背压只对写有效，读方向呢？**
  A：读方向靠 `autoRead=false` + 业务自己的有界队列（如 `ArrayBlockingQueue`），队列满则停止读。

**延伸**：这是 **Reactive Streams 的背压** 在 Netty 的落地。对比 **Akka Streams** 的 `maxInFlight`、**gRPC 的 flow control**（HTTP/2 窗口），本质相同：**用"窗口/水位"把下游压力反向传导给上游**。

---

### 题 4.3　`ioRatio` 是什么？`runAllTasks` 的时间预算怎么算？
**难度**：★★★★　**类型**：源码细节题

**答题主线**：
```java
long ioStartTime = System.nanoTime();
processSelectedKeys();
long ioTime = System.nanoTime() - ioStartTime;
if (ioRatio == 100) runAllTasks();
else runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
```
- 默认 `ioRatio = 50` → **任务预算 = IO 耗时**（各占 50%）；
- 含义：不是"精确 50% CPU"，而是**用上次 IO 耗时做参照**，简单有效；
- `runAllTasks(timeout)` 内部每 64 个任务检查一次时间（`i & 0x3F == 0`），超预算就停，剩余留到下轮。

**常见追问**：
- **Q：为什么每 64 个才检查时间？**
  A：`nanoTime()` 有开销（即使走 vDSO），批量检查摊薄成本。
- **Q：`ioRatio=100` 什么场景用？**
  A：几乎不用。任务队列很大时会让 IO 饿死（连接不被读取 → 窗口收缩 → 吞吐崩）。
- **Q：任务队列满了怎么办？**
  A：`taskQueue` 是无界 MPSC 队列，**没有拒绝策略**，所以"任务投递速度 > 消费速度"时会 OOM。业务要自己有界（比如限制并发写）。

**延伸**：这类"用上一轮耗时预估下一轮预算"的调度思想，在 **Go 的 GMP 调度**、**Linux CFS 的 vruntime** 里都有类似体现（当然机制不同）。核心是**避免精确记账的开销**。

---

### 题 4.4　线上 Netty 服务 CPU 100%，你怎么排查？
**难度**：★★★★　**类型**：工程题（美团/滴滴的"故障排查"风格）

**答题主线**：**先分层定位，再给具体命令。**

**第一步：确认是"用户态忙"还是"内核态忙"**
```bash
top -Hp <pid>            # 看是 us 高还是 sy 高
pidstat -p <pid> -u 1    # %usr vs %system
```
- `%sy` 高 → 系统调用过多（小包、频繁 write、日志）
- `%us` 高 → 业务逻辑/编解码/空轮询

**第二步：抓线程栈**
```bash
jstack <pid> > stack.txt
# 找 RUNNABLE 且占比高的线程（nid 对应 top -Hp 的 tid 转十六进制）
```
**常见栈特征**：

| 栈顶 | 根因 |
|---|---|
| `sun.nio.ch.EPollSelectorImpl.doSelect` / `select` 反复出现 | **epoll 空轮询**（JDK bug） |
| `io.netty.channel.nio.NioEventLoop.processSelectedKeys` | 事件太多（连接风暴/广播风暴） |
| 业务方法（JSON/正则/DB） | **业务阻塞 IO 线程**（最危险） |
| `Unsafe.write` / `writev` | 大量小写（未批量 flush） |

**第三步：对症下药**
- 空轮询：`-Dio.netty.selectorAutoRebuildThreshold=512`（默认），或看日志确认重建；根治要升级 JDK。
- 事件风暴：看 `ss -s` 连接数是否突增；是不是广播没分组。
- 业务阻塞：`EventExecutorGroup` 隔离 / 改异步。
- 小包多：`TCP_NODELAY` + 批量 flush + 检查日志级别。

**常见追问**：
- **Q：怎么确认是"空轮询"而不是"真有大量事件"？**
  A：`processSelectedKeys` 的栈里如果 `selectedKeys.size == 0` 却反复返回，就是空轮询；也可以加监控打点。
- **Q：CPU 不高但 RT 很高，怎么查？**
  A：看 GC 日志（STW）、看 `ss -ti` 的 cwnd/rtt、看线程池队列、看下游依赖（DB/Redis）耗时。

---

## 13.5 京东（偏"消息可靠性 + 中间件"）

### 题 5.1　用 Netty 做一个消息推送/IM，怎么保证消息不丢、不乱序？
**难度**：★★★　**类型**：工程题

**答题主线**：**"不丢"和"不乱"是两个独立问题，要分开答。**

**不丢（三层）**：
1. **连接层**：`ChannelFuture` 只保证"写进内核缓冲区"，不保证"对端收到" → 必须**应用层 ACK**；
2. **应用层**：消息带 `msgId`，服务端维护"待确认表"，客户端回 ACK 才删除；超时重发（`HashedWheelTimer` 适合海量超时）；
3. **存储层**：离线消息落库/Redis，上线后拉取（**"推拉结合"**：在线推、离线拉）。

**不乱序（两层）**：
1. **同一 Channel 天然有序**：单 EventLoop 串行 + TCP 按序 → 只要不跨线程乱写，顺序就有保证；
2. **跨连接/跨机器**：需要**序列号 + 客户端重排**（或"服务端按 uid 哈希到同一队列"）。

**经典坑**：
- 用 `writeAndFlush` 后立刻认为"发送成功" → 网络断了就丢；
- 重发没有幂等 → 客户端收到重复消息；
- 多线程往同一 Channel 写 → `ChannelOutboundBuffer` 链表竞争（应统一走 `channel.writeAndFlush`，它会调度到 eventLoop）。

**追问**：
- **Q：ACK 机制会不会拖慢吞吐？**
  A：会。所以采用"批量 ACK"（每 N 条或每 T 毫秒确认一次），或用 TCP 本身的可靠性 + 只在"关键消息"上做 ACK。
- **Q：怎么处理"客户端 ACK 丢了但消息已收到"？**
  A：幂等（`msgId` 去重），服务端重发、客户端丢弃。

---

### 题 5.2　Netty 的定时任务是用时间轮实现的吗？两者怎么选？
**难度**：★★★　**类型**：概念辨析（高频陷阱）

**标准答案**：**不是。`EventLoop.schedule()` 用的是 `PriorityQueue`（小顶堆），`HashedWheelTimer` 是独立的工具类。**

| | `EventLoop.schedule` | `HashedWheelTimer` |
|---|---|---|
| 结构 | 小顶堆 | 时间轮（512 bucket 环形数组 + 链表） |
| 插入 | O(log n) | **O(1)** |
| 精度 | 高（受 select 唤醒影响，可到 ms 级） | `tickDuration`（默认 100ms） |
| 数量级 | 数百~数千 | **数十万~百万** |
| 典型用途 | `IdleStateHandler`、`ReadTimeoutHandler`、连接超时 | Dubbo 请求超时、海量订单超时关闭 |

**时间轮原理（一句话）**：时间轴切成 tick，任务按"到期 tick"挂到对应 bucket 的链表；worker 线程每 tick 前进一格处理该 bucket；跨圈任务用 `remainingRounds` 计数。**用精度换 O(1)**。

**追问**：
- **Q：为什么 EventLoop 不用时间轮？**
  A：EventLoop 的定时任务量通常不大，且需要高精度；小顶堆实现简单、与 `select(timeout)` 天然配合（"超时时间就是最近任务的到期时间"）。
- **Q：时间轮的空推进问题？**
  A：Netty 的 `HashedWheelTimer` 用 `sleep` 到下一个 tick，不是忙等；但 tick 太小时 CPU 占用高。

---

### 题 5.3　`ByteBuf` 相比 `ByteBuffer` 到底好在哪？能手写一个"读半包"的例子吗？
**难度**：★★　**类型**：及格题

**答题主线**（七点，简洁列）：
1. 读写指针分离（无需 `flip`）；
2. 可扩容（`capacity`→`maxCapacity`）；
3. 池化（`PooledByteBufAllocator`）；
4. 引用计数（确定性释放）；
5. 复合缓冲（`CompositeByteBuf`）；
6. 视图（`slice`/`duplicate`）；
7. 丰富的 API（`readUnsignedInt`、`getByte` 越界检查等）。

**手写半包判断（这是加分项，很多人写不出）**：
```java
// 假设：| len(2B) | body |，len 只表示 body 长度
if (in.readableBytes() < 2) return;             // 半包：连长度都不够
int len = in.getUnsignedShort(in.readerIndex()); // 先 peek，不移动指针
if (in.readableBytes() < 2 + len) return;       // 半包：body 不够
int bodyLen = in.readUnsignedShort();            // 真正消费长度字段
ByteBuf body = in.readSlice(bodyLen);            // 零拷贝切出 body（注意生命周期！）
// 若要把 body 传给下游，必须 retainedSlice 或 copy
```

**追问**：
- **Q：`readSlice` 和 `readRetainedSlice` 的区别？**
  A：前者不增加引用计数（父 buf 释放后失效），后者 +1（可独立存活）。**跨 handler 传递必须用 retained。**
- **Q：为什么 `getUnsignedShort(readerIndex)` 不移动指针？**
  A：`get` 系列是"窥视"，`read` 系列才消费。半包判断必须先 peek 再决定是否消费。

---

## 13.6 百度（偏"操作系统 + 底层实现"）

### 题 6.1　`select`、`poll`、`epoll` 三者的区别，各自的适用场景？
**难度**：★★★　**类型**：基础但要求准确

**表格化回答**：

| | select | poll | epoll |
|---|---|---|---|
| fd 上限 | 1024（`FD_SETSIZE`） | 无（数组） | 无（红黑树） |
| 每次是否拷贝 fd 集合 | **是**（全量） | **是**（全量） | 否（只在 ctl 时） |
| 返回后是否遍历全部 | **是**（O(n)） | **是**（O(n)） | 否（O(k)，k=就绪数） |
| 触发方式 | LT | LT | LT / ET |
| 内核数据结构 | 位图 | 数组 | 红黑树 + 就绪链表 |
| 适用 | 几乎淘汰 | 少量 fd 且无上限 | **高并发（>1000）** |

**加分点**：
- select 的 1024 限制来自 `fd_set` 是 1024 位；
- poll 用 `pollfd` 数组，但"拷贝 + 遍历"没解决，**只是解除了数量限制**；
- epoll 的优势是**复杂度与"关注的 fd 数"解耦**，只与"活跃 fd 数"相关。

**追问**：
- **Q：epoll 一定比 select 快吗？**
  A：**不一定**。活跃 fd 很多（比如 90% 都活跃）时，epoll 的"回调 + 链表"开销反而可能超过 select 的一次遍历；**epoll 的优势场景是"少量活跃 + 大量空闲"**（典型长连接）。这是很多人不知道的细节。
- **Q：epoll 的 ET 适合什么场景？**
  A：高并发、短连接/大量小包（Nginx、Redis）。需要"一次读空"，减少唤醒次数。

---

### 题 6.2　Netty 的直接内存和 JVM 的 `DirectByteBuffer` 有什么关系？为什么不直接用 `allocateDirect`？
**难度**：★★★★　**类型**：深挖题

**答题主线**：

**JVM 的 `allocateDirect` 路径**：
```
DirectByteBuffer → Bits.reserveMemory（记账，检查 MaxDirectMemorySize）
                 → Unsafe.allocateMemory（malloc）
                 → 注册 Cleaner（PhantomReference）
释放：只有对象被 GC + Cleaner 被处理时 → free
```
问题：① **释放时机不可控**（依赖 GC，Full GC 才可靠）；② 堆内存充足时可能永不 GC → **直接内存 OOM**；③ `Cleaner` 本身有对象开销。

**Netty 的做法**（`PlatformDependent`）：
- 通过反射/Unsafe **直接调 `Unsafe.allocateMemory` / `freeMemory`**，自己记账（`DIRECT_MEMORY_LIMIT`）；
- `ByteBuf.release()` 归零时**立即 `free`**（池化时归还 chunk），**不依赖 GC**；
- 参数：`-Dio.netty.maxDirectMemory`（默认 = `MaxDirectMemorySize`；0 = 不限制；-1 = 用 JVM 的）。

**代价**：**必须保证 release**，否则 JVM 完全不知道这块内存（`jmap` 看不到），只能靠 NMT 或 RSS 排查。

**追问**：
- **Q：`-XX:MaxDirectMemorySize` 对 Netty 还有效吗？**
  A：部分有效。Netty 默认会读取它作为自己的上限，但 Netty 的分配**不经过 JVM 的 `Bits.reserveMemory`**，所以 JVM 不会拦；超过 Netty 自己的限制才会抛 `OutOfDirectMemoryError`。
- **Q：为什么不全部用堆内存？**
  A：① 堆内数组会被 GC 移动，write 前可能需要 pin/copy；② 堆外减少 GC 扫描压力；③ 堆外可直接 `writev`，无需临时拷贝。
- **Q：堆外内存怎么监控？**
  A：`-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory`；或 Netty 的 `PooledByteBufAllocator.metric()`（`directArenas` 的 `used`/`chunkSize`）。

---

### 题 6.3　Netty 的 native transport（`EpollEventLoopGroup`）和默认 NIO 有什么差别？什么时候该用？
**难度**：★★★　**类型**：区分题

**差别**：

| 维度 | NIO (`NioEventLoopGroup`) | Native (`EpollEventLoopGroup`) |
|---|---|---|
| 实现 | JDK `Selector`（封装 epoll） | **JNI 直接调 epoll** |
| 支持 ET | 否 | **是**（`EPOLL_MODE`） |
| 额外能力 | 无 | `SO_REUSEPORT`、`TCP_QUICKACK`、`SO_BUSY_POLL`、`EpollChannelOption` |
| 性能 | 好 | **略好**（少一层 JDK 封装，事件处理更快） |
| 部署 | 开箱即用 | 需引入 `netty-transport-native-epoll` + 对应平台的 classifier |
| 跨平台 | 全平台 | 仅 Linux（macOS 用 KQueue） |

**什么时候用**：
- 高并发网关（QPS 高、连接多）→ 值得；
- 需要 `SO_REUSEPORT` 多进程/多 boss → 必须；
- 需要 ET 模式 → 必须；
- 普通业务服务 → **NIO 足够**，别为 3% 的性能引入运维复杂度。

**追问**：
- **Q：怎么判断 native transport 加载成功？**
  A：`Epoll.isAvailable()`，失败原因可用 `Epoll.unavailabilityCause()`（常见：glibc 版本、`netty-transport-native-epoll` 未引入）。
- **Q：`SO_REUSEPORT` 有什么用？**
  A：多个进程/多个 listen socket 绑定同一端口，**内核做负载均衡**，避免惊群；Nginx 多 worker、Netty 多 boss 都用它。

---

## 13.7 快手（偏"极端规模 + 流量治理"）

### 题 7.1　单机千万连接，架构上要怎么做？（开放题）
**难度**：★★★★★　**类型**：架构题

**答题主线**：**先算账，再讲架构，最后讲取舍。**

**① 算账（面试官最看重这个）**
- 1000w fd：`ulimit -n` 要 1000w+，`fs.file-max` 要更高；**单机通常做不到**（内核 `file` 结构、`dentry` 内存开销大）→ 现实是**多机 + 连接调度**。
- 每连接内核成本：`struct sock` ~1.5KB + 收发缓冲（可压到 4KB/4KB）→ 1000w × 10KB ≈ 10GB → **需要 64GB+ 内存**。
- **结论**：单机千万连接在"纯心跳、无流量"场景理论可行（业界有 100w~200w 的实践，1000w 多靠多机 + 代理层）。

**② 架构**
```
L4 负载均衡（DPDK/LVS）→ 接入层（多机，每机 50w~100w）
                              → 逻辑层（无状态，按 uid 路由）
```
- 接入层只做"连接保持 + 心跳 + 转发"，**不执行业务**；
- 用 `SO_REUSEPORT` + 多进程/多 boss 分摊 accept；
- 心跳**分批 + 抖动**（不能同一秒 100w 个包）；
- 每连接状态极简（`Channel.attr()` 只存 uid/sessionId）。

**③ 取舍**
- 小 `SO_RCVBUF`（4~8KB）换连接数，代价是吞吐；
- 关闭 `SO_KEEPALIVE`（内核每连接一个 timer，1000w 个 timer 很贵），改用**应用层批量扫描**（时间轮扫全表）；
- 用 `Epoll LT` 而非 ET（避免漏事件导致连接"假死"）。

**追问**：
- **Q：1000w 连接怎么做心跳而不爆 CPU？**
  A：**不要每连接一个定时任务**（1000w 个 `ScheduledFuture` 会占几十 GB）。用**分层时间轮 + 批量扫描**：每秒扫一批（如 1/60），把"超时"判定分散到 60s 内。
- **Q：`ChannelGroup` 能存 1000w 吗？**
  A：`DefaultChannelGroup` 内部是 `ConcurrentMap`，能存但遍历/广播开销大；大规模应自己做分片（按 EventLoop 分组）。

---

### 题 7.2　什么是惊群？Netty 里怎么避免？
**难度**：★★★★　**类型**：深挖题

**定义**：多个进程/线程等待同一事件（如 accept），事件到来时**全部被唤醒**，但只有一个能处理，其余徒劳 → CPU 浪费 + 延迟抖动。

**三种解法**：
1. **`EPOLLEXCLUSIVE`**（Linux 4.5+）：只唤醒一个等待者；
2. **`SO_REUSEPORT`**：每个进程独立 listen socket，内核按四元组哈希分流，**从根本上没有共享**；
3. **单线程 accept**（Netty 默认）：只有一个 boss EventLoop 绑 listen fd，**天然无惊群**。

**Netty 的实际情况**：
- 默认单 boss → 无惊群；
- 多 boss（`new NioEventLoopGroup(4)`）→ **一个 listen fd 仍只注册在一个 EventLoop**，其他 boss 空闲 → 不是惊群，是**浪费**；
- 真正多 boss 并行要 `SO_REUSEPORT` + 多个 `ServerBootstrap`。

**追问**：
- **Q：`SO_REUSEPORT` 有什么坑？**
  A：① 同一端口必须所有 socket 都设该选项；② 内核按哈希分流，**可能不均**（连接数少时）；③ 进程重启时，旧进程的连接不会自动迁移。
- **Q：为什么 Netty 默认 boss 是 2×CPU 而不是 1？**
  A：历史设计（`MultithreadEventLoopGroup` 统一默认值）。实践中**建议显式设 1**，避免无谓线程。

---

### 题 7.3　在 Netty 上怎么做限流/流量整形？
**难度**：★★★★　**类型**：架构题

**答题主线**：分三个位置。

**① 接入限流（连接级）**
- 限制总连接数：`ChannelGroup.size()` + 拒绝（`ctx.close()`）；
- 限制单 IP 连接数：`Map<ip, count>` + 定时清理；
- 限制 accept 速率：`AbstractTrafficShapingHandler`（Netty 内置）。

**② 请求限流（QPS 级）**
- **令牌桶/漏桶**：`RateLimiter`（Guava）或自己实现，在 `channelRead` 入口判断；
- **不要在 IO 线程里 `sleep` 等令牌**（会阻塞整个 EventLoop）→ 用"异步等待 + 定时重试"或"直接拒绝"；
- 分布式限流用 Redis + Lua（滑动窗口）。

**③ 带宽限流（字节级）**
- `AbstractTrafficShapingHandler`（`ChannelTrafficShapingHandler` / `GlobalTrafficShapingHandler`）：
  原理是"计算应发时间，延迟写出"，底层用 `schedule` + 检查 `ChannelOutboundBuffer`，**会让写变慢而不是丢弃**；
- 注意：它会给每个 channel 加定时任务，**海量连接时开销大**。

**追问**：
- **Q：限流应该"拒绝"还是"排队"？**
  A：接入层**拒绝**（快速失败，保护自己）；业务层可**排队**（有界队列 + 超时）。
- **Q：怎么防止限流把 IO 线程拖死？**
  A：限流逻辑放在业务线程池或独立的 `EventExecutorGroup`，IO 线程只做"快速判断"。

---

## 13.8 拼多多（偏"高并发 + 框架对比"）

### 题 8.1　Netty、Reactor Netty、Spring WebFlux 是什么关系？
**难度**：★★★　**类型**：概念题（很多人答不清）

**层级关系**：
```
Spring WebFlux（响应式 Web 框架，基于 Reactive Streams）
      ↓ 默认使用
Reactor Netty（基于 Netty 的响应式网络层，封装了 HTTP 客户端/服务端）
      ↓ 底层
Netty（通用网络框架：EventLoop、ByteBuf、Pipeline）
```

- **Netty**：通用、协议无关；
- **Reactor Netty**：把 Netty 的 `Channel`/`ByteBuf` 适配成 `Flux`/`Mono`，提供 `HttpServer`/`HttpClient`、连接池、背压；
- **Spring WebFlux**：注解式/函数式 Web 框架，底层可切 Reactor Netty 或 Servlet 3.1（Tomcat）。

**追问**：
- **Q：WebFlux 比 Spring MVC 快吗？**
  A：**不一定**。WebFlux 的优势在"高并发 + 少量线程"（IO 密集、长连接、流式），如果业务是 CPU 密集或用了阻塞 JDBC，反而更慢（线程模型不匹配）。
- **Q：Reactor Netty 怎么做背压？**
  A：用 Reactive Streams 的 `request(n)` 机制，与 Netty 的 `isWritable`/`autoRead` 对接。

---

### 题 8.2　描述一个请求从"网卡收到包"到"业务返回响应"的完整链路。
**难度**：★★★　**类型**：综合题（考的是"能不能串起来"）

**答题主线**：**一条主线 + 每层的动作**。

```
① 网卡 DMA → ring buffer → 硬中断 → 软中断（ksoftirqd）→ 协议栈剥头 → socket 接收队列
   → sk_data_ready → ep_poll_callback → 就绪链表 → 唤醒 epoll_wait
② NioEventLoop.processSelectedKeys → NioByteUnsafe.read()
   → allocHandle.allocate（自适应 buffer）→ read() syscall → 数据进 ByteBuf
③ pipeline.fireChannelRead(buf)：decoder（拆帧）→ 业务 handler（处理）
④ 业务 handler 调 channel.writeAndFlush(resp)：
   → outbound handler（encoder 编码）→ head → ChannelOutboundBuffer.addMessage
   → doWrite → writev syscall → 数据进内核发送缓冲区
⑤ 内核 TCP：滑动窗口/拥塞控制 → 分段 → 网卡 DMA → 发出
   → 对端回 ACK → 本端内核释放发送缓冲区 → 后续 write 不再阻塞
⑥ ChannelFuture 完成（在 ④ 的 write 返回时，不是 ⑤）
```

**关键"钩子"（面试官会追问的点）**：
- ③ 的解码器如果发现半包 → 累积，**不 fireChannelRead**；
- ④ 的 `write` 与 `flush` 分离 → 可以批量；
- ⑤ 的 ACK 与 `ChannelFuture` **无关** → 业务要可靠必须自己做 ACK；
- 如果 ③ 的业务处理阻塞 → 整个 EventLoop 卡住（要隔离）。

---

### 题 8.3　Netty 服务里怎么隔离"快业务"和"慢业务"？
**难度**：★★★　**类型**：工程题

**答题主线**：
1. **IO 线程只做"快"的事**：解码、鉴权、路由、写回；
2. **慢业务用 `EventExecutorGroup`**：
```java
pipeline.addLast(bizGroup, "biz", new SlowHandler());
```
   - 同一 Channel 仍落到同一 executor（按 `channel.id()` hash），保持串行；
3. **更彻底的做法**：拆成独立服务（网关 + 业务服务），通过 RPC/MQ 通信；
4. **代价**：线程切换、内存可见性、调试变难。

**追问**：
- **Q：为什么不能直接 `new Thread` 处理？**
  A：连接数上万时线程爆炸；且丢失"同 Channel 串行"语义（要自己加锁）。
- **Q：`DefaultEventExecutorGroup` 的线程数怎么定？**
  A：按"慢业务的并发度/耗时"算：`线程数 ≈ QPS × 平均耗时(s)`（Little's Law），再加余量。

---

## 13.9 滴滴（偏"稳定性 + RPC 链路"）

### 题 9.1　一次 RPC 调用超时，你怎么从 Netty 层往上排查？
**难度**：★★★★　**类型**：故障排查

**答题主线**：**自下而上，逐层排除**。

| 层 | 检查点 | 工具 |
|---|---|---|
| 网络 | 丢包、RST、重传 | `tcpdump`、`ss -ti`（retrans、cwnd） |
| 连接 | 是否复用、是否被对端关闭 | `ss -tan state`、`channel.isActive()` |
| Netty IO | EventLoop 是否卡住 | `jstack` 看 `processSelectedKeys` 是否长时间在业务栈 |
| 编解码 | 是否半包堆积 | 监控 `cumulation` 大小、`maxFrameLength` |
| 业务 | 线程池队列、下游 DB/Redis | `jstack` 线程池、APM |
| GC | STW 停顿 | GC 日志、`-XX:+PrintGCApplicationStoppedTime` |

**高频根因 top5**：
1. **业务阻塞 IO 线程**（最常见）；
2. **GC STW**（大堆 + 频繁 Full GC）；
3. **连接池耗尽**（等待连接超时）；
4. **对端 accept 队列满**（`SO_BACKLOG` 太小）；
5. **Nagle + 延迟 ACK 的 40ms**（小包 RPC）。

---

### 题 9.2　Netty 长连接 vs 短连接，怎么选？连接池怎么设计？
**难度**：★★★　**类型**：架构题

| | 长连接 | 短连接 |
|---|---|---|
| 建连成本 | 一次（握手摊销） | 每次（三次握手 + 慢启动） |
| 资源占用 | 持续占 fd/内存 | 用完即释放 |
| 故障感知 | 需要心跳 | 天然（连不上就报错） |
| 适用 | 高频调用、推送、IM | 低频、突发、对外 API |

**连接池设计要点**：
- 每个 `EventLoop` 一个池（**避免跨线程**）；
- `maxConnections` + `pendingAcquireTimeout`；
- **健康检查**：`IdleStateHandler` + 定期 ping；
- **借出/归还**：注意 `Channel` 上的 `attr` 要清理（避免串状态）；
- **故障剔除**：`channelInactive` 时从池中移除。

---

## 13.10 网易（偏"游戏服务器 + 协议灵活性"）

### 题 10.1　游戏服务器的协议怎么设计？为什么用二进制而不是 JSON？
**难度**：★★★

**设计要点**：
1. **帧头**：`magic + version + cmd/msgId + seq + bodyLen`；
2. **body**：protobuf / flatbuffers / 自定义二进制；
3. **压缩**：大包用 snappy/lz4，小包不压（压缩本身有开销）；
4. **加密**：`SslHandler` 或自定义对称加密（游戏常用 XOR/RC4 变体，性能优先）。

**为什么二进制**：
- 体积小（JSON 有大量 key 和括号）；
- 解析快（无字符串解析）；
- 强类型（protobuf schema）。

**追问**：
- **Q：怎么支持"协议热更新"？**
  A：`cmd` 用整数而非字符串；handler 用 `Map<cmd, Handler>` 动态注册；pipeline 上可动态增删 handler。
- **Q：怎么处理"一个连接多种协议"（登录用 HTTP、游戏用自定义）？**
  A：pipeline 动态切换（`ChannelPipeline` 支持运行时 `addLast`/`remove`），或用 `PortUnificationHandler`（Netty 官方示例）根据首字节分流。

---

### 题 10.2　Pipeline 的"动态插拔"是怎么做到的？有什么坑？
**难度**：★★★

**机制**：`addLast` / `addFirst` / `addBefore` / `addAfter` / `remove` / `replace`，内部 `synchronized` + 维护双向链表。

**坑**：
1. **在 `channelRead` 里 `remove(this)`** → 当前事件继续按已取到的 `next` 传播，但被移除节点的后续逻辑不执行；
2. **`handlerAdded` 里再 `addLast`** → 可能死循环；
3. **跨线程 `addLast`** → 会投递到 eventLoop，但"什么时候生效"不确定；
4. **同一个非 `@Sharable` 实例加两次** → 抛异常。

**正确姿势**：用 `ChannelInitializer` 一次性装好；动态切换用"多个 handler 都装上，用状态标志位决定是否 `fireChannelRead`"，而不是频繁增删。

---

## 13.11 华为（偏"端到端全链路 + 高性能网络"）

### 题 11.1　从网卡收到一个包，到 Netty 的 `channelRead` 被调用，中间经历了什么？
**难度**：★★★★★　**类型**：全链路题（华为/字节的"看你到底懂多深"）

**答题主线**：**七个阶段，每个阶段给出"谁做的、开销在哪"**。

1. **网卡 → DMA → Ring Buffer**：DMA 拷贝，CPU 不参与；环形缓冲区满则丢包（`RX dropped`）。
2. **硬中断 → 软中断**：硬中断只做最少的事（记录、触发 `NET_RX_SOFTIRQ`）；软中断（`ksoftirqd`）做真正的协议处理。
3. **协议栈**：以太网头 → IP 头（校验、分片重组）→ TCP 头（序列号、乱序重组、ACK 生成）。
4. **socket 查找**：按四元组 hash 找到 `struct sock`，数据放入 `sk_receive_queue`。
5. **唤醒**：`sk_data_ready` → `sock_def_readable` → 遍历 `sk_wq` → `ep_poll_callback` → 挂 `rdllist` + 唤醒 `epoll_wait`。
6. **JDK**：`epoll_wait` 返回 → `EPollSelectorImpl` 填充 `selectedKeys` → `SelectionKey`。
7. **Netty**：`processSelectedKeys` → `unsafe.read()` → `read()` syscall（**数据从内核缓冲区拷到用户态 ByteBuf**）→ `pipeline.fireChannelRead`。

**关键强调**：
- **第 7 步才是真正的"数据拷贝到用户态"**，前面都是"数据在内核里搬"；
- 所以"零拷贝"能省的只有"内核内部拷贝"，**无法省掉最后一次"内核→用户"的拷贝**（除非用 `splice` 做中转，不到用户态）。

---

### 题 11.2　DPDK / 用户态协议栈和 Netty 是什么关系？什么场景会用？
**难度**：★★★★★　**类型**：架构延伸题

**DPDK**：绕过内核协议栈，在用户态用轮询（poll mode driver）直接收发包：
- **优势**：零中断、零拷贝、可绑核、极高吞吐（10M+ pps）；
- **代价**：独占 CPU 核、要自己实现 TCP/IP（或用 F-Stack/lwIP）、运维复杂。

**和 Netty 的关系**：
- Netty 是"内核 socket API 之上的框架"，DPDK 是"绕过内核"；
- 可以用 `netty-incubator-transport-*` 或自研 `EventLoop` 对接 DPDK/F-Stack；
- **绝大多数业务不需要 DPDK**（内核已经够快，瓶颈常在业务）。

**什么时候考虑**：
- 单机 > 100w pps、超低延迟（金融交易）、NFV/负载均衡设备；
- 否则**优先优化业务逻辑与 GC**。

---

## 13.12 小米 / OPPO / vivo（偏"移动端 IM + 弱网"）

### 题 12.1　移动端弱网环境下，长连接怎么保证消息可达？
**难度**：★★★

**要点**：
1. **多通道**：TCP 长连接 + HTTP 兜底（长连接断了走短连）；
2. **心跳自适应**：网络切换（WiFi↔4G）时立即重建；心跳间隔随网络质量调整；
3. **消息可靠**：本地队列 + 服务端 ACK + 重发（指数退避）；
4. **幂等**：`msgId` 去重；
5. **省电**：心跳合并、使用系统推送（厂商通道）替代长连接唤醒。

**追问**：
- **Q：Android 上长连接为什么容易被杀？**
  A：Doze 模式、后台限制、厂商省电策略。所以大厂都做"自建长连接 + 厂商推送兜底"。

---

## 13.13 快问快答 20 连（面试官用来"扫盲区"）

| # | 问题 | 要点 |
|---|---|---|
| 1 | Netty 的 `Bootstrap` 和 `ServerBootstrap` 区别 | 客户端 vs 服务端；后者多 `childHandler`/`childOption` |
| 2 | `option` 与 `childOption` 区别 | listen fd vs 每个新连接 fd |
| 3 | `Channel` 的 `attr()` 是什么 | `AttributeKey` + `AttributeMap`，随 channel 生命周期 |
| 4 | `ChannelGroup` 有什么用 | 批量操作（广播/关闭），线程安全 |
| 5 | `ChannelHandlerContext` 能拿到什么 | channel、pipeline、executor、handler、prev/next |
| 6 | `SimpleChannelInboundHandler` 的好处 | 自动 release，避免泄漏 |
| 7 | `ByteToMessageDecoder` 什么时候调用 `decode` | 每次有新数据且 `readableBytes > 0` |
| 8 | `MessageToByteEncoder` 为什么要求泛型 | 只处理匹配类型，其他透传 |
| 9 | `IdleStateHandler` 构造参数 | readerIdle, writerIdle, allIdle, unit |
| 10 | `SslHandler` 加在哪 | pipeline 第一个（靠近 socket） |
| 11 | `writeAndFlush` 返回什么 | `ChannelFuture` |
| 12 | `ChannelFuture.sync()` 能在 IO 线程用吗 | **不能**，会抛 `BlockingOperationException` |
| 13 | `EventLoopGroup` 的 `next()` 做什么 | 轮询选下一个 EventLoop |
| 14 | `NioEventLoop` 的线程何时创建 | **懒启动**，第一次 `execute`/`register` |
| 15 | `selector.selectNow()` 与 `select()` 区别 | 非阻塞 vs 阻塞（带超时） |
| 16 | `SelectionKey` 的 `attachment` 是什么 | 注册时传入的对象（Netty 里是 `AbstractNioChannel`） |
| 17 | `ChannelOutboundBuffer` 三个指针 | flushedEntry / unflushedEntry / tailEntry |
| 18 | `ByteBuf` 的 `maxCapacity` 默认值 | `Integer.MAX_VALUE`（可显式设小） |
| 19 | `PooledByteBufAllocator` 默认 chunk 大小 | 16MB（pageSize 8K × 2^11） |
| 20 | Netty 4 与 3 的最大变化 | 内存管理重构、`Channel` 统一 API、`Future` 体系 |

---

# 第十四部分　自测清单与学习路径

## 14.1 三级自测：你现在在哪个档位？

### L1　初级（能答 8/10 → 可以写 Netty 业务代码）

- [ ] 能说出 Netty 解决 JDK NIO 的 5 个问题
- [ ] 能画 pipeline 的双向链表，说明入站/出站方向
- [ ] 知道 `channel.writeAndFlush` 是异步的，返回 `ChannelFuture`
- [ ] 能写出"长度字段"半包处理的代码
- [ ] 知道 `ByteBuf` 的 `readerIndex`/`writerIndex`/`capacity`
- [ ] 知道必须 `release`，`SimpleChannelInboundHandler` 会自动释放
- [ ] 会用 `IdleStateHandler` 做心跳
- [ ] 知道 `option` 与 `childOption` 的区别

### L2　中级（能答 9/12 → 可以独立设计一个网关/RPC 传输层）

- [ ] 能讲清 EventLoop 的 `run()` 四阶段
- [ ] 能解释"为什么 select 阻塞时能被任务唤醒"
- [ ] 能说清 `ctx.write` 与 `channel.write` 的起点差异及后果
- [ ] 能解释"为什么不能一直注册 OP_WRITE"
- [ ] 能说清水位线与 `isWritable`/`autoRead` 的配合
- [ ] 能说出 `PoolArena`/`PoolChunk`/`PoolSubpage` 各自解决什么
- [ ] 能解释引用计数的 CAS 实现与泄漏检测原理
- [ ] 知道 `ioRatio` 的计算公式
- [ ] 能讲 `bind` 的 syscall 序列
- [ ] 知道业务阻塞要 `EventExecutorGroup` 隔离
- [ ] 能说出 `TIME_WAIT` 与 `CLOSE_WAIT` 的不同含义
- [ ] 知道 Netty 定时任务用优先队列不是时间轮

### L3　高级（能答 10/12 → 可以定级 P7/专家，能主导中间件选型）

- [ ] 能讲 epoll 的 `eventpoll`/`epitem`/`rdllist` 与回调机制
- [ ] 能说清 LT/ET 的"状态跃迁"本质，以及 Netty 如何用 LT 模拟 ET
- [ ] 能讲 JDK `EPollSelectorImpl` 的 `eventfd` 唤醒原理
- [ ] 能讲空轮询 Bug 与 512 次重建的细节
- [ ] 能说清两种"零拷贝"的区别与 `FileRegion` + `SslHandler` 的冲突
- [ ] 能讲 `PoolChunk` 完全二叉树（及新版 pageMap+runs）的分配/释放
- [ ] 能算"单机 10w 连接需要多少内存"
- [ ] 能设计"百万在线"的分层架构并给出取舍
- [ ] 能说出 `FastThreadLocal` 为什么严格 O(1)
- [ ] 能解释 `SelectedSelectionKeySet` 替换 JDK HashSet 的动机与代价
- [ ] 能定位"CPU 100% 但 QPS 低"的 5 种根因
- [ ] 能讲 `FileRegion`/`writev`/`sendfile` 在网关中的实际收益

> **自测方法**：遮住答案，用"讲给别人听"的方式口述 3 分钟；讲不顺的地方就是你的知识缺口（不是"看过"就等于"会"）。

## 14.2 源码阅读顺序（不要一上来啃 `NioEventLoop`）

很多人读 Netty 源码失败，是因为**从最难的地方开始**。正确顺序是"从数据流向倒推"：

| 顺序 | 入口类 | 关注什么 | 预计耗时 |
|---|---|---|---|
| ① | `ByteBuf` / `UnpooledByteBuf` | 读写指针、`ensureWritable`、`slice` | 1 天 |
| ② | `DefaultChannelPipeline` / `AbstractChannelHandlerContext` | 双向链表、`findContextInbound/Outbound`、`invokeChannelRead` | 1~2 天 |
| ③ | `AbstractBootstrap` / `ServerBootstrap` | `initAndRegister`、`doBind0`、`ServerBootstrapAcceptor` | 1 天 |
| ④ | `AbstractNioChannel` / `NioSocketChannel` | `doRegister`、`doBeginRead`、`doReadBytes`、`doWrite` | 2 天 |
| ⑤ | `NioEventLoop` / `SingleThreadEventExecutor` | `run()`、`select()`、`runAllTasks()`、`wakeup()` | 2~3 天 |
| ⑥ | `PooledByteBufAllocator` / `PoolArena` / `PoolChunk` | 分配/回收全链路、`PoolThreadCache` | 3~5 天（最难） |
| ⑦ | `ByteToMessageDecoder` / `LengthFieldBasedFrameDecoder` | `cumulation`、`callDecode` | 1 天 |
| ⑧ | `IdleStateHandler` / `HashedWheelTimer` | 定时任务的两种实现 | 1 天 |

**阅读技巧**：
1. **带着问题读**（如"半包时数据存哪"），不要通读；
2. **打断点**（IDEA 里对 `channelRead` 打断点，看调用栈，一次就能看清全链路）；
3. **看 `git blame` 和 issue**（很多"奇怪"的设计背后有一个 bug 号）；
4. **画时序图**（把一次 read 的调用栈画成图，比看十遍代码有用）。

## 14.3 全文因果链（一张图收束）

```
【内核】数据到 socket 缓冲区 → sk_data_ready → ep_poll_callback → rdllist → 唤醒 epoll_wait
   ↓
【JDK】epoll_wait 返回 → EPollSelectorImpl 填 selectedKeys（Netty 换成数组）
   ↓
【Netty-1】EventLoop 的 run()：select → processSelectedKeys → runAllTasks（ioRatio 控预算）
   ↓
【Netty-2】unsafe.read()：自适应 buffer → read() syscall → ByteBuf（池化、引用计数）
   ↓
【Netty-3】pipeline.fireChannelRead：解码器拆帧（半包累积）→ 业务 handler
   ↓
【Netty-4】业务 writeAndFlush：encoder → ChannelOutboundBuffer 入队 → 水位线判定
   ↓
【Netty-5】flush → doWrite → writev syscall → 写不动则注册 OP_WRITE（等可写）
   ↓
【内核】TCP 分段/窗口/拥塞控制 → 网卡 DMA → 发出 → 对端 ACK
```

**每一层都有一个"不得不这样"的理由**，而面试问的每一个"为什么"，答案都在这条链上：

| 现象 | 因果链上的原因 |
|---|---|
| 必须处理半包 | 内核不保留消息边界（第五部分） |
| 必须池化 | 每次 read 都要一块 buffer（第九部分） |
| 必须串行 | 避免多线程抢同一 Channel 的状态（第七部分） |
| 只在需要时注册 OP_WRITE | 避免 epoll 退化成忙等（第七部分） |
| 必须 wakeup | epoll_wait 不知道"用户态有任务"（第三部分） |
| 512 次重建 Selector | JDK epoll 实现的 bug（第三部分） |
| 心跳必须应用层做 | TCP keepalive 2 小时且不能发现"假死"（第五部分） |

## 14.4 面试表达模板（技术题通用）

**四段式**（对任何"为什么这么设计"的题都适用）：

1. **结论前置**：一句话给出答案（"因为 A，所以 B"）；
2. **机制展开**：2~3 个关键步骤/组件，带**具体名字**（类名、syscall、参数）；
3. **依据/证据**：源码行号、内核数据结构、或"我压测时观察到 X"；
4. **代价与取舍**：这个设计的成本是什么、什么场景不适用。

**加分动作**：
- **给数字**（"默认 64KB 高水位"、"2×CPU"、"16MB chunk"、"512 次重建"）——数字证明你真读过；
- **主动说坑**（"这个方案在 XX 场景会失效"）——证明你有工程经验；
- **画图**（哪怕在纸上画个链表）——比说十句话有效。

**减分动作**：
- 只背结论不给原因；
- 把"零拷贝"、"时间轮"、"ET"说错（这三处最容易露馅）；
- 说"我们用的是 XXX 框架封装的，没关注底层"。

## 14.5 学习资料（按可信度排序）

| 类型 | 名称 | 用途 |
|---|---|---|
| **源码** | Netty 4.1.x `io.netty.channel.nio` / `io.netty.buffer` | 第一手，永远最准 |
| **官方文档** | Netty 官网 `user-guide` / `api` | 架构与 API 语义 |
| **书** | 《Netty in Action》（Norman Maurer） | 体系化入门 |
| **书** | 《Netty 权威指南》（李林锋） | 中文，源码分析细 |
| **内核** | `man 7 epoll`、`man 2 sendfile` | syscall 语义 |
| **内核源码** | `fs/eventpoll.c`、`net/ipv4/tcp_input.c` | 想答"内核级"必看 |
| **JDK 源码** | `sun.nio.ch.EPollSelectorImpl` / `EPollArrayWrapper` | 理解 Netty 的"上游" |
| **经典文章** | LWN 的 epoll 系列、Dan Kegel 的 "The C10K problem" | 建立 I/O 模型世界观 |

## 14.6 结语：这份文档该怎么用

1. **第一遍**：按顺序读第一~十二部分，建立"因果链"，不要背结论；
2. **第二遍**：只看每节的"面试加分点/高分细节"，抄成自己的话；
3. **第三遍**：拿第十三部分的题目自测，**遮住答案口述**；
4. **实战**：找一个真实场景（自己写个简易 RPC / 网关），把"半包、心跳、背压、优雅关闭"全部实现一遍——**踩过的坑才是你的**。

> Netty 的知识密度很高，但它**不是靠记忆**，而是靠一条"内核 → JDK → Netty → 业务"的因果链。把这条链打通，你会发现面试官问的每一个"为什么"，你都能从下游一层层推上去。
>
> **真正的区分度不在于"你知道 EventLoop 有几个阶段"，而在于"你能说清每个阶段为什么必须存在"。**

---

*文档版本：v1.0 ｜ 适用 Netty 4.1.x / JDK 8~21 ｜ 内容含公开面经归纳，仅供学习参考*

