# GDB 简单使用入门

### 什么是GDB

`GDB` =>`GNU debugger`  ，是在终端调试代码的一个工具.

### GDB常用指令

1. `Ctrl-c` 程序暂停执行
2. `c` (or continue)  继续执行，除非遇到断点或者 `Ctrl-c`.
3. `si`执行一条指令
4. `b`  在函数或者代码指定行数打断点
5. `b *addr` (or breakpoint) 在`eip`为`addr`时设置断点
6. `set print pretty` 使数组、结构体好好打印
7. `info registers ` 打印寄存器的值

- `x/Nx addr` 从虚拟地址`addr`开始以16进制打印N个字
- `x/Ni addr` 打印从`addr`开始的N个汇编指令

### 其他

layout使用

- `layout regs`

https://blog.csdn.net/zhangjs0322/article/details/10152279



## 参考

https://pdos.csail.mit.edu/6.828/2018/labguide.html#gdb

http://sourceware.org/gdb/current/onlinedocs/gdb/