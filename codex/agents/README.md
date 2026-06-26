# Codex Custom Agents(v2.1)

> 把"评审者"扩展为完整角色分工:explorer / implementer / 三类 verifier / triage-scorer / checker。
> 关键原则:**写代码的 agent 不能是判断 done 的 agent**(Addy:maker/checker split)。
> Codex custom agent TOML 必须包含 `name`、`description`、`developer_instructions`;
> 模型默认继承当前 Codex 会话,每个 agent 只固定自己的推理深度与职责边界。

## 在你的项目里怎么放

- **Codex**:整体移到 `.codex/agents/`
- **自建编排**:可以直接读取本目录 TOML 的 `name`、`description`、`developer_instructions` 与 `model_reasoning_effort`

## 模型分层原则(省钱 + 提质两条都顾)

| 角色 | 推理深度 | 为什么 |
|---|---|---|
| `explorer` | low | 只读探查,快且便宜,不需要深思考 |
| `implementer` | medium | 写码主力,需要计划、编辑与验证 |
| `verifier-quality` | medium | 关心逻辑/性能/可维护性 |
| `verifier-security` | **high** | 安全错一次代价最大,这里值得花推理预算 |
| `verifier-dependency` | low | 形态简单(版本/许可证),靠规则 + 轻量推理已足够 |
| `triage-scorer` | medium | 错误簇打分需要稳且可解释 |
| `checker` | medium | goal_loop 的 done 判定,需要稳但不必最高 |

## 当前 agents

| name | 典型触发 |
|---|---|
| explorer | architect-task-writer 调用前先探查;pr-investigator 第 1 拍 |
| implementer | 写实现代码 + 自带测试 |
| verifier-quality | ai-review.yml 第 1 趟 |
| verifier-security | ai-review.yml 第 2 趟 |
| verifier-dependency | ai-review.yml 第 3 趟 |
| triage-scorer | triage_engine.py 内部调用 |
| checker | goal_loop.py 的 done 判定 |

## 加一个新 agent

1. 写 TOML 文件:`agents/<name>.toml`,至少包含 `name`、`description`、`developer_instructions`
2. 在本 README 表格里加一行
3. 如果新 agent 替换了已有调用点,改 workflow / 脚本里的引用
4. 跑 `bash tools/verify.sh`(检查 TOML 与 workflow 基本合法)
