# Secret

secret对象类型用来保存敏感信息，例如密码，令牌，和SSH Key，将这些信息放在Secret中比较安全和灵活。用户可以创建Secret并且引用到Pod中，比如使用Secret初始化Redis，Mysql等密码

## 1 创建Secret

创建Secret的方式有很多，比如使用命令行Kubectl或者使用Yaml/Json文件创建等

### 1 使用Kubectl创建Secret

假设有些Pod需要访问数据库，可以将账户密码存储在username.txt和password.txt文件里，然后以文件的形式创建Secret供Pod使用。

创建账户信息文件

```
echo -n "admin" > username.txt
echo -n "123456" > password.txt
```

以文件username.txt和password.txt创建Secret：

```
kubectl create secret generic db-user-pass --from-file=username.txt --from-file=password.txt
```

输出信息

```
secret/db-user-pass created
```

查看Secret：

```
kubectl get secrets
```

输出信息

```
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      34s
```

```
kubectl describe secrets db-user-pass
```

输出信息

```
Name:         db-user-pass
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  6 bytes
username.txt:  5 bytes
```

默认情况下，get和describe命令都不会显示文件的内容，这是为了防止Secret中的内容被意外暴露

### 2 手动创建Secret

手动创建Secret，因为每一项内容都必须是base64编码，所以要先对其进行编码

```
echo -n "admin" | base64
```

输出信息

```
YWRtaW4=
```

```
echo -n "123456" | base64
```

输出信息

```
MTIzNDU2
```

然后编辑一个yaml文件

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Poaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

最后，使用该文件创建一个Secret

```
kubectl create -f Secret.yaml
```

输出信息

```
secret/mysecret created
```

## 2 解码Secret

Secret被创建后，会以加密的方式存储于Kubernetes集群中，可以对其进行解码获取内容。

首先以yaml的形式获取刚才创建的Secret

```
kubectl get secrets mysecret -o yaml
```

输出信息

```
apiVersion: v1
data:
  password: MTIzNDU2
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2019-11-01T12:28:47Z"
  name: mysecret
  namespace: default
  resourceVersion: "5379"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 0a64ec02-09af-4f33-b53a-955a6bf31479
type: Poaque
```

然后通过--decode解码Secret：

```
echo  "MTIzNDU2" | base64 --decode
```

输出信息为加密前的内容

```
123456
```



## 3 使用Secret

Secret可以作为数据卷被挂载，或作为环境变量以供Pod的容器使用

### 1 在Pod中使用Secret

在Pod中的volume里使用Secret

1. 首先创建一个Secret或者使用已有的Secret，多个Pod可以引用一个Secret
2. 在spce.volumes下增加一个volume，命名随意，spec.volumes.secret.secretName必须和Secret对象的名字相同，并且在同一个NameSpace中。
3. 将spec.containers.volumeMounts加到需要用到该Secret的容器中，并且设置spec.containers.volumeMounts.readOnly=true。
4. 使用spec.containers.volumeMounts.mountPath指定Sectet挂载目录

例如将上面创建的名字为mysecret的Secret挂载到Pod中的/etc/foo:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-mysectet
spec:
  containers:
  - name: mypod-mysecret
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

应用Pod 无报错 Pod正常启动即可

```
kubectl apply -f Pod-Secret.yaml
```

输出信息：

```
pod/mypod-mysectet created
```

验证

```
kubectl exec -it mypod-mysectet /bin/bash
```

查看文件

```
ls /etc/foo/
```

查看文件内容

```
cat /etc/foo/password
```

输出信息

```
123456
```

```
cat /etc/foo/username
```

输出信息

```
admin
```

用到的每个Secret都需要在spec.volumes中指明，如果Pod中有多个容器，每个容器都需要自己的volumeMounts配置块，但是每个Secret只需要一个spec.volumes，可以根据自己的应用场景将多个文件打包到一个Secret中，或者使用多个Secret。

### 2 自定义文件名挂载

挂载Screct时，可以使用spec.volumes.screct.items字段修改每个key的目标路径，即控制Serect Key在容器中的映射路径。

如下所示 最下面的path是在/etc/foo/下

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-mysectet
spec:
  containers:
  - name: mypod-mysecret
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

上述挂载方式，将myserct中的username存储到了/etc/foo/my-group/my-username文件中，而不是/etc/foo/username（不指定items是在这），由于items没有指定password，因此password不会被挂载，如果使用了spec.volumes/secret.items,只有在items中指定的key才会被挂载。

挂载的Secret在容器中作为文件，我们可以在Pod中查看挂载的文件内容

```
ls /etc/foo/
```

输出信息

```
my-group
```

```
cat /etc/foo/my-group/my-username
```

输出信息

```
admin
```

### 3 Secret作为环境变量

Secret可以作为环境变量使用，步骤如下：

1. 创建一个Secret或者使用一个已存在的Secret，多个Pod可以引用用一个Secret。
2. 为每个容器添加对应的Secret Key环境变量env.valueFrom. sectetKeyRef.

比如，定义SECRET_USERNAME和SECRET_PASSWORD两个环境变量，其值来自于名字为mysqlsecret的Serect

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-env-mysectet
spec:
  containers:
  - name: mypod-env-mysecret
    image: nginx
    env:
     - name: SECRET_USERNAME
       valueFrom:
         secretKeyRef:
           name: mysecret
           key: username
     - name: SECRET_PASSWORD
       valueFrom:
         secretKeyRef:
           name: mysecret
           key: password
  restartPolicy: Never
```

启动Pod验证

```
echo $SECRET_USERNAME
```

输出信息

```
admin
```

```
echo $SECRET_PASSWORD
```

输出信息

```
123456
```



## 4 Secret文件权限

Secret默认挂载的文件的权限为0644，可以通过defaultMode方式更改权限

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-mysectet
spec:
  containers:
  - name: mypod-mysecret
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 256
```

更改Secret挂载到/etc/foo目录的文件权限为0400,新版本可以直接指定400

## ５ImagePullSecret

在拉取私有镜像库中的镜像时，可能需要认证后才可拉取，此时可以使用imagePullSecret 将包含Docker镜像注册表密码的Secret传递给Kubelet，然后即可拉取私有镜像。

kubernetes支持在Pod中指定Registry Key，用于拉取私有仓库中的镜像。

首先创建一个镜像仓库账户信息的Secret：

```
kubectl create secret docker-registry myregistrykey  --docker-username=<username> --docker-password=<password> --docker-server=<DOCKER_REGISTRY_SERVER> --docker-email=<email>
```

如果不指定docker-server则默认找hub.docker

如果需要创建多个Registry，则可以为每个注册表创建一个Secret，在Pods拉取镜像时，Kubelet会合并imagePullSecrets到.docker/config.json。注意Secret需要和Pod在同一个命名空间中。

创建完imagePullSecrets后，可以使用imagePullSecrets的方式引用该Secret：

注意Secrets的名字和上面创建的secrets的名字要相同

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod-mysectet
  namespace: default
spec:
  containers:
  - name: mypod-mysecret
    image: mytting/chang:php
  imagePullSecrets:
  - name: myregistrykey
```

测试





## 6 使用案例

本节演示的是Secrets的一些常用配置，比如配置SSH密钥，创建隐藏文件等。

1. 定义包含SSH密钥的Pod

   首先，创建一个包含SSH Key的Secrets：

   ```
   kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/root/.ssh/id_rsa --from-file=ssh-publickey=/root/.ssh/id_rsa.pub
   ```

   然后将其挂载使用：

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: mypod-env-mysectet
   spec:
     containers:
     - name: mypod-env-mysecret
       image: nginx
       volumeMounts:
       - name: secret-volume
         readOnly: true
         mountPath: "/etc/secret-volume"
     volumes:
     - name: secret-volume
       secret:
         secretName: ssh-key-secret
   ```

   上述密钥会被挂载到/etc/secret-volume.注意，挂载SSH Key需要考虑安全性的问题

2. 创建隐藏文件

   为了将数据“隐藏起来”(即文件名以句点符合开头的文件)，可以让Key以一个句点符合开始，比如定义一个以句点符号开头的Secret：

   ```
   kubectl create secret generic ssh-key-secrett --from-file=.ssh-privatekey=/root/.ssh/id_rsa --from-file=.ssh-publickey=/root/.ssh/id_rsa.pub
   ```

   挂载使用

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: mypod-env-mysectet
   spec:
     containers:
     - name: mypod-env-mysecret
       image: nginx
       volumeMounts:
       - name: secret-volume
         readOnly: true
         mountPath: "/etc/secret-volume"
     volumes:
     - name: secret-volume
       secret:
         secretName: ssh-key-secrett
   ```

   验证

   ```
   ls -a /etc/secret-volume/
   ```

   输出信息

   ```
   .  ..  ..2019_11_01_16_00_09.719751794	..data	.ssh-privatekey  .ssh-publickey
   ```

   

