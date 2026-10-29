# dsh-model-limit-maximizer

> 一个 DeepSeek Harness（DSH）Skill：模型导入后，把 `contextWindow` 与 `maxTokens` 校准到该路由下可用的最高值，并可选补全输入类型与推理等级。
>
> A DeepSeek Harness (DSH) skill that calibrates `contextWindow` / `maxTokens` to the highest usable values for a model's route after import.

## 解决什么问题

DSH 的模型条目如果没有显式 `contextWindow` / `maxTokens`，会回退到路由的保守默认值（`dsh-llm-pi-ai` 默认 `262144` / `32768`），而不是模型本身的上限。导入自定义模型（尤其第三方中转、OpenAI 兼容端点）时，端点通常不会在 `/v1/models` 里披露这两个字段，于是模型看起来能用，上下文和输出却被砍了一大截。

本 Skill 把这一步补上。

## 它做什么

| 步骤 | 动作 |
|---|---|
| 0 | 读取 `~/.dsh/profiles/<profile>/cordis.patch.yml`，定位目标模型 |
| 1 | 查询 [models.dev](https://models.dev) 基线（`limit.context` / `limit.output`） |
| 2 | 读取提供方真实限制（`/v1/models`、本地服务自报、官方文档、同路由已有配置） |
| 3 | 取两者较小值；`maxTokens` 施加 0.85 安全边际并对齐 1024 |
| 4 | 写入前探测：一次 `max_tokens=16` 的最小请求，失败按档降级 |
| 5 | 写入配置：先备份，只改目标字段，默认不覆盖已有配置 |
| 6 | 输出「旧值 → 新值」对照表 |

## 安装

全局（对所有 DSH profile 生效）：

```bash
mkdir -p ~/.dsh/skills/model-limit-maximizer
cp SKILL.md ~/.dsh/skills/model-limit-maximizer/SKILL.md
```

项目级：放到项目的 `.dsh/skills/model-limit-maximizer/SKILL.md`。

安装后重启 DSH（或重开 profile）即可被识别。

## 使用

对 Agent 说任意一句即可触发：

- 「我刚导入了 `xxx` 模型，帮我把上下文和输出上限拉到可用最高」
- 「校准 `xxx` 的 contextWindow 和 maxTokens，强制更新」
- 「这个模型 contextWindow 太小 / maxTokens 是空的」

## 配置真相：DSH 的模型配置在哪

| 文件 | 作用 |
|---|---|
| `~/.dsh/profiles/<profile>/cordis.yml` | 生效配置（bundle 合成结果） |
| `~/.dsh/profiles/<profile>/cordis.patch.yml` | **用户补丁层 —— 要改的就是它** |

模型条目位置：

- `llm-pi-ai` 插件：`config.providers.<provider>.models[]`
- DeepSeek 直连适配器：`config.models[]`

条目示例（**配置里是整数 token 数，不是 `256K` 这种 UI 简写**）：

```yaml
- id: deepseek-v4-pro
  name: deepseek-v4-pro
  input:                # 只有原生视觉模型才加 image
    - text
    - image
  reasoningEfforts:     # off 可为 null；其余等级的值是同名 wire 字符串
    off: null
    low: low
    high: high
    max: max
  contextWindow: 1000000
  maxTokens: 325632
```

> 补充：`input` 只接受 `text` / `image`；`reasoningEfforts` 的等级顺序为
> `off, minimal, low, medium, high, xhigh, max`，未声明的等级会被固定为「不支持」。

## 安全约定

- **默认不覆盖已有配置**；只在字段缺失/为空，或用户显式要求「强制更新」时写入。
- 写入前备份 `cordis.patch.yml`。
- 只改目标字段，不动 `name` / `api` / `baseURL` / 推理等级 / 其他模型。
- 拿不到凭据时不做运行时验证：写入 models.dev 基线值，并在报告中标注「未验证」。
- 安全边际只加在 `maxTokens`：上下文超限报错明确、容易回退；输出上限过高可能在生成中途失败，更难排查。

## 兼容性

针对 DSH `0.2.1-rc.3`（`@deepseek-ai/dsh-llm-pi-ai`）的配置结构验证；字段名与默认值以该版本源码为准。

## License

MIT
