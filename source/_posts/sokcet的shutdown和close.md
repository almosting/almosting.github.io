---
title: sokcet的shutdown和close
date: 2021-04-29 14:41:21
categories: Linux
---


## 现象

转载自:http://www.cnblogs.com/wainiwann/p/3942203.html

在开发的一个基于rtmp聊天的程序时发现了一个很奇怪的现象。

在windows下当我们执行 closesocket的操作之后，阻塞的 recv会立即返回 -1 。

而在linux下当我们执行close操作之后阻塞的recv会出现不能立即返回的现象。后来在网上一搜发现很多遇到类似这种现象的情况，大致意思应该是

当socket被动被close的时候进入了 “CLOSE_WAIT（被动关闭一方）”的情况。
解决方法就是在你close之前调用一下 ：

shutdown(socket, SHUT_RDWR);

就是关闭socket的读写功能。

------------------------------------------------------
## Shutdown和Close

**下面是对譬如 “CLOSE_WAIT”现象的一些解释：**

主动关闭方和被动方经历的状态：

FIN_WAIT_1（主动关闭一方）：当SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN报文，此时该SOCKET即进入到FIN_WAIT_1状态。而当对方回应ACK报文后，则进入到FIN_WAIT_2状态，

FIN_WAIT_2（主动关闭一方）：上面已经详细解释了这种状态，实际上FIN_WAIT_2 状态下的SOCKET，表示半连接，也即有一方要求close连接，但另外还告诉对方，我暂时还有点数据需要传送给你，稍后再关闭连接。

TIME_WAIT（主动关闭一方）：表示收到了对方的FIN报文，并发送出了ACK报文就等2MSL（2倍最大生存时间）后即可回到CLOSED可用状态了。

CLOSE_WAIT（被动关闭一方）：这种状态的含义其实是表示在等待关闭。当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话，那么你也就可以close这个SOCKET，发送FIN报文给对方，也即关闭连接。所以你在CLOSE_WAIT状态下，需要完成的事情是等待你去关闭连接。

LAST_ACK（被动关闭一方）：它是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也即可以进入到CLOSED可用状态

-------------------------------------------------------------------
**下面是 socket的close和shutdown的一些说明：**

Linux socket关闭连接的方法有两种分别是shutdown和close，首先看一下shutdown的定义
`#include <sys/socket.h>`
`int shutdown(int sockfd,int how);`

how的方式有三种分别是

- SHUT_RD（0）：关闭sockfd上的读功能，此选项将不允许sockfd进行读操作。

- SHUT_WR（1）：关闭sockfd的写功能，此选项将不允许sockfd进行写操作。

- SHUT_RDWR（2）：关闭sockfd的读写功能。

成功则返回0，错误返回-1，错误码errno：EBADF表示sockfd不是一个有效描述符；ENOTCONN表示sockfd未连接；ENOTSOCK表示sockfd是一个文件描述符而不是socket描述符。

close的定义如下：
  `# include <unistd.h>`
  `int close(int fd);`
关闭读写。

  成功则返回0，错误返回-1，错误码errno：EBADF表示fd不是一个有效描述符；EINTR表示close函数被信号中断；EIO表示一个IO错误。
  下面摘用网上的一段话来说明二者的区别：
  close-----关闭本进程的socket id，但链接还是开着的，用这个socket id的其它进程还能用这个链接，能读或写这个socket id
  shutdown--则破坏了socket 链接，读的时候可能侦探到EOF结束符，写的时候可能会收到一个SIGPIPE信号，这个信号可能直到socket buffer被填充了才收到，shutdown还有一个关闭方式的参数，0 不能再读，1不能再写，2 读写都不能。
  **socket 多进程中的shutdown, close使用**
  当所有的数据操作结束以后，你可以调用close()函数来释放该socket，从而停止在该socket上的任何数据操作：
  close(sockfd);
  你也可以调用shutdown()函数来关闭该socket。该函数允许你只停止在某个方向上的数据传输，而一个方向上的数据传输继续进行。如你可以关闭某socket的写操作而允许继续在该socket上接受数据，直至读入所有数据。
  int shutdown(int sockfd,int how);
  sockfd是需要关闭的socket的描述符。参数 how允许为shutdown操作选择以下几种方式：

- SHUT_RD：关闭连接的读端。也就是该套接字不再接受数据，任何当前在套接字接受缓冲区的数据将被丢弃。进程将不能对该套接字发出任何读操作。对TCP套接字该调用之后接受到的任何数据将被确认然后无声的丢弃掉。
- SHUT_WR:关闭连接的写端，进程不能在对此套接字发出写操作
- SHUT_RDWR:相当于调用shutdown两次：首先是以SHUT_RD,然后以SHUT_WR
  使用close中止一个连接，但它只是减少描述符的参考数，并不直接关闭连接，只有当描述符的参考数为0时才关闭连接。
  shutdown可直接关闭描述符，不考虑描述符的参考数，可选择中止一个方向的连接。
  注意:
   如果有多个进程共享一个套接字，close每被调用一次，计数减1，直到计数为0时，也就是所用进程都调用了close，套接字将被释放。
   在多进程中如果一个进程中shutdown(sfd, SHUT_RDWR)后其它的进程将无法进行通信. 如果一个进程close(sfd)将不会影响到其它进程. 得自己理解引用计数的用法了. 有Kernel编程知识的更好理解了.

**更多关于close和shutdown的说明:**

1. 只要TCP栈的读缓冲里还有未读取（read）数据，则调用close时会直接向对端发送RST。
2. shutdown与socket描述符没有关系，即使调用shutdown(fd, SHUT_RDWR)也不会关闭fd，最终还需close(fd)。
3. 可以认为shutdown(fd, SHUT_RD)是空操作，因为shutdown后还可以继续从该socket读取数据，这点也许还需要进一步证实。
4. 在已发送FIN包后write该socket描述符会引发EPIPE/SIGPIPE。
5. 当有多个socket描述符指向同一socket对象时，调用close时首先会递减该对象的引用计数，计数为0时才会发送FIN包结束TCP连接。shutdown不同，只要以 SHUT_WR/SHUT_RDWR方式调用即发送FIN包。
6. SO_LINGER与close，当SO_LINGER选项开启但超时值为0时，调用close直接发送RST（这样可以避免进入TIME_WAIT状态，但破坏了TCP协议的正常工作方式），SO_LINGER对shutdown无影响。
7. TCP连接上出现RST与随后可能的TIME_WAIT状态没有直接关系，主动发FIN包方必然会进入TIME_WAIT状态，除非不发送FIN而直接以发送RST结束连接。

-----------------------------------------------------------------------------
**下面是对于该主题相关网络讨论资料总结：**
http://bbs.csdn.net/topics/350147238
http://bbs.csdn.net/topics/350050711
http://blog.csdn.net/helpxs/article/details/6661951
