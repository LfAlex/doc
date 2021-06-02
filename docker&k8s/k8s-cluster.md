# kubeadm安装kubernetes

## 基本环境

实验环境：

| 主机名     | IP地址          | k8s版本 | docker version | flannel | 主机配置 |
| ---------- | --------------- | ------- | -------------- | ------- | -------- |
| k8s-master | 192.168.119.191 | v1.21.0 | 20.10.6        | v0.2.0  | 4G4核    |
| k8s-node1  | 192.168.119.192 | v1.21.0 | 20.10.6        | v0.2.0  | 4G4核    |
| k8s-node1  | 192.168.119.193 | v1.21.0 | 20.10.6        | v0.2.0  | 4G4核    |

开放端口：

Control-plane node(s)

| Protocol | Direction | Port Range | Purpose                 | Used By              |
| -------- | --------- | ---------- | ----------------------- | -------------------- |
| TCP      | Inbound   | 6443*      | Kubernetes API server   | All                  |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, Control plane  |
| TCP      | Inbound   | 10251      | kube-scheduler          | Self                 |
| TCP      | Inbound   | 10252      | kube-controller-manager | Self                 |


Worker node(s)

| Protocol | Direction | Port Range  | Purpose            | Used By             |
| -------- | --------- | ----------- | ------------------ | ------------------- |
| TCP      | Inbound   | 10250       | Kubelet API        | Self, Control plane |
| TCP      | Inbound   | 30000-32767 | NodePort Services† | All                 |

## 一、安装准备工作

### 1 配置阿里源：

阿里源链接：http://mirrors.aliyun.com/repo/

### 1.1 下载阿里云的repo

```
$ yum -y install wget vim
 
$ mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
 
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

--2020-06-28 21:23:11--  http://mirrors.aliyun.com/repo/Centos-7.repo
Resolving mirrors.aliyun.com (mirrors.aliyun.com)... 220.194.69.114, 119.167.169.226, 119.167.169.225, ...
Connecting to mirrors.aliyun.com (mirrors.aliyun.com)|220.194.69.114|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2523 (2.5K) [application/octet-stream]
Saving to: ‘/etc/yum.repos.d/CentOS-Base.repo’

100%[=================================================================>] 2,523       --.-K/s   in 0s      

2020-06-28 21:23:12 (242 MB/s) - ‘/etc/yum.repos.d/CentOS-Base.repo’ saved [2523/2523]
```

### 1.2 清除缓存并生成新的缓存

```
$ yum clean all && yum makecache
```

### 1.3 安装net-tools工具，运行ifconfig命令

```
$ yum install net-tools -y
```

## 二、环境配置

### 2.0 操作系统及硬件需求

- Ubuntu 16.04+
- Debian 9+
- CentOS 7
- Red Hat Enterprise Linux（RHEL）7
- Fedora 25+
- HypriotOS v1.0.1+
- Container Linux（tested with 1930.6.0）
- 每台计算机2GB或更多的RAM
- 2个更多CPU

### 2.1 配置主机名

```
$ hostnamectl set-hostname k8s-master
$ hostnamectl set-hostname k8s-node1
$ hostnamectl set-hostname k8s-node2
```

### 2.2 设置主机名互相解析，通过/etc/hosts文件

```
$ cat >> /etc/hosts << EOF
192.168.119.191 k8s-master
192.168.119.192 k8s-node1
192.168.119.193 k8s-node2
EOF
```

### 2.3 确保MAC地址唯一

```
$ cat /sys/class/net/ens33/address
00:0c:29:a5:5f:9e
$ cat /sys/class/net/ens33/address 
00:0c:29:45:f4:32
$ cat /sys/class/net/ens33/address 
00:0c:29:94:32:fd
```

### 2.4 确保product_uuid唯一

```
$ cat /sys/class/dmi/id/product_uuid
AC484D56-8A09-0B1D-20C2-8DBB53A55F9E
$ cat /sys/class/dmi/id/product_uuid
EBFF4D56-A998-0373-1D67-785D7D45F432
$ cat /sys/class/dmi/id/product_uuid
4F414D56-37F6-8A60-8368-9BAB069432FD
```

### 2.5 禁用swap

```
# 必须禁用swap功能才能使kubelet正常工作
# Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。如果开启了swap分区，kubelet会启动失败(可以通过将参数 --fail-swap-on 设置为false来忽略swap on)，故需要在每台机器上关闭swap分区

# 永久禁用
$ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
# 临时禁用
$ swapoff -a
```

### 2.6 关闭防火墙

```
# 未放行端口可关闭防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld
```

### 2.7 关闭selinux

```
# 永久关闭方法 – 需要重启服务器
$ sed -i 's/enforcing/disabled/' /etc/selinux/config
# 临时关闭方法 – 暂时可以不用重启服务器
$ setenforce 0
```

### 2.8 br_netfilter模块加载

```
# 本文的k8s网络使用flannel，该网络需要设置内核参数bridge-nf-call-iptables=1，修改这个参数需要系统有br_netfilter模块。

# 查看br_netfilter模块：
$ lsmod |grep br_netfilter
 
# 如果系统没有br_netfilter模块则执行下面的新增命令，如有则忽略
# 临时新增br_netfilter模块:
$ modprobe br_netfilter

$ lsmod |grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter
# 该方式重启后会失效

# 永久新增br_netfilter模块：
$ cat > /etc/rc.sysinit << EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF

$ cat > /etc/sysconfig/modules/br_netfilter.modules << EOF
modprobe br_netfilter
EOF

$ chmod 755 /etc/sysconfig/modules/br_netfilter.modules
```

### 2.9 参数内核参数

```
# 内核参数临时修改
$ sysctl net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
$ sysctl net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-ip6tables = 1

# 内核参数永久修改
$ cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

$ sysctl --system
# 忽略部分
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
* Applying /etc/sysctl.conf ...
```

## 三、Docker安装

### 3.1 安装依赖包

```
$ yum install -y yum-utils   device-mapper-persistent-data  lvm2
```

### 3.2 设置Docker源

```
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
```

### 3.3 安装Docker CE

#### 3.3.1 docker安装版本查看

```
$ yum list docker-ce --showduplicates | sort -r
```

#### 3.3.2 安装docker

```
$ yum install docker-ce docker-ce-cli containerd.io -y
```

### 3.4 启动Docker

```
$ systemctl start docker
$ systemctl enable docker
```

### 3.5 命令补全

#### 3.5.1 安装bash-completion

```
$ yum -y install bash-completion
```

#### 3.5.2 加载bash-completion

```
$ source /etc/profile.d/bash_completion.sh
```

### 3.6 镜像加速

#### 3.6.1 登陆阿里云

登陆地址为：[https://cr.console.aliyun.com](https://cr.console.aliyun.com/) ,未注册可以先注册阿里云账户容器模块

![img](C:\Users\ALEX\Desktop\learn\docker&k8s\371f703c2321216990b2408d10af2ca1.png)

#### 3.6.2 配置镜像加速器

```
# 配置daemon.json文件

$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://ogeydad1.mirror.aliyuncs.com"]
}
EOF
```

**重启服务**

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

### 3.7 验证

```
$ docker --version
Docker version 20.10.6, build 48a66213fe

$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:d58e752213a51915838f9eed2b7a498ffa1cb3aa7f946dda11af39286c3db9a9
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 3.8 修改Cgroup Driver

#### 3.8.1 修改daemon.json

修改daemon.json，新增"exec-opts": [“native.cgroupdriver=systemd”]

```
$ vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://qusw6gmy.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

#### 3.8.2 重新加载docker

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

## 四、安装kubeadm、kubelet and kubectl

kubelet 运行在集群所有节点上，用于启动Pod和容器等对象的工具

kubeadm 用于初始化集群，启动集群的命令工具

kubectl 用于和集群通信的命令行，通过kubectl可以部署和管理应用，查看各种资源，创建、删除和更新各种组件

### 4.1 安装阿里源

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 更新缓存
$ yum clean all
$ yum -y makecache
```

### 4.2 版本查看

```
$ yum list kubelet --showduplicates | sort -r
```

### 4.3 安装

```
$ yum install -y kubelet-1.21.0-0 kubeadm-1.21.0-0 kubectl-1.21.0-0
```

**安装的软件版本如下:**

![img](C:\Users\ALEX\Desktop\learn\docker&k8s\1f9a820a0e0861401e26f523354da1a3.png)

### 4.4 启动kubelet

```
# 启动kubelet并设置开机启动
$ systemctl enable kubelet && systemctl start kubelet
```

### 4.5 kubectl命令补全

```
$ echo "source <(kubectl completion bash)" >> ~/.bash_profile
$ source .bash_profile
```

### 4.6 下载镜像

**只需在k8s-maste节点操作**

#### 4.6.1 镜像下载的脚本

```
$ vi image.sh
#!/bin/bash
url=registry.cn-hangzhou.aliyuncs.com/google_containers
version=v1.21.0
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
```

#### 4.6.2 下载镜像

```
$ chmod 777 image.sh
$ ./image.sh
W0628 08:57:23.635193   20100 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
v1.21.0: Pulling from google_containers/kube-apiserver
597de8ba0c30: Pull complete 
d99dcd6b92d9: Pull complete 
Digest: sha256:7944dd5c67df9581ff5b1218f4d30c348b704a75cd0297ffc167f3b6113746b5
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver@sha256:7944dd5c67df9581ff5b1218f4d30c348b704a75cd0297ffc167f3b6113746b5
v1.21.0: Pulling from google_containers/kube-controller-manager
597de8ba0c30: Already exists 
def2b8f8694f: Pull complete 
Digest: sha256:0f57654686ddf834e8a7147ec8291efb9c64d6dd611f14410bb81438add0d077
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager@sha256:0f57654686ddf834e8a7147ec8291efb9c64d6dd611f14410bb81438add0d077
v1.21.0: Pulling from google_containers/kube-scheduler
597de8ba0c30: Already exists 
89ea45425d1b: Pull complete 
Digest: sha256:9f5a733905ff574f648e87d21f0879e1b6597b8f1cfd7381b2b808f7b60ddfa7
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler@sha256:9f5a733905ff574f648e87d21f0879e1b6597b8f1cfd7381b2b808f7b60ddfa7
v1.21.0: Pulling from google_containers/kube-proxy
597de8ba0c30: Already exists 
3f0663684f29: Pull complete 
e1f7f878905c: Pull complete 
3029977cf65d: Pull complete 
cc627398eeaa: Pull complete 
d3609306ce38: Pull complete 
79b420f95193: Pull complete 
Digest: sha256:340978f547e525df039e25afa4da0aefd0d25f4d5464ad482412ea9ea7e29a25
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy@sha256:340978f547e525df039e25afa4da0aefd0d25f4d5464ad482412ea9ea7e29a25
3.2: Pulling from google_containers/pause
c74f8866df09: Pull complete 
Digest: sha256:4a1c4b21597c1b4415bdbecb28a3296c6b5e23ca4f9feeb599860a1dac6a0108
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/pause@sha256:4a1c4b21597c1b4415bdbecb28a3296c6b5e23ca4f9feeb599860a1dac6a0108
3.4.3-0: Pulling from google_containers/etcd
39fafc05754f: Pull complete 
3736e1e115b8: Pull complete 
79de61f59f2e: Pull complete 
Digest: sha256:4198ba6f82f642dfd18ecf840ee37afb9df4b596f06eef20e44d0aec4ea27216
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/etcd@sha256:4198ba6f82f642dfd18ecf840ee37afb9df4b596f06eef20e44d0aec4ea27216
1.6.7: Pulling from google_containers/coredns
c6568d217a00: Pull complete 
ff0415ad7f19: Pull complete 
Digest: sha256:695a5e109604331f843d2c435f488bf3f239a88aec49112d452c1cbf87e88405
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.7
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.7
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.7
Untagged: registry.cn-hangzhou.aliyuncs.com/google_containers/coredns@sha256:695a5e109604331f843d2c435f488bf3f239a88aec49112d452c1cbf87e88405

$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.21.0             0d40868643c6        2 months ago        117MB
k8s.gcr.io/kube-scheduler            v1.21.0             a3099161e137        2 months ago        95.3MB
k8s.gcr.io/kube-apiserver            v1.21.0             6ed75ad404bd        2 months ago        173MB
k8s.gcr.io/kube-controller-manager   v1.21.0             ace0a8c17ba9        2 months ago        162MB
k8s.gcr.io/pause                     3.2                 80d28bedfe5d        4 months ago        683kB
k8s.gcr.io/coredns                   1.6.7               67da37a9a360        5 months ago        43.8MB
hello-world                          latest              bf756fb1ae65        5 months ago        13.3kB
k8s.gcr.io/etcd                      3.4.3-0             303ce5db0e90        8 months ago        288MB
```

#### 4.6.3 错误及解决

```
# 错误：[ERROR ImagePull]: failed to pull image registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0: output: Error response from daemon: pull access denied for registry.aliyuncs.com/google_containers/coredns/coredns, repository does not exist or may require ‘docker login’: denied: requested access to the resource is denied

# 解决：
$ docker pull coredns/coredns:1.8.0
$ docker tag coredns/coredns:1.8.0 registry.cn-hangzhou.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
# 或
$ docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
```

## 五、master节点自举集群

### 5.1 部署Kubernetes Master(只需在k8s-maste节点操作)

```
$ kubeadm init \
--apiserver-advertise-address=192.168.119.191 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--kubernetes-version v1.21.0 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
# 或
$ kubeadm init \
--apiserver-advertise-address=192.168.119.191 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.21.0 \
--pod-network-cidr=10.244.0.0/16 \
--token-ttl 0 \
--ignore-preflight-errors=Swap
```

### 5.2 初始化失败：

如果初始化失败，可执行**kubeadm reset**后重新初始化

```
$ kubeadm reset
$ rm -rf $HOME/.kube/config
```

### 5.3 输出信息

```
W0628 09:01:05.568405   20714 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.21.0
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 192.168.119.191]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.119.191 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.119.191 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0628 09:01:27.439519   20714 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0628 09:01:27.440621   20714 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.003658 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: eufass.9mj0z1oafwjwna8y
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.119.191:6443 --token eufass.9mj0z1oafwjwna8y \
    --discovery-token-ca-cert-hash sha256:1899cb7904899c7377fa01ca452d7260241f2f333f6dfff1bc843f338224ffb9 
```

### 5.4 加载环境变量

```
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
$ source .bash_profile
```

### 5.5 根据提示进行操作

```
$ mkdir -p $HOME/.kube
$ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5.6 查看集群状态

```
$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}
```

### 5.7 错误及解决

```
# 错误
$ kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+						# 警告
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
etcd-0               Healthy     {"health":"true"}

# 解决
## 注释掉port=0这一行
$ vim /etc/kubernetes/manifests/kube-controller-manager.yaml
$ vim /etc/kubernetes/manifests/kube-scheduler.yaml
```

### 5.8 确认个组件都处于healthy状态。集群初始化如果遇到问题，可以使用下面的命令进行清理：

```
$ kubeadm reset
$ ifconfig cni0 down
$ ip link delete cni0
$ ifconfig flannel.1 down
$ ip link delete flannel.1
$ rm -rf /var/lib/cni/
```

## 六、安装flannel网络

```
# 需要挂载VPN才能下载
# master01上新建flannel网络
# 可修改DNS服务器地址为8.8.8.8
$ yum provides nslookup
$ yum install -y bind-utils
$ vim /etc/resolv.conf
nameserver 8.8.8.8

$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

# 由于网络原因，可能会安装失败，可以直接下载kube-flannel.yml文件，然后再执行apply
```

**下载flannel网络**

```
# https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

$ vi kube-flannel.yml
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:

# 忽略部分
  
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg

$ kubectl apply -f kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```

**通过以下命令可以查看到：**

```
$ kubectl get ds -l app=flannel -n kube-system
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-flannel-ds-amd64     1         1         1       1            1           <none>          39s
kube-flannel-ds-arm       0         0         0       0            0           <none>          39s
kube-flannel-ds-arm64     0         0         0       0            0           <none>          39s
kube-flannel-ds-ppc64le   0         0         0       0            0           <none>          39s
kube-flannel-ds-s390x     0         0         0       0            0           <none>          39s
```

## 七、node加入集群

**将2个节点加入到集群中：**

```
# 在master节点查看集群的token值
$ kubeadm token create --print-join-command
W0628 09:12:59.371222   26258 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
kubeadm join 192.168.119.191:6443 --token jf0j4u.lieaijygwth7jcoo     --discovery-token-ca-cert-hash sha256:1899cb7904899c7377fa01ca452d7260241f2f333f6dfff1bc843f338224ffb9 

# 在另外两个节点执行
$ kubeadm join 192.168.119.191:6443 --token jf0j4u.lieaijygwth7jcoo     --discovery-token-ca-cert-hash sha256:1899cb7904899c7377fa01ca452d7260241f2f333f6dfff1bc843f338224ffb9

W0628 09:13:24.541566   21146 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

**查看加入情况：**

**成功安装后，可以查看集群情况**

```
# 查看加入情况
$ kubectl get node
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   9m13s   v1.21.0
k8s-node1    Ready    <none>   6m55s   v1.21.0
k8s-node2    Ready    <none>   6m52s   v1.21.0

$ kubectl get po -o wide -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE     IP                NODE         NOMINATED NODE   READINESS GATES
coredns-546565776c-j2rw7             1/1     Running   0          6m44s   10.244.0.2        k8s-master   <none>           <none>
coredns-546565776c-j5jq9             1/1     Running   0          6m44s   10.244.0.3        k8s-master   <none>           <none>
etcd-k8s-master                      1/1     Running   0          6m59s   192.168.119.191   k8s-master   <none>           <none>
kube-apiserver-k8s-master            1/1     Running   0          6m59s   192.168.119.191   k8s-master   <none>           <none>
kube-controller-manager-k8s-master   1/1     Running   0          6m59s   192.168.119.191   k8s-master   <none>           <none>
kube-flannel-ds-amd64-jckwp          1/1     Running   0          4m42s   192.168.119.193   k8s-node2    <none>           <none>
kube-flannel-ds-amd64-wztw8          1/1     Running   0          4m45s   192.168.119.192   k8s-node1    <none>           <none>
kube-flannel-ds-amd64-zc42k          1/1     Running   0          5m20s   192.168.119.191   k8s-master   <none>           <none>
kube-proxy-dzvbd                     1/1     Running   0          4m42s   192.168.119.193   k8s-node2    <none>           <none>
kube-proxy-fgjl2                     1/1     Running   0          6m45s   192.168.119.191   k8s-master   <none>           <none>
kube-proxy-nj824                     1/1     Running   0          4m45s   192.168.119.192   k8s-node1    <none>           <none>
kube-scheduler-k8s-master            1/1     Running   0          6m59s   192.168.119.191   k8s-master   <none>           <none>
```

**如果想将节点移除集群：**

- 在master节点操作

```
$ kubectl drain k8s-node2 --delete-local-data --force --ignore-daemonsets
node/k8s-node2 already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/kube-flannel-ds-amd64-7qx8x, kube-system/kube-proxy-8fzpb
node/k8s-node2 drained

$ kubectl delete node k8s-node2
node "k8s-node2" deleted

$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   56m   v1.21.0
k8s-node1    Ready    <none>   44m   v1.21.0
```

- 在node上操作：

```
# 重置，提醒：如果你使用的是外部etcd，你需要手动删除etcd数据，这意味着如果使用相同的etcd端点再次运行kubeadm init，您将看到先前集群的状态。

$ kubeadm reset
$ ifconfig cni0 down
$ ip link delete cni0
$ ifconfig flannel.1 down
$ ip link delete flannel.1
$ rm -rf /var/lib/cni/
```

## 八、测试集群

### 8.1 查看集群状态

```
# master节点查看node状态
$ kubectl get node
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   9m13s   v1.21.0
k8s-node1    Ready    <none>   6m55s   v1.21.0
k8s-node2    Ready    <none>   6m52s   v1.21.0

# 测试集群是否正常

$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

# 创建service
$ kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed

$ kubectl get pods,svc
NAME                         READY   STATUS              RESTARTS   AGE
pod/nginx-6799fc88d8-wp9db   0/1     ContainerCreating   0          19s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP        31m
service/nginx        NodePort    10.1.245.2   <none>        80:31532/TCP   6s

# 测试nginx
# 通过任意一个nodeip+端口 ，既可以访问到nginx页面
```

课件nginx-ds服务信息如下：

- Service Cluster IP：10.1.245.2
- 服务端口：80
- NodePort端口：31532

### 8.2 访问CLUSTER-IP：10.1.30.18即可访问到nginx访问

```
$ curl 10.1.245.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 8.3 在集群外面访问

![img](C:\Users\ALEX\Desktop\learn\docker&k8s\f18d48cf23c912f7bca948af9cebc05e.png)

## 九、安装Dashboard UI

### 9.1下载yaml

```
# 需挂载VPN，如果连接超时，可以多试几次。
$ wget -c https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

### 9.2 配置yaml

#### 9.2.1 外网访问

```
$ sed -i '/targetPort: 8443/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' recommended.yaml

# 配置NodePort，外部通过https://NodeIp:NodePort 访问Dashboard，此时端口为30001
```

#### 9.2.2 新增管理员帐号

```
$ cat >> recommended.yaml << EOF
---
# ------------------- dashboard-admin ------------------- #
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
 
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
EOF

# 创建超级管理员的账号用于登录Dashboard
```

### 9.3. 部署访问

#### 9.3.1 部署Dashboard

```
$ kubectl apply -f recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
serviceaccount/dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created

# 警告（非报错）
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
```

#### 9.3.2 状态查看

```
$ kubectl get all -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-778b77d469-pc5gr   1/1     Running   1          39m
pod/kubernetes-dashboard-5cd89984f5-xzwf7        1/1     Running   0          39m

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.1.209.66    <none>        8000/TCP        39m
service/kubernetes-dashboard        NodePort    10.1.213.240   <none>        443:30001/TCP   39m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           39m
deployment.apps/kubernetes-dashboard        1/1     1            1           39m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-778b77d469   1         1         1       39m
replicaset.apps/kubernetes-dashboard-5cd89984f5        1         1         1       39m
```

#### 9.3.3 令牌查看

```
$ kubectl describe secrets -n kubernetes-dashboard
Name:         dashboard-admin-token-tjp46
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 2434f0bc-f8db-4f2a-aee0-10fedf87de49

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkxJMEpqeDBmeGViUER4Y1hhVVBmb1dOcWJKWExTc3I5OUdZLWkwZUlqOVEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdGpwNDYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMjQzNGYwYmMtZjhkYi00ZjJhLWFlZTAtMTBmZWRmODdkZTQ5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.oDQieiP5_7zHyA_VYw9SQMT6dRD5rkLYcflh0d4riIMD6_86QGhDHUOj2UfFM1VbmIpTJ0cJcYuvUQzg1L174Nwq4-JZ5CvK_L0A00ghmZAIfsqpgeDt52eZpu8ghCsqT6UKX5x3lD0wiNboYZCfNGadxnJ2-kWq2DlgD7K2BvNUB9uj6cMq-vPzGPnIt5oId1Ley7wFNmKu5tU2IsiUN3nGpVCV_FGRGqjuEmHLxaLmFkWVnLaxpab45P621J8t8yPGWmUVoZSzqIqs3dQBuBBL-ykwR-mEqUwr-2HzMtKtBa_qjBa5-xGe8VgDnuI0SWX_MWQ2hgzfxGkNRmlg8A
```

3.4 登录dashboard UI
使用浏览器（推荐：Firefox）访问任意节点都能打开，格式为：https://节点中任意IP地址:30001

https://192.168.119.191:30001

https://192.168.119.192:30001

https://192.168.119.193:30001

Chrome浏览器需要进行以下设置

```
mkdir key && cd key
#生成证书
openssl genrsa -out dashboard.key 2048 
openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=主节点IP'
openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 
#删除原有的证书secret
kubectl delete secret kubernetes-dashboard-certs -n kube-system
#创建新的证书secret
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kube-system
#查看pod
kubectl get pod -n kube-system
#重启pod
kubectl delete pod <pod name> -n kube-system
```

![img](C:\Users\ALEX\Desktop\learn\docker&k8s\964e905a13ecb7cf1943ba7550ab3900.png)

#### 3.3.4 可以看到集群中的情况

![img](C:\Users\ALEX\Desktop\learn\docker&k8s\4b0bca530c8e38df7cc4142fade2ccb4.png)