---
title: 逆向分析—SMC
date: 2019-05-30 12:41:20
tags:
  - "网络空间安全"
  - "逆向分析"
categories:
  - ["实验","网络安全"]
---

了解SMC代码自修改程序在逆向分析中的表示，并体验其分析过程。
<!-- more -->
## 初识程序
{% asset_img 1.png %}

打开所给程序，发现里面什么都没有，在里面试着输入一些内容后，发现它返回`try again`，看来只有想办法看到源代码了。

把这个程序拖入IDA，发现可以找到`main`函数，且结构并不是很复杂，所以现在可以着手开始看了~

{% gp 2-1 %}
{% asset_img 2.png %}
{% asset_img 3.png %}
{% endgp %}

## 逐层分析并解密
### 第一层加密
{% asset_img 4.png %}

从结构图来看，程序整体是分支-循环结构，且观察下边的分支可以大概分析出，我们的输入多半走了最左端的红线，输出了一个`Try again`（但是因为没有`system("pause")`而直接退出了程序）。而想要继续分析下去，找到flag，右边的循环部分是重点。来，看看循环吧。

输入的内容长度必须为`28`（`1Ch`），才能进入到右边的绿线。

当最后一个字符为7Dh（`}`右花括号）时，进入右侧的循环。右侧的循环一共进行了67次（`43h`），每次都对`byte_414C3C[i]`中的内容异或了一个常数`0x7D`。67次循环结束后，程序call了一下`[ebp-6Ch]`中的内容，结合刚才的过程，`[ebp-6Ch]`中的东西应该是一个函数，但是现在我们还不知道它是什么，我们需要写一个脚本来把隐藏的函数弄出来。

{% asset_img 5.png %}

Hex-Rays 为快速修改二进制码提供了一个API接口，其Python接口称为IDAPython，文档为 https://www.hex-rays.com/products/ida/support/idapython_docs/，我们只需要其中的 `patch_bytes(address, buf)` 即可。

{% note info %}
#### IDA-Python 常用的 API
可以看到，idaapi.py提供了这些常用函数，对于本次实验来说已经足够。
```python
get_bytes(address,count) # 从address处读取count个字节的内容
patch_bytes(address,buf) # 将address地址处patch为buf的内容
Xrefsto(address,flags=0) # 找到所有引用了address的地址
byte(address) # 获取address地址的一个字节的内容
```
{% endnote %}

{% codeblock part_1 lang:python %}
def xorize(start_loc, end_loc, num):
  for addr in range(start_loc, end_loc):
    patch_bytes(addr, get_bytes(addr) ^ num))
    addr+=1
xorize(0x00414c3c, 0x00414c7f, 0x7d) # for part 1
{% endcodeblock %}

可以观察到，执行脚本后，`byte_414C3C`中的内容发生了改变，我们紧接着将其转换为汇编代码（按<kbd>C</kbd>键），可以看到`.data`字段变红，然后，右键起始地址，选择create function，将其反编译为函数，

{% gp 2-2 %}
{% asset_img 6.png %}
{% asset_img 7.png %}
{% endgp %}

{% asset_img 8.png %}
现在这样就非常容易分析了，我们输入字符的前五位一定是`flag{`，加上结尾的`}`，已经解决了28个字符的6个，剩下的还需慢慢来。

### 第二层加密
{% asset_img 9.png %}
们留意到，下方有一个`do-while`循环，一共循环了90次，它的作用是将`a2`中的内容与`0x43`异或，想要解密，我们异或回去即可。留意到前面`main`函数调用的方式，可知这次利用到了`unk_414BE0`中的内容。在这一步结束后，对剩余的部分调用`a2`函数，我们现在不知道它是什么，必须先把它解出来。

继续写payload，加载payload。来到第三层加密。
{% gp 2-2 %}
{% asset_img 10.png %}
{% asset_img 11.png %}
{% endgp %}

{% codeblock part_2 lang:python %}
xorize(0x00414be0, 0x00414c3a, 0x43) # for part 2, see previous for xorize() definition
{% endcodeblock %}
### 第三层加密
{% asset_img 12.png %}
终于来到了第三层函数，留意一下`sub_414C3C`中的`a2(a1 + 5, &unk_414A84);`。`unk_414A84`其中的内容需要经过347次循环，每一次将一位内容与`0x55`异或，步骤类似前面。

{% codeblock pre_part_3 lang:python %}
xorize(0x00414a84, 0x00414bdf, 0x55) # for part 3, see previous for xorize() definition
{% endcodeblock %}

经过以上Payload得到以下这样：

{% gp 2-2 %}
{% asset_img 13.png %}
{% asset_img 14.png %}
{% endgp %}

{% asset_img 15.png %}
注意到这里还有一层异或加密，先解开再说。由`414be0`中的`a2(a1 + 4, (const char *)&unk_414A30);`得知，是`414a30`中的东西被调用了，所以要把它解开，逐位异或`0x4d` 83次即可

{% codeblock part_3_smc lang:python %}
xorize(0x00414a30, 0x00414a84, 0x4d) # for part 3, see previous for xorize()
{% endcodeblock %}

{% gp 2-2 %}
{% asset_img 16.png %}
{% asset_img 17.png %}
{% endgp %}

（截图中v2的值不太对，实际上是-1）

第三层的主要作用则是判断接下来的4个字符与 `0xCC`异或后的结果要与一串硬编码的值相同（注意到`v5 = 0x93A9A498`）。经过运算该值为`The_`

第三层的解密脚本如下（注意逆序）
{% codeblock part_3 lang:python %}
def part_3():
  src=[0x98,0xa4,0xa9,0x93]
  res=''
  for num in src:
    num^=0xcc
    res+=chr(num)
  print(res)
{% endcodeblock %}

### 第四层加密
经过提示，`sub_414A84`中进行的是base64运算，之后的几个字符串加密应该得到`cmVhbEN0Rl8=`，随便找一个在线base64解密网站得到该字符串为`realCtF_`

### 第五层加密
第五层，所得字符每一个加一，得到字符串 kvtu\`C4h"o(s[5]=\`)，经脚本计算后得到字符串为 `just_B3g!n`

第五层的解密代码如下
{% codeblock last_part lang:python %}
def last_part():
  str='''kvtu`C4h"o'''
  ori_str=list(str)
  new_str=[chr(ord(i)-1) for i in ori_str]
  print(''.join(new_str))
{% endcodeblock %}

所有的payload代码如下所示
{% codeblock all_payload.py lang:python %}
from idaapi import *

def xorize(start_loc,end_loc,num):
  for addr in range(start_loc,end_loc):
    patch_byte(addr,get_byte(addr)^num)
    addr+=1

def last_part():
  str='''kvtu`C4h"o'''
  ori_str=list(str)
  new_str=[chr(ord(i)-1) for i in ori_str]
  print(''.join(new_str))

if __name__=="__main__":
  # may run separately
  xorize(0x00414c3c,0x00414c7f,0x7d) # part 1
  xorize(0x00414be0,0x00414c3a,0x43) # part 2
  xorize(0x00414a84,0x00414bdf,0x55) # part 3
  xorize(0x00414a30,0x00414a84,0x4d) # part 4
  last_part() # last_part
{% endcodeblock %}

## 快结束了！
所有的东西拼在一起为`flag{The_realCtF_just_B3g!n}`，此即为最终的flag目标。