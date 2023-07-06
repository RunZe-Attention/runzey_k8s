# <font color='red'>node亲和性</font>

##### pod.spec.nodeAffinity

- **preferredDuringSchedulingIgnoredDuringExecution：软策略**
- **requiredDuringSchedulingIgnoredDuringExecution：硬策略**

**键值运算关系**

- **In：label 的值在某个列表中**
- **NotIn：label 的值不在某个列表中**
- **Gt：label 的值大于某个值**
- **Lt：label 的值小于某个值**
- **Exists：某个 label 存在**
- DoesNotExist：某个 label 不存在

## <font color = 'puple'>requiredDuringSchedulingIgnoredDuringExecution</font>

- required的风格是:**必须满足条件才能被调度**,否则就是一个**pending**状态

****

-  **查看node的label:** kubectl get node --show-labels

```shell
[root@k8s-master01 ~]#  kubectl get node --show-labels
NAME           STATUS   ROLES    AGE   VERSION   LABELS
k8s-master01   Ready    master   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node-role.kubernetes.io/master=

k8s-node01     Ready    <none>   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux

k8s-node02     Ready    <none>   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux

[root@k8s-master01 ~]# kubectl get node  k8s-node01  --show-labels
NAME         STATUS   ROLES    AGE   VERSION   LABELS
k8s-node01   Ready    <none>   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux
```

- 可以看到node也是有label的,比如在K8S-node01中,default label有

  ```yaml
  # beta.kubernetes.io/arch=amd64      架构
  # beta.kubernetes.io/os=linux	       os
  # kubernetes.io/arch=amd64           架构
  # kubernetes.io/hostname=k8s-node01  节点名称
  # kubernetes.io/os=linux			   os
  ```

- 给K8S-node01增加一个标签:disktype=ssd**(kubectl label node node名字 标签键值对)**

  ```BASH
  # kubectl label node k8s-node01 disktype=ssd  # 
  [root@k8s-master01 ~]# kubectl get node  k8s-node01  --show-labels
  NAME         STATUS   ROLES    AGE   VERSION   LABELS
  k8s-node01   Ready    <none>   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux
  ```

- 给K8S-node02增加一个标签:disktype=ssd**(kubectl label node node名字 标签键值对)**

  ```BASH
  # kubectl label node k8s-node02 disktype=ssd
  [root@k8s-master01 ~]# kubectl get node  k8s-node01  --show-labels
  NAME         STATUS   ROLES    AGE   VERSION   LABELS
  k8s-node02   Ready    <none>   10d   v1.15.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux
  ```

- yaml资源

  ```YAML
  apiVersion: v1
  kind: Pod
  metadata:
    name: affinity
    labels:
      app: node-affinity-pod
  spec:
    containers:
    - name: with-node-affinity
      image: wangyanglinux/myapp:v1
    affinity:  #声明亲和性
      nodeAffinity: # 选择node亲和性
        requiredDuringSchedulingIgnoredDuringExecution: # 使用硬策略 
          nodeSelectorTerms:    
          - matchExpressions:
            - key: disktype # 这个标签
              operator: NotIn # 不存在
              values:
              - hdd # hdd这个value就可以调度到这个node中
  ```

- 观察结果,所有的node都不存在disktype=hdd这个label,所以两者会走有选择略尽心调度

  ```BASH
  [root@k8s-master01 yaml_pro]# kubectl get pod -o wide
  NAME       READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
  affinity   1/1     Running   0          9s    10.244.1.97   k8s-node01   <none>           <none>
  
  ```

## <font color = 'puple'>preferredDuringSchedulingIgnoredDuringExecution</font>

- 从下面的例子可以看出,**软策略**如果不满足亲和性定义的条件,会启动默认的调度算法,也会让pod顺利的运行起来,所以在实际的业务中,选择`硬亲和性`还是`软亲和性`应该根据实际业务进行选择

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-prefer
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: wangyanglinux/myapp:v1
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - xdd 
```

- 即使没有xdd类型的磁盘,依旧会使用默认调度算法调度到一个更优的node中

  ```BASH
  [root@k8s-master01 scheduler]# kubectl get pod affinity-prefer -o wide
  NAME              READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
  affinity-prefer   1/1     Running   0          5m22s   10.244.2.111   k8s-node02   <none>           <none>
  
  ```

## <font color = 'puple'>硬亲和性可以进行预选.软亲和性可以进行优选</font>

- 两种亲和性放到一起

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-require-prefe
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: wangyanglinux/myapp:v1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: NotIn
            values:
            - hdd
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - k8s-node02
            

```

- 硬亲和性使得两者node01与node02都可以被选择,软亲和性的声明更倾向于node02

  ```BASH
  [root@k8s-master01 scheduler]# kubectl get pod affinity-require-prefer -o wide
  NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
  affinity-require-prefer   1/1     Running   0          22s   10.244.2.112   k8s-node02   <none>           <none>
  ```

  

# <font color='red'>Pod亲和性</font>

**pod.spec.affinity.podAffinity/podAntiAffinity**

- **preferredDuringSchedulingIgnoredDuringExecution：软策略**
- **requiredDuringSchedulingIgnoredDuringExecution：硬策略**



## <font color ='puple'>举例</font>

- yaml资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-test
  namespace: default
  labels:
    app: pod-1
spec:
  containers:
  - name: myapp
    image: wangyanglinux/myapp:v1
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-3
  labels:
    app: pod-3
spec:
  containers:
  - name: pod-3
    image: wangyanglinux/myapp:v1
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions: # 必须调度到有app=pod-1这个pod所在的node中
          - key: app
            operator: In
            values:
            - pod-1
        topologyKey: disktype
    podAntiAffinity: # pod的反亲和性
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchExpressions: # 尽量不要调度到有app=pod-2这个pod所在的node中
            - key: app
              operator: In
              values:
              - pod-2
          topologyKey: kubernetes.io/hostname

```

- **着重强调一下topologyKey这个字段**

  - 如果是pod的亲和性

  ```
  那么寻找满足的所有POD,再寻找这些POD所在的所有node中有topologyKey提到的标签的node
  ```

  - 如果是pod的反亲和性

  ```
  寻找所有满足的POD,这些POD的特征是不存在反亲和性时提到的label,再寻找这些POD所在的所有node中有topologyKey提到的标签的node
  ```

  



# <font color="red">亲和性/反亲和性调度策略比较如下：</font>



| **调度策略**        | **匹配标签** | **操作符**                                  | **拓扑域支持** | **调度目标**                   |
| ------------------- | :----------: | ------------------------------------------- | :------------: | ------------------------------ |
| **nodeAffinity**    |   **主机**   | **In, NotIn, Exists, DoesNotExist, Gt, Lt** |     **否**     | **指定主机**                   |
| **podAffinity**     |   **POD**    | **In, NotIn, Exists, DoesNotExist**         |     **是**     | **POD与指定POD同一拓扑域**     |
| **podAnitAffinity** |   **POD**    | **In, NotIn, Exists, DoesNotExist**         |     **是**     | **POD与指定POD不在同一拓扑域** |
