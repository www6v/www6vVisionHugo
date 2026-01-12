---
title: Flamingo
---




# Flamingo[1]
### 架构
它在Frozen模型的基础上做进一步的改进，不同点主要有两个：一是使用了更大的LLMs，二是**冻结视觉编码器**，引入**perceiver resampler**和**XAttn-Dense**两个适配单元作为可训练的模块。
  - perceiver resampler：
      类似DETR，通过设计多个Perceiver Resampler来生成**64个固定长度的tokens**，主要作用在于可以**从图像中提取固定长度的特征向量**，能够解决图像甚至多帧视频的**feature map不一致的问题**。【图像和文本对齐】
  - XAttn-Dense：在每一层LLM上都会增加**corss- attention**以入到**LLM中与视觉向量进行交互**，**融合多模态信息**。【融合】


# 参考
### Flamingo
1xx. [[论文速览]Flamingo: a Visual Language Model for Few-Shot Learning[2204.14198]](https://www.bilibili.com/video/BV1pu411G7ce) V
1xx.  [DeepMind出手！多模态小样本打败精调](https://zhuanlan.zhihu.com/p/511517344)
1xx. [Otter  on OpenFlamingo](https://github.com/Luodian/Otter) git
1xx. [open_flamingo](https://github.com/mlfoundations/open_flamingo) git