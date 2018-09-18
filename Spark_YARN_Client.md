## 总体介绍  

1. Spark应用程序的运行架构主要包括SparkContext（驱动程序）、ClusterManager（集群资源管理器）和Executor（任务执行进程）三部分。  
    - SparkContext用于负责与ClusterManager通信，进行资源申请、任务分配和监控，负责作业执行的全生命周期管理。  
    - ClusterManager提供了资源的分配和管理，在YARN模式下由ResourceManager担任。  
    - 当SparkContext对运行的作业进行划分并分配资源后，会把任务发送Executor进行运行。  
    
2. TaskScheduler负责具体任务的调度执行，定义了任务调度相关的实现方法，实现类是TaskSchedulerImpl是重要的子类。SchedulerBackend负责应用程序运行期间与底层资源调度系统交互，YARN运行模式分为YarnClientSchedulerBackend和YarnClusterSchedulerBackend。  
3. 应用程序运行过程是，在SparkContext启动时，调用TaskScheduler.start方法启动TaskScheduler调度器；当DAGScheduler调度阶段和任务拆分完毕时，调用TaskScheduler.submitTasks方法提交任务；SchedulerBackend接到执行任务时，通过reviewOffers方法分配运行资源并启动运行节点的Executor；由TaskScheduler接收任务运行状态，如果任务运行完毕，则继续分配，直至应用程序的所有任务运行完毕。  
 
## YARN-Client模式  

1. 启动SparkContext  
    (1) 使用spark-submit脚本提交应用程序，命令如下  
`./spark-submit --master yarn --deploy-mode client --class org.apache.spark.examples.JavaWordCount --name testPi --driver-memory 2G --driver-cores 1 --executor-memory 2G --num-executors 3 ../examples/jars/spark-examples_2.11-2.1.0-hbp2.0.0.jar /hbp/jobscheduler/storage/jobs/group001/job_1536544041884_0001/job.info`  
    (2) 执行SparkSubmit类的main()方法，解析传入参数，解析后参数为SparkSubmitArguments类。  
    (3) 执行submit()方法提交应用。分为两步，  
第一步准备启动环境：调用prepareSubmitEnvironment()方法，将参数解析为执行类主函数所需的参数（待count的文件路径）、classpath（指定的jar包，spark-examples_2.11-2.1.0-hbp2.0.0.jar）、系统参数（spark.jars=spark-examples_2.11-2.1.0-hbp2.0.0.jar）和需要运行的类（org.apache.spark.examples.JavaWordCount），返回一个4元元组。  
第二步使用第一步准备的启动环境执行应用程序主函数。MutableURLClassLoader,加载执行类，反射执行main方法。  
    (4) SparkContext初始化，提交的spark应用的第一步一般是构造sparksession初始化或者sparkcontext初始化：创建Spark执行环境SparkEnv，Hadoop配置及Executor环境变量的设置，创建TaskScheduler、SchedulerBackend和DAGScheduler，启动TaskScheduler，创建和启动Executor分配管理器ExecutorAllocationManager。启动ListenerBus，并post环境信息和应用信息，添加确保context停止的hook。  
    (5) 调用TaskSchedulerImpl的start方法实现TaskScheduler的启动(Yarnscheduler继承TaskschedulerImpl，TaskschedulerImpl实现TaskScheduler)。  
    - 启动YarnClientSchedulerBackend；  
        - 构造Client；  
        - 调用Client的submitApplication方法提交应用  
            - 连接LauncherBackend  
            - 创建YarnClient，用于与YARN集群交互
            - 向ResourceManager申请应用程序编号
            - 确认Yarn集群中是否有足够的资源启动ApplicationMaster
            - 构造ContainerLaunchContext，用于启动applicationmaster的container，包括启动环境，java选项，启动命令：启动amClass是org.apache.spark.deploy.yarn.ExecutorLauncher
            - 通过containerContext构造submissionContext：主要是设置app名称，队列，类型和资源大小等
            - 调用YarnClient的submitApplication方法提交应用，参数是submissionContext
        - 等待应用状态为running状态
        - 调用YarnClientSchedulerBackend的父类CoarseGrainedSchedulerBackend的start方法，启动driverEndpoint
            
2. ResourceManager收到请求后，在集群中选择一个Nodemanager，为这个应用程序分配一个container，在这个container中启动ApplicationMaster。

3. 启动Am后，执行ExecutorLauncher类的main方法。
    - 在registerAM方法中，获取应用程序编号，获取DriverEndpoint引用地址，通过YarnRMClient的客户端，ResourceManager向终端点DriverEndpoint发送消息通知ApplicationMaster启动完毕
    - 通过YarnAllocator的allocateResource方法申请executor资源  
        - 在allocateResource方法中，通过AMRMClient的allocate方法申请container执行executor
        - 申请到container后，在runAllocatedContainers方法中，调用ExecutorRunnable的run方法，构造NMClient启动container。在container中启动CoarseGrainedExecutorBackend，启动后，会向客户端中的SparkContext注册并申请任务集
        
4. SparkContext分配任务集给CoarseGrainedExecutorBackend并追踪运行状态。  

5. 应用程序运行完成后，SparkContext向ResourceManager申请注销并关闭。


## YARN-Cluster模式  
1. 在SparkContext启动时，初始化DAGScheduler调度器，在createTaskScheduler方法中匹配为Yarn-Cluster模式，通过反射初始化YarnClusterScheduler和YarnClusterSchedulerBackend两个对象，其中YarnClusterSchedulerBackend是CoarseGrainedSchedulerBackend的子类，YarnClusterScheduler是TaskschedulerImpl的子类。
2. SparkSubmit类的main方法，第一步是准备环境，prepareSubmitEnvironment()方法，childMainClass是org.apache.spark.deploy.yarn.Client。  
第二步使用第一步准备的启动环境执行应用程序主函数。MutableURLClassLoader,加载Client类，执行main方法。
3. 客户端执行run方法，调用submitApplication方法提交应用程序。启动Client向YARN提交应用程序，包括启动ApplicationMaster的命令，提交给ApplicationMaster的程序和需要在executor中运行的程序。  
    - 创建YarnClient，用于与YARN集群交互  
    - 构造ContainerLaunchContext，用于启动applicationmaster的container，包括启动环境，java选项，启动命令：启动amClass是org.apache.spark.deploy.yarn.ApplicationMaster  
    - 通过containerContext构造submissionContext：主要是设置app名称，队列，类型和资源大小等  
    - 调用YarnClient的submitApplication方法提交应用，参数是submissionContext
4. ResourceManager收到请求后，在集群中选择一个Nodemanager，为该应用程序分配第一个container，在这个container中启动ApplicationMaster，在这个ApplicationMaster中进行SparkContext的初始化。(与YarnClient模式的区别，SparkContext初始化的位置) 
    - 执行ApplicationMaster的run方法  
    - 在runDriver方法中，执行startUserApplication方法启动用户应用程序，用户的应用程序会启动SparkContext  
    - 等待SparkContext初始化完成
5. ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManager查看应用程序的状态。ApplicationMaster采用轮询的方式为每个任务申请资源，监控他们的运行状态。  
6. ApplicationMaster申请到资源后，与对应的Nodemanager通信，要求在获得的container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请任务集。  
7. SparkContext分配任务集给CoarseGrainedExecutorBackend并追踪运行状态。  
8. 应用程序运行完成后，SparkContext向ResourceManager申请注销并关闭。
