---
title: "LLM 训练范式：适用场景、优劣与核心算法原理"
date: 2026-05-07
draft: false
math: true
tags:
  - ai
  - 强化学习
  - LLM
  - RLHF
  - 训练
categories:
  - tech
---

## 总览

```
Pretraining (预训练)
    ↓
SFT (监督微调)
    ↓
Alignment (对齐)
    ├── RLHF (PPO)
    ├── 纯偏好对齐 (DPO / IPO / KTO / SimPO)
    ├── 纯 RL (GRPO / DAPO / GSPO / R1-Zero)
    └── RLAIF (AI 反馈替代人工)
    ↓
Inference-Time Scaling (推理时增强，不更新权重)
```

---

## 1. Pretraining（预训练）

### 适用场景
一切 LLM 训练的起点。让模型学会语言的统计规律、知识和基本推理能力。

### 核心原理
**Next-Token Prediction (NTP)**：给定上文 $x_{<t}$，最大化下一个 token $x_t$ 的概率。

$$\mathcal{L}_{\text{LM}} = -\frac{1}{T}\sum_{t=1}^{T} \log P_\theta(x_t \mid x_{<t})$$

实际训练中，一次 forward 对整个序列所有位置同时计算 loss，因为 causal attention mask 保证每个位置只能看到上文。

### 优势
- 海量无标注文本可用（数十 T tokens），scalable
- 学到的 representation 是后续一切能力的基础

### 劣势
- 算力成本极高（千卡 GPU × 数月）
- Base model 只能续写，不能对话
- 数据质量决定上限（Garbage in, garbage out）

### 常用改进
- **Fill-in-the-Middle (FIM)**：给定前后文预测中间，代码模型标配
- **MoE 稀疏训练**：降低训练/推理成本（Mixtral, DeepSeek-V2/V3, Qwen3-MoE）
- **Long-context extension**：RoPE 插值、YaRN 等

---

## 2. SFT（Supervised Fine-Tuning，监督微调）

### 适用场景
让 base model 学会"对话格式"和"指令跟随"——问什么答什么。

### 核心原理
和 Pretraining 完全相同的 NTP loss，但数据从"任意文本"变成"(instruction, response) 对"。

$$\mathcal{L}_{\text{SFT}} = -\frac{1}{|y|}\sum_{t=1}^{|y|} \log P_\theta(y_t \mid x, y_{<t})$$

其中 $x$ 是指令，$y$ 是标准回答。**只在 $y$ 部分计算 loss**，对 $x$ 部分不计算（mask 掉）。

### 优势
- 简单直接，和预训练同一套代码
- 数据量要求不高（几千到几万条高质量数据即可见效）
- 格式遵循效果好（输出 JSON、代码块等）

### 劣势
- 答案质量上限受限于标注数据（弱模型标注 → 弱模型能力天花板）
- 纯 SFT 无法处理偏好 / 价值观
- 分布外泛化差——遇到没见过的指令风格容易崩

### 关键细节
- **Noise injection**：SFT 数据中加入少量多样化 prompt 能提升泛化
- **Data mix**：不同能力（推理、翻译、代码）的数据需要配比，否则 catastrophic forgetting

---

## 3. RLHF（Reinforcement Learning from Human Feedback）

### 适用场景
让模型产出"人觉得好"的回答——风格、安全、helpfulness。

### 三步流程

```
Step 1: SFT（打底）
Step 2: 训 Reward Model (RM)
Step 3: PPO 用 RM 打分优化 policy
```

### Step 2 — Reward Model

人工对同一 prompt 的多条回答排序（A > B > C > D），用 Bradley-Terry 模型训练 RM 学会打分：

$$\mathcal{L}_{\text{RM}} = -\log \sigma(r_\phi(x, y_w) - r_\phi(x, y_l))$$

其中 $y_w$ 是更好（win）的回答，$y_l$ 是更差（lose）的回答。RM 实际上学的是"相对偏好"，不是绝对分数。

### Step 3 — PPO（Proximal Policy Optimization）

**核心思想**：最大化 reward，但不能离 SFT 模型太远（防 reward hacking 和 catastrophic forgetting）。

目标函数：

$$J_{\text{PPO}}(\theta) = \mathbb{E}\left[ \min\left(r_t(\theta) \cdot A_t,\ \text{clip}(r_t(\theta), 1-\varepsilon, 1+\varepsilon) \cdot A_t\right) - \beta \cdot \text{KL}(\pi_\theta \| \pi_{\text{ref}}) \right]$$

其中：
- $r_t(\theta) = \frac{\pi_\theta(a_t|s_t)}{\pi_{\text{old}}(a_t|s_t)}$：token 级重要性采样比率
- $A_t$：Advantage，由 Critic (Value Model) + GAE 估计
- $\varepsilon$：clip 范围，典型值 0.2
- $\text{KL}$ 惩罚：防止 policy 偏离 SFT 参考模型太远

**Critic (Value Model)** 的作用：
- 学习 $V(s_t)$，预测当前状态（prompt + 已生成 token）的期望未来 reward
- 用来算 $A_t = R_t - V(s_t)$（实际 reward 减去期望），去掉 baseline 降低方差
- Critic 是另一套模型参数，通常和 policy 共享主干 + 独立 head

### 优势
- 训练稳定（clip 机制防止大更新）
- Critic 提供 token 级精确信用分配
- 经过广泛验证（ChatGPT、Claude 早期都用这套）

### 劣势
- 需要单独训一个 Critic，显存翻倍
- 需要先训 Reward Model，流程长
- Reward Model 也是学出来的，有 bias 和 reward hacking 风险
- 人工标注偏好数据昂贵

---

## 4. DPO（Direct Preference Optimization）

### 适用场景
有偏好数据但没有 reward model，需要做对齐（安全、风格、helpfulness）。

### 核心原理
**去掉显式 RM**：从 RLHF 的 Bradley-Terry 模型出发，推导出"直接优化 policy 就能隐式学到 reward"的等价形式。

RLHF 的优化目标 + Bradley-Terry 的 optimal policy 有闭式解：

$$\pi^*(y|x) \propto \pi_{\text{ref}}(y|x) \cdot \exp\left(\frac{1}{\beta} r(x, y)\right)$$

反解出 reward：

$$r(x, y) = \beta \cdot \log\frac{\pi^*(y|x)}{\pi_{\text{ref}}(y|x)} + \beta \log Z(x)$$

把 $r$ 代入 Bradley-Terry，$Z(x)$ 消掉，得到 **DPO loss**：

$$\mathcal{L}_{\text{DPO}} = -\log \sigma\left(\beta \cdot \log\frac{\pi_\theta(y_w|x)}{\pi_{\text{ref}}(y_w|x)} - \beta \cdot \log\frac{\pi_\theta(y_l|x)}{\pi_{\text{ref}}(y_l|x)}\right)$$

**直觉**：增大 chosen 回答的相对概率，压低 rejected 回答的相对概率。$\beta$ 控制更新强度（越小越保守）。

### 优势
- 不需要训 Reward Model，流程简化
- 不需要 Critic，显存开销小
- 偏好数据够好即可训练

### 劣势
- 偏好标注质量决定效果（标注不一致 → 信号弱）
- 对 prompt 分布敏感，OOD prompt 表现不如 RLHF
- Preference 和 generation 的分布不匹配问题（policy 在训，偏好数据是静态的）
- 没有探索机制，只学"哪个更好"，不会自己发现新策略

### 变体
| 变体 | 改进 |
|------|------|
| **IPO** | 加 L2 正则防止 $\log\pi$ 比值过拟合 |
| **KTO** | 不需要 pairwise，单条偏好即可（更灵活） |
| **SimPO** | 用序列长度做隐式长度惩罚 + 去掉 reference model |

---

## 5. GRPO（Group Relative Policy Optimization）

### 适用场景
有可自动验证 reward 的任务——数学、代码、推理。不需要 reward model，只需 reward function（规则）。

### 核心原理
**去 Critic + 组内相对 advantage**：对同一 prompt 采样 $G$ 条回答，组内归一化算 advantage。

$$A_i = \frac{r_i - \text{mean}(r_1, ..., r_G)}{\text{std}(r_1, ..., r_G)}$$

其中 $r_i$ 是第 $i$ 条回答的 reward（规则打分）。然后用类似 PPO 的 clipped 损失更新策略，额外加 KL 惩罚：

$$J_{\text{GRPO}} = \mathbb{E}\left[ \min\left(r_t \cdot A_i,\ \text{clip}(r_t, 1-\varepsilon, 1+\varepsilon) \cdot A_i\right) - \beta \cdot \text{KL}(\pi_\theta \| \pi_{\text{ref}}) \right]$$

**关键区别 vs PPO**：
- PPO：Critic 学 $V(s)$ 预测期望 reward → 逐 token 算 advantage 的精细估计
- GRPO：同一组内多条回答的直接 reward 比较 → 组内归一化 = 无参 advantage

### 优势
- **省显存**：不需要 Critic，比 PPO 省 ~40-50%
- 不需要 reward model，只需规则打分（数学对错、测试通过率）
- 简单、收敛快，DeepSeek-R1 验证了有效性

### 劣势
- **高方差**：组内只有 $G$ 条回答（通常 4-8），均值估计粗糙。方差沿 token 序列累积
- **采样多样性依赖**：如果一组内所有回答 reward 相同（全对或全错），advantage 为 0，梯度消失
- **长序列不稳定**：Qwen3 团队（2025）发现 per-token importance sampling 在长 CoT + MoE 中方差爆炸，可能崩溃
- **长度偏差 (reward hack)**：模型学会写更长 → 覆盖更多"可能碰对"的 token → reward 上升
- **KL 系数敏感**：$\beta$ 需要精细调（建议 0.0075-0.01）

### 关键超参
| 参数 | 建议范围 | 说明 |
|------|---------|------|
| Group size $G$ | 4-8 | 越大 advantage 越准，显存也越大 |
| KL penalty $\beta$ | 0.0075-0.01 | 太高学不动，太低策略漂移 |
| Clip $\varepsilon$ | 0.2 | 同 PPO 标准值 |

---

## 6. DAPO（Decoupled Clip and Dynamic Sampling Policy Optimization）

### 适用场景
GRPO 的升级版——长 CoT 推理任务、数学证明、代码生成。

### 核心原理
在 GRPO 基础上做了四项改进：

**1. Token-Level Loss**
GRPO 按序列平均 loss（先每个序列内部平均，再序列之间平均）。DAPO 改为所有 token 直接平均：

```
GRPO:  loss = avg( avg(tokens_in_seq1), avg(tokens_in_seq2) )
DAPO:  loss = avg( all_tokens_in_batch )
```

长序列中每个 token 被平等对待，错误 token 无处稀释。

**2. Decoupled Clip（非对称裁剪）**
PPO/GRPO 的 clip 是对称的（$\pm\varepsilon$）。DAPO 拆开：
- $\varepsilon_{\text{low}}$（下界）：0.2
- $\varepsilon_{\text{high}}$（上界）：0.28

上界更大 → 好的更新不被过度截断 → 鼓励探索多样性 → 防止 entropy collapse。

**3. Dynamic Sampling（动态采样）**
过滤掉"全对"或"全错"的 prompt 组（reward 方差 = 0），只保留有梯度信号的组。不足的 batch 重新采样填补。

**4. Overlong Reward Shaping**
对超长回答施加软惩罚（线性递增的负 reward），无需硬截断。

### 优势
- 长推理链学得更好（token-level loss + 非对称 clip）
- 训练效率高（动态采样排除无信号样本）
- ByteDance/清华验证 AIME 2024 达 52%（DeepSeek-R1-Zero-Qwen-32B 同期基准更高），且训练步数少 ~50%

### 劣势
- 工程复杂度比 GRPO 高
- Dynamic sampling 在实际大规模训练中收益存疑（有论文认为应关掉）
- 仍然依赖 reward function 质量

---

## 7. GSPO（Group Sampling Policy Optimization）

### 适用场景
Qwen3 提出的 GRPO 替代方案，解决 token 级方差累积问题，尤其适合 MoE 架构。

### 核心改进
GRPO 在 token 级做 importance sampling（每个 token 单独算 $r_t = \pi_\theta(a_t)/\pi_{\text{old}}(a_t)$），长序列中 $r_t$ 乘积 → 方差爆炸。GSPO 把 importance sampling 提升到**序列级**：

$$r_{\text{seq}} = \frac{\pi_\theta(y|x)}{\pi_{\text{old}}(y|x)} = \prod_t \frac{\pi_\theta(y_t|x, y_{<t})}{\pi_{\text{old}}(y_t|x, y_{<t})}$$

整个序列一个比值 → 无 token 级乘法累积 → 方差可控。

---

## 8. R1-Zero 路线（纯 RL，无需 SFT 打底）

### 核心思想
DeepSeek-R1-Zero 证明：从 base model 出发，不经过任何 SFT，纯用 GRPO + 规则 reward 就能涌现推理能力。

### 训练流程
```
Base Model → GRPO (reward = 答案正确 + 格式遵循) → R1-Zero
```

- 不需要 SFT 数据
- 不需要 reward model
- 仅靠规则 reward（数学对错、代码通过率）

### 优势
- 数据成本几乎为零
- "冷启动"探索，模型自己发现推理策略

### 劣势
- 输出可读性差（格式混乱、语言混杂）
- 推理链质量可能需要后续 SFT 兜底（DeepSeek-R1 的做法：R1-Zero → SFT → RL → 最终模型）

---

## 9. Rejection Sampling Fine-Tuning

### 适用场景
有强 verifier 但没有人工标注，想低成本获得"正确回答"的 SFT 数据。

### 核心流程
```
Base/SFT Model → 对每个 prompt 采样 N 条回答 → Verifier 筛选正确的 → 用正确回答做下一轮 SFT
```

### 代表工作
- **ReST (Reinforced Self-Training)**：Google DeepMind，采样 → 筛选 → SFT → 循环
- **STaR (Self-Taught Reasoner)**：模型自己生成 + 筛选推理链做 SFT
- **ReST^EM**：用期望最大化解释采样 + 筛选 + SFT 流程

### 优势
- 不需要人工标注
- 可以迭代多轮，持续提升

### 劣势
- 需要可靠的 verifier（规则 or RM），否则错误回答混入训练
- 采样成本随 N 线性增长
- 可能 diversity collapse（多轮后回答模式收窄）

---

## 10. Inference-Time Scaling（推理时增强，不更新权重）

### 核心理念
不改变模型参数，通过推理时的计算策略提升任务表现。

| 方法 | 原理 | 成本 |
|------|------|------|
| **Chain-of-Thought (CoT)** | Prompt 引导逐步推理 | 推理 token 增加 |
| **Self-Consistency** | 采样多条 CoT，多数投票 | N 倍采样成本 |
| **Best-of-N** | 采样 N 条，verifier 选最优 | N 倍采样 + verifier |
| **Tree-of-Thought** | BFS/DFS 搜索推理树 | 远大于 N 倍 |
| **Speculative Decoding** | 小模型草稿 + 大模型验证 | 更低延迟 |
| **Test-Time Compute Scaling** | 动态分配算力，难题多算 | 按难度自适应 |

### 优势
- 不修改模型权重，安全
- 和训练方案正交（可叠加）
- Speculative decoding 是纯工程收益（加速不降质）

### 劣势
- 推理成本增加（1 条 → N 条）
- 搜索空间大但无理论学习信号

---

## 选型速查

| 你有的 | 你想要的 | 选 |
|--------|---------|----|
| 海量无标注文本 | 语言模型 | Pretraining |
| (prompt, 标准答案) 对 | 指令跟随 | SFT |
| (prompt, 好答, 差答) 偏好对 | 对齐风格/安全 | DPO |
| 人工偏好数据 + 充足算力 + 追求稳定 | 对齐 + 推理 | RLHF (PPO) |
| 规则 reward + 算力有限 | 推理能力 | GRPO |
| 规则 reward + 长推理链 | 更强推理 | DAPO |
| 规则 reward + MoE + 长序列 | 推理 + 稳定 | GSPO |
| 规则 reward + 无 SFT 数据 | 从零探索推理 | R1-Zero |
| 强 verifier + 无标注数据 | 自我进化 | Rejection Sampling |
| 训练好的模型 + 想再提点能力 | 推理效果 | Inference-Time Scaling |
