#### 1. Application
   Spark应用程序，指的是用户编写的Spark应用程序，包含Driver的功能代码一级分布在集群中多个节点上运行的Executer代码
#### 2. Driver
   Spark中Driver就是application中main()并创建sparkcontext, SparkContext是为运行apllication应用程序准备的应用环境。
#### 3. Executer
   Application运行在Worker节点上的一个`进程`，该进程负责运行Task，并且负责将数据存在内存或者磁盘上，每个Application都有各自独立的一批Executor
    
