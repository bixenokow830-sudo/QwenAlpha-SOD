# QwenAlpha-SOD 共用进度说明书（Harness / Codex 通用）

**版本：** v1.1

**更新时间：** 2026-10-30 19:00（Asia/Shanghai）

**维护规则：** 本文件是**活文档**。任何 agent 每完成一个原子步骤，必须在本文件 §11 变更日志追加一行，并同步更新 §0 速览、§2 进度表、§6 未决事项。**更新本文件是步骤收尾的一部分，不是可选项。**

**用途：** 供任意 agent（DeepSeek Harness、Codex 或后续接手者）在**不读历史聊天**的前提下恢复到可执行状态。本文件与远端状态文件互为校验；两者冲突时以**实际产物**为准，并回写本文件。

**当前计划版本：** `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md` v1.0（W0–W9 系列），提交 `1840208`。该计划把主路线从"全监督微调"改为"非全监督"，并新增 W 编号体系。

---

## 0. 三十秒速览

| 问题 | 答案 |
|---|---|
| 项目现在在哪一步 | S2.3 已判 **No-go**；S2.4 探针与分辨率配对已完成；**W1（100 张零样本开发基线）尚未跑** |
| 最重要的一组数 | 零样本 P2@1024：**Sα 0.5900 / MAE 0.0322**（10 张配对）；表示上限 0.9643；传统方法约 0.90 |
| 下一步唯一动作 | W1：在 100 张 `vae_audit` 上跑 P2@1024（约 2 小时，物理 GPU 0） |
| 绝不能做的事 | 重跑已有 done marker 的步骤；碰他人 GPU 进程；未经确认改冻结协议；让训练池混入 100 张开发样本 |

---

## 1. 环境与路径（含已验证的坑）

| 项 | 值 |
|---|---|
| 本地工作区 | `D:\BaiduSyncdisk\CSCD\Qwenalpha-SOD`（**新路径**；旧副本 `D:\桌面\CSCD\Qwenalpha-SOD` 仍存在，未删） |
| SSH | `ssh wangsiyi`（`10.101.1.254:19059`，需实验室网络或 VPN） |
| 远端项目 | `/data2/wangsiyi/qwenalpha-sod` |
| 远端允许写入根 | `/data2/wangsiyi`（该账号 HOME 也在此目录内） |
| 模型 | `/data2/wangsiyi/models/Qwen-Image-2.1`，29 文件 / 33,134,789,600 字节，HF `d26bb612…`，ModelScope `master` |
| 数据集 | `/data2/wangsiyi/datasets/{raw,derived}`（注意不在项目目录下） |
| Python | `/data2/wangsiyi/envs/qwenalpha-sod/bin/python` |
| 监控 | 远端 `127.0.0.1:8765`，本地隧道：`ssh -N -L 127.0.0.1:8765:127.0.0.1:8765 wangsiyi` |

### 1.1 已知的坑（每条都踩过）

1. **PowerShell 会破坏远端命令的引号**。含引号、中文、`|`、多行的命令一律用 base64 走 stdin：
   `$b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($script))`，然后 `ssh wangsiyi "echo $b64 | base64 -d | bash"`。
2. **`git` 不在 PATH**。用 Codex 自带的：
   `C:\Users\28446\.cache\codex-runtimes\codex-primary-runtime\dependencies\native\git\cmd\git.exe`。
3. **GitHub 需要代理**。Clash 在 `127.0.0.1:7890`，直连 443 会超时；跑同步前设 `HTTPS_PROXY=http://127.0.0.1:7890`、`HTTP_PROXY` 同值、`GIT_TERMINAL_PROMPT=0`、`GCM_INTERACTIVE=Never`。凭据由 `manager` 助手缓存，无需交互。
4. **本地工作区在百度同步盘内**：不要把权重、数据集、runs、checkpoint 放进来；Git 操作在本机进行，不要依赖网盘同步 `.git`。
5. **GPU 1 目前被他人占用**：`yuanxiaofeng` 的 `python train.py --epoch 800`，约 8.4 GB，已运行 21 小时以上。**禁止**终止或挤占。当前改用**物理 GPU 0**（空闲，仅 Xorg 占几十 MiB）。
6. **监控隧道会随本地断网/重启失效**（浏览器报 `ERR_CONNECTION_REFUSED`）。远端服务不受影响，重建隧道即可，**不要重启远端监控进程以外的任何任务**。
7. **`curl` 在远端不存在**，用 Python `urllib` 或 `wget`。

---

## 2. 已完成步骤与证据

| 步骤 | 结论 | 关键证据 |
|---|---|---|
| S0.2–S0.5 | 工程骨架、Python 环境、单卡包装器、暂停恢复测试 | `state/done/S0.*.done.json` |
| S1.1–S1.4 | 数据盘点、manifest 与重复审计、RGBA 目标、难例子集 | `state/done/S1.*.done.json` |
| S2.1 | 模型下载与逐文件 SHA-256 校验 | `reports/S2.1_model_download_manifest.json`、`state/done/S2.1.done.json` |
| S2.2 | 8 张原生 RGBA 推理 | `reports/S2.2_rgba_probe.json`、`state/done/S2.2.done.json` |
| S2.3 | 原生 VAE Alpha 上限审计 → **No-go** | `reports/S2_VAE_NOGO.md`、`reports/S2.3_vae_audit*.json` |
| S2.4（探针） | 提示词模板 P2 胜出；1024 与 352 配对完成 | `reports/S2.4_prompt_selection_352.json`、`reports/S2.4_prompt_selection_1024p2.json` |
| 计划（非全监督主路线） | W0–W9 详细计划书 v1.0 已入仓 | `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md`（commit `1840208`） |

**S1 关键事实：** 训练对 6,999；manifest `ba11c6c3…1446a0`；2 组 train/test 精确重复 + 158 组跨源同源样本（已披露，未删除）；RGBA 派生 4,000；审计集 100；overfit 48/16。

**S2.2 关键事实：** 8 张全部 RGBA，alpha 0–255 非常量；峰值 17,301 MiB；同种子跨运行输出**逐字节相同**（`3a4677aa…`）。

---

## 3. 关键数字（一页看完）

### 3.1 表示上限（S2.3，oracle 输入，仅 VAE）

| 口径 | Sα 均值 | MAE 均值 | Fmax | Boundary-F p5 |
|---|---:|---:|---:|---:|
| 352（主） | **0.9643** | **0.005218** | 0.9996 | 0.9982 |
| 384 | 0.9642 | 0.005206 | 0.9997 | 0.9983 |
| 1024 | 0.9537 | 0.005632 | 0.9998 | 1.0000 |

判定 No-go 的两项：Sα 0.9643 < 0.985、MAE 0.005218 > 0.005。小目标子集 Sα 0.8596（非小目标 0.9921）。**提高分辨率没有抬高上限，反而更差。**

### 3.2 零样本基线（S2.4，仅原图输入）

| 模板 | 352 亮度 Sα | 1024 亮度 Sα（10 张配对） |
|---|---:|---:|
| P1 原生透明 | 0.0071 | — |
| **P2 去背景+纯黑底** | **0.4835** | **0.5900** |
| P3 显式术语 | 0.2681 | — |
| P4 极简高亮 | 0.0419 | — |

1024 相对 352 的配对增益 **+0.1065**，8/10 提升，峰值显存 19,537 MiB。

### 3.3 参照线

- 传统全监督方法：约 **0.90**（具体数值必须取自原文表格，尚待录入）。
- 计划书 S6 目标线：Sα ≥ 0.92、MAE ≤ 0.010。
- 表示上限：0.9643。

---

## 4. 已冻结协议（改则必须由用户书面确认并记录）

| 项 | 冻结值 |
|---|---|
| 输入 | **仅原图**，不提供 GT/mask/类别/数据集名 |
| 输出 | 灰度图；`Y = 0.299R + 0.587G + 0.114B`（BT.601），不拉伸、不加 gamma |
| 提示词 | P2：`Remove the background and keep only the salient object, on a pure black background.` |
| 分辨率 | 1024 面积预算（边长按原图长宽比、对齐 32） |
| 采样 | 40 步，seed = 20261029 + index，bf16，`enable_model_cpu_offload` |
| 评测集 | 100 张 `vae_audit`（EORSSD 38 / ORSI-4199 51 / ORSSD 11，全部 train 划分） |
| 指标 | 直接 `from s2_3_vae_audit import s_measure, fmax, boundary_f, iou`，不重写 |
| 口径 | 主：GT 原尺寸（预测双线性、GT 最近邻）；辅：352² |
| 模板选择规则 | 亮度 Sα 均值定胜负；alpha 只并列报告、不参与选择 |

**协议出处：** `paper/QwenAlpha-SOD_S2.4_零样本显著图基线协议.md`。

**后续非全监督主路线：** `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md`。该计划采用 Qwen 语义先验、SAM 边界提议、DINOv2 一致性、点/涂鸦弱标注和直接灰度学生网络；尚未启动训练。

---

## 5. 状态文件与权威性顺序

1. 用户最新指令（最高）。
2. **实际产物**（`reports/`、`runs/`、`monitor/` 日志）与本说明书。
3. `state/progress.yaml`、`state/done/*.json`、`state/history.jsonl`。
4. `paper/` 下的计划书与协议。

**已知滞后（必须知道，否则会被误导）：**

- `state/progress.yaml` 停在 **S2.3 nogo**（2026-10-29 19:53），未反映 S2.4 的探针/配对，也未反映 W 系列计划。
- `reports/NEXT_ACTION.md` 停在 **S2.2 passed**（2026-10-29 19:45），内容已过期。
- S2.4 目前**没有** done marker（它是诊断子步骤，W1 跑完才收口）。
- W0–W9 尚无任何 `state/done/W*.done.json`。
- 因此：**S2.4 的权威证据是 `reports/S2.4_*` 与本文件第 3.2 节；W 系列的权威定义是非全监督计划书，不是 progress.yaml。**

---

## 6. 未决事项（按优先级）

1. **跑 W1（100 张零样本开发基线）**：P2@1024，GPU 0，约 2 小时。当前唯一阻塞项。
2. **训练池剔除 110 张开发样本**：100 张 `vae_audit` + 10 张探针全部来自官方 train 划分，若"官方训练划分 = 主训练池"不排除它们，学生会见过开发样本，开发指标虚高。需在 manifest 写明剔除列表。
3. **GapClosure 分母定义**：`S_full_reference` 到底是"传统方法最优值"还是"自跑全监督参照"，两者可差 0.02–0.03，足以改变"接近/显著缩小"的措辞等级。
4. **主路线监督预算选择**（计划书 §12）：① 点监督 + 约 20% 难例涂鸦（推荐）；② 纯点监督；③ 严格零标注。决定后写入 `configs/weak_supervision/protocol_v1.yaml`。
5. **SAM 与 DINOv2 权重下载与校验**：远端尚无，需要一次等价于 S2.1 的下载 + 逐文件 SHA-256，先定仓库与固定 revision。
6. **产物命名统一**：W1 要求 `reports/W1_qwen_zero_shot_100.csv/json`，现有脚本输出 `S2.4_prompt_selection_*`。建议改脚本直接写 W1 名，避免两套文件名并存。
7. **编号体系明确**：W 系列是否取代 S3 及以后（S3 缓存 / S4 100-step 冒烟 / S5 64 张过拟合）。W4 已复用 48+16 过拟合子集，两套 done marker 并存会语义冲突。
8. **录入传统方法数值**：MCCNet / GeleNet / IPDiff 在三个数据集上的指标，必须来自原文表格，不能凭记忆。
9. **官方测试集评测计划**：100 张只是开发集，论文表格需在 EORSSD/ORSSD/ORSI-4199 官方测试集上一次性评测。
10. **补更新状态文件**：`progress.yaml`、`reports/NEXT_ACTION.md`。
11. **修正旧计划书冷启动命令**：仍指向 `state/progress.json` 与 `state/NEXT_ACTION.md`，实际是 `state/progress.yaml` 与 `reports/NEXT_ACTION.md`。
12. **删除旧工作区副本** `D:\桌面\CSCD\Qwenalpha-SOD`（两份并存有分叉风险，删除需用户同意）。
13. **把 `.codex-tmp/`、`archive/` 移出百度同步目录**。

---

## 7. 交接检查清单

**开工前：**

- [ ] 确认能 `ssh wangsiyi`（不通则先解决 VPN，不要改远端）
- [ ] 读本文件 §0、§3、§4
- [ ] `nvidia-smi` 确认目标卡空闲，确认 GPU 1 是否仍被他人占用
- [ ] 读目标步骤的既有 done marker，避免重复执行

**每个原子步骤：**

- [ ] 监控置 `ready` 并给出 `http://127.0.0.1:8765/`
- [ ] 再置 `running`，再用 `tmux` 启动（`qas_*` 命名）
- [ ] 经 `scripts/run_single_gpu.sh <物理卡号> -- ...` 启动，不绕过锁
- [ ] 结束时核对退出码、产物、显存回落
- [ ] 通过才写 `state/done/<STEP>.done.json`；失败保留日志，不写 marker
- [ ] 更新 `progress.yaml`、`history.jsonl`、监控页、本文件

**禁止：**

- 重复执行已有有效证据的步骤
- 触碰 GPU 1 上他人的 `train.py`（PID 39244）
- `device_map="auto"`、多卡、`sudo`、写入 `/data2/wangsiyi` 之外
- 事后修改门槛、剔除不利样本、只报赢家
- 未经确认改动 §4 冻结协议

---

## 8. 命令速查

```bash
# 冷启动汇总（路径已修正）
ssh wangsiyi 'cd /data2/wangsiyi/qwenalpha-sod && \
  sed -n 1,80p state/progress.yaml; \
  cat reports/NEXT_ACTION.md; \
  tail -n 20 state/history.jsonl; \
  ls -1 state/done | tail -5; \
  tmux ls 2>/dev/null; \
  nvidia-smi --query-gpu=index,memory.used --format=csv,noheader'
```

```bash
# 单卡运行模板（物理 GPU 0）
cd /data2/wangsiyi/qwenalpha-sod
bash scripts/run_single_gpu.sh 0 -- /data2/wangsiyi/envs/qwenalpha-sod/bin/python scripts/<script>.py
```

```bash
# 零样本基线（待跑）
bash scripts/run_single_gpu.sh 0 -- /data2/wangsiyi/envs/qwenalpha-sod/bin/python \
  scripts/s2_4_probe.py --output-resolution 1024 --variants P2 --tag baseline100
```

```powershell
# 本地同步（先注入 git 与代理）
$env:Path = 'C:\Users\28446\.cache\codex-runtimes\codex-primary-runtime\dependencies\native\git\cmd;' + $env:Path
$env:HTTPS_PROXY = 'http://127.0.0.1:7890'; $env:HTTP_PROXY = $env:HTTPS_PROXY
$env:GIT_TERMINAL_PROMPT = '0'; $env:GCM_INTERACTIVE = 'Never'
powershell -NoProfile -ExecutionPolicy Bypass -File ".\workspace-tools\sync-qwenalpha-artifacts-once.ps1"
```

---

## 9. 编号对照

| 编号 | 内容 | 出处 |
|---|---|---|
| S2.1 | 模型下载与校验 | 分步计划书 B2 |
| S2.2 | 8 张 CPU-offload RGBA 推理 | 分步计划书 B2 |
| S2.3 | 100 张 VAE Alpha 上限审计 | 分步计划书 B2 |
| S2.4 | 零样本显著图基线（模板选择 + 分辨率配对） | 本轮新增，已写入 `S2.4 协议` |
| **W0** | 协议与 GT 数据防火墙 | 非全监督主路线计划书 §7 |
| **W1** | 100 张零样本开发基线（= S2.4 的正式版） | 同上 |
| **W2** | 多教师伪标签原型（Qwen / SAM / DINOv2 逐级叠加） | 同上 |
| **W3** | 弱标注 100 张与标注工具验收 | 同上 |
| **W4** | 64 张过拟合与损失验证 | 同上 |
| **W5** | 500 张主路线训练 | 同上 |
| **W6** | 1,000 张与全训练池弱监督 | 同上 |
| **W7** | 两轮以内自训练 | 同上 |
| **W8** | 冻结、官方测试与统计 | 同上 |
| **W9** | 消融与论文证据链 | 同上 |

**两套编号的关系：** S 系列是已完成阶段（S0–S2.3）；W 系列是新的主路线。S3（缓存）、S4（100-step 冒烟）、S5（64 张过拟合）在 W 路线中由 W4/W5 以弱监督形式承接；是否正式作废待用户确认（见 §6 第 7 项）。

S2.4 源于用户的论文构想：先测量"不做任何处理时 Qwen 生成显著图的能力"，再谈提升。W1 是它的正式化版本，产物命名也改为 `W1_*`。

---

## 10. 一页结论

- 表示层没坏到不可用（上限 0.9643），但**不是无损**（软边、小目标弱），因此"原生 RGBA 无损"这一卖点不成立，应改为"生成式范式 + 可控提升"。
- 零样本能出图，但**质量有限**（Sα 0.59），距传统方法约 0.3。
- 这 0.3 的差距就是论文要证明的"创新处理"的价值所在。
- 主路线已定为**非全监督**（Qwen + SAM + DINOv2 + 点/涂鸦 + 直接灰度学生），不再走全监督微调；GT 只做一次性评测。
- 一切比较都必须同集、同口径、配对统计；基线只测一次。

---

## 11. 维护规则与变更日志

### 11.1 谁在什么时候更新

**任何 agent**（Harness、Codex 或其他人）**每完成一个原子步骤**，必须：

1. 在 §11.2 变更日志追加一行（时间、agent、步骤、一句话改动）；
2. 更新 §0 速览的"现在在哪一步 / 下一步唯一动作"；
3. 在 §2 进度表增删行（新增完成项或状态变化）；
4. 调整 §6 未决事项（完成项移除、新发现项按优先级插入）；
5. 递增头部版本号与更新时间。

**这属于步骤收尾的一部分**，与写 `state/done/<STEP>.done.json`、更新 `progress.yaml`、更新监控页同等必要。只更新远端状态而不更新本文件，视为步骤未收尾。

### 11.2 变更日志

| 时间（+08:00） | Agent | 版本 | 改动 |
|---|---|---|---|
| 2026-10-30 18:40 | Harness | v1.0 | 初版：环境坑、S0–S2.4 进度、冻结协议、未决事项 |
| 2026-10-30 19:00 | Harness | v1.1 | 纳入非全监督主路线计划书（W0–W9）；新增训练池剔除 110 张、GapClosure 分母、监督预算、SAM/DINOv2 下载、W1 产物命名、S/W 编号关系共 6 项未决；编号对照扩为 S+W 两套；加入本维护规则 |

### 11.3 冲突处理

- 本文件与远端 `progress.yaml` / `NEXT_ACTION.md` 冲突时，以**实际产物**为准，并立即回写两者。
- 本文件与计划书冲突时，以**用户最新指令**为准；若无新指令，以计划书为准并在此登记差异。
- 禁止为了"让文档好看"修改已完成步骤的结论；历史结论只增不改，修正以新增日志行说明。
