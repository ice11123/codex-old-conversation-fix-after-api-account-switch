# Codex 切换 API/账号后无法操作旧对话修复

[English](./README.en.md) | **简体中文**

> 修复 Codex Desktop 切换 API、账号或模型 Provider 后，旧任务出现 `Model provider '<PROVIDER_ID>' not found`、无法读取或无法继续工作的安全 AI 指令。

**作者：离子怪**

## 解决什么问题

Codex 桌面端的旧任务可能保存了创建时使用的 `model_provider`。切换 ChatGPT 登录账号、官方 API Key、API 中转站或自定义 Provider 后，如果当前用户级 `config.toml` 已不再定义旧 Provider，旧任务可能无法重新加载或继续。

本仓库提供一份可以直接交给当前正常工作的 Codex 执行的 Markdown 指令，指导它：

1. 根据用户提供的 `codex://threads/<THREAD_ID>` 定位目标任务；
2. 只读识别原 Provider、当前可用 Provider 和真实数据结构；
3. 检查任务是否仍在运行或持有 writer lock；
4. 备份 rollout JSONL 与 SQLite 状态数据库；
5. 同步修正 rollout 和 SQLite 中的 Provider 元数据；
6. 验证任务 ID、模型、推理等级、工作目录和历史内容未被意外修改；
7. 任一步失败时回滚并报告实际状态。

## 快速使用

1. 下载或打开 [`Codex切换API账号后旧对话无法操作-自动修复指令.md`](./Codex切换API账号后旧对话无法操作-自动修复指令.md)。
2. 在当前能够正常工作的 Codex 新任务中附上该文件。
3. 同时发送需要修复的旧任务 URI：

```text
请严格执行附件中的修复指令。

目标任务：
codex://threads/<THREAD_ID>
```

4. 关闭正在运行的目标旧任务，避免热修改。
5. 等待 Codex 完成诊断、备份、修复和双重校验。

## 适用症状

```text
Model provider '<PROVIDER_ID>' not found
```

相关搜索关键词：

- Codex model provider not found
- Codex Desktop old thread cannot continue
- Codex switch API old conversation
- Codex config.toml provider missing
- Codex rollout SQLite provider migration
- Codex 切换 API 后旧对话无法继续
- Codex 切换账号后历史任务打不开
- Codex 中转站 Provider 迁移

## 不适用的情况

以下问题不能通过修改任务 Provider 元数据解决：

- ChatGPT 套餐额度耗尽；
- API 余额不足或 Rate Limit；
- 当前账号没有目标模型权限；
- API 中转站不兼容 Responses API；
- Base URL、网络、证书或代理故障；
- 任务本身损坏且不存在可恢复的 rollout/数据库记录。

## 安全说明

> [!WARNING]
> 这是社区维护的非官方方案。它会在确认和备份后修改 Codex 本地任务元数据。Codex 内部数据库与 rollout 格式可能随版本变化，请勿跳过结构检查、锁检查、备份和校验。

- 不要上传或公开 `auth.json`、API Key、Token、完整 `config.toml` 或真实任务数据库。
- 不要对整个 `.codex` 目录执行删除、覆盖或全局替换。
- 不要在目标任务仍运行时修改其 rollout 或 SQLite 记录。
- 如果只需恢复一个缺失的自定义 Provider，应优先评估重新补全用户级 Provider 配置是否更合适。
- 数据结构与文档预期不一致时必须停止，不要猜测字段名和路径。

## 技术依据与边界

OpenAI 官方文档说明：`model_provider` 指向 `model_providers` 中的 Provider ID，Provider 与认证、Base URL 和请求协议相关；Provider 配置应位于用户级配置中。

- [Advanced Configuration — Custom model providers](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Configuration Reference — model_provider](https://learn.chatgpt.com/docs/config-file/config-reference)

官方文档没有把直接修改 rollout/SQLite 元数据描述为标准公开修复接口，因此本项目明确将其标记为版本敏感的社区应急方案。

## 仓库内容

- [`README.md`](./README.md)：双语入口。
- [`README.en.md`](./README.en.md)：英文说明。
- [`Codex切换API账号后旧对话无法操作-自动修复指令.md`](./Codex切换API账号后旧对话无法操作-自动修复指令.md)：中文完整修复指令。
- [`Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md`](./Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md)：英文完整修复指令。
- [`LICENSE`](./LICENSE)：MIT License。

## 贡献

欢迎提交 Issue 或 Pull Request，补充不同 Codex 版本、操作系统和 Provider 场景下的验证结果。请务必清除任务 ID、本机路径、用户名、API 地址和任何凭证。

## 作者

离子怪

## License

MIT License
