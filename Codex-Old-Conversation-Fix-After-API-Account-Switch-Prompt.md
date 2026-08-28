# Fix Codex Old Conversations After Switching API or Account — AI Repair Prompt

> Author: 离子怪  
> Project: `codex-old-conversation-fix-after-api-account-switch`

## Important notice

This is an unofficial, community-maintained AI execution prompt, not an official OpenAI repair tool. OpenAI documentation describes how `model_provider` and custom providers are configured, but the rollout and SQLite task metadata addressed here are internal local-client structures and may change between Codex versions.

This prompt currently focuses on Codex Desktop for Windows. The executing Codex instance must begin with read-only diagnosis, inspect the actual database schema, and create recoverable backups. If any structure, field, or target task cannot be identified unambiguously, stop. Never apply fixed paths blindly or guess before writing.

## Overview

After switching an API, API proxy, or login account in Codex, an old conversation created by another API, account, or provider may no longer load or continue. A common cause is that the old task's saved `model_provider` is absent from the current user-level `config.toml` or does not match the currently available provider.

Attach this Markdown file to a Codex task that currently works and also provide the URI or ID of the task to repair. Codex should then locate the target task, identify the old and currently usable providers, create backups, update the two provider metadata locations in rollout and SQLite storage, validate the result, and roll back if the repair fails.

This prompt specifically handles `Model provider '<PROVIDER_ID>' not found` or a case confirmed by diagnosis to be inconsistent task provider metadata. If the actual cause is account access, plan limits, API balance, model authorization, or networking, stop the migration and report the real cause.

## Instructions for the Codex instance executing this file

Perform the repair directly. Do not merely give the user a list of commands to run.

The user must provide a target task URI or task ID when sending this file, for example:

```text
codex://threads/<THREAD_ID>
```

A typical error is shown below. The provider name is not fixed:

```text
Model provider '<PROVIDER_ID>' not found
```

The user's actual intent is to migrate the specified old task to a provider already working in the current Codex Desktop environment, so the old conversation can continue. Do not create a replacement task, copy only the visible chat, or modify project code.

Your objective is to repair a task that cannot continue because it references a nonexistent model provider. After repair, the target task must reload through the current valid provider while retaining its conversation, model, reasoning effort, working directory, and project files.

## Mandatory constraints

1. Perform diagnosis, backup, modification, and verification directly. Do not delegate command execution to the user.
2. Never read, print, echo, or copy an API key, token, password, or complete authentication object.
3. Do not modify source code or business files in the target project.
4. Do not delete the original task, the entire `.codex` directory, or any unrelated task.
5. Do not use `git reset --hard`, recursive deletion, or other destructive commands.
6. Before modifying anything, confirm the exact target task ID, rollout file, and state database.
7. Create a recoverable backup before any modification.
8. If the target task holds a writer lock or is currently running, stop and ask the user to close it first.
9. If the target provider cannot be determined from an explicit user choice or a usable reference task, ask the user. Never guess an API URL, provider name, or configuration key.
10. Provider selection and authentication method are separate concepts. Do not claim that changing the provider automatically switches ChatGPT/API-key authentication.

## Input resolution

Resolve parameters in the following order.

### Target task

Extract `THREAD_ID` from `codex://threads/<THREAD_ID>` in the user's message. The user may also provide the task ID directly.

If no target URI or ID was provided, ask the user for it first. Do not infer the target from its title, timestamp, or working directory, and do not modify any task.

Do not assume a fixed old provider. Identify it using the following evidence:

1. `threads.model_provider` in the target task's state database;
2. `session_meta.payload.model_provider` in the first entry of the target rollout;
3. The original error supplied by the user.

The provider stored in the database and rollout must match. If they differ, report the inconsistency first and proceed only through the cautious, backed-up repair flow. Do not assume that the provider named in the error is necessarily the actual database value.

### Target provider

The target is a provider that the current environment can already use successfully and, when requested, can use API billing. Determine it in this priority order:

1. A provider explicitly specified by the user;
2. The top-level `model_provider` in the current user-level `config.toml`;
3. If no top-level value is set, the built-in Codex default provider `openai`;
4. Cross-check the result against a recently successful task that is not the target old task;
5. If configuration and successful-task evidence conflict, or if the target provider is unavailable, stop and report the evidence instead of guessing.

If the currently configured provider is identical to the old provider in the error but its custom provider definition is missing, do not treat it as a valid target. Continue by checking the provider used by a recently successful new task. Ask the user only if a unique valid target still cannot be determined.

Do not infer a provider from the model name alone.

### Target model

Preserve the original model and reasoning effort by default. Modify the model only when the user explicitly requests it.

## Automated execution procedure

### Step 1: Determine the Codex data root

Prefer the current process's `CODEX_HOME`. If it is unset, use `.codex` under the current user's home directory.

Confirm that the following objects actually exist:

- The Codex data root;
- The `sessions` directory;
- A task state database;
- The rollout file for the target task.

Do not assume that the database is named `state_5.sqlite`. Search for `state_*.sqlite` under the Codex data root and use read-only queries to find the database whose `threads` table contains the target `THREAD_ID`.

If the target cannot be found, or multiple databases appear authoritative and cannot be distinguished, stop and report the evidence.

### Step 2: Read-only diagnosis

First inspect the schema without assuming that every Codex version uses identical columns:

```sql
PRAGMA table_info(threads);
```

After confirming which columns exist, read and record available fields from `threads`:

```text
id
rollout_path
model_provider
model
reasoning_effort
cwd
title
```

The fields `id`, `rollout_path`, and `model_provider` are central to this repair. If `model`, `reasoning_effort`, `cwd`, or `title` are absent from the current table, verify them from the rollout or another read-only metadata source. Do not invent column names or run a fixed query that references absent optional columns.

Verify that:

- `id` exactly matches the user-provided ID;
- `rollout_path` is inside the current Codex `sessions` directory;
- The rollout file exists;
- The database provider matches the provider in the error;
- `session_meta.payload.model_provider` in the first rollout entry matches the database.

Also run:

```powershell
codex login status
```

Record only the authentication type, such as `ChatGPT` or `API key`. Do not read or expose credential contents.

### Step 3: Decide whether metadata modification is appropriate

Inspect the provider IDs currently defined in the user-level `config.toml`, without printing any secret fields.

If the provider referenced by the target task actually exists, stop the direct metadata repair and instead diagnose configuration syntax, Base URL, or missing environment variables.

If the old provider does not exist and the target provider either exists or is the built-in `openai`, continue with migration.

If the target is a custom provider but the necessary `base_url`, `env_key`, or actual protocol support is unknown, stop and ask the user to supply non-secret configuration details. Code may use only clear placeholders such as:

```text
YOUR_PROVIDER_ID_HERE
YOUR_RESPONSES_API_BASE_URL_HERE
YOUR_API_KEY_ENV_NAME_HERE
```

### Step 4: Check the writer lock

Check:

```text
<CODEX_HOME>\thread-writer-locks\<THREAD_ID>.lock
```

Do not modify active task data when this lock exists.

### Step 5: Create consistent backups

Create a separate directory under the user's `Documents\Codex\backups`, containing the task ID and a timestamp, for example:

```text
thread-provider-<THREAD_ID>-yyyyMMdd-HHmmss
```

Back up at least:

1. The complete rollout JSONL;
2. The state database using SQLite `.backup`;
3. A short migration record containing the old provider, target provider, task ID, time, and file hashes, but no credentials.

When SQLite is using WAL mode, do not substitute an ordinary file copy for SQLite `.backup`.

Do not continue if backup creation fails.

### Step 6: Update the rollout

Modify only this field in the first rollout `session_meta` entry:

```text
payload.model_provider
```

Do not change any other field, history message, tool call, working directory, or task ID.

Safety requirements:

1. Write a temporary file in the same directory first;
2. Verify that the temporary file's first line parses as JSON;
3. Verify that the task ID did not change;
4. Verify that `payload.model_provider` equals the target provider;
5. Confirm that the old-provider marker matched as expected and that only the target field changed;
6. Use atomic replacement only after validation succeeds;
7. Always retain an independent backup of the original file.

If the old and target provider values have equal byte lengths, a fixed-length byte replacement may be used only after verifying a unique match, to preserve all other bytes.

If their lengths differ, rewrite only the first `session_meta` entry and preserve the remaining JSONL content as its original bytes.

Never run an unbounded global string replacement across the rollout.

### Step 7: Update the state database transactionally

Perform an exact update in a single SQLite transaction:

```sql
PRAGMA busy_timeout=10000;
BEGIN IMMEDIATE;

UPDATE threads
SET model_provider = '<TARGET_PROVIDER>'
WHERE id = '<THREAD_ID>'
  AND model_provider = '<OLD_PROVIDER>';

SELECT changes();
COMMIT;
```

`changes()` must be exactly `1`.

If the database update fails:

1. Restore the rollout from its backup;
2. Do not perform any additional writes;
3. Report the failure reason and the current consistency state.

### Step 8: Verify both storage locations

Read and compare again:

- Database `threads.model_provider`;
- Rollout `session_meta.payload.model_provider`.

Both values must equal the target provider.

Also verify that the following were not unintentionally changed:

- Task ID;
- Model;
- Reasoning effort;
- Working directory;
- All rollout content except the target field;
- Target project files.

### Step 9: Reload the task

If a Codex Desktop task-navigation tool is available, navigate to the target `THREAD_ID` so the app reloads it.

Otherwise tell the user to:

1. Close the target task tab;
2. Reopen the target task;
3. Fully quit and restart Codex Desktop only if the cached error remains.

Do not restart, kill, or clean up unrelated processes unless necessary.

## Authentication handling

After repairing the provider, run again:

```powershell
codex login status
```

The final report must distinguish:

- `Logged in using ChatGPT`: the current session still uses ChatGPT account entitlements;
- `Logged in using API key`: the current session uses usage-based official API-key billing;
- Custom proxy provider: its key is usually supplied by the environment variable named by `env_key`, and the true upstream cannot be inferred from `codex login status` alone.

If the user requires API billing but status still shows ChatGPT, do not tamper with `auth.json`. Use the official Codex login flow to switch credentials, or ask the user for the non-secret configuration details of the custom provider.

## Completion criteria

Report success only when every condition below is satisfied:

- The target task was uniquely identified;
- The target task has no writer lock;
- Both rollout and state database were backed up;
- The rollout provider was updated;
- The database provider was updated;
- Both provider values match exactly;
- The original model, reasoning effort, task ID, and working directory remain unchanged;
- No project source code was modified;
- The current authentication type was reported;
- The report states whether the user must restart Codex.

## Final report template

```text
Task provider repair completed/failed:

- Thread ID: <THREAD_ID>
- Old provider: <OLD_PROVIDER>
- New provider: <TARGET_PROVIDER>
- Model: <MODEL>
- Reasoning effort: <REASONING_EFFORT>
- Rollout verification: passed/failed
- SQLite verification: passed/failed
- Current authentication: ChatGPT/API key/custom provider/unknown
- Backup directory: <ABSOLUTE_BACKUP_PATH>
- Project source code: unchanged
- Next action: none/reopen task/restart Codex/complete provider configuration
```

Do not say only that the task "should be fixed." Report the actual database and rollout verification results.

## Official configuration references

- [Codex Advanced Configuration: Custom model providers](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex Configuration Reference: model_provider / model_providers](https://learn.chatgpt.com/docs/config-file/config-reference)

## License

MIT License. Use, modification, and redistribution are permitted when the copyright and license notice are retained.
