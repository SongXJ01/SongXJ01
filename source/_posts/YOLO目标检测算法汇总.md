---
title: YOLO目标检测算法汇总
date: 2023-08-09 16:36:01
copyright: true
tags: [计算机视觉, YOLO, 目标检测]
categories:
- 技术笔记
- 计算机视觉

top: 92
---


&emsp;&emsp;YOLO（You Only Look Once）是一系列目标检测算法，其核心思想是将目标检测视为回归问题，从而实现了端到端的训练。从最初的YOLOv1开始，该系列算法在速度和准确性方面不断取得突破。YOLOv1引入了单网络结构，实现了端到端的训练，虽然定位精度有待提高，但其高速性能为后续版本奠定了基础。YOLOv2（YOLO9000）通过引入批归一化、高分辨率分类器、锚点框等改进，显著提升了检测性能。YOLOv3则在保持高速的同时，通过多尺度预测、更好的基础分类网络和对象分类损失函数等优化，进一步提高了检测精度。而最新的YOLOv4和YOLOv5则在保持YOLO系列一贯的高速特性的同时，通过引入各种新的技巧和模块，如CSPNet、PANet、SiLU激活函数等，进一步提升了检测精度和速度。总体而言，YOLO系列算法以其高效、准确的特点，在目标检测领域持续引领创新，成为实际应用中的热门选择。

<!--more-->

<br/><br/>

---

<br/>

## 基本知识

#### 目标检测经典算法

-   两阶段：Faster-RCNN，Mask-RCNN系列
-   单阶段：YOLO系列
    -   优势：速度快，实时监测

#### 衡量指标

-   mAP：越大，效果越好。（面积之和）

    ![](/SongXJ01/images/YOLO目标检测算法汇总/image_oVjRjsHkrR.png)
-   IoU：预测框与ground truth的交集和并集的比值。

    ![](/SongXJ01/images/YOLO目标检测算法汇总/image_tUSzCO6bWA.png)
-   FPS：帧率，越大，速度越快，网络结构越简单
-   P和R：

    ![](/SongXJ01/images/YOLO目标检测算法汇总/image_4aqpOax2km.png)

<br/><br/>

---

<br/>

## 发展历程

![发展历程](/SongXJ01/images/YOLO目标检测算法汇总/image_-ZryUIZcpe.png)

<br/><br/>

## YOLOv1

### 概述

&emsp;&emsp;以往的二阶段检测算法，例如Faster-RCNN，在检测时需要经过两步：边框回归和softmax分类。由于大量预选框的生成，该方法检测精度较高，但实时性较差。鉴于此，YOLO之父Joseph Redmon创新性的提出了通过直接回归的方式获取目标检测的具体位置信息和类别分类信息，极大的降低了计算量，显著提升了检测的速度，达到了45FPS。

### **核心思想**

&emsp;&emsp;将整张图作为网络的输入，直接在输出层对BBox的位置和类别进行回归。

1. 给个一个输入图像，首先将图像划分成 7 × 7 的网格
2. 对于每个网格，我们都预测 2个边框（包括每个边框是目标的置信度以及每个边框区域在多个类别上的概率）
3. 根据上一步可以预测出 7 × 7 × 2个目标窗口，然后根据阈值去除可能性比较低的目标窗口，最后NMS去除冗余窗口即可

### 优缺点

**优点：**

-   快速，pipline简单；
-   背景误检率低；
-   通用性强。

**缺点：**

-   **对于小而密集的物体检测效率低：** 由于输出层为全连接层，因此在检测时，YOLO v1训练模型只支持与训练图像相同的输入分辨率。虽然每个格子可以预测 几个bounding box，但是最终只选择只选择IOU最高的bounding box作为物体检测输出，即每个格子最多只预测出一个物体。当物体占画面比例较小，如图像中包含畜群或鸟群时，每个格子包含多个物体，但却只能检测出其中一个。

<br/><br/>

## YOLOv2

### 概述

&emsp;&emsp;针对YOLOv1的问题，YOLO之父Joseph Redmon不甘屈服，对v1版本进行了大刀阔斧的改革，继而提出了YOLOv2网络，重要改革举措包括：

1.  更换骨干网络；
2.  引入PassThrough;
3.  借鉴二阶段检测的思想，添加了预选框。

&emsp;&emsp;YOLOv2检测算法是将图片输入到darknet19网络中提取特征图，然后输出目标框类别信息和位置信息。

### PassThrough操作

&emsp;&emsp;该方法将28x28x512调整为14x14x2048，后续v5版本中的**Focus**操作类似该操作。将生成的14x14x2048与原始的14x14x1024进行concat操作。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_36lEwKsrgk.png)

### 引入anchor

&emsp;&emsp;**引入anchor，调整位置预测为偏移量预测** 借鉴了Faster-RCNN的思想，引入了anchor，将目标框的位置预测由直接预测坐标调整为偏移量预测，大大降低了预测难度，提升了预测准确性。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_O18zvUQ-HJ.png)

### 优缺点

（1）优势：利用passthrough操作对高低层语义信息进行融合，在一定程度上增强了小目标的检测能力。采用小卷积核替代7x7大卷积核，降低了计算量。同时改进的位置偏移的策略降低了检测目标框的难度。

（2）不足：尚未采用残差网络结构。且当存在多物体密集挨着的时候或者小目标的时候，检测效果有待提升。

<br/><br/>

## YOLOv3

&emsp;&emsp;针对YOLOv2的问题，YOLO之父Joseph Redmon决定深化改革，于是乎吸收当下较好的网络设计思想，引入了**残差网络模块**。基本解决了小目标检测的问题，在速度和精度上实现了较好的平衡。

1.  在Darknet19的基础上推陈出新，引入残差，并加深网络深度，提出了Darkent53（类似于ResNet引入残差结构）。
2.  借鉴了**特征金字塔**的思想，在三个不同的尺寸上分别进行预测（多尺度预测 ，引入FPN）。
3.  分类器不再使用Softmax，分类损失采用binary cross-entropy loss（二分类交叉损失熵）

![YOLOv3网络结构图](/SongXJ01/images/YOLO目标检测算法汇总/image_OFLllUQUbS.png)

### 特征金字塔

&emsp;&emsp;YOLOv3检测算法是将图片输入到darknet53网络中提取特征图，然后借鉴特征金字塔网络思想，将高级和低级语义信息进行融合，在低、中、高三个层次上分别预测目标框，最后输出三个尺度的特征图信息（52×52×75、26×26×75、13×13×75）。

<br/><br/>

## YOLOv4

YOLOv4就是 **筛选** 了一些从YOLOv3发布至今，能够**提高检测精度**的**tricks**，并以YOLOv3为**基础**进行改进。

-   相较于YOLOv3的DarkNet53，YOLOv4用了CSPDarkNet53
-   相较于YOLOv3的FPN，YOLOv4用了**SPP**+PAN
-   CutMix数据增强和马赛克（Mosaic）数据增强
-   DropBlock正则化

![YOLOv4网络结构图](/SongXJ01/images/YOLO目标检测算法汇总/image_eDki8sf-KG.png)

### 输入数据采用Mosaic数据增强

&emsp;&emsp;借鉴了2019年CutMix的思路，并在此基础上进行了拓展，Mosaic数据增强方式采用了4张图片，随机缩放、随机裁剪、随机排布的方式进行拼接。从而对小目标的检测起到进一步的提升的作用。

![YOLOv4的效果](/SongXJ01/images/YOLO目标检测算法汇总/image_261WFN5zom.png)

### 修改骨干网络为 CSPDarknet53

&emsp;&emsp;借鉴了2019CSPNet的经验，并结合先前的Darkent53，获得了新的骨干网络CSPDarknet53。在CSPNet中，存在如下操作，即：进入每个stage先将数据划分为两部分，如下图中的part1、part2，区别在于CSPNet中直接对通道维度进行划分，而YOLOv4应用时是利用两个1x1卷积层来实现的。两个分支的信息在交汇处进行Concat拼接。

![CSPDarknet53](/SongXJ01/images/YOLO目标检测算法汇总/image_y1XwtlmUpj.png)

### 引入spp空间金字塔池化模块

&emsp;&emsp;引入SPP结构来增加感受野，采用1x1、5x5、9x9、13x13的最大池化的方式，进行多尺度融合，输出按照通道进行concat融合。

&emsp;&emsp;由于CNN网络后面接的全连接层需要固定的输入大小，故往往通过将输入图像resize到固定大小的方式输入卷积网络，这会造成几何失真影响精度。SPP模块就解决了这一问题，他通过三种尺度的池化，将任意大小的特征图固定为相同长度的特征向量，传输给全连接层。因为卷积层后面的全连接层的结构是固定的。但在现实中，我们的输入的图像尺寸总是不能满足输入时要求的大小，然而通常的手法就是裁剪(crop)和拉伸(warp)，但这样做总归是不好的，其扭曲了原始的特征。而SPP层通过将候选区的特征图划分为多个不同尺寸的网格，然后对每个网格内都做最大池化，这样依旧可以让后面的全连接层得到固定的输入。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_HZM9ceelJG.png)

### **在Neck部分采用FPN+PAN的结构**

&emsp;&emsp;借鉴2018年图像分割领域PANet, 相比于原始的PAN结构，YOLOV4实际采用的PAN结构将addition的方式改为了concatenation。如下图：

![](/SongXJ01/images/YOLO目标检测算法汇总/image_O9U0izUx4Y.png)

&emsp;&emsp;由于FPN结构是自顶向下的，将高级特征信息以上采样的方式向下传递，但是融合的信息依旧存在不足，因此YOLOv4在FPN之后又添加了PAN结构，再次将信息从底部传递到顶部，如此一来，FPN自顶向下传递强语义信息，而PAN则自底向上传递强定位信息，达到更强的特征聚合效果。

&emsp;&emsp;整个NECK结构如下图所示：

![](/SongXJ01/images/YOLO目标检测算法汇总/image_FsjBe7If49.png)

<br/><br/>

## YOLOv5

![YOLOv5网络结构图](/SongXJ01/images/YOLO目标检测算法汇总/d2f4aa5f2f69400b8e680c55760ad5f2_4SKImTYBPa.png)

&emsp;&emsp;2020年2月YOLO之父Joseph Redmon宣布退出计算机视觉研究领域，2020 年 4 月 23 日YOLOv4 发布，2020 年 6 月 10 日YOLOv5发布。

-   使用Pytorch框架，对用户非常友好，能够方便地训练自己的数据集，相对于YOLO v4采用的Darknet框架，Pytorch框架更容易投入生产。
-   代码易读，整合了大量的计算机视觉技术，非常有利于学习和借鉴。
-   能够轻松的将Pytorch权重文件转化为安卓使用的ONXX格式，然后可以转换为OpenCV的使用格式，或者通过CoreML转化为IOS格式，直接部署到手机应用端。
-   有非常轻量级的模型大小， YOLOv5 的大小仅有27MB ， 使用 Darknet 架构的 YOLOv4 有  244MB。

### 使用SPPF结构代替了SPP

&emsp;&emsp;主要区别就是MaxPool由原来的并行调整为了串行，值得注意的是：串行两个 5 x 5 大小的 MaxPool 和一个 9 x 9 大小的 MaxPool 是等价的，串行三个 5 x 5 大小的 MaxPool 层和一个 13 x 13 大小的 MaxPool 是等价的。虽然并行和串行的效果一样，但是串行的效率更高，降低了耗时。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_mZbM7N3E_B.png)

![](/SongXJ01/images/YOLO目标检测算法汇总/image__NkT1SV9e7.png)

### Focus操作

![Focus操作](/SongXJ01/images/YOLO目标检测算法汇总/image_1bR38WssFU.png)

<br/><br/>

## YOLOX

![YOLOX网络结构图-1](/SongXJ01/images/YOLO目标检测算法汇总/3fd25ff946474c6b8a51b47cd2c240bd_IWu_YJdvCV.png)

![YOLOX网络结构图-2](/SongXJ01/images/YOLO目标检测算法汇总/image_W9BH6ePYx9.png)

-   一般情况下，可以选择**Yolox-Nano、Yolox-Tiny、Yolox-s用于移动端部署**。
-   **Yolox-m、Yolox-l、Yolox-x用于GPU服务器部署**

&emsp;&emsp;YOLO检测器调整为了Anchor-Free形式并集成了其他先进检测技术（比如decoupled head、label assignment SimOTA）取得了SOTA性能，比如：

&emsp;&emsp;具有与YOLOv4-CSP、YOLOv5-L相当的参数量，YOLOX-L取得了50.0%AP指标同时具有68.9fps推理速度，指标超过YOLOv5-L 1.8%;

&emsp;&emsp;值得一提的是，作者使用的baseline是 YOLOv3 + DarkNet53（所谓的YOLOv3-SSP）

### 1. Decoupled Head 解耦头

&emsp;&emsp;检测头耦合会影响模型性能。采用解耦头替换YOLO的检测头可以显著改善模型收敛速度。解耦头结构考虑到 **分类** 和 **定位** 所关注的内容的不同。
同时为了避免计算量的大量增加，YOLOX的Decoupled Head结构（轻量解耦头），会先进行1x1的降维操作，然后再接上分类和定位两个并行分支（均为 3 × 3卷积）。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_Liqa3kRe4J.png)

### 2. Mosaic + Mixup 数据增强

-   **Mosaic数据增强**：基本原理就是在训练集中随机选择若干个（一般是4个）图像，经过裁剪拼接形成新的训练集元素，可以缓解训练集元素少或者增强识别能力。

-   Mixup数据增强： 对图像进行混类增强的算法，它将不同类之间的图像进行混合，从而扩充训练数据集

![](/SongXJ01/images/YOLO目标检测算法汇总/image_xmL0MKtaB8.png)

### 3. Anchor-free

&emsp;&emsp;YOLOv4、YOLOv5均采用了YOLOv3原始的anchor设置。然而anchor机制存在诸多问题：

(1) 为获得最优检测性能，需要在训练之前进行聚类分析以确定最佳anchor集合，这些anchor集合存在数据相关性，泛化性能较差；
(2) anchor机制提升了检测头的复杂度。

&emsp;&emsp;将YOLO转换为anchor-free形式非常简单，我们将每个位置的预测从3下降为1并直接预测四个值：即两个offset以及高宽。

[AnchorFree系列算法详解](https://blog.csdn.net/SMF0504/article/details/109214527 )

### 4. Multi positives

&emsp;&emsp;为确保与YOLOv3的一致性，前述anchor-free版本仅仅对每个目标赋予一个正样本，而忽视了其他高质量预测。参考FCOS，我们简单的赋予中心3×3区域为正样本。

### 5. SimOTA

&emsp;&emsp;SimOTA的作用是为不同目标设定不同的正样本数量，例如蚂蚁和西瓜，传统的正样本分配方案常常为同一场景下的西瓜和蚂蚁分配同样的正样本数，那要么蚂蚁有很多低质量的正样本，要么西瓜仅仅只有一两个正样本。对于哪个分配方式都是不合适的。

### 6. 主干网络（CSPDarknet）加入Fcous结构

&emsp;&emsp;主干网络加入Fcous结构，将图片宽高信息缩小，减小参数量，提升网络计算速度

&emsp;&emsp;Fcous结构：将输入的图片先经过Fcos结构对图片进行每隔一个像素取出一个值，得到四个特征层，然后再进行concat。从而图片宽高的信息缩小，通道数增加。在原始信息丢失较少的情况下，减小了参数量。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_C5q9dlW-Bg.png)

### 7. **SiLU激活函数**

&emsp;&emsp;**SiLU函数**相比于**ReLU**非线性能力更强，同时继承了**ReLU**收敛更快的优点。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_g1bv8gGFv9.png)

<br/><br/>

## YOLOv6

[YOLOv6网络结构图](https://blog.csdn.net/qq_34795071/article/details/125442065)

![](/SongXJ01/images/YOLO目标检测算法汇总/a0510238a9404914a6d341c0575973ad_yNpGFYrQkO.png)

&emsp;&emsp;YOLOv6是由美团推出的，所做的主要工作是为了更加适应GPU设备，将2021年的RepVGG结构引入到了YOLO。YOLOv6 主要在 Backbone、Neck、Head 以及训练策略等方面进行了诸多的改进：

-   统一设计了更高效的 Backbone 和 Neck：受到硬件感知神经网络设计思想的启发，基于 RepVGG style设计了可重参数化、更高效的骨干网络 **EfficientRep Backbone** 和 **Rep-PAN Neck**。
-   检测头部分模仿YOLOX，进行了解耦操作，优化设计了更简洁有效的 Efficient Decoupled Head，进一步降低了一般解耦头带来的额外延时开销。
-   在训练策略上，采用Anchor-free 无锚范式，同时辅以 SimOTA 标签分配策略以及 SIoU边界框回归损失来进一步提高检测精度。

### 1. 更适应GPU的骨干网络设计

&emsp;&emsp;YOLOv5/YOLOX 使用的 Backbone 和 Neck 都基于 CSPNet搭建，采用了多分支的方式和残差结构。对于 GPU 等硬件来说，这种结构会一定程度上增加延时，同时减小内存带宽利用率。按照RepVGG的思路，为每一个3x3的卷积添加平行了一个1x1的卷积分支和恒等映射分支，然后在推理时融合为3x3的结构，这种方式对计算密集型的硬件设备会比较友好。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_vc92ntzriY.png)

### 2. 简洁高效的 Decoupled Head

&emsp;&emsp;在 YOLOv6 中，采用了解耦检测头（Decoupled Head）结构，并对其进行了精简设计。原始 YOLOv5 的检测头是通过分类和回归分支融合共享的方式来实现的，而 YOLOX 的检测头则是将分类和回归分支进行解耦，同时新增了两个额外的 3x3 的卷积层，虽然提升了检测精度，但一定程度上增加了网络延时。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_zoR2HNFoDr.png)

### 3. **Anchor-free 无锚范式**

&emsp;&emsp;YOLOv6 采用了更简洁的 Anchor-free 检测方法。由于 Anchor-based检测器需要在训练之前进行聚类分析以确定最佳 Anchor 集合，这会一定程度提高检测器的复杂度；同时，在一些边缘端的应用中，需要在硬件之间搬运大量检测结果的步骤，也会带来额外的延时。而 Anchor-free 无锚范式因其泛化能力强，解码逻辑更简单，在近几年中应用比较广泛。经过对 Anchor-free 的实验调研，我们发现，相较于Anchor-based 检测器的复杂度而带来的额外延时，Anchor-free 检测器在速度上有51%的提升。

### 4. **SimOTA 标签分配策略**

&emsp;&emsp;为了获得更多高质量的正样本，YOLOv6 引入了 SimOTA 算法动态分配正样本，进一步提高检测精度。YOLOv5 的标签分配策略是基于 Shape 匹配，并通过跨网格匹配策略增加正样本数量，从而使得网络快速收敛，但是该方法属于静态分配方法，并不会随着网络训练的过程而调整。

### 5. **SIoU 边界框回归损失**

&emsp;&emsp;为了进一步提升回归精度，YOLOv6 采用了 SIoU 边界框回归损失函数来监督网络的学习。

&emsp;&emsp;SIoU损失函数由4个Cost函数组成：

-   Angle cost角度
-   Distance cost距离
-   Shape cost形状
-   IoU cost重合度

<br/><br/>

## YOLOv7

&emsp;&emsp;官方版的YOLOv7相同体量下比YOLOv5精度更高，速度快120%（FPS），比 YOLOX 快180%（FPS）。YOLOv7依旧基于anchor based的方法，同时在网络架构上增加E-ELAN层，并将REP层也加入进来，方便后续部署，同时在训练时，在head时，新增Aux\_detect用于辅助检测，对预测结果的一种初筛，有种two-stage的感觉。

![](/SongXJ01/images/YOLO目标检测算法汇总/f8a6ccbd93094b548804bc64b46468df_ZsJIM3i-zZ.png)

&emsp;&emsp;根据上图的架构图走一遍网络流程：先对输入的图片预处理，对齐成640\*640大小的RGB图片，输入到backbone网络中，根据backbone网络中的三层输出，在head层通过backbone网络继续输出三层不同size大小的**feature map**，经过RepVGG block 和conv，对图像检测的三类任务（分类、前后背景分类、边框）预测，输出最后的结果。

&emsp;&emsp;YOLOv7因为基于anchor based的目标检测，与YOLOv5相同，YOLOv6的正负样本的匹配策略则与YOLOX相同，YOLOv7则基本集成两家之所长。

&emsp;&emsp;YOLOv7大部分继承自YOLOv5，包括整体网络架构、配置文件的设置和训练、推理、验证过程等等；此外，YOLOv7也有不少继承自YOLOR，毕竟是同一个作者前后年的工作，包括不同网络的设计、超参数设置以及隐性知识学习的加入；还有就是在正样本匹配时仿照了YOLOX的SimOTA策略。

**优势：参数量和计算量大幅度减少**。

### 1. 正样本分配策略

-   中心点增加了一个0.5个单位的偏移扩散，提升召回。
-   为了让正样本更多
-   正样本分配，IoU计算，通过累加和动态筛选正样本

![](/SongXJ01/images/YOLO目标检测算法汇总/image_uITEYuxEGh.png)

&emsp;&emsp;YOLOv7的标签分配策略（正样本筛选），集成了YOLOv5和YOLOX两者的精华：

-   **YOLOv5** &#x20;

    Step1：Autoanchor策略，获得数据集最佳匹配的9个anchor（可选） &#x20;

    Step2：根据GT框与anchor的宽高比，过滤掉不合适的anchor &#x20;

    Step3：选择GT框的中心网格以及最邻近的2个邻域网格作为正样本筛选区域（辅助头则选择周围4个邻域网格）

![](/SongXJ01/images/YOLO目标检测算法汇总/image_ZtqkIEEhGJ.png)

-   **YOLOX** &#x20;

    Step4：计算GT框与正样本IOU并从大到小排序，选取前10个值进行求和（P6前20个），并取整作为当前GT框的K值 &#x20;

    Step5：根据损失函数计算每个GT框和候选anchor的损失，保留损失最小的前K个 &#x20;

    Step6：去掉同一个anchor被分配到多个GT框的情况

### 2. SPPCSPC模块

[https://yolov5.blog.csdn.net/article/details/126354660](https://yolov5.blog.csdn.net/article/details/126354660?spm=1001.2101.3001.6661.1\&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~Rate-1-126354660-blog-126531046.pc_relevant_recovery_v2\&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~Rate-1-126354660-blog-126531046.pc_relevant_recovery_v2\&utm_relevant_index=1)

&emsp;&emsp;总的输入会被分成三段进入不同的分支，最中间的分支其实就是金字塔池化操作，左侧分支类似于depthwise conv，但是请注意，中间的3×3卷积并未进行分组，依旧是标准卷积，右侧则为一个point conv，最后将所有分支输出的信息流进行concat。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_6RGn2lC09Q.png)

### 3. E-ELAN模块

![](/SongXJ01/images/YOLO目标检测算法汇总/image_jFVY-Qwc0g.png)

&emsp;&emsp;E-ELAN只改变了计算模块中的结构，而过渡层的结构则完全不变。作者的策略是利用分组卷积来扩展计算模块的通道和基数，将相同的group parameter和channel multiplier用于计算每一层中的所有模块。然后，将每个模块计算出的特征图根据设置的分组数打乱成G组，最后将它们连接在一起。此时，每一组特征图中的通道数将与原始体系结构中的通道数相同。最后，作者添加了G组特征来merge cardinality。除了维护原始的ELAN设计架构外，E-ELAN还可以指导不同的分组模块来学习更多样化的特性。

### 4. 辅助头检测

&emsp;&emsp;常用的方式是图（c）所示，即辅助头和引导头各自独立，分别利用ground truth和它们（辅助头、引导头）各自的预测结果实现标签分配。YOLOV7算法中提出了利用引导头的预测结果作为指导，生成从粗到细的层次标签，将这些层次标签分别用于辅助头和引导头的学习，如下图（d）和（e）所示。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_dkzANMbGMQ.png)

### 5. 复合模型缩放

&emsp;&emsp;类似于YOLOv5、Scale YOLOv4、YOLOX，一般是对depth、width或者module scale进行缩放，实现扩大或缩小baseline的目的。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_hGrBsf_uYx.png)

### 6.  **卷积重参化**

&emsp;&emsp;**引入了卷积重参化并进行了改进**采用梯度传播路径来分析不同的重参化模块应该和哪些网络搭配使用。同时分析出RepConv中的identity破坏了ResNet中的残差结构和DenseNet中的跨层连接，因此作者做了改进，采用没有Identity连接的RepConv结构进行卷积重参数化。下图是设计的用于PlainNet和ResNet的计划重参数卷积。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_P-a1KGH305.png)

### YOLOv7基础版本的区别

&emsp;&emsp;YOLOv7基础版本有三种，分别是YOLOv7、YOLOv7-tiny和YOLOv7-W6：

-   **YOLOv7**是针对普通GPU计算优化的基础模型。
-   **YOLOv7-tiny**是针对边缘GPU优化的基础模型。计算机视觉模型的后缀“小”意味着它们针对边缘AI和深度学习工作负载进行了优化，并且更轻量级，可以在移动计算设备或分布式边缘服务器和设备上运行ML。该模型对于分布式现实世界的计算机视觉应用程序很重要。与其他版本相比，边缘优化的YOLOv7-tiny使用leaky ReLU作为激活函数，而其他模型使用SiLU作为激活函数。
-   **YOLOv7-W6**是针对云GPU计算优化的基础模型。此类云图形单元(GPU)是用于运行应用程序以在云中处理大量AI和深度学习工作负载的计算机实例，而无需在本地用户设备上部署GPU。

<br/><br/>

---

<br/>

&#x20;

<br/><br/>

## 正负样本匹配策略对比

### YOLOv5的正负样本匹配策略

&emsp;&emsp;YOLOv5基于anchor based，在开始训练前，会基于训练集中gt框，通过k-means聚类算法，先验获得9个从小到大排列的anchor框。先将每个gt与9个anchor匹配（以前是IOU匹配，yolov5中变成shape匹配，计算gt与9个anchor的长宽比，如果长宽比小于设定阈值，说明该gt和对应的anchor匹配）

&emsp;&emsp;YOLOv5有三层网络，9个anchor, 从小到大，每3个anchor对应一层prediction网络，gt与之对应anchor所在的层，用于对该gt做训练预测，一个gt可能与几个anchor均能匹配上。
所以一个gt可能在不同的网络层上做预测训练，大大增加了正样本的数量，当然也会出现gt与所有anchor都匹配不上的情况，这样gt就会被当成背景，不参与训练，说明anchor框尺寸设计的不好。

&emsp;&emsp;在训练过程中怎么定义正负样本呢，因为yolov5中负样本不参与训练，所以要增加正样本的数量。gt框与anchor框匹配后，得到anchor框对应的网络层的grid，看gt中心点落在哪个grid上，不仅取该grid中和gt匹配的anchor作为正样本，还取相邻的的两个grid中的anchor为正样本。

&emsp;&emsp;如下图所示， **绿色的gt框中心点落在红色grid的第三象限里，那不仅取该grid,还要取左边的grid和下面的grid，** 这样基于三个grid和匹配的anchor就有三个中心点位于三个grid中心点，长宽为anchor长宽的正样本，同时gt不仅与一个anchor框匹配，如果跟几个anchor框都匹配上，所以可能有3-27个正样本，增大正样本数量。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_o0omt9JvWT.png)

### YOLOv6的正负样本匹配策略

&emsp;&emsp;YOLOv6的正负样本匹配策略同YOLOX，YOLOX因为是anchor free，anchor free因为缺少先验框这个先验知识，理论上应该是对场景的泛化性更好，同时参见旷视的官方解读：Anchor 增加了检测头的复杂度以及生成结果的数量，将大量检测结果从NPU搬运到CPU上对于某些边缘设备是无法容忍的。

&emsp;&emsp;OLOv6中的正样本筛选，主要分成以下几个部分：
①：基于两个维度来粗略筛选；
②：基于simOTA进一步筛选。
具体步骤如下：

![](/SongXJ01/images/YOLO目标检测算法汇总/image_Y3TQa7VFVz.png)

&emsp;&emsp;tie标签的gt如图所示，找到gt的中心点（Cx,Cy）,计算中心点到左上角的距离（l\_l,l\_t）,右下角坐标（l\_r,l\_b）,然后从两步筛选正样本：

&emsp;&emsp;第一步粗略筛选第一个维度是如果grid的中心点落在gt中，则认为该grid所预测的框为正样本，如图所示的红色和橙色部分 **，第二个维度是**以gt的中心点所在grid的中心点为中心点，上下左右扩充2.5个grid步长范围内的grid，则默认该grid所预测的框为正样本，如图紫色和橙色部分。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_XY1F6R5rc3.png)

第二步：通过SimOTA进一步筛选：

&emsp;&emsp;SimOTA流程如下：
①计算初筛正样本与gt的IOU，并对IOU从大到小排序，取前十之和并取整,记为b。
②计算初筛正样本的cos代价函数，将cos代价函数从小到大排列，取cos前b的样本为正样本。
同时考虑同一个grid预测框被两个gt关联的情况，取cos较小的值，该预测框为对应的gt的正样本。

### YOLOv7的正负样本匹配策略

&emsp;&emsp;YOLOv7因为基于anchor based , 集成v5和v6两者的精华，即YOLOv6中的第一步的初筛换成了YOLOv5中的筛选正样本的策略，保留第二步的simOTA进一步筛选策略。

&emsp;&emsp;同时YOLOv7中有aux\_head 和lead\_head 两个head ,aux\_head做为辅助，其筛选正样本的策略和lead\_head相同，但更宽松。如在第一步筛选时，lead\_head 取中心点所在grid和与之接近的两个grid对应的预测框做为正样本，如图绿色的grid，aux\_head则取中心点以及周围的4个预测框为正样本。如下图绿色＋蓝色区域的grid。

![](/SongXJ01/images/YOLO目标检测算法汇总/image_5XPTEswQNp.png)

&emsp;&emsp;同时在第二步SimOTA部分，lead\_head 是计算初筛正样本与gt的IOU，并对IOU从大到小排序，取前十之和并取整，记为b。aux\_head 则取前二十之和并取整。其他步骤相同，aux\_head主要是为了增加召回率，防止漏检，lead\_head再基于aux\_head 做进一步筛选。

<br/><br/>

## YOLOv5 和 YOLOX的对比

[深入浅出Yolo系列之Yolox核心基础完整讲解](https://zhuanlan.zhihu.com/p/397993315)

**YOLOv5：**

![](/SongXJ01/images/YOLO目标检测算法汇总/image_97fBqdMwff.png)

**YOLOX：**

![](/SongXJ01/images/YOLO目标检测算法汇总/image_JR25nDFTWY.png)

由上面两张图的对比，及前面的内容可以看出，**Yolov5s和Yolox-s主要区别**在于：

**（1）输入端：** 在Mosa数据增强的基础上，增加了Mixup数据增强效果；

**（2）Backbone：** 激活函数采用SiLU函数；

**（3）Neck：** 激活函数采用SiLU函数；

**（4）输出端：** 检测头改为Decoupled Head、采用anchor free、multi positives、SimOTA的方式。

在前面Yolov3 baseline的基础上，以上的tricks，取得了很不错的涨点。

<br/><br/>

## 参考

* [https://rockyding.blog.csdn.net/article/details/107199675](https://rockyding.blog.csdn.net/article/details/107199675?spm=1001.2101.3001.6650.4\&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-4-107199675-blog-126392748.pc_relevant_3mothn_strategy_and_data_recovery\&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-4-107199675-blog-126392748.pc_relevant_3mothn_strategy_and_data_recovery\&utm_relevant_index=9)
* [ YOLO家族进化史（v1-v7）](https://zhuanlan.zhihu.com/p/539932517 )


<br/><br/><br/><br/>
