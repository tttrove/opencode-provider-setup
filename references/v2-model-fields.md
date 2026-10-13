# OpenCode V2 供应商 / 模型字段参考

来源：OpenCode V2 OpenAPI schema（`/openapi.json`）+ 实测（2026-10，v2.0.x）。V2 演进较快，与本文不一致时以本机实测为准。

## Provider 完整字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 显示名 |
| `package` | string | 运行时包。OpenAI 兼容端点固定用 `@opencode/ai/providers/openai-compatible`；此外还有 `openai`、`openai/responses`、`anthropic`、`google`、`openrouter` 等官方包，也支持任意 npm 包 |
| `settings` | object | 类型化控制 + 传给运行时包的 JSON 选项。`baseURL` 就写在这里 |
| `settings.baseURL` | string | 端点根地址，一般以 `/v1` 结尾 |
| `settings.timeout` / `headerTimeout` / `chunkTimeout` | number \| false | 毫秒，可选 |
| `headers` / `body` | object | 附加请求头/体，可选 |
| `canonical` | string | 声明等同某个内建 provider（继承其 catalog 默认值），可选 |
| `env` | string[] | 凭据环境变量名，可选 |
| `models` | object | 按模型 ID 键控的模型定义 |

## 模型完整字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 显示名 |
| `family` | string | 家族（可选，影响 UI 分组） |
| `limit.context` | number | 上下文上限（token） |
| `limit.input` | number | 最大输入（可选，但强烈建议填——不填 UI 显示 unknown） |
| `limit.output` | number | 最大输出（token） |
| `capabilities.tools` | boolean | 工具调用 |
| `capabilities.input` | string[] | text / image / audio / video / pdf |
| `capabilities.output` | string[] | 通常 `["text"]` |
| `cost` | array | 每元素：`{ "tier"?: { "type": "context", "size": number }, "input": n, "output": n, "cache": { "read": n, "write": n } }`，单位是每百万 token 的美元价 |
| `variants` | array | 每元素：`{ "id": string, "settings"?: object, "headers"?: object, "body"?: object }` |
| `settings` | object | 模型级透传选项，可选 |
| `headers` / `body` | object | 模型级请求覆盖，可选 |
| `compatibility` | object | 特殊兼容开关（`reasoningField`、`maxTokensField`、`requireReasoning` 等），一般用不到 |
| `disabled` | boolean | `true` 时模型仍存在但从选择列表隐藏 |
| `status` | string | alpha / beta / deprecated / active |
| `time.released` | number | 发布日（Unix 毫秒），可选 |

## V1 → V2 常见误写对照

| V1 写法（旧教程） | V2 写法 | 备注 |
|---|---|---|
| `"provider": {...}`（单数） | `"providers": {...}`（复数） | 写错会静默丢弃 |
| `"npm": "@ai-sdk/openai-compatible"` | `"package": "@opencode/ai/providers/openai-compatible"` | |
| `"options": { "baseURL": ... }` | `"settings": { "baseURL": ... }` | |
| `"options": { "apiKey": ... }` | 推荐不写，用 `/connect` | 若必须写：`"settings": { "apiKey": ... }`，有泄露风险 |
| `"variants": { "low": {}, "high": {} }`（对象） | `"variants": [ { "id": "low" }, { "id": "high" } ]`（数组） | 对象写法会让整个 provider 被丢弃（issue #53744） |
| `"whitelist": [...]` | 不适用 | V2 直接用 `models` 显式列出 |
| `"reasoning": true`（模型级） | 显式 `"variants": [...]` | V1 的 reasoning 标记会被兼容层翻译成 `low, medium, high` 三档；V2 里想要精确档位就直接写 variants |

## 真实配置示例（脱敏）

### 例 1：中转站 + 私有模型 + 三档思考

```jsonc
{
  "providers": {
    "my-relay": {
      "name": "My Relay",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": { "baseURL": "https://token.example.cn/v1", "setCacheKey": true },
      "models": {
        "deepseek/deepseek-v4.1-flash": {
          "name": "DeepSeek V4.1 Flash",
          "family": "deepseek-flash",
          "limit": { "context": 1000000, "input": 922000, "output": 384000 },
          "capabilities": {
            "tools": true,
            "input": ["text", "image"],
            "output": ["text"]
          },
          "cost": [{ "input": 0.15, "output": 0.6, "cache": { "read": 0.003 } }],
          "variants": [
            { "id": "low", "settings": { "reasoningEffort": "low" } },
            { "id": "high", "settings": { "reasoningEffort": "high" } },
            { "id": "max", "settings": { "reasoningEffort": "high" } }
          ]
        }
      }
    }
  }
}
```

### 例 2：档位映射到思考预算（示意）

`settings` 里可以放服务商特有的档位参数，把档位映射成不同的思考强度。下例用思考预算示意——**此具体结构未在 V2 实测，字段名与取值请以服务商文档为准**：

```jsonc
"variants": [
  { "id": "none", "settings": { "thinking": { "type": "disabled" } } },
  { "id": "low", "settings": { "thinking": { "type": "enabled", "budgetTokens": 8000 } } },
  { "id": "medium", "settings": { "thinking": { "type": "enabled", "budgetTokens": 24000 } } },
  { "id": "high", "settings": { "thinking": { "type": "enabled", "budgetTokens": 48000 } } },
  { "id": "max", "settings": { "thinking": { "type": "enabled", "budgetTokens": 64000 } } }
]
```

### 例 3：多价格档位（按上下文长度计价）

```jsonc
"cost": [
  { "input": 1.0, "output": 4.0 },
  { "tier": { "type": "context", "size": 200000 }, "input": 2.0, "output": 8.0 }
]
```

## 模型引用语法

- 选择模型：`providerID/modelID`（如 `microvideo/deepseek/deepseek-v4.1-flash`）
- 选择档位：`providerID/modelID#variantID`（如 `microvideo/z-ai/glm-5.3-flash#high`）
- 因此 provider ID 建议用简短小写、避免 `/` 与 `#`（它们是引用语法的分隔符）；模型键可以包含 `/`（实测可用）。

## 验证命令速查

```sh
# 1. 列出已注册的 provider/model（配置加载成功即可看到）
opencode models

# 2. 全部模型定义（找目标 provider 的模型，核对字段）
opencode api get /api/model

# 3. 单看某个供应商（配置层，不含模型）
opencode api get /api/provider/<providerID>
```

`/api/model` 的输出结构是 `{ "location": ..., "data": [ ...Model.Info... ] }`，每个元素的字段与上面的模型字段表一致。

## 实测过的档位行为（2026-10，v2.0.x 隔离复测）

| 场景 | 结果 |
|---|---|
| 不写 `variants` 字段 | 自动补 `low, medium, high` 三档 |
| 写 `"variants": []`（空数组） | 零档位（不会被补默认三档） |
| 写单个档位（如只有 `max`） | 只有 `max` |
| 写 `low/high/max` | 就是 `low, high, max`，无额外档位 |
| 发现插件注入的模型 + 配置显式覆盖（两个来源） | 曾实测到并集 `low, medium, high, max`（同 ID 替换、不同 ID 并存） |
| V1 写法 `"reasoning": true`（无 variants，经兼容层） | 翻译为 `low, medium, high` 三档 |
