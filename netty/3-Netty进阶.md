# 1. 黏包半包问题

案例：客户端在成功连接到服务端后，向服务端发送10次16k消息，观测服务端接收消息的次数和大小。

```java
// 服务端代码
@Slf4j
public class StickyPackage {

    public static void main(String[] args) {
        ChannelFuture channelFuture = new ServerBootstrap()
                .group(new NioEventLoopGroup(), new NioEventLoopGroup())
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                        ChannelPipeline pipeline = nioSocketChannel.pipeline();
                        // LoggingHandler用于日志监控
                        pipeline.addLast(new LoggingHandler());
                    }
                }).bind(8765);Channel channel = channelFuture.channel();
    }
}
```

```java
// 客户端代码
@Slf4j
public class StickyClient {

    public static void main(String[] args) {
        ChannelFuture channelFuture = new Bootstrap()
                .group(new NioEventLoopGroup())
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) {
                        ChannelPipeline pipeline = nioSocketChannel.pipeline();
                        pipeline.addLast(new ChannelInboundHandlerAdapter() {
                            // 连接建立后，会触发active事件
                            @Override
                            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                // 为测试黏包问题，连续发10次数据到服务端，观察服务端接收数据的次数，大小
                                for (int i = 0; i < 10; i++) {
                                    ByteBuf buffer = ctx.alloc().buffer(16);
                                    buffer.writeBytes(new byte[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15});
                                    ctx.writeAndFlush(buffer);
                                }
                            }
                        });
                    }
                }).connect(new InetSocketAddress(8765));
    }
}
```

> 服务端接收结果情况
> 
> 17:50:15.173 [nioEventLoopGroup-3-1] DEBUG io.netty.handler.logging.LoggingHandler - [id: 0x9d497ea1, L:/10.200.252.66:8765 - R:/10.200.252.66:10309] READ: 150B
>          +-------------------------------------------------+
>          |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
> +--------+-------------------------------------------------+----------------+
> |00000000| 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 01 |................|
> |00000010| 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 01 02 |................|
> |00000020| 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 01 02 03 |................|
> |00000030| 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 01 02 03 04 |................|
> |00000040| 05 06 07 08 09 0a 0b 0c 0d 0e 0f 01 02 03 04 05 |................|
> |00000050| 06 07 08 09 0a 0b 0c 0d 0e 0f 01 02 03 04 05 06 |................|
> |00000060| 07 08 09 0a 0b 0c 0d 0e 0f 01 02 03 04 05 06 07 |................|
> |00000070| 08 09 0a 0b 0c 0d 0e 0f 01 02 03 04 05 06 07 08 |................|
> |00000080| 09 0a 0b 0c 0d 0e 0f 01 02 03 04 05 06 07 08 09 |................|
> |00000090| 0a 0b 0c 0d 0e 0f                               |......          |
> +--------+-------------------------------------------------+----------------+

可以观察到服务器接收消息只用了一次，就将客户端发送的10次消息全部接收完成，这就产生了黏包问题。

## 1.1 问题产生的原因

## 1.2 解决问题的方法

### 1.2.1 短链接

客户端服务端建立后，只发送一次消息，就将连接断开。服务端就只会读取到一条消息。

这种方式显然不适合多数的使用场景，其性能较低。
