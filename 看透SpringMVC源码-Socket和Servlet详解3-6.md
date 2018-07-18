---
title: 看透SpringMVC源码-Socket和Servlet详解3-6
tags: socket,nio,servlet,http
grammar_cjkRuby: true
---

第4章 Java中socket的用法

>Java中的socket有两种：普通Socket和NioSocket

4.1 普通socket的用法
socket主要用来进行网络通信，即服务器和客户端通信，由于服务器和客户端的不同所以相应的也有一点区别。socket相当于通信的载体。socket有两种：ServerSocket和Socket，Socket主要用于服务器和客户端通信；ServerSocket只用于服务器，用来监听服务器的端口，查看是否有来自客户端的连接请求，而客户端并不需要监听。

I. 服务器ServerSocket通信的步骤：
1）创建ServerSocket类-参数为端口
2）ServerSocket调用accept方法对端口进行监听，返回为Socket类
3）使用Socket进行通信，读取为getINputStream，发送为getOutputStream

``` stylus
public class Server {
    public static void main(String[] args) {
        try {

            // ServerSocket用于服务器监听端口，客户端不需要监听
            ServerSocket server = new ServerSocket(8088);
            Socket socket = server.accept();

            // 1、接收数据
            BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String line = br.readLine();
            System.out.println("received from client:"+line);

            // 2、发送数据
            PrintWriter pw = new PrintWriter(socket.getOutputStream());
            // 注意：println和print的区别，换行结束，否则无输出, 因为readLine()
            // 是要输入回车/换行才结束的!!
            pw.println("I have received the data");
            pw.flush();


            System.out.println("12313");
            // 3、关闭资源
            pw.close();
            br.close();
            socket.close();
            server.close();


        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```


II. 客户端的通信
与服务端类似，但是只有两步，少了监听步骤，Socket(String host,Int port)

``` stylus
public class Client {
    public static void main(String[] args) {
        String msg = "Hello World,this is client";
        try {
            Socket socket = new Socket("127.0.0.1", 8088);
            System.out.println("hello");
            // 1、 发送数据
            PrintWriter pw = new PrintWriter(socket.getOutputStream());
            pw.println(msg);
            pw.flush();

            // 2、接收数据
            BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String line = br.readLine();
            System.out.println("received data is :" + line);

            // 3、关闭资源
            pw.close();
            br.close();
            socket.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

4.2 NioSocket的用法
>NIO即新IO，自1.4开始添加的，底层采用新的处理方式，大大加快了IO效率。与普通IO不同的是，NIO采用了分拣员的角色Selector，对数据进行整理一起发送。就像送外卖，传统的订餐服务是来一份送一份，这样的效率很低；通过采用新的订餐模式后，所有的订单先进行分拣，然后再根据目的地的不同进行统一分配，这样一次可以分配多份，减少了IO次数，这种情况类似于数据库批量读取和单独读取类似。

几个改变：

* ServerSocket和Socket改为ServerSocketChannel和SocketChannel；Channel相当于货车一次批量运送。
* Buffer、Channel、Selector几个概念：Buffer相当于货物数据；Channel相当于货车；Selector相当于分拣员

I. NioSocket服务器端处理步骤-5步

* 1、ServerSocketChannel监听端口
* 2、创建Selector，并将其注册到ServerSocketChannel，同时定义数据传送方式：read、write等
* 3、调用selector.select()等待请求
* 4、selector.selectedKeys()获取数据集合：对read、write进行分别处理
* 5、迭代获取数据

``` stylus
public class NIOServer {
    private Selector selector;
    private ByteBuffer readBuffer = ByteBuffer.allocate(1024);//调整缓存的大小可以看到打印输出的变化
    private ByteBuffer sendBuffer = ByteBuffer.allocate(1024);//调整缓存的大小可以看到打印输出的变化

    String str;

    public void start() throws IOException {

        // 1、ServerSocketChannel监听端口
        ServerSocketChannel ssc = ServerSocketChannel.open();
        // 设置为非阻塞模式
        ssc.configureBlocking(false);
        ssc.bind(new InetSocketAddress(8088));

        // 2、创建Selector，并将其注册到ServerSocketChannel，同时定义数据传送方式：read、write等
        // 注册选择器-相当于货物分拣员
        selector = Selector.open();
        ssc.register(selector, SelectionKey.OP_ACCEPT);

        // 处理连接
        while (!Thread.currentThread().isInterrupted()) {
            // 3、调用selector.select()等待请求
            selector.select();
            // 4、selector.selectedKeys()获取数据集合：对read、write进行分别处理
            Set<SelectionKey> keys = selector.selectedKeys();

            // 5、迭代获取数据
            Iterator<SelectionKey> keyIterator = keys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();

                if (!key.isValid()) {
                    continue;
                }
                if (key.isAcceptable()) {
                    accept(key);
                } else if (key.isReadable()) {
                    read(key);
                } else if (key.isWritable()) {
                    write(key);
                }

                // 注意，这一句一定要加上，
                // 处理完后，从SelectionKey移除当前使用的key
                keyIterator.remove();
            } 
        } 
    }


    public void write(SelectionKey key) throws IOException {
        SocketChannel socket = (SocketChannel) key.channel();
        sendBuffer.clear();
        sendBuffer.put(str.getBytes());
        // flip调转缓冲区-就可以从头开始遍历
        sendBuffer.flip();

        socket.write(sendBuffer);
        socket.register(selector, SelectionKey.OP_READ);
    }

    public void read(SelectionKey key) throws IOException {
        SocketChannel socket = (SocketChannel) key.channel();
        readBuffer.clear();

        // Attempt to read off the channel
        int numRead;
        try {
            numRead = socket.read(readBuffer);
        } catch (IOException e) {
            // The remote forcibly closed the connection, cancel
            // the selection key and close the channel.
            key.cancel();
            socket.close();

            return;
        }

        str = new String(readBuffer.array(), 0, numRead);
        System.out.println(str);
        socket.register(selector, SelectionKey.OP_WRITE);
    }
    public void accept(SelectionKey key) throws IOException {
        ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
        SocketChannel socket = ssc.accept();
        socket.configureBlocking(false);
        socket.register(selector, SelectionKey.OP_READ);
        System.out.println("a new client conneted" + socket.getRemoteAddress());
    }


    public static void main(String[] args) throws Exception {
        System.out.println("server started...");
        new NIOServer().start();
    }
}

```


II. NioSocket客户端步骤类似

``` stylus
public class NIOClient {
    ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
    ByteBuffer readBuffer = ByteBuffer.allocate(1024);

    public void start() throws IOException {
        // 打开socket通道
        SocketChannel sc = SocketChannel.open();
        //设置为非阻塞
        sc.configureBlocking(false);
        //连接服务器地址和端口
        sc.connect(new InetSocketAddress("localhost", 8088));
        //打开选择器
        Selector selector = Selector.open();
        //注册连接服务器socket的动作
        sc.register(selector, SelectionKey.OP_CONNECT);

        Scanner scanner = new Scanner(System.in);
        while (true) {
            //选择一组键，其相应的通道已为 I/O 操作准备就绪。
            //此方法执行处于阻塞模式的选择操作。
            selector.select();
            //返回此选择器的已选择键集。
            Set<SelectionKey> keys = selector.selectedKeys();
            System.out.println("keys=" + keys.size());
            Iterator<SelectionKey> keyIterator = keys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                keyIterator.remove();
                // 判断此通道上是否正在进行连接操作。
                if (key.isConnectable()) {
                    sc.finishConnect();
                    sc.register(selector, SelectionKey.OP_WRITE);
                    System.out.println("server connected...");
                    break;
                } else if (key.isWritable()) { //写数据
                    System.out.print("please input message:");
                    String message = scanner.nextLine();
                    //ByteBuffer writeBuffer = ByteBuffer.wrap(message.getBytes());
                    writeBuffer.clear();
                    writeBuffer.put(message.getBytes());
                    //将缓冲区各标志复位,因为向里面put了数据标志被改变要想从中读取数据发向服务器,就要复位
                    writeBuffer.flip();
                    sc.write(writeBuffer);

                    //注册写操作,每个chanel只能注册一个操作，最后注册的一个生效
                    //如果你对不止一种事件感兴趣，那么可以用“位或”操作符将常量连接起来
                    //int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
                    //使用interest集合
                    sc.register(selector, SelectionKey.OP_READ);
                    sc.register(selector, SelectionKey.OP_WRITE);
                    sc.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()){//读取数据
                    System.out.print("receive message:");
                    SocketChannel client = (SocketChannel) key.channel();
                    //将缓冲区清空以备下次读取
                    readBuffer.clear();
                    int num = client.read(readBuffer);
                    System.out.println(new String(readBuffer.array(),0, num));
                    //注册读操作，下一次读取
                    sc.register(selector, SelectionKey.OP_WRITE);
                }
            }
        }
    }

    public static void main(String[] args) throws IOException {
        new NIOClient().start();
    }
}
```


>注意，这里用到了数据ByteBuffer，继承自Buffer。Buffer是NIO专门用来传递数据的类，有四个重要的属性：

* capacity：容量，在创建时设置，使用过程中不可以改变
* limit：最大访问上限，一般设置为和capacity相同，如果不同，则表示，每次只能使用（如读取）limit大小的数据
* position：当前操作元素所在的位置
* mark：用来暂时保存position的值。有时候我们需要读取特定位置的数据，比如中间段，这是就需要用到mark，先标记原始的position位置。读取固定的位置，读取完成后恢复position的位置

>更多的NIO知识需要详细了解


----------

第5章 自己动手写http协议
> http协议，应该类似一种信息格式化规范，通过将数据按照一定的格式进行包装，这样便于网页进行读取。这样的话，http协议实际上就是在socket传输的数据上加上固定的格式：如请求头、请求方法、类型、url等等


----------

第6章 详解Servlet
>Servlet是javaweb网络处理的一种方式，也是各种javaweb框架的底层原型。Servlet即Server+Applet=服务器应用

Servlet继承关系：

``` stylus
public abstract class HttpServlet extends GenericServlet implements Serializable{
```

``` stylus
public abstract class GenericServlet implements Servlet, ServletConfig, Serializable {
```

``` stylus
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```
``` stylus
public interface ServletConfig {
    String getServletName();

    ServletContext getServletContext();

    String getInitParameter(String var1);

    Enumeration getInitParameterNames();
}

```

![Spring MVC 原理图-关注Servlet][1]
Servlet访问流程：
>servlet即服务器的请求处理程序，即一个简单的程序。
>生命周期：init()实例化--service()方法处理请求(内部调用doGet等方法处理不同的请求)--destroy()方法

* tomcat服务器启动，通过web容器调用init方法初始化Servlet实例，并且可以传递一个实现 javax.servlet.ServletConfig 接口的对象给它。这个配置对象（configuration object）使Servlet能够读取在web应用的web.xml文件里定义的名值（name-value）初始参数。这个方法在Servlet实例的生命周期里只调用一次。
* 初始化后，Servlet实例就可以处理客户端请求了。web容器调用Servlet的service()方法来处理每一个请求。service() 方法定义了能够处理的请求类型并且调用适当方法来处理这些请求。编写Servlet的开发者必须为这些方法提供实现。如果发出一个Servlet没实现的请求，那么父类的方法就会被调用并且通常会给请求方（requester）返回一个错误信息。
* 最后，web容器调用destroy()方法来终结Servlet。如果你想在Servlet的生命周期内关闭或者销毁一些文件系统或者网络资源，你可以调用这个方法来实现。destroy() 方法和init()方法一样，在Servlet的生命周期里只能调用一次。

**更加详细的参考链接：[Servlet 工作原理解析-IBM][2]**


  [1]: http://ohfk1r827.bkt.clouddn.com/spring-mvc1.jpg
  [2]: https://www.ibm.com/developerworks/cn/java/j-lo-servlet/
