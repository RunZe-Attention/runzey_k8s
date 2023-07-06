# <font color = 'red'>HOOK</font>

- Pod hook（钩子）是由 Kubernetes 管理的 kubelet 发起的
- 当容器中的进程启动前或者容器中的进程终止之前运行
- 这是包含在容器的生命周期之中。可以同时为 Pod 中的所有容器都配置 hook



# <font color= 'red'>Hook类型包含2种</font>

- **exec**：执行一段命令
- **HTTP**:发送HTTP请求



# <font color = 'red'>举例</font>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: wangyanglinux/myapp:v1
    lifecycle:
      postStart: # 启动hook
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:   # 结束前hook
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the poststop handler > /usr/share/message"]
```

- 1.先看结果

  ```SHELL
  # Hello from the postStart handler
  # Hello from the postStart handler
  # Hello from the postStart handler
  # Hello from the postStart handler
  # Hello from the postStart handler
  # Hello from the postStart handler
  # Hello from the poststop handler
  # Hello from the poststop handler
  # Hello from the poststop handler
  # Hello from the poststop handler
  # Hello from the poststop handler
  ```

- 2.kubectl exec -it lifecycle-demo -c lifecycle-demo-container -- /bin/sh

  ```shell
  while 2>1
  do
  cat /usr/share/message
  done
  ```

- 3.删除这个pod kubectl delete pod

- 4.出现上述结果

