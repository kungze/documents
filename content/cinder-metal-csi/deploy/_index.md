---
"title": "cinder-metal-csi 部署文档"
"weight": 2
---

cinder-metal-csi 是 k8s 上的一个csi存储插件，用于管理 openstack cinder 卷的生命周期。 
cinder-metal-csi 支持 noauth、keystone 两种认证方式，其中 noauth 只需部署 cinder 服务即可。keystone 认证则需部署 keystone 服务。 
cinder-metal-csi 存储后端支持 ceph、 lvm、 local 三种。

## 命令行参数

cinder-metal-csi 支持以下参数：

- nodeid 
  - csi 插件所在的k8s节点IP
- endpoint
  - csi 服务监听的本地socket地址， 默认为 unix:///csi/csi.sock
- cloud-config
  - 驱动程序配置文件的路径，默认为 /etc/cloud/cloud.conf

## 配置文件配置

配置文件分为 Global 和 BlockStorage 两哥部分，其中 Global 主要负责 openstack 认证相关， BlockStorage 负责 cinder-metal-csi 的认证方式以及存储类型等

- Global
  - username keystone 用户名
  - password keystone 密码
  - user-domain-name keystone user 域名
  - project-domain-name keystone 项目域名
  - tenant-name keystone 项目名
  - auth-url keystone 认证 URL
  - region keystone region 名
  - endpoint-type keystone endpoint 类型
- BlockStorage
  - auth-strategy 认证策略，支持keystone 和 noauth 两种方式，如果认证方式配置noauth，则无须配置GLoabl下的参数，配置为keystone，则需配置Global下的参数
  - cinder-listen-addr noauth 认证方式配置cinder api 的监听地址
  - node-volume-attach-limit 节点可挂载卷的最大数量，默认为256
  - lvm-volume-type lvm 卷后端名称，需要跟 cinder 卷存储后端的 volume-type 名称以及 storageclass 的 [type](https://github.com/kungze/cinder-metal-csi/blob/main/example/nginx-lvm.yaml#L10) 保持一致
  - ceph-volume-type ceph 卷后端名称，需要跟 cinder 卷存储后端的 volume-type 名称以及 storageclass 的 [type](https://github.com/kungze/cinder-metal-csi/blob/main/example/nginx-rbd.yaml#L10) 保持一致
  - local-volume-type local 卷后端名称，需要跟 cinder 卷存储后端的 volume-type 名称以及 storageclass 的 [type](https://github.com/kungze/cinder-metal-csi/blob/main/example/nginx-local.yaml#L10) 保持一致

## 部署

部署 cinder-metal-csi 插件有两种方式，一种是通过 manifests 清单文件进行部署，另一种是通过helm charts进行部署。

### manifests 资源清单部署

部署资源清单的文件在项目的 [manifests](https://github.com/kungze/cinder-metal-csi/tree/main/manifests) 目录下。
首先，修改配置文件配置 [cloud.conf](https://github.com/kungze/cinder-metal-csi/blob/main/example/cloud.conf)，
配置文件以 secrets 的形式存储在 kubernetes 环境中，将修改后的配置文件通过 base64 命令进行加密：

```
$ base64 -w 0 cloud.conf
```

将加密后的内容写入 csi-secret-cinderplugin.yaml 文件下的 cloud.conf 配置中，通过以下命令创建 secrets：

```
$ kubectl create -f manifests/csi-secret-cinderplugin.yaml
```

如果存储后端为 ceph，需要修改 [cinder-config.yaml](https://github.com/kungze/cinder-metal-csi/blob/main/manifests/cinder-config.yaml) 下的 ceph.conf 中的 mon_host 配置，配置为连接 ceph 集群的 monitor 地址，配置完成后通过以下命令镜像创建：

```
$ kubectl create -f manifests/cinder-config.yaml
```

还需获取 ceph 集群 admin 用户的 keyring （暂时 ceph 认证只支持 admin 用户），配置格式：
```
[client.admin]
key = AQAsxc9ipU1ELhAAf9zZKZvyVPLNev1XkEWeKg==
```

将 keyring 配置文件，通过 base64 命令进行加密，将加密后的内容替换到[cinder-volume-rbd-keyring.yaml
](https://github.com/kungze/cinder-metal-csi/blob/main/manifests/cinder-volume-rbd-keyring.yaml) 中的 key 字段

```
$ base64 -w 0 keyring
```

通过以下命令进行创建

```
$ kubectl create -f manifests/cinder-volume-rbd-keyring.yaml
```

配置修改完成，通过以下命令进行创建 controllerplugin 和 nodeplugin

```
$ kubectl create -f manifests/
```

创建完成，获取 pod 的状态

```
$ kubectl get pods -n kube-system
NAME                                                  READY   STATUS    RESTARTS         AGE
cinder-metal-csi-controller-plugin-7ff586b8f5-fvb88   6/6     Running   0                2d22h
cinder-metal-csi-nodeplugin-58mwv                     4/4     Running   0                2d22h
```

查看 CSI Driver 的信息

```
kubectl get csidrivers.storage.k8s.io
NAME                            ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
cinder.metal.csi                true             true             false             <unset>         false               Persistent   2d22h
```

### helm charts 部署

通过以下命令添加 kungze helm repo

```
helm repo add kungze https://kungze.github.io/cinder-metal-csi
```

下载 cinder-metal-csi helm charts 压缩包

```
helm fetch kungze/cinder-metal-csi
```

解压 cinder-metal-csi-1.0.0.tar.gz 压缩包

```
tar -xvf cinder-metal-csi-1.0.0.tar.gz
```

修改 values.yaml 文件

修改 cloud 相关配置，authStrategy 表示 cinder-metal-csi 的认证方式，配置为 keystone 则需配置 username、userPassword、tenantName、authUrl 字段，配置为 noauth 只需配置 cinderListenAddr 即可

```
cloud:
  authStrategy: keystone
  username: admin
  userPassword: o2DkgbcwDZ
  tenantName: admin
  authUrl: http://keystone-api.default.svc.cluster.local:5000/v3
  cinderListenAddr: ""
```

修改 backend 相关配置，部署环境中 cinder 支持的存储后端

```
backend:
  lvm: true
  local: true
  ceph: true
```

修改 ceph 相关配置，keyring 为 ceph 集群中 admin 用户的 keyring 文件配置，通过 base64 加密后的结果，monAddr 为 ceph 集群的 monitor 地址

```
ceph:
  keyring: W2NsaWVudC5hZG1pbl0Ka2V5ID0gQVFBc3hjOWlwVTFFTGhBQWY5elpLWnZ5VlBMTmV2MVhrRVdlS2c9PQo=
  monAddr: 10.111.43.63:6789
```

修改 storageclass 相关配置

```
storageClass:
  enabled: true
  allowVolumeExpansion: true
```

通过以下命令安装 cinder-metal-csi charts

```
helm install cinder-metal-csi cinder-metal-csi/
```
