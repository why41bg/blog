---
title: "OS Lab Multithreading" 
date: 2024-07-06T16:04:03+08:00
draft: false
tags:
  - Mit 6.S081
  - OS
ShowToc: true
TocOpen: false 
---

> This lab will familiarize you with multithreading. You will implement switching between threads in a user-level threads package, use multiple threads to speed up a program, and implement a barrier.
>
> 关于这节实验，我找到一个很棒的[视频](https://www.bilibili.com/video/BV1rS4y1n7y1/?p=10&share_source=copy_web&vd_source=648b3a8f2dffc97d9c07c0e073ceb75b)，希望能帮助到你。



# Uthread: switching between threads ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it. 
>

实验的提示已经说得很清晰了，我们只需要修改 `user/uthread.c` 中的 `thread_scheduled` 函数和 `thread_create`函数，和 `user/uthread_switch.S` 文件。代码中需要修改的地方及修改目标都已经说得很清楚了，理解了线程切换的原理后，跟着做就很简单了。

首先在 `user/uthread.c` 文件中需要添加一个 `context` 结构体，然后将这个结构体添加到 `thread` 结构体中，代码如下：

```c
struct thread_context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
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
};

struct thread {
  char            stack[STACK_SIZE]; /* the thread's stack */
  int             state;             /* FREE, RUNNING, RUNNABLE */
  struct thread_context context;     /* thread's context*/
};
```

然后是修改 `thread_scheduled` 函数，只需要在该函数中调用一下 `thread_switch` 函数，传入两个 context 的指针即可，代码如下：

```c
void 
thread_schedule(void)
{
	// ...
    
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&next_thread->context);
  } else
    next_thread = 0;
}
```

接下来修改 `thread_create` 函数，修改如下：

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)&t->stack[STACK_SIZE - 1];
  t->context.s0 = (uint64)&t->stack[STACK_SIZE - 1];
}
```

最后是 `uthread_switch.S` 文件，这个在课中讲得已经很清楚了，该文件代码如下：

```assembly
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)

	ld ra, 0(a1)
	ld sp, 8(a1)
	ld s0, 16(a1)
	ld s1, 24(a1)
	ld s2, 32(a1)
	ld s3, 40(a1)
	ld s4, 48(a1)
	ld s5, 56(a1)
	ld s6, 64(a1)
	ld s7, 72(a1)
	ld s8, 80(a1)
	ld s9, 88(a1)
	ld s10, 96(a1)
	ld s11, 104(a1)
	
	ret    /* return to ra */
```



# Using threads ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

> 分析并解决一个在对哈希表进行操作的过程中，由于 race-condition 导致的数据丢失的问题。

这个实验也比较简单，在找到出现 missing 的原因后，加锁就能解决。出现 missing 的原因如下：

```
thread 1: 尝试设置 k1
thread 1: 发现 k1 不存在，尝试在 bucket 末尾插入 k1
--- scheduler 切换到 thread 2
thread 2: 尝试设置 k2
thread 2: 发现 k2 不存在，尝试在 bucket 末尾插入 k2
thread 2: 分配 entry，在桶末尾插入 k2
--- scheduler 切换回 thread 1
thread 1: 分配 entry，没有意识到 k2 的存在，在其认为的 “桶末尾”（实际为 k2 所处位置）插入 k1

[k1 被插入，但是由于被 k1 覆盖，k2 从桶中消失了，引发了键值丢失]
```

因此，只需要在插入 key 的时候，加桶级锁确保操作原子性即可。修改 `ph.c` 代码如下：

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <assert.h>
#include <pthread.h>
#include <sys/time.h>

#define NBUCKET 5
#define NKEYS 100000

struct entry {
  int key;
  int value;
  struct entry *next;
};
struct entry *table[NBUCKET];
int keys[NKEYS];
int nthread = 1;
pthread_mutex_t lock[NBUCKET];  // 为每一个 bucket 声明一个锁

double
now()
{
 struct timeval tv;
 gettimeofday(&tv, 0);
 return tv.tv_sec + tv.tv_usec / 1000000.0;
}

static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}

static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&lock[i]);  // 加锁

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock[i]);  // 释放锁
}

int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;


  if (argc < 2) {
    fprintf(stderr, "Usage: %s nthreads\n", argv[0]);
    exit(-1);
  }
  nthread = atoi(argv[1]);
  tha = malloc(sizeof(pthread_t) * nthread);
  srandom(0);
  assert(NKEYS % nthread == 0);
  for (int i = 0; i < NKEYS; i++) {
    keys[i] = random();
  }

  for(int i = 0; i < NBUCKET; i++){
    pthread_mutex_init(&lock[i], NULL);  // init pthread_mutex_lock
  }
}
```



# Barrier([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

修改 `barrier.c` 中的 `barrier` 函数即可，修改代码如下：

```c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&(bstate.barrier_mutex));
  bstate.nthread ++;
  if(bstate.nthread == nthread){
    bstate.nthread = 0;
    bstate.round ++;
    pthread_cond_broadcast(&(bstate.barrier_cond));
  } else{
    pthread_cond_wait(&(bstate.barrier_cond), &(bstate.barrier_mutex));
  }
  pthread_mutex_unlock(&(bstate.barrier_mutex));
}
```

