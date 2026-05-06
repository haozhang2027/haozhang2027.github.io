---
title: "DAPO 算法原理详解"
date: 2026-05-07
draft: false
math: true
tags:
  - ai
  - 强化学习
  - LLM
  - RLHF
categories:
  - tech
---

## 一句话定义

DAPO (Decoupled Clip and Dynamic sAmpling Policy Optimization) = GRPO + 解耦裁剪上限 + 动态采样过滤 + Token级策略梯度 + 超长惩罚整形。在 Qwen2.5-32B 上用 50% 训练步数超越 DeepSeek-R1-Zero-Qwen-32B（AIME 2024: 50 分 vs 47 分）。

由 ByteDance Seed 和清华 AIR 联合提出，代码、数据、模型权重全部开源。

---

## 1. 背景：从 PPO 到 GRPO 到 DAPO

```
PPO → GRPO（组内相对 advantage，去掉 critic network）
       → DAPO（Clip-Higher + Dynamic Sampling + Token-Level PG + Overlong Penalty）
```

GRPO 是 PPO 的简化变体——用组内均值和标准差估计 baseline，省去 value network。但它仍有三个问题：

1. **Entropy collapse**：对称 clip 导致策略过早收敛，失去探索能力
2. **梯度浪费**：全对或全错的 prompt 组不贡献有效梯度
3. **长 CoT 的 credit assignment**：序列级 loss 无法区分关键 token

DAPO 的四项技术分别解决这些问题。

---

## 2. 技术一：Clip-Higher（解耦裁剪上限）

### 2.1 问题：对称 Clip 导致熵塌缩

标准 PPO 的 importance sampling clipping 是对称的：

$$
\min\left(\rho A,\ \text{clip}(\rho, 1-\varepsilon, 1+\varepsilon)A\right)
$$

其中 $\rho = \frac{\pi_\theta}{\pi_{\theta_{old}}}$ 是重要性采样比。

对称 clip 同时限制上界和下界。当 $A \gt 0$ 时，$\rho$ 被限制在 $1+\varepsilon$，策略无法大幅提升好 token 的概率。久而久之，策略分布变得尖锐（低熵），丧失探索能力——这就是 **entropy collapse**。

### 2.2 解耦

DAPO 将上下界解耦，使用不对称的 clip 范围（$\varepsilon_{\text{high}} \gg \varepsilon_{\text{low}}$）：

$$
\min\left(\rho A,\ \text{clip}(\rho, 1-\varepsilon_{\text{low}}, 1+\varepsilon_{\text{high}})A\right)
$$

- **上界 $\varepsilon_{\text{high}}$**：较大（如 0.28），允许好 token 的概率大幅提升 → 保留探索
- **下界 $\varepsilon_{\text{low}}$**：较小（如 0.2），严格限制坏 token 的概率下降 → 防止塌缩

**直觉**：当 advantage 为正时，模型可以大胆提升概率；当 advantage 为负时，仍然保守限制下降幅度。策略保持多样性。

### 2.3 实验证据

DAPO 论文记录了训练过程中的熵变化：

- 标准 PPO/GRPO：熵持续下降 → 策略过早确定
- DAPO Clip-Higher：熵在初期下降后稳定甚至回升 → 健康的 exploration-exploitation 平衡

---

## 3. 技术二：Dynamic Sampling（动态采样过滤）

### 3.1 问题：零梯度样本

GRPO 对每个 prompt 采样 $G$ 条 response，组内计算 advantage：

$$
\hat{A}_i = \frac{r_i - \text{mean}(\{r_1, \ldots, r_G\})}{\text{std}(\{r_1, \ldots, r_G\})}
$$

当组内所有 response 奖励相同时：
- **acc=1**（全对）：均值=1，标准差=0，$\hat{A}_i = 0/0$
- **acc=0**（全错）：均值=0，标准差=0，$\hat{A}_i = 0/0$

两种情况都产生零梯度，但这组 prompt 仍然消耗了 GPU 显存和计算。

### 3.2 过滤策略

DAPO 在每个 batch 中：

1. 计算各组 accuracy
2. 丢弃 acc=1 和 acc=0 的组
3. 重新采样新 prompt 替代，保持有效梯度 prompt 数量恒定

**训练后期**：模型变强后越来越多 prompt 变成 acc=1，此时自动增大采样量来维持足够的有效样本。

### 3.3 效果

- 每个 batch 都保证产生有意义的梯度信号
- 训练中期不浪费计算在"已经会了"或"完全不会"的题上
- 训练后期自动 focus 在模型尚不确定的边界 prompt

---

## 4. 技术三：Token-level Policy Gradient Loss

### 4.1 问题：序列级 Loss 的局限

GRPO 在序列级别计算 loss——同一 response 内所有 token 共享一个 advantage 值：

$$
\mathcal{L}_{\text{GRPO}} = \frac{1}{G} \sum_{i=1}^{G} \frac{1}{|o_i|} \sum_{t=1}^{|o_i|} \min\left(\rho_{i,t} A_i,\ \text{clip}(\rho_{i,t}, 1-\varepsilon, 1+\varepsilon)A_i\right)
$$

注意：一个 response 中所有 token 的 $A_i$ 相同。这在短 response 场景尚可接受，但在长 CoT 推理中，**不同 token 对最终答案的贡献差别巨大**。

举例：一个 2000 token 的数学推理 chain，前 1800 步推导完全正确，最后 200 步在错误方向计算得出错误答案。序列级 loss 将整个 response 标记为"坏"，惩罚了本应保留的前 1800 步。

### 4.2 Token 级 DAPO Loss

DAPO 将 loss 公式化为 token 级别：

$$
\mathcal{L}_{\text{DAPO}}(\theta) = \frac{1}{\sum_{i} |o_i|} \sum_{i} \sum_{t} \min\left(\rho_{i,t} \hat{A}_i,\ \text{clip}(\rho_{i,t}, 1-\varepsilon_{\text{low}}, 1+\varepsilon_{\text{high}})\hat{A}_i\right)
$$

其中：

$$
\rho_{i,t} = \frac{\pi_\theta(o_{i,t} \mid q, o_{i,\lt t})}{\pi_{\theta_{old}}(o_{i,t} \mid q, o_{i,\lt t})}
$$

虽然 advantage $\hat{A}_i$ 仍共享（因为奖励是 response-level 的），但 **梯度通过 token 级 importance weight $\rho_{i,t}$ 自适应调节**：
- 当前策略与旧策略一致的 token → $\rho_{i,t} \approx 1$ → 正常梯度
- 当前策略大幅偏离旧策略的 token → clip 生效 → 梯度限幅
- 每个 token 独立计算 clip，精确控制更新幅度

---

## 5. 技术四：Overlong Reward Shaping（超长惩罚）

### 5.1 问题

长 CoT 训练中，模型可能产生极长 response。这些超长 response：
- 通常逻辑混乱、质量低
- 但奖励信号可能偶然为正 → 引入噪声
- 消耗巨量显存和计算

### 5.2 惩罚项

DAPO 对超长 response 施加线性惩罚：

$$
R_{\text{shaped}} = R_{\text{original}} - \lambda \cdot \max(0, L - L_{\text{max}})
$$

其中：
- $L$：response 的 token 数
- $L_{\text{max}}$：长度阈值
- $\lambda$：惩罚系数

**效果**：模型学会在合理长度内给出答案，同时仍允许必要的长推理（不超过阈值不惩罚）。论文观察到训练稳定性显著提升，生成长度的增长更加平滑可控。

---

## 6. 四项技术总结

| 技术 | 解决问题 | 核心思路 |
|---|---|---|
| Clip-Higher | Entropy collapse | 不对称 clip：高上界 + 低下界 |
| Dynamic Sampling | 梯度浪费 | 过滤 acc=0/1 组，保持有效梯度恒定 |
| Token-Level PG Loss | 长 CoT credit assignment | 每个 token 独立计算 importance weight |
| Overlong Reward Shaping | 超长 response 噪声 | 超出阈值的线性长度惩罚 |

---

## 7. 训练效果

基于 Qwen2.5-32B base model，128 张 H20 GPU：

- **AIME 2024: 50 分** → 超越 DeepSeek-R1-Zero-Qwen-32B（47 分）
- **训练步数减少 50%** → 更高效
- **熵稳定**：训练全程保持健康的探索水平
- **长度稳定增长**：response 长度平滑上升，无突变

---

## 8. 参考

- **DAPO Paper (2025)**: Yu et al., *DAPO: An Open-Source LLM Reinforcement Learning System at Scale* — [arXiv:2503.14476](https://arxiv.org/abs/2503.14476)
- **DAPO GitHub**: [BytedTsinghua-SIA/DAPO](https://github.com/BytedTsinghua-SIA/DAPO)
- **GRPO (2024)**: Shao et al., *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models*
- **DeepSeek-R1 (2025)**: Guo et al., *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning*
