# Lab1 实验报告 — 探索最小内核与理解入口机制

> 实验目标：分析QEMU/OpenSBI到内核`entry.S`的启动链路，理解两条关键指令的作用，并用GDB验证关键节点。  
> 实验环境：QEMU `virt`平台 + OpenSBI固件，RISC-V 64位交叉工具链。

---

## 一、实验概览与执行流程

**整体启动流程**  
系统上电后从`0x1000`开始执行，很快跳转到`0x80000000`的OpenSBI固件，固件完成初始化后将控制权交给我们的内核（位于`0x80200000`）。内核从`entry.S`开始执行，先设置好栈，然后跳到C语言的`kern_init`函数，清空`.bss`段后通过SBI接口打印启动信息，表明内核已经成功运行。

**关键文件和组件**
- `kernel.ld`：链接脚本，规定了内核的装载地址和段布局
- `entry.S`：内核入口，搭建C语言运行环境
- `init.c`：包含`kern_init()`函数，进行基本初始化
- 一系列支持文件：提供字符输出能力

我在实验中重点分析了入口汇编与内存布局的关系，以及从硬件到C世界的过渡过程。

---

## 二、从地址布局到入口代码的分析

### 2.1 启动地址链路

从硬件角度看，QEMU启动时PC被设置为`0x1000`，经过简单的跳转后控制权到达OpenSBI（位于`0x80000000`）。OpenSBI完成基础初始化后，会将控制权转移到约定的内核基址`0x80200000`。

所以，我们的第一条内核指令必须恰好位于`0x80200000`这个物理地址，否则系统启动会失败。

### 2.2 链接脚本如何保证入口位置正确

查看`kernel.ld`可以发现三个关键设置：
1. 基地址设为`0x80200000`
2. 入口点指定为`kern_entry`符号
3. 巧妙地将`.text.kern_entry`段放在最前面

这确保了无论代码如何变化，`kern_entry`都会位于镜像最前端，正好对应OpenSBI跳转的目标地址。

### 2.3 入口处的两条核心指令

```asm
kern_entry:
    la sp, bootstacktop    // 设置栈指针
    tail kern_init         // 跳转到C入口
```

这两条看似简单的指令非常重要：

**关于`la sp, bootstacktop`：**
这条指令把栈顶地址加载到栈指针寄存器。为什么要这样做？因为C语言依赖栈来保存返回地址和局部变量，没有栈的话函数调用无法正常工作。栈指针指向高地址是因为RISC-V的栈是向下生长的，每次分配栈帧都会减小sp。

**关于`tail kern_init`：**
这是个尾调用，它不保存返回地址就直接跳转。为什么用尾调用而不是普通跳转？因为entry.S只是一个"一次性跳板"，设置好环境后就永不回头地进入C世界，没必要保存返回地址，也符合"不可返回"的语义。

### 2.4 启动栈的设计考虑

```asm
.section .data
.align PGSHIFT          // 按页对齐（4KB）
bootstack:              // 栈底（低地址）
    .space KSTACKSIZE   // 预留栈空间
bootstacktop:           // 栈顶（高地址）
```
后续代码是栈的内存分配。

为什么栈要放在`.data`段？因为启动早期还没有内存管理机制，预留一段静态内存作为栈是最简单可靠的方案。

为什么要页对齐？这有利于后期内存管理，比如设置保护页或实现栈保护机制，也满足了一些ABI的对齐要求。

### 2.5 从kern_init到屏幕输出的路径

`kern_init`先清空`.bss`段，然后通过层层封装的函数最终调用到`sbi_console_putchar`，再通过`ecall`指令陷入M模式，请求OpenSBI帮我们完成字符输出。这条调用链路验证了我们内核的基本I/O能力。

---

## 三、GDB 验证结果与分析（基于本次终端输出）

为验证 `entry.S` 的两项职责（**立栈**与**交棒**），我在 `kern_entry` / `kern_init` / `cprintf` / `sbi_console_putchar` 等关键点设置断点，并逐步观察寄存器与指令流。

### 3.1 成功进入 C 端入口 `kern_init`
命中 `kern_init` 的断点：
```gdb
Breakpoint 1, kern_init () at kern/init/init.c:8
8           memset(edata, 0, end - edata);
```
**结论**：说明 `entry.S` 的 `tail kern_init` 已完成“只去不回”的交接，控制权从汇编世界成功转入 C 世界。

---

### 3.2 启动栈初始化正确（`sp == bootstacktop`）
现场查询启动栈符号与寄存器：
```gdb
(gdb) p/x &bootstack
$1 = 0x80201000
(gdb) p/x &bootstacktop
$2 = 0x80203000
(gdb) info reg sp
sp             0x80203000       0x80203000 <SBI_CONSOLE_PUTCHAR>
(gdb) p/x &bootstacktop - &bootstack
$3 = 0x2000
```
**解读**：
- `sp = 0x80203000 = &bootstacktop`，说明 `la sp, bootstacktop` 生效；  
- `bootstacktop - bootstack = 0x2000 (= 8 KiB)`，与 `KSTACKSIZE` 一致；  
- GDB 将 `sp` 地址标注为 `<SBI_CONSOLE_PUTCHAR>` 仅是“最近符号”提示；栈从高地址向低地址增长，函数序言会先 `addi sp, sp, -N` 再写入，不会踩该符号，**属于正常现象**。

---

### 3.3 `kern_init` 函数序言与参数准备（清 `.bss` 前）
查看 `kern_init` 附近的指令：
```gdb
(gdb) x/6i $pc
=> 0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
   0x8020001a <kern_init+16>:   addi    sp,sp,-16
   0x8020001c <kern_init+18>:   li      a1,0
   0x8020001e <kern_init+20>:   sub     a2,a2,a0
   0x80200020 <kern_init+22>:   sd      ra,8(sp)
```
并尝试回溯：
```gdb
(gdb) where
#0  0x0000000080200012 in kern_init () at kern/init/init.c:8
#1  0x0000000080000a02 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
```
**解读**：
- `addi sp, sp, -16`、`sd ra, 8(sp)` 等是标准的函数序言；  
- `a0/a1/a2` 的设置对应 `memset(edata, 0, end-edata)` 的参数准备；  
- 回溯出现 “corrupt stack?” 是因为入口使用 **`tail`**（尾调用不写返回地址），GDB 试图回溯上一帧时落到 OpenSBI 区域，**这是预期现象而非错误**。

---

### 3.4 输出链路畅通（`cprintf → sbi_console_putchar → ecall`）
命中 `cprintf`：
```gdb
Breakpoint 2, cprintf (fmt=fmt@entry=0x802004c8 "%s\n\n")
    at kern/libs/stdio.c:40
40          va_start(ap, fmt);
(gdb) x/6i $pc
=> 0x80200054 <cprintf>:        addi    sp,sp,-96
   0x80200056 <cprintf+2>:      addi    t1,sp,40
   0x8020005a <cprintf+6>:      sd      a1,40(sp)
   0x8020005c <cprintf+8>:      sd      a2,48(sp)
   0x8020005e <cprintf+10>:     sd      a3,56(sp)
   0x80200060 <cprintf+12>:     mv      a2,a0
```
继续到 SBI 字符输出处：
```gdb
Breakpoint 3, sbi_console_putchar (ch=40 '(') at libs/sbi.c:33
33          sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);
```
**解读**：
- `cprintf` 进入后建立自己的栈帧，准备可变参；  
- 在 `sbi_console_putchar` 命中时 `ch=40 '('`，正是启动字符串 **"(THU.CST) os is loading ..."** 的第一个字符；  
- 随后 `sbi_call` 通过设置寄存器并执行 `ecall` 进入 M 模式，由 OpenSBI 完成真正的字符输出。

---

### 3.5 小结
- **交棒成功**：命中 `kern_init`，`tail kern_init` 工作正常；  
- **立栈成功**：`sp==bootstacktop`，栈大小 8 KiB，页对齐 & ABI 对齐均满足；  
- **C 环境就绪**：`kern_init` 序言与参数设置正确；  
- **打印链路可达**：`cprintf → sbi_console_putchar → ecall` 全链路触发；  
- **回溯提示可解释**：因尾调用不写返回地址，GDB 回溯出现 OpenSBI 地址属预期。

**结论**：本次 GDB 观察与理论分析完全一致，`entry.S` 在“最小内核”中顺利完成 **“搭栈桥 → 交接到 C”** 的使命。


我用GDB设置了几个关键断点，具体观察启动过程中的各个环节：

### 3.1 成功进入kern_init

设置断点后，GDB显示正确停在了`kern_init`函数开头，证明控制权成功从entry.S转移到了C代码。

### 3.2 栈设置正确

通过检查寄存器和符号值，我确认：
- 栈指针确实指向了`bootstacktop`
- 栈大小为8KB（`bootstacktop - bootstack = 0x2000`）
- 栈位置符合预期

### 3.3 函数调用正常

观察指令序列，可以看到正常的函数序言，包括保存返回地址、设置参数等，说明C环境已经准备就绪。

### 3.4 输出链路畅通

断点成功在`cprintf`和`sbi_console_putchar`处触发，验证了整个输出路径是正常工作的。

这些验证结果表明我们的启动机制设计是正确的，内核能够顺利从汇编世界过渡到C世界，并具备基本的输出能力。

---

## 四、练习问题回答

**问题1：`la sp, bootstacktop`的作用和目的是什么？**  

这条指令将栈指针设置到预留栈空间的高地址端。没有这一步，后续的C代码将无法正常运行，因为函数调用需要使用栈来存储返回地址和局部变量。可以说，这一步是从汇编过渡到C语言的必要桥梁。

**问题2：`tail kern_init`的作用和目的是什么？**  

这条指令以尾调用方式跳转到C入口函数，它不会保存返回地址。这符合我们的设计意图——入口汇编代码只是一次性的跳板，完成基本设置后就应该"永不回头"地进入C世界进行更复杂的初始化工作。

---

## 五、关键知识点与操作系统原理对照

| 实验中的知识点 | 对应的OS原理 | 关系和意义 |
|---|---|---|
| 链接脚本与地址设置 | 程序装载与地址空间 | 确定内核在物理内存中的位置，这是后续虚拟内存管理的基础 |
| 栈设置与函数调用 | 运行时环境与调用约定 | 搭建C语言执行环境，为后续更复杂的系统功能奠定基础 |
| 页对齐的内存布局 | 内存管理与保护机制 | 符合硬件要求，便于后期实现分页和内存保护 |
| 特权级切换 | CPU特权级与安全隔离 | 通过ecall在M模式和S模式间切换，是系统调用机制的雏形 |

---

## 六、思考与拓展

虽然这个实验只是OS的一小步，但它涉及了很多重要概念的基础。后续实验还会深入探索：
- 中断和异常处理
- 物理内存管理和页表机制
- 进程与线程模型
- 系统调用实现
- 文件系统与设备管理

这些主题将在后续实验中逐步展开。

---

## 七、结语

通过这次实验，我深入理解了操作系统启动的第一步——从固件到内核的转交过程。`entry.S`虽然只有短短几行，却承担了至关重要的角色：设置好栈环境，并把控制权交给C代码。链接脚本精确控制了内核的装载位置，确保OpenSBI能顺利找到我们的内核入口。通过GDB调试，我验证了整个启动过程中的关键节点都符合预期。

这个最小内核虽然功能简单，但它确实"活"了起来，成功打印出了启动信息，标志着我们迈出了操作系统开发的第一步。
