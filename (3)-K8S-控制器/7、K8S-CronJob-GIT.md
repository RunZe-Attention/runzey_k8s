## <font color = 'red'>CronJob Spec</font>

- **spec.template 格式同 Pod**
- **RestartPolicy仅支持Never或OnFailure**
- **单个Pod时，默认Pod成功运行后Job即结束**
- **`.spec.completions`标志Job结束需要成功运行的Pod个数，默认为1**
- **`.spec.parallelism`标志并行运行的Pod的个数，默认为1**
- **`spec.activeDeadlineSeconds`标志失败Pod的重试最大时间，超过这个时间不会继续重试**



## <font color = 'red'>CronJob</font>

***Cron Job* 管理基于时间的 Job，即：** 

- **在给定时间点只运行一次**

- **周期性地在给定时间点运行**

- **使用条件：当前使用的 Kubernetes 集群，版本 >= 1.8（对 CronJob）** 

  

## <font color = 'red'>典型的用法如下所示：</font>

- **在给定的时间点调度 Job 运行**
- **创建周期性运行的 Job，例如：数据库备份、发送邮件**



## <font color = 'red'>CronJob Spec</font>

- **`.spec.schedule`：调度，必需字段，指定任务运行周期，格式同 Cron**

- **`.spec.jobTemplate`：Job 模板，必需字段，指定需要运行的任务，格式同 Job**

- **`.spec.startingDeadlineSeconds` ：启动 Job 的期限（秒级别），该字段是可选的。如果因为任何原因而错过了被调度的时间，那么错过执行时间的 Job 将被认为是失败的。如果没有指定，则没有期限**

- **`.spec.concurrencyPolicy`：并发策略，该字段也是可选的。它指定了如何处理被 Cron Job 创建的 Job 的并发执行。只允许指定下面策略中的一种：**

  - **`Allow`（默认）：允许并发运行 Job**
  - **`Forbid`：禁止并发运行，如果前一个还没有完成，则直接跳过下一个**
  - **`Replace`：取消当前正在运行的 Job，用一个新的来替换**

  **注意，当前策略只能应用于同一个 Cron Job 创建的 Job。如果存在多个 Cron Job，它们创建的 Job 之间总是允许并发运行。**

- **`.spec.suspend` ：挂起，该字段也是可选的。如果设置为 `true`，后续所有执行都会被挂起。它对已经开始执行的 Job 不起作用。默认值为 `false`。**

- **`.spec.successfulJobsHistoryLimit` 和 `.spec.failedJobsHistoryLimit` ：历史限制，是可选的字段。它们指定了可以保留多少完成和失败的 Job。默认情况下，它们分别设置为 `3` 和 `1`。设置限制的值为 `0`，相关类型的 Job 完成后将不会被保留。**



## <font color = 'red'>CronJob举例</font>

- **cronjob资源配置**

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate: # 需要注意的是:这里增加Cronjob自己的jobtemplate
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.34.1
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

- **间隔一分钟就会创建一个job任务（下面是2分钟+的结果**）

  ```shell
  [root@k8s-master01 job]# kubectl get cronjob,job,pod
  NAME                  SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
  cronjob.batch/hello   */1 * * * *   False     1        11s             2m13s
  
  NAME                         COMPLETIONS   DURATION   AGE
  job.batch/hello-1653361860   1/1           1s         2m7s
  job.batch/hello-1653361920   1/1           1s         67s
  job.batch/hello-1653361980   1/1           1s         7s
  
  NAME                         READY   STATUS      RESTARTS   AGE
  pod/hello-1653361860-4ff4s   0/1     Completed   0          2m7s
  pod/hello-1653361920-zlgrc   0/1     Completed   0          67s
  pod/hello-1653361980-jhxhg   0/1     Completed   0          7s
  
  ```

  ```shell
  [root@k8s-master01 job]# kubectl logs pod/hello-1653361980-jhxhg
  Tue May 24 03:13:05 UTC 2022
  Hello from the Kubernetes cluster
  
  ```

- **注意，删除 cronjob 的时候不会自动删除 job，这些 job 可以用 kubectl delete job 来删除**

  ```shell
  # 注意，删除 cronjob 的时候不会自动删除 job，这些 job 可以用 kubectl delete job 来删除
  $ kubectl delete cronjob hello
  cronjob "hello" deleted
  ```

## <font color = 'red'>CrondJob 本身的一些限制</font>

**创建 Job 操作应该是 *幂等的* **



## <font color = 'red'>周期性任务暂停与恢复:suspend</font>

- 现在这个CronJob正以每分钟创建一次job来不断运行,如果我们暂停的话

- 可以kubectl edit cronjob  hello(cronjob资源文件)

  ```shell
  # suspend: false
  # 将spec.suspend下面的status改为true,就不会继续运行了
  # 当需要猜词运行的时候,将suspend置为false
  ```

  