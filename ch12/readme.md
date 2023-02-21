**完整版文章请参考：**
[TCP/IP网络编程完整版文章](https://blog.csdn.net/u011895157/article/details/128678250?spm=1001.2014.3001.5502)

[TOC]

## 第 12 章 I/O 复用

### 12.1 基于 I/O 复用的服务器端

#### 12.1.1 多进程服务端的缺点和解决方法

为了构建并发服务器，只要有客户端连接请求就会创建新进程。但是创建进程需要大量的运算和内存空间，由于每个进程都具有独立的内存空间，所以相互间的数据交换也要采用相对复杂的方法，I/O 复用技术可以解决这个问题。

#### 12.1.2 理解复用

在电子及通信工程领域，**复用**可以表示为：

> 在 1 个通信频道中传递多个数据（信号）的技术

**复用**的含义：

> 为了提高物理设备的效率，只用最少的物理要素传递最多数据时使用的技术

上述两种方法的内容完全一致。可以用纸电话模型做一个类比：

![在这里插入图片描述](https://img-blog.csdnimg.cn/df66ddc857014606a60c98054cf1fed8.png)

同时也可以简化为如下图的改进：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ea2a2d9561894943a9cda08e29638a84.png)

如图做出改进，就是引入了复用技术。

复用技术的优点：

- 减少连线长度
- 减少纸杯个数

即使减少了连线和纸杯的量仍然可以进行三人同时说话，但是如果碰到以下情况：

> 「好像不能同时说话？」

实际上，因为是在进行对话，所以很少发生同时说话的情况。也就是说，上述系统采用的是**「时分复用」**技术。因为说话人声频率不同，即使在同时说话也能进行一定程度上的区分（杂音也随之增多）。因此，也可以说是「频分复用技术」。

#### 12.1.3 复用技术在服务器端的应用

服务器端引入复用技术可以减少所需进程数。下图是多进程服务端的模型：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a7f7de0d7af348ce8dbf2312cd8ad1fd.png)

下图是引入复用技术之后的模型：

![在这里插入图片描述](https://img-blog.csdnimg.cn/af966ae1bf4d459aa84efea1946fcf53.png)

从图上可以看出，引入复用技术之后，可以减少进程数。重要的是，无论连接多少客户端，**提供服务的进程只有一个**。

### 12.2 理解 select 函数并实现服务端

#### 12.2.1 select 函数的功能和调用顺序

使用 select 函数时可以将多个文件描述符集中到一起统一监视，项目如下：

- 是否存在套接字接收数据？
- 无需阻塞传输数据的套接字有哪些？
- 哪些套接字发生了异常？

> 术语：「事件」。当发生监视项对应情况时，称「发生了事件」。

select 函数的使用方法与一般函数的区别并不大，更准确的说，他很难使用。但是为了实现 I/O 复用服务器端，我们应该掌握 select 函数，并运用于套接字编程当中。认为「select 函数是 I/O 复用的全部内容」也并不为过。select 函数的调用过程如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/adc006dbd6684478886c24315cd5908b.png)

#### 12.2.2 设置文件描述符

利用 select 函数可以同时监视多个文件描述符。当然，监视文件描述符可以视为监视套接字。此时首先需要将要监视的文件描述符集中在一起。集中时也要按照监视项（接收、传输、异常）进行区分，即按照上述 3 种监视项分成 3 类。

利用 fd_set 数组变量执行此操作，如图所示，该数组是存有0和1的位数组。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c3e08a5f733f4b16904026cee8277cf3.png)

图中最左端的位表示文件描述符 0（所在位置）。如果该位设置为 1，则表示该文件描述符是监视对象。监视对象呢是描述符 1 和 3。在 fd_set 变量中注册或更改值的操作都由下列宏完成。

- `FD_ZERO(fd_set *fdset)`：将 fd_set 变量所指的位全部初始化成0
- `FD_SET(int fd,fd_set *fdset)`：在参数 fdset 指向的变量中注册文件描述符 fd 的信息
- `FD_CLR(int fd,fd_set *fdset)`：从参数 fdset 指向的变量中清除文件描述符 fd 的信息
- `FD_ISSET(int fd,fd_set *fdset)`：若参数 fdset 指向的变量中包含文件描述符 fd 的信息，则返回「真」

上述函数中，FD_ISSET 用于验证 select 函数的调用结果，通过下图解释这些函数的功能：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c83a3e7511ff4e4b9cc2195cb50a6399.png)

#### 12.2.3 设置检查（监视）范围及超时

下面是 select 函数的定义：

```c
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfd, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
/*
成功时返回大于 0 的值，失败时返回 -1
maxfd: 监视对象文件描述符数量
readset: 将所有关注「是否存在待读取数据」的文件描述符注册到 fd_set 型变量，并传递其地址值。
writeset: 将所有关注「是否可传输无阻塞数据」的文件描述符注册到 fd_set 型变量，并传递其地址值。
exceptset: 将所有关注「是否发生异常」的文件描述符注册到 fd_set 型变量，并传递其地址值。
timeout: 调用 select 函数后，为防止陷入无限阻塞的状态，传递超时(time-out)信息
返回值: 发生错误时返回 -1,超时时返回0,。因发生关注的时间返回时，返回大于0的值，该值是发生事件的文件描述符数。
*/
```

如上所述，select 函数用来验证 3 种监视的变化情况，根据监视项声明 3 个 fd_set 型变量，分别向其注册文件描述符信息，并把变量的地址值传递到上述函数的第二到第四个参数。但在此之前（调用 select 函数之前）需要决定下面两件事：

1. 文件描述符的监视（检查）范围是？
2. 如何设定 select 函数的超时时间？

第一，文件描述符的监视范围和 select 的第一个参数有关。实际上，select 函数要求通过第一个参数传递监视对象文件描述符的数量。因此，需要得到注册在 fd_set 变量中的文件描述符数。但每次新建文件描述符时，其值就会增加 1 ，故只需将最大的文件描述符值加 1 再传递给 select 函数即可。加 1 是因为文件描述符的值是从 0 开始的。

第二，select 函数的超时时间与 select 函数的最后一个参数有关，其中 timeval 结构体定义如下：

```c
struct timeval
{
    long tv_sec;
    long tv_usec;
};
```

本来 select 函数只有在监视文件描述符发生变化时才返回。如果未发生变化，就会进入阻塞状态。指定超时时间就是为了防止这种情况的发生。通过上述结构体变量，将秒数填入 tv_sec 的成员，将微妙数填入 tv_usec 的成员，然后将结构体的地址值传递到 select 函数的最后一个参数。此时，即使文件描述符未发生变化，只要过了指定时间，也可以从函数中返回。不过这种情况下， select 函数返回 0 。因此，可以通过返回值了解原因。如果不想设置超时，则传递 NULL 参数。

#### 12.2.4 调用 select 函数查看结果

select 返回正整数时，怎样获知哪些文件描述符发生了变化？向 select 函数的第二到第四个参数传递的 fd_set 变量中将产生如图所示的变化：

![](https://s2.ax1x.com/2019/01/23/kA06dx.png)

由图可知，select 函数调用完成后，向其传递的 fd_set 变量将发生变化。原来为 1 的所有位将变成 0，但是发生了变化的文件描述符除外。因此，可以认为值仍为 1 的位置上的文件描述符发生了变化。

#### 12.2.5 select 函数调用示例

下面是一个 select 函数的例子：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/select.h>
#define BUF_SIZE 30

int main(int argc, char *argv[])
{
    fd_set reads, temps;
    int result, str_len;
    char buf[BUF_SIZE];
    struct timeval timeout;

    FD_ZERO(&reads);   //初始化变量
    FD_SET(0, &reads); //将文件描述符0对应的位设置为1

    /*
    timeout.tv_sec=5;
    timeout.tv_usec=5000;
    */

    while (1)
    {
        temps = reads; //为了防止调用了select 函数后，位的内容改变，先提前存一下
        timeout.tv_sec = 5;
        timeout.tv_usec = 0;
        result = select(1, &temps, 0, 0, &timeout); //如果控制台输入数据，则返回大于0的数，没有就会超时
        if (result == -1)
        {
            puts("select error!");
            break;
        }
        else if (result == 0)
        {
            puts("Time-out!");
        }
        else
        {
            if (FD_ISSET(0, &temps)) //验证发生变化的值是否是标准输入端
            {
                str_len = read(0, buf, BUF_SIZE);
                buf[str_len] = 0;
                printf("message from console: %s", buf);
            }
        }
    }
    return 0;
}
```

编译运行：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a98a7300b22a42f19a414f82d8831827.png)

可以看出，如果运行后在标准输入流输入数据，就会在标准输出流输出数据，但是如果 5 秒没有输入数据，就提示超时。

#### 12.2.6 实现 I/O 复用服务器端

下面通过 select 函数实现 I/O 复用服务器端。下面是基于 I/O 复用的回声服务器端。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <sys/select.h>

#define BUF_SIZE 100
void error_handling(char *message);

int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    struct timeval timeout;
    fd_set reads, cpy_reads;

    socklen_t adr_sz;
    int fd_max, str_len, fd_num, i;
    char buf[BUF_SIZE];
    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }
    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error");
    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    FD_ZERO(&reads);
    FD_SET(serv_sock, &reads); //注册服务端套接字
    fd_max = serv_sock;

    while (1)
    {
        cpy_reads = reads;
        timeout.tv_sec = 5;
        timeout.tv_usec = 5000;

        if ((fd_num = select(fd_max + 1, &cpy_reads, 0, 0, &timeout)) == -1) //开始监视,每次重新监听
            break;
        if (fd_num == 0)
            continue;

        for (i = 0; i < fd_max + 1; i++)
        {
            if (FD_ISSET(i, &cpy_reads)) //查找发生变化的套接字文件描述符
            {
                if (i == serv_sock) //如果是服务端套接字时,受理连接请求
                {
                    adr_sz = sizeof(clnt_adr);
                    clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_adr, &adr_sz);

                    FD_SET(clnt_sock, &reads); //注册一个clnt_sock
                    if (fd_max < clnt_sock)
                        fd_max = clnt_sock;
                    printf("Connected client: %d \n", clnt_sock);
                }
                else //不是服务端套接字时
                {
                    str_len = read(i, buf, BUF_SIZE); //i指的是当前发起请求的客户端
                    if (str_len == 0)
                    {
                        FD_CLR(i, &reads);
                        close(i);
                        printf("closed client: %d \n", i);
                    }
                    else
                    {
                        write(i, buf, str_len);
                    }
                }
            }
        }
    }
    close(serv_sock);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

从图上可以看出，虽然只用了一个进程，但是却实现了可以和多个客户端进行通信，这都是利用了 select 的特点。