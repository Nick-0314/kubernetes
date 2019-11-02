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

管理计算机资源需要关注的另一个问题是管理存储，PersistentVolume子系统为用户和管理提供了一个API,用于抽象如何根据使用类型提供存储的详细信息。为此，Kubernetes引入了两个新 的API资源：PersistentVolume和PersistentVolumeClaim

PersistentVolume(简称PV)是由管理页设置的存储，它同样是集群中的一类资源，PV是容量插件，如Volumes（卷），但其生命周期独立使用PV的任何Pod，PV的创建可使用NFS，ISCSI，GFS，CEPH等。

PersistentVolumeClaim（简称PVC）是用户对存储的请求，类似于Pod，Pod消耗节点资源，PVC消耗PV资源，Pod可以请求特定级别的资源（CPU和内存）。PVC可以请求特点的大小和访问模式，例如，可以以一次读/写或只读多次的模式挂载。

虽然PVC允许用户使用抽象存储资源，但是用户可能需要具不同性质的PV来解决不同的问题，比如使用SSD硬盘来提高性能。所以集群管理员需要能够提供各种PV，而不仅是大小和访问模式，并且无需让用户了解这些卷的实现方式，对于这些需求可以使用StorageClasse资源实现。

目前PV的提供方式有两种：静态或动态。

静态PV有管理员提前创建，动态PV无需提前创建，只需指定PVC的StorageClasse即可.

### 1 回收策略

当用户使用完卷时，可以从API中删除PVC对象，从而允许回收资源。回收策略会告诉PV如何处理该卷，目前卷可以保留，回收或删除。

- Retain：保留，该策略允许手动回收资源，当删除PVC时，PV仍然存在，volume被视为已释放，管理员可以手动回收卷
- Recycle：回收，如果volume插件支持，Recycle策略会对卷执行rm -rf 清理该PV，并使其可用于下一个新的PVC，但是本策略已弃用，建议使用动态配置。
- Delete：删除，如果Volume插件支持，删除PVC时会同时删除PV，动态卷默认为Delete。

### 2 创建PV

在使用持久化时，需要先创建PV，然后再创建PVC，PVC会和匹配的PV进行绑定，然后Pod即可使用该存储。

创建一个基于NFS的PV：

先配置NFS服务器：

编辑配置文件

```
echo "/nfs *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

创建目录

```
mkdir /nfs/tmp -p
```

启动nfs服务

```
exportfs -r
systemctl restart nfs-server
```

编辑PV的yaml文件

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
  - hard
  - nfsvers=4.1
  nfs:
    path: /tmp
    server: 192.168.10.10
```

说明:

- capacity: 容量

- accessModes：访问模式，包括以下三种
  - ReadWriteOnce：可以被单节点以读写模式挂载，命令行中可以被缩写为RWO.
  - ReadOnlyMany:可以被多个节点以只读模式挂载，命令行中可以被缩写为ROX.
  - ReadWriteMany:可以被多个节点以读写模式挂载，命令行中可以被缩写为RWX.
- storageClassName:PV的类，一个特定类型的PV只能绑定到特定类别的PVC。
- persistentVolumeReclaimPolicy:回收策略
- mountOptions:非必须，新版本中已弃用
- nfs：NFS服务配置。包括一些两个选项：
  - path：NFS上的目录
  - server： NFS服务器的IP地址

创建的PV有以下几种状态：

- Available(可用)，没有被PVC绑定的空间资源
- Bound（已绑定），已经被PVC绑定
- Released(已释放)，PVC被删除，但是资源还未被重新使用
- Failed(失败)，自动回收失败

上面的PV创建后应如下所示

```
kubectl get persistentvolume
```

输出信息

```
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv     1Gi        RWO            Recycle          Available           slow                    10s
```

可以创建一个基于hostPath的PV：

创建本地目录

```
mkdir /mnt/data
```

编辑yaml文件

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

### 3 创建PVC

创建PVC需要注意的是，各个方面都符合PVC要求才能和PV进行绑定，比如accessModes,StorageClassName,VolumeMode都需要相同才能进行绑定。

创建PVC的规范示例如下

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
    - {key: environment,operator: In, values: [dev]}
```

比如上述基于hostPath的PV可以使用以下PVC进行绑定，storage可以比PV小

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

查看PVC状态

```
kubectl get pvc
```

输出信息

```
NAME            STATUS    VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound     task-pv-volume   1Gi        RWO            manual         6s
```

然后创建一个Pod指定volumes即可使用这个PV：

```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
  - name: task-pv-storage
    persistentVolumeClaim:
      claimName: task-pv-claim
  containers:
  - name: task-pv-container
    image: nginx
    ports:
    - containerPort: 80
      name: "http-server"
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: task-pv-storage
```

claimName需要和上述定义的PVC名称task-pv-claim一致

## 3 StorageClass

StorageClass为管理员提供了一种描述存储"类"的办法，可以满足用户不同的服务质量级别，备份策略和任意策略要求的存储需求，一般动态PV都会通过StorageClass来定义。

每个StorageClass包含字段provisioner,parameters和reclaimPolicy,StorageClass对象的名称很重要，管理员在首次创建StorageClass对象时设置的类的名称和其他参数，在被创建后无法再更新这些对象。

定义一个StorageClass的示例如下：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
mountOptions:
  - debug
volumeBindingMode: Immediate
```

1 Provisioner 存储分配器

StorageClass有一个provisioner字段，用于指定配置PV的卷的类型，必须指定此字段，目前支持的卷插件如下表所示

Storage class 有一个分配器，用来决定使用哪个卷插件分配 PV。该字段必须指定。

| Volume Plugin        | Internal Provisioner | Config Example   |
| -------------------- | -------------------- | ---------------- |
| AWSElasticBlockStore | ✓                    | AWS              |
| AzureFile            | ✓                    | Azure File       |
| AzureDisk            | ✓                    | Azure Disk       |
| CephFS               | -                    | -                |
| Cinder               | ✓                    | OpenStack Cinder |
| FC                   | -                    | -                |
| FlexVolume           | -                    | -                |
| Flocker              | ✓                    | -                |
| GCEPersistentDisk    | ✓                    | GCE              |
| Glusterfs            | ✓                    | Glusterfs        |
| iSCSI                | -                    | -                |
| PhotonPersistentDisk | ✓                    | -                |
| Quobyte              | ✓                    | Quobyte          |
| NFS                  | -                    | -                |
| RBD                  | ✓                    | Ceph RBD         |
| VsphereVolume        | ✓                    | vSphere          |
| PortworxVolume       | ✓                    | Portworx Volume  |
| ScaleIO              | ✓                    | ScaleIO          |
| StorageOS            | ✓                    | StorageOS        |

您不限于指定此处列出的”内置”分配器（其名称前缀为 kubernetes.io 并打包在 Kubernetes 中）。 您还可以运行和指定外部分配器，这些独立的程序遵循由 Kubernetes 定义的 规范。 外部供应商的作者完全可以自由决定他们的代码保存于何处、打包方式、运行方式、使用的插件（包括Flex）等。 代码仓库 kubernetes-incubator/external-storage包含一个用于为外部分配器编写功能实现的类库，以及各种社区维护的外部分配器。

例如，NFS 没有内部分配器，但可以使用外部分配器。一些外部分配器列在代码仓库 kubernetes-incubator/external-storage 中。 也有第三方存储供应商提供自己的外部分配器。

2 ReclaimPolicy

回收策略，可以是Delete，Retain，默认为Delete。

3 MountOptions

通过StorageClass动态创建的PV可以使用MountOptions指定挂载参数。如果指定的卷插件不支持指定的挂载选项，就不会被创建成功，因此再设置时需要进行确认。

4 Parameters

PVC具有描述属于StorageClass卷的参数，根据具体情况，取决于provisioner,可以接受不同类型的参数。比如，type为io和特定参数iopsPerGB是EBS所具有的。如果省略配置参数，将采用默认值。

## 4 定义StorageClass

StorageClass一般用于定义动态存储卷，只需要再Pod指定StorageClass的名字即可自动创建对应的PV，无需再手工创建。

以下为常用的StorageClass定义方式。

1 AWS EBS

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

说明

- type: io1 gp2 sc1 st1 默认为gp2
- iopsPerGB: 仅适用于io1,即每GiB每秒的I/O操作。
- fsType： Kubernetes支持的fsType，默认值为：ext4.

2 GCE PD

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
    type: pd-standard
    replication-type: none
```

说明：

- type：pd-standard或pd-ssd，默认为pd-standard.
- replication-type:none或regional-pd默认值为none

3 GFS

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://127.0.0.1:8081"
  clusterid: "xxxxxxxxxxxxxxxxxxxxxxxxx"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "mysecret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
```

说明：

- resturl: Gluster REST服务/Heketi服务的URL，这是GFS动态存储必须的参数

- restauthenabled：用于对REST服务器进行身份验证，此选项已被启用。如果需要启用身份验证，只需指定restuser,restuserkey,secretName或secretNamespace其中一个即可。

- restuser：访问Gluster REST服务的用户

- secretNamespace,secretName: 与Gluster REST 服务交互时使用的Secret。这些参数是可选的，如果没有身份认证不用配置此参数。该Secret使用type为kubernetes.io/glusterfs的Secret进行创建，例如

  ```
  kubectl create secret generic heketi-secret --type="kubernetes.io/glusterfs" --from-literal=key='opensesame' --namespace=default
  ```

- clusterid:  Heketi创建集群的ID，可以是一个列表，用逗号分隔。

- gidMin，gidMax: StorageClass的GID范围，可选，默认为2000-2147483647

- volumetype: 创建的GFS卷的类型，主要分为以下三种：

  - Replica卷： volumetype:replicate:3,表示每个PV会创建3个副本。
  - Disperse/EC卷：volumetype:disperse:4:2 其中4是数据，2是冗余
  - Distribute卷: volumetype:none

当使用GFS作为动态配置PV时，会自动创建一个格式为gluster-dynamic-<claimname>的Endpoint和Headless Service，删除PVC会自动删除PV，Endpoint和Headless Service。

4 Ceph RBD

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 8.8.8.8:6789
  adminId: kube
  adminSecretName: mysecret
  adminSecretNamespace: default
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretName: mysecret
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

说明：

- monitors: Ceph的monitor，用逗号分隔，此参数是必需的。
- adminId： Ceph客户端的ID，默认为admin
- adminSecretName: adminID的Secret名称，此参数是必需的，该Secret必须是kubernetes.io/rbd类型
- adminSecretNamespace:Secret所在的NameSpace(命名空间)，默认为default
- pool：Ceph RBD池，默认值为rbd
- userId：Ceph客户端ID，默认值与adminID相同。
- userSecretName: 和adminSecretName类似，必须与PVC存在于同一个命名空间

- imageFormat: Ceph RBD镜像格式，默认值为2，旧一些的为1

- imagefeatures： 可选参数，只有设置ImageFormat为2时才能使用，目前仅支持layerimg

##  4 动态存储卷

动态卷的配置允许按需自动创建PV，如果没有动态配置，集群管理员必须手动创建PV

动态卷的配置基于StorageClass API组中的API对象 storage.K8S.io。

1 定义GCE动态预配置

要启用动态配置，集群管理员需要手动为用户创建一个或多个StorageClass对象，比如创建一个名字为slow且使用gcc提供存储卷的StorageClass：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

再例如创建一个能提供SSD磁盘的Storage Class:(此yaml文件有问题)

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```



用户通过定义包含StorageClass的PVC来请求动态调配的存储。在Kubernetes1.6之前，是通过volume.beta.kubernetes.io/storage-class注解来完成的，在1.6版本之后，此注解已弃用。

例如,创建一个快速存储类，定义的PersistentVolumeClaim如下：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claiml
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: slow
  resources:
    requests:
      storage: 1Gi
```

storageClassName要与上面创建的StorageClass名字相同

之后会创建一个PV与该PVC进行绑定，然后Pod即可挂载使用。

2 定义GFS动态预配置

后面会整理定义一个GFS的StorageClass：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-heketi
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://xx.xx.xx.xx:xx"
  restauthenabled: "false"
```

之后定义一个PVC：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-gluster-heketi
spec:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: "gluster-heketi"
  resources:
    requests:
      storage: 1Gi
```

PVC一旦被定义，系统便发出Heketi进行相应的操作，在GFS集群上创建brick，再创建并启动一个volume。

然后定义一个Pod使用该存储卷

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-use-pvc
spec:
  containers:
  - name: pod-use-pvc
    image: nginx
    volumeMounts:
    - name: gluster-volume
      mountPath: "/pv-data"
      readOnly: false
  volumes:
  - name: gluster-volume
    persistentVolumeClaim:
      claimName: pvc-gluster-heketi
```

claimName为上述创建PVC的名称