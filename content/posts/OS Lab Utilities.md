---
title: "OS Lab Utilities" 
date: 2024-07-06T14:53:38+08:00
draft: false
tags:
  - Mit 6.S081
  - OS
ShowToc: true
TocOpen: false 
---

# sleep([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

Lab1 的第一个实验是实现 **sleep 系统调用**，这个比较简单，理解了上面的内容就行了。完整过评测代码如下：

```python
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
        int n;
        if(argc != 2){
                fprintf(2, "Please enter a number\n");
                exit(1);
        }else{
                n = atoi(argv[1]);
                sleep(n);
                exit(0);
        }
}
```

对导入的包的说明： `user.h` 为 `XV6` 提供的系统函数，`types.h` 为其提供的变量类型。

第 11 行的 `atoi()` 函数，该函数用于将字符串类型转化为整型。在 `XV6` 系统中对于该函数的实现如下：

```c
int
atoi(const char *s)
{
  int n;
  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
```



# pingping([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

该实验主要考察对 `fork` 系统调用和 `pipe` 的理解。该实验的流程如下：

- 父进程：父进程向子进程传递信息，等待子进程结束，读取子进程传递的消息
- 子进程：读取父进程传递的消息，传递给父进程消息

对于 `fork` 系统调用 需要知道，`fork` 会拷贝当前进程的内存（这里并不会真正去复制内容，而是采用了一种 `COW` 的技术，这里只需要知道 `fork` 系统调用不会去真正复制父进程的内存），并创建一个新的进程，这里的内存包括指令和数据两个方面。在父进程中，会返回一个大于 0 的整数，即新创建的进程的 PID。而在子进程中，会返回 0。因此，`fork` 返回的值可以用来区分当前进程是父进程还是子进程，这是关键。

对于 `pipe` ，即管道，又叫做无名管道，常用于进程之间的通信，但是效率较低（涉及到内核态和用户态的相互转换，了解即可）。以下几点特别重要：

1. 半双工，数据在同一时刻只能在一个方向流动；
2. 管道传送的数据不是无格式，必须事先约定好；
3. 管道不是普通的文件，不属于某个文件系统，只存在于内存中；
4. 管道只能在具有公共祖先的进程之间使用。

创建一个管道的示例如下：

```c
#include<unistd.h>
int pipe(int fds[2]);
```

`pipe` 函数定义中的 `fds` 参数是一个大小为 2 的数组类型指针。通过 `pipe` 函数创建的这两个文件描述符 `fds[0]` 和 `fds[1]` 分别构成管道的两端，往 `fds[1]` 写入的数据可以从 `fds[0]` 读出，并且 `fds[1]` 一端只能进行写操作，`fds[0]` 一端只能进行读操作，不能反过来使用。

对于 `read` 系统调用，一共接受 3 个参数：

1. 第一个参数是文件描述符，指向一个之前打开的文件。shell 会确保默认情况下，当一个程序启动时，文件描述符 0 连接到 console 的输入，文件描述符 1 连接到了 console 的输出。这里的 0，1 文件描述符是非常普遍的 Unix 风格，许多的 Unix 系统都会从文件描述符 0 读取数据，然后向文件描述符 1 写入数据；
2. 第二个参数是指向某段内存的指针，程序可以通过指针对应的地址读取内存中的数据。`read` 系统调用将把读到的数据写入该指针指向的内存空间；
3. 第三个参数是代码想读取的最大长度。

返回值可能是读到的字节数，但是如果到达了文件的结尾没有更多的内容了，`read` 会返回0。**通常来说，系统调用通常是通过返回 -1 来表示错误，了解这点很关键**。

对于 `write`系统调用，也接受 3 个参数，每个参数对应的逻辑和 `read系统调用` 区别不大。

关于 `read` 系统调用 和 `write` 系统调用 还有一个值得注意的点就是：**它们并不关心读写的数据格式，它们就是单纯的读写，而 copy 程序会按照 8bit 的字节流处理数据，你怎么解析它们，完全是用应用程序决定的。**

了解了上述这些之后，该实验也就很好解决了。完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char* argv[]){
        int fds[2];
        char buf[2];
        char *msg = "o";
        pipe(fds);

        int pid = fork();
        if(pid == 0){
                if(read(fds[0], buf, 1) != 1){
                        fprintf(2, "Can't read from parent\n");
                        exit(1);
                }
                close(fds[0]);
                printf("%d: received ping\n", getpid());

                if(write(fds[1], msg, 1) != 1){
                        fprintf(2, "Can't write to parent\n");
                        exit(1);
                }
                close(fds[1]);
                exit(0);

        }else{
                if(write(fds[1], msg, 1) != 1){
                        fprintf(2, "Can't write to child\n");
                        exit(1);
                }
                close(fds[1]);
                wait(0);

                if(read(fds[0], buf, 1) != 1){
                        fprintf(2, "Can't read from child\n");
                        exit(1);
                }
                close(fds[0]);
                printf("%d: received pong\n", getpid());
                exit(0);
        }

}
```



# find([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

该实验主要仿照 **ls.c** 的实现思路完成，额外的注意点如下：

1. 当前路径为文件，直接检查是否是要查找的文件名
2. 当前路径为文件夹，对该目录下的所有文件递归，**\. 和 .. 除外**

这里还需要用的 `open` 系统调用，一共接受2个参数：

1. 第一个参数是想要打开的文件路径
2. 第二个参数是一些标志位，用来告诉 `open` 系统调用在内核中的实现

`open` 系统调用会返回一个新分配的文件描述符（关于文件描述符，可以参考我的另一篇文章）。

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void
compare(char *path, char *name)
{
    int fp = 0;
    int cp = 0;
    while(path[fp] != 0){
        cp = 0;
        int tp = fp;
        while(name[cp] != 0){
            if(path[tp] != name[cp]) break;
            cp++;
            tp++;
        }
        if(name[cp] == 0){
            printf("%s\n", path);
            return;
        }
        fp++;
    }
}

void
find(char *path, char *name)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_FILE:
            compare(path, name);
            break;
        
        case T_DIR:
            // Checks if the total length of the path and the directory entry name exceeds the size of the buffer buf
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0) continue;
                if(de.name[0] == '.' && de.name[1] == 0) continue;
                if(de.name[0] == '.' && de.name[1] == '.' && de.name[2] == 0) continue;
                memmove(p, de.name, DIRSIZ);
                // Set EOF
                p[DIRSIZ] = 0;
                if(stat(buf, &st) < 0){
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }
                find(buf, name);
            }
            break;
    }
    close(fd);
}

int
main(int argc, char *argv[])
{
    if(argc<3){
        fprintf(2, "Usage: find [path] [filename]\n");
        exit(-1);
    }
    find(argv[1], argv[2]);
    exit(0);
}
```



# primes([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

管道筛素数的过程类似于一个递归调用的过程：

1. `primes()` 函数首先会创建一个自己的管道，接着从管道中读出一个数 A，这里有一个注意点就是该数一定是素数（可参考埃氏筛法）。然后继续读取管道中剩下的数 B，判断 B 是否是 A 的倍数，如果不是就写入管道；
2. `primes()` 函数还会创建一个子进程用于递归处理自己的管道，从管道中拿出第一个数（一定是素数），然后接着读取管道并作比较。

特别需要注意的点就是：一定要关闭不需要的文件描述符，因为文件描述符是有限的。

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void
prime(int rd)
{
    int num, cnum;
    if(read(rd, &num, 4) != 4){
        fprintf(2, "Cann't read from pipe\n");
        exit(1);
    }
    printf("prime %d\n", num);

    int cfds[2];
    pipe(cfds);
    if(fork() == 0){
        close(cfds[1]);
        prime(cfds[0]);
        close(cfds[0]);
    }else{
        close(cfds[0]);
        while(read(rd, &cnum, 4)){
            if(cnum % num != 0){
                if(write(cfds[1], &cnum, 4) != 4){
                    fprintf(2, "Cann't write to pipe\n");
                    exit(1);
                }
            }
        }
        close(rd);
        close(cfds[1]);
        wait(0);
    }
    exit(0);
}

int main(int argc, char *argv[])
{
    if(argc != 1){
        fprintf(2, "Usage: primes\n");
        exit(1);
    }

    int fds[2];
    pipe(fds);

    int pid = fork();
    if(pid != 0){
        close(fds[0]);
        for(int i = 2; i <= 35; i++){
            if(write(fds[1], &i, 4) != 4){
                fprintf(2, "Cann't write to pipe\n");
                exit(1);
            }
        }
        close(fds[1]);
        wait(0);
    }else{
        close(fds[1]);
        prime(fds[0]);
        close(fds[0]);
    }
    exit(0);
}
```



# xargs([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

首先介绍一下什么是 `xargs`。简单来说，`xargs` 命令是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具。它擅长将标准输入数据转换成命令行参数，`xargs` 能够处理管道或者 `stdin` 并将其转换成特定命令的命令参数。`xargs` 也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。`xargs` 的默认命令是 `echo`，空格是默认定界符。这意味着通过管道传递给 `xargs` 的输入将会包含换行和空白，不过通过 `xargs` 的处理，换行和空白将被空格取代。`xargs` 是构建单行命令的重要组件之一。

了解了这些之后，就清楚了任务是什么：从 `stdin` 中获取到参数，拼接到 `xargs` 命令后面。如果 `stdin` 中的输出是多行，则需要拼接多行命令执行。

程序代码具体流程如下：

1. 将 `xargs` 传入的命令单独保存，将 ` xargs` 传入的参数单独保存到一个数组中；
2. 将 `stdin` 中获取到的数据根据换行符 "\n"，拼接到 `xargs` 传入参数的数组后面；
3. 对每行参数依据空格进行划分；
4. 调用一个 `fork` 系统调用和 `exec` 系统调用 执行命令。

**这里有个常用的写法要注意，先调用 `fork`，再在子进程中调用 `exec`。尽管有些浪费，但是优化过程后面再提。**

完整的过评测代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"
#define MAXLEN 100

int main(int argc, char *argv[])
{
    if(argc < 2){
        fprintf(2, "Usage: xargs [command]\n");
        exit(1);
    }
    // Get the command
    char *cmd = argv[1];
    
    char nargv[MAXARG][MAXLEN];
    char *pnargv[MAXARG];
    char buf;
    
    // Loop lines
    while(1){
        memset(nargv, 0, MAXARG*MAXLEN);
        for(int i = 1; i < argc; i++) strcpy(nargv[i-1], argv[i]);
        int cargc = argc - 1;
        int offset = 0;
        int is_read = 0;
        // Get all params of one line
        while((is_read = read(0, &buf, 1)) > 0){
            if(buf == ' '){
                cargc++;
                offset = 0;
                continue;
            }
            if(buf == '\n'){
                break;
            }
            if(offset == MAXLEN){
                fprintf(2, "xargs: parameter too long\n");
                exit(1);
            }
            if(cargc == MAXARG){
                fprintf(2, "xargs: too many arguments\n");
                exit(1);
            }
            nargv[cargc][offset++] = buf;
        }
        if(is_read <= 0) break;
        for(int i = 0; i <= argc; i++) pnargv[i] = nargv[i];
        if(fork() == 0){
            exec(cmd, pnargv);
            exit(1);
        }
        wait(0);
    }
    exit(0);
}
```

