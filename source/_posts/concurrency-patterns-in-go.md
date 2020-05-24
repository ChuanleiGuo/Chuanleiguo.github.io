---
title: Go 语言中的常用并发模式
date: 2019-12-22 10:34:23
categories:
- 编程语言
tags:
- 编程语言
- Go-lang
---

正式工作半年来，公司项目以 Go 作为主要语言。由于有 C 语言基础，刚开始我感觉上手 Go 语言很容易，而且比 C 语言有更好的可读性，再加上「函数」变成了一等公民，函数式的写法也让部分功能的实现变得更加简练和易读。但是，我对 Go 语言的并发特性却一直理解不深，总觉得在处理并发任务的时候我的思路依然是使用 Java 的 *communicate by sharing memory*，而不是 Go 语言所提倡的 *share memory by communicating*。所以，近期阅读和实践了一些 Go 语言并发编程相关的书籍和资料，总结了 Go 语言中常用的并发编程模式，记录在这里。

## for-select-loop

`select` 组合多个 `channel` ，`channel` 组合多个 `goroutine`。`select` 是 Go 语言并发编程中最终的指令之一，我们可以在任何上下文，无论是函数、还是多个子系统，中将多个 `channel` 组合在一起，并加入如「取消」、「限时等待」和「默认值」等功能。最简单的例子就是：

```go
var ch1, ch2 <-chan interface{}
var ch3 chan<- interface{}

select {
case <-ch1:
    // logic 1
case <-ch2:
    // logic 2
case ch3<- struct{}{}:
    // logic 3
}
```

与 `switch` 不同的是，多个 `case` 不是同步判断是否满足条件，而是异步判断，如果所有 `case` 都不满足，那么 `select` 语句将一直阻塞。、

`for-select-loop` 是 Go 语言中最常见的使用方法，可以被应用在多个场景中：

顺序写入变量到 `channel`：

```go
for _, str := range []string{"str1", "str2", "str3"} {
    select {
    case <-stop:
        return
    case strChan<- str:

    }
}
```

无限循环执行任务直到被取消：

```go
for{
    select {
    case <-stop:
        return
    default：
    }
    // 循环任务
}
```

## 错误处理

并发编程中，错误处理非常困难。我们花费大量时间思考多个线程之间的内存共享和协调，但是忽略如何优雅地处理错误。Go 语言抛弃了在其他语言中常见的异常抛出机制，并提出开发者应该给予错误处理逻辑分支与正产流程相同的关注。在错误处理汇总最基本的问题是「谁应该处理错误？」。有时候，程序需要停止进一步沿着调用栈向上传递异常，而是处理异常。

在并发编程中，这个问题会变得更加复杂。因为 `goroutine` 是独立于它的父亲和兄弟 goroutine 执行的，很难确定应该如何处理错误。例如：

```go
checkStatus := func(stop <-chan interface{}, urls ...string) <-chan *http.Response {
    responses := make(chan *http.Response)
    go func() {
        defer close(responses)
        for _, url := range urls {
            resp, err := http.Get(url)
            if err != nil {
                fmt.Println(err)
                continue
            }
            select {
            case <-stop:
                return
            case responses <- resp:
            }
        }
    }()
    return responses
}

stop := make(chan interface{})
defer close(stop)

urls := []string{"https://google.com", "https://host"}
for response := range checkStatus(stop, urls...) {
    fmt.Printf("Response: %v\n", response.Status)
}
```

例子中发送 http 请求的 goroutine 没有办法将错误返回，只能打印错误信息防止将错误吞掉。所以，不应该使 goroutine 无法回传错误。更好的方式是注意点分离，并发的 goroutine 应该将错误信息传递给另外一个更够访问程序全部状态的组件，并正确处理错误。如我们可以封装返回值和错误信息：

```go
type Result struct {
    Error error
    Response *http.Response
}

checkStatus := func(stop <-chan interface{}, urls ...string) <-chan *http.Response {
    results := make(chan Result)
    go func() {
        defer close(results)

        for _, url := range urls {
            resp, err := http.Get(url)
            result = Result{Error: err, Response: resp}

            select {
            case <-stop:
                return
            case results <- result:
            }
        }
    }()
    return results
}

stop := make(chan interface{})
defer close(stop)

urls := []string{"https://google.com", "https://host"}
for result := range checkStatus(stop, urls...) {
    if result.Error != nil {
        fmt.Printf("error: %v", result.Error)
        continue
    }
    fmt.Printf("Response: %v\n", result.Response.Status)
}
```

这种方式的关键在于我们将可能的结果和错误封装在一起，可能够表示 `checkStatus` 所能够产生的全部结果，而使我们的主流程可能决定应该处理错误情况。

