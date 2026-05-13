---
title: "π0 模型架构深度解析：从源码看 VLA 的每一个细节"
date: 2026-05-13
draft: false
math: false
tags:
  - VLA
  - Robotics
  - Flow Matching
  - PaliGemma
categories:
  - 具身智能
---

{{< rawhtml >}}
<style>
  .pi0-root { color: #cbd5e1; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; line-height: 1.75; }
  .pi0-root h2 { color: #60a5fa; border-bottom: 2px solid #2d3748; padding-bottom: 0.5rem; margin-top: 2.5rem; }
  .pi0-root h3 { color: #a78bfa; margin-top: 2rem; }
  .pi0-root h4 { color: #34d399; margin-top: 1.5rem; }
  .pi0-hero { background: linear-gradient(135deg, #1e293b, #0f1117); padding: 2rem; border-radius: 12px; border: 1px solid #2d3748; margin-bottom: 2rem; }
  .pi0-hero h1 { color: #e2e8f0; font-size: 2rem; margin: 0 0 0.5rem 0; }
  .pi0-hero p { color: #94a3b8; margin: 0; }
  .pi0-toc { background: #1e293b; border: 1px solid #2d3748; border-radius: 8px; padding: 1rem 1.5rem; margin-bottom: 2rem; }
  .pi0-toc ol { margin: 0; padding-left: 1.5rem; }
  .pi0-toc a { color: #60a5fa; text-decoration: none; }
  .pi0-toc a:hover { text-decoration: underline; }
  .pi0-card { background: #1e293b; border: 1px solid #2d3748; border-radius: 8px; padding: 1rem 1.25rem; margin: 1rem 0; }
  .pi0-badge { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 0.82rem; font-family: "SF Mono", Consolas, monospace; margin: 0 2px; }
  .pi0-dim { background: #312e81; color: #c7d2fe; border: 1px solid #4f46e5; }
  .pi0-shape { background: #064e3b; color: #a7f3d0; border: 1px solid #059669; }
  .pi0-param { background: #7c2d12; color: #fed7aa; border: 1px solid #ea580c; }
  .pi0-code-inline { background: #0f1117; color: #fbbf24; padding: 2px 6px; border-radius: 4px; font-family: "SF Mono", Consolas, monospace; font-size: 0.9em; }
  .pi0-root pre { background: #0f1117 !important; border: 1px solid #2d3748; border-radius: 8px; padding: 1rem; overflow-x: auto; font-family: "SF Mono", Consolas, monospace; font-size: 0.88rem; }
  .pi0-root code { color: #e2e8f0; }
  .pi0-root .tok-kw { color: #c084fc; }
  .pi0-root .tok-fn { color: #60a5fa; }
  .pi0-root .tok-str { color: #86efac; }
  .pi0-root .tok-num { color: #fbbf24; }
  .pi0-root .tok-cm { color: #64748b; font-style: italic; }
  .pi0-flow { background: #0f1117; border: 1px solid #2d3748; border-radius: 8px; padding: 1rem; font-family: "SF Mono", Consolas, monospace; font-size: 0.85rem; white-space: pre; overflow-x: auto; color: #cbd5e1; }
  .pi0-table { width: 100%; border-collapse: collapse; margin: 1rem 0; font-size: 0.92rem; }
  .pi0-table th { background: #1e293b; color: #60a5fa; padding: 0.75rem; text-align: left; border: 1px solid #2d3748; }
  .pi0-table td { padding: 0.7rem; border: 1px solid #2d3748; color: #cbd5e1; }
  .pi0-table tr:nth-child(even) td { background: #161b22; }
  .pi0-seq { display: flex; flex-wrap: wrap; gap: 4px; padding: 1rem; background: #0f1117; border-radius: 8px; border: 1px solid #2d3748; margin: 1rem 0; }
  .pi0-tok { padding: 8px 10px; border-radius: 4px; font-size: 0.78rem; font-family: "SF Mono", Consolas, monospace; text-align: center; color: #e2e8f0; }
  .pi0-tok-img { background: #1e40af; border: 1px solid #3b82f6; }
  .pi0-tok-lang { background: #6b21a8; border: 1px solid #a855f7; }
  .pi0-tok-state { background: #9a3412; border: 1px solid #ea580c; }
  .pi0-tok-act { background: #166534; border: 1px solid #22c55e; }
  .pi0-callout { border-left: 4px solid #60a5fa; background: #1e293b; padding: 0.75rem 1rem; margin: 1rem 0; border-radius: 0 8px 8px 0; }
  .pi0-callout-warn { border-left-color: #f59e0b; }
  .pi0-callout-tip { border-left-color: #34d399; }
  .pi0-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; margin: 1rem 0; }
  @media (max-width: 768px) { .pi0-grid { grid-template-columns: 1fr; } }

  /* ========== Diagram styles ========== */
  .pi0-diagram { background: #0b1220; border: 1px solid #2d3748; border-radius: 10px; padding: 1.5rem; margin: 1.5rem 0; overflow-x: auto; }
  .pi0-diagram-title { color: #60a5fa; font-weight: 600; margin-bottom: 1rem; text-align: center; font-size: 0.95rem; }
  .pi0-node { display: inline-block; padding: 0.5rem 0.8rem; border-radius: 6px; font-family: "SF Mono", Consolas, monospace; font-size: 0.82rem; text-align: center; color: #e2e8f0; border: 1px solid; min-width: 80px; }
  .pi0-node-input { background: #0c4a6e; border-color: #0ea5e9; }
  .pi0-node-proc { background: #1e3a8a; border-color: #3b82f6; }
  .pi0-node-attn { background: #4c1d95; border-color: #a855f7; }
  .pi0-node-out { background: #064e3b; border-color: #10b981; }
  .pi0-node-loss { background: #7f1d1d; border-color: #ef4444; }
  .pi0-node-cache { background: #78350f; border-color: #f59e0b; }
  .pi0-node-tag { display: block; font-size: 0.68rem; color: #94a3b8; margin-top: 2px; font-style: italic; }
  .pi0-row { display: flex; align-items: center; justify-content: center; gap: 8px; flex-wrap: wrap; margin: 6px 0; }
  .pi0-arrow { color: #64748b; font-family: "SF Mono", Consolas, monospace; font-size: 1.1rem; }
  .pi0-arrow-label { color: #fbbf24; font-size: 0.72rem; font-style: italic; }
  .pi0-col { display: flex; flex-direction: column; align-items: center; gap: 4px; }
  .pi0-arch-svg { width: 100%; max-width: 880px; margin: 0 auto; display: block; }
  .pi0-arch-svg text { fill: #cbd5e1; font-family: "SF Mono", Consolas, monospace; font-size: 11px; }
  .pi0-arch-svg .lbl { fill: #94a3b8; font-size: 10px; }
  .pi0-arch-svg .title { fill: #e2e8f0; font-size: 12px; font-weight: bold; }
  .pi0-arch-svg .dim { fill: #fbbf24; font-size: 10px; }
  .pi0-legend { display: flex; flex-wrap: wrap; gap: 12px; justify-content: center; margin-top: 1rem; font-size: 0.8rem; color: #94a3b8; }
  .pi0-legend-item { display: flex; align-items: center; gap: 4px; }
  .pi0-legend-box { display: inline-block; width: 12px; height: 12px; border-radius: 2px; border: 1px solid; }
</style>

<div class="pi0-root">

<div class="pi0-hero">
  <h1>π0 模型架构深度解析</h1>
  <p>基于 openpi 源码逐层拆解 VLA 的预处理、Token 拼接、Attention Mask、Flow Matching 训练与推理。所有维度来自代码 <code style="color:#fbbf24">src/openpi/models/pi0.py</code> 与相关模块。</p>
</div>

<div class="pi0-toc">
<strong>目录</strong>
<ol>
  <li><a href="#s1">整体架构</a></li>
  <li><a href="#s2">输入数据格式</a></li>
  <li><a href="#s3">SigLIP 图像编码</a></li>
  <li><a href="#s4">PaliGemma 语言编码</a></li>
  <li><a href="#s5">Prefix 拼接（embed_prefix）</a></li>
  <li><a href="#s6">Suffix 拼接（embed_suffix）</a></li>
  <li><a href="#s7">完整 Token 序列</a></li>
  <li><a href="#s8">Attention Mask 设计</a></li>
  <li><a href="#s9">Flow Matching 训练目标</a></li>
  <li><a href="#s10">推理：Euler ODE Solver</a></li>
  <li><a href="#s11">数据 Pipeline</a></li>
  <li><a href="#s12">π0 vs π0.5 差异</a></li>
</ol>
</div>

<h2 id="s1">1. 整体架构</h2>

<p>π0 本质上是 <strong>双专家 Transformer</strong>（Mixture of Experts with two weight groups），两个专家共享一套 attention 操作，但各自拥有独立的 QKV、FFN、LayerNorm 参数。这个设计来自 <code class="pi0-code-inline">gemma.py</code> 里的 <code class="pi0-code-inline">Module(configs=[paligemma_config, action_expert_config])</code>。</p>

<div class="pi0-grid">
  <div class="pi0-card">
    <h4 style="margin-top:0">PaliGemma (VLM expert)</h4>
    <div>variant: <span class="pi0-badge pi0-param">gemma_2b</span></div>
    <div>width: <span class="pi0-badge pi0-dim">2048</span></div>
    <div>depth: <span class="pi0-badge pi0-dim">18</span></div>
    <div>mlp_dim: <span class="pi0-badge pi0-dim">16384</span></div>
    <div>num_heads: <span class="pi0-badge pi0-dim">8</span></div>
    <div>num_kv_heads: <span class="pi0-badge pi0-dim">1</span> (MQA)</div>
    <div>head_dim: <span class="pi0-badge pi0-dim">256</span></div>
    <div style="margin-top:0.5rem; color:#94a3b8">处理 image + language tokens（prefix）</div>
  </div>
  <div class="pi0-card">
    <h4 style="margin-top:0">Action Expert</h4>
    <div>variant: <span class="pi0-badge pi0-param">gemma_300m</span></div>
    <div>width: <span class="pi0-badge pi0-dim">1024</span></div>
    <div>depth: <span class="pi0-badge pi0-dim">18</span></div>
    <div>mlp_dim: <span class="pi0-badge pi0-dim">4096</span></div>
    <div>num_heads: <span class="pi0-badge pi0-dim">8</span></div>
    <div>num_kv_heads: <span class="pi0-badge pi0-dim">1</span> (MQA)</div>
    <div>head_dim: <span class="pi0-badge pi0-dim">256</span></div>
    <div style="margin-top:0.5rem; color:#94a3b8">处理 state + action tokens（suffix）</div>
  </div>
</div>

<div class="pi0-callout pi0-callout-tip">
<strong>关键设计</strong>：两个专家的 <code class="pi0-code-inline">head_dim / num_heads / num_kv_heads</code> 必须一致，因为 attention 是对拼接后的全序列做的。不同的是 <code class="pi0-code-inline">width</code> 和 <code class="pi0-code-inline">mlp_dim</code> —— 每个 token 属于哪个专家，就走哪个专家的 QKV 投影与 FFN。
</div>

<h3>原理：为什么是双专家而不是单一大模型？</h3>

<div class="pi0-callout">
<strong>动机</strong>：VLM 预训练（PaliGemma 2B）已经在大规模图文数据上学到了强大的感知-语言表征。但动作空间分布和自然语言完全不同 —— 动作是连续的、低维的、频率很高（50Hz）。如果让 2B 的 VLM 直接输出动作，一方面参数量浪费（动作空间不需要那么多容量），另一方面会破坏预训练的感知能力。
<br/><br/>
<strong>解决方案</strong>：Action Expert（300M）是专门为动作生成新初始化的一路"小专家"，与 VLM 共享 attention（信息可以流通），但各自有独立的 FFN 参数。VLM 的参数可以冻结或以极小学习率微调，Action Expert 从头训练。这样感知能力被保留，动作能力被独立优化。
<br/><br/>
<strong>实现细节</strong>：attention 的 Q, K, V 是拼接后一起算的（<code class="pi0-code-inline">q, k, v = jnp.concatenate(qkvs, axis=1)</code>），所以 image token 生成的 Q 能 attend 到 action token 生成的 K —— 跨专家的信息流通是天然的。而 FFN 是分段做的（<code class="pi0-code-inline">for i, (x, config) in enumerate(zip(xs, self.configs))</code>），每段 token 走自己的 FFN 权重。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">π0 整体数据流（训练时）</div>
<svg class="pi0-arch-svg" viewBox="0 0 880 560" xmlns="http://www.w3.org/2000/svg">
  <!-- Input row -->
  <rect x="20" y="20" width="160" height="60" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="100" y="42" text-anchor="middle" class="title">3× Camera RGB</text>
  <text x="100" y="58" text-anchor="middle" class="dim">[B, 224, 224, 3]</text>
  <text x="100" y="72" text-anchor="middle" class="lbl">uint8 → [-1, 1]</text>

  <rect x="200" y="20" width="160" height="60" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="280" y="42" text-anchor="middle" class="title">Language Prompt</text>
  <text x="280" y="58" text-anchor="middle" class="dim">str → int32[B, 48]</text>
  <text x="280" y="72" text-anchor="middle" class="lbl">SentencePiece</text>

  <rect x="380" y="20" width="160" height="60" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="460" y="42" text-anchor="middle" class="title">Robot State</text>
  <text x="460" y="58" text-anchor="middle" class="dim">[B, 32]</text>
  <text x="460" y="72" text-anchor="middle" class="lbl">zero-pad + norm</text>

  <rect x="560" y="20" width="160" height="60" rx="6" fill="#7f1d1d" stroke="#ef4444"/>
  <text x="640" y="42" text-anchor="middle" class="title">Target Actions</text>
  <text x="640" y="58" text-anchor="middle" class="dim">[B, 50, 32]</text>
  <text x="640" y="72" text-anchor="middle" class="lbl">(training only)</text>

  <rect x="740" y="20" width="120" height="60" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="800" y="42" text-anchor="middle" class="title">t ~ Beta(1.5, 1)</text>
  <text x="800" y="58" text-anchor="middle" class="dim">∈ (0.001, 1.0)</text>
  <text x="800" y="72" text-anchor="middle" class="lbl">+ noise ~ N(0,I)</text>

  <!-- Arrows down -->
  <line x1="100" y1="80" x2="100" y2="110" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>
  <line x1="280" y1="80" x2="280" y2="110" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>
  <line x1="460" y1="80" x2="460" y2="230" stroke="#64748b" stroke-width="1.5" stroke-dasharray="4,3" marker-end="url(#arr)"/>
  <line x1="640" y1="80" x2="640" y2="195" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>
  <line x1="800" y1="80" x2="800" y2="195" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>

  <!-- SigLIP box -->
  <rect x="20" y="110" width="160" height="70" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="100" y="132" text-anchor="middle" class="title">SigLIP So400m/14</text>
  <text x="100" y="148" text-anchor="middle" class="lbl">27-layer ViT</text>
  <text x="100" y="164" text-anchor="middle" class="dim">→ [B, 256, 1152]</text>
  <text x="100" y="178" text-anchor="middle" class="lbl">head proj → 2048</text>

  <!-- Embedder box -->
  <rect x="200" y="110" width="160" height="70" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="280" y="132" text-anchor="middle" class="title">PaliGemma Embedder</text>
  <text x="280" y="148" text-anchor="middle" class="lbl">vocab=257152, d=2048</text>
  <text x="280" y="164" text-anchor="middle" class="dim">→ [B, 48, 2048]</text>
  <text x="280" y="178" text-anchor="middle" class="lbl">× √2048</text>

  <!-- x_t mixer -->
  <rect x="560" y="195" width="300" height="60" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="710" y="218" text-anchor="middle" class="title">x_t = t·noise + (1-t)·actions</text>
  <text x="710" y="234" text-anchor="middle" class="lbl">u_t = noise - actions   (target velocity)</text>
  <text x="710" y="248" text-anchor="middle" class="dim">[B, 50, 32]</text>

  <!-- Merge arrows -->
  <line x1="100" y1="180" x2="100" y2="280" stroke="#64748b" stroke-width="1.5"/>
  <line x1="280" y1="180" x2="280" y2="280" stroke="#64748b" stroke-width="1.5"/>
  <line x1="460" y1="230" x2="460" y2="280" stroke="#64748b" stroke-width="1.5"/>
  <line x1="710" y1="255" x2="710" y2="280" stroke="#64748b" stroke-width="1.5"/>

  <!-- Token concat strip -->
  <rect x="20" y="280" width="840" height="40" rx="4" fill="#0f1117" stroke="#2d3748"/>
  <rect x="25" y="285" width="220" height="30" rx="3" fill="#1e40af" stroke="#3b82f6"/>
  <text x="135" y="304" text-anchor="middle" class="lbl">image tokens × 768 (3×256)</text>
  <rect x="250" y="285" width="90" height="30" rx="3" fill="#6b21a8" stroke="#a855f7"/>
  <text x="295" y="304" text-anchor="middle" class="lbl">lang × 48</text>
  <rect x="345" y="285" width="35" height="30" rx="3" fill="#9a3412" stroke="#ea580c"/>
  <text x="362" y="304" text-anchor="middle" class="lbl">st</text>
  <rect x="385" y="285" width="470" height="30" rx="3" fill="#166534" stroke="#22c55e"/>
  <text x="620" y="304" text-anchor="middle" class="lbl">action tokens × 50 (noisy x_t + time)</text>

  <!-- Labels -->
  <text x="135" y="340" text-anchor="middle" class="lbl">─── PaliGemma expert (d=2048) ───</text>
  <text x="295" y="340" text-anchor="middle" class="lbl"></text>
  <text x="620" y="340" text-anchor="middle" class="lbl">─── Action expert (d=1024) ───</text>

  <!-- Double-expert transformer -->
  <line x1="440" y1="345" x2="440" y2="370" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>
  <rect x="140" y="370" width="600" height="80" rx="8" fill="#4c1d95" stroke="#a855f7" stroke-width="1.5"/>
  <text x="440" y="395" text-anchor="middle" class="title">Double-Expert Transformer (18 layers, shared attention)</text>
  <text x="440" y="412" text-anchor="middle" class="lbl">MQA (8 heads Q, 1 head KV, head_dim=256) · RoPE · RMSNorm</text>
  <text x="440" y="428" text-anchor="middle" class="lbl">Per-layer: QKV concat → softmax(QK^T / √d + mask) · V → split to experts → expert FFN</text>
  <text x="440" y="444" text-anchor="middle" class="dim">π0.5 only: adaRMSNorm conditioned on timestep (action expert only)</text>

  <!-- Output -->
  <line x1="620" y1="450" x2="620" y2="475" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr)"/>
  <rect x="500" y="475" width="240" height="50" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="620" y="497" text-anchor="middle" class="title">action_out_proj: Linear(1024→32)</text>
  <text x="620" y="513" text-anchor="middle" class="dim">v_t: [B, 50, 32] (predicted velocity)</text>

  <!-- Loss -->
  <line x1="740" y1="500" x2="810" y2="500" stroke="#ef4444" stroke-width="1.5" marker-end="url(#arr-red)"/>
  <rect x="760" y="475" width="100" height="50" rx="6" fill="#7f1d1d" stroke="#ef4444"/>
  <text x="810" y="497" text-anchor="middle" class="title">MSE Loss</text>
  <text x="810" y="513" text-anchor="middle" class="lbl">‖v_t − u_t‖²</text>

  <defs>
    <marker id="arr" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#64748b"/>
    </marker>
    <marker id="arr-red" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#ef4444"/>
    </marker>
  </defs>
</svg>
<div class="pi0-legend">
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#0c4a6e;border-color:#0ea5e9"></span>输入</div>
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#1e3a8a;border-color:#3b82f6"></span>编码器</div>
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#4c1d95;border-color:#a855f7"></span>Transformer</div>
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#064e3b;border-color:#10b981"></span>输出投影</div>
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#78350f;border-color:#f59e0b"></span>Flow Matching</div>
  <div class="pi0-legend-item"><span class="pi0-legend-box" style="background:#7f1d1d;border-color:#ef4444"></span>训练目标</div>
</div>
</div>

<h2 id="s2">2. 输入数据格式</h2>

<p>来自 <code class="pi0-code-inline">src/openpi/models/model.py</code> 的 <code class="pi0-code-inline">Observation</code> 结构：</p>

<pre><code><span class="tok-kw">class</span> <span class="tok-fn">Observation</span>:
    images: dict[str, Float[Array, <span class="tok-str">"*b h w c"</span>]]    <span class="tok-cm"># 3 cameras, (224, 224, 3)</span>
    image_masks: dict[str, Bool[Array, <span class="tok-str">"*b"</span>]]       <span class="tok-cm"># 是否有效</span>
    state: Float[Array, <span class="tok-str">"*b s"</span>]                     <span class="tok-cm"># (B, 32)</span>
    tokenized_prompt: Int[Array, <span class="tok-str">"*b l"</span>]              <span class="tok-cm"># (B, 48) or (B, 200)</span>
    tokenized_prompt_mask: Bool[Array, <span class="tok-str">"*b l"</span>]
</code></pre>

<table class="pi0-table">
  <tr><th>字段</th><th>Shape</th><th>Dtype</th><th>预处理</th></tr>
  <tr><td>images[base_0_rgb]</td><td><span class="pi0-badge pi0-shape">[B, 224, 224, 3]</span></td><td>float32</td><td><code class="pi0-code-inline">uint8/255*2-1</code> → [-1, 1]</td></tr>
  <tr><td>images[left_wrist_0_rgb]</td><td><span class="pi0-badge pi0-shape">[B, 224, 224, 3]</span></td><td>float32</td><td>同上</td></tr>
  <tr><td>images[right_wrist_0_rgb]</td><td><span class="pi0-badge pi0-shape">[B, 224, 224, 3]</span></td><td>float32</td><td>同上</td></tr>
  <tr><td>state</td><td><span class="pi0-badge pi0-shape">[B, 32]</span></td><td>float32</td><td>zero-padded 到 action_dim=32</td></tr>
  <tr><td>tokenized_prompt</td><td><span class="pi0-badge pi0-shape">[B, 48]</span> / <span class="pi0-badge pi0-shape">[B, 200]</span></td><td>int32</td><td>SentencePiece + padding</td></tr>
  <tr><td>actions (train)</td><td><span class="pi0-badge pi0-shape">[B, 50, 32]</span></td><td>float32</td><td>normalize + pad</td></tr>
</table>

<p>图像归一化逻辑（<code class="pi0-code-inline">Observation.from_dict</code>）：</p>

<pre><code><span class="tok-kw">for</span> key <span class="tok-kw">in</span> data[<span class="tok-str">"image"</span>]:
    <span class="tok-kw">if</span> data[<span class="tok-str">"image"</span>][key].dtype == np.uint8:
        data[<span class="tok-str">"image"</span>][key] = data[<span class="tok-str">"image"</span>][key].astype(np.float32) / <span class="tok-num">255.0</span> * <span class="tok-num">2.0</span> - <span class="tok-num">1.0</span>
</code></pre>

<h2 id="s3">3. SigLIP 图像编码</h2>

<p>三个相机视角共享同一个 SigLIP backbone（<code class="pi0-code-inline">siglip.Module</code>, variant=<code class="pi0-code-inline">"So400m/14"</code>）。每个相机独立前向一次。</p>

<div class="pi0-card">
<strong>SigLIP So400m 配置</strong>
<ul style="margin: 0.5rem 0">
  <li>width = <span class="pi0-badge pi0-dim">1152</span></li>
  <li>depth = <span class="pi0-badge pi0-dim">27</span></li>
  <li>mlp_dim = <span class="pi0-badge pi0-dim">4304</span></li>
  <li>num_heads = <span class="pi0-badge pi0-dim">16</span></li>
  <li>patch_size = <span class="pi0-badge pi0-dim">14</span></li>
  <li>pool_type = <span class="pi0-badge pi0-param">"none"</span>（保留所有 patch tokens，不做池化）</li>
  <li>num_classes = paligemma_config.width = <span class="pi0-badge pi0-dim">2048</span>（最后 head 投影）</li>
</ul>
</div>

<p><strong>前向流程：</strong></p>

<div class="pi0-flow">image: [B, 224, 224, 3]
    │
    ▼   Conv2d(1152, kernel=14, stride=14, padding=VALID)
    │
[B, 16, 16, 1152]                       # 224/14 = 16
    │
    ▼   reshape
    │
[B, 256, 1152]                          # 16×16 = 256 patches
    │
    ▼   + sincos2d positional embedding
    │
[B, 256, 1152]
    │
    ▼   27 × Transformer Encoder Block  (MHSA + MLP, LayerNorm pre-norm)
    │
[B, 256, 1152]                          # encoded
    │
    ▼   Dense(num_classes=2048)         # head 投影到 PaliGemma width
    │
[B, 256, 2048]                          # image tokens，直接进 PaliGemma expert</div>

<p>3 个相机 → <span class="pi0-badge pi0-dim">3 × 256 = 768</span> 个 image tokens，每个 <span class="pi0-badge pi0-dim">dim=2048</span>。</p>

<h3>原理：为什么用 SigLIP 而不是 CLIP？为什么 patch=14 不是 16？</h3>

<div class="pi0-callout">
<strong>SigLIP vs CLIP</strong>：SigLIP 把 CLIP 的 softmax contrastive loss 换成了 sigmoid loss —— 每个图文对独立判定正负，不需要 batch 内归一化。在 batch size 很大时（本模型在 400M 图文对上预训练），这个变化带来更稳定的训练和更强的零样本迁移能力。对 VLA 尤其重要，因为机器人场景的视觉域和 CLIP 训练数据差距大。
<br/><br/>
<strong>So400m</strong>：Shape-Optimized 400M 参数的变体（width=1152, depth=27），在参数效率上优于标准 L/16（大约 304M 参数但性能更差）。
<br/><br/>
<strong>patch=14, image=224</strong>：224/14 = 16，得到 16×16=256 个 patch。比 patch=16（196 patch）多 30% 的 token，对机器人场景中精细操作（如抓取小物体）很关键。patch=14 在 DINOv2 和 SigLIP 都成为主流。
<br/><br/>
<strong>pool_type="none"</strong>：标准 ViT 分类用 <code class="pi0-code-inline">[CLS]</code> token 或 global average pooling，但 VLA 需要保留空间信息（机器人需要知道物体在画面哪里），所以所有 256 个 patch token 都进入 LLM。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">SigLIP 单相机前向流程（每个相机独立执行一次）</div>
<svg class="pi0-arch-svg" viewBox="0 0 860 460" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="20" width="140" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="90" y="42" text-anchor="middle" class="title">RGB Image</text>
  <text x="90" y="58" text-anchor="middle" class="dim">[B, 224, 224, 3]</text>

  <line x1="90" y1="70" x2="90" y2="95" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>
  <text x="250" y="82" class="arrow-label" fill="#fbbf24">Conv2d(width=1152, kernel=14, stride=14, VALID padding)</text>

  <rect x="20" y="95" width="140" height="50" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="90" y="117" text-anchor="middle" class="title">Patch Feature Map</text>
  <text x="90" y="133" text-anchor="middle" class="dim">[B, 16, 16, 1152]</text>

  <line x1="90" y1="145" x2="90" y2="170" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>
  <text x="250" y="157" fill="#fbbf24" style="font-size:10px;font-style:italic">reshape: (16,16) → 256 tokens</text>

  <rect x="20" y="170" width="140" height="50" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="90" y="192" text-anchor="middle" class="title">Patch Tokens</text>
  <text x="90" y="208" text-anchor="middle" class="dim">[B, 256, 1152]</text>

  <line x1="90" y1="220" x2="90" y2="245" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>
  <text x="250" y="232" fill="#fbbf24" style="font-size:10px;font-style:italic">+ posemb_sincos_2d (sin/cos on x,y)</text>

  <rect x="20" y="245" width="140" height="50" rx="6" fill="#4c1d95" stroke="#a855f7"/>
  <text x="90" y="267" text-anchor="middle" class="title">Encoder Input</text>
  <text x="90" y="283" text-anchor="middle" class="dim">[B, 256, 1152]</text>

  <!-- Right side: encoder stack -->
  <rect x="480" y="95" width="360" height="200" rx="8" fill="#4c1d95" stroke="#a855f7" stroke-width="1.5"/>
  <text x="660" y="118" text-anchor="middle" class="title">27 × Encoder Block (pre-norm)</text>

  <rect x="510" y="135" width="300" height="30" rx="4" fill="#312e81" stroke="#6366f1"/>
  <text x="660" y="155" text-anchor="middle" class="lbl">LayerNorm → MHSA (16 heads) → + residual</text>

  <rect x="510" y="175" width="300" height="30" rx="4" fill="#312e81" stroke="#6366f1"/>
  <text x="660" y="195" text-anchor="middle" class="lbl">LayerNorm → MLP (1152→4304→1152, GELU) → + residual</text>

  <text x="660" y="230" text-anchor="middle" class="lbl">... repeat 27× ...</text>

  <rect x="510" y="245" width="300" height="30" rx="4" fill="#1e293b" stroke="#475569"/>
  <text x="660" y="265" text-anchor="middle" class="lbl">encoder_norm (final LayerNorm)</text>

  <text x="660" y="285" text-anchor="middle" class="dim">Output: [B, 256, 1152]</text>

  <line x1="160" y1="270" x2="480" y2="195" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>

  <!-- Head projection -->
  <line x1="660" y1="295" x2="660" y2="325" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="520" y="325" width="280" height="50" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="660" y="347" text-anchor="middle" class="title">Head: Dense(num_classes=2048)</text>
  <text x="660" y="363" text-anchor="middle" class="dim">[B, 256, 2048]   ← PaliGemma width</text>

  <line x1="660" y1="375" x2="660" y2="405" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="450" y="405" width="420" height="40" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="660" y="425" text-anchor="middle" class="title">Image tokens for one camera</text>
  <text x="660" y="440" text-anchor="middle" class="dim">3 cameras → [B, 768, 2048] after concat in embed_prefix</text>

  <defs>
    <marker id="arr2" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#64748b"/>
    </marker>
  </defs>
</svg>
</div>

<h2 id="s4">4. PaliGemma 语言编码</h2>

<p>Tokenizer 是 SentencePiece（从 <code class="pi0-code-inline">gs://big_vision/paligemma_tokenizer.model</code> 下载），词表大小 <span class="pi0-badge pi0-dim">257152</span>。</p>

<h4>π0 的 prompt 格式</h4>

<pre><code>cleaned_text = prompt.strip().replace(<span class="tok-str">"_"</span>, <span class="tok-str">" "</span>).replace(<span class="tok-str">"\n"</span>, <span class="tok-str">" "</span>)
tokens = tokenizer.encode(cleaned_text, add_bos=<span class="tok-kw">True</span>) + tokenizer.encode(<span class="tok-str">"\n"</span>)
<span class="tok-cm"># 例如: "pick up the red cube" → [BOS, pick, up, the, red, cube, \n]</span>
<span class="tok-cm"># pad 到 max_token_len=48</span>
</code></pre>

<h4>π0.5 的 prompt 格式（含离散化 state）</h4>

<pre><code><span class="tok-cm"># state 离散化：每一维映射到 [0, 255] 的 bin index</span>
discretized_state = np.digitize(state, bins=np.linspace(-<span class="tok-num">1</span>, <span class="tok-num">1</span>, <span class="tok-num">257</span>)[:-<span class="tok-num">1</span>]) - <span class="tok-num">1</span>
state_str = <span class="tok-str">" "</span>.join(map(str, discretized_state))
full_prompt = <span class="tok-str">f"Task: {cleaned_text}, State: {state_str};\nAction: "</span>
tokens = tokenizer.encode(full_prompt, add_bos=<span class="tok-kw">True</span>)
<span class="tok-cm"># pad 到 max_token_len=200（比 π0 长很多，因为 state 占了不少 token）</span>
</code></pre>

<p><strong>Embedding 层（<code class="pi0-code-inline">gemma.Embedder</code>）</strong>：</p>

<pre><code>x = self.input_embedding_table[tokens]   <span class="tok-cm"># (B, L, 2048)</span>
x *= sqrt(<span class="tok-num">2048</span>).astype(x.dtype)         <span class="tok-cm"># 缩放，Transformer 惯例</span>
</code></pre>

<h3>原理：为什么 embedding 要乘 √d？为什么 π0.5 把 state 塞进 language token？</h3>

<div class="pi0-callout">
<strong>√d 缩放的来源</strong>：Transformer 原论文中，embedding 初始化为 <code class="pi0-code-inline">N(0, 1/d)</code>，所以 embedding 的模长量级是 <code class="pi0-code-inline">O(1/√d)</code>。而 positional encoding 是 sin/cos 函数，模长为 O(1)。如果直接相加，positional 信号会压过 embedding。乘 √d 让 embedding 模长变成 O(1)，与 pos emb 平衡。虽然后续做了 RMSNorm，这个缩放对训练早期的信号量级仍然重要。
<br/><br/>
<strong>π0.5 把 state 离散化进 language</strong>：核心动机是统一表征。π0 的 state 是 32 维连续向量，通过一个独立的 Linear 投影成 1 个 token；这个 token 只能和 action expert 交互，和 VLM 的感知分离。π0.5 把每一维 state 离散化为 256 个 bin（"26 14 203 ..." 这样的整数序列），用 PaliGemma 的词表直接 tokenize。结果：
<br/>&nbsp;&nbsp;•&nbsp;state 能参与 image+language 的双向 attention，让感知模块"看到"机器人自己的姿态；
<br/>&nbsp;&nbsp;•&nbsp;所有输入都走同一个 embedding 表，语义空间统一；
<br/>&nbsp;&nbsp;•&nbsp;代价是 max_token_len 从 48 变成 200（state 占很多 token）。
</div>

<h2 id="s5">5. Prefix 拼接（embed_prefix）</h2>

<p>来自 <code class="pi0-code-inline">pi0.py</code> 的 <code class="pi0-code-inline">embed_prefix</code>：</p>

<pre><code><span class="tok-kw">def</span> <span class="tok-fn">embed_prefix</span>(self, obs):
    tokens, input_mask, ar_mask = [], [], []

    <span class="tok-cm"># 1. 3 个相机的 image tokens</span>
    <span class="tok-kw">for</span> name <span class="tok-kw">in</span> obs.images:
        image_tokens, _ = self.PaliGemma.img(obs.images[name], train=<span class="tok-kw">False</span>)   <span class="tok-cm"># (B, 256, 2048)</span>
        tokens.append(image_tokens)
        input_mask.append(repeat(obs.image_masks[name], <span class="tok-str">"b -> b s"</span>, s=<span class="tok-num">256</span>))
        ar_mask += [<span class="tok-kw">False</span>] * <span class="tok-num">256</span>   <span class="tok-cm"># 图像内部双向 attention</span>

    <span class="tok-cm"># 2. 语言 tokens</span>
    <span class="tok-kw">if</span> obs.tokenized_prompt <span class="tok-kw">is</span> <span class="tok-kw">not</span> <span class="tok-kw">None</span>:
        tokenized_inputs = self.PaliGemma.llm(obs.tokenized_prompt, method=<span class="tok-str">"embed"</span>)  <span class="tok-cm"># (B, 48, 2048)</span>
        tokens.append(tokenized_inputs)
        input_mask.append(obs.tokenized_prompt_mask)
        ar_mask += [<span class="tok-kw">False</span>] * tokenized_inputs.shape[<span class="tok-num">1</span>]  <span class="tok-cm"># 图像+语言全双向</span>

    tokens = concat(tokens, axis=<span class="tok-num">1</span>)           <span class="tok-cm"># (B, 768+48, 2048)</span>
    <span class="tok-kw">return</span> tokens, input_mask, ar_mask
</code></pre>

<div class="pi0-seq">
  <div class="pi0-tok pi0-tok-img">base_rgb<br/>256 tok</div>
  <div class="pi0-tok pi0-tok-img">left_wrist<br/>256 tok</div>
  <div class="pi0-tok pi0-tok-img">right_wrist<br/>256 tok</div>
  <div class="pi0-tok pi0-tok-lang">language<br/>48 tok</div>
</div>

<p>Prefix 总长：<span class="pi0-badge pi0-dim">768 + 48 = 816</span> tokens，每个 dim=<span class="tok-num">2048</span>。所有 <code class="pi0-code-inline">ar_mask=False</code>，意味着这一段内部是完全双向的 prefix-LM attention。</p>

<h3>原理：为什么 prefix 要全双向？三个相机 token 如何区分空间位置？</h3>

<div class="pi0-callout">
<strong>双向的必要性</strong>：prefix 是感知-语言侧，属于"理解"任务。image patches 之间本来就有空间关系（上下左右），语言 token 之间有语法关系，image 和 language 之间有指代关系（"the red cube" 对应画面里哪个 patch）。这些都是需要双向推理的。如果做成 causal，language 的第一个 token 就看不到后面的词，也看不到 image —— 信息被人为限制。
<br/><br/>
<strong>三个相机的空间区分</strong>：代码里三个相机的 image tokens 是直接 concat 的（没有额外的 "camera embedding"），模型怎么区分哪 256 个 token 属于哪个相机？
<br/>&nbsp;&nbsp;•&nbsp;SigLIP 内部已经加了 sincos2d positional embedding，同一相机内的 16×16 patch 有空间信号；
<br/>&nbsp;&nbsp;•&nbsp;跨相机的区分靠 RoPE —— 拼接后每个 token 的 <code class="pi0-code-inline">position</code> 是 0, 1, ..., 815，三个相机处于不同 position 区间（0–255, 256–511, 512–767），RoPE 给不同区间施加不同的相位旋转，让相机身份在 attention 中隐式编码；
<br/>&nbsp;&nbsp;•&nbsp;更重要的是 base / left_wrist / right_wrist 在训练数据中是固定顺序，模型可以从数据里学到这种隐式绑定。
<br/><br/>
<strong>image_masks 的作用</strong>：如果某个相机数据缺失（比如 wrist 相机脱落），对应 mask 设为 False，那 256 个 token 就不参与 attention（被 <code class="pi0-code-inline">valid_mask</code> 屏蔽）。训练数据里故意随机 drop 相机，让模型学会在缺失相机时仍能决策。
</div>

<h2 id="s6">6. Suffix 拼接（embed_suffix）</h2>

<h3>π0 模式（无 adaRMS）</h3>

<pre><code><span class="tok-cm"># 1. state token（连续向量）</span>
state_token = self.state_proj(obs.state)[:, <span class="tok-kw">None</span>, :]     <span class="tok-cm"># Linear(32 → 1024) → (B, 1, 1024)</span>
ar_mask += [<span class="tok-kw">True</span>]                                          <span class="tok-cm"># causal boundary</span>

<span class="tok-cm"># 2. action tokens</span>
action_tokens = self.action_in_proj(noisy_actions)           <span class="tok-cm"># Linear(32 → 1024) → (B, 50, 1024)</span>

<span class="tok-cm"># 3. timestep embedding（sincos, dim=1024, period ∈ [4e-3, 4.0]）</span>
time_emb = posemb_sincos(timestep, <span class="tok-num">1024</span>, min_period=<span class="tok-num">4e-3</span>, max_period=<span class="tok-num">4.0</span>)   <span class="tok-cm"># (B, 1024)</span>

<span class="tok-cm"># 4. timestep 与 action 融合（concat + MLP）</span>
time_tokens = repeat(time_emb, <span class="tok-str">"b d -> b s d"</span>, s=<span class="tok-num">50</span>)                       <span class="tok-cm"># (B, 50, 1024)</span>
action_time_tokens = concat([action_tokens, time_tokens], axis=-<span class="tok-num">1</span>)           <span class="tok-cm"># (B, 50, 2048)</span>
x = self.action_time_mlp_in(action_time_tokens)                              <span class="tok-cm"># Linear(2048 → 1024)</span>
x = nnx.swish(x)
x = self.action_time_mlp_out(x)                                              <span class="tok-cm"># Linear(1024 → 1024)</span>
x = nnx.swish(x)                                                             <span class="tok-cm"># (B, 50, 1024)</span>

ar_mask += [<span class="tok-kw">True</span>] + [<span class="tok-kw">False</span>] * <span class="tok-num">49</span>   <span class="tok-cm"># 第一个 action token 是 causal boundary</span>
adarms_cond = <span class="tok-kw">None</span>                     <span class="tok-cm"># π0 不用 adaRMS</span>
</code></pre>

<h3>π0.5 模式（adaRMS conditioning）</h3>

<pre><code><span class="tok-cm"># 1. 无 state token（state 已编码进 language tokens）</span>

<span class="tok-cm"># 2. action tokens（直接投影，不混 timestep）</span>
action_tokens = self.action_in_proj(noisy_actions)           <span class="tok-cm"># (B, 50, 1024)</span>

<span class="tok-cm"># 3. timestep → time MLP → adaRMS conditioning</span>
time_emb = posemb_sincos(timestep, <span class="tok-num">1024</span>, min_period=<span class="tok-num">4e-3</span>, max_period=<span class="tok-num">4.0</span>)
time_emb = self.time_mlp_in(time_emb)      <span class="tok-cm"># Linear(1024 → 1024)</span>
time_emb = nnx.swish(time_emb)
time_emb = self.time_mlp_out(time_emb)     <span class="tok-cm"># Linear(1024 → 1024)</span>
time_emb = nnx.swish(time_emb)             <span class="tok-cm"># (B, 1024)</span>
adarms_cond = time_emb                     <span class="tok-cm"># 传给 action expert 的每层 RMSNorm</span>
</code></pre>

<div class="pi0-callout">
<strong>adaRMSNorm 公式</strong>（在 <code class="pi0-code-inline">gemma.RMSNorm</code>）：给定条件向量 <code class="pi0-code-inline">cond</code>，
<pre style="margin-top:0.5rem"><code>modulation = Dense(width * <span class="tok-num">3</span>, init=zeros)(cond)
scale, shift, gate = split(modulation, <span class="tok-num">3</span>)
x_normed = rms_norm(x) * (<span class="tok-num">1</span> + scale) + shift
<span class="tok-cm"># gate 用于残差：x + gate * f(x_normed)</span></code></pre>
这让 timestep 能直接调制 action expert 每层的规范化 + 残差权重，比 concat 式注入更强。
</div>

<h3>原理：concat-MLP vs adaRMS 谁更强？为什么 sincos 周期 [4e-3, 4.0]？</h3>

<div class="pi0-callout">
<strong>π0 的 concat-MLP 注入</strong>：timestep embedding 与 action token concat 后过两层 MLP，timestep 信号只在"输入端"注入一次。后续 18 层 Transformer 必须靠自己把这个信号传递到深层。
<br/><br/>
<strong>π0.5 的 adaRMS 注入</strong>：DiT (Diffusion Transformer) 的标准做法。timestep 通过 zero-init 的 Dense 生成 (scale, shift, gate) 三个调制向量，在每一层 RMSNorm 后做 affine 变换，并控制残差权重。"zero-init" 意味着初始时 modulation = 0，模型行为退化为标准 RMSNorm，训练时逐步学到非零调制。这种"逐层注入 + zero-init 安全启动"是 DiT 在生成质量上压过 U-Net 的关键，π0.5 直接借用过来。
<br/><br/>
<strong>sincos 周期 [4e-3, 4.0]</strong>：来自 <code class="pi0-code-inline">posemb_sincos(timestep, 1024, min_period=4e-3, max_period=4.0)</code>。1024 维分成 512 对 sin/cos，周期在 [4e-3, 4.0] 上对数均匀分布。timestep 取值范围是 (0.001, 1.0)：
<br/>&nbsp;&nbsp;•&nbsp;最小周期 4e-3 ≈ 4×取值精度，对应高频维度，可以分辨 t 的微小变化；
<br/>&nbsp;&nbsp;•&nbsp;最大周期 4.0 大于取值范围 1.0，对应低频维度，提供 t 在区间内的"绝对位置"信号。
<br/>这样高低频维度组合，让模型能同时辨别 "t 是哪个量级" 和 "t 在量级内具体位于哪里"。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">Suffix 拼接流程对比：π0 vs π0.5</div>
<svg class="pi0-arch-svg" viewBox="0 0 880 380" xmlns="http://www.w3.org/2000/svg">
  <!-- π0 column -->
  <text x="220" y="20" text-anchor="middle" class="title" style="font-size:13px">π0 模式</text>
  <rect x="20" y="35" width="180" height="45" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="110" y="55" text-anchor="middle" class="title">state [B, 32]</text>
  <text x="110" y="70" text-anchor="middle" class="lbl">connected vector</text>

  <rect x="240" y="35" width="180" height="45" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="330" y="55" text-anchor="middle" class="title">noisy x_t [B, 50, 32]</text>
  <text x="330" y="70" text-anchor="middle" class="lbl">+ time t [B]</text>

  <line x1="110" y1="80" x2="110" y2="105" stroke="#64748b" marker-end="url(#arr3)"/>
  <line x1="330" y1="80" x2="330" y2="105" stroke="#64748b" marker-end="url(#arr3)"/>

  <rect x="20" y="105" width="180" height="40" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="110" y="123" text-anchor="middle" class="title">state_proj: Linear(32→1024)</text>
  <text x="110" y="138" text-anchor="middle" class="dim">[B, 1, 1024]</text>

  <rect x="240" y="105" width="180" height="40" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="330" y="123" text-anchor="middle" class="title">action_in_proj: Linear(32→1024)</text>
  <text x="330" y="138" text-anchor="middle" class="dim">[B, 50, 1024]</text>

  <line x1="330" y1="145" x2="330" y2="170" stroke="#64748b" marker-end="url(#arr3)"/>
  <rect x="240" y="170" width="180" height="40" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="330" y="188" text-anchor="middle" class="title">+ sincos(t, dim=1024)</text>
  <text x="330" y="203" text-anchor="middle" class="lbl">repeat → concat → [B,50,2048]</text>

  <line x1="330" y1="210" x2="330" y2="235" stroke="#64748b" marker-end="url(#arr3)"/>
  <rect x="240" y="235" width="180" height="55" rx="6" fill="#4c1d95" stroke="#a855f7"/>
  <text x="330" y="253" text-anchor="middle" class="title">action_time_mlp_in/out</text>
  <text x="330" y="268" text-anchor="middle" class="lbl">Linear(2048→1024)+swish</text>
  <text x="330" y="283" text-anchor="middle" class="lbl">Linear(1024→1024)+swish</text>

  <line x1="110" y1="145" x2="110" y2="320" stroke="#64748b" marker-end="url(#arr3)"/>
  <line x1="330" y1="290" x2="330" y2="320" stroke="#64748b" marker-end="url(#arr3)"/>

  <rect x="20" y="320" width="400" height="40" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="220" y="338" text-anchor="middle" class="title">Suffix tokens for π0</text>
  <text x="220" y="354" text-anchor="middle" class="dim">[state×1 | action×50] = 51 tokens, dim=1024, ar_mask=[T, T, F×49]</text>

  <!-- π0.5 column -->
  <text x="660" y="20" text-anchor="middle" class="title" style="font-size:13px">π0.5 模式</text>
  <rect x="460" y="35" width="180" height="45" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="550" y="55" text-anchor="middle" class="title">state (already in prompt)</text>
  <text x="550" y="70" text-anchor="middle" class="lbl">256-bin discrete tokens</text>

  <rect x="680" y="35" width="180" height="45" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="770" y="55" text-anchor="middle" class="title">noisy x_t [B, 50, 32]</text>
  <text x="770" y="70" text-anchor="middle" class="lbl">+ time t [B]</text>

  <line x1="770" y1="80" x2="770" y2="105" stroke="#64748b" marker-end="url(#arr3)"/>

  <rect x="680" y="105" width="180" height="40" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="770" y="123" text-anchor="middle" class="title">action_in_proj: Linear(32→1024)</text>
  <text x="770" y="138" text-anchor="middle" class="dim">[B, 50, 1024]</text>

  <rect x="460" y="105" width="180" height="40" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="550" y="123" text-anchor="middle" class="title">sincos(t, 1024)</text>
  <text x="550" y="138" text-anchor="middle" class="dim">[B, 1024]</text>

  <line x1="550" y1="145" x2="550" y2="170" stroke="#64748b" marker-end="url(#arr3)"/>
  <rect x="460" y="170" width="180" height="55" rx="6" fill="#4c1d95" stroke="#a855f7"/>
  <text x="550" y="188" text-anchor="middle" class="title">time_mlp_in/out</text>
  <text x="550" y="203" text-anchor="middle" class="lbl">Linear(1024→1024)+swish ×2</text>
  <text x="550" y="218" text-anchor="middle" class="dim">[B, 1024] adarms_cond</text>

  <line x1="550" y1="225" x2="550" y2="245" stroke="#a855f7" stroke-width="2" stroke-dasharray="4,3"/>
  <text x="550" y="263" text-anchor="middle" class="lbl" fill="#a855f7">↓ inject as adaRMS to every layer</text>

  <line x1="770" y1="145" x2="770" y2="320" stroke="#64748b" marker-end="url(#arr3)"/>

  <rect x="460" y="320" width="400" height="40" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="660" y="338" text-anchor="middle" class="title">Suffix tokens for π0.5</text>
  <text x="660" y="354" text-anchor="middle" class="dim">[action×50] = 50 tokens, dim=1024, ar_mask=[T, F×49]</text>

  <defs>
    <marker id="arr3" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#64748b"/>
    </marker>
  </defs>
</svg>
</div>

<h2 id="s7">7. 完整 Token 序列</h2>

<div class="pi0-seq">
  <div class="pi0-tok pi0-tok-img" title="base_0_rgb image tokens">img×256</div>
  <div class="pi0-tok pi0-tok-img">img×256</div>
  <div class="pi0-tok pi0-tok-img">img×256</div>
  <div class="pi0-tok pi0-tok-lang">lang×48</div>
  <div class="pi0-tok pi0-tok-state">state×1</div>
  <div class="pi0-tok pi0-tok-act">action×50</div>
</div>

<table class="pi0-table">
  <tr><th>段</th><th>长度</th><th>Expert</th><th>dim</th><th>ar_mask</th></tr>
  <tr><td>image (3×256)</td><td>768</td><td>PaliGemma</td><td>2048</td><td>全 False</td></tr>
  <tr><td>language</td><td>48</td><td>PaliGemma</td><td>2048</td><td>全 False</td></tr>
  <tr><td>state (π0 only)</td><td>1</td><td>Action Expert</td><td>1024</td><td>True</td></tr>
  <tr><td>action</td><td>50</td><td>Action Expert</td><td>1024</td><td>[True, False×49]</td></tr>
  <tr><td colspan="5"><strong>π0 总长: 816 + 1 + 50 = 867 tokens</strong></td></tr>
</table>

<p>训练时，<strong>一次 forward pass</strong> 处理整条序列：</p>

<pre><code>prefix_tokens, prefix_mask, prefix_ar_mask = self.embed_prefix(observation)
suffix_tokens, suffix_mask, suffix_ar_mask, adarms_cond = self.embed_suffix(observation, x_t, time)
input_mask = concat([prefix_mask, suffix_mask], axis=<span class="tok-num">1</span>)
ar_mask    = concat([prefix_ar_mask, suffix_ar_mask], axis=<span class="tok-num">0</span>)
attn_mask  = make_attn_mask(input_mask, ar_mask)
positions  = cumsum(input_mask, axis=<span class="tok-num">1</span>) - <span class="tok-num">1</span>

(prefix_out, suffix_out), _ = self.PaliGemma.llm(
    [prefix_tokens, suffix_tokens],              <span class="tok-cm"># 两个专家各收一份 token</span>
    mask=attn_mask,
    positions=positions,
    adarms_cond=[<span class="tok-kw">None</span>, adarms_cond],          <span class="tok-cm"># 只有 action expert 用 adaRMS（π0.5）</span>
)
v_t = self.action_out_proj(suffix_out[:, -<span class="tok-num">50</span>:])   <span class="tok-cm"># Linear(1024 → 32) → (B, 50, 32)</span>
</code></pre>

<h3>原理：双专家共享 attention 的实现细节（MQA + RoPE + RMSNorm）</h3>

<div class="pi0-callout">
<strong>MQA (Multi-Query Attention)</strong>：<code class="pi0-code-inline">num_heads=8, num_kv_heads=1</code>，意味着 8 个 query head 共享 1 套 K/V。普通 MHA 是 8×8，MQA 是 8×1，KV cache 内存占用降为 1/8。代码里 <code class="pi0-code-inline">G = num_heads / num_kv_heads = 8</code>，einsum 写作 <code class="pi0-code-inline">"BTKGH,BSKH->BKGTS"</code>（K 和 G 轴显式分开），8 个 group 共用同一个 K, V。
<br/>对 VLA 推理尤其重要：50Hz 的实时控制要求每步推理 &lt; 20ms，KV cache 越小，显存压力越小，可以塞下更长的 context。
<br/><br/>
<strong>RoPE (Rotary Positional Embedding)</strong>：把 Q, K 在每个 head 的维度上分成两半 (x1, x2)，用 sin/cos 旋转：<code class="pi0-code-inline">[x1·cos − x2·sin, x2·cos + x1·sin]</code>。结果是 <code class="pi0-code-inline">⟨RoPE(q, m), RoPE(k, n)⟩</code> 只依赖相对位置 <code class="pi0-code-inline">m − n</code>，不依赖绝对位置。
<br/>π0 用 RoPE 而非 learned positional embedding 的好处：长度可变（不同任务 token 数不同）时不用插值，外推性好。代码在 <code class="pi0-code-inline">_apply_rope</code>，max_wavelength=10000（同 LLaMA）。
<br/><br/>
<strong>RMSNorm vs LayerNorm</strong>：<code class="pi0-code-inline">RMSNorm(x) = x / sqrt(mean(x²)) · scale</code>，比 LayerNorm 少一个减均值步骤，参数少 50%，速度快 7–10%。Gemma 全程用 RMSNorm，在大模型上几乎没有效果损失。π0.5 在此基础上扩展为 adaRMSNorm（多了 scale + shift + gate）。
<br/><br/>
<strong>共享 attention 的细节</strong>：在每一层里，
<br/>&nbsp;&nbsp;1.&nbsp;PaliGemma expert 的 Q/K/V 投影矩阵形状 (2048, 8·256)，Action expert 是 (1024, 8·256)，head_dim=256 必须一致；
<br/>&nbsp;&nbsp;2.&nbsp;两组 Q, K, V 沿序列维 concat 成一条，attention 一次算完；
<br/>&nbsp;&nbsp;3.&nbsp;输出按原长度切回两段，分别走自己的 attn_out 投影、自己的 RMSNorm、自己的 FFN。
<br/>这样 PaliGemma 的"感知-语言"特征和 Action 的"动作"特征通过 attention 自由交换信息，但各自的 width 和 FFN 容量是独立的。
</div>

<h2 id="s8">8. Attention Mask 设计</h2>

<p>核心函数 <code class="pi0-code-inline">make_attn_mask</code>：</p>

<pre><code><span class="tok-kw">def</span> <span class="tok-fn">make_attn_mask</span>(input_mask, mask_ar):
    mask_ar   = broadcast_to(mask_ar, input_mask.shape)
    cumsum    = jnp.cumsum(mask_ar, axis=<span class="tok-num">1</span>)
    attn_mask = cumsum[:, <span class="tok-kw">None</span>, :] <= cumsum[:, :, <span class="tok-kw">None</span>]    <span class="tok-cm"># token i → j 当且仅当 cumsum[j] ≤ cumsum[i]</span>
    valid     = input_mask[:, <span class="tok-kw">None</span>, :] * input_mask[:, :, <span class="tok-kw">None</span>]
    <span class="tok-kw">return</span> attn_mask & valid
</code></pre>

<p><strong>直觉</strong>：<code class="pi0-code-inline">mask_ar</code> 把序列切成若干个 "block"。<code class="pi0-code-inline">ar_mask=True</code> 的 token 开启一个新 block，该 block 内部的 token 可以互相看见，但之前 block 的 token 看不见它。</p>

<div class="pi0-flow">position:   0    1    2    3    4    5    6    7
ar_mask:    F    F    F    F    T    F    F    F
cumsum:     0    0    0    0    1    1    1    1
            └── block 0 ──┘   └── block 1 ──┘

block 0 内部：双向
block 1 → block 0：可看
block 0 → block 1：不可看 (causal boundary)</div>

<p><strong>π0 的实际效果</strong>：</p>

<ol>
  <li><strong>image + language（816 tokens, ar_mask=False）</strong>：cumsum 都是 0 → 互相全可见，prefix-LM style</li>
  <li><strong>state token（ar_mask=True）</strong>：cumsum 变 1 → 看得见 image/language，但 image/language 看不见 state</li>
  <li><strong>action 第 1 个（ar_mask=True）</strong>：cumsum 变 2 → 看得见 prefix + state</li>
  <li><strong>action 第 2~50 个（ar_mask=False）</strong>：cumsum 仍然是 2 → action 内部双向，都看得见 prefix + state</li>
</ol>

<div class="pi0-callout pi0-callout-tip">
<strong>为什么 action 内部双向？</strong><br/>
因为动作是 chunk 预测（50 步一起出），50 个 action tokens 本质上是 50 个并行的"待去噪变量"，没有因果顺序。双向 attention 允许它们在 denoising 过程中互相协调。
</div>

<p><strong>positions（RoPE 位置）</strong>：<code class="pi0-code-inline">positions = cumsum(input_mask, axis=1) - 1</code>，按序分配 0, 1, 2, ...，padding 位置不占位。</p>

<h3>原理：为什么用 cumsum 这个技巧？和标准 causal mask 什么区别？</h3>

<div class="pi0-callout">
<strong>标准 causal mask（GPT 式）</strong>：<code class="pi0-code-inline">mask[i, j] = (j <= i)</code>，每个 token 只能看自己和之前的 token。这在语言生成中很合理（"下一个词"），但在 VLA 中会损失信息 —— image patches 之间本应双向互看，language 和 image 也应双向（描述与画面互相关联）。
<br/><br/>
<strong>标准 full mask（BERT 式）</strong>：所有 token 互相可见。但 action tokens 不能让 prefix 看见 —— 训练时 action 是 noisy 的，如果 image/language 能 attend 到 noisy action，感知特征会被噪声污染。
<br/><br/>
<strong>cumsum 技巧</strong>：把两种模式合而为一。<code class="pi0-code-inline">ar_mask</code> 标记"block 边界"：<code class="pi0-code-inline">True</code> 表示"这个 token 及以后进入新 block"，<code class="pi0-code-inline">False</code> 表示"延续当前 block"。cumsum 自然分配 block 索引 0, 1, 2, ...。规则 <code class="pi0-code-inline">cumsum[j] ≤ cumsum[i]</code> 等价于：
<br/>&nbsp;&nbsp;•&nbsp;同一 block 内部：双向；
<br/>&nbsp;&nbsp;•&nbsp;跨 block：只能后面 block 看前面 block（causal between blocks）。
<br/>一行 cumsum 就实现了 prefix-LM + block-causal 的复合 mask，优雅且 GPU 友好。
<br/><br/>
<strong>与 Llama 等 LLM 的差异</strong>：LLM 都是纯 causal，每个 token 预测下一个；VLA 中 action 是 chunk 预测（50 步一起出），没有顺序意义，所以 action 内部双向。这是 VLA 和 LLM 架构分叉的关键点。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">π0 Attention Mask 可视化（行=query，列=key，绿=可见，灰=屏蔽）</div>
<svg class="pi0-arch-svg" viewBox="0 0 720 480" xmlns="http://www.w3.org/2000/svg">
  <!-- matrix -->
  <!-- segments: image(0-3, compressed as 4 blocks for illustration), lang(4-5), state(6), action(7-10, compressed) -->
  <!-- We'll draw a 11x11 cell grid -->
  <!-- Column headers -->
  <text x="85" y="30" text-anchor="middle" class="lbl">K: image</text>
  <text x="205" y="30" text-anchor="middle" class="lbl">K: lang</text>
  <text x="275" y="30" text-anchor="middle" class="lbl">K: state</text>
  <text x="385" y="30" text-anchor="middle" class="lbl">K: action</text>

  <!-- headers markers -->
  <line x1="35" y1="35" x2="475" y2="35" stroke="#475569"/>
  <line x1="145" y1="35" x2="145" y2="435" stroke="#475569" stroke-dasharray="2,2"/>
  <line x1="245" y1="35" x2="245" y2="435" stroke="#475569" stroke-dasharray="2,2"/>
  <line x1="305" y1="35" x2="305" y2="435" stroke="#475569" stroke-dasharray="2,2"/>

  <!-- Row headers -->
  <text x="20" y="80" text-anchor="end" class="lbl">Q: image</text>
  <text x="20" y="170" text-anchor="end" class="lbl">Q: lang</text>
  <text x="20" y="235" text-anchor="end" class="lbl">Q: state</text>
  <text x="20" y="335" text-anchor="end" class="lbl">Q: action</text>

  <!-- Row segment lines -->
  <line x1="35" y1="145" x2="475" y2="145" stroke="#475569" stroke-dasharray="2,2"/>
  <line x1="35" y1="215" x2="475" y2="215" stroke="#475569" stroke-dasharray="2,2"/>
  <line x1="35" y1="275" x2="475" y2="275" stroke="#475569" stroke-dasharray="2,2"/>

  <!-- Filled blocks -->
  <!-- image queries: can see image, lang. NOT state, NOT action -->
  <rect x="35" y="40" width="210" height="105" fill="#166534" opacity="0.75"/>
  <rect x="245" y="40" width="230" height="105" fill="#1e293b"/>

  <!-- lang queries: can see image, lang. NOT state, NOT action -->
  <rect x="35" y="145" width="210" height="70" fill="#166534" opacity="0.75"/>
  <rect x="245" y="145" width="230" height="70" fill="#1e293b"/>

  <!-- state query: can see image, lang, state. NOT action -->
  <rect x="35" y="215" width="270" height="60" fill="#166534" opacity="0.75"/>
  <rect x="305" y="215" width="170" height="60" fill="#1e293b"/>

  <!-- action queries: can see all (image, lang, state, action) -->
  <rect x="35" y="275" width="440" height="160" fill="#166534" opacity="0.75"/>

  <!-- Grid outline -->
  <rect x="35" y="40" width="440" height="395" fill="none" stroke="#475569" stroke-width="1"/>

  <!-- Annotations -->
  <text x="140" y="95" text-anchor="middle" class="dim">✓ bidirectional</text>
  <text x="140" y="180" text-anchor="middle" class="dim">✓ bidirectional</text>
  <text x="170" y="250" text-anchor="middle" class="dim">✓ state 可看 prefix</text>
  <text x="255" y="355" text-anchor="middle" class="dim">✓ action 可看所有 prefix</text>
  <text x="395" y="355" text-anchor="middle" class="dim">✓ action 内部双向</text>

  <text x="360" y="95" text-anchor="middle" class="lbl" fill="#ef4444">✗ image 看不见 state/action</text>
  <text x="360" y="180" text-anchor="middle" class="lbl" fill="#ef4444">✗ lang 看不见 state/action</text>
  <text x="390" y="250" text-anchor="middle" class="lbl" fill="#ef4444">✗ state 看不见 action</text>

  <!-- Legend & formula -->
  <rect x="520" y="40" width="180" height="20" fill="#166534" opacity="0.75"/>
  <text x="530" y="55" class="lbl" fill="#e2e8f0">✓ attend (True)</text>
  <rect x="520" y="70" width="180" height="20" fill="#1e293b"/>
  <text x="530" y="85" class="lbl" fill="#e2e8f0">✗ masked (False)</text>

  <text x="520" y="120" class="lbl">ar_mask 序列:</text>
  <text x="520" y="140" class="dim">img:    F...F</text>
  <text x="520" y="158" class="dim">lang:   F...F</text>
  <text x="520" y="176" class="dim">state:  T</text>
  <text x="520" y="194" class="dim">act[0]: T</text>
  <text x="520" y="212" class="dim">act[1:]:F...F</text>

  <text x="520" y="250" class="lbl">cumsum:</text>
  <text x="520" y="270" class="dim">img:    0...0</text>
  <text x="520" y="288" class="dim">lang:   0...0</text>
  <text x="520" y="306" class="dim">state:  1</text>
  <text x="520" y="324" class="dim">act[0]: 2</text>
  <text x="520" y="342" class="dim">act[1:]:2...2</text>

  <text x="520" y="380" class="lbl">Rule: Q_i attends K_j iff</text>
  <text x="520" y="400" class="dim" fill="#fbbf24">cumsum[j] ≤ cumsum[i]</text>
</svg>
</div>

<div class="pi0-callout pi0-callout-tip">
<strong>读图</strong>：绿色 = 该 (query, key) 对可以 attend，灰色 = 被 mask 屏蔽。
<br/>image 和 language 的 cumsum 都是 0，互相都 ≤ 0，所以左上 4 个块全绿（双向）。
<br/>state 的 cumsum=1，state 作为 query 可以 attend 到所有 cumsum ≤ 1 的 key（image+lang+state），但作为 key 时只有 cumsum ≥ 1 的 query 能看见它。
<br/>action 的 cumsum=2，可以 attend 到全体；但 action 作为 key 时只有 action 自己能看见它 —— 这就是为什么 image/lang/state 对 action 列全是灰色。
</div>

<h2 id="s9">9. Flow Matching 训练目标</h2>

<p>π0 不是 diffusion，是 <strong>flow matching</strong>（条件 rectified flow）。数据分布是真实动作 <code class="pi0-code-inline">a</code>，噪声分布是 <code class="pi0-code-inline">N(0, I)</code>，两者之间学一条直线。</p>

<pre><code>preprocess_rng, noise_rng, time_rng = jax.random.split(rng, <span class="tok-num">3</span>)

<span class="tok-cm"># 1. 采样 time，偏向 t 较大的区域（噪声多的地方学得更仔细）</span>
noise = jax.random.normal(noise_rng, actions.shape)                      <span class="tok-cm"># (B, 50, 32)</span>
time  = jax.random.beta(time_rng, <span class="tok-num">1.5</span>, <span class="tok-num">1</span>, batch_shape) * <span class="tok-num">0.999</span> + <span class="tok-num">0.001</span>   <span class="tok-cm"># ∈ (0.001, 1.0)</span>
t = time[..., <span class="tok-kw">None</span>, <span class="tok-kw">None</span>]                                                     <span class="tok-cm"># (B, 1, 1)</span>

<span class="tok-cm"># 2. 线性插值：t=1 → 纯噪声，t=0 → 真实动作</span>
x_t = t * noise + (<span class="tok-num">1</span> - t) * actions
u_t = noise - actions                                                    <span class="tok-cm"># 目标速度场</span>

<span class="tok-cm"># 3. 模型预测 velocity</span>
v_t = model(observation, x_t, time)    <span class="tok-cm"># 通过上述 prefix+suffix 前向，输出 (B, 50, 32)</span>

<span class="tok-cm"># 4. 损失</span>
loss = jnp.mean((v_t - u_t)**<span class="tok-num">2</span>, axis=-<span class="tok-num">1</span>)     <span class="tok-cm"># (B, 50)</span>
</code></pre>

<div class="pi0-callout pi0-callout-warn">
<strong>Beta(1.5, 1) 采样</strong>的 PDF 在 t≈1 处更密集。这意味着训练时会更多地看到"噪声多"的样本 —— 因为去噪初期的速度场预测对整条轨迹的误差影响最大。
</div>

<h3>原理：flow matching 凭什么替代 diffusion？为什么目标是 velocity？</h3>

<div class="pi0-callout">
<strong>Diffusion 做的事</strong>：定义一个从数据到噪声的 SDE（Stochastic Differential Equation），前向逐步加噪，反向训练一个网络预测噪声 <code class="pi0-code-inline">ε</code>（或 score ∇log p）。采样时用 SDE solver 积分，通常需要 50–100 步。
<br/><br/>
<strong>Flow matching 做的事</strong>：直接在数据 <code class="pi0-code-inline">a</code> 和噪声 <code class="pi0-code-inline">ε</code> 之间画一条路径（rectified flow 用直线 <code class="pi0-code-inline">x_t = t·ε + (1-t)·a</code>），训练网络预测路径上任意时刻 t 的速度场 <code class="pi0-code-inline">u_t = dx_t/dt = ε - a</code>。采样时用 ODE solver（Euler）积分 <code class="pi0-code-inline">dx/dt = v_θ(x, t)</code>，只需 5–10 步。
<br/><br/>
<strong>为什么 velocity 比 noise prediction 更好？</strong>
<br/>&nbsp;&nbsp;•&nbsp;路径是直线，v_t 在 t 上变化平缓 → 模型更容易学习，少量 Euler 步就够；
<br/>&nbsp;&nbsp;•&nbsp;确定性 ODE 采样没有额外噪声，复现性强 —— 机器人控制最需要确定性；
<br/>&nbsp;&nbsp;•&nbsp;50Hz 的控制频率要求低延迟，10 步 ODE 远快于 50 步 diffusion。
<br/><br/>
<strong>Beta(1.5, 1) 采样的动机</strong>：其 PDF 在 t=1 附近（纯噪声端）密度更高。去噪初期误差会被后续 step 放大，所以那里更值得训练；t 接近 0 时速度场几乎是常数（直线），模型不用学太多。这种非均匀采样相当于"课程学习"，把训练预算分配给关键时段。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">Flow Matching 训练流程（单个 step）</div>
<svg class="pi0-arch-svg" viewBox="0 0 880 340" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="20" width="160" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="100" y="42" text-anchor="middle" class="title">actions a</text>
  <text x="100" y="58" text-anchor="middle" class="dim">[B, 50, 32]</text>

  <rect x="200" y="20" width="160" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="280" y="42" text-anchor="middle" class="title">noise ε ~ N(0, I)</text>
  <text x="280" y="58" text-anchor="middle" class="dim">[B, 50, 32]</text>

  <rect x="380" y="20" width="160" height="50" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="460" y="42" text-anchor="middle" class="title">t ~ Beta(1.5, 1)</text>
  <text x="460" y="58" text-anchor="middle" class="dim">[B], ∈ (0.001, 1.0)</text>

  <rect x="560" y="20" width="160" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="640" y="42" text-anchor="middle" class="title">observation o</text>
  <text x="640" y="58" text-anchor="middle" class="dim">images, state, prompt</text>

  <line x1="100" y1="70" x2="100" y2="105" stroke="#64748b" marker-end="url(#arr4)"/>
  <line x1="280" y1="70" x2="280" y2="105" stroke="#64748b" marker-end="url(#arr4)"/>
  <line x1="460" y1="70" x2="460" y2="105" stroke="#64748b" marker-end="url(#arr4)"/>

  <rect x="20" y="105" width="520" height="70" rx="8" fill="#78350f" stroke="#f59e0b" stroke-width="1.5"/>
  <text x="280" y="128" text-anchor="middle" class="title">Linear Interpolation (Rectified Flow Path)</text>
  <text x="280" y="148" text-anchor="middle" class="lbl">x_t = t · ε + (1 − t) · a</text>
  <text x="280" y="165" text-anchor="middle" class="lbl">u_t = ε − a   (ground-truth velocity, constant along the path)</text>

  <line x1="280" y1="175" x2="280" y2="210" stroke="#64748b" marker-end="url(#arr4)"/>
  <line x1="640" y1="70" x2="640" y2="210" stroke="#64748b" marker-end="url(#arr4)"/>
  <line x1="460" y1="175" x2="460" y2="210" stroke="#64748b" stroke-dasharray="3,2" marker-end="url(#arr4)"/>

  <rect x="200" y="210" width="480" height="60" rx="8" fill="#4c1d95" stroke="#a855f7" stroke-width="1.5"/>
  <text x="440" y="232" text-anchor="middle" class="title">π0 Double-Expert Transformer</text>
  <text x="440" y="249" text-anchor="middle" class="lbl">embed_prefix(o) + embed_suffix(o, x_t, t) → one forward pass</text>
  <text x="440" y="264" text-anchor="middle" class="dim">suffix_out[:, −50:] → action_out_proj → v_θ(x_t, o, t)</text>

  <line x1="440" y1="270" x2="440" y2="295" stroke="#64748b" marker-end="url(#arr4)"/>
  <rect x="280" y="295" width="320" height="40" rx="6" fill="#7f1d1d" stroke="#ef4444"/>
  <text x="440" y="318" text-anchor="middle" class="title">Loss = E [ ‖ v_θ(x_t, o, t) − (ε − a) ‖² ]</text>

  <defs>
    <marker id="arr4" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#64748b"/>
    </marker>
  </defs>
</svg>
</div>

<h2 id="s10">10. 推理：Euler ODE Solver</h2>

<pre><code><span class="tok-kw">def</span> <span class="tok-fn">sample_actions</span>(self, rng, observation, *, num_steps=<span class="tok-num">10</span>):
    dt = -<span class="tok-num">1.0</span> / num_steps                                     <span class="tok-cm"># -0.1</span>
    noise = jax.random.normal(rng, (B, <span class="tok-num">50</span>, <span class="tok-num">32</span>))                  <span class="tok-cm"># x_1 ~ N(0, I)</span>

    <span class="tok-cm"># 1. 预填 KV cache：prefix 只算一次</span>
    prefix_tokens, prefix_mask, prefix_ar_mask = self.embed_prefix(observation)
    prefix_attn_mask = make_attn_mask(prefix_mask, prefix_ar_mask)
    positions = cumsum(prefix_mask, axis=<span class="tok-num">1</span>) - <span class="tok-num">1</span>
    _, kv_cache = self.PaliGemma.llm([prefix_tokens, <span class="tok-kw">None</span>], mask=prefix_attn_mask, positions=positions)

    <span class="tok-cm"># 2. 循环 num_steps 次 Euler 积分（t: 1.0 → 0.0）</span>
    <span class="tok-kw">def</span> <span class="tok-fn">step</span>(carry):
        x_t, time = carry
        suffix_tokens, suffix_mask, suffix_ar_mask, adarms_cond = self.embed_suffix(obs, x_t, broadcast(time, B))
        full_attn_mask = concat([
            repeat(prefix_mask, <span class="tok-str">"b p -> b s p"</span>, s=suffix_len),
            make_attn_mask(suffix_mask, suffix_ar_mask),
        ], axis=-<span class="tok-num">1</span>)
        positions = sum(prefix_mask, -<span class="tok-num">1</span>)[:, <span class="tok-kw">None</span>] + cumsum(suffix_mask, -<span class="tok-num">1</span>) - <span class="tok-num">1</span>

        (_, suffix_out), _ = self.PaliGemma.llm(
            [<span class="tok-kw">None</span>, suffix_tokens],         <span class="tok-cm"># 只跑 suffix，prefix 复用 KV cache</span>
            mask=full_attn_mask,
            positions=positions,
            kv_cache=kv_cache,
            adarms_cond=[<span class="tok-kw">None</span>, adarms_cond],
        )
        v_t = self.action_out_proj(suffix_out[:, -<span class="tok-num">50</span>:])     <span class="tok-cm"># (B, 50, 32)</span>
        <span class="tok-kw">return</span> x_t + dt * v_t, time + dt

    x_0, _ = jax.lax.while_loop(cond, step, (noise, <span class="tok-num">1.0</span>))
    <span class="tok-kw">return</span> x_0    <span class="tok-cm"># 预测的 action chunk (B, 50, 32)</span>
</code></pre>

<div class="pi0-callout pi0-callout-tip">
<strong>效率关键</strong>：prefix（视觉 + 语言）在整个去噪过程中不变，所以只需要算一次 forward 并缓存所有层的 K, V。之后 10 次 denoising step 只对 51 个 suffix tokens 做 attention，每个 query attend to [prefix 816 + suffix 51] 的 K, V。这样推理成本约等于 "1 次 full pass + 10 次 suffix-only pass"，远小于"10 次 full pass"。
</div>

<h3>原理：KV cache 为什么这么重要？10 步 Euler 够用的数学依据</h3>

<div class="pi0-callout">
<strong>KV cache 的复用条件</strong>：对任意一层 attention，K、V 是由输入 token 的 embedding 通过固定线性变换得到的。只要 prefix 的 token embedding 不变，K、V 就不变。Denoising 循环中只有 suffix（action tokens）随 t 变化，prefix 完全不动 → K、V 可以缓存。
<br/><br/>
<strong>计算量对比</strong>（假设 B=1，prefix=816，suffix=51）：
<br/>&nbsp;&nbsp;•&nbsp;full pass（训练）：Q, K, V 形状 (867, d)，attention 复杂度 O(867²·d)；
<br/>&nbsp;&nbsp;•&nbsp;cached pass（推理 step）：Q 只算 suffix (51, d)，K/V 直接从缓存取 (867, d)，attention 只算 51 行 → O(51·867·d)，约 5.9% 的开销。
<br/><br/>
<strong>10 步 Euler 的理论支撑</strong>：rectified flow 的路径是直线 <code class="pi0-code-inline">x_t = t·ε + (1-t)·a</code>，其"真实"速度场 <code class="pi0-code-inline">u_t = ε - a</code> 沿 t 是常数。如果模型完美学到了 u_t，1 步积分就够了（直接从 t=1 跳到 t=0）。实际模型对 u_t 的拟合有误差，误差随 t 变化，所以需要多步。经验上：
<br/>&nbsp;&nbsp;•&nbsp;1 步：直接预测动作，相当于 behavior cloning，表达力弱；
<br/>&nbsp;&nbsp;•&nbsp;4–5 步：大部分任务已经收敛；
<br/>&nbsp;&nbsp;•&nbsp;10 步：性能平台期，再多步边际收益极小。
<br/>相比 diffusion 的 50–100 步，这是数量级的效率提升，使 50Hz 实时控制成为可能。
<br/><br/>
<strong>dt 的符号</strong>：代码中 <code class="pi0-code-inline">dt = -1/num_steps</code>，是因为训练时约定 t=1 是噪声、t=0 是动作（与原 π0 论文相反）。所以采样从 t=1 反向走到 t=0，dt 为负。
</div>

<div class="pi0-diagram">
<div class="pi0-diagram-title">推理流程：Prefix KV-Cache + 10 步 Euler Denoising</div>
<svg class="pi0-arch-svg" viewBox="0 0 880 420" xmlns="http://www.w3.org/2000/svg">
  <!-- Phase 1: prefix forward -->
  <text x="20" y="25" class="title" fill="#60a5fa">Phase 1 — 预填 KV Cache（只执行 1 次）</text>

  <rect x="20" y="40" width="160" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="100" y="60" text-anchor="middle" class="title">images + prompt</text>
  <text x="100" y="76" text-anchor="middle" class="lbl">(observation)</text>

  <line x1="180" y1="65" x2="215" y2="65" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="215" y="40" width="180" height="50" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="305" y="60" text-anchor="middle" class="title">embed_prefix</text>
  <text x="305" y="76" text-anchor="middle" class="dim">→ (B, 816, 2048)</text>

  <line x1="395" y1="65" x2="430" y2="65" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="430" y="40" width="200" height="50" rx="6" fill="#4c1d95" stroke="#a855f7"/>
  <text x="530" y="60" text-anchor="middle" class="title">PaliGemma LLM (18 layers)</text>
  <text x="530" y="76" text-anchor="middle" class="lbl">suffix input = None</text>

  <line x1="630" y1="65" x2="665" y2="65" stroke="#f59e0b" marker-end="url(#arr5-amber)"/>

  <rect x="665" y="40" width="200" height="50" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="765" y="60" text-anchor="middle" class="title">KV Cache</text>
  <text x="765" y="76" text-anchor="middle" class="dim">18 × (K, V) stored</text>

  <!-- Separator -->
  <line x1="20" y1="110" x2="860" y2="110" stroke="#475569" stroke-dasharray="5,3"/>

  <!-- Phase 2: denoising loop -->
  <text x="20" y="135" class="title" fill="#60a5fa">Phase 2 — Euler 循环（重复 num_steps = 10 次）</text>

  <rect x="20" y="150" width="120" height="50" rx="6" fill="#0c4a6e" stroke="#0ea5e9"/>
  <text x="80" y="170" text-anchor="middle" class="title">x_1 ~ N(0, I)</text>
  <text x="80" y="186" text-anchor="middle" class="dim">[B, 50, 32]</text>

  <line x1="140" y1="175" x2="175" y2="175" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="175" y="150" width="130" height="50" rx="6" fill="#1e3a8a" stroke="#3b82f6"/>
  <text x="240" y="170" text-anchor="middle" class="title">embed_suffix</text>
  <text x="240" y="186" text-anchor="middle" class="dim">x_t, t → suffix</text>

  <line x1="305" y1="175" x2="340" y2="175" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="340" y="150" width="230" height="50" rx="6" fill="#4c1d95" stroke="#a855f7"/>
  <text x="455" y="170" text-anchor="middle" class="title">PaliGemma LLM</text>
  <text x="455" y="186" text-anchor="middle" class="lbl">suffix only, attend to cached K,V</text>

  <line x1="440" y1="150" x2="440" y2="125" stroke="#f59e0b" stroke-width="1.5"/>
  <line x1="440" y1="125" x2="765" y2="125" stroke="#f59e0b" stroke-width="1.5"/>
  <line x1="765" y1="125" x2="765" y2="90" stroke="#f59e0b" stroke-width="1.5"/>
  <text x="600" y="120" text-anchor="middle" class="lbl" fill="#f59e0b">read cache</text>

  <line x1="570" y1="175" x2="605" y2="175" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="605" y="150" width="160" height="50" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="685" y="170" text-anchor="middle" class="title">action_out_proj</text>
  <text x="685" y="186" text-anchor="middle" class="dim">v_t: [B, 50, 32]</text>

  <line x1="765" y1="175" x2="800" y2="175" stroke="#64748b" marker-end="url(#arr5)"/>

  <rect x="760" y="210" width="100" height="40" rx="6" fill="#78350f" stroke="#f59e0b"/>
  <text x="810" y="228" text-anchor="middle" class="title">Euler step</text>
  <text x="810" y="244" text-anchor="middle" class="lbl">x_t += dt · v_t</text>

  <line x1="810" y1="200" x2="810" y2="210" stroke="#64748b" marker-end="url(#arr5)"/>

  <!-- Loop-back arrow -->
  <path d="M 760 230 L 20 230 L 20 200 L 30 200" fill="none" stroke="#a855f7" stroke-width="1.5" stroke-dasharray="4,3" marker-end="url(#arr5-purple)"/>
  <text x="400" y="225" text-anchor="middle" class="lbl" fill="#a855f7">loop while t ≥ −dt/2  (10 iterations)</text>

  <!-- Final output -->
  <line x1="80" y1="260" x2="80" y2="290" stroke="#64748b" marker-end="url(#arr5)"/>
  <rect x="20" y="290" width="250" height="50" rx="6" fill="#064e3b" stroke="#10b981"/>
  <text x="145" y="310" text-anchor="middle" class="title">x_0: predicted action chunk</text>
  <text x="145" y="326" text-anchor="middle" class="dim">[B, 50, 32] → unnormalize → robot</text>

  <!-- Cost annotation -->
  <rect x="320" y="290" width="540" height="75" rx="6" fill="#1e293b" stroke="#475569"/>
  <text x="330" y="310" class="lbl" fill="#60a5fa">推理成本分解（B=1）</text>
  <text x="330" y="328" class="lbl">• Phase 1 (1×): QKV over 816 tokens → Q,K,V + full attention</text>
  <text x="330" y="346" class="lbl">• Phase 2 (10×): Q only over 51 suffix tokens, K/V from cache</text>
  <text x="330" y="364" class="lbl" fill="#fbbf24">总开销 ≈ 1.6 × full-pass cost（相比 10 × full pass 减少约 85%）</text>

  <defs>
    <marker id="arr5" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#64748b"/>
    </marker>
    <marker id="arr5-amber" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#f59e0b"/>
    </marker>
    <marker id="arr5-purple" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <polygon points="0 0, 8 4, 0 8" fill="#a855f7"/>
    </marker>
  </defs>
</svg>
</div>

<h2 id="s11">11. 数据 Pipeline</h2>

<p>从 LeRobot dataset 到模型输入的完整流水线（<code class="pi0-code-inline">training/data_loader.py</code> + <code class="pi0-code-inline">transforms.py</code>）：</p>

<div class="pi0-flow">LeRobotDataset  (hugging face)
    │
    ▼  delta_timestamps: 采 50 步 action chunk
    │
RepackTransform   (字段重命名, e.g. observation.images.top → image.base_0_rgb)
    │
    ▼
DataTransforms.inputs   (dataset-specific, e.g. Aloha/Droid 的维度映射)
    │
    ▼
DeltaActions(mask)      (绝对动作 → 相对动作，给指定维度 actions -= state)
    │
    ▼
Normalize(norm_stats)   (z-score 或 quantile norm，使用 compute_norm_stats.py 预计算)
    │
    ▼
ModelTransforms.inputs:
    ├─ InjectDefaultPrompt       (补一个默认 prompt, 如果 episode 没有)
    ├─ ResizeImages(224, 224)    (pad-resize 到模型输入尺寸)
    ├─ PadStatesAndActions(32)   (zero-pad 到 action_dim=32)
    └─ TokenizePrompt            (SentencePiece + mask)
    │
    ▼
Observation.from_dict            (dict → 结构化对象；同时做 uint8→[-1,1])</div>

<h4>训练时图像增强</h4>

<p>非 wrist 相机（<code class="pi0-code-inline">base_0_rgb</code>）做几何增强，wrist 相机只做颜色增强：</p>

<pre><code><span class="tok-kw">if</span> <span class="tok-str">"wrist"</span> <span class="tok-kw">not in</span> key:
    transforms += [
        augmax.RandomCrop(<span class="tok-kw">int</span>(w * <span class="tok-num">0.95</span>), <span class="tok-kw">int</span>(h * <span class="tok-num">0.95</span>)),
        augmax.Resize(w, h),
        augmax.Rotate((-<span class="tok-num">5</span>, <span class="tok-num">5</span>)),
    ]
transforms += [
    augmax.ColorJitter(brightness=<span class="tok-num">0.3</span>, contrast=<span class="tok-num">0.4</span>, saturation=<span class="tok-num">0.5</span>),
]
</code></pre>

<div class="pi0-callout pi0-callout-warn">
<strong>为什么 wrist 相机不做几何增强？</strong><br/>
wrist 相机随 end-effector 运动，画面本身就高度关联动作空间。做 crop/rotate 会破坏"图像-动作"的几何对应关系。base 相机是固定视角，几何扰动模拟视角抖动反而有益。
</div>

<h2 id="s12">12. π0 vs π0.5 关键差异</h2>

<table class="pi0-table">
  <tr><th>维度</th><th>π0</th><th>π0.5</th></tr>
  <tr><td>state 输入方式</td><td>连续向量，suffix 中一个独立 token</td><td>离散化为 256 bins，编码进 language tokens</td></tr>
  <tr><td>max_token_len</td><td>48</td><td>200</td></tr>
  <tr><td>timestep 注入</td><td>concat 到 action tokens，MLP 混合</td><td>adaRMSNorm 条件调制（每层）</td></tr>
  <tr><td>action expert conditioning</td><td>无</td><td>adaRMS: scale + shift + gate</td></tr>
  <tr><td>suffix 中是否有 state token</td><td>有（1 个）</td><td>无</td></tr>
  <tr><td>action 模块 Linear</td><td><code class="pi0-code-inline">state_proj, action_in_proj, action_time_mlp_{in,out}, action_out_proj</code></td><td><code class="pi0-code-inline">action_in_proj, time_mlp_{in,out}, action_out_proj</code></td></tr>
  <tr><td>prompt 模板</td><td><code class="pi0-code-inline">"{prompt}\n"</code></td><td><code class="pi0-code-inline">"Task: {prompt}, State: {state};\nAction: "</code></td></tr>
</table>

<div class="pi0-callout pi0-callout-tip">
<strong>π0.5 的设计哲学</strong>：把 state 从连续向量改成离散 token，统一了"感知-状态-语言"的表示 —— 全都是 PaliGemma 词表里的 token。这让 state 能直接参与 prefix 的双向 attention，语义上比单独开一个 state token 更 naturally fused。adaRMS 比 concat+MLP 的 conditioning 更强，因为它在每一层都能调制。
</div>

<h2>小结</h2>

<p>π0 的架构精髓：</p>

<ul>
  <li><strong>两个专家共享 attention</strong>：VLM 段(2048 dim) 和 Action 段(1024 dim) 通过同一套 QKV 计算互相交换信息，但各自有独立的 FFN / LayerNorm 参数 —— VLM 处理感知-语言，Action 专家处理去噪。</li>
  <li><strong>Prefix-LM + causal boundary</strong>：image/language 双向，action chunk 也双向，之间用 <code class="pi0-code-inline">ar_mask</code> 切开，用 cumsum 巧妙地同时支持两种 attention 模式。</li>
  <li><strong>Flow matching over diffusion</strong>：直线路径 <code class="pi0-code-inline">x_t = t·noise + (1-t)·action</code>，学速度场而非噪声，ODE 积分 10 步即可出动作。</li>
  <li><strong>KV cache 推理优化</strong>：prefix 算一次，suffix 循环 10 次，这是能做到实时控制（50Hz action chunk）的关键。</li>
</ul>

</div>
{{< /rawhtml >}}
