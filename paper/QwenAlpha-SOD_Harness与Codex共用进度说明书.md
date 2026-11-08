# QwenAlpha-SOD 共用进度说明书（Harness / Codex 通用）

**版本：** v9.31

**更新时间：** 2026-11-08 11:37（Asia/Shanghai）

**维护规则：** 本文件是所有 agent 的**强制活文档**。任何 `running / paused / failed / nogo / passed` 状态、下一步、冻结协议或安全恢复点发生变化时，必须在同一原子步骤内更新本文件。完成步骤时须在 §11 变更日志追加一行，并同步更新 §0 速览、§2 进度表、§6 未决事项。长任务逐文件进度由 HTML 监控和日志承担，不为每个进度点提交本文。**未更新本文的步骤视为未收尾。**

**用途：** 供任意 agent（DeepSeek Harness、Codex 或后续接手者）在**不读历史聊天**的前提下恢复到可执行状态。本文件与远端状态文件互为校验；两者冲突时以**实际产物**为准，并回写本文件。

**当前计划版本：** `paper/QwenAlpha-SOD_Qwen弱监督修正实验与Harness共用计划书.md` v1.0（R0–R6）。当前目标是训练冻结 Qwen 输出的弱监督修正器。Qwen 本体 LoRA 和 W0–W9 计划保留为历史路线。

---

## 0. 三十秒速览

| 问题 | 答案 |
|---|---|
| 项目现在在哪一步 | R4 两组各500步均通过验收，10个检查点已核验，开发集尚未评分 |
| 最重要的一组数 | 500×4类产物齐全，2000/2000哈希一致，500/500 NPZ重开与点/涂鸦像素一致性通过；非空SAM predicted-IoU均值0.8920，弱提示一致性均值0.9759，20空场景；dense-GT访问0、模拟标注0、峰值5180 MiB |
| 下一步唯一动作 | R5：用固定第500步检查点生成开发集预测，按统一评测器比较A/B/C/D及冻结门槛 |
| 绝不能做的事 | 重跑已有 done marker 的步骤；碰他人 GPU 进程；未经确认改冻结协议；让训练池混入 100 张开发样本 |

---

## 1. 环境与路径（含已验证的坑）

| 项 | 值 |
|---|---|
| 本地工作区 | `D:\BaiduSyncdisk\CSCD\Qwenalpha-SOD`；旧路径当前只是指向新目录的兼容 Junction，不是第二份仓库 |
| SSH | `ssh wangsiyi`（`10.101.1.254:19059`，需实验室网络或 VPN） |
| 远端项目 | `/data2/wangsiyi/qwenalpha-sod` |
| 远端允许写入根 | `/data2/wangsiyi`（该账号 HOME 也在此目录内） |
| 模型 | `/data2/wangsiyi/models/Qwen-Image-2.1`，29 文件 / 33,134,789,600 字节，HF `d26bb612…`，ModelScope `master` |
| 数据集 | `/data2/wangsiyi/datasets/{raw,derived}`（注意不在项目目录下） |
| Python | `/data2/wangsiyi/envs/qwenalpha-sod/bin/python` |
| 监控 | 远端 `127.0.0.1:8765`，本地隧道：`ssh -N -L 127.0.0.1:8765:127.0.0.1:8765 wangsiyi` |

### 1.1 已知的坑（每条都踩过）

2026-11-06补充：远端监控服务也可能退出，本次重启该服务并重建隧道后HTTP200；不要因此重启训练。C1页面按浏览器请求读取训练日志末尾更新进度，无新增后台轮询；退出后显示待验收或失败，不自动宣布科学验收通过。

1. **PowerShell 会破坏远端命令的引号**。含引号、中文、`|`、多行的命令一律用 base64 走 stdin：
   `$b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($script))`，然后 `ssh wangsiyi "echo $b64 | base64 -d | bash"`。
2. **`git` 不在 PATH**。用 Codex 自带的：
   `C:\Users\28446\.cache\codex-runtimes\codex-primary-runtime\dependencies\native\git\cmd\git.exe`。
3. **GitHub 需要代理**。Clash 在 `127.0.0.1:7890`，直连 443 会超时；跑同步前设 `HTTPS_PROXY=http://127.0.0.1:7890`、`HTTP_PROXY` 同值、`GIT_TERMINAL_PROMPT=0`、`GCM_INTERACTIVE=Never`。凭据由 `manager` 助手缓存，无需交互。
4. **本地工作区在百度同步盘内**：不要把权重、数据集、runs、checkpoint 放进来；Git 操作在本机进行，不要依赖网盘同步 `.git`。
5. **GPU 占用情况只是瞬时态**：不得沿用本文历史空闲卡结论；每个 GPU 原子步启动前必须重做预检，禁止终止、挤占或共享他人进程。
6. **监控隧道会随本地断网/重启失效**（浏览器报 `ERR_CONNECTION_REFUSED`）。远端服务不受影响，重建隧道即可，**不要重启远端监控进程以外的任何任务**。
7. **`curl` 在远端不存在**，用 Python `urllib` 或 `wget`。

---

## 2. 步骤进度与证据

| 步骤 | 结论 | 关键证据 |
|---|---|---|
| S0.2–S0.5 | 工程骨架、Python 环境、单卡包装器、暂停恢复测试 | `state/done/S0.*.done.json` |
| S1.1–S1.4 | 数据盘点、manifest 与重复审计、RGBA 目标、难例子集 | `state/done/S1.*.done.json` |
| S2.1 | 模型下载与逐文件 SHA-256 校验 | `reports/S2.1_model_download_manifest.json`、`state/done/S2.1.done.json` |
| S2.2 | 8 张原生 RGBA 推理 | `reports/S2.2_rgba_probe.json`、`state/done/S2.2.done.json` |
| S2.3 | 原生 VAE Alpha 上限审计 → **No-go** | `reports/S2_VAE_NOGO.md`、`reports/S2.3_vae_audit*.json` |
| S2.4（探针） | 提示词模板 P2 胜出；1024 与 352 配对完成 | `reports/S2.4_prompt_selection_352.json`、`reports/S2.4_prompt_selection_1024p2.json` |
| 计划（非全监督主路线） | W0–W9 详细计划书 v1.0 已入仓 | `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md`（commit `1840208`） |
| W0 | 非全监督协议、数据角色与 dense-GT 防火墙冻结 | `reports/W0_protocol_and_gt_firewall.json`、`state/done/W0.done.json` |
| W1 | 100张 P2@1024 零训练开发基线验收通过；100/100、失败0 | `reports/W1_acceptance.json`、`reports/W1_grayscale_predictions.sha256`、`state/done/W1.done.json` |
| W2.0 | SAM ViT-H / DINOv2 ViT-B/14 官方权重冻结、下载与逐文件 SHA-256 校验通过；不占 GPU | `reports/W2.0_teacher_models.json`、`reports/W2.0_teacher_models.sha256`、`state/done/W2.0.done.json` |
| W2.1 | 单图 SAM 自动候选 + DINOv2 特征冒烟通过；SAM 11个候选，patch shape `[1,256,768]`，峰值5491 MiB | `reports/W2.1_smoke.json`、`state/done/W2.1.done.json`、`monitor/w2_1.log` |
| W2.2 | 100张 SAM-only 候选生成通过；100/100 NPZ、10/10分片、失败0，共1988个候选，峰值5493 MiB | `reports/W2.2_sam_candidates.json`、`reports/W2.2_sam_candidates.sha256`、`state/done/W2.2.done.json` |
| W2.3 | 冻结无GT规则的 SAM-only 聚合与100张开发评测通过；Sα 0.2957 / MAE 0.3086 / Fmax 0.6477 | `reports/W2.3_sam_only_baseline.json`、`reports/W2.3_sam_only_metrics.csv`、`state/done/W2.3.done.json` |
| W2.4 | 12个预声明 Qwen+SAM 变体执行与验收通过；选中 `t10_l25`，但ΔSα +0.0075、MAE/Boundary-F均略退化，冻结科学 Go 门槛未通过 | `reports/W2.4_qwen_sam_fusion.json`、`reports/W2.4_qwen_sam_grid_metrics.csv`、`state/done/W2.4.done.json` |
| W2.5 | 100张 DINOv2 特征与3个预声明变体验收通过；选中 `d75`，相比 W2.4 有小幅改善，但相比 W1 仅ΔSα +0.0120且 MAE/Boundary-F仍略差，冻结 W2 Go 门槛失败 | `reports/W2.5_qwen_sam_dino_fusion.json`、`reports/W2.5_selected_predictions.sha256`、`state/done/W2.5.done.json` |
| W2.6 | GT-assisted 候选/点提示上限诊断通过；贪心 oracle IoU 0.7293（small 0.6503），模拟点提示 IoU 0.7532（small 0.7891），两项均过冻结门槛，推荐真实弱点阶段；该结果不是无监督成绩 | `reports/W2.6_sam_oracle_point_diagnostic.json`、`reports/W2.6_predictions.sha256`、`state/done/W2.6.done.json` |
| W3 | **passed_with_persistent_sam_failures_excluded**：100张真实点标注全部保留；96张使用点+三值伪标签，4张持续失败样本降级为point-only并由机器清单阻断伪标签损失。最终复核4/4均revise但问题已按排除策略闭环；W4未启动 | `reports/W3_acceptance.json`、`manifests/weak_supervision/w3_training_eligibility_v1.csv`、`manifests/weak_supervision/w3_persistent_sam_failures_v1.csv`、`state/done/W3.done.json` |
| W4 | **passed**：48 train + 16 val、point-only=0；step 1/10/100/500检查点均可重载且哈希一致。最终弱点满足率1.0000，验证Sα 0.6325（较W1 +0.1013），打乱/纯黑反事实分别下降0.5150/0.6319，峰值456 MiB；训练dense-GT访问0，独立评估读取16张GT | `reports/W4_acceptance.json`、`reports/W4_resume_validation.json`、`reports/W4_artifacts.sha256`、`state/done/W4.done.json` |
| W5（C1 nogo） | 14 checkpoint/1400 PNG/1442 hash通过；3500步早停核实。最佳step2000 Sα 0.817936、MAE 0.021741、small 0.603760；较基线总体+0.001699<0.005、small+0.006498<0.02，MAE与单集退化门槛通过，故C1 rejected，保留seed0 | `reports/W5_C1_acceptance.json`、`reports/W5_C1_final_artifacts.sha256` |
| W5 C2（nogo） | GPU1训练退出0，2750步早停、最佳1250；11 checkpoint/1100 PNG/1133 hash及GT防火墙通过。总体Sα 0.816812，较基线+0.000575<0.005；small 0.589992，较基线−0.007270<+0.02；MAE改善0.000877、单集退化门槛通过。C2 rejected，保留seed0 | `reports/W5_C2_acceptance.json`、`reports/W5_C2_final_artifacts.sha256` |
| W5复盘（passed） | seed0 checkpoint1250 SHA-256复核；C1/C2两份最终哈希清单通过。冻结保留模型，整理导师决策材料及下阶段选项；没有新训练或官方测试 | `reports/W5_route_review.json`、`state/done/W5.ROUTE_REVIEW.done.json`、`paper/QwenAlpha-SOD_W5阶段复盘与路线决策_2026-11-07.md` |
| seed0单模型官方测试（passed） | 2999/2999灰度预测、30分片、3029 hash通过；合并Sα 0.723305/MAE 0.044214。EORSSD 0.660627、ORSI-4199 0.735621、ORSSD 0.775915；单seed，仅锁定模型测试，非完整W8 | `reports/W5_seed0_official_test_acceptance.json`、`reports/W5_seed0_official_test_artifacts.sha256`、`state/done/W5.SEED0_TEST.done.json` |
| seed0测试后描述审计（passed） | 2999张按GT面积分组：≤2% 806张Sα0.558276；2%–20% 1381张0.768502；>20% 790张0.804962；空场景22张约1.0。跨清单相同图像hash 42组/84条。仅描述，不做模型选择 | `reports/W5_seed0_official_test_descriptive_audit.json`、`state/done/W5.SEED0_TEST_AUDIT.done.json` |
| 修订对照协议（passed） | B500 500/500 train GT登记可用于独立FS参考；训练池与官方test有2条精确图像重合，三个test清单跨源重复42组。全监督参照尚未实现/训练；GapClosure尚不可计算 | `reports/W5_revised_comparison_preflight.json`、`state/done/W5.REVISED_PROTOCOL.done.json`、`paper/QwenAlpha-SOD_修订后对照与评测协议_2026-11-07.md` |
| FS参考数据/损失预检（passed） | 500/500 train GT精确允许清单，500条独立FS访问审计；350增强掩码值域/尺寸、BCE+Dice有限反传通过。独立FS配置hash `95bf3959…f41f6`；尚无训练/评测链路和分数 | `reports/W5_FS_reference_CPU_preflight.json`、`reports/W5_FS_reference_preflight_artifacts.sha256`、`state/done/W5.FS_REF_PREFLIGHT.done.json` |
| FS参考执行链路（passed，未训练） | 独立train/evaluate/orchestrate及单GPU tmux脚本接入；CPU数据确定性、项目checkpoint恢复与错误hash拒绝通过。为绑定实现hash，最终配置修订为`94f620a6…2e3058`；无GPU run | `reports/W5_FS_reference_execution_preflight.json`、`reports/W5_FS_reference_execution_artifacts.sha256`、`state/done/W5.FS_REF_EXEC_PREFLIGHT.done.json` |
| FS参考开发集（passed） | GPU1训练退出0、3750步早停，最佳2250步dev100 Sα0.866471/MAE0.014728；15 checkpoint/1500预测/1545 hash通过，FS专用GT审计30000条均限B500。与弱监督seed0 Sα差+0.050234，FS官方test未运行 | `reports/W5_FS_reference_acceptance.json`、`reports/W5_FS_reference_final_artifacts.sha256`、`state/done/W5.FS_REF_DEV.done.json` |
| FS参考事后官方测试（passed） | FS Sα0.750656，弱监督同2999图0.723305；分集FS 0.722935/0.751967/0.819407，2999预测/3029 hash通过。排除2条train/test重合后差0.027360；按相同图像去重后差0.027261。非盲测、单seed | `reports/W5_FS_reference_official_test_acceptance.json`、`reports/W5_FS_reference_paired_sensitivity.json`、`state/done/W5.FS_REF_TEST.done.json` |
| 同集零训练参照（paused） | 用户要求暂停；529/2999行CSV与529张PNG一致，tmux停止、exit130、GPU进程退出；可按冻结配置从CSV续跑。未完整验收，不计算GapClosure | `reports/W5_zero_shot_official_test_freeze.json`、`reports/W5_zero_shot_official_test.csv`、`monitor/w5_zero_shot_official_test.exit` |
| Qwen LoRA小测试（CPU结构通过，GPU待测） | 本地Qwen-Image-2.1生成Transformer为32层；rank4注意力q/k/v/out LoRA已在meta模型挂载，256个可训练张量、4,194,304参数；PEFT/Diffusers/Accelerate可用，500张训练图路径可读。尚未真实加载训练、反传或验证22GB峰值，无新Sα | `reports/W5_Qwen_LoRA_small_test_preflight.json` |
| Q0 Qwen本体LoRA计划与交接（passed） | 冻结Q0–Q7分步、弱监督主实验定义、旧路线与新路线区别、状态事务、双Agent接管及测试后设计披露；未启动新GPU训练 | `paper/QwenAlpha-SOD_Qwen本体LoRA显著图适配分步与双Agent交接计划_2026-11-07.md`、移交说明书v2.1 |
| Q1 弱监督目标图审计（passed） | 409份中排除1张与官方测试图同哈希的训练图，408份通过来源哈希、尺寸、值域和点／涂鸦像素检查。未确定区域用原SAM候选填充，另存不确定掩码。抽查5份，其中1份暂缓，扩量前仍需复核。CPU执行，GT掩码读取0 | `reports/Q1_weak_targets.json`、`state/done/Q1.done.json` |
| Q2 单图 Qwen LoRA（passed） | ORSI-4199/train/490，512px，rank4，GPU1。退出码0，流匹配损失0.024789，LoRA梯度L2为0.002347，256个LoRA张量更新，冻结参数无梯度且版本未变，适配器重载逐张量一致。PyTorch保留显存峰值17180 MiB，阶段采样整卡显存最高16034 MiB，GPU1已释放 | `reports/Q2_lora_smoke_acceptance.json`、`state/done/Q2.done.json`、`runs/weak_supervision/Q2-smoke-v1/result.json` |
| Q3 四图训练诊断（nogo） | GPU1退出0，64次更新及8张预测均验收。三张非空图弱标签Sα分别0.3222→0.3743、0.0510→0.0491、0.5995→0.5848。仅1/3可见改善，湖泊图轮廓可见但亮度阈值0.5下无前景。PyTorch保留显存峰值18298 MiB，GPU已释放 | `reports/Q3_overfit_acceptance.json`、`runs/weak_supervision/Q3-overfit-v1/review_metrics.json` |
| Q3 失败诊断（passed） | 8张VAE重建验收，弱目标Sα最低0.9746。湖泊前景平均亮度：原模型0.116、LoRA 0.113、VAE重建0.991。湖泊重建前景召回99.96%，原模型与LoRA均为0%。无训练和GT掩码读取，PyTorch保留显存峰值2808 MiB | `reports/Q3_target_diagnosis_acceptance.json`、`state/done/Q3_DIAGNOSIS.done.json` |
| Q3 P3 对照重试（nogo） | 仅提示词不同，64次更新及8张预测均核验，退出0。P3原模型→LoRA的非空均值0.0337→0.3251，P2原模型→LoRA为0.3242→0.3361。小船图明显改善，另一图留有紫色条纹，湖泊前景召回0.052%。仅1/3通过视觉判断。PyTorch保留显存峰值18304 MiB，GPU1已释放 | `reports/Q3_p3_acceptance.json`、`runs/weak_supervision/Q3-p3-retry-v1/review_metrics.json` |
| Q3 路线复核（passed） | CPU核验16张已有预测和四张弱目标。P3湖泊的确定前景仍比背景暗，VAE重建能表示白色目标。现有配置保持nogo，只立单图损失可行性验证 | `reports/Q3_route_review_acceptance.json`、`state/done/Q3_ROUTE_REVIEW.done.json` |
| Q3 单图辅助损失验证（passed） | 湖泊弱目标，512px，P3，sigma 0.25。VAE和DiT分阶段反传，DiT重算逐值相等。辅助梯度L2为8.35e−5，乘0.1后约为流匹配梯度L2的0.37%。排除6918个不确定像素，零权重更新、零GT读取。PyTorch保留显存峰值17182 MiB，GPU已释放 | `reports/Q3_loss_smoke_acceptance.json`、`state/done/Q3_LOSS_SMOKE.done.json` |
| Q3 四图辅助损失对照（nogo） | GPU1退出0，64步和四组对照验收，首步流匹配损失、逐步sigma与生成种子匹配。P3流匹配0.3251→辅助损失0.2999，低于P2 LoRA 0.3361。0/3有可见改善，EORSSD/1158预测前景1.41%→81.71%，空场景假阳性略升。PyTorch保留显存峰值18312 MiB，GPU1已释放 | `reports/Q3_loss_ab_acceptance.json`、`runs/weak_supervision/Q3-loss-ab-v1/flow_vs_aux.jpg` |
| Q3 收尾复核（passed） | 三轮配置、适配器及24条预测记录的哈希核验通过，三个训练检查点已留档。科学结论为已测试配置nogo，Q4仍未获准。零训练参照529行CSV、529张PNG、exit130复核通过 | `reports/Q3_stop_review_acceptance.json`、`state/done/Q3_STOP_REVIEW.done.json` |
| R0 弱监督修正计划与预检（passed） | 四组对照、划分和交接规则成文。100份W1亮度图、100份W2.5修正图与100份DINO特征哈希匹配，407张训练候选与开发ID无交集，训练缓存缺口32张。未读取GT掩码，无GPU | `reports/R0_refiner_preflight.json`、`state/done/R0.done.json` |
| R1 训练清单与协议冻结（passed） | EORSSD/ORSI-4199/ORSSD=10/14/8，含11小目标和3空场景。替换5张有明显伪标签问题的初选图，32张的点、涂鸦和掩码核验通过。两组各500步，冻结评价门槛和生成种子。CPU，GT掩码读取0 | `reports/R1_refiner_freeze_acceptance.json`、`state/done/R1.done.json` |
| R2 P2@1024 训练缓存（passed） | 续跑退出0，32条记录、64份PNG哈希及32张BT.601亮度图逐像素核验通过。PyTorch保留显存最高20466 MiB，未读GT。首次OOM日志保留 | `reports/R2_refiner_cache_acceptance.json`、`state/done/R2.done.json` |
| R3 双组修正器短试验（passed） | 冻结DINOv2编码32图，两组同初始化同批次，C读Qwen亮度图，D对应通道全零。未知像素梯度为0，恢复原图342×535，重载后续步参数最大误差4.18e-7。128条训练数据访问均命中允许清单，GPU1保留峰值698 MiB | `reports/R3_refiner_smoke_acceptance.json`、`state/done/R3.done.json` |
| R4 双组配对训练（passed） | GPU1退出0，C/D各500步，初始化和500批样本顺序匹配。10个检查点均重载并记录哈希，128条访问均命中训练允许清单，GT读取0。PyTorch保留显存峰值720 MiB | `reports/R4_refiner_train_acceptance.json`、`state/done/R4.done.json` |

**S1 关键事实：** 训练对 6,999；manifest `ba11c6c3…1446a0`；2 组 train/test 精确重复 + 158 组跨源同源样本（已披露，未删除）；RGBA 派生 4,000；审计集 100；overfit 48/16。

**S2.2 关键事实：** 8 张全部 RGBA，alpha 0–255 非常量；峰值 17,301 MiB；同种子跨运行输出**逐字节相同**（`3a4677aa…`）。

---

**C1后诊断（passed）：** `reports/W5_C1_failure_diagnosis.json`、`reports/W5_C1_diagnosis_artifacts.sha256`、`state/done/W5.C1_DIAG.done.json`。500训练样本/4000增强+100开发样本，训练GT与官方test未访问。首次350缩放使12/480非空图失去所有前景弱像素；small-proxy增强后前景缺失5.12%。dev small召回/精确率：基线80.18%/53.51%，C1 82.29%/52.03%。诊断通过不改变C1拒绝结论。

## 3. 关键数字（一页看完）

**C2执行链路（CPU预检passed，训练paused）：** `reports/W5_C2_execution_preflight.json`、`reports/W5_C2_execution_artifacts.sha256`、`state/done/W5.C2_EXEC_PREFLIGHT.done.json`。独立train/evaluate/orchestrate/runtime与tmux脚本已部署；实际checkpoint/restore函数在CPU小模型上验证optimizer/EMA/scheduler/RNG下一步一致，错误配置/清单拒绝。完整GPU模型恢复尚未实测。训练和评测算法文本除数据模块/配置hash/输出与审计路径外保持基线一致，GPU启动条件不满足，run尚未创建。

**C2实现与CPU预检（passed）：** `reports/W5_C2_preflight.json`、`reports/W5_C2_preflight_artifacts.sha256`、`state/done/W5.C2_PREFLIGHT.done.json`。500×8增强，3840非空图增强中前景弱信号完全丢失124→10；99887个视野内点保留检查通过、3745个视野外点排除、冲突像素0；图像/三值伪标签保持一致，数据层重复seed一致，训练GT访问0。此结果不是模型精度提升，完整训练checkpoint恢复尚待后续链路验证。

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

### 3.3 W1 冻结零训练开发基线（100张）

| 样本 | Sα 均值 | MAE 均值 | Fmax 均值 | 失败 | 峰值显存 |
|---:|---:|---:|---:|---:|---:|
| 100 | **0.5312** | **0.0905** | **0.8930** | 0 | 19,594 MiB |

三数据集分组 Sα：EORSSD 0.5986（38张）、ORSI-4199 0.4747（51张）、ORSSD 0.5608（11张）。small 子集 21张，Sα 0.5203、MAE 0.0040。这是**开发基线**，不是官方测试结果。100张冻结 RGBA 输出均已按 BT.601 派生为单通道灰度图，没有重跑模型。

### 3.4 W4 弱监督64张过拟合与损失验证

| 训练/验证 | 弱点满足率 | 验证 Sα | MAE | Fmax | Boundary-F | 峰值显存 |
|---:|---:|---:|---:|---:|---:|---:|
| 48 / 16 | **1.0000**（932/932） | **0.6325** | **0.0518** | **0.8160** | **0.2631** | 456 MiB |

验证 Sα 较 W1 参照 0.5312 提升 **+0.1013**。打乱输入与纯黑输入的 Sα 分别为 0.1175、0.0006，相对正确输入下降 0.5150、0.6319，说明模型确实使用图像内容。训练访问审计中 dense GT 为0；16张GT只由独立 evaluator 在训练后读取，因此该结果仍是开发阶段的弱监督验证，不是官方测试集最终成绩。

### 3.5 参照线

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

**后续非全监督主路线：** `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md`。该计划采用 Qwen 语义先验、SAM 边界提议、DINOv2 一致性、点/涂鸦弱标注和直接灰度学生网络；W4小样本训练与损失验证已通过，W5主训练尚未启动。

**W5学生训练冻结协议：** `configs/weak_supervision/w5_student_training_protocol_v1.json`（hash `b6a1fe39…bf005`）。冻结值包括：DINOv2-B编码器冻结、双头轻量解码器、350输入、AdamW `3e-4`、最多5000 optimizer steps、七项loss权重、EMA阈值、seed `20261105/06/07`严格串行、开发100张仅独立eevaluator读GT。伪标签资格不读GT：人工确认空场景，或弱提示一致率≥0.90且SAM predicted-IoU≥0.80。其余41张不删除，仅把伪标签loss置0。

---

## 5. 状态文件与权威性顺序

1. 用户最新指令（最高）。
2. **实际产物**（`reports/`、`runs/`、`monitor/` 日志）与本说明书。
3. `state/progress.yaml`、`state/done/*.json`、`state/history.jsonl`。
4. `paper/` 下的计划书与协议。

**当前状态：**

- W3 已以 `passed_with_persistent_sam_failures_excluded` 完成；验收报告为 `reports/W3_acceptance.json`，done marker 为 `state/done/W3.done.json`。
- W4 `overfit64-v1` 已通过并闭环：`state/done/W4.done.json`、`reports/W4_acceptance.json`、`reports/W4_resume_validation.json` 与 `reports/W4_artifacts.sha256` 全部复验一致；四个检查点均可重载，逐文件哈希无不匹配。最终checkpoint为 `runs/weak_supervision/W4/overfit64-v1/checkpoints/step_0500.pt`。
- W4训练配置hash `9331abc7…7855ff`，64张manifest hash `ab384cff…9cca8`，独立16张评估GT manifest hash `9f693586…43c6b`。训练dense-GT访问0、评估GT 16张；GPU 1已释放（收尾时18 MiB、利用率0%），GPU 0上的他人进程未触碰。
- W5 B500标注配置hash `db4d0771…b8110c`，任务manifest hash `fed744c0…2ecea`。任务从3443张 `weak_train_pool` 中冻结，继承W3的100张并新增400张；100张难例按34/51/15分配涂鸦要求，空场景不会被选为涂鸦任务。任务构建访问审计未读取dense GT。
- W5标注验收已通过：500/500、来源为W5 425份+W3继承75份，annotation index hash `b50daa8e…a6898`、500份标注hash清单 `064ef2e8…a4e78`；图像hash不匹配0、额外文件0、dense-GT使用0、模拟标注0。实测总标注页面用时7875.524秒（约131.3分钟）。
- W5伪标签配置hash `e32f69ec…9c561f`：SAM使用全部真实点，并从每条真实涂鸦确定性采样最多8个提示点（每类总上限32），最终三值标签强制保留完整点与涂鸦像素；20张人工确认空场景跳过SAM并生成全背景标签。物理GPU 1启动前15 MiB占用、24121 MiB空闲、无计算PID、项目锁空闲；GPU 0他人进程未触碰。
- W5伪标签attempt1退出码1：第219张触发“前景与背景弱标注像素必须同时一致”的安全断言；全量CPU诊断仅发现2张涂鸦交叉，共123像素，点—涂鸦冲突0。报告为 `reports/W5_annotation_conflicts.json`。前218套结果未损坏且保留，恢复前必须先由用户修正两张并更新标注/索引hash；不得用自动前景或背景优先级掩盖冲突。
- 两张定向修正后全量复验为500/500、交叉图像0、交叉像素0；修正报告状态 `corrected_and_passed`。前218套输出的四类文件均存在并可重开，NPZ样本ID、配置hash、shape、值域、dense-GT=false、simulated=false全部通过；恢复报告为 `reports/W5_pseudo_resume_validation.json`，attempt2只生成219–500。
- W5 attempt2启动前预检曾见GPU 1空闲，但包装器取得项目锁后的最终检查发现用户 `xuwenyu` 的PID 3625096已占用GPU 1约12472 MiB，故以退出码75拒绝启动；这是并发占用保护，不是模型或数据失败。未启动CUDA模型、未写新伪标签、未触碰他人进程，恢复点仍为218/500。
- 2026-11-05用户明确授权改用GPU 0。`reports/W5_gpu0_execution_override.json` 冻结该执行级覆盖：物理卡1→0，科学配置hash、SAM权重、标注索引、提示采样、三值规则、输出目录均不变；生成器只解除硬编码卡号并从包装器环境读取实际物理卡。GPU 0启动前无进程且项目锁空闲；GPU 1与其他用户进程保持不动。
- W5 GPU 0 attempt3退出码0并完成500/500。最终报告：非空样本SAM predicted-IoU均值0.891984、弱提示一致性均值0.975884、原始SAM完全满足弱提示306/500；平均前景/背景/ignore比例0.106524/0.868991/0.024486。四类产物各500、2000项hash不匹配0、访问审计1006条且dense-GT访问0，最终验收 `reports/W5_pseudo_final_validation.json` 通过。
- W5学生协议冻结验收通过：500张均保留真实弱监督，409张过无GT置信门可用三值伪标签loss，91张仅真实弱监督；开发100张零重叠。配置hash可重算一致，3份冻结产物hash全过；未访问官方test、未启动训练、无W5 done、W6未启动。
- W5 seed-0实现/预检最终通过：配置hash不变，Python编译与bash launcher语法、数据解码/增强、GT防火墙、七项loss有限反传、91张伪标签loss置0、checkpoint/resume、单卡wrapper接线均通过。可训练参数1,869,698；预检瞬时GPU 0为102 MiB/0%且项目锁空闲，GPU 1有他人负载未触碰；真正启动前必须重检。attempt1/2是预检器列名/自检误报，无GPU、无训练checkpoint，日志保留。
- W5 seed-0训练验收通过：退出0，冻结早停在2750 step生效，最佳step 1250的Sα/MAE/Fmax/Boundary-F/IoU为0.816237/0.022035/0.862000/0.612809/0.743555，相较W1的Sα提升+0.285037。分组中small仅0.597262，明显是优先修正点；EORSSD/ORSI-4199/ORSSD分别0.744142/0.863401/0.846620。训练访问审66011条且dense-GT访问0；evaluator只读冻结dev100，官方test访问0。GPU0已回落至100 MiB/0%。
- C1仅修正采样分布：基线22000次抽样中small-proxy实际占32.6318%，与161/500=32.2%的均匀期望一致；C1预声明提高到50%（micro-batch 2+2）。small-proxy为161张（EORSSD/ORSI-4199/ORSSD=95/45/21），其中97张可用伪标签loss。训练分池不读GT；dev GT仅用于确认small是最弱类别与后续配对验收。
- C1.1实现不修改基线脚本：新增C1 sampler/train/evaluator/orchestrator/launcher，使用独立 `student-b500-c1-seed0-v1` run目录；checkpoint绑定C1配置hash `cf1b1c3e…79814`和采样清单hash `f2187e94…1e504`。2000个micro-batch覆盖测试全部通过，500张全部可达；预检时GPU0仅100 MiB/0%，GPU1他人负载不触碰，正式启动前仍须重检。
- C1 attempt1启动时GPU0独占条件通过，但运行到step 1250后发现其他用户新进程占用同卡约7020 MiB。项目以Ctrl-C仅中断自身tmux，保留已完成分片；中断发生在1500 chunk中，因无1500 checkpoint，权威恢复点为1250。已完成训练访问审35958条、dense-GT访问0，evaluator仅读dev100，官方test访问0。
- 监控页曾因前端只读取 `progress.done` 而状态写入使用 `progress.current`，显示错误地停在0；实际生成日志与产物正常。已把完成态纠正为500/500，并在 `monitor/server.py` 加入current→done兼容转换后仅重启项目监控进程；本地8765隧道也已带保活重建。该问题未影响实验。
- W4 attempt1 在首个optimizer step之前因PIL `mode=I`不接受int8三值数组而退出1，无checkpoint、无参数更新；日志保留为 `monitor/w4_attempt1.log`。attempt2仅把resize输入转换为int32，实际读取验证为`{-1,0,1}`不变；数据、loss、阈值、seed和配置hash均未改变，GPU重新预检后从step 0恢复。
- `manifests/weak_supervision/w3_training_eligibility_v1.csv` 是后续训练的强制入口：100行唯一ID，其中96行 `point_plus_tristate / pseudo_label_eligible=true`，4行 `point_only / false`。4张保留的SAM产物仅供审计，不得静默重新启用。
- 修正后复核配置hash为 `30e981ed…08f9f`，13份记录全部通过ID、格式、配置hash和 `dense_gt_used=false` 校验；结论9 accept、4 revise。复核报告为 `reports/W3_targeted_correction_review.json`，第二轮清单hash为 `e48dd728…a54a4`。
- W2.4 预声明12变体：候选匹配阈值 `{0.05,0.10,0.20,0.30}` × 融合系数 `{0.25,0.50,0.75}`；固定选择目标 `J=Sα−MAE+0.1×Boundary-F`，配置 hash `d13a8a59…`。生成器未读 GT，独立 evaluator 访问恰好100张开发 GT。
- 选中 `t10_l25`（阈值0.10，融合系数0.25）；同 evaluator 重算 W1 为 Sα 0.5333 / MAE 0.0903 / Boundary-F 0.5149，融合为 Sα 0.5408 / MAE 0.0912 / Boundary-F 0.5014，因此 W2.4 科学 Go 门槛失败。
- W2.5 沿用 W2.4 的阈值0.10、融合系数0.25和选择目标，只新增 DINO 权重 `{0.25,0.50,0.75}`；配置 hash `94016d66…a5fd0`。生成器未读开发 GT，evaluator 恰好访问100张开发 GT。
- W2.5 `d75` 相比 W2.4：Sα +0.0046、MAE -0.0006、Boundary-F +0.0052；相比 W1：Sα +0.0120、MAE +0.0002、Boundary-F -0.0082。因此 DINOv2 有小幅正效果，但不足以通过冻结 W2 门槛。
- W2.6 明示使用开发 GT，仅用于路线诊断；100张共155个正点、200个每图可复用背景点，共355次唯一点击，截断目标面积为0。attempt1/2 均未产生样本结果，attempt3 完成300张预测和300行指标，全部哈希通过；配置 hash `9cb32055…f984`。
- W3 初始配置 hash `ced360fd…5d76`；空场景兼容修订后的 v2 配置 hash `7f05d2ed…8990`。从 `weak_train_pool` 冻结 EORSSD/ORSI-4199/ORSSD = 34/51/15，并设置 small/thin/multi/shadow/low_contrast/holes/large/empty 覆盖下限。非空场景每张至少1个前景点、2个背景点；empty 模式允许0前景但仍需至少2背景点。
- W3 任务 manifest hash `7e5e8896…05ab`；实际难例覆盖 empty 5 / holes 10 / large 10 / low_contrast 14 / multi 35 / shadow 16 / small 30 / thin 47。标注页为 `http://127.0.0.1:8766/`，只服务原图，少于1前景+2背景点时拒绝保存。
- S2.4 探针没有独立 done marker；其权威证据仍是 `reports/S2.4_*` 与本文件第3.2节。
- W0 的权威完成标记是 `state/done/W0.done.json`。
- W1 的权威完成标记是 `state/done/W1.done.json`，验收汇总是 `reports/W1_acceptance.json`。

---

## 6. 未决事项（按优先级）

1. **Q3分支结案，Q4未获准**。三轮结果仅涵盖四张训练图，均值来自三张非空图和SAM弱标签。适配器、配置、检查点与预测保留，未做正式测试。停车场图 ORSI-4199/train/1185 暂缓。独立零训练参照暂停在529/2999，恢复须由用户指定。
2. **R5 待执行**。两组正式训练已验收。R5才生成并评分，训练损失只反映训练目标。首轮开发集曾用于W2.5选参，独立确认另行划分。
2. **GapClosure 分母定义**：`S_full_reference` 使用传统方法最优值还是自跑全监督参照，需要在正式比较前冻结。
3. **编号体系明确**：W 系列取代原 S3 以后训练主线；旧 S 编号只作历史追溯。
4. **录入传统方法数值**：MCCNet / GeleNet / IPDiff 指标必须来自原文表格。
5. **官方测试集评测计划**：100张只是开发集，论文表格需在三个官方测试集上一次性评测。
6. **修正旧计划书冷启动命令**：旧文档仍有 `progress.json`/`state/NEXT_ACTION.md` 路径，实际是 `progress.yaml`/`reports/NEXT_ACTION.md`。
7. **兼容 Junction 管理**：旧路径指向新工作区；待所有任务绑定新路径后再决定是否移除。
8. **把 `.codex-tmp/`、`archive/` 移出百度同步目录**。

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
# W1 零样本开发基线（脚本已就绪，等待空闲 GPU）
cd /data2/wangsiyi/qwenalpha-sod
bash scripts/run_single_gpu.sh <free_gpu> -- /data2/wangsiyi/envs/qwenalpha-sod/bin/python \
  scripts/s2_4_probe.py --output-resolution 1024 --variants P2 \
  --samples audit --prefix W1_qwen_zero_shot_100
# 产物：reports/W1_qwen_zero_shot_100.{csv,json}、runs/W1_qwen_zero_shot_100/P2/
# 不加载模型即可先校验路径：加 --dry-run，会打印 run_dir / csv / json / config_hash
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
- 零样本能出图，但**质量有限**：10张配对探针 Sα 0.5900，冻结100张开发基线 Sα 0.5312，距传统方法约 0.37。
- 这 0.3 的差距就是论文要证明的"创新处理"的价值所在。
- 主路线已定为**非全监督**（Qwen + SAM + DINOv2 + 点/涂鸦 + 直接灰度学生），不再走全监督微调；GT 只做一次性评测。
- W4已证明当前弱监督损失链路能够在64张小样本上稳定学习：验证 Sα 0.6325、较W1 +0.1013，反事实和GT防火墙均通过；这支持进入W5，但不代表已达到最终方法成绩。
- W5学生训练协议已冻结，409/500可用伪标签loss、91/500安全降级为仅真实弱监督；这是训练输入口径，不是模型成绩。
- W5 seed-0已取得Sα 0.816237，较W1提升0.285037，但差0.82门槛0.003763；不得四舍五入为达标。small子集Sα 0.597262是当前主要短板，下一步仅允许一个小目标或伪标签修正方案。
- 已选择且仅选择C1小目标平衡采样；它的价值必须由同seed、同配置的配对实验证明，冻结本身不代表已提分。
- C1已产生部分checkpoint/评测，但尚未完成早停过程；partial最好Sα 0.794149低于基线，但不能以中途结果判定C1失败。必须从冻结恢复点完成后再按配对门槛裁决。
- 一切比较都必须同集、同口径、配对统计；基线只测一次。

---

## 11. 维护规则与变更日志

### 11.1 谁在什么时候更新

**任何 agent**（Harness、Codex 或其他人）必须在开工前读取本文件。发生 `running / paused / failed / nogo / passed`、NEXT_ACTION、冻结协议或安全恢复点变化时，必须在同一原子步骤内回写；完成一个原子步骤时还必须：

1. 在 §11.2 变更日志追加一行（时间、agent、步骤、一句话改动）；
2. 更新 §0 速览的"现在在哪一步 / 下一步唯一动作"；
3. 在 §2 进度表增删行（新增完成项或状态变化）；
4. 调整 §6 未决事项（完成项移除、新发现项按优先级插入）；
5. 递增头部版本号与更新时间。

**这属于步骤收尾的一部分**，与写 `state/done/<STEP>.done.json`、更新 `progress.yaml`、更新监控页同等必要。只更新远端状态而不更新本文件，视为步骤未收尾。

“实时更新”采用事务式里程碑同步，而不是高频轮询：长任务的文件数、batch、百分比、日志尾部只进入 `http://127.0.0.1:8765/` 和远端日志；本文件只在状态、结论、下一步、恢复点或协议改变时更新。禁止为此创建 watcher、scheduler 或持续 Git 提交。

### 11.2 变更日志

| 时间（+08:00） | Agent | 版本 | 改动 |
|---|---|---|---|
| 2026-10-30 18:40 | Harness | v1.0 | 初版：环境坑、S0–S2.4 进度、冻结协议、未决事项 |
| 2026-10-30 19:00 | Harness | v1.1 | 纳入非全监督主路线计划书（W0–W9）；新增训练池剔除 110 张、GapClosure 分母、监督预算、SAM/DINOv2 下载、W1 产物命名、S/W 编号关系共 6 项未决；编号对照扩为 S+W 两套；加入本维护规则 |
| 2026-10-30 19:01 | Codex | v1.2 | W0 通过：冻结6999样本角色与点监督+约20%难例涂鸦预算；训练池剔除110张开发样本，登记并阻断6999条 dense-GT 路径；状态与 NEXT_ACTION 更新到 W1 |
| 2026-10-30 19:03 | Codex | v1.3 | 将共用说明书提升为所有 agent 的强制事务式进度入口：状态、下一步、协议或恢复点变化时同原子步骤回写；逐文件进度继续由 HTML 监控承担 |
| 2026-10-30 19:05 | Harness | v1.4 | 尝试执行 W1，预检发现两张卡均被同一用户占用（GPU 0 空闲17049 MiB < 峰值19537 MiB；GPU 1 承载他人 test.py 85%），按规程记 `paused` 并禁用抢占/共享；progress.yaml、history、NEXT_ACTION、监控同步为 paused；W1 未启动、无产物 |
| 2026-10-30 19:12 | Harness | v1.5 | W1 脚本就绪：新增 `--samples audit`（冻结100张）、`--prefix`（W1 命名）、`--dry-run`、config hash（`78a61650…`）与按数据集/小目标分组统计；CSV 增加 dataset/categories 列；dry-run 与探针回归均已验证，未写任何产物、未占 GPU |
| 2026-10-31 14:50 | Codex | v1.6 | W1 验收通过并完成状态闭环：100/100、失败0，Sα 0.5312 / MAE 0.0905 / Fmax 0.8930，峰值显存19594 MiB；按 BT.601 补齐100张单通道灰度预测，done/progress/NEXT_ACTION/监控均已更新，W2 未启动 |
| 2026-10-31 14:56 | Codex | v1.7 | W2.0 启动：冻结官方 SAM ViT-H `87aecf0…` 与 DINOv2 ViT-B/14 `f9e44c8…`（Apache-2.0），仅下载 safetensors 与配置并做逐文件 SHA-256；不占 GPU，W2.1 未启动 |
| 2026-10-31 15:02 | Codex | v1.8 | W2.0 通过：SAM ViT-H / DINOv2 ViT-B/14 共10个文件已通过字节数、JSON、safetensors 和 SHA-256 校验；最终总清单生成器的 `KeyError` 已从既有文件恢复并保留原日志，未重下、未占 GPU；W2.1 未启动 |
| 2026-10-31 15:04 | Codex | v1.9 | W2.1 预检暂停：GPU 0 有3个他人计算进程（余额6663 MiB），GPU 1 有1个他人计算进程（余额17089 MiB）；未抢占、未共享、未占锁、未启动 GPU 任务，恢复点仍为 W2.0 done marker |
| 2026-11-01 14:32 | Codex | v2.0 | W2.1 重试启动：GPU 1 无计算进程且有24121 MiB 空闲，已经单卡 wrapper 对 `EORSSD/train/0982` 原图顺序运行 SAM 自动候选与 DINOv2 特征冒烟；不读 GT，配置 hash `96022468…` |
| 2026-11-01 14:32 | Codex | v2.1 | W2.1 attempt1 因 Transformers 5.17 SAM fp16 NMS 的 boxes/scores dtype 不一致退出，无有效产物；保留失败日志后重做 GPU 1 预检，attempt2 改为 SAM float32 / DINOv2 float16 顺序恢复，配置 hash `136f4323…` |
| 2026-11-01 14:33 | Codex | v2.2 | W2.1 通过：SAM 生成11个候选，DINOv2 patch 特征 `[1,256,768]` 且全部 finite；两模型顺序加载，GPU 1 峰值5491 MiB，未读 GT；NPZ、launcher、日志与 done marker 均已验收，W2.2 未启动 |
| 2026-11-01 14:43 | Codex | v2.3 | W2.2 启动：GPU 1 无计算进程且有24121 MiB 空闲，冻结100张无GT原图 manifest 后经单卡 wrapper 生成 SAM-only 候选；10个可恢复分片、逐样本原子 NPZ，配置 hash `f4dfb189…` |
| 2026-11-01 14:45 | Codex | v2.4 | W2.2 通过：100/100 NPZ、10/10分片、失败0，共1988个 SAM 候选（均值19.88，范围1–65，零候选样本0）；逐文件 SHA-256、NPZ 重开、launcher 均通过，GPU 1 峰值5493 MiB，未读 GT；W2.3 未启动 |
| 2026-11-01 14:48 | Codex | v2.5 | W2.3 启动：先冻结 SAM-only 无GT聚合规则（hash `b6933492…`），再创建 evaluation-only GT manifest；聚合器和 evaluator 为两个独立 CPU 进程，分别写访问审计，不根据评分回改规则 |
| 2026-11-01 14:49 | Codex | v2.6 | W2.3 通过：SAM-only Sα 0.2957 / MAE 0.3086 / Fmax 0.6477 / Boundary-F 0.3864，相比 W1 的 ΔSα=-0.2355，明显更弱；聚合器未读 GT，独立 evaluator 仅访问100张开发 GT。验收字段命名 bug 已记录修正，未改预测或指标；W2.4 未启动 |
| 2026-11-01 14:51 | Codex | v2.7 | W2.4 启动：冻结12个 Qwen+SAM 变体和选择目标 `J=Sα−MAE+0.1×Boundary-F`（hash `d13a8a59…`）；无GT生成器先产生融合结果，独立 evaluator 才读取100张开发 GT 选优与检查 Go 门槛，CPU-only |
| 2026-11-01 14:52 | Codex | v2.8 | W2.4 技术验收通过、科学 Go 门槛未通过：选中 `t10_l25`，Sα 0.5333→0.5408（+0.0075），MAE 0.0903→0.0912，Boundary-F 0.5149→0.5014；生成器未读 GT，evaluator 仅访问100张开发 GT。网格冻结不扩展，W2.5 未启动 |
| 2026-11-01 14:58 | Codex | v2.9 | W2.5 启动：沿用 W2.4 `t10_l25` 冻结参数，新增 DINOv2 候选紧致度/内外分离度/水平翻转一致性及3个预声明权重（hash `94016d66…`）；GPU 1 单卡执行，生成器无 GT，后续 evaluator 独立读取开发 GT |
| 2026-11-01 15:00 | Codex | v3.0 | W2.5 技术验收通过、冻结 W2 Go 门槛失败：100份特征/NPZ/预测与全部哈希通过，选中 `d75`，Sα 0.5333→0.5453（+0.0120），MAE 0.0903→0.0906，Boundary-F 0.5149→0.5067；生成器0张GT、evaluator 100张GT，GPU 1 锁已释放。停止等待 W2 路线决策，不自动进入 W3 |
| 2026-11-01 15:06 | Codex | v3.1 | W2.6 启动：冻结 SAM 单候选/贪心联合 oracle 与每目标1正点+2背景点的模拟提示协议（hash `87a2e25a…`）；诊断明示读取100张开发 GT，不训练、不计作无监督成绩，GPU 1 单卡执行 |
| 2026-11-01 15:07 | Codex | v3.2 | W2.6 attempt1 在模型加载后因 evaluation manifest 缺少预期 `image_path` 字段退出，未生成诊断结果；保留失败日志，attempt2 仅按 `sample_id` 连接冻结 W2.2 原图 manifest（hash `9cb32055…`）后恢复，协议、GT口径与门槛不变 |
| 2026-11-01 15:09 | Codex | v3.3 | W2.6 attempt2 进入首张 SAM 点提示后，因 Transformers 5.17 后处理要求保留 multimask 单例维而退出，仍未落盘样本结果；经单图形状探针确认 `pred_masks=(1,N,3,256,256)` 后，attempt3 只修正后处理张量维度并恢复，协议与门槛不变 |
| 2026-11-01 15:11 | Codex | v3.4 | W2.6 通过：attempt3 完成300张诊断预测/300行指标/全部哈希；SAM 贪心 oracle IoU 0.7293（small 0.6503），模拟点提示 IoU 0.7532（small 0.7891）均过冻结门槛，推荐 W3 真实弱点阶段而非先替换 SAM。诊断使用100张开发 GT，不是无监督成绩；GPU 1 锁已释放，W3 未启动 |
| 2026-11-01 15:14 | Codex | v3.5 | W3 setup 启动：冻结弱训练池100张数据集/难例配额与真实前景—背景点标注协议（hash `ced360fd…`），建立仅读原图的标注网页、原子保存、用时/点击审计与GT禁读约束；工具就绪后暂停等待真实人工标注，不伪造完成 |
| 2026-11-01 15:16 | Codex | v3.6 | W3 setup 通过并暂停等待人工输入：冻结100张 `weak_train_pool` 任务（EORSSD/ORSI-4199/ORSSD=34/51/15，manifest hash `7e5e8896…`），难例配额和GT禁读验收通过；`127.0.0.1:8766` 标注服务及本地隧道可用，非法保存会被拒绝。当前0/100，无 W3 done marker，不自动伪造标注 |
| 2026-11-02 18:26 | Codex | v3.7 | 恢复 W3 人工点标注：核验远端标注与监控服务仍在，实际已有2/100；复用冻结任务清单、已有标注和 `qas_w3_annotator`，重建本机8765/8766 SSH转发，并将 progress、active run、NEXT_ACTION 与监控统一为 running；无 GPU、无 W3 done marker |
| 2026-11-02 18:48 | Codex | v3.8 | W3人工标注达到100/100；机械验收100份全通过（662前景点、1353背景点，ID/格式/图像hash无误），但发现5张empty任务均含旧UI强制的1–2个前景点，与“空场景只允许背景”协议冲突。状态安全暂停等待空场景复核，未启动GPU/SAM、未写W3 done |
| 2026-11-02 18:52 | Codex | v3.9 | 部署W3空场景协议v2（hash `7f05d2ed…8990`）与网页：仅冻结为empty的任务可显式勾选空场景，以0前景+至少2背景保存；保留其余95份标注。服务HTTP与100/100清单验收通过，暂停等待5张人工复核，未启动GPU/SAM |
| 2026-11-02 19:01 | Codex | v4.0 | W3真实人工点标注最终验收通过：100/100、662前景+1353背景，覆盖/ID/图像hash/最少点数/GT与模拟点禁用均通过；5张empty-category样本经用户连续确认后保留为非空显著场景，原点击未改写。生成验收报告与逐文件SHA-256，安全暂停等待SAM点提示；未写W3 done、未进入W4 |
| 2026-11-02 19:13 | Codex | v4.1 | 启动W3 SAM点提示与三值伪标签：冻结无GT协议（hash `22170b74…3f084`），以真实2015个点击联合提示SAM并按点击一致性+SAM predicted-IoU选择multimask，腐蚀/膨胀边界带记ignore；GPU 1启动时无进程且空闲24121 MiB，单卡wrapper/tmux运行，GPU 0他人进程未触碰 |
| 2026-11-02 19:20 | Codex | v4.2 | W3 SAM/三值生成技术验收通过：100×4类产物、400项hash、201条访问审计，最终标签2015/2015点击一致，GPU峰值5178 MiB并已释放。attempt1因误把原始SAM 100%点击一致设为硬门槛而退出2；失败报告保留，CPU修复仅重算验收、不重跑GPU/不改输出。88张原始SAM完全一致，12张进入worst-20待人工审计 |
| 2026-11-02 19:26 | Codex | v4.3 | W3 worst-20审计工具就绪：冻结20张清单与review协议（hash `410c2646…80e7`），页面逐张显示原图/叠加图/SAM mask/三值PNG并原子保存accept或revise；四类媒体HTTP验收通过，当前0/20。工具不服务GT、不使用GPU、不修改伪标签，暂停等待人工结论 |
| 2026-11-02 19:44 | Codex | v4.4 | W3 worst-20人工审计完成：20/20，7 accept、13 revise；11个extra-foreground（其中4张明确纯背景）与2个missed-object。20份review hash与13张定向修正manifest已冻结；未使用GT、未静默改写伪标签、未写W3 done，安全暂停等待定向修正指令 |
| 2026-11-02 19:48 | Codex | v4.5 | W3 13张定向修正工具就绪：冻结修正协议（hash `4dfa505a…1816`），页面提供原图+当前错误结果、点增删与4张纯背景模式；四类关键接口验收通过，当前0/13。修正记录独立保存，不覆盖原标注/伪标签，不使用GPU/GT，暂停等待人工输入 |
| 2026-11-02 20:02 | Codex | v4.6 | W3 13张修正记录验收并启动定向重生成：4张纯背景为0前景+足量背景，9张存在真实点变更；冻结生成配置（hash `71feed34…3375a`），GPU 1启动时空闲24121 MiB且无进程。旧13份结果先备份，仅重写13份，87份保持不变；单卡wrapper/tmux运行 |
| 2026-11-02 20:06 | Codex | v4.7 | W3定向重生成通过并暂停等待复核：4张全背景+9张SAM结果已落盘，52/52定向hash、400/400全体hash与13×4备份均验收，GPU峰值5178 MiB并已释放；部署无GT修正后复核页（hash `30e981ed…08f9f`），当前0/13，不写W3 done、不进入W4 |
| 2026-11-02 20:12 | Codex | v4.8 | W3修正后人工复核验收完成：13/13记录合法且未使用GT，结论9 accept、4 revise；52/52定向hash与400/400全体hash复验通过。冻结仅含4张未解决样本的第二轮清单（hash `e48dd728…a54a4`），状态暂停等待下一轮点修正，不写W3 done、不进入W4 |
| 2026-11-03 10:47 | Codex | v4.9 | W3第二轮4张定向修正页就绪：冻结配置hash `ade0711b…ea6f5`，页面从第一轮修正点继续编辑，只提供原图和当前错误结果、不提供GT；4任务、两类媒体及禁止无改动保存均验收，当前0/4。SSH隧道已恢复至8770，暂停等待人工输入 |
| 2026-11-03 11:03 | Codex | v5.0 | W3第二轮4张重生成通过并暂停等待最终复核：退出码0，仅预期16个产物变化，其余384个hash保持；16/16定向hash、400/400全体hash和4×4备份验收通过，GPU峰值5180 MiB并释放。部署无GT复核页（hash `885899e4…101f6`），当前0/4，不写W3 done、不进入W4 |
| 2026-11-03 11:42 | Codex | v5.1 | W3完成闭环：最终4/4仍为revise，按用户确认停止继续加点并冻结为persistent SAM failure；生成100行训练资格清单，96张point+tristate、4张point-only且禁止伪标签损失。最终hash、GT防火墙、验收报告与done marker均通过，W3状态passed，W4未启动 |
| 2026-11-03 11:51 | Codex | v5.2 | W4预检通过并启动：冻结48 train+16 val（point-only=0、三数据集和八类难例覆盖），损失与ignore单测通过；训练进程禁读GT、独立评估仅访问16张val GT。GPU 1启动前空闲24118 MiB，单卡wrapper/tmux运行step 1/10/100/500与恢复/反事实验收，W5未启动 |
| 2026-11-03 11:53 | Codex | v5.3 | W4 attempt1在step1前因PIL三值resize的int8兼容错误退出1，无checkpoint/参数更新；保留日志并验证int32转换后值域仍为{-1,0,1}。GPU 1重新预检空闲，attempt2只应用该兼容修复并从step0启动，科学协议与配置hash不变 |
| 2026-11-03 12:02 | Codex | v5.4 | W4完成并通过：step 1/10/100/500检查点重载与全产物哈希复验通过；弱点满足率1.0000，16张验证Sα 0.6325（较W1 +0.1013），打乱/纯黑反事实显著下降，训练dense-GT访问0、峰值456 MiB。done/progress/NEXT_ACTION/监控一致，GPU 1释放，W5未启动 |
| 2026-11-03 12:09 | Codex | v5.5 | W5 B500真实弱标注阶段启动并安全暂停等待人工输入：冻结500张分层任务（169/255/76），继承W3真实点100张、新增400张，指定100张非空难例补前景+背景涂鸦；八类覆盖、manifest hash、GT防火墙和页面API验收通过。当前75/500、待425，标注页8772；未生成伪标签、未训练、无W5 done，W6未启动 |
| 2026-11-03 14:14 | Codex | v5.6 | W5真实弱标注500/500机械验收通过并启动三值伪标签生成：共4756前景点、8198背景点、397前景涂鸦、352背景涂鸦及20空场景，500张图像/标注hash、坐标、100张指定涂鸦与GT/模拟禁用均通过。冻结点+涂鸦SAM协议（hash `e32f69ec…9c561f`），GPU 1预检空闲后通过单卡锁/tmux运行；尚未训练、无W5 done，W6未启动 |
| 2026-11-03 14:18 | Codex | v5.7 | W5伪标签attempt1在218/500后因安全断言退出1：全量诊断发现第219/229张前景—背景涂鸦分别交叉106/17像素，点—涂鸦冲突0；未自动裁决类别。218套产物保留，GPU 1释放；标注页将两张标红并提供清除涂鸦后重画，状态暂停等待人工修正，不训练、不进入W6 |
| 2026-11-03 14:23 | Codex | v5.8 | 两张涂鸦冲突经用户重画后复验通过：500张交叉图像/像素均为0，重新冻结annotation index/hash；前218套产物四类文件、NPZ、shape、值域、样本/配置hash和无GT标志全部通过。GPU 1重新预检空闲，attempt2从219/500恢复，不重算前218张，不训练、不进入W6 |
| 2026-11-03 14:24 | Codex | v5.9 | W5 attempt2未进入模型：GPU 1在预检后被其他用户 `xuwenyu` 的PID 3625096占用约12472 MiB，单卡包装器最终硬检查以退出码75安全拒绝。新增产物0、前218套保持有效，未干预他人进程；状态暂停等待GPU空闲后从219恢复，不训练、不进入W6 |
| 2026-11-05 14:20 | Codex | v6.0 | 按用户明确授权将W5执行卡从GPU 1切换到GPU 0：仅修改生成器的运行卡读取与资源报告，科学配置hash/协议/输出不变；GPU 0预检100 MiB占用、24027 MiB空闲、无PID、锁空闲，attempt3从219/500恢复并复用前218套。GPU 1不触碰，不训练、不进入W6 |
| 2026-11-05 14:27 | Codex | v6.1 | W5 B500三值伪标签GPU 0 attempt3退出0并最终验收通过：四类产物各500、2000项hash与500份NPZ/点涂鸦一致性全过，dense-GT/模拟标注0、峰值5180 MiB，GPU 0释放。修复监控current/done字段不兼容并恢复8765隧道，网页现为500/500；暂停等待单独冻结学生训练协议，不写W5 done、不进入W6 |
| 2026-11-05 14:40 | Codex | v6.3 | W5学生训练协议与资格冻结通过：500张全部保留真实弱监督，409张启用伪标签loss、91张降级；开发100张零重叠，配置/清单hash验收通过。未启动训练、无W5 done、W6未启动；下一步为单独实现与预检seed-0 B500 |
| 2026-11-05 14:56 | Codex | v6.5 | seed-0 B500训练/独立评估/恢复/早停/单卡launcher已实现，CPU预检通过：500=409+91、dev100零重叠、GT防火墙/loss反传/checkpoint resume全过，可训练参数1,869,698。未启动GPU训练、无W5 done、W6未启动；下一步仅能在重检GPU 0后单独启动seed-0 |
| 2026-11-05 15:12 | Codex | v6.7 | seed-0退出0并在2750 step早停，最佳1250 step：Sα 0.816237 / MAE 0.022035 / Fmax 0.862000 / Boundary-F 0.612809 / IoU 0.743555，较W1 +0.285037。11个checkpoint、1100预测、1136项hash、GT隔离与712 MiB峰值验收全过，GPU0释放。未达0.82，只允许单一修正，W6未启动 |
| 2026-11-05 15:23 | Codex | v6.9 | 冻结唯一C1小目标平衡采样：无GT三值前景比例定义161 small-proxy/339 standard，micro-batch从均匀4改为2+2；其他所有变量不变。配对接受门槛冻结为总体Sα +0.005、small +0.02、MAE≤0.002退化、单集Sα≤0.01退化；C1未训练，W6未启动 |
| 2026-11-05 16:28 | Codex | v7.1 | C1.1独立采样/训练/评测/恢复链路实现与CPU预检通过：2000个micro-batch/8000抽样严格2+2，500张全覆盖、无批内重复、resume序列一致；GT防火墙/checkpoint/单卡launcher全过，基线hash未变。C1未训练，W6未启动 |
| 2026-11-05 16:42 | Codex | v7.3 | C1 attempt1完成并验收至step1250；运行中GPU0被用户xuwenyu新PID占用，为避免共享仅中断项目自身tmux（exit130），对方未触碰。5 checkpoint/500预测/515 hash全过，step1500无checkpoint；从1250恢复，C1结论pending，W6未启动 |
| 2026-11-06 11:43 | Codex | v7.4 | GPU0空闲、项目锁与515项partial/预检hash复检通过，恢复C1至下一250-step分片；重启已退出的监控服务并重建本地隧道，HTTP200。保留旧exit和日志，从1250恢复，C1结论pending，W6未启动 |
| 2026-11-06 12:30 | Codex | v7.5 | 一次检查确认C1正常早停3500、exit0、最佳step2000；70%源于最大5000步分母，不是卡住。监控和状态统一为训练结束待验收，未重启训练，未执行W6 |
| 2026-11-06 12:34 | Codex | v7.6 | C1 CPU验收完成：14检查点可重载、1400 PNG、1442 hash、汇总指标/早停/GT审计通过；总体与small增益不达冻结阈值，C1 rejected，保留seed0。首轮验收脚本误用ema字段，按实际ema_student修正后通过，错误日志保留；未改训练产物，未创建科学通过done marker，未启动W6 |
| 2026-11-06 12:42 | Codex | v7.7 | 完成500×8弱监督几何诊断与dev100既有预测诊断，发现首次缩放前景弱信号丢失12/480及小目标精确率低；提出单一C2坐标变换保真及止损门槛，记录伪标签密集质量/训练预测证据缺口。未训练、未访问训练GT或test，诊断done与状态一致 |
| 2026-11-06 12:47 | Codex | v7.8 | C2协议与独立数据模块完成，4000组CPU预检通过：前景弱信号缺失124→10，图像与伪标签逐像素一致，随机序列一致，视野内点检查及视野外点排除通过。首轮数据模块缺私有函数导入已修复并保留日志；训练未启动，模型精度待验证 |
| 2026-11-06 12:50 | Codex | v7.9 | 部署C2独立训练/评测/恢复与checkpoint级监控链路；CPU实际checkpoint函数续步一致、错误hash拒绝，基线算法对比通过。GPU0预检PID45102占8606 MiB，按隔离要求paused；训练run未创建，无后台轮询 |
| 2026-11-07 | Codex | v8.0 | 用户改指定GPU1，实时预检无计算PID、约1 MiB、锁空闲；C2专用训练/评测/launcher卡号改为1，执行覆盖和状态写入后tmux启动返回成功。随后SSH断开、监控隧道拒绝连接；启动结果待复核，不据此重启或宣称训练完成 |
| 2026-11-07 | Codex | v8.1 | 重建本地8765隧道，页面HTTP200；SSH复核C2 tmux存活、GPU1项目进程及首个250步训练/评测，监控正确显示250/5000。继续等待现有任务事件，无轮询或重启 |
| 2026-11-07 | Codex | v8.2 | C2完成并验收：2750步早停、最佳1250，11 checkpoint/1100预测/1133 hash、训练GT访问0、开发集外评估0；总体Sα +0.000575、small −0.007270，未达预设增益，C2 rejected。原seed0保留，停止该配置修正，不启动W6 |
| 2026-11-07 | Codex | v8.3 | W5阶段复盘完成：复核seed0 checkpoint哈希及C1/C2最终产物清单，形成导师决策材料，保留seed0、拒绝C1/C2；建议下一独立步骤为只读误差审计与公平比较协议，未启动新训练或W6 |
| 2026-11-07 | Codex | v8.4 | 按用户要求锁定seed0后续单模型评测，冻结checkpoint、指标代码及2999张官方测试清单；GPU1独占预检后tmux启动。明确W6准入门槛未达，W6/W7及三seed统计未执行；等待测试验收 |
| 2026-11-07 | Codex | v8.5 | seed0一次性官方测试完成：2999图、30分片、3029 hash全过；合并Sα 0.723305，分集0.660627/0.735621/0.775915；单模型单seed，未做W6/W7、完整W8或W9。锁定模型不根据测试结果重选 |
| 2026-11-07 | Codex | v8.6 | 继续完成seed0官方测试的只读描述审计：small≤2%组806张Sα0.558276，三组非空面积及空场景分层报告；跨清单相同图像SHA-256共42组。仅解释锁定结果，不重选模型。W6/W7/完整W8/W9未完成，待新协议 |
| 2026-11-07 | Codex | v8.7 | 冻结修订后对照/评测边界并审计B500 500/500 train GT登记；确认train/test精确图像重合2条、test跨源重复42组。提出隔离FS参照、同集合GapClosure与测试后设计披露；无新训练，未虚报W6/W7/W8/W9 |
| 2026-11-07 | Codex | v8.8 | 实现隔离FS参考数据模块、GT精确允许清单与BCE+Dice损失；500张GT审计、增强尺寸/值域和CPU有限反传通过。原弱监督训练审计哈希记录，GPU训练未启动；下一步接入独立训练/评测/恢复 |
| 2026-11-07 | Codex | v8.9 | 实现独立FS训练/评测/恢复编排与GPU1 tmux入口；CPU数据确定性、checkpoint往返和错误配置拒绝通过。执行前增加实现hash核验使最终配置hash修订为94f620a6…2e3058，旧版本只作历史预检；未启动训练 |
| 2026-11-07 | Codex | v9.0 | GPU1实时无计算PID、项目锁空闲，FS独立参考tmux启动且模型加载成功，监控HTTP200；等待分片、dev100结果与完整验收。不改弱监督seed0，不用官方test选FS模型 |
| 2026-11-07 | Codex | v9.1 | FS参考3750步正常早停、最佳2250；15 checkpoint/1500 dev预测/1545 hash，30000条FS训练GT访问均限B500。dev100 Sα0.866471较弱监督+0.050234，MAE0.014728；官方FS测试未执行，GapClosure不可跨集计算 |
| 2026-11-07 | Codex | v9.2 | FS最佳2250 checkpoint/指标代码/2999官方test清单冻结，GPU1独占预检后事后FS官方测试tmux启动、模型加载成功；弱监督官方结果已知，不作盲测声明或测试后模型选择 |
| 2026-11-07 | Codex | v9.3 | FS事后官方测试退出0并验收2999预测/3029 hash，合并Sα0.750656，弱监督同集合0.723305，差0.027351；分集和排除train/test重合、按图像哈希去重敏感性分析完成。缺同集合零训练参照，GapClosure暂不算 |
| 2026-11-07 11:33 | Codex | v9.4 | 冻结同2999图P2@1024零训练参照（40步、seed按清单序号），GPU1无计算进程且项目锁空闲、监控HTTP200后通过单卡wrapper/tmux启动；远端状态/NEXT_ACTION/监控同步为running。只做事后描述性比较，等待退出与验收，不提前计算GapClosure或重选模型 |
| 2026-11-07 22:08 | Codex | v9.5 | 用户将目标改为直接提升Qwen生成显著图能力并要求暂停长任务。零训练参照在529/2999安全停止，exit130，529行CSV/529张PNG保留、GPU进程退出；远端状态/NEXT_ACTION/监控改paused。下一步小规模Qwen LoRA可行性测试，仅训练图、不使用官方测试调参；现有学生网络不是Qwen蒸馏成果 |
| 2026-11-07 22:10 | Codex | v9.6 | Qwen LoRA小范围CPU结构测试通过：真实配置32层生成Transformer，rank4 q/k/v/out LoRA meta挂载成功，4194304可训练参数；500张训练图入口可读。GPU真实训练、22GB峰值与Sα均未验证，状态paused等待单步GPU可行性；监控/NEXT_ACTION已同步 |
| 2026-11-07 | Codex | v9.7 | 用户要求先制定可由Codex与DeepSeek Harness任意接管的Qwen本体LoRA详细计划；Q0–Q7新计划和移交说明书v2.1已建立。主实验明确定义为弱监督，使用真实点／涂鸦＋SAM伪标签，完整训练GT分支若启动则单列全监督。下一步Q1目标图审计；未启动新训练 |
| 2026-11-07 22:20 | Codex | v9.8 | Q1审计通过。兼容W3继承的点标注格式，保留前三次失败日志。409份排除1张测试重复，408份候选，抽查5份并暂缓1份。冻结Q2单图输入，GPU训练尚未开始 |
| 2026-11-07 | Codex | v9.9 | Q2 经单卡包装器启动，验证真实 Qwen 的前向、反向、一次 LoRA 更新与重载 |
| 2026-11-07 22:25 | Codex | v9.10 | Q2 退出0并通过验收。512px单步更新256个LoRA张量，适配器重载精确一致，PyTorch保留显存峰值17180 MiB。GPU已释放，下一步Q3 |
| 2026-11-07 | Codex | v9.11 | Q3 冻结已抽查的4张弱标签图，启动64次更新和同条件配对生成 |
| 2026-11-07 22:36 | Codex | v9.12 | Q3 退出0，四组配对和64步记录核验通过。修正验收脚本对输出均为正方形的错误假设，按每组匹配尺寸验收，沿用已生成产物。质量门槛未通过，状态nogo，Q4暂停 |
| 2026-11-07 | Codex | v9.13 | 启动 Q3 失败诊断，复用原四图及配对预测，单卡执行 VAE 重建 |
| 2026-11-07 22:39 | Codex | v9.14 | Q3失败诊断通过。8张VAE重建Sα均≥0.9746，湖泊白色区域可以重建。记录提示与目标不匹配的待验证假设，下一步Q3_P3_RETRY，Q4暂停 |
| 2026-11-07 | Codex | v9.15 | Q3 P3 对照重试经单卡包装器启动。训练与推理统一换用 P3，保留 P2 产物和门槛 |
| 2026-11-07 22:50 | Codex | v9.16 | Q3 P3 重试正常退出并完成配对验收。P3相对原模型平均弱标签Sα增0.2914，低于P2 LoRA的绝对均值，仅1张非空图有可见改善。状态nogo，Q4暂停 |
| 2026-11-07 22:52 | Codex | v9.17 | Q3路线复核完成。按确定前景与背景量化亮暗错误，记录单图区域均衡损失验证协议。Q4仍暂停，未启动GPU |
| 2026-11-07 | Codex | v9.18 | Q3单图辅助损失验证经单卡包装器启动 |
| 2026-11-07 22:57 | Codex | v9.19 | Q3_LOSS_SMOKE退出0并通过验收。分阶段反传得到有限非零的LoRA辅助梯度，显存峰值17182 MiB。零更新，下一步四图配对实验，Q4暂停 |
| 2026-11-07 | Codex | v9.20 | Q3_LOSS_AB启动，配置对齐P3，辅助损失系数0.1，空场景仅使用背景项 |
| 2026-11-07 23:17 | Codex | v9.21 | Q3_LOSS_AB退出0，64步及4组配对核验。非空均值0.3251→0.2999，视觉0/3改善，空场景出现轻微假阳性。状态nogo，GPU释放，Q4暂停 |
| 2026-11-07 23:21 | Codex | v9.22 | 完成Q3_STOP_REVIEW，核对三轮配置、适配器、检查点及配对图。保留三轮负结果，已测试分支结案，Q4未获准。独立零训练参照529/2999断点复核通过 |
| 2026-11-07 23:37 | Codex | v9.23 | 建立Qwen弱监督修正的R0–R6共用计划，完成R0资产预检，冻结开发缓存依据。下一步R1冻结32张训练清单 |
| 2026-11-07 23:47 | Codex | v9.24 | R1完成32图逐图复核、5张同层替换和划分排重，冻结C/D各500步、生成种子与评分门槛。下一步R2 |
| 2026-11-07 23:51 | Codex | v9.25 | R2按冻结配置在GPU1远端tmux启动，已加载模型并开始生成。逐图持久化，退出后等待验收 |
| 2026-11-08 | Codex | v9.26 | 检查昨晚R2。远端exit1，第4张VAE编码OOM。3条恢复记录与6张PNG哈希匹配，保持failed，未重启 |
| 2026-11-08 | Codex | v9.27 | R2核验前三张并归档首轮失败证据。GPU1启用CUDA可扩展分段后续跑，第4张输出验收通过。恢复本地8765转发并确认状态可读 |
| 2026-11-08 11:27 | Codex | v9.28 | R2续跑退出0，32组缓存验收完成。64份PNG哈希通过，32张亮度图逐像素一致。下一步R3 |
| 2026-11-08 11:33 | Codex | v9.29 | R3配对短试验退出0并验收。两组参数更新、置零对照、未知像素梯度和断点重载通过，下一步R4 |
| 2026-11-08 | Codex | v9.30 | R4 使用已验收的实现经GPU1单卡锁启动 |
| 2026-11-08 11:37 | Codex | v9.31 | R4退出0，C/D各500步、10个可恢复检查点、匹配样本顺序和无GT访问验收通过。下一步R5 |

### 11.3 冲突处理

- 本文件与远端 `progress.yaml` / `NEXT_ACTION.md` 冲突时，以**实际产物**为准，并立即回写两者。
- 本文件与计划书冲突时，以**用户最新指令**为准；若无新指令，以计划书为准并在此登记差异。
- 禁止为了"让文档好看"修改已完成步骤的结论；历史结论只增不改，修正以新增日志行说明。
