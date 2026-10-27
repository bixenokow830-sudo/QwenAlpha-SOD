# QwenAlpha-SOD S1.3：RGBA 监督目标构建报告

> 执行日期：2026-10-27
> 结果：**Passed**
> 执行位置：远端 `wangsiyi`，`/data2/wangsiyi/qwenalpha-sod`
> 输入：`datasets/raw`（只读镜像）+ `manifests/all_samples.csv`
> 输出：`datasets/derived/rgba/{EORSSD,ORSSD,ORSI-4199}/train/`

## 1. 转换定义（与计划书一致）

对 `official_split == train` 的每个样本：

```text
Alpha_uint8 = GT 转为 L 模式后的原始 0～255 像素
Alpha_float = Alpha_uint8 / 255.0
RGB_target  = rint(ORSI_RGB × Alpha_float)     # 半偶舍入
RGBA_target = concat(RGB_target, Alpha_uint8)
```

- 背景（alpha=0）的 RGB 强制为 0，不依赖乘法结果；
- 非二值 GT 保留软灰度，**不阈值化**；
- 空 GT 保留，不因全零跳过。

只构建训练划分。测试划分不生成 RGBA：评测用预测 Alpha 与原始 GT 掩码比较，不需要 RGBA 目标。

## 2. 结果

| 项 | 值 |
|---|---|
| 记录数 | 4,000（EORSSD 1400 / ORSSD 600 / ORSI-4199 2000） |
| 分片 | 32 个，每片 128 张，均可续跑 |
| 错误 | 0 |
| 输出体积 | 130,179,552 字节（130.2 MB） |
| 清单 | `manifests/s1_3_rgba_manifest.csv`（4,000 行），SHA-256 `9574951b…9dff95` |
| 证据 | `manifests/S1.3_evidence.json`，`passed: true` |

体积远小于源图合计，是因为 alpha 之外全为黑，PNG 压缩率极高——这是预期结果，不是图像丢失。

## 3. 验证

**构建器内建（写盘后重开文件比对）**

| 检查 | 结果 |
|---|---|
| PNG 模式为 RGBA、尺寸与源图一致 | 4,000 / 4,000 |
| alpha 通道与原 GT 的 L 像素最大差 | **0**（全部样本） |
| alpha=0 处 RGB 非零像素数 | **0**（全部样本） |
| 源图 / 源 GT 哈希与 S1.2 清单一致 | 4,000 / 4,000 |

**独立复核（另写脚本，不共用构建器代码）**

| 检查 | 结果 |
|---|---|
| 重新哈希全部 4,000 个 PNG，与清单比对 | 0 处不符 |
| 文件大小与清单比对 | 0 处不符 |
| 分层抽样 12 张重算 `rint(RGB × alpha/255)` | 12 / 12 逐像素一致 |
| 确认无任何测试划分样本被写入 | 是（`splits present: {train}`） |

## 4. 特殊样本

| 类别 | 训练集合计数 | 处置 |
|---|---:|---|
| 非二值 GT | 89 | 保留软灰度，实测样本软像素数 1～4,645 |
| 空 GT | 74 | 保留，输出为全黑 RGBA，不跳过 |

抽样示例（均为逐像素 0 差异）：

```text
ORSI-4199/train/123    alpha_diff=0  rgb_diff=0  soft_px=1
ORSI-4199/train/732    alpha_diff=0  rgb_diff=0  soft_px=4645
EORSSD/train/0116      alpha_diff=0  rgb_diff=0  soft_px=0     （空 GT）
```

## 5. 可复现信息

```text
构建脚本：qwenalpha_sod/data/build_rgba_targets.py
          SHA-256 9ad5d1b2fb4dfa3b8395652815ec863bf603ae87877c1bb60d78784e6c493271
代码提交：6786566（含该脚本），收尾提交 9f92350
输入清单：manifests/all_samples.csv  ba11c6c3…1446a0
输出清单：manifests/s1_3_rgba_manifest.csv  9574951b…9dff95
完成标记：state/done/S1.3.done.json
```

写入方式：每张先写 `<name>.png.tmp`，重新打开校验通过后原子改名；每个分片写完 `.jsonl` 后回读比对行数，再写 `.done.json`。中断恢复只重做缺失或哈希不符的分片。

## 6. 过程中修正的一处问题

首次全量运行时，`build_rgba_targets.py` 尚未提交到版本库，导致完成标记里的 `code_revision` 指向的树里没有该脚本。已补记 `builder_script_sha256` 并把 `code_revision` 指向包含脚本的提交 `6786566`，使"跑的是哪份代码"可追溯。

## 7. 下一步

S1.4：从训练域计算前景面积比、连通域、周长、细长度、孔洞、对比度、阴影与空场景特征，固定 100 张 VAE 审计集与 48+16 过拟合集，只保存 sample ID 与选择理由，不得纳入任何公开测试样本。
