# 安装Helm到K8S集群中

对于复杂应用中间件需要设置镜像运行的需求，环境变量，并且需要定制存储，网络等设置，以及设计和编写Deployment，ConfigMap,Service,及Ingress等相关YAML文件，再提交给Kubernetes进行部署，这些复杂的过程正在逐步被Helm应用包管理工具所实现。

## 1 基本概念

Helm是一个由CNCF孵化和管理的项目，用于对需要再K8S上部署复杂应用进行定义，安装和更新。Helm以Chart的方式对应用软件进行描述，可以方便地创建，版本化，共享和发布复杂的应用软件。

使用Helm会涉及一下几个术语：

- Chart：一个Helm包，其中包含了运行一个应用程序所需要的工具和资源定义，还可能包含Kubernetes集群中的服务定义，类似于Homebrew中的formula，apt中的dpkg，或者YUM中的rpm文件。
- Release：在K8S集群上运行一个Chart实例。在同一个集群上，一个Chart可以安装多次，例如有一个MySQL Chart，如果想在服务器上运行两个数据库，就可以基于这个Chart安装两次。每次安装都会生成新的Release，并有独立的Release名称。
- Repository：用于存放和共享Chart的仓库，类似于dockerhub

简单的说，Helm的任务是在仓库中查找需要的Chart，然后将Chart以Release的形式安装到K8S集群中。

## 2 安装Helm

Helm由以下两个组件组成

- HelmClient：客户端，拥有对Repository，Chart，Release等对象的管理能力
- TillerServer：负责客户端指令和K8S集群之间交互，根据Chart的定义生成和管理各种K8S的资源对象。

可以通过二进制文件或脚本方式安装HelmClient。

下载最新版二进制文件并压缩，下载地址为https://github.com/helm/helm/releases

```
wget https://get.helm.sh/helm-v2.16.0-linux-amd64.tar.gz
```

