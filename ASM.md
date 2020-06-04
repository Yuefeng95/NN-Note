# 内联汇编（GCC）

使用内联汇编，实现循环中的代码加速

> 相关学习资源    
> http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0203ic/Babjhjbd.html
> https://www.ibm.com/developerworks/cn/aix/library/au-inline_assembly/index.html


## 基本格式

```
asm ( assembler template 
    : output operands                  /* optional */
    : input operands                   /* optional */
    : list of clobbered registers      /* optional */
    );

// e.g.
asm ("leal (%0,%0,4), %0"
     : "=r" (five_times_x)
     : "0" (x) 
     );
```

## 约束条件

### 通用约束

1. 寄存器操作数约束（r）   
指定这个变量使用的寄存器   
使“r”，指定任意可用的寄存器，使用其他标志，指定使用特定寄存器

```
asm ("movl %%eax, %0\n" :"=r"(myval));
```

| r |    Register(s)     |
| - | ------------------ |
| a |   %eax, %ax, %al   |
| b |   %ebx, %bx, %bl   |
| c |   %ecx, %cx, %cl   |
| d |   %edx, %dx, %dl   |
| S |   %esi, %si        |
| D |   %edi, %di        |

2. 内存操作数约束（m）   
直接修改内存

```
asm("sidt %0\n" : :"m"(loc));
```

3. 匹配（数字）约束

使输入输出作用于同一个寄存器

```
asm ("incl %0" :"=a"(var):"0"(var));
```

### 其他约束

1. “ m”：允许使用内存操作数，并且通常使用机器支持的任何类型的地址。
2. “ o”：允许使用内存操作数，但前提是该地址是可偏移的。也就是说，在地址上加上一个小的偏移量就可以得到一个有效的地址。
3. “ V”：不可偏移的内存操作数。换句话说，任何符合“ m”约束但不符合“ o”约束的事物。
4. “ i”：允许使用立即数整数（具有恒定值的一个）。这包括符号常量，其值仅在组装时才知道。
5. “ n”：允许使用具有已知数值的立即数整数。许多系统不能支持小于一字宽的操作数的汇编时常数。这些操作数的约束应使用“ n”而不是“ i”。
6. “ g”：允许使用任何寄存器，内存或立即数整数，但不是通用寄存器的寄存器除外。

### 约束修饰符

一些可选的修饰符   

1. “=”：该指令 write only
2. “&”：大概是引用的意思，作为输出
3. “+”：可读写的

### 标记被修改的寄存器

通过标记内联汇编中可能修改的寄存器，使编译器对上下文的寄存器使用进行调整和暂存。避免寄存器中的值被覆盖。

1. "cc"     
指令会影响条件寄存器
2. "memory"    
指令会访问未知的内存
3. "r0" "r1" ...   
指令会修改通用寄存器

#### ARMv8-A寄存器

1. x    
aarch64下。31个64位通用寄存器，始终可以访问。
2. v    
aarch64下。v寄存器
3. r    
32位的寄存器
4. w    
neon寄存器

> neon寄存器中内联汇编的使用帮助   
> https://github.com/Tencent/ncnn/blob/master/docs/developer-guide/armv7-mix-assembly-and-intrinsic.md    