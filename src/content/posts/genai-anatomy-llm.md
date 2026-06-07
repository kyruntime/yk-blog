---
title: "生成式 AI (3)：解剖大型语言模型"
description: "Tokenizer → Embedding → Transformer Layer → LM Head → SoftMax，一层层打开 LLM 黑盒"
pubDatetime: 2026-06-07T16:00:00.000Z
featured: true
tags:
  - 李宏毅课程
  - LLM原理
---

> 整理自李宏毅 2025「生成式 AI 导论」第 3 讲。前置知识：[第 1 讲](/posts/genai-how-it-works/)。

---

## 1. 全景：从输入到输出

```
"台湾最高的山是"
       │
       ▼ Tokenizer
  [Token IDs: 1423, 892, ...]
       │
       ▼ Embedding Table（查表）
  [Token Embeddings: 向量 ×N]
       │
       ▼ Layer 1 → Layer 2 → ... → Layer L
  [Contextualized Embeddings]
       │
       ▼ 取最后一个位置的向量
       │
       ▼ × LM Head 矩阵（V维）
  [Logits: 每个 Token 一个分数]
       │
       ▼ SoftMax（÷ Temperature）
  [概率分布: "玉" 72%, "阿" 15%, ...]
```

实际模型规模：
- **Llama 3B**：32 亿参数，28 层，词表 128K，Embedding 维度 3072
- **Gemma 4B**：43 亿参数，34 层，词表 262K，Embedding 维度 2560

---

## 2. 每一步在做什么

### 2.1 Tokenizer + Embedding

Token ID 通过 **Embedding Table**（一个巨大矩阵，行数 = 词表大小）查表得到向量。

关键特性：
- 相同 Token → 相同 Embedding（不管上下文）
- 语义相近的 Token → Embedding 也相近（如 apple 和 banana）

### 2.2 Transformer Layer（核心）

每层做两件事：

```
输入向量
    │
    ▼ Self-Attention（融合上下文）
    │  + Residual Connection
    ▼
    │
    ▼ Feed Forward Network（非线性变换）
    │  + Residual Connection
    ▼
输出向量（Contextualized Embedding）
```

**Self-Attention 机制：**

```
对每个 Token:
  Q = 向量 × W_Q    ← "我在找什么"
  K = 向量 × W_K    ← "我能提供什么"
  V = 向量 × W_V    ← "我的信息内容"

Attention Weight = SoftMax(Q · K^T)
输出 = Attention Weight × V
```

- **Multi-Head**：多组 QKV 矩阵并行，捕捉不同关注点（一个 Head 关注颜色，另一个关注数量）
- **Causal Attention**：只看左边（前面）的 Token，不能偷看未来
- **Positional Embedding**：加入位置信息（否则模型不知道 Token 顺序）

**Feed Forward Network**：两层线性变换 + 激活函数（ReLU/GeLU），相当于对每个位置独立做非线性处理。

### 2.3 LM Head + SoftMax

最后一层输出的向量 × LM Head 矩阵 → Logit 向量（V 维）→ SoftMax 转概率。

有趣的设计：很多模型（Llama、Gemma）的 LM Head **就是 Embedding Table 的转置**。本质上是在算"最终向量跟哪个 Token 的 Embedding 最像"。

---

## 3. 从中间层能看到什么

### Contextualized Embedding 解析歧义

同样是 "Apple"：
- 上下文是 "for breakfast" → representation 靠近 banana
- 上下文是 "the company" → representation 靠近 Microsoft

第 0 层相似度 100%，从第 1 层开始急剧分化。

### Logit Lens — 窥探每层在想什么

对每一层的输出都做 Unembedding（乘 LM Head），看模型在该层"心里想输出什么 Token"。

例子：法文→中文翻译时：
- 前几层：模型内部在想英文 "flower"
- 最后几层：才转换成中文 "花"

说明翻译是**逐层渐进**的，不是一步到位。

### 表征工程（Representation Engineering）

通过加减向量直接操控模型行为：
- 收集"拒绝回答"和"不拒绝回答"的中间层向量
- 求差 → 得到"拒绝向量"
- 把它加到某层的输出上 → 模型变得什么都拒绝
- 减掉它 → 模型什么都答应

---

## 4. Temperature 在这里的位置

Temperature 作用于 SoftMax 之前：

```
概率 = SoftMax(Logits / T)
```

| T | 效果 |
|---|------|
| T → 0 | 概率集中到最高分 Token（贪心） |
| T = 1 | 标准概率分布 |
| T > 1 | 概率变平坦，低概率 Token 也有机会 |

这就是你在 API 里调 Temperature 时，底层真正发生的事。

---

## 5. 对 Java 开发者的类比

| LLM 概念 | Java 类比 |
|----------|-----------|
| Embedding Table | HashMap<TokenID, float[]> |
| Transformer Layer | 一个处理管道（Pipeline），输入 List<float[]>，输出 List<float[]> |
| Self-Attention | Stream.map() + 关联查询（像 JOIN） |
| Multi-Head | 多线程并行处理不同维度 |
| LM Head | 最终分类器，输出 scores[vocabSize] |
| SoftMax | 归一化为概率分布 |

---

## 参考

- 李宏毅 2025「生成式 AI 与 ML 导论」第 3 讲：[YouTube](https://www.youtube.com/watch?v=8iFvM7WUUs8)
- 课程网页：[NTU Speech Lab](https://speech.ee.ntu.edu.tw/~hylee/GenAI-ML/2025-fall.php)
