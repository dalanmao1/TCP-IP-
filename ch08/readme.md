**完整版文章请参考：**
[TCP/IP网络编程完整版文章](https://blog.csdn.net/u011895157/article/details/128678250?spm=1001.2014.3001.5502)

[TOC]

## 第 8 章 域名及网络地址

### 8.1 域名系统

DNS 是对IP地址和域名进行相互转换的系统，其核心是 DNS 服务器(Domain Name System)

#### 8.1.1 什么是域名

域名就是我们常常在地址栏里面输入的地址，将比较难记忆的IP地址变成人类容易理解的信息，比如这个IP (36.152.44.96) 是百度的地址，但是我们肯定记不住啊，所以将这个IP变成了 www.baidu.com 我们就都能记住了

#### 8.1.2 DNS 服务器

相当于一个字典，可以查询出某一个域名对应的IP地址

如图所示，显示了 DNS 服务器的查询路径：

![](https://i.loli.net/2019/01/18/5c41854859ae3.png)

可以用以下两条指令查询地址

- `ping` ：如 ping www.baidu.com 后即可查看百度的IP
- `nslookup` ：输入该命令之后，进一步输入网址即可查询网站IP地址

### 8.2 IP地址和域名之间的转换

#### 8.2.1 程序中有必要使用域名吗？

一句话，需要，**因为IP地址可能经常改变**，而且也不容易记忆，通过域名可以随时更改解析，达到更换IP的目的

#### 8.2.2 利用域名获取IP地址

```c
#include <netdb.h>
struct hostent *gethostbyname(const char *hostname);
/*
成功时返回 hostent 结构体地址，失败时返回 NULL 指针
*/
```

这个函数使用方便，只要传递字符串，就可以返回域名对应的IP地址。只是返回时，地址信息装入 hostent 结构体。此结构体的定义如下：

```c
struct hostent
{
    char *h_name;       /* Official name of host.  */
    char **h_aliases;   /* Alias list.  */
    int h_addrtype;     /* Host address type.  */
    int h_length;       /* Length of address.  */
    char **h_addr_list; /* List of addresses from name server.  */
};
```

从上述结构体可以看出，不止返回IP信息，同事还带着其他信息一起返回。**域名转换成IP时只需要关注 h_addr_list**。下面简要说明上述结构体的成员：

- h_name：该变量中存有官方域名（Official domain name）。官方域名代表某一主页，但实际上，一些著名公司的域名并没有用官方域名注册。
- h_aliases：可以通过多个域名访问同一主页。同一IP可以绑定多个域名，因此，除官方域名外还可以指定其他域名。这些信息可以通过 h_aliases 获得。
- h_addrtype：gethostbyname 函数不仅支持 IPV4 还支持 IPV6 。因此可以通过此变量获取保存在 h_addr_list 的IP地址族信息。若是 IPV4 ，则此变量中存有 AF_INET。
- h_length：保存IP地址长度。若是 IPV4 地址，因为是 4 个字节，则保存4；IPV6 时，因为是 16 个字节，故保存 16
- h_addr_list：这个是最重要的的成员。通过此变量以整数形式保存域名相对应的IP地址。另外，用户比较多的网站有可能分配多个IP地址给同一个域名，利用多个服务器做负载均衡，。此时可以通过此变量获取IP地址信息。

调用 gethostbyname 函数后，返回的结构体变量如图所示：

![](https://i.loli.net/2019/01/18/5c41898ae45e8.png)

下面的代码通过一个例子来演示 gethostbyname 的应用，并说明 hostent 结构体变量特性：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

void error_handling(char* message);

int main(int argc, char* argv[])
{
    int i;
    struct hostent* host;

    if (argc != 2)
    {
        printf("Usage : %s <addr>\n", argv[0]);
        exit(1);
    }
   
    host = gethostbyname(argv[1]);
    if (!host)
        error_handling("gethost... error");

    printf("Official name: %s \n", host->h_name);
    for (i = 0; host->h_aliases[i]; i++)
        printf("Aliases %d: %s \n", i + 1, host->h_aliases[i]);

    printf("Official name: %s \n", (host->h_addrtype == AF_INET) ? "AF_INET" : "AF_INET6");

    for (i = 0; host->h_aliases[i]; i++)
        printf("IP addr %d: %s \n", i + 1, inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));

    return 0;
}

void error_handling(char* message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

