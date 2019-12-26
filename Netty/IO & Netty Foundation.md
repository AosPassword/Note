# IO模型与Netty

## 名词分析

​	在高性能的IO体系设计中，有几个关键名词。

- 同步
- 异步
- 阻塞
- 非阻塞

​    很明显，同步和异步是一对概念，阻塞和非阻塞是一对概念。所以这两对两两组合就又有四个概念：

- 同步阻塞
- 同步非阻塞
- 异步阻塞
- 异步非阻塞

### 同步异步——阻塞非阻塞

​	现在老板给我打了个电话。

​	那我就接起电话来听。

​	结果老板半天没说话。

​	我就在那傻等着一直听，木得办法，谁让我不敢挂电话，又怕错过老板说话。

​	好久之后，老板说了，让我把写好的文件发给他。

​	然后我就把文件发给了他。

​	这叫做**同步阻塞**。

​	这么干了几天我觉得，等老板说话太无聊了。

​	哎，老板又来电话了。

​	我接起来电话，直接往手机那里放一录音机，我看电视去了，每过段时间过来看看录音机，看看老板说话了没，如果说了事情，我去干。

​	这叫做**同步非阻塞**。

​	老板对我的耐心表示深感敬佩，就给我好多事情干。

​	我一个人干不完啊，那咋整啊。

​	我就给我手机写了个AI，当老板问我要文件的时候，AI自动把文件发给老板。

​	现在老板来电话了，我接起来电话，然后我就看电视去了。

​	老板现在说话了，AI听见老板让我发文件，AI自动把我的文件发给他。

​	这叫做**异步非阻塞**

​    其中同步和异步这两个词的关注点为，当老板说话之后，处理消息的人是不是我。

​	而阻塞和非阻塞的关注点为，老板还没说话之前，我是否在那里一直听着手机。

​	当然这样说是不够准确的，接下来看看专业的。

## 基础知识

### Socket

​	我们知道两个进程如果需要进行通讯最基本的一个前提能能够唯一的标示一个进程，在本地进程通讯中我们可以使用PID来唯一标示一个进程，但PID只在本地唯一，网络中的两个进程PID冲突几率很大，这时候我们需要另辟它径了，我们知道IP层的ip地址可以唯一标示主机，而TCP层协议和端口号可以唯一标示主机的一个进程，这样我们可以利用ip地址＋协议＋端口号唯一标示网络中的一个进程。

​	能够唯一标示网络中的进程后，它们就可以利用socket进行通信了，什么是socket呢？我们经常把socket翻译为套接字，socket是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用已实现进程在网络中通信。

​	![img](https://images0.cnblogs.com/blog/349217/201312/05225723-2ffa89aad91f46099afa530ef8660b20.jpg)

​	socket起源于UNIX，在Unix一切皆文件哲学的思想下，socket是一种"打开—读/写—关闭"模式的实现，服务器和客户端各自维护一个"文件"，在建立连接打开后，可以向自己文件写入内容供对方读取或者读取对方内容，通讯结束时关闭文件。

### socket通信流程

​	socket是"打开—读/写—关闭"模式的实现，以使用TCP协议通讯的socket为例，其交互流程大概是这样子的

![img](https://images0.cnblogs.com/blog/349217/201312/05232335-fb19fc7527e944d4845ef40831da4ec2.png)



## IO模型

​	按照《Unix网络编程》的划分，IO模型可以分为：阻塞IO、非阻塞IO、IO复用、信号驱动IO和异步IO，按照POSIX[^4]标准来划分只分为两类：同步IO和异步IO。

​	我们根据《Unix网络编程》的IO模型来详细讲一下IO。背景是Linux环境下的network IO。

​	对于一个network IO（以read举例），他会涉及到两个系统对象，一个是调用这个IO的process/thread，另一个就是系统内核（kernel）。当也给read操作发生的时候，会经历两个阶段：

- 等待数据阶段
- 将数据从内核拷贝到进程

​    这两个阶段很重要，因为这些IO模型的区别就是在两个阶段上各有不同的情况。

### 阻塞IO(Blocking IO)

​	在linux中，默认情况下所有的socket都是blocking，一个经典的流程如下

![img](http://static.oschina.net/uploads/space/2014/0726/163413_LWxd_220225.gif)	

​	当用户进程调用了recvfrom[^1]这个系统调用，kernel就开始了IO的第一个阶段:准备数据。对于network io来说，很多数据在一开始还没有完成（比如开没收到一个完整的UDP），这个时候kernel就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当kernel一直等到数据准备好了，就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block状态，重新运行起来。

​	所以blocking IO的特点在于，他在IO执行的两个阶段都block了。

​	这也就是最简单的一问一答形式的服务器了：

![img](http://static.oschina.net/uploads/space/2014/0726/163457_uDiZ_220225.jpg)

​	这里的大部分的Socket接口都是阻塞型的，比如``recy()``和``send()``。

​	事实上，除非特别指定，几乎所有的IO接口都是阻塞的。但是这有个很大的问题。比如调用``send()``的同时，线程将会被阻塞，阻塞时，线程无法执行任何运算或者响应任何的网络请求。

​	一个简单的改进方法是在服务器端使用多线程（或者多进程），多线程的目的是让所有连接都有独立的线程，这样任何一个连接的阻塞都不会影响到其他的连接。

​	改进后的模型如下：

![img](http://static.oschina.net/uploads/space/2014/0726/163636_t211_220225.jpg)

​	

​	在上述的线程 / 时间图例中，主线程持续等待客户端的连接请求，如果有连接，则创建新线程，并在新线程中提供为前例同样的问答服务。

​	那为什么主线程的socket可以accept多次？

​	实际上socket的设计者可能特意为多客户机的情况留下了伏笔，让accept()能够返回一个新的socket[^2]。

```c++
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

​	参数sockfd是从socket(),bind()和linsten()中沿用下来的**socket**句柄，执行完bind()和listen()后，操作系统已经开始在指定的端口处监听所有的连接请求，如果有请求，则将该连接请求加入请求队列。调用accept()接口正是从 socket s 的请求队列抽取第一个连接信息，创建一个与s同类的新的socket返回句柄。新的socket句柄即是后续read()和recv()的输入参数。如果请求队列当前没有请求，则accept() 将进入阻塞状态直到有请求进入队列。

​	但是这个模型有个重大的**缺点**：

​	如果要同时响应成百上千路的连接请求，则无论多线程还是多进程都会严重占据系统资源，降低系统对外界响应效率，而线程与进程本身也更容易进入假死状态。

​	那能不能用“线程池”或者“连接池”改善一下？

​	“线程池”旨在减少创建和销毁线程的频率，其维持一定合理数量的线程，并让空闲的线程重新承担新的执行任务。“连接池”维持连接的缓存池，尽量重用已有的连接、减少创建和关闭连接的频率。这两种技术都可以很好的降低系统开销。

​	但是，“线程池”和“连接池”技术也只是在一定程度上缓解了频繁调用IO接口带来的资源占用。而且，**所谓“池”始终有其上限，当请求大大超过上限时，“池”构成的系统对外界的响应并不比没有池的时候效果好多少。所以使用“池”必须考虑其面临的响应规模，并根据响应规模调整“池”的大小。**

​	总之，多线程模型可以方便高效的解决小规模的服务请求，但面对大规模的服务请求，多线程模型也会遇到瓶颈，可以用非阻塞接口来尝试解决这个问题。

### 非阻塞IO（non-blocking IO）

​	Linux下，可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子：

![img](http://static.oschina.net/uploads/space/2014/0726/163739_X6pl_220225.gif)

​		从图中可以看出，当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

​	非阻塞的接口相比于阻塞型接口的显著差异在于，在被调用之后立即返回。

​	使用fcntl函数可以将某句柄fd设为非阻塞状态。

```c++
int fcntl(int fd, int cmd); 
int fcntl(int fd, int cmd, long arg); 
int fcntl(int fd, int cmd, struct flock *lock); 
```

参数：   

​	fd：文件描述词。 
​	cmd：操作命令。 
​	arg：供命令使用的参数。 
​	lock：同上

​	而此处我们要使用的是：

```c++
fcntl( fd, F_SETFL, O_NONBLOCK ); 
```

​	F_SETFL ：设置文件状态标志。

​	那么一个线程就可以同时从多个连接中检测数据是否到达。

![img](http://static.oschina.net/uploads/space/2014/0726/163824_lWc7_220225.jpg)

​	在非阻塞状态下，recv()接口在被调用后立即返回，返回值代表了不同的含义。如在本例中：

- recv() 返回值大于 0，表示接受数据完毕，返回值即是接受到的字节数。
- recv() 返回 0，表示连接已经正常断开。
- recv() 返回 -1，且 errno 等于 EAGAIN，表示 recv 操作还没执行完成；
- recv() 返回 -1，且 errno 不等于 EAGAIN，表示 recv 操作遇到系统错误 error

​     

​	回忆一下我们BIO所带来的缺点：太多的阻塞线程导致了系统资源严重浪费，我们这就整了一个线程处理多个客户端，线程确实少了，上一个问题被解决了！

​	但是，这个模型还有一个很大的缺陷：：

​	**循环调用recv()将大幅度推高CPU 占用率；此外，在这个方案中recv()更多的是起到检测“操作是否完成”的作用。**一个线程既要recv()还要处理消息。

​	**实际操作系统提供了更为高效的检测“操作是否完成“作用的接口，例如select()多路复用模式，可以一次检测多个连接是否活跃。**

### 多路复用IO(IO multiplexing)

​		IO multiplexing的核心是select/epoll[^3]，有些地方也称这种IO方式为**事件驱动IO**(event driven IO)，select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select/epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。它的流程如图：

​	![img](http://static.oschina.net/uploads/space/2014/0726/163932_w5nW_220225.gif)

​	当用户进程调用了select，那么整个进程都会被block，同时kernel会监视所有的select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。

​	这个图其实和Blocking IO那个图长得挺像的，这个方法又回到了阻塞了，不仅如此，还比纯粹的recvfrom多调用了一个方法。但是和Blocking IO不相同之处，也是比NIO优越之处在于，用select的优势在于它可以同时处理多个connection。

​	如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。

​	多路复用模型中，对于每一个Socket，一般都设置为Non-blocking，但是你看上面那个进程调用select方法其实是阻塞的。

#### select

​	老规矩，来康康select()方法。

```java
    int select(int maxfdp1, 
               fd_set *readset, 
               fd_set *writeset, 
               fd_set *exceptset,
               struct timeval *timeout);
```

​	select函数的参数会告诉内核：

- 我们所关心的文件描述符（也就是我们关心的Socket）
- 对每个描述符，我们所关心的状态。(我们是要想从一个文件描述符中读或者写，还是关注一个描述符中是否出现异常)
- 我们要等待多长时间。(我们可以等待无限长的时间，等待固定的一段时间，或者根本就不等待)

​    而Select函数返回后，内核会告诉我们一些消息：

- 对我们的要求已经做好准备的描述符的个数
- 对于三种条件哪些描述符已经做好准备(读，写，异常)

​    先看一下最简单的最后一个参数：

```C++
struct timeval{      
        long tv_sec;   /*秒 */
        long tv_usec;  /*微秒 */   
    }
```

有三种情况：

​    timeout == NULL  等待无限长的时间。等待可以被一个信号中断。当有一个描述符做好准备或者是捕获到一个信号时函数会返回。如果捕获到一个信号， select函数将返回 -1,并将变量 erro设为 EINTR。

​	timeout->tv_sec == 0 &&timeout->tv_usec == 0不等待，直接返回。加入描述符集的描述符都会被测试，并且返回满足要求的描述符的个数。这种方法通过轮询，无阻塞地获得了多个文件描述符状态。

​	timeout->tv_sec !=0 ||timeout->tv_usec!= 0 等待指定的时间。当有描述符符合条件或者超过超时时间的话，函数返回。在超时时间即将用完但又没有描述符合条件的话，返回 0。对于第一种情况，等待也会被信号所中断。

​	即使select是可以是非阻塞的，但是因为Select直接就可以检查多个socket，没必要让他非阻塞，并且如果设置为非阻塞，只会增大系统的开销。

​	中间的三个参数 readset, writset, exceptset,指向描述符集。这些参数指明了我们关心哪些描述符，和需要满足什么条件(可写，可读，异常)。

​	一个文件描述集保存在 fd_set 类型中。fd_set类型变量每一位代表了一个描述符。我们可以认为它只是一个由很多二进制位构成的数组。如下图所示：

![img](http://img.blog.csdn.net/20131007164301375?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGluZ2Zlbmd0ZW5nZmVp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	例如要在某 fd_set 中标记一个值为16的句柄，则该fd_set的第16个bit位被标记为1。具体的置位、验证可使用 FD_SET、FD_ISSET等宏实现。

```c++
#include <sys/select.h>   

int FD_ZERO(int fd, fd_set *fdset);   

int FD_CLR(int fd, fd_set *fdset);   

int FD_SET(int fd, fd_set *fd_set);   

int FD_ISSET(int fd, fd_set *fdset);
```

​	如果这个集合中有一个事件发生，select就会返回一个大于0的值，表示有事件发生；如果没有事件发生，则根据timeout参数再判断是否超时，若超出timeout的时间，select返回0，若发生错误返回负值。

​	如果给set传入null，则说明不关心此事件。

​	所以说理解select模型的关键在于理解fd_set。

​	设fd_set长度为1字节，fd_set中的每一bit可以对应一个文件描述符fd。则1字节长的fd_set最大可以对应8个fd。

-  执行FD_ZERO(&set)，则set用位表示是0000,0000。

- 若fd＝5,执行FD_SET(fd,&set);后set变为0001,0000(第5位置为1)

- 若再加入fd＝2，fd=1,则set变为0001,0011
- 执行select(6,&set,0,0,0)阻塞等待，只关心&set中的读事件
- 若fd=1,fd=2上都发生可读事件，则select返回，此时set变为0000,0011。注意：没有事件发生的fd=5被清空。

####新的IO模型

​	新的IO模型如下：

![img](http://static.oschina.net/uploads/space/2014/0726/164051_MYc9_220225.jpg)

​	这里需要指出的是，客户端的一个 connect() 操作，将在服务器端激发一个“可读事件”，所以 select() 也能探测来自客户端的 connect() 行为。

​	最关键的地方是如何动态维护select()的三个参数readfds、writefds和exceptfds。

​	作为**输入参数**，readfds应该标记所有的需要探测的“可读事件”的句柄，其中永远包括那个探测 connect() 的那个“母”句柄；同时，writefds 和 exceptfds 应该标记所有需要探测的“可写事件”和“错误事件”的句柄 ( 使用 FD_SET() 标记 )。

​	作为**输出参数**，readfds、writefds和exceptfds中的保存了 select() 捕捉到的所有事件的句柄值。程序员需要检查的所有的标记位 ( 使用FD_ISSET()检查 )，以确定到底哪些句柄发生了事件。

​	所以如果select()发现某句柄捕捉到了“可读事件”，服务器程序应及时做recv()操作，并根据接收到的数据准备好待发送数据，并将对应的句柄值加入writefds，准备下一次的“可写事件”的select()探测。同样，如果select()发现某句柄捕捉到“可写事件”，则程序应及时做send()操作，并准备好下一次的“可读事件”探测准备。

​	下图描述的就是上述模型的执行周期。

​	![img](http://static.oschina.net/uploads/space/2014/0726/164207_O9we_220225.jpg)

​	这种模型的特征在于每一个执行周期都会探测一次或一组事件，一个特定的事件会触发某个特定的响应。我们可以将这种模型归类为“**事件驱动模型**”。

​	相比其他模型，使用select() 的事件驱动模型只用单线程（进程）执行，占用资源少，不消耗太多 CPU，同时能够为多客户端提供服务。如果试图建立一个简单的事件驱动的服务器程序，这个模型有一定的参考价值。

​	但这个模型依旧有着很多问题。**首先select()接口并不是实现“事件驱动”的最好选择。因为当需要探测的句柄值较大时，select()接口本身需要消耗大量时间去轮询各个句柄。**很多操作系统提供了更为高效的接口，如linux提供了epoll，BSD提供了kqueue，Solaris提供了/dev/poll，…。如果需要实现更高效的服务器程序，类似epoll这样的接口更被推荐。遗憾的是不同的操作系统特供的epoll接口有很大差异，所以使用类似于epoll的接口实现具有较好跨平台能力的服务器会比较困难。

​	**其次，该模型将事件探测和事件响应夹杂在一起，一旦事件响应的执行体庞大，则对整个模型是灾难性的。**如下例，庞大的执行体1的将直接导致响应事件2的执行体迟迟得不到执行，并在很大程度上降低了事件探测的及时性。

​	第四种模型用的少，而且JDK没实现，就不说了。

### 异步IO（Asynchronous IO）

​	Linux下的asynchronous IO用的不多，从内核2.6版本才开始引进。（现在应该是4.15+）

​	![img](http://static.oschina.net/uploads/space/2014/0726/164333_8ZHk_220225.gif)

​	当进程aio_read之后，就可以去做其他的事情了，而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先他会立刻返回，不会block。然后kernel会等待数据准备完成，然后讲数据拷贝到用户内存。所有事情干完之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。

​	异步IO是真正非阻塞的，它不会对请求进程产生任何的阻塞，因此对高并发的网络服务器实现至关重要。

### 回顾问题

​	现在回头看看最初的问题，阻塞和非阻塞的区别在哪，同步和异步的区别在哪。

​	阻塞和非阻塞式很好判断其去别的。阻塞IO会一直阻塞对应进程，而非阻塞会立即返回结果。

​	至于的区别，先要给出POSIX[^4]对两者的定义：

>A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes
>
>An asynchronous I/O operation does not cause the requesting process to be blocked

​	按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO。

​	哎，为啥人家都叫non-blocking了还要依旧定义它是block了process？

​	因为定义中的IO operation是指真实的IO操作，就是例子中的recvfrom这个系统调用。**non-blocking IO在执行recvfrom这个系统调用的时候，如果kernel的数据没有准备好，这时候不会block进程。但是当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内进程是被block的。**

​	而asynchronous IO则不一样，当进程发起IO操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。

## Java IO

JAVA的IO主要有三种：BIO，NIO，AIO

### BIO（Blocking IO）

​	BIO主要由三个部分组成：Client，Server，Thread。每当有一个Client连接上了Server，就起一个Thread去处理这个Client。

​	服务器提供IP地址和监听的端口，客户端通过TCP的三次握手与服务器连接，连接成功后，双放才能通过套接字通信。

​	通过Socket和ServerSocket完成套接字通道的实现。阻塞，同步，建立连接耗时。

![img](https://images0.cnblogs.com/i/288799/201408/172148504055625.jpg)

```java
package bio;

import java.io.IOException;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
/**
 * @since 1.8
 */
public class Server {
    private static final int bytesSize = 1024;
    public static void main(String[] args) {
        try(ServerSocket serverSocket = new ServerSocket()){
            serverSocket.bind(new InetSocketAddress("127.0.0.1",8159));
            while (true){
                Socket s = serverSocket.accept();
                new Thread(()->{
                    handle(s);
                }).run();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void handle(Socket s) {
        try(s){
            byte[] bytes = new byte[bytesSize];
            int len = s.getInputStream().read(bytes);//阻塞

            System.out.println(new String(bytes,0,len));

            s.getOutputStream().write(bytes,0,len);
            s.getOutputStream().flush();
            System.out.println("write success");
            s.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

```java
package bio;

import java.io.IOException;
import java.net.Socket;
import java.net.UnknownHostException;

public class Client {
    private static final int bytesSize = 1024;
    public static void main(String[] args) {
        try(Socket s = new Socket("127.0.0.1",8159)){
            Thread.sleep(1000);
            s.getOutputStream().write("hello world".getBytes());
            s.getOutputStream().flush();
            System.out.println("write success");
            Thread.sleep(1000);
            byte[] bytes = new byte[bytesSize];
            int len = s.getInputStream().read(bytes);
            System.out.println(new String(bytes,0,len));
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	这种模式虽然处理起来简单方便，但是由于服务器为每个client的连接都采用一个线程去处理，使得资源占用非常大。

​	若客户端数量增多，频繁地创建和销毁线程会给服务器很大的压力。后改良为用线程池的方式代替新增线程，被称为伪异步IO。

​	那么为什么说它是伪异步IO呢？

​	因为对于接受连接的线程来说，它另外起了一个线程去处理这个Socket，但是这个处理Socket的线程底层使用到的还是**同步阻塞IO**。

​	当有大量并发请求，超过最大数量的线程就只能等待，直到线程池中有空闲的线程可以被复用。而对Socket的输入流进行读取的时候，会一直阻塞。

​	虽然我们有一群线程帮忙处理程序，但是因为这些线程大多都是处于阻塞状态，尽管这些线程就阻塞在那里，什么也没干，还是要告诉你他忙。

​	所以说轻松了那个连接者的线程，但是其他线程还是在那里傻等。

### NIO（New/Non-Blocking IO)

​	NIO可以在`Channel`进行读写操作。
​	这些`Channel`都会被注册在`Selector`多路复用器上。`Selector`通过一个线程不停的轮询这些`Channel`。找出已经准备就绪的Channel执行IO操作。

#### 缓冲区Buffer

​	缓冲区(Buffer)就是在内存中预留指定大小的存储空间用来对输入/输出(I/O)的数据作临时存储，这部分预留的内存空间就叫做缓冲区

​	使用缓冲区有这么两个好处：

1、减少实际的物理读写次数

2、缓冲区在创建时就被分配内存，这块内存区域一直被重用，可以减少动态分配和回收内存的次数

​	在Java NIO中，缓冲区的作用也是用来临时存储数据，可以理解为是I/O操作中数据的中转站。缓冲区直接为通道(Channel)服务，写入数据到通道或从通道读取数据，这样的操利用缓冲区数据来传递就可以达到对数据高效处理的目的。在NIO中主要有八种缓冲区类(其中MappedByteBuffer是专门用于内存映射的一种ByteBuffer)：

![img](https://img-blog.csdn.net/20140803152029718)

​	ByteBuffer类提供了4个静态工厂方法来获得ByteBuffer的实例：

| 方法                                        | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| allocate(int capacity)                      | 从堆空间中分配一个容量大小为capacity的byte数组作为缓冲区的byte数据存储器 |
| allocateDirect(int capacity)                | 是不使用JVM堆栈而是通过操作系统来创建内存块用作缓冲区，它与当前操作系统能够更好的耦合，因此能进一步提高I/O操作速度。但是分配直接缓冲区的系统开销很大，因此只有在缓冲区较大并长期存在，或者需要经常重用时，才使用这种缓冲区 |
| wrap(byte[] array)                          | 这个缓冲区的数据会存放在byte数组中，bytes数组或buff缓冲区任何一方中数据的改动都会影响另一方。其实ByteBuffer底层本来就有一个bytes数组负责来保存buffer缓冲区中的数据，通过allocate方法系统会帮你构造一个byte数组 |
| wrap(byte[] array,  int offset, int length) | 在上一个方法的基础上可以指定偏移量和长度，这个offset也就是包装后byteBuffer的position，而length呢就是limit-position的大小，从而我们可以得到limit的位置为length+position(offset) |

先看一眼Buffer类的属性

```java
public abstract class Buffer {
    // Cached unsafe-access object
    static final Unsafe UNSAFE = Unsafe.getUnsafe();

    /**
     * The characteristics of Spliterators that traverse and split elements
     * maintained in Buffers.
     */
    static final int SPLITERATOR_CHARACTERISTICS =
        Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.ORDERED;

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    // Used by heap byte buffers or direct buffers with Unsafe access
    // For heap byte buffers this field will be the address relative to the
    // array base address and offset into that array. The address might
    // not align on a word boundary for slices, nor align at a long word
    // (8 byte) boundary for byte[] allocations on 32-bit systems.
    // For direct buffers it is the start address of the memory region. The
    // address might not align on a word boundary for slices, nor when created
    // using JNI, see NewDirectByteBuffer(void*, long).
    // Should ideally be declared final
    // NOTE: hoisted here for speed in JNI GetDirectBufferAddress
    long address;

```

```java
public abstract class ByteBuffer
    extends Buffer
    implements Comparable<ByteBuffer>
{

    // These fields are declared here rather than in Heap-X-Buffer in order to
    // reduce the number of virtual method invocations needed to access these
    // values, which is especially costly when coding small buffers.
    //
    final byte[] hb;                  // Non-null only for heap buffers
    final int offset;
    boolean isReadOnly;

```

介绍一下这些属性：

- position ：当前读取的位置。
- mark ：为某一读过的位置做标记，便于某些时候回退到该位置。
- capacity：初始化时候的容量。
- limit ：当写数据到buffer中时，limit一般和capacity相等，当读数据时，limit代表buffer中有效数据的长度。

​    这些属性总是满足以下条件：
　　**0 <= mark <= position <= limit <= capacity**

Buffer中一些值得关注的方法：

##### clear()

写完数据，需要开始读的时候，将postion复位到0，并将limit设为当前postion。

```java
public Buffer clear() {    
    position = 0;    
    limit = capacity;   
    mark = -1;    
    return this;
}
```

##### flip()

写完数据，需要开始读的时候,将limit设为当前postion,将postion复位到0，

```java
    public Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

#### 通道Channel

和流不同，通道是双向的。

通道分为两大类：

- 网络读写（SelectableChannel）
  我们使用的SocketChannel和ServerSocketChannel都是SelectableChannel的子类
- 文件操作（FileChannel）

​    就是Selector会不断地轮询注册在其上的通道（Channel），如果某个通道处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以取得就绪的Channel集合，从而进行后续的IO操作。服务器端只要提供一个线程负责Selector的轮询，就可以接入成千上万个客户端，这就是JDK NIO库的巨大进步。

#### Single Thread

​	NIO由三个部分组成，Client，Selector，Server，把所有的事情都交给Selector去处理。

​	Selector轮询Server和Client的状态，来了Client，就把Clinet插到Server上。Selector不仅负责连接Client，还负责处理Client的消息。

​	![img](https://images0.cnblogs.com/i/288799/201408/180940159566985.jpg)

```java
package nio;

import java.io.BufferedReader;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) {
        try(ServerSocketChannel channel = ServerSocketChannel.open();
            Selector selector = Selector.open()){
            channel.socket().bind(new InetSocketAddress("127.0.0.1",8159));
            channel.configureBlocking(false);

            channel.register(selector, SelectionKey.OP_ACCEPT);//有人连接

            while (true){
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterable = keys.iterator();
                while (iterable.hasNext()){
                    SelectionKey key = iterable.next();
                    iterable.remove();
                    handle(key);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void handle(SelectionKey key) {
        if (key.isAcceptable()){
            handleAccept(key);
        }else if (key.isReadable()){
            handleReadable(key);
        }
    }
//处理请求连接请求
    public static void handleAccept(SelectionKey key){
        try {
            ServerSocketChannel channel = (ServerSocketChannel) key.channel();
            SocketChannel socketChannel = channel.accept();
            socketChannel.configureBlocking(false);

            socketChannel.register(key.selector(),SelectionKey.OP_READ);//监听可读事件

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
//处理可读请求
    public static void handleReadable(SelectionKey key){
        try(SocketChannel channel = (SocketChannel) key.channel()) {
            if  (channel.isConnected()) {
                ByteBuffer byteBuffer = ByteBuffer.allocate(512);
                byteBuffer.clear();
                int len = channel.read(byteBuffer);
                if (len != -1) {
                    System.out.println(new String(byteBuffer.array(), 0, len));
                }
                ByteBuffer b = ByteBuffer.wrap("hello world Client".getBytes());
                channel.write(b);
                byteBuffer.flip();
                channel.write(byteBuffer);
            }else {
                System.out.println("channel is unconnected!");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

```java
    /**
     * Selects a set of keys whose corresponding channels are ready for I/O
     * operations.
     *
     * <p> This method performs a blocking <a href="#selop">selection
     * operation</a>.  It returns only after at least one channel is selected,
     * this selector's {@link #wakeup wakeup} method is invoked, the current
     * thread is interrupted, or the given timeout period expires, whichever
     * comes first.
     *
     * <p> This method does not offer real-time guarantees: It schedules the
     * timeout as if by invoking the {@link Object#wait(long)} method. </p>
     *
     * @param  timeout  If positive, block for up to {@code timeout}
     *                  milliseconds, more or less, while waiting for a
     *                  channel to become ready; if zero, block indefinitely;
     *                  must not be negative
     *
     * @return  The number of keys, possibly zero,
     *          whose ready-operation sets were updated
     *
     * @throws  IOException
     *          If an I/O error occurs
     *
     * @throws  ClosedSelectorException
     *          If this selector is closed
     *
     * @throws  IllegalArgumentException
     *          If the value of the timeout argument is negative
     */
    public abstract int select(long timeout) throws IOException;

    /**
     * Selects a set of keys whose corresponding channels are ready for I/O
     * operations.
     *
     * <p> This method performs a blocking <a href="#selop">selection
     * operation</a>.  It returns only after at least one channel is selected,
     * this selector's {@link #wakeup wakeup} method is invoked, or the current
     * thread is interrupted, whichever comes first.  </p>
     *
     * @return  The number of keys, possibly zero,
     *          whose ready-operation sets were updated
     *
     * @throws  IOException
     *          If an I/O error occurs
     *
     * @throws  ClosedSelectorException
     *          If this selector is closed
     */
    public abstract int select() throws IOException;
```

#### Pool Thread

​	通过之上的例子我们可以发现，一个Selector要做所有的事情实在是太累了。那就给Selector招募一些帮手来帮他处理一下这些channel的具体事务。

```java
package nio;

import java.io.IOException;
import java.net.InetSocketAddress;

import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;

import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class PoolServer {
    ExecutorService executorService = Executors.newFixedThreadPool(50);
    private Selector selector;

    public static void main(String[] args) {
        PoolServer poolServer = new PoolServer();
        poolServer.initServer(8159);
    }

    private void initServer(int port){
        try{
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().bind(new InetSocketAddress(port));
            this.selector = Selector.open();
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            while (true){
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterable = keys.iterator();
                while (iterable.hasNext()){
                    SelectionKey key = iterable.next();
                    iterable.remove();
                    handleKey(key);
                }
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void handleKey(SelectionKey key) {
        if (key.isAcceptable()){
            NIOServer.handleAccept(key);
        }else if (key.isReadable()){
            executorService.execute(new Handler(key));
        }
    }

}

```

### AIO（asynchronous）

​	上面有了线程池的NIO依旧有一个比较大的问题，就是当没有消息过来的时候它会一直轮询，这也很耗了很多不必要的性能。

​	那么就有了一个新主意。

​	当客户端想要连接服务器端口的时候，就让操作系统去通知Selecter，说有人想要链接服务器。然后Selector来决定这个连接该怎么处理。在这里这个Selecter可以有多个，当然这些Selector只负责连接客户端。

​	Seletor连接好了之后，就把channel交给线程池去处理。

​	AIO和NIO在Linux底层实现是相同的，而AIO在windows上有特殊的实现。

​	所以说穿了，AIO放Linux上跑还是需要linux轮询，只是API上看着你没有明着写轮询。

​	但是也是因为这样，windows跑AIO比Linux跑AIO要快。

​	这个范例代码有点问题跑不了

```java
package aio;

import java.io.BufferedReader;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;

public class AIOServer {
    public static void main(String[] args) throws InterruptedException {
        try (final AsynchronousServerSocketChannel channel = AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress("127.0.0.1", 8159))) {
            channel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
                @Override
                public void completed(AsynchronousSocketChannel client, Object attachment) {
                    channel.accept(null, this);
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    client.read(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer attachment) {
                            attachment.flip();
                            System.out.println(new String(attachment.array(),0,result));
                            client.write(attachment);
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer attachment) {
                            exc.printStackTrace();
                        }
                    });

                }

                @Override
                public void failed(Throwable exc, Object attachment) {
                    exc.printStackTrace();
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
        while (true){
            Thread.sleep(1000);
        }
    }
}

```

当然AIO也可以使用线程池，而且API封装的非常好

```java
package aio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousChannelGroup;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class PoolAIOServer {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newCachedThreadPool();
        try {
            AsynchronousChannelGroup group = AsynchronousChannelGroup.withCachedThreadPool(executorService,1);
            try (final AsynchronousServerSocketChannel channel = AsynchronousServerSocketChannel.open()
                    .bind(new InetSocketAddress("127.0.0.1", 8159))) {
                channel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
                    @Override
                    public void completed(AsynchronousSocketChannel client, Object attachment) {
                        channel.accept(null, this);
                        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                        client.read(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                            @Override
                            public void completed(Integer result, ByteBuffer attachment) {
                                attachment.flip();
                                System.out.println(new String(attachment.array(),0,result));
                                client.write(attachment);
                            }

                            @Override
                            public void failed(Throwable exc, ByteBuffer attachment) {
                                exc.printStackTrace();
                            }
                        });

                    }

                    @Override
                    public void failed(Throwable exc, Object attachment) {
                        exc.printStackTrace();
                    }
                });
            } catch (IOException e) {
                e.printStackTrace();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        while (true){
            Thread.sleep(1000);
        }
    }
}

```



## Netty

### 什么是netty

> Netty 是一个利用 Java 的高级网络的能力，隐藏其背后的复杂性而提供一个易于使用的 API 的客户端/服务器框架。

### Netty和Tomcat有什么区别

​	Netty和Tomcat最大的区别就在于通信协议，Tomcat是基于Http协议的，他的实质是一个基于http协议的web容器，但是Netty不一样，他能通过编程自定义各种协议，因为netty能够通过codec自己来编码/解码字节流

### 例子

​	先从一个简单的demo开始：

```java
package netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.logging.LoggingHandler;

import java.net.InetSocketAddress;

public class NettyServer {
    private final EventLoopGroup bossGroup = new NioEventLoopGroup();
    private final EventLoopGroup workerGroup = new NioEventLoopGroup();

    private Channel channel;

    public static void main(String[] args) throws InterruptedException {
        NettyServer nettyServer = new NettyServer();
        ChannelFuture future = nettyServer.run(new InetSocketAddress(8159));
    }

    public ChannelFuture run(InetSocketAddress address) {
        ChannelFuture f = null;
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .handler(new LoggingHandler())
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ServerChannelInit())
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            f = b.bind(address ).sync();
            channel = f.channel();
        } catch (Exception e) {
            System.out.println("Netty start error:");
        } finally {
            if (f != null && f.isSuccess()) {
                System.out.println("Netty server listening " + address.getHostName() + " on port " + address.getPort() + " and ready for connections...");
            } else {
                System.out.println("Netty server start up Error!");
            }
        }
        return f;
    }
}
```

```java
package netty;

import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

public class ServerChannelInit extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        socketChannel.pipeline()
                .addLast(new StringEncoder())
                .addLast(new StringDecoder())
                .addLast(new TestHandler());
    }
}
```

```java
package netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class TestHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channelRegistered");
        System.out.println("Server: channelRegistered");
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channel  Un  registered");
        System.out.println("Server: channel  Un  registered");
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channelActive");
        System.out.println("Server: channelActive");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channel    In  active");
        System.out.println("Server: channel    In  active");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.channel().writeAndFlush("Server: channelRead");
        ctx.channel().writeAndFlush(msg);
        System.out.println("Server: channelRead");
        System.out.println("Client send:"+msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channelReadComplete");
        System.out.println("Server: channelReadComplete");
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        ctx.channel().writeAndFlush("Server: userEventTriggered");
        System.out.println("Server: userEventTriggered");
    }

    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
        ctx.channel().writeAndFlush("Server: channelWritabilityChanged");
        System.out.println("Server: channelWritabilityChanged");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
    }
}
```

### ChannelPipeline

​	可以把ChannelPipeline看成是一个ChandlerHandler的链表，当需要对Channel进行某种处理的时候，Pipeline负责依次调用每一个Handler进行处理。每个Channel都有一个属于自己的Pipeline，调用channel.pipeline()方法可以获得Channel的Pipeline，调用pipeline.channel()方法可以获得pipeline的Channel。

​	![img](https://img-blog.csdn.net/20131211170811796?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	ChannelPipeline的方法有很多，其中一部分是用来管理ChannelHandler的，如下面这些：

```java
ChannelPipeline addFirst(String name, ChannelHandler handler);
ChannelPipeline addLast(String name, ChannelHandler handler);
ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
ChannelPipeline remove(ChannelHandler handler);
ChannelHandler remove(String name);
ChannelHandler removeFirst();
ChannelHandler removeLast();
ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler);
ChannelHandler replace(String oldName, String newName, ChannelHandler newHandler);
ChannelHandler first();
ChannelHandler last();
ChannelHandler get(String name);
```

先重点观察以下我们在代码中一直写的addLast()

```java
public final ChannelPipeline addLast(String name, ChannelHandler handler) {
        return this.addLast((EventExecutorGroup)null, name, handler);
    }
```

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized(this) {
            checkMultiplicity(handler);
            newCtx = this.newContext(group, this.filterName(name, handler), handler);
            this.addLast0(newCtx);
            if (!this.registered) {
                newCtx.setAddPending();
                this.callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    public void run() {
                        DefaultChannelPipeline.this.callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }

        this.callHandlerAdded0(newCtx);
        return this;
    }
```

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    static final InternalLogger logger = InternalLoggerFactory.getInstance(DefaultChannelPipeline.class);
    private static final String HEAD_NAME = generateName0(DefaultChannelPipeline.HeadContext.class);
    private static final String TAIL_NAME = generateName0(DefaultChannelPipeline.TailContext.class);
    private static final FastThreadLocal<Map<Class<?>, String>> nameCaches = new FastThreadLocal<Map<Class<?>, String>>() {
        protected Map<Class<?>, String> initialValue() throws Exception {
            return new WeakHashMap();
        }
    };
    private static final AtomicReferenceFieldUpdater<DefaultChannelPipeline, Handle> ESTIMATOR = AtomicReferenceFieldUpdater.newUpdater(DefaultChannelPipeline.class, Handle.class, "estimatorHandle");
    final AbstractChannelHandlerContext head;
    final AbstractChannelHandlerContext tail;
    private final Channel channel;
    private final ChannelFuture succeededFuture;
    private final VoidChannelPromise voidPromise;
    private final boolean touch = ResourceLeakDetector.isEnabled();
    private Map<EventExecutorGroup, EventExecutor> childExecutors;
    private volatile Handle estimatorHandle;
    private boolean firstRegistration = true;
    private DefaultChannelPipeline.PendingHandlerCallback pendingHandlerCallbackHead;
    private boolean registered;

    protected DefaultChannelPipeline(Channel channel) {
        this.channel = (Channel)ObjectUtil.checkNotNull(channel, "channel");
        this.succeededFuture = new SucceededChannelFuture(channel, (EventExecutor)null);
        this.voidPromise = new VoidChannelPromise(channel, true);
        this.tail = new DefaultChannelPipeline.TailContext(this);
        this.head = new DefaultChannelPipeline.HeadContext(this);
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }
```

### ChannelHandlerContext

![img](https://img-blog.csdn.net/20131211194121609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```java
package io.netty.channel;

import io.netty.util.concurrent.EventExecutor;

final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {
    private final ChannelHandler handler;

    DefaultChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        } else {
            this.handler = handler;
        }
    }

    public ChannelHandler handler() {
        return this.handler;
    }

    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler;
    }

    private static boolean isOutbound(ChannelHandler handler) {
        return handler instanceof ChannelOutboundHandler;
    }
}
```

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap implements ChannelHandlerContext, ResourceLeakHint {
    private static final InternalLogger logger = InternalLoggerFactory.getInstance(AbstractChannelHandlerContext.class);
    
    volatile AbstractChannelHandlerContext next;
    volatile AbstractChannelHandlerContext prev;
    
    private static final AtomicIntegerFieldUpdater<AbstractChannelHandlerContext> HANDLER_STATE_UPDATER = AtomicIntegerFieldUpdater.newUpdater(AbstractChannelHandlerContext.class, "handlerState");
    private static final int ADD_PENDING = 1;
    private static final int ADD_COMPLETE = 2;
    private static final int REMOVE_COMPLETE = 3;
    private static final int INIT = 0;
    private final boolean inbound;
    private final boolean outbound;
    private final DefaultChannelPipeline pipeline;
    private final String name;
    private final boolean ordered;
    final EventExecutor executor;
    private ChannelFuture succeededFuture;
    private Runnable invokeChannelReadCompleteTask;
    private Runnable invokeReadTask;
    private Runnable invokeChannelWritableStateChangedTask;
    private Runnable invokeFlushTask;
    private volatile int handlerState = 0;

    AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name, boolean inbound, boolean outbound) {
        this.name = (String)ObjectUtil.checkNotNull(name, "name");
        this.pipeline = pipeline;
        this.executor = executor;
        this.inbound = inbound;
        this.outbound = outbound;
        this.ordered = executor == null || executor instanceof OrderedEventExecutor;
    }
```



我们可以得到以下的一个模型

![img](https://img-blog.csdn.net/20131212120733046?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

那消息在pipeline中式如何传递的呢？

在刚才的例子中我们可以看到我们使用的方法是``channel.writeAndFlush()``写的，那我们去看看channl.write()这个方法是怎么写的吧。

```java
	public ChannelFuture writeAndFlush(Object msg) {
        return this.pipeline.writeAndFlush(msg);
    }
```

```java
    public final ChannelFuture writeAndFlush(Object msg) {
        return this.tail.writeAndFlush(msg);
        //注意这里是tail
    }
```

因为write是个outbound事件，所以DefaultChannelPipeline直接找到tail部分的context，调用其write()方法

![img](https://img-blog.csdn.net/20131212134216687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```java
public ChannelFuture writeAndFlush(Object msg) {
        return this.writeAndFlush(msg, this.newPromise());
    }
```

```java
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
        if (msg == null) {
            throw new NullPointerException("msg");
        } else if (this.isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            return promise;
        } else {
            this.write(msg, true, promise);
            return promise;
        }
    }
```

```java
	public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
        if (msg == null) {
            throw new NullPointerException("msg");
        } else if (this.isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            return promise;
        } else {
            this.write(msg, true, promise);
            return promise;
        }
```

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
        AbstractChannelHandlerContext next = this.findContextOutbound();
        Object m = this.pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            Object task;
            if (flush) {
                task = AbstractChannelHandlerContext.WriteAndFlushTask.newInstance(next, m, promise);
            } else {
                task = AbstractChannelHandlerContext.WriteTask.newInstance(next, m, promise);
            }

            safeExecute(executor, (Runnable)task, promise, m);
        }

    }
```

```java
private AbstractChannelHandlerContext findContextOutbound() {
        AbstractChannelHandlerContext ctx = this;

        do {
            ctx = ctx.prev;
        } while(!ctx.outbound);

        return ctx;
    }
```

​	context的write()方法沿着context链往前找，直至找到一个outbound类型的context为止，然后调用其invokeWrite()方法：

```java
private void invokeWrite(Object msg, ChannelPromise promise) {
        if (this.invokeHandler()) {
            this.invokeWrite0(msg, promise);
        } else {
            this.write(msg, promise);
        }

    }
```

```java
private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            ((ChannelOutboundHandler)this.handler()).write(this, msg, promise);
        } catch (Throwable var4) {
            notifyOutboundHandlerException(var4, promise);
        }

    }
```

![img](https://img-blog.csdn.net/20131212140450171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![img](https://img-blog.csdn.net/20131212140949687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```jAVA
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        ctx.write(msg, promise);
    }
```

​	这里可以看出handler又调用了ctx的write方法，这样write事件就沿着outbound链继续传播：

![img](https://img-blog.csdn.net/20131212141601265?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenhob28=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### EventExecutor![img](https://img-blog.csdn.net/20131028203048359?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmVyZHk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 参考资料

http://blog.chinaunix.net/uid-28458801-id-4464639.html

http://blog.csdn.net/historyasamirror/article/details/5778378

http://www.ibm.com/developerworks/cn/linux/l-cn-edntwk/

https://yq.aliyun.com/articles/635886?spm=a2c4e.11153940.0.0.c6801cb7hEQhgC

https://blog.csdn.net/zxhoo/article/details/17264263

https://www.cnblogs.com/dolphinX/p/3460545.html





[^1]:recvfrom功能描述：从套接字上接收一个消息。对于recvfrom ，可同时应用于面向连接的和无连接的套接字。如果消息太大，无法完整存放在所提供的缓冲区，根据不同的套接字，多余的字节会丢弃。假如套接字上没有消息可以读取，除了套接字已被设置为非阻塞模式，否则接收调用会等待消息的到来。
[^2]:如果accept成功，那么其返回值是由内核自动生成的一个全新描述符，代表与客户端的TCP连接。一个服务器通常仅仅创建一个监听套接字，它在该服务器生命周期内一直存在。内核为每个由服务器进程接受的客户端连接创建一个已连接套接字。当服务器完成对某个给定的客户端的服务器时，相应的已连接套接字就被关闭。
[^3]:相比于select，epoll最大的好处在于它不会随着监听fd数目的增长而降低效率。因为在内核中的select实现中，它是采用轮询来处理的，轮询的fd数目越多，自然耗时越多。
[^4]:[POSIX](https://baike.baidu.com/item/POSIX)表示[可移植操作系统接口](https://baike.baidu.com/item/可移植操作系统接口/12718298)（Portable Operating System Interface of UNIX，缩写为 POSIX ），POSIX标准定义了操作系统应该为应用程序提供的接口标准，是[IEEE](https://baike.baidu.com/item/IEEE)为要在各种UNIX操作系统上运行的软件而定义的一系列API标准的总称，其正式称呼为IEEE 1003，而国际标准名称为ISO/IEC 9945。