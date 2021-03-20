---
title: 记一次虚拟机搭建 k8s 集群踩的坑
date: 2021-03-07 13:47:17
tags:
	- Kubernetes
	- Virtual Machine
categories:
	- 问题解决
---

## 背景

为了做毕设我用 VMware Workstation 创建了三台 Ubuntu Live Server 20.04.2 LTS 虚拟机来模拟一个三节点（Master * 1, Node * 2）的 k8s 集群，也成功地搭建了整个集群，然后我就将虚拟机挂机搁置了几天。当我重新打开三台虚拟机并在 master 上使用 `kubectl get nodes` 查看集群地时候，我发现两个 node 节点都处于 `Not Ready` 的状态，这明显不符合预期现象，于是我就把三台虚拟机都重启了（毕竟重启大法好），结果好家伙，这一重启什么都无了。

## 问题发现

一直盯着报错干瞪眼肯定是不可取的，于是我终于想起来可以看看运行日志。

```bash
service kubelet status
# 或者
journalctl -u kubelet.service -e
```

结果发现是因为 master 节点（以及 node 节点）因为重启 IP地址都变了！然后 APIServer 就找不到各个节点。所以我现在要做的就是把 IP 地址归位。

一开始我只是简单的用 `ifconfig` 命令把节点的 IP 地址强行改了，然后发现还是不行，估计是由于 APIServer 需要配置各种证书，然后 IPv4 网关和 DNS 什么的也都没有配，一大堆乱七八糟的，索性直接重新搭建集群，并给虚拟机分配静态地址。

## 问题解决

### 给虚拟机分配静态地址

此方法只针对 Ubuntu Live Server 20.04.2 LTS，不同 OS 或者 Ubuntu  版本可能略有差异，并且我的虚拟机网络采用的 NAT 模式。

这个做起来其实很简单，首先进入 VMware Workstation 的编辑 -> 虚拟网络编辑器 -> NAT 设置，记下网关 IP，比如我的是 `192.168.58.2`，然后宿主机侧就不需要干什么了。

进入虚拟机，编辑文件 `/etc/netplan/00-installer-config.yaml`:

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      # 要分配的静态地址
      addresses: [192.168.58.128/24]
      # 关闭 DHCP
      dhcp4: false
      # 网关地址，即上面的那个地址
      gateway4: 192.168.58.2
      nameservers:
      	  # 网关地址
            addresses: [192.168.58.2]
  version: 2
```

然后执行：

```bash
sudo netplan apply
```

这样便可给虚拟机指定一个静态的 IP 地址，也不影响虚拟机和宿主机的互相访问。然后对每个节点进行同样的操作，只要 IP 地址不一样就行。

### 重置 Kubernetes 集群

在所有节点上执行重置命令：

```bash
sudo kubeadm reset
```

在 master 节点上执行 init 命令，并重新创建 `.kube`目录。

```bash
sudo kubeadm init --image-repository=registry.aliyuncs.com/google_containers
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

然后再让 node 节点重新 join 集群就可以了。

```bash
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   40m   v1.20.4
k8s-node1    Ready    <none>                 28m   v1.20.4
k8s-node2    Ready    <none>                 28m   v1.20.4
```

## 补充

后来又发现每次恢复虚拟机的时候  `coredns` 的 pod 一直没法 Ready，查看日志发现是 dial 一直超时，目前还不知道原因是什么，解决办法是重启  `coredns` 的 Deployment：

```bash
kubectl rollout restart deployment coredns -n kube-system
```

不过这之后 kube-proxy 就没办法把所有来自集群外部的访问自动定向到相应的 Node 上了，这个也还不知道为什么。不过可以通过 Node 的 IP 地址访问部署到它上面的服务。我的网络使用的是 flannel，目前还没有仔细研究到底是哪里出了问题。

## 最后

Kubernetes的理念就是构建一个长期运行的系统，所以我这种 reset 的方法其实也是不太可取的办法，然后就是一定不要改变集群中节点的 IP 地址，要不然会非常麻烦。我这也算是体验了一把运维人员的工作？

还有就是，VMware Workstation Pro 16 推出了一个叫 `vctl`的工具，他会建立一个容器给我们创建一个 Powershell 环境，让我们可以在里面运行 [Kind](https://github.com/kubernetes-sigs/kind) 来模拟 k8s 集群，今天试了试，拉镜像有点慢，所以就先没有搞了，以后可以好好研究一下。下面附上几个用于参考的地址：

- https://docs.vmware.com/en/VMware-Workstation-Pro/16.0/com.vmware.ws.using.doc/GUID-1CA929BB-93A9-4F1C-A3A8-7A3A171FAC35.html （English）
- https://kind.sigs.k8s.io/docs/user/quick-start/ （English）
- https://qiita.com/YasuhiroABE/items/993a623c05d4dfe4d79c （日本語）

