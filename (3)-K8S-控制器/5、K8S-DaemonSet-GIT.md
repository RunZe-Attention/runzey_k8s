# <font color = 'red'>DaemonSet</font>

**DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod**

**使用 DaemonSet  的一些典型用法：**

- **运行集群存储 daemon，例如在每个 Node 上运行 `glusterd`、`ceph`**
- **在每个 Node 上运行日志收集 daemon，例如`fluentd`、`logstash`**
- **在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter、`collectd`、Datadog 代理、New Relic 代理，或 Ganglia `gmond`**



# <font color = 'red'>举例</font>

- <font color = 'puple'>**DaemontSet资源文件**</font>

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: deamonset-example
  labels:
    app: daemonset
spec:
  selector:
    matchLabels:
      name: deamonset-example
  template:
    metadata:
      labels:
        name: deamonset-example
    spec:
      containers:
      - name: daemonset-example
        image: wangyanglinux/myapp:v
```

- <font color = 'puple'>**查看Daemonset分布情况,可以看到除了具有污点的master,所有的worker node都已经运行了一个有Daemonset守护的pod**</font>

```SHELL
[root@k8s-master01 yaml_pro]# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
deamonset-example-h95bx   1/1     Running   0          8s    10.244.1.241   k8s-node01   <none>           <none>
deamonset-example-p87jk   1/1     Running   0          8s    10.244.2.233   k8s-node02   <none>           <none>

```

- <font color = 'puple'>**查看master的Taint（不允许被调度）**</font>

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl describe node k8s-master01 | grep Taint
  Taints:             nodetype=master:NoSchedule
  
  ```

  
