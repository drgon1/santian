---
name: "browser-provider-implementer"
description: "网页端大模型对接路由开发全流程。触发条件：浏览器网页模型对接、Playwright转发、内网穿透端口配置、IDE代理调试、接口通信异常修复。兼容豆包/千问/Kimi/DeepSeek等所有网页端模型。"
---

# 网页端大模型对接路由开发 Skill

> **设计原则**：弱化自主推理，强化固定流程。按本文档步骤执行即可，不自行变通。**测试脚本和截图一律保存到 test/ 目录。**

---

## 一、适用场景（触发条件）

以下任一关键词出现时，自动启用本 Skill：

| 触发关键词 | 典型需求 |
|-----------|---------|
| 浏览器网页模型对接 | "新增豆包/Kimi/xxx 网页版模型"、"对接新的浏览器大模型" |
| Playwright 转发 | "Playwright 自动化"、"浏览器代理转发"、"网页版 AI 转 API" |
| 内网穿透端口配置 | "ngrok 配置"、"公网暴露"、"内网穿透"、"隧道配置" |
| IDE 代理调试 | "Cursor 代理配置"、"IDE 接入 new-api"、"自定义 API 地址" |
| 接口通信异常修复 | "模型不回复"、"消息发不出去"、"流式输出中断"、"会话清理失败" |

**非触发场景**（不启用本 Skill）：
- 纯后端 API 模型对接（如 OpenAI/Claude API Key 直连）
- 前端 UI 开发
- 数据库操作

---

## 二、执行步骤（严格按顺序，不可跳步）

### 步骤 0：确认项目结构（必做）

```
f:\myclaw\lama-puppeteer\
├── lama_puppeteer\
│   └── providers\
│       ├── base.py          ← 抽象基类，所有 Provider 的接口定义
│       ├── deepseek.py      ← 参考模板（简单结构，355行）
│       ├── qwen.py          ← 参考模板（复杂结构，含登录，700行）
│       ├── ollama.py        ← HTTP 直连（非浏览器，不参考）
│       └── __init__.py      ← 注册新 Provider 的地方
├── api_utils\
│   ├── app.py               ← 服务启动入口，Provider 注册逻辑
│   ├── request_processor.py ← 请求处理，消息合并与工具调用转换
│   └── ngrok_tunnel.py      ← ngrok 隧道管理
├── lama_puppeteer\
│   ├── smart_router.py      ← 模型层级与降级路由
│   ├── pool.py              ← Provider 池管理（含 auth_state 自动保存/加载）
│   └── model_filter.py      ← 模型身份过滤器
├── auth_states\             ← 认证状态文件目录（免登录核心）
│   ├── deepseek_auth_state.json
│   ├── kimi_auth_state.json
│   └── qwen_auth_state.json
├── test\                    ← 所有测试脚本放这里
├── start_all.bat            ← 一键启动脚本
└── .env                     ← 环境变量配置
```

### 步骤 1：用 Playwright Codegen 采集页面选择器（必做）

```bash
# 命令（在终端执行）
playwright codegen <目标网站URL>
```

**必须采集的 5 类选择器**（在 Codegen 浏览器中手动操作，记录生成的代码）：

| 序号 | 操作 | 要记录的内容 |
|------|------|-------------|
| 1 | 找到输入框，输入"测试" | 输入框的 CSS 选择器（textarea 或 contenteditable div） |
| 2 | 点击发送按钮 / 按 Enter | 发送按钮的选择器，确认 Enter 键是否可用 |
| 3 | 等待 AI 回复出现 | AI 回复内容所在的 DOM 容器选择器 |
| 4 | 点击"新建对话"按钮 | 新建对话按钮的文本或选择器 |
| 5 | hover 会话 → 点"..."→ 点"删除"→ 确认 | 会话列表选择器、更多按钮选择器、删除按钮文本、确认按钮文本 |

**产出**：将采集到的选择器填入下面的模板：

```
输入框选择器:     ____________________
发送按钮选择器:   ____________________
AI回复选择器:     ____________________
新建对话按钮文本:  ____________________
会话列表选择器:   ____________________
更多按钮选择器:   ____________________
删除按钮文本:     ____________________
确认删除按钮文本:  ____________________
```

### 步骤 2：用浏览器 DevTools 验证选择器（必做）

在目标网站按 F12 打开 Console，逐条执行验证：

```javascript
// 验证 1：输入框存在且唯一
document.querySelector('你采集的输入框选择器')

// 验证 2：AI 回复容器存在
document.querySelectorAll('你采集的AI回复选择器')

// 验证 3：新建对话按钮存在
document.querySelector('你采集的新建对话选择器')

// 验证 4：会话列表项存在
document.querySelectorAll('你采集的会话列表选择器')
```

**验证标准**：每个选择器必须返回非 null 且非空 NodeList。不通过则回到步骤 1 重新采集。

### 步骤 3：复制模板文件（必做）

**规则**：不手写新文件。从现有 Provider 复制。

```bash
# 如果目标网站结构简单（无 OAuth 登录流程）
copy f:\myclaw\lama-puppeteer\lama_puppeteer\providers\deepseek.py f:\myclaw\lama-puppeteer\lama_puppeteer\providers\<新模型名>.py

# 如果目标网站有 OAuth 登录流程
copy f:\myclaw\lama-puppeteer\lama_puppeteer\providers\qwen.py f:\myclaw\lama-puppeteer\lama_puppeteer\providers\<新模型名>.py
```

### 步骤 4：修改模板文件（必做，按顺序改）

**4.1 修改文件顶部常量和选择器**（用步骤 1-2 采集的值替换）：

```python
# 必须修改的常量（替换为步骤1-2采集的值）
INPUT_TEXTAREA_SELECTORS = "你采集的输入框选择器, textarea, div[contenteditable='true']"
SEND_BUTTON = "你采集的发送按钮选择器"
STOP_BUTTON = "button[aria-label='Stop'], button[class*='stop']"
AI_RESPONSE_SELECTOR = "你采集的AI回复选择器"
NEW_CONVERSATION_TEXT = "你采集的新建对话按钮文本"
```

**4.2 修改类名和三个配置方法**：

```python
class XxxProvider(BaseProvider):  # 改类名
    def get_name(self) -> str:
        return "xxx"              # 改 provider 名称

    def get_default_model(self) -> str:
        return "xxx-web"          # 改默认模型 ID

    def get_url(self) -> str:
        return "https://xxx.com/chat"  # 改目标 URL
```

**4.3 修改 `_cleanup_old_conversations` 中的选择器**：

```python
conv_selector = "你采集的会话列表选择器"  # 必须改
# 下拉菜单选择器（根据实际页面调整）
dropdown_selector = ".dropdown-menu-item, .ant-dropdown-menu-item, [class*='dropdown']"
# 删除确认按钮
confirm_btn = page.get_by_role("button", name="你采集的确认删除按钮文本")
```

### 步骤 5：注册 Provider 到系统（必做）

**5.1 修改 `__init__.py`**：

文件：`f:\myclaw\lama-puppeteer\lama_puppeteer\providers\__init__.py`

```python
from .xxx import XxxProvider  # 新增 import
__all__ = ["BaseProvider", "DeepSeekProvider", "OllamaProvider", "QwenProvider", "XxxProvider"]  # 追加
```

**5.2 修改 `smart_router.py`**：

文件：`f:\myclaw\lama-puppeteer\lama_puppeteer\smart_router.py`

```python
MODEL_TIERS: Dict[str, int] = {
    "deepseek-web": 3,
    "qwen-web": 2,
    "xxx-web": 2,     # 新增，根据模型能力设置层级 1-3
}

TIER_FALLBACK_CHAINS: Dict[int, List[str]] = {
    3: ["deepseek-web"],
    2: ["qwen-web", "xxx-web"],  # 追加到对应层级
    1: [],
    0: [],
}
```

**5.3 修改 `app.py`**（服务启动注册 + 免登录配置）：

文件：`f:\myclaw\lama-puppeteer\api_utils\app.py`

在 `_register_providers` 函数中添加：

```python
from lama_puppeteer.providers.xxx import XxxProvider
import os

# 配置 auth_state 文件路径（免登录核心）
auth_state_dir = os.path.join(os.path.dirname(__file__), "..", "auth_states")
os.makedirs(auth_state_dir, exist_ok=True)
auth_state_file = os.path.join(auth_state_dir, "xxx_auth_state.json")

# 在注册代码区域添加
await pool.register(
    provider_factory=XxxProvider,
    model="xxx-web",
    auth_state_file=auth_state_file,  # ← 关键：启用免登录
)
```

**免登录工作原理**：
1. **首次启动**：auth_state_file 不存在 → 打开浏览器 → 用户手动登录 → 系统自动保存 cookies/session 到 auth_state.json
2. **后续启动**：auth_state_file 存在 → 直接加载 cookies/session → 跳过登录页面 → 已处于登录状态
3. **自动维护**：每次使用后自动更新 auth_state.json，保持登录态有效

### 步骤 6：编写测试脚本（必做）

在 `f:\myclaw\test\` 目录创建 `test_xxx.py`：

```python
import asyncio
import logging
import sys
import os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "lama-puppeteer"))

from playwright.async_api import async_playwright
from lama_puppeteer.providers.xxx import XxxProvider

logging.basicConfig(level=logging.DEBUG, format="%(asctime)s [%(name)s] %(levelname)s: %(message)s")
logger = logging.getLogger("test-xxx")

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        context = await browser.new_context(viewport={"width": 1280, "height": 800})
        page = await context.new_page()
        provider = XxxProvider(max_keep_conversations=1)

        # 测试 1: 导航
        logger.info("=== 测试 1: navigate ===")
        await provider.navigate(page)

        # 测试 2: 发送消息
        logger.info("=== 测试 2: submit_prompt ===")
        await provider.submit_prompt(page, "你好，1+1等于几？")

        # 测试 3: 流式提取
        logger.info("=== 测试 3: extract_streaming ===")
        async for chunk in provider.extract_streaming(page):
            print(chunk, end="", flush=True)
        print()

        # 测试 4: 完整回复
        logger.info("=== 测试 4: extract_full_response ===")
        await provider.submit_prompt(page, "请用一句话介绍你自己")
        full = await provider.extract_full_response(page)
        logger.info(f"完整回复: {full[:100]}...")

        # 测试 5: 新建对话 + 清理
        logger.info("=== 测试 5: reset_conversation ===")
        await provider.reset_conversation(page)

        input("\n按 Enter 关闭浏览器...")
        await browser.close()

if __name__ == "__main__":
    asyncio.run(main())
```

### 步骤 7：逐项测试验证（必做，按顺序）

每项通过才能进入下一项：

- [ ] **7.1 navigate**：浏览器打开目标网站，输入框可见，无报错
- [ ] **7.2 submit_prompt**：提示词成功填入并发送，AI 开始回复
- [ ] **7.3 extract_streaming**：能实时流式输出 AI 回复文本
- [ ] **7.4 extract_full_response**：能获取完整回复
- [ ] **7.5 is_generating**：生成中返回 True，结束后返回 False
- [ ] **7.6 reset_conversation**：能新建对话
- [ ] **7.7 _cleanup_old_conversations**：会话数超过 max_keep 时自动删除旧会话
- [ ] **7.8 连续 3 次请求**：不报错，每次回复正常

---

## 三、标准写法（代码模板，直接套用）

### 3.1 完整文件结构模板

```python
"""<模型名> web UI provider implementation.

Target: <目标URL>
"""

import asyncio
import logging
from typing import AsyncGenerator, Optional

from playwright.async_api import Page, TimeoutError as PlaywrightTimeout

from .base import BaseProvider

logger = logging.getLogger("lama-puppeteer")

# === CSS 选择器（从步骤1-2采集） ===
INPUT_TEXTAREA_SELECTORS = "<输入框选择器>, textarea, div[contenteditable='true']"
SEND_BUTTON = "<发送按钮选择器>"
STOP_BUTTON = "button[aria-label='Stop'], button[class*='stop']"
AI_RESPONSE_SELECTOR = "<AI回复选择器>"
NEW_CONVERSATION_TEXT = "<新建对话按钮文本>"


class XxxProvider(BaseProvider):
    def __init__(self, max_keep_conversations: int = 5):
        self._last_text = ""
        self._response_started = asyncio.Event()
        self._max_keep = max_keep_conversations

    # === 配置方法（必实现） ===
    def get_name(self) -> str:
        return "xxx"

    def get_default_model(self) -> str:
        return "xxx-web"

    def get_url(self) -> str:
        return "https://xxx.com/chat"

    # === 以下方法按模板实现，只改选择器引用 ===
```

### 3.2 `navigate()` — 标准模板

```python
async def navigate(self, page: Page) -> None:
    logger.info(f"Navigating to {self.get_url()}...")
    await page.goto(self.get_url(), wait_until="domcontentloaded", timeout=60000)

    # 登录检测（通用逻辑，不改）
    current_url = page.url
    if "/login" in current_url or "/sign_in" in current_url:
        logger.warning("需要登录，请在浏览器中手动登录...")
        while "/login" in page.url or "/sign_in" in page.url:
            await asyncio.sleep(2)
        logger.info("登录检测完成")

    # 等待输入框就绪
    try:
        await page.wait_for_selector(INPUT_TEXTAREA_SELECTORS, timeout=30000)
        logger.info("页面加载完成，输入框就绪。")
    except PlaywrightTimeout:
        await page.wait_for_selector("textarea, div[contenteditable='true']", timeout=30000)
```

### 3.3 `submit_prompt()` — 标准模板

```python
async def submit_prompt(self, page: Page, prompt: str) -> None:
    logger.info(f"Submitting prompt ({len(prompt)} chars)...")

    # 1. 找输入框
    input_el = None
    try:
        input_el = await page.wait_for_selector(INPUT_TEXTAREA_SELECTORS, timeout=10000)
    except PlaywrightTimeout:
        pass
    if not input_el:
        input_el = await page.wait_for_selector("textarea, div[contenteditable='true']", timeout=10000)
    if not input_el:
        raise RuntimeError("Could not find input box.")

    # 2. 清空并填入
    await input_el.evaluate("""(el) => {
        if (el.tagName === 'TEXTAREA' || el.tagName === 'INPUT') { el.value = ''; }
        else if (el.isContentEditable) { el.innerHTML = ''; }
    }""")

    if await input_el.get_attribute("contenteditable") == "true":
        await input_el.click()
        await page.keyboard.type(prompt, delay=10)
    else:
        await input_el.fill(prompt)

    await asyncio.sleep(0.5)

    # 3. 发送（优先 Enter 键）
    await page.keyboard.press("Enter")

    # 4. 重置状态
    self._last_text = ""
    self._response_started = asyncio.Event()
```

### 3.4 `_get_response_text()` — 标准模板

```python
async def _get_response_text(self, page: Page) -> str:
    elements = page.locator(AI_RESPONSE_SELECTOR)
    count = await elements.count()
    if count > 0:
        for i in range(count - 1, -1, -1):
            try:
                text = await elements.nth(i).inner_text()
                if text.strip():
                    return text.strip()
            except Exception:
                pass
    return ""
```

### 3.5 `_is_generating()` — 标准模板

```python
async def _is_generating(self, page: Page) -> bool:
    # 策略 1: 输入框 disabled（最可靠）
    try:
        ta = page.locator("textarea")
        if await ta.is_visible(timeout=500):
            if await ta.get_attribute("disabled") is not None:
                return True
    except Exception:
        pass
    # 策略 2: 停止按钮可见
    try:
        stop_btn = page.locator(STOP_BUTTON)
        if await stop_btn.is_visible(timeout=300):
            return True
    except Exception:
        pass
    return False
```

### 3.6 `extract_streaming()` — 标准模板（不改）

```python
async def extract_streaming(self, page: Page, poll_interval: float = 0.3) -> AsyncGenerator[str, None]:
    yield ""
    stable_count = 0
    while True:
        current_text = await self._get_response_text(page)
        if current_text and len(current_text) > len(self._last_text):
            delta = current_text[len(self._last_text):]
            self._last_text = current_text
            yield delta
            stable_count = 0
        else:
            still_generating = await self._is_generating(page)
            if not still_generating:
                stable_count += 1
                if stable_count >= 3:
                    break
        await asyncio.sleep(poll_interval)
```

### 3.7 `extract_full_response()` — 标准模板（不改）

```python
async def extract_full_response(self, page: Page, timeout: float = 120) -> str:
    try:
        await page.wait_for_selector(AI_RESPONSE_SELECTOR, timeout=int(timeout * 1000))
        last_text = ""
        stable_checks = 0
        for _ in range(200):
            await asyncio.sleep(1)
            current_text = await self._get_response_text(page)
            if current_text == last_text and current_text:
                stable_checks += 1
                if stable_checks >= 3:
                    break
            else:
                last_text = current_text
                stable_checks = 0
    except PlaywrightTimeout:
        logger.warning("Timeout waiting for response completion.")
    await asyncio.sleep(0.5)
    return await self._get_response_text(page)
```

### 3.8 `wait_for_response_start()` — 标准模板

```python
async def wait_for_response_start(self, page: Page) -> None:
    try:
        await page.wait_for_selector(AI_RESPONSE_SELECTOR, timeout=30000)
        self._response_started.set()
        logger.info("AI response container appeared.")
    except PlaywrightTimeout:
        logger.warning("Could not detect AI response container, proceeding anyway.")
        self._response_started.set()
```

### 3.9 `reset_conversation()` — 标准模板

```python
async def reset_conversation(self, page: Page) -> None:
    try:
        new_chat_btn = page.get_by_text(NEW_CONVERSATION_TEXT, exact=True).first
        if await new_chat_btn.count() > 0 and await new_chat_btn.is_visible():
            await new_chat_btn.click()
            await asyncio.sleep(0.8)
            logger.info("Started new conversation.")
    except Exception as e:
        logger.warning(f"Failed to start new conversation: {e}")
    await self._cleanup_old_conversations(page)
```

### 3.10 `_cleanup_old_conversations()` — 标准模板（关键！）

```python
async def _cleanup_old_conversations(self, page: Page) -> None:
    try:
        conv_selector = "<会话列表选择器>"  # 从步骤1采集

        # 【必做】等待会话列表渲染
        try:
            await page.wait_for_selector(conv_selector, timeout=10000)
        except Exception:
            logger.debug("No conversation list found, skipping cleanup")
            return

        total = await page.locator(conv_selector).count()
        if total <= self._max_keep:
            return

        to_delete = total - 1
        for i in range(to_delete):
            # 【必做】每次循环重新查询（DOM 会变）
            conv = page.locator(conv_selector).last

            # Step 1: 【必做】用真实鼠标移动触发 CSS :hover
            box = await conv.bounding_box()
            if not box:
                continue
            await page.mouse.move(box["x"] + box["width"] - 10, box["y"] + box["height"] // 2)
            await asyncio.sleep(1.0)

            # Step 2: JS 点击"..."按钮（多策略查找）
            more_clicked = await page.evaluate("""() => {
                const items = document.querySelectorAll('<会话列表选择器>');
                if (items.length === 0) return false;
                const last = items[items.length - 1];
                // 策略 A: 查找内部按钮
                const clickables = last.querySelectorAll('button, [role="button"], svg, [class*="more"], [class*="menu"]');
                for (const el of clickables) {
                    if (el.offsetParent !== null) { el.click(); return true; }
                }
                // 策略 B: 基于位置查找右侧按钮
                const rect = last.getBoundingClientRect();
                const allBtns = document.querySelectorAll('button, [role="button"]');
                for (const btn of allBtns) {
                    const br = btn.getBoundingClientRect();
                    if (br.top >= rect.top - 5 && br.bottom <= rect.bottom + 5 &&
                        br.left >= rect.right - 60 && br.left <= rect.right + 10 &&
                        btn.offsetParent !== null) {
                        btn.click(); return true;
                    }
                }
                return false;
            }""")
            if not more_clicked:
                continue

            # Step 3: 【必做】等待下拉菜单出现
            try:
                await page.wait_for_selector("<下拉菜单选择器>", timeout=3000)
            except Exception:
                await page.keyboard.press("Escape")
                continue
            await asyncio.sleep(0.3)

            # Step 4: JS 点击删除选项
            delete_clicked = await page.evaluate("""() => {
                const menus = document.querySelectorAll('<下拉菜单选择器>');
                for (const menu of menus) {
                    if ((menu.textContent || '').includes('删除')) {
                        menu.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
                        return true;
                    }
                }
                return false;
            }""")
            if not delete_clicked:
                await page.keyboard.press("Escape")
                continue
            await asyncio.sleep(0.5)

            # Step 5: 点击确认按钮
            confirm_btn = page.get_by_role("button", name="<确认删除按钮文本>")
            if await confirm_btn.count() > 0:
                await confirm_btn.first.click()
                await asyncio.sleep(1.5)
                logger.info(f"  Deleted conversation [{i+1}/{to_delete}]")
            else:
                await page.keyboard.press("Escape")

    except Exception as e:
        logger.warning(f"Conversation cleanup failed: {e}")
```

---

## 四、报错排查（按症状查表，直接执行修复动作）

### 症状 A：`wait_for_selector` 超时

```
PlaywrightTimeout: Timeout 30000ms exceeded while waiting for selector "xxx"
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | F12 打开目标网站，Console 执行 `document.querySelector('你的选择器')` |
| 2 | 如果返回 null → 选择器错误，回到步骤 1 重新采集 |
| 3 | 如果返回元素但测试仍超时 → 页面加载慢，增加 timeout 到 60000 |
| 4 | 如果页面需要登录 → 检查是否被重定向到登录页，先手动登录 |

### 症状 B：`fill()` 无效，文字没填进去

```
# 文字填入后输入框仍为空
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | Console 执行 `document.querySelector('你的选择器').tagName` |
| 2 | 如果是 `DIV` → 改用 `keyboard.type()` |
| 3 | Console 执行 `document.querySelector('你的选择器').isContentEditable` |
| 4 | 如果是 `true` → 改用 `keyboard.type()` |
| 5 | 如果是 `TEXTAREA` 但仍无效 → 先用 `evaluate` 清空 `el.value = ''` |

### 症状 C：发送后 AI 无响应

```
# 提示词发送了，但页面没有反应
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | 检查是否按了 Enter 键 → 改为点击发送按钮 |
| 2 | 检查发送按钮是否被禁用 → 等待按钮 enabled |
| 3 | 检查是否有弹窗遮挡 → 按 Escape 关闭弹窗 |
| 4 | 检查是否需要登录 → 查看 page.url 是否包含 login |

### 症状 D：流式提取卡住不结束

```
# extract_streaming 一直循环，不退出
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | Console 执行 `document.querySelector('textarea').getAttribute('disabled')` |
| 2 | 如果生成结束后 disabled 属性不消失 → 改用停止按钮检测 |
| 3 | Console 执行 `document.querySelector('button[class*="stop"]')` |
| 4 | 如果停止按钮在生成结束后仍存在 → 增加 stable_count 阈值到 5 |

### 症状 E：会话清理失败（最常见）

```
# 删除操作执行了但会话没被删掉
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | 检查是否用了 `dispatchEvent(new MouseEvent('mouseover'))` → **必须改用 `page.mouse.move()`** |
| 2 | 检查是否在 SVG 元素上直接 `.click()` → **必须改用 `dispatchEvent(MouseEvent)`** |
| 3 | 检查是否等待了下拉菜单出现 → **必须加 `wait_for_selector`** |
| 4 | 检查删除后是否重新查询了 DOM → **必须用 `page.locator(selector).last`** |
| 5 | 检查会话列表是否渲染完成 → **必须加 `wait_for_selector` 等待列表** |

### 症状 F：ngrok 隧道启动失败

```
# ngrok 启动后无公网 URL 或连接被拒绝
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | 检查 ngrok.exe 是否在项目根目录 → `dir f:\myclaw\lama-puppeteer\ngrok.exe` |
| 2 | 检查 authtoken 是否配置 → `ngrok config check` |
| 3 | 检查端口是否被占用 → `netstat -ano | findstr 9091` |
| 4 | 检查防火墙是否拦截 → 临时关闭防火墙测试 |
| 5 | 访问 `http://127.0.0.1:4040` 查看 ngrok 状态面板 |

### 症状 G：new-api 渠道不通

```
# new-api 测试对话报错 "channel not available"
```

| 排查步骤 | 操作 |
|---------|------|
| 1 | 检查 LamaPuppeteer 是否在运行 → `curl http://localhost:9091/health` |
| 2 | 检查渠道 URL 是否正确 → new-api 渠道配置中 base_url 应为 `http://localhost:9091` |
| 3 | 检查渠道 Key 是否匹配 → 渠道 Key 必须与 `.env` 中 `API_KEY` 一致 |
| 4 | 检查模型名称是否匹配 → 渠道模型列表必须包含请求的 model 名称 |

---

## 五、调试优先级（卡死时按此顺序排查）

```
优先级 1（最先查）：选择器是否正确
  └→ F12 Console 验证每个选择器

优先级 2：页面是否加载完成
  └→ 检查 page.url，检查是否有登录/弹窗/验证码

优先级 3：元素是否可见/可交互
  └→ 检查 offsetParent !== null，检查 disabled 属性

优先级 4：时序问题
  └→ 增加 asyncio.sleep()，增加 wait_for_selector()

优先级 5：事件触发方式
  └→ CSS :hover 用 page.mouse.move()，SVG 点击用 dispatchEvent

优先级 6（最后查）：网络/代理问题
  └→ 检查代理设置，检查 ngrok 隧道状态
```

### 卡死卡点优先解决方案

| 卡死场景 | 第一动作 | 第二动作 | 第三动作 |
|---------|---------|---------|---------|
| 页面打不开 | 检查 URL 是否正确 | 检查网络代理 | 手动浏览器访问测试 |
| 输入框找不到 | `wait_for_selector` 加 timeout | 用通用选择器 `textarea` | 截图看页面状态 |
| 消息发不出去 | 改 Enter 为点击按钮 | 检查按钮 disabled | 手动点击测试 |
| 回复提取为空 | 检查 AI_RESPONSE_SELECTOR | 用 `page.inner_text('body')` 看全文 | 截图看 DOM |
| 流式卡死 | 检查 `_is_generating` 逻辑 | 增加 stable_count | 加超时强制退出 |
| 清理失败 | 改 `page.mouse.move()` | 加 `wait_for_selector` | 简化：只保留最新1个 |
| ngrok 不通 | `ngrok config check` | 检查防火墙 | 重启 ngrok |

---

## 六、部署规范

### 6.1 文件位置规范

| 文件类型 | 位置 | 命名规则 |
|---------|------|---------||
| Provider 实现 | `lama_puppeteer/providers/xxx.py` | 模型名小写，如 `doubao.py` |
| 测试脚本 | `test/test_xxx.py` | `test_` 前缀 + 模型名 |
| **认证状态文件** | **`auth_states/xxx_auth_state.json`** | **模型名 + `_auth_state.json`（免登录核心）** |
| 环境变量 | `.env` | 统一在项目根目录的 `.env` 中配置 |

### 6.1.1 免登录配置详解（必读）

**核心机制**：Playwright `storage_state` 持久化

```python
# pool.py 自动处理（无需手动编写）
# 加载阶段
if os.path.exists(auth_state_file):
    context_options["storage_state"] = auth_state_file  # 加载登录态

# 保存阶段
await context.storage_state(path=auth_state_file)  # 保存登录态
```

**配置步骤**：

1. **在 `.env` 中添加配置**（可选，推荐）：
```env
XXX_AUTH_STATE_FILE=auth_states/xxx_auth_state.json
```

2. **首次启动流程**：
```bash
# 启动服务
cd f:\myclaw\lama-puppeteer
start_all.bat

# 观察日志输出
# 首次："Auth state file not found: auth_states/xxx_auth_state.json"
# 浏览器会自动打开目标网站
```

3. **手动完成登录**：
   - 在打开的浏览器中完成登录（账号密码/OAuth/扫码等）
   - 登录成功后，系统会自动保存 auth_state.json
   - 日志会显示："Saved auth state to auth_states/xxx_auth_state.json"

4. **验证免登录生效**：
```bash
# 重启服务
start_all.bat

# 观察日志
# 应看到："Loading auth state from auth_states/xxx_auth_state.json"
# 浏览器直接打开已登录状态的页面，无需再次登录
```

**安全注意事项**：
- ⚠️ `auth_states/*.json` 包含敏感认证信息（cookies/session tokens）
- ⚠️ **不要上传到公共 Git 仓库**（已在 `.gitignore` 中排除）
- ⚠️ **不要分享给他人**
- ✅ 可以备份到自己的私有存储

**常见问题**：

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 每次启动都要重新登录 | auth_state_file 路径错误 | 检查 `app.py` 中的路径配置 |
| auth_state.json 为空 | 登录未完成就关闭浏览器 | 确保登录成功后再关闭 |
| 登录后仍然提示需要登录 | cookies 过期或失效 | 删除旧的 auth_state.json，重新登录 |
| 多个模型共用一个 auth_state | 配置错误 | 每个模型使用独立的 auth_state 文件 |

### 6.2 注册清单（部署前逐项确认）

- [ ] `providers/__init__.py` 已添加 import 和 `__all__`
- [ ] `smart_router.py` 已添加 `MODEL_TIERS` 和 `TIER_FALLBACK_CHAINS`
- [ ] `api_utils/app.py` 已添加 Provider 注册代码
- [ ] `.env` 已添加新模型的限流配置（如 `XXX_DAILY_LIMIT=100`）
- [ ] `start_all.bat` 无需修改（Provider 通过 app.py 自动注册）
- [ ] new-api 管理面板已添加新模型到渠道模型列表

### 6.3 启动验证

```bash
# 1. 启动服务
cd f:\myclaw\lama-puppeteer
start_all.bat

# 2. 验证 Provider 注册成功（查看日志输出）
# 应看到: "Registering provider: xxx (model=xxx-web)"
# 应看到: "Provider xxx registered. Ready models: ['deepseek-chat', 'qwen-web', 'xxx-web']"

# 3. 验证 new-api 渠道
curl http://localhost:3000/api/channel/ -H "New-Api-User: 1"

# 4. 测试对话
curl -X POST http://localhost:9091/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-lamapuppeteer-admin-key-2026" \
  -d '{"model":"xxx-web","messages":[{"role":"user","content":"你好"}]}'
```

### 6.4 禁止事项

- ❌ 不要在 Provider 中硬编码敏感信息（密钥、密码放 `.env`）
- ❌ 不要修改 `base.py` 的接口定义
- ❌ 不要跳过步骤 7 的测试清单直接部署
- ❌ 不要在 `_cleanup_old_conversations` 中使用 `dispatchEvent` 触发 hover
- ❌ 不要使用 `page.wait_for_load_state("networkidle")` 等待页面加载（SPA 页面永不触发）
- ❌ 不要从零手写 Provider，必须从 `deepseek.py` 或 `qwen.py` 复制模板