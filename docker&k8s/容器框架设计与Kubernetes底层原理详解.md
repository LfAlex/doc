# 容器框架设计与 Docker / Kubernetes 底层原理详解

> 一份从「进程隔离」讲到「集群编排」的深度笔记，附国内大厂面试题库（含考核程度分级与详解）。

---

## 阅读说明

**内容组织逻辑**：由浅入深，每一章都遵循同一个范式：

```
一句话本质 → 结构图 → 内核/代码层面的机制 → 工程上的坑与结论
```

**难度标识**（贯穿全文与题库）：

| 标识 | 含义 | 面向 |
|---|---|---|
| ⭐ | 入门（会背概念即可） | 校招初面 |
| ⭐⭐ | 基础（要能说清机制） | 社招一面 |
| ⭐⭐⭐ | 进阶（要能说清内核/源码行为） | 社招二面、P6/P7 |
| ⭐⭐⭐⭐ | 高阶（要能给出设计取舍与踩坑经验） | 专家岗、架构面 |
| ⭐⭐⭐⭐⭐ | 压轴（工程体系级问题，无标准答案） | 技术终面、TL 面 |

**关键认知前提**（先记住这一句，后面所有内容都是它的展开）：

> **容器不是虚拟化，容器是被 Linux 内核"隔离了视野、限制住资源、固定住根目录"的一个普通进程。**
> Docker 没有创造隔离技术，它创造的是**镜像的分层打包与分发格式**，以及**把复杂的 namespace/cgroup/rootfs 操作标准化成一条命令**。

---

## 第 0 章 全局地图：一条 `docker run` 到底发生了什么

这是全书的"总索引"。先看全景，再逐层拆。

### 0.1 全景调用链

```
你敲下：docker run -d -p 8080:80 --memory 512m nginx:latest

┌──────────────────────────────────────────────────────────────────────┐
│ ① CLI 层                                                            │
│    /usr/bin/docker (Go 写的客户端)                                    │
│    → 把命令转成 REST 请求 POST /containers/create                     │
│    → 通过 Unix Domain Socket: /var/run/docker.sock 发给 dockerd      │
│    （这就是为什么"挂载 docker.sock 等于给了宿主机 root"）              │
└──────────────────────────────────────────────────────────────────────┘
                                  ↓ HTTP/1.1 over UDS
┌──────────────────────────────────────────────────────────────────────┐
│ ② dockerd（Docker Daemon，守护进程）                                  │
│    · 解析请求，做 API 层校验                                          │
│    · 镜像管理：本地有 nginx:latest 吗？没有 → 去 registry 拉取         │
│      (manifest → config → 逐层 layer 下载 → 解压到                     │
│       /var/lib/docker/overlay2/<layer-id>/diff)                       │
│    · 网络：分配 IP、创建 veth pair、接 docker0、写 iptables DNAT      │
│    · 存储：创建/挂载 volume，准备容器 rootfs（overlay2 的 merged 目录）│
│    · 生成 OCI runtime spec（config.json）：namespaces、cgroups 资源、  │
│      mounts、capabilities、seccomp、env、entrypoint...                │
│    · 通过 gRPC 调用 containerd：Create / Start                        │
└──────────────────────────────────────────────────────────────────────┘
                                  ↓ gRPC (containerd)
┌──────────────────────────────────────────────────────────────────────┐
│ ③ containerd（高层运行时 / High-level Runtime）                       │
│    · 镜像的 pull / unpack / snapshotter（真正管镜像落地）             │
│    · 容器生命周期管理，把 OCI spec 落成 bundle 目录                    │
│      /run/containerd/io.containerd.runtime.v2.task/default/<id>/     │
│       ├── config.json    ← OCI runtime spec                          │
│       └── rootfs/        ← 指向 overlay2 的 merged 目录（mount）       │
│    · fork 出 containerd-shim-runc-v2，把容器交给它托管                │
└──────────────────────────────────────────────────────────────────────┘
                                  ↓ exec
┌──────────────────────────────────────────────────────────────────────┐
│ ④ containerd-shim（垫片进程，每个容器一个）                           │
│    · 为什么要它？                                                        │
│      a) 让 containerd 重启/升级时容器不受影响（shim 是容器父进程）      │
│      b) 接管容器的 stdio（stdout/stderr 的 pipe）与退出码回收          │
│      c) 通过 ttrpc 向 containerd 汇报 Task 状态                        │
└──────────────────────────────────────────────────────────────────────┘
                                  ↓ fork+exec
┌──────────────────────────────────────────────────────────────────────┐
│ ⑤ runc（底层运行时 / Low-level Runtime，遵循 OCI runtime-spec）       │
│    runc create 阶段（这是真正"造容器"的地方）：                        │
│    1. 读取 config.json                                                │
│    2. 准备 rootfs：mount overlay（lowerdir=各镜像层, upperdir=容器层） │
│    3. clone() 系统调用，一次性带上所有 CLONE_NEW* 标志 → 新进程诞生   │
│    4. 在子进程里：写 cgroup 文件（cpu.max / memory.max / pids.max）   │
│    5. pivot_root 切换根目录，umount 掉旧根（防止 chroot 逃逸）         │
│    6. mount /proc /sys /dev /dev/pts /dev/shm（注意 /proc 必须重建，   │
│       因为它是和 PID namespace 绑定的）                               │
│    7. 设置 rlimits、capabilities bounding set、seccomp filter、       │
│       no_new_privs、UID/GID、环境变量、工作目录                        │
│    8. 写好 sync pipe，等 `runc start` → execve("/docker-entrypoint.sh")│
│    9. runc 主进程退出，容器进程被 shim（subreaper）收养                │
└──────────────────────────────────────────────────────────────────────┘
                                  ↓
┌──────────────────────────────────────────────────────────────────────┐
│ ⑥ Linux 内核（真正的"容器"在这里成型）                                │
│    · 8 种 namespace  → 看不见别的进程/网络/挂载点                     │
│    · cgroup v1/v2     → 用不到超过 512M 内存、0.5 个 CPU               │
│    · OverlayFS        → / 看起来是完整的 nginx 系统                   │
│    · capabilities/seccomp/LSM → 即使 root 也干不了危险事               │
└──────────────────────────────────────────────────────────────────────┘
```

### 0.2 记住这张"三层抽象"图

```
        ┌───────────────────────────────────────────┐
        │ 编排层 Orchestration                       │  Kubernetes / Swarm / Nomad
        │  解决：调度、伸缩、自愈、服务发现、滚动发布  │
        ├───────────────────────────────────────────┤
        │ 运行时层 Runtime                           │  Docker / containerd / CRI-O / Podman
        │  解决：镜像格式、生命周期、分发、CRI 接口    │
        ├───────────────────────────────────────────┤
        │ 隔离层 Isolation (Linux Kernel)            │  namespace / cgroup / LSM / Capabilities
        │  解决：进程看见什么、能用多少资源、能做什么  │
        └───────────────────────────────────────────┘
```

**面试高频对比**：虚拟化 vs 容器

| 维度 | 虚拟机（VM） | 容器 |
|---|---|---|
| 隔离单位 | 硬件级（Hypervisor 模拟 CPU/内存/设备） | 进程级（内核数据结构） |
| 内核 | 每个 VM 一套独立内核 | 共享宿主内核 |
| 启动耗时 | 秒~分钟（要 boot 内核） | 毫秒~百毫秒（只是 fork 一个进程） |
| 开销 | 额外内核 + 完整 OS，内存 GB 级 | 仅镜像层占用，近乎零额外开销 |
| 隔离强度 | 强（硬件边界） | 较弱（共享内核，逃逸面大） |
| 密度 | 单机几十个 | 单机几百上千个 |
| 典型逃逸 | 极少（需要 Hypervisor 漏洞） | 较多（内核漏洞 / 特权容器 / 配置错误） |

> **一句话总结**：VM 隔离的是"机器"，容器隔离的是"视野"。因此容器的安全性天然弱于 VM，所以生产上出现了 **gVisor / Kata Containers / Firecracker** 这类"用 VM 的强隔离去跑容器镜像"的折中方案（见第 10 章）。

---

## 第 1 章 容器的本质：隔离的进程

### 1.1 从 chroot 到容器（历史脉络帮你理解"每个技术解决什么问题"）

```
1979  chroot(2)        → 只能改根目录，可被 chdir 到保存的 fd 逃逸，且不隔离 mount
2000  FreeBSD Jail     → 第一个"完整"的轻量隔离（pid/网络/文件系统）
2005  Solaris Zones    → 商业化的容器雏形
2002  Mount Namespace  → Linux 2.4.19 引入，第一个 namespace
2006  UTS / IPC NS     → 2.6.19
2008  PID / NET NS     → 2.6.24
2008  LXC               → 第一个把 cgroups+ns 打包成"容器"的工具（但难用）
2008  Cgroups v1        → 2.6.24，Google 工程师 Paul Menage / Rohit Seth 提交
2013  Docker            → dotCloud 的 Solomon Hykes，把 LXC 换成了 libcontainer，
                          并补齐了"镜像分层 + Registry"这个关键拼图
2013  Cgroups v2        → 起步（3.10 初版，4.5 可用，4.15/5.x 逐步完备）
2015  OCI 成立           → Docker、CoreOS、Google 等成立 Open Container Initiative
2016  containerd 剥离   → Docker 把运行时拆出去捐给 CNCF
2017  CRI / CRI-O       → Kubernetes 与 Docker 解耦
2020  K8s 宣布弃用 dockershim
2022  K8s 1.24 正式移除 dockershim → 生产上只剩 containerd / CRI-O
```

**关键洞察**：Docker 之所以赢 LXC，不是因为隔离做得更好，而是因为：
1. **镜像分层 + 内容寻址（content-addressable）**——可复用、可校验、可增量分发；
2. **Dockerfile**——把"环境搭建"变成了可版本控制的代码；
3. **Registry + `docker pull`**——一条命令获得完整运行环境，彻底消灭"我这能跑"。

### 1.2 Namespace：容器"看不见"别人的原理

**一句话本质**：Namespace 是内核给"全局资源"做的**多份命名视图**。同一份内核资源（进程表、网络栈、挂载树）在不同 namespace 里呈现为不同的实例，进程只能看到自己所属 namespace 的那一份。

**关键认知**：namespace 不是"把资源切开"，而是"给同一类资源起了多个名字空间"。内核内部真正做的是**在 task_struct 里挂一个 nsproxy 指针**，所有"查找资源"的系统调用都先经过 `task->nsproxy->xxx_ns` 去解析 ID。

```c
// 简化示意（kernel/nsproxy.c）
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net           *net_ns;
    struct time_namespace *time_ns;
    struct time_namespace *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;
};

// 进程在 fork/clone 时，若带 CLONE_NEW* 标志，就 copy 一份新的 ns 而不是共享
```

#### 1.2.1 八种 Namespace 全表

| Namespace | clone 标志 | 隔离内容 | 内核版本 | 容器里体现为 |
|---|---|---|---|---|
| Mount (mnt) | `CLONE_NEWNS` | 挂载点视图、挂载传播 | 2.4.19 | 独立 rootfs、独立 `/proc` |
| UTS | `CLONE_NEWUTS` | hostname / NIS domain | 2.6.19 | 容器有自己的 hostname |
| IPC | `CLONE_NEWIPC` | System V IPC、POSIX mq | 2.6.19 | 独立共享内存/信号量 |
| PID | `CLONE_NEWPID` | 进程 ID 空间 | 2.6.24 | 容器内 PID 从 1 开始 |
| Network (net) | `CLONE_NEWNET` | 网卡、路由、iptables、端口空间 | 2.6.24 (完整 2.6.29) | 独立 eth0 / 独立 IP / 端口不冲突 |
| User | `CLONE_NEWUSER` | UID/GID 映射 | 3.8 | rootless 容器、容器内 root ≠ 宿主 root |
| Cgroup | `CLONE_NEWCGROUP` | cgroup 根视图 | 4.6 | 容器内看不到宿主 cgroup 树 |
| Time | `CLONE_NEWTIME` | CLOCK_MONOTONIC / BOOTTIME 偏移 | 5.6 | 支撑 CRIU 热迁移 |

#### 1.2.2 逐个深入（挑最有面试价值的讲）

**① PID Namespace —— 为什么容器里 PID 1 很特殊**

PID namespace 是**树状层级结构**：父 namespace 能看到子 namespace 的所有进程（但 PID 不同），子 namespace 看不到父的进程。

```
Host PID NS (PID: 1 systemd, 1000 dockerd, 2000 shim, 2001 nginx ...)
        │
        ├── Container A PID NS:   1 → 宿主 2001 (nginx master)
        │                         2 → 宿主 2010
        │
        └── Container B PID NS:   1 → 宿主 3001
```

- **一个进程在每层 ns 里有不同 PID**，`/proc` 显示的是当前 ns 视角的 PID。
- **PID 1 的特殊性**（这就是容器里"僵尸进程"问题的根源）：
  1. 内核**不会**对 PID 1 施加默认信号行为——发给 PID 1 的 `SIGTERM`/`SIGKILL`，如果 PID 1 没有注册 handler，**内核直接忽略**。→ 所以 `docker stop` 有时会等满 10 秒然后才 `SIGKILL`。
  2. PID 1 是**孤儿进程的收养者**：容器里任何进程的父进程死了，孤儿会挂到 PID 1 下，PID 1 必须负责 `wait()` 回收，否则产生僵尸。
  - 解决：`docker run --init`（注入 tini）、K8s 里用 `shareProcessNamespace: false` 时业务容器要自己处理；或让 entrypoint 用 `exec` 形式（`exec java ...`）而不是 `shell` 形式（`sh -c "java ..."`，否则 PID 1 是 sh，java 收不到 SIGTERM）。

- **`CLONE_NEWPID` 的坑**：`clone()` 带这个标志创建的子进程**自己不在新的 PID ns 里**，只有它 **fork 出来的子孙**才进入新 ns。这就是为什么 `unshare --pid --fork` 必须带 `--fork`。

**② Mount Namespace —— 容器 rootfs 的舞台**

- mount namespace 隔离的是**挂载树**。创建新 mount ns 时，子 ns 会 copy 一份父 ns 的挂载树（**copy-on-write 语义的"挂载点副本"**）。
- **挂载传播（mount propagation）**：
  | 类型 | 行为 | 容器里的用法 |
  |---|---|---|
  | `shared` | 双向传播（宿主的挂载变化会传到容器，反之亦然） | 默认；**docker 会把 / 设为 shared 再给容器设 private** |
  | `private` | 不传播 | 容器默认 |
  | `slave` | 单向：宿主 → 容器 | 需要宿主挂载透传给容器时 |
  | `unbindable` | 不可被 bind mount 引用 | 极少用 |
  - 这就是 `-v /host/path:/container/path` 在宿主上重挂载后容器能看到的原因（`rshared`）。
- **`/proc` 必须重新挂载**：`/proc` 的内容是相对 PID namespace 动态生成的。如果容器直接继承宿主的 `/proc` 挂载，容器内会看到宿主所有进程 → 隔离失效。所以 runc 一定执行 `mount -t proc proc /proc`。
- 同理 `/sys`（cgroup 视图相关）、`/dev`（需要 devpts、shm、mqueue）。

**③ Network Namespace —— 一个容器一套完整网络栈**

隔离内容极其彻底：**网卡（除 loopback 且 lo 默认 down）、路由表、ARP 表、iptables/nftables 规则、netfilter conntrack、socket 端口空间、/proc/net、/sys/class/net**。

- 关键推论：**不同 net namespace 的进程可以监听同一个端口**（端口空间独立）。这就是"容器端口与宿主端口不冲突"的原理，也解释了为什么 `-p` 是必须的——不加 `-p`，容器 80 端口只在容器自己的 netns 里有效。
- 因为进 netns 需要 `CAP_SYS_ADMIN`（或 user ns 下的权限），所以容器内**看不到也改不了**宿主 iptables。

**④ User Namespace —— rootless 容器的基石**

- 映射关系写在 `/proc/<pid>/uid_map`、`gid_map`：
```
         0          100000          65536
   （容器内UID）（宿主UID起点）  （映射范围）
```
即：容器内 uid 0（root）→ 宿主 uid 100000（普通用户）。
- **效果**：容器内是 root，但所有文件操作、系统调用在宿主看来都是 uid 100000 干的。→ 即使挂载 `docker.sock` 或拿到 root，也无法真正控制宿主。
- **限制**：user ns 里的 root 不能 `mknod` 任意设备（除白名单）、不能 mount 非白名单文件系统、不能访问未映射的 UID 拥有的文件。
- **Rootless Docker / Podman** 就靠这个；`/etc/subuid`、`/etc/subgid` 配置映射区间。

**⑤ IPC Namespace**：System V 共享内存/信号量/消息队列 + POSIX mq。K8s Pod 内容器**共享** IPC ns（`shareProcessNamespace` 无关，Pod 内默认共享 IPC），这是 Pod 内可以用 SysV IPC 通信的原因。

**⑥ Time Namespace**（5.6，很新，面试加分项）：偏移 `CLOCK_MONOTONIC` 和 `CLOCK_BOOTTIME`，主要用于 **CRIU（Checkpoint/Restore In Userspace）** 做容器热迁移——恢复时把时钟"倒回"到 checkpoint 时刻。`CLOCK_REALTIME` 不受影响（要全局一致）。

#### 1.2.3 如何"看见"和实践这些 namespace

```bash
# 看进程属于哪些 namespace（inode 号相同即共享同一 ns）
ls -l /proc/$$/ns
# lrwxrwxrwx 1 root root 0 ... cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 root root 0 ... ipc    -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 root root 0 ... mnt    -> 'mnt:[4026531840]'
# lrwwrxrwx 1 root root 0 ... net    -> 'net:[4026531992]'
# ... pid -> 'pid:[4026531836]'  ...

# 手工造一个"容器"（不借助 docker）
unshare --uts --pid --net --mount --fork --user --map-root-user bash
# 此刻你已经在一套独立 ns 里了，但还没有 rootfs、没有资源限制 → 只能算"半个容器"

# 进入某个容器的 namespace 排障（诊断神器）
nsenter -t $(docker inspect -f '{{.State.Pid}}' nginx) -n -m -p -u -i bash
# 等价于 docker exec，但 docker exec 走的是 CRI/exec 路径，nsenter 更"底层"
```

对比 `docker exec` vs `nsenter`：
| | docker exec | nsenter |
|---|---|---|
| 依赖 | daemon 存活、容器状态正常 | 只需知道宿主机 PID |
| 会创造新进程 | 是（在容器 ns 里 fork） | 是 |
| 需要的能力 | 只依赖 dockerd 访问权限 | 需要宿主 root |
| 典型场景 | 常规运维 | daemon 挂了/容器僵死时的抢救 |

---

### 1.3 Cgroups：容器"用不了超过这么多"的原理

**一句话本质**：Cgroups（Control Groups）是内核的**资源计量与限制框架**。它把一组进程组织成树状层级，在每个节点上挂载"控制器（controller/subsystem）"，对组内所有进程（含未来 fork 出的子孙，自动继承）施加统一限制。

**关键认知**：cgroup 的两个作用必须区分开——
1. **限制（limit）**：超过就限流/杀进程（`memory.max`、`cpu.max`、`pids.max`）。
2. **统计（accounting）**：`memory.current`、`cpu.stat`、`memory.stat` → 这是容器监控（cAdvisor、Prometheus）的数据源。

#### 1.3.1 v1 vs v2：为什么要重构

```
【v1：每个控制器一棵独立的树 —— 混乱之源】

  /sys/fs/cgroup/cpu/           /sys/fs/cgroup/memory/
        ├── docker/                  ├── docker/
        │     ├── <cid>/             │     ├── <cid>/
        │     └── <cid2>/            │     └── <cid2>/
        └── system/                  └── system/

  问题：
  1. 一个进程可以在 cpu 树里属于组 A，在 memory 树里属于组 B → 语义冲突
  2. 无法原子地"把整个容器迁到另一组"
  3. 各控制器特性重复实现（如 freezer 在 v1 是独立控制器）
  4. 层级数量无上限，命名空间混乱，难以做统一权限

【v2：单一层级 + 控制器可下放 —— 统一模型】

  /sys/fs/cgroup/  (统一挂载, cgroup2)
        ├── system.slice/
        │     ├── docker-<cid>.scope/     ← 一个容器一个 cgroup
        │     │     ├── cgroup.controllers      (可用控制器列表)
        │     │     ├── cgroup.subtree_control  (向子节点启用的控制器)
        │     │     ├── memory.max  memory.high  memory.current
        │     │     ├── cpu.max     cpu.weight   cpu.stat
        │     │     ├── pids.max    io.max
        │     │     └── cgroup.procs
        └── user.slice/
```

**v2 的核心规则（面试必问）**：
1. **单一层级（single hierarchy）**：所有控制器在同一棵树上按需启用，进程归属唯一。
2. **no internal process 约束**：**一个 cgroup 只要向子节点下放了控制器（`cgroup.subtree_control` 非空），它自己就不能直接挂进程**（只能挂叶子）。这是 v2 最反直觉的规则，也避免了"父子同时管进程"的计量歧义。
3. **`cgroup.controllers` / `cgroup.subtree_control`**：父节点声明"我有哪些控制器可用"和"我给子节点开哪些"。子节点只有被 enable 的控制器才出现对应文件。
4. **`cgroup.stat`、`cgroup.events`（`populated`、`frozen`）** 提供统一的观测/冻结语义（替代 v1 freezer）。
5. **PSI（Pressure Stall Information）**：`memory.pressure`、`cpu.pressure`、`io.pressure` —— v2 才有，能测量"因资源不足导致的停滞时间"，是容器排障的利器。

#### 1.3.2 控制器与关键参数对照表（v1 → v2 → Docker/K8s）

| 资源 | cgroup v1 文件 | cgroup v2 文件 | docker 参数 | K8s 对应 |
|---|---|---|---|---|
| CPU 配额 | `cpu.cfs_quota_us` / `cpu.cfs_period_us` | `cpu.max`（"$MAX $PERIOD"） | `--cpus=0.5` → `cpu.max = 50000 100000` | `limits.cpu` |
| CPU 权重 | `cpu.shares`（默认 1024） | `cpu.weight`（默认 100，范围 1–10000） | `--cpu-shares` | `requests.cpu` |
| CPU 绑核 | `cpuset.cpus` / `cpuset.mems` | 同（`cpuset.cpus` / `cpuset.mems`） | `--cpuset-cpus` | `cpu manager` static policy |
| 内存硬限 | `memory.limit_in_bytes` | `memory.max` | `--memory` | `limits.memory` |
| 内存软限 | `memory.soft_limit_in_bytes` | `memory.high` | `--memory-reservation` | — |
| 内存当前 | `memory.usage_in_bytes` | `memory.current` | `docker stats` | `container_memory_working_set_bytes` |
| Swap | `memory.memsw.limit_in_bytes` | `memory.swap.max` | `--memory-swap` | — |
| 进程数 | `pids.max` | `pids.max` | `--pids-limit` | `podPidsLimit` |
| 块 IO | `blkio.throttle.read_bps_device` | `io.max`（"$MAJ:$MIN rbps=…"） | `--device-read-bps` | — |
| 冻结 | `freezer.state`（FROZEN/THAWED） | `cgroup.freeze`（0/1） | `docker pause` | — |

**内存的几个细节（深挖点）**：

- `memory.max` 是**硬限**：达到后内核先尝试**直接回收（direct reclaim）**（把文件页 page cache 丢掉），如果还是回收不出来，就触发 **cgroup 级 OOM killer**，杀组内 `oom_score_adj` 最高的进程 → 这就是 `exit code 137`（128+9=SIGKILL）和 `OOMKilled: true`。
- `memory.current` **包含 page cache**（属于该 cgroup 的文件缓存）。所以"容器内存看着很高但其实没泄漏"很常见 → 要看 `memory.stat` 里的 `anon`（堆/栈/匿名页）和 `file`（缓存）。
- 排查内存必须用 **working set** 而不是 RSS：`working_set = 匿名页 + 活跃文件页 + 不可回收的 slab`，这是 K8s 驱逐判定（`memory.available = capacity - working_set`）的口径。
- `memory.high` 是**软限**：超过后内核进入 `throttle` 态，强制回收并让进程变慢（不杀），v2 独有，适合做"软隔离防抖"。
- `oom_score_adj`：`-1000` 表示永不被 OOM 杀；K8s Guaranteed QoS 的 Pod 会得到 `-997`（保证关键 Pod 优先存活）。这个值由 QoS 决定 —— **这才是 QoS 的真实价值，不只是"调度标签"**。

**CPU 的几个细节**：

- `cpu.max = 50000 100000` 表示 100ms 周期内最多用 50ms CPU 时间 → 等价于 `--cpus=0.5`。
- **CPU 限流（throttling）的坑**：limits 设得太死（如 0.1 核），进程每 100ms 只能跑 10ms，其余 90ms 被移出运行队列 → 延迟抖动、GC 停顿。看 `cpu.stat` 的 `nr_throttled` / `throttled_usec` 判断。
- `cpu.weight` 是**相对权重**，只有**竞争时（CPU 跑满）**才起作用；CPU 空闲时权重无关。→ 这就是 K8s 里 `requests.cpu` 不保证绝对算力的原因。
- **CFS bandwidth control** 的副作用：单线程极限性能受 `quota/period` 影响，不适合做低延迟计算；这类负载应改用 `cpuset`（绑核，避免上下文切换和限流）。

#### 1.3.3 Docker 与 K8s 如何写 cgroup

- Docker：`--cpus`、`-m` 等参数由 dockerd 转成 `oci.LinuxResources` 字段，最终由 **runc** 在容器启动时写入 cgroup 文件。
- K8s：**kubelet 负责创建 cgroup 层级**（v2 下路径大致为
  `kubepods/burstable/pod<UID>/<containerID>`，systemd driver 下是 `kubepods-burstable-pod<UID>.slice`），CRI 请求里带 `LinuxContainerResources`，runtime（containerd）通过 runc 落实。
- **cgroup driver 必须统一**：kubelet `--cgroup-driver=systemd`（v1 时代 cgroupfs 也可，**v2 下必须 systemd**），否则 kubelet 和 runc 抢同一批文件，出现"资源限制不生效"或"节点不稳定"。→ 这是生产上最常见的低级事故之一。

#### 1.3.4 cgroup 的边界

- cgroup **只管资源，不管隔离**。cgroup 不能阻止进程看到别的进程。
- cgroup **不保证绝对性能**：`cpu.weight` 只在竞争时生效；`memory.max` 是"能用到这么多内存"，不是"保证有这么多内存"。
- 用 cgroup 限制磁盘 IO 依赖 **blkio / io**，对 `fuse`、网络存储、NFS 生效有限；真正的磁盘 IO 隔离在云上通常要靠 **cgroup v2 + io.latency**（基于延迟而非带宽）。

---

### 1.4 文件系统：rootfs、chroot、pivot_root

**为什么容器需要一个"独立根目录"？**
因为镜像里必须包含系统的全部依赖（`/lib`、`/usr`、`/etc`、二进制），而与宿主的文件系统隔离。

**演进：chroot → pivot_root**

```
【chroot 的做法与缺陷】
chroot("/newroot") 只是把进程的 root 和 cwd 改到 /newroot。
缺陷 1：可逃逸 —— 先 open 一个指向 root 之外的 fd，再 chdir("../../..")，
       根目录就"退"回去了。（经典逃逸手法）
缺陷 2：不隔离 mount —— 进程仍可以做新的 mount（若有 CAP_SYS_ADMIN），
       而且之前已存在的挂载点仍在可见范围内。

【pivot_root 的做法（runc 默认）】
pivot_root(new_root, put_old)
  · 把当前 root 挂到 new_root 下的 put_old
  · 把 new_root 提为新 root
  · 然后 umount put_old → 旧根彻底消失，没有 fd 能"退回"
优点：能配合 mount namespace 彻底切断旧根，无逃逸路径。
```

**容器 rootfs 的完整构造（runc 在 create 阶段做的事）**：

```
1. 准备挂载点目录（bundle/rootfs 通常只是个空目录或 overlay 的 merged 目录）
2. 对每个镜像层做 bind mount，或用 overlay 一次性 mount：
   mount -t overlay overlay \
     -o lowerdir=/var/lib/docker/overlay2/l/L1:/var/lib/docker/overlay2/l/L2,...,
        upperdir=/var/lib/docker/overlay2/<cid>/diff,
        workdir=/var/lib/docker/overlay2/<cid>/work
        /var/lib/docker/overlay2/<cid>/merged
3. 在新 mount ns 内 pivot_root 到 merged
4. 挂载必要的伪文件系统：
   /proc    proc     (相对 PID ns)
   /sys     sysfs
   /dev     tmpfs + 按需 mknod/bind 宿主 /dev/null 等
   /dev/pts devpts
   /dev/shm tmpfs
   /dev/mqueue mqueue
5. 特殊 device：/dev/console、/dev/ptmx 等按 cgroup devices 白名单放行
6. 应用只读挂载（视配置）：/proc/sys 常被设为只读，防容器改内核参数
```

> 补充：**容器内 `/dev` 里的设备是"按需存在的"**。runc 并不把宿主 `/dev` 全 mount 进去，而是创建 tmpfs，再 bind 少量安全设备（null/zero/full/random/urandom/tty）。这是"容器内看不到宿主硬盘"的原因。

---

### 1.5 联合文件系统：OverlayFS 与镜像分层

**一句话本质**：OverlayFS 让多个目录（lowerdir）**叠加**成一个看起来完整的目录（merged），写操作全部落在**可写层（upperdir）**，读时"从上往下"逐层查找 —— 即 **CoW（Copy-on-Write）**。

```
      merged/   ← 容器进程看到的 / (合并视图)
        │
  ┌─────┴────────────────────────────────────────┐
  │ 读：从最上层开始找，找到第一个就返回          │
  │ 写：文件在 lowerdir → 先 copy-up 到 upperdir  │
  │     再修改（除非是只读操作）                  │
  │ 删：在 upperdir 建 whiteout 文件（隐藏下层）   │
  └───────────────────────────────────────────────┘

  upperdir (可写层, 容器层 rw)   ← 容器的所有修改都在这里
  ────────────────────────────────────────────────
  lowerdir_3 (镜像层 只读)
  lowerdir_2 (镜像层 只读)
  lowerdir_1 (镜像层 只读)
```

**核心机制细节**：

| 机制 | 说明 | 工程含义 |
|---|---|---|
| **copy-up** | 修改 lower 层文件时，先整文件复制到 upper 再改 | 大文件改一个字节也要整份拷贝 → **数据库/大日志文件绝不该放容器层**，必须用 volume |
| **whiteout** | 删除 lower 层文件时在 upper 建 `.wh.<name>` 字符设备 | 删除不会真正释放镜像层空间 |
| **opaque dir** | 目录被标 `trusted.overlay.opaque=y` 时，下层同目录完全不可见 | `rm -rf /dir && mkdir /dir` 会产生 opaque |
| **redirect_dir / index** | 允许目录 rename 而不 copy-up；index 用 inode 索引加速 hardlink | 未开启时跨层 `mv dir` 会整树 copy-up（慢）→ 老版本 docker 常见"重命名卡住" |
| **page cache 归因** | 同一文件被多个容器共享时，page cache 只算一次，但**算在最后读取者的 cgroup 上** | 容器内存统计"抖动"、宿主机 cache 归属不准 |
| **层数限制** | overlay2 建议 ≤ 128 层（受 `lowerdir` 参数限制，内核 `OVL_MAX_STACK`） | 镜像层数过多 → 每次 lookup 要遍历更多层 → 启动慢、`docker build` 缓存易断 |

**为什么 Docker 用 overlay2 而不是别的**：

| 驱动 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| overlay2 | 内核 OverlayFS | 无需额外存储、性能好、inode 压力小 | 需要 4.x+ 内核、层多时变慢 |
| overlay (旧) | 同上但只支持 1 层 lower | — | 已废弃 |
| au类似分块/loop 设备（devicemapper） | 块设备快照（thin pool） | 支持 COW 块级 | 需要 LVM，性能一般，已弃用 |
| btrfs / zfs | 文件系统快照 | 快照/压缩强 | 内存消耗大、稳定性争议 |
| vfs | 完全复制目录 | 最简单，任何场景可用 | **无 CoW，每层整份复制，极慢、极占空间**（只用于调试） |

> **延伸（面试加分）**：`overlay2` 的 `upperdir` 就是"容器可写层"，`docker commit` 实际上是把这个 upperdir 打包成一个新 layer；`docker diff` 是比对 upperdir 与 lower 的差异。
> **`/var/lib/docker` 会爆盘的典型原因**：① 容器内写了大量数据（应挂 volume）；② 日志驱动 json-file 无限增长（应配 `log-opts max-size`）；③ 悬空镜像/停止容器未清理；④ overlay2 的 `l/` 短链接目录与 layer 目录不一致导致占用重复统计（需 `docker system df -v` 逐项查）。

---

### 1.6 从 root 到"受限 root"：Capabilities / Seccomp / LSM

**问题**：Linux 的权限模型是"非 0 即神"。容器里跑 nginx 要以 root 绑 80 端口，但绝不能让 root 去 `mount`、`mknod`、改内核。

#### 1.6.1 Capabilities：把 root 特权切成约 40 块

- **原理**：`task_struct` 里有 5 个 capability 集合：`Permitted` / `Effective` / `Inheritable` / `Bounding` / `Ambient`。
- **Docker 默认保留的 capabilities（约 14 个）**：
  `CHOWN, DAC_OVERRIDE, FOWNER, FSETID, KILL, SETGID, SETUID, SETPCAP, NET_BIND_SERVICE, NET_RAW, SYS_CHROOT, MKNOD, AUDIT_WRITE, SETFCAP`
- **Docker 默认 drop 掉的高危能力**：`CAP_SYS_ADMIN`（几乎所有逃逸的万能钥匙）、`CAP_SYS_PTRACE`、`CAP_SYS_MODULE`、`CAP_DAC_READ_SEARCH`、`CAP_NET_ADMIN` 等。
  - `--privileged` = 给全部 capabilities + 放开 cgroup 设备白名单 + 关闭 seccomp/AppArmor → **等于 root 全部权限**，生产禁用。
  - `--cap-add=SYS_PTRACE` 是排障常用但要评估风险（可读宿主进程内存？不，只在本 ns 内；但有 `CAP_SYS_PTRACE` + 宿主 pid ns 才能看宿主进程）。

#### 1.6.2 Seccomp（Secure Computing Mode）：系统调用白名单/黑名单

- **原理**：`prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)` 装一个 **BPF 程序**，内核在每个 syscall 入口用该程序校验 `(syscall_nr, args...)`，决定 `ALLOW / ERRNO / KILL / TRAP / TRACE`。
- **Docker 的默认 profile**（`seccomp/default.json`）是一个**黑名单**：屏蔽约 40+ 个 syscall / 或对某些 syscall 的参数做限制。典型被屏蔽的：
  `mount`, `umount2`, `ptrace`, `kexec_load`, `reboot`, `swapon`, `settimeofday`, `add_key`, `keyctl`, `clone`（带 `CLONE_NEWUSER` 且非 `CAP_SYS_ADMIN` 时）、`unshare`, `bpf`（旧版）、`perf_event_open`（受限）。
  → 所以容器内 `date -s`、`mount`、`sysctl -w` 会直接 `Operation not permitted`，**不是权限问题，是 seccomp 拒绝**。
- **K8s 的 `securityContext.seccompProfile`**：`RuntimeDefault`（用 runtime 默认）、`Localhost`（挂载自定义 profile）、`Unconfined`。
- **延伸**：seccomp 是**唯一能拦住"还没被 LSM 覆盖"的内核攻击面**的手段，是高安全场景（多租户、Serverless）的核心防线。gVisor 更是把整个 syscall 层搬到用户态（见第 10 章）。

#### 1.6.3 LSM：SELinux / AppArmor

- **SELinux**（RHEL/CentOS 默认）：基于**标签（label）**的强制访问控制，用 `container_t` 等类型标记容器进程，限制其对宿主资源的访问。需要 Docker 以 `--selinux-enabled` 启动。
- **AppArmor**（Ubuntu/Debian 默认）：基于**路径**的 MAC，Docker 自带 `docker-default` profile，禁止 `mount`、禁写 `/sys` 关键路径、禁 `ptrace` 宿主等。
- **两者对比**：SELinux 更细粒度（对象级标签），更复杂；AppArmor 更易用（路径规则），覆盖略粗。

#### 1.6.4 容器逃逸攻击面清单（安全岗必背）

| 路径 | 原理 | 防御 |
|---|---|---|
| `--privileged` | 全 capability + 放开设备 | 禁用；用 `--cap-drop=ALL --cap-add=` 白名单 |
| `docker.sock` 挂载 | 容器内可访问宿主 daemon → 等同 root | 不挂载；必须挂时用 `:ro`+ 只读代理 |
| 内核漏洞 | Dirty COW / Dirty Pipe / CVE-2022-0492（cgroup release_agent）等 | 及时打补丁；seccomp + 只读 `/sys/fs/cgroup` |
| `/proc` 未重挂载 | 看到宿主进程 | 确保 `mount -t proc` 与 PID ns 一致 |
| `hostPath` 挂载宿主 `/` | 直接读写宿主 | 限制挂载路径；PSA/PSP 限制 |
| runc 漏洞 | CVE-2019-5736（覆写宿主 runc 二进制）、CVE-2024-21626（`/proc/self/fd` 泄漏导致逃逸） | 升级 runc；避免 `--privileged` |
| 内核模块加载 | `CAP_SYS_MODULE` | drop capability |
| cgroup v1 `release_agent` | 写 `release_agent` 后在宿主执行 | 只读挂载、user ns |

---
### 1.7 OCI 标准与运行时分层：为什么 Docker 不是"容器"本身

**一句话本质**：OCI（Open Container Initiative）把"容器"拆成了三份可互换的标准，任何人只要实现标准就能接入生态 —— 这也是 K8s 能踢掉 Docker 的技术前提。

```
┌────────────────────────────────────────────────────────────┐
│ OCI Image Spec          镜像怎么描述（manifest/config/layer）│
│ OCI Runtime Spec        容器怎么被创建（config.json 的结构）  │
│ OCI Distribution Spec   镜像怎么分发（Registry API）         │
└────────────────────────────────────────────────────────────┘
```

#### 1.7.1 高层运行时 vs 底层运行时（面试常问的分层）

| 层级 | 代表 | 职责 |
|---|---|---|
| **High-level Runtime**（容器管理器） | containerd、CRI-O、Podman、Docker Engine(dockerd) | 镜像拉取/解压/存储管理、通过 snapshotter 管理 rootfs、调用低级运行时、管理 shim、提供 CRI/gRPC API |
| **Low-level Runtime**（OCI runtime） | runc、crun、youki、kata-runtime、runsc(gVisor) | 只做一件事：读 `config.json` → 用 namespace/cgroup/pivot_root 造出容器 → exec 用户进程 |
| **Shim** | containerd-shim-runc-v2、conmon（CRI-O 用） | 容器的"养父"：容器进程由它托管，daemon 重启不影响容器；接管 stdio/退出码 |

**为什么必须有 shim（这是理解 Docker 架构的关键）**：
1. **进程树稳定**：如果 containerd/dockerd 是容器进程的父，重启 daemon 会让容器变成孤儿甚至被杀（早期 docker 就是"重启 docker 服务 = 全容器重启"）。
2. **stdio 归属**：容器 stdout/stderr 是 pipe，必须有常驻进程读它，否则写满 pipe buffer 后容器会阻塞。
3. **退出码回收**：容器退出后要有人 `wait4` 回收并上报。

#### 1.7.2 镜像格式解剖（必须能画出来）

```
Registry 中的镜像（如 docker.io/library/nginx:latest）

  manifest (application/vnd.oci.image.index.v1+json  ← 多架构 index)
      │
      ├── (linux/amd64) manifest: application/vnd.oci.image.manifest.v1+json
      │      ├── config  → application/vnd.oci.image.config.v1+json
      │      │     { architecture, os, config{Env, Cmd, Entrypoint, User, Volumes},
      │      │       rootfs{ type:layers, diff_ids:[sha256:...] }, history[...] }
      │      └── layers[]  (application/vnd.oci.image.layer.v1.tar+gzip)
      │            ├── sha256:aaa  (基础层, 如 debian)
      │            ├── sha256:bbb
      │            └── sha256:ccc  (最后一层, 如 COPY 进去的配置文件)
      │
      └── (linux/arm64) manifest: ...
```

**关键点**：
- **内容寻址（content-addressable）**：每个层用 `sha256(layer) ` 命名 → 天然去重、可校验、可跨镜像共享（"nginx 和 mysql 都基于 debian，共享基础层"）。
- **config 里存的是"元数据"，不是文件**；真正文件在 layer tar 里。
- **层 tar 里包含的是"文件系统 diff"**：新增/修改的文件 + whiteout 标记（`.wh.xxx`）。
- **`diff_ids` vs `layer digests`**：`diff_ids` 是**未压缩 tar 的 sha256**（用于运行时校验），digest 是**压缩后的 sha256**（用于传输）。两者不同 → 面试可问"为什么有两个 id"。
- **多架构（manifest list / image index）**：`docker pull nginx` 在 amd64 机器上自动选 amd64 版本。这是 manifest list 的作用。

#### 1.7.3 OCI runtime spec：`config.json` 里到底有什么

简化示意（runc 造容器时**唯一**的输入）：

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": { "uid": 0, "gid": 0, "additionalGids": [101] },
    "args": ["nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:...", "NGINX_VERSION=1.25.3"],
    "cwd": "/",
    "capabilities": {
      "bounding":  ["CAP_CHOWN","CAP_NET_BIND_SERVICE","..."],
      "effective": ["CAP_CHOWN","CAP_NET_BIND_SERVICE","..."],
      "permitted": ["CAP_CHOWN","CAP_NET_BIND_SERVICE","..."]
    },
    "rlimits": [ { "type": "RLIMIT_NOFILE", "hard": 1048576, "soft": 1048576 } ],
    "noNewPrivileges": true
  },
  "root": {
    "path": "rootfs",
    "readonly": false
  },
  "hostname": "3f8e...",
  "mounts": [
    { "destination": "/proc", "type": "proc", "source": "proc" },
    { "destination": "/dev",  "type": "tmpfs", "source": "tmpfs",
      "options": ["nosuid","strictatime","mode=755","size=65536k"] },
    { "destination": "/dev/pts", "type": "devpts", "source": "devpts" },
    { "destination": "/dev/shm", "type": "tmpfs", "source": "shm" },
    { "destination": "/dev/mqueue", "type": "mqueue", "source": "mqueue" },
    { "destination": "/sys", "type": "sysfs", "source": "sysfs", "options": ["nosuid","noexec","nodev","ro"] },
    { "destination": "/etc/hosts", "type": "bind", "source": "/var/lib/docker/containers/<id>/hosts", "options": ["rbind","rprivate","rw"] }
  ],
  "linux": {
    "resources": {
      "devices":  [ { "allow": false, "access": "rwm" } ],   // 设备白名单（默认全禁，再放行）
      "memory":   { "limit": 536870912, "swap": 536870912 },
      "cpu":      { "quota": 50000, "period": 100000, "shares": 512 },
      "pids":     { "limit": 1024 }
    },
    "namespaces": [
      { "type": "pid" }, { "type": "network" }, { "type": "ipc" },
      { "type": "uts" }, { "type": "mount" }, { "type": "cgroup" }
    ],
    "seccomp": { "defaultAction": "SCMP_ACT_ERRNO", "syscalls": [ ... ] },
    "maskedPaths":   ["/proc/kcore", "/proc/keys", "/proc/timer_list", "/sys/firmware"],
    "readonlyPaths": ["/proc/bus", "/proc/fs", "/proc/irq", "/proc/sys", "/proc/sysrq-trigger"],
    "cgroupsPath": "/docker/<cid>"
  }
}
```

> **这就是"容器只是配置"的最好证据**：一个 JSON 就完整描述了一个容器的全部内核态配置。`maskedPaths`/`readonlyPaths` 就是容器内 `/proc/sys` 不可写的原因。

---

## 第 2 章 Docker 深度拆解

### 2.1 架构演进：从"一个大 daemon"到"三层解耦"

```
【2013–2015 早期】
docker CLI ──HTTP──> dockerd ──exec──> libcontainer(内置) ──> 容器进程
问题：dockerd 管了镜像、网络、卷、exec、日志、构建…… 一个二进制巨兽，
      daemon 崩溃 = 所有容器失联；K8s 想用容器必须依赖整个 docker。

【2016–2017 拆分】
docker CLI ──HTTP──> dockerd（API/镜像/网络/卷/构建）
                       │ gRPC
                       ▼
                    containerd（镜像 pull/unpack、容器生命周期、shim 管理）
                       │ exec
                       ▼
                    containerd-shim ──fork──> runc ──> 容器进程
                       │
                       └─ runc 创建完即退出，容器父进程 = shim

【2018– 现在（K8s 生态）】
kubectl → kube-apiserver → kubelet ──CRI(gRPC)──> containerd(Cri plugin) → runc
（不再经过 dockerd；Docker 只是在 containerd 之上包了一层"开发者友好"的壳）
```

**必须记住的结论**：
- **`docker` 命令 ≠ 容器运行时**。它只是 containerd 的一个客户端（外加镜像构建、网络/卷管理这些 containerd 不管的"编排便利层"）。
- K8s 1.24 移除 dockershim，**不是"抛弃 Docker"，而是抛弃了 kubelet 中间那层 dockershim**；用 Docker 构建的镜像（OCI 格式）依然能在 containerd 上跑。
- 容器标准的边界：**CRI 是 K8s 与运行时的接口，OCI 是运行时的内部接口**。

### 2.2 镜像与分层：构建缓存、可复现、体积治理

#### 2.2.1 层的语义

每一条 `RUN / COPY / ADD` 都产生一层。层是**只读**的，容器启动时在最上面加一个**可写层**。

```
Dockerfile                       镜像层
FROM ubuntu:22.04          →  L0: ubuntu rootfs (只读)
RUN apt-get update         →  L1: /var/lib/apt/lists/* 更新
RUN apt-get install -y curl→  L2: curl 及其依赖文件
COPY app.conf /etc/app/    →  L3: 一个配置文件
CMD ["./app"]              →  (仅元数据，不产生层)
```

#### 2.2.2 构建缓存机制（面试常问"为什么我的缓存失效了"）

- Docker 对每条指令计算 **cache key = 父层 digest + 指令内容**。命中即复用，未命中则该指令及**其后所有指令**全部重建。
- **`COPY`/`ADD` 的 cache key 包含文件内容的 checksum**（不只是文件名）→ 改一个字节就会让后面全失效。
- 因此经典的"依赖先装、代码后拷"范式：
```dockerfile
COPY go.mod go.sum /app/        # 先只拷依赖清单
RUN go mod download             # 依赖变了才重跑
COPY . /app/                    # 代码改了只影响这一层
RUN go build -o /app/server .
```
- `ARG` 参与 cache key：`ARG VERSION` 值变化 → 该层失效。用 `--build-arg` 传版本号要小心"每次都不一样导致全量重建"。
- `--no-cache` 全量重建；`--cache-from` 可复用远端缓存（配合 CI）。
- **多阶段构建（multi-stage）**：编译产物从 builder 阶段 `COPY --from=builder` 拷到精简的运行镜像，最终镜像不含 gcc/源码 → 体积可降 90%。**这是镜像瘦身第一手段**。
- **BuildKit 的新能力**：并行构建、`--mount=type=cache`（`RUN --mount=type=cache,target=/root/.cache/go-build`，缓存不落层）、`--mount=type=secret`（构建期密钥不进镜像层，避免泄露）、`--mount=type=bind`。
- **`docker history` / `dive`** 可以逐层看体积，"哪一层把镜像撑到 2GB"是常见的排查动作。

#### 2.2.3 镜像治理的工程结论

| 问题 | 手段 |
|---|---|
| 体积大、拉取慢 | 多阶段构建 + distroless/alpine 基础镜像 + 清理 apt 缓存（`rm -rf /var/lib/apt/lists/*` 必须与 install **同一层**） |
| 层数过多启动慢 | 合并 RUN、减少 COPY；overlay2 lookup 随层数增加 |
| 镜像不可复现 | 基础镜像固定 digest（`FROM ubuntu@sha256:...`）；`apt-get install` 固定版本 |
| 供应链安全 | 镜像签名（cosign/notation）、SBOM（syft）、漏洞扫描（trivy/grype）、私有 registry 代理 |
| 拉取慢（大规模） | 镜像预热、P2P 分发（Dragonfly）、按需加载（Nydus / eStargz / SOCI） |

### 2.3 存储驱动 overlay2 的工程实践

```
目录结构（/var/lib/docker/overlay2/）
├── l/                       ← 短符号链接（避免 lowerdir 路径过长超 4096 上限）
│    └── ABCD... -> ../<layer-id>/diff
├── <layer-id>/
│    ├── diff/               ← 该层实际文件
│    ├── link                 ← 短名
│    └── lower                ← 该层下面所有层的短名（用 : 连接）
└── <container-layer-id>/
     ├── diff/               ← 容器可写层（upperdir）
     ├── work/               ← overlay 的 workdir（内部用，勿动）
     ├── merged/             ← 容器看到的 /（挂载点）
     └── lower               ← 指向镜像各层
```

**实践禁忌**：
- 不要在容器层写大文件（copy-up 放大 IO 与空间），用 volume；
- 不要改 `merged/` 或 `diff/` 里的东西（会破坏 overlay 一致性），容器运行中只能通过容器内改；
- 清理：`docker system prune`、`docker builder prune`，以及手动清 `/var/lib/docker/overlay2` 前**必须先停 daemon**（否则句柄泄漏）。

### 2.4 网络模型：veth / docker0 / iptables 的真实数据流

#### 2.4.1 默认 bridge 网络的构造

```
宿主机                                 容器 netns
┌──────────────────────────────────┐   ┌──────────────────────────┐
│  docker0 (172.17.0.1/16, bridge) │   │  eth0 (172.17.0.2/16)    │
│        │                         │   │   default route          │
│        │ vethXXXX (宿主侧)        │   │   → 172.17.0.1           │
│        └───────── veth pair ─────┼───┘                          │
│  iptables: nat/DOCKER, FORWARD   │   │  lo (127.0.0.1)          │
└──────────────────────────────────┘   └──────────────────────────┘
```

**创建过程**（dockerd 在 `container start` 之前做）：
1. `ip link add vethXXX type veth peer name eth0`
2. 把 `eth0` 移到容器 netns：`ip link set eth0 netns <pid>`
3. 给容器 eth0 配 IP（从 `172.17.0.0/16` 的 IPAM 池分配，网关 172.17.0.1）
4. 宿主侧 veth 挂到 docker0，`ip link set vethXXX up`、docker0 up
5. 容器内起 `lo`、加默认路由指向 172.17.0.1
6. 写 `/etc/hosts`、`/etc/resolv.conf`（从宿主 `/etc/resolv.conf` 复制或指定 `--dns`）

**数据流（容器 → 外网）**：

```
容器进程 write() → 容器 eth0 → veth pair 对端（宿主 vethXXX）
  → docker0 网桥转发 → 宿主路由表决定走 eth0（物理网卡）
  → netfilter nat/OUTPUT? 不，是 nat/POSTROUTING：
       规则：-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
  → SNAT 成宿主 IP → 出网卡 → 外部
回包：外部 → 宿主 IP:随机源端口 → conntrack 反查 → DNAT 回 172.17.0.2
```

**关键前提**：`net.ipv4.ip_forward=1`（没有它，容器根本出不了网，只有 ping docker0 能通）。所以 `docker run` 会自动设这个 sysctl（这就是为什么容器里 `sysctl -w net.ipv4.ip_forward=0` 会失败——它是全局参数，容器里只读）。

**数据流（外部 → 容器，`-p 8080:80`）**：

```
方式 A：iptables DNAT（真正生效的转发路径）
  nat/DOCKER: -A DOCKER ! -i docker0 -p tcp --dport 8080 \
              -j DNAT --to-destination 172.17.0.2:80
  外部请求 宿主IP:8080 → PREROUTING 命中 → 目标改 172.17.0.2:80
  → FORWARD 链放行（-A FORWARD -o docker0 -j ACCEPT 等）
  → 到容器，容器回包经 conntrack 反向 SNAT 成 宿主IP:8080

方式 B：docker-proxy（userland proxy，默认开启）
  dockerd 为每个 -p 起一个 docker-proxy 进程监听宿主 8080，
  收到连接后自己再"代理"到 172.17.0.2:80。
  存在的意义：① 让宿主机上 127.0.0.1:8080 也能访问（DNAT 在某些
  回环场景/hairpin 下不生效）；② 兼容不支持 NAT 回环的环境。
  代价：多一次用户态拷贝，连接数高时是瓶颈 → 生产常配
  /etc/docker/daemon.json: {"userland-proxy": false}
```

> **`ip_forward` + `MASQUERADE` + `DNAT` + `conntrack`** 这四件套就是 Docker 网络的全部魔法。iptables 规则可用 `iptables -t nat -L -n -v`、`iptables -t filter -L DOCKER-USER -n` 观察。

#### 2.4.2 五种网络模式对比

| 模式 | 命令 | 网络栈 | IP | 端口 | 典型用途 |
|---|---|---|---|---|---|
| bridge（默认） | `--network bridge` | 独立 netns + docker0 | 172.17.x.x | 需 `-p` 映射 | 绝大多数场景 |
| host | `--network host` | **共享宿主 netns** | 宿主 IP | **直接用宿主端口** | 高性能网络（避免 NAT 开销）、需监听大量端口的服务 |
| none | `--network none` | 只有 lo | 无 | 无 | 完全离线、安全隔离的计算任务 |
| container | `--network container:<id>` | 共享另一容器 netns | 同目标 | 同目标 | K8s Pod 内部就是这个模型 |
| 自定义 bridge | `docker network create` | 独立网桥、**内置 DNS** | 自选网段 | 容器名互访 | 需要服务发现/隔离的多容器应用 |
| macvlan | `--network macvlan` | 独立 netns，二层直连物理网 | **物理网段 IP** | 无 NAT | 需要"看起来像物理机"、跑 DHCP/组播 |
| overlay | `docker network create -d overlay` | VXLAN 跨主机 | 虚拟网段 | — | Swarm 多机组网（K8s 里对应 Flannel/Calico） |

**两个高频考点**：
1. **默认 bridge 下容器间能否用容器名互访？** 不能（没有内嵌 DNS），只能用 IP；`--link` 是过时且不推荐的写法；需自定义网络（`docker network create mynet`）才有 DNS 解析容器名。
2. **`--network host` 的坑**：容器与宿主共享端口空间，**端口冲突直接启动失败**；且容器内改 iptables 会影响宿主（在 host 模式下 netns 是同一个）。K8s 里的 `hostNetwork: true` 同理。

### 2.5 数据卷：为什么必须用 volume 而不是容器层

| 类型 | 命令 | 存储位置 | 生命周期 | 适用 |
|---|---|---|---|---|
| volume（命名卷） | `-v myvol:/data` | `/var/lib/docker/volumes/myvol/_data` | 独立于容器，可复用、可备份 | **生产数据首选** |
| bind mount | `-v /host/path:/data` | 宿主任意路径 | 宿主管理 | 配置注入、开发挂载、需要宿主可见 |
| tmpfs | `--tmpfs /data` | 内存 | 容器停止即失 | 敏感临时数据、极高性能缓存 |

- **bind mount vs volume 的本质差别**：volume 由 Docker 管理（首次自动初始化：**会把镜像里该目录的已有内容复制进空卷**）；bind mount 直接覆盖，镜像里的内容被遮蔽。
- **性能**：overlay2 上的写有 copy-up 开销；bind mount 直通宿主文件系统（无 CoW）；tmpfs 最快但吃内存。
- **macOS/Windows 上的 bind mount 很慢**，因为要经过 VM 文件共享层（gRPC-FUSE/osxfs），Linux 原生无此问题。
- **K8s 对应物**：`emptyDir`（Pod 生命周期）、`hostPath`（危险）、`PVC`（持久化，走 CSI）。

### 2.6 Docker 资源限制的完整映射

```bash
docker run \
  --cpus=1.5                  → cpu.max = "150000 100000"（v2）
  --cpu-shares=512            → cpu.weight ≈ 51（v1 cpu.shares=512）
  --cpuset-cpus=0-1           → cpuset.cpus = "0-1"
  --memory=1g                 → memory.max = 1073741824
  --memory-reservation=512m   → memory.high
  --memory-swap=2g            → memory.swap.max = 1g（swap = swap-limit - limit）
  --pids-limit=200            → pids.max
  --blkio-weight=500          → io.weight
  --device-read-bps=/dev/sda:1mb → io.max
```

> 注意 `--memory-swap` 的语义坑：它是 **"内存+swap" 的总量**。`--memory=1g --memory-swap=2g` 表示可用 1g swap；`--memory-swap=-1` 表示不限 swap；只给 `--memory` 不给 swap 时，swap 默认 = 2 倍 memory（且 `--memory-swap=0`/不设时行为随配置变化）。生产 K8s 节点通常**关闭 swap**（`--fail-swap-on`），因为 swap 会让内存限制失真、驱逐判定失准。

### 2.7 `docker exec`、`attach`、`commit` 的底层行为

| 命令 | 底层动作 |
|---|---|
| `docker exec` | 调 containerd 的 `ExecProcess` → shim → 在**容器已有 ns/cgroup 内** `fork+exec` 新进程。**进程直接在容器的 cgroup 里**（所以会占容器 CPU/内存配额）。`exec` 无法在已暂停容器上执行。 |
| `docker attach` | 连接到容器**已存在的 PID 1 的 stdio**（不是新建进程）。`Ctrl-C` 可能直接把容器主进程杀掉（取决于 tty 配置）。 |
| `docker commit` | 把容器的 **upperdir（可写层）** 打包成新 layer，生成新镜像。**不会包含 volume 里的数据**，也**不会记录端口/环境之外的运行时状态**（如已运行的内存）。 |
| `docker cp` | 直接在宿主文件系统与容器 `merged/` 目录之间拷贝（走 daemon，不需要容器内有 tar）。 |
| `docker pause` | 写 `cgroup.freeze=1`（v2）或 `freezer.state=FROZEN`（v1），**SIGSTOP 全部进程**。 |
| `docker stop` | 发 `SIGTERM` 给 PID 1 → 等 `--time`（默认 10s）→ `SIGKILL`。→ **PID 1 没注册 SIGTERM handler 时，必然等满 10 秒**。 |

### 2.8 Dockerfile 与 Engineering Best Practices（生产级清单）

```dockerfile
# ---- 阶段 1：构建 ----
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

# ---- 阶段 2：运行 ----
FROM gcr.io/distroless/static:nonroot     # 无 shell、无包管理器 → 攻击面最小
COPY --from=builder --chown=65532:65532 /out/app /app
USER 65532:65532                          # 非 root 运行
EXPOSE 8080
ENTRYPOINT ["/app"]                       # exec 形式，PID 1 直接是 app
```

**清单（每条都说得出理由）**：
1. **固定基础镜像 digest**，避免"今天构建和昨天构建跑出两个镜像"。
2. **多阶段构建**，运行镜像不含编译工具链。
3. **非 root 运行**（`USER`），配合 `readOnlyRootFilesystem: true`（K8s）。
4. **exec 形式 ENTRYPOINT**（`["/app"]`），让 PID 1 = 你的进程，能正确接收信号；需要 shell 特性时用 `tini`/`--init`。
5. **合并 `apt-get update && install && rm -rf /var/lib/apt/lists/*` 到同一 RUN**（否则缓存的 lists 留在层里既占空间又可能让后续 install 用旧索引）。
6. **`.dockerignore`** 排除 `.git`、`node_modules`、测试数据 → 减小 build context、避免 COPY 缓存全部失效。
7. **不把密钥写进镜像**（`ARG`/`ENV` 都会留在 `docker history` 里）；用 BuildKit `--mount=type=secret` 或运行时注入。
8. **健康检查**（`HEALTHCHECK` 或 K8s 探针），不要靠"进程还在"判断健康。
9. **单进程原则**（一个容器一个职责），日志写 stdout/stderr 交给 runtime 收集。

### 2.9 Docker 常见故障与排障路径（工程价值最高的部分）

```
【容器启动即退出】
→ docker logs <cid>；docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}'
→ 137 = SIGKILL（多为 OOM）；126 = 权限不足/不可执行；127 = 命令找不到；
  139 = SIGSEGV；1 = 应用自身错误；0 但立刻退出 = 主进程前台化没做好（如 nginx 未 daemon off）

【docker: Error response from daemon: ... no space left on device】
→ df -h /var/lib/docker；docker system df -v 逐项看镜像/容器/卷/构建缓存
→ 常见元凶：日志未轮转、容器内写大文件、悬空镜像、inode 耗尽（df -i）

【容器内存"虚高"但没泄漏】
→ cat /sys/fs/cgroup/.../memory.stat 区分 anon / file / slab
→ 用 working_set 而非 current 判断

【容器网络不通】
→ 容器内：ip addr / ip route / cat /etc/resolv.conf
→ 宿主：sysctl net.ipv4.ip_forward、iptables -t nat -L DOCKER -n -v、
       iptables -t filter -L DOCKER-USER -n -v（FORWARD 被 DROP 是经典坑）
→ 是否与 firewalld 冲突（firewalld 的 FORWARD 默认 DROP）

【容器 DNS 解析失败】
→ 默认用宿主 /etc/resolv.conf 的 nameserver；宿主用 127.0.0.53（systemd-resolved）
  时容器内解析会失败 → 需在 daemon.json 配 "dns": ["8.8.8.8"] 或改 stub listener
  （这是极高频的运维事故）

【端口映射不生效 / 只能本机访问】
→ 检查 DOCKER-USER 链是否有阻断；检查 bind 地址（-p 127.0.0.1:8080:80 只监听回环）；
  检查云厂商安全组/主机防火墙

【docker exec 失败 / 容器僵死】
→ nsenter -t <pid> -a 直接进；若连 nsenter 都不行，检查 shim 进程是否存活
```

---

## 第 3 章 Kubernetes 架构与内核机制

### 3.1 设计哲学：三个词概括整个 K8s

```
        ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
        │  声明式 API  │ →  │  控制器循环  │ →  │  水平扩展    │
        │ Declarative │    │  Reconcile  │    │  Horizontal │
        └─────────────┘    └─────────────┘    └─────────────┘
```

1. **声明式 API**：你提交的是"期望状态"（`replicas: 3`），不是"操作步骤"。→ 幂等、可重放、天然适合分布式收敛。
2. **控制器循环（Reconcile）**：所有组件都在做同一件事 —— **观测实际状态 → 与期望状态 diff → 执行动作使其收敛**。这不是"命令传递"，而是"各自独立地收敛到同一目标"。
3. **水平扩展优先于垂直扩展**：K8s 假设"任何组件都可能挂"，用多副本 + 选举 + 重试来获得可靠性，而不是追求单点不出错。

**由此推出的核心工程结论（面试常考的"为什么"）**：
- **为什么 K8s 不保证"立即"？** 因为它是最终一致（eventual consistency）的收敛模型，允许延迟（watch 延迟、调度排队、探针周期）。
- **为什么 `kubectl apply` 可以反复执行？** 声明式 + 幂等。
- **为什么 K8s 里没有"重启服务"这种操作？** 只有"改变期望状态"（改镜像版本 / 删 Pod 让控制器重建）。

### 3.2 组件全景与职责边界

```
┌────────────────────────────── Control Plane（控制面）──────────────────────────────┐
│                                                                                    │
│  kubectl / dashboard / CICD ──HTTPS──> kube-apiserver（无状态，可多副本）            │
│                                          │ 认证 → 授权 → 准入 → 写入               │
│                                          ▼                                         │
│                                    etcd（唯一真相源，Raft 强一致）                    │
│                                          ▲ watch                                  │
│         ┌────────────────────────────────┴────────────────────────────────┐        │
│         │                                                                 │        │
│  kube-controller-manager                                       kube-scheduler      │
│   · Deployment/RS/StatefulSet/DaemonSet/Job 控制器               · 调度队列         │
│   · Node/Endpoint/Namespace/PV 控制器                           · Filter/Score      │
│   · GC、HPA、CSR 签发                                           · 抢占             │
│   · 只有 1 个 leader 生效（Lease 选主）                          · 多实例/leader     │
└────────────────────────────────────────────────────────────────────────────────────┘
                                          │ watch / 上报
┌─────────────────────────────── Node（工作节点）───────────────────────────────────┐
│                                                                                    │
│  kubelet ──CRI gRPC──> containerd ──> runc ──> 容器                                │
│    ├─ PLEG（每秒 relist 容器状态）                                                  │
│    ├─ syncLoop（收敛 Pod 期望 vs 实际）                                             │
│    ├─ probeManager（执行 liveness/readiness/startup）                              │
│    ├─ cAdvisor（内置，容器资源指标）                                                │
│    ├─ 通过 Node Lease 续约心跳                                                      │
│    └─ 调用 CNI 插件（配置 Pod 网络）                                                │
│                                                                                    │
│  kube-proxy ──> iptables / ipvs / nftables 规则（Service 转发）                      │
│  CNI 插件（flannel / calico / cilium）──> veth / 路由 / 隧道                        │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**职责边界（容易答错的地方）**：

| 组件 | 明确**不做**的事 |
|---|---|
| kube-apiserver | 不做调度、不管理容器、不做健康检查；它只是"带准入策略的 etcd 读写网关 + watch 分发" |
| kube-scheduler | 不创建 Pod，只**在 Pod 的 spec 里写 `nodeName`**（Binding 操作），然后 apiserver 更新 etcd |
| controller-manager | **不直接指挥 kubelet**，只操作 API 对象（创建/删除 Pod 对象） |
| kubelet | 不参与调度、不管理 Service；只管"本节点上的 Pod 应该是这样" |
| kube-proxy | 不管端点健康（那是探针和 Endpoint 控制器的活），只负责"把规则同步到内核" |
| etcd | 不参与任何业务逻辑，只提供一致性的 KV + watch |

### 3.3 etcd 与 list-watch：K8s 的"神经系统"

#### 3.3.1 etcd 的核心机制

- **Raft 共识**：3 节点容忍 1 节点故障，5 节点容忍 2。写入必须多数派（quorum）确认才返回成功 → **强一致**。所以 etcd 节点是**奇数**，且**跨故障域部署**。
- **MVCC 多版本**：每次写入生成一个全局递增 `revision`。key 的历史版本可读（用于 watch 从某个 `resourceVersion` 开始补事件）。
- **Watch**：客户端可 watch 一个 key 前缀，etcd 推送 `PUT/DELETE` 事件，带 `revision`。**k8s 的实时性完全建立在这个能力上**。
- **Compaction**：历史版本会无限增长 → 需要定期 compact（K8s 默认每 5 分钟 compact 一次，保留 5 分钟历史）。压缩后老的 `resourceVersion` 无法再 watch → 客户端收到 `410 Gone` 必须 **relist（全量重新拉取）**。
- **存储**：WAL + boltdb（bbolt 的 B+ 树），`--quota-backend-bytes` 默认 2GB（生产常调到 8GB）。超配额后 etcd 进入 **NOSPACE 告警并拒绝写入** → 整个集群只读 → 这是"集群突然全挂"的经典原因。
- **defrag**：删除 key 后文件不缩小，需 `etcdctl defrag` 回收空间。
- **性能关键**：etcd 对 **fsync 延迟**极度敏感 → 必须用 SSD/NVMe、避免网络存储、避免和其他 IO 密集进程混部。

#### 3.3.2 Informer：K8s 客户端如何"高效感知变化"

```
       ┌───────────────────────────────────────────────────────┐
       │ 客户端进程（kubelet / kube-proxy / controller）         │
       │                                                       │
       │  ┌──────────────┐   ListAndWatch  ┌───────────────┐    │
       │  │  Reflector   │────────────────>│ kube-apiserver│    │
       │  │  (每个资源   │  LIST (全量)    │ (watch cache) │    │
       │  │   一个 goroutine)  WATCH(增量) └───────────────┘    │
       │  └──────┬───────┘                                     │
       │         │ 写入                                        │
       │  ┌──────▼───────┐    ┌──────────────┐                 │
       │  │  DeltaFIFO   │───>│   Indexer    │ (本地内存缓存)    │
       │  │ (变更队列)    │    │ thread-safe  │                 │
       │  └──────┬───────┘    └──────┬───────┘                 │
       │         │ pop               │ Get/ByIndex             │
       │  ┌──────▼───────┐           │                        │
       │  │  ProcessLoop │──> 事件处理函数 / workqueue          │
       │  └──────────────┘                                     │
       └───────────────────────────────────────────────────────┘
```

**为什么不是"每次轮询 API"**：
- 轮询：N 个组件 × M 个资源 × 每秒 1 次 → apiserver 被打爆（几千节点的集群每天几十亿请求）。
- list-watch：**一次 LIST 拿全量 + 一次 WATCH 拿增量**，正常情况下几乎零轮询。这是 K8s 能支撑 5000 节点的关键设计之一。

**apiserver 侧的配套：watch cache**
- apiserver 自己也维护一份 etcd 数据的缓存（`watch cache`，默认大小可通过 `--watch-cache-sizes` 调），LIST 请求优先从 cache 返回，避免每次都打 etcd。
- 单个 etcd watch 被 apiserver 扇出（fan-out）给成千上万个客户端 watch → **etcd 只承受 apiserver 的少量 watch**。
- 客户端指定的 `resourceVersion` 太老（已被 compact）→ apiserver 返回 `410 Gone` → informer 自动重新 LIST。

**workqueue（控制器的工作队列）**：
- 三类：`FIFO`、`RateLimiting`（带 `rateLimiter`，如每秒最多 N 次）、`Delaying`（延迟执行，用于退避重试）。
- 特性：**去重（dedup）+ 标记脏（dirty）+ 处理中（processing）**。同一个对象在队列里只会有一份，处理期间再变更会重新入队 → 保证"最终处理到最新状态"，且天然抗抖动。
- **`AddRateLimited` + `NumRequeues`** 实现指数退避 → 这就是控制器"失败了会越来越慢地重试"的原理。

### 3.4 kubelet 内核：节点上的"总管家"

#### 3.4.1 kubelet 的启动与心跳

```
启动 → 读配置（/var/lib/kubelet/config.yaml）→ 向 apiserver 注册 Node 对象
     → 启动若干 goroutine：
         · syncLoop         主循环（处理 config/housekeeping/PLEG 三种 channel 事件）
         · pleg             每 1s 调 CRI 的 ListPodSandbox + ListContainers
         · statusManager    管理 Pod status 上报（合并探针/容器状态）
         · probeManager     每个容器一组 worker 跑探针
         · evictionManager  节点压力驱逐
         · volumeManager    挂载卷（配合 CSI）
         · imageManager     镜像 GC（默认 85% 磁盘占用触发淘汰）
         · certificateManager 证书轮转
心跳：Node Lease（1.14+）—— 每 10s 更新一次 Lease 对象（默认
      nodeLeaseDurationSeconds=40），**不写 NodeStatus**（避免每次心跳都
      产生一次全量 Node 对象写放大）。NodeStatus 只在状态变化或每 5 分钟上报。
```

**为什么引入 Lease 心跳（常考）**：
- 旧方案：kubelet 每 10s 更新 `Node.status` → 每次更新都触发全集群 watch 事件 + etcd 写入，5000 节点集群下 etcd 会被心跳淹没。
- 新方案：心跳写一个**极小的 Lease 对象**（只有 `spec.holderIdentity` + `renewTime`），Node 对象的 `status` 更新频率降为 5 分钟或状态变化时 → etcd 压力下降一个数量级。

#### 3.4.2 PLEG 与 syncLoop：为什么容器挂了 kubelet 会知道

- **PLEG（Pod Lifecycle Event Generator）**：每 1 秒调 CRI `ListPodSandbox`/`ListContainers` 拿到**本节点所有容器真实状态**，与内存中的 `podCache` 比对，生成事件（ContainerStarted/ContainerDied/ContainerChanged）。
  - 为什么不用 CRI 的事件流？因为 CRI 早期没有事件接口，且 runtime 重启会丢事件 → **轮询（relist）是最终可信的来源**。
  - PLEG 慢会打出 `PLEG is not healthy` 日志 → 通常意味着 runtime（containerd）卡住或节点负载极高（会导致 Pod 无法及时重建）。
- **syncLoop** 收到事件后进入 `syncPod`：
```
syncPod(期望 Pod)：
  1. 若 Pod 应被删除 → kill 所有容器 + 清理 sandbox（含 CNI teardown）
  2. 计算 sandbox 变化 → 若需要，创建/重建 sandbox（调用 CRI RunPodSandbox → 触发 CNI ADD）
  3. 按顺序创建 init 容器（串行，全部成功才继续）；失败的 init 容器按
     restartPolicy 重试（Pod 级 restartPolicy 对 init 容器生效）
  4. 创建业务容器（CRI CreateContainer + StartContainer）
  5. 更新 statusManager（Ready 条件需要 readiness 探针通过）
```
- **关键设计：kubelet 重启不影响容器运行**。因为容器归 containerd/runc 管，kubelet 重启后靠 PLEG 重新"认领"（adopt）已有容器 —— 依据是容器 label 里的 `io.kubernetes.pod.uid`。这也是为什么**不要手工改容器 label**。

#### 3.4.3 探针机制

| 探针 | 失败后果 | 支持 exec/httpGet/tcpSocket/grpc | 备注 |
|---|---|---|---|
| `livenessProbe` | **重启容器**（kill → 按策略重建） | 是 | 不应依赖外部服务（否则外部故障引发雪崩重启） |
| `readinessProbe` | 从 Service Endpoints 摘除（不重启） | 是 | `Pod Ready` 条件；`kubectl get pod` 的 READY 列 |
| `startupProbe` | 重启容器；**成功前屏蔽 liveness/readiness** | 是 | 解决"启动慢但运行稳"的应用被 liveness 误杀 |

**必须知道的细节**：
- `initialDelaySeconds`、`periodSeconds`、`timeoutSeconds`、`successThreshold`（liveness/startup 必须为 1）、`failureThreshold`。
- **探针在容器内执行**（exec 探针 fork 一个进程，占用容器资源）；高频探针本身可能成为 CPU 消耗大户。
- **探针超时是"容器内时间"还是"kubelet 时间"**：由 kubelet 计时，但 exec 探针受容器调度延迟影响（容器被 CPU 限流时，探针也会超时）→ 这会导致"CPU 打满 → 探针超时 → 被判死 → 重启 → 更打满"的死亡螺旋。生产上要给探针留足 `timeoutSeconds` 余量。
- `terminationGracePeriodSeconds` + `preStop`：优雅退出的正确姿势是 **preStop hook 里 sleep 几秒（等 Endpoint 摘除传播完成）→ 再 SIGTERM**。因为 Endpoint 摘除与 SIGTERM 是**并发**的，不同步就会丢请求。

#### 3.4.4 节点压力驱逐（Eviction）

```
kubelet 定期（每 10s）检查：
  memory.available  < 100Mi   （默认 hard eviction threshold）
  nodefs.available   < 10%
  imagefs.available  < 15%
  pid.available      < 10%
        ↓ 触发
按 "驱逐排名" 排序：超出 requests 的程度 + Pod 优先级 + QoS
   BestEffort > Burstable > Guaranteed
   同 QoS 下：使用量超出 requests 最多者优先被杀
        ↓
驱逐 = 优雅删除 Pod（发 SIGTERM，等 grace period），不是直接 kill
```

- **soft eviction（`evictionSoft` + `evictionSoftGracePeriod`）**：先给一段宽限期，允许 Pod 自行释放资源。
- **`system-reserved` / `kube-reserved`**：为系统进程和 K8s 自身预留内存，防止 kubelet 自己被 OOM。
- **重要区分**：**驱逐 ≠ OOM Kill**。驱逐是 kubelet 主动行为（有排名、有优先级），OOM 是内核行为（按 `oom_score_adj` 直接 SIGKILL，无法优雅）。两者都表现为 Pod 消失，但日志和退出码不同。

### 3.5 kube-proxy：Service 转发在内核里怎么落地

#### 3.5.1 三种模式的演进

| 模式 | 原理 | 复杂度 | 性能 | 状态 |
|---|---|---|---|---|
| userspace | 用户态进程监听端口并转发 | O(n) | 差（两次内核/用户态拷贝） | 1.2 起废弃，1.14 移除 |
| **iptables** | netfilter 规则链 + DNAT | **规则数 O(services × endpoints)** | 中（规则多时匹配慢、conntrack 压力大） | 长期默认 |
| **ipvs** | LVS 内核模块（哈希表） | 规则数 O(1) 查找 | 好（万级 Service 也稳） | 1.11 GA，大集群首选 |
| **nftables** | nft 规则集 | 比 iptables 好（集合查找） | 好 | 1.29 beta，1.31+ 逐步成熟 |
| **eBPF（无 kube-proxy）** | Cilium 在 socket/tc 层重定向 | O(1) | 最好（绕过 netfilter） | 主流趋势 |

#### 3.5.2 iptables 模式的数据流（必须能画）

```
外部/集群内请求 → 宿主内核 netfilter：

PREROUTING
  └─ KUBE-SERVICES (匹配 ClusterIP:port)
        └─ KUBE-SVC-<hash>   ← 负载均衡：多个 KUBE-SEP 按概率分配
              ├─ KUBE-SEP-<hash-A>  -j DNAT --to 10.244.1.5:8080   (50%)
              ├─ KUBE-SEP-<hash-B>  -j DNAT --to 10.244.2.7:8080   (50%)
              └─ ...
        （每个 SEP 用 statistic mode random 实现加权随机；
          sessionAffinity: ClientIP 时用 recent 模块做源 IP 黏性）

之后：DNAT 改目标 → 路由 → FORWARD (KUBE-FORWARD) → 到 Pod
回包：conntrack 反向转换 → 源地址恢复为 ClusterIP

本节点 Pod 访问 Service（hairpin）走 OUTPUT 链上的同类规则
NodePort：KUBE-NODEPORTS（在 KUBE-SERVICES 里跳入）
```

**iptables 模式的三大痛点**：
1. **规则数爆炸**：1000 个 Service × 20 个 Endpoint = 2 万条规则，每次包都要线性遍历 → 延迟随规模上升。
2. **conntrack 表满**：`nf_conntrack: table full, dropping packet` 是大集群经典故障，需调 `nf_conntrack_max`。
3. **规则更新开销**：Endpoint 变化时 kube-proxy 要重写大量规则（虽然有 `--sync-period` 和增量同步优化）。

#### 3.5.3 ipvs 模式

- 依赖内核 `ip_vs` 模块，用 `ipvsadm` 查看：`ipvsadm -Ln`。
- **Service → 一组 Virtual Server（ClusterIP:Port）→ Real Server（Pod IP:Port）**。
- 调度算法：`rr`（轮询）、`wrr`、`lc`（最小连接）、`sh`（源地址哈希）→ 对应 `sessionAffinity`。
- **为什么快**：IPVS 用哈希表，查找 O(1)；规则更新是增删哈希项，不是重写整条链。
- **坑**：ipvs 模式仍然需要少量 iptables 规则（用于 SNAT、masquerade、NodePort 的某些路径），且**不支持** `--masquerade-all` 之外的复杂匹配；排查时要同时看 `ipvsadm -Ln` 和 `iptables -L`。

#### 3.5.4 kube-proxy 如何知道 Endpoint 变化

```
kube-proxy 启动 → 用 ServiceAccount token 调 apiserver
  → Informer watch Service + EndpointSlice（1.21+ 用 EndpointSlice 替代 Endpoints）
  → 变更进入 workqueue → syncProxyRules() → 重写内核规则（幂等全量重建）
```

> **EndpointSlice 的意义**：把一个大 Endpoints 对象拆成多个最多 100 个端点的切片 → 单个对象变更只影响一个切片，避免"2000 个 Pod 的 Service 每次变更都推送全量对象给所有节点"（这是 Endpoints 的著名性能瓶颈）。

---

## 第 4 章 Pod 与调度

### 4.1 Pod 是什么：不是"一组容器"，而是"一个共享网络与生命周期的调度单元"

**一句话本质**：Pod 是 K8s 的最小调度单位，它把若干容器**关进同一套 namespace 和一个 cgroup**，使它们像一个"逻辑主机"一样工作。

```
                    Pod（一个 cgroup：kubepods/burstable/pod<UID>）
┌──────────────────────────────────────────────────────────────┐
│  pause 容器（sandbox / infra container）                       │
│    · 镜像极简（几百 KB），只做一件事：sleep                          │
│    · 它持有 net / ipc / uts / (可选)pid namespace              │
│    · 它的存在 = Pod 的网络命名空间"活着"                         │
│                                                              │
│  容器 A ──┐   join 上述 namespace（containerd 的 "container:" 模式） │
│  容器 B ──┘   → 与 pause 共享 netns：同 IP、同端口空间、可用 127.0.0.1 │
│             → 共享 IPC：可用 SysV 共享内存                        │
│             → 默认**不共享 PID**（各自 PID 1 独立）                 │
│             → 共享 volume：挂载同一 volumeMount                  │
└──────────────────────────────────────────────────────────────┘
```

**面试必答：为什么必须有 pause 容器？**
1. **netns 需要一个"持有者"**。如果以业务容器的 namespace 为准，业务容器重启（崩溃重建）时 netns 会跟着消失 → Pod IP 变化、Service 端点抖动、正在建立的连接全部失败。
2. pause 容器与业务容器**生命周期解耦**：业务容器重启，pause 不动，netns/IP 保持稳定。
3. pause 还负责僵尸进程回收（它是 PID ns 的 init，当 `shareProcessNamespace: true` 时尤其重要）。

**`shareProcessNamespace: true` 的影响（1.17 GA）**：
- Pod 内容器共享 PID namespace → 容器 A 能用 `ps` 看到容器 B 的进程；
- 副作用：所有容器里 **PID 1 变成 pause**，业务容器的进程不再是 PID 1 → 信号处理语义变化；`kill -9 1` 会杀 pause（Pod 重建）；
- 用途：sidecar 需要观察/操作主容器进程（如调试 sidecar、日志采集需要读到所有进程）。

### 4.2 Pod 的完整生命周期与状态机

```
创建请求 (Deployment 控制器建 Pod 对象)
   │
   ▼  kube-apiserver 写入 etcd，Pod 处于 Pending（未调度）
┌─────────────┐
│  Pending    │ ← 无 nodeName / 调度失败 / 镜像拉取中 / sandbox 创建中
└──────┬──────┘
       │ scheduler 绑定 nodeName  → kubelet 接管
       ▼
┌─────────────┐
│  Running    │ ← 至少一个容器在运行，或正在启动/重启
└──────┬──────┘
       │ restartPolicy=Always 时不会进入终态（会一直 Running/重启）
       ▼
┌─────────────┬──────────────┬──────────────┐
│  Succeeded  │   Failed     │   Unknown    │
│  (全 0 退出)│ (非 0 退出)  │ (kubelet 失联)│
└─────────────┴──────────────┴──────────────┘
```

**Pod 的 Conditions（比 phase 更有诊断价值）**：
- `PodScheduled`：是否已调度
- `Initialized`：所有 init 容器是否成功
- `ContainersReady`：所有容器 readiness 是否通过
- `Ready`：Pod 是否可接收流量（= ContainersReady + readinessGates 全通过）
- `PodReadyToStartContainers`（1.29+）：sandbox 网络是否就绪

**Init 容器 vs 普通容器 vs Sidecar（1.28+ 原生 sidecar）**：

| | init 容器 | 普通容器 | 原生 sidecar（`restartPolicy: Always` 的 init 容器） |
|---|---|---|---|
| 执行 | **串行，按序，每个必须成功** | 并行 | 在 init 阶段启动，**但一直运行** |
| 资源 | 取所有 init 容器的 max 作为 Pod 的 init 需求 | 累加 | 累加 |
| 探针 | 不支持 | 支持 | 支持 |
| 用途 | 初始化（等 DB、拉配置、改权限） | 业务 | 日志采集、代理、mesh |

**Init 容器的一个关键细节**：Pod 的**有效资源请求 = max(所有 init 容器的 requests) + sum(sidecar 容器 requests)**，而业务容器则是 sum。这是因为 init 容器串行执行、不会同时占用。调度器按这个口径算，`kubectl describe pod` 里能看到 `Init Containers` 的独立资源。

### 4.3 QoS：从调度标签到 OOM 保护级别

| QoS | 判定条件 | `oom_score_adj` | 驱逐优先级 | cgroup 位置 |
|---|---|---|---|---|
| **Guaranteed** | 所有容器都设置了 **cpu/memory 的 requests == limits**（且非 0） | **-997** | 最后 | `kubepods/guaranteed/` |
| **Burstable** | 至少一个容器设了 requests/limits，但不满足 Guaranteed | 2 ~ 999（按 requests 占比计算） | 中间 | `kubepods/burstable/` |
| **BestEffort** | 所有容器都**没设** requests/limits | **1000** | 最先 | `kubepods/besteffort/` |

> **QoS 的真实作用是双重身份**：
> ① 调度器用它决定"节点资源不足时先杀谁"；
> ② **内核 OOM killer 用它决定杀谁**（通过 `oom_score_adj`）—— 这才是"Guaranteed 更安全"的物理原因，不是心理安慰。

**Burstable 的 `oom_score_adj` 计算**（值得知道，面试可说）：
```
if memoryRequestMiB >= 0:
    oomScoreAdj = 1000 - (999 * memoryRequestMiB) / machineMemoryCapacityMiB
```
即请求越多，越接近 0（越不容易被杀）。

### 4.4 调度器全流程：从队列到 bind

```
                    ┌──────────────────────┐
    新的 Pod ──────>│ Scheduling Queue      │
    (未指定 nodeName)│  activeQ / backoffQ / │
                    │  unschedulableQ        │
                    └──────────┬───────────┘
                               │ Pop（优先级 + 入队时间排序）
                    ┌──────────▼───────────────────────────────┐
                    │  调度框架 Scheduling Framework            │
                    │                                          │
                    │  PreEnqueue                              │
                    │    ↓                                     │
                    │  Sort（QueueSort：PrioritySort）          │
                    │    ↓                                     │
                    │  PreFilter   ← 预计算，例如 NodeAffinity  │
                    │    ↓           的节点集合、Pod 资源总量     │
                    │  Filter      ← 硬性筛选，可多个插件       │
                    │    ↓           （任一不过 → 该节点淘汰）    │
                    │  PostFilter  ← 全部节点都不行时：抢占逻辑  │
                    │    ↓                                     │
                    │  PreScore / Score / NormalizeScore       │
                    │    ↓           （给剩余节点打分 0-100）     │
                    │  Reserve     ← 预占资源（防止并发超卖）    │
                    │    ↓                                     │
                    │  Permit      ← 等待（如 gang 调度）        │
                    │    ↓                                     │
                    │  Bind        ← 写 Pod.spec.nodeName       │
                    └──────────────────────────────────────────┘
```

**Filter 插件的完整清单（能说出 5 个以上就很稳）**：

| 插件 | 检查什么 | 典型失败事件 |
|---|---|---|
| `NodeResourcesFit` | CPU/内存/临时存储/扩展资源是否够 | `Insufficient cpu` / `Insufficient memory` |
| `NodeAffinity` | `nodeSelector` / `nodeAffinity` required | `node(s) didn't match Pod's node affinity` |
| `TaintToleration` | 污点与容忍是否匹配 | `node(s) had untolerated taint` |
| `NodeUnschedulable` | `spec.unschedulable`（cordon 的节点） | `node(s) were unschedulable` |
| `PodTopologySpread` | 拓扑分布约束（`whenUnschedulable: DoNotSchedule`） | `would violate pod topology spread constraints` |
| `InterPodAffinity` | Pod 亲和/反亲和的 required 部分 | `node(s) didn't satisfy existing pods' affinity rules` |
| `NodePorts` | 需要的 hostPort 是否被占用 | `node(s) didn't have free ports` |
| `VolumeZone` / `VolumeBinding` | PV 的可用区 / PVC 绑定延迟 | `node(s) had volume node affinity conflict` |
| `NodeName` | 指定了 nodeName 就直接通过（其他插件跳过该节点检查） | — |
| `NodeVolumes` | 节点上卷数量上限 | `Too many attached volumes` |

**Score 插件与权重**：

| 插件 | 打分逻辑 | 默认权重 |
|---|---|---|
| `NodeResourcesFit`（`LeastAllocated`） | 空闲比例越高分越高（**打散**） | 1 |
| `NodeResourcesFit`（`MostAllocated`） | 已分配比例越高分越高（**装箱**，省机器） | 1 |
| `NodeResourcesBalancedAllocation` | CPU/内存使用率**越均衡**分越高 | 1 |
| `ImageLocality` | 节点上已有该镜像 → 高分（加快启动） | 1 |
| `InterPodAffinity` | 已有亲和 Pod 的节点加分 | 1 |
| `TaintToleration` | 容忍污点数量越少越优先（少打扰专用节点） | 3 |
| `PodTopologySpread` | 分布越均匀分越高 | 2 |

**打分公式**：`最终分 = Σ(插件分 × 权重)`，然后取最高分；同分时随机选（**这就是为什么"每次重建 Pod 落到不同节点"**）。

**抢占（Preemption）**：
- PostFilter → 找"如果杀掉某些低优先级 Pod 就能容纳当前 Pod"的节点 → 选"牺牲代价最小"的（优先杀 BestEffort、低优先级、数量少）。
- 被杀的 Pod 不会"恢复"，而是由它的控制器重建（可能又回到原节点 → 需要配合优先级设计）。
- `system-node-critical` / `system-cluster-critical`（`priorityClass` 值极大）几乎不可被抢占。

### 4.5 亲和性与拓扑约束：怎么让 Pod "落到该落的地方"

```yaml
# 1) nodeSelector（最简单，仅支持精确匹配）
nodeSelector: { disktype: ssd }

# 2) nodeAffinity（支持操作符与软约束）
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:   # 硬：不满足就不调度
      nodeSelectorTerms:
      - matchExpressions:
        - { key: disktype, operator: In, values: [ssd] }
        - { key: zone, operator: NotIn, values: [zone-c] }
    preferredDuringSchedulingIgnoredDuringExecution:  # 软：加权偏好
    - weight: 80
      preference:
        matchExpressions:
        - { key: gpu-type, operator: In, values: [a100] }

# 3) podAffinity / podAntiAffinity（以"已有 Pod"为参照，靠 topologyKey 定义拓扑域）
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:   # 同一个 zone 只能有一个副本 → 高可用
    - labelSelector:
        matchLabels: { app: myapp }
      topologyKey: topology.kubernetes.io/zone

# 4) topologySpreadConstraints（比反亲和更可控的"均匀分布"）
topologySpreadConstraints:
- maxSkew: 1                          # 各域数量差的最大值
  topologyKey: kubernetes.io/hostname
  whenUnschedulable: DoNotSchedule     # 或 ScheduleAnyway（软）
  labelSelector: { matchLabels: { app: myapp } }
```

**`topologyKey` 的本质**：节点 label 的 key。**同一 value 的节点属于同一拓扑域**。常用：
- `kubernetes.io/hostname`（域 = 单节点）
- `topology.kubernetes.io/zone`（域 = 可用区）
- 自定义 label（如 `rack-id`）

**`IgnoredDuringExecution` 的含义**：**只在调度时检查，运行中标签变化不会驱逐**。这就是为什么"节点改了 label 后 Pod 不会自动迁移"。

### 4.6 污点与容忍

```yaml
# 节点打污点（三种效果）
kubectl taint node n1 gpu=true:NoSchedule          # 不调度新 Pod（已运行的不动）
kubectl taint node n1 maintenance:NoExecute        # 还驱逐不容忍的已有 Pod
kubectl taint node n1 disk=full:PreferNoSchedule   # 尽量不调度

# Pod 容忍
tolerations:
- key: gpu
  operator: Equal        # 或 Exists（只判 key 是否存在）
  value: "true"
  effect: NoSchedule
- operator: Exists        # tolerationSeconds 省略 = 永久容忍
  effect: NoExecute
  tolerationSeconds: 3600   # 仅在指定的 key 上，允许被驱逐前额外停留 1 小时
```

**内置污点（必须知道）**：
| 污点 | 谁会自动加上 | 含义 |
|---|---|---|
| `node.kubernetes.io/not-ready` | NodeController | 节点 NotReady |
| `node.kubernetes.io/unreachable` | NodeController | 节点失联 |
| `node.kubernetes.io/memory-pressure` | kubelet | 内存压力 |
| `node.kubernetes.io/disk-pressure` | kubelet | 磁盘压力 |
| `node.kubernetes.io/pid-pressure` | kubelet | PID 压力 |
| `node.kubernetes.io/unschedulable` | `kubectl cordon` | 已 cordon |
| `node.kubernetes.io/network-unavailable` | kubelet | 网络未就绪 |

> Pod 默认会自动容忍 `not-ready`/`unreachable`（默认 `tolerationSeconds: 300`），→ **节点失联 5 分钟后 Pod 才会被驱逐**。这个 5 分钟的"假死期"是很多故障处理的盲区（要改 Pod 的 tolerationSeconds 才能缩短）。

---

## 第 5 章 控制器与工作负载

### 5.1 控制器的通用原理（所有控制器都是这一个模板）

```
                    ┌────────────────────────────────────────────┐
                    │  Informer（watch API 对象，维护本地缓存 Indexer）│
                    └───────────────────┬────────────────────────┘
                                        │ 事件 → 塞进 workqueue（按 key 去重）
                    ┌───────────────────▼────────────────────────┐
                    │  workqueue                                │
                    │   同一个对象只保留一份 → 天然抗抖动          │
                    └───────────────────┬────────────────────────┘
                                        │ 一个或多个 worker goroutine
                    ┌───────────────────▼────────────────────────┐
                    │  syncHandler(key)                         │
                    │    1. 从 Indexer 取"实际状态"               │
                    │    2. 计算"期望状态"（spec）                │
                    │    3. 差值 → 创建/更新/删除 API 对象         │
                    │    4. 出错 → AddRateLimited（指数退避重入队）│
                    └────────────────────────────────────────────┘
```

**关键设计原则（面试常考"为什么这样设计"）**：
1. **不对等观测（level-triggered，而非 edge-triggered）**：控制器每次处理都**从当前实际状态重新计算**，而不是"收到一个"创建 Pod"的命令就创建"。所以**丢事件不会导致状态错误**，只会延迟收敛。
2. **幂等**：同一个 key 处理多次结果一致。这就是为什么控制器可以"无脑重试"。
3. **单一写者**：每类对象只有一个控制器负责（Deployment 归 DeploymentController，Pod 归 kubelet+调度器……）→ 避免竞争。**不要自己写程序去改 StatefulSet 管理的 Pod**。

### 5.2 Deployment 滚动更新：`maxSurge` / `maxUnavailable` 的真实算法

```
Deployment (replicas=10, maxSurge=25%, maxUnavailable=25%)
  → 期望副本 10，允许"多出"2.5→向上取整 3，允许"不可用"2.5→向下取整 2

滚动过程（新版 ReplicaSet 逐步扩容，旧版逐步缩容）：
  旧RS: 10 → 8 → 6 → 4 → 2 → 0
  新RS:  0 → 3 → 5 → 7 → 8 → 10
  每一步都必须满足：总Pod ≤ 10+3=13，且可用Pod ≥ 10-2=8
```

- **取整规则**：`maxSurge` 向上取整，`maxUnavailable` 向下取整 → 保证"至少不会少太多"。若 `maxUnavailable=0` 则必须 `maxSurge>0`（否则无法滚动）。
- **`progressDeadlineSeconds`**（默认 600s）：超过这个时间还没进展 → Deployment 的 `Progressing` 条件变为 `False` 并给出原因（但**不会自动回滚**，回滚要手动 `kubectl rollout undo`）。
- **`revisionHistoryLimit`**（默认 10）：保留的旧 ReplicaSet 数量，用于回滚。设 0 就无法回滚了。
- **回滚的机制**：`rollout undo` 就是把旧 ReplicaSet 的 Pod 模板 `spec` 复制回 Deployment.spec，然后重新走一遍滚动 —— **不是"快照恢复"**，所以回滚期间同样会走 maxSurge/maxUnavailable。
- **`Deployment` 的 `Available` 条件** = `minReadySeconds` 期间保持 Ready 的副本数 ≥ replicas - maxUnavailable。
- **常见坑**：镜像 tag 用 `latest` + `imagePullPolicy: Always` 时，`rollout` 可能"看起来成功了但跑的还是旧代码"（因为新 RS 的 podTemplate 没变 → 不会触发滚动，需要 `kubectl rollout restart`）。

### 5.3 StatefulSet：为什么有状态服务必须用它

| 能力 | Deployment | StatefulSet |
|---|---|---|
| Pod 名称 | 随机后缀（`app-7d9f-abcde`） | **有序稳定**（`app-0`、`app-1`） |
| 网络标识 | 无（Service 随机负载均衡） | **稳定的 DNS**（`app-0.svc.ns.svc.cluster.local`，靠 **Headless Service**） |
| 存储 | 所有副本共享同一 PVC（通常是错误用法） | **每个副本独立 PVC**（`volumeClaimTemplates`，`data-app-0`） |
| 启停顺序 | 并行 | **默认 OrderedReady**：0 → 1 → 2 依次就绪才启动下一个；缩容反向 2 → 1 → 0 |
| 更新策略 | RollingUpdate | `RollingUpdate`（倒序逐个，**每个就绪后才更新下一个**）/ `OnDelete` |
| 扩缩容保证 | 无 | `podManagementPolicy: OrderedReady` / `Parallel` |
| 删除 Pod 后 | 副本数自动补齐（可能是新 Pod） | 同名重建（`app-0` 还是 `app-0`），PVC 保留 |

**StatefulSet 的隐含契约（这些细节能体现是否真用过）**：
1. **必须配 Headless Service**（`clusterIP: None`）才能有稳定 DNS；普通 Service 也能用，但 DNS 是负载均衡的。
2. **PVC 不会随 Pod/StatefulSet 删除而删除**（`persistentVolumeClaimRetentionPolicy` 1.27+ 才可配置 `whenDeleted/whenScaled: Delete`）。→ 手动清理是常见事故。
3. **缩容只删 Pod，不删 PVC**；再扩容时**会复用同名 PVC** → 数据还在，但如果之前的数据格式变了可能启动失败。
4. **`podManagementPolicy: Parallel`** 让所有 Pod 同时启停（适合"彼此无顺序依赖但要快"的场景），但会破坏"有序"的假设。
5. **它的滚动更新是"逐个等待 Ready"** → 20 个副本的更新可能非常慢（每个都要过 `minReadySeconds` + 探针）。
6. **不适合做"最终一致的集群成员变更"**（如 etcd 集群自己管成员），通常还需要应用层协调。

### 5.4 DaemonSet / Job / CronJob

- **DaemonSet**：每个（满足 nodeSelector/affinity 的）节点跑一个 Pod。控制器按节点计算"应该有几个"，给每个节点建一个 Pod 并**直接指定 `nodeName`**（绕过调度器，用 `nodeAffinity` 表达目标节点）。常用于日志采集（fluentd）、监控（node-exporter）、网络插件（calico-node）。
  - 它会**自动加 toleration** 使 Pod 能在有污点的节点上运行（`operator: Exists, effect: NoSchedule/NoExecute`），否则有污点的节点就没有采集器。
- **Job**：`completions`（要成功几次）、`parallelism`（并发几个）、`backoffLimit`（失败重试上限，默认 6）、`activeDeadlineSeconds`（总超时）、`ttlSecondsAfterFinished`（完成后多久清理）。**Job 的 Pod 是 `restartPolicy: Never` 或 `OnFailure`**，不能是 Always。
  - `podFailurePolicy`（1.31 GA）：区分"可重试失败"与"不可重试失败"（如配置错误不该重试）。
- **CronJob**：cron 表达式 + `concurrencyPolicy`（Allow/Forbid/Replace）+ `startingDeadlineSeconds`（错过时间窗多久内还补跑）+ `suspend`。
  - 注意：**CronJob 不保证精确时刻**，且"错过的调度"最多补跑有限次数（`startingDeadlineSeconds` 逻辑）。

### 5.5 HPA：自动扩缩容的真实逻辑

```
HPA 控制器每 15s（--horizontal-pod-autoscaler-sync-period）执行：
   ① 取当前指标（metrics-server / custom metrics adapter / external metrics）
   ② 计算期望副本：
        desiredReplicas = ceil( currentReplicas × ( currentMetricValue / desiredMetricValue ) )
   ③ 与 minReplicas/maxReplicas 取边界
   ④ 应用"容忍度"（默认 tolerance=0.1）：只有偏差超过 10% 才动，防止抖动
   ⑤ 应用"缩容稳定窗口"（--horizontal-pod-autoscaler-downscale-stabilization，默认 5min）：
      取过去 5 分钟内的**最大**推荐值 → 防止"一波动就缩容"
   ⑥ 更新 scale 子资源（Deployment 的 replicas）
```

**HPA 的工程坑（高频面试题）**：
- **HPA 只读取 requests 的百分比**：`targetAverageUtilization: 50` 是"实际使用 / requests 的 50%"。**Pod 没设 requests 时 HPA 直接不工作**（无法算百分比）。
- **扩容快、缩容慢**：缩容有稳定窗口；扩容立即生效。→ 所以"缩放抖动"通常表现为"缩得太慢"。
- **HPA 与 VPA 不能同时对同一资源（CPU/内存）生效**，会打架。
- **指标延迟**：metrics-server 默认 15s 抓一次、聚合周期 60s → 扩缩容有分钟级滞后。突发流量要靠**提前扩容/集群容量预留**（cluster autoscaler / Karpenter）补足。
- **扩容的"排队"问题**：HPA 扩出 Pod 后还要调度 + 拉镜像 + 启动 → 冷启动时间可能远超流量峰值窗口。生产上要配 **镜像预热 + PDB（PodDisruptionBudget）+ 预留 buffer**。

---

## 第 6 章 Service / Ingress / CNI 网络

### 6.1 Service 的四种类型与本质

**一句话本质**：Service 是一个**稳定虚拟 IP + 一组动态 Endpoint 的映射**，它把"Pod IP 会变"这件事屏蔽掉了。

| 类型 | 行为 | 访问范围 | 实现 |
|---|---|---|---|
| **ClusterIP**（默认） | 分配一个集群内虚拟 IP | 仅集群内 | kube-proxy 内核规则（DNAT） |
| **Headless**（`clusterIP: None`） | **不分配 IP**，DNS 直接返回所有 Pod IP | 集群内 | 无 iptables 规则，纯 DNS（`A` 记录多条 / `SRV`） |
| **NodePort** | 在每个节点上监听一个端口（30000-32767） | 集群外 | iptables/ipvs 规则 + `KUBE-NODEPORTS` |
| **LoadBalancer** | 向云厂商申请一个外部 LB（SLB/ELB） | 公网/外网 | 云控制器（cloud-controller-manager）调 IaaS API + NodePort |
| **ExternalName** | CNAME 到外部域名 | — | 纯 DNS，无代理 |

**ClusterIP 是"虚拟"的——它根本不存在于任何网卡上**。所以你 `ping ClusterIP` 不通（ICMP 没有对应规则），但 TCP 通。这是最常见的误解。

**Headless Service 的三种 DNS 记录（面试可展开）**：
```
普通 Service（有 ClusterIP）：
  my-svc.ns.svc.cluster.local.  A  10.96.0.10        ← 只有一条，指向虚拟 IP

Headless Service（clusterIP: None）：
  my-svc.ns.svc.cluster.local.  A  10.244.1.5        ← 直接返回所有 Pod IP
                                A  10.244.2.7
  （客户端自己做负载均衡/选主，这是 StatefulSet 的基础）

带名字的 Pod（StatefulSet 或 hostname 指定）：
  pod-0.my-svc.ns.svc.cluster.local.  A  10.244.1.5   ← 稳定的单 Pod 记录

SRV 记录（给非 HTTP 协议用）：
  _http._tcp.my-svc.ns.svc.cluster.local. SRV 0 33 8080 pod-0.my-svc...
```

### 6.2 Endpoint / EndpointSlice 的生成链路

```
   Service 创建
        ↓
  EndpointController（在 controller-manager 里）watch Service + Pod
        ↓
  筛选：Pod 的 labels 匹配 Service 的 selector？
       且 Pod 的 readiness（Ready 条件）为 true？  ← 关键！
       且 Pod 不在 terminating 状态？
        ↓ 通过
  写入 EndpointSlice（每片最多 100 个 endpoint，带 topology 信息）
        ↓
  kube-proxy watch EndpointSlice → 更新内核转发规则
```

**关键细节**：
- **`publishNotReadyAddresses: true`** 会让"未 Ready 的 Pod"也进入 Endpoints，用于"必须能访问但还没就绪"的场景（如集群成员互相发现）。
- **Pod 处于 `Terminating` 时**会被**立刻**从 Endpoints 移除（1.26 起 `Endpoints` 会标记 `terminating` 并逐步摘除）→ 这就是"为什么不建议立刻重建同名 Pod"以及"为什么需要 preStop 延时"。
- **Readiness 是摘流量的唯一开关** → 灰度发布时把"金丝雀 Pod"的 readiness 设为 false 就是"部署但不接流量"，这是很实用的技巧。

### 6.3 Ingress 与 Gateway API

```
Ingress 解决"七层路由"：把不同域名/路径的流量分发到不同 Service

        外部流量 → Ingress Controller（一个真正在跑的 Pod，如 nginx-ingress）
                        │
                        │ 读 Ingress 对象（规则）→ 生成 nginx.conf
                        ▼
                  ┌──────────┬──────────┐
             host: a.com     host: b.com
             path: /api      path: /
                  │               │
             Service A        Service B
```

**必须澄清的三点（面试高频误区）**：
1. **Ingress 只是一个"声明对象"**，它本身不会做任何转发。必须有 **Ingress Controller**（nginx-ingress、Traefik、HAProxy、Envoy-based）在实际干活。
2. **Ingress 的语法由 Controller 决定**（`nginx.ingress.kubernetes.io/xxx` 这些 annotation 只对 nginx-ingress 有效）→ 换 Controller 可能要改配置。
3. **Ingress 不支持 TCP/UDP**（除少数 Controller 的自定义配置）→ 四层要靠 Service 的 NodePort/LB，或 `Gateway API` + TCPRoute。

**Gateway API（Ingress 的继任者）**：把角色拆分得更清晰 —— `GatewayClass`（基础设施提供方）、`Gateway`（运维）、`HTTPRoute`（开发），支持多协议、流量镜像、请求头匹配等。这是当前演进方向。

### 6.4 CNI：Pod 网络到底是怎么被配置的

**调用链**：
```
kubelet 创建 pod sandbox（pause 容器）之后：
   kubelet → CRI: RunPodSandbox
       → containerd（或 runtime）读取配置：/etc/cni/net.d/*.conf
       （通常 containerd 的原生 CNI 配置，或由 CNI 插件如 calico 提供的配置）
       → 执行 CNI 插件二进制：/opt/cni/bin/<plugin>
       → 通过 stdin 传 JSON，环境变量传参数：
            CNI_COMMAND=ADD
            CNI_CONTAINERID=<sandbox-id>
            CNI_NETNS=/proc/<pid>/ns/net
            CNI_IFNAME=eth0
            CNI_ARGS=K8S_POD_NAME=...;K8S_POD_NAMESPACE=...
            CNI_PATH=/opt/cni/bin
       → 插件返回 JSON：{ "ips": [...], "routes": [...], "dns": {...} }
```
- **CNI 插件必须做三件事**：创建 veth pair 并把一端放进 sandbox netns、给 eth0 配 IP、设置路由（通常网关指向宿主侧的 bridge/对端地址）。
- **`CNI_COMMAND=DEL`** 在 Pod 删除时调用，负责清理（kubelet 会在 sandbox 销毁前调用）。
- **`CHECK`**（1.10+）可让 kubelet 周期性校验网络状态。

#### 6.4.1 三大 CNI 方案对比（面试必答）

| | Flannel | Calico | Cilium |
|---|---|---|---|
| 数据面 | VXLAN / host-gw / UDP（弃用） | BGP 路由、IPIP、eBPF | **纯 eBPF**（tc / XDP / socket） |
| Pod 网段分配 | 每节点一个 /24 子网（由 flanneld 分配） | IPPool（IPAM 插件） | IPAM（cluster-pool / kvstore） |
| 跨节点通信 | VXLAN 封装（叠加网络） | BGP 学习对端路由（**无封装，性能好**）或 IPIP 隧道 | 原生路由 / 隧道 / eBPF 直连 |
| 网络策略（NetworkPolicy） | **不支持**（原生 flannel 不做策略） | 支持（Felix 把策略转成 iptables/eBPF） | 支持（基于 **identity**，非 IP） |
| kube-proxy 替代 | 否 | 可（eBPF 模式，1.28 后称 "calico ebpf"） | **可**（`kubeProxyReplacement`） |
| 性能 | 中（VXLAN 有封装开销，MTU 变小） | 好（BGP 无封装） | 最好（绕过 netfilter/conntrack） |
| 复杂度 | 低 | 中 | 高 |
| 典型适用 | 小中型集群、简单场景 | 中大型、需要网络策略 | 大规模、需要可观测/高性能 |

**Flannel 三种后端的工作方式（能讲清原理就够）**：
```
① VXLAN（默认，推荐）
   容器 → 主机 bridge(cni0) → flannel.1（VTEP，虚拟网卡）
   → 按 FDB 表查到对端节点 VTEP 的 MAC → 封装成 VXLAN(UDP 8472) 发到对端物理 IP
   → 对端 flannel.1 解封装 → bridge → 目标容器
   代价：封装 → 有效 MTU 减少（1500 - 50 = 1450），需注意应用 MTU 设置

② host-gw（仅同二层子网可用，性能最好）
   每节点把"对端 Pod 网段"写成一条静态路由：ip route add 10.244.2.0/24 via <对端节点IP>
   → 无封装，直接二层转发。缺点：节点必须二层互通（同一个交换机/子网）

③ UDP（已不推荐）
   用户态转发，性能极差，仅历史遗留
```

**Calico 的 BGP 模式原理（加分项）**：
- 每个节点跑一个 `BGP Speaker`（bird/GoBGP），把自己的 Pod 网段通过 BGP 宣告给对端节点（**node-to-node mesh**）或路由反射器（**RR**，大规模必须用，否则连接数 n²）。
- 上游路由器学到这些路由 → **Pod IP 在物理网络里直接可达**，无需 NAT/隧道 → 性能最优、可与其他系统直连。
- **`Typha` 组件**：Calico 的 Felix（每节点一个）都要 watch apiserver 的 CRD，几千节点会把 apiserver watch 打爆 → Typha 作为"缓存代理"，让 Felix 只连 Typha。**这是大规模 Calico 必备组件**。

**Cilium 的 eBPF 核心机制（⭐⭐⭐⭐ 难点，必讲）**：
- 在 **socket 层（`connect`/`sendmsg`）** 就用 eBPF 程序把目标 ClusterIP **直接改写成 Pod IP** → 包根本不出协议栈、不经过 iptables/conntrack → 这就是它比 ipvs 更快的原因（省掉整条 netfilter 路径）。
- 在 **tc（traffic control）层** 处理跨节点流量、策略执行（L3/L4/L7）。
- **XDP** 用于 DDoS 前置过滤（在网卡驱动层就丢包）。
- **安全策略基于 identity**（Pod label 的哈希），不是 IP → Pod 重建换 IP 后策略自动生效，且策略数不随 IP 数量增长（用 eBPF map 存）。
- 代价：依赖较新内核（建议 5.4+）、排障工具链不同（`cilium monitor`、`hubble`）、某些场景（如需要 iptables 的旧应用）不兼容。

### 6.5 集群 DNS：CoreDNS 的工作方式

```
Pod 内 /etc/resolv.conf（由 kubelet 生成）：
  nameserver 10.96.0.10                  ← CoreDNS 的 ClusterIP（kube-dns Service）
  search  myns.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

解析 "redis" 时（ndots:5 表示名字里点少于 5 个就先拼 search 域）：
  redis.myns.svc.cluster.local.  → 命中 → 返回 ClusterIP
  （所以短名字能解析的前提是 ndots 与 search 域配置正确）

解析 "api.example.com"（点 ≥5? 不，是 2 个点 → 仍先试 search 域 → 4 次失败查询）
  → 这就是为什么"访问外部域名会慢几毫秒"：ndots:5 导致先试 4 个集群域
```

**排障要点**：
- `ndots:5` + 外部域名 → 每次解析多 4 次 NXDOMAIN 往返。优化：`dnsConfig.options: ndots:2`，或直接用 FQDN（带结尾点）。
- **CoreDNS 的 `hosts` / `forward` 插件**：默认 forward 到 `/etc/resolv.conf` 的 upstream；节点用 systemd-resolved 的 `127.0.0.53` 时要注意 CoreDNS 是否配了 `forward . /etc/resolv.conf`（可能形成回环）。
- **DNS 是集群的隐形单点**：CoreDNS 副本数不足 + HPA/PDB 缺失 → DNS 抖动会导致"整个集群的服务 inexplicably 变慢"。
- **`dnsPolicy`**：`ClusterFirst`（默认）、`Default`（用节点 DNS）、`ClusterFirstWithHostNet`（hostNetwork 的 Pod 想用集群 DNS 必须设这个）、`None`（自定义）。

---

## 第 7 章 存储：PV / PVC / CSI

### 7.1 三层解耦的设计意图

```
   ┌──────────────┐         ┌──────────────┐        ┌──────────────┐
   │  PVC（需求）  │──绑定──>│  PV（供给）   │──实现──>│  后端存储     │
   │ 我要 10Gi    │         │ 一块 10Gi 卷  │        │ Ceph / EBS /  │
   │ ReadWriteOnce│         │ RWO/RWX...   │        │ NFS / 本地盘  │
   └──────────────┘         └──────────────┘        └──────────────┘
                            ↑
                  由 StorageClass 动态供给（provisioner/CSI）

   传统方式的问题：Pod 直接写"我要一个 NFS 路径" → 应用与存储耦合、
   换存储要改应用、跨环境不可移植。
   K8s 的方式：应用只声明"我要 10Gi 可读写"（PVC），存储由平台侧提供（PV/StorageClass）。
```

### 7.2 AccessModes 的真实语义（极易答错）

| AccessMode | 含义 | 真实约束 |
|---|---|---|
| `ReadWriteOnce` (RWO) | **单个节点**可读写 | 多个 Pod **在同一节点**时可以共用；跨节点不行 |
| `ReadOnlyMany` (ROX) | 多节点只读 | 由后端支持 |
| `ReadWriteMany` (RWX) | **多节点**可读写 | 需要支持共享的文件存储（NFS、CephFS、SMB）；**块存储（EBS/RBD）不支持** |
| `ReadWriteOncePod` (RWOP, 1.22+) | **单个 Pod** 独占 | 语义最强，依赖 CSI 支持 |

> **高频误区**：`RWO` ≠ "只能一个 Pod 用"，而是"只能一个**节点**挂载"。同一个节点上的两个 Pod 用同一个 RWO PVC 是可以的（但通常不建议，因为没有并发写保护）。

### 7.3 CSI：存储插件标准

```
CSI Driver = Controller Service（控制面） + Node Service（数据面）

  ┌─ external-provisioner ──> 处理 PVC 创建/删除 → CreateVolume/DeleteVolume
  ├─ external-attacher    ──> 处理 VolumeAttachment → ControllerPublishVolume
  ├─ external-resizer     ──> 处理 PVC 扩容 → ControllerExpandVolume / NodeExpandVolume
  ├─ external-snapshotter ──> 处理 VolumeSnapshot → CreateSnapshot
  └─ external-health-monitor ─> 上报卷健康

Node 侧：csi-node 插件以 DaemonSet 部署，kubelet 通过 CSI 调 NodeStageVolume / NodePublishVolume
        → 真正的 mount 由它在节点上执行
```

**Pod 挂载卷的完整链路（面试可展开）**：
```
PVC 创建 → provisioner 调 CreateVolume（后端真的创建了卷）
  ↓
Pod 调度 → VolumeBinding 插件：如果 PVC 未绑定，先"延迟绑定"（等调度到节点再选 PV）
  ↓
kubelet 的 volumeManager：AttachVolume（controller 侧授权）
  → NodeStageVolume（节点上先 mount 到全局目录，如 /var/lib/kubelet/plugins/kubernetes.io/csi/...）
  → NodePublishVolume（bind mount 到 Pod 的挂载点）
  ↓
容器启动，看到的是 bind mount 后的目录
```

**关键概念**：**VolumeBinding 插件的 `WaitForFirstConsumer`**（StorageClass 里配 `volumeBindingMode`）—— 它把"PV 绑定"推迟到 Pod 被调度后，这样动态供给的卷就能落在与 Pod 同一个可用区（否则可能出现"跨 AZ 挂载失败"或"跨区性能差"）。

### 7.4 回收策略与数据安全

| `persistentVolumeReclaimPolicy` | 行为 | 风险 |
|---|---|---|
| `Retain` | PVC/PV 删除后卷保留，需手工处理 | 安全，但会积累垃圾 |
| `Delete` | 自动删除后端卷 | **极易造成数据永久丢失**（误删 PVC = 删数据） |

**生产建议**：核心数据一律 `Retain` + 定期备份（Velero）；StorageClass 默认策略显式声明；**禁止在 CI 里对生产 PVC 执行任何删除操作**。

**本地存储的坑（`local-path` / `hostPath`）**：
- Pod 被调度到别的节点 → 数据不存在 → 应用看似"数据丢失"。必须配合 **nodeAffinity 固定节点** 或使用分布式存储。
- 单副本存储没有 HA 能力 → 节点故障 = 数据不可用。

### 7.5 emptyDir / configMap / secret 的本质

| 卷类型 | 底层 | 生命周期 | 注意 |
|---|---|---|---|
| `emptyDir` | 节点上的临时目录（或 tmpfs） | **Pod 删除即销毁** | 适合缓存、临时文件；`medium: Memory` 走 tmpfs 但计入内存 |
| `configMap` | 把 ConfigMap 的 key 映射成文件 | 随 Pod | 挂载后**只读**；更新 ConfigMap 不会更新已挂载的文件（需重启或 reload） |
| `secret` | 与 configMap 同机制，值是 base64 | 随 Pod | **base64 不是加密**！只是编码；真正的加密要靠 etcd 加密（KMS）+ `encryptionConfig` |
| `projected` | 组合多个来源（SA token、configmap、downwardAPI） | 随 Pod | 1.21+ SA token 就是通过 projected volume 注入，带 audience 和过期时间 |
| `downwardAPI` | 把 Pod 元信息（name/namespace/limits/labels）写成文件 | 随 Pod | 适合应用读取自身规格做自适配 |

---

## 第 8 章 安全：认证、授权、准入

### 8.1 apiserver 请求处理的四道门

```
kubectl ──HTTPS(mTLS/token)──> kube-apiserver

  ① 认证 Authentication：「你是谁？」
      · X.509 客户端证书（kubelet、kube-proxy 用，CN 作为 用户名）
      · ServiceAccount Token（JWT，projected volume 注入）
      · OIDC / Webhook（企业 SSO 集成）
      · Bootstrap Token（节点加入集群）

  ② 授权 Authorization：「你能做什么？」
      · RBAC（Role/ClusterRole + RoleBinding/ClusterRoleBinding）
      · Node 授权器（kubelet 的请求：只能读自己节点相关的对象）
      · ABAC（已基本废弃）

  ③ 准入 Admission：「这个请求「内容」合不合法？」
      · MutatingAdmissionWebhook（可改对象：注入 sidecar、加默认值、改 resource）
      · ValidatingAdmissionWebhook（只校验，拒绝）
      · 内置准入：LimitRanger、ResourceQuota、ServiceAccount、Priority、
        PodSecurity（1.25 起取代 PSP）、TaintNodesByCondition...

  ④ 写入 etcd（带 revision），并扇出 watch 事件
```

**顺序很重要**：认证 → 授权 → 变更准入 → **校验准入（最后）** → 写对象。所以 Validating 看到的是"已被 Mutating 修改后"的对象。

### 8.2 RBAC 的关键点

```yaml
# Role（命名空间内） vs ClusterRole（集群范围）
kind: Role
rules:
- apiGroups: [""]                 # 核心 API 组（pod/service/configmap）
  resources: ["pods", "pods/log"] # subresource 要单独列
  verbs: ["get", "list", "watch"] # create/update/patch/delete/deletecollection

# 绑定
kind: RoleBinding
subjects:
- kind: ServiceAccount            # 或 User / Group
  name: reader
  namespace: prod
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```

**必须知道的坑**：
1. **`pods/log` 是独立资源**，只给 `pods` 权限看不了日志。
2. **`list` 权限的隐含授权**：有 `list` 就能列举所有这类对象（即使没有 `get`）；有 `watch` 就能持续感知变化。→ 权限审计要关注这两个。
3. **`RoleBinding` 可以引用 `ClusterRole`** → 把集群级权限**限制到某个 namespace**（这是最常用的"最小权限"技巧）。
4. **ServiceAccount 默认不会自动挂 token 到 Pod？** 会（`automountServiceAccountToken: true` 默认）。不需要访问 apiserver 的 Pod 应显式设为 false，**减少泄露面**（这是 PSA `restricted` 级别的检查项）。
5. **`default` ServiceAccount 也存在**，且很多 Chart 用它 → 别给它绑高权限。

### 8.3 Pod Security：从 PSP 到 PSA

- **PSP（PodSecurityPolicy）在 1.25 被移除**（设计缺陷：绑定模型复杂、难以理解、无法表达"部分字段"）。
- **PSA（Pod Security Admission）** 用命名空间 label 表达三档：
  | 级别 | 约束 |
  |---|---|
  | `privileged` | 无限制（默认） |
  | `baseline` | 禁止 privileged、hostNetwork/hostPID/hostIPC、hostPath、危险的 capabilities（除白名单）、禁止 `allowPrivilegeEscalation` 之外的越权 |
  | `restricted` | 在 baseline 之上：必须 `runAsNonRoot`、必须 drop ALL capabilities、必须设 `seccompProfile`、禁止 `allowPrivilegeEscalation`、必须设 resources limits |
```yaml
# namespace 上加 label 启用
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/audit: restricted      # 只记录不拒绝
pod-security.kubernetes.io/warn: restricted
```

### 8.4 镜像与供应链安全（当前热点）

```
构建 → 扫描 → 签名 → 分发 → 部署时校验 → 运行时监控
 │       │      │      │         │            │
 │       │      │      │         │            └─ Falco / Tetragon（eBPF 行为监控）
 │       │      │      │         └─ admission 校验签名（cosign verify → Kyverno/Connaisseur 策略）
 │       │      │      └─ 私有 registry（Harbor）作为唯一入口 + 镜像代理缓存
 │       │      └─ cosign / notation（Sigstore，keyless 签名基于 OIDC）
 │       └─ trivy / grype / snyk（扫 OS 包与语言依赖）+ SBOM 生成（syft）
 └─ 不落密钥（BuildKit secret）、固定 base digest、最小镜像（distroless）
```

---

## 第 9 章 可观测性与排障体系

### 9.1 指标从哪来

```
容器/节点指标来源：
  kubelet 内置 cAdvisor（1.24 起 metrics 字段走 CRI，cAdvisor 内嵌）
     → /metrics/cadvisor（container_cpu_usage_seconds_total、container_memory_working_set_bytes...）
     → /metrics（kubelet 自身：apiserver 请求延迟、pod 启动时长、PLEG 延迟）

  注意：container_memory_working_set_bytes 才是"有效内存"，
       container_memory_usage_bytes 含 page cache，会误导。

节点级：node_exporter（CPU/内存/磁盘/网络/文件句柄）
控制面：apiserver 的 --metrics（请求 QPS、延迟、etcd 请求耗时）、etcd 自身指标、
       scheduler 的调度延迟（scheduler_scheduling_duration_seconds）、
       controller 的 workqueue 深度（用来判断"控制器跟不上事件量"）
```

### 9.2 排障的黄金顺序（面试给场景题时按这个答）

```
① kubectl get pod -o wide        → 看 phase / node / IP / restarts
② kubectl describe pod           → 看 Events（**90% 的问题在这一行**）
③ kubectl logs [-p] [--previous] → 看应用日志；-p 看 init 容器，--previous 看上次崩溃
④ kubectl exec / nsenter         → 进容器看现场（进程、网络、磁盘、环境变量）
⑤ kubectl get events --sort-by=.lastTimestamp
⑥ kubectl top pod/node           → 资源热点
⑦ 看节点：dmesg（OOM/kernel panic）、journalctl -u kubelet、crictl ps/stats
```

### 9.3 高频故障 ↔ 根因对照表

| 现象 | 常见根因 | 验证动作 |
|---|---|---|
| `Pending` + `FailedScheduling: 0/10 nodes are available: 10 Insufficient cpu` | 资源不足（requests 太大 / 节点已满） | `kubectl describe node` 看 Allocated resources |
| `Pending` + `unbound immediate PersistentVolumeClaims` | PVC 未绑定（无可用 PV / StorageClass 不存在） | `kubectl get pvc,pv`；看 provisioner 日志 |
| `Pending` + `node(s) had untolerated taint` | 缺 toleration | `kubectl describe node | grep Taints` |
| `ContainerCreating` 卡住 | 挂载卷慢（CSI 卡）、CNI 失败、镜像拉取慢 | `kubectl describe pod` Events；节点上 `crictl ps -a` |
| `ImagePullBackOff` | 镜像名/tag 错、鉴权失败（imagePullSecrets）、网络不通/被墙 | `crictl pull` 手工试；检查 SA 的 secret |
| `CrashLoopBackOff` | 应用自身启动失败（配置/依赖/权限） | `kubectl logs --previous` |
| `OOMKilled: true`, exit 137 | 超过 `limits.memory` | 看 `memory.current`/`memory.stat`，调 limits 或修泄漏 |
| `exit 139` (SIGSEGV) | 应用段错误（常见于 CGO、错误的 ldd） | core dump / `dmesg` |
| Pod `Running` 但 `READY 0/1` | readiness 探针不过 | `kubectl describe` 看 `Readiness probe failed` |
| Service 不通 | selector 不匹配 / endpoints 为空 / readiness 未过 / kube-proxy 规则缺失 / 网络策略阻断 | `kubectl get endpointslices`；`iptables -t nat -L KUBE-SERVICES` |
| 节点 `NotReady` | kubelet 挂了 / 心跳 Lease 过期 / 容器运行时故障 / 磁盘满 / PLEG unhealthy | `journalctl -u kubelet`；`df -h`；`crictl info` |
| 集群整体不可写 | etcd 空间不足（NOSPACE）/ etcd 慢（fsync 延迟） | `etcdctl endpoint status --cluster -w table`；`etcdctl alarm list` |
| 大规模 Pod 删除/更新很慢 | apiserver/etcd 写入瓶颈、控制器 workqueue 积压 | 看 workqueue 深度、apiserver 请求延迟 |
| DNS 解析慢 | `ndots:5` + 外部域名、CoreDNS 副本不足 | `dig` 计时、CoreDNS metrics |

---

## 第 10 章 延伸知识（区分"会用"与"懂行"的分界线）

### 10.1 强隔离容器：gVisor / Kata / Firecracker

| 方案 | 原理 | 隔离强度 | 性能开销 | 适用 |
|---|---|---|---|---|
| **gVisor（runsc）** | 在用户态实现一个"内核"（Sentry），拦截容器所有 syscall，自己实现文件/网络 | 高（syscall 面在用户态） | 10%~30%（系统调用密集场景更高） | 多租户 Serverless、不可信代码 |
| **Kata Containers** | 每个 Pod/容器跑在**一个轻量 VM** 里（用 QEMU/Cloud-Hypervisor/Firecracker），内核是独立的 | 很高（硬件虚拟化边界） | 中（VM 启动 + 内存开销） | 强合规场景、公有云多租户 |
| **Firecracker** | 极简 VMM（只用 virtio，去掉了 BIOS/PCI），启动 ~125ms，内存 <5MiB | 很高 | 低 | AWS Lambda / Fargate 的底座 |
| **KrunVM / libkrun** | 类似，面向本地 | 高 | 低 | 桌面/开发环境 |

> 这正是"容器安全"的本质矛盾：**共享内核换性能**。要更强隔离，就必须把"内核"也隔离掉，代价是性能与复杂度。

### 10.2 eBPF：为什么它改变了容器技术

- **本质**：在内核里跑沙箱化的"用户态提交的程序"（BPF 字节码由 verifier 校验安全性），挂到内核钩子点上。
- **容器相关钩子**：
  - `kprobe/tracepoint`：观测（`bpftrace` 一行脚本查容器 syscall 延迟）
  - `cgroup_skb` / `cgroup_sock_addr`：在 cgroup 层做网络策略/重定向（Cilium 的核心）
  - `tc`（ingress/egress）：数据面处理、封装/解封装
  - `XDP`：驱动层丢包（DDoS 防护）
  - `LSM`：安全策略（替代/补充 Seccomp、AppArmor）
- **工程意义**：把"转发、策略、观测"从"iptables 规则 + 用户态进程"变成"内核内的一次函数调用" → 更高性能、更低延迟、更好可观测性（可以拿到"哪个 Pod 的哪个进程发起了什么连接"）。

### 10.3 镜像分发优化（大规模集群的命门）

| 方案 | 原理 | 收益 |
|---|---|---|
| 镜像预热 | 提前 pull 到所有节点 | 消除冷启动的拉取时间 |
| **P2P 分发（Dragonfly）** | 节点之间互相传块（BitTorrent 式），registry 只做种子 | registry 带宽压力降 90% |
| **按需加载（Nydus / eStargz / SOCI）** | 镜像转成可随机读的分块格式 + 内置索引；容器启动只拉"真正读到"的块 | 启动从分钟级到秒级 |
| 镜像瘦身 | 多阶段 + distroless | 传输量与攻击面双降 |
| 分层复用 | 统一 base image 规范 | 增量拉取几乎为零 |

### 10.4 在 K8s 之上：批处理与超大规模调度

- **Volcano**（华为开源）：面向 AI/HPC 的 gang 调度（"要么全部 8 个 Pod 一起调度成功，要么都不调度"）、队列与配额、binpacking。
- **YuniKorn**：多租户队列 + 公平调度。
- **Kueue**：K8s 官方社区的"作业队列"方案（1.30 前后活跃），把"配额与排队"从调度器里解耦出来。
- **Karpenter**（AWS）：**直接管理 Node** 而非只调度 Pod —— 它看到"Pending 的 Pod 规格"，直接创建最合适的 EC2 实例类型（比 Cluster Autoscaler 更细粒度、更快），并做整合（consolidation）。

### 10.5 多集群与集群治理

- **多集群动机**：故障隔离、地域合规、爆炸半径控制、团队边界。
- **方案**：Karmada（策略分发）、Cluster API（集群生命周期）、vCluster（虚拟集群，在一个物理集群里"套娃"出多个 apiserver）、ArgoCD/Flux（GitOps 跨集群部署）。
- **爆炸半径**：单集群上限（社区经验值：5000 节点 / 15 万 Pod 量级，受 etcd 与 apiserver 制约），超过就要拆集群。

### 10.6 常用"版本演进"清单（体现对生态的了解）

| 版本 | 关键变化 | 面试价值 |
|---|---|---|
| 1.24 | **移除 dockershim** | 必知 |
| 1.25 | 移除 PSP，引入 PSA GA | 必知 |
| 1.26 | CEL 准入（ValidatingAdmissionPolicy，无需 webhook 即可做校验）、`kubectl` 弃用 docker 相关 | 加分 |
| 1.27 | **in-place Pod resize（alpha）**、`podFailurePolicy`、`StatefulSet` 的 PVC 保留策略 | 加分 |
| 1.28 | **原生 sidecar 容器**（`restartPolicy: Always` 的 init container）、Job 的 `PodReplacementPolicy` | 加分 |
| 1.29 | **nftables 模式 kube-proxy（beta）**、`PodReadyToStartContainers` 条件 | 加分 |
| 1.30 | `AllowInsecureRegistry`、结构化调度参数、Contextual Logging | — |
| 1.31 | **Device Plugin 转向 DRA（动态资源分配）**、`Job` 的 `successPolicy` GA、`podReplacementPolicy` GA | 加分 |
| 1.32 | DRA 结构化参数、`kube-scheduler` 的异步 API | — |
| 1.33/1.34 | DRA 持续演进、sidecar 更成熟、in-place resize beta 推进 | — |

---

# 第二部分：国内大厂面试题库（分级 + 详解）

> **说明**：
> - 题目来自公开面经与同类岗位真题的归纳整理，按公司技术风格归类，不代表某次具体面试的原题。
> - 每题标注：**考核程度**（难度标识 + 出现频次）、**考核点**、**参考答案**、**延伸追问**。
> - 建议复习路径：先做 ⭐⭐ 打底 → 再做 ⭐⭐⭐ 攻深度 → 最后用 ⭐⭐⭐⭐ 练表达结构。

---

## 第 11 章 阿里巴巴（重底层、重规模、重场景设计）

### Q1. 容器和虚拟机的本质区别是什么？容器为什么能做到秒级启动？ ⭐⭐｜高频

**考核点**：是否理解"容器 = 进程隔离"而不是"轻量虚拟机"。

**解答**：
- 虚拟机通过 Hypervisor（KVM/Xen/ESXi）模拟硬件，运行**一套独立的内核与完整操作系统**。启动虚拟机需要引导内核、初始化 systemd、挂载根文件系统，耗时秒级到分钟。
- 容器**没有自己的内核**，它只是宿主机上的一组（或多个）普通进程，通过 namespace 改变"看到的资源视图"、通过 cgroup 限制"能用多少资源"、通过 pivot_root 换一个根目录。启动容器的本质是 `clone() + execve()`，即 **fork 一个进程并执行程序**，这就是毫秒级的原因。
- 因此隔离强度差异明显：容器的隔离边界是**内核数据结构**，任何内核漏洞都可能成为逃逸通道；VM 的隔离边界是 CPU 的虚拟化扩展（VT-x/AMD-V），逃逸需要攻破 Hypervisor。

**延伸追问**：
- "容器里 `uname -r` 看到的内核版本是谁的？" → **宿主的**。这是最直观的证明。
- "容器里能加载内核模块吗？" → 默认不能（缺 `CAP_SYS_MODULE` + seccomp 屏蔽 + 模块是全局的）。这进一步说明容器与宿主共享内核。
- "如果既要容器密度又要强隔离怎么办？" → gVisor / Kata / Firecracker（见 10.1）。

---

### Q2. Namespace 有哪几种？容器是如何被"关进"这些 namespace 的？ ⭐⭐⭐｜高频

**考核点**：8 种 namespace 的完整性与底层创建方式。

**解答**：
完整 8 种（按内核版本）：mnt(2.4.19)、uts(2.6.19)、ipc(2.6.19)、pid(2.6.24)、net(2.6.24)、user(3.8)、cgroup(4.6)、time(5.6)。

创建方式有两种：
1. **`clone(flags)`**：创建新进程的同时带 `CLONE_NEWPID|CLONE_NEWNET|...`，一次性把新进程放进新 namespace。runc 用的就是这个（实际上是 `clone()` 后接一系列配置，早期用 `clone` 加所有标志）。
2. **`unshare(flags)`**：让**当前已存在的进程**脱离原 namespace 进入新的。

关键细节：
- `CLONE_NEWPID` 有特殊性：**调用 `clone` 的进程自身仍留在旧 PID namespace，只有它 fork 出的子进程才进入新 namespace**。这就是为什么 `unshare --pid` 必须配 `--fork`。
- 内核里所有 namespace 都通过 `task_struct → nsproxy` 引用，`fork` 时若不带 `CLONE_NEW*` 则**共享**（引用计数 +1），带标志则 `copy` 出新实例。

**延伸追问**：
- "哪个 namespace 容器默认**不**隔离？" → **user namespace**（默认 docker 不启用 user ns，所以容器内 root 就是宿主 root，除非配置 `userns-remap` / rootless）。
- "K8s Pod 内的容器共享哪些 namespace？" → net、ipc、uts 共享（由 pause 持有）；pid 默认不共享（`shareProcessNamespace` 可开）；mnt 不共享（每个容器自己的 rootfs，通过 volume 共享目录）。

---

### Q3. 为什么容器里的 PID 1 很特殊？生产上会踩什么坑？ ⭐⭐⭐｜高频

**考核点**：信号与僵尸进程的真实理解。

**解答**：
两个内核级特殊性：
1. **信号默认行为被屏蔽**：内核对 PID 1 的信号处理有特殊保护 —— 如果 PID 1 没有注册对应 handler，则**不发**默认动作（SIGTERM/SIGINT 会被忽略，只有 SIGKILL 能杀）。目的（在宿主上）是防止误杀 init。
   → 后果：`docker stop` 发 SIGTERM 给 PID 1，若应用没处理，**必须等满 `--time`（默认 10s）后才 SIGKILL**。表现为"停止容器要 10 秒"。
2. **孤儿进程收养者**：容器内的子进程父进程死掉后，会被 reparent 到 PID 1；PID 1 必须 `wait()` 回收，否则变成僵尸（zombie），耗尽 PID/进程表。

**踩坑场景**：
- `ENTRYPOINT sh -c "java -jar app.jar"` → PID 1 是 `sh`，`sh` 收到 SIGTERM 后不会转发给 java，java 被 SIGKILL 强杀 → 请求中断、事务未提交。
- 修复：① 用 exec 形式 `ENTRYPOINT ["java","-jar","app.jar"]`；② `docker run --init`（注入 tini）；③ K8s 里用 `tini` 作为 entrypoint 的 wrapper；④ 应用自身注册 SIGTERM handler。

**延伸追问**：
- "僵尸进程在容器里的表现？" → `ps` 看到 `<defunct>`，父进程是 PID 1；如果 PID 1 不 wait，会累积。
- "为什么 `kubectl exec` 进来的进程不是 PID 1？" → exec 是在容器已有 namespace 内**新建**一个进程，PID 是递增的；它继承容器的 cgroup，所以会消耗容器的资源配额。

---

### Q4. cgroup v1 和 v2 的核心区别？为什么 K8s 在 v2 下必须用 systemd 作为 cgroup driver？ ⭐⭐⭐⭐｜中高频

**考核点**：是否真的理解 cgroup 层级模型与 K8s 的耦合。

**解答**：
- **v1**：每个控制器（cpu、memory、blkio、pids…）各自一棵独立的树，同一进程可以在 cpu 树属于 A 组、在 memory 树属于 B 组 → 归属语义混乱、无法原子迁移；`freezer`、`devices`、`net_cls` 这些"奇怪"的控制器各自实现。
- **v2**：**单一统一层级**（一个挂载点 `/sys/fs/cgroup`），控制器按需通过 `cgroup.subtree_control` 下放；进程归属唯一；引入 `cgroup.stat`、`cgroup.events`、**PSI（pressure）**、`memory.high` 软限、`io.latency` 等统一能力。核心约束是 **"no internal process"**：向子节点下放了控制器的 cgroup 自己不能再挂进程。
- **为什么必须 systemd**：
  1. v2 采用**单一层级 + 进程唯一归属**，而 systemd 本身就是这个层级的**管理者**（`system.slice`、`user.slice`、`.scope`、`.slice`）。如果 kubelet 用 `cgroupfs` driver 直接往 `/sys/fs/cgroup/...` 里创建目录，就会与 systemd 创建的 slice 冲突，出现**两套视图打架**：systemd 认为某个 scope 没有进程、kubelet 却往里塞；节点重启后 systemd 会清理掉 kubelet 建的 cgroup → 资源限制丢失甚至节点异常。
  2. v1 时代之所以允许 `cgroupfs`，是因为 v1 多层级本就松散，冲突面小。
  3. 在 v2 上，**只有 systemd 知道"谁是真正的父"**（它持有整个树的 `cgroup.procs`），因此 runtime（runc/containerd）与 kubelet 都必须走 systemd 的接口（`systemd-run --scope` / D-Bus），由 systemd 创建 `<slice>/<scope>`。
- 生产事故表现：`--cgroup-driver` 不统一时，`docker info` 与 kubelet 配置不一致 → **容器资源限制不生效**、`failed to create cgroup` 报错、节点 NotReady。

**延伸追问**：
- "v2 下 kubelet 的 cgroup 路径长什么样？" → systemd driver：`kubepods-burstable-pod<UID>.slice`，Pod 内容器 `cri-containerd-<id>.scope`。
- "PSI 有什么用？" → `memory.pressure` 的 `some avg10/avg60/avg300` 能直接告诉你"因内存不足导致任务停滞的时间比例"，比看使用率更接近真实体验。

---

### Q5. 容器设置了 `memory limit` 后，进程多久会被 OOM？为什么有时"内存没满"也被杀了？ ⭐⭐⭐⭐｜高频

**考核点**：内存限制、page cache、working set、OOM 触发条件的真实链路。

**解答**：
触发链路：
```
进程申请内存 → 内核分配（匿名页 anon）
   → cgroup memory.current 上升
   → 达到 memory.max
   → 内核先尝试"直接回收"（direct reclaim）：丢弃该 cgroup 的 file page cache
   → 回收后仍不足（anon 页无法回收，除非有 swap）
   → 触发 cgroup 级 OOM killer
   → 选 oom_score_adj 最高的进程 → SIGKILL → 容器 exit code 137
```

"内存没满却被杀"的三种真实原因：
1. **`memory.current` 包含 page cache**。业务看到的"RSS 300MB"可能实际 `current` 已经 900MB（读了大量文件缓存）。→ 排查要看 `memory.stat` 的 `anon` / `file` / `slab`，以及 `container_memory_working_set_bytes`。
2. **内核内存（slab、socket buffer、页表）也计入 cgroup**。大量短连接、大量文件句柄会让 `slab` 暴涨 → "业务没占内存但 cgroup 满了"。
3. **瞬时峰值**：Java/Go 的 GC 或大对象分配会出现瞬时高水位，即使稳态不高也会撞上限被杀。
4. **`memory.max` 之外还受 `memory.swap.max` 影响**；同时**父 cgroup**（如 `kubepods.slice`）也有限制，Pod 之间会互相挤压。

**延伸追问**：
- "为什么 K8s 驱逐用 `memory.available` 而不用使用率？" → `memory.available = capacity - working_set`，`working_set` 排除了可回收的 inactive file cache。用使用率会导致"缓存多就被误判为内存不足"。
- "Java 容器里怎么设堆？" → 用 `-XX:MaxRAMPercentage=75` 而非固定 `-Xmx`，让其读 cgroup 限制自动计算（老版本 JDK 读不到 cgroup v2 限制，需要 `-XX:+UseContainerSupport` + 升级，这是常见事故）。
- "OOM 和驱逐的区别？" → OOM 是内核行为（无排名、无法优雅、直接 SIGKILL）；驱逐是 kubelet 行为（有 QoS 排名、发 SIGTERM、等 grace period）。

---

### Q6. OverlayFS 是怎么工作的？为什么数据库绝不应该把数据写在容器层？ ⭐⭐⭐｜高频

**考核点**：CoW、copy-up、whiteout、性能与数据安全。

**解答**：
- 结构：`lowerdir`（多个只读镜像层，自下而上）+ `upperdir`（可写容器层）+ `workdir`（内部用）+ `merged`（合并视图）。
- **读**：自上而下逐层查找，命中第一个即返回（所以上层的同名文件会遮蔽下层）。
- **写（copy-up）**：要修改的文件若在 lower 层，内核先**把整个文件复制到 upperdir**，再在副本上修改。→ 修改 10GB 数据文件里的一个字节，也要复制 10GB（虽然 overlay 有 `metacopy` 优化，但默认行为如此）。
- **删（whiteout）**：在 upperdir 创建 `.wh.<name>` 字符设备标记，让下层同名文件"不可见"。→ 不释放镜像层空间。
- **rename 目录的坑**：早期 overlay 不支持跨层 rename 目录（需要 `redirect_dir`），`mv` 会整树 copy-up，表现为"容器内 mv 一个目录卡死几分钟"。

**为什么数据库不能写容器层**：
1. **copy-up 写放大**：数据库文件通常很大且随机写 → 每次触碰大文件都触发整文件复制，IO 放大到不可接受。
2. **数据不持久**：容器删除 = upperdir 删除 = 数据丢失。
3. **overlay2 上的 IO 语义与 `fsync` 行为**与裸文件系统不同，数据库（尤其 MySQL/PostgreSQL）对 fsync/rename 语义有强假设 → 可能出现**数据损坏**。
4. **多实例共享**：overlay 是每个容器一份 upperdir，无法在副本间共享。
→ 正确做法：数据库数据目录必须挂 **volume（本地 SSD/NVMe）或 PVC**，且用宿主的本地文件系统（xfs/ext4）直通。

**延伸追问**：
- "`docker commit` 做了什么？" → 把 upperdir 打包成一个新 layer。
- "镜像层数太多有什么影响？" → lookup 要逐层遍历（层数越多越慢）、`lowerdir` 参数有长度上限（overlay2 用 `/var/lib/docker/overlay2/l/` 的短链接就是为了规避路径长度超限）。
- "overlay2 vs devicemapper？" → devicemapper 是块级快照（thin pool），有 COW 但性能与稳定性较差，已在 Docker 中弃用；overlay2 是文件级，性能更好且实现简单。

---

### Q7. 阿里云/大厂场景：如何设计一个能支撑 5000 节点、15 万 Pod 的 K8s 集群？ ⭐⭐⭐⭐⭐｜终面级

**考核点**：体系化设计能力、瓶颈识别、取舍表达。

**解答框架（按"瓶颈 → 手段"分层回答）**：

**1) etcd 层（第一瓶颈）**
- 3 或 5 节点独立集群，NVMe SSD 本地盘，独占机器（不与业务混部），关闭 swap，`--quota-backend-bytes=8G`，定期 defrag + compact，监控 `etcd_disk_wal_fsync_duration_seconds`（P99 应 < 25ms）。
- 把**高频变化的数据**（Event、Lease）与大对象（ConfigMap）分离，减小 etcd 压力。→ 大规模集群通常**禁用或大幅缩短 Event TTL、关闭不需要的 watch**。

**2) apiserver 层**
- 无状态多副本（≥3）+ 前置 LB；水平扩展靠加副本。
- 打开 **watch cache** 并调整 `--watch-cache-sizes`；用 **`--max-requests-inflight` / `--max-mutating-requests-inflight`** 做优先级限流（1.19+ APF，`--enable-priority-and-fairness`）。
- **用聚合/分层**：把"读多写少"的请求（list）打到大量副本；写操作天然串行于 etcd。

**3) kubelet 层**
- **Node Lease 心跳**（已默认），避免 NodeStatus 高频写。
- 降低 PLEG/status 上报频率（`--node-status-update-frequency`、housekeeping 周期）。
- **限制每节点容器数量**（`--max-pods`，官方建议 ≤110；大规模场景常调到 250 但要注意 docker/containerd 的 fd 与内存开销）。

**4) 网络层**
- **放弃 iptables 模式**：用 **ipvs**（或 Cilium eBPF）→ 避免万级 Service 下的规则爆炸。
- **EndpointSlice** 已默认；限制单个 Service 的端点规模，或做分片。
- CNI 选 Calico BGP（无封装、性能好）+ **Typha**（否则 Felix 的 watch 会压垮 apiserver）。

**5) 调度层**
- scheduler 多副本但只有一个 leader 生效（吞吐瓶颈在单实例）→ 用 **调度框架的 `PercentageOfNodesToScore`**（默认按节点数动态取 5%~50%）控制打分节点数量；对超大集群设固定值减少评分开销。
- 大量小 Pod 用 **binpacking 策略**（`MostAllocated`）提高密度；或用 `topologySpreadConstraints` 保高可用。
- 引入 **Volcano/Kueue** 做队列与配额，避免"业务方无限抢占"。

**6) 镜像与启动**
- **镜像预热 + P2P 分发（Dragonfly）+ 按需加载（Nydus/eStargz）**——大规模集群的"批量发布"往往死在镜像拉取上（几千 Pod 同时拉同一个 1GB 镜像会把 registry 打挂）。
- 控制**单节点并发拉取数**（containerd 的 `max_concurrent_downloads`）。

**7) 监控与爆炸半径**
- 监控数据量随节点线性增长 → Prometheus 联邦/分片（Thanos、Mimir），**采样 + 降精度**。
- 明确**单个集群上限**，用多集群（Karmada/vCluster）切分业务域；**控制一次变更影响的节点比例**（分批灰度、PDB、`maxUnavailable`）。

**答题技巧**：先给"一条链路"（用户请求 → apiserver → etcd），指出每一跳的瓶颈与对应手段，最后用一句话总结取舍：**"K8s 的规模上限不是某一个组件决定的，而是 etcd 的写放大、apiserver 的扇出、调度的单 leader、镜像分发的带宽四者共同决定。"**

**延伸追问**：
- "为什么要关闭 Event 或缩短 TTL？" → Event 产生量极大（每个失败重试都有事件），是 etcd 空间的隐形杀手，且大部分没有长期价值。
- "`PercentageOfNodesToScore` 设小会怎样？" → 打分只看部分节点 → 调度更快但质量下降（可能错过"最合适"节点）。

---

## 第 12 章 腾讯（重链路、重排障、重线上经验）

### Q8. 一个 Pod 从 `kubectl apply` 到真正跑起来，完整经过了哪些步骤？ ⭐⭐⭐⭐｜超高频（几乎所有公司都会问）

**考核点**：全链路掌握度。这题答得好可以直接定级。

**解答（八步法）**：

```
① kubectl 客户端：把 YAML 序列化 → 发 POST/PATCH 到 apiserver
   （kubectl 会做 OpenAPI 校验、计算 patch（apply 时走 server-side apply））

② apiserver：认证 → 授权（RBAC）→ Mutating 准入（注入默认值/SA/sidecar）
   → Validating 准入（PSA、自定义 webhook）→ 写入 etcd（带 resourceVersion）
   → 返回 201，并扇出 watch 事件

③ controller-manager（以 Deployment 为例）：
   informer 收到 Deployment 变更 → workqueue → 计算期望 RS 数量
   → 创建 ReplicaSet 对象 → ReplicaSet controller 创建 Pod 对象
   （此时 Pod 里没有 nodeName，处于 Pending）

④ scheduler：
   把 Pod 放进 activeQ → Filter 筛选可行节点（资源/亲和/污点/PVC 拓扑）
   → Score 打分 → 选最高 → **在 Pod.spec 里写 nodeName（Binding）**
   → 更新 apiserver

⑤ kubelet（目标节点）：
   watch 到自己节点上"有 nodeName 的 Pod" → 放入 syncLoop
   → syncPod：
      a. 计算 sandbox → CRI RunPodSandbox
         → runtime 创建 pause 容器，**调用 CNI 插件 ADD**（创建 veth、配 IP、配路由）
      b. 拉取镜像（CRI PullImage，并发受限）
      c. 顺序执行 init 容器（每个成功后继续下一个）
      d. 创建并启动业务容器（CreateContainer → StartContainer）
      e. 启动容器内的探针（startup 成功后启用 liveness/readiness）
   → statusManager 上报 Pod status 到 apiserver

⑥ 网络就绪与端点：
   EndpointController 发现 Pod Ready（readiness 通过）
   → 写入 EndpointSlice → kube-proxy watch → 更新 iptables/ipvs 规则
   → Service 开始把流量打到这个 Pod

⑦ 卷挂载（与 5 并行/前置）：
   kubelet 的 volumeManager / CSI：Attach → NodeStage → NodePublish（bind mount）

⑧ 就绪完成：Pod Conditions 全部 True，Pod phase=Running，Ready=1/1
```

**关键时序细节（能说出这些就是加分）**：
- **控制器不是"串行传递"**：每个组件都只是"独立地对 API 对象变化作出反应"，没有谁"调用"谁。所以链路里任何一环延迟，都只表现为整体变慢，不会报错。
- **CNI 在 sandbox 创建阶段执行，早于业务容器** → 所以"网络不通"时业务容器根本不会启动（卡在 ContainerCreating）。
- **`Pending` 最久可能停在"调度"或"镜像拉取"两个地方**，靠 `describe` 的 Events 区分。

**延伸追问**：
- "哪一步最容易慢？" → 镜像拉取（无本地缓存时）、卷挂载（CSI 后端慢）、调度（资源碎片化导致的重试）。
- "如何测量端到端启动耗时？" → `kubelet_pod_start_sli_duration_seconds`、`scheduler_scheduling_duration_seconds`、`apiserver_request_duration_seconds` 三个指标拼起来。

---

### Q9. 容器逃逸有哪些方式？你如何排查一台机器是否已被逃逸？ ⭐⭐⭐⭐｜中高频（安全岗必问）

**考核点**：安全思维 + 实操能力。

**解答**：
**逃逸路径**（按利用难度排序）：
1. **配置类（最容易，占绝大多数）**：
   - 挂载了 `docker.sock` → 容器内 `docker run --privileged -v /:/host` 直接拿下宿主。
   - `privileged: true` → 全部 capabilities + 放开设备 + 关闭 seccomp/AppArmor → 直接 `nsenter` 到宿主 pid ns。
   - `hostPath: /` 或挂载 `/etc`、`/root`、`/var/run` → 改宿主文件、写 cron。
   - `hostPID: true` / `hostNetwork: true` + 特权 → `nsenter -t 1 -a`。
2. **内核特性滥用**：
   - `CAP_SYS_ADMIN` + `cgroup v1` → 写 `release_agent` 在宿主执行任意命令（CVE-2022-0492 类手法的通用版）。
   - `CAP_DAC_READ_SEARCH` → `open_by_handle_at` 读任意文件。
   - 未 masked 的 `/proc/sys` 可写 → 改 `core_pattern`，让宿主 coredump 执行你的程序。
3. **运行时/内核漏洞**：
   - CVE-2019-5736（runc 被覆写）、CVE-2024-21626（runc `/proc/self/fd` 泄漏）、Dirty Pipe、Dirty COW、`io_uring` 系列漏洞。

**排查动作（可操作清单）**：
```bash
# 1) 找可疑的容器配置
docker ps -q | xargs docker inspect --format '{{.Name}} privileged={{.HostConfig.Privileged}} pid={{.HostConfig.PidMode}} net={{.HostConfig.NetworkMode}} mounts={{.Mounts}}'
# 关注：privileged=true / pid=host / 挂载 docker.sock 或 / 或 /etc

# 2) 检查是否有异常进程从容器内"跑出来"（父进程是 shim 但 namespace 与宿主一致）
for p in /proc/[0-9]*; do
  ns=$(readlink $p/ns/mnt); [ "$ns" = "$(readlink /proc/1/ns/mnt)" ] && echo "$p 与宿主同 mnt ns"
done

# 3) 检查宿主关键文件是否被篡改
stat /etc/passwd /etc/crontab /root/.ssh/authorized_keys
rpm -Va / dpkg -V  # 校验系统文件完整性

# 4) 检查网络异常外联（挖矿/反弹 shell）
ss -antp | grep -v ESTABLISHED | head
iptables -t nat -L -n | grep -v DOCKER   # 可疑 DNAT/重定向

# 5) 检查内核模块是否被加载
lsmod | head; dmesg | tail -50    # 关注未知模块、oops

# 6) 检查 cgroup release_agent 是否被写过
cat /sys/fs/cgroup/*/release_agent 2>/dev/null
```

**防御总结（回答这题的后半段很重要）**：
- 禁用 `privileged`、禁止挂 `docker.sock`、`hostPath` 白名单；
- `runAsNonRoot` + `readOnlyRootFilesystem` + `capabilities.drop: [ALL]` + `seccompProfile: RuntimeDefault`；
- 用 **PSA restricted** 或 Kyverno/OPA 做准入拦截；
- 内核与 runtime 及时打补丁（这是最有效的一条）；
- 用 eBPF（Tetragon/Falco）做**行为级检测**（如"容器内进程 execve 了 nsenter"、"容器内进程访问了 /proc/1/root"）。

---

### Q10. 容器/Docker 的端口映射是怎么实现的？为什么有时候要 `userland-proxy: false`？ ⭐⭐⭐｜中频

**考核点**：netfilter 与 DNAT 的真实数据流。

**解答**：
两条路径同时存在：
1. **iptables DNAT**（内核转发）：`nat` 表的 `DOCKER` 链里有一条 `-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80`。外部包命中后目标地址被改写，经 FORWARD 链到容器；回包由 conntrack 反向转换。
2. **docker-proxy**（用户态）：有一个进程监听宿主 8080，收到连接后自己再向 `172.17.0.2:80` 建连并转发数据。

**为什么还需要 docker-proxy**：
- **回环访问**：从宿主机 `curl 127.0.0.1:8080` 在某些内核/配置下 DNAT 不生效（hairpin NAT 需要额外规则），用户态代理能兜住。
- **兼容性**：不支持 NAT 的环境、某些云主机特殊网络栈。
- **缺点**：多一次用户态数据拷贝，高并发连接时 CPU 与延迟明显上升 → 生产上常设 `"userland-proxy": false`（前提是确认回环访问不需要）。

**排查"端口映射不通"的顺序**：
```
1. 容器内服务是否真的监听 0.0.0.0（而非 127.0.0.1）→ netstat -lntp
2. iptables -t nat -L DOCKER -n -v        → 规则存在且计数增长
3. iptables -t filter -L DOCKER-USER -n -v → 是否被自定义规则 DROP
4. iptables -t filter -L FORWARD -n -v     → firewalld 默认 DROP FORWARD 是经典坑
5. sysctl net.ipv4.ip_forward               → 必须为 1
6. 云安全组/主机防火墙
```

---

### Q11. Pod 里 `preStop` 和 `SIGTERM` 是什么关系？为什么加了 preStop 还是丢请求？ ⭐⭐⭐⭐｜高频（真正区分"用过"和"理解"）

**考核点**：优雅退出的时序。

**解答**：
Pod 删除时的真实时序（**注意这些动作是并发的**）：
```
t0: 用户/控制器发起删除 → Pod 进入 Terminating
    ├─(并发 A) EndpointSlice 更新，把这个 Pod 标记摘除 → kube-proxy 开始更新内核规则（**有传播延迟**）
    └─(并发 B) kubelet 开始终止容器：
         1) 先执行 preStop hook（阻塞，直到完成或超时）
         2) preStop 完成后发 SIGTERM 给容器 PID 1
         3) 等待 terminationGracePeriodSeconds（从 t0 开始计时！不是从 SIGTERM 开始）
         4) 超时 → SIGKILL
```

**丢请求的三个真实原因**：
1. **preStop 里没 sleep**：A 与 B 并发，kube-proxy 规则更新需要时间（尤其是大集群有几秒延迟），而 SIGTERM 立即到达 → 应用立刻停止接收新连接，但仍有流量打进来 → 丢。
   **正确做法**：`preStop: exec: command: ["sh","-c","sleep 5"]`，用 5 秒等 Endpoint 摘除传播完成。
2. **应用收到 SIGTERM 后立刻退出**，没有等待在途请求完成。→ 应用要实现"收到信号 → 停止 accept → 等待 in-flight 完成 → 退出"。
3. **`terminationGracePeriodSeconds` 被 preStop 吃掉了**：这个计时**从 t0 开始**，如果 preStop 睡了 30s 而 grace 只有 30s，SIGTERM 后没有时间留给应用优雅退出 → 直接被 SIGKILL。→ preStop 的 sleep 要计入 grace 时长。
4. **长连接（HTTP/2、gRPC、WebSocket）**：kube-proxy 摘除后已建立的连接不会被切断，客户端仍复用连接 → 需要在应用层做连接优雅关闭（发 GOAWAY）。

**延伸追问**：
- "`kubectl delete pod` 与 `delete --grace-period=0 --force` 的区别？" → 后者直接删 etcd 记录，**容器可能还在跑**（因为 kubelet 没收到优雅终止）→ 会产生"幽灵容器"，是经典的资源泄漏来源。
- "StatefulSet 缩容为什么是倒序？" → 保证集群成员的"有序退出"，避免同时失去多数派（如 etcd 集群）。

---

### Q12. etcd 出现 `NOSPACE` 告警或写入变慢，你怎么处理？ ⭐⭐⭐⭐｜中高频（腾讯/字节很爱问）

**考核点**：etcd 运维细节与故障判断。

**解答**：
**现象分层**：
- `NOSPACE`：`etcdctl alarm list` 显示 `NOSPACE`，集群**拒绝所有写入** → 表现为"kubectl apply 全部失败，但 get 正常"。根因是 db 文件超过 `--quota-backend-bytes`（默认 2GB）。
- 写入慢：`etcd_disk_wal_fsync_duration_seconds` P99 高（>25ms 就危险）、`etcd_server_leader_changes_seen_total` 频繁、`etcd_network_peer_round_trip_time` 抖动。

**处置步骤**：
```bash
# 1) 解除告警 + 释放空间
etcdctl alarm list
etcdctl alarm disarm
etcdctl compact $(etcdctl endpoint status -w json | jq '.[0].Status.header.revision')
etcdctl defrag --cluster          # 回收物理空间（**会阻塞该节点**，逐个做）
etcdctl endpoint status --cluster -w table

# 2) 长短期治理
· 提高 quota（--quota-backend-bytes=8G），但要确认磁盘余量
· 降低写入量：缩短 Event TTL、减少高频写入的 CRD、禁用不必要的 watch
· 排查大对象：ConfigMap/Secret 体积过大（单对象上限 1MiB 的限制来自 etcd 的请求大小）
· 硬件：换 NVMe SSD、独占机器、关闭 swap、避免与其他 IO 密集进程混部
· 网络：etcd 节点同机房/低延迟；peer 端口不要跨公网
```

**必须能说清的机制**：
- etcd **不会自动释放空间**：删除 key 只是删逻辑版本，`bbolt` 文件不缩小 → 必须 compact（清历史版本）+ defrag（整理物理页）。
- **defrag 是阻塞操作**，期间该节点不服务 → 必须逐节点执行，且要保证多数派存活。
- **etcd 的写是两阶段**：先写 WAL（fsync），再更新内存索引（B+ 树），后台再持久化。所以**磁盘 fsync 延迟直接决定写延迟** → 网络存储（EBS/NFS）是致命选择。

---

### Q13. Docker 与 Kubernetes 的关系演进：为什么 K8s 1.24 移除了 dockershim？移除后还能用 Docker 吗？ ⭐⭐⭐⭐｜高频

**考核点**：是否真正理解 CRI/OCI 的分层。

**解答**：
- 历史的耦合：早期 kubelet 只会说 Docker 的 API，所以 K8s 内置了一个 **dockershim**（把 CRI 请求翻译成 Docker API），再由 dockerd 调 containerd 调 runc。**调用链长了两跳**：kubelet → dockershim → dockerd → containerd → runc。
- **CRI 的出现**（1.5 起）：把 kubelet 与运行时解耦，定义 `RuntimeService`（RunPodSandbox/CreateContainer/StartContainer…）和 `ImageService`（PullImage/ListImages…）两个 gRPC 接口。containerd 和 CRI-O 原生实现 CRI，**不需要 shim**。
- **移除原因**：
  1. **维护成本**：dockershim 是 K8s 代码里"为兼容 Docker 而写的适配层"，要跟着 Docker API 变化维护，且是 K8s 仓库里最不受欢迎的代码之一。
  2. **多了一层无用开销与语义差异**：Docker 的容器模型（单个容器）与 Pod（sandbox + 多容器）不完全对齐，导致 kubelet 要做额外适配（如"为什么 docker 里看不到 pause 容器"这类问题）。
  3. **Docker 本身已把运行时捐给 containerd**：实际干活的是 containerd，dockerd 更多是"用户体验层"（build、network、volume、CLI）。
- **移除后还能用 Docker 吗**：
  - **镜像完全兼容**：Docker 产出的镜像就是 OCI 镜像，containerd/CRI-O 都能跑。
  - **`docker build` 照常可用**：CI 里用 Docker 构建，集群里用 containerd 运行 —— 这是现在的标准组合。
  - **`docker` CLI 不能再用来看集群里的容器**（要用 `crictl`）；节点上也不再有 dockerd。
  - 想继续用 Docker 作为 runtime 的集群，需要装 **cri-dockerd**（Mirantis 维护的独立适配器），但已不推荐。

**延伸追问**：
- "`crictl` 和 `docker` 命令的对应关系？" → `crictl ps` ≈ `docker ps`；`crictl images` ≈ `docker images`；`crictl logs` 略有不同；**`crictl` 不能 build/exec 交互式 shell 之外的 docker 特性**。
- "为什么 `docker ps` 看不到 pause 容器，但 `crictl ps` 能看到 sandbox？" → 因为 Docker 把 pause 藏起来了（内部实现），containerd 把 sandbox 作为一等对象。

---

## 第 13 章 字节跳动（重性能、重大规模、重工程细节）

### Q14. 大规模集群里 Pod 启动很慢，你会怎么定位和优化？ ⭐⭐⭐⭐｜高频

**考核点**：能从"端到端耗时"拆解到具体环节。

**解答**：
**先量化**：把启动耗时拆成四段，逐段测：
```
t1 调度延迟   = Pod 创建 → 绑定 nodeName      （指标：scheduler_scheduling_duration_seconds）
t2 镜像就绪   = 绑定 → 镜像拉取完成            （日志：kubelet "PullImage" 耗时；containerd 日志）
t3 沙箱/网络  = 拉镜像后 → sandbox Ready       （CNI 插件耗时、IPAM）
t4 应用启动   = 容器启动 → readiness 通过      （应用日志 + 探针周期）
```

**各段常见原因与手段**：

| 阶段 | 根因 | 手段 |
|---|---|---|
| t1 调度 | 节点资源碎片化（大 Pod 无处可放）、打分节点过多、Pod 数量大导致队列积压 | 用 binpacking；调 `PercentageOfNodesToScore`；用 gang 调度（Volcano）；给大 Pod 单独节点池 |
| t2 镜像 | 镜像大（多阶段没做）、registry 慢/被墙、并发拉取打满带宽、无本地缓存、解压慢（overlay 层多） | 镜像瘦身 + distroless；P2P 分发（Dragonfly）；按需加载（Nydus/eStargz）；预热；`max_concurrent_downloads`；用 SSD |
| t3 网络 | CNI 插件串行、IPAM 分配慢（大网段位图扫描）、Calico Typha 未部署导致 Felix 卡 | 换高性能 CNI（Cilium）；IPAM 优化（block 分配）；加 Typha |
| t4 应用 | 探针 `initialDelaySeconds` 太大、`startupProbe` 没配、JVM 启动慢、依赖外部服务超时 | 用 startupProbe；调优应用启动；懒加载依赖；预热 JIT |

**必须提到的"隐形杀手"**：
- **`kubelet_pod_start_sli_duration_seconds`** 是官方的端到端 SLI；
- **PLEG 不健康**会让所有 Pod 操作排队（表现为"节点上所有 Pod 都慢"）；
- **节点 CPU/磁盘饱和**时，容器创建被节流（cgroup 限流会拖慢一切）；
- **`--max-pods` 过大会让 kubelet 的 status 上报与 PLEG 变慢**（每增加 100 个 Pod，kubelet 的 CPU 与内存占用明显上升）。

**延伸追问**：'为什么"批量发布 500 个 Pod"时特别慢？' → 因为镜像同时拉取会打满 node 的网络与磁盘 IO，且 registry 成为瓶颈。解决要靠 **P2P + 分批 + 预热**，而不是调大并发。

---

### Q15. iptables 和 ipvs 模式的 kube-proxy 有什么区别？什么时候必须换 ipvs？ ⭐⭐⭐⭐｜高频

**考核点**：netfilter 与 LVS 的机制差异、规模拐点。

**解答**：
- **iptables**：
  - 每个 Service 生成 `KUBE-SVC-xxx` 链，每个 Endpoint 生成 `KUBE-SEP-xxx` 链，用 `statistic mode random` 做随机负载均衡。
  - 包处理时**线性遍历规则链** → 匹配成本随规则数增长（O(n)）。
  - 每次 Endpoint 变更要**重写大量规则**（虽然 kube-proxy 做了全量重建 + 原子替换，但 CPU 开销大）。
  - conntrack 表压力大 → `nf_conntrack: table full` 是经典故障。
- **ipvs**：
  - 用内核 `ip_vs` 模块的**哈希表**（Virtual Server → Real Servers），查找 O(1)。
  - 支持多种调度算法（rr/wrr/lc/sh），比 iptables 的"随机"更可控。
  - 规则更新是增删哈希项，代价小。
  - 仍需少量 iptables 规则（SNAT、NodePort 的马甲规则），所以排查要两边都看。

**何时必须换**：
- Service 数量 > ~1000 或 Endpoint 总量 > ~10000；
- 高频变更（频繁扩缩容）导致 kube-proxy CPU 打满；
- 出现 conntrack 相关丢包；
- 需要更精确的负载均衡（最小连接）。

**注意点（容易被追问）**：
- ipvs 模式下 **Pod 访问自身 Service（hairpin）**、**NodePort 从本机访问**等场景需要额外的 `masquerade` 规则；`kube-proxy --ipvs-strict-arp` 等参数在容器化网络下必须开。
- 现在的新选择是 **nftables 模式**（1.29 beta）与 **Cilium eBPF（完全绕过 netfilter）**。

---

### Q16. `docker build` 的缓存是怎么算的？怎么把镜像从 2GB 压到 100MB？ ⭐⭐⭐｜中频

**考核点**：工程实践能力。

**解答**：
- 缓存 key = **父层 digest + 指令内容（+ COPY 的文件内容 checksum）**。命中即复用；一旦某层未命中，其后所有层全部失效。
- 因此顺序很重要：**变化频率低的放前面，变化频繁的放后面**。典型的"先 copy 依赖清单 → 装依赖 → 再 copy 源码"。
- `ARG` 的值参与 key → 用 `--build-arg` 传时间戳/commit 会让缓存全失效（要避免）。
- `RUN --mount=type=cache`（BuildKit）可以让包管理器缓存**不落进镜像层**，既保留缓存又减小体积。

**瘦身五步**：
1. **多阶段构建**：builder 装编译工具，运行镜像只 `COPY --from=builder` 产物 → 常见从 1.5GB 降到 50MB。
2. **换基础镜像**：`ubuntu`(77MB) → `alpine`(5MB) → `distroless/static`(2MB) / `scratch`。
3. **合并 RUN + 清理缓存**：`apt-get update && apt-get install -y x && rm -rf /var/lib/apt/lists/*` 必须在同一层。
4. **去掉不必要文件**：`.git`、测试、文档、`node_modules` 的开发依赖（用 `npm ci --omit=dev`）、编译中间产物。
5. **静态编译**：Go 用 `CGO_ENABLED=0 -ldflags="-s -w"` → 单文件二进制，可放 scratch。

**延伸**："`docker history` 看某层 800MB 但不知道是什么？" → 用 `dive` 交互式逐层查看文件增删；常见元凶是 apt 缓存、日志、构建期下载的临时包。

---

### Q17. 什么是 eBPF？Cilium 为什么能替代 kube-proxy 并且更快？ ⭐⭐⭐⭐｜进阶加分

**考核点**：对新数据面的理解。

**解答**：
- **eBPF**：内核内运行的沙箱化程序，挂在 kprobe/tracepoint/tc/XDP/cgroup/LSM 等钩子上；用 map 与用户态交换数据；有 verifier 保证不会死循环/越界。
- **Cilium 替代 kube-proxy 的机制**：
  1. 在 **socket 层**（`cgroup/connect4`、`cgroup/sendmsg4` 等钩子）就拦截连接建立，把目标 **Service ClusterIP 直接改写为后端 Pod IP** → 包**不经过 netfilter**，不需要 DNAT 规则、不需要 conntrack 查表。
  2. 对跨节点的包用 **tc（ingress/egress）** 做转发与封装；对 LB 场景可用 eBPF 直接做四层负载均衡（含 Maglev 一致性哈希）。
  3. 用 **eBPF map** 存 service→backend 映射与策略，查找 O(1)，且更新是"map 内更新"而非"重写规则链"。
- **为什么快**：省掉了 netfilter 的规则遍历、conntrack 的状态维护、以及一次 DNAT 后的路由再查找；同时避免了 iptables 的"规则数线性增长"。
- **代价**：内核版本要求高（建议 ≥5.4）、排障方式不同（`cilium monitor`/`hubble`）、某些依赖 iptables 的老应用/工具会不适应、学习曲线陡。

**延伸追问**：'Cilium 的 NetworkPolicy 为什么基于 identity？' → 它给每个 Pod 的 label 集合算一个 identity（数字），策略是"identity → identity"的规则，用 eBPF map 快速匹配。**好处**：IP 变了策略依旧生效、规则数不随 Pod 数线性增长、支持 L7 策略（HTTP 方法/路径）。

---

### Q18. 你如何设计一套"大规模滚动发布"的方案？要考虑哪些风险？ ⭐⭐⭐⭐⭐｜终面级

**考核点**：工程体系设计，不只是 K8s 参数。

**解答框架**：
**1) 发布单元与节奏**
- 用 `maxSurge`/`maxUnavailable` 控制并发度；大集群要配合**分批**（按节点/可用区/业务线），每批之间人工或自动校验。
- 用 **PDB（PodDisruptionBudget）** 保证自愿中断时可用副本不低于阈值（`minAvailable` / `maxUnavailable`）。
- 用 `minReadySeconds` 让"刚启动的 Pod"稳定一段时间才算可用，防止"探针通过但立刻崩溃"。

**2) 流量灰度**
- 用 **readiness 作为流量开关**：新版本先扩容但不 Ready → 观察日志/指标 → 再 Ready 接流量。
- 更精细：Service Mesh 分流（Istio/Envoy：按权重/请求头/用户 ID）、或 Ingress 的 canary 注解。
- 关键指标：错误率、P99 延迟、业务转化指标（不能只看技术指标）。

**3) 自动回滚**
- 监控告警联动 → 触发 `kubectl rollout undo` 或由发布平台回滚；`progressDeadlineSeconds` 触发"发布停滞"告警。
- **回滚本身也可能失败**（旧镜像被清理、配置已变）→ 镜像必须保留（`revisionHistoryLimit` 足够 + registry 不删旧 tag）。

**4) 风险清单（这是这题的重点）**
| 风险 | 说明 | 对策 |
|---|---|---|
| 镜像拉取打爆 registry / 节点带宽 | 批量起 Pod 同时拉镜像 | 预热 + P2P + 分批 + 限制并发 |
| 数据库 schema 变更不兼容 | 新旧版本并行期间 schema 冲突 | 先兼容性变更（双写/加列），再切版本（expand-contract 模式） |
| 配置/密钥不一致 | 新版本读旧 ConfigMap | 版本化配置；发布与配置变更解耦 |
| 依赖服务被打挂 | 新版本 QPS 上涨 | 限流熔断、连接池预热、预发压测 |
| 探针误判导致雪崩 | 启动慢 → liveness 杀 → 重启循环 | 用 startupProbe，liveness 不依赖外部 |
| 节点资源不足 | HPA 扩出 Pod 但调度失败 | 预留 buffer + cluster autoscaler |
| 单批次失败扩大 | 灰度批次划分不合理 | 小批起（1% → 5% → 25% → 100%），每批有 gate |

**5) 兜底**
- **Canary 集群 + 全链路压测**；发布窗口避免与业务高峰/其他变更重叠；发布期间值班 + 一键回滚脚本演练。

---

### Q19. Pod 里的容器之间怎么通信？为什么它们能通过 `localhost` 互访？ ⭐⭐⭐｜高频（基础但常答错）

**考核点**：Pod 网络模型的准确理解。

**解答**：
- Pod 内容器**共享同一个 network namespace**（由 pause 容器持有），因此：
  - 共享同一个 IP、同一个端口空间 → 容器 A 监听 8080，容器 B 就不能再监听 8080（会冲突）；
  - `127.0.0.1` 是同一个 loopback → **可以直接 `localhost:8080` 访问彼此**；
  - 它们之间没有 NAT、没有额外转发，走的是内核 loopback，性能最高。
- 共享的方式：containerd/runc 在创建后续容器时，用 **"加入已有 netns"** 的方式（`/proc/<pause-pid>/ns/net` 作为目标 netns），而不是新建。
- 除 net 之外，Pod 内容器还共享 **IPC**（SysV 共享内存，可用于高性能通信）与 **UTS**（同一个 hostname）；**默认不共享 PID**（各自 PID 1，用 `shareProcessNamespace` 可开）；**不共享 mnt**（各自 rootfs，通过 volume 共享文件）。

**延伸追问**：
- "同一 Pod 内两个容器都监听 80 端口会怎样？" → 第二个启动失败（`bind: address already in use`）。
- "Pod 的 IP 是谁分配的？" → CNI 的 IPAM 插件（Calico IPPool / host-local / Cilium）；由 kubelet 在 sandbox 创建时通过 CNI 获得并写回 Pod status。
- "sidecar 采集主容器日志怎么做的？" → 通过共享的 `emptyDir` volume（应用写日志到 volume，sidecar 读同一目录）或 stdout 由 runtime 统一收集。

---

### Q20. 节点 NotReady 了，你的排查顺序是什么？ ⭐⭐⭐⭐｜高频

**考核点**：kubelet 内部机制与实操。

**解答（从"控制面视角 → 节点视角"）**：
```
1) 控制面：kubectl describe node <n> | grep -A5 Conditions
   → Ready=False 的 reason/message（如 KubeletNotReady, NetworkUnavailable）
   → kubectl get lease <n> -n kube-node-lease   看 renewTime 是否过期（心跳）
2) 到节点上：systemctl status kubelet → journalctl -u kubelet -n 200 --no-pager
   典型错误：
     · "PLEG is not healthy"        → runtime 卡住/节点过载
     · "failed to run Kubelet: unable to load bootstrap kubeconfig" → 证书问题
     · "Error getting node: not found" → 节点对象被误删/DNS 问题
     · "cgroup driver mismatch"     → 配置不一致
3) 容器运行时：crictl info / systemctl status containerd
   → runtime 挂了 → kubelet 无法获知容器状态 → NotReady
4) 资源：df -h（/var/lib/kubelet、/var/lib/containerd 满）、df -i（inode 耗尽）、free -m
   → kubelet 有磁盘压力时会主动 NotReady（DiskPressure taint）
5) 网络：ip_forward、CNI 插件进程是否存活、能否到达 apiserver（curl -k https://apiserver/healthz）
6) 时间：chrony/ntp 是否漂移（etcd 与证书都依赖时间）
```

**处理动作**：修复后 kubelet 会自愈重新注册；若节点脏数据（容器泄漏），要 `crictl rm -f` 清理孤儿容器 + `systemctl restart containerd`（注意：**重启 containerd 会让该节点所有容器短暂不可用**，要评估）。

**关键认知**：**节点 NotReady 时，Pod 不会立刻迁移** —— NodeController 会打 `not-ready`/`unreachable` 污点，但 Pod 默认容忍 300s（`tolerationSeconds`），5 分钟后才被驱逐重建。所以"故障恢复时间"= 检测时间（40s）+ 容忍期（5min）+ 重建时间。

---

## 第 14 章 美团（重业务落地、重资源利用、重有状态服务）

### Q21. 你们线上 requests/limits 怎么设？设得不合理会有什么后果？ ⭐⭐⭐⭐｜高频

**考核点**：资源管理的实战理解，不是背概念。

**解答**：
**两者的不同作用（必须分清）**：
| | requests | limits |
|---|---|---|
| 谁用 | **调度器**（决定能否放下）+ **cgroup weight/shares**（竞争时的相对权重）+ **驱逐排名** | **cgroup max**（硬限：CPU 被限流、内存被 OOM） |
| 超卖 | 允许（节点上 requests 总和 ≤ allocatable，但实际使用可超） | 不允许（每个容器单独受限，不能超） |
| 后果 | 设太小 → 被调度到很挤的节点 → 竞争时被压制、优先被驱逐 | CPU 设太小 → 限流抖动；内存设太小 → OOMKilled |

**典型错误配置及其现象**：
1. **只设 limits 不设 requests**（或 requests 远小于 limits）→ QoS 变 Burstable，节点超卖严重，CPU 竞争时所有 Pod 一起变慢，驱逐时优先被杀。
2. **CPU limits 设得刚好等于峰值** → 每次接近峰值就 throttling，表现为"P99 延迟莫名抖动，但 CPU 使用率只有 50%"（因为被 quota 卡住）。→ **经验：CPU 可以只设 requests 不设 limits**（让它在空闲时用满机器），内存必须设 limits（内存无法"超用后回收"）。
3. **requests 设太大** → 调度困难（碎片化），节点利用率低（比如 requests 4C 但实际用 0.5C）。
4. **Java 堆没随 limits 调整** → 堆 + 元空间 + 直接内存 + 线程栈超过 limit → 容器被 OOMKilled，而 JVM 自己没抛 OOM。
5. **Guaranteed QoS 的代价**：必须 requests == limits，失去弹性 → 适合核心服务，不适合批处理。

**方法论**：
- 用历史监控（P95/P99 使用量）定 requests，用 P99.9 或峰值 × 1.2 定内存 limit；
- CPU 用 `requests = P95`，不设 limits 或设 `3~5 倍`；
- 用 **VPA 的 recommend 模式**（只看建议不改）辅助定值；
- 定期跑 **资源使用率报表**，找出"requests/实际使用 > 5 倍"的浪费。

---

### Q22. 什么是有状态服务？StatefulSet 为什么能支撑有状态服务，它解决了哪些问题？ ⭐⭐⭐⭐｜中高频

**考核点**：是否有真实的有状态部署经验。

**解答**：
**有状态服务的四个需求**：
1. **稳定身份**：重启后还是"同一个成员"（不能像 Deployment 那样随机名字）。
2. **稳定网络标识**：其他成员要能通过固定地址找到它（集群成员互相发现）。
3. **独立存储**：每个副本一份数据，不能共享（MySQL 主从不能共用一块盘）。
4. **有序启停**：集群引导要按序（先起 seed 节点，再起其他），缩容要反向（先摘从再摘主）。

**StatefulSet 的对应设计**：
| 需求 | StatefulSet 的答案 |
|---|---|
| 稳定身份 | Pod 名为 `<name>-<ordinal>`，删除重建同名 |
| 稳定网络 | 必须配 Headless Service → `pod-0.svc.ns.svc.cluster.local` 稳定 DNS |
| 独立存储 | `volumeClaimTemplates` → 每个 Pod 自动创建 `data-<pod-name>` 的 PVC，**Pod 重建后复用同名 PVC** |
| 有序启停 | 默认 `podManagementPolicy: OrderedReady`：0 就绪→起 1→起 2；缩容反序 |

**它没有解决的问题（这题的高分点）**：
- **成员变更的一致性**：StatefulSet 只保证"有序启停"，不保证"集群拓扑正确"。真正的成员管理要靠应用自己（如 etcd 的 `member add`、Redis Cluster 的 slot 迁移）或 operator。
- **数据迁移与再平衡**：扩容后新 Pod 拿到空 PVC，数据怎么搬？→ 需要应用层或 operator（如 Vitess、TiDB Operator）。
- **主从切换**：谁是主？→ 靠应用/operator，K8s 不管。
- **PVC 不随缩容删除**（1.27 才可配），容易留下垃圾。
- **滚动更新慢**：逐个等 Ready，20 个副本可能几十分钟；`OnDelete` 策略可手动控制节奏。

**延伸**："为什么很多团队用 Operator 而不是裸 StatefulSet？" → Operator 把"领域知识"（如何选主、如何扩缩容、如何备份恢复）编码进控制器，把运维操作变成声明式 API。这是"有状态服务上云"的通用解法。

---

### Q23. 如何提高 K8s 集群的资源利用率？ ⭐⭐⭐⭐｜高频（成本敏感型公司必问）

**考核点**：成本治理的体系化思路。

**解答**：
**先量化现状**：
```
利用率 = 实际使用 / allocatable
两个损耗点：
  ① 调度损耗：requests 总和 << allocatable（requests 设太大 / 节点规格不合适）
  ② 运行损耗：实际使用 << requests（超配）
```

**六类手段**：
1. **调小 requests**（最直接）：基于 P95 实测值 + VPA recommend；把"CPU requests 虚高"的负载降下来。
2. **允许 CPU 超卖**（不设 CPU limits 或设宽）：CPU 是可压缩资源，超卖只会变慢不会崩；内存是**不可压缩**资源，不能超卖。
3. **提升装箱密度**：调度器打分改为 `MostAllocated`（binpacking）→ 把 Pod 塞满少数节点，空出整机；
   配合 **节点池分级**（大规格/小规格）+ **node 亲和**（同规格聚拢）。
4. **混部（在线 + 离线）**：在线服务（延迟敏感，CPU 波动大）与离线任务（批处理，可被压制）混跑 → 提高平均利用率。技术上需要：
   - **CPU 隔离**：`cpu manager` static 策略给在线服务绑核（独占），离线用剩余核；
   - **离线任务低优先级 + 可抢占**（PriorityClass + 驱逐）；
   - **cgroup v2 + PSI** 做干扰检测，避免离线把在线拖慢。
   → 这是阿里/字节"在离线混部"的核心，能提升 20%~40% 利用率。
5. **弹性伸缩**：HPA（业务维度）+ Cluster Autoscaler/Karpenter（节点维度）+ 定时伸缩（预测性）。
6. **清理浪费**：僵尸 Pod、超大 requests 的调试负载、长期空闲的 namespace、未使用的 PVC。

**注意事项（体现工程判断）**：
- 利用率不是越高越好：**预留 buffer** 应对突发（否则 HPA 扩不出来、调度失败）。
- 混部会引入**性能干扰**，核心链路必须有保障机制（绑核、限流、PSI 监控）。
- 利用率提升必须配**容量水位告警**（否则一次流量高峰就雪崩）。

---

## 第 15 章 京东 / 百度（重排障、重存储、重稳定性）

### Q24. 一个 Pod 一直 `Pending`，怎么排查？把所有可能性列出来。 ⭐⭐⭐｜高频

**考核点**：排障的系统性。

**解答**：
先区分两类 Pending：
```
A. 未调度（没有 nodeName）→ kubectl get pod -o wide 看 NODE 为空
   → kubectl describe pod 看 Events: "FailedScheduling 0/xx nodes are available: <原因>"

B. 已调度但沙箱/容器未就绪 → NODE 有值，但 ContainerCreating
```

**A 类（调度失败）的完整原因清单**：
| 事件关键字 | 根因 | 处理 |
|---|---|---|
| `Insufficient cpu/memory` | 节点 requests 已满 | 扩节点、降 requests、清理闲置 Pod |
| `node(s) had untolerated taint` | 节点有污点（含 NotReady/DiskPressure） | 加 toleration 或修节点 |
| `node(s) didn't match Pod's node affinity` | nodeSelector/affinity 太严 | 检查 label 是否打上 |
| `node(s) didn't match Pod's node affinity/selector`（NoSchedule） | `nodeName` 指定了不存在的节点 | 检查拼写 |
| `persistentvolumeclaim not found / unbound` | PVC 未绑定 | 检查 StorageClass、provisioner 日志 |
| `node(s) had volume node affinity conflict` | PV 在别的可用区 | 换节点或换 PV |
| `would violate pod topology spread constraints` | 拓扑约束太严（如 maxSkew:1 + 单节点） | 放宽约束 |
| `node(s) didn't satisfy existing pods anti-affinity` | 反亲和导致没地方 | 放宽 |
| `exceeded quota`（不是调度事件，是创建失败） | ResourceQuota | 调整配额 |
| `0/xx nodes available` 且无具体原因 | 所有节点都被过滤（如全部 cordon） | `kubectl get nodes` 看状态 |
| 完全没有事件 | **调度器没在工作**（scheduler 挂了）或 Pod 被 `schedulingGates` 阻塞（1.26 的 Pod Scheduling Readiness） | 检查 scheduler 日志 |

**B 类（创建阶段卡住）的原因**：
- **镜像拉取慢/失败**：`ImagePullBackOff`、registry 网络不通、imagePullSecret 缺失/过期；
- **CNI 失败**：`plugin "calico" failed`、IPAM 无可用 IP（网段耗尽）；
- **卷挂载慢**：CSI Attach 卡住（后端存储慢/配额满）、RWO 卷仍被别的节点挂载（"Multi-Attach error"）；
- **sandbox 创建失败**：runtime 异常、`/var/lib/containerd` 磁盘满、PID 耗尽。

**高分要点**：不要只答"资源不足"，要展示"先分 A/B 两类，再按事件关键字定位，最后给可操作命令"。

---

### Q25. PV/PVC 绑定失败有哪些原因？`Multi-Attach error` 怎么处理？ ⭐⭐⭐⭐｜中高频

**考核点**：存储链路的真实排障。

**解答**：
**绑定失败原因**：
1. **无匹配 PV**：capacity 不够、accessModes 不匹配（PVC 要 RWX 但 PV 只有 RWO）、`storageClassName` 不一致（PV 有 class 而 PVC 写 `""` 表示不要动态供给）。
2. **StorageClass 不存在 / provisioner 不可用**：`kubectl get sc`、provisioner Pod 日志（external-provisioner）。
3. **`volumeBindingMode: WaitForFirstConsumer`**：PVC 会一直 Pending 直到有 Pod 使用它 —— **这是正常现象**，不是故障（新手常误判）。
4. **配额/权限**：云账号配额满、CSI 的 IAM 权限不足（无法创建卷）。
5. **selector 太严**：PVC 上写了 `selector` 但 PV 的 label 不匹配。

**`Multi-Attach error for volume ... Volume is already exclusively attached to one node`**：
- 根因：**RWO 卷在被旧节点挂载时，新节点又要挂载**。典型场景：节点故障后 Pod 被重新调度到新节点，但旧节点的 kubelet 还没释放 attachment（或 attachment 记录残留）。
- 处理顺序：
```bash
# 1) 确认旧节点是否真的挂了
kubectl get node <old>                     # 若还是 Ready，说明旧 Pod 还占着卷
kubectl get pod -o wide | grep <pvc>       # 找到消费者

# 2) 若旧节点真的失联 → 强制清理 attachment
kubectl delete volumeattachment <name>     # 让 external-attacher 重新 attach
# 或（云环境）在 IaaS 控制台强制 detach

# 3) 若旧节点存活：先确保旧 Pod 被真正删除（--force 也不行，要确认容器已停）

# 4) 长期方案：用 RWX 存储 / 用 StatefulSet 的独立 PVC / 开启 CSI 的
#    "attach 超时 + 重试" 配置
```
- **根本原因**：K8s 的卷 attach 是**节点级独占**语义，且依赖 `VolumeAttachment` 对象的状态机；节点异常时状态不会自动纠正，需要人工/控制器介入。

---

### Q26. NetworkPolicy 是怎么生效的？为什么配了 NetworkPolicy 还是不通/还在通？ ⭐⭐⭐⭐｜中高频

**考核点**：网络策略的实现层次。

**解答**：
- **NetworkPolicy 本身只是声明**，由 **CNI 插件执行**（Calico 的 Felix → iptables/eBPF；Cilium → eBPF map）。**Flannel 原生不支持**（配了不生效，这是一个高频陷阱）。
- 语义要点（极易配错）：
  1. **策略是"白名单"**：一旦某个 Pod 被某条 policy 的 `podSelector` 选中，且 policyTypes 包含 Ingress/Egress，则该方向**默认拒绝所有**，只放行 rules 里列出的。
  2. **Ingress 与 Egress 必须成对考虑**：只配 Ingress 放行了 A→B，但 B 如果有 Egress 策略没放行到 A，仍然不通；反之如果 A 有 Egress 策略不允许到 B，也断。
  3. **`from/to` 里的三种选择器**：`podSelector`（同 namespace 或配 `namespaceSelector` 跨 ns）、`namespaceSelector`、`ipBlock`（CIDR，注意 `except`）。
  4. **空 `podSelector: {}` 表示"选中本 namespace 所有 Pod"**；`policyTypes` 省略时由 rules 推断（有 ingress 规则则隐含 Ingress）。
- **"配了还是通"的排查**：
  - 确认 CNI 支持策略且 Felix/Cilium agent 正常；
  - `kubectl get networkpolicy -o yaml` 确认 `podSelector` 真的匹配目标 Pod 的 label；
  - 用 `kubectl exec` 在源 Pod 里试；同时在目标节点 `iptables -L -n | grep -i policy` 或 `cilium policy get` 看规则是否落地；
  - 注意 **hostNetwork 的 Pod 不受 NetworkPolicy 约束**（因为它用宿主网络栈）；
  - 注意 **DNS 出站**：如果配了 Egress 策略，必须放行到 CoreDNS 的 53 端口，否则"服务全挂但看起来网络通"。

---

### Q27. 容器里 `nproc`/`free -m` 看到的是宿主资源，怎么让应用正确识别容器限额？ ⭐⭐⭐｜中频（Java/中间件团队必问）

**考核点**：`/proc` 与 cgroup 的关系，实操经验。

**解答**：
- 原因：`/proc/cpuinfo`、`/proc/meminfo`、`sysconf(_SC_NPROCESSORS_ONLN)` 读的是**宿主全局信息**，不是 cgroup 限制。所以 `nproc` 看 64 核，实际容器只有 2 核。
- 正确读法：
```bash
# cgroup v2
cat /sys/fs/cgroup/cpu.max          # "200000 100000" → 2 核
cat /sys/fs/cgroup/memory.max       # 字节数，或 "max"
# cgroup v1
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us / cpu.cfs_period_us
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```
- 应用侧适配：
  - **JVM**：JDK 8u191+ / 10+ 支持 `UseContainerSupport`（默认开），会自动读 cgroup。但**在 cgroup v2 上老版本 JDK 读不到**（只认 v1 路径）→ 表现为"堆按宿主内存算，容器被 OOMKilled"。解决：升级 JDK 或用 `-XX:MaxRAM=...`。
  - **Go**：`runtime.NumCPU()` 读的是宿主核数 → 协程池/`GOMAXPROCS` 会开太大 → 大量线程争抢 2 核 → 上下文切换爆炸。解决：用 `automaxprocs`（uber-go/automaxprocs）读 cgroup 限制。
  - **Node.js**：同理需 `--max-old-space-size` 或使用 `cgroup` 感知的库。
  - **Nginx**：`worker_processes auto` 会用宿主核数 → 应显式设。
- 另一个更隐蔽的坑：**`cpuset` 与 `cpu.max` 不一致**。`cpu.max` 限"总量"但不限"哪些核"，会导致进程在多核间反复迁移 → cache 失效；需要 `cpuset` 绑核（K8s 的 `cpu manager static` 策略）。

---

## 第 16 章 快手 / 滴滴 / 小红书 / 华为 / 金融行业

### Q28. 你了解 CRI 吗？containerd 收到 `RunPodSandbox` 之后做了什么？ ⭐⭐⭐⭐｜进阶

**考核点**：运行时内部链路的细节。

**解答**：
```
containerd 的 CRI plugin 收到 CRI gRPC RunPodSandbox：
 1) 校验/补全配置（namespace、cgroup 路径、log 目录）
 2) 生成 sandbox 的 OCI spec（以 pause 镜像为入口）：
      · namespaces: 新建 net/ipc/uts/pid/mnt，**不新建 user**（除非配置）
      · cgroup: 创建 pod 级 cgroup（kubepods/<qos>/pod<UID>）
 3) 通过 snapshotter（overlayfs）准备 pause 容器的 rootfs
 4) fork containerd-shim-runc-v2（新 shim 进程）
 5) shim → runc create/start → pause 容器运行
 6) **调用 CNI**：
      · 读 /etc/cni/net.d/*.conf 与 /opt/cni/bin
      · 执行 CNI 二进制，环境变量带 CNI_COMMAND=ADD、CNI_NETNS=/proc/<pause-pid>/ns/net
      · 解析返回的 IP，写入 sandbox 的 status（后续 CreateContainer 时用于生成 /etc/hosts、
        resolv.conf 的 sandbox 上下文）
 7) 返回 sandbox ID 给 kubelet
```
**后续 `CreateContainer`（业务容器）**：
- 关键点：**复用 sandbox 的 netns（不新建）**，cgroup 挂到 pod 级 cgroup 下，rootfs 用业务镜像的 layers + 一个新 upperdir。
- 所以：**业务容器的 rootfs 是独立的，但 net/ipc/uts 归属 sandbox** —— 这就是"Pod 内共享网络但不共享文件系统"的机制来源。

**延伸**：
- `crictl inspectp <sandbox>` 能看到 pause 容器的信息与 IP；
- `crictl pods` 列出所有 sandbox，`crictl ps -a` 列出所有容器；
- K8s 1.2x 起 sandbox 也支持 **Windows HostProcess / 多网络接口（Multus）**，CNI 调用可能多次（每个额外网卡一次）。

---

### Q29. 华为/运营商方向：容器在 ARM64、国产化环境、离线环境下的适配问题？ ⭐⭐⭐⭐｜特色题

**考核点**：异构环境工程经验。

**解答**：
- **多架构镜像**：必须构建 `manifest list`（`docker buildx build --platform linux/amd64,linux/arm64`），否则 ARM 节点上拉取 amd64 镜像会报 `exec format error`。
- **基础镜像与依赖**：ARM64 生态下某些库（尤其 Java native、Python C 扩展）需重新编译；`distroless`/`alpine` 的 arm 版本要确认存在。
- **CPU 特性差异**：ARM 上 `nproc`、`lscpu` 输出不同；绑核（`cpuset`）在 ARM 的大小核（如鲲鹏/飞腾）上要注意 NUMA/核类型；**内存页大小**（64K 页的 ARM 发行版）会让某些应用（如 JVM、Redis）行为不同。
- **离线环境**：
  - 镜像仓库内网化（Harbor）+ 定期同步清单（`skopeo copy` / `regctl`）；
  - 所有依赖（CNI 插件、CSI 驱动、helm chart）离线包；
  - 时间同步、证书签发（离线 CA）；
  - 无法拉取官方镜像时，用 `docker save/load` 或 `crictl` 直接导入。
- **信创栈适配**：麒麟/统信 OS + 达梦/人大金仓 + 鲲鹏/飞腾 → 常遇到内核版本较老（cgroup v2 不可用、overlay 不支持某些特性）→ 需要 **backport 或换运行时**（如用 cri-o + v1 cgroup）。

---

### Q30. 金融行业方向：容器化后如何满足"审计、隔离、可追溯"的合规要求？ ⭐⭐⭐⭐｜特色题

**考核点**：非功能需求的工程化。

**解答**：
- **审计**：apiserver 的 **audit policy**（记录谁在何时对哪个对象做了什么，分 Metadata/Request/RequestResponse 三级）；日志集中存储且**不可篡改**（写入 WORM 存储）；开启 etcd 的审计日志。
- **隔离**：
  - 命名空间 + 网络策略（默认拒绝）+ 资源配额（ResourceQuota + LimitRange）；
  - 敏感业务用**独立节点池 + 污点隔离**（避免与普通业务混部）；
  - 关键业务用 **Guaranteed QoS + 绑核**；核心交易链路可用 Kata/gVisor 做更强隔离。
- **可追溯**：
  - 镜像**全链路签名**（cosign）+ 部署时准入校验（只允许已签名镜像）；
  - **SBOM**（记录每个镜像的依赖清单）+ 漏洞扫描结果留档；
  - 镜像版本固定 digest（禁止 `latest`），保证"现场可复现"；
  - 变更留痕：GitOps（ArgoCD）→ 每次部署对应一个 commit，可回滚可审计。
- **数据安全**：
  - etcd 加密（KMS provider）+ Secret 不明文；
  - 敏感配置通过外部密钥管理（Vault）注入，不落 ConfigMap；
  - 容器内不落盘敏感数据（readOnlyRootFilesystem + emptyDir 加密）。
- **合规常见硬要求**：日志留存 ≥6 个月、变更双人复核、生产与测试物理/逻辑隔离、灾备演练记录。

---

## 第 17 章 综合场景设计题（终面 / TL 面）

### Q31. 【场景】你们的 K8s 集群突然出现大量 Pod 被重启，业务方反馈"服务偶尔 502"。给出你的完整排查与治理方案。 ⭐⭐⭐⭐⭐

**考核点**：多因素并发故障的定位能力、收敛思路。

**解答框架**：

**第一步：确认现象边界（避免瞎猜）**
```
· 是全集群还是某几个节点？（kubectl get pod -A | grep -v Running 分布）
· 是特定服务还是所有服务？（看 502 的来源：Ingress？网关？Service？）
· 时间上是否与某次变更/发布/扩缩容相关？（对齐发布记录、监控曲线）
· 重启原因是什么？（kubectl describe pod 的 LastState/Reason：
    OOMKilled / Error / Completed / ContainerCannotRun / Evicted）
```

**第二步：按原因分类处置**

| 原因分布 | 根因方向 | 处置 |
|---|---|---|
| 大量 `OOMKilled` | 节点内存压力（超卖）、应用内存增长、limit 太小 | 查 `memory.working_set` vs requests；调 limits 或降密度；关 swap；加 `system-reserved` |
| 大量 `Evicted` | 节点磁盘/内存压力触发 | 看 `kubelet` 的 eviction 日志、`df -h`、日志未轮转、容器写盘 |
| `Error` + 应用日志报连接失败 | 依赖服务（DB/Redis）问题 | 查依赖方的连接数/慢查询，确认是否被上游打挂 |
| 全部容器同时重启 | **kubelet/containerd 重启、节点重启、镜像仓库不可用** | `journalctl -u kubelet`、节点 uptime |
| 只有新版本 Pod 重启 | 发布引入的问题（配置/探针/资源） | 回滚验证，用 `rollout undo` |
| 502 但 Pod 没重启 | **流量打到未 Ready 的 Pod / 优雅退出不当 / 连接复用** | 检查 readiness、preStop、keepalive |

**第三步：502 的专项分析（这题的关键）**
- **502 = 网关/Ingress 拿到了无效的上游响应**，常见四个原因：
  1. **Pod 正在终止**：Endpoint 摘除滞后（kube-proxy 更新延迟）→ 需要 **preStop sleep**；
  2. **Pod 未 Ready 却接了流量**：readiness 探针不灵敏（依赖的服务挂了但探针只看端口）；
  3. **连接复用（keepalive）打到已关闭的 Pod**：上游连接池未感知后端下线 → 需要在网关上配 `maxRetries`/`proxy_next_upstream`，或后端返回 `Connection: close` 时正确处理；
  4. **上游超时**：Pod 变慢（CPU 限流、GC）→ 网关超时后返回 502/504。

**第四步：治理（防止再犯）**
- **可观测性**：加 `kube_pod_container_status_last_terminated_reason` 告警、OOM 告警、重启率告警；
- **容量**：留 buffer、去掉过度超卖、核心服务设 Guaranteed；
- **发布规范**：preStop + grace 调优、startupProbe、分批灰度；
- **节点治理**：日志轮转（`log-opts max-size`）、镜像 GC、磁盘水位告警、`system-reserved` 预留；
- **演练**：定期做"节点故障/大规模重启"演练，验证自愈链路。

---

### Q32. 【场景】让你从零设计一个公司的容器平台（不只是装个 K8s），你会怎么规划？ ⭐⭐⭐⭐⭐

**考核点**：平台工程视角。

**解答框架（六层）**：
```
① 基础设施层：节点池规划（规格分级、可用区、GPU 池、边缘节点）、
   网络规划（Pod CIDR 与 Service CIDR 大小、与现有 IDC 网段的冲突检查、
   是否需要 BGP 与物理网络互通）、存储（本地 SSD / 分布式存储 / 云盘 CSI）
② 集群层：单集群 vs 多集群（爆炸半径）、集群版本与升级策略、
   控制面高可用（etcd 独立部署、apiserver 多副本）、
   节点自愈（NPD/kubelet 自动恢复、坏节点自动替换）
③ 运行时与安全层：containerd + runc（或 Kata）、
   镜像仓库（Harbor 做代理缓存 + 漏洞扫描 + 签名）、
   准入体系（PSA + Kyverno/OPA 策略）、RBAC 与多租户隔离、
   密钥管理（Vault / KMS）
④ 平台能力层（对业务暴露什么）：
   统一的应用模型（Deployment/StatefulSet 模板 + 默认最佳实践）、
   配置与密钥注入、服务发现与流量接入（Ingress/Gateway/Mesh）、
   弹性（HPA/VPA/CA）、发布体系（灰度/回滚/审批）、
   中间件托管（Operator：MySQL/Redis/Kafka）
⑤ 可观测与运维层：
   指标（Prometheus 分片 + 长期存储）、日志（Loki/ES + 采集 DaemonSet）、
   链路追踪、告警分级与值班、成本分摊（按 namespace/团队统计资源与费用）
⑥ 工程与治理层：
   GitOps（ArgoCD）单一事实源、变更审计、容量规划（季度评审）、
   SLO 体系（平台自身 SLI：调度成功率、Pod 启动时延、apiserver 可用性）、
   故障演练（混沌工程）、文档与培训
```

**必须讲清的取舍（面试官最看重）**：
- **单集群 vs 多集群**：单集群运维简单但爆炸半径大；多集群隔离好但成本高、跨集群调度复杂 → 一般按"业务域/地域/合规"切分。
- **自建 vs 托管**：自建可控但需要专门团队维护控制面；托管省人力但受限于云厂商能力与成本。
- **CNI 选型**：Flannel（简单）/ Calico（策略 + BGP）/ Cilium（性能 + 可观测），取决于规模与是否需要网络策略。
- **"平台不是 K8s"**：业务方要的是"部署一个应用"，不是"写 YAML"。所以要有模板化、默认值、校验、一键回滚 —— **降低认知负担才是平台的价值**。

---

### Q33. 【场景】线上一个 Service 的流量全部打到一个 Pod 上，怎么排查？ ⭐⭐⭐⭐

**考核点**：负载均衡链路的细节。

**解答**：
按"三层可能"排查：
1. **Service 层（kube-proxy）**
   - iptables 模式下用 `statistic mode random` → 理论上均匀；但如果 **Endpoint 数量少（比如 2 个，概率 50/50）**、或者**客户端用了长连接**，看起来就像"全打一个"。
   - 检查 `kubectl get endpointslice` 端点数量是否正确（少了很多 → readiness/selector 问题）。
   - ipvs 模式：`ipvsadm -Ln` 看 Real Server 列表和权重。
2. **客户端层（最常见）**
   - **HTTP keepalive / 连接池**：客户端建立少量长连接并复用 → 所有请求都走同一条 TCP → 落到同一个 Pod。
   - gRPC 尤其明显（单个 HTTP/2 连接多路复用）。
   - 解决：客户端连接池轮转、`maxIdleConns`、gRPC 的 `round_robin` + 多 subchannel、或服务端主动 `GOAWAY`。
3. **应用层**
   - `sessionAffinity: ClientIP` 配置导致按源 IP 黏性（NAT 后所有客户端同一个源 IP）。
   - 某个 Pod 处理慢 → 其他 Pod 空闲但连接被占满（表象是"打到一个"，实质是"另一个已经过载不可用"）。

**验证方法**：在网关/客户端侧统计"每个 Pod IP 的请求数分布"；在 Pod 侧看 `ss -tn state established` 的连接来源；对比 `container_network_receive_packets_total`。

---

### Q34. 什么是"声明式"与"命令式"？K8s 为什么坚持声明式？幂等性从哪来？ ⭐⭐⭐

**考核点**：设计哲学理解。

**解答**：
- **命令式**：你告诉系统"做什么动作"（`docker run`、`kubectl run`、`kubectl scale`）。问题是：动作不可重放、失败后状态未知、无法描述"目标状态"。
- **声明式**：你提交"期望状态"（YAML 的 spec），系统持续收敛。优势：
  1. **幂等**：重复提交同一 spec，结果一致（`apply` 天然可重复）；
  2. **可恢复**：任何组件重启后都能从"当前状态 vs 期望状态"重新计算，不需要知道"历史操作序列"；
  3. **多写者安全**：控制器/运维/CI 都可以"表达意图"，由系统协调；
  4. **可审计**：期望状态本身就是记录。
- **幂等的来源**：控制器每次都做 `desired - actual` 的 diff，只对差异动作；加上 workqueue 去重、API 的乐观并发（resourceVersion），所以重试安全。
- **注意**：声明式不保证"实时"，只保证"最终一致"；且**不是所有操作都能声明式**（如"重启一次"没有声明式表达，只能 `rollout restart` 改 annotation）。

---

### Q35. 如果一个团队要把 100 个微服务从虚拟机迁到 K8s，你会怎么排优先级和规避风险？ ⭐⭐⭐⭐⭐

**考核点**：迁移工程。

**解答**：
**分四批，按"风险/收益"排**：
1. **第一批（试点，低风险高收益）**：无状态、外部依赖少的服务（如内部工具、报表、网关边角服务）。目的：验证平台能力、跑通 CI/CD、暴露问题（镜像、健康检查、配置）。
2. **第二批（主力无状态服务）**：核心 API 服务。此时平台已稳定，重点做**容量规划、HPA、灰度**。
3. **第三批（有状态/中间件）**：MySQL/Redis/Kafka。要点：
   - 用 Operator 而非裸 StatefulSet；
   - 存储性能必须先压测（本地 SSD vs 云盘差异巨大）；
   - 数据迁移方案（停机 vs 双写 vs DTS）；
   - **回滚预案**必须能退回虚拟机。
4. **第四批（特殊负载）**：GPU、需要特权、需要 hostNetwork、超低延迟交易链路。这些可能**不适合容器化**，要接受"部分保留在 VM"。

**风险清单**：
| 风险 | 对策 |
|---|---|
| 性能回退（网络 NAT 开销、CPU 限流） | 迁移前压测对比；关键链路用 hostNetwork 或 eBPF CNI；不设 CPU limits |
| 配置/密钥管理变化 | 统一走 ConfigMap/Secret + Vault，禁止配置烧进镜像 |
| 发布节奏变化 | 先把 CI/CD 与灰度能力建好，再迁业务 |
| 监控断层 | 迁移前后指标口径一致（容器指标 vs 主机指标要能对齐） |
| 组织阻力 | 培训 + 提供"无痛模板" + 保留旧路径一段时间 |
| 单点故障放大 | 多可用区 + PDB + 反亲和，避免"一个节点挂一半服务"|

---

## 第 18 章 补充题库（快手 / 滴滴 / 小红书 / 华为 / 网易 / 通用）

### Q36. 【快手方向】K8s 里怎么调度 GPU？Device Plugin、MIG、Time-Slicing 分别是什么？ ⭐⭐⭐⭐

**考核点**：异构资源调度，AI/视频业务必备。

**解答**：
**基础模型：Extended Resource + Device Plugin**
- GPU 这类设备**不由 CPU/内存的 requests 模型管理**，而是通过 **Extended Resource**（`nvidia.com/gpu: 1`）申请。
- **Device Plugin** 是运行在每个 GPU 节点上的 DaemonSet，向 kubelet 注册资源名与数量；Pod 请求时，kubelet 通过 **Allocate 接口**拿到"分配哪几张卡"，并**在容器启动前把设备节点（`/dev/nvidia0`）注入容器**（同时设置环境变量 `NVIDIA_VISIBLE_DEVICES`）。
- 关键约束：**Extended Resource 只支持整数、且 requests 必须等于 limits**（因为设备不可"部分分配"、不可压缩）。

**三种共享 GPU 的方式**：

| 方式 | 原理 | 粒度 | 隔离性 | 适用 |
|---|---|---|---|---|
| **独占**（默认） | 一张卡给一个 Pod | 整卡 | 强 | 训练、大模型推理 |
| **MIG**（Multi-Instance GPU，A100/H100） | 硬件把卡切成最多 7 个实例（各自有独立显存/SM/缓存） | 1/7 卡 | **硬件级强隔离** | 小模型推理、多租户 |
| **Time-Slicing**（GPU 共享） | 驱动层按时间片轮转，多个容器"看到"同一张卡 | 时间片 | **弱（显存不隔离，会互相 OOM）** | 开发测试、低负载推理 |
| **MPS**（Multi-Process Service） | 多个进程共享一个 CUDA context | 更细 | 中（可配 SM 占比） | 小批量推理 |

**工程要点（高分点）**：
1. **NUMA 与拓扑**：GPU 挂在不同 PCIe/NVLink 拓扑下，跨 NUMA 访问会显著降带宽 → K8s 的 **Topology Manager**（`--topology-manager-policy=best-effort/restricted/single-numa-node`）配合 **CPU Manager static 策略**做"CPU-GPU-NIC 同 NUMA"对齐。这是训练集群调优的核心。
2. **GPU 的调度与"装箱"**：GPU 是最贵资源，调度应优先 `MostAllocated`（把 8 卡机填满）而不是打散，否则会出现"每台机器剩 1 张卡，8 卡任务永远调度不上"的碎片化死局。
3. **DRA（Dynamic Resource Allocation，1.31 起推进）**：新一代模型，支持"带参数的设备请求"（如"需要 2 张同 NUMA 的 A100，每张切 3 个 MIG 实例"），替代粗糙的扩展资源模型。
4. **视频转码场景**（快手的典型负载）：用 **硬件编码器**（NVENC/QSV）时，注意单卡并发会话数有限制（消费级卡有 license 限制）；转码任务多为**短时高吞吐**，适合用 **Job + 队列（Volcano/Kueue）** 做批量调度，而不是常驻 Deployment。

---

### Q37. 【滴滴方向】K8s 集群怎么做"两地三中心 / 同城双活"？K8s 本身能解决跨机房问题吗？ ⭐⭐⭐⭐⭐

**考核点**：架构边界认知，是否知道 K8s 的"不管什么"。

**解答**：
**先给结论：K8s 本身不能解决多活，它只是"单机房内的编排器"。** 跨机房的所有问题（延迟、脑裂、数据一致性、流量调度）都需要在 K8s 之上或之外解决。

**为什么单集群跨机房不可行**：
1. **etcd 的 Raft 对延迟极度敏感**：跨城 RTT 3~10ms，跨城 30~50ms → fsync + 多数派确认会让写延迟涨到不可接受，且网络抖动会引发 leader 频繁切换。**etcd 官方不建议跨可用区部署**（同城可勉强，跨城不行）。
2. **apiserver 与 etcd 之间的调用、kubelet 心跳**都受延迟影响。
3. **网络**：Pod 跨机房通信要经过隧道，MTU 与延迟都会劣化。

**主流方案（四选一或组合）**：
| 方案 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| **多集群 + 上层流量调度**（最常用） | 每个机房一个独立 K8s 集群，DNS/GSLB/网关按机房路由 | 故障域隔离清晰、etcd 本地化 | 需要统一的部署与配置下发（GitOps/Karmada） |
| **单集群 + 多 AZ（同城）** | etcd 同城 3 AZ（RTT <2ms），Pod 按 `topologySpreadConstraints` 分布 | 运维简单 | 跨 AZ 网络成本、AZ 故障时仍有部分 Pod 需重建 |
| **Karmada / Cluster API** | 多集群统一管控，策略分发、故障时自动迁移副本 | 全局视图 | 复杂度高，跨集群调度不是强一致 |
| **vCluster（虚拟集群）** | 在一个物理集群里"套"出多个 apiserver，实现软隔离 | 成本低、隔离快 | 共享物理故障域，不是真多活 |

**真正的多活难点（不在 K8s，而在数据）**：
- **有状态服务**：MySQL 跨城双写/主从延迟、Redis 跨城复制、消息队列的顺序性 → 这是"多活"的真正门槛。多数公司采用"**同城双活 + 异地冷备/温备**"而不是"异地双活"。
- **流量切换**：需要能"秒级"把入口切走（DNS TTL 太长不行，要靠 GSLB + 网关 + 客户端重试）。
- **数据一致性分级**：核心交易链路往往接受"异地 RPO > 0"，不能为了双活牺牲正确性。

**答题要点**：明确"K8s 是单故障域内的编排器"，然后给出"故障域划分 + 上层调度 + 数据分级"的三层架构。不要试图用 K8s 特性（如 topologySpread）去解决跨城问题。

---

### Q38. 【小红书方向】HPA 为什么"跟不上"突发流量？弹性伸缩的真正瓶颈在哪？ ⭐⭐⭐⭐

**考核点**：对"弹性"链路的完整理解。

**解答**：
**弹性是一条长链路，任何一环慢都会导致"扩不出来"**：
```
流量上涨 → 指标采集（15s 周期）→ 聚合（60s 窗口）→ HPA 计算（15s 周期）
 → 更新 replicas → 控制器创建 Pod → 调度（秒级）→ 拉镜像（**最慢，可能分钟级**）
 → 启动容器 → 应用预热（JVM/连接池/缓存）→ 探针通过 → 接流量
```
**总延迟通常在 1~5 分钟**，且镜像未预热时更长。

**为什么"跟不上"**：
1. **指标滞后**：`metrics-server` 的抓取 + 聚合导致 30~90s 的观测延迟 → HPA 看到的是"过去"的负载。
2. **HPA 的容忍度**：`tolerance: 0.1` 意味着偏差 10% 以内不动 → 小幅度上升不会扩。
3. **扩容的物理延迟**：调度 + 拉镜像 + 应用启动。**JVM 应用尤其慢**（类加载、JIT 预热、连接池建立）。
4. **节点不够**：HPA 扩出 Pod 但集群没资源 → 全部 Pending，还要等 Cluster Autoscaler 开机器（云上 1~3 分钟）。
5. **应用不能"横向扩展"**：有状态服务、单例任务、依赖 DB 连接数上限 → 扩了也没用。

**解法（按性价比排序）**：
| 手段 | 说明 |
|---|---|
| **预留 buffer** | 保持 `minReplicas` 高于均值，用"余量"抗突发（最有效、最朴素） |
| **提前扩容（预测式）** | 按历史曲线在高峰前定时扩容（CronHPA / KEDA 的 cron scaler） |
| **消除冷启动** | 镜像预热 + P2P 分发 + 应用预热（启动即建连接池/加载热点数据）+ 减小镜像 |
| **KEDA** | 基于事件源（Kafka lag、QPS、队列长度）扩展，比 CPU 指标更贴近真实压力 |
| **容量层弹性** | Karpenter（直接创建合适规格的 Node，比 Cluster Autoscaler 快）+ 预留节点池 |
| **入口层削峰** | 限流、排队、降级（把"扩不出来"变成"用户体验可控的等待"） |
| **多级缓存** | 减少扩容带来的缓存击穿风险 |

**关键认知**：**"弹性"不等于"自动扩容"，而是"容量、成本、体验三者的动态平衡"**。真正的弹性能力包含：快速扩容、**平滑缩容**（避免抖动）、**过载保护**（扩不出来时不崩）。

---

### Q39. 【华为方向】Kata Containers 是怎么做到"容器接口 + VM 隔离"的？和 gVisor 的本质区别？ ⭐⭐⭐⭐

**考核点**：安全容器两条技术路线的取舍。

**解答**：
**Kata 的架构**：
```
kubelet → CRI → containerd → kata-runtime（一个实现了 OCI runtime 接口的程序）
                                   │
                                   ├─ 启动一个轻量 VM（QEMU/Cloud-Hypervisor/Firecracker）
                                   ├─ 在 VM 内跑一个极简的 kata-agent（guest 内的代理）
                                   └─ 把容器的 rootfs 挂进 VM（virtio-fs / block device）
                                      然后在 VM 内创建 namespace 并 exec 进程
```
- **对上层完全透明**：因为 kata-runtime 遵循 OCI runtime-spec，所以 containerd/K8s 完全不知道背后是 VM。这就是"标准接口"的价值。
- **隔离强度 = VM 级**：容器进程运行在**独立的 guest 内核**里，宿主内核完全不可达 → 内核漏洞无法跨 VM 逃逸。
- **开销**：VM 启动时间（Firecracker 可到 ~125ms，QEMU 更慢）、内存开销（每个 Pod 一套 guest 内核，约几十 MB）、virtio 的 IO 路径比直接 syscall 长。

**gVisor 的架构**：
```
容器进程 → 发出 syscall → 被 runsc 拦截（用 seccomp/ptrace 让进程进入 Sentry）
   → Sentry（一个用 Go 写的用户态"内核"，实现了大部分 Linux 语义）
   → Sentry 再向宿主内核发起"安全的"少量 syscall
```
- **没有 VM**，而是在**用户态重新实现了一套内核**（Sentry），把绝大部分 syscall 在用户态处理掉。
- **优点**：不用硬件虚拟化，启动快、密度高、可跨平台。
- **缺点**：syscall 兼容性不完美（某些 ioctl/特殊 syscall 不支持，跑数据库/GPU 应用会踩坑）；syscall 密集场景性能损失大（10%~30%+）。

**本质区别（一句话）**：
> **Kata 用"虚拟化"隔离内核（硬件边界），gVisor 用"用户态内核"减少 syscall 面（软件边界）。**
> 前者适合"要跑任何镜像、要强隔离"，后者适合"跑不可信的、syscall 简单的负载（如 Serverless、函数计算）"。

**延伸**：这也解释了云厂商的选择差异 —— AWS Fargate 用 Firecracker（Kata 路线），Google Cloud Run 早期用 gVisor。

---

### Q40. 【网易 / 携程 / 平安方向】数据库到底能不能跑在 K8s 上？给出你的判断标准和落地路径。 ⭐⭐⭐⭐

**考核点**：技术判断力，敢不敢说不。

**解答**：
**先回答"能，但有条件"**，然后给判断标准。

**不适合直接上 K8s 的场景**：
1. **IO 延迟敏感 + 写密集**（OLTP 核心库）：容器层（overlay）+ 网络存储（云盘）带来的额外延迟不可接受。K8s 里数据库必须用**本地 NVMe 直通**（`hostPath`/local PV），且要 `Guaranteed QoS` + 绑核。
2. **需要稳定的内核参数与特权**：数据库常需调整 `vm.swappiness`、`huge pages`、`fs.aio-max-nr` 等**全局 sysctl**（容器里默认只读）；某些需要 `IPC_LOCK`、RAW IO。
3. **升级/重启对业务的冲击**：StatefulSet 的滚动更新是"逐个重启"，无法像 VM 那样"先做副本再切"。需要 operator 支撑。
4. **存储的"持久化确定性"**：`Delete` 回收策略 + 误删 PVC = 数据永久丢失。

**适合上 K8s 的场景**：
- 开发/测试环境、非核心业务库；
- 云原生数据库（TiDB、CockroachDB、Vitess）本来就是为分布式设计、有 operator；
- 中等规模、可接受"运维复杂度换弹性"的场景。

**落地的六条硬要求**：
```
1. 用 Operator（而非裸 StatefulSet）—— 它负责主从切换、备份、扩缩容、故障自愈
2. 本地 SSD + local PV + nodeAffinity 固定（或分布式存储专门优化）
3. Guaranteed QoS（requests == limits），cpu manager static 绑核，关闭 swap
4. 存储回收策略设为 Retain，PVC 与数据分离，独立备份（Velero / 物理备份）
5. 内核参数通过 DaemonSet 在节点层统一设置（不是容器内）
6. 明确的"降级路径"：容器化失败时能否快速切回虚拟机部署
```

**答题时的立场**：不要一味说"能上"，也不要一味否定。给出**分级判断（核心/边缘、读多/写多、有无 operator、存储类型）**，这比任何单一答案都更像专家。

---

### Q41. 【通用】Pod 一直 `CrashLoopBackOff`，但 `kubectl logs` 什么都看不到，怎么办？ ⭐⭐⭐⭐

**考核点**：排障的"信息缺口"处理能力。

**解答**：
日志为空说明进程**根本没执行到"打印日志"那一步**，甚至在容器启动前就失败了。按顺序排查：

```
1) 看 exit code（关键线索）
   kubectl describe pod <p> | grep -E 'Last State|Exit Code|Reason'
   137 → OOMKilled / SIGKILL      139 → SIGSEGV
   126 → 权限/不可执行（noexec、权限位、架构不匹配 exec format error）
   127 → 命令不存在（ENTRYPOINT 路径错、动态链接库缺失）
   1   → 应用自身错误（但没打日志，可能写到了文件而非 stdout）
   0   → 主进程前台化失败（进程立即正常退出，容器就结束了）

2) 用 --previous 看上一个实例
   kubectl logs <p> --previous

3) 区分"init 容器失败"与"业务容器失败"
   kubectl describe pod 看 "Init Containers" 的 State
   kubectl logs <p> -c <init-container>

4) 覆盖 entrypoint 手动进去看
   kubectl run debug --image=<same-image> -it --restart=Never -- sh
   （若镜像无 shell，用 kubectl debug 加 ephemeral container）
   kubectl debug -it <p> --image=busybox --target=<container>

5) 检查"日志写到哪了"
   · 应用把日志写入文件（/var/log/app.log）→ runtime 读不到 → 应改 stdout
   · 日志被 buffer 住（Python 的 print 缓冲、Java 的 logback 异步）→ 崩溃时未 flush
   · 日志驱动配置错误（json-file 的 max-size 太小、或日志被轮转删掉）

6) 检查"根本不是容器问题"
   · ConfigMap/Secret 未挂载成功 → 启动脚本先失败
   · 卷挂载失败（subPath 不存在、只读卷被写）
   · 环境变量缺失导致程序在参数解析阶段退出（有些语言不打印）
   · 探针把容器杀了（liveness 太激进 → 启动中就重启 → 日志来不及写）
   
7) 抓现场
   · 给容器配 preStop/postStart 打点、或用 init 容器打印环境
   · 开启 core dump（需要 ulimits + core_pattern，注意容器内 core_pattern 是宿主全局）
   · strace（需要 CAP_SYS_PTRACE，且容器内通常无 strace，要 copy 进去）
```

**高分要点**：先判断"是容器没起来"还是"起来了立刻死"，前者查 runtime/配置，后者查 exit code + 应用。**日志为空本身就是最重要的信息**。

---

### Q42. 【通用】一个 Service 挂了 5000 个 Endpoint，会发生什么？如何治理？ ⭐⭐⭐⭐

**考核点**：规模化的第二阶问题（大多数人只知道第一阶）。

**解答**：
**发生的四件事**：
1. **Endpoints 对象巨大**：5000 个地址 + 端口 + 拓扑信息，单个对象可能几 MB → 每次变更（一个 Pod Ready/NotReady）都要**全量推送**给所有节点上的 kube-proxy/informer → apiserver 扇出风暴、网络与内存压力。
2. **kube-proxy 重写规则极慢**：iptables 模式下 5000 条 SEP 链 → 每次同步耗时数秒到数十秒，期间规则处于"半更新"状态（kube-proxy 用 iptables-restore 原子替换，但生成规则本身耗 CPU）→ **同步期间新端点不生效**。
3. **内存/CPU**：kube-proxy 与 apiserver 的 watch cache 内存暴涨（kube-proxy 会缓存整个 Endpoints 对象）。
4. **iptables 匹配耗时**：每个包遍历数千条规则 → 延迟上升、conntrack 压力大。
5. **客户端负载均衡退化**：如果客户端是 Headless + 自己轮询，5000 个 IP 的连接池管理成本极高。

**治理手段**：
| 手段 | 说明 |
|---|---|
| **EndpointSlice 分片** | 默认每片 100 个 → 单个变更只影响一片，扇出降低 50 倍。这是 K8s 官方的正解（1.21 GA） |
| **改用 ipvs / Cilium eBPF** | 查找 O(1)，规则更新是增删哈希项 |
| **`endpointSlice` 的 topology 感知** | 只给节点推送"本可用区"的端点（`topology-aware routing`），减少无效端点 |
| **限制单 Service 的规模** | 业务上做分片（按租户/地域拆多个 Service），而不是一个 Service 挂全量 |
| **用 NodePort/LB 的旁路** | 超大后端用"网关 → 少量固定地址"的两层结构，而不是让 Service 直接挂 5000 个 |
| **调整同步参数** | `kube-proxy --sync-period`、`--iptables-min-sync-period`，避免高频重建 |

**关键认知**：K8s 的 Service 抽象**适合"几十到几百个端点"**，上千端点就该考虑"分层"或"换数据面"。这是很多团队撞过的墙。

---

## 附录 0 高频速答卡（面试前 10 分钟过一遍）

| # | 问题 | 一句话答案 |
|---|---|---|
| 1 | 容器是什么 | 被 namespace 隔离、cgroup 限流、pivot_root 换根的一个（组）普通进程 |
| 2 | 和 VM 区别 | VM 隔离机器（独立内核），容器隔离视野（共享内核） |
| 3 | 8 种 namespace | mnt/uts/ipc/pid/net/user/cgroup/time |
| 4 | 容器内 PID 1 问题 | 内核不施加默认信号动作 + 负责收养孤儿 → 需 --init 或 exec 形式 entrypoint |
| 5 | cgroup v1 vs v2 | v1 多棵独立树易冲突；v2 单树 + subtree_control + no-internal-process + PSI |
| 6 | 为什么 v2 用 systemd driver | v2 是单层级，systemd 是树的真正持有者，cgroupfs 会打架 |
| 7 | exit 137 | 128+SIGKILL，最常见是 OOMKilled 或超时被强杀 |
| 8 | overlay2 三个目录 | lowerdir（镜像层只读）、upperdir（容器可写层）、merged（合并视图） |
| 9 | 为什么数据库不能写容器层 | copy-up 写放大 + 数据不持久 + fsync 语义差异 |
| 10 | docker run 的进程链 | docker CLI → dockerd → containerd → shim → runc → 容器进程 |
| 11 | 为什么需要 shim | daemon 重启不影响容器、接管 stdio 与退出码 |
| 12 | -p 映射原理 | iptables DNAT（+ docker-proxy 兜底回环），依赖 ip_forward 与 conntrack |
| 13 | Pod 为什么需要 pause | 持有 netns，使网络在业务容器重启时保持稳定 |
| 14 | Pod 内 localhost | 共享同一 netns，端口空间也共享（会冲突） |
| 15 | Service ClusterIP 为何 ping 不通 | 它是 netfilter 规则里的虚拟地址，不存在于任何网卡 |
| 16 | iptables vs ipvs | iptables O(n) 遍历，ipvs 哈希 O(1)，大规模必须 ipvs/eBPF |
| 17 | 为什么移 dockershim | CRI 让 kubelet 与运行时解耦，dockershim 是多余适配层，镜像仍兼容 |
| 18 | PLEG 是什么 | kubelet 每秒 relist 容器真实状态并与缓存比对，是"最终可信"的状态源 |
| 19 | Node 失联多久驱逐 Pod | 默认容忍 300s（tolerationSeconds），加上 40s 检测期 |
| 20 | HPA 为什么慢 | 指标滞后 + 调度 + 拉镜像 + 应用预热，总延迟 1~5 分钟 |
| 21 | Guaranteed 的意义 | requests==limits，oom_score_adj=-997，内核 OOM 时最后被杀 |
| 22 | RWO 的真实含义 | 单个**节点**可读写，不是单个 Pod |
| 23 | 优雅退出丢请求 | Endpoint 摘除与 SIGTERM 并发，需 preStop sleep 等传播 |
| 24 | etcd 慢的根因 | fsync 延迟（磁盘/网络存储），必须本地 NVMe + 独占机器 |
| 25 | NOSPACE 怎么办 | alarm disarm → compact → defrag（逐节点）→ 提高 quota + 减写入 |

---

## 附录 A 术语速查

| 术语 | 一句话解释 |
|---|---|
| namespace（ns） | 内核的资源视图隔离机制，容器隔离的基础 |
| cgroup | 内核的资源限制与统计框架，v2 为统一层级 |
| rootfs | 容器的根文件系统，由镜像层叠加而来 |
| pivot_root | 切换根目录并让旧根不可达，比 chroot 安全 |
| OverlayFS | 多层目录叠加 + CoW 的联合文件系统 |
| OCI | 容器镜像/运行时/分发的标准组织与规范 |
| CRI | K8s 与容器运行时的 gRPC 接口标准 |
| CNI | K8s 与网络插件的接口标准（ADD/DEL/CHECK） |
| CSI | K8s 与存储插件的接口标准 |
| shim | 托管容器进程的垫片，让 daemon 重启不影响容器 |
| pause / sandbox | Pod 内持有 namespace 的基础容器 |
| PLEG | kubelet 每秒轮询容器状态以生成事件的机制 |
| informer | 客户端 list-watch + 本地缓存 + 事件分发的组件 |
| EndpointSlice | Service 后端地址的分片对象，替代 Endpoints |
| QoS | Pod 的资源保障等级，影响调度与 OOM 优先级 |
| PV/PVC | 存储的"供给"与"需求"抽象 |
| PSA | Pod Security Admission，替代 PSP 的准入标准 |
| DRA | 动态资源分配，面向 GPU/FPGA 等设备的新模型 |

## 附录 B 命令速查

```bash
# ---- 容器层 ----
docker inspect -f '{{.State.Pid}} {{.State.OOMKilled}} {{.State.ExitCode}}' <cid>
nsenter -t <pid> -a                                  # 进容器所有 ns
ls -l /proc/<pid>/ns                                 # 看 namespace inode
cat /sys/fs/cgroup/<path>/memory.max cpu.max pids.max
docker stats --no-stream
docker system df -v

# ---- K8s 排障 ----
kubectl get pod -o wide -A | grep -v -E 'Running|Completed'
kubectl describe pod <pod> | sed -n '/Events/,$p'
kubectl logs <pod> -c <container> --previous --tail=200
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get endpointslice -o wide
kubectl get pvc,pv -o wide
kubectl describe node <node> | grep -A10 'Allocated resources'
kubectl top pod --sort-by=memory -A
kubectl auth can-i list pods --as=system:serviceaccount:ns:sa
kubectl rollout status deploy/<name> ; kubectl rollout undo deploy/<name>

# ---- 节点层 ----
crictl ps -a ; crictl pods ; crictl inspectp <sandbox>
crictl stats ; crictl info
journalctl -u kubelet -n 200 --no-pager
iptables -t nat -L KUBE-SERVICES -n -v | head -50
ipvsadm -Ln
etcdctl endpoint status --cluster -w table
etcdctl alarm list ; etcdctl defrag --cluster
```

## 附录 C 学习路径建议

```
第 1 周：跑通"手工造容器" —— 用 unshare/cgroup/overlay 手搭一个最小容器
         （做完这一步，namespace 与 cgroup 就再也不会混了）
第 2 周：Docker 深挖 —— 网络（自己抓包看 veth/iptables）、overlay2（dive 看层）、
         镜像优化（把一个大镜像瘦到 1/10）
第 3 周：K8s 主干 —— 手写 Deployment/Service/Ingress，用 kubectl describe 读事件，
         故意制造 Pending/CrashLoop/OOM 并定位
第 4 周：调度与控制器 —— 读一遍 scheduler 框架文档、手写一个简单 controller（client-go）
第 5 周：网络与存储 —— 用 Calico/Cilium 各搭一次，抓包对比 VXLAN 与 BGP 的差别
第 6 周：规模与治理 —— 压测 apiserver/etcd、看 workqueue 指标、做一次滚动发布演练
```

---

> **最后一句总结**：
> 容器的全部内容可以压缩成一句话 —— **"用 namespace 骗过进程的眼睛，用 cgroup 管住它的胃口，用 overlayfs 给它一个完整的家，用 capabilities/seccomp 锁住它的手。"**
> 而 Kubernetes 的全部内容可以压缩成另一句 —— **"让一群互不通信的组件，各自盯着同一个'期望状态'，把世界慢慢掰成那个样子。"**








