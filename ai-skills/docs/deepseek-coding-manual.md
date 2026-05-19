# deepseek-coding-manual

> DeepSeek V4 Pro 编程思维手册。触发条件：需求拆解、项目搭建规划、技术选型分析、多模块任务分解、系统架构设计。不匹配排错调试和代码编写等执行层任务，这些任务由其他专属Skill处理。

---


# DeepSeek V4 Pro 全能编程专属SKILL规则手册

> **核心原则**：本手册是完整复刻 DeepSeek V4 Pro 全部思考逻辑、解题思路、排错经验、编码习惯、项目搭建逻辑的专属行为准则。
> **使用对象**：推理弱、思路乱、调试低效、只会盲目试错的低配AI模型（通义灵码、轻量模型等）。
> **强制要求**：低配AI拿到本手册后，执行任何匹配场景任务时，**彻底抛弃自身混乱思维，全程1:1照搬本手册思维模式做事**。**如需编写测试脚本或使用截图功能，一律保存到 test/ 目录。**

---

# 第一部分：SKILL触发调用机制

## 1.1 自动触发关键词与场景

只要用户下发需求包含以下**任意一个**关键词或场景，必须**立刻启用本全套SKILL**，禁止使用自身原有思维：

### A类：编程开发（最高优先级触发）

| 触发词 | 典型需求示例 |
|--------|-------------|
| Python脚本 | "写一个Python脚本"、"帮我写个py文件" |
| 自动化程序 | "自动化发布"、"批量处理"、"定时任务" |
| Playwright | "浏览器操控"、"网页自动化"、"Playwright脚本" |
| 爬虫开发 | "爬取网页"、"数据采集"、"网页抓取" |
| API开发 | "写个接口"、"FastAPI服务"、"后端接口" |

### B类：AI模型相关

| 触发词 | 典型需求示例 |
|--------|-------------|
| 模型路由 | "模型中转"、"API转发"、"多模型调度" |
| 模型对接 | "对接大模型"、"接入新模型"、"网页版模型" |
| IDE代理 | "Cursor配置"、"Trae代理"、"IDE接入" |
| 聚合调度 | "多模型聚合"、"统一API"、"模型池" |

### C类：网络与部署

| 触发词 | 典型需求示例 |
|--------|-------------|
| 内网穿透 | "ngrok配置"、"公网暴露"、"隧道" |
| 端口进程 | "端口占用"、"进程排查"、"端口冲突" |
| 服务部署 | "部署服务"、"启动服务"、"后台运行" |
| 环境配置 | "环境变量"、".env配置"、"依赖安装" |

## 1.2 关闭规则

以下场景**自动停用本SKILL**，恢复普通对话模式：
- 日常闲聊、问好
- 生活问答（天气、饮食、旅游等）
- 文案娱乐（写诗、讲故事、笑话等）
- 纯理论探讨（不涉及实际编码）
- 非技术开发类需求

## 1.3 低配AI强制使用规范

1. **拿到本SKILL后，执行任何匹配场景任务时，优先通读对应章节流程**
2. **严格按照手册内既定顺序、既定思路、既定方案执行**
3. **禁止自主创新思路** — 手册没写的方案不要用
4. **禁止随意简化步骤** — 手册标注"必做"的步骤一个不能少
5. **禁止凭自身浅薄认知乱修改写法** — 手册给的代码模板直接套用

## 1.4 优先级规则

手册内写明的最优方案、排查顺序、避坑方法，**优先级高于低配AI自身所有认知**，必须严格遵守，不得质疑。

---

# 第二部分：标准需求拆解思维

## 2.1 需求理解流程（接到任何需求后，按此顺序思考）

```
步骤1：确认输入输出
  ├→ 输入是什么？（文件？URL？用户输入？API请求？）
  ├→ 输出是什么？（文件？打印？API响应？数据库写入？）
  └→ 中间需要经过哪些转换？

步骤2：确认边界条件
  ├→ 数据量多大？（几个文件还是几万个？）
  ├→ 频率多高？（一次性的还是持续运行？）
  ├→ 容错要求？（失败了重试还是跳过？）
  └→ 性能要求？（毫秒级还是分钟级？）

步骤3：确认已有资源
  ├→ 项目里有没有类似代码可以参考？
  ├→ 有没有现成的库/工具可以用？
  └→ 有没有配置文件/环境变量需要读取？

步骤4：确认技术选型
  ├→ 优先用项目已有的库和框架
  ├→ 不要引入新依赖除非必要
  └→ 简单优先：能用一个脚本解决的不建项目
```

## 2.2 模块拆分方法

```
拆分原则：
  ├→ 单一职责：一个函数只做一件事
  ├→ 输入输出明确：函数签名一看就懂
  ├→ 可测试：每个函数可以独立测试
  └→ 可复用：相似逻辑不写两遍

拆分示例（以"网页内容采集"为例）：
  ├→ fetch_page(url) → 获取网页HTML
  ├→ parse_content(html) → 解析提取正文
  ├→ extract_images(html) → 提取图片
  ├→ clean_text(text) → 清洗文本
  ├→ save_article(data) → 保存结果
  └→ main() → 串联上述步骤
```

## 2.3 主次划分原则

```
优先级排序（从高到低）：
  1. 核心功能（必须能跑通） — 先实现，先测试
  2. 错误处理（不能崩溃） — 核心功能跑通后加
  3. 日志记录（方便排查） — 错误处理后加
  4. 性能优化（快一点） — 最后做，非必要不做
  5. 代码美化（好看一点） — 永远不做，除非用户明确要求

判断标准：
  ├→ 没有这个功能，整个程序能不能跑？
  │   ├→ 不能 → 核心功能，先做
  │   └→ 能 → 锦上添花，后做
  └→ 这个功能用户明确要求了吗？
      ├→ 是 → 必须做
      └→ 否 → 不做（YAGNI原则）
```

## 2.4 技术选型决策树

```
需要存储数据？
  ├→ 少量配置 → .env 或 config.yaml
  ├→ 中等量结构化数据 → JSON文件
  ├→ 大量数据需要查询 → SQLite
  └→ 高并发读写 → PostgreSQL/MySQL

需要HTTP服务？
  ├→ 简单API（<5个接口） → FastAPI单文件
  ├→ 复杂API → FastAPI + Router分模块
  └→ 静态文件服务 → nginx 或 python -m http.server

需要浏览器自动化？
  ├→ 简单爬取 → requests + BeautifulSoup
  ├→ 需要JS渲染 → Playwright
  └→ 需要登录态 → Playwright + storage_state

需要定时任务？
  ├→ Windows → 任务计划程序 + bat脚本
  ├→ Linux → cron + shell脚本
  └→ 跨平台 → Python + schedule库
```

---

# 第三部分：全场景项目落地思路

## 3.1 从零新建项目思路

```
步骤顺序（严格按此执行）：

1. 确认需求边界
   ├→ 问清楚：输入什么？输出什么？运行环境？
   └→ 不确定就问，不要猜

2. 搜索项目内是否有类似代码
   ├→ 用 SearchCodebase 搜索关键词
   ├→ 用 Glob 搜索相似文件名
   └→ 有类似代码 → 复制模板再改（改动>30%必须复制）

3. 确定文件位置
   ├→ 单脚本 → 放在相关功能目录中
   ├→ 多文件模块 → 新建子目录
   └→ 测试脚本 → 必须放 test/ 目录

4. 先写核心逻辑（最小可用版本）
   ├→ 不写错误处理
   ├→ 不写日志
   ├→ 不写配置
   └→ 只写核心功能，跑通再说

5. 核心跑通后，逐步加：
   ├→ 错误处理（try-except）
   ├→ 日志记录（logger.info/error）
   ├→ 配置化（.env / config）
   └→ 边界情况处理

6. 测试验证
   ├→ 正常情况测试
   ├→ 异常情况测试
   └→ 边界情况测试
```

## 3.2 残缺代码补全思路

```
1. 先理解现有代码在做什么
   ├→ 读函数名和参数
   ├→ 读注释（如果有）
   └→ 追踪调用链

2. 找到缺口
   ├→ 哪个函数没实现？
   ├→ 哪个步骤被跳过了？
   └→ 哪个变量没定义？

3. 参考项目内类似实现
   ├→ 搜索相似函数名
   ├→ 搜索相似功能代码
   └→ 复制最接近的实现再修改

4. 补全后验证
   ├→ 检查导入是否完整
   ├→ 检查类型是否匹配
   └→ 运行测试
```

## 3.3 旧项目重构优化思路

```
重构三原则：
  1. 不改变外部行为（输入输出不变）
  2. 小步快跑（一次只改一个模块）
  3. 每步可验证（改完立刻测试）

重构步骤：
  1. 先写测试（保证改后行为不变）
  2. 提取重复代码为函数
  3. 统一命名风格
  4. 拆分大函数
  5. 移除无用代码
  6. 统一配置管理
```

## 3.4 批量模板化开发思路

```
适用场景：需要创建多个结构相似的脚本/模块

步骤：
  1. 先做好一个完整模板（能跑通）
  2. 提取可变部分（选择器、URL、类名等）
  3. 复制模板文件 → 改名 → 修改可变部分
  4. 规则：改动>30%必须复制模板，不允许手写

示例（新增浏览器模型Provider）：
  copy deepseek.py → doubao.py
  修改：类名、URL、选择器常量
  不改：navigate()、submit_prompt()、extract_streaming() 等核心逻辑
```

---

# 第四部分：高频疑难问题全套解决方案

## 4.1 端口占用问题

**现象**：启动服务报 `Address already in use` 或 `端口被占用`

**排查**：
```bash
netstat -ano | findstr "端口号"
```

**解决**：
```bash
taskkill /F /PID 进程ID
```

**预防**：启动前检查端口，写进启动脚本

## 4.2 进程残留问题

**现象**：上次关闭了但进程还在跑，端口被占

**排查**：
```bash
tasklist | findstr "python"
tasklist | findstr "node"
```

**解决**：
```bash
taskkill /F /IM python.exe
taskkill /F /IM node.exe
```

**预防**：脚本加信号处理，确保 Ctrl+C 能正常退出

## 4.3 编码乱码问题

**现象**：中文显示为乱码、问号、方块

**成因**：文件编码与读取编码不一致

**解决**：
- Python 文件第一行加 `# -*- coding: utf-8 -*-`
- 读写文件时显式指定 `encoding='utf-8'`
- PowerShell 脚本首行加 `chcp 65001`
- 不要用 PowerShell 的 `Set-Content` 写中文文件，用 Python 写

**预防**：所有文件统一 UTF-8 无 BOM

## 4.4 环境变量未加载

**现象**：`os.environ.get('KEY')` 返回 None

**排查**：
```python
from dotenv import load_dotenv
load_dotenv(override=True)
print(os.environ.get('KEY'))
```

**解决**：确认 `.env` 文件在运行目录下，格式为 `KEY=VALUE`（不要引号）

## 4.5 依赖缺失/版本冲突

**现象**：`ModuleNotFoundError` 或 `ImportError`

**排查**：
```bash
pip list | findstr "包名"
```

**解决**：
```bash
pip install -r requirements.txt
pip install 包名 --upgrade
```

**预防**：用 `requirements.txt` 锁定版本，用虚拟环境隔离

## 4.6 Playwright 浏览器未安装

**现象**：`BrowserType.launch: Executable doesn't exist`

**解决**：
```bash
playwright install chromium
```

## 4.7 Playwright 选择器超时

**现象**：`TimeoutError: Waiting for selector "xxx" failed`

**排查顺序**：
1. 手动打开网页，F12检查选择器是否正确
2. 检查元素是否在 iframe 内
3. 检查是否需要先登录
4. 检查页面加载是否完成（`wait_until="networkidle"`）

**解决**：增加超时时间，或改用 `page.wait_for_selector(selector, timeout=30000)`

## 4.8 CSS :hover 不触发

**现象**：用 `dispatchEvent(new MouseEvent('mouseover'))` 无法触发 hover 效果

**成因**：CSS `:hover` 伪类只响应真实鼠标移动，不响应 JS 事件

**解决**：使用 `page.mouse.move(x, y)` 模拟真实鼠标移动

## 4.9 SVG 元素点击失败

**现象**：`svgElement.click()` 报错或无效

**成因**：SVG 元素不支持 `click()` 方法

**解决**：使用 `dispatchEvent(new MouseEvent('click', {bubbles: true}))`

## 4.10 会话/状态污染

**现象**：多次运行后行为异常，数据混乱

**成因**：上次运行的会话、cookie、localStorage 残留

**解决**：
- Playwright：每次启动用新的 `browser_context`，或 `context.clear_cookies()`
- 网页模型：每次运行前清理旧会话
- 数据文件：运行前检查并清理临时文件

## 4.11 请求超时

**现象**：`TimeoutError` 或请求卡住不动

**排查顺序**：
1. 检查网络连通性（ping 目标地址）
2. 检查代理/VPN 是否正常
3. 检查防火墙是否拦截
4. 增加超时时间测试

**解决**：设置合理的超时 + 重试机制（最多3次，指数退避）

## 4.12 并发/队列异常

**现象**：多任务并发时结果混乱、丢失

**解决**：
- 使用 `asyncio.Semaphore` 限制并发数
- 使用 `asyncio.Queue` 管理任务队列
- 关键操作加锁 `asyncio.Lock`

---

# 第五部分：专属业务定向思路

## 5.1 本项目核心架构速查

```
lama-puppeteer: f:\myclaw\lama-puppeteer
  ├→ API Key: sk-lamapuppeteer-admin-key-2026
  ├→ new-api端口: 3000
  ├→ new-api账号: admin / admin123456
  └→ 启动: start_all.bat

data_collector: f:\myclaw\data_collector
  ├→ tracking: output/tracking (≤7天文章)
  ├→ knowledge_base: output/knowledge_base/external (>7天文章)
  │   ├→ archive (临时仓库)
  │   ├→ base (基础库)
  │   ├→ incremental (增量库)
  │   └→ permanent_archive (永久归档)
  └→ ID规则: MD5(URL)[:12]

self_evolving_engine: f:\myclaw\multi-platform-automation\self_evolving_engine
  ├→ 模板: output/evolution/patterns/writing_templates.json
  ├→ 分类器: content_classifier.py (统一分类入口)
  └→ 调度器: task_scheduler.py
```

## 5.2 本项目关键规则

1. **分类统一入口**：所有分类逻辑必须通过 `content_classifier.py` 的 `ContentClassifier` 类
2. **ID生成规则**：12位ID = MD5(URL)[:12]，全项目统一
3. **知识库四层架构**：新文章 → archive → organize() → base/incremental
4. **增量分发覆盖更新**：`external_dispatch.py` 处理增量文章，覆盖同名文件更新互动数据
5. **基础库分发不覆盖**：`classify_external_articles.py` 处理基础库文章，跳过已存在文章
6. **模板选择**：调度器通过聚类提炼生成的 `writing_templates.json` 选择模板
7. **双轨纠错**：对比本周与上周数据，评估内容质量趋势（已启用）
8. **范式提炼**：暂不启用，作为聚类提炼的备选方案

## 5.3 本项目常用命令

```bash
# === Python ===
python --version                    # Python版本
pip list                            # 已安装包列表
pip install -r requirements.txt     # 安装依赖
pip install 包名 --upgrade          # 升级包

# === 端口进程 ===
netstat -ano | findstr "端口号"      # 查端口占用
tasklist | findstr "PID"            # 查进程名
taskkill /F /PID PID号             # 杀进程
taskkill /F /IM python.exe          # 批量杀

# === Playwright ===
playwright install chromium         # 安装浏览器
playwright codegen URL              # 启动代码生成器

# === ngrok ===
ngrok config check                  # 检查配置
ngrok config add-authtoken TOKEN    # 配置token
ngrok http 端口号 --log=stdout      # 启动隧道
curl http://127.0.0.1:4040/api/tunnels  # 查看隧道状态

# === Git ===
git status                          # 查看状态
git diff                            # 查看改动
git log -n 5                        # 最近5条提交

# === 健康检查 ===
curl http://localhost:9091/health   # 健康检查
curl http://localhost:3000/api/     # new-api检查
```

---

# 附录：避坑速查表

| 序号 | 坑点 | 正确做法 |
|------|------|---------|
| 1 | 用 PowerShell Set-Content 写中文 | 用 Python open() 写文件 |
| 2 | 用 dispatchEvent 触发 hover | 用 page.mouse.move() |
| 3 | 在 SVG 上直接 .click() | 用 dispatchEvent(MouseEvent) |
| 4 | 不等待页面加载就操作 | 用 wait_for_selector 等待 |
| 5 | 不检查返回值类型 | 用 isinstance() 检查 |
| 6 | 静默吞掉异常 | 至少 logger.error 记录 |
| 7 | 硬编码路径和配置 | 用 .env 或 config 文件 |
| 8 | IDE URL 加 /v1 后缀 | new-api自动追加，不要手动加 |
| 9 | 不清理旧会话就运行 | 每次运行前清理 |
| 10 | 不检查端口就启动 | 启动前 netstat 检查 |


---

## How to Use

### Option 1: View directly on GitHub
Open this document directly in your browser to read the full content.

### Option 2: Clone to local
`ash
git clone https://github.com/drgon1/santian.git
cd santian/ai-skills/.lingma/skills/deepseek-coding-manual
`

### Option 3: Import to Lingma IDE
Copy the entire $skillName folder to your .lingma/skills/ directory.

---

*This document was automatically generated by [generate_github_docs.ps1](../generate_github_docs.ps1)*