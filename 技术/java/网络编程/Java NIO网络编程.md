#nio  

### OSI网络七层模型

-   低三层  
    物料层：使用原始数据比特流能再物理介质上传输  
    数据链路层： 通过校验、确认和反馈重发等手段，行程稳定的数据链路  
    网络层：进行路由选择和流量控制（IP协议）
-   传输层：（TCP/UDP协议）提供可靠的端口到端口的数据传输服务
-   高三层：一般都指JAVA程序  
    会话层：负责建立、管理和终止进程之间的会话和数据交换  
    表示层：负责数据格式转换、数据加密与解密、压缩与解压等  
    应用层：为用户的应用进程提供网络服务

#### 传输控制协议TCP

面向连接、可靠、有序、字节流传输服务，应用程序在使用TCP前必须建立TCP连接

-   TCP三次握手
    
    1.  客户端发送消息至服务端等待确认
    2.  服务端收到，并发送请求等待确认
    3.  客户端收到，建立连接
-   TCP四次挥手
    
    1.  客户端发送断开连接至服务端等待确认
    2.  服务端收到，半关闭状态，服务端等待释放
    3.  服务端发送等待确认
    4.  客户端等待一会，发送确认关闭，等待一会关闭

---

#### 用户数据报协议UDP

UDP是Internet传输协议，提供无连接、不可靠、数据报尽力传输服务

-   无需建立连接
-   无需连接状态
-   数据不可靠

### Socket编程

-   数据报类型套接字SOCK_DGRAM（面向UDP接口）
-   流式套接字SOCK_STREAM（面向TCP接口）
-   原始套接字SOCK_RAW（面向网络层协议接口IP、ICMP等）

-   #### Socket API及其调用过程
    

![](https://upload-images.jianshu.io/upload_images/5888874-e198cca2b6cc1b9b.png?imageMogr2/auto-orient/strip|imageView2/2/w/745/format/webp)

socket

-   #### Socket API函数定义
    
    -   listen()、accept()函数只能用于服务端
    -   connect()函数只能用于客户端
    -   socket() 、bind()、send()、recv()、sendto()、recvfrom()、close()

---

## 网络编程

-   **阻塞（bloking）IO**  
    资源不可用时，IO请求一直阻塞，直到反馈结果（有数据或超时）
    
-   **非阻塞（non-bloking）IO**  
    资源不可用时，IO请求离开返回，返回数据标识资源不可用
    
-   **同步（synchronous）IO**  
    应用阻塞在发送或接受数据的状态，直到数据成功传输或返回失败
    
-   **异步（asynchronous）IO**  
    应用发送或接受数据后立即返回，实际处理是异步执行
    

---

### BIO网络编程

##### 客户端阻塞 - OutputStream.write()

```swift
import java.io.OutputStream;
import java.net.Socket;
import java.nio.charset.Charset;
import java.util.Scanner;

public class BIOClient {
    private static Charset charset = Charset.forName("UTF-8");

    public static void main(String[] args) throws Exception {
        Socket s = new Socket("localhost", 8080);
        OutputStream out = s.getOutputStream();

        Scanner scanner = new Scanner(System.in);
        System.out.println("请输入：");
        String msg = scanner.nextLine();
        out.write(msg.getBytes(charset)); // 阻塞，写完成
        scanner.close();
        s.close();
    }
}
```

##### 服务端阻塞 - ServerSocket.accept() 与 BufferedReader.readLine()

```swift
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.ServerSocket;
import java.net.Socket;

public class BIOServer {

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("服务器启动成功");
        while (!serverSocket.isClosed()) {
            Socket request = serverSocket.accept();// 阻塞
            System.out.println("收到新连接 : " + request.toString());
            try {
                // 接收数据、打印
                InputStream inputStream = request.getInputStream(); // net + i/o
                BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                String msg;
                while ((msg = reader.readLine()) != null) { // 没有数据，阻塞
                    if (msg.length() == 0) {
                        break;
                    }
                    System.out.println(msg);
                }
                System.out.println("收到数据,来自："+ request.toString());
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    request.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        serverSocket.close();
    }
}
```

##### 服务端 - 加入多线程并模拟HTTP请求协议返回数据 - 子线程阻塞

```swift
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer2 {

    private static ExecutorService threadPool = Executors.newCachedThreadPool();

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("服务器启动成功");
        while (!serverSocket.isClosed()) {
            Socket request = serverSocket.accept();
            System.out.println("收到新连接 : " + request.toString());
            threadPool.execute(() -> {
                try {
                    // 接收数据、打印
                    InputStream inputStream = request.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                    String msg;
                    while ((msg = reader.readLine()) != null) { // 没有数据阻塞
                        if (msg.length() == 0) {
                            break;
                        }
                        System.out.println(msg);
                    }

                    System.out.println("收到数据,来自："+ request.toString());
                    // 响应结果 200
                    OutputStream outputStream = request.getOutputStream();
                    outputStream.write("HTTP/1.1 200 OK\r\n".getBytes());
                    outputStream.write("Content-Length: 11\r\n\r\n".getBytes());
                    outputStream.write("Hello World".getBytes());
                    outputStream.flush();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        request.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        serverSocket.close();
    }
}
```

---

## NIO网络编程

NIO中三个核心组件：Buffer缓冲区、Channel通道、Sellector选择器

#### Buffer缓冲区

缓冲区是一个可以写入数据的内存块，然后可以再次读取，Buffer三个重要属性：

-   capacity容量：做位一个内存块，Buffer具有一定的固定大小
-   position位置：写入模式时代表写数据的位置，读取模式时代表读取数据的位置
-   limit限制：写入模式，限制等于buffer的容量；读取模式下，limit等于写入的数据量

```csharp
import java.nio.ByteBuffer;
import java.nio.IntBuffer;
import java.nio.LongBuffer;

public class BufferDemo {
    public static void main(String[] args) {
        // 构建一个byte字节缓冲区，容量是4
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(4);
        // 默认写入模式，查看三个重要的指标
        System.out.println(String.format("初始化：capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));
        // 写入2字节的数据
        byteBuffer.put((byte) 1);
        byteBuffer.put((byte) 2);
        byteBuffer.put((byte) 3);
        // 再看数据
        System.out.println(String.format("写入3字节后，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // 转换为读取模式(不调用flip方法，也是可以读取数据的，但是position记录读取的位置不对)
        System.out.println("#######开始读取");
        byteBuffer.flip();
        byte a = byteBuffer.get();
        System.out.println(a);
        byte b = byteBuffer.get();
        System.out.println(b);
        System.out.println(String.format("读取2字节数据后，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // 继续写入3字节，此时读模式下，limit=3，position=2.继续写入只能覆盖写入一条数据
        // clear()方法清除整个缓冲区。compact()方法仅清除已阅读的数据。转为写入模式
        byteBuffer.compact(); // buffer : 1 , 3
        byteBuffer.put((byte) 3);
        byteBuffer.put((byte) 4);
        byteBuffer.put((byte) 5);
        System.out.println(String.format("最终的情况，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // rewind() 重置position为0
        // mark() 标记position的位置
        // reset() 重置position为上次mark()标记的位置

    }
}

```

##### ByteBuffer内存类型

ByteBuffer为性能关键型代码提供了直接内存（direct堆外）和非直接内存（heap堆）两种实现  
堆外内存获取方式：ByteBuffer buffer = ByteBuffer.allocateDirect(noBytes);

> 堆外内存的好处
> 
> -   进行网络IO或者文件IO时比heapBuffer少一次拷贝。因为GC会移动对象内存，在写file或者socket的过程中，JVM的实现中会把数据复制到堆外，再进行写入
> -   堆外内存不收GC控制，能降低GC的压力，实现自动管理。DirectByteBuffer中有一个Cleaner对象（PhantomReference）,Cleaner被GC前会执行clean方法，能触发DirectByteBuffer中定义的Deallocator

---

### Channel通道

Channel的API涵盖了UDP/TCP网络和文件IO,他可以创建连接与传输数据，可以非阻塞的读取和写入通道，通道始终读取或吸入缓冲区

##### SocketChannel

SocketChannel用户建立TCP网络连接，有两种创建形式：

1.  客户端主动发起服务器连接
2.  服务器获取新的连接

> 读和写都变成了非阻塞的方法，wirte()在尚未写数据就可能返回了，read()可能返回空数据。所以需要再循环中使用

##### ServerSocketChannel

ServerSocketChannel可监听新建立的TCP连接通道，通过open()方法创建

> ServerSocketChannel.accept() : 如果该通道处于非阻塞模式，那么如果没有挂起的连接，该方法将立即返回NULL

##### NIO服务端 - 基础实现

-   循环内只能实现单线程处理一个任务

```swift
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * 直接基于非阻塞的写法
 */
public class NIOServer {

    public static void main(String[] args) throws Exception {
        // 创建网络服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式 - 默认都是阻塞的
        serverSocketChannel.socket().bind(new InetSocketAddress(8080)); // 绑定端口
        System.out.println("启动成功");
        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept(); // 获取新tcp连接通道
            // tcp请求 读取/响应
            if (socketChannel != null) {
                System.out.println("收到新连接 : " + socketChannel.getRemoteAddress());
                socketChannel.configureBlocking(false); // 默认是阻塞的,一定要设置为非阻塞
                try {
                    ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                    while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
                        // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                        if (requestBuffer.position() > 0) break;
                    }
                    if(requestBuffer.position() == 0) continue; // 如果没数据了, 则不继续后面的处理
                    requestBuffer.flip();
                    byte[] content = new byte[requestBuffer.limit()];
                    requestBuffer.get(content);
                    System.out.println(new String(content));
                    System.out.println("收到数据,来自："+ socketChannel.getRemoteAddress());

                    // 响应结果 200
                    String response = "HTTP/1.1 200 OK\r\n" +
                            "Content-Length: 11\r\n\r\n" +
                            "Hello World";
                    ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                    while (buffer.hasRemaining()) {
                        socketChannel.write(buffer);// 非阻塞
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        // 用到了非阻塞的API, 在设计上,和BIO可以有很大的不同.继续改进
    }
}
```

##### NIO客户端

```swift
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Scanner;

public class NIOClient {

    public static void main(String[] args) throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        while (!socketChannel.finishConnect()) {
            // 没连接上,则一直等待
            Thread.yield();
        }
        Scanner scanner = new Scanner(System.in);
        System.out.println("请输入：");
        // 发送内容
        String msg = scanner.nextLine();
        ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
        while (buffer.hasRemaining()) {
            socketChannel.write(buffer);
        }
        // 读取响应
        System.out.println("收到服务端响应:");
        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);

        while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
            if (requestBuffer.position() > 0) break;
        }
        requestBuffer.flip();
        byte[] content = new byte[requestBuffer.limit()];
        requestBuffer.get(content);
        System.out.println(new String(content));
        scanner.close();
        socketChannel.close();
    }
}
```

##### NIO服务端优化

-   用到了非阻塞的API, 再设计上,和BIO可以有很大的不同
-   问题: 轮询通道的方式,低效,浪费CPU

```swift
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.ArrayList;
import java.util.Iterator;

/**
 * 直接基于非阻塞的写法,一个线程处理轮询所有请求
 */
public class NIOServer1 {
    /**
     * 已经建立连接的集合
     */
    private static ArrayList<SocketChannel> channels = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        // 创建网络服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式
        serverSocketChannel.socket().bind(new InetSocketAddress(8080)); // 绑定端口
        System.out.println("启动成功");
        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept(); // 获取新tcp连接通道
                // tcp请求 读取/响应
                if (socketChannel != null) {
                System.out.println("收到新连接 : " + socketChannel.getRemoteAddress());
                socketChannel.configureBlocking(false); // 默认是阻塞的,一定要设置为非阻塞
                channels.add(socketChannel);
            } else {
                // 没有新连接的情况下,就去处理现有连接的数据,处理完的就删除掉
                Iterator<SocketChannel> iterator = channels.iterator();
                while (iterator.hasNext()) {
                    SocketChannel ch = iterator.next();
                    try {
                        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);

                        if (ch.read(requestBuffer) == 0) {
                            // 等于0,代表这个通道没有数据需要处理,那就待会再处理
                            continue;
                        }
                        while (ch.isOpen() && ch.read(requestBuffer) != -1) {
                            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                            if (requestBuffer.position() > 0) break;
                        }
                        if(requestBuffer.position() == 0) continue; // 如果没数据了, 则不继续后面的处理
                        requestBuffer.flip();
                        byte[] content = new byte[requestBuffer.limit()];
                        requestBuffer.get(content);
                        System.out.println(new String(content));
                        System.out.println("收到数据,来自：" + ch.getRemoteAddress());

                        // 响应结果 200
                        String response = "HTTP/1.1 200 OK\r\n" +
                                "Content-Length: 11\r\n\r\n" +
                                "Hello World";
                        ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                        while (buffer.hasRemaining()) {
                            ch.write(buffer);
                        }
                        iterator.remove();
                    } catch (IOException e) {
                        e.printStackTrace();
                        iterator.remove();
                    }
                }
            }
        }
    }
}

```

---

### Selector选择器

Selector可以坚持一个或者多个NIO通道，并确定哪些通道已经准备好进行读取或写入，实现一个线程处理多个通道的核心概念理解：**事件驱动机制**

##### 通过Selector 事件轮询实现 NIO服务端

```swift
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 结合Selector实现的非阻塞服务端(放弃对channel的轮询,借助消息通知机制)
 */
public class NIOServerV2 {

    public static void main(String[] args) throws Exception {
        // 1. 创建网络服务端ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式

        // 2. 构建一个Selector选择器,并且将channel注册上去
        Selector selector = Selector.open();
        SelectionKey selectionKey = serverSocketChannel.register(selector, 0, serverSocketChannel);// 将serverSocketChannel注册到selector
        selectionKey.interestOps(SelectionKey.OP_ACCEPT); // 对serverSocketChannel上面的accept事件感兴趣(serverSocketChannel只能支持accept操作)

        // 3. 绑定端口
        serverSocketChannel.socket().bind(new InetSocketAddress(8080));

        System.out.println("启动成功");

        while (true) {
            // 不再轮询通道,改用下面轮询事件的方式.select方法有阻塞效果,直到有事件通知才会有返回
            selector.select();
            // 获取事件
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            // 遍历查询结果e
            Iterator<SelectionKey> iter = selectionKeys.iterator();
            while (iter.hasNext()) {
                // 被封装的查询结果
                SelectionKey key = iter.next();
                iter.remove();
                // 关注 Read 和 Accept两个事件
                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.attachment();
                    // 将拿到的客户端连接通道,注册到selector上面
                    SocketChannel clientSocketChannel = server.accept(); // mainReactor 轮询accept
                    clientSocketChannel.configureBlocking(false);
                    clientSocketChannel.register(selector, SelectionKey.OP_READ, clientSocketChannel); //再次注册事件
                    System.out.println("收到新连接 : " + clientSocketChannel.getRemoteAddress());
                }

                if (key.isReadable()) {
                    SocketChannel socketChannel = (SocketChannel) key.attachment();
                    try {
                        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                        while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
                            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                            if (requestBuffer.position() > 0) break;
                        }
                        if(requestBuffer.position() == 0) continue; // 如果没数据了, 则不继续后面的处理
                        requestBuffer.flip();
                        byte[] content = new byte[requestBuffer.limit()];
                        requestBuffer.get(content);
                        System.out.println(new String(content));
                        System.out.println("收到数据,来自：" + socketChannel.getRemoteAddress());
                        // TODO 业务操作 数据库 接口调用等等

                        // 响应结果 200
                        String response = "HTTP/1.1 200 OK\r\n" +
                                "Content-Length: 11\r\n\r\n" +
                                "Hello World";
                        ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                        while (buffer.hasRemaining()) {
                            socketChannel.write(buffer);
                        }
                    } catch (IOException e) {
                        // e.printStackTrace();
                        key.cancel(); // 取消事件订阅
                    }
                }
            }
            selector.selectNow();
        }
        // 问题: 此处一个selector监听所有事件,一个线程处理所有请求事件. 会成为瓶颈! 要有多线程的运用
    }
}
```

---

### Reactor模式

Reactor模式是一种典型的事件驱动的编程模型，是一种为处理并发服务请求，并将请求提交到一个或  
者多个服务处理程序的事件设计模式

#### NIO与多线程结合 Reactor模式

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * NIO selector 多路复用reactor线程模型
 */
public class NIOServerV3 {
    /** 处理业务操作的线程 */
    private static ExecutorService workPool = Executors.newCachedThreadPool();

    /**
     * 封装了selector.select()等事件轮询的代码
     */
    abstract class ReactorThread extends Thread {

        Selector selector;
        LinkedBlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<>();

        /**
         * Selector监听到有事件后,调用这个方法
         */
        public abstract void handler(SelectableChannel channel) throws Exception;

        private ReactorThread() throws IOException {
            selector = Selector.open();
        }

        volatile boolean running = false;

        @Override
        public void run() {
            // 轮询Selector事件
            while (running) {
                try {
                    // 执行队列中的任务
                    Runnable task;
                    while ((task = taskQueue.poll()) != null) {
                        task.run();
                    }
                    selector.select(1000);

                    // 获取查询结果
                    Set<SelectionKey> selected = selector.selectedKeys();
                    // 遍历查询结果
                    Iterator<SelectionKey> iter = selected.iterator();
                    while (iter.hasNext()) {
                        // 被封装的查询结果
                        SelectionKey key = iter.next();
                        iter.remove();
                        int readyOps = key.readyOps();
                        // 关注 Read 和 Accept两个事件
                        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                            try {
                                SelectableChannel channel = (SelectableChannel) key.attachment();
                                channel.configureBlocking(false);
                                handler(channel);
                                if (!channel.isOpen()) {
                                    key.cancel(); // 如果关闭了,就取消这个KEY的订阅
                                }
                            } catch (Exception ex) {
                                key.cancel(); // 如果有异常,就取消这个KEY的订阅
                            }
                        }
                    }
                    selector.selectNow();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        private SelectionKey register(SelectableChannel channel) throws Exception {
            // 为什么register要以任务提交的形式，让reactor线程去处理？
            // 因为线程在执行channel注册到selector的过程中，会和调用selector.select()方法的线程争用同一把锁
            // 而select()方法实在eventLoop中通过while循环调用的，争抢的可能性很高，为了让register能更快的执行，就放到同一个线程来处理
            FutureTask<SelectionKey> futureTask = new FutureTask<>(() -> channel.register(selector, 0, channel));
            taskQueue.add(futureTask);
            return futureTask.get();
        }

        private void doStart() {
            if (!running) {
                running = true;
                start();
            }
        }
    }

    private ServerSocketChannel serverSocketChannel;
    // 1、创建多个线程 - accept处理reactor线程 (accept线程)
    private ReactorThread[] mainReactorThreads = new ReactorThread[1];
    // 2、创建多个线程 - io处理reactor线程  (I/O线程)
    private ReactorThread[] subReactorThreads = new ReactorThread[8];

    /**
     * 初始化线程组
     */
    private void newGroup() throws IOException {
        // 创建IO线程,负责处理客户端连接以后socketChannel的IO读写
        for (int i = 0; i < subReactorThreads.length; i++) {
            subReactorThreads[i] = new ReactorThread() {
                @Override
                public void handler(SelectableChannel channel) throws IOException {
                    // work线程只负责处理IO处理，不处理accept事件
                    SocketChannel ch = (SocketChannel) channel;
                    ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                    while (ch.isOpen() && ch.read(requestBuffer) != -1) {
                        // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                        if (requestBuffer.position() > 0) break;
                    }
                    if (requestBuffer.position() == 0) return; // 如果没数据了, 则不继续后面的处理
                    requestBuffer.flip();
                    byte[] content = new byte[requestBuffer.limit()];
                    requestBuffer.get(content);
                    System.out.println(new String(content));
                    System.out.println(Thread.currentThread().getName() + "收到数据,来自：" + ch.getRemoteAddress());

                    // TODO 业务操作 数据库、接口...
                    workPool.submit(() -> {
                    });

                    // 响应结果 200
                    String response = "HTTP/1.1 200 OK\r\n" +
                            "Content-Length: 11\r\n\r\n" +
                            "Hello World";
                    ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                    while (buffer.hasRemaining()) {
                        ch.write(buffer);
                    }
                }
            };
        }

        // 创建mainReactor线程, 只负责处理serverSocketChannel
        for (int i = 0; i < mainReactorThreads.length; i++) {
            mainReactorThreads[i] = new ReactorThread() {
                AtomicInteger incr = new AtomicInteger(0);

                @Override
                public void handler(SelectableChannel channel) throws Exception {
                    // 只做请求分发，不做具体的数据读取
                    ServerSocketChannel ch = (ServerSocketChannel) channel;
                    SocketChannel socketChannel = ch.accept();
                    socketChannel.configureBlocking(false);
                    // 收到连接建立的通知之后，分发给I/O线程继续去读取数据
                    int index = incr.getAndIncrement() % subReactorThreads.length;
                    ReactorThread workEventLoop = subReactorThreads[index];
                    workEventLoop.doStart();
                    SelectionKey selectionKey = workEventLoop.register(socketChannel);
                    selectionKey.interestOps(SelectionKey.OP_READ);
                    System.out.println(Thread.currentThread().getName() + "收到新连接 : " + socketChannel.getRemoteAddress());
                }
            };
        }


    }

    /**
     * 初始化channel,并且绑定一个eventLoop线程
     *
     * @throws IOException IO异常
     */
    private void initAndRegister() throws Exception {
        // 1、 创建ServerSocketChannel
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        // 2、 将serverSocketChannel注册到selector
        int index = new Random().nextInt(mainReactorThreads.length);
        mainReactorThreads[index].doStart();
        SelectionKey selectionKey = mainReactorThreads[index].register(serverSocketChannel);
        selectionKey.interestOps(SelectionKey.OP_ACCEPT);
    }

    /**
     * 绑定端口
     *
     * @throws IOException IO异常
     */
    private void bind() throws IOException {
        //  1、 正式绑定端口，对外服务
        serverSocketChannel.bind(new InetSocketAddress(8080));
        System.out.println("启动完成，端口8080");
    }

    public static void main(String[] args) throws Exception {
        NIOServerV3 nioServerV3 = new NIOServerV3();
        nioServerV3.newGroup(); // 1、 创建main和sub两组线程
        nioServerV3.initAndRegister(); // 2、 创建serverSocketChannel，注册到mainReactor线程上的selector上
        nioServerV3.bind(); // 3、 为serverSocketChannel绑定端口
    }
}
```

# Reference
https://www.jianshu.com/p/85b6d44726ca