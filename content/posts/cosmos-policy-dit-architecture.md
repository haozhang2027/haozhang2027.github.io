---
title: "Cosmos Policy DiT 架构图"
date: 2026-05-22
draft: false
math: false
tags:
  - robotics
  - diffusion-policy
  - world-model
  - cosmos
  - nvidia
  - architecture
categories:
  - 源码解析
---

{{< rawhtml >}}
<style>
.cpd-root {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  color: #e2e8f0;
  max-width: 900px;
  margin: 0 auto;
  padding: 20px 0;
}
.cpd-root h2 {
  color: #60a5fa;
  border-bottom: 1px solid #2d3748;
  padding-bottom: 8px;
}
.cpd-root p {
  color: #cbd5e1;
  line-height: 1.7;
}
.cpd-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 12px;
  margin: 20px 0;
}
.cpd-stat {
  background: #1e293b;
  border: 1px solid #2d3748;
  border-radius: 8px;
  padding: 12px 16px;
  text-align: center;
}
.cpd-stat-val {
  font-size: 1.4em;
  font-weight: bold;
  color: #60a5fa;
}
.cpd-stat-label {
  font-size: 0.85em;
  color: #94a3b8;
  margin-top: 4px;
}
.cpd-svg-wrap {
  overflow-x: auto;
  margin: 30px 0;
  padding: 20px;
  background: #0f1117;
  border-radius: 12px;
  border: 1px solid #2d3748;
}
.cpd-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
  margin: 16px 0;
  padding: 12px 16px;
  background: #1e293b;
  border-radius: 8px;
}
.cpd-legend-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.85em;
  color: #94a3b8;
}
.cpd-legend-color {
  width: 16px;
  height: 16px;
  border-radius: 3px;
}
</style>

<div class="cpd-root">

<h2>Cosmos Policy DiT — Wan2.1 1.3B</h2>

<p>基于 Wan2.1 视频生成模型的 DiT backbone，用于在 latent space 做 diffusion denoising。输入 noisy latent video + timestep + T5 text embedding，输出 predicted clean latent。</p>

<div class="cpd-stats">
  <div class="cpd-stat"><div class="cpd-stat-val">1.3B</div><div class="cpd-stat-label">Parameters</div></div>
  <div class="cpd-stat"><div class="cpd-stat-val">1536</div><div class="cpd-stat-label">Hidden Dim</div></div>
  <div class="cpd-stat"><div class="cpd-stat-val">30</div><div class="cpd-stat-label">Layers</div></div>
  <div class="cpd-stat"><div class="cpd-stat-val">12</div><div class="cpd-stat-label">Heads</div></div>
  <div class="cpd-stat"><div class="cpd-stat-val">8960</div><div class="cpd-stat-label">FFN Dim</div></div>
  <div class="cpd-stat"><div class="cpd-stat-val">1764</div><div class="cpd-stat-label">Seq Length (LIBERO)</div></div>
</div>

<div class="cpd-legend">
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#3b82f6"></div>Input / Output</div>
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#8b5cf6"></div>Embedding</div>
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#f59e0b"></div>Self-Attention</div>
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#10b981"></div>Cross-Attention</div>
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#ef4444"></div>FFN</div>
  <div class="cpd-legend-item"><div class="cpd-legend-color" style="background:#6366f1"></div>Modulation (AdaLN)</div>
</div>

<div class="cpd-svg-wrap">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 1580" width="720" height="1580">
  <defs>
    <marker id="cpd-arrow" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto">
      <polygon points="0 0, 8 3, 0 6" fill="#94a3b8"/>
    </marker>
    <linearGradient id="cpd-grad-blue" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:#1e40af"/>
      <stop offset="100%" style="stop-color:#3b82f6"/>
    </linearGradient>
    <linearGradient id="cpd-grad-purple" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:#5b21b6"/>
      <stop offset="100%" style="stop-color:#8b5cf6"/>
    </linearGradient>
    <filter id="cpd-shadow">
      <feDropShadow dx="0" dy="2" stdDeviation="3" flood-opacity="0.3"/>
    </filter>
  </defs>

  <!-- Background -->
  <rect width="720" height="1580" fill="#0f1117" rx="8"/>

  <!-- ========== INPUTS ========== -->
  <!-- Noisy Latent -->
  <rect x="40" y="30" width="200" height="60" rx="8" fill="#1e3a5f" stroke="#3b82f6" stroke-width="1.5" filter="url(#cpd-shadow)"/>
  <text x="140" y="55" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">Noisy Latent x_t</text>
  <text x="140" y="75" text-anchor="middle" fill="#94a3b8" font-size="10">(B, 16, 9, 28, 28)</text>

  <!-- Timestep -->
  <rect x="260" y="30" width="200" height="60" rx="8" fill="#1e3a5f" stroke="#3b82f6" stroke-width="1.5" filter="url(#cpd-shadow)"/>
  <text x="360" y="55" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">Timestep σ</text>
  <text x="360" y="75" text-anchor="middle" fill="#94a3b8" font-size="10">(B,)</text>

  <!-- T5 Text -->
  <rect x="480" y="30" width="200" height="60" rx="8" fill="#1e3a5f" stroke="#3b82f6" stroke-width="1.5" filter="url(#cpd-shadow)"/>
  <text x="580" y="55" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">T5 Text Embedding</text>
  <text x="580" y="75" text-anchor="middle" fill="#94a3b8" font-size="10">(B, 512, 4096)</text>

  <!-- Arrows from inputs -->
  <line x1="140" y1="90" x2="140" y2="120" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>
  <line x1="360" y1="90" x2="360" y2="120" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>
  <line x1="580" y1="90" x2="580" y2="120" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>

  <!-- ========== EMBEDDING LAYER ========== -->
  <rect x="20" y="120" width="680" height="200" rx="10" fill="none" stroke="#8b5cf6" stroke-width="1.5" stroke-dasharray="4,4"/>
  <text x="360" y="140" text-anchor="middle" fill="#a78bfa" font-size="11" font-weight="bold">EMBEDDING LAYER</text>

  <!-- Patch Embed -->
  <rect x="40" y="155" width="200" height="70" rx="6" fill="#2d1b69" stroke="#8b5cf6" stroke-width="1" filter="url(#cpd-shadow)"/>
  <text x="140" y="175" text-anchor="middle" fill="#e2e8f0" font-size="11" font-weight="bold">3D Patch Embed</text>
  <text x="140" y="192" text-anchor="middle" fill="#94a3b8" font-size="9">patch=(1,2,2)</text>
  <text x="140" y="206" text-anchor="middle" fill="#94a3b8" font-size="9">Linear(64→1536)</text>
  <text x="140" y="220" text-anchor="middle" fill="#cbd5e1" font-size="9">→ (B, 1764, 1536)</text>

  <!-- Time Embed -->
  <rect x="260" y="155" width="200" height="70" rx="6" fill="#2d1b69" stroke="#8b5cf6" stroke-width="1" filter="url(#cpd-shadow)"/>
  <text x="360" y="175" text-anchor="middle" fill="#e2e8f0" font-size="11" font-weight="bold">Time Embedding</text>
  <text x="360" y="192" text-anchor="middle" fill="#94a3b8" font-size="9">sin(256) → MLP(1536)</text>
  <text x="360" y="206" text-anchor="middle" fill="#94a3b8" font-size="9">→ SiLU → Linear(9216)</text>
  <text x="360" y="220" text-anchor="middle" fill="#cbd5e1" font-size="9">→ (B, 6, 1536)</text>

  <!-- Text Embed -->
  <rect x="480" y="155" width="200" height="70" rx="6" fill="#2d1b69" stroke="#8b5cf6" stroke-width="1" filter="url(#cpd-shadow)"/>
  <text x="580" y="175" text-anchor="middle" fill="#e2e8f0" font-size="11" font-weight="bold">Text Embedding</text>
  <text x="580" y="192" text-anchor="middle" fill="#94a3b8" font-size="9">Linear(4096→1536)</text>
  <text x="580" y="206" text-anchor="middle" fill="#94a3b8" font-size="9">→ GELU → Linear(1536)</text>
  <text x="580" y="220" text-anchor="middle" fill="#cbd5e1" font-size="9">→ (B, 512, 1536)</text>

  <!-- RoPE -->
  <rect x="40" y="240" width="200" height="40" rx="6" fill="#2d1b69" stroke="#8b5cf6" stroke-width="1"/>
  <text x="140" y="262" text-anchor="middle" fill="#e2e8f0" font-size="10">+ RoPE 3D Position Emb</text>
  <text x="140" y="276" text-anchor="middle" fill="#94a3b8" font-size="9">(T'=9, H=14, W=14)</text>

  <!-- Arrow to transformer block -->
  <line x1="360" y1="320" x2="360" y2="360" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>

  <!-- ========== TRANSFORMER BLOCK (×30) ========== -->
  <rect x="60" y="360" width="600" height="820" rx="12" fill="#111827" stroke="#4b5563" stroke-width="2" filter="url(#cpd-shadow)"/>
  
  <!-- Block header -->
  <rect x="230" y="348" width="260" height="26" rx="13" fill="#1f2937" stroke="#4b5563" stroke-width="1"/>
  <text x="360" y="366" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">WanAttentionBlock  × 30</text>

  <!-- Modulation -->
  <rect x="100" y="395" width="520" height="50" rx="6" fill="#1e1b4b" stroke="#6366f1" stroke-width="1"/>
  <text x="360" y="415" text-anchor="middle" fill="#a5b4fc" font-size="11" font-weight="bold">⚡ AdaLN Modulation</text>
  <text x="360" y="432" text-anchor="middle" fill="#94a3b8" font-size="9">learnable_param(1,6,1536) + time_proj(B,6,1536) → 6 scale/shift vectors: γ₁,β₁,α₁,γ₂,β₂,α₂</text>

  <!-- Arrow -->
  <line x1="360" y1="445" x2="360" y2="470" stroke="#94a3b8" stroke-width="1" marker-end="url(#cpd-arrow)"/>

  <!-- Self-Attention -->
  <rect x="100" y="470" width="520" height="180" rx="8" fill="#1c1917" stroke="#f59e0b" stroke-width="1.5"/>
  <text x="360" y="495" text-anchor="middle" fill="#fbbf24" font-size="12" font-weight="bold">Self-Attention</text>
  
  <rect x="130" y="510" width="460" height="30" rx="4" fill="#292524" stroke="#78716c" stroke-width="0.5"/>
  <text x="360" y="530" text-anchor="middle" fill="#e2e8f0" font-size="10">LayerNorm(x) × (1 + γ₁) + β₁</text>

  <rect x="130" y="548" width="460" height="30" rx="4" fill="#292524" stroke="#78716c" stroke-width="0.5"/>
  <text x="360" y="568" text-anchor="middle" fill="#e2e8f0" font-size="10">Q, K, V = Linear(1536→1536) each  |  12 heads × 128 dim</text>

  <rect x="130" y="586" width="460" height="30" rx="4" fill="#292524" stroke="#78716c" stroke-width="0.5"/>
  <text x="360" y="606" text-anchor="middle" fill="#e2e8f0" font-size="10">Apply RoPE 3D on Q, K  →  Flash Attention  →  Linear(1536→1536)</text>

  <rect x="130" y="624" width="460" height="18" rx="4" fill="none" stroke="#6366f1" stroke-width="1" stroke-dasharray="3,3"/>
  <text x="360" y="637" text-anchor="middle" fill="#a5b4fc" font-size="9">x = x + attn_output × α₁</text>

  <!-- Arrow -->
  <line x1="360" y1="650" x2="360" y2="675" stroke="#94a3b8" stroke-width="1" marker-end="url(#cpd-arrow)"/>

  <!-- Cross-Attention -->
  <rect x="100" y="675" width="520" height="160" rx="8" fill="#022c22" stroke="#10b981" stroke-width="1.5"/>
  <text x="360" y="700" text-anchor="middle" fill="#34d399" font-size="12" font-weight="bold">Cross-Attention (Text Conditioning)</text>

  <rect x="130" y="715" width="460" height="30" rx="4" fill="#14352b" stroke="#2d6a4f" stroke-width="0.5"/>
  <text x="360" y="735" text-anchor="middle" fill="#e2e8f0" font-size="10">Q = Linear(LayerNorm(x))  |  K, V = Linear(text_emb)</text>

  <rect x="130" y="753" width="460" height="30" rx="4" fill="#14352b" stroke="#2d6a4f" stroke-width="0.5"/>
  <text x="360" y="773" text-anchor="middle" fill="#e2e8f0" font-size="10">12 heads × 128 dim  →  Attention(Q_video, K_text, V_text)  →  Linear</text>

  <!-- Text input arrow -->
  <line x1="660" y1="190" x2="680" y2="190" stroke="#10b981" stroke-width="1" stroke-dasharray="3,3"/>
  <line x1="680" y1="190" x2="680" y2="750" stroke="#10b981" stroke-width="1" stroke-dasharray="3,3"/>
  <line x1="680" y1="750" x2="620" y2="750" stroke="#10b981" stroke-width="1" stroke-dasharray="3,3" marker-end="url(#cpd-arrow)"/>
  <text x="690" y="470" text-anchor="start" fill="#34d399" font-size="9" transform="rotate(90, 690, 470)">text emb</text>

  <rect x="130" y="791" width="460" height="18" rx="4" fill="none" stroke="none"/>
  <text x="360" y="810" text-anchor="middle" fill="#94a3b8" font-size="9">x = x + cross_attn_output</text>

  <!-- Arrow -->
  <line x1="360" y1="835" x2="360" y2="860" stroke="#94a3b8" stroke-width="1" marker-end="url(#cpd-arrow)"/>

  <!-- FFN -->
  <rect x="100" y="860" width="520" height="140" rx="8" fill="#1f0a0a" stroke="#ef4444" stroke-width="1.5"/>
  <text x="360" y="885" text-anchor="middle" fill="#f87171" font-size="12" font-weight="bold">Feed-Forward Network</text>

  <rect x="130" y="900" width="460" height="30" rx="4" fill="#292524" stroke="#78716c" stroke-width="0.5"/>
  <text x="360" y="920" text-anchor="middle" fill="#e2e8f0" font-size="10">LayerNorm(x) × (1 + γ₂) + β₂</text>

  <rect x="130" y="938" width="460" height="30" rx="4" fill="#292524" stroke="#78716c" stroke-width="0.5"/>
  <text x="360" y="958" text-anchor="middle" fill="#e2e8f0" font-size="10">Linear(1536→8960) → GELU → Linear(8960→1536)</text>

  <rect x="130" y="976" width="460" height="18" rx="4" fill="none" stroke="#6366f1" stroke-width="1" stroke-dasharray="3,3"/>
  <text x="360" y="989" text-anchor="middle" fill="#a5b4fc" font-size="9">x = x + ffn_output × α₂</text>

  <!-- Loop indicator -->
  <path d="M 80 1000 C 50 1000, 50 400, 80 400" fill="none" stroke="#4b5563" stroke-width="1.5" stroke-dasharray="4,4"/>
  <text x="35" y="700" text-anchor="middle" fill="#9ca3af" font-size="11" font-weight="bold" transform="rotate(-90, 35, 700)">× 30 layers</text>

  <!-- Time embed arrow to modulation -->
  <line x1="360" y1="225" x2="360" y2="320" stroke="#6366f1" stroke-width="1" stroke-dasharray="3,3"/>

  <!-- ========== Repeat indicator end ========== -->

  <!-- Arrow to head -->
  <line x1="360" y1="1180" x2="360" y2="1220" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>

  <!-- ========== HEAD ========== -->
  <rect x="140" y="1220" width="440" height="130" rx="10" fill="#1e293b" stroke="#64748b" stroke-width="1.5" filter="url(#cpd-shadow)"/>
  <text x="360" y="1245" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">Output Head</text>

  <rect x="170" y="1260" width="380" height="28" rx="4" fill="#1e1b4b" stroke="#6366f1" stroke-width="0.5"/>
  <text x="360" y="1279" text-anchor="middle" fill="#a5b4fc" font-size="10">AdaLN: param(1,2,1536) + time_emb → 2 scales</text>

  <rect x="170" y="1296" width="380" height="28" rx="4" fill="#111827" stroke="#4b5563" stroke-width="0.5"/>
  <text x="360" y="1315" text-anchor="middle" fill="#e2e8f0" font-size="10">LayerNorm × (1+γ) + β → Linear(1536 → 64)</text>

  <text x="360" y="1342" text-anchor="middle" fill="#94a3b8" font-size="9">Unpatchify: (B, 1764, 64) → rearrange → (B, 16, 9, 28, 28)</text>

  <!-- Arrow to output -->
  <line x1="360" y1="1350" x2="360" y2="1400" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#cpd-arrow)"/>

  <!-- ========== OUTPUT ========== -->
  <rect x="160" y="1400" width="400" height="60" rx="8" fill="#1e3a5f" stroke="#3b82f6" stroke-width="1.5" filter="url(#cpd-shadow)"/>
  <text x="360" y="1425" text-anchor="middle" fill="#e2e8f0" font-size="12" font-weight="bold">Predicted Clean Latent x₀</text>
  <text x="360" y="1445" text-anchor="middle" fill="#94a3b8" font-size="10">(B, 16, 9, 28, 28)</text>

  <!-- ========== ANNOTATIONS ========== -->
  <!-- Sequence length annotation -->
  <rect x="160" y="1490" width="400" height="70" rx="8" fill="#1e293b" stroke="#2d3748" stroke-width="1"/>
  <text x="360" y="1510" text-anchor="middle" fill="#94a3b8" font-size="10">Sequence: 9 latent frames × 14×14 spatial patches = 1764 tokens</text>
  <text x="360" y="1528" text-anchor="middle" fill="#94a3b8" font-size="10">Each token: 1536-dim vector processed by 30 transformer layers</text>
  <text x="360" y="1546" text-anchor="middle" fill="#94a3b8" font-size="10">Conditioned on: timestep σ (AdaLN) + text instruction (cross-attn)</text>

</svg>
</div>

</div>
{{< /rawhtml >}}
