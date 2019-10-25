# 创建第一个Pod

写yaml文件

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
spec:
  containers:
  - name: nginx
    image: nginx
```

应用yaml文件

```
kubectl apply -f nginx.yaml
```

输出信息

```
pod/nginx created
```

查看创建的Pod

```
kubectl get pods -o wide
```

输出信息

```
NAME                                READY   STATUS              RESTARTS   AGE   IP                NODE    NOMINATED NODE   READINESS GATES
nginx                               1/1     Running             0          12m   192.168.166.130   node1   <none>           <none>
```



## 创建第一个deployment

写yaml文件（deployment的yaml文件较pod较为负载一些）

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment		# deployment的名称
  labels:
    nginx: nginx			    # 标签用于选择节点使用 先确保节点有这个标签
spec:
  replicas: 3					# 副本数
  selector:  # 定义deployment如何找到要管理的pod与template的label标签相对应
    matchLabels:
      nginx: nginx
  template:
    metadata:
      labels:
        nginx: nginx  # nginx使用label（标签）标记pod
    spec:	# 表示pod运行一个名字为nginx-deployment的容器
      containers:
        - name: nginx-deployment
          image: nginx   # 使用的镜像
```

应用yaml文件

```
kubectl apply -f deploy.yaml
```

输出信息

```
deployment.apps/nginx-deployment created
```

查看创建的deployment

```
kubectl get deployments -o wide
```

输出信息

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS         IMAGES   SELECTOR
nginx-deployment   3/3     3            3           10m   nginx-deployment   nginx    nginx=nginx
```

查看创建的pod

```
kubectl get pods --show-labels
```

输出信息

```
nginx-deployment-76b9f6868b-chx2l   1/1     Running   0          13m   nginx=nginx,pod-template-hash=76b9f6868b
nginx-deployment-76b9f6868b-ggwdw   1/1     Running   0          13m   nginx=nginx,pod-template-hash=76b9f6868b
nginx-deployment-76b9f6868b-s66hn   1/1     Running   0          13m   nginx=nginx,pod-template-hash=76b9f6868b
```

