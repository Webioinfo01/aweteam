# CCB 借鉴 Todo

**来源**: [SeemSeam/claude_codex_bridge](https://github.com/SeemSeam/claude_codex_bridge) v7.0.4
**对比项目**: aweteam v0.1.1
**原则**: 只借鉴符合 aweteam "intentionally small" 定位的特性，不引入 daemon、mailbox kernel 等重量级架构。

---

## TODO-1: 共享项目记忆

**优先级**: P0
**工作量**: 半天
**来源**: CCB `.ccb/ccb_memory.md`

aweteam worker 只能看到自己的 `task.md`，看不到其他 worker 的发现或项目级约定。

**做法**:
- run 目录下加 `shared.md`，leader 写入后所有 worker instructions 自动注入
- dispatcher 在 worker spawn 时读取 `shared.md` 内容，追加到 `instructions.md` 末尾
- leader 可以通过自然语言更新 shared memory（写文件指令）

**验证标准**:
- leader 写入 `shared.md` 后，新 spawn 的 worker 能看到内容
- 已运行的 worker 不受影响（不改已有 instructions）

---

## TODO-2: Worker 超时监控

**优先级**: P0
**工作量**: 1-2 天
**来源**: CCB heartbeat + keeper 系统（简化版）

dispatcher 目前只发通知，不监控 worker 健康状态。worker 卡死时无感知。

**做法**:
- dispatcher 定期检查每个 worker 的 `status.json` 更新时间
- 超过阈值（默认 10 分钟，可配置）标记 status 为 `timeout`
- 向 leader pane 发送超时通知（`tmux send-keys`）
- 不自动终止 worker，只通知（尊重用户控制权）

**验证标准**:
- 手动暂停一个 worker 的 provider CLI，10 分钟后 leader pane 收到超时通知
- worker 恢复后 status 自动回到正常

---

## TODO-3: Git Worktree 隔离选项

**优先级**: P1
**工作量**: 2-3 天
**来源**: CCB `agent:provider(worktree)` 语法

所有 worker 共享同一工作目录，存在文件互相覆盖的风险。

**做法**:
- profile 配置加可选字段 `"worktree": true`
- spawn worker 时自动 `git worktree add .aweteam/runs/<run-id>/wt/<worker-id>`
- worker 的工作目录设为 worktree 路径
- run 结束时 `git worktree remove` 清理
- 不加字段时行为不变（默认 `inplace`）

**验证标准**:
- 配置 `"worktree": true` 的 worker 在独立目录工作，git status 互不影响
- 未配置的 worker 行为不变
- run 结束后 worktree 被正确清理

---

## TODO-4: Worker → Leader 反馈通道

**优先级**: P1
**工作量**: 1 天
**来源**: CCB callback chain（简化版）

worker 只能在完成时写 `result.md`，中间无法反馈进度或请求澄清。

**做法**:
- worker 目录下加 `feedback.jsonl`（append-only）
- worker 的 instructions 中说明可以追加行到该文件来反馈
- dispatcher 监听 `feedback.jsonl` 变化，新行出现时转发到 leader pane
- 格式: `{"time": "...", "type": "progress|question", "body": "..."}`

**验证标准**:
- worker 追加一行到 `feedback.jsonl`，leader pane 显示该反馈
- 不影响 worker 正常完成流程和 `result.md` 写入

---

## TODO-5: Per-Agent Provider 状态隔离

**优先级**: P2（按需）
**工作量**: 3-5 天
**来源**: CCB managed homes

所有 Claude worker 共享 `~/.claude` 状态，session 可能互相干扰。

**做法**:
- spawn worker 时设置 `CLAUDE_CONFIG_DIR=.aweteam/runs/<run-id>/state/<worker-id>/claude`
- 或对 Claude provider 使用 `--settings` 指向隔离的 settings 文件
- Codex provider 使用 `CODEX_HOME` 隔离
- 先观察实际使用中是否遇到问题，有问题再实施

**验证标准**:
- 两个同 provider 的 worker 并行运行，session 互不干扰
- worker 完成后隔离状态被清理（或可选保留用于调试）

---

## 不做的事

| CCB 特性 | 原因 |
|---|---|
| Daemon 控制平面 (ccbd + Unix socket) | 与 "thin handoff" 定位冲突 |
| 完整 Mailbox 内核 | 文件交接模型已够用，mailbox 为 5+ provider 设计 |
| Rust 原生 Sidebar | 重量级依赖，tmux 多 pane 已提供可视性 |
| 多窗口拓扑 `[windows]` | 增加配置复杂度，单 session 足够 |
| Skill 继承系统 | CLAUDE.md/AGENTS.md 注入更简单 |
| 3 层配置解析 | 单 JSON 文件更符合项目定位 |
