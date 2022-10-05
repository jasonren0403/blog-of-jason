---
title: 逆向分析—仿射加密
date: 2019-04-24 23:32:39
mathjax: true
tags:
  - "网络空间安全"
  - "逆向分析"
categories:
  - ["实验","网络安全"]

---

通过一个简单的仿射加密程序逆向，学习仿射加密在逆向程序中的表示以及如何进行关键点的处理。熟悉了基本的逆向分析方法和流程。
<!-- more -->

## 初识程序
运行初见截图如下：
{% asset_img 1.png %}

在 IDA 中观察主程序流程大致如下图：
{% asset_img 2.png %}

这是一个32位exe程序，通过试运行，大致观察到程序接受了一个字符串，若满足相应条件，会输出 `ok,you really know`，否则，会输出 `sorry`。我们的目标，就是要让程序输出前者。

## 结构分析
用32位IDA打开这个程序。
{% asset_img 3.png %}

{% gp 2-2 %}
{% asset_img 4.png %}
{% asset_img 5.png %}
{% endgp %}

在`test ecx,ecx`后，通过不大于转移(jle)命令，判断是否跳转到`short loc_40105A`地址上，否则，进入`loc_401047`地址，进行循环，循环结束后仍然进入`loc_40105A`地址，继续进行比较。循环时有`inc eax`操作，说明这里的循环可能是`for`循环。

{% asset_img 6.png %}
继续顺序执行。在`test ecx,ecx`后发生了又一次不大于转移命令，判断是否需要直接跳转到`loc_401062`地址上，在`loc_401062`内部又发生了循环。当不满足小于转移条件时，出循环，继续向下执行。

{% asset_img 7.png %}
看来是要从内存中取字符串了，记下来，可能是分析的重点。

{% asset_img 8.png %}
一堆条件判断和跳转，根据前三条红色路线和绿色路线走，还有循环，略有些复杂。

{% asset_img 9.png %}
终于到结尾了！这一部分就是简单的条件分支结构了。由于整个程序有效代码部分只有一个`main`函数，所以整个程序的结构分析到此就结束了。

## 逻辑分析
首先从`main`函数开始，从`sub`开始数起。
{% asset_img 10.png %}
我们输入的字符串被存放在`esp+68h+var_64`中。然后经过`test`检验，当寄存器的值小于等于`0`时，跳转到`40105A`地址上。否则进入`401047`地址中进行循环。

{% asset_img 11.png %}
从`esp+eax+68h+var64`中，即用户输入的值中取值与`61h`、`7Ah`进行比较，大体意思即为判断用户输入满足在`"a"~"z"`之间。若不满足条件，跳至`short loc_4010B3`中，而它直接终止了整个函数的运行。

{% asset_img 12.png %}
关键的一步。`esp+esi+70h+var_64`中的内容被移入值`eax`中，该段内容经`eax+eax*2-0x11C=3*eax-0x11C`的结果被放入了`eax`中，之后`mod26`运算，得到的结果再加上`61h`（`a`的ascii码表示）构成密文并写入内存的`esp+esi+70h+var64`中。

{% asset_img 13.png %}
根据上述过程，有 `3*eax-0x11C=a*eax-a*0x61+b`，得到$a=3$，$b=7$。即仿射加密函数为 $y=3x+7(mod 26)$

{% asset_img 14.png %}
最后，加密后的内容（在`esp+70h+var_64`）与程序中的字符串`qxbxpluxvwhuzjct`比较，若成功就会得到`You really knows`了。

再用<kbd>F5</kbd>看下伪代码确认下分析是否正确。
{% gp 2-2 %}
{% asset_img 16.png %}
{% asset_img 15.png %}
{% endgp %}

该知道的都知道了，可以开始写payload了！

## Payload 编写
整理一下思路：
输入的内容经过仿射密钥加密，函数为 $y=3x+7(mod26)$，密文`e(x)`为`qxbxpluxvwhuzjct`
根据密码学的知识，解密密钥应为 $x=3^{-1}(e(x)-7)(mod26)=9(e(x)-7)(mod26)$

编写python解密程序如下

```python rev_payload.py
def rev_3():
  str = '''qxbxpluxvwhuzjct'''
  ori_str = list(str)
  # 解密x=9(y-7) ord('a')=97
  new_str = [chr((ord(i) - 97 - 7 + 26) * 9 % 26 + 97) for i in ori_str]
  print(''.join(new_str))
```

其输出结果为`doyouknownfangshe`。

将其作为命令行参数运行程序，结果为下图所示
{% asset_img 17.png %}
成功达成目标
