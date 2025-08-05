# 一、物理内存探测

操作系统需要知道了解整个计算机系统中的物理内存如何分布的，哪些被可用，哪些不可用。其基本方法是通过BIOS中断调用来帮助完成的。`bootasm.S`中新增了一段代码，使用BIOS中断检测物理内存总大小。

在讲解该部分代码前，先引入一个结构体

```c
struct e820map {      // 该数据结构保存于物理地址0x8000
    int nr_map;       // map中的元素个数
    struct {
        uint64_t addr;    // 某块内存的起始地址
        uint64_t size;    // 某块内存的大小
        uint32_t type;    // 某块内存的属性。1标识可被使用内存块；2表示保留的内存块，不可映射。
    } __attribute__((packed)) map[E820MAX];
};
```

以下是`bootasm.S`中新增的代码，详细信息均以注释的形式写入代码中。

```assembly
probe_memory:
    movl $0, 0x8000   # 初始化，向内存地址0x8000，即uCore结构e820map中的成员nr_map中写入0
    xorl %ebx, %ebx   # 初始化%ebx为0，这是int 0x15的其中一个参数
    movw $0x8004, %di # 初始化%di寄存器，使其指向结构e820map中的成员数组map
start_probe:
    movl $0xE820, %eax  # BIOS 0x15中断的子功能编号 %eax == 0xE820
    movl $20, %ecx    # 存放地址范围描述符的内存大小，至少20
    movl $SMAP, %edx  # 签名， %edx == 0x534D4150h("SMAP"字符串的ASCII码)
    int $0x15     # 调用0x15中断
    jnc cont      # 如果该中断执行失败，则CF标志位会置1，此时要通知UCore出错
    movw $12345, 0x8000 # 向结构e820map中的成员nr_map中写入特殊信息，报告当前错误
    jmp finish_probe    # 跳转至结束，不再探测内存
cont:
    addw $20, %di   # 如果中断执行正常，则目标写入地址就向后移动一个位置
    incl 0x8000     # e820::nr_map++
    cmpl $0, %ebx   # 执行中断后，返回的%ebx是原先的%ebx加一。如果%ebx为0，则说明当前内存探测完成
    jnz start_probe
finish_probe:
```

# 二、链接地址

- 审计`lab2/tools/kernel.ld`这个链接脚本，我们可以很容易的发现，链接器设置kernel的链接地址(link address)为`0xC0100000`，这是个虚拟地址。在uCore的`bootloader`中，使用如下语句来加载kernel：

  - ```c
    readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    ```

  - `0xC0010000 & 0xFFFFFF == 0x00100000`即kernel最终被装载入物理地址`0x10000`处，其相对偏移为`-0xc0000000`，与uCore中所设置的虚拟地址的偏移量相对应。

- 需要注意的是，在实验2的一些代码中会使用到如下两个变量，但这两个变量并没有被定义在任何C语言的源代码中：

  - ```c
    extern char end[];
    extern char edata[];
    ```

  - 实际上，它们定义于`kernel.ld`这个链接脚本中

  - ```assembly
    . = ALIGN(0x1000);
    .data.pgdir : {
        *(.data.pgdir)
    }
    
    PROVIDE(edata = .);
    
    .bss : {
        *(.bss)
    }
    
    PROVIDE(end = .);
    
    /DISCARD/ : {
        *(.eh_frame .note.GNU-stack)
    }
    // 脚本文件的结尾
    ```

  - `edata`表示`kernel`的`data`段结束地址；`end`表示`bss`段的结束地址（即整个`kernel`的结束地址）

    `edata[]`和 `end[]`这些变量是`ld`根据`kernel.ld`链接脚本生成的全局变量，表示相应段的结束地址，它们不在任何一个.S、.c或.h文件中定义，但仍然可以在源码文件中使用。

# 三、entry.S

`entry.S` 是内核启动后运行的第一个汇编代码，主要完成以下任务：

1. **设置处理器初始状态（特权模式、关中断等）**

- 设置 CPU 进入特权模式（如 ARM 的 SVC 模式、x86 的 Protected Mode）。
- 关闭中断，防止初始化阶段被打断。

2. **初始化内核栈指针**

- 给每个内核线程或 CPU 设置内核栈的起始地址（`sp`），以保证之后调用 C 函数不会栈溢出或乱跳。

3. **跳转到内核主函数 `kern_init`**

- `entry.S` 完成最基本的设置后，会调用 C 的 `kern_init()` 函数，正式进入内核主流程。

### debug命中问题

debug命中方式`break *0x00100000`

1. **链接地址设置为高地址（虚拟地址）**

- `entry.S` 和 `kern_init` 等内核代码的 **链接地址** 设置为 `0xC0100000`。
- 这样编译出来的代码中的符号地址、跳转地址等全是以 `0xC0100000` 为基础。

2. **启动时实际加载到物理地址 `0x00100000`**

- 在 bootloader 阶段（如 `boot/bootloader.S`），内核被加载到内存的 `0x00100000`（1MB）处。
- 此时系统还 **没有开启分页机制（paging）**，所以 CPU 使用的是实地址（物理地址）。

3. **分页机制建立后，虚拟地址才生效**

- 启动初期运行在物理地址（低地址）。
- 在 `kern_init()` 里，内核会开启页表，把虚拟地址 `0xC0000000+` 映射到物理地址（比如 0x00000000、0x00100000）。
- 从那时开始，代码才能正常使用 `0xC0100000` 这样的高地址。



# 四、段页式存储管理（*）

在保护模式中，`x86`体系结构将内存分为三种：逻辑地址（也称为虚拟地址）、线性地址和物理地址

- 逻辑地址：程序使用的地址空间，程序员看到和使用的地址。每个进程拥有独立的虚拟地址空间，程序通过逻辑地址访问内存。
  - 逻辑地址通过**段机制**（段寄存器 + 偏移）得到线性地址。
  - 逻辑地址由**段选择子+偏移量**组成。
- 线性地址：逻辑地址经段机制**（段选择子高13位对应的段描述符索引中的段基址地址+偏移量）**转换后的地址，即平坦化的地址，只有一个32位（或64位）值。
  - 在启用分页机制之前，线性地址和物理地址相同。
  - 启用分页后，线性地址会经过页表转换成物理地址。
- 物理地址：真正的物理内存地址，对应物理内存芯片上的地址。CPU最终访问的实际内存地址。
  - 由页表机制将线性地址映射到物理地址。
  - 可能存在内存映射I/O等特殊区域。

### 1、页目录项（`PDE`）

每个页目录项是 4 字节（32 位），结构如下：

| 位数  | 名称                   | 描述                           |
| ----- | ---------------------- | ------------------------------ |
| 31–12 | 页表物理基地址         | 指向对应页表（4KB对齐）        |
| 11    | 保留/OS 可用位         | 操作系统自定义使用（如访问位） |
| 10    | AVL                    | 操作系统可用                   |
| 9     | G（Global）            | 全局页，不随 CR3 刷新失效      |
| 8     | PS（Page Size）        | 0=4KB页，1=4MB页（若启用 PSE） |
| 7     | PAT                    | 页面属性表（通常忽略）         |
| 6     | D（Dirty）             | 写入时由硬件置位               |
| 5     | A（Accessed）          | 访问时由硬件置位               |
| 4     | PCD（Cache Disable）   | 禁止缓存该页表中的页           |
| 3     | PWT（Write Through）   | 写直达/回写                    |
| 2     | U/S（User/Supervisor） | 1=用户可访问，0=仅内核         |
| 1     | R/W（Read/Write）      | 1=可读写，0=只读               |
| 0     | P（Present）           | 1=页存在，0=页失效             |

### 2、页表项（`PTE`）

每个页表项占用 **4 字节（32 位）**，结构如下：

| 位数     | 名称             | 描述                                  |
| -------- | ---------------- | ------------------------------------- |
| 31–12 位 | 物理页框地址     | 页框基地址（4KB 对齐）                |
| 11–9 位  | AVL（OS 可用）   | 操作系统可用位                        |
| 第 8 位  | G（Global）      | 是否为全局页，TLB 不因 CR3 刷新而失效 |
| 第 7 位  | PAT              | 页面属性表位（通常忽略）              |
| 第 6 位  | D（Dirty）       | 写入该页时由硬件置 1                  |
| 第 5 位  | A（Accessed）    | 访问该页时由硬件置 1                  |
| 第 4 位  | PCD              | 禁止缓存该页                          |
| 第 3 位  | PWT              | 写直达/回写缓存策略                   |
| 第 2 位  | U/S（用户/内核） | 1=用户模式可访问，0=仅内核            |
| 第 1 位  | R/W              | 1=可写，0=只读                        |
| 第 0 位  | P（Present）     | 页是否存在                            |

### 3、页表项 vs 页目录项

| 项目     | 页目录项（PDE）       | 页表项（PTE）           |
| -------- | --------------------- | ----------------------- |
| 指向对象 | 页表的物理地址        | 页框（物理页）          |
| 地址字段 | [31:12] 页表地址      | **[31:12] 页框地址(*)** |
| 控制位   | 权限、存在等          | 权限、存在、访问、写等  |
| 每项大小 | 4 字节                | 4 字节                  |
| 数量     | 每个页目录 1024 项    | 每个页表 1024 项        |
| 覆盖范围 | 每项映射 4MB 虚拟地址 | 每项映射 4KB 虚拟地址   |

### 4、页目录和页表项的构建

```assembly
section .data.pgdir
.align PGSIZE
__boot_pgdir:
.globl __boot_pgdir
    # map va 0 ~ 4M to pa 0 ~ 4M (temporary)
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space (KERNBASE >> PGSHIFT >> 10 << 2) - (. - __boot_pgdir) # pad to PDE of KERNBASE
    # map va KERNBASE + (0 ~ 4M) to pa 0 ~ 4M
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space PGSIZE - (. - __boot_pgdir) # pad to PGSIZE

.set i, 0
__boot_pt1:
.rept 1024
    .long i * PGSIZE + (PTE_P | PTE_W)
    .set i, i + 1
.endr
```

### 5、开启分页

```assembly
#define REALLOC(x) (x - KERNBASE)

.text
.globl kern_entry
kern_entry:
    # load pa of boot pgdir
    movl $REALLOC(__boot_pgdir), %eax
    movl %eax, %cr3

    # enable paging
    movl %cr0, %eax
    orl $(CR0_PE | CR0_PG | CR0_AM | CR0_WP | CR0_NE | CR0_TS | CR0_EM | CR0_MP), %eax
    andl $~(CR0_TS | CR0_EM), %eax
    movl %eax, %cr0

    # update eip
    # now, eip = 0x1.....
    leal next, %eax
    # set eip = KERNBASE + 0x1.....
    jmp *%eax
next:

    # unmap va 0 ~ 4M, it's temporary mapping
    xorl %eax, %eax
    movl %eax, __boot_pgdir

    # set ebp, esp
    movl $0x0, %ebp
    # the kernel stack region is from bootstack -- bootstacktop,
    # the kernel stack size is KSTACKSIZE (8KB)defined in memlayout.h
    movl $bootstacktop, %esp
    # now kernel stack is ready , call the first C function
    call kern_init
```

#### 5.1、启动页机制过程如下：

1.  将一级页表`__boot_pgdir`(页目录表PDE)的物理地址加载进`%cr3`寄存器
   - 此时该一级页表将虚拟地址0xC0000000+(0~4M)以及虚拟地址(0~4M)都映射到物理地址(0-4M)
   - 为啥将这两段虚拟内存都映射到同一段物理地址呢？见5.2。
2. 设置`%cr0`寄存器中的的PE、PG、AM、WP、NE、MP位，关闭TS与EM位，以启动分页机制
   - PE（Protection Enable）：启动保护模式，默认只是打开分段
   - PG（Paging）：设置分页标志。只有PE和PG同时设置时，才能启动分页机制。
   - WP（Write Protection）：当该位为1时，CPU会禁止ring0（内核态）代码向read only page写入数据。这个标志位主要与**写时复制**有关系。
3. jmp指令，将eip从物理地址“修改”为虚拟地址

#### 5.2、为啥两段虚拟内存需要都映射到同一段物理地址？

​	只有当执行程序至少有部分代码和数据在线性地址空间和物理地址空间中具有相同地址时，我们才能改变PG位的设置。

​	因为因为当`%cr0`寄存器一旦设置，则**分页机制立即开启**。此时这部分具有相同地址的代码在分页和未分页之间起着桥梁的作用。无论是否开启分页机制，这部分代码都必须具有相同的地址。

​	而这一步的操作能否成功，关键就在于一级页表的设置。一级页表将虚拟内存中的两部分地址**KERNBASE+(0-4M)** 与 **(0-4M)** 暂时都映射至物理地址 **(0-4M)** 处，这样就可以满足上述的要求。

#### 5.3、页目录项第一项谁使用了？

# 六、uCore的内存空间布局（*）

