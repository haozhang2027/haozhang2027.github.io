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

<p>Prefix 总长：<span class="pi0-badge pi0-dim">768 + 48 = 816</span> tokens，每个 dim=<span class="pi0-badge pi0-dim">2048</span>。所有 <code class="pi0-code-inline">ar_mask=False</code>，意味着这一段内部是完全双向的 prefix-LM attention。</p>

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
