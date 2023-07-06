## <font color = 'red'>configMap 描述信息</font>

- **提供了一种配置文件的保存、读写方式**

- **将文件转换成key:value 存储到etcd中、反之一致**

- **ConfigMap 功能在 Kubernetes1.2 版本中引入**
- **许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。**
- **ConfigMap API 给我们提供了向容器中注入配置信息的机制**
- **ConfigMap 可以被用来保存单个属性，也可以用来保存整个配置文件或者 JSON 二进制等对象**



## <font color = 'red'>ConfigMap 的创建</font>

<font color = 'puple'>**Ⅰ、使用目录创建**</font>

```BASH
$ ls docs/user-guide/configmap/kubectl/
        game.file
        ui.file
```

```BASH
$ cat docs/user-guide/configmap/kubectl/game.file
        version=1.17
        name=dave
        age=18
```

```BASH
$ cat docs/user-guide/configmap/kubectl/ui.file
        level=2
        color=yellow
```

```bash
$ kubectl create configmap game-config  --from-file=docs/user-guide/configmap/kubectl
# 我们直接使用命令行的方式创建configMap资源

# game-config:configMap资源名字
# --from-file:创建的文件资源所在的目录
```

-  **会将文件名字作为key,资源下面所有的key=value作为values**

  ```YAML
  [root@k8s-master01 use_dir]# kubectl get cm 
  NAME          DATA   AGE
  game-config   2      75s
  
  [root@k8s-master01 use_dir]# kubectl get configMap game-config -o yaml
  apiVersion: v1
  data:
    game.file: |
      version=1.17
      name=dave
      age=18
    ui.file: |
      level=2
      color=yellow
  kind: ConfigMap
  metadata:
    creationTimestamp: "2022-05-26T09:28:48Z"
    name: game-config
    namespace: default
    resourceVersion: "1034053"
    selfLink: /api/v1/namespaces/default/configmaps/game-config
    uid: e7622d1d-18c7-4802-8acd-7ef2d17e2b67
  
  ```
  
- **通过描述命令查看一下key与value**

```shell
[root@k8s-master01 use_dir]# kubectl describe cm game-config
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.file:
----
version=1.17
name=dave
age=18

ui.file:
----
level=2
color=yellow

Events:  <none>

```

##### <font color = 'puple'>Ⅱ、使用文件创建</font>

- **也可以直接从文件中进行创建,这与从目录中是一样的,或者是一种更简单的方式**

- **只要指定为一个文件就可以从单个文件中创建 ConfigMap**

  ```shell
  # kubectl create configmap game-config-2 --from-file=./game.file
  ```

  ```shell
  [root@k8s-master01 use_dir]# kubectl describe cm game-config-2
  Name:         game-config-2
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  
  Data
  ====
  game.file:
  ----
  version=1.17
  name=dave
  age=18
  
  Events:  <none>
  
  ```

-  **可以看见此时只有game.file的数据域**

##### <FONT COLOR = 'puple'>Ⅲ、使用字面值创建</font> 

- **使用文字值创建，利用 `—from-literal` 参数传递配置信息，该参数可以使用多次，格式如下**
- **这是一种最简单的创建configMap资源的方式**

```bash
$ kubectl create configmap literal-config --from-literal=name=dave --from-literal=password=pass
```

```bash
[root@k8s-master01 use_dir]# kubectl describe configMap literal-config 
Name:         literal-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
name:
----
dave
password:
----
pass
Events:  <none>

```

```bash
$ kubectl get configmaps literal-config -o yaml

[root@k8s-master01 use_dir]# kubectl get configmaps literal-config -o yaml
apiVersion: v1
data:
  name: dave
  password: pass
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-26T09:39:24Z"
  name: literal-config
  namespace: default
  resourceVersion: "1034980"
  selfLink: /api/v1/namespaces/default/configmaps/literal-config
  uid: 169a686f-ea2b-49bf-bb8e-d9ee12921a50
```



## <font color = 'red'>Pod 中使用 ConfigMap</font>

- **使用 ConfigMap 来替代环境变量**

- **用 ConfigMap 设置命令行参数**

**Ⅰ、使用 ConfigMap 来替代环境变量**

- 创建第一个configMap资源:literal-config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: literal-config
  namespace: default
data:
  name: dave
  password: pass
```

- 创建第二个configMap资源:env-config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

- 在POD中使用config资源的变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test-pod
spec:
  containers:
    - name: test-container
      image: wangyanglinux/myapp:v1
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: USERNAME  # 在test-container容器中定义USERNAME环境变量
          valueFrom:      # USERNAME值的来源
            configMapKeyRef: # 来源configMap资源
              name: literal-config # 具体哪个configMap资源
              key: name            # 具体哪个configMap资源里面的哪个字段
        - name: PASSWORD
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password
      envFrom: # 将cm资源env-config中的全部环境变量都注入到container中
        - configMapRef:
            name: env-config
  restartPolicy: Never
```

- 可以看到POD中存在上面两个cm资源中创建的三个环境变量

  ```BASH
  [root@k8s-master01 new]# kubectl logs cm-env-test-pod | grep PASSWORD
  PASSWORD=pass
  [root@k8s-master01 new]# kubectl logs cm-env-test-pod | grep USERNAME
  USERNAME=dave
  [root@k8s-master01 new]# kubectl logs cm-env-test-pod | grep log_level
  log_level=INFO
  ```

##### <font color = 'puple'>Ⅱ、**用 ConfigMap 设置命令行参数**</font>

- 创建Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-command-dapi-test-pod
spec:
  containers:
    - name: test-container
      image: wangyanglinux/myapp:v1
      command: [ "/bin/sh", "-c", "echo $(USERNAME) $(PASSWORD)" ] # 系统启动命令是可以调用环境变量的
      env:
        - name: USERNAME 
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: name
        - name: PASSWORD
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password
  restartPolicy: Never
```

- 查看configMap资源literal-config的具体描述

```shell
[root@k8s-master01 storage]# kubectl get cm literal-config -o yaml
apiVersion: v1
data:
  name: dave
  password: pass
kind: ConfigMap
metadata:
  creationTimestamp: "2022-05-09T05:52:47Z"
  name: literal-config
  namespace: default
  resourceVersion: "236248"
  selfLink: /api/v1/namespaces/default/configmaps/literal-config
  uid: be4e21a3-098c-4bb0-9d0c-d80f20a6201d

```

- 查看pod内部进程的输出log

```shell
[root@k8s-master01 storage]# kubectl logs cm-command-dapi-test-pod
dave pass
```



##### <font color = 'puple'>Ⅲ、**通过数据卷插件使用ConfigMap**</font>

- **在数据卷里面使用这个 ConfigMap，有不同的选项。最基本的就是将文件填入数据卷，在这个文件中，键就是文件名，键值就是文件内容**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-test-pod
spec:
  containers:
    - name: test-container
      image: wangyanglinux/myapp:v1
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: literal-config
  restartPolicy: Never
```



# <font color = 'red'>ConfigMap 的热更新</font>

- **configMap以数据卷的形式才能提供热更新服务**

```yaml
apiVersion: v1 # configMap资源 为deployment提供存储服务
kind: ConfigMap
metadata:
  name: log-config 
  namespace: default
data:
  LOG_LEVEL: INFO
---
apiVersion: extensions/v1beta1 # Deployment资源
kind: Deployment
metadata:
  name: hot-update
spec:
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume # 卷的名字叫做config-volume
          mountPath: /etc/config
      volumes:
        - name: config-volume # 卷是基于configmap资源log-config挂载上去的
          configMap:
            name: log-config
```

- **下面我们检查configMap已挂在卷的形式将注入到pod中**

  - 注意已经挂载卷的形式注入到POD中
  - configMap中data中的key将成为文件名字
  - configMap中data中的value将成为文件内容

  - 进入到创建好的pod中去

    ```bash
    $ kubectl exec -it hot-update-c484b98b4-pwktw -c my-nginx -- /bin/sh
    ```

  - 会发现/etc/config存在一个软连接文件LOG_LEVEL,文件内容为INFO

    ```BASH
    /etc/config # cat  LOG_LEVEL 
    INFO
    ```

    

- 下面我们修改log-config 中的data:**kubectl edit configmap log-config** 

  ```bash
  # LOG_LEVEL : 从INFO级别修改为error级别
  ```

- 再次查看hot-update-c484b98b4-pwktw POD中my-nginx container的/etc/config/LOG_LEVEL文件

  ```basic
  # cat /etc/config/LOG_LEVEL 
  # error
  ```



# <font color = 'red'>**ConfigMap 更新后滚动更新 Pod**</font>

更新 ConfigMap 目前并不会触发相关 Pod 的滚动更新，可以通过修改 pod annotations 的方式强制触发滚动更新

```bash
$ kubectl patch deployment my-nginx --patch '{"spec": {"template": {"metadata": {"annotations": {"version/config": "20190411" }}}}}'
```

这个例子里我们在 `.spec.template.metadata.annotations` 中添加 `version/config`，每次通过修改 `version/config` 来触发滚动更新



<font color = 'red'>**！！！ 更新 ConfigMap 后：**</font>

- **使用该 ConfigMap 挂载的 Env 不会同步更新**
- **使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新**

