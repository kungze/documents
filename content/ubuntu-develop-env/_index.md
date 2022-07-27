---
"title": "ubuntu-develop-env"
"weight": 2
---

ubuntu-develop-env 用于快速在 k8s 平台启动一个独立的开发调试环境，当研发人员想开发一个基于 k8s 的组件，尤其是这个组件
还与 k8s 平台上的其他组件或服务有交互时，这个组件的调试工作将会尤为困难。另外在多人团队中，为每个人打造一套隔离的开发调试
环境减少相互之间的干扰也能大大提高开发效率。

## 快速开始

```console
helm repo add kungze https://charts.kungze.net
helm install ubuntu-develop-env kungze/ubuntu-develop-env
```

在部署成功后可以使用 ssh 登录

```console
ssh ubuntu@<宿主机节点 IP> -p 30022
```

密码为 ChangeMe

## 详细说明

在通过 helm 部署成功后，相应的 release 会包含一个 statefulset 很一个 nodeport 类型
的 service。这个 statefulset 的 pod 的副本数为 1。pod 的 /home 目录默认会被挂载到宿主机的
/data/ubuntu 目录（可以通过 `--set hostConf.homeDirPath` 指定别的目录），这样即使发送意外
情况导致 pod 重建 /home 目录下的数据也会被保留，所以在使用这个环境时
要注意把重要数据放在 /home 目录下。还可以通过 `--set homeStorageClass=<storageclass 名称>` 来为 pod 的
/home 目录分配持久化存储。需要特别注意的是：在部署多个环境时我们需要通过 `--set hostConf.nodePort` 为每个环
境指定不同的 nodeport。更多其他参数可以参考[文档](https://github.com/kungze/ubuntu-develop-env/tree/main/chart)

pod 的镜像已经预先安装了 python3，golang 和 nodejs 开发环境，并且安装了 kubectl，helm 和 openstack 命令。

### kubectl

k8s 集群相关环境变量被设置在了 root 用户，为了能正常使用 kubectl 命令，我们需要先切换到 root 用户

```console
$ sudo su -
# kubectl get pods
NAME                                READY   STATUS    RESTARTS         AGE
ubuntu-develop-env-statusfulset-0   1/1     Running   0                158m
```

为了能在非 root 用户下使用 kubectl 命令，可以先通过 kubectl config 命令生成 kubeconfig 文件，然后把文件内容复制到
对应用户的家目录下的 .kube/config 文件。

### helm

helm 命令没什么好说的，只要能执行 kubectl 命令就能执行 helm 命令

### openstack

openstack 命令是为我们开发调试 kolla-helm 准备的，在部署 keystone 的 chart 时，我们会的得到下面的提示信息

```text
** Please be patient while the chart is being deployed **

Create a `openstackrc` file, and write the below lines to the file:

  export OS_USERNAME=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_USERNAME}" | base64 --decode)
  export OS_PROJECT_DOMAIN_NAME=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_PROJECT_DOMAIN_NAME}" | base64 --decode)
  export OS_USER_DOMAIN_NAME=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_USER_DOMAIN_NAME}" | base64 --decode)
  export OS_PROJECT_NAME=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_PROJECT_NAME}" | base64 --decode)
  export OS_REGION_NAME=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_REGION_NAME}" | base64 --decode)
  export OS_PASSWORD=$(kubectl get secrets -n test-glance openstack-password -o jsonpath="{.data.keystone-admin-password}" | base64 --decode)
  export OS_AUTH_URL=$(kubectl get secret -n test-glance openstack-keystone -o jsonpath="{.data.OS_CLUSTER_URL}" | base64 --decode)
```

我们把 openstackrc 创建在 pod 内，然后只执行 source openstackrc 就可以使 openstck 命令正常执行了

```console
$ source openstackrc
$ openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                                        |
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------------------------+
| 0b753562864f4059bf50388bda412819 | RegionOne | keystone     | identity     | True    | public    | http://openstack.test-glance.svc.cluster.local/identity/v3 |
| 19223d9dd12340b4bd2f6700b35c86cd | RegionOne | glance       | image        | True    | internal  | http://glance-api.test-glance.svc.cluster.local:9292       |
| 1c5484c3a9604c30880b602b34672ad8 | RegionOne | keystone     | identity     | True    | admin     | http://keystone-api.test-glance.svc.cluster.local:5000/v3  |
| 2b30abd10b134011bb2f83f3648030bb | RegionOne | keystone     | identity     | True    | internal  | http://keystone-api.test-glance.svc.cluster.local:5000/v3  |
| a407d86aeaf94701a2158b505bd95eaf | RegionOne | glance       | image        | True    | public    | http://openstack.test-glance.svc.cluster.local/image/v2    |
| f942d88dbaf148798ac7c9f9c8937eed | RegionOne | glance       | image        | True    | admin     | http://glance-api.test-glance.svc.cluster.local:9292       |
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------------------------+
```

## 代码开发

对于代码的开发调试，极力推荐大家结合 vscode 的 remote-ssh 插件使用
