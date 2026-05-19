# playwright-test-generator

> Playwright 测试脚本生成器。触发条件：需要快速创建浏览器自动化测试、验证发布流程、调试元素选择、生成调试代码。自动将截图保存到 test/ 目录。

---


# Playwright 测试脚本生成器 Skill

> **设计原则**：一键生成，自动配置，**截图一律保存到 test/ 目录**。按模板生成即可，不自行变通。

---

## 一、适用场景（触发条件）

以下任一关键词出现时，自动启用本 Skill：

| 触发关键词 | 典型需求 |
|-----------|---------|
| 需要快速创建浏览器自动化测试 | "帮我写个测试脚本"、"测试头条发布"、"验证选择器" |
| 验证发布流程 | "测试头条发布流程"、"百家号发布调试"、"大鱼号发布验证" |
| 调试元素选择 | "测试标签选择"、"验证按钮点击"、"测试下拉菜单" |
| 生成调试代码 | "写个 Playwright 脚本"、"生成浏览器测试代码" |

**非触发场景**（不启用本 Skill）：
- 纯后端 API 测试
- 数据库操作测试
- 单元测试编写

---

## 二、执行步骤（严格按顺序，不可跳步）

### 步骤 1：确认测试目标（必做）

```
明确要测试的内容：
- 测试目标：____________________（如：头条号标签选择、百家号内容粘贴）
- 目标URL：____________________（如：https://mp.toutiao.com/profile_v4/graphic/publish）
- 关键操作：____________________（如：输入标签、点击按钮、上传封面）
- 预期结果：____________________（如：标签成功添加到正文）
- **是否需要登录**：____________________（是/否，如需登录则配置 auth_state）
```

### 步骤 2：选择测试模板（必做）

根据测试目标选择合适的模板：

#### 模板A：简单元素验证（单步骤，含免登录支持）

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
测试：<测试目标>

触发词：<触发关键词>
"""

import asyncio
import time
from pathlib import Path
from playwright.async_api import async_playwright

# 强制保存到 test/ 目录
TEST_DIR = Path(r"f:\myclaw\test")
TEST_DIR.mkdir(parents=True, exist_ok=True)

async def debug_with_screenshot(page, step_name="unknown"):
    """快速截图并保存到 test/ 目录（强制要求！）"""
    timestamp = int(time.time())
    filename = TEST_DIR / f"debug_{step_name}_{timestamp}.png"
    await page.screenshot(path=str(filename), full_page=False)
    print(f"✅ 截图已保存: {filename}")
    return str(filename)

async def main():
    async with async_playwright() as p:
        # 1. 启动浏览器
        browser = await p.chromium.launch(
            headless=False,
            args=['--lang=zh-CN']
        )
        
        # 2. 【免登录配置】加载 auth_state（如果存在）
        AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\xxx_auth_state.json")
        context_options = {
            "locale": "zh-CN",
            "viewport": {"width": 1280, "height": 800}
        }
        if AUTH_STATE_FILE.exists():
            print(f"✅ 加载登录态: {AUTH_STATE_FILE}")
            context_options["storage_state"] = str(AUTH_STATE_FILE)
        else:
            print(f"⚠️ 未找到登录态文件，需要手动登录")
        
        context = await browser.new_context(**context_options)
        page = await context.new_page()
        
        # 3. 打开目标页面
        print(f"=== 打开目标页面 ===")
        await page.goto(
            "<目标URL>",
            wait_until="domcontentloaded",
            timeout=60000
        )
        await page.wait_for_timeout(3000)
        await debug_with_screenshot(page, "initial_page")
        
        # 4. 执行测试操作
        print(f"\n=== 执行测试操作 ===")
        try:
            # 在这里编写测试逻辑
            # 示例：点击按钮
            # await page.click('button:has-text("确认")')
            # await debug_with_screenshot(page, "after_click")
            
            print("  ✅ 测试操作执行成功")
        except Exception as e:
            print(f"  ❌ 测试操作失败: {e}")
            await debug_with_screenshot(page, "error")
        
        # 5. 【首次登录】如果需要登录且没有 auth_state，等待用户手动登录
        if not AUTH_STATE_FILE.exists():
            print("\n⚠️ 检测到需要登录，请在浏览器中完成登录...")
            print("   登录完成后，按 Enter 保存登录态并继续测试")
            input("按 Enter 继续...")
            
            # 保存登录态供下次使用
            try:
                await context.storage_state(path=str(AUTH_STATE_FILE))
                print(f"✅ 登录态已保存: {AUTH_STATE_FILE}")
                print("   下次运行将自动登录！")
            except Exception as e:
                print(f"⚠️ 保存登录态失败: {e}")
        
        # 6. 等待用户确认
        print(f"\n=== 测试完成 ===")
        input("按 Enter 关闭浏览器...")
        
        await browser.close()

if __name__ == "__main__":
    asyncio.run(main())
```

#### 模板B：多步骤流程测试

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
测试：<测试目标>（多步骤流程）

触发词：<触发关键词>
"""

import asyncio
import time
from pathlib import Path
from playwright.async_api import async_playwright

# 强制保存到 test/ 目录
TEST_DIR = Path(r"f:\myclaw\test")
TEST_DIR.mkdir(parents=True, exist_ok=True)

async def debug_with_screenshot(page, step_name="unknown"):
    """快速截图并保存到 test/ 目录"""
    timestamp = int(time.time())
    filename = TEST_DIR / f"debug_{step_name}_{timestamp}.png"
    await page.screenshot(path=str(filename), full_page=False)
    print(f"✅ 截图已保存: {filename}")
    return str(filename)

async def verify_element(page, selector, description="元素"):
    """验证元素是否存在且可见"""
    try:
        element = page.locator(selector).first
        count = await element.count()
        
        if count == 0:
            print(f"❌ {description} 不存在: {selector}")
            return False
        
        is_visible = await element.is_visible(timeout=2000)
        if not is_visible:
            print(f"❌ {description} 不可见: {selector}")
            return False
        
        print(f"✅ {description} 已找到: {selector}")
        return True
    except Exception as e:
        print(f"❌ {description} 验证失败: {e}")
        return False

async def click_with_fallback(page, target_text, description="元素"):
    """多策略点击元素（4层降级）"""
    
    # 策略1: locator
    try:
        locator = page.locator(f'div:has-text("{target_text}")').first
        if await locator.count() > 0 and await locator.is_visible(timeout=2000):
            await locator.click()
            print(f"✅ {description} 点击成功（locator）")
            return True
    except:
        pass
    
    # 策略2: get_by_text
    try:
        btn = page.get_by_text(target_text, exact=False).first
        if await btn.is_visible(timeout=2000):
            await btn.click()
            print(f"✅ {description} 点击成功（get_by_text）")
            return True
    except:
        pass
    
    # 策略3: JavaScript
    try:
        js_result = await page.evaluate(f"""
            () => {{
                const all = document.querySelectorAll('button, div, li, span');
                for (let el of all) {{
                    if (el.textContent.includes('{target_text}')) {{
                        el.click();
                        return true;
                    }}
                }}
                return false;
            }}
        """)
        if js_result:
            print(f"✅ {description} 点击成功（JS）")
            return True
    except:
        pass
    
    # 策略4: 回车
    try:
        await page.keyboard.press('Enter')
        print(f"✅ {description} 点击成功（回车）")
        return True
    except:
        pass
    
    print(f"❌ {description} 所有策略失败")
    return False

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=False,
            args=['--lang=zh-CN']
        )
        
        context = await browser.new_context(
            locale="zh-CN",
            viewport={"width": 1280, "height": 800}
        )
        page = await context.new_page()
        
        # 步骤配置
        steps = [
            {"name": "打开页面", "action": "goto", "url": "<目标URL>"},
            {"name": "填写标题", "action": "fill", "selector": "input", "value": "测试标题"},
            {"name": "点击按钮", "action": "click", "text": "确认"},
        ]
        
        # 执行步骤
        for i, step in enumerate(steps, 1):
            print(f"\n=== 步骤 {i}: {step['name']} ===")
            
            try:
                action = step['action']
                
                if action == "goto":
                    await page.goto(step['url'], wait_until="domcontentloaded", timeout=60000)
                    await page.wait_for_timeout(3000)
                    
                elif action == "fill":
                    await page.fill(step['selector'], step['value'])
                    print(f"  ✅ 已填写: {step['value']}")
                    
                elif action == "click":
                    if 'text' in step:
                        await click_with_fallback(page, step['text'], step['name'])
                    else:
                        await page.click(step['selector'])
                        print(f"  ✅ 已点击")
                
                # 每步后截图
                await debug_with_screenshot(page, f"after_step_{i}")
                await page.wait_for_timeout(1000)
                
            except Exception as e:
                print(f"  ❌ 步骤失败: {e}")
                await debug_with_screenshot(page, f"error_step_{i}")
                break
        
        print(f"\n=== 测试完成 ===")
        input("按 Enter 关闭浏览器...")
        
        await browser.close()

if __name__ == "__main__":
    asyncio.run(main())
```

### 步骤 3：生成测试脚本（必做）

**文件位置**：`f:\myclaw\test\test_<测试目标>.py`

**命名规则**：
```
test_toutiao_tag_selection.py          # 头条标签选择测试
test_baijia_content_paste.py           # 百家号内容粘贴测试
test_dayu_cover_upload.py              # 大鱼号封面上传测试
```

### 步骤 4：运行测试（必做）

```bash
# 命令
cd f:\myclaw
python test/test_<测试目标>.py
```

### 步骤 5：查看截图（必做）

**截图位置**：`f:\myclaw\test\debug_*.png`

**查看命令**：
```bash
# Windows 资源管理器打开 test 目录
explorer f:\myclaw\test
```

---

## 三、标准写法（代码模板，直接套用）

### 3.1 截图函数（强制要求）

```python
from pathlib import Path

# 强制定义 test/ 目录
TEST_DIR = Path(r"f:\myclaw\test")
TEST_DIR.mkdir(parents=True, exist_ok=True)

async def debug_with_screenshot(page, step_name="unknown"):
    """快速截图并保存到 test/ 目录（强制要求！）"""
    timestamp = int(time.time())
    filename = TEST_DIR / f"debug_{step_name}_{timestamp}.png"
    await page.screenshot(path=str(filename), full_page=False)
    print(f"✅ 截图已保存: {filename}")
    return str(filename)
```

**使用规则**：
- ✅ **在每个关键步骤后调用**
- ✅ **在错误捕获时调用**
- ❌ **禁止使用其他路径**

### 3.2 元素验证函数

```python
async def verify_element(page, selector, description="元素"):
    """验证元素是否存在且可见"""
    try:
        element = page.locator(selector).first
        count = await element.count()
        
        if count == 0:
            print(f"❌ {description} 不存在: {selector}")
            return False
        
        is_visible = await element.is_visible(timeout=2000)
        if not is_visible:
            print(f"❌ {description} 不可见: {selector}")
            return False
        
        print(f"✅ {description} 已找到: {selector}")
        return True
    except Exception as e:
        print(f"❌ {description} 验证失败: {e}")
        return False
```

### 3.3 多策略点击函数

```python
async def click_with_fallback(page, target_text, description="元素"):
    """多策略点击元素（4层降级）"""
    
    # 策略1: locator
    try:
        locator = page.locator(f'div:has-text("{target_text}")').first
        if await locator.count() > 0 and await locator.is_visible(timeout=2000):
            await locator.click()
            print(f"✅ {description} 点击成功（locator）")
            return True
    except:
        pass
    
    # 策略2: get_by_text
    try:
        btn = page.get_by_text(target_text, exact=False).first
        if await btn.is_visible(timeout=2000):
            await btn.click()
            print(f"✅ {description} 点击成功（get_by_text）")
            return True
    except:
        pass
    
    # 策略3: JavaScript
    try:
        js_result = await page.evaluate(f"""
            () => {{
                const all = document.querySelectorAll('button, div, li, span');
                for (let el of all) {{
                    if (el.textContent.includes('{target_text}')) {{
                        el.click();
                        return true;
                    }}
                }}
                return false;
            }}
        """)
        if js_result:
            print(f"✅ {description} 点击成功（JS）")
            return True
    except:
        pass
    
    # 策略4: 回车
    try:
        await page.keyboard.press('Enter')
        print(f"✅ {description} 点击成功（回车）")
        return True
    except:
        pass
    
    print(f"❌ {description} 所有策略失败")
    return False
```

---

## 八、常见测试场景模板

### 4.1 测试标签选择

```python
# 测试步骤
steps = [
    {"name": "打开发布页面", "action": "goto", "url": "https://mp.toutiao.com/profile_v4/graphic/publish"},
    {"name": "输入标签", "action": "keyboard", "keys": "#游戏"},
    {"name": "等待下拉菜单", "action": "wait", "timeout": 2000},
    {"name": "截图验证", "action": "screenshot", "name": "after_input_tag"},
    {"name": "点击标签选项", "action": "click_text", "text": "#游戏#"},
    {"name": "截图验证", "action": "screenshot", "name": "after_select_tag"},
]
```

### 4.2 测试内容粘贴

```python
# 测试步骤
steps = [
    {"name": "打开发布页面", "action": "goto", "url": "https://mp.toutiao.com/profile_v4/graphic/publish"},
    {"name": "点击编辑器", "action": "click", "selector": "div[contenteditable='true']"},
    {"name": "粘贴内容", "action": "keyboard", "keys": "Control+V"},
    {"name": "截图验证", "action": "screenshot", "name": "after_paste"},
]
```

### 4.3 测试按钮点击

```python
# 测试步骤
steps = [
    {"name": "打开页面", "action": "goto", "url": "目标URL"},
    {"name": "点击确认按钮", "action": "click_text", "text": "确认发布"},
    {"name": "截图验证", "action": "screenshot", "name": "after_click_confirm"},
    {"name": "等待弹窗", "action": "wait", "timeout": 3000},
    {"name": "截图验证", "action": "screenshot", "name": "after_popup"},
]
```

---

## 九、文件位置规范（强制要求）

| 文件类型 | 位置 | 命名规则 |
|---------|------|---------|
| 测试脚本 | `f:\myclaw\test\test_<目标>.py` | `test_` 前缀 + 测试目标 |
| **调试截图** | **`f:\myclaw\test\debug_<步骤>_<时间戳>.png`** | **`debug_` 前缀 + 步骤名 + 时间戳（强制！）** |
| 验证结果 | `f:\myclaw\test\result_<目标>.txt` | `result_` 前缀 + 测试目标 |

### 截图命名示例

```
f:\myclaw\test\debug_initial_page_1768900000.png
f:\myclaw\test\debug_after_input_tag_1768900005.png
f:\myclaw\test\debug_after_click_tag_1768900010.png
f:\myclaw\test\debug_error_1768900015.png
```

---

## 六、免登录配置（调试发布脚本必备）

### 6.1 为什么需要免登录？

调试发布脚本时，每次测试都要重新登录会极大降低效率。通过 `auth_state` 持久化，可以实现：
- ✅ 首次手动登录一次
- ✅ 后续测试自动登录
- ✅ 保持会话状态，更接近真实场景

### 6.2 配置步骤

**步骤 1：确定目标平台的 auth_state 文件路径**

```python
# 头条号
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\toutiao_auth_state.json")

# 百家号
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\baijia_auth_state.json")

# 大鱼号
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\dayu_auth_state.json")

# 微信公众号
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\wechat_auth_state.json")

# 小红书
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\xiaohongshu_auth_state.json")
```

**步骤 2：在测试脚本中添加免登录代码**

```python
# 在创建 context 之前添加
AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\xxx_auth_state.json")
context_options = {
    "locale": "zh-CN",
    "viewport": {"width": 1280, "height": 800}
}

if AUTH_STATE_FILE.exists():
    print(f"✅ 加载登录态: {AUTH_STATE_FILE}")
    context_options["storage_state"] = str(AUTH_STATE_FILE)
else:
    print(f"⚠️ 未找到登录态文件，需要手动登录")

context = await browser.new_context(**context_options)
```

**步骤 3：首次运行时保存登录态**

```python
# 在测试完成后，如果之前没有 auth_state，提示用户登录并保存
if not AUTH_STATE_FILE.exists():
    print("\n⚠️ 检测到需要登录，请在浏览器中完成登录...")
    input("登录完成后，按 Enter 保存登录态...")
    
    # 保存登录态
    await context.storage_state(path=str(AUTH_STATE_FILE))
    print(f"✅ 登录态已保存: {AUTH_STATE_FILE}")
```

### 6.3 完整示例（头条号调试）

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""测试：头条号标签选择功能"""

import asyncio
from pathlib import Path
from playwright.async_api import async_playwright

TEST_DIR = Path(r"f:\myclaw\test")
TEST_DIR.mkdir(parents=True, exist_ok=True)

async def debug_with_screenshot(page, step_name="unknown"):
    timestamp = int(asyncio.get_event_loop().time() * 1000)
    filename = TEST_DIR / f"debug_{step_name}_{timestamp}.png"
    await page.screenshot(path=str(filename))
    print(f"✅ 截图: {filename.name}")

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        
        # 免登录配置
        AUTH_STATE_FILE = Path(r"f:\myclaw\lama-puppeteer\auth_states\toutiao_auth_state.json")
        context_options = {"locale": "zh-CN", "viewport": {"width": 1280, "height": 800}}
        
        if AUTH_STATE_FILE.exists():
            context_options["storage_state"] = str(AUTH_STATE_FILE)
            print("✅ 已加载头条号登录态")
        
        context = await browser.new_context(**context_options)
        page = await context.new_page()
        
        # 导航到发布页面
        await page.goto("https://mp.toutiao.com/profile_v4/graphic/publish")
        await page.wait_for_timeout(3000)
        await debug_with_screenshot(page, "publish_page")
        
        # 测试标签选择
        await page.keyboard.type("#游戏")
        await page.wait_for_timeout(2000)
        await debug_with_screenshot(page, "after_input_tag")
        
        # 如果是首次运行，保存登录态
        if not AUTH_STATE_FILE.exists():
            print("\n如需保存登录态，请按 Enter...")
            input()
            await context.storage_state(path=str(AUTH_STATE_FILE))
            print("✅ 登录态已保存")
        
        input("\n按 Enter 关闭...")
        await browser.close()

if __name__ == "__main__":
    asyncio.run(main())
```

### 6.4 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 每次都要重新登录 | auth_state 文件不存在或路径错误 | 检查文件路径是否正确 |
| 登录后仍然提示未登录 | cookies 过期 | 删除旧的 auth_state.json，重新登录 |
| auth_state.json 很大（几百KB） | 包含大量缓存数据 | 正常现象，不影响使用 |
| 多个平台共用一个 auth_state | 配置错误 | 每个平台使用独立的 auth_state 文件 |

---

## 七、快速启动命令

```bash
# 1. 生成测试脚本
# （AI 自动生成到 f:\myclaw\test\test_<目标>.py）

# 2. 运行测试
cd f:\myclaw
python test/test_<目标>.py

# 3. 查看截图
explorer f:\myclaw\test

# 4. 清理旧截图（可选）
cd f:\myclaw\test
del debug_*.png
```

---

## 十、禁止事项（强制要求）

- ❌ **禁止将截图保存在 `test/` 目录以外的任何位置**
- ❌ 禁止使用 headless=True（必须 False 才能截图）
- ❌ 禁止跳过截图步骤
- ❌ 禁止使用硬编码的 sleep（用 wait_for_timeout）
- ❌ 禁止在失败时不截图
- ❌ 禁止删除 test/ 目录的截图
-  **禁止使用相对路径保存截图**（必须使用绝对路径）

---

## 十一、使用示例

### 示例1：快速测试头条标签选择

**用户请求**：
> 帮我测试头条号标签选择功能，看看 #游戏# 标签能不能选中

**AI 执行**：
1. 生成 `f:\myclaw\test\test_toutiao_tag_selection.py`
2. 自动包含截图函数（保存到 test/ 目录）
3. 运行测试：`python test/test_toutiao_tag_selection.py`
4. 查看截图：`f:\myclaw\test\debug_after_input_tag_*.png`

### 示例2：验证百家号发布流程

**用户请求**：
> 测试百家号发布流程，重点验证内容粘贴和封面上传

**AI 执行**：
1. 生成 `f:\myclaw\test\test_baijia_publish_flow.py`
2. 多步骤测试，每步后自动截图
3. 运行测试并查看截图结果

---

## 十二、核心原则

1. **截图一律保存到 `f:\myclaw\test\` 目录**（强制！）
2. **使用绝对路径，禁止相对路径**
3. **每个关键步骤后必须截图**
4. **失败时必须截图保留现场**
5. **测试脚本命名规范：`test_<目标>.py`**
6. **截图命名规范：`debug_<步骤>_<时间戳>.png`**


---

## How to Use

### Option 1: View directly on GitHub
Open this document directly in your browser to read the full content.

### Option 2: Clone to local
`ash
git clone https://github.com/drgon1/santian.git
cd santian/ai-skills/.lingma/skills/playwright-test-generator
`

### Option 3: Import to Lingma IDE
Copy the entire $skillName folder to your .lingma/skills/ directory.

---

*This document was automatically generated by [generate_github_docs.ps1](../generate_github_docs.ps1)*