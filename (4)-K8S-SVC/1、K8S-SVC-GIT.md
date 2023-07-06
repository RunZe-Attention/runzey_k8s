# <font color = 'red'>Service</font>

[TOC]

## <font color = 'red'>Service 的概念</font>

**Kubernetes `Service` 定义了这样一种抽象：一个 `Pod` 的逻辑分组，一种可以访问它们的策略 —— 通常称为微服务。 这一组 `Pod` 能够被 `Service` 访问到，通常是通过 `Label Selector`** 

[<img src="https://s4.ax1x.com/2021/12/30/TW3MZD.png" style="zoom:150%;" />](https://imgtu.com/i/TW3MZD)



## <font color = 'red'>核心迭代</font>

**在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。`kube-proxy` 负责为 `Service` 实现了一种 VIP（虚拟 IP）的形式，而不是 `ExternalName` 的形式。 在 Kubernetes v1.0 版本，代理完全在 userspace。在 Kubernetes v1.1 版本，新增了 iptables 代理，但并不是默认的运行模式。 从 Kubernetes v1.2 起，默认就是 iptables 代理。** **在 Kubernetes v1.8.0-beta.0 中，添加了 ipvs 代理**

**在 Kubernetes 1.14 版本开始默认使用  ipvs 代理**

**在 Kubernetes v1.0 版本，`Service` 是 “4层”（TCP/UDP over IP）概念。 在 Kubernetes v1.1 版本，新增了 `Ingress` API（beta 版），用来表示 “7层”（HTTP）服务**

<!--猜想：为什么不擦爱用更为传统的 DNS 实现负载均衡？-->



### <font color = 'blue'>Ⅰ、userspace 代理模式</font>

[![](https://s4.ax1x.com/2021/12/30/TW8chd.png)]()



### <font color = 'blue'>Ⅱ、iptables 代理模式</font>

[![](https://s4.ax1x.com/2021/12/30/TW8xBT.png)]()

### <font color = 'blue'>Ⅲ、ipvs 代理模式</font>

-  **注意： ipvs 模式假定在运行 kube-proxy 之前在节点上都已经安装了 IPVS 内核模块。当 kube-proxy 以 ipvs 代理模式启动时，kube-proxy 将验证节点上是否安装了 IPVS 模块，如果未安装，则 kube-proxy 将回退到 iptables 代理模式**

[![](https://s4.ax1x.com/2021/12/30/TWGl8A.png)]()



### <font color = 'red'>限制</font>

**Service能够提供负载均衡的能力，但是在使用上有以下限制：**

- **只提供 4 层负载均衡能力，而没有 7 层功能，但有时我们可能需要更多的匹配规则来转发请求，这点上 4 层负载均衡是不支持的**



## <font color = 'red'>Service 的类型</font>

- **ClusterIp：默认类型，自动分配一个仅 Cluster 内部可以访问的虚拟 IP**
- **NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 <NodeIP>: NodePort 来访问该服务**
- **LoadBalancer：在 NodePort 的基础上，借助 cloud provider 创建一个外部负载均衡器，并将请求转发到<NodeIP>: NodePort**
- **ExternalName：把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建，这只有 kubernetes 1.7 或更高版本的 kube-dns 才支持**





# <font color = 'red'>ClusterIP</font>



[![](https://s4.ax1x.com/2021/12/30/TWJtQ1.jpg)]()



**为了实现图上的功能，主要需要以下几个组件的协同工作： **

- **apiserver：用户通过 kubectl 命令向 apiserver 发送创建 service 的命令，apiserver 接收到请求后将数据存储到 etcd 中**
- **kube-proxy：kubernetes 的每个节点中都有一个叫做 kube-porxy 的进程，这个进程负责感知service，pod 的变化，并将变化的信息写入本地的 ipvs 规则中**
- xxxxxxxxxx apiVersion: extensions/v1beta1kind: Ingressmetadata:  name: nginx-test  annotations:    nginx.ingress.kubernetes.io/rewrite-target: http://foo.bar.com:31795/hostname.htmlspec:  rules:  - host: foo10.bar.com    http:      paths:      - path: /        backend:          serviceName: nginx-svc          servicePort: 80yaml



# <font color = 'red'>ClusterIP举例</font>

- **创建 myapp-deploy.yaml 文件** 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      release: stabel
  template:
    metadata:
      labels:
        app: myapp
        release: stabel
        env: test
    spec:
      containers:
      - name: myapp
        image: wangyanglinux/myapp:v2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 80
```

- **创建 Service 信息**

```yaml
apiVersion: v1
kind: Service   # svc类型资源
metadata:       
  name: myapp  
  namespace: default
spec:
  type: ClusterIP
  selector:  # SVC也是通过标签选择器来匹配到需要被myapp svc服务代理的pod,选择其中的标签只要是pod中标签的子集就可以代理
    app: myapp
    release: stabel
  ports:
  - name: http
    port: 80
    targetPort: 80
```

- **查看创建的的svc资源与POD资源**

  ```SHELL
  [root@k8s-master01 svc]# kubectl get pod -o wide
  NAME                            READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
  myapp-deploy-6cc7c66999-2rbjg   1/1     Running   0          2m48s   10.244.1.78    k8s-node01   <none>           <none>
  myapp-deploy-6cc7c66999-lwwf5   1/1     Running   0          2m48s   10.244.1.79    k8s-node01   <none>           <none>
  myapp-deploy-6cc7c66999-whmhj   1/1     Running   0          2m48s   10.244.2.234   k8s-node02   <none>           <none>
  [root@k8s-master01 svc]# kubectl get svc -o wide
  NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE    SELECTOR
  kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   21d    <none>
  myapp        ClusterIP   10.103.64.17   <none>        80/TCP    118s   app=myapp,release=stabel
  ```

- **通过SVC来访问集群内部的POD,并查看负载均衡情况**

  ```shell
  [root@k8s-master01 svc]# curl 10.103.64.17/hostname.html
  myapp-deploy-6cc7c66999-whmhj
  [root@k8s-master01 svc]# curl 10.103.64.17/hostname.html
  myapp-deploy-6cc7c66999-lwwf5
  [root@k8s-master01 svc]# curl 10.103.64.17/hostname.html
  myapp-deploy-6cc7c66999-2rbjg
  
  # 此时发现所有POD的访问都是通过SVC代理,并实现了负载均衡
  ```

- **svc中的IP对应的域名**

  ```
  ;; ANSWER SECTION:
  myapp.default.svc.cluster.local. 30 IN	A	10.103.64.17
  
  ```



# <font color = 'red'>Headless Service</font>

- **有时不需要或不想要负载均衡，以及单独的 Service IP 。遇到这种情况，可以通过指定 Cluster  IP ( spec.clusterIP ) 的值为 “ None ” 来创建 Headless Service 。这类 Service 并不会分配 Cluster IP， kube-proxy 不会处理它们，而且平台也不会为它们进行负载均衡和路由** 
- 属于ClusterIP里
- 为StatefulSet提供域名解析

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: default
spec:
  selector:
    app: myapp
  clusterIP: "None"
  ports: 
  - port: 80
    targetPort: 80
```

```shell
[root@k8s-master01 svc]# kubectl  get svc
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP   21d
myapp            ClusterIP   10.103.64.17   <none>        80/TCP    32m
myapp-headless   ClusterIP   None           <none>        80/TCP    21s
# myapp-headless将在statefulset控制器中使用
```

## <font color = 'red'>NodePort</font>

- **clusterIP升级为nodePort比较平稳,nodePort降级为clusterIP不要轻易尝试**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  type: NodePort
  selector:
    app: myapp
    release: stabel
  ports:
  - name: http
    port: 80   # 内部访问端口
    targetPort: 80 #目标端口
    nodePort: 30006 # 外部访问端口是可以被修改的
```

- 可以看到出现了个myapp并且类型为NodePort的SVC

```SHELL
[root@k8s-master01 svc]# kubectl get svc
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP        23d
myapp            NodePort    10.103.64.17   <none>        80:30632/TCP   36h
myapp-headless   ClusterIP   None           <none>        80/TCP         36h

```

- 查看ipvs规则

```SHELL
[root@k8s-master01 svc]# ipvsadm -Ln | grep 30632
TCP  172.17.0.1:30632 rr
TCP  10.4.7.11:30632 rr
TCP  127.0.0.1:30632 rr
TCP  10.244.0.0:30632 rr
# 所有的物理网卡都被绑定30632端口上

[root@k8s-master01 svc]# ipvsadm -Ln |grep 10.103.64.17
TCP  10.103.64.17:80 rr
# nodePort的IP绑定在80端口上

```

-  **这样我们通过浏览器就能在外部访问POD中了**

## <font color = 'red'>clusterIP与nodePort总结</font>

- **clusterIP**给集群内部访问,比如我们的一个集群内部部署了10个tomcat的pod和一个nginx的pod,通过一个clusterIP类型的svc代理10个tomcat,nginx访问这个clusterIP类型的svc,整个工作都是在集群内部完成的
- **nodePort**提供了一种集群外部访问的服务,比如我们现在的nginx服务放到集群外部(单纯举例),如果外部的nginx访问内部的10个tomcat,那么我们完全可以在集群中建立一个nodePort类型的svc,直接提供给外部访问。
- 可以将nodePort看成是增强版本的clusterIP



## <font color = 'red'>LoadBalancer</font>

**loadBalancer 和 nodePort 其实是同一种方式。区别在于 loadBalancer 比 nodePort 多了一步，就是可以调用 cloud provider 去创建 LB 来向节点导流**

[![](https://s4.ax1x.com/2021/12/30/TWtF3j.png)]()



# <font color = 'red'>ExternalName</font>

**这种类型的 Service 通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容( 例如：hub.hongfu.com )。ExternalName Service 是 Service 的特例，它没有 selector，也没有定义任何的端口和 Endpoint。相反的，对于运行在集群外部的服务，它通过返回该外部服务的别名这种方式来提供服务**

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service-1
  namespace: default
spec:
  type: ExternalName
  externalName: hub.hongfu.com
```

**当查询主机 my-service.defalut.svc.cluster.local ( SVC_NAME.NAMESPACE.svc.cluster.local )时，集群的 DNS 服务将返回一个值 my.database.example.com 的 CNAME 记录。访问这个服务的工作方式和其他的相同，唯一不同的是重定向发生在 DNS 层，而且不会进行代理或转发**



