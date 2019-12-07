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




