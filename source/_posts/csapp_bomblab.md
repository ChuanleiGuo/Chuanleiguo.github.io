---
title: 【深入理解计算机系统实验笔记】| Bomb Lab
date: 2017-4-17 21:00:23
categories: 
- 计算机科学
tags:
- 计算机科学
- 深入理解计算机系统
- 计算机系统
---

我阅读 CSAPP 的速度非常慢，因为希望自己能够真正「深入理解」操作系统。这周末，我终于磕磕绊绊地完成了 Bomb Lab，实验过程中主要涉及到阅读汇编代码和 GDB 调试器的使用，还有就是需要一点耐心去分析汇编。

## 实验目标

Bomb Lab 的实验目标主要是要求对一个二进制文件进行逆向工程。这个二进制文件名叫 bomb，要求你输入七个密码，如果输入不符合要求的话，「炸弹」就会被引爆。所以目标就是要求通过分析二进制文件的汇编代码来得到这七个密码。

对获得 bomb 的汇编表示，我们需要使用 objdump 工具执行一条简单的命令：

```
objdump -d bomb > bomb.s
```

点开 bomb.s 乍一看会被吓到，什么鬼！但仔细分析会发现一些线索，比如程序的起点`main`函数：

```
0000000000400da0 <main>:
```

分析 `main` 的内容我们可以敏锐地发现「拆弹」密码对应的函数：

```
400e32:	e8 67 06 00 00       	callq  40149e <read_line>
  400e37:	48 89 c7             	mov    %rax,%rdi
  400e3a:	e8 a1 00 00 00       	callq  400ee0 <phase_1>
  400e3f:	e8 80 07 00 00       	callq  4015c4 <phase_defused>
  400e44:	bf a8 23 40 00       	mov    $0x4023a8,%edi
  400e49:	e8 c2 fc ff ff       	callq  400b10 <puts@plt>
  400e4e:	e8 4b 06 00 00       	callq  40149e <read_line>
  400e53:	48 89 c7             	mov    %rax,%rdi
  400e56:	e8 a1 00 00 00       	callq  400efc <phase_2>
  400e5b:	e8 64 07 00 00       	callq  4015c4 <phase_defused>
  400e60:	bf ed 22 40 00       	mov    $0x4022ed,%edi
  400e65:	e8 a6 fc ff ff       	callq  400b10 <puts@plt>
  400e6a:	e8 2f 06 00 00       	callq  40149e <read_line>
  400e6f:	48 89 c7             	mov    %rax,%rdi
  400e72:	e8 cc 00 00 00       	callq  400f43 <phase_3>
  400e77:	e8 48 07 00 00       	callq  4015c4 <phase_defused>
  400e7c:	bf 0b 23 40 00       	mov    $0x40230b,%edi
  400e81:	e8 8a fc ff ff       	callq  400b10 <puts@plt>
  400e86:	e8 13 06 00 00       	callq  40149e <read_line>
  400e8b:	48 89 c7             	mov    %rax,%rdi
  400e8e:	e8 79 01 00 00       	callq  40100c <phase_4>
  400e93:	e8 2c 07 00 00       	callq  4015c4 <phase_defused>
  400e98:	bf d8 23 40 00       	mov    $0x4023d8,%edi
  400e9d:	e8 6e fc ff ff       	callq  400b10 <puts@plt>
  400ea2:	e8 f7 05 00 00       	callq  40149e <read_line>
  400ea7:	48 89 c7             	mov    %rax,%rdi
  400eaa:	e8 b3 01 00 00       	callq  401062 <phase_5>
  400eaf:	e8 10 07 00 00       	callq  4015c4 <phase_defused>
  400eb4:	bf 1a 23 40 00       	mov    $0x40231a,%edi
  400eb9:	e8 52 fc ff ff       	callq  400b10 <puts@plt>
  400ebe:	e8 db 05 00 00       	callq  40149e <read_line>
  400ec3:	48 89 c7             	mov    %rax,%rdi
  400ec6:	e8 29 02 00 00       	callq  4010f4 <phase_6>
  400ecb:	e8 f4 06 00 00       	callq  4015c4 <phase_defused>
  400ed0:	b8 00 00 00 00       	mov    $0x0,%eax
  400ed5:	5b                   	pop    %rbx
```

6 个密码分别对应了 `phase_1` 到 `phase_6`，这样，我们的目标就可以具体到分析这些函数。

## 关卡 1

`phase_1` 的汇编代码很简单：

```
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq  
```

这段函数的功能很直接：将读入的字符串和「密码」比较，如果两个字符串不相同，那么就引爆。比较字符串是否相等，也可以从调用 `strings_not_equal` 看出来。在汇编中使用了一个神奇的地址`mov    $0x402400,%esi`，所以我们使用 GDB 查看这个地址中的内容。

![1](/images/2017-4-17/so1/1.PNG)

再次运行程序，并且输入密码试试看：

![2](/images/2017-4-17/so1/2.PNG)

成功！

## 关卡 2

`phase_2`  的汇编代码比前一个稍长：

```
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq
```

其中 `callq  40145c <read_six_numbers>` 表明函数要求用户读入六个数字，并存放在栈中。下几行很关键：
```
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
```
在读取 6 个数字之后，会拿第一个数字和 1 进行比较，如果不相等，那么就引爆炸弹，所以我们可以确定第一个数字一定是 1。

下面几行在一个循环中被执行，意思是从栈中某位置取出一个数，和自身相加，并判断是否和下一个数字相等，也就是相当于判断 `s[i] + s[i] == s[i + 1]`：

```
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
```

现在我们就可以知道这 6 个数字从 1 开始，下一个是前一个的两倍：1， 2， 4， 8， 16， 32。我们输入试试：

![2](/images/2017-4-17/so2/2.PNG)

## 关卡 3

`phase_3` 的开头表示我们调用了 `sscanf` 函数解析输入，如果输入参数的个数小于 1则引爆。

```
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
```

那么我们应该输入什么呢？注意到解析输入之前，用到了一个地址 `0x4025cf`，我们查看他的内容：

![1](/images/2017-4-17/so3/1.PNG)

所以我们输入的应该是两个整数。

下面两行表示我们输入的第一个数不可以大于 7，否则会跳转到引爆点：

```
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
```

接下来是一连串的跳转，根据我们输入的第一个值决定跳转位置：

```
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
```

这上面这段代码里，根据第一个值决定跳转位置的代码非常明显：`jmpq   *0x402470(,%rax,8)`。我们可以使用 GDB 在查看跳转表的内容：

![2](/images/2017-4-17/so3/2.PNG)

这样我们就得到了跳转表：

| 值 | 目标值 |
| --- | --- |
| 0 | 207 |
| 1 | 311  |
| 2 | 707 |
| 3 | 256 |
| 4 | 389 |
| 5 | 206 |
| 6 | 682 |
| 7 | 327 |

也就是所，上表中的任意一对值都可以接触炸弹。

![3](/images/2017-4-17/so3/3.PNG)

## 关卡 4

关卡 4 开头的套路和关卡 3 一样，首先调用 `sscanf` 读入值，如果参数个数不为 2，则引爆。

```
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
```

我们查看在 `0x4025cf` 保存的输入格式要求：

![1](/images/2017-4-17/so4/1.PNG)

可以看到我们需要输入两个整数。

接下来要求第一个参数小于 14，第二个参数是0， 并把14、x、0作为参数传入 `fun4`，要求 `fun4` 返回0在经过仔细推导之后，我发现，当答案为 7、0时，可以成功返回 0。

![2](/images/2017-4-17/so4/2.PNG)

## 关卡 5

关卡 5 要求我们输入一个长度为 6 的字符串，否则就引爆。

```
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb>
```

接下来代码每次从字符串中取出一个字符，并做变换：

```
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
```
可惜我们不知道变换方法是什么，但是代码中有一个突兀的地址，我们打印地址中的内容：

![2](/images/2017-4-17/so5/2.PNG)

里面开头是一段奇怪的字符："maduiersnfotvbyl"

然后，将变换后的字符串和存储在 0x40245e 的字符串作比较，判断是否相当，如果相等，最解除成功。所以我们查看目标字符串：

![1](/images/2017-4-17/so5/1.PNG)

在有了目标字符串后，我们现在知道了上述的字符串可以看做是一张查找表，可以得到对应关系：

0123456789abcdef
maduiersnfotvbyl

所以，我们要得到 “flyers”，就应该输入“9 f e 5 6 7”，将数字对应到 ASCII 码中的字符，可以得到“INOEFG”。

![4](/images/2017-4-17/so5/4.PNG)

## 关卡 6

关卡 6 需要仔细分析，我使用了 GDB 单步运行的方法来分析了运行过程。首先，函数要求读入 6 个数，并确认个数是否为 6。

```
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi
  401106:	e8 51 03 00 00       	callq  40145c <read_six_numbers>
  40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
```

然后，通过双重循环，判断 6 个数之间不存在重复。

```
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx
  401164:	89 10                	mov    %edx,(%rax)
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
  40116f:	be 00 00 00 00       	mov    $0x0,%esi
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
```
这里有一个要细节，就是这段代码保存了和 7 的差：

```
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
```

接下来的代码中，有一个奇怪的地址，我们打印它的内容：

![1](/images/2017-4-17/so6/1.PNG)

通过分析这个结构，我们可以看到它是一个链表，定义类似于:

```
struct Node {
	int value;
	int index;
	struct node *next;
}
```

接下来，代码要求由大到小获取链表中的值：

所以我们打印链表的节点值：

![2](/images/2017-4-17/so6/2.PNG)

从大到小排列他们的索引分别是：3 4 5 6 1 2，考虑被 7 减的操作，所以答案是：4 3 2 1 6 5

![3](/images/2017-4-17/so6/3.PNG)

## 隐藏关卡

隐藏关卡的入口在 `phase_defused` 中，所以，我们在函数入口设置断点，研究怎样才能进入`secret_phase`。`phase_defused` 的开头统计了我们目前输入了多少个答案，如果不为 6，那么你永远都无法触发隐藏关卡。

```
00000000004015c4 <phase_defused>:
  4015c4:	48 83 ec 78          	sub    $0x78,%rsp
  4015c8:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  4015cf:	00 00 
  4015d1:	48 89 44 24 68       	mov    %rax,0x68(%rsp)
  4015d6:	31 c0                	xor    %eax,%eax
  4015d8:	83 3d 81 21 20 00 06 	cmpl   $0x6,0x202181(%rip)        # 603760 <num_input_strings>
  4015df:	75 5e                	jne    40163f <phase_defused+0x7b>
```

接下来，我们分析触发 `secret_phase` 的代码：

```
  4015e1:	4c 8d 44 24 10       	lea    0x10(%rsp),%r8
  4015e6:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  4015eb:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  4015f0:	be 19 26 40 00       	mov    $0x402619,%esi
  4015f5:	bf 70 38 60 00       	mov    $0x603870,%edi
  4015fa:	e8 f1 f5 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  4015ff:	83 f8 03             	cmp    $0x3,%eax
  401602:	75 31                	jne    401635 <phase_defused+0x71>
  401604:	be 22 26 40 00       	mov    $0x402622,%esi
  401609:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  40160e:	e8 25 fd ff ff       	callq  401338 <strings_not_equal>
  401613:	85 c0                	test   %eax,%eax
  401615:	75 1e                	jne    401635 <phase_defused+0x71>
  401617:	bf f8 24 40 00       	mov    $0x4024f8,%edi
  40161c:	e8 ef f4 ff ff       	callq  400b10 <puts@plt>
  401621:	bf 20 25 40 00       	mov    $0x402520,%edi
  401626:	e8 e5 f4 ff ff       	callq  400b10 <puts@plt>
  40162b:	b8 00 00 00 00       	mov    $0x0,%eax
  401630:	e8 0d fc ff ff       	callq  401242 <secret_phase>
```

首先程序将三个值保存在寄存器，我们查看它们的内容：

![sss](/images/2017-4-17/secret/sss.png)

分别是7、4、3。

下面是两个特殊的地址，并且将地址指向的值作为了 `sscanf` 的参数。我们打印这两个特殊地址的值：

![ss](/images/2017-4-17/secret/ss.png)

现在我们可以看到，`sscanf` 要求的输入格式是两个整数和一个字符串，而其中的两个整数是我们在关卡 4输入的值，那么剩下的字符串应该是什么呢？

我们可以看到，剩余的代码中还有一系列地址，我们打印他们指向的内存中的值：

![s](/images/2017-4-17/secret/s.png)

可以看到最后一个就是我们想要的字符串。我们把它加到关卡 4 的答案后面即可进入隐藏关卡。

接下来我们查看隐藏关卡的代码：

```
0000000000401242 <secret_phase>:
  401242:	53                   	push   %rbx
  401243:	e8 56 02 00 00       	callq  40149e <read_line>
  401248:	ba 0a 00 00 00       	mov    $0xa,%edx
  40124d:	be 00 00 00 00       	mov    $0x0,%esi
  401252:	48 89 c7             	mov    %rax,%rdi
  401255:	e8 76 f9 ff ff       	callq  400bd0 <strtol@plt>
  40125a:	48 89 c3             	mov    %rax,%rbx
  40125d:	8d 40 ff             	lea    -0x1(%rax),%eax
  401260:	3d e8 03 00 00       	cmp    $0x3e8,%eax
  401265:	76 05                	jbe    40126c <secret_phase+0x2a>
  401267:	e8 ce 01 00 00       	callq  40143a <explode_bomb>
  40126c:	89 de                	mov    %ebx,%esi
  40126e:	bf f0 30 60 00       	mov    $0x6030f0,%edi
  401273:	e8 8c ff ff ff       	callq  401204 <fun7>
  401278:	83 f8 02             	cmp    $0x2,%eax
  40127b:	74 05                	je     401282 <secret_phase+0x40>
  40127d:	e8 b8 01 00 00       	callq  40143a <explode_bomb>
  401282:	bf 38 24 40 00       	mov    $0x402438,%edi
  401287:	e8 84 f8 ff ff       	callq  400b10 <puts@plt>
  40128c:	e8 33 03 00 00       	callq  4015c4 <phase_defused>
  401291:	5b                   	pop    %rbx
  401292:	c3                   	retq   
```

代码首先读入一个字符串，然后调用 `strtol` 将它转换成 10 进制数，并和 0x3e8（也就是 1000）比较，如果大于1000，则引爆；否则，调用 `fun7`，`fun7`的参数是一个地址和一个数。如果 `fun7` 的返回值等于 2，则拆弹成功。

我们看看 `fun7` 做了什么。

```
0000000000401204 <fun7>:
  401204:	48 83 ec 08          	sub    $0x8,%rsp
  401208:	48 85 ff             	test   %rdi,%rdi
  40120b:	74 2b                	je     401238 <fun7+0x34>
  40120d:	8b 17                	mov    (%rdi),%edx
  40120f:	39 f2                	cmp    %esi,%edx
  401211:	7e 0d                	jle    401220 <fun7+0x1c>
  401213:	48 8b 7f 08          	mov    0x8(%rdi),%rdi
  401217:	e8 e8 ff ff ff       	callq  401204 <fun7>
  40121c:	01 c0                	add    %eax,%eax
  40121e:	eb 1d                	jmp    40123d <fun7+0x39>
  401220:	b8 00 00 00 00       	mov    $0x0,%eax
  401225:	39 f2                	cmp    %esi,%edx
  401227:	74 14                	je     40123d <fun7+0x39>
  401229:	48 8b 7f 10          	mov    0x10(%rdi),%rdi
  40122d:	e8 d2 ff ff ff       	callq  401204 <fun7>
  401232:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401236:	eb 05                	jmp    40123d <fun7+0x39>
  401238:	b8 ff ff ff ff       	mov    $0xffffffff,%eax
  40123d:	48 83 c4 08          	add    $0x8,%rsp
  401241:	c3                   	retq   
```

`fun7` 是一个递归函数，它的 C 语言版本为：

```c
// %rdi 为 &x, %esi 为 y

int fun7(int *x, int y) {
    if (x == NULL)
        return -1;
    int ret = 0;
    if (*x < y) {
        ret = fun7(x + 0x10, y);
        ret = ret * 2 + 1;
    } else if (*x == y) {
        return 0;
    } else {
        ret = fun7(x + 0x8, y);
        ret *= 2;
    }
}
```

为了让 `fun7` 返回 2，需要递归三层：

> 2 = 2 * 1
> 1 = 2 * 0 + 1
> 0 = 0

我们让 0x6030f0 + 0x08 + 0x10 = 0x603108，访问这个地址，我们得到 0x16（十进制为 22）。

![su](/images/2017-4-17/secret/su.png)

终于，解决了隐藏关卡！

## 总结

这个实验耗费了我将近 4 天的时间，期间我无数次有过摔电脑的冲动。不过完成之后，我还是获得了慢慢的成就感，加深了对汇编语言的认识，也学习了 GDB 的使用方法。最关键的收获就是要自己冷静地分析和思考问题，相信自己可以做出来。

下一个 Lab，我来了。


