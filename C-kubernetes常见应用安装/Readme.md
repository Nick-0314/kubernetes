# 介绍

本节主要介绍容器化部署公司一些常见的应用，在实际使用中，本章的应用采用容器化部署并非必须的，也可自行选择传统的部署方式。

本节的Ingress均使用Traefik(可以更改为Nginx-ingress)进行部署，建议首先将之前部署的Ingress删除，然后再部署Traefik：

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=master"
kubectl -n kube-system create secret generic traefik-cert --from-file=tls.key --from-file=tls.crt
```

yaml[文件](./yaml/traefik)

```
kubectl apply -f traefik/
```

