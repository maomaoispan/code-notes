# [K8S（2）-集群搭建（Ubuntu版）](https://www.cnblogs.com/xlizi/p/14022963.html)

## 1. 介绍

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 ubuntu 20.04
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

## 2. 准备环境

| 角色   | IP              |
| ------ | --------------- |
| master | 192.168.153.150 |
| node1  | 192.168.153.151 |
| node2  | 192.168.153.152 |

**以下操作需要管理员权限**

#### 2.1 关闭防火墙

```bash
ufw disable
```

#### 2.2 关闭selinux

```bash
Ubuntu没有安装，不需要操作
```

#### 2.3 关闭swap

```bash
swapoff -a  # 临时

sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久
```

#### 2.4 设置主机名

```bash
hostnamectl set-hostname <hostname>
```

#### 2.5 在**master**中添加hosts

```bash
cat >> /etc/hosts << EOF
192.168.153.150 k8smaster
192.168.153.151 k8snode1
192.168.153.152 k8snode2
EOF
```

#### 2.6 将桥接的IPV4流量传递到iptables的链（Ubuntu默认全开放）

```bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system  # 生效
```

#### 2.7同步时间

```bash
apt install ntpdate

# 将系统时间与网络同步
ntpdate cn.pool.ntp.org

#将时间写入硬件
hwclock --systohc
```

## 3. 在所有节点上安装Docker、kubeadm、kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

#### 3.1 安装Docker

参考[Ubuntu安装docker](https://www.cnblogs.com/xlizi/p/13452547.html)

```bash

```

修改为国内镜像仓库

```bash
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

重启docker

```bash
service docker restart
```

#### 3.2 安装kubeadm，kubelet和kubectl

```bash
apt-get update && apt-get install -y apt-transport-https

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  

apt-get update

apt-get install -y kubelet kubeadm kubectl
```

#### 3.3 重新启动 kubelet

```bash
systemctl daemon-reload
systemctl restart kubelet
```

## 4. 部署Kubernetes Master

#### 4.1 在Master（192.168.153.150）执行。

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.153.150 \
  --image-repository registry.aliyuncs.com/google_containers \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

> 参数说明：
>
> | 参数                          | 说明                                                         |
> | ----------------------------- | ------------------------------------------------------------ |
> | --apiserver-advertise-address | API 服务器所公布的其正在监听的 IP 地址。如果未设置，则使用默认网络接口。 |
> | --image-repository            | 指定阿里镜像仓库                                             |
> | --service-cidr string         | 默认值："10.96.0.0/12"，为服务的虚拟 IP 地址另外指定 IP 地址段 |
> | --pod-network-cidr string     | 指明 pod 网络可以使用的 IP 地址段。如果设置了这个参数，控制平面将会为每一个节点自动分配 CIDRs。 |

> 安装失败重新安装时需要执行**重置命令**：
>
> ```bash
> kubeadm reset
> ```

#### 4.2 配置用户运行kubectl

使非 root 用户可以运行 kubectl，请运行以下命令， 它们也是 `kubeadm init` 输出的一部分：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

或者，如果你是 `root` 用户，则可以运行：

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### 4.3 查看节点

```
kubectl get nodes
```

## 5. 加入Kubernetes Node

在Node（192.168.153.151、192.168.153.151）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```bash
kubeadm join 192.168.1.11:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```bash
kubeadm token create --print-join-command
```

## 6. 部署CNI网络插件

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

从官方地址获取到flannel的yaml，在master1上执行

```bash
mkdir flannel
cd flannel
wget -c https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

安装flannel网络

```bash
kubectl apply -f kube-flannel.yml 
```

检查

```bash
kubectl get pods -n kube-system

NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s
```

## 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```bash
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：[http://NodeIP](http://nodeip/):Port

## 8. 清理

如果你在集群中使用了一次性服务器进行测试，则可以关闭这些服务器，而无需进一步清理。你可以使用 `kubectl config delete-cluster` 删除对集群的本地引用。

但是，如果要更干净地取消配置群集， 则应首先[清空节点](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain)并确保该节点为空， 然后取消配置该节点。

#### 8.1 删除节点

使用适当的凭证与控制平面节点通信，运行：

```bash
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
```

在删除节点之前，请重置 `kubeadm` 安装的状态：

```bash
kubeadm reset
```

重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

如果要重置 IPVS 表，则必须运行以下命令：

```bash
ipvsadm -C
```

现在删除节点：

```bash
kubectl delete node <node name>
```

如果你想重新开始，只需运行 `kubeadm init` 或 `kubeadm join` 并加上适当的参数。

#### 8.2 清理控制平面

你可以在控制平面主机上使用 `kubeadm reset` 来触发尽力而为的清理。

有关此子命令及其选项的更多信息，请参见[`kubeadm reset`](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-reset/)参考文档。





#### 9.镜像配置脚本

````
#!/bin/bash
# Script For Quick Pull K8S Docker Images
# by Hellxz Zhang <hellxz001@foxmail.com>

# please run kubeadm for get version msg. e.g `kubeadm config images list --kubernetes-version v1.18.3`
# then modified the Version's ENV, Saved and Run. 


# k8s.gcr.io/kube-apiserver:v1.21.0
# k8s.gcr.io/kube-controller-manager:v1.21.0
# k8s.gcr.io/kube-scheduler:v1.21.0
# k8s.gcr.io/kube-proxy:v1.21.0
# k8s.gcr.io/pause:3.4.1
# k8s.gcr.io/etcd:3.4.13-0
# k8s.gcr.io/coredns/coredns:v1.8.0





KUBE_VERSION=v1.21.0
PAUSE_VERSION=3.4.1
CORE_DNS_VERSION=1.8.0
ETCD_VERSION=3.4.13-0

# pull aliyuncs mirror docker images
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull docker.io/coredns/coredns:$CORE_DNS_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION

# retag to k8s.gcr.io prefix
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$KUBE_VERSION  k8s.gcr.io/kube-proxy:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$KUBE_VERSION k8s.gcr.io/kube-controller-manager:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:$KUBE_VERSION k8s.gcr.io/kube-apiserver:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$KUBE_VERSION k8s.gcr.io/kube-scheduler:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
docker tag docker.io/coredns/coredns:$CORE_DNS_VERSION k8s.gcr.io/coredns/coredns:v$CORE_DNS_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION


# untag origin tag, the images won't be delete.
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$KUBE_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$KUBE_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:$KUBE_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$KUBE_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns/coredns:v$CORE_DNS_VERSION
# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION
````

