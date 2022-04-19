---
"title": "Kungze"
---

# kungze

Kungze 是由一群从事云计算方面工作小伙伴组建的一个社区，计划提供一系列云计算相关的开源项目，我们的工作主要聚焦在
[kubernetes][k8s] 和 [openstack][openstack]，当前的目标是打造一套 [kubernetes][k8s] 和 [openstack][openstack] 混合交付方案 (我们不想
提供一套 k8s on openstack 的方案，我相信这类方案市场上有很多了)，在我们的部
署方案中 [kubernetes][k8s] 和 [openstack][openstack] 是平级的，结合两者的长处打造一个稳定，高效，灵活且
易于部署的云平台。

k8s 能弥补 openstack 的哪些不足？

* 简化 openstack 的部署，我们第一阶段的主要工作就是打造一个通过 [helm][helm] 轻松部署 openstack 的方案
* 弥补 openstack 应用市场和 DBaaS 的不足
* 丰富 openstack 日志和监控告警的手段

openstack 能给 k8s 带来哪些好处？

* 借助 openstack cinder 提供多种后端存储
* 借助 openstack neutron 提供丰富灵活的 CNI 插件
* 借助 openstack keystone 完善 k8s 租户体系

基于以上的想法我们打算提供一些列的工具。


## kolla-helm

这个项目包含一些列 helm chart，基于这些 chart 我们可以很容易的在 k8s 平台上部署 openstack，
在 openstack 社区有一个 [openstack-helm][openstack-helm] 项目提供了类似的功能，但是由于
openstack-helm 设计过于复杂，项目活跃度底，不兼容最新 helm，所以我们打算提供一套新的 chart。
我们为什么叫 kolla-helm 呢？顾名思义，我们想要借助 [kolla][kolla] 容器来完成新版 chart 的编写。


## cinder-csi-plugin

由于 k8s 官方提供的 [cinder-csi-plugin][cinder-csi-plugin] 只适用于 k8s on openstack 的场景 (即：k8s 需
要运行在 openstack 虚机里面)。所以我们打算提供一套新的 cinder csi 插件，使运行在裸金属系统上的容器也能
很方便的使用 cinder 存储。


[openstack]: https://docs.openstack.org
[k8s]: https://kubernetes.io/docs/home
[helm]: https://helm.sh
[openstack-helm]: https://docs.openstack.org/openstack-helm/latest
[kolla]: https://docs.openstack.org/kolla/latest
[cinder-csi-plugin]: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md

