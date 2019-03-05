

# ZooKeeper #


## ZooKeeper命令操作
ZooKeeper支持某些四字字母与其交互，可以查询zk当前状态和相关信息。用户可以通过**telnet**和**nc**提交相应命令。

    echo ruok|nc localhost 2181

### 四字命令 - 功能描述

 - conf - 输出相关服务配置的详细信息
 - cons - 列出所有连接到服务器的客户端的完全的连接或会话的详细信息，包括发送和接收的包的数量，会话id，操作延迟等
 - dump - 列出未经处理的会话和临时节点
 - envi - 输出关于服务环境的详细信息
 - reqs - 列出未经处理的请求
 - ruok - 测试服务是否处于正确状态。是，则返回imok
 - stat - 输出关于性能和连接的客户端的列表
 - wchs - 列出服务器watch的详细信息
 - wchc - 通过**session**列出服务器watch的详细信息
 - wchp - 通过**路径**列出服务器watch的详细信息

## ZooKeeper的命令行操作

 - 查看当前Zookeeper中所包含的内容：`ls /`
 - 创建一个新的Znode节点以及和它相关字符：`create /zk myData`
 - 确认创建的Znode是否包含我们创建的字符串：`get /zk`
 - 对zk所关联的字符串进行设置：`set /zk jiang1234`
 - 删除：`delete /zk`


## 客户端与ZooKeeper的交互步骤

1. 连接到ZooKeeper：ZooKeeper分配客户端的会话ID。
2. 客户端定期发送心跳到服务器。超时，客户端需要重新连接。
3. 获得/设置。只要znodes会话ID是活动的。
4. 断开。任务完成后，如果客户端长时间处于非活动状态，自动断开。

## Java API

ZooKeeper API的中心部分**org.apache.zookeeper**包，包含**Zookeeper类**，这个类是Zookeeper客户端的主要类文件。

如果要使用Zookeeper服务，应用程序首先必须**创建一个Zookeeper实例**。一旦客户端和Zookeeper服务建立起了连接，Zookeeper系统将会给这次连接会话分配一个ID值，并且客户端将会周期性的向服务器端发送心跳来维持会话连接。只要连接有效，客户端就可以使用Zookeeper API来做相应处理了。

主要方法有

* connect − 连接到ZooKeeper集群。
* create − 创建一个znode
* exists − 检查znode是否存在及其信息
* getData − 从特定的znode获取数据
* setData − 给特定znode设置数据
* getChildren − 得到特定znode的所有可用子节点
* delete − 删除特定znode
* close − 关闭连接

### 连接到Zookeeper集群

ZooKeeper类通过**构造函数**提供了**连接功能**。

    ZooKeeper(String connectionString, int sessionTimeout, Watcher watcher)

- connectionString − ZooKeeper集群。
- sessionTimeout − 以毫秒为单位会话超时。
- watcher − 一个执行对象“观察者”的接口。ZooKeeper集群返回通过监控器对象的连接状态。

Watcher接口是由Zookeeper定义的Java API。Zookeeper通过这个接口来与其container通信。它只支持一个方法`process()`。Zookeeper使用它通信主线程感兴趣的通用事件,如Zookeeper连接的状态或Zookeeper会话。

当一个ZooKeeper的实例被创建时，会启动一个线程连接到ZooKeeper服务。对构造函数的调用是立即返回的，因此在使用新建的ZooKeeper对象之前一定要等待其与ZooKeeper服务之间的连接建立成功。可以使用Java的**CountDownLatch类**来阻止使用新建的ZooKeeper对象，直到这个ZooKeeper对象已经准备就绪。这就是Watcher类的用途，在它的接口中只有一个方法 `public void process(WatcherEvent event);`
**客户端与ZooKeeper建立连接后，Watcher的process()方法会被调用**，参数是一个表示该连接的事件。

#### zookeeper观察机制:
服务端只存储事件的信息,客户端存储事件的信息和Watcher的执行逻辑。
ZooKeeper客户端是线程安全的，每一个应用只需要实例化一个ZooKeeper客户端即可，同一个ZooKeeper客户端实例可以在不同的线程中使用。ZooKeeper客户端会将这个Watcher对应Path路径存储在ZKWatchManager中,同时通知ZooKeeper服务器记录该Client对应的Session中的Path下注册的事件类型。当ZooKeeper服务器发生了指定的事件后,ZooKeeper服务器将通知ZooKeeper客户端哪个节点下发生事件类型，ZooKeeper客户端再从ZKWatchManager中找到相应Path，取出相应watcher引用执行其回调函数process
ZooKeeper 允许客户端向服务端注册一个 Watcher 监听，当服务端的一些指定事件触发了这个 Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。
ZooKeeper 的 Watcher 机制主要包括客户端线程、客户端 WatchManager 和 ZooKeeper 服务器三部分。在具体工作流程上，简单地讲，客户端在向 ZooKeeper 服务器注册 Watcher 的同时，会将 Watcher 对象存储在客户端的 WatchManager 中。当 ZooKeeper 服务器端触发 Watcher 事件后，会向客户端发送通知，客户端线程从 WatchManager 中取出对应的 Watcher 对象来执行回调逻辑。

### 创建一个Znode

ZooKeeper类提供了一个方法来在ZooKeeper集群创建一个新的znode

    create(String path, byte[] data, List<ACL> acl, CreateMode createMode)

- path − Znode的路径。例如/myapp1,/myapp1/mydata1,myapp2/mydata1/myanothersubdata
- data − 在指定的znode路径存储数据
- acl − 要创建节点的访问控制列表。ZooKeeperAPI提供了一个静态接口ZooDefs.Ids得到一些基本的ACL列表。
- createMode − 节点的类型，枚举类型。

### 检查一个Znode是否存在

ZooKeeper类提供了`exists`方法来检查znode是否存在。如果指定的znode存在，返回一个znode元数据。

    exists(String path, boolean watcher)

- path − Znode路径
- watcher − 布尔值，指定是否监视指定的znode

### getData 方法

ZooKeeper类提供`getData`方法来获取连接在指定znode及其状态的数据。

    getData(String path, Watcher watcher, Stat stat)

- path − Znode路径。
- watcher − Watcher类型的回调函数。ZooKeeper集群将通知通过观察者回调时指定的节点改变的数据。这是一次性的通知。
- stat − 返回znode元数据。

### setData方法

ZooKeeper类提供`SetData`方法来修改附着在指定znode的数据。

    setData(String path, byte[] data, int version)

- path − Znode路径
- data − 数据存储在一个指定的znode路径。
- version − 当前znode的版本。
