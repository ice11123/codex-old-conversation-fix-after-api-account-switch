# Codex 切换 API/账号后无法操作旧对话修复

## Fix Codex Old Conversations After Switching API or Account

[English](./README.en.md) | [简体中文](./README.zh-CN.md)

A safety-first AI repair prompt for Codex Desktop tasks that fail with `Model provider '<PROVIDER_ID>' not found` after switching an API, account, proxy, or model provider.

用于修复 Codex Desktop 在切换 API、登录账号、中转站或模型 Provider 后，旧任务出现 `Model provider '<PROVIDER_ID>' not found` 的安全 AI 指令。

## Choose a language / 选择语言

- [English documentation](./README.en.md)
- [简体中文说明](./README.zh-CN.md)

## Repair prompts / 修复指令

- [English AI repair prompt](./Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md)
- [中文 AI 自动修复指令](./Codex切换API账号后旧对话无法操作-自动修复指令.md)

The prompt guides Codex through target task identification, read-only diagnosis, writer-lock checks, recoverable backups, synchronized rollout/SQLite provider metadata repair, verification, and rollback on failure.

该指令会指导 Codex 自动识别目标任务、只读诊断、检查 writer lock、创建可恢复备份、同步修正 rollout/SQLite 两处 Provider 元数据、验证结果，并在失败时回滚。

> [!WARNING]
> This is an unofficial, community-maintained, version-sensitive recovery method. It modifies local Codex task metadata only after inspection and backup. Never expose credentials or skip validation.
>
> 这是社区维护、非官方且版本敏感的恢复方案。它只应在检查和备份后修改 Codex 本地任务元数据。请勿泄露凭证，也不要跳过校验。

**Author / 作者：离子怪**  
**License / 许可证：MIT**
