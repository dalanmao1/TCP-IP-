[toc]

# 一、理解网络编程和套接字编程

## 1.1 socket套接字

网络编程又称为**套接字编程**，为什么要用套接字？我们把插头插到插座上就能从电网获得电力供给，同样为了与远程计算机进行数据传输，需要连接到因特网，**而编程中的“套接字”就是用来连接该网络的工具**。它本身就带有“连接”的含义，如果将其引申，则还可以表示两台计算机之间的**网络连接**。

### 1.1.1 一个例子来表示TCP的网络连接

因为拨打和接听是有区别的,因此需要分开看表示

- **接听方的流程如下：**

  1. 首先需要**安装一台电话机**

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/f3451f9af7334e54a3de1cfea250e458.png)
  2. 准备好电话机后要考虑**分配电话号码**的问题，这样别人才能联系到自己

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/1652734377af44baae0ae2f1562eb3ee.png)
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/f5c1c3d426bd4e5e90dfa89dce4316bc.png)
  3. 用blind函数给套接字分配地址后，电话机就装好了。接下来需要**连接电话线**并等待来电

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/6142e2c4a20a4b38abb843652d03e4b1.png)
  4. 连接好电话线后，如果有人拨打就会响铃，这时候需要**拿起电话线**才能对话

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/d0ec82c0361740c5bedc78f9e10eeed3.png)

- **打电话的流程如下：**

  1. 和上面一样，调用socket套接字
  2. 调用connect函数

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/03ec374688b347a3837f4f65e35751d3.png)

### 1.1.2 程序实现

所有的内容是在Linux系统下实现，因此该程序只支持Linux

服务端代码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/6418083698ed42af98816e1b513e76ee.png)

客户端代码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/4716d36ef6fe4267b818ca4ae6897d5d.png)

**实现：**
代码获取：[程序下载](https://download.csdn.net/download/u011895157/87376114)

 我是用visual studio开发，在Linux虚拟机下部署文件并且实行，定位到两个文件位置，执行两个文件，注意在文件后面需要加一个端口号(这边用到9091，可以自定义5000-6535都可)

![在这里插入图片描述](https://img-blog.csdnimg.cn/94a80a331313472390bea2193076bed2.png)

## 1.2 文件操作

### 1.2.1 文件描述符

实际上，文件描述符是为了方便称呼操作系统创建的文件或套接字而赋予的**数**。
试想一下，“把那本18号书帮我打印一下” 比 “把那本叫《物联网感知、识别与控制技术》的书帮我打印一下”是不是更简介明了？
文件描述符有时也称为文件句柄，但“句柄”主要是Windows中的术语。因此，本书中如果涉及Windows平台将使用“句柄”，如果是Linux平台则用**描述符**

分配给标准输入输出及标准错误的文件描述符。

| 文件描述符 |           对象            |
| :--------: | :-----------------------: |
|     0      | 标准输入：Standard Input  |
|     1      | 标准输出：Standard Output |
|     2      | 标准错误：Standard Error  |

- **打开文件**

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *path, int flag);
/*
成功时返回文件描述符，失败时返回-1
path : 文件名的字符串地址
flag : 文件打开模式信息
*/
```

文件打开模式如下表：

| 打开模式 |            含义            |
| :------: | :------------------------: |
| O_CREAT  |       必要时创建文件       |
| O_TRUNC  |      删除全部现有数据      |
| O_APPEND | 维持现有数据，保存到其后面 |
| O_RDONLY |          只读打开          |
| O_WRONLY |          只写打开          |
|  O_RDWR  |          读写打开          |

- **关闭文件**

```c
#include <unistd.h>
int close(int fd);
/*
成功时返回 0 ，失败时返回 -1
fd : 需要关闭的文件或套接字的文件描述符
*/
```

- **将数据写入文件**

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);
/*
成功时返回写入的字节数 ，失败时返回 -1
fd : 显示数据传输对象的文件描述符
buf : 保存要传输数据的缓冲值地址
nbytes : 要传输数据的字节数
*/
```

在此函数的定义中，size_t 是通过 typedef 声明的 unsigned int 类型。对 ssize_t 来说，ssize_t 前面多加的 s 代表 signed ，即 ssize_t 是通过 typedef 声明的 signed int 类型。

- **读取文件中的数据**

与之前的`write()`函数相对应，`read()`用来输入（接收）数据。

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t nbytes);
/*
成功时返回接收的字节数（但遇到文件结尾则返回 0），失败时返回 -1
fd : 显示数据接收对象的文件描述符
buf : 要保存接收的数据的缓冲地址值。
nbytes : 要接收数据的最大字节数
*/
```

