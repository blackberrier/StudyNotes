
## 过程
角色：client kdc Service
client对kdc说我想访问某个Service，因为client和service之间需要用session-key认证，所以kdc把这个session-key(client<->service之间需要用到的)用kdc<->client之间的密钥加密后发给client。同时发给client的还有用[kdc<->Service之间的密钥加密后的]client的信息和session-key(client<->service之间需要用到的)。客户端收到两个，只有一个他能解密，就是session-key(client<->service之间需要用到的)，用client<->kdc之间共享密钥解密这个session-key，这样客户端就有了可以与Service通信的密钥。客户端用这个session-key将自己的信息进行加密后，连同他收到的来自kdc的他不能解密的包，发送给Service。所以Service一共收到两个东西，一个是用session-key加密的client的信息，另一个是[用kdc<->service之间共享密钥加密的]client的信息和和session-key(client<->service之间需要用到的)，Service将第二包用其与kdc之间的共享密钥进行解密，得到的session-key再来解密第一个包，得到两个client的信息，进行对比后验证client的合法性。

## 名词解释
KDC：即Key Distribution Center, 密钥分发中心，负责颁发凭证
Kinit：Kerberos认证命令，可以使用密码或者Keytab。
Realm：Kerberos的一个管理域，同一个域中的所有实体共享同一个数据库
Principal：Kerberos主体，即我们通常所说的Kerberos账号(name@realm) ，可以为某个服务或者某个用户所使用
Keytab：以文件的形式呈现，存储了一个或多个Principal的长期的key，用途和密码类似，用于kerberos认证登录；其存在的意义在于让用户不需要明文的存储密码，和程序交互时不需要人为交互来输入密码。

在Kerberos域（realm）中每添加一个服务或者用户就要添加一条principal，每个principal都有一个密码。
- 用户principal的密码用户自己记住
- 服务的principal密码服务自己记录在硬盘上（keytab文件中）；

- 用户principal的命名类似elis/admin@EXAMPLE.COM，形式是用户名／角色／realm域。
- 服务principal的命名类似ftp/station1.example.com@EXAMPLE.COM，形式是服务名／地址（提供者）／realm域；

## Kerberos用户登录过程：
a> 用户提供用户名密码；
b> login程序（/usr/bin/login）把用户名转成princical名称，给KDC发送登录请求；
c> KDC生成一个用于对称加密的session key，也叫TGT（Ticket-granting Ticket） key。KDC自己留一份，然后复制一份用用户的密码（其实是princical的密码）加密后传给用户（login程序）；
d> login程序收到之后用刚才用户输入的密码解密，获得了TGT key。
e> 认证过程完成，之后的通信使用TGT key加密；

## Kerberos用户获得TGT之后登录服务过程：
a> 用户向KDC请求服务授权（用户和KDC的沟通是用TGT加密的）；
b> KDC在验证princical通过后生成一个新的session key。将此key用用户的密码加密一次，再用服务的密码加密一次，把两次加密后的内容都发给用户；
c> 用户把第一份自己能解开的用TGT解密后得到session key（用于和服务沟通）。然后用此session key加密一个时间戳（作为一个对服务的验证）和自己解不开的那份用服务的密码加密的session key一并发给服务；
d> 服务用存储在keytab文件中的密码解密，也得到session key（这证明这玩意儿来自拥有自己密码的KDC）。然后解开用户加密的时间戳（这证明用户确实也拿到session key了，意即通过了KDC的允许）。这里面的时间戳用于防止重放攻击，所以所有的机器都必需使用安全的类似NTP的机制把时间同步好；
e> 登录服务过程完成，整个过程中大家都不知道对方的密码，密码也没有在网络中传输过。之后用户与服务的通信使用session key来加密；

## 参考资料
http://www.360doc.com/content/15/0803/10/13047933_489182493.shtml
https://www.cnblogs.com/luxianghao/p/5269739.html
http://blog.csdn.net/wulantian/article/details/42418231
