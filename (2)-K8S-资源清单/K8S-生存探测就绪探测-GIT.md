# <font color = 'red'>POD-生存探测、就绪探测</font>

```SHELL
# POD中的容器重启就是容器重建,重新启动的容器已经不是之前的container了
```

#### 就绪探测:保证Pod在可用的时候提供给外界服务

- **未设置** : 默认此容器就绪
- **设置** : 必须通过检测后,才能标记为就绪

#### 生存探测:确保应用容器的正常存货

- **未设置**:容器只要有前台进程就是存活
- **设置**:通过手段(TCP、HTTP、EXEC)探测容器内部的服务是否异常

-------------------------------------------------------------------------------------------------------------------------------------------------

-  apiVersion: v1kind: Podmetadata:  name: lifecycle-demospec:  containers:  - name: lifecycle-demo-container    image: harbor.hongfu.com/library/myapp:v1    lifecycle:      postStart:        exec:          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]      preStop:        exec:          command: ["/bin/sh", "-c", "echo Hello from the poststop handler > /usr/share/message"]yaml

  - **ExecAction**：在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。

  - **TCPSocketAction**：对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。

  - **HTTPGetAction**：对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态

    码大于等于200 且小于 400，则诊断被认为是成功的。

- **探针结果**

  - **成功**:容器沟通过了诊断
  - **失败**:容器未通过诊断
  - **未知**:诊断失败了,不会采取任何行动

- **探针类型**

  - **livenessProbe**：指示容器是否正在运行。如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其

    重启策略 的影响。如果容器不提供存活探针，则默认状态为 Success

  - **readinessProbe**：指示容器是否准备好服务请求。如果就绪探测失败，端点控制器将从与 Pod 匹配的所有

    Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 Failure。如果容器不提供就

    绪探针，则默认状态为 Success

### <font color = 'red'>就绪探测举例</font>

```SHELL
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
  namespace: default
  labels:
    app:myapp
spec:
  containers:
  - name: readiness-httpget-container
    image: wangyanglinux/myapp:v1
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1  # 初始化的延时时间:POD初始化1s后再次进行检测
      periodSeconds: 3 # 重复间隔,Pod创建1s后如果监测失败了,那么3s后再次进行检测,直到检测成功为止
```

- 1.这个pod中挂载的container肯定没有index1.html这文件

- 2.所以在进行就绪探测的时候,肯定返回失

  ```shell
  # 使用httpGet手段返回404错误码: Readiness probe failed: HTTP probe failed with statuscode: 404
  ```

- 3.Pod虽然运行,但是一直没有通过就绪探针,容器还没有起来

  ```SHELL
   [root@k8s-master01 yaml_pro]# kubectl get pod
       NAME                    READY   STATUS    RESTARTS   AGE
       readiness-httpget-pod   0/1     Running   0          94s
       # 具体进程报错
       /usr/share/nginx/html/index1.html" failed (2: No such file or directory)
  ```

  

- 4.进入到pod中的容器中增加这个文件:

  ```shell
  # kubectl exec -it  readiness-httpget-pod -c readiness-httpget-container -- /bin/sh
  
  # date > /usr/share/nginx/html/index1.html(增加html文件)
  ```

- 5.会发现Pod立刻进入ready(表示已经通过就绪探测,可以被K8S(比如SVC代理等)集群调度进行实际工作了)

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl get pod
      NAME                    READY   STATUS    RESTARTS   AGE
      readiness-httpget-pod   1/1     Running   0          6m50s
  ```

- 6.有一种情景,使用控制器控制了5个POD,由于业务扩容,replica到十个,创建后但是没有准备好,可能就被使用,使用就绪检测可以避免这个问题,没有准备好,不会提供给业务使用

### <font color = 'red'>存活探测</font>

**1.livenessProbe-exec**

```shell
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
  namespace: default
spec:
  containers:
  - name: liveness-exec-container
    image: wangyanglinux/myapp:v1
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","touch /tmp/live ; sleep 60; rm -rf /tmp/live; sleep 3600"]
    livenessProbe: # 生存探针
      exec: # exec类型
        command: ["test","-e","/tmp/live"]
      initialDelaySeconds: 1  # pod创建后延时检测时间
      periodSeconds: 3        # 从第一次存活检测开始每个3s进行一次检测

```

**2.livenessProbe-httpGet**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-httpget-pod
  namespace: default
spec:
  containers:
  - name: liveness-httpget-container
    image: wangyanglinux/myapp:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    livenessProbe:
      httpGet:
        port: 80
        path: /index.html
      initialDelaySeconds: 1
      periodSeconds: 3
      timeoutSeconds: 3

```

- 1.创建资源是没有问题的

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                   READY   STATUS    RESTARTS   AGE
  liveness-httpget-pod   1/1     Running   0          3s
  
  ```

- 2.kubectl exec -it  liveness-httpget-pod -c  liveness-httpget-container -- /bin/sh

  ```shell
  # pod被restart pod的restart概念就是被重建
  # 重建后修改的index1.html文件也被恢复成index.html文件,重建后的pod的http探针能够顺利通过
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                   READY   STATUS    RESTARTS   AGE
  liveness-httpget-pod   1/1     Running   1          5m16s
  
  ```

**3.tcpsocket**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-tcp
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    livenessProbe:
      initialDelaySeconds: 5
      timeoutSeconds: 1
      tcpSocket:
        port: 80
```



