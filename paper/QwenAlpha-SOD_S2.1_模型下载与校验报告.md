# QwenAlpha-SOD：S2.1 模型下载与校验报告

**状态：** Passed

**完成时间：** 2026-10-29 19:19:53（Asia/Shanghai）

**模型：** `Qwen/Qwen-Image-2.1`

**模型目录：** `/data2/wangsiyi/models/Qwen-Image-2.1`

## 1. 下载来源与版本固定

- Hugging Face 官方提交：`d26bb61231c349cf6b7896fa83353113880e1ba3`。
- 下载通道：ModelScope 官方镜像。
- ModelScope revision：`master`。
- 下载支持断点续传，缓存位于 `/data2/wangsiyi/cache/modelscope`。
- 权重未加入公开 Git 仓库。

ModelScope 的 `master` 未提供与 Hugging Face 相同形式的固定提交号，因此本次实际下载内容使用逐文件 SHA-256 清单冻结。

## 2. 验收结果

| 检查项 | 结果 |
|---|---:|
| 普通文件数量 | 29 |
| 普通文件总字节数 | 33,134,789,600 |
| 成功解析的 JSON | 16 |
| safetensors 文件 | 7 |
| SHA-256 记录 | 29 |
| pipeline class | `QwenImage21Pipeline` |
| diffusers 本地配置加载 | 通过 |
| GPU 使用 | 否 |

以下组件均已验证存在且配置可解析：

- `processor`：Qwen3-VL processor/tokenizer。
- `scheduler`：FlowMatchEulerDiscreteScheduler。
- `text_encoder`：Qwen3-VL，4 个权重分片及索引。
- `transformer`：Qwen-Image-2.1 DiT，2 个权重分片及索引。
- `vae`：Qwen-Image-2.1 VAE 权重及配置。

所有权重索引引用的分片均存在；7 个 safetensors 文件均能读取非空 tensor 头。`QwenImage21Pipeline.load_config(..., local_files_only=True)` 验证通过，校验过程中未把模型权重加载到 GPU。

## 3. 远端证据

- 下载日志：`monitor/s2_1.log`
- 校验日志：`monitor/s2_1_verify.log`
- 结构化清单：`reports/S2.1_model_download_manifest.json`
- 逐文件哈希：`reports/S2.1_model_files.sha256`
- 完成标记：`state/done/S2.1.done.json`
- 状态文件：`state/progress.yaml`

证据文件摘要：

- `S2.1_model_download_manifest.json` SHA-256：`c5e1653f73b4f8cac09b02060ccdc841c0ced62c5493c9e29cc3d2a77ae9d597`
- `S2.1_model_files.sha256` SHA-256：`9915b5b5211e83e9b6835547ddba588695e368729e37708a1ab83384382f8c27`

## 4. 隔离与资源影响

- 所有新增远端内容均位于 `/data2/wangsiyi`。
- 未使用 `sudo`，未修改系统环境或其他用户目录。
- 校验仅使用 CPU、内存和本用户磁盘读取，没有占用 GPU。
- 未终止、修改或抢占其他用户进程。

## 5. 下一步

S2.1 已通过，安全暂停点为 `state/done/S2.1.done.json`。下一步是 S2.2：8 张单 GPU CPU-offload 推理。S2.2 涉及 RTX 3090，必须在用户明确启动后先检查 GPU 锁、可见设备和 22 GB 显存边界，不得自动运行。
