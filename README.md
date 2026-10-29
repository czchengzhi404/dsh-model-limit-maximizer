# dsh-model-limit-maximizer

> 一个 DeepSeek Harness（DSH）Skill：模型导入后，自动把 `contextWindow` / `maxTokens` 校准到该路由下可用的最高值，并补全 `input`（原生视觉勾选图片）与 `reasoningEfforts`（推理等级）。
>
> A DeepSeek Harness (DSH) skill that calibrates `contextWindow` / `maxTokens` to the highest usable values for a model's route, and fills in `input` (native vision → image) and `reasoningEfforts` (thinking levels).

## 解决什么问题

DSH 的模型条目缺字段时会退到保守默认值，或者继承已安装目录的能力：

| 字段 | 缺失时的行为 |
|---|---|
| `contextWindow` / `maxTokens` | 回退路由默认值（`dsh-llm-pi-ai` 为 `262144` / `32768`），而不是模型本身的上限 |
| `input` | 默认只有 `text`，原生视觉模型的图片能力被关掉 |
| `reasoningEfforts` | 继承目录能力，未必等于该模型实际支持的等级 |

导入自定义模型（尤其第三方中转、OpenAI 兼容端点）时，端点通常不会在 `/v1/models` 里披露这些信息，于是模型"看起来能用，能力和上限却被砍了一大截"。

本 Skill 把这一步补上：**上限拉到可用最高，能力按模型真实情况补全。**

## 它做什么

| 步骤 | 动作 |
|---|---|
| 0 | 读取 `~/.dsh/profiles/<profile>/cordis.patch.yml`，定位目标模型 |
| 1 | 查询 [models.dev](https://models.dev) 基线：`limit.*` + `modalities.input` + `reasoning_options` |
| 2 | 读取提供方真实限制（`/v1/models`、本地服务自报、官方文档、同路由已有配置） |
| 3 | 取两者较小值；`maxTokens` 施加 0.85 安全边际并对齐 1024 |
| 4 | 计算能力字段：有原生视觉 → `input: [text, image]`；按并集声明推理等级 |
| 5 | 写入前探测：一次 `max_tokens=16` 的最小请求，失败按档降级 |
| 6 | 写入配置：先备份，只改目标字段，默认不覆盖已有配置 |
| 7 | 输出「旧值 → 新值」对照表（含输入类型与推理等级） |

## 安装

全局（对所有 DSH profile 生效）：

```bash
mkdir -p ~/.dsh/skills/model-limit-maximizer
cp SKILL.md ~/.dsh/skills/model-limit-maximizer/SKILL.md
```

项目级：放到项目的 `.dsh/skills/model-limit-maximizer/SKILL.md`。

Skill 是**热加载**的：写完文件，DSH 的 skill 目录下一次刷新即可看到，无需重启 DSH。

## 使用

对 Agent 说任意一句即可触发：

- 「我刚导入了 `xxx` 模型，帮我把上下文和输出上限拉到可用最高」
- 「这些模型有原生视觉的帮我把图片勾上，推理等级能开的都开」
- 「校准 `xxx` 的 contextWindow / maxTokens / input / reasoningEfforts，强制更新」

## 能被别的 agent 调用吗

可以，前提是对方 harness 也会扫描同一个 skill 目录：

- **同一台机器的其他 DSH agent**（子 agent、team 成员、其他 profile）：只要它们加载了 `~/.dsh/skills`（本机默认会扫描这里，也会扫 `~/.agents/skills`），就能看到并调用本 Skill。
- **其他机器 / 其他人**：把 `SKILL.md` 放到他们的 `~/.dsh/skills/model-limit-maximizer/` 即可。
- **其他 harness**：`SKILL.md` 是通用格式，凡支持该格式（如 `.agents/skills`）的工具都能读取；但内容里的路径与字段是 DSH 专用的。

一个前提要清楚：**Skill 是指令，不是自动钩子。** 它会在 agent 判断触发条件成立时被调用（或用户点名调用），不会像插件那样在"导入模型"那一刻自动执行。要保证导入即生效，得用插件钩子，或在导入后明确说一句触发。

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

字段约束（来自 `@deepseek-ai/dsh-llm-pi-ai` 源码）：

- `input` 只接受 `text` / `image`。
- 推理等级顺序：`off, minimal, low, medium, high, xhigh, max`。
- `reasoningEfforts` 是 dict；只有 `off` 可以为 `null`，其余等级必须给 wire 字符串。**未声明的等级会被固定为「不支持」**，所以"能开都开"= 把支持的等级全部声明出来。
- 非推理模型不要写 `reasoningEfforts`（省略即继承目录能力）。

## 安全约定

- **默认不覆盖已有配置**；只在字段缺失/为空，或用户显式要求「强制更新」时写入。
- 写入前备份 `cordis.patch.yml`。
- 只改目标字段，不动 `name` / `api` / `baseURL` / `compat`。
- 拿不到凭据时不阻塞写入：写入 models.dev 基线值，并在报告中标注「未验证」。
- 安全边际只加在 `maxTokens`：上下文超限报错明确、容易回退；输出上限过高可能在生成中途失败，更难排查。

## 兼容性

针对 DSH `0.2.1-rc.3`（`@deepseek-ai/dsh-llm-pi-ai`）的配置结构验证；字段名、模态枚举与推理等级枚举以该版本源码为准。

## License

MIT
