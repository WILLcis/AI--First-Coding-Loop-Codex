# handoffs/ — 接力包模板

> 接力包(handoff)= 你写给"代你执行的 agent"(通常是 Codex)的一份**自包含任务规约**。
> 它的价值就在 **maker/checker 分离**:你是 checker,Codex 是 maker;
> 它沿着接力包跑、在每个 🛑 检查点停下来等你拍板。

## 用法

把对应模板**整体**贴进 Codex 第一条消息,或者:
```
请读 ~/path/to/handoff-xxx.md 并按它的指令从检查点 0 开始,
严格遵守所有 🛑 检查点。
```

## 模板清单

| 文件 | 项目预设 | LLM 预设 | 适用 |
|---|---|---|---|
| `handoff-new-project-template.md` | **未预填**(检查点 0 问用户) | **未预填**(检查点 0 问用户) | 任意新仓库 + 任意 LLM |
| `handoff-upgrade-existing-project-to-codex-v2.6.md` | **未预填**(检查点 0 问用户) | 复用现有或检查点 0 指定 | 已在开发/上线项目升级到最新 Codex harness |
| `examples/handoff-openai-gpt5.5.md` | 未预填 | **OpenAI / GPT-5.5** | 想用 OpenAI 的项目,省一些配置思考 |

## 写自己的接力包(给某个特定项目)

复制 `handoff-new-project-template.md`,在前几节填上具体值即可:
- 项目名
- 仓库 URL
- LLM 厂商 + 模型
- 想跳过的检查点(不建议跳)

文件名约定:`handoff-<project>-bootstrap.md`(下次 onboarding 同事或新机器,直接复用)。
