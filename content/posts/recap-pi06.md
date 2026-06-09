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

## 术语速查

阅读本文前，可以先扫一遍这些基本概念。遇到不认识的词也可以随时回来查。

| 术语 | 一句话解释 |
|------|-----------|
| **log-likelihood（对数似然）** | 衡量"模型认为这个数据出现的概率有多大"。$\log \pi_\theta(a\|o)$ 就是在观测 $o$ 下，模型输出动作 $a$ 的概率取对数。值越大（越接近 0），说明模型越"认可"这个动作。它是监督学习中最重要的训练信号——我们通过最大化对数似然来让模型的行为接近数据中的行为 |
| **policy gradient（策略梯度）** | 一种 RL 训练方法：直接用"奖励 × 动作概率的梯度"来更新策略。数学上是 $\nabla_\theta \mathcal{J} = \mathbb{E}[\nabla_\theta \log\pi_\theta(a\|o) \cdot A]$。它的前提是能算 $\log\pi_\theta$（对动作概率求梯度），但 flow-matching 模型算不出这个值 |
| **KL divergence（KL 散度）** | 衡量两个概率分布有多"像"。$D_{KL}(P\|Q)$ 越大，P 和 Q 差异越大。在 RL 中用作正则项：惩罚新策略偏离旧策略太远，防止训崩 |
| **advantage function（优势函数）** | $A(o,a) = Q(o,a) - V(o)$。在状态 $o$ 下做动作 $a$ 比"平均"好多少。正值 = 好动作，负值 = 差动作。RL 的核心信号 |
| **ELBO（证据下界）** | Evidence Lower BOund。当真实概率算不出来时（如 flow-matching 模型），用一个下界来近似。训练时最小化这个下界 ≈ 最大化真实的 log-likelihood。RECAP 的 flow-matching loss 本质就是在优化 ELBO |
| **flow matching** | 一种生成连续数据的框架。思路：学习一个从噪声到目标数据的"流动路径"。比 diffusion 更简洁——不需要前向加噪过程。VLA 用它来生成平滑的机器人动作轨迹（关节角序列） |
| **off-policy RL vs on-policy RL** | on-policy：只能用"当前策略自己采集的"数据训练，采一批训一批，训完就丢。off-policy：可以用历史数据（包括旧策略收集的）反复训练，数据效率高得多。RECAP 是迭代离线 RL，属于 off-policy |
| **behavior cloning（行为克隆）** | 最基础的模仿学习：把人类演示数据当监督信号，让模型直接预测人类会做什么动作。等于"有样学样"，不涉及奖励或探索 |
| **delta function（δ 函数）** | 数学工具：$\delta(x > 0)$ 在 $x > 0$ 时取 1，否则取 0。在本文中用于将连续 advantage 值"二值化"为好/坏标签 |
| **Bayes' rule（贝叶斯公式）** | $P(A\|B) = P(B\|A) \cdot P(A) / P(B)$。在本文中用于将"以 advantage 为条件的动作概率"转化为"以动作为条件的 advantage 概率"，从而消去显式的 advantage 计算 |
| **n-step TD（时序差分）** | 一种 value function 的训练方法：不等到 episode 结束（Monte Carlo），也不只看下一步（1-step TD），而是看未来 $n$ 步的实际奖励 + 第 $n$ 步的估计值。平衡了偏差和方差 |
| **trust-region（信任域）** | PPO 的核心机制：限制每次策略更新的幅度不能太大（用 clip 或 KL 约束）。防止一次"学太多"导致策略崩溃。RECAP 实验中发现即使有 trust-region，PPO 在 offline 数据上也难以训练 |
| **FAST tokenizer** | PI 自研的动作离散化工具：把连续的动作序列（关节角等）压缩成离散 token，类似文本 tokenizer。好处是可以用语言模型的方式训练动作预测（next-token prediction） |
| **Knowledge Insulation（KI）** | PI 的训练技巧：action expert 通过 stop gradient 与 VLM 主干断开。flow-matching loss 不反向传播到 VLM，保护视觉语言能力不被动作训练"污染" |
| **stop gradient** | 深度学习操作：在反向传播时截断梯度流，让某些参数不被更新。KI 中用 stop gradient 隔开 VLM 和 action expert |

---

## 一句话

VLA 模型只靠模仿学习训练，最多做到和演示数据一样好，且存在复合误差（compounding errors）。RECAP 用 **advantage-conditioned policy extraction** 替代传统的 policy gradient，让大规模 flow-matching VLA 能从离线数据 + 在线采集数据中做迭代 RL——叠衣服、做咖啡、装纸箱等长周期真实任务上吞吐量翻倍、失败率减半。

> 论文：*π\*₀.₆: a VLA That Learns From Experience*  
> arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)  
> 博客: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)  
> 作者：Physical Intelligence（42 位作者）

---

## 背景：为什么 VLA 需要 RL

模仿学习训练的策略有两个根本问题：

1. **复合误差（compounding errors）**：policy 在部署时偏离训练分布，错误随时间累积——第 $t$ 步的小偏差导致第 $t+1$ 步的观测分布漂移，进而引发更大的偏差，形成恶性循环
2. **天花板效应**：最多只能做到和演示数据一样好，无法超越人类遥操作水平——因为学习目标就是模仿，没有"做得更好"的机制

机器人需要像人一样通过反复练习来掌握技能——自己做、看结果、调整行为，这就是 RL。但把 RL 用在 VLA 上有三个难点：

- **Flow-matching 模型没有闭式 log-likelihood**（见术语速查）：$\pi_\theta(\mathbf{a}|\mathbf{o})$ 是通过多步迭代去噪定义的，无法像简单分布那样一步算出"这个动作的概率是多少"。PPO/REINFORCE 这一类 policy gradient 方法需要 $\log\pi_\theta$ 来计算梯度，因此不能直接套用在 flow-matching VLA 上
- **数据来源异构**：离线 demonstrations + 在线 autonomous rollouts + human corrective interventions，三者来自不同策略、不同场景，数据分布差异大，需要一种能统一利用的方法
- **大规模训练稳定性**：VLA 有 Gemma 3 4B + 860M action expert，传统 on-policy RL 要求每做几步梯度更新就需要新数据（因为旧数据"过时"了），在真实机器人上完全不现实

---

## 核心方法：RECAP

RECAP（RL with Experience and Corrections via Advantage-conditioned Policies）的核心循环只有三步：

```
1. Data Collection
   部署当前策略到真实机器人，每个 episode 标注成功/失败（稀疏二元奖励）
   可选：人类专家在机器人犯错时遥控接管，提供 corrective intervention

2. Value Function Training
   用所有历史数据训练分布式 value function V^{π_ref}
   输出是对"预期还需多少步才能完成任务"的估计

3. Advantage-Conditioned Training
   用 value function 对每个 action 计算 advantage（比平均好多少/差多少）
   将二值化 advantage 作为额外条件输入 policy，做监督学习
   训练目标同时包括"无条件模仿"和"以 advantage 为条件"
```

### 核心 insight：advantage conditioning 的数学推导

RECAP 最关键的设计选择是不用 policy gradient，而是把 RL 退化成一个条件监督学习问题。下面是完整的数学推导。

**第一步：正则化 RL 的最优解形式**

标准 RL 的目标是最大化期望累积奖励 $\mathcal{J}(\pi) = \mathbb{E}_{\tau}[R(\tau)]$。但为了防止策略在离线数据上训崩（over-optimization——模型"过拟合"到数据中的偏差，导致实际部署时表现反而变差），需要加正则化项约束新策略不要离参考策略 $\pi_{\text{ref}}$（通常是收集这批数据的行为策略）太远：

$$
\begin{aligned}
\mathcal{J}(\pi, \pi_{\text{ref}}) &= \mathbb{E}_{\tau \sim \rho_{\pi}}\Big[\sum_{t=0}^{T} r_t\Big] - \beta \cdot \mathbb{E}_{\mathbf{o} \sim \rho_{\pi}}\Big[D_{\text{KL}}(\pi(\cdot|\mathbf{o}) \parallel \pi_{\text{ref}}(\cdot|\mathbf{o}))\Big]
\end{aligned}
$$

- 第一项 $\mathbb{E}_{\tau}[\sum r_t]$：期望总奖励，越大越好。$\mathbb{E}$ 是"期望/平均值"的符号，$\tau$ 是一条轨迹
- 第二项 $\beta \cdot D_{\text{KL}}$：KL 散度惩罚（见术语速查）。$\beta$ 越大，新策略越接近参考策略，越保守；但也越难超越参考策略。这是 RL 中经典的"探索-利用"权衡

这个正则化问题有闭式最优解（即可以直接写出 $\hat{\pi}$ 的公式，不需要迭代优化）：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot \exp\!\big(A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}) / \beta\big)
\end{aligned}
$$

- $A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})$ 是 advantage function（见术语速查）
- $\exp(A/\beta)$：advantage 越大（动作越好），该动作在最优策略 $\hat{\pi}$ 中的权重指数级增长。$\propto$ 表示"正比于"——还需要对动作空间做归一化

但这个形式要求能显式计算 $\exp(A/\beta)$ 并做归一化——即对所有可能的动作 $\mathbf{a}$ 求和/积分。在 VLA 的高维连续动作空间（如 7 自由度 × 50 步 = 350 维）中，这完全不可行。

**第二步：引入 improvement indicator，绕过归一化**

RECAP 的核心 trick 来自一个不太为人熟知的理论结果。定义 improvement indicator $I$——一个简单的二值标签，"这个动作好不好"：

$$
\begin{aligned}
p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})) &= \delta\!\big(A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}) > \epsilon_{\ell}\big)
\end{aligned}
$$

- $\delta(\cdot)$ 是 delta 函数（见术语速查）：如果 advantage 超过阈值 $\epsilon_{\ell}$，$I=\text{True}$；否则 $I=\text{False}$
- $\epsilon_{\ell}$ 是任务相关的阈值——$\epsilon_{\ell}=0$ 表示"比平均好就算好"；$\epsilon_{\ell}$ 更大表示"只有显著优于平均才算好"，更严格
- 为什么要二值化？因为它把连续值变成了简单的"好/坏"标签，使得后续可以用监督学习的方式处理

如果将策略定义为以这个 improvement indicator 为条件的分布：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}))^{\beta}
\end{aligned}
$$

- 这是将 $\exp(A/\beta)$ 替换为 $p(I|A)^{\beta}$——本质性质不变：**$\hat{\pi}$ 仍然保证比 $\pi_{\text{ref}}$ 更好**（严格的数学证明见论文原文）
- 关键区别：$p(I|A)$ 是二值的（只取 0 或 1），不需要对动作空间积分做归一化

**第三步：利用贝叶斯公式消去显式的 advantage**

对 $p(I|A)$ 应用贝叶斯公式（见术语速查）：

$$
\begin{aligned}
p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})) &= \frac{\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})}{\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})}
\end{aligned}
$$

- 分子 $\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})$：在"$I=\text{True}$，即这是一个好动作"这个条件下的动作概率分布——可以理解为"好动作长什么样"
- 分母 $\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})$：无条件的动作概率分布——"所有动作（不分好坏）长什么样"
- 两个分布的比值天然地反映了"这个动作比平均好多少"，与 advantage 的作用完全等价

代入 $\beta=1$ 的特例（$\beta$ 控制保守程度，设为 1 是最直接的配置）：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot \frac{\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})}{\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})} = \pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})
\end{aligned}
$$

**这就是整个方法最漂亮的结果：最优策略 = 参考策略在"$I=\text{True}$（这是一个好动作）"条件下的条件分布。** advantage 被隐式地编码在条件变量 $I$ 中——模型不需要知道具体的 advantage 数值，只需要知道"现在是不是在生成一个好动作"。

**第四步：退化为监督学习**

这意味着我们只需要训练一个模型同时表示两种分布：无条件（正常行为）和以"这是好动作"为条件（优化后的行为）。训练目标就是两个行为克隆 loss 的加权和：

$$
\begin{aligned}
\min_{\theta} \mathbb{E}_{\mathcal{D}_{\pi_{\text{ref}}}}\Big[&-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell) \\
&- \alpha \cdot \log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)\Big]
\end{aligned}
$$

逐项解释：

- $-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell)$：标准的 behavior cloning loss。让模型学会模仿数据中的所有动作。这一项保证模型不会忘记原始的模仿学习能力——即便我们加了 RL，基础的行为能力不能丢
- $-\alpha \cdot \log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)$：advantage-conditioned loss。当 $I_t=\text{True}$（好动作）时，让模型建立"$I=\text{True}$ → 输出好动作"的映射；当 $I_t=\text{False}$（差动作）时，让模型学会区分"$I=\text{True}$ 时的动作分布"和"$I=\text{False}$ 时的动作分布"的不同。参数 $\alpha$ 控制 RL 优化相对于模仿的权重——$\alpha$ 越大，越激进地倾向优化后的行为
- $I_t = \mathds{1}(A^{\pi_{\text{ref}}}(\mathbf{o}_t, \mathbf{a}_t, \ell) > \epsilon_{\ell})$：这是 advantage 到二值标签的转换。$\mathds{1}$ 是指示函数——条件成立时取 1（True），不成立时取 0（False）
- $\mathcal{D}_{\pi_{\text{ref}}}$：所有历史数据的并集——demonstrations + autonomous rollouts + human interventions。**重点：每个 action 都被保留，不会因为"这是差动作"就被丢弃。** 只是给它加上了 $I_t$ 标签

**推理时**：直接 condition on $I_t=\text{True}$，模型输出的就是经过 RL 优化的策略 $\pi_{\theta}(\mathbf{a}_t|I_t=\text{True}, \mathbf{o}_t, \ell)$。无需额外采样，无需调参。

**为什么比 AWR/PPO 好**：

- **AWR（Advantage-Weighted Regression）**：只对"好"数据做加权模仿学习（权重 $w = \exp(A/\beta)$），本质上丢弃了差数据的训练信号。差数据得到的权重接近 0，等于白收集了。RECAP 保留所有数据——差数据告诉模型"$I_t=\text{False}$ 时输出应该和 $I_t=\text{True}$ 时不同"，这个区分信号和好数据同等重要
- **PPO**：需要在 on-policy 数据上做小步梯度更新（每次更新后数据就"过期"了）。对于 flow-matching 模型，还要用 ELBO 下界（见术语速查）来近似 $\log\pi_\theta$，近似误差层层累积。在真实机器人迭代离线 RL 的设置下（collect → retrain → repeat），PPO 需要极小的 trust-region（$\eta=0.01$，见术语速查）才能勉强稳定，但即便如此性能仍然远不如 RECAP
- **Human corrections 的特殊处理**：人类纠正动作强制设置 $I_t=\text{True}$——假设人类专家总是给出好动作。这相当于把人类纠正视为"最优动作的额外样本"，直接扩充了好动作的分布

### 价值函数

多任务分布式 value function $p_{\phi}(V|\mathbf{o}_t, \ell)$，将观测和语言指令映射到一个在 $B=201$ 个离散 bin 上的价值分布。所谓"分布式"，意思是不只预测一个数值，而是预测 value 落在每个 bin 的概率——这比单值预测更稳健，能捕捉不确定性：

$$
\begin{aligned}
\min_{\phi} \mathbb{E}_{\tau \in \mathcal{D}}\Big[\sum_{\mathbf{o}_t \in \tau} H\!\big(R^B_t(\tau),\; p_{\phi}(V|\mathbf{o}_t, \ell)\big)\Big]
\end{aligned}
$$

- $R_t(\tau) = \sum_{t'=t}^{T} r_{t'}$：从时刻 $t$ 到 episode 结束的实际累积奖励（也叫 return）。如果 episode 成功了，return 大；失败了就小
- $R^B_t(\tau)$：将连续 return 离散化到 201 个 bin 后的 one-hot 标签。每个 bin 对应一个 value 区间
- $H(\cdot, \cdot)$：交叉熵损失——让模型预测的分布 $p_{\phi}(V|\mathbf{o}_t)$ 尽量接近 $R^B_t$ 这个"正确答案"。交叉熵越小 = 两个分布越像

从分布提取连续 value（从概率分布还原成一个数值）：

$$
\begin{aligned}
V^{\pi_{\text{ref}}}(\mathbf{o}_t, \ell) &= \sum_{b \in [0, B]} p_{\phi}(V=b|\mathbf{o}_t) \cdot v(b)
\end{aligned}
$$

- $v(b)$：bin $b$ 中心对应的实际 value 值（归一化到 $[-1, 0]$，0 表示成功）
- 公式就是简单的加权平均：每个 bin 的概率 × 该 bin 的 value，得到期望值

然后用 n-step TD 估计（见术语速查）计算 advantage：

$$
A^{\pi_{\text{ref}}}(\mathbf{o}_t, \mathbf{a}_t) = \underbrace{\sum_{t'=t}^{t+N-1} r_{t'}}_{\text{未来 N 步实际奖励}} + \underbrace{V^{\pi_{\text{ref}}}(\mathbf{o}_{t+N})}_{\text{第 N 步后的估计值}} - \underbrace{V^{\pi_{\text{ref}}}(\mathbf{o}_t)}_{\text{当前状态的估计值}}
$$

- 直觉：如果做完动作后的 N 步奖励 + 估计的未来价值 比 当前的估计价值 高，advantage 就是正的——说明这个动作导向了更好的结果
- $N$ 控制偏差-方差权衡：$N$ 小 → 更依赖 value function 估计（高偏差低方差）；$N$ 大 → 更依赖实际奖励（低偏差高方差）

论文中的可视化显示：value function 能准确识别 episode 中的错误时刻（value 骤降，红色区域）和顺利进展（value 上升，绿色区域）。这种时序上的精确感知是 advantage 计算可靠性的基础。

---

## 模型架构

π₀.₆ 是 π₀.₅ 的升级版，在此基础上 π\*₀.₆ 增加了 advantage conditioning：

| 组件 | 规格 | 说明 |
|------|------|------|
| VLM Backbone | Gemma 3 4B | 负责视觉理解和语言推理，是整个模型的"大脑" |
| Action Expert | 860M params, Flow Matching | 专门的"小脑"，只负责生成动作。用 flow matching 从噪声逐步去噪出平滑的动作轨迹 |
| 输入 | 多相机 + 关节状态 + 语言指令 + $I_t$ | 视觉 + 本体感知 + 任务描述 + 好坏标签 |
| 输出 | Action chunk + 子任务文本 + FAST tokens | 50Hz 关节角/夹爪序列 + "下一步做什么"的描述 + 离散动作 token |
| 训练 | Knowledge Insulation + stop gradient | 保护 VLM 不被动作训练"污染" |

**Knowledge Insulation**（见术语速查）和 **stop gradient** 是关键设计：action expert 的 flow-matching loss 不会反向传播到 VLM backbone。这保证 VLM 的视觉语言理解能力不受动作训练影响——你可以理解为"大脑负责看和想，小脑负责动，两者分工明确但可以对话"。

模型同时输出两种动作表示：连续动作（flow matching 去噪生成的平滑轨迹）和离散 FAST tokens（动作序列压缩成的 token 序列）。两者独立预测，推理时连续动作用于实际控制，离散 token 用于辅助训练。

预训练数据包含数万小时的 demonstrations，覆盖多种任务和多种机器人平台。

---

## 关键结果

**任务**：做意式咖啡、叠各种衣物、组装纸箱——都是长周期、涉及柔性物体/液体的真实部署任务。

| 指标 | 效果 |
|------|------|
| 吞吐量（最难任务） | **2×+ 提升** |
| 失败率 | **~2× 降低** |
| 连续运行 | 咖啡 13 小时、叠衣服 2 小时、工厂纸箱——不间断 |
| vs. PPO | 显著优于 PPO（PPO 需要极小 trust-region $\eta=0.01$ 才稳定，但性能仍远不如 RECAP） |
| vs. AWR | 成功率接近但吞吐量远高于 AWR（AWR 策略更慢、更保守） |
| 消除特定 failure mode | 严格叠衣（领口居中朝上、对抗性初始条件），2 轮 × 600 trajectory → 97% |

---

## 局限

- **非完全自主**：仍需人工标注奖励 + 场景 reset
- **探索策略 naïve**：主要靠 policy 随机性 + human intervention，无结构化探索
- **迭代式离线更新**：collect → retrain → repeat，不是真正的在线 RL。后续 RLT 工作（[Token RL](https://haozhang2027.github.io/posts/rlt-token-rl/)）补充了这一点

---

## 启发

1. **Advantage Conditioning 是大模型 RL 的实用方案**：绕过了 flow-matching/diffusion 模型上 policy gradient 的无解问题。核心是把 RL 退化成两个条件分布之间的监督学习。对任何使用扩散/flow 模型的 VLA 都有借鉴意义。

2. **"用所有数据，标记好坏" 比 "只挑好数据" 更有效**：AWR 丢弃差数据，RECAP 保留所有数据并加标签。差数据告诉模型"什么不要做"——这个信号和好数据同等重要。

3. **Large VLA + Small Value Function 双模型**：VLA 做策略（Gemma 3 4B + 860M action expert），更小的 VLM 做 value function——非对称设计兼顾了策略表达能力和价值估计的稳定性。

4. **Human corrections 是第一等公民**：不把人类干预当作"不得已的补充"，而是作为正式数据源融入 RL 框架——干预动作强制标记 $I_t=\text{True}$。

---

## 论文链接

- arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)
- PI Blog: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)
- 下一篇：[RLT / Token RL](https://haozhang2027.github.io/posts/rlt-token-rl/)
