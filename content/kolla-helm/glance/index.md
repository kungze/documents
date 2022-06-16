---
"title": "部署 glance"
"weight": 4
---

目前 kolla-helm 支持部署两种 glance 后端存储：file 和 ceph。

## 部署 glance

- 使用 file 做为存储后端
```shell
helm install -n openstack openstack-glance kolla-helm/glance --set ceph.enabled=false
```
创建的 image 文件，将存储在 pod 名称为“ glance-api-xxxx” 的 `/var/lib/glance/images/` 目录下，默认通过 hostPath 的方式映射到宿主机的 `/var/lib/glance/images/` 目录下。

- 使用 ceph 做为存储后端

特别注意的是，ceph 后端依赖 [rook](https://github.com/rook/rook)，如果想使用 ceph 做为存储后端，需要在 

部署 glance 之前安装 rook，并用 rook 创建一个 ceph 集群或使用 rook 纳管一个外部的 ceph 集群。

```shell
helm install -n openstack openstack-glance kolla-helm/glance \
    --set ceph.cephClusterNamespace=rook-ceph \
    --set ceph.cephClusterName=rook-ceph
```
`ceph.cephClusterNamespace` 和 `ceph.cephClusterName` 需要根据具体情况进行修改，默认情况下是使用 ceph 做为存储后端。

## 验证

通过下面命令观察
```shell
watch -n 1 kubectl -n openstack get pods -l app.kubernetes.io/instance=openstack-glance
```
等待所有的 pod 都 ready 后，创建 image 检验 glance 的功能是否正常。

### 下载 cirros 镜像

```shell
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

### 创建镜像

```shell
openstack image create "cirros" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```

### 查看镜像

```shell
openstack image list
+--------------------------------------+-----------+--------+
| ID                                   | Name      | Status |
+--------------------------------------+-----------+--------+
| 4b525cef-840b-4533-af05-91ae4acfc3f2 | cirros    | active |
+--------------------------------------+-----------+--------+
```
