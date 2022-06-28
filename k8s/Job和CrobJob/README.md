# Job
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```
restartPolicy 在 Job 对象里只允许被设置为 Never 和 OnFailure；而在 Deployment 对象里，restartPolicy 则只允许被设置为 Always。
- restartPolicy=Never。离线作业失败后 Job Controller 就会不断地尝试创建一个新 Pod
- restartPolicy=OnFailure。离线作业失败后，Job Controller 就不会去尝试创建新的 Pod。但是，它会不断地尝试重启 Pod 里的容器。

在restartPolicy=Never时，如果作业失败，这时候会不断地有新 Pod 被创建出来。Job 对象的 spec.backoffLimit 字段里定义了重试次数为 4（即，backoffLimit=4），而这个字段的默认值是 6。需要注意的是，Job Controller 重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后。

在 Job 对象中，负责并行控制的参数有两个：
- spec.parallelism，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行；
- spec.completions，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数。

# CronJob
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
在这个 YAML 文件中，最重要的关键词就是 jobTemplate。实际上 CronJob 是一个 Job 对象的控制器（Controller）！
>CronJob 与 Job 的关系，正如同 Deployment 与 ReplicaSet 的关系一样。CronJob 是一个专门用来管理 Job 对象的控制器。只不过，它创建和删除 Job 的依据，是 schedule 字段定义的、一个标准的Unix Cron格式的表达式。

由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。这时候，你可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略。比如：
- concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在；
- concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；
- concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。