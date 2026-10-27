# QwenAlpha-SOD S1.2：清单固化与重复审计报告

> 执行日期：2026-10-27
> 结果：**产物完备，状态 `blocked`（等待重复样本处置决定）**
> 执行位置：远端 `wangsiyi`，`/data2/wangsiyi/qwenalpha-sod`
> 操作性质：只读镜像上构建，未修改 `datasets/raw` 任何样本

## 1. 本步做了什么

按 S1.2-0～S1.2-5 执行：

1. **S1.2-0** 把本地三套完整源以 tar 单流经 ssh 推到远端，落 `.staging` 校验后原子改名为 `datasets/raw`，并置只读（见 `reports/S1.2-0_dataset_mirror.json`）。
2. **S1.2-1** 定稳定 sample ID：`<dataset>/<official_split>/<stem>`，由规范化相对路径确定，不依赖绝对路径、遍历顺序或 mtime。
3. **S1.2-2** 分片计算 SHA-256：30 个分片（256 对/片），每片先写 `.jsonl.tmp`、回读校验行数与 JSON 可解析后再原子改名，并写 `.done.json`；中断可续跑。
4. **S1.2-3** 生成 `manifests/all_samples.csv` 与六份划分 CSV。
5. **S1.2-4** 重复与近重复审计（精确 SHA-256 + dHash≤6），只报告不删除。
6. **S1.2-5** 用同一代码全量重算第二遍，比对总哈希。

新增代码：`qwenalpha_sod/data/build_manifests.py`。

## 2. 验收结果

| 项 | 结果 |
|---|---|
| 记录数 | 6,999（EORSSD 1400/600、ORSSD 600/200、ORSI-4199 2000/2199） |
| 分片 | 30 / 30 完成，均可续跑 |
| 解码错误 | 0 |
| `all_samples.csv` SHA-256 | `ba11c6c34b899c8b555bd09533734b244ff1ff8a0f95f372d80f3264711446a0` |
| 确定性复跑 | 第二遍全量重算得到**同一哈希**，行数一致 |
| 非二值 GT / 空 GT | 575 / 96，与 S1.1 完全一致 |
| 数据源写保护 | `data_root_read_only: true`，镜像为只读 |

产物：`manifests/{all_samples,eorssd_train,eorssd_test_public,orssd_train,orssd_test_public,orsi4199_train,orsi4199_test_public}.csv`、`manifests/S1.2_manifest_evidence.json`、`manifests/duplicate_report.json`、`manifests/duplicate_analysis.json`。

## 3. 重复审计发现

### 3.1 训练/测试同图（真正的泄漏候选）

| 组 | 样本 A | 样本 B | 图像 | GT 前景比 |
|---|---|---|---|---|
| 1 | `EORSSD/test/0005` | `EORSSD/train/0003` | 字节完全相同，973×760，73,330 B | 0.014467 vs 0.014126 |
| 2 | `ORSSD/test/0005` | `ORSSD/train/0003` | 字节完全相同，973×760，73,309 B | 同上 |

两组的图像哈希相同，**GT 不同**。

### 3.2 同划分内部重复（数据质量，非泄漏）

| 数据集 | 重复对 |
|---|---|
| ORSI-4199 test | `2008≡2010`、`bridges_NSO_34≡bridges_NSO_45`、`ships_SA_12≡ships_SA_32`、`stadiums_BSO_ (40)≡stadiums_MSO_ (1)`、`stadiums_BSO_ (8)≡urban landmarks_BSO_ (38)` |
| ORSI-4199 train | `1139≡1201`、`1605≡1606` |
| EORSSD test | `0015≡0016` |
| ORSSD test | `0015≡0016` |

共 9 组、18 张图像，同样全部是"图像相同、GT 不同"。

### 3.3 跨数据集重叠

- 完全相同图像：158 组，均为 `EORSSD/<split>/<stem>` 与 `ORSSD/<split>/<stem>` 同名同字节；
- dHash 近重复对：1,100 对（跨数据集）、239 对（同数据集跨划分）、263 对（同划分）。

## 4. 为什么会重复（证据）

三条独立证据指向同一结论：**重复来自数据集的构造方式，不是我们的处理错误**。

1. **图像逐字节相同、GT 却不同。** 若是文件拷贝事故，GT 会一并相同。GT 不同说明这是同一张源图被独立标注了两次。差异幅度都很小（如 0.014467 vs 0.014126），符合两名标注者或两轮标注的偏差，而非两张不同图。
2. **目录时间戳分层明显。** ORSSD 图像 mtime 为 2017-08-09，EORSSD 为 2019-11-21。EORSSD 即 ORSSD 的扩展版（Extended ORSSD），沿用同一批遥感场景并重新标注，因此出现大量同名同图的跨数据集重复；少数场景在扩展时被分到了不同划分，形成第 3.1 节的泄漏对。
3. **ORSI-4199 的文件名带类别标签。** 重复样本是 `ships_SA_12`、`stadiums_BSO_ (40)`、`urban landmarks_BSO_ (38)` 这类"场景_类别_编号"命名，说明该数据集是采集后按类别归档的。同一张图被归入两个类别（如 `ships_SA_12` 与 `ships_SA_32`，前者 GT 前景比 1.0、后者 0.0022），是归档时重复录入，并顺带产生了互相矛盾的标注。

结论：重复属于**数据集固有属性**。我们只做检测与记录，未删除、未改写、未重命名。

## 5. 数量与占比

| 类别 | 组数 | 涉及图像 | 占对应划分比例 |
|---|---:|---:|---|
| 训练/测试同图 | 2 | 2 张唯一图像 | EORSSD 测试 1/600（0.17%）；ORSSD 测试 1/200（0.50%） |
| 同划分重复 | 9 | 18 张 | ORSI 测试 10/2199（0.45%）；ORSI 训练 4/2000（0.20%） |
| 跨数据集重叠 | 158（精确）+ 1,100（近重复） | — | ORSSD 800 张中约 158 张与 EORSSD 同源 |

即：**真正的训练/测试泄漏只有 1 张图，且在两个互为扩展版的 dataset 里各出现一次。** 数量极小，不是系统性缺陷。

## 6. 处置方案与后果

| 方案 | 做法 | 后果 |
|---|---|---|
| **A 最小侵入** | 把 `EORSSD/train/0003`、`ORSSD/train/0003` 从**训练集**剔除，测试集保持官方 600/200 不变 | 训练集 1400→1399（-0.07%）、600→599；泄漏消除；测试集与文献口径一致，指标仍可横向比较。**推荐** |
| **A′** | 改为剔除测试图 `test/0005` | 测试集 600→599、200→199；与已发表 EORSSD/ORSSD 指标不再严格可比，审稿易被质疑 |
| **B 保留并声明** | 数据不动，仅在论文数据章节声明各 1 张同图 | 零改动；泄漏影响上界约 0.17%（单图在 600 张均值中的权重），但需在论文中主动披露 |
| **C 严格去重** | S6.1 内部划分时对训练/测试做哈希去重，并隔离 158 组 ORSSD∩EORSSD 重叠 | 最干净；但需额外记录排除清单，且测试指标将基于非官方子集 |

对 S9 的独立影响（与 A/B/C 无关）：由于 ORSSD 与 EORSSD 部分同源，**不能把 `EORSSD→ORSSD` 或 `ORSSD→EORSSD` 作为严格跨数据集泛化证据**，除非先扣除重叠样本；ORSI-4199 与二者无完全相同图像，可作为主要外部验证集。

## 7. 当前状态与下一步

- `progress.yaml`：`step: S1.2`、`status: blocked`、`reason: train_test_duplicate_candidates_pending_user_decision`；
- 未创建 `state/done/S1.2.done.json`（未通过不等于通过）；
- `reports/NEXT_ACTION.md` 只保留一个动作：记录处置决定；
- 若方案改变 manifest 内容，需重跑 `python qwenalpha_sod/data/build_manifests.py --verify-rerun` 并重新验收两次哈希一致；
- 处置决定记录后，方可进入 S1.3（构建 RGBA 监督目标）。

## 8. 处置决定（2026-10-27，已生效）

用户选择 **方案 B：保留并声明**。

- `datasets/raw` 全部样本保持原样，未删除、未改名、未重生成；
- `all_samples.csv` 的 SHA-256 仍为 `ba11c6c3…1446a0`，manifest 无需重跑；
- 论文数据章节须声明：EORSSD、ORSSD 各有 1 张训练图像在测试集中重复出现（`test/0005 ≡ train/0003`），占测试集 1/600 与 1/200，未做剔除；
- ORSSD 与 EORSSD 部分同源（158 组完全相同图像），`EORSSD→ORSSD` 不能作为严格跨数据集泛化证据，须先扣除重叠；
- 同划分内部重复 9 组保留，作为数据质量说明写入论文。

S1.2 据此关闭为 `passed`，`state/done/S1.2.done.json` 已生成，下一步为 S1.3（构建 RGBA 监督目标）。
远端处置记录：`reports/S1.2_duplicate_decision.md`，提交 `c89ad8c`。

> 同步状态：本报告尚未推送到 GitHub。本机当前没有可用的 git 二进制，一次同步尝试在环境检查阶段即失败（详见对话记录）；本地文件完整保留，未改投其他仓库，待工具链可用后补推。
