# multi-model-review

`multi-model-review` 是一个给 Codex 使用的多模型审查 Skill。它适合在 Codex 写完代码、改完配置、产出技术方案或分析文档之后，再让多个独立 reviewer 和 judge 复核问题。

它默认是**只读审查**：发现问题、投票确认、输出报告，但不会自动修改代码，除非你在请求里明确要求“审查后修复”。

## 你最常用的方式

Codex 写完代码后，继续在同一个对话里追加一句：

```text
请用 multi-model-review builtin 审查当前改动。
```

更推荐写得明确一点：

```text
请用 multi-model-review builtin 审查当前 git diff，重点检查正确性、测试缺口和边界情况。只做审查，不要修改文件。
```

如果你想审查后让 Codex 修复确认的问题：

```text
请用 multi-model-review builtin 审查当前 git diff。审查完成后，只修复 2/3 或 3/3 确认的问题，并重新运行相关测试。
```

## 审查会不会生成 Markdown 文件

默认不会。默认情况下，Codex 会把审查结果直接回复在对话里。

如果你想生成一个问题报告文件，需要明确说：

```text
请用 multi-model-review builtin 审查当前 git diff，并把审查结果保存到 review-findings.md。
```

或者：

```text
请用 multi-model-review hybrid 审查当前改动，并生成 docs/multi-model-review.md 问题报告。
```

建议报告里包含：

- `3/3 High-confidence findings`
- `2/3 Supported findings`
- 每个问题的位置、证据、影响、建议修复
- 使用了哪些 reviewer 和 judge
- 运行过哪些验证命令

## 会不会自动解决问题

默认不会自动解决。这个 Skill 的默认行为是**审查，不改文件**。

如果你只说：

```text
请用 multi-model-review builtin 审查当前。
```

Codex 应该只报告问题，不应该直接修复。

如果你想让 Codex 自动修复，需要明确说：

```text
请用 multi-model-review builtin 审查当前 git diff。审查完成后，修复所有经过 2/3 或 3/3 确认的问题，并重新运行测试。
```

对于高风险修改、删除文件、数据库、远程服务、发布操作等，Codex 仍然应该先征求确认。

## 工作流程

这个 Skill 分两阶段：

1. **Discovery / 发现问题**
   三个独立 reviewer 分别从不同角度找候选问题。

2. **Judge / 投票确认**
   三个独立 judge 使用同一个 `judge-review` 标准，对每个候选问题投票。

只有 `confirmed` 算支持票。

最终默认只报告：

- `3/3`：高置信问题
- `2/3`：有支持的问题

`1/3` 和 `0/3` 默认不进入最终报告，避免噪声污染结论。

## 使用 3 个独立 GPT/Codex 模型审查

这是最安全、最省配置的方式，不会把代码发送给外部 API。

直接使用 `builtin`：

```text
请用 multi-model-review builtin 审查当前 git diff。
```

更完整的写法：

```text
请用 multi-model-review builtin 审查当前 git diff，范围限定为本次改动和相关测试文件。请使用 3 个独立的 Codex reviewer 做 discovery，再用 3 个独立 judge 投票。只报告 3/3 和 2/3 确认的问题，不要修改文件。
```

`builtin` 默认组合是：

- Reviewer 1：`gpt-5.5` + `correctness-review`
- Reviewer 2：`gpt-5.5` + `testing-review`
- Reviewer 3：`gpt-5.5` + `adversarial-review`
- Judge 阶段：3 个独立 `gpt-5.5` judge，全部使用 `judge-review`

注意：这里的“3 个独立”指独立上下文、独立 prompt、独立 judge 过程；模型可以都是 `gpt-5.5`。

## 使用不同模型审查

如果你想让 GLM、DeepSeek、Claude 或你自己的中转模型参与，需要使用 OpenAI-compatible `chat/completions` 接口。

有两种常见模式：

- `hybrid`：2 个外部模型 + 1 个 Codex 内置模型，推荐日常使用。
- `external`：3 个外部模型，不使用 Codex judge/reviewer。只有当你明确配置了 3 个外部模型时才建议使用。

### 配置 2 个外部模型，用于 hybrid

运行：

```bash
python3 scripts/configure_external_models.py
```

当它询问数量时输入：

```text
2
```

然后依次输入：

```text
name
url
model
api key
```

示例：

```text
number of external models, 2 for hybrid or 3 for external [2]: 2

Model 1
  name [glm]: glm
  url: https://your-proxy.example.com/v1/chat/completions
  model: glm-custom
  api key for MMR_GLM_API_KEY: ********

Model 2
  name [deepseek]: deepseek
  url: https://your-proxy.example.com/v1/chat/completions
  model: deepseek-v4-pro
  api key for MMR_DEEPSEEK_API_KEY: ********
```

脚本会生成：

```text
.runtime/external-reviewers.json
.runtime/external-judges.json
.env.local
```

API key 只写到 `.env.local`；JSON 里只保存环境变量名。

然后在 Codex 里说：

```text
请用 multi-model-review hybrid 审查当前 git diff，使用 .runtime/external-reviewers.json 和 .runtime/external-judges.json 作为外部模型配置。允许把本次 diff 和必要上下文发送给这两个外部模型。只做审查，不要修改文件。
```

hybrid 的默认结构是：

- 外部模型 1：`correctness-review`
- 外部模型 2：`testing-review`
- Codex 内置模型：`adversarial-review`
- Judge 阶段：两个外部 judge + 一个 Codex judge

### 配置 3 个外部模型，用于 external

如果你想使用 3 个不同模型，例如 GLM、DeepSeek、Claude：

```bash
python3 scripts/configure_external_models.py
```

当它询问数量时输入：

```text
3
```

示例：

```text
number of external models, 2 for hybrid or 3 for external [2]: 3

Model 1
  name [glm]: glm
  url: https://your-proxy.example.com/v1/chat/completions
  model: glm-custom
  api key for MMR_GLM_API_KEY: ********

Model 2
  name [deepseek]: deepseek
  url: https://your-proxy.example.com/v1/chat/completions
  model: deepseek-v4-pro
  api key for MMR_DEEPSEEK_API_KEY: ********

Model 3
  name [claude]: claude
  url: https://your-proxy.example.com/v1/chat/completions
  model: claude-custom
  api key for MMR_CLAUDE_API_KEY: ********
```

然后在 Codex 里说：

```text
请用 multi-model-review external 审查当前 git diff，使用 .runtime/external-reviewers.json 和 .runtime/external-judges.json 作为外部模型配置。允许把本次 diff 和必要上下文发送给这三个外部模型。只做审查，不要修改文件。
```

3 个外部模型会分别使用：

- 外部模型 1：`correctness-review`
- 外部模型 2：`testing-review`
- 外部模型 3：`adversarial-review`
- Judge 阶段：三个外部模型全部使用 `judge-review`

## 检查配置是否有效

先准备一个临时 review packet：

```bash
printf 'review packet smoke test\n' > /tmp/mmr-review-packet.txt
```

检查 discovery reviewer 配置：

```bash
python3 scripts/external_review.py \
  --config .runtime/external-reviewers.json \
  --input /tmp/mmr-review-packet.txt \
  --dry-run
```

检查 judge 配置：

```bash
python3 scripts/external_review.py \
  --config .runtime/external-judges.json \
  --input /tmp/mmr-review-packet.txt \
  --dry-run
```

`--dry-run` 不会调用网络，只会检查 JSON、prompt、环境变量是否能被读取。

## 常用请求模板

审查当前改动，不改文件：

```text
请用 multi-model-review builtin 审查当前 git diff，重点检查正确性、测试缺口和边界情况。只报告 3/3 和 2/3 确认的问题，不要修改文件。
```

审查并生成报告：

```text
请用 multi-model-review builtin 审查当前 git diff，并把结果保存到 review-findings.md。只报告 3/3 和 2/3 确认的问题，不要修改代码。
```

使用 2 个外部模型 + Codex 审查：

```text
请用 multi-model-review hybrid 审查当前 git diff，使用 .runtime/external-reviewers.json 和 .runtime/external-judges.json 作为外部模型配置。允许把本次 diff 和必要上下文发送给这两个外部模型。不要修改文件。
```

使用 3 个外部模型审查：

```text
请用 multi-model-review external 审查当前 git diff，使用 .runtime/external-reviewers.json 和 .runtime/external-judges.json 作为外部模型配置。允许把本次 diff 和必要上下文发送给这三个外部模型。不要修改文件。
```

审查后自动修复：

```text
请用 multi-model-review builtin 审查当前 git diff。审查完成后，只修复 2/3 或 3/3 确认的问题，并重新运行相关测试。
```

审查某个目录：

```text
请用 multi-model-review builtin 审查 src/ 和 tests/，重点看刚才实现的功能是否有逻辑错误、边界遗漏和测试缺口。
```

审查一个技术方案：

```text
请用 multi-model-review builtin 审查 docs/design.md，重点检查事实错误、隐藏假设、不可执行的方案和缺失的验证路径。
```

## Prompt 在哪里改

真实会被读取的 prompt 文件是：

```text
references/reviewer-prompts.md
```

主要 prompt：

- `correctness-review`
- `testing-review`
- `adversarial-review`
- `judge-review`
- `product-review`
- `architecture-review`

如果要调整审查风格，直接改对应的 `## Prompt: <prompt-id>` 小节。

## 密钥安全

不要把 API key 写进：

- prompt
- JSON 配置
- 命令行参数
- 日志
- Git 仓库

推荐使用：

```text
.env.local
```

这个文件会被 `.gitignore` 忽略。

## 仓库结构

```text
.
├── SKILL.md
├── README.md
├── agents/
│   └── openai.yaml
├── references/
│   ├── reviewer-prompts.md
│   ├── external-reviewers.md
│   ├── external-reviewers.example.json
│   └── external-judges.example.json
└── scripts/
    ├── configure_external_models.py
    └── external_review.py
```

## 设计原则

- 默认只读审查，不修改用户文件。
- 外部模型调用前必须明确用户授权。
- 只发送有边界的 review packet，不发送整个工作区。
- 票数是置信信号，不是证明。
- 最终报告前必须回到原始材料验证 `3/3` 和 `2/3` 问题。
- 只有用户明确要求修复时，才基于确认问题修改代码。
