## 概念
- 什么是Servlet？
    Servlet其实就是一个遵循Servlet开发的java类。Servlet是由服务器调用的，运行在服务器端。
- 为什么要用到Servlet？
    我们编写java程序想要在网上实现 聊天、发帖、这样一些的交互功能，普通的java技术是非常难完成的。sun公司就提供了Servlet这种技术供我们使用。
- 什么是JSP
    JSP全名为Java Server Pages，java服务器页面。JSP是一种基于文本的程序，其特点就是HTML和Java代码共同存在
- 为什么需要JSP
    JSP是为了简化Servlet的工作出现的替代品，Servlet输出HTML非常困难，JSP就是替代Servlet输出HTML的。

## 上古时期
前端写好html静态页面，丢给后端，后端在servlet中调用service拿到数据，根据情况将数据和html页面拼接，然后输出到前端。
!(https://pic2.zhimg.com/v2-fc8bb977784e7b703bfd157192825899_r.jpg)
主要目的:在最终输出的html的代码中嵌入后台数据
两种思路：把html语句拿出来在Servlet里拼接好再输出（前述方式）
        直接在html语句中写入动态数据（不是HTML文件，必须是JSP之类的动态模板文件中的HTML语句）
## JSP
Java Server Page “运行在服务器端的页面”
JSP = HTML + Java片段（各种标签本质上还是Java片段）
是一种动态网页开发技术。它使用JSP标签在HTML网页中插入Java代码。标签通常以<%开头以%>结束。
浏览器请求JSP时，服务器内部会经历一次动态资源（JSP）到静态资源（HTML）的转化，服务器会把JSP中的HTML片段和数据拼接成静态资源响应给浏览器

WEB容器接收到以.jsp为扩展名的URL的访问请求时，它将把该请求交给JSP引擎去处理。Tomcat中的JSP引擎就是一个Servlet程序，它负责解释和执行JSP页面。每个JSP 页面在第一次被访问时，JSP引擎将它翻译成一个Servlet源程序，接着再把这个Servlet源程序编译成Servlet的class类文件，然后再由WEB容器像调用普通Servlet程序一样的方式来装载和解释执行这个由JSP页面翻译成的Servlet程序。
JSP请求---->web服务器---->JSP引擎---->Servlet引擎---->Web服务器---->客户端
具体实现流程：
1. 首先浏览器发送HTTP请求到服务器端
2. Web服务器会根据请求的URL或.jsp识别到该请求是一个JSP网页请求，并将该请求转发给JSP引擎（JSP引擎其实就是来处理JSP网页的）
3. JSP引擎会根据请求加载对应的JSP文件，并将该JSP文件转化为servlet类文件(将所有的JSP元素转化为Java代码，Servlet是一个服务器端的特殊的Java程序，被加载的JSP文件正是该请求所在的页面）
4. JSP引擎会将该servlet类文件编译为可执行的类，并将对应的原始请求转发给servlet引擎（JSP为什么可以编译servlet类文件？因为JSP是Servlet的一种简化)
5. 接受到请求的servlet引擎会根据该请求加载对应的Servlet类文件（该类就是第4步通过JSP编译的servlet类），执行过程中Servlet产生的HTML格式输出将被内嵌到HTTP的response上下文交给Web服务器
6. Web会将以静态HTML网页的形式将接收到的HTTP Respone返回给浏览器
7. 最终Web浏览器处理HTTP Response中动态产生的HTML内容

一个http请求
!(https://pic2.zhimg.com/80/v2-e8748e765b5613d223cd995d5bf9d1a1_hd.jpg)
