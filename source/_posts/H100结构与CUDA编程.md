---
title: H100结构与CUDA编程
date: 2024-01-16 13:24:39
copyright: true
tags: [CUDA, GPU, 并行计算]
categories:
- 技术笔记
- CUDA
---




&emsp;&emsp;满配的 H100 有 8 个 GPC（图形处理集群，Graphics Processor Cluster），每个 GPC 有 9 个 TPC（纹理处理集群，Texture Processor Cluster），每个 TPC 内有 2 个 SM（Streaming MultiProcessor），总共有 144 个 SM。基于 SXM5（Server PCI Express Module） 的 H100 砍掉了 6 个 TPC，只有 66 个 TPC，总计 132 个 SM。

<!--more-->



![简化的H100结构](/SongXJ01/images/H100结构与CUDA编程/简化的H100结构.png)


- 对于每一个 GPU，通过 GPC 和 TPC 的层级可以划分为很多的 SM
- SM 进一步可以划分为 4 组，每组都有自己的 64KB Register File 和很多的计算核心，
- 同一个 SM 中所有运行的 thread 共享 256KB 的 Shared Memory 和 L1 Data Cache
- 同一个 GPU 内的所有 SM 共享 50MB L2 Cache 和 80GB HBM3 Memory
- 进一步向外看，同一个节点上的 GPU 通过 NVLink/NVSwitch 连接在一起


<br/><br/>

---

<br/>


## H100结构与CUDA编程

&emsp;&emsp;为了实现 GPU 并行加速计算，我们需要在 Host  上执行  `kernel launch`，让核函数在  Device 上的多个线程并发执行。CUDA 将核函数所定义的运算称为**线程（Thread）**，多个线程组成一个**块（Block）**，多个块组成**网格（Grid）**。具体的方式就是在调用核函数的时候通过`<<<grid, block>>>` 来指定核函数要执行的线程数量 N，之后 GPU 上的 N 个 Core 会并行执行核函数，并且每个线程会分配一个唯一的线程号 `threadID`，这个 ID 值可以通过核函数的内置变量 `threadIdx` 来获得。

&emsp;&emsp;一个线程需要两个内置的坐标变量 `（blockIdx，threadIdx）` 来唯一标识，它们都是 `dim3` 类型变量，其中 `blockIdx` 指明线程所在 grid 中的位置，而 `threaIdx` 指明线程所在 block 中的位置。下面即是一个典型的矩阵乘法的 CUDA 程序示例，定义 block 大小为 16 x 16，也就是每个 block 有 256 个 threads。grid 的值则根据矩阵大小算出来需要多少个 block。

&emsp;&emsp;为了进一步理解这里的 CUDA 编程模型概念与硬件结构，我们继续聊聊刚才没有提到的 WARP Scheduler。NVIDIA SM 采用 SIMT（Single Instruction Multiple Thread，单指令多线程）架构，线程束 warp 是最基本的执行和调度单元，一个 warp 一般包含 32 threads，这些 threads 以不同的数据资源执行相同的指令。

![三款显卡的计算能力比较](/SongXJ01/images/H100结构与CUDA编程/三款显卡的计算能力比较.png)


## Cluster

&emsp;&emsp;前面介绍的 CUDA 编程模型实际上都是在 Hopper 架构以前的抽象，也就是 grid/block 两级调度，block 映射到 SM 上。随着 Cooperative Groups 的引入和异步编程的支持，多个 Kernel 之间以生产者和消费者的方式通信，SM 到 SM 之间的通信带宽也在增加。在 Hopper 架构中，新增了 Distributed Shared Memory (DSMEM) 的概念，在一个 GPC 内部的 SM 有了专用的通信带宽，因此 CUDA 上新增了一层 Cluster 的调度层次。

![未加Cluster前的GPU架构](/SongXJ01/images/H100结构与CUDA编程/未加Cluster前的GPU架构.png)

![带有Cluster的GPU架构](/SongXJ01/images/H100结构与CUDA编程/带有Cluster的GPU架构.png)

![threads cluster](/SongXJ01/images/H100结构与CUDA编程/threads_cluster.png)


&emsp;&emsp;有了 cluster 这一层抽象之后，类似于同一个 block 的 threads 都会被调度到同一个 SM，**同一个 cluster 的 thread blocks 都会被调度到同一个 GPC 中**。

![Cluster 和 GPC](/SongXJ01/images/H100结构与CUDA编程/ClusterGPC.png)


&emsp;&emsp;这样同一个 cluster 中不同 block 的 threads 可以通过 SM to SM Network 访问另一个 block 的 DSMEM（Distributed Shared Memory）。这样在一个 GPC 内部实现多个 SM 的Load/Store，Atomic，Reduce 和异步 DMA（Direct Memory Access，通过锁页内存直接访问） 操作都变得非常的简洁。

![block之间的交互](/SongXJ01/images/H100结构与CUDA编程/block之间的交互.png)


**CUDA 编程模型的内存模型也就变成了下图所示：**
![新的CUDA内存模型](/SongXJ01/images/H100结构与CUDA编程/新的CUDA内存模型.png)


&emsp;&emsp;本质上看，CUDA 引入 block 和 cluster 的抽象，都是为了更好地利用空间局部性原理。Block 可以让所有的 threads 调度到同一个 SM，让 threads 可以通过 fast barriers 快速同步，并且通过 SM 的 Shared Memory 交换数据。**随着 GPU 的 SM 越来越多，仅仅使用 Block 这一层抽象已经不能够更好地利用局部性原理，因此 Hopper 引入了 Cluster 这层抽象，让所有的 threads 运行在同一个 GPC 内部。**

**参考链接：**
[疯狂的 H100：现代 GPU 体系结构浅析，从算力焦虑开始聊起](https://mp.weixin.qq.com/s/ccSHfgus5GvG6OleeOfglw)
[In The Loop | 疯狂的 H100：现代 GPU 体系结构浅析，从算力焦虑开始聊起](https://loop.houmin.site/context/gpu-arch/)



## PCI 和 SXM
V100 有两种总线规格：**PCIe** 和 **SXM2**。

![V100的总线规格](/SongXJ01/images/H100结构与CUDA编程/V100的总线规格.png)


- **PCIe**（Peripheral Component Interconnect express）是一种高速串行计算机扩展总线标准，是英特尔公司在2001年提出来的，它的出现主要是为了取代AGP接口，优点就是兼容性比较好，数据传输速率高、潜力大。
- **SXM**（Server PCI Express Module）是英伟达公司设计出来的，它的出现主要是为高性能计算和数据中心提高更强的计算能力和传输速度。SXM接口的GPU通常是存在于**DGX系统板**上，该DGX系统板支持4张GPU-SXM或则8张GPU-SXM，而每个GPU之间通过**NVLink**进行通信。 

![这是 H100 GPU 封装在 SXM5 模块中](/SongXJ01/images/H100结构与CUDA编程/SXM5模块.png)

![NVIDIA HGX H100，由8个H100 SXM5 模块加上4个NVSwitch Chip 在同一个 system board 上](/SongXJ01/images/H100结构与CUDA编程/HGX.png)

![DGX H100 在 HGX 100 的基础上，进一步配置了 CPU、存储与网卡](/SongXJ01/images/H100结构与CUDA编程/DGX.png)


- DGX（Deep-learning GPU Accelerator）：NVIDIA 推出的一系列专门用于加速深度学习工作负载的高性能计算平台
   - DGX系统板：[https://www.nvidia.cn/data-center/dgx-1/](https://www.nvidia.cn/data-center/dgx-1/)
   - DGX-1：8块 V100 （[技术文档](https://www.nvidia.cn/content/dam/en-zz/Solutions/Data-Center/dgx-1/dgx-1-rhel-centos-datasheet-update-r2.pdf)）

**NVLink 与 NVSwitch ：**
NVLink 是一种 GPU 之间的直接互连，可扩展服务器内的多 GPU 输入/输出 (IO)。
NVSwitch 可连接多个 NVLink，在单节点内和节点间实现以 NVLink 能够达到的最高速度进行多对多 GPU 通信。

![NVSwitch 使 NVIDIA DGX H100 系统中的 8 个 GPU 能够在一个具有全带宽连接的集群中协同工作。](/SongXJ01/images/H100结构与CUDA编程/NVSwitch.png)


参考：

- [https://www.nvidia.cn/data-center/nvlink/](https://www.nvidia.cn/data-center/nvlink/)
- [https://www.bilibili.com/read/cv24855760/](https://www.bilibili.com/read/cv24855760/)



<br/><br/><br/><br/>