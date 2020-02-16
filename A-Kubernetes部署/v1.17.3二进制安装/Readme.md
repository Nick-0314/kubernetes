# 本文介绍

本文将介绍使用二进制安装部署kubernetes v1.17.3 集群，部署文档将会简单逻辑各组件的功能，给出详细部署过程、启动参数及说明，二进制安装文档网上已经有很多，在这里参考网上的资料，并加入个人理解形成此学习文档，作为总结，方便后续查看。在这里说明一下，线上环境不建议使用二进制安装了，升级过程很麻烦，线上建议使用kubeadm安装部署。

**组件说明**

**etcd**

| etcd     |                                                              |
| -------- | ------------------------------------------------------------ |
| 版本     | v3.4.1                                                       |
| 下载     | https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-amd64.tar.gz |
| 高可用   | 部署奇数台服务器，以便leader选主                             |
| 功能     | 1. 使用raft一致性算法来实现的一款分布式KV存储； 2. 用于存储kubernetes集群中的配置、服务发现、各对对象的状态以及元数据信息等； 3. 用于存储网络插件flannel、calico等网络配置信息； |
| 注意事项 | 1.默认为v3版本的API  2. kubernetes集群需要使用v3版本的API；  |



**Docker引擎**

| Docker引擎 |                                                              |
| ---------- | ------------------------------------------------------------ |
| 版本       | 19.03.6                                                      |
| 下载       | yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo &&yum makecache && yum -y install docker-ce |
| 功能       | 1. 提供容器操作的底层引擎 2. 为kubernets中Pod提供容器；      |
| 注意事项   | 1.需要配合flannel提供的网络插件一起使用                      |



**kube-apiserver** 

| kube-apiserver |                                                              |
| -------------- | ------------------------------------------------------------ |
| 版本           | 1.17.3                                                       |
| 下载           | https://storage.googleapis.com/kubernetes-release/release/v1.17.3/bin/linux/amd64/kube-apiserver |
| 高可用         | 水平扩展（横向扩展）实现                                     |
| 功能           | 1. kube-apiserver负责与etcd交互，其它组件不与etcd交互； 2. 对集群进行操作和管理都需要通过kube-apiserver组件提供的API来完成； 3. 集群中各组件不会相互调用，都是通过kube-apiserver组件来完成； 4. kube-apiserver开启bootstrap token认证，支持kubelet TLS bootstrapping，确保集群安全； 5. 为避免其它组件频繁查询apiserver，apiserver为其它组件提供了watch接口，用于监视资源的增删改操作； |
| 注意事项       | 1. 关闭非安全端口8080和匿名访问； 2. 使用安全端口6443； 3. 使用nginx 4层代理与keepalive实现高可用； |



**kube-controller-manager**

| kube-controller-manager |                                                              |
| ----------------------- | ------------------------------------------------------------ |
| 版本                    | 1.17.3                                                       |
| 下载                    | https://storage.googleapis.com/kubernetes-release/release/v1.17.3/bin/linux/amd64/kube-controller-manager |
| 高可用                  | leader选举实现，只有leader处于运行状态，其它副本去竞争leader； |
| 功能                    | 1. 它是控制器管理器，逻辑上每个控制器都有一个单独的协程；  2. 每个控制器，负责监视（watch）apiserver暴露的集群状态，并不断地尝试把当前状态向所期待的状态迁移；  3. 默认非安全端口10252，安全端口10257；  4. 使用kubeconfig访问kube-apiserver安全端口；  5. 使用approve kubelet证书签名请求（CSR），证书过期后自动轮转；  6. kube-controller-manager各节点使用自己的ServiceAccount访问kube-apiserver；  7. kube-controller-manager 3节点高可用，去竞争leader锁，成为leader； |
| 注意事项                | 1. 多副本情况下需要--leader-elect=true启动参数；  2. 安全端口和非安全端口都需要打开；  3. kubectl get cs使用的是http； |



**kube-scheduler**

| kube-scheduler |                                                              |
| -------------- | ------------------------------------------------------------ |
| 版本           | 1.17.3                                                       |
| 下载           | https://storage.googleapis.com/kubernetes-release/release/v1.17.3/bin/linux/amd64/kube-scheduler |
| 高可用         | leader选举实现，只有leader处于运行状态，其它副本去竞争leader |
| 功能           | 1. 使用kubeconfig访问kube-apiserver安全端口  2. kube-scheduler 3节点高可用，自选举；  3. 它监视kube-apiserver提供的watch接口，找一个最佳适配的节点，然后创建资源，如Pod资源;  4. 需要根据预选（Predicates）与优选（Priorities）两个环节找最佳适配节点；  5. 预选策略是过滤掉那些不满足Polices的节点，预选的输出作为优选的输入；  6. 优选策略是根据预选后的节点进行打分排名，得分最高的节点作为合适的节点，要创建的资源即被调到到此节点；  7. 默认非安全端口10251，安全端口10259； |
| 注意事项       | 1. 多副本情况下需要--leader-elect=true启动参数；  2. 安全端口和非安全端口都需要打开；  3. kubectl get cs使用的是http； |



**kubelet**

| kubelet |                                                              |
| ------- | ------------------------------------------------------------ |
| 版本    | 1.17.3                                                       |
| 下载    | https://storage.googleapis.com/kubernetes-release/release/v1.17.3/bin/linux/amd64/kubelet |
| 高可用  | 每台机器部署，不需要高可用，节点故障会自动迁移到其它节点     |
| 功能    | 1. 每个Node节点上面都可以接受master节点下发的任务，这些任务由kubelet来管理，主要用来管理Pod和其中的容器；  2. kubelet会在API Server上注册节点信息，它内置了cAdvisor，通过它监控容器和节点资源，并定期向Master节点汇报资源使用情况；  3. 使用kubeadm动态创建bootstrap token；  4. 使用TLS bootstrap机制自动生成Client和Server证书，过期后自动轮转；  5. 默认非安全端口10248，安全端口10250；  6. 安全端口接受https请求，对请求进行认证和授权；  7. 使用kubeconfig访问kube-apiserver的安全端口 |



**kube-proxy**

| kube-proxy |                                                              |
| ---------- | ------------------------------------------------------------ |
| 版本       | 1.17.0                                                       |
| 下载       | https://storage.googleapis.com/kubernetes-release/release/v1.17.3/bin/linux/amd64/kube-proxy |
| 高可用     | 每台机器部署，不需要高可用，节点故障会自动迁移到其它节点     |
| 功能       | 1. service是一组Pod的抽象，相当于一组pod的负载均衡器，负责将请求分发到对应的pod；  2. kube-proxy就是负责service实现的，内部请求到达service，它通过label关联并转发到后端某个pod;  3. 外部的node port向service的访问，通过label关联并转发到后端的某个pod；  4. kube-proxy提供了三种负载均衡模式（代理模式），用户态（userspace）、iptables、ipvs模式；  5. service的服务暴露方式由NodePort、hostNetwork、LoadBalancer、Ingress；  6. 使用kubeconfig访问kube-apiserver的安全端口；  8. 默认非安全端口10249，安全端口10256； |



Calico

| Flanneld |                                                              |
| -------- | ------------------------------------------------------------ |
| 版本     | 3.10                                                         |
| 下载     | curl https://docs.projectcalico.org/v3.10/manifests/calico.yaml -O |
| 高可用   | 每台机器部署，不需要高可用，节点故障会自动迁移到其它节点     |
| 功能     | 1. 为每台Node节点分配Pod网段  2. 为集群Service网络分配IP（如有需要）  3. Calico只是做为网络插件的一种，还有很多； |
| 注意事项 | 镜像可能需要手动下载；                                       |



**插件CoreDNS**

| 插件CoreDNS |                                                              |
| ----------- | ------------------------------------------------------------ |
| 版本        |                                                              |
| 下载        |                                                              |
| 高可用      | 以Pod的方式部署在集群当中，部署2个副本                       |
| 功能        | 1. 通过监听service与endpoints的变更事件，将域名和IP对应信息同步到CoreDNS配置中； 2. CoreDNS中ClusterIP，这个IP是虚拟的，具有TCP/IP协议栈，早期版本无法ping通些IP（这个是iptables规则没有允许icmp通过），新版本中可以ping通； |
| 注意事项    | Pod副本数，根据需要自行调整；                                |



**总结**

本文主要讲解了安装kubenetes二进制安装时所使用到的组件版本、下载链接、高可用实现方式、简单的功能说明及注意事项；