---
title: "Physical Intelligence π0.6 与 Token RL 论文解读：让 VLA 从经验中学习"
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

## 引言

Physical Intelligence (PI) 继 π₀、π₀.₅ 之后，连续推出了两篇关于 VLA（Vision-Language-Action）与强化学习结合的重要工作：

1. **RECAP / π*₀.₆**（2025.11）：让 VLA 通过大规模离线 RL 从自主部署经验中学习，在叠衣服、做咖啡、装纸箱等长周期任务上吞吐量翻倍、失败率减半
2. **RLT / Token RL**（2026.03）：在 RECAP 的基础上，针对精密操作场景，从 VLA 内部提取一个 compact RL Token，用极小网络做高效在线 RL，15 分钟数据即可显著提升

这两篇工作构成了 PI 的"从经验中学习"路线的两个层级：**RECAP 做全局策略的离线迭代优化，RLT 做关键阶段的在线快速精调。** 本文对两篇论文进行系统解读。

---

## 第一篇：RECAP — π\*₀.₆

> 论文：*π\*₀.₆: a VLA That Learns From Experience*  
> arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)  
> 博客: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)

### 一句话概括

**问题**：VLA 模型用模仿学习训练，最多只能做到和演示数据一样好，且存在复合误差（compounding errors）。机器人需要像人一样通过反复练习来掌握技能。

**方案**：RECAP（RL with Experience and Corrections via Advantage-conditioned Policies）——用 advantage-conditioned policy extraction 替代传统 policy gradient，使得大规模 flow-matching VLA 也能稳定地从离线数据 + 在线采集数据中做迭代 RL。

### 核心方法

RECAP 的核心循环只有三步，可以迭代任意多轮：

```
1. Data Collection:  部署当前策略，采集 episode，标注成功/失败（稀疏奖励），
                      可选地由人类专家在机器人犯错时提供 corrective intervention
2. Value Function Training:  用所有历史数据训练一个分布式 value function V^{π_ref}
3. Advantage-Conditioned Training:  基于 value function 计算 advantage，
                      将二值化的 advantage 作为额外条件输入 policy 做监督学习
```

### 为什么不做 Policy Gradient？

这是本文最关键的 design choice。对于 flow-matching 模型，log-likelihood 没有闭式解，PPO/REINFORCE 这类 policy gradient 方法很难直接应用。而且 PI 做的是**迭代离线 RL**（collect → retrain → repeat），不是每步更新，off-policy 的 PPO 极不稳定。

RECAP 的 insight 来自一个理论结果：如果定义

$$
\hat{\pi}(\mathbf{a}|\mathbf{o}) \propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}))^{\beta}
$$

则 $\hat{\pi}$ 保证比 $\pi_{\text{ref}}$ 更好。当 $\beta=1$ 且 improvement indicator $I$ 是 advantage 的二值函数时，$\hat{\pi}(\mathbf{a}|\mathbf{o}) = \pi_{\text{ref}}(\mathbf{a}|I,\mathbf{o})$——即**最优策略就是参考策略在"这是一个好动作"条件下的条件分布**。

于是 policy extraction 退化成一个监督学习问题：同时训练模型输出 $\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t, \ell)$（无条件）和 $\pi_{\theta}(\mathbf{a}_t|I_t, \mathbf{o}_t, \ell)$（以 advantage 为条件），推理时直接 condition on $I_t=\text{True}$：

$$
\min_{\theta} \mathbb{E}_{\mathcal{D}}\Big[-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell) - \alpha\log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)\Big]
$$

这比 PPO/AWR 简单得多，且能利用所有数据（好数据和坏数据都参与训练，只是条件不同）。

### π₀.₆ 模型架构

π₀.₆ 是 π₀.₅ 的升级版，在此基础上 π\*₀.₆ 增加了 advantage conditioning 能力：

| 组件 | 规格 |
|------|------|
| VLM Backbone | Gemma 3 4B |
| Action Expert | 860M params, Flow Matching |
| 输入 | 多相机图像 + 机器人关节状态 + 语言指令 + **二值化 advantage** |
| 输出 | Action chunk（50Hz 关节角 + 夹爪）+ 子任务文本预测 |
| 训练 | Knowledge Insulation (KI) + FAST tokenizer |

**Knowledge Insulation** 是关键设计：action expert 通过 stop gradient 与 VLM 主干隔离，使得 flow-matching 训练不会破坏 VLM 的视觉语言理解能力。同时模型既输出连续 action（flow matching）又输出离散 tokenized action（FAST），两者独立预测。

### 关键结果

**任务**：做意式咖啡、叠各种衣物、组装纸箱——都是长周期、涉及柔性物体/液体的真实任务。

| 指标 | 效果 |
|------|------|
| 吞吐量（最难任务） | **2×+ 提升** |
| 失败率 | **减半** |
| 连续运行 | 咖啡 13 小时、叠衣服 2 小时、工厂纸箱组装——不间断 |
| vs. PPO baseline | 显著优于 PPO（PPO 在不稳定 off-policy 设置下几乎无法提升） |
| vs. AWR baseline | 成功率接近但吞吐量远高于 AWR（AWR 策略更慢） |
| 消除特定 failure mode | 2 轮迭代（600 trajectory/轮）→ 97% 成功率 |

**价值函数可视化**：训练的 value function 能准确识别 episode 中的错误时刻（value 下降）和顺利进展（value 上升）——这是 token-level advantage 计算的基础。

### 局限

- 仍需人工标注奖励和 reset 场景
- 探索策略较 naive（主要靠 policy 随机性 + human intervention）
- 是迭代式离线更新，不是真正的在线 RL

---

## 第二篇：RLT — Token RL

> 论文：*Precise Manipulation with Efficient Online RL*  
> 博客: [pi.website/research/rlt](https://pi.website/research/rlt)  
> PDF: [RLT.pdf](https://physicalintelligence.company/RLT.pdf)（非 arXiv，仅 PI 官网）

### 一句话概括

**问题**：RECAP 虽然能做全局策略优化，但对精密操作（如拧螺丝、插网线），需要的是对特定关键阶段的**快速在线精调**，而不是重新训练整个大模型。训练整个 VLA 做 online RL 在算力和数据效率上都不现实。

**方案**：从 VLA 内部提取一个 compact **RL Token** 作为信息瓶颈，外接一个小 actor/critic 网络做在线 RL。VLA 本体冻结，只训练 actor/critic。几十分钟的 robot 数据就能显著提升精密操作的吞吐量。

### 核心方法：RL Token 的提取与使用

RLT 分两步：

**Step 1 — VLA Adaptation**：在 VLA 上加一个 encoder-decoder transformer。encoder 把 VLA 的内部 embedding 压缩成一个 bottleneck representation（即 RL Token）；decoder 用 reconstruction loss 从 RL Token 恢复原始 VLA embedding。这迫使 RL Token 成为 VLA 感知状态的 compact summary。

**Step 2 — Online RL**：冻结 VLA + encoder，只训练基于 RL Token 的小 actor/critic：

- Actor 接收 RL Token + VLA 原始预测 action → 输出修正后的 action
- Critic 接收 RL Token → 估计 value
- 使用 sample-efficient off-policy RL（数百次更新/秒，直接在机器人上跑）

### 设计要点

四个关键设计让极小的 actor/critic 也能有效学习：

1. **Action Chunk 保持一致性**：actor 预测 action chunks，与 VLA 的 temporally extended 动作结构对齐
2. **Reference-Action Regularization**：actor 以 VLA 原始预测 action 为输入，学习**编辑**而非替换——在 VLA 行为合理时保持接近，只在 critic 认为有更好选择时偏离
3. **Reference-Action Dropout**：随机丢弃 VLA 参考 action，防止 actor 早期阶段简单复制 VLA
4. **Human Intervention Folding**：可选地将人类纠正动作直接纳入 RL 更新

### 关键结果

RLT 在四个需要亚毫米精度的接触式操作任务上评估：

| 任务 | 精度要求 | 效果 |
|------|----------|------|
| 电动螺丝刀打 M3 螺丝 | 亚毫米位姿对齐 | 吞吐量提升显著 |
| 扎带紧固 | 精确穿入 + 拉紧 | 吞吐量提升显著 |
| 以太网线插入 | 精准对准 RJ45 口 | **3× 速度提升** |
| 电源线插入 | 精准对准 | 吞吐量提升显著 |

**最惊人的结果**：以太网线插入任务上，RLT 训练后的策略**中位速度（66 timesteps）超过了人类遥操作（146 timesteps）**，且一半的 RL 策略 trial 比任何人类遥操作 demo 都快。

**数据效率**：以太网线插入任务总共 2 小时训练时间，其中仅 **15 分钟**是实际 robot 数据，其余是 reset 等开销。

### 与 RECAP 的关系

这是理解两篇工作的关键：

| 维度 | RECAP (π\*₀.₆) | RLT (Token RL) |
|------|-----------------|----------------|
| **目标** | 全局策略优化，提升整体成功率与速度 | 关键精密阶段局部优化 |
| **RL 范式** | 迭代离线 RL（collect → retrain） | 在线 RL（实时数百次更新/秒） |
| **更新范围** | 整个 VLA 端到端 | 仅小 actor/critic，VLA 冻结 |
| **时间尺度** | 数百 trajectory，多轮迭代 | 15 分钟 robot 数据 |
| **适用场景** | 长周期、多阶段任务 | 高精度、接触式关键步骤 |
| **算力需求** | 大规模 GPU 集群 | 直接在 robot 上运行 |
| **数据复用** | 所有历史数据（offline + online） | 实时 online data + replay buffer |

**两者互补**：RECAP 先全局提升策略，RLT 再针对性打磨最需要精度的环节。未来路线图是让 VLA 在多个时间尺度和抽象层级上都能从经验学习——RECAP 做大规模全局 RL，RLT 做精细局部 RL，再往上可能是基于人类反馈的高层推理 RL。

---

## 总结与思考

### 对 VLA + RL 路线的启示

1. **Advantage Conditioning 是大模型 RL 的实用方案**：避免了 flow-matching 模型上 policy gradient 的困难（无闭式 log-likelihood），把 RL 退化成了两个条件分布之间的监督学习。这个思路对任何使用扩散/flow 模型的 VLA 都有借鉴意义。

2. **Information Bottleneck 是在线 RL 的关键**：RLT 的 RL Token 本质上是一个 task-agnostic 的感知状态摘要，把 VLA 的 rich internal representation 压缩给 RL。这让 online RL 可以脱离全模型训练，在实际部署中实时学习。

3. **"编辑而非替换"的策略**：无论 RECAP 的 advantage-conditioned regularization 还是 RLT 的 reference-action regularization，核心思想都是让 RL 在已有行为基础上微调，而非从零探索。这是真实机器人 RL 的实用主义——完全自由探索太危险且低效。

4. **Human in the Loop 的务实融合**：两篇工作都把人类干预作为有效的数据源（RECAP 的 corrective interventions + RLT 的 intervention folding），不做"完全自主"的教条假设。

### 局限与开放问题

- **自动化程度**：两篇工作仍然依赖人类提供奖励信号和场景重置。如何用 VLA 自身的高层推理能力实现自动 reset 和奖励标注，是走向完全自主 RL 的关键一步。
- **探索效率**：当前探索主要靠 policy 随机性，在初始策略已经不错时够用，但对于需要发现全新行为模式的任务可能不足。
- **RLT 的泛化性**：RL Token 是针对特定 VLA 训练的，换一个 VLA 就需要重新训练 encoder-decoder。如何训练通用的 RL Token 值得探索。
- **安全约束**：真实场景中在线 RL 可能产生危险动作，当前方法通过 reference-action regularization 隐式约束，但缺乏显式的安全保证。

---

## 论文链接

- π\*₀.₆ / RECAP: [arXiv 2511.14759](https://arxiv.org/abs/2511.14759) | [PI Blog](https://pi.website/blog/pistar06)
- RLT / Token RL: [PI Research](https://pi.website/research/rlt) | [PDF](https://physicalintelligence.company/RLT.pdf)
- PI 全部研究: [pi.website/blog](https://pi.website/blog)

