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

- **Flow-matching/diffusion 模型没有闭式 log-likelihood**：$\pi_\theta(\mathbf{a}|\mathbf{o})$ 是通过迭代去噪过程定义的，无法像高斯分布那样直接算出概率值。PPO/REINFORCE 依赖 log-probability 来计算 policy gradient，因此不能直接套用
- **数据来源异构**：离线 demonstrations + 在线 autonomous rollouts + human corrective interventions，三者的数据分布不同，需要一种能统一利用的方法
- **大规模训练稳定性**：VLA 有 Gemma 3 4B + 860M action expert，传统 on-policy RL 的每次梯度更新就需要新数据，在真实机器人上完全不现实

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

标准 RL 的目标是最大化期望累积奖励 $\mathcal{J}(\pi) = \mathbb{E}_{\tau}[R(\tau)]$。但为了防止策略在离线数据上训崩（over-optimization），需要加正则化项约束新策略不要离参考策略 $\pi_{\text{ref}}$ 太远：

$$
\begin{aligned}
\mathcal{J}(\pi, \pi_{\text{ref}}) &= \mathbb{E}_{\tau \sim \rho_{\pi}}\Big[\sum_{t=0}^{T} r_t\Big] - \beta \cdot \mathbb{E}_{\mathbf{o} \sim \rho_{\pi}}\Big[D_{\text{KL}}(\pi(\cdot|\mathbf{o}) \parallel \pi_{\text{ref}}(\cdot|\mathbf{o}))\Big]
\end{aligned}
$$

- 第一项 $\mathbb{E}_{\tau}[\sum r_t]$：期望总奖励，越大越好
- 第二项 $\beta \cdot D_{\text{KL}}$：KL 散度惩罚，$\beta$ 控制约束强度——$\beta$ 越大，新策略越接近参考策略，越保守

这个正则化问题有闭式最优解：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot \exp\!\big(A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}) / \beta\big)
\end{aligned}
$$

- $A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})$ 是 advantage function：在当前状态 $\mathbf{o}$ 下，做动作 $\mathbf{a}$ 比"平均"好多少
- $\exp(A/\beta)$：advantage 越大，该动作在最优策略中的权重越大

但这个形式要求能显式计算 $\exp(A/\beta)$ 并做归一化（即对动作空间积分），在高维连续动作空间中不可行。

**第二步：引入 improvement indicator，绕过归一化**

RECAP 的核心 trick 来自一个不太为人熟知的理论结果。定义一个二值 improvement indicator $I$：

$$
\begin{aligned}
p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})) &= \delta\!\big(A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}) > \epsilon_{\ell}\big)
\end{aligned}
$$

- $\delta(\cdot)$ 是 delta 函数：如果 advantage 超过阈值 $\epsilon_{\ell}$，$I=\text{True}$（这是一个好动作）；否则 $I=\text{False}$
- $\epsilon_{\ell}$ 是任务相关的阈值，控制"好"的标准有多严格

如果将策略定义为以这个 improvement indicator 为条件的分布：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a}))^{\beta}
\end{aligned}
$$

- 这是将 $\exp(A/\beta)$ 替换为 $p(I|A)^{\beta}$，核心性质不变：**$\hat{\pi}$ 仍然保证比 $\pi_{\text{ref}}$ 更好**（定理证明见论文）
- 关键区别：$p(I|A)$ 是二值的（0 或 1），不需要对动作空间积分

**第三步：利用贝叶斯公式消去显式的 advantage**

对 $p(I|A)$ 应用贝叶斯公式：

$$
\begin{aligned}
p(I|A^{\pi_{\text{ref}}}(\mathbf{o}, \mathbf{a})) &= \frac{\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})}{\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})}
\end{aligned}
$$

- 分子 $\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})$：在"这是一个好动作"条件下的动作分布
- 分母 $\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})$：无条件的动作分布

代入 $\beta=1$ 的特例：

$$
\begin{aligned}
\hat{\pi}(\mathbf{a}|\mathbf{o}) &\propto \pi_{\text{ref}}(\mathbf{a}|\mathbf{o}) \cdot \frac{\pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})}{\pi_{\text{ref}}(\mathbf{a}|\mathbf{o})} = \pi_{\text{ref}}(\mathbf{a}|I, \mathbf{o})
\end{aligned}
$$

**这就是整个方法最漂亮的结果：最优策略 = 参考策略在"I=True（这是一个好动作）"条件下的条件分布。** advantage 被隐式地编码在条件变量 $I$ 中，策略不再需要显式地计算 advantage。

**第四步：退化为监督学习**

这意味着我们只需要训练一个模型同时表示两种分布：无条件 $\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t, \ell)$ 和以 advantage 为条件 $\pi_{\theta}(\mathbf{a}_t|I_t, \mathbf{o}_t, \ell)$。训练目标：

$$
\begin{aligned}
\min_{\theta} \mathbb{E}_{\mathcal{D}_{\pi_{\text{ref}}}}\Big[&-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell) \\
&- \alpha \cdot \log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)\Big]
\end{aligned}
$$

逐项解释：

- $-\log\pi_{\theta}(\mathbf{a}_t|\mathbf{o}_t,\ell)$：标准的 behavior cloning loss——让模型学会模仿数据中的动作。这一项保证模型不会忘记原始的模仿学习能力
- $-\alpha \cdot \log\pi_{\theta}(\mathbf{a}_t|I_t,\mathbf{o}_t,\ell)$：advantage-conditioned loss——当 $I_t=\text{True}$（好动作）时，让模型学会"这是一个好动作条件下的动作分布"；当 $I_t=\text{False}$（差动作）时，让模型学会区分好坏。参数 $\alpha$ 控制 RL 优化相对于模仿的权重
- $I_t = \mathds{1}(A^{\pi_{\text{ref}}}(\mathbf{o}_t, \mathbf{a}_t, \ell) > \epsilon_{\ell})$：二值化 advantage indicator
- $\mathcal{D}_{\pi_{\text{ref}}}$：所有历史数据的并集——demonstrations + autonomous rollouts + human interventions。每个 action 都被保留，只是加上了 $I_t$ 标签

**推理时**：直接 condition on $I_t=\text{True}$，模型输出的就是经过 RL 优化的策略 $\pi_{\theta}(\mathbf{a}_t|I_t=\text{True}, \mathbf{o}_t, \ell)$。无需额外采样，无需 CFG 调参。

**为什么比 AWR/PPO 好**：

- **AWR（Advantage-Weighted Regression）**：只对"好"数据做加权模仿学习（$w = \exp(A/\beta)$），本质上丢弃了差数据的训练信号。RECAP 保留所有数据，让模型自己从 $I_t$ 标签中学到好坏的区别——差数据告诉模型"什么不要做"，同等重要
- **PPO**：需要在 on-policy 数据上做小步梯度更新，对于 flow-matching 模型还需要用 ELBO 下界近似 log-probability。在真实机器人迭代离线 RL 的设置下（collect → retrain → repeat），极端不稳定
- **Human corrections 的特殊处理**：人类纠正动作强制设置 $I_t=\text{True}$——假设人类专家总是给出好动作，这是合理的

### 价值函数

多任务分布式 value function $p_{\phi}(V|\mathbf{o}_t, \ell)$，将观测和语言指令映射到一个在 201 个离散 bin 上的价值分布：

$$
\begin{aligned}
\min_{\phi} \mathbb{E}_{\tau \in \mathcal{D}}\Big[\sum_{\mathbf{o}_t \in \tau} H\!\big(R^B_t(\tau),\; p_{\phi}(V|\mathbf{o}_t, \ell)\big)\Big]
\end{aligned}
$$

- $R_t(\tau) = \sum_{t'=t}^{T} r_{t'}$：从时刻 $t$ 到 episode 结束的实际累积奖励
- $R^B_t(\tau)$：将连续 return 离散化到 $B=201$ 个 bin
- $H(\cdot, \cdot)$：交叉熵损失——让预测分布匹配实际 return 分布

从分布提取连续 value：

$$
\begin{aligned}
V^{\pi_{\text{ref}}}(\mathbf{o}_t, \ell) &= \sum_{b \in [0, B]} p_{\phi}(V=b|\mathbf{o}_t) \cdot v(b)
\end{aligned}
$$

- $v(b)$：bin $b$ 对应的实际 value 值
- 这是期望值计算：每个 bin 的概率 × 该 bin 的 value

然后用 n-step TD 估计计算 advantage：$A^{\pi_{\text{ref}}}(\mathbf{o}_t, \mathbf{a}_t) = \sum_{t'=t}^{t+N-1} r_{t'} + V^{\pi_{\text{ref}}}(\mathbf{o}_{t+N}) - V^{\pi_{\text{ref}}}(\mathbf{o}_t)$。

论文中的可视化显示：value function 能准确识别 episode 中的错误时刻（value 骤降，红色）和顺利进展（value 上升，绿色）——这就是 advantage 计算的基础。

---

## 模型架构

π₀.₆ 是 π₀.₅ 的升级版，在此基础上 π\*₀.₆ 增加了 advantage conditioning：

| 组件 | 规格 |
|------|------|
| VLM Backbone | Gemma 3 4B |
| Action Expert | 860M params, Flow Matching |
| 输入 | 多相机图像 + 机器人关节状态 + 语言指令 + **二值化 advantage indicator $I_t$** |
| 输出 | Action chunk（50Hz 关节角 + 夹爪）+ 子任务文本预测 + FAST tokenized actions |
| 训练 | Knowledge Insulation (KI) + stop gradient |

**Knowledge Insulation** 是关键设计：action expert 通过 stop gradient 与 VLM 主干隔离。这意味着 flow-matching 的 loss 不会反向传播到 VLM backbone——VLM 的视觉语言理解能力得到保护，只有 action expert 专门学习动作生成。

模型同时输出连续 action（flow matching 去噪过程）和离散 tokenized action（FAST tokenizer），两者独立预测但在推理时可以互补。

预训练数据包含数万小时的 demonstrations，覆盖多种任务和多种机器人平台。

---

## 关键结果

**任务**：做意式咖啡、叠各种衣物、组装纸箱——都是长周期、涉及柔性物体/液体的真实部署任务。咖啡任务需要倒液体，叠衣服需要处理各种形状的柔性织物，装纸箱涉及多步骤操作。

| 指标 | 效果 |
|------|------|
| 吞吐量（最难任务） | **2×+ 提升** |
| 失败率 | **~2× 降低** |
| 连续运行 | 咖啡 13 小时、叠衣服 2 小时、工厂纸箱组装——不间断 |
| vs. PPO | 显著优于 PPO（PPO 在 off-policy 设置下需要极小的 trust-region $\eta=0.01$ 才能稳定，但仍无法获得好性能） |
| vs. AWR | 成功率接近但吞吐量远高于 AWR（AWR 学出的策略更慢、更保守） |
| 消除特定 failure mode | 严格叠衣任务（领口居中朝上、对抗性初始条件），2 轮迭代（每轮 600 trajectory）→ 97% 成功率 |

---

## 局限

- **非完全自主**：仍需人工标注奖励 + 场景 reset。论文指出 VLA 的高层推理能力（子任务预测 $\hat{\ell}$）有望用于自动 reset，但尚未实现
- **探索策略 naïve**：主要靠 policy 随机性 + human intervention，无结构化探索（如 intrinsic motivation、curiosity）。在初始策略已经不错时够用，但对需要发现全新行为模式的任务不足
- **迭代式离线更新**：collect → retrain → repeat，不是真正的在线 RL。后续的 RLT 工作（[Token RL](https://haozhang2027.github.io/posts/rlt-token-rl/)）补充了这一点

---

## 启发

1. **Advantage Conditioning 是大模型 RL 的实用方案**：绕过了 flow-matching/diffusion 模型上 policy gradient 的无解问题（没有闭式 log-likelihood）。核心是把 RL 退化成两个条件分布之间的监督学习——$I=\text{True}$ 和 $I=\text{False}$ 的区别。对任何使用扩散/flow 模型的 VLA 都有借鉴意义。

2. **"用所有数据，标记好坏" 比 "只挑好数据" 更有效**：AWR 本质是 filtered imitation——丢弃或降权大量数据。RECAP 保留所有数据，只是用 advantage 标记好坏。差数据告诉模型"什么不要做"，这个信号和好数据同等重要。

3. **Large VLA + Small Value Function 双模型**：VLA 做策略（Gemma 3 4B + 860M action expert），更小的 VLM 做 value function——非对称设计兼顾了策略表达能力和价值估计的稳定性。

4. **Human corrections 是第一等公民**：RECAP 不把人类干预当作"不得已的补充"，而是把它们作为正式的数据源融入 RL 框架——干预动作强制标记 $I_t=\text{True}$，直接提升策略。

---

## 论文链接

- arXiv: [2511.14759](https://arxiv.org/abs/2511.14759)
- PI Blog: [pi.website/blog/pistar06](https://pi.website/blog/pistar06)
- 下一篇：[RLT / Token RL](https://haozhang2027.github.io/posts/rlt-token-rl/)
