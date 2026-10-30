# QwenAlpha-SOD：W0 协议与 GT 防火墙报告

**状态：** Passed

**完成时间：** 2026-10-30 18:57:48（Asia/Shanghai）

**执行位置：** `/data2/wangsiyi/qwenalpha-sod`

**GPU 使用：** 否

## 1. 冻结结论

W0 已按非全监督主路线的推荐预算冻结：

- 主监督形式：点监督；
- 每个显著对象至少 1 个正点；
- 每图 2 个背景点；
- 约 20% 难例增加前景/背景涂鸦；
- 完整像素 GT 禁止进入训练过程；
- 从 GT 自动生成的点/涂鸦必须标记为“模拟弱标注”，不得冒充真实人工成本。

冻结配置：`configs/weak_supervision/protocol_v1.yaml`。

## 2. 数据角色

6999 个样本均获得唯一角色：

| 角色 | 数量 | 用途 |
|---|---:|---|
| `weak_train_pool` | 3443 | 弱监督主训练池 |
| `overfit_train` | 48 | 弱监督过拟合训练 |
| `overfit_validation` | 16 | 过拟合验证，不进入训练 |
| `development_calibration` | 110 | 100张 `vae_audit` + 10张提示词/分辨率校准样本 |
| `internal_validation` | 383 | 按数据集分层、固定哈希选取的内部验证集 |
| `official_test` | 2999 | 配置冻结后的正式一次性测试 |

10张已经参与 P2/1024 选择的样本被独立识别并并入开发校准集，没有回流弱训练池。内部验证集从剩余训练样本中按数据集分别使用固定 seed `20261030` 和 sample ID SHA-256 排序选取约10%。

主要清单：

- `manifests/weak_supervision/splits_v1.csv`
- `manifests/weak_supervision/training_pool_v1.csv`
- `manifests/weak_supervision/dense_gt_registry_v1.csv`
- `manifests/weak_supervision/blocked_dense_gt_paths_v1.txt`
- `manifests/weak_supervision/official_test_ids_v1.txt`

训练清单共3491行，只包含3443个弱训练样本和48个 overfit 训练样本；字段中没有 GT 路径、GT hash 或 GT 像素信息。

## 3. GT 防火墙

防火墙模块：`scripts/weak_supervision/gt_firewall.py`。

已登记全部6999条 dense-GT 真实路径。训练目的的文件访问必须经过 `DenseGTGuard(purpose="training")`：

- 原图访问：允许；
- 已登记 dense-GT：拒绝；
- `/data2/wangsiyi` 之外的路径：拒绝。

独立自测使用 `EORSSD/train/0002`：原图通过，`train-labels/0002.png` 被拒绝；`/tmp` 路径也被拒绝。完整结果：`reports/W0_gt_firewall_selftest.json`。

私有 GT registry 与阻断路径列表权限均为 `0600`。该权限防止其他普通用户读取，但真正的科研防火墙仍由训练 manifest、代码守卫、访问审计和实验协议共同构成。

## 4. 验收结果

| 检查 | 结果 |
|---|---:|
| split 行数/唯一 sample ID | 6999 / 6999 |
| dense-GT registry | 6999 |
| dense-GT 阻断路径 | 6999 |
| 官方测试进入训练池 | 0 |
| 训练 manifest GT 字段 | 0 |
| GT 阻断自测 | Passed |
| 越界路径阻断自测 | Passed |
| GPU 使用 | 否 |

结构化报告：`reports/W0_protocol_and_gt_firewall.json`。

报告 SHA-256：`e3b6e7c64483a78ccab3b4a4f68708aebd54ca5af65f2b1c5cac2333dc161603`。

完成标记：`state/done/W0.done.json`。

## 5. 状态与下一步

远端 `progress.yaml`、`history.jsonl`、`reports/NEXT_ACTION.md` 和监控页已经更新。W0 完成后没有 active run、没有 `tmux` 任务，GPU 字段为空。

下一原子步骤是 W1：在冻结的100张开发集上运行 P2@1024 零训练基线。W1 是 GPU 任务，必须由用户明确启动，并在启动前重新检查物理 GPU、其他用户进程、单卡锁和22 GB显存边界。
