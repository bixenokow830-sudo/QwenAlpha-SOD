# QwenAlpha-SOD：Codex 额度不足时 Harness 续执行计划书

版本：v1.0  
编制日期：2026-10-27  
项目：QwenAlpha-SOD  
本地工作区：`D:\OneDrive\Desktop\my paper`  
远端主机别名：`wangsiyi`  
远端工程：`/data2/wangsiyi/qwenalpha-sod`  
唯一公开同步仓库：`https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git`

---

## 0. 文档用途与授权边界

本计划书用于 Codex 因额度不足、会话中断或需要长期暂停时，将 QwenAlpha-SOD 的后续工作交给另一执行者继续。它提供足以脱离聊天记录独立恢复的状态、顺序、验收、暂停、失败和同步规则。

### 0.1 当前 Harness 禁用策略仍然有效

本工作区当前 `AGENTS.md` 明确禁止调用 DeepSeek Harness、DSH、Harness 技能、启动脚本及 Harness 驱动的 SSH 工作流。因此：

- 本文的创建不等于授权 Harness 执行；
- Codex 不得因为本文存在而调用 Harness；
- Harness 不得把本文视为自动授权；
- 将来确需 Harness 接管时，用户必须先明确要求修改全局禁用策略，并明确授权其按本计划书执行；
- 在授权生效前，只允许人工或 Codex 阅读本文，不允许启动 Harness。

推荐的未来授权应明确说明允许范围、远端主机、单卡限制、起始步骤和停止条件。不要使用含糊的“继续做”替代策略变更。

### 0.2 本文与其他资料的优先级

执行优先级从高到低为：

1. 用户当前明确指令；
2. 当前有效的 `AGENTS.md`；
3. 本计划书；
4. `paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md`；
5. 远端 `reports/NEXT_ACTION.md`；
6. 远端 `state/progress.yaml`、`state/history.jsonl` 和 `.done.json`；
7. 历史聊天、临时笔记和未验证日志。

若资料冲突，立即停止，不得自行选择更激进的解释。先保留现场，再向用户报告冲突位置、两种解释及可能影响。

---

## 1. 项目目标与最终产物

项目以 Qwen-Image-2.1 为主模型，将其原生 RGBA 生成能力重构为 Alpha 显著性预测：输入光学遥感图像，输出像素值范围为 0～255 的单通道灰度显著图。最终交付给 SOD 评测程序的产物仍是黑底、显著目标趋近白色的灰度 PNG，而不是彩色效果图。

预期成果包括：

- 可在单张 RTX 3090 上部署、训练、恢复和推理的 QwenAlpha-SOD；
- EORSSD、ORSSD、ORSI-4199 三个数据集的灰度预测图；
- `Sα / Fmax / Emax / MAE` 等统一指标；
- 与 MCCNet、GeleNet、IPDiff 等传统或专用 SOD 方法的对比；
- RGBA/Alpha 目标、区域损失、边界质量和难例策略的消融；
- 环境锁、配置、数据 manifest、日志、checkpoint 关联和可复现说明；
- 不含数据集、模型权重、运行目录和秘密文件的公开代码与论文材料。

该路线使用 GT 显著图训练，属于全监督 SOD。不得在论文或报告中把它描述为零样本、无监督或仅依靠文本提示的直接生成。

---

## 2. 永久约束：任一接管者都不得违反

### 2.1 全程仅使用一张 RTX 3090

- 课题组虽有两张 RTX 3090，本项目任一时刻只能调用其中一张；
- 运行前由课题组或当前用户确定空闲物理卡；
- 所有 GPU 任务必须通过 `/data2/wangsiyi/qwenalpha-sod/scripts/run_single_gpu.sh`；
- 包装器参数只能是一个物理 GPU 编号，禁止逗号、多编号和自动选两卡；
- 进程内部只能看到一个逻辑设备 `cuda:0`；
- 禁止 `device_map="auto"`、DDP、多卡 Accelerate、FSDP、张量并行和流水线并行；
- 禁止用另一张 GPU 同时验证、推理、做消融或下载后试跑；
- 不得抢占、暂停、终止或修改其他成员的 GPU 进程；
- 指定卡忙时状态应设为 `blocked`，等待用户重新分配，不得自动轮询。

所有 GPU 任务的硬门槛：峰值显存不超过 22 GB。OOM 时只允许按既定单卡降配顺序处理，不得启用第二张卡。

### 2.2 远端目录和权限边界

允许写入的远端根目录仅为：

```text
/data2/wangsiyi/qwenalpha-sod/   # 代码、配置、状态、日志、结果
/data2/wangsiyi/models/          # 模型权重
/data2/wangsiyi/datasets/        # 数据只读镜像或项目数据目录
/data2/wangsiyi/cache/           # 缓存
/data2/wangsiyi/tmp/             # 临时文件
/data2/wangsiyi/envs/            # 项目环境
```

执行前必须验证：

```bash
test "$(realpath -e /data2/wangsiyi)" = "/data2/wangsiyi"
test ! -L /data2/wangsiyi
```

禁止：

- 使用 `sudo`；
- 向系统 Python 或系统根分区安装依赖；
- 对未经解析确认的路径执行递归删除、移动或覆盖；
- 自动删除 checkpoint、源数据、模型权重或其他成员文件；
- 在源数据目录内重命名、重编码、修正或生成派生文件。

系统根分区空间紧张。安装和下载前固定：

```bash
export TMPDIR=/data2/wangsiyi/tmp/pip
export PIP_CACHE_DIR=/data2/wangsiyi/cache/pip
export HF_HOME=/data2/wangsiyi/cache/huggingface
export TORCH_HOME=/data2/wangsiyi/cache/torch
```

### 2.3 GitHub 和公开同步边界

- 唯一同步目标是 `https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git`；
- 禁止使用旧 DS-SOD 仓库或脚本；
- 禁止公开数据集、第三方论文、模型权重、`runs/`、`tmp/`、缓存、密钥和凭证；
- 若本轮修改了本地 `paper/` 或 `output/pdf/`，完成验证后只运行一次：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "D:\OneDrive\Desktop\my paper\sync-qwenalpha-artifacts-once.ps1"
```

- 不创建计划任务、轮询、watcher 或后台持续同步；
- 同步失败时保留本地文件，记录错误，不得改投其他仓库。

### 2.4 不得绕过科学门槛

- 公开测试集不得参与超参数选择；
- No-go 不得通过增加 GPU、偷偷改测试集或手工修图绕过；
- 不使用 CRF、形态学处理或人工修图包装指标，除非作为明确标注的独立消融；
- 每次只改变一个实验变量；
- 指标不佳时必须保留失败结果和诊断，不得覆盖旧 run；
- 所有结论须追溯到固定配置、代码版本、输入 hash 和原始 `metrics.json`。

---

## 3. 已完成状态快照

快照日期：2026-10-27。接管时仍须以远端状态文件复核，不能只依赖此段。

| 步骤 | 状态 | 核心结果 |
|---|---|---|
| S0.1 | Passed | SSH 主机、用户和 `/data2/wangsiyi` 边界通过 |
| S0.2 | Passed | 远端骨架、Git、状态文件和忽略规则已建立 |
| S0.3 | Passed | Python 3.11.18 项目环境及模型依赖可导入 |
| S0.4 | Passed | 单卡包装器 9/9 测试通过 |
| S0.5 | Passed | step 6 暂停、step 7 恢复，最终参数最大差异 0.0 |
| S1.1 | Passed | 三数据集 6999 对只读盘点完成 |

### 3.1 已锁定环境

```text
环境目录：/data2/wangsiyi/envs/qwenalpha-sod
Python：3.11.18
PyTorch：2.10.0+cu128
torchvision：0.25.0+cu128
transformers：5.17.0
diffusers：0.41.0
accelerate：1.11.0
peft：0.21.2
huggingface_hub：1.33.0
DiffSynth-Studio：2.1.12 editable
DiffSynth commit：e5a44c42776978457b61d51d595a1f6746c3d87a
```

已确认 segmented SDPA 路径存在，`use_flex_attention=False`。不得在无必要时升级依赖；如确需改变版本，必须新建环境锁和兼容性报告，不能覆盖当前环境记录。

### 3.2 S1.1 数据盘点结论

| 数据集 | 训练 | 测试 | 非二值 GT | 空 GT | 面积异常 |
|---|---:|---:|---:|---:|---:|
| EORSSD | 1400 | 600 | 5 | 70 | 118 |
| ORSSD | 600 | 200 | 0 | 26 | 26 |
| ORSI-4199 | 2000 | 2199 | 570 | 0 | 111 |
| 合计 | 4000 | 2999 | 575 | 96 | 255 |

全部 6999 对图像和 GT 均可解码，无缺失配对、无尺寸不一致。ORSSD 与 ORSI-4199 的训练目录含 8 倍离线增强；正式样本只选择无增强后缀的原始训练图。

远端当前只有 2999 对测试副本，完整训练源仍在本地。`/data2/wangsiyi/datasets` 尚未建立完整只读镜像。此项是 S1.2 的首要阻塞条件，禁止假装训练集已在远端。

远端 S1.1 JSON SHA-256：

```text
EORSSD：fe1edd2cf6ee0380f3ace0e31ec9d8e74a048b837e9293f7eae1be3a26342880
ORSSD：99a4c0d2377ea152067881b1dcea0d3fa2fdb3c4e3a53abc990542359d9228bf
ORSI-4199：b512d8a6b1300c6766573cff627fb68b346484c2737df7ef90d5c917c05a329a
汇总：05ed2214143efaf7e388e852bf6b839a321f7c5abf28a8325de40024f3f08f0e
盘点脚本：c7034680f36b00bada7eac392826651040bb15e4d694465ed0e2182f268ee5b2
```

本地报告：`paper/QwenAlpha-SOD_S1.1_数据集只读盘点报告.md`。  
当前唯一下一阶段：S1.2 固化 manifest 和 hash。

---

## 4. 接管后的冷启动读取顺序

不得凭聊天摘要直接执行。接管者按以下顺序读取并交叉校验：

1. 本地 `AGENTS.md`；
2. 本计划书；
3. `paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md`；
4. 最新已完成阶段的本地报告；
5. 远端 `reports/NEXT_ACTION.md`；
6. 远端 `state/progress.yaml`；
7. 远端 `state/active_run.txt`；
8. 远端 `state/history.jsonl` 最后 30 条；
9. 远端 `state/done/` 中最近的 `.done.json`；
10. 若存在 active run，读取其 `config.resolved.yaml`、日志末尾 200 行和最近 checkpoint 校验结果。

读取后必须先形成一份“接管核验摘要”，至少列出：

```text
已完成的最后步骤：
当前状态：pending/running/paused/passed/failed/blocked
当前活动 run：
最近安全 checkpoint：
下一唯一动作：
需要使用 GPU：是/否
预期输出：
验收门槛：
发现的冲突：
```

若 `progress.yaml` 显示 `running`，先查 `tmux`、PID、日志更新时间和 GPU 进程。不能因为当前终端没有输出就重复启动。

---

## 5. 每个原子步骤的统一执行协议

每次只执行一个编号步骤，例如只执行 S1.2，不得顺带启动 S1.3。每步均遵守以下九段协议。

### 5.1 Preflight

- 验证真实路径仍位于 `/data2/wangsiyi`；
- 查看磁盘空间和 inode；
- 读取进度、历史、active run 和完成标记；
- 校验当前步骤输入文件、配置和代码 hash；
- 查看 `tmux ls`，排除重复任务；
- 仅在需要 GPU 时运行一次 `nvidia-smi`；
- GPU 忙则停止，不循环查询；
- 确认本步不会修改原始数据或公开禁区。

### 5.2 将状态设为 running

先写 `progress.yaml.tmp`，验证 YAML 可解析后原子改名；向 `history.jsonl` 追加一条事件。必须记录：步骤、时间、输入 hash、代码版本、预期输出和唯一恢复策略。

### 5.3 执行

- CPU 步骤按数据集和 shard 运行；
- GPU 步骤只能经单卡包装器；
- 长任务放入单一命名 `tmux` 会话；
- 输出先写 `.tmp`，校验后原子改名；
- 不用 `kill -9` 作为正常暂停方式；
- 不在失败后自动无限重试，同一错误最多做一次有依据的修复重试。

### 5.4 验证

验证数量、可解码性、尺寸、dtype、数值范围、有限性、hash、重复项、漏项和步骤专属门槛。GPU 步骤额外核验单卡可见性、物理卡号、峰值显存和进程退出后 GPU 是否释放。

### 5.5 通过标记

仅在验收全部通过时创建 `state/done/<STEP>.done.json`。最低字段：

```json
{
  "project": "qwenalpha-sod",
  "step": "S1.2",
  "status": "passed",
  "completed_at": "ISO-8601 with timezone",
  "input_hashes": {},
  "config_hash": null,
  "code_revision": "git revision or script sha256",
  "outputs": [],
  "output_hashes": {},
  "acceptance": {},
  "next_step": "S1.3"
}
```

`.done.json` 存在但输出缺失或 hash 不符时，该步骤不能视为完成；将状态设为 `failed` 或 `blocked`，保留异常证据。

### 5.6 更新交接状态

每一步结束、暂停或失败都要同步更新：

- `state/progress.yaml`；
- `state/history.jsonl`；
- `state/active_run.txt`；
- `reports/NEXT_ACTION.md`；
- 必要的阶段报告。

`NEXT_ACTION.md` 必须只给出一个下一动作，不能列出多个含糊选项。

### 5.7 本地文档

对用户可见的重要步骤应在本地 `paper/` 写报告，并在主计划中把复选框从 `[ ]` 改为 `[x]`，标题标记 `✅ 已完成：日期`。失败则写失败报告，但不得勾选完成。

### 5.8 Git 与同步

先做敏感文件和 allowlist 检查，再运行本工作区规定的一次性同步脚本。一个用户回合无论修改多少个 `paper/` 文件，只在全部修改及验证结束后同步一次。

### 5.9 停止

完成用户指定步骤后立即停止并报告，不自动执行下一步。报告只需包含：结果、验收、关键产物、异常、远端状态和下一步骤。

---

## 6. 当前下一步：S1.2 固化 manifest 和 hash

### 6.1 目标

在不修改原始数据的前提下，为三数据集建立可复现的训练、内部验证预留和公开测试清单，并对所有输入图和 GT 做 SHA-256，建立重复样本审计证据。

### 6.2 S1.2-0：解决完整训练数据来源

这是当前的显式阻塞项。先确认本地完整源目录与远端传输方式。不得猜测路径，不得把现有 2999 对测试副本冒充完整数据。

推荐远端结构：

```text
/data2/wangsiyi/datasets/raw/EORSSD/
/data2/wangsiyi/datasets/raw/ORSSD/
/data2/wangsiyi/datasets/raw/ORSI-4199/
```

要求：

- 镜像过程只读源数据；
- 复制前后分别保存源数量、相对路径、尺寸和抽样 hash；
- 远端目标先写临时目录，完成校验后再原子重命名；
- 排除无关缓存和派生目录；
- ORSSD 与 ORSI-4199 的 8 倍增强文件可以保留作证据，但正式 manifest 只能纳入无增强后缀原图；
- 不删除现有测试副本，除非另有明确、已验证且可恢复的迁移方案；
- 若当前执行者无法访问本地源数据，应把 S1.2 标记为 `blocked`，写清所缺路径和期望操作，停止等待用户；不得自行联网寻找替代数据集。

### 6.3 S1.2-1：定义稳定 sample ID

推荐格式：

```text
<dataset>/<official_split>/<relative_stem>
```

sample ID 必须由规范化相对路径确定，不得依赖绝对路径、目录遍历顺序或文件修改时间。Windows 与 Linux 路径分隔符统一为 `/`，排序使用字节序或明确记录的 Python 排序策略。

### 6.4 S1.2-2：分片计算 SHA-256

- 每个数据集独立分片，建议每 shard 128 或 256 对；
- 每个 shard 先写 `.tmp`，重新读取并验证行数后改名；
- shard 完成后写独立 `.done.json`；
- 恢复时仅跳过 hash 与 schema 均正确的 shard；
- 每个样本分别记录 image SHA-256 与 GT SHA-256；
- 记录字节大小、解码尺寸、图像模式和扩展名；
- 失败样本进入错误清单，不能从 manifest 静默丢弃。

建议 manifest 字段：

```text
sample_id,dataset,official_split,image_relpath,gt_relpath,
image_sha256,gt_sha256,image_bytes,gt_bytes,width,height,
image_mode,gt_mode,is_nonbinary_gt,is_empty_gt,foreground_ratio
```

### 6.5 S1.2-3：划分文件

本步只固定官方训练/测试身份和内部验证预留规则，不使用公开测试集调参。内部划分可在 S6.1 最终落实，但现在必须确保 manifest 中存在稳定、可追溯的候选信息。

至少输出：

```text
manifests/eorssd_train.csv
manifests/eorssd_test_public.csv
manifests/orssd_train.csv
manifests/orssd_test_public.csv
manifests/orsi4199_train.csv
manifests/orsi4199_test_public.csv
manifests/all_samples.csv
```

### 6.6 S1.2-4：重复与近重复审计

- 用 image SHA-256 检查跨划分和跨数据集完全重复；
- 用 GT SHA-256 检查完全相同标签，但不能仅凭相同 GT 判为泄漏；
- 对 ORSSD/EORSSD 做感知 hash 或稳健特征近重复检查；
- 输出候选对、距离、缩略图索引和处置建议；
- 不在本步骤删除候选样本；
- 若训练/测试存在疑似泄漏，状态设为 `blocked`，提交用户决定。

### 6.7 S1.2-5：确定性复跑

用同一代码对 manifest 重新生成至少两次。两次必须满足：

- 行数一致；
- sample ID 顺序一致；
- 每行字段一致；
- 总 SHA-256 一致；
- 与 S1.1 的正式样本数量一致；
- 源目录前后快照一致。

通过后创建 `state/done/S1.2.done.json`。下一唯一动作设为 S1.3。

### 6.8 S1.2 失败与恢复

| 情况 | 处理 |
|---|---|
| 完整训练源不可访问 | `blocked`，写所缺数据和源路径，不执行 S1.3 |
| 传输中断 | 保留已验证临时 shard；按 hash 续传，不从头覆盖 |
| 文件解码失败 | 对照 S1.1 与源文件 hash，记录差异并停止 |
| 数量不一致 | 输出缺失/多余相对路径差集，停止 |
| manifest 重跑 hash 不同 | 排查排序、时间戳和非确定字段，未解决前不得通过 |
| 疑似训练/测试泄漏 | 只报告候选，等待用户决定，不擅自删除 |

---

## 7. S1.3：构建 RGBA 监督目标

### 7.1 转换定义

对每个正式训练样本：

```text
Alpha_uint8 = GT 转为 L 模式后的原始 0～255 像素
Alpha_float = Alpha_uint8 / 255.0
RGB_target = round(ORSI_RGB × Alpha_float)
RGBA_target = concat(RGB_target, Alpha_uint8)
```

背景区域 RGB 必须为 0。非二值 GT 保留其软灰度，不在本步骤擅自阈值化；指标评测若需要阈值，必须作为独立评测规则记录。空 GT 必须保留，不能因全零而跳过。

### 7.2 分片与验证

- 每 128 张为一个 shard；
- 输出路径必须与 sample ID 可逆映射；
- PNG 先写临时文件，重新打开后原子改名；
- 验证 RGBA 模式、尺寸、Alpha 数值范围；
- Alpha 与原 GT 的逐像素最大差必须为 0；
- Alpha 为 0 的像素处 RGB 必须全为 0；
- manifest 记录源图、GT、RGBA 三者 hash；
- shard 通过后写 `.done.json`；恢复仅重做缺失或 hash 错误 shard。

S1.3 通过后才能执行 S1.4。

---

## 8. S1.4：难例标签与固定子集

### 8.1 样本特征

计算并记录：前景面积比例、连通域数量、周长、边界复杂度、细长度、孔洞、目标与背景对比度、阴影指标和空场景标记。

### 8.2 类别

至少覆盖：小目标、多目标、细长目标、孔洞、低对比、阴影、空场景和大目标。类别规则必须写入配置，不能人工凭印象临时挑图。

### 8.3 固定集合

- 从训练域分层选择 100 张 VAE 审计集；
- 从 EORSSD 训练域固定 48 张训练 + 16 张内部验证过拟合集；
- 不含任何公开测试样本；
- 只保存 sample ID 和选择理由，不复制为失去来源的新数据；
- 固定随机种子、分层规则和选择脚本 hash。

通过条件：指定难例类别均被覆盖，ID 均可回查 manifest，测试泄漏为 0。

---

## 9. S2：Qwen-Image-2.1 部署与 Alpha 表示上限

### 9.1 S2.1 下载和校验

- 权重只放 `/data2/wangsiyi/models/`；
- 使用支持断点续传的官方下载方式；
- 下载前检查剩余空间；
- 记录仓库 revision、文件清单、大小和 hash；
- 验证配置、DiT、Qwen3-VL 与 VAE 齐全；
- 不提交权重到 GitHub；
- 不在下载完成后自动启动 GPU 推理。

### 9.2 S2.2 八张 CPU-offload 推理

经单卡包装器运行，禁止 `device_map="auto"`。Qwen3-VL、DiT、VAE 顺序进入同一张 GPU。保存 8 张 RGBA、Alpha 灰度图、耗时、峰值显存和随机种子。

通过条件：8 张均成功；Alpha 非恒定、非丢失；峰值显存不超过 22 GB；进程仅见一张 GPU。失败则保留完整日志，不进入 S2.3。

### 9.3 S2.3 百张 VAE 审计

先执行 352 分辨率；只有接近门槛时才补 384 对照。对 100 张目标 RGBA 执行 encode/decode，提取重建 Alpha，输出逐图 CSV、整体统计、小目标统计和最差 20 张拼图。

Go 门槛必须同时满足：

```text
Sα >= 0.985
Fmax >= 0.985
MAE <= 0.005
至少 95% 样本 Boundary-F >= 0.970
小目标不存在系统性消失
```

No-go：写 `reports/S2_VAE_NOGO.md`，停止 Qwen 主线。不得重训 Qwen VAE、不得启用第二张卡。是否转入其他路线必须由用户决定。

---

## 10. S3：冻结编码器后的离线缓存

### 10.1 S3.1 增强表

先生成确定性增强参数表。同一 sample ID 的 ORSI、RGBA 和 Alpha 必须使用相同几何变换；Alpha 插值使用 nearest。每条记录有唯一 `augmentation_id` 和参数表 hash。暂停后不得重新随机生成已存在增强。

### 10.2 S3.2 条件 embedding

冻结 Qwen3-VL，每 32～64 条一个 shard。保存 embedding、mask、位置和尺寸元数据。每 shard 重新加载，检查 shape、dtype、finite 后原子改名并写完成标记。随机 32 条在线重算必须与缓存一致。

### 10.3 S3.3 RGBA latent

冻结 VAE encoder，按相同事务规则缓存目标 RGBA latent，并记录模型版本、latent scale、分辨率和源 RGBA hash。

### 10.4 S3.4 总审计

- 数量与训练 manifest 一致；
- 无 NaN/Inf；
- 无重复、无漏 ID；
- embedding、latent、GT hash 对齐；
- 随机解码 32 条，Alpha 指标与 S2 结论一致。

通过后锁定 cache manifest。训练阶段遇到缓存版本不符必须拒绝运行，不能静默重算。

---

## 11. S4：训练代码与 100-step 冒烟

### 11.1 固定基础配置

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

### 11.2 顺序

1. S4.1：损失、`z0_hat`、梯度链和冻结参数单元测试；
2. S4.2：batch=1 的 1-step 前后向、参数更新和 checkpoint 重载；
3. S4.3：10-step 稳定性与暂停恢复复测；
4. S4.4：固定配置的 100-step 正式冒烟。

初始损失：`L_flow + L_wBCE + L_wIoU`。若逐步 VAE 解码超显存，可改成 3 个纯 flow step 加 1 个 Alpha 辅助 step，但必须记录为配置变化。

通过条件：单卡、峰值不超过 22 GB、loss 有限、梯度无持续爆炸、LoRA 确实更新、Alpha 非持续噪声/全黑/全白、估算 EORSSD 单 epoch 不超过 12 小时。

OOM 降配顺序：降低 Alpha 解码频率 → LoRA 限于最后 8～12 层 → rank 8 降至 4 → 保持 352 → 分层 CPU offload。只允许一次有记录的降配复验；仍失败则停止 Qwen 路线。

---

## 12. S5：64 张过拟合门槛

固定 48 训练 + 16 内部验证样本、sampler、推理步数和 seed。先保存未微调 Qwen 基线，训练 0～500 step；只有 loss 正常且指标仍提升时才继续到 1000 step。每 50 step 保存 checkpoint/预测，并监控全黑、全白和平均前景面积。

验收：训练 `Sα >= 0.98`；验证显著优于未微调基线；无彩色内容渗入 Alpha；无系统性灰雾、全黑、全白、整体漂移；打乱、纯黑或错误输入使性能显著下降。

未通过时只排查数据错配、缓存、attention、条件注入和损失实现。不得靠扩大数据或盲目延长训练掩盖错误。

---

## 13. S6：EORSSD 最小闭环

从 1400 张官方训练集固定 90%/10% 内部划分并分层难例；600 张公开测试只在配置冻结后使用一次。串行完成：未微调 Qwen、DiT-LoRA + `L_flow`、DiT-LoRA + `L_flow + L_region`。

进入后续阶段的门槛：

```text
EORSSD Sα >= 0.920
MAE <= 0.010
小目标子集不完全失效
单张 3090 可完成推理
固定 checkpoint/config/seed 可复现
```

输出为 L 模式灰度 PNG，必须检查数量、尺寸、像素范围和命名。若未达到门槛，停止三数据集正式训练，先写瓶颈报告。

---

## 14. S7：Alpha 灰度图质量增强

按固定顺序一次只加入一个变量：F-measure loss → 边界损失 → 四方向梯度损失 → FFT 损失 → 难例重采样 → 384 分辨率 → 传统 SOD 教师软目标 → 可选二次 RGBA 精修。

每个变量用相同内部划分运行 3 个种子，记录均值、标准差和相对增益。平均 `Sα` 增益小于 0.2% 或 MAE 恶化则标记 `rejected`，不叠加。二次精修只有平均提升至少 0.3% `Sα` 才保留。

此阶段是“MLLM 直接灰度效果不佳”的主要改善机制，另外还包括：RGBA/Alpha 监督、区域损失、边界与方向结构约束、频域约束、难例采样、教师蒸馏和可选精修。任何改进都须以统一验证集和独立消融证明。

---

## 15. S8：三数据集正式训练

先创建 `FINAL_RECIPE.lock`，锁定代码、环境、缓存、manifest、超参数和评测协议。按 EORSSD → ORSSD → ORSI-4199 的顺序串行，每个数据集 seed 0 → 1 → 2 串行；一个 seed 完成训练、验证、测试与结果校验后才启动下一个。

每个数据集输出：最佳验证 checkpoint、三种子均值±标准差、四项核心指标、参数量、可训练参数、峰值显存、GPU 小时、单图延迟、全部灰度 PNG 及最好/典型/最差案例。

不得为某数据集的公开测试结果继续调参，也不得并行占用第二张 GPU。

---

## 16. S9：消融、泛化与收尾

### 16.1 必做证据

- 黑白 RGB 目标 vs RGBA/Alpha 目标；
- 冻结基座 vs DiT-LoRA；
- `L_flow` vs `+region` vs `+F` vs `+edge` vs `+FFT`；
- 无蒸馏 vs 单教师；
- 352 vs 384；
- 单次生成 vs 二次精修；
- 固定 seed vs 多 seed；
- ORSI-4199→EORSSD、ORSSD→EORSSD；
- 正确、打乱、纯黑和错误图像条件；
- 难例分组和最差 50 张分析；
- VAE 表示上限与最终误差相关性。

### 16.2 可复现包

- 从空环境到单卡推理的 README；
- 环境锁、配置、manifest 和评测协议；
- 一条单卡推理命令；
- 一条统一评测命令；
- 论文表格可追溯到原始 `metrics.json`；
- 公共包不含权重、数据、缓存、运行目录和秘密文件。

若最终指标没有超过 GeleNet 等传统方法，不能伪造“超越”。可将论文定位为：生成式 MLLM 原生 Alpha 能力在 ORSI-SOD 上的可行性、效率、边界与失败模式研究，并以可复现性、统一单卡约束、消融和新范式贡献支撑投稿。

---

## 17. 暂停、崩溃与恢复

### 17.1 计划暂停

1. 不启动新阶段；
2. 在 active run 的 `control/PAUSE_REQUESTED` 创建暂停请求；
3. 等当前 optimizer step 或 shard 完成；
4. 保存并重新加载 checkpoint/manifest；
5. 将 `progress.yaml` 写为 `paused`；
6. 写最后安全点、下一唯一动作和完整恢复命令；
7. 释放 GPU；
8. 更新 `NEXT_ACTION.md`。

### 17.2 网络或 VS Code 断线

长任务必须在 `tmux` 中运行。重新连接后先看 `tmux ls`、日志、PID 和 GPU，不重复启动。断线不等于暂停。

### 17.3 崩溃或重启

- 不删除现场；
- 找到最后一条完整 JSONL 日志；
- 校验最近 checkpoint；
- `.tmp`、截断日志和未完成 shard 标为无效；
- 先执行 `--resume-dry-run`；
- 从最近完整 checkpoint 恢复；
- 任何 hash 不一致都新建 run，不覆盖旧结果。

Checkpoint 必须含 LoRA、optimizer、scheduler、AMP、step/epoch/数据游标、Python/NumPy/PyTorch CPU/CUDA RNG、sampler seed、manifest/cache/config/base model hash 和最近验证指标。

### 17.4 Codex 额度即将不足时

在额度耗尽前完成以下最小交接事务：

```text
1. 停止启动新任务
2. 请求当前任务安全暂停
3. 验证最后 checkpoint 或 shard
4. 更新 progress.yaml/history.jsonl/active_run.txt
5. 更新 NEXT_ACTION.md，只留一个动作
6. 保存日志末尾与异常摘要
7. 若修改 paper/，完成验证后执行一次同步脚本
8. 向用户报告是否已达到安全接管点
```

如果额度不足以完成全部事务，应优先保障 GPU 安全暂停和 checkpoint，其次更新 `progress.yaml`，最后更新说明文档。

---

## 18. 状态文件规范

`progress.yaml` 建议保持以下字段：

```yaml
project: qwenalpha-sod
updated_at: "ISO-8601+08:00"
host: wangsiyi
remote_root: /data2/wangsiyi/qwenalpha-sod
physical_gpu: null
visible_gpu_count: 0
stage: S1
step: S1.2
status: blocked
reason: full_training_sources_not_mirrored_to_remote
active_run: ""
last_safe_checkpoint: null
last_verified_artifact: reports/S1_inventory_summary.json
next_action: establish_read_only_complete_dataset_mirror_then_generate_manifests
resume_command: null
```

状态只允许：`pending`、`running`、`paused`、`passed`、`failed`、`blocked`。`history.jsonl` 只追加，不重写。所有 YAML/JSON 更新先写临时文件、解析验证后原子替换。

`reports/NEXT_ACTION.md` 固定格式：

```markdown
# 当前状态
- 已完成：
- 当前阶段/步骤：
- 状态：
- 最后安全 checkpoint：
- 已验证输出：
- 当前问题：
- 下一条唯一命令：
- 预期通过条件：
- 禁止事项：
```

---

## 19. 失败处理决策表

| 事件 | 状态 | 必须做 | 禁止做 |
|---|---|---|---|
| GPU 被占用 | blocked | 记录占用事实，等待重新分配 | 轮询、抢占、kill 他人进程 |
| 路径越界/符号链接异常 | blocked | 停止并报告 realpath | 继续写入、sudo |
| 数据不完整 | blocked | 输出缺失清单 | 用测试副本冒充训练集 |
| OOM | failed/blocked | 按既定顺序降配一次 | 启用第二张卡 |
| NaN/Inf | failed | 保留 checkpoint 前日志并定位首个异常 step | 从头重跑掩盖问题 |
| checkpoint hash 不符 | failed | 新建 run 或回退到上一个完整 checkpoint | 覆盖旧 checkpoint |
| VAE 审计 No-go | failed | 写 NOGO 报告，停止主线 | 重训 VAE 或继续大训练 |
| EORSSD 未达门槛 | failed | 写瓶颈报告 | 启动三数据集正式训练 |
| GitHub 同步失败 | blocked | 保留本地文件和错误 | 改投其他仓库 |
| 指令冲突 | blocked | 列出冲突并请求用户裁决 | 自行扩大权限 |

---

## 20. Harness 未来获授权后的启动提示词

以下文本只在用户已经明确修改 Harness 禁用策略并授权接管后使用。未授权时不得发送或执行。

```text
你现在接管 QwenAlpha-SOD。先读取并严格遵守：
1) D:\OneDrive\Desktop\my paper\AGENTS.md
2) paper/QwenAlpha-SOD_Codex额度不足时Harness续执行计划书.md
3) paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md
4) 远端 /data2/wangsiyi/qwenalpha-sod/reports/NEXT_ACTION.md
5) 远端 state/progress.yaml、active_run.txt、history.jsonl 末尾和最近 .done.json

不要依赖历史聊天推断进度。先提交接管核验摘要，再只执行 progress.yaml 与 NEXT_ACTION.md 共同指向的一个原子步骤。远端只通过 SSH 别名 wangsiyi；不得 sudo；所有写入限定在 /data2/wangsiyi；所有 GPU 任务必须经 scripts/run_single_gpu.sh，且全程只用一张 RTX 3090、峰值不超过 22 GB，不得占用或终止他人进程，不得自动轮询忙卡。

每步必须完成 preflight、原子状态更新、执行、验收、.done.json、NEXT_ACTION 和本地报告；没有通过验收不得勾选完成或进入下一步。遇到数据缺失、路径异常、GPU 忙、Go/No-go 失败、权限或指令冲突时停止并报告，不自行扩大范围。禁止公开数据、权重、runs、tmp、秘密和第三方论文。若修改 paper/ 或 output/pdf/，在全部验证完成后仅运行一次规定的 QwenAlpha 同步脚本。

当前已知最后通过步骤为 S1.1，候选下一步为 S1.2；但必须先用远端状态文件复核。完成用户指定步骤后停止，不自动执行后续步骤。
```

---

## 21. 每次交接给用户的最小报告模板

```markdown
## 执行结果
- 步骤：
- 状态：Passed / Failed / Blocked / Paused
- 是否使用 GPU：
- 物理 GPU：
- 峰值显存：

## 验收
- 已通过：
- 未通过：
- 关键 hash：

## 产物
- 远端：
- 本地报告：
- GitHub 同步：

## 异常与处理
- 异常：
- 已采取措施：
- 未擅自采取的措施：

## 接管点
- 当前安全状态：
- 最后 checkpoint/shard：
- 下一唯一步骤：
- 需要用户决定：
```

---

## 22. 总验收清单

接管机制只有同时满足以下条件才算可用：

- [ ] 当前 `AGENTS.md` 已被读取，Harness 授权状态无歧义；
- [ ] 接管者无需历史聊天即可确定最后完成步骤；
- [ ] `.done.json` 与实际输出 hash 一致；
- [ ] 当前 active run、checkpoint 和唯一恢复命令明确；
- [ ] 单卡包装器是唯一 GPU 入口；
- [ ] 数据、模型、缓存、运行结果和公开仓库边界明确；
- [ ] 每个阶段有输入、输出、验收、暂停和失败条件；
- [ ] No-go 会真正停止后续高成本训练；
- [ ] `NEXT_ACTION.md` 只有一个下一动作；
- [ ] 本地文档修改后只进行一次合规同步；
- [ ] 用户能够从报告中判断项目是否安全暂停、是否可以接管。

当前项目的预期接管点为：**S0.1～S0.5、S1.1 已通过；S1.2 尚未执行；先建立三数据集完整只读远端镜像，再生成确定性 manifest 与 hash。**
