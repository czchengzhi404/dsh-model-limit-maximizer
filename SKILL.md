---
name: model-limit-maximizer
description: 当用户导入或添加了自定义模型后，把 contextWindow 和 maxTokens 校准为该路由下可用的最高值，并按模型真实能力补全输入类型（原生视觉勾选图片）与推理等级。触发词包括“导入模型”“上下文没拉满”“输出上限没设置”“contextWindow 太小”“maxTokens 是空的”“没勾图片”“看图能力没开”“推理等级没配”“思考模式没开”“校准模型能力”。
---

# 模型能力自动校准

## 目标

DSH 的模型条目缺字段时会退到保守默认值或继承目录能力：

| 字段 | 缺失时的行为 |
|---|---|
| `contextWindow` / `maxTokens` | 回退路由默认值（`dsh-llm-pi-ai` 为 262144 / 32768），而不是模型本身的上限 |
| `input` | 默认只有 `text`，原生视觉模型的图片能力被关掉 |
| `reasoningEfforts` | 继承已安装目录的能力，未必等于该模型实际支持的等级 |

本 Skill 在模型导入后做两件事：

1. **上限**：以 models.dev 理论值为基线，与提供方实际限制取较小者，写入前做一次最小探测。
2. **能力**：按 models.dev 的模型元数据补全 `input`（是否勾图片）与 `reasoningEfforts`（开哪些推理等级）。

核心原则：**默认不覆盖已有配置**；只补缺失字段，用户显式要求“强制更新”时才覆盖。

## 配置真相

DSH 的模型配置不在 `settings.yaml`，而在 profile 目录下：

- 生效配置：`~/.dsh/profiles/<profile>/cordis.yml`
- 用户补丁层（**要改这个文件**）：`~/.dsh/profiles/<profile>/cordis.patch.yml`

桌面应用对应 `desktop`，`dsh web` 对应 `web`。模型条目位于：

- `llm-pi-ai` 插件：`config.providers.<provider>.models[]`
- DeepSeek 直连适配器（`dsh-llm`）：`config.models[]`（较少见）

条目是 YAML 对象，本 Skill 涉及的四个字段：

```yaml
- id: deepseek-v4-pro
  name: deepseek-v4-pro
  input:                # 可接受 text / image；原生视觉才有 image
    - text
    - image
  reasoningEfforts:     # off 可为 null；其余等级的值是同名 wire 字符串
    off: null
    minimal: minimal
    low: low
    medium: medium
    high: high
    xhigh: xhigh
    max: max
  contextWindow: 1000000   # 整数 token 数，不是 256K 这种 UI 简写
  maxTokens: 325632
```

同一个模型 id 出现在多个 provider 时，按 provider 路由分别校准——不同路由的可用上限与能力可能不同。

## 工作流

### 0. 读取当前配置

1. 确认当前 profile：默认 `desktop`；用户提到 `dsh web` 时用 `web`。
2. 读 `cordis.patch.yml`，在 `llm-pi-ai.config.providers.<provider>.models` 中定位目标模型。patch 层没有该 provider 时，读 `cordis.yml` 确认继承自 bundle 的值。
3. 记录四个字段现值（缺失 / 空 / 数值 / 已声明）。

完成标准：能说出目标模型在哪个 provider，四个字段各自是“缺失 / 已有值 X”。

### 1. 查询 models.dev

请求 `https://models.dev/api.json`（按 provider 分组的模型表）。按优先级匹配目标 id：

1. **精确匹配**：`data[provider].models[id]`
2. **规范 id**：`data[provider].models["<厂商>/<id>"]`，如 `deepseek/deepseek-v4-pro`
3. **家族推断**：裸 id 匹配同族快照（如 `deepseek-v4-flash` → `deepseek-v4-flash-0731`）

取三组信息：

| 用途 | 字段 | 取值方式 |
|---|---|---|
| 上限基线 | `limit.context` / `limit.output` | 家族取最大；`output` 为 0 表示不产文本 |
| 输入类型 | `modalities.input` | 所有匹配项取**并集**；含 `image`/`video` → 有原生视觉 |
| 推理等级 | `reasoning_options` | 所有匹配项取**并集**；`{"type":"effort","values":[...]}` 是等级，`{"type":"toggle"}` 是开关式 |

匹配是大小写敏感的：`Kimi-k2.6` 这类大写 id 要额外按小写再匹配一次。

查不到 → 标记“无基线”，步骤 2 只依赖提供方数据；提供方也没有 → 跳过上限，并报告原因。

完成标准：拿到上限基线、输入类型结论、推理等级集合三者，或各自的“无基线”标记。

### 2. 读取提供方真实限制

按优先级取第一个可信来源：

**a. 提供方 `/v1/models`**：`GET {baseURL}/models`（baseURL 来自该 provider 的配置），找 `context_length` / `contextWindow` / `max_context_length` 和 `max_output_tokens` / `max_completion_tokens` / `maxTokens`。大多数 OpenAI 兼容端点不返回这些字段——这是正常的，DSH 的 `discoverModels` 也只带回端点披露的值，这正是本 Skill 要补的洞。

**b. 本地服务自报**：Ollama / LM Studio / one-api 等本地路由，优先用服务实际报告的 `num_ctx` / `max_tokens`。

**c. 官方文档 / 用户已有配置**：同 provider 其他条目填过的值（同一路由已填的上限往往就是该路由接受的截断值）。

完成标准：每个字段都标出“有提供方值”或“无提供方值”。

### 3. 计算上限

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

### 4. 计算能力字段

**输入类型**：任一匹配条目的 `modalities.input` 含 `image` 或 `video` → 写 `input: [text, image]`；否则不写（默认就是纯文本，所有模型都能收文本）。

**推理等级**：把并集里的 effort 值映射成等级，写进 `reasoningEfforts`：

| models.dev 值 | 等级键 | 值 |
|---|---|---|
| `none` | `off` | `null` |
| `minimal` / `low` / `medium` / `high` / `xhigh` / `max` | 同名 | 同名字符串 |

- 只报了 `toggle`（没有 effort 值）的推理模型 → 用标准集 `off: null, low: low, high: high, max: max`。
- 完全没有推理能力的模型 → **不写** `reasoningEfforts`（省略即继承目录能力；不要写 `false`，除非确定该模型不支持推理）。
- 未声明的等级在 DSH 里被固定为“不支持”，所以“能开都开”就是把这一个集合声明全。

完成标准：每个模型都得到 `input` 决定（加/不加 image）与 `reasoningEfforts` 决定（等级集合 / 不写）。

### 5. 写入前探测

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
- 降到 8192 仍失败 → 不写入上限，报告“该路由拒绝了所有尝试的输出上限，需人工检查”。
- 返回 401/403（凭据不可用）或网络不可达 → **不阻塞写入**：这是配置层补漏，不是运行时验证。写入基线值，报告中标注“未验证”。

探测只验证上限，不验证 `input` / `reasoningEfforts`；后两者始终按“未验证”报告。

完成标准：探测通过，或明确记录降档后的值，或标注“未验证”及原因。

### 6. 写入配置

1. **先备份**：把 `cordis.patch.yml` 复制为 `cordis.patch.yml.bak-<时间戳>`。
2. 只修改目标条目的四个字段，不动 `name`、`api`、`baseURL`、`compat` 等其他字段。
3. 写入规则（逐字段判断）：
   - 字段缺失或为空 → 写入计算值。
   - 字段已有且与计算值不同，用户未要求强制更新 → **保留原值**，报告“当前 X，可用 Y，如需更新请说强制更新”。
   - 字段已有且用户要求强制更新 → 覆盖。
   - 字段已有且已是可用最高 → 保留。
4. 保持 YAML 缩进与现有结构一致（条目列表缩进 10 空格、字段 12 空格）；patch 文件里可能混有 `!!js` 表达式，不要改动它们。

完成标准：目标条目只有这四个字段发生变化，其余内容不动；备份文件已生成。

### 7. 报告与验证

给出一张对照表：

| 模型 | provider | contextWindow 旧→新 | maxTokens 旧→新 | 输入类型 | 推理等级 | 状态 |
|---|---|---|---|---|---|---|

状态 ∈ {已填充，已跳过（已有配置），已强制更新，无基线跳过，探测失败未写入，未验证（无凭据）}。

告知用户：配置在 DSH 启动时加载，需重启桌面应用或 `dsh web` 进程生效；可用 `dsh --profile <profile> --dump-config` 只读检查合成结果。**不要声称“已生效”**，只报告“已写入配置”。

## 边界

- 不修改非 `api: openai-completions` 的路由，除非用户明确要求。
- 不写 `compat` 等与本次目标无关的字段。
- 图像/视频生成模型（如 `gpt-image-*`、`agnes-image-*`）只补 `input`，不补 `reasoningEfforts`。
- 不把 `maxTokens` 设为超过探测验证或 models.dev 基线的值；未验证时必须在报告里标注。
- 强制更新前一定先备份。
