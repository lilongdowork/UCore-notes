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

# 三、段页式存储管理（*）

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

1、启动页机制

2、

# 四、uCore的内存空间布局（*）

# 五、uCore栈的迁移