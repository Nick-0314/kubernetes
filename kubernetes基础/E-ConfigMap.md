# ConfigMap

一般用ConfigMap管理一些程序的配置文件或者Pod变量比如nginx配置,MavenSetting配置文件等

## 1 什么是ConfigMap

ConfigMap是一个将配置文件,命令行参数,环境变量等,端口号和其他配置绑定到Pod的容器和系统组件.ConfigMap允许将配置与Pod和组件分开,这有助于保持工作负载的可移植性,使配置更易于更改和管理.比如在生产环境中,可以将Nginx.Redis等应用的配置文件存储在ConfigMap上,然后将其挂载即可使用

相对于Secret,ConfigMap更倾向于存储和共享非敏感,未加密的配置信息,如果要在集群中使用敏感信息,最好使用Secret.

 **configMap的主要作用:**
		就是为了让镜像 和  配置文件解耦，以便实现镜像的可移植性和可复用性，因为一个configMap其实就是一系列配置信息的集合，将来可直接注入到Pod中的容器使用，而注入方式有两种，一种将configMap做为存储卷，一种是将configMap通过env中configMapKeyRef注入到容器中； configMap是KeyValve形式来保存数据的，如: name=zhangsan 或  nginx.conf="http{server{...}}"  对于configMap的Value的长度是没有限制的，所以它可以是一整个配置文件的信息。

configMap: 它是K8s中的标准组件,它通过两种方式实现给Pod传递配置参数:

​		A. 将环境变量直接定义在configMap中，当Pod启动时,通过env来引用configMap中定义的环境变量。

​		B. 将一个完整配置文件封装到configMap中,然后通过共享卷的方式挂载到Pod中,实现给应用传参。

 secret: 它时一种相对安全的configMap，因为它将configMap通过base64做了编码,  让数据不是明文直接存储在configMap中，起到了一定的保护作用，但对Base64进行反编码，对专业人士来说，没有任何难度，因此它只是相对安全。 

## 2 创建ConfigMap

可以使用kubectl create configmap命令从目录 文件或字符值创建ConfigMap

```
kubectl create configmap <map-name> <date-source>
```

说明：

- map-name ConfigMap的名称

- data-source 数据源，数据的目录，文件或字符值

  数据源对应与ConfigMap中的键-值对（key-value pair），其中

  - key: 文件名或密钥
  - value： 文件内容或字符值

1 从目录创建ConfigMap

可以使用kubectl create configmap 命令从同一个目录中的多个文件创建ConfigMap

创建一个配置文件目录并且下载两个文件作为测试配置文件

```
mkdir configmap
```

编辑一个nginx的配置文件

```
vim configmap/nginx.conf
```

添加如下内容

```
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  0;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```



然后写一个mysql的配置文件

```
vim configmap/my.cnf
```

添加

```
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
character-set-server=utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
```

创建ConfigMap,默认在default命名空间下，可以使用-n更改NampSpace（命名空间）：

这里是测试使用 生产环境下不可能会把mysql的配置文件和nginx的配置文件放在一个ConfigMap内

```
kubectl create configmap nginx-config --from-file=configmap/
```

输出信息

```
configmap/nginx-config created
```

查看创建的ConfigMap

```
kubectl get configmaps
```

输出信息

```
NAME           DATA   AGE
nginx-config   2      5s
```

查看当前的ConfigMap

```
kubectl describe configmaps nginx-config
```

输出信息

```
Name:         nginx-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
my.cnf:
----
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
character-set-server=utf8
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql

nginx.conf:
----
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  0;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

Events:  <none>
```

可以看到 内容是configmap目录下的两个配置文件



2 从文件创建



可以使用kuubectl create configmap 命令从单个文件或多个配置文件创建ConfigMap

例如以mysql配置文件创建ConfigMap

```
kubectl create configmap mysql-config --from-file=configmap/my.cnf
```

输出信息

```
configmap/mysql-config created
```

也可以使用--from-file多次传入参数以从多个数据源创建ConfigMap

```
kubectl create configmap ln-config --from-file=configmap/nginx.conf --from-file=configmap/my.cnf
```

输出信息

```
configmap/ln-config created
```

查看当前的ConfigMap

```
kubectl get configmaps ln-config -o yaml
```

输出信息

```
apiVersion: v1
data:
  my.cnf: |
    [client]
    default-character-set=utf8

    [mysql]
    default-character-set=utf8

    [mysqld]
    character-set-server=utf8
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
  nginx.conf: |
    worker_processes  1;
    events {
        worker_connections  1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  0;
        server {
            listen       80;
            server_name  localhost;
            location / {
                root   html;
                index  index.html index.htm;
            }
            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-30T09:13:08Z"
  name: ln-config
  namespace: default
  resourceVersion: "8186"
  selfLink: /api/v1/namespaces/default/configmaps/ln-config
  uid: 7a1180da-8f68-47d3-94a1-f823e6072e8d
```

3 从ENV文件创建ConfigMap

可以使用--from-env-file从ENV文件创建ConfigMap

先创建一个文件

```
cat > configmap/envfile.txt <<EOF
content='Hello,this is chinoukin 's evnfile'
EOF
```

创建ConfigMap

```
kubectl create configmap env-config --from-env-file=configmap/envfile.txt
```

输出信息

```
configmap/env-config created
```

查看当前的ConfigMap

```
kubectl get configmaps env-config -o yaml
```

输出信息

```
apiVersion: v1
data:
  content: '''Hello,this is chinoukin ''s evnfile'''
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-30T09:39:10Z"
  name: env-config
  namespace: default
  resourceVersion: "10474"
  selfLink: /api/v1/namespaces/default/configmaps/env-config
  uid: 180f1414-f783-4126-805c-a6a1e0197077
```

注意： 如果使用--fron-env-file多次传递参数以从多个数据源创建ConfigMap时，仅最后一个ENV生效

4 自定义data文件名创建ConfigMap

可以使用以下命令自定义文件文件名

```
kubectl create configmap <configmap-name> --fron-file=<my-key-name>=<path-to-file>
```

比如将mysql的配置文件定义为mysql-key

```
kubectl create configmap my-config --from-file=mysql-key=configmap/my.cnf
```

输出信息

```
configmap/my-config created
```

查看创建的ConfigMap

```
kubectl get configmaps my-config -o yaml
```

输出信息

```
apiVersion: v1
data:
  mysql-key: |
    [client]
    default-character-set=utf8

    [mysql]
    default-character-set=utf8

    [mysqld]
    character-set-server=utf8
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-30T09:47:55Z"
  name: my-config
  namespace: default
  resourceVersion: "11232"
  selfLink: /api/v1/namespaces/default/configmaps/my-config
  uid: f654db7d-f8a5-4f76-91ae-58a4528d40d3
```

5 从字符值创建ConfigMaps

可以使用kubectl create configmap 与--from-literal 参数来定义命令行的字符值

```
kubectl create configmap literal --from-literal=literal=yes --from-literal=literal.2=yes
```

输出信息

```
configmap/literal created
```

查看创建的ConfigMap

```
kubectl get configmaps literal -o yaml
```

输出信息

```
apiVersion: v1
data:
  literal: "yes"
  literal.2: "yes"
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-30T09:54:47Z"
  name: literal
  namespace: default
  resourceVersion: "11830"
  selfLink: /api/v1/namespaces/default/configmaps/literal
  uid: 6f2a4532-8888-49b5-877a-cfc6421432fe
```

## 3 ConfigMap实践

本节主要讲解ConfigMap的一些常见用法，比如通过单个ConfigMap定义环境变量，通过多个ConfigMap定义环境变量和将ConfigMap作为卷使用等

1 使用单个ConfigMap定义容器环境变量

首先在ConfigMap中将环境变量定义为键-值对（key-value pair）

```
kubectl create configmap special-config --from-literal=special.how=very
```

输出信息

```
configmap/special-config created
```

然后创建一个Pod将ConfigMap中定义的值special.how分配给Pod的环境变量SPECIAL_LEVEL_KEY:

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: env-config
              key: log_level
  restartPolicy: Never
```

创建pod

```
kubectl apply -f pod.yaml
```

输出信息

```
pod/dapi-test-pod created
```

然后等待一会 等运行完毕

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS      RESTARTS   AGE
dapi-test-pod                       0/1     Completed   0          30s
```

查看日志

```
kubectl logs  dapi-test-pod
```

输出信息 注意第十行 的环境变量

```
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.250.0.1:443
HOSTNAME=dapi-test-pod
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.250.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
SPECIAL_LEVEL_KEY=very
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.250.0.1:443
KUBERNETES_SERVICE_HOST=10.250.0.1
PWD=/
```

2 使用多个ConfigMap定义容器环境变量

首先定义两个或多个ConfigMap

基于上面创建的ConfigMap再创建一个ConfigMap

```
kubectl create configmap type-config --from-literal=special.type=charm
```

输出信息

```
configmap/type-config created
```

然后删除上面创建的Pod

```
kubectl delete -f pod.yaml
```

修改yaml文件   在Pod中引用两个ConfigMap 添加16到20行

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: type-config
              key: special.type
  restartPolicy: Never
```

创建pod

```
kubectl apply -f pod.yaml
```

输出信息

```
pod/dapi-test-pod created
```

查看pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS      RESTARTS   AGE
dapi-test-pod                       0/1     Completed   0          27s
```

查看日志

```
kubectl logs dapi-test-pod
```

输出信息 注意第6行和第11行的环境变量

```
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.250.0.1:443
HOSTNAME=dapi-test-pod
SHLVL=1
HOME=/root
SPECIAL_TYPE_KEY=charm
KUBERNETES_PORT_443_TCP_ADDR=10.250.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
SPECIAL_LEVEL_KEY=very
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.250.0.1:443
KUBERNETES_SERVICE_HOST=10.250.0.1
PWD=/
```

3 将ConfigMap中所有的键-值对配置为容器的环境变量

创建含有多个键-值对的ConfigMap：

记得删除上面创建的ConfigMap

```
kubectl create configmap env-config --from-literal=special.type=charm --from-literal=special.how=very
```

查看创建的ConfigMap

```
kubectl get configmaps env-config -o yaml
```

输出信息 可以看到两个环境变量都在里面

```
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-30T11:19:10Z"
  name: env-config
  namespace: default
  resourceVersion: "19306"
  selfLink: /api/v1/namespaces/default/configmaps/env-config
  uid: 2a660330-4e4f-4450-8faf-b7db53f87b10
```

使用envFrom将ConfigMap所有的键-值对作为容器的环境变量，其中ConfigMap中的键作为Pod中环境变量的名称

修改yaml文件 将上面的环境变量配置部分替换为envFrom 

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
        - configMapRef:
           name: env-config
  restartPolicy: Never
```

创建Pod 记得先删除上面创建的Pod

```
kubectl apply -f pod.yaml
```

输出信息

```
pod/dapi-test-pod created
```

查看Pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS      RESTARTS   AGE
dapi-test-pod                       0/1     Completed   0          109s
```

查看日志

```
kubectl logs dapi-test-pod
```

输出信息 注意第10 行和第13行的环境变量

```
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.250.0.1:443
HOSTNAME=dapi-test-pod
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.250.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
special.type=charm
KUBERNETES_PORT_443_TCP=tcp://10.250.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
special.how=very
KUBERNETES_SERVICE_HOST=10.250.0.1
PWD=/
```

4 将ConfigMap添加到卷

大部分情况下，ConfigMap定义的都是配置文件，不是环境变量，因此需要将ConfigMap中的文件（一般是--from-file创建）挂载到Pod中，然后Pod中的容器就可引用，此时可以通过volume进行挂载

例如，将名称为env-config的ConfigMap挂载到容器的/etc/config/目录下（/etc/config目录会被覆盖）

编辑yaml文件:

先在containers字段内指定一个卷的名字和挂载到容器的位置

然后在spec字段内指定卷的名字和使用的ConfigMap

通过输出ls 目录 来查看文件

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: env-config-volume
        mountPath: /etc/config
  volumes:
    - name: env-config-volume
      configMap:
        name: env-config
  restartPolicy: Never
```

应用yaml文件

```
kubectl apply -f pod.yaml 
```

输出信息

```
pod/dapi-test-pod created
```

查看创建的Pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS      RESTARTS   AGE
dapi-test-pod                       0/1     Completed   0          30s
```

查看日志

```
kubectl logs dapi-test-pod
```

输出信息为ConfigMap中的两个data字段的内容

```
special.how
special.type
```

5 将ConfigMap添加到卷并指定文件名

使用path字段可以指定ConfigMap挂载的文件名，比如将special.how挂载到/etc/config/并指定名称为keys：

修改yaml文件：

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/keys" ]
      volumeMounts:
      - name: env-config-volume
        mountPath: /etc/config
  volumes:
    - name: env-config-volume
      configMap:
        name: env-config
        items:
        - key: special.type
          path: keys
  restartPolicy: Never
```

应用yaml文件

```
kubectl apply -f pod.yaml
```

输出信息

```
pod/dapi-test-pod created
```

查看Pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS      RESTARTS   AGE
dapi-test-pod                       0/1     Completed   0          21s
```

查看日志

```
kubectl logs dapi-test-pod
```

输出信息为（输出信息输出后终端不会换行）

```
charm
```

6 指定特定路径和文件权限

和Secret相似 可以参考

## 4 ConfigMap限制

1.  必须先创建ConfigMap才能在Pod中引用它，如果Pod引用的ConfigMap不存在，Pod将无法启动
2. Pod引用的键必须存在与ConfigMap中，否则无法启动
3. 使用envFrom配置容器环境变量时，默认会跳过被视为无效的键，但不会影响Pod启动，无效的变量会记录在事件日志中，可以通过kubectl get events看到
4. ConfigMap和引用它的Pod需要在同一个命名空间。