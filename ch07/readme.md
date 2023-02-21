**完整版文章请参考：**
[TCP/IP网络编程完整版文章](https://blog.csdn.net/u011895157/article/details/128678250?spm=1001.2014.3001.5502)

[TOC]

## 第 7 章 优雅的断开套接字的连接

### 7.1 基于 TCP 的半关闭

TCP 的断开连接过程比建立连接更重要，因为连接过程中一般不会出现大问题，但是断开过程可能发生预想不到的情况。因此应该准确掌控，所以要**掌握半关闭（Half-close）**，才能明确断开过程。

#### 7.1.1 单方面断开连接带来的问题

Linux 的 close 函数和 Windows 的 closesocket 函数意味着完全断开连接。完全断开不仅指无法传输数据，而且也不能接收数据。因此在某些情况下，通信一方调用 close 函数或者 closesocket 函数，显得不太优雅。如图所示：

![](https://i.loli.net/2019/01/18/5c412a8baa2d8.png)

图中描述的是 2 台主机正在进行双向通信，主机 A 发送完最后的数据后，调用 close 函数断开了最后的连接，之后主机 A 无法再接受主机 B 传输的数据。实际上，是完全无法调用与接受数据相关的函数。最终，由主机 B 传输的、主机 A 必须要接受的数据也销毁了。

为了解决这类问题，**只关闭一部分数据交换中使用的流**的方法应运而生。断开一部分连接是指，可以传输数据但是无法接收，或**可以接受数据但无法传输**。顾名思义就是只关闭流的一半。

#### 7.1.2 套接字和流（Stream）

两台主机通过套接字建立连接后进入可交换数据的状态，又称**流形成的状态**。也就是把建立套接字后可交换数据的状态看作一种流。

此处的流可以比作水流。水朝着一个方向流动，同样，在套接字的流中，数据也止呕能向一个方向流动。因此，为了进行双向通信，需要如图所示的两个流：

![](https://i.loli.net/2019/01/18/5c412c3ba25dd.png)

一旦两台主机之间建立了套接字连接，每个主机就会拥有单独的输入流和输出流。**其中一个主机的输入流与另一个主机的输出流相连，而输出流则与另一个主机的输入流相连**。断开连接方式只断开其中 1 个流，而非同时断开两个流。Linux 的 close 函数和 Windows 的 closesocket 函数将同时断开这两个流，这样的方式确实不够优雅。

#### 7.1.3 针对优雅断开的 shutdown 函数

shutdown 用来关闭其中一个流：

```c
#include <sys/socket.h>
int shutdown(int sock, int howto);
/*
成功时返回 0 ，失败时返回 -1
sock: 需要断开套接字文件描述符
howto: 传递断开方式信息
*/
```

调用上述函数时，第二个参数决定断开连接的方式，其值如下所示：

- `SHUT_RD` : 断开输入流
- `SHUT_WR` : 断开输出流
- `SHUT_RDWR` : 同时断开 I/O 流

若向 shutdown 的第二个参数传递 `SHUT_RD`，则断开输入流，套接字无法接收数据。即使输入缓冲收到数据也会抹去，而且无法调用相关函数。如果向 shutdown 的第二个参数传 `SHUT_WR`，则中断输出流，也就无法传输数据。若如果输出缓冲中还有未传输的数据，则将传递给目标主机。最后，若传递关键 `SHUT_RDWR`，则同时中断 I/O 流。这相当于分 2 次调用 shutdown ，其中一次以`SHUT_RD`为参数，另一次以 `SHUT_WR`为参数。

#### 7.1.4 为何要半关闭

考虑以下情况：

> 客户端断开连接前还有数据要传输的情况。

比客户端连接到服务器，服务器将约定的文件传输给客户端，客户端收到后发送字符串 Thank you 给服务器端，但在这之前两者断开了，那么Thank you发不出去了。

程序的实现难度并不小，因为传输文件的服务器端只需连续传输文件数据即可，而客户端无法知道需要接收数据到何时。客户端也没办法无休止的调用输入函数，因为这有可能导致程序**阻塞**。

> 是否可以让服务器和客户端约定一个代表文件尾的字符？

这种方式也有问题，因为这意味这文件中不能有与约定字符相同的内容。为了解决该问题，服务端应最后向客户端传递 EOF 表示文件传输结束。客户端通过函数返回值接收 EOF ，这样可以避免与文件内容冲突。那么问题来了，服务端如何传递 EOF ？

> 断开输出流时向主机传输 EOF。

如果调用 close 函数的会关闭 I/O 流，这样也会向对方发送 EOF 。但此时无法再接受对方传输的数据。换言之，若调用 close 函数关闭流，就无法接受客户端最后发送的字符串「Thank you」。这时需要调用 shutdown 函数，只关闭服务器的输出流。这样既可以发送 EOF ，同时又保留了输入流。下面实现收发文件的服务器端/客户端。

#### 7.1.5 基于半关闭的文件传输程序

上述文件传输服务器端和客户端的数据流可以整理如图：

![](https://i.loli.net/2019/01/18/5c41326280ab5.png)

代码如下：

服务端：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char* message);

int main(int argc, char* argv[])
{
    int serv_sd, clnt_sd;
    FILE* fp;
    char buf[BUF_SIZE];
    int read_cnt;

    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t clnt_adr_sz;

    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }
    fp = fopen("file_server.c", "rb");
    serv_sd = socket(PF_INET, SOCK_STREAM, 0);

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    bind(serv_sd, (struct sockaddr*)&serv_adr, sizeof(serv_adr));
    listen(serv_sd, 5);

    clnt_adr_sz = sizeof(clnt_adr);
    clnt_sd = accept(serv_sd, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);

    while (1)
    {
        //从文件流中读取数据，buffer为接收数据的地址，size为一个单元的大小，count为单元个数，stream为文件流
        //返回实际读取的单元个数
        read_cnt = fread((void*)buf, 1, BUF_SIZE, fp);
        if (read_cnt < BUF_SIZE)
        {
            write(clnt_sd, buf, read_cnt);
            break;
        }
        write(clnt_sd, buf, BUF_SIZE);
    }

    shutdown(clnt_sd, SHUT_WR);
    read(clnt_sd, buf, BUF_SIZE);
    printf("Message from client: %s \n", buf);

    fclose(fp);
    close(clnt_sd);
    close(serv_sd);
    return 0;
}

void error_handling(char* message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

客户端：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char* message);

int main(int argc, char* argv[])
{
    int sd;
    FILE* fp;

    char buf[BUF_SIZE];
    int read_cnt;
    struct sockaddr_in serv_adr;

    if (argc != 3)
    {
        printf("Usage : %s <IP> <port>\n", argv[0]);
        exit(1);
    }

    fp = fopen("receive.dat", "wb");
    sd = socket(PF_INET, SOCK_STREAM, 0);

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    connect(sd, (struct sockaddr*)&serv_adr, sizeof(serv_adr));

    while ((read_cnt = read(sd, buf, BUF_SIZE)) != 0)
        fwrite((void*)buf, 1, read_cnt, fp);

    puts("Received file data");
    write(sd, "Thank you", 10);
    fclose(fp);
    close(sd);
    return 0;
}

void error_handling(char* message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

编译运行：

![在这里插入图片描述](https://img-blog.csdnimg.cn/6e74b9ede37e475cb8861abd4b0076b2.png)