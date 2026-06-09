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

## 术语速查

本文涉及的概念如果在上一篇 [RECAP 解读](https://haozhang2027.github.io/posts/recap-pi06/)中出现过，这里只简要回顾。新概念会详细解释。

| 术语 | 一句话解释 |
|------|-----------|
| **online RL vs offline RL** | online RL：策略在真实环境中边交互边学习，实时更新。offline RL：从固定的历史数据集中学习，不与环境交互。RLT 是 online，RECAP 是 offline |
| **replay buffer（经验回放池）** | 存储"（状态，动作，奖励，下一状态）"四元组的缓冲区。off-policy RL 从中随机采样训练，打破数据的时间相关性 |
| **actor-critic** | RL 的一种架构：actor（演员）负责选择动作，critic（评论家）负责评价动作好坏。critic 训练 actor，两者互相促进 |
| **TD error（时序差分误差）** | critic 的训练信号：$\delta = r + \gamma V(s') - V(s)$。实际奖励 + 对未来价值的估计 - 当前对价值的估计。如果 $\delta > 0$，说明当前状态比预期好 |
| **information bottleneck（信息瓶颈）** | 来自信息论的概念：强迫信息通过一个窄"瓶颈"（低维表示），只保留对下游任务最关键的信息。RLT 中 RL Token 就是这样的瓶颈 |
| **reconstruction loss（重建损失）** | 自编码器的核心训练目标：输入 → 压缩 → 解压，让解压结果尽可能接近原始输入。$\mathcal{L} = \|\text{Dec}(\text{Enc}(x)) - x\|^2$。迫使压缩表示保留足够信息 |
| **L2 regularization（L2 正则化）** | 在 loss 中加 $\lambda\|w\|^2$ 项，惩罚参数过大。在 RLT 中用于约束 actor 输出不要偏离 VLA 原始动作太远 |
| **SAC / TD3** | 两种主流 off-policy actor-critic RL 算法。SAC 加 entropy bonus 鼓励探索，TD3 用双 Q 网络减少 overestimation。RLT 可以选用其中任意一种 |
| **action chunk** | 一次预测未来多步动作（如 50Hz × 1 秒 = 50 步），而不是每步单独决策。好处是保持动作序列的 temporal consistency，避免逐步独立预测导致的抖动 |
| **M3 螺丝** | M3 = 直径 3mm。这是非常小的螺丝——一般手机、笔记本内部用的就是这个尺寸。需要亚毫米精度的对准才能成功拧入 |

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

1. **算力不现实**：训练整个 VLA（Gemma 3 4B + 860M action expert）做 online RL 需要大规模 GPU 集群，不可能在部署的机器人 onboard 计算机上实时运行。online RL 要求每收集少量数据就做梯度更新——全模型训练的每次梯度更新可能需要数秒甚至数分钟，远远跟不上"实时交互→实时学习"的节奏

2. **粒度不匹配**：很多任务的 90% 阶段 base model 已经做得不错（比如拿起螺丝刀、移动到螺丝上方），只有某个接触式关键步骤需要精度提升。全模型重训是杀鸡用牛刀——浪费算力，而且可能破坏已有的良好行为（灾难性遗忘）

以拧 M3 螺丝（直径仅 3mm，见术语速查）为例：拿起螺丝刀、移动到螺丝上方——这些粗粒度动作 VLA 做得很稳。但对准螺丝——螺丝刀尖端距离握持点约 10cm，任何微小的手腕抖动（比如 0.5° 的角度偏差）都会被这个 10cm 力臂放大成尖端约 0.9mm 的位移误差，远超 M3 螺丝的容差。从机器人自带的 wrist camera 视角，这些交互细节几乎看不清——这就是整个任务的性能瓶颈。

**核心洞察**：不训练 VLA 本体做 RL，而是让 VLA 输出一个 compact representation（RL Token），只训练基于这个 token 的小网络。VLA 负责"看懂世界"（rich perception），小网络负责"做得更准"（fast adaptation）。

---

## 核心方法

RLT 分两步：

### Step 1：VLA Adaptation——学习 RL Token

在冻结的 VLA 上加一个 encoder-decoder transformer。所谓"冻结"，就是 VLA 的所有参数在这一步之后不再更新。

**Encoder**：接收 VLA 某一层或多层的内部 embeddings（$\mathbf{h}_{\text{VLA}} \in \mathbb{R}^{d}$，$d$ 通常是几千维——VLA 在这个高维空间里编码了丰富的视觉感知、语言理解和任务上下文），通过若干 transformer 层逐步压缩成 bottleneck representation：

$$
\begin{aligned}
\mathbf{z}_{\text{RL}} &= \text{Encoder}(\mathbf{h}_{\text{VLA}}) \in \mathbb{R}^{k}
\end{aligned}
$$

- $\mathbf{z}_{\text{RL}}$：压缩后的 RL Token。$k \ll d$（例如 $d=4096$，$k=256$）
- 类比：$\mathbf{h}_{\text{VLA}}$ 是一张高清照片，$\mathbf{z}_{\text{RL}}$ 是这张照片的缩略图——保留了关键信息（谁、在哪、干什么），但丢弃了细节

**Decoder + Reconstruction Loss**：Decoder 尝试从 RL Token 恢复原始 VLA embeddings：

$$
\begin{aligned}
\hat{\mathbf{h}}_{\text{VLA}} &= \text{Decoder}(\mathbf{z}_{\text{RL}}) \\[4pt]
\mathcal{L}_{\text{recon}} &= \|\hat{\mathbf{h}}_{\text{VLA}} - \mathbf{h}_{\text{VLA}}\|^2
\end{aligned}
$$

- $\|\cdot\|^2$ 是 MSE（均方误差）loss，计算 reconstructed embeddings 与原始 embeddings 在每个维度上的平方差之和。越小说明重建得越好
- **Information bottleneck 原理**：reconstruction loss 迫使 $\mathbf{z}_{\text{RL}}$ 必须保留足够的信息才能让 decoder 恢复原始 $\mathbf{h}_{\text{VLA}}$。同时，低维度 $k \ll d$ 天然淘汰了噪声和冗余——那些对重建帮助不大的维度在压缩过程中被丢弃
- 训练收敛后，$\mathbf{z}_{\text{RL}}$ 成为一个 compact 但 informative 的感知状态摘要：它包含了 VLA 对当前场景的理解，但维度足够小，使得后续的网络可以极轻量

这一阶段训练完成后，VLA + Encoder 全部冻结。Decoder 被丢弃——它只是训练 RL Token 的"拐杖"，训练完成后不再需要。

### Step 2：Online RL——小 actor/critic

只训练两个小网络（通常只有几层 MLP，参数量在几万到几十万级别，相比 VLA 的数十亿参数几乎可以忽略）：

**Actor**（策略网络）：$\pi_{\text{RL}}(\mathbf{a}_t^{\text{final}} | \mathbf{z}_{\text{RL}}, \mathbf{a}_t^{\text{VLA}})$
- 输入：RL Token + VLA 原始预测动作
- 输出：最终执行的动作——可以是 VLA 动作 + 修正偏移量，或直接输出最终动作
- 训练目标：最大化 critic 给出的 value 估计（即"让 critic 觉得我做得好"）

**Critic**（价值网络）：$Q(\mathbf{z}_{\text{RL}}, \mathbf{a})$ 或 $V(\mathbf{z}_{\text{RL}})$
- Q 函数估计"在状态 $\mathbf{z}_{\text{RL}}$ 做动作 $\mathbf{a}$ 的期望累积奖励"
- V 函数估计"在状态 $\mathbf{z}_{\text{RL}}$ 的期望累积奖励"（不依赖具体动作）
- 训练目标：最小化 TD error（见术语速查）——让预测的 value 尽可能接近"实际奖励 + 下一状态的估计 value"

使用 sample-efficient off-policy RL 算法（如 SAC 或 TD3，见术语速查），直接在机器人 onboard 计算机上运行。因为 actor/critic 极小：

- **数百次梯度更新/秒**——每次机器人做一个动作，RL 就能做几百次"反思和调整"
- 数据存入 replay buffer（见术语速查），反复采样训练，充分利用每一份真实 robot 数据

---

## 四个关键设计

### 1. Action Chunk 保持一致性

VLA 输出的是 action chunks（见术语速查）——一次预测未来 $H$ 步的关节角序列。RL actor 同样预测 action chunks，而不是逐步单独决策。

为什么重要：精密操作的成功往往取决于一段连续动作的协调性（如对准 → 慢慢插入 → 微调角度）。如果 RL 逐步修正，可能破坏动作序列的 temporal consistency（时间一致性），导致抖动或冲突——比如第 1 步被 RL 向左调了 1mm，第 2 步又被向右调了 0.5mm，来回震荡。

### 2. Reference-Action Regularization

Actor 以 VLA 原始预测动作为输入，学习**编辑**而非从零生成：

$$
\begin{aligned}
\mathcal{L}_{\text{actor}} &= \mathcal{L}_{\text{RL}} + \lambda \cdot \|\mathbf{a}_{\text{actor}} - \mathbf{a}_{\text{VLA}}\|^2
\end{aligned}
$$

- $\mathcal{L}_{\text{RL}}$：标准的 actor-critic loss，驱动 actor 向更高奖励的方向更新——这是 RL 的"进步动力"
- $\|\mathbf{a}_{\text{actor}} - \mathbf{a}_{\text{VLA}}\|^2$：L2 正则项（见术语速查），惩罚 actor 输出偏离 VLA 原始动作太远——这是"安全约束"
- $\lambda$：控制保守程度的超参数。$\lambda$ 大 → actor 保守，只在确有把握时才修正；$\lambda$ 小 → actor 激进，更自由地探索新行为

实际效果：在 VLA 已经做得好的区域（如任务的粗粒度阶段），actor 输出接近零的修正量，几乎完全跟随 VLA；只在 critic 认为有显著改进空间的关键精密阶段（如螺丝对准、网线插入），才大幅偏离 VLA 原始动作。这是真实机器人 RL 的实用主义——在物理世界中，完全自由的探索既可能损坏硬件，也极其低效（大部分随机动作都是无意义的）。

### 3. Reference-Action Dropout

以一定概率（如 30%）将输入给 actor 的 VLA 参考动作替换为零向量：

$$
\mathbf{a}_t^{\text{VLA}} \rightarrow \begin{cases}
\mathbf{0} & \text{with probability } p_{\text{drop}} \\
\mathbf{a}_t^{\text{VLA}} & \text{otherwise}
\end{cases}
$$

- **目的**：防止 actor 在训练早期学会"简单复制 VLA 输出 + 微小噪声"。如果 actor 一开始就发现"照抄 VLA 就能拿不错的分数"，critic 学不到有价值的判断，actor 也永远不会形成独立的动作生成能力——这就是"懒惰学习者"问题
- **效果**：dropout 迫使 actor 同时维护两条通路——一条是"参考 VLA → 微调"（有参考时），一条是"完全自主生成"（无参考时）。随着训练进行，actor 逐渐学会在两条通路间做最优权衡：大部分时候参考 VLA，只有关键阶段才自主发挥
- **与 L2 正则化的互补**：L2 防止偏离太远（"别乱动"），dropout 防止粘得太紧（"别偷懒"）。两者配合形成合理的探索-利用平衡

### 4. Human Intervention Folding

当机器人卡住（比如螺丝刀对不准，反复尝试无果）或犯错（比如用力过猛弹出螺丝）时，人类遥控接管产生的纠正动作**直接加入 replay buffer**，作为额外的训练数据参与 RL 更新。

与 RECAP 中 human corrections 的对比：
- RECAP：纠正动作被当作 supervised learning target，强制 $I_t=\text{True}$，直接告诉模型"照这个做"
- RLT：纠正动作混入 replay buffer，让 critic 自己从中学习"什么样的状态-动作对是好的"，让 actor 自己从中学习"在类似情况下应该怎么做"

两种方式可以互补——RECAP 的方式更直接但有 bias（假设人类总是最优），RLT 的方式更灵活但需要更多数据来从中提取信号。

---

## 实验结果

四个需要亚毫米精度的接触式操作任务：

| 任务 | 精度要求 | 瓶颈描述 | 效果 |
|------|----------|----------|------|
| 电动螺丝刀打 M3 螺丝 | 亚毫米位姿对齐 | 10cm 力臂放大抖动；wrist camera 看不清交互 | 吞吐量显著提升 |
| 扎带紧固 | 精确穿入孔洞 + 拉紧 | 柔性物体形变不可预测 | 吞吐量显著提升 |
| 以太网线插入 | RJ45 口（~1mm 容差） | 需同时控制位置 + 角度 + 插入力 | **3× 速度** |
| 电源线插入 | 精准对准 | 类似网线但物理约束不同（更大力、更紧配合） | 吞吐量显著提升 |

### 以太网线插入：超越人类的速度

这是最惊人的结果：

- RLT 策略**中位速度 66 timesteps**（约 1.3 秒 @ 50Hz），人类遥操作 **146 timesteps**（约 2.9 秒）
- **50% 的 RLT trial 比任何人类遥操作 demo 都快**——不是"平均比人快"，而是"一半的 trial 比最快的人都快"
- 训练开销：总时间 2 小时，其中**实际 robot 数据仅 15 分钟**（其余是 reset 场景、replay buffer 采样等计算开销）

这个结果的意义：RLT 不仅学到了如何完成任务，而且学到的策略**超越了人类遥操作者的速度上限**。这直接证明了 RL 可以突破 imitation learning 的天花板——不是因为 VLA 不会做，而是因为 VLA 不够快/不够准，RL 正是解决这两个维度的正确工具。

### 螺丝刀任务详解

机器人需要在 10cm 力臂的末端实现亚毫米精度的位姿对齐。从信息论角度，这是一个极端困难的感知问题：

- Wrist camera 距离交互点很近但视角受限，看不清全局
- Base camera 能看到全局但螺丝细节被压缩到几个像素
- 即使人类操作者通过遥操作也需要反复微调

Base VLA 能顺利完成拿起螺丝刀、移动到工位等粗粒度动作，但在对准螺丝这一关键阶段反复失败——对准偏差通常只有 1-2mm，但对 M3 螺丝来说这就是失败。RLT 专门针对这一"最后一毫米"优化，15 分钟数据即显著提升成功率。

---

## 与 RECAP 的对比

| 维度 | RECAP | RLT |
|------|-------|-----|
| **目标** | 全局策略优化，提升整体成功率与速度 | 关键精密阶段局部优化 |
| **RL 范式** | 迭代离线 RL（collect → retrain） | 在线 RL（实时数百次更新/秒） |
| **更新范围** | 整个 VLA 端到端 | 仅小 actor/critic，VLA 冻结 |
| **时间尺度** | 数百 trajectory，多轮迭代 | 15 分钟 robot 数据 |
| **算力** | 大规模 GPU 集群 | 机器人 onboard 直接运行 |
| **数据复用** | 所有历史数据 | 实时 online data + replay buffer |
| **场景** | 长周期、多阶段、柔性物体 | 高精度、接触式关键步骤 |

**两者互补**：RECAP 做全局粗调（让 VLA 整体变好）→ RLT 做局部精调（在 VLA 基础上磨最关键环节）。PI 的长期路线图是让 VLA 在多个时间尺度上都能从经验学习——分钟级的精密操作 RL（RLT）、小时级的单任务 RL（RECAP specialist）、天级的大规模全局 RL（RECAP generalist）。

---

## 启发

1. **Information Bottleneck 是"大模型感知 + 小网络决策"的桥梁**：RL Token 把 VLA 几千维的 rich internal representation 压缩成几百维，让 online RL 脱离全模型训练成为可能。这个模式可以推广——任何"大模型负责感知理解、小网络负责快速决策"的场景，都可以用类似的信息瓶颈来桥接。

2. **"编辑而非替换"是真实机器人 RL 的实用主义**：reference-action regularization 让 RL 在已有行为基础上微调。在安全敏感的物理交互中，完全自由探索既危险又低效——大多数情况下 VLA 已经做得不错，只需要在瓶颈处修正。

3. **RL 的粒度决定效率**：不是所有 RL 都需要端到端。识别任务的瓶颈阶段、只对瓶颈做 RL，是最经济高效的方式。这需要好的任务分析——哪些阶段 base model 已经够好？哪些阶段是性能瓶颈？回答这个问题的过程本身就是对任务理解的深化。

4. **Reconstruction Loss 是表示学习的万能工具**：RLT 用 autoencoder-style 的自监督训练让 VLA 自己学到什么是"对 RL 有用的 compact representation"。没有手工设计 reward，没有人工标注——纯粹靠"能不能重建原始 embedding"这个客观标准。

5. **RL Token 的设计体现了"最小接口"原则**：VLA 和 RL 之间只需要一个几百维的向量——不需要共享梯度、不需要联合训练、不需要知道对方的内部结构。这种松耦合让两个子系统可以独立演进。

---

## 局限与开放问题

- **RL Token 泛化性**：当前 RL Token 针对特定 VLA 训练，换模型就要重新训练 encoder-decoder。能否训练 task-agnostic、model-agnostic 的通用 RL Token？
- **安全约束**：在线 RL 可能产生危险动作（如用力过猛损坏硬件）。当前靠 L2 正则化隐式约束，缺乏显式的 safety layer
- **自动化程度**：仍依赖人类 reset 场景。自动 reset 是实现完全自主在线 RL 的最后拼图
- **多瓶颈任务**：当前 RL Token 针对单一瓶颈。对于有多个精密阶段的长任务，需要多个 RL Token 或层次化结构
- **VLA 选择**：当前基于 π₀.₆。不同 VLA 的内部表示结构不同，RL Token 的最优设计（维度、层数、从哪层提取 embedding）可能也需要适配

---

## 论文链接

- PI 博客: [pi.website/research/rlt](https://pi.website/research/rlt)
- PDF: [RLT.pdf](https://physicalintelligence.company/RLT.pdf)
- 上一篇：[RECAP / π\*₀.₆](https://haozhang2027.github.io/posts/recap-pi06/)
