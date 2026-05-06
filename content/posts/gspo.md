---
title: "GSPO 算法原理详解"
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

GSPO (Group Sequence Policy Optimization) = GRPO + 序列级 importance ratio（几何平均） + 序列级 clip。由阿里 Qwen 团队提出（arXiv:2507.18071），是 Qwen3 模型背后使用的 RL 算法。核心洞察：**reward 是序列级给的，优化粒度也应该在序列级**。

---

## 1. 核心问题：Token 级 Importance Sampling 的根本缺陷

GRPO 和 DAPO 都在 token 级别计算 importance sampling ratio：

$$
w^{\text{GRPO}}_{i,t} = \frac{\pi_\theta (y_{i, t} \mid x, y_{i, \lt t})}{\pi_{\theta_{\text{old}}} (y_{i, t} \mid x, y_{i, \lt t})}
$$

GSPO 论文指出这有一个根本性问题：**每个 token 在 rollout 中只被采样一次**。Importance sampling 的前提是能从同一分布多次采样来估计期望——token 级别的单次采样无法实现有效的分布修正（off-policy correction），反而把高方差噪声引入梯度，导致训练不稳定甚至崩溃。

具体来说，假设一条 response 有 2000 个 token：

- GRPO 对 2000 个 token 各自计算 ratio $w_{i,t}$，每个都带噪声
- 2000 个噪声源叠加 → 梯度估计方差极大
- MoE 模型对此尤其敏感（每个 token 路由到不同 expert，ratio 波动直接扰动 expert 激活平衡）

GSPO 的洞见：**reward 是给整个 sequence 的，所以 importance weight 也应该是 sequence 级的**。

---

## 2. 核心改动：序列级 Importance Ratio

GSPO 对整个序列计算单个 importance ratio，用**几何平均**来归一化长度差异：

$$
w^{\text{GSPO}}_{i} = \left[ \frac{\pi_\theta (y_i \mid x)}{\pi_{\theta_{\text{old}}} (y_i \mid x)} \right]^{\frac{1}{|y_i|}} = \exp\left( \frac{1}{|y_i|} \sum_{t=1}^{|y_i|} \log \frac{\pi_\theta (y_{i, t} \mid x, y_{i, \lt t})}{\pi_{\theta_{\text{old}}} (y_{i, t} \mid x, y_{i, \lt t})} \right)
$$

**直觉**：

- 把 2000 个 token 的 log ratio 求平均 → 一个标量
- 噪声通过平均被平滑掉 → 方差大幅降低
- 长度归一化（$1/|y_i|$）确保长短 response 的 ratio 可比

---

## 3. 目标函数对比

### GRPO（token 级 loss）

$$
\mathcal{L}_{\text{GRPO}}(\theta) = \frac{1}{G} \sum_{i=1}^{G} \frac{1}{|o_i|} \sum_{t=1}^{|o_i|} \min\left( w_{i,t} \hat{A}_i,\ \text{clip}(w_{i,t}, 1-\varepsilon, 1+\varepsilon) \hat{A}_i \right)
$$

- $w_{i,t}$ 逐 token 不同 → clip 也逐 token 不同 → 梯度碎片化

### GSPO（序列级 loss）

$$
\mathcal{L}_{\text{GSPO}}(\theta) = \frac{1}{G} \sum_{i=1}^{G} \min\left( w_i^{\text{GSPO}} \hat{A}_i,\ \text{clip}(w_i^{\text{GSPO}}, 1-\varepsilon, 1+\varepsilon) \hat{A}_i \right)
$$

- 一个 sequence 一个 $w_i$ → clip 统一 → 梯度一致

---

## 4. GSPO-token 变体

GSPO 还提出了一个混合变体，将序列级权重（stop-gradient）与 token 级概率结合：

$$
w_{i, t}^{\text{GSPO-token}} = \text{sg}\left[w_i^{\text{GSPO}}\right] \cdot \frac{\pi_{\theta}(y_{i, t} \mid x, y_{i, \lt t})}{\text{sg}\left[\pi_{\theta}(y_{i, t} \mid x, y_{i, \lt t})\right]}
$$

其中 $\text{sg}[\cdot]$ 表示 stop-gradient（`.detach()`）。

在当前 GRPO 框架下（所有 token 共享同一个 advantage），GSPO-token 与 GSPO 理论上等价。但 GSPO-token 为未来 token 级细粒度 reward 留了扩展接口——如果某天有了 per-token reward，可以直接用。

---

## 5. 为什么对 MoE 特别有效

MoE 模型中，每个 token 路由到不同的 expert。Token 级的 importance ratio 波动会导致：

- 某些 expert 收到的更新信号剧烈震荡
- expert 激活分布失衡 → 部分 expert 过载，部分闲置
- 需要额外的负载均衡技巧来维持稳定

GSPO 的序列级 ratio 把逐 token 波动平滑掉了：

- Expert 激活分布保持稳定
- **不需要额外负载均衡技巧**就能训练 MoE
- 这是 Qwen3（MoE 架构）选择 GSPO 的直接原因

---

## 6. 代码示意

```python
log_ratio = per_token_logps - old_per_token_logps  # (B, seq_len)

# GRPO: token 级 importance weight
importance_weights_grpo = torch.exp(log_ratio)  # (B, seq_len)

# GSPO: 序列级 importance weight（几何平均 = 对数空间算术平均）
seq_log_weight = (log_ratio * mask).sum(-1) / mask.sum(-1)  # (B,)
importance_weights_gspo = torch.exp(seq_log_weight).unsqueeze(-1)  # (B, 1)

# GSPO-token: 序列级权重(stop-grad) × token 级概率
seq_weight_detached = seq_log_weight.detach().exp().unsqueeze(-1)  # (B, 1)
token_prob_ratio = torch.exp(per_token_logps - per_token_logps.detach())
importance_weights_gspo_token = seq_weight_detached * token_prob_ratio
```

训练中通过参数切换：

```bash
--importance_sampling_level token           # GRPO
--importance_sampling_level sequence        # GSPO
--importance_sampling_level sequence_token  # GSPO-token
```

---

## 7. GSPO vs GRPO 对比

| | GRPO | GSPO |
|---|---|---|
| Importance Ratio | Token 级 $w_{i,t}$ | 序列级（几何平均）$w_i$ |
| Clip 粒度 | 逐 token | 逐序列 |
| 方差 | 高（N 个 token → N 个噪声源） | 低（一个标量 → 噪声被平均） |
| MoE 稳定性 | 不稳定，需额外技巧 | **天然稳定** |
| 梯度一致性 | token 间 clip 不一致 → 梯度碎片化 | 序列统一 clip → 梯度一致 |
| 精度鲁棒性 | 对数值精度敏感 | 对精度差异鲁棒 |

---

## 8. 超参数参考（来自论文 Section 5.1）

```yaml
importance_sampling_level: sequence   # GSPO
epsilon: 3e-4                         # clip 下界
epsilon_high: 4e-4                    # clip 上界（非对称 clip，类似 DAPO）
beta: 0                               # 不加 KL 正则
steps_per_generation: 4               # 每批 rollout 分 4 个 minibatch 更新
```

---

## 9. 算法演进关系

```
PPO → GRPO（组内 relative advantage，去 critic）
       → DAPO（Clip-Higher + Dynamic Sampling + Token-Level PG + Overlong Penalty）
              → GSPO（序列级 ratio + 序列级 clip，解决 token 级方差问题）
```

GSPO 和 DAPO 的改进维度是**正交**的：

- DAPO 的 Clip-Higher 和 Dynamic Sampling → 可以叠加到 GSPO
- GSPO 的序列级 ratio → 替代 DAPO 的 Token-Level PG Loss（两者解决不同问题但都涉及 ratio 计算粒度）
- Qwen3 的训练融合了两者的技术

---

## 10. 参考

- **GSPO Paper (2025)**: Zheng et al., *Group Sequence Policy Optimization* — [arXiv:2507.18071](https://arxiv.org/abs/2507.18071)
- **Qwen3**: Qwen Team, Alibaba Inc.
- **ROLL (Alibaba RL Framework)**: [GSPO 文档](https://alibaba.github.io/ROLL/docs/User%20Guides/Algorithms/GSPO/)
- **GRPO (2024)**: Shao et al., *DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models*
