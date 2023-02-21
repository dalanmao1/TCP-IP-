**完整版文章请参考：**
[TCP/IP网络编程完整版文章](https://blog.csdn.net/u011895157/article/details/128678250?spm=1001.2014.3001.5502)

[TOC]

## 第 15 章 套接字和标准I/O

### 15.1 标准 I/O 的优点

#### 15.1.1 标准 I/O 函数的两个优点

除了使用 read 和 write 函数收发数据外，还能使用标准 I/O 函数收发数据。下面是标准 I/O 函数的两个优点：

- 标准 I/O 函数具有良好的移植性
- 标准 I/O 函数可以利用缓冲提高性能

创建套接字时，操作系统会准备 I/O 缓冲。此缓冲在执行 TCP 协议时发挥着非常重要的作用。此时若使用标准 I/O 函数，将得到额外的缓冲支持。如下图：

![](https://i.loli.net/2019/01/29/5c500e53ad9aa.png)

假设使用 fputs 函数进行传输字符串 「Hello」时，首先将数据传递到标准 I/O 缓冲，然后将数据移动到套接字输出缓冲，最后将字符串发送到对方主机。

设置缓冲的主要目的是为了提高性能。从以下两点可以说明性能的提高：

- 传输的数据量
- 数据向输出缓冲移动的次数。

比较 1 个字节的数据发送 10 次的情况和 10 个字节发送 1 次的情况。发送数据时，数据包中含有头信息。头信与数据大小无关，是按照一定的格式填入的。假设头信息占 40 个字节，需要传输的数据量也存在较大区别：

- 1 个字节 10 次：40*10=400 字节
- 10个字节 1 次：40*1=40 字节。

#### 15.1.2 标准 I/O 函数和系统函数之间的性能对比

下面是利用系统函数的示例：

syscpy.c 程序：

```c
#include <stdio.h>
#include <fcntl.h>
#define BUF_SIZE 3

int main(int argc, char *argv[])
{
    int fd1, fd2;
    int len;
    char buf[BUF_SIZE];

    fd1 = open("news.txt", O_RDONLY);
    fd2 = open("cpy.txt", O_WRONLY | O_CREAT | O_TRUNC);

    while ((len = read(fd1, buf, sizeof(buf))) > 0)
        write(fd2, buf, len);

    close(fd1);
    close(fd2);

    return 0;
}

```

如果是采用上述代码，数据传输的时间需要很长

下面采用标准I/O函数复制文件

stdcpy.c 程序：

```c
#include <stdio.h>
#define BUF_SZIE 3

int main(int argc, char *argv[])
{
    FILE *fp1;
    FILE *fp2;
    char buf[BUF_SZIE];

    fp1 = open("news.txt", "r");
    fp2 = open("cpy.txt", "w");

    while (fgets(buf, BUF_SZIE, fp1) != NULL)
        fputs(buf, fp2);
    fclose(fp1);
    fclose(fp2);
    return 0;
}
```

#### 15.1.3 标准 I/O 函数的几个缺点

标准 I/O 函数存在以下几个缺点：

- 不容易进行双向通信
- 有时可能频繁调用 fflush 函数
- 需要以 FILE 结构体指针的形式返回文件描述符。

打开文件时，如果希望同时进行读操作，则应以 r+、w+、a+ 模式打开。
但因为缓冲的缘故，每次切换读写工作状态时应调用fIush为数。这也会影响基于缓冲的性能提高。

### 15.2 使用标准 I/O 函数

#### 15.2.1 利用 fdopen 函数转换为 FILE 结构体指针

函数原型如下：

```c
#include <stdio.h>
FILE *fdopen(int fildes, const char *mode);
/*
成功时返回转换的 FILE 结构体指针，失败时返回 NULL
fildes ： 需要转换的文件描述符
mode ： 将要创建的 FILE 结构体指针的模式信息
*/
```

desto.c程序：

```c
#include <stdio.h>
#include <fcntl.h>

int main()
{
    FILE *fp;
    int fd = open("data.dat", O_WRONLY | O_CREAT | O_TRUNC); //创建文件并返回文件描述符
    if (fd == -1)
    {
        fputs("file open error", stdout);
        return -1;
    }
    fp = fdopen(fd, "w"); //返回 写 模式的 FILE 指针
    fputs("NetWork C programming \n", fp);
    fclose(fp);
    return 0;
}
```

编译运行：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5c3b214d38ce4ed6848f3041e41a647f.png)

文件描述符转换为 FILE 指针，并可以通过该指针调用标准 I/O 函数。

#### 15.2.2 利用 fileno 函数转换为文件描述符

函数原型如下：

```c
#include <stdio.h>
int fileno(FILE *stream);
/*
成功时返回文件描述符，失败时返回 -1
*/
```

todes.c 程序：

```c
#include <stdio.h>
#include <fcntl.h>

int main()
{
    FILE *fp;
    int fd = open("data.dat", O_WRONLY | O_CREAT | O_TRUNC);
    if (fd == -1)
    {
        fputs("file open error");
        return -1;
    }

    printf("First file descriptor : %d \n", fd);
    fp = fdopen(fd, "w"); //转成 file 指针
    fputs("TCP/IP SOCKET PROGRAMMING \n", fp);
    printf("Second file descriptor: %d \n", fileno(fp)); //转回文件描述符
    fclose(fp);
    return 0;
}
```

### 15.3 基于套接字的标准 I/O 函数使用

把第四章的回声客户端和回声服务端的内容改为基于标准 I/O 函数的数据交换形式。

代码如下：

echo_client.c 程序：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 1024
void error_handling(char *message);

int main(int argc, char *argv[])
{
    int sock;
    char message[BUF_SIZE];
    int str_len;
    struct sockaddr_in serv_adr;
    FILE *readfp;
    FILE *writefp;
    if (argc != 3)
    {
        printf("Usage : %s <IP> <port>\n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        error_handling("socket() error");

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    if (connect(sock, (struct sockaddr *)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("connect() error!");
    else
        puts("Connected...........");
    readfp = fdopen(sock, "r");
    writefp = fdopen(sock, "w");
    while (1)
    {
        fputs("Input message(Q to quit): ", stdout);
        fgets(message, BUF_SIZE, stdin);

        if (!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
            break;

        fputs(message, writefp);
        fflush(writefp);
        fgets(message, BUF_SIZE, readfp);
        printf("Message from server: %s", message);
    }
    fclose(writefp);
    fclose(readfp);
    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

echo_stdserv.c 程序：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 1024
void error_handling(char *message);

int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    char message[BUF_SIZE];
    int str_len, i;

    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t clnt_adr_sz;
    FILE *readfp;
    FILE *writefp;

    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    if (serv_sock == -1)
        error_handling("socket() error");

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error");

    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    clnt_adr_sz = sizeof(clnt_adr);
    //调用 5 次 accept 函数，共为 5 个客户端提供服务
    for (i = 0; i < 5; i++)
    {
        clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_adr, &clnt_adr_sz);
        if (clnt_sock == -1)
            error_handling("accept() error");
        else
            printf("Connect client %d \n", i + 1);

        readfp = fdopen(clnt_sock, "r");
        writefp = fdopen(clnt_sock, "w");
        while (!feof(readfp))
        {
            fgets(message, BUF_SIZE, readfp);
            fputs(message, writefp);
            fflush(writefp);
        }

        fclose(readfp);
        fclose(writefp);
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
