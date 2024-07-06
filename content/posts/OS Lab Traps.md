---
title: "OS Lab Traps" 
date: 2024-07-06T16:20:41+08:00
draft: false
tags:
  - Mit 6.S081
  - OS
ShowToc: true
TocOpen: false 
---

> This lab explores how system calls are implemented using traps. 



# RISC-V assembly ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

首先要运行下面的指令生成 `user/call.asm` 文件，需要重点关注其中的 g、f 和 main 三个函数，然后回答下面五个问题。

```bash
make fs.img
```



*Q1*：Which registers contain arguments to functions? For example, which register holds 13 in main's call to `printf` ?

`user/call.ams` 文件中关于 main 函数如下：

```assembly
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7a050513          	addi	a0,a0,1952 # 7c8 <malloc+0xe8>
  30:	00000097          	auipc	ra,0x0
  34:	5f8080e7          	jalr	1528(ra) # 628 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	274080e7          	jalr	628(ra) # 2ae <exit>
```

第 7 行将 13 保存到了 **a2** 寄存器，因此答案为 a2 。



*Q2*：Where is the call to function `f` in the assembly code for main? Where is the call to `g`? (Hint: the compiler may inline functions.)

从代码可以看出，这两个都被内联优化处理了，无函数调用。



*Q3*：At what address is the function `printf` located?

第 1076 行 `0000000000000628 <printf>:`



*Q4*：What value is in the register `ra` just after the `jalr` to `printf` in `main` ?

0x0000000000000038, jalr 指令的下一条汇编指令的地址。



# Backtrace ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> For debugging it is often useful to have a backtrace: a list of the function calls on the stack above the point at which the error occurred.
>

简单来说，该实验就是将函数调用堆栈中的返回地址依次打印出来。

每一个栈帧的结构图如下：

![XV6中栈帧的简单示意图](https://s2.loli.net/2024/07/06/U6sxI58Won3kBKP.png)

栈帧从栈底（高地址处）开始的第一个8字节存储返回地址，第二个 8 字节存储上一个栈帧的栈底（fp）。

关于 RISC-V 中的必要的寄存器说明如下图：

![Regster-info](https://s2.loli.net/2024/07/06/iF3GJILry9DAW5b.png)

其中 `s0` 存储堆栈帧指针，在实验说明中的 hints 中也指出了：“ The GCC compiler stores the frame pointer of the currently executing function in the register `s0` ”。整个实验流程大致分为如下几步：

1. 在 `kernel/defs.h` 中添加 `backtrace` 函数的声明。

2. 在 `kernel/ riscv.h` 中添加一个 `r_fp` 函数用于获取当前 `s0` 寄存器中的值，即当前堆栈的帧指针。`r_fp` 函数实现如下：

	```c
	static inline uint64
	r_fp()
	{
	  uint64 x;
	  asm volatile("mv %0, s0" : "=r" (x) );
	  return x;
	}
	```

3. 在 `kernel/printf` 中定义 `backtrace` 函数的实现。函数实现如下：

	```c
	void
	backtrace(void)
	{
	  printf("backtrace:\n");
	
	  uint64 fp = r_fp();
	
	  uint64 base = PGROUNDUP(fp);  // high address
	  uint64 top = PGROUNDDOWN(fp);  // low address
	
	  while((fp < base) && (top < base)){
	    printf("%p\n", *((uint64*)(fp-8)));
	    fp = *((uint64*)(fp-16));
	  }
	}
	```

4. 在 `kernel/printf` 中的 `panic` 函数中调用 `backtrace` 函数。

5. 由于该实验需用通过 `sys_sleep` 系统调用来调用 `backtrace` 函数，达到评测的目的。因此需要在 `kernel/sysproc.c` 中的 `sys_sleep` 函数中调用 `backtrace` 函数。

最后通过在系统中执行 `bttest` 或者执行测试程序，前者能正确打印出 function calls list，后者能正确通过。



# Alarm ([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> 本实验将自己实现一种原始形式的用户级中断/故障处理程序，类似的方法将用于处理应用程序中的 page fault 。

自己实现两个系统调用 `sigalarm` 和 `sigreturn`，这两个系统调用将在测试程序中被调用。下面按照从用户空间到内核空间最终到达内核程序的过程依次完成代码。

1. 用户空间中定义这两个系统调用，定义在 `user/user.h` 文件中。

	```c
	// system calls
	// ...
	int sigalarm(int ticks, void (*handler)());  // Alarm
	int sigreturn(void);  // Alarm
	```

2. `user/user.pl` 文件中定义入口。

	```perl
	entry("sigalarm");
	entry("sigreturn");
	```

3. 接下来进入内核空间，通过内核空间中的 `syscall` 函数找到对应的内核程序，在这之前，先定义内核程序号。

	```c
	#define SYS_sigalarm  22
	#define SYS_sigreturn 23
	```

	```c
	extern uint64 sys_sigalarm(void);
	extern uint64 sys_sigreturn(void);
	
	[SYS_sigalarm]   sys_sigalarm,
	[SYS_sigreturn]  sys_sigreturn,
	```

4. 这里需要在 `kernel/sysproc.c` 中完成两个内核程序，先来看第一个 `sys_sigalarm`。接收两个参数，第一个是回调函数执行周期，以tick为单位（tick是时钟芯片发出时钟中断的周期），第一个参数是回调函数的地址。也就是说，当一个用户进程每经过 n 次时钟中断后，便触发一个中断处理函数。这里需要注意一点，**在多任务处理环境下，当操作系统需要切换执行不同的线程或进程时，会保存当前线程或进程的寄存器状态到 trapframe 中，并将其存储到相应的数据结构中（如线程控制块或进程控制块）。这样，在切换到下一个线程或进程时，可以从其对应的数据结构中加载 trapframe 中保存的寄存器状态，以恢复到该线程或进程的执行状态。**因此，当首次进入中断处理函数时，我们需要先将当前寄存器的值保存到 trapframe 中的对应位置，再保存到 PCB 中。
	PCB 中需要额外保存的信息就有：触发中断处理函数的周期，当前时钟走过的周期数，是否已执行过中断处理函数（仅第一次执行时需要保存寄存器的值到 PCB 中），中断处理函数的地址，保存的寄存器信息。

	在 `kernel/proc.h` 中的 `proc` 结构体中额外添加如下信息：

	```c
	struct proc {
	    // ...
	    int ticks;                   // Alarm ticks
	    int psdticks;                // Passed alarm ticks
	    uint64 handler;              // Alarm handler pointer
	    int in_handler;              // is in handler?
	
	    uint64 epc;           // saved user program counter
	    uint64 ra;
	    uint64 sp;
	    uint64 gp;
	    uint64 tp;
	    uint64 t0;
	    uint64 t1;
	    uint64 t2;
	    uint64 s0;
	    uint64 s1;
	    uint64 a0;
	    uint64 a1;
	    uint64 a2;
	    uint64 a3;
	    uint64 a4;
	    uint64 a5;
	    uint64 a6;
	    uint64 a7;
	    uint64 s2;
	    uint64 s3;
	    uint64 s4;
	    uint64 s5;
	    uint64 s6;
	    uint64 s7;
	    uint64 s8;
	    uint64 s9;
	    uint64 s10;
	    uint64 s11;
	    uint64 t3;
	    uint64 t4;
	    uint64 t5;
	    uint64 t6;
	}
	```

	在 `kernel/sysproc.c` 文件中添加第一个系统调用：

	```c
	// alarm
	int 
	sys_sigalarm(void)
	{
	  int ticks;
	  uint64 handler;
	
	  if(argint(0, &ticks) < 0)
	    return -1;
	  if(argaddr(1, &handler) < 0)
	    return -1;
	
	  struct proc *p = myproc();
	  p->ticks = ticks;
	  p->handler = handler;
	  p->psdticks = 0;
	  p->in_handler = 0;
	
	  return 0;
	}
	```

5. 修改 `kernel/trapc.c` 中的 `usertrap` 函数，该函数中 `if(which_dev == 2)` 就是判断中断是否来自时钟中断信号，在确定进入中断处理函数前，要保存用户进程中的寄存器信息到 trapframe 中，具体代码如下：

	```c
	// give up the CPU if this is a timer interrupt.
	  if(which_dev == 2){
	    p->psdticks += 1;
	    if(p->ticks && (p->psdticks == p->ticks)){
	      p->psdticks = 0;  // reload ticks counter
	      if(p->in_handler == 0){
	        p->in_handler = 1;  // in handler
	
	        // save regster
	        p->ra = p->trapframe->ra;
	        p->sp = p->trapframe->sp;
	        p->gp = p->trapframe->gp;
	        p->tp = p->trapframe->tp;
	        p->t0 = p->trapframe->t0;
	        p->t1 = p->trapframe->t1;
	        p->t2 = p->trapframe->t2;
	        p->s0 = p->trapframe->s0;
	        p->s1 = p->trapframe->s1;
	        p->a0 = p->trapframe->a0;
	        p->a1 = p->trapframe->a1;
	        p->a2 = p->trapframe->a2;
	        p->a3 = p->trapframe->a3;
	        p->a4 = p->trapframe->a4;
	        p->a5 = p->trapframe->a5;
	        p->a6 = p->trapframe->a6;
	        p->a7 = p->trapframe->a7;
	        p->s2 = p->trapframe->s2;
	        p->s3 = p->trapframe->s3;
	        p->s4 = p->trapframe->s4;
	        p->s5 = p->trapframe->s5;
	        p->s6 = p->trapframe->s6;
	        p->s7 = p->trapframe->s7;
	        p->s8 = p->trapframe->s8;
	        p->s9 = p->trapframe->s9;
	        p->s10 = p->trapframe->s10;
	        p->s11 = p->trapframe->s11;
	        p->t3 = p->trapframe->t3;
	        p->t4 = p->trapframe->t4;
	        p->t5 = p->trapframe->t5;
	        p->t6 = p->trapframe->t6;
	        p->epc = p->trapframe->epc;
	
	        // change PC
	        p->trapframe->epc = p->handler;
	      } 
	```

6. 最后完成 `sys_sigreturn` 函数，将 PCB 中的寄存器信息恢复到 trapframe 中。

	```c
	int 
	sys_sigreturn(void)
	{
	  struct proc *p = myproc();
	
	  if(p->in_handler != 0){
	    // resumption regster
	    p->trapframe->ra = p->ra;
	    p->trapframe->sp = p->sp;
	    p->trapframe->gp = p->gp;
	    p->trapframe->tp = p->tp;
	    p->trapframe->t0 = p->t0;
	    p->trapframe->t1 = p->t1;
	    p->trapframe->t2 = p->t2;
	    p->trapframe->s0 = p->s0;
	    p->trapframe->s1 = p->s1;
	    p->trapframe->a0 = p->a0;
	    p->trapframe->a1 = p->a1;
	    p->trapframe->a2 = p->a2;
	    p->trapframe->a3 = p->a3;
	    p->trapframe->a4 = p->a4;
	    p->trapframe->a5 = p->a5;
	    p->trapframe->a6 = p->a6;
	    p->trapframe->a7 = p->a7;
	    p->trapframe->s2 = p->s2;
	    p->trapframe->s3 = p->s3;
	    p->trapframe->s4 = p->s4;
	    p->trapframe->s5 = p->s5;
	    p->trapframe->s6 = p->s6;
	    p->trapframe->s7 = p->s7;
	    p->trapframe->s8 = p->s8;
	    p->trapframe->s9 = p->s9;
	    p->trapframe->s10 = p->s10;
	    p->trapframe->s11 = p->s11;
	    p->trapframe->t3 = p->t3;
	    p->trapframe->t4 = p->t4;
	    p->trapframe->t5 = p->t5;
	    p->trapframe->t6 = p->t6;
	
	    // resumption PC
	    p->trapframe->epc = p->epc;
	
	    // exit handler
	    p->in_handler = 0;
	  }
	  
	  return 0;
	}
	```

记得把测试程序写到 Makefile 中。

当然，上述保存寄存器信息也可以通过**新建一个 trapframe 对象**，存储到 PCB 中来实现。
