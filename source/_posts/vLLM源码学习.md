---
title: vLLM源码学习
date: 2024-03-04 18:36:01
copyright: true
tags: [LLM, 推理加速, vLLM]
categories:
- 技术笔记
- LLM
---

&emsp;&emsp;本文中将学习一个大模型推理加速的工具 VLLM。vLLM 是伯克利大学LMSYS组织开源的大语言模型高速推理框架，旨在极大地提升实时场景下的语言模型服务的吞吐与内存使用效率。

<!--more-->

## 算法基本原理

- 论文：[Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/pdf/2309.06180.pdf)
- 官网：[vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention](https://blog.vllm.ai/2023/06/20/vllm.html)
### Paged Attention

- **类似操作系统中的虚拟内存和分页**

PagedAttention是vLLM的一个关键技术特性，它通过使用虚拟内存和分页技术来解决大型语言模型服务中的内存瓶颈。传统的注意力机制需要在GPU内存中保存所有输入令牌的键值对，这在自回归解码时生成新令牌。PagedAttention通过将键值对缓存分割成小块存储，提高了内存使用效率，几乎达到了最优化，只浪费少于4%的内存。
![vLLM中的Block Table](/SongXJ01/images/vLLM源码学习/vLLM中的BlockTable.png)

![KV cache](/SongXJ01/images/vLLM源码学习/KV_cache.png)


### Parallel sampling
由于两个输出共享相同的提示，vllm 只在提示阶段为提示状态的一个副本保留空间；两个序列的提示的逻辑块被映射到相同的物理块。总结而言，vLLM 能够共享用于跨多个输出样本存储提示的 KV 缓存的大部分空间。通过跨多个样本共享物理块，特别是对于长输入提示，可以大大减少内存使用。
![并行采样](/SongXJ01/images/vLLM源码学习/并行采样.png)


### Beam search 

- **通过使用树状搜索，避免完全局遍历**。
![树状搜索](/SongXJ01/images/vLLM源码学习/树状搜索.png)


### Shared prefix

- **共享前缀**：对于许多用户提示语具有相同前缀的情况，在 vLLM 中可以通过提供一组预定义共享前缀保留一组物理块来方便地实现。
![共享前缀](/SongXJ01/images/vLLM源码学习/共享前缀.png)


### Continuous batching
迭代调度处理，当部分序列处理完成，插入新序列。
![Continuous batching](/SongXJ01/images/vLLM源码学习/Continuous_batching.png)



### 

## 在 vllm 中增加自己的新模型

- [如何让vLLM适配一个新模型](https://zhuanlan.zhihu.com/p/680636375)
### 步骤

1. 从vLLM的代码仓库中把源码下载下来
2. 对模型中各模块的 **forward** 代码进行一定的修改
3. 实现张量并行及量化方法（可选步骤）
4. 实现加载模型的方法
5. 将新模型注册进 executor 的 init 模块

可参考：[Support Orion model by dachengai · Pull Request #2539 · vllm-project/vllm](https://github.com/vllm-project/vllm/pull/2539/files)
### 核心修改点

1. **Attention 机制的改写**：attention 机制中的 QKV 矩阵及输出 O 由原来的nn.Linear替换成了vLLM自带的QKVParallelLinear和RowParallel模块，方便之后并行处理张量。（此处注意 Attetion Head 的数量需要被 GPU 数量整整除）
2. **Forward 函数的改写**：因为现在使用的 self.attn 是 PageAttention，所以之前传入的参数通通可以用kv_cache 来替代，然后PageAttention底层来处理相应的分片。
3. **增加 load_weights 方法**：这里主要是使用一层循环，从HuggingFace的模型文件中加载权重，并将它们分配给模型中对应的层。


## 源码分析
[LLM推理框架2：vLLM源码学习](https://zhuanlan.zhihu.com/p/643336063)

### 加载模型

- llm 类加载模型时首先根据参数生成 llm_engine 类
![生成 llm_engine 类](/SongXJ01/images/vLLM源码学习/生成llm_engine类.png)


- llm_engine 类在创建时会加载 `tokenizer`，创建 `worker`（负责加载模型，执行推理）和 `scheduler`（负责并行计算），同时预分配内存。

![llm_engine 类中的核心步骤](/SongXJ01/images/vLLM源码学习/llm_engine类中的核心步骤.png)



### 推理生成

- 推理时，llm 类的 `generate()` 会调用 `_run_engine()` 函数，同时设置 sampling 策略。

![run_engine()](/SongXJ01/images/vLLM源码学习/run_engine.png)


- llm_engine 类中的 `step()`控制单步迭代
1. 安排下一次要执行的序列，其中的 SG 代表同一个提示生成的一组序列；
2. 调用 worker 执行模型
3. 根据采样参数更新模型输出

![step()](/SongXJ01/images/vLLM源码学习/step.png)

![step()的代码](/SongXJ01/images/vLLM源码学习/step的代码.png)

`step()` 中的 `_process_model_outputs()` 进行后处理

![_process_model_outputs](/SongXJ01/images/vLLM源码学习/process_model_outputs.png)


### 架构设计

- [vllm 架构设计 Top-down](https://zhuanlan.zhihu.com/p/681156634)
#### LLMEngine
LLMEngine 是一个处理请求并生成文本的大型语言模型引擎，通过精心设计的内存和调度策略，可以在分布式环境中高效生成文本。LLMEngine 包括：分词器（**tokenizer**）、一个可能分布在多个GPU上的语言模型，分配给中间状态的GPU内存空间（**KV cache**）。这个类通过迭代级调度和高效的内存管理来最大化服务吞吐量。
LLM 类将此类封装用于离线批量推理，而 AsyncLLMEngine 类封装此类用于在线服务。请注意，配置参数是从 EngineArgs 类派生的。要查看参数的完整列表，请参见 EngineArgs 类。
参数包括：

- model_config：与LLM模型相关的配置。
- cache_config：与KV缓存内存管理相关的配置。
- parallel_config：与分布式执行相关的配置。
- scheduler_config：与请求调度器相关的配置。
- device_config：与设备相关的配置。
- placement_group：用于分布式执行的Ray放置组，分布式执行时必需。
- log_stats：是否记录统计数据。

#### LLM
是对 LLMEngine 类的封装，用于处理离线的批量推理。

#### Worker
在GPU上执行模型（或其中部分）的类。
每个 Worker 与单个 GPU 相关联。Worker 负责维护 **KV cache ** 并在GPU上执行模型。在分布式推理的情况下，每个 Worker 被分配一个模型的分区。

![推理的基本过程](/SongXJ01/images/vLLM源码学习/推理的基本过程.png)

![](/SongXJ01/images/vLLM源码学习/架构_1.png)

![](/SongXJ01/images/vLLM源码学习/架构_2.png)


<br/><br/><br/><br/>

