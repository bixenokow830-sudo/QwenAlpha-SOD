# QwenAlpha-SOD：S2.2 八张原生 RGBA 推理报告

**状态：** Passed

**完成时间：** 2026-10-29 19:41:29（Asia/Shanghai），验收固化 19:45 左右

**物理 GPU：** 1（进程内仅 1 张可见），RTX 3090 24 GB

**运行目录：** `/data2/wangsiyi/qwenalpha-sod/runs/S2.2/S2.2-20261029T193337+0800`

## 1. 目的与配置

验证 Qwen-Image-2.1 在单卡 3090、纯 CPU offload 下能稳定产出**原生 RGBA**，且 alpha 通道非恒定、非空。

| 项 | 值 |
|---|---|
| 模型 | `/data2/wangsiyi/models/Qwen-Image-2.1`（S2.1 已固化 29 文件 / 33,134,789,600 字节） |
| 精度 | bfloat16 |
| 显存策略 | `enable_model_cpu_offload()`；**未使用** `device_map="auto"` |
| 分辨率 | 1024 × 1024 |
| 采样步数 | 40 |
| 提示词 | 官方 RGBA 模板（`This is an RGBA image with transparency. … The image has alpha channel and the background is transparent.`） |
| 种子 | 20261029 + index（8 张，固定） |
| 启动方式 | `scripts/run_single_gpu.sh 1 -- …`（项目 flock 锁 + 单卡断言 + 忙碌卡拒绝） |
| diffusers / torch | 0.41.0 / 2.10.0+cu128 |

依据：模型 `vae/config.json` 为 `in_channels=4 / out_channels=4`，pipeline 源码中 RGBA 条件图经视觉编码器时按白底合成、而 **VAE 读取全部四个通道**（`pipeline_qwenimage21.py` 第 114–119 行）。因此原生 alpha 前提成立。

## 2. 结果

| # | 种子 | 耗时(s) | alpha 标准差 | 透明像素占比 | 输出模式 |
|---:|---:|---:|---:|---:|---|
| 0 | 20261029 | 59.4 | 126.76 | 0.504 | RGBA |
| 1 | 20261030 | 58.5 | 120.40 | 0.344 | RGBA |
| 2 | 20261031 | 58.7 | 74.59 | 0.907 | RGBA |
| 3 | 20261032 | 58.7 | 123.63 | 0.611 | RGBA |
| 4 | 20261033 | 58.7 | 121.41 | 0.644 | RGBA |
| 5 | 20261034 | 58.7 | 117.18 | 0.694 | RGBA |
| 6 | 20261035 | 58.7 | 115.90 | 0.706 | RGBA |
| 7 | 20261036 | 58.7 | 116.87 | 0.697 | RGBA |

- 8 张 alpha 全部覆盖 0–255 全量程，无一张为常量或空通道。
- 单张稳定在约 59 s；首张略高（59.4 s），无明显抖动。
- 显存：`torch.cuda.max_memory_allocated` 16.39 GiB、`max_memory_reserved` 16.57 GiB；设备级 nvidia-smi 峰值 **17,301 MiB（16.9 GiB）**，上限 22,528 MiB。

## 3. 验收对照

| 计划书 S2.2 条件 | 结果 |
|---|---|
| 单卡包装器启动 | ✅ `run_single_gpu.sh 1` |
| 禁止 `device_map="auto"` | ✅ 未使用，仅 offload 钩子 |
| Qwen3-VL → DiT → VAE 依次进入同一张 GPU | ✅ 文本编码、去噪、解码阶段逐一上卡 |
| 生成 8 张 RGBA | ✅ 8/8，模式均为 RGBA |
| 保存 Alpha 灰度图、耗时和显存 | ✅ 每张 `*_alpha.png`，JSON 记录耗时与峰值 |
| Alpha 不是恒定值或空通道 | ✅ 标准差 74.6–126.8，透明占比 0.34–0.91 |
| 8 张全部完成、峰值 ≤ 22 GB、仅一张 GPU 可见 | ✅ 17,301 MiB ≤ 22,528 MiB，可见卡数 1 |

`reports/S2.2_rgba_probe.json` 中 `acceptance.passed = true`。

## 4. 可复现性

单图复核运行（`--images 1 --tag peakcheck`，同种子）输出的 `00_rgba.png` 与 8 图运行的第 1 张**逐字节相同**：

```
3a4677aad82501f3a6e2987d4f3592cb8f9f149af1d78e774f3b79cd9049b5d5
```

两次运行的 `alpha_std` 也同为 126.7608。固定种子下结果确定，可作后续审计的基线。

## 5. 事故与处理（如实记录）

第一次 8 图运行在**全部图像完成后**的收尾阶段抛异常：

```
TypeError: 'Event' object is not callable
```

原因是探针脚本把线程属性命名为 `self._stop`，覆盖了 `threading.Thread._stop()`，`sampler.join()` 调用时崩溃。后果仅限于：进程退出码 1、汇总块未写入；**八张图像与逐图记录均已落盘且有效**。

处理方式：

1. 修复属性名（`_stop` → `_halt`），保留原运行目录与日志，不删除、不重跑 8 图；
2. 用相同配置跑 1 张做设备级峰值复核，同时验证修复后的脚本可正常收尾（退出码 0）；
3. 由 CPU 端的 `scripts/finalize_s2_2.py` 从已保存 PNG **重新计算** alpha 统计并与运行内记录逐项比对，8 张全部一致（`stat_mismatches = 0`），再合并峰值写回权威报告。

因此本轮不存在"因缺陷重跑已达标的步骤"，也没有用估算值替代测量值；设备级峰值来自同配置复核运行，来源在报告 `peak_vram_source` 中标注。

## 6. 远端证据

- 结构化结果：`reports/S2.2_rgba_probe.json`
- 峰值复核：`reports/S2.2_peakcheck_probe.json`
- 完成标记：`state/done/S2.2.done.json`
- 日志：`monitor/s2_2.log`、`monitor/s2_2_peakcheck.log`
- 相位表：`state/progress.yaml`；历史：`state/history.jsonl`
- 产物：`runs/S2.2/S2.2-20261029T193337+0800/`（8 × `*_rgba.png`、`*_alpha.png`、`*_preview.jpg`）

## 7. 资源与隔离

- 仅占用物理 GPU 1，未触碰 GPU 0，未抢占或终止他人进程。
- 全部写入位于 `/data2/wangsiyi`；未用 `sudo`，未改系统环境。
- 启动与结束时均为锁内运行，运行后显存回落至 15 MiB，无遗留进程。

## 8. 监控页修复

原监控页是 S2.1 下载视图：`total_bytes` 残留 33,134,789,600，而服务端把 `target_dir` 目录大小当作"已下载字节"，于是 S2.2 期间显示 `1.15 MiB / 30.86 GiB · 0.00%`，与事实无关。

已改为步骤无关：`index.html` 优先渲染 `progress{done,total,label}`，缺省才回退字节进度；`server.py` 在 `status.json` 声明 `progress_total` 时按 `progress_glob` 统计产物数。S2.2 期间显示"已生成 RGBA n/8"，结束时 8/8。此次修复只动监控，不影响科研任务。

## 9. 下一步

S2.3：在 S1.4 冻结的 100 张审计集上做原生 VAE 的 Alpha 上限审计（352 分辨率，必要时 384 对照），门槛为 Sα ≥ 0.985、Fmax ≥ 0.985、MAE ≤ 0.005、95% 样本 Boundary-F ≥ 0.970、小目标无系统性消失。

按规程**不自动启动**，等待明确指令。
