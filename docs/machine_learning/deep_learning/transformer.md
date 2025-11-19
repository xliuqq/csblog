# Transformer

## Attention is all you need

> 论文地址：[[1706.03762\] Attention Is All You Need](https://arxiv.org/abs/1706.03762)

核心创新点：Transformer 架构和 Self-Attention 自注意力机制。

序列转导模型（Sequence Transduction Model）

- 处理序列数据（具有顺序关系的数据）的模型，如文本翻译、文本生成、语音转文本等；

之前的实现：

- 基于 RNN/CNN，使用 Encoder-Decoder 结构，使用注意力机制增强；

Transformer 结构的创新：

- 完全摒弃 RNN/CNN，仍然使用 Encoder-Decoder 结构，完全基于注意力机制。



语义信息、位置信息、上下文信息（QKV） 



Bert 的位置编码方式不一样；

