---
"title": 开发文档
"weight": 20
---

这篇文档会深入介绍 kolla-helm 各个 chart 之间是如何配合的，chart 内部 job 的执行顺序以及如何通过 ingress 向外部暴露 openstack 组件的 api 服务。

## 通用 chart

在 koll-helm 有三个 chart 比较特殊：common，password，openstack-dep 这个三个都不是以 openstack 服务命名的，我们称它们为通用 chart。

* common

  common chart 是一个 library chart，包含了很多通用模板（template）和模板函数（template function）,这个 chart 无法直接部署，只能作为其他 chart 的依赖。

* password

  这个 chart 的作用与 kolla-ansible 的 kolla-genpwd 命令作用类似，password chart 会自动生成部署 openstack 服务所需的
  密码（主要是数据库用户密码和 keystone 用户密码），这些密码会存储在一个特定的 secert（名称与 password chart release 名称相同）中。

```yaml
apiVersion: v1
data:
  cinder-database-password: akNMc3hBVWUzWA==
  cinder-keystone-password: T3FSbnNvTTg2Yg==
  cinder-rbd-secret-uuid: ZjVhM2ZmNTQtODU3Mi00ZmY3LWI4N2ItM2MzNDcyMDJkNDFk
  glance-database-password: WlpuTXdOaXdlQQ==
  glance-keystone-password: WVJWSEhRelFUcw==
  keystone-admin-password: ajdGNWhGN25hVQ==
  keystone-database-password: OEI0S2ZCdU5VQg==
  mariadb-password: dVZCbFE4S2RXaA==
  mariadb-replication-password: bXJveHZTdjJObg==
  mariadb-root-password: TEROdVpxakNBYg==
  neutron-database-password: MUZwMHp3SmVBUw==
  neutron-keystone-password: ZUlRcWNyNjlibA==
  nova-database-password: NzF3bWEzODJ4MQ==
  nova-keystone-password: Q2pKUWJ5VWhsSg==
  placement-database-password: eEhDWUFlVVRTWg==
  placement-keystone-password: c2JoNmQxWHpIcg==
  rabbitmq-password: RXpvdXZFc2t1Wg==
  rbd-secret-uuid: MjI4ZWQ2M2MtYjAwYy00ZWVkLWE2ZGUtYmMzNTM4NzJkYzMz
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: openstack-password
    meta.helm.sh/release-namespace: openstack
  creationTimestamp: "2022-06-06T06:33:11Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  name: openstack-password
  namespace: openstack
  resourceVersion: "23578343"
  uid: f8501cd3-31ab-497e-a0ae-2f93432ab15a
type: Opaque
```

* openstack-dep

  这个 chart 主要安装 openstack 组件依赖的通用的服务，如：mariadb，rabbitmq，memcached。还有一个 nginx-ingress-controller，
  主要用于统一 openstack 各个服务对外暴露的 endpoint。另外要说明的是这个 chart 仅是封装了 [bitnami](https://github.com/bitnami/charts)
  提供的对应的 chart，如果想了解各个服务的详细信息和参数请参考 [bitnami 文档](https://github.com/bitnami/charts)。

  这个 chart 部署后会生成一个和 release 同名的 secret，这个 secret 中记录了各个服务的连接信息。

## job 执行顺序

在部署 openstack 服务的过程中需要执行很多 job，如初始化数据库，创建 keystone 用户，创建 loop 设备等，这其中有很多 job 是每个 openstack 服务
都需要执行的，这些 job 的 manifests 模板都放在了 common chart 中，这一章节来介绍一下这些通用 job 的作用，和这些 job 的依赖顺序。

* *-cm-render

  这个 job 没有任何依赖，通常是最先执行的 job，这个 job 主要是渲染相应的 openstack 服务的配置文件的 configmap。在上文提到了，openstack 服务需
  要用到的各种账号的密码都是由 password chart 提前生成并存放的指定的 secert 中的，helm 在渲染模板阶段是无法获取这些密码的，因此在配置文件模板中
  需要用到密码的地方我们都放置了一个特殊的字符串占位，这个 job 的作用就是用正确的密码替换这些占位字符串。

  如数据库连接的配置，在执行这个 job 之前如下（以 cinder 为例）：

  ```ini
  [database]
  connection = mysql+pymysql://cinder:database_password_placeholder@database_endpoint_placeholder/cinder
  ```

  在执行完这个 job 后相应的占位符被替换：

  ```ini
  [database]
  connection = mysql+pymysql://cinder:jCLsxAUe3X@openstack-dependency-mariadb.openstack.svc.cluster.local:3306/cinder
  ```

* *-db-init

  这个 job 在 *-cm-render 之后执行，用于创建相应服务的 database，数据库用户，并授权

* *-db-sync

  这个 job 在 *-db-init 和 *-cm-render 之后执行，执行相关服务的 ``db-manage db sync`` 命名同步表到数据库。

* *-register

  这个 job 依赖 keystone-api service，只要在 keystone api 接口能正常使用后才执行这个 job，这个 job 用于创建各个服务的 keystone 账号，向 keystone 注册 endpoints。

* *--update-openstack-conn-info

  在上文我们提到了，部署完 openstack-dep 后生成一个记录各个服务连接信息的 secret 这个 job 就是用来更新这个 secert 的，在我们部署完一个新的 openstack 的服务后
  我们需要把这个服务的连接信息写入这个 secert 方便后期其他服务的 chart 使用。

## ingress

在部署完 openstack-dep 后会默认创建一个名为 openstack-nginx 的 ingressclass，koll-helm 通过创建 ingress 向外暴露 openstack 各个组件的 API 服务，以
cinder  为例，其 manifest 如下

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    meta.helm.sh/release-name: openstack-cinder
    meta.helm.sh/release-namespace: openstack
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  creationTimestamp: "2022-06-06T07:44:43Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: openstack-cinder
  namespace: openstack
  resourceVersion: "23589571"
  uid: c93635d3-3815-498b-810a-c4ca0821deb6
spec:
  ingressClassName: openstack-nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: cinder-api
            port:
              number: 8776
        path: /volumev3(/|$)(.*)
        pathType: Prefix
status:
  loadBalancer:
    ingress:
    - ip: 172.18.30.166
```
