## 1. 概述

## 2. hello world

netty实现服务端

```java
package com.netty.hello;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import lombok.extern.slf4j.Slf4j;

/**
 * 使用netty实现hello world网络服务端
 *
 * @author zhaixinwei
 * @date 2022/10/27
 */
@Slf4j
public class HelloServer {

    public static void main(String[] args) {
        // 1. 服务端启动类
        new ServerBootstrap()
                // 2. 添加eventLoop，真正去处理事件的组件
                .group(new NioEventLoopGroup())
                // 3. 选择channel实现
                .channel(NioServerSocketChannel.class)
                // 4. 添加处理器
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                        // pipeline工作流程管道
                        nioSocketChannel.pipeline().addLast(new StringDecoder());
                        nioSocketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                log.debug("msg:{}", msg);
                            }
                        });
                    }
                }).bind(8899);
    }
}
```

netty实现客户端

```java
package com.netty.hello;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringEncoder;
import lombok.extern.slf4j.Slf4j;

import java.net.InetSocketAddress;

/**
 * @author zhaixinwei
 * @date 2022/10/27
 */
@Slf4j
public class HelloClient {
    public static void main(String[] args) throws InterruptedException {
        new Bootstrap()
                .group(new NioEventLoopGroup())
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                        nioSocketChannel.pipeline().addLast(new StringEncoder());
                    }
                })
                .connect(new InetSocketAddress(8899))
                .sync()
                .channel()
                .writeAndFlush("hello netty");
    }
}
```

## 3. 组件

### 3.1 EventLoop

EventLoop（事件循环对象）本质是一个单线程执行器，维护了一个Selector。其线程任务（#run()方法）就是处理Channel上源源不断的IO事件。

EventLoop继承关系比较复杂：

- 一条线继承自j.u.cScheduleExecutorService，包含了线程池中所有的方法。

- 一条线继承自netty自己的OrderedEventExecutor。

EventLoopGroup（事件循环组）是一组EventLoop，Channel一般会调用EventLoopGroup的register方法来绑定其中一个EventLoop，后续这个Channel上的IO事件都由EventLoop来处理（保证率IO事件处理室的线程安全）。

### 3.2 Channel

### 3.3 Future & Promise

### 3.4 Handler &  Pipeline

### 3.5 ByteBuf

Netty中重新封装了ByteBuffer。ByteBuf默认初始化容量为256kb，支持动态扩容。

1. 关于ByteBuf内存

ByteBuf可以选择使用堆内存和直接内存。

```java
// 使用直接内存
ByteBuf buffer0 = ByteBufAllocator.DEFAULT.buffer();
ByteBuf buffer1 = ByteBufAllocator.DEFAULT.directBuffer();
// 使用队内存
ByteBuf buf = ByteBufAllocator.DEFAULT.heapBuffer();
```

使用直接内存可以少一次复制过程，速度较快，且不需要GC。

2. 关于ByteBuf池化

Netty提供了ByteBuf的池化功能，可以实现对ByteBuf进行重复使用，减少对象创建和回收的过程。在Netty4.1之后默认ByteBuf开启了池化功能。

> 池化功能开关配置：
> 
> -Dio.netty.allocator.type={unpooled|pooled}
