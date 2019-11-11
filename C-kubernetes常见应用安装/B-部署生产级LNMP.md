# 搭建LNMP

环境规范

nginx和php使用nfs挂载网页文件

mysql使用集群外的mysql

镜像版本

nginx： 基于CentOS的自定义nginx镜像

php：基于CentOS的自定义php镜像

## 1 搭建nfs服务

创建目录

```
mkdir /nfs/web/nginx -p
mkdir /nfs/web/php
mkdir /nfs/web/db
```

启动nfs服务

```
echo "/nfs/web/ *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```

```
exportfs -r
systemctl restart nfs-server
systemctl enable nfs
```

编辑网页文件

```
echo "Hello this is LNMP on kubernetes" > /nfs/web/nginx/index.html
```

```
echo "<?php" > /nfs/web/php/index.php
echo "phpinfo();" >> /nfs/web/php/index.php
echo "?>" >> /nfs/web/php/index.php
```

## 2 创建php的deployment和service

编辑yaml文件

```
apiVersion: v1
kind: Service
metadata:
  name: php
spec:
  selector:
    php: php
  ports:
  - protocol: TCP
    port: 9000
    targetPort: 9000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-deployment                # deployment的名称
  labels:
    php: php                         # 标签用于选择节点使用 先确保节点有这个标签
spec:
  replicas: 5                                   # 副本数
  selector:  # 定义deployment如何找到要管理的pod与template的label标签相对应
    matchLabels:
      php: php
  template:
    metadata:
      labels:
        php: php  # nginx使用label（标签）标记pod
    spec:       # 表示pod运行一个名字为nginx-deployment的容器
      containers:
        - name: php-deployment
          image: 192.168.10.66:5000/php   # 使用的镜像
          volumeMounts:
          - name: php
            mountPath: /usr/local/nginx/html
      volumes:
      - name: php
        nfs:  #设置NFS服务器
          server: 192.168.10.10 #设置NFS服务器地址
          path: /nfs/web/php #设置NFS服务器路径
          readOnly: true #设置是否只读
```

应用YAML文件

```
kubectl apply -f php.yaml
```

输出信息

```
service/php created
deployment.apps/php-deployment created
```

查看创建的Pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-595cfb5774-ljbm2   1/1     Running   0          2m35s
nginx-deployment-595cfb5774-mt26n   1/1     Running   0          2m35s
nginx-deployment-595cfb5774-nfntn   1/1     Running   0          2m35s
nginx-deployment-595cfb5774-nggdf   1/1     Running   0          2m35s
nginx-deployment-595cfb5774-qvk9c   1/1     Running   0          2m35s
php-deployment-6f899c8fdb-9chzb     1/1     Running   0          40s
php-deployment-6f899c8fdb-nr8sv     1/1     Running   0          40s
php-deployment-6f899c8fdb-vxnq8     1/1     Running   0          40s
php-deployment-6f899c8fdb-xgpq7     1/1     Running   0          40s
php-deployment-6f899c8fdb-zwqk2     1/1     Running   0          40s
```

查看创建的Service

```
kubectl get service
```

输出信息

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes      ClusterIP   10.250.0.1     <none>        443/TCP    12d
nginx-service   ClusterIP   10.250.0.183   <none>        80/TCP     3m9s
php             ClusterIP   10.250.0.16    <none>        9000/TCP   74s
```

## 3搭建nginx

编辑YAML文件 创建nginx的Service ConfigMap 和 Deployment

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-dp-config
data:
  nginx.conf: |-
    worker_processes  1;
    events {
        worker_connections  1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
            server_tokens off;
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
            location ~ \.php$ {
                root           html;        #站点Php文件应该放在php服务器上
                fastcgi_pass   php:9000;    #Php服务器的IP地址
                fastcgi_index  index.php;
                fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
                include        fastcgi.conf;        #Fastcgi的配置文件
            }
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    nginx: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment                # deployment的名称
  labels:
    nginx: nginx                            # 标签用于选择节点使用 先确保节点有这个标签
spec:
  replicas: 5                                   # 副本数
  selector:  # 定义deployment如何找到要管理的pod与template的label标签相对应
    matchLabels:
      nginx: nginx
  template:
    metadata:
      labels:
        nginx: nginx  # nginx使用label（标签）标记pod
    spec:       # 表示pod运行一个名字为nginx-deployment的容器
      containers:
        - name: nginx-deployment
          image: mytting/chang:nginx   # 使用的镜像
          volumeMounts:
          - name: www
            mountPath: /usr/local/nginx/html
          - name: nginx-config
            mountPath: /usr/local/nginx/conf/nginx.conf
            subPath: nginx.conf
      volumes:
      - name: www
        nfs:  #设置NFS服务器
          server: 192.168.10.10 #设置NFS服务器地址
          path: /nfs/web/nginx #设置NFS服务器路径
          readOnly: true #设置是否只读
      - name: nginx-config
        configMap:
          name: nginx-dp-config
          items:
          - key: nginx.conf
            path: nginx.conf
          defaultMode: 0755
```

应用YAML文件

```
kubectl apply -f nginx.yaml
```

输出信息

```
configmap/nginx-dp-config created
service/nginx-service created
deployment.apps/nginx-deployment created
```

查看创建的Pod

```
kubectl get pods
```

输出信息

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-595cfb5774-ljbm2   1/1     Running   0          50s
nginx-deployment-595cfb5774-mt26n   1/1     Running   0          50s
nginx-deployment-595cfb5774-nfntn   1/1     Running   0          50s
nginx-deployment-595cfb5774-nggdf   1/1     Running   0          50s
nginx-deployment-595cfb5774-qvk9c   1/1     Running   0          50s
```

## 访问nginx测试

```
curl 10.250.0.183
```

输出信息

```
Hello this is LNMP on kubernetes
```

使用火狐浏览器访问nginx的ClusterIP下的index.php文件

```
firefox 10.250.0.183/index.php
```

输出信息

![](image/LNMP/test-1.png)



## 4 搭建配置mysql

## 1 创建数据库密码secret

```
kubectl create secret generic mysql-pass --from-literal=password=123.com
```

输出信息

```
secret/mysql-pass created
```

### 2 创建PV

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /nfs/web/db
    server: 192.168.10.10
```

应用yaml文件

```
kubectl create -f pv-db.yaml
```

输出信息

```
persistentvolume/mysql-pv created
```

## 3 创建PVC

yaml文件

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-claim
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

应用yaml文件

```
kubectl apply -f mysql-pvc.yaml
```

输出信息

```
persistentvolumeclaim/mysql-claim created
```

## 4 创建Deployment

yaml文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    mysql: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      mysql: mysql
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        mysql: mysql
        tier: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql57-lnmp
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: dz-mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-claim
```

应用YAML文件

```
kubectl create -f dp-mysql.yaml
```

输出信息

```
deployment.apps/mysql created
```

## 5 创建Service

yaml文件

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    mysql: mysql
spec:
  ports:
    - port: 3306
  selector:
    mysql: mysql
    tier: mysql
```

应用YAML文件

```
kubectl apply -f db-service.yaml
```

输出信息

```
service/mysql created
```

## 6 搭建WordPress

下载源码

```
wget https://raw.githubusercontent.com/hejianlai/Docker-Kubernetes/master/Kubernetes/Project/k8s_wordporss/wordpress-4.7.4-zh_CN.tar.gz
```

解压到nginx和PHP的nfs目录下各一份

访问安装

最后结果

![](image/LNMP/test2.png)