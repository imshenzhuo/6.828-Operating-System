# Homework: xv6 system calls

更改`xv6`,添加一个系统调用

## Part One: System call tracing

当系统调用时候打印系统调用的名字和返回值

这是一道考察`C语言`的题目, 根据提示修改文件`syscall.c`的`syscall`函数,如下

``` C
void
syscall(void)
{
  int num;
  struct proc *curproc = myproc();

  num = curproc->tf->eax;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
}
```

`syscalls`比较特殊, 是一个函数指针数组, `syscalls[num]()`是对第`num`个函数的调用, 很容易看得出来, `num`就是调用的编号, 返回的值就存在`curproc->tf->eax`中. 剩下的就是`print`就可以了.



## Part Two: Date System Call

首先添加`data.c`

``` C
#include "types.h"
#include "user.h"
#include "date.h"

int
main(int argc, char *argv[])
{
  struct rtcdate r;

  if (date(&r)) {
    printf(2, "date failed\n");
    exit();
  }

  // your code to print the time in any format you like...

  exit();
}
```

在`Makefile`中的`UPROGS`添加`_date`

照猫画虎, `grep -n uptime *.[chS]`

``` bash
syscall.c:107:extern int sys_uptime(void);
syscall.c:124:[SYS_uptime]  sys_uptime,
syscall.h:15:#define SYS_uptime 14
sysproc.c:83:sys_uptime(void)
user.h:25:int uptime(void);
usys.S:31:SYSCALL(uptime)
```

但是不懂啊......