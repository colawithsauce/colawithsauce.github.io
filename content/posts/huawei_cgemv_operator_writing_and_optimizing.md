+++
title = "Cgemv 算子的调优"
author = ["colawithsauce"]
draft = true
mathjax = "t"
+++

## Cgemv 算子介绍 <span class="tag"><span class="ATTACH">ATTACH</span></span> {#cgemv-算子介绍}

Gemv 主要是指形如

\begin{equation}
  y = \alpha \cdot \mathrm{op}(A) \cdot x + \beta \cdot y
\end{equation}

的计算，其中op(A)在transA为1时对A做转置，否则不转置。而 Cgemv 算子则是在这个基础上，将数据类型全部设定为复数。整个过程示意如下：

非转置情况
![](/ox-hugo/_20240129_135906screenshot.png)

转置的情况
![](/ox-hugo/_20240129_135935screenshot.png)

转置情况需要另外说明：这里转置是说将 A 先转置再进行计算。而不是说 A 已经转置了。这两者的区别是在于：前者（真实情况）指 A 仍然以 M\*N 的列优先矩阵的方式给出，但是同时还提出了一个要求即“请把A转置再参与运算”；后者（非真实的情况）是说我已经先行将 A 转置好了给你，你拿去参与运算就好了。这一点在理解Gemv类算子的转置情况的功能上是需要特别注意的。


## 目前的方案 <span class="tag"><span class="ATTACH">ATTACH</span></span> {#目前的方案}

我们的做法仍然是先对矩阵进行划分基块，然后再对基块进行处理。这样我们就将要处理的对象从任意大小的矩阵划分成了若干个固定大小（至少大小有上限）的矩阵。我们按竖直方向上分核，在水平方向上分步骤（下图中的1，2，3是同一个核的任务的三个步骤），这样，各个核的任务相互无关，从而获得了最大的并行性。

{{< figure src="/ox-hugo/_20240129_140005screenshot.png" >}}

具体到基块，由于 NPU 架构以向量运算为主，因此对于转置与非转置的情况，要有不同的处理

首先是非转置的做法，我采用的做法是做 szHorz 次点乘以矩阵。每次结果都累加到存放着 \\(\beta y\\) 结果的内存区。
![](/ox-hugo/_20240129_140032screenshot.png)

再看转置的做法：使用『向量点乘向量』的接口和『向量accumualte』的接口实现『向量乘以向量的转置』，并且将得到的『点结果』累加到放着 \\(\beta y\\) 的内存区。
![](/ox-hugo/_20240129_140049screenshot.png)


## 矩阵不整时的最后一块的处理 <span class="tag"><span class="ATTACH">ATTACH</span></span> {#矩阵不整时的最后一块的处理}

当矩阵不整时，我们的划分方法可能会使我们访问到本不应该访问的 Global Memory 内存区，造成内存越界错误。因此我们在读入矩阵 A 的部分的时候需要特别注意到处理最后一轮循环的情况。

以转置的情况为例，可以画图表示如下：
![](/ox-hugo/_20240129_150444screenshot.png)

用代码来描述就是这样

```cpp
// 针对不整的情况作特别处理
int64_t szVert_real = (idxVert == numVert - 1) ? (N - idxVert * szVert) : szVert;
int64_t szHorz_real = (idxHorz == numHorz - 1) ? (M - idxHorz * szHorz) : szHorz;
int64_t szHorz_real_pad = ROUND(szHorz * 8, 256) / 8; // 字节数要与 256 B 对齐以方便使用接口
```

在做\\(A\cdot x\\)的乘法时，非转置时，我们做点乘以向量时即使多乘了一截，最后累加到Y的时候也并无造成无效数据对于有效数据的覆盖。而在转置的情况下，我们做点乘的时候多乘了，却肯定会在accumulate的时候造成无效数据被累加有效数据的结果上。所以我们在转置的情况还需要对最后一块加一个Mask矩阵，使得这一截尾巴不会被算入结果中。


## 对于复数的处理 {#对于复数的处理}

GPU 编程中，复数类型的加减乘除是被重载过的函数，因此它能够直接完全复用Sgemv的算法流程。但是对于NPU编程，向量运算硬件本身便不支持这一特性，因此如何使用只支持单精度浮点数的向量处理器来实现复数的运算，便成为了需要解决的问题。

我们先将实数虚数进行分离，形成 `A_r`, `A_i`, 两个矩阵，及 `x_r`, `x_i`, `y_r`, `y_i` 四个向量，然后再相对于普通的Gemv的每次只做一个运算，Cgemv将做四次运算: `real = rr - ii`, `imag = ri + ir` ，其中的 rr, ii, ri, ir, 分别代指一次向量处理运算。最后我们再将所得到的 `y_r`, `y_i` 做虚实结合，得到元素为

```cpp
struct complex{
    float real;
    float imag;
};
```

的向量作为结果，存回 Global Memory 中。


## Cgemv 性能优化 {#cgemv-性能优化}


### 基块大小在主序上设置成 256B 的倍数。 {#基块大小在主序上设置成-256b-的倍数}

因为向量处理一次能够处理的数据量大小是 256B，所以这样分基块，FLOPS 数目将会更高
