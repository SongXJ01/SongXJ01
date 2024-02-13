---
title: CUDA 学习笔记
date: 2024-01-15 16:54:49
copyright: true
tags: [CUDA, GPU, 并行计算]
categories:
- 技术笔记
- CUDA

top: 100
---


&emsp;&emsp;在这篇学习笔记中，我将从CUDA的编译开始，逐步深入到程序的运行与调试，旨在全面理解并掌握CUDA的工作流程。同时，我们还将触及CUDA的底层机制，包括内存操作、线程管理，以及如何通过代码优化提升程序的执行效率。希望通过这篇笔记，大家能够建立起对CUDA技术的完整认识，并在实践中不断提升自身技能。

<!--more-->

# 基本知识
## 编译

```bash
nvcc add.cu -o add_cuda
```
nvcc 编译时输出核函数寄存器使用情况 `-res-usage`（或`-Xptxas -v`）
```bash
nvcc add.cu -o add_cuda -res-usage -w
```
输出说明
```bash
nvcc thread_fence.cu -o thread_fence -res-usage -w

ptxas info: 35 bytes gmem
ptxas info: Compiling entry function '_Z6kernelPi' for 'sm_52'
ptxas info: Function properties for _Z6kernelPi
    16 bytes stack frame, 0 bytes spill stores, 0 bytes spill loads
ptxas info: Used 10 registers, 328 bytes cmem[0]
```

- `-res-usage`：显示资源使用情况的选项。它告诉编译器在编译过程中显示有关资源（如全局内存、常量内存、纹理内存等）使用情况的信息。
- `-w`：禁用警告信息的选项。

输出信息：

- `35 bytes gmem`：表示使用了35字节的**全局内存**。
- `Compiling entry function '_Z6**kernel**Pi' for 'sm_52'`：表示正在为名为 kernel 的入口函数编译为计算能力为 sm_52 的设备。
- `16 bytes stack frame, 0 bytes spill stores, 0 bytes spill loads`：表示函数使用了16字节的堆栈帧，并没有spill存储或加载操作。
- `Used 10 registers, 328 bytes cmem[0]`：表示函数使用了10个寄存器，328字节的常量内存，0字节的共享内存。

**【注意】**：

> - 一个空的核函数会占用 `2 registers，32bytes cmem[0]`
> - **例子**：每个 Thread 占用0个寄存器，每个Block有个128线程，且每个Block占用48KB共享内存。由于SM内共享内存大小限制，此时一个SM只能并发2个Block，即可并发2*128/32=8个Warp，该情况下的 SM Occupany = 8/64 = 12.5%，可见SM的利用率极低。对于 Tesla M6，为了将SM利用率达到100%，在块数够多的情况下，则：
>    - 每个Block有1024个Thread时，每个Block占用共享内存需小于等于48KB
>    - 每个Block有512个Thread时，每个Block占用共享内存需小于等于24KB
>    - 每个Block有256个Thread时，每个Block占用共享内存需小于等于12KB
>    - 每个Block有128个Thread时，每个Block占用共享内存需小于等于6KB

从以上信息可以看出，对于同一个 GPU 而言，Block 内的 Thread 数越多，每个 Block 可占用的共享内存上限越大。

## 运行

```bash
./add_cuda
```

快速检查
```bash
nvprof ./myApp
```


## 调试
官方调试方案：[Debugging Solutions](https://developer.nvidia.cn/debugging-solutions)

### CUDA-GDB
链接：[CUDA-GDB](https://developer.nvidia.cn/cuda-gdb)
编译成可调试版本：-g 表示将CPU代码编译成可调试版本，-G表示将GPU代码编译成可调试版本
```bash
nvcc -g -G main.cu -o main
```
运行
```bash
cuda-gdb main
```
开始
```bash
start
```
在53行设置断点，`c`是continue
```cpp
b 53
c
```
查看变量值
```cpp
p val_num
```
变换threads：跳到threadsIdx.x=0, threadsIdx.y=7的线程
```cpp
cuda thread (0,7,0)
```

![调试常用参数](/images/CUDA学习笔记/调试常用参数.png)



### **NVIDIA Nsight Systems**
链接：[NVIDIA Nsight Systems](https://developer.nvidia.cn/zh-cn/nsight-systems)
NVIDIA Nsight Systems是一款低开销性能分析工具，旨在为开发人员提供优化软件所需的洞察力。无偏差的活动数据可在工具中可视化，可帮助用户调查瓶颈，避免推断误报，并以更高的性能提升概率实现优化。

![NVIDIA Nsight Systems](/images/CUDA学习笔记/NVIDIANsightSystems.png)


<br/><br/>





# 内存操作


## 内存层次结构
&emsp;&emsp;在 GPU 内存层次结构中，最主要的两种内存是**全局内存**和**共享内存**。全局类似于CPU的系统内存，而共享内存类似于CPU的缓存。然而 GPU 的共享内存可以由 CUDA C 的内核直接控制。

![GPU内存结构](/images/CUDA学习笔记/GPU内存结构.png)


### 共享内存


&emsp;&emsp;共享内存是较小的片上内存，具有相对较低的延迟，并且共享内存可以提供比全局内存高得多的带宽。共享内存相较于全局内存而言，延迟要低大约20~30倍，而带宽高其大约10倍。
共享内存可以当作一个可编程管理的缓存。共享内存通常的用途有:

- 块内线程通信的通道 
- 用于全局内存数据的可编程管理的缓存 
- 高速暂存存储器，用于转换数据以优化全局内存访问模式

&emsp;&emsp;可以静态或动态地分配共享内存变量。在CUDA的源代码文件中，共享内存可以被声明为一个本地的CUDA核函数或是一个全局的 CUDA 核函数。如果在核函数中进行声明，那么这个变量的作用域就局限在该内核中。如果在文件的任何核函数外进行声明，那么这个变量的作用域对所有核函数来说都是全局的。CUDA支持一维、二维和三维共享内存数组的声明。
&emsp;&emsp;如果共享内存的大小在编译时是未知的，那么可以用 `extern` 关键字声明一个未知大小的数组（仅限一维）。

```shell
extern __shared__ int arr [];
```
&emsp;&emsp;因为这个数组的大小在编译时是未知的，所以在每个核函数被调用时，需要动态分配共享内存，如下所示：

```cpp
kernel<<<grid, block, isize * sizeof(int)>>>(...)
```

- **位置**：设备内存。
- **形式**：关键字`__**shared__**`添加到变量声明中。如`__**shared__** float cache[10]`。
- **目的**：对于GPU上启动的每个线程块，CUDA C编译器都将创建该共享变量的一个副本。线程块中的每个线程都共享这块内存，但线程却无法看到也不能修改其他线程块的变量副本。这样使得一个线程块中的多个线程能够在计算上通信和协作。
### 常量内存

- **位置**：设备内存
- **形式**：关键字`__**constant__**`添加到变量声明中。如`__**constant__** float s[10];`。
- **目的**：为了提升性能。常量内存采取了不同于标准全局内存的处理方式。在某些情况下，用常量内存替换全局内存能有效地减少内存带宽。
- **特点**：常量内存用于保存在核函数执行期间不会发生变化的数据。变量的访问限制为只读。NVIDIA硬件提供了64KB的常量内存。不再需要cudaMalloc()或者cudaFree()，而是在编译时，静态地分配空间。
- **要求**：当我们需要拷贝数据到常量内存中应该使用cudaMemcpyToSymbol()，而cudaMemcpy()会复制到全局内存。
- 性能提升的原因：
&emsp;&emsp;对常量内存的单次读操作可以广播到其他的“邻近”线程。这将节约15次读取操作。（为什么是15，因为“邻近”指半个线程束，一个线程束包含32个线程的集合。）
常量内存的数据将缓存起来，因此对相同地址的连续读操作将不会产生额外的内存通信量。

### 纹理内存（Texture Cache）

- 位置：设备内存
- 目的：能够减少对内存的请求并提供高效的内存带宽。是专门为那些在内存访问模式中存在大量空间局部性的图形应用程序设计，意味着一个线程读取的位置可能与邻近线程读取的位置“非常接近”。
- 纹理变量（引用）必须声明为文件作用域内的全局变量。
- 形式：分为一维纹理内存 和 二维纹理内存。

![纹理内存](/images/CUDA学习笔记/纹理内存.png)





## 同步
### 障碍点

&emsp;&emsp;`__syncthreads()`用于协调同一 block 中线程间的通信。它要求块中的线程必须等待直到所有线程都到达该点。

### 内存栅栏
&emsp;&emsp;内存栅栏确保栅栏前的任何内存写操作对栅栏后的其他线程都是可见的。memory fence不能保证所有线程运行到同一位置，只保证执行memory fence函数的线程生产的数据能够安全地被其他线程消费。根据所需范围，有3种内存栅栏：Block、Grid或System。

1. `__threadfence()`：一个线程调用__threadfence后，该线程在该语句前对全局存储器或共享存储器的访问已经全部完成，执行结果对**grid中**的所有线程可见。
2. `__threadfence_block()`：一个线程调用__threadfence_block后，该线程在该语句前对全局存储器或者共享存储器的访问已经全部完成，执行结果对**block中**的所有线程可见。
3. `__threadfence_system()`：可以跨系统(包括主机和设备)设置内存栅栏。__threadfence_system 挂起调用的线程，以确保该线程对全局内存、锁页主机内存和其他设备内存中的所有写操作对全部设备中的线程和主机线程都是可见的。

注意：**threadfence不是保证所有线程都完成同一操作，而只保证正在进行fence的线程本身的操作能够对所有线程安全可见，fence不要求线程运行到同一指令。**

（参考：[https://blog.csdn.net/yutianzuijin/article/details/8507355](https://blog.csdn.net/yutianzuijin/article/details/8507355)）



<br/><br/>



# 线程管理

## 硬件层面

![CUDA软件和硬件结构](/images/CUDA学习笔记/CUDA软件和硬件结构.png)


## 软件层面
&emsp;&emsp;一个SM（Streaming MultiProcessor）由多个CUDA core组成，每个SM根据GPU架构不同有不同数量的CUDA core。SM还包括特殊运算单元(SFU)，共享内存(shared memory)，寄存器文件(Register File)和调度器(Warp Scheduler)等。register 和 shared memory 是稀缺资源，**这些有限的资源就使每个SM中active warps有非常严格的限制，也就限制了并行能力。**


![CUDA软件结构](/images/CUDA学习笔记/CUDA软件结构.png)


&emsp;&emsp;由一个内核启动所产生的所有线程统称为一个**网格（Grid）**。同一 Grid 中的所有线程共享相同的全局内存空间。一个网格由多个**线程块（Block）**构成，其包含一组**线程（Thread）**，同一个 block 中的 thread 可以同步，也可以通过 shared memory 进行通信，不同块内的线程不能协作。

## Warp 线程束

&emsp;&emsp;SM 采用的 SIMT（Single-Instruction, Multiple-Thread，单指令多线程）架构，warp 是最基本的执行单元，一个 warp 包含 32 个并行 thread，这些 thread **以不同数据资源执行相同的指令**。当一个 kernel 被执行时，grid 中的线程块被分配到 SM 上，**一个 Block 中的 thread 只能在一个SM上调度**，SM 一般可以调度多个线程块。每个thread 拥有它自己的程序计数器和状态寄存器，并且用该线程自己的数据执行指令，这就是所谓的单指令多线程。
一个CUDA core可以执行一个thread，一个SM的 CUDA core 会分成几个 warp（即CUDA core在SM中分组），由 warp scheduler 负责调度。**一个SM同时并发的warp是有限的**，因为资源限制，SM要为每个线程块分配共享内存，而也要为每个线程束中的线程分配独立的寄存器，所以SM的配置会影响其所支持的线程块和 warp 并发数量。
一个warp中的线程必然在同一个 block 中，如果block所含线程数目不是warp大小的整数倍，那么多出的那些thread所在的warp中，会剩余一些inactive的thread。**由于warp的大小一般为32，所以block所含的thread的大小一般要设置为32的倍数。**



<br/><br/>

# CUDA 优化

## CUDA核函数中的printf原理
&emsp;&emsp;printf就是GPU往CPU回传数据，然后显示在终端窗口里。由CUDA负责安排多个线程之间的都有哪些数据需要被准备，又是什么格式的，当前kernel运行期间因为不能暂停，临时性的又将数据放到哪里。kernel结束后，检查是否又有需要回传的东西，回传，然后调用host上的printf。


## 优化 Host 和 GPU 之间的内存传输
### 常规传输方式 cudaMemcpy
&emsp;&emsp;在很多情况下都是最慢的方式，但他近乎适用于所有情况，所以也可能是被使用最多的方式。

```cpp
cudaMemcpy ( void* dst, const void* src, size_t count, cudaMemcpyKind kind )
```
### 高维矩阵传输 cudaMemcpy2D / cudaMalloc3D

&emsp;&emsp;以二维矩阵为例，可以用`cudaMalloc`来分配一维数组来存储一张图像数据，但这不是效率最快的方案，推荐的方式是使用`cudaMallocPitch`来分配一个二维数组，存取效率更快。`cudaMallocPitch`有一个非常好的特性是二维矩阵的每一行是内存对齐的，访问效率比一维数组更高。而通过`cudaMallocPitch`分配的内存必须配套使用`cudaMemcpy2D`完成数据传输。

```cpp
cudaMallocPitch ( void** devPtr, size_t* pitch, size_t width, size_t height )
```
```cpp
cudaMemcpy2D ( void* dst, size_t dpitch, const void* src, size_t spitch, size_t width, size_t height, cudaMemcpyKind kind )
```
&emsp;&emsp;相比于普通的`cudaMemcpy`，`cudaMemcpy2D`多了两个参数`dpitch`和`spitch`，他们是每一行的实际字节数，是对齐分配`cudaMallocPitch`返回的值。

### 异步传输 cudaMemcpyAsync / cudaMemcpy2DAsync / cudaMemcpy3DAsync
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/105957079/1697100322397-33ce42bf-7542-4061-a3db-3a75879a17b4.png#clientId=uf86abeb8-28e2-4&from=paste&height=85&id=u967794ea&originHeight=181&originWidth=1353&originalType=binary&ratio=1&rotation=0&showTitle=false&size=75812&status=done&style=none&taskId=u4a20f5be-b99b-4aed-b9bb-36520f395ba&title=&width=637)

```cpp
cudaMemsetAsync ( void* devPtr, int  value, size_t count, cudaStream_t stream = 0 )
```
```cpp
cudaMemcpy2DAsync ( void* dst, size_t dpitch, const void* src, size_t spitch, size_t width, size_t height, cudaMemcpyKind kind, cudaStream_t stream = 0 )
```

### 锁页内存
&emsp;&emsp;**当从可分页内存传输数据到设备内存时，CUDA驱动程序首先分配临时页面锁定的主机内存，将“可分页内存”复制到“锁页内存”中 [copy 1]，然后再从“锁页内存”传输到“设备内存”[copy 2]**。为了让传输只有一次，我们可以在主机端分配**锁页内存**。锁页内存是主机端一块固定的物理内存，它不能被操作系统移动，不参与虚拟内存相关的交换操作。简而言之，分配之后，地址就固定了，被释放之前不会再变化。GPU知道锁页内存的物理地址，可以通过“**直接内存访问（Direct Memory Access，DMA）**”技术直接在主机和GPU之间复制数据，传输仅一次，效率更高。

```cpp
cudaMallocHost ( void** ptr, size_t size )
```
`ptr`为分配的锁页内存地址，`size`为分配的字节数。
```cpp
cudaHostAlloc ( void** pHost, size_t size, unsigned int flags )
```
`pHost`为分配的锁页内存地址，`size`为分配的字节数，`flags`为内存分配类型，取值如下：

   - `cudaHostAllocDefault`：默认值，等同于cudaMallocHost
   - `cudaHostAllocPortable`：分配所有GPU都可使用的锁页内存
   - `cudaHostAllocMapped`：分配的锁页内存可实现零拷贝功能，主机端和设备端各维护一个地址，通过地址直接访问该块内存，无需传输。
   - `cudaHostAllocWriteCombined`：将分配的锁页内存声明为write-combined写联合内存，此类内存不使用L1 和L2 cache。

**分配的锁页内存必须使用**`**cudaFreeHost**`**接口释放。**
对于一个已存在的可分页内存，可使用`cudaHostRegister() `函数将其注册为锁页内存：
```cpp
cudaHostRegister ( void* ptr, size_t size, unsigned int flags )
```
**注意：**锁页内存相比可分页内存可减少一次传输过程，显著提高传输效率，但过多的分配会影响操作系统性能。对于图像这类小内存应用还是比较合适的。

### 零拷贝内存
&emsp;&emsp;使用零拷贝内存时不需要`cudaMemcpy`之类的显式拷贝操作，直接通过指针取值，所以对调用者来说似乎是没有拷贝操作。但实际上是在引用内存中某个值时隐式走PCIe总线拷贝，这样的方式有几个优点：

- 无需所有数据一次性显式拷贝到设备端，而是引用某个数据时即时隐式拷贝
- 隐式拷贝是**异步**的，可以和计算并行，隐藏内存传输延时

零拷贝内存是一块主机端和设备端共享的内存区域，是锁页内存，使用`cudaHostAlloc`接口分配，分配标志是`cudaHostAllocMapped`。
避免显式的数据传输，适用于数据量少且数据使用次数少的情况

参考：
[https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/](https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/)
[https://zhuanlan.zhihu.com/p/188246455](https://zhuanlan.zhihu.com/p/188246455)

## 多卡CUDA编程

### 多卡通信框架NCCL
[如何理解Nvidia英伟达的Multi-GPU多卡通信框架NCCL？ - 知乎](https://www.zhihu.com/question/63219175)

<br/><br/>


# CUDA Toolkit

[CUDA Toolkit Documentation-11.2](https://docs.nvidia.com/cuda/archive/11.2.0/index.html)


## Thrust
[Thrust :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/archive/11.2.0/thrust/index.html#abstract)

Thrust 是基于 STL 的 CUDA C++模板库。安装 CUDA 工具包会将 Thrust 头文件复制到系统的标准 CUDA 包含目录，无需进一步安装即可使用。

## cuBLAS
[cuBLAS :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/archive/11.2.0/cublas/index.html)

cuBLAS 是 CUDA 基本线性代数子程序库，它允许用户访问NVIDIA图形处理单元（GPU）的计算资源。

## cuSOLVER

[https://docs.nvidia.com/cuda/cusolver/index.html](https://docs.nvidia.com/cuda/cusolver/index.html)

&emsp;&emsp;cuSolver 库是基于 cuBLAS 和 cuSPARSE 库的高级包。它由对应于两组 API 的两个模块组成：

- 单个 GPU 上的 cuSolver API
- 单节点多GPU上的 cuSolverMG API

&emsp;&emsp;cuSolver 的目的是提供有用的类似 LAPACK （[https://netlib.org/lapack/](https://netlib.org/lapack/)）的功能，例如：矩阵分解、密集矩阵的三角求解、稀疏最小二乘求解器和特征值求解器。 此外，cuSolver 还提供了一个新的重构库，可用于求解矩阵序列稀疏模式。
cuSolver 将三个独立的组件组合在一起。

1. **cuSolverDN**，处理密集矩阵分解和求解，诸如 LU、QR、SVD 和 LDLT 等，以及矩阵和矢量排列等实用程序。
2. **cuSolverSP** 提供了一组基于稀疏矩阵的 QR 分解。 并非所有矩阵在因式分解中都具有良好的并行性稀疏模式，因此 cuSolverSP 库还提供了处理这些类似顺序矩阵的 CPU 路径。 对于那些具有丰富并行性的矩阵，GPU 路径将提供更高的性能。 
3. **cuSolverRF**，是一个稀疏重构包，可以提供非常好的求解矩阵序列时的性能，其中仅更改系数但稀疏模式保持不变。

## cuSPARSE
[https://docs.nvidia.com/cuda/cusparse/index.html](https://docs.nvidia.com/cuda/cusparse/index.html)


<br/><br/>




# CUDA 的版本敏感性
&emsp;&emsp;版本冲突记录（10.1、11.2、11.4）：cusparseScsrmv、cusparseDcsrmv 替换成 cusparseSpMV，参考链接：
[CUDA Toolkit Documentation-11.2](https://docs.nvidia.com/cuda/archive/11.2.0/index.html)
[CUDA Toolkit Documentation-10.1（POGS）](https://docs.nvidia.com/cuda/archive/10.1/index.html)
[CUDALibrarySamples/cuSPARSE/spmv_csr/spmv_csr_example.c at master · NVIDIA/CUDALibrarySamples](https://github.com/NVIDIA/CUDALibrarySamples/blob/master/cuSPARSE/spmv_csr/spmv_csr_example.c)

- cusparseScsr2csc (10.1) ➡️ cusparseCsr2cscEx2 (11.2)
- cusparseDcsrmv (10.1) ➡️ cusparseSpMV (11.2)
- 在 11.3 或更高版本中，NVIDIA 使用 _**CUSPARSE_SPMV_ALG_DEFAULT**_ 作为稀疏矩阵乘法标志；但在 11.2 或更早版本中，它们使用 _**CUSPARSE_MV_ALG_DEFAULT**_ 作为标志。

<br/><br/>

# CUDA 常用参考网站

- 不同版本的说明文档：[CUDA Toolkit Archive](https://developer.nvidia.cn/cuda-toolkit-archive)
- [CUDA Toolkit Documentation-11.2](https://docs.nvidia.com/cuda/archive/11.2.0/index.html)
- [CUDA Toolkit Documentation-10.1（POGS）](https://docs.nvidia.com/cuda/archive/10.1/index.html)


<br/><br/><br/><br/>