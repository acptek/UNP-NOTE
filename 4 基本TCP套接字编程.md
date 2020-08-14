## socket

```c
int socket(int domain, int type, int protocol);
// domain: 指明协议族 / 协议域
	/*
		AF_INET
		AF_INET6
		AF_LOCAL
		AF_ROUTE
		AF_KEY
	*/
// type: 套接字类型
	/*
		SOCK_STREAM
		SOCK_DGRAM
		SOCK_SEQPACKET // 有序分组套接字
		SOCK_RAW // 原始套接字
	*/
// protocol : 协议类型常值

/// return : 成功：返回小整数：sfd：sockfd套接字描述符
```

### 对比AF_ 和 PF_

协议族：协议族就是不同协议的集合，在Linux中，用宏来表示不同的协议族，这个宏的形式是PF开头，比如IPv4协议族为PF_INET,PF的意思是PROTOCOL FAMILY，这些宏定义在/usr/include/bits/socket.h中

地址族：地址族就是一个协议族所使用的地址集合，也是用宏来表示不同的地址族，这个宏的形式是AF开头，比如IP地址族为AF_INET,AF的意思是ADDRESS FAMILY，这些宏定义在/usr/include/bits/socket.h中

AF_ : 表示地址族

PF_ : 表示协议族

在<sys/socket.h>中 PF _ xx = AF _ xx



## connect

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// sockfd
// addr : 指向一个套接字地址结构的指针
// addrlen
```

客户在调用connect前不必非得调用bind函数：内核会确定源IP地址，并选择一个临时端口作为源端口



connect返回出错的情况：

- TCP客户端没有收到SYN响应：ETIMEOUT  - 等待一段时间再发送，超过一定时间返回出错

- 对客户的响应是RST：立即返回 ECONNREFUSED

  RST产生的条件：1 服务端口上没有监听连接  2 TCP取消一个已有连接的一个方式  3 TCP收到一个不存在的连接上的段

- ICMP目标不可达：EHOSTUNREACH 或 ENETUNREACH：同第一种情况，等待、重发、返回



## bind

```c
// 将本地协议地址赋予一个套接字
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// sockfd
// addr : 一个指向特定于协议的地址结构指针 （本地地址）
```

如果指定端口号为0，内核就在bind被调用时选择一个临时端口

对于IPv4而言，通配地址由常值 **INADDR_ANY** 来指定，其值一般为0：告知内核取选择IP地址

对于IPv6的地址赋值：server.sin6_addr = in6addr_any : 系统预先分配in6addr_any变量并将其初始化为常值 **IN6ADDR_ANY_INIT** 



获取内核选择的一个临时端口号，调用函数 getsockname() 来返回协议地址，从中查看

（书上的：进程捆绑非通配符IP地址到套接字上的例子：将所有对应的IP地址定义成单个网络接口的别名，启动HTTP服务器副本）

（或者：服务器调用getsockname函数获取客户的目的IP地址，依此处理相应请求）

（捆绑非通配IP的优点：把一个给定的目的IP地址解复用到一个给定的服务器进程是由内核完成的）



## listen

```c
int listen(int sockfd, int backlog);
```

1 将一个未连接的套接字转换成一个被动套接字，指示内核应接受指向该套接字的连接请求

2 backlog参数规定了内核应该为该套接字排队的最大连接的个数

- 内核为任何一个给定的套接字维护两个队列：

  1 **未完成连接队列**

  相应的SYN段对应：已经由某个客户端发出并且到达服务器，此时服务器正在等待完成相应的TCP三路握手过程（SYN_RCVD状态）

  2 **已完成连接队列**

  每个已完成TCP三路握手的客户对应其中的一项 （ESTABLISHED状态）

--》 这两个队列中数量的和不能超过 backlog  （也就是说在同一个sfd上监听的连接不能超过backlog个）

--》或者说最大是 backlog * 1.5

如果三路握手正常完成，在未完成队列中建立的条目会移到已完成连接队列的队尾。当进程调用accept时，已完成的链接队列中的队头项将返回给进程，如果队列为空则进程进入睡眠，直到TCP在该队列放入一项时才唤醒

--> backlog不要定义为0



关于backlog的设定：可以使用包裹函数，先用环境变量来指定backlog的值，然后再调用listen



关于SYN洪泛：使用IP欺骗向服务程序高速发送大量IP连接请求来装填未完成队列，使得合法的SYN排不上队

backlog ---》 应该指定某个给定套接字上内核为之排队的最大已完成连接数

限制已完成连接的数的目的：在监听某个给定套接字的应用程序停止接收连接时，防止内核在该套接字上继续接收新的请求。



## accept

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// addr : 返回已连接的对端进程的协议地址


// 返回值：如果连接成功，返回值是由内核自动生成的一个全新描述符（已连接套接字）
```

从已完成连接队列队头返回下一个已完成连接。当已完成队列为空，则进程被投入睡眠。<u>（**会将队首的已完成连接移出队列？**）~~（可以以另外一个进程或线程继续来处理这个连接，同时关闭本进程的该连接（移出队列？）~~）</u>：是的，当accept成功后会将队头返回给进程，并且从队列中移出



## fork  exec

（书上p91：exec的6个函数之间的关系图：图4-12）





## close





## getsockname  getpeername

```c
// 返回与某个套接字关联的本地协议地址
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// 返回与，某个套接字关联的外地协议地址
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- 在一个没有调用bind的TCP客户上，getsockname用于返回由内核赋予该连接的本地IP和端口
- 端口号为0调用bind，同上
- 通配IP调用的bind，同上
- getsockname可用于获取某个套接字的地址族
- 当一个服务器是由调用过accept的某个进程通过exec执行程序时，它能够获取客户身份的唯一途径就是调用getpeername。 （**这里的举例需要在看一下**）