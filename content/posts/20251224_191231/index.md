---
title: "MacTux 开发笔记 #2：Alpine Linux 容器"
date: 2025-12-24
draft: false
summary: "关于开发 MacTux 项目的一些技术细节与杂谈。"
tags: ["技术", "MacTux"]
series: ["MacTux 开发笔记"]
series_order: 2
---

> 注意，这是对之前的开发历程的回顾，文章更新进度远落后于 MacTux 实际开发进度。

晚上好！接上篇内容，能够运行 Hello World 之后，下一步就是运行一些... 更复杂的东西。而对于实现 MacTux 来说，比较重要的事情就是让
shell 能够运行起来。我们先从比较简单的开始，Alpine Linux：它的 `minirootfs` 仅有 5 MB，使用的内核特性也比较基础，
对我们来说是一个比较不错的起点。

## MacTux Server

在“一切皆文件”的思想下，文件系统几乎可以视为在 Linux 上访问系统资源的基础。因此，实现文件相关的系统调用，比如 `read()`, `write()`,
`open()` 是非常重要的。之前我们已经实现了 `write()`，它把 Linux 的 fd 和 macOS 的 fd 对等映射（identity mapping）：我们在 Linux
上执行 `read(1, ...)`，在 macOS 上操作的 fd 也是 1。最常见的几种 fd 是 Linux 和 macOS 共有的，所以这种方式到这个阶段还没有太大问题。

但是，`open()` 似乎... 不能直接映射到 macOS。Linux 和 macOS 的目录结构存在较为显著的差异，不仅如此，`chroot()` 在 macOS 下几乎不可用。
我们还需要支持 procfs、chroot、挂载命名空间等特性，这一切都要在非特权用户下完成。这就使得简单地为所有涉及路径的请求加上前缀也变得不可行。

如果要支持 procfs 的话，对 `read()` 等 fd 系统调用沿用原来的对等映射还可行吗？似乎也不可行。诸如 procfs 内的文件的特殊文件，
在 macOS 下并不具有作为文件的实体，而且没有一种特殊 fd 类型，可以在没有文件实体的情况下，“在用户空间”实现所有文件应当具有的操作（比如 `seek()`）。

综合考虑上面的因素，我们必须拥有自己的 VFS 堆栈，来正确地处理 `open()` 等涉及路径的请求。我们还需要定义一种特殊的 fd：对它们的所有操作都会
由 MacTux 的代码执行。这种特殊的 fd 称为 vfd (virtual file descriptor)。为了性能，fd 仍然采用对等映射；但所有 vfd 在 macOS
中都记录为打开 `/dev/null`。MacTux 代码会追踪所有打开的 vfd，并在 fd 操作前检查记录以选择正确的处理方法。

多个 MacTux 模拟的进程可能共用同一套文件系统，有共享的资源，需要互相通讯。因此，我们创建一个新的子项目，`mactux_server`（下面简称 Server）。
一个 Server 为很多 MacTux 进程服务；Server 内部将会维护 VFS 和 VFD，以及其他任何需要在进程间共享的系统结构。

服务器与客户端通信的方式选择了 Unix Stream Socket，因为它提供了比较丰富的特性，并且使用起来很方便，性能也不会很差。最初，MacTux
使用 `bincode` 用于消息的序列化；但由于 [Bincode v3.0.0 事件](https://docs.rs/bincode/3.0.0)，序列化协议已切换到 Postcard。

## 虚拟文件系统

当务之急是实现 VFS。在预想中，我们首先需要这样几个文件系统：

 - nativefs: 直接将 macOS 的某个目录挂载到 Linux。这将是使用最频繁的文件系统。
 - tmpfs: 和 Linux 的 tmpfs 功能一样。我们的 `/dev` 将需要挂载到 tmpfs。
 - proc: 挂载到 `/proc`。

对于 nativefs 而言，我们需要将已经处理以去掉 "." 和 ".." 的路径，处理好后完整地传递到 macOS。我们还需要处理符号链接。在用户空间，
我们不可能为 nativefs 的每个节点创建像内核里那样的 VFS 数据结构。因此，我们的 VFS 实现更像是 Fuchsia。比如，对于挂载，
我们将挂载点实现为 `Vec<Mountpoint>`，挂载新的文件系统时，将新的 `Mountpoint` 对象插入到数组的最后一个元素；解析路径时，
从数组尾向数组首查找对应的挂载点。

每个进程的 "current working directory" 直接记录为路径。每打开一个文件，其“真实路径”（在打开它的挂载命名空间下去除任何符号链接的完整路径）
会被记录：这是为了实现 `xxxat()` 系列系统调用。当我们发起解析路径的请求时，如果是相对路径，"current working directory"
（或者用户请求 `xxxat()` 的 dirfd 的真实路径）会被追加到用户给出的路径之前；之后整个路径，作为绝对路径，会被“清楚化”，
即处理好所有 "." 和 ".."。最终输出的绝对路径将不含任何 "." 或 ".."。找到挂载点后，相对于挂载点的路径和完整绝对路径将被发送到文件系统驱动。

发送完整绝对路径是有必要的，因为我们需要实现符号链接。文件系统驱动将会负责处理符号链接。对于符号链接是相对路径的情况，会用到完整绝对路径。
文件系统驱动在解决好符号链接（即将符号链接指向的路径解析为绝对路径）后，将会递归发起解析请求。这种方法可能降低了代码的重用性，
但是对于提升 nativefs 的性能是很有必要的。

要实现 nativefs 的符号链接，我们应该怎么做？很容易想到一种方法：从前到后对每一段调用 `lstat()`。但这样，性能是非常糟糕的。在 macOS 上，
我发现了一些独有功能 `O_NOFOLLOW_ANY` 等，它的语义是，只要传入的路径“中间”（而不是本身/最后）包含符号链接，整个系统调用就会报错
ELOOP。使用这个功能，在不包含符号链接时，只需要一次额外的系统调用（检查路径本身/最后是否为符号链接）；包含符号链接时，性能也不会差于前面的
简单方法。这种方法要求从后向前逆序检查符号链接，因此我们必须要求文件系统驱动自行处理符号链接。

## 实现 `fork()`

在 Unix 系统中，`fork()` 被用来创建新进程，所以 Shell 的核心功能之一就是调用 `fork()`。macOS 当然也有 `fork()` 这个系统调用。
我们在 fork 前后处理好 MacTux 运行时本身所用的资源，直接在系统调用处理函数中调用 `fork()`。敲下 `cargo run -- test.elf`，运行测试程序，
却看到：

```text
Breakpoint Trap
```

显然，程序发生了 SIGTRAP 错误，这意味着我们的操作触发了 libSystem 中的断言代码。和其他 macOS 用户交流并给出最小复现代码后，
我发现，“在信号处理程序中调用 `fork()` 得到的子进程在函数返回后会立即崩溃”不仅限于 Rosetta 2 和当前 macOS 版本。在 macOS 12
的 Intel 机器就已经存在。深入到 macOS 源代码，这个错误似乎和 `sigreturn` 校验码机制有关：

信号处理程序在返回前需要调用 `sigreturn()` 系统调用（这个调用一般由 libc 自动完成）。macOS 上面，出于安全考虑，`sigreturn()`
接受一个“密钥”，只有内核和用户空间的密钥一致，这个系统调用才能成功；否则返回 EACCES。这种失败情况在 macOS 的 libc 实现将会直接触发
`ud2` 指令，即产生 SIGTRAP 异常。

然而，macOS 手册和 POSIX 手册都将 `fork()` 列为能够在信号处理程序中安全调用的函数；Linux 下，类似的问题也并不存在。因此，
我推测这是一个在 macOS 中潜藏多年的 bug。

... 但如何绕过这个问题，解决在 MacTux 中的需要呢？MacTux 引入了 "indirect system calls"：

```rust
macro_rules! impl_syscall_indirect {
    ($name:ident = $blk:expr) => {
        unsafe fn $name(uctx: &mut libc::ucontext_t) {
            unsafe extern "sysv64" fn __impl(mut ctx: Box<libc::__darwin_mcontext64>) -> ! {
                unsafe {
                    rtenv::emuctx::leave_emulated();
                    ctx.__ss.__rax = $blk(&mut *ctx);
                    rtenv::emuctx::enter_emulated();
                    core::arch::asm!(
                        "mov rdi, {}",
                        "mov rax, 479", // pseudo_restorectx
                        "syscall",
                        in(reg) Box::into_raw(ctx),
                        options(nostack, noreturn),
                    );
                }
            }

            unsafe {
                rtenv::emuctx::leave_emulated();
                let original_ctx = Box::new(*uctx.uc_mcontext);
                (*uctx.uc_mcontext).__ss.__rdi = Box::into_raw(original_ctx) as usize as u64;
                (*uctx.uc_mcontext).__ss.__rip = __impl as *const () as u64;
                rtenv::emuctx::enter_emulated();
            }
        }
    };
}

impl_syscall_indirect!(
    sys_fork = |_| {
        match rtenv::process::fork() {
            Ok(n) => n as _,
            Err(err) => -(err.0 as i32) as u64,
        }
    }
);

unsafe fn pseudo_restorectx(uctx: &mut libc::ucontext_t) {
    unsafe {
        rtenv::emuctx::leave_emulated();
        let ctx = Box::from_raw(uctx.arg0() as *mut libc::__darwin_mcontext64);
        *uctx.uc_mcontext = *ctx;
        rtenv::emuctx::enter_emulated();
    }
}
```

间接系统调用速度稍微慢于一般的系统调用，但由于 macOS 上面的问题，这是必须的。它会将系统调用转换为普通的函数调用：
原本的上下文会被完整地复制到堆上，其指针会被入栈；之后会调用实际的系统调用处理函数。当实际的系统调用处理函数返回后，将会调用
“助手”伪系统调用，恢复原来的上下文。这就是间接系统调用的完整过程。

这一方法非常有效。引入间接系统调用后，我们的 `fork()` 测试程序能够成功运行了。

## IPC 与信号可重入性

我们为每个线程设置一个 IPC 客户端，它内层封装单一的 Socket 流。当我们执行 Linux 信号处理程序时，即使是最基础的操作，如 `read()`，
也可能会需要调用 IPC 函数。这就需要我们妥善处理好 IPC 调用，确保不会将流中的数据处在一个未定义的状态。

注意到，Linux 中，许多系统调用就已经是不可打断的，如 `read()`，调用这些系统调用时，进程会处于 `STATE_UNINTERRUPTIBLE`，
这意味着执行到这些系统调用之时，一切发来的信号都会被阻塞。因此，我们可以模拟这一行为。

这带来了区分“可中断 IPC 请求”和“不可中断 IPC 请求“的必要。目前而言，我们定义的绝大多数 IPC 请求都是不可中断 IPC 请求。

不可中断 IPC 请求在调用时，客户端会屏蔽所在线程的所有信号。这保证了流不被破坏。它使用每个线程唯一的客户端，
以确保服务器能准确识别出调用线程，并提供足够高的性能。

可中断 IPC 请求由不可中断 IPC 请求升级而来。在发起一个可中断 IPC 请求前，客户端会建立一个全新的 IPC 连接。该 IPC 连接会被视为一个单独的线程，
因此实现可中断 IPC 请求处理程序时应注意不要依赖于线程号。可中断 IPC 请求可以由客户端中断（发送任意字节或关闭连接）。执行完成后，
服务器会将应答直接写入流，之后关闭流。这种性质决定了对可中断 IPC 请求的调用可以随时被信号打断。

我们处理信号屏蔽字的方式给 `fork()` 带来了问题。在调用 `execve()` 之后，本应调用系统调用处理函数的 `SIGSYS` 会无限阻塞。因此，
我们在 MacTux 的 `sigmask` 实现中忽略我们依赖的信号，`SIGSYS` 和 `SIGSEGV`。

## 实现 `execve()`

实现 `execve()`，我们需要：

 - 将“当前环境”和“execve 的参数”转换为“mactux 命令行参数”
 - 处理好 VFD Table 的 `O_CLOEXEC`

我们在 MacTux Server 中新建一个 IPC 调用。在调用 macOS 的 `execve()` 之前，客户端会被告知哪些 VFD 由于该 `execve()` 已被关闭。
我们无须再对它们对应的 macOS fd 调用 `close()`，因为它们在 macOS 中已经具有 `FD_CLOEXEC`。

对于仍然存在的 VFD，我们整理得到类似这样的字符串：

```text
1:10,2:24,3:31,4:21,5:28 ......
```

这表示 macOS fd 与 VFD 序号的映射。逗号用于分割不同的记录；每项纪录中，冒号前面的是 macOS fd 的数字；冒号后面的是 VFD 在服务器中的标号。
它将通过 MacTux 命令行参数传递。

实现好上面的内容之后，我们可以看到了：

```text
$ mactux /bin/bash
bash-5.3$ ls -a
. ..
```

好耶！

## 总结

在实现了这些内容之后，我们能够运行 bash 了。下一个目标是运行 glibc，我们将会用 Void Linux 进行测试。
