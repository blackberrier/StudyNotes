#YARN应用程序编写流程分析以Distributed-Shell源码分析为例#
***
本文简单介绍YARN的组件和功能，主要分析Distributed-Shell的源码，抽象出YARN应用程序的编写流程。

##YARN架构

###基本思想
YARN的基本思想是将资源管理和作业的调度/监控分为两个独立的进程：一个全局的**ResourceManager**和与每个应用对应的**ApplicationMaster**。**ResourceManager**和每个节点上的**NodeManager**组成了全新的通用操作系统，以分布式的方式管理应用程序。

####主要组件
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

####应用程序执行流程
1. 客户端向ResourceManager提交应用程序，并提交用于启动ApplicationMaster的ContainerLaunchContext。
2. ResourceManager找到可以运行一个Container的NodeManager，并在这个Container中启动ApplicationMaster实例
3. 启动ApplicationMaster后，ApplicationMaster向ResourceManager注册，并定期汇报其活跃状态与资源需求。
4. 一旦ResourceManager为App分配一个Container，ApplicationMaster可以构造一个ContainerLaunchContext，并在这个Container对应的NodeManager上启动Container。
5. 应用程序的代码在启动的Container中运行。ApplicationMaster监控正在运行的Container的状态，当工作完成后，可以停止这个Container。
6. 一旦ApplicationMaster整体工作完成后，应从ResourceManager注销。


用户需要实现的是Client和ApplicationMaster。

##Distributed-Shell源码分析

Distributed-Shell主要有三个类组成，Client、ApplicationMaster和DSConstants，功能是在一个Hadoop集群的多个节点上运行shell命令或脚本。
使用方法是
```
yarn org.apache.hadoop.yarn.applications.distributedshell.Client -jar hadoop-yarn-applications-distributedshell.jar -shell_command ls -shell_args -l
```
DistributedShell相当于Hadoop 2的HelloWorld，理解了内部实现后，进行简单的修改即可快速开发一个可运行的应用程序。

###Client分析###

客户端的工作主要包括：

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


YARN Client有一个main函数，主要流程为创建client对象，用命令行参数对其进行初始化，然后调用client的run()方法。

```java
public static void main(String[] args)
    try {
        Client client = new Client();
        try {
            boolean doRun = client.init(args);
            if (!doRun) {
                System.exit(0);
            }
        } catch (IllegalArgumentException e) {
            System.exit(-1);
        }
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

####Client构造函数

Client类的构造函数对Client对象进行了初始化。

```java
public Client() throws Exception  {
    this(new YarnConfiguration());
}
```

```java
public Client(Configuration conf) throws Exception  {
    this("org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster", conf);
}
```

```java
Client(String appMasterMainClass, Configuration conf) {
    this.conf = conf;
    this.appMasterMainClass = appMasterMainClass;
    yarnClient = YarnClient.createYarnClient();
    yarnClient.init(conf);
    opts = new Options();
    opts.addOption("appname", true, "Application Name. Default value - DistributedShell");
    opts.addOption("priority", true, "Application Priority. Default 0");
    ...
    ...
}
```


1. 确保YARN的环境对客户端是可用的，创建客户端时，新建一个YarnConfiguration类的对象conf，该类的静态初始化块中会读取yarn-default.xml和yarn-site.xml这两个配置文件。

2. 创建Client时指定Client要用到的ApplicationMaster。

3. 创建yarnClient对象，并用conf对yarnClient进行初始化（默认）。
YARN将client与ResourceManager的交互抽象出了编程库**YarnClient**，用户可以通过这个库实现创建创建应用程序客户端，提交应用，状态查询和控制应用等功能。
为啥要用conf配置信息进行初始化呢？YarnClient类继承了AbstractService类，所以YarnClient是一个YARN的服务。服务都有管理自身生命周期的初始化、开始和终止等方法，还可以获取服务状态，向服务注册监听器，并在某些事件发生时执行注册的回调函数。
![YarnClient类图](file:///D:/study/printscreen/YarnClient.png)

4. 通过Options添加命令行参数。对每个命令行命令有三个参数，参数名、是否需要用户指定值和参数描述，比如上述的appname，priority等。


####Client初始化

init会解析命令行传入的参数，例如使用的jar包、内存大小、cpu个数等。代码里使用GnuParser解析，Client的构造函数中定义了所有的参数opts，可以认为是一个模板，然后将opts和实际的args传入解析后得到一个CommnadLine对象，在这里可以进行输入参数的检查。


```
  public boolean init(String[] args) throws ParseException {

    CommandLine cliParser = new GnuParser().parse(opts, args);
    
    ...    
    appMasterJar = cliParser.getOptionValue("jar");    
    ...
    containerMemory = Integer.parseInt(cliParser.getOptionValue("container_memory", "10"));
    containerVirtualCores = Integer.parseInt(cliParser.getOptionValue("container_vcores", "1"));
    numContainers = Integer.parseInt(cliParser.getOptionValue("num_containers", "1"));
    if (containerMemory < 0 || containerVirtualCores < 0 || numContainers < 1) {
      throw new IllegalArgumentException("Invalid no. of containers or container memory/vcores specified,"
          + " exiting."
          + " Specified containerMemory=" + containerMemory
          + ", containerVirtualCores=" + containerVirtualCores
          + ", numContainer=" + numContainers);
    }
    ...

    return true;
  }
```

####提交应用

client的run()方法启动客户端，提交应用。

1. 前面提到YarnClient是YARN的一个服务，有管理其生命周期的一些方法。在Client构造函数中，我们对其进行了初始化init，在client的run()方法中，调用YarnClient的start()方法，会建立与ResourceManager的RPC连接。
2. YarnClient启动后，通过它创建一个app（YarnClientApplication类的对象app）。
    1. 通过app获得集群最大的可用资源（memory和vCore），并与用户请求的资源进行比较，如果请求资源大于可用资源，提示用户将最大可用资源分配给app。这里有个TODO：TODO get min/max resource capabilities from RM and change memory ask if needed，就是说获取RM的最大可用资源，如果有必要的话更改用户请求的资源。
    2. 通过app获取app的提交上下文**ApplicationSubmissionContext**类对象appContext，进而获取app的ID，设置app的name等。
3. ApplicationMaster本身是一个YARN Container，是一个特殊的Container，称为**Contianer0**。我们需要一些上下文信息来启动AM本身的这个Container0，描述上下文信息的是**ContainerLaunchContext**类。
    1. 设置ContainerLaunchContext的本地资源。
    
       > 用户通过命令行参数指定ApplicationMaster的Jar包在本地文件系统中的路径，这个路径是用户运行YARN client的机器上的本地文件系统路径。但是YARN client所在机器的本地文件系统对YARN集群是不可见的，为了保证文件对集群可见，我们可以将所需的文件放到HDFS上。conf中包含了HDFS的信息。
       
        1. 上传ApplicationMaster的Jar包至HDFS。
        调用addToLocalResources()方法将appMasterJar上传到HDFS，这个Jar是用户的命令行参数指定的，前面在初始化时被GnuParser解析过。在addToLocalResources()方法内，首先是将jar包上传至HDFS，然后设置ApplicationMaster的Jar包对应的LocalResources对象的类型（FILE/ARCHIVE）、资源的可见性（APPLICATION...public、usr），另外还添加了修改时间和文件大小。最后将该LocalResources对象添加到LocalResource Map中。
            - *注LocalResource*：LocalResource是Application Master或者Task运行时需要从HDFS下载到本地的资源，在提交APP时，需要上传文件到HDFS，然后指定需要的LocalResource所在HDFS的Path。在往Map<String, LocalResource>里添加资源时，这个文件被NodeManager本地化时会创建一个链接到该文件，这个链接的命名就是key，value就是刚刚创建的LocalResource对象。（xxx->disk1/...）
        2.  上传shell script。ApplicationMaster运行不需要shell脚本，因此不用加入到am的localResources中。final container中执行shell脚本。（？？）
        3.  shell command 与 shell args
        
    2.  设置启动ApplicationMaster的环境变量。
    脚本上传到hdfs后，采用环境变量来记录脚本的位置。使用这个环境变量信息，application master为最终container创建正确的local resource，然后这个container会执行脚本（？？）
    在ApplicationMaster的设置环境中，我们获取客户端的类路径、客户端的YARN环境的类路径，利用这些值构建类路径字符串，添加至环境变量中。
    3. 构造启动ApplicationMaster的命令行。
    大多是命令行参数是用户指定的，比如-Xmx、优先级、jar包路径等。YARN通过Environment类提供了便利的方法设置启动命令行。构造一个List接收构造的commands命令行。
    > ContainerLauncherContext接受List对象，其中每个元素都是一个shell命令，YARN会将这些shell命令组成一个shell脚本，用来启动Container。
    
    4. 使用LocalResources对象、环境变量env和命令行commands构造ContainerLaunchContext类的对象amContainer。
    5. 准备好container0的启动上下文后，就可以告诉ApplicationSubmissionContext，刚才准备的ContainerLaunchContext是针对ApplicationMaster的，使用的方法就是`appContext.setAMContainerSpec(amContainer);`。
4. 设置应用程序的资源需求，包括内存和虚拟内核数。
5. 设置应用程序的队列和优先级。（优先级的范围？怎么决定？）
   > YARN支持调度器的概念，包括Capacity调度器等，在yarn-site.xml中指定的，便于管理运行着多个并行作业的YARN集群。一个典型的YARN集群中，会同时运行着多个作业。这意味着需要对使用了超过已分配资源的应用程序进行管理。调度器通过提供YARN应用程序所属的队列以及赋予的优先级进行控制。
6. 通过YarnClient提交应用程序。
   客户端中，提交应用程序的方法会阻塞，知道ResourceManager返回应用程序的状态为ACCEPTED。
    > 如果提交应用程序失败，怎么能重新提交同一个请求？



```
public boolean run() throws IOException, YarnException {

    yarnClient.start();
    ...
    YarnClientApplication app = yarnClient.createApplication();
    GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
    int maxMem = appResponse.getMaximumResourceCapability().getMemory();
    int maxVCores = appResponse.getMaximumResourceCapability().getVirtualCores();
    
    ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
    ApplicationId appId = appContext.getApplicationId();
    appContext.setApplicationName(appName);
    ...		
    Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();
    FileSystem fs = FileSystem.get(conf);
    addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),localResources, null);	
    fs.copyFromLocalFile(false, true, shellSrc, shellDst);
    addToLocalResources(fs, null, shellCommandPath, appId.toString(),localResources, shellCommand);
    addToLocalResources(fs, null, shellArgsPath, appId.toString(),localResources, StringUtils.join(shellArgs, " "));
    ...
    Map<String, String> env = new HashMap<String, String>();
    env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
    ...
    env.put("CLASSPATH", classPathEnv.toString());

    Vector<CharSequence> vargs = new Vector<CharSequence>(30);
    vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
    vargs.add("-Xmx" + amMemory + "m"); 
    ...
    StringBuilder command = new StringBuilder();
    for (CharSequence str : vargs) {
      command.append(str).append(" ");
    } 
    List<String> commands = new ArrayList<String>();
    commands.add(command.toString());		

    ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(localResources, env, commands, null, null, null);

    Resource capability = Resource.newInstance(amMemory, amVCores);
    appContext.setResource(capability);

    appContext.setAMContainerSpec(amContainer);

    Priority pri = Priority.newInstance(amPriority);
    appContext.setPriority(pri);
    appContext.setQueue(amQueue);

    yarnClient.submitApplication(appContext);

    return monitorApplication(appId);

  }
```


###ApplicationMaster分析###

一个新创建的应用程序首先向ResourceManager注册自己，然后以Container形式从ResourceManager请求资源，请求到资源以后跟NodeManager通信，启动Container，监控Container的运行。具体包括：
1. 跟RM通信，向RM注册自己，并取得自己的所在位置（机器，端口等）；
2. 和RM保持heartbeat；
3. 向RM申请执行真正shell的container；
4. 对于申请到的container，设置各种执行资源、环境变量、执行命令等
5. 和RM通讯，了解每一个container的执行情况

与YarnClient类似，ApplicationMaster也有一个main函数，也是以此执行构造函数、初始化和运行的流程。
```
public static void main(String[] args) {
    try {
      ApplicationMaster appMaster = new ApplicationMaster();
      boolean doRun = appMaster.init(args);
      if (!doRun) {
        System.exit(0);
      }
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
####构造函数

构造函数里，先创建一个YarnConfiguration对象，读取YARN环境配置。

####初始化
初始化方法中，分析args参数以及system.env()。
- args参数：container的memory和vcore等，这些是运行shell command的container需要的。
- system.env()的值是client提交时指定的。Container在启动时，会将这些值设置到system系统变量中。

####ApplicationMaster运行

run()方法主要做的工作有：
- 实现一个回调函数处理器监听来自ResourceManager的事件；
- 通过YARN API创建一个对象，封装ApplicationMaster与ResourceManager通信的客户端；
- 实现一个回调函数处理监听来自NodeManager的事件；
- 通过YARN API创建一个对象，封装ApplicationMaster与NodeManager通信的客户端；
- 实现一个类来启动Container。

1. 初始化ResourceManager回调函数处理器，然后用这个处理器创建一个AMRMClientAsync类的对象amRMClient。这个类是YARN所提供的用于ApplicationMaster与ResourceManager异步通信的库。启动Container的类会被ResourceManager回调函数处理器调用。
2. NodeManager回调函数处理器实例化，并创建一个NMClientAsync类的对象nmClientAsync，用于ApplicationMaster和NodeManager之间的异步通信。
3. AMRMClientAsync和NMClientAsync类都是AbstractService的子类，所以创建对象后都需要以YarnConfiguration对象为参数进行初始化，然后调用start方法开启服务。
4. 向ResourceManager注册ApplicationMaster。注册时，需要提供ApplicationMaster的hostname、RPC端口和TrackingURL等基本信息。其中TrackingURL可以由开发者自己定义，YARN的Web界面会对每个应用程序显示这个URL的链接。注册成功后，ApplicationMaster与ResourceManager通信的心跳线程就启动了。
5. 注册成功后会返回一个response。
    - 通过这个response可以dump out ResourceManager看到的集群资源信息。对container请求的资源进行资源检查，超过可用资源就会报错。
    - 通过这个resonanse可以得到ResourceManager之前分配给这个app的container的list，保存在previousAMRunningContainers中。总可用的container数目减之前已经分配给这个app的container数目可以得到当前可以分配的container数（由于硬件错误可能导致ApplicationMaster暂时不可用，因此可能有多次请求，主要是解决这个问题）
6. 建立发往ResourceManager的用于请求container的请求，然后以这个请求为参数，调用addContainerRequest()方法将请求发往ResourceManager。


```
public void run() throws YarnException, IOException {
    ...
    AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
    amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
    amRMClient.init(conf);
    amRMClient.start();

    containerListener = createNMCallbackHandler();
    nmClientAsync = new NMClientAsyncImpl(containerListener);
    nmClientAsync.init(conf);
    nmClientAsync.start();

    appMasterHostname = NetUtils.getHostname();
    RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(
            appMasterHostname, appMasterRpcPort,appMasterTrackingUrl);

    int maxMem = response.getMaximumResourceCapability().getMemory();  //will this useful to hbp??   
    int maxVCores = response.getMaximumResourceCapability().getVirtualCores();
    ...

    List<Container> previousAMRunningContainers = response.getContainersFromPreviousAttempts();
    numAllocatedContainers.addAndGet(previousAMRunningContainers.size());
    int numTotalContainersToRequest = numTotalContainers - previousAMRunningContainers.size();

    for (int i = 0; i < numTotalContainersToRequest; ++i) {
      ContainerRequest containerAsk = setupContainerAskForRM();
      amRMClient.addContainerRequest(containerAsk);
    }
    numRequestedContainers.set(numTotalContainers);
    ...
  }
```

#####RMCallbackHandler回调函数
RMCallbackHandler类实现了AMRMClientAsync.CallbackHandler接口。这些方法都比较直接，当某个事件发生时执行响应的方法。
1. 首先关注onContainersAllocated()方法。收到ResourceManager返回的请求container的响应后，执行该方法。通过一个LaunchContainerRunnable类来启动分配到的container。这个类的构造函数需要两个参数，一个是分配到的container，另外一个是监听NodeManager消息的回调函数处理器。LaunchContainerRunnable实现了Runnable接口，通过这个类来创建线程。
LaunchContainerRunnable类的run()方法把Container的环境，启动命令和本地资源的设置加到ContainerLaunchContext对象中，然后把这个对象提交给Container所在的NodeManager。然后NodeManager就可以启动这个Container了。

```
private class RMCallbackHandler implements AMRMClientAsync.CallbackHandler {
    @Override
    public void onContainersCompleted(List<ContainerStatus> completedContainers) {
      for (ContainerStatus containerStatus : completedContainers) {
        assert (containerStatus.getState() == ContainerState.COMPLETE);
        int exitStatus = containerStatus.getExitStatus();
        if (0 != exitStatus) {
          // container failed
          if (ContainerExitStatus.ABORTED != exitStatus) {
            // shell script failed,counts as completed
            numCompletedContainers.incrementAndGet();
            numFailedContainers.incrementAndGet();
          } else {
            // container was killed by framework, possibly preempted
            // we should re-try as the container was lost for some reason
            numAllocatedContainers.decrementAndGet();
            numRequestedContainers.decrementAndGet();
            // we do not need to release the container as it would be done by the RM
          }
        } else {
          // container completed successfully
          numCompletedContainers.incrementAndGet();
        }
        if(timelineClient != null) {
          publishContainerEndEvent(
              timelineClient, containerStatus, domainId, appSubmitterUgi);
        }
      }
      
      // ask for more containers if any failed
      int askCount = numTotalContainers - numRequestedContainers.get();
      numRequestedContainers.addAndGet(askCount);

      if (askCount > 0) {
        for (int i = 0; i < askCount; ++i) {
          ContainerRequest containerAsk = setupContainerAskForRM();
          amRMClient.addContainerRequest(containerAsk);
        }
      }
      
      if (numCompletedContainers.get() == numTotalContainers) {
        done = true;
      }
    }

    @Override
    public void onContainersAllocated(List<Container> allocatedContainers) {
      for (Container allocatedContainer : allocatedContainers) {
        ...
        LaunchContainerRunnable runnableLaunchContainer = new LaunchContainerRunnable(allocatedContainer, containerListener);
        Thread launchThread = new Thread(runnableLaunchContainer);

        launchThreads.add(launchThread);
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

```
    public void run() {
      Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

      if (!scriptPath.isEmpty()) {
        renameScriptFile(renamedScriptPath);
        URL yarnUrl = null;
        yarnUrl = ConverterUtils.getYarnUrlFromURI(new URI(renamedScriptPath.toString()));
        LocalResource shellRsrc = LocalResource.newInstance(yarnUrl,
          LocalResourceType.FILE, LocalResourceVisibility.APPLICATION,
          shellScriptPathLen, shellScriptPathTimestamp);
        localResources.put(Shell.WINDOWS ? ExecBatScripStringtPath : ExecShellStringPath, shellRsrc);
        shellCommand = Shell.WINDOWS ? windows_command : linux_bash_command;
      }

      Vector<CharSequence> vargs = new Vector<CharSequence>(5);
      vargs.add(shellCommand);
      ...
      StringBuilder command = new StringBuilder();
      for (CharSequence str : vargs) {
        command.append(str).append(" ");
      }

      List<String> commands = new ArrayList<String>();
      commands.add(command.toString());

      ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
        localResources, shellEnv, commands, null, allTokens.duplicate(), null);
      containerListener.addContainer(container.getId(), container);
      nmClientAsync.startContainerAsync(container, ctx);
    }
  }

```

2. onContainersCompleted方法，收到Container执行完毕的消息，检查其执行结果，如果执行失败，则重新发起请求。

#####NMCallbackHandler回调函数
MCallbackHandler回调函数处理NodeManager通知的各种事件。比较直接，修改ApplicationMaster维护的Container执行完成、失败的个数。待Container执行完毕后，可以重启发起请求。失败处理和上面Container执行完毕消息的处理类似，达到了上面问题里所说的loopback效果？？。



> timelineClient?

> DSConstants类，有三个字符串静态变量，提供了一种简单的方法来通过环境变量传递信息给Container。严格来讲，这些环境变量只是一种传递静态信息给ApplicationMaster的方法。ApplicationMaster通过环境变量获取到这些信息后，把它们作为ContainerLaunchContext信息的一部分传递给Container。复杂的应用可能有更多静态数据和动态数据需要传递，除了使用静态变量，还可以使用配置文件，配置文件作为本地资源进行分发，或者作为客户端与ApplicationMaster都能访问到的共享服务比如HDFS。
