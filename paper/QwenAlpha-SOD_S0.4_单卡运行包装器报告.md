# QwenAlpha-SOD S0.4：单卡运行包装器报告

> 执行日期：2026-10-27  
> 远端主机：`wangsiyi`  
> 远端工程：`/data2/wangsiyi/qwenalpha-sod`  
> 结果：**✅ Passed（9/9）**

## 1. 包装器接口

远端实现：

```text
/data2/wangsiyi/qwenalpha-sod/scripts/run_single_gpu.sh
```

统一调用格式：

```bash
/data2/wangsiyi/qwenalpha-sod/scripts/run_single_gpu.sh <物理GPU编号> -- <命令> [参数...]
```

例如使用物理 GPU 1：

```bash
/data2/wangsiyi/qwenalpha-sod/scripts/run_single_gpu.sh 1 -- \
  /data2/wangsiyi/envs/qwenalpha-sod/bin/python your_program.py
```

包装器要求 `GPU编号` 后必须出现 `--`，因此不会把第二个 GPU 编号误当成命令。

## 2. 已实现的保护

- 只接受一个非负整数物理 GPU 编号；
- 显式拒绝 `0,1` 等逗号分隔参数；
- 显式拒绝 `1 0 -- command` 等两个 GPU 参数；
- 用 `nvidia-smi` 验证 GPU 存在；
- 在启动前读取该卡计算进程 PID，忙卡立即拒绝；
- 设置 `CUDA_DEVICE_ORDER=PCI_BUS_ID` 和唯一的 `CUDA_VISIBLE_DEVICES`；
- 在执行用户命令前运行 PyTorch 单卡断言；
- 使用 `flock` 锁定 `state/locks/gpu-<编号>.lock`；
- 命令成功、失败、被拒绝或收到中断信号时均写运行摘要；
- 原样传递用户命令退出码；
- 无论成功或失败均释放项目锁；
- 运行摘要不记录完整命令参数，避免把潜在凭据写入日志。

运行摘要写入：

```text
/data2/wangsiyi/qwenalpha-sod/runs/launcher_summaries/
```

## 3. 测试结果

测试使用物理 GPU 1，完整结果为：

| 用例 | 预期 | 结果 |
|---|---:|---|
| 无 GPU 参数 | 64 | PASS |
| 逗号分隔两个 GPU | 64 | PASS |
| 两个独立 GPU 参数 | 64 | PASS |
| 正常空闲卡 | 0 | PASS |
| Python 异常退出 | 7 | PASS |
| 异常后再次运行 | 0 | PASS，证明锁已释放 |
| 项目锁竞争 | 73 | PASS |
| 真实 CUDA 忙卡 | 75 | PASS |
| 成功、失败和拒绝摘要 | 均存在 | PASS |

总计：**9 passed，0 failed**。

正例探针确认：

```text
physical=1
CUDA_VISIBLE_DEVICES=1
torch.cuda.device_count()=1
logical_device=cuda:0
name=NVIDIA GeForce RTX 3090
```

测试结束后 `nvidia-smi` 未发现该测试留下的计算进程，GPU lock 可立即重新获取。

## 4. 测试过程中的修正

第一次测试有 7 项通过、2 项失败。原因是测试夹具使用 `flock -c 'sleep 20'` 模拟锁竞争，终止 `flock` 父进程后，`sleep` 子进程继承了锁文件描述符，导致后续忙卡测试被错误判定为锁冲突。

修正仅针对测试夹具：改为单个 Python 进程直接持有 `fcntl` 文件锁，终止该 PID 即释放锁。包装器实现没有因该问题改变。修正后同一测试集 9/9 通过。

## 5. 文件与哈希

```text
scripts/run_single_gpu.sh
23de9de32f882e3117aa6cf0d191a28f06d0cd88bd422495bf45331141838990

tests/test_run_single_gpu.sh
8134a6c87a236e8987c9498bfc02c3976e45d30fd03ceae9b14eb91d77427054

tests/gpu_probe.py
f8015fdfde43191a24fa0f2368b32d55a156470a4a583538f7ba06bc006c924f

reports/S0.4_wrapper_test_results.txt
5d772c1f9bda1ed70b92d2fc9860c3ed95ee5ab9da89385b43bb9bba6de2f49f
```

## 6. 边界说明

- 所有新增和修改均位于 `/data2/wangsiyi`；
- 未使用 `sudo`；
- 未终止他人进程；
- 忙卡测试只终止本测试启动且已记录 PID 的短生命周期进程；
- 未下载 Qwen 模型权重；
- 未进行训练；
- 全程仅使用一张 RTX 3090。

## 7. 下一步骤

下一步骤为 **S0.5：建立暂停恢复单元测试**。任何后续 GPU 命令都必须通过 `scripts/run_single_gpu.sh` 启动。
