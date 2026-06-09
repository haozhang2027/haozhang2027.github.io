---
title: "RLT / Token RL 论文解读：VLA 的高效在线精密操作 RL"
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

RECAP 做全局策略优化，但对拧螺丝、插网线这类需要亚毫米精度的操作，真正拖后腿的是某个关键接触阶段。RLT 从 VLA 内部提取一个 compact **RL Token** 作为信息瓶颈，外接极小 actor/critic 做在线 RL——VLA 冻结，仅 15 分钟 robot 数据就能让策略速度超越人类遥操作。

> 论文：*Precise Manipulation with Efficient Online RL*  
> 博客: [pi.website/research/rlt](https://pi.website/research/rlt)  
> PDF: [RLT.pdf](https://physicalintelligence.company/RLT.pdf)（非 arXiv，仅 PI 官网）  
> 作者：Charles Xu, Jost Tobias Springenberg, Michael Equi, Ali Amin, Adnan Esmail, Sergey Levine, Liyiming Ke

---

## 动机：从全局优化到局部精调

RECAP 虽然能端到端优化整个 VLA 策略，但有两个关键局限：

1. **算力不现实**：训练整个 VLA（Gemma 3 4B + 860M action expert）做 online RL 需要大规模 GPU 集群，不可能在部署的机器人上实时运行。online RL 要求每收集少量数据就做梯度更新——全模型训练做不到
2. **粒度不匹配**：很多任务的 90% 阶段 base model 已经做得不错（比如拿起螺丝刀、走到目标位置），只有某个接触式关键步骤需要精度提升——全模型重训是杀鸡用牛刀，浪费算力且可能破坏已有的良好行为

以一个拧螺丝任务为例：拿起螺丝刀、移动到螺丝上方——这些粗粒度动作 VLA 做得很稳。但对准 M3 螺丝（直径 3mm）——需要亚毫米精度对齐，且螺丝刀尖端距离握持点 10cm，任何微小的手腕抖动都会被放大数倍——这就是瓶颈。

**核心洞察**：不训练 VLA 本体做 RL，而是让 VLA 输出一个 compact representation（RL Token），只训练基于这个 token 的小网络。VLA 负责"看懂世界"，小网络负责"做得更准"。

---

## 核心方法

RLT 分两步：

### Step 1：VLA Adaptation——学习 RL Token

在冻结的 VLA 上加一个 encoder-decoder transformer：

**Encoder**：接收 VLA 某一层或多层的内部 embeddings（$\mathbf{h}_{\text{VLA}} \in \mathbb{R}^{d}$），通过一系列 transformer 层压缩成 bottleneck representation：

$$
\begin{aligned}
\mathbf{z}_{\text{RL}} &= \text{Encoder}(\mathbf{h}_{\text{VLA}}) \in \mathbb{R}^{k}
\end{aligned}
$$

- $\mathbf{h}_{\text{VLA}}$：VLA 内部的 hidden states，包含了丰富的视觉感知、语言理解和任务上下文信息。维度 $d$ 通常是几千
- $\mathbf{z}_{\text{RL}}$：压缩后的 RL Token，维度 $k \ll d$。这是对 VLA 内部状态的 compact summary
- 关键约束：$k$ 足够小，使得后续的 actor/critic 可以极轻量

**Decoder + Reconstruction Loss**：Decoder 从 RL Token 恢复原始 VLA embeddings：

$$
\begin{aligned}
\hat{\mathbf{h}}_{\text{VLA}} &= \text{Decoder}(\mathbf{z}_{\text{RL}}) \\[4pt]
\mathcal{L}_{\text{recon}} &= \|\hat{\mathbf{h}}_{\text{VLA}} - \mathbf{h}_{\text{VLA}}\|^2
\end{aligned}
$$

- $\|\cdot\|^2$ 是 MSE loss——让 decoder 重建的 embeddings 尽可能接近原始
- Information bottleneck 原理：reconstruction loss 迫使 $\mathbf{z}_{\text{RL}}$ 保留足够的信息来恢复 $\mathbf{h}_{\text{VLA}}$，同时维度压缩淘汰了噪声和冗余
- 训练收敛后，$\mathbf{z}_{\text{RL}}$ 成为 VLA 感知状态的 compact 且 informative 的摘要

这一阶段训练完成后，VLA + Encoder 全部冻结，只有 Decoder 被丢弃——后续 RL 只需要 $\mathbf{z}_{\text{RL}}$。

### Step 2：Online RL——小 actor/critic

只训练两个小网络（通常只有几层 MLP）：

**Actor**：$\pi_{\text{RL}}(\mathbf{a}_t^{\text{correction}} | \mathbf{z}_{\text{RL}}, \mathbf{a}_t^{\text{VLA}})$
- 输入：RL Token $\mathbf{z}_{\text{RL}}$ + VLA 原始预测动作 $\mathbf{a}_t^{\text{VLA}}$
- 输出：对 VLA 动作的修正量（或直接输出最终动作）
- 学习目标：最大化期望累积奖励

**Critic**：$Q(\mathbf{z}_{\text{RL}}, \mathbf{a}_t^{\text{correction}})$ 或 $V(\mathbf{z}_{\text{RL}})$
- 输入：RL Token（+ 可选的动作）
- 输出：估计的 state-action value 或 state value
- 学习目标：TD error 最小化

使用 sample-efficient off-policy RL 算法（如 SAC 或 TD3 的变体），直接在机器人上运行。因为 actor/critic 极小，可以实现**数百次更新/秒**。

---

## 四个关键设计

### 1. Action Chunk 保持一致性

VLA 输出的是 action chunks——一次预测未来 $H$ 步的关节角序列（如 50Hz × 1 秒 = 50 步）。RL actor 同样预测 action chunks，而不是逐 control step 决策。

为什么重要：精密操作的成功往往取决于一段连续动作的协调性（如对准→插入→旋转），单步修正可能破坏动作序列的 temporal consistency。

### 2. Reference-Action Regularization

Actor 以 VLA 原始预测动作为输入，学习**编辑**而非替换：

$$
\begin{aligned}
\mathcal{L}_{\text{actor}} &= \mathcal{L}_{\text{RL}} + \lambda \cdot \|\mathbf{a}_{\text{actor}} - \mathbf{a}_{\text{VLA}}\|^2
\end{aligned}

- $\mathcal{L}_{\text{RL}}$：标准 RL 目标（actor-critic loss），驱动 actor 向更高奖励的方向更新
- $\|\mathbf{a}_{\text{actor}} - \mathbf{a}_{\text{VLA}}\|^2$：L2 正则项，惩罚 actor 偏离 VLA 原始动作太远
- $\lambda$：控制正则化强度。$\lambda$ 大 → 保守，只在确有把握时修正；$\lambda$ 小 → 激进，更自由地探索新行为

实际效果：在 VLA 动作已经合理的区域（如任务的前 90%），actor 输出接近零的修正量；只在 critic 认为有显著改进空间的关键阶段才大幅偏离。这是真实机器人 RL 的实用主义——完全自由探索太危险且低效。

### 3. Reference-Action Dropout

以一定概率（如 30%）将 VLA 参考动作替换为零向量：

- **目的**：防止 actor 在训练早期学会"简单复制 VLA 输出 + 微小噪声"，导致 critic 无法学到有意义的 value，actor 也无法形成独立的动作生成能力
- **效果**：dropout 迫使 actor 维护两条通路——一条是"参考 VLA 并微调"，一条是"完全自主生成"。随着训练进行，actor 逐渐学会在两条通路间做最优权衡
- **与正则化的关系**：dropout 和 L2 正则化互补。L2 防止偏离太远，dropout 防止粘得太紧

### 4. Human Intervention Folding

当机器人卡住或犯错时，人类遥控接管产生的纠正动作**直接加入 replay buffer**，作为额外的训练数据参与 RL 更新。

与 RECAP 中 human corrections 的对比：RECAP 把纠正动作作为 supervised learning target（强制 $I_t=\text{True}$），RLT 则是直接混入 RL 的经验池（replay buffer），让 critic 和 actor 自己从中学习。两种方式可以互补。

---

## 实验结果

四个需要亚毫米精度的接触式操作任务：

| 任务 | 精度要求 | 瓶颈描述 | 效果 |
|------|----------|----------|------|
| 电动螺丝刀打 M3 螺丝 | 亚毫米位姿对齐 | 螺丝刀尖端距握持点 10cm，抖动被放大；wrist camera 视角几乎看不清交互细节 | 吞吐量提升显著 |
| 扎带紧固 | 精确穿入孔洞 + 拉紧 | 扎带是柔性物体，形变不可预测 | 吞吐量提升显著 |
| 以太网线插入 | RJ45 口对准（~1mm 容差） | 需要同时控制位置 + 角度 + 插入力 | **3× 速度提升** |
| 电源线插入 | 精准对准 | 类似网线但物理约束不同 | 吞吐量提升显著 |

### 核心指标：以太网线插入

这是最惊人的结果：

- RLT 策略**中位速度 66 timesteps**，人类遥操作 **146 timesteps**
- **50% 的 RLT trial 比任何人类遥操作 demo 都快**
- 训练总时间 2 小时，其中**实际 robot 数据仅 15 分钟**（其余是 reset、replay buffer 采样等开销）

这个结果的意义：RLT 不仅学到了如何完成任务，而且学到的策略**速度超过了人类遥操作者**。这证明了 RL 可以突破模仿学习的天花板——不是因为 VLA 不会做，而是因为 VLA 不够快/不够准，RL 正是解决这两个维度的正确工具。

---

## 与 RECAP 的对比

| 维度 | RECAP | RLT |
|------|-------|-----|
| **目标** | 全局策略优化，提升整体成功率与速度 | 关键精密阶段局部优化 |
| **RL 范式** | 迭代离线 RL（collect → retrain → repeat） | 在线 RL（实时数百次更新/秒） |
| **更新范围** | 整个 VLA 端到端 | 仅小 actor/critic，VLA 冻结 |
| **时间尺度** | 数百 trajectory，多轮迭代 | 15 分钟 robot 数据 |
| **算力需求** | 大规模 GPU 集群 | 直接在机器人 onboard 运行 |
| **数据复用** | 所有历史数据（offline + online + interventions） | 实时 online data + replay buffer |
| **适用场景** | 长周期、多阶段、柔性物体任务 | 高精度、接触式关键步骤 |

**两者互补**：RECAP 做全局粗调（让 VLA 整体变得更好），RLT 做局部精调（在 VLA 的基础上磨最关键的环节）。PI 的长期路线图是让 VLA 在多个时间尺度上都能从经验学习——分钟级的精密操作 RL（RLT）、小时级的单任务 RL（RECAP specialist）、天级的大规模全局 RL（RECAP generalist）。

---

## 启发

1. **Information Bottleneck 是在线 RL 接入大模型的关键**：RL Token 本质上是 task-agnostic 的感知状态摘要。把 VLA 的 rich internal representation（几千维）压缩成几百维的 token，让 online RL 脱离全模型训练成为可能。这个模式可以推广到任何"大模型感知 + 小网络决策"的场景。

2. **"编辑而非替换"是真实机器人 RL 的实用主义**：reference-action regularization 让 RL 在已有行为基础上微调。在安全敏感的物理交互中，完全自由探索既危险又低效——大部分情况下 VLA 已经做得不错，只需要在瓶颈处修正。

3. **RL 的粒度选择决定效率**：不是所有 RL 都需要端到端。识别任务的瓶颈阶段、只对瓶颈做 RL，是最经济高效的方式。这需要好的任务分析——哪些阶段 base model 已经够好？哪些阶段是性能瓶颈？

4. **Reconstruction Loss 是表示学习的万能工具**：RLT 用 autoencoder-style 的训练让 VLA 自己学到什么是"对 RL 有用的 compact representation"。没有手工设计 reward，没有人工标注，纯粹的自监督。

---

## 局限与开放问题

- **RL Token 的泛化性**：当前 RL Token 是针对特定 VLA 训练的，换模型就要重新训练 encoder-decoder。能否训练 task-agnostic、model-agnostic 的通用 RL Token？这可能需要大规模多模型、多任务预训练
- **安全约束**：在线 RL 可能产生危险动作（如用力过猛损坏硬件）。当前靠 reference-action regularization 隐式约束，缺乏显式的 safety layer 或约束优化
- **自动化程度**：仍然依赖人类 reset 场景。对于连续生产场景，自动 reset 是实现完全自主 RL 的最后一块拼图
- **长周期扩展**：当前 RL Token 针对单一瓶颈阶段。对于有多个瓶颈阶段的长任务，需要多个 RL Token 或层次化 RL Token 结构

---

## 论文链接

- PI 博客: [pi.website/research/rlt](https://pi.website/research/rlt)
- PDF: [RLT.pdf](https://physicalintelligence.company/RLT.pdf)
- 上一篇：[RECAP / π\*₀.₆](https://haozhang2027.github.io/posts/recap-pi06/)
