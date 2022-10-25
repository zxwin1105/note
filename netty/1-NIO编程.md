NIO基础

non-blocking io 非阻塞I/O

## 1. NIO三大组件

### 1.1 Channel 和 Buffer组件

channel类似于stream，channel是读写数据的双向通道，可以从buffer中读入数据，也可以将数据写出带channel。而stream只能输入或者输出，channel比stream更底层。

buffer是用于缓冲读写数据。

常见的Channel有：

- FileChannel：文件管道

- DatagramChannel：用于实现UDP连接

- SocketChannel：用于客户端实现TCP连接

- ServerSocketChannel：用于服务端实现TCP连接

常见的Buffer有：

- ByteBuffer：字节缓存，抽象类，其实现类有
  
  - MapperByteBuffer
  
  - DirectByteBuffer
  
  - HeapByteBuffer

- ShortBuffer

- IntBuffer

- LongBuffer

- FloatBuffer

- DoubleBuffer

- CharBugger

### 1.2 Selector

对于Selector的理解可以从服务器端设计的演化过程来理解。

1. 多线程版服务端设计

服务端一般都需要再统一时刻处理多条请求，为了能增加每一时刻处理请求的数量，提高服务端处理能力，就需要引入多线程。理论上线程越多，处理请求能力越强。设计如下图：

![多线程版服务端设计](../picture/netty/多线程服务端设计.png)

但是实际情况并不是线程越多越好，多线程处理存在以下缺点：

- 占用内存高，默认每个线程都会占用1M的内存，如果有大量的线程存在，可能会导致内存不足。

- 线程上下文切换成功较高。线程数超过了CPU核数，CPU就需要切换线程执行。

因此多线程版本的服务端只适用于请求较小的场景，如果存在大量的请求，会造成服务端内存、CPU压力过大。

2. 线程池版服务端设计

线程池的应用可以对创建的线程进行重复利用，避免频繁的创建销毁线程带来的额外开销。虽然解决了线程过多的问题，但是因为socket是阻塞式的，所以每个线程一次仍然只能处理一个请求。

![线程池版服务端设计](../picture/netty/线程池服务端设计.png)

线程池处理存在的缺点：

- 阻塞模式中，线程只能处理一个socket请求

- 仅适合短连接场景，每次请求处理时间较短，不会长时间阻塞线程
3. Seletor版服务端设计

selector的作用是配合一个线程来管理多个channel，获取这些channel上发生的事件，这些channel工作在非阻塞模式下，不会让线程吊在一个channel上，适合连接数较多，但是流量较低场景（low traffic）。

![selector服务端设计](../picture/netty/seletor服务端设计.png)
