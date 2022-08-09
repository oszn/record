#  IO多路复用

说实话这个东西可能非必要，但是一直困扰我，我也很烦恼，所以最近决定要弄清楚80%。

首先第一个概念同步/异步,阻塞/非阻塞。

同步/异步，肯定是函数调用方式，肯定有一个最小的入口，例如我父函数觉得某个子函数执行太慢了，当然我可以选择一直等他，或者说，等他来提醒我，他做完了，也就是回调一个过程，一般来说这个过程有很多实现方法，但阻塞与非阻塞，是操作系统的一个操作的说，也就是我父函数子函数都可能会有的一个操作，好吧，这个问题先芳芳，好难啊。

[link1](https://aijishu.com/a/1060000000186917)

[link2](https://www.51cto.com/article/693213.html)

网络io的阶段：

1. 硬件接口到内核态
2. 内核态到用户态

![image.png](https://aijishu.com/img/bVWM3)

阻塞io与非阻塞io区别在于从硬件接口到内核态这个过程是否发生阻塞等待，阻塞io会等待，非阻塞不会。



## 伪代码

同步阻塞

```c++
while(true) {
  // accept阻塞
  client_fd = accept(listen_fd)
  // 开启线程read数据（fd增多导致线程数增多）
  new Thread func() {
    // recv阻塞（多线程不影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}

作者：一角钱技术
链接：https://juejin.cn/post/6882984260672847879
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

同步非阻塞

````python
// 伪代码描述
while(true) {
  // accept非阻塞（cpu一直忙轮询）
  client_fd = accept(listen_fd)
  if (client_fd != null) {
    // 有人连接
    fds.append(client_fd)
  } else {
    // 无人连接
  }  
  for (fd in fds) {
    // recv非阻塞
    setNonblocking(client_fd)
    // recv 为非阻塞命令
    if (len = recv(fd) && len > 0) {
      // 有读写数据
      // logic
    } else {
       无读写数据
    }
  }  
}
````

io多路复用

```python
// 伪代码描述
while(true) {
  // 通过内核获取有读写事件发生的fd，只要有一个则返回，无则阻塞
  // 整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，accept/recv是不会阻塞
  for (fd in select(fds)) {
    if (fd == listen_fd) {
        client_fd = accept(listen_fd)
        fds.append(client_fd)
    } elseif (len = recv(fd) && len != -1) { 
      // logic
    }
  }  
}

作者：一角钱技术
链接：https://juejin.cn/post/6882984260672847879
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```





## io多路复用

```c++
// readfds:关心读的fd集合；writefds：关心写的fd集合；excepttfds：异常的fd集合  
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout); 
```



```c++
int poll (struct pollfd *fds, unsigned int nfds, int timeout);  
  
struct pollfd {  
    int fd; /* file descriptor */  
    short events; /* requested events to watch */  
    short revents; /* returned events witnessed */  
};  
```

**select 和 poll 的区别**

1. select 能处理的最大连接，默认是 1024 个，可以通过修改配置来改变，但终究是有限个；而 poll 理论上可以支持无限个
2. select 和 poll 在管理海量的连接时，会频繁的从用户态拷贝到内核态，比较消耗资源。



**epoll**

```c++
//创建epollFd，底层是在内核态分配一段区域，底层数据结构红黑树+双向链表  
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大  
  
//往红黑树中增加、删除、更新管理的socket fd  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；  
  
//这个api是用来在第一阶段阻塞，等待就绪的fd。  
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);  
```

```c++
1. int epoll_create(int size);  
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。  
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。  
  
2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；  
函数是对指定描述符fd执行op操作。  
- epfd：是epoll_create()的返回值。  
- op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。  
- fd：是需要监听的fd（文件描述符）  
- epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：  
  
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
  
3. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);  
等待epfd上的io事件，最多返回maxevents个事件。  
参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。
```

epoll 对文件描述符的操作有两种模式：LT（level trigger）和 ET（edge trigger）。LT 模式是默认模式，LT 模式与 ET 模式的区别如下：　　 LT 模式：当 epoll\_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用 epoll\_wait 时，会再次响应应用程序并通知此事件。　　 ET 模式：当 epoll\_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用 epoll\_wait 时，不会再次响应应用程序并通知此事件

**ET模式**

ET(edge-triggered)是高速工作方式，只支持 no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过 epoll 告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个 EWOULDBLOCK 错误）。但是请注意，如果一直不对这个 fd 作 IO 操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)

ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

### 就绪过程

```c++
#include <sys/epoll.h>

// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
};

// API
int epoll_create(int size); // 内核中间加一个 ep 对象，把所有需要监听的 socket 都放到 ep 对象中
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // epoll_ctl 负责把 socket 增加、删除到内核红黑树
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);// epoll_wait 负责检测可读队列，没有可读 socket 则阻塞进程


作者：一角钱技术
链接：https://juejin.cn/post/6882984260672847879
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```



在epoll中，对于每一个事件，都会建立一个epitem结构体，如下所示：

```cpp
struct epitem{
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}

```

当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户。



### 图解过程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c06b4e5d90e49529f818807edeb3e8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)





**总结**

目前看了很多博客，io多路复用的知识比之前了解的更深了，首先从io角度出发。正常io的过程有2个过程，

1. 从设备到内核态。
2. 从内核态到用户态。

每一个过程都会进行系统调用，而阻塞与非阻塞都是与同步，二者区别在于第一个过程，非阻塞状态下会轮询第一个过程，一定会有一个返回值，然后可以同步的处理其他请求，而第二个过程则是第一个过程就绪后，从内核态将内容拷贝到用户态，从这里可以看出非阻塞可以处理更多的请求，但是会有更大的cpu开销，因为无论准备就绪与否都会去查询第一个过程。那1000个请求，你1k次系统调用，而且并不是每一次都有返回值，所以系统会很繁忙，而io多路复用，既然我需要访问很多次，那我可以直接一次性在设备上访问，这样可以减少系统调用次数，也就是第一个过程减少，用一个线程去处理这个问题。当然也衍生出一些小问题，我每次都需要将用户态需要访问的fdset输入到内核态，赋值开销也很大，而且很多时候，并不能返回很多，也就是每次也需要将同样的大小查询返回，造成很大的开销。所以有了epoll，对于epoll来说，生成后，直接将内容注册到内核态，这样会减少开销，然后通过红黑树储存文件描述符，双向链表储存已经就绪的eptime结构体,当被调用epoll wait就直接查看rdlist里面是否有内容即可。如果有，返回这个rdlist，然后处理，这样直接返回需要处理的内容即可少很多的处理。还有就是水平出发和边缘触发，总的来说水平你没读那个数据会一直提醒你，而边缘出发，只会提醒你一次。然后调用wait会直接阻塞第一个过程，知道有数据到来，或者超时了。



目前这个还可以在继续深度挖挖，我感觉可以再看看其他的具体内容。