# 一、寄存器

### 1.1、控制寄存器

| 寄存器     | 名称                     | 用途简介                                                     |
| ---------- | ------------------------ | ------------------------------------------------------------ |
| **CR0**    | 控制寄存器0              | 控制保护模式、分页、协处理器、缓存等基本特性                 |
| **CR2**    | 控制寄存器2              | 保存发生**页面错误**时的**线性地址**                         |
| **CR3**    | 控制寄存器3              | 存储当前页目录（Page Directory）的物理地址，用于分页机制     |
| **CR4**    | 控制寄存器4              | 控制更高级别的功能（如物理地址扩展、SSE、页大小扩展等）      |
| **CR8**    | 控制寄存器8（仅 x86_64） | 控制中断优先级（TLP，任务优先级级别）                        |
| IP/EIP/RIP | 程序计数器 / 指令指针    | 指向下一条指向的指令（IP：16位实模式下，EIP：32位保护模式，RIP：64位） |



#### 1.1.1、**CR0：核心控制器**

`CR0` 是最重要的控制寄存器，包含多个位（bit flags），用于开启/关闭核心特性。

常用位说明：

| 位号 | 名称   | 含义                                        |
| ---- | ------ | ------------------------------------------- |
| 0    | **PE** | Protection Enable，开启保护模式（1 = 开启） |
| 1    | **MP** | Monitor Coprocessor                         |
| 2    | **EM** | Emulation：无浮点支持时开启（模拟）         |
| 3    | **TS** | Task Switched：任务切换标志                 |
| 4    | **ET** | Extension Type（遗留用途）                  |
| 5    | **NE** | Numeric Error：开启 FPU 错误报告            |
| 16   | **WP** | Write Protect：用户态是否能写只读页         |
| 18   | **AM** | Alignment Mask：对齐检查                    |
| 31   | **PG** | Paging：是否开启分页机制                    |

### 1.2、段寄存器

| 寄存器 | 名称                                 | 通常用途       |
| ------ | ------------------------------------ | -------------- |
| `CS`   | **代码段**寄存器                     | 代码的段基地址 |
| `DS`   | **数据段**寄存器                     | 一般数据访问   |
| `SS`   | **堆栈段**寄存器                     | 栈操作使用的段 |
| `ES`   | 扩展段寄存器                         | 字符串指令中用 |
| `FS`   | 通常是线程局部存储（TLS）等          |                |
| `GS`   | 同上，也可用于 SMP 的 CPU 本地数据等 |                |

#### 1.2.1、实模式与保护模式下，段寄存器的区别

| 模式         | 作用机制                                                     |
| ------------ | ------------------------------------------------------------ |
| **实模式**   | 段寄存器的值 × 16 + 偏移（`seg * 16 + offset`）构成 **物理地址** |
| **保护模式** | 段寄存器存的是**段选择子**（selector），要通过 GDT/LDT 查找段描述符，计算出段基址和段限长等 |

####  1.2.2、段选择子的格式（16 位）

| 位段 | 含义                                                       |
| ---- | ---------------------------------------------------------- |
| 0-1  | RPL：请求者特权级（0-3）                                   |
| 2    | TI：GDT (0) / LDT (1)                                      |
| 3-15 | 段描述符索引（index），**占用13位（段描述符最大=8192-1）** |

#### 1.2.3、GDT描述符

```assembly
gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
    
    
# notes:
# lgdt gdtdesc 将 GDTR 设为 gdtdesc 指向的内容
```

#### 1.2.4、段描述符

```c
* segment descriptors */
struct segdesc {
    unsigned sd_lim_15_0 : 16;        // low bits of segment limit
    unsigned sd_base_15_0 : 16;        // low bits of segment base address
    unsigned sd_base_23_16 : 8;        // middle bits of segment base address
    unsigned sd_type : 4;            // segment type (see STS_ constants)
    unsigned sd_s : 1;                // 0 = system, 1 = application
    unsigned sd_dpl : 2;            // descriptor Privilege Level
    unsigned sd_p : 1;                // present
    unsigned sd_lim_19_16 : 4;        // high bits of segment limit
    unsigned sd_avl : 1;            // unused (available for software use)
    unsigned sd_rsv1 : 1;            // reserved
    unsigned sd_db : 1;                // 0 = 16-bit segment, 1 = 32-bit segment
    unsigned sd_g : 1;                // granularity: limit scaled by 4K when set
    unsigned sd_base_31_24 : 8;        // high bits of segment base address
};
```

### 1.3、通用寄存器

| 寄存器  | 用途常见缩写      | 常见用途说明                     |
| ------- | ----------------- | -------------------------------- |
| **EAX** | Accumulator       | 累加器，常用于算术运算、返回值   |
| **EBX** | Base              | 基址寄存器，常用于地址计算       |
| **ECX** | Counter           | 计数器，常用于循环、字符串操作   |
| **EDX** | Data              | 数据寄存器，常用于 I/O、乘除法   |
| **ESI** | Source Index      | 源地址寄存器，字符串操作源       |
| **EDI** | Destination Index | 目标地址寄存器，字符串操作目的地 |
| **EBP** | Base Pointer      | 基指针，栈帧定位                 |
| **ESP** | Stack Pointer     | 栈顶指针，函数调用栈使用         |

# 二、实模式

​	实模式是 CPU 上电后默认的运行模式，主要特点是：

| 特性            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| 地址空间        | **最多 1MB（2<sup>20</sup>）** 可访问内存（从 `0x00000` 到 `0xFFFFF`） |
| 地址计算方式    | 通过 **段寄存器:偏移量** 形式，例如 `CS:IP`、`DS:BX`         |
| 内存保护        | ❌ 没有任何保护机制，程序可任意读写所有内存                   |
| 多任务/分页     | ❌ 不支持                                                     |
| 指令集          | 基本是 8086 指令集                                           |
| 可直接访问 BIOS | ✅ 支持调用 BIOS 中断（如 `int 0x10` 显示字符）               |

### 2.1、什么是A20

​	A20（Address Line 20）是x86架构中的一个概念，在早期的8086/8088处理器**没有第21根地址线（A20）**,所以x86实模式中，cpu使用20位地址总线来访问内存（最多1MB）。

​	到了 80286/80386 时代，CPU 实际可以访问超过 1MB 的内存，但为了兼容旧软件，**A20 线默认是关闭的**（高地址位被屏蔽）。

### 2.2、如何开启A20

- 通过键盘控制器（最传统，最兼容）

开启A20逻辑

| 步骤 | 动作                   | I/O 端口             | 含义                                |
| ---- | ---------------------- | -------------------- | ----------------------------------- |
| 1    | 等待输入缓冲区为空     | 0x64                 | 检查 bit 1 = 0（输入缓冲区空）      |
| 2    | 发送命令 `0xD1`        | `out 0x64`           | 8042 命令：**写输出端口**           |
| 3    | 再次等待输入缓冲区为空 | 0x64                 | 再检查 bit 1 = 0                    |
| 4    | 写输出端口值           | `out 0x60, val`      | `val` = 设置 bit1（A20），如 `0xDF` |
| 5    | 可选验证是否成功       | 比较低1MB和高1MB地址 | 用来判断 A20 是否真的开启           |

```assembly
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
    
    
# notes:    
# inb 从 I/O 端口中读取一个 字节（byte） 数据
# testb 按位与（AND）测试指令，作用是对两个操作数进行按位与，但不保存结果，只影响标志位（Flags），特别是 ZF（零标志） 和 SF（符号标志）
# jnz jump if Not Zero（若非零则跳转） 的缩写，是基于 ZF（零标志位） 的条件跳转指令
# outb I/O 指令，用于将一个 字节（byte）数据输出 到指定的 I/O 端口，通常用于和硬件设备通信（如键盘控制器、串口、A20、CMOS 等）。
```

- 快速法（向端口0x92写入）

```
无
```

### 2.3、实模式下地址计算

​	线性地址 = 段寄存器 x 16 + 偏移量 = (段寄存器 << 4) + offset

### 2.4、实模式能访问的内存

​	0xFFFF:0xFFFF = 0xFFFF << 4 + 0xFFFF = 0x10FFEF > 1MB，由于A20地址线默认关闭，高出部分会wrap around到0x00000。

# 三、实模式切换保护模式

### 3.1、实模式 → 保护模式 切换的步骤（标准流程）

| 步骤 | 操作                              | 说明                             |
| ---- | --------------------------------- | -------------------------------- |
| 1️⃣    | **关闭中断**：`cli`               | 避免中断在切换过程中发生         |
| 2️⃣    | **开启 A20 地址线**（必要）       | 否则无法访问 1MB 以上内存        |
| 3    | **构建 GDT 表**                   | 包含 null 描述符、代码段、数据段 |
| 4    | **加载 GDTR**：`lgdt [gdtdesc]`   | 设置 GDT 寄存器                  |
| 5    | **设置 CR0 的 PE 位**             | 设置 CR0[0] = 1，开启保护模式    |
| 6    | **使用 `ljmp` 跳转刷新 CS 段**    | 必须使用 far jump 更新 CS        |
| 7    | **设置数据段寄存器（ds, ss 等）** | 切换到新段模型                   |
| 8    | 可选：开启分页，加载 IDT 等       | 如进入分页保护模式时             |

### 3.2、ljmp跳转刷新CS段

```assembly
# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
ljmp $PROT_MODE_CSEG, $protcseg


# notes:
# CS不可以直接使用MOV来幅值，只能通过ljmp或iret
# ljmp 远跳转指令,同时更新CS段寄存器和EIP/IP指令指针
# 指令格式：ljmp $selector, $offset
```

### 3.3、设置栈基址和栈顶寄存器(*)

```c
# Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
movl $0x0, %ebp
movl $start, %esp
call bootmain
```

