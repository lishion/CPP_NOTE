[TOC]

# winsock基础

## socket

广义上的**socket** 是一组对TCP(UDP)/IP进行封装的接口，是对TCP或UDP通信协议的封装。而**socket** 本意是指包含通信端ip地址和端口号的组合，用于作为通信时身份的标识。 

# winsock开发基础 

winsock是以一个在windosw下用于进行socket通信的集合。提供了许多的api供开发人员使用。下面通过对一些函数和宏定义以及结构体的介绍。

## WSAStartup

函数的原型为

```c++
int WSAStartup ( WORD wVersionRequested, LPWSADATA lpWSAData )；
```

用于对winsock进行初始化。其中参数的含义为:

* wVersionRequested: 表示请求的版本号，需要传入
* lpWSAData：存放winsock相关信息的结构体，初始化成功后返回数据使用
* 如果成功，该函数将返回0

其中需要传入的参数wVersionRequested可以通过**MAKEWORD(h,l)** 产生，h表示版本号的高字节，l表示版本号的低字节。一个调用的例子如下:

```c++
WSAData wsa;
if (WSAStartup(MAKEWORD(2,2),&wsa) != 0)//初始化版本号为2.2的winsock,如果不成功直接返回
{
	return ; 
}
```

## socket

函数原型为

```c++
socket(
__in int af,
__in int type,
__in int protocol
);
```
其中__in 表示该参数为输入，且函数运行的过程中不会改变，是一个空的宏定义，不会产生编译，仅作为提示。在windows很多库中都有这种表示方法，例如MFC中。此外还有 inout等，都是相同的效果。参数含义如下:

* af：address family的缩写，表示地址协议簇，一般使用的是利用宏定义的**AF_INET ** ，值为2
* type： 表示socket的类型，同意，一般使用**SOCK_STREAM** 表示 TCP协议，**SOCK_DGRAM** 表示UDP协议
* protocol：表示采用的协议，其实第二个参数制定后就可以不用再指定这个参数。一般设为0，表示自动选择类型

一个调用的例子为

```c++
SOCKET tcp_socket = socket(AF_INET,SOCK_STREAM,0)
```

## sockaddr_in

**sockaddr_in** 是一个表示地址的结构体，其中包含了协议簇，地址，端口号等信息。例如:

```c++
sockaddr_in local_addr;
local_addr.sin_family = AF_INET; //指定协议簇
local_addr.sin_addr.s_addr =inet_addr("127.0.0.1");//指定协议簇
local_addr.sin_port = htons(9000);//制定端口号
```

## bind

作用是将socket类型和地址信息绑定在一个，一般在作为服务器的时候使用。原型为:

```c++
bind(
    __in SOCKET s,
    __in_bcount(namelen) const struct sockaddr FAR * name,
    __in int namelen
    );
```

其中in在前面已经提到过，是一个空的宏定义。s表示socket信息的结构体，name表示一个sockaddr类型的结构体。namelen表示结构体的长度。 sockaddr 和 sockaddr_in 都是用来表示地址信息的结构体，两者存放的信息一样。只是实现的细节不一样。在使用的时候可以通过类型的转换来适配。一个典型的调用如下:

```c++
bind( tcp_socket,(sockaddr*)&local_addr,sizeof(local_addr);
```

同样，调用成功会返回0。可以通过返回值来判断是否调用成功。

## listen

作用是将一个socket设置到等待连接的状态。如果有请求的连接会放在一个队列中。有两个参数，第一个参数是绑定了地址信息的socket结构体，另一个参数就是队列存放的请求的最大数量。

## accept

一般放在**listen** 之后使用。当请求队列中存在请求时，会从队列的顶部拿出一个请求处理。如果队列中一个请求都没有，会直到出现一个请求之后才放回。也就是说，这个函数是**阻塞** 的。调用的方法如下:

```c++
sockaddr_in client_addr;
int len = sizeof(client_addr);
Socket socket_new = accept(tcp_socket,(sockaddr*)&client_addr,&len);
```

当取出一个请求后，回返回请求的socket信息，并且会通过将地址信息和地址的长度通过参数返回。

## select

在上文中提到**accept**是一个典型的阻塞函数。在等待请求的时候不会返回。windows提供了**select** 函数来达到非阻塞的效果。先看一下这个函数的定义:

```c++
select(
    __in int nfds,
    __inout_opt fd_set FAR * readfds,
    __inout_opt fd_set FAR * writefds,
    __inout_opt fd_set FAR * exceptfds,
    __in_opt const struct timeval FAR * timeout
    );
```

第一个参数可以忽略，不起作用。第2-4个参数都是一个类型fd_set的指针。第四个参数表示等待超时的时间，也是一个结构体的指针。fd_set 表示的是一系列socket的集合。select的作用就是对集合中符合条件的socket进行检查，函数运行后，符合条件的socket将被保留，不符合的将被删除。第一参数表示对fd_set 进行**可读**检查，第二个参数表示对集合进行**可写**检查，第三个进行**异常**检查。 对于可读的定义，在msdn中定义如下:

> The parameter *readfds* identifies the sockets that are to be checked for readability. If the socket is currently in the [**listen**](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms739168(v=vs.85).aspx) state, it will be marked as readable if an incoming connection request has been received such that an [**accept**](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms737526(v=vs.85).aspx) is guaranteed to complete without blocking. 

可以看出，如果一个处于**listen** 状态**socket** 将会在一个请求到来的时候被检查为可读。同时，通过可读检查后**accept** 函数可以不用阻塞，马上返回。于是就达到了将**accept** 变为不阻塞的效果。其外，还有其他的定义请参照：

> [msdn:select function](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms740141(v=vs.85).aspx)

利用**select**的这个特性，可以完成许多高效的操作。

# 利用winsock搭建一个简易的服务器

下面将给出用winsock搭建一个检验服务器的代码，作用是一直等待一个连接，连接到来后输出连接地址，然后推出。

```c++
	WSAData wsa;
	if (WSAStartup(MAKEWORD(2,2),&wsa) != 0)
	{
		return 0;   //失败
	}

	SOCKET tcp_socket = socket(AF_INET,SOCK_STREAM,0);//产生socket信息

	if(tcp_socket == INVALID_SOCKET){
		 return 0;//失败
	}

	sockaddr_in local_addr;

	local_addr.sin_family = AF_INET;
	local_addr.sin_addr.s_addr =inet_addr("127.0.0.1");//本机地址
	local_addr.sin_port = htons(9000);//端口9000 

	long flag = 1;
	ioctlsocket( tcp_socket, FIONBIO,( u_long*)&flag);//设为非阻塞

	//绑定地址到socket
	if( bind( tcp_socket,(sockaddr*)&local_addr,sizeof(local_addr) ) == SOCKET_ERROR ){
		return 0;
	}
	// 启动到listen状态
	if( listen(tcp_socket,10) == SOCKET_ERROR){
		return 0;
	}

	timeval tv;
	tv.tv_sec = 2;
	tv.tv_usec = 0;//设置select等待时间

	fd_set socket_set;
	FD_ZERO(&socket_set);//初始化一个fd_set
	FD_SET( tcp_socket , &socket_set);//将处于listen状态的socket放入fd_set中

	while(true){
		select(0,&socket_set,NULL,NULL,&tv);//读检查，2s后返回
		if(FD_ISSET(tcp_socket,&socket_set)){ //如果socket_set还在fd_set中 说明检查通过 有请求到来
          	sockaddr_in client_addr;
			int len = sizeof(client_addr);
			accept(tcp_socket,(sockaddr*)&client_addr,&len);//取出请求 
			cout<<inet_ntoa(client_addr.sin_addr)<<":"<<ntohs(client_addr.sin_port)<<endl;//打印地址
			closesocket(tcp_socket);
			WSACleanup();//释放资源
			return 0;
		}
		else{//如果没有请求，则一直打印等待链接。
            //这里也可以放其他代码 在等待请求时做其他事情
			cout<<"等待连接..."<<endl;
			FD_SET( tcp_socket , &socket_set);
		}
		
	}
 
```

运行效果如下:

![运行效果](http://www.z4a.net/images/2017/03/16/socket1.png)