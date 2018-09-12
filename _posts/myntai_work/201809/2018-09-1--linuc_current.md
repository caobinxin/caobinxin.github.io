# 内核 current宏解析

在内核中，可以通过current宏来获得当前执行进程的task_struct指针。现在来简要分析以下：

​      最原始的定义如下：

```cpp
   #define current get_current()
   #define get_current() (current_thread_info()->task)
   可以看出，current调用了 current_thread_info函数，此函数的内核路径为： arch/arm/include/asm/thread_info.h，内核版本为2.6.32.65
```

     static inline struct thread_info *current_thread_info(void) 
    { 
         register unsigned long sp asm ("sp"); 
         return (struct thread_info *)(sp & ~(THREAD_SIZE - 1)); 
    }

​    其中 thread_info结构体如下：

```cpp
struct thread_info {
	unsigned long		flags;		/* low level flags */
	int			preempt_count;	/* 0 => preemptable, <0 => bug */
	mm_segment_t		addr_limit;	/* address limit */
	struct task_struct	*task;		/* main task structure */
	struct exec_domain	*exec_domain;	/* execution domain */
	__u32			cpu;		/* cpu */
	__u32			cpu_domain;	/* cpu domain */
	struct cpu_context_save	cpu_context;	/* cpu context */
	__u32			syscall;	/* syscall number */
	__u8			used_cp[16];	/* thread used copro */
	unsigned long		tp_value;
	struct crunch_state	crunchstate;
	union fp_state		fpstate __attribute__((aligned(8)));
	union vfp_state		vfpstate;
#ifdef CONFIG_ARM_THUMBEE
	unsigned long		thumbee_state;	/* ThumbEE Handler Base register */
#endif
	struct restart_block	restart_block;
};
```

​     当内核线程执行到此处时，其SP堆栈指针指向调用进程所对应的内核线程的栈顶。**通过 sp & ~(THREAD_SIZE-1)向上对齐，达到栈底部**。如下图所示

​      ![image](http://images.cnitblog.com/blog/350213/201501/281409208002531.png)

​	上面的这个图，我cbx认为有两个错误：

* 将  **栈底部**  放置到 sp & ~(THREAD_SIZE-1) 旁边，
* 然后第一步修改后的结果，整个来一个翻转，因为ARM使用的是  **满减栈**



   将结果强制类型转换为thread_info类型，此类型中有一个成员为task_struct，它就是 当前正在运行进程的 task_struct指针。

   

 备注：

​     在内核中，进程的task_struct是由slab分配器来分配的，slab分配器的优点是对象复用和缓存着色。

​     联合体：

​     \#define THREAD_SIZE        8192       //内核线程栈 可以通过内核配置成4K 或者 8K ，此处是8K   。**在X86体系结构上，32位的内核栈为8K，64位的为16K。**

     union thread_union { 
     struct thread_info thread_info;                            // sizeof(thread_info) = 
     unsigned long stack[THREAD_SIZE/sizeof(long)];     //stack 大小为 8K，**union联合体的地址是严格按照小端排布的，因此，内核栈的低位地址是thread_info结构体。  **
     };

​      整个8K的空间，顶部供进程堆栈使用，最下部为thread_info。从用户态切换到内核态时，进程的内核栈还是空的，所以sp寄存器指向栈顶，一旦有数据写入，sp的值就会递减，内核栈按需扩展，理论上最大可扩展到  【8192- sizeof(thread_info)  】大小，考虑到函数的现场保护，往往不会有这么大的栈空间。内核在代表进程执行时和所有的中断服务程序执行时，共享8K的内核栈。

## 总结

* current 是一个宏，他就是struct task_struct 的**指针**
* 通过移动 栈顶的sp(栈指针)  移动到 栈底，先找到 thread_info  然后拿出其中的 task_struct结构体
* 拿 thread_info 的手段是   **sp & ~(THREAD_SIZE-1)** 