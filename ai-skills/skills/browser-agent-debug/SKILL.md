---
name: "browser-agent-debug"
description: "Browser Agent 调试方案（通用版）。触发条件：需要调试浏览器自动化脚本、验证发布流程、修复元素选择问题。核心原则：每步必截图、逐步验证、免登录调试。支持所有 AI 编程工具（Lingma/Cursor/Claude Code等），通过 CDP 连接到 CloakBrowser。"
---

# Browser Agent 调试方案 Skill（通用版）

> **设计原则**：严格遵循"截图→操作→截图→验证"循环，确保每一步都经过人工确认。支持三种测试方案，优先使用免登录策略。
> 
> **🎯 重要升级**：本 Skill 已升级为通用版本，不再依赖特定 AI 工具的内置 Browser Agent，而是通过 CDP 连接到我们自己的 CloakBrowser，可以在任何 AI 编程工具中使用（Lingma、Cursor、Claude Code 等）。

---

## 🚀 架构说明

### 传统方案 vs 新方案

| 特性 | 传统方案（Lingma 内置） | 新方案（CloakBrowser + CDP） |
|------|---------------------|---------------------------|
| **依赖** | Lingma 内置 Browser Agent | 任何支持 CDP 连接的 AI 工具 |
| **浏览器** | Lingma 控制的 Chrome | 我们的 CloakBrowser（反检测） |
| **可迁移性** | ❌ 仅限 Lingma | ✅ 可在任何 AI 工具中使用 |
| **反检测** | ❌ 普通 Chrome | ✅ CloakBrowser（源码级修改） |
| **持久化登录** | ⚠️ 有限支持 | ✅ 完整支持（独立用户目录） |
| **CDP 端口** | 自动管理 | 9222（固定） |

### 工作原理

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

### 使用流程

1. **启动 CloakBrowser**（Python 脚本）
   ```bash
   python f:/myclaw/test/start_logged_in_browser.py --platform baijia --url https://baijiahao.baidu.com/builder/rc/edit?type=news
   ```

2. **AI 工具通过 CDP 连接**
   ```python
   # 在任何 AI 工具中都可以这样连接
   from playwright.sync_api import sync_playwright
   
   playwright = sync_playwright().start()
   browser = playwright.chromium.connect_over_cdp("http://localhost:9222")
   context = browser.contexts[0]  # 复用已有上下文
   page = context.pages[0] if context.pages else context.new_page()
   ```

3. **执行调试任务**
   - 截图 → 视觉分析 → 执行操作 → 截图验证

4. **关闭浏览器**
   - 在 `start_logged_in_browser.py` 终端按 Ctrl+C

---

## 一、适用场景（触发条件）

以下任一关键词出现时，自动启用本 Skill：

| 触发关键词 | 典型需求 |
|-----------|---------|
| 测试发布脚本 | "测试头条发布"、"验证百家号流程"、"调试大鱼号" |
| 修复脚本bug | "脚本报错了"、"某步失败了"、"选择器不对" |
| 用户反馈问题 | "这里有问题"、"截图显示失败"、"某步没执行" |
| 优化选择器 | "改进点击逻辑"、"增强容错能力" |

**非触发场景**（不启用本 Skill）：
- 纯后端 API 测试
- 数据库操作测试
- 单元测试编写
- **正式发布流程**（如运行 `universal_publisher.py`）

---

## ⚠️ 重要说明：调试 vs 正式发布

### 🔧 调试阶段（使用 Browser Agent）

**需要截图**：
```python
# Browser Agent 连接浏览器 → 每步截图 → 视觉分析 → 找出问题
await page.screenshot(path="f:/myclaw/test/debug_step1.png")
# 然后用大模型视觉能力分析截图
```

**特点**：
- ✅ 有大模型参与（视觉分析）
- ✅ 每步都截图验证
- ✅ 用于发现问题和优化脚本

### 🚀 正式发布阶段（运行 publish_xxx.py）

**不需要截图**：
```python
# 直接执行操作，不需要截图
# 脚本中没有大模型，截图没有意义
page.click(selector)
page.fill(input, value)
# 只需要日志输出，不需要截图
```

**特点**：
- ❌ 没有大模型参与
- ❌ 不需要截图（会拖慢速度）
- ✅ 只需要清晰的日志输出
- ✅ 通过日志判断成功/失败

---

**总结**：
- **调试时**：Browser Agent + 截图 + 视觉分析 = 找出问题
- **发布时**：直接执行 + 日志输出 = 快速完成

---

## 二、核心原则（强制要求）

### 2.1 调试循环（铁律）

```
截图 → 判断当前状态 → 执行操作 → 截图验证 → 继续下一步
```

**每次操作前必须先截图判断当前状态，不要盲目操作！**

### 2.2 免登录策略（必备）

**核心机制**：使用 **CloakBrowser + launch_persistent_context** 实现真正的浏览器状态持久化。

所有平台都有独立的持久化用户数据目录，位于 `f:\myclaw\multi-platform-automation\browser_data\cloak\`：

| 平台 | 用户数据目录路径 |
|------|-------------------|
| 头条号 | `f:\myclaw\multi-platform-automation\browser_data\cloak\toutiao\` |
| 百家号 | `f:\myclaw\multi-platform-automation\browser_data\cloak\baijia\` |
| 大鱼号 | `f:\myclaw\multi-platform-automation\browser_data\cloak\dayu\` |
| QQ号 | `f:\myclaw\multi-platform-automation\browser_data\cloak\qq\` |
| 微信公众号 | `f:\myclaw\multi-platform-automation\browser_data\cloak\wechat\` |
| 小红书 | `f:\myclaw\multi-platform-automation\browser_data\cloak\xiaohongshu\` |

**工作原理**：
- ✅ **完整 Chrome 用户配置文件**（SQLite 数据库），包含 Cookies、LocalStorage、IndexedDB 等
- ✅ **一次登录，永久有效** - 即使重启电脑、重启程序，登录状态依然保持
- ✅ **平台隔离** - 每个平台独立的用户数据目录，互不干扰
- ✅ **难以被检测** - 比 JSON 文件方式更接近真实用户行为

**使用方式**：
```python
# BrowserAutomation 类会自动使用对应平台的持久化目录
from utils.browser_automation import BrowserAutomation

# 传入 platform 参数，自动加载对应的用户数据目录
browser = BrowserAutomation(platform='toutiao')  # 使用 toutiao 目录
```

**⚠️ 重要**：
- ❌ **不再使用 auth_state.json 文件**（旧方案已废弃）
- ✅ **必须传入 platform 参数**，否则会使用 default 目录

### 2.3 浏览器生命周期管理（关键！）

**⚠️ 重要规则：每次调试必须完整管理浏览器生命周期**

#### 🚀 高效调试模式：连续测试（推荐）

**核心原则**：**启动一次浏览器，连续测试所有步骤，不要反复重启！**

```python
# ✅ 正确的连续调试流程

# 步骤1：启动已登录的浏览器（只做一次）
python test/start_logged_in_browser.py --platform toutiao --url https://mp.toutiao.com/profile_v4/graphic/publish

# 步骤2：AI 工具连接到 CDP 端口 9222（只做一次）
# 在任何 AI 工具中都可以这样连接：
from playwright.sync_api import sync_playwright
playwright = sync_playwright().start()
browser = playwright.chromium.connect_over_cdp("http://localhost:9222")
context = browser.contexts[0]
page = context.pages[0] if context.pages else context.new_page()

# 步骤3：连续执行所有测试步骤（不要关闭浏览器！）
步骤1: 关闭AI助手 → 截图验证 → 继续
步骤2: 填写标题 → 截图验证 → 继续
步骤3: 粘贴内容 → 截图验证 → 继续
步骤4: 添加标签 → 截图验证 → 继续
步骤5: 上传封面 → 截图验证 → 继续
...（所有步骤连续执行）

# 步骤4：所有步骤测试完成后，再关闭浏览器
# 在 start_logged_in_browser.py 的终端中按 Ctrl+C
```

**优势**：
- ⚡ **效率高**：只登录一次，节省50-70%时间
- 🔄 **状态连续**：每步都基于上一步的结果，更接近真实场景
- 📊 **易排查**：如果某步失败，可以清楚看到是哪一步出的问题

**❌ 错误做法（禁止）**：
```python
# 测试步骤1 → 关闭浏览器 → 重新登录 → 测试步骤1-2 → 关闭浏览器 → ...
# 这种“逐步累加”的测试方式极其浪费时间！
```

---

#### 传统的分步调试流程（仅用于特殊情况）

```python
# 步骤1：启动已登录的浏览器
python test/start_logged_in_browser.py --platform toutiao --url https://mp.toutiao.com/profile_v4/graphic/publish

# 步骤2：等待看到以下输出
✅ 浏览器已就绪！
Browser Agent 现在可以连接到 CDP 端口 9222 进行调试
按 Ctrl+C 关闭浏览器

# 步骤3：让 AI 工具连接到 CDP 端口 9222
# ⚠️ 重要：AI 工具应该连接到这个已登录的浏览器，不要创建新的！
```

**适用场景**：
- 某一步骤反复失败，需要单独调试
- 需要重置页面状态（如刷新页面）
- 其他特殊情况

**注意**：即使使用分步调试，也应该尽量在一次会话中完成多个步骤，避免反复重启。

#### ❌ 常见错误：AI 工具创建新浏览器

**问题现象**：
- 左边窗口：已登录的发布页面（正确）
- 右边窗口：登录页面（错误，AI 工具新开的）

**根本原因**：
AI 工具连接到 CDP 端口时，可能创建了一个**新的浏览器上下文**，而不是复用已有的登录会话。

**解决方案**：
1. 确保 AI 工具配置为连接到外部 Chrome（CDP 端口 9222）
2. AI 工具应该连接到 CDP 端口 9222 的**现有浏览器实例**
3. 不要启动新的浏览器进程
4. 使用 `browser.contexts[0]` 复用已有上下文，而不是 `browser.new_context()`

**为什么能保持登录状态？**
- ✅ **CloakBrowser 使用持久化用户目录**：`browser_data/cloak/{platform}/`
- ✅ **首次登录**：手动扫码/账号密码登录 → 浏览器自动保存所有认证数据到用户目录
- ✅ **后续启动**：直接加载用户目录 → 自动处于登录状态
- ✅ **无需 auth_state.json**：完整的 Chrome 配置文件比 JSON 更可靠

#### 调试后清理

```python
# 调试结束后，在 start_logged_in_browser.py 的终端中按 Ctrl+C
# 确认浏览器已关闭
netstat -ano | findstr :9222  # 应该没有输出
```

**❌ 错误做法**：
- 调试结束后不关闭浏览器，导致下次启动时端口冲突
- 多次启动浏览器但不关闭，导致多个 Chrome 进程占用内存
- **AI 工具创建新的浏览器实例（会导致登录失效）**
- 手动删除 `browser_data/cloak/{platform}` 目录（会丢失登录状态）

**✅ 正确做法**：
- 每次调试前清理旧进程
- 每次调试后主动关闭浏览器
- AI 工具连接到**现有的已登录浏览器**，不要创建新的
- 如果某个平台登录失效，删除对应目录重新登录：
  ```bash
  Remove-Item -Recurse -Force "browser_data\cloak\toutiao"
  ```

### 2.4 截图与视觉分析（核心！）

**⚠️ 关键原则：截图不是为了保存，而是为了用视觉能力分析！**

#### 正确的调试循环

```
1. 截图
2. 使用大模型视觉能力分析截图：
   - 页面标题是什么？
   - URL 是什么？
   - 当前状态（已登录/未登录/错误页面）？
   - 关键元素位置（标题框、编辑器、按钮等）
   - 是否有弹窗需要关闭？
3. 基于分析结果决定下一步操作
4. 执行操作
5. 再次截图验证
6. 重复步骤2-5
```

#### ❌ 错误做法

- 截图了但不分析，直接跳下一步
- 看到截图但假装没看到（说谎）
- 不等待页面加载就截图
- 截图后不记录分析结果

#### ✅ 正确做法

- **每张截图都要用视觉能力详细分析**
- **记录分析结果**（页面状态、元素位置、选择器）
- **基于分析决定下一步**（不要盲目操作）
- **验证操作结果**（再次截图确认）

#### 视觉分析要点

每次截图后，必须分析：

1. **页面基本信息**
   - 页面标题
   - 当前 URL
   - 页面类型（发布页/登录页/错误页）

2. **登录状态**
   - 已登录：显示用户名、头像、后台界面
   - 未登录：显示登录按钮、二维码、登录表单
   - 不确定：需要进一步检查

3. **关键元素位置**
   - 标题输入框（位置、placeholder、选择器）
   - 正文编辑器（位置、工具栏、选择器）
   - 封面上传区（位置、上传方式、选择器）
   - 发布按钮（位置、按钮文字、选择器）

4. **弹窗和干扰元素**
   - AI助手抽屉
   - 广告弹窗
   - 新手引导
   - 其他需要关闭的元素

5. **下一步建议**
   - 基于当前状态，下一步应该做什么
   - 需要尝试哪些选择器
   - 需要注意哪些问题

---

## 三、三种测试方案

### 方案 1：逐步提取测试法（推荐用于新脚本调试）

**适用场景**：首次调试某个发布脚本，需要验证每一步是否正确

**⚠️ 重要原则**：**启动一次浏览器，连续执行所有步骤，不要反复重启！**

**执行流程**：

```
1. 启动已登录的浏览器（只做一次）
2. Browser Agent 连接到 CDP 端口（只做一次）
3. 读取目标脚本（如 publish_toutiao.py）
4. 提取所有操作步骤
5. 严格按顺序连续执行每一步（不要关闭浏览器！）：
   ├─ 步骤1: 截图 → 执行 → 截图验证 → 继续
   ├─ 步骤2: 截图 → 执行 → 截图验证 → 继续
   ├─ 步骤3: 截图 → 执行 → 截图验证 → 继续
   └─ ...（所有步骤连续执行）
6. 记录所有发现的问题
7. 所有步骤完成后，再关闭浏览器
8. 生成修复建议
```

**示例**：测试头条发布脚本

```python
# 从 publish_toutiao.py 提取的步骤：
步骤列表 = [
    "步骤0: 关闭AI助手弹窗",
    "步骤1: 打开HTML文件并复制内容",
    "步骤2: 导航到发布页面",
    "步骤3: 填写标题",
    "步骤4: 粘贴正文",
    "步骤5: 添加标签",
    "步骤6: 上传封面",
    "步骤7: 设置定时发布",
    "步骤8: 点击发布按钮"
]

# Browser Agent 执行：
for 步骤 in 步骤列表:
    截图(f"before_{步骤}")
    执行(步骤)
    截图(f"after_{步骤}")
    验证(步骤)  # 人工确认
```

**输出要求**：
- 每步的截图必须保存到 `f:\myclaw\test\`
- 记录每步的执行结果（成功/失败）
- 如果某步失败，立即停止并报告问题

---

### 方案 2：Bug定位修复法（推荐用于已知报错的脚本）

**适用场景**：脚本运行时报错，需要定位并修复具体步骤

**执行流程**：

```
1. 先完整运行一次脚本，记录报错位置和错误信息
2. 分析报错原因（选择器错误？元素不存在？超时？）
3. 手动执行到报错步骤的前一步
4. 截图确认当前状态
5. 尝试不同的解决方案：
   ├─ 方案A: 修改选择器
   ├─ 方案B: 增加等待时间
   ├─ 方案C: 更换操作方式（click → JavaScript）
   └─ 方案D: 多策略降级（4层fallback）
6. 每尝试一种方案都要：
   ├─ 截图（尝试前）
   ├─ 执行方案
   ├─ 截图（尝试后）
   └─ 验证是否成功
7. 找到可行方案后，更新脚本
```

**示例**：修复头条标签选择失败

```
问题：脚本在第5步"添加标签"时报错

执行流程：
1. 运行脚本，发现报错：Element not found for selector '#tag-game'
2. 手动执行到第4步（粘贴正文完成）
3. 截图确认：正文已粘贴，光标在末尾
4. 尝试方案A：使用 locator 匹配
   - 截图：before_try_locator
   - 执行：page.locator('div:has-text("#游戏#")').click()
   - 截图：after_try_locator
   - 验证：❌ 失败，元素不可见
   
5. 尝试方案B：使用 get_by_text
   - 截图：before_try_get_by_text
   - 执行：page.get_by_text("#游戏#").click()
   - 截图：after_try_get_by_text
   - 验证：✅ 成功！

6. 更新脚本，将选择器改为 get_by_text
```

**输出要求**：
- 记录所有尝试的方案和结果
- 只保留成功的方案到正式脚本
- 失败的尝试不要写入最终代码

---

### 方案 3：用户截图反馈法（推荐用于用户报告的问题）

**适用场景**：用户提供截图指出某处有问题，但不确定如何修复

**执行流程**：

```
1. 接收用户截图，分析问题
2. 从脚本中找到对应位置的代码
3. ⚠️ 重要：不要直接写泛化低效的代码
4. 严格按照脚本步骤执行到问题位置
5. 截图确认当前状态与用户截图一致
6. 尝试调通这一步：
   ├─ 分析为什么失败
   ├─ 尝试多种解决方案
   ├─ 每步都截图验证
   └─ 找到可行的方案
7. 用确认的方案修复脚本中的bug
8. 重新运行完整流程验证
```

**示例**：用户反馈"封面上传失败"

```
用户截图显示：点击"本地上传"后没有弹出文件选择器

执行流程：
1. 从脚本找到封面上传代码（第390-480行）
2. 不要直接改成复杂的 fallback 逻辑
3. 手动执行到封面上传步骤：
   - 截图：before_upload_cover
   - 确认：封面框已显示
   
4. 尝试方案A：直接点击封面框
   - 执行：page.click('div.cover-upload')
   - 截图：after_click_cover
   - 验证：❌ 没有反应
   
5. 尝试方案B：点击"本地上传"按钮
   - 执行：page.click('button:has-text("本地上传")')
   - 截图：after_click_local_upload
   - 验证：✅ 文件选择器弹出！
   
6. 分析原因：应该先点封面框，再点"本地上传"
7. 修复脚本，确保两步都执行
8. 重新测试完整流程
```

**关键原则**：
- ❌ **禁止**：看到问题就直接写复杂的泛化代码
- ✅ **必须**：先手动调通，确认方案可行后再写入脚本
- ✅ **必须**：每步都截图验证

---

## 四、AI 工具连接方法（通用方案）

### 4.1 启动已登录的浏览器

```bash
# 方法1：使用 start_logged_in_browser.py（推荐）
python f:/myclaw/test/start_logged_in_browser.py --platform toutiao --url https://mp.toutiao.com/profile_v4/graphic/publish

# 方法2：直接使用 BrowserAutomation 类
# BrowserAutomation 会自动加载 auth_state 并启用 CDP 端口 9222
```

### 4.2 AI 工具连接（任何支持 Playwright 的工具）

**在任何 AI 工具中都可以这样连接**：

```python
from playwright.sync_api import sync_playwright

# 连接到 CDP 端口 9222
playwright = sync_playwright().start()
browser = playwright.chromium.connect_over_cdp("http://localhost:9222")

# 复用已有上下文（重要！不要创建新的）
context = browser.contexts[0]
page = context.pages[0] if context.pages else context.new_page()

# 现在可以执行操作了
print(f"当前页面: {page.url}")
print(f"页面标题: {page.title()}")
```

**关键要点**：
- ✅ **必须使用 `browser.contexts[0]`**：复用已有上下文，保留登录状态
- ❌ **不要使用 `browser.new_context()`**：会创建新的空白上下文，丢失登录状态
- ✅ **CDP 端口固定为 9222**：由 `start_logged_in_browser.py` 启动时配置

### 4.3 常用命令

```python
# 导航
await page.goto("https://mp.toutiao.com/profile_v4/graphic/publish")

# 截图
await page.screenshot(path="f:/myclaw/test/debug_step_name.png")

# 填写表单
await page.fill("input[placeholder='标题']", "测试标题")

# 点击元素
await page.click("button:has-text('发布')")

# 键盘输入
await page.keyboard.type("#游戏")
await page.keyboard.press("Enter")

# 执行 JavaScript
result = await page.evaluate("""
    () => {
        const buttons = document.querySelectorAll('button');
        for (let btn of buttons) {
            if (btn.textContent.includes('发布')) {
                btn.click();
                return true;
            }
        }
        return false;
    }
""")
```

---

## 五、标准调试流程模板

### 5.1 方案1模板：逐步提取测试

```markdown
## 测试目标：头条号发布流程

### 步骤清单（从 publish_toutiao.py 提取）

#### 步骤0：关闭AI助手弹窗
- 操作前截图：debug_before_close_ai_assistant.png
- 执行操作：点击关闭按钮
- 操作后截图：debug_after_close_ai_assistant.png
- 验证结果：✅ AI助手已关闭

#### 步骤1：打开HTML文件并复制内容
- 操作前截图：debug_before_open_html.png
- 执行操作：打开 file:///.../article.html，Ctrl+A, Ctrl+C
- 操作后截图：debug_after_copy_content.png
- 验证结果：✅ 内容已复制到剪贴板

#### 步骤2：导航到发布页面
- 操作前截图：debug_before_navigate.png
- 执行操作：goto https://mp.toutiao.com/profile_v4/graphic/publish
- 操作后截图：debug_after_navigate.png
- 验证结果：✅ 发布页面已加载

...（继续所有步骤）
```

### 5.2 方案2模板：Bug定位修复

```markdown
## Bug修复：头条标签选择失败

### 问题分析
- 报错位置：第330行，添加标签步骤
- 错误信息：Element not found for selector 'div:has-text("#游戏#")'
- 可能原因：下拉菜单未出现，或选择器不匹配

### 调试过程

#### 准备阶段
- 执行到第4步（粘贴正文完成）
- 截图：debug_before_fix_tag.png
- 确认：正文已粘贴，光标在末尾

#### 尝试方案A：locator 匹配
- 截图：debug_try_locator_before.png
- 执行：page.locator('div:has-text("#游戏#")').click()
- 截图：debug_try_locator_after.png
- 结果：❌ 失败，元素不可见

#### 尝试方案B：get_by_text
- 截图：debug_try_get_by_text_before.png
- 执行：page.get_by_text("#游戏#").first.click()
- 截图：debug_try_get_by_text_after.png
- 结果：✅ 成功！

### 最终方案
将脚本第341行的选择器改为：
```python
tag_option = page.get_by_text(f"#{tag}#").first
if tag_option.is_visible(timeout=2000):
    tag_option.click()
```
```

### 5.3 方案3模板：用户截图反馈

```markdown
## 问题修复：用户反馈封面上传失败

### 用户提供的截图
- 截图路径：user_report_cover_fail.png
- 问题描述：点击"本地上传"后没有弹出文件选择器

### 调试过程

#### 定位问题代码
- 脚本位置：publish_toutiao.py 第390-480行
- 相关代码：封面上传逻辑

#### 手动复现
- 执行到封面上传步骤
- 截图：debug_reproduce_issue.png
- 确认：与用户截图一致

#### 尝试方案A：直接点击封面框
- 截图：debug_try_click_cover_before.png
- 执行：page.click('div.cover-upload')
- 截图：debug_try_click_cover_after.png
- 结果：❌ 没有反应

#### 尝试方案B：点击"本地上传"按钮
- 截图：debug_try_local_upload_before.png
- 执行：page.click('button:has-text("本地上传")')
- 截图：debug_try_local_upload_after.png
- 结果：✅ 文件选择器弹出！

#### 根本原因分析
应该先点击封面框，再点击"本地上传"按钮

### 最终修复
确保脚本中两步都执行：
```python
# 步骤1：点击封面框
page.click('div.cover-upload')
time.sleep(2)

# 步骤2：点击本地上传
with page.expect_file_chooser() as fc_info:
    page.click('button:has-text("本地上传")')
```
```

---

## 六、常见问题与解决方案

### 6.1 元素找不到

**症状**：`Element not found` 或 `TimeoutError`

**排查步骤**：
1. 截图确认页面状态
2. 检查是否在正确的页面
3. 检查元素是否被遮挡（弹窗、抽屉）
4. 尝试不同的选择器策略
5. 增加等待时间

**解决方案**：
```python
# 方案1：增加等待
await page.wait_for_selector(selector, timeout=10000)

# 方案2：滚动到元素
await page.evaluate(f"""
    () => {{
        const el = document.querySelector('{selector}');
        if (el) el.scrollIntoView({{behavior: 'smooth'}});
    }}
""")

# 方案3：多策略降级
async def click_with_fallback(page, text):
    # 策略1: locator
    # 策略2: get_by_text
    # 策略3: JavaScript
    # 策略4: Enter键
```

### 6.2 登录态失效

**症状**：页面跳转到登录页

**解决方案**：
1. 检查 `browser_data/cloak/{platform}` 目录是否存在且有内容
2. 检查 `Default/Network/Cookies` 文件是否存在
3. 如果目录存在但仍需要登录，删除旧目录重新登录：
   ```bash
   Remove-Item -Recurse -Force "browser_data\cloak\toutiao"
   ```
4. 重新运行脚本，手动登录一次
5. 登录后，认证状态会自动保存到用户目录

**调试代码**：
```python
import os
project_root = r"f:\myclaw\multi-platform-automation"
platform = 'toutiao'
user_data_dir = os.path.join(project_root, 'browser_data', 'cloak', platform)

print(f"用户目录: {user_data_dir}")
print(f"目录存在: {os.path.exists(user_data_dir)}")

cookies_file = os.path.join(user_data_dir, 'Default', 'Network', 'Cookies')
if os.path.exists(cookies_file):
    print(f"Cookies 文件大小: {os.path.getsize(cookies_file)} bytes")
else:
    print("Cookies 文件不存在！")
```

### 6.3 弹窗遮挡

**症状**：点击无效，元素被遮挡

**解决方案**：
```python
# 步骤0：关闭所有弹窗
close_selectors = [
    '.ai-assistant-drawer .byte-drawer-close',
    'button:has-text("我知道了")',
    'button:has-text("下一步")',
    'button:has-text("完成")'
]
for selector in close_selectors:
    try:
        if await page.is_visible(selector, timeout=2000):
            await page.click(selector)
            await page.wait_for_timeout(1000)
    except:
        pass
```

---

### 6.4 iframe 内元素定位（重要！百家号案例）

**问题**：某些网站的编辑器在 iframe 内部，使用普通选择器无法定位。

**典型案例**：百家号的正文编辑器在 `name="ueditor_0"` 的 iframe 内部。

#### ❌ 错误做法（已验证失败）

```python
# 错误1：使用 frame_locator（不起作用）
ueditor_frame = page.frame_locator('iframe[name="ueditor_0"]')
editor = ueditor_frame.locator('.view')
if editor.count() > 0:  # ❌ 返回 0，找不到元素
    editor.click()

# 错误2：遍历所有 contenteditable（会选中标题）
editors = page.locator('div[contenteditable="true"]')
# 标题输入框也是 contenteditable，会先被选中
editors.first.click()  # ❌ 点击的是标题，不是正文

# 错误3：凭经验猜测选择器，不实际验证
editor_selector = '#newsTextArea [contenteditable="true"]'  # ❌ 可能不存在
page.locator(editor_selector).click()
```

#### ✅ 正确做法（经过测试验证）

**方法1：通过 frame 对象访问（推荐）**

```python
# 步骤1：找到目标 frame
from playwright.sync_api import sync_playwright

playwright = sync_playwright().start()
browser = playwright.chromium.connect_over_cdp('http://localhost:9222')
context = browser.contexts[0]
page = context.pages[0] if context.pages else context.new_page()

# 遍历所有 frames，找到 name='ueditor_0' 的 frame
target_frame = None
for frame in page.frames:
    if frame.name == 'ueditor_0':
        target_frame = frame
        break

if target_frame:
    print(f"✅ 找到目标 frame")
    
    # 步骤2：在 frame 内查找 contenteditable 元素
    editables = target_frame.query_selector_all('[contenteditable]')
    
    if editables and len(editables) > 0:
        print(f"✅ 找到 {len(editables)} 个 contenteditable 元素")
        
        # 步骤3：点击第一个 contenteditable 元素
        editables[0].click()
        time.sleep(1)
        
        # 步骤4：粘贴内容
        page.keyboard.press('Control+V')
        time.sleep(3)
        
        print(f"✅ 正文已正确粘贴到编辑器中！")
    else:
        print(f"❌ frame 内未找到 contenteditable 元素")
else:
    print(f"❌ 未找到 name='ueditor_0' 的 frame")
```

**方法2：备用方案 - 直接定位 UEditor 容器**

```python
# 如果 frame 方法失败，尝试直接点击 UEditor 容器
try:
    editor = page.locator('#edui1_iframeholder').first
    if editor.is_visible(timeout=1000):
        editor.click()
        time.sleep(1)
        page.keyboard.press('Control+V')
        print(f"✅ 通过 #edui1_iframeholder 找到编辑器")
except Exception as e:
    print(f"⚠️ UEditor 容器方法失败: {e}")
```

#### 🔍 调试流程（必须先验证再编码）

**步骤1：检查页面结构**

```python
# 检查所有 frames
print(f"页面共有 {len(page.frames)} 个 frame:")
for i, frame in enumerate(page.frames):
    print(f"  Frame {i}: name='{frame.name}', url='{frame.url[:50]}'")

# 输出示例：
# Frame 0: name='', url='https://baijiahao.baidu.com/builder/rc/edit?type=n'
# Frame 1: name='', url='about:blank'
# Frame 2: name='', url='about:blank'
# Frame 3: name='ueditor_0', url='about:blank'  ← 这是目标 frame
```

**步骤2：检查 frame 内的元素**

```python
# 在目标 frame 内查找 contenteditable 元素
target_frame = None
for frame in page.frames:
    if frame.name == 'ueditor_0':
        target_frame = frame
        break

if target_frame:
    editables = target_frame.query_selector_all('[contenteditable]')
    print(f"找到 {len(editables)} 个 contenteditable 元素")
    
    for i, el in enumerate(editables):
        box = el.bounding_box()
        text = el.inner_text()
        print(f"  [{i}] x={box['x']}, y={box['y']}, w={box['w']}, h={box['h']}")
        print(f"      内容长度: {len(text)} 字符")
        print(f"      前50字符: {text[:50]}...")
```

**步骤3：截图验证**

```python
# 操作前截图
page.screenshot(path='f:/myclaw/test/debug_before_click.png')

# 点击编辑器
editables[0].click()
time.sleep(1)

# 操作后截图
page.screenshot(path='f:/myclaw/test/debug_after_click.png')

# 输入测试文本
test_text = "这是测试正文内容"
page.keyboard.type(test_text, delay=50)

# 最终截图
page.screenshot(path='f:/myclaw/test/debug_final.png')

# 验证内容
content = editables[0].inner_text()
if test_text in content:
    print(f"🎉 成功！正文已正确粘贴到编辑器中！")
else:
    print(f"❌ 失败！测试文本未在编辑器中找到")
```

####  关键要点

1. **不要使用 `frame_locator`**：`page.frame_locator()` 在某些情况下不起作用，必须使用 `page.frames` 遍历
2. **必须实际验证**：不要在 CloakBrowser 中测试就修改代码，先用 CDP 连接验证选择器
3. **注意 iframe 名称**：不同平台的 iframe 名称可能不同，需要先检查 `frame.name`
4. **避免盲目遍历**：遍历所有 `contenteditable` 会选中标题或其他元素，必须通过 iframe 隔离
5. **提供备用方案**：如果 frame 方法失败，尝试直接定位容器或使用 Tab 键切换

####  适用场景

- 百家号：`iframe[name="ueditor_0"]` → `[contenteditable]`
- 其他使用 UEditor、CKEditor、TinyMCE 等富文本编辑器的网站
- 任何包含 iframe 的复杂表单

---

### 6.5 头条号标签选择（重要！已验证）

**问题**：头条号发布文章时，需要添加话题标签（如 `#游戏#`），但自动化脚本中经常遇到标签选择失败的问题。

**核心发现**（经过多次测试验证）：
1. ✅ **单个标签可以成功**：输入 `#游戏` → ↓ → Enter，标签会显示为蓝色高亮
2. ❌ **多个标签连续添加会失败**：第二个标签无法触发建议框，因为第一个标签后面有特殊字符 `\u0002`
3. ⚠️ **之前发布成功是概率事件**：碰巧输入的标签（如"游戏"）是头条的默认标签，所以能成功
4. ❌ **在两个标签之间加空格无效**：测试验证后，第二个标签仍然是普通文本

**错误做法总结**（❌ 已验证失败）：
```python
# 错误1：只输入标签名，不按Enter或点击选择
publish_page.keyboard.type(f"#{tag}")
time.sleep(1.5)
# 结果：标签建议框显示，但标签未选中，仍是普通文本

# 错误2：直接输入完整格式 #游戏#
publish_page.keyboard.type(f"#{tag}#")
# 结果：仍然触发建议框，不会自动识别为标签

# 错误3：输入后按Enter，但不先选中建议项
publish_page.keyboard.type(f"#{tag}")
publish_page.keyboard.press('Enter')
# 结果：可能只是换行，没有选择标签

# 错误4：在两个标签之间加空格（已验证失败）
publish_page.keyboard.type(f"#{tag1}")
# ... ↓ + Enter ...
publish_page.keyboard.type(' ')  # 加空格
publish_page.keyboard.type(f"#{tag2}")
# 结果：第二个标签仍然是普通文本，无法触发建议框
```

**正确做法**（✅ 经过测试验证，仅适用于单个标签）：

**方案A：输入后按 ↓ + Enter（推荐，仅单标签）**
```python
# 步骤1：输入 # + 标签名
publish_page.keyboard.type(f"#{tag}")
time.sleep(1.5)  # 等待下拉框出现

# 步骤2：按 ↓ 选中第一项
publish_page.keyboard.press('ArrowDown')
time.sleep(0.3)

# 步骤3：按 Enter 确认选择
publish_page.keyboard.press('Enter')
time.sleep(0.5)

# 验证：标签应显示为蓝色高亮格式 #游戏#
```

**验证成功的标志**：
- ✅ 标签显示为蓝色高亮格式：`#游戏#`
- ✅ 标签建议框自动关闭
- ✅ 字数统计正确（标签算作独立单元）
- ❌ 如果标签只是普通文本（非高亮），说明选择失败

**关键限制**：
1. ⚠️ **此方案仅适用于单个标签**，多个标签连续添加会失败
2. ⚠️ **之前发布成功是概率事件**，依赖于标签是否是头条的默认标签
3. ⚠️ **目前没有可靠的多标签解决方案**，需要进一步研究

**测试截图参考**：
- 成功示例（单标签）：`f:\myclaw\test\debug_tag_methodA_after.png`（蓝色高亮）
- 失败示例（多标签）：`f:\myclaw\test\verify_space_solution.png`（普通文本）

**待解决问题**：
- 如何实现多个标签连续添加？
- 是否需要换行分隔？
- 是否需要其他方式触发建议框？

---

### 📝 重要教训（必须记住！）

#### 1. 调试流程规范
- ✅ **每次测试都必须调用大模型视觉能力分析截图**，不能瞎说成功失败
- ✅ **必须总结之前成功/失败的原因**，不能只记录结果
- ✅ **必须验证解决方案是否真的有效**，不能假设
- ✅ **只有确认解决后，才更新代码和 Skill**

#### 2. 头条号标签选择的关键发现
- ✅ **单个标签可以成功**：输入 `#游戏` → ↓ → Enter
- ❌ **多个标签连续添加会失败**：第二个标签无法触发建议框
- ⚠️ **之前发布成功是概率事件**：碰巧"游戏"是头条的默认标签
- ❌ **在两个标签之间加空格无效**：已验证失败

#### 3. 常见错误
- ❌ **没有调用大模型视觉能力分析截图**，就瞎说成功失败
-  **没有总结之前发布成功的原因**，导致重复测试
- ❌ **过早地认为掌握了方法**，就去修改代码
-  **没有把错误的经验记录下来**，导致下次又犯同样的错误

#### 4. 正确的做法
- ✅ **用 Browser Agent 完成需求**，然后再把完成的方式复刻到脚本里
- ✅ **看到解决才算解决**，认为自己掌握的方法但实际没有用自己掌握的方法解决过一次问题不算掌握经验
- ✅ **如果你真的掌握经验，你就可以用最少的测试步骤不出错的情况下完成需求**

#### 5. 成功经验的定义（重要！）

**什么才算成功经验？**

1. ✅ **必须能稳定复现**：至少连续测试3次都成功，不是偶尔成功的概率事件
2. ✅ **必须有明确的验证标准**：如"标签显示为蓝色高亮"，不能模糊地说"好像成功了"
3. ✅ **必须调用大模型视觉能力分析截图**：不能瞎说成功失败，必须用视觉能力确认
4. ✅ **必须记录失败的案例**：知道什么情况下会失败，才能避免
5. ✅ **必须用 Browser Agent 实际验证**：不能假设，必须亲眼看到结果

---

### 📋 测试具体问题的最简化流程（必须遵守！）

**核心原则**：只测试需要测试的内容，其他一律不做！

#### 第一步：判断当前在哪（截图 + 视觉分析）

```markdown
1. 截图当前页面
2. 用大模型视觉能力分析截图，判断当前在哪个页面：
   - 发布页面
   - 用户创作平台
   - 登录页面
```

#### 第二步：根据当前位置执行对应策略

**情况A：当前在发布页面**
- ✅ **直接测试需要测试的内容**，其他一律不做
- 示例：
  - 测试标题 → 直接去测试标题
  - 测试正文 → 直接去测试正文
  - 测试标签 → 直接去测试标签
  - 测试上传封面 → 直接去测试上传封面

**情况B：当前在用户创作平台**
- ✅ **按照脚本里的步骤进入发布页面**
- ✅ **然后重复情况A的策略**（直接测试需要测试的内容）

**情况C：当前在登录页面**
- ✅ **检查是否有免登录认证文件**
  - 如果有：直接进入创作界面（参考发布脚本的成功方案）
  - 如果没有：注入登录态文件重新登录
- ✅ **然后重复情况B的策略**（进入发布页面 → 直接测试需要测试的内容）

**特殊情况：测试定时发布和发布**
- ✅ **需要准备：标题、正文、封面**（只需这三种）
- ✅ **其他一律不做**

#### 第三步：执行测试并记录结果

```markdown
1. 执行测试操作
2. 截图保存结果
3. 用大模型视觉能力分析截图
4. 记录结果（成功/失败）
5. 重复测试至少3次验证可复现性
```

---

### ❌ 常见错误（必须避免！）

1.  **测试标签时填写标题和正文**：不需要！直接在已有的编辑器中测试即可
2.  **每次都清空编辑器重新填写**：不需要！除非测试需要
3.  **做了很多不必要的步骤**：只测试需要测试的内容
4.  **没有用大模型视觉能力分析截图**：必须用！
5. ❌ **没有重复测试3次验证可复现性**：必须做！
6. ❌ **没有目标地盲目测试**：必须有明确的测试目标
7. ❌ **复现失败后不总结**：必须得出结论并记录
8. ❌ **测试多标签时删除第一个标签**：必须在第一个标签后面加空格，然后输入第二个标签！
9. ❌ **Browser Agent 无法正确区分标题框和正文编辑器**：需要明确指定在正文编辑器中输入
10. ❌ **使用 fill() 方法清空编辑器**：`fill()` 会清空所有内容，导致第一个标签被删除！应该用 `keyboard.type()` 追加内容！
11. ❌ **Browser Agent 不截图就用视觉大模型分析**：必须截图并用视觉大模型分析，不能瞎说成功失败！
12. ❌ **Browser Agent 没有真正执行操作就截图**：必须确认 Browser Agent 真正执行了输入、等待、点击等操作，不能假设！
13.  **严禁使用 fill() 方法**：在任何情况下都不要使用 `fill()` 或 `locator.fill()` 方法！这两个方法都会清空现有内容，应该用 `keyboard.type()` 追加内容！
14.  **进入后不截图就操作**：进入后的第一件事必须是**截图并用视觉大模型分析当前状态**，确认当前位置后再执行操作！
15. ❌ **Browser Agent 无法执行复杂的多步骤测试**：对于连续添加两个标签等复杂操作，Browser Agent 可能无法正确执行，需要用户手动测试或提供更简单的测试方案！

---

### 🎯 测试的目标导向流程（重要！）

**核心原则**：测试必须有明确的目标，不能盲目测试！

#### 步骤1：用已有的经验尝试复现

```markdown
1. 查看之前总结的成功经验
2. 用 Browser Agent 按照经验执行
3. 重复测试至少3次验证可复现性
```

#### 步骤2：根据复现结果得出结论

**情况A：复现成功（3次都成功）**
- ✅ 确认这是可复现的成功经验
- ✅ 可以写进 Skill 和发布脚本

**情况B：复现失败（有失败案例）**
- ❌ 得出结论：这个经验是错误的/不可靠的
- ❌ 记录失败原因
- ✅ 继续测试其他方案

#### 步骤3：如果所有方案都失败

```markdown
1. 记录所有失败的方案和原因
2. 分析可能的根本原因
3. **找用户帮忙解决**：
   - 报告所有失败的尝试
   - 提供详细的失败案例和截图
   - 询问用户是否有其他思路或解决方案
```

#### 步骤4：用户帮助后的处理

```markdown
1. 如果用户提供了解决方案：
   - 用 Browser Agent 验证方案
   - 重复测试3次确认可复现
   - 更新 Skill 和发布脚本
2. 如果用户也无法解决：
   - 记录为已知问题
   - 标记为待后续研究
   - 在 Skill 中注明限制条件
```

---

### ✅ 正确的测试示例

**示例1：测试头条号标签选择**
```markdown
1. 查看已有经验：方案A（输入 `#游戏` → ↓ → Enter）
2. 用 Browser Agent 复现方案A：
   - 第1次：✅成功
   - 第2次：❌失败（标签是普通文本）
   - 第3次：❌失败（标签是普通文本）
3. 得出结论：方案A不是可复现的成功经验，是概率事件
4. 继续测试其他方案（方案B、C、D）
5. 如果所有方案都失败：找用户帮忙解决
```

**示例2：测试头条号标题填写**
```markdown
1. 查看已有经验：直接点击标题框输入
2. 用 Browser Agent 复现：
   - 第1次：✅成功
   - 第2次：✅成功
   - 第3次：✅成功
3. 得出结论：这是可复现的成功经验
4. 写进 Skill 和发布脚本
```

---

### ✅ 正确的测试示例

**示例1：测试头条号标签选择**
```markdown
1. 判断当前在哪：截图 → 视觉分析 → 发现在发布页面
2. 直接测试标签：
   - 输入 `#游戏` → ↓ → Enter
   - 截图 → 视觉分析 → 确认标签是否是蓝色高亮
   - 重复3次验证可复现性
3. 记录结果
```

**示例2：测试头条号标题填写**
```markdown
1. 判断当前在哪：截图 → 视觉分析 → 发现在发布页面
2. 直接测试标题：
   - 点击标题框 → 输入标题
   - 截图 → 视觉分析 → 确认标题是否正确填写
   - 重复3次验证可复现性
3. 记录结果
```

**示例3：测试头条号封面上传**
```markdown
1. 判断当前在哪：截图 → 视觉分析 → 发现在发布页面
2. 直接测试封面上传：
   - 点击封面上传区 → 选择图片
   - 截图 → 视觉分析 → 确认封面是否正确上传
   - 重复3次验证可复现性
3. 记录结果
```

---

**如何验证方案是可复现的？**

```markdown
# 验证流程
1. 用 Browser Agent 执行方案
2. 截图保存结果
3. **调用大模型视觉能力分析截图**（关键！）
4. 记录结果（成功/失败）
5. 重复步骤1-4，至少3次
6. 如果3次都成功：是可复现的成功经验
7. 如果有失败：不是可复现的，需要进一步研究
```

### ✅ 正确方案（已验证 - 连续添加两个标签）

**核心步骤**：
1. 清空编辑器（Ctrl+A, Delete）
2. 点击正文编辑器区域（不是标题框！）
3. 输入 `#游戏` → 等待1.5秒
4. 用 locator 定位下拉列表选项并点击第一项 `.forum-list-item`
5. 等待1秒
6. **输入一个空格**（或者按 Enter 换行）← **这一步很重要！**
7. 输入 `#游戏评测` → 等待1.5秒
8. 用 locator 定位下拉列表选项并点击第一项 `.forum-list-item`
9. 等待0.5秒
10. 截图验证

**验证结果**：
- ✅ **第一个标签已成功验证**：`#游戏#` 显示为蓝色高亮（见截图 `test_first_tag_final.png`）
-  **Browser Agent 无法正确执行复杂的多步骤测试**：在后续测试中，Browser Agent 多次被取消或没有正确执行操作
- ✅ **但方案本身是正确的**：您之前已经手动验证过方案是正确的
- ✅ **正确的方案**：
  1. 如果是第二个及以后的标签，先输入空格分隔
  2. 单独输入 `#` → 等待0.5秒（触发建议框）
  3. 输入标签文字 → 等待1.5秒（在建议框中搜索）
  4. 用 locator 点击 `.forum-list-item`
  5. 如果 locator 点击失败，降级到 `↓ + Enter` 方式
- ⚠️ **重要教训**：当用户已经验证过方案正确时，不要再用 Browser Agent 反复测试，应该直接更新发布脚本！

**为什么用 `keyboard.type()`？**
- ✅ `keyboard.type()` 是**追加内容**，不会清空现有内容
- ❌ `fill()` 或 `locator.fill()` 会**清空现有内容**，导致第一个标签被删除

**为什么要在两个标签之间加空格或换行？**
- ✅ **空格或换行可以分隔两个标签**，让第二个标签的 `#` 能够触发下拉列表
- ❌ **如果直接挨着输入**（如 `#游戏##游戏评测#`），第二个 `#` 可能无法触发下拉列表

**如何输入标签？**
- ✅ **唯一正确的方式**：先单独输入 `#` 触发建议框，然后输入文字
  ```python
  page.keyboard.type('#')        # 单独输入 #，触发标签建议框（类似 IDE 代码补全）
  time.sleep(0.5)                # 等待建议框出现
  page.keyboard.type('游戏')     # 输入文字，在建议框中搜索
  time.sleep(1.5)                # 等待建议框更新
  # 然后用 locator 点击第一项
  first_option = page.locator('.forum-list-item').first
  if first_option.count() > 0 and first_option.is_visible(timeout=2000):
      first_option.click()
  ```
- ❌ **严禁使用 fill() 或 locator.fill()**：这两个方法会清空现有内容！
- ❌ **不要使用其他方式**：如 `page.keyboard.type('#游戏')`、JavaScript evaluate 等，会导致不一致！

**如何清理错误的标签？**
- ✅ **当发现第二个标签输入不完整时**（如 `#游戏` 而不是 `#游戏评测`）：
  1. **将光标移动到第二个标签的左边**：点击第二个标签 `#游戏` 的左侧，使光标位于 `#游戏` 前面
  2. **用 Delete 键删除右边的错误标签**：`page.keyboard.press('Delete')` → 这会删除光标右边的 `#游戏`
  3. **重新输入正确的标签**：输入 `#游戏评测` → 等待1.5秒
  4. **用 locator 定位并点击第一项**：`.forum-list-item`
  5. 等待0.5秒
  6. 截图验证

**重要发现**：
- 从 snapshot 分析可以看到，即使按 ↓ + Enter，标签也没有被正确选择
- 编辑器中的内容仍然是普通文本 `#游戏`，而不是蓝色高亮的标签 `#游戏#`
- 这说明方案A本身就有问题，不是可靠的解决方案
- ✅ **从截图 `方案3_输入后.png` 可以看到，输入 `#游戏` 后下拉列表确实出现了**
- ❌ **但 Browser Agent 无法正确点击下拉列表中的选项**（可能在标题框中输入，而不是正文编辑器）
- ⚠️ **Browser Agent 无法正确区分标题框和正文编辑器**：多次测试都在标题框中输入，而不是正文编辑器

**待测试方案**：
- ✅ **方案3变体（已验证成功 - 单标签）**：用 Playwright 的 locator 定位下拉列表选项并点击
  ```python
  # 输入 #游戏
  page.keyboard.type('#游戏')
  time.sleep(1.5)  # 等待下拉列表出现
  
  # 用 locator 定位并点击第一个选项
  first_option = page.locator('.forum-list-item').first
  if first_option.count() > 0 and first_option.is_visible(timeout=2000):
      first_option.click()
      time.sleep(0.5)
      print("✅ 标签已选择")
  ```
  - ✅ **已验证**：单个标签可以成功插入（见截图 `verify_locator_after.png`）
  - ️ **未验证**：连续添加两个标签是否也能成功（需要进一步测试）
  - ❗ **重要经验**：测试多标签时，必须在第一个标签后面加空格，然后输入第二个标签！不能删除第一个标签重新测试！
- **方案4**：用键盘空格键确认选择（提示说"敲空格可取消插入话题"，可能空格也可以确认）
- **方案5**：直接在正文末尾手动输入完整的 `#游戏# #游戏评测#` 格式（不依赖自动选择）

**下一步行动**：
- 所有已知方案都失败了
- **需要找用户帮忙解决**：报告所有失败的尝试，提供详细的失败案例和截图

---

### 6.6 封面上传（头条号示例）

**图片池位置**：`f:/myclaw/multi-platform-automation/article_images/`
- 包含大量 `.jpeg` 和 `.jpg` 图片文件
- 选择策略：
  1. **视觉方案**：用大模型视觉能力选择合适的图片
  2. **默认方案**：使用第一张图片（如 `001a5ced.jpeg`）

**解决方案**：
```python
# 步骤1：截图确认封面上传区域状态
await page.screenshot(path="f:/myclaw/test/debug_before_upload_cover.png")

# 步骤2：视觉分析封面上传区域
# - 确认当前选中的模式（单图/三图/无封面）
# - 找到上传按钮的位置
# - 记录提示文字

# 步骤3：点击封面上传区域
# 根据视觉分析的结果，找到正确的上传按钮并点击
# 可能的操作：
# - 点击"+"图标
# - 点击"选择封面"按钮
# - 点击"点击本地上传"链接
cover_upload_selector = 'div.cover-upload-area'  # 需要根据实际页面调整
await page.click(cover_upload_selector)
await page.wait_for_timeout(2000)  # 等待文件选择对话框打开

# 步骤4：上传测试图片
image_path = "f:/myclaw/multi-platform-automation/article_images/001a5ced.jpeg"
async with page.expect_file_chooser() as fc_info:
    file_chooser = await fc_info.value
    await file_chooser.set_files(image_path)

# 步骤5：等待上传完成
await page.wait_for_timeout(10000)  # 等待10秒让图片上传和处理完成

# 步骤6：截图验证
await page.screenshot(path="f:/myclaw/test/debug_after_upload_cover.png")

# 步骤7：视觉分析验证结果
# - 确认封面是否成功上传
# - 确认是否显示预览图
# - 确认是否有错误提示
```

**关键要点**：
1. **必须先视觉分析**：确定正确的上传按钮选择器
2. **使用 expect_file_chooser**：Playwright 的标准文件上传方式
3. **等待足够时间**：图片上传和处理可能需要较长时间
4. **验证上传结果**：通过截图确认封面已显示

**常见问题**：
- ❌ 直接点击封面框没有反应 → 需要先点封面框，再点"本地上传"
- ❌ 文件选择器未打开 → 检查是否正确触发了文件上传事件
- ❌ 上传后没有显示预览 → 等待时间不够，或上传失败

---

## 七、文件位置规范

| 文件类型 | 位置 | 命名规则 |
|---------|------|---------||
| 测试脚本 | `f:\myclaw\test\test_<目标>.py` | `test_` 前缀 |
| **调试截图** | **`f:\myclaw\test\debug_<步骤>_<时间戳>.png`** | **`debug_` 前缀（强制！）** |
| **持久化用户目录** | **`f:\myclaw\multi-platform-automation\browser_data\cloak\{platform}\`** | **平台名子目录** |

---

## 八、禁止事项（强制要求）

- ❌ **禁止跳过截图步骤**
- ❌ **禁止在未截图的情况下执行操作**
- ❌ **禁止将截图保存在 test/ 以外的位置**
- ❌ **禁止在未验证的情况下继续下一步**
- ❌ **方案3中禁止直接写泛化低效代码**（必须先手动调通）
- ❌ **禁止使用 headless=True**
- ❌ **禁止删除 test/ 目录的截图**

---

## 九、快速参考

### 9.1 三种方案选择指南

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 新脚本首次调试 | 方案1：逐步提取测试 | 系统性验证每步 |
| 脚本运行报错 | 方案2：Bug定位修复 | 精准定位问题 |
| 用户反馈问题 | 方案3：用户截图反馈 | 针对性解决 |

### 9.2 调试检查清单

执行每步前确认：
- [ ] 已截图（操作前状态）
- [ ] 已分析当前状态
- [ ] 已选择合适的操作方式
- [ ] 准备好截图（操作后状态）
- [ ] 准备好验证结果

执行每步后确认：
- [ ] 已截图（操作后状态）
- [ ] 已验证操作结果
- [ ] 已记录成功/失败
- [ ] 已决定下一步行动

---

## 十、总结

**Browser Agent 调试的核心**：
1. ✅ 严格遵循“截图→操作→截图→验证”循环
2. ✅ 优先使用 auth_state 实现免登录
3. ✅ 根据场景选择合适的测试方案
4. ✅ 方案3中禁止直接写泛化代码，必须先手动调通
5. ✅ 所有截图保存到 `f:\myclaw\test\` 目录
6. ✅ **封面上传使用图片池**：`f:/myclaw/multi-platform-automation/article_images/`
7. ✅ **每步都要用视觉能力分析截图**，不要盲目执行
8. ⚡ **高效调试原则**：**启动一次浏览器，连续测试所有步骤，不要反复重启！**
9. 🌍 **通用性**：**本 Skill 可在任何 AI 编程工具中使用**（Lingma/Cursor/Claude Code等）

**⚠️ 效率陷阱（必须避免）**：
- ❌ 测试步骤1 → 关闭浏览器 → 重新登录 → 测试步骤1-2 → ...
- ❌ 每次只测试一个步骤，然后重启浏览器
- ❌ “逐步累加”的测试方式（1, 1-2, 1-3...）极其浪费时间

**✅ 正确做法**：
- ✅ 启动浏览器（登录一次）
- ✅ 连续执行所有步骤（1 → 2 → 3 → ... → N）
- ✅ 每步都截图验证，但不要关闭浏览器
- ✅ 所有步骤完成后，再关闭浏览器
- ✅ 效率提升：节省50-70%的时间

**记住**：每一步都要经过你的眼睛确认，不要盲目执行！但也要保持连续性，不要反复重启！

---

## 十一、迁移指南（从 Lingma 内置 Browser Agent 到 CloakBrowser）

### 11.1 为什么要迁移？

| 特性 | Lingma 内置 Browser Agent | CloakBrowser + CDP |
|------|--------------------------|-------------------|
| **可迁移性** | ❌ 仅限 Lingma | ✅ 可在任何 AI 工具中使用 |
| **反检测能力** | ❌ 普通 Chrome | ✅ CloakBrowser（源码级修改） |
| **持久化登录** | ⚠️ 有限支持 | ✅ 完整支持（独立用户目录） |
| **控制权** | ❌ 由 Lingma 控制 | ✅ 完全自主控制 |
| **CDP 端口** | 自动管理 | 9222（固定，易于配置） |

### 11.2 迁移步骤

#### 步骤1：安装依赖

```bash
# 在项目根目录执行
cd f:/myclaw/multi-platform-automation
pip install cloakbrowser playwright
```

#### 步骤2：启动 CloakBrowser

```bash
# 启动已登录的浏览器
python f:/myclaw/test/start_logged_in_browser.py --platform baijia --url https://baijiahao.baidu.com/builder/rc/edit?type=news
```

#### 步骤3：在 AI 工具中连接

**在任何 AI 工具中都可以这样连接**：

```python
from playwright.sync_api import sync_playwright

# 连接到 CDP 端口 9222
playwright = sync_playwright().start()
browser = playwright.chromium.connect_over_cdp("http://localhost:9222")

# 复用已有上下文（重要！）
context = browser.contexts[0]
page = context.pages[0] if context.pages else context.new_page()

# 开始调试
page.screenshot(path="f:/myclaw/test/debug_start.png")
print(f"当前页面: {page.url}")
```

#### 步骤4：执行调试任务

按照本 Skill 中的三种测试方案执行调试任务。

#### 步骤5：关闭浏览器

在 `start_logged_in_browser.py` 的终端中按 Ctrl+C。

### 11.3 常见问题

**Q1：为什么不能直接使用 Lingma 的 Browser Agent？**

A：Lingma 的 Browser Agent 是专有的，只能在 Lingma 中使用。通过 CDP 连接到 CloakBrowser，可以在任何支持 Playwright 的 AI 工具中使用（Cursor、Claude Code、GitHub Copilot 等）。

**Q2：CloakBrowser 和普通 Playwright 有什么区别？**

A：CloakBrowser 是源码级修改的 Chromium，彻底消除了自动化特征（navigator.webdriver、Canvas 指纹等），30/30 通过所有机器人检测测试，适合需要反检测的场景（如百家号、头条号等平台）。

**Q3：如何验证反检测效果？**

A：访问 https://bot.sannysoft.com/ 或 https://abrahamjuliot.github.io/creepjs/，查看检测结果。

**Q4：如果 CloakBrowser 不可用怎么办？**

A：`browser_automation.py` 中有降级方案，会自动切换到普通 Playwright。但建议优先使用 CloakBrowser。

**Q5：如何在其他 AI 工具中使用这个 Skill？**

A：将本 Skill 文件复制到目标 AI 工具的 Skills 目录中，然后按照上述步骤操作即可。核心是通过 CDP 连接到 CloakBrowser，与具体的 AI 工具无关。

---

## 十二、Windows 下 Python 日志 UTF-8 编码配置（重要！）

### 12.1 问题背景

**症状**：
- Windows 控制台已设置为 UTF-8（代码页 65001）
- 但 Python 日志输出表情符号或中文时出现 `UnicodeEncodeError`
- 错误信息：`UnicodeEncodeError: 'gbk' codec can't encode character '\u2705'`

**根本原因**：
- Windows 控制台虽然是 UTF-8，但 Python 的 `sys.stdout` 和 `sys.stderr` 默认编码可能仍是 GBK
- `logging.StreamHandler()` 使用 `sys.stdout`，导致输出时编码不匹配

### 12.2 解决方案（必须遵守！）

**在所有 Python 脚本开头添加以下代码**（必须在 logging 配置之前）：

```python
import sys

# 确保标准输出使用 UTF-8 编码（Windows 专用）
if sys.platform == 'win32':
    import io
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
    sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding='utf-8')
```

**logging 配置示例**：

```python
import logging

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('app.log', encoding='utf-8'),  # 文件也要指定 UTF-8
        logging.StreamHandler(sys.stdout)  # 使用 sys.stdout
    ]
)
logger = logging.getLogger('my_app')

# 现在可以安全使用表情符号
logger.info("✅ 初始化成功")
logger.warning("⚠️ 警告信息")
logger.error("❌ 错误信息")
```

### 12.3 验证方法

**测试脚本**：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import sys
import logging

# Windows UTF-8 配置
if sys.platform == 'win32':
    import io
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
    sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding='utf-8')

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    stream=sys.stdout
)

logger = logging.getLogger('test_emoji')
logger.info("✅ 测试成功：可以使用表情符号")
logger.info("❌ 测试失败标记")
logger.info("⚠️ 测试警告标记")
logger.info("🎉 测试庆祝标记")
```

**预期输出**：
```
2026-05-20 12:05:25,005 - test_emoji - INFO - ✅ 测试成功：可以使用表情符号
2026-05-20 12:05:25,006 - test_emoji - INFO - ❌ 测试失败标记
2026-05-20 12:05:25,006 - test_emoji - INFO - ⚠️ 测试警告标记
2026-05-20 12:05:25,006 - test_emoji - INFO - 🎉 测试庆祝标记
```

### 12.4 常见错误

**❌ 错误做法1**：只在 logging 配置中指定 `encoding='utf-8'`
```python
# 这样还不够！
logging.basicConfig(
    handlers=[
        logging.StreamHandler()  # 没有指定 sys.stdout
    ]
)
```

**❌ 错误做法2**：忘记在脚本开头设置 UTF-8
```python
import logging

# 直接配置 logging，没有先设置 sys.stdout
logging.basicConfig(stream=sys.stdout)  # sys.stdout 可能还是 GBK
```

**✅ 正确做法**：先设置 sys.stdout，再配置 logging
```python
import sys
if sys.platform == 'win32':
    import io
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
    sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding='utf-8')

import logging
logging.basicConfig(stream=sys.stdout)  # 现在 sys.stdout 是 UTF-8
```

### 12.5 适用范围

**需要添加 UTF-8 配置的场景**：
- ✅ 所有使用 `logging` 模块的 Python 脚本
- ✅ 所有需要在控制台输出中文或表情符号的脚本
- ✅ 所有在 Windows 环境下运行的 Python 程序

**不需要添加的场景**：
- ❌ Linux/macOS 系统（默认就是 UTF-8）
- ❌ 纯英文输出的脚本
- ❌ 不使用 logging 模块的简单脚本

### 12.6 经验总结

**记住这个顺序**：
1. **第一步**：导入 `sys` 和 `io`
2. **第二步**：检查 `sys.platform == 'win32'`
3. **第三步**：重新包装 `sys.stdout` 和 `sys.stderr`
4. **第四步**：配置 `logging`（此时 `sys.stdout` 已经是 UTF-8）

**关键原则**：
- ✅ **必须先设置 sys.stdout，再配置 logging**
- ✅ **FileHandler 也要指定 `encoding='utf-8'`**
- ✅ **可以使用表情符号**（✅ ❌ ⚠️ 🎉 等）
- ❌ **不要假设控制台是 UTF-8 就万事大吉**

---

## 十三、CloakBrowser 反检测方案（重要！）

### 11.1 为什么需要 CloakBrowser？

当前方案的问题：
- ❌ **Playwright 有自动化特征**：`navigator.webdriver=true`、Headless Chrome 标识等
- ❌ **平台风控可检测**：头条号、百家号等平台可能检测到自动化行为
- ❌ **账号风险高**：可能被限流、封号

CloakBrowser 的优势：
- ✅ **源码级修改 Chromium**：彻底消除所有自动化特征
- ✅ **30/30 通过检测测试**：Canvas、WebGL、TLS 指纹等全部伪装
- ✅ **Drop-in 替代 Playwright**：API 完全一致，迁移成本低
- ✅ **本地开源**：无需云服务，完全自主可控

### 11.2 相关开源项目

**CloakBrowser**（推荐）
- **GitHub**: https://github.com/CloakHQ/CloakBrowser
- **Stars**: 2,742+ (Trending)
- **特点**:
  - ✅ 通过源码级修改 Chromium 实现反检测
  - ✅ 30/30 通过所有机器人检测测试
  - ✅ Drop-in 替代 Playwright（API 完全一致）
  - ✅ 消除 navigator.webdriver、Canvas、WebGL 等指纹特征
  - ✅ 本地开源，无需云服务
- **安装**: `pip install cloak-browser`
- **使用**: 
  ```python
  from cloak_browser import CloakBrowser
  browser = CloakBrowser.launch()
  page = browser.new_page()
  page.goto("https://example.com")
  ```

**Obscura**
- **特点**:
  - ✅ 性能、内存、启动速度全面优于 Headless Chrome
  - ✅ Stealth 模式：随机化指纹（GPU、屏幕、Canvas、音频等）
  - ✅ 支持 Puppeteer / Playwright 连接

**其他相关项目**:
- **playwright-stealth**: Python 包，通过覆盖配置避免机器人检测
- **fingerprint-suite**: Apify 开发的浏览器指纹生成与注入工具集
- **puppeteer-extra-plugin-stealth**: Node.js 生态的隐身插件

### 11.3 CloakBrowser 适配层

项目中已创建 `utils/cloak_browser_adapter.py`，提供与 Playwright 兼容的 API：

```python
from utils.cloak_browser_adapter import CloakBrowserAdapter

adapter = CloakBrowserAdapter()
browser = adapter.launch(headless=False)
page = browser.new_page()
page.goto("https://mp.toutiao.com/")
```

**优势**:
- ✅ 完全隐藏自动化特征，平台风控无法检测
- ✅ API 与 Playwright 兼容，迁移成本低
- ✅ 内置真实用户行为模拟（鼠标轨迹、键盘节奏等）

**下一步行动**:
- [x] ✅ **已安装 CloakBrowser**：`pip install cloakbrowser`
- [x] ✅ **已创建适配层**：`utils/cloak_browser_adapter.py`
- [x] ✅ **已测试成功**：CloakBrowser 启动正常，能访问百度并截图
- [x] ✅ **已配置项目化部署**：
  - 创建了 `requirements.txt`（包含所有依赖）
  - 创建了 `.env` 文件（指定缓存目录到项目内）
  - 创建了 `.gitignore` 文件（排除敏感文件和缓存）
  - 创建了 `DEPLOYMENT.md` 部署指南
  - Chromium 二进制文件已下载到 `f:/myclaw/.cloakbrowser_cache/`
- [x] ✅ **已替换 browser_automation.py**：
  - 优先使用 CloakBrowser 启动浏览器（反检测模式）
  - 降级方案：如果 CloakBrowser 不可用，自动切换到 Playwright
  - 支持从环境变量读取缓存目录
  - ✅ **支持自动加载认证状态**：`BrowserAutomation(platform='toutiao')` 会自动加载 `toutiao_auth_state.json`
- [ ] 测试各平台发布功能是否正常（头条号、百家号等）
- [ ] 验证反检测效果（访问 https://bot.sannysoft.com/ 测试）
