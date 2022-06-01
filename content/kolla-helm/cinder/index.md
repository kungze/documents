---
"title": "部署 cinder"
"weight": 4
---

目前 kolla-helm 支持部署两种 cinder 后端：lvm 和 ceph。但是 lvm 仅用于测试，正式使用推荐 ceph。

## 部署 cinder

* 仅安装 lvm 后端

```shell
helm install -n openstack openstack-cinder kolla-helm/cinder --set ceph.enabled=false
```

注意：lvm 后建议仅用于测试，cinder chart 会创建一个 loop 设备作为 lvm 的 pv 设备，可以通过
参数 ``lvm.loop_device_directory`` 和 ``lvm.loop_device_size`` 来指定 loop 文件在宿主机上
的存放目录和大小。

* 仅安装 ceph 后端

特别注意的是，ceph 后端依赖 [rook](https://github.com/rook/rook)，如果想开启 ceph 后端，需要在
部署 cinder 之前安装 rook，并用 rook 创建一个 ceph 集群或使用 rook 纳管一个外部的 ceph 集群。

```shell
helm install -n openstack openstack-cinder kolla-helm/cinder \
    --set lvm.enabled=false \
    --set ceph.cephClusterNamespace=rook-ceph \
    --set ceph.cephClusterName=rook-ceph
```

``ceph.cephClusterNamespace`` 和 ``ceph.cephClusterName`` 需要根据具体情况进行修改，默认情况下如果开启了 ceph 后端，cinder 的 backup 服务也会开启，如果
想关闭，需要设置 ``ceph.backup.enabled`` 为 false。

* 同时安装 lvm，ceph 后端

默认情况下（不改变任何参数） lvm 和 ceph 后端同时开启

```shell
helm install -n openstack openstack-cinder kolla-helm/cinder
```

## 验证

通过下面命令观察

```shell
watch -n 1 kubectl -n openstack get pods -l app.kubernetes.io/instance=openstack-cinder
```

等待所有的 pod 都 ready 后，创建 volume type 和 volume 检验 cinder 的功能是否正常。

### 创建默认 volume type

如果开启了 lvm 后端，默认 volume type 为 lvm；如果只开启了 ceph 后端或者 ceph 和 lvm 后端同时开启， 默认 volume type 为 rbd。

```shell
openstack volume type create <默认 volume type 名称>
```

### 创建卷

```shell
cinder create 1 --name vol-test
```
