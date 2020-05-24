---
title: Guard 的用法
date: 2016-09-28 15:14:46
categories: 
- 编程语言
tags:
- swift
---

`guard` 关键字从 Swift 2.0 开始被引入，但是因为这一个关键字同时可以发挥四个作用，所以我们需要全面地理解它。

第一个作用显而易见：`guard` 用来实现提前退出。这意味着一个函数在一些前提条件不满足的情况下可以直接返回。例如，我们写一个简单的函数，实现为一个歌手颁奖：

``` swift
func giveAward(to name: String) {
    guard name == "ChuanleiGuo" else {
        print("No way!")
        return
    }
    
    print("Congratulations, \(name)!")
}
giveAward(to: "ChuanleiGuo")
```

在 `giveAward(to:)` 方法中使用 `guard` 关键字使得只有 ChuanleiGuo 同学才能获奖😋。虽然这个函数没什么实际作用，但是方法能够继续运行的前提条件非常清晰：只有在获奖人是 ChunaleiGuo 同学的时候，方法才能继续进行。

这个例子看起来和直接使用 `if` 语句做判断几乎是一样的，但是使用 `guard` 有一个巨大的优势：使你的意图更加清晰，不仅仅对于人，对于编译器也是一样。这是一个提前退出语句，意味着当条件不满足的时候，方法应该退出。使用 `guard` 说明：这个条件并不是方法的一部分，它存在的意义就是要保证后面的代码**安全**地执行。这对编译器也更加清晰，如果我们删除了上面的 `return` 语句，代码就无法编译了。因为编译器发现这是一个提前退出语句，而我们却没有退出。

第二个作用可以说是意外的收获：使用 `guard` 可以减少我们代码的缩进量。一些开发者朋友仍未永远都不应该使用提前退出，相反，每一个函数仅仅应该在一个地方返回。这就要求在函数的主要部分增加额外的缩进量：

``` swift
func giveAward(to name: String) -> String {
    let message: String
    
    if name == "ChuanleiGuo" {
        message = "Congratulations, \(name)!"
    } else {
        message = "No way!"
    }
    
    return message
}

giveAward(to: "ChuanleiGuo")
```

使用 `guard` 后，我们立刻处理的函数运行的前提条件，并且消除了额外的缩进量，代码也变得更经凑了呢！

第三个作用是使得代码的成功路径更加清晰。成功路径，代表着在没有错误和异常发生的情况下，你的代码的运行路径。有了 `guard` 之后，所有的错误都被立刻消除了。现在代码里仅仅剩下了成功路径。

以上的三个作用都比较简单，但是 `guard` 的还有很重的的一个作用，这也是它和 `if` 语句的重要区别：在我们使用 `guard` 判断并解包一个可选值之后，可选值会继续存在于作用域中。为了演示，我们重写 `giveAward(to:)`:

``` swift
func giveAward(to name: String?) {
    guard let winner = name else {
        print("No one won the award")
        return
    }
    
    print("Congratulations, \(winner)!")
}
```

如果使用 `if - let` 语句的话，`winner` 仅仅会存在于被属于 `if - let` 大括号包围的作用域中。然而，`guard` 使得解包后的可选值继续存在于函数的作用域中，所以我们可以使用 `print()` 直接输出它。这段代码可以这样理解：“试着将解包后的可选值复制给winner，来让我们继续使用它，但是如果可选值为空，那么打印信息并退出”。

最后一个作用，并不新鲜。相反，它仅仅是对于已知知识点的新运用：如果条件不满足的话，`guard`可以让我们跳出任何作用域，不仅仅是函数的方法。也就是说，你可以跳出 `switch`语句，也可以跳出 `while` 和 `for` 循环。

下面的例子，用两种方法实现从1遍历到100，输出所有可以被8整除的数：

``` swift
for i in 1...1000 {
    guard i % 8 == 0 else { continue }
    print(i)
}

for i in 1...1000 where i % 8 == 0 {
    print(i)
}
```

我们可以在不同的情况下选择使用不同的方法。



