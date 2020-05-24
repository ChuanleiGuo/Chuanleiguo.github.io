---
title: Python 中的 Data Model
date: 2017-05-31 15:13:46
categories:
- 编程语言
tags:
- python
---

## 什么是 Data Model

Data Model 简单来说就是对 Python 语言的基本结构的描述。如果将 Python 本身看做一个 Framework，Data Model 就定义了和 Python 的交互方式。

对象（Object）是 Python 对数据的抽象方式，Python 将所有数据都抽象为对象以及对象见的关系。每一个对象都由三部分构成：`id`、`type` 和 `value`。一个对象在被创建后，它的 `id` 就不会在改变了，我们可以将它当做对象在内存中的地址。`is` 操作符可以比较两个对象的是否有相同的 `id`，`id()` 函数将返回对象地址的整数表示。

对象的 `type` 决定了对象支持哪些操作，以及该类型对象可能持有的值。`type()` 函数返回一个对象的类型，和 `id` 一样，一个对象的类型是*不可改变*的。

如果一个对象的 `value` 是可变的，那么这个对象就是可变的（mutable）；如果对象的 `value` 是不可变的，那么这个对象也就是不可变的（immutable）。如果一个不可变的容器对象包含了可变对象的引用，那么虽然引用所指向的对象可以被改变，但是该容器仍然被认为是不可变的，因为它包含的引用并没有改变。一个对象的可变性（mutability）由该对象的 `type` 决定。比如，数字，字符串和 tuple 都是不可变对象，而字典和集合是可变对象。

Python 是一种公认的易用的语言，我们可以很容易的考猜测或者直觉来正确使用它的功能。但是在刚开始使用 Python 的时候，我们可能会奇怪，为什么我们要使用 `len(array)` 而不是 `array.len()` 来得到一个数组的长度呢？这里的原因就在于 Python 的 Data Model。Data Model 描述了我们与 Python 的交互方式，它通过特殊的语法来调用定义在对象上的魔法方法（Magic Method）。魔法方法通常在方法名的首位都添加 `__`，例如 `__getitem__`。当我们通过下标来访问对象时，`obj[key]`，解析器实际上回去调用 `obj.__getitem__(key)`。我们通过重定义对象中的魔法方法来与 Python 中的基础设施（Iteration Collection，etc）交互。

## Pythonic Card Deck

下面，我们通过一个简单的例子来演示实现 `__getitem__` 和 `__len__` 之后类的功能。

```python
import collections

Card = collections.namedtuple("Card", ["rank", "suit"])

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list("JQKA")
    suits = "spades diamonds clubs hearts".split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]

    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```

上面的代码中我们使用 namedtuple 在表示一张牌，用 `FrenchDeck` 类来代表一副牌。我们可以将它的实例传入 `len()` 在得到一共有多少张牌：

```python
>>> deck = FrenchDeck()
>>> len(deck)
52
```

我们也可以直接通过下标来访问某一张牌，使用通过实现 `__getitem__` 来使类拥有的能力：

```python
>>> deck[0]
Card(rank='2', suit='spades')
>>> deck[-1]
Card(rank='A', suit='hearts')
```

因为我们在 `__getitem__` 中使用了 `self._card` 的 `[]` 操作，所以，我们的类也可以直接获得切片的功能：

```python
>>> deck[:3]
[Card(rank='2', suit='spades'), Card(rank='3', suit='spades'), Card(rank='4', suit='spades')]
>>> deck[12::13]
[Card(rank='A', suit='spades'), Card(rank='A', suit='diamonds'), Card(rank='A', suit='clubs'), Card(rank='A', suit='hearts')]
```

在实现了 `__getitem__` 之后，我们的类也是可遍历的（iterable）：

```python
>>> for card in deck:
...     print(card)
...
Card(rank='2', suit='spades')
Card(rank='3', suit='spades')
Card(rank='4', suit='spades')
...
```

遍历也可以是隐式的，如果一个集合没有实现 `__contains__` 方法，`in` 操作就是做顺序扫描：

```python
>>> Card('Q', "hearts") in deck
True
>>> Card('7', "beats") in deck
False
```

我们可以与标准库交互来随机选择一张牌，`random.choice`可以从一个序列中随机选择一个元素：

```python
from random import choice
>>> choice(deck)
Card(rank='K', suit='hearts')
>>> choice(deck)
Card(rank='J', suit='spades')
```

通过上面的例子我们可以看到，实现魔法方法有两个好处：

1. 类的用户不需要去记忆标准操作的方法名。比如，要得到序列中元素的个数，可直接使用 `len(s)`，而不需要记忆应该使用 `.size()` 还是 `.length()`。
2. 可以更容易地与 Python 标准库协作，从而避免重新发明轮子。

通过实现魔法方法，我们的类的行为很像一个标准的 Python 序列，可以使我们的类获得与语言核心交互的能力（如：迭代和切片）。

## 魔法方法的工作方式

通常，魔法方法是由解析器来调用的而不是用户，除非用户在做元编程（meta programming）。但是对于內建的类型如 list，str，bytearray 等，解析器并不会去数元素的个数。在 CPython 的实现中，`len()` 会之间返回 C 语言结构体 `PyVarObject` 中 `ob_size` 的值，这个值代表了内存中对象的大小。

魔法方法的被调用过程是隐式的。例如，当我们使用 `for i in x` 时，会导致调用 `iter(x)`，再调用 `x.__iter__()`。用户唯一可能会需要经常直接调用的魔法方法是 `__init__`，在作为类的初始化方法。

如果要实现一个魔法方法，通常通过调用內建的方法来实现是更好的选择，应为內建方法可以提供额外的功能而且速度往往更快。

## 总结

Data Model 是对 Python 语言的基本结构的描述，定义了內建或自定义类型与语言的交互方式。我们可以通过实现魔法方法来充分利用 Python 提供的功能，构建强大的类型。

完整的魔法方法列表见：[Data Model](https://docs.python.org/3/reference/datamodel.html)

