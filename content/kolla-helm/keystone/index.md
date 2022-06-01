---
"title": "部署 keystone"
"weight": 3
---

``keystone`` 是 openstack 的认证和服务发现组件。

## 部署 keystone

```shell
helm -n openstack install openstack-keystone kolla-helm/keystone
```

## 安装 openstackclient

安装 openstack 命令行，在安装完成后可以通过执行 openstack 命令验证安装是否成功。

```shell
apt install python3-openstackclient
```

## 创建 openstackrc 文件

openstack 认证相关信息会存放在一个专门的 secert （openstack-keystone，与 keystone chart 的 release 名称一致）
中，在 shell 终端执行下面命令导出 OS_* 相关环境变量以便后续 openstack 命令能正常执行。另外特别要注意：openstack 相
关的组件的 API 服务都是通过 service 暴露的，因此要求执行命令的节点能解析 k8s service de 域名（需要安装
nodelocaldns 插件）。

```shell
export OS_USERNAME=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_USERNAME}" | base64 --decode)
export OS_PROJECT_DOMAIN_NAME=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_PROJECT_DOMAIN_NAME}" | base64 --decode)
export OS_USER_DOMAIN_NAME=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_USER_DOMAIN_NAME}" | base64 --decode)
export OS_PROJECT_NAME=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_PROJECT_NAME}" | base64 --decode)
export OS_REGION_NAME=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_REGION_NAME}" | base64 --decode)
export OS_PASSWORD=$(kubectl get secrets -n openstack openstack-password -o jsonpath="{.data.keystone-admin-password}" | base64 --decode)
export OS_AUTH_URL=$(kubectl get secret -n openstack openstack-keystone -o jsonpath="{.data.OS_CLUSTER_URL}" | base64 --decode)
export OS_INTERFACE=internal
```

## 验证

通过下面命令观察

```shell
watch -n 1 kubectl -n openstack get pods -l app.kubernetes.io/instance=openstack-keystone
```

等待所有的 pod 都 ready 后，然后执行下面命令能否执行成功。

```shell
$ source openstackrc
$ openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                                      |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------------------------+
| 94e00ea36c2841ca82fd92fe73601870 | RegionOne | keystone     | identity     | True    | public    | http://openstack.openstack.svc.cluster.local/identity/v3 |
| ba9d6067919f4f86b60afb073538b7ee | RegionOne | keystone     | identity     | True    | admin     | http://keystone-api.openstack.svc.cluster.local:5000/v3  |
| ea680d2d7362424b8c9715e55516291d | RegionOne | keystone     | identity     | True    | internal  | http://keystone-api.openstack.svc.cluster.local:5000/v3  |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------------------------+
```
