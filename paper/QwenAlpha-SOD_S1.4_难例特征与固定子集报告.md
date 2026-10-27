# QwenAlpha-SOD S1.4：难例特征与固定子集报告

> 执行日期：2026-10-27
> 结果：**Passed**
> 执行位置：远端 `wangsiyi`
> 输入：`manifests/all_samples.csv`（4,000 训练样本）+ `datasets/raw`（只读）
> 输出：`manifests/s1_4_features.csv`、`manifests/s1_4_subsets.csv`、`manifests/S1.4_evidence.json`

## 1. 规则（已冻结进配置）

全部阈值写在 `configs/hard_example_rules.yaml`，种子 `20261027`，改动必须升版本号并重跑。

| 特征 | 定义 |
|---|---|
| area_ratio | 前景像素 / 总像素 |
| components | 8-连通前景连通域数 |
| perimeter_px | 4-邻域越界的前景像素数 |
| thinness | `perimeter / (2√(π·area))`，圆形 = 1.0 |
| holes | 不接触图像边界的 8-连通背景区域数 |
| contrast | `\|mean_fg − mean_bg\|`（灰度 0–255 的幅值） |
| shadow_ratio | 前景中灰度低于 `0.4 × mean_bg` 的像素占比 |

| 类别 | 判据 | 训练集样本数 |
|---|---|---:|
| small | `0 < area_ratio < 0.01` | 660 |
| large | `area_ratio > 0.40` | 166 |
| multi | `components ≥ 3` | 951 |
| thin | `thinness > 3.0` | 1,360 |
| holes | `holes ≥ 1` | 167 |
| low_contrast | `contrast < 12` | 235 |
| shadow | `shadow_ratio > 0.45` | 301 |
| empty | `area_ratio == 0` | 74 |

## 2. 固定子集

**VAE 审计集（S2.3 使用）：100 张**

- 来源：全部训练域（三个数据集混合）
- 构成：EORSSD 38 / ORSI-4199 51 / ORSSD 11
- 类别覆盖：small 21、large 12、multi 31、thin 33、holes 13、low_contrast 13、shadow 13、empty 11 —— 八类均 ≥ 8
- 选取方式：按类别顺序各取 8 张（种子打乱后），不足 100 的部分用同一种子补齐

**过拟合集（S5 使用）：48 训练 + 16 内部验证**

- 来源：仅 EORSSD 训练域
- 类别覆盖：small 22、large 3、multi 8、thin 19、holes 3、low_contrast 5、shadow 5、empty 2 —— 八类均 ≥ 2
- 验证集优先按类别各取 1 张，其余随机补齐，保证小样本下仍有类别覆盖

两个子集**互斥**（重叠 0），且**不含任何公开测试样本**（`test_samples_selected: 0`）。子集只保存 sample ID、类别与选择理由，不复制像素。

## 3. 验证

| 检查 | 结果 |
|---|---|
| 审计集数量 | 100 / 100 |
| 过拟合集数量 | 48 + 16 |
| 八类是否全覆盖 | 是（审计 ≥8、过拟合 ≥2） |
| 审计集与过拟合集重叠 | 0 |
| 公开测试样本混入 | 0 |
| 审计集样本的 RGBA 目标是否齐备 | 100 / 100 存在 |
| 两遍独立运行输出 | 逐字节一致（`features` `fe3b3b2d…`，`subsets` `7c07ded4…`） |

## 4. 过程中修正的一处定义错误

初版把 `contrast` 定义为带符号的 `mean_bg − mean_fg`，结果 4,000 个样本里 3,129 个（78%）被判为"低对比"——因为本数据集中前景常常比背景**更亮**，中位数为 −54，负值全被误判。

改为对比度**幅值** `\|mean_fg − mean_bg\|` 后，`< 12` 灰阶对应总体约 6%（235 张），与"低对比"的语义相符。原始的有符号均值仍保留在 `mean_fg` / `mean_bg` 两列，可随时还原方向。

## 5. 可复现信息

```text
规则配置：configs/hard_example_rules.yaml
构建脚本：qwenalpha_sod/data/build_hard_subsets.py
代码提交：5724426（代码与产物）、3eb698e（收尾）
完成标记：state/done/S1.4.done.json
种子：20261027
```

## 6. 下一步

S1 到此全部完成。S2.1 需要：下载 Qwen-Image-2.1 权重到 `/data2/wangsiyi/models/`（记录 revision、文件清单、大小与 hash），并首次占用一张 RTX 3090。开工前需课题组指定物理卡号，并确认远端到模型仓库的下载通路。
