# <font color = 'red'>RS(ReplicaSet)</font>

**作用:在新版本的 Kubernetes 中建议使用 ReplicaSet 来取代 ReplicationController 。ReplicaSet 跟ReplicationController 没有本质的不同，只是名字不一样，并且 ReplicaSet 支持集合式的 selector**

**两种匹配方式:**

- <font color = 'puple'>`matchExpressions`</font>
- <font color = 'puple'>`matchLabels`</font>

```SHELL
FIELDS:
   matchExpressions	<[]Object>
     matchExpressions is a list of label selector requirements. The requirements
     are ANDed.

   matchLabels	<map[string]string>
     matchLabels is a map of {key,value} pairs. A single {key,value} in the
     matchLabels map is equivalent to an element of matchExpressions, whose key
     field is "key", the operator is "In", and the values array contains only

```

## <font color = 'red'>举例</font> 

### 一、selector.matchLabels

```YAML
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:  # 使用与RC同样形式的标签直接匹配
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: myapp
        image: wangyanglinux/myapp:v1
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80
```

### 二、selector.matchExpressions

- **In：label 的值在某个列表中**
- **NotIn：label 的值不在某个列表中**
- **Exists：某个 label 存在**
- **DoesNotExist：某个 label 不存在**

####  <font color = 'puple'>(1).Exists规则</font>

```YAML
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-demo
spec:
  selector:
    matchExpressions:
      - key: app       
        operator: Exists   #只要POD中存在key为app这样的label,那么pod就可以被这个rs-demo控制   
  template:
    metadata:
      labels:
        app: spring-k8s
    spec:
      containers:
        - name: rs-c1
          image: wangyanglinux/myapp:v1
          ports:
            - containerPort: 80
```

- 运行后的效果:

  ```shell
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME            READY   STATUS    RESTARTS   AGE
  rs-demo-p7jj6   1/1     Running   0          5s
  ```

- 可以看到由于没有指定rs.spec.replicas,所以默认是:1,如果我们想实现扩容,可以使用kubectl scale命令

- kubectl scale --help

  ```shell
  Examples:
    # Scale a replicaset named 'foo' to 3.
    kubectl scale --replicas=3 rs/foo
    
    # Scale a resource identified by type and name specified in "foo.yaml" to 3.
    kubectl scale --replicas=3 -f foo.yaml
    
    # If the deployment named mysql's current size is 2, scale mysql to 3.
    kubectl scale --current-replicas=2 --replicas=3 deployment/mysql
    
    # Scale multiple replication controllers.
    kubectl scale --replicas=5 rc/foo rc/bar rc/baz
    
    # Scale statefulset named 'web' to 3.
    kubectl scale --replicas=3 statefulset/web
  
  ```

- 现在我们对上述Controller进行扩容

  ```shell
  # kubectl scale --replicas=3 rs/rs-demo
  ```

  ```SHELL
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME            READY   STATUS    RESTARTS   AGE
  rs-demo-4hlxl   1/1     Running   0          5s
  rs-demo-9rq67   1/1     Running   0          5s
  rs-demo-p7jj6   1/1     Running   0          4m7s
  
  # 可以看到除了最早的rs-demo-p7jj6,又增加了两个re-demo-MD5,且AGE比较短
  ```

#### <font color = 'puple'>(2).In规则(NotIn反之不再赘述)</font>

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-demo-in
spec:
  replicas: 10
  selector:
    matchExpressions: # 匹配规则为 标签为app values必须为spring-k8s或者spring-k8s1其中一个即可以
      - key: app    
        operator: In
        values:
        - GPU-k8s
        - CPU-k8s        
  template:
    metadata:
      labels:
        app: GPU-k8s
    spec:
      containers:
        - name: rs-c1
          image: wangyanglinux/myapp:v1
          ports:
            - containerPort: 80
```

- 运行结果

  ```SHELL
  # NAME               READY   STATUS    RESTARTS   AGE
  # rs-demo-in-c8sxw   1/1     Running   0          4s
  # rs-demo-in-c99g8   1/1     Running   0          4s
  # rs-demo-in-h7z6r   1/1     Running   0          4s
  # rs-demo-in-jxhnw   1/1     Running   0          4s
  # rs-demo-in-k99zm   1/1     Running   0          4s
  # rs-demo-in-pqmcs   1/1     Running   0          4s
  # rs-demo-in-qg9mw   1/1     Running   0          4s
  # rs-demo-in-rl5nj   1/1     Running   0          4s
  # rs-demo-in-stmmv   1/1     Running   0          4s
  # rs-demo-in-t2ljh   1/1     Running   0          4s
  
  ```

- 更高级的版本会有更加完善的报错机制,Controller如果没有任何可以标签匹配的pod,那么会报错

  ```shell
  # The ReplicaSet "rs-demo-in" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"app":"GPU-k8s1"}: `selector` does not match template `labels`
  ```

  
