#ucore在v9-CPU上的移植

邓志会 2015210926 人工智能所

对于
[https://github.com/leopard1/v9-ucore/tree/dev]
给出的代码和示例进行了学习，实验并且给出了自己的思考，通过这次实验进一步的了解了cpu，编译器，操作系统,它们之间是如何互动的，因为能力和时间有限，没有进行改进和创新，通过老师的课程扩展了自己的视野，希望自己一步一个脚印，不断积累。

## 概述

swieros是v9-CPU的原型，作者设计了一个兼具RISC和SISC特性的指令集，并为指令集实现了一个简化的C编译器以及相应的模拟器。而ucore是清华大学操作系统课程采用的教学操作系统，这套操作系统具有较为完善的抽象层次和代码结构，操作系统被拆分成了8个lab，每个lab都在前一个lab上进行了一些修改和加入了新的功能。

	gcc -o xc -O3 -m32 -Ilinux -Iucore/lib ucore/tool/c.c
	gcc -o xem -O3 -m32 -Ilinux -Iucore/lib ucore/tool/em.c -lm
	cpp-5 -Iucore/lib -Iucore/kern/include -Iucore/kern/libs -Iucore/kern/mm -Iucore/kern/fs -Iucore/kern/driver -Iucore/kern/sync -Iucore/kern/trap -Iucore/kern/process -Iucore/kern/schedule -Iucore/kern/syscall ucore/kern/main.c ucore.c
	./xc -v -o ucore.bin ucore.c 
	./xem ucore.bin

在执行过程中先编译生成编译器xc，再编译生成模拟器xem，再把所有的程序编译整合到一个文件ucore.c中。再使用xc来整合程序文件。再使用xxc来编译为操作系统启动文件，最后在c9-cpu的模拟器xem上面运行。


###Lab1-Lab2

为了完成物理内存管理，以固定页面大小来划分整个物理内存空间，并准备以此为最小内存分配单位来管理整个物理内存，管理在内核运行过程中每页内存，设定其可用状态（free的，used的，还是reserved的），这其实就是连续内存分配概念和原理的具体实现；接着ucore kernel就要建立页表， 启动分页机制，让CPU的MMU把预先建立好的页表中的页表项读入到TLB中，根据页表项描述的虚拟页（Page）与物理页帧（Page Frame）的对应关系完成CPU对内存的读、写和执行操作。这一部分其实就对应了内存映射、页表、多级页表等概念和原理。

init_pmm_manager(void) 初始化物理内存页管理器框架pmm_manager；alloc_pages(size_t n) 分配页面内存，free_pages(struct Page *base, size_t n) 释放连续页面的物理内存，
建立空闲的page链表，这样就可以分配以页（4KB）为单位的空闲内存了；
检查物理内存页分配算法；
get_pte(pde_t *pgdir, uintptr_t la, bool create获取pte返回内核中的虚拟地址
get_page(pde_t *pgdir, uintptr_t la, pte_t **ptep_store) 获取相关页的线性地址
check_pgdir(void) 检查页表建立是否正确；




在atomic.h中主要是置位函数，比如set_bit,clear_bit,change_bit,test_bit

在call.h中主要是函数指针调用的函数定义，在def.h是ucore的相关定义，errors.h是ucore的错误定义，list.h定义了双向链表的实现，io.h定义了输入输出的相关操作，string.h定义了string以及内存复制拷贝的底层操作定义，default_pmm.h定义了物理页分配算法实现，memlayout.h定义了物理空间的构成定义，mmu.h是物理地址操作的定义，pmm.h定义了物理地址相关操作的实现，trap.h定义了陷阱操作的相关实现，main.c定义了操作系统程序入口


在trap.h中定义了相关的寄存器入栈，陷阱定义，其中pad之类的是用以填充空间的

下面定义了不同的处理器错误代码

	enum {    	// processor fault codes
	    FMEM,   // bad physical address
	    FTIMER, // timer interrupt
	    FKEYBD, // keyboard interrupt
	    FPRIV,  // privileged instruction
	    FINST,  // illegal instruction
	    FSYS,   // software trap
	    FARITH, // arithmetic trap
	    FIPAGE, // page fault on opcode fetch
	    FWPAGE, // page fault on write
	    FRPAGE, // page fault on read
	    USER=16 // user mode exception
	};

trap的类型，即下文fc(fault code)的取值有上述几种可能性，内核和用户的区分就是通过USER来进行的，例如fc为FTIMER表示内核栈的时钟中断，fc为USER+FTIMER表示用户栈的时钟中断

void alltraps()在中断发生时会将当前的寄存器状况和中断类型、指令地址等压入内核栈中，用以在中断处理完成后能够恢复出现中断的现场寄存器环境从而继续进行下去

void idt_init() 将alltraps函数地址设置为中断发生时的跳转地址，换句话说，中断发生时会跳到alltraps进行中断操作

对于系统的内存分布memlayout.h，32位机器一个能够分配4G的空间，其中内核程序都应放置到0xC0000000以上部分，页表放置在0xFAC00000开始的连续空间内，事实上在页机制未启动时所有的段都是分布在0x00000000上的，且物理地址等于逻辑地址，在pmm_init中要做的就是初始化页表使得操作系统的各段的虚拟地址变为0xC0000000，或者说是0xC0000000指向物理地址0x00000000。


在物理内存管理中，page_init(void) 获取物理内存大小，计算所有的物理内存一共能划分出的页数，初始化所有的页的标志位，对于每个可以分配的页都将建立一个页头结构来管理，即struct Page，从pages之后连续的若干个struct Page将所有能分配的物理页串联起来用以分配算法，这里就是初始化部分

在启动页表机制之后，页表的虚拟地址不再是0x00000000向上的部分而是0xC0000000向上的部分，所以加上KERNBASE，即0xC0000000，在完成过渡之后，所有的指令地址都处于0xC0000000向上，此时0x00000000和0xC0000000映射到相同的物理内存已经不需要了，在页机制算法初始化时函数指针也是0x00000000向上的部分，重新植入函数指针将函数指针指向到0xC0000000向上的部分，在页表的管理头里的指针都是0x00000000为基址时的状态，在页机制启动后，其中的指针值并不会随之改变，要遍历一次进行更新

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab1.png" style="width: 50%; height: 50%"/>​

该实验实现了时钟，中断和物理内存管理。

###Lab3

磁盘构建

*	v9底层是没有成熟的磁盘接口的，所以在这里我们在内存中开出了一块空间来构建了一块虚拟磁盘。

```
static char disk[1 * 1024 * 8 * 512];	//在bss段中开出空间构建一块虚拟磁盘作为交换分区

void
ide_init(void) {
  memset(disk, sizeof(disk), 0);		//磁盘初始化即是将bss段中的虚拟磁盘区域清空
}

size_t
ide_device_size(unsigned short ideno) {
  return 1 * 1024 * 8;
}

int										//从磁盘中读则是直接从虚拟磁盘中将内容拷贝进目标地址dst中
ide_read_secs(unsigned short ideno, uint32_t secno, void *dst, size_t nsecs) {
  int ret = 0;
  int i;
  secno = secno * SECTSIZE;
  for (; nsecs > 0; nsecs--, dst += SECTSIZE, secno += SECTSIZE) {
    for (i = 0; i < SECTSIZE; i++) {
      *((char *)(dst) + i) = disk[secno + i];
    }
  }
  return ret;
}

int										//往磁盘中写则是直接从来源地址src中向虚拟磁盘中拷贝内容
ide_write_secs(unsigned short ideno, uint32_t secno, uint32_t *src, size_t nsecs) {
  int ret = 0;
  int i;
  secno = secno * SECTSIZE;
  for (; nsecs > 0; nsecs--, src += SECTSIZE, secno += SECTSIZE) {
    for (i = 0; i < SECTSIZE; i++) {
      disk[secno + i] = *((char *)(src) + i);
    }
  }
  return ret;
}

```

####*	页缺失

```
int
do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
 

int pgfault_handler(struct trapframe *tf) {
    
```


<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab1.png" style="width: 50%; height: 50%"/>​

该实验实现了虚拟内存页面错误时的异常处理算法的实现，还有一些页面替代算法，虚拟内存的管理

###Lab4-Lab5

Lab4-Lab5分别启动了内核线程和用户进程。进程管理包含进程控制块(pstruct.h)和进程控制(proc.h)及进度调度算法(sched.h)。(1) 进程控制块包括进程的基本信息，用于操作系统管理进程。(2) 进程控制包含了进程的fork, execve, exit等方法，相应的会涉及到一些数据结构和对进程控制块的修改，内存分配、中断等问题。(3)调度算法是进程切换的策略。


切换内核栈，由于汇编指令的差异，需要修改变换内核栈的代码。之前core还一直在核心态“打转”，没有到用户态执行。提供各种操作系统功能的内核线程只能在CPU核心态运行是操作系统自身的要求，操作系统就要呆在核心态，才能管理整个计算机系统。但应用程序员也需要编写各种应用软件，且要在计算机系统上运行。如果把这些应用软件都作为内核线程来执行，那系统的安全性就无法得到保证了。所以，ucore要提供用户态进程的创建和执行机制，给应用程序执行提供一个用户态运行环境。从ucore的初始化部分来看，会发现初始化的总控函数kern_init没有任何变化。但这并不意味着lab4与lab5差别不大。其实kern_init调用的物理内存初始化，进程管理初始化等都有一定的变化。

在进程管理方面，主要涉及到的是进程控制块中与内存管理相关的部分，包括建立进程的页表和维护进程可访问空间（可能还没有建立虚实映射关系）的信息；加载一个ELF格式的程序到进程控制块管理的内存中的方法；在进程复制（fork）过程中，把父进程的内存空间拷贝到子进程内存空间的技术。另外一部分与用户态进程生命周期管理相关，包括让进程放弃CPU而睡眠等待某事件；让父进程等待子进程结束；一个进程杀死另一个进程；给进程发消息；建立进程的血缘关系链表。

void switch_to(struct context *oldc, struct context *newc) 切换用户态和核心态的协议栈


复制进程，在这一部分，造成差异的是寄存器的不同。

static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {

binary是二进制的虚地址指针，bsize是无效的参数
load_icode(unsigned char *binary, size_t bsize) 

  


<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab4-51.png" style="width: 50%; height: 50%"/>​


<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab4-52.png" style="width: 50%; height: 50%"/>​

该实验实现了线程和进程的创建启动。


###Lab6

调度本质上体现了对CPU资源的抢占。对于用户进程而言，由于有中断的产生，可以随时打断用户进程的执行，转到操作系统内部，从而给了操作系统以调度控制权，让操作系统可以根据具体情况（比如用户进程时间片已经用完了）选择其他用户进程执行。

实验6实现了进程之间的调度  实现时候主要是在schedule文件夹中，使用队列来存储进程的handle，用时间片进行调度

	void sched_class_enqueue(struct proc_struct *proc) 

	void sched_class_dequeue(struct proc_struct *proc) 

	struct proc_struct* sched_class_pick_next(void) 

	void sched_class_proc_tick(struct proc_struct *proc) 

	void sched_init(void) 

	void wakeup_proc(struct proc_struct *proc) 

	void schedule(void) 

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab6.png" style="width: 50%; height: 50%"/>​

###lab7

在时钟片列表中添加定时器

	// add timer to timer_list
	void
	add_timer(timer_t *timer)

	  local_intr_save(intr_flag);
	  

在时钟片列表中删除定时

	// del timer from timer_list
	void
	del_timer(timer_t *timer) {
	  

	调用调度程序看时钟片是否过期

	// call scheduler to update tick related info, and check the timer is expired? If expired, then wakup proc
	void
	run_timer_list(void) {


<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab7.png" style="width: 50%; height: 50%"/>​

###Lab8
Lab8加入了文件系统，主要实现实在fs文件夹下面，其中fsstruct.h中存储了文件系统中的数据结构，如盘符大小secSize,inode的内存内部复制，dinode存储了磁盘上的inode数据结构，direct是一个存储一系列直接数据结构的目录。console.h可以用console端口与用户交互，而在fs.h中存储了binit()函数创建缓冲区链接列表，fs_init存储了初始化的文件存储,bread函数，brelease函数，ilock函数，iget函数，bwrite函数，balloc函数来实现文件系统。


struct { 
 uint magic //0xC0DEF00D 文件的magic number 
 uint bss　 //在v9-cpu执行时没有用到
 uint entry //程序的执行入口地址
 uint flags;//程序的数据段起始地址 
} hdr;

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/lab8.png" style="width: 50%; height: 50%"/>​

