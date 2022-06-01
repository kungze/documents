---
"title": "准备工作"
"weight": 1
---

## 部署 k8s

首先我们需要有一套 k8s 集群，k8s 集群可以有多种方式，由于国内网络环境的问题，我们推荐两个可以在国内丝滑部署 k8s 集群的方案

* kubeadm，可以参考我们的文档[在国内使用 kubeadm 部署 k8s](../../post/kubeadm)
* kubeasz，这是由国内大神编写的一套部署 k8s 的 ansible 脚本，可以满足各种需要，使用方式可以参考这个项目的[文档](https://github.com/easzlab/kubeasz)

需要特别注意的是：为了能在宿主机节点上解析 k8s service 的域名（方便在宿主机上执行 openstack 命令），
[nodelocaldns](https://kubernetes.io/zh/docs/tasks/administer-cluster/nodelocaldns/) 是强烈推荐安装的。

## 部署 metallb

我们提供的方案需要在裸金属上直接部署 k8s，所以我们需要部署 metallb 为 loadbalancer 类型的 service 提供支撑，部署方式可以参考[文档](../../post/metallb)。
另外 [openelb](https://github.com/openelb/openelb) 也是一个不错的方案，但是 openelb 作者没有部署过，感兴趣的可以自己找文档研究部署。

## 部署 rook（可选）

[rook](https://github.com/rook/rook) 是一个开源的云原生存储编排平台，可以用来管理 ceph 集群，当我们
设置 cinder，glance，nova 的后端存储为 ceph 时，我们先必须要部署 rook，并使用 rook 创建创建一个 ceph 集群
或者纳管一个外部的 ceph 集群。

### ceph 集群（可选）

我们可以参考[官方文档](https://rook.github.io/docs/rook/v1.9/Getting-Started/quickstart/) 使用 rook 创建
一套 ceph 集群，但是在生产环境中我们更推荐使用传统先行部署一套 ceph 集群，然后再用 rook 纳管已经部署好的 ceph
集群。ceph 集群的部署可以使用 [ceph-ansible](https://github.com/ceph/ceph-ansible)，rook 纳管已经存在的 ceph
集群的方法可以参考[文档](https://github.com/rook/rook/blob/master/design/ceph/ceph-external-cluster.md).

## 安装 helm

helm 的安装就比较简单了，可以参考[官方文档](https://helm.sh/docs/intro/install/)，而且只用在执行部署命令的节点安装就行。
另外还可以安装 [kubeapps](https://github.com/vmware-tanzu/kubeapps) 来为我们提供图形界面部署 openstack。kubeapps 的安装
文档可以参考我们专门写的一篇[博客](../../blog/kubeapps/)。

### 添加 helm 仓库

```shell
helm repo add kolla-helm https://kungze.github.io/kolla-helm
```

在国内访问 github 可能会失败，你可以使用我们在国内的备份仓库

```shell
helm repo add kolla-helm https://charts.kungze.net
```

## 创建 namespace（可选）

建议单独创建一个 k8s namespace 用于部署 openstack 相关的 chart。

```shell
kubectl create namespace openstack
```

在准备工作完成后我们可以正式部署 openstack 了，注意：`password`，`openstack-dep` 和 `keystone` 需要优先部署。
