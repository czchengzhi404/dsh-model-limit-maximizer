---
name: model-limit-maximizer
description: 当用户导入或添加了自定义模型后，自动将 contextWindow 和 maxTokens 校准为该路由下经过验证的可用最高值。触发词包括“导入模型”“上下文没拉满”“输出上限没设置”“模型能力没填全”“模型参数填充”“contextWindow 太小”“maxTokens 是空的”“模型导入后要不要手动改”“校准模型上限”。
---

# 模型上限自动校准

## 目标

DSH 的模型条目如果没有显式 `contextWindow` / `maxTokens`，会回退到路由的保守默认值（`dsh-llm-pi-ai` 默认 262144 / 32768），而不是模型本身的上限。本 Skill 在模型导入后，把这两个字段校准为**该路由可用最高值**：以 models.dev 理论值为基线，与提供方实际限制取较小者，写入前做一次最小探测，最终写入 DSH 配置补丁层。

核心原则：**默认不覆盖已有配置**；字段缺失、为空，或用户显式要求“强制更新”时才写入。

## 配置真相

DSH 的模型配置不在 `settings.yaml`，而在 profile 目录下：

- 生效配置：`~/.dsh/profiles/<profile>/cordis.yml`
- 用户补丁层（**要改这个文件**）：`~/.dsh/profiles/<profile>/cordis.patch.yml`

桌面应用对应 `desktop`，`dsh web` 对应 `web`。模型条目位于：

- `llm-pi-ai` 插件：`config.providers.<provider>.models[]`
- DeepSeek 直连适配器（`dsh-llm`）：`config.models[]`（较少见）

每个模型条目是 YAML 对象，常用字段：`id`、`name`、`contextWindow`（整数 token 数）、`maxTokens`（整数 token 数）、`input`、`reasoningEfforts`。**配置里是整数，不是 `256K` 这种写法**（只有设置页 UI 用 K/M 简写）。同一个模型 id 出现在多个 provider 时，按 provider 路由分别校准——不同路由的可用上限可能不同。

## 工作流

### 0. 读取当前配置

1. 确认当前 profile：默认 `desktop`；用户提到 `dsh web` 时用 `web`。
2. 读 `cordis.patch.yml`，在 `llm-pi-ai.config.providers.<provider>.models` 中定位目标模型。patch 层没有该 provider 时，读 `cordis.yml` 确认继承自 bundle 的值。
3. 记录该条目的 `contextWindow` / `maxTokens` 现值（缺失 / 空 / 数值）。

完成标准：能说出目标模型在哪个 provider，两个字段当前是“缺失 / 空 / 数值 X”。

### 1. 查询 models.dev 基线

请求 `https://models.dev/api.json`（按 provider 分组的模型表）。查找目标模型 id，按优先级：

1. **精确匹配**：`data[provider].models[id]`
2. **规范 id**：`data[provider].models["<厂商>/<id>"]`，如 `deepseek/deepseek-v4-pro`
3. **家族推断**：裸 id（如 `deepseek-v4-flash`）匹配同族快照（如 `deepseek-v4-flash-ga-260731`），取其中最大的 `limit.context` / `limit.output`

记录 `limit.context`（理论上下文）和 `limit.output`（理论输出上限）。`limit.output` 为 0 表示该模型不产文本（embedding 等），没有输出基线。

查不到基线 → 标记“无基线”，步骤 2 只依赖提供方数据；提供方也没有 → 跳过该模型并报告原因。

完成标准：拿到 models.dev 基线、提供方数据、或“无基线”标记三者之一。

### 2. 读取提供方真实限制

按优先级取第一个可信来源：

**a. 提供方 `/v1/models`**：`GET {baseURL}/models`（baseURL 来自该 provider 的配置），在模型条目中找 `context_length` / `contextWindow` / `max_context_length` 和 `max_output_tokens` / `max_completion_tokens` / `maxTokens`。大多数 OpenAI 兼容端点不返回这些字段——这是正常的，DSH 的 `discoverModels` 也只带回端点披露的值，这正是本 Skill 要补的洞。

**b. 本地服务自报**：Ollama / LM Studio / one-api 等本地路由，优先用服务实际报告的 `num_ctx` / `max_tokens`，不用 models.dev 理论值。

**c. 官方文档 / 用户已有配置**：同 provider 其他条目填过的值、或提供方文档明确给出的截断值（如某中转站把上下文截到 128K），以该值为准。

完成标准：每个字段都标出“有提供方值”或“无提供方值”。

### 3. 计算最终值

```
contextWindow = min(models.dev context, provider context)   # 单一来源直接用该来源
maxTokens     = min(models.dev output, provider output)     # 单一来源直接用该来源
```

对 `maxTokens` 施加安全边际：

```
maxTokens = floor(maxTokens × 0.85 / 1024) × 1024
```

- `maxTokens` 小于 4096 → 回退 4096。
- 本地小模型且明确自报了更低上限 → 以自报值为准，不加边际。
- **边际只加在输出上限，不加在上下文窗口**：上下文超限报错明确、容易回退；输出上限过高可能在生成中途失败，更难排查。

完成标准：算出两个整数，且知道每个值来自哪个来源。

### 4. 写入前探测

用目标 `baseURL` 发一次最小请求：

```
POST {baseURL}/chat/completions
{
  "model": "<model-id>",
  "messages": [{"role": "user", "content": "hi"}],
  "max_tokens": 16
}
```

- 返回 2xx → 路由可用，继续。
- 返回 400 且错误与 `max_tokens` / context 相关 → 按 `当前值 → 64000 → 32000 → 16000 → 8192` 降档重试（`contextWindow` 不参与降档）。
- 降到 8192 仍失败 → 不写入，报告“该路由拒绝了所有尝试的输出上限，需人工检查”。
- 返回 401/403（凭据不可用）或网络不可达 → **不阻塞写入**：这是配置层补漏，不是运行时验证。写入 models.dev 基线值，报告中明确标注“未验证”。

完成标准：探测通过，或明确记录降档后的值，或标注“未验证”及原因。

### 5. 写入配置

1. **先备份**：把 `cordis.patch.yml` 复制为 `cordis.patch.yml.bak-<时间戳>`。
2. 只修改目标条目的 `contextWindow` 和 `maxTokens`，不动 `name`、`input`、`reasoningEfforts`、`api`、`baseURL` 等字段。
3. 写入规则：
   - 字段缺失或为空 → 写入计算值。
   - 字段已有且低于计算值，用户未要求强制更新 → **保留原值**，报告“当前值 X 低于可用值 Y，如需更新请说强制更新”。
   - 字段已有且低于计算值，用户要求强制更新 → 覆盖。
   - 字段已有且 ≥ 计算值 → 保留。
4. 保持 YAML 缩进与现有结构一致；patch 文件里可能混有 `!!js` 表达式，不要改动它们。

完成标准：目标条目只有 `contextWindow` / `maxTokens` 发生变化，其余内容不动；备份文件已生成。

### 6. 报告与验证

给出一张对照表：

| 模型 | provider | contextWindow 旧 → 新 | maxTokens 旧 → 新 | 状态 |
|---|---|---|---|---|

状态 ∈ {已填充，已跳过（已有配置），已强制更新，无基线跳过，探测失败未写入，未验证（无凭据）}。

告知用户：配置在 DSH 启动时加载，需重启桌面应用或 `dsh web` 进程生效；可用 `dsh --profile <profile> --dump-config` 只读检查合成结果。**不要声称“已生效”**，只报告“已写入配置”。

## 边界

- 不修改非 `api: openai-completions` 的路由，除非用户明确要求。
- 不写 `reasoningEfforts`、`input`、`compat` 等字段。
- 不把 `maxTokens` 设为超过探测验证或 models.dev 基线的值；未验证时必须在报告里标注。
- 强制更新前一定先备份。