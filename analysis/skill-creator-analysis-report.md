# Anthropic Skill Creator 完整分析报告

> 本报告用于指导实现自定义的 Skill Creator 技能

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心架构设计](#2-核心架构设计)
3. [目录结构与文件职责](#3-目录结构与文件职责)
4. [数据流与工作流](#4-数据流与工作流)
5. [核心模块详细分析](#5-核心模块详细分析)
6. [JSON Schema 完整定义](#6-json-schema-完整定义)
7. [代理系统设计](#7-代理系统设计)
8. [脚本系统设计](#8-脚本系统设计)
9. [触发优化系统](#9-触发优化系统)
10. [关键设计决策与原理](#10-关键设计决策与原理)
11. [可复用的设计模式](#11-可复用的设计模式)
12. [实现建议](#12-实现建议)

---

## 1. 项目概述

### 1.1 定位

Anthropic Skill Creator 是一个**元技能**（Meta-Skill）—— 用于创建、改进和评估其他技能的技能。它体现了 Anthropic 对 LLM 工程的核心理念：

- **迭代优化循环**：起草 → 测试 → 评估 → 改进 → 重复
- **人机协作**：自动化评估 + 人工反馈
- **渐进披露**：分层加载，按需扩展

### 1.2 核心能力

| 能力 | 描述 |
|------|------|
| 创建技能 | 从零开始编写 SKILL.md 及配套资源 |
| 改进技能 | 基于测试结果迭代优化 |
| 评估技能 | 量化测试 + 定性分析 |
| 描述优化 | 自动优化触发描述 |
| 打包分发 | 生成可安装的 .skill 文件 |

### 1.3 设计哲学

```
"不要写'ALWAYS'、'NEVER'这种大写词——用解释替代。现代 LLM 很聪明，理解原因比死记规则更有效。"
```

核心理念：
1. **解释"为什么"** 比 强制规则 更有效
2. **泛化** 比 过拟合 更有价值
3. **用户反馈** 是改进的核心驱动力
4. **量化数据** 辅助决策但不替代判断

---

## 2. 核心架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        SKILL.md (主控)                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  创建流程 │ 测试流程 │ 改进流程 │ 描述优化 │ 打包流程      │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        agents/ (代理系统)                        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │  grader.md   │ │ comparator.md│ │  analyzer.md │            │
│  │  (评分代理)   │ │ (盲对比代理) │ │ (分析代理)   │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       scripts/ (脚本系统)                        │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐      │
│  │ run_eval.py    │ │ aggregate_     │ │ run_loop.py    │      │
│  │ (触发测试)     │ │ benchmark.py   │ │ (优化循环)     │      │
│  │                │ │ (聚合统计)     │ │                │      │
│  └────────────────┘ └────────────────┘ └────────────────┘      │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐      │
│  │ improve_       │ │ package_       │ │ utils.py       │      │
│  │ description.py │ │ skill.py       │ │ (工具函数)     │      │
│  │ (描述改进)     │ │ (打包)         │ │                │      │
│  └────────────────┘ └────────────────┘ └────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       references/ (参考文档)                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  schemas.md — JSON 结构定义                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        assets/ (资源文件)                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  eval_review.html — 触发测试审查界面                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     eval-viewer/ (评估查看器)                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  generate_review.py — 生成评估报告 Web UI                   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 三级加载机制

```
┌─────────────────────────────────────────────────────────────┐
│ Level 1: 元数据 (始终加载, ~100词)                          │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ name: skill-name                                         │ │
│ │ description: 触发描述 (核心！)                           │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ Level 2: SKILL.md 主体 (触发时加载, <500行)                 │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 创建流程、测试流程、改进流程等详细指令                    │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ Level 3: 捆绑资源 (按需加载, 无限制)                        │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ scripts/ → 可执行脚本                                    │ │
│ │ references/ → 参考文档                                   │ │
│ │ assets/ → 静态资源                                       │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 目录结构与文件职责

### 3.1 完整目录树

```
skill-creator/
├── SKILL.md                    # 主控文件 (核心指令)
├── LICENSE.txt                 # 许可证
│
├── agents/                     # 代理指令 (子代理读取)
│   ├── grader.md              # 评分代理：评估输出是否符合期望
│   ├── comparator.md          # 盲对比代理：A/B 盲测比较
│   └── analyzer.md            # 分析代理：分析结果、找出模式
│
├── scripts/                    # Python 脚本
│   ├── __init__.py            # 包初始化
│   ├── utils.py               # 通用工具函数
│   ├── run_eval.py            # 触发测试：测试描述是否正确触发
│   ├── run_loop.py            # 优化循环：eval + improve 迭代
│   ├── improve_description.py # 描述改进：LLM 辅助优化描述
│   ├── aggregate_benchmark.py # 聚合统计：生成 benchmark.json
│   ├── generate_report.py     # 报告生成：生成 HTML 报告
│   ├── package_skill.py       # 打包：生成 .skill 文件
│   └── quick_validate.py      # 快速验证：验证技能结构
│
├── references/                 # 参考文档
│   └── schemas.md             # JSON Schema 定义
│
├── assets/                     # 静态资源
│   └── eval_review.html       # 触发测试审查界面
│
└── eval-viewer/               # 评估查看器
    └── generate_review.py     # 生成评估报告 Web UI
```

### 3.2 文件职责详解

| 文件 | 职责 | 调用时机 |
|------|------|----------|
| `SKILL.md` | 主控逻辑，定义所有流程 | 用户请求创建/改进技能时 |
| `agents/grader.md` | 评分断言，判断输出是否符合预期 | 评估阶段 |
| `agents/comparator.md` | 盲对比，不知来源比较两个输出 | 高级对比阶段 |
| `agents/analyzer.md` | 分析结果，找出模式和建议改进 | 基准测试后 |
| `scripts/run_eval.py` | 测试触发率 | 描述优化阶段 |
| `scripts/run_loop.py` | 迭代优化描述 | 描述优化阶段 |
| `scripts/improve_description.py` | LLM 改进描述 | 优化循环中 |
| `scripts/aggregate_benchmark.py` | 聚合统计数据 | 评估完成后 |
| `scripts/package_skill.py` | 打包技能 | 最终输出时 |

---

## 4. 数据流与工作流

### 4.1 创建技能流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      技能创建流程                                │
└─────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │  用户请求    │ "我想创建一个技能..."
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  捕捉意图    │ 问4个问题:
    │              │ 1. 做什么？
    │              │ 2. 何时触发？
    │              │ 3. 输出格式？
    │              │ 4. 需要测试吗？
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  编写 SKILL.md│
    │  - name      │
    │  - description (关键!)
    │  - 主体指令  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  创建测试用例│
    │  evals.json  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  运行测试    │ ←─────────────────┐
    │  (并行!)     │                    │
    │  - with_skill│                    │
    │  - baseline  │                    │
    └──────┬───────┘                    │
           │                            │
           ▼                            │
    ┌──────────────┐                    │
    │  评分 + 聚合 │                    │
    │  grading.json│                    │
    │  benchmark.  │                    │
    │  json        │                    │
    └──────┬───────┘                    │
           │                            │
           ▼                            │
    ┌──────────────┐                    │
    │  启动查看器  │                    │
    │  (Web UI)    │                    │
    └──────┬───────┘                    │
           │                            │
           ▼                            │
    ┌──────────────┐                    │
    │  用户反馈    │                    │
    │  feedback.   │                    │
    │  json        │                    │
    └──────┬───────┘                    │
           │                            │
           ▼                            │
    ┌──────────────┐                    │
    │  改进技能    │ ───────────────────┘
    └──────┬───────┘      (如果不满)
           │
           ▼ (满意)
    ┌──────────────┐
    │  描述优化    │ (可选)
    │  run_loop.py │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  打包分发    │
    │  .skill 文件 │
    └──────────────┘
```

### 4.2 测试评估流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      测试评估流程                                │
└─────────────────────────────────────────────────────────────────┘

Step 1: 并行启动测试 (关键！同时启动!)
────────────────────────────────────────
    ┌─────────────────┐    ┌─────────────────┐
    │   with_skill    │    │    baseline     │
    │   子代理 #1     │    │    子代理 #2    │
    │                 │    │                 │
    │  读取技能       │    │  不读取技能     │
    │  执行任务       │    │  执行任务       │
    └────────┬────────┘    └────────┬────────┘
             │                      │
             ▼                      ▼
    outputs/with_skill/      outputs/baseline/

Step 2: 捕获计时数据 (立即保存!)
────────────────────────────────
    子代理完成通知:
    { total_tokens, duration_ms }
           │
           ▼
    timing.json

Step 3: 评分
────────────
    ┌─────────────────┐
    │  grader 子代理  │
    │  读取 grader.md │
    │  评估每个断言   │
    └────────┬────────┘
             │
             ▼
    grading.json:
    {
      "expectations": [
        { "text": "...", "passed": true, "evidence": "..." }
      ],
      "summary": { "passed": 2, "failed": 1, "pass_rate": 0.67 }
    }

Step 4: 聚合统计
────────────────
    python aggregate_benchmark.py <workspace>/iteration-N
    
    输出:
    - benchmark.json (完整数据)
    - benchmark.md (人类可读)
    
    包含:
    - run_summary (均值±标准差)
    - delta (改进量)
    - notes (分析师观察)

Step 5: 启动查看器
──────────────────
    python generate_review.py <workspace>/iteration-N \
      --skill-name "my-skill" \
      --benchmark benchmark.json
    
    打开 Web UI:
    - Outputs 标签: 查看每个测试的输出
    - Benchmark 标签: 量化对比
    - 反馈框: 用户输入

Step 6: 读取反馈
───────────────
    feedback.json:
    {
      "reviews": [
        { "run_id": "...", "feedback": "图表缺坐标轴" }
      ]
    }
```

### 4.3 描述优化流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      描述优化流程                                │
└─────────────────────────────────────────────────────────────────┘

Step 1: 生成测试查询集
─────────────────────
    20 个查询:
    - 8-10 个 should_trigger
    - 8-10 个 should_not_trigger
    
    特点:
    - 真实、具体、有细节
    - 有边缘情况
    - 有近义词陷阱

Step 2: 用户审查
───────────────
    用 eval_review.html 让用户:
    - 编辑查询
    - 切换 should_trigger
    - 添加/删除条目
    
    导出 → eval_set.json

Step 3: 运行优化循环
───────────────────
    python run_loop.py \
      --eval-set eval_set.json \
      --skill-path /path/to/skill \
      --model <model-id> \
      --max-iterations 5
    
    内部逻辑:
    ┌────────────────────────────────────┐
    │  1. 60/40 划分 train/test          │
    │  2. 评估当前描述                    │
    │  3. 如果失败:                       │
    │     调用 improve_description.py    │
    │     让 LLM 提出新描述               │
    │  4. 重复直到:                       │
    │     - 全部通过                      │
    │     - 达到最大迭代次数              │
    │  5. 选择 test 分数最高的版本        │
    └────────────────────────────────────┘

Step 4: 应用最佳描述
───────────────────
    更新 SKILL.md frontmatter
    
    展示 before/after:
    - 原始描述
    - 最佳描述
    - 分数提升
```

---

## 5. 核心模块详细分析

### 5.1 SKILL.md 结构分析

#### 5.1.1 Frontmatter

```yaml
---
name: skill-creator
description: |
  Create new skills, modify and improve existing skills, and measure skill 
  performance. Use when users want to create a skill from scratch, edit, or 
  optimize an existing skill, run evals to test a skill, benchmark skill 
  performance with variance analysis, or optimize a skill's description for 
  better triggering accuracy.
---
```

**设计要点**：
- description 是**唯一触发机制**
- 包含"做什么" + "何时使用"
- 要"激进"一点，避免触发不足
- 1024 字符硬限制

#### 5.1.2 主体结构

```markdown
# Skill Creator

## Communicating with the user
- 适应不同技术水平的用户
- 解释专业术语

## Creating a skill
### Capture Intent
### Interview and Research
### Write the SKILL.md
### Skill Writing Guide
### Test Cases

## Running and evaluating test cases
### Step 1: Spawn all runs
### Step 2: Draft assertions
### Step 3: Capture timing
### Step 4: Grade, aggregate, launch viewer
### Step 5: Read feedback

## Improving the skill
### How to think about improvements
### The iteration loop

## Advanced: Blind comparison

## Description Optimization
### Step 1-4

## Claude.ai-specific instructions
## Cowork-Specific Instructions
## Reference files
```

### 5.2 Grader 代理分析

#### 5.2.1 角色

评估执行结果是否满足期望断言。

#### 5.2.2 输入

```json
{
  "expectations": ["输出包含X", "使用了脚本Y"],
  "transcript_path": "path/to/transcript.md",
  "outputs_dir": "path/to/outputs/"
}
```

#### 5.2.3 流程

```
1. 读取转录文件
2. 检查输出文件
3. 评估每个断言
4. 提取并验证声明
5. 读取用户笔记
6. 批评评估本身 (重要！)
7. 输出评分结果
```

#### 5.2.4 关键特性

**双重职责**：
1. 评分输出
2. 批评评估本身

```markdown
"A passing grade on a weak assertion is worse than useless — 
it creates false confidence. When you notice an assertion that's 
trivially satisfied, or an important outcome that no assertion checks, say so."
```

**评分标准**：
- **PASS**: 有明确证据 + 证据反映真实完成
- **FAIL**: 无证据 / 证据矛盾 / 证据肤浅

### 5.3 Comparator 代理分析

#### 5.3.1 角色

**盲对比** — 在不知道来源的情况下比较两个输出。

#### 5.3.2 核心机制

```
┌─────────────────────────────────────┐
│  输出 A (来源未知)                   │
│  输出 B (来源未知)                   │
│  原始任务                            │
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  生成评分标准                        │
│  ┌─────────────────────────────────┐│
│  │ Content:                        ││
│  │   - Correctness (1-5)           ││
│  │   - Completeness (1-5)          ││
│  │   - Accuracy (1-5)              ││
│  │ Structure:                      ││
│  │   - Organization (1-5)          ││
│  │   - Formatting (1-5)            ││
│  │   - Usability (1-5)             ││
│  └─────────────────────────────────┘│
└───────────────────┬─────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│  评分 → 选择胜者 → 解释原因          │
└─────────────────────────────────────┘
```

#### 5.3.3 输出结构

```json
{
  "winner": "A",
  "reasoning": "Output A provides...",
  "rubric": {
    "A": {
      "content": { "correctness": 5, ... },
      "content_score": 4.7,
      "overall_score": 9.0
    },
    "B": { ... }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["Complete solution", ...],
      "weaknesses": ["Minor style issue", ...]
    }
  }
}
```

### 5.4 Analyzer 代理分析

#### 5.4.1 双重角色

1. **盲对比后分析** — 解释为什么胜者胜出
2. **基准测试分析** — 发现聚合统计看不到的模式

#### 5.4.2 基准测试分析模式

```python
# 模式检测逻辑
for each assertion:
    if always_passes_in_both_configs:
        # 不区分技能价值，可能需要改进
    if always_fails_in_both_configs:
        # 可能超出能力或断言有问题
    if passes_with_skill_fails_without:
        # 技能明显有价值的点
    if fails_with_skill_passes_without:
        # 技能可能有负面影响
    if high_variance:
        # 可能不稳定或依赖模型
```

---

## 6. JSON Schema 完整定义

### 6.1 evals.json — 测试定义

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的任务提示",
      "expected_output": "期望结果描述",
      "files": ["evals/files/sample.pdf"],
      "expectations": [
        "输出包含 X",
        "使用了脚本 Y"
      ]
    }
  ]
}
```

**字段说明**：

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `skill_name` | string | 是 | 技能名称 |
| `evals[].id` | int | 是 | 唯一标识 |
| `evals[].prompt` | string | 是 | 用户任务 |
| `evals[].expected_output` | string | 是 | 期望结果描述 |
| `evals[].files` | string[] | 否 | 输入文件路径 |
| `evals[].expectations` | string[] | 是 | 可验证的断言列表 |

### 6.2 grading.json — 评分结果

```json
{
  "expectations": [
    {
      "text": "输出包含名字 'John Smith'",
      "passed": true,
      "evidence": "在转录第3步找到: '提取的名字: John Smith...'"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "execution_metrics": {
    "tool_calls": { "Read": 5, "Write": 2, "Bash": 8 },
    "total_tool_calls": 15,
    "total_steps": 6,
    "errors_encountered": 0,
    "output_chars": 12450,
    "transcript_chars": 3200
  },
  "timing": {
    "executor_duration_seconds": 165.0,
    "grader_duration_seconds": 26.0,
    "total_duration_seconds": 191.0
  },
  "claims": [
    {
      "claim": "表单有12个可填充字段",
      "type": "factual",
      "verified": true,
      "evidence": "在 field_info.json 中计数了12个字段"
    }
  ],
  "user_notes_summary": {
    "uncertainties": ["使用了2023数据，可能过时"],
    "needs_review": [],
    "workarounds": ["对不可填充字段回退到文本覆盖"]
  },
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "输出包含名字 'John Smith'",
        "reason": "幻觉文档也会通过 — 考虑验证它是主要联系人"
      }
    ],
    "overall": "断言检查存在但不验证正确性"
  }
}
```

### 6.3 benchmark.json — 基准汇总

```json
{
  "metadata": {
    "skill_name": "pdf",
    "skill_path": "/path/to/pdf",
    "executor_model": "claude-sonnet-4-20250514",
    "timestamp": "2026-01-15T10:30:00Z",
    "evals_run": [1, 2, 3],
    "runs_per_configuration": 3
  },
  "runs": [
    {
      "eval_id": 1,
      "eval_name": "Ocean",
      "configuration": "with_skill",
      "run_number": 1,
      "result": {
        "pass_rate": 0.85,
        "passed": 6,
        "failed": 1,
        "total": 7,
        "time_seconds": 42.5,
        "tokens": 3800,
        "tool_calls": 18,
        "errors": 0
      },
      "expectations": [...],
      "notes": [...]
    }
  ],
  "run_summary": {
    "with_skill": {
      "pass_rate": { "mean": 0.85, "stddev": 0.05, "min": 0.80, "max": 0.90 },
      "time_seconds": { "mean": 45.0, "stddev": 12.0, "min": 32.0, "max": 58.0 },
      "tokens": { "mean": 3800, "stddev": 400, "min": 3200, "max": 4100 }
    },
    "without_skill": {
      "pass_rate": { "mean": 0.35, "stddev": 0.08, "min": 0.28, "max": 0.45 },
      "time_seconds": { "mean": 32.0, "stddev": 8.0, "min": 24.0, "max": 42.0 },
      "tokens": { "mean": 2100, "stddev": 300, "min": 1800, "max": 2500 }
    },
    "delta": {
      "pass_rate": "+0.50",
      "time_seconds": "+13.0",
      "tokens": "+1700"
    }
  },
  "notes": [
    "断言 '输出是PDF文件' 在两种配置下都100%通过 — 可能不区分技能价值",
    "Eval 3 显示高方差 (50% ± 40%) — 可能不稳定"
  ]
}
```

### 6.4 timing.json — 计时数据

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3,
  "executor_start": "2026-01-15T10:30:00Z",
  "executor_end": "2026-01-15T10:32:45Z",
  "executor_duration_seconds": 165.0,
  "grader_start": "2026-01-15T10:32:46Z",
  "grader_end": "2026-01-15T10:33:12Z",
  "grader_duration_seconds": 26.0
}
```

**重要**：计时数据**必须立即保存**，因为它只在子代理完成通知中出现，不会持久化。

### 6.5 comparison.json — 盲对比结果

```json
{
  "winner": "A",
  "reasoning": "Output A 提供完整解决方案...",
  "rubric": {
    "A": {
      "content": { "correctness": 5, "completeness": 5, "accuracy": 4 },
      "structure": { "organization": 4, "formatting": 5, "usability": 4 },
      "content_score": 4.7,
      "structure_score": 4.3,
      "overall_score": 9.0
    },
    "B": { ... }
  },
  "output_quality": {
    "A": {
      "score": 9,
      "strengths": ["Complete solution", "Well-formatted"],
      "weaknesses": ["Minor style inconsistency"]
    }
  },
  "expectation_results": {
    "A": { "passed": 4, "total": 5, "pass_rate": 0.80 },
    "B": { "passed": 3, "total": 5, "pass_rate": 0.60 }
  }
}
```

### 6.6 history.json — 版本历史

```json
{
  "started_at": "2026-01-15T10:30:00Z",
  "skill_name": "pdf",
  "current_best": "v2",
  "iterations": [
    {
      "version": "v0",
      "parent": null,
      "expectation_pass_rate": 0.65,
      "grading_result": "baseline",
      "is_current_best": false
    },
    {
      "version": "v1",
      "parent": "v0",
      "expectation_pass_rate": 0.75,
      "grading_result": "won",
      "is_current_best": false
    },
    {
      "version": "v2",
      "parent": "v1",
      "expectation_pass_rate": 0.85,
      "grading_result": "won",
      "is_current_best": true
    }
  ]
}
```

---

## 7. 代理系统设计

### 7.1 代理协作模式

```
┌─────────────────────────────────────────────────────────────────┐
│                        主代理 (SKILL.md)                         │
│                                                                 │
│  职责: 协调整个流程，与用户交互                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │  Executor  │  │   Grader   │  │ Comparator │
    │  (执行器)  │  │  (评分器)  │  │  (对比器)  │
    └────────────┘  └────────────┘  └────────────┘
                           │
                           ▼
                    ┌────────────┐
                    │  Analyzer  │
                    │  (分析器)  │
                    └────────────┘
```

### 7.2 代理指令设计原则

从源码中提取的设计原则：

#### Grader 代理

```markdown
## 评分标准

**PASS 当**:
- 转录或输出明确证明期望为真
- 可以引用具体证据
- 证据反映真实实质，不只是表面合规

**FAIL 当**:
- 没有找到期望的证据
- 证据与期望矛盾
- 期望无法从可用信息验证
- 证据是肤浅的 — 断言技术上满足但底层任务结果是错误或不完整的

**不确定时**: 通过的举证责任在于期望。
```

**关键设计**：双重职责 — 评分 + 批评评估本身

#### Comparator 代理

```markdown
## 指导原则

- **保持盲测**: 不要试图推断哪个技能产生了哪个输出
- **要具体**: 引用具体例子解释优缺点
- **要果断**: 除非输出真的等效，否则选择胜者
- **输出质量优先**: 断言分数次于整体任务完成度
- **解释推理**: reasoning 字段应该清楚说明为什么选择胜者
```

#### Analyzer 代理

```markdown
## 模式检测

查看每个断言在所有运行中的表现:
- 是否**在两种配置下都总是通过**？(可能不区分技能价值)
- 是否**在两种配置下都总是失败**？(可能断言有问题)
- 是否**有技能时通过，无技能时失败**？(技能明显有价值)
- 是否**高方差**？(可能不稳定)
```

---

## 8. 脚本系统设计

### 8.1 脚本概览

| 脚本 | 功能 | 输入 | 输出 |
|------|------|------|------|
| `run_eval.py` | 测试触发率 | eval_set.json | JSON 结果 |
| `run_loop.py` | 优化循环 | eval_set.json | results.json |
| `improve_description.py` | LLM 改进描述 | 评估结果 | 新描述 |
| `aggregate_benchmark.py` | 聚合统计 | 运行目录 | benchmark.json |
| `package_skill.py` | 打包技能 | 技能目录 | .skill 文件 |
| `utils.py` | 工具函数 | - | - |

### 8.2 run_eval.py 详细分析

#### 核心逻辑

```python
def run_single_query(query, skill_name, skill_description, timeout, project_root, model):
    """
    测试单个查询是否触发技能
    
    方法:
    1. 创建临时命令文件在 .claude/commands/
    2. 运行 `claude -p` 带查询
    3. 监听流事件检测触发
    4. 清理临时文件
    """
    
    # 创建临时技能命令
    command_file = project_commands_dir / f"{skill_name}-skill-{unique_id}.md"
    command_content = f"""---
description: |
  {skill_description}
---

# {skill_name}

This skill handles: {skill_description}
"""
    
    # 运行 claude -p 并监听流事件
    cmd = ["claude", "-p", query, "--output-format", "stream-json", 
           "--verbose", "--include-partial-messages"]
    
    # 早期检测: 监听 content_block_start 事件
    # 如果工具名是 "Skill" 或 "Read"，且包含技能名，则触发
```

#### 关键设计

1. **早期检测**：不等待完整响应，监听流事件
2. **临时命令**：创建临时技能文件测试触发
3. **并行执行**：ProcessPoolExecutor 并行测试多个查询

### 8.3 run_loop.py 详细分析

#### 核心逻辑

```python
def run_loop(eval_set, skill_path, ...):
    """
    优化循环:
    1. 60/40 划分 train/test
    2. 评估当前描述
    3. 如果失败，让 LLM 改进
    4. 重复直到全部通过或达到最大迭代
    5. 选择 test 分数最高的版本
    """
    
    # 划分数据集
    train_set, test_set = split_eval_set(eval_set, holdout=0.4)
    
    for iteration in range(1, max_iterations + 1):
        # 评估
        all_results = run_eval(...)
        
        # 分离 train/test 结果
        train_results = [...]
        test_results = [...]
        
        # 记录历史
        history.append({
            "iteration": iteration,
            "description": current_description,
            "train_passed": ...,
            "test_passed": ...,
        })
        
        # 如果全部通过，退出
        if train_summary["failed"] == 0:
            break
            
        # 改进描述
        new_description = improve_description(...)
        current_description = new_description
    
    # 选择最佳版本 (按 test 分数)
    best = max(history, key=lambda h: h["test_passed"] or 0)
    return best["description"]
```

#### 关键设计

1. **Train/Test 划分**：防止过拟合
2. **盲化历史**：改进时不让 LLM 看到 test 分数
3. **最佳选择**：按 test 分数选最佳，而非最终版本

### 8.4 improve_description.py 详细分析

#### 核心逻辑

```python
def improve_description(skill_name, skill_content, current_description, 
                        eval_results, history, model):
    """
    让 LLM 改进描述
    
    输入:
    - 当前描述
    - 失败案例 (应该触发但没触发 / 不该触发但触发了)
    - 历史尝试 (不要重复)
    - 技能内容 (上下文)
    
    输出:
    - 新描述 (100-200词, <1024字符)
    """
    
    # 构建提示词
    prompt = f"""You are optimizing a skill description...

Current description: "{current_description}"

FAILED TO TRIGGER:
{failed_triggers}

FALSE TRIGGERS:
{false_triggers}

PREVIOUS ATTEMPTS (do NOT repeat):
{history}

Skill content:
{skill_content}

Tips:
- Use imperative: "Use this skill for..."
- Focus on user intent, not implementation
- Be distinctive
- Keep 100-200 words, under 1024 chars

Respond with only the new description in <new_description> tags.
"""
    
    # 调用 LLM
    text = _call_claude(prompt, model)
    
    # 解析结果
    description = extract_new_description(text)
    
    # 如果超长，再次请求缩短
    if len(description) > 1024:
        description = shorten(description, model)
    
    return description
```

### 8.5 aggregate_benchmark.py 详细分析

#### 核心逻辑

```python
def aggregate_results(results):
    """
    聚合运行结果为统计摘要
    
    输入:
    - results: { "with_skill": [...], "without_skill": [...] }
    
    输出:
    - run_summary: {
        "with_skill": { "pass_rate": {mean, stddev, min, max}, ... },
        "without_skill": { ... },
        "delta": { "pass_rate": "+0.50", ... }
      }
    """
    
    for config in configs:
        runs = results[config]
        pass_rates = [r["pass_rate"] for r in runs]
        
        run_summary[config] = {
            "pass_rate": calculate_stats(pass_rates),
            "time_seconds": calculate_stats(times),
            "tokens": calculate_stats(tokens)
        }
    
    # 计算改进量
    run_summary["delta"] = {
        "pass_rate": f"{primary - baseline:+.2f}",
        "time_seconds": f"{primary - baseline:+.1f}",
        "tokens": f"{primary - baseline:+.0f}"
    }
    
    return run_summary

def calculate_stats(values):
    """计算均值、标准差、最小、最大"""
    n = len(values)
    mean = sum(values) / n
    variance = sum((x - mean) ** 2 for x in values) / (n - 1)
    stddev = math.sqrt(variance)
    return { "mean": mean, "stddev": stddev, "min": min, "max": max }
```

---

## 9. 触发优化系统

### 9.1 触发机制解析

```
┌─────────────────────────────────────────────────────────────────┐
│                      技能触发流程                                │
└─────────────────────────────────────────────────────────────────┘

    用户查询
        │
        ▼
    ┌─────────────────────────────────────┐
    │  Claude 检查 available_skills 列表  │
    │                                     │
    │  [                                  │
    │    { name, description },           │
    │    { name, description },           │
    │    ...                              │
    │  ]                                  │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │  决策: 是否需要某个技能?            │
    │                                     │
    │  - 简单任务 → 直接处理              │
    │  - 复杂任务 → 查找匹配技能          │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │  如果匹配: 读取 SKILL.md            │
    │  如果不匹配: 直接回答               │
    └─────────────────────────────────────┘
```

### 9.2 描述优化策略

#### 从 improve_description.py 提取的策略

```markdown
Tips that we've found to work well:

1. **祈使语气**: "Use this skill for..." 而非 "This skill does..."

2. **关注用户意图**: 描述用户想达成什么，而非技能如何工作

3. **区分度**: 描述要独特、易识别，与其他技能竞争注意力

4. **泛化**: 不要列举具体查询，要概括为用户意图类别

5. **简洁**: 100-200词，<1024字符

6. **变化**: 如果多次失败，尝试不同的句式结构
```

#### 描述模板示例

```markdown
<!-- 好 -->
description: |
  Use this skill for building dashboards, data visualization, and displaying 
  company metrics. Trigger whenever the user mentions dashboards, charts, 
  graphs, data visualization, internal metrics, or wants to display any 
  kind of company data, even if they don't explicitly ask for a 'dashboard.'

<!-- 差 (太被动) -->
description: |
  How to build a simple fast dashboard to display internal data.
```

### 9.3 测试查询设计

#### 好查询 vs 坏查询

```json
// 坏 - 太抽象
{ "query": "Format this data", "should_trigger": true }

// 好 - 真实、具体、有细节
{ 
  "query": "ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think",
  "should_trigger": true
}
```

#### 测试集组成

```
20 个查询:
├── should_trigger (8-10)
│   ├── 正式表达
│   ├── 随意表达
│   ├── 边缘情况
│   └── 与其他技能竞争的场景
│
└── should_not_trigger (8-10)
    ├── 近义词陷阱
    ├── 模糊表达
    ├── 相邻领域
    └── 其他工具更合适的场景
```

---

## 10. 关键设计决策与原理

### 10.1 为什么并行启动测试？

```python
# 正确做法: 同时启动
spawn(with_skill_subagent)
spawn(baseline_subagent)  # 同一回合！

# 错误做法: 串行
spawn(with_skill_subagent)
wait_for_completion()
spawn(baseline_subagent)  # 太晚了
```

**原因**：
1. 节省时间 — 并行执行
2. 同步完成 — 便于对比
3. 避免遗忘 — 防止忘记启动 baseline

### 10.2 为什么立即保存计时数据？

```python
# 子代理完成通知 (唯一机会!)
notification = {
    "total_tokens": 84852,
    "duration_ms": 23332
}

# 必须立即保存
timing_file.write_text(json.dumps(notification))
```

**原因**：
- 计时数据**只在通知中出现**
- 不会持久化到其他地方
- 之后无法恢复

### 10.3 为什么 Train/Test 划分？

```
┌─────────────────────────────────────────┐
│  优化循环使用 train 分数改进            │
│  最终选择使用 test 分数                 │
└─────────────────────────────────────────┘
```

**原因**：
- 防止过拟合到测试集
- LLM 改进时只能看到 train 分数
- 最佳版本由 test 分数决定

### 10.4 为什么评分代理要批评评估本身？

```json
{
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "输出包含名字 'John Smith'",
        "reason": "幻觉文档也会通过 — 考虑验证它是主要联系人"
      }
    ]
  }
}
```

**原因**：
- 弱断言通过 → 虚假信心
- 重要结果无断言 → 漏洞
- 评分代理最了解输出质量

### 10.5 为什么描述要"激进"？

```
"Claude has a tendency to 'undertrigger' skills -- to not use them 
when they'd be useful. To combat this, please make the skill 
descriptions a little bit 'pushy'."
```

**原因**：
- Claude 默认偏向"触发不足"
- 宁可多触发也不要漏掉
- 用户会自然纠正错误触发

---

## 11. 可复用的设计模式

### 11.1 渐进披露模式

```
Level 1 (始终): 元数据 (name + description)
Level 2 (触发时): 主体指令
Level 3 (按需): 捆绑资源
```

**应用场景**：所有需要分层的技能

### 11.2 代理协作模式

```
主代理 (协调) 
    │
    ├── 执行代理 (做事)
    ├── 评分代理 (评价)
    ├── 对比代理 (盲比)
    └── 分析代理 (总结)
```

**应用场景**：复杂评估流程

### 11.3 迭代优化模式

```
while not done:
    1. 测试
    2. 收集反馈 (人工 + 自动)
    3. 改进
    4. 重复
```

**应用场景**：所有需要渐进改进的场景

### 11.4 盲对比模式

```
输出 A (来源未知) vs 输出 B (来源未知)
    │
    ▼
独立评判 → 选择胜者 → 揭示来源
```

**应用场景**：需要无偏见比较的场景

### 11.5 Train/Test 防过拟合模式

```
评估数据 ─┬─ 60% → Train (用于改进)
          └─ 40% → Test (用于选择最佳)
```

**应用场景**：机器学习式迭代

### 11.6 流式事件检测模式

```python
# 不等待完整响应
while stream:
    event = parse_stream_event()
    if event.type == "content_block_start":
        if event.tool_name == "Skill":
            return True  # 早期检测!
```

**应用场景**：需要快速响应的场景

---

## 12. 实现建议

### 12.1 最小可行实现 (MVP)

如果要实现一个简化版，优先实现：

```
核心组件:
├── SKILL.md (主控)
├── agents/grader.md (评分)
├── scripts/aggregate_benchmark.py (聚合)
└── 评估查看器 (简化版)
```

可以暂时跳过：
- 盲对比 (高级功能)
- 描述优化 (可选)
- 打包 (最后再做)

### 12.2 适配 OpenClaw

| Anthropic | OpenClaw | 备注 |
|-----------|----------|------|
| `claude -p` | `sessions_spawn` | 子代理执行 |
| 子代理通知 | 异步回调 | 计时数据捕获 |
| Web 查看器 | Canvas | 评估报告展示 |
| ProcessPoolExecutor | 内置并行 | 触发测试 |

### 12.3 关键适配点

#### 1. 子代理执行

```python
# Anthropic 方式
subprocess.run(["claude", "-p", query])

# OpenClaw 方式
sessions_spawn(
    task=query,
    runtime="subagent",
    cwd=workspace
)
```

#### 2. 触发测试

```python
# Anthropic: 创建临时命令文件
command_file = project_commands_dir / f"{skill_name}.md"

# OpenClaw: 需要适配技能加载机制
# 可能需要通过 sessions_spawn 带技能路径
```

#### 3. 评估查看器

```python
# Anthropic: 生成 HTML 并打开浏览器
webbrowser.open(str(live_report_path))

# OpenClaw: 使用 Canvas 展示
canvas(
    action="present",
    url="file:///path/to/report.html"
)
```

### 12.4 建议的实现顺序

```
Phase 1: 核心流程
├── 创建 SKILL.md 主控
├── 实现测试用例格式
├── 实现评分代理
└── 实现基础聚合

Phase 2: 用户交互
├── 实现评估查看器
├── 实现反馈收集
└── 实现迭代循环

Phase 3: 高级功能
├── 实现盲对比
├── 实现描述优化
└── 实现打包

Phase 4: 集成优化
├── 适配 OpenClaw 特性
├── 性能优化
└── 文档完善
```

---

## 附录 A: 完整文件清单

```
skill-creator/
├── SKILL.md                    # ~33KB, 核心指令
├── LICENSE.txt                 # 许可证
│
├── agents/
│   ├── grader.md              # ~9KB, 评分代理指令
│   ├── comparator.md          # ~8KB, 盲对比代理指令
│   └── analyzer.md            # ~11KB, 分析代理指令
│
├── scripts/
│   ├── __init__.py
│   ├── utils.py               # ~2KB, 工具函数
│   ├── run_eval.py            # ~12KB, 触发测试
│   ├── run_loop.py            # ~14KB, 优化循环
│   ├── improve_description.py # ~12KB, 描述改进
│   ├── aggregate_benchmark.py # ~15KB, 聚合统计
│   ├── generate_report.py     # 报告生成
│   ├── package_skill.py       # ~5KB, 打包
│   └── quick_validate.py      # 快速验证
│
├── references/
│   └── schemas.md             # ~13KB, JSON Schema
│
├── assets/
│   └── eval_review.html       # ~8KB, 审查界面
│
└── eval-viewer/
    └── generate_review.py     # 评估报告生成
```

---

## 附录 B: 关键代码片段

### B.1 解析 SKILL.md

```python
def parse_skill_md(skill_path: Path) -> tuple[str, str, str]:
    """解析 SKILL.md，返回 (name, description, full_content)"""
    content = (skill_path / "SKILL.md").read_text()
    lines = content.split("\n")
    
    # 查找 frontmatter
    if lines[0].strip() != "---":
        raise ValueError("SKILL.md missing frontmatter")
    
    end_idx = None
    for i, line in enumerate(lines[1:], start=1):
        if line.strip() == "---":
            end_idx = i
            break
    
    # 解析 YAML 字段
    name = ""
    description = ""
    for line in lines[1:end_idx]:
        if line.startswith("name:"):
            name = line[len("name:"):].strip()
        elif line.startswith("description:"):
            # 处理多行描述
            value = line[len("description:"):].strip()
            if value in (">", "|", ">-", "|-"):
                # 多行 YAML
                ...
            else:
                description = value
    
    return name, description, content
```

### B.2 统计计算

```python
def calculate_stats(values: list[float]) -> dict:
    """计算均值、标准差、最小、最大"""
    if not values:
        return {"mean": 0.0, "stddev": 0.0, "min": 0.0, "max": 0.0}
    
    n = len(values)
    mean = sum(values) / n
    
    if n > 1:
        variance = sum((x - mean) ** 2 for x in values) / (n - 1)
        stddev = math.sqrt(variance)
    else:
        stddev = 0.0
    
    return {
        "mean": round(mean, 4),
        "stddev": round(stddev, 4),
        "min": round(min(values), 4),
        "max": round(max(values), 4)
    }
```

### B.3 描述长度检查

```python
# 如果超长，再次请求缩短
if len(description) > 1024:
    shorten_prompt = (
        f"A previous attempt produced this description, which at "
        f"{len(description)} characters is over the 1024-character hard limit:\n\n"
        f'"{description}"\n\n'
        f"Rewrite it to be under 1024 characters while keeping the most "
        f"important trigger words and intent coverage."
    )
    description = _call_claude(shorten_prompt, model)
```

---

## 附录 C: 设计理念总结

### 核心理念

1. **迭代优化**: 起草 → 测试 → 评估 → 改进 → 重复
2. **人机协作**: 自动化评估 + 人工反馈
3. **渐进披露**: 分层加载，按需扩展
4. **解释优先**: 用原因替代强制规则
5. **泛化思维**: 不要过拟合具体案例

### 关键原则

1. **description 是触发核心** — 写得激进一点
2. **并行测试** — 同时启动 with-skill 和 baseline
3. **立即保存计时** — 唯一机会
4. **Train/Test 划分** — 防止过拟合
5. **评分代理批评评估** — 发现弱断言
6. **早期流事件检测** — 不等待完整响应

### 避免的反模式

1. ❌ 串行启动测试
2. ❌ 忘记保存计时数据
3. ❌ 描述太被动
4. ❌ 断言只检查表面
5. ❌ 列举具体查询而非泛化意图
6. ❌ 用 ALWAYS/NEVER 大写词替代解释

---

**报告完成**

本报告详细分析了 Anthropic Skill Creator 的完整架构、数据流、模块设计、代理系统、脚本系统和关键设计决策。可作为实现自定义 Skill Creator 的完整参考。