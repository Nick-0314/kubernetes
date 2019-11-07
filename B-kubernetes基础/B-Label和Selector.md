# Label和Selector

kubernetes对系统的任何API对象如Pod和节点进行“分组”，会对其添加Label(key=value形式的”键-值对“)用以精确地选择对应的API对象。而Selector(标签选择器)则是针对匹配对象的查询方法。注：键-值对就是key-value pair.

例如，常用的标签tier可用于区分容器的属性，如frontend,backend;或者一个release_track用于区分容器的环境，如canary,production

## 1 定义Label

公司与XX银行有一条专属的高速光纤通道，此通道只能与192.168.10.7.0网段进行通信，因此只能将与XX银行通信的应用部署到192.168.7.0网段所在的节点上，此时可以对节点进行Label（即加标签）：

```
kubectl label nodes node1  region=subnet7
```

输出信息

```
node/node1 labeled
```

然后可以通过Selector对其筛选

```
kubectl get node -l region=subnet7
```

输出节点

```
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    <none>   45h   v1.16.1
```

最后，在Deployment或其他控制器中指定将Pod部署到该节点：

```
.......
containers:
nodeSelector:
  region:subnet7
restartPolicy: Always
 ........  
```

也可以使用同样的方式对Service进行Label

在有Service的情况下可以进行Label

```
kubectl label service <service-name> -n <namespace-name> <key=value> <key=value> ....
```

查看Labels：

```
kubectl label service -n <namespace-name> --show-labels
```

还可以查看所有的Version为v1的service

```
kubectl get service --all-namespaces -l version=v1
```

其他资源的Label方式相同

## 2 Selector 条件匹配

Selector主要用于资源的匹配，只有符合条件的资源才会被调用或使用,可以使用该方式对集群中的各类资源进行匹配.

假如对Selector进行条件匹配，目前已有的Label如下：

```
kubectl get service --show-labels
```

输出信息

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   LABELS
kubernetes   ClusterIP   10.250.0.1   <none>        443/TCP   45h   component=apiserver,provider=kubernetes
```



命令示例：

选择key为value1或者value2的service：

```
kubectl get service -l '<key> in (<value1>,<value2>)' --show-labels
```

选择key1为value1或value2但不包括key2=value3的service

```
kubectl get service -l <key2>!=<value3>, '<key1> in (<value1>,<value2>)' --show-labels
```

选择key=app的service（键等于app）

```
kubectl get service -l app --show-labels
```

## 3 修改标签

在实际使用中，Label的更改时经常发生的事情，可以使用overwrite参数修改标签。

修改标签，比如将version=v1改为version=v2

```
kubectl label service <service-name> -n <namespace-name> version=v2 --overwrite
```

## 4 删除标签

删除标签，比如删除version

```
kubectl label service <sercice-name> -n <namespace-name> version-
```

