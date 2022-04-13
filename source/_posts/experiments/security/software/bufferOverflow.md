---
title: 基本漏洞利用 date: 2019-10-08 13:54:07 tags:

- "网络空间安全"
- "软件安全"
  categories:
- ["实验","软件安全"]
- ["漏洞","缓冲区溢出"]
- ["漏洞","二进制"]
  comments: true

---

## 目标

通过分析密码验证小程序，初步掌握缓冲区溢出漏洞的原理和应用
<!-- more -->

## 测试步骤与结果

实验的源码如下

```c password.c
#include <stdio.h>
#include <string>
#define PASSWORD "1234567"
int verify_password (char *password)
{
	int authenticated;
	authenticated=strcmp(password,PASSWORD);
	return authenticated;
}

int main()
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
	return 0;
}
```

1. 简单阅读代码，发现程序检查了一个用户的输入，与内部存储的一个字符串进行比较，若相同，就返回验证成功。否则，显示验证失败并继续验证。 {% asset_img 0.png %}
2. 打开ollydbg，运行程序，在关键调用处（401030，401080等地址）打上断点以便观察。 {% asset_img 1.png %} {% tabs path, 1 %}

<!-- tab Fail@exclamation-circle -->
{% asset_img 2.png %} {% asset_img 3.png %}
<!-- endtab -->
<!-- tab Success@check-circle -->
{% asset_img 4.png %} {% asset_img 5.png %}
<!-- endtab -->
{% endtabs %}

{% note default %} 这里，已经进入了密码验证函数中，传进来的参数`s1`与内部存储的`s2`（`"1234567"`）进行了`strcmp`比较，`strcmp`是一个C语言的比较函数，若两个字符串`str1`
和`str2`相等，则返回`0`，这里不相等，且`str1<str2`，所以`strcmp`返回值为`-1`，`authenticated`的值也为`-1`，非零，验证失败。这里也可以看到，在`4010E5`处有一个关键的`JE`
跳转，表示表达式为零即跳转，跳转不成功，就会执行打印`"incorrect password!"`。若输入了正确密码，则跳转成功，转而执行输出`"Congratulations"`分支。 {% endnote %}

3. 跳过验证的其中一个方法是修改关键部位的汇编语句。比如，把地址`004010E5`的`JE`改为`JNZ`，具体方法是把`opcode`从`74`改成`75`
   。这样修改的道理是取反逻辑，把原来的“输入正确即通过验证”改为“输入不正确即可通过验证”，由于未知密码的概率相当大，这样修改相当于跳过了原有的验证过程（当然，假如经过修改后，作了一个“错误输入”程序反而崩溃退出了……就证明内部定义的密码被你蒙中了）。除此以外，这样的方法是不会对程序运行流程图有大的改变的。
   {% asset_img 6.png %} {% asset_img 7.png %} {% asset_img 8.png "输错密码也能进" %}

4. 修改汇编语句后，程序运行流程图表示。调试成功后，可以在IDA中静态修改相应位置的opcode，然后patch回原来的程序中 {% asset_img 9.png %}

## 测试结论

本次实验掌握了ollydbg的基本使用技巧和动态调试的快捷键：<kbd>F7</kbd>，<kbd>F8</kbd>，<kbd>F9</kbd>
，同时也学会在调试中打断点，以便于观察栈运行情况。通过阅读一个模拟认证的代码，了解简单的认证流程，从安全角度试图对代码进行破解。

## 思考题

{% tabs thinking, 1 %}
<!-- tab 思路1 -->
通过阅读代码并分析得知，只需要让`004010F6`处的代码得以执行，验证就可以通过。除了修改判断逻辑，我们也可以直接切断控制流，让代码不会执行到认证失败流中，从而通过验证。

具体方法：将`0x004010e5`的`jz`改为`jmp`无条件跳转，然后抹掉(`nop`)`0x004010f4`的`jmp`语句，让验证成功语句得以输出。 {% grouppicture 2-2 %} {% asset_img
10.png %} {% asset_img 11.png %} {% endgrouppicture %}

根据修改之后的程序流图来看，这样改动直接孤立了验证失败流。用户的任何输入都可以一次成功通过验证流程，而不关`verify_password`的结果如何。 {% grouppicture 2-2 %} {% asset_img 12.png
%} {% asset_img 13.png %} {% endgrouppicture %}
<!-- endtab -->
<!-- tab 思路2-->
{% note warning %} 没有成功做出来，只能先把思路留在这里了 {% endnote %} 能不能用缓冲区溢出的方法来破解这个程序呢？

重新阅读代码，发现其中的`password`变量是一个`1024*2`字节长的`char`数组，是有限长度的。而查阅资料知，`scanf`是存在安全问题的。例如存在以下代码和运行结果：

{% tabs code, 1 %}
<!-- tab 源码@code -->
{% codeblock lang:c %}

# include <stdio.h>

int main(){ int i=5; char pass[6]; while(1){ scanf("%s",pass); printf("%d\n",i); } return 0; } {% endcodeblock %}
<!-- endtab -->
<!-- tab 运行结果@terminal -->
{% codeblock lang:bash %} 12345 5 123456 5 1234567 5 12345678 0 1234567890 12345 123456789 57 {% endcodeblock %}
<!-- endtab -->
{% endtabs %}

由于`scanf`没有检验结束节点，当数组被溢出到一定程度时，就会覆盖整型变量`i`，输出非预期值。我们可以借鉴这个思路，让输入的`password`强行将`valid_flag`置为`0`
，最好可以跳过检验密码函数，直接进入输出验证成功的函数中，以完成破解。

<!-- endtab -->
{% endtabs %}
