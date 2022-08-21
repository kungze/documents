---
"title": "cinder-metal-csi 使用文档"
"weight": 2
---

cinder-metal-csi 支持 Dynamic Provisioning、Block Volume、Volume Expansion、Volume Cloning、Volume Snapshots等特性。

## Dynamic Provisioning

Dynamic Provisioning 使用持久化卷(PVC)请求 Kuberenetes 创建 Cinder 卷，并从容器内部使用该卷。

创建 nginx pod 测试挂载 rbd 类型的 pvc

```BASH
kubectl apply -f example/nginx-rbd.yaml

# 查看 pvc 状态
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
cinder-pvc-rbd                         Bound    pvc-6ffafe41-1215-4ccc-af0c-efb1b4507f87   2Gi        RWO            cinder-metal-csi-rbd   17s

# 查看 pod 状态
$ kubectl get pod
NAME                                                             READY   STATUS      RESTARTS      AGE
nginx-rbd                                                        1/1     Running     0             20s

# 查看 cinder 卷信息
$ cinder list
+--------------------------------------+-----------+------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status    | Name                                     | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+-----------+------------------------------------------+------+-------------+----------+--------------------------------------+
| f66d4f41-ab9b-4e37-aa5a-8b601e92a6cb | in-use    | pvc-6ffafe41-1215-4ccc-af0c-efb1b4507f87 | 2    | rbd         | false    | None                                 |
+--------------------------------------+-----------+------------------------------------------+------+-------------+----------+--------------------------------------+

# 在 pod 中写入文件测试
$ kubectl exec -it nginx-rbd bash

# 查看已挂载的文件系统，如下所示，块设备挂载到了  /var/lib/www/html 目录
$ df -Th
Filesystem     Type     Size  Used Avail Use% Mounted on
overlay        overlay  549G   53G  474G  11% /
tmpfs          tmpfs     64M     0   64M   0% /dev
tmpfs          tmpfs    252G     0  252G   0% /sys/fs/cgroup
/dev/sdx1      ext4     549G   53G  474G  11% /etc/hosts
shm            tmpfs     64M     0   64M   0% /dev/shm
/dev/rbd2      ext4     2.0G   24K  1.9G   1% /var/lib/www/html
tmpfs          tmpfs    504G   12K  504G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs          tmpfs    252G     0  252G   0% /proc/acpi
tmpfs          tmpfs    252G     0  252G   0% /proc/scsi
tmpfs          tmpfs    252G     0  252G   0% /sys/firmware

# 写入文件测试
$ echo `date` > /var/lib/www/html/index.html
$ cat /var/lib/www/html/index.html
Fri Aug 26 01:18:41 UTC 2022
```

## Block Volume

Cinder 卷作为块设备暴露在容器中，而不是作为一个挂载的文件系统。

使用本特性的前提条件:

- 确保持久化卷 Spec 文件中的 [volumeMode](https://github.com/kungze/cinder-metal-csi/blob/example/example/block/nginx.yaml#L18) 是 Block
- 确保使用块设备 PVC 的 pod 使用的是 [volumeDevices](https://github.com/kungze/cinder-metal-csi/blob/example/example/block/nginx.yaml#L37) 而不是 volumounts

创建使用 Block volume 特性的 pod

```BASH
$ kubectl apply -f example/block/nginx.yaml

#查看 pod 的状态
$ kubectl get pod
NAME                                                             READY   STATUS      RESTARTS      AGE
nginx-block                                                       1/1     Running     0             36s

# 检查块设备是否挂载在容器内
$ kubectl exec -it nginx-block bash
$ root@test-block:/# ls -al /dev/xvda
brw-rw---- 1 root disk 252, 32 Aug 21 10:36 /dev/xvda
```

## Volume Expansion

cinder-metal-csi 插件支持在线和离线两种方式调整 cinder 卷大小。
前提条件:
- 需在 Storageclass Spec 中将 [allowVolumeExpansion](https://github.com/kungze/cinder-metal-csi/blob/example/example/resize/resize.yaml#L7) 设置为true

创建 pod 测试扩容卷

```BASH
$ kubectl apply -f example/resize/resize.yaml

# 查看 pvc 信息
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
pvc-resize-demo                        Bound    pvc-07a946ca-5aeb-4c03-b2f9-cf4cace05ae2   2Gi        RWO            cinder-csi-rbd         19s

# 查看 pod 信息
$ kubectl get pod
NAME                                                             READY   STATUS      RESTARTS      AGE
nginx-resize                                                     1/1     Running     0             16s

# 修改 pvc 的存储大小为10Gi
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

# 查看调整后的 pvc 信息
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
pvc-resize-demo                        Bound    pvc-64b1e67e-870b-4073-aa7e-659b46c413e0   10Gi       RWO            cinder-csi-rbd    125m

# 查看调整后的 cinder 卷大小
cinder list
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status         | Name                                     | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+
| fdacff2a-e73e-4764-8545-9a0ae4909ce7 | in-use         | pvc-64b1e67e-870b-4073-aa7e-659b46c413e0 | 10   | rbd         | false    | None                                 |
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+

# 进入 pod 验证文件系统大小，如下所示，挂载的块设备的容量已变成10G
$ kubectl exec -it nginx-resize  bash
root@nginx-resize:/# df -Th
Filesystem     Type     Size  Used Avail Use% Mounted on
/dev/rbd2      ext4     9.8G   24K  9.8G   1% /var/lib/www/html
```

## Volume Cloning

这个特性支持从 Kubernetes 现有的 PVC 中克隆卷
前提条件:
- PVC 的状态必须是bound and available (not in use).
- 源和目标 pvc 必须在相同的名称空间中.
- 克隆只支持在相同的存储类中。目标卷必须与源卷具有相同的存储类

创建source pvc

```BASH
$ kubectl apply -f example/clone/source-pvc.yaml
storageclass.storage.k8s.io/cinder-csi-rbd created
persistentvolumeclaim/pvc-source-demo created

# 查看 pvc 的状态
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
pvc-source-demo                        Bound    pvc-e579508b-0bce-4082-b91e-b9b3db2d15f9   1Gi        RWO            cinder-csi-rbd         34s

# 查看 volume 卷的状态
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status         | Name                                     | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+
| bdc8e5da-be35-4c8b-b57c-f691fc7a32da | available      | pvc-e579508b-0bce-4082-b91e-b9b3db2d15f9 | 1    | rbd         | false    |                                      |
+--------------------------------------+----------------+------------------------------------------+------+-------------+----------+--------------------------------------+
```

根据 source pvc， 创建 clone pvc
```BASH
$ kubectl apply -f example/clone/clone-pvc.yaml
kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
pvc-clone-demo                         Bound    pvc-60228500-5773-49b9-b117-b46c8c4b01b8   1Gi        RWO            cinder-csi-rbd         4s
```

## Volume Snapshots

该特性支持创建卷快照和从快照恢复卷
前提条件:
- 环境中已经安装了 [snapshot CRD](https://github.com/kubernetes-csi/external-snapshotter/tree/master/client/config/crd)
- 环境中已经安装了 [snapshot controller](https://github.com/kubernetes-csi/external-snapshotter/tree/master/deploy/kubernetes/snapshot-controller)

部署应用程序，创建存储类，快照类和 PVC

```BASH
$ kubectl apply -f example/snapshot/example.yaml

# 查看卷快照存储类
$ kubectl get volumesnapshotclasses.snapshot.storage.k8s.io
NAME                    DRIVER             DELETIONPOLICY   AGE
cinder-snapshot-class   cinder.metal.csi   Delete           35m

# 查看 pvc 的状态
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
pvc-snapshot-demo                      Bound    pvc-6d592494-1063-4fa4-8523-b6043ad638e5   1Gi        RWO            cinder-csi-rbd         17s

# 创建卷快照
$ kubectl apply -f example/snapshot/snapshotcreate.yaml
volumesnapshot.snapshot.storage.k8s.io/new-snapshot-demo created

# 查看卷快照的状态
$ kubectl get volumesnapshot
NAME                READYTOUSE   SOURCEPVC           SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS           SNAPSHOTCONTENT                                    CREATIONTIME   AGE
new-snapshot-demo   true         pvc-snapshot-demo                           1Gi           cinder-snapshot-class   snapcontent-581843f9-c78c-4f36-b387-ef949886c819   11m            16m

# 查看 volumesnapshotcontents 

$ kubectl get volumesnapshotcontents.snapshot.storage.k8s.io
NAME                                               READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER             VOLUMESNAPSHOTCLASS     VOLUMESNAPSHOT      VOLUMESNAPSHOTNAMESPACE   AGE
snapcontent-581843f9-c78c-4f36-b387-ef949886c819   true         1073741824    Delete           cinder.metal.csi   cinder-snapshot-class   new-snapshot-demo   default                   19m

# 查看 cinder snapshot volume 
$ openstack volume snapshot list
+--------------------------------------+-----------------------------------------------+----------------------------------------+-----------+------+
| ID                                   | Name                                          | Description                            | Status    | Size |
+--------------------------------------+-----------------------------------------------+----------------------------------------+-----------+------+
| 727eef0c-6be0-4144-bb4d-998ad8397769 | snapshot-dfbd8ce1-70e1-4a3e-9e2a-6aa25e706c96 | Created by OpenStack Cinder CSI driver | available |    1 |
+--------------------------------------+-----------------------------------------------+----------------------------------------+-----------+------+

```

从快照恢复卷

```BASH
$ kubectl apply -f example/snapshot/snapshotrestore.yaml

# 验证从快照创建的卷是否创建成功
$ kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
restore-volume      Bound    pvc-8e9ecb97-170f-443d-b9a8-f119c8eb7f19   1Gi        RWO            csi-sc-cinderplugin   21s
```
