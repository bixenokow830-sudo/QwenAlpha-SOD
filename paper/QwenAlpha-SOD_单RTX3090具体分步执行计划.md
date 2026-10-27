# QwenAlpha-SOD：单 RTX 3090 具体分步执行计划

> 版本：v2.0（2026-10-26，可中断恢复版）  
> 执行主体：全程由 Codex 独立完成方案、代码、检查与结果整理，不调用 Harness。  
> 计算位置：通过 VS Code Remote-SSH 连接 SSH 主机 `wangsiyi`，在远端完成模型部署、数据缓存、GPU 训练和评测。  
> 远端写入边界：所有新增或修改内容必须位于 `/data2/wangsiyi` 及其子目录；禁止 `sudo`，禁止写入其他目录。  
> GPU 硬约束：从推理、缓存、训练、验证、消融到导出，全流程只使用 **1× RTX 3090 24 GB**；任何时刻进程只能看到一张 GPU，峰值显存目标不超过 22 GB。
> 恢复目标：任何阶段均可在安全点暂停；即使 VS Code 断线、GPU 临时让出或 Codex 任务中断，也能仅依靠远端状态文件、日志和 checkpoint 判断“已完成什么、下一步做什么”。

---

## 1. 最终目标和产物

输入一张光学遥感图像，Qwen-Image-2.1 生成 RGBA 主体图，最终只提取 Alpha 通道并保存为标准单通道灰度 PNG：

```text
ORSI + 固定任务指令
        ↓
冻结的 Qwen3-VL 条件编码
        ↓
Qwen-Image-2.1 DiT-LoRA 条件生成
        ↓
冻结的 RGBA VAE 解码
        ↓
提取 Alpha → L 模式、0～255 灰度 PNG
```

最终必须产出：

1. QwenAlpha-SOD 代码与锁定环境；
2. ORSSD、EORSSD、ORSI-4199 三套独立训练结果；
3. `Sα / Fmax / Emax / MAE` 主表；
4. 显存、速度、训练时长和可训练参数统计；
5. 消融、跨数据集测试、失败案例和最差 50 张分析；
6. 可复现的单卡推理入口和单通道灰度预测图。

---

## 2. 不可违反的单卡与远端规则

### 2.1 远端目录

建议固定为：

```text
/data2/wangsiyi/qwenalpha-sod/     # 代码、配置、日志、结果
/data2/wangsiyi/models/            # 模型权重
/data2/wangsiyi/datasets/          # ORSI 数据集
/data2/wangsiyi/cache/             # embedding、latent 和临时缓存
```

首次创建前，在 VS Code 的 `wangsiyi` 远端终端中验证：

```bash
test "$(realpath -e /data2/wangsiyi)" = "/data2/wangsiyi"
test ! -L /data2/wangsiyi
```

遇到意外符号链接、同名非目录或解析路径越出 `/data2/wangsiyi` 时立即停止。不得使用 `sudo`。

### 2.2 唯一 GPU 入口

每次运行先由课题组确定一张空闲物理卡，例如物理 GPU 0：

```bash
export CUDA_DEVICE_ORDER=PCI_BUS_ID
export CUDA_VISIBLE_DEVICES=0
python -c "import torch; assert torch.cuda.device_count()==1; print(torch.cuda.get_device_name(0))"
```

若分配的是物理 GPU 1，只把 `CUDA_VISIBLE_DEVICES` 改成 `1`；程序内部仍统一使用 `cuda:0`。

所有 Python 入口必须重复断言：

```python
assert torch.cuda.is_available()
assert torch.cuda.device_count() == 1
```

禁止事项：

- 禁止 `device_map="auto"`；
- 禁止 DDP、多卡 Accelerate、FSDP、张量并行和流水线并行；
- 禁止用第二张 GPU 做验证、消融或最终复跑；
- OOM 时只能降低配置、CPU/NVMe offload 或停止路线；
- 不得抢占或终止其他人的 GPU 进程。

### 2.3 每次运行的资源检查

```bash
nvidia-smi
python -m qwenalpha_sod.tools.check_runtime --max-vram-gb 22 --require-one-gpu
```

每个任务保存：物理 GPU 号、进程内可见 GPU 数、峰值显存、开始/结束时间、Git 提交号、配置文件和随机种子。

---

## A. 可中断、可恢复执行机制

### A.1 唯一事实来源

远端工程必须维护以下状态文件：

```text
/data2/wangsiyi/qwenalpha-sod/state/
├── progress.yaml          # 当前阶段和下一动作
├── history.jsonl          # 只追加的状态变更历史
├── active_run.txt         # 当前实验目录；无任务时为空
└── locks/
    └── gpu.lock           # 本项目自己的单卡运行锁
```

`progress.yaml` 建议格式：

```yaml
project: qwenalpha-sod
updated_at: "2026-10-26T19:30:00+08:00"
host: wangsiyi
remote_root: /data2/wangsiyi/qwenalpha-sod
physical_gpu: 0
visible_gpu_count: 1
stage: S4
step: S4.3
status: paused            # pending/running/paused/passed/failed/blocked
reason: gpu_released_for_lab_member
active_run: runs/S4_smoke/eorssd/20261026_193000
last_safe_checkpoint: runs/S4_smoke/eorssd/20261026_193000/checkpoints/step_0080.ckpt
last_verified_artifact: runs/S4_smoke/eorssd/20261026_193000/metrics/step_0080.json
next_action: resume_from_step_0080_and_finish_to_step_0100
resume_command: bash scripts/run_single_gpu.sh 0 python -m qwenalpha_sod.train --config configs/smoke_100step.yaml --resume runs/S4_smoke/eorssd/20261026_193000/checkpoints/step_0080.ckpt
```

所有状态更新采用原子写入：先写 `progress.yaml.tmp`，验证 YAML 可解析后使用同文件系统的原子 rename 覆盖正式文件。`history.jsonl` 只追加，不重写历史。

### A.2 子步骤状态和完成标记

每个子步骤只能处于以下状态之一：

| 状态 | 含义 | 是否允许进入下一步 |
|---|---|---:|
| `pending` | 尚未开始 | 否 |
| `running` | 正在执行 | 否 |
| `paused` | 已保存安全状态，等待恢复 | 否 |
| `passed` | 输出和验收均完成 | 是 |
| `failed` | 已执行但未通过门槛 | 否 |
| `blocked` | 缺数据、磁盘、用户决定或外部条件 | 否 |

子步骤通过后创建：

```text
state/done/S4.3.done.json
```

完成标记至少包含输入 hash、配置 hash、代码提交号、输出路径、验收结果和完成时间。只有 `.done.json` 存在且其中的输出校验仍通过，才能跳过该步骤。

### A.3 四种暂停场景

#### 计划内暂停

完成当前 optimizer step 或当前缓存 shard 后：

1. 写完整 checkpoint/manifest；
2. 验证文件可读取；
3. 更新 `progress.yaml` 为 `paused`；
4. 写清 `next_action` 和完整 `resume_command`；
5. 释放 GPU，保留 tmux 会话日志。

#### VS Code 或网络断线

长任务必须在 VS Code 远端终端的 `tmux` 会话中运行：

```bash
tmux new -s qas_s4_smoke
```

VS Code 断线时任务继续运行。重新连接后：

```bash
tmux ls
tmux attach -t qas_s4_smoke
```

断线本身不等于暂停；先读取日志和 `progress.yaml`，不得重复启动同一任务。

#### 需要主动让出 GPU

训练循环应每个 optimizer step 检查：

```text
<active_run>/control/PAUSE_REQUESTED
```

请求安全暂停：

```bash
touch <active_run>/control/PAUSE_REQUESTED
```

程序检测到后完成当前 step，保存模型、LoRA、优化器、调度器、AMP、随机数和数据游标状态，验证 checkpoint 后正常退出。禁止优先使用 `kill -9`。

#### 进程崩溃或主机重启

1. 不删除现场；
2. 检查最后一条完整 JSONL 日志；
3. 从最近一个通过完整性检查的 checkpoint 恢复；
4. 临时文件、缺尾日志和未完成 shard 标记为无效；
5. 重跑最多丢失一个 checkpoint 周期的工作。

### A.4 Checkpoint 必须包含的状态

- DiT LoRA 权重；
- optimizer、scheduler、AMP scaler；
- global step、epoch、当前样本/批次游标；
- Python、NumPy、PyTorch CPU/CUDA RNG 状态；
- sampler 和固定生成 seed；
- 数据清单、缓存 manifest、配置和基座模型 hash；
- 最近验证指标和最佳 checkpoint 路径。

Checkpoint 先保存为 `.tmp`，成功重新加载后再原子改名为 `.ckpt`。至少保留 `last`、`best` 和最近 3 个周期 checkpoint。

### A.5 每次恢复前的固定检查

恢复任务前依次执行：

1. `realpath` 验证仍在 `/data2/wangsiyi`；
2. 查看 `state/progress.yaml` 与 `active_run.txt`；
3. `tmux ls`，确认没有同一任务仍在运行；
4. `nvidia-smi`，确认分配 GPU 空闲且不终止他人进程；
5. 设置单一 `CUDA_VISIBLE_DEVICES`；
6. 断言 `torch.cuda.device_count()==1`；
7. 验证 checkpoint、配置、数据和缓存 hash；
8. 先运行 `--resume-dry-run`，只加载并打印下一 step；
9. dry-run 通过后执行 `progress.yaml` 中记录的恢复命令。

### A.6 Codex 交接记录

每次阶段结束或准备暂停时，Codex必须更新：

```text
reports/NEXT_ACTION.md
```

内容固定为：

```markdown
# 当前状态
- 已完成：
- 当前阶段/步骤：
- 最后安全 checkpoint：
- 已验证输出：
- 当前问题：
- 下一条唯一命令：
- 预期通过条件：
```

恢复任务时先读取该文件、`progress.yaml` 和当前 run 的末尾日志，不依赖聊天记忆推断进度。

---

## 3. 远端工程结构

```text
/data2/wangsiyi/qwenalpha-sod/
├── configs/
│   ├── base_single3090.yaml
│   ├── smoke_100step.yaml
│   ├── overfit64.yaml
│   ├── eorssd.yaml
│   ├── orssd.yaml
│   └── orsi4199.yaml
├── qwenalpha_sod/
│   ├── data/
│   │   ├── build_rgba_pairs.py
│   │   ├── audit_splits.py
│   │   ├── cache_conditions.py
│   │   └── cache_latents.py
│   ├── models/
│   │   ├── qwen_alpha_lora.py
│   │   ├── alpha_decode.py
│   │   └── losses.py
│   ├── tools/
│   │   ├── check_runtime.py
│   │   └── monitor_vram.py
│   ├── train.py
│   ├── infer.py
│   └── evaluate.py
├── scripts/
│   ├── run_single_gpu.sh
│   ├── stage0_env.sh
│   ├── stage1_vae_audit.sh
│   ├── stage2_cache.sh
│   ├── stage3_smoke.sh
│   ├── stage4_overfit64.sh
│   └── stage5_train_dataset.sh
├── tests/
├── state/
│   ├── progress.yaml
│   ├── history.jsonl
│   ├── active_run.txt
│   ├── done/
│   └── locks/
├── runs/
├── reports/
│   └── NEXT_ACTION.md
└── README.md
```

模型权重、数据集、`cache/`、`runs/` 和临时文件不得提交到公开仓库。

---

## 4. 阶段总览与依赖关系

| 阶段 | 内容 | 预计时长 | 必须通过的门槛 |
|---|---|---:|---|
| S0 | 远端、环境和单卡锁定 | 0.5～1 天 | 只见一张 GPU；依赖可导入 |
| S1 | 数据审计与 RGBA 标签构建 | 1～2 天 | 划分、尺寸、配对、hash 全正确 |
| S2 | Qwen 原生 RGBA VAE 上限审计 | 0.5～1 天 | MAE≤.005，Sα/Fmax≥.985 |
| S3 | 条件 embedding 与目标 latent 缓存 | 1～3 天 | 缓存可复算、无错配、单卡≤22 GB |
| S4 | 100-step 资源与梯度冒烟 | 0.5～1 天 | 单卡≤22 GB，loss 正常下降 |
| S5 | 64 张过拟合 | 1～2 天 | 训练 Sα≥.98，输入打乱后明显下降 |
| S6 | EORSSD 最小闭环 | 3～7 天 | Sα≥.92，MAE≤.01 |
| S7 | Alpha 质量增强与消融筛选 | 5～10 天 | 每项增益经 3 种子验证 |
| S8 | 三数据集独立正式训练 | 2～4 周 | 完成冻结配置的主结果 |
| S9 | 泛化、消融和失败分析 | 1～2 周 | 证据链完整、可复现 |

所有阶段严格串行执行。前一阶段未通过时，不下载或训练额外候选模型，也不进入下一阶段。

---

## 5. S0：远端环境与单卡锁定

### 5.1 操作

1. 用 VS Code Remote-SSH 连接 `wangsiyi`；
2. 在远端终端验证 `/data2/wangsiyi` 的真实路径和目录类型；
3. 在该目录下创建工程、模型、数据和缓存目录；
4. 创建独立 Python 环境，不修改系统 Python；
5. 安装 PyTorch、Transformers、Diffusers、Accelerate、DiffSynth-Studio；
6. 使用包含 segmented-attention 修复的 DiffSynth-Studio；
7. 训练和推理统一设置 `use_flex_attention=False`；
8. 冻结精确依赖版本到 `reports/environment.lock.txt`。

### 5.2 验收

- `torch.cuda.device_count()==1`；
- 识别到 RTX 3090 24 GB；
- Qwen-Image-2.1 能以 CPU offload 加载；
- 8 张普通图像编辑推理无崩溃；
- 峰值显存≤22 GB；
- 没有任何其他 GPU 进程由本项目启动。

失败处理：优先修复依赖和卸载策略；不得开启第二张卡。

---

## 6. S1：数据审计与 RGBA 目标构建

### 6.1 固定数据协议

| 数据集 | 训练 | 测试 |
|---|---:|---:|
| ORSSD | 600 | 200 |
| EORSSD | 1400 | 600 |
| ORSI-4199 | 2000 | 2199 |

主实验分别训练三个模型。联合训练只能作为附加实验，并且必须先做跨数据集 hash 与近重复去重。

### 6.2 RGBA 标签

```text
Alpha = GT / 255
RGB   = ORSI × Alpha
背景 RGB = 0
输出 = [R,G,B,A]
```

Alpha 保存原始二值 GT，不提前做羽化。几何增强对 ORSI、RGB 和 Alpha 同步；Alpha 使用 nearest 插值。颜色增强只作用于 ORSI/RGB，不作用于 Alpha。

### 6.3 输出

- `reports/dataset_audit.json`；
- 三数据集样本清单、尺寸分布、前景面积分布；
- 空目标、小目标、多目标、细长目标等难例标签；
- 训练/测试 SHA-256 与重复检查报告；
- 16 组 ORSI、GT、RGBA、Alpha 可视化拼图。

验收：不存在缺图、错配、尺寸不一致、标签非预期灰度或训练测试重复。

---

## 7. S2：原生 VAE 的 Alpha 上限审计

### 7.1 样本

从 EORSSD 训练集分层抽取 100 张，覆盖小目标、多目标、细边界、低对比、阴影和大目标。

### 7.2 流程

```text
目标 RGBA → Qwen VAE encode → Qwen VAE decode
                              ↓
                       提取重建 Alpha
```

分别在 352、384 分辨率测试；只有 352 通过才作为默认单卡配置，384 仅作为后续消融。

### 7.3 Go/No-go

必须同时满足：

- `Sα ≥ 0.985`；
- `Fmax ≥ 0.985`；
- `MAE ≤ 0.005`；
- 95% 样本 `Boundary-F ≥ 0.970`；
- 小目标子集不存在系统性消失。

未通过：停止 QwenAlpha 主线并提交失败报告；不得重训 Qwen VAE、不得启用第二张卡。是否转入 Show-o2 单卡回退由用户另行决定。

---

## 8. S3：离线缓存

### 8.1 条件缓存

冻结 Qwen3-VL，对每个已经固化的训练增强版本缓存：

- 文本指令 token；
- ORSI 条件 embedding；
- attention mask、位置与尺寸元数据；
- 源图 hash、增强参数和模型版本。

### 8.2 目标缓存

冻结 RGBA VAE encoder，缓存目标 RGBA latent。验证集和测试集不做随机增强，只保留唯一版本。

### 8.3 完整性检查

- 随机抽取 32 个缓存重新编码，误差在数值容忍范围内；
- sample ID、条件 embedding、目标 latent 与 GT 一一对应；
- 缓存键包含模型版本、分辨率和增强版本；
- 缓存缺失或版本不一致时拒绝训练，而不是静默重算。

缓存阶段与训练阶段不能并行占用同一张 GPU。

---

## 9. S4：100-step 单卡冒烟

### 9.1 固定配置

```yaml
resolution: 352
micro_batch: 1
gradient_accumulation: 16
precision: bf16
lora_rank: 8
lora_alpha: 16
gradient_checkpointing: true
train_modules: dit_attention_lora_only
condition_source: cached
target_latent_source: cached
use_flex_attention: false
max_vram_gb: 22
```

第一轮只使用：

```text
L = L_flow + L_wBCE(Alpha,GT) + L_wIoU(Alpha,GT)
```

如每步 VAE 解码超显存，则采用 3 个纯 `L_flow` step + 1 个 Alpha 辅助 step 的交替策略。

### 9.2 验收

- 峰值显存≤22 GB；
- 进程内始终只见一张 GPU；
- loss 有限、无 NaN/Inf、无持续发散；
- 梯度范数合理，LoRA 参数确实更新；
- 预测不是持续纯噪声、全黑或全白；
- 估算 EORSSD 单 epoch 不超过 12 小时。

OOM 降配顺序：减少 Alpha 解码频率 → LoRA 仅最后 8～12 层 → rank 8→4 → 384/352 保持 352 → 分层 CPU offload。仍失败则停止 Qwen 路线，不启用多卡。

---

## 10. S5：64 张过拟合门槛

### 10.1 数据

固定 EORSSD 的 48 张训练 + 16 张内部验证样本，覆盖不同目标面积和场景；禁止使用公开测试集调参。

### 10.2 训练

- 500～1000 step；
- LoRA rank 8；
- 固定 sampler、推理步数和 seed；
- 每 50 step 导出 16 张 Alpha 灰度图；
- 每 100 step 保存 checkpoint、显存和指标。

### 10.3 验收

- 训练集 `Sα ≥0.98`；
- 验证集指标明显高于未微调 Qwen；
- 无彩色内容渗入 Alpha；
- 无大面积灰雾、全黑、全白和边界整体漂移；
- 输入 ORSI 打乱后性能显著下降。

未通过时只允许排查数据错配、缓存、attention、条件注入和损失实现；不得直接扩大数据或训练更久掩盖错误。

---

## 11. S6：EORSSD 最小闭环

1. 只训练 EORSSD；
2. 从 1400 张训练集内部固定 10% 作为验证，不接触 600 张测试集；
3. 训练配置冻结后只在测试集评测一次；
4. 先完成单次生成，不启用二次 refine；
5. 报告未微调 Qwen、仅 `L_flow`、`L_flow+region` 三组结果。

进入后续阶段的最低门槛：

- EORSSD `Sα ≥0.920`；
- `MAE ≤0.010`；
- 小目标子集不完全失效；
- 单图推理可在单张 3090 完成；
- 结果可由固定 checkpoint、配置和 seed 复现。

若低于门槛，停止三数据集训练，先形成瓶颈报告。

---

## 12. S7：改善 Alpha 灰度图质量

严格按顺序一次只加入一个变量：

1. `F-measure loss`：改善区域整体与前景召回；
2. 边界损失：改善轮廓模糊；
3. 四方向梯度损失：改善飞机、船、道路等方向结构；
4. FFT 频域损失：约束大尺度形状和高频边缘；
5. 小目标/难例重采样；
6. 384 分辨率消融；
7. 传统 SOD 教师软目标；
8. 可选第二次 RGBA 编辑精修。

每个变量至少运行 3 个训练随机种子。平均 `Sα` 增益小于 0.2% 或 MAE恶化时，不进入最终配置。第二次精修只有平均提升≥0.3% Sα 才保留。

---

## 13. S8：三数据集正式训练

冻结唯一配置后，按照以下顺序串行运行：

1. EORSSD；
2. ORSSD；
3. ORSI-4199。

每个数据集独立训练、独立 checkpoint、3 个随机种子；同一时刻只运行一个任务。每次训练结束并完成评测后，才启动下一个数据集。

每个数据集输出：

- 最佳验证 checkpoint；
- 三种子均值±标准差；
- `Sα / Fmax / Emax / MAE`；
- 可训练参数、峰值显存、训练 GPU 小时、单图延迟；
- 全部预测灰度 PNG；
- 最佳、典型和最差案例。

---

## 14. S9：消融、泛化和论文证据

必做消融：

- 普通黑白 RGB 目标 vs RGBA/Alpha 目标；
- 冻结基座 vs DiT-LoRA；
- `L_flow` vs `+region` vs `+F` vs `+edge` vs `+FFT`；
- 无蒸馏 vs 单教师；
- 352 vs 384；
- 单次生成 vs 二次精修；
- 固定 seed vs 多 seed；
- QwenAlpha vs Show-o2 单卡回退（仅当 Qwen 触发回退且用户决定继续）。

泛化与可信性：

- ORSI-4199→EORSSD、ORSSD→EORSSD；
- 输入打乱、纯黑输入和错误图像条件；
- 小目标、低对比、阴影、多目标、空场景、细长和孔洞结构分类统计；
- 最差 50 张定量与定性分析；
- 不使用 CRF、形态学操作或人工修图包装结果。

---

## 15. Checkpoint、日志与恢复

每个阶段保存：

```text
runs/<stage>/<dataset>/<timestamp>/
├── config.resolved.yaml
├── environment.lock.txt
├── git_revision.txt
├── gpu.json
├── train.log
├── metrics.json
├── checkpoints/
└── predictions/
```

恢复训练必须同时校验：模型基座 hash、LoRA 配置、缓存版本、数据清单 hash、优化器状态和随机数状态。任何一项不一致时新建实验，不覆盖旧结果。

磁盘清理只删除已确认可由源数据重建的缓存；checkpoint、指标和配置先归档。不得在脚本中加入自动递归清理。

---

## 16. 最终 Go/No-go 决策

| 检查点 | Go | No-go |
|---|---|---|
| VAE Alpha 重建 | MAE≤.005，Sα/Fmax≥.985 | 停止 Qwen 路线 |
| 100-step 冒烟 | ≤22 GB，loss 稳定 | 降配一次；仍失败则停止 |
| 64 张过拟合 | 训练 Sα≥.98 | 排查实现；仍失败则停止 |
| EORSSD 闭环 | Sα≥.92，MAE≤.01 | 不进入三数据集 |
| 竞争性结果 | 接近或超过 MCCNet/GeleNet | 重新定位为方法边界研究 |

任何 No-go 都不得通过第二张 GPU 绕过。

---

## 17. 推荐执行顺序

```text
S0 单卡环境
  ↓
S1 数据与RGBA标签
  ↓
S2 VAE Alpha审计 ──失败→停止
  ↓
S3 离线缓存
  ↓
S4 100-step冒烟 ──失败→单卡降配一次→仍失败停止
  ↓
S5 64张过拟合 ──失败→修实现→仍失败停止
  ↓
S6 EORSSD闭环 ──失败→不跑三集
  ↓
S7 逐项增强
  ↓
S8 三数据集串行训练
  ↓
S9 消融、泛化、失败分析与论文材料
```

该顺序的核心原则是：先证明表示上限，再证明单卡可训练，再证明模型可学习，最后才消耗时间完成三数据集。整个过程中不启用第二张 GPU。

---

## B. 可逐项打勾的详细运行手册

以下步骤是实际执行顺序。每个步骤必须更新 `progress.yaml`；只有通过验收并产生 `.done.json` 后才能勾选完成。

### B0：开始前的统一准备

#### S0.1 连接远端并确认边界（✅ 已完成：2026-10-26 19:19 +08:00）

- ✅ VS Code Remote-SSH 已连接 `wangsiyi`；
- ✅ 远端 `hostname` 与预期一致：`ubuntu-22`；
- ✅ `/data2/wangsiyi` 存在且不是符号链接；
- ✅ `realpath -e /data2/wangsiyi` 精确返回 `/data2/wangsiyi`；
- ✅ 远端身份为 `wangsiyi`，全程未使用 `sudo`；
- ✅ 结果已写入本地核验报告 `paper/S0.1_远端连接与路径边界核验报告.md`。

执行结论：**✅ Passed**。本步骤只执行只读检查，没有创建或修改任何远端文件。  
通过条件：五项全部成立。  
安全暂停点：检查报告保存后。  
恢复入口：重新执行只读检查；结果一致即可跳过。

#### S0.2 创建工程骨架（✅ 已完成：2026-10-26）

- ✅ 创建工程、模型、数据、缓存目录；
- ✅ 创建 `state/`、`state/done/`、`state/locks/`、`runs/`、`reports/`；
- ✅ 写入初始 `progress.yaml`、`history.jsonl` 和 `active_run.txt`；
- ✅ 在 `/data2/wangsiyi/qwenalpha-sod` 初始化 Git 工作树；
- ✅ 创建 `.gitignore`，排除模型、数据、缓存、运行目录和秘密文件；
- ✅ 写入 `reports/NEXT_ACTION.md` 和 `reports/S0_directory_manifest.json`；
- ✅ 创建并验证 `state/done/S0.2.done.json`；
- ✅ 本地核验记录见 `paper/S0.2_远端工程骨架创建报告.md`。

执行结论：**✅ Passed**。所有解析后的目录均位于 `/data2/wangsiyi`，没有意外符号链接，未使用 `sudo`。  
安全暂停点：目录清单和初始状态文件完成后。  
输出：`reports/S0_directory_manifest.json`。

#### S0.3 建立 Python 环境（✅ 已完成：2026-10-27）

- [x] 记录 GPU 驱动和 CUDA 信息；
- [x] 创建项目独立环境；
- [x] 安装 PyTorch 后先验证单卡；
- [x] 安装 Qwen/Diffusers/DiffSynth 相关依赖；
- [x] 确认 segmented-attention 修复存在；
- [x] 写 `environment.lock.txt` 和 `pip_freeze.txt`。

执行结论：**✅ Passed**。Python 3.11 独立环境位于 `/data2/wangsiyi/envs/qwenalpha-sod`；在新 SSH 进程中固定 `CUDA_VISIBLE_DEVICES=1` 后，PyTorch 仅识别一张 RTX 3090，核心库与 `QwenImage21Pipeline` 均可导入。DiffSynth-Studio 固定到提交 `e5a44c42776978457b61d51d595a1f6746c3d87a`，已核验分段 SDPA 路径存在且 `use_flex_attention=False`。本地核验记录见 `paper/QwenAlpha-SOD_S0.3_Python环境建立报告.md`。  
通过条件：核心库全部可导入，环境重开后仍可导入。  
失败处理：首次安装因系统 `/tmp` 空间不足失败；未改变依赖版本，仅将 `TMPDIR` 切换到 `/data2/wangsiyi/tmp/pip` 后成功，详见远端 `reports/S0_dependency_failures.md`。  
安全暂停点：环境锁、冻结清单和 `state/done/S0.3.done.json` 已写入；下一步为 S0.4。

#### S0.4 实现单卡运行包装器（✅ 已完成：2026-10-27）

`scripts/run_single_gpu.sh` 必须：

- [x] 只接受一个物理 GPU 编号；
- [x] 拒绝包含逗号的 GPU 参数；
- [x] 设置 `CUDA_VISIBLE_DEVICES`；
- [x] 启动前检查指定卡已有进程；
- [x] 运行 Python 单卡断言；
- [x] 创建项目 GPU lock；
- [x] 捕获退出码并写运行摘要；
- [x] 无论成功或失败都释放项目 lock。

测试用例：无 GPU 参数、两个 GPU 参数、忙卡、正常空闲卡、Python 异常退出。  
执行结论：**✅ Passed（9/9）**。上述五类必测场景全部通过，并额外验证逗号双卡参数、项目锁竞争、异常退出后锁释放和运行摘要完整性。正例固定物理 GPU 1 后仅看到一个逻辑设备 `cuda:0`；忙卡返回码 75，锁冲突返回码 73，Python 异常退出码 7 被原样传递；测试结束后没有遗留 GPU 进程。远端完成标记为 `state/done/S0.4.done.json`，本地报告见 `paper/QwenAlpha-SOD_S0.4_单卡运行包装器报告.md`。  
通过条件：所有负例均拒绝启动，正例只看到一个 `cuda:0`。  
安全暂停点：包装器、测试结果、哈希和恢复状态均已固化；下一步为 S0.5。

#### S0.5 建立暂停恢复单元测试（✅ 已完成：2026-10-27）

- [x] 训练 10 step；
- [x] 第 6 step 创建 `PAUSE_REQUESTED`；
- [x] 确认 step 6 后安全退出；
- [x] `--resume-dry-run` 显示下一步为 step 7；
- [x] 恢复到 step 10；
- [x] 连续训练和暂停恢复训练的最终参数差异处于容忍范围内。

执行结论：**✅ Passed**。连续 10-step 运行与 step 6 暂停、step 7 恢复的运行均到达 step 10；dry-run 明确输出 `next_step=7` 且未修改 checkpoint。模型、优化器、调度器、Python/NumPy/PyTorch CPU/PyTorch CUDA RNG、采样顺序、数据游标与 loss 历史全部一致，19 个张量的最大绝对差为 `0.0`，低于 `1e-7` 容忍阈值。测试结束后 GPU 1 无遗留进程且项目锁可重新获取。本地报告见 `paper/QwenAlpha-SOD_S0.5_暂停恢复单元测试报告.md`。  
通过条件：模型、优化器、调度器、RNG 和数据游标均成功恢复。  
完成标记：`state/done/S0.5.done.json`。  
安全暂停点：S0 阶段全部完成；下一步为 S1.1 数据集只读盘点。

### B1：数据准备

#### S1.1 数据集只读盘点（✅ 已完成：2026-10-27）

- [x] 定位 ORSSD、EORSSD、ORSI-4199；
- [x] 不改变原始数据；
- [x] 统计训练/测试数量；
- [x] 检查图片和 GT 是否可解码；
- [x] 检查同名配对和尺寸；
- [x] 记录非二值 GT、空 GT 和异常面积样本。

预期数量：ORSSD 600/200、EORSSD 1400/600、ORSI-4199 2000/2199。  
执行结论：**✅ Passed**。三套数据共6999对，训练/测试数量与预期完全一致；全部图像和GT可解码，同名配对完整，图像/GT尺寸无不一致。盘点记录575张非二值GT、96张空GT和255张面积阈值异常样本；这些样本均已逐项写入JSON，不在S1.1擅自修正。ORSSD与ORSI-4199训练目录包含8倍离线增强，正式计数只选择无增强后缀原图。源目录盘点前后快照一致。远端目前仅有2999对测试副本，完整训练源仍在本地；进入S1.2时须先建立位于`/data2/wangsiyi/datasets`的只读镜像或等价挂载。本地报告见 `paper/QwenAlpha-SOD_S1.1_数据集只读盘点报告.md`。  
输出：`reports/S1_inventory_<dataset>.json`。  
安全暂停点：三套JSON、汇总JSON与 `state/done/S1.1.done.json` 已写入；下一步为S1.2。

#### S1.2 固化 manifest 和 hash（✅ 已完成：2026-10-27）

- [x] 为每张输入和 GT 计算 SHA-256；
- [x] 写训练、内部验证、公开测试 manifest；
- [x] 检查 ORSSD/EORSSD 跨集相同和近重复；
- [x] manifest 按 sample ID 排序；
- [x] 生成 manifest 总 hash。

中断恢复：按数据集和 shard 写 hash；已完成 shard 通过校验后跳过。  
通过条件：同一 manifest 重跑 hash 完全一致。

执行结论：**✅ Passed**。6,999 对样本全部生成稳定 sample ID（`<dataset>/<split>/<stem>`）与 image/GT 双 SHA-256，30 个分片均可续跑；`all_samples.csv` 按 sample ID 排序，总哈希 `ba11c6c3…1446a0`，两遍全量重算结果完全一致。重复审计发现 2 组训练/测试完全同图（`EORSSD/test/0005 ≡ train/0003`，ORSSD 同位置同图）及 158 组 EORSSD↔ORSSD 同源图像；经用户确认按「保留并声明」处置，未删除或改写任何样本，论文中须披露。  
输出：`manifests/` 七份 CSV、`manifests/S1.2_manifest_evidence.json`、`reports/S1.2_leakage_report.md`。  
安全暂停点：`state/done/S1.2.done.json`；下一步为 S1.3。  

#### S1.3 构建 RGBA 标签（✅ 已完成：2026-10-27）

- [x] 读取 ORSI 和 GT；
- [x] 将 GT 规范为 `[0,1]` Alpha；
- [x] 计算 `RGB=ORSI×Alpha`；
- [x] 背景 RGB 清零；
- [x] 保存 RGBA PNG；
- [x] 重新读取 PNG，验证 Alpha 与 GT 像素一致；
- [x] 写样本级构建 manifest。

分片建议：每 128 张一个 shard。  
暂停点：任一 shard 完成、验证并写 `.done.json` 后。  
恢复方式：只重做缺失或 hash 错误的 shard。

执行结论：**✅ Passed**。4,000 个训练样本全部生成 RGBA（EORSSD 1400 / ORSSD 600 / ORSI-4199 2000），32 个分片、0 错误、输出 130.2 MB。写盘后重开文件校验：alpha 与原 GT 逐像素最大差为 0，alpha=0 处 RGB 非零像素为 0；独立复核重新哈希 4,000 个文件无差异，12 张分层抽样（含软掩码与空掩码）重算像素完全一致。89 个非二值 GT 保留软灰度、74 个空 GT 保留，均未阈值化或剔除。  
输出：`datasets/derived/rgba/`、`manifests/s1_3_rgba_manifest.csv`、`manifests/S1.3_evidence.json`。  
安全暂停点：`state/done/S1.3.done.json`；下一步为 S1.4。  

#### S1.4 难例标签与 100/64 子集

- [ ] 计算目标面积比例、目标数量、周长、细长度和边界复杂度；
- [ ] 标注小目标、多目标、细长、孔洞、低对比、阴影、空场景；
- [ ] 分层选择 100 张 VAE 审计集；
- [ ] 固定 48+16 的过拟合集；
- [ ] 保存 sample ID，不复制成无法追踪的新数据。

通过条件：子集覆盖所有指定难例类别且不含公开测试样本。

### B2：模型部署与表示上限

#### S2.1 下载与校验 Qwen-Image-2.1

- [ ] 权重仅保存到 `/data2/wangsiyi/models/`；
- [ ] 支持断点续传；
- [ ] 下载完成后记录文件大小和 hash；
- [ ] 验证模型配置、DiT、Qwen3-VL 和 VAE 文件齐全；
- [ ] 不把权重加入公开仓库。

安全暂停点：每个权重分片完成后。  
恢复：使用原下载工具续传，不删除已验证分片。

#### S2.2 8 张 CPU-offload 推理

- [ ] 单卡包装器启动；
- [ ] 禁止 `device_map="auto"`；
- [ ] Qwen3-VL、DiT、VAE 顺序进入同一张 GPU；
- [ ] 生成 8 张 RGBA；
- [ ] 保存 Alpha 灰度图、耗时和显存；
- [ ] 确认 Alpha 不是恒定值或空通道。

通过条件：8 张全部完成，峰值≤22 GB，仅一张 GPU 可见。  
失败时保留完整错误日志，不继续 VAE 审计。

#### S2.3 100 张 VAE 审计

执行 352 分辨率；只有结果接近门槛时才补 384 对照。

- [ ] 每张 RGBA encode/decode；
- [ ] 提取重建 Alpha；
- [ ] 计算 Sα、Fmax、MAE、Boundary-F；
- [ ] 保存逐图 CSV 和总体统计；
- [ ] 单独统计小目标子集；
- [ ] 输出最差 20 张拼图。

中断恢复：逐图输出使用临时文件后原子改名；恢复时跳过 hash 匹配的已完成样本。  
Go：满足第 7.3 节全部红线。  
No-go：写 `reports/S2_VAE_NOGO.md` 并停止后续训练。

### B3：缓存阶段

#### S3.1 固化增强表

- [ ] 所有随机增强先生成参数表；
- [ ] 同一 sample ID 的 ORSI/RGBA/Alpha 使用相同几何参数；
- [ ] Alpha 使用 nearest；
- [ ] 每条增强拥有唯一 `augmentation_id`；
- [ ] 写增强表 hash。

暂停后不得重新随机生成已有增强参数。

#### S3.2 缓存条件 embedding

建议每 32～64 个样本一个 shard：

- [ ] 加载冻结 Qwen3-VL；
- [ ] 生成条件 embedding 和 mask/位置元数据；
- [ ] 保存为临时 shard；
- [ ] 重新加载并检查 shape、dtype、finite；
- [ ] 原子改名并写 shard `.done.json`；
- [ ] 更新 cache manifest。

安全暂停点：每个 shard。  
恢复：读取 manifest，只运行 `pending/failed` shard。  
验收：随机 32 条在线重算与缓存一致。

#### S3.3 缓存目标 RGBA latent

流程与 S3.2 相同，但使用冻结 VAE encoder。额外记录 latent scale、VAE 版本和分辨率。

#### S3.4 缓存总审计

- [ ] 数量与训练 manifest 一致；
- [ ] 无 NaN/Inf；
- [ ] sample ID 无重复/缺失；
- [ ] embedding/latent/GT 三者 hash 对齐；
- [ ] 随机解码 32 个 latent，Alpha 指标与 S2 一致。

通过后将缓存 manifest 设为只读版本，训练期间不得静默修改。

### B4：训练代码与 100-step 冒烟

#### S4.1 损失单元测试

- [ ] 完美 Alpha 的各损失接近理论最小；
- [ ] 全黑、全白、边界偏移的损失方向正确；
- [ ] wBCE、wIoU、F、edge、direction、FFT 无 NaN；
- [ ] 预测 latent 到 `z0_hat` 的公式通过数值测试；
- [ ] 冻结 VAE decoder 能把 Alpha 梯度传回预测 latent；
- [ ] Qwen3-VL/VAE 参数无梯度，LoRA 参数有梯度。

#### S4.2 1-step 前后向

- [ ] batch=1；
- [ ] 完成 forward/backward/optimizer step；
- [ ] 输出参数更新统计；
- [ ] 记录显存分解；
- [ ] 保存并重新加载 checkpoint。

#### S4.3 10-step 稳定性

- [ ] loss 有限；
- [ ] 梯度范数无持续爆炸；
- [ ] 预测 Alpha 可解码；
- [ ] 暂停恢复测试再次通过。

#### S4.4 100-step 正式冒烟

- [ ] 使用固定 `smoke_100step.yaml`；
- [ ] 每 10 step 写指标；
- [ ] 每 20 step checkpoint；
- [ ] 每 25 step 导出 Alpha；
- [ ] 记录峰值≤22 GB；
- [ ] 估算 EORSSD epoch 时间≤12小时。

Go/No-go 按第 9.2 节。暂停后从最近 20-step checkpoint 恢复，禁止从头重跑掩盖恢复问题。

### B5：64 张过拟合

#### S5.1 冻结配置和基线

- [ ] 保存未微调 Qwen 对 16 张验证样本的输出；
- [ ] 固定训练/验证 ID；
- [ ] 固定 sampler、steps、seed；
- [ ] 创建独立 run 目录和配置 hash。

#### S5.2 训练 0～500 step

- [ ] 每 50 step checkpoint/预测；
- [ ] 观察训练 Sα、MAE；
- [ ] 观察全黑/全白比例和平均前景面积；
- [ ] 第 500 step 做第一次 Go/No-go。

#### S5.3 必要时训练 500～1000 step

只有 loss 正常且指标仍在提升时继续；若已停滞，先诊断而不是无条件延长。

#### S5.4 反事实检查

- [ ] 正确 ORSI；
- [ ] 打乱 ORSI；
- [ ] 纯黑 ORSI；
- [ ] 错误图像条件；
- [ ] 比较输出变化和指标下降。

通过条件：训练 Sα≥.98，且反事实性能显著下降。暂停恢复以 50-step checkpoint 为边界。

### B6：EORSSD 最小闭环

#### S6.1 内部划分

- [ ] 从 1400 张训练集固定 90%/10%；
- [ ] stratify 难例和面积分布；
- [ ] 公开 600 张测试集只在最终冻结后使用；
- [ ] 保存 split hash。

#### S6.2 三个基础配置

串行运行并分别保存：

1. 未微调 Qwen；
2. DiT-LoRA + `L_flow`；
3. DiT-LoRA + `L_flow + L_region`。

每个配置训练完成后先验证，再决定是否启动下一个。不得并行占卡。

#### S6.3 冻结与测试

- [ ] 按内部验证选 checkpoint；
- [ ] 锁定配置和 seed；
- [ ] 在 600 张测试集一次性推理；
- [ ] 输出 L 模式灰度 PNG；
- [ ] 计算统一指标；
- [ ] 检查输出数量、尺寸和像素范围。

Go：Sα≥.92、MAE≤.01。  
Pause：每个 epoch 和每 100 张测试推理均为安全点。  
Resume：训练加载 epoch checkpoint；推理跳过 hash 正确的已完成 PNG。

### B7：质量增强实验

每项遵循同一模板：

1. 从当前最佳配置复制新 config；
2. 只改变一个变量；
3. 使用相同内部划分；
4. 运行 3 个种子；
5. 写均值、标准差和相对增益；
6. 增益不足则标记 rejected，不叠加到后续配置。

顺序固定：F-loss → edge → direction → FFT → 难例采样 → 384分辨率 → 教师蒸馏 → 二次精修。

每个“变量×种子”是独立可恢复任务；一个任务完成后才能启动下一个。`experiment_matrix.csv` 记录 `pending/running/paused/passed/rejected/failed`。

### B8：三数据集正式训练

#### S8.1 冻结最终配方

- [ ] 所有超参数来自内部验证；
- [ ] 禁止针对公开测试集继续调权；
- [ ] 配置、代码、环境、缓存全部 hash；
- [ ] 创建 `FINAL_RECIPE.lock`。

#### S8.2 EORSSD 三种子

按 seed 0→1→2 串行。每个 seed 完成训练、验证、测试和结果校验后再运行下一个。

#### S8.3 ORSSD 三种子

独立模型、独立内部验证、相同最终配方；不得使用 EORSSD 测试数据训练。

#### S8.4 ORSI-4199 三种子

同上。数据较大时提高 checkpoint 频率，保证计划暂停最多损失 30～60 分钟工作。

#### S8.5 汇总

- [ ] 三种子均值±标准差；
- [ ] 参数量和可训练参数；
- [ ] 峰值显存；
- [ ] GPU 小时和单图延迟；
- [ ] 输出完整性；
- [ ] 与 MCCNet、GeleNet、IPDiff 指标对照。

### B9：消融、泛化和收尾

#### S9.1 必做消融队列

按第 14 节逐个执行，维护 `experiment_matrix.csv`。不得为了赶进度并行运行多个 GPU 任务。

#### S9.2 跨数据集

- [ ] ORSI-4199→EORSSD；
- [ ] ORSSD→EORSSD；
- [ ] 保存源域/目标域配置和 checkpoint 关联。

#### S9.3 失败分析

- [ ] 最差 50 张；
- [ ] 小目标、低对比、阴影、多目标、空场景分类；
- [ ] VAE 上限与最终误差的相关性；
- [ ] 生成随机性；
- [ ] 典型全黑、全白、边界漂移和孔洞案例。

#### S9.4 最终可复现包

- [ ] README 从空环境到单卡推理；
- [ ] 环境锁、配置和 manifest；
- [ ] 不含模型权重和数据的代码包；
- [ ] 一条单卡推理命令；
- [ ] 一条统一评测命令；
- [ ] 论文表格来源可追踪到原始 `metrics.json`。

---

## C. 暂停或恢复时的最短操作清单

### 需要暂停时

```text
1. 不启动新阶段
2. 对当前 run 创建 PAUSE_REQUESTED
3. 等待程序保存并验证 checkpoint 后退出
4. 检查 GPU 已释放
5. 更新 progress.yaml、history.jsonl、NEXT_ACTION.md
6. 记录唯一恢复命令
```

### 需要恢复时

```text
1. 连接 VS Code Remote-SSH: wangsiyi
2. 读取 NEXT_ACTION.md 和 progress.yaml
3. 检查 tmux 是否已有任务
4. 检查指定 GPU 是否空闲
5. 验证单卡、路径、checkpoint 和 hash
6. 运行 --resume-dry-run
7. 执行已记录的唯一恢复命令
8. 确认 step 连续且没有从头重跑
```

### Codex 新会话接手时

Codex只需要依次读取：

1. 本计划书；
2. `reports/NEXT_ACTION.md`；
3. `state/progress.yaml`；
4. 当前 run 的 `config.resolved.yaml`；
5. 当前日志末尾 200 行；
6. 最近 checkpoint 的完整性检查结果。

完成上述读取后再继续工作，不依赖此前聊天上下文，也不重新执行已有 `.done.json` 的步骤。
