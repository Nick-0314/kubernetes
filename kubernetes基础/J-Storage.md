# Storage

本节介绍Kubernetes Storage的相关概念及使用，一般做持久化或者有状态的应用程序才会用到Storage。

## 1 Volumes

Containers（容器）中的磁盘文件是短暂的，当容器崩溃时，kubelet会重新启动容器，但最初的文件将丢失，Container会以最干净的状态启动。另外，当一个Pod运行多个Container时，各个容器可能需要共享一些文件。kubernetes Volume可以解决这两个问题。

1 背景

Docker也有卷的概念，但是在Docker中卷只是磁盘上或另一个Container中的目录，其生命周期不受管理。虽然目前Docker已经提供了卷驱动程序，但是功能非常有限，例如从Docker1.7版本开始，每个Container只允许一个卷驱动程序，并且无法将参数传递给卷，

另一方面，Kubernetes卷具有明确的生命周期，与使用它的Pod相同。因此,在Kubernetes中的卷可以比Pod中运行的任何Container都长，并且可以在Container重启或者销毁之后保留数据。Kubernetes支持多种类型的卷，Pod可以同时使用任意数量的卷.

从本质上讲，卷只是一个目录，可能包含一些数据，Pod中的容器可以访问它。要使用卷Pod需要通过.spec.volumes字段指定为Pod提供的卷，以及使用.spec.containers.volumeMounts字段指定卷挂载的目录。从容器中的进程可以看到由Docker镜像和卷组成的文件系统视图，卷无法挂载其他卷或具有到其他卷的硬链接，Pod中的每个Container必须独立指定每个卷的挂载位置。

2 卷的类型

Kubernetes支持的卷的类型有很多，以下为常用的卷.

1 awsElasticBlockStore（EBS）只适用于亚马逊云产品

awsRlasticBlockStore卷挂载一个AWS EBS Volume到Pod中，与emptyDir卷不同的是，当移除Pod时EBS卷的内容不会被删除，这意味着可以将数据预先放置在EBS卷中，并且可以在Pod之间切换该数据。

使用aws ElasticBlockStore卷的限制

- 运行Pod的节点必须是AWS EC2 实例。
- AWS EC2 实例需要和EBS卷位于同一区域和可用区域。
- EBS仅支持挂载卷的单个EC2实例(一对一)。

在将Pod与EBS卷一起使用之前，需要先创建EBS卷，确保该卷的区域与集群的区域匹配，并检查size和EBS卷类型是否合理：

```
aws ec2 create-volume --availability-zone=eu-west-la --size=10 --vilume-type=gp2
```

AWS EBS 实例配置：

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - name: test-webserver
    image: nginx
    volumeMounts:
    - name: test-volume
      readOnly: true
      mountPath: "/test/ebs"
  volumes:
  - name: test-volume
    awsElasticBolockStore:
      volumeID: <volume-id>
      fsType: ext4
```

2 CephFS

CephFS卷允许将一个已经存在的卷挂载到Pod中，和emptyDir卷不同的是，当移除Pod时，CephFS卷的内容不会被删除，这意味着可以将数据预先放在在CephFS卷中，并且可以在Pod之间切换该数据.CephFS卷可以被多个写设备同时挂载。

和AWS EBS一样，需要先创建CephFS卷后才能使用它。

前提是公司使用CephFS作为后端存储

3 ConfigMap

ConfigMap也可以作为volume使用 前面已经详细了解过了

4 emptyDir

和上述Volume不同的是，如果删除Pod，emptyDir卷中的数据也将被删除，一般emptyDir卷用于Pod中不同Container共享数据。它可以被挂载到相同或不同的路径上。

默认情况下，emptyDir卷支持节点上的任何介质，可能是SSD,磁盘或网络存储，具体取决于自身的环境。可以将emptyDir.medium字段设置为Memory，让Kubernetes使用tmpfs（内存支持的文件系统），虽然tmpfs非常快，但是tmpfs在节点重启时，数据同样被清除，并且设置的大小会被计入到Container的内存限制当中。

使用emptyDIr卷的示例，之间指定emptyDir为{}即可

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - name: test-webserver
    image: nginx
    volumeMounts:
    - name: cache-volume
      readOnly: true
      mountPath: "/cache"
  volumes:
  - name: cache-volume
    emptyDir: {}
```

5 GlusterFS

GlusterFS（以下简称GFS）是一个开源的网络文件系统，常被用于为Kubernetes提供动态存储，和emptyDir不同的是，删除Pod时GFS卷中的数据会被保留。

关于GFS的使用后面会整理

6 hostPath

hostPath卷可将节点上的文件或目录挂载到Pod上，用于Pod自定义日志输出输出或访问Docker内部的容器等。

使用hostPath卷的示例。将主机的/data目录挂载到Pod的/test-pod目录：

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - name: test-webserver
    image: nginx
    volumeMounts:
    - name: test-volume
      readOnly: true
      mountPath: "/test-pd"
  volumes:
  - name: test-volume
    hostPath:
      path: /data
      type: Directory
```

hostPath卷常用的type（类型）如下。

- type为空字符串：默认选项，意味着挂载hostPath卷之前不会执行任何检查。
- DirectoryOrCreate： 如果给定的path不存在任何东西，那么将根据需要创建一个权限为0755的空目录，和Kubelet具有相同的组和权限。
- Directory：目录必须存在于给定的路径下。
- FileOrCreate：如果给定的路径不存储任何内容，则会根据需要创建一个空文件，权限设置为0644，和Kubelet具有相同的组和所有权，
- FIle：文件，必须存在于给定的路径中。
- Socket：UNIX套接字，必须存在于给定的路径中。
- CharDevice：字符设备，必须存在于给定路径中。
- BlockDevice：块设备，必须存在于给定的路径中。

7 NFS

nfs卷也是一种网络文件系统，同时也可以作为动态存储，和GFS类似，删除Pod时，NFS中的数据不会被删除。NFS可以被多个写入同时挂载。

关于NFS的使用，后面会进行整理

8 persistentVolumeClaim

persistentVolumeClaim卷用于将PersistentVolume(持久化卷)挂载到容器中，PersistentVolume分为动态存储和静态存储，静态存储的PersistentVolume需要手动提前创建PV，动态存储无需手动创建PV。

9 Secret

Secret卷和ConfigMap卷类似，前面已经整理过了

10 SubPath

有时可能需要将一个卷挂载到不同的子目录，此时使用volumeMounts.subPath可以实现不同子目录的挂载，

本示例为一个LAMP共享一个卷，使用subPath卷挂载不同的目录：

```
kind: Pod
metadata:
  name: my-lnmp-site
spec:
  containers:
  - name: mysql
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "rootpassword"
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: site-data
      subPath: mysql
  - name: php
    image: php
    volumeMounts:
    - mountPath: /usr/local/nginx/html
      name: site-data
      subPath: php
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPaht: /usr/local/nginx/html
      name: site-data
      subPath: nginx
  volumes:
  - name: site-data
    persistentVolumeClaim:
      claimName: mysql-lnmp-site-date
```

更多volume可参考kubernetes官网

## 2 PersistentVolume

