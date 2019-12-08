# 目录

* <a href="#1">netty 宏观理解</a>

* <a href="#2">netty 第一个应用程序</a>

* <a href="#3">netty 执行流程分析</a>

* <a href="#4">netty 重要组件介绍</a>

* <a href="#6">netty socket编程</a>

* <a href="#7">netty 多客户端通信</a>

* <a href="#8">netty 心跳检测机制</a>

* <a href="#9">netty websocket实现</a>

* <a href="#10">netty 集成protobuf通信</a>

* <a href="#11">Bio 与 Nio 详解</a>

* <a href="#12">Java nio 介绍</a>





# <a name="1">netty 宏观理解</a>
## 1. netty 是什么

netty 的官网中有这么一句话

> Netty is an asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients

简单理解 netty 是一个异步基于事件驱动的高性能网络框架，并且 netty 本身实现了大量的常见协议，简化了应用程序的开发

&nbsp; 

异步，与我们平常编写的代码有点不太一样，netty 中所有的请求都是异步的，对于请求结果、请求成功与否并不能马上返回，而是通过设置监听器回调来获得请求结果

事件驱动，所有的网络请求在网络通信里无非是 客户端发起连接 服务端接收连接 客户端发送数据 服务端接收数据 服务端发送数据，所以 netty 把这些当作一个一个事件进行处理，当有一个连接事件发生的时候，netty 会通过回调的方式告知应用程序 有客户端连接了

协议，netty 中另外一个重要的方面就是对协议的支持，在所有的网络应用程序编程，对于开发者最重要的就是制定协议，协议约定了客户端与服务器通信的数据格式，netyy 本身已经实现了很多的协议，例如 http协议、websocket协议、protobuf协议、fttp协议等

## 2. jdk NIO 

jdk 原生也有一套网络应用程序 API，因其不好用且存在很多问题，故 jdk 原生网络 API 在日常的开发中用的很少

* API 使用繁杂，需要熟练掌握 selector、serverSocketChannel、socketChannel 等API
* 辅助技能，必须熟练掌握 java 多线程开发，熟悉 Reactor 线程模型，才能开发出高质量的网络应用程序
* 网络异常处理难度大，常见的客户端断线重连、tcp 的粘包半包处理、网络闪断、网络拥塞等处理难度非常大
* jdk NIO bug，jdk NIO 本身存在的 Epoll bug 会导致 selector 空轮询，造成 cpu 100%

## 3. netty 特点
netty 是对 jdk NIO 进行了再次包装处理，简化 java 网络应用程序的开发

* netty 的 IO 模型和 Reactor 线程模型，使其具有很高的性能，IO 模型决定了netty 如何收发数据，Reactor 线程模型决定了 netty 如何处理数据
* netty 支持很多应用层协议，提供了很多的 decoder encoder
* netty 解决了 tcp粘包半包的问题
* netty 规避了 jdk NIO 本身的一些问题

## 4. netty 常见的应用

* Dubbo RPC 远程通信框架，默认使用 netty 作为通信基础组件
* Flink、Spark等经典的大数据应用程序，也是基于 netty 进行通信
* Moquette MQTT协议实现，基于 netty 进行协议的编码解码
* ......



&nbsp; 
# <a name="2">netty 第一个应用程序</a>

使用 netty 开发一个简单的 hello world 应用程序，对于初学者来说并不是那么的轻松。接下来，将一步一步实现 netty 第一个应用程序的开发，了解 netty 的开发流程

&nbsp;

> 完整代码见  src/appone

&nbsp;

## 1.  项目结构

```
|-- appone
    |-- pom.xml
    |-- src
        |-- main
            |-- java
                |-- cnt
                    |-- HttpServerHandler.java
                    |-- NettyServer.java
                    |-- ServerInitializer.java
```

## 2. 编写代码

1. pom.xml 文件引入 netty jar 包依赖
```xml
 <dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.43.Final</version>
</dependency>
```

2. 编写 netty 服务端启动代码
```java
 public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); //[1]
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try{
            ServerBootstrap serverBootstrap = new ServerBootstrap();//[2]
            serverBootstrap.group(bossGroup,workGroup)//[3]
                    .channel(NioServerSocketChannel.class)//[4]
                    .childHandler(new ServerInitializer())//[5]
                    .option(ChannelOption.SO_BACKLOG, 128)          // (6)
                    .childOption(ChannelOption.SO_KEEPALIVE, true); // (7);

            ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();//[8]
            channelFuture.channel().closeFuture().sync();//[9]
        }finally {
            bossGroup.shutdownGracefully();//[10]
            workGroup.shutdownGracefully();
        }
    }
```
上面这段代码展示了 netty 服务端启动的一个基本步骤</br>

> ## 1. NioEventLoopGroup
> 初始化主从事件循环组，可简单理解为两个“线程池”，主线程池用于处理客户端的连接，从线程池用于网络 I/O 数据的收发


> ## 2. ServerBootstrap
> 初始化 ServerBootstrap ， 是 netty 服务端的启动类 , 聚合 netty 各种组件，完成 netty 服务器的启动

> ## 3. serverBootstrap.group
>设置 serverBootstrap 启动线程组

>## 4. serverBootstrap.channel
>设置 netty 服务器的 channel (通道) 类型

> ## 5. serverBootstrap.childHandler
>初始化客户端与服务端连接通道的数据处理器，一般包括数据的编码器、解码器、数据的业务逻辑处理器，ServerInitializer 是我们自定义的通道处理器初始化器

>## 6. serverBootstrap.option
>配置 ServerSocketChannel 通道的选项，即服务器生成用于接收客户端连接的通道选项

> ## 7. serverBootstrap.childOption
> 配置 socketChannel 通道的选项，客户端发与服务器连接通道的选项

> ## 8. serverBootstrap.bind(8080).sync()
> 绑定端口并等待接收客户端的连接

> ## 9. channelFuture.channel().closeFuture().sync()
> 等待服务器通道关闭


2. 编写自定义通道处理器初始化器
```java
public class ServerInitializer extends ChannelInitializer<SocketChannel> {
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        ChannelPipeline pipeline = socketChannel.pipeline();
        pipeline.addLast("HttpServerCodecn",new HttpServerCodec());
        pipeline.addLast("HttpServerHandler",new HttpServerHandler());
    }
}
```
上面是通道初始化器的一个基本格式,主要就是添加各种 I/O 处理器

> ChannelInitializer：通道初始化器 需要继承 ChannelInitializer 并实现 initChannel 方法，SocketChannel 是一个可选的通道类型

> ChannelPipeline：Channel 是通过 ChannelPipeline 添加的处理器，对 I/O 数据处理。数据的处理流程一般为如下：</br>
客户端发送的数据 --> 数据解码器 --> 业务逻辑处理器 --> 数据编码器 --> 服务端发送数给客户端

> HttpServerCodecn: netty 本身已经实现的 http 协议编解码，包括 httpRequet 解码器 和 httpResponse 响应编码器

> HttpServerHandler：这个是我们自定义的业务逻辑处理器，这里只是简单的向浏览器 发送 Hello World

3. 自定义业务逻辑处理器
```java
public class HttpServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, HttpObject httpObject) throws Exception {
        ByteBuf msg = Unpooled.copiedBuffer("Hello World", CharsetUtil.UTF_8);
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,msg);
        response.headers().set(HttpHeaderNames.CONTENT_TYPE,"text/plain");
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH,msg.readableBytes());

        channelHandlerContext.channel().writeAndFlush(response);
    }
}
```
上面代码，定义了一个简单的业务逻辑处理器，向浏览器发送一串文字
> SimpleChannelInboundHandler : netty 定义了好几种入站处理器，供开发者根据需要选择不同的入站处理器接收数据并作出处理, SimpleChannelInboundHandler 是一个泛型，用于接收不同的数据格式，HttpObject 表示经过解码器的处理之后，转换成HttpObject 对应的数据格式，由于是 http 协议，故我们用 HttpObject 来存放浏览器的请求信息

> ChannelHandlerContext: 通道处理器上下文，获取到对应的 Channel 并向客户达发送数据

> FullHttpResponse: netty 定义的 http 响应数据格式

4. 启动服务，打开浏览器发送请求

> http://127.0.0.1:8080/ </br>
> 可以看到浏览器显示：  Hello World


5. 小结
通过 Hello World 熟悉 netty 开发的一个基本步骤，主要分为三步
* 通过 ServerBootstrap ，启动 netty 服务器
* 编写 Channel 处理器初始化器，把 编码器、解码器、业务处理器等处理器加入到 ChannelPipeLine 对 I/O 数据进行处理
* 根据需要自定义 编解码器、业务处理器




















