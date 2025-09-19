# IO多路复用

<!-- more -->

## 需求：高性能网络服务器

设计一个高性能的网络服务器，提供多个客户端同时连接，并处理客户端的处理请求。

## 1.第一印象

   当我们知道这个需求后，我们第一印象为了应对并发，可以基于多线程，写一个多线程的程序，但是多线程会有一些弊端就是需要cpu上下文切换，这样就会导致处理操作句柄，代价大。那多线程不够好的话，我们就把目光放在了单线程，用单线程处理大量客户端的连接，先抛出一个问题：加入有多个客户端连接，在处理A用户发过来的消息的同时，B用户也发来了小心，会不会导致B的消息丢失，答案是不会的，原因是处理IO时，处理IO操作时，接收B传过来消息的并不是CPU，而是DMA控制器，不会造成数据的丢失，为数据的不丢失性，提供了保障。

  那我们知道每一个网络连接再内核中都已一个文件描述符来表示，我们可以用单线程程序写一个网络服务器

```c++
while(1)
{
for(fdx in (fdA~FdB))
	{
if(Fdx 有数据)
		{
读Fdx,处理数据
		}
	}
}
```

那如果这样写一个网络服务器，他的性能也不够低，但是不够好，原因时判断有数据到来是程序在判断，效率不够好。那么我们看一下Select 是怎么做的？

## 2.Select

我们先看select的有关的函数：

①:nfds：最大的文件描述符+1，fd_set *readfds：读文件描述符集合，

```c
 int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```

②：从fd_set 移除一个文件描述符

```c
void FD_CLR(int fd, fd_set *set);

```

③：判断fd是否在fd_set集合中

```c
int  FD_ISSET(int fd, fd_set *set);

```

④：向fd_set 加入一个文件描述符，**当面向网络服务器设计中，这里加入的使tcp协议中三次握手中的accept返回的文件描述符**

```c
void FD_SET(int fd, fd_set *set);

```

⑤：初始化一个fd_set

```c
void FD_ZERO(fd_set *set);

```

**其中最核心的是fd_set**,他是一个bitmap(位图)，

1.调用selsct提前的准备工作：

2.首先要创建一个fd_set 类型的变量，（监听集合）

3.调用FD_ZERO,初始化这个监听集合，

4.按需求调用FD_SET增加监听，

5.调用select函数，使调用的进程陷入阻塞，操作系统轮询监听集合，

那么select函数底层做了什么操作呢？

select函数会将用户态空间的fd_set拷贝到内核态，由内核态来判断是否有数据到来，如果没有数据到来，那么select函数就会阻塞，当有数据到来的时候select函数会将fd_set中标识有数据到来的fd标记，select会返回，然后程序再遍历文件描述符，遍历出就绪的文件描述符并做出相应的数据处理。

那么，这样做的**优点**就是，判断文件描述符有数据到来变为了由内核来判断，提高了效率，也不会大量再内核态和用户态切换，

缺点：

①：fd_set 位图限制了数量，该数量需要重新编译内核

②：数据仍然有大量的内核态和用户态之间的拷贝

③：监听集合和就绪集合耦合

④：再海量监听，少量就绪的情况下，大部分时间会浪费再FD_ISSET()中，原因使并不知道就绪的是哪一个！

那么pool函数又做了哪些优化呢？简单说一下。

## 3.Poll

直接说结果，Poll函数中将select 中的bitmap 改为了结构体，那么就解决了位图数量限制的问题，

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
// fds 是一个链表的指针，链表的节点是一个pollfd 的结构体，nfds 是节点的个数，
struct pollfd {
               int   fd;         /* file descriptor */
               short events;     /* requested events */
               short revents;    /* returned events */
           };
//其中fd，依然是文件描述符，
//events 标识的是，这个文件描述符在意的事件，当是这个事件来临的时候，poll函数会将pollfd.revects置位
//来表征这个文件描述已经就绪
//pollfd.revents 可以用来每次处理完就绪的文件描述否后，再置为0；
//虽然并没有完全解决就序集合与遍历集合耦合的问题，但是poolfds 是可以重用的；select中的fd_set不可以重用！
```

这样的优化，解决了部分问题，但是仍然不够完美，那么epoll又是怎样优化的：

## 4.epoll

epoll 不支持跨平台，linux下独有的，不属于posxi规范，

epoll相对于select 和 poll来说就比较复杂一点了

我们先看相关的函数

```c
 int epoll_create(int size);
//创建一个epoll的文件对象，size值没有意义，只要是一个大于0的数值即可，

```

调用epoll_create 时，内核除了我们在epoll文件系统里建了个file结点，再内核cache（缓冲区）里建了一个红黑树 用于存储以后epoll_ctl 传来的socket外还会建立一个list链表，用于存储准备就绪的事件。当就绪以后，会将就绪集合拷贝到用户。

```c
int  epoll_ctl(int  epfd,  int  op,  int  fd,  struct  epoll_event *event);
// epfd是epoll_create 创建的epoll的文件对象，
//op的选项：
//EPOLL_CTL_ADD 向epfd中加入一个文件描述符  
//EPOLL_CTL_MOD 向epfd更改与目标文件关联的事件事件描述符fd
//EPOLL_CTL_DEL 向epfd中删除一个文件描述符
//event 是一个指向结构体 epoll_event 的指针，
//而 epoll_event 中的 events 描述的是事件的属性，读阻塞/写阻塞，data是携带的额外的信息，
//epoll_data_t 是一个联合体一般是fd。
typedef union epoll_data {
               void        *ptr;
               int          fd;//一般是这个
               uint32_t     u32;
               uint64_t     u64;
           } epoll_data_t;

struct epoll_event {
               uint32_t     events;      /* Epoll events */
               epoll_data_t data;        /* User data variable */
           };
```

所以我们可以得出结论，epoll_ctl()函数中的event 所指向的结构体相比较poll中的结构体是去掉了revent。

**在这里我们不仅将fd 经过op操作可以加入epfd中，同时我们还传入一个携带同样fd的event结构体，所以，传入两份相同的数据确实会对空间造成影响，但是结构体中的fd，可以使我们再epoll_wait 函数中能够遍历这个数组**

**用来遍历就绪集合，达到一个空间换时间的功能。他同时也处理了select 的就绪集合和监听集合耦合的问题**。

```c
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
//如果 timeout 的值为负数，epoll_wait 函数会一直阻塞，直到有事件发生。
//如果 timeout 的值为零，epoll_wait 函数会立即返回，无论是否有事件发生。这相当于在非阻塞模式下调用 epoll_wait。
//如果 timeout 的值为正数，epoll_wait 函数将等待指定的时间，直到有事件发生或者超时。如果在超时之前有事件发生，epoll_wait 函数将立即返回，并将事件存储到 events 数组中。如果超时时间到达而没有事件发生，epoll_wait 函数也会返回，此时返回值为 0，表示没有事件发生。
//struct epoll_event *events 是一个传入传出参数，events 是一个元素类型为struct epoll_event，长度为maxevents，他们将用来保存就绪集合，
// return value 是就序集合的长度，再event.data.fd中找到就绪文件描述符。
```

接下来我们看一个 实现epoll的一个示例代码：

```c
#define MAX_EVENTS 10
           struct epoll_event ev, events[MAX_EVENTS];
           int listen_sock, conn_sock, nfds, epollfd;

           /* Code to set up listening socket, 'listen_sock',
              (socket(), bind(), listen()) omitted */

           epollfd = epoll_create1(0);
           if (epollfd == -1) {
               perror("epoll_create1");
               exit(EXIT_FAILURE);
           }

           ev.events = EPOLLIN;
           ev.data.fd = listen_sock;
           if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
               perror("epoll_ctl: listen_sock");
               exit(EXIT_FAILURE);
           }

           for (;;) {
               nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
               if (nfds == -1) {
                   perror("epoll_wait");
                   exit(EXIT_FAILURE);
               }

               for (n = 0; n < nfds; ++n) {
                   if (events[n].data.fd == listen_sock) {
                       conn_sock = accept(listen_sock,
                                     (struct sockaddr *) &addr, &addrlen);
                       if (conn_sock == -1) {
                           perror("accept");
                           exit(EXIT_FAILURE);
                       }
                       setnonblocking(conn_sock);
                       ev.events = EPOLLIN | EPOLLET;
                       ev.data.fd = conn_sock;
                       if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                                   &ev) == -1) {
                           perror("epoll_ctl: conn_sock");
                           exit(EXIT_FAILURE);
                       }
                   } else {
                       do_use_fd(events[n].data.fd);
                   }
               }
           }

```

epoll 的边缘触发，水平触发，以及海量监听下性能也很好  scales well to 

redis 是用的epoll 。njinx javaNIO (linux 下)

问题：既然fd_set 运用了bitmap select之前是标识监听的文件描述符，select 会把就绪的集合置为，所以底层怎么技既能标识监听又能表示就绪的？