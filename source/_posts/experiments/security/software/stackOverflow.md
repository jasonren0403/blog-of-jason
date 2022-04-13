---
title: 栈溢出 date: 2019-10-26 17:12:22 tags:

- "网络空间安全"
- "软件安全"
  categories:
- ["实验","软件安全"]
- ["漏洞","栈溢出"]
  comments: true

---

## 目标

学会使用淹没相邻变量或返回地址的方法利用缓冲区溢出漏洞。
<!-- more -->

## 测试步骤与结果

本次实验的源码如下所示

```c stackvar.c
#include <stdio.h>
#include <string>
#define PASSWORD "1234567"

int verify_password (char *password)
{
	int authenticated;
	char buffer[8];// add local buff
	authenticated=strcmp(password,PASSWORD);
	strcpy(buffer,password);//overflowed here!
	return authenticated;
}

main()
{
	int valid_flag=0;
	char password[1024];
	while(1)
	{
		printf("please input password:       ");
		scanf("%s",password);
		valid_flag = verify_password(password);
		if(valid_flag)
		{
			printf("incorrect password!\n\n");
		}
		else
		{
			printf("Congratulation! You have passed the verification!\n");
			break;
		}
	}
}
```

### 验证缓冲区溢出的发生

1. 正常情况下，输入正确密码，程序提示“输入正确”；输入错误密码（少于8位），程序提示“输入错误”。 {% asset_img 1.png %}
2. 进入ollydbg，在函数调用处打断点。开始运行程序，输入一个较短的密码，就以`"444"`为例。 {% asset_img 2.png %}
3. `strcpy`的调用点在`0x401055`处。 {% asset_img 3.png %}
4. 输入的密码存储在`0x12fb7c`处，要拷贝到`0x12fb18`处。 {% grouppicture 2-2 %} {% asset_img 4.png "from" %} {% asset_img 5.png "to" %}
   {% endgrouppicture %}
5. `0x12fb18`处的数据如下，显示为`00 34 34 34`，恰好是`"4"`的ascii码十六进制表示 {% asset_img 5补.png %}

### 淹没相邻变量改变程序流程

{% tabs overflow2, 1 %}
<!-- tab 输入qqqqqqqqrst -->
{% asset_img 6.png %} 拷贝前后，内存的变化如下 {% grouppicture 2-2 %} {% asset_img 7.png "前" %} {% asset_img 8.png "后" %} {%
endgrouppicture %} {% note info no-icon %} 已经观察到`0x12fb20`位置的变量被覆盖了 {% asset_img 9.png %} {% endnote %}
<!-- endtab -->
<!-- tab 输入qqqqqqqq -->
{% asset_img 10.png %} 拷贝前后，内存变化如下 {% grouppicture 2-2 %} {% asset_img 11.png "前" %} {% asset_img 12.png "后" %} {%
endgrouppicture %} {% asset_img 13.png %} {% note info no-icon %}
`0x12fb20`的值被覆盖成了`0x00000000`。导致函数返回值为零，认证通过。 {% asset_img 14.png %} {% endnote %}
<!-- endtab -->
<!-- tab 输入01234567 -->
{% asset_img 15.png %} {% grouppicture 2-2 %} {% asset_img 16.png %} {% asset_img 17.png %} {% endgrouppicture %} {%
asset_img 18.png %} {% note info no-icon %} 虽然也淹没了`0x12fb20`处的`authenticated`变量，但是我们输入的密码小于`1234567`，`strcmp`会返回`-1`
，`-1`是用补码表示的，末尾的`\0`只可以淹没`-1`补码的后两位，程序不会向我们预想的方向走去。 {% asset_img 19.png %} {% endnote %}
<!-- endtab -->
{% endtabs %}

### 淹没返回地址改变程序流程

{% note default %} 为了方便调试，我们使用文件来输入“密码”。 {% codeblock lang:c %} if(!fp=fopen("password.txt","rw+")){ exit(0); } fscanf(
fp,"%s",password); valid_flag = verify_password(password); {% endcodeblock %} {% endnote %}

1. 找到输入点。这个地址跳转到“验证成功”的输出上。 {% asset_img 20.png %}
2. 先填满8字节的`buffer`数组，4字节的`authenticated`变量，4字节的ebp，接下来的4个字节我们就可以放上我们的返回地址。 {% asset_img 21.png %}
3. 运行程序，显示“验证成功”，但由于直接跳转地址破坏了栈平衡，程序崩溃。 {% asset_img 22.png %} 此时的栈结构如此图所示 {% asset_img 23.png %}

## 测试结论

覆盖的过程中发生了什么？我认为可以用以下几张图来表示 {% tabs res, 1 %}
<!-- tab 输入qqqqqqqqrst时 -->
{% asset_img 覆盖1.png %}
<!-- endtab -->
<!-- tab 输入qqqqqqqq时 -->
{% asset_img 覆盖2.png %}
<!-- endtab -->
<!-- tab 输入01234567时 -->
{% asset_img 覆盖3.png %}
<!-- endtab -->
<!-- tab 覆盖返回地址时 -->
{% asset_img 覆盖4.png %}
<!-- endtab -->
{% endtabs %} 两种溢出方法都是比较“危险”的，一旦被利用，会造成意想不到的后果。轻则引起程序崩溃，重则引发安全漏洞。

## 思考题

程序的源码如下

```c stackOverrun.c
/*
  StackOverrun.c
  This program shows an example of how a stack-based
  buffer overrun can be used to execute arbitrary code.  Its
  objective is to find an input string that executes the function bar().
*/
#include <stdio.h>
#include <string.h>

void foo(const char* input)
{
    char buf[10];

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

本次的目标是通过栈溢出，想办法使`bar`函数得到执行。

{% tabs solution, 1 %}
<!-- tab 解法一 -->

1. 程序通过命令行参数执行了`foo`函数，在`foo`函数中，有一个10个字节长的`buf`字符数组，在第14行中发生了未经检查的无界向有界拷贝的行为，很容易引发溢出。分析到这里，思路就很简单了：参数中先用任意字符填满`buf`
   ，然后再填入4字节的`bar`函数的地址。

2. 先调试，填上一个较短的参数看看发生了什么 {% grouppicture 2-2 %} {% asset_img 思考题1.png "填入参数3344" %} {% asset_img 思考题2.png "运行" %} {%
   endgrouppicture %}
3. 发现我们给的这个参数在DS段中存储着，而且运行时程序会打印出`foo`和`bar`函数的地址值。所以`bar`函数地址我们已经有了。
4. 我们的命令行参数从DS段拷贝到了`0x12ff60`的位置。 {% grouppicture 2-2 %} {% asset_img 思考题3.png %} {% asset_img 思考题4.png %} {%
   endgrouppicture %} {% grouppicture 2-2 %} {% asset_img 思考题5.png %} {% asset_img 思考题6.png %} {% endgrouppicture %}
5. 构造payload时要注意考虑到内存的对齐因素，缓冲区是10个字节长没错，但是填充时要淹没掉`0x40109b`的返回地址值，又要使最后4个字节为`bar`函数的地址值，所以要填上12个任意字符。综合考虑，构造payload如下。
   {% asset_img 思考题7.png %}
6. 把这个值复制到调试命令行中，重启程序 {% asset_img 思考题8.png %}
7. 此时已经可以看到`0x12ff6c`中存储的返回地址值变成了`0x401060`。前面的`0x12ff60~0x12ff68`也已悉数占满。 {% asset_img 思考题9.png %} {% asset_img
   思考题10.png %}
8. 按<kbd>F8</kbd>单步跟下来，可以发现程序紧接着跳到了`0x401060`，`bar`函数的地址，目标达到了。紧接着程序就因为栈不平衡而崩溃了。 {% grouppicture 2-2 %} {% asset_img
   思考题11.png %} {% asset_img 思考题12.png %} {% endgrouppicture %} {% asset_img 思考题最终.png "命令行运行结果" %}
9. 覆盖返回地址法利用全过程图示 {% asset_img 1571749477979.jpg "图中红笔为溢出执行过程" %}

<!-- endtab -->
<!-- tab 解法二 -->
接下来，我们使用`jmp esp`的方法来完成程序的破解。

1. 使用od自带的插件overflow return address寻找可用的`jmp esp`地址，查找结果如下 {% grouppicture 2-2 %} {% asset_img 思考题2-1.png %} {%
   asset_img 思考题2-2.png %} {% endgrouppicture %}
2. 本次实验选用了一个位于`ntdll.text`段的`jmp esp`，地址为`77f8948bh`。

根据`jmp esp`构造相关知识，构造payload为12个填充字符"a"+`jmp esp`地址`0x77f8948b`（大端书写）+shellcode（`call bar()`的机器码）。下面简单看一下运行该参数之后，栈的情况。 {%
asset_img 思考题2-3.png "payload图示" %}

3. `foo`和`bar`的地址输出等同于方法一 {% grouppicture 2-2 %} {% asset_img 思考题2-4.png %} {% asset_img 思考题2-5.png %} {%
   endgrouppicture %}

4. 将命令行的内容复制到缓冲区时，观察输出，可以发现`0x12ff6c`被覆盖成了`jmp esp`的地址 {% asset_img 思考题2-6.png "而临接地址的内容，就是本次我们要执行的shellcode" %}

5. 在函数`return`之前，注意到它即将返回到一个内核地址`77f8948b`
   {% asset_img 思考题2-7.png %}

6. 陷入内核的一瞬间，可以看到`eip`指针所指向的位置上，命令为`jmp esp`。观察右侧知`esp`现在的值为`12ff70h`，接下来，程序到`12ff70h`执行内容。 {% asset_img 思考题2-8.png %}

7. 跳转到`12ff70`地址后，其中所写入的数据都被当作指令来执行，我们的shellcode被成功解析为了`call`指令。 {% asset_img 思考题2-9.png %}

8. 果然它紧接着调用了`00401060`处的`bar`函数。 {% asset_img 思考题2-10.png %}

9. 调用完成后，它返回了`12ff70`处，然后继续向下执行指令，直到崩溃。 {% asset_img 思考题2-11.png %}

{% note warning %} 如果没有任何中断的话，接下来地址的所有的hex数据都会按照命令来执行，相当危险…… {% endnote %}

10. 刚才执行的过程中成功产生预期输出。 {% asset_img 思考题2-12.png %}

11. 刚才的payload可以正常在命令行中传参执行，成功触发`bar`函数。 {% asset_img 思考题方法二最终.png %}

12. `jmp esp`法利用全过程图示 {% asset_img 1572080943051.jpg "图中红笔为溢出执行过程" %}

<!-- endtab -->
{% endtabs %}
