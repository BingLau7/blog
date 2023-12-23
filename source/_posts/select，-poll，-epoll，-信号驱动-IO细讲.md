---
title: select， poll， epoll， 信号驱动 IO细讲
date: 2017-04-30 16:51:10
tags:
    - IO
    - 异步编程
categories: 基础原理
---

> 63章 其他备选的 IO 模型

### 整体概览

实际上 I/O 多路复用，信号驱动 I/O 以及 epool 都是用来实现同一个目标的技术——同时检查多个文件描述符，看它们是否准备好了执行 I/O 操作（准确地说，是看 I/O 系统调用是否可以非阻塞地执行）。文件描述符就绪状态的转化是通过一些 I/O 事件来触发的，比如输入数据到达，套接字连接建立完成，或者是之前满载的套接字发送缓冲区在 TCP 将队列中的数据传送到对端之后由了剩余空间。同时检查多个文件描述符在类似网络服务器的应用中很有用处，或者是那么必须同事检查终端以及管道或套接字输入的应用程序。

需要注意的是这些技术都不会执行实际的 I/O 操作。它们只是告诉我们某个文件描述符已经处于就绪状态了，这时需要调用其他的系统调用来完成实际的 I/O 操作。

<!-- more -->

#### 水平触发和边缘触发

- 水平触发通知：如果文件描述符上可以非阻塞地执行 I/O 系统调用，此时任务它已经就绪。
- 边缘触发通知：如果文件描述符自上次状态检查以来有了新的 I/O 活动（比如新的输入），此时需要触发通知。

|      I/O 模式      | 水平触发 | 边缘触发 |
| :--------------: | :--: | :--: |
| select(), pool() |  √   |      |
|     信号驱动 I/O     |      |  √   |
|      epool       |  √   |  √   |

当采用水平触发通知时，我们可以在任意时刻检查文件描述符的就绪状态。这表示当我们确定了文件描述符处于就绪态时（比如存在输入数据），就可以对其执行一些 I/O 操作，然后重复检查文件描述符，看看是否仍然处于就绪态（比如还有更多的输入数据），此时我们就能执行更多的 I/O，以此类推。换句话说，由于水平触发模式允许我们在任意时刻重复检查 I/O 状态，没有必要每次当文件描述符就绪后需要尽可能多地执行 I/O （也就是尽可能多地读取字节，亦或是根本不去执行任何 I/O）。

与此相反的是，当我们采用边缘触发时，只有当 I/O 事件发生时我们才会收到通知。在另一个 I/O 事件到来前我们不会收到任何新的通知。另外，当文件描述符收到 I/O 事件通知时，通常我们并不知道要处理多少 I/O（例如有多少字节可读）。因此，采用边缘触发通知的程序通常要按照如下规则来设计。

- 在接收到一个 I/O 事件通知后，程序在某个时刻应该在相应的文件描述符上尽可能多地执行 I/O（比如尽可能多地读取字节）。如果程序没有这么做，那么就可能失去执行 I/O 的机会。因为直到产生另一个 I/O 事件位置，在此之前程序都不会再接收到通知了，因此也就不知道此时应该执行 I/O 操作。这将导致数据丢失或者程序中出现阻塞。
- 如果程序采用循环来对文件描述符执行尽可能多的 I/O ，而文件描述符又被置为可阻塞的，那么最终当没有更多的 I/O 可执行时，I/O 系统调用就会被阻塞。基于这个原因，每个被检查的文件描述符通常都应该置为非阻塞模式，在得到 I/O 事件通知后重复执行 I/O 操作，直到相应的系统调用(比如 write(), read()) 以错误码 EAGAIN 或 EWOULDBLOCK 的形式失联。

### I/O 多路复用

#### select() 系统调用

系统调用 `select()` 会一直阻塞，直到一个或多个文件描述符集合成为就绪态。

```C
#include<sys/time.h>
#include<sys/select.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

// Retrun number of ready file descriptors, 0 on timeout, or -1 on error.
```

参数 `readfds`, `writefds` 以及 `exceptfds` 都是指向文件描述符集合的指针，所指向的数据类型是 fd_set,。这些参数按照如下方式使用。

-  `readfds` 是用来检测输入是否就绪的文件描述符集合

- `writefds` 是用来检测输出是否就绪的文件描述符集合

- `exceotfds` 是用来检测异常情况是否发生的文件描述符集合

   在 Linux 上，一个异常情况只有下面两种情况下发生：

   -  连接到处于信号模式下的伪终端设备上的从设备状态发生了改变
   -  流式套接字上接收到了带外数据

通常，数据类型 `fd_set` 以位掩码的形式来实现。但是是由下面四个宏来完成

```C
#include <sys/select.h>

void FD_ZERO(fd_set *fdset);             // 初始化为空
void FD_SET(int fd, fd_set *fdset);      // 将文件描述符 fd 添加入集合
void FD_CLR(int fd, fd_set *fdset);      // 将 fd 移除
int FD_ISSET(int fd, fd_set *fdset);     // fd 是否在集合中，是返回1，否则返回0
```

文件描述符集合有一个最大容量限制，由常量 FD_SETSIZE 来决定。在 Linux 上，该常量的值为1024。

参数`readfds`、`writefds` 和 `exceptfds` 所指向的结构体都是保存结果值的地方。在调用 `select()` 之前，这些参数指向的结构体必须初始化（通过`FD_ZERO() 和 FD_SET()`），以包含我们感兴趣的文件描述符集合。之后 `select()` 调用会修改这些结构体，当 `select()` 返回时，它们包含的就是已处于就绪态的文件描述符集合了（值-结果参数）。（由于这些结构体会在调用中被修改，如果要在循环中反复调用`select()`，我们必须保证每次都要重新初始化它们。）之后这些结构体可以通过 `FD_ISSET()` 来检查。

**timeout 参数**

参数 `timeout` 控制着 `select()` 的阻塞行为。该参数可指定为 `NULL`，此时 `select()` 会一直阻塞。又或者是指向一个  `timeval` 结构体。

```c
struct timeval {
  time_t      tv_sec;      /*Seconds */
  suseconds_t tv_usec;     /* Microseconds (long int) */
};
```

如果结构体 `timeval` 的两个域都为0的话，此时 `select()` 不会阻塞，它只是简单地轮询指定的文件描述符集合，看看其中是否有就绪的文件描述符并立即返回。否则，`timeout` 将为 `select()` 指定一个等待时间的上限值。

当 `timeout` 设为 `NULL` ，或其指向的结构体字段非零时， `select()` 将阻塞直到有下列事件发生：

-   `readfds`、 `writefds` 或 `exceptfds` 中指定的文件描述符中至少有一个成为就绪态；
-   该调用被信号处理例程中断
-   `timeout` 中指定的时间上限已超时

`select()` 返回所在3个集合中被标记为就绪态的文件描述符总数。如果返回 -1 则是错误发生，包括 `EBADF` 和 `EINTR`。如果返回 0 则说明超时。

##### 示例程序

```C
#include <sys/time.h>
#include <sys/select.h>

#include "t_select.h"
#include "tlpi_hdr.h"

static void
usageError(const char *progName)
{
    fprintf(stderr, "Usage: %s {timeout|-} fd-num[rw]...\n", progName);
    fprintf(stderr, "    - means infinite timeout; \n");
    fprintf(stderr, "    r = monitor for read\n");
    fprintf(stderr, "    w = monitor for write\n\n");
    fprintf(stderr, "    e.g.: %s - 0rw 1w\n", progName);
    exit(EXIT_FAILURE);
}

/**
 * 介绍一下用法
 * ./basic 10 0r 1w 第一个10是超时秒数， 0是文件描述符号，r代表读文件，w 代表写文件
 * 文件描述符中，0是标准输入，1是标准输出
 */
void
testSelect(int argc, char *argv[])
{
    fd_set readfds, writefds;
    int ready, nfds, fd, numRead, j;
    struct timeval timeout;
    struct timeval *pto;
    char buf[10];                       /* Large enough to hold "rw\0" */

    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageError(argv[0]);

    /* Timeout for select() is specified in argv[1] */

    if (strcmp(argv[1], "-") == 0) {
        pto = NULL;                     /* Infinite timeout */
    } else {
        pto = &timeout;
        // 输入中获取时间
        timeout.tv_sec = getLong(argv[1], 0, "timeout");
        timeout.tv_usec = 0;            /* No microseconds */
    }

    /* Process remaining arguments to build file descriptor sets */

    // 初始化读写描述符集合
    nfds = 0;
    FD_ZERO(&readfds);
    FD_ZERO(&writefds);

    // 分别将读描述符和写描述符加入
    for (j = 2; j < argc; j++) {
        numRead = sscanf(argv[j], "%d%2[rw]", &fd, buf);
        if (numRead != 2)
            usageError(argv[0]);
        if (fd >= FD_SETSIZE)
            cmdLineErr("file descriptor exceeds limit (%d)\n", FD_SETSIZE);

        if (fd >= nfds)
            nfds = fd + 1;              /* Record maximum fd + 1 */
        if (strchr(buf, 'r') != NULL)
            FD_SET(fd, &readfds);
        if (strchr(buf, 'w') != NULL)
            FD_SET(fd, &writefds);
    }

    /* We've built all of the arguments; now call select() */

    // 调用
    ready = select(nfds, &readfds, &writefds, NULL, pto);
    /* Ignore exceptional events */
    if (ready == -1)
        errExit("select");

    /* Display results of select() */

    printf("ready = %d\n", ready);
    for (fd = 0; fd < nfds; fd++)
        printf("%d: %s%s\n", fd, FD_ISSET(fd, &readfds) ? "r" : "",
               FD_ISSET(fd, &writefds) ? "w" : "");

    if (pto != NULL)
        printf("timeout after select(): %ld.%03ld\n",
               (long) timeout.tv_sec, (long) timeout.tv_usec / 1000);
    exit(EXIT_SUCCESS);
}
```

#### `poll()` 系统调用

系统调用 `poll()` 执行的任务同 `select()` 很相似。两者间主要的区别在于我们要如何制定待检查的文件描述符。在 `select()` 中，我们提供三个集合，在每个集合中标明我们感兴趣的文件描述符。而在  `poll()` 中我们提供一列文件描述符，并在每个文件描述符上标明我们感兴趣的事件。

```C
#include <poll.h>

int poll(struct pollfd fds[], nfds_t nfds, int timeout);

// Returns number of ready file descriptors, 0 on timeout, or -1 on error
```

参数 `fds` 列出了我们需要 `poll()` 来检查的文件描述符。该参数为 `pollfd` 结构体数组，其定义如下。

```C
struct pollfd {
  int   fd;           /* File descriptor */
  short events;       /* Requested events bit mask */
  short revents;      /* Returned events bit mask */
};
```

`pollfd` 结构体中的 `events` 和 `revents` 字段都是位掩码。调用者初始化 `events` 来指定需要为描述符 `fd` 做检查的事件。当 `poll()` 返回时，`revents` 被设定以此来表示该文件描述符上实际发生的事件。

**输入事件相关位掩码**

| 位掩码        | events 中的输入 | 返回到 revents | 描述                   |
| ---------- | :---------: | :---------: | -------------------- |
| POLLIN     |     √      |      √      | 可读取非高优先级的数据          |
| POLLRDNORM |      √      |      √      | 等同于 POLLIN           |
| POLLRDBAND |      √      |      √      | 可读取优先级数据（Linux 中不使用） |
| POLLPRI    |      √      |      √      | 可读取高优先级数据            |
| POLLRDHUP  |      √      |      √      | 对端套接字关闭              |

**输出事件相关位掩码**

| 位掩码        | events 中的输入 | 返回到 revents | 描述          |
| ---------- | :---------: | :---------: | ----------- |
| POLLOUT    |      √      |      √      | 普通数据可写      |
| POLLWRNORM |      √      |      √      | 等同于 POLLOUT |
| POLLWRBAND |      √      |      √      | 优先级数据可写入    |

**返回有关文件描述符附加信息的位掩码**

| 位掩码      | events 中的输入 | 返回到 revents | 描述       |
| -------- | :---------: | :---------: | -------- |
| POLLERR  |             |      √      | 有错误发生    |
| POLLHUP  |             |      √      | 出现挂断     |
| POLLNVAL |             |      √      | 文件描述符未打开 |

**timeout 参数**

-  -1：阻塞直到有一个文件描述符达到就绪态或者捕获到一个信号
-  0: 不会阻塞，只是执行一次检查看看哪个文件描述符处于就绪态
-  大于0：至多阻塞 timeout 毫秒，知道 fds 列出的文件描述符中有一个达到就绪态，或者知道捕获到一个信号位置。

**返回值**

-  -1：有错误
-  0：超时
-  大于0：表示数组 fds 重有用非零 revents 字段的 pollfd 结构体数量

##### 示例程序

```C
#include <time.h>
#include <poll.h>
#include "poll_pipes.h"
#include "tlpi_hdr.h"

/**
 * 这个程序创建了一些管道（每个管道使用一对连续的文件描述符），将
 * 字节写到随机选择的管道写端，然后通过 poll() 来检查看哪个管道中有数据可进行读取
 */

int
testPoll(int argc, char *argv[])
{
    int numPipes, j, ready, randPipe, numWrites;
    int (*pfds)[2];                                 /* File descriptors for all pipes */
    struct pollfd *pollFd;


    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s num-pipes [num-writes]\n", argv[0]);

    // 管道数
    numPipes = getInt(argv[1], GN_GT_0, "num-pipes");

    // 分配读写管道文件描述符内存
    pfds = calloc(numPipes, sizeof(int [2]));
    if (pfds == NULL)
        errExit("malloc");
    // 对于每对管道分配一个 pollFd 内存
    pollFd = calloc(numPipes, sizeof(struct pollfd));
    if (pollFd == NULL)
        errExit("malloc");

    for (j = 0; j < numPipes; j++)
        // 创建管道, pfds[j][0] 是读管道，pfds[j][1] 是写管道
        if (pipe(pfds[j]) == -1)
            errExit("pipe %d", j);

    // 可写数据管道数
    numWrites = (argc > 2) ? getInt(argv[2], GN_GT_0, "num-writes") : 1;
    srandom((int) time(NULL));
    for (int j = 0; j < numWrites; ++j) {
        randPipe = random() % numPipes;
        printf("Writing to fd: %3d (read fd: %3d)\n",
                pfds[randPipe][1], pfds[randPipe][0]);
        // 写数据进入管段
        if (write(pfds[randPipe][1], "a", 1) == -1)
            errExit("write %d", pfds[randPipe][1]);
    }

    // 设置 pollFd
    for (int j = 0; j < numPipes; ++j) {
        pollFd[j].fd = pfds[j][0];
        pollFd[j].events = POLLIN;
    }

    // 调用 poll
    ready = poll(pollFd, numPipes, -1);
    if (ready == -1)
        errExit("poll");

    printf("poll() returned: %d\n", ready);

    // 得到哪些管段被写然后可读
    for (int j = 0; j < numPipes; ++j) {
        if (pollFd[j].revents & POLLIN)
            printf("Readable: %d %3d\n", j, pollFd[j].fd);
    }
}

```

#### 文件描述符何时就绪

`select()` 和 `poll()` 只会告诉我们 I/O 操作是否会阻塞，而不是告诉我们到底能否成功传输数据。

##### 普通文件

代表普通文件的文件描述符总是被 `select()` 标记为可读和可写。对于 `poll()` 来说，则会在 `revents` 字段返回 POLLIN 和 POLLOUT 标志。原因如下：

-  `read()` 总是会立刻返回数据、文件结尾符或者错误
-  `write()` 总是会立刻传输数据或者因出现某些错误而失败

##### 终端和伪终端

**在终端和伪终端上 `select()` 和 `poll()` 所代表的含义**

| 条件或事件                     | `select()` | `poll()` |
| ------------------------- | :--------: | :------: |
| 有输入                       |     r      |  POLLIN  |
| 可输出                       |     w      | POLLOUT  |
| 伪终端对端调用 `close()` 后       |     rw     | POLLHUP  |
| 处于信包模式下的伪终端主设备检测到从设备端状态改变 |     x      | POLLPRI  |

##### 管道和 FIFO

**`select()` 和 `poll()` 在管道或 FIFO 读端上的通知**

| 条件或事件        | `select()` |    `poll()`     |
| ------------ | :--------: | :-------------: |
| 管道无数据，写端没打开 |     r      |     POLLHUP     |
| 管道有数据，写端打开   |     r      |     POLLIN      |
| 管道有数据，写端没打开  |     r      | POLLIN\|POLLHUP |

**`select()` 和 `poll()` 在管道或 FIFO 写端上的通知**

| 条件或事件                    | `select()` |     `poll()`     |
| ------------------------ | :--------: | :--------------: |
| 没有 PIPE_BUF 个字节空间，读端没打开 |     w      |     POLLERR      |
| 有 PIPE_BUF 个字节空间，读端打开   |     w      |     POLLOUT      |
| 有 PIPE_BUF 个字节空间，读端没打开  |     w      | POLLOUT\|POLLERR |

##### 套接字

| 条件或事件                               | `select()` |          `poll()`          |
| ----------------------------------- | :--------: | :------------------------: |
| 有输入                                 |     r      |           POLLIN           |
| 可输出                                 |     w      |          POLLOUT           |
| 在监听套接字上建立连接                         |     r      |           POLLIN           |
| 接收到带外数据（只限 TCP）                     |     x      |            POLL            |
| 流套接字的对端关闭连接或执行了 `shutdown(SHUT_WR)` |     rw     | POLLIN\|POLLOUT\|POLLRDHUP |

#### 比较 `select()` 和 `poll()`

##### 实现细节

在 Linux 内核层面，`select()` 和 `poll()` 都使用了相同的内核 `poll` 例程集合。这些例程有别于系统调用 `poll()` 本身。每个例程都返回有关单个文件描述符就绪的信息。这个就绪信息以位掩码的形式返回，其值同 `poll()` 系统调用中返回的 `revents` 字段中的比特值相关。 `poll()` 系统调用的实现包括为每个文件描述符调用内核 `poll` 例程，并将结果信息填到对应的 `revents` 字段中去。

##### API 之间的区别

-  `select()` 有文件描述符上限
-  由于 `select()` 的参数 `fd_set` 同事也是保存调用结果的地方，如果要在循环中重复调用 `select()` 的话，我们必须每次都要重新初始化 `fd_set`。而`poll()`通过独立的两个字段`events` （针对输入）和 `revents` （针对输出）来处理，从而避免每次都要重新初始化参数。
-  `select()` 提供的超时精度比较高
-  如果其中一个被检查的文件描述符关闭了，通过在对应的 `revents` 字段中设定 `POLLNVAL` 标记，`poll()` 会准确高数我们是哪一个文件描述符关闭了。与之相反，`select()` 只会返回 -1，并设错误码为 `EBADF`。通过在描述符上执行 I/O 系统调用并检查错误码，让我们自己来判断哪个文件描述符关闭了。

##### 性能

当满足如下两条中任意一条时，`poll()` 和 `select()` 将具有相似的性能表现。

-  待检查的文件描述符范围较小
-  有大量的文件描述符待检查，但是它们分布得很密集。

然而，如果被检查的文件描述符集合很稀疏的话，`select()` 和 `poll()` 的性能差异将变得非常明显，在这种情况下，后者更优。

#### `select()` 和 `poll()` 存在的问题

当检查大量的文件描述符时，这两个 API 都会遇到一些问题

-  每次调用 `select()` 或 `poll()`，内核都必须检查所有被指定的文件描述符，看它们是否处于就绪态。当检查大量处于密集范围，该挫折耗费时间将大大超过接下来的操作。
-  每次调用 `select()` 或 `poll()`，程序都必须传递一个表示所有需要被检查的文件描述符的数据结构到内核，内核检查过描述符后，修改这个数据结构并返回给程序。（此外，对于`select()` 来说，我们还必须在每次调用前初始化这个数据结构。）对于 `poll()`来说，随着待检查的文件描述符数量的增加，传递给内核的数据结构大小也会随之增加。当检查大量文件描述符时，从用户控件到内核控件来回拷贝这个数据结构将占用大量的 CPU 时间。对于`select()` 来说，这个数据结构的大小固定为 `FD_SETSIZE`，与待检查的文件描述符数量无关。
-  `select()` 或 `poll()` 调用完成后，程序必须检查返回的数据结构中的每个元素，以此查明哪个文件描述符处于就绪态了。

### 信号驱动 IO

在信号驱动 I/O 中，当文件描述符上可执行 I/O 操作时，进程请求内核为自己发送一个信号。之后进程就可以执行任何其他的任务知道 I/O 就绪为止，此时内核会发送信号给进程。要使用信号驱动 I/O，程序需要按照如下步骤来执行。

1. 为内核发送的通知信号安装一个信号处理例程。默认情况下，这个通知信号为 `SIGIO`。

2. 设定文件描述符的属主，也就是当文件描述符上可执行 I/O 时会接收到通知信号的进程或进程组。通常我们让调用进程成为属主。设定属主可通过 `fcntl()` 的 `F_SETOWN` 操作来完成:

   `fcntl(fd, F_SETOWN, pid);`

3. 通过设定 `O_NONBLOCK` 标志使能非阻塞 I/O。

4. 通过打开 `O_ASYNC` 标志使能信号驱动 I/O。这可以和上一步合并为一个操作，因为它们都需要用到 `fcntl()` 的 `F_SETFL` 操作。

   ```c
   flags = fcntl(fd, F_GETFL);
   fcntl(fd, F_SETFL, flags | O_ASYNC | ONONBLOCK);
   ```

5. 调用进程现在可以执行其他的任务了。当 I/O 操作就绪时，内核为进程发送一个信号，然后调用在第 1 步中安装好的信号处理例程。

6. 信号驱动 I/O 提供的是边缘触发通知。这表示一旦进程被通知 I/O 就绪，它就应该尽可能多地执行 I/O （例如尽可能多地读取字节）。假设文件描述符是非阻塞式的，这表示需要在循环中执行 I/O 系统调用直到失败为止，此时错误码 `EAGAIN` 或 `EWOULDBLOCK`。

#### 示例程序

```C
#include <signal.h>
#include <ctype.h>
#include <fcntl.h>
#include <termios.h>
#include "tlpi_hdr.h"
#include "tty_functions.h"


static volatile sig_atomic_t gotSigio = 0;

static void
sigioHandler(int sig)
{
    gotSigio = 1;
}

int
testSigio(int argc, char *argv[])
{
    int flags, j, cnt;
    struct termios origTermios;
    char ch;
    struct sigaction sa;
    Boolean done;

    // 1
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sa.sa_handler = sigioHandler;
    if (sigaction(SIGIO, &sa, NULL) == -1)
        errExit("sigaction");

    // 2
    if (fcntl(STDIN_FILENO, F_SETOWN, getpid()) == -1)
        errExit("fcntl(F_SETOWN)");

    // 3 & 4
    flags = fcntl(STDIN_FILENO, F_GETFL);
    if (fcntl(STDIN_FILENO, F_SETFL, flags | O_ASYNC | O_NONBLOCK) == -1)
        errExit("fcntl(F_SETFL");

    if (ttySetCbreak(STDIN_FILENO, &origTermios) == -1)
        errExit("ttySetCbreak");

    for (done = FALSE, cnt = 0; !done; cnt++) {
        for (j = 0; j < 10000000; j++)
            continue;

        // 被触发
        if (gotSigio) {
            while (read(STDIN_FILENO, &ch, 1) > 0 && !done) {
                printf("cnt=%d; read %c\n", cnt, ch);
                done = ch == '#';
            }

            gotSigio = 0;
        }
    }

    if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &origTermios) == -1)
        errExit("tcsetattr");
    exit(EXIT_SUCCESS);
}
```

#### 何时发送『I/O 就绪』信号

##### 终端和伪终端

当产生新的输入时

##### 管道和 FIFO

-  读端：
   -  数据写入到管道中（即使已经有未读取的输入存在）
   -  管道的写端关闭
-  写端：
   -  对管道的读操作增加了管道中的空间大小，因此现在可以写入 PIPE_BUF 个字节而不被阻塞
   -  管道的读端关闭


##### 套接字

-  Unix 和 Internet 域下的数据报套接字
   -  一个输入数据报达到套接字（即使已经有未读取的数据报正等待读取）
   -  套接字上发生了异步错误
-  Unix 和 Internet 域下的流式套接字
   -  监听套接字上接收到新的连接
   -  TCP `connect()` 请求完成，也就是 TCP 连接的主动端进入 `ESTABLISHED` 状态。
   -  套接字上接收到了新的输入（即使已经有未读取的输入存在）
   -  套接字对端使用 `showdown()` 关闭了写连接（半关闭），或者通过 `close()` 完全关闭
   -  套接字上输出就绪
   -  套接字上发生了异步错误

##### inotify 文件描述符

当 inotify 文件描述符称为可读状态时会产生一个信号——也就是由 inotify 文件描述符监视的其中一个文件上有事件发生时。

### epoll 编程接口

`epoll`API  的主要优点如下：

-  当检查大量的文件描述符时，`epoll` 的性能延展性比 `select()` 和 `poll()` 高很多。
-  `epoll`API 既支持水平触发也支持边缘触发。与之相反，`select()` 和 `poll()` 只支持水平触发，而信号驱动 I/O 只支持边缘触发。
-  可以避免复杂的信号处理流程（比如信号队列溢出时的处理）。
-  灵活性高，可以指定我们希望检查的事件类型（例如，检查套接字文件描述符的读就绪，写就绪或者两者同时指定）。

`epoll`API 的核心数据结构称作 `epoll` 实例，它和一个打开的文件描述符相关联。这个文件描述符不是用来做 I/O 操作的，相反，它是内核数据结构的句柄，这些内核数据结构实现了两个目的。

-  记录了在进程中声明过的感兴趣的文件描述符列表 —— interest list (兴趣列表)
-  维护了处于 I/O 就绪态的文件描述符列表 —— ready list (就绪列表)

ready list 中的成员是 interest list 的子集。

`epoll`API 由以下 3 个系统调用组成：

-  系统调用 `epoll_create()` 创建一个 `epoll` 实例，返回代表该实例的文件描述符。
-  系统调用 `epoll_ctl()` 操作同 `epoll` 实例相关联的兴趣列表。通过 `epoll_ctl()`，我们可以增加新的描述符到列表中，将已有的文件描述符从该列表中移除，以及修改代表文件描述符上事件类型的位掩码。
-  系统调用 `epoll_wait()` 返回与 `epoll` 实例相关联的就绪列表中的成员。

#### 创建 `epoll` 实例： `epoll_create()`

系统调用 `epoll_create()` 创建了一个新的 `epoll` 实例，其对应的兴趣列表初始化为空。

```C
#include <sys/epoll.h>

int epoll_create(int size);

// Return file descriptor on success, or -1 on error

// size 指定想要检查的文件描述符个数，该参数并不是上限，而是告诉内核应该如何为内部数据结构划分初始大小
```

作为函数返回值，`epoll_create()` 返回了代表新创建的 `epoll()` 实例的文件描述符。这个文件描述符在其他几个 `epoll` 系统调用中用来表示 `epoll` 实例。当这个文件描述符不再需要时，应该通过 `close()` 来关闭。当所有与 `epoll` 实例相关的文件描述符都背关闭时，实例被销毁，相关的资源都返还给系统。

#### 修改 `epoll` 的兴趣列表： `epoll_ctl()`

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);

// return 0 on success, or -1 on error
```

-  `fd` 指定要修改的文件描述符，它甚至可以是另一个 `epoll` 实例的文件描述符，但是，这里 `fd` 不能作为普通文件或目录的文件描述符

- `op` 指定需要执行的操作：
   -  `EPOLL_CTL_ADD`，加入兴趣列表中
   -  `EPOLL_CTL_MOD`，修改 fd 上设定的时间，需要用到由 ev 所指向的结构体中的信息
   -  `EPOLL_CTL_DEL`，移除兴趣列表

- `ev` 是指向结构体 `epoll_even `  的指针，结构体定义如下:

   ```C
   struct epoll_event {
     uint32_t     event;         /* epoll events (bit mask) */
     epoll_data_t data;          /* User data */
   };
   ```

   其中 data 的字段如下

   ```C
   typedef union epoll_data {
     void       *ptr;           /* Pointer to user-defined data */
     int        fd;             /* File descriptor */
     uint32_t   u32;            /* 32-bit integer */
     uint64_t   u64;            /* 64-bit integer */
   };
   ```

   参数 `ev` 为文件描述符 `fd` 所做的设置如下：

   -  结构体 `epoll_event` 中的 `events` 字段是一个位掩码，它指定了我们为待检查的描述符 `fd` 上感兴趣的事件集合。
   -  `data` 字段是一个联合体，当描述符 `fd` 稍后称为就绪态时，联合体的成员可用来指定传回给调用进程的信息。

**使用 `epoll_create()` 和 `epoll_ctl()`**

```C
int epfd;
struct epoll_event ev;

epfd = epoll_create(5);
if (epfd == -1)
  errExit("epoll_create");

ev.data.fd = fd;
ev.events = EPOLLIN;
if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, ev) == -1)
  errExit("epoll_ctl");
```

#### 事件等待：`epoll_wait()`

系统调用 `epoll_wait()` 返回 `epoll` 实例中处于就绪态的文件描述符信息。单个 `epoll_wait()` 调用能返回多个就绪态文件描述符的信息。

```C
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);

// Return number of ready file descriptor, 0 on timeout, or -1 on error
```

参数 `evlist` 所指向的结构体数组重返回的是有关就绪态文件描述符的信息。数组`evlist` 的空间由调用者负责申请，所包含的袁术个数在参数 `maxevents` 中指定。

在数组 `evlist` 中，每个元素返回的都是单个就绪态文件描述符的信息。`events` 字段返回了在该描述符上一届发生的事件掩码。`data` 字段返回的是我们在描述符下使用 `epoll_ctl` 注册感兴趣的事件时在 `ev.data` 中所指定的值。注意，`data` 字段是唯一一个可获知同这个事件相关的文件描述符号的途径。因此，当我面调用 `epoll_ctl()` 将文件描述符添加到兴趣列表中时，应该要么将 `ev.data.fd` 设为文件描述符号，要么将 `ev.data.ptr` 设为指定包含文件描述符号的结构体。

参数 `timeout` 用来确定 `epoll_wait()` 的阻塞行为，有如下几种:

-  -1: 一直阻塞
-  0: 执行一次非阻塞式检查
-  \>0: 调用将阻塞至多 timeout 毫秒，直到文件描述符上有事件发送，或者直到捕捉到一个信号为止

##### `epoll` 事件

当我们调用 `epoll_ctl()` 时可以在 `ev.events` 中指定的位掩码以及由 `epoll_wait()` 返回的 `evlist[].events` 中的值在下表给出：

**`epoll` 中 `events` 字段的位掩码值**

| 位掩码          | 作为 `epoll_ctl()` 的输入？ | 作为 `epoll_wait()` 返回 | 描述            |
| ------------ | :-------------------: | :------------------: | ------------- |
| EPOLLIN      |           √           |          √           | 可读取非高优先级的数据   |
| EPOLLPRI     |           √           |          √           | 可读取高优先级的数据    |
| EPOLLRDHUP   |           √           |          √           | 套接字对端关闭       |
| EPOLLOUT     |           √           |          √           | 普通数据可写        |
| EPOLLET      |           √           |                      | 采用边缘触发事件通知    |
| EPOLLONESHOT |           √           |                      | 在完成事件通知之后禁用检查 |
| EPOLLERR     |                       |          √           | 有错误发生         |
| EPOLLHUP     |                       |          √           | 出现挂断          |

##### `EPOLLONESHOT` 标志

默认情况下，一旦通过 `epoll_ctl()` 的 `EPOLL_CTL_ADD` 操作将文件描述符添加到 `epoll` 实例的兴趣列表中后，它会保持激活状态（即，之后对 `epoll_wait()` 的调用会在描述符处于就绪态时通知我们） 直到我们显示地通知 `epoll_ctl()` 的 `EPOLL_CTL_DEL` 操作将其从列表中移除。如果我们希望在某个特定的文件描述符上只得到**一次通知**，那么可以在传给 `epoll_ctl()` 的 `ev.events` 中指定 `EPOLLONESHOT` 标志。如果指定了这个标志，那么在下一个 `epoll_wait()` 调用通知我们对应的文件描述符处于就绪态之后，这个描述符就会在兴趣列表中被标记为非激活态，之后的 `epoll_wait()` 调用都不会再通知我们有关这个描述符的状态了。如果需要，我们可以稍后通过 `epoll_ctl()` 的 `EPOLL_CTL_MOD` 操作重新激活对这个文件描述符的检查。

#### 示例程序

```C
#include <sys/epoll.h>
#include <fcntl.h>
#include "tlpi_hdr.h"

#define MAX_BUF      1000         /* Maximum bytes fetched by a single read() */
#define MAX_EVENTS   5            /* Maximum number of events to be returned from a signle epoll_wait() call */


int
testEpoll(int argc, char *argv[])
{
    int epfd, ready, fd, s, j, numOpenFds;
    struct epoll_event ev;
    struct epoll_event evlist[MAX_EVENTS];
    char buf[MAX_BUF];

    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file ...\n", argv[0]);

    // 创建一个 epoll 实例
    epfd = epoll_create(argc - 1);
    if (epfd == -1)
        errExit("epoll_create");

    // 打开由命令行参数指定的每个文件，以此作为输入
    for (int j = 1; j < argc; j++) {
        fd = open(argv[j], O_RDONLY);
        if (fd == -1)
            errExit("open");
        printf("Opened \"%s\" on fd %d\n", argv[j], fd);

        ev.events = EPOLLIN;           /* Only interested in input events */
        ev.data.fd = fd;
        // 将得到的文件描述符添加到 epoll 实例的兴趣列表中
        if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1)
            errExit("epoll_ctl");
    }

    numOpenFds = argc - 1;

    // 执行一个循环
    while (numOpenFds > 0) {
        /* Fetch up to MAX_EVENTS items from the ready list */

        printf("About to epoll_wait()\n");
        // 循环中调用 epoll_wait() 来检查 epoll 实例的兴趣列表中的文件描述符，
        // 并处理每个调用返回的事件
        ready = epoll_wait(epfd, evlist, MAX_EVENTS, -1);
        if (ready == -1) {
            // 被信号打断处理
            if (errno == EINTR)
                continue;
            else
                errExit("epoll_wait");
        }
        printf("Ready: %d\n", ready);

        /* Deal with returned list of events */

        // 如果 epoll_wait() 调用成功，程序就再执行一个内层循环检查 evlist
        // 中每个已就绪的元素。
        for (int j = 0; j < ready; j++) {
            printf(" fd=%d; events: %s%s%s\n", evlist[j].data.fd,
                    // 对于 evlist 中的每个元素，程序不只是检查 events 字段中的 EPOLLIN 标记
                   (evlist[j].events & EPOLLIN) ? "EPOLLIN" : "",
                   // EPOLLHUP、EPOLLERR 也要检查
                   (evlist[j].events & EPOLLHUP) ? "EPOLLHUP" : "",
                   (evlist[j].events & EPOLLERR) ? "EPOLLERR" : "");
            if (evlist[j].events & EPOLLIN) {
                s = read(evlist[j].data.fd, buf, MAX_BUF);
                if (s == -1)
                    errExit("read");
                printf("    read %d bytes: %.*s\n", s, s, buf);
            } else if (evlist[j].events & (EPOLLHUP | EPOLLERR)) {
                /* If EPOLLIN and EPOLLHUP were both set, then there might
                 * be more than MAX_BUF bytes to read. Therefore, we close
                 * the file descriptor only if EPOLLIN was not set.
                 * We'll read further bytes after the next epoll_wait(). */
                // 当所有打开的文件描述符都关闭后，循环终止
                printf("    closing fd %d\n", evlist[j].data.fd);
                if (close(evlist[j].data.fd) == -1)
                    errExit("close");
                numOpenFds--;
            }
        }
    }
    printf("All file descriptors closed; bye\n");
    exit(EXIT_SUCCESS);
}

```

#### 深入探究 `epoll` 的语义

当我面通过 `epoll_create()` 创建一个 `epoll` 实例时，内核在内存中创建了一个新的 `i-node` 并打开文件描述（文件描述符表示的是一个打开文件的上下文信息（大小、内容、编码等与文件有关的信息），这部分实际是由内核来维护的。），随后在调用进程中为打开的这个文件描述分配一个新的文件描述符。同 `epoll` 实例的兴趣列表相关联的是打开的文件描述，而不是 `epoll` 文件描述符。这将产生下列结果：

-  如果我们使用 `dup()` （或类似的函数）复制一个 `epoll` 文件描述符，那么被复制的描述符所指代的 `epoll` 兴趣列表和就绪列表同原始的 `epoll` 文件描述符相同。若要修改兴趣列表，在 `epoll_ctl()` 的参数 `epfd` 上设定文件描述符可以是原始的也可以是复制的。
-  上条观点同意也适用于 `fork()` 调用之后的情况。此时子进程通过继承复制了父进程的 `epoll` 文件描述符，而这个复制的文件描述符所指向的 `epoll` 数据结果同原始的描述符相同。

当我们执行 `epoll_ctl()` 的 `EPOLL_CTL_ADD` 操作时，内核在 `epoll` 兴趣列表中添加了一个元素，这个元素同时记录了需要检查的文件描述符数量以及对应的打开文件描述的引用。 `epoll_wait()` 调用的目的就是让内核负责监视打开的文件描述符。这表示我们必须对之前的观点做改进：如果一个文件描述符是 `epoll` 兴趣列表中的成员，当关闭它后会自动从列表中删除。改进版应该是这样的：一旦所有指向打开文件描述的文件描述符都被关闭后，这个打开的文件描述符将从`epoll` 的兴趣列表中移除。这表示如果我们通过 `dup()` （或类似的函数）或者 `fork()` 为打开的文件创建了描述符副本，那么这个打开的文件只会在原始的描述符以及所有其他的副本都被关闭时才会移除。

#### `epoll` 同 I/O 多路复用的性能对比

**`poll()`、`select()` 以及 `epoll` 进行 100000 次监视操作所花费的时间**

| 被监视的文件描述符数量（N） | `poll()` 所占用的 CPU 时间（秒） | `select()` 所占用的 CPU 时间（秒） | `epoll`  所占用的 CPU 时间（秒） |
| :------------: | :---------------------: | :-----------------------: | :---------------------: |
|       10       |          0.61           |           0.73            |          0.41           |
|      100       |           2.9           |            3.0            |          0.42           |
|      1000      |           35            |            35             |          0.53           |
|     10000      |           990           |            930            |          0.66           |

为什么：

-  每次调用 `select()` 和 `poll()` 时，内核必须检查所有在调用中指定的文件描述符。与之相反，当通过 `epoll_ctl()` 指定了需要监视的文件描述符时，内核会在与打开的文件描述上下文相关联的列表中记录该描述符。之后每当执行 I/O 操作使得文件描述符成为就绪态时，内核就在 `epoll` 描述符的就绪列表中添加一个元素。（单个打开的文件描述上下文中一次 I/O 事件可能导致与之相关的多个文件描述符成为就绪态。）之后的 `epoll_wait()` 调用从就绪列表中简单地取出这些元素。
-  每次调用 `select()` 或 `poll()` 时，我传递一个标记了所有待监视的文件描述符的数据结构给内核，调用返回时，内核将所有标记为就绪态的文件描述符的数据结构再传回给我们。与之相反，在 `epoll` 中我们使用 `epoll_ctl()` 在内核控件中建立一个数据结构，该数据结构会将待监视的文件描述符都记录下来。一旦这个数据结构建立完成，稍后每次调用 `epoll_wait()` 时就不需要再传递任何与文件描述符有关的信息给内核了，而调用返回的信息中只包含那些已经处于就绪态的描述符。

#### 边缘触发通知

`epoll` API 还能以边缘触发方式进行通知——也就是说，会告诉我们自从上一次调用 `epoll_wait()` 以来文件描述符上是否已经有 I/O 活动了（或者由于描述符被打开了，如果之前没有调用的话）。使用 `epoll` 的边缘触发通知在语义上类似于信号驱动 I/O ，只是如果有多个 I/O 事件发生的话，`epoll` 会将它们合并成一次单独的通知，通过 `epoll_wait()` 返回，而再信号驱动 I/O 中则可能会产生多个信号。

采用 `epoll` 的边缘触发通知机制的程序基本框架如下：

1. 让所有监视的文件描述符都称为非阻塞的。
2. 通过 `epoll_ctl()` 构建 `epoll` 的兴趣列表。
3. 通过如下的循环处理 I/O 事件：
   1. 通过 `epoll_wait()` 取得处于就绪态的描述符列表
   2. 针对每一个处于就绪态的文件描述符，不断进行 I/O 处理知道相关的系统调用（例如 `read(), write(), recv(), send()` 或 `accept()`）返回 `EAGAIN` 或 `EWOULDBLOCK` 错误。

#### 当采用边缘触发通知时避免出现文件描述符饥饿现象

假设其中一个就绪态文件描述符又大量输入，如果使用非阻塞式读操作将所有输入都读取，那么此时就会有使其他的文件描述符处于饥饿状态的风险存在（即，在我们再次检查这些文件描述符是否处于就绪态并执行 I/O 操作前会有很长的一段处理时间）。该问题的一个解决方案是让应用程序维护一个列表，列表中存放着已经被通知为就绪态的文件描述符。通过一个循环按照如下方式不断处理。

1. 调用 `epoll_wait()` 监视文件描述符，并将处于就绪态的描述符添加到应用程序维护的列表中。如果这个文件描述符已经注册到应用程序维护的列表中了，那么这次监视操作的超时时间应该设为较小的值或者是0。这样如果没有新的文件描述符成为就绪态，应用程序就可以迅速进行到下一步，去处理那些已经处于就绪态的文件描述符了。
2. 在应用程序维护的列表中，只在那些已经注册为就绪态的文件描述符上进行一定限度的 I/O 操作（可能是以轮转调度（round-robin）方式循环处理，而不是每次 `epoll_wait()` 调用后从列表头开始处理 ）。当相关的非阻塞 I/O 系统调用出现 `EAGAIN` 或 `EWOULDBLOCK` 错误时，文件描述符就可以在应用程序维护的列表中移除了。

### 参考资料
《Linux/Unix 系统编程手册》
