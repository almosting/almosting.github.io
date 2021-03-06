---
title: shutdown 和 close
date: 2018-03-15 14:41:21
categories: Linux
tags: Linux
---
## 现象

转载自：http://www.cnblogs.com/wainiwann/p/3942203.html

在开发的一个基于 rtmp 聊天的程序时发现了一个很奇怪的现象。

在 windows 下当我们执行 closesocket 的操作之后，阻塞的 recv 会立即返回 -1 。

而在 linux 下当我们执行 close 操作之后阻塞的 recv 会出现不能立即返回的现象。后来在网上一搜发现很多遇到类似这种现象的情况，大致意思应该是

当 socket 被动被 close 的时候进入了“CLOSE_WAIT（被动关闭一方）”的情况。
解决方法就是在你 close 之前调用一下：

`shutdown(socket, SHUT_RDWR)`

就是关闭 socket 的读写功能。

## Shutdown 和 Close

> 下面是对譬如“CLOSE_WAIT”现象的一些解释

主动关闭方和被动方经历的状态：

FIN_WAIT_1（主动关闭一方）：当 SOCKET 在 ESTABLISHED 状态时，它想主动关闭连接，向对方发送了 FIN 报文，此时该 SOCKET 即进入到 FIN_WAIT_1 状态。而当对方回应 ACK 报文后，则进入到 FIN_WAIT_2 状态，

FIN_WAIT_2（主动关闭一方）：上面已经详细解释了这种状态，实际上 FIN_WAIT_2 状态下的 SOCKET，表示半连接，也即有一方要求 close 连接，但另外还告诉对方，我暂时还有点数据需要传送给你，稍后再关闭连接。

TIME_WAIT（主动关闭一方）：表示收到了对方的 FIN 报文，并发送出了 ACK 报文就等 2MSL（2 倍最大生存时间）后即可回到 CLOSED 可用状态了。

CLOSE_WAIT（被动关闭一方）：这种状态的含义其实是表示在等待关闭。当对方 close 一个 SOCKET 后发送 FIN 报文给自己，你系统毫无疑问地会回应一个 ACK 报文给对方，此时则进入到 CLOSE_WAIT 状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话，那么你也就可以 close 这个 SOCKET，发送 FIN 报文给对方，也即关闭连接。所以你在 CLOSE_WAIT 状态下，需要完成的事情是等待你去关闭连接。

LAST_ACK（被动关闭一方）：它是被动关闭一方在发送 FIN 报文后，最后等待对方的 ACK 报文。当收到 ACK 报文后，也即可以进入到 CLOSED 可用状态

>下面是 socket 的 close 和 shutdown 的一些说明

Linux socket 关闭连接的方法有两种分别是 shutdown 和 close，首先看一下 shutdown 的定义

```c
#include <sys/socket.h>
int shutdown(int sockfd,int how);
```

how 的方式有三种分别是

- SHUT_RD（0）：关闭 sockfd 上的读功能，此选项将不允许 sockfd 进行读操作。

- SHUT_WR（1）：关闭 sockfd 的写功能，此选项将不允许 sockfd 进行写操作。

- SHUT_RDWR（2）：关闭 sockfd 的读写功能。

成功则返回 0，错误返回 -1，错误码 errno：EBADF 表示 sockfd 不是一个有效描述符；ENOTCONN 表示 sockfd 未连接；ENOTSOCK 表示 sockfd 是一个文件描述符而不是 socket 描述符。

close 的定义如下：

```c
  # include <unistd.h>
  int close(int fd);
```

关闭读写。

  成功则返回 0，错误返回 -1，错误码 errno：EBADF 表示 fd 不是一个有效描述符；EINTR 表示 close 函数被信号中断；EIO 表示一个 IO 错误。
  下面摘用网上的一段话来说明二者的区别：
  close--关闭本进程的 socket id，但链接还是开着的，用这 socket id 的其它进程还能用这个链接，能读或写这个 socket id
  shutdown--则破坏了 socket 链接，读的时候可能侦探到 EOF 结束符，写的时候可能会收到一个 SIGPIPE 信号，这个信号可能直到 socket buffer 被填充了才收到，shutdown 还有一个关闭方式的参数，0 不能再读，1 不能再写，2 读写都不能。

  > socket 多进程中的 shutdown, close 使用

  当所有的数据操作结束以后，你可以调用 close() 函数来释放该 socket，从而停止在该 socket 上的任何数据操作：close(sockfd);

  你也可以调用 shutdown() 函数来关闭该 socket。该函数允许你只停止在某个方向上的数据传输，而一个方向上的数据传输继续进行。如你可以关闭某 socket 的写操作而允许继续在该 socket 上接受数据，直至读入所有数据。

  int shutdown(int sockfd,int how);

  sockfd 是需要关闭的 socket 的描述符。参数 how 允许为 shutdown 操作选择以下几种方式：

- SHUT_RD：关闭连接的读端。也就是该套接字不再接受数据，任何当前在套接字接受缓冲区的数据将被丢弃。进程将不能对该套接字发出任何读操作。对 TCP 套接字该调用之后接受到的任何数据将被确认然后无声的丢弃掉。
- SHUT_WR：关闭连接的写端，进程不能在对此套接字发出写操作。
- SHUT_RDWR：相当于调用 shutdown 两次；首先是以 SHUT_RD，然后以 SHUT_WR
  使用 close 中止一个连接，但它只是减少描述符的参考数，并不直接关闭连接，只有当描述符的参考数为 0 时才关闭连接。
  shutdown 可直接关闭描述符，不考虑描述符的参考数，可选择中止一个方向的连接。

  **注意：**

   如果有多个进程共享一个套接字，close 每被调用一次，计数减 1，直到计数为 0 时，也就是所用进程都调用了 close，套接字将被释放。
   在多进程中如果一个进程中 `shutdown(sfd, SHUT_RDWR)` 后其它的进程将无法进行通信。如果一个进程 `close(sfd)` 将不会影响到其它进程，得自己理解引用计数的用法了，有 Kernel 编程知识的更好理解了。

> 更多关于 close 和 shutdown 的说明

1. 只要 TCP 栈的读缓冲里还有未读取（read）数据，则调用 close 时会直接向对端发送 RST。
2. shutdown 与 socket 描述符没有关系，即使调用 `shutdown(fd, SHUT_RDWR)` 也不会关闭 fd，最终还需 `close(fd)`。
3. 可以认为 `shutdown(fd, SHUT_RD)` 是空操作，因为 shutdown 后还可以继续从该 socket 读取数据，这点也许还需要进一步证实。
4. 在已发送 FIN 包后 write 该 socket 描述符会引发 EPIPE/SIGPIPE。
5. 当有多个 socket 描述符指向同一 socket 对象时，调用 close 时首先会递减该对象的引用计数，计数为 0 时才会发送 FIN 包结束 TCP 连接。shutdown 不同，只要以 SHUT_WR/SHUT_RDWR 方式调用即发送 FIN 包。
6. SO_LINGER 与 close，当 SO_LINGER 选项开启但超时值为 0 时，调用 close 直接发送 RST（这样可以避免进入 TIME_WAIT 状态，但破坏了 TCP 协议的正常工作方式），SO_LINGER 对 shutdown 无影响。
7. TCP 连接上出现 RST 与随后可能的 TIME_WAIT 状态没有直接关系，主动发 FIN 包方必然会进入 TIME_WAIT 状态，除非不发送 FIN 而直接以发送 RST 结束连接。

> 下面是对于该主题相关网络讨论资料总结

http://bbs.csdn.net/topics/350147238
http://bbs.csdn.net/topics/350050711
http://blog.csdn.net/helpxs/article/details/6661951
