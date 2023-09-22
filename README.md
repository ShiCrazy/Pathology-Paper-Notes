# HIPT(Hierarchical Image Pyramid Transformer)论文及库的理解应用

HIPT原始库请参照[HIPT](https://github.com/mahmoodlab/HIPT)，其对应的CVPR论文为[CVPR](https://openaccess.thecvf.com/content/CVPR2022/papers/Chen_Scaling_Vision_Transformers_to_Gigapixel_Images_via_Hierarchical_Self-Supervised_Learning_CVPR_2022_paper.pdf)。 编写此库的原因主要是为了记录学习使用的过程，这不是成熟的代码/笔记库，而是自己学习的过程。



## 摘要

Vision Transformers (ViTs)能够用于捕捉图片中的信息，但是对图像分辨率要求通常为256*256或384*384
在20倍镜下拍摄得到的病理图像的分辨率可以高达150000×150000, 展示出不同分辨率下视觉向量的分层结构，16×16的images展示的可能是单独的细胞，而4096×4096的image展示的可能是肿瘤微环境之间的关联
作者提出了HIPT结构，这个结构基于Vit，能够使用两个level的自监督学习去学习高分辨率图像的表征信息。
HIPT用涵盖33个肿瘤类型的10,678张WSIs，408,218张 4k图像和近1亿张256×256图像来预训练HIPT模型
HIPT在9个slide-level任务上进行了基准测试，得到的结果是HIPT在肿瘤分型和生存预测上表现最佳，且自监督ViTs能够对肿瘤微环境中表型的分层结构建模重要的归纳偏见。


## Introduction

### 1. 
多实例学习（MIL）的基本框架如下：1）在单一放大倍数视角下进行组织分割，2）patch级别的特征提取构建嵌入向量，以及3）向量的全局池化得到单一的slide级别的表征向量

这种三步框架有一定的设计缺陷。首先，分割和特征提取总是先定在256*256区域。虽然能勉强得到精细的形态学特征（例如核异型和肿瘤存在），但是这样的固定窗口在捕捉粗糙特征（例如肿瘤入侵、肿瘤大小、淋巴细胞浸润和更宽的空间组织结构）上语义有限。

相比其他基于image的序列建模方法（例如ViTs），因为WSI的序列长度太大，MIL只能使用全局池化操作符来聚合向量。这使得Transformer模型中的注意力机制不能应用于学习长距离的依赖关系，特别是设计肿瘤-免疫定位这样的表型时。长距离的依赖关系指的是在一个很大的空间或时间范围内相互作用或关联。例如在一张WSI中，一个区域的特征可能与另一个远离它的区域有关。

虽然近期的MIL方法采用了很多自监督学习作为patch级别特征提取的策略，聚合层中的参数仍然需要有监督的训练。考虑到与ViTs相关的WSIs的基于片段的序列建模时，我们注意到使用Transformer注意机制的架构设计选择使得在ViT模型中对标记化（tokenization）和聚合层（aggregation layers）进行预训练成为可能。这在防止多实例学习（MIL）模型在低数据量环境中过度拟合或欠拟合方面具有重要意义。

相比普通图像，我们注意到对WSI的建模得到的视觉向量通常是在一个固定放大倍数的情况下的一个固定尺度。例如，在20x倍镜下扫描的对象其尺度通常接近0.5μm每像素，这使得视觉向量可以进行一致的比较，揭示出超出其正常参考范围的重要组织形态学特征。此外，全切片图像在20×放大倍数下也展示了不同图像分辨率的视觉令牌的分层结构：16×16的图像包含了细胞和其他细粒度特征（基质、肿瘤细胞、淋巴细胞）的边界框；256×256的图像捕捉到细胞与细胞之间的局部集群互动（肿瘤细胞度）；1024×1024-4096×4096的图像进一步描述了细胞集群之间以及它们在组织中的宏观规模互动（描述肿瘤浸润性与肿瘤远离的淋巴细胞的肿瘤-免疫定位程度）；最后，全切片图像的幻灯片级别描绘了组织微环境内的整体肿瘤异质性。

因此介绍了一个基于Transformer的模型架构，能够将不同层级的视觉标记进行聚合并在完整病理图像上进行预训练。对slide级别的表征学习使用了类似于在语言模型中长文本表征的学习方式，具体来说，是一种三级的分层结构，从256×256和4096×4096窗口下最底层[16*16]视觉标记到最终形成slide级别的表征、

两个贡献：①把学习一张slide完整表征的任务分解成可用自监督学习学习的分层相关的表征；②我们使用DINO来对每个聚合层进行自监督的预训练，目标区域的尺寸可高达4096*4096

在需要高度以来上下文的任务中（例如生存预测），HIPT和传统MIL的差别特别明显。

通过在HIPT模型的4096×4096表征上使用KNN，在slide-level的分类任务上效果比很多弱监督模型还好

在组织病理学图像处理中，使用自监督的ViTs模型能够有效学习不同粒度的视觉概念，例如在ViT256-16模型上学习到细粒度视觉概念（细胞位置），而在ViT4096-256模型上学习到粗粒度视觉概念，例如更广泛的肿瘤细胞性。这一点在图3、图4中得以体现

### 2.
多实例学习实例有很多例子，最早的起源如何，最早在病理图像上的应用如何。
HIPT是在一个固定的放大倍数下来进行patch的，用较大的patch尺寸来捕捉更宏观尺度的形态学特征，有助于推动在WSI上进行上下文建模的思考
ViT很重要，目前该架构的最新进展主要集中在提高效率和整合多尺度信息。与病理学相比，如果图像尺度在给定的放大倍数下是固定的，那么学习尺度不变性可能没那么必要。NesT和Hierarchical Perceiver通过Transformer模块将非重叠的图像区域特征进行划分和聚合。HIPT的一个区别是在于每个阶段的ViT模块都可以进行单独预训练以实现高分辨率下的编码（4096×4096）


## Method

### 1. 问题构建
用下面的符号来区分“images”的尺寸和与“images”相对应的“tokens”。对一张大小为L×L的图像**x**(即**x**<sub>L</sub>)，我们会将该图像**x**<sub>L</sub>中非重叠的大小为l*l的patches中提取得到的视觉向量记为{**x**<sup>(i)</sup><sub>l</sub>}<sup>M</sup><sub>i=1</sub> ∈ R<sup>M×d<sub>l</sub></sup>，M是序列的长度而d是对l大小的标记中提取的嵌入维度。对**x**<sub>256</sub>大小的自然图像，ViTs通常会使用l=16来产生一个M=256的一个序列。此外，我们将在一个L大小的图像上的[l×l]标记工作的ViT命名为ViT<sub>L</sub>-l

对于一张具有y结果的WSI图像**x**<sub>WSI</sub>，目标通常是解决slide级别的分类问题 P(y|**x**<sub>WSI</sub>)。传统的方法是使用三步MIL框架：1）[256*256] patching; 2) 标记向量化和3）全局注意力池化。**x**<sub>WSI</sub>通常是以序列{**x**<sup>(i)</sup><sub>256</sub>}<sup>M</sup><sub>i=1</sub> ∈ R<sup>M×1024</sup>来表示，这来自于使用预训练的ResNet-50编码器来得到的。由于用l=256得到的序列向量很长，计算资源要求高，所以需要对神经网络结构加以限制以降低复杂性，方法包括逐块池化和全局池化，最后得到一个整张WSI的嵌入向量，用于下游任务。

### 2. HIPT构造
在改造ViTs以适应slide级别表征学习的过程中，再次需要重复两个最重要的挑战：1）视觉标记区域的固定尺度以及他们在不同图像分辨率下的层级关系和2）将WSI展开后得到的巨大向量长度。 病理图像中的视觉标记区域通常是以对象为中心的，在不同分辨率下有不同的粒度。这些视觉标记区域是有重要的上下文依赖性的，比如肿瘤-免疫和肿瘤-基质。在高分辨率下使用小视觉标记区域来patch会导致序列长度过大，使自注意力机制难以处理，而低分辨率下使用大视觉标记区域又会导致细节丢失，综合考量之后，仍然选择在20倍镜下使用[256*256]的patch操作。
为了捕捉这种分层结构及在各个图像分辨率上的重要依赖关系，我们将WSI视作类似于长文档的嵌套聚合形式的视觉标记区域，这些视觉区域不断递归细分为更小的视觉标记区域直到细胞级别：

HIPT(**x**<sub>WSI</sub>) = ViT<sub>WSI</sub>-4096({CLS<sup>(k)</sup><sub>4096</sub>}<sup>M</sup><sub>k=1</sub>) → CLS<sup>(k)</sup><sub>4096</sub> = ViT<sub>4096</sub>-256({CLS<sup>(j)</sup><sub>256</sub>}<sup>256</sup><sub>i=1</sub>) → CLS<sup>(j)</sup><sub>256</sub> = ViT<sub>256</sub>-16({**x**<sup>(i)</sup><sub>16</sub>}<sup>256</sup><sub>i=1</sub>)

256是在**x**<sub>256</sub>和**x**<sub>4096</sub>图像中分别通过[16*16]和[256*256]进行patching得到的序列长度，而*M*是**x**<sub>WSI</sub>中**x**<sub>4096</sub>的图像数目。对每个notation，我们将**x**<sub>16</sub>图像视作细胞级，**x**<sub>256</sub>视作patch级，而**x**<sub>4096</sub>视作区域级别，而**x**<sub>WSI</sub>视作slide级别。在选择这些图像大小的过程中，数据标记区域的序列长度通常是M=256，主要是对ViT<sub>256</sub>-16和ViT<sub>4096</sub>-256的前向传输过程，而在ViT<sub>WSI</sub>-4096的的前向传播过程中，M通常小于256。在每个阶段，总的视觉标记区域的数量都按256的因子进行递减，通过对每个阶段选择小型的ViT骨架，小参数的HIPT得以在普通的服务器上使用。

ViT<sub>256</sub>-16用作用于细胞层面的聚合。给定一个256*256的patch，ViT将其展开为一系列不重叠的[16*16]令牌，然后通过一个线性嵌入层和附加的位置嵌入来生成一系列384维的嵌入向量，并添加一个可学习的CLS标签来聚合序列间的细胞嵌入。
ViT<sub>4096</sub>-256用作patch级别的聚合。
ViT<sub>WSI</sub>-4096用作区域级别的聚合。

### 3.分层预训练
提出了一个新挑战，即slide级别的自监督学习，以便在下游的诊断和预后任务重从WSI中得到slide级别的特征表示。目前可得到的slide级别的训练数据集通常只有100-10,000个数据点
首先使用DINO对ViT256-16进行预训练。然后在固定了ViT<SUB>256</SUB>-16的权重之后，将ViT<SUB>256</SUB>-16作为ViT<SUB>4096</SUB>-256的嵌入层，在DINO的第二阶段中重新使用。尽管分层预训练没有达到slide级别，仍然展示了1)在自监督评估中预训练的**x**4096表征在slide级别的分类任务中相比监督学习有竞争力；2）二阶段分层预训练的HIPT可以达到最优性能。

Stage 1：256×256 patch级别的预训练
DINO构建了一个学生网络和教师网络，分别记为φ<sub>s<sub>256</sub></sub>和φ<sub>t<sub>256</sub></sub>，这两个网络结构通常相似，但有不同的参数；对于每一个256×256的patch，DINO生成一组局部视图（M<sub>l</sub>）和全局视图<M<sub>g</sub>>来作为数据增强的方式，局部视图数量为8，大小为96×96；而全局视图数量为2，大小为224×224。学生网络和教师网络的输出之间的概率分布被匹配起来使用交叉熵函数来实现，根据损失用诸如SGD这样的优化算法来更新学生网络，用动量法来逐渐适应学生网络的变化
Stage 2: 4096×4096 region级别的预训练

## Experiment
略

