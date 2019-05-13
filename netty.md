# Netty
## 一、概述
Netty是由JBOSS提供的一个java开源框架。它是一款异步的事件驱动的网络应用程序框架，支持快速地开发可维护的高性能的面向协议的服务器和客户端。
Netty是一个简单却不失强大的架构。这个架构由三部分组成——缓冲（Buffer）、通道（Channel）、事件模型（Event Model）——所有的高级特性都构建在这三个核心组件之上。

### netty的核心组件
Netty的主要构件
* Channel
* 回调
* Future
* 事件和ChannelHandler

**Channel**
Channel是Java NIO的一个基本构造。它代表一个到实体（一个能够执行一个或多个不同IO操作的程序组件、一个硬件设备、一个文件、一个套接字等）的开发连接，如读操作和写操作。目前可以把Channel看作一个传入或传出的数据载体。

**回调**
回调在广泛的编程场景中都有应用，一般是在完成某个特定的操作后对相关方法进行调用。
Netty在内部使用回调来处理事件；当一个回调被触发时，相关的事件可以被一个interfaceChannelHandler的实现处理，例如Channel激活时会调用ChannelActive方法

**Future**
Future一般用在当执行异步操作时需要获取未来的某个时候才能获取到的结果。JDK预置了interface-java.util.concurrent.Future，但是其所提供的实现，只允许手动检查对应的操作是否已经完成，或者一直阻塞直到它完成。这是非常繁琐的，所以Netty提供了它自己的实现——ChannelFuture，用于在执行异步操作的时候使用。

ChannelFuture提供了几种额外的方法，这些方法使得我们能够注册一个或者多个ChannelFutureListener实例。
监听器的回调方法operationComplete()，将会在对应的操作完成时被调用。然后监听器可以判断该操作是成功地完成了还是出错了。如果是后者，我们可以检索产生的Throwable。
通过使用ChannelFutureListener机制可以避免对操作结果进行手动检查。每个 Netty 的出站 I/O 操作都将返回一个ChannelFuture，即不会阻塞后续的操作。

**事件和ChannelHandler**
Netty使用不同的事件来通知状态的改变或者是操作的状态。事件可能包括： 
- 连接已被激活或者连接失活 
- 数据读取； 
- 用户事件； 
- 错误事件； 
- 打开或者关闭到远程节点的连接； 
- 将数据写到或者冲刷到套接字

每个事件都可以被分发给 ChannelHandler 类中的某个用户实现的方法。这是将事件驱动范式直接转换为应用程序逻辑处理比较理想的位置。 对每个事件可以进行记录日志、数据转换、应用程序逻辑处理等操作，Netty提供了大量预定义的可以开箱即用的ChannelHandler实现，包括用于各种协议(HTTP和SSL/TLS)的ChannelHandler。

### NIO实现的核心思路
* NIO模型中通常会有两个线程，每个线程绑定一个轮询器selector，在我们这个例子中serverSelector负责轮询是否有新的连接，clientSelector负责轮询连接是否有数据可读
* 服务端监测到新的连接之后，不再创建一个新的线程，而是直接将新连接绑定到clientSelector上，这样就不用IO模型中1w个while循环在死等，参见(1)
* clientSelector被一个while死循环包裹着，如果在某一时刻有多条连接有数据可读，那么通过 clientSelector.select(1)方法可以轮询出来，进而批量处理，参见(2)
* 数据的读写以内存块为单位，参见(3)


**不使用JDK原生NIO的原因**
1. 使用JDK自带的NIO需要了解太多的概念，编程复杂，一不小心bug横飞
1. Netty底层IO模型随意切换，而这一切只需要做微小的改动，改改参数，Netty可以直接从NIO模型变身为IO模型
1. Netty自带的拆包解包，异常检测等机制让你从NIO的繁重细节中脱离出来，让你只需要关心业务逻辑
1. Netty解决了JDK的很多包括空轮询在内的bug
1. Netty底层对线程，selector做了很多细小的优化，精心设计的reactor线程模型做到非常高效的并发处理
1. 自带各种协议栈让你处理任何一种通用协议都几乎不用亲自动手
1. Netty社区活跃，遇到问题随时邮件列表或者issue
1. Netty已经历各大rpc框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

**Netty有主要特性如下：** 
- 优雅的设计 
- 统一的API接口，支持多种传输类型，例如OIO,NIO 
- 简单而强大的线程模型 
- 丰富的文档 
- 卓越的性能 
- 拥有比原生Java API 更高的性能与更低的延迟 
- 基于池化和复用技术，使资源消耗更低 
- 安全性 
- 完整的SSL/TLS以及StartTLS支持 
- 可用于受限环境，如Applet以及OSGI

Netty对NIO的API进行了封装，主要通过以下手段让性能得到提升 
* 使用多路复用技术，提高处理连接的并发性 
* 零拷贝： Netty的接收和发送数据采用DIRECT BUFFERS，使用堆外直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝 
* Netty提供了组合Buffer对象，可以聚合多个ByteBuffer对象进行一次操作 
* Netty的文件传输采用了transferTo方法，它可以直接将文件缓冲区的数据发送到目标Channel，避免了传统通过循环write方式导致的内存拷贝问题 
* 内存池：为了减少堆外直接内存的分配和回收产生的资源损耗问题，Netty提供了基于内存池的缓冲区重用机制 
* 使用主从Reactor多线程模型，提高并发性 
* 采用了串行无锁化设计，在IO线程内部进行串行操作，避免多线程竞争导致的性能下降 
* 默认使用Protobuf的序列化框架 
* 灵活的TCP参数配置

 
## 二、体系结构图
![](https://i.imgur.com/cAvYO4T.png)

 　　
## 三、Netty的核心结构
Netty是典型的Reactor模型结构，在实现上，Netty中的Boss类充当mainReactor，NioWorker类充当subReactor（默认NioWorker的个数是当前服务器的可用核数）。
在处理新来的请求时，NioWorker读完已收到的数据到ChannelBuffer中，之后触发ChannelPipeline中的ChannelHandler流。
Netty是事件驱动的，可以通过ChannelHandler链来控制执行流向。因为ChannelHandler链的执行过程是在subReactor中同步的，所以如果业务处理handler耗时长，将严重影响可支持的并发数。
![](https://i.imgur.com/ApfZAV5.png)

## 代码示例
```
Server-main:

ChannelFactory factory = new NioServerSocketChannelFactory(Executors.newCachedThreadPool(), Executors.newCachedThreadPool());
ServerBootstrap bootstrap = new ServerBootstrap(factory);
bootstrap.setPipelineFactory(new ChannelPipelineFactory(){  
    @Override
    public ChannelPipeline getPipeline() throws Exception {
        return Channels.pipeline(new TimeServerHandler());
    }
});
bootstrap.setOption("child.tcpNoDelay", true);
bootstrap.setOption("child.keepAlive", true);
bootstrap.bind(new InetSocketAddress(1989));
```
ChannelFactory是一个创建和管理Channel通道及其相关资源的工厂接口，它处理所有的I/O请求并产生相应的I/O ChannelEvent通道事件。这个工厂并自己不负责创建I/O线程。应当在其构造器中指定该工厂使用的线程池，这样我们可以获得更高的控制力来管理应用环境中使用的线程。

ServerBootstrap是一个设置服务的帮助类。设置了一个继承自ChannelPipelineFactory的匿名类，用来作为ChannelPipeline通道，当服务器接收到一个新的连接，一个新的ChannelPipeline管道对象将被创建，并且所有在这里添加的ChannelHandler对象将被添加至这个新的ChannelPipeline管道对象。

```
Server-Handler:
@Override
public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception { 
    //TimeServer    
    Channel ch = e.getChannel();
    ChannelBuffer time = ChannelBuffers.buffer(8);
    time.writeLong(System.currentTimeMillis()); 
    ChannelFuture future = ch.write(time);  
    future.addListener(new ChannelFutureListener() {        
        @Override      
        public void operationComplete(ChannelFuture arg0) throws Exception {        
            Channel ch = arg0.getChannel();
            ch.close();
        }
    });
}
```

Handler中是我们的业务逻辑，在Server的Handler里重载了channelConnected方法，当收到连接请求时，将当前服务器时间写入到Channel，并且在写完后触发关闭Channel。
```
 Client-main:
ChannelFactory factory = new NioClientSocketChannelFactory(Executors.newCachedThreadPool(), Executors.newCachedThreadPool());
ClientBootstrap bootstrap = new ClientBootstrap(factory);
bootstrap.setPipelineFactory(new ChannelPipelineFactory() { 
    @Override  
    public ChannelPipeline getPipeline() throws Exception {
        return Channels.pipeline(new TimeClientHandler());
    }
});
bootstrap.setOption("tcpNoDelay",true);
bootstrap.setOption("keepAlive", true);
bootstrap.connect(new InetSocketAddress("127.0.0.1", 1989));
```

Client端初始化Netty的过程和Server类似，只是将使用到的类替换为Client端的。
```
Client-Handler:
@Override
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
    ChannelBuffer buf = (ChannelBuffer)e.getMessage();
    //Client端的Handler里，我们将从服务器端接收到的信息转换为时间打印到控制台
    Long currentTimeMillis = buf.readLong();
    System.out.println(new Date(currentTimeMillis));
    e.getChannel().close();
}
```

**基于HTTP协议的服务器端实现**
```
//HttpServerPipelineFactory.java
public class HttpServerPipelineFactory implements ChannelPipelineFactory {
    @Override
    public ChannelPipeline getPipeline() throws Exception {
        ChannelPipeline pipeline = Channels.pipeline();
        pipeline.addLast("decoder", new HttpRequestDecoder());
        pipeline.addLast("encoder", new HttpResponseEncoder());
        pipeline.addLast("handler", new HttpServerHandler());
        return pipeline;
    }
}
```
新建一个HttpServerPipelineFactory类，在getPipeline()方法中添加了对Http协议的支持。
```
// HttpServer.java
bootstrap.setPipelineFactory(new HttpServerPipelineFactory());
```
在Server里面使用我们新建的HttpServerPipelineFactory。
```
//HttpServerHandler.java
public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
    DefaultHttpRequest defaultHttpRequest = (DefaultHttpRequest)e.getMessage();
    String uri = defaultHttpRequest.getUri();
    byte[] data = defaultHttpRequest.getContent().array();
    String content = URLDecoder.decode(new String(data),"utf-8").trim();
    System.out.println(uri+"|"+content);
    Channel ch = e.getChannel();
    HttpResponse response = new DefaultHttpResponse(HTTP_1_1, OK);
    ChannelBuffer buffer = new DynamicChannelBuffer(2048);
    buffer.writeBytes("200".getBytes("UTF-8"));
    response.setContent(buffer);
    response.setHeader("Content-Type", "text/html;charset=UTF-8");
    response.setHeader("Content-Length", response.getContent().writerIndex());
    if (ch.isOpen() && ch.isWritable()) {   
        ChannelFuture future = ch.write(response);  
        future.addListener(new ChannelFutureListener() {        
            @Override      
            public void operationComplete(ChannelFuture arg0) throws Exception {            
                Channel ch = arg0.getChannel();         
                ch.close();
            }   
        });
    }
}
```

在Handler里面我们可以直接拿到DefaultHttpRequest类型的对象，因为Netty已经用HttpRequestDecoder帮我们把接受到的数据都转换为HttpRequest类型了。
