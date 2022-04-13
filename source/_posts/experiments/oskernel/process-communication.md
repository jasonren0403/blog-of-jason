---
title: 进程通信实验 date: 2019-11-24 20:18:43 tags: ["OS","进程通信"]
categories:

- ["技术"]
- ["实验","操作系统内核"]
  comments: true

---

## 实验目的

通过几则进程通信的例子，理解进程的同步机制，理解锁、信号量在同步编程的重要地位。
<!--more-->

## 步骤

1. 解压给定程序包得到代码文件 {% asset_img 1.png %}
2. 运行Makefile编译所有代码文件 {% asset_img 2.png %}
3. 依次运行每个目标文件，观察它的执行现象

## 实验关键里程碑数据与结果

### 文件共享实验(consumer & producer)

通过阅读源码得知，consumer和producer共用了一个`data.dat`文件，其中producer向其写入内容

```c
write(fd, DataString, strlen(DataString)); /* populate data file */
```

其中DataString是我们写入的文字内容。 consumer从中读出内容并写入到标准输出`stdout`中

```c
/* Read the bytes (they happen to be ASCII codes) one at a time. */
int c; /* buffer for read bytes */
while (read(fd, &c, 1) > 0)    /* 0 signals EOF */
write(STDOUT_FILENO, &c, 1); /* write one byte to the standard output */
```

为了增强理解，我将DataString修改为了“2017****** xxx(名字和学号已打码)”，`data.dat`中的原来内容做如下修改 {% asset_img 3.png %} 然后重新编译程序。

接下来，先运行consumer，以验证`data.dat`中的内容未修改。运行一次producer，我们的datastring被写入到`data.dat`
中，由于producer指定了开始指针为0（文件中的第一个字节），所以我们的字符串从第一个字节开始，修改了原来data.dat中的内容，再调用consumer，就会显示此时文件的内容已被我们修改。 {% asset_img 4.png %}

{% note info %} 看上去是简单的文件写入和读取操作，实际上还有一些额外的工作

{% tabs extra,1 %}
<!-- tab <code>producer.c</code> -->
首先设定读/写锁。

```c
  struct flock lock;
  lock.l_type = F_WRLCK;    /* read/write (exclusive versus shared) lock */
```

然后试图打开文件。

```c
if ((fd = open(FileName, O_RDWR | O_CREAT, 0666)) < 0)  /* -1 signals an error */
report_and_exit("open failed...");
```

并为它加锁（不阻塞）。

```c
if (fcntl(fd, F_SETLK, &lock) < 0) /** F_SETLK doesn't block, F_SETLKW does **/
report_and_exit("fcntl failed to get lock...");
```

在文件写入结束后，释放锁。

```c
lock.l_type = F_UNLCK;
if (fcntl(fd, F_SETLK, &lock) < 0)
report_and_exit("explicit unlocking failed...");
```

<!-- endtab -->
<!-- tab <code>consumer.c</code> -->
在`consumer.c`中，锁的类型仍然是读/写锁。

```c
struct flock lock;
lock.l_type = F_WRLCK;    /* read/write (exclusive) lock */
```

然后试图打开文件。

```c
if ((fd = open(FileName, O_RDONLY)) < 0)  /* -1 signals an error */
report_and_exit("open to read failed...");
```

文件被加写锁时，不能从中读出内容。

```c
/* If the file is write-locked, we can't continue. */
fcntl(fd, F_GETLK, &lock); /* sets lock.l_type to F_UNLCK if no write lock */
if (lock.l_type != F_UNLCK)
report_and_exit("file is still write locked...");
```

读出内容时，要加读锁。

```c
lock.l_type = F_RDLCK; /* prevents any writing during the reading */
if (fcntl(fd, F_SETLK, &lock) < 0)
report_and_exit("can't get a read-only lock...");
```

读出结束后，释放锁。

```c
/* Release the lock explicitly. */
lock.l_type = F_UNLCK;
if (fcntl(fd, F_SETLK, &lock) < 0)
report_and_exit("explicit unlocking failed...");
```

手动释放锁即使失败，`close(fd)`和`return 0`也可以释放掉进程。
<!-- endtab -->
{% endtabs %} {% endnote %}

### 内存共享实验(memwriter & memreader)

通过源码得知，memwriter和memreader共用了`/dev/shm`内的一块空间，由writer向其中写入内容，reader向外读出并写入`stdout`中。 {% asset_img 5.png %}

在memwriter启动的一瞬间，`/dev/shm/`下会多出一个叫`shMemEx`
的文件，写入的内容就暂存在那个文件里。读出时也从那个文件中读出内容。Memwriter和memreader的正确工作要依靠计数信号量（semaphore），具体来说：

1. memwriter中，`semptr`的初始值为0，如果`semptr`的值为-1，那么程序退出。

```c
/* semphore code to lock the shared mem */
sem_t* semptr = sem_open(SemaphoreName, /* name */
		   O_CREAT,       /* create the semaphore */
		   AccessPerms,   /* protection perms */
		   0);            /* initial value */
if (semptr == (void*) -1) report_and_exit("sem_open");
```

在`strcpy`将`contents`拷贝到内存后，`sem_post`增加信号量的值（+1），如果`semptr`的值还是-1，那么程序退出。

```c
/* increment the semaphore so that memreader can read */
if (sem_post(semptr) < 0) report_and_exit("sem_post");
```

2. memreader中，`semptr`的初始值还是0。当`semptr`不为0时，将`memcontents`的内容全部逐字读出。每次`sem_wait`都将使信号量的值-1。

```c
/* use semaphore as a mutex (lock) by waiting for writer to increment it */
if (!sem_wait(semptr)) { /* wait until semaphore != 0 */
    int i;
    for (i = 0; i < strlen(MemContents); i++)
      write(STDOUT_FILENO, memptr + i, 1); /* one byte at a time */
    sem_post(semptr);
}
```

### 管道通信实验(fifoWriter & fifoReader)

fifoWriter和fifoReader共享了一个管道文件`fifoChannel`，fifoWriter向其中写入若干随机数，fifoReader读文件并判断其中质数的多少。 {% asset_img 6.png %}
与实验3.2不同的是，当同时运行此实验的writer和reader时，一定是writer先结束后，Reader才会有结果显示。当事先没有writer的写入时，reader会直接退出（因为没有管道文件）。运行一瞬间产生的管道文件会在写入完成后消失（源代码中写完就`close`
了）。

写进程以只写模式打开管道文件。

```c
int fd = open(pipeName, O_CREAT | O_WRONLY); /* open as write-only */
```

读进程以只读模式打开管道文件。

```c
int fd = open(file, O_RDONLY);
```

从而分时占用了文件。

### 信号量互斥实验(shutdown)

运行结果如下 {% asset_img 7.png %} {% note info %}

#### 这是啥意思呢？

结合源码得知，主进程运行到`fork`后分开为两个进程，父进程调用`parent_code`并阻塞，调用`sleep(5)`睡眠5秒，等待子进程退出。(
这可以解释为什么main打印了一遍，而main…2和main…3被打印了两遍的原因——main…2和main…3都在fork调用之后)
然后，子进程从`fork`开始继续运行，进入了`child_code`，`child_code`中，注册了`SIGTERM`（id=15）为其处理信号。然后，尝试先调用`sleep(1)`，睡眠1秒，然后输出"Child just woke
up, but going back to sleep"。5秒过后主进程醒了，继续运行`parent_code`，其调用了`kill`
来杀死子进程，（所以上述信息输出了4次）子进程接收到信号后，打印出信号id并执行相应的处理函数。待子进程调用`_exit(0)`返回内核后，父进程也退出，整个程序运行结束。 {% endnote %}

### 队列通信实验(sender & receiver)

通过观察源码得知，此部分实验运用了消息队列函数`msgget`、`msgsnd`、`msgrcv`、`msgctl`，它们所需要的头文件为 `<sys/types.h>`、`<sys/ipc.h>`和`<sys/msg.h>`
。sender以队列形式发出了六条信息"msg1"、 "msg2"、 "msg3"、 "msg4"、 "msg5"、 "msg6"
，msg1和2是类型1的，3和4是类型2的，5和6是类型3的。Receiver以与发送方不同的类型顺序接收，具体来说，按照3→1→2→1→3→2的顺序，实际运行结果如下 {% asset_img 8.png %}
仍然可以正确地接收信息，而且队列内部的顺序是“先来先接收”的。比如msg1在type1中先于msg2发送，那么接收时也是msg1比msg2先接送。

当msg接收完成后，它就从队列中被移除，这也是为什么再次运行receiver后接受失败的原因。 {% asset_img 9.png %}

### 实现内核和用户程序之间的文件通信

{% tabs code,1 %}
<!-- tab server源码@code -->

```c server.c
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include "sock.h"

void report(const char* msg, int terminate) {
  perror(msg);
  if (terminate) exit(-1); /* failure */
}

int main() {
  int fd = socket(AF_INET,     /* network versus AF_LOCAL */
		  SOCK_STREAM, /* reliable, bidirectional: TCP */
		  0);          /* system picks underlying protocol */
  if (fd < 0) report("socket", 1); /* terminate */

  /* bind the server's local address in memory */
  struct sockaddr_in saddr;
  memset(&saddr, 0, sizeof(saddr));          /* clear the bytes */
  saddr.sin_family = AF_INET;                /* versus AF_LOCAL */
  saddr.sin_addr.s_addr = htonl(INADDR_ANY); /* host-to-network endian */
  saddr.sin_port = htons(PortNumber);        /* for listening */

  if (bind(fd, (struct sockaddr *) &saddr, sizeof(saddr)) < 0)
    report("bind", 1); /* terminate */

  /* listen to the socket */
  if (listen(fd, MaxConnects) < 0) /* listen for clients, up to MaxConnects */
    report("listen", 1); /* terminate */

  fprintf(stderr, "Listening on port %i for clients...\n", PortNumber);
  /* a server traditionally listens indefinitely */
  while (1) {
    struct sockaddr_in caddr; /* client address */
    int len = sizeof(caddr);  /* address length could change */

    int client_fd = accept(fd, (struct sockaddr*) &caddr, &len);  /* accept blocks */
    if (client_fd < 0) {
      report("accept", 0); /* don't terminated, though there's a problem */
      continue;
    }

    /* read from client */
    int i;
    for (i = 0; i < ConversationLen; i++) {
      char buffer[BuffSize + 1];
      memset(buffer, '\0', sizeof(buffer));
      int count = read(client_fd, buffer, sizeof(buffer));
      if (count > 0) {
	puts(buffer);
	write(client_fd, buffer, sizeof(buffer)); /* echo as confirmation */
      }
    }
    close(client_fd); /* break connection */
  }  /* while(1) */
  return 0;
}
```

<!-- endtab -->
<!-- tab client源码@code -->

```c client.c
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <netdb.h>
#include "sock.h"

const char* books[] = {"War and Peace",
		       "Pride and Prejudice",
		       "The Sound and the Fury"};

void report(const char* msg, int terminate) {
  perror(msg);
  if (terminate) exit(-1); /* failure */
}

int main() {
  /* fd for the socket */
  int sockfd = socket(AF_INET,      /* versus AF_LOCAL */
		      SOCK_STREAM,  /* reliable, bidirectional */
		      0);           /* system picks protocol (TCP) */
  if (sockfd < 0) report("socket", 1); /* terminate */

  /* get the address of the host */
  struct hostent* hptr = gethostbyname(Host); /* localhost: 127.0.0.1 */
  if (!hptr) report("gethostbyname", 1); /* is hptr NULL? */
  if (hptr->h_addrtype != AF_INET)       /* versus AF_LOCAL */
    report("bad address family", 1);

  /* connect to the server: configure server's address 1st */
  struct sockaddr_in saddr;
  memset(&saddr, 0, sizeof(saddr));
  saddr.sin_family = AF_INET;
  saddr.sin_addr.s_addr =
     ((struct in_addr*) hptr->h_addr_list[0])->s_addr;
  saddr.sin_port = htons(PortNumber); /* port number in big-endian */

  if (connect(sockfd, (struct sockaddr*) &saddr, sizeof(saddr)) < 0)
    report("connect", 1);

  /* Write some stuff and read the echoes. */
  puts("Connect to server, about to write some stuff...");
  int i;
  for (i = 0; i < ConversationLen; i++) {
    if (write(sockfd, books[i], strlen(books[i])) > 0) {
      /* get confirmation echoed from server and print */
      char buffer[BuffSize + 1];
      memset(buffer, '\0', sizeof(buffer));
      if (read(sockfd, buffer, sizeof(buffer)) > 0) puts(buffer);
    }
  }
  puts("Client done, about to exit...");
  close(sockfd); /* close the connection */
  return 0;
}
```

<!-- endtab -->
{% endtabs %} 主要利用了C的`socket`通信函数来实现文件通信。先运行server，再运行client，结果如下。 {% asset_img 10.png %}
Server开启了9876号端口的监听，接收client端向服务器写入信息，server端会向client端回应所写入的内容，client端写入完成，就会退出。

此时查看`netstat`状态，会发现server端的9876号端口在监听任意地址。 {% asset_img 11.png %}

## 实验难点与收获

本次实验主要为给定文件，运行可执行程序并对执行结果进行解释。通过运行给定的示例程序，我对于进程的同步有了更深层的认识，明白了信号量和锁机制的使用对于同步编程的正确性来说是个很重要的保证。同时，也学习到了一些Linux系统头文件中自带的一些同步函数。

## 实验思考

Server和client的socket通信过程是否也是文件通信的一种形式？文件描述符是由socket函数获得的，我们能不能在linux系统中找到相应的文件呢？
