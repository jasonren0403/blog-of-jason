---
title: 借助 ANTLR 分析代码中的漏洞 date: 2019-06-09 14:38:46 tags:

- "ANTLR"
- "编译原理"
  categories:
- ["实验","编译原理"]
- ["漏洞","双重释放"]

---

## 实验目的

通过分析特定漏洞代码特点，借助 ANTLR 分析代码中的特定漏洞。

<!-- more -->

## 实验条件

1. Windows 10 专业版
2. JDK 1.8版本（64位，版本号为1.8.0_202）
3. ANTLR环境——ANTLR 4

## 实验内容

### 漏洞描述

* Double Free，多次释放漏洞，是对指向同一个地址的指针进行两次及以上的操作，可能会造成任意代码执行或其他非预期危害。虽然一般把它叫做double free。其实只要是`free`一个指向堆内存的指针都有可能产生可以利用的漏洞。

* 一段简单的漏洞代码示例如下所示，其中对`char * buf2R1`所指向的内存进行了两次释放操作 {% codeblock lang:c %}

# include <stdio.h>

# include <unistd.h>

# define BUFSIZE1 512

# define BUFSIZE2 ((BUFSIZE1/2) - 8) //248

int main(int argc, char **argv) { char *buf1R1; char *buf2R1; char *buf1R2; buf1R1 = (char *) malloc(BUFSIZE2); buf2R1
= (char *) malloc(BUFSIZE2); free(buf1R1); free(buf2R1); buf1R2 = (char *) malloc(BUFSIZE1); strncpy(buf1R2, argv[1],
BUFSIZE1-1); free(buf2R1); free(buf1R2); } {% endcodeblock %}

* double free的原理其实和堆溢出的原理差不多，都是通过unlink这个双向链表删除的宏来利用的。只是double free需要由自己来伪造整个chunk并且欺骗操作系统。

### 语法识别

{% note info %} 利用ANTLR4的C++语言语法规则文件，对漏洞代码进行分析。 {% endnote %}

1. 右键点击g4中的translationunit，选择test rule，将那个有漏洞的C代码制作成expr文件后，作为输入。

2. 然后在输出位置就可以看到好大一棵parseTree了！右键另存为一张图。

{% asset_img parseTree-full.png 500 500 %}

3. ParseTree全貌，其中左边的一小撮是函数头定义，右边的一大撮才是函数体定义。
    * 首先定义了三个字符指针型变量`buf1R1`、`buf2R1`、`buf1R2`
      {% asset_img 1.png %}
    * 然后为`buf1R1`分配了`BUFSIZE2`大小的空间，为`buf2R1`分配了`BUFSIZE2`大小的空间 {% asset_img 3.png %} {% asset_img 4.png %}

4. 紧接着`free`掉它们 {% asset_img 5.png %}

5. 为`buf1R2`分配空间，大小为`bufsize1`
   {% asset_img 6.png %}

6. 把`argv[1]`中的内容复制到`buf1R2`所指空间中 {% asset_img 7.png %} {% asset_img 8.png %}

7. 最后的两个`free`语句都在语法分析树的右侧 {% asset_img 9.png %}

{% note primary %} 从语法分析树中看不出任何问题，因为这个存在漏洞的代码是语法正确的，而且也符合C语言语法结构。所以我们在编写相关代码时，要自行检查指针的使用情况，最好在释放指针所指向的空间后，将其置为<code>
NULL</code>。 {% endnote %}

## 实验总结

本实验中通过学习带有漏洞的C语言代码，了解到一些常见的二进制相关漏洞，借助到C语言的语法规则文件，对漏洞代码进行分解研究，了解到了我们所写的程序通过语法树展开后的样子，并试着提取其规则，编写漏洞描述xml文件。

中途遇到了IntelliJ IDEA无法解析规则，找不到start rule的问题，后来通过在cpp14.g4文件中右击translationunit规则，点击test translationunit
rule找到了语法树的生成点，从而成功在ANTLR Output中看到了所生成的语法树。
