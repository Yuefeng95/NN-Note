## ARM汇编 —— 获取优化运行耗时的极限

[ARM-NEON.md](ARM-NEON.md) 中介绍NEON的基本用法。[ASM.md](ASM.md)中介绍了内联汇编的使用。相比使用c api的neon内联函数，直接编写 汇编指令可实现更精准的CPU Pipeline控制。   
在对模型运算进行优化前，需要先对运算耗时做摸底，也就是拿到这些运算最优情况下的运算时间。这不包含内存和pipeline阻塞。这里介绍两种方式：1. 运行测试方式。采用汇编代码进行测试。 2. 直接计算方式。根据官方文档中对指令周期的叙述计算。

## 运行测试方式

测试vadd.i8指令的耗时   
实现代码：
```c
#include <iostream>
#include <chrono>

#define DURATION_MS(start, end) (std::chrono::duration_cast<std::chrono::microseconds>(end - start).count() / 1000.0)

inline void neon_vldls8_6() {
  // 每条3周期
  asm volatile (
    "vadd.i8 d10, d11, d12 \n"
    "vadd.i8 d12, d11, d12 \n"
    "vadd.i8 d13, d11, d12 \n"
    "vadd.i8 d14, d11, d12 \n"
    "vadd.i8 d15, d11, d12 \n"
    "vadd.i8 d16, d11, d12 \n"
  );
}

int main() {
  int loop = 1000;
  auto start = std::chrono::system_clock::now();
  // 运行6000次
  while (loop--) neon_vldls8_6();
  auto end0 = std::chrono::system_clock::now();


  std::cout << " end0: " << DURATION_MS(start, end0) << " ms" << std::endl;

  return 0;
}
```
运行结果显示时间为 0.015ms。   
在运行环境中输入指令
```shell
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
```
得到当前cpu的频率 1008000 千赫兹。
```
once_cycle = 1008000 * 0.015 / 6000
once_cycle: 2.52
```
得到vadd.i8指令的单次消耗时间为2.52个cpu周期。

## 直接计算方式

简单高效，直接查表 [Cortex_A57_Software_Optimization_Guide_external](./doc/Cortex_A57_Software_Optimization_Guide_external.pdf)，3.14 ASIMD Integer Instructions 的表格中标明VADD耗时3个周期，与计算结果相近。

## 总结

以上是计算最小运行耗时的两种方法，在对运算进行前，务必对运算时间进行摸底。