# opencode-provider-setup

**给 OpenCode 加第三方中转站时，你只需要报出「地址 + 模型名」，剩下的交给它。**

不写 JSON、不查文档、不记格式——你说话，AI 补全参数、写入配置、验证结果。

## 它帮你做什么

你说一句话：

> 帮我接入 xx 中转站，地址 `https://api.example.com/v1`，用 `deepseek-v4.1-flash` 和 `glm-5.3-flash`

它自动完成：

- 探测你的端点，拿到**准确的模型 ID**（不猜）
- 从 models.dev 匹配每个模型的：限额、输入模态、工具支持、**官方思考档位**
- 按 OpenCode V2 的正确格式写入配置
- 改完跑三级验证，把结果给你看

## 配置过程中，你只做一件事：填一次 API key

第一次配置某个供应商时，AI 会引导你（约 30 秒）：

1. 在 OpenCode 里输入 `/connect`
2. 搜索、找到你的供应商名
3. 回车，粘贴 API key

key 存进 OpenCode 的凭据库，**不写进任何配置文件**（不上云、不进 git）。以后不用再碰。

## 一次完整对话长这样

| 你 / AI | 内容 |
|---|---|
| 你 | 帮我加个中转站 `https://api.example.com/v1`，里面有 `deepseek-v4.1-flash` 和 `glm-5.3-flash`，思考档位保留 low/high/max |
| AI | （探测端点 → 匹配元数据 → 给你一张参数确认表） |
| AI | 配置写好了。请在 OpenCode 里输入 `/connect`，找到 **example** 供应商，回车并粘贴 API key，完成后告诉我 |
| 你 | 好了 |
| AI | ✅ 两个模型已注册；参数核对（限额/模态/档位）无误；发了一条测试消息，正常返回 |

公开渠道查不到的数据（比如你中转站的实际价格），AI 会**明确问你**，而不是编一个填进去。

## 为什么可以放心

- **不编数据**：每个数字都有来源（你的文档 > 端点实测 > models.dev），拿不到就问你
- **只写正确语法**：网上教程大多还是旧的 V1 写法，照抄会让 provider 从列表里**静默消失**；本技能内置 V2 正确写法和真实踩坑清单
- **改完必验证**：改前自动备份、改后跑命令核对、失败回滚
- **经过对照测试**：同一模型三个任务，带技能 vs 不带技能：

| 任务 | 带技能 | 不带技能 |
|---|---|---|
| 坏配置诊断 | 5/5 | 0/5（V1/V2 语法整个搞反） |
| 新供应商（信息不全） | 5/5 | 4/5 |
| 档位核对 | 5/5 | 4/5 |

## 什么时候用得上

- 接入新的第三方中转站 / 网关 / 自建 OpenAI 兼容端点
- 某供应商的模型突然从列表里"消失"
- 模型的思考档位（variants）多了、少了、不对
- 补全模型的限额 / 价格 / 模态信息

## 安装

```sh
npx skills add tttrove/opencode-provider-setup
```

手动安装：把 `SKILL.md` 和 `references/` 复制到技能目录，如 `~/.agents/skills/opencode-provider-setup/`。

## 适用范围

基于 OpenCode v2.0.x（2026-10）的实测与官方 V2 文档整理；V2 迭代较快，遇到差异以本机实测为准。主要面向使用第三方供应商的用户。

## License

MIT

---

## English

**Adding a third-party relay to OpenCode? Just name the URL and the models — the skill fills in everything else.**

The one thing you do yourself: the first time you set up a provider, the agent walks you through `/connect` (search your provider's name → enter → paste the API key). The key lives in OpenCode's credential store — never in a config file. Typically a one-time, 30-second step.

What the skill handles: probing your endpoint for exact model IDs, matching each model against models.dev (limits, modalities, tool support, official reasoning levels), writing a correct OpenCode V2 config, and verifying the result afterwards.

Why it's safe to hand over:

- **No invented data** — every number has a source (your docs > endpoint probe > models.dev); anything unavailable gets asked, not guessed
- **V2 syntax only** — most online tutorials are V1-shaped and silently break your provider
- **Always verified** — backup, check, rollback on failure
- **Tested** — 15/15 vs 8/15 assertions for with-skill vs baseline across three tasks

Install:

```sh
npx skills add tttrove/opencode-provider-setup
```

Based on OpenCode v2.0.x (verified 2026-10). MIT.
