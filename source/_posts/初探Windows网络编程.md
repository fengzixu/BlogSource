title: 初探Windows网络编程
date: 2014-11-26 21:20:10
categories: [Windows]
tags: [Windows socket]
description:
---

# 初探网络编程

> 也是为了写键盘记录的那个项目吧，准备把网络编程和多线程还有Hook趁着最近课少全都看了。没想到，哎，真是一拖再拖，各种事打断我。周二开始写，到今天才写了个网络编程的部分。看了看有关socket编程的知识吧，根绝TCP和UDP协议写了两个小例子。(PS:教我们计算机网络的老师烂的一逼，讲TCP三次握手的时候，我都没认真听，这块遇到了些，期末之前我还得自己把书啃一遍)。以及一个UDP的网络聊天程序，不过现在只能是实现1v1,并且是分客户端和服务端的，服务端在不接受到客户端的请求的时候，是不能主动给客户端发送信息的。这就很蛋疼啊，不能实现动态加入聊天者的地址信息，多对多的聊天形式估计也得用多线程才行。呼呼，这周继续写。



<br>
## 基于TCP协议的socket程序
<br>
```c
/*
名称：基于TCP协议的网络编程的服务器端程序
时间：2014、11、25
作者：fengzixu
简介:
总体来讲就是根据TCP的socket编程写一个服务器和客户端的程序然后运行在本地。
关于服务器端的程序，整个过程中需要用到两个套接字，一个是监听套接字，还有一个是连接套接字。
监听套接字从创建开始一直到服务器程序没有终止的之前，都是存在的，而且没有关闭过。因为他要时时刻刻监听客户端的要求。但是
连接套接字主要是用于和客户端建立连接，然后进行通信，这样一来，每次客户端的不同请求都会造成连接套接字关闭和打开，甚至是有不同的
客户端依次和服务器端进行通信。
在给socketaddr_in对象的成员赋值的时候，出了family成员不需要强制使用网络字节顺序，其他的都需要。
过程很简单，比较容易理解，主要是一些细节和API不熟悉，不过慢慢用了应该还好。
1.初始化套接字库
2.创建套接字
3.绑定端口
4.开始监听
5.接受请求，建立连接
6.处理请求，发送回应
*/




#include<stdio.h>
#include<WinSock2.h>

int main()
{
	WORD wVerionRequested = MAKEWORD(1,1);			//指定加载的套接字动态库的版本，保存相应的版本号
	WSADATA wsadata;								//加载的有关套接字的版本库的信息都会存在这个结构体里面
	int start = WSAStartup(wVerionRequested, &wsadata);             //加载套接字动态库
	if(start != 0)									//如果成功的话，那么返回值为0
		return 0;
	if(LOBYTE(wsadata.wVersion)!=1 || HIBYTE(wsadata.wVersion)!=1)       //wVersion数据成员中的高位部分表示了套接字库的副版本，低位部分表示了套接字库的主版本，检查是否是我们需要的版本号
	{
		WSACleanup();	//如果不是就要释放套接字库分配给此进程的资源
		return 0;
	}
	SOCKET server_socket = socket(AF_INET,SOCK_STREAM,0);    //创建一个套接字

	SOCKADDR_IN addrsrv;			//bind函数要绑定信息，需要用到一个结构体，里面存放本地地址的信息，他根据使用的TCP协议而变为SOCKADDR_IN类型
	addrsrv.sin_family = AF_INET;
	addrsrv.sin_port = htons(6000);				//指定要分配给这个套接字的端口
	addrsrv.sin_addr.S_un.S_addr = INADDR_ANY;		//指定本机的ip地址
	bind(server_socket, (SOCKADDR *)&addrsrv, sizeof(SOCKADDR));
	listen(server_socket,10);					//监听此端口
	SOCKADDR_IN buffptr;						//创建一个结构体，用于接收客户端发来请求的时候，携带的客户端的地址信息，IP和端口号等
	int bufflen = sizeof(SOCKADDR);
	while(1)
	{
		SOCKET accept_socket = accept(server_socket,(SOCKADDR*)&buffptr,&bufflen);		//开始不断的接受客户端请求的到来
		char sendbuff[1000] = {0};														//发送缓冲区
		char recivebuff[1000] = {0};													//接收缓冲区
		sprintf(sendbuff,"welcome %s to www.fengzixu.net",inet_ntoa(buffptr.sin_addr));					//把我需要的信息写入到发送缓冲区，利用inet_ntoa函数将buffptr指向的存储客户端地址信息结构体中的sin_addr转换为点分十进制的字符串
		send(accept_socket,sendbuff,sizeof(sendbuff)+1,0);					//之所以+1是因为发送一个字符数组过去之后一定要在最后留有一个\0字符的位置
		recv(accept_socket,recivebuff,sizeof(recivebuff),0);
		printf("%s\n",recivebuff);
		closesocket(accept_socket);					//本次处理请求结束，释放为此次请求所分配的资源，关闭连接套接字。
	}
	return 0;
}




/*
名称：基于TCP协议的网络编程的客户端程序
时间：2014、11、25
作者：fengzixu
简介：
在客户端程序中，只需要用一个套接字就可以完成通信功能。因为他不用绑定和监听端口，在向服务器发送请求的时候，就会携带端口和IP地址信息。
客户端在发送了连接请求之后，会首先收到一条服务器端发回的连接成功的信息，然后正式进入通讯阶段.
*/



#include<stdio.h>
#include<WinSock2.h>

int main()
{
	WORD wVersionRequested;
	WSADATA wsdata;

	wVersionRequested = MAKEWORD(1,1);
	int result = WSAStartup(wVersionRequested,&wsdata);
	if(result != 0)
	{
		return 0;
	}
	if(LOBYTE(wsdata.wVersion) != 1 || HIBYTE(wsdata.wVersion) != 1)
	{
		WSACleanup();
		return 0;
	}
	while(1){
	SOCKET connect_socket = socket(AF_INET,SOCK_STREAM,0);
	sockaddr_in sockad;
	sockad.sin_family = AF_INET;
	sockad.sin_port = htons(6000);
	sockad.sin_addr.S_un.S_addr = inet_addr("125.221.225.14");
	//connect(connect_socket,(sockaddr*)&sockad,sizeof(SOCKADDR));
	connect(connect_socket,(sockaddr*)&sockad,sizeof(SOCKADDR));
	char sendbuff[1000] = {0};
	char recivebuff[1000] = {0};
	recv(connect_socket,recivebuff,sizeof(recivebuff),0);
	printf("%s\n",recivebuff);
	printf("input your message: ");
	gets(sendbuff);
	send(connect_socket,sendbuff,sizeof(sendbuff)+1,0);
	closesocket(connect_socket);
	}
	WSACleanup();
	return 0;
}
```

<br>
## 基于UDP协议的Socket程序
<br>
```c
//客户端
#include<stdio.h>
#include<WinSock2.h>

int main()
{
	WORD wVersionRequested = MAKEWORD(1,1);
	WSAData wsdata;
	int flag = WSAStartup(wVersionRequested,&wsdata);
	if(flag != 0)
		return 0;
	if(HIBYTE(wsdata.wVersion) != 1 || LOBYTE(wsdata.wVersion) != 1)
	{
		WSACleanup();
		return 0;
	}

	SOCKET client_socket = socket(AF_INET,SOCK_DGRAM,0);
	char sendbuff[1000] = {0};
	sockaddr_in clientadd;
	clientadd.sin_family = AF_INET;
	clientadd.sin_port=htons(6000);
	clientadd.sin_addr.S_un.S_addr = inet_addr("125.221.225.14");
	gets(sendbuff);
	sendto(client_socket,sendbuff,sizeof(sendbuff),0,(SOCKADDR*)&clientadd,sizeof(sockaddr_in));
	closesocket(client_socket);
	WSACleanup();
	return 0;
}


//服务器端
/*
名称：基于UDP协议的服务端程序
作者： fengzixu
时间: 2014/11/25
简介：
基于UDP和TCP两个协议的不同版本，大致的框架都是相同的，只是有几处细节不同。
首先，基于TCP协议的服务端，需要两个套接字，一个用来绑定端口并且监听，一个用来建立连接。
但是UDP协议中不需要监听和建立连接，绑定端口之后直接就可以通信，且从始至终只用一个套接字(在创建套接字的时候，TCP和UDP的类型是不一样的)
其次，就是接受信息的API不同。TCP是recv而UDP是recvfrom
*/


#include<stdio.h>
#include<WinSock2.h>

int main()
{
	WORD wVersionRequested = MAKEWORD(1,1);
	WSAData wsdata;
	int flag = WSAStartup(wVersionRequested,&wsdata);
	if(flag != 0)
		return 0;
	if(HIBYTE(wsdata.wVersion) != 1 || LOBYTE(wsdata.wVersion) != 1)
	{
		WSACleanup();
		return 0;
	}

	SOCKET server_socket = socket(AF_INET,SOCK_DGRAM,0);
	sockaddr_in sockadd;
	sockadd.sin_family = AF_INET;
	sockadd.sin_port = htons(6000);
	sockadd.sin_addr.S_un.S_addr = htonl(INADDR_ANY);

	bind(server_socket, (SOCKADDR*)&sockadd,sizeof(SOCKADDR));

	char recivebuff[1000] = {0};
	sockaddr_in clientaddr;
	int len = sizeof(clientaddr);
	recvfrom(server_socket,recivebuff,sizeof(recivebuff),0,(SOCKADDR*)&clientaddr,&len);
	printf("%s\n",recivebuff);
	closesocket(server_socket);
	WSACleanup();
	return 0;
}
```
<br>
## 基于UDP协议的网络聊天程序

### [源码，戳我](https://github.com/fengzixu/SocketTestPrograme/tree/master/UDPSocketTalkPrograme)


<br>




## 所有程序的源码
### [Click it](https://github.com/fengzixu/SocketTestPrograme)