# QwenAlpha-SOD：S2.3 原生 VAE Alpha 上限审计报告

**判定：** **No-go**（第 7.3 节五项红线未同时满足）

**执行时间：** 2026-10-29 19:50—19:52（Asia/Shanghai）

**物理 GPU：** 1（进程内仅 1 张可见），RTX 3090；峰值显存 1,508 MiB

**样本：** S1.4 冻结的 `vae_audit` 子集 100 张（EORSSD 38 / ORSI-4199 51 / ORSSD 11）

## 1. 做了什么

只加载 VAE（`AutoencoderKLQwenImage21`，1.35 GB，bf16），不加载 DiT 与文本编码器，因此这不是生成质量测试，而是**表示上限**测试：把 RGBA 目标压进 latent 再解出来，看 alpha 能保留多少。

```text
RGBA → encode(latent mode) → decode → 第 4 通道 → α∈[0,1]
GT  → 同尺寸最近邻 α（二值）
```

缩放：以 352² 为目标面积等比缩放，边长对齐 32（VAE 空间压缩 16×，pipeline 要求 32 的倍数）；384² 作对照。

指标（脚本内 `--self-test` 自检：完美预测必须 S=1、Fmax=1、MAE=0、BF=1）：

| 指标 | 定义 |
|---|---|
| Sα | Fan 等人的 structure measure：`0.5·S_object + 0.5·S_region`，区域项为四象限按 GT 均值加权，结果钳到 [0,1] |
| Fmax | 256 个阈值上 Fβ（β²=0.3）的最大值 |
| MAE | 重建 α 与二值 GT 的平均绝对误差 |
| Boundary-F | 形态学梯度提边，±2 px 容忍窗下的边界 F |
| IoU | α>0.5 与 GT 的交并比（辅助读数，非门槛） |

## 2. 结果

| 指标 | 352（主） | 384（对照） | 门槛 | 判定 |
|---|---:|---:|---|---|
| Sα 均值 | **0.9643** | **0.9642** | ≥ 0.985 | ❌ |
| Fmax 均值 | 0.9996 | 0.9997 | ≥ 0.985 | ✅ |
| MAE 均值 | **0.005218** | **0.005206** | ≤ 0.005 | ❌ |
| Boundary-F p5 | 0.9982 | 0.9983 | ≥ 0.970 | ✅（100/100） |
| 小目标消失 | 0 | 0 | 0 | ✅ |
| IoU 均值 | 0.9986 | 0.9988 | — | — |

384 与 352 相差 1e-4 量级，说明**分辨率不是瓶颈**，补 384 对照没有改变结论。

## 3. 失败结构（关键）

两处未达标来自不同机理，不能合并解释：

**（1）Sα 由极小目标拖累。** 21 张 `small` 样本 Sα 仅 0.8596；排除它们后的 79 张为 **0.9921，已越过 0.985 门槛**。面积占比 <1% 的 32 张 Sα 0.9060，≥1% 的 68 张 0.9917。这些目标面积占比低至 0.17%，S-measure 的区域项在近乎全零的分块上退化，对边缘轻微软化极其敏感。

**（2）MAE 是全子集一致的软边偏差。** `small` 0.0051、非 `small` 0.0052，与目标大小无关——每个边界像素都贡献约 0.5% 的误差。这是 VAE 重建 alpha 的系统性软边，不是个别样本的失败。

**没有发生的事**：没有任何样本消失（`pred_area/gt_area` 最小 0.9943）、没有形状崩坏（IoU 0.9986，仅 1 张 0.9877）、Boundary-F 全部 100 张 ≥0.9943。最差 20 张的 GT/预测对照蒙太奇显示轮廓基本重合，差异集中在细尖端变细。

## 4. 判定说明

第 7.3 节要求五项红线**同时**满足。Sα 与 MAE 未过，故判定 No-go。**没有**因为结论不利而剔除小目标、没有改门槛、没有换指标重算——若要改变判定口径，必须由用户明确授权并事前固定。

## 5. 处置

按计划书第 7.3 节：

- 停止 QwenAlpha 主线；
- 不得重训 Qwen VAE；
- 不得启用第二张卡；
- 是否转入 Show-o2 单卡回退，**由用户决定**。

本轮未创建 `state/done/S2.3.done.json`（验收未通过，按规程不得创建）；`state/progress.yaml` 记为 `nogo`，`state/history.jsonl` 已追加事件，监控页显示 No-go。

## 6. 证据

- `reports/S2_VAE_NOGO.md`（远端 No-go 报告）
- `reports/S2.3_vae_audit.csv`、`reports/S2.3_vae_audit_384.csv`（逐图指标）
- `reports/S2.3_vae_audit.json`、`reports/S2.3_vae_audit_384.json`（汇总）
- `runs/S2.3/alpha_pred/`、`runs/S2.3/alpha_gt/`（重建与 GT alpha）
- `runs/S2.3/worst20_montage.png`（最差 20 张对照）
- `monitor/s2_3.log`、`monitor/s2_3_384.log`
- `scripts/s2_3_vae_audit.py`（含指标自检与断点续跑）

单张 encode/decode 约 0.012–0.014 s，全量 100 张约 12 s（不含加载）。

## 7. 复现

```bash
cd /data2/wangsiyi/qwenalpha-sod
bash scripts/run_single_gpu.sh 1 -- /data2/wangsiyi/envs/qwenalpha-sod/bin/python scripts/s2_3_vae_audit.py
bash scripts/run_single_gpu.sh 1 -- /data2/wangsiyi/envs/qwenalpha-sod/bin/python scripts/s2_3_vae_audit.py --base-size 384
```
