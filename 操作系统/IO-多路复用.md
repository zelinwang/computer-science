# I/O 多路复用

select，poll，epoll都是多路复用的机制，就是通过一个线程，监视多个IO，一旦某个文件描述符就绪，就通知应用程序进行对应的读写操作。但是他们都是同步IO，他们都需要在数据准备完毕后自己进行读写，也就是这个过程是阻塞的，而异步IO会自行进行读写操作，负责把数据从内核拷贝到用户空间。

## 一、Select

```c++
struct timeval
{
    time_t tv_sec;             //秒
    long tv_usec;              //微秒
};

int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述符就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过遍历fdset，来找到就绪的描述符。

**时间复杂度O(N)**

**缺点：**

1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大

2. 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大

3. select支持的文件描述符数量小，默认是1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是也会造成效率降低。

## 二、Poll

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch 要监视的event*/
    short revents; /* returned events witnessed 发生的event*/
};//不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。

int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，其他的都差不多,管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll没有最大文件描述符数量的限制。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

时间复杂度O(N)

## 三、Epoll

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

时间复杂度O(1)

### 一）内部机制

epoll的实现重点有几个关键词**红黑树**，**回调函数**，**rdlist**

调用epoll_create会返回一个文件描述符fd，里面创建好了用于监听事件的红黑树和用于存放就绪事件的双向链表rdlist。

在调用epoll_ctl时，会将监听事件存放到红黑树（红黑树本身插入和删除时间复杂度为LogN，效率是很高的）中，并且与网络适配器绑定一个回调函数。

当监听的事件发生时，将调用之前绑定的回调函数，这个回调函数的作用是将该事件存放到rdlist中去。

调用epoll_wait函数，去查找rdlist中的就绪事件，存放在传入的参数events中，并返回就绪事件的数量。

### 二）操作过程

```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
 
//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

int epoll_create(int size);
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

int epoll_ctl(int epfd, int op, int fd, struct epoll_event \*event)；
函数是对指定描述符fd执行op操作。
\- epfd：是epoll_create()的返回值。
\- op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。
\- fd：是需要监听的fd（文件描述符）

int epoll_wait(int epfd, struct epoll_event \* events, int maxevents, int timeout);
等待epfd上的io事件，最多返回maxevents个事件。
参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

### 三）工作模式

epoll对文件描述符的操作有两种模式：**LT（level trigger）**和**ET（edge trigger）**。LT模式是默认模式

**1、LT模式（level triggered水平触发）**

1. LT是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的。
2. 内部实现方法：将从rdlist上取出的事件重新放回去，再次调用epoll_wait仍会继续通知，直到用户处理完成主动关闭fd。
3. 在该模式下有两种工作方式：阻塞和非阻塞。
   这两种工作方式并没有很明显的区别，因为只要被唤醒就一定有事件可读或者可写，阻塞模式下，一般情况并不会进入阻塞状态。在非阻塞模式下，会使用循环的方式进行读/写，直到完成或出现异常循环退出。

**2、ET模式（edge-triggered边缘触发）**

1. ET是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个EWOULDBLOCK 错误）。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)
2. ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用**非阻塞**套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。
3. 内部实现方法：调用epoll_wait从rdlist取出事件后就不会再放回。
   ET模式下推荐使用非阻塞，我在这件事情上面纠结了很久，为什么阻塞就不行呢？网上给出的答案也都模棱两可，说阻塞模式下可能会
4. 造成永久阻塞，但是又不说明情况。思考后我的理解是这样的：
5. 非阻塞模式：在ET模式下，因为事件只会被通知一次，为了保证数据成功被读取或写入，在非阻塞模式下，采用循环的方式进行读写，直到完成或出现异常时退出。
   阻塞模式：如果不采用循环的方式进行读写，就会造成数据读/写不完的情况，因为下一次再调用epoll_wait就不会再通知了，如果尝试采用循环的方式进行读写，则会造成永久阻塞。
   造成阻塞的原因只有没有数据可读/可写，在非阻塞模式下出现没有数据可读/可写可以返回相应的错误信息，但是阻塞模式就会进入阻塞状态，而处理的该fd永远也不可能再有可读数据了，所以就被永久阻塞了。

**详解**

（https://blog.csdn.net/xdx2ct1314/article/details/7825951）

ET模型的逻辑：内核的读buffer有内核态主动变化时，内核会通知你， 无需再去mod。写事件是给用户使用的，最开始add之后，内核都不会通知你了，你可以强制写数据（直到EAGAIN或者实际字节数小于 需要写的字节数），当然你可以主动mod OUT，此时如果句柄可以写了（send buffer有空间），内核就通知你。
这里内核态主动的意思是：内核从网络接收了数据放入了读buffer（会通知用户IN事件，即用户可以recv数据）
并且这种通知只会通知一次，如果这次处理（recv）没有到刚才说的两种情况（EAGIN或者实际字节数小于 需要读写的字节数），则该事件会被丢弃，直到下次buffer发生变化。
与LT的差别就在这里体现，LT在这种情况下，事件不会丢弃，而是只要读buffer里面有数据可以让用户读，则不断的通知你。

**3、区别**：

1. LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件；可以是阻塞也可以是非阻塞运行；效率低于ET。
2. ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。必须是非阻塞；效率高。

## 四）比较

**句柄选择：**Select需要遍历所有句柄，时间复杂度是O(N)；Poll同Select；Epoll无需遍历，时间复杂度是O(1)；

**最大连接数：**Select的最大连接数由**FD_SETSIZE**宏定义，一般是1024但是可以修改，对应的效率也会下降；Poll本质上和Select没有区别，但是它没有最大连接限制的区别，原因是Poll是使用链表存放FD；Epoll没有最大连接限制，能打开的FD远大于1024，（1G的内存上能监听约10万个端口。

**消息传递：**Select和Poll都需要将消息传递到用户空间，需要内核拷贝动作；Epoll通过内核和用户空间共享一块内存来实现。

**总结**

1. 表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。
2. select低效是因为每次它都需要轮询。但低效也是相对的，视情况而定，也可通过良好的设计改善 