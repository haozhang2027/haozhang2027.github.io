---
title: "GRPO 算法原理"
date: 2026-05-05
draft: false
tags:
  - llm
  - rlhf
  - 强化学习
categories:
  - ai
---

## GRPO（Group Relative Policy Optimization）

DeepSeek 在 2024 年提出，主要用于训练 DeepSeek-Math 和 DeepSeek-R1。

---

## 核心思想：砍掉 Critic 网络

PPO 训练需要两个模型同时跑：

```
PPO = 策略网络（Actor）+ 价值网络（Critic）
```

Critic 和 Actor 一样大。训练一个 70B 模型，Critic 也是 70B，显存翻倍。GRPO 的做法：**不要 Critic，用同一组回答的相对排名来估计谁好谁差**。

---

## 工作流程

对一个 prompt（比如 `"2 + 3 = ?"`），用当前策略采样 **G 个回答**（比如 G=4）：

```
y₁: "5"              → reward = 1.0
y₂: "The answer is 5" → reward = 0.9
y₃: "I think 6"       → reward = 0.1
y₄: "2+3 equals 5"    → reward = 0.8
```

### Advantage 计算（组内归一化）

```
mean = (1.0 + 0.9 + 0.1 + 0.8) / 4 = 0.7
std  = 0.37

A₁ = (1.0 - 0.7) / 0.37 = +0.81   ← 好回答，加分
A₂ = (0.9 - 0.7) / 0.37 = +0.54   ← 好回答
A₃ = (0.1 - 0.7) / 0.37 = -1.62   ← 差回答，减分
A₄ = (0.8 - 0.7) / 0.37 = +0.27   ← 还行
```

**本质**：不用绝对分数，用"你在组里排第几"来判断好坏。

### 策略更新

```
ρ_i = π_θ(y_i|x) / π_old(y_i|x)   ← 整条回答的概率比

Loss = -E[ min(ρ_i · A_i, clip(ρ_i, 1-ε, 1+ε) · A_i) - β·KL ]
```

- `ρ_i`：新策略和旧策略在整条回答上的概率比值
- `clip`：限制更新幅度，防止一步迈太大
- `β·KL`：拉回参考模型，防止跑偏

训练目标：**让好回答（A>0）概率变大，差回答（A<0）概率变小**。

---

## 对比 PPO

| | PPO | GRPO |
|---|-----|------|
| 需要 Critic | 是（显存 ×2） | 否（省一半显存） |
| Advantage 来源 | Critic 网络估计 | 组内 reward 相对排名 |
| 训练复杂度 | 高 | 低 |
| 代表作 | ChatGPT / GPT-4 | DeepSeek-R1 |

## 一句话

GRPO 把昂贵的 Critic 网络换成"多采样几个回答，组内比大小"，用极低的代价实现了 PPO 的效果。这也是 DeepSeek-R1 能以较低成本做推理能力训练的关键原因之一。
