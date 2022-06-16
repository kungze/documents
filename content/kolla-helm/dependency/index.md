---
"title": "通用中间件部署"
"weight": 2
---

通用中间件包括 mariadb，rabbitmq，memcached，nginx-ingress-controller 这些中间件我们都通用封装在了 openstack-dep chart 中了。

## 部署 ``password`` chart

在部署 openstack 的过程中会注册很多数据库和 keystone 用户，对应的我们需要设置很多密码，为了后面部署方便和密
码的安全性，我们专门编写了一个 password chart，部署这个 chart 可以随机生成一些密码，后面我们在部署其他项目时
不用在关心密码问题。

```shell
$ helm -n openstack install openstack-passwork kolla-helm/password
NAME: openstack-passwork
LAST DEPLOYED: Mon Jun  6 14:15:45 2022
NAMESPACE: openstack
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: password
CHART VERSION: 1.0.0

** Please be patient while the chart is being deployed **

Check the Secret:

  kubectl get secret --namespace openstack openstack-passwork -o yaml
```

## 部署 openstack-dep

``openstack-dep`` 把部署 openstack 所需要的中间件封装在一个 chart 里，通过这种方式后续部署 openstack 其他项目时可以共用这一套中间件：

* mariadb https://github.com/bitnami/charts/tree/master/bitnami/mariadb
* rabbitmq https://github.com/bitnami/charts/tree/master/bitnami/rabbitmq
* memcached https://github.com/bitnami/charts/tree/master/bitnami/memcached
* nginx-ingress-controller https://github.com/bitnami/charts/tree/master/bitnami/nginx-ingress-controller

默认情况下，openstack-dep 会部署 ceph 相关的 configmap。该 configmap 主要存储 ceph 集群的 endpoint 信息，通过指定的 `ceph.cephClusterNamespace` 和 `ceph.cephClusterName` 参数来同步 ceph 集群中 endpoint 配置。 为 cinder, glance 等需要使用 ceph 存储的服务，提供 ceph 集群的 endpoint 配置。由于该操作是公共的一次性操作，所以在 openstack-dep 中运行。

```shell
helm -n openstack install openstack-dependency kolla-helm/openstack-dep \
    --set ceph.cephClusterNamespace=rook-ceph \
    --set ceph.cephClusterName=rook-ceph
```

这里推荐使用 ceph 做为存储后端，所以默认环境中 `ceph.enabled` 参数为 true，即默认环境中部署了ceph。 若环境中未部署 ceph，可通过增加 `ceph.enabled=false` 同时去除 `ceph.cephClusterNamespace` 和 `ceph.cephClusterName` 参数来跳过这一步骤。

```shell
helm -n openstack install openstack-dependency kolla-helm/openstack-dep \
    --set ceph.enabled=false
```

默认情况下 openstack-dep 会创建一个名称为 ``openstack`` NodePort 的 service 向外部暴露 ingerss，设置 ``externalService.type`` 为
LoadBalancer 则该 service 会变为 loadbalancer 类型，通过 ``externalService.loadBalancerIP`` 可以为这个 service 指定一个特定的 IP，在
外部可以通过这个 IP 访问 openstack 服务。

通过下面命令观察

```shell
watch -n 1 kubectl -n openstack get pods -l app.kubernetes.io/instance=openstack-dependency
```

等待所有的 pod 都 ready 后，安装 openstack keystone。

