## 基于FPGA异构平台的目标检测算法设计与实现

#### 1. 研究内容

​    本项目将基于FPGA与CPU异构平台特性并结合目标检测算法流程设计一种低功耗、高精度且具有实时性的目标检测解决方案。

​	典型的目标检测流程如图 1所示，光线信息通过以光学传感器为基础的视觉采集设备转化为数字信号并以视频流的形式输出，视频帧经过图像滤波、图像增强等预处理之后输入基于神经网络目标检测模块，最终模块将目标的坐标信息以及目标类型信息输出。将目标检测模型部署在异构平台将会面临任务划分、资源分配等问题，我们研究的重点也集中在如何通过软硬件协同设计实现高性能低功耗的目标检测方案。

![image-20210207155356146](./images/image-20210207155356146.png)

<center>图 1 目标检测流程与研究内容</center>

##### 1.1 视觉传感器的高速读取方案

​	高帧率、高分辨率的图像传感器往往会有更高的观测精度，但与此同时也会产生大量的视频信息数据，高速稳定的获取视觉传感器内容会为后续图像预处理与目标检测提供保障。本项目将探究如何高效地从视觉传感器中读取视觉信息。

​	为了提高传输效率通常视觉采集设备是以某种编码的视频流的形式输出图像信息，而图像处理的单位是视频帧。所以需要对视频流进行解码，再将其分割为多个视频帧。常见的编码方式包括H.264、H.265等，它们通过某种形式的编码去除视频帧数据中的冗余信息从而压缩视频信息。因为处理单元的处理速度要小于图像信息产生的速度，所以需要对解码后的视频帧进行数据缓存，并对缓存队列中的视频帧进行关键帧提取，以保证信息处理的实时性和低功耗的要求。关键帧数量取决于被观测物体的移动状态、环境光强度等因素，在不同的条件下关键帧数量是变化的，这需要设计一种带自反馈功能的视频帧读取策略，以在视频帧获取和视频帧处理部分进行协同控制。

​	**综上所述，为了完成视觉传感器的高速读取方案，本项目拟开展以下工作**：

​	1)  在FPGA上设计视频高速解码硬核；

​	2)  设计视频帧数据分级缓存机制，实现高吞吐量数据缓存；

​	3)  设计一种可控精度且适用于的FPGA关键帧提取方法；

​	在完成高速视频获取的基础上，本项目将利用提取出的关键帧进行后续的图像预处理过程以及目标识别过程。

##### 1.2 加速图像预处理

​	在进行图像目标检测之前需要对图像进行一系列预处理，基本的图像预处理方法包括图像滤波以及图像增强。图像滤波可以消除部分由视觉传感器或者环境带来的噪声，其方法为利用一个指定大小的滤波器对图像进行卷积操作，而图像增强可以增强图像细节提高图像对比度。按照输入输出之间是否有唯一且确定的传递函数滤波器可以分为线性滤波器和非线性滤波器，从增强处理的作用域出发可以分为频域增强与空域增强。图像处理中增强和滤波都属于底层处理，需要进行大量计算，是图像预处理过程中非常耗时的环节，所以本项目拟将图像运算处理方法部署到FPGA中，加速图像预处理。

​	**综上所述，在图像预处理增强方案中，本项目拟开展以下工作：**

​	1)  设计一种适配滤波器大小的高效行列寻址机制以优化流水线结构；

​	2)  RTL级的连续求和电路设计；

##### 1.3 基于FPGA 实现目标检测神经网络的搭建

​	在基于FPGA平台实现神经网络搭建的同时需要优化神经网络算法与硬件部署方式。FPGA加速网络卷积计算过程，计算吞吐量与FPGA平台提供的存储器带宽不匹配从而导致计算单元无法达到理想的加速效果。本项目拟从算法与计算资源两个层面探究神经网络部署优化方法，充分挥FPGA并行计算特性，达到加速神经网络卷积计算过程的效果。

​	**综上所述，在基于FPGA 平台搭建神经网络的任务中，本项目拟从以下两个方面进行优化：**

​	**算法层面：**

​	1) 在充分且均衡地利用FPGA硬件平台提供的所有计算资源的同时，如图2，图3所示，实现高效的循环展开，循环流水线化；

![image-20210207161614172](./images/image-20210207161614172.png)

<center>图2 循环展开优化</center>

![image-20210207161116355](./images/image-20210207161116355.png)

<center>图 3 FPGA Pipeline 优化</center>

​	2) 使用多种卷积层量化方法对计算进行量化，使得计算模式更适合于FPGA硬件平台；

​	3) 对神经网络模型进行剪枝以压缩网络规模，降低卷积计算规模；

​	**资源优化：**

​	1) 实现卷积运算基本运算单元的数据处理吞吐量与FPGA平台提供的带宽匹配；

​	2) 通过使用片上存储的方式减少算法中不同的循环迭代之间存在的冗余的存储器操作；

​	3) 受限于单个FPGA芯片计算资源与片上存储资源限制，采用FPGA集群部署大规模神经网络；

##### 1.4 算法在异构处理器架构上的拆分与部署

​	算法在异构处理器架构上的拆分与部署的目的是依据计算任务和计算特征在CPU与FPGA之间做任务分配。在目标检测工作中通常要进行纹理特征判断、色差判断统计、图像滤波去噪、边缘提取等操作，在基于CPU与DSP的视频处理方法无法满足高速高清图像处理要求的情况下，将并行任务部署到FPGA中是可行的方案，但是在部署过程中需对软件图像处理方法进行FPGA算法映射。而映射后处理速度与精度都会发生变化，所以建立一个准确的软硬件协同设计模型高效利用计算资源实现低功耗、高速度的目标识别是十分重要的。

​	**综上所述，本项目拟从以下方面建立软硬件协同计算模型：**

​	1） 任务访存行为建模，访存优化，通过在片上设置缓存区提升片外数据搬运速度、片内数据搬运速度、数据重用速度，尽可能减少访存的操作次数，从而达到系统整体的加速；

​	2） 计算资源建模，通过对包括CPU以及FPGA计算资源特性建模，依据算力以及时序分配任务，并通过数据重用和并行计算以达到预期的加速效果；

​	3） 计算任务建模，在建模过程中要量化任务完成时间、任务完成质量需求以及从任务层级分析是否可并行操作、是否可使用流水线结构；

#### 2. 拟解决的科学问题

​	1）建立循环展开高效流水模型：基于现有FPGA的硬件计算资源，针对卷积运算中的嵌套循环设计合理的循环展开规则以达到高效流水化效果，该模型的建立同时需要考虑基本运算单元的数据处理吞吐量与FPGA平台提供的带宽相匹配，最终达到加速神经网络卷积计算过程的效果。

​	2）建立卷积运算规模压缩模型：通过使用多种量化方法，如聚类、最小浮点等方法对浮点运算进行量化，使用定点执行引擎，以最大程度减少计算量，并基于神经网络的模型与量级设计裁剪方法，使用阈值与权值剪裁以压缩网络规模并分析网络剪枝效果，得出最优网络压缩方案。同时在此基础上对模型进行如图4所示的分块处理，从而更好的适配FPGA平台的资源，充分发挥FPGA的高并行度以加速网络模型的计算。

​	3）建立高效的异构计算与数据存储交互模型：使得CPU和FPGA在其算力承受能力之内，充分分担系统的目标检测任务，并探究计算中不同的循环迭代之间存在的冗余的存储器操作，从而建立片上数据片上存储的模型，同时考虑基本运算单元和片上缓冲区的组织及互连方式，建立片上数据交互模型，保证在异构平台架构下有效处理片上数据。

![image-20210207155954283](./images/image-20210207155954283.png)

<center>图4 卷积分块代码</center>

#### 3. 研究目标

​	本项目基于 FPGA 的并行计算特性及视觉目标检测等神经网络加速技术，力求达到以下目标：

​	1)  在FPGA上设计视频高速解码硬核，实现高速低功耗视频解码。

​	2)  设计高效流水的循环展开方案，可以针对卷积运算中的嵌套循环设计合理的展开规则，从而提高数据重用的效率和数据处理的并行性。

​	3)  设计针对神经网络卷积计算规模的压缩模型，结合多种量化方法，从而实现浮点运算模式向定点运算模式的转化，最大化发挥FPGA硬件平台的并行处理能力。并基于现有的网络裁剪方法，权衡检测精度要求与加速目标，探究最佳网络压缩方案，以适应FPGA有限的硬件资源，最终实现网络在FPGA硬件平台的成功部署。

​	4)  实现对CPU与FPGA处理任务的合理划分，使得计算量大，计算复杂的部分在FPGA进行，信号控制、数据预处理等操作在CPU进行，以建立高效的异构计算平台，并在此基础上准确分析片上数据存储结构与交互方式，可以使用片上数据本地存储的方式减少计算过程中循环迭代之间的冗余存取操作，优化片上逻辑组织互连形式，实现在异构平台架构下高效处理片上数据。

​	5)  从程序能耗比、实时性等方面对所提技术进行验证和评估。

#### 4. 研究方法

​	项目的整体研究方案，如图5所示，包括系统架构设计、目标检测算法特性以及硬件部署三个层面，讨论如何在基于FPGA的异构平台加速目标检测任务。拟提出一种通用的部署框架以及量化分析方法，使目标检测任务拆分与部署方法适用与大多数应用场景。

![image-20210207160042505](./images/image-20210207160042505.png)

<center>图5 整体方案架构图</center>

#### 5. 技术路线

##### 5.1 视觉传感器的高速读取技术

​	首先，在FPGA上实现视频高速解码硬核；

​	然后，设计视频帧数据分级缓存机制，实现高吞吐量数据缓存；

​	最后，设计一种可控精度且适用于的FPGA关键帧提取方法。

##### 5.2 图像预处理增强技术

​	首先，设计一种适配滤波器大小的高效行列寻址机制以优化流水线结构；

​	然后，RTL级的连续求和电路设计。

##### 5.3 基于FPGA实现目标检测神经网络的搭建技术

​	将神经网络模型部署在FPGA硬件平台，不仅要考虑硬件资源是否充足，还要考虑网络中基本运算单元的数据处理吞吐量与FPGA平台提供的带宽是否匹配，以实现最优加速效果。本项目将从计算与存储两个层面进行优化从而实现神经网络在硬件平台的成功、高效部署。

​	第一，基于FPGA硬件平台与卷积神经网络模型，进行硬件资源评估与数据传输带宽确认。

​	第二，针对卷积运算中的循环/嵌套循环设计循环展开方案，以实现高效流水化。

​	第三，对卷积层数据进行量化操作，进行低精度处理。

​	第四，分析神经网络的规模与量级，设计合理的网络压缩方案。

​	第五，借助FPGA片上缓存与优化片上逻辑组织形式，减少片上数据传输时间。

##### 5.4 算法在异构处理器架构上的拆分与部署技术

​	算法在异构处理器架构上的拆分旨在将高效的目标检测系统合理地分配在CPU端和FPGA端进行，包括对图像数据的预处理、硬件网络的加速、目标结果位置矫正。该项目拟采取以下思路对异构处理器架构上的算法进行拆分。

​	首先，为了便于 FPGA 中神经网络对图像的处理，本项目在CPU端对原始图像进行切割、转换等预处理操作，以提高网络的运行效率。

​	其次，在 FPGA 前馈计算中，考虑到内部有限的计算和存储资源，本项目拟采用自建轻量型 CNN 模型， 甚至在精度允许的范围内可将之转换为二值化网络，部署在FPGA端，从而实现图像识别速率的提升。

​	然后，本项目在FPGA端网络硬化中采用 Pipeline、Unroll、Partition 等 HLS 优化策略，结合 AXI4 协议进行传输、存储、计算优化，从而提高 FPGA 数据吞吐量，实现网络硬件加速。

​	最后，FPGA 将计算结果返回 CPU，利用自建的基于广度搜索的目标位置矫正策略优化识别位置，并得到最终结果。

#### 6. 实验方案

​	基于以上研究内容，本项目中所提方案的验证将从以下四个方面开展。首先，通过在FPGA上设计视频高速解码硬核与视频帧分级缓存机制，实现视频图像的高速读取；第二，针对图像进行预处理时耗时的问题，设计一种适配滤波器大小的高效行列寻址机制、加以RTL级连续求和电路设计，实现图像预处理过程的加速；第三，通过对卷积神经网络进行算法层面与存储层面的优化，实现神经网络在FPGA硬件平台的加速处理；第四，基于第三阶段的结果对算法进行拆分以优化加速效果，辅助以FPGA布局布线加速，以实现高精度且具有实时性的目标检测系统。
