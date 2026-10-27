# QwenAlpha-SOD S0.3：Python 环境建立报告

> 执行日期：2026-10-27  
> 远端主机：`wangsiyi`  
> 远端工程：`/data2/wangsiyi/qwenalpha-sod`  
> 结果：**✅ Passed**

## 1. 独立环境

- 环境管理器：Micromamba 2.10.0；
- Python 环境：`/data2/wangsiyi/envs/qwenalpha-sod`；
- Python：3.11.18；
- pip 临时目录：`/data2/wangsiyi/tmp/pip`；
- pip 缓存：`/data2/wangsiyi/cache/pip`；
- Hugging Face 缓存：`/data2/wangsiyi/cache/huggingface`；
- Torch 缓存：`/data2/wangsiyi/cache/torch`。

环境、缓存、临时文件和第三方源码全部位于 `/data2/wangsiyi` 内，没有使用 `sudo`，没有修改系统 Python。

## 2. GPU 与 PyTorch 验收

验证时只暴露物理 GPU 1：

```text
CUDA_DEVICE_ORDER=PCI_BUS_ID
CUDA_VISIBLE_DEVICES=1
torch.cuda.device_count() = 1
torch.cuda.get_device_name(0) = NVIDIA GeForce RTX 3090
GPU memory = 25307578368 bytes
NVIDIA driver = 570.133.20
PyTorch = 2.10.0+cu128
PyTorch CUDA runtime = 12.8
Torchvision = 0.25.0+cu128
```

验证进程只看见一张 GPU，程序内部设备编号为 `cuda:0`。本步骤没有下载模型权重，也没有启动训练。

## 3. 核心依赖

```text
Transformers 5.17.0
Diffusers 0.41.0
Accelerate 1.11.0
PEFT 0.21.2
Hugging Face Hub 1.33.0
DiffSynth 2.1.12（editable）
```

`pip check` 返回 `No broken requirements found`。在全新的 SSH 进程中重新执行验证脚本，以下项目全部通过：

- 核心库导入；
- `diffusers.QwenImage21Pipeline` 导入；
- CUDA 可用且可见 GPU 数量严格等于 1；
- GPU 名称为 RTX 3090；
- 环境重开后仍可复现以上结果。

## 4. DiffSynth 分段注意力核验

- 官方仓库：`https://github.com/modelscope/DiffSynth-Studio.git`；
- 固定提交：`e5a44c42776978457b61d51d595a1f6746c3d87a`；
- 源码位置：`diffsynth/models/qwen_image_21_dit.py`；
- 已确认 `_qwenimage21_prefix_segments` 与 `segments = _qwenimage21_prefix_segments(...)` 存在；
- 已确认 Qwen-Image-2.1 管线默认 `use_flex_attention=False`。

因此当前环境具备计划要求的 segmented SDPA 路径，不依赖构建完整 BlockMask。

## 5. 安装失败与恢复

首次 PyTorch 安装在解包 CUDA wheel 时出现：

```text
OSError: [Errno 28] No space left on device
```

原因是系统根分区 `/` 仅余约 1 GiB，而 pip 默认使用系统 `/tmp`。本次没有修改依赖版本，仅把 `TMPDIR` 改为 `/data2/wangsiyi/tmp/pip` 后用完全相同的 PyTorch 版本重试，安装与单卡验证均成功。详细记录保存在远端 `reports/S0_dependency_failures.md`。

后续所有 pip 安装必须继续显式设置 `/data2/wangsiyi/tmp/pip`，避免再次占用系统盘。

## 6. 远端可恢复产物

```text
reports/environment.lock.txt
reports/pip_freeze.txt
reports/conda_explicit.txt
reports/S0_dependency_failures.md
scripts/s0_3_verify.py
state/done/S0.3.done.json
state/progress.yaml
state/history.jsonl
reports/NEXT_ACTION.md
```

关键哈希：

```text
environment.lock.txt  ab577bd468e8f654be21666380d6e4df247324d899aa5a31dfec89c2421b388e
pip_freeze.txt        37851f1ca95267a3ca9308e637b956d3163a5de8d79557353cbb78406d92a49a
conda_explicit.txt    e39b10ea6e60a2e072950241a96be133d44a02a0317f32741ec6b898284b8dc1
s0_3_verify.py        3e8806fab801e4d068eb7bca5be431f3295e8ba39a180462e17b11b31b3ff216
```

## 7. 下一步骤

下一步骤为 **S0.4：实现单卡运行包装器**。在 S0.4 通过前，不下载 Qwen 模型权重，不进入训练。
