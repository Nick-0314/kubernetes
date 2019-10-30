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



