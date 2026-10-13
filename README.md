# opencode-provider-setup

> 为 OpenCode V2 配置第三方供应商与模型元数据的 agent skill。
> An agent skill for correctly setting up third-party providers and model metadata in OpenCode V2.

## 为什么需要它

给 OpenCode 接第三方供应商（中转站、网关、自建 OpenAI 兼容端点）时，配置看起来简单，实际处处是坑：

- 网上大量教程是 V1 语法（`provider` 单数 / `npm` / `options`），照抄会被**静默丢弃**——provider 从模型列表消失，UI 上没有任何报错
- OpenCode 官方在线 schema（`opencode.ai/config.json`）至今仍是 V1 形状，编辑器和工具会被它带偏
- 思考档位（variants）有一套自动机制：不写档位会被自动补默认三档，想要精确档位必须写全；用发现类插件时两个来源还可能合并
- 一个字段写错（哪怕只是一个引号），整个 provider 会从模型列表消失，只有一个日志行提示

这个技能把这些知识固定成**可重复、可验证的流程**，让配置任务不依赖执行者的记忆和经验——包括用能力较弱的模型来执行时。

## 安装

### 方式一：skills CLI（推荐）

```sh
npx skills add tttrove/opencode-provider-setup
```

### 方式二：手动

把本目录复制到 OpenCode 的技能目录，例如：

- 全局：`~/.config/opencode/skills/opencode-provider-setup/`
- 或 agent 通用布局：`~/.agents/skills/opencode-provider-setup/`

（以你的 OpenCode 版本实际识别的技能目录为准。）

## 覆盖范围

| 场景 | 内容 |
|---|---|
| 接入新供应商 | 端点探测、V2 providers 结构、`/connect` 凭据、避免明文 key |
| 模型元数据 | limits（含常被漏掉的 `input` 限额）、capabilities（模态/工具支持）、cost（含按上下文分档的价格） |
| 思考档位 variants | 实测机制（显式定义 = 完全控制）、与 models.dev 官方档位对齐、零档位写法 |
| 验证 | `opencode models` / `opencode api get /api/model` / 端到端发消息，三级验证与失败回滚 |

## 三条铁律

1. **规格必须来自实证，不许编造**——每个数字都要有来源（服务商文档 > 端点实测 > models.dev 匹配），拿不到就停下来问用户
2. **只写 V2 语法**——并警惕被 V1 形状的在线 schema 带偏
3. **改完必须验证**——配置有静默失败模式，验证结果要给用户看

## 包含文件

```
SKILL.md                          # 主流程：收集事实 → 写配置 → 档位专项 → 验证 → 坑清单
references/v2-model-fields.md     # 完整字段表、V1→V2 对照、真实示例、实测档位行为
```

## 适用范围

- 基于 OpenCode v2.0.x（2026-10）实测与官方 V2 文档整理；V2 仍在演进，遇到差异以本机实测为准
- 少数未实测内容（如思考预算映射示例）在文中明确标注为"示意"
- 面向"给 OpenCode 接第三方供应商"的用户；只使用官方渠道的用户可能用不上

## 发布前的对照测试

同一模型、三个代表性任务（坏配置诊断 / 信息不全的新供应商配置 / 档位核对），各跑一次"带技能 vs 不带技能"：

| 任务 | 带技能 | 不带技能 |
|---|---|---|
| 坏配置诊断 | 5/5 | 0/5（把 V1/V2 语法整个搞反） |
| 新供应商（信息不全） | 5/5 | 4/5 |
| 档位核对 | 5/5 | 4/5 |

价值最大的场景：**没有现场可查、纯凭记忆判断**的时候——这正是弱模型最容易翻车的地方。

## 致谢

- [OpenCode](https://opencode.ai) 官方 V2 文档
- [models.dev](https://models.dev)（模型元数据来源）
- OpenCode 社区 issue 中的真实故障报告（#54973、#53744 等）

## License

MIT

---

## English

**opencode-provider-setup** is an [agent skill](https://skills.sh) for setting up third-party providers and model metadata in OpenCode V2 (OpenAI-compatible relays, gateways, and self-hosted endpoints).

It enforces a verifiable workflow: gather facts (endpoint probing, models.dev matching, user confirmation) → write V2-syntax config → configure reasoning variants against the official effort list → verify with `opencode models` / `opencode api get /api/model` / an end-to-end message.

Key knowledge baked in:

- V1 vs V2 syntax differences (most online tutorials are V1 and get silently dropped; the online schema is still V1-shaped)
- Reasoning variant behavior: explicit `variants` gives full control; omitting the field auto-adds low/medium/high
- Cost tiers, capabilities, and limits (including the often-missing `input` limit)
- Safe credential handling via `/connect` instead of plaintext keys

Install:

```sh
npx skills add tttrove/opencode-provider-setup
```

Based on OpenCode v2.0.x (verified 2026-10). Validated with a with-skill vs baseline comparison: 15/15 vs 8/15 assertions across three tasks. Licensed under MIT.
