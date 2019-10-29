# ConfigMap

一般用ConfigMap管理一些程序的配置文件或者Pod变量比如nginx配置,MavenSetting配置文件等

## 1 什么是ConfigMap

ConfigMap是一个将配置文件,命令行参数,环境变量等,端口号和其他配置绑定到Pod的容器和系统组件.ConfigMap允许将配置与Pod和组件分开,这有助于保持工作负载的可移植性,使配置更易于更改和管理.比如在生产环境中,可以将Nginx.Redis等应用的配置文件存储在ConfigMap上,然后将其挂载即可使用

相对于Secret,ConfigMap更倾向于存储和共享非敏感,未加密的配置信息,如果要在集群中使用敏感信息,最好使用Secret.

## 2 创建ConfigMap

可以使用kubectl create configmap命令从目录 文件或字符值创建ConfigMap

