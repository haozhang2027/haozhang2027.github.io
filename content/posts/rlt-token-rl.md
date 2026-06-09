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

1. **算力不现实**：训练整个 VLA（Gemma 3 4B + 860M action expert）做 online RL 需要大规模 GPU 集群，不可能在部署的机器人上实时运行
2. **粒度不匹配**：很多任务的大部分阶段 base model 已经做得不错，只有某个接触式关键步骤需要精度提升——全模型重训是杀鸡用牛刀

**核心洞察**：我们不训练 VLA 本体做 RL，而是让 VLA 输出一个 compact representation（RL Token），只训练基于这个 token 的小网络。

---

## 核心方法

RLT 分两步：

### Step 1：VLA Adaptation——学习 RL Token

在 VLA 上加一个 encoder-decoder transformer：

- **Encoder** 把 VLA 的内部 embeddings 压缩成一个 bottleneck representation（RL Token）
- **Decoder** 用 reconstruction loss 从 RL Token 恢复原始 VLA embeddings

这迫使 RL Token 成为 VLA 感知状态的 compact summary——类似自编码器，但保留的是对 RL 有用的信息。VLA + encoder 训练完成后全部冻结。

$$
\mathcal{L}_{\text{recon}} = \|\text{Decoder}(\text{Encoder}(h_{\text{VLA}})) - h_{\text{VLA}}\|^2
$$

### Step 2：Online RL——小 actor/critic

只训练两个小网络：

- **Actor**：RL Token + VLA 原始预测 action → 输出**修正后的 action**
- **Critic**：RL Token → 估计 value

使用 sample-efficient off-policy RL，直接在机器人上运行，**数百次更新/秒**。

---

## 四个关键设计

### 1. Action Chunk 保持一致性

Actor 预测 action chunks，与 VLA 的 temporally extended 动作结构对齐——不是逐 control step 决策，而是对整个动作序列做偏移修正。

### 2. Reference-Action Regularization

Actor 以 VLA 原始预测 action 为输入，学习**编辑**而非替换。正则化让策略在 VLA 行为合理时保持接近，只在 critic 认为有更好选择时偏离。

$$
\mathcal{L}_{\text{actor}} = \mathcal{L}_{\text{RL}} + \lambda \|\mathbf{a}_{\text{actor}} - \mathbf{a}_{\text{VLA}}\|^2
$$

### 3. Reference-Action Dropout

随机丢弃 VLA 参考 action（用零向量替代），防止 actor 早期阶段简单复制 VLA 而学不到独立行为。

### 4. Human Intervention Folding

可选地将人类纠正动作直接纳入 RL 更新——当机器人卡住或犯错时，人类的纠正变成额外的训练信号。

---

## 实验结果

四个需要亚毫米精度的接触式操作任务：

| 任务 | 精度要求 | 效果 |
|------|----------|------|
| 电动螺丝刀打 M3 螺丝 | 亚毫米位姿对齐 | 吞吐量提升显著 |
| 扎带紧固 | 精确穿入 + 拉紧 | 吞吐量提升显著 |
| 以太网线插入 | 精准对准 RJ45 口 | **3× 速度提升** |
| 电源线插入 | 精准对准 | 吞吐量提升显著 |

### 最惊人的结果

以太网线插入任务：

- RLT 策略**中位速度 66 timesteps**，人类遥操作 **146 timesteps**
- **50% 的 RLT trial 比任何人类遥操作 demo 都快**
- 训练总时间 2 小时，其中**实际 robot 数据仅 15 分钟**（其余是 reset 等开销）

### 螺丝刀任务详解

机器人需要在 10cm 的力臂末端实现亚毫米精度的位姿对齐——即使微小的握持误差或手腕抖动也会在接触点被放大数倍。从机器人自带的 wrist camera 视角甚至看不清交互细节。base VLA 能完成拿起螺丝刀等粗粒度动作，但在对准螺丝这一关键阶段反复失败。RLT 专门针对这一阶段优化，15 分钟数据即显著提升。

---

## 与 RECAP 的对比

| 维度 | RECAP | RLT |
|------|-------|-----|
| **目标** | 全局策略优化 | 关键精密阶段局部优化 |
| **RL 范式** | 迭代离线 RL | 在线 RL（实时数百次/秒） |
| **更新范围** | 整个 VLA 端到端 | 仅小 actor/critic，VLA 冻结 |
| **时间尺度** | 数百 trajectory，多轮迭代 | 15 分钟 robot 数据 |
| **算力需求** | 大规模 GPU 集群 | 直接在 robot 上运行 |
| **适用场景** | 长周期、多阶段任务 | 高精度、接触式关键步骤 |

**两者互补**：RECAP 做全局粗调 → RLT 做局部精调。PI 的路线图是让 VLA 在多个时间尺度上都能从经验学习。

---

## 启发

1. **Information Bottleneck 是在线 RL 的关键**：RL Token 本质上是 task-agnostic 的感知状态摘要。把 VLA 的 rich internal representation 压缩给 RL，让 online RL 脱离全模型训练。

2. **"编辑而非替换"是真实机器人 RL 的实用主义**：reference-action regularization 让 RL 在已有行为基础上微调——完全自由探索太危险且低效。

3. **RL 的粒度选择决定效率**：不是所有 RL 都需要端到端。识别任务的瓶颈阶段、只对瓶颈做 RL，是最经济高效的方式。

---

## 局限与开放问题

- **RL Token 的泛化性**：当前 RL Token 是针对特定 VLA 训练的，换模型就要重新训练 encoder-decoder。通用的 RL Token 值得探索
- **安全约束**：在线 RL 可能产生危险动作，当前靠 reference-action regularization 隐式约束，缺乏显式安全保证
- **自动化程度**：仍然依赖人类 reset 场景

---

## 论文链接

- PI 博客: [pi.website/research/rlt](https://pi.website/research/rlt)
- PDF: [RLT.pdf](https://physicalintelligence.company/RLT.pdf)
- 上一篇：[RECAP / π\*₀.₆](https://haozhang2027.github.io/posts/recap-pi06/)
