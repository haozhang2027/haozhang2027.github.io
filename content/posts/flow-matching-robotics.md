---
title: "Flow Matching 在具身机器人中的作用"
date: 2026-05-12
draft: false
math: false
tags:
  - robotics
  - flow-matching
  - pi0
  - deep-learning
categories:
  - robotics
---

{{< rawhtml >}}
<style>
  .fm-wrap * { box-sizing: border-box; }
  .fm-wrap { font-family: 'Segoe UI', system-ui, sans-serif; color: #e2e8f0; line-height: 1.7; }

  .fm-hero { background: linear-gradient(135deg, #1a1f35 0%, #0f1117 100%); padding: 48px 32px; text-align: center; border-radius: 12px; margin-bottom: 40px; border: 1px solid #2d3748; }
  .fm-hero h1 { font-size: 2rem; font-weight: 700; background: linear-gradient(90deg, #60a5fa, #a78bfa, #34d399); -webkit-background-clip: text; -webkit-text-fill-color: transparent; margin-bottom: 10px; }
  .fm-hero p { color: #94a3b8; font-size: 1rem; max-width: 560px; margin: 0 auto; }

  .fm-section { margin-bottom: 48px; }
  .fm-section h2 { font-size: 1.3rem; font-weight: 600; color: #60a5fa; margin-bottom: 16px; padding-bottom: 8px; border-bottom: 1px solid #1e293b; }
  .fm-section h3 { font-size: 1rem; font-weight: 600; color: #a78bfa; margin: 20px 0 8px; }
  .fm-section p { color: #cbd5e1; margin-bottom: 12px; }

  .fm-card { background: #1e293b; border: 1px solid #2d3748; border-radius: 12px; padding: 20px; margin-bottom: 14px; }
  .fm-card-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
  @media (max-width: 600px) { .fm-card-grid { grid-template-columns: 1fr; } }

  .fm-highlight { background: #0f2744; border-left: 3px solid #60a5fa; padding: 14px 18px; border-radius: 0 8px 8px 0; margin: 14px 0; color: #cbd5e1; }

  .fm-demo-box { background: #1e293b; border: 1px solid #2d3748; border-radius: 12px; padding: 20px; }
  .fm-demo-box canvas { display: block; margin: 0 auto; border-radius: 8px; background: #0f1117; max-width: 100%; }
  .fm-controls { display: flex; gap: 10px; justify-content: center; margin-top: 14px; flex-wrap: wrap; }
  .fm-btn { background: #3b82f6; color: white; border: none; padding: 7px 18px; border-radius: 8px; cursor: pointer; font-size: 0.88rem; transition: background 0.2s; }
  .fm-btn:hover { background: #2563eb; }
  .fm-btn.sec { background: #374151; }
  .fm-btn.sec:hover { background: #4b5563; }
  .fm-slider-row { display: flex; align-items: center; gap: 10px; justify-content: center; margin-top: 10px; color: #94a3b8; font-size: 0.88rem; flex-wrap: wrap; }
  .fm-slider-row input[type=range] { accent-color: #60a5fa; }

  .fm-formula { background: #0d1b2a; border: 1px solid #1e3a5f; border-radius: 8px; padding: 16px; text-align: center; font-family: 'Courier New', monospace; color: #93c5fd; font-size: 0.95rem; margin: 14px 0; }
  .fm-formula .fm-cmt { color: #64748b; font-size: 0.82rem; display: block; margin-top: 5px; font-family: system-ui; }

  .fm-step { display: flex; gap: 14px; margin-bottom: 18px; }
  .fm-step-num { background: #3b82f6; color: white; width: 30px; height: 30px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: 700; flex-shrink: 0; font-size: 0.85rem; }
  .fm-step-content { flex: 1; color: #cbd5e1; }
  .fm-step-content strong { color: #e2e8f0; }

  .fm-table { width: 100%; border-collapse: collapse; margin: 14px 0; font-size: 0.9rem; }
  .fm-table th { background: #1e3a5f; color: #93c5fd; padding: 9px 12px; text-align: left; font-weight: 600; }
  .fm-table td { padding: 9px 12px; border-bottom: 1px solid #1e293b; color: #cbd5e1; }
  .fm-table tr:hover td { background: #1e293b; }
  .fm-good { color: #34d399; }
  .fm-bad { color: #f87171; }
  .fm-mid { color: #fbbf24; }

  .fm-pre { background: #0d1b2a; border: 1px solid #1e3a5f; border-radius: 8px; padding: 18px; overflow-x: auto; font-size: 0.82rem; line-height: 1.6; font-family: 'Courier New', monospace; color: #e2e8f0; }
  .fm-kw { color: #c084fc; }
  .fm-fn { color: #60a5fa; }
  .fm-cm { color: #475569; }
  .fm-nm { color: #fb923c; }

  .fm-tag { display: inline-block; background: #1e3a5f; color: #93c5fd; padding: 2px 9px; border-radius: 20px; font-size: 0.78rem; margin: 2px; }
  .fm-tag.g { background: #064e3b; color: #34d399; }
  .fm-tag.p { background: #2e1065; color: #c084fc; }
</style>

<div class="fm-wrap">

<div class="fm-hero">
  <h1>Flow Matching 在具身机器人中的作用</h1>
  <p>从数学原理到 pi0 实现，理解为什么 Flow Matching 成为机器人策略学习的核心技术</p>
</div>

<!-- 1 -->
<div class="fm-section">
  <h2>1. 机器人动作预测的核心难题</h2>
  <p>机器人需要从观测（图像、语言指令、关节状态）预测一段动作序列。这个问题的本质困难在于：</p>
  <div class="fm-card-grid">
    <div class="fm-card">
      <h3>多模态分布</h3>
      <p>同一个任务（"拿起杯子"）有无数种合理的执行方式。动作分布是<strong style="color:#f87171">多峰的</strong>，简单回归会预测出所有模式的均值——一个没有意义的中间动作。</p>
    </div>
    <div class="fm-card">
      <h3>高维连续空间</h3>
      <p>机械臂有 6-7 个关节，双臂有 14 个，加上夹爪。预测未来 50 步的动作序列意味着在 <strong style="color:#a78bfa">700 维空间</strong>中建模分布。</p>
    </div>
  </div>
  <div class="fm-highlight">
    <strong>核心问题：</strong>我们需要一个能表达复杂、多峰、高维连续分布的生成模型，同时推理速度要足够快（机器人实时控制需要 &lt;100ms）。
  </div>
</div>

<!-- 2 -->
<div class="fm-section">
  <h2>2. Flow Matching 是什么</h2>
  <p>Flow Matching 是一种生成模型，学习将简单分布（高斯噪声）变换为目标分布（真实动作）的<strong style="color:#60a5fa">连续流</strong>。</p>
  <h3>核心思想：学习速度场</h3>
  <p>不直接预测动作，而是预测一个向量场 <code style="color:#34d399">v(x, t)</code>，沿着这个场积分就能从噪声走到真实动作。</p>
  <div class="fm-formula">
    x<sub>t</sub> = t · ε + (1 - t) · x<sub>0</sub>
    <span class="fm-cmt">在噪声 ε 和真实动作 x₀ 之间线性插值，t ∈ [0, 1]</span>
  </div>
  <div class="fm-formula">
    u<sub>t</sub> = ε - x<sub>0</sub>
    <span class="fm-cmt">真实速度场：从 x₀ 指向 ε 的方向</span>
  </div>
  <div class="fm-formula">
    L = E[ ‖ v_θ(x<sub>t</sub>, t) - u<sub>t</sub> ‖² ]
    <span class="fm-cmt">训练目标：让模型预测的速度场匹配真实速度场</span>
  </div>
  <h3>推理：Euler 积分</h3>
  <p>从 t=1（纯噪声）出发，按照学到的速度场走 N 步到达 t=0（干净动作）：</p>
  <div class="fm-formula">
    x<sub>t+dt</sub> = x<sub>t</sub> + dt · v_θ(x<sub>t</sub>, t)
    <span class="fm-cmt">dt = -1/N，N 步（pi0 默认 N=10）</span>
  </div>
</div>

<!-- 3 交互演示 -->
<div class="fm-section">
  <h2>3. 交互演示：Flow Matching 去噪过程</h2>
  <p>下图模拟机器人末端执行器的 2D 轨迹分布。从高斯噪声出发，观察粒子如何收敛到双峰目标分布。</p>
  <div class="fm-demo-box">
    <canvas id="fmCanvas" width="560" height="340"></canvas>
    <div class="fm-controls">
      <button class="fm-btn" onclick="fmStart()">▶ 开始流动</button>
      <button class="fm-btn sec" onclick="fmReset()">↺ 重置</button>
      <button class="fm-btn sec" onclick="fmToggleTrails()">轨迹: <span id="fmTrailLabel">开</span></button>
    </div>
    <div class="fm-slider-row">
      <span>步数 N:</span>
      <input type="range" id="fmStepsSlider" min="3" max="30" value="10" oninput="fmUpdateSteps(this.value)">
      <span id="fmStepsVal">10</span>
      <span style="margin-left:16px">当前 t:</span>
      <span id="fmTimeVal" style="color:#60a5fa;font-weight:600">1.00</span>
    </div>
  </div>
</div>

<!-- 4 -->
<div class="fm-section">
  <h2>4. pi0 中的具体实现</h2>
  <p>pi0 将 Flow Matching 与 PaliGemma（视觉-语言模型）结合，条件化在观测上预测动作流。</p>
  <h3>训练流程</h3>
  <div class="fm-step">
    <div class="fm-step-num">1</div>
    <div class="fm-step-content"><strong>采样噪声和时间步</strong><br><span style="color:#94a3b8">从 Beta(1.5, 1) 采样 t，偏向 t≈1（噪声端），让模型多练习难的去噪步骤</span></div>
  </div>
  <div class="fm-step">
    <div class="fm-step-num">2</div>
    <div class="fm-step-content"><strong>构造 noisy action</strong><br><span style="color:#94a3b8">x_t = t·noise + (1-t)·action，在噪声和真实动作之间插值</span></div>
  </div>
  <div class="fm-step">
    <div class="fm-step-num">3</div>
    <div class="fm-step-content"><strong>Prefix + Suffix 前向传播</strong><br><span style="color:#94a3b8">图像/语言 token 作为 prefix（双向注意力），noisy action token 作为 suffix（因果注意力）</span></div>
  </div>
  <div class="fm-step">
    <div class="fm-step-num">4</div>
    <div class="fm-step-content"><strong>预测速度场并计算 MSE loss</strong><br><span style="color:#94a3b8">模型输出 v_t，目标是 u_t = noise - action</span></div>
  </div>

  <h3>对应代码（pi0.py）</h3>
  <div class="fm-pre"><span class="fm-cm"># compute_loss: 第 188-214 行</span>
noise = jax.random.normal(noise_rng, actions.shape)
time  = jax.random.beta(time_rng, <span class="fm-nm">1.5</span>, <span class="fm-nm">1</span>, batch_shape) * <span class="fm-nm">0.999</span> + <span class="fm-nm">0.001</span>

x_t = time * noise + (<span class="fm-nm">1</span> - time) * actions   <span class="fm-cm"># 插值</span>
u_t = noise - actions                          <span class="fm-cm"># 真实速度场</span>

v_t = self.action_out_proj(suffix_out[:, -self.action_horizon:])
<span class="fm-kw">return</span> jnp.mean(jnp.square(v_t - u_t), axis=<span class="fm-nm">-1</span>)  <span class="fm-cm"># MSE loss</span>

<span class="fm-cm"># sample_actions: Euler 积分</span>
dt = <span class="fm-nm">-1.0</span> / num_steps
<span class="fm-kw">def</span> <span class="fm-fn">step</span>(carry):
    x_t, time = carry
    v_t = model(x_t, time)
    <span class="fm-kw">return</span> x_t + dt * v_t, time + dt

x_0, _ = jax.lax.while_loop(cond, step, (noise, <span class="fm-nm">1.0</span>))</div>

  <h3>注意力结构</h3>
  <div class="fm-card" style="font-family:monospace;font-size:0.88rem;color:#94a3b8;">
    <div style="margin-bottom:8px">
      <span style="background:#1e3a5f;padding:3px 9px;border-radius:4px;color:#93c5fd">图像 tokens</span>
      <span style="margin:0 4px">|</span>
      <span style="background:#1e3a5f;padding:3px 9px;border-radius:4px;color:#93c5fd">语言 tokens</span>
      <span style="margin:0 4px">|</span>
      <span style="background:#2e1065;padding:3px 9px;border-radius:4px;color:#c084fc">state token</span>
      <span style="margin:0 4px">|</span>
      <span style="background:#064e3b;padding:3px 9px;border-radius:4px;color:#34d399">action tokens ×50</span>
    </div>
    <div style="color:#475569;font-size:0.78rem">
      ←────── 双向注意力（prefix）──────→ ←── 因果注意力（suffix）──→<br>
      action tokens 可以看所有 prefix，但 prefix 看不到 action tokens
    </div>
  </div>
</div>

<!-- 5 -->
<div class="fm-section">
  <h2>5. 为什么不用其他方法</h2>
  <table class="fm-table">
    <tr><th>方法</th><th>多模态</th><th>推理速度</th><th>训练稳定性</th><th>机器人适用性</th></tr>
    <tr><td>MSE 回归</td><td class="fm-bad">✗ 均值塌陷</td><td class="fm-good">极快</td><td class="fm-good">稳定</td><td class="fm-bad">差</td></tr>
    <tr><td>DDPM 扩散</td><td class="fm-good">✓</td><td class="fm-bad">慢（100步）</td><td class="fm-good">稳定</td><td class="fm-mid">可用但慢</td></tr>
    <tr><td>GAN</td><td class="fm-good">✓</td><td class="fm-good">快</td><td class="fm-bad">不稳定</td><td class="fm-bad">难训练</td></tr>
    <tr><td>VAE / CVAE</td><td class="fm-mid">部分</td><td class="fm-good">快</td><td class="fm-good">稳定</td><td class="fm-mid">表达能力有限</td></tr>
    <tr><td><strong style="color:#34d399">Flow Matching</strong></td><td class="fm-good">✓</td><td class="fm-good">快（10步）</td><td class="fm-good">稳定</td><td class="fm-good">最优</td></tr>
  </table>
  <div class="fm-highlight">
    Flow Matching 相比 DDPM 的关键优势：轨迹是<strong style="color:#60a5fa">直线</strong>（最优传输），不需要 100 步，10 步就够。这对机器人实时控制至关重要。
  </div>
</div>

<!-- 6 -->
<div class="fm-section">
  <h2>6. 在具身机器人中的实际意义</h2>
  <div class="fm-card-grid">
    <div class="fm-card">
      <h3>处理演示数据的多样性</h3>
      <p>人类示教数据天然多样——不同人抓同一个物体路径不同。Flow Matching 能学到这个分布，而不是崩溃到均值。</p>
      <div style="margin-top:8px"><span class="fm-tag g">多峰分布</span> <span class="fm-tag g">模仿学习</span></div>
    </div>
    <div class="fm-card">
      <h3>泛化到新场景</h3>
      <p>条件化在语言指令和图像上，同一个模型可以执行不同任务。Flow Matching 的生成能力让模型能插值到训练中未见过的情况。</p>
      <div style="margin-top:8px"><span class="fm-tag p">条件生成</span> <span class="fm-tag p">零样本泛化</span></div>
    </div>
    <div class="fm-card">
      <h3>动作平滑性</h3>
      <p>Flow 的连续性保证了生成的动作序列是平滑的，不会出现突变。这对机械臂安全控制很重要。</p>
      <div style="margin-top:8px"><span class="fm-tag">轨迹平滑</span> <span class="fm-tag">安全控制</span></div>
    </div>
    <div class="fm-card">
      <h3>实时推理</h3>
      <p>10 步 Euler 积分 + KV cache 复用（pi0 的 prefix 只算一次），在现代 GPU 上可以达到 &gt;10Hz 的控制频率。</p>
      <div style="margin-top:8px"><span class="fm-tag g">KV cache</span> <span class="fm-tag g">实时控制</span></div>
    </div>
  </div>
</div>

</div><!-- fm-wrap -->

<script>
(function() {
  const canvas = document.getElementById('fmCanvas');
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  let particles = [], targets = [], animId = null;
  let showTrails = true, numSteps = 10, currentStep = 0, isRunning = false;

  function randn() {
    let u = 0, v = 0;
    while (!u) u = Math.random();
    while (!v) v = Math.random();
    return Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v);
  }

  function generateTargets(n) {
    const t = [];
    for (let i = 0; i < n; i++) {
      if (Math.random() < 0.5)
        t.push({ x: W * 0.3 + randn() * 38, y: H * 0.45 + randn() * 38 });
      else
        t.push({ x: W * 0.7 + randn() * 38, y: H * 0.45 + randn() * 38 });
    }
    return t;
  }

  function fmInit() {
    targets = generateTargets(80);
    particles = targets.map(tgt => ({
      x: W / 2 + randn() * 85, y: H / 2 + randn() * 85,
      tx: tgt.x, ty: tgt.y, trail: [],
      color: tgt.x < W / 2 ? '#60a5fa' : '#a78bfa'
    }));
    currentStep = 0;
    document.getElementById('fmTimeVal').textContent = '1.00';
  }

  function fmDraw() {
    ctx.fillStyle = '#0f1117';
    ctx.fillRect(0, 0, W, H);
    targets.forEach(t => {
      ctx.beginPath(); ctx.arc(t.x, t.y, 4, 0, Math.PI * 2);
      ctx.fillStyle = t.x < W / 2 ? 'rgba(96,165,250,0.15)' : 'rgba(167,139,250,0.15)';
      ctx.fill();
    });
    ctx.font = '12px system-ui';
    ctx.fillStyle = 'rgba(96,165,250,0.6)'; ctx.fillText('动作模式 A', W * 0.17, H * 0.84);
    ctx.fillStyle = 'rgba(167,139,250,0.6)'; ctx.fillText('动作模式 B', W * 0.61, H * 0.84);
    particles.forEach(p => {
      if (showTrails && p.trail.length > 1) {
        ctx.beginPath(); ctx.moveTo(p.trail[0].x, p.trail[0].y);
        for (let i = 1; i < p.trail.length; i++) ctx.lineTo(p.trail[i].x, p.trail[i].y);
        ctx.strokeStyle = p.color === '#60a5fa' ? 'rgba(96,165,250,0.2)' : 'rgba(167,139,250,0.2)';
        ctx.lineWidth = 1; ctx.stroke();
      }
      ctx.beginPath(); ctx.arc(p.x, p.y, 4, 0, Math.PI * 2);
      ctx.fillStyle = p.color; ctx.fill();
    });
    const prog = currentStep / numSteps;
    ctx.fillStyle = '#1e293b'; ctx.fillRect(20, H - 22, W - 40, 7);
    ctx.fillStyle = '#3b82f6'; ctx.fillRect(20, H - 22, (W - 40) * prog, 7);
    ctx.fillStyle = '#94a3b8'; ctx.font = '11px system-ui';
    ctx.fillText('步骤 ' + currentStep + '/' + numSteps, 20, H - 28);
  }

  function fmStep() {
    if (currentStep >= numSteps) { isRunning = false; return; }
    const t = 1 - currentStep / numSteps;
    const dt = 1 / numSteps;
    particles.forEach(p => {
      const vx = (p.tx - p.x) / (t + 0.1) * dt * 1.1;
      const vy = (p.ty - p.y) / (t + 0.1) * dt * 1.1;
      if (showTrails) p.trail.push({ x: p.x, y: p.y });
      p.x += vx; p.y += vy;
    });
    currentStep++;
    document.getElementById('fmTimeVal').textContent = (1 - currentStep / numSteps).toFixed(2);
    fmDraw();
    if (currentStep < numSteps) animId = requestAnimationFrame(fmStep);
    else isRunning = false;
  }

  window.fmStart = function() {
    if (isRunning) return;
    if (currentStep >= numSteps) fmReset();
    isRunning = true; fmStep();
  };
  window.fmReset = function() {
    if (animId) cancelAnimationFrame(animId);
    isRunning = false; fmInit(); fmDraw();
  };
  window.fmToggleTrails = function() {
    showTrails = !showTrails;
    document.getElementById('fmTrailLabel').textContent = showTrails ? '开' : '关';
    fmDraw();
  };
  window.fmUpdateSteps = function(v) {
    numSteps = parseInt(v);
    document.getElementById('fmStepsVal').textContent = v;
    fmReset();
  };

  fmInit(); fmDraw();
})();
</script>
{{< /rawhtml >}}
