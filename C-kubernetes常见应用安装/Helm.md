# Helm

简介略

Harbor可以存储chart包 但是需要在启动的时候加一个参数

```
./install.sh  --with-chartmuseum
```

然后创建一共项目  

![image-20200228104227384](image/Helm/image-20200228104227384.png)

然后helm安装push插件

```shell
helm plugin install https://github.com/chartmuseum/helm-push
```

然后添加仓库（项目需要设置为公开）然后还要验证密码

```
helm repo add   --username=admin --password=Harobr12345  helm  https://harbor.devops.com/chartrepo/helm
```

然后helm repo list

![image-20200228104447875](image/Helm/image-20200228104447875.png)

上传chart

```
helm push helm-test/php helm --username=admin --password=Harbor12345
```

![image-20200228104603735](image/Helm/image-20200228104603735.png)

![image-20200228104612470](image/Helm/image-20200228104612470.png)

安装chart

更新仓库

```
helm repo update
```

安装chart

```
helm install php  --username=admin --password=Harbor12345 --version 0.1.0 helm/php
```

