# Codex 切换 API/账号后旧对话无法操作——自动修复指令

> 作者：离子怪  
> 项目：`codex-old-conversation-fix-after-api-account-switch`

## 重要声明

这是一份社区维护的非官方 AI 执行指令，不是 OpenAI 官方修复工具。OpenAI 官方文档公开说明了 `model_provider` 与自定义 Provider 的配置方式，但本文涉及的 rollout 和 SQLite 任务元数据属于本地客户端内部数据结构，可能随 Codex 版本变化。

本指令当前主要面向 Windows 上的 Codex 桌面端。执行者必须先只读诊断、检查实际数据库结构并创建可恢复备份；任何结构、字段或目标任务无法唯一确认时，都必须停止，禁止套用固定路径或盲目写入。

## 简介

Codex 在切换 API、API 中转站或登录账号后，由其他 API、账号或 Provider 创建的旧对话，可能无法在当前环境中读取或继续工作。常见原因是旧任务保存的 `model_provider` 已不在当前用户级 `config.toml` 中，或与当前可用 Provider 不一致。

把本 Markdown 文档交给当前能够正常运行的 Codex，并同时告诉它需要修复的目标任务 URI 或任务 ID，Codex 即可按照本文指令自动定位目标任务、识别原 Provider 与当前可用 Provider、创建备份、同步修正 rollout 和 SQLite 中的两处 Provider 元数据、执行完整校验，并在修改失败时回滚。

本指令专门处理 `Model provider '<PROVIDER_ID>' not found` 或经诊断确认属于任务 Provider 元数据不一致的情况。若实际错误是账号权限、套餐额度、API 余额、模型权限或网络问题，应停止迁移并报告真实原因。

## 给执行本文件的 Codex

你需要直接完成修复，而不是只给出操作建议。

用户需要在发送本文件时，同时提供目标任务 URI 或任务 ID，例如：

```text
codex://threads/<THREAD_ID>
```

典型错误如下，其中 Provider 名称并不固定：

```text
Model provider '<PROVIDER_ID>' not found
```

用户的真实意图是：把指定旧任务迁移到当前 Codex 桌面端新任务已经能够正常使用的 Provider，使旧对话能够继续，而不是新建任务、复制聊天内容或改项目代码。

你的目标是修复该任务因引用不存在的模型 Provider 而无法继续的问题。典型错误为：

```text
Model provider '<PROVIDER_ID>' not found
```

修复完成后，目标任务必须能够由当前有效 Provider 重新加载，同时保留原对话、模型、推理等级、工作目录和项目文件。

## 必须遵守的约束

1. 直接执行诊断、备份、修改和验证；不要把命令丢给用户自己执行。
2. 不得读取、打印、回显或复制任何 API Key、Token、密码和完整认证对象。
3. 不得修改目标项目的代码或业务文件。
4. 不得删除原任务、整个 `.codex` 目录或任何无关任务。
5. 不得使用 `git reset --hard`、递归删除或其他破坏性命令。
6. 修改前必须确认准确的目标任务 ID、rollout 文件和状态数据库。
7. 修改前必须创建可恢复备份。
8. 如果目标任务存在 writer lock 或正在运行，停止修改并要求用户先关闭该任务。
9. 如果目标 Provider 无法从用户要求或可用参考任务中确定，必须询问用户，禁止猜测 API URL、Provider 名称或配置键。
10. Provider 与认证方式是两个独立概念；不得声称修改 Provider 会自动切换 ChatGPT/API Key 认证。

## 输入解析

按以下顺序确定参数：

### 目标任务

从用户消息中的 `codex://threads/<THREAD_ID>` 提取 `THREAD_ID`；用户也可以直接提供任务 ID。

如果用户没有提供任务 URI 或 ID，先要求用户补充，不要通过标题、时间或工作目录猜测目标任务，也不要修改任何任务。

原 Provider 不预设固定值。必须从以下证据中识别：

1. 目标任务状态数据库中的 `threads.model_provider`；
2. 目标 rollout 第一条 `session_meta.payload.model_provider`；
3. 用户提供的原始报错。

数据库和 rollout 中的原 Provider 必须一致。若二者不一致，先报告不一致状态并以备份后的谨慎修复流程处理；不得直接假定报错中的 Provider 就是数据库真实值。

### 目标 Provider

目标是当前新任务正在正常使用、且可走 API 计费的 Provider。按以下优先级自动确定：

1. 用户明确指定的 Provider；
2. 当前用户级 `config.toml` 顶层 `model_provider`；
3. 若顶层未设置，则按 Codex 默认值使用内置 Provider `openai`；
4. 用一个最近成功运行、且不是目标旧任务的任务记录交叉验证当前 Provider；
5. 如果配置值、成功任务记录互相冲突，或目标 Provider 本身不可用，停止并报告证据，不要猜测。

如果当前配置的 Provider 与目标任务报错中的旧 Provider 相同，但对应的自定义 Provider 定义不存在，则不能把它当作有效目标 Provider。此时应继续查找最近成功运行的新任务所用 Provider；仍不能唯一确定时才询问用户。

不要仅凭模型名推断 Provider。

### 目标模型

默认保留原任务的模型和推理等级。只有用户明确要求时才修改模型。

## 自动执行流程

### 第一步：确认 Codex 数据根目录

优先读取当前进程的 `CODEX_HOME`；未设置时使用当前用户目录下的 `.codex`。

确认以下对象真实存在：

- Codex 数据根目录；
- `sessions` 目录；
- 任务状态数据库；
- 目标任务对应的 rollout 文件。

不要固定假设数据库一定叫 `state_5.sqlite`。搜索 Codex 数据根目录中的 `state_*.sqlite`，只读查询具有 `threads` 表且包含目标 `THREAD_ID` 的数据库。

如果没有找到目标任务，或多个数据库都像是当前权威数据库但无法区分，停止并报告证据。

### 第二步：只读诊断

先执行以下只读结构检查，禁止假定所有 Codex 版本具有完全相同的列：

```sql
PRAGMA table_info(threads);
```

确认真实存在的列后，从 `threads` 表读取并记录可用字段：

```text
id
rollout_path
model_provider
model
reasoning_effort
cwd
title
```

其中 `id`、`rollout_path` 和 `model_provider` 是本修复流程需要重点确认的字段。如果 `model`、`reasoning_effort`、`cwd` 或 `title` 不在当前数据库表中，应从 rollout 或其他只读元数据来源核对，不得因可选列缺失而臆造列名或直接执行失败的固定查询。

验证：

- `id` 与用户提供的 ID 完全一致；
- `rollout_path` 位于当前 Codex 的 `sessions` 目录内；
- rollout 文件存在；
- 数据库中的 Provider 与报错 Provider 一致；
- rollout 第一条 `session_meta.payload.model_provider` 与数据库一致。

同时执行：

```powershell
codex login status
```

只记录认证类型，例如 `ChatGPT` 或 `API key`，不要读取或输出凭证内容。

### 第三步：判断是否需要修改

检查用户级 `config.toml` 中当前已定义的 Provider ID，但不要输出其中任何秘密字段。

如果目标任务引用的 Provider 实际已经存在，应停止直接修改，转为检查配置语法、Base URL 或环境变量缺失问题。

如果旧 Provider 不存在，而目标 Provider 已存在或是内置 `openai`，继续迁移。

如果目标是自定义 Provider，但缺少 `base_url`、`env_key` 或真实协议支持信息，停止并要求用户提供。代码中只能使用如下占位符：

```text
YOUR_PROVIDER_ID_HERE
YOUR_RESPONSES_API_BASE_URL_HERE
YOUR_API_KEY_ENV_NAME_HERE
```

### 第四步：检查写入锁

检查：

```text
<CODEX_HOME>\thread-writer-locks\<THREAD_ID>.lock
```

存在时禁止热修改。

### 第五步：创建一致性备份

在用户的 `Documents\Codex\backups` 下建立带任务 ID 和时间戳的独立目录，例如：

```text
thread-provider-<THREAD_ID>-yyyyMMdd-HHmmss
```

至少备份：

1. 完整 rollout JSONL；
2. 使用 SQLite `.backup` 创建的状态数据库备份；
3. 一份简短的迁移记录，包含原 Provider、目标 Provider、任务 ID、时间和文件哈希，不包含凭证。

不要在数据库正在使用 WAL 时仅用普通文件复制代替 SQLite `.backup`。

备份失败时禁止继续。

### 第六步：修改 rollout

只修改 rollout 第一条 `session_meta` 中的：

```text
payload.model_provider
```

不得修改其他字段、历史消息、工具调用、工作目录或任务 ID。

安全要求：

1. 先写入同目录临时文件；
2. 验证临时文件第一行能解析为 JSON；
3. 验证临时文件的任务 ID 未变化；
4. 验证 `payload.model_provider` 等于目标 Provider；
5. 原 Provider 标记必须符合预期且只修改目标字段；
6. 验证成功后使用原子替换；
7. 原文件始终有独立备份。

如果原 Provider 与目标 Provider 字节长度相同，可以使用经过唯一匹配计数校验的定长字节替换，以最大限度保持文件其他内容不变。

如果长度不同，只重写第一条 `session_meta`，其余 JSONL 内容必须按原始字节保留。

禁止对整个 rollout 执行未经限定的全局字符串替换。

### 第七步：事务更新状态数据库

在单个 SQLite 事务内执行精确更新：

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

`changes()` 必须恰好为 `1`。

如果数据库更新失败：

1. 用备份恢复 rollout；
2. 不要继续其他写入；
3. 报告失败原因和当前一致性状态。

### 第八步：双重校验

重新读取并比较：

- 数据库 `threads.model_provider`；
- rollout `session_meta.payload.model_provider`。

两者必须等于目标 Provider。

同时验证以下字段没有意外变化：

- 任务 ID；
- 模型；
- 推理等级；
- 工作目录；
- rollout 除目标字段外的内容；
- 目标项目文件。

### 第九步：重新加载任务

如果有可用的 Codex 桌面端任务导航工具，导航到目标 `THREAD_ID`，让应用重新加载任务。

否则提示用户：

1. 关闭目标任务页签；
2. 重新打开目标任务；
3. 如果仍显示缓存错误，完全退出并重新启动 Codex 桌面端。

不要在没有必要时重启、杀死或清理其他进程。

## 认证方式处理

修复 Provider 后再次执行：

```powershell
codex login status
```

必须在最终报告中明确说明：

- `Logged in using ChatGPT`：当前仍使用 ChatGPT 账号权益；
- `Logged in using API key`：当前使用官方 API Key 按量计费；
- 自定义中转 Provider：其 Key 通常由 `env_key` 指定的环境变量提供，不能仅凭 `codex login status` 推断真实上游。

如果用户要求 API 计费，但当前仍显示 ChatGPT，不要篡改 `auth.json`。应使用 Codex 官方登录流程切换凭证，或要求用户提供其自定义 Provider 的非秘密配置信息。

## 完成标准

只有全部满足以下条件才可报告成功：

- 目标任务定位唯一；
- 目标任务没有 writer lock；
- rollout 和状态数据库均已备份；
- rollout Provider 已修改；
- 数据库 Provider 已修改；
- 两处 Provider 完全一致；
- 原模型、推理等级、任务 ID 和工作目录保持不变；
- 没有修改项目源代码；
- 已报告当前认证类型；
- 已说明是否需要用户重启 Codex。

## 最终报告格式

```text
任务 Provider 修复已完成/失败：

- Thread ID：<THREAD_ID>
- 原 Provider：<OLD_PROVIDER>
- 新 Provider：<TARGET_PROVIDER>
- 模型：<MODEL>
- 推理等级：<REASONING_EFFORT>
- rollout 校验：通过/失败
- SQLite 校验：通过/失败
- 当前认证：ChatGPT/API key/自定义 Provider/无法确认
- 备份目录：<ABSOLUTE_BACKUP_PATH>
- 项目源代码：未修改
- 后续动作：无需操作/重新打开任务/重启 Codex/补充 Provider 配置
```

不要只说“应该已经修好”。必须给出数据库和 rollout 的实际校验结果。

## 官方配置参考

- [Codex Advanced Configuration：Custom model providers](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [Codex Configuration Reference：model_provider / model_providers](https://learn.chatgpt.com/docs/config-file/config-reference)

## 许可证

MIT License。允许使用、修改与再分发，但请保留版权和许可声明。
