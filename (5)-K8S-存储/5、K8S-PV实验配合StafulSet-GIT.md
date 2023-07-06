# <FONT COLOR = 'RED'>搭建持久化存储NFS服务</FONT>

- 搭建与启动NFS服务

  ```SHELL
  # 在搭建NFS服务的物理机器上部署(10.4.7.52)
  yum install -y nfs-common nfs-utils  rpcbind
  
  # 创建10个目录用于对外服务
  mkdir /nfs/
  cd /nfs
  mkdir {1..10}
  echo "1" > 1/index.html...
  chmod -R 777 /nfs
  
  # 将10个目录写入exports中
  cat /etc/exports
  	/nfs/1 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/2 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/3 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/4 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/5 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/6 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/7 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/8 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/9 *(rw,no_root_squash,no_all_squash,sync)
      /nfs/10 *(rw,no_root_squash,no_all_squash,sync)
      
  	
  # 开机自启	
  systemctl start rpcbind
  systemctl start tart nfs
  systemctl enable rpcbind
  systemctl enable tart nfs
  ```

- 测试NFS服务

  ```SHELL
  # 在K8S节点中进行测试(10.4.7.11)
  mkdir /test
  mount -t nfs 10.4.7.52:/nfs/1 /test
  
  [root@k8s-master01 /]# mount -t nfs 10.4.7.52:/nfs/1 /test
  [root@k8s-master01 /]# cd /test/
  drwxrwxrwx   2 root root  24 May 11 22:26 .
  dr-xr-xr-x. 18 root root 236 May 11 22:25 ..
  -rwxrwxrwx   1 root root   2 May 11 22:18 index.html
  [root@k8s-master01 test]# cat index.html 
  1
  ```

- 释放test目录

  ```
  umount /test 测试完成
  ```

  

# <FONT COLOR = 'RED'>部署PV</FONT>

```yaml
apiVersion: v1 
kind: PersistentVolume
metadata:
  name: nfspv1
spec:
  capacity:
    storage: 1Gi # 容量
  accessModes:
    - ReadWriteOnce # 访问模式
  persistentVolumeReclaimPolicy: Retain # 回收策略:手动回收
  storageClassName: nfs  # PV属于哪一类
  nfs:  # PV实际使用了NFS
    path: /nfs/1
    server: 10.4.7.52
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv2
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/2
    server: 10.4.7.52
    
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv3
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs1
  nfs:
    path: /nfs/3
    server: 10.4.7.52
    
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv4
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/4
    server: 10.4.7.52
--- 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv5
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/5
    server: 10.4.7.52






---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv6
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/6
    server: 10.4.7.52
    
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv7
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfs/7
    server: 10.4.7.52
```

```shell
[root@k8s-master01 pvc]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfspv1   1Gi        RWO            Retain           Available           nfs                     5s
nfspv2   2Gi        RWO            Retain           Available           nfs                     5s
nfspv3   1Gi        RWO            Retain           Available           nfs1                    5s
nfspv4   1Gi        RWX            Retain           Available           nfs                     5s
nfspv5   5Gi        RWO            Retain           Available           nfs                     5s
```

# <FONT COLOR = 'RED'>创建(headless)SVC</font>

- 资源定义

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

```

- 运行结果

  ```BASH
  [root@k8s-master01 pvc]# kubectl get svc
  NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   23d
  nginx        ClusterIP   None         <none>        80/TCP    5s
  
  ```

  

# <FONT COLOR = 'RED'>创建StatefulSet</font>

- 资源定义

```YAML
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
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
          name: web
        volumeMounts:    # 通过PVC挂载外部存储
        - name: pvcName
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # 定义绑定的PVC
  - metadata:
      name: pvcName # PVC资源的名字
    spec:
      accessModes: ["ReadWriteOnce"] # PVC访问模式:单节点读写
      storageClassName: "nfs" # 绑定哪一类PV
      resources:   # 申请资源
        requests:
          storage: 1Gi
          
```

- 查看PVC

```bash
# kubectl get pvc -o wide
NAME                              STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/www-web-0   Bound    nfspv1   1Gi        RWO            nfs            2m46s
persistentvolumeclaim/www-web-1   Bound    nfspv2   2Gi        RWO            nfs            2m44s
persistentvolumeclaim/www-web-2   Bound    nfspv5   5Gi        RWO            nfs            2m42s
```

- 查看pod

```SHELL
[root@k8s-master01 ~]# kubectl get pod -o wide
NAME               READY   STATUS      RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
web-0              1/1     Running     0          5s      10.244.1.110   k8s-node01   <none>           <none>
web-1              1/1     Running     0          45m     10.244.2.106   k8s-node02   <none>           <none>
web-2              1/1     Running     0          45m     10.244.1.108   k8s-node01   <none>           <none>
web-3              1/1     Running     0          9m44s   10.244.2.107   k8s-node02   <none>           <none>
web-4              1/1     Running     0          9m41s   10.244.1.109   k8s-node01   <none>           <none> 


[root@k8s-master01 pv]# curl 10.244.1.110
1

# 如果进入web-0这个POD中:kubectl exec -it web-0 -- /bin/sh 
# cd /usr/share/nginx/html
# 对index.html随便写入写东西shjahfahf
[root@k8s-master01 pv]# curl 10.244.1.110
1
shjahfahf

# 或者干掉这个pod:kubectl delete pod web-0
stafulset重新拉起后,还会看到这个pod中的内容
```

- 如果我们删除statefulset controller,但是没有删除PV与PVC,那么当重新apply 的时候,挂载的PV与PVC依然不变

```shell
[root@k8s-master01 pv]# kubectl get pvc
NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    nfspv1   1Gi        RWO            nfs            136m
www-web-1   Bound    nfspv2   2Gi        RWO            nfs            136m
www-web-2   Bound    nfspv5   5Gi        RWO            nfs            136m
www-web-3   Bound    nfspv7   1536Mi     RWO            nfs            100m
www-web-4   Bound    nfspv6   2Gi        RWO            nfs            100m

```

# <FONT COLOR = 'RED'>删除POD与PVC(应该先删除POD,否则PVC不能被删除)</font>

```shell
删除了POD与PVC之后
[root@k8s-master01 pv]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
nfspv1   1Gi        RWO            Retain           Released    default/www-web-0   nfs                     155m
nfspv2   2Gi        RWO            Retain           Released    default/www-web-1   nfs                     154m
nfspv3   1Gi        RWO            Retain           Available                       nfs1                    154m
nfspv4   1Gi        RWX            Retain           Available                       nfs                     154m
nfspv5   5Gi        RWO            Retain           Released    default/www-web-2   nfs                     154m
nfspv6   2Gi        RWO            Retain           Released    default/www-web-4   nfs                     139m
nfspv7   1536Mi     RWO            Retain           Released    default/www-web-3   nfs                     139m



```

- 可以看到之前BOND的PV变成了Released状态,不能被使用,有2种方式

  - 1.删除PV再apply kubectl delete pv nfspv1 ; kubectl apply -f nfspv1
  - 2.删除绑定信息

  ```SHELL
  [root@k8s-master01 pv]# kubectl edit pv nfspv1 -o yaml
  
  # Please edit the object below. Lines beginning with a '#' will be ignored,
  # and an empty file will abort the edit. If an error occurs while saving this file will be
  # reopened with the relevant failures.
  #
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"nfspv1"},"spec":{"accessModes":["ReadWriteOnce"],"capacity":{"storage":"1Gi"},"nfs":{"path":"/nfs/1","server":"10.4.7.52"},"persistentVolumeReclaimPolicy":"Retain","storageClassName":"nfs"}}
      pv.kubernetes.io/bound-by-controller: "yes"
    creationTimestamp: "2022-05-12T04:09:01Z"
    finalizers:
    - kubernetes.io/pv-protection
    name: nfspv1
    resourceVersion: "346562"
    selfLink: /api/v1/persistentvolumes/nfspv1
    uid: 9104673a-7593-4a5b-adff-8cb3120c4b57
  spec:
    accessModes:
    - ReadWriteOnce
    capacity:
      storage: 1Gi
    `claimRef:
    `  apiVersion: v1
    `  kind: PersistentVolumeClaim
    `  name: www-web-0
    `  namespace: default
    `  resourceVersion: "333473"
    `  uid: d790b692-d1ed-4bb9-b151-482b6d1d3293
    nfs:
      path: /nfs/1
      server: 10.4.7.52
    persistentVolumeReclaimPolicy: Retain
    storageClassName: nfs
    volumeMode: Filesystem
  status:
    phase: Released
  
  # 将spec.claimRef（上述绿色部分）全部删除
  ```

- 可以看到已经释放了

  ```SHELL
  [root@k8s-master01 pv]# kubectl get pv
  NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
  nfspv1   1Gi        RWO            Retain           Available                       nfs                     160m
  nfspv2   2Gi        RWO            Retain           Released    default/www-web-1   nfs                     159m
  nfspv3   1Gi        RWO            Retain           Available                       nfs1                    159m
  nfspv4   1Gi        RWX            Retain           Available                       nfs                     159m
  nfspv5   5Gi        RWO            Retain           Released    default/www-web-2   nfs                     159m
  nfspv6   2Gi        RWO            Retain           Released    default/www-web-4   nfs                     145m
  nfspv7   1536Mi     RWO            Retain           Released    default/www-web-3   nfs                     1
  ```

  

- **还是建议使用删除PV重新创建**
