# <font color = 'red'>Taint 和 Toleration</font>

- **节点亲和性，是 *pod* 的一种属性（偏好或硬性要求），它使 *pod* 被吸引到一类特定的节点。**
- **Taint 则相反，它使 *POD* 能够 *排斥* 一类特定的 node**

- **Taint 和 toleration 相互配合，可以用来避免 pod 被分配到不合适的节点上。**
- **每个节点上都可以应用一个或多个 taint ，这表示对于那些不能容忍这些 taint 的 pod，是不会被该节点接受的。**
- **如果将 toleration 应用于 pod 上，则表示这些 pod 可以（但不要求）被调度到具有匹配 taint 的节点上**
- **Toleration容忍Taint** 

##  <font color = 'red'>污点(Taint)</font>

##### <font color = 'puple'>Ⅰ、 污点 ( Taint ) 的组成</font>

- **使用 `kubectl taint` 命令可以给某个 Node 节点设置污点，Node 被设置上污点之后就和 Pod 之间存在了一种相斥的关系，可以让 Node 拒绝 Pod 的调度执行，甚至将 Node 已经存在的 Pod 驱逐出去** 

- **每个污点的组成如下：** 

  ```bash
  # key=value:effect
  ```

- **每个污点有一个 key 和 value 作为污点的标签 value 可以为空，effect 描述污点的作用。当前 taint effect 支持如下三个选项：**
  - **`NoSchedule`：表示 k8s 将不会将 Pod 调度到具有该污点的 Node 上**
  - **`PreferNoSchedule`：表示 k8s 将尽量避免将 Pod 调度到具有该污点的 Node 上**
  - **`NoExecute`：表示 k8s 将不会将 Pod 调度到具有该污点的 Node 上，同时会将 Node 上已经存在的 Pod 驱逐出去**

##### <font color = 'puple'>Ⅱ、污点的设置、查看和去除</font>

- 设置taint

  ```bash
  # kubectl taint nodes node1 key1=value1:NoSchedule
  ```

- 节点描述中，查找 Taints 字段

  ```bash
  # kubectl describe node  node-name  | grep Taints 
  ```

- 去除污点(减号 or untainted)

  ```bash
  # kubectl taint nodes node1 key1:NoSchedule-
       
  ```

- master的默认taint

  ```SHELL
  # 可以看到master默认是有污点的
  [root@k8s-master01 scheduler]# kubectl describe node k8s-master01  | grep Taints
  Taints:             node-role.kubernetes.io/master:NoSchedule
  
  # 干掉污点
  [root@k8s-master01 scheduler]# kubectl taint nodes k8s-master01 node-role.kubernetes.io/master:NoSchedule- 
  node/k8s-master01 untainted
  ```

- 举例

  ```YAML
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    replicas: 3
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: wangyanglinux/myapp:v1
          ports:
          - containerPort: 80
        tolerations:
        - key: "nodetype"
          operator: "Equal"
          value: "master"
          effect: "NoSchedule"
  ```

-  已经有pod被调度在了K8S-master01上面

  ```BASH
  [root@k8s-master01 yaml_pro]# kubectl get pod -o wide
  NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
  nginx-deployment-5c478875d8-7xqrv   1/1     Running   0          14s   10.244.1.116   k8s-node01     <none>           <none>
  nginx-deployment-5c478875d8-969dw   1/1     Running   0          14s   10.244.2.121   k8s-node02     <none>           <none>
  nginx-deployment-5c478875d8-9k65v   1/1     Running   0          14s   10.244.0.2     k8s-master01   <none>           <none>
  ```

## <font color = red>POD不能被调度到有污点的node是默认的,但是一些POD就是需要这些有污点的node上,可以给POD增加容忍度与亲和性,就可以调度到指定的node上面了</font>



# <font color = 'red'>容忍(Tolerations)</font>

- **设置了污点的 Node 将根据 taint 的 effect：NoSchedule、PreferNoSchedule、NoExecute 和 Pod 之间产生互斥的关系，Pod 将在一定程度上不会被调度到 Node 上。**
- **但我们可以在 Pod 上设置容忍 ( Toleration ) ，意思是设置了容忍的 Pod 将可以容忍污点的存在，`可能`调度到存在污点的 Node 上**

### <font color = 'puple'>pod.spec.tolerations</font> 

```YAML
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
- key: "key2"
  operator: "Exists"
  effect: "NoSchedule
```

## <font color = 'puple'>举个例子</font>

- 创建资源

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
```

-  此时我们的master是有Taints的

```SHELL
[root@k8s-master01 scheduler]# kubectl describe node k8s-master01 | grep Taints
Taints:             nodetype=master:NoSchedule

```

- 此时我们apply yaml可以看到值不能被调度到K8S-master01上的

```SHELL
[root@k8s-master01 scheduler]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
nginx-deployment-7c678675fc-6knm7   1/1     Running   0          21s   10.244.1.120   k8s-node01   <none>           <none>
nginx-deployment-7c678675fc-g5l5n   1/1     Running   0          21s   10.244.1.121   k8s-node01   <none>           <none>
nginx-deployment-7c678675fc-szn2p   1/1     Running   0          21s   10.244.2.127   k8s-node02   <none>           <none>

```

- 但是当我们的创建的POD增加了对这个标签对的容忍度的话,就可以容忍nodetype=master:Noshedule了

```yaml
[root@k8s-master01 scheduler]# cat tol.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
      tolerations: # pod增加容忍
      - key: "nodetype" # key值
        operator: "Equal" # 容忍的方式
        value: "master" # value值
        effect: "NoSchedule" # 污点类型
```

- 可以看到POD可以被调度到这个K8S-MASTER01这个node上面了

```SHELL
[root@k8s-master01 scheduler]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-65686dd5b6-64j59   1/1     Running   0          12s   10.244.2.128   k8s-node02     <none>           <none>
nginx-deployment-65686dd5b6-jhz6p   1/1     Running   0          12s   10.244.0.3     k8s-master01   <none>           <none>
nginx-deployment-65686dd5b6-vc2ff   1/1     Running   0          12s   10.244.1.122   k8s-node01     <none>           <none
```



# <font color = 'red'>几个特殊的语法</font>

<font color = 'puple'>**Ⅰ、当不指定 key 值时，表示容忍所有的污点 key：** </font>

```yaml
tolerations:
- operator: "Exists"
```

<font color = 'puple'>**Ⅱ、当不指定 effect 值时，表示容忍所有的污点作用**</font>

```yaml
tolerations:
- key: "key"
  operator: "Exists"
```

##### <font color = 'puple'>Ⅲ、有多个 Master 存在时，防止资源浪费，可以如下设置,也就是说尽量别使用master</font>

```shell
  kubectl taint nodes Node-Name node-role.kubernetes.io/master=:PreferNoSchedule
```

