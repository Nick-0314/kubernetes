# Pod

## 1 什么是Pod

Pod可简单的理解为是一组，一个或多个容器，具有共享存储/网络及如何运行容器的规范。Pod包含一个或多个相对紧密耦合的应用程序容器，处于同一个Pod中的容器共享同样的存储空间（Volume，卷或存储卷），IP地址和Port端口，容器之间使用localhost:port相互访问。根据Docker的构造，Pod可被建模为一组具有共享命名空间，卷，IP地址和Port端口的Docker容器

Pod包含的容器最好是一个容器只运行一个进程。每个Pod包含一个pause容器，pause容器是Pod的父容器，它主要负责僵尸进程的回收管理。

Kubernetes为每个Pod都分配一个唯一的IP地址，这样就可以保证应用程序使用同一端口，避免了发生冲突的问题。

一个Pod的状态信息保存在PodStatus对象中，在PodStatus中有一个Phase字段，用于描述Pod在其生命周期中的不同状态。

Pod状态字段Phase的不同取值

| 状态            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| Pending(挂起)   | Pod已被Kubernetes系统接收，但仍有一个或多个容器未被创建。可以通过describe查看处于Pending状态的原因 |
| Running(完成)   | Pod已经被绑定到一个节点上，并且所有的容器都已经被创建。而且至少有一个是运行状态，或者是正在启动或者重启。可以通过logs查看Pod的日志 |
| Completed(成功) | 所有容器执行成功并终止，并且不会再次重启                     |
| Failed(失败)    | 所有的容器都已终止，并且至少有一个容器以失败的方式终止，也就是说这个容器要么以非零状态退出，要么被系统终止 |
| Unknown(未知)   | 通常是由于通信问题造成的无法获得Pod的状态                    |

## 2 Pod探针

Pod探针用来检测容器内的应用是否正常，目前有三种实现方式

Pod探针的实现方式

| 实现方式        | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| ExecAction      | 在容器内执行一个指定的命令，如果命令返回值为0，则认为容器健康 |
| TCPSocketAction | 通过TCP连接检查容器指定的端口，如果端口开发，则认为容器健康  |
| HTTPGetAction   | 对指定的URL进行GET请求，如果状态码在200~400之间，则认为容器健康 |

Pod探针每次检查容器后可能得到的容器状态

Pod探针每次检查容器后可能得到的容器状态

| 状态            | 说明                         |
| --------------- | ---------------------------- |
| Success（成功） | 容器通过检测                 |
| Failure（失败） | 容器检测失败                 |
| Unknown（未知） | 诊断失败，因此不采取任何措施 |

Kubelet有两种探针（即探测器）可以选择性地对容器进行检测

探针的种类

| 种类           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| livenessProbe  | 用于探测容器是否在运行，如果探测失败，kubelet会'杀死'容器并根据重启策略进行相应的处理。如果未指定该探针，将默认未Success |
| readinessProbe | 一般用于探测容器内的程序是否健康，即判断容器是否为就绪（Ready）状态。如果是，则可以处理请求，反之Endpoints Controller将从所有的Services的Endpoints中删除此容器所在Pod的IP地址。如果未指定，将默认为Success |

## 3 Pod镜像拉取策略和重启策略

Pod镜像拉去策略。用于配置当节点部署Pod时，对镜像的操作方式。

镜像拉取策略

| 操作方式     | 说明                                    |
| ------------ | --------------------------------------- |
| Always       | 总是拉取，当tag为latest时，默认为Always |
| Never        | 不管是否存在都不会拉取                  |
| IfNotPresent | 镜像不存在时拉取镜像，默认，排除latest  |

Pod重启策略。在Pod发生故障时对Pod的处理方式

Pod重启策略

| 操作方式  | 说明                                    |
| --------- | --------------------------------------- |
| Always    | 默认策略。容器失效时，自动重启该容器    |
| OnFailure | 容器以不为0的状态码终止，自动重启该容器 |
| Never     | 无论何种状态，都不会重启                |

## 4 创建一个Pod

在生产环境中，很少会单独启动一个Pod直接使用，经常会用Deployment,DaemonSet,StatefulSet等方式调度并管理Pod，定义Pod的参数同时适应于Deployment,DaemonSet,StatefulSet等方式

在kubeadm安装方法下,kubernetes系统组件都是用单独的Pod启动的，当然有时候也会单独启动一个Pod用于测试业务等，此时可以单独创建一个Pod

创建一个Pod的标准格式如下：

```
apiVersion: v1        　　          #必选，版本号，例如v1,版本号必须可以用 kubectl api-versions 查询到 .
kind: Pod       　　　　　　         #必选，Pod
metadata:       　　　　　　         #必选，元数据
  name: string        　　          #必选，Pod名称
  namespace: string     　　        #必选，Pod所属的命名空间,默认为"default"
  labels:       　　　　　　          #自定义标签
    - name: string      　          #自定义标签名字
  annotations:        　　                 #自定义注释列表
    - name: string
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像,否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]     　　        #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      　　             #容器的启动命令参数列表
    workingDir: string                     #容器的工作目录
    volumeMounts:     　　　　        #挂载到容器内部的存储卷配置
    - name: string      　　　        #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string                 #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean                 #是否为只读模式
    ports:        　　　　　　        #需要暴露的端口库号列表
    - name: string      　　　        #端口的名称
      containerPort: int                #容器需要监听的端口号
      hostPort: int     　　             #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string                  #端口协议，支持TCP和UDP，默认TCP
    env:        　　　　　　            #容器运行前需设置的环境变量列表
    - name: string      　　            #环境变量名称
      value: string     　　            #环境变量的值
    resources:        　　                #资源限制和请求的设置
      limits:       　　　　            #资源限制的设置
        cpu: string     　　            #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string                  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:       　　                #资源请求的设置
        cpu: string     　　            #Cpu请求，容器启动的初始可用数量
        memory: string                    #内存请求,容器启动的初始可用数量
    livenessProbe:      　　            #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject   　　    #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:     　　　　#Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false      　　    #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:        　　　　　　    #在该pod上定义共享存储卷列表
    - name: string     　　 　　    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}      　　　　    #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string      　　    #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　　　　    #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      　　　　            #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

