# QwenAlpha-SOD workspace instructions

This file applies to the entire repository. Harness and other execution agents must read it before taking any action.

## Local workspace

- Current Windows workspace: `D:\BaiduSyncdisk\CSCD\Qwenalpha-SOD`.
- Resolve repository-relative paths from this root. Do not use the former `D:\桌面\CSCD\Qwenalpha-SOD` path.
- The repository is inside a synchronization folder. Do not rely on Baidu Sync for Git operations, do not edit `.git` manually, and do not place model weights, datasets, caches, runs, or checkpoints in this local workspace.

## Execution contract

- Read `paper/QwenAlpha-SOD_Codex额度不足时Harness续执行计划书.md`, `paper/QwenAlpha-SOD_Harness与Codex共用进度说明书.md`, `paper/QwenAlpha-SOD_单RTX3090具体分步执行计划.md`, and `paper/QwenAlpha-SOD_非全监督主路线详细计划书.md` before continuing the research plan.
- Treat the user’s latest instruction as authoritative. Use remote state files, done markers, logs, and actual artifacts as the source of truth for progress.
- Execute only the single atomic step requested by the user. After verification and state updates, stop and report; do not automatically start the next step.
- Never repeat a step that already has valid completion evidence.

## Shared progress transaction

- `paper/QwenAlpha-SOD_Harness与Codex共用进度说明书.md` is the mandatory live handoff document for every agent.
- Read it before preflight. Do not rely on chat history as the current project state.
- In the same atomic step that changes remote state to `running`, `paused`, `failed`, `nogo`, or `passed`, update the live document's quick status, progress table, unresolved items, version, timestamp, and changelog as applicable.
- A completed step is not closed until remote state, `reports/NEXT_ACTION.md`, the monitor, and the live document agree with actual artifacts.
- Long-running per-file or per-batch progress belongs in the HTML monitor and logs. Do not create a watcher, polling loop, or Git commit for every progress tick.
- Resolve conflicts using this order: user instruction, actual artifacts/logs, remote state, live document, other plans. Record corrections in the live document instead of rewriting history silently.

## Remote host and isolation

- Connect with `ssh wangsiyi`.
- Remote project: `/data2/wangsiyi/qwenalpha-sod`.
- Every remote write, download, cache, log, temporary file, checkpoint, and result must remain under `/data2/wangsiyi`.
- Do not use `sudo`, modify system services or packages, access another user’s directory, or stop/change another user’s process.
- Resolve paths before destructive operations. Stop if a target escapes `/data2/wangsiyi`.

## Monitoring and long-running work

- Before every atomic execution step, update the task monitor to `ready` and give the user `http://127.0.0.1:8765/`.
- Then set the monitor to `running` and start the task. The remote monitor files are under `/data2/wangsiyi/qwenalpha-sod/monitor`.
- Run long tasks in a named `tmux` session and prefer event/exit-code waits. Avoid frequent polling and unnecessary status output.
- Do not restart a task merely because the monitor, SSH connection, or chat was interrupted. Inspect the existing process, state, log, shard, and checkpoint first.

## GPU safety

- GPU work is single-card RTX 3090 only and must use the project’s existing GPU wrapper/lock mechanism.
- Peak GPU memory must stay at or below 22 GB.
- Do not use `device_map=auto`, multi-GPU execution, or another user’s GPU allocation.
- If the required GPU is unavailable, record the step as paused and report it instead of competing for resources.

## State and scientific integrity

- Update state atomically using a temporary file followed by replacement.
- Create `state/done/<STEP>.done.json` only after all acceptance checks pass.
- Preserve failed logs, configurations, hashes, run IDs, seeds, checkpoints, and completed shards needed for recovery.
- Never fabricate, hide, selectively discard, or silently alter scientific results.

## GitHub synchronization

- The only GitHub synchronization target is `https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git` on branch `main`.
- Public synchronization is allowlist-only: root `AGENTS.md`, `paper/QwenAlpha-SOD_*`, selected legacy QwenAlpha documents, and `output/pdf/**/QwenAlpha-SOD_*`.
- Never publish datasets, model weights, checkpoints, runs, caches, temporary files, credentials, tokens, private logs, or unrelated project files.
- After a user-visible task creates or modifies an allowed public artifact and verification succeeds, run exactly once:

  ```powershell
  powershell -NoProfile -ExecutionPolicy Bypass -File ".\workspace-tools\sync-qwenalpha-artifacts-once.ps1"
  ```

- Do not create a scheduler, watcher, polling loop, or continuous background synchronization.
- If synchronization fails, preserve local work and report the failure. Do not force-push or switch to another repository.
