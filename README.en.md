# Fix Codex Old Conversations After Switching API or Account

**English** | [简体中文](./README.zh-CN.md)

> A safety-first AI repair prompt for Codex Desktop tasks that fail to load or continue with `Model provider '<PROVIDER_ID>' not found` after switching an API, account, proxy, or model provider.

**Author: 离子怪**

## What this project solves

An existing Codex Desktop task may retain the `model_provider` used when it was created. After switching a ChatGPT account, official API key, API proxy, or custom provider, the old task may no longer load or continue if its provider is absent from the current user-level `config.toml`.

This repository provides a Markdown prompt that can be attached to a working Codex task. It instructs Codex to:

1. Locate the target task from a user-supplied `codex://threads/<THREAD_ID>` URI;
2. Identify the old provider, current provider, and actual local data schema using read-only checks;
3. Check whether the task is running or holds a writer lock;
4. Back up the rollout JSONL and SQLite state database;
5. Repair provider metadata in both rollout and SQLite storage;
6. Verify that the task ID, model, reasoning effort, working directory, and conversation history were not unintentionally changed;
7. Roll back and report the actual state if any step fails.

## Quick start

1. Download or open [`Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md`](./Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md).
2. Attach it to a new Codex task that currently works.
3. Include the URI of the old task that needs repair:

```text
Follow the attached repair prompt exactly.

Target task:
codex://threads/<THREAD_ID>
```

4. Close the target old task if it is running to avoid modifying active state.
5. Let Codex complete diagnosis, backup, repair, and two-source verification.

## Matching symptoms

```text
Model provider '<PROVIDER_ID>' not found
```

Search terms:

- Codex model provider not found
- Codex Desktop old thread cannot continue
- Codex switch API old conversation
- Codex config.toml provider missing
- Codex rollout SQLite provider migration
- Codex old task after account switch
- Codex API proxy provider migration

## What this does not fix

Changing task provider metadata cannot resolve:

- Exhausted ChatGPT plan usage;
- Insufficient API balance or rate limits;
- Missing model access for the current account;
- An API proxy that does not support the Responses API;
- Base URL, network, certificate, or proxy failures;
- A corrupted task with no recoverable rollout or database record.

## Safety

> [!WARNING]
> This is an unofficial, community-maintained method. It modifies local Codex task metadata only after identification and backup. Codex database and rollout formats may change between versions. Do not skip schema inspection, lock checks, backups, or validation.

- Never upload or publish `auth.json`, API keys, tokens, a complete `config.toml`, or a real task database.
- Never delete, overwrite, or globally replace content across the entire `.codex` directory.
- Never modify rollout or SQLite records while the target task is running.
- If only a missing custom provider definition needs restoring, first consider restoring the user-level provider configuration instead.
- Stop if the observed data structure differs from the expected structure. Do not guess column names or paths.

## Technical basis and limitations

OpenAI documentation states that `model_provider` refers to a provider ID in `model_providers`, and that providers determine authentication, base URL, and wire protocol behavior. Provider settings belong in user-level configuration.

- [Advanced Configuration — Custom model providers](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Configuration Reference — model_provider](https://learn.chatgpt.com/docs/config-file/config-reference)

The official documentation does not describe direct rollout/SQLite mutation as a public repair interface. This project therefore treats it as a version-sensitive community recovery procedure.

## Repository contents

- [`README.md`](./README.md): bilingual landing page.
- [`README.zh-CN.md`](./README.zh-CN.md): Chinese documentation.
- [`Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md`](./Codex-Old-Conversation-Fix-After-API-Account-Switch-Prompt.md): complete English repair prompt.
- [`Codex切换API账号后旧对话无法操作-自动修复指令.md`](./Codex切换API账号后旧对话无法操作-自动修复指令.md): complete Chinese repair prompt.
- [`LICENSE`](./LICENSE): MIT License.

## Contributing

Issues and pull requests covering additional Codex versions, operating systems, and provider setups are welcome. Always redact task IDs, local paths, usernames, API endpoints, and credentials.

## Author

离子怪

## License

MIT License
