---
title: "GRPO 算法原理详解"
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

GRPO (Group Relative Policy Optimization) = PPO - Critic + 组内标准化奖励 + KL 直接惩罚。通过让同一问题的多个回答互相"竞争"，消除对价值网络的依赖。

---

## 1. 核心动机：为什么不用 Critic？

传统 PPO 需要同时维护两个模型：

- **Actor（策略网络）**：生成动作
- **Critic（价值网络）**：估计状态价值，用于计算 Advantage

训练一个大语言模型时，Critic 通常和 Actor 规模相当，这意味着**显存翻倍、训练复杂度翻倍**。GRPO 的洞见是：对于 LLM 的 RL 微调，我们可以通过对同一个 prompt 采样多条响应，用组内比较来替代 Critic 的基线估计。

---

## 2. 核心公式

### 2.1 问题设定

对每个问题 $q$，从旧策略 $\pi_{\theta_{old}}$ 采样一组 $G$ 条响应：

$$
\{o_1, o_2, \ldots, o_G\} \sim \pi_{\theta_{old}}(q)
$$

每条响应 $o_i$ 获得一个奖励 $r_i$（由奖励模型或规则给出）。

### 2.2 组内相对优势（Group Relative Advantage）

**不训练 Critic**，直接用组内标准化后的奖励作为 Advantage：

$$
\hat{A}_i = \frac{r_i - \text{mean}(\{r_1, \ldots, r_G\})}{\text{std}(\{r_1, \ldots, r_G\})}
$$

这就是 GRPO 名字的由来：在**组内（Group）**计算**相对（Relative）**优势。

### 2.3 GRPO 目标函数

对每个问题 $q$，损失函数为：

$$
\begin{aligned}
\mathcal{J}_{GRPO}(\theta) = \mathbb{E}_{q, \{o_i\}_{i=1}^G \sim \pi_{\theta_{old}}} 
\Bigg[ \frac{1}{G} \sum_{i=1}^{G} \frac{1}{|o_i|} \sum_{t=1}^{|o_i|}
&\min\Big( \rho_t \hat{A}_i,\; \text{clip}(\rho_t, 1-\epsilon, 1+\epsilon) \hat{A}_i \Big) \\
&- \beta \cdot \mathbb{D}_{KL}(\pi_\theta \parallel \pi_{ref}) \Bigg]
\end{aligned}
$$

其中：

- $\rho_t = \frac{\pi_\theta(o_{i,t} \mid q, o_{i,<t})}{\pi_{\theta_{old}}(o_{i,t} \mid q, o_{i,<t})}$ — 概率比（probability ratio）
- $\epsilon$ — clip 范围（通常取 0.2）
- $\beta$ — KL 惩罚系数
- $\pi_{ref}$ — 参考模型（通常是初始 SFT 模型，训练期间冻结）

### 2.4 KL 散度估计

GRPO 使用 Schulman et al. (2020) 的无偏估计器（k3 estimator）：

$$
\mathbb{D}_{KL}(\pi_\theta \parallel \pi_{ref}) = 
\frac{\pi_{ref}(o_{i,t} \mid q, o_{i,<t})}{\pi_\theta(o_{i,t} \mid q, o_{i,<t})} 
- \log\frac{\pi_{ref}(o_{i,t} \mid q, o_{i,<t})}{\pi_\theta(o_{i,t} \mid q, o_{i,<t})} 
- 1
$$

> **为什么用这个形式？** 因为它不依赖于对 $\pi_{ref}$ 的采样，是无偏且低方差的估计。

---

## 3. GRPO vs PPO 对比

| | PPO | GRPO |
|---|---|---|
| Value Network | 需要 Critic | **不需要** |
| Advantage 计算 | GAE（广义优势估计） | **组内标准化奖励** |
| 模型数量 | Actor + Critic（2 个） | **仅 Actor（1 个）** |
| 每条 prompt 采样数 | 1 条响应 | G 条响应（通常 4-64） |
| KL 约束 | 通常用 reward shaping | **直接加入损失函数** |

---

## 4. Python 实现

### 4.1 组内相对优势计算

```python
import torch

def compute_group_relative_advantage(
    rewards: torch.Tensor,  # shape: (batch_size, num_generations)
    eps: float = 1e-8,
) -> torch.Tensor:
    """
    对每组响应，用组内均值和标准差对奖励做标准化，得到 Advantage。

    Args:
        rewards: (B, G) — B 个 prompt，每个 prompt 有 G 条响应
        eps: 防止除零

    Returns:
        advantages: (B, G) — 组内标准化后的 advantage
    """
    mean = rewards.mean(dim=1, keepdim=True)   # (B, 1)
    std = rewards.std(dim=1, keepdim=True)      # (B, 1)
    advantages = (rewards - mean) / (std + eps)
    return advantages
```

### 4.2 KL 散度（k3 无偏估计器）

```python
def compute_kl_divergence(
    log_probs: torch.Tensor,      # log π_θ(o_t | q, o_<t)
    ref_log_probs: torch.Tensor,  # log π_ref(o_t | q, o_<t)
) -> torch.Tensor:
    """
    Schulman et al. (2020) 的无偏 KL 估计器 (k3 estimator)。

    k3 = exp(r) - r - 1,  where r = log(π_ref / π_θ)
    """
    ratio = ref_log_probs - log_probs  # log(π_ref / π_θ)
    kl = torch.exp(ratio) - ratio - 1.0
    return kl
```

### 4.3 GRPO 损失函数

```python
from typing import Tuple

def grpo_loss(
    log_probs: torch.Tensor,           # (B*G, seq_len) 当前策略的 log prob
    old_log_probs: torch.Tensor,       # (B*G, seq_len) 旧策略的 log prob
    ref_log_probs: torch.Tensor,       # (B*G, seq_len) 参考模型的 log prob
    advantages: torch.Tensor,          # (B*G, seq_len) 每个 token 的 advantage
    response_mask: torch.Tensor,       # (B*G, seq_len) 1=response, 0=prompt
    clip_epsilon: float = 0.2,
    kl_beta: float = 0.04,
) -> Tuple[torch.Tensor, dict]:
    """
    GRPO 损失函数。

    对每条响应中的每个 token：
      L = -min(ρ·A, clip(ρ, 1-ε, 1+ε)·A) + β·KL

    where ρ = π_θ / π_θ_old (probability ratio).
    """
    # ---- 概率比 ----
    ratio = torch.exp(log_probs - old_log_probs)  # ρ_t

    # ---- PPO-style clipped objective ----
    advantages_expanded = advantages.unsqueeze(-1).expand_as(log_probs)

    surr1 = ratio * advantages_expanded               # ρ·A
    surr2 = torch.clamp(ratio, 1 - clip_epsilon,
                        1 + clip_epsilon) * advantages_expanded
    policy_loss = -torch.min(surr1, surr2)            # 取 min → 保守更新

    # ---- KL 惩罚 ----
    kl = compute_kl_divergence(log_probs, ref_log_probs)

    # ---- 总损失（仅对 response token） ----
    combined_loss = policy_loss + kl_beta * kl
    masked_loss = (combined_loss * response_mask).sum() / (
        response_mask.sum() + 1e-8
    )

    # 统计信息
    with torch.no_grad():
        stats = {
            "loss": masked_loss.item(),
            "policy_loss": (
                (policy_loss * response_mask).sum().item()
                / (response_mask.sum().item() + 1e-8)
            ),
            "kl": (
                (kl * response_mask).sum().item()
                / (response_mask.sum().item() + 1e-8)
            ),
            "mean_ratio": ratio[response_mask.bool()].mean().item(),
            "clip_fraction": (
                ((ratio - 1).abs() > clip_epsilon)
                .float()[response_mask.bool()]
                .mean()
                .item()
            ),
        }

    return masked_loss, stats
```

### 4.4 完整训练步骤

```python
def grpo_training_step(
    model,                  # 当前策略模型 π_θ
    ref_model,              # 参考模型 π_ref（冻结，不参与梯度更新）
    prompts: list[str],     # B 个 prompt
    tokenizer,
    num_generations: int = 8,  # G：每个 prompt 生成 G 条响应
    reward_func,               # 奖励函数 r(response, prompt)
    optimizer,
    clip_epsilon: float = 0.2,
    kl_beta: float = 0.04,
):
    """
    单步 GRPO 训练的完整流程：

    Step 1: 对每个 prompt，用旧策略 π_θ_old 采样 G 条响应
    Step 2: 计算每条响应的奖励（规则或奖励模型）
    Step 3: 组内标准化 → 得到 relative advantage
    Step 4: 用 GRPO loss 反向传播更新 π_θ
    """
    model.train()
    ref_model.eval()

    B = len(prompts)

    # ---- Step 1 & 2: 采样 + 计算奖励 ----
    all_rewards = []
    all_responses = []

    with torch.no_grad():
        for prompt in prompts:
            group_rewards = []
            group_responses = []

            for _ in range(num_generations):
                input_ids = tokenizer.encode(prompt, return_tensors="pt")
                output = model.generate(
                    input_ids, max_new_tokens=512, do_sample=True
                )
                response_ids = output[:, input_ids.shape[1]:]
                response_text = tokenizer.decode(response_ids[0])

                r = reward_func(response_text, prompt)
                group_rewards.append(r)
                group_responses.append(response_text)

            all_rewards.append(group_rewards)
            all_responses.append(group_responses)

    # ---- Step 3: 组内相对优势 ----
    rewards_tensor = torch.tensor(all_rewards)  # (B, G)
    advantages = compute_group_relative_advantage(rewards_tensor)  # (B, G)

    # ---- Step 4: GRPO 损失 + 反向传播 ----
    optimizer.zero_grad()
    # loss, stats = grpo_loss(
    #     log_probs=current_log_probs,       # 当前策略再前向一次
    #     old_log_probs=old_log_probs,       # 采样时保存的 log prob
    #     ref_log_probs=ref_log_probs,       # 参考模型前向
    #     advantages=advantages.repeat_interleave(num_generations, dim=0),
    #     response_mask=response_mask,
    #     clip_epsilon=clip_epsilon,
    #     kl_beta=kl_beta,
    # )
    # loss.backward()
    # optimizer.step()

    return {"avg_reward": float(rewards_tensor.mean())}
```

### 4.5 奖励函数示例（规则型，适用于数学题）

```python
import re

def rule_based_reward(response: str, ground_truth: str) -> float:
    """
    基于规则的奖励函数。

    适用于数学推理、代码生成等有明确 ground truth 的场景。
    DeepSeek-R1 就使用了类似的结构化奖励。
    """
    def extract_answer(text: str) -> str:
        """从 boxed{} 或最后一行提取答案"""
        match = re.search(r'\\boxed\{([^}]+)\}', text)
        if match:
            return match.group(1).strip()
        return text.strip().split("\n")[-1].strip()

    pred = extract_answer(response)
    truth = extract_answer(ground_truth)

    # 准确性奖励：答案是否正确
    accuracy_reward = 1.0 if pred == truth else 0.0

    # 格式奖励：是否正确使用了  标签
    format_reward = 1.0 if (
        "" in response and "" in response
    ) else 0.0

    return accuracy_reward + 0.1 * format_reward
```

### 4.6 验证 Advantage 计算

```python
# batch_size=2, 每个 prompt 生成 4 条响应
test_rewards = torch.tensor([
    [0.0, 0.5, 1.0, 0.5],   # prompt 1 的 4 条响应奖励
    [0.3, 0.3, 0.3, 0.3],   # prompt 2：全部相同
])
adv = compute_group_relative_advantage(test_rewards)
print("Rewards:\n", test_rewards)
print("Advantages:\n", adv)
# prompt 2 标准差为 0 → 优势全为 0（不能再 normalize）
```

---

## 5. 关键设计要点

1. **G 的选择**：通常取 4-64。G 越大，Advantage 估计越稳定（组统计更可靠），但显存和计算开销线性增长。

2. **KL 约束直接加入损失**：不同于 PPO 在 reward 中减 KL 惩罚，GRPO 将 KL 项直接加入损失函数，梯度更直接，训练更稳定。

3. **Advantage 的 token 级复用**：同一条响应中所有 token 共享同一个 advantage 值（因为奖励是 response-level 的）。每个 token 位置的 advantage 等于其所属响应的组内标准化奖励。

4. **不需要 GAE 和 λ-return**：因为 advantage 是通过组内相对比较得到的，不依赖于时序差分（TD），省去了 GAE 的所有超参数（$\gamma, \lambda$）。

5. **参考模型作为锚点**：$\pi_{ref}$ 通常是初始 SFT 模型，训练期间冻结。它确保策略不会偏离太远，防止 reward hacking。

6. **无偏 KL 估计**：k3 estimator 不需要对 $\pi_{ref}$ 采样，比 Monte Carlo 估计更稳定。

---

## 6. 与 DeepSeek-R1 的关系

GRPO 是 DeepSeek-R1 训练流程中的核心 RL 算法：

```
DeepSeek-V3-Base
    │
    ├── SFT (数千条冷启动数据)
    │
    ├── RL Stage 1: GRPO（推理导向 RL）
    │   ├── 奖励：准确性 + 格式 + 语言一致性
    │   └── 产生：DeepSeek-R1-Zero 风格模型
    │
    ├── Rejection Sampling + SFT（扩充数据）
    │
    └── RL Stage 2: GRPO（全场景 RL）
        └── 产生：DeepSeek-R1
```

DeepSeek-R1 的奖励函数组合：
- **准确性奖励**：数学题答案是否匹配、代码是否通过测试用例
- **格式奖励**：推理过程是否放在 `  ` 标签内
- **语言一致性奖励**：目标语言占比

---

## 7. 参考论文

- **DeepSeekMath (2024)**: Shao et al., *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models* — 首次提出 GRPO
- **DeepSeek-R1 (2025)**: Guo et al., *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning* — GRPO 的大规模验证
- **k3 KL Estimator**: Schulman et al., *Approximating KL Divergence* (2020) — 无偏 KL 估计器
