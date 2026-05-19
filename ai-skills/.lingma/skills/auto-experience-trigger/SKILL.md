---
name: auto-experience-trigger
description: 全局自动经验沉淀触发器 - 全程静默运行，自动判定场景并生成可复用 Skill，杜绝重复耗时思考与无效试错。触发条件：调试超时40秒、试错3轮未解决、同源场景识别。
---

# 全局自动经验沉淀触发器

## 🎯 核心宗旨

全程静默运行，无需人工下达总结指令。执行任务过程中自动判定场景，达标自动完成经验固化，生成可复用 Skill，杜绝重复耗时思考与无效试错。

---

## ⚡ 自动触发判定规则

满足以下**任一条件**即立即启动经验沉淀流程：

### 1. 时长触发
处理以下问题时，整体推理+调试耗时 **超过 40 秒**：
- 代码报错调试
- 消息链路异常
- 接口对接故障
- 服务运维问题
- 业务逻辑错误

**动作**：立即终止发散式思考，启动经验提炼流程

**⚠️ 重要**：生成的 Skill 必须遵循以下规范：
1. 在 `.lingma/skills/` 下创建独立文件夹（如 `.lingma/skills/code-error-debugger/`）
2. 文件夹内创建 `SKILL.md` 文件
3. 顶部必须有 YAML Front Matter（name + description）
4. 完成后立即 Git 提交

### 2. 试错次数触发
针对同一个故障点，进行以下操作累计达到 **3 轮**仍未彻底解决：
- 修改代码测试
- 重启服务调试
- 调整参数尝试
- 更换方案验证

**动作**：停止盲目尝试，复盘整套流程

**⚠️ 重要**：生成的 Skill 必须遵循以下规范：
1. 在 `.lingma/skills/` 下创建独立文件夹（如 `.lingma/skills/retry-mechanism/`）
2. 文件夹内创建 `SKILL.md` 文件
3. 顶部必须有 YAML Front Matter（name + description）
4. 完成后立即 Git 提交

### 3. 同源场景触发
识别出当前问题属于历史已处理过的同类问题：
- 同类链路问题（如中间层消息流）
- 同类路由异常（如端口冲突）
- 同类客户端对接问题（如 Cursor 代理）
- 同类框架集成问题（如 Playwright）

**动作**：直接优先调用已有成熟 Skill 执行，不再重新推演逻辑

**⚠️ 注意**：如果找不到匹配的 Skill，解决问题后必须按上述规范创建新 Skill

---

## 🔄 触发后固定执行动作

### 第 1 步：复盘全流程
拆分本次操作中的两类步骤：
- ✅ **必须保留的核心步骤**（有效操作）
- ❌ **无意义的浪费操作**（无效尝试）

### 第 2 步：压缩精简
剔除以下内容：
- 多余的调试命令
- 多余的猜想分析
- 多余的排查步骤

梳理出**最短最优解决路径**

### 第 3 步：标准化成文
按照统一格式整理成 Skill 文档：

**📍 生成位置规范（强制）**：
- **必须**在 `.lingma/skills/` 目录下创建**独立文件夹**
- **文件夹命名**：使用小写字母 + 连字符（kebab-case），如 `port-conflict-resolver`
- **文件命名**：文件夹内必须包含 `SKILL.md` 文件
- **完整路径示例**：`.lingma/skills/port-conflict-resolver/SKILL.md`

**❌ 禁止**：直接在 `.lingma/skills/` 根目录创建 `.md` 文件

**Skill 文档格式**：
```markdown
---
name: skill-name-in-kebab-case
description: 简短描述 Skill 的功能和触发条件（不超过 100 字）
---

# [问题名称] 解决方案

## 适用场景
[什么情况下会触发这个问题]

## 故障现象
[具体的错误表现]

## 一键处理方案
[最简固定解决步骤，可直接复制执行]

## 原理说明（可选）
[简要解释为什么这样能解决]

## 常见变体（可选）
[类似问题的快速判断方法]
```

### 第 4 步：优先级锁定
- 新生成/优化后的 Skill 自动提升执行优先级
- 后续同类问题优先调用，不再从零开始思考

### 第 5 步：双仓库同步（强制）

Skill 生成后，**必须同时更新两个位置**：

#### **位置 1：本地 Lingma Skills**
```bash
# 已在第 3 步创建
# .lingma/skills/<skill-name>/SKILL.md
```

#### **位置 2：GitHub Skill 仓库**

**方式 A：使用自动化同步脚本（推荐）**
```powershell
# 运行同步脚本（自动检测变更 + Git 提交）
powershell -ExecutionPolicy Bypass -File "f:\myclaw\.lingma\skills\sync_skills_to_github.ps1"
```

**方式 B：手动同步**
```bash
# 复制到 GitHub 仓库
copy-item "f:\myclaw\.lingma\skills\<skill-name>" "f:\myclaw\git changku\santian\ai-skills\.lingma\skills\<skill-name>" -Recurse -Force

# 进入 GitHub 仓库目录
cd "f:\myclaw\git changku\santian"

# Git 提交
git add ai-skills/.lingma/skills/<skill-name>/
git commit -m "feat: 添加[Skill 名称]

- 新增 <skill-name>/SKILL.md
- 整合 [核心功能简述]
- 核心理念：[一句话概括]"
git push
```

**⚠️ 重要**：
- 两个位置的 Skill 文件必须**完全一致**
- 每次生成新 Skill 后**必须立即同步**
- 禁止只更新一个位置

### 第 6 步：禁止行为
触发总结期间：
- ❌ 不新增无关思路
- ❌ 不拓展多余方案
- ❌ 不在 `.lingma/skills/` 根目录直接创建 `.md` 文件
- ✅ 只固化最优落地解法
- ✅ 必须使用独立文件夹 + SKILL.md 格式

---

## 📊 执行优先级

```
全局自检沉淀规则 > 临场自由思考
既定排错流程 > 发散分析猜测
故障处理动作 > 长时间逻辑推演
```

---

## 💡 实际应用示例

### 示例 1：中间层消息流问题

**触发条件**：调试耗时超过 40 秒 + 多次查看日志未定位

**沉淀结果**：
- Skill 名称：`中间层消息流问题排查.md`
- 核心步骤：两端消息排查法 → 立即重启 → 定向修复
- 保存位置：`.lingma/skills/中间层消息流问题排查.md`

### 示例 2：端口占用冲突

**触发条件**：同一故障点试错 3 次（kill 进程、换端口、查占用）

**沉淀结果**：
- Skill 名称：`端口占用快速解决.md`
- 核心步骤：`netstat -ano | findstr <端口>` → `taskkill /PID <进程ID> /F`
- 保存位置：`.lingma/skills/端口占用快速解决.md`

### 示例 3：Playwright 元素找不到

**触发条件**：同源场景触发（历史已处理过类似问题）

**沉淀结果**：
- 直接调用已有 Skill：`网页端元素定位技巧.md`
- 不再重新推导选择器写法

---

## 🔍 监控范围

本触发器全程静默监控以下所有类型的问题：

| 问题类型 | 典型场景 | 触发阈值 |
|---------|---------|---------|
| 代码报错 | Python/JS 语法错误、运行时异常 | 40秒 或 3轮试错 |
| 消息链路 | 客户端 ↔ 中间层 ↔ 服务端通信异常 | 40秒 或 3轮试错 |
| 接口对接 | API 调用失败、参数错误、认证问题 | 40秒 或 3轮试错 |
| 服务运维 | 进程崩溃、端口冲突、资源不足 | 40秒 或 3轮试错 |
| 业务逻辑 | 数据处理错误、状态不一致、循环调用 | 40秒 或 3轮试错 |
| 框架集成 | Playwright/Docker/API 框架使用问题 | 40秒 或 3轮试错 |

---

## 📝 注意事项

1. **不要手动触发**：本规则全程自动运行，无需人工干预
2. **不要重复沉淀**：如果已有类似 Skill，优先复用而非新建
3. **保持精简**：每个 Skill 只保留最核心的解决步骤，不超过 50 行
4. **及时更新**：如果发现更优解法，立即更新现有 Skill
5. **分类存储**：按问题领域分文件夹存储，便于检索

---

## 🗂️ Skill 组织结构规范

**标准结构**（每个 Skill 必须是独立文件夹）：
```
.lingma/skills/
├── port-conflict-resolver/        # 端口冲突解决
│   └── SKILL.md
├── playwright-element-locator/    # Playwright 元素定位
│   └── SKILL.md
├── api-debugging-guide/           # API 接口调试
│   └── SKILL.md
├── python-dependency-fixer/       # Python 依赖冲突
│   └── SKILL.md
├── message-flow-troubleshooter/   # 中间层消息流排查
│   └── SKILL.md
└── ...                            # 其他领域问题
```

**❌ 错误结构**（禁止直接在根目录放 .md 文件）：
```
.lingma/skills/
├── 端口占用快速解决.md            # ❌ 禁止！
├── Playwright元素定位技巧.md      # ❌ 禁止！
└── ...
```

---

## 📅 创建时间
2026-05-18

## 🔗 适用范围
所有项目的所有类型问题（通用元规则）
