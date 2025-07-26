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
| 1    | **关闭中断**：`cli`               | 避免中断在切换过程中发生         |
| 2    | **开启 A20 地址线**（必要）       | 否则无法访问 1MB 以上内存        |
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
    

# notes:
# start为程序的起始地址（向上增长），可以作为栈顶指针（栈是从高地址向地址生张的）
# push指令，是先减栈顶指针，再写入值
```

# 四、函数调用

- 函数调用前，参数需要手动push，且从右到左依次调用
- 函数调用时，返回地址（**下一条执行指令的地址**）由call指令push到栈中
- 函数返回前，需要清除局部变量栈空间**move ebp, esp**
- 函数返回前，ret指令会pop栈，并且将pop得到的值（返回地址）进行跳转

### 4.1、32位程序中，栈从高地址往低地址增长

```
高地址
  ↑
  |     参数1 ← [EBP + 8]
  |     返回地址 ← [EBP + 4]
  |     旧 EBP ← [EBP]
  |     局部变量1 ← [EBP - 4]
  |     局部变量2 ← [EBP - 8]
  ↓
低地址
```

### 4.2、典型函数调用过程中的变化

```
call foo

foo:
    push ebp            ; 保存上一层 ebp
    mov esp, ebp        ; 建立新的栈帧，ebp 记录当前帧起点
    sub esp, N          ; 为局部变量分配栈空间
    ...
    mov ebp, esp        ; 清除局部变量栈空间
    pop ebp             ; 恢复上层 ebp
    ret                 ; 返回，使用栈中的返回地址
```

# 五、bootloader

### 5.1、ucore.img的组成

```
┌──────────────┬──────────────────────┬─────────────┐
│ 第 0 扇区     │ 第 1 扇区起：kernel   │ 剩余填充     │
│ bootblock     │ 内核代码             │ 全为 0       │
│ (512 bytes)  │                      │             │
└──────────────┴──────────────────────┴─────────────┘
```

### 5.2、生成ucore.img的Makefile

```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	# 用0来填充10000个block（默认大小为512B）=5MB的空白镜像文件
	$(V)dd if=/dev/zero of=$@ count=10000
	# 用bootblock来写入ucore.img
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
    # 用kernel来写入ucore.img的第一个扇区
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
# bootblock的大小不能超过512，因为kernel就是从第一个扇区开始写入的，会覆盖bootblock超过的部分
```

### 5.3、bootblock

​	根据tools/sign.c代码可知，bootblock大小不能超过510个字节（不够的部分填充0），同时sign会在bootblock后面添加0x55,0xAA两个字节一起构成512个字节完整的bootloader.

```
BIOS 加电后会读取磁盘 第 0 扇区（LBA 0） 的 512 字节到内存的 0x7C00。

这就是为什么 bootblock 必须恰好写到第一个扇区。

这个扇区的最后两个字节必须是 0x55AA（MBR magic number），表示是一个可引导扇区。
```

### 5.4、kernel

​	连接器ld使用32位ELF格式（elf_i386）和tools/kernel.dl链接脚本来生成kernel可执行程序

# 六、特权级

### 1、DPL、CPL、、RPL与IOPL

- DPL存在段描述符中，规定访问改段的权限级别（Descriptor Privilege Level），每个段的DPL固定，当进程访问一个段时，需要进程特权级检查。
- CPL存在于CS寄存器的低两位，即CPL是CS段描述符的DPL，是当前代码的权限级别（Current Privilege Level）
- RPL存在段选择子中，说明的是进程对段访问的请求权限，是对于段选择子而言的，每个段选择子有自己的RPL。而且RPL对每个段来说不是固定的，两次访问同一段时的RPL可以不同。**RPL可能会削弱CPL的作用**，例如当前CPL=0的进程要访问一个数据段，它把段选择符中的RPL设为3，这样它对该段仍然只有特权为3的访问权限。
- IOPL（I/O Privilege Level）即IO特权标志，位于eflag寄存器，用两位二进制位来表示，也称为I/O特权级字段。该字段指定了要求执行I/O指令的特权级。如果当前特权级别在数值上小于等于IOPL的值，那么，该I/O指令可执行，否则将发生一个保护异常。
  - 只有当CPL=0时，可以改变IOPL的值，当CPL<=IOPL时，可以改变IF标志位（**中断使能标志**）。

### 2、特权级检查

- 访问段时，有效权限 **EPL = max(CPL, RPL)**，只有当`EPL ≤ DPL`才允许访问段，RPL会影响段访问权限
- 访问门（调用门、陷阱门、中断门、任务门）时，**忽略RPL**，只考虑 CPL 和门的 DPL，当CPL ≤ DPL时，才能允许通过门切换。
  - 通过门，能达到特权提升的目的。若是CPL=目标段DPL，直接跳转不需要特权切换，反之进行特权提升（**CPU自动完成CPL切换，栈切换（使用TSS）**）。

### 3、通过中断切换特权级

#### 3.1、TSS

```c
struct taskstate {
    uint32_t ts_link;        // old ts selector
    uintptr_t ts_esp0;        // stack pointers and segment selectors
    uint16_t ts_ss0;        // after an increase in privilege level
    uint16_t ts_padding1;
    uintptr_t ts_esp1;
    uint16_t ts_ss1;
    uint16_t ts_padding2;
    uintptr_t ts_esp2;
    uint16_t ts_ss2;
    uint16_t ts_padding3;
    uintptr_t ts_cr3;        // page directory base
    uintptr_t ts_eip;        // saved state from last task switch
    uint32_t ts_eflags;
    uint32_t ts_eax;        // more saved state (registers)
    uint32_t ts_ecx;
    uint32_t ts_edx;
    uint32_t ts_ebx;
    uintptr_t ts_esp;
    uintptr_t ts_ebp;
    uint32_t ts_esi;
    uint32_t ts_edi;
    uint16_t ts_es;            // even more saved state (segment selectors)
    uint16_t ts_padding4;
    uint16_t ts_cs;
    uint16_t ts_padding5;
    uint16_t ts_ss;
    uint16_t ts_padding6;
    uint16_t ts_ds;
    uint16_t ts_padding7;
    uint16_t ts_fs;
    uint16_t ts_padding8;
    uint16_t ts_gs;
    uint16_t ts_padding9;
    uint16_t ts_ldt;
    uint16_t ts_padding10;
    uint16_t ts_t;            // trap on task switch
    uint16_t ts_iomb;        // i/o map base address
};
```

- TSS分布保留了特权等级0，1，2的栈（ss，esp寄存器值）

- TSS段的段描述符保存在GDT中，TR寄存器会保存TSS的段描述符，以提高索引速度

  - ```c
    static struct taskstate ts = {0};
    #
    static struct segdesc gdt[] = {
        SEG_NULL,
        [SEG_KTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),
        [SEG_KDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),
        [SEG_UTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),
        [SEG_UDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),
        [SEG_TSS]    = SEG_NULL,
    };
    
    static struct pseudodesc gdt_pd = {
        sizeof(gdt) - 1, (uint32_t)gdt
    };
    
    #
    /* gdt_init - initialize the default GDT and TSS */
    static void
    gdt_init(void) {
        // Setup a TSS so that we can get the right stack when we trap from
        // user to the kernel. But not safe here, it's only a temporary value,
        // it will be set to KSTACKTOP in lab2.
        // 设置TSS的ring0栈地址，包括esp寄存器和SS段寄存器
        ts.ts_esp0 = (uint32_t)&stack0 + sizeof(stack0);
        ts.ts_ss0 = KERNEL_DS;
    
        // initialize the TSS filed of the gdt
         // 将TSS写入GDT中
        gdt[SEG_TSS] = SEG16(STS_T32A, (uint32_t)&ts, sizeof(ts), DPL_KERNEL);
        gdt[SEG_TSS].sd_s = 0;
    
        // reload all segment registers
        // 加载GDT至GDTR寄存器
        lgdt(&gdt_pd);
    
        // load the TSS
        // 加载TSS至TR寄存器
        ltr(GD_TSS);
    }
    ```

#### 3.2、trapframe

​	trajpframe结构是进入中断门所必须的结构，其结构如下	

```C
/* registers as pushed by pushal */
struct pushregs {
    uint32_t reg_edi;
    uint32_t reg_esi;
    uint32_t reg_ebp;
    uint32_t reg_oesp;            /* Useless */
    uint32_t reg_ebx;
    uint32_t reg_edx;
    uint32_t reg_ecx;
    uint32_t reg_eax;
};

struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```

#### 3.3、中断处理例程的入口

中断处理例程的入口代码用于**保存上下文并构建一个trapframe**，其源代码如下

```assembly
#include <memlayout.h>

# vectors.S sends all traps here.
.text
.globl __alltraps
__alltraps:
    # push registers to build a trap frame
    # therefore make the stack look like a struct trapframe
    pushl %ds
    pushl %es
    pushl %fs
    pushl %gs
    pushal

    # load GD_KDATA into %ds and %es to set up data segments for kernel
    movl $GD_KDATA, %eax
    movw %ax, %ds
    movw %ax, %es

    # push %esp to pass a pointer to the trapframe as an argument to trap()
    pushl %esp

    # call trap(tf), where tf=%esp
    call trap

    # pop the pushed stack pointer
    popl %esp

    # return falls through to trapret...
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret
```

#### 3.4、切换特权级的过程
