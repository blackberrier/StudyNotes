## 什么是oozie
Oozie 是一个工作流调度系统用来管理Hadoop任务  
通俗解释：
Oozie的作用是，当有一系列Map/Reduce作业，它们之间彼此是相互依赖的关系。正常的思路是写shell脚本来完成这些事情，但是当任务多，依赖关系复杂时，写脚本是费时费力的，而且复用性也差。Oozie应运而生。  
工作流调度：工作流程的编排，调度：安排事件的触发执行(时间触发,事件触发)  
1. Cloudeara共享给Apache的开源项目，提供对hadoop任务（MapReduce、Spark、Pig、Hive等）的调度。  
2. oozie需要部署到tomcat或其他servlet容器中，提供对外的接口；需要使用关系型数据库**存储调度信息**。  
3. 通过xml格式的文件定义工作流。可以有不同的功能节点，比如分支、合并和汇合等。  
4. oozie定义了控制流程节点和动作节点：**控制节点**定义了流程的开始和结束，还提供了控制工作流执行路径的机制（decision，fork，join等）；**动作节点**能够触发一个计算任务（Computation Task）或者处理任务（Processing Task）执行的节点。  

## oozie的各个版本
Oozie v1 is a server based **Workflow Engine** specialized in running workflow jobs with actions that execute Hadoop Map/Reduce and Pig jobs.
![v1](https://i.imgur.com/OXo6Lby.png)  
Oozie v2 is a server based **Coordinator Engine** specialized in running workflows based on time and data triggers(时间触发 数据触发). It can continuously run workflows based on time (e.g. run it every hour), and data availability (e.g. wait for my input data to exist before running my workflow).
![v2](https://i.imgur.com/Ot6FwIQ.png)  
Oozie v3 is a server based **Bundle Engine** that provides a higher-level oozie abstraction that will batch a set of coordinator applications. The user will be able to start/stop/suspend/resume/rerun a set coordinator jobs in the bundle level resulting a better and easy operational control.
![v3](https://i.imgur.com/fariJ0F.png)  
## 整体框架
![struct](https://i.imgur.com/KdSyKss.png)  
1. Oozie通过Tomcat对外提供了JAVA API、REST API、CLI、Web接口（hue），产生的数据存储在Oozie object database上  
2. Oozie的三层结构  
3. Oozie的Coordinator Engine协调引擎能够监控基于Time-based triggers和HDFS上的Data-based triggers；每一个Oozie Job都是一个只有Map Task的 MapReduce程序。

## 控制流节点
控制流控制工作流的开始和结束，以及工作流Job的执行路径的节点，它定义了流程的开始（start节点）和结束（end节点或kill节点），同时提供了一种控制流程执行路径的机制（decision节点、fork节点、join节点）
1. start节点
```
<workflow-app name="[WF-DEF-NAME]" xlmns="uri:oozie:workflow:0.1">
    ...
    <start to="[NODE-NAME]" />
    ...
</workflow-app>
```  
2. end节点 kill节点
写法类似start节点
3. decision节点
decision节点通过预定义一组条件，当工作流Job执行到该节点时，会根据其中的条件进行判断选择，满足条件的路径将被执行  
```
<workflow-app name="[WF-DEF-NAME]" xmlns="uri:oozie:workflow:0.1">
    ...
    <decision name="[NODE-NAME]">
        <switch>
            <case to="[NODE_NAME]">[PREDICATE]</case>
            ...
            <case to="[NODE_NAME]">[PREDICATE]</case>
            <default to="[NODE_NAME]"/>
        </switch>
    </decision>
    ...
</workflow-app>
```
4. fork节点和join节点
```
<workflow-app name="[WF-DEF-NAME]" xmlns="uri:oozie:workflow:0.1">
    ...
    <fork name="[FORK-NODE-NAME]">
        <path start="[NODE-NAME_1]" />
        ...
        <path start="[NODE-NAME_N]" />
    </fork>
    ...
    <join name="[JOIN-NODE-NAME]" to="[NODE-NAME]" />
    ...
</workflow-app>
```
fork和join必须一起使用。  
fork元素下面会有多个path元素，指定了可以并发执行的多个执行路径。fork中多个并发执行路径会在join节点的位置会合：只有所有的路径都到达后，才会继续执行join节点to属性指向的节点。 
也就是说，从NODE-NAME_1直到NODE-NAME_N，如果执行成功后，ok元素应该指向一个相同的join作业。  
```
<workflow-app name="sample-wf" xmlns="uri:oozie:workflow:0.1">
    ...
    <fork name="forking">
        <path start="firstparalleljob"/>
        <path start="secondparalleljob"/>
    </fork>
    <action name="firstparallejob">
        <map-reduce>
            <job-tracker>foo:8021</job-tracker>
            <name-node>bar:8020</name-node>
            <job-xml>job1.xml</job-xml>
        </map-reduce>
        <!-- 若firstparallejob任务执行成功，则指向任务"joining"-->
        <ok to="joining"/>
        <error to="kill"/>
    </action>
    <action name="secondparalleljob">
        <map-reduce>
            <job-tracker>foo:8021</job-tracker>
            <name-node>bar:8020</name-node>
            <job-xml>job2.xml</job-xml>
        </map-reduce>
        <!-- 若secondparalleljob任务执行成功，则指向任务"joining"-->
        <ok to="joining"/>
        <error to="kill"/>
    </action>
    <!-- 当firstparallejob和secondparalleljob都运行完毕，汇集到join元素的时候，join元素执行to指向的nextaction操作-->
    <join name="joining" to="nextaction"/>
    ...
</workflow-app>
```

## action节点  
### 基本特性
1. 远程执行  
Oozie可能部署在一个单独的服务器上，而工作流Job是在Hadoop集群的节点上执行的。  
即使Oozie在Hadoop集群的某个节点上，它也是处于与Hadoop进行独立无关的JVM示例之中（因为Oozie部署在Servlet容器当中）。  
2. 异步性
Oozie启动一个工作流Job，这个工作流Job便开始执行。  
Oozie可以通过两种方式来探测工作流Job的执行情况：一种是基于**回调机制**（call backs），对**每个任务的执行都设置一个唯一的URL**，如果任务执行结束或者执行失败，会通过回调这个URL通知Oozie已经完成；另一种就是轮询（polling）。如果由于网络故障回调机制失败，也会使用轮询的方式去获取任务的状态。 
3. 可恢复性  
如果一个动作节点执行失败，Oozie提供了一些恢复执行的策略，这个要根据失败的特点来进行：如果是状态转移过程中失败，Oozie会根据指定的重试时间间隔去重新执行；如果不是转移性质的失败，则只能通过手工干预来进行恢复；如果重试恢复执行都没有解决问题，则最终会跳转到error节点。  
当成功执行时会执行ok定义的to name,失败时执行error定义的to name,当为error时，必须提供error-code和error-message 信息给 Oozie。可以通过使用decision node对error-code进行switch-case从而实现不同错误代码不同的处理逻辑。

### 典型Oozie内置支持的动作节点类型
#### Spark动作
```
<workflow-app name="[WF-DEF-NAME]" xmlns="uri:oozie:workflow:0.3">
    ...
    <action name="[NODE-NAME]">
        <spark xmlns="uri:oozie:spark-action:0.1">
            <job-tracker>[JOB-TRACKER]</job-tracker>
            <name-node>[NAME-NODE]</name-node>
            <prepare>
               <delete path="[PATH]"/>
               ...
               <mkdir path="[PATH]"/>
               ...
            </prepare>
            <job-xml>[SPARK SETTINGS FILE]</job-xml>
            <configuration>
                <property>
                    <name>[PROPERTY-NAME]</name>
                    <value>[PROPERTY-VALUE]</value>
                </property>
                ...
            </configuration>
            <master>[SPARK MASTER URL]</master>
            <mode>[SPARK MODE]</mode>
            <name>[SPARK JOB NAME]</name>
            <class>[SPARK MAIN CLASS]</class>
            <jar>[SPARK DEPENDENCIES JAR / PYTHON FILE]</jar>
            <spark-opts>[SPARK-OPTIONS]</spark-opts>
            <arg>[ARG-VALUE]</arg>
                ...
            <arg>[ARG-VALUE]</arg>
            ...
        </spark>
        <ok to="[NODE-NAME]"/>
        <error to="[NODE-NAME]"/>
    </action>
    ...
</workflow-app>
```
参数说明：  
- master: 为spark作业指定调度器：yarn-master yarn-cluster mesos local等。  
  示例：spark://host:port, mesos://host:port, yarn-cluster, yarn-master, or local  
- mode: 为spark作业指定driver运行位置：client或者cluster。等效于在master中，可以指定： master=yarn-client is equivalent to master=yarn, mode=client。  
此外，一个本地的master总是以client模式运行？？  
基于不同的master和mode，Spark作业运行情况如下：  
● local mode: everything runs here in the Launcher Job.  
● yarn-client mode: the driver runs here in the Launcher Job and the executor in Yarn.  
● yarn-cluster mode: the driver and executor run in Yarn.  

## oozie环境配置与使用
### 安装
1. 安装包  
可以在官网上获取oozie的源码并自行编译，也可以使用cloudera编译好的版本。  
自行编译时候需要注意需要修改源码的pom中关于jdk版本、hadoop版本等的配置项。  
2. 修改hadoop和oozie的配置文件
hadoop配置core-site.xml中添加  
```
<property>
  <name>hadoop.proxyuser.root.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.root.groups</name>
  <value>*</value>
</property>
```
修改oozie配置文件./conf/oozie-site.xml，根据实际情况修改，另外需要添加hadop的配置文件路径  
3. 创建libext目录，添加依赖  
将hadoop目录的share路径下的jar包添加到libext中，另外需要将hadoop lib中与tomcat lib中冲突的jar去掉  
下载mysq连接器 mysql-connector-java-<版本>.zip放到libext中  
下载ext-2.2.zip，放到libext中  
4. 将ext2.2.0.zip、hadoop的相关jar包、以及mysql-connector-java-<版本>.jar、htrace-core-<版本>.jar、avro-<版本>.jar打进oozie.war包里  
5. 安装数据库mysql，创建名为oozie的数据库，执行`bin/ooziedb.sh create -sqlfile oozie.sql -run`   
6. 安装oozie-sharelib，将sharelib中的包上传至dhfs，hdfs目标路径需要与oozie-site.xml中的oozie.service.WorkflowAppService.system.libpath的值保持一致  
7. 启动oozie `bin/oozie-start.sh`  
通过`oozie admin -oozie http://localhost:11000/oozie -status`查看状态  
8. 通过客户端提交  
`oozie job -oozie http://<hostname>:11000/oozie -config ./examples/apps/sqoop/job.properties -run `  
