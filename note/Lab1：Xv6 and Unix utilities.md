### 实验目的

This lab will familiarize you with xv6 and its system calls.

主了解如何使用系统调用编写一些实用函数。

### sleep

简要：实现xv6的sleep程序。

没什么难度。按照hints做就是了

~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) 
{
	sleep(atoi(argv[1]));
	exit(0);
}

~~~

### pingpong

简要：实现进程间各写和读取一个字节。

这是第一个接触pipe和fork的实验，建议在做之前先看看xv6 book关于这两个函数的章节（第一章），为今后实验打下基础。

fork()：复制一个与当前进程完全一样的新进程（除了fork()返回值），旧进程的fork()返回值为新进程的pid，而新进程返回0，这就实现了新进程执行不同的程序逻辑。但由于原封不动的复制太耗时间和空间，底层实际使用copy-on-write（写时复制）的技术。

简单介绍一下pipe函数：

![img](https://pic002.cnblogs.com/images/2012/426620/2012110216160766.jpg)

1. pipe函数会在内核中开辟一个缓冲区，用于实现**进程间通信**，然后将管道的两端的文件描述符传入参数(int数组，第一个为读端，第二个为写端)。
2. 默认情况下pipe是阻塞的，从而导致对文件描述符的read和write都是阻塞的。
3. 这里我们可以知道，文件描述符是**int类型**（非负整数）。因为在Unix一切皆文件，所以**这里的文件不是我们传统意义上了解的文件**，管道也是通过文件描述符进行读写的。

关于pipe易错的点：

1. 由于默认的管道是阻塞的，所以read()是阻塞的，因此先print出信息后再write，这样父进程就不会和子进程同时print了。
2. pipe的fd随着fork也会跟着fork，如果在父进程里创建的一个pipe之后再fork一个子进程，那么这个pipe相当于有**4个end**

这里的pipe还不是很复杂，按照hints做就行，主要思想是创建两个pipe分别往两个方向读写。

~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, int argv[])
{
    int p1[2];
    int p2[2];
    pipe(p1);
    pipe(p2);
    write(p1[1],"1",1);
    close(p1[1]);
    if (fork() == 0) {
        close(p2[0]);
        int pid = getpid();
        char* buf = "";
        read(p1[0], buf, 1);
        printf("%d: received ping\n", pid);
        write(p2[1], buf, 1);
        close(p1[0]);
        close(p2[1]);
    } else {
        close(p1[0]);
        close(p2[1]);
        char* buf = "";
        read(p2[0], buf, 1);
        close(p2[0]);
        int pid = getpid();
        printf("%d: received pong\n", pid);
    }
    exit(0);
}

~~~

### primes

简要：使用pipes完成一个并发版本的素数筛选器，根据材料得到的伪代码：

~~~c
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
~~~

完成自己的函数。

思路感觉跟递归差不多，只不过进入新函数换成了创建子进程。

注意点：

- Be careful to close file descriptors that a process doesn't need, because otherwise your program will run xv6 out of resources before the first process reaches 35.

  **没用的端一定要关闭**！一定要关闭！只要有一个end没关，read就会一直阻塞！且end占用文件描述符的，而XV6的文件描述符和进程数量是有限的。

* 仔细读hints
* **dup()函数可以复制文件描述符，并返回一个新的文件描述符（复制到序号最小的文件描述符）**

~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, int argv[])
{
    int fd[2];
    pipe(fd);
    int num = 2;
    for (int* i= &num; (*i) <= 35; (*i)++) {
        write(fd[1],i,4);
    }
    int baga = 0;
    int *p = &baga;
    while (1) {
        close(fd[1]);
        if (read(fd[0], p, 4) != 0) {
            printf("prime %d\n", *p);
            if (fork() == 0) {
                int fd2[2];
                pipe(fd2);
                int bagayalu = 0;
                int* n = &bagayalu;
                //printf("1 %d\n", getpid());
                while(read(fd[0], n, 4) != 0) {
                    if ((*n)%(*p) != 0) {
                        write(fd2[1], n, 4);
                    }
                }
                close(fd[0]);
                fd[0] = dup(fd2[0]);
                fd[1] = dup(fd2[1]);
                close(fd2[0]);
                close(fd2[1]);
            } else {
                close(fd[0]);
                break;
            }
            //printf("2 %d\n", getpid());
        } else {
            exit(0);//for last process
        }
    }
    wait(0);
    exit(0);
}
~~~

### find

简要：匹配给定目录的文件

![img](https://pic3.zhimg.com/80/v2-9c04a9b4dbb1d6351efb95f12a937b46_720w.jpg)

这里出现的几个新面孔：

* fstat()：根据指定文件描述符获取文件信息传入stat结构体
* stat()：根据文件名（在程序中放在了buf数组）获取文件信息传入stat结构体
* stat结构体用于存储，其中包括文件的inode、文件类型等信息
* dirent结构体（即directory entry，联想到map是由entry组成的）包括文件名和inode等信息，一个目录文件由一组dirent结构体构成

思路就是，改改ls的代码就完事了

~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"


//format name
char*
    fmtname(char *path)
{
    char *p;

    // Find first character after last slash.
    for(p=path+strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;

    return p;
}

void
    find(char *path, char *name)
{
    //buf for a sequence of directories
    char buf[512], *p;
    int fd;
    // dir file contains a sequence of dirent structures.
    struct dirent de;
    // extract file stats from inode
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
            if (strcmp(fmtname(path),name)==0) {
                printf("%s\n", path);
            }
            break;

        case T_DIR:

            //path args too long for adding another dir name
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("ls: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';

            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                // inode 0 is a special inode
                if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0 || de.inum == 0)
                    continue;

                //de.name is a sub-directory name
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;//put a end to str
                if(stat(buf, &st) < 0){
                    printf("ls: cannot stat %s\n", buf);
                    continue;
                }
                //printf("%s\n", fmtname(buf));
                if (strcmp(fmtname(buf), name) == 0) {
                    printf("%s\n", buf);
                }
                if (st.type == T_DIR) {
                    //printf("%s\n", buf);
                    find(buf,name);
                }
            }
            break;
    }
    close(fd);
}

int
    main(int argc, char *argv[])
{
    find(argv[1], argv[2]);
    exit(0);
}
~~~

### xargs

简要：写一个简单版本的UNIX xargs程序：从标准输入读入行，对每行都执行一个相同的命令，其中每一行都作为参数加入到命令参数组的末尾。

记得我当时因为没理解题意还有管道|，调试好久。。后来才明白，原来|的功能已经执行了，而我要做的只需要从标准输入中读取即可。总体来说明白这点，就不是很难了。

此题出现了fork()和exec()组合，这也是在一个命令程序中另外一个程序（命令）的方式。

简单介绍一下exec()函数：执行一个可执行文件，并且新进程替换当前进程的除了pid和文件描述符表外（所以能实现标准输出）的所有内容。参数为文件名和命令行参数数组（指针）。只有在找不到相应文件时才会返回-1；执行成功时不会返回，因为已经没有地方返回了。

比如：

~~~bash
$ echo "1\n2" | xargs -n 1 echo line
line 1
line 2
$
~~~

容易踩坑的点：执行exec时，没有意识到参数xargs需要去掉。

~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/param.h"

int main(int argc, char *argv[])
{
    char arg[MAXARG+1];
    int t = 0;
    while (read(0, &arg[t], 1) != 0) {
        if ( arg[t] != '\n') {
            t++;
            continue;
        } else {
            arg[t] = 0;
            //printf("%s", arg);
            char *new_argv[argc+1];
            int i;
            for (i = 1; i < argc; i++) {
                //printf("%s ", argv[i]);
                new_argv[i-1] = argv[i];
            }
            //printf("%s %s\n", new_argv[i],arg);
            new_argv[i-1] = arg;
            //new_argv[i] = 0;
            if (fork() == 0) {
                exec(new_argv[0], new_argv);
            } else {
                t = 0;
                wait(0);
            }
        }
    }
    //printf("%d exits\n", getpid());
    exit(0);
}
~~~

### 总结

通过以上实验，我了解了xv6的命令行程序编写方式，一些常见的系统调用函数如pipe、fork、exec等，还有进程间如何通信（pipe）以及文件描述符的使用，浅入文件系统的一些结构。

