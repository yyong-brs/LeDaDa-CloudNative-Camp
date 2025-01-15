# K8s 集群中存储组件的安装与配置（NFS/Longhorn/MinIO）

## 引言

> 前面我们已经通过 Vagrant 创建了集群所需虚拟机环境，并且通过 kubeadm 部署并验证了基于 Kube-Vip 的高可用集群。接下来我们继续在集群中部署配置存储组件。

在现代云原生应用中，存储是必不可少的组件。尤其是对于生产环境中的 Kubernetes（K8s）集群，可靠的存储解决方案至关重要。本文将介绍如何在K8s 集群中部署并配置文件存储和对象存储解决方案，具体包括 Longhorn、NFS 和 MinIO。所有这些存储组件都将通过 Helm 安装，简化了部署过程，提高了管理效率。


**集群的环境配置可参考前面的文章：**
- [高可用K8S集群搭建指南（一） : Vagrant 多节点虚拟机集群搭建](https://mp.weixin.qq.com/s/uqbpcEovKLZ61Deq_9qQGw)。
- [高可用K8S集群搭建指南（二） : Kubernetes 1.32 + Kube-Vip：高可用集群部署全攻略](https://mp.weixin.qq.com/s/PQ4bK-xRj8hd3gCBLJirPQ)。



##  存储组件的选择

关于 k8s 存储组件的对比、介绍等可参考往期文章：[七个开源最佳 Kubernetes 存储解决方案](https://mp.weixin.qq.com/s/eWTLcj0qQvHREQAPesgeiQ)、[Longhorn：Kubernetes 原生块存储](https://mp.weixin.qq.com/s/1VkPYAhgRxrOpYzqO19b7w)。接下来将对Longhorn、nfs、minio 展开介绍。

- **文件存储**：Longhorn 和 NFS。
  - **Longhorn**：Longhorn 是一个轻量级、高可用的分布式块存储系统，特别适用于 Kubernetes 环境。它提供易于使用的图形界面、自动快照、备份和恢复等功能，支持动态扩展，并且可以与 Kubernetes 的存储类（StorageClass）集成，满足大多数存储需求。
  - **NFS**：Network File System（NFS）是一个用于文件共享的协议，适合需要多节点共享文件的场景。NFS 在 Kubernetes 中常作为一个共享存储解决方案，适合那些需要跨多个 Pod 或节点共享数据的应用场景。
- **对象存储**：MinIO。
  - **MinIO**：**MinIO**是一个兼容 Amazon S3 的高性能对象存储解决方案，广泛用于存储非结构化数据（如图片、视频、大型日志文件等）。MinIO 在 Kubernetes 中作为对象存储组件，能够提供持久化存储和高性能的对象数据访问。

## Helm 安装工具介绍

Helm 是 Kubernetes 的包管理工具，简化了应用程序的安装、升级、删除和管理。通过 Helm，我们可以轻松地安装、配置和管理 Kubernetes 中的存储组件。详细介绍，参考之前文章：[什么是 Helm Charts？深入了解 Kubernetes 包管理器](https://mp.weixin.qq.com/s/-vljvyfpBBAUxK3w23s66w)。

**为什么选择 Helm：**
- **简化管理**：Helm 使 Kubernetes 应用的管理变得更加简便，通过定义模板和配置，自动化安装和部署。
- **易于扩展**：使用 Helm Chart 可以方便地对应用进行版本控制和升级。
- **支持社区应用**：Helm 的官方和社区仓库提供了大量经过验证的应用部署模板。

**快捷安装**：通过[helm release](https://github.com/helm/helm/releases "helm release")下载指定 OS 的二进制包，windows 设置 PATH 环境变量指向二进制程序目录，Linux 则将二进制程序放入 `/usr/local/bin/helm` 即可。（无法获取安装包的，可以到文末我的仓库中获取哦）。安装成功，通过以下命令检查：

```shell
root@master1:/opt# helm version
version.BuildInfo{Version:"v3.17.0-rc.1", GitCommit:"301108edc7ac2a8ba79e4ebf5701b0b6ce6a31e4", GitTreeState:"clean", GoVersion:"go1.23.4"}
```

**TIPS**: 可以安装在本地电脑，那么在部署 helm 应用时需要指定远端的 k8s 集群配置文件。如果直接安装在 k8s 集群 master 节点中，则只要配置了`~/.kube/config` 文件，则不需要指定配置文件。

## 安装 Longhorn

**Longhorn** 是一个开源的分布式块存储系统，可以为 Kubernetes 提供可靠的存储。

- **步骤 1：添加 Helm 仓库**

  首先，在 Helm 中添加 Longhorn 的 Helm 仓库：

  ```bash
  helm repo add longhorn https://charts.longhorn.io
  helm repo update
  ```

- **步骤 2：安装 Longhorn**
  Longhorn 的 Helm Chart 信息查看 [ArtifactHub](https://artifacthub.io/packages/helm/longhorn/longhorn "Longhorn Chart") 获取。

  - 安装 NFSv4客户端
    在 Longhorn 系统中, 备份功能需要 NFSv4, v4.1 或是 v4.2, 同时， ReadWriteMany (RWX) 卷功能需要 NFSv4.1。因此，需要在所有节点提前安装 NFSv4 客户端。
    ```bash
    apt-get install nfs-common
    ```

  - 安装 open-iscsi
    必要组件，Longhorn 依赖主机上的 iscsiadm 向 Kubernetes 提供持久卷。
    ```bash
    apt-get install open-iscsi
    systemctl enable iscsid
    systemctl start iscsid
    ```

  - 方式一：使用 Helm 安装远程 Charts ：
    
    **注：** 适用于快速部署，少量配置调整的场景。
    ```bash
    helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace -f longhorn-values.yaml --kubeconfig=k8s.conf
    ```
    - `longhorn` 本次安装的 Chart 的实例名称(Release Name)。
    - `longhorn/longhorn` 本次安装的 Chart。
    - `--namespace` 指定目标命名空间。
    - `--create-namespace` 标识当命名空间不存在时，自动创建命名空间。
    - `-f longhorn-values.yaml` 用于指定具体的参数值，详细内容可参考文末的本人Git 仓库。
    - `--kubeconfig=k8s.conf` 用于指定目标k8s 集群的配置文件，若直接在集群中执行 helm 命令则可以不增加此参数。

  - 方式二：使用 Helm 安装本地 Charts
    
    **注：** 适用于精细部署，个性化配置，甚至调整部署的 manifest。

    下载并解压 Chart 包
   
    ```bash
    helm pull longhorn/longhorn
    tar zxf longhorn-1.7.2.tgz
    ```
    - `helm pull` 可以增加 `--version  xxxx` 用于指定具体版本，不指定，则获取最新版本。
  
    拷贝默认的 `Values` 文件：
    ```bash
    cd longhorn
    cp values.yaml longhorn-values.yaml
    ```

    vi 修改 `longhorn-values.yaml`，主要修改内容：
    ```yaml
    # 镜像调整为我的阿里云仓库，否则没有梯子拉不下来
    image:
      longhorn:
        engine:
          # -- Repository for the Longhorn Engine image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn Engine image.
          tag: longhorn-engine-v1.7.2
        manager:
          # -- Repository for the Longhorn Manager image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn Manager image.
          tag: longhorn-manager-v1.7.2
        ui:
          # -- Repository for the Longhorn UI image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn UI image.
          tag: longhorn-ui-v1.7.2
        instanceManager:
          # -- Repository for the Longhorn Instance Manager image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn Instance Manager image.
          tag: longhorn-instance-manager-v1.7.2
        shareManager:
          # -- Repository for the Longhorn Share Manager image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn Share Manager image.
          tag: longhorn-share-manager-v1.7.2
        backingImageManager:
          # -- Repository for the Backing Image Manager image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Backing Image Manager image. When unspecified, Longhorn uses the default value.
          tag: backing-image-manager-v1.7.2
        supportBundleKit:
          # -- Repository for the Longhorn Support Bundle Manager image.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the Longhorn Support Bundle Manager image.
          tag: support-bundle-kit-v0.0.45
      csi:
        attacher:
          # -- Repository for the CSI attacher image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI attacher image. When unspecified, Longhorn uses the default value.
          tag: csi-attacher-v4.7.0
        provisioner:
          # -- Repository for the CSI Provisioner image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI Provisioner image. When unspecified, Longhorn uses the default value.
          tag: csi-provisioner-v4.0.1-20241007
        nodeDriverRegistrar:
          # -- Repository for the CSI Node Driver Registrar image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI Node Driver Registrar image. When unspecified, Longhorn uses the default value.
          tag: csi-node-driver-registrar-v2.12.0
        resizer:
          # -- Repository for the CSI Resizer image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI Resizer image. When unspecified, Longhorn uses the default value.
          tag: csi-resizer-v1.12.0
        snapshotter:
          # -- Repository for the CSI Snapshotter image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI Snapshotter image. When unspecified, Longhorn uses the default value.
          tag: csi-snapshotter-v7.0.2-20241007
        livenessProbe:
          # -- Repository for the CSI liveness probe image. When unspecified, Longhorn uses the default value.
          repository: registry.cn-shanghai.aliyuncs.com/yydd/longhornio
          # -- Tag for the CSI liveness probe image. When unspecified, Longhorn uses the default value.
          tag: livenessprobe-v2.14.0

    # 默认配置调整
    defaultSettings:
      # 设置允许 Longhorn 仅在标签为“node.longhorn.io/create-default-disk=true”的节点上自动创建默认磁盘（如果不存在其他磁盘）。禁用此设置时，Longhorn 在添加到集群的每个节点上创建一个默认磁盘。
      createDefaultDiskLabeledNodes: ~
      # 主机上存储数据的默认路径。默认值为“/var/lib/longhorn/”。
      defaultDataPath: /data/storage/longhorn
    # 先设置ui 访问通过 nodePort 方式，后续等部署到 ingress 的文章中，再调整为 ingress 访问。
    service:
      ui:
        # 设置 ui service 为 nodePort
        type: NodePort
        # 指定端口，可用端口范围 30000 and 32767. （不指定的话会默认创建一个）
        nodePort: 30080
    ```

    执行安装：
    ```bash
    cd ..
    PS D:\workspace\github\LeDaDa-CloudNative-Camp\k8s-ha-cluster-practice\ch3\helm-charts> helm install longhorn ./longhorn --namespace longhorn-system --create-namespace -f longhorn/longhorn-values.yaml --kubeconfig=C:\Users\brains\Downloads\admin.conf
    NAME: longhorn
    LAST DEPLOYED: Wed Jan 15 15:29:30 2025
    NAMESPACE: longhorn-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    Longhorn is now installed on the cluster!

    Please wait a few minutes for other Longhorn components such as CSI deployments, Engine Images, and Instance Managers to be initialized.

    Visit our documentation at https://longhorn.io/docs/
    ```
    - `longhorn` 本次安装的 Chart 的实例名称(Release Name)。
    - `longhorn/longhorn` 本次安装的 Chart。
    - `--namespace` 指定目标命名空间。
    - `--create-namespace` 标识当命名空间不存在时，自动创建命名空间。
    - `-f longhorn-values.yaml` 用于指定具体的参数值，详细内容可参考文末的本人Git 仓库。
    - `--kubeconfig=k8s.conf` 用于指定目标k8s 集群的配置文件，若直接在集群中执行 helm 命令则可以不增加此参数。
  - Watch Pod 达到 Running 状态，代表部署成功：
    ```bash
    watch kubectl get pod -n longhorn-system -o wide
    Every 2.0s: kubectl get pod -n longhorn-system -o wide                                                           master1: Wed Jan 15 09:49:29 2025

    NAME                                                READY   STATUS    RESTARTS      AGE   IP               NODE    NOMINATED NODE   READINESS GATE
    S
    csi-attacher-94b689496-58wrs                        1/1     Running   0             10m   10.244.166.172   node1   <none>           <none>
    csi-attacher-94b689496-dgjj7                        1/1     Running   0             10m   10.244.166.165   node1   <none>           <none>
    csi-attacher-94b689496-kkzp7                        1/1     Running   0             10m   10.244.166.163   node1   <none>           <none>
    csi-provisioner-86f577fddd-9lt6r                    1/1     Running   0             10m   10.244.166.173   node1   <none>           <none>
    csi-provisioner-86f577fddd-n5gsq                    1/1     Running   0             10m   10.244.166.169   node1   <none>           <none>
    csi-provisioner-86f577fddd-xh4qh                    1/1     Running   0             10m   10.244.166.164   node1   <none>           <none>
    csi-resizer-7589b8b586-65k5c                        1/1     Running   0             10m   10.244.166.166   node1   <none>           <none>
    csi-resizer-7589b8b586-hh9bp                        1/1     Running   0             10m   10.244.166.175   node1   <none>           <none>
    csi-resizer-7589b8b586-qq4mt                        1/1     Running   0             10m   10.244.166.168   node1   <none>           <none>
    csi-snapshotter-db9dcd54f-brn58                     1/1     Running   0             10m   10.244.166.171   node1   <none>           <none>
    csi-snapshotter-db9dcd54f-nt588                     1/1     Running   0             10m   10.244.166.167   node1   <none>           <none>
    csi-snapshotter-db9dcd54f-zh44r                     1/1     Running   0             10m   10.244.166.174   node1   <none>           <none>
    engine-image-ei-bc6697f9-fc9j8                      1/1     Running   0             10m   10.244.166.160   node1   <none>           <none>
    instance-manager-81f3df2191f0e4a5a0b7c5f126c1a995   1/1     Running   0             10m   10.244.166.161   node1   <none>           <none>
    longhorn-csi-plugin-79dfm                           3/3     Running   0             10m   10.244.166.170   node1   <none>           <none>
    longhorn-driver-deployer-566cf4b476-pcq8g           1/1     Running   0             11m   10.244.166.159   node1   <none>           <none>
    longhorn-manager-hp5bf                              2/2     Running   1 (10m ago)   11m   10.244.166.158   node1   <none>           <none>
    longhorn-ui-8495965bd5-bh6pg                        1/1     Running   0             11m   10.244.166.157   node1   <none>           <none>
    longhorn-ui-8495965bd5-lsjkz                        1/1     Running   0             11m   10.244.166.156   node1   <none>           <none>
    ```


- **步骤 3：验证功能**
  - 依据部署时的参数设定，添加标签至 node1 节点：
    ```bash
    kubectl label node node1 node.longhorn.io/create-default-disk=true
    ```
  - 通过 VIP + nodePort 端口，我们的是 `192.168.33.250:30080` 访问 UI:
    ![](images/1.png)
    
  - 我们可以通过开启 master 允许pod 调度，同时打上 longhorn 磁盘标签，查看具体的变化：
    ```bash
    # master2 允许调度
    kubectl taint node master2 node-role.kubernetes.io/control-plane:NoSchedule-
    # master2 打上 longhorn 创建磁盘 标签
    kubectl label node master2 node.longhorn.io/create-default-disk=true
    # master3 允许调度
    kubectl taint node master3 node-role.kubernetes.io/control-plane:NoSchedule-
    # master3 打上 longhorn 创建磁盘 标签
    kubectl label node master3 node.longhorn.io/create-default-disk=true
    ```
    可以看到已经有 3个节点可供调度：
    ![](images/2.png)
  - 查看 `StorageClass` 清单：
    ```bash
    root@master1:~# kubectl get storageclass
    NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   34m
    longhorn-static      driver.longhorn.io   Delete          Immediate           true                   34m
    ```
    可以看到一共创建了两个 StorageClass，其中`longhorn` 是默认StorageClass，用来动态提供创建 longhorn存储，而 `longhorn-static` 是用于预先配置 PVC（静态配置）的假储存类别。
  - 创建 `test-longhorn.yaml` ，用于部署一个 Pod，并且创建 动态PVC 挂载（longhorn）：
    ```yaml
    # pvc
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: longhorn-pvc
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
      # 指定上面查到的动态 sc
      storageClassName: longhorn
      resources:
        requests:
          storage: 1Gi
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: longhorn-pod
      namespace: 
    spec:
      containers:
      - name: busybox-container
        image: registry.cn-shanghai.aliyuncs.com/yydd/busybox:1.28
        command: ["/bin/sh", "-c", "while true; do echo $(date) >> /data/log.txt; sleep 5; done"]
        volumeMounts:
        - mountPath: "/data"
          name: storage
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: longhorn-pvc
    ```
  - 部署 `test-longhorn.yaml`:
    ```bash
    root@master1:~/yy# kubectl apply -f test-longhorn.yaml 
    persistentvolumeclaim/longhorn-pvc created
    pod/longhorn-pod created
    root@master1:~/yy# 
    ```
  - 进入容器查看 /data/log.txt 是否持续在产生内容：
    ```bash

    root@master1:~/yy# kubectl exec -it longhorn-pod -- tail -f /data/log.txt
    Wed Jan 15 10:25:48 UTC 2025
    Wed Jan 15 10:25:53 UTC 2025
    Wed Jan 15 10:25:58 UTC 2025
    Wed Jan 15 10:26:03 UTC 2025
    Wed Jan 15 10:26:08 UTC 2025
    Wed Jan 15 10:26:13 UTC 2025
    Wed Jan 15 10:26:18 UTC 2025
    Wed Jan 15 10:26:23 UTC 2025
    Wed Jan 15 10:26:28 UTC 2025
    Wed Jan 15 10:26:33 UTC 2025
    Wed Jan 15 10:26:38 UTC 2025
    ```
    可以看到 Dashboard 中有一个 Volume ，同时 也有多副本信息：
    ![](images/3.png)
    ![](images/4.png)

## 安装 NFS
   - 安装步骤：
     - 安装 NFS 服务：通过 Helm 安装 NFS 的 chart。
     - 配置 NFS 共享：设置 NFS 共享目录和权限。
     - 配置 PersistentVolume 和 PersistentVolumeClaim：如何在 K8s 中配置 NFS 存储。

## 安装 MinIO

- **步骤 1：添加 MinIO Helm 仓库**

  ```bash
  helm repo add minio https://charts.min.io
  helm repo update
  ```

- **步骤 2：安装 MinIO**

  使用 Helm 安装 MinIO：

  ```bash
  helm install minio minio/minio --namespace minio --create-namespace
  ```

- **步骤 3：配置 MinIO**

  在安装 MinIO 后，可以设置访问密钥和桶策略，确保数据的安全和可访问性。

- **步骤 4：验证安装**

  通过以下命令验证 MinIO 是否成功安装并运行：

  ```bash
  kubectl get pods -n minio
  ```

  你可以通过 `kubectl port-forward` 访问 MinIO 控制台：

  ```bash
  kubectl port-forward service/minio 9000:9000 -n minio
  ```

  访问 `http://localhost:9000`，使用 MinIO 提供的默认访问密钥进行登录。

## helm 扩展命令

```shell
# 列出已添加的 helm 仓库清单
helm repo list
# 列出某个 helm 仓库下的 charts
helm search repo [repo name]

```

## 配置和验证
   - **验证存储组件**
     - 验证 Longhorn、NFS 和 MinIO 是否正常运行。
     - 检查 PVC 和 Pod 的状态，确保持久化存储工作正常。
   - **示例应用**：部署一个简单的应用，验证文件存储和对象存储的使用。
     - 如何在 Pod 中挂载 NFS 存储。
     - 如何在应用中访问 MinIO 对象存储。

## 总结
本文详细介绍了如何在高可用 Kubernetes 集群中通过 Helm 安装并配置 Longhorn、NFS 和 MinIO 存储组件。使用 Helm 进行部署简化了存储组件的安装和管理，同时确保了 Kubernetes 环境中的数据持久性和可扩展性。未来可以根据需要，扩展更多存储组件，满足不同业务场景的需求。
