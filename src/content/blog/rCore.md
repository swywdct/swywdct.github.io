---
title: 'rCore 学习笔记'
description: '记录 rCore 实验中从启动入口、批处理到系统调用的实现过程。'
pubDate: '2026-07-22'
---

## lab1

1. 汇编入口设置栈指针，跳转到 rust\_main
2. 清 BSS 段、初始化日志、打印 Hello, world!
3. 通过 SBI 调用关机

### sbi.rs
获取、接受字符；关机；
```rust
#![allow(unused)]
#[allow(deprecated)]
//包装
sbi_rt::legacy::console_putchar(c);
sbi_rt::legacy::console_getchar()
sbi_rt::{NoReason, Shutdown, SystemFailure, system_reset};
```
### logging.rs
日志系统
```rust
impl Log for SimpleLogger{

}
let color = match record.level() {
	Level::Error => 31, // Red
    Level::Warn => 93,  // BrightYellow
    Level::Info => 34,  // Blue
    Level::Debug => 32, // Green
    Level::Trace => 90, // BrightBlack
};
```
### lang\_items.rs
```rust
use core::panic::PanicInfo;
fn panic(info: &PanicInfo) -> ! {
	//处理

	关机
}
```
### entry.asm
```rust
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main //进入rust_main函数（而非main）

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```
### console.rs
控制台内 打印自定义宏
```rust
/// 打印 自定义宏
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}
```

### main.rs
```rust
#![deny(missing_docs)]
//所有公开项必须有文档注释，否则编译错误
#![deny(warnings)]
//所有警告视为错误
#![no_std]
//不使用标准库（裸机环境）
#![no_main]
//不使用标准 main 函数（自定义入口）

pub fn clear_bss() {
    unsafe extern "C" {  ///使用不安全的C函数，声明startbss ,endbbs为safe
        safe fn sbss(); 
        safe fn ebss(); 
    }
    (linker_symbol_addr!(sbss)..linker_symbol_addr!(ebss))
        .for_each(|a| unsafe { (a as *mut u8).write_volatile(0) }); //全部赋值为0
}

#[unsafe(no_mangle)]
pub fn rust_main() -> ! {
    unsafe extern "C" {
        safe fn stext(); // begin addr of text segment
        safe fn etext(); // end addr of text segment
        safe fn srodata(); // start addr of Read-Only data segment
        safe fn erodata(); // end addr of Read-Only data ssegment
        safe fn sdata(); // start addr of data segment
        safe fn edata(); // end addr of data segment
        safe fn sbss(); // start addr of BSS segment
        safe fn ebss(); // end addr of BSS segment
        safe fn boot_stack_lower_bound(); // stack lower bound
        safe fn boot_stack_top(); // stack top
    }
    clear_bss(); //清空bbs段
    logging::init();  //日志初始化
    println!("[kernel] Hello, world!  (can you see it?)");
    trace!(
        "[kernel] .text [{:#x}, {:#x})",
        linker_symbol_addr!(stext),
        linker_symbol_addr!(etext)
    );
    debug!(
        "[kernel] .rodata [{:#x}, {:#x})",
        linker_symbol_addr!(srodata),
        linker_symbol_addr!(erodata)
    );
    info!(
        "[kernel] .data [{:#x}, {:#x})",
        linker_symbol_addr!(sdata),
        linker_symbol_addr!(edata)
    );
    warn!(
        "[kernel] boot_stack top=bottom={:#x}, lower_bound={:#x}",
        linker_symbol_addr!(boot_stack_top),
        linker_symbol_addr!(boot_stack_lower_bound)
    );
    error!(
        "[kernel] .bss [{:#x}, {:#x})",
        linker_symbol_addr!(sbss),
        linker_symbol_addr!(ebss)
    );

    // CI autotest success: sbi::shutdown(false)
    // CI autotest failed : sbi::shutdown(true)
    sbi::shutdown(false)
}

```

## lab2

1.构造包含操作系统内核和多个应用程序的单一执行程序
2.通过批处理支持多个程序的自动加载和运行
3.操作系统利用硬件特权级机制，实现对操作系统自身的保护
4.实现特权级的穿越
5.支持跨特权级的系统调用功能

#### 文件清单

| 文件路径                        | 作用                                                                                    |
| --------------------------- | ------------------------------------------------------------------------------------- |
| `.cargo/config.toml`        | Cargo 构建配置，指定 `riscv64gc-unknown-none-elf` 目标及链接脚本等 rustflags                         |
| `Cargo.toml`                | 项目清单，依赖 `riscv`、`lazy_static`、`log`、`sbi-rt`                                          |
| `Cargo.lock`                | 依赖版本锁定文件                                                                              |
| `build.rs`                  | 构建脚本，读取 `../user/` 下的 app 二进制，自动生成 `src/link_app.S` 嵌入各 app                           |
| `Makefile`                  | 构建自动化，含 `build`/`run`/`debug`/`clean` 等目标，QEMU 启动入口                                   |
| `scripts/qemu-ver-check.sh` | 检查 QEMU 版本 \>= 7 的脚本                                                                  |
| `src/main.rs`               | **内核 Rust 入口**。清除 BSS、初始化日志/trap/batch、运行首个 app                                       |
| `src/entry.asm`             | **汇编入口 `_start`**，设置启动栈，跳转 `rust_main`                                                |
| `src/link_app.S`            | 由 `build.rs` 自动生成，`.incbin` 嵌入各用户 app 二进制，提供 `_num_app` 等符号                           |
| `src/linker-qemu.ld`        | 链接脚本，入口 `_start`，基址 `0x80200000`，定义 text/rodata/data/bss 段                            |
| `src/batch.rs`              | **批处理子系统**。管理 `AppManager`，实现 `load_app()` 和 `run_next_app()`                         |
| `src/console.rs`            | 控制台输出，实现 `print!`/`println!` 宏，底层调用 SBI `console_putchar`                             |
| `src/sbi.rs`                | SBI 调用封装：`console_putchar`、`console_getchar`、`shutdown`                               |
| `src/logging.rs`            | 内核日志，彩色 ANSI 输出，支持 TRACE/DEBUG/INFO/WARN/ERROR 级别                                     |
| `src/lang_items.rs`         | panic 处理器（`#![no_std]` 必需），记录信息后调用 `shutdown`                                         |
| `src/trap/mod.rs`           | **Trap 处理模块**。设置 `stvec` 指向 `__alltraps`，分发 ecall/StoreFault/IllegalInstruction 等     |
| `src/trap/context.rs`       | `TrapContext` 结构体，保存 32 个通用寄存器 + `sstatus`/`sepc`，提供 `app_init_context()`             |
| `src/trap/trap.S`           | **汇编 trap 入口/恢复**。`__alltraps` 保存寄存器调用 `trap_handler`；`__restore` 恢复寄存器后 `sret` 返回用户态 |
| `src/syscall/mod.rs`        | 系统调用分发，匹配 `SYSCALL_WRITE(64)` 和 `SYSCALL_EXIT(93)`                                    |
| `src/syscall/fs.rs`         | `sys_write` 实现，fd=1(stdout) 时输出字符串                                                    |
| `src/syscall/process.rs`    | `sys_exit` 实现，打印退出码后加载下一个 app                                                         |
| `src/sync/mod.rs`           | 同步模块入口，重新导出 `UPSafeCell`                                                              |
| `src/sync/up.rs`            | `UPSafeCell<T>` — 单核环境下的 `RefCell` 封装，用于 lazy\_static 全局可变状态                          |

## 目录结构

```
os/
├── .cargo/
│   └── config.toml
├── scripts/
│   └── qemu-ver-check.sh
├── src/
│   ├── batch.rs
│   ├── console.rs
│   ├── entry.asm
│   ├── lang_items.rs
│   ├── link_app.S          (自动生成)
│   ├── linker-qemu.ld
│   ├── logging.rs
│   ├── main.rs
│   ├── sbi.rs
│   ├── sync/
│   │   ├── mod.rs
│   │   └── up.rs
│   ├── syscall/
│   │   ├── fs.rs
│   │   ├── mod.rs
│   │   └── process.rs
│   └── trap/
│       ├── context.rs
│       ├── mod.rs
│       └── trap.S
├── target/                  (构建产物，自动生成)
├── Cargo.toml
├── Cargo.lock
├── build.rs
└── Makefile
```

## 执行流程

```
Bootloader → _start → rust_main
                        ├── clear_bss()
                        ├── logging::init()
                        ├── trap::init()
                        └── batch::run_next_app()
                              ├── load_app()
                              └── __restore → 用户态
                                    │
                                    ▼
                              app 执行 → ecall
                                    │
                                    ▼
                              __alltraps → trap_handler
                                    │
                                    ▼
                              syscall (write/exit)
                                    │
                                    ▼
                              __restore → 下一个 app (或 shutdown)
```











#### 应用程序设计
应用程序、用户库（包括入口函数、初始化函数、I/O 函数和系统调用接口等多个 rs 文件组成）放在项目根目录的 user 目录下，它和第一章的裸机应用不同之处主要在项目的目录文件结构和内存布局上：
user/src/bin/\*.rs ：各个应用程序
user/src/\*.rs ：用户库（包括入口函数、初始化函数、I/O 函数和系统调用接口等）
user/src/linker.ld ：应用程序的内存布局说明。

将程序的起始物理地址调整为 0x80400000 ，三个应用程序都会被加载到这个物理地址上运行；

将 \_start 所在的 .text.entry 放在整个程序的开头，也就是说批处理系统只要在加载之后跳转到 0x80400000 就已经进入了 用户库的入口点，并会在初始化之后跳转到应用程序主逻辑；
提供了最终生成可执行文件的 .bss 段的起始和终止地址，方便 clear\_bss 函数使用。\_

#### 特权级机制
#### 实现应用程序 - 切换用户态和内核态
#### 实现批处理操作系统
静态绑定：通过一定的编程技巧，把多个应用程序代码和批处理操作系统代码“绑定”在一起。
动态加载：基于静态编码留下的“绑定”信息，操作系统可以找到每个应用程序文件二进制代码的起始地址和长度，并能加载到内存中运行。

#### syscall.rs
```rust
//向文件描述符写入数据、退出当前进程。
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2], //三个参数
            in("x17") id	   //系统调用号		
        );
    }
    ret //输入返回共用寄存器
}
//写入
//args[]: fd文件描述符（1 = stdout）
// buffer.as_ptr缓冲区指针 buffer.len()缓冲区长度

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(exit_code: i32) -> isize {
    syscall(SYSCALL_EXIT, [exit_code as usize, 0, 0])
}
sys_exit(0);  // 正常退出
sys_exit(-1); // 异常退出
```



## Lib3

```rust
./os/src
Rust        18 Files   511 Lines
Assembly     3 Files    82 Lines

├── bootloader
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs
│   ├── Cargo.toml
│   ├── Makefile
│   └── src
│       ├── batch.rs(移除：功能分别拆分到 loader 和 task 两个子模块)
│       ├── config.rs(新增：保存内核的一些配置)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S
│       ├── linker-qemu.ld
│       ├── loader.rs(新增：将应用加载到内存并进行管理)
│       ├── main.rs(修改：主函数进行了修改)
│       ├── sbi.rs(修改：引入新的 sbi call set_timer)
│       ├── sync
│       │   ├── mod.rs
│       │   └── up.rs
│       ├── syscall(修改：新增若干 syscall)
│       │   ├── fs.rs
│       │   ├── mod.rs
│       │   └── process.rs
│       ├── task(新增：task 子模块，主要负责任务管理)
│       │   ├── context.rs(引入 Task 上下文 TaskContext)
│       │   ├── mod.rs(全局任务管理器和提供给其他模块的接口)
│       │   ├── switch.rs(将任务切换的汇编代码解释为 Rust 接口 __switch)
│       │   ├── switch.S(任务切换的汇编代码)
│       │   └── task.rs(任务控制块 TaskControlBlock 和任务状态 TaskStatus 的定义)
│       ├── timer.rs(新增：计时器相关)
│       └── trap
│           ├── context.rs
│           ├── mod.rs(修改：时钟中断相应处理)
│           └── trap.S
├── README.md
├── rust-toolchain
└── user
    ├── build.py(新增：使用 build.py 构建应用使得它们占用的物理地址区间不相交)
    ├── Cargo.toml
    ├── Makefile(修改：使用 build.py 构建应用)
    └── src
        ├── bin(修改：换成第三章测例)
        │   ├── 00power_3.rs
        │   ├── 01power_5.rs
        │   ├── 02power_7.rs
        │   └── 03sleep.rs
        ├── console.rs
        ├── lang_items.rs
        ├── lib.rs
        ├── linker.ld
        └── syscall.rs
```
