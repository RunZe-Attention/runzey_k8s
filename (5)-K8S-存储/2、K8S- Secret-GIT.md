# <font color = 'red'>Secret 存在意义</font>

- **configMap使用明文**

- **Secret 解决了密码、token、密钥等敏感数据的配置问题**
- **而不需要把这些敏感数据暴露到镜像或者 Pod Spec 中**
- **Secret 可以以 Volume 或者环境变量的方式使用**



<font color = 'red'>**Secret 有三种类型：**</font>

- **Service Account** ：用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到 Pod 的 `/run/secrets/kubernetes.io/serviceaccount` 目录中
- **Opaque** ：base64 编码格式的 Secret，用来存储密码、密钥等
- **kubernetes.io/dockerconfigjson** ：用来存储私有 docker 仓库的认证信息



## <font color = 'YELLOW'>Service Account</font>

**Service Account 用来访问 Kubernetes API，由 Kubernetes 自动创建，并且会自动挂载到 Pod的 `/run/secrets/kubernetes.io/serviceaccount` 目录中**

```sh
$ kubectl run nginx --image nginx
deployment "nginx" created
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-3137573019-md1u2   1/1       Running   0          13s
$ kubectl exec nginx-3137573019-md1u2 ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```



## <font color = 'red'>Opaque Secret</font>

- base64算法加密"runze"字符串: echo -n 'runze' | base64

  ```bash
  [root@k8s-master01 new]# echo -n 'runze' | base64
  cnVuemU=
  ```

- 反向解码

  ```bash
  [root@k8s-master01 new]# echo -n 'cnVuemU=' | base64 -d
  runze
  ```

  

##### <font color = 'puple'>Ⅰ、创建说明</font>

- **Opaque 类型的数据是一个 map 类型，要求 value 是 base64 编码格式：**

```shell
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

- **创建secret资源**

```yaml
apiVersion: v1
kind: Secret #创建secret资源
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm  # data使用base64算法加密过的value
  username: YWRtaW4=
```



##### <font color = 'puple'>Ⅱ、使用方式</font>

- **1、将 Secret 挂载到 Volume 中**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: secret-test
  name: secret-test
spec:
  volumes: # 定义一个卷 卷的来源为 上面的的secret:mysecret
  - name: volumes12 # 卷的名称为volumes12
    secret:
      secretName: mysecret
  containers:
  - image: wangyanglinux/myapp:v1
    name: db
    volumeMounts:  # 将上面定义的卷volumes12卷挂载在container的/data目录中
    - name: volumes12
      mountPath: "/data"
```

- 与configMap一样,会将secret的data的key作为文件名字,value作为文件内容

  ```BASH
  rwxrwxrwx    1 root     root            15 May 26 13:55 password -> ..data/password
  lrwxrwxrwx    1 root     root            15 May 26 13:55 username -> ..data/username
  /data # cat password 
  1f2d1e2e67df/data # 
  /data # cat username 
  admin/data # 
  
  ```

**2、将 Secret 导出到环境变量中**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: pod-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: pod-deployment
    spec:
      containers:
      - name: pod-1
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
        env:
        - name: TEST_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: TEST_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

- 这与configMap一致,只不过这次环境变量的数据来源于secret资源

## <font color ='yellow'>kubernetes.io/dockerconfigjson</font>

**使用 Kuberctl 创建 docker 仓库认证的 secret**

```shell
$ kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistrykey" created.
```

**在创建 Pod 的时候，通过 `imagePullSecrets` 来引用刚创建的 `myregistrykey`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
    - name: foo
      image: hub.hongfu.com/wangyang/myapp:v1
  imagePullSecrets:
    - name: myregistrykey
```
