# k8s master 节点安装

## 概述

说明 k8s 大致架构，以及 master worker 的区别，说明本文着重实践 k8s master 节点的安装过程，同时说明基础的虚拟机集群可参考上一篇文章。

## 环境信息

说明基于上一篇文章的 3+1 4 节点集群规划方案，配置信息列出、操作系统版本等。计划安装的 k8s 版本。

| 主机名      | 配置 | 角色 | 系统版本 |IP |安装的组件 |
| ----------- | ----------- | ----------- |----------- |----------- |----------- |
| master1      | 2C2G40G       | master |ubuntu22.04 |192.168.33.11 |kube-vip、apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| master2      | 2C2G40G       | master |ubuntu22.04 |192.168.33.12 |kube-vip、apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| master3      | 2C2G40G       | master |ubuntu22.04 |192.168.33.13 |kube-vip、apiserver、controller-manager、scheduler、kubelet、etcd、kube-proxy、容器运行时、calico |
| node1      | 2C2G40G       | worker |ubuntu22.04 |192.168.33.14 |kubelet、kube-proxy、容器运行时、calico、coredns |

上图 + 参考之前的 vip 文章
https://blog.csdn.net/networken/article/details/132594119


> [kube-vip](https://kube-vip.io/docs/ "kube-vip") 为 Kubernetes 集群提供虚拟 IP 和负载均衡器，用于控制平面（用于构建高可用集群）并支持 Kubernetes LoadBalancer 类型 Services ，无需依赖任何外部硬件或软件。当然你也可以选择 keepalived+haproxy 方式，我选择部署比较方便的 kube-vip。



## 安装

说明性信息

### 步骤1：系统初始化配置

> 说明： 以下操作在所有节点执行。


- 配置 hosts：
  ```shell
  $ cat >> /etc/hosts << EOF
  192.168.33.250 lb.k8s.local
  192.168.33.11 master1 
  192.168.33.12 master2 
  192.168.33.13 master3 
  192.168.33.14 node1
  EOF
  ```
  **注：** hosts 信息根据自身实际情况调整 ip 及对应的主机名。`lb.k8s.local` 是后续 kube-vip 用到的虚拟 ip，按需配置。

- 关闭系统的 swap 分区：
  ```shell
  $ sed -ri 's/^([^#].*swap.*)$/#\1/' /etc/fstab && grep swap /etc/fstab && swapoff -a && free -h
  ```

- 设置内核参数：
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


> 说明： 以下操作在所有节点执行。

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

- 下载 Docker 的官方 GPG 密钥，并将其转换为适用于 apt 的格式：

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

- 重启 Docker 服务，使配置生效：

  ```shell
  $ systemctl restart docker.service
  ```

- 设置 Docker 服务开机自启：

  ```shell
  $ systemctl enable docker.service
  ```

- 查看 Docker 系统信息，验证安装和配置是否成功：

  ```shell
  docker info
  ```

### 步骤3：安装最新版本的 kubeadm、kubelet、kubectl

> 说明： 以下操作在所有节点执行。

- 配置安装源：
  ```shell
  $ apt-get update && apt-get install -y apt-transport-https
  $ curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.32/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  $ echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.32/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
  ```
  **注**：我目前安装的是 1.32 ,如果你需要安装其他版本，替换版本号即可。
  - `apt-get update` 更新本地包索引，确保获取最新的软件包信息。
  -  `apt-get install -y apt-transport-https` 安装 apt-transport-https 包，使 apt 能够通过 HTTPS 协议访问软件源。
  - `curl -fsSL` 静默模式下载，跟随重定向，显示错误信息。
  - `gpg --dearmor` 将 GPG 密钥转换为二进制格式。
  - `-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg` 将转换后的密钥保存到指定路径。
  - `[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg]` 指定 GPG 密钥路径，用于验证软件包的签名。
  - `https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.32/deb/` 阿里云的 Kubernetes 软件源地址。

- 安装 kubelet、kubeadm、kubectl：
  ```shell
  $ apt-get update
  $ apt-get install -y kubelet kubeadm kubectl
  ```
- 确认安装版本：
  ```shell
  $ kubectl version
  Client Version: v1.32.0
  Kustomize Version: v5.5.0

  $ kubeadm version
  kubeadm version: &version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.0", GitCommit:"70d3cc986aa8221cd1dfb1121852688902d3bf53", GitTreeState:"clean", BuildDate:"2024-12-11T18:04:20Z", GoVersion:"go1.23.3", Compiler:"gc", Platform:"linux/amd64"}

  $ kubelet --version
  Kubernetes v1.32.0
  ```

- 设置 kubelet 自启动：
  ```shell
  $ systemctl enable kubelet
  ```
  **注**：此时，还不能启动 kubelet，因为集群还没配置，仅仅设置开机自启动。

### 步骤4：Docker 垫片安装 （可选）

> 说明： 以下操作在所有节点执行。**注**：非 docker 容器运行时可跳过本步骤。



> 自 Kubernetes v1.24 版本起，移除了对 Docker Shim 的支持，而 Docker Engine 默认并不兼容 CRI（容器运行时接口）规范，导致两者无法直接集成。为了解决这一问题，Mirantis 和 Docker 共同开发了 **cri-dockerd** 项目。该项目为 Docker Engine 提供了一个支持 CRI 规范的适配层，使得 Kubernetes 能够通过 CRI 接口管理和控制 Docker。

[cri-dockerd 项目](https://github.com/Mirantis/cri-dockerd "cri-dockerd 项目") 提供了 RPM 包，可在 github 仓库的 release 界面获取：

![](images/cri-dockerd-pkg.png)

> Ubuntu的版本分别代表不同的大版本
> 
> 24.04：Noble
> 
> 22.04：jammy
> 
> 20.04：focal
> 
> 18.04：bionic
> 
> 16.04：xenial
> 
> 14.04：trusty

我用的是 22.04 版本，所以选择 jammy。

- 安装 cri-dockerd:
  ```shell
  $ wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.16/cri-dockerd_0.3.16.3-0.ubuntu-jammy_amd64.deb
  $ dpkg -i ./cri-dockerd_0.3.16.3-0.ubuntu-jammy_amd64.deb
  ```
  **注**：无法访问 github 的同学，可以至文末参考链接中的 gitee 项目中获取。

- 配置 cri-dockerd:
  因为国内无法下载 k8s.gcr.io 的仓库镜像，所以需要修改 cri-dockerd 使用国内镜像源，设置国内源方法如下：
  ```shell
  $ sed -ri 's@^(.*fd://).*$@\1 --pod-infra-container-image registry.aliyuncs.com/google_containers/pause@' /usr/lib/systemd/system/cri-docker.service

  # 重启
  $ systemctl daemon-reload && systemctl restart cri-docker && systemctl enable cri-docker
  ```
  - `sed -ri` 命令修改cri-docker.service 文件涉及镜像的一行内容为 `ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image registry.aliyuncs.com/google_containers/pause`

### 步骤5：生成 kube-vip 配置文件
> 说明：以下操作在第一个 master 节点执行即可。

- 生成 kube-vip 静态 yaml 配置文件：
  ```shell
  $ export VIP=192.168.33.250
  $ export INTERFACE=enp0s8
  $ export KVVERSION=v0.8.8
  $ docker run --network host --rm registry.cn-shanghai.aliyuncs.com/yydd/kube-vip:$KVVERSION manifest pod \
      --interface $INTERFACE \
      --address $VIP \
      --controlplane \
      --services \
      --arp \
      --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml
  ```
  - `export VIP=192.168.33.250` 用于设置 vip 的地址，主要要与节点 ip 在同一子网。和第一步中设置的 lb 主机名的 ip 要一致。
  - `export INTERFACE=enp0s8` 虚拟ip 绑定的网卡，通过 `ip a` 可查看网卡清单，此处我们选择 基于 vagrant 创建的第二块卡，你需要根据实际来调整。
  - `export KVVERSION=v0.8.8` kube-vip 的版本，通过 `https://api.github.com/repos/kube-vip/kube-vip/releases` 可查看版本信息。
  - `docker run` 命令会基于 kube-vip 镜像加上你的参数，生成最终的一个用于后续部署的 manifest 文件，注意输出目录是`/etc/kubernetes/manifests`,此目录不可调整，这是 k8s static pod 目录，后续 init 时会自动创建服务。
  - `kube-vip:v0.8.8` 镜像我已推送到阿里云仓库，直接使用即可。
  - `manifest pod ...` 是 kube-vip 容器运行的指令，用于生成一个 kubernetes pod 的清单文件，该清单文件用于运行 Kube-Vip 的Pod。
  - `--controlplane` 启用 kube-vip 控制平面功能。
  - `--services` 使 kube-vip 能够监视 LoadBalancer 类型的Service。
  - `--arp` 启用 ARP（地址解析协议）广播，用于让其他网络设备能够知道虚拟 IP 地址。这对于确保网络中的其他机器能够正确解析虚拟 IP 地址并路由流量至 Kube-Vip 是必要的。
  - `--leaderElection` 启用 Kubernetes LeaderElection,由 ARP 使用，因为只有领导者可以广播,领导者选举机制，确保集群中只有一个 Kube-Vip 实例在任何时刻管理虚拟 IP。

- 修改 `kube-vip.yaml` 镜像为阿里云镜像：
  ```shell
  $ sed -i 's|ghcr.io/kube-vip/kube-vip:v0.8.8|registry.cn-shanghai.aliyuncs.com/yydd/kube-vip:v0.8.8|' /etc/kubernetes/manifests/kube-vip.yaml
  ```
- 复制 `kube-vip.yaml` 到其他两个 master 节点：

  ```shell 
  $ scp -r /etc/kubernetes/manifests/kube-vip.yaml 192.168.33.12:/etc/kubernetes/manifests/
  $ scp -r /etc/kubernetes/manifests/kube-vip.yaml 192.168.33.13:/etc/kubernetes/manifests/

  ```
- 修改 `kube-vip.yaml` 中挂载 k8s 配置文件为 super-admin.conf:

  ```shell
  $ sed -i 's|path: /etc/kubernetes/admin.conf|path: /etc/kubernetes/super-admin.conf|' /etc/kubernetes/manifests/kube-vip.yaml
  ```
  **注**：因为没有更改这个配置，kube-vip 一直起不来，报错没有权限获取 k8s 的某些资源，通过查看 issue 才发现 k8s1.29 之后，需要在第一个主节点设置挂载 super-admin.conf(也只有第一个初始化主节点才有这个配置文件)。

### 步骤6：开始初始化
> 说明：以下操作在第一个 master 节点执行即可。

- 生成初始化配置文件：
  ```shell
  $ kubeadm config print init-defaults > kubeadm.yaml
  ```

- 修改上一步生成的`kubeadm.yaml`配置文件：
  ```yaml
  apiVersion: kubeadm.k8s.io/v1beta4
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint:
    # 修改成当前执行操作的 master ip
    advertiseAddress: 192.168.33.11
    bindPort: 6443
  nodeRegistration:
    # 修改成 cri-dockerd 的 sock
    criSocket: unix:///run/cri-dockerd.sock
    imagePullPolicy: IfNotPresent
    imagePullSerial: true
    # 修改成当前 master 的主机名
    name: master1
    taints: null
  timeouts:
    controlPlaneComponentHealthCheck: 4m0s
    discovery: 5m0s
    etcdAPICall: 2m0s
    kubeletHealthCheck: 4m0s
    kubernetesAPICall: 1m0s
    tlsBootstrap: 5m0s
    upgradeManifests: 5m0s
  ---
  apiServer: {}
  apiVersion: kubeadm.k8s.io/v1beta4
  caCertificateValidityPeriod: 87600h0m0s
  certificateValidityPeriod: 8760h0m0s
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  controllerManager: {}
  dns: {}
  encryptionAlgorithm: RSA-2048
  etcd:
    local:
      # 按需修改 etcd 的数据目录
      dataDir: /data/etcd
  # 修改镜像加速地址
  imageRepository: registry.aliyuncs.com/google_containers
  kind: ClusterConfiguration
  # 修改成具体对应的版本号
  kubernetesVersion: 1.32.0
  # 如果是多master节点，就需要添加这项，指向第一步设定的 vip 的hosts
  controlPlaneEndpoint: "lb.k8s.local:6443"
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12
    # 添加pod的IP地址设置（按需调整）
    podSubnet: 10.244.0.0/16
  proxy: {}
  scheduler: {}
  # 在最后添加上下面两部分
  ---
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: ipvs
  ---
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  cgroupDriver: systemd
  ```

- 基于上一步修改完成的 `kubeadm.yaml`文件，执行集群初始化：
  ```shell
  $ kubeadm init --config=kubeadm.yaml
  ```
- 等待出现类似以下的输出，代表初始化成功：
  ```shell
  root@master1:~# kubeadm init --config=kubeadm.yaml
  [init] Using Kubernetes version: v1.32.0
  [preflight] Running pre-flight checks
  [preflight] Pulling images required for setting up a Kubernetes cluster
  [preflight] This might take a minute or two, depending on the speed of your internet connection
  [preflight] You can also perform this action beforehand using 'kubeadm config images pull'
  W0111 15:02:22.808751    2723 checks.go:846] detected that the sandbox image "registry.aliyuncs.com/google_containers/pause" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.aliyuncs.com/google_containers/pause:3.10" as the CRI sandbox image.
  [certs] Using certificateDir folder "/etc/kubernetes/pki"
  [certs] Generating "ca" certificate and key
  [certs] Generating "apiserver" certificate and key
  [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local lb.k8s.local master1] and IPs [10.96.0.1 192.168.33.11]
  [certs] Generating "apiserver-kubelet-client" certificate and key
  [certs] Generating "front-proxy-ca" certificate and key
  [certs] Generating "front-proxy-client" certificate and key
  [certs] Generating "etcd/ca" certificate and key
  [certs] Generating "etcd/server" certificate and key
  [certs] etcd/server serving cert is signed for DNS names [localhost master1] and IPs [192.168.33.11 127.0.0.1 ::1]
  [certs] Generating "etcd/peer" certificate and key
  [certs] etcd/peer serving cert is signed for DNS names [localhost master1] and IPs [192.168.33.11 127.0.0.1 ::1]
  [certs] Generating "etcd/healthcheck-client" certificate and key
  [certs] Generating "apiserver-etcd-client" certificate and key
  [certs] Generating "sa" key and public key
  [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
  [kubeconfig] Writing "admin.conf" kubeconfig file
  [kubeconfig] Writing "super-admin.conf" kubeconfig file
  [kubeconfig] Writing "kubelet.conf" kubeconfig file
  [kubeconfig] Writing "controller-manager.conf" kubeconfig file
  [kubeconfig] Writing "scheduler.conf" kubeconfig file
  [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
  [control-plane] Using manifest folder "/etc/kubernetes/manifests"
  [control-plane] Creating static Pod manifest for "kube-apiserver"
  [control-plane] Creating static Pod manifest for "kube-controller-manager"
  [control-plane] Creating static Pod manifest for "kube-scheduler"
  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
  [kubelet-start] Starting the kubelet
  [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
  [kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
  [kubelet-check] The kubelet is healthy after 1.003468522s
  [api-check] Waiting for a healthy API server. This can take up to 4m0s
  [api-check] The API server is healthy after 12.126753636s
  [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
  [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
  [upload-certs] Skipping phase. Please see --upload-certs
  [mark-control-plane] Marking the node master1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
  [mark-control-plane] Marking the node master1 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
  [bootstrap-token] Using token: abcdef.0123456789abcdef
  [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
  [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
  [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
  [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
  [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
  [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
  [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
  [addons] Applied essential addon: CoreDNS
  [addons] Applied essential addon: kube-proxy

  Your Kubernetes control-plane has initialized successfully!

  To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

  Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

  You can now join any number of control-plane nodes by copying certificate authorities
  and service account keys on each node and then running the following as root:

    kubeadm join lb.k8s.local:6443 --token abcdef.0123456789abcdef \
          --discovery-token-ca-cert-hash sha256:f3b88b3b85946768087b3642a373ec891bb1da2fc79cf13b82ce4a9b1b13db14 \
          --control-plane 

  Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join lb.k8s.local:6443 --token abcdef.0123456789abcdef \
          --discovery-token-ca-cert-hash sha256:f3b88b3b85946768087b3642a373ec891bb1da2fc79cf13b82ce4a9b1b13db14 
  ```
  **注：** 最后的两个关键的输出，分别指出了新的 master 以及 worker 节点加入集群的命令。
- 按照以上输出，创建配置文件目录并复制配置文件：

  ```shell
  $ mkdir -p $HOME/.kube
  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
- 此时，通过执行 `ip a` 可以发现 vip 已经在第一个节点生效：

  ```shell
  root@master1:~# ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host 
        valid_lft forever preferred_lft forever
  2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 02:3e:24:fa:6f:1a brd ff:ff:ff:ff:ff:ff
      inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
        valid_lft 85677sec preferred_lft 85677sec
      inet6 fd00::3e:24ff:fefa:6f1a/64 scope global dynamic mngtmpaddr noprefixroute 
        valid_lft 86209sec preferred_lft 14209sec
      inet6 fe80::3e:24ff:fefa:6f1a/64 scope link 
        valid_lft forever preferred_lft forever
  3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:51:5f:7e brd ff:ff:ff:ff:ff:ff
      inet 192.168.33.11/24 brd 192.168.33.255 scope global enp0s8
        valid_lft forever preferred_lft forever
      inet 192.168.33.250/32 scope global enp0s8
        valid_lft forever preferred_lft forever
      inet6 fe80::a00:27ff:fe51:5f7e/64 scope link 
        valid_lft forever preferred_lft forever
  4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
      link/ether 02:42:68:e4:92:3e brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
        valid_lft forever preferred_lft forever
  5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
      link/ether 22:ec:0f:67:18:c7 brd ff:ff:ff:ff:ff:ff
      inet 10.96.0.1/32 scope global kube-ipvs0
        valid_lft forever preferred_lft forever
      inet 10.96.0.10/32 scope global kube-ipvs0
        valid_lft forever preferred_lft forever
  ```
- 查看节点状态，当前因未安装网络插件，节点处于 NotReady 状态：

  ```shell
  root@master1:~# kubectl get node
  NAME      STATUS     ROLES           AGE     VERSION
  master1   NotReady   control-plane   9m38s   v1.32.0
  ```
- 查看 Pod 状态，当前因未安装网络插件， coredns pod 处于 Pending 状态：

  ```shell
  root@master1:~# kubectl get pod -A
  NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
  kube-system   coredns-6766b7b6bb-f2shx          0/1     Pending   0          10m
  kube-system   coredns-6766b7b6bb-vsl65          0/1     Pending   0          10m
  kube-system   etcd-master1                      1/1     Running   0          10m
  kube-system   kube-apiserver-master1            1/1     Running   0          10m
  kube-system   kube-controller-manager-master1   1/1     Running   0          10m
  kube-system   kube-proxy-ldsp7                  1/1     Running   0          10m
  kube-system   kube-scheduler-master1            1/1     Running   0          10m
  kube-system   kube-vip-master1                  1/1     Running   0          10m
  ```


### 步骤7：安装 Calico 网络插件

> 说明：以下操作在第一个 master 节点执行即可。

- 下载 calico 清单文件：

  ```shell
  $ wget https://docs.projectcalico.org/manifests/calico.yaml
  ```
- 无法拉取 docker 官方仓库镜像的同学，替换阿里云镜像：

  ```shell
  $ sed -i 's|docker.io/calico/cni:v3.25.0|registry.cn-shanghai.aliyuncs.com/yydd/calico:cni_v3.25.0|' calico.yaml
  $ sed -i 's|docker.io/calico/node:v3.25.0|registry.cn-shanghai.aliyuncs.com/yydd/calico:node_v3.25.0|' calico.yaml
  $ sed -i 's|docker.io/calico/kube-controllers:v3.25.0|registry.cn-shanghai.aliyuncs.com/yydd/calico:kube-controllers_v3.25.0|' calico.yaml
  ```
- 因为 vagrant 创建有多网卡，需要指定网卡信息：
  
  ```shell
  $ sed -i '/name: CALICO_IPV4POOL_VXLAN/i \
              - name: IP_AUTODETECTION_METHOD\n              value: "interface=enp0s8"' calico.yaml

  ```
  **注**：网卡根据实际调整。

- 修改 CIDR，与上面初始化设置的 Pod 网段一致：

  ```shell
  $ sed -i '/# - name: CALICO_IPV4POOL_CIDR/ {s/# //; n; s/# //; s/"192.168.0.0\/16"/"10.244.0.0\/16"/}' calico.yaml
  ```
- 执行命令，部署 Calico:

  ```shell
  $ kubectl apply -f calico.yaml

  ```

- 通过命令持续观察各服务容器状态（耐心等待 Running）：

  ```shell
  $ watch kubectl get pods --all-namespaces -o wide
  Every 2.0s: kubectl get pods --all-namespaces -o wide                                                                                                     master1: Sat Jan 11 16:08:02 2025

  NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE      NOMINATED NODE   READINESS GATES
  kube-system   calico-kube-controllers-74dfd97c55-vgw8b   1/1     Running   0          3m21s   10.244.137.65   master1   <none>           <none>
  kube-system   calico-node-82g4v                          1/1     Running   0          3m21s   192.168.33.11   master1   <none>           <none>
  kube-system   coredns-6766b7b6bb-f2shx                   1/1     Running   0          64m     10.244.137.67   master1   <none>           <none>
  kube-system   coredns-6766b7b6bb-vsl65                   1/1     Running   0          64m     10.244.137.66   master1   <none>           <none>
  kube-system   etcd-master1                               1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  kube-system   kube-apiserver-master1                     1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  kube-system   kube-controller-manager-master1            1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  kube-system   kube-proxy-ldsp7                           1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  kube-system   kube-scheduler-master1                     1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  kube-system   kube-vip-master1                           1/1     Running   0          64m     192.168.33.11   master1   <none>           <none>
  ```
- 查看节点状态，正常：

  ```shell
  root@master1:~# kubectl get node
  NAME      STATUS   ROLES           AGE   VERSION
  master1   Ready    control-plane   66m   v1.32.0
  ```






https://blog.csdn.net/networken/article/details/132594119


- 让我们来测试集群网络是否正常：
https://www.cnblogs.com/guangdelw/p/18222715

提醒 git /gitee 查看相关清单或程序包等
## 其它

```shell
vagrant snapshot save [机器名] <snapshot_name>

vagrant snapshot list [机器名]

vagrant snapshot restore [机器名] <snapshot_name>

vagrant snapshot delete [机器名] <snapshot_name>
```

