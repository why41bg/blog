---
title: "OS Lab COW" 
date: 2024-07-06T15:53:54+08:00
draft: false
tags:
  - Mit 6.S081
  - OS
ShowToc: true
TocOpen: false 
---

# Copy on write

当 shell 执行一个用户程序时，shell 在背后做了一系列的操作。首先使用 `fork` 系统调用拷贝一份 shell 的内存空间，然后使用 `exec` 系统调用，将需要执行的用户进程的指令、数据等替换新拷贝的内存空间。在这个过程中，如果 `fork` 拷贝的父进程的内存空间很大（在 xv6 中，shell 的内存空间一般是4个 page），那么将花费大量时间，这部分时间实际上来说就是被浪费了的。

Copy on write 做的事情实际上就是在执行 `fork` 系统调用的时候不会拷贝父进程的内存空间，而是将父进程的物理内存地址映射到子进程的 page table 中，并将父进程与子进程的 page table 中的 PTE 的写标志位置为 0，表明不能对这部分物理内存进行写操作。当父进程或子进程对这部分内存进行写操作时，则会触发一个 page fault。

当处理这个 page fault 时，再为这个虚拟地址分配实际的物理内存，将原来物理内存中的内容拷贝过去，然后将这个新分配的物理内存映射到 page table 中，并修改响应的标志位。当完成这一切后，重新返回到用户空间执行引发 page fault 的指令，如果前面的步骤都完成了的话，这时候指令就应该可以正确地运行了。这就是 copy on write 的含义，仅仅在进行写操作时再分配实际的物理内存，这样就可以避免许多无用的拷贝操作。

这里还有两个点需要注意：

1. 操作系统必须能够识别 copy on write 的场景，在 PTE 的标志位中，保留了两位 bit 位，我们就可以使用第七位来标记一个 copy on write 场景。如果不这样做的话，当程序尝试写一个没有写权限的内存时，这可能是一个禁止的操作，而不是一个 copy on write 场景。对于这种错误的情况，操作系统正确的做法就应该是禁止这种操作。
2. 当释放一个物理页的时候，需要格外小心，因为如果采用了 copy on write 机制，一个物理页就有可能不止被一个进程所引用（存在于一个进程的 page table 中）。因此，我们需要记录每个物理页的引用数，当释放一个物理页时，仅仅只是将该物理页的引用数减一，当引用数变为0的时候，说明当前没有进程在使用该物理页了，这时候才真正地去释放物理页。对于每个物理页的引用计数，可以用一个数组来保存，数组的大小为物理内存大小除以4096，也就是物理页的数量。



# Implement copy-on write([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

1. 修改 `vm.c` 中的 `uvmcopy` 函数，这个函数就是 `fork` 系统调用执行的函数，该函数将父进程的物理内存空间拷贝给子进程的物理内存空间，然后将拷贝的物理内存映射到子进程的 page table 中。现在需要修改这个函数，使之不为子进程分配物理内存空间，而是直接将父进程的物理内存空间映射到子进程的 page table 中，然后修改父进程与子进程的 page table 中的标志位，取消写操作权限，标记 copy on write 场景。需要注意的是，父进程的物理页需要将引用数执行加一操作，这个操作的函数在后面会实现，这里先直接使用。具体的代码如下：

	```c
	int
	uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
	{
	    pte_t *pte;
	    uint64 pa, i;
	    uint flags;
	
	    for(i = 0; i < sz; i += PGSIZE){
	      if((pte = walk(old, i, 0)) == 0)  // parent's PTE
	        panic("uvmcopy: pte should exist");
	      if((*pte & PTE_V) == 0)
	        panic("uvmcopy: page not present");
	      pa = PTE2PA(*pte); 
	
	      // ref of pa +1
	      refcnt_add(pa);
	
	      *pte &= (~PTE_W);
	      *pte |= PTE_COW;
	      flags = PTE_FLAGS(*pte);
	      if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
	        goto err;
	      }
	    }
	    return 0;
	    
	    err:
	      uvmunmap(new, 0, i/PGSIZE, 1);
	      return -1;
	}
	```

2. 修改 `tarp.c` 中的 `usertrap` 函数，当出现 page fault 时，执行 copy on write 场景的识别与相应的处理，场景识别的代码放到了函数中。具体的代码如下：

	```c
	void
	usertrap(void)
	{
	  // ...
	    
	  } else if((which_dev = devintr()) != 0){
	    // ok
	  } else if(r_scause() == 15){
	    // page fault
	    uint64 va = r_stval();
	    // copy on write
	    if(cow(p->pagetable, va) < 0)
	      p->killed = 1;
	  } else {
	    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
	    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
	    p->killed = 1;
	  }
	
	  // ...
	}
	```

3. 接下来完成 copy on write 场景处理的函数 `cow`，该函数在 `vm.c` 中，函数实现如下：

	```c
	int
	cow(pagetable_t pgtbl, uint64 va)
	{
	  pte_t *pte;
	  uint64 pa, ka;
	
	  if((pte = walk(pgtbl, va, 0)) == 0)
	    panic("uvmcopy: pte should exist");
	  if((*pte & PTE_V) && (*pte & PTE_U) && (*pte & PTE_COW) == 0)  // is copy on write?
	    return -1;
	  pa = PTE2PA(*pte);
	  if((ka = (uint64)kalloc()) == 0)
	    return -1;
	  memmove((char*)ka, (char*)pa, PGSIZE);
	  uint64 flags = PTE_FLAGS(*pte);
	  *pte = PA2PTE(ka) | flags | PTE_W;
	  *pte &= (~PTE_COW);
	
	  // update count
	  kfree((void *)pa);
	  return 0;
	}
	```

4. 在这个函数的最后，将引起 page fault 的进程执行对应的物理地址的引用数减一，即返回前的 `kfree` 函数。我们需要修改这个函数，仅仅执行将物理页的引用数减一的操作，当引用数为 0 时才真正释放内存。这个函数在 `kalloc.c` 文件中，在这个文件的最开始，还需要定义一个数组，用来记录每个物理页的引用数，该数组定义如下：

	```c
	int refcnt[PHYSTOP >> 12];
	```

	接下来修改 `kfree` 函数，`kfree` 函数完整实现如下：

	```c
	void
	kfree(void *pa)
	{
	  struct run *r;
	
	  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
	    panic("kfree");
	
	  acquire(&kmem.lock);
	  int index = ((uint64)pa) >> 12;
	  if (refcnt[index] >= 1) {
	    refcnt[index] -= 1;
	  }
	  release(&kmem.lock);
	  if (refcnt[index] > 0) {
	    return;
	  }
	
	  // Fill with junk to catch dangling refs.
	  memset(pa, 1, PGSIZE);
	
	  r = (struct run*)pa;
	
	  acquire(&kmem.lock);
	  r->next = kmem.freelist;
	  kmem.freelist = r;
	  release(&kmem.lock);
	}
	```

5. 还需要修改 `kalloc` 函数，在分配物理页的时候，初始化该物理页的引用数为 1，`kalloc` 函数实现如下：

	```c
	void *
	kalloc(void)
	{
	  struct run *r;
	
	  acquire(&kmem.lock);
	  r = kmem.freelist;
	  if(r){
	    kmem.freelist = r->next;
	    refcnt[((uint64)r) >> 12] = 1;
	  }
	  release(&kmem.lock);
	
	  if(r)
	    memset((char*)r, 5, PGSIZE); // fill with junk
	  return (void*)r;
	}
	```

6. 在该文件中，还需要实现在第一步中使用的到的 `refcnt_add` 函数，该函数实现如下：

	```c
	void
	refcnt_add(uint64 pa)
	{
	  acquire(&kmem.lock);
	  refcnt[((uint64)pa) >> 12] += 1;
	  release(&kmem.lock);
	}
	```

7. 根据 plan 的最后一条，还需要修改 `vm.c` 中的 `copyout` 函数，使之一样能识别 copy on write 场景，并执行相应的 copy on write 逻辑，`copyout` 函数实现如下：

	```c
	int
	copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
	{
	  uint64 n, va0, pa0;
	  pte_t *pte;
	
	  while(len > 0){
	    va0 = PGROUNDDOWN(dstva);
	    pa0 = walkaddr(pagetable, va0);
	    if(pa0 == 0)
	      return -1;
	
	    // is copy on write?
	    if((pte = walk(pagetable, va0, 0)) == 0)
	      return -1;
	    if((*pte & PTE_W) == 0)
	      if(cow(pagetable, va0) < 0)
	        return -1;
	    pa0 = PTE2PA(*pte);
	
	    n = PGSIZE - (dstva - va0);
	    if(n > len)
	      n = len;
	    memmove((void *)(pa0 + (dstva - va0)), src, n);
	
	    len -= n;
	    src += n;
	    dstva = va0 + PGSIZE;
	  }
	  return 0;
	}
	```
