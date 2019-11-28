# Job



```
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
  namespace: default
spec:
  completions: 6  # 一共执行6次 
  parallelism: 2  # 每次两个job并行执行
  template:
    metadata:
      name: myjob
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo","tt"]
      restartPolicy: Never
```

