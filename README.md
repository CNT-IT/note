# 目录

* <a href="#1">netty 宏观概述</a>

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

* <a href="cmake">cmake 使用摘要</a>


https://www.bilibili.com/video/av76127421






# <a name="1">netty 宏观概述</a>
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

利用 netty 构建一个简单的 http 服务器，便是我们的第一个应用程序 

需求很简单，在浏览器的地址栏里输入 http://127.0.0.1:8080/ 并回车，然后显示 Hello World

这在以往用 SpringMvc 开发是轻而易举的事，然而使用 netty 开发一个简单的 hello world 应用程序，对于初学者来说并不是那么的轻松。接下来，将一步一步实现 netty 第一个应用程序的开发，了解 netty 的开发流程

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


5. 小结</br>
通过 Hello World 熟悉 netty 开发的一个基本步骤，主要分为三步
* 通过 ServerBootstrap ，启动 netty 服务器
* 编写 Channel 处理器初始化器，把 编码器、解码器、业务处理器等处理器加入到 ChannelPipeLine 对 I/O 数据进行处理
* 根据需要自定义编解码器、业务处理器


&nbsp; 
# <a name="3">netty 执行流程分析 </a>

从浏览器发送 http://127.0.0.1:8080/ 请求到服务器发送 Hello World 响应，期间发生了哪些事件，netty 的执行流程是怎么样的，还是以 Hello World 为例作一些调整进行阐述

> 完整代码见  src/apptwo

## 1.  项目结构

```
|-- apptwo
    |-- pom.xml
    |-- src
        |-- main
            |-- java
                |-- cnt
                    |-- HttpServerHandler.java
                    |-- NettyServer.java
                    |-- ServerInitializer.java
```

## 2. 修改 HttpServerHandler 代码
 
 自定义的 HttpServerHandler 继承了 SimpleChannelInboundHandler 并实现了其channelRead0 方法获取数据，可以发现 SimpleChannelInboundHandler 有如下的类图关系

 ![](./res/SimpleChannelInboundHandler1.png)
 
 可以发现其父类定义了一系列的事件方法，故修改 HttpServerHandler 添加以下代码
 ```java
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        super.handlerAdded(ctx);
        System.out.println("handlerAdded");
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        super.channelRegistered(ctx);
        System.out.println("channelRegistered");
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        super.channelUnregistered(ctx);
        System.out.println("channelUnregistered");
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
        System.out.println("channelActive");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        System.out.println("channelInactive");
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        super.channelReadComplete(ctx);
        System.out.println("channelReadComplete");
    }


    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        super.exceptionCaught(ctx,cause);
        System.out.println("exceptionCaught");
    }
 ```

## 3. 运行访问
打开浏览器访问，观察控制台输出将看到如下输出
![](./res/channelFlow.png)

根据输出结果，不难得出 channel 回调事件的执行顺序







&nbsp; 
# <a name="cmake">camke 使用摘要 </a>

## 1. 认识CMake
一款优秀的工程构建工具

* 开放源代码，具有BSD许可
* 跨平台，支持 Linux 、Mac、Windows等不同的操作系统
* 编译语言简答，易用，简化编译构建过程和编译过程


## 2.CMake 语法主体框架

* 如何去组织一个项目的编译框架
* 最终输出目标有哪些（可执行程序、动态库、静态库）
* 如何配置输出目标文件的指定编译参数（需要哪些编译参数及环境、需要哪些源文件）
* 如何为指定的输出目标连接参数（怎么配置内外部依赖的pkg、lib，怎么链接外部库）


### 主体框架

* 工程配置部分

工程名、编译调试模式、编译系统语言

* 依赖执行部分

工程包、头文件、依赖库等

* 其他辅助部分（非必须）

参数打印、遍历目录等

* 判断控制部分（非必须）

条件判断、函数定义、条件执行等

--------------------------------------------------------------
1. 基本语法

> command(arg1 arg2 ...)  运行命令

> set(var_name var_value) 定义变量，或者给已经存在的变量赋值

> command(arg1 $(var_name)) 使用变量

--------------------------------------------------------------
2. 工程配置部分

> cmake_mini_required(VERSION num)  cmake 最低版本号要求

> project(cur_project_name)  项目信息

> set(CMAKE_CXX_FLAGS "XXX")  设置编译器

> set(CMAKE_BUILD_TYPE "XXX")  设定编译模式，如 Debug/Release

---------------------------------------------------------------
3. 依赖执行部分

> find_package(std_lib_name VERSION REQUIRED)   引入外部依赖

> add_library(<name> [lib_type] source1)   生成库类型（动态，静态）

> include_directories(${std_lib_name_INCLUDE_DIRS})  指定 include路径，放在 add_executable前面

> add_executeable(cur_project_name xxx.cpp)  指定生成目标

> target_link_library(${std_lib_name_LIBRARIES})  指定library路径，放在 add_executeable 后面

-------------------------------------------------------------

4. 其他辅助部分

> function(function_name  arg)  定义一个函数

> add_subdirectory(dir)  添加一个字目录

> AUX_SOURCE_DIRECTORY(. SRC_LIST)  查找当前目录所有文件，并保存到SRC_LIST变量中

> `FOREACH(one_dir ${SRC_LIST})
     MESSAGE(${one_dir})
  ENDFOREACH(onedir) `

> while(condition)
   command(args)
   endwhile(condtion)

---------------------------------------------------
5. 判断控制部分

```
if(expression)
    command(args)
else(expression)
   command(args)
endif(expression)
```
> if(var) 

> if(not var)

> if(var1 AND var2)

> if(COMMAND cmd)

> if(EXISTS file)


## 常用指令和变量

```
 PROJECT 
 ADD_EXECUTEABLE
 ADD_SUBDIRECTORY
 INCLUDE_DIRECTORY
 LINK_DIRECTORY
 TARGET_LINK_LIBRARIES
 ADD_LIBRARY
 AUX_SOURCE_DIRECTORY
 FOREACH
 MEAASAGE
 IF ELSE ENDIF
 WHILW ENDWHILE
 FIND_PACKAGE
 SET
 ```
> ADD_DEFINITIONS 

为源文件的编译添加 -D 引入的宏定义
命令格式 ADD_DEFINITIONS(-DFOO -DBAR)

> OPTION(<var> "description" [init_value])

提供用户可以选择的选项

> ADD_CUSTOM_COMMAND/TARGET

[COMMAND] 为工程添加一条自定义的构建规则
[TARGET] 用于给指定的名称的目标执行指定的命令，该目标没有输出，并始终被构建

> ADD_DEPENDENCIES

用于链接时依赖的问题，用于target_link_libraries 可以搞定？

当定义的target依赖的另一个target，确保在源代码编译本target之前，其他的target已经被构建

> INSTALL

用于定义安装规则，安装的内容可以包括目标二进制、动态库、静态库以及文件、目录、脚本等

> TARGET_INCLUDE_DIRECTORIES

TARGET_INCLUDE_DIRECTORIES(<target> [SYSTEM|BEFORE] <INTERFACE|PUBLIC|PRIVATE> [items])


设置 include 文件查找的目录，具体包含头文件应用形式，安装位置等


> SET_TARGET_PROPERTIES

设置目标的一些属性来改变他们的构建方式

> ENABLE_TESTING/ADD_TEST



CMAKE_INSTALL_PREFIX 构建install 的路径

$ENV{HOME} HOME 环境下的目录

PROJECT_NAME 工程名变量

<PKG>_INCLUDE_DIR 导入包头文件全路径

<PKG_LIBRARIES> 导入库文件的全路径

PROJECT_SOURCE_DIR 构建工程的全路径

CMAKE_VERSION cmake版本号

CMAKE_SOURCE_DIR 源码树的顶层路径



















