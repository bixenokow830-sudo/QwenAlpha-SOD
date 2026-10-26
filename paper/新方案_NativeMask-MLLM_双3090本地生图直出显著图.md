# QwenAlpha-SOD：全流程单 RTX 3090 的本地生成式 ORSI 显著目标检测最终方案

> 版本：v2.0 Final（2026-10-21，主模型确定为 Qwen-Image-2.1）  
> 目标：本地部署 Qwen-Image-2.1，输入一张光学遥感图像后生成 RGBA 主体图，并将模型生成的 Alpha 通道直接导出为**单通道灰度显著图**，最终在 ORSSD、EORSSD、ORSI-4199 上达到或逼近传统全监督 SOD 方法。  
> 硬件边界：**从环境验证、数据缓存、训练、验证、消融到最终推理，全流程始终只允许使用 1× NVIDIA RTX 3090 24 GB**。任何阶段都不得启用第二张 GPU、多卡并行或自动设备映射。不依赖在线 API。  
> 方案代号：**QwenAlpha-SOD**。

---

## 1. 先给结论

唯一主模型确定为 **Qwen-Image-2.1**。本方案不让模型输出类别、框、点、多边形或 SAM 提示，而是利用其原生 RGBA 生成能力，把 Alpha 通道定义为显著性概率图：

```text
原始 ORSI + “提取最显著目标并生成透明背景主体图”
        │
        ▼
Qwen3-VL 条件编码 + Qwen-Image-2.1 DiT
        │
        ▼
生成 RGBA 图像（RGB 为显著目标外观，A 为显著性）
        │  只取 Alpha
        ▼
单通道灰度显著图（0～255 PNG）
```

**最终交付物仍是普通灰度图**：黑色 0 表示背景、白色 255 表示显著目标、中间灰度表示不确定性。RGBA 只是模型内部的生成载体，RGB 在评测时完全丢弃。

核心技术不是提示模型“画一张黑白图”，而是完成三项针对性改造：

1. **RGBA 目标重构**：把每个训练标签构造成 `RGB=原图×GT，Alpha=GT`，将 SOD 对齐到 Qwen 已具备的透明主体提取能力。
2. **单卡缓存式 DiT-LoRA**：离线缓存 Qwen3-VL 条件特征和目标 RGBA latent，训练时只常驻 7B DiT、LoRA 与必要的 VAE 解码阶段。
3. **Alpha 指标对齐训练**：在原生 flow-matching 损失之外，对预测 Alpha 加入区域、IoU、边界、方向和频域监督；传统 SOD 模型只作为可选训练教师，不参与最终推理。

最终推理路径只有“ORSI → Qwen-Image-2.1 → Alpha → 灰度 PNG”，没有 SAM、传统 SOD 解码器或坐标渲染器。

---

## 2. 为什么必须新建这条路线

### 2.1 旧路线的两个上限

工作区已有实验已经暴露两类上限：

- **坐标/框/多边形路线**：MLLM 决定大致区域，但几何量化误差、漏实例和边界粗糙会直接恶化 MAE、F-measure；确定性渲染器无法补回模型没有输出的像素细节。
- **MLLM 提示 + 冻结视觉主干 + 小解码器路线**：即使提示接近理想，冻结特征仍不是针对 ORSI 显著性的特征，已有方案记录的三集平均上限约为 0.9072，难以追平 GeleNet/IPDiff。

QwenAlpha-SOD 同时打破这两点：模型输出的是包含显著性 Alpha 的像素图像，不是坐标；生成主干通过 LoRA 适配 ORSI-SOD，而不是完全冻结。

### 2.2 四篇参考工作的可继承部分

| 参考方法 | 已验证的关键思想 | 本方案中的继承方式 |
|---|---|---|
| MCCNet | 前景、边缘、背景和全局内容互补；BCE + IoU + F-measure 联合监督 | 训练期构造区域/边缘/背景三类监督；加入 metric-aware F loss |
| GeleNet | Transformer 全局建模；方向卷积增强任意朝向目标；全局到局部 | 保留 MLLM 全局语义能力，加入四方向边界一致性损失与多尺度条件 token |
| SAM2-RS | 参数高效双适配器；多尺度、频域、记忆先验；weighted BCE + weighted IoU | LoRA/Adapter 微调；难例原型记忆仅用于训练采样；采用加权结构损失 |
| IPDiff | 条件生成、动态迭代修正、多层先验；空间 + 边缘 + FFT 频域监督 | 主方案采用 1 次粗生成 + 1 次残差精修；加入空间-边缘-频域联合损失 |

### 2.3 与 IPDiff 的本质区别

IPDiff 是专用扩散式 SOD 网络，PVT-v2 提取先验，专用去噪网络在 10 个时间步内恢复掩码。QwenAlpha-SOD 则从通用图像理解/编辑基础模型出发，用 Qwen3-VL 理解 ORSI、由 DiT 条件生成 RGBA latent，并把原生 VAE 解码出的 Alpha 直接作为灰度显著图。传统 SOD 教师在部署时全部移除。

---

## 3. 最终模型决策

### 3.1 唯一主选：Qwen-Image-2.1

主线只维护 [Qwen-Image-2.1](https://huggingface.co/Qwen/Qwen-Image-2.1)。其视觉生成组件约 7B，使用 Qwen3-VL 编码文本与参考图像，DiT 负责生成，原生 VAE 支持 RGBA 与透明主体提取。它并非传统“只看图回答文字”的 VLM，而是能执行 `image + instruction → image` 的统一生成/编辑模型，功能上满足本课题。

选择它不是因为品牌本身，而是因为它同时满足：

1. 已开放本地权重，可离线部署；
2. 原生图像编辑路径无需另接 SAM；
3. 原生 Alpha 通道可以承载连续显著性概率；
4. Diffusers 已支持推理，[DiffSynth-Studio 已提供 DiT-LoRA 训练入口](https://github.com/modelscope/DiffSynth-Studio/blob/main/examples/qwen_image_21/model_training/lora/Qwen-Image-2.1.sh)；
5. 相比新近小型学术原型，来源、生态与论文可解释性更稳妥。

### 3.2 对“MLLM”称谓的边界

论文中应准确称其为“具备视觉语义条件理解的统一图像生成与编辑模型”，不要声称它等价于通用对话式 MLLM。[Qwen VLo](https://qwenlm.github.io/blog/qwen-vlo/) 才是 Qwen 官方定义的统一理解与生成 MLLM，但当前没有可用于本地微调的开放权重，不能承担本项目主线。

### 3.3 只保留一个技术回退

[Show-o2-1.5B](https://github.com/showlab/show-o) 仅作为 **P0/P1 失败后的回退模型**，不与 Qwen 同时开展完整实验。触发回退必须满足下列任一条件：

- Qwen 原生 RGBA VAE 的 Alpha 重建未达到第 4.2 节红线；
- 缓存条件特征后，100-step DiT-LoRA 仍无法控制在单卡峰值 22 GB 内；
- 单步训练时间导致一个 EORSSD epoch 超过 12 小时；
- 64 张训练集无法过拟合，且已排除数据、attention 后端和目标构造错误。

BAGEL、Emu3、Janus、Lance、UniLiP 等不进入主线，只在论文相关工作中讨论。不得用多卡资源挽救一个在单张 3090 条件下不可行的主方案。

### 3.4 主模型准入门槛

Qwen-Image-2.1 必须依次通过：

1. **权重与推理**：`CUDA_VISIBLE_DEVICES=<唯一指定卡>`，且进程内 `torch.cuda.device_count()==1`，通过 CPU offload 完成 8 张 ORSI 条件编辑；
2. **Alpha codec**：100 张 GT 构造 RGBA 后经 VAE 编解码，达到 Sα/Fmax/MAE/Boundary-F 红线；
3. **训练资源**：352×352、batch 1、LoRA rank 8 的 100 step 峰值显存 ≤22 GB，进程全程只能看到一张 GPU；
4. **可学习性**：64 张样本在 500～1000 step 内训练 Sα ≥0.98；
5. **条件依赖**：打乱输入 ORSI 后性能显著下降，证明不是只记住掩码先验。

只有五项全部通过，才进入三数据集正式训练；否则按第 3.3 节回退一次，不继续扩张候选清单。

---

## 4. 模型结构

### 4.1 总体结构

```text
ORSI + 固定任务指令
        │
        ▼
冻结的 Qwen3-VL 条件编码器 ──→ condition embeddings
                                      │
固定随机种子 ─→ 初始噪声              │
        │                             ▼
        └────────────────────→ Qwen-Image-2.1 DiT
                               （只训练 LoRA）
                                      │
                                  RGBA latent
                                      │
                                      ▼
                               冻结的原生 RGBA VAE
                                      │
                                 RGBA 输出图
                                      │
                                  提取 Alpha
                                      ▼
                           单通道 8-bit 灰度 PNG
```

最终只保存 Alpha 通道。输出 clamp 到 `[0,1]` 后映射为 8-bit 单通道 PNG；不二值化后再评测，不使用框、点、多边形、SAM 或传统 SOD 解码器。

### 4.2 原生 RGBA VAE 审计：第一道否决门槛

Qwen-Image-2.1 的 VAE 具有 16 倍空间压缩。352×352 输入对应约 22×22 latent，因此细边界和小目标可能在任何 DiT 训练之前就已损失。正式训练前必须对原生 VAE 做上限审计。

对每个样本构造目标：

```text
R,G,B = 原始 ORSI × GT Alpha
A     = GT 显著图
背景 RGB 置 0
```

执行 `目标 RGBA → VAE encode → VAE decode → 重建 Alpha`，在 100 张分层样本和完整训练集上报告：

- Sα ≥0.985；
- Fmax ≥0.985；
- MAE ≤0.005；
- 95% 样本 Boundary-F ≥0.970；
- 极小目标子集不能出现系统性消失。

若失败，**不替换 Qwen VAE、不另训 Mask Codec 来强行延续主线**，因为改变 latent 分布会破坏预训练 DiT 的生成空间；直接触发 Show-o2 回退或调整课题表述。

### 4.3 缓存式单卡训练

训练前离线完成：

- 固定训练增强版本并缓存每个 ORSI 的 Qwen3-VL 条件 embedding；
- 缓存每个目标 RGBA 的 VAE latent；
- 保存增强参数、源文件 hash 和缓存版本，避免缓存与标签错位；
- 验证集不做随机增强，缓存唯一确定版本。

训练时 Qwen3-VL 不进 GPU、不反向传播，只加载缓存 embedding；VAE encoder 不进训练循环。GPU 主要容纳 DiT、LoRA、激活，以及计算 Alpha 辅助损失时短暂使用的 VAE decoder。

### 4.4 Alpha 监督路径

DiT 仍学习 Qwen 原生 flow-matching 目标。为对齐 SOD 指标，从模型预测的 flow/velocity 还原估计的干净 latent `z0_hat`，通过冻结的 VAE decoder 得到 `A_hat`，再对 Alpha 计算第 5 节损失。RGB 只受原生生成损失约束，不进入 SOD 指标。

若每步解码导致显存超过 22 GB，则采用交替策略：3 个 step 仅做原生 latent 损失，第 4 个 step 加 Alpha 像素损失；不得改成独立 SOD head 作为最终输出。

### 4.5 单次生成优先，二次精修后置

正式主模型默认只生成一次。只有单次模型在 EORSSD 达到 Sα ≥0.92 后，才增加第二次编辑精修：

- Pass A：生成 RGBA 与 Alpha `A0`；
- Pass B：输入 ORSI、Pass A RGBA 和“修复漏检/边界”指令，生成最终 `A1`；
- 两个 pass 共享 LoRA；
- 若第二次生成三随机种子平均增益小于 0.3% Sα，最终方案删除 Pass B。

主表必须同时报告单次和两次生成的精度、延迟与显存。

---

## 5. 训练目标

设最终解码 Alpha 为 `A`，二值 GT 为 `G`，可选粗 Alpha 为 `A0`。

### 5.1 Qwen 原生生成损失

保留 Qwen-Image-2.1 的 flow-matching 目标 `L_flow`，监督完整 RGBA latent。训练不要求模型生成文本推理，也不修改 Qwen3-VL。Alpha 像素损失只能作为附加约束，不能完全替代 `L_flow`，否则容易破坏预训练生成空间。

### 5.2 空间结构损失

采用传统 SOD 中稳定有效的组合：

```text
L_region = L_wBCE(A,G) + L_wIoU(A,G)
L_metric = L_Fmeasure(A,G)
```

- `wBCE + wIoU` 对应 SAM2-RS/IPDiff 的结构监督；
- `F-measure loss` 对应 MCCNet 的 metric-aware 训练。

### 5.3 边界与方向损失

```text
E(X) = |X - AvgPool3x3(X)|
L_edge = BCE(E(A), E(G))
```

再分别用水平、垂直、主对角线、反对角线四组固定 Sobel/方向核提取边界，约束四方向梯度一致性，继承 GeleNet 对任意朝向目标的处理思想。

### 5.4 频域损失

参考 IPDiff，对预测与 GT 的 FFT 实部、虚部或幅度谱做 SSIM/L1 对齐：

```text
L_freq = 0.5 × [L_ssim(Re FFT(A), Re FFT(G))
              + L_ssim(Im FFT(A), Im FFT(G))]
```

其目标不是“让掩码更像自然图”，而是同时约束大区域低频形状和边缘高频结构。

### 5.5 蒸馏与生成排序

传统 SOD 方法只在训练阶段使用：

- 离线保存 GeleNet、SAM2-RS、IPDiff 或当前可复现强模型的软显著图；
- 教师集成图 `T` 只用于训练，不作为推理输入；
- 对 GT 清晰样本以 GT 为主，对边界模糊或存在标注争议的样本以 `0.8G + 0.2T` 构造软目标；
- 每张图可采样 2 个候选，按真实 GT 的 `0.4Sα + 0.3Fmax + 0.3Emax - MAE` 排序，构造 preferred/rejected 对进行短程偏好优化；验证集和测试集绝不参与偏好对构造。

这一步的作用是让生成模型学习“什么样的灰度图会被 SOD 指标判为好”，不是在推理时调用教师。

### 5.6 总损失建议

第一版使用：

```text
L = L_flow
  + 1.0 L_region
  + 0.5 L_metric
  + 0.5 L_edge
  + 0.2 L_direction
  + 0.2 L_freq
  + 0.3 L_distill
  + 0.5 L_region(A0,G)   # 仅二次精修阶段启用
```

第一阶段只用 `L_flow + L_region` 验证可学习性；其余项逐一加入。损失权重必须在训练集内部验证集上确定；禁止根据三套公开测试集反复调权重。

---

## 6. 数据协议

### 6.1 主实验：严格复现传统方法协议

三套数据分别训练、分别测试：

| 数据集 | 训练 | 测试 |
|---|---:|---:|
| ORSSD | 600 | 200 |
| EORSSD | 1400 | 600 |
| ORSI-4199 | 2000 | 2199 |

这是与 MCCNet、GeleNet、SAM2-RS、IPDiff 公平对比的主结果。

### 6.2 联合模型：只作为附加实验

可训练一个统一模型覆盖三套数据，但必须先做图像 SHA-256 与近重复检索。EORSSD 是 ORSSD 的扩展，直接把所有训练集拼接可能造成跨数据集测试泄漏。联合实验必须：

1. 固定所有测试集；
2. 从联合训练候选中删除与任一测试图相同或近重复的图；
3. 报告去重前后样本数及重复清单；
4. 联合模型结果不能替代“各数据集独立训练”的主表结果。

### 6.3 样本格式

每个训练样本是：

```json
{
  "instruction": "Extract the most visually salient object from this optical remote sensing image as an RGBA image. Preserve the salient object and make the background transparent.",
  "input_image": "xxx.jpg",
  "target_rgba": "xxx_rgba.png",
  "target_alpha": "xxx_mask.png",
  "sample_id": "dataset_split_xxx"
}
```

`target_rgba` 由数据脚本确定性生成：RGB 为 `input_image × target_alpha`，透明背景 RGB 固定为 0，A 为原始 GT。增强必须对 ORSI、RGBA 和 Alpha 同步：水平/垂直翻转、90°旋转、任意角小旋转、尺度裁剪、颜色/亮度扰动、轻度模糊与云雾扰动。不能对 Alpha 使用会产生灰边的普通双线性插值，几何变换应使用 nearest 或明确的软标签策略。所有增强版本在缓存条件 embedding 前离线固化。

### 6.4 难例平衡

建立以下采样标签：极小目标、多目标、无显著目标、阴影、低对比、复杂背景、细长结构、孔洞结构、超大目标。每个 batch 至少包含一种难例；无显著目标样本单独提高采样权重，防止模型“任何图都生成白块”。

---

## 7. 全流程严格单卡的落地配置

### 7.1 推荐环境

- Ubuntu 22.04 或 Windows 11 + WSL2；
- CUDA 12.x；PyTorch、Transformers、Diffusers 与 DiffSynth-Studio 使用一次冒烟测试通过后锁定的精确版本；
- 使用包含 [2026-10-09 segmented-attention 修复](https://github.com/modelscope/DiffSynth-Studio/pull/1732)的 DiffSynth-Studio；训练和推理均显式 `use_flex_attention=False`；
- 单进程 Accelerate + gradient checkpointing；
- 系统内存建议至少 64 GB，模型、缓存和数据放在 NVMe；
- 不在第一轮引入 NF4、FP8 或自定义 kernel，先建立 bf16 正确性基线；只有 bf16 超显存且训练稳定后才单独验证量化；
- 不启用多卡 DeepSpeed/FSDP，不允许训练框架自动发现其他 GPU。

不建议原生 Windows 直接搭建正式训练环境，主要风险是 FlashAttention、NCCL 和分布式训练兼容性。

### 7.2 GPU 使用纪律

1. 每次运行显式设置 `CUDA_VISIBLE_DEVICES=<当日分配卡号>`；映射后程序内部只使用 `cuda:0`；
2. 每个入口脚本启动时断言 `torch.cuda.device_count() == 1`，否则立即退出；
3. 禁止 `device_map="auto"`、多卡 Accelerate、DDP、FSDP、张量并行和流水线并行；
4. 推理需要卸载时只允许 GPU↔CPU/NVMe 顺序卸载，不允许把模块卸载到另一张 GPU；
5. 所有实验记录物理卡号、峰值显存、持续时间和进程内可见 GPU 数；
6. 发现指定卡已有他人进程时不抢占、不终止进程，改用空闲时段。

### 7.3 单张 3090 显存预算（起始配置）

| 项目 | 目标显存 | 说明 |
|---|---:|---|
| Qwen DiT bf16/分层卸载 | 14～16 GB | 7B 生成主体；Qwen3-VL 不进入训练 GPU |
| 激活与梯度 | 3～4 GB | 352×352、checkpointing、micro-batch=1 |
| LoRA 与优化器 | 0.5～1.5 GB | rank 8 起步；只训练注意力投影 |
| VAE decoder/Alpha loss | 1～2 GB | 交替解码；VAE encoder 已离线缓存 |
| CUDA/采样/碎片余量 | 2～3 GB | 总峰值目标 ≤22 GB |

若单卡 OOM，按以下顺序降配：

1. 确认 Qwen3-VL embedding 和目标 VAE latent 已完全离线缓存；
2. 开启 checkpointing，micro-batch 固定为 1，梯度累积提高到 16～32；
3. Alpha 像素损失由每步改为每 4 步一次；
4. LoRA 仅挂到 DiT 最后 8～12 层，rank 16→8；
5. 384→352，仍不通过则 320 仅作诊断，不作为最终配置；
6. 尝试框架已验证的分层 CPU offload；
7. 仍超过 22 GB 或速度不可接受，触发 Show-o2-1.5B 单卡回退；不得启用任何第二张 GPU。

### 7.4 建议训练参数

| 参数 | 起始值 |
|---|---|
| 精度 | bf16 起步；量化另做消融 |
| LoRA | r=8 起步，alpha=16，dropout=0.05；资源通过后再试 r=16 |
| micro-batch | 单卡 1 |
| 梯度累积 | 16 |
| 有效 batch | 16 |
| 学习率 | LoRA 1e-5 起步；稳定后搜索 5e-6、2e-5、5e-5 |
| 优化器 | AdamW，weight decay 0.01 |
| 调度 | 5% warmup + cosine |
| 输入/输出 | 352×352 RGBA；最终 Alpha 恢复原图尺寸 |
| 采样 | 固定 sampler、步数与 seed；初始 20 steps |
| 训练轮数 | 10～30 epoch，按内部验证集早停 |
| 随机种子 | 至少 3 个最终种子 |

### 7.5 三阶段冻结策略

1. **Stage A：审计与缓存**  
   冻结全部权重，完成 Alpha codec 审计、Qwen3-VL embedding 与目标 latent 缓存。
2. **Stage B：DiT-LoRA 可学习性**  
   冻结 Qwen3-VL 和 VAE，只训练 DiT LoRA；先用 `L_flow + L_region` 完成 64 张过拟合。
3. **Stage C：指标对齐精修**  
   仍冻结基座，加入 F、边界、方向、频域和可选蒸馏损失；VAE decoder 只负责可微 Alpha 解码。
4. **Stage D：可选二次编辑**  
   仅在单次生成通过 P3 后训练共享 LoRA 的 refine pass。

绝不全参微调 Qwen-Image-2.1，也不训练 Qwen3-VL 和 VAE。

---

## 8. 指标目标与分级验收

四篇参考论文给出的可核查锚点如下。主表统一报告 `Sα / Fmax / Emax / MAE`。

| 层级 | 方法 | EORSSD | ORSSD | ORSI-4199 |
|---|---|---|---|---|
| 最低传统强基线 | MCCNet | .9327 / .8904 / .9755 / .0066 | .9437 / .9155 / .9800 / .0087 | .8746 / .8690 / .9413 / .0316 |
| 竞争目标 | GeleNet | .9376 / .8923 / .9828 / .0064 | .9469 / .9254 / .9860 / .0079 | .8862 / .8842 / .9544 / .0264 |
| 冲刺目标 | IPDiff | .9461 / .9033 / .9861 / .0044 | .9557 / .9362 / .9915 / .0054 | .8907 / .8885 / .9572 / .0237 |

验收分为四级：

- **L0 技术可行**：64 张过拟合 Sα ≥ 0.98，说明模型能学习“输出掩码图”。
- **L1 可投稿原型**：三集分别达到 MCCNet 的 98% 相对性能，且生成图无彩色污染或纹理伪影。
- **L2 竞争性结果**：至少两个数据集达到或超过 GeleNet；第三个数据集差距不超过 1% Sα。
- **L3 冲刺结果**：三数据集平均与 IPDiff 的 Sα 差距 ≤0.5%，同时 MAE 不高于 GeleNet。

“接近传统 SOD”的正式定义建议采用 L2，不应只凭视觉样例或单一 Sα 宣称接近。

---

## 9. 实验路线与 go/no-go

### P0：环境和数据审计（1～2 天）

- 验证唯一指定的 3090、bf16、segmented attention，并确认进程内只可见一张 GPU；
- 固化三个数据集划分与文件 hash；
- 用同一评测脚本复算已有显著图，确认指标与论文/现有记录一致；
- 输出数据泄漏审计报告。

### P1：Qwen 准入测试（2～4 天）

- CPU offload 跑 8 张 ORSI 条件编辑，确认能输出有效 RGBA；
- 用 100 张分层 GT 审计原生 RGBA VAE 的 Alpha 重建上限；
- 固化增强并缓存条件 embedding/目标 latent；
- 在 352×352、batch 1、rank 8 下完成 100 step DiT-LoRA 冒烟；
- 记录峰值显存、step 时间、loss、Alpha 可视化和进程内可见 GPU 数；
- **Go**：codec 过红线且峰值 ≤22 GB；**No-go**：触发一次 Show-o2 回退，不修改 Qwen codec 强行继续。

### P2：64 张过拟合试验（2～3 天）

- 固定 48 训练 + 16 验证；
- 先只用 `L_flow + L_region`；
- 检查是否生成全黑、全白、彩色纹理、轮廓漂移；
- **Go**：训练集 Sα ≥0.98、验证集有明显学习；否则定位为条件注入或输出 token 映射错误。

### P3：单数据集最小闭环（3～5 天）

- 先跑 EORSSD，因为规模适中且已有最多可比方法；
- 对比仅 `L_flow`、`+L_region`、完整 Alpha 损失；此阶段只做 1-pass；
- **Go**：EORSSD Sα ≥0.92 且 MAE ≤0.01；否则暂不跑三个数据集。

### P4：结构损失与蒸馏（5～8 天）

按顺序加入：F-measure → 边界 → 方向 → 频域 → 教师软目标。每次只改一个因素；若某项三随机种子平均增益 <0.2% Sα，则不进入最终模型。偏好优化和二次编辑只在上述项完成后作为可选扩展。

### P5：三数据集主实验（5～10 天）

- 独立训练三个模型；
- 每个最终配置跑 3 个随机种子，报告均值±标准差；
- 最终配置冻结后再评测试集；
- 输出参数量、可训练参数、峰值显存、训练 GPU 小时、单图延迟、能耗估算。

### P6：泛化与失败分析（3～5 天）

- ORSI-4199→EORSSD、ORSSD→EORSSD 跨集测试；
- 分析小目标、低对比、阴影、多目标、空场景、细长/孔洞结构；
- 报告最差 50 张，而不是只展示成功样例。

预计 1 周得到 Qwen 单卡可行/不可行结论，2 周得到 EORSSD 最小闭环，完成三集主实验约 4～6 周。

---

## 10. 必做消融

| 编号 | 消融 | 要回答的问题 |
|---|---|---|
| A1 | 灰度 RGB 目标 vs RGBA/Alpha 目标 | Alpha 重构是否优于“画黑白图” |
| A2 | 文本提示生图 vs ORSI 条件编辑 | 模型是否真的利用输入图像 |
| A3 | 冻结基座 vs DiT-LoRA | 遥感域适配是否必要 |
| A4 | 1-pass vs 2-pass refine | 动态修正是否有效 |
| A5 | BCE+IoU vs +F vs +edge vs +FFT | 四篇论文中的监督如何贡献 |
| A6 | 无蒸馏 vs 单教师 vs 多教师 | 传统 SOD 知识是否提升生成模型 |
| A7 | 无方向损失 vs 四方向损失 | 飞机、船、道路等任意朝向目标是否受益 |
| A8 | 352 vs 384/512 | 小目标收益是否值得显存和时间 |
| A9 | 固定 seed vs 多 seed | 生成随机性对 SOD 指标的影响 |
| A10 | Qwen-Image-2.1 vs Show-o2-1.5B 回退 | 结果是否依赖特定生成架构 |

还必须做“输入打乱”反事实：给模型错误 ORSI、纯黑图或打乱图像 token。如果输出几乎不变，说明模型只学到数据集掩码先验，不能算真正的 SOD。

---

## 11. 风险与应对

| 风险 | 概率 | 影响 | 应对 |
|---|---:|---:|---|
| Alpha 出现灰雾、孔洞或透明边污染 | 高 | 高 | 原生 VAE 上限审计；Alpha 区域/边界损失；不过线即回退 |
| 16× VAE 压缩导致小目标消失 | 高 | 高 | 100 张分层审计、384/512 消融、小目标重采样；不另训 codec 掩盖问题 |
| Qwen 单卡训练 OOM或过慢 | 中高 | 高 | 缓存 embedding/latent、checkpointing、交替 Alpha loss、分层 offload；仍失败则回退 Show-o2 |
| attention 后端产生错误梯度 | 中 | 高 | 使用已合并修复的 DiffSynth；显式 `use_flex_attention=False`；锁定依赖版本 |
| 脚本意外启用多张 GPU | 中 | 高 | 显式 `CUDA_VISIBLE_DEVICES`、启动时断言仅一张可见卡、禁用自动 device map |
| 生成随机性导致指标波动 | 中 | 中 | 固定 sampler/步数/seed；最终 3 个训练种子；多 seed 只作消融 |
| 只学会全黑/全白数据先验 | 中 | 高 | 空场景平衡、输入打乱反事实、区域面积校准 |
| 二次生成增加延迟但收益小 | 中 | 低 | 设置 +0.3% Sα 保留门槛，否则改单 pass |
| 训练集过小导致灾难性遗忘 | 中 | 中 | PEFT、低学习率、增强、早停；不全参微调 |
| 与 IPDiff 被认为只是“换个大模型” | 中 | 高 | 强调 RGBA→Alpha 任务重构、缓存式单卡训练及 SOD 对齐的可微 Alpha 监督；完整消融 |
| 联合数据存在 ORSSD/EORSSD 泄漏 | 高 | 高 | 主实验分开训练；联合实验做 hash 和近重复去重 |

---

## 12. 论文可写的创新点

建议只主张以下四个可验证创新，不夸大“首个”或“超越”：

1. **任务重构**：把 ORSI-SOD 表述为统一生成式多模态模型的条件视觉生成任务，直接预测包含显著性的视觉 latent，而不是文本坐标、提示或传统判别式 logits。
2. **Alpha 显著性表征**：首次在本工作范围内把 Qwen 原生 RGBA 主体生成重构为连续 Alpha 显著性预测，避免让自然图像生成器模拟黑白绘图风格。
3. **SOD 对齐的生成训练**：从 DiT 的预测干净 latent 可微解码 Alpha，把区域、metric-aware、边界、方向和频域监督统一到生成路径，并用传统 SOD 教师完成可选训练期蒸馏。
4. **资源受限部署协议**：通过冻结 Qwen3-VL/VAE、离线缓存条件 embedding 与目标 latent，在默认单张 24 GB RTX 3090 上完成 DiT-LoRA 适配，并公开严格的显存与失败门槛。

可选题目：

- *QwenAlpha-SOD: Generating Saliency Maps through Native Alpha Prediction for Optical Remote Sensing Images*
- *From RGBA Generation to Saliency Detection: Adapting Qwen-Image for ORSI-SOD*

---

## 13. 最小代码工程规划

建议在新目录 `ds_sod/qwen_alpha_sod/` 实现，避免破坏现有实验：

```text
qwen_alpha_sod/
├── configs/
│   ├── qwen_alpha_lora_eorssd.yaml
│   └── eval.yaml
├── data/
│   ├── build_pairs.py
│   ├── deduplicate.py
│   ├── cache_conditions.py
│   └── cache_rgba_latents.py
├── models/
│   ├── qwen_alpha_lora.py
│   ├── alpha_decode.py
│   └── losses.py
├── audit_alpha_vae.py
├── train_lora.py
├── train_refine.py
├── infer.py
├── eval_sod.py
└── export_gray_png.py
```

首批只实现 `build_pairs.py`、`audit_alpha_vae.py`、`cache_conditions.py`、`cache_rgba_latents.py` 和 `train_lora.py`。在 Alpha VAE 上限和 64 张过拟合通过前，不写二次精修、大规模训练或论文包装代码。

---

## 14. 最终决策规则

- 如果 P1 的原生 RGBA VAE Alpha 重建不能达到 MAE ≤0.005：Qwen 主线停止，说明其原生 latent 表征不适合精确 ORSI 掩码；触发 Show-o2 回退。
- 如果 Qwen 100-step 冒烟峰值 >22 GB，或预计单个 EORSSD epoch >12 小时：Qwen 主线停止，不使用多卡维持它。
- 如果 P2 不能过拟合 64 张：方案停止，优先检查条件注入与视觉 token，而不是盲目扩数据。
- 如果 P3 的 EORSSD 低于 Sα 0.92：不跑三集，先修模型；禁止靠后处理包装结果。
- 如果最终达到 MCCNet、但未达到 GeleNet：论文定位为“首个本地生成式 MLLM 基线 + 精度/统一能力权衡”。
- 如果达到 GeleNet：可以表述为“与强判别式 ORSI-SOD 方法竞争”。
- 只有达到或超过 IPDiff 且同协议复评后，才可以写“state of the art”。

---

## 15. 建议立即执行的第一周任务

1. 锁定空闲卡号并验证 `CUDA_VISIBLE_DEVICES`，记录基线显存；
2. 下载 Qwen-Image-2.1，在单张 3090 上用 CPU offload 完成 8 张 ORSI→RGBA 推理；
3. 从 EORSSD 分层抽取 100 张，构造 `RGB=ORSI×GT、A=GT` 的 RGBA 标签；
4. 完成原生 VAE 的 Alpha 重建审计；不过红线立即停止 Qwen 路线；
5. 生成离线增强版本，缓存 Qwen3-VL 条件 embedding 和目标 RGBA latent；
6. 使用已修复的 segmented attention，在 352×352、batch 1、rank 8 下跑 100 step DiT-LoRA，确认峰值 ≤22 GB；
7. 在 EORSSD 固定 64 张样本做 500～1000 step 过拟合，只使用 `L_flow + L_region`；
8. 导出 Alpha 为单通道灰度 PNG，用统一脚本计算 Sα/Fmax/Emax/MAE，并执行输入打乱反事实；
9. 五项准入门槛全部通过才进入第二周；失败则只回退 Show-o2-1.5B，不同时开启更多模型。

第一周的交付物必须包括：环境锁文件、显存/速度日志、100 张 VAE 重建表、64 张过拟合曲线、预测灰度图和明确的 Go/No-go 结论。整个项目禁止使用第二张 GPU。

