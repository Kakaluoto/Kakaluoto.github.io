---
title: 【论文】MSSL Hyperspectral and Panchromatic Images Fusion via Multiresolution Spatial–Spectral Feature Learning Networks
date: 2022-1-25
tags: [论文,机器学习]
cover: https://s4.ax1x.com/2022/01/25/7bJfeA.jpg
mathjax: true
---



# MSSL：通过多分辨率空间-光谱特征学习网络的高光谱和全色图像融合

## 摘要

高光谱hyperspectral(HS)和全色panchromatic(PAN)图像的融合是将高光谱(HS)图像的光谱信息与PAN图像的空间信息相结合，生成融合的高光谱(HS)图像。在本文中，我们提出了一种多分辨率空间-光谱特征学习(MSSL)框架来融合HS和PAN图像。该算法将现有的深度网络和复杂网络转换为多个简单子网和浅层子网，简化了特征学习过程。MSSL对HS图像进行上采样，对PAN图像进行下采样，并设计了具有光谱约束的多分辨率3-D卷积自动编码器(CAEs)网络，以学习HS图像的完整空间-光谱特征。MSSL设计了具有空间约束的多分辨率二维CAE，以较低的计算成本提取PAN图像的空间特征。为了有效地生成具有高空间和光谱保真度的全锐高光谱图像，提出了一种多分辨率残差网络对提取的高光谱空间特征进行重建。在三个广泛使用的遥感数据集上进行了大量实验，并与最先进的HS图像融合方法进行了比较，证明了所提出的MSSL方法的优越性。代码可以在https://github.com/Jiahuiqu/MSSL上找到。

### 关键词： 

**Convolutional autoencoder(CAE),hyper-spectral (HS) pansharpening,image fusion,multiresolution,spatial–spectral feature**

## I. Introduction

HS和PAN图像融合，也被称为HS pansharpening。

主要有四类: component substitution (CS), multiresolution analysis (MRA), Bayesian, and matrix factorization.

