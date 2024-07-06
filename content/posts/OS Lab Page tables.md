---
title: "OS Lab Page Tables" 
date: 2024-07-06T16:16:35+08:00
draft: false
tags:
  - Mit 6.S081
  - OS
ShowToc: true
TocOpen: false 
---

> 很喜欢 Frans 教授的开场白：当我还是个学生并第一次听到学到这个词（pagetable）时，我认为它还是很直观简单的。这能有多难呢？无非就是个表单，将虚拟地址和物理地址映射起来，实际可能稍微复杂一点，但是应该不会太难。可是当我开始通过代码管理虚拟内存，我才知道虚拟内存比较棘手，比较有趣，功能也很强大。
>
> 这部分实验也证明了 pagetable 确实如此。



# 用户进程的地址空间

A process's user address space, with its initial stack.

![用户进程的地址空间](https://s2.loli.net/2024/07/06/fdYv7tSVPOal6CH.png)

这是XV6中关于用户进程的地址空间定义，在下面的实验的会用到。



# Speed up system calls ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. 
>

每个进程创建时，都会创建一个属于它自己的虚拟地址空间（页表），该地址空间的结构图在上面也给出来了，主要有三个部分：TRAMPOLINE、TRAPFRAME、USYSCALL（图中未给出，应该位于 TRAPFRAME 前面）。而 USYSCALL 就是映射的与内核共享的**只读**区域。用户进程的虚拟地址空间布局在 `kernel/memlayout.h` 文件中，这里面同时还定义了一个 `usyscall` 结构体用来存储用户进程的 PID。相关定义如下：

```c
// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   USYSCALL (shared with kernel)
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
#endif
```

按照提示就能够完成该实验，详细的步骤说明如下：

1. 创建用户进程的页表，创建页表的函数在 `kernel/proc.c` 中，函数名为 `proc_pagetable`。
	仿照 TRAMPOLINE 和 TRAPFRAME 的样子为 USYSCALL 分配 PTE：

	``` c
	pagetable_t
	proc_pagetable(struct proc *p)
	{
	      // ...
	      // map the trapframe just below TRAMPOLINE, for trampoline.S.
	      if(mappages(pagetable, TRAPFRAME, PGSIZE,
	                  (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
	        uvmunmap(pagetable, TRAMPOLINE, 1, 0);
	        uvmfree(pagetable, 0);
	        return 0;
	      }
	
	      // map the USYSCALL just below trapframe
	      if(mappages(pagetable, USYSCALL, PGSIZE, 
	                  (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
	        uvmunmap(pagetable, TRAMPOLINE, 1, 0);
	        uvmunmap(pagetable, TRAPFRAME, 1, 0);
	        uvmfree(pagetable, 0);
	        return 0;
	      }
	
	      return pagetable;
	}
	```

	

2. 为用户进程分配实际的物理内存，同时初始化。相关函数为 `kernel/proc.c` 中的 `allocproc`。
	仿照为 TRAPFRAME 分配内存的方式为 USYSCALL 分配实际的物理内存，**不要忘记对 `USYSCALL` 初始化**：

	```c
	static struct proc*
	allocproc(void)
	{
	    // ...
	    // Allocate a trapframe page.
	    if((p->trapframe = (struct trapframe *)kalloc()) == 0){
	      freeproc(p);
	      release(&p->lock);
	      return 0;
	    }
	
	    // Allocate a usyscall page.
	    if((p->usyscall = (struct usyscall *)kalloc()) == 0){
	      freeproc(p);
	      release(&p->lock);
	      return 0;
	    }
	    p->usyscall->pid = p->pid;
	    // ...
	}
	```

	

3. 完善 `kernel/proc.c` 中与释放物理内存有关的函数 `freeproc`。

	```c
	static void
	freeproc(struct proc *p)
	{
	    // ...
	    if(p->usyscall)
	    	kfree((void*)p->usyscall);
	  	p->usyscall = 0;
	    // ...
	}
	```

	

4. 完善 `kernel/proc.c` 中释放页表有关的函数 `proc_freepagetable`。

	```c
	void
	proc_freepagetable(pagetable_t pagetable, uint64 sz)
	{
	  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
	  uvmunmap(pagetable, TRAPFRAME, 1, 0);
	  uvmunmap(pagetable, USYSCALL, 1, 0);
	  uvmfree(pagetable, sz);
	}
	```

该实验最后还提出了一个问题，**Which other xv6 system call(s) could be made faster using this shared page?** 答案是：fork 系统调用。只需要在 `usyscall` 结构体中加入父进程的 `parent` 结构体，即可在 fork 系统调用复制父进程内存空间时，无需切换到内核即可找到父进程实际的物理内存地址，从而完成父进程内存空间的复制。



# Print a page table ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> 此实验要求打印一个进程的页表，目的在于帮助我们理解XV6中三级页表的结构。

任务要求如下：在 `kernel/vm.c` 中定义一个  `vmprint` 函数，该函数接收一个 `pagetable_t`  参数，然后格式化输出该页表的内容。

有一点提示很关键：**The function `freewalk` may be inspirational.** `kernel/vm.c` 中的 `freewalk` 函数通过递归的方式对进程页表进行了处理。因此，我们可以参考这个函数实现来 `vmprint`。

具体的流程如下：

1. 在 `kernel/vm.c` 中实现  `vmprint` 函数。

	```c
	static char* prefix[3] = {"..", ".. ..", ".. .. .."};
	// ...
	
	// Cycle through the contents of a page table
	// Reference freewalk() function
	void
	vmprint(pagetable_t pgtbl)
	{
	  printf("page table %p\n", pgtbl);
	  // Entry recursion to print pgtbl
	  vmprint_re(pgtbl, 0);
	}
	
	void
	vmprint_re(pagetable_t pgtbl, int layer){
	  // End condition
	  if(layer > 2){
	    return;
	  }
	
	  // There are 2^9 = 512 PTEs in a page table.
	  for(int i = 0; i < 512; i++){
	    pte_t pte = pgtbl[i];
	    if((pte & PTE_V)){
	      uint64 pa = PTE2PA(pte);
	      printf("%s%d: pte %p pa %p\n", prefix[layer], i, pte, pa);
	      vmprint_re((pagetable_t)pa, layer+1);
	    }
	  }
	}
	```

	

2. 在 `kernel/defs.h` 中声明  `vmprint` 函数。

	```c
	// vm.c
	// ...
	int             copyinstr(pagetable_t, char *, uint64, uint64);
	void            vmprint(pagetable_t);
	void            vmprint_re(pagetable_t, int);
	```

	

3. 在 `kernel/exec.c` 中的 `exec` 函数结束前，调用 `vmprint` 函数打印进程的页表。

	```c
	int
	exec(char *path, char **argv)
	{
	    // ...
	    
	    // Print the page table before the end of the exec
	  	vmprint(p->pagetable);
	  	return argc; // this ends up in a0, the first argument to main(argc, argv)
	    
	    // ...
	}
	```




# Detecting which pages have been accessed ([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

>Some garbage collectors (a form of automatic memory management) can benefit from information about which pages have been accessed (read or write). In this part of the lab, you will add a new feature to xv6 that detects and reports this information to userspace by inspecting the access bits in the RISC-V page table. The RISC-V hardware page walker marks these bits in the PTE whenever it resolves a TLB miss.
>

这个实验看似很难，其实还好。需要实现一个内核程序（系统调用），扫描页表中哪些页表项的标志位 PTE_A 被标记为了 1。

在测试程序的代码中，就为我们初始化了一个页表，并将其中某些PTE的标志位 PTE_A 标记为了1。测试例程代码如下：

```c
void
pgaccess_test()
{
  char *buf;
  unsigned int abits;
  printf("pgaccess_test starting\n");
  testname = "pgaccess_test";
  buf = malloc(32 * PGSIZE);
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  buf[PGSIZE * 1] += 1;
  buf[PGSIZE * 2] += 1;
  buf[PGSIZE * 30] += 1;
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  if (abits != ((1 << 1) | (1 << 2) | (1 << 30)))
    err("incorrect access bits set");
  free(buf);
  printf("pgaccess_test: OK\n");
}
```

那么 PTE_A 在 PTE 的哪个位置呢？在 PTE 从右到左第 7 个（index 为 6）比特位，提示中也说明了需要在 `kernel/riscv.h` 中设置宏定义：

```c
// ...

#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_A (1L << 6) // 1 -> page has been accessed
```

如何找到虚拟地址对应的最后一级 PTE 呢？关键就是 `walk` 函数，该函数可以根据虚拟地址找到对应的 PTE。因此，整个流程就应该为：

1. 解析用户程序参数，获得初始虚拟地址，扫描页表的上限，一个用户空间的缓存区（store the results into a bitmask）。
2. 从初始地址开始检测，通过 `walk` 函数获得其对应 PTE。如果 PTE_A 为 1，则将内核缓冲区中的 bitmask 对应位置设置为 1，然后清空 PTE中的 PTE_A。
3. 循环第 2 步至上限。
4. 利用 `copyout` 函数将内核空间的缓冲区复制到用户空间。

因此，`sys_pgaccess` 函数的完整实现为：

```c
#ifdef LAB_PGTBL
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 va;  // The starting virtual address of the first user page to check.
  int pgnum;  // The number of pages to check
  uint64 bitmask;  // Use one bit per page
  if(argaddr(0, &va) < 0){
    return -1;
  }
  if(argint(1, &pgnum) < 0){
    return -1;
  }
  if(argaddr(2, &bitmask) < 0){
    return -1;
  }

  if(pgnum > 64){
    return -1;
  }

  struct proc *p = myproc();
  uint64 kbuf = 0; // Kernel buffer
  pte_t *pte;

  for(int i = 0; i < pgnum; i++){
    if(va > MAXVA)
      return -1;

    pte = walk(p->pagetable, va, 0);
    if(pte == 0)
      return -1;

    if(*pte & PTE_A){
      kbuf |= (1 << i);  // Set the page bit in kernel buffer
      *pte &= (~PTE_A);
    }

    va += PGSIZE;  // To the next page
  }

  if(copyout(p->pagetable, bitmask, (char*)&kbuf, sizeof(kbuf)) < 0)
    return -1;

  return 0;
}
#endif
```

最后还有一个很关键的点，但提示中没有说：**在 `kernel/defs.h` 中声明一下 `walk` 函数**，我也不知道为什么没有声明🥴🥴。
