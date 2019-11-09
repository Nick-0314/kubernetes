# 安装GFS到K8S集群中

在集群中，我们可以使用GFS，CEPH，NFS等为Pod提供动态持久化存储。本节动态存储主要介绍GFS的使用，静态存储方式采用NFS，其他类型的存储配置类似。

## 1 准备工作

### 1 为了保证Pod能够正常使用GFS作为后端存储，需要每台允许Pod的节点上提前安装GFS的客户端工具，其他存储方式也类似。

所有节点安装GFS客户端：

```
yum -y install glusterfs glusterfs-fuse
```

给需要作为GFS节点提供存储的节点打上标签

```
[root@master ~]# kubectl label node master storagenode=glusterfs 
```

输出信息

```
node/master labeled
```

另外两个节点

```
kubectl label node node1 storagenode=glusterfs 
node/node1 labeled
kubectl label node node2 storagenode=glusterfs 
node/node2 labeled
```

### 2 创建GFS集群

这里采用容器化方式部署GFS集群，同样也可以使用传统模式部署，在生产环境中，GFS集群最好是独立于集群之外进行部署，之后只需创建对应的Service和EndPoints即可。

本例部署采用DaemonSet方式，同时保证已经打上标签的节点上哦都运行一个GFS服务，并且均有提供存储的磁盘。

下载向关安装文件：

```
wget https://github.com/heketi/heketi/releases/download/v9.0.0/heketi-client-v9.0.0.linux.amd64.tar.gz
```

创建集群

```
tar -xvf heketi-client-v9.0.0.linux.amd64.tar.gz
cd heketi-client/share/heketi/kubernetes/
```

由于kubernetes版本原因需要修改yaml文件

```
vim glusterfs-daemonset.json
```

![](image/GFS/xg1.png)

```
        "selector": {
            "matchLabels": {
                "glusterfs-node": "daemonset"
            }
        },
```

应用yaml文件

```
kubectl create -f glusterfs-daemonset.json
```

输出信息

```
daemonset.apps/glusterfs created
```

注意

1. 此处创建的时默认的挂载方式，可使用其他磁盘作为GFS的工作目录
2. 此处创建的NameSpace为default，可按需修改
3. 可使用gluster/gluster-centos:gluster3u12_centos7镜像

查看GFS Pods

```
kubectl get pods -l glusterfs-node=daemonset
```

输出信息

```
NAME              READY   STATUS    RESTARTS   AGE
glusterfs-sv5b4   1/1     Running   0          5m4s
glusterfs-tfchd   1/1     Running   0          5m4s
glusterfs-xxr7w   1/1     Running   0          5m4s
```

### 3 创建Heketi服务

Heketi是一个提供RESTful API管理GFS卷的框架，能够在Kubernetes,OpenShift,OpenStack等云平台上实现动态存储资源供应，支持GFS多集群管理，便于管理员对GFS进行操作，在Kubernetes集群中，Pod将存储的请求发送至Heketi，然后Heketi控制GFS集群创建对应的存储卷。

创建Heketi的ServiceAccount对象（在上一节的基础上 目录没有切换）

```
kubectl create -f heketi-service-account.json
```

输出信息

```
serviceaccount/heketi-service-account created
```

```
kubectl get sa
```

输出信息

```
NAME                     SECRETS   AGE
default                  1         9d
heketi-service-account   1         102s
```

创建Heketi对应的权限和Secret：

```
kubectl create  clusterrolebinding heketi-gluster-admin --clusterrole=edit --serviceaccount=default:heketi-service-account
```

输出信息

```
clusterrolebinding.rbac.authorization.k8s.io/heketi-gluster-admin created
```

```
kubectl create secret generic heketi-config-secret --from-file=heketi.json
```

输出信息

```
secret/heketi-config-secret created
```

初始化部署Heketi:

由于kubernetes版本原因 需要修改json文件

```
vim heketi-bootstrap.json
```

![](image/GFS/xg2.png)

应用json文件

```
kubectl apply -f heketi-bootstrap.json
```

输出信息

```
service/deploy-heketi created
deployment.apps/deploy-heketi created
```

### 4 创建GFS集群

本节使用Heketi创建GFS集群，其管理方式更加简单和高效

复制heketi-cli至/usr/local/bin:

依然在上一节所在的目录内操作

```
cd ../../../bin/
cp -p heketi-cli /usr/local/bin/
heketi-cli -v
```

输出信息

```
heketi-cli v9.0.0
```

修改topology-sample，manage为GFS管理服务的节点(Node)主机名，storage为节点IP地址，devices为节点上的裸设备，也就是用于提供存储的磁盘最好使用裸设备：

```
cd ../share/heketi/kubernetes/
vim topology-sample.json
```

修改为下面示例

```
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "master"
              ],
              "storage": [
                "192.168.10.10"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sda",
              "destroydata": false
            }
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node1"
              ],
              "storage": [
                "192.168.10.11"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sda",
              "destroydata": false
            }
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node2"
              ],
              "storage": [
                "192.168.10.12"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sda",
              "destroydata": false
            }
          ]
        }
      ]
    }
  ]
}
```

查看当前Heketi的ClusterIP

```
kubectl get svc | grep heketi
```

输出信息

```
deploy-heketi    ClusterIP   10.250.0.181   <none>        8080/TCP   10m
```

```
export HEKETI_CLI_SERVER=http://10.250.0.181:8080
```

使用Heketi创建GFS集群

```

```

