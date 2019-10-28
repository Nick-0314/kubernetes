# StatefulSet

## StatefulSet(有状态集)常用于部署有状态的且需要有序启动的应用程序

## 1 statefulset的基本概念

Statefulset主要用于管理有状态应用程序的工作负载API对象。比如在生产环境中，可以部署ElasticSearch集群，MongoDB集群或者需要持久化的RabbitMQ集群，Redis集群，kafka集群和Zookeeper集群等。

和Deployment类似，一个Statefulset也同样管理者基于相同容器规范的Pod。不同的是，StatefulSet为每个Pod维护了一个粘性标识。这些Pod是根据相同的规范创建的，但是不可交换，每个Pod都有一个持久的标识符，在重新调度时也会保留，一般格式为StatefulSetName-Number。比如定义一个名字是Redis——Sentinel的StatefulSet，指定创建三个Pod，那么创建出来的Pod名字就为Redis-Sentinel-0，Redis-Sentinel-1，Redis-Sentinel-2。而StatefulSet创建的Pod一般使用Headless Service（无头服务）进行通信，和普通的Service的区别在于Headless Service 没有ClusterIP，它使用的是Endpoint进行互相通信，Headless一般的格式为:

```
statefulSetName-(0..N-1).serviceName.namespace.svc.cluster.local
```

说明

- serviceName 为Headless service的名字
- 0..N-1为Pod所在的序号，从0开始到N-1。
- statefulSetName为StatefulSet的名字
- namespace为服务所在的命名空间
- .cluster.local为Cluster Domain（集群域）