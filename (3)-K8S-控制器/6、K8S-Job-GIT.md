## <font color = 'red'>Job</font>

**Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束**

- **spec.template格式同Pod**
- **RestartPolicy仅支持Never或OnFailure(Job守护的pod的任务就是保证任务执行完,不可能有Always策略)**
- **单个Pod时，默认Pod成功运行后Job即结束**
- **`.spec.completions`标志Job结束需要成功运行的Pod个数，默认为1**
- **`.spec.parallelism`标志并行运行的Pod的个数，默认为1**
- **`spec.activeDeadlineSeconds`标志失败Pod的重试最大时间，超过这个时间不会继续重试**

# <font color = 'red'>举例</font>

- **求 π 值**

- **算法：马青公式**

```python
π/4=4arctan1/5-arctan1/239
```

- **这个公式由英国天文学教授 约翰·马青 于 1706 年发现。他利用这个公式计算到了 100 位的圆周率。马青公式每计算一项可以得到 1.4 位的 十进制精度。因为它的计算过程中被乘数和被除数都不大于长整数，所以可以很容易地在计算机上编程实现**

```python
# -*- coding: utf-8 -*-
from __future__ import division
# 导入时间模块
import time
# 计算当前时间
time1=time.time()
# 算法根据马青公式计算圆周率 #
number = 1000
# 多计算10位，防止尾数取舍的影响
number1 = number+10
# 算到小数点后number1位
b = 10**number1
# 求含4/5的首项
x1 = b*4//5
# 求含1/239的首项
x2 = b // -239
# 求第一大项
he = x1+x2
#设置下面循环的终点，即共计算n项
number *= 2
#循环初值=3，末值2n,步长=2
for i in xrange(3,number,2):
  # 求每个含1/5的项及符号
  x1 //= -25
  # 求每个含1/239的项及符号
  x2 //= -57121
  # 求两项之和
  x = (x1+x2) // i
  # 求总和
  he += x
# 求出π
pai = he*4
#舍掉后十位
pai //= 10**10
# 输出圆周率π的值
paistring=str(pai)
result=paistring[0]+str('.')+paistring[1:len(paistring)]
print result
time2=time.time()
print u'Total time:' + str(time2 - time1) + 's'
```

# <font color = 'red'>制作镜像</font>

```dockerfile
FROM hub.c.163.com/public/python:2.7
ADD ./main.py /root
CMD /usr/bin/python /root/main.py
```

# <font color = 'red'>Job资源创建</font>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: pi:v1
      restartPolicy: Never
```

- 创建JOB资源（可以看到任务结束后POD为complete状态,）

  ```SHELL
  [root@k8s-master01 job]# kubectl get pod -o wide
  NAME       READY   STATUS      RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
  pi-gnjbj   0/1     Completed   0          2m50s   10.244.1.242   k8s-node01   <none>           <none>
  
  ```

  ```
  [root@k8s-master01 job]# kubectl  logs pi-gnjbj 
  3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989
  Total time:0.00167798995972s
  
  ```

# <font color = 'red'>复杂一点的job</font>

```YAML
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 4  #job结束需要运动的pod个数
  parallelism: 2  # 并行数量
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: pi:v1
      restartPolicy: Never
```





