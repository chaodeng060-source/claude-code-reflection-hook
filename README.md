# 🪞 给 Claude Code 装一个「反思系统」

> 让 AI 记住自己踩过的坑，越用越聪明 — 一个 PostToolUse hook 就够了

## 痛点

Claude Code 很强，但它不记得自己犯过的错。上次写错了 `DankeTheme.secondaryText`（正确的是 `textSecondary`），下次还会错。每次都要你纠正，累不累？

## 方案

一个 PostToolUse hook，每次编辑文件后自动检查 + 匹配经验库。犯过的错自动记住，下次写到同样的代码直接弹警告。

## 架构：三层防线

```
编辑文件 → 🔍 语法检查 → 📚 经验匹配 → 🧠 自动学习
```

### 第一层：实时语法检查

每次 Edit/Write 后自动跑：

- Python → `py_compile` + `import` 验证
- JSON → `json.loads` 解析
- Shell → `bash -n` 语法检查
- 自定义规则（比如检查 UI 组件属性是否存在）

### 第二层：经验库匹配（lessons.json）

存着过去踩过的坑，每条有 trigger + scope + weight：

```json
{
  "lessons": [
    {
      "scope": "*.swift",
      "triggers": ["DankeTheme.secondaryText", "DankeTheme.primaryText"],
      "lesson": "属性名是 textSecondary/textPrimary，改之前 grep 确认",
      "weight": 3,
      "hits": 2,
      "last_hit": 1779300000
    }
  ]
}
```

编辑代码时，diff 内容跟 triggers 匹配。命中了弹警告：

```
⚠️ 反思(权重3)：属性名是 textSecondary/textPrimary，改之前 grep 确认
```

trigger 支持两种模式：
- **普通字符串**：`"DankeTheme.secondaryText"` — 包含即命中
- **正则表达式**：`"regex:(?<!_)re\\.compile"` — 正则匹配

### 第三层：自动学习（候选池）

```
第一次犯错 → 进候选池（lesson_candidates.json）
同类错误第二次出现 → 自动升级为正式 lesson
30 天没再犯 → 候选过期清理
```

不需要手动维护经验库，系统自己从错误中学习。

## 自动进化机制

| 机制 | 说明 |
|------|------|
| **权重增减** | 命中一次 weight+1（上限10），90 天没命中 weight-1，降到 0 自动清理 |
| **防连续衰减** | 用 `last_decay_at` 记录衰减时间，90 天内不重复衰减 |
| **合并** | 同 scope 的 lessons，triggers 交集 ≥ 2 时自动合并 |
| **蒸馏** | 攒够 5+ 条时，用 LLM 分析提炼更通用的规则（dry-run 模式，只建议不自动执行） |

## 自测回放器

改了反思系统的代码，自动跑回归测试：

```
🧪 validate-edit.py 自测回放器

📋 Test: lesson 命中增重
  ✅ weight 增加
  ✅ hits 增加
  ✅ last_hit 存在

📋 Test: 衰减 + last_decay_at 防连续
  ✅ weight 衰减1
  ✅ last_decay_at 存在
  ✅ 第二次不衰减

📋 Test: 衰减到 0 自动清理
  ✅ lesson 被清理

结果: ✅ 10 passed, ❌ 0 failed
全部通过！🦊
```

通过 PostToolUse hook 自动触发 — 改了 `validate-edit.py` 后自动跑测试，失败就 block，防止静默失效。

## 快速开始

### Step 1：注册 hook

在 `.claude/settings.json` 里加：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/validate-edit.py"
          }
        ]
      }
    ]
  }
}
```

### Step 2：创建经验库

在 `.claude/hooks/lessons.json` 里写几条你遇到过的坑：

```json
{
  "lessons": [
    {
      "scope": "*.py",
      "triggers": ["import os.path"],
      "lesson": "用 from pathlib import Path 代替 os.path",
      "weight": 2,
      "created": "2026-05-21"
    }
  ]
}
```

### Step 3：创建候选池

```json
{}
```

保存为 `.claude/hooks/lesson_candidates.json`，系统会自动填充。

### Step 4：正常用 Claude Code

它会自动：
- 每次编辑后检查语法
- 匹配经验库弹警告
- 犯错自动记录，第二次自动升级为正式经验
- 长期不犯的经验自动衰减清理

## 核心思路

> 不是教 AI "应该怎么做"，而是让它记住 "上次哪里错了"。
> 
> 犯错 → 记录 → 下次匹配 → 弹警告 → 越用越准。

## 文件结构

```
.claude/hooks/
├── validate-edit.py          # 主 hook（语法检查 + 经验匹配 + 自动学习）
├── lessons.json              # 经验库
├── lesson_candidates.json    # 候选池（自动填充）
├── lesson_maintenance.py     # 维护脚本（合并 + 蒸馏）
├── test_validate_edit.py     # 自测回放器（10 个断言）
└── validate-reflection-tests.py  # 自动触发测试的 wrapper
```

---

🦊 蛋壳 × 蛋宝 | 2026-05-21

