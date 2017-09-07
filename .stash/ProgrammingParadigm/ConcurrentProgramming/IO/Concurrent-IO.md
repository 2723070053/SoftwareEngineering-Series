<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
  - [BIO](#bio)
    - [BIO下的多进程/多线程模式](#bio%E4%B8%8B%E7%9A%84%E5%A4%9A%E8%BF%9B%E7%A8%8B%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%BC%8F)
    - [Leader-Follow模型](#leader-follow%E6%A8%A1%E5%9E%8B)
  - [Unix的5种IO模型](#unix%E7%9A%845%E7%A7%8Dio%E6%A8%A1%E5%9E%8B)
    - [阻塞式I/O](#%E9%98%BB%E5%A1%9E%E5%BC%8Fio)
    - [非阻塞式I/O](#%E9%9D%9E%E9%98%BB%E5%A1%9E%E5%BC%8Fio)
    - [I/O复用（select,poll）](#io%E5%A4%8D%E7%94%A8%EF%BC%88selectpoll%EF%BC%89)
    - [信号驱动式I/O](#%E4%BF%A1%E5%8F%B7%E9%A9%B1%E5%8A%A8%E5%BC%8Fio)
    - [异步I/O](#%E5%BC%82%E6%AD%A5io)
- [IO多路复用](#io%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8)
  - [Reactor模型](#reactor%E6%A8%A1%E5%9E%8B)
    - [核心组件](#%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6)
    - [处理逻辑](#%E5%A4%84%E7%90%86%E9%80%BB%E8%BE%91)
  - [Proactor模型](#proactor%E6%A8%A1%E5%9E%8B)
- [Linux NIO](#linux-nio)
  - [select/poll](#selectpoll)
    - [函数分析](#%E5%87%BD%E6%95%B0%E5%88%86%E6%9E%90)
    - [处理逻辑](#%E5%A4%84%E7%90%86%E9%80%BB%E8%BE%91-1)
  - [epoll/kqueue](#epollkqueue)
    - [select不足与epoll中的改进](#select%E4%B8%8D%E8%B6%B3%E4%B8%8Eepoll%E4%B8%AD%E7%9A%84%E6%94%B9%E8%BF%9B)
    - [函数分析](#%E5%87%BD%E6%95%B0%E5%88%86%E6%9E%90-1)
      - [int epoll_create(int size);](#int-epoll_createint-size)
      - [int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);](#int-epoll_ctlint-epfd-int-op-int-fd-struct-epoll_event-event)
      - [int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);](#int-epoll_waitint-epfd-struct-epoll_event--events-int-maxevents-int-timeout)
    - [处理逻辑](#%E5%A4%84%E7%90%86%E9%80%BB%E8%BE%91-2)
  - [Demo](#demo)
    - [阻塞式网络编程接口](#%E9%98%BB%E5%A1%9E%E5%BC%8F%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E6%8E%A5%E5%8F%A3)
    - [select](#select)
    - [epoll](#epoll)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Introduction

在传统的网络服务器的构建中，IO模式会按照Blocking/Non-Blocking、Synchronous/Asynchronous这两个标准进行分类，其中Blocking与Synchronous基本上一个意思，而NIO与Async的区别在于NIO强调的是Polling(轮询)，而Async强调的是Notification(通知)。譬如在一个典型的单进程单线程Socket接口中，阻塞型的接口必须在上一个Socket连接关闭之后才能接入下一个Socket连接。而对于NIO的Socket而言，Server Application会从内核获取到一个特殊的"Would Block"错误信息，但是并不会阻塞到等待发起请求的Socket Client停止。一般来说，在Linux系统中可以通过调用独立的`select`或者`poll`方法来遍历所有读取好的数据，并且进行写操作。而对于异步Socket而言(譬如Windows中的Sockets或者.Net中实现的Sockets模型)，Server Application会告诉IO Framework去读取某个Socket数据，在数据读取完毕之后IO Framework会自动地调用你的回调(也就是通知应用程序本身数据已经准备好了)。以IO多路复用中的Reactor与Proactor模型为例，非阻塞的模型是需要应用程序本身处理IO的，而异步模型则是由Kernel或者Framework将数据准备好读入缓冲区中，应用程序直接从缓冲区读取数据。

总结一下：

- 同步阻塞：在此种方式下，用户进程在发起一个IO操作以后，必须等待IO操作的完成，只有当真正完成了IO操作以后，用户进程才能运行。

- 同步非阻塞：在此种方式下，用户进程发起一个IO操作以后边可返回做其它事情，但是用户进程需要时不时的询问IO操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的CPU资源浪费。

- 异步非阻塞：在此种模式下，用户进程只需要发起一个IO操作然后立即返回，等IO操作真正的完成以后，应用程序会得到IO操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的IO读写操作，因为真正的IO读取或者写入操作已经由内核完成了。


而在并发IO的问题中，较常见的就是所谓的C10K问题，即有10000个客户端需要连上一个服务器并保持TCP连接，客户端会不定时的发送请求给服务器，服务器收到请求后需及时处理并返回结果。



## BIO

BIO即同步阻塞式I/O，是面向流的，阻塞式的，串行的一个过程。对每一个客户端的socket连接，都需要一个线程来处理，而且在此期间这个线程一直被占用，直到socket关闭。

![](http://7xkt0f.com1.z0.glb.clouddn.com/ed484965-b93f-4f6a-aa44-419f042d5872.png)

采用BIO通信模型的服务端，通常由一个独立的Acceptor线程负责监听客户端的连接，接收到客户端连接之后为客户端连接创建一个新的线程处理请求消息，处理完成之后，返回应答消息给客户端，线程销毁，这就是典型的一请求一应答模型。该架构最大的问题就是不具备弹性伸缩能力，当并发访问量增加后，服务端的线程个数和并发访问数成线性正比，由于线程是JAVA虚拟机非常宝贵的系统资源，当线程数膨胀之后，系统的性能急剧下降，随着并发量的继续增加，可能会发生句柄溢出、线程堆栈溢出等问题，并导致服务器最终宕机。

还有一些并不是由于并发数增加而导致的系统负载增加：连接服务器的一些客户端，由于网络或者自身性能处理的问题，接收端从socket读取数据的速度跟不上发送端写入数据的速度。 而在TCP/IP网络编程过程中，已经发送出去的数据依然需要暂存在send buffer，只有收到对方的ack，kernel才从buffer中清除这一部分数据，为后续发送数据腾出空间。接收端将收到的数据暂存在receive buffer中，自动进行确认。但如果socket所在的进程不及时将数据从receive buffer中取出，最终导致receive buffer填满，由于TCP的滑动窗口和拥塞控制，接收端会阻止发送端向其发送数据。

作为发送端，服务器由于迟迟不能释放被占用的线程，导致内存占用率不断升高，堆回收的效率越来越低，导致Full GC，最终导致服务宕机。



### BIO下的多进程/多线程模式

本文初提及，BIO的一个缺陷在于某个Socket在其连接到断上期间会独占线程，那么解决这个问题的一个朴素想法就是利用多进程多线程的办法，即是创建一个新的线程来处理新的连接，这样就保证了并发IO的实现。本节即是对这种思路进行分析。

最早的服务器端程序都是通过多进程、多线程来解决并发IO的问题。进程模型出现的最早，从Unix系统诞生就开始有了进程的概念。最早的服务器端程序一般都是Accept一个客户端连接就创建一个进程，然后子进程进入循环同步阻塞地与客户端连接进行交互，收发处理数据。



![](http://rango.swoole.com/static/io/4.png)

![](http://rango.swoole.com/static/io/4.png)

多线程模式出现要晚一些，线程与进程相比更轻量，而且线程之间是共享内存堆栈的，所以不同的线程之间交互非常容易实现。比如聊天室这样的程序，客户端连接之间可以交互，比聊天室中的玩家可以任意的其他人发消息。用多线程模式实现非常简单，线程中可以直接读写某一个客户端连接。而多进程模式就要用到管道、消息队列、共享内存实现数据交互，统称进程间通信（IPC）复杂的技术才能实现。

![](http://rango.swoole.com/static/io/1.png)

多进程/线程模型的流程如下：



1. 创建一个 socket，绑定服务器端口（bind），监听端口（listen），在PHP中用stream_socket_server一个函数就能完成上面3个步骤，当然也可以使用php sockets扩展分别实现。

2. 进入while循环，阻塞在accept操作上，等待客户端连接进入。此时程序会进入随眠状态，直到有新的客户端发起connect到服务器，操作系统会唤醒此进程。accept函数返回客户端连接的socket

3. 主进程在多进程模型下通过fork（php: pcntl_fork）创建子进程，多线程模型下使用pthread_create（php: new Thread）创建子线程。下文如无特殊声明将使用进程同时表示进程/线程。

4. 子进程创建成功后进入while循环，阻塞在recv（php: fread）调用上，等待客户端向服务器发送数据。收到数据后服务器程序进行处理然后使用send（php: fwrite）向客户端发送响应。长连接的服务会持续与客户端交互，而短连接服务一般收到响应就会close。

5. 当客户端连接关闭时，子进程退出并销毁所有资源。主进程会回收掉此子进程。



### Leader-Follow模型

![](http://www.dengshenyu.com/assets/redis-reactor/reactor-mode2.png)

上文描述的多进程/多线程模型最大的问题是，进程/线程创建和销毁的开销很大。所以上面的模式没办法应用于非常繁忙的服务器程序。对应的改进版解决了此问题，这就是经典的**Leader-Follower**模型。

![](http://rango.swoole.com/static/io/2.png)

它的特点是程序启动后就会创建N个进程。每个子进程进入Accept，等待新的连接进入。当客户端连接到服务器时，其中一个子进程会被唤醒，开始处理客户端请求，并且不再接受新的TCP连接。当此连接关闭时，子进程会释放，重新进入Accept，参与处理新的连接。这个模型的优势是完全可以复用进程，没有额外消耗，性能非常好。很多常见的服务器程序都是基于此模型的，比如Apache、PHP-FPM。



多进程模型也有一些缺点。

1. 这种模型严重依赖进程的数量解决并发问题，一个客户端连接就需要占用一个进程，工作进程的数量有多少，并发处理能力就有多少。操作系统可以创建的进程数量是有限的。

2. 启动大量进程会带来额外的进程调度消耗。数百个进程时可能进程上下文切换调度消耗占CPU不到1%可以忽略不接，如果启动数千甚至数万个进程，消耗就会直线上升。调度消耗可能占到CPU的百分之几十甚至100%。



另外有一些场景多进程模型无法解决，比如即时聊天程序（IM），一台服务器要同时维持上万甚至几十万上百万的连接（经典的C10K问题），多进程模型就力不从心了。还有一种场景也是多进程模型的软肋。通常Web服务器启动100个进程，如果一个请求消耗100ms，100个进程可以提供1000qps，这样的处理能力还是不错的。但是如果请求内要调用外网Http接口，像QQ、微博登录，耗时会很长，一个请求需要10s。那一个进程1秒只能处理0.1个请求，100个进程只能达到10qps，这样的处理能力就太差了。









## Unix的5种IO模型

> [Asynchronous and non-blocking IO](http://blog.omega-prime.co.uk/?p=155)



Unix的5种I/O模型：阻塞式I/O, 非阻塞式I/O，I/O复用模型，信号驱动式I/O和异步I/O。



### 阻塞式I/O



![blocking_io](https://lukangping.gitbooks.io/java-nio/content/resources/blocking_io.jpg)



### 非阻塞式I/O



![nonblocking_io](https://lukangping.gitbooks.io/java-nio/content/resources/nonblocking_io.jpg)



### I/O复用（select,poll）



I/O多路复用通过把多个I/O的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况可以同时处理多个客户端请求。 目前支持I/O多路复用的**系统调用**有select，pselect，poll，epoll，在linux网络编程过程中，很长一段时间都是用select做轮询和网络事件通知，然而select的一些固有缺陷导致了它的应用受到了很大的限制，最终linux不得不载新的内核版本中寻找select的替代方案，最终选择了epoll。



![multplexing_io](https://lukangping.gitbooks.io/java-nio/content/resources/multiplexing_io.jpg)



### 信号驱动式I/O



![signal_driven](https://lukangping.gitbooks.io/java-nio/content/resources/signal_driven.jpg)



### 异步I/O

> Asynchronous IO refers to an interface where you supply a callback to an IO operation, which is invoked when the operation completes. This invocation often happens to an entirely different thread to the one that originally made the request, but this is not necessarily the case. Asynchronous IO is a manifestation of the ["proactor" pattern](https://en.wikipedia.org/wiki/Proactor_pattern).



![asynchronous_io](https://lukangping.gitbooks.io/java-nio/content/resources/asynchronous_io.jpg)



并发IO问 题一直是后端编程中的技术挑战，从最早的同步阻塞Fork进程，到多进程/多线程，到现在的异步IO、协程。









# IO多路复用

IO多路复用技术通俗阐述，即是由一个线程轮询每个连接，如果某个连接有请求则处理请求，没有请求则处理下一个连接。首先来看下可读事件与可写事件：

当如下**任一**情况发生时，会产生套接字的**可读**事件：



- 该套接字的接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的大小；

- 该套接字的读半部关闭（也就是收到了FIN），对这样的套接字的读操作将返回0（也就是返回EOF）；

- 该套接字是一个监听套接字且已完成的连接数不为0；

- 该套接字有错误待处理，对这样的套接字的读操作将返回-1。



当如下**任一**情况发生时，会产生套接字的**可写**事件：

- 该套接字的发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水位标记的大小；

- 该套接字的写半部关闭，继续写会产生SIGPIPE信号；

- 非阻塞模式下，connect返回之后，该套接字连接成功或失败；

- 该套接字有错误待处理，对这样的套接字的写操作将返回-1。






## Reactor模型

Reactor模型在Linux系统中的具体实现即是select/poll/epoll/kqueue，像Redis中即是采用了Reactor模型实现了单进程单线程高并发。Reactor模型的理论基础可以参考[reactor-siemens](http://www.dre.vanderbilt.edu/%7Eschmidt/PDF/reactor-siemens.pdf)

### 核心组件

![](http://www.dengshenyu.com/assets/redis-reactor/reactor-mode3.png)

- Handles ：表示操作系统管理的资源，我们可以理解为fd。



- Synchronous Event Demultiplexer ：同步事件分离器，阻塞等待Handles中的事件发生。



- Initiation Dispatcher ：初始分派器，作用为添加Event handler（事件处理器）、删除Event handler以及分派事件给Event handler。也就是说，Synchronous Event Demultiplexer负责等待新事件发生，事件发生时通知Initiation Dispatcher，然后Initiation Dispatcher调用event handler处理事件。



- Event Handler ：事件处理器的接口



- Concrete Event Handler ：事件处理器的实际实现，而且绑定了一个Handle。因为在实际情况中，我们往往不止一种事件处理器，因此这里将事件处理器接口和实现分开，与C++、Java这些高级语言中的多态类似。



### 处理逻辑

Reactor模型的基本的处理逻辑为：

（1）我们注册Concrete Event Handler到Initiation Dispatcher中。

（2）Initiation Dispatcher调用每个Event Handler的get_handle接口获取其绑定的Handle。

（3）Initiation Dispatcher调用handle_events开始事件处理循环。在这里，Initiation Dispatcher会将步骤2获取的所有Handle都收集起来，使用Synchronous Event Demultiplexer来等待这些Handle的事件发生。

（4）当某个（或某几个）Handle的事件发生时，Synchronous Event Demultiplexer通知Initiation Dispatcher。

（5）Initiation Dispatcher根据发生事件的Handle找出所对应的Handler。

（6）Initiation Dispatcher调用Handler的handle_event方法处理事件。



时序图如下：

![](http://www.dengshenyu.com/assets/redis-reactor/reactor-mode4.png)


抽象来说，Reactor有4个核心的操作：



1. add添加socket监听到reactor，可以是listen socket也可以使客户端socket，也可以是管道、eventfd、信号等

2. set修改事件监听，可以设置监听的类型，如可读、可写。可读很好理解，对于listen socket就是有新客户端连接到来了需要accept。对于客户端连接就是收到数据，需要recv。可写事件比较难理解一些。一个SOCKET是有缓存区的，如果要向客户端连接发送2M的数据，一次性是发不出去的，操作系统默认TCP缓存区只有256K。一次性只能发256K，缓存区满了之后send就会返回EAGAIN错误。这时候就要监听可写事件，在纯异步的编程中，必须去监听可写才能保证send操作是完全非阻塞的。

3. del从reactor中移除，不再监听事件

4. callback就是事件发生后对应的处理逻辑，一般在add/set时制定。C语言用函数指针实现，JS可以用匿名函数，PHP可以用匿名函数、对象方法数组、字符串函数名。



Reactor只是一个事件发生器，实际对socket句柄的操作，如connect/accept、send/recv、close是在callback中完成的。具体编码可参考下面的伪代码：



![](http://rango.swoole.com/static/io/6.png)



Reactor模型还可以与多进程、多线程结合起来用，既实现异步非阻塞IO，又利用到多核。目前流行的异步服务器程序都是这样的方式：如

- Nginx：多进程Reactor

- Nginx+Lua：多进程Reactor+协程

- Golang：单线程Reactor+多线程协程

- Swoole：多线程Reactor+多进程Worker



协程从底层技术角度看实际上还是异步IO Reactor模型，应用层自行实现了任务调度，借助Reactor切换各个当前执行的用户态线程，但用户代码中完全感知不到Reactor的存在。









## Proactor模型

Reactor和Proactor模式的主要区别就是真正的读取和写入操作是有谁来完成的，Reactor中需要应用程序自己读取或者写入数据，而 Proactor模式中，应用程序不需要进行实际的读写过程，它只需要从缓存区读取或者写入即可，操作系统会读取缓存区或者写入缓存区到真正的IO设备。Proactor模型的基本处理逻辑如下：



1. 应用程序初始化一个异步读取操作，然后注册相应的事件处理器，此时事件处理器不关注读取就绪事件，而是关注读取完成事件，这是区别于Reactor的关键。

2. 事件分离器等待读取操作完成事件。

3. 在事件分离器等待读取操作完成的时候，操作系统调用内核线程完成读取操作（异步IO都是操作系统负责将数据读写到应用传递进来的缓冲区供应用程序操作，操作系统扮演了重要角色），并将读取的内容放入用户传递过来的缓存区中。这也是区别于Reactor的一点，Proactor中，应用程序需要传递缓存区。

4. 事件分离器捕获到读取完成事件后，激活应用程序注册的事件处理器，事件处理器直接从缓存区读取数据，而不需要进行实际的读取操作。






# Linux NIO

select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。



## select/poll

![](http://images.cnitblog.com/blog/305504/201308/17201205-8ac47f1f1fcd4773bd4edd947c0bb1f4.png)








### 函数分析



```

select(int nfds, fd_set *r, fd_set *w, fd_set *e, struct timeval *timeout)

```



- `maxfdp1`表示该进程中描述符的总数。



- `fd_set`则是配合`select`模型的重点数据结构，用来存放描述符的集合。



- `timeout`表示`select`返回需要等待的时间。



对于select()，我们需要传3个集合，r，w和e。其中，r表示我们对哪些fd的可读事件感兴趣，w表示我们对哪些fd的可写事件感兴趣。每个集合其实是一个bitmap，通过0/1表示我们感兴趣的fd。例如，我们对于fd为6的可读事件感兴趣，那么r集合的第6个bit需要被 设置为1。这个系统调用会阻塞，直到我们感兴趣的事件（至少一个）发生。调用返回时，内核同样使用这3个集合来存放fd实际发生的事件信息。也就是说，调 用前这3个集合表示我们感兴趣的事件，调用后这3个集合表示实际发生的事件。



select为最早期的UNIX系统调用，它存在4个问题：1）这3个bitmap有大小限制（FD_SETSIZE，通常为1024）；2）由于 这3个集合在返回时会被内核修改，因此我们每次调用时都需要重新设置；3）我们在调用完成后需要扫描这3个集合才能知道哪些fd的读/写事件发生了，一般情况下全量集合比较大而实际发生读/写事件的fd比较少，效率比较低下；4）内核在每次调用都需要扫描这3个fd集合，然后查看哪些fd的事件实际发生， 在读/写比较稀疏的情况下同样存在效率问题。



由于存在这些问题，于是人们对select进行了改进，从而有了poll。

```

poll(struct pollfd *fds, int nfds, int timeout)
struct pollfd {
    int fd;     
    short events;
    short revents;
    }
```

poll调用需要传递的是一个pollfd结构的数组，调用返回时结果信息也存放在这个数组里面。 pollfd的结构中存放着fd、我们对该fd感兴趣的事件(events)以及该fd实际发生的事件(revents)。poll传递的不是固定大小的 bitmap，因此select的问题1解决了；poll将感兴趣事件和实际发生事件分开了，因此select的问题2也解决了。但select的问题3和问题4仍然没有解决。

### 处理逻辑

总的来说，Select模型的内核的处理逻辑为：



（1）使用copy_from_user从用户空间拷贝fd_set到内核空间

（2）注册回调函数__pollwait

（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）

（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。

（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll 来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数 据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。

（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。

（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是 current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout 指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。

（8）把fd_set从内核空间拷贝到用户空间。

多客户端请求服务端，服务端与各客户端保持长连接并且能接收到各客户端数据大体思路如下：



（1）初始化`readset`，并且将服务端监听的描述符添加到`readset`中去。



（2）然后`select`阻塞等待`readset`集合中是否有描述符可读。



（3）如果是服务端描述符可读，那么表示有新客户端连接上。通过`accept`接收客户端的数据，并且将客户端描述符添加到一个数组`client`中，以便二次遍历的时候使用。



（4）执行第二次循环，此时通过`for`循环把`client`中的有效的描述符都添加到`readset`中去。



（5）`select`再次阻塞等待`readset`集合中是否有描述符可读。



（6）如果此时已经连接上的某个客户端描述符有数据可读，则进行数据读取。




## epoll/kqueue

### select不足与epoll中的改进

> [select、poll、epoll之间的区别总结[整理]](http://www.cnblogs.com/Anker/p/3265058.html)



select与poll问题的关键在于无状态。对于每一次系统调用，内核不会记录下任何信息，所以每次调用都需要重复传递相同信息。总结而言，select/poll模型存在的问题即是每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大并且每次都需要在内核遍历传递进来的所有的fd，这个开销在fd很多时候也很大。讨论epoll对于select/poll改进的时候，epoll和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。对于上面所说的select/poll的缺点，主要是在epoll_ctl中解决的，每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把 current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会 把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用 schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。



（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。

（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。




### 函数分析

#### int epoll_create(int size);


创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个 参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在 linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被 耗尽。


#### int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);


epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
- EPOLL_CTL_ADD：注册新的fd到epfd中；
- EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
- EPOLL_CTL_DEL：从epfd中删除一个fd；


第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
```
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```
events可以是以下几个宏的集合：
- EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
- EPOLLOUT：表示对应的文件描述符可以写；
- EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
- EPOLLERR：表示对应的文件描述符发生错误；
- EPOLLHUP：表示对应的文件描述符被挂断；
- EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
- EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里


#### int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);


等 待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有 说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

### 处理逻辑

使用epoll 来实现服务端同时接受多客户端长连接数据时，的大体步骤如下：

（1）使用epoll_create创建一个 epoll 的句柄，下例中我们命名为epollfd。

（2）使用epoll_ctl把服务端监听的描述符添加到epollfd指定的 epoll 内核事件表中，监听服务器端监听的描述符是否可读。

（3）使用epoll_wait阻塞等待注册的服务端监听的描述符可读事件的发生。

（4）当有新的客户端连接上服务端时，服务端监听的描述符可读，则epoll_wait返回，然后通过accept获取客户端描述符。

（5）使用epoll_ctl把客户端描述符添加到epollfd指定的 epoll 内核事件表中，监听服务器端监听的描述符是否可读。

（6）当客户端描述符有数据可读时，则触发epoll_wait返回，然后执行读取。



几乎所有的epoll模型编码都是基于以下模板：

```

    for( ; ; )
    {
        nfds = epoll_wait(epfd,events,20,500);
        for(i=0;i<nfds;++i)
        {
            if(events[i].data.fd==listenfd) //有新的连接
            {
                connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
                ev.data.fd=connfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
            }
            else if( events[i].events&EPOLLIN ) //接收到数据，读socket
            {
                n = read(sockfd, line, MAXLINE)) < 0    //读
                ev.data.ptr = md;     //md为自定义类型，添加数据
                ev.events=EPOLLOUT|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
            }
            else if(events[i].events&EPOLLOUT) //有数据待发送，写socket
            {
                struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
                sockfd = md->fd;
                send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
                ev.data.fd=sockfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
            }
            else
            {
                //其他的处理
            }
        }
    }

```



## Demo

> 本部分代码实现参考[可能是最接地气的 I/O 多路复用小结](https://mengkang.net/726.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)



### 阻塞式网络编程接口

```

    #include <stdio.h>

    #include <unistd.h>

    #include <sys/types.h>

    #include <sys/socket.h>

    #include <arpa/inet.h>

    #include <netinet/in.h>

    #include <string.h>



    #define SERV_PORT 8031

    #define BUFSIZE 1024



    int main(void)

    {

        int lfd, cfd;

        struct sockaddr_in serv_addr,clin_addr;

        socklen_t clin_len;

        char recvbuf[BUFSIZE];

        int len;



        lfd = socket(AF_INET,SOCK_STREAM,0);



        serv_addr.sin_family = AF_INET;

        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

        serv_addr.sin_port = htons(SERV_PORT);



        bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));



        listen(lfd, 128);



        while(1){

            clin_len = sizeof(clin_addr);

            cfd = accept(lfd, (struct sockaddr *)&clin_addr, &clin_len);

            while(len = read(cfd,recvbuf,BUFSIZE)){

                write(STDOUT_FILENO,recvbuf,len);//把客户端输入的内容输出在终端

                // 只有当客户端输入 stop 就停止当前客户端的连接

                if (strncasecmp(recvbuf,"stop",4) == 0){

                    close(cfd);

                    break;

                }

            }

        }      

        close(lfd);

        return 0;

    }

```

编译运行之后，开启两个终端使用命令`nc 10.211.55.4 8031`（假如服务器的 ip 为 10.211.55.4）。如果首先连上的客户端一直不输入`stop`加回车，那么第二个客户端输入任何内容都不会被客户端接收。如下图所示



![](https://mengkang.net/upload/image/2016/0405/1459863994163365.png)



输入`abc`的是先连接上的，在其输入`stop`之前，后面连接上的客户端输入`123`并不会被服务端收到。也就是说一直阻塞在第一个客户端那里。当第一个客户端输入`stop`之后，服务端才收到第二个客户端的发送过来的数据。



![](https://mengkang.net/upload/image/2016/0405/1459864077691859.png)

### select

```

    #include <stdio.h>

    #include <stdlib.h>

    #include <unistd.h>

    #include <errno.h>

    #include <sys/types.h>

    #include <sys/socket.h>

    #include <arpa/inet.h>

    #include <netinet/in.h>

    #include <fcntl.h>

    #include <sys/select.h>

    #include <sys/time.h>

    #include <string.h>



    #define SERV_PORT     8031

    #define BUFSIZE       1024

    #define FD_SET_SIZE   128



    int main(void) {

        int lfd, cfd, maxfd, scokfd, retval;

        struct sockaddr_in serv_addr, clin_addr;



        socklen_t clin_len; // 地址信息结构体大小



        char recvbuf[BUFSIZE];

        int len;



        fd_set read_set, read_set_init;



        int client[FD_SET_SIZE];

        int i;

        int maxi = -1;





        if ((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {

            perror("套接字描述符创建失败");

            exit(1);

        }



        int opt = 1;

        setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));



        memset(&serv_addr, 0, sizeof(serv_addr));

        serv_addr.sin_family = AF_INET;

        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

        serv_addr.sin_port = htons(SERV_PORT);



        if (bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) == -1) {

            perror("绑定失败");

            exit(1);

        }



        if (listen(lfd, FD_SET_SIZE) == -1) {

            perror("监听失败");

            exit(1);

        }



        maxfd = lfd;



        for (i = 0; i < FD_SET_SIZE; ++i) {

            client[i] = -1;

        }



        FD_ZERO(&read_set_init);

        FD_SET(lfd, &read_set_init);



        while (1) {

            // 每次循环开始时，都初始化 read_set

            read_set = read_set_init;



            // 因为上一步 read_set 已经重置，所以需要已连接上的客户端 fd （由上次循环后产生）重新添加进 read_set

            for (i = 0; i < FD_SET_SIZE; ++i) {

                if (client[i] > 0) {

                    FD_SET(client[i], &read_set);

                }

            }



            printf("select 等待\n");

            // 这里会阻塞，直到 read_set 中某一个 fd 有数据可读才返回，注意 read_set 中除了客户端 fd 还有服务端监听的 fd

            retval = select(maxfd + 1, &read_set, NULL, NULL, NULL);

            if (retval == -1) {

                perror("select 错误\n");

            } else if (retval == 0) {

                printf("超时\n");

                continue;

            }

            printf("select 返回\n");



            //------------------------------------------------------------------------------------------------

            // 用 FD_ISSET 来判断 lfd (服务端监听的fd)是否可读。只有当新的客户端连接时，lfd 才可读

            if (FD_ISSET(lfd, &read_set)) {

                clin_len = sizeof(clin_addr);

                if ((cfd = accept(lfd, (struct sockaddr *) &clin_addr, &clin_len)) == -1) {

                    perror("接收错误\n");

                    continue;

                }



                for (i = 0; i < FD_SET_SIZE; ++i) {

                    if (client[i] < 0) {

                        // 把客户端 fd 放入 client 数组

                        client[i] = cfd;

                        printf("接收client[%d]一个请求来自于: %s:%d\n", i, inet_ntoa(clin_addr.sin_addr), ntohs(clin_addr.sin_port));

                        break;

                    }

                }



                // 最大的描述符值也要重新计算

                maxfd = (cfd > maxfd) ? cfd : maxfd;

                // maxi 用于下面遍历所有有效客户端 fd 使用，以免遍历整个 client 数组

                maxi = (i >= maxi) ? ++i : maxi;

            }

            //------------------------------------------------------------------------------------------------



            for (i = 0; i < maxi; ++i) {

                if (client[i] < 0) {

                    continue;

                }



                // 如果客户端 fd 中有数据可读，则进行读取

                if (FD_ISSET(client[i], &read_set)) {

                    // 注意：这里没有使用 while 循环读取，如果使用 while 循环读取，则有阻塞在一个客户端了。

                    // 可能你会想到如果一次读取不完怎么办？

                    // 读取不完时，在循环到 select 时 由于未读完的 fd 还有数据可读，那么立即返回，然后到这里继续读取，原来的 while 循环读取直接提到最外层的 while(1) + select 来判断是否有数据继续可读

                    len = read(client[i], recvbuf, BUFSIZE);

                    if (len > 0) {

                        write(STDOUT_FILENO, recvbuf, len);

                    }else if (len == 0){

                        // 如果在客户端 ctrl+z

                        close(client[i]);

                        printf("clinet[%d] 连接关闭\n", i);

                        FD_CLR(client[i], &read_set);

                        client[i] = -1;

                        break;

                    }

                }

            }



        }



        close(lfd);



        return 0;

    }

```

![](https://mengkang.net/upload/image/2016/0407/1459997845662935.png)



### epoll

```

    #include <stdio.h>

    #include <stdlib.h>

    #include <unistd.h>

    #include <errno.h>

    #include <sys/types.h>

    #include <sys/socket.h>

    #include <arpa/inet.h>

    #include <netinet/in.h>

    #include <fcntl.h>

    #include <sys/epoll.h>

    #include <sys/time.h>

    #include <string.h>



    #define SERV_PORT           8031

    #define MAX_EVENT_NUMBER    1024

    #define BUFFER_SIZE         10





    /* 将文件描述符 fd 上的 EPOLLIN 注册到 epollfd 指示的 epoll 内核事件表中 */

    void addfd(int epollfd, int fd) {

        struct epoll_event event;

        event.data.fd = fd;

        event.events = EPOLLIN | EPOLLET;

        epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);

        int old_option = fcntl(fd, F_GETFL);

        int new_option = old_option | O_NONBLOCK;

        fcntl(fd, F_SETFL, new_option);

    }



    void et(struct epoll_event *events, int number, int epollfd, int listenfd) {

        char buf[BUFFER_SIZE];

        for (int i = 0; i < number; ++i) {

            int sockfd = events[i].data.fd;

            if (sockfd == listenfd) {

                struct sockaddr_in client_address;

                socklen_t length = sizeof(client_address);

                int connfd = accept(listenfd, (struct sockaddr *) &client_address, &length);

                printf("接收一个请求来自于: %s:%d\n", inet_ntoa(client_address.sin_addr), ntohs(client_address.sin_port));



                addfd(epollfd, connfd);

            } else if (events[i].events & EPOLLIN) {

                /* 这段代码不会被重复触发，所以我们循环读取数据，以确保把 socket 缓存中的所有数据读取*/

                while (1) {

                    memset(buf, '\0', BUFFER_SIZE);

                    int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);

                    if (ret < 0) {

                        /* 对非阻塞 I/O ，下面的条件成立表示数据已经全部读取完毕。此后 epoll 就能再次触发 sockfd 上的 EPOLLIN 事件，以驱动下一次读操作 */

                        if ((errno == EAGAIN) || (errno == EWOULDBLOCK)) {

                            printf("read later\n");

                            break;

                        }

                        close(sockfd);

                        break;

                    } else if (ret == 0) {

                        printf("断开一个连接\n");

                        close(sockfd);

                    } else {

                        printf("get %d bytes of content: %s\n", ret, buf);

                    }

                }

            }

        }

    }





    int main(void) {

        int lfd, epollfd,ret;

        struct sockaddr_in serv_addr;



        if ((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {

            perror("套接字描述符创建失败");

            exit(1);

        }



        int opt = 1;

        setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));



        memset(&serv_addr, 0, sizeof(serv_addr));

        serv_addr.sin_family = AF_INET;

        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

        serv_addr.sin_port = htons(SERV_PORT);



        if (bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) == -1) {

            perror("绑定失败");

            exit(1);

        }



        if (listen(lfd, 5) == -1) {

            perror("监听失败");

            exit(1);

        }



        struct epoll_event events[MAX_EVENT_NUMBER];

        if ((epollfd = epoll_create(5)) == -1) {

            perror("创建失败");

            exit(1);

        }



        // 把服务器端 lfd 添加到 epollfd 指定的 epoll 内核事件表中，添加一个 lfd 可读的事件

        addfd(epollfd, lfd);

        while (1) {

            // 阻塞等待新客户端的连接或者客户端的数据写入，返回需要处理的事件数目

            if ((ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1)) < 0) {

                perror("epoll_wait失败");

                exit(1);

            }



            et(events, ret, epollfd, lfd);

        }



        close(lfd);

        return 0;

    }

```
