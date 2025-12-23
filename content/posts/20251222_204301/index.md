---
title: "MacTux 开发笔记 #1：Hello World"
date: 2025-12-22
draft: false
summary: "关于开发 MacTux 项目的一些技术细节与杂谈。"
tags: ["技术", "MacTux"]
series: ["MacTux 开发笔记"]
series_order: 1
---

## 适用于 macOS 的 Linux 兼容层？

（注意，这是对之前的开发历程的回顾，文章更新进度远落后于 MacTux 实际开发进度。）

作为 ~~使用 macOS 的~~ Linux 深度用户，很多时候我会给 Linux 开发一些软件，不可避免地用到 Linux 特有的特性。在我的 MacBook Air 上，
有这样的两个选择：

 - 在 Asahi Linux 上开发：如果要用到 macOS 上的程序和数据就要重启切换系统，比较麻烦；
 - 使用虚拟机：8GB 的内存显得不太充裕；

嗯... 我的配置是 8+256，似乎这两个选择都不是太适合，但如果要在上面开发 Linux 程序，似乎也只能这样做。

适用于 macOS 的 Linux 兼容层？类似 WSL1 和 Wine？Wine 的原理是模拟 `ntdll.dll`，但是对于 Linux，模拟 `glibc` 似乎不行... 每个
Linux 发行版都要携带自己的 `libc` 动态库，可能有不同的补丁；而且 `musl libc` 程序和静态链接程序也很普遍... 所以，唯一可行的路径是对
Linux 系统调用进行模拟。

似乎 macOS 上没有能模拟系统调用的方法... 直到我看到了 Wine 的一个 PR，其大意为：所有 Windows 系统调用都是非法的 macOS 系统调用，因此，
安装 `SIGSYS` 信号处理程序可以在 macOS 上模拟 Windows 系统调用。

具体来说，在 macOS 上，系统调用号（指通过 `rax` 或 ARM 上对应寄存器传递的数字）由两部分组成：

```
syscall_number = syscall_kind | syscall_no
```

其中，`syscall_no` 是类似于 Windows 和 Linux 上“系统调用号”的概念，是比较小的数字。而 `syscall_kind` 是很大的数字，激活位均处于高
位，且永远不为 `0`。

`syscall_kind` 有三种选择：`MACHDEP`、`BSD` 和 `MACH`。我们常用的系统调用，比如 `read()`、`write()` 属于 `BSD`。Apple 定义，
`BSD` 的值是 `0x80000000`，因此，假如我们要调用 `BSD` 下的 1 号系统调用，最终的系统调用号将会是：

```
syscall_number = 0x80000000 | 0x1 = 0x80000001
```

恰好，在 x86_64 平台上，Linux 和 macOS 的系统调用传参顺序是一样的。所以，对于模拟 Linux 系统调用而言，Wine 用于 Windows 的那种办法
是一样适用的。理解了上述内容，就可以开始第一个实验了。

在开始之前，要给项目想一个好的名字。`MacTux` 来自 `Mac` 和 `Tux` 的组合，其中 `Tux` 就是 Linux 的吉祥物的名字了，这应该算是一个
不错的名字 :-\)

## 第一个实验：~~简陋到不能称为 Hello World 的 Hello World~~

先实验一下这个想法的可能性。先让 AI 写一段汇编，在 x86 Linux 上输出 "Hello, world"：

```assembly
section .data
    msg db 'Hello World', 0xA
    len equ $ - msg

section .text
    global _start

_start:
    ; 64位系统调用
    mov rax, 1                    ; sys_write = 1
    mov rdi, 1                    ; stdout = 1
    mov rsi, msg                  ; 字符串地址
    mov rdx, len                  ; 字符串长度
    syscall                       ; 64位系统调用指令
```

将它作为内联汇编添加到 `main.rs` 的主函数，之后让我们编写 `SIGSYS` 的信号处理程序。使用 `sigaction`，我们可以在信号发生时读取，甚至
修改处理器上下文，这是能够利用 `SIGSYS` 实现系统调用模拟的基础。为了让上面的程序能够运行，我们在主函数的开头安装我们刚刚编写的信号处理
程序。

输下 `cargo run`，我们看到了：

```text
Hello World
```

成功了！这说明这个想法确实是可行的，可以继续编写了。

## 第二个实验：通过过程宏简化系统调用编写

目前，要编写系统调用的话，我们需要把系统调用传入的 `usize` 转换成我们需要的类型，再把返回值转换回 `usize`。这固然很直接，但是，这会导致
我们需要大量使用 `unsafe` 代码，而且也没办法利用诸如 `?` 运算符的方便的特性。

我们定义 `FromSyscall` 和 `ToSysret` 两个 trait 作为转换，之后编写过程宏，这样，我们就可以像这样编写系统调用了：

```rust
#[syscall]
pub unsafe fn sys_swapoff(_dev: *const c_char) -> Result<(), LxError> {
    Err(LxError::EPERM)
}
```

## 第三个实验：加载 ELF

一个很重要的事情是从 ELF 文件加载 Linux 代码。再一次，让 AI 写一个 Hello World 汇编程序：

```assembly
section .data
    msg db 'Hello World', 0xA
    len equ $ - msg

section .text
    global _start

_start:
    ; 64位系统调用
    mov rax, 1                    ; sys_write = 1
    mov rdi, 1                    ; stdout = 1
    mov rsi, msg                  ; 字符串地址
    mov rdx, len                  ; 字符串长度
    syscall                       ; 64位系统调用指令

    ; 退出
    mov rax, 60                   ; sys_exit = 60
    xor rdi, rdi                  ; 退出码 = 0
    syscall
```

我们把它编译成静态二进制程序，`a.out`。编写了 ELF Loader 之后尝试加载这个二进制程序，似乎... `mmap()` 失败了？

是的，这是因为 macOS 默认存在 4GB 大小的 `pagezero` 段，这会阻止加载器将静态链接二进制程序加载到 Linux 上可用的位置。如果将它编译为
启用 `static-pie` 的程序，执行起来就没有问题了。但... 静态程序一样很重要。

所幸，macOS 提供了一种链接器参数，可以自定义 `pagezero` 段的大小：

```
-Wl,-pagezero_size,0x4000
```

它至少为一页。考虑到 ARM 支持，我们将它设置为 16K。设置在 `build.rs` 之后，我们成功看到了：

```text
Hello World
```

那么... 可以运行一下真正的，使用 musl libc 的 Hello World 了。我们一样从静态版本开始。

## 第四个实验：真正的... Hello World

这是每个人入门 C 语言时都要编写的代码：

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, world!\n");
}
```

我们在 Alpine Linux 用特定参数下编译它，产生静态链接二进制文件 `a.out`。我们第一个没有实现的东西就是 `argc` 和 `argv`。当然，
熟悉 Unix 操作系统的会知道，后面还有一个 `envp`。翻阅 Linux ABI 文档之后，我们很快可以知道程序启动时，栈的布局是：

```text
|---------------------------| <-- 栈顶（高地址）
|           argc            |
|          argv...          |
|          envp...          |
|          auxv...          |
|---------------------------| <-- 栈底（低地址）
```

在此阶段，我们用不到 `auxv`，将它留在以后实现。以 Rust 代码为主，加上少量汇编，我们就可以配置好 Linux 程序入口的堆栈了：

```rust
core::arch::asm!(
    "sub rsp, {stkinfo_len}",
    "mov rdi, rsp",
    "mov rsi, {stkinfo_ptr}",
    "mov rcx, {stkinfo_len}",
    "rep movsb",
    "jmp {entry}",

    stkinfo_ptr = in(reg) stkinfo_ptr,
    stkinfo_len = in(reg) stkinfo_len,
    entry = in(reg) entry,
    options(noreturn),
);
```

忽略掉不支持的系统调用后，静态链接二进制程序... 崩溃了：

```
Segmentation fault
```

观察崩溃日志，我们发现出错原因是访问了 0 地址。观察 `rip` 寄存器的地址，对应到二进制文件，我们发现出错的指令类似于：

```assembly
mov %fs:0x0, %rax
```

其中，`fs` 段被 Linux 用于线程本地存储，在 musl libc 中存储 `struct _pthread`，里面包含 `errno` 结构。而 musl 上，该结构体的
第一个成员就是指向它本身的自引用指针。问题很清晰了。我们注意到一个系统调用：

```
arch_prctl(ARCH_SET_FS, ...) = 0
```

似乎与出错的代码有关。等等... 似乎从信号上下文并无法设置 `fs` 段寄存器？

首先想到的变通办法是通过 `wrfsbase` 指令设置 `fs` 段寄存器。但是经过尝试后，这行不通。macOS，或者说 Rosetta 2，也许，没有启用这个
处理器特性。因此，执行这条指令会立即：

```text
Illegal Instruction
```

通过扩展 Mach 线程 API 的上下文是否可以呢？或许在原生 x86_64 机器上可以，但 Rosetta 2 上会出问题。Rosetta 2 对于这套 API，以及
调试 API 实现的上下文会永远返回 `fsbase=0` 和 `gsbase=0`，任何写操作也不会实际生效。

难道就要这么失败了吗？Linux 上，`gs` 不被使用，而恰好我们在 macOS 上有办法设置 `gsbase`... 而且 macOS 上 `fs` 不被使用... 我们
似乎可以用到一些动态二进制修改技术。

## 第五个实验：动态二进制修改

不设置 `fs` 的情况下，`fsbase` 永远都是 0。`%fs:` 后面会跟一个很小的数字，至少是远小于 16K，也就是 `pagezero` 段的大小的。这就意味着
对 `fs` 段的访问总是意味着 `SIGSEGV`。

我们注意到，对于所有使用了 `fs` 段寄存器的指令，其第一个字节是 `0x64`。将它替换成 `0x65` 就可以变成一条新指令，这条指令和原来的指令唯一的
区别就是把引用的段寄存器从 `fs` 替换成了 `gs`。

一个相当 tricky 但有效的方法：

```rust
#[cfg(target_arch = "x86_64")]
unsafe extern "C" fn handle_sigsegv(_: c_int, info: &libc::siginfo_t, ctx: &mut libc::ucontext_t) {
    // This special handler may process all `fs` accesses to `gs` ones.
    if !reentrant_in_emulated(info) {
        return raise(SigNum::SIGSEGV, info, ctx, false);
    }

    unsafe {
        let insc_byte = (*ctx.uc_mcontext).__ss.__rip as usize as *const AtomicU8;
        match (*insc_byte).compare_exchange(
            0x64,
            0x65,
            atomic::Ordering::Relaxed,
            atomic::Ordering::Relaxed,
        ) {
            Ok(_) => (),
            Err(_) => {
                crate::emuctx::leave_emulated();
                raise(SigNum::SIGSEGV, info, ctx, true);
            }
        }
    }
}
```

并且把所有从 Linux 创建或为 Linux 使用的、带有 EXEC 权限的内存映射都叠加上 WRITE 权限。

如果出错位置位于 macOS 代码，那么会 `raise` 段错误，最终使程序立即退出，这是预期的情况。如果出错位置位于 Linux 代码，那么既能在使用
`fs` 段寄存器时替换到 `gs`，又能在其他时候调用 Linux 程序设定的信号处理程序，很好！

然而，macOS 本身大量依赖于 `gs` 寄存器做线程本地存储。所以，我们设置一个全局映射，允许用 TID 获取 macOS 的 `gsbase` 值。这样，当我们
切换到 Linux 代码时，就使用 Linux 程序设置的 `gsbase`；跳转到 macOS 代码时，就恢复 macOS 原本的 `gsbase`。

做好这些工作，我们再次执行 "Hello World"，观察到：

```text
Hello World
```

成功了！

## 第六个实验：动态链接

虽说静态链接程序在 Linux 上很重要，但还是动态链接程序更常见。让我们编译一个 `static-pie` 的二进制程序 —— 虽然名字带 `static`，但它使用
和动态链接一样的技术。我们重新编译那个文件，使用 `static-pie`，然后执行... 又得到了：

```text
Segmentation Fault
```

调试后发现和 `auxv` 有关。对于目前的目标 —— 运行 PIE 的 Hello World，我们需要这些 Auxv Entry:

```rust
/// Type of an auxiliary vector entry.
#[derive(Debug, Clone, Copy)]
enum AuxType {
    Null = 0,
    // ExecFd = 2,
    Phdr = 3,
    PhEnt = 4,
    PhNum = 5,
    PageSz = 6,
    Base = 7,
    Entry = 9,
    // Random = 25,
    // Sysinfo = 32,
    // SysinfoEhdr = 33,
}
```

一一实现后，我们得到了熟悉的：

```text
Hello World
```

有了这些积累，再实现读取 `PT_INTERP` 的功能，很快就对动态链接的 Hello World 也一样的结果。

## 总结

总之，我们已经能够成功运行~~各种各样的~~ Hello World 了，这是运行各种复杂 Linux 程序的基础。后面的文章将会分享开发 MacTux 的过程中
遇到的各种问题或有趣的事情，以及各个阶段性目标。
