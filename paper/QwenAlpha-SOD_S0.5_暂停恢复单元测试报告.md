# QwenAlpha-SOD S0.5：暂停恢复单元测试报告

> 执行日期：2026-10-27  
> 远端主机：`wangsiyi`  
> 远端工程：`/data2/wangsiyi/qwenalpha-sod`  
> 结果：**✅ Passed**

## 1. 测试目标

使用一个确定性的微型 GPU 训练任务验证完整恢复协议，而不是验证模型效果。测试严格执行以下序列：

```text
连续运行：step 1 → step 10

恢复运行：step 1 → step 6
                    ↓ 创建 PAUSE_REQUESTED
                 保存原子 checkpoint 并安全退出
                    ↓
              resume-dry-run → next_step=7
                    ↓
                 step 7 → step 10
                    ↓
             与连续运行逐项比较
```

所有 CUDA 进程均由 S0.4 的 `scripts/run_single_gpu.sh` 启动，只暴露物理 GPU 1。

## 2. Checkpoint 内容

暂停 checkpoint 保存：

- 模型 `state_dict`；
- AdamW 优化器状态；
- CosineAnnealingLR 调度器状态；
- global step；
- Python `random` 状态；
- NumPy RNG 状态；
- PyTorch CPU RNG 状态；
- PyTorch CUDA RNG 状态；
- 固定采样顺序；
- 数据游标；
- loss 历史。

Checkpoint 先写入临时文件，再通过原子替换成为正式文件，避免中断留下半写入文件。

## 3. 暂停与 dry-run

第 6 step 更新完成后创建 `PAUSE_REQUESTED`，训练器随即在安全边界保存 checkpoint 并退出：

```text
status=paused
completed_step=6
next_step=7
data_cursor=6
```

随后执行 `--resume-dry-run`，得到：

```json
{
  "completed_step": 6,
  "data_cursor": 6,
  "marker_present": true,
  "mode": "resume_dry_run",
  "mutation_performed": false,
  "next_step": 7
}
```

dry-run 前后的暂停 checkpoint SHA-256 完全相同，证明该操作没有修改恢复点。

## 4. 恢复一致性结果

恢复运行从 step 7 开始并完成 step 10。比较器递归检查以下部分：

```text
model
optimizer
scheduler
python_rng
numpy_rng
torch_cpu_rng
torch_cuda_rng
sampler_order
data_cursor
global_step
loss_history
```

结果：

```text
global_step=10
data_cursor=10
tensors_compared=19
max_abs_diff=0.0
tolerance=1e-7
within_tolerance=true
```

因此暂停恢复运行与连续运行不只是“接近”，而是在本测试检查范围内完全一致。

## 5. 运行与文件哈希

运行目录：

```text
/data2/wangsiyi/qwenalpha-sod/runs/s0_5_pause_resume/20261027T103257Z-pid1438098
```

Checkpoint 哈希：

```text
step-6 pause checkpoint  0265bdd6921917754158f6d099f45af9bd44b68c8859d33175f4bfcf59e39f04
continuous step-10       01801694f82c95a35b68c49b57712c4c581108162c0cdbbfb49543dd4400cc87
resumed step-10          6d7dec5d4401d6e6ee1c658526c739ccf4104b26e937707509a58d5377dcedc2
```

最终 checkpoint 文件哈希不同是正常的：序列化容器元数据不保证逐字节一致；加载后逐项比较的数值与状态完全一致。

代码与报告哈希：

```text
tests/pause_resume_trainer.py
4024363d1e6fa2fa069ebf8a8898843d926aa3bcb0c06990eb99eb311c450042

tests/compare_pause_resume.py
4479d9cb80e8631dbdabf965b810e76d816febc4ff01b3b837623abefe43b527

tests/test_pause_resume.sh
b159eeab393ad03c847570cc789351fa58463d9075365e00c9348c4e53f0b884

reports/S0.5_pause_resume_test.json
23b14ace09a6707a5880864947d32d0f4e5a208fa6bb71b9e0b2faae207c7a4d

reports/S0.5_run_manifest.txt
ff385d76c86e24f971ed50fb75933dce82d70f77cd5a4536206859f3f0f46bd4
```

## 6. 安全边界

- 全部新增和修改位于 `/data2/wangsiyi`；
- 未使用 `sudo`；
- 未下载 Qwen 模型权重；
- 未读取或修改正式数据集；
- 测试只使用合成数据和一张 RTX 3090；
- 测试后 GPU 1 没有遗留计算进程；
- GPU lock 可立即重新获取。

## 7. 结论与下一步

S0.1—S0.5 均已完成，单卡环境、依赖、运行包装器与暂停恢复机制已经具备。下一步骤为 **S1.1：数据集只读盘点**。
