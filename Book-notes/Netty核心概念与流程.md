# Netty

## 核心概念
ServerBootstrap/Bootstrap 服务端/客户端启动器
Future              继承自java的 Future，用于异步回调，子接口如 ChannelFuture
EventLoopGroup      反应器，用于管理一组的 EventLoop, 如 NioEventLoopGroup
NioEventLoop        事件分派器，一个实例对应一个java Thread，对应一个IO的底层事件：OP_READ,OP_WRITE,OP_ACCEPT,OP_CONNECT等，获取到事件后分派给具体的handler执行
Channel             通道，处理底层的IO的事件,对应一个连接，不同于java的Channel
ChannelHandler      通道处理器，子类如 ChannelInboundHandler/ChannelOutboundHandler，进行读写IO操作
ChannelPipeline     将 Channel 和 ChannelHandler 绑定在一起形成一个流水线操作
ChannelHandlerContext       
=======
在Channel、ChannelPipeline、ChannelHandlerContext三个类中，会有同样的出站和入站处理方法，同一个操作出现在不同的类中，功能有何不同呢？
如果通过Channel或ChannelPipeline的实例来调用这些方法，它们就会在整条流水线中传播。
如果是通过ChannelHandlerContext通道处理器上下文进行调用，就只会从当前的节点开始执行Handler业务处理器，并传播到同类型处理器的下一站（节点）。
=====
Channel、Handler、ChannelHandlerContext三者的关系为：
Channel通道拥有一条ChannelPipeline通道流水线，每一个流水线节点为一个ChannelHandlerContext通道处理器上下文对象，
每一个上下文中包裹了一个ChannelHandler通道处理器。在ChannelHandler通道处理器的入站/出站处理方法中，
Netty都会传递一个Context上下文实例作为实际参数。通过Context实例的实参，在业务处理中，
可以获取ChannelPipeline通道流水线的实例或者Channel通道的实例。



===========================================
## 关键步骤与Java NIO的关联

### NioServerSocketChannel#accept 接受客户端的连接
io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages
io.netty.util.internal.SocketUtils#accept

### 父EventLoopGroup注册子ChannelHandler到子EventLoopGroup执行
io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor#channelRead
io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)

io.netty.channel.AbstractChannel.AbstractUnsafe#register ## 最终实例为: NioSocketChannel#NioSocketChannelUnsafe
io.netty.channel.AbstractChannel.AbstractUnsafe#register0

### 调用java NIO读取数据到ByteBuf,然后pipeline.fireChannelRead(byteBuf)传入pipeline
io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)
io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read

### ByteBuf的释放，release后方便内存回收
io.netty.channel.DefaultChannelPipeline.TailContext#channelRead
io.netty.util.ReferenceCountUtil#release(java.lang.Object)
#### 入站 释放 ByteBuf 的几种方式
1.手动 byteBuf.release
2.自定义InBoundHandler继承自SimpleChannelInboundHandler,并将业务处理逻辑重写到 SimpleChannelInboundHandler#channelRead0 中，由父类释放
3.调用 ChannelHandlerContext.fireChannelRead(msg); 最终由DefaultChannelPipeline.TailContext#channelRead来调用release

#### 出站 ByteBuf的释放

io.netty.channel.AbstractChannelHandlerContext#writeAndFlush(java.lang.Object, io.netty.channel.ChannelPromise)

