# Oh My OpenCode 使用指南

> 多智能体协作的 OpenCode 增强插件，让 AI 开发团队为你工作。

## 简介

**Oh My OpenCode** 是一个超级增强版的 OpenCode 插件，让你输入 `ultrawork` 就能让多个 AI 智能体协同完成复杂任务。

### 核心特性

| 特性 | 说明 |
|------|------|
| **`ultrawork`** | 一个命令激活所有智能体，不完成不停止 |
| **多智能体协作** | Sisyphus、Hephaestus、Prometheus 等协同工作 |
| **多模型编排** | 自动根据任务类型选择最优模型 |
| **Hash-Anchored Edits** | 用内容哈希验证每次编辑，零"旧行错误" |
| **LSP + AST-Grep** | IDE 级别的重构、重命名、诊断能力 |
| **内置 MCP** | Exa（网页搜索）、Context7（官方文档）、Grep.app（GitHub搜索） |

## 智能体分工

| 智能体 | 职责 | 模型 |
|--------|------|------|
| **Sisyphus** | 主调度器，规划、委派、并行执行 | GLM-5 |
| **Hephaestus** | 深度工作者，自主探索执行 | GPT-5.3 Codex |
| **Prometheus** | 战略规划师，面试模式收集需求 | GLM-5 |
| **Oracle** | 架构、调试专家 | GLM-5 |
| **Librarian** | 文档、代码搜索 | GLM-5 |
| **Explore** | 快速代码库探索 | GLM-5 |

## 安装

```bash
# 安装 OpenCode
curl -fsSL https://opencode.ai/install | bash

# 安装 oh-my-opencode
bunx oh-my-opencode install --no-tui \
  --claude=no \
  --gemini=no \
  --copilot=no \
  --openai=no \
  --zai-coding-plan=yes
```

## 使用方式

### 方式 1：Ultrawork（快速自主工作）

```bash
cd /path/to/project
opencode run "ulw 实现用户认证功能"
```

智能体会自动：
1. 探索代码库理解现有模式
2. 通过专家智能体研究最佳实践
3. 遵循项目约定实现功能
4. 运行诊断和测试验证
5. 持续工作直到 100% 完成

### 方式 2：Prometheus 规划模式（复杂任务）

```bash
opencode run "/start-work 给应用添加消息通知系统"
```

Prometheus 会像工程师一样面试你：
- 收集需求细节
- 识别范围和模糊点
- 制定详细计划
- 然后再开始实现

### 方式 3：交互模式

```bash
cd /path/to/project
opencode
# 然后在 TUI 中输入任务
```

## 配置

### 配置文件位置

- 全局配置: `~/.config/opencode/opencode.json`
- 插件配置: `~/.config/opencode/oh-my-opencode.json`

### 当前配置 (ali/glm-5)

```json
{
  "agents": {
    "sisyphus": { "model": "ali/glm-5" },
    "oracle": { "model": "ali/glm-5" },
    "librarian": { "model": "ali/glm-5" },
    "explore": { "model": "ali/glm-5" },
    "prometheus": { "model": "ali/glm-5" }
  },
  "categories": {
    "visual-engineering": { "model": "ali/glm-5" },
    "ultrabrain": { "model": "ali/glm-5" },
    "quick": { "model": "ali/glm-5" }
  }
}
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `ulw` 或 `ultrawork` | 激活多智能体协作 |
| `/start-work` | Prometheus 规划模式 |
| `/init-deep` | 生成项目 AGENTS.md 文件 |
| `/undo` | 撤销上次更改 |
| `/redo` | 重做更改 |

## 最佳实践

### 任务描述技巧

1. **像对初级工程师一样描述任务**
   - 提供足够的背景信息
   - 指明相关文件和模块
   - 说明期望的结果

2. **复杂任务先规划**
   ```
   /start-work 实现...（描述需求）
   ```

3. **引用现有代码模式**
   ```
   参考src/auth/中的认证逻辑，在src/api/中实现类似的中
   ```

### 推荐工作流

```
1. /init-deep          → 生成项目上下文
2. /start-work 需求    → 制定计划
3. 确认计划后执行       → ulw 实现
4. 验收和调整          → 迭代完善
```

## 注意事项

- **Sisyphus 最适合 Claude Opus 4.5**，使用其他模型可能性能下降
- 使用 `ulw` 时智能体会持续工作直到完成
- 复杂任务建议先用 `/start-work` 规划

## 相关链接

- GitHub: https://github.com/code-yeongyu/oh-my-openagent
- ClawHub: https://clawhub.ai/McOso/oh-my-opencode
- OpenCode 文档: https://opencode.ai/docs