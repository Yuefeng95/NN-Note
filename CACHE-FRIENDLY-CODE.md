> 文章翻译自英文博客 [What is a “cache-friendly” code?
](https://stackoverflow.com/questions/16699247/what-is-a-cache-friendly-code/16699282#16699282)

#  内存友好的代码（Cache Friendly Code）

现代计算机中，只有低等级内存结构（如寄存器）可以在一个机器中期中移动数据。然而，寄存器是非常昂贵的，在大多数计算机芯片中仅有几十个寄存器（几百或一千多字节的空间）。其他的存储模块（DRAM）与寄存器相比价格便宜许多，但请求一个数据需要数百个机器周期。