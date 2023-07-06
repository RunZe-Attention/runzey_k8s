# <font color = 'red'>前情回顾</font>

## kubeadm优缺点:

- 基于容器镜像构建的:kubeadm,将K8S使用容器化方案运行

- 缺点:
  - 需要科学上网
  - 集群证书只有一年
    - 集群更新
    - 修改kubeadm证书办法源码
  - 掩盖了部分的通信逻辑

- 优点:
  - 安装简单
  - 自愈性

# <font color='red'>什么是对象、什么是资源</font>

- K8S中所有的内容都叫做资源，资源被实例化之后,就是一个具体的对象

  ```shell
  # 我们建立的节点K8S-master01就是一个资源,隶属于node对象
  ```

# <font color='red'>资源划分</font>

## (1).名称空间级别 

- 工作负载型资源： Pod、ReplicaSet、Deployment ...
- 服务发现及负载均衡型资源: Service、Ingress... 
- 配置与存储型资源：Volume、CSI ...
-  特殊类型的存储卷：ConfigMap、Secret ...

```shell
# 查看所有的名称空间中的资源 : kubectl get pod --all-namespaces(-A)
```

## (2).集群级资源

- Namespace、Node、ClusterRole、ClusterRoleBinding

```SHELL
# 查看所有的名称空间 : kubectl get pod --all-namespaces(ns)
# 名称空间不受资源控制会使用当前集群的最大资源
```

## (3).元数据级别资源

-  HPA、PodTemplate、LimitRange

# <font color = 'RED'>资源清单</font>

- 在K8S里面，我们通常使用yaml文件来描述创建`资源对象`的清单,比如创建POD、创建一个controller
- <font color = 'blue'>kubectl create deployment mynginx --image=imagename --dry-run -o yaml > /tmp/myngnix.yaml</font>
  - --dry-run:只测试不运行
  - -o yaml:将实例化的对象转换为yaml格式
  - -o json:也可以保存json
- 将创建的资源提交给K8S集群:kubelctl  create/apply -f  yaml文件
  - yaml 资源文件提交给kubectl转换成json格式在提交给api-server进行执行

# <font color = 'red'>资源清单格式</font>

## 资源清单顶级字段

- <font color='blue'>apiVersion</font>:

```shell
# apiVersion存在的意义:
  目前K8S愈加庞大,包含了众多的功能,不可能是一个团队搞定的,比如团队A开发POD以及各类controller,团队B开发service以及ingress,   为了防止并行开发期间的互不干扰,分为不同的目录,每个目录形成了一个apiVersion的版本,我们想使用不同的功能就去不同的apiVersion中   去寻找我们所需要的apiVersion。
  还有一个情景,即使同一个团队开发也会有正式版本的api以及测试版本的api,新的功能也会新开一个目录,对应的一个新的apiVersion的beta   版本。
  apiVersion是通过kind顶级字段来判断到底使用哪个目录

# group/apiversion,如果没有给定group名称,那么默认为core。
# v1就是一个核心组,不用写组名
   rbac.authorization.k8s.io/v1beta1
   scheduling.k8s.io/v1
   scheduling.k8s.io/v1beta1
   storage.k8s.io/v1
   storage.k8s.io/v1beta1
   v1

# 使用命令 kubectl api-versions获取当前K8S上面所有的apiversion信息。

# 有一个很重要的操作就是当你不知道具体使用哪个apiVersion的时候,你可以使用kubectl explain命令,比如单纯的创建一个POD,可以看到version信息为v1
[root@k8s-master01 yaml_pro]# kubectl explain pod
KIND:     Pod
VERSION:  v1
...
```

- <font color = 'blue'>kind:资源类别(Pod RC RS deployment等)</font>

```shell
# kubectl explain Pod
```

- <font color = 'blue'>metadata</font>

```
Pod的元数据都有哪些:kubectl explain pod.metadata
```

- <font color = 'blue'>spec</font>

```
资源的期望定义
```

- <font color = 'blue'>status(不需要用户定义,K8S自身维护)</font>

```shell
apiVersion: group/apiversion  # 如果没有给定 group 名称，那么默认为 core，可以使用 kubectl api-versions # 获取当前 k8s 版本上所有的 apiVersion 版本信息( 每个版本可能不同 )
kind:       #资源类别
metadata：  #资源元数据
   name     
   namespace
   lables
   annotations   # 主要目的是方便用户阅读查找
spec: # 期望的状态（disired state）
status: # 当前状态，本字段有 Kubernetes 自身维护，用户不能去定义
```

```shell
[root@k8s-master01 yaml_pro]# kubectl explain Pod.metadata.labels
KIND:     Pod
VERSION:  v1

FIELD:    labels <map[string]string>  ({key: value}格式)

```

## <font color = 'red'>资源清单的常用命令</font>

**获取 apiversion 版本信息**

```shell
[root@k8s-master01 ~]# kubectl api-versions 
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
......(以下省略)
v1
```



**获取资源的 apiVersion 版本信息**

```shel
[root@k8s-master01 ~]# kubectl explain pod
KIND:     Pod
VERSION:  v1
.....(以下省略)

[root@k8s-master01 ~]# kubectl explain Ingress
KIND:     Ingress
VERSION:  extensions/v1beta1
```



**获取字段设置帮助文档**

```shell
[root@k8s-master01 ~]# kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion    <string>
     ........
     ........
```

**字段配置格式**

```yaml
apiVersion <string>          #表示字符串类型
metadata <Object>            #表示需要嵌套多层字段
labels <map[string]string>   #表示由k:v组成的映射
finalizers <[]string>        #表示字串列表
ownerReferences <[]Object>   #表示对象列表
hostPID <boolean>            #布尔类型
priority <integer>           #整型
name <string> -required-     #如果类型后面接 -required-，表示为必填字段
```



## <font color = 'red'>通过定义清单文件创建 Pod</font>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo1
  namespace: default
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-1
    image: wangyanglinux/myapp:v1
  -name: busybox-1
    image: busybox:1.34.1
    command: ["/bin/sh","-c","sleep 3600"]
```

```shell
# kubectl get pod xx.xx.xx -o yaml  
```

#### POD底层依旧是container,下面的内容是上方创建pod后所展现的container的内容,会发现此时pod-demo1中有三个container,

- myapp-1
- busybox-1
- pause(只要存在pod就会被创建的container)

```SHELL
[root@k8s-node01 ~]# docker ps | grep pod-demo1
a2ceea17252d   1a80408de790           "/bin/sh -c 'sleep 3…"   5 minutes ago   Up 5 minutes             k8s_busybox-1_pod-demo1_default_2bf6eed5-c8a9-42e8-b644-e1f546420999_0
25d121aab450   d4a5e0eaa84f           "nginx -g 'daemon of…"   5 minutes ago   Up 5 minutes             k8s_myapp-1_pod-demo1_default_2bf6eed5-c8a9-42e8-b644-e1f546420999_0
0f1fbac5be9c   k8s.gcr.io/pause:3.1   "/pause"                 5 minutes ago   Up 5 minutes             k8s_POD_pod-demo1_default_2bf6eed5-c8a9-42e8-b644-e1f546420999_0
```

# <font color='red'>目前一些常用的命令</font>

- kubectl get pod pod名字 --show-labels <font color='blue'> 显示pod的标签</font>
- kubectl get pod -l app  <font color='blue'>(这是一个key值) 过滤有这个key的pod</font>
- kubectl get pod pod名字 -o yaml<font color='blue'> 将pod打印出yaml格式</font>

- kubectl describe pod pod名字  <font color='blue'>查看pod本身信息</font>
- kubectl logs pod名字 <font color='blue'>查看pod内部进程日志</font>
- kubectl exec -it pod名字  -c container名字 /bin/sh  <font color='blue'>进入pod内部的container</font>

# <font color = "red">常用解释列举</font>

![常用解释1](.\images\常用解释1.PNG)

![常用解释1](.\images\常用解释2.PNG)

![常用解释1](.\images\常用解释3.PNG)

![常用解释1](.\images\常用解释4.PNG)

![常用解释1](.\images\常用解释5.PNG)
