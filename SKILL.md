---
name: opencode-provider-setup
description: |-
  为 OpenCode V2 配置第三方供应商（providers）与模型元数据：baseURL、/connect 凭据、模型 limits / capabilities / cost / reasoning variants，并要求改完必须验证。只要用户提到给 OpenCode 加供应商、中转站、网关、自建端点，往 opencode.json 里加模型，补限额/模态/价格/思考档位，或说 /models 列表里缺模型、档位不对，就必须使用本技能——即使用户只说"帮我加个模型"或"variants 不对"。Use for OpenCode custom provider setup, model metadata, and reasoning variant configuration tasks.
---

# OpenCode 供应商与模型配置

> 版本注记：本文基于 OpenCode v2.0.x（2026-10）的实测与官方 V2 文档整理。V2 仍在快速演进，遇到与本文不一致的情况，以本机实测和官方文档为准。

目标：**一次配对，且可验证**。这个技能同时服务两种场景——帮用户接入新的第三方供应商（中转站、网关、自建端点），以及修正已有 supplier 下模型元数据（限额、模态、价格、思考档位）中不对的部分。

## 三条铁律

1. **规格必须来自实证，不许编造。** 每个数字（context / input / output limit、价格、档位值域）都要有来源，优先级：用户提供的服务商文档 > 对端点实测 > models.dev 匹配。三条路都拿不到，就停下来把问题列给用户——编一个数字比缺一个数字更糟：限额大了请求会被上游拒绝或截断，档位写错会让模型报错并自动重试。**绝不"猜一个大概"。**
2. **只写 V2 语法。** 网上大量教程是 V1 的（`provider` 单数、`npm`、`options`），照抄会被静默丢弃或报错。V2 用 `providers`（复数）、`package`、`settings`、`models`。特别提醒：`https://opencode.ai/config.json` 这个在线 schema 至今仍是 V1 形状——如果编辑器按它报错、或某个资料与它一致，都不代表你是对的，以 V2 文档和本机实测为准。
3. **改完必须验证，不是"建议用户验证"。** OpenCode 的配置有静默失败模式：一个字段写错，整个 provider 会从列表里消失，服务端只留一行日志。所以改完必须自己跑第 4 节的验证命令，把结果给用户看。

## 0. 典型流程（照着走，别跳步）

1. **用户报需求**：供应商地址 + 想要的模型（可能附带文档截图/链接）。
2. **你收集事实**：端点探测拿模型 ID（1.1）→ models.dev 匹配元数据（1.2）→ 整理确认表（1.3）。
3. **你写配置**：按第 2 节模板写入。不写 apiKey。
4. **你引导用户填 key**：给出 2.3 的 `/connect` 操作指引，然后**等用户回复"完成"**。
5. **你验证并汇报**：跑第 4 节验证，把结果给用户看。拿不到的数据列出来问用户，不编。

顺序不要颠倒：先写配置、再引导 /connect、用户确认后**才**能跑端到端测试。

## 1. 收集事实

先问清楚目标：

- 是新增供应商，还是给已有供应商补/修模型？
- 用户手上有什么：端点 URL、API key、模型清单、服务商文档截图/链接？
- 用哪个配置文件：全局 `~/.config/opencode/opencode.json(c)`（Windows：`C:\Users\<用户名>\.config\opencode\`）还是项目 `./opencode.json(c)`？
- 本机如果已有能正常工作的供应商配置（用户之前配好的其他中转），先读一份当**活样板**——新条目的字段结构和写法照它抄，比记忆和网上教程都可靠。

### 1.1 端点探测（拿模型 ID）

OpenAI 兼容端点通常暴露模型列表（**只给 ID，不给限额和价格**）：

```sh
curl -sS "https://<地址>/v1/models" -H "Authorization: Bearer <key>"
```

Windows PowerShell 里请用 `curl.exe`（`curl` 在 PowerShell 中默认指向 Invoke-WebRequest，参数不兼容）。拿到的 ID 就是 `models` 里的键名。注意有些中转商返回的是带前缀的 ID（如 `deepseek/deepseek-r1`）——以端点返回的为准。

### 1.2 models.dev 匹配（能匹配就继承）

在 https://models.dev 搜模型名，或者按名字归一化后比对（去掉 `-expires-on-xxxx`、日期后缀、前缀差异）。匹配到就继承：显示名、context/output limit、输入模态、工具支持。**input limit、价格、档位通常拿不到，必须自己填**（见 1.3）。

### 1.3 写配置前，让用户确认这张表

把下面的表填好，随配置一起交给用户过目：能从公开来源（models.dev / 端点探测）拿到的**直接填好并标注来源**，拿不到的写"未知"并明确问用户——用户没有资料就留空，绝不填猜测值。这是防止幻觉规格的最后一道闸：

| 项 | 值 | 来源 |
|---|---|---|
| 模型 ID（端点返回的键名） | | 端点实测 |
| 显示名 | | 用户/文档/models.dev |
| context / input / output 限额 | | 文档/models.dev/用户 |
| 输入模态（text/image/audio/video/pdf） | | models.dev/文档 |
| 工具支持（capabilities.tools） | | models.dev/文档 |
| 价格（输入/输出/缓存读/缓存写） | | 用户提供 |
| 思考档位（有哪些、各映射到什么） | | 用户/文档 |

不确定的格子就写"未知"，并明确问用户"这一项你有资料吗？没有就先留空，不要填猜测值。"

## 2. 写配置

### 2.1 providers 结构（V2 模板）

```jsonc
{
  "providers": {
    "<provider-id>": {
      "name": "供应商显示名",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "https://<地址>/v1"
        // 不要写 apiKey：先让用户执行 /connect 把 key 存进凭据库
      },
      "models": {
        "<model-id>": {
          "name": "模型显示名",
          "limit": { "context": 0, "input": 0, "output": 0 },
          "capabilities": { "tools": true, "input": ["text"], "output": ["text"] },
          "cost": [{ "input": 0, "output": 0, "cache": { "read": 0, "write": 0 } }],
          "variants": []
        }
      }
    }
  }
}
```

### 2.2 字段速查（完整表见 `references/v2-model-fields.md`）

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 显示名 |
| `limit.context` / `limit.input` / `limit.output` | number | `input` 可省略，但填上更完整（UI 里显示为"最大输入"） |
| `capabilities.tools` | boolean | 是否支持工具调用 |
| `capabilities.input[]` / `output[]` | array | 可选值：text / image / audio / video / pdf |
| `cost[]` | array | input / output / cache.read / cache.write；需要按上下文长度分档时可给多元素（带 `tier`） |
| `variants[]` | array | 思考档位，见第 3 节 |
| `settings` / `headers` / `body` | object | 透传给运行时包的任意结构，最后手段才用 |

### 2.3 凭据：引导用户去 /connect（唯一的协同步骤）

配置里**省略 apiKey**。写好后，明确引导用户操作——把下面这份指引直接发给用户（把 `<供应商名>` 换成实际显示名）：

> 请在 OpenCode 里输入 `/connect` → 搜索或找到 **\<供应商名\>** → 回车 → 选择 API key 方式并粘贴 key → 完成后告诉我。

- key 会存进 OpenCode 的凭据库，不写进任何文件（不上云、不进 git）。
- 用户回复"完成"之后，才能跑端到端测试（第 4 节）；用户没填完时端到端测试必然失败，别跑。
- 如果用户在 `/connect` 里找不到该供应商：先检查 provider 配置是否写对（用 4.1 的 `opencode models` 确认注册），配置正确的情况下它就会出现在列表里。
- 用户坚持把 key 写在配置里：可以，但要提醒——这个文件如果进了 git 仓库或云同步，key 就泄露了（中转 key 被盗刷是常见事故）。

## 3. variants（思考档位）专项

这是最容易配错的字段，也是用户最关心的部分。

### 3.1 机制（实测结论，V2.0.x）

- V2 的 variants 是**数组**：`[{ "id": "low", "settings": { "reasoningEffort": "low" } }]`。
- **没写 `variants` 字段的模型会被自动补一套默认档位 low / medium / high**（实测：纯配置的私有模型 ID 和 catalog 模型 ID 都会补）。
- **写了 `variants` 的模型完全按写的来**（实测：写 low/high/max → 就这三个；写单个 max → 只有 max；写空数组 `[]` → 零档位）。
- 一句话：**显式 variants = 完全控制**。想要精确档位，永远把档位写全；不写就会被补三档。
- 唯一例外场景：同一个模型同时来自两个来源（如发现类插件注入的版本 + 配置的显式覆盖）时，两者可能合并（同 ID 替换、不同 ID 并存），仍会出现默认档位（实测见过 `low, medium, high, max`）。用发现类插件且要精确档位时，检查这一点。
- 档位 settings 的常见内容：`reasoningEffort`（OpenAI 兼容系，已实测）。其他字段（如思考预算类设置）的写法和值域因服务商而异——以服务商文档为准，不要凭印象写未实测的字段。

### 3.2 怎么配

1. 先确认该模型真实支持的档位和值域，**基准取官方档位**：在 models.dev 查该模型（或等价模型）的 `reasoning_options`，配置档位集合应与官方列表一致——不多、不少、映射到同名 effort 值。查不到官方数据再问用户/查文档；都不确认就不写。
2. 照用户给出的档位命名原样写（如果用户说"这模型的档位是 low/high/max"，就写这三个 ID）：

```jsonc
"variants": [
  { "id": "low", "settings": { "reasoningEffort": "low" } },
  { "id": "high", "settings": { "reasoningEffort": "high" } },
  { "id": "max", "settings": { "reasoningEffort": "high" } }
]
```

3. 不要写未经 V2 验证的字段（例如 V1 时代出现过 `"disabled": true` 禁档写法，V2 下未验证，不要用——想"禁掉"某档，直接不写它）。

### 3.3 零档位的情况

模型确实不支持思考档位、或不想展示任何档位：写 `"variants": []`（实测 = 零档位，不会被补默认三档）。**不要用"省略 variants 字段"表达零档位**——省略会被补上 low/medium/high。

## 4. 验证（改完按顺序执行，把结果给用户看）

### 4.1 配置能否加载

```sh
# 列出全部已注册的 provider/model（最快的一眼确认）
opencode models
```

配置有语法错误时，opencode 会直接报错（JSONC 缺逗号、括号不配对最常见）。这条命令能跑通、且你的模型出现在列表里，说明配置已被正确解析、provider 已注册。

- 列表里**找不到**你的模型 → provider 被丢弃。常见原因：字段拼写错、JSONC 语法错、V1/V2 语法混用。逐字段核对后重试。

### 4.2 字段核对

```sh
# 输出全部模型定义，搜索目标 provider 的模型
opencode api get /api/model
```

在输出里搜索该模型，检查 `limit` / `capabilities` / `cost` / `variants` 是否与预期一致（以这张输出为准，不要以"文件里写对了"为准）。最后可以发一条真实消息做端到端验证（例：`opencode run -m <provider>/<model> "Reply with exactly: OK"`）。

### 4.3 档位核对

同上输出里看 `variants` 数组：

- 与目标一致 → 通过；
- 多出 low/medium/high → 该模型的定义里 variants 没写（被补默认三档），或同时有另一来源（发现插件）定义了它，见 3.1 与第 3 节；
- 一个都没有（空数组）→ 你写了 `"variants": []`（正常），或 variants 写法有误没被读到。

### 4.4 失败的兜底

仍然异常时：把改动前的备份恢复回去（**铁律：改之前先复制一份配置文件做备份**），确认系统回到可用状态，再把问题和证据整理给用户，不要带着坏配置反复试。

## 5. 真实世界踩过的坑（写配置前速览）

- 一个模型字段写错（哪怕只是一个引号）→ 整个 provider 从 /models 消失，服务端只有一行日志警告，UI 上看不出原因（issue #54973）。
- `variants` 按 V1 的对象写法（`{"low": {}}`）写，而 schema 要求数组 → provider 被静默丢弃（issue #53744）。
- 改了配置后，模型列表刷新一般不需要重启服务；若界面没更新，先试刷新/新开会话，再考虑重启。
- 明文 API key 进 git/云同步 = 泄露；配置目录如果本身是 git 仓库，尤其注意。

## 参考

- `references/v2-model-fields.md` —— 完整字段表、真实配置示例（含档位映射实例）。
