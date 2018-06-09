## epoll应用--大并发处理

```c++
//头文件 pub.h
#ifndef _vsucess

#define _vsucess

#ifdef __cplusplus
extern "C"
{

#endif
//服务器创建socket
int server_socket(int port);

//设置非阻塞
int setnonblock(int st);

//接收客户端socket
int server_accept(int st);

//关闭socket
int close_socket(int st);

//接收消息
int socket_recv(int st);

//连接服务器
int connect_server(char *ipaddr,int port);

//发送消息
int socket_send(int st);

//将sockaddr_in转化成IP地址
int sockaddr_toa(const struct sockaddr_in * addr, char * ipaddr);

#ifdef __cplusplus
}
#endif

#endif
```

```c++
//辅助方法--pub.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>//htons()函数头文件
#include <netinet/in.h>//inet_addr()头文件
#include <fcntl.h>
#include "pub.h"

#define MAXBUF 1024

//创建socket
int socket_create()
{
    int st = socket(AF_INET, SOCK_STREAM, 0);
    if (st == -1)
    {
        printf("create socket failed ! error message :%s\n", strerror(errno));
        return -1;
    }
    return st;
}

//设置服务端socket地址重用
int socket_reuseaddr(int st)
{
    int on = 1;
    if (setsockopt(st, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) == -1)
    {
        printf("setsockopt reuseaddr failed ! error message :%s\n",
                strerror(errno));
        //close socket
        close_socket(st);
        return -1;
    }
    return 0;
}

//服务器绑定--监听端口号
int socket_bind(int st, int port)
{
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    //type
    addr.sin_family = AF_INET;
    //port
    addr.sin_port = htons(port);
    //ip
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    //bind ip address
    if (bind(st, (struct sockaddr *) &addr, sizeof(addr)) == -1)
    {
        printf("bind failed ! error message :%s\n", strerror(errno));
        //close socket
        close_socket(st);
        return -1;
    }
    //listen
    if (listen(st, 20) == -1)
    {
        printf("listen failed ! error message :%s\n", strerror(errno));
        //close socket
        close_socket(st);
        return -1;
    }
    return 0;
}

//服务器创建socket
int server_socket(int port)
{
    if (port < 0)
    {
        printf("function server_socket param not correct !\n");
        return -1;
    }
    //create socket
    int st = socket_create();
    if (st < 0)
    {
        return -1;
    }
    //reuseaddr
    if (socket_reuseaddr(st) < 0)
    {
        return -1;
    }
    //bind and listen
    if (socket_bind(st, port) < 0)
    {
        return -1;
    }
    return st;
}

//连接服务器
int connect_server(char *ipaddr,int port)
{
    if(port<0||ipaddr==NULL)
    {
        printf("function connect_server param not correct !\n");
        return -1;
    }
    int st=socket_create();
    if(st<0)
    {
        return -1;
    }
    //conect server
    struct sockaddr_in addr;
    memset(&addr,0,sizeof(addr));
    addr.sin_family=AF_INET;
    addr.sin_port=htons(port);
    addr.sin_addr.s_addr=inet_addr(ipaddr);
    if(connect(st,(struct sockaddr *)&addr,sizeof(addr))==-1)
    {
        printf("connect failed ! error message :%s\n",strerror(errno));
        return -1;
    }
    return st;
}

//设置非阻塞
int setnonblock(int st)
{
    if (st < 0)
    {
        printf("function setnonblock param not correct !\n");
        //close socket
        close_socket(st);
        return -1;
    }
    int opts = fcntl(st, F_GETFL);
    if (opts < 0)
    {
        printf("func fcntl failed ! error message :%s\n", strerror(errno));
        return -1;
    }
    opts = opts | O_NONBLOCK;
    if (fcntl(st, F_SETFL, opts) < 0)
    {
        printf("func fcntl failed ! error message :%s\n", strerror(errno));
        return -1;
    }
    return opts;
}

//接收客户端socket
int server_accept(int st)
{
    if (st < 0)
    {
        printf("function accept_clientsocket param not correct !\n");
        return -1;
    }
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    socklen_t len = sizeof(addr);
    int client_st = accept(st, (struct sockaddr *) &addr, &len);
    if (client_st < 0)
    {
        printf("accept client failed ! error message :%s\n", strerror(errno));
        return -1;
    } else
    {
        char ipaddr[20] = { 0 };
        sockaddr_toa(&addr, ipaddr);
        printf("accept by %s\n", ipaddr);
    }
    return client_st;
}

//关闭socket
int close_socket(int st)
{
    if (st < 0)
    {
        printf("function close_socket param not correct !\n");
        return -1;
    }
    close(st);
    return 0;
}

//将sockaddr_in转化成IP地址
int sockaddr_toa(const struct sockaddr_in * addr, char * ipaddr)
{
    if (addr == NULL || ipaddr == NULL)
    {
        return -1;
    }
    unsigned char *p = (unsigned char *) &(addr->sin_addr.s_addr);
    sprintf(ipaddr, "%u.%u.%u.%u", p[0], p[1], p[2], p[3]);
    return 0;
}

//接收消息
int socket_recv(int st)
{
    if (st < 0)
    {
        printf("function socket_recv param not correct !\n");
        return -1;
    }
    char buf[MAXBUF] = { 0 };
    int rc=0;
    rc=recv(st,buf,sizeof(buf),0);
    if(rc==0)
    {
        printf("client is close ! \n");
        return -1;
    }else if(rc<0)
    {
        /*
         * recv错误信息：Connection reset by peer
         * 错误原因：服务端给客户端发送数据，但是客户端没有接收，直接关闭，那么就会报错
         * 如果客户端接受了数据，再关闭，也不会报错，rc==0.
         */
        printf("recv failed ! error message :%s \n",strerror(errno));
        return -1;
    }
    printf("%s",buf);
    //send message
    /*
    memset(buf,0,sizeof(buf));
    strcpy(buf,"i am server , i have recved !\n");
    if(send(st,buf,strlen(buf),0)<0)
    {
        printf("send failed ! error message :%s \n",strerror(errno));
        return -1;
    }
    */
    return 0;
}

//发送消息
int socket_send(int st)
{
    char buf[MAXBUF]={0};
    while(1)
    {
        //read from keyboard
        read(STDIN_FILENO,buf,sizeof(buf));
        if(buf[0]=='0')
        {
            break;
        }
        if(send(st,buf,strlen(buf),0)<0)
        {
            printf("send failed ! error message :%s \n",strerror(errno));
            return -1;
        }
        memset(buf,0,sizeof(buf));
    }
    return 0;
}
```



```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>//htons()函数头文件
#include <netinet/in.h>//inet_addr()头文件
#include <fcntl.h>
#include <sys/epoll.h>
#include "pub.h"

#define MAXSOCKET 20

int main(int arg, char *args[])
{
	if(arg < 2)
	{
		printf("please print one param!\n");
		return -1;		
	}
	//create server socket
	int listen_st = server_socket(atoi(args[1]));
	if (listen_st < 0)
	{
		return -1;
	}
	/*
	 *声明epoll_event结构体变量ev，变量ev用于注册事件，
	 *数组events用于回传需要处理的事件
	 */
	struct epoll_event ev, events[100];
	//生成用于处理accept的epoll专用文件描述符 
	int epfd = epoll_create(MAXSOCKET);
	//把socket设置成非阻塞方式
	setnonblock(listen_st);
	//设置需要放到epoll池里的文件描述符
	ev.data.fd = listen_st;
	//设置这个文件描述符需要epoll监控的事件
	/*
	 *EPOLLIN代表文件描述符读事件
	 *accept，rev都是读事件
	 */
	ev.events = EPOLLIN | EPOLLERR | EPRLLHUP;
	/*
	 *注册epoll事件
	 *函数epoll_ctl中&ev参数表示需要epoll监视的listen_st这个socket中的一些时间
	 */
	 epoll_ctl(epfd,EPOLL_CTL_ADD, listen_st, &ev);
	 while (1)
	 {
	 	/*
		 *等待epoll池中的socket发生事件，这里一般设置为阻塞的
		 *events这个参数的类型是epoll_event类型的数组
		 *如果epoll池中的一个或者多个socket发生事件，
		 *epoll_wait就会返回，参数events中存放了发生事件的socket和这个socket所发生的事件
		 *这里强调一点，epoll池存放的是一个个socket，不是一个个socket事件
		 *如果epoll_wait返回事件数组后，下面的程序代码却没有处理当前socket发生的事件
		 *那么epoll_wait将不会再次阻塞，而是直接返回，参数events里面的就是刚才那个socket没有被处理的时间
		 */
		 int nfds = epoll_wait(epfd, events,MAXSOCKET, -1);
		 if (nfds == -1) 
		 {
		 	printf("epoll_wait failed ! error message :%s \n", strerror(errno));
			 break;	
		 }
		 int i = 0;
		 for (; i < nfds; i++) 
		 {
		 	if (events[i].data.fd < 0)
			 	continue;
			if (events[i].data.fd == listen_st)
			{
				//接收客户端socket
				int client_st = server_accept(listen_st);
				/*
				 *监测到一个用户的socket连接到服务器listen_st绑定的端口
				 *
				 */
				if (client_st < 0)
				{
					continue;	
				} 
				//设置客户端socket非阻塞
				setnonblock(client_st);
				//将客户端socket加入到epoll池中
				struct epoll_event client_ev;
				client_ev.data.fd = client_st;
				client_ev.data.fd = client_st;
				client_ev.events = EPOLLIN | EPOLLERR | EPOLLHUP;
				epoll_ctl(epfd, EPOLL_CTL_ADD, client_st, &client_ev);
				/*
				 *注释：当epoll池中listen_st这个服务器socket有消息的时候
				 *只可能是来自客户端的连接消息
				 *recv，send使用的都是客户端的socket，不会向listen_st发送消息的
				 */
				 continue; 
			}	
			//客户端有事件到达
			if (events[i].events & EPOLLIN)
			{
				//表示服务器这边的client_st接收到消息
				if (socket_recv(events[i].data.fd) < 0)
				{
					close_socket(events[i].data.fd);
					//接收数据出错或者客户端已经关闭
					events[i].data.fd = -1;
					/*这里continue是因为客户端socket已经被关闭了，
					 *但是这个socket可能还有其他的事件，会继续执行其他时间，
					 *但是这个socket已经被设置成为-1
					 *所以豁免的close_socket()函数都会报错
					 */
					continue; 	
				}	
				/*
				 *此处不能continue，因为每个socket都可能有多个事件同时发送到服务端
				 *这也是下面语句用if而不是if-else的原因
				 */ 
			} 
			//客户端有事件到达
			if(events[i].events & EPOLLERR)
			{
				printf("EPOLLERR\n");
				/*
				 *返回出错事件，关闭socket，清理epoll池
				 *当关闭socket并且events[i].data.fd=-1, epoll会自动将该socket从池中清除
				 */
				 close_socket(events[i].data.fd);
				 events[i].data.fd = -1;
				 continue; 
			 } 
			 //客户端有事件到达
			 if (events[i].events & EPOLLHUP)
			 {
			 	printf("EPOLLHUP\n");
			 	//返回挂起事件，关闭socket，清理epoll
				close_socket(events[i].data.fd);
				events[i].data.fd = -1;
				continue; 
			 } 
		 }	 	
	 } 
	 //close epoll
	 close(epfd);
	 //close server socket
	 close_socket(listen_st);
	 return 0; 
}
```

```c++
//网络编程客户端
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>//htons()函数头文件
#include <netinet/in.h>//inet_addr()头文件
#include <fcntl.h>
#include <sys/epoll.h>
#include "pub.h"

int main(int arg,char *args[])
{
    if(arg<2)
    {
        printf("please print two param !\n");
    }
    //端口号
    int port=atoi(args[2]);
    //服务端IP地址
    char ipaddr[30]={0};
    strcpy(ipaddr,args[1]);
    //connect server
    int st=connect_server(ipaddr,port);
    //send message
    //发送消息--
    socket_send(st);
    //close socket
    close(st);
    return 0;
}
```

