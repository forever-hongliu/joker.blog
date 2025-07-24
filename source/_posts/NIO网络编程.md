---
title: NIO网络编程
date: 2025-07-24 09:55:53
tags: 
    - Nio
    - Java
categories: NIO
cover: https://cdn.pixabay.com/photo/2019/10/18/14/42/technology-4559283_640.jpg
---


# 基本知识
## 网络三种I/O模型
+ BIO：（同步阻塞IO），jdk1.4以前的java.io包
+ NIO：（同步非阻塞IO），jdk1.4出现的java.nio包
+ AIO：（异步非阻塞IO），jdk1.7补充在java.nio包下

## 阻塞与非阻塞
+ 阻塞：没有数据传过来时，读会阻塞直到有数据；缓冲区满时，写操作也会阻塞
+ 非阻塞：遇到阻塞的情况，直接返回；轮询访问是否有读写操作

## 同步和异步
+ 同步：数据就绪后需要主动去读是同步
+ 异步：数据就绪后通过回调去告知是异步

# JDK中的NIO
> NIO库是JDK1.4中引入的，NIO弥补了原来I/O的不足，它在标准Java代码中提供了高速的、面向块的I/O（buffer）
>

## NIO和BIO的主要区别
### 面向流与面向缓冲
+ **BIO是面向流的，NIO是面向缓冲区的**；Java IO面向流意味着每次从流中读一个或者多个字节，直到读取所有字节，数据没有被缓存在任何地方，此外它不能前后移动流中读数据，如果需要前后移动流中读取的数据，需要先将它缓存到一个缓冲区

![1](image1.png)

+ Java NIO的缓冲导向方法略有不同，数据读取到一个缓冲区，需要时可在缓冲区前后移动，增加了处理数据的灵活性；但是还需要检查是否该缓冲区包含所有需要处理的数据，需要确保党更多的数据读入缓冲区时，不要覆盖缓冲区中尚未处理的数据

![2](image2.png)

### 阻塞与非阻塞IO
Java IO的各种流是阻塞的，这意味着，当一个线程调用read()或write()时，该线程被阻塞，直到有一些数据被读区，或数据完全写入，该线程在此期间不能再干任何事情

Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就说嘛都不会获取，而不是保持线程阻塞，所以直至数据变的可读取之前，该线程可以继续做其它的事情；非阻塞写也是如此；一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事；线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道（channel）

## NIO三大核心组件
+ Selector选择器
+ Channel管道
+ Buffer缓冲区

### Selector
> 英文含义是选择器、也称为轮询代理器、事件订阅器、channel容器管理机
>

Java NIO的选择器允许一个单独的线程来监视多个输入通道，可以注册多个通道使用一个选择器，然后使用一个单独的线程来操作这个选择器，进而选择通道；这些通道里已经有可以处理的输入或者选择已准备写入的通道，这种选择机制，使得一个单独的线程很容易来管理多个通道

应用程序将向Selector对象注册需要它关注的Channel，以及具体的某一个Channel会对那些IO事件感兴趣；Selector中也会维护一个“已经注册的channel”的容器

### Channel
通道，被建立的一个应用程序和操作系统交互事件、传递内容的渠道；那么既然是和操作系统进行内容的传递，那么说明应用程序可以通过通道读取数据，也可以通过通道向操作系统写数据，而且可以同时进行读写。

+ 所有被Selector注册的通道，只能是继承了SelectbleChannel类的子类；
+ ServerSocketChannel：应用服务程序的监听通道，应用程序才能向操作系统注册支持“多路复用IO”的端口监听；同时支持UDP和TCP协议；
+ SocketChannel：TCP
    - Socket套接字的监听通道，一个Socket套接字对应了一个客户端IP：端口到服务器IP：端口的通信连接

通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。

### Buffer缓冲区
网络通讯中负责数据读写的区域

# JDK的NIO实战
![](image3.png)

```java
public class NioServer{
    
    private static NioServerHandle nioServerHandle;

    public static void start() {
        if (noiServerHandle != null) {
            nioServerHandle.stop();
        }
        nioServerHandle = new NioServerHandle(8888);
        new Thread(nioServerHandle, "Server").start();
    }

    public static void main(String[] args) {
        start();
    }
}
```

```java
public class NioServerHandle implements Runnable {
    private Selector selector;
    private ServerSocketChannel serverChannel;
    private volatile boolean started;

    public NioServerHandle(int port) {
        try {
            selector = Selector.open();
            serverChannel = ServerSocketChannel.open();
            serverChannel.configureBlocking(false); // 开启阻塞模式
            serverChannel.socket().bind(new InetSocketAddress(port));
            serverChannel.register(selector, SelectionKey.OP_ACCEPT);
            started = true;
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    public void stop() {
        started = false;
    }

    @Override
    public void run() {
        while(started) {
            try {
                selector.select();		// 没有事件会挂起
                Set<SelectionKey> keys = selector.selectedKeys(); // 获取当前所有事件
                Iterator<SelectionKey> iterator = keys.iterator();
                SelectionKey key = null;
                while(iterator.hasNext()) {
                    key = iterator.next();
                    try {
                        handleInput(key);
                    } catch(Exception e) {
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    } finally {
                        iterator.remove();		// 移除，防止重复处理
                    }
                }
            } catch(Throwable t) {
                t.printStackTrace();
            }
        }
        if (Selector != null) {
            try {
                select.close();
            } catch(Exception e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            if (key.isAcceptable()) {		// 已连接
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                SocketChannel sc = ssc.accept();	// 获取连接的socketChannel
                sc.configureBlocking(false);		// 设置非阻塞
                sc.register(selector, SelectionKey.OP_READ);	// 注册读取事件
            }		
            if (key.isReadable()) {			// 数据已准备好
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer buffer = ByteBuffer.allocate(1024);		// 开辟缓冲区
                int readBytes = sc.read(buffer);					// 读取
                if (readBytes > 0) {								
                    buffer.flip();   // 转换成读取状态
                    byte[] bytes = new byte[buffer.remaining());	// 创建字节数组
                    buffer.get(bytes);
                    String message = new String(bytes, "UTF-8");
                    System.out.println("服务端收到消息:" + message);
                    String result = "服务端响应消息:" + message);
                    doWriter(sc, result);
                } else if (readBytes < 0) {
                    key.cancel();
                    sc.close();
                }
            }
        }
    }

    private void doWriter(SocketChannel sc, String msg) throw IOException {
        byte[] bytes = msg.getBytes();
        ByteBuffer buffer = Buffer.allocate(bytes.length);		// 开辟写缓冲区
        buffer.put(bytes);										// 塞数据
        buffer.flip();
        sc.writer(buffer);										// 发送缓冲区的字节数组
    }
}

```

```java
public class NioClient {

    private static NioClientHandle nioClientHandle;

    public void start() {
        if (nioClientHandle != null) {
            nioClientHandle.stop();
        }
        nioClientHandle = new NioClientHandle("127.0.0.1", "8888");
        new Thread(nioClientHandle, "Client").start();
    }

    public static void main(String[] args) {
        start();
    }
}
```

```java
public class NioClientHandle implements Runnable {
    private Selector selector;
    private SocketChannel socketChannel;
    private String host;
    private Integer port;
    private volatile boolean started;

    public NioClientHandle(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
            started = true;
        } catch(IOException e) {
            e.printStackTrace();
            System.exit(-1);
        }
    }

    public void stop() {
        this.started = false;
    }

    @Override
    public void run() {
        try {
            doConnect();
        } catch (IOException e) {
            e.printStockTrace();
            System.exit(-1);
        }
        while(started) {
        	try {
                this.selector.select();
                Set<SelectionKey> keys = this.selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();
                SelectionKey key = null;
                while(iterator.hasNext()) {
                    key = iterator.next();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        e.printStockTrace();
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    }
                }
            } catch (IOException e) {
                e.printStockTrace();
                System.exit(-1);
            }
        }
        if (selector != null) {
            try {
                selecor.close();
            } catch (IOException e) {
                e.printStockTrace();
            }
        }
    }

    public void doConnect() {
        if (this.socketChannel.connect(new InetSocketAddress(this.host, this.port))) {
            this.socketChannel.register(this.selector, SelectionKey.OP_READ);
        } else {
            this.socketChannel.register(this.selector, SelectionKey.OP_CONNECT);
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            SocketChannel sc = (SocketChannel) key.channel();
            // 判断是否连接
            if (key.isConnectable()) {
                // 判断是否完成连接
                if (key.finishConnect()) {
                    sc.register(selector, SeletionKey.OP_READ);
                } else {
                    System.exit(-1);
                }
            }
            if (key.isReadable()) {
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(buffer);
                if (readBytes > 0) {
                    buffer.flip();
                    byte[] bytes = new byte[buffer.remaining()];
                    buffer.get(bytes);
                    String result = new String(bytes, "UTF-8");
                    System.out.println("客户端收到消息:" + result);
                } else if (readBytes < 0) {
                    key.cannel();
                    sc.close();
                }
            }
        }
    }

    private void doWrite(SocketChannel sc, String msg) throws IOException {
        byte[] bytes = msg.getBytes();
        ByteBuffer buffer = ByteBuffer.allocate(bytes.length);
        buffer.put(bytes);
        buffer.flip();
        sc.write(buffer);
    }

    public void sendMsg(String msg) throws IOException {
    	diWrite(this.socketChannel, msg);
    }
}
```

+ Selector对象是通过静态工厂方法 open()来实例化的：

```java
Selector selector = Selector.open();
```

+ 要实现selector管理Channel，需要将Channel注册到相应的Selector上；

```java
channel.configureBlocking(false);
channel.register(channel, SelectionKey.OP_READ);
```

    - 通过调用渠道的register()方法会将它注册到一个选择器上，与Selector一起使用时，Channel必须处于非阻塞模式下，否则将抛出IllefalBlockingModeException异常，这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式，而套接字通道都可以；另外通道一旦被注册，将不能回到阻塞状态，此时若调用socketChannel.configureBlocking(true)将抛出BlockingModeException异常。
+ 在实际运行中，通过Selector的select()方法，可以选择已经准备就绪的通道

```java
select();		// 阻塞到至少有一个事件准备就绪
select(long timeout);	// 最长阻塞时间为timeout
selectNow(); 	// 非阻塞，立即返回
```

    - select()方法返回的int值表示有多少通道已经就绪，是自上次调用select()方法后有多少通道变成就绪状态；一旦调用select()方法，并且返回值不为0时，则可以通过调用Selector的selectedKeys()方法来访问已选择键集合

```java
Set<SelectionKey> selectedKeys = selecotor.selectedKeys();
```

    - 然后循环遍历selectedKeys集中的每一个键，并检测各个键所对应的通道的就绪时间，再通过SelectionKey关联的Selector和channel进行实际的业务处理
    - 注意每次迭代末尾的keyIterator.remove()调用；Selector不会自己从已选择键集中移除SelectionKey实例，必须在处理完通道时自己移除，否则下次该通道变成就绪时，Selector会再次将其放入已选择键集中。

## SelectionKey
### 什么是SelectionKey
SelectionKey是一个抽象类，表示selectableChannel在Selector中注册的标识；每个Channel向Selector注册时，都将会创建一个SelectionKey；SelectionKey将Channel与Selector建立了关系，并维护了channel事件。

可以通过cancel()方法取消键，取消的键不会立即从selector中移除，而是添加到cancelledKeys中，在下一次select操作时移除它，所以在调用某个key时，需要使用isValid进行校验。

### SelectionKey类型和就绪条件
在向Selector对象注册感兴趣的事件时，JAVA NIO共定义了四种：

+ OP_READ：读事件
+ OP_WRITE：写事件
+ OP_CONNECT：请求连接事件
+ OP_ACCEPT：接受连接事件

| 操作类型 | 就绪条件及说明 |
| --- | --- |
| OP_READ | 当操作系统读缓冲区有数据可读时就绪，并非时刻都有数据可读，所以一般需要注册该操作，仅当有就绪时才发起读操作，避免浪费CPU |
| OP_WRITE | 当操作系统写缓冲区有空闲空间时就绪，一般情况下写缓冲区都有空闲空间，小块数据直接写入即可，没必要注册该操作类型，否则该条件不断就绪浪费CPU；但如果是写密集型的任务，比如文件下载等，缓冲区很可能满，注册该操作类型就很有必要，同时注意完成后取消注册 |
| OP_CONNECT | 当SocketChanne.connect()请求连接成功后就绪，改操作只给客户端使用 |
| OP_ACCEPT | 当接收到一个客户端连接请求时就绪，该操作只给服务端使用 |


ServerSocketChannel和SocketChannel可以注册自己感兴趣的操作类型，当对应操作类型的就绪条件满足时，OS会通知channel

![4](image4.png)

```java
SocketChannel sc = serverSocketChannel.accept();
```

+ 服务端启动ServerSocketChannel关注OP_ACCEPT事件
+ 客户端启动SocketChannel关注OP_CONNECT事件
+ 服务端接受连接，启动一个服务器的SocketChannel，这个SocketChannel可以关注OP_READ、OP_WRITE事件，一般建立连接后会直接关注OP_READ事件
+ 客户端SocketChannel发现连接建立后，可以关注OP_READ、OP_WRITE事件，一般是需要客户端需要发送数据了才关注OP_READ事件
+ 连接建立后客户端与服务端开始相互发送消息（读写），根据实际情况来关注OP_READ、OP_WRITE事件

## BUFFER
Buffer用于和NIO通道进行交互，数据是从通道读入缓冲区，从缓冲区写入到通道的；以写为例，应用程序都是将数据写入缓冲区，再通过通道把缓冲的数据发送出去，读也是一样，数据总是先从通道读到缓冲，应用程序再读缓冲的数据

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存，这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存

```java
public class BufferMethod {
    public static void main(String[] args) {

        System.out.println("------Test get-------------");
        ByteBuffer buffer = ByteBuffer.allocate(32);
        buffer.put((byte) 'a')//0
              .put((byte) 'b')//1
              .put((byte) 'c')//2
              .put((byte) 'd')//3
              .put((byte) 'e')//4
              .put((byte) 'f');//5
        System.out.println("before flip()" + buffer);
        /* 转换为读取模式*/
        buffer.flip();
        System.out.println("before get():" + buffer);
        System.out.println((char) buffer.get());
        System.out.println("after get():" + buffer);

        /* get(index)不影响position的值*/
        System.out.println((char) buffer.get(2));
        System.out.println("after get(index):" + buffer);
        byte[] dst = new byte[10];

        /* position移动两位*/
        buffer.get(dst, 0, 2);
        /*这里的buffer是 abcdef[pos=3 lim=6 cap=32]*/
        System.out.println("after get(dst, 0, 2):" + buffer);
        System.out.println("dst:" + new String(dst));

        System.out.println("--------Test put-------");
        ByteBuffer bb = ByteBuffer.allocate(32);
        System.out.println("before put(byte):" + bb);
        System.out.println("after put(byte):" + bb.put((byte) 'z'));
        // put(2,(byte) 'c')不改变position的位置
        bb.put(2, (byte) 'c');
        System.out.println("after put(2,(byte) 'c'):" + bb);
        System.out.println(new String(bb.array()));

        // 这里的buffer是 abcdef[pos=3 lim=6 cap=32]
        bb.put(buffer);
        System.out.println("after put(buffer):" + bb);
        System.out.println(new String(bb.array()));

        System.out.println("--------Test reset----------");
        buffer = ByteBuffer.allocate(20);
        System.out.println("buffer = " + buffer);
        buffer.clear();
        buffer.position(5);//移动position到5
        buffer.mark();//记录当前position的位置
        buffer.position(10);//移动position到10
        System.out.println("before reset:" + buffer);
        buffer.reset();//复位position到记录的地址
        System.out.println("after reset:" + buffer);

        System.out.println("--------Test rewind--------");
        buffer.clear();
        buffer.position(10);//移动position到10
        buffer.limit(15);//限定最大可写入的位置为15
        System.out.println("before rewind:" + buffer);
        buffer.rewind();//将position设回0
        System.out.println("before rewind:" + buffer);

        System.out.println("--------Test compact--------");
        buffer.clear();
        //放入4个字节，position移动到下个可写入的位置，也就是4
        buffer.put("abcd".getBytes());
        System.out.println("before compact:" + buffer);
        System.out.println(new String(buffer.array()));
        buffer.flip();//将position设回0，并将limit设置成之前position的值
        System.out.println("after flip:" + buffer);
        //从Buffer中读取数据的例子，每读一次，position移动一次
        System.out.println((char) buffer.get());
        System.out.println((char) buffer.get());
        System.out.println((char) buffer.get());
        System.out.println("after three gets:" + buffer);
        System.out.println(new String(buffer.array()));
        //compact()方法将所有未读的数据拷贝到Buffer起始处。
        // 然后将position设到最后一个未读元素正后面。
        buffer.compact();
        System.out.println("after compact:" + buffer);
        System.out.println(new String(buffer.array()));
    }
}
```

### 重要属性
![5](image5.png)

+ **capacity**：作为一个内存块，Buffer有一个固定的大小值（容量），只能往里写 capacity 个byte、long、char等类型，一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据
+ **position**：表示当前写的位置；初始化时postion的值为0，当一个byte、long等数据写到Buffer后，position会向前移动到下一个可插入数据的Buffer单元，position最大为capacity - 1
+ **limit**：在写模式下，Buffer的limit表示最多能往Buffer里写多少数据；写模式下，limit等于capacity
    - 当切换Buffer到读模式时，limit表示最多能读到多少数据；因此当切换Buffer到读模式时，limit会被设置成写模式下的position值

### Buffer的分配
要想获得一个Buffer对象，首先要进行分配；每一个buffer类都有**allocate**方法（可以通过allocate在堆上分配，也可以通过allocateDirect在直接内存上分配）

```java
ByteBuffer buf = ByteBuffer.allocate(1024);
CharBuffer buf = CharBuffer.allocate(1024);
```

```java
ByteBuffer buf = ByteBuffer.allocateDirect(1024);
```

### 直接内存
HeapByteBuffer与DirectByteBuffer，在原理上，前者可以看出分配的buffer是在heap区域的，其实真正flush到远程时会先拷贝到直接内存，再做下一步操作；在NIO到框架下，很多框架会采用DirectByteBuffer来操作，这样分配到内存不再是在java heap上，经过性能测试，可以得到非常快速的网络交互，在大量的网络交互下，一般速度会比HeapByteBuffer要快速好几倍

直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是JVM规范中定义的内存区域，但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现

NIO可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java 堆中的DirectByteBuffer对象作为这块内存的引用进行操作；这样在一些场景中可以显著提高性能，因为避免了Java 堆和Native堆中来回复制数据

#### 直接内存和对内存的比较
+ 直接内存申请空间耗费更高的性能，当频繁申请到一定量时尤为明显
+ 直接内存IO读写的性能要优于普通的堆内存，在多次读写操作的情况下差异明显

# NIO之Reactor模式
**Reactor是一种开发模式，模式的核心流程：**

+ 注册事件 -> 扫描是否有注册事件发生 -> 对事件进行处理

## 单线程Reactor模式
![6](image6.png)

+ 服务端的Reactor是一个线程对象，该线程会启动时间循环，并使用Selector来实现IO的多路复用；注册一个Acceptor事件处理器到Reactor中，Acceptor事件处理器所关注的事件是ACCEPT事件，这样Reactor会监听客户端向服务端发起的连接请求事件
+ 客户端向服务端发起一个连接请求，Reactor监听到了该ACCEPT事件的发生并将该ACCEPT事件派发给相应的Acceptor处理器来进行处理；Acceptor处理器通过accept()方法得到这个客户端对应的连接（SocketChannel），然后将该连接所关注的READ事件以及对应的READ事件处理器注册到Reactor中，这样Reactor就会监听该连接到READ事件
+ 当Reactor监听到有读或者写事件发生时，将相关的事件派发给对应的处理器进行处理
+ 每当处理完所有就绪的I/O事件后，Reactor线程会再次执行select()阻塞等待新的事件就绪并将其分派给对应处理器进行处理

注意：Reactor的单线程模式的单线程主要针对于I/O操作而言，也就是所有I/O的accept()、read()、write()以及connect()操作都是在一个线程上完成的

但在目前的单线程Reactor模式中，不仅I/O操作在该Reactor线程上，连非I/O的业务操作也在该线程上进行处理了，这可能会大大延迟I/O请求的响应，所以我们应该将非I/O的业务逻辑操作从Reactor线程上卸载，以此来加速Reactor线程对I/O请求的响应

## 多线程Reactor模式
![7](image7.png)

与单线程Reactor模式不同的是，添加了一个工作者线程池，并将非I/O操作从Reactor线程中移出转交给工作者线程池来执行，这样就能提高Reactor线程的I/O显影，不至于因为一些耗时的业务逻辑而延迟对后面I/O请求对处理

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">使用线程池的优势：</font>

+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">通过重用现有的线程而不是创建新线程，可以在处理多个请求时分摊在线程创建和销毁过程产生的巨大开销。</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">另一个额外的好处是，当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，从而提高了响应性。</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态。同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">改进的版本中，所以的I/O操作依旧由一个Reactor来完成，包括I/O的accept()、read()、write()以及connect()操作。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">对于一些小容量应用场景，可以使用单线程模型。但是对于高负载、大并发或大数据量的应用场景却不合适，主要原因如下：</font>

+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">一个NIO线程同时处理成百上千的链路，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的读取和发送；</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，成为系统的性能瓶颈；</font>

## 主从多线程Reactor模式
![8](image8.png)

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">Reactor线程池中的每一Reactor线程都会有自己的Selector、线程和分发的事件循环逻辑。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">mainReactor可以只有一个，但subReactor一般会有多个。mainReactor线程主要负责接收客户端的连接请求，然后将接收到的SocketChannel传递给subReactor，由subReactor来完成和客户端的通信。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">流程：</font>

+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">注册一个Acceptor事件处理器到mainReactor中，Acceptor事件处理器所关注的事件是ACCEPT事件，这样mainReactor会监听客户端向服务器端发起的连接请求事件(ACCEPT事件)。启动mainReactor的事件循环。</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">客户端向服务器端发起一个连接请求，mainReactor监听到了该ACCEPT事件并将该ACCEPT事件派发给Acceptor处理器来进行处理。Acceptor处理器通过accept()方法得到与这个客户端对应的连接(SocketChannel)，然后将这个SocketChannel传递给subReactor线程池。</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;"> subReactor线程池分配一个subReactor线程给这个SocketChannel，即，将SocketChannel关注的READ事件以及对应的READ事件处理器注册到subReactor线程中。当然你也注册WRITE事件以及WRITE事件处理器到subReactor线程中以完成I/O写操作。Reactor线程池中的每一Reactor线程都会有自己的Selector、线程和分发的循环逻辑。</font>
+ <font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">当有I/O事件就绪时，相关的subReactor就将事件派发给响应的处理器处理。注意，这里subReactor线程只负责完成I/O的read()操作，在读取到数据后将业务逻辑的处理放入到线程池中完成，若完成业务逻辑后需要返回数据给客户端，则相关的I/O的write操作还是会被提交回subReactor线程来完成。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">注意，所有的I/O操作(包括，I/O的accept()、read()、write()以及connect()操作)依旧还是在Reactor线程(mainReactor线程或 subReactor线程)中完成的。Thread Pool(线程池)仅用来处理非I/O操作的逻辑。</font>

<font style="color:rgba(0, 0, 0, 0.85);background-color:#ffffff;">多Reactor线程模式将“接受客户端的连接请求”和“与该客户端的通信”分在了两个Reactor线程来完成。mainReactor完成接收客户端连接请求的操作，它不负责与客户端的通信，而是将建立好的连接转交给subReactor线程来完成与客户端的通信，这样一来就不会因为read()数据量太大而导致后面的客户端连接请求得不到即时处理的情况。并且多Reactor线程模式在海量的客户端并发请求的情况下，还可以通过实现subReactor线程池来将海量的连接分发给多个subReactor线程，在多核的操作系统中这能大大提升应用的负载和吞吐量。</font>

