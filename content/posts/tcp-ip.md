+++
title = "TCP/IP网络编程"
date = 2020-03-28T21:26:30+08:00
images = []
tags = ["tcp/ip"]
categories = ["net"]
draft = false

+++

> 本文图片来源于geektime网络编程实战，版权归其所有

网络编程的大部分内容就是设计并实现应用层协议，应用层以下的都由套接字封装了。下面逐步讲解套接字。

## 服务器端建立连接

调用socket函数，返回fd（买了个电话机）

调用bind函数，分配地址信息（去电信局申请下来了电话号码）

调用listen函数，转化为🉑️接收连接状态（安装好电话线了）

调用accept函数，受理连接请求（接听电话）

总结：

第一步调用socket创建套接字。

第二步调用bind函数分配IP地址和端口号。

第三步调用listen函数转为可接收请求状态。

第四步调用accpet函数受理连接请求。

### 客户端建立连接

调用socket函数，返回fd（买了个电话机）

调用connect，拨通电话，等待服务器accept

#### 完整过程如下图

![tcp-ip](/img/tcp-ip.png)

### 套接字

首先：协议是为了完成数据交换而定好的约定。

`int socket(int domain, int type, int protocol)`

这是socket函数的声明。那么这里可以看到3个参数，第三个就是协议，这就是为什么开头要先确定协议是什么。

协议其实一般叫协议族。这个名字不是乱起的，族最字面的解释就是一大家子。比如TCP/IP名字就叫TCP/IP协议族，从头文件`sys/socket.h`中可以找到以下各种协议族。

| 名称     | 协议族               |
| -------- | -------------------- |
| PF_INET  | IPv4互联网协议族     |
| PF_INET6 | IPv6互联网协议族     |
| PF_LOCAL | 本地通信的UNIX协议族 |
| ...      | ...                  |

而第一个参数 `domain`，其实就是协议族，第一个参数协议族，第三个参数是具体协议族里的某个协议。第一个参数决定了第三个参数的范围。


第二个参数 `type`, 查看man手册可以看到`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`这三种传输方式，事实上，协议族定了并不能决定传输方式。

首先无视掉`SOCK_RAW`因为man下查看文档你会发现这句话 `The type SOCK_RAW, which is available only to the super-user.`。

`SOCK_STREAM`  这种类型的将创建面向连接的套接字。`SOCK_STREAM类型提供基于序列的，可靠的，双向连接的字节流` , 因为套接字内部存在buffer，简单来说就是字节数组，只要不超过buffer的大小，所以read和write的调用次数不限，所以不存在数据边界。

`SOCK_DGRAM` 这种类型将创建面向消息的套接字。`SOCK_DGRAM类型提供不可靠、不按序传递、以数据的高速传输为目的的套接字`强调快速而非传输顺序。传输的数据可能丢失也可能损毁。传输的数据有边界（发几次就得收几次，而不能攒着一次收了）。限制传输的数据的大小。

第三个参数`protocol`，如果一个族内一种数据传输方式的协议只有一种，严格意义上来说，前两个参数就足以确定某个协议了，但是有些族中存在一些一些有多种数据传输方式。

### 分配IP地址和端口

当获取到socket的文件描述符之后，需要分配IP地址和端口号。

#### IP地址

IP地址分为两类，IPv4， IPv6。二者区别主要是字节数，目前通用的IPv4，IPv6是为了解决IP地址耗尽提出的标准。

IPv4地址是有分类的，IP地址，逻辑上分为网络号、子网号、主机号。

IP地址的分类以网络号作为区分，网络号占1字节的为A类：0～127，B类占据2字节： 128～191，c类占据3字节192～223，D类4字节。

网络的实际构成，在NAT技术下，可以理解为公网IP是一个个路由器或者交换机，实际上是先向路由器发送数据，再由路由器NAT 映射进子网内，再在子网内通过子网IP地址去(arp)找到MAC地址寻址发送过去。MAC地址只能在子网内起作用（虽然说要求全球唯一）

#### 端口号

主机是能运行多个程序的，且大多数情况下共用同一个IP地址，为了区分哪些数据是哪些程序的。操作系统通过端口号，去分配相应的数据给对应端口的套接字（也就是相应的程序）。

端口号由16位构成，所以范围是0～65535。但是0～1023的知名端口是分配给特定应用程序的，所以一般应用分配的端口为1024～65535.

此外，TCP和UDP是不同的模块分配的，所以相同端口的UDP和TCP套接字是允许的。

IPv4地址的是以结构体的方式表示的。其结构如下:

```C
struct sockaddr_in
{
  sa_family_t    sin_family;   //地址族
  uint16_y       sin_port;     //16位端口， htons()网络字节序保存
  struct in_addr sin_addr;     //32位IP地址, htons()
  char           sin_zero[8];  //不使用,为确保与sockaddr结构保持一致的填充
};

struct in_addr
{
  in_addr_t s_addr; //32位IPv4， 网络字节序保存,htons()
};
```

除了填充地址结构体外，传输的数据等都无需要转换字节序，因为会自动做。
IPv4与IPv6及通用地址结构对比如下图:
![addr-struct](/img/addr-struct.png)



IP地址是点分十进制的，为了转换成32位的整数型数据。在`<arpa/inet.h>`里提供了一个函数`in_addr_t inet_addr(const char* string);` 该函数会获取一个点分十进制IP地址的字符串，返回32位整数型数据并返回。

INADDR_ANY常量可以作为初始化服务器IP地址时的IP使用，因为会自动获取服务器端的计算机IP地址。

地址结构体初始化好后，可以给套接字分配地址信息。将初始化好的地址信息通过bind函数分配。

`int bind(int socket, const struct sockaddr *address, sockelen_t address_len);`

sockfd经过bind这步完成后，物料已经准备好了，可以开始等待请求了。进入等待请求需要调用`int listen(int socket, int backlog)`  backlog是最多允许多少个连接请求进度队列。

到这里服务器的所有工作已经就绪，当有请求来的时候，其实就可以处理了。但是，由于之前那个套接字是为了接收连接请求而创建的。真正连接着客户并交换数据的是另一个套接字。可以调用` int accept(int socket, struct sockaddr *restrict address, socklen_t *restrict address_len)` 来创建并连接到发起请求的客户端。

accept函数受理连接请求（backlog, 未完成连接队列的大小）中待处理的客户端连接请求。

TCP连接的server端是在accept系统调用中完成从LISTEN到SYN-RCVD（收到SYN），再到ESTABLISHED（收到ACK）的变迁过程。

#### 优雅的关闭套接字

tcp的半关闭：

Linux的close函数意味着完全断开连接。这意味着不仅不能传输也不能接收数据。

这会造成数据的损毁。那么半关闭的方法就产生了。半关闭就是将全双工的socket变为可以接收数据但无法传输，或者可以传输但无法接收。只关闭流的一半。

socket是全双工的，本质上来说 有两个流，客户端的输出流对着服务器端的输入流，客户端的输入流对着服务器端的输出流。半关闭就是断掉其中一个。

`int shutdown(int socket, int how)` 两个参数分别是需要断开的套接字文件描述符和传递断开方式信息。

第二个参数有三个选项 ：

* SHUT_RD: 断开输入流
* SHUT_WR: 断开输出流
* SHUT_RDWR: 同时断开I/O流，相当于调用了两次hsutdown，第一次以SHUT_RD为参数，第二次以SHUT_WR为参数。

既要发送传输结束的信号，又要接收数据时，服务器端可以关闭输出流，通知客户端(客户端接收数据)。

#### 套接字的多种可选项

套接字分层 `IPPROTO_IP`层可选项是IP协议相关的， `IPPROTO_TCP`是TCP协议相关的，`SOL_SOCKET`是套接字相关的可选项。

`int getsocketopt(int socket, int level, int option_name, const void *option_value, socklen_t options_len`) 这个函数用户读取套接字的可选项。

` int setsockopt(int socket, int level, int option_name, const void *option_value, socklen_t option_len);` 这个函数用于更改可选项。

套接字类型只能在创建的时候决定，以后不能再改，也就是说tcp套接没法改成udp套接字。

#### Time-wait

Time-wait是指主动发起方，四次握手后其实并没有真正关闭，而是会有一小段的存活期，在此期间占据的端口是无法绑定的，因为还在使用中呢。

四次挥手的时候，假设是服务器端主动发起Fin，如果服务器端不存在Time-wait，当服务器主动发起Fin的时候，最后一次的ACK是第一个发FIN的发出去。如果没有Time-wait那么，如果这个ACK回给客户端的时候失败了，客户端会重发自己的Fin，然而这个Fin永远得不到ACK，于是会一直重传，如果服务器处于Time-wait，则有机会向客户端重传最后的ACK，客户端有机会正常终止。

客户端其实也有Time-wait，只是客户端的端口是随机分配的，所以比较少绑定端口错误。

如果需要快速重启，避过Time-wait，在套接字可选项中更改`SO_REUSEADDR`的状态，将其改为1（真）就表示可以分配Time-wait状态下的端口号。

#### Nagle算法

TCP默认使用Nagle算法，最大限度的进行缓冲，直到之前的收到了ACK，于是将缓冲区内的全部填入一个数据报内，减少网络包数目，降低负载。

当流量未受影响时，Nagle算法用了比不用慢，典型的是传输大文件数据时。

将`TCP_NODELAY` 改为1（真）就会禁用Nagle算法。

# 知识补给

ssize_t 表示 signed int类型，size_t表示unsigned in类型，这么做是为了可移植性。因为操作系统的位数在增加，到时候只需要改头文件里的声明就好了。

_t结尾表示操作系统定义的数据类型。

uint16_t表示16位无符号整数

sockaddr_in是IPv4的地址信息结构体，但是sockaddr结构体不是只为IPv4设计，所以需要特别设计填充。

大端字节序：

 假设内存地址是: 0x20, 0x21,0x22,0x23

那么0x12345678如果是高位在低地址

0x20 放0x12 //最高位放入地址的最低位

0x21 放0x34

0x22放 0x56

0x23放0x57

这种叫大端序也叫网络字节序。

系统有四个函数帮助转换字节序：

```C
unsigned short htons(unsigned short);  //为端口转换
unsigned short ntohs(unsigned short);   // 为端口转换
unsigned long htonl(unsigned long);  //IP地址
unsigned long ntohl(unsigned long);  //IP地址转换
// h表示主机字节序
// n表示网络字节序
```

服务器端的IP的目的在于，一台主机可以有多个NIC，添加了IP后就会决定接收该IP的数据。如果只有一个IP地址，直接使用INADDR_ANY。

### TCP套接字的I/O缓冲

write和read，write调用瞬间将数据移至输出缓冲，read函数掉用瞬间从输入缓冲读取数据。

* I/O缓冲在每个TCP套接字中单独存在。
* I/O缓冲在创建套接字时自动生成。
* 即使关闭套接字也会继续传递输出缓冲中遗留的数据。
* 关闭套接字将丢失输入缓冲中的数据。

  



