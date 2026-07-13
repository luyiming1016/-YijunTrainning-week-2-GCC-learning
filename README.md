# GCC 练习记录：从 C 源码到 Linux 进程

**日期：** 2026-07-13  
**环境：** WSL Debian、GCC 14.2.0、x86-64

## 练习目标

理解并亲自验证一段 C 代码如何经过 GCC 的四个阶段，成为可执行的 ELF 文件，并在 Linux 中以进程形式运行。

```text
loop.c → loop.i → loop.s → loop.o → loop（ELF）→ Process
```

## 源码与练习产物

`loop.c` 使用宏定义一个无限循环：

```c
#define LOOP 1

int main(void)
{
    while (LOOP) {
    }
}
```

| 文件 | 含义 |
| --- | --- |
| `loop.c` | 原始 C 源代码 |
| `loop.i` | 预处理输出，保留行号标记 |
| `loop-clean.i` | 使用 `-P` 生成的干净预处理输出 |
| `loop.s` | x86-64 汇编代码 |
| `loop.o` | 可重定位 ELF 目标文件 |
| `loop` | 可运行的 PIE ELF 可执行文件 |

## GCC 四阶段

```bash
# 1. 预处理：展开宏、处理头文件和条件编译
gcc -E loop.c -o loop.i
gcc -E -P loop.c -o loop-clean.i

# 2. 编译：C 代码转换为汇编
gcc -S -O0 loop-clean.i -o loop.s

# 3. 汇编：汇编代码转换为目标文件
gcc -c loop.s -o loop.o

# 4. 链接：生成可执行文件
gcc loop.o -o loop
```

也可以由 GCC 一次完成全部阶段：

```bash
gcc -Wall -Wextra loop.c -o loop
```

## 关键观察与结论

### 1. 预处理

在 `loop-clean.i` 中，`while (LOOP)` 已被替换为 `while (1)`。这说明 `#define LOOP 1` 在编译前就完成了宏展开。

### 2. 汇编

`loop.s` 的核心逻辑是：

```asm
.L2:
    jmp .L2
```

`jmp` 是无条件跳转，因此它准确对应 `while (1) {}` 的无限循环。`pushq %rbp` 和 `movq %rsp, %rbp` 是在关闭优化（`-O0`）时保留的典型函数栈帧初始化；`.cfi_*` 行用于调试器的栈回溯信息。

### 3. 目标文件与可执行文件的区别

```text
loop.o: ELF 64-bit LSB relocatable, x86-64
loop:   ELF 64-bit LSB pie executable, x86-64, dynamically linked
```

- `loop.o` 是 `REL (Relocatable file)`：入口地址为 `0x0`，没有 Program Header，不能由 Linux 直接运行。
- `loop` 是 PIE 可执行文件：入口地址为 `0x1040`，包含 14 个 Program Header，可被内核装载。
- `loop` 显示为 `DYN` 是 Debian 默认启用 PIE 的正常现象；它仍然是可执行程序。
- `loop` 使用 `/lib64/ld-linux-x86-64.so.2` 作为动态加载器，启动时负责装载共享库并完成运行前准备。

## 程序与进程

**程序（Program）**是磁盘上的静态指令与数据；本练习中的 `loop` 就是一个 ELF 程序文件。**进程（Process）**是 Linux 将程序加载到内存并执行后产生的运行实例，它拥有 PID、内存、CPU 调度状态和打开的文件等运行资源。

执行 `./loop &` 会创建一个进程；`kill "$pid"` 只结束该运行实例，`loop` 文件仍然保留。一个程序也可以同时启动多个独立进程，每个进程都有不同的 PID 和状态。

本例是无限循环，运行时会持续占用 CPU。安全观察方式：

```bash
./loop &
pid=$!

ps -p "$pid" -o pid,ppid,stat,%cpu,%mem,comm,args
top -b -n 1 -p "$pid"

# 观察结束后正常终止进程（默认发送 SIGTERM）
kill "$pid"
wait "$pid" 2>/dev/null || true
```

---

# GCC Practice Notes: From C Source Code to a Linux Process

**Date:** 2026-07-13  
**Environment:** WSL Debian, GCC 14.2.0, x86-64

## Goal

Understand and verify how a C program passes through the four GCC stages, becomes an executable ELF file, and then runs as a Linux process.

```text
loop.c → loop.i → loop.s → loop.o → loop (ELF) → Process
```

## Source Code and Generated Files

`loop.c` uses a macro to create an infinite loop:

```c
#define LOOP 1

int main(void)
{
    while (LOOP) {
    }
}
```

| File | Purpose |
| --- | --- |
| `loop.c` | Original C source code |
| `loop.i` | Preprocessor output with line markers |
| `loop-clean.i` | Cleaner preprocessor output generated with `-P` |
| `loop.s` | x86-64 assembly code |
| `loop.o` | Relocatable ELF object file |
| `loop` | Runnable PIE ELF executable |

## The Four GCC Stages

```bash
# 1. Preprocessing: expand macros, process headers and conditional compilation
gcc -E loop.c -o loop.i
gcc -E -P loop.c -o loop-clean.i

# 2. Compilation: convert C code to assembly
gcc -S -O0 loop-clean.i -o loop.s

# 3. Assembly: convert assembly to an object file
gcc -c loop.s -o loop.o

# 4. Linking: create the executable
gcc loop.o -o loop
```

GCC can also complete all stages in one command:

```bash
gcc -Wall -Wextra loop.c -o loop
```

## Key Observations

### 1. Preprocessing

In `loop-clean.i`, `while (LOOP)` becomes `while (1)`. This proves that `#define LOOP 1` is expanded before compilation.

### 2. Assembly

The core logic in `loop.s` is:

```asm
.L2:
    jmp .L2
```

`jmp` is an unconditional jump, so this exactly implements `while (1) {}`. `pushq %rbp` and `movq %rsp, %rbp` are the typical function stack-frame setup retained when optimization is disabled (`-O0`). The `.cfi_*` directives provide stack-unwinding information for debuggers.

### 3. Object File vs. Executable

```text
loop.o: ELF 64-bit LSB relocatable, x86-64
loop:   ELF 64-bit LSB pie executable, x86-64, dynamically linked
```

- `loop.o` is `REL` (a relocatable file): its entry point is `0x0` and it has no Program Header, so Linux cannot run it directly.
- `loop` is a PIE executable: it has entry point `0x1040` and 14 Program Headers, so the kernel can load it.
- `DYN` is normal for a Debian PIE executable; it is still runnable.
- `/lib64/ld-linux-x86-64.so.2` is the dynamic loader, which loads shared libraries and prepares the process to run.

## Program and Process

A **program** is static instructions and data stored on disk; `loop` is an ELF program file in this exercise. A **process** is a running instance created after Linux loads that program into memory. It owns runtime resources such as a PID, memory, CPU scheduling state, and open files.

Running `./loop &` creates a process. `kill "$pid"` terminates only that running instance; the `loop` file remains on disk. One program can create multiple independent processes, each with its own PID and state.

This example contains an infinite loop and continuously consumes CPU. Observe it safely with:

```bash
./loop &
pid=$!

ps -p "$pid" -o pid,ppid,stat,%cpu,%mem,comm,args
top -b -n 1 -p "$pid"

# End the process normally (sends SIGTERM by default)
kill "$pid"
wait "$pid" 2>/dev/null || true
```
