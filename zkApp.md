# YARN应用程序编写流程分析以Distributed-Shell源码分析为例 #
***
本文简单介绍YARN的组件和功能，主要分析Distributed-Shell的源码，抽象出YARN应用程序的编写流程。

## YARN架构

### 基本思想
YARN的基本思想是将资源管理和作业的调度/监控分为两个独立的进程：一个全局的**ResourceManager**和与每个应用对应的**ApplicationMaster**。**ResourceManager**和每个节点上的**NodeManager**组成了全新的通用操作系统，以分布式的方式管理应用程序。

#### 主要组件
YARN主要有以下三个组件组成
1. **ResourceManager**：全局的进程，为系统中所有应用分配资源。
    - Scheduler：是RM的一个组件，负责分配NodeManager上的Container资源
    - ApplicationsManager：管理集群中的用户作业
2. **NodeManager**：与每台机器对应的从属进程，负责启动应用程序的Container，监控并汇报。
    - Container：节点上一组CPU和内存资源，所有的应用都会运行在Container中。
      要启动一个Container需要：
        - 启动Container内进程的命令行
        - 环境变量
        - 启动Container前所需的本地资源，比如Jar包
        - 安全相关的...
3. **ApplicationMaster**：与每个app相关的进程，与Scheduler协商合适的Container，跟踪应用的状态，监控进程。作为一个Container运行的，通常称为Container0。

#### 应用程序执行流程


![Yarn应用程序](https://raw.githubusercontent.com/blackberrier/StudyNotes/master/MKPhotos/YarnApp.png)

1. 客户端向ResourceManager提交应用程序，并提交用于启动ApplicationMaster的ContainerLaunchContext。
2. ResourceManager找到可以运行一个Container的NodeManager，并在这个Container中启动ApplicationMaster实例
3. 启动ApplicationMaster后，ApplicationMaster向ResourceManager注册，并定期汇报其活跃状态与资源需求。
4. 一旦ResourceManager为App分配一个Container，ApplicationMaster可以构造一个ContainerLaunchContext，并在这个Container对应的NodeManager上启动Container。
5. 应用程序的代码在启动的Container中运行。ApplicationMaster监控正在运行的Container的状态，当工作完成后，可以停止这个Container。
6. 一旦ApplicationMaster整体工作完成后，应从ResourceManager注销。


用户需要实现的是Client和ApplicationMaster。

## Distributed-Shell源码分析

Distributed-Shell主要有三个类组成，Client、ApplicationMaster和DSConstants，功能是在一个Hadoop集群的多个节点上运行shell命令或脚本。
使用方法是
```
./yarn org.apache.hadoop.yarn.applications.distributedshell.Client -jar\
     hadoop-yarn-applications-distributedshell.jar -shell_command ls -shell_args -l
```
DistributedShell相当于Hadoop 2的HelloWorld，理解了内部实现后，进行简单的修改即可快速开发一个可运行的应用程序。

### Client分析 ###

**客户端的工作**主要包括：

- 实例化一个YarnClient对象
- 根据YarnConfiguration对象进行初始化
- 启动YarnClient
- 获取YARN集群参数
- 获取YARN节点报告
- 获取队列信息
- 获取用户的ACL信息
- 创建客户端应用程序
- 向ResourceManager提交应用程序
- 获取应用程序的状态报告


YARN Client有一个main函数，主要流程为
- 创建client对象
- 用命令行参数对其进行初始化
- 调用client的run();

```java
public static void main(String[] args)
    try {
        //1.实例client对象
        Client client = new Client();
        try {
            //2.初始化
            boolean doRun = client.init(args);
            if (!doRun) {
                System.exit(0);
            }
        } catch (IllegalArgumentException e) {
            System.exit(-1);
        }
        //3. 运行（开启服务）
        result = client.run();
    } catch (Throwable t) {
        System.exit(1);
    }
    if (result) {
        System.exit(0);			
    }
    System.exit(2);
}
```

#### Client构造函数

Client类的构造函数对Client对象进行了初始化。

```java
public Client() throws Exception  {
    this(new YarnConfiguration());
}

public Client(Configuration conf) throws Exception  {
    //1.指定ApplicationMaster
    this("org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster", conf);
}

Client(String appMasterMainClass, Configuration conf) {
    this.conf = conf;
    this.appMasterMainClass = appMasterMainClass;
    //2.通过编程库创建yarnClient
    yarnClient = YarnClient.createYarnClient();
    yarnClient.init(conf);
    //3.创建opts，后面解析参数的时候用
    opts = new Options();
    opts.addOption("appname", true, "Application Name. Default value - DistributedShell");
    opts.addOption("jar", true, "Jar file containing the application master");
    opts.addOption("shell_command", true, "Shell command to be executed by " +
        "the Application Master. Can only specify either --shell_command " +
        "or --shell_script");
    ...
    ...
}
```


1. 实例一个YarnConfiguration类的对象conf，该类的静态初始化块中会**读取配置文件**`yarn-default.xml`和`yarn-site.xml`。

2. 指定Client要用到的ApplicationMaster。

3. 实例YarnClient对象，并用conf对yarnClient进行初始化（默认）。
    > YARN将client与ResourceManager的交互抽象出了编程库**YarnClient**，用户可以通过这个库实现创建应用程序客户端，提交应用，状态查询和控制应用等功能。
    > 
    > YarnClient类继承了AbstractService类，所以YarnClient是一个YARN的服务。服务都有管理自身生命周期的初始化、开始和终止等方法，还可以获取服务状态，向服务注册监听器，并在某些事件发生时执行注册的回调函数。
 
   
![YarnClient类图](https://raw.githubusercontent.com/blackberrier/StudyNotes/master/MKPhotos/YarnClient.png)


4. 通过Options类**添加命令行参数**，这些命令行参数就是我们构造客户端时指定的。
比如刚才demo中提到的`-jar`和`-shell_command`等。Jar参数等用于启动ApplicationMaster，shell_command或shell_script等将来会被ApplicationMaster执行。
每个命令行的命令有三个参数，参数名、是否需要用户指定值和参数描述。


#### Client初始化

init会**解析命令行传入的参数**，使用GnuParser解析，Client的构造函数中定义了参数opts，可以认为是一个模板，然后将opts和实际的args传入解析后得到一个CommnadLine对象。

传入的参数有app的name、优先级、ApplicationMaster使用的jar包、内存大小、shell_command和shell_script等。
    - 有些参数是所有app通用的，比如app的name，申请的内存、cpu大小、ApplicationMaster的jar包位置；
    - 有些参数是和当前app相关的，比如这个DistributedShell应用下的shell_command和shell_script命令对别的应用来说是不需要的，当我们改写这个框架来编写其他应用的Client时，应该去掉这几个参数，添加其他应用所必须的参数。
    
这里可以进行输入参数的检查，比如jar包是否存在，app必须的参数（比如当前示例下的shell_command或shell_script）是否存在。


```java
  public boolean init(String[] args) throws ParseException {

    CommandLine cliParser = new GnuParser().parse(opts, args);   
    ...    
    appMasterJar = cliParser.getOptionValue("jar");    
    ...
    if (!cliParser.hasOption("shell_command") && !cliParser.hasOption("shell_script")) {
      throw new IllegalArgumentException(
          "No shell command or shell script specified to be executed by application master");
    } else if (cliParser.hasOption("shell_command") && cliParser.hasOption("shell_script")) {
      throw new IllegalArgumentException("Can not specify shell_command option " +
          "and shell_script option at the same time");
    } else if (cliParser.hasOption("shell_command")) {
      shellCommand = cliParser.getOptionValue("shell_command");
    } else {
      shellScriptPath = cliParser.getOptionValue("shell_script");
    }
    ...

    return true;
  }
```

#### 提交应用

client的run()方法启动客户端，提交应用。

1. 前面提到YarnClient是YARN的一个服务，有管理其生命周期的一些方法。在Client构造函数中，我们对其进行了初始化，在client的run()方法中，我们调用YarnClient的`start()`方法**开启服务**，建立与ResourceManager的**RPC连接**，之后就跟调用本地方法一样。
- 通过此yarnClient查询NodeManager的个数、NodeManager详细信息（ID/地址/Container个数等）、Queue info（只是调试信息）。
2. YarnClient启动后，调用createApplication()方法创建一个app对象（YarnClientApplication类的对象app）。**（怎么实现的？）**
    1. 通过app获得集群最大的可用资源（memory和vCore），并与用户请求的资源进行比较，如果请求资源大于可用资源，提示用户将最大可用资源分配给app。~~这里有个TODO：TODO get min/max resource capabilities from RM and change memory ask if needed(how?)~~。
    2. 获取新的ApplicationID。
3. 构建**ApplicationSubmissionContext**类对象appContext，用于提交作业。
    > ApplicationMaster本身是一个YARN Container，是一个特殊的Container，称为**Contianer0**。启动ApplicationMaster需要一些信息。
    
    客户端把启动ApplicationMaster所需的信息打包到ApplicationSubmissionContext这个数据结构中，主要包括以下几个字段：
    - applicationID:ID
    - applicationName：名称
    - queue：所属队列
    - priority：优先级
    - ContainerLaunchContext：启动ApplicationMaster所在Container所需的上下文
        * usr：Application所属用户
        * resource：启动ApplicationMaster所需的资源，指的是CPU和内存
        * LocalResource：本地资源
        * command：ApplicationMaster的启动命令
        * environment：环境变量
    - ...
    
    接着客户端根据用户的args输入依次设置这些字段来构造ApplicationSubmissionContext。
    1. 设置应用程序的ID、name和所属队列等。
    2. 设置ContainerLaunchContext的**本地资源LocalResources**。
    
       > 用户通过命令行参数指定ApplicationMaster的Jar包在本地文件系统中的路径，这个路径是用户运行YARN client的机器上的本地文件系统路径。但是YARN client所在机器的本地文件系统对YARN集群是不可见的，为了保证文件对集群可见，我们可以将所需的文件放到HDFS上。
       
        1. 上传ApplicationMaster的Jar包至HDFS，并加入到LocalResource中
        调用`addToLocalResources()`方法将appMaster的Jar包上传到HDFS，这个Jar的位置是用户的命令行参数指定的，前面在初始化时被GnuParser解析过。
        在`addToLocalResources()`方法内，首先是将jar包上传至HDFS，然后设置ApplicationMaster的Jar包对应的LocalResources对象的类型（FILE/ARCHIVE）、资源的可见性（APPLICATION...public、usr），另外还添加了修改时间和文件大小。最后将该LocalResources对象（记录了路径、文件类型、修改时间和长度等元数据）添加到LocalResource的Map中。
            - **LocalResource**：LocalResource是Application Master或者Task运行时需要从HDFS下载到本地的资源。在提交APP时，需要上传文件到HDFS，然后指定需要的LocalResource所在HDFS的Path。用一个Map<String, LocalResource>维护本地资源，在往Map里添加资源时，这个文件被NodeManager本地化时会创建一个链接到该文件，这个链接的命名就是key，value就是刚刚创建的LocalResource对象。（xxx->disk1/...）
        2.  上传shell script至HDFS。
        如果用户在命令行中制定了需要DistributedShell需要运行的脚本，为了能在最终的Container中运行这个脚本，需要将其上传至HDFS中。但是ApplicationMaster运行不需要这个脚本，因此不用加入到ApplicationMaster的localResources中。
        3.  上传shell command与shell args至HDFS，并加入到LocalResource中
        
    3. 设置启动ApplicationMaster的**环境变量**。
        1. 3.3.2提到，ApplicationMaster所在的container并不需要用户脚本作为启动信息，所以我们并未将脚本信息加入LocalResource中。但是ApplicationMaster所启动的运行脚本的container需要这个脚本，所以ApplicationMaster需要知道脚本的元数据（路径、修改时间与长度）。所以脚本上传到hdfs后，客户端收集脚本的元数据，并以环境变量的形式传递给ApplicationMaster。使用这个环境变量信息，application master为最终container创建正确的local resource，然后这个container会执行脚本。
        2. 获取客户端的类路径、客户端的YARN环境的类路径，利用这些值构建类路径字符串，添加至环境变量中。
    4. 构造**启动ApplicationMaster的命令行**。
    大多是命令行参数是用户指定的，比如-Xmx、优先级、jar包路径等。YARN通过Environment类提供了便利的方法设置启动命令行。构造一个List接收构造的commands命令行。
    构建出来的commands为
`$JAVA_HOME/bin/java -Xmx10m org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster --container_memory 10`
    
    5. 使用LocalResources对象、环境变量env和命令行commands**实例ContainerLaunchContext类的对象**amContainer。
    6. 准备好ApplicationMaster的启动上下文后，就可以告诉ApplicationSubmissionContext，刚才准备的ContainerLaunchContext是针对ApplicationMaster的，使用的方法就是`appContext.setAMContainerSpec(amContainer);`。

4. 通过YarnClient**提交应用程序**。
   客户端中，提交应用程序的方法会阻塞，直到ResourceManager返回应用程序的状态为ACCEPTED。
    > 如果提交应用程序失败，怎么能重新提交同一个请求？



```java
public boolean run() throws IOException, YarnException {

    //1.开启服务
    yarnClient.start();
    ...
    //2.创建app，得到app的ID和集群可用资源
    YarnClientApplication app = yarnClient.createApplication();
    GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
    int maxMem = appResponse.getMaximumResourceCapability().getMemory();
    int maxVCores = appResponse.getMaximumResourceCapability().getVirtualCores();

    //3.构建app的提交上下文appContext    
    ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
    ApplicationId appId = appContext.getApplicationId();
    appContext.setApplicationName(appName);
    ...	
    //3.2 ApplicationMaster需要的本地资源，如jar包、log文件	
    Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();
    //3.2.1 将application master的jar包传至HDFS并加入LocalResource
    addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),localResources, null);	
    //3.2.2 将shell script上传至hdfs
    if (!shellScriptPath.isEmpty()) {
      Path shellSrc = new Path(shellScriptPath);
      Path shellDst = new Path(fs.getHomeDirectory(), shellPathSuffix);
      fs.copyFromLocalFile(false, true, shellSrc, shellDst);
    }
    //3.2.3 将shell command和shell args上传至hdfs并加入LocalResource
    addToLocalResources(fs, null, shellCommandPath, appId.toString(),localResources, shellCommand);
    addToLocalResources(fs, null, shellArgsPath, appId.toString(),localResources, StringUtils.join(shellArgs, " "));
    ...
    //3.3 设置启动ApplicationMaster的环境变量
    Map<String, String> env = new HashMap<String, String>();
    //3.3.1 脚本位置的环境变量
    env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
    ...
    //3.3.2 设置classpath
    env.put("CLASSPATH", classPathEnv.toString());
    ...
    //3.4 构建启动ApplicationMaster的命令行
    vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
    vargs.add("-Xmx" + amMemory + "m"); 
    ...
    StringBuilder command = new StringBuilder();
    for (CharSequence str : vargs) {
      command.append(str).append(" ");
    } 
    List<String> commands = new ArrayList<String>();
    commands.add(command.toString());
    		
    //3.5 创建Container加载上下文，包含本地资源，环境变量，实际命令
    ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(localResources, env, commands, null, null, null);
    //3.6 告知ApplicationSubmissionContext，刚才准备的ContainerLaunchContext是针对ApplicationMaster的
    appContext.setAMContainerSpec(amContainer);

    //4. 客户端提交ApplicationMaster到ResourceManager
    yarnClient.submitApplication(appContext);

    return monitorApplication(appId);

  }
```


### ApplicationMaster分析 ###

一个新创建的应用程序首先向ResourceManager注册自己，然后以Container形式从ResourceManager请求资源，请求到资源以后跟NodeManager通信，启动Container，监控Container的运行。具体包括：
1. 跟ResourceManager通信，向ResourceManager注册自己，并取得自己的所在位置（机器，端口等）；
2. 和ResourceManager保持heartbeat；
3. 向ResourceManager申请执行任务的container；
4. 对于申请到的container，设置各种执行资源、环境变量、执行命令等；
5. 和ResourceManager通讯，了解每一个container的执行情况。

与YarnClient类似，ApplicationMaster也有一个main函数，也是以此执行构造函数、初始化和运行的流程。
```
public static void main(String[] args) {
    try {
      //1.构造ApplicationMaster
      ApplicationMaster appMaster = new ApplicationMaster();
      //2.初始化
      boolean doRun = appMaster.init(args);
      if (!doRun) {
        System.exit(0);
      }
      //3.运行
      appMaster.run();
      result = appMaster.finish();
    } catch (Throwable t) {
    }
    if (result) {
      System.exit(0);
    } else {
      System.exit(2);
    }
  }
```
#### 构造函数

与客户端类似，构造函数里，先创建一个YarnConfiguration对象，读取YARN环境配置。

#### 初始化
初始化方法中，分析args参数以及system.env()。
- args参数：3.4中构造的启动ApplicationMaster的命令行。
- system.env()的值是client提交时指定的。

#### ApplicationMaster运行

run()方法主要做的工作有：
- 实现一个回调函数处理器**监听来自ResourceManager的事件**；
- 通过YARN API创建一个对象，封装ApplicationMaster与ResourceManager通信的客户端；
- 实现一个回调函数处理**监听来自NodeManager的事件**；
- 通过YARN API创建一个对象，封装ApplicationMaster与NodeManager通信的客户端；
- 实现一个类来启动Container。

1. 设置RM、NM消息的异步处理方法
    1. 初始化ResourceManager回调函数处理器，然后用这个处理器实例一个AMRMClientAsync类的对象amRMClient。
    **AMRMClientAsync类**是YARN所提供的用于ApplicationMaster与ResourceManager异步通信的库。
    2. NodeManager回调函数处理器实例化，并创建一个**NMClientAsync类**的对象nmClientAsync，用于ApplicationMaster和NodeManager之间的异步通信。
    3. AMRMClientAsync和NMClientAsync类都是AbstractService的子类，所以创建对象后都需要以YarnConfiguration对象为参数进行初始化，然后调用start方法开启服务。
2. 向ResourceManager注册ApplicationMaster。
    1. 注册时，需要提供ApplicationMaster的hostname、RPC端口和TrackingURL等基本信息。其中TrackingURL可以由开发者自己定义，YARN的Web界面会对每个应用程序显示这个URL的链接。注册成功后，ApplicationMaster与ResourceManager通信的心跳线程就启动了。
    2. 注册成功后会返回一个response。
        - 通过这个response可以dump out ResourceManager看到的集群资源信息。对container请求的资源进行资源检查，超过可用资源就会报错。
        - 通过这个resonanse可以得到ResourceManager之前分配给这个app的container的list，保存在previousAMRunningContainers中。总需求的container数目减之前已经分配给这个app的container数目可以得到当前可以分配的container数（由于硬件错误可能导致ApplicationMaster暂时不可用，因此可能有多次请求，主要是解决这个问题）
3. 建立发往ResourceManager的用于请求container的请求，然后以这个请求为参数，调用addContainerRequest()方法将请求发往ResourceManager。！！这里然后呢？？


```java
public void run() throws YarnException, IOException {
    ...
    //1. 设置RM、NM消息的异步处理方法
    AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
    amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
    amRMClient.init(conf);
    amRMClient.start();

    containerListener = createNMCallbackHandler();
    nmClientAsync = new NMClientAsyncImpl(containerListener);
    nmClientAsync.init(conf);
    nmClientAsync.start();

    //2.向RM注册，返回一个response
    RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(
            appMasterHostname, appMasterRpcPort,appMasterTrackingUrl);
    //3.response中包含集群的信息
    int maxMem = response.getMaximumResourceCapability().getMemory(); 
    int maxVCores = response.getMaximumResourceCapability().getVirtualCores();
    ...
    //计算需要的container
    List<Container> previousAMRunningContainers = response.getContainersFromPreviousAttempts();
    numAllocatedContainers.addAndGet(previousAMRunningContainers.size());
    int numTotalContainersToRequest = numTotalContainers - previousAMRunningContainers.size();
    //3. 发送container请求
    for (int i = 0; i < numTotalContainersToRequest; ++i) {
      ContainerRequest containerAsk = setupContainerAskForRM();
      amRMClient.addContainerRequest(containerAsk);
    }
    numRequestedContainers.set(numTotalContainers);
    ...
  }
```

##### RMCallbackHandler回调函数
RMCallbackHandler类实现了AMRMClientAsync.CallbackHandler接口。这些方法都比较直接，当某个事件发生时执行响应的方法。
1. **onContainersAllocated()**方法。
    1. 当ApplicationMaster收到ResourceManager分配container的消息（在RM返回的心跳应答中携带）后，执行该方法。
    2. LaunchContainerRunnable实现了Runnable接口，通过这个类来创建线程并启动线程，新线程中启动分配到的container。
    3. 类的构造函数需要两个参数，一个是分配到的container，另外一个是监听NodeManager消息的回调函数处理器。
    4. LaunchContainerRunnable类的run()方法构建最终执行任务的Container的上下文信息，把Container的环境信息，启动命令和本地资源的设置加到ContainerLaunchContext对象中，然后把这个对象提交给Container所在的NodeManager，然后NodeManager就可以启动这个Container了。

```java
private class RMCallbackHandler implements AMRMClientAsync.CallbackHandler {
    @Override
    public void onContainersCompleted(List<ContainerStatus> completedContainers) {
      ...
    }

    @Override
    public void onContainersAllocated(List<Container> allocatedContainers) {
      for (Container allocatedContainer : allocatedContainers) {
        ...
        //创建runnable容器
        LaunchContainerRunnable runnableLaunchContainer = new LaunchContainerRunnable(allocatedContainer, containerListener);
        //新建线程
        Thread launchThread = new Thread(runnableLaunchContainer);
        launchThreads.add(launchThread);
        //线程中提交Container到NodeManager
        launchThread.start();
      }
    }

    @Override
    public void onShutdownRequest() {
        ...
    }

    @Override
    public void onNodesUpdated(List<NodeReport> updatedNodes) {}

    @Override
    public float getProgress() {
        ...
    }

    @Override
    public void onError(Throwable e) {
        ...
    }
  }
```

```java
    public void run() {
      Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

      if (!scriptPath.isEmpty()) {
        yarnUrl = ConverterUtils.getYarnUrlFromURI(new URI(renamedScriptPath.toString()));
        LocalResource shellRsrc = LocalResource.newInstance(yarnUrl,
          LocalResourceType.FILE, LocalResourceVisibility.APPLICATION,
          shellScriptPathLen, shellScriptPathTimestamp);
        localResources.put(Shell.WINDOWS ? ExecBatScripStringtPath : ExecShellStringPath, shellRsrc);
        shellCommand = Shell.WINDOWS ? windows_command : linux_bash_command;
      }

      //生成最终需要执行的command
      vargs.add(shellCommand);
      ...
      StringBuilder command = new StringBuilder();
      for (CharSequence str : vargs) {
        command.append(str).append(" ");
      }

      List<String> commands = new ArrayList<String>();
      commands.add(command.toString());
      //根据命令、环境变量、本地资源等创建Container加载上下文
      ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
        localResources, shellEnv, commands, null, allTokens.duplicate(), null);
      containerListener.addContainer(container.getId(), container);
      //异步启动Container
      nmClientAsync.startContainerAsync(container, ctx);
    }
  }

```

2. onContainersCompleted方法，收到Container执行完毕的消息，检查其执行结果，如果执行失败，则重新发起请求。

##### NMCallbackHandler回调函数
MCallbackHandler回调函数处理NodeManager通知的各种事件。比较直接，修改ApplicationMaster维护的Container执行完成、失败的个数。待Container执行完毕后，可以重启发起请求。失败处理和上面Container执行完毕消息的处理类似，达到了上面问题里所说的loop效果。


> timelineClient?

> DSConstants类，有三个字符串静态变量，提供了一种简单的方法来通过环境变量传递信息给Container。严格来讲，这些环境变量只是一种传递静态信息给ApplicationMaster的方法。ApplicationMaster通过环境变量获取到这些信息后，把它们作为ContainerLaunchContext信息的一部分传递给Container。复杂的应用可能有更多静态数据和动态数据需要传递，除了使用静态变量，还可以使用配置文件，配置文件作为本地资源进行分发，或者作为客户端与ApplicationMaster都能访问到的共享服务比如HDFS。


## 参考资料
《Hadoop YARN权威指南》
https://ieevee.com/tech/2015/05/05/yarn-dist-shell.html
《Hadoop技术内幕：深入解析YARN架构设计与实现原理》
