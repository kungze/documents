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

```shell
helm -n openstack install openstack-dependency kolla-helm/openstack-dep
```

默认情况下 openstack-dep 会创建一个名称为 ``openstack`` NodePort 的 service 向外部暴露 ingerss，设置 ``externalService.type`` 为
LoadBalancer 则该 service 会变为 loadbalancer 类型，通过 ``externalService.loadBalancerIP`` 可以为这个 service 指定一个特定的 IP，在
外部可以通过这个 IP 访问 openstack 服务。

通过下面命令观察

```shell
watch -n 1 kubectl -n openstack get pods -l app.kubernetes.io/instance=openstack-dependency
```

等待所有的 pod 都 ready 后，安装 openstack keystone。
