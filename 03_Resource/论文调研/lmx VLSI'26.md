**标题：** A 122.2μJ/Token Reasoning LLM Accelerator with Reinforcement Fine-Tune Featuring Two-Level KV-Cache Compression and Spike-Driven Predictive Update

## feature1：

![[Pasted image 20261102132043.png|531]]

核心其实就是SVD低秩分解
[【无痛线代】彻底搞懂SVD！矩阵究竟怎么就奇异了？](https://www.bilibili.com/video/BV1XcfiBeEwQ/?share_source=copy_web&vd_source=c4a1d5841dc5113d3fbdde7cc4c38c58)
用**旋转矩阵**的方式进行低秩分解


## feature2：

![[Pasted image 20261102132211.png|550]]

核心是 minhash+局部敏感哈希

[LLM中常用minhash算法原理和代码讲解](https://www.bilibili.com/video/BV1XfHCz7EjY/?share_source=copy_web&vd_source=c4a1d5841dc5113d3fbdde7cc4c38c58)

[LLM中常用的LSH算法原理和代码讲解](https://www.bilibili.com/video/BV1rknjznERT/?share_source=copy_web&vd_source=c4a1d5841dc5113d3fbdde7cc4c38c58)


minhash：估算文本相似度的方法
局部敏感哈希：最近邻搜索

| 主要步骤                 | 输入              | 做了什么                                               | 输出                                         | 主要目的                 |
| -------------------- | --------------- | -------------------------------------------------- | ------------------------------------------ | -------------------- |
| **1. 提取特征**          | 回答序列 \(R_i\)    | 分词并获取 embedding 表示                                 | 高维特征，正文标注为 \(1024\times\mathrm{FP8}\)      | 将文本转换成可比较的数值特征       |
| **2. 生成 MinHash 签名** | 高维特征            | 进行多组哈希映射，每组取最小值，拼成签名                               | 短签名 \(S_i\)，规格为 \(16\times\mathrm{INT10}\) | 压缩表示，保留用于相似性判断的线索    |
| **3. 将签名分段**         | 16 项短签名 \(S_i\) | 切成 4 个 band；均分时每段包含 4 项                            | \(B_{i,0},B_{i,1},B_{i,2},B_{i,3}\)        | 允许通过部分签名匹配识别相似回答     |
| **4. 为每段生成桶编号**      | 一个 band 内的所有签名值 | 用移位、XOR、乘质数常量等操作，将整段编码为一个编号                        | 每个回答对应 4 个 Bucket ID                       | 把一段数据的比较简化成短编号比较     |
| **5. 检测桶碰撞**         | 不同回答的 Bucket ID | 比较**同一 band** 的编号是否相等；Fig. 4 将各 band 的相等结果通过 OR 汇合 | 回答之间的碰撞关系及桶成员记录                            | 快速识别可能相似的回答，替代高维余弦计算 |
| **6. 根据成员数量评分**      | 桶及其成员           | 统计成员数量；成员更多的桶获得更高分数                                | RM score                                   | 将回答之间的一致性转换为奖励信号     |
| **7. 生成后续更新信号**      | RM score        | 按原文描述，取分数倒数                                        | \(1/\mathrm{score}\)，作为后续更新的 loss 信号       | 让高分对应较小的损失           |

## feature3：