## <font color = 'red'>什么是 Helm</font>

**在没使用 helm 之前，向 kubernetes 部署应用，我们要依次部署 deployment、svc 等，步骤较繁琐。况且随着很多项目微服务化，复杂的应用在容器中部署以及管理显得较为复杂，helm 通过打包的方式，支持发布的版本管理和控制，很大程度上简化了 Kubernetes 应用的部署和管理** 



**Helm 本质就是让 K8s 的应用管理（Deployment,Service 等 ) 可配置，能动态生成。通过动态生成 K8s 资源清单文件（deployment.yaml，service.yaml）。然后调用 Kubectl 自动执行 K8s 资源部署** 



**Helm 是官方提供的类似于 YUM 的包管理器，是部署环境的流程封装。Helm 有两个重要的概念：chart 和 release**

- **chart 是创建一个应用的信息集合，包括各种 Kubernetes 对象的配置模板、参数定义、依赖关系、文档说明等。chart 是应用部署的自包含逻辑单元。可以将 chart 想象成 apt、yum 中的软件安装包** 
- **release 是 chart 的运行实例，代表了一个正在运行的应用。当 chart 被安装到 Kubernetes 集群，就生成一个 release。chart 能够多次安装到同一个集群，每次安装都是一个 release** 



**Helm 包含两个组件：Helm 客户端和 Tiller 服务器，如下图所示**

[![](https://s4.ax1x.com/2022/01/05/TjJwdK.png)]()

**Helm 客户端负责 chart 和 release 的创建和管理以及和 Tiller 的交互。Tiller 服务器运行在 Kubernetes 集群中，它会处理 Helm 客户端的请求，与 Kubernetes API Server 交互**

- <font color = 'puple'>chart:集群信息的封装类</font>
- <font color = 'puple'>release:具体的实例化出集群</font>

## <font color = 'red'>Helm 部署</font>

***越来越多的公司和团队开始使用 Helm 这个 Kubernetes 的包管理器，我们也将使用 Helm 安装 Kubernetes 的常用组件。 Helm 由客户端命 helm 令行工具和服务端 tiller 组成，Helm 的安装十分简单。 下载 helm 命令行工具到 master 节点 node1 的 /usr/local/bin 下，这里下载的 2.13. 1版本：***

```shell
$ ntpdate ntp1.aliyun.com
$ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
$ tar -zxvf helm-v2.13.1-linux-amd64.tar.gz
$ cd linux-amd64/
$ cp helm /usr/local/bin/
```

**为了安装服务端 tiller，还需要在这台机器上配置好 kubectl 工具和 kubeconfig 文件，确保 kubectl 工具可以在这台机器上访问 apiserver 且正常使用。 这里的 node1 节点以及配置好了 kubectl**

**因为 Kubernetes APIServer 开启了 RBAC 访问控制，所以需要创建 tiller 使用的 service account: tiller 并分配合适的角色给它。 详细内容可以查看helm文档中的 [Role-based Access Control](https://docs.helm.sh/using_helm/#role-based-access-control)。 这里简单起见直接分配 cluster- admin 这个集群内置的 ClusterRole 给它。创建 rbac-config.yaml 文件：**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

```shell
$ kubectl create -f rbac-config.yaml
	serviceaccount/tiller created
	clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

```shell
$ helm init --service-account tiller --skip-refresh
```



## tiller 默认被部署在 k8s 集群中的 kube-system 这个namespace 下

```shell
$ kubectl get pod -n kube-system -l app=helm
NAME                            READY   STATUS    RESTARTS   AGE
tiller-deploy-c4fd4cd68-dwkhv   1/1     Running   0          83s
```

```shell
$ helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```



# <font color = 'red'>使用HELM(重点)</font>

## <font color = 'puple'>0).Helm 自定义模板</font>

- **创建文件夹**

```shell
$ mkdir ./hello-world
$ cd ./hello-world
```

### <font color = 'puple'>1).每个Chart必须有一个Chart.yaml文件,且name与version字段是必须的</font>

- **创建自描述文件` Chart.yaml` , 这个文件必须有 name 和 version 定义**

  - 在helm模板中创建最重要的文件**Chart.yaml**

    ```bash
    # vim Chart.yaml
    ```

  - 写入必须字段 **name**、**version**

    ```bash
    $ cat <<'EOF' > ./Chart.yaml
    name: hello-world
    version: 1.0.0
    EOF
    ```

    

### <font color = 'PUPLE'>2).所有的业务yaml文件都放到template文件夹中(比如deployment,svc)</font>

- **创建与Chart.yaml统计的业务文件夹template**

```bash
# mkdir ./templates
```

- **创建模板文件， 用于生成 Kubernetes 资源清单（manifests表明）** 

  - 创建deployment资源(helm_deployment.yaml)

    ```YAML
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: hello-world
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: hello-world
        spec:
          containers:
            - name: hello-world
              image: wangyanglinux/myapp:v1
              ports:
                - containerPort: 8080
                  protocol: TCP
    ```

  - 创建service资源(helm_service.yaml)

    ```YAML
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-world
    spec:
      type: NodePort
      ports:
      - port: 8080
        targetPort: 8080
        protocol: TCP
      selector:
        app: hello-world
    ```

#### <font color = 'PUPLE'>3).创建Release</font>

- 使用命令 helm install RELATIVE_PATH_TO_CHART 创建一次Release

- 创建release时必须在Chart.yaml所在的目录

  - **创建release**

  ```bash
  #  helm install --name=myapp . 
  ```

  - **执行结果**

  ```BASH
  [root@k8s-master01 hello]# helm install --name=myapp . 
  NAME:   myapp
  LAST DEPLOYED: Sat May 28 23:13:03 2022
  NAMESPACE: default
  STATUS: DEPLOYED
  
  RESOURCES:
  ==> v1/Pod(related)
  NAME                         READY  STATUS             RESTARTS  AGE
  hello-world-74fc94977-cnn6j  0/1    ContainerCreating  0         0s
  
  ==> v1/Service
  NAME         TYPE      CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
  hello-world  NodePort  10.110.166.119  <none>       8080:30932/TCP  0s
  
  ==> v1beta1/Deployment
  NAME         READY  UP-TO-DATE  AVAILABLE  AGE
  hello-world  0/1    1           0          0s
  
  ```

  - 查看这个release的状态 :helm status myapp

  ```BASH
  [root@k8s-master01 hello]# helm status myapp
  LAST DEPLOYED: Sat May 28 23:13:03 2022
  NAMESPACE: default
  STATUS: DEPLOYED
  
  RESOURCES:
  ==> v1/Pod(related)
  NAME                         READY  STATUS   RESTARTS  AGE
  hello-world-74fc94977-cnn6j  1/1    Running  0         36s
  
  ==> v1/Service
  NAME         TYPE      CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
  hello-world  NodePort  10.110.166.119  <none>       8080:30932/TCP  36s
  
  ==> v1beta1/Deployment
  NAME         READY  UP-TO-DATE  AVAILABLE  AGE
  hello-world  1/1    1           1          36s
  
  ```

#### <font color = 'puple'>4).Chart的一些常用命令</font>

```shell
# 列出已经部署的 Release
$ helm ls

# 查询一个特定的 Release 的状态
$ helm status RELEASE_NAME

# 移除所有与这个 Release 相关的 Kubernetes 资源
$ helm delete cautious-shrimp

# helm rollback RELEASE_NAME REVISION_NUMBER
$ helm rollback cautious-shrimp 1

# 使用 helm delete --purge RELEASE_NAME 移除所有与指定 Release 相关的 Kubernetes 资源和所有这个 Release 的记录
$ helm delete --purge cautious-shrimp
$ helm ls --deleted
```

### <font color = 'puple'>5).能体现出动态配置的操作</font>

- **创建配置文件,给Chart中的Deployment文件读取,配置文件与Chart.yaml在同级目录**

- 注意这个配置文件必须是Chart.yaml同级目录下的values.yaml文件**(必须叫做values.yaml)**

  ```YAML
  # 配置体现在配置文件 values.yaml
  cat <<'EOF' > ./values.yaml
  image:
    repository: wangyanglinux/myapp
    tag: v1
  EOF
  ```

- **这个文件中定义的值，在模板文件中可以通过 .values对象访问到**

- 要非常注意的是 配置文件叫做values.yaml,但是引用的第一个字段必须叫做Values

  ```YAML
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: hello-world
  spec:
    replicas: 1
    template:
      metadata:
        labels:
          app: hello-world
      spec:
        containers:
          - name: hello-world
            image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
            ports:
              - containerPort: 8080
                protocol: TCP
  ```

  ```BASH
  [root@k8s-master01 hello]# helm install --name=mytest .
  NAME:   mytest
  LAST DEPLOYED: Sat May 28 23:46:20 2022
  NAMESPACE: default
  STATUS: DEPLOYED
  
  RESOURCES:
  ==> v1/Pod(related)
  NAME                         READY  STATUS             RESTARTS  AGE
  hello-world-74fc94977-795zj  0/1    ContainerCreating  0         0s
  
  ==> v1/Service
  NAME         TYPE      CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
  hello-world  NodePort  10.105.151.193  <none>       8080:31474/TCP  0s
  
  ==> v1beta1/Deployment
  NAME         READY  UP-TO-DATE  AVAILABLE  AGE
  hello-world  0/1    1           0          0s
  
  ```

  

- **在 values.yaml 中的值可以被部署 release 时用到的参数 --values YAML_FILE_PATH 或 --set key1=value1, key2=value2 覆盖掉**

  ```SHELL
  # helm upgrade --set image.tag=v2 mytest .
  ```

- **直接使用values.yaml做版本升级升级版本**

  - 直接改yaml,最后统一helm upgrade1一下就好

  ```SHELL
  # helm upgrade  -f values.yaml mytest .
  ```

## <font color= 'red '>Debug</font>

- 使用模板动态生成K8s资源清单，非常需要能提前预览生成的结果。
- 使用--dry-run --debug 选项来打印出生成的清单文件内容，而不执行部署

```shell
# helm install . --dry-run --debug --set image.tag=v2
```

```yaml
[root@k8s-master01 hello]# helm install . --dry-run --debug --set image.tag=v2
[debug] Created tunnel using local port: '43201'

[debug] SERVER: "127.0.0.1:43201"

[debug] Original chart version: ""
[debug] CHART PATH: /root/yaml_pro/helm/second/hello

NAME:   ill-donkey
REVISION: 1
RELEASED: Sat May 28 23:52:50 2022
CHART: hello-world-1.0.0
USER-SUPPLIED VALUES:
image:
  tag: v2

COMPUTED VALUES:
image:
  repository: wangyanglinux/myapp
  tag: v2

HOOKS:
MANIFEST:

---
# Source: hello-world/templates/helm_service.yaml # service部分
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
  selector:
    app: hello-world
---
# Source: hello-world/templates/helm_deployment.yaml # deployment 部分
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: wangyanglinux/myapp:v2
          ports:
            - containerPort: 8080
              protocol: TCP

```

