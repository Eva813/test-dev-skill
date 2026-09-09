# Agent Skills for Your Org
AI Agent 技能集合，包含 10 個核心開發流程技能。

## 可用技能

| Skill | 描述 |
|-------|------|
| `axure-spec-scraper` |  figma 擷取 |
| ... | ... |

## 安裝方式

```bash
# 使用 gh CLI（推薦）
gh skill install your-org/agent-skills --skill playwright-testing --pin v1.0.0

# 或使用 npx skills
npx skills add your-org/agent-skills --skill playwright-testing
```

## 版本策略

所有技能使用 Git tag 進行版本控制，生產環境請使用 `--pin <tag>` 固定版本。

## 授權

本專案以 [Apache License 2.0](LICENSE) 釋出。

`skill-creator` 源自 Anthropic 的 skill-creator，同樣以 Apache-2.0 授權，原始授權條款保留於
[skills/skill-creator/LICENSE.txt](skills/skill-creator/LICENSE.txt)。
