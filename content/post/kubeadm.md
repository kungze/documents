---
"title": 在国内使用 kubeadm 部署 k8s
"weight": 1
---

[kubeadm](https://github.com/kubernetes/kubeadm) 是 k8s 官方提供的一个部署管理 k8s 集群的工具，但是 kubeadm 使用到的很多资源的下载地址都在国外，如果按照官方文档操作很容易因为网络原因失败。这里基于 ubuntu 20.04 展示如何让 kubeadm 使用国内的资源部署 k8s 集群。

下面所有命令都是使用 root 用户执行的

## 安装 docker

参照 docker 官方[安装文档](https://docs.docker.com/engine/install/ubuntu/)

```console
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ apt-get update
$ apt-get install docker-ce docker-ce-cli containerd.io
```

通过阿里源安装：

```console
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ apt-get update
$ apt-get install docker-ce docker-ce-cli containerd.io
```

## 安装 kubeadm

官方文档使用的 <https://apt.kubernetes.io/> apt 源在国内被屏蔽了，所以我们需要找一个国内的镜像源，这里我们以阿里源为例

```console
apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
```

在安装之前可以通过下面命令我们可以安装的 kubeadm 的版本

```console
apt-cache madison kubeadm
```

根据情况选择适合的版本安装

```console
apt-get install kubectl=1.23.7-00 kubelet=1.23.7-00 kubeadm=1.23.7-00
```

**注意**：这里选择的版本决定了最终安装的 k8s 的版本，从 1.24 开始 k8s 从代码中彻底移除了 dockershim，所以我们这里选择用 k8s 1.23.7，如果想在 k8s 1.24 及以后使用 docker 作为 cri，则需要安装一个外部的 dockershim，mirantis 的 [cri-dockerd](https://github.com/Mirantis/cri-dockerd) 是一个不错的选择。

## 节点准备

关闭 swap 分区 ( 貌似最新的 k8s 版本不在要求这一步 )

```console
sudo swapoff -a # 暂时关闭，永久关闭可以上网查询
```

## 初始化环境

创建一个 yaml 文件，我们命名为 kubeadm-init-config.yaml，填入以下内容

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- token: abcdef.0123456789abcdef
  ttl: 24h0m0s
localAPIEndpoint:
  advertiseAddress: 172.18.30.127
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  imagePullPolicy: IfNotPresent
  taints: []
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  timeoutForControlPlane: 4m0s
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kubernetesVersion: 1.23.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
address: 0.0.0.0
enableServer: true
cgroupDriver: cgroupfs
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  strictARP: true
```

这里有几个参数要特别注意一下：

* `advertiseAddress`：这个要改为当前主机的管理网 IP 地址
* `imageRepository`：这是一个关键的配置，从国内源下载相关镜像
* `podSubnet`: 这个是下面部署的 flannel cni 插件要求的一个参数，需要和 flannel 网络配置 `net-conf.json` 中的 `Network` 保持一致
* `cgroupDriver`: 这个需要和 docker 的 Cgroup Driver 保持一致，可以通过命令 `docker info|grep "Cgroup Driver"` 查看

然后执行：

```console
sudo kubeadm init --config kubeadm-init-config.yaml
```

如果你不想关闭 swap 分区，使用下面命令初始化

```console
sudo kubeadm init --config kubeadm-init-config.yaml --ignore-preflight-errors Swap
```

等待一会儿，初始化成功后会获得如下输出：

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.18.30.127:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:159072d62927901183f5cda29fbb4e110ee8e354a350aad7774239e19c57169a
```

按照输出提示执行下面命令配置 kubeconfig 以便后面我们能正常使用 kubectl 命令

```console
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 安装网络插件

在安装网络插件之前我们查看一下 node 信息

```console
# kubectl get nodes
NAME          STATUS     ROLES    AGE    VERSION
yjf-kubeadm   NotReady   master   4h8m   v1.19.16
```

发现 node 的状态为 NotReady

通过下面命令安装 flannel 网络插件

```console
$ sudo kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

等一会儿，大概 5 分钟之后在查看 node 信息

```console
# kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
kubeadm   Ready    master   4h20m   v1.19.16
```

node status 已经变为 ready 了

## 添加节点

在新的节点重新执行 **安装 docker**，**安装 kubeadm**，**节点准备** 三个步骤。

然后执行

```console
kubeadm join 172.18.30.127:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:a86f804bc33460936ce8c6a9bdde774190815fafcd5a093c0d53a8f7bfc72ad3
```

执行完成后在第一个节点查看 nodes

```console
# kubectl get nodes -o wide
NAME           STATUS   ROLES    AGE     VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kubeadm    Ready    master   42m     v1.19.16   172.18.30.127   <none>        Ubuntu 20.04.3 LTS   5.11.0-34-generic   docker://20.10.12
kubeadm2   Ready    <none>   4m43s   v1.19.16   172.18.30.151   <none>        Ubuntu 20.04.3 LTS   5.11.0-34-generic   docker://20.10.12
```
