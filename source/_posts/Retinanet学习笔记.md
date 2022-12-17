---
title: 【目标检测】Retinanet学习笔记
date: 2022-12-16
tags: [机器学习,目标检测,论文]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171832604.webp
mathjax: true
---
## 1.Retinanet阅读笔记

### 1.1 网络结构
![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171827792.webp)

用Resnet作为特征提取层，之后经过FPN网络进行多尺度的特征融合。![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171826632.webp)

每一个尺度分配9个对应大小的anchor，每个尺度下有3x3个anchor(即3种scale的anchor和3种ratio构成9种anchor)。

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171826527.webp)

其中A=9，4A代表每个锚点对应的九个不同锚框对应的x,y,w,h，KA代表每个锚框对应的K个种类

主干特征提取网络如下

```python
import torch.nn as nn
import torch
import math
import torch.utils.model_zoo as model_zoo
from torchvision.ops import nms
from retinanet.utils import BasicBlock, Bottleneck, BBoxTransform, ClipBoxes
from retinanet.anchors import Anchors
from retinanet import losses

# FPN特征图尺度融合
class PyramidFeatures(nn.Module):
    def __init__(self, C3_size, C4_size, C5_size, feature_size=256):
        super(PyramidFeatures, self).__init__()

        # upsample C5 to get P5 from the FPN paper
        self.P5_1 = nn.Conv2d(C5_size, feature_size, kernel_size=1, stride=1, padding=0)
        self.P5_upsampled = nn.Upsample(scale_factor=2, mode='nearest')
        self.P5_2 = nn.Conv2d(feature_size, feature_size, kernel_size=3, stride=1, padding=1)

        # add P5 elementwise to C4
        self.P4_1 = nn.Conv2d(C4_size, feature_size, kernel_size=1, stride=1, padding=0)
        self.P4_upsampled = nn.Upsample(scale_factor=2, mode='nearest')
        self.P4_2 = nn.Conv2d(feature_size, feature_size, kernel_size=3, stride=1, padding=1)

        # add P4 elementwise to C3
        self.P3_1 = nn.Conv2d(C3_size, feature_size, kernel_size=1, stride=1, padding=0)
        self.P3_2 = nn.Conv2d(feature_size, feature_size, kernel_size=3, stride=1, padding=1)

        # "P6 is obtained via a 3x3 stride-2 conv on C5"
        self.P6 = nn.Conv2d(C5_size, feature_size, kernel_size=3, stride=2, padding=1)

        # "P7 is computed by applying ReLU followed by a 3x3 stride-2 conv on P6"
        self.P7_1 = nn.ReLU()
        self.P7_2 = nn.Conv2d(feature_size, feature_size, kernel_size=3, stride=2, padding=1)

    def forward(self, inputs):
        C3, C4, C5 = inputs

        P5_x = self.P5_1(C5)
        P5_upsampled_x = self.P5_upsampled(P5_x)
        P5_x = self.P5_2(P5_x)

        P4_x = self.P4_1(C4)
        P4_x = P5_upsampled_x + P4_x
        P4_upsampled_x = self.P4_upsampled(P4_x)
        P4_x = self.P4_2(P4_x)

        P3_x = self.P3_1(C3)
        P3_x = P3_x + P4_upsampled_x
        P3_x = self.P3_2(P3_x)

        P6_x = self.P6(C5)

        P7_x = self.P7_1(P6_x)
        P7_x = self.P7_2(P7_x)

        return [P3_x, P4_x, P5_x, P6_x, P7_x]
```

### 1.2 正负样本的匹配

针对每一个 anchor 与事先标注好的 GT box 进行比对，如果 iou 大于 0.5 则是正样本，如果某个 anchor 与所有的 GT box 的 iou 值都小于 0.4，则是负样本。其余的进行舍弃。

### 1.3 损失计算

### 4.6 Focal Loss

p_t值越大说明分类情况越好，在正样本y=1的情况下，p_t=p，p值越大说明分类越好，负样本y=0的时候,p_t=1-p，此时p越小说明分类效果越好，不管哪种情况分类效果越好p_t的值就越大

![image-20221102210517249](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171827137.webp)

alpha-balanced focal-loss

提供了一个超参数alpha，最终形式：

![image-20221102211123507](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171827618.webp)

或者：
$$
FL(pt)=−α_t(1−p_t)^γlog(p_t).
$$


对于 Focal loss，总结如下：

+ 无论是前景类还是背景类，$p_t$越大，权重$ (1-p_t)^{\gamma}$就越小，即简单样本的损失可以通过权重进行抑制；
+ $\alpha_t$用于调节正负样本损失之间的比例，前景类别使用$ \alpha_t$时，对应的背景类别使用$1-\alpha_t$

## 2. 其他

高斯模糊+拉普拉斯算子求二阶梯度+二值化+闭操作
