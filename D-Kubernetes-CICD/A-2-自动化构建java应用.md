# 自动化构建Java应用

本节演示自动构建Java应用，使用的是Tomcat。由于步骤是按照公司内部代码编写的，因此请将Dockerfile和Jenkinsfile放置于目标公司代码的根目录，然后进行测试，或者使用其他项目测试，本示例只是为了演示相关步骤和流水线（pipline）的编写

## 1 定义Dockerfile

dockerfile主要写的是如何生成公司业务的镜像：



```
  containerTemplate(name: 'jnlp', image:  'jenkins/jnlp-slave:3.35-5-alpine', args: '${computer.jnlpmac} ${computer.name}'),
```

