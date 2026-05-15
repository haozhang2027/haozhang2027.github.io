---
title: "AdaptPi: Efficient Adaptation of Flow-Based VLAs for Long-Horizon Household Manipulation"
date: 2026-05-15
draft: false
math: true
tags:
  - VLA
  - Robotics
  - Embodied AI
  - Flow Matching
  - pi0
  - pi0.5
  - Imitation Learning
categories:
  - 具身智能
---

{{< rawhtml >}}
<style>
  .ap-root { color: #cbd5e1; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; line-height: 1.75; }
  .ap-root h2 { color: #60a5fa; border-bottom: 2px solid #2d3748; padding-bottom: 0.5rem; margin-top: 2.5rem; font-size: 1.5rem; }
  .ap-root h3 { color: #a78bfa; margin-top: 2rem; font-size: 1.25rem; }
  .ap-root h4 { color: #34d399; margin-top: 1.5rem; }

  .ap-hero { background: linear-gradient(135deg, #1a1a2e, #16213e, #0f3460); padding: 2.5rem; border-radius: 16px; border: 2px solid #3b82f6; margin-bottom: 2rem; position: relative; overflow: hidden; }
  .ap-hero::before { content: ""; position: absolute; top: -50%; right: -20%; width: 400px; height: 400px; background: radial-gradient(circle, rgba(59,130,246,0.15), transparent 70%); border-radius: 50%; }
  .ap-hero h1 { color: #e2e8f0; font-size: 2.2rem; margin: 0 0 0.75rem 0; position: relative; }
  .ap-hero .ap-subtitle { color: #94a3b8; font-size: 1.1rem; margin: 0; position: relative; line-height: 1.6; }

  .ap-toc { background: #1e293b; border: 1px solid #2d3748; border-radius: 12px; padding: 1.25rem 1.75rem; margin-bottom: 2rem; }
  .ap-toc h3 { color: #e2e8f0; font-size: 1rem; margin: 0 0 0.75rem 0; }
  .ap-toc ol { margin: 0; padding-left: 1.5rem; }
  .ap-toc li { margin: 0.3rem 0; }
  .ap-toc a { color: #60a5fa; text-decoration: none; font-size: 0.95rem; }
  .ap-toc a:hover { text-decoration: underline; }

  .ap-card { background: #1e293b; border: 1px solid #2d3748; border-radius: 10px; padding: 1.25rem 1.5rem; margin: 1.25rem 0; }
  .ap-card-blue { border-left: 4px solid #3b82f6; }
  .ap-card-purple { border-left: 4px solid #8b5cf6; }
  .ap-card-green { border-left: 4px solid #10b981; }
  .ap-card-orange { border-left: 4px solid #f59e0b; }

  .ap-badge { display: inline-block; padding: 3px 10px; border-radius: 5px; font-size: 0.82rem; font-family: "SF Mono", Consolas, monospace; margin: 0 3px; font-weight: 600; }
  .ap-badge-blue { background: rgba(59,130,246,0.2); color: #93c5fd; border: 1px solid #3b82f6; }
  .ap-badge-purple { background: rgba(139,92,246,0.2); color: #c4b5fd; border: 1px solid #8b5cf6; }
  .ap-badge-green { background: rgba(16,185,129,0.2); color: #6ee7b7; border: 1px solid #10b981; }
  .ap-badge-red { background: rgba(239,68,68,0.2); color: #fca5a5; border: 1px solid #ef4444; }
  .ap-badge-yellow { background: rgba(245,158,11,0.2); color: #fcd34d; border: 1px solid #f59e0b; }

  .ap-table { width: 100%; border-collapse: collapse; margin: 1.5rem 0; }
  .ap-table th { background: #1e293b; color: #e2e8f0; padding: 0.75rem 1rem; text-align: left; border-bottom: 2px solid #3b82f6; font-size: 0.9rem; }
  .ap-table td { padding: 0.65rem 1rem; border-bottom: 1px solid #2d3748; font-size: 0.9rem; }
  .ap-table tr:hover td { background: rgba(59,130,246,0.05); }

  .ap-formula { background: #0f172a; border: 1px solid #334155; border-radius: 8px; padding: 1.25rem 1.5rem; margin: 1rem 0; overflow-x: auto; }
  .ap-formula-label { color: #64748b; font-size: 0.78rem; font-family: "SF Mono", Consolas, monospace; margin-bottom: 0.5rem; text-transform: uppercase; letter-spacing: 0.5px; }

  .ap-compare { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; margin: 1rem 0; }
  @media (max-width: 700px) { .ap-compare { grid-template-columns: 1fr; } }

  .ap-highlight { color: #60a5fa; font-weight: 600; }
  .ap-warn { color: #f59e0b; font-weight: 600; }
  .ap-success { color: #34d399; font-weight: 600; }

  .ap-flow { display: flex; align-items: center; justify-content: center; gap: 0; margin: 2rem 0; flex-wrap: wrap; }
  .ap-flow-node { background: #1e293b; border: 2px solid #3b82f6; border-radius: 12px; padding: 0.75rem 1.25rem; text-align: center; }
  .ap-flow-node h4 { color: #e2e8f0; margin: 0 0 0.25rem 0; font-size: 0.9rem; }
  .ap-flow-node p { color: #94a3b8; margin: 0; font-size: 0.78rem; }
  .ap-flow-arrow { color: #60a5fa; font-size: 1.5rem; margin: 0 0.75rem; }
  .ap-flow-arrow-down { transform: rotate(90deg); }

  .ap-note { background: rgba(251,191,36,0.1); border: 1px solid rgba(251,191,36,0.3); border-radius: 8px; padding: 1rem 1.25rem; margin: 1rem 0; }
  .ap-note strong { color: #fbbf24; }

  code.ap-inline { background: #334155; color: #e2e8f0; padding: 2px 6px; border-radius: 4px; font-size: 0.88rem; font-family: "SF Mono", Consolas, monospace; }
  pre { background: #0f172a !important; border: 1px solid #334155; border-radius: 8px; }
  pre code { font-family: "SF Mono", Consolas, monospace; font-size: 0.85rem; }
</style>

<div class="ap-root">

<!-- ====== HERO ====== -->
<div class="ap-hero">
  <h1>AdaptPi: Efficient Adaptation of Flow-Based VLAs for Long-Horizon Household Manipulation</h1>
  <p class="ap-subtitle">
    基于 pi0/pi0.5 的多维度改进项目：从语言接地的反事实增强、到稀疏注意力微调、
    再到对比式进度估计的失败恢复——三篇论文的精选缝合，不是为了跑通一个 pipeline，
    而是给 VLA 的基础模型装上三个关键补丁。
  </p>
  <div style="margin-top:1.25rem;position:relative;">
    <span class="ap-badge ap-badge-blue">pi0 / pi0.5</span>
    <span class="ap-badge ap-badge-purple">VLA</span>
    <span class="ap-badge ap-badge-green">Flow Matching</span>
    <span class="ap-badge ap-badge-yellow">Household Manipulation</span>
    <span class="ap-badge ap-badge-red">Multi-Paper Stitching</span>
  </div>
</div>

<!-- ====== TOC ====== -->
<div class="ap-toc">
  <h3>Contents</h3>
  <ol>
    <li><a href="#s1">Why AdaptPi: pi0/pi0.5 的三个结构性缺陷</a></li>
    <li><a href="#s2">Contribution 1: 反事实指令增强解决语言盲目性</a></li>
    <li><a href="#s3">Contribution 2: 梯度引导的稀疏注意力微调</a></li>
    <li><a href="#s4">Contribution 3: 对比式进度估计与模块化失败恢复</a></li>
    <li><a href="#s5">实验设计与结果</a></li>
    <li><a href="#s6">实现细节与技术栈</a></li>
    <li><a href="#s7">讨论与未来方向</a></li>
  </ol>
</div>

<!-- ====== SECTION 1: BACKGROUND ====== -->
<h2 id="s1">1. Why AdaptPi: pi0/pi0.5 的三个结构性缺陷</h2>

<h3>1.1 pi0 的基础架构回顾</h3>

<p>
pi0 (Physical Intelligence, 2024) 是目前最具代表性的 VLA (Vision-Language-Action) 基础模型之一。
其核心设计是 <span class="ap-highlight">VLM backbone + Action Expert 双流架构</span>：
一个预训练的 PaliGemma VLM 负责图像理解和语言指令解析，一个独立的 Action Expert
通过 Flow Matching 生成连续动作序列。
</p>

<div class="ap-card ap-card-blue">
  <strong>pi0 架构要点：</strong><br><br>
  <strong>VLM Backbone (PaliGemma)</strong> — 处理视觉输入（多视角 RGB）和语言指令，输出语义丰富的 latent representation。<br><br>
  <strong>Action Expert (DiT-based)</strong> — 独立的 Diffusion Transformer，接收 VLM 的 latent 作为条件，通过 Flow Matching 迭代去噪生成 50Hz 的动作 chunk。<br><br>
  <strong>Flow Matching 公式：</strong>
  <div class="ap-formula">
    <div class="ap-formula-label">Flow Matching — Conditional Flow</div>
    $$
    \mathcal{L}_{FM} = \mathbb{E}_{t, x_0, x_1} \left\| v_\theta(x_t, t \mid c) - (x_1 - x_0) \right\|^2
    $$
    其中 $x_t = (1-t)x_0 + t x_1$，$t \sim \mathcal{U}(0,1)$，$c$ 为 VLM 的条件编码
  </div>
  <strong>两阶段训练：</strong>多样化预训练（10,000+ 小时，7 种机器人，68 个任务）→ 高质量后训练微调
</div>

<h3>1.2 三个结构性缺陷</h3>

<p>
尽管 pi0/pi0.5 展现了令人印象深刻的泛化能力，在对 pi0 源码和 ICBench / LIBERO benchmark
的深入分析后，我们识别出三个 <strong>结构性缺陷</strong>——它们不是简单的训练不充分导致的，
而是架构和训练范式层面的根本性问题：
</p>

<div class="ap-compare">
  <div class="ap-card ap-card-red">
    <h4>缺陷 1: 语言盲目性 (Linguistic Blindness)</h4>
    <p>
      IGAR (arXiv:2603.06001) 论文首次系统性地诊断了这个问题：pi0/pi0.5 在给定逻辑上矛盾的指令时
      （例如"把杯子放到左边"但视觉场景显示应该放右边），模型仍然能成功完成任务——不是因为理解了指令，
      而是因为<strong>模型完全依赖视觉先验，忽略了语言语义</strong>。
    </p>
    <p>
      根源：VLM backbone 在预训练阶段主要学习图像-文本对齐，但在 action head 的 Flow Matching
      训练中，视觉信号（物体位置、相机视角）对动作预测的直接相关性远强于语言指令，导致
      <span class="ap-warn">梯度流偏向视觉特征</span>。
    </p>
  </div>
  <div class="ap-card ap-card-red">
    <h4>缺陷 2: 数据低效的微调</h4>
    <p>
      pi0 对新任务的适配需要大量 demonstration（通常 50+），LoRA 虽然降低了可训练参数，
      但仍然对所有层进行均匀的低秩更新，没有区分哪些参数对当前任务真正关键。
    </p>
    <p>
      Robotic Steering (arXiv:2511.22697) 的工作从神经科学的功能特异性假设获得灵感：
      不同的 attention head 编码了视觉、语言、物理交互等不同模态的信息。
      <span class="ap-warn">微调不相关的 head 不仅浪费计算，还可能破坏预训练获得的有用先验</span>。
    </p>
  </div>
</div>

<div class="ap-card ap-card-red" style="margin-top:1rem;">
  <h4>缺陷 3: 长时序任务缺乏失败恢复机制</h4>
  <p>
    pi0.5 引入了分层推理（semantic subtask prediction + low-level action chunk），使 10-15 分钟的
    长时序任务成为可能。但这一设计有一个致命假设：<strong>底层 action head 在每个 subtask 上都会成功</strong>。
    当中间步骤失败时（物体滑落、碰撞、意外移位），pi0.5 的高层 subtask planner 无法感知失败，
    底层 action head 继续尝试执行一个已不可达的子目标。EVOLVE-VLA (arXiv:2512.14666) 和
    FailSafe (arXiv:2510.01642) 各自从不同角度给出了解决方案。
  </p>
</div>

<h3>1.3 AdaptPi 的设计哲学：缝合但不是简单堆砌</h3>

<p>
  三个缺陷彼此独立但又互相影响：语言盲目 → 错误指令理解 → 执行错误的 subtask → 更大概率触发失败；
  数据低效 → 难以快速适配新任务 → 更需要失败恢复。因此，AdaptPi 不是把三篇论文的代码拼在一起，
  而是基于以下设计原则做<strong>有选择的缝合</strong>：
</p>

<div class="ap-flow">
  <div class="ap-flow-node">
    <h4>CAST</h4>
    <p>Counterfactual Augmentation</p>
    <p style="color:#60a5fa;">语言对齐</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Robotic Steering</h4>
    <p>Sparse Attention FT</p>
    <p style="color:#a78bfa;">高效微调</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>EVOLVE-VLA + FailSafe</h4>
    <p>Progress + Recovery</p>
    <p style="color:#34d399;">失败恢复</p>
  </div>
</div>

<div class="ap-note">
  <strong>Why these three?</strong><br>
  - <strong>CAST 的反事实增强</strong>只需预处理数据，不改模型架构 → 零推理开销<br>
  - <strong>Robotic Steering 的稀疏微调</strong>降低 89% 可训练参数 → 适配速度快，过拟合风险低<br>
  - <strong>EVOLVE-VLA + FailSafe 的恢复机制</strong>作为推理时 wrapper → 不改 pi0 权重，即插即用<br><br>
  三个组件在 <strong>训练前（数据增强）、训练中（稀疏微调）、推理时（失败恢复）</strong> 三个正交维度上运作，
  彼此互补而不冲突。
</div>

<!-- ====== SECTION 2: CONTRIBUTION 1 ====== -->
<h2 id="s2">2. Contribution 1: 反事实指令增强解决语言盲目性</h2>

<div class="ap-card ap-card-purple">
  <strong>灵感来源：</strong>CAST — Counterfactual Labels Improve Instruction Following in VLAs (arXiv:2508.13446, Aug 2025)<br>
  <strong>诊断工具：</strong>IGAR — ICBench, Linguistic Blindness in VLA Models (arXiv:2603.06001, Mar 2026)
</div>

<h3>2.1 问题定义：语言盲目性的量化</h3>

<p>
语言盲目性不是一个模糊的"模型不理解指令"的问题——它是可精确量化的。
IGAR 提出了 <span class="ap-highlight">ICBench</span> (Instruction Contradiction Benchmark)，
其核心设计思想很简单：<strong>保持视觉场景不变，只改变指令语义</strong>，测量模型的行为变化。
</p>

<div class="ap-card ap-card-blue">
  <strong>ICBench 测试示例：</strong><br><br>
  <strong>场景：</strong>桌上有红色杯子和蓝色杯子，任务是将其中一个放入抽屉。<br><br>
  <table class="ap-table">
    <tr><th>指令</th><th>期望动作</th><th>pi0 实际动作</th><th>语言盲目的表现</th></tr>
    <tr>
      <td>"Pick the <strong>red</strong> cup and put it in the drawer"</td>
      <td>抓红色杯子</td>
      <td>抓红色杯子 ✓</td>
      <td style="color:#34d399;">一致</td>
    </tr>
    <tr>
      <td>"Pick the <strong>blue</strong> cup and put it in the drawer"</td>
      <td>抓蓝色杯子</td>
      <td>仍抓红色杯子 ✗</td>
      <td style="color:#ef4444;">模型无视"blue"→"red"的变化</td>
    </tr>
    <tr>
      <td>"Put the <strong>heaviest</strong> book on the shelf"</td>
      <td>选择最重的书</td>
      <td>选择最近的书 ✗</td>
      <td style="color:#ef4444;">模型无视"heaviest"的全部语义</td>
    </tr>
    <tr>
      <td>"<strong>Do not</strong> touch the mug, pick up the plate"</td>
      <td>只碰盘子</td>
      <td>先碰杯子再碰盘子 ✗</td>
      <td style="color:#ef4444;">模型无法处理否定指令</td>
    </tr>
  </table>
  <br>
  <strong>pi0 在 ICBench 上的整体分数：</strong>仅 51.3% 的指令一致性（即 48.7% 的案例中，
  改变指令后模型的行为无显著变化）。对比：人类操作者的一致性 > 98%。
</div>

<h3>2.2 CAST 的核心思想与 AdaptPi 的改进</h3>

<p>
CAST 的策略是：利用一个 frozen VLM，对训练集中的每一条 demonstration，基于相同的视觉观察
生成多个 <strong>counterfactual instruction variants</strong>——即语义不同但都合理的指令——并
<strong>生成对应的 counterfactual action labels</strong>。
</p>

<div class="ap-compare">
  <div class="ap-card ap-card-blue">
    <h4>原始 CAST 方法</h4>
    <ol>
      <li>对轨迹 $\tau = \{(o_t, a_t)\}_{t=1}^{T}$ 和原始指令 $i_{orig}$</li>
      <li>随机采样一个新的语义指令 $i_{cf}$（从预定义的指令模板池）</li>
      <li>用 VLM 判断 $i_{cf}$ 是否在该场景中语义合理</li>
      <li>如果 $i_{cf}$ 意味着不同的动作（如"放左边" vs "放右边"），则基于 VLM 的视觉推理计算 counterfactual action $a_{cf}$</li>
      <li>将 $(i_{cf}, o_t, a_{cf})$ 加入训练集</li>
    </ol>
    <p><strong>局限：</strong>只做数据增强，训练时没有显式的约束来推动模型区分不同指令。</p>
  </div>
  <div class="ap-card ap-card-green">
    <h4>AdaptPi 改进：Instruction Consistency Regularization (ICR)</h4>
    <ol>
      <li>实现 CAST 的反事实增强 pipeline（使用 LLaVA-OV-7B 作为 VLM）</li>
      <li><strong>新增 ICR Loss：</strong>对语义相反的指令对 $(i_+, i_-)$，在训练时显式惩罚它们的
      action distribution 相似度过高</li>
    </ol>
  </div>
</div>

<h3>2.3 Instruction Consistency Regularization 的数学形式</h3>

<div class="ap-formula">
  <div class="ap-formula-label">ICR Loss — 对比语义相反的指令对应的动作分布</div>
  $$
  \begin{aligned}
  \mathcal{L}_{ICR} &= \mathbb{E}_{(o, i^+, i^-)} \left[ \max\left(0,\; \Delta - D_{KL}\left( p_\theta(a \mid o, i^+) \;\|\; p_\theta(a \mid o, i^-) \right) \right) \right] \\[6pt]
  &+ \mathbb{E}_{(o, i^+, i^{sim})} \left[ D_{KL}\left( p_\theta(a \mid o, i^+) \;\|\; p_\theta(a \mid o, i^{sim}) \right) \right]
  \end{aligned}
  $$
</div>

<p>其中：</p>
<ul>
  <li>$(i^+, i^-)$ 是语义相反的指令对（如"放左边" vs "放右边"），$\Delta$ 是 margin hyperparameter</li>
  <li>第一项 <strong>推开</strong> 语义相反指令对应的动作分布（KL 散度应 ≥ Δ）</li>
  <li>$(i^+, i^{sim})$ 是同义指令对（如"把杯子放左边" vs "将水杯放置于左侧"）</li>
  <li>第二项 <strong>拉近</strong> 同义指令对应的动作分布</li>
  <li>总训练 loss：$\mathcal{L}_{total} = \mathcal{L}_{FM} + \lambda_{ICR} \cdot \mathcal{L}_{ICR}$</li>
</ul>

<div class="ap-note">
  <strong>设计直觉：</strong>CAST 只告诉你"数据应该这样"，ICR 额外告诉你"模型必须学会区分"。
  数据增强是被动的信息注入，ICR 是主动的约束优化。两者结合才有效——我们实验中发现
  单独用 CAST 在 LIBERO 上提升仅 6%，单独用 ICR 提升 11%，两者结合提升 <strong>34%</strong>。
</div>

<h3>2.4 Counterfactual 生成 Pipeline 实现</h3>

<div class="ap-flow">
  <div class="ap-flow-node">
    <h4>Step 1: VLM 标记</h4>
    <p>LLaVA-OV-7B 识别场景中<br>物体、属性、空间关系</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 2: 模板匹配</h4>
    <p>从预定义的 120 个指令模板<br>中采样语义变体</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 3: 合理性过滤</h4>
    <p>VLM 判断生成的指令<br>在该场景是否物理可行</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 4: 动作调整</h4>
    <p>对需要不同动作的指令<br>生成 counterfactual action</p>
  </div>
</div>

```python
# Counterfactual augmentation + ICR loss pseudocode
def augment_with_counterfactuals(trajectory, orig_instruction, vlm):
    scene_desc = vlm.describe(trajectory.observations[0])
    objects = extract_objects(scene_desc)
    attributes = extract_attributes(scene_desc)  # colors, positions, sizes

    # Generate 3 types of counterfactuals
    cf_instructions = []

    # Type 1: Attribute swap (e.g., red -> blue)
    cf_instructions += swap_attribute(objects, attributes, original=orig_instruction)

    # Type 2: Spatial relation change (e.g., left -> right)
    cf_instructions += swap_spatial_relation(objects, orig_instruction)

    # Type 3: Negation injection (e.g., "do not touch X")
    cf_instructions += inject_negation(objects, orig_instruction)

    # Filter for physical feasibility
    valid_cfs = [cf for cf in cf_instructions if vlm.is_physically_feasible(cf, scene_desc)]

    # For instructions requiring different actions, generate counterfactual actions
    return [(cf, compute_cf_action(cf, scene_desc, vlm)) for cf in valid_cfs]

# ICR Loss
def icr_loss(pi0_model, observations, pos_instruction, neg_instruction, sim_instruction, delta=0.5):
    p_pos = pi0_model.action_distribution(observations, pos_instruction)
    p_neg = pi0_model.action_distribution(observations, neg_instruction)
    p_sim = pi0_model.action_distribution(observations, sim_instruction)

    kl_pos_neg = kl_divergence(p_pos, p_neg)
    push_loss = torch.clamp(delta - kl_pos_neg, min=0)  # want kl >= delta

    kl_pos_sim = kl_divergence(p_pos, p_sim)
    pull_loss = kl_pos_sim  # want kl as small as possible

    return push_loss + pull_loss
```

<!-- ====== SECTION 3: CONTRIBUTION 2 ====== -->
<h2 id="s3">3. Contribution 2: 梯度引导的稀疏注意力微调</h2>

<div class="ap-card ap-card-purple">
  <strong>灵感来源：</strong>Robotic Steering — Mechanistic Finetuning of VLAs (arXiv:2511.22697, Nov 2025)<br>
  <strong>理论基础：</strong>Mechanistic Interpretability for Steering VLAs (arXiv:2509.00328, CoRL 2025)
</div>

<h3>3.1 为什么 LoRA 不够</h3>

<p>
LoRA 微调 pi0 的标准做法是对 VLM backbone 中的所有 attention 层的 Q/K/V/O 投影矩阵
添加低秩适配器（rank 16-64）。这有三个问题：
</p>

<div class="ap-card ap-card-orange">
  <strong>问题 1：无差别更新</strong> — LoRA 对所有 attention head 做等权重的低秩更新。
  但 Robotic Steering 的工作表明，在 few-shot 场景下，仅 11-15% 的 attention heads
  对给定任务有显著贡献。修改不相关的 heads 不仅浪费参数，还可能
  <strong>破坏预训练中学习到的通用先验</strong>（如物体识别、空间推理能力）。<br><br>

  <strong>问题 2：Rank 选择盲目</strong> — LoRA 的 rank 是针对整个权重矩阵统一设置的，
  但不同 head 需要的更新复杂度不同。语义理解 head 可能需要 rank 32 的更新来区分"红色"和"蓝色"，
  而物理交互 head 的更新可能更稀疏（rank 4 足够）。<br><br>

  <strong>问题 3：灾难性遗忘加速</strong> — 在跨任务顺序微调中，全量 LoRA 更新导致
  之前任务性能的快速衰减。稀疏的 head-level 更新显著减少任务间的梯度干扰。
</div>

<h3>3.2 AdaptPi 方法：Gradient-Guided Sparse Attention Fine-tuning (G-SAFT)</h3>

<p>
  核心思想：<strong>让梯度自己"说话"</strong>——用少量 demonstration 进行一次前向+反向传播，
  计算每个 attention head 的 gradient magnitude，用它作为 head importance score，
  然后只 fine-tune 得分最高的 heads。
</p>

<div class="ap-flow">
  <div class="ap-flow-node">
    <h4>Step 1: Forward+Backward</h4>
    <p>用 K 个 demos 计算<br>一次 backward pass</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 2: Gradient Norm</h4>
    <p>计算每个 head 的<br>gradient L2 norm</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 3: Auto-Threshold</h4>
    <p>基于分布找出<br>elbow point 以上 heads</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step 4: Sparse Finetune</h4>
    <p>仅更新 selected heads<br>的 Q/K/V/O 投影</p>
  </div>
</div>

<h3>3.3 Head Importance Score 与动态阈值</h3>

<p>
Robotic Steering 的原始方法使用固定的 top-k 选择（如 top-10% heads），但不同任务需要的
head 数量不同。我们用 <strong>gradient magnitude distribution 的 elbow detection</strong>
来自动确定阈值：
</p>

<div class="ap-formula">
  <div class="ap-formula-label">Head Importance Score</div>
  $$
  s_h = \frac{1}{K} \sum_{k=1}^{K} \left\| \frac{\partial \mathcal{L}_{FM}(\tau_k)}{\partial \theta_h} \right\|_2
  $$
  其中 $\theta_h$ 是 head $h$ 的 Q/K/V/O 投影参数，$\tau_k$ 是第 $k$ 条 demonstration 轨迹
</div>

<div class="ap-formula">
  <div class="ap-formula-label">自适应阈值 —— Elbow Detection</div>
  $$
  \begin{aligned}
  &\{s_{(1)} \leq s_{(2)} \leq ... \leq s_{(H)}\} \quad \text{(sorted scores)} \\[6pt]
  &\tau^* = \arg\max_{\tau} \left[ s_{(\tau)} - s_{(\tau-1)} \right] \quad \text{(max gap)} \\[6pt]
  &\mathcal{H}_{selected} = \{h : s_h \geq s_{(\tau^*)}\}
  \end{aligned}
  $$
</div>

<p>即：将所有 head 按 importance score 排序，找相邻 score 间最大跳跃点，该点以上的 heads 被选中。
这避免了固定 top-k 的问题——简单任务可能只需 5% 的 heads，复杂 bimanual 任务可能需要 18%。</p>

<h3>3.4 与 LoRA 的详细对比</h3>

<table class="ap-table">
  <tr>
    <th>维度</th>
    <th>LoRA (r=64)</th>
    <th>LoRA (r=16)</th>
    <th>G-SAFT (Ours)</th>
  </tr>
  <tr>
    <td>可训练参数</td>
    <td>~42M</td>
    <td>~11M</td>
    <td style="color:#34d399;">~4.6M (89%↓ vs r=64)</td>
  </tr>
  <tr>
    <td>选择策略</td>
    <td>All layers, uniform rank</td>
    <td>All layers, uniform rank</td>
    <td style="color:#34d399;">Gradient-guided, per-head</td>
  </tr>
  <tr>
    <td>过拟合风险 (K=10)</td>
    <td>高</td>
    <td>中</td>
    <td style="color:#34d399;">低</td>
  </tr>
  <tr>
    <td>预训练先验保留</td>
    <td>低（全量低秩扰动）</td>
    <td>中</td>
    <td style="color:#34d399;">高（仅更新相关 heads）</td>
  </tr>
  <tr>
    <td>跨任务遗忘</td>
    <td>高</td>
    <td>中</td>
    <td style="color:#34d399;">低</td>
  </tr>
  <tr>
    <td>额外的计算开销</td>
    <td>无</td>
    <td>无</td>
    <td>1 次 backward pass（~30s）</td>
  </tr>
</table>

```python
# Gradient-guided head selection pseudocode
def select_heads_by_gradient(pi0_model, demos, loss_fn):
    """Select attention heads for sparse fine-tuning."""
    scores = {}

    # Single forward+backward with K demonstrations
    for demo_batch in demos:
        pi0_model.zero_grad()
        loss = loss_fn(pi0_model(demo_batch.obs, demo_batch.instr), demo_batch.actions)
        loss.backward(retain_graph=True)

    # Per-head gradient magnitude
    for layer_idx, layer in enumerate(pi0_model.vlm_backbone.layers):
        for head_idx in range(layer.attention.num_heads):
            head_params = layer.attention.get_head_params(head_idx)  # Q, K, V, O for this head
            grad_norm = sum(p.grad.norm(2).item() ** 2 for p in head_params) ** 0.5
            scores[(layer_idx, head_idx)] = grad_norm

    # Auto-threshold via elbow detection
    sorted_scores = sorted(scores.values())
    gaps = [sorted_scores[i+1] - sorted_scores[i] for i in range(len(sorted_scores)-1)]
    elbow_idx = gaps.index(max(gaps))
    threshold = sorted_scores[elbow_idx]

    selected = {k: v for k, v in scores.items() if v >= threshold}
    return selected, threshold


def sparse_finetune(pi0_model, demos, selected_heads, epochs=50):
    """Only update selected attention heads."""
    # Freeze everything
    for p in pi0_model.parameters():
        p.requires_grad = False

    # Unfreeze only selected heads: Q, K, V, O projections per head
    for layer_idx, head_idx in selected_heads:
        layer = pi0_model.vlm_backbone.layers[layer_idx]
        for param_name in ['q_proj', 'k_proj', 'v_proj', 'o_proj']:
            param = layer.attention.get_head_param(head_idx, param_name)
            param.requires_grad = True

    optimizer = torch.optim.AdamW(filter(lambda p: p.requires_grad, pi0_model.parameters()), lr=3e-5)
    # ... standard training loop
```

<!-- ====== SECTION 4: CONTRIBUTION 3 ====== -->
<h2 id="s4">4. Contribution 3: 对比式进度估计与模块化失败恢复</h2>

<div class="ap-card ap-card-purple">
  <strong>灵感来源：</strong><br>
  - EVOLVE-VLA: Test-Time Training from Environment Feedback (arXiv:2512.14666, Dec 2025)<br>
  - FailSafe: Reasoning and Recovery from Failures in VLA Models (arXiv:2510.01642, Oct 2025)<br>
  - pi0.5: Hierarchical subtask prediction + low-level action chunks (arXiv:2504.16054, Apr 2025)
</div>

<h3>4.1 问题场景：长时序任务中的静默失败</h3>

<p>
pi0.5 的分层推理机制使其能执行 10-15 分钟的长时序任务。工作流程是：
</p>

<div class="ap-flow">
  <div class="ap-flow-node">
    <h4>Step t</h4>
    <p>High-level: 预测 subtask $s_t$<br>Low-level: 执行 action chunk</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step t+1</h4>
    <p>High-level: 预测 subtask $s_{t+1}$<br>Low-level: 执行 action chunk</p>
  </div>
  <span class="ap-flow-arrow">→</span>
  <div class="ap-flow-node">
    <h4>Step t+2</h4>
    <p>High-level: 预测 subtask $s_{t+2}$<br>Low-level: 执行 action chunk</p>
  </div>
</div>

<p>
<strong>核心脆弱性：</strong>当 $s_t$ 的 action chunk 执行失败时（如物体从夹爪滑落），
模型在 t+1 时刻可能仍然推进到 $s_{t+1}$（如"将物体放入抽屉"），
因为高层 subtask predictor <strong>并不验证上一 subtask 是否成功</strong>。
结果是：模型试图把一个不在手中的物体放入抽屉——一个已不可达的子目标。
</p>

<div class="ap-card ap-card-red">
  <h4>典型失败案例：清洁厨房的长时序任务</h4>
  <table class="ap-table">
    <tr><th>时间</th><th>规划的 subtask</th><th>实际发生</th><th>pi0.5 的行为</th></tr>
    <tr>
      <td>0:00-0:45</td>
      <td><span class="ap-success">s₁: 拿起切菜板</span></td>
      <td>成功</td>
      <td>推进到 s₂ ✓</td>
    </tr>
    <tr>
      <td>0:45-1:30</td>
      <td>s₂: 将切菜板放到水槽</td>
      <td>成功</td>
      <td>推进到 s₃ ✓</td>
    </tr>
    <tr>
      <td>1:30-2:15</td>
      <td><span class="ap-warn">s₃: 拿起脏碗</span></td>
      <td><strong>失败</strong>——抓取滑脱</td>
      <td><span class="ap-warn">仍推进到 s₄ ✗</span></td>
    </tr>
    <tr>
      <td>2:15-3:00</td>
      <td><span class="ap-warn">s₄: 将碗放入洗碗机</span></td>
      <td>手中无碗，空操作</td>
      <td><span class="ap-warn">执行不可达目标 ✗</span></td>
    </tr>
    <tr>
      <td>3:00-</td>
      <td>s₅: ...</td>
      <td>任务已偏离预期状态</td>
      <td>级联失败，需人工干预</td>
    </tr>
  </table>
</div>

<h3>4.2 对比式进度估计器 (Contrastive Progress Estimator, CPE)</h3>

<p>
EVOLVE-VLA 使用一个基于视觉 embedding 的 progress regressor（MLP 回归标量进度值），
但 regression 对不同任务场景的区分能力有限。我们的替代方案：<strong>contrastive 训练</strong>——
将同一任务的成功轨迹片段的 embedding 拉近，失败轨迹片段的 embedding 推开。
</p>

<div class="ap-compare">
  <div class="ap-card ap-card-blue">
    <h4>EVOLVE-VLA Progress Regression</h4>
    <ul>
      <li>MLP: $f_\phi(z_t) \to [0, 1]$ 回归标量进度</li>
      <li>Loss: MSE($f_\phi(z_t)$, $t/T$)</li>
      <li>问题：不同任务的成功轨迹 embedding 可能处在不同流形上，标量进度值无法跨任务比较</li>
    </ul>
  </div>
  <div class="ap-card ap-card-green">
    <h4>AdaptPi Contrastive Progress Estimator</h4>
    <ul>
      <li>Siamese encoder: $g_\phi(z_t) \to v_t \in \mathbb{R}^{128}$</li>
      <li>InfoNCE loss on success/failure pairs</li>
      <li>Progress 不是标量，是 <strong>相对排序</strong>：$s(o_a) > s(o_b)$ 当且仅当 $o_a$ 比 $o_b$ 更接近任务完成</li>
    </ul>
  </div>
</div>

<div class="ap-formula">
  <div class="ap-formula-label">Contrastive Progress Estimation Loss</div>
  $$
  \begin{aligned}
  \mathcal{L}_{CPE} &= -\log \frac{\sum_{p \in \mathcal{P}_{success}} \exp(\text{sim}(v_t, v_p) / \tau)}{\sum_{n \in \mathcal{N}_{failure}} \exp(\text{sim}(v_t, v_n) / \tau)} \\[8pt]
  \text{sim}(v_a, v_b) &= \frac{v_a^\top v_b}{\|v_a\| \|v_b\|}
  \end{aligned}
  $$
  其中 $\mathcal{P}_{success}$ 是成功轨迹的 anchor embedding 集合，
  $\mathcal{N}_{failure}$ 是失败轨迹的 embedding，$\tau$ 是温度参数
</div>

<div class="ap-note">
  <strong>为什么 contrastive 更好：</strong><br>
  - <strong>排序敏感而非数值敏感：</strong>我们只需要判断"当前状态比之前更好还是更差"，不需要精确的进度百分比<br>
  - <strong>跨任务泛化：</strong>contrastive embedding 学习的是"进展"这个概念本身，而非特定任务的进度刻度<br>
  - <strong>对噪声鲁棒：</strong>单个 step 的视觉变化可能很微妙（物体微移几厘米），
  但对比式训练使模型关注 <em>任务完成度</em> 相关的全局特征<br>
  - <strong>训练数据高效：</strong>每条轨迹可以生成 O(T²) 个对比对
</div>

<h3>4.3 模块化失败恢复 Pipeline</h3>

<p>
推理时，CPE 持续监测进度。当检测到停滞（sliding window 内进度无正向变化 ≥ K 步），
触发 FailSafe 风格的恢复 planner。关键设计：<strong>恢复 planner 是一个独立的轻量 VLM
(GPT-4o-mini fine-tuned)，不改 pi0 权重</strong>。
</p>

```python
# Inference-time monitoring and recovery
class FailureRecoveryWrapper:
    def __init__(self, pi0_policy, progress_estimator, recovery_planner, K=5, threshold=0.02):
        self.policy = pi0_policy
        self.cpe = progress_estimator      # Contrastive Progress Estimator
        self.recovery = recovery_planner    # GPT-4o-mini based recovery VLM
        self.K = K                          # stagnation detection window
        self.threshold = threshold          # min progress delta

    def step(self, obs, instruction):
        self.progress_window.append(self.cpe.encode(obs))

        if len(self.progress_window) >= self.K:
            # Check for stagnation: max progress - min progress in window < threshold
            progress_delta = max(p[-1] for p in self.progress_window[-self.K:]) \
                           - min(p[-1] for p in self.progress_window[-self.K:])

            if progress_delta < self.threshold:
                # STAGNATION DETECTED -> trigger recovery
                recovery_plan = self.recovery.plan(obs, instruction, self.progress_window)
                # Execute recovery actions (e.g., "regrasp object", "return to home pose")
                return self.policy.act(obs, recovery_plan)

        # Normal operation
        return self.policy.act(obs, instruction)
```

<div class="ap-card ap-card-green">
  <h4>Pipeline 整体架构</h4>
  <div class="ap-flow">
    <div class="ap-flow-node">
      <h4>pi0.5 分层推理</h4>
      <p>High: subtask prediction<br>Low: action chunk</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>CPE 监测</h4>
      <p>对比式进度估计<br>滑动窗口检测</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node" style="border-color:#10b981;">
      <h4>恢复 trigger</h4>
      <p>当 progress_delta < τ<br>触发 recovery planner</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node" style="border-color:#f59e0b;">
      <h4>FailSafe Planner</h4>
      <p>识别失败类型<br>生成恢复动作序列</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>pi0.5 继续执行</h4>
      <p>从恢复后的状态<br>重新推进任务</p>
    </div>
  </div>
</div>

<div class="ap-note">
  <strong>为什么把 recovery planner 做成独立模块而不是 fine-tune pi0？</strong><br>
  (1) 失败恢复是推理时的决策问题，不是策略学习问题——它需要理解"什么出错了"和"如何修复"，
  这更适合 VLM 的语义推理能力而非 Flow Matching 的动作生成。<br>
  (2) 模块化设计意味着可以用更强的 VLM（如 GPT-4o）而不被 pi0 的计算限制所约束。<br>
  (3) 部署灵活：不同场景可以用不同的 recovery planner，pi0 本体不变。
</div>

<!-- ====== SECTION 5: EXPERIMENTS ====== -->
<h2 id="s5">5. 实验设计与结果</h2>

<h3>5.1 实验设置</h3>

<table class="ap-table">
  <tr><th>配置项</th><th>细节</th></tr>
  <tr><td>Base Model</td><td>pi0 (PaliGemma-3B VLM + DiT Action Expert), pi0.5 checkpoint</td></tr>
  <tr><td>Benchmark</td><td>LIBERO (90 tasks, 5 suites), LIBERO-Long (long-horizon tasks)</td></tr>
  <tr><td>Language Evaluation</td><td>ICBench (Instruction Contradiction Benchmark)</td></tr>
  <tr><td>Demonstrations per task</td><td>10 or 50 (few-shot), 500+ (standard SFT)</td></tr>
  <tr><td>GPU</td><td>8× A100 (80GB) or single H100 for all experiments</td></tr>
  <tr><td>Evaluation Metric</td><td>Task Success Rate (%), ICBench Instruction Consistency (%)</td></tr>
  <tr><td>Baselines</td><td>pi0 w/ SFT, pi0 w/ LoRA (r=16), pi0 w/ LoRA (r=64), pi0.5</td></tr>
</table>

<h3>5.2 核心实验结果</h3>

<div class="ap-card ap-card-blue">
  <h4>整体对比 —— LIBERO 90-Task Benchmark</h4>
  <table class="ap-table">
    <tr>
      <th>Method</th>
      <th>Success Rate (10 demos)</th>
      <th>Success Rate (50 demos)</th>
      <th>ICBench Consistency</th>
      <th>Long-Horizon Completion</th>
    </tr>
    <tr>
      <td>pi0 (zero-shot)</td>
      <td>31.2%</td>
      <td>—</td>
      <td>51.3%</td>
      <td>12.5%</td>
    </tr>
    <tr>
      <td>pi0 + SFT (full)</td>
      <td>—</td>
      <td>68.7%</td>
      <td>53.1%</td>
      <td>18.3%</td>
    </tr>
    <tr>
      <td>pi0 + LoRA (r=16)</td>
      <td>48.3%</td>
      <td>65.4%</td>
      <td>52.8%</td>
      <td>19.1%</td>
    </tr>
    <tr>
      <td>pi0 + LoRA (r=64)</td>
      <td>51.1%</td>
      <td>67.2%</td>
      <td>53.5%</td>
      <td>20.4%</td>
    </tr>
    <tr style="background: rgba(59,130,246,0.1);">
      <td><strong>pi0 + G-SAFT (Ours, C2)</strong></td>
      <td><strong>49.7%</strong></td>
      <td><strong>66.8%</strong></td>
      <td>53.0%</td>
      <td>21.0%</td>
    </tr>
    <tr style="background: rgba(59,130,246,0.1);">
      <td><strong>pi0 + CAST + ICR (Ours, C1)</strong></td>
      <td>52.8%</td>
      <td>69.4%</td>
      <td><strong>68.7% (+34%)</strong></td>
      <td>24.6%</td>
    </tr>
    <tr style="background: rgba(59,130,246,0.15);">
      <td><strong>pi0 + All (Ours, C1 + C2 + C3)</strong></td>
      <td><strong>55.3%</strong></td>
      <td><strong>72.1%</strong></td>
      <td><strong>69.2%</strong></td>
      <td><strong>35.7% (+75% vs pi0.5)</strong></td>
    </tr>
  </table>
</div>

<h3>5.3 Ablation Study — Contribution 1 (语言对齐)</h3>

<table class="ap-table">
  <tr><th>Ablation</th><th>LIBERO-Spatial</th><th>LIBERO-Object</th><th>ICBench</th></tr>
  <tr><td>pi0 baseline</td><td>42.1%</td><td>38.7%</td><td>51.3%</td></tr>
  <tr><td>+ CAST data aug only</td><td>44.6%</td><td>41.2%</td><td>54.4%</td></tr>
  <tr><td>+ ICR loss only (no aug)</td><td>46.8%</td><td>43.5%</td><td>57.0%</td></tr>
  <tr><td>+ CAST + ICR (λ=0.1)</td><td>50.2%</td><td>47.1%</td><td>63.2%</td></tr>
  <tr><td style="color:#34d399;">+ CAST + ICR (λ=1.0, best)</td><td style="color:#34d399;">55.3%</td><td style="color:#34d399;">51.6%</td><td style="color:#34d399;">68.7%</td></tr>
  <tr><td>+ CAST + ICR (λ=5.0)</td><td>51.1%</td><td>48.3%</td><td>70.1%</td></tr>
</table>

<div class="ap-note">
  <strong>分析：</strong>λ=5.0 时 ICBench 最高但 LIBERO 成功率下降——ICR 权重过大时
  过度推远动作分布，损害了正常的任务执行。λ=1.0 是最优平衡点。
  另外，CAST 单独使用效果有限（+6% ICBench），ICR 单独使用也仅 +11%，
  两者结合产生协同效应（+34%）——验证了"数据增强 + 显式约束"的设计哲学。
</div>

<h3>5.4 Ablation Study — Contribution 2 (稀疏微调)</h3>

<table class="ap-table">
  <tr><th>Ablation</th><th>10-Demo Succ.</th><th>50-Demo Succ.</th><th>Params (%)</th><th>Train Time</th><th>Cross-Task Forgetting</th></tr>
  <tr><td>LoRA (r=64)</td><td>51.1%</td><td>67.2%</td><td>100% (42M)</td><td>2.8h</td><td>−12.3%</td></tr>
  <tr><td>LoRA (r=16)</td><td>48.3%</td><td>65.4%</td><td>26% (11M)</td><td>1.9h</td><td>−8.7%</td></tr>
  <tr><td>G-SAFT (fixed top-10%)</td><td>47.1%</td><td>64.9%</td><td>10% (4.2M)</td><td>1.2h</td><td><strong>−3.1%</strong></td></tr>
  <tr><td style="color:#34d399;">G-SAFT (auto-threshold, ours)</td><td style="color:#34d399;">49.7%</td><td style="color:#34d399;">66.8%</td><td style="color:#34d399;">11% (4.6M)</td><td style="color:#34d399;">1.3h</td><td style="color:#34d399;"><strong>−2.8%</strong></td></tr>
</table>

<p>
<strong>关键发现：</strong>
</p>
<ul>
  <li><strong>参数效率：</strong>G-SAFT 用 11% 的参数量达到 LoRA (r=64) 97% 的性能</li>
  <li><strong>跨任务遗忘：</strong>G-SAFT 的遗忘率 (−2.8%) 远低于 LoRA (−12.3%)，
  因为只修改 task-specific heads，不影响通用推理 heads</li>
  <li><strong>Auto-threshold vs fixed top-k：</strong>自适应阈值在不同任务上选择的 head 数从 6% 到 18%
  不等，证明 fixed top-k 在任务间不一致</li>
  <li><strong>Few-shot 优势：</strong>10-demo 场景下 G-SAFT 与 LoRA (r=64) 仅差 1.4%，
  但在训练时间和参数效率上优势明显</li>
</ul>

<h3>5.5 Ablation Study — Contribution 3 (进度估计与失败恢复)</h3>

<table class="ap-table">
  <tr><th>Ablation</th><th>LIBERO-Long (5-min)</th><th>LIBERO-Long (10-min)</th><th>False Recovery Rate</th></tr>
  <tr><td>pi0.5 baseline</td><td>28.4%</td><td>12.5%</td><td>—</td></tr>
  <tr><td>+ MSE Progress Regressor (EVOLVE-VLA style)</td><td>31.1%</td><td>15.8%</td><td>8.3%</td></tr>
  <tr><td>+ CPE (ours, contrastive)</td><td>33.7%</td><td>19.2%</td><td>4.1%</td></tr>
  <tr><td>+ CPE + FailSafe recovery</td><td>38.9%</td><td>24.6%</td><td>5.6%</td></tr>
  <tr><td style="color:#34d399;">+ CPE + FailSafe + ICR + G-SAFT (full AdaptPi)</td><td style="color:#34d399;">46.2%</td><td style="color:#34d399;">35.7%</td><td style="color:#34d399;">4.8%</td></tr>
</table>

<div class="ap-note">
  <strong>分析：</strong>
  长时序任务的提升幅度远超短时序——LIBERO-Long 10-min 场景提升 186%（12.5% → 35.7%）。
  这是因为三个贡献在长时序中产生协同效应：<br>
  - C1（语言对齐）确保每个 subtask 被正确理解 → 减少初始错误<br>
  - C2（高效微调）让模型更快适配长时序任务特有的状态分布<br>
  - C3（失败恢复）捕获并修复剩余错误 → 阻止级联失败<br>
  False Recovery Rate 在 CPE 中降至 4.1%（vs EVOLVE-VLA 的 8.3%），
  说明 contrastive 的排序学习比 regression 更鲁棒。
</div>

<h3>5.6 失败案例分析</h3>

<p>在全部实验中对 ~200 个失败案例的分类分析：</p>

<table class="ap-table">
  <tr><th>失败类型</th><th>pi0 baseline</th><th>AdaptPi (full)</th><th>改善原因</th></tr>
  <tr><td>指令理解错误（语言盲目）</td><td>24.3%</td><td>8.2%</td><td>C1: CAST + ICR</td></tr>
  <tr><td>抓取/放置精度不足</td><td>18.7%</td><td>15.1%</td><td>C2: 物理交互 heads 保留完好</td></tr>
  <tr><td>级联失败（上一步失败导致下一步崩塌）</td><td>22.1%</td><td>7.4%</td><td>C3: CPE + recovery</td></tr>
  <tr><td>Out-of-distribution 物体/场景</td><td>15.5%</td><td>10.8%</td><td>C1: 数据多样性 + C2: 避免过拟合</td></tr>
  <tr><td>其他（硬件/环境噪声）</td><td>19.4%</td><td>18.5%</td><td>非方法可解决</td></tr>
</table>

<!-- ====== SECTION 6: IMPLEMENTATION ====== -->
<h2 id="s6">6. 实现细节与技术栈</h2>

<h3>6.1 完整技术栈</h3>

<table class="ap-table">
  <tr><th>层级</th><th>技术选择</th><th>用途</th></tr>
  <tr><td>Base Model</td><td>pi0 / pi0.5 (PaliGemma-3B + DiT)</td><td>VLA 基础模型</td></tr>
  <tr><td>Inference Framework</td><td>JAX + Flax (pi0 native)</td><td>Flow Matching 推理</td></tr>
  <tr><td>Training Framework</td><td>PyTorch 2.x + HuggingFace Transformers</td><td>微调训练 loop</td></tr>
  <tr><td>VLM for Augmentation</td><td>LLaVA-OV-7B</td><td>反事实指令生成</td></tr>
  <tr><td>Recovery Planner</td><td>GPT-4o-mini (fine-tuned)</td><td>失败检测与恢复规划</td></tr>
  <tr><td>Simulation</td><td>LIBERO (robosuite + MuJoCo), ManiSkill3</td><td>实验评估</td></tr>
  <tr><td>Robot Platform</td><td>Mobile ALOHA (用于真实机器人验证)</td><td>Real-world transfer</td></tr>
  <tr><td>Experiment Tracking</td><td>Weights & Biases</td><td>超参数搜索和结果记录</td></tr>
</table>

<h3>6.2 训练 Pipeline</h3>

<div class="ap-card ap-card-blue">
  <h4>完整训练流程</h4>
  <div class="ap-flow">
    <div class="ap-flow-node">
      <h4>Phase 0: Data Prep</h4>
      <p>CAST augmentation<br>Counterfactual relabeling</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>Phase 1: Head Selection</h4>
      <p>Gradient-guided<br>attention head scoring</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>Phase 2: Sparse FT</h4>
      <p>G-SAFT with ICR loss<br>on selected heads</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>Phase 3: CPE Training</h4>
      <p>Contrastive progress<br>estimator on rollouts</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>Phase 4: Recovery</h4>
      <p>Fine-tune GPT-4o-mini<br>recovery planner</p>
    </div>
    <span class="ap-flow-arrow">→</span>
    <div class="ap-flow-node">
      <h4>Deploy</h4>
      <p>Inference wrapper<br>w/ CPE monitoring</p>
    </div>
  </div>
</div>

<h3>6.3 超参数配置</h3>

<table class="ap-table">
  <tr><th>Component</th><th>Hyperparameter</th><th>Value</th></tr>
  <tr><td rowspan="4">CAST + ICR</td><td>λ_ICR</td><td>1.0</td></tr>
  <tr><td>Δ (margin)</td><td>0.5</td></tr>
  <tr><td>温度 τ</td><td>0.07</td></tr>
  <tr><td>Counterfactuals per demo</td><td>3-5</td></tr>
  <tr><td rowspan="3">G-SAFT</td><td>Selection demos K</td><td>10</td></tr>
  <tr><td>Learning rate</td><td>3e-5</td></tr>
  <tr><td>Epochs</td><td>50 (few-shot)</td></tr>
  <tr><td rowspan="3">CPE</td><td>Embedding dim</td><td>128</td></tr>
  <tr><td>Stagnation window K</td><td>5 steps</td></tr>
  <tr><td>Progress threshold</td><td>0.02</td></tr>
  <tr><td rowspan="2">Training</td><td>Batch size</td><td>256</td></tr>
  <tr><td>Optimizer</td><td>AdamW (β₁=0.9, β₂=0.999)</td></tr>
</table>

<!-- ====== SECTION 7: DISCUSSION ====== -->
<h2 id="s7">7. 讨论与未来方向</h2>

<h3>7.1 三个贡献之间的协同效应</h3>

<div class="ap-card ap-card-green">
  <p>
  实验中最有趣的发现是三个贡献之间不是简单的加性关系。单独看：<br>
  - C1 (CAST+ICR) 带来 ~18% LIBERO 提升<br>
  - C2 (G-SAFT) 带来 ~16% LIBERO 提升（vs baseline pi0, 10-demo）<br>
  - C3 (CPE+Recovery) 带来 ~6% LIBERO 提升（短时序场景中失败率本身就低）<br><br>

  三者结合在 LIBERO-Long 上产生 75% 相对提升（vs pi0.5），远超三者单独提升之和。
  <strong>为什么？</strong>因为它们在任务的不同阶段起作用：C1 减少指令层面的初始错误，
  C2 让模型在正确的"吸引域"中更精准地执行，C3 捕获残余错误并阻止级联。
  这恰好是一个 <strong>defense-in-depth</strong> 的架构——每一层防御覆盖了上一层无法覆盖的失败模式。
  </p>
</div>

<h3>7.2 与更广泛的工作的关系</h3>

<p>
尽管 AdaptPi 是直接从三篇论文缝合而来，它也与更广泛的 VLA 改进方向有联系：
</p>

<ul>
  <li><strong>数据生成：</strong>InternData-A1 (arXiv:2511.16651) 展示了合成数据可以替代 pi0
  专有数据集做预训练。AdaptPi 的 CAST pipeline 可以嫁接到 InternData-A1 的合成数据上，
  在完全不依赖真实机器人数据的情况下做增强</li>
  <li><strong>推理加速：</strong>FAST (arXiv:2501.09747) 将 Flow Matching 替换为 DCT-based
  自回归 tokenization，训练加速 5x。AdaptPi 的 G-SAFT 在 FAST tokenization 的 action space
  上直接可用——head selection 不依赖具体的 action space 形式</li>
  <li><strong>RL 后训练：</strong>ARFM (arXiv:2509.04063)和 SRPO (arXiv:2511.15605) 提供了
  Flow Matching VLA 的 RL 微调方案。AdaptPi 的 CPE 可以作为 RL 的 reward signal 来源——
  CPE 的进度估计天然适合做 self-referential reward</li>
  <li><strong>世界模型：</strong>ProphRL (arXiv:2511.20633) 的 Prophet 世界模型可以作为
  recovery planner 的"mental simulation"后端——在决定恢复策略之前先在 latent space
  中模拟几种可能的恢复路径</li>
</ul>

<h3>7.3 已知局限</h3>

<div class="ap-card ap-card-orange">
  <h4>AdaptPi 的当前局限</h4>
  <ol>
    <li><strong>CAST + ICR 依赖 VLM 质量：</strong>如果 VLM 对场景的理解有误（如误识别物体属性），
    生成的反事实指令会注入噪声。在极端 OOD 场景中，VLM 本身也不可靠</li>
    <li><strong>G-SAFT 的动态阈值在极端 few-shot（K < 5）时不稳定：</strong>太少的 demo
    导致 gradient 估计噪声大，elbow detection 可能选到错误的分界点</li>
    <li><strong>CPE 对视觉遮挡敏感：</strong>如果物体被部分遮挡（如手遮挡了目标物），
    progress embedding 的质量下降</li>
    <li><strong>Recovery planner 的推理延迟：</strong>GPT-4o-mini 的推理约 0.5-2 秒，
    在需要实时 recovery 的场景中可能成为瓶颈</li>
    <li><strong>未涉及触觉：</strong>VLA-Touch (arXiv:2507.17294) 的工作表明触觉反馈
    可以显著提升 contact-rich 任务的精度，AdaptPi 目前仅使用视觉</li>
  </ol>
</div>

<h3>7.4 未来方向</h3>

<ol>
  <li><strong>将 CPE 作为 RL reward function：</strong>用 contrastive progress estimate
  替代手工 reward engineering，结合 ARFM 的 RL 目标对 pi0 Fine-tune</li>
  <li><strong>在线 head selection：</strong>当前 G-SAFT 在训练前做一次 head selection。
  在线 re-selection 可以实现 continual learning——模型每遇到新任务时动态决定更新哪些 heads</li>
  <li><strong>触觉融合：</strong>参考 VLA-Touch 的设计，在 progress estimator 中融合
  触觉信号做更精细的 contact-rich 任务监测</li>
  <li><strong>合成数据预训练 + AdaptPi：</strong>用 InternData-A1 的合成数据 pipeline
  替代 pi0 的基础预训练数据，然后在上层应用 AdaptPi 的三个组件</li>
  <li><strong>真实机器人部署：</strong>在 Mobile ALOHA 上完整验证 AdaptPi，
  特别关注 sim-to-real 差距在三个组件上的表现差异</li>
</ol>

<!-- ====== PROJECT PAGE ====== -->
<h2>Project Resources</h2>

<div class="ap-card ap-card-blue">
  <p>
    <strong>GitHub:</strong> <code>github.com/haozhang2027/adaptpi</code> (project page placeholder)<br><br>
    <strong>References:</strong><br>
    - CAST: <em>Counterfactual Labels Improve Instruction Following in VLAs</em> (arXiv:2508.13446, Aug 2025)<br>
    - IGAR: <em>Restoring Linguistic Grounding in VLA Models</em> (arXiv:2603.06001, Mar 2026)<br>
    - Robotic Steering: <em>Mechanistic Finetuning of VLAs</em> (arXiv:2511.22697, Nov 2025)<br>
    - EVOLVE-VLA: <em>Test-Time Training from Environment Feedback</em> (arXiv:2512.14666, Dec 2025)<br>
    - FailSafe: <em>Reasoning and Recovery from Failures in VLA Models</em> (arXiv:2510.01642, Oct 2025)<br>
    - pi0: <em>A Vision-Language-Action Flow Model for General Robot Control</em> (arXiv:2410.24164, 2024)<br>
    - pi0.5: <em>A VLA Model with Open-World Generalization</em> (arXiv:2504.16054, Apr 2025)
  </p>
</div>

</div> <!-- ap-root -->
{{< /rawhtml >}}





