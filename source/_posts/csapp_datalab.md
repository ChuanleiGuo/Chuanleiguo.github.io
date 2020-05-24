---
title: 【深入理解计算机系统实验笔记】| Data Lab
date: 2017-3-12 17:31:23
categories: 
- 计算机科学
tags:
- 计算机科学
- 深入理解计算机系统
- 计算机系统
---

最近阅读「深入理解计算机系统」，并在读书的过程中完成练习题和章节附带实验。[实验一](http://csapp.cs.cmu.edu/3e/labs.html)是「信息的表示和处理」章节的实验，主要考察对有/无符号整数、浮点数在 bite 级别存储和操作的理解。

## 总体要求

实验对可使用的操作、数字有严格的要求：
1. 使用的整数必须在 0~255(0xFF)之间，而不允许使用大的常量，如：0xFFFFFFFF
2. 不可以使用全局变量
3. 在整数上可以使用的一元运算符只有：! 和 ~
4. 在整数上可以使用的二元运算符只有：& ^ | + << >>
5. 对个别题目的限制可能会略微放宽，如允许使用循环
6. 每个答案会限制允许的最大操作数
7. 在完成实验后，可以使用 dlc 工具检查代码的格式和操作数是否符合要求，使用 btest 检查结果是否通过测试。

## 题目及思路

### bitAnd

* 要求：计算 `x & y` 的值
* 允许操作：`~ |`
* 操作数限制：8次

使用了简单的逻辑推导

$$ ~(x & y) = (~x) | (~y) ==> (x & y) = ~((~x) | (~y)) $$

可以得到答案：

```c
int bitAnd(int x, int y) {
  /*
   * ~(x & y) = (~x) | (~y)
   * ==> (x & y) = ~((~x) | (~y))
   */
  return ~((~x) | (~y));
}
```

### getByte

* 要求：得到 `x` 中第 n 个字节
* n 的范围：[0, 3]
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：6

以 x=0x12345678 为例，`getByte(x, 0)=0x78`, `getByte(x, 1)=0x56`。我使用了 0xff 作为 mask，将 x 的指定字节移动到低位后与 mask 做 `&` 运算得到结果。

```c
int getByte(int x, int n) {
  return (x >> (n << 3)) & 0xff;
}
```

做完成这个题目的时候，我犯了一个错误。我认为整数在以 Intel 作为芯片的计算机使用小端存储，所以`0x12345678`在内存中的布局应该是`0x78 0x56 0x34 0x12`，那么如果要取得第 n 个字节的话，首先应该将`x`右移`(3 - n) * 8`位。按照我的错误理解，自然也得到了错误地结果。后来我在认识到：**大/小端存储仅仅关系到整数的字节在内存的布局方式，当整数被加载在 CPU 中之后，字节的布局方式就不再存在大/小端的问题了**

> Endianness only matters for layout of data in memory. As soon as data is loaded by the processor to be operated on, endianness is completely irrelevent. Shifts, bitwise operations, and so on perform as you would expect (data logically laid out as low-order bit to high) regardless of endianness

### logicalShift

*  要求：将 `x` 逻辑右移 n 位
*  假定 `0 <= n <= 31`
*  允许操作：`! ~ & ^ | + << >>`
*  操作数限制：20

这个问题主要是要解决负数右移后会在高位进行[符号扩展](https://en.wikipedia.org/wiki/Sign_extension)的问题，需要在将 x 右移后将高位拓展后的位清零。所以我使用`~(((0x1 << 31) >> n) << 1)`作为 mask 将「扩展位」清零。这里用到的小技巧就是将`(0x01 << 31)`右移可以得到在高位做符号扩展从而得到模板，如`((0x1 << 31) >> 8) `可以得到`0xff000000`。

```c
int logicalShift(int x, int n) {
  return (x >> n) & ~(((0x1 << 31) >> n) << 1);
}
```

### bitCount

* 要求：得到在某个字中为 1 的 bit 个数
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：40

这个题目我不会做，在网上找到了[Hamming weight](https://en.wikipedia.org/wiki/Hamming_weight)算法。这个算法的思想使用分治策略，先计算每两个相邻的 bit 中有多少个 1，之后再依次求解在4、8、16、32个 bit 中共有多少个 1。

```c
int bitCount(int x) {
  x = (x & 0x55555555) + ((x >> 1) & 0x55555555);
  x = (x & 0x33333333) + ((x >> 2) & 0x33333333);
  x = (x & 0x0F0F0F0F) + ((x >> 4) & 0x0F0F0F0F);
  x = (x & 0x00FF00FF) + ((x >> 8) & 0x00FF00FF);
  x = (x & 0x0000FFFF) + ((x >> 16) & 0x0000FFFF);
  return x;
}
```

这个算法还有优化空间，所以也有使用更少操作的实现方法，但是我认为这个基础版本是最容易读的，用户体验更好。该问题及其他高效实现在[StackOverflow](http://stackoverflow.com/questions/109023/how-to-count-the-number-of-set-bits-in-a-32-bit-integer)

### bang

* 要求：计算 `!x`
* 允许操作：`~ & ^ | + << >>`
* 操作数限制：12

这个题目要注意的一点就是在 C 语言中`||`、`&&`和`!`是**逻辑运算**，而`~`、`&`、`|`和`^`是**位级运算**。在逻辑运算中，任何**非零**参数都是表示 True，0 表示 False。所以`bang(3)=bang(-3)=0`，而`bang(0)=1`。我们知道在整数的补码表示中`+0`和`-0`的符号为都为 0。根据这个特点，可以得到解法：

```c
int bang(int x) {
  int negx = ~x + 1;
  int x_sign = (x >> 31) & 0x1;
  int negx_sign = (negx >> 31) & 0x1;

  return (x_sign | negx_sign) ^ 0x1;
}
```

### tmin

* 要求：返回使用2进制补码表示的最小的整数
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：4

了解补码的表示方式后，直接返回`0x80000000`即可。

```c
int tmin(void) {
  return 0x1 << 31;
}
```

### fitsBits

* 要求：如果`x`可以被 n 位的2进制补码表示则返回1，否则返回0
* 假定`1 <= n <= 32`
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：15

n 位补码能够表示的数的范围是$[-2^{n-1}, 2^{n-1}-1]$，所以如果在这个范围内则返回1，否则返回0。如果`x`大于$2^{n-1}-1$，则会发生正溢出，得到负数；如果`x`小于$-2^{n-1}$则会发生负溢出。所以题目的求解方法就是将`x`的符号位从第 n 位拓展到第32位得到`extened_x`，之后通过判断`x == exntened_x`来判断是否发生了溢出。

```c
int fitsBits(int x, int n) {
  int shiftNum = 32 + (~n + 1);
  return !(((x << shiftNum) >> shiftNum) ^ x);
}
```

### divpwr2

* 要求：计算$x/2^{n}$，并向零舍入
* 假定`0 <= n <= 30`
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：15

因为当`x > 0`时，`x >> n`会自动向下(向零)舍入，我们不需要做特殊操作。所以在整数`x`除以$2^{n}$时主要是要实现当`x < 0`时的向零(向上)舍入，实现方法就是为`x`添加一个偏移值`bias = (1 << n) - 1`，计算`(x + bias) >> k`。

> 在为 `x` 添加偏移量后，低 k 位左边的位可能会加 1，也可能不会加 1。对于不需要舍入的情况，加上偏移量只影响那些被移除的位；对于需要舍入的情况，加上偏移量导致较高的位加 1，所以结果会向零舍入。

```c
int divpwr2(int x, int n) {
  int sign = x >> 31;
  int bias = sign & ((1 << n) + (~0));
  return (x + bias) >> n;
}
```

### negate

* 要求：计算 -x
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：5

在整数的补码表示中 `-x = (~x) + 1`，所以程序很简单：

```c
int negate(int x) {
  return ~x + 1;
}
```

### isPositive

* 要求：如果`x > 0`返回1，否则返回 0
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：8

利用向右算术移位运算中的符号拓展原则，如果`x < 0`那么`x >> 31`会得到 0xFFFFFFFF。注意还要处理`x = 0`的情况。

```c
int isPositive(int x) {
  return !((x >> 31) | (!x));
}
```

### isLessOrEqual

* 要求：如果`x <= y`返回 1，否则返回 0
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：24

只判断三种情况：

1. x < 0 并且 y > 0
2. x 和 y 的符号位相同，并且 x - y < 0
3. x = y

```c
int isLessOrEqual(int x, int y) {
  int signx = (x >> 31) & 0x1;
  int signy = (y >> 31) & 0x1;
  int differ = x + (~y + 1);
  int sign_d = (differ >> 31) & 0x1;
  return (signx & (!signy)) | ((!(signx ^ signy)) & sign_d) | (!(x ^ y));
}
```

### ilog2

* 要求：计算$log_2^{x}$，向下舍入
* 允许操作：`! ~ & ^ | + << >>`
* 操作数限制：90

看这道题目的操作数的限制，就知道很有难度。因为在得到结果后需要向下舍入，所以我们只需要判断`x`的补码表示中最高位`1`的位置，或者说要得到最高位`1`后面的位数。这里主要是使用了类似分治的策略：考虑字的左边 16 位是否为全 0，如果是的话，不做任何操作，如果不为全零的话，说明最高位的`1`在`x`补码表示的左半边，那么可以通过右移“砍掉”右半边的数字，并记录被“砍掉”的位数。

```c
int ilog2(int x) {
  int shift1, shift2, shift3, shift4, shift5;

  int sign = !!(x >> 16);
  shift1 = sign << 4;
  x = x >> shift1;

  sign = !!(x >> 8);
  shift2 = sign << 3;
  x = x >> shift2;

  sign = !!(x >> 4);
  shift3 = sign << 2;
  x = x >> shift3;

  sign = !!(x >> 2);
  shift4 = sign << 1;
  x = x >> shift4;

  sign = !!(x >> 1);
  shift5 = sign;
  x = x >> shift5;

  return shift1 + shift2 + shift3 + shift4 + shift5;
}
```

感觉代码要比我的文字清晰易读多了。

### float_neg

* 要求：f 为浮点数的位级表示，返回`-f`的位级表示，如果 f 为 NaN，返回 f
* 允许操作：允许所有有/无符号数的操作，并允许使用 if 和 while
* 操作数限制：10

浮点数使用 [IEEE754](https://en.wikipedia.org/wiki/IEEE_floating_point)标准，单精度的浮点数中有 1 位符号位，8 为指数位和 23 位小数位。

![](/images/2017-3-12/14893087870697.png)

所以当 `f` 不为 NaN 时，要返回`-f`，只需要将符号位取反即可；判断 f 是否为 NaN 则要判断指数位是否为全 1，并且小数为不为全 0。

```c
unsigned float_neg(unsigned uf) {
  unsigned exp_mask = 0xff << 23;
  if ((!((uf & exp_mask) ^ exp_mask)) && (uf & ((1 << 23) + (~0)))) {
    return uf;
  }  return (uf ^ (0x1 << 31));
}
```

### float_i2f

* 要求：返回整数 x 的浮点数的位级表示
* 允许操作：允许所有有/无符号数的操作，并允许使用 if 和 while
* 操作数限制：30

讲真，最开始做这个题目的时候我的大脑是一片混乱的，因为要处理的细节很多：舍入、指数位增加偏移量等。最终我参考了[小土刀](http://wdxtub.com/2016/04/16/thick-csapp-lab-1/)的答案，才基本理清了思路。

要返回的答案，需要组合浮点数三部分(符号位、指数位和小数位)，即 
`(sign << 31) | (exponent) << 23 | fraction`。
1. 当 x = 0 时，返回 0
2. 当 x = 0x80000000 时，令 exponent = 158，因为 158 - 127 = 31，其中 Bias = $2^{n-1} - 1$
3. 如果 x < 0, 则 sign = 1；并令 x = -x，这样之后在做移位等操作就不用担心符号位了
4. 计算整数 x 的位数，作为计算 exponent 的依据
5. 得到 exponent 后，将 x 右移计算小数值 fraction，并且根据移除的值是否大于等于1.5来判断是否要进位，从而实现向零舍入
6. 如果小数位过多，则丢弃最高位并增加 exponent
7. 组合符号位、指数位和小数位

```c
unsigned float_i2f(int x) {
  int sign = (x >> 31) & 0x1;
  int i;
  int exponent = 0;
  int fraction = 0;
  int delta = 0;
  int fraction_mask;
  if (x == 0) {
    return x;
  } else if (x == 0x80000000) {
    exponent = 158;
  } else {
    if (sign) {
      x = -x;
    }
    i = 30;
    while ( !(x >> i) ) {
      i --;
    }
    exponent = i + 127;

    x = x << (31 - i);
    fraction_mask = 0x7fffff;
    fraction = fraction_mask & (x >> 8);
    x = x & 0xff;
    delta = x > 128 || ((x == 128) && (fraction & 0x1));
    fraction += delta;
    if (fraction >> 23) {
      fraction &= fraction_mask;
      exponent += 1;
    }
  }
  return (sign << 31) | (exponent << 23) | fraction;
}
```

### float_twice

* 要求：返回 2 * f 的位级表示，f 为浮点数 f 的位级表示
* 如果 f 为 NaN 则返回 f
* 允许操作：允许所有有/无符号数的操作，并允许使用 if 和 while
* 操作数限制：30

我们可以针对不同的情况来操作：
1. 若 f 为`+0`或`-0`，返回 f
2. 若 f 为无穷大或 NaN，返回 f
3. 若 f 为非规格化浮点数，则保留符号位，并将 f 左移1位
4. 若 f 为规格化浮点数，则将 f 的指数位增加1

```c
unsigned float_twice(unsigned uf) {
  if (uf == 0 || uf == 0x80000000) { return uf; }
  if (((uf >> 23) & 0xff) == 0xff) { return uf; }
  if (((uf >> 23) & 0xff) == 0x00) {
    return (uf & (0x1 << 31)) | (uf << 1);
  }
  return uf + (1 << 23);
}
```

## 最终结果

代码规范检查：
![dl](/images/2017-3-12/dlc.png)

代码结果检查：

![btest](/images/2017-3-12/btest.png)



