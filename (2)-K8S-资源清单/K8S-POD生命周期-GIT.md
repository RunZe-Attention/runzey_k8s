# <font color = 'red'>POD的生命周期</font>

- POD的启动到死亡的一系列过程

![常用解释1](.\images\POD生命周期.png)



## 上图讲解

1.当我们创建一个POD的时候,K8S内部都会给这个POD创建一个叫做pause的container,必须存在的默认container,有两个作用

- (1).初始化网络栈
- (2).共享网络卷

2.当pause container被创建完成后会创建initC(C代表container),initC有几个特性

- (1).initC可以不被定义,不是必须的,一般就是为mainC做准备,比如拉取代码等等.
- (2).iniC数量范围为 0~MAX,并且initC的创建顺序是根据yaml文件中的定义顺序一次创建的,串行执行.
- (3).任何initC的创建失败(即错误码不为0),都会导致整个POD的重建,即从创建pause重新开始
- <font color = 'blue'>(4).initC可以挂载Secret对象,但是mainC不可以</font>

3.当iniC创建完成之后,就开始创建我们的mainC了。

- (1).mainC的数量的取值范围是1~max,也就是说POD中必须存在业务container,否则没有意义
- (2).所有的mainC都是并行创建,只是快慢的问题
- (3).单个POD内部的mainC不能有port冲突以及container名称冲突
- (4).对于单个mainC有以下特性
  - 1).当一个mainC启动前,会执行一个启动前的钩子函数,值得说明的是,这个钩子函数不是与mainC进程串行的,而是一个单独的fork,也就是说,`不一定是启动前的hook完成后,mainC才能正常执行`。
  - 2).当一个mainC结束前,一定会执行结束的hook,这个是串行的
  - 3).启动hook与结束hook是为mainC做`准备`与`善后`工作
  - 4).mainC声明周期中存在两种探测
    - 1.就绪探测:在mainC启动后的一段时间内,并以一定的时间间隔定期向mainC进行探测,当就绪探测就绪后,方可对外进行服务
    - 2.存活探测:在pod中的container对外服务的时候,判断mainC是否存活,默认的话,挂掉的话给你拉起
  - 5).探测结果有三种类型: <font color = 'red'>`成功、失败、未知`</font>



