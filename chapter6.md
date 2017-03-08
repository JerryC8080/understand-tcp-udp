# 一切皆 Socket
我们已经知道网络中的进程是通过 socket 来通信的，那什么是 socket 呢？
socket 起源于 UNIX，而 UNIX/Linux 基本哲学之一就是「一切皆文件」，都可以用「open → write/read → close」模式来操作。
socket 其实就是该模式的一个实现，socket 即是一种特殊的文件，一些 socket 函数就是对其进行的操作。

使用 TCP/IP 协议的应用程序通常采用系统提供的编程接口：**UNIX BSD 的套接字接口（Socket Interfaces）**
以此来实现网络进程之间的通信。
就目前而言，几乎所有的应用程序都是采用 socket，所以说现在的网络时代，网络中进程通信是无处不在，**一切皆 socket**

# 套接字接口 Socket Interfaces
套接字接口是一组函数，由操作系统提供，用以创建网络应用。
大多数现代操作系统都实现了套接字接口，包括所有 Unix 变种，Windows 和 Macintosh 系统。

> **套接字接口的起源**
套接字接口是加州大学伯克利分校的研究人员在 20 世纪 80 年代早起提出的。
伯克利的研究者使得套接字接口适用于任何底层的协议，第一个实现就是针对 TCP/IP 协议，他们把它包括在 Unix 4.2 BSD 的内核里，并且分发给许多学校和实验室。
这在因特网的历史成为了一个重大事件。
—— 《深入理解计算机系统》

从 Linux 内核的角度来看，一个套接字就是通信的一个端点。
从 Linux 程序的角度来看，套接字是一个有相应描述符的文件。
普通文件的打开操作返回一个文件描述字，而 socket() 用于创建一个 socket 描述符，唯一标识一个 socket。
这个 socket 描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些操作。

常用的函数有：
- socket()
- bind()
- listen()
- connect()
- accept()
- write()
- read()
- close()

# Socket 的交互流程
![socket 交互过程.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/46872611EB6C0874FE9E4C290E8F3FE9.png)

图中展示了 TCP 协议的 socket 交互流程，描述如下：
1. 服务器根据地址类型、socket 类型、以及协议来创建 socket。
2. 服务器为 socket 绑定 IP 地址和端口号。
3. 服务器 socket 监听端口号请求，随时准备接收客户端发来的连接，这时候服务器的 socket 并没有全部打开。
4. 客户端创建 socket。
5. 客户端打开 socket，根据服务器 IP 地址和端口号试图连接服务器 socket。
6. 服务器 socket 接收到客户端 socket 请求，被动打开，开始接收客户端请求，知道客户端返回连接信息。这时候 socket 进入阻塞状态，阻塞是由于 accept() 方法会一直等到客户端返回连接信息后才返回，然后开始连接下一个客户端的连接请求。
7. 客户端连接成功，向服务器发送连接状态信息。
8. 服务器 accept() 方法返回，连接成功。
9. 服务器和客户端通过网络 I/O 函数进行数据的传输。
10. 客户端关闭 socket。
11. 服务器关闭 socket。

这个过程中，服务器和客户端建立连接的部分，就体现了 TCP 三次握手的原理。

下面详细讲一下 socket 的各函数。

## Socket 接口
socket 是系统提供的接口，而操作系统大多数都是用 C/C++ 开发的，自然函数库也是 C/C++ 代码。

## socket 函数
该函数会返回一个套接字描述符（socket descriptor），但是该描述符仅是部分打开的，还不能用于读写。
如何完成打开套接字的工作，取决于我们是客户端还是服务器。

### 函数原型
```C++
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

### 参数说明
**domain**: 
协议域，决定了 socket 的地质类型，在通信中必须采用对应的地址。
常用的协议族有：`AF_INET`（ipv4地址与端口号的组合）、`AF_INET6`（ipv6地址与端口号的组合）、`AF_LOCAL`（绝对路径名作为地址）。
该值的常量定义在 `sys/socket.h` 文件中。

**type**:
指定 socket 类型。
常用的类型有：`SOCK_STREAM`、`SOCK_DGRAM`、`SOCK_RAW`、`SOCK_PACKET`、`SOCK_SEQPACKET`等。
其中 `SOCK_STREAM` 表示提供面向连接的稳定数据传输，即 TCP 协议。
该值的常量定义在 `sys/socket.h` 文件中。

**protocol**:
指定协议。
常用的协议有：`IPPROTO_TCP`（TCP协议）、`IPPTOTO_UDP`（UDP协议）、`IPPROTO_SCTP`（STCP协议）。
当值位 0 时，会自动选择 `type` 类型对应的默认协议。

## bind 函数
由服务端调用，把一个地址族中的特定地址和 socket 联系起来。

### 函数原型
```c++
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

### 参数说明
**sockfd**:
即 socket 描述字，由 socket() 函数创建。

***addr**：
一个 `const struct sockaddr` 指针，指向要绑定给 `sockfd` 的协议地址。
这个地址结构根据地址创建 socket 时的地址协议族不同而不同，例如 ipv4 对应 `sockaddr_in`，ipv6 对应 `sockaddr_in6`.
这几个结构体在使用的时候，都可以强制转换成 `sockaddr`。
下面是这几个结构体对应的所在的头文件：
1. `sockaddr`： `sys/socket.h`
2. `sockaddr_in`： `netinet/in.h`
3. `sockaddr_in6`： `netinet6/in.h`

> _in 后缀意义：互联网络(internet)的缩写，而不是输入(input)的缩写。

## listen 函数
服务器调用，将 socket 从一个主动套接字转化为一个监听套接字（listening socket）, 该套接字可以接收来自客户端的连接请求。
在默认情况下，操作系统内核会认为 socket 函数创建的描述符对应于主动套接字（active socket）。

### 函数原型
```C++
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

### 参数说明
**sockfd**:
即 socket 描述字，由 socket() 函数创建。

**backlog**:
指定在请求队列中的最大请求数，进入的连接请求将在队列中等待 accept() 它们。

## connect 函数
由客户端调用，与目的服务器的套接字建立一个连接。

### 函数原型
```C++
#include <sys/socket.h>
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen);
```

### 参数说明
**clientfd**:
目的服务器的 socket 描述符

***addr**:
一个 `const struct sockaddr` 指针，包含了目的服务器 IP 和端口。

**addrlen**：
协议地址的长度，如果是 ipv4 的 TCP 连接，一般为 `sizeof(sockaddr_in)`;

## accept 函数
服务器调用，等待来自客户端的连接请求。
当客户端连接，accept 函数会在 `addr` 中会填充上客户端的套接字地址，并且返回一个已连接描述符（connected descriptor），这个描述符可以用来利用 Unix I/O 函数与客户端通信。

### 函数原型
```C++
#indclude <sys/socket.h>
int accept(int listenfd, struct sockaddr *addr, int *addrlen);
```

### 参数说明
**listenfd**:
服务器的 socket 描述字，由 socket() 函数创建。

***addr**:
一个 `const struct sockaddr` 指针，用来存放提出连接请求客户端的主机的信息

***addrlen**:
协议地址的长度，如果是 ipv4 的 TCP 连接，一般为 `sizeof(sockaddr_in)`。

## close 函数
在数据传输完成之后，手动关闭连接。

### 函数原型
```C++
#include <sys/socket.h>
#include <unistd.h>
int close(int fd);
```

### 参数说明
**fd**:
需要关闭的连接 socket 描述符

## 网络 I/O 函数
当客户端和服务器建立连接后，可以使用网络 I/O 进行读写操作。
网络 I/O 操作有下面几组：
1. read()/write()
2. recv()/send()
3. readv()/writev()
4. recvmsg()/sendmsg()
5. recvfrom()/sendto()

最常用的是 read()/write()
他们的原型是：

```C++
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

鉴于该文是侧重于描述 socket 的工作原理，就不再详细描述这些函数了。

# 实现一个简单 TCP 交互
## 服务端

```C++
// socket_server.cpp

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

#define MAXLINE 4096 // 4 * 1024

int main(int argc, char **argv)
{
    int listenfd, // 监听端口的 socket 描述符
        connfd;   // 连接端 socket 描述符
    struct sockaddr_in servaddr;
    char buff[MAXLINE];
    int n;

    // 创建 socket，并且进行错误处理
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1)
    {
        printf("create socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    // 初始化 sockaddr_in 数据结构
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(6666);

    // 绑定 socket 和 端口
    if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) == -1)
    {
        printf("bind socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    // 监听连接
    if (listen(listenfd, 10) == -1)
    {
        printf("listen socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    printf("====== Waiting for client's request======\n");

    // 持续接收客户端的连接请求
    while (true)
    {
        if ((connfd = accept(listenfd, (struct sockaddr *)NULL, NULL) == -1))
        {
            printf("accept socket error: %s(errno: %d)\n", strerror(errno), errno);
            continue;
        }

        n = recv(connfd, buff, MAXLINE, 0);
        buff[n] = '\0';
        printf("recv msg from client: %s\n", buff);
        close(connfd);
    }

    close(listenfd);
    return 0;
}
```

## 客户端
```C++
// socket_client.cpp

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

#define MAXLINE 4096

int main(int argc, char **argv)
{
    int sockfd, n;
    char recvline[4096], sendline[4096];
    struct sockaddr_in servaddr;

    if (argc != 2)
    {
        printf("usage: ./client <ipaddress>\n");
        return 0;
    }

    // 创建 socket 描述符
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        printf("create socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    // 初始化目标服务器数据结构
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(6666);
    // 从参数中读取 IP 地址
    if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0)
    {
        printf("inet_pton error for %s\n", argv[1]);
        return 0;
    }

    // 连接目标服务器，并和 sockfd 联系起来。
    if (connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0)
    {
        printf("connect error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    printf("send msg to server: \n");

    // 从标准输入流中读取信息
    fgets(sendline, 4096, stdin);

    // 通过 sockfd，向目标服务器发送信息
    if (send(sockfd, sendline, strlen(sendline), 0) < 0)
    {
        printf("send msg error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }

    // 数据传输完毕，关闭 socket 连接
    close(sockfd);
    return 0;
}
```

# Run
首先创建 `makefile` 文件
```makefile
all:server client
server:socket_server.o
	g++ -g -o socket_server socket_server.o
client:socket_client.o
	g++ -g -o socket_client socket_client.o
socket_server.o:socket_server.cpp
	g++ -g -c socket_server.cpp
socket_client.o:socket_client.cpp
	g++ -g -c socket_client.cpp
clean:all
	rm all
```

然后使用命令:
```
$ make
```

会生成两个可执行文件：
1. `socket_server`
2. `socket_client`

分别打开两个终端，运行：
1. `./socket_server`
2. `./socket_client 127.0.0.1`

然后在 `socket_client` 中键入发送内容，可以再 `socket_server` 接收到同样的信息。

# 参考
[《后台开发 核心技术与应用实践》](https://book.douban.com/subject/26850616/)
[《计算机网络》](https://book.douban.com/subject/2970300/)