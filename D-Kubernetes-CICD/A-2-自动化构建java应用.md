# 自动化构建Java应用

本节演示自动构建Java应用，使用的是Tomcat。由于步骤是按照公司内部代码编写的，因此请将Dockerfile和Jenkinsfile放置于目标公司代码的根目录，然后进行测试，或者使用其他项目测试，本示例只是为了演示相关步骤和流水线（pipline）的编写



## 1创建deployment和Secret

```
kubectl create secret docker-registry harbor --docker-server=harbor.devops.com --docker-username=admin --docker-email=tt@qq.com --docker-password=Harbor12345
```



```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-java
  labels:
    app: test-java
    env: test
spec:
  selector:
    matchLabels:
      app: test-java
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: test-java
    spec:
      containers:
        - name: test-java
          image: harbor.devops.com/devops/tomcat:8
          imagePullPolicy: IfNotPresent

          # volumeMounts:
          #  - name: openldap-data
          #    mountPath: /var/lib/ldap
          #    subPath: data
          #  - name: openldap-data
          #    mountPath: /etc/ldap/slapd.d
          #    subPath: config
          #  - name: openldap-data
          #    mountPath: /container/service/slapd/assets/certs
          #    subPath: certs
          #  - name: secret-volume
          #    mountPath: /container/environment/01-custom
          #  - name: container-run
          #    mountPath: /container/run
          ports:
            - containerPort: 8080
              name: tomcat
            # - name: ssl-ldap-port
            #  containerPort: 636
          livenessProbe:
            tcpSocket:
              port: tomcat
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 10
          readinessProbe:
            tcpSocket:
              port: tomcat
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 10
      imagePullSecrets:
        - name: harbor
      #volumes:
      #  - name: openldap-data
      #    persistentVolumeClaim:
      #      claimName: openldap-data
      #  - name: "secret-volume"
      #    secret:
      #      secretName: "ldap-secret"
      #  - name: container-run
      #    emptyDir: {}
```

先创建Deployment

##  2定义Dockerfile

```
ROM harbor.devops.com/devops/tomcat:8
RUN rm -rf /usr/local/tomcat/webapps/* && mkdir /usr/local/tomcat/webapps/ROOT -p
COPY ./ROOT /usr/local/tomcat/webapps/ROOT
CMD [ "sh", "-c", "catalina.sh run" ]
```



## 3 定义Jenkinsfile

```
def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(label: label, containers: [
  containerTemplate(name: 'jnlp', image:  'jenkins/jnlp-slave:3.35-5-alpine', args: '${computer.jnlpmac} ${computer.name}'),
  containerTemplate(name: 'docker', image: 'harbor.devops.com/devops/docker:19.03',  command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'maven', image: 'harbor.devops.com/devops/maven:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'harbor.devops.com/devops/kubectl', command: 'cat', ttyEnabled: true)
],
volumes: [
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  hostPathVolume(mountPath: '/root/.kube/config', hostPath: '/root/.kube/config'),
  hostPathVolume(mountPath: '/etc/hosts', hostPath: '/etc/hosts'),
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def shortGitCommit = "${gitCommit[0..10]}"
    def previousGitCommit = sh(script: "git rev-parse ${gitCommit}~", returnStdout: true)
    def JOB_NAME = "test-java"
    def BUILD_NAME = "dxh"
    def APP_IMAGE_URL = "harbor.devops.com/devops/"
    def NS = "default"
    def PROJECT = "trustee-task"
    stage('build') {
      try {
        container('maven') {
          sh """
            mkdir ROOT -p
            mvn clean install
            cd ROOT && jar -xvf ../target/*.war
            """
        }
      }
      catch (exc) {
        println "Failed to build - ${currentBuild.fullDisplayName}"
        throw(exc)
      }
    }
    stage('Create Docker images') {
      container('docker') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
          credentialsId: 'HARBOR_USER',
          usernameVariable: 'HARBOR_USER',
          passwordVariable: 'HARBOR_PASSWORD']]) {
          sh """
            docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWORD} ${APP_IMAGE_URL}
            docker build -t ${APP_IMAGE_URL}${JOB_NAME}:${BUILD_NAME} .
            docker push ${APP_IMAGE_URL}${JOB_NAME}:${BUILD_NAME}
            """
          }
        }
      }
    stage('Run kubectl') {
      container('kubectl') {
        sh """
            kubectl set image deployment/${JOB_NAME} ${JOB_NAME}=${APP_IMAGE_URL}${JOB_NAME}:${BUILD_NAME} -n ${NS} --record
            kubectl get pod -n ${NS} -l app=${JOB_NAME} -o wide
           """
      }
    }
    }
  }
```

将源码中的src目录和pom文件拷贝到本地仓库内，并上传到gitlab上

```
git　clone https://github.com/mytting/kubernetes.git
```

将代码中的所有数据传到本地仓库内，并上传至gitlab

然后立即构建

![image-20191221170814619](image/A-2-自动化构建java应用/image-20191221170814619.png)

![image-20191221171336747](image/A-2-自动化构建java应用/image-20191221171336747.png)





构建完成之前访问Pod

![image-20191221171422573](image/A-2-自动化构建java应用/image-20191221171422573.png)

构建完成后访问

![image-20191221171501600](image/A-2-自动化构建java应用/image-20191221171501600.png)

harbor镜像仓库

![image-20191221171531496](image/A-2-自动化构建java应用/image-20191221171531496.png)

持续集成完毕