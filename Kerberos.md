
## Kerberos

### 过程
角色：client kdc Service  
client对kdc说我想访问某个Service，因为client和service之间需要用session-key认证，所以kdc把这个session-key(client<->service之间需要用到的)用kdc<->client之间的密钥加密后发给client。同时发给client的还有用[kdc<->Service之间的密钥加密后的]client的信息和session-key(client<->service之间需要用到的)。客户端收到两个，只有一个他能解密，就是session-key(client<->service之间需要用到的)，用client<->kdc之间共享密钥解密这个session-key，这样客户端就有了可以与Service通信的密钥。客户端用这个session-key将自己的信息进行加密后，连同他收到的来自kdc的他不能解密的包，发送给Service。所以Service一共收到两个东西，一个是用session-key加密的client的信息，另一个是[用kdc<->service之间共享密钥加密的]client的信息和和session-key(client<->service之间需要用到的)，Service将第二包用其与kdc之间的共享密钥进行解密，得到的session-key再来解密第一个包，得到两个client的信息，进行对比后验证client的合法性。

### 名词解释
1. KDC：即Key Distribution Center, 密钥分发中心，负责颁发凭证
2. Kinit：Kerberos认证命令，可以使用密码或者Keytab
3. Realm：Kerberos的一个管理域，同一个域中的所有实体共享同一个数据库
4. Principal：Kerberos主体，即我们通常所说的Kerberos账号(name@realm) ，可以为某个服务或者某个用户所使用【中文理解为主用户？】
5. Keytab：以文件的形式呈现，存储了一个或多个Principal的长期的key，用途和密码类似，用于kerberos认证登录；其存在的意义在于让用户不需要明文的存储密码，和程序交互时不需要人为交互来输入密码。

Principal是一种身份的ID，在这种身份上可以唯一地标识一个user。例如，同一个person，他可能有不同的身份，公民，学生，读者等。那么他的公民ID，学生ID以及读者ID都分别是一种Principal。不同的认证机制可以定义不同的Principal来标识user。  
Subject是一个container，包含一组关于某个user的安全相关信息。Subject最主要的内容是Principals和Credentials，Principals比如上面提到的公民ID，学生ID。Subject可以在不同的认证机制中传递，每个认证机制都可以将自己定义的Principal加入该Subject。而Credential是对Principal的补充

在Kerberos域（realm）中每添加一个服务或者用户就要添加一条principal，每个principal都有一个密码。
- 用户principal的密码用户自己记住
- 服务的principal密码服务自己记录在硬盘上（keytab文件中）；

- 用户principal的命名类似elis/admin@EXAMPLE.COM，形式是用户名／角色／realm域。
- 服务principal的命名类似ftp/station1.example.com@EXAMPLE.COM，形式是服务名／地址（提供者）／realm域；

### Kerberos用户登录过程：
- a> 用户提供用户名密码；
- b> login程序（/usr/bin/login）把用户名转成princical名称，给KDC发送登录请求；
- c> KDC生成一个用于对称加密的session key，也叫TGT（Ticket-granting Ticket） key。KDC自己留一份，然后复制一份用用户的密码（其实是princical的密码）加密后传给用户（login程序）；
- d> login程序收到之后用刚才用户输入的密码解密，获得了TGT key。
- e> 认证过程完成，之后的通信使用TGT key加密；

### Kerberos用户获得TGT之后登录服务过程：
- a> 用户向KDC请求服务授权（用户和KDC的沟通是用TGT加密的）；
- b> KDC在验证princical通过后生成一个新的session key。将此key用用户的密码加密一次，再用服务的密码加密一次，把两次加密后的内容都发给用户；
- c> 用户把第一份自己能解开的用TGT解密后得到session key（用于和服务沟通）。然后用此session key加密一个时间戳（作为一个对服务的验证）和自己解不开的那份用服务的密码加密的session key一并发给服务；
- d> 服务用存储在keytab文件中的密码解密，也得到session key（这证明这玩意儿来自拥有自己密码的KDC）。然后解开用户加密的时间戳（这证明用户确实也拿到session key了，意即通过了KDC的允许）。这里面的时间戳用于防止重放攻击，所以所有的机器都必需使用安全的类似NTP的机制把时间同步好；
- e> 登录服务过程完成，整个过程中大家都不知道对方的密码，密码也没有在网络中传输过。之后用户与服务的通信使用session key来加密；

## JAAS
JAAS是一个Java程序专用的认证和授权框架。

### JAAS使用方式
要使用JAAS，除了程序之外，还需要准备额外的配置文件。示例中有两个配置文件，一个.policy文件，这是一个Policy File，另一个文件是.config文件，它就是 JAAS必须用到的配置文件。另外，要使用JAAS，还必须以-Djava.security.auth.login.config=... 这个参数运行java命令。
示例程序运行的时候，会跳出一个对话框让我们输入keystore中的条目的别名以及keystore的密码。（hbp-123456）。
我们输入的别名会被用来认证我们的身份，然后决定我们是否有权访问相应的资源。比如，当我们输入的是hbp，则可以顺利运行程序，如果输入的是别的，就无法运行程序。

### JAAS的架构
1. 首先，JAAS认证的核心是LoginModel，它代表了认证的方式。认证的方式有很多种，但JDK自带的几种实现不一定能满足我们的需求。所以，如果我们真的要在产品中使用JAAS的话，一定要学会实现自己的LoginModel。JDK自带了JndiLoginModel、UnixLoginModel、KeyStoreLoginModel、Krb5LoginModel、LdapLoginModel等实现，看名字就能猜到它们使用的认证方式。
2. JAAS和用户交互的核心是CallbackHandler，它代表了和用户交互的方式。DialogCallbackHandler就是使用对话框和用户交互，而TextCallbackHandler就是使用标准输入输出和用户交互。
3. LoginModel和CallbackHandler之间沟通的桥梁是Callback。当 LoginModel 需要从用户获取 username 时，就向 CallbackHandler 发一个 NameCallback， CallbackHandler 和用户交互，获得 username 后再发回去。当 LoginModel 需要密码的时候就向 CallbackHandler 发一个 PasswordCallback，CallbackHandler 和用户交互，获得 password 后再发回去。当 LoginModel 需要给用户发送一些消息时，就向 CallbackHandler 发一个 TextOutputCallback，让 CallbackHandler 将消息显示给用户。以此类推，总之，和用户进行交互的永远是 CallbackHandler。

## Hadoop方式
两种方式 simple和Kerberos
1. Simple Auth机制下，Client端提取本地OS login username发送给Server，Server毫无保留地接受username，并以此身份运行Job。实际上，Hadoop本身没有做任何认证。这是不安全的，例如，恶意用户在自己的机器上伪造一个其他人的username(比如hdfs)，然后向JobTracker发送Job，JobTracker将以hdfs身份运行Job，用户Job将拥有一切hdfs所有的权限。
http://qq85609655.iteye.com/blog/2180717




## 参考资料
http://www.360doc.com/content/15/0803/10/13047933_489182493.shtml
https://www.cnblogs.com/luxianghao/p/5269739.html
http://blog.csdn.net/wulantian/article/details/42418231



信息加密传输是保证网络安全的基本手段，不管是对称加密还是非对称加密，都需要在通信的两端维护密钥。 如果密钥一直不变的话，根据大量截获的加密消息，迟早可能将密钥破解出来；而如果密钥定时更新的话，总可以通过减小密钥更新时间，来保证密钥破解之前更新掉。
用定时更新的短期密钥来加密消息，就是Kerberos方案的出发点。下面详细介绍Kerberos协议涉及的概念和交互消息。
## 1. 基本概念  
长期密钥：Long-term Key，Kerberos协议中称作主密钥(Master Key，简写为MK)  
短期密钥：Short-term Key，Kerberos协议中称作会话密钥(Session Key，简写为SK)  
票据：Ticket，可以认为是加密过的会话密钥，包括TGT(Ticket Granting Ticket)和SST(Service Session Ticket)，分别用来访问票据授予服务(Ticket Granting Service)和具体业务服务(Service)  
主体：Principal，使用Kerberos协议进行认证的主体，比如，配置了Kerberos认证的hdfs客户端、NameNode和DataNode都是Kerberos协议的Principal
Kerberos服务  
kadmind：管理KDB数据库中Principal，KDB存储着TGS和所有Principal的Master Key  
kdc：会话密钥的分发中心(Key Distributing Center)，包括认证(AS, Authentication Service)和票据授予(TGS, Ticket Granting Service)两种服务  
AS: 对请求发起端的身份进行认证，生成用于发起端和TGS的会话密钥，返回加密的会话密钥和票据TGT  
TGS: 验证请求发起端的TGT，并生成用于发起端和业务服务端的会话密钥，返回加密的会话密钥和票据SST  
## 2. 协议分析  
A P_a B P_b ts: 主体A、主体A的信息、主体B、主体B的信息、时间戳timestamp  
MK_a MK_b MK_k: 分别表示主体A、主体B和KDC的主密钥(Master Key)  
SK_ak SK_ab: 分别表示主体A与KDC之间的会话密钥、主体A与主体B之间的会话密钥  
TGT_ak: 主体A用来访问KDC TGS服务用的票据，可理解为用KDC主密钥MK_k加密的会话秘密SK_ak  
SST_ab: 主体A用来访问主体B服务用的票据，可理解为用主体B的主密钥MK_b加密的会话秘密SK_ab  

前述，使用会话密钥加密消息是Kerberos协议的出发点，接下来的问题就是如何传输会话密钥？答案是，用主密钥对会话密钥进行加密传输。下面根据 Kerberos协议的时序图，介绍一下协议过程中的交互的消息。  
### 2.1 KDC认证  
s1. 请求发起端A将主体信息(P_a)和用MK_a加密的认证信息(时间戳ts)，发给KDC的AS服务。  
s2. AS会从KDB中获取P_a的主密钥MK_a，解密出时间戳，校验时间戳是否合法，且时间偏差是否超出阈值(默认10分钟)  
若校验通过，AS向TGS申请P_a访问TGS的会话密钥SK_ak，并将SK_ak分别用MK_a和MK_k加密(MK_k加密的SK_ak就是票据TGT_ak)，返回A  
若校验不过，则返回异常  
因为Kerberos认证过程中需要校验时间戳，所以参与认证的各个主体和Kerberos服务器节点要进行时钟对其。  
### 2.2 请求业务票据  
s3. A收到MK_a(SK_ak)和TGT_ak后，解密出SK_ak来加密ts,P_a,P_b，与TGT_ak一起发给KDC的TGS  
s4. TGS主密钥MK_k从TGT_ak中解密出SK_ak，再用SK_ak解密出ts,P_a,P_b并校验时间戳ts和P_a  
若校验通过，TGS生成主体A和主体B的会话密钥SK_ab，并从KDB中获取MK_a和MK_b分别加密SK_ab(MK_b加密的SK_ab就是票据SST_ab)，返回给A  
若校验不过，则返回异常  
相比AS仅校验时间戳ts，TGS会同时检验时间戳ts和请求发起端的主体信息P_a。另外，步骤s4中TGS没有将加密的SK_ab分别发送给主体A和主体B，而是全都发给了A；主要是为避免网络延时异常，例如，A用SK_ab加密的请求发送到B时，由于传输延时，B可能还没收到TGS发送的SK_ab；该协议的处理方法就是会话密钥一开始都是由发起端持有，步骤s2 AS将TGT_ak发给P_a也是同样的思路。  
### 2.3 执行业务请求  
s5. A收到MK_a(SK_ab)和SST_ab后，解密出SK_ak来加密ts,P_a,REQ，与SST_ab一起发给B  
s6. B主密钥MK_b从SST_ab中解密出SK_ab，再用SK_ab解密ts,P_a,REQ，并校验时间戳ts和P_a  
若校验通过，则B处理业务请求REQ，并将响应RSP用SK_ab加密后返回给A  
若校验不过，则返回异常  
Kerberos协议步骤分析完之后，可以看到会话密钥和票据都是由主密钥加密的，那么，主密钥怎么传输呢？Kerberos如何安全将主密钥分发给各自对应的主体呢？实际上，Kerberos不处理主密钥传输的问题，这个需要使用者(管理员)来保证主密钥安全地传输和部署。管理员可以从KDB中将一个和多个主体的主密钥及相关信息导出为固定的格式，通常命名为keytab。  
Kerberos协议有时被称为三方认证协议，其时也可认为是通过第三方来认证。  
通常情况下，主体A请求主体B的服务时，只要A向B发送认证信息，认证过了的话，即可响应A的请求，否则拒绝。  
Kerberos协议中，如果主体A要请求主体B的服务，A不是向B认证，而向KDC发送认证信息，A在KDC那里认证过了的话，会获得一个访问B的票据SST_ab，A向B请求服务时附带这个票据，即B验证这个票据后，即可响应请求。  
## 3. 总结  
Kerberos协议的基本思路：  
短期密钥(Session Key)加密传输业务数据  
长期密钥(Master Key)加密传输短期密钥  

