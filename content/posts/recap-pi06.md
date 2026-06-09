---
title: "RECAP / π*₀.₆ 论文解读：让 VLA 从经验中学习"
date: 2026-06-09
draft: false
math: true
tags:
  - Robot Learning
  - VLA
  - Reinforcement Learning
  - Physical Intelligence
categories:
  - Robotics
---

## 一句话

VLA 模型只靠模仿学习训练，最多做到和演示数据一样好，且存在复合误差。RECAP 用 **advantage-conditioned policy extraction** 替代传统的 policy gradient，让大规模 flow-matching VLA 能从离线数据 + 在线采集数据中做迭代 RL——叠衣服、做咖啡、装纸箱等长周期真实任务上吞吐量翻倍、失败率减半。

> 论文：*π\*₀.₆: a VLA That Learns From Experience*  
> arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)  
> 博客: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)  
> 作者：Physical Intelligence（42 位作者）

---

## 背景：为什么 VLA 需要 RL

模仿学习训练的策略有两个根本问题：

1. **复合误差（compounding errors）**：policy 在部署时偏离训练分布，错误随时间累积
2. **天花板效应**：最多只能做到和演示数据一样好，无法超越人类遥操作水平

机器人需要像人一样通过反复练习来掌握技能——这就是 RL。但把 RL 用在 VLA 上有三个难点：

- Flow-matching/diffusion 模型没有闭式 log-likelihood，PPO/REINFORCE 这类 policy gradient 方法难以直接应用
- 离线 + 在线混合数据（demonstrations + autonomous rollouts + human interventions），数据来源异构
- 大规模 VLA（Gemma 3 4B + 860M action expert）的训练稳定性

---

## 核心方法：RECAP

RECAP（RL with Experience and Corrections via Advantage-conditioned Policies）的核心循环只有三步：

```
1. Data Collection
   部署当前策略，标注成功/失败（稀疏奖励）
   可选：人类专家在机器人犯错时提供 corrective intervention

2. Value Function Training
   用所有历史数据训练分布式 value function V^{π_ref}

3. Advantage-Conditioned Training
   基于 value function 计算 advantage
   将二值化 advantage 作为额外条件输入 policy 做监督学习
```

### 核心 insight：advantage conditioning

RECAP 最关键的设计选择是不用 policy gradient，而是把 RL 退化成一个条件监督学习问题。

理论基础：如果定义

$$
\hat{\pi}(\mathbf{a}|\mathbf{o}) \propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}))^{\beta}
$$

则 $\hat{\pi}$ 保证比 $\pi_{\text{ref}}$ 更好。当 $\beta=1$ 且 improvement indicator $I$ 是 advantage 的二值函数时：

$$
\hat{\pi}(\mathbf{a}|\mathbf{o}) = \pi_{\text{ref}}(\mathbf{a}|I,\mathbf{o})
$$

**最优策略就是参考策略在"这是一个好动作"条件下的条件分布。**

于是 policy extraction 退化成：

$$
\min_{\theta} \mathbb{E}_{\mathcal{D}}\Big[-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell) - \alpha\log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)\Big]
$$

其中 $I_t = \mathds{1}(A^{\pi_{\text{ref}}}(\mathbf{o}_t, \mathbf{a}_t, \ell) > \epsilon_{\ell})$。

这个方法的关键优势：
- **不需要 flow-matching 的 log-likelihood 闭式解**：只需要 flow matching 训练时已有的 ELBO 下界
- **所有数据都参与训练**：好数据和坏数据都用，只是条件不同（$I_t=\text{True}$ vs $I_t=\text{False}$）
- **推理时直接 condition on $I_t=\text{True}$**：无需额外采样或 CFG 调参
- 相比 PPO baseline，RECAP 显著更好；相比 AWR，成功率接近但吞吐量远高于 AWR

### 价值函数

训练一个多任务分布式 value function $p_{\phi}(V|\mathbf{o}_t, \ell)$，201 个离散 bin，架构与 VLA 相同但 VLM backbone 更小。从 value distribution 提取连续 advantage：

$$
V^{\pi_{\text{ref}}}(o_t,\ell) = \sum_{b\in[0,B]} p_{\phi}(V=b|\mathbf{o}_t) \cdot v(b)
$$

然后用 n-step 估计计算 advantage $A^{\pi_{\text{ref}}}$。

可视化显示该 value function 能准确识别 episode 中的错误时刻（value 下降）和顺利进展（value 上升）。

---

## 模型架构

π₀.₆ 是 π₀.₅ 的升级版，在此基础上 π\*₀.₆ 增加了 advantage conditioning：

| 组件 | 规格 |
|------|------|
| VLM Backbone | Gemma 3 4B |
| Action Expert | 860M params, Flow Matching |
| 输入 | 多相机图像 + 机器人关节状态 + 语言指令 + **二值化 advantage indicator** |
| 输出 | Action chunk（50Hz 关节角 + 夹爪）+ 子任务文本预测 + FAST tokenized actions |
| 训练 | Knowledge Insulation (KI) + stop gradient |

**Knowledge Insulation** 是关键设计：action expert 通过 stop gradient 与 VLM 主干隔离，flow-matching 训练不会破坏 VLM 的视觉语言理解能力。模型同时输出连续 action（flow matching）和离散 tokenized action（FAST），两者独立预测。

预训练数据包含数万小时的 demonstrations，覆盖多种任务和多种机器人平台。

---

## 关键结果

**任务**：做意式咖啡、叠各种衣物、组装纸箱——都是长周期、涉及柔性物体/液体的真实部署任务。

| 指标 | 效果 |
|------|------|
| 吞吐量（最难任务） | **2×+ 提升** |
| 失败率 | **~2× 降低** |
| 连续运行 | 咖啡 13 小时、叠衣服 2 小时、工厂纸箱组装——不间断 |
| vs. PPO | 显著优于 PPO（PPO 在 off-policy 设置下几乎无法提升） |
| vs. AWR | 成功率接近但吞吐量远高于 AWR（AWR 策略更慢） |
| 消除特定 failure mode | 2 轮迭代（每轮 600 trajectory）→ 97% 成功率 |

**Policy gradient 为什么不行？** 实验对比了 PPO 变体（SPO+CoVLA）和 AWR。PPO 需要极小的 trust-region constraint（$\eta=0.01$）才能稳定训练，但即便如此也无法获得好的性能。AWR 虽然能达到可接受的成功率，但学习出的策略更慢、吞吐量更低。

---

## 局限

- **非完全自主**：仍需人工标注奖励 + 场景 reset。论文指出 VLA 的高层推理能力有望用于自动 reset
- **探索策略 naïve**：主要靠 policy 随机性 + human intervention，无结构化探索
- **迭代式离线更新**：collect → retrain → repeat，不是真正的在线 RL（后续 RLT 工作补充了这一点）

---

## 启发

1. **Advantage Conditioning 是大模型 RL 的实用方案**：绕过了 flow-matching/diffusion 模型上 policy gradient 的无解问题，把 RL 退化成两个条件分布之间的监督学习。对任何使用扩散/flow 模型的 VLA 都有借鉴意义。

2. **"用所有数据，标记好坏" 比 "只挑好数据" 更有效**：AWR 本质是 filtered imitation——丢弃或降权大量数据。RECAP 保留所有数据，只是用 advantage 标记好坏，让模型自己学会区分。

3. **Large VLA + Value Function 双模型架构**：VLA 做策略，更小的 VLM 做 value function——这种非对称设计兼顾了策略表达能力和价值估计的稳定性。

---

## 论文链接

- arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)
- PI Blog: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)
- 下一篇：[RLT / Token RL](https://haozhang2027.github.io/posts/rlt-token-rl/)
