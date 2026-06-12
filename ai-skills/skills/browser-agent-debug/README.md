# Browser Agent Debug Skill - 通用版

## 📖 概述

这是一个通用的浏览器自动化调试技能，可以在**任何 AI 编程工具**中使用（Lingma、Cursor、Claude Code、GitHub Copilot 等）。

通过 CDP（Chrome DevTools Protocol）连接到我们自己的 CloakBrowser，实现：
- ✅ **跨平台兼容**：不依赖特定 AI 工具的内置 Browser Agent
- ✅ **反检测能力**：CloakBrowser 源码级修改 Chromium，30/30 通过检测测试
- ✅ **持久化登录**：独立用户数据目录，保存登录状态
- ✅ **完全控制**：自主管理浏览器生命周期

## 🚀 快速开始

### 1. 安装依赖

```bash
cd f:/myclaw/multi-platform-automation
pip install cloakbrowser playwright
```

### 2. 启动 CloakBrowser

```bash
# 百家号示例
python f:/myclaw/test/start_logged_in_browser.py --platform baijia --url https://baijiahao.baidu.com/builder/rc/edit?type=news

# 头条号示例
python f:/myclaw/test/start_logged_in_browser.py --platform toutiao --url https://mp.toutiao.com/profile_v4/graphic/publish
```

等待看到以下输出：
```
✅ 浏览器已就绪！
Browser Agent 现在可以连接到 CDP 端口 9222 进行调试
按 Ctrl+C 关闭浏览器
```

### 3. 在 AI 工具中连接

在任何 AI 工具中执行以下代码：

```python
from playwright.sync_api import sync_playwright

# 连接到 CDP 端口 9222
playwright = sync_playwright().start()
browser = playwright.chromium.connect_over_cdp("http://localhost:9222")

# 复用已有上下文（重要！不要创建新的）
context = browser.contexts[0]
page = context.pages[0] if context.pages else context.new_page()

# 开始调试
page.screenshot(path="f:/myclaw/test/debug_start.png")
print(f"当前页面: {page.url}")
print(f"页面标题: {page.title()}")
```

### 4. 执行调试任务

按照 SKILL.md 中的三种测试方案执行调试任务：
- **方案1**：逐步提取测试法（新脚本调试）
- **方案2**：Bug定位修复法（已知报错的脚本）
- **方案3**：用户截图反馈法（用户报告的问题）

### 5. 关闭浏览器

在 `start_logged_in_browser.py` 的终端中按 `Ctrl+C`。

## 📁 文件结构

```
.lingma/skills/browser-agent-debug/
├── SKILL.md          # 完整的技能文档（可在任何 AI 工具中使用）
└── README.md         # 本文件（快速开始指南）

test/
└── start_logged_in_browser.py  # 启动已登录浏览器的脚本

multi-platform-automation/
├── utils/
│   └── browser_automation.py   # BrowserAutomation 类（支持 CloakBrowser）
└── browser_data/
    └── cloak/
        ├── baijia/              # 百家号持久化用户数据
        ├── toutiao/             # 头条号持久化用户数据
        └── ...                  # 其他平台
```

## 🔧 核心原理

### 架构图

```
┌─────────────────────────────────────────────────┐
│           AI 编程工具（任意）                      │
│  ┌──────────────────────────────────────┐       │
│  │  Browser Agent / Subagent            │       │
│  │  - 接收任务指令                        │       │
│  │  - 执行 Playwright 代码               │       │
│  │  - 截图分析                           │       │
│  └──────────────┬───────────────────────┘       │
│                 │ CDP 连接 (ws://localhost:9222) │
└─────────────────┼───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│         CloakBrowser（反检测浏览器）              │
│  ┌──────────────────────────────────────┐       │
│  │  Chromium（源码级修改）                │       │
│  │  - 消除 navigator.webdriver          │       │
│  │  - Canvas/WebGL 指纹伪装             │       │
│  │  - TLS 指纹伪装                      │       │
│  │  - 30/30 通过检测测试                │       │
│  └──────────────────────────────────────┘       │
│                                                 │
│  ┌──────────────────────────────────────┐       │
│  │  持久化用户数据目录                     │       │
│  │  - browser_data/cloak/{platform}/    │       │
│  │  - 保存登录状态（Cookies/LocalStorage）│       │
│  │  - 下次启动自动登录                    │       │
│  └──────────────────────────────────────┘       │
└─────────────────────────────────────────────────┘
```

### 关键要点

1. **CDP 端口固定为 9222**：由 `start_logged_in_browser.py` 启动时配置
2. **必须复用已有上下文**：使用 `browser.contexts[0]`，不要创建新的
3. **持久化登录**：每个平台有独立的用户数据目录，保存登录状态
4. **反检测**：CloakBrowser 彻底消除自动化特征

## ❓ 常见问题

### Q1：为什么不能直接使用 Lingma 的 Browser Agent？

A：Lingma 的 Browser Agent 是专有的，只能在 Lingma 中使用。通过 CDP 连接到 CloakBrowser，可以在任何支持 Playwright 的 AI 工具中使用（Cursor、Claude Code、GitHub Copilot 等）。

### Q2：CloakBrowser 和普通 Playwright 有什么区别？

A：CloakBrowser 是源码级修改的 Chromium，彻底消除了自动化特征（navigator.webdriver、Canvas 指纹等），30/30 通过所有机器人检测测试，适合需要反检测的场景（如百家号、头条号等平台）。

### Q3：如何验证反检测效果？

A：访问 https://bot.sannysoft.com/ 或 https://abrahamjuliot.github.io/creepjs/，查看检测结果。

### Q4：如果 CloakBrowser 不可用怎么办？

A：`browser_automation.py` 中有降级方案，会自动切换到普通 Playwright。但建议优先使用 CloakBrowser。

### Q5：如何在其他 AI 工具中使用这个 Skill？

A：将 `SKILL.md` 文件复制到目标 AI 工具的 Skills 目录中，然后按照上述步骤操作即可。核心是通过 CDP 连接到 CloakBrowser，与具体的 AI 工具无关。

### Q6：多个 AI 工具可以同时连接吗？

A：可以！CDP 端口 9222 支持多个客户端同时连接。但建议一次只在一个 AI 工具中进行调试，避免冲突。

## 📝 更新日志

### v2.0（2026-05-20）
- ✅ 升级为通用版本，不再依赖 Lingma 内置 Browser Agent
- ✅ 支持通过 CDP 连接到 CloakBrowser
- ✅ 可在任何 AI 编程工具中使用
- ✅ 添加迁移指南和常见问题

### v1.0（早期版本）
- 仅支持 Lingma 内置 Browser Agent
- 功能有限，无法迁移到其他工具

## 🎯 下一步

1. 阅读完整的 [SKILL.md](SKILL.md) 文档
2. 尝试启动 CloakBrowser 并连接
3. 执行一个调试任务（如关闭百家号弹窗）
4. 验证反检测效果

---

**祝调试愉快！** 🎉
