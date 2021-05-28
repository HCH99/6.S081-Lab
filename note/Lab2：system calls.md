### 实验目的

In the last lab you used systems calls to write a few utilities. In this lab you will add some new system calls to xv6, which will help you understand how they work and will expose you to some of the internals of the xv6 kernel. You will add more system calls in later labs.

了解系统调用的工作方式，以及初步接触xv6内核。

系统调用：是操作系统为进程提供的服务接口。

### System call tracing

简要：创建一个名为`trace`的系统调用，这个系统调用还接收一个int类型的mask，其中得每个bit指定了需要追踪的系统调用（通过系统调用号）。使用这个命令时，打印每次追踪的系统调用的名字以及它们的返回值。这个效果应该在子进程也生效。

以下是自带的trace.c，所以任务是要我们完成这个系统调用的底层实现（即内核部分）。

~~~c
int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
~~~

具体步骤（hints基本已经告诉你怎么做了）：

1.加入makefile

2.user/user.h加入trace原型

3.user/usys.pl加入trace存根，用于生成usys.S（真正的系统调用存根，使用ecall指令从用户态跳入内核态）

4.定义trace的系统调用号

从这里开始，编译qemu已经没有错误。

5.在kernel/sysproc.c里写sys_trace()函数，作用已经告诉你了：将参数(int类型的mask)记录到kernel/proc.h中的proc结构体上（每个进程都有的）。

~~~c
uint64
sys_trace(void)
{
  int n;
  if(argint(0, &n) < 0)
    return -1;
  myproc()->mask = n;
  return 0;
}
~~~

argint()是用来获取寄存器中的值的，上述取a0寄存器的值传入n。

这里在需要在proc结构体上加入新的字段，这里我命名为mask：

~~~c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  //一堆加锁才能获得的变量

  // these are private to the process, so p->lock need not be held.
  //一堆不加锁的变量

  //我们加入的变量
  int mask;
};

~~~

（这里使用mask是否需要获取进程锁呢？）

6.修改kernel/pro.c里的fork()使得mask可以传到子进程里。

~~~c
int
fork(void)
{
  //一堆代码
  //我们加入到从copy父进程的用户内存到子进程这一代码段区间，比较安全
  np->sz = p->sz  
    
  np->mask = p->mask;

  np->parent = p;  
    
  //一堆代码
}

~~~

7.修改kernel/syscall.c的syscall()函数来打印trace的输出，这个syscall函数是内核通过系统调用号来执行系统调用函数的地方。除了声明系统调用函数以及将系统调用号加入到数组外，它还热心提醒我们需要系统调用名组成的数组用来print。

~~~c
//其他声明的函数
extern uint64 sys_trace(void);

static uint64 (*syscalls[])(void) = {
    //其他系统调用号
    
   	//加入trace
	[SYS_trace]   sys_trace,

}

void
    syscall(void)
{
    int num;
    struct proc *p = myproc();

    num = p->trapframe->a7;
    if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
        //加入的代码，判断该系统调用是否被trace
        if ( p->mask & 1<<num ) {
            printf("%d: syscall %s -> %d\n", p->pid, syscalls_name[num-1], p->trapframe->a0);
        }
        //

    } else {
        printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}

//这里也可以用GNU扩展语法来编写。
char* syscalls_name[] = {"fork","exit", "wait", "pipe", "read", "kill","exec","fstat","chdir","dup","getpid","sbrk","sleep",
                         "uptime","open","write","mknod","unlink","link","mkdir",
                         "close","trace", "sysinfo"};

~~~

### Sysinfo

简要：编写一个系统调用sysinfo，能够收集系统运行信息。有一个参数是指向kernel/sysinfo.h里定义的`sysinfo`结构体的指针。内核需要为结构体的变量赋值，其中的`freemem`域为空闲内存的数量，`nproc`是`state`为`UNUSED`的进程数量。

两个点：

1. 统计空闲内存数

   观察kernel/kalloc.c中的kalloc和kfree函数，易发现空闲页使用一条空闲链表来分配，且新的空闲页直接作为这条链表的首部。这样，我们只需要统计链表结点数即可计算空闲内存的大小（count * PGSIZE）。在该文件里我们添加计算的函数：

   ~~~c
   uint64 free_mem(void)
   {
       struct run *r;
   
       //注意加锁！
       acquire(&kmem.lock);
       r = kmem.freelist;
       uint64 count = 0;
       while (r) {
           count++;
           r = r->next;
       }
       release(&kmem.lock);
       return count * PGSIZE;
   }
   ~~~

2. state为UNUSED的进程数，这就更简单了，我们只需要遍历进程的结构体数组即可，根据hint找到kernel/proc.c，正好allocproc函数是找UNUSED的进程，我们照猫画虎即可：

~~~c
uint64 n_proc(void)
{
    struct proc *p;

    uint64 count = 0;
    for (p = proc; p < &proc[NPROC]; p++) {
        //记得加锁！
        acquire(&p->lock);
        if (p->state != UNUSED) {
            count++;
        }
        release(&p->lock);
    }
    return count;
}
~~~

以上工作做完后，别忘了在kernel/defs.h添加函数原型。

最后来编写我们的sys_sysinfo函数（在此之前根据hint搞清如何使用copyout()）:

~~~c
uint64
    sys_sysinfo(void)
{
    printf("in sysinfo\n");
    struct sysinfo info;
    info.freemem = free_mem();
    info.nproc = n_proc();

    struct proc *p = myproc();
    uint64 addr;
    //取得用户指向sysinfo结构体的指针
    if (argaddr(0, &addr) < 0)
        return -1;
    //将内核中的数据复制到给定页表的用户进程
    if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
        return -1;
    return 0;
}
~~~

为了能够成功编译qemu，除了需要在user/user.h上预定义sysinfo结构体，其他做的事情与前面的trace系统调用是一样的。

### 总结

通过此次实验，我们应该了解系统调用的底层是如何实现的，以及如何实现一个系统调用。

