# QwenAlpha-SOD：任意节点移交 Harness 续执行说明书

**版本：** v2.0

**更新日期：** 2026-10-29

**本地工作区：** `D:\桌面\CSCD\Qwenalpha-SOD`

**SSH 别名：** `wangsiyi`

**远端项目根目录：** `/data2/wangsiyi/qwenalpha-sod`

**远端唯一允许写入根目录：** `/data2/wangsiyi`
**唯一 GitHub 同步仓库：** `https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git`

## 1. 用途与接管原则

本说明书用于在任意阶段、任意分片、任意 checkpoint 或任意实验种子完成后，把尚未完成的工作交给 Harness 继续执行。它不是一次性提示词，也不表示 Harness 已经获得自动执行权限；每次接管仍须由用户明确发起。

Harness 接管时必须遵循以下优先级：

1. 用户在当前会话中的最新要求，以及仓库根目录 `AGENTS.md`。
2. 远端状态文件、完成标记、日志和实际产物。
3. 本文中的 `当前进度快照` 与 `NEXT_ACTION`。
4. `paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md`。
5. 历史聊天记录和旧版计划。

若文档与远端状态冲突，以远端可验证事实为准；先修正文档，不得为了符合文档而重跑已通过任务。

## 2. 当前进度快照

### 2.1 已完成

- S0.1—S0.5：通过，环境、路径、代码基线、GPU 边界和状态机制已建立。
- S1.1—S1.4：通过，数据资产、派生集、审计集和 overfit 子集已冻结。
- S1 核心事实：
  - 训练配对数：6,999。
  - manifest 与稳定哈希已生成；manifest 哈希摘要为 `ba11c6c3…1446a0`。
  - 发现 2 组 train/test 精确重复，以及 158 组跨数据源同源样本；已保留并披露，不得静默删除。
  - RGBA 派生集：4,000。
  - 人工审计集：100。
  - overfit 子集：训练 48、验证 16。

### 2.2 当前正在执行：S2.1 模型下载与本地校验

- 官方模型：`Qwen/Qwen-Image-2.1`。
- Hugging Face 固定提交：`d26bb61231c349cf6b7896fa83353113880e1ba3`。
- 下载通道：ModelScope，revision 为 `master`；完成后必须用逐文件 SHA-256 固化本次实际快照。
- 预期清单：35 个条目，文件总字节数 `33,134,789,600`。
- 模型目录：`/data2/wangsiyi/models/Qwen-Image-2.1`。
- 缓存目录：`/data2/wangsiyi/cache/modelscope`。
- 后台会话：`tmux` 会话 `qas_s2_1_download`。
- 下载脚本：`/data2/wangsiyi/qwenalpha-sod/monitor/download_s2_1.py`。
- 日志：`/data2/wangsiyi/qwenalpha-sod/monitor/s2_1.log`。
- 监控状态：`/data2/wangsiyi/qwenalpha-sod/monitor/status.json`。
- 当前步骤不占用 GPU。

接管者必须先读取真实状态，不得依据本段文字直接断言 S2.1 仍在下载或已经完成。

## 3. 永久边界

### 3.1 文件系统和用户隔离

- 所有远端新建、修改、缓存、下载、日志、临时文件和产物只能位于 `/data2/wangsiyi`。
- 禁止 `sudo`，禁止修改系统包、系统服务、其他用户目录或共享配置。
- 写入前解析真实路径；任何目标若越出 `/data2/wangsiyi`，立即停止。
- 不删除未知文件，不清理其他任务缓存，不修改其他用户进程。

### 3.2 GPU 边界

- 机器为共享单卡 RTX 3090；只有明确进入 GPU 步骤后才允许占用 GPU。
- 必须通过项目既有 GPU 包装器/锁机制启动，显存上限不超过 22 GB。
- 禁止 `device_map=auto`、多 GPU、抢占或终止他人进程。
- 启动前检查 GPU；资源不满足时记录为 `paused`，不得强行运行。

### 3.3 科研与发布边界

- 不得伪造指标、日志、样本、时间戳或完成标记。
- 未通过阶段门禁不得进入下一阶段。
- 远端状态与原始日志默认不公开；只有计划规定的论文、报告、图表和必要代码进入公开同步范围。
- Harness 只执行用户指定的一个原子步骤；通过后停止并汇报，不自动扩展到下一步。

## 4. 接管时的最低成本冷启动

先执行一次只读汇总，避免逐文件、逐命令反复查询：

```bash
ssh wangsiyi 'cd /data2/wangsiyi/qwenalpha-sod && \
  printf "=== progress ===\n" && sed -n "1,220p" state/progress.json 2>/dev/null || true; \
  printf "=== next ===\n" && cat state/NEXT_ACTION.md 2>/dev/null || true; \
  printf "=== active ===\n" && cat state/active_run.json 2>/dev/null || true; \
  printf "=== history tail ===\n" && tail -n 20 state/history.jsonl 2>/dev/null || true; \
  printf "=== latest done ===\n" && find state/done -maxdepth 1 -type f -printf "%T@ %p\n" 2>/dev/null | sort -nr | head -n 5; \
  printf "=== tmux ===\n" && tmux ls 2>/dev/null || true'
```

然后提交一份接管摘要，至少包括：

- 最后一个通过的步骤及其证据。
- 当前运行中、暂停或失败的步骤。
- 当前安全恢复点。
- 用户指定步骤的前置条件是否满足。
- 计划执行的唯一原子动作、验收标准与预计产物。
- 将使用的监控页地址。

## 5. 每步执行前的 HTML 实时监控

监控文件位于：

- `/data2/wangsiyi/qwenalpha-sod/monitor/index.html`
- `/data2/wangsiyi/qwenalpha-sod/monitor/server.py`
- `/data2/wangsiyi/qwenalpha-sod/monitor/status.json`
- `/data2/wangsiyi/qwenalpha-sod/monitor/update_status.py`

远端服务只监听 `127.0.0.1:8765`。本地需要时建立 SSH 隧道：

```powershell
ssh -N -L 127.0.0.1:8765:127.0.0.1:8765 wangsiyi
```

监控地址：`http://127.0.0.1:8765/`

每个原子步骤必须按以下顺序开始：

1. 把 `status.json` 原子更新为 `ready`，写明步骤、命令摘要、日志和验收条件。
2. 向用户给出监控 URL。
3. 再把状态更新为 `running` 并启动任务。
4. 页面只读取轻量 JSON 和日志尾部；默认 5 秒刷新，不扫描整个模型或数据目录。
5. 长任务优先等待明确事件或退出码，避免高频状态轮询。

如果监控服务或隧道中断，只恢复监控，不得重启仍在运行的科研任务。

## 6. 统一原子步骤协议

每一步都必须满足以下闭环：

1. **Preflight**：验证路径、磁盘、依赖、前置 done marker；GPU 步骤额外验证锁和显存。
2. **登记**：原子写入 `state/active_run.json`，向 `state/history.jsonl` 追加 `running` 事件，并更新监控页。
3. **执行**：长任务进入命名 `tmux` 会话；分片任务记录 shard、seed、checkpoint 和输出目录。
4. **验收**：检查退出码、产物完整性、哈希/指标、日志错误和科学门禁。
5. **提交状态**：验收通过后才创建 `state/done/<STEP>.done.json`；失败或暂停不得创建 done marker。
6. **收尾**：清空或归档 active run，追加历史记录，更新 `progress.json`、`NEXT_ACTION.md`、监控页和本地文档。
7. **停止**：只汇报本步骤，不自动执行下一步。

状态写入必须采用“临时文件 + 原子替换”，防止断电或中断留下半个 JSON。

## 7. S2.1 当前接管与恢复方法

### 7.1 一次性检查

```bash
ssh wangsiyi 'cd /data2/wangsiyi/qwenalpha-sod && \
  cat monitor/status.json 2>/dev/null; \
  tmux has-session -t qas_s2_1_download 2>/dev/null && echo TMUX_RUNNING || echo TMUX_NOT_RUNNING; \
  tail -n 80 monitor/s2_1.log 2>/dev/null'
```

### 7.2 恢复决策

- `tmux` 仍运行：不得启动第二个下载；继续事件式等待。
- 日志出现 `S2_1_EVENT download_complete`：进入独立校验步骤，不重复下载。
- 日志出现 `download_failed`：记录错误；确认目标与缓存仍在允许目录后，使用相同目录恢复下载。
- `tmux` 消失但没有结束事件：先核对进程、文件和退出信息；状态记为 `paused` 或 `failed`，不得猜测为成功。

下载固定参数：

- repo：`Qwen/Qwen-Image-2.1`
- revision：`master`
- `local_dir`：`/data2/wangsiyi/models/Qwen-Image-2.1`
- `cache_dir`：`/data2/wangsiyi/cache/modelscope`
- 并发：2

### 7.3 S2.1 验收标准

- 预期文件全部存在，预期总字节数匹配 `33,134,789,600`。
- `model_index.json`、各组件 config/index JSON 可解析。
- text encoder、transformer、VAE、scheduler、processor 等组件齐全。
- 生成逐文件 SHA-256 清单，并记录 HF 固定提交与 ModelScope revision。
- `diffusers` 能从本地目录读取 pipeline 配置；本步骤不加载大权重到 GPU。
- 所有真实路径仍位于 `/data2/wangsiyi`。

预期产物：

- `reports/S2.1_model_download_manifest.json`
- `reports/S2.1_model_files.sha256`
- `state/done/S2.1.done.json`
- 本地 `paper/QwenAlpha-SOD_S2.1_模型下载与校验报告.md`

S2.1 通过后将 `NEXT_ACTION` 指向 S2.2，但不得自动启动 GPU 测试。

## 8. 后续阶段的任意节点接管索引

| 阶段 | 允许接管的安全点 | 进入条件 | 必须守住的门禁 |
|---|---|---|---|
| S2.2—S2.x | 单次探测、单个组件验证结束 | S2.1 done | 单卡、显存 ≤22 GB、先 overfit/最小配置 |
| S3 | 单个基线或 seed 完成 | S2 全部门禁通过 | 配置、seed、checkpoint、样本和指标可追溯 |
| S4 | 单个 alpha/方法变体完成 | 对应 baseline 可复现 | 不改变未声明变量，不挑选性丢弃失败运行 |
| S5 | 单项消融完成 | 主方法已冻结 | 每项消融独立记录，保持对照公平 |
| S6 | 单个 benchmark/seed 完成 | 评测脚本冻结 | 数据划分不变，失败样本保留并披露 |
| S7 | 单张表/图生成完成 | 结果清单与哈希齐全 | 图表可从原始结果重建 |
| S8 | 单个论文小节完成 | 对应实验已通过 | 结论不得超出证据，数字与报告一致 |
| S9 | 单次公开同步前 | 全部发布检查通过 | 只同步允许公开的文件，执行既定一次性同步 |

更细的命令、产物和验收条件以详细步骤计划书及远端 `NEXT_ACTION.md` 为准。

## 9. 暂停、崩溃与恢复

- 长任务必须可从最近 shard/checkpoint 恢复；不得因为聊天中断就重跑已完成部分。
- 发现未知输出时先隔离并记录，不覆盖、不删除。
- 失败时保留日志、配置、退出码、环境摘要和已完成分片。
- 恢复前比对 run ID、参数哈希、输入哈希和输出路径；不一致则新建运行目录。
- 同一原子步骤最多只有一个 active run；发现重复进程时停止新启动动作并报告，不得随意杀进程。
- 状态无法确定时使用 `paused`，补充只读证据后再决定恢复或失败。

## 10. 通用 Harness 接管提示词

```text
请接管 QwenAlpha-SOD 的后续执行。

本地工作区：D:\桌面\CSCD\Qwenalpha-SOD
SSH：ssh wangsiyi
远端项目：/data2/wangsiyi/qwenalpha-sod
远端所有写入只能发生在 /data2/wangsiyi；禁止 sudo，禁止影响其他用户、进程和目录。

先完整阅读：
1. paper/QwenAlpha-SOD_Codex额度不足时Harness续执行计划书.md
2. paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md
3. 仓库根目录 AGENTS.md（文件名使用复数形式）

然后用说明书中的“最低成本冷启动”命令读取远端真实状态。不要仅依据文档推断进度，不要重复已通过任务。

先向我提交接管摘要：最后通过步骤、当前运行/暂停任务、安全恢复点、前置条件、唯一计划动作、验收标准、监控 URL。每一步启动前必须先更新并展示 http://127.0.0.1:8765/；长任务使用 tmux 和事件式等待，减少无意义轮询与 token 消耗。

本次只执行我指定的原子步骤：<在这里填写步骤或“从 NEXT_ACTION 继续一个步骤”>。
通过后更新远端状态、done marker、NEXT_ACTION、监控页和相关本地文档，然后停止并汇报。不得自动进入下一步。
```

## 11. 最小交接报告模板

```text
步骤：
状态：passed / failed / paused / running
run_id：
安全恢复点：
监控页：http://127.0.0.1:8765/
远端日志：
关键配置/哈希：
验收结果：
产物：
资源使用：
对其他用户的影响：无 / 说明
NEXT_ACTION：
```

## 12. 接管前检查表

- [ ] 已读取用户最新要求、仓库根目录 `AGENTS.md` 和两份计划书。
- [ ] 已通过一次冷启动命令核对远端状态。
- [ ] 已确认所有写入目标位于 `/data2/wangsiyi`。
- [ ] 已确认不会影响其他用户或共享资源。
- [ ] 已给出监控 URL，并在执行前将监控状态置为 `ready`。
- [ ] 已明确唯一原子步骤、验收标准和停止点。
- [ ] 已确认现有任务不会被重复启动。
- [ ] GPU 步骤已通过锁、显存和单卡检查；非 GPU 步骤未占用 GPU。
- [ ] 已定义失败、暂停和恢复策略。
- [ ] 完成后将更新状态、done marker、NEXT_ACTION、监控页与本地文档。

---

本手册的设计目标是：移交发生在任何节点时，Harness 都先从可验证状态恢复上下文，只执行一个有明确边界的原子步骤，并留下下一位执行者可以继续使用的完整证据链。
