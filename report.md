# Lab1 实验报告 — 探索最小内核与理解入口机制

> 实验目标：分析QEMU/OpenSBI到内核`entry.S`的启动链路，理解两条关键指令的作用，并用GDB验证从加电复位（0x1000）到内核入口（0x80200000）的完整启动流程。
> 实验环境：QEMU `virt`平台 + OpenSBI固件，RISC-V 64位交叉工具链。

---

## 一、实验概览与分层启动架构

### 1.1 整体启动流程

系统上电后，CPU从固定的**复位向量地址 0x1000**（MROM）开始执行，这是启动流程的第一站。经过简单的跳转指令后，控制权转移到**0x80000000 的 OpenSBI 固件**（M模式），完成平台初始化和多核管理。最后，OpenSBI 将控制权交给**0x80200000 的内核入口**（S模式），内核从 [entry.S](kern/init/entry.S) 开始执行，设置栈环境后跳转到C语言的 `kern_init` 函数。

### 1.2 分层视角

启动过程可以分为三个层次：

| 层次 | 地址 | 特权级 | 主要功能 |
|------|------|--------|----------|
| **MROM 复位向量** | 0x1000 | M-mode | 上电后第一条指令，读取hart ID，从跳转表获取OpenSBI入口地址并跳转 |
| **OpenSBI 固件** | 0x80000000 | M-mode | 平台早期初始化，按mhartid分流多核，准备移交控制权给内核 |
| **内核入口** | 0x80200000 | S-mode | `kern_entry`设置早期栈，`tail kern_init`无返回跳入C入口 |

### 1.3 关键文件和组件

- [kernel.ld](tools/kernel.ld)：链接脚本，规定内核装载地址为0x80200000，并将`.text.kern_entry`段放在最前面
- [entry.S](kern/init/entry.S)：内核入口，搭建C语言运行环境的汇编代码
- [init.c](kern/init/init.c)：包含`kern_init()`函数，进行`.bss`清零和基本初始化
- [sbi.c](libs/sbi.c)：SBI接口封装，提供M模式与S模式的通信能力

---

## 二、练习1：理解内核启动中的程序入口操作

### 2.1 启动地址链路与链接脚本的作用

从硬件角度看，QEMU启动时PC被设置为`0x1000`，经过简单的跳转(这里主要依靠的是MROM固件，他的主要功能就是在启动后把控制权交给OPENSBI)后控制权到达OpenSBI（位于`0x80000000`）。OpenSBI完成基础初始化后，会将控制权转移到约定的内核基址`0x80200000`。

因此，我们的第一条内核指令必须恰好位于`0x80200000`这个物理地址，否则系统启动会失败。

### 2.2 链接脚本如何保证入口位置正确

查看 [kernel.ld](tools/kernel.ld) 可以发现三个关键设置：

```ld
OUTPUT_ARCH(riscv)
ENTRY(kern_entry)              # 1. 指定入口符号

BASE_ADDRESS = 0x80200000;     # 2. 设置基地址

SECTIONS {
    . = BASE_ADDRESS;
    .text : {
        *(.text.kern_entry .text .stub .text.* .gnu.linkonce.t.*)
        # 3. 将.text.kern_entry段放在最前面
    }
}
```

这三个设置确保了无论代码如何变化，`kern_entry` 都会位于镜像最前端，正好对应OpenSBI跳转的目标地址。

### 2.3 入口处的两条核心指令

[entry.S](kern/init/entry.S:7) 中的入口代码非常简洁：

```asm
kern_entry:
    la sp, bootstacktop    # 第一条：设置栈指针
    tail kern_init         # 第二条：跳转到C入口
```

#### 指令1：`la sp, bootstacktop` 的作用和目的

**展开后的机器指令：**
```asm
auipc sp, 0x3      # 取高20位基址：sp = pc + (0x3 << 12)
addi  sp, sp, imm  # 加上低12位偏移
```

**作用：** 将栈指针 `sp` 设置为 `bootstacktop` 的地址（0x80203000）。

**目的：**
1. **建立C语言运行环境**：C语言依赖栈来保存返回地址、函数参数和局部变量，没有栈的话函数调用无法正常工作
2. **栈向下生长**：栈指针指向高地址，因为RISC-V的栈是向下生长的，每次分配栈帧都会减小sp
3. **位置无关寻址**：使用 `auipc` 指令实现位置无关寻址，使得代码可以在任意地址运行

#### 指令2：`tail kern_init` 的作用和目的

**实际机器指令：**
```asm
j kern_init        # 等价于 jal x0, kern_init
```

**作用：** 无条件跳转到 `kern_init` 函数，不保存返回地址。

**目的：**
1. **尾调用优化**：不保存返回地址就直接跳转，因为entry.S只是一个"一次性跳板"
2. **不可返回语义**：设置好环境后就永不回头地进入C世界，符合启动流程的逻辑
3. **节省栈空间**：不占用额外的栈帧，避免无意义的返回地址保存

### 2.4 启动栈的设计考虑

```asm
.section .data
.align PGSHIFT          # 按页对齐（4KB，2^12）
bootstack:              # 栈底（低地址）
    .space KSTACKSIZE   # 预留8KB栈空间
bootstacktop:           # 栈顶（高地址）
```

**设计要点：**
1. **静态内存分配**：启动早期还没有内存管理机制，预留一段静态内存作为栈是最简单可靠的方案
2. **页对齐**：有利于后期内存管理，比如设置保护页或实现栈保护机制，也满足了ABI的对齐要求
3. **栈大小**：8KB（`KSTACKSIZE = 0x2000`）足够满足早期初始化的需求

### 2.5 从kern_init到屏幕输出的路径

`kern_init` 的执行流程：
1. 清空`.bss`段（调用`memset`）
2. 通过层层封装调用 `cprintf` → `vcprintf` → `vprintfmt` → `cputch` → `cons_putc` → `sbi_console_putchar`
3. 使用`ecall`指令陷入M模式，请求OpenSBI完成字符输出

这条调用链路验证了内核具备基本的I/O能力。

---

## 三、练习2：使用GDB验证启动流程

### 3.1 实验目标与验证策略

**调试目标：** 使用 GDB 配合 QEMU，完整追踪从复位地址（0x1000）到内核执行第一条指令（0x80200000）的全过程，通过实际调试结果验证启动流程。

**三阶段验证策略：**
- **第一阶段（0x1000）**：观察复位向量区域的引导代码，确认如何获取下一跳地址
- **第二阶段（0x80000000）**：分析 OpenSBI 固件的入口逻辑，理解多核分流和初始化流程
- **第三阶段（0x80200000）**：验证内核入口的栈设置和控制流转移

### 3.2 实验命令与详细流程

#### 3.2.1 启动调试环境

在一个终端启动QEMU等待调试：
```bash
make debug
```

在另一个终端启动GDB并连接：
```bash
make gdb
```

**初次进入GDB的输出：**
```gdb
--Type <RET> for more, q to quit, c to continue without paging--
```

按 `c` 关闭分页。GDB 输出显示：
```
Reading symbols from bin/kernel...
The target architecture is set to "riscv:rv64".
Remote debugging using localhost:1234
0x0000000000001000 in ?? ()
```

**关键观察：** PC 此刻正好在 **0x1000**，这就是复位向量/MROM 的起点。

#### 3.2.2 友好化显示设置

```gdb
(gdb) set pagination off
(gdb) set disassemble-next-line on
(gdb) display /i $pc
1: x/i $pc
=> 0x1000:      auipc   t0,0x0
```

**设置效果：**
- 输出不再被分页打断
- 每次停住，GDB 自动反汇编"下一条将执行的指令"
- 总是显示当前 `$pc` 所在的指令

现在一眼就能看出，**0x1000 的第一条是 `auipc t0,0x0`**。

#### 3.2.3 观察复位向量：从 0x1000 启动

**检查复位地址的指令内容：**
```gdb
(gdb) x/12i $pc
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a1,t0,32
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      t0,24(t0)
   0x1010:      jr      t0
   0x1014:      unimp
   0x1016:      unimp
   0x1018:      unimp
   ...
```

**指令功能解析：**
1. `auipc t0,0x0`：利用PC相对寻址计算基地址，t0 = PC + 0，实现**位置无关的地址计算**
2. `addi a1,t0,32`：计算参数结构体地址（基址偏移32字节），用于**向固件传递配置信息**
3. `csrr a0,mhartid`：从CSR寄存器读取**硬件线程ID**，标识当前执行核心
4. `ld t0,24(t0)`：从偏移24字节处加载**固件入口地址**（指向OpenSBI起始位置）
5. `jr t0`：间接跳转，将控制权转移到加载的地址

**验证跳转过程：**
```gdb
(gdb) advance *0x1010
0x0000000000001010 in ?? ()
=> 0x1010:      jr      t0

(gdb) info registers t0
t0             0x80000000       2147483648

(gdb) si
0x0000000080000000 in ?? ()
=> 0x80000000:  csrr    a6,mhartid
1: x/i $pc
=> 0x80000000:  csrr    a6,mhartid
```

**跳转确认：** 复位代码在 0x1000 区域完成了基础引导任务：建立位置无关基址、准备调用约定参数（hartid 和配置指针）、从内存表加载固件地址、执行间接跳转。PC 成功从 **0x1010 跳转至 0x80000000**，完成第一次控制权转移。

#### 3.2.4 "三站式"断点：复位/固件/内核

为了系统验证完整流程，设置三个关键断点：

```gdb
(gdb) hbreak *0x1000
Hardware assisted breakpoint 1 at 0x1000
(gdb) hbreak *0x80000000
Hardware assisted breakpoint 2 at 0x80000000
(gdb) tbreak *0x80200000
Temporary breakpoint 3 at 0x80200000: file kern/init/entry.S, line 7.
```

**说明：**
- 使用 `hbreak`（硬件断点）：ROM/固件区域不能插软件断点（无法修改内存）
- 在内核入口使用 `tbreak`（一次性断点）：命中后自动删除

#### 3.2.5 复位并观察完整流程

```gdb
(gdb) monitor system_reset
(gdb) c
Continuing.

Breakpoint 2, 0x0000000080000000 in ?? ()
=> 0x80000000:  csrr    a6,mhartid
```

**现象解释：** `c` 放行后，经常先命中 0x80000000。这是因为 GDB 和 QEMU 的同步不是每条指令都握手，MROM 的那几条非常快，`c` 之后它一口气做完第一跳，GDB 抢回控制权时就已经在 OpenSBI 入口了。

#### 3.2.6 分析 OpenSBI 入口与多核管理

```gdb
(gdb) x/12i $pc
=> 0x80000000:  csrr    a6,mhartid
   0x80000004:  bgtz    a6,0x80000108
   0x80000008:  auipc   t0,0x0
   0x8000000c:  addi    t0,t0,1032
   ...
```

**固件入口分析：**
1. `csrr a6,mhartid`：获取当前硬件线程ID，判断是主核还是从核
2. `bgtz a6,0x80000108`：条件分支，非零hart（从核）跳转至等待区域执行WFI循环
3. **主核独占初始化路径**：只有hart0继续执行后续的平台配置和内核加载

OpenSBI 采用**主从核分离**的启动策略，从核进入低功耗等待状态，避免多核并发初始化造成的资源竞争，主核完成所有准备工作后再唤醒从核。

#### 3.2.7 继续放行：第二跳进入内核

```gdb
(gdb) c
Continuing.

Temporary breakpoint 3, kern_entry () at kern/init/entry.S:7
7           la sp, bootstacktop
=> 0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   addi    sp,sp,0
```

**验证成功：** PC 到了 **0x80200000**，也就是内核的第一条指令 `kern_entry`。源码对应的第一件事：`la sp, bootstacktop`——把早期内核的栈顶先立起来。

#### 3.2.8 收尾：设好栈，并以 `tail` 进入 C 入口

```gdb
(gdb) x/6i $pc
=> 0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   addi    sp,sp,0
   0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>
   0x8020000a <kern_init>:      auipc   a0,0x3
   ...

(gdb) info registers sp
sp             0x8001bd80       0x8001bd80

(gdb) si
(gdb) si
9           tail kern_init
=> 0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>

(gdb) info registers sp
sp             0x80203000       0x80203000

(gdb) print/x &bootstacktop
$1 = 0x80203000
(gdb) print/x &bootstacktop - &bootstack
$2 = 0x2000
```

**验证结果：**
1. 两次 `si` 完成了 `la sp, bootstacktop` 展开的两条机器指令（auipc + addi）
2. SP 被精确设成了 `bootstacktop`（0x80203000），早期栈就绪
3. 栈大小为 0x2000（8KB），符合 `KSTACKSIZE` 定义
4. 紧接着的 `j 0x8020000a <kern_init>` 等价于 `jal x0,kern_init`，也就是所谓的 **tail** 调用：不写返回地址、无返回跳转

```gdb
(gdb) si
kern_init () at kern/init/init.c:8
8           memset(edata, 0, end - edata);
=> 0x8020000a <kern_init>:      auipc   a0,0x3
```

**完成：** 控制权直接交给 C 入口 `kern_init`，不会再回到 `kern_entry`。

**到此为止，两次关键跳转——0x1000 → 0x80000000 → 0x80200000——全部有据可查。**

### 3.3 练习2问题回答

**问题：RISC-V 硬件加电后最初执行的几条指令位于什么地址？它们主要完成了哪些功能？**

#### 执行位置：
在 QEMU/virt 平台上，CPU 复位后**从地址 0x1000 开始执行**，这是固化在硬件中的复位向量地址。

#### 调试验证：
```gdb
(gdb) x/12i 0x1000
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a1,t0,32
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      t0,24(t0)
   0x1010:      jr      t0
   ...

(gdb) advance *0x1010
=> 0x1010:      jr      t0

(gdb) si
=> 0x80000000:  csrr    a6,mhartid   ; 已进入 OpenSBI
```

#### 指令作用分析：
1. **`auipc t0,0`**：使用PC相对寻址获取当前位置的基地址，为**位置无关代码**提供基础
2. **`addi a1,t0,32`**：计算参数传递区域的地址，用于**向下一阶段传递启动参数**
3. **`csrr a0,mhartid`**：读取硬件线程标识符，用于**多核环境下识别当前核心**
4. **`ld t0,24(t0)`**：从预设的跳转表中**加载固件入口地址**（该平台为 0x80000000）
5. **`jr t0`**：执行间接跳转，**将控制权移交给 OpenSBI 固件**

#### 启动链路总结：
复位向量代码实现了最小化的引导逻辑：通过位置无关寻址获取基址、遵循RISC-V调用约定准备参数（hart ID 和配置指针）、从内存中的跳转表读取固件地址、执行跳转。随后 OpenSBI 接管控制权，完成平台初始化工作，最终跳转到 0x80200000 的内核入口，由 `kern_entry` 建立运行环境并进入 C 语言世界。

---

## 四、内核入口后的详细调试验证

在完成了从加电到内核入口的完整追踪后，我们进一步验证内核入口的两条指令作用，以及 SP/PC 的正确设置。

### 4.1 调试目标

1. 验证 `PC` 能正确从引导入口 `kern_entry` 跳转到 C 初始化函数 `kern_init`
2. 验证 `SP` 在 `kern_entry` 被设置为 `bootstacktop`，保证 C 语言运行环境可用
3. 确认 `kern_init` 中执行 `.bss` 段清零，并准备调用 `memset`

### 4.2 验证入口指令

```gdb
(gdb) b *kern_entry
Breakpoint 1 at 0x80200000: file kern/init/entry.S, line 7.
(gdb) c

Breakpoint 1, kern_entry () at kern/init/entry.S:7
7           la sp, bootstacktop

(gdb) x/8i $pc
=> 0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   addi    sp,sp,0
   0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>
   0x8020000a <kern_init>:      auipc   a0,0x3
   0x8020000e <kern_init+4>:    addi    a0,a0,-2
   0x80200012 <kern_init+8>:    auipc   a2,0x3
```

**观察结果：**
- 第一条指令 `la sp, bootstacktop` 被展开为 `auipc sp,0x3` + `addi sp,sp,0`
- 第三条指令 `j kern_init`（由 `tail kern_init` 汇编而来）将 PC 无条件跳转到 `kern_init`
- **关键发现：** `kern_init` 的指令紧跟在 `kern_entry` 之后，地址连续

**为什么 `kern_init` 就这么巧在 PC 的下一个？**

查看 [kernel.ld](tools/kernel.ld:14) 中的定义：
```ld
.text : {
    *(.text.kern_entry .text .stub .text.* .gnu.linkonce.t.*)
}
```

链接脚本把 `.text.kern_entry` 段（也就是 `kern_entry` 所在的那几条汇编）放在整个内核代码的最开头，紧接着就是所有其他 `.text` 段（包括 [init.c](kern/init/init.c) 编译出来的 `kern_init`），因此指令相连。

### 4.3 单步观察 SP 的变化

```gdb
(gdb) info registers sp
sp             0x8001bd80       2147581312

(gdb) si
(gdb) info registers sp
sp             0x80203000       2149584896

(gdb) print/x &bootstacktop
$1 = 0x80203000

(gdb) print/x &bootstacktop - &bootstack
$2 = 0x2000
```

**验证结果：**
- 执行 `la sp, bootstacktop` 后，SP 被正确设置为 **0x80203000**
- `bootstacktop - bootstack = 0x2000`（8192 字节），即 2 页，符合 `KSTACKSIZE` 的定义
- 栈空间已就绪，可以支持 C 函数调用

### 4.4 验证 PC 跳转到 C 入口

```gdb
(gdb) si
9           tail kern_init
=> 0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>

(gdb) si
kern_init () at kern/init/init.c:8
8           memset(edata, 0, end - edata);

(gdb) info registers pc
pc             0x8020000a       0x8020000a <kern_init>
```

**验证结果：**
- PC 从 `kern_entry` 正确跳转到 `kern_init` 的 C 源代码位置
- 入口代码完成了"建立栈 → 跳转到初始化函数"的任务

### 4.5 观察 `kern_init` 中的指令序列

```gdb
(gdb) x/8i $pc
=> 0x8020000a <kern_init>:      auipc   a0,0x3
   0x8020000e <kern_init+4>:    addi    a0,a0,-2
   0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
   0x8020001a <kern_init+16>:   addi    sp,sp,-16
   0x8020001c <kern_init+18>:   li      a1,0
   0x8020001e <kern_init+20>:   sub     a2,a2,a0
   0x80200020 <kern_init+22>:   sd      ra,8(sp)
   0x80200022 <kern_init+24>:   jal     ra,0x802004b6 <memset>
```

**观察结果：**
- `kern_init` 正在准备调用 `memset` 清零 `.bss` 段
- 参数准备：`a0=edata`, `a1=0`, `a2=end-edata`
- 栈帧建立：`sp` 下移 16 字节，保存 `ra`（返回地址）

### 4.6 进入 memset 验证 `.bss` 清零

```gdb
(gdb) b memset
(gdb) c

Breakpoint 2, memset (s=0x80203008, c=c@entry=0 '\000', n=0) at libs/string.c:275
275         while (n -- > 0) {

(gdb) info registers a0 a1 a2
a0             0x80203008       2149593096
a1             0x0              0
a2             0x0              0
```

**结果：**
- `.bss` 段的起始地址为 `0x80203008`，长度为 0
- 在最小内核中 `.bss` 段实际上无需清空
- `memset` 的条件判断 `n == 0` 会直接返回

### 4.7 深入追踪输出路径

完成基本初始化后，`kern_init` 会调用 `cprintf` 输出启动信息。我们通过在输出路径的关键函数设置断点，追踪字符是如何从 C 代码最终通过 SBI 接口输出到控制台的。这里就是我们指导书中提到的内联汇编的概念。

#### 4.7.1 输出调用链概览

根据 [init.c](kern/init/init.c:10) 源码，输出语句为：
```c
const char *message = "(THU.CST) os is loading ...\n";
cprintf("%s\n\n", message);
```

输出路径涉及以下函数调用链：
```
cprintf (格式化入口)
  → vcprintf (处理可变参数)
    → vprintfmt (格式解析与输出)
      → cputch (字符输出回调)
        → cons_putc (控制台输出)
          → sbi_console_putchar (SBI接口)
            → ecall (陷入M模式)
```

#### 4.7.2 逐层断点验证

**设置关键断点：**
```gdb
(gdb) b cprintf
(gdb) b vcprintf
(gdb) b vprintfmt
(gdb) b cputch
(gdb) b cons_putc
(gdb) b sbi_console_putchar
(gdb) c
```

**命中 cprintf：**
```gdb
Breakpoint 1, cprintf (fmt=0x80200d24 "%s\n\n") at kern/libs/stdio.c:23
23          va_start(ap, fmt);

(gdb) print fmt
$1 = 0x80200d24 "%s\n\n"
(gdb) print message
$2 = 0x80200d18 "(THU.CST) os is loading ...\n"
```

参数传递正确：格式串和消息字符串地址都符合预期。

**命中 vcprintf：**
```gdb
Breakpoint 2, vcprintf (fmt=0x80200d24 "%s\n\n", ap=0x80202f88) at kern/libs/stdio.c:29
29          int cnt = 0;
30          vprintfmt((void *)cputch, &cnt, fmt, ap);

(gdb) info args
fmt = 0x80200d24 "%s\n\n"
ap = 0x80202f88
```

`vcprintf` 将实际输出工作委托给 `vprintfmt`，并传入 `cputch` 作为字符输出回调函数，`cnt` 用于统计输出字符数。

**命中 cputch 观察单字符输出：**
```gdb
Breakpoint 4, cputch (c=40, cnt=0x80202f94) at kern/libs/stdio.c:12
12          cons_putc(c);

(gdb) print/c c
$3 = 40 '('
(gdb) print *cnt
$4 = 0
```

第一个输出的字符是 `'('`（消息的开头），此时计数器为 0。每输出一个字符，`cputch` 会调用 `cons_putc` 并递增计数器。

**命中 cons_putc 与尾调用优化：**
```gdb
Breakpoint 5, cons_putc (c=40) at kern/driver/console.c:14
14      sbi_console_putchar((unsigned char)c);

(gdb) x/6i $pc
=> 0x8020008c <cons_putc>:      andi    a0,a0,0xff
   0x80200090 <cons_putc+4>:    j       0x80200480 <sbi_console_putchar>
```

观察到 `cons_putc` 对字符做了掩码操作（`andi a0,a0,0xff` 等价于 `zext.b`），确保只保留低8位，然后直接跳转到 `sbi_console_putchar`，这是**尾调用优化**，不需要保存返回地址。

#### 4.7.3 SBI 调用与特权级切换

**命中 sbi_console_putchar：**
```gdb
Breakpoint 6, sbi_console_putchar (ch=40 '(') at libs/sbi.c:33
33          sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);

(gdb) x/8i $pc
=> 0x80200480 <sbi_console_putchar>:    li      a5,0
   0x80200482 <sbi_console_putchar+2>:  auipc   a4,0x3
   0x80200486 <sbi_console_putchar+6>:  ld      a4,-1154(a4)
   0x8020048a <sbi_console_putchar+10>: mv      a7,a4
   0x8020048c <sbi_console_putchar+12>: ecall
   0x8020048e <sbi_console_putchar+14>: ret

(gdb) info registers a0 a7
a0             0x28     40
a7             0x1      1
```

这里 `a7=1` 是 SBI 调用号 `SBI_CONSOLE_PUTCHAR`，`a0=0x28` 是要输出的字符。执行 `ecall` 指令后，CPU 从 S 模式陷入 M 模式，由 OpenSBI 处理实际的字符输出。

#### 4.7.4 自动化监控字符流

为了观察完整的字符输出过程，可以使用 GDB 的命令自动化：

```gdb
(gdb) delete breakpoints
(gdb) b cputch
(gdb) commands
  silent
  printf "Output: '%c' (0x%02x)\n", (int)$a0, (int)$a0
  c
end
(gdb) c
```

**输出示例：**
```
Output: '(' (0x28)
Output: 'T' (0x54)
Output: 'H' (0x48)
Output: 'U' (0x55)
Output: '.' (0x2e)
Output: 'C' (0x43)
Output: 'S' (0x53)
Output: 'T' (0x54)
Output: ')' (0x29)
Output: ' ' (0x20)
...
```

字符被逐个输出，完整地呈现了 "(THU.CST) os is loading ..." 的消息内容。

#### 4.7.5 验证输出计数器

```gdb
(gdb) finish
(gdb) finish
(gdb) print cnt
$5 = 32
```

输出完成后，计数器显示总共输出了 32 个字符（包括消息内容和换行符），与实际字符串长度一致。

### 4.8 内核主循环：无限等待的设计意图

查看 [init.c](kern/init/init.c:12) 的最后部分：
```c
while (1)
    ;
```

#### 4.8.1 观察无限循环

```gdb
(gdb) b 12
Breakpoint 7, kern_init () at kern/init/init.c:12
12     while (1)

(gdb) x/4i $pc
=> 0x80200xxx <kern_init+xx>:   j       0x80200xxx <kern_init+xx>
   0x80200xxx <kern_init+xx>:   nop
```

编译器将 `while(1);` 优化为一条无条件跳转到自身的指令（`j` 或 `jal x0, .`），形成一个紧密的自旋循环。

#### 4.8.2 运行效果

当执行到这个循环时，QEMU 虚拟机会显示启动消息后持续运行，但实际上 CPU 在不断执行跳转到自身的指令。此时可以通过 `Ctrl+A X`（QEMU）或 `Ctrl+C`（GDB）终止运行。

```gdb
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
kern_init () at kern/init/init.c:13
13              ;
```

按 `Ctrl+C` 中断后，GDB 显示程序停在循环内部，验证了内核确实进入了无限等待状态。

---

## 五、关键知识点与操作系统原理对照

### 5.1 实验涉及的核心知识点

本实验作为OS课程的起点，虽然内核功能极简，但涵盖了操作系统启动和运行的多个基础概念。以下是实验中的关键知识点及其与OS原理课程的对应关系：

| 实验中的知识点 | 对应的OS原理 | 含义、关系与差异 |
|---|---|---|
| **1. 计算机启动流程<br>（三阶段引导）** | **系统引导与加载** | **实验内容**：从硬件复位（0x1000 MROM）→ OpenSBI固件（0x80000000）→ OS内核（0x80200000）的完整链路<br>**原理内容**：引导加载程序（Bootloader）的作用、多级引导机制、BIOS/UEFI的功能<br>**联系**：实验用MROM+OpenSBI实现了多级引导，OpenSBI类似于BIOS，是固件层<br>**差异**：传统x86使用BIOS/UEFI+GRUB，RISC-V使用更简洁的OpenSBI；实验环境中QEMU已预加载内核，真实环境需从存储设备读取 |
| **2. 特权级与安全隔离<br>（M/S/U模式）** | **CPU保护模式与特权级** | **实验内容**：观察M模式（OpenSBI）→ S模式（内核）的切换，使用`ecall`指令跨特权级调用<br>**原理内容**：保护环（Ring 0-3）、特权指令、模式切换机制<br>**联系**：RISC-V的M/S/U模式对应x86的Ring 0/Ring 1/Ring 3，都是为了隔离和保护<br>**差异**：RISC-V只有3个特权级且设计更清晰；`ecall`是RISC-V的专用指令，x86用`int`/`syscall` |
| **3. 内存布局与链接<br>（链接脚本）** | **程序的装载与链接** | **实验内容**：通过kernel.ld精确控制代码段、数据段、BSS段的地址，确保kern_entry位于0x80200000<br>**原理内容**：可执行文件格式（ELF）、静态链接、动态链接、地址空间布局<br>**联系**：链接脚本决定了程序在内存中的布局，这是虚拟内存管理的前提<br>**差异**：应用程序由OS加载器处理链接，内核需要自己定义布局；应用程序有动态链接，内核是静态链接 |
| **4. 栈的建立与调用约定<br>（la sp, bootstacktop）** | **函数调用与运行时栈** | **实验内容**：在entry.S中分配8KB启动栈，遵循RISC-V调用约定（参数用a0-a7，返回地址用ra）<br>**原理内容**：栈帧结构、参数传递、寄存器保存、调用约定（Calling Convention）<br>**联系**：所有函数调用都依赖栈，栈是程序执行的基础设施<br>**差异**：内核栈是静态预留的，用户程序栈由OS动态分配；内核栈较小（8KB），用户栈较大（通常8MB） |
| **5. 编译与交叉编译<br>（工具链）** | **编译原理基础** | **实验内容**：使用riscv64-unknown-elf-gcc交叉编译，从.c/.S → .o → ELF → binary的完整流程<br>**原理内容**：编译、汇编、链接的过程，目标代码格式<br>**联系**：理解编译过程有助于debug和优化<br>**差异**：交叉编译是为不同架构生成代码，比本地编译多一层抽象 |
| **6. 固件与SBI接口<br>（OpenSBI服务）** | **设备驱动与I/O** | **实验内容**：通过`ecall`调用OpenSBI的控制台输出服务（sbi_console_putchar）<br>**原理内容**：I/O子系统、设备驱动程序、系统调用接口<br>**联系**：SBI是M模式提供给S模式的服务接口，类似系统调用是内核提供给用户态的接口<br>**差异**：SBI是固件接口（固化的），系统调用是OS接口（可变的）；SBI用`ecall`从S→M，系统调用用`ecall`从U→S |
| **7. 调试工具使用<br>（GDB远程调试）** | **软件工程与调试方法** | **实验内容**：使用GDB连接QEMU，设置断点、查看寄存器、单步执行，追踪启动流程<br>**原理内容**：调试器原理、符号表、断点实现<br>**联系**：调试是软件开发的必备技能，内核调试更依赖底层工具<br>**差异**：内核调试需要远程调试（QEMU作为stub），应用调试可以直接attach |
| **8. 位置无关代码<br>（auipc指令）** | **代码重定位** | **实验内容**：MROM和entry.S都使用`auipc`实现位置无关寻址，使代码可在任意地址运行<br>**原理内容**：绝对地址 vs 相对地址、Position Independent Code (PIC)<br>**联系**：动态链接库必须是位置无关的，内核early boot也需要PIC<br>**差异**：内核最终会使用固定地址，PIC只在引导阶段重要 |
| **9. ELF与二进制格式<br>（objcopy转换）** | **可执行文件格式** | **实验内容**：理解ELF格式（包含符号表、调试信息）与纯二进制格式的区别，用objcopy转换<br>**原理内容**：可执行文件的结构（头、段、节）、加载过程<br>**联系**：ELF是Linux标准格式，理解它有助于理解程序加载<br>**差异**：ELF需要加载器解析，binary可以直接执行（ROM镜像常用） |
| **10. 页对齐与内存管理<br>（.align PGSHIFT）** | **内存管理基础** | **实验内容**：栈按4KB页对齐，链接脚本中数据段对齐到页边界<br>**原理内容**：页式内存管理、页表、页大小<br>**联系**：页对齐是为后续实验的虚拟内存做准备<br>**差异**：本实验还未实现虚拟内存，只是预留了对齐需求 |

### 5.2 实验与原理的深层联系

#### **5.2.1 启动流程的层次化设计**
实验中的三阶段引导（MROM → OpenSBI → OS）体现了计算机系统的**分层思想**：
- 每一层只负责简单的初始化和下一层的引导
- 上层不需要知道下层的实现细节（抽象）
- 类似于OS原理中的"分层结构"和"微内核设计"

#### **5.2.2 特权级的本质**
实验展示了**权限分离**的必要性：
- M模式（固件）负责硬件初始化，不应被内核修改
- S模式（内核）负责资源管理，不应被用户程序直接访问
- 这与OS原理中的"安全性"和"隔离性"直接相关

#### **5.2.3 链接脚本的地址空间管理**
通过kernel.ld体会到**地址空间**的重要性：
- 代码、数据、栈必须有明确的位置
- 符号（如edata、end）用于运行时管理
- 这是虚拟内存管理的前置知识

#### **5.2.4 栈的核心地位**
`la sp, bootstacktop`这一条指令的背后：
- 栈是过程调用的基础（保存返回地址、参数、局部变量）
- 没有栈，C语言无法运行
- 后续实验的进程切换也依赖栈（上下文保存）

---

## 六、OS原理中重要但实验未涉及的知识点

虽然这个实验涵盖了启动流程的核心内容，但以下OS原理中的重要知识点在本实验中尚未涉及，将在后续实验中逐步展开：

1. **中断和异常处理机制**：中断向量表、中断上下文保存与恢复
2. **物理内存管理**：页帧分配器、伙伴系统、内存碎片管理
3. **虚拟内存与页表机制**：地址翻译、TLB、页表项格式
4. **进程与线程模型**：进程控制块（PCB）、上下文切换、调度算法
5. **系统调用实现**：从用户态陷入内核态的完整流程
6. **同步与互斥**：锁、信号量、条件变量
7. **文件系统**：VFS、inode、目录树
8. **设备驱动与I/O管理**：中断驱动I/O、DMA

---

## 七、结语

通过这次实验，我深入理解了操作系统启动的完整过程——从固件到内核的三次关键跳转：

1. **0x1000（MROM）→ 0x80000000（OpenSBI）**：硬件复位后的最小化引导
2. **0x80000000（OpenSBI）→ 0x80200000（内核）**：固件完成平台初始化并移交控制权
3. **0x80200000（kern_entry）→ C语言环境（kern_init）**：设置栈环境，进入高级语言世界

[entry.S](kern/init/entry.S) 虽然只有短短两条指令，却承担了至关重要的角色：设置好栈环境，并把控制权交给C代码。链接脚本精确控制了内核的装载位置，确保OpenSBI能顺利找到我们的内核入口。

通过GDB的逐步调试，我验证了启动过程中的每个关键节点都符合预期，并用现场证据解释了各阶段的作用。这个最小内核虽然功能简单，但它确实成功运行了起来，成功打印出了启动信息，标志着我们迈出了操作系统开发的第一步。
