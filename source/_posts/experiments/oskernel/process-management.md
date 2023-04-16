---
title: 进程管理实验 
date: 2019-10-12 11:52:10 
tags: ["OS","进程管理"]
categories:
  - ["OS","进程管理"]
  - ["技术"]
  - ["实验","操作系统内核"]
comments: true

---

## 实验目的

本次实验主要进行了进程有关的实验，利用Linux下的系统API——`fork()`函数来在Linux下创建一个新的进程，运行一个小程序来观察进程的争用资源和互相通信的现象。

<!-- more -->

## 实验步骤

### 进程的软中断通信

{% note primary %}
#### 要求
使用系统调用`fork()`创建两个子进程，再用系统调用`signal()`让父进程捕捉键盘上来的中断信号（即按<kbd>Delete</kbd>键），当父进程接受到这两个软中断的其中某一个后，父进程用系统调用`kill()`向两个子进程分别发送整数值为16和17软中断信号，子进程获得对应软中断信号后，分别输出下列信息后终止：

```text
Child process 1 is killed by parent!!
Child process 2 is killed by parent!!
```

父进程调用`wait()`函数等待两个子进程终止后，输出以下信息后终止:

```text
Parent process is killed!!
```

{% endnote %}

{% tabs res1,1 %}
<!-- tab 源代码@code -->

```c process.c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdlib.h>
int wait_flag;
void stop();

int main(){
	int pid1,pid2;
	signal(3,stop);
	while((pid1=fork())<0);
	if(pid1>0){
		while((pid2=fork())<0);
		if(pid2>0){
			printf("----\npid1:%d\npid2:%d\n\n----",pid1,pid2);
			wait_flag=1;
			sleep(5);
			kill(pid1,16);
			kill(pid2,17);
			wait(0);
			wait(0);
			printf("\n Parent process is killed!\n");
			exit(0);
		}
		else{
			wait_flag = 1;
			signal(17,stop);
			printf("\n Child process 2 is killed by parent!\n");
			exit(0);
		}
	}else{
		wait_flag=1;
		signal(16,stop);
		printf("\n Child process 1 is killed by parent!\n");
		exit(0);
	}
}
void stop(){
	wait_flag=0;
}
```

<!-- endtab -->
<!-- tab 运行结果@check -->
{% asset_img 00.png %}
<!-- endtab -->
{% endtabs %}

### 进程的管道通信

{% note primary %}
#### 要求
使用系统调用`pipe()`建立一条管道线，两个子进程分别向管道写一句话:

```text
Child process 1 is sending a message!
Child process 2 is sending a message!
```

而父进程则从管道中读出来自于两个子进程的信息，显示在屏幕上。父进程先接收子进程P1发来的消息，然后再接收子进程P2发来的消息。
{% endnote %}

{% tabs res2,1 %}
<!-- tab 源代码@code -->

```c pipe.c
#include <unistd.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
int pid1,pid2;

int main(){
	int fd[2];
	char outpipe[100],inpipe[100];
	pipe(fd);
	while((pid1=fork())== -1);
	if(pid1==0){
		lockf(fd[1],1,0); //lock the pipe
		sprintf(outpipe,"\nChild process 1 is sending message!\n");
		write(fd[1],outpipe,50);
		sleep(5);         //wait for read process
		lockf(fd[1],0,0); //unlock the pipe
		exit(0);
	}
	else{
		printf("\npid1:%d\n",pid1);
		while((pid2=fork())== -1);
		if(pid2==0){
			lockf(fd[1],1,0);
			sprintf(outpipe,"\nChild process 2 is sending message!\n");
			write(fd[1],outpipe,50);
			sleep(5);
			lockf(fd[1],0,0);
			exit(0);
		}
		else{
			printf("\npid2:%d\n",pid2);
			wait(0);  //wait for child process 1
			read(fd[0],inpipe,50);
			printf("%s\n",inpipe);
			wait(0);  //wait for child process 2
			read(fd[0],inpipe,50);
			printf("%s\n",inpipe);
			exit(0);  //parent process terminated
		}
	}
}
```

<!-- endtab -->
<!-- tab 运行结果@check -->
{% asset_img 2.png %}
<!-- endtab -->
{% endtabs %}

## 实验关键里程碑数据与结果

### 进程的软中断通信

前面的运行结果中，虽然`pid1`总是小于`pid2`，但是两个进程结束的顺序总是不一样的。我们来把程序后台运行一下，观察程序中所有的`pid`值和行为。

{% asset_img 0.png %}
{% asset_img 1.png %}

我们运行了`ps -aux`命令来查看系统中的后台进程，在最下方找到了`process`进程，其父进程`pid`为4473，而两个子进程`pid`分别为4475和4476。

之所以两个进程结束的顺序不同，是因为`kill`发出的信号没有被及时处理或可能被封锁。虽然调用`kill`的顺序是先1后2，但是发送的都是软中断信号，它们都需要等待所有相关资源的释放。而释放的顺序是不确定的，由于存在资源争夺，信号接受的先后也是不确定的。

观察它们在后台表现的程序名发现，它们都是由中括号括起的，后面有一个`defunct`标志，同时又显示为"Z"状态，表明此进程为僵尸进程，直接`kill`其`pid`将不会奏效。需要对其父进程`pid`进行`kill`操作，才可以正确结束进程。

### 进程的管道通信

仍然将程序置于后台运行，观察后台`pid`和行为。

{% asset_img 3.png %} {% asset_img 4.png %}

以上后台输出表明，在管道通信时，实质上启动了三个`pipe`进程，其中一个是父进程，它开启了下面的两个子进程，并监听管道中子进程传达的消息。在这个过程中，进程均为"S"状态，为休眠状态。

{% note info %}
那为什么每次运行程序时，子进程发送信息的先后总是不同呢？这其中应该也存在资源的争用问题。虽然`pid1`的创建一定早于`pid2`，但是调用`lockf`的先后顺序是不一定的，如果子进程1率先执行到`lockf(fd[1],1,0)`，那么它将先阻塞管道，把信息传入管道中，供父进程读出并打印在终端中。子进程2此时即使创建成功，有`pid`号，但是因为管道被阻塞，无法进入管道，故阻塞在它的`lockf(fd[1],1,0)`语句中，等到子进程1释放管道时，才可进入管道。反之相似。
{% endnote %}

## 实验难点与收获

* 这次的实验步骤不是很多，主要难点在于编程和对进程资源争用的理解上。
* 掌握了Linux下进程管理相关的C语言API。
* 对于进程的并发执行有了更深层次的理解。

## 实验思考

子进程的结束和父进程的运行是一个异步过程，即父进程永远无法预测子进程到底什么时候结束。那么会不会因为父进程太忙来不及 `wait` 子进程，或者说不知道子进程什么时候结束，而丢失子进程结束时的状态信息呢？

查阅资料得知：“不会。因为Linux提供了一种机制可以保证，只要父进程想知道子进程结束时的状态信息，就可以得到。这种机制就是:当子进程走完了自己的生命周期后，它会执行`exit()`系统调用，内核释放该进程所有的资源，包括打开的文件，占用的内存等。但是仍然为其保留一定的信息(包括进程号、退出码、退出状态、运行时间等)，这些数据会一直保留到系统将它传递给它的父进程为止，直到父进程通过`wait` / `waitpid`来取时才释放。

也就是说，当一个进程死亡时，它并不是完全的消失了。进程终止，它不再运行，但是还有一些残留的数据等待父进程收回。当父进程 `fork()` 一个子进程后，它必须用 `wait()` (或者 `waitpid()`)等待子进程退出。正是这个 `wait()` 动作来让子进程的残留数据消失。”

根据这个资料，我想到了一个问题：假如进程通信代码中去掉父进程的`wait`调用，`fork`出来的两个子进程还能正确结束吗？

{% gp 2-2 %}
{% asset_img 5.png %}
{% asset_img 6.png %}
{% endgp %}

仍然正确地结束了两个进程。出现这样的现象可能是因为Linux系统中有一些特殊的机制来保证“僵尸进程”被正确地接管(具体来说，由init进程来接管，其pid为1)。

### Windows对应的API

{% tabs windowsapi,1 %}
<!-- tab <code>createProcess</code> -->
对应于Linux的`fork()`API

```c
BOOL CreateProcess(
    LPCTSTR lpApplicationName,        //指向一个NULL结尾的、用来指定可执行模块的字符串。
    LPTSTR lpCommandLine,        //指向一个以NULL结尾的字符串，该字符串指定要执行的命令行。
    LPSECURITY_ATTRIBUTES lpProcessAttributes。//指向一个SECURITY_ATTRIBUTES结构体，这个结构体决定是否返回的句柄可以被子进程继承。如果lpProcessAttributes参数为空（NULL），那么句柄不能被继承。
    LPSECURITY_ATTRIBUTES lpThreadAttributes,        //同lpProcessAttribute，不过这个参数决定的是线程是否被继承，通常置为NULL。
    BOOL bInheritHandles,       //指示新进程是否从调用进程处继承了句柄。
    DWORD dwCreationFlags,  //指定附加的、用来控制优先类和进程的创建的标志。
    LPVOID lpEnvironment,        //指向一个新进程的环境块。如果此参数为空，新进程使用调用进程的环境。
    LPCTSTR lpCurrentDirectory,        //指向一个以NULL结尾的字符串，这个字符串用来指定子进程的工作路径。
    LPSTARTUPINFO lpStartupInfo,        //指向一个用于决定新进程的主窗体如何显示的STARTUPINFO结构体。
    LPPROCESS_INFORMATION lpProcessInformation  //指向一个用来接收新进程的识别信息的PROCESS_INFORMATION结构体。
);
```

<!-- endtab -->
<!-- tab <code>WaitForSingleObject</code> -->
对应于Linux的`waitpid()`

```c
DWORD WaitForSingleObject(
HANDLE hHandle,   //线程的Handle
DWORDdwMilliseconds  // 相应的Timeout时间
);
```

<!-- endtab -->
<!-- tab <code>ExitProcess</code> -->
对应于`exit()`

```c
void ExitProcess(UINT uExitCode);
```

<!-- endtab -->
<!-- tab 信号机制 -->
Windows也使用`signal.h`并也有`signal`函数作为信号控制函数。

```c
void __cdecl *signal(int sig, int (*func)(int, int))
```

<!-- endtab -->
<!-- tab <code>TerminateProcess</code> -->
有关进程终止，Windows使用`TerminateProcess`函数

```c
BOOL TerminateProcess(
HANDLE hProcess,  //要终止(杀死)进程的句柄，需要有PROCESS_TERMINATE权限。
UINT uExitCode  //设置进程的退出值。可通过GetExitCodeProcess函数得到一个进程的退出值。
)
```

<!-- endtab -->
{% endtabs %}
