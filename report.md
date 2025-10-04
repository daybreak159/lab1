# Lab1 实验报告（练习 1）— 最小可执行内核与入口 `entry.S` 分析

> 目标：理解从 QEMU/OpenSBI 到内核 `entry.S` 的启动链路，并说明 `la sp, bootstacktop` 与 `tail kern_init` 的作用与目的；在此基础上用 GDB 对关键节点进行验证。  
> 实验环境：QEMU `virt` + OpenSBI（`-bios default`），交叉工具链 `riscv64-unknown-elf-*`。

---

## 一、实验概述与执行流

**完整流程**  
上电复位 `PC=0x1000` → 进入 OpenSBI@`0x80000000` → 按约定把控制权交给内核装载基址 `0x80200000` → **执行 `kern/init/entry.S:kern_entry`** → 立好内核栈并**不可返回**地跳到 `kern_init()`（C 端）→ 清 `.bss`、初始化最小输出 → 通过 SBI 打印启动信息（证明“内核已活”）。

**关键文件（最小路径）**
- `tools/kernel.ld`：指定镜像起点 `0x80200000` 与入口符号 `kern_entry`；组织 `.text/.rodata/.data/.bss` 布局。
- `kern/init/entry.S`：立栈（`la sp, bootstacktop`）→ 交棒（`tail kern_init`）。
- `kern/init/init.c`：`kern_init()` 清 `.bss`，调用 `cprintf()` 打印。
- `libs/sbi.c` / `kern/driver/console.c` / `kern/libs/stdio.c` / `libs/printfmt.c`：把“打印一个字符”的 SBI 能力封装为 `cprintf()`。

---

## 二、理论分析：从地址布局到 `entry.S` 两条关键指令

### 2.1 启动地址与控制权移交（PC 的路径）
- **复位点**：QEMU 复位后 `PC=0x1000`，执行极少量引导代码并进入固件。  
- **固件（OpenSBI）**：驻留 **`0x80000000`**，完成最小机器态初始化（M 模式），随后把控制权移交给内核。  
- **内核装载基址**：内核镜像被摆放在 **`0x80200000`**；OpenSBI 最终把 `PC` 指向这里，内核开始执行。

> 结论：**第一条内核指令**应当正好位于 **`0x80200000`**。

### 2.2 链接脚本如何保证“第一条就是入口”
`tools/kernel.ld` 的三条关键约束：
1. `BASE_ADDRESS = 0x80200000`：把镜像整体从这里开始排布；  
2. `ENTRY(kern_entry)`：设置 ELF 层面的入口符号（调试/装载参考）；  
3. `.text : { *(.text.kern_entry) *(.text ...) }`：**强制把 `.text.kern_entry` 放在 .text 最前**。

配合在 `entry.S` 中把入口符号写进 `.text.kern_entry`（或 `.text` 且确保排序），就能保证**`kern_entry` 的机器码在 0x80200000**，使得 OpenSBI 跳过来后第一条执行的就是我们定义的入口。

### 2.3 `entry.S` 的两条关键指令

```asm
kern_entry:
    la   sp, bootstacktop    # ① 立好启动栈（sp 指到高地址端）
    tail kern_init           # ② 不可返回地跳到 C 的入口
```

#### ① `la sp, bootstacktop` —— 给 C 铺好“背包”
- **作用**：把符号 `bootstacktop` 的地址装入 `sp`（`la` 会展开为 PC 相对的 `auipc+addi`），令 **`sp` 指向启动栈的“高地址端”**。  
- **必要性**：C 函数调⽤必须依赖对齐且可写的栈（保存返回地址、临时变量等）。刚进内核时**没有现成的栈**，因此入口必须先“背好背包”。  
- **为何是高地址端**：RISC-V 栈**向下生长**（从高地址往低地址使用），先把 `sp` 放在高位，后续 `addi sp, sp, -N` 建立栈帧最自然。

#### ② `tail kern_init` —— 只去不回的交接
- **作用**：尾调用伪指令，汇编为**不写返回地址**的跳转（近似 `jalr x0, ...`），把控制权直接交给 `kern_init()`。  
- **目的**：`entry` 只是“一次性跳板”，完成立栈后应**不可返回**地进入 C 世界。尾调用语义简洁，也省去保存/恢复返回地址的开销。

### 2.4 启动栈为何放在 `.data` 段、为何“页对齐”
```asm
.section .data
.align PGSHIFT          # 页对齐（4KB）
bootstack:              # 启动栈起点（低地址）
    .space KSTACKSIZE   # 预留 KSTACKSIZE 字节
bootstacktop:           # 启动栈终点（高地址）= 栈顶
```
- **.data 的用途**：启动早期**还没有内存分配器/页表**，用镜像里预留的一整块连续内存当“启动栈”最稳妥；这段内存**不会被执行**，只是被 `sp` 引用和使用。  
- **页对齐（`.align PGSHIFT`）**：使该内存块落在页边界，有利于后续分页、加“保护页”、以及满足 ABI 对齐（配合合适的 `KSTACKSIZE` 还能满足 16B 对齐）。  
- **地址关系**：`bootstacktop = bootstack + KSTACKSIZE`（高地址端），因此 `sp` 初始化为 `bootstacktop`。

### 2.5 从 `kern_init` 到 `cprintf` 再到 SBI：调用路径与约定
- `kern_init()` 首先清 `.bss`：  
  链接器提供符号 `edata` / `end` → `memset(edata, 0, end-edata)`。  
- `cprintf()` → 内部通过 `cputch/cons_putc` 输出字符 → `sbi_console_putchar()`：  
  封装 `sbi_call(type, a0, a1, a2)`，按 RISC-V 约定把**功能号放 a7**、参数放 **a0..a2**，执行 `ecall` **陷入 M 模式**，由 OpenSBI 完成实际输出，返回值放 `a0`。

> 这条“从 C 到 SBI”的链路，验证了我们在最小内核阶段的**唯一 I/O 能力**（可控地打印字符）。

### 2.6 理论层面的“可验证不变量”
用 GDB 验证时，重点检查这些不变量是否成立：
1. **入口地址**：`0x80200000` 处应当是 `kern_entry` 的机器码；  
2. **栈指针初始化**：`sp == &bootstacktop`，且 `&bootstacktop - &bootstack == KSTACKSIZE`；  
3. **函数序言/参数**：在 `kern_init` 附近能看到 `addi sp,sp,-N`、保存 `ra`、设置 `a0/a1/a2` 等；  
4. **打印路径可达**：可断在 `cprintf` 和 `sbi_console_putchar`，看到 `ecall` 前的寄存器设置；  

---

## 三、实验验证：GDB 观察要点与结论

> 启动方式：`make debug`（QEMU 暂停并开放 1234 端口） + `make gdb` 连接。  
> 断点方案：`b *0x80200000`（或 `b kern_entry`）、`b kern_init`、`b cprintf`、`b sbi_console_putchar`。

### 3.1 断到 `kern_init`（验证“交棒成功”）
```
Breakpoint 1, kern_init () at kern/init/init.c:8
8           memset(edata, 0, end - edata);
```

### 3.2 栈指针与启动栈布局正确（验证“立栈成功”）
```
(gdb) p/x &bootstack
$1 = 0x80201000
(gdb) p/x &bootstacktop
$2 = 0x80203000
(gdb) info reg sp
sp             0x80203000       0x80203000 <SBI_CONSOLE_PUTCHAR>
(gdb) p/x &bootstacktop - &bootstack
$3 = 0x2000  # 8 KiB
```

### 3.3 `kern_init` 序言与参数准备（清 `.bss` 之前的现场）
```
(gdb) x/6i $pc
=> 0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
   0x8020001a <kern_init+16>:   addi    sp,sp,-16
   0x8020001c <kern_init+18>:   li      a1,0
   0x8020001e <kern_init+20>:   sub     a2,a2,a0
   0x80200020 <kern_init+22>:   sd      ra,8(sp)
```

### 3.4 从 `cprintf` 到 `sbi_console_putchar`（打印链路贯通）
```
Breakpoint 2, cprintf (fmt=fmt@entry=0x802004c8 "%s\n\n")
    at kern/libs/stdio.c:40
40          va_start(ap, fmt);
(gdb)  x/6i $pc
=> 0x80200054 <cprintf>:        addi    sp,sp,-96
   0x80200056 <cprintf+2>:      addi    t1,sp,40
   0x8020005a <cprintf+6>:      sd      a1,40(sp)
   0x8020005c <cprintf+8>:      sd      a2,48(sp)
   0x8020005e <cprintf+10>:     sd      a3,56(sp)
   0x80200060 <cprintf+12>:     mv      a2,a0

Breakpoint 3, sbi_console_putchar (ch=40 '(') at libs/sbi.c:33
33          sbi_call(SBI_CONSOLE_PUTCHAR, ch, 0, 0);
```

**结论（与理论逐项对照）**  
- 入口对：OpenSBI 跳转至 `0x80200000`（`kern_entry`）。  
- 立栈对：`sp=bootstacktop`，大小/对齐均符合期望。  
- 进 C 对：`kern_init` 的序言与参数寄存器设置正确。  
- 打印对：`cprintf`→`sbi_console_putchar` 可断可见，`ecall` 路径畅通。  
- 回溯提示可解释：因 `tail` 不设置返回地址，GDB 回溯到 OpenSBI 区域属预期。

---

## 四、练习 1 问答（核心作答）

**Q1：`la sp, bootstacktop` 做了什么？目的是什么？**  
- **做了什么**：把 `bootstacktop` 的地址加载到 `sp`，将**栈顶**初始化到我们在 `.data` 段预留的启动栈的**高地址端**。  
- **目的**：为 C 函数提供对齐且可写的栈空间；没有栈，C 调用无法安全进行（保存返回地址、局部变量等）。

**Q2：`tail kern_init` 做了什么？目的是什么？**  
- **做了什么**：以**尾调用**方式把控制权转移到 `kern_init()`，不再返回 `kern_entry`。  
- **目的**：`entry` 是一次性“桥”，完成立栈后应直接进入 C 世界进行真正初始化；尾调用语义简洁、开销更小。

---

## 五、重要知识点 & OS 原理映射

| 本实验知识点 | OS 原理点 | 含义/关系/差异（简述） |
|---|---|---|
| 链接脚本 `kernel.ld`（`BASE=0x80200000`，`ENTRY(kern_entry)`） | 装载与链接、地址空间布局 | 指定镜像装载地址与入口符号；保证第一条就是入口指令。 |
| `entry.S` 立栈+尾调用 | 调用约定、启动序列 | 栈对齐与函数调用依赖；入口只做“环境铺设与交接”。 |
| `.data` 预留启动栈（4KB 页对齐） | 内存布局、对齐策略 | 页对齐便于分页/保护；启动早期先用镜像里连续空间。 |
| OpenSBI（M 态）→ 内核（S 态） | 特权级与陷入/返回 | 固件初始化后交接到内核；SBI 提供受控服务（打印/定时器）。 |
| ELF ↔ BIN（`objcopy`） | 可执行格式、装载机制 | ELF 便于调试（符号/段表），BIN 便于裸机装载（线性字节流）。 |
| GDB 断点/寄存器/反汇编 | 调试与验证 | 以事实验证入口地址、栈位置、指令序列、调用路径。 |

---

## 六、OS 原理中重要但本实验未覆盖（或只轻触）的内容

- 中断/异常控制与 `stvec` 配置、陷入返回流程  
- 物理/虚拟内存与页表、TLB、缺页异常处理  
- 进程/线程模型与调度、上下文切换、多核支持  
- 系统调用完整路径（U→S→M）、权限隔离与安全  
- 文件系统/磁盘 I/O 与缓存管理  
- 设备驱动模型、总线/DMA

---

## 七、结语

`entry.S` 是“从固件世界到 C 世界”的跳板：**先立栈**（`la sp, bootstacktop`），**再交棒**（`tail kern_init`）。链接脚本把它放到 **0x80200000** 的最前，因此 OpenSBI 一跳进来，CPU 执行的第一条内核指令就是这里。我们用 GDB 实证了每一步：**入口对、栈对、C 在跑、SBI 可达**，达成“最小内核”的目标。
