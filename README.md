https://www.jianshu.com/p/b4de9b85c79d



1、基础I/O模型
在《UNIX网络编程》中介绍了5中I/O模型：阻塞I/O、非阻塞I/O、I/O复用、SIGIO 、异步I/O；

1.1、I/O阻塞
通常把阻塞的文件描述符（file descriptor，fd）称之为阻塞I/O。默认条件下，创建的socket fd是阻塞的，针对阻塞I/O调用系统接口，可能因为等待的事件没有到达而被系统挂起，直到等待的事件触发调用接口才返回，例如，tcp socket的connect调用会阻塞至第三次握手成功（不考虑socket 出错或系统中断），如图所示。另外socket 的系统API ，如，accept、send、recv等都可能被阻塞。

IO阻塞.png
网络编程中，通常把可能永远阻塞的系统API调用 称为慢系统调用，典型的如 accept、recv、select等。慢系统调用在阻塞期间可能被信号中断而返回错误，相应的errno 被设置为EINTR，我们需要处理这种错误，解决办法有：

重启系统调用：

以accept为例，被中断后重启accept 。有个例外，若connect 系统调用在阻塞时被中断，是不能直接重启的（与内核socket 的状态有关)，有兴趣的同学可以深入研究一下connect 的内核实现。使用I/O复用等待连接完成，能避免connect不能重启的问题。

int client_fd = -1; 
struct sockaddr_in client_addr; 
socklen_t child_addrlen; 
while (1) { 
  call_accept: client_fd = accept(server_fd,NULL,NULL)； 
  if (client_fd < 0) { 
    if (EINTR == errno) 
    { 
      goto call_accept; 
    } 
    else 
    { 
      sw_sysError("accept fail"); 
      break; 
    }
  } 
}
信号处理：

利用信号处理，可以选择忽略信号，或者在安装信号时设置SA_RESTART属性。设置属性SA_RESTART，信号处理函数返回后，被安装信号中断的系统调用将自动恢复，示例代码如下。需要知道的是，设置SA_RESTART属性方法并不完全适用，对某些系统调用可能无效，这里只是提供一种解决问题的思路，示例代码如下：

int client_fd = -1; 
struct sigaction action,old_action; 
action.sa_handler = sig_handler; 
sigemptyset(&action.sa_mask); 
action.sa_flags = 0; 
action.sa_flags |= SA_RESTART; /// 若信号已经被忽略，则不设置 
sigaction(SIGALRM, NULL, &old_action)； 
if (old_action.sa_handler != SIG_IGN) 
{ 
  sigaction(SIGALRM, &action, NULL)； 
} 
while (1) { 
  client_fd = accept(server_fd,NULL,NULL)； 
  if (client_fd < 0) { 
    sw_sysError("accept fail"); break; 
  } 
}
1.2、I/O非阻塞
把非阻塞的文件描述符称为非阻塞I/O。可以通过设置SOCK_NONBLOCK标记创建非阻塞的socket fd，或者使用fcntl将fd设置为非阻塞。

对非阻塞fd调用系统接口时，不需要等待事件发生而立即返回，事件没有发生，接口返回-1，此时需要通过errno的值来区分是否出错，有过网络编程的经验的应该都了解这点。不同的接口，立即返回时的errno值不尽相同，如，recv、send、accept errno通常被设置为EAGIN 或者EWOULDBLOCK，connect 则为EINPRO- GRESS 。

IO非阻塞.png
当我们需要读取，在有数据可读的事件触发时，再调用recv，避免应用层不断去轮询检查是否可读，提高程序的处理效率。通常非阻塞I/O与I/O事件处理机制结合使用。

1.3、I/O复用
最常用的I/O事件通知机制就是I/O复用(I/O multiplexing)。Linux 环境中使用select/poll/epoll 实现I/O复用，I/O复用接口本身是阻塞的，在应用程序中通过I/O复用接口向内核注册fd所关注的事件，当关注事件触发时，通过I/O复用接口的返回值通知到应用程序，如图3所示,以recv为例。I/O复用接口可以同时监听多个I/O事件以提高事件处理效率。

IO复用png.png
1.4、SIGIO
除了I/O复用方式通知I/O事件，还可以通过SIGIO信号来通知I/O事件，如图所示。两者不同的是，在等待数据达到期间，I/O复用是会阻塞应用程序，而SIGIO方式是不会阻塞应用程序的。

SIGIO.png
1.5、异步I/O
POSIX规范定义了一组异步操作I/O的接口，不用关心fd 是阻塞还是非阻塞，异步I/O是由内核接管应用层对fd的I/O操作。异步I/O向应用层通知I/O操作完成的事件，这与前面介绍的I/O 复用模型、SIGIO模型通知事件就绪的方式明显不同。以aio_read 实现异步读取IO数据为例，如图5所示，在等待I/O操作完成期间，不会阻塞应用程序。

异步IO.png
1.6、I/O模型对比
前面介绍的5中I/O中，I/O 阻塞、I/O非阻塞、I/O复用、SIGIO 都会在不同程度上阻塞应用程序，而只有异步I/O模型在整个操作期间都不会阻塞应用程序。

IO模型对比.png
2、Reactor模型
Reactor的核心思想：

将关注的I/O事件注册到多路复用器上，一旦有I/O事件触发，将事件分发到事件处理器中，执行就绪I/O事件对应的处理函数中。模型中有三个重要的组件：

多路复用器：由操作系统提供接口，Linux提供的I/O复用接口有select、poll、epoll；
事件分离器：将多路复用器返回的就绪事件分发到事件处理器中；
事件处理器：处理就绪事件处理函数。
Reactor 类结构中包含有如下角色：

Reactor类结构角色.png
Handle：标示文件描述符；
Event Demultiplexer：执行多路事件分解操作，对操作系统内核实现I/O复用接口的封装；用于阻塞等待发生在句柄集合上的一个或多个事件（如select/poll/epoll）；
Event Handler：事件处理接口；
Event Handler A(B)：实现应用程序所提供的特定事件处理逻辑；
Reactor：反应器，定义一个接口，实现以下功能：
a)供应用程序注册和删除关注的事件句柄； b)运行事件处理循环； c)等待的就绪事件触发，分发事件到之前注册的回调函数上处理.

Reactor的工作流程：

Reactor工作流程图.png
以epoll_wait为例，使用同步I/O模型实现的Reactor模式的工作流程是：

1、主线程通过事件处理器epoll向内核注册socket的读或写事件。

2、主线程调用事件处理器epoll_wait阻塞等待（或超时等待）注册的事件。

3、当socket上有事件时，事件处理器epoll_wait返回到主线程。主线程调度事件处理器对事件进行处理。

4、事件处理器处理I/O操作，并处理数据。

Reactor模式特点：

Reactor通过调度相应的处理程序来相应I/O事件
处理程序执行非阻塞操作
通过绑定处理程序来管理事件。
2.1、经典Reactor模式
经典Reactor模式.png
在经典Reactor模式中，包含以下角色：

Reactor ：将I/O事件发派给对应的Handler
Acceptor ：处理客户端连接请求
Handlers ：执行非阻塞读/写
示例代码如下：

public class BasicReactorServer {

    public static void start(int port){
        try {
            Selector selector = Selector.open();
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().setReuseAddress(true);
            serverSocketChannel.bind(new InetSocketAddress(port), 128);

            //注册accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT,new Acceptor(selector, serverSocketChannel));

            //阻塞等待就绪事件
            while (selector.select() > 0){
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();

                //遍历就绪事件
                while (iterator.hasNext()){
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    Runnable handler = (Runnable) key.attachment();
                    handler.run();
                    keys.remove(key);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }


    /**
     * 接受连接处理
     */
    public static class Acceptor implements Runnable{

        private Selector selector;

        private ServerSocketChannel serverSocketChannel;

        public Acceptor(Selector selector, ServerSocketChannel serverSocketChannel) {
            this.selector = selector;
            this.serverSocketChannel = serverSocketChannel;
        }

        public void run() {
            try {
                SocketChannel socketChannel = serverSocketChannel.accept();
                socketChannel.configureBlocking(false);
                socketChannel.register(selector, SelectionKey.OP_READ,new DispatchHandler(socketChannel));
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }


    /**
     * 读取数据处理
     */
    public static class DispatchHandler implements Runnable{
        private SocketChannel socketChannel;

        public DispatchHandler(SocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        public void run() {
            try {
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int cnt = 0, total = 0;
                String msg = "";
                do {
                    cnt = socketChannel.read(buffer);
                    if (cnt > 0) {
                        total += cnt;
                        msg += new String(buffer.array());
                    }
                    buffer.clear();
                } while (cnt >= buffer.capacity());
                System.out.println("read data num:" + total);
                System.out.println("recv msg:" + msg);

                //回写数据
                ByteBuffer sendBuf = ByteBuffer.allocate(msg.getBytes().length + 1);
                sendBuf.put(msg.getBytes());
                socketChannel.write(sendBuf);

            }catch (Exception e){
                e.printStackTrace();
                if(socketChannel != null){
                    try {
                        socketChannel.close();
                    }catch (Exception ex){
                        ex.printStackTrace();
                    }
                }
            }
        }
    }





    public static void main(String[] args){
        BasicReactorServer.start(9999);
    }
}

处理流程：

Reactor打开Selector、创建服务端socket连接并绑定本地端口；
Reactor将服务器channel及SelectionKey.OP_ACCEPT注册到Selector中，同事绑定事件处理类Acceptor用于处理OP_ACCEPT事件；
Reactor循环selector.selectedKeys()，阻塞等待事件，遍历事件的SelectionKey并调用绑定的事件处理方法（run()）；
Acceptor()的run()中接收客户端连接，并将客户端的Channel及SelectionKey.OP_READ注册到Selector中，同时绑定客户端Channel的事件处理类DispatchHandler进行read事件处理；
DispatchHandler的run()中循环读取客户端数据，并将数据回写给客户端；
github:https://github.com/zhaozhou11/java-io.git

如上代码所示，Server的Channel和Client的Channel注册在同一个Selector上，且Server的accept处理和Client的channel的读写在同一个线程中。selector.select()阻塞等待就绪事件，一单有至少一个通道有就绪事件就会返回，selector.selectedKeys()中是就绪channel的SelectionKey的集合。

2.2、多工作线程Reactor模式
多工作线程Reactor模式.png
示例代码：

public class MultiReactorReactorServer {

    public static void start(int port){
        try {
            Selector selector = Selector.open();
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().setReuseAddress(true);
            serverSocketChannel.bind(new InetSocketAddress(port), 128);

            //注册accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT,new BasicReactorServer.Acceptor(selector, serverSocketChannel));

            //阻塞等待就绪事件
            while (selector.select() > 0){
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();

                //遍历就绪事件
                while (iterator.hasNext()){
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    Runnable handler = (Runnable) key.attachment();
                    handler.run();
                    keys.remove(key);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }


    /**
     * 接受连接处理
     */
    public static class Acceptor implements Runnable{

        private Selector selector;

        private ServerSocketChannel serverSocketChannel;

        public Acceptor(Selector selector, ServerSocketChannel serverSocketChannel) {
            this.selector = selector;
            this.serverSocketChannel = serverSocketChannel;
        }

        public void run() {
            try {
                SocketChannel socketChannel = serverSocketChannel.accept();
                socketChannel.configureBlocking(false);
                socketChannel.register(selector, SelectionKey.OP_READ,new BasicReactorServer.DispatchHandler(socketChannel));
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }


    /**
     * 读取数据处理
     */
    public static class DispatchHandler implements Runnable{

        private static Executor executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() << 1);
        private SocketChannel socketChannel;

        public DispatchHandler(SocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        public void run() {
            executor.execute(new ReaderHandler(socketChannel));
        }
    }


    public static class ReaderHandler implements Runnable{
        private SocketChannel socketChannel;

        public ReaderHandler(SocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        public void run() {
            try {
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int cnt = 0, total = 0;
                String msg = "";
                do {
                    cnt = socketChannel.read(buffer);
                    if (cnt > 0) {
                        total += cnt;
                        msg += new String(buffer.array());
                    }
                    buffer.clear();
                } while (cnt >= buffer.capacity());
                System.out.println("read data num:" + total);
                System.out.println("recv msg:" + msg);

                //回写数据
                ByteBuffer sendBuf = ByteBuffer.allocate(msg.getBytes().length + 1);
                sendBuf.put(msg.getBytes());
                socketChannel.write(sendBuf);

            }catch (Exception e){
                e.printStackTrace();
                if(socketChannel != null){
                    try {
                        socketChannel.close();
                    }catch (Exception ex){
                        ex.printStackTrace();
                    }
                }
            }
        }
    }




    public static void main(String[] args){
        BasicReactorServer.start(9999);
    }
处理流程：

Reactor打开Selector、创建服务端socket连接并绑定本地端口；
Reactor将服务器channel及SelectionKey.OP_ACCEPT注册到Selector中，同事绑定事件处理类Acceptor用于处理OP_ACCEPT事件；
Reactor循环selector.selectedKeys()，阻塞等待事件，遍历事件的SelectionKey并调用绑定的事件处理方法（run()）；
Acceptor()的run()中接收客户端连接，并将客户端的Channel及SelectionKey.OP_READ注册到Selector中，同时绑定客户端Channel的事件处理类DispatchHandler进行read事件处理；
DispatchHandler的run()中通过多线程处理ReadHandler类，进行读事件处理；
ReadHandler中循环读取客户端数据，并将数据回写给客户端；
github：https://github.com/zhaozhou11/java-io.git

2.3、多Reactor的Reactor模式
多Reactor的Reactor模式.png
示例源码：

public class MultiWorkerThreadReactorServer {

    private final static int PROCESS_NUM = 10;


    public static void start(int port){
        try {
            Selector selector = Selector.open();
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().setReuseAddress(true);
            serverSocketChannel.bind(new InetSocketAddress(port), 128);

            //注册accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT,new BasicReactorServer.Acceptor(selector, serverSocketChannel));

            DispatchHandler[] handlers = new DispatchHandler[PROCESS_NUM];
            for (DispatchHandler h: handlers){
                h = new DispatchHandler();
            }

            int count = 0;

            //阻塞等待就绪事件
            while (selector.select() > 0){
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();

                //遍历就绪事件
                while (iterator.hasNext()){
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    if(key.isAcceptable()){
                        SocketChannel socketChannel = serverSocketChannel.accept();
                        socketChannel.configureBlocking(false);
                        handlers[count ++ % PROCESS_NUM].addChannel(socketChannel);
                    }
                    keys.remove(key);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    /**
     * 读取数据处理
     */
    public static class DispatchHandler{

        private static Executor executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() << 1);
        private Selector selector;

        public DispatchHandler() throws IOException {
           selector = Selector.open();
           this.start();
        }

        public void addChannel(SocketChannel socketChannel) throws ClosedChannelException{
            socketChannel.register(selector, SelectionKey.OP_READ);
            this.selector.wakeup();
        }


        public void start() {
            executor.execute(new Runnable() {

                public void run() {
                    while (true){
                        Set<SelectionKey> keys = selector.selectedKeys();
                        if(CollectionUtils.isNotEmpty(keys)){
                            Iterator<SelectionKey> iterator = keys.iterator();
                            if(iterator.hasNext()){
                                SelectionKey key = iterator.next();
                                SocketChannel socketChannel = (SocketChannel)key.channel();
                                iterator.remove();
                                if(key.isReadable()){
                                    try {
                                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                                        int cnt = 0, total = 0;
                                        String msg = "";
                                        do {
                                            cnt = socketChannel.read(buffer);
                                            if (cnt > 0) {
                                                total += cnt;
                                                msg += new String(buffer.array());
                                            }
                                            buffer.clear();
                                        } while (cnt >= buffer.capacity());
                                        System.out.println("read data num:" + total);
                                        System.out.println("recv msg:" + msg);

                                        //回写数据
                                        ByteBuffer sendBuf = ByteBuffer.allocate(msg.getBytes().length + 1);
                                        sendBuf.put(msg.getBytes());
                                        socketChannel.write(sendBuf);

                                    }catch (Exception e){
                                        e.printStackTrace();
                                        if(socketChannel != null){
                                            try {
                                                socketChannel.close();
                                            }catch (Exception ex){
                                                ex.printStackTrace();
                                            }
                                        }
                                    }
                                }
                                keys.remove(key);
                            }
                        }
                    }
                }
            });

        }
    }





    public static void main(String[] args){
        BasicReactorServer.start(9999);
    }

}

github：https://github.com/zhaozhou11/java-io.git

处理流程：

Reactor打开Selector、创建服务端socket连接并绑定本地端口；
Reactor将服务器channel及SelectionKey.OP_ACCEPT注册到Selector中，同事绑定事件处理类Acceptor用于处理OP_ACCEPT事件；
Reactor提前创建PROCESS_NUM 个事件处理类DispatchHandler；
Reactor循环selector.selectedKeys()，阻塞等待事件，遍历事件的SelectionKey，当是客户端连接事件时，接收客户端连接，并将客户端Channel通过DispatchHandler.addChannel()派发给提前创建好的DispatchHandler类；
每个DispatchHandler中会公用事件处理线程池，并有独立的事件管理器Selector，当调用addChannel()时，会将客户端Channel及read事件注册到本地Selector中,并在线程池中等待事件；
3、Proactor模型
与Reactor不同的是，Proactor使用异步I/O系统接口将I/O操作托管给操作系统，Proactor模型中分发处理异步I/O完成事件，并调用相应的事件处理接口来处理业务逻辑。

Preactor模型.png
Proactor类结构中包含有如下角色：

Handle：用来标识socket连接或是打开文件；
Async Operation Processor：异步操作处理器；负责执行异步操作，一般由操作系统内核实现；
Async Operation：异步操作；
Completion Event Queue：完成事件队列；异步操作完成的结果放到队列中等待后续使用；
Proactor：主动器；为应用程序进程提供事件循环；从完成事件队列中取出异步操作的结果，分发调用相应的后续处理逻辑；
Completion Handler：完成事件接口；一般是由回调函数组成的接口；
Completion Handler A(B)：完成事件处理逻辑；实现接口定义特定的应用处理逻辑。
Proactor模型的简化的工作流程：

Reactor工作流程图.png
发起I/O异步操作，注册I/O完成事件处理器;
事件分离器等待I/O操作完成事件；
内核并行执行实际的I/O操作，并将结果数据存入用户自定义缓 冲区；
内核完成I/O操作，通知事件分离器，事件分离器调度对应的事件处理器；
事件处理器处理用户自定义缓冲区中的数据。
Proactor利用异步I/O并行能力，可给应用程序带来更高的效率，但是同时也增加了编程的复杂度。windows对异步I/O提供了非常好的支持，常用Proactor的模型实现服务器；而Linux对异步I/O操作(aio接口)的支持并不是特别理想，而且不能直接处理accept，因此Linux平台上还是以Reactor模型为主。

Asynchronous I/O：

java提供Asynchronous I/O相关接口以支持异步I/O，AIO有两种api进行操作：

Future 方式；
Callback 方式。
Future 方式：

Future 方式：即提交一个 I/O 操作请求，返回一个 Future。然后您可以对 Future 进行检查，确定它是否完成，或者阻塞 IO 操作直到操作正常完成或者超时异常。

AsynchronousSocketChannel ch = AsynchronousSocketChannel.open();

// 连接远程服务器，等待连接完成或者失败

Future<Void> result = ch.connect(remote);

// 进行其他工作，例如，连接后的准备环境，f.e.

//prepareForConnection();

//Future 返回 null 表示连接成功

if(result.get()!=null){

   // 连接失败，清理刚才准备好的环境，f.e.

   //clearPreparation();

   return;

}

// 网络连接正常建立

...

ByteBuffer buffer = ByteBuffer.allocateDirect(8192);

// 进行读操作

Future<Integer> result = ch.read(buffer);

// 此时可以进行其他工作，f.e.

//prepareLocalFile();

// 然后等待读操作完成

try {

   int bytesRead = result.get();

   if(bytesRead==-1){

   // 返回 -1 表示没有数据了而且通道已经结束，即远程服务器正常关闭连接。

       //clear();

       return;

   }

   // 处理读到的内容，例如，写入本地文件，f.e.

   //writeToLocolFile(buffer);

} catch (ExecutionExecption x) {

   //failed

}
需要注意的是，因为 Future.get()是同步的，所以如果不仔细考虑使用场合，使用 Future 方式可能很容易进入完全同步的编程模式，从而使得异步操作成为一个摆设。如果这样，那么原来旧版本的 Socket API 便可以完全胜任，大可不必使用异步 I/O。

Callback 方式:

Callback 方式：即提交一个 I/O 操作请求，并且指定一个 CompletionHandler。当异步 I/O 操作完成时，便发送一个通知，此时这个 CompletionHandler 对象的 completed 或者 failed 方法将会被调用。

public interface CompletionHandler<V,A> {

   // 当操作完成后被调用，result 参数表示操作结果，

   //attachment 参数表示提交操作请求时的参数。

   void completed(V result, A attachment);

   // 当操作失败是调用，exc 参数表示失败原因。attachment 参数同上。

   void failed(Throwable exc, A attachment);

}
V表示结果值的类型。对于异步网络通道的读写操作而言，这个结果值 V 都是整数类型，表示已经操作的卦数，如果是 -1，NIO.2 内核实现保证传递的 ByteBuffer参数不会有变化。
A表示关联到 I/O 操作的对象的类型。用于传递操作环境。通常会封装一个连接环境。
如果成功则 completed 方法被调用。如果失败则 failed 方法被调用。
AIO 提供四种类型的异步通道以及不同的 I/O 操作接受一个 CompletionHandler 对象，它们分别是：

AsynchronousSocketChannel：connect，read，write
AsynchronousFileChannel：lock，read，write
AsynchronousServerSocketChannel：accept
AsynchronousDatagramChannel：read，write，send，receive
示例源码：

public class Preactor {

    private final static int port = 9999;

    public static void start() throws IOException{

        AsynchronousServerSocketChannel channel = null;
        try {
            AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(Executors.newFixedThreadPool(10));
            channel = AsynchronousServerSocketChannel.open(group).bind(new InetSocketAddress(port), 128);
            System.out.println("服务器已启动，端口号：" + port);

            channel.accept(null, new Acceptor());
            CountDownLatch latch = new CountDownLatch(1);
            latch.await();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args){

        try {

            Preactor.start();

        }catch (Exception e){
            e.printStackTrace();
        }

    }


    public static class Acceptor implements CompletionHandler<AsynchronousSocketChannel,AsynchronousServerSocketChannel>{

        public void completed(AsynchronousSocketChannel result, AsynchronousServerSocketChannel attachment) {
            try {
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                result.read(buffer, buffer, new ReadHandler(result));
            }catch (Exception e){
                e.printStackTrace();
            }

        }

        public void failed(Throwable exc, AsynchronousServerSocketChannel attachment) {
            exc.printStackTrace();
        }
    }




    public static class ReadHandler implements CompletionHandler<Integer, ByteBuffer> {

        private AsynchronousSocketChannel socketChannel;

        public ReadHandler(AsynchronousSocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        public void completed(Integer result, ByteBuffer attachment) {
            try {
                attachment.flip();
                String msg = new String(attachment.array());
                System.out.println("recv client msg:" + msg);
                socketChannel.write(attachment,attachment, new WritterHandler(socketChannel));

            }catch (Exception e){
                e.printStackTrace();
            }
        }


        public void failed(Throwable exc, ByteBuffer attachment) {
            exc.printStackTrace();
            try {
                socketChannel.close();;
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }



    public static class WritterHandler implements CompletionHandler<Integer, ByteBuffer>{

        private AsynchronousSocketChannel socketChannel;

        public WritterHandler(AsynchronousSocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        public void completed(Integer result, ByteBuffer attachment) {
            try {
                attachment.clear();
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                socketChannel.read(buffer,buffer, new ReadHandler(socketChannel));
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        public void failed(Throwable exc, ByteBuffer attachment) {
            exc.printStackTrace();
            try {
                socketChannel.close();;
            }catch (Exception e){
                e.printStackTrace();
            }

        }
    }
}
处理步骤：

start()方法中打开一个异步channel并绑定端口及完成的回调类Acceptor ；
Acceptor 的completed()中进行read并绑定ReadHandler ；
ReadHandler的completed()获取读取的数据，进行write并绑定WritterHandler；
WritterHandler中的写完成方法completed()中绑定ReadHandler，就可实现循环度写；
github：https://github.com/zhaozhou11/java-io.git

4、Reactor与Proactor对比
主动和被动

以主动写为例：

Reactor将handle放到select()，等待可写就绪，然后调用write()写入数据；写完处理后续逻辑；

Proactor调用aoi_write后立刻返回，由内核负责写操作，写完后调用相应的回调函数处理后续逻辑；

可以看出，Reactor被动的等待指示事件的到来并做出反应；它有一个等待的过程，做什么都要先放入到监听事件集合中等待handler可用时再进行操作；

Proactor直接调用异步读写操作，调用完后立刻返回；

实现

Reactor实现了一个被动的事件分离和分发模型，服务等待请求事件的到来，再通过不受间断的同步处理事件，从而做出反应；

Proactor实现了一个主动的事件分离和分发模型；这种设计允许多个任务并发的执行，从而提高吞吐量；并可执行耗时长的任务（各个任务间互不影响）

优点

Reactor实现相对简单，对于耗时短的处理场景处理高效；

操作系统可以在多个事件源上等待，并且避免了多线程编程相关的性能开销和编程复杂性；

事件的串行化对应用是透明的，可以顺序的同步执行而不需要加锁；

事务分离：将与应用无关的多路分解和分配机制和与应用相关的回调函数分离开来，

Proactor性能更高，能够处理耗时长的并发场景；

缺点

Reactor处理耗时长的操作会造成事件分发的阻塞，影响到后续事件的处理；

Proactor实现逻辑复杂；依赖操作系统对异步的支持，目前实现了纯异步操作的操作系统少，实现优秀的如windows IOCP，但由于其windows系统用于服务器的局限性，目前应用范围较小；而Unix/Linux系统对纯异步的支持有限，应用事件驱动的主流还是通过select/epoll来实现；

适用场景

Reactor：同时接收多个服务请求，并且依次同步的处理它们的事件驱动程序；

Proactor：异步接收和同时处理多个服务请求的事件驱动程序；

作者：桥头放牛娃
链接：https://www.jianshu.com/p/b4de9b85c79d
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
