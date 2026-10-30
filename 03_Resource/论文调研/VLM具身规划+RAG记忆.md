
## CacheBlend（ EuroSys'25）

**核心贡献**：在检索增强生成 (RAG) 场景下突破Prefix Caching的限制，实现non-Prefix KV Cache 的高效复用与推理加速

### RAG场景下的KV 复用

retrieval augmented generation
检索增强生成

输入：用户Query+检索出的相关文本块Text Chunks
![[Pasted image 20261029221025.png|540]]

RAG是一个典型的长上下文场景，具有以下特点：
1. KV cache复用以文本块为粒度；
2. 数据库包含的文档数量巨大，但是一小部分热文档会在请求中重复出现。

目标：
1. 在请求间复用相同文本块的KV cache；
2. 构建一个多级缓存架构，把hot chunk对应的KV cache存储到GPU，把cold chunk卸载到主存/磁盘

最简单的想法：
	缓存某一文本块的KV cache，在之后的请求中，如果需要该文本块，就直接复用缓存好的KV cache

实际上会遇到问题：
1. KV cache的位置编码错误：
	RAG在拼接文本块时是有顺序的，因此同一个chunk，虽然token相同，但是位置编码不同，导致KV也不同；
2. 文本块之间的交叉注意力缺失：
	在prefill阶段，位置在后的chunk的token，会注意到位置靠前的chunk的token。假设[chunk1，chunk2] 中，chunk2是复用的kv cache，那么后者则失去了对chunk1的注意力。

最差情况，为了保护精度，只能复用前面所有文本块都相同的KV cache

### 几种方案的比较
#### 完全的KV重计算
	完全没有复用，prefill 开销巨大
#### 只在prefix完全相同时复用
	· attention计算完全等效，没有质量损失
	· 服用条件极其苛刻，缓存占用空间极大，因为一个文档对应不同的位置时，存在不同的KV Cache块

![[Pasted image 20261029223921.png|543]]
#### 完全的KV复用
	· 复用最好
	· 位置编码缺失
	· 交叉注意力缺失
### CacheBlend方案

#### RoPE编码
 Rotary Position Embedding
 相对位置编码
 作用在token的q和k上
 将第$m$个token，旋转$m*\theta$角度

![[Pasted image 20261029224814.png|557]]

一个请求 [C1、C2、C3] ，如果缓存的 C1、C2、C3 的 KV Cache 都是从位置 0 开始的，拼接时只需要把 C2 每一层的 K 都旋转 Length(C1) 个角度，把 C3 每一层的 K 都旋转 Length(C1)+Length(C2) 个角度，就能恢复出正确的位置编码。

我们可以把拼接出来的 KV Cache 成为 **pre-computed KV Cache**。


#### 交叉注意力恢复
***attention map稀疏性***
![[Pasted image 20261029225137.png]]
**KV 偏差：一个 token 在有无文本块交叉注意力的情况下 k、v 的差别**

只要能够将KV偏差大的token，重新计算一下KV，采用recompute kv，而不是pre-compute kv，就能有效提高精度

![[Pasted image 20261029225342.png]]

那么，如何知道哪些token的KV偏差大呢？


***相邻层之间的相似性***
如果计算了真实的KV，再去获得KV偏差，是完全没意义的，因为没有节约计算

因此只能预测

**核心观察：上一层 KV 偏差大的 token 大概率也是下一层 KV 偏差大的**

![[Pasted image 20261030233224.png]]

在第一层 Layer 1 完整计算所有 token，
这样得到的 Layer 2 的 KV Cache 将和 pre-computed Layer 2 KV Cache 不同，
从中筛选出 KV 偏差大的 r1 个 token 重新计算，
这样在 Layer 3 中将至多有 r1 个 token 的 KV 与 pre-computed 的不同，
再其中再挑选出 KV 偏差大的 r2 个 token 重新计算，
以此类推……

作者形象地把这个方法称为 gradual filtering scheme（渐进过滤方案），下一层选择进行 reompute 的 token 是上一层 recompute token 的子集。


有效降低了注意力计算的复杂度
![[Pasted image 20261030233443.png]]


### Pipeline KV loading and recompute
![[Pasted image 20261030233627.png]]

### 完整架构
![[Pasted image 20261030233654.png]]

### 实验

相较于 Prefix Caching（SGLang），复用效率更高，TTFT 低，但生成质量可能会有所较低
相较于 Full KV Reuse（PromptCache），会有交叉注意力恢复产生的开销，但是生成质量更高

![[Pasted image 20261030233837.png]]



## RoboMemory
