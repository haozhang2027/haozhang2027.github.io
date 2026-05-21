---
title: "Cosmos Policy 源码解析：get_action 完整推理流程"
date: 2026-05-22
draft: false
math: true
tags:
  - robotics
  - diffusion-policy
  - world-model
  - cosmos
  - nvidia
categories:
  - 源码解析
---

## 1. 概述

[Cosmos Policy](https://github.com/nvidia-cosmos/cosmos-policy) 是 NVIDIA 基于 Cosmos Video World Model 构建的机器人策略模型。它的核心思想极其优雅：**把 action prediction 转化为 video latent space 中的 diffusion 生成问题**。

具体来说，模型把多种模态的信号——摄像头图像、机器人本体感知 (proprioception)、动作序列 (action chunk)、价值估计 (value)——统一编码为"伪视频帧"，然后利用预训练 video diffusion model 的强大生成能力，在 latent space 中联合推理出动作、未来状态预测和价值估计。

这篇文章将从源码层面，完整剖析 `get_action` 函数的推理流程，覆盖每一个被调用的函数及其作用。

---

## 2. get_action 函数签名

函数位于 `cosmos_policy/experiments/robot/cosmos_utils.py` L851：

```python
def get_action(
    cfg,
    model: torch.nn.Module,
    dataset_stats: dict,
    obs: Dict[str, Any],
    task_label_or_embedding: Any,
    seed: int = 1,
    randomize_seed: bool = False,
    num_denoising_steps_action: int = 5,
    generate_future_state_and_value_in_parallel: bool = True,
    worker_id: int = 0,
    batch_size: int = 1,
) -> List[np.ndarray]:
```

**输入：**
- `cfg` — 实验配置（包含 suite 类型、是否使用 proprio/wrist image 等开关）
- `model` — 加载好的 Cosmos Policy diffusion model
- `dataset_stats` — 训练集统计量（action/proprio 的 min/max，用于归一化）
- `obs` — 当前观测 dict，包含各摄像头图像和 proprio
- `task_label_or_embedding` — 语言指令字符串或预计算的 T5 embedding

**输出：** dict 包含：
- `actions` — 预测的动作序列 (action chunk)
- `future_image_predictions` — 未来图像预测
- `value_prediction` — 价值估计 [0, 1]
- `generated_latent` — 完整生成的 latent（用于可视化/调试）

---

## 3. 输入预处理阶段

### 3.1 T5 Text Embedding

```python
if isinstance(task_label_or_embedding, str):
    text_embedding = get_t5_embedding_from_cache(task_label_or_embedding)
```

`get_t5_embedding_from_cache` (L387) 的工作流程：

1. 检查内存 cache `t5_text_embeddings_cache`
2. Cache miss → 调用 T5-11B encoder 编码语言指令
3. 输出 shape: `(1, 512, 1024)` — 512 个 token position，1024 维 hidden state
4. 结果持久化到磁盘，避免重复计算

这个 embedding 后续作为 DiT 的 **cross-attention conditioning signal**，引导模型生成与任务指令一致的动作。

### 3.2 图像预处理

```python
all_camera_images = prepare_images_for_model(all_camera_images, cfg)
```

`prepare_images_for_model` (L528) 执行：

1. `np.stack` → `(N, H, W, 3)`
2. 可选垂直翻转（某些 suite 摄像头倒装）
3. 可选 JPEG compression (quality=95)，匹配训练时数据增强
4. `resize_images` → 统一到 `224×224`
5. 可选 center crop（90% 面积裁剪后 resize 回 224×224）

不同 suite 的摄像头配置：

| Suite | 摄像头 |
|-------|--------|
| LIBERO | 1 wrist + 1 primary (third-person) |
| RoboCasa | 1 wrist + 1 primary + 1 secondary |
| ALOHA | 2 wrist (左右) + 1 primary |

### 3.3 Proprio 归一化

```python
if cfg.normalize_proprio:
    proprio = rescale_proprio(proprio, dataset_stats, non_negative_only=False, scale_multiplier=1.0)
```

`rescale_proprio` (L651) 做简单的线性映射：

$$
x_{norm} = 2 \cdot \frac{x - x_{min}}{x_{max} - x_{min}} - 1
$$

将关节角度/速度从原始物理范围映射到 $[-1, +1]$，使其与 latent space 的数值范围兼容。

---

## 4. 构建 Image Sequence（核心设计）

这是整个方法最精妙的部分。

### 4.1 设计原理

Cosmos VAE 使用 **temporal compression**：每 4 帧原始图像压缩为 1 个 latent frame。第一帧是特殊的 anchor frame（1帧→1 latent）。

关键常量：
```python
COSMOS_IMAGE_SIZE = 224
COSMOS_TEMPORAL_COMPRESSION_FACTOR = 4
```

`duplicate_array` 函数（位于 `cosmos_policy/utils/utils.py` L41）把单帧图像重复 4 次：

```python
def duplicate_array(arr, total_num_copies=4):
    return np.stack([arr] * total_num_copies)  # (H,W,C) → (4, H, W, C)
```

这样 4 帧相同图像经 VAE encode 后恰好坍缩为 **1 个 latent frame**，实现"一个概念占一个 latent slot"的设计。

### 4.2 Sequence Layout

以 LIBERO 为例，完整的 image sequence 布局：

| Slot | 帧数 | 内容 | Placeholder 类型 | 说明 |
|------|------|------|-----------------|------|
| 0 | 1 | blank | 全零 | VAE anchor frame |
| 1 | 4 | current proprio | 全零 → latent 注入 | 后续在 latent space 注入 proprio 向量 |
| 2 | 4 | current wrist image | 真实图像 ×4 | 手腕摄像头当前帧 |
| 3 | 4 | current primary image | 真实图像 ×4 | 第三人称摄像头当前帧 |
| 4 | 4 | **action chunk** | 全零 → 模型生成 | 模型需要在这个 slot 生成动作 |
| 5 | 4 | future proprio | 全零 → 模型生成 | 预测未来本体状态 |
| 6 | 4 | future wrist image | 当前 wrist copy → 模型生成 | 预测未来手腕图像 |
| 7 | 4 | future primary image | 当前 primary copy → 模型生成 | 预测未来第三人称图像 |
| 8 | 4 | value | 全零 → 模型生成 | 预测成功概率 |

**总帧数** = 1 + 8×4 = 33 帧原始图像 → VAE encode 后 = **9 个 latent frames**

注意 future image slots 用当前图像的 copy 做 placeholder（而非全零），给模型一个合理的生成起点。

### 4.3 data_batch 构建

```python
raw_image_sequence = np.concatenate(image_sequence, axis=0)      # (33, 224, 224, 3)
raw_image_sequence = np.expand_dims(raw_image_sequence, axis=0)  # (1, 33, 224, 224, 3)
raw_image_sequence = np.tile(raw_image_sequence, (batch_size, 1, 1, 1, 1))  # (B, 33, 224, 224, 3)
raw_image_sequence = np.transpose(raw_image_sequence, (0, 4, 1, 2, 3))      # (B, 3, 33, 224, 224)
```

最终的 `data_batch` dict 包含：
- `video`: `(B, 3, 33, 224, 224)` uint8 — 伪视频输入
- `t5_text_embeddings`: `(B, 512, 1024)` — 语言条件
- `proprio`: `(B, proprio_dim)` — 归一化后的本体感知
- 各种 `*_latent_idx`: 告诉模型每个 slot 在 latent 序列中的位置

---

## 5. 模型推理：generate_samples_from_batch

### 5.1 整体流程

位于 `cosmos_policy/models/policy_text2world_model.py` L702：

```python
generated_latent_with_action, orig_clean_latent_frames = model.generate_samples_from_batch(
    data_batch,
    n_sample=batch_size,
    num_steps=num_denoising_steps_action,  # 默认 5
    seed=seed,
    use_variance_scale=cfg.use_variance_scale,
    return_orig_clean_latent_frames=True,
)
```

内部依次执行三个核心步骤。

### 5.2 get_x0_fn_from_batch — 条件构建

位于 `cosmos_policy/models/policy_video2world_model.py` L569，这是推理的核心：

**Step 1: 组装 Conditioner**

```python
condition, uncondition = self.conditioner.get_condition_uncondition(data_batch)
```

Conditioner（`cosmos_policy/conditioner.py`）将 T5 embedding 打包为 cross-attention 条件。当所有 embedder 的 dropout = 0 时，`uncondition = None`，跳过 CFG 的额外 forward pass，推理速度翻倍。

**Step 2: VAE Encode**

```python
latent_state = self.tokenizer.encode(raw_video) * sigma_data
# Shape: (B, 16, 9, 28, 28)
```

将 `(B, 3, 33, 224, 224)` 的像素视频编码为 `(B, 16, 9, 28, 28)` 的 latent：
- Channel: 3 → 16 (latent channels)
- Temporal: 33 frames → 9 latent frames (1+32/4)
- Spatial: 224×224 → 28×28 (compression factor = 8)

**Step 3: 设置 Video Conditioning Mask**

```python
condition.set_video_condition(
    gt_frames=latent_state,
    num_conditional_frames=N  # 前 N 个 latent frame 作为条件
)
```

创建 binary mask `(B, 1, 9, 28, 28)`：前 N 帧 = 1（conditioning），后面 = 0（generation target）。对于 LIBERO，N=4 意味着 slot 0-3（blank + proprio + wrist + primary）是条件帧。

**Step 4: Proprio Latent Injection**

```python
replace_latent_with_proprio(condition.gt_frames, proprio_tensor, proprio_latent_idx)
```

将归一化后的 proprio 向量直接写入对应 slot 的 latent frame。具体做法是将 proprio 值 tile 填满 `16×28×28=12544` 个 latent 元素。

**Step 5: 返回 x0_fn 闭包**

```python
def x0_fn(noise_x, sigma):
    return self.denoise(noise_x, sigma, condition).x0
```

### 5.3 Diffusion Sampling

位于 `cosmos_policy/modules/cosmos_sampler.py`：

```python
# 1. 采样初始噪声
x_sigma_max = randn(B, 16, 9, 28, 28) * sigma_max  # σ_max ≈ 80

# 2. 可选 variance scaling 增加多样性
sigma_max *= uniform(1.0, 3.0)
sigma_min *= uniform(0.1, 1.0)

# 3. ODE 求解
samples = self.sampler(
    x0_fn,
    x_sigma_max,
    num_steps=5,           # 4 ODE steps + 1 final clean step
    sigma_max=sigma_max,   # ≈80
    sigma_min=sigma_min,   # ≈0.002
    solver_option="2ab",   # Adams-Bashforth 2阶
)
```

Sigma schedule 使用 Karras et al. 的公式：

$$
\sigma_i = \left(\sigma_{max}^{1/\rho} + \frac{i}{N-1}\left(\sigma_{min}^{1/\rho} - \sigma_{max}^{1/\rho}\right)\right)^\rho
$$

其中 $\rho = 7$。从纯噪声 $\sigma_{max}$ 逐步降到近似干净 $\sigma_{min}$。

### 5.4 Conditioning 机制（denoise 内部）

每步 denoising 时，Video2World 的 `denoise` 方法执行 **frame replace**：

```python
# 强制把 conditioning 帧设回 GT latent
xt = gt_frames * condition_mask + xt * (1 - condition_mask)
```

这意味着：
- **Conditioning 帧**（当前观测）在每步都被强制恢复为 clean latent
- **Generation 帧**（action/future/value）正常参与 denoising
- 不使用 Classifier-Free Guidance（推理时跳过 uncondition 分支）

---

## 6. 结果提取

### 6.1 Action 提取

```python
actions = extract_action_chunk_from_latent_sequence(
    generated_latent_with_action,
    action_shape=(cfg.chunk_size, ACTION_DIM),  # e.g. (8, 7)
    action_indices=action_indices
)
```

`extract_action_chunk_from_latent_sequence` (L557) 的工作原理：

1. 取 action slot 对应的 latent frame: `(B, 16, 28, 28)`
2. Flatten: `(B, 12544)`
3. 计算: `num_chunks = 12544 // (chunk_size × action_dim)` = 224
4. Reshape: `(B, 224, chunk_size, action_dim)`
5. 取 mean: `(B, chunk_size, action_dim)`

**为什么要 averaging？** 训练时，action 被 tile 224 次填满 12544 维 latent。推理时模型生成的是这个 tiled pattern，对 224 份 copy 取平均可以消除噪声，恢复准确的 action。

### 6.2 Action 反归一化

```python
if cfg.unnormalize_actions:
    actions = unnormalize_actions(actions, dataset_stats)
```

`unnormalize_actions` (L623) 是归一化的逆运算：

$$
x = \frac{x_{norm} + 1}{2} \cdot (x_{max} - x_{min}) + x_{min}
$$

从 $[-1, +1]$ 映射回原始物理单位（meters, radians）。

### 6.3 Value 提取

```python
value_prediction = extract_value_from_latent_sequence(generated_latent_with_action, value_indices)
value_prediction = (value_prediction + 1) / 2  # [-1,+1] → [0,1]
value_prediction = torch.clamp(value_prediction, min=0, max=1)
```

`extract_value_from_latent_sequence` (L602)：取 value slot 的 latent frame → flatten → 对所有 12544 元素取 mean → 得到标量值。映射到 [0,1] 后表示任务成功概率。

### 6.4 Future Image 提取

```python
future_image_predictions = get_future_images_from_generated_samples(
    model, generated_latent_with_action.clone(), cfg,
    orig_clean_latent_frames, INDICES_TO_REPLACE, ...
)
```

`get_future_images_from_generated_samples` (L706)：

1. **`undo_latent_injection`** — 把非图像 slot（proprio/action/value）的 latent 恢复为原始 clean 值。这些 slot 被注入了非图像数据，直接 decode 会产生视觉噪声
2. **`model.decode()`** — VAE decoder 将 latent → 像素视频 `(B, 3, 33, 224, 224)`
3. 从 future image slot 对应的帧位置提取预测图像

---

## 7. 完整 Call Chain

```
get_action()
│
├── get_t5_embedding_from_cache()
│     └── T5-11B encoder → (1, 512, 1024)
│
├── prepare_images_for_model()
│     └── resize/crop → (N, 224, 224, 3) uint8
│
├── rescale_proprio()
│     └── linear map → [-1, +1]
│
├── Build image_sequence
│     ├── duplicate_array() × N slots
│     └── concat + transpose → (B, 3, 33, 224, 224)
│
├── model.generate_samples_from_batch()
│   │
│   ├── get_x0_fn_from_batch()
│   │   ├── conditioner → cross-attention condition (T5 emb)
│   │   ├── tokenizer.encode() → latent (B, 16, 9, 28, 28)
│   │   ├── set_video_condition() → conditioning mask
│   │   ├── replace_latent_with_proprio() → inject proprio
│   │   └── return x0_fn closure
│   │
│   ├── Sample noise: x ~ N(0, σ_max² I)
│   │
│   └── CosmosPolicySampler (ODE, 5 steps)
│       ├── Karras sigma schedule: [80, ..., 0.002]
│       ├── Adams-Bashforth 2AB solver (4 steps)
│       │     └── x0_fn(x_noisy, σ) → DiT forward → x0_pred
│       └── Final clean step → output latent (B, 16, 9, 28, 28)
│
├── extract_action_chunk_from_latent_sequence()
│     └── index → flatten → tile-average → (B, chunk_size, action_dim)
│
├── unnormalize_actions()
│     └── [-1,+1] → physical units (m, rad)
│
├── extract_value_from_latent_sequence()
│     └── index → flatten → mean → [0, 1] success probability
│
└── get_future_images_from_generated_samples()
      ├── undo_latent_injection()
      ├── VAE decode → pixel video
      └── index future frames → predicted images
```

---

## 8. 与论文的对应

| 论文概念 | 代码实现 |
|---------|---------|
| Unified latent sequence | image_sequence 的 slot 布局设计 |
| Multi-output prediction | 一次 diffusion 同时生成 action + future state + value |
| World model pre-training | 继承 Cosmos Video2World 的 DiT 权重 |
| Frame conditioning | `denoise()` 中的 frame replace 策略 |
| Action-as-latent | action 被 tile 编码进 latent frame，extract 时 averaging |
| Value function | value slot 编码为标量填满 latent，extract 时 mean |

论文的核心贡献在于证明了：**video world model 的预训练表征可以直接用于 policy learning**，无需从头训练 action prediction 网络。通过巧妙的 latent slot 设计，所有模态共享同一个 diffusion 生成过程。

---

## 9. 关键设计洞察

1. **不输入真正的 video** — 单帧观测被组装成伪视频序列，利用 temporal VAE 的压缩特性让每个"概念"占一个 latent frame

2. **非视觉信号的 latent 编码** — action/proprio/value 通过 tiling 填满整个 latent frame (12544 维)，extraction 时 averaging 消除噪声

3. **Conditioning = Frame Replace** — 不需要修改模型架构，直接复用 Video2World 的 conditioning 机制

4. **预训练能力复用** — 整个系统站在 Cosmos video model 的肩膀上，policy fine-tuning 只需学会在特定 slot 生成正确的 action pattern

5. **5步推理的 tradeoff** — 相比图像生成需要 50+ 步，action 精度要求低得多，5 步 (Adams-Bashforth 2AB) 足够且保证实时性

6. **无 CFG** — 推理时所有 dropout=0，跳过 uncondition 分支，节省一半计算量
