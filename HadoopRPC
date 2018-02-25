## RPC应用实例
### 1. 定义RPC协议
如下所示，我们定义一个IProxyProtocol 通信接口，声明了一个Add()方法。
<code>
public interface IProxyProtocol extends VersionedProtocol {
    static final long VERSION = 23234L; //版本号，默认情况下，不同版本号的RPC Client和Server之间不能相互通信
    int Add(int number1,int number2);
}
</code>
需要注意的是：
（1）Hadoop中所有自定义RPC接口都需要继承VersionedProtocol接口，它描述了协议的版本信息。
（2）默认情况下，不同版本号的RPC Client和Server之间不能相互通信，因此客户端和服务端通过版本号标识。

### 2. 实现RPC协议
Hadoop RPC协议通常是一个Java接口，用户需要实现该接口。对IProxyProtocol接口进行简单的实现如下所示：
<code>
public class MyProxy implements IProxyProtocol {
    public int Add(int number1,int number2) {
        System.out.println("我被调用了!");
        int result = number1+number2;
        return result;
    }

    public long getProtocolVersion(String protocol, long clientVersion)
            throws IOException {
        System.out.println("MyProxy.ProtocolVersion=" + IProxyProtocol.VERSION);
        // 注意：这里返回的版本号与客户端提供的版本号需保持一致
        return IProxyProtocol.VERSION;
    }
}
</code>
这里实现的Add方法很简单，就是一个加法操作。为了查看效果，这里通过控制台输出一句：“我被调用了！”

### 3. 构造RPC Server并启动服务
这里通过RPC的静态方法getServer来获得Server对象，如下代码所示：
<code>
public class MyServer {
    public static int PORT = 5432;
    public static String IPAddress = "127.0.0.1";

    public static void main(String[] args) throws Exception {
        MyProxy proxy = new MyProxy();
        final Server server = RPC.getServer(proxy, IPAddress, PORT, new Configuration());
        server.start();
    }
}
</code>
这段代码的核心在于第5行的RPC.getServer方法，该方法有四个参数，第一个参数是被调用的java对象，第二个参数是服务器的地址，第三个参数是服务器的端口 。获得服务器对象后，启动服务器。这样，服务器就在指定端口监听客户端的请求。到此为止，服务器就处于监听状态，不停地等待客户端请求到达。

### 4. 构造RPC Client并发出请求
这里使用静态方法getProxy或waitForProxy构造客户端代理对象，直接通过代理对象调用远程端的方法，具体如下所示：
<code>
public class MyClient {

    public static void main(String[] args) {
        InetSocketAddress inetSocketAddress = new InetSocketAddress(
                MyServer.IPAddress, MyServer.PORT);

        try {
            // 注意：这里传入的版本号需要与代理保持一致
            IProxyProtocol proxy = (IProxyProtocol) RPC.waitForProxy(
                    IProxyProtocol.class, IProxyProtocol.VERSION, inetSocketAddress,
                    new Configuration());
            int result = proxy.Add(10, 25);
            System.out.println("10+25=" + result);

            RPC.stopProxy(proxy);
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }

}
</code>
以上代码中核心在于RPC.waitForProxy()，该方法有四个参数，第一个参数是被调用的接口类，第二个是客户端版本号，第三个是服务端地址。返回的代理对象，就是服务端对象的代理，内部就是使用java.lang.Proxy实现的。

经过以上四步，我们便利用Hadoop RPC搭建了一个非常高效的客户机–服务器网络模型。
