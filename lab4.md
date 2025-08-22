# 一、线程结构

```c
enum proc_state {
    PROC_UNINIT = 0,  // 未初始化的     -- alloc_proc
    PROC_SLEEPING,    // 等待状态       -- try_free_pages, do_wait, do_sleep
    PROC_RUNNABLE,    // 就绪/运行状态   -- proc_init, wakeup_proc,
    PROC_ZOMBIE,      // 僵死状态       -- do_exit
};
struct context {  // 保存的上下文寄存器，注意没有eax寄存器和段寄存器
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};

struct proc_struct {
    enum proc_state state;          // 当前进程的状态
    int pid;                        // 进程ID
    int runs;                       // 当前进程被调度的次数
    uintptr_t kstack;               // 内核栈
    volatile bool need_resched;     // 是否需要被调度
    struct proc_struct *parent;     // 父进程ID
    struct mm_struct *mm;           // 当前进程所管理的虚拟内存页，包括其所属的页目录项PDT
    struct context context;         // 保存的上下文
    struct trapframe *tf;           // 中断所保存的上下文
    uintptr_t cr3;                  // 页目录表的地址
    uint32_t flags;                 // 当前进程的相关标志
    char name[PROC_NAME_LEN + 1];   // 进程名称（可执行文件名）
    list_entry_t list_link;         // 用于连接list
    list_entry_t hash_link;         // 用于连接hash list
};
```

# 二、进程初始化

### 1、proc_init

```c
// proc_init - set up the first kernel thread idleproc "idle" by itself and 
//           - create the second kernel thread init_main
void
proc_init(void) {
    int i;

    list_init(&proc_list);
    for (i = 0; i < HASH_LIST_SIZE; i ++) {
        list_init(hash_list + i);
    }
	// 分配一个proc_struct结构
    if ((idleproc = alloc_proc()) == NULL) {
        panic("cannot alloc idleproc.\n");
    }
    // 将空闲进程作为第一个进程，pid = 0
    idleproc->pid = 0;
    idleproc->state = PROC_RUNNABLE;
    idleproc->kstack = (uintptr_t)bootstack;
    idleproc->need_resched = 1;
    set_proc_name(idleproc, "idle");
    nr_process ++;

    current = idleproc;
	// 创建第二个进程，init主进程（第一个内核进程）
    int pid = kernel_thread(init_main, "Hello world!!", 0);
    if (pid <= 0) {
        panic("create init_main failed.\n");
    }

    initproc = find_proc(pid);
    set_proc_name(initproc, "init");

    assert(idleproc != NULL && idleproc->pid == 0);
    assert(initproc != NULL && initproc->pid == 1);
}
```

### 2、kernel_thread

```c
// kernel_thread - create a kernel thread using "fn" function
// NOTE: the contents of temp trapframe tf will be copied to 
//       proc->tf in do_fork-->copy_thread function
int
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    tf.tf_regs.reg_ebx = (uint32_t)fn;
    tf.tf_regs.reg_edx = (uint32_t)arg;
    tf.tf_eip = (uint32_t)kernel_thread_entry;
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}

// entry.S
.text
.globl kernel_thread_entry
kernel_thread_entry:        # void kernel_thread(void)

    pushl %edx              # push arg
    call *%ebx              # call fn

    pushl %eax              # save the return value of fn(arg)
    call do_exit            # call do_exit to terminate current thread

// 1、新线程被调度 → 从 kernel_thread_entry 开始执行
// 2、取出创建时保存的 %ebx = fn，%edx = arg
// 3、调用 fn(arg)
// 4、拿到返回值 retval
// 5、调用 do_exit(retval)，退出线程并交还 CPU
```

### 3、do_fork

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS)
        goto fork_out;
    ret = -E_NO_MEM;

    // 首先分配一个PCB
    if ((proc = alloc_proc()) == NULL)
        goto fork_out;
    // fork肯定存在父进程，所以设置子进程的父进程
    // 当一个进程调用 fork() 时，它就是发起者，也就是子进程的父进程。
    proc->parent = current;
    // 分配内核栈
    // 保证每个进程内核态运行独立，安全
    if (setup_kstack(proc) != 0)
        goto bad_fork_cleanup_proc;
    // 将所有虚拟页数据复制过去
    if (copy_mm(clone_flags, proc) != 0)
        goto bad_fork_cleanup_kstack;
    // 复制线程的状态，包括寄存器上下文等等
    copy_thread(proc, stack, tf);
    // 将子进程的PCB添加进hash list或者list
    // 需要注意的是，不能让中断处理程序打断这一步操作
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    local_intr_restore(intr_flag);
    // 设置新的子进程可执行
    wakeup_proc(proc);
    // 返回子进程的pid
    ret = proc->pid;

fork_out:
    return ret;
bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

### 4、get_pid

```c
// get_pid - alloc a unique pid for process
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                // 缩小边界
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
// next_safe 安全边界
```

### 4、cpu_idle

```c
// cpu_idle - at the end of kern_init, the first kernel thread idleproc will do below works
void
cpu_idle(void) {
    while (1) {
        if (current->need_resched) {
            schedule();
        }
    }
}
```

### 5、schedule

```c
void
schedule(void) {
    bool intr_flag;
    list_entry_t *le, *last;
    struct proc_struct *next = NULL;
    local_intr_save(intr_flag);
    {
        current->need_resched = 0;
        // 决定进程链表调度起点
        last = (current == idleproc) ? &proc_list : &(current->list_link);
        le = last;
        do {
            if ((le = list_next(le)) != &proc_list) {
                next = le2proc(le, list_link);
                if (next->state == PROC_RUNNABLE) {
                    break;
                }
            }
        } while (le != last);
        if (next == NULL || next->state != PROC_RUNNABLE) {
            next = idleproc;
        }
        next->runs ++;
        if (next != current) {
            proc_run(next);
        }
    }
    local_intr_restore(intr_flag);
}
```

### 6、proc_run

```c
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```

### 7、switch_to（***）

调用逻辑为switch_to(&(prev->context), &(next->context))，参考C调用函数逻辑，对参数和返回地址的处理

```assembly
.text
.globl switch_to
switch_to:                      # switch_to(from, to)

    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    // 将返回地址出栈，赋值给from->eip
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)          # save esp::context of from
    movl %ebx, 8(%eax)          # save ebx::context of from
    movl %ecx, 12(%eax)         # save ecx::context of from
    movl %edx, 16(%eax)         # save edx::context of from
    movl %esi, 20(%eax)         # save esi::context of from
    movl %edi, 24(%eax)         # save edi::context of from
    movl %ebp, 28(%eax)         # save ebp::context of from

    # restore to's registers
    // eas指向to
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp         # restore ebp::context of to
    movl 24(%eax), %edi         # restore edi::context of to
    movl 20(%eax), %esi         # restore esi::context of to
    movl 16(%eax), %edx         # restore edx::context of to
    movl 12(%eax), %ecx         # restore ecx::context of to
    movl 8(%eax), %ebx          # restore ebx::context of to
    movl 4(%eax), %esp          # restore esp::context of to
	
	// 将to-eip放在栈顶（返回地址）
	// eip指向forkret(在copy_thread时进行的处理)
    pushl 0(%eax)               # push eip

    ret
```

### 8、forkret

当该子进程被调度运行，上下文切换后（即此时current为该子进程的PCB地址），子进程会跳转至`forkret`，而该函数是`forkrets`的一个wrapper

```c
// forkret -- the first kernel entry point of a new thread/process
// NOTE: the addr of forkret is setted in copy_thread function
//       after switch_to, the current proc will execute here.
static void
forkret(void) {
    forkrets(current->tf);
}
```

forkrets是干什么用的呢？从current->tf中恢复上下文，跳转至current->tf->tf_eip，

**如果是用户进程 fork 出来的子进程** → `iret` 跳到 `tf->eip = 父进程被中断时的 eip`（所以子进程像从 `fork()` 返回）。

**如果是新建的内核线程** → `iret` 跳到 `tf->eip = kernel_thread_entry`，开始跑线程函数。

```assembly
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
    // 此时eip为栈顶指针
    iret

.globl forkrets
forkrets:
    # set stack to this new process's trapframe
    movl 4(%esp), %esp
    jmp __trapret
```

### 9、kernel_thread_entry

`kernel_thread`函数设置控制流起始地址为`kernel_thread_entry`的目的，是想让一个内核进程在执行完函数后能够**自动调用`do_exit`回收资源**

```assembly
.text
.globl kernel_thread_entry
kernel_thread_entry:        # void kernel_thread(void)

    pushl %edx              # push arg
    call *%ebx              # call fn

    pushl %eax              # save the return value of fn(arg)
    call do_exit            # call do_exit to terminate current thread
```

