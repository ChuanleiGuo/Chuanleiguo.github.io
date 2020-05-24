---
title: Docker 基础知识之 Namespace, Cgroup
date: 2018-08-05 22:14:46
categories:
- 计算机科学
tags:
- docker
---

最近工作上需要使用 Docker，在阅读「[第一本 Docker 书](https://www.amazon.cn/dp/B01E5P05KU/ref=sr_1_1?s=books&ie=UTF8&qid=1532857475&sr=1-1&keywords=第一本docker书)」后了解了如何成为 Docker 的用户，但对 Docker 中用到技术却不甚了解。都说 Docker 是「新瓶装旧球」，文中笔者将学习到的 Docker 基础技术中的 Namespace，Cgroup 与 AUFS 记录如下。

## Namespace

Linux Namespace 是 Linux 内核提供的一个功能，可以实现系统资源的隔离，如：PID、User ID、Network 等。Linux 中的 chroot 命令可以将当前目录设置为根目录，使得用户的操作被限制在当前目录之下而不影响其他目录。

假设我们成立了一家向外售卖计算资源的公司，用户购买了一个实例在运行自己的应用。如果某些用户能够进入到其他人的实例中，修改或关闭其他实例中应用的状态，那么就会导致不同用户之间相互影响；用户的某些操作可能需要 root 权限，假如我们给每个用户都赋予了 root 权限，那么我们的机器也就没有任何安全性可言了。使用 Namespace，Linux 可以做到 UID 级别的隔离，也就是说，UID 为 n 的用户在自己的 Namespace 中是有 root 权限的，但是在真实的物理机上，他仍然是 UID 为 n 的用户。

目前 Linux 共实现了 6 种不同的 Namespace。

| 类型 | 系统调用参数 | 内核版本 |
|---|---|---|
| Mount Namespace | CLONE_NEWNS | 2.4.19 |
| UTS Namespace | CLONE_NEWUTS | 2.6.19 |
| IPC Namespace | CLONE_NEWIPC | 2.6.19 |
| PID Namespace | CLONE_NEWPID | 2.6.24 |
| Network Namespace | CLONE_NEWNET | 2.6.29 |
| User Namespace | CLONE_NEWUSER | 3.8 |


### UTS Namespace

> UTS namespaces allow a single system to appear to have different host and domain names to different processes.

UTS(UNIX Timesharing System) Namespace 可以用来隔离 nodename 和 domainname 两个系统标识。在 UTS Namespace 中，每个 Namespace 可以有自己的 hostname。

我们运行下面程序：

```go
func main() {
    cmd := exec.Command("zsh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }
}
```

这段代码主要是通过系统调用 `clone`，并传入 `CLONE_NEWUTS` 作为参数创建一个新进程，并在新进程内运行 `zsh` 命令。在 Ubuntu 14.04 上运行这段代码，就可以进入一个交互环境，在环境中运行 `ps -af --forest` 就可以看到如下的进程树：

![UTS PS](/images/2018-08-05/uts-pstree.png)

验证下父进程和子进程是否在同一个 UTS Namespace 中：

![UTS NS](/images/2018-08-05/uts-ns.png)

可以看到他们的 UTS Namespace 的编号不同。因为 UTS Namespace 对 hostname 做了隔离，所以在这个环境内修改 hostname 不会影响外部主机。

在目前的 zsh 环境中我们修改 hostname 并打印:

![hostname-chuan](/images/2018-08-05/hostname-chuan.png)

在宿主机上打印 hostname：

![hostname-root](/images/2018-08-05/hostname-main.png)

可以看到，外部的 hostname 没有被内部的修改所影响。

### IPC Namespace

> IPC namespaces isolate processes from SysV style inter-process communication.

IPC(Interprocess Communication) Namespace 用来隔离 System V IPC 和 POSIX message queues。每一个 IPC Namespace 都有自己的 System V IPC 和 POSIX message queue。

我们在上一段代码的基础上增加 `CLONE_NEWIPC` 标识，表示我们要创建 IPC Namespace。

```go
func main() {
    cmd := exec.Command("zsh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }
}
```

在宿主器机查看并创建一个 message queue：

![ipc-main](/images/2018-08-05/ipc-main.png)

运行代码并查看 message queue：

![ipc-chuan](/images/2018-08-05/ipc-chuan.png)

### PID Namespace

> The PID namespace provides processes with an independent set of process IDs (PIDs) from other namespaces.

PID(Process ID) Namespace 可以用来隔离进程 ID。同一个进程在不同的 PID Namespace 中可以拥有不同的 PID。在 Docker Container 中，使用 `ps -ef` 可以看到启动容器的进程 PID 为 1，但是在宿主机上，该进程却又有不同的 PID。

继续在代码上添加 `CLONE_NEWPID` 为子进程创建 PID Namespace。

```go
func main() {
    cmd := exec.Command("zsh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }
}
```

运行代码，首先在宿主机上查看进程树：

![pid-main](/images/2018-08-05/pid-main.png)

可以看到 zsh 的 PID 为 11321。在 Namespace 中打印进程 PID：

![pid-chuan](/images/2018-08-05/pid-chuan.png)

可以看到，打印出的当前 Namespace 的 PID 为 1，也就是说 11321 的进程被映射到 Namespace 中后 PID 为 1。

### Mount Namespace

> Mount namespaces control mount points.

Mount Namespace 用来隔离各个进程看到的挂载点视图。在不同的 Namespace 中，看到的挂载点文件系统层次是不一样的。在 Mount Namespace 中调用 `mount` 和 `unmount` 仅仅会影响当前 Namespace 内的文件系统，而对全局文件系统是没有影响的。

在代码中，我们继续加入 `CLONE_NEWNS` 标识。

```go
func main() {
    cmd := exec.Command("zsh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }
}
```

首先运行代码，然后查看 `/proc` 的文件内容：

![mount-main](/images/2018-08-05/mount-main.png)

可以看到宿主机的 `/proc` 中文件较多，其中的数字是对应进程的相关信息。下面，将 `/proc` mount 到 Namespace 中。

![mount-chuan](/images/2018-08-05/mount-chuan.png)

可以看到现在以 PID 命名的文件夹明显减少。下面使用 `ps -ef` 查看系统进程：

![mount-chuan-ps](/images/2018-08-05/mount-chuan-ps.png)

可以看到，在当前的 Namespace 中，zsh 是 PID 为 1 的进程。这就说明当前 Namespace 中的 mount 和外部是隔离的，mount 操作没有影响到外部。Docker 的 volumn 正是利用了这个特性。

### User Namespace

> User namespaces are a feature to provide both privilege isolation and user identification segregation across multiple sets of processes.

User Namespace 主要是隔离用户的用户组 ID。也就是说，一个进程的 User ID 和 Group ID 在 User Namespace 内外可以是不同的。比较常用的是，在宿主机上以一个非 root 用户运行创建一个 User Namespace，然后在 User Namespace 中被映射为了 root 用户。这意味着这个进程在 User Namespace 中有 root 权限，但是在宿主机上却没有 root 权限。

继续修改代码，添加 `CLONE_NEWUSER` 标识。

```go
func main() {
    cmd := exec.Command("zsh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
    Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID |
        syscall.CLONE_NEWNS | syscall.CLONE_NEWUSER,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }

    os.Exit(1)
}
```

首先在宿主机上查看当前用户和用户组：

![user-main](/images/2018-08-05/user-main.png)

接下来运行程序，并查看用户组：

![user-chuan](/images/2018-08-05/user-chuan.png)

可以看到，UID 是不同的，说明 User Namespace 生效了。

### Network Namespace

> Network namespaces virtualize the network stack. On creation a network namespace contains only a loopback interface.

Network Namespace 用来隔离网络设置、IP 地址和端口号等网络栈的 Namespace。Network Namespace 可以让每个容器拥有自己独立的网络设备，而且容器内的应用可以绑定到自己的端口，每个 Namespace 的端口都不会有冲突。在宿主机搭建网桥后，就能很方便地实现容器之间的通信。

我们继续在代码基础上添加 `CLONE_NEWNET` 标识。

```go
func main() {
    cmd := exec.Command("sh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID |
        syscall.CLONE_NEWNS | syscall.CLONE_NEWUSER | syscall.CLONE_NEWNET,
    }

    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        log.Fatal(err)
    }

    os.Exit(1)
}
```

首先，在宿主机上查看自己的网络设备：

![network-main](/images/2018-08-05/network-main.png)

可以看到在宿主机上有 eth0 和 lo 等网络设备。下面，运行程序，并运行 `ifconfig`：

![network-chuan](/images/2018-08-05/network-chuan.png)

我们发现，在 Namespace 中什么网络设备都没有。这可以断定 Namespace 与宿主机之间的网络是处于隔离状态的。

## Cgroups

Linux Namespace 帮助进程隔离出自己的单独空间，而 Cgroups 则可以限制每个空间的大小。Cgroups 提供了对一组进程及将来子进程的资源限制、控制和统计的能力。

Cgroups 有三个组件：

1. cgroup 负责对进程分组管理，一个 cgroup 包含一组进程并可以设置进程参数
2. subsystem 是一组资源控制模块，可以关联到 cgroup 上，并对 cgroup 中的进程做出相应限制。
3. hierarchy 可以把一组 cgroup 串成一个树状结构，这样 cgroup 可以做到继承。

Cgroups 中的 hierarchy 是一种树状结构，Kernel 为了使得对 Cgroups 的配置更加直观，通过一个虚拟的树状文件系统配置 Cgroups 的，通过层级的目录虚拟出 cgroup 树。我们可以在系统上做实验：

1. 首先，创建并挂载一个 hierarchy
![cgroup-mount](/images/2018-08-05/cgroup-mount.png)
    * `cgroup.clone_children`，`cpuset` 的 subsystem 会读取这个配置文件，如果这个值是 1，子 cgroup 才会继承父 cgroup 的 `cputset` 的配置
    * `cgroup.procs` 是树中当前节点 cgroup 中的进程组 ID
    * `notify_on_release` 和 `release_agent` 会一起使用。`notify_on_release` 标识当这个 cgroup 最后一个进程退出的时候是否执行了 `release_agent`；`release_agent` 使进程退出后自动清理掉不再使用的 cgroup
    * `tasks` 标识该 cgroup 下的进程 ID，将进程 ID 写入 `tasks` 文件中，便会将相应进程加入到这个 cgroup 中

2. 在刚创建好的 hierarchy 上 cgroup 的根节点中拓展出两个子 cgroup
![cgroup-tree](/images/2018-08-05/cgroup-tree.png)
可以看到在 cgroup 目录下创建文件夹的时候，Kernel 会把文件夹标记为子 cgroup，她们继承父 cgroup 的属性。

3. 在 cgroup 中添加和移动进程只需要将进程 ID 写到或移动到 cgroup 节点的 `tasks` 文件中即可

![cgroup-mv](/images/2018-08-05/cgroup-mv.png)

这样，我们就把当前的 3217 进程加入到 cgroup-test:/cgroup-1 中了

4. 通过 subsystem 限制 cgroup 中的进程的资源。我们使用系统为每个 subsystem 默认创建的 hierarchy，如 memory 的 hierarchy 来完成实验。

![cgroup-mem](/images/2018-08-05/cgroup-mem.png)

![cgroup-stress](/images/2018-08-05/cgroup-stress.png)

可以看到系统总的内存为 2GB，其中 stess 只能占用到 5% 左右，也就是 100MB。
