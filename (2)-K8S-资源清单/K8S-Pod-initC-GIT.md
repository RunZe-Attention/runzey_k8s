# <font color = 'red'>POD InitC</font>

**1.Pod 能够具有多个容器，应用运行在容器里面，但是它也可能有一个或多个先于应用容器启动的 Init 容器**

**2.Init 容器与普通的容器非常像，除了如下两点：**

- **1).Init 容器总是运行到成功完成为止**
- **2).每个 Init 容器都必须在下一个 Init 容器启动之前成功完成** 

**3.如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果** 

**Pod 对应的 restartPolicy 为 Never，它不会重新启动(默认的restartPolicy为Always)**

**4.因为 Init 容器具有与应用程序容器分离的单独镜像，所以它们的启动相关代码具有如下优势：** 

- **它们可以包含并运行实用工具，但是出于安全考虑，是不建议在应用程序容器镜像中包含这些实用工具** 

- **应用程序镜像可以分离出创建和部署的角色，而没有必要联合它们构建一个单独的镜像,(比如tomcat在initC中拉取代码,在mainC中部署使用)** 

- **<font color = 'red'>Init 容器使用 Linux Namespace，所以相对应用程序容器来说具有不同的文件系统视图。因此，它们</font>** 

  **能够具有访问 Secret 的权限，而应用程序容器则不能**

- **它们必须在应用程序容器启动之前运行完成，而应用程序容器是并行运行的，所以 Init 容器能够提供** 

  **了一种简单的阻塞或延迟应用容器的启动的方法，直到满足了一组先决条件**

  

  ```YAML
  # 创建两个svc作为initC实验的前提
  kind: Service
  apiVersion: v1
  metadata:
    name: myservice
  spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9376
  ---
  kind: Service
  apiVersion: v1
  metadata:
    name: mydb
  spec:
    ports:
      - protocol: TCP
        port: 80
        targetPort: 9377
        
  [root@k8s-master01 yaml_pro]# kubectl get svc
  NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   16d
  mydb         ClusterIP   10.102.153.209   <none>        80/TCP    11s
  myservice    ClusterIP   10.103.89.66     <none>        80/TCP    11s
  
  ```
  
  
  
  ```YAML
  # 这是一个简单的initC,使用与containers的同级字段initContainers进行创建初始化的container
  
  
  # vim iniC.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
  spec:
    containers: # 定义mainContianer
    - name: myapp-container
      image:  busybox:1.34.1
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers: # 定义initContainer
    - name: init-myservice
      image: busybox:1.34.1
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
    - name: init-mydb
      image: busybox:1.34.1
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
  ```
  
  ### 运行上述资源清单yaml
  
  ```shell
  kubectl apply -f inic.yaml
  
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME        READY   STATUS     RESTARTS   AGE
  myapp-pod   0/1     Init:0/2   0          4s
  
  [root@k8s-master01 yaml_pro]# kubectl get pod
  NAME        READY   STATUS    RESTARTS   AGE
  myapp-pod   1/1     Running   0          10m
  
  # 初始化完两个initC之后才能进行mainC的建设
  ```
  
  
  
  ```
  1.构造POD
  2.tomcat与nginx
  3.
  ```
  
  
  
  
  
  
  
  
  
  
  
  
  
  













