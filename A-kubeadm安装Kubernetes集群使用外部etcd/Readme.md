# This is mytting .

本文以kubeadm工具部署kubernetes集群为基础，讲解了如何使用外部etcd集群部署kubernetes。

etcd作为kubernetes集群的数据库，里面保存了整个集群状态信息，重要性不言而喻，完全基于kubeadm的安装方式中，将etcd数据库作为一个Pod来运行，没有做到高可用，如果主节点所在的主节因不可抗拒因素无法恢复数据，则整个kubernetes集群将无法恢复，即便做到多master节点的集群，其每个master节点中的etcd互相不通信，无法利用etcd集群的高可用性，集群某个主节点宕机后，很难恢复集群状态。

而本文先在kubernetes集群外部搭建了一个etcd集群，本文采用三个etcd节点，其中任意一个etcd节点宕机后，etcd集群会重新选举出主节点，集群仍然可用，不过宕机的etcd数量不能大于节点数量的半数，只能小于半数，在实际使用中，可以采用ssd的硬盘作为存储介质，提高读写效率。也可以搭建5个或者7个etcd节点，那么5个节点的etcd集群可以容忍两个节点宕机，7个etcd节点的集群可以容忍3个节点的宕机后集群仍然可用。

外部etcd集群更容易做数据备份，迁移等操作，相较于纯kubeadm的安装方法，有着很大的好处。

具体安装步骤请见

1. [环境准备](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-A%20%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87.md)
2. [下载命令](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-B%20%E4%B8%8B%E8%BD%BD%E5%91%BD%E4%BB%A4.md)
3. [生产证书](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-C%20%E7%94%9F%E6%88%90%E8%AF%81%E4%B9%A6.md)
4. [部署etcd集群](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-D%20%E9%83%A8%E7%BD%B2etcd%E9%9B%86%E7%BE%A4.md)
5. [下载kubernetes工具](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-E%20%E4%B8%8B%E8%BD%BDkubernetes%E5%B7%A5%E5%85%B7.md)
6. [使用kubeadm初始化集群](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-F%20%E5%88%9D%E5%A7%8B%E5%8C%96%E9%9B%86%E7%BE%A4.md)
7. [从节点加入集群](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-G%20%E4%BB%8E%E8%8A%82%E7%82%B9%E5%8A%A0%E5%85%A5%E9%9B%86%E7%BE%A4.md)
8. [配置网络插件](https://github.com/mytting/kubernetes/blob/master/A-kubeadm%E5%AE%89%E8%A3%85Kubernetes%E9%9B%86%E7%BE%A4%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8etcd/v1.16.3-H%20%E9%85%8D%E7%BD%AE%E7%BD%91%E7%BB%9C%E6%8F%92%E4%BB%B6.md)