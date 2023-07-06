# <font color= 'RED'>POD分类</font>

- **自主式POD**
- **非自主式POD(由控制器管理)**

# <font color = 'red'>官方POD控制器的分类</font>

- **ReplicaController**

- **ReplicaSet**

- **Deployment**

  ```shell
  # 创建顺序
  # Deployment > RS > Pod
  ```

- **DaemonSet**

- **Job**

- **CronJob**

  ```shell
  # 创建顺序
  # CronJob > Job >Pod
  ```

- **StatefulSet(给有状态服务提供服务)**

  ```shell
  # 有序扩容缩(myapp为控制器名称)
  myapp-0 myapp-1 myapp-2 myapp-3 ...
  ```

  ```shell
  # 稳定的网络标识
  ```

  ```shell
  # 稳定的存储卷
  ```

  



