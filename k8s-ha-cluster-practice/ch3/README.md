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

  首先，确保 Helm 已经安装并配置好 Kubernetes 集群。在 Helm 中添加 Longhorn 的 Helm 仓库：

  ```bash
  helm repo add longhorn https://charts.longhorn.io
  helm repo update
  ```

- **步骤 2：安装 Longhorn**

  使用 Helm 安装 Longhorn：

  ```bash
  helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
  ```

- **步骤 3：配置存储类**

  安装完成后，Longhorn 将自动创建一个存储类。你可以通过以下命令确认：

  ```bash
  kubectl get storageclass
  ```

  确保 Longhorn 被设置为默认存储类，如果不是，可以手动配置：

  ```bash
  kubectl patch storageclass longhorn -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"true"}}}'
  ```

- **步骤 4：验证安装**

  检查 Longhorn 是否正常运行：

  ```bash
  kubectl get pods -n longhorn-system
  ```

  你应该看到所有的 Longhorn 相关 Pod 都处于 `Running` 状态。

- **步骤 5：通过 Longhorn UI 进行管理**

  你可以访问 Longhorn 的 Web UI，进行存储的管理和监控。默认情况下，Longhorn UI 可通过以下命令访问：

  ```bash
  kubectl port-forward service/longhorn-frontend 8080:80 -n longhorn-system
  ```

  访问 `http://localhost:8080`，可以进行更详细的配置和管理。

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

## 配置和验证
   - **验证存储组件**
     - 验证 Longhorn、NFS 和 MinIO 是否正常运行。
     - 检查 PVC 和 Pod 的状态，确保持久化存储工作正常。
   - **示例应用**：部署一个简单的应用，验证文件存储和对象存储的使用。
     - 如何在 Pod 中挂载 NFS 存储。
     - 如何在应用中访问 MinIO 对象存储。

## 总结
本文详细介绍了如何在高可用 Kubernetes 集群中通过 Helm 安装并配置 Longhorn、NFS 和 MinIO 存储组件。使用 Helm 进行部署简化了存储组件的安装和管理，同时确保了 Kubernetes 环境中的数据持久性和可扩展性。未来可以根据需要，扩展更多存储组件，满足不同业务场景的需求。
