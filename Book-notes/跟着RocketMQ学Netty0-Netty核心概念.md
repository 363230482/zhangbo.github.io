
# Netty核心概念
本文只是跟着RocketMQ学Netty的使用，不会深入解析RocketMQ的实现。本篇先来说明一下Netty中需要了解的几个核心类。

## Bootstrap/ServerBootstrap
Netty客户端/服务端的启动类，用于帮助我们快速的配置启动客户端/服务端

## NioSocketChannel/NioServerSocketChannel
用户在配置启动类时，说明使用NIO 的selector模式，Netty可以配置使用OIO,NIO,Epoll等

## EventLoopGroup/NioEventLoopGroup
类似Java中的线程池，用于注册`Channel`,并分派Channel后续的IO事件

## EventLoop/NioEventLoop
真正处理Channel的IO事件的处理器

## Channel
连接到网络套接字或能够进行读、写、连接和绑定等I/O操作的组件。
上面的`NioSocketChannel/NioServerSocketChannel`都是它的实现类,类似`java.nio.Channel`

## Future/ChannelFuture
类似java中的Future，用于添加异步回调处理。Netty中所有的操作都是异步的

## ChannelHandler
真正处理Channel上的IO事件的处理器，又分为`ChannelInboundHandler`和`ChannelOutboundHandler`，将读写分开

## ChannelPipeline
一个Channel对应一个流水线，用于向Channel中添加`ChannelHandler`,
一个pipeline中对应多个`ChannelHandler`，每个`ChannelHandler`按照添加的顺序被执行

## ChannelHandlerContext
`ChannelHandler`的上下文，默认实现为`DefaultChannelHandlerContext`,用于通知下一个ChannelHandler开始处理

## ByteBuf
封装了java中的`ByteBuffer`,提供了更友好的读写方法

下面我们就来看看RocketMQ中怎么使用Netty的。
