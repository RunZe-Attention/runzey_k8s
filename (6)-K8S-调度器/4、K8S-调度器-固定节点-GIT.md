## <font color = 'red'>**指定(固定)调度节点**</font>

##### <font color = 'puple'>Ⅰ、Pod.spec.nodeName 将 Pod 直接调度到指定的 Node 节点上，会跳过 Scheduler 的调度策略，该匹配规则是强制匹配 </font>

- 创建资源

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myweb
spec:
  replicas: 7
  template:
    metadata:
      labels:
        app: myweb
    spec:
      nodeName: k8s-node02  # Pod.spec.nodeName不为空,跳过Scheduler的调度算法,必须调度到这个k8s-node02上面
      containers:
      - name: myweb
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
```

- 会发现常见的7个pod都运行在k8s-node02上面

  ```BASH
  [root@k8s-master01 scheduler]# kubectl get pod -o wide
  NAME                   READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
  myweb-5c976c64-4qgx8   1/1     Running   0          21s   10.244.2.131   k8s-node02   <none>           <none>
  myweb-5c976c64-6br5j   1/1     Running   0          21s   10.244.2.133   k8s-node02   <none>           <none>
  myweb-5c976c64-dzw8t   1/1     Running   0          21s   10.244.2.130   k8s-node02   <none>           <none>
  myweb-5c976c64-f64f6   1/1     Running   0          21s   10.244.2.135   k8s-node02   <none>           <none>
  myweb-5c976c64-mvltw   1/1     Running   0          21s   10.244.2.132   k8s-node02   <none>           <none>
  myweb-5c976c64-rpngc   1/1     Running   0          21s   10.244.2.129   k8s-node02   <none>           <none>
  myweb-5c976c64-v877n   1/1     Running   0          21s   10.244.2.134   k8s-node02   <none>           <none>
  ```

  

<FONT COLOR = 'PUPLE'>**Ⅱ、Pod.spec.nodeSelector：通过 kubernetes 的 label-selector 机制选择节点，由调度器调度策略匹配 label，而后调度 Pod 到目标节点，该匹配规则属于强制约束** </font>

- 创建资源

```YAML
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myweb
spec:
  replicas: 10
  template:
    metadata:
      labels:
        app: myweb
    spec:
      nodeSelector: # 必须调度到标签为  disktype: ssd上,这比直接指定nodeName要更加灵活
        disktype: ssd
      containers:
      - name: myweb
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
```

-  node01与node02都存在标签disktype=ssd

```SHELL
[root@k8s-master01 scheduler]# kubectl  get node --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
k8s-master01   Ready    master   13d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node-role.kubernetes.io/master=
k8s-node01     Ready    <none>   13d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux
k8s-node02     Ready    <none>   13d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux

```

- 那么查看此时由deployment创建的10个POD在两个node上面的分布情况为

```SHELL
[root@k8s-master01 scheduler]# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
myweb-7964877d67-26hmh   1/1     Running   0          2m27s   10.244.1.126   k8s-node01   <none>           <none>
myweb-7964877d67-4m98h   1/1     Running   0          2m27s   10.244.2.136   k8s-node02   <none>           <none>
myweb-7964877d67-4qbgf   1/1     Running   0          2m27s   10.244.2.138   k8s-node02   <none>           <none>
myweb-7964877d67-9kxnn   1/1     Running   0          2m27s   10.244.1.127   k8s-node01   <none>           <none>
myweb-7964877d67-gfhwf   1/1     Running   0          2m27s   10.244.1.125   k8s-node01   <none>           <none>
myweb-7964877d67-przvd   1/1     Running   0          2m27s   10.244.1.124   k8s-node01   <none>           <none>
myweb-7964877d67-px9tc   1/1     Running   0          2m27s   10.244.2.140   k8s-node02   <none>           <none>
myweb-7964877d67-qpwbf   1/1     Running   0          2m27s   10.244.2.137   k8s-node02   <none>           <none>
myweb-7964877d67-s97qt   1/1     Running   0          2m27s   10.244.2.139   k8s-node02   <none>           <none>
myweb-7964877d67-xmjzd   1/1     Running   0          2m27s   10.244.1.123   k8s-node01   <none>           <none>
```

