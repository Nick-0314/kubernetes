# CronJob

CronJob用于以时间为基准周期性地执行任务，这些自动化任务和运行在Linux或Unix系统上的CronJob一样。CronJob对于创建周期和重复任务非常有用，例如执行备份任务，周期性调度程序节点，发送电子邮件等。

对于Kubernetes1.8以前的版本，需要添加--runtime-config=batch/v2alpha=true参数至APIServer中，然后重启APIServer和Controller Manager用于启用API，对于1.8以后的版本无须修改任何参数，可以直接使用

## 1 创建CronJob

创建CronJob由两种方式，一种是直接使用kubectl 命令行创建，一种说使用yaml文件创建

使用kubectl创建CronJob的命令如下:

```
kubectl run hello --schedule="*/1 * * * * " --restart=OnFailure --image=busybox -- /bin/sh -c "data; echo Hello from the Kubernetes Cluster"
```

输出信息

```
kubectl run --generator=cronjob/v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
cronjob.batch/hello created
```

查看CronJob

```
kubectl get cronjobs.batch
```

输出信息

```
NAME    SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *    False     1        10s             11s
```

输出为yaml文件

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
  namespace: default
spec:
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - args:
            - /bin/sh
            - -c
            - data; echo Hello from the Kubernetes Cluster
            image: busybox
            name: hello
          restartPolicy: OnFailure
  schedule: '*/1 * * * * '
```

本例创建一个每分钟执行一次，打印当前时间和Hello from the Kubernetes cluster的计划任务

等待一分钟可以查看执行的任务(Jobs)

```
kubectl get jobs.batch 
```

输出信息

```
NAME               COMPLETIONS   DURATION   AGE
hello-1572877140   1/1           12s        47s
```

CronJob每次调用任务的时候会创建一个Pod执行命令，执行完任务后，Pod状态就会变成 Completed，如下所示

```
kubectl get pods
```

输出信息 最多只保留三个已完成的Pod，当有新的Pod完成后，会把最旧的Pod删除

```
NAME                                READY   STATUS      RESTARTS   AGE
hello-1572877140-89kc4              0/1     Completed   0          2m18s
hello-1572877200-gr2dr              0/1     Completed   0          77s
hello-1572877260-dchmp              0/1     Completed   0          17s
```

可以通过logs 查看Pod的执行日志

```
kubectl logs hello-1572877320-pf8mg
```

输出信息

```
/bin/sh: data: not found
Hello from the Kubernetes Cluster
```

如果要删除CronJob，直接删除即可

```
kubectl delete cronjobs.batch hello
```

输出信息

```
cronjob.batch "hello" deleted
```

## 2 可选参数的配置

运行一个CronJob的yaml文件如下

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
  namespace: default
spec:
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      creationTimestamp: null
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - /bin/sh
            - -c
            - data; echo Hello from the Kubernetes Cluster
            image: busybox
            imagePullPolicy: Always
            name: hello
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          restartPolicy: OnFailure
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
  schedule: '*/1 * * * * '
  successfulJobsHistoryLimit: 3
  suspend: false
```

其中各参数的说明如下(可以按需修改)：

- schedule 调度周期，和Linux一致，分别是分时日月周。
- restartPolicy 重启策略，和Pod一致。
- concurrencyPolicy 并发调度策略。可选参数如下:
  - Allow 允许同时运行多个任务。
  - Forbid 不允许并发运行，如果之前的任务的任务尚未完成，新的任务不会被创建。
  - Replace 如果之前的任务尚未完成，新的任务会替换掉之前的任务
- Suspend 如果设置为true 则暂停后续的任务，默认为false
- successfulJobsHistoryLimit 保留多少失败的任务

相对于Linux上的计划任务，Kubernetes的CronJob 更具有可配置性，并且对于执行计划任务的环境只需启动相对应的镜像即可。比如，如果需要Go或者PHP环境执行任务，就只需要更改任务的镜像为Go或者PHP即可，而对于Linux上的计划任务，则需要安装相对应的执行环境，此外，Kubernetes的CronJob是创建Pod来执行，更加清晰明了，查看日志也比较方便，

