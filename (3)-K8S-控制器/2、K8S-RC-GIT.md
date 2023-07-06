# <font color = 'red'>RC(ReplicaController)</font>

**作用:ReplicationController（RC）用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的 Pod 来替代；而如果异常多出来的容器也会自动回收**

## 举例

```YAML
apiVersion: v1
kind: ReplicationController # RC控制器kind
metadata:
  name: frontend
spec:
  replicas: 3
  selector: # 标签选择器,必须选择app: nginx标签的POD
    app: nginx
  template:  # 在template中定义POD的样子
    metadata:
      labels:
        app: nginx # 如果想被这个RC控制,那么POD应该具有app: nginx标签
    spec:
      containers:
      - name: php-redis
        image: wangyanglinux/myapp:v1
        env:   # 定义了4个环境变量
        - name: GET_HOSTS_FROM
          value: dns
          name: zhangsan
          value: "123"
        ports:  # container的端口为80(nginx端口)
        - containerPort: 80
```

- 1.按照资源清单创建资源 kubectl apply -f

- 2.会创建以RC名称为基础+MD5值的名称的3个POD

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME             READY   STATUS    RESTARTS   AGE
  frontend-6v7xc   1/1     Running   0          6s
  frontend-ff7cg   1/1     Running   0          6s
  frontend-z2852   1/1     Running   0          6s
  ```

- 3.删除一个POD

  ```
  kubectl delete  pod frontend-6v7xc
  ```

- 4.RC会自动帮您重建一个POD

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME             READY   STATUS    RESTARTS   AGE
  frontend-ff7cg   1/1     Running   0          4m8s
  frontend-nlx6k   1/1     Running   0          10s
  frontend-z2852   1/1     Running   0          4m8s
  ```

- 5.`被Controller控制器空中的标签千万不要改`

  ```SHELL
  我们上面看到的是控制器要控制的POD具有app: nginx标签,因此我们创建完控制器资源后,会看到所有的POD使用的标签为
  NAME             READY   STATUS    RESTARTS   AGE    LABELS
  frontend-df64c   1/1     Running   0          26m    app=nginx
  frontend-hjmnx   1/1     Running   0          26m    app=nginx
  frontend-jgnnj   1/1     Running   0          8m8s   app=nginx
  
  如果我们对一个控制器的标签进行修改:比如对frontend-df64c继续修改
  # kubectl label pod frontend-df64c app=nginx1 --overwrite
  [root@k8s-master01 ~]# kubectl get pod --show-labels
  NAME             READY   STATUS    RESTARTS   AGE   LABELS
  frontend-df64c   1/1     Running   0          28m   app=nginx1
  frontend-hjmnx   1/1     Running   0          28m   app=nginx
  frontend-jgnnj   1/1     Running   0          10m   app=nginx
  frontend-jv86x   1/1     Running   0          20s   app=nginx
  
  # 此时RC控制器为了保证replicas=3需求,又创建了frontend-jv86x这个POD,但是被修改了标签的frontend-df64c已经不受frontend控制器所控制,类似于僵尸般存在,也不会存在于任何调度中,这种方式比较极端,实际工作中,不会这样操作,此例只是证明控制器是通过label与实际的POD进行对接的。
  ```

- 6.删除控制器管理的POD 直接删除POD是不行的,应该删除POD的控制器:kubectl  delete rc rc名字

#### <font color = 'red'>在RC包括RS的matchLabels中selector中的标签应该是pod中标签的子集,这点很符合常理,Controller选择的POD必须完全满足Controller中的selector才能被管控</font>











