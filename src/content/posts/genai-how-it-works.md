---
title: "生成式 AI (1)：一堂课搞懂原理"
description: "Token 接龙、概率采样、Chat Template、多模态、AI 幻觉 — 李宏毅 2025 课程精华"
pubDatetime: 2026-06-07T08:00:00.000Z
featured: true
tags:
  - 李宏毅课程
  - LLM原理
---

> 整理自李宏毅 2025「生成式 AI 导论」课程。目标读者：写过业务代码、但没碰过 LLM 底层的 Java 开发者。

---

## 1. LLM 本质 = 文字接龙

**一句话：** 大语言模型不做"理解"，只做一件事——给定前文，预测下一个 Token 的概率，采样一个拼上去，循环往复。

```
你输入: "台湾最高的山是"
              │
              ▼
        ┌───────────┐
        │   LLM     │  输出概率分布:
        │  (接龙机)  │  "玉山" 72%  "阿里山" 15%  "雪山" 8% ...
        └───────────┘
              │
              ▼ 采样（非永远取最高）
        拼上 "玉山"
              │
              ▼ 把 "台湾最高的山是玉山" 再喂回去
        ┌───────────┐
        │   LLM     │  下一个 Token: "，" 45%  "。" 30% ...
        └───────────┘
              │
              ▼ 重复，直到遇到结束符或达到长度上限
        最终输出完整句子
```

这就是 **Autoregressive Generation（自回归生成）**。你看到的"流式输出"就是每生成一个 Token 就推给你一次。

### Token ≠ 字

Tokenizer 把文本切成模型词表里的编号。**一个字可能被切成多个 Token，一个 Token 也可能对应多个字或英文片段。**

| 概念 | 说明 |
|------|------|
| LLaMA 词表 | 约 128,000 个 Token，覆盖多语言、标点、代码片段 |
| 中文 "人工智能" | 可能被切成 `["人工", "智能"]` 两个 Token |
| 英文 "ing" | 常作为独立 Token 出现，便于复用 |

举例：前文是"人工"，模型输出 `"智"` 50%、`"呼"` 20%、`"岛"` 5%……采样策略决定最终接哪个。

**采样策略决定"创造力"：**

| 参数 | 做了什么 | 典型值 | 使用场景 |
|------|----------|--------|----------|
| **Temperature** | 缩放 logits 再 softmax。越低分布越尖锐（趋向贪心），越高越平坦（更随机） | 0~2，默认 0.7~1.0 | **最常调的旋钮**：写代码/做数学 → 0；写文案/头脑风暴 → 1.2+ |
| **Top-K** | 只保留概率最高的 K 个 Token，其余直接归零后重新归一化 | 40~100 | DashScope 支持，OpenAI API 不直接暴露 |
| **Top-P (nucleus)** | 按概率从大到小累加，直到累积概率 ≥ P，只从这些 Token 里采样 | 0.9~0.95 | OpenAI 默认方案，和 Temperature 配合使用 |

三者关系：Temperature 先"拉平/压尖"分布，Top-K / Top-P 再"砍掉尾巴"。实际 API 调用通常只调 **Temperature + Top-P**（OpenAI 风格）或 **Temperature + Top-K**（DashScope 风格），不必三个同时动。

### 直观理解 Top-P

假设模型对下一个 Token 的概率是：

```
"玉" → 50%    累积 50%  ✓
"阿" → 20%    累积 70%  ✓
"雪" → 15%    累积 85%  ✓
"合" →  8%    累积 93%  ✓ ← 刚过 90%，到此为止
"大" →  4%    ✗ 砍掉
"奇" →  2%    ✗ 砍掉
```

**Top-P = 0.9：** 从大到小累加到 ≥ 90%，只在这些 Token 里采样。

对比 Top-K：
- **Top-K = 3** → 死板取前 3 个（不管第 4 名概率多高）
- **Top-P = 0.9** → 自适应：分布集中时候选少，分布平坦时候选多

一句话：**Top-K 按排名截断，Top-P 按累积概率截断。** Top-P 更灵活，所以 OpenAI 默认用它。

> **收：** 没有"思考过程"藏在里面。你问它任何问题，底层都是同一个循环：算概率 → 抽一个 → 拼接 → 再算。

---

## 2. 为什么能"回答问题"— Chat Template

**一句话：** 模型只会接龙，不会区分"用户说的"和"我该回答的"——是 Chat Template 用特殊标记把对话结构编码进输入里。

你在界面上打：

```text
台湾最高的山？
```

模型实际看到的是套了 Template 的长文本，大致结构：

```text
[system] 你是助手，今天是 2026-06-07。
[user]   台湾最高的山？
[assistant] ← 模型从这里开始接龙
```

`assistant` 标记之后是接龙起点——模型"学到"了：这个位置该续写助手回复。

### Message 结构

各厂商格式不同，但逻辑一致。OpenAI 风格：

```json
[
  {
    "role": "system",
    "content": "你是 PayReach 助手。今天是 2026-06-07。只回答展会和支付相关问题。"
  },
  {
    "role": "user",
    "content": "台湾最高的山？"
  }
]
```

| role | 作用 |
|------|------|
| `system` | 人设、日期、行为边界——用户通常看不到，但决定模型"扮演谁" |
| `user` | 用户输入 |
| `assistant` | 历史回复 + 模型当前要续写的位置 |
| `tool` | 工具执行结果（Function Calling 场景） |

调用 API 时，SDK 把 messages 数组套进对应模型的 Template，转成一长串 Token 序列再送进模型。

> **收：** "聊天"是工程层包装出来的体验。模型眼里只有一段文本，Template 告诉它"哪里该你接话了"。

---

## 3. 模型怎么学会接龙的 — 训练三阶段

**一句话：** 先在海量文本上学语言规律，再用人工问答教它当助手，最后用人类偏好打磨语气与安全边界。

```
互联网文本（TB 级）
        │
        ▼ 阶段一：预训练（Pre-training）
   "遮住下一个词，猜！"
        │
        ▼ 阶段二：SFT（Supervised Fine-Tuning）
   人工标注的 (问题, 理想回答) 对
        │
        ▼ 阶段三：RLHF（Reinforcement Learning from Human Feedback）
   人类对多个回答排序/点赞，训练奖励模型
        │
        ▼
   可部署的 Chat 模型
```

| 阶段 | 数据 | 学什么 | 类比 |
|------|------|--------|------|
| **预训练** | 网页、书籍、代码库 | 语法、事实、推理模式 | 婴儿听遍天下对话，学会语言 |
| **SFT** | 人工写的 QA 对、对话 | 按指令回答、遵循格式 | 上岗培训：标准话术 |
| **RLHF** | 人类偏好标注 | 有用、无害、诚实 | 客户满意度考核 |

预训练任务：**Next Token Prediction**——遮住下一个词让模型猜，猜错就反向传播。SFT 把任务收窄到"看到问题，生成助手式回答"。RLHF 训练 **Reward Model** 给回答打分，强化学习让策略偏向高分。

> **收：** 三个阶段各干一件事——预训练给知识，SFT 给技能，RLHF 给品味。没有魔法，是规模 + 数据 + 算力。

---

## 4. AI 幻觉

**一句话：** 幻觉不是 bug，是自回归接龙的必然副产品——模型在"做梦"，不是在"查资料"。

### 为什么会编造

```
用户: "你们公司的客服邮箱是？"
              │
              ▼
模型内部: （没有数据库，没有联网，只有权重里的统计规律）
          "客服邮箱" 后面接 "@company.com" 的概率很高
              │
              ▼
输出: "请联系 support@example.com"   ← 看起来合理，纯属编造
```

| 幻觉类型 | 表现 | 根因 |
|----------|------|------|
| 事实性 | 错误日期、虚构论文、假网址 | 训练数据有噪声；接龙优先"流畅"而非"正确" |
| 归因性 | "根据 XX 报告…"（报告不存在） | 学到了"引用格式"，没学到"必须真有这篇" |
| 自信性 | 斩钉截铁地胡说 | RLHF 奖励"有帮助"，惩罚"我不知道" |

模型没有数据库、搜索引擎（除非接 Tool）、也没有自我校验机制。

### 解决方案：RAG

**Retrieval-Augmented Generation** — 先搜，再答。

```
用户提问
    │
    ▼
┌─────────┐     检索相关文档片段
│ 向量库   │ ──────────────────▶ Top-K 段落
└─────────┘
    │
    ▼ 拼进 Prompt
"参考以下资料回答：[文档1]...[文档2]... 用户问题：..."
    │
    ▼
  LLM 接龙（有据可依，幻觉率大幅下降）
```

> **收：** 别把 LLM 当数据库。它是概率文本生成器，RAG / Tool Calling / 人工审核才是工程上的安全网。

---

## 5. 多模态 = 万物皆 Token

**一句话：** 图片、声音、视频都可以被编码成 Token 序列，丢进同一个接龙模型里统一处理。

黄仁勋那句话的工程含义：**所有模态进模型前都变成整数序列，出模型后再由 Decoder 还原。**

```
┌────────┐    Vision Encoder     ┌─────────────────────────────┐
│  图片   │ ──────────────────▶  │ [IMG_42, IMG_17, IMG_89, …] │  ← 图片 Token
└────────┘    （压缩成 N 个）    └─────────────────────────────┘
                                              │
┌────────┐                                    ▼
│  文字   │ ── Tokenize ──▶ [101, 2345, ...] ─┐
└────────┘                                    │
                                              ▼
                                    ┌───────────────┐
                                    │   统一 LLM     │  文字接龙，跨模态理解
                                    │  （接龙核心不变）│
                                    └───────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
              文字 Token 输出           图片 Token 输出            音频 Token 输出
                    │                         │                         │
                    ▼                         ▼                         ▼
              直接可读                   Image Decoder              Audio Decoder
                                         生成图片                   生成声音
```

GPT-4o 看图说话、Sora 文生视频，底层一样：**Encoder 压成 Token → LLM 接龙 → Decoder 还原。**

> **收：** 多模态不是换了新大脑，是给接龙机装了不同感官的翻译器。

---

## 6. Context Engineering = 喂对输入

**一句话：** 模型关在暗室里只会接龙——你不知道的，它也不知道；Context 的质量和长度直接决定输出质量。

### 暗室隐喻

```
┌─────────────────────────────────────┐
│              暗室（LLM）              │
│                                     │
│   唯一信息来源 = 你塞进来的 Context    │
│   没有日历、没有联网、没有你的业务库    │
│                                     │
│   你: "今天天气怎么样？"               │
│   它: （不知道你在哪、不知道是哪天）     │
│       → 只能靠训练数据里的"平均天气"瞎编  │
└─────────────────────────────────────┘
```

**Context Engineering** = 调用前把日期、业务规则、RAG 文档、工具结果、对话历史组装进 Prompt。

### Context 越长 ≠ 越好

| 现象 | 含义 | 工程应对 |
|------|------|----------|
| **Lost in the Middle** | 超长 Context 中间段落被模型"忽略" | 关键信息放开头或结尾 |
| **Context Rot** | Token 数接近上限时，整体推理质量下降 | 控制窗口用量，及时摘要 |
| **成本线性增长** | 输入 Token 按量计费 | 能压缩就压缩 |

### 三个套路

```
┌──────────────────────────────────────────────────────────┐
│                  Context Engineering 三板斧               │
├──────────────┬───────────────────────────────────────────┤
│  选择 (RAG)   │  不塞全部，只检索与问题相关的片段           │
├──────────────┼───────────────────────────────────────────┤
│  压缩 (摘要)  │  历史对话太长 → 先让模型摘要，再带着摘要用   │
├──────────────┼───────────────────────────────────────────┤
│  Multi-Agent │  复杂任务拆分，每个 Agent 拿精简 Context    │
└──────────────┴───────────────────────────────────────────┘
```

Java 开发者可以这么类比：LLM 是 `@Stateless` 的无状态服务，**每次请求必须把完整上下文传进去**，没有 Session 帮你记着。

> **收：** Prompt 不是玄学，是接口设计。Garbage In, Garbage Out——在 LLM 时代依然成立。

---

## 7. 一图总结

从用户输入到最终回答的完整链路：

```
用户输入 "帮我查台湾最高的山"
        │
        ▼
┌─────────────────┐
│  Context 组装     │  System Prompt + 历史 + RAG 片段 + 工具结果
│  (工程层)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Chat Template   │  messages[] → 带特殊标记的长文本
│  (格式层)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Tokenizer       │  文本 → [101, 2345, 8901, ...]
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM 接龙循环     │  概率分布 → 采样 → 拼接 → 再算 → ...
│  (核心)          │  （预训练+SFT+RLHF 训练出的权重）
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Token → 文本    │  Detokenize
└────────┬────────┘
         │
         ▼
    "玉山，海拔 3952 米"
         │
         ▼
    流式推给用户
```

**五个关键词回顾：**

| 关键词 | 本质 |
|--------|------|
| Token 接龙 | 一切输出的根基 |
| Chat Template | 让接龙看起来像对话 |
| 三阶段训练 | 知识 → 技能 → 品味 |
| 幻觉 | 流畅 ≠ 正确，工程上靠 RAG 兜底 |
| Context Engineering | 暗室模型，喂什么得什么 |

---

## 8. 动手验证：Colab 实操

不信？自己跑一遍就彻底明白了。用 Google Colab + HuggingFace Transformers，5 分钟验证上面所有概念。

### 环境准备

```python
# 安装 + 登录 HuggingFace
!pip install transformers -q
from huggingface_hub import login
login(token="你的HF_TOKEN")
```

### 下载模型

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

model_id = "meta-llama/Llama-3.2-3B-Instruct"  # 或 "google/gemma-3-4b-it"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")
```

### 看 Vocabulary

```python
print(tokenizer.vocab_size)  # 128000 个 Token

# 编号 → 文字
tokenizer.decode([0])       # "!"
tokenizer.decode([6151])    # "hi"

# 文字 → 编号
tokenizer.encode("大家好", add_special_tokens=False)  # [109429, 53901]
```

### 手动接龙（核心！）

```python
import torch

prompt = "1+1="
ids = tokenizer.encode(prompt, return_tensors="pt").to(model.device)
output = model(ids)

# 取最后一个位置的 logits → 转概率
logits = output.logits[0, -1, :]
probs = torch.softmax(logits, dim=0)

# 概率最高的 5 个 Token
top5 = torch.topk(probs, 5)
for prob, idx in zip(top5.values, top5.indices):
    print(f"{tokenizer.decode([idx]):>10} → {prob:.2%}")
# 输出：  2 → 65.7%,  3 → 12.1%, ...
```

### Top-K 采样 vs 贪心 vs Temperature

```python
# 贪心（Temperature=0，每次选概率最高）→ 输出固定
output = model.generate(ids, max_new_tokens=20, do_sample=False)

# Top-K 采样（前3名掷骰子）→ 每次不同
output = model.generate(ids, max_new_tokens=20, do_sample=True, top_k=3)

# Temperature 调高 → 更有创意/更跑偏
output = model.generate(ids, max_new_tokens=20, do_sample=True, temperature=1.5, top_k=50)

# Temperature 调低 → 接近贪心但不完全确定
output = model.generate(ids, max_new_tokens=20, do_sample=True, temperature=0.2, top_k=50)

print(tokenizer.decode(output[0]))
```

> **动手试：** 同一个 prompt 跑 3 次，`temperature=0.2` 几乎一样，`temperature=1.5` 每次都不同。这就是为什么 API 有时给你"稳定回答"有时"天马行空"。

### 加 Chat Template 让模型正常回答

```python
messages = [
    {"role": "system", "content": "你的名字是 Llama，用中文回答"},
    {"role": "user", "content": "你是谁？"},
]

# apply_chat_template 帮你加模板 + encode
ids = tokenizer.apply_chat_template(messages, return_tensors="pt").to(model.device)
output = model.generate(ids, max_new_tokens=50)
print(tokenizer.decode(output[0], skip_special_tokens=True))
# → "我是 Llama，一个大型语言模型..."
```

### 一行代码搞定（Pipeline）

```python
from transformers import pipeline

pipe = pipeline("text-generation", model=model_id, device_map="auto")
result = pipe(messages, max_new_tokens=100)
print(result[0]["generated_text"][-1]["content"])
```

**验证清单：**
- [ ] Token ≠ 字：同一个 "GOOD"，句首和句中编号不同
- [ ] 概率分布：改 Prompt 前缀（"在二进制中"），2 的概率骤降
- [ ] Chat Template：不加模板 → 模型乱接；加了 → 正常对话
- [ ] 幻觉：问它不知道的事，它还是会编（只是接龙概率高的词）

> Colab 原始链接：[课程范例代码](https://colab.research.google.com/drive/1EjiX46muxSMy0avtHPXiulVUhmu37Kyi)

---

## 参考

- 李宏毅 2025 生成式 AI 导论（一堂课版）：[YouTube](https://www.youtube.com/watch?v=TigfpYPJk1s)
