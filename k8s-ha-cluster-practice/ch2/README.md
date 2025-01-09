# k8s master 节点安装

## 概述

说明 k8s 大致架构，以及 master worker 的区别，说明本文着重实践 k8s master 节点的安装过程，同时说明基础的虚拟机集群可参考上一篇文章。

## 环境信息

说明基于上一篇文章的 3+1 4 节点集群规划方案，配置信息列出、操作系统版本等。计划安装的 k8s 版本。

| 主机名      | 配置 | 角色 | 系统版本 |IP |安装的组件 |
| ----------- | ----------- | ----------- |----------- |----------- |----------- |
| master1      | 2C2G40G       | master |ubuntu22.04 |192.168.33.11 |apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| master2      | 2C2G40G       | master |ubuntu22.04 |192.168.33.12 |apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| master3      | 2C2G40G       | master |ubuntu22.04 |192.168.33.13 |apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| node1      | 2C2G40G       | worker |ubuntu22.04 |192.168.33.14 |kubelet、kube-proxy、容器运行时、calico、coredns |

上图

## master 节点安装

说明性信息

### 步骤1：系统初始化配置

- 配置 hosts
  ```shell
  $ cat >> /etc/hosts << EOF
  192.168.33.11 master1 
  192.168.33.12 master2 
  192.168.33.13 master3 
  192.168.33.14 node1
  EOF
  ```
  **注：** hosts 信息根据自身实际情况调整 ip 及对应的主机名。

- 关闭系统的 swap 分区
  ```shell
  $ sed -ri 's/^([^#].*swap.*)$/#\1/' /etc/fstab && grep swap /etc/fstab && swapoff -a && free -h
  ```

- 设置内核参数
  ```shell
  $ cat >> /etc/sysctl.conf <<EOF
  vm.swappiness = 0
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  EOF

  $ cat >> /etc/modules-load.d/neutron.conf <<EOF
  br_netfilter
  EOF

  #加载模块
  $ modprobe  br_netfilter
  #让配置生效
  $ sysctl -p
  ```
  - `vm.swappiness = 0` 表示尽可能不使用交换分区，优先使用物理内存
  - `net.bridge.bridge-nf-call-iptables = 1` 允许 iptables 规则应用于网桥流量
  - `net.ipv4.ip_forward = 1` 允许系统作为路由器转发 IPv4 数据包
  - `net.bridge.bridge-nf-call-ip6tables = 1` 允许 ip6tables 规则应用于网桥流量
  - `br_netfilter` 用于在网桥上启用 Netfilter 功能，支持 iptables 和 ip6tables 规则

### 步骤2：安装 docker

我们选择常用的 docker 作为容器运行时，当然你也可以选择其它主流选择，比如 podman。我之前的一些文章对他们有一些讲解，可供参考。

- 更新包管理器索引，确保获取最新的软件包信息：
  
  ```shell
  $ apt update
  ```

- 安装必要的工具和依赖：

  ```shell

  $ apt install -y ca-certificates curl gnupg lsb-release
  ```
  - ca-certificates：用于管理 CA 证书
  - curl：用于从网络下载文件
  - gnupg：用于管理 GPG 密钥
  - lsb-release：用于获取系统发行版信息

- 下载 Docker 的官方 GPG 密钥，并将其转换为适用于 apt 的格式

  ```shell
  
  $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  ```
  - -fsSL：静默模式下载，跟随重定向，显示错误信息
  - gpg --dearmor：将 GPG 密钥转换为二进制格式
  - 输出到 /usr/share/keyrings/docker-archive-keyring.gpg

- 添加 Docker 的官方 APT 源：

  ```shell
  
  $ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

  # 再次更新包管理器索引，以加载新添加的 Docker 源
  $ apt-get update
  ```
  - [arch=$(dpkg --print-architecture)]：自动检测系统架构（如 amd64、arm64）
  - signed-by=/usr/share/keyrings/docker-archive-keyring.gpg：指定 GPG 密钥路径
  - $(lsb_release -cs)：获取系统发行版代号（如 focal、jammy）
  - 将源配置写入 /etc/apt/sources.list.d/docker.list

- 安装 Docker 及相关组件：

  ```shell
  $ apt install docker-ce docker-ce-cli containerd.io docker-compose -y
  ```
  - docker-ce：Docker 社区版
  - docker-ce-cli：Docker 命令行工具
  - containerd.io：容器运行时
  - docker-compose：Docker 容器编排工具

- 创建 Docker 的配置文件 /etc/docker/daemon.json，并写入以下内容：
 
  ```shell

  $ cat > /etc/docker/daemon.json <<EOF
  {
    "registry-mirrors": [
      "https://docker.mirrors.ustc.edu.cn",
      "https://hub-mirror.c.163.com",
      "https://reg-mirror.qiniu.com",
      "https://registry.docker-cn.com"
    ],
    
    "exec-opts": ["native.cgroupdriver=systemd"],
    "data-root": "/data/docker",
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "20m",  
      "max-file": "5"     
    }
  }
  EOF
  ```
  - `registry-mirrors` 配置 Docker 镜像加速器（国内镜像源）
  - `exec-opts` 设置 Cgroup 驱动为 systemd（适用于使用 systemd 的系统）
  - `data-root` 设置 Docker 数据存储路径为 /data/docker
  - `log-driver` 配置日志驱动为 json-file
  - `log-opts` 配置日志文件的大小和数量限制

- 重启 Docker 服务，使配置生效

  ```shell
  $ systemctl restart docker.service
  ```

- 设置 Docker 服务开机自启

  ```shell
  $ systemctl enable docker.service
  ```

- 查看 Docker 系统信息，验证安装和配置是否成功

  ```shell
  docker info
  ```


## 其它

```shell
vagrant snapshot save [机器名] <snapshot_name>

vagrant snapshot list [机器名]

vagrant snapshot restore [机器名] <snapshot_name>

vagrant snapshot delete [机器名] <snapshot_name>
```