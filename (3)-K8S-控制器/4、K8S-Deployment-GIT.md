# <font color = 'red'>Deployment</font>

**Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义 (declarative) 方法，用来替代以前的ReplicationController 来方便的管理应用。典型的应用场景包括；** 

- **定义 Deployment 来创建 Pod 和 ReplicaSet**
- <font color = 'red'>**滚动升级和回滚应用**</font>
- **扩容和缩容**
- **暂停和继续 Deployment**

# <font color = 'red'>Deployment与RS关系示意图</font>

![deployment与RS](.\images\deployment与RS.PNG)

# <font color = 'red'>问题:使用Deployment间接的对RS进行升级与回滚,为何不直接使用RS改变image版本呢?</font>

## 命令的定义方式:

- **命令时定义**

  ```shell
  # 将执行方式事先定义好
  kubectl create -f									   
  ```

- **声明式定义**

  ```shell
  # 更灵活,由执行者自行决断,在运行的途间修改目标
  kubectl apply -f 
  ```

- RC与RS都属于**命令式定义**、Deployment属于**声明式定义**

## 举例说明:

```YAML
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
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

- **1.1.**我们使用命令式定义deployment资源

  ```shell
  # kubectl create -f yaml文件
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                                READY   STATUS    RESTARTS   AGE
  nginx-deployment-7c678675fc-j9wd7   1/1     Running   0          9s
  nginx-deployment-7c678675fc-mtgjb   1/1     Running   0          9s
  nginx-deployment-7c678675fc-v7v89   1/1     Running   0          9s
  ```

- **1.2.**如果我们将image的v1版本修改为v2版本,再次命令式修改资源

  ```shell
  # kubectl create -f yaml文件
  # Error from server (AlreadyExists): error when creating "10.yaml": deployments.extensions "nginx-deployment" already exists
  # 会发现直接报错,无法感知yaml文件的修改
  ```

- **2.1.**如果我们使用了声明式进行资源定义

  ```shell
  # kubectl apply -f yaml文件
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                                READY   STATUS    RESTARTS   AGE
  nginx-deployment-7c678675fc-49rck   1/1     Running   0          3s
  nginx-deployment-7c678675fc-9wtp7   1/1     Running   0          3s
  nginx-deployment-7c678675fc-sm2x7   1/1     Running   0          3s
  
  ```

- **2.2.**再次使用声明式定义同意资源文件

  ```YAML
  # kubectl apply -f yaml文件
  # deployment.extensions/nginx-deployment unchanged
  # 不会傻瓜是的报错,而会更加智能的判断文件是否出现修改
  
  ```

- **2.3.**如果修改image版本为v2

  ```
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                                READY   STATUS    RESTARTS   AGE
  nginx-deployment-5c478875d8-gw7p5   1/1     Running   0          9s
  nginx-deployment-5c478875d8-nsjfk   1/1     Running   0          10s
  nginx-deployment-5c478875d8-zs57v   1/1     Running   0          10s
  [root@k8s-master01 yaml_pro]# kubectl get rs
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-5c478875d8   3         3         3       15s
  nginx-deployment-7c678675fc   0         0         0       5m4s
  
  # 会在新的RS上进行pod的重建,这也是滚动更新的基础
  ```

- <font color = 'red'>2.4.日常建议直接使用声明式资源,遇到命令式会自动降级</font>

# <font color = 'red'>部署一个简单的例子</font>

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
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

### 清晰的看见创建一个deployment的时间线: deployment -> rs -> pod

```shell
[root@k8s-master01 yaml_pro]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           3m24s
[root@k8s-master01 yaml_pro]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-7c678675fc   3         3         3       3m28s
[root@k8s-master01 yaml_pro]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c678675fc-2d5nj   1/1     Running   0          3m31s
nginx-deployment-7c678675fc-7m5px   1/1     Running   0          3m31s
nginx-deployment-7c678675fc-wvqgt   1/1     Running   0          3m31

# 现在一个pod的命名规则就是 : deployment名称-rsMd5-podMd5
```

#### 一、deployment扩容

- **运行命令**:kubectl scale deployment deployment名字 --replicas=扩容数量

```yaml
# 可以查看kubectl scale --help命令,比较简单
# kubectl scale deployment nginx-deployment --replicas=5
```

- **查看结果**

```SHELL
[root@k8s-master01 yaml_pro]# kubectl scale deployment nginx-deployment --replicas=5
deployment.extensions/nginx-deployment scaled
[root@k8s-master01 yaml_pro]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c678675fc-2d5nj   1/1     Running   0          17m
nginx-deployment-7c678675fc-7m5px   1/1     Running   0          17m
nginx-deployment-7c678675fc-phbfd   1/1     Running   0          28s
nginx-deployment-7c678675fc-vbr8d   1/1     Running   0          28s
nginx-deployment-7c678675fc-wvqgt   1/1     Running   0          17m
```

### 二、HPA解释



### 三、更新镜像版本

- **kubectl  set image deployment/deployment名字 container名字=新版本imag**e

  ```shell
  # kubectl set image deployment/nginx-deployment nginx:wangyanglinux/myapp:v2
  ```

- 解释一下

  ```shell
  # 是通过创建新的rs,在新的rs上进行pod重建
  [root@k8s-master01 yaml_pro]# kubectl get rs
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-5c478875d8   5         5         5       24m
  nginx-deployment-7c678675fc   0         0         0       50m
  
  pod从nginx-deployment-7c678675fc重建到了nginx-deployment-5c478875d8,此时查看pod也能证明这一点
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                                READY   STATUS    RESTARTS   AGE
  nginx-deployment-5c478875d8-2ggqz   1/1     Running   0          27m
  nginx-deployment-5c478875d8-b7w2p   1/1     Running   0          27m
  nginx-deployment-5c478875d8-f67nx   1/1     Running   0          27m
  nginx-deployment-5c478875d8-qxc66   1/1     Running   0          27m
  nginx-deployment-5c478875d8-ttlw5   1/1     Running   0          27m
  
  使用了5c478875d8这个MD5值
  ```

### 四、版本回滚

- **kubectl  rollout undo deployment/nginx-deployment**

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl get rs
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-5c478875d8   0         0         0       28m
  nginx-deployment-7c678675fc   5         5         5       55m
  
  Pod从nginx-deployment-5c478875d8重建到了nginx-deployment-7c678675f
  ```

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME                                READY   STATUS    RESTARTS   AGE
  nginx-deployment-7c678675fc-hkzmk   1/1     Running   0          57s
  nginx-deployment-7c678675fc-tch6h   1/1     Running   0          55s
  nginx-deployment-7c678675fc-whpjx   1/1     Running   0          54s
  nginx-deployment-7c678675fc-xtxz7   1/1     Running   0          55s
  nginx-deployment-7c678675fc-zlsgg   1/1     Running   0          57s
  # 也使用了7c678675fc这个MD5值
  ```

### 五、回滚进阶

#### 问题:当我们的image从v1->v2->v3->v4这么升级的话,当set image 为v4版本后,回滚后为v3,再次回滚为v4,那如果我们想回到v1或者v2怎么办呢?

- 答案是我们可以使用版本号

#### 查看历史版本号

-  **kubectl rollout  history deployment/nginx-deployment**

```SHELL
[root@k8s-master01 yaml_pro]# kubectl rollout  history deployment/nginx-deployment
deployment.extensions/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

```



- 通过滚动历史查看版本号,两个字段

  - REVISON
  - CHANGE-CAUSE

  但是查看后发现CHANGE-CAUSE没有任何信息,原因在于使用kubectl apply -f 声明式常见资源的时候没有进行record

- 增加版本说明

  ```shell
  # kubectl apply -f 11.yaml --record=true
  ```

- 升级版本

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl set image deployment/nginx-deployment nginx=wangyanglinux/myapp:v2 --record=true
  deployment.extensions/nginx-deployment image updated
  
  ```

- 再次查看rollout history

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl rollout history deployment/nginx-deployment
  deployment.extensions/nginx-deployment 
  REVISION  CHANGE-CAUSE
  1         kubectl apply --filename=11.yaml --record=true
  2         kubectl set image deployment/nginx-deployment nginx=wangyanglinux/myapp:v2 --record=true
  
  ```

- 现在我们指定具体版本进行回滚(这样就比较灵活了)

  - 最重要的式进行rollout undo的时候使用  **--to-revision**=2参数

  ```shell
  # kubectl rollout undo deployment/nginx-deployment  --to-revision=1
  ```

  

# 正式的介绍一下Edit命令

- 修改资源保存的持久化文件
- kubectl edit 资源类型 资源名称

# <font color = 'red'>更新策略声明</font>

给资源打补丁

- kubectl patch deployment nginx-deployment -p {JSON层级}

- kubectl patch deployment  nginx-deployment -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'



# <font color = 'red'>金丝雀部署</font>

- **名称来源:**金丝雀是一种非常宝贵的鸟类,K8S指的是将new version image放在Controller控制的所有的pod中的很小一部分pod中去,作为beta版本,当beta版本测试没有问题,则将所有的image都更新为new version image

  ```SHELL
  # 只要有一小组版本在运行 就是金丝雀
  # kubectl set image deploy  nginx-deployment nginx=wangyanglinux/myapp:v2 && kubectl rollout pause deploy  nginx-deployment
  
  # 继续更新
  # kubectl rollout resume deploy  nginx-deployment
  ```




# <font color = 'red'>查看更新是否成功</font>

- kubectl rollout status deployment/nginx-deploymen

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl rollout status deploy/nginx-deployment
  deployment "nginx-deployment" successfully rolled out
  
  [root@k8s-master01 yaml_pro]# echo $?
  0
  
  ```

  

# <font color = 'red'>K8S对rollout的设计不是很优雅,本土化的更新回滚方式很直接,对不同版本的代码常见不同的yaml文件,直接使用声明式命令定义资源就可以</font>

- kubectl -apply -f 不同版本的yaml文件



# <font color = 'red'>清理 Policy</font>

- **您可以通过设置`.spec.revisonHistoryLimit`项来指定 deployment 最多保留多少 revision 历史记录。默认的会保留所有的 revision；如果将该项设置为0，Deployment 就不允许回退了,所谓的revison记录就是rs的数量。**

- **创建第一个关闭revision的v1版本的deployme**nt 

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    revisionHistoryLimit: 0
    selector:
      matchLabels:
        app: nginx
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

- **再次使用yaml文件创建第一个关闭revision的v2版本的deployme**nt 

  ```YAML
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: nginx-deployment
  spec:
    revisionHistoryLimit: 0
    selector:
      matchLabels:
        app: nginx
    replicas: 3
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: wangyanglinux/myapp:v2
          ports:
          - containerPort: 80
  ```

- **当我们创建了这两个资源之后,会发现只有一个rs，这样就不会支持rollout了**

  ```
  [root@k8s-master01 yaml_pro]# kubectl get rs
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-5c478875d8   3         3         3       8m16s
  
  ```

  
