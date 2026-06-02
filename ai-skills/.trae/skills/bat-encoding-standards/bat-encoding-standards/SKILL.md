---
name: "bat-encoding-standards"
description: "BAT 脚本标准化生成技能。AI 在编写、修改、生成 .bat 文件前必须主动调用本 Skill 加载规范。确保 UTF-8 编码、首行切 65001、子 bat 禁止重复 chcp、必须模拟双击自测。"
---

# BAT 脚本标准化生成技能

## 触发条件（关键字的强制响应）

### 关键字自检列表

AI 在收到用户消息或任务后，**必须首先检查**消息中是否包含以下任一关键字：

| 关键字 | 触发动作 |
|--------|---------|
| `.bat` | 立即调用本 Skill |
| `bat` 且上下文涉及批处理 | 立即调用本 Skill |
| `批处理` | 立即调用本 Skill |
| `脚本` 且上下文涉及 Windows 命令 | 立即调用本 Skill |
| `cmd` 且上下文涉及文件执行 | 立即调用本 Skill |

### 强制调用规则

1. **命中任意关键字** → 必须先调用本 Skill，再回答或动手
2. **没有例外**——即使你觉得"这只是个小改动"、"只是改一行"、"先回答再处理"，也必须先调 Skill
3. **反向触发器**：即使你的思考过程中**自己生成了** `.bat` 操作方案（而非用户提出），也要立即触发
4. **本技能不会自动启用**——依赖上述关键字检查机制和 auto-experience-trigger 的同源场景识别

## 核心背景（必须理解）

存在**开发环境与用户本地环境编码差异**，这是所有 BAT 编码 Bug 的根源：

| 环境 | 终端编码 | 说明 |
|------|---------|------|
| Trae IDE 内嵌终端 | **UTF-8（65001）** | AI 开发测试的环境 |
| 用户本地 CMD（双击） | **GBK（936）** | 用户实际运行的环境 |
| SearchReplace 工具写入 | **UTF-8** | 文件保存方式固定 |

**关键推论：**
- AI 在 IDE 终端测试通过 ≠ 用户双击运行正常
- AI 无法在 IDE 中复现用户本地乱码问题
- 必须以「用户双击运行」为最终验证标准

## 核心规则

### 0. 最高优先级：禁止中文（强制执行）
- **`.bat` 文件中禁止出现任何中文字符**（包括注释、echo 输出、title、提示信息）
- 所有文本使用纯 ASCII/英文
- 原因：
  - Write 工具保存文件使用 LF 换行（非 CRLF），CMD 解析含中文的 LF 换行 bat 文件时必乱码
  - Write + Python 生成器（CRLF）可以解决，但流程复杂易出错
  - **纯英文 = 编码问题归零**，UTF-8 和 GBK 下英文字符编码一致
- 任何情况下不得在 `.bat` 文件中写入中文，包括：
  - `echo 中文`
  - `title 中文`
  - `set /p "=中文"`
  - `REM 中文注释`
  - 所有位置的文字

### 1. 固定编码：UTF-8
- 所有 `.bat` 文件保持 **UTF-8 编码**（无 BOM）
- 禁止转换为 GBK/ANSI 格式（避免 SearchReplace 工具重新保存为 UTF-8 导致编码错乱）
- 文件中的中文直接书写（UTF-8 原生支持中文，IDE 显示正常）

### 2. 入口文件首行编码切换（强制）
- **入口批处理文件**（用户双击执行的 .bat）首行必须固定如下：

```bat
@echo off
chcp 65001 >nul 2>&1
```

- 禁止简化省略 `2>&1`，禁止写成 `>nul` 不带错误重定向
- **必须放在第二行**（`@echo off` 之后、任何其他命令之前）

### 3. 子 bat 禁止重复 chcp
- 被入口文件 `call` 或 `start` 调用的**子 bat 文件**，禁止包含 `chcp` 命令
- 编码切换只执行一次，子 bat 继承入口的 UTF-8 环境
- 违反此规则会导致编码回退冲突，引发中文乱码

### 4. 被调用的 Python 脚本添加编码声明
- 被 `.bat` 调用的 `.py` 文件，文件头部必须固定添加：

```python
# -*- coding: utf-8 -*-
```

- 确保 Python 脚本在处理中文输出时不会产生编码异常

### 5. emoji 处理
- UTF-8 编码原生支持 emoji，可以正常使用
- 但如果目标环境是 `cmd.exe` 较旧版本，emoji 可能显示为方框
- 推荐使用 ASCII 替代方案确保兼容：
  - `[ok]` 替代 ✅
  - `[x]` 替代 ❌
  - `[!]` 替代 ⚠️
  - `[info]` 替代 📋
  - `[path]` 替代 📍
  - `[py]` 替代 🐍

## 强制自测要求（核心约束）

### 禁止仅在 IDE 内嵌终端完成自测

必须 **模拟用户在本地直接双击 bat 文件** 的行为，在原生独立 CMD 窗口中执行验证。

**验证方法：**

```powershell
# 在独立的 cmd.exe 进程中运行（模拟双击）
Start-Process cmd.exe -ArgumentList "/c 目标.bat & pause" -Wait

# 或者直接文件资源管理器中双击目标.bat
```

**验收标准：**
- [x] IDE 内嵌终端运行：中文显示正常
- [x] 独立 CMD 窗口运行（模拟双击）：中文显示正常
- [x] 调用子 bat 时：子 bat 中文参数传递正确
- [x] 调用 Python 脚本时：Python 输出中文无乱码
- [x] 退出后 CMD 窗口无报错信息

> 凡是 IDE 运行正常、本地双击出现中文乱码、参数异常、子进程调用失败的脚本，**一律判定为不合格**，必须重新修复。

## 标准模板

### 入口启动脚本模板

```bat
@echo off
chcp 65001 >nul 2>&1
title Project Name - Description
setlocal enabledelayedexpansion

echo ========================================
echo   Title
echo ========================================
echo.

REM ------ Pre-checks ------
if not exist "some_path" (
    echo [x] Check failed description
    pause
    exit /b 1
)

REM ------ Main Logic ------
echo [1/3] Step one...
call some_command
if errorlevel 1 (
    echo [x] Step one failed
    pause
    exit /b 1
)
echo [ok] Step one done
echo.

echo [2/3] Step two...
python main.py --param value
echo [ok] Step two done
echo.

echo [3/3] Launch complete...
echo.

echo ========================================
echo [ok] All operations completed!
echo ========================================
pause
```

### 子脚本模板（无 chcp）

```bat
@echo off
REM Sub bat: no chcp, inherit parent encoding
setlocal enabledelayedexpansion

REM Handle parameters when called by entry bat
set "INPUT_FILE=%~1"
set "OUTPUT_DIR=%~2"

echo [info] Processing: %INPUT_FILE%
python script.py "%INPUT_FILE%" "%OUTPUT_DIR%"
if errorlevel 1 (
    echo [x] Processing failed
    exit /b 1
)
echo [ok] Processing done
exit /b 0
```

### Python 调用规范

```bat
@echo off
chcp 65001 >nul 2>&1
REM Python script header MUST have # -*- coding: utf-8 -*-
python script.py
if errorlevel 1 (
    echo [x] Script execution failed
    pause
    exit /b 1
)
```

## 验证清单

编写/修改 `.bat` 文件后逐项检查：

- [ ] **全程未使用中文字符**（文件内搜索 `[^\x00-\x7F]` 无匹配）
- [ ] 文件编码为 **UTF-8（无 BOM）**
- [ ] 入口文件 **第二行** 为 `chcp 65001 >nul 2>&1`
- [ ] 子 bat 文件 **不包含** `chcp` 命令
- [ ] 调用 Python 脚本头部有 `# -*- coding: utf-8 -*-`
- [ ] 已通过 **独立 CMD 窗口** 验证中文显示正常（模拟双击）
- [ ] IDE 内嵌终端中文显示正常
- [ ] `@echo off` 和 `setlocal enabledelayedexpansion` 按需使用

## 常见错误模式

| 错误模式 | 正确做法 |
|---------|---------|
| 文件保存为 GBK/ANSI | 保持 UTF-8 编码 |
| `chcp 65001 >nul` 省略 `2>&1` | 写全 `chcp 65001 >nul 2>&1` |
| 在子 bat 中重复写 `chcp` | 删除子 bat 中的 `chcp` |
| 调用 Python 脚本未声明编码 | Python 文件头部加 `# -*- coding: utf-8 -*-` |
| 仅在 IDE 终端测试就认为通过 | 必须模拟双击，在独立 CMD 中验证 |
| 使用 `::` 写中文注释 | 用 `REM` 写中文注释（`::` 在 `setlocal` 块内可能出错） |

## 工作原理

### 编码链路

```
文件编码 (UTF-8)
    ↓
入口 bat → chcp 65001 → CMD 切换到 UTF-8 模式
    ↓
子 bat / Python 脚本 → 继承 UTF-8 环境输出正确中文
```

### 为什么不用 GBK？

1. SearchReplace 工具和 Write 工具自动以 **UTF-8** 保存文件
2. 强制转换为 GBK 后，工具再次编辑时又变回 UTF-8 → 文件编码错乱 → 乱码
3. Trae IDE 内嵌终端使用 UTF-8（65001）→ GBK 文件在 IDE 中显示乱码
4. UTF-8 是通用编码标准，跨平台兼容性更好

### 为什么入口必须 `chcp 65001`？

- 用户 Windows 10+ 已有良好的 UTF-8 支持
- `chcp 65001 >nul 2>&1` 将 CMD 窗口切换到 UTF-8 模式
- 后续所有 echo 输出、参数传递、子进程输出都与文件 UTF-8 编码一致
- `2>&1` 确保即使 chcp 执行失败（如旧系统），也不会中断脚本执行

### 为什么子 bat 禁止重复 chcp？

- `chcp` 会切换当前进程的代码页
- `call` 调用子 bat 在**同一进程**中执行
- 子 bat 再次 `chcp` 可能导致编码回退到 GBK → 后续输出乱码
- 编码切换应当由入口文件统一管理

## 历史教训（经验总结）

此规范源于多次调试 BAT 编码问题积累的经验，记录典型坑点以防重犯：

1. **GBK 转换陷阱**：将 bat 文件从 UTF-8 转为 GBK 后，AI 工具（SearchReplace/Write）再次编辑会以 UTF-8 重新保存，产生「UTF-8+GBK 混合」的畸形编码，中文显示为类似 `缁熶竴` 的乱码。

2. **IDE 测试陷阱**：AI 在 IDE 终端（UTF-8）测试通过后，用户双击（GBK CMD）必然乱码。AI 无法在 IDE 中复现乱码，导致误判为「已修复」。

3. **chcp 省略陷阱**：写 `chcp 65001 >nul` 省略 `2>&1`，在极少数场景下 `chcp` 失败的错误输出会污染管道。

4. **子 bat 递归陷阱**：入口 bat 调用的子 bat 中如果再次写 `chcp 65001`，会导致同一个 CMD 进程编码被重复切换，产生不可预测的行为。

5. **Python 编码陷阱**：Python 3 默认输出使用 UTF-8，与 CMD 的 GBK 模式不匹配。脚本头部加 `# -*- coding: utf-8 -*-` 统一声明。

6. **chcp 删除陷阱**：修改现有 bat 文件时误删或覆盖掉 `chcp 65001 >nul 2>&1` 行（尤其是用 Write 工具完整重写时），导致双击 bat 时 CMD 以 GBK 编码读取 UTF-8 文件，中文全部乱码。修复时必须保证第2行恢复 `chcp 65001 >nul 2>&1`。

7. **SearchReplace 编码陷阱**：在 UTF-8 编码的 bat 文件上使用 SearchReplace 工具匹配包含中文的 old_str 时，工具内部编码处理可能产生错乱，导致中文被「双重编码」变为乱码。**正确的修改方式**：使用 Python 生成 bat 文件（见第8条），禁止在 .bat 上使用 SearchReplace 修改含中文的行。

8. **Write+SearchReplace 双重破坏陷阱**：Write 工具写入 bat 文件时使用 LF 换行（而非 Windows 标准的 CRLF），而 SearchReplace 工具进一步修改时会再次破坏换行格式（可能产生双重 CRLF 或混合换行）。CMD 在解析 LF 换行的 bat 文件时，遇到括号块（`if/else` 等）可能解析错乱，产生看似乱码的报错。

   **正确的 BAT 文件修改方法（核心流程）：**

   ```
   ❌ 错误做法：Write 直接写 bat → SearchReplace 修改 → 编码/换行双重破坏
   ✅ 正确做法：Write 写 Python 生成器 → Python 生成 bat → 删除生成器
   ```

   **Python 生成器的标准写法：**
   ```python
   # -*- coding: utf-8 -*-
   path = r'目标.bat'
   lines = []
   lines.append('@echo off')
   lines.append('chcp 65001 >nul 2>&1')
   # ... 所有行按顺序加入 lines 列表 ...
   text = '\r\n'.join(lines) + '\r\n'
   with open(path, 'w', encoding='utf-8', newline='\r\n') as f:
       f.write(text)
   ```

   **关键约束：**
   - 必须先删除旧 bat 文件（清除历史编码污染）
   - `encoding='utf-8'` → 确保 UTF-8 无 BOM
   - `newline='\r\n'` → 确保 CRLF 换行（Windows 标准）
   - 用 `lines` 列表逐行构建内容 → 避免字符串拼接遗漏换行
   - 运行生成器后立即删除它 → 不给用户留垃圾文件

9. **模板参照原则**：修改 bat 文件前，先找一个用户本地**能正常运行的 bat 文件**作为参照模板，严格模仿其风格，包括：
   - 换行方式（CRLF ✅ vs 其他 ❌）
   - `%ERRORLEVEL%` vs `if errorlevel` 的选择（与模板一致）
   - 缩进风格
   - `@echo off` / `chcp 65001` / `title` 等头部格式
   - 注释风格（`::` 或 `REM`）