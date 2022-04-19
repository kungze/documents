---
"title": "kolla-helm"
"weight": 1
---

kolla-helm 包含一些列 helm chart，基于这些 chart 我们可以很容易的在 k8s 平台上部署 openstack，
在 openstack 社区有一个 [openstack-helm][openstack-helm] 项目提供了类似的功能，但是由于
openstack-helm 设计过于复杂，项目活跃度底，不兼容最新 helm，所以我们打算提供一套新的 chart。
我们为什么叫 kolla-helm 呢？顾名思义，我们想要借助 [kolla][kolla] 容器来完成新版 chart 的编写。

借助这些 chart 你可以有多种方式来轻松的部署 openstack

## 通过命令行部署

你可以通过 helm 命令部署 openstack。

## 图形化部署

你还可以通过 [kubeapp][kubeapp] 提供的 web UI 轻松的部署 openstack。

[openstack-helm]: https://docs.openstack.org/openstack-helm/latest
[kolla]: https://docs.openstack.org/kolla/latest
[kubeapp]: https://github.com/vmware-tanzu/kubeapps
