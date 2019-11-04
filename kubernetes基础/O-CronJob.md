# CronJob

CronJob用于以时间为基准周期性地执行任务，这些自动化任务和运行在Linux或Unix系统上的CronJob一样。CronJob对于创建周期和重复任务非常有用，例如执行备份任务，周期性调度程序节点，发送电子邮件等。

对于Kubernetes1.8以前的版本，需要添加--runtime-config=batch/v2alpha=true参数至APIServer中，然后重启APIServer和Controller Manager用于启用API，对于

