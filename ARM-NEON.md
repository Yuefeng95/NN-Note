## NEON优化

NEON是用来实现ARM芯片上计算加速的一套系统。以下介绍NEON的基本知识和使用实例。

> 喜欢啃英文手册的同学可以直接跳转 [NEON在线手册](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dht0002a/ch01s03s02.html) 

（文中提及的都是armv8 32位指令）

## 什么是SIMD

NEON是一种加速技术的实现，而这个技术就是SIMD。
SIMD可通过并行控制，实现对寄存器的并行操作，比如对32bit寄存器中的4个8bit数据进行4条指令并行的运算。现代高级编程语言不能做到如此精细的控制。

![SIMD-Instructions](./pictures/SIMD-Instructions.png)

上图描述了对32bit寄存器中4个8bit数并行操作的过程。R2和R1分别存储4x8bit数据，通过并行执行4个Lane（运算通道）在一个计算周期中完成矩阵乘法，结果存在R0中。

> R0[0] = R1[0] x R2[0]

> R0[1] = R1[1] x R2[1]

> R0[2] = R1[2] x R2[2]

> R0[3] = R1[3] x R2[3]

## 什么是NEON

对SIMD技术的实现，就是NEON。通过C函数的方式，对并行操作进行封装。

## 为什么需要在NEON中实现并行运算

SIMD的寄存器内的并行优于现代编程语言多线程并行操作的原因在于数据加速速度与CPU运算速度的差异。
现代CPU的运算速度远超过内存加载的速度，Core i7 CPU读取离他最近（不论是物理还是逻辑上，都是最近）的L1缓存仍需要约4个CPU周期，而读取L2缓存或更远的内存的时间成本更加昂贵。

### CPU读取内存的过程

> 喜欢啃英文博客的同学可直接跳转 [How L1 and L2 CPU Caches Work, and Why They’re an Essential Part of Modern Chips](https://www.extremetech.com/extreme/188776-how-l1-and-l2-cpu-caches-work-and-why-theyre-an-essential-part-of-modern-chips)

CPU读取L1缓存时，就像人们挑选自己的扑克牌，必须每个牌都看一遍才能正确拾取。

## NEON的硬件基础

NEON实现的SIMD技术，依赖一套NEON寄存器，这套寄存器与L1 cache L2 cache相同，量小质优，读写速度极快。下面列举ARMv7芯片中的cache信息。

| 名称 | 编号 | 大小 |
| --- | --- | --- |
| L1 Cache | R0 - R15 | 16 x 32bit |
| NEON寄存器 | Q0 - Q15 或 D0 - D31 | 16 x 128bit 或 32 x 64bit |
| VFP寄存器（浮点运算）| S0 - S15 | 16 x 32bit |

表中NEON寄存器是NEON函数调用的主要寄存器。ARMv7有16个128bit寄存器，每个128bit寄存器可拆分成2 x 64bit使用。

![ARMv7-Registers.png](./pictures/ARMv7-Registers.png)

## NEON的数据结构

NEON包含的数据类型作为运算时的计算单位，包含：
* 8bit、16bit、32bit、64bit的有符号无符号整数
* 32bit单精度浮点数
* 8bit、16bit的多项式，用于无进位的乘法

在NEON的函数中，这些数据类型不会直接作为运算的单位，会组合成固定长度的数组。这些数组是NEON运算的基本单位。

```c
// e.g.
int8x8_t
uint8x8_t
float32x4_t
```

数组会组合成结构体，举个例子：

```c
// e.g.
typedef struct int8x8x2_t  
{  
   int8x8_t val[2];  
} int8x8x2_t;
```

## NEON的函数形式

NEON的函数凸显其并行运算的本质，每个函数都将数组中的数据并行执行相同的运算，将结果存储在数组中。以两个8x8bit的数组做 [点积](https://zh.wikipedia.org/wiki/%E7%82%B9%E7%A7%AF) 为例。

```c
// ri = ai * bi
int16x8_t vmull_s8 (int8x8_t __a, int8x8_t __b);  
```

这里函数名的开头“v”是固定开头，“mul”代表乘法（multiplication），“l”代表此运算针对32位的寄存器进行操作（8个8bit的整数），“s8”代表有符号的8位整数。
> 头文件概要 [arm_neon.h](./include/arm_neon.h)

## NEON优化实例

这里以 向量 × 矩阵 为场景。先敲一段C实现的代码做对照组。

```cpp
template<typename T = int8_t, typename M = int32_t>
inline void do_plain_multipy_tensor_with_arr(const T *tensor, const T *arr, M *output, int tensor_size) {
    for (int h = 0; h < tensor_size; ++h) {
        for (int w = 0; w < tensor_size; ++w) {
            output[h] += arr[w] * tensor[h * tensor_size + w];
        }
    }
}
```

采用最简单的思路完成 矩阵 × 向量 的运算。  
我们为这段代码写一些测试程序，限定输入的数据格式均为 int8_t，限定输出的数据格式为int32_t（防止运算结果溢出），限定输入的矩阵形状为320x320，同时数组的长度也为320。
将这段代码及测试程序运行在ARMv8芯片安卓系统上，编译选项中开启 -O3。运行一次的时间为 1.20663 ms。

下面采用NEON函数对这段代码进行优化。  
因为NEON全部采用并行运算，所以基本思路是把同样的运算放在一组数中进行。代码如下，每次选取8个数组中的数和8个向量中的数进行乘法，最后把结果加在一起。
```cpp
inline void do_plain_neon_mul_ten_with_arr(const int8_t *tensor, const int8_t *arr, int32_t *output, int tensor_size) {
    for (int h = 0; h < tensor_size; ++h) {
        output[h] = 0;

        int8x8_t arr_mem;
        int8x8_t ten_mem;
        int32x4_t res_mem = vdupq_n_s32(0);

        int w = 0;
        for (; w + 7 < tensor_size; w += 8) {
            // 加载8个数组中的数据
            arr_mem = vld1_s8(arr + w);
            // 加载8个向量中的数据
            ten_mem = vld1_s8(tensor + h * tensor_size + w);
            // 将两个数组中的数进行乘法，将得到的结果与之前的结果做加法并保存
            res_mem = vpadalq_s16(res_mem, vmull_s8(arr_mem, ten_mem));
        }
        // 当数组中剩下的数据不足以填充到8位时，采用C代码进行补充运算
        for (; w < tensor_size; ++w) {
            output[h] += tensor[h * tensor_size + w] * arr[w];
        }
        // 取出NEON寄存器中的点积运算结果，并加到一起，得到一行点积的最终结果
        output[h] += vgetq_lane_s32(res_mem, 0);
        output[h] += vgetq_lane_s32(res_mem, 1);
        output[h] += vgetq_lane_s32(res_mem, 2);
        output[h] += vgetq_lane_s32(res_mem, 3);
    }
}
```
与纯C代码的输入一致的情况下，耗时 0.237708 ms，只有原始纯C代码耗时的20%。  
然而这还存在较大的优化空间。  
通过调查每步的执行耗时，可分析出具体的优化点。   
首先我们将这段NEON + C的代码中执行的操作进行分类：  
* 内存读操作。例如 `vld1_s8`
* 内存写操作。例如 `vgetq_lane_s32`，`vpadalq_s16`中的写内存
* 计算操作。例如`vmull_s8` `vpadalq_s16` 中的计算部分。

测试以上操作的具体耗时，我们的测试方法如下：
```cpp
inline void do_plain_neon_mul_ten_with_arr(const int8_t *tensor, const int8_t *arr, int32_t *output, int tensor_size) {
    for (int h = 0; h < tensor_size; ++h) {
        output[h] = 0;

        // 在声明时进行初始化，保证数组的声明没有被优化在计算操作中
        int8x8_t arr_mem = vdup_n_s8(0);
        int8x8_t ten_mem;
        int32x4_t res_mem = vdupq_n_s32(0);

        int w = 0;
        for (; w + 7 < tensor_size; w += 8) {
            // 注释掉这段都内存操作
            // arr_mem = vld1_s8(arr + w);
            ten_mem = vld1_s8(tensor + h * tensor_size + w);
            res_mem = vpadalq_s16(res_mem, vmull_s8(arr_mem, ten_mem));
        }
        for (; w < tensor_size; ++w) {
            output[h] += tensor[h * tensor_size + w] * arr[w];
        }
        output[h] += vgetq_lane_s32(res_mem, 0);
        output[h] += vgetq_lane_s32(res_mem, 1);
        output[h] += vgetq_lane_s32(res_mem, 2);
        output[h] += vgetq_lane_s32(res_mem, 3);
    }
}
```

运行上一段代码，耗时为 0.199791 ms。可见将64位内存读取到NEON寄存器中，消耗时间的占比还是不小的。   
按照这个思路，我们对都内存的操作进行优化，具体思路是减少都内存的操作。在矩阵×数组的过程中，每次点积运算都会使用同一段数组与不同的矩阵行相乘，也就是说减少读相同数组的次数，就可以减少读内存操作。    
接下来敲这段优化代码。
```cpp
    for (int i = 0; i < arr_len; i += 8) {
        w_mem = vld1_s8(a);

        h_mem = vld1_s8(t0);
        mul_res_mem0 = vpadalq_s16(mul_res_mem0, vmull_s8(w_mem, h_mem));
        h_mem = vld1_s8(t1);
        mul_res_mem1 = vpadalq_s16(mul_res_mem1, vmull_s8(w_mem, h_mem));
        h_mem = vld1_s8(t2);
        mul_res_mem2 = vpadalq_s16(mul_res_mem2, vmull_s8(w_mem, h_mem));
        h_mem = vld1_s8(t3);
        mul_res_mem3 = vpadalq_s16(mul_res_mem3, vmull_s8(w_mem, h_mem));
    }
```

以上代码完成了对读内存的优化，消耗时间为 0.154 ms。   
这个耗时比单纯的压缩对每行矩阵运算读操作的耗时还要短，是因为在计算前对读内存的指针距离进行了优化，这个原理可以从文章开头对CPU-cache的介绍中了解到。    


## 总结

以上是对NEON优化的介绍。通过合理的并行运算＋内存优化，可将ARM上的运算耗时大大压缩。    
> 完整的 [测试代码](https://github.com/Yuefeng95/nn-test)

> TODO 关于运算pipeline的优化 (未填完的坑)
> 1. 乘法存在延迟，几个周期
> 2. 访存存在延迟，可以提前加载，2次load，4次浮点乘法操作
> 3. 访存范围小于cache，将加速。NEON寄存器与L1 L2直连
> 4. 预加载指令
> 5. 使用汇编代码测试加速极限
> 6. 使用汇编代码进一步优化速度
> [内联汇编的使用](./ASM.md)