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

## 三、GDB验证结果与分析

我用GDB设置了几个关键断点，具体观察启动过程中的各个环节。通过实际的调试验证，我完成了以下几个方面的确认：

### 3.1 实验验证概述

**成功进入kern_init：** 设置断点后，GDB显示正确停在了`kern_init`函数开头，证明控制权成功从entry.S转移到了C代码。

**栈设置正确：** 通过检查寄存器和符号值，我确认栈指针确实指向了`bootstacktop`，栈大小为8KB（`bootstacktop - bootstack = 0x2000`），栈位置符合预期。

**函数调用正常：** 观察指令序列，可以看到正常的函数序言，包括保存返回地址、设置参数等，说明C环境已经准备就绪。

**输出链路畅通：** 断点成功在`cprintf`和`sbi_console_putchar`处触发，验证了整个输出路径是正常工作的。

这些验证结果表明我们的启动机制设计是正确的，内核能够顺利从汇编世界过渡到C世界，并具备基本的输出能力。

---

### 3.2 实际终端输出分析

在完成了代码阅读之后，我们使用 GDB 对内核启动的关键过程进行了逐步调试，验证了 **入口两条指令** 的作用，以及 **SP/PC 的正确设置**，并观察了跳转到 `kern_init` 后的执行情况。

#### 调试目标
1. 验证 `PC` 能正确从引导入口 `kern_entry` 跳转到 C 初始化函数 `kern_init`。
2. 验证 `SP` 在 `kern_entry` 被设置为 `bootstacktop`，保证 C 语言运行环境可用。
3. 确认 `kern_init` 中执行 `.bss` 段清零，并准备调用 `memset`。

#### 调试过程

首先，进入 QEMU 调试模式，并在 GDB 中连接：
```bash
make debug    # 启动 QEMU 等待调试
make gdb      # 启动 GDB 并连接
```

##### 1) 命中内核入口并查看入口指令
```gdb
(gdb) b *kern_entry
Breakpoint 1 at 0x80200000: file kern/init/entry.S, line 7.
(gdb) c
Breakpoint 1, kern_entry () at kern/init/entry.S:7
7           la sp, bootstacktop
(gdb) x/8i $pc
=> 0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   mv      sp,sp
   0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>
   0x8020000a <kern_init>:      auipc   a0,0x3
   0x8020000e <kern_init+4>:    addi    a0,a0,-2
   0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
```

可以看到：
- 第一条指令 `la sp, bootstacktop` 将栈指针初始化为内核栈顶。
- 第三条指令 `j kern_init`（由 `tail kern_init` 汇编而来）将 **PC 无条件跳转**到 `kern_init`。

##### 2) 单步观察 SP 的变化
在执行入口指令前后查看栈指针：
```gdb
(gdb) i r sp
sp             0x8001bd80   # 跳转前的旧 SP 值

(gdb) si
(gdb) i r sp
sp             0x80203000   # 已更新为 bootstacktop
```

结果显示，执行 `la sp, bootstacktop` 后，SP 已经被正确设置为 0x80203000，对应我们在 `entry.S` 中定义的内核栈顶位置。  
进一步可用以下命令验证栈大小：
```gdb
(gdb) p/x &bootstacktop - &bootstack
$1 = 0x2000   # 8192 字节，即 2 页，符合 KSTACKSIZE 的定义
```

##### 3) 验证 PC 跳转到 C 入口
继续单步执行，观察 PC 的变化：
```gdb
(gdb) si
9           tail kern_init
(gdb) si
kern_init () at kern/init/init.c:8
8           memset(edata, 0, end - edata);
(gdb) i r pc
pc             0x8020000a <kern_init>
```

可以看到，PC 已经从 `kern_entry` 正确跳转到 `kern_init` 的 C 源代码位置。这说明入口代码完成了“建立栈 → 跳转到初始化函数”的任务。

##### 4) 观察 `kern_init` 中的指令序列
```gdb
(gdb) x/8i $pc
=> 0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
   0x8020001a <kern_init+16>:   addi    sp,sp,-16
   0x8020001c <kern_init+18>:   li      a1,0
   0x8020001e <kern_init+20>:   sub     a2,a2,a0
   0x80200020 <kern_init+22>:   sd      ra,8(sp)
   0x80200022 <kern_init+24>:   jal     ra,0x802004b6 <memset>
```

可以看到 `jal memset` 将被执行，用于清零 `.bss` 段，这正是实验要求中的关键步骤。

#### 实验现象与结论

1. **入口两条指令作用得到验证**：  
   - `la sp, bootstacktop` 成功设置 SP 为内核栈顶。  
   - `tail kern_init` 将 PC 跳转至 `kern_init`，进入 C 初始化。

2. **SP 的正确性**：  
   - SP 被置为 0x80203000，正好是 `bootstacktop`。  
   - `bootstack` 与 `bootstacktop` 的差值为 0x2000，说明栈大小为 8KB，符合设计。

3. **PC 的正确性**：  
   - PC 从 `0x80200008` 跳转至 `0x8020000a <kern_init>`，说明控制流正确。
   - 但是此处通过汇编代码可以发现调用了`j`跳转指令，跳转后的`PC`也是正常+4，这说明`<kern_init>`的指令就仅仅挨着`<kern_entry>`之后，哪怕不跳转也可以继续执行。但是这里我产生了一个问题：**为什么`<kern_init>`的指令就这么巧在`PC`的下一个呢？**
   - 解答：在文件`kernel.ld`中，定义了
   ```
       .text : {
        *(.text.kern_entry .text .stub .text.* .gnu.linkonce.t.*)
    }
    ```
    把 `.text.kern_entry` 段（也就是 `kern_entry` 所在的那几条汇编）放在整个内核代码的最开头。紧接着就是所有其他 `.text` 段，此处下一个就是`init.c` 编译出来的 `kern_init`），因此指令相连。

4. **初始化过程验证**：  
   - 在 `kern_init` 中，准备调用 `memset` 清零 `.bss` 段，为后续 C 代码运行做好准备。

#### Kern_init 内部调试过程

##### 5) 进入 memset 验证 `.bss` 清零

```gdb
(gdb) b memset
(gdb) c
Breakpoint 2, memset (s=0x80203008, c=c@entry=0 '\000', n=0) at libs/string.c:275
275         while (n -- > 0) {
(gdb) info registers a0 a1 a2
a0             0x80203008       2149593096
a1             0x0      0
a2             0x0      0
```

此时 `.bss` 段的起始地址为 `0x80203008`，长度为 0，说明在最小内核中 `.bss` 段实际上无需清空。
`beqz a2, ret` 指令会直接返回，完成初始化逻辑。

##### 6) 验证输出路径

继续运行程序并设置输出相关断点：

```gdb
(gdb) b cprintf
(gdb) b vcprintf
(gdb) b vprintfmt
(gdb) b cputch
(gdb) b cons_putc
(gdb) b sbi_console_putchar
(gdb) c
```

触发 `cprintf` 调用：

```
cprintf(fmt="%s\n\n", message="(THU.CST) os is loading ...\n")
```

逐层进入函数并检查参数传递情况：

* `a0=fmt`, `a1=message`：确认字符串常量地址正确。
* `a0=cputch`, `a1=&cnt`, `a2=fmt`, `a3=ap`：`vcprintf` 参数传递正常。
* `vprintfmt` 建立栈帧，`sp` 下移 128 字节，用于保存 `s` 系列寄存器。

##### 7) 命中 cputch 并验证输出字符

```gdb
(gdb) b cputch
(gdb) c
Breakpoint 5, cputch (c=40, cnt=0x80202f94) at kern/libs/stdio.c:12
12          cons_putc(c);
(gdb) i r a0 a1
a0 = 0x28 ('(')
a1 = 0x80202f94
```

可以看到当前输出的字符为 `'('`，计数器的地址也正确。接下来 `cons_putc()` 会将该字符交给 SBI 接口进行输出。

##### 8) 命中 cons_putc 并进入 sbi_console_putchar

```gdb
(gdb) s
Breakpoint 6, cons_putc (c=40) at kern/driver/console.c:14
14      sbi_console_putchar((unsigned char)c);
(gdb) x/6i $pc
=> 0x8020008c <cons_putc>:      zext.b  a0,a0
   0x80200090 <cons_putc+4>:    j       0x80200480 <sbi_console_putchar>
```

`zext.b a0,a0` 指令将字符扩展为无符号字节，确保高位清零。随后通过 `j` 指令直接跳转到 `sbi_console_putchar()`，不保存返回地址，从而实现了“尾调用”优化。

##### 9) 观察 SBI 调用与 ECALL

```gdb
(gdb) s
Breakpoint 7, sbi_console_putchar (ch=40 '(') at libs/sbi.c:33
33          sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);
(gdb) x/6i $pc
=> 0x80200480 <sbi_console_putchar>:    li a5,0
   0x80200482 <sbi_console_putchar+2>:  auipc a4,0x3
   0x80200486 <sbi_console_putchar+6>:  ld a4,-1154(a4)
   0x8020048a <sbi_console_putchar+10>: mv a7,a4
```

此时 `a0=0x28 ('(')`，`a7=1`（表示 SBI 调用号），执行 `ecall` 后陷入 M 模式，由 OpenSBI 完成字符输出。返回时 `a0=0`，说明调用执行成功。

##### 10) 验证计数器自增

```gdb
(gdb) finish
(gdb) x/wx 0x80202f94
0x80202f94: 0x00000001
```

每输出一个字符，`(*cnt)` 的值都会增加 1，说明输出计数器工作正常。

##### 11) 自动字符监控验证

通过自定义命令链观察每次输出的字符：

```gdb
(gdb) b cputch
(gdb) commands
  silent
  i r a0
  printf "Output char: %c\n", (int)$a0
  c
end
(gdb) c
```

输出结果如下（节选）：

```
Output char: (
Output char: T
```

此处可以观察到字符流被逐个打印，验证了字符输出路径完全畅通。




---

## 四、练习问题回答

**问题1：`la sp, bootstacktop`的作用和目的是什么？**  

这条指令将栈指针设置到预留栈空间的高地址端。没有这一步，后续的C代码将无法正常运行，因为函数调用需要使用栈来存储返回地址和局部变量。可以说，这一步是从汇编过渡到C语言的必要桥梁。

**问题2：`tail kern_init`的作用和目的是什么？**  

这条指令以尾调用方式跳转到C入口函数，它不会保存返回地址。这符合我们的设计意图——入口汇编代码只是一次性的跳板，完成基本设置后就应该"永不回头"地进入C世界进行更复杂的初始化工作。

**问题3：RISC-V 加电后最初执行的指令与启动流程验证**

**调试目标：** 追踪 CPU 从加电复位（0x1000）到内核入口（0x80200000）的全过程，确认最初指令的地址与作用。

**调试步骤：**

1. **启动 QEMU 并连接 GDB**

   GDB 连接成功后显示：
   ```
   Remote debugging using localhost:1234
   0x0000000000001000 in ?? ()
   ```

2. **查看复位向量处的前几条指令**

   ```gdb
   (gdb) x/10i $pc
   => 0x1000:  auipc t0,0x0
      0x1004:  addi  a1,t0,32
      0x1008:  csrr  a0,mhartid
      0x100c:  ld    t0,24(t0)
      0x1010:  jr    t0
   ```

   这些是 QEMU 固件的最初几条汇编指令，用于初始化寄存器并跳转到 OpenSBI 固件入口。

3. **验证内核入口执行**
   启动时，QEMU在CPU运行前已将内核预装到 0x80200000,在内核入口设置 硬件断点，以捕捉 CPU 实际执行到该地址的时刻
   ```gdb
   (gdb) hbreak *0x80200000
   (gdb) c   
   ```

   当 CPU 执行到该地址时触发断点：

   ```
   Breakpoint 1, kern_entry () at kern/init/entry.S:7
   7           la sp, bootstacktop
   ```

   表明 OpenSBI 已完成初始化，并将控制权交给内核开始执行。

4. **查看内核入口指令**

   ```gdb
   (gdb) b *0x80200000
   (gdb) c
   Breakpoint 2, 0x0000000080200000 in kern_entry ()
   ```

   说明控制权已转交至内核。继续查看：

   ```gdb
   (gdb) x/4i $pc
   => 0x80200000 <kern_entry>: la sp, bootstacktop
      0x80200004 <kern_entry+4>: tail kern_init
   ```

   可以看到内核的第一条指令为 `la sp, bootstacktop`，第二条为 `tail kern_init`，成功验证控制权已从 OpenSBI 转移到内核，并进入了 C 语言环境的初始化阶段。

---

**问题3：RISC-V 加电后最初执行的几条指令位于什么地址？它们主要完成了哪些功能？**

* **指令地址：** 位于固定复位向量地址 `0x1000`。
* **主要功能：**

  1. 读取硬件线程 ID（`csrr a0, mhartid`）。
  2. 从跳转表中取出 OpenSBI 入口地址（`ld t0, 24(t0)`）。
  3. 跳转到 OpenSBI 执行（`jr t0`）。

这几条指令组成了最小的引导序列，CPU 从 ROM 的复位向量 0x1000 开始执行，通过寄存器和跳转表找到 OpenSBI 的入口地址，从而把控制权交给固件。在 QEMU 的实验环境中，OpenSBI 运行在 0x80000000 附近，负责完成底层硬件初始化；随后它会将控制权移交给已经预加载在 0x80200000 的内核，此时 CPU 执行的第一条指令就是 la sp, bootstacktop，正式进入内核启动阶段。


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

这个最小内核虽然功能简单，但它确实成功运行了起来，成功打印出了启动信息，标志着我们迈出了操作系统开发的第一步。

