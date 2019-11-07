# Replication Controller 和 ReplicaSet

Replication Controller(复制控制器，RC)和ReplicaSet（复制集，RS）是两种部署Pod的方式。因为在生产环境中，主要使用更高级的Deployment等方式进行Pod的管理和部署，所以本节只对Replication Controller 和ReplicaSet 的部署方式进行简单介绍

## 1 Replication Controller

Replication Controller 可确保Pod副本数达到期望值，也就是RC定义的数量。换句话说，Replication Controller可确保一个Pod或一组同类Pod总是可用。

如果存在的Pod大于设定的值，则Replication Controller 将终止额外的Pod。如果太少，Replication Controller 将启动更多的Pod用于保证达到期望值。与手动创建Pod不同的是,用Replication Controller维护的Pod在失败，删除或终止时会自动替换。因此即使应用程序只需要一个Pod，也应该使用Replication Controller。Replication Controller 类似与进程管理程序，但是Replication Controller不是监视单个节点上的各个进程，而是监视多个节点上的多个Pod。

定义一个Replication Controller 的示例如下

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    rcapp: nginx-rc
  template:
    metadata:
      name: nginx-rc
      labels:
        rcapp: nginx-rc
    spec:
      containers:
      - name: nginx-rc
        image: nginx
        ports:
        - containerPort: 80
```

## 2 ReplicaSet

ReplicaSet是下一代复本控制器。ReplicaSet和 Replication Controller之间的唯一区别是现在的选择器支持。*Replication Controller*只支持基于等式的selector（env=dev或environment!=qa），但ReplicaSet还支持新的，基于集合的selector（version in (v1.0, v2.0)或env notin (dev, qa)）。在试用时官方推荐ReplicaSet。

大多数kubectl支持*Replication Controller*的命令也支持ReplicaSets。rolling-update命令有一个例外 。如果您想要滚动更新功能，请考虑使用Deployments。此外， rolling-update命令是必须的，而Deployments是声明式的，因此我们建议通过rollout命令使用Deployments。

虽然ReplicaSets可以独立使用，但是今天它主要被 Deployments作为协调pod创建，删除和更新的机制。当您使用Deployments时，您不必担心管理他们创建的ReplicaSets。Deployments拥有并管理其ReplicaSets。

ReplicaSet 是支持基于集合的标签选择器的下一代Replication Controller，它主要用作Deployment协调创建，删除和更新Pod，和Replication Controller唯一的区别是，ReplicaSet支持标签选择器。

 ReplicaSet可确保指定数量的pod“replicas”在任何设定的时间运行。然而，Deployments是一个更高层次的概念，它管理ReplicaSets，并提供对pod的声明性更新以及许多其他的功能。因此，我们建议您使用Deployments而不是直接使用ReplicaSets，除非您需要自定义更新编排或根本不需要更新。 

定义一个ReplicatSet的示例如下：

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80
```

