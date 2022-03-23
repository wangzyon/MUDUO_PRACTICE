# 1 socket基本概念

1. socket组成

socket是传输层上为了简便开发封装一套接口API，socket在linux内核是一个文件描述符，并指向两个缓存区，一个读缓存，一个写缓存，因此是全双工，可同时读写；

> 管道只能同一个方向读写，半双工；socket可以同时读写，全双工，因此，有两个缓冲区
>

2. socket含义

- IP标识网络环境中唯一主机
- port标识主机唯一进程
- socket=ip+port标识网络中唯一的进程

3. socket使用

- 成对使用；
- 必须绑定地址；可以通过bind方法绑定固定地址，不绑定时将随机分配有效地址；

4. 网络字节序和主机字节序

网络数据流采用大端字节序（高位放在低地址，低位放在高地址），主机数据流采用小端字节序（高位放在高地址，低位放在低地址）

因此，主机数据在网络中需要进行字节序转换；

> 小端，低-低，高-高，低地址存低位，例如，ox12345678，低地址存78,56,34,12依次向高地址递增

# 2 socket编程

1. 创建一个socket

```cpp
#include <sys/types.h>        
#include <sys/socket.h>

// int socket(int domain, int type, int protocol); 
int lfd = socket(AF_INET, SOCK_STREAM, 0);
/*
参数：
domain（英译:域，指通信的域）：地址类型定义IPV4还是IPV6
type: 数据流模式（TCP）还是数据报模式（UDP）
protocol：协议，TCP采用哪个协议

返回：
创建成功返回文件描述符fd，创建失败，没有可用socket返回-1

上述表示创建一个IPV4的数据流模式，采用数据流模式的一种，即TCP
*/
```

2. socket地址

![](images\image-20211130065415578.png)

```cpp
/*
socket地址由结构体sockaddr_in定义，sockaddr_in中in表示internet,包含三个数据成员
sockaddr_in addr;

addr.sin_family: 地址类型IPV4 or IPV6，(2 bytes)
addr.sin_port: 端口号(2 bytes)
addr.sin_addr: IP(4 bytes)

端口号大小端转换：htons(host to net short in)，ntohs(net to host short int)
IP大小端转换：inet_pton，inet_ntop,主机IP到网络IP转换也称为点分十进制到网络字节序转换
*/

sockaddr_in make_one_net4_addr(std::string ip = "", int port = -1)
{
    /*
	构建sockaddr需要大小端转换;

    主机PORT->网络PORT：
    htons(port)，其中h(host) to n(net) s(short,2 bytes)

    网络PORT->主机PORT：
    ntohs(port)

    主机IP转网络IP：
    inet_pton(AF_INET, ip.c_str(), &addr.sin_addr)

    addr.sin_addr为socket大端字节序变量，inet_pton将小端字节序ip.c_str()转换后大端字节序存入addr.sin_addr;
    addr.sin_addr此时为传入参数

    网络IP转主机IP：
    char buf[1024] = {0};
    socklen_t addr_len = sizeof(addr);
    inet_ntop(AF_INET, &addr.sin_addr, buf, addr_len);
    printf("ip: %s\n", buf);

    inet_pton和inet_ntop接口一步到位，不用先将主机IP转为unsigned int等；
    */
    sockaddr_in addr;
    if (ip == "" or port == -1)
    {
        return addr;
    }
    else
    {
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        if (inet_pton(AF_INET, ip.c_str(), &addr.sin_addr) < 0)
        {
            printf("inet_pton error: %d\n", errno);
            exit(EXIT_FAILURE);
        }
        return addr;
    }
}
```

3. bind

socket绑定地址，一般服务器需要绑定地址；

**重点：**sockaddr_in和sockaddr均表示socket地址的结构体

- sockaddr已废弃，定义socket地址必须采用sockaddr_in;
- 由于历史遗留原因，bind、accept只能接收sockaddr，因此需要进行强转；

```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
/*
sockfd: 文件描述符
sockaddr: socket地址
socklen_t: socket地址长度，用于判别IPV4和IPV6

绑定成功返回0，失败返回-1
*/
sockaddr_in server_addr = make_one_net4_addr(ip, port);
socklen_t server_addr_len = sizeof(server_addr);
if (bind(lfd, (struct sockaddr *)&server_addr, server_addr_len) < 0)
{
    printf("bind error: %d\n", errno);
}
```

4. listen

listen并不是监听，而是设置监听的限制（同时最大连接数量）

```cpp
/*
int listen(int sockfd, int backlog);
sockfd设置sockfd监听的最大连接数量backlog

设置成功返回0，失败返回-1
*/
if (listen(lfd, 128) < 0)
{
    printf("listen error: %d\n", errno);
}
```

5. accept

启动监听，进入阻塞状态，直至有客户端连入，返回客户端文件描述符；

```cpp
/*
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
accept将接收到客户端请求，在服务器端创建一个新socket与客户端socket建立连接，也称为cfd（此cfd非客户端cfd，即服务端cfd和客户端cfd组成一条连接），服务端cfd地址保存到预先定义的sock地址变量，地址大小保存到预先定义的socklen_t，因此需要传递指针；即传入参数；
*/

sockaddr_in client_addr = make_one_net4_addr(); //创建一个用于跟客户端通信的sock地址
socklen_t client_addr_len;
int cfd = accept(lfd, (struct sockaddr *)&client_addr, &client_addr_len); //将新创建的sock地址存到client_addr
if (cfd < 0)
{
	printf("accept error: %d\n", errno);
}
else
{
	printf("create new connection, client sockfd %d\n", cfd);
}
```

# 3 socket编程demo

server.cpp

```cpp
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
#include <unistd.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <string>
#include <errno.h>
#include <iostream>
#include <strings.h>

// socket编程示例

sockaddr_in make_one_net4_addr(std::string ip = "", int port = -1)
{
    sockaddr_in addr;
    bzero(&addr, sizeof(addr));
    if (ip == "" or port == -1)
    {
        return addr;
    }
    else
    {
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        if (inet_pton(AF_INET, ip.c_str(), &addr.sin_addr) < 0)
        {
            printf("inet_pton error: %d\n", errno);
            exit(EXIT_FAILURE);
        }
        return addr;
    }
}

int main()
{
    int port = 8807;
    std::string ip("127.0.0.1");

    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    sockaddr_in server_addr = make_one_net4_addr(ip, port);
    socklen_t server_addr_len = sizeof(server_addr);

    if (bind(lfd, (struct sockaddr *)&server_addr, server_addr_len) < 0)
    {
        printf("bind error: %d\n", errno);
    }
    if (listen(lfd, 128) < 0)
    {
        printf("listen error: %d\n", errno);
    }

    sockaddr_in client_addr = make_one_net4_addr();
    socklen_t client_addr_len;
    int cfd = accept(lfd, (struct sockaddr *)&client_addr, &client_addr_len);
    if (cfd < 0)
    {
        printf("accept error: %d\n", errno);
    }
    else
    {
        printf("create new connection, client sockfd %d, port %d\n", cfd, ntohs(client_addr.sin_port));
    }

    char buf[1024] = {0};
    while (true)
    {
        size_t n = read(cfd, buf, sizeof buf);
        if (n < 0)
        {
            printf("read error: %d\n", errno);
        }
        else if (n == 0)
        {
            break;
        }
        else
        {
            for (int i = 0; i < n; ++i)
            {
                buf[i] = toupper(buf[i]);
            }
        }
        write(cfd, &buf, n);
    }

    close(lfd);
    close(cfd);

    return 0;
}
```

client.cpp

```cpp
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
#include <unistd.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <string>
#include <errno.h>
#include <iostream>
#include <strings.h>
#include <string.h>

// socket编程示例

sockaddr_in make_one_net4_addr(std::string ip = "", int port = -1)
{
    sockaddr_in addr;
    bzero(&addr, sizeof(addr));
    if (ip == "" or port == -1)
    {
        return addr;
    }
    else
    {
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        if (inet_pton(AF_INET, ip.c_str(), &addr.sin_addr) < 0)
        {
            printf("inet_pton error: %d\n", errno);
            exit(EXIT_FAILURE);
        }
        return addr;
    }
}

int main()
{
    int port = 8807;
    std::string ip("127.0.0.1");

    int cfd = socket(AF_INET, SOCK_STREAM, 0);
    printf("clent cfd: %d\n", cfd);

    sockaddr_in server_addr = make_one_net4_addr(ip, port);
    socklen_t server_addr_len = sizeof(server_addr);

    if (connect(cfd, (struct sockaddr *)&server_addr, server_addr_len) < 0)
    {
        printf("connect error: %d\n", errno);
    }

    int n;
    char buf[1024] = {0};
    std::string exit = "EXIT";
    while (true)
    {
        fgets(buf, sizeof(buf), stdin);
        write(cfd, buf, strlen(buf));
        n = read(cfd, buf, sizeof(buf));
        
        buf[strlen(buf)-1]=0; //去除末尾的换行符
        printf("%s (num:%d)\n", buf, n);
        if (strcmp(buf, exit.c_str()) == 0)
        {
            break;
        }
    }
    close(cfd);
    return 0;
}
```

Makefile

```makefile
mysocket:
	g++ -std=c++11 -o server server.cpp
	g++ -std=c++11 -o client client.cpp

# make clean
clean:
	rm -f server client
```

可以直接使用nc命令作为client 

```bash
nc 127.0.0.1 8807
```
