# Browser-Use 使用手册

## 概述

Browser-Use 是一个基于 AI 的自动化浏览器操作框架，能够理解自然语言任务并自动执行网页交互、数据提取和文件操作。

## 核心特性

### 🤖 AI 驱动的自动化
- **自然语言任务描述**：用中文或英文描述你想要完成的任务
- **智能决策**：AI 自动分析页面内容，决定下一步操作
- **视觉理解**：支持截图分析，理解页面布局和元素

### 🌐 浏览器操作能力
- **页面导航**：访问任意 URL，支持多标签页
- **元素交互**：点击、输入、滚动、拖拽等操作
- **表单处理**：自动填写表单，处理登录流程
- **数据提取**：结构化提取网页信息

### 📁 文件系统集成
- **文件读写**：支持本地文件操作
- **数据保存**：自动保存提取的数据
- **格式转换**：支持多种数据格式

## 快速开始

### 1. 环境准备

```bash
# 安装依赖
pip install browser-use

# 设置 API 密钥
export OPENAI_API_KEY="your-api-key"
```

### 2. 基本使用

```python
from browser_use import Agent
from browser_use.llm import ChatOpenAI

# 创建 Agent
llm = ChatOpenAI(model='gpt-4o')
agent = Agent(
    task="访问 GitHub 并搜索 Python 项目",
    llm=llm
)

# 执行任务
history = await agent.run()
```

### 3. 高级配置

```python
agent = Agent(
    task="复杂任务描述",
    llm=llm,
    # 浏览器配置
    browser_session=BrowserSession(
        headless=False,  # 显示浏览器窗口
        allowed_domains=["github.com"],  # 限制访问域名
    ),
    # 文件系统配置
    file_system_path="/path/to/your/project",
    # 敏感数据处理
    sensitive_data={
        "username": "your-username",
        "password": "your-password"
    }
)
```

## 功能详解

### 任务描述格式

```
任务描述应该包含：
1. 目标网站或 URL
2. 具体操作步骤
3. 期望的输出结果
4. 特殊要求（如格式、文件路径等）

示例：
"访问 https://example.com，登录后搜索 'Python'，
提取前 5 个搜索结果，保存到 results.txt 文件中"
```

### 支持的操作类型

#### 基础操作
- `go_to_url()` - 页面导航
- `click_element_by_index()` - 点击元素
- `type_text()` - 输入文本
- `scroll()` - 页面滚动
- `wait_for_element()` - 等待元素加载

#### 数据操作
- `extract_structured_data()` - 结构化数据提取
- `write_file()` - 写入文件
- `read_file()` - 读取文件
- `append_to_file()` - 追加文件内容

#### 高级操作
- `take_screenshot()` - 截图
- `download_file()` - 下载文件
- `upload_file()` - 上传文件
- `multi_act()` - 批量操作

### 文件系统使用

#### 文件路径说明
- **相对路径**：相对于 Agent 工作目录
- **绝对路径**：需要配置 `file_system_path`
- **临时目录**：默认写入到 Agent 沙箱目录

#### 推荐的文件操作方式
```python
# 方式1：配置工作目录
agent = Agent(
    task="...",
    llm=llm,
    file_system_path="/Users/your/project"
)

# 方式2：自定义 Action（推荐）
@controller.registry.action("写入本地文件")
async def write_local_file(content: str, file_path: str):
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(content)
    return ActionResult(extracted_content=f"写入成功: {file_path}")
```

## 最佳实践

### 1. 任务描述优化
- **具体明确**：详细描述每个步骤
- **格式要求**：明确输出格式和文件路径
- **错误处理**：说明异常情况的处理方式

### 2. 安全配置
```python
# 限制访问域名
browser_session = BrowserSession(
    allowed_domains=["trusted-site.com"],
    sensitive_data={"api_key": "your-key"}
)

# 设置超时和重试
agent = Agent(
    task="...",
    llm=llm,
    max_failures=3,
    retry_delay=10
)
```

### 3. 性能优化
```python
# 启用缓存
agent = Agent(
    task="...",
    llm=llm,
    use_vision=False,  # 关闭视觉分析提升速度
    max_actions_per_step=5  # 限制每步操作数量
)
```

### 4. 调试和监控
```python
# 启用详细日志
import os
os.environ['BROWSER_USE_LOGGING_LEVEL'] = 'debug'

# 保存对话记录
agent = Agent(
    task="...",
    llm=llm,
    save_conversation_path="./conversations"
)
```

## 常见问题

### Q: 文件写入到临时目录怎么办？
A: 使用 `file_system_path` 参数或自定义 Action 直接写入本地路径。

### Q: 如何处理登录验证？
A: 使用 `sensitive_data` 参数传递凭据，或手动登录后保存 cookies。

### Q: 任务执行失败怎么办？
A: 检查日志，调整任务描述，增加重试次数和错误处理。

### Q: 如何提高执行效率？
A: 关闭不必要的功能（如视觉分析），优化任务描述，使用更快的模型。

## 限制和注意事项

### 技术限制
- **模型依赖**：需要 OpenAI API 或其他支持的 LLM
- **网络要求**：需要稳定的网络连接
- **资源消耗**：浏览器和 AI 调用会消耗较多资源

### 安全限制
- **域名限制**：建议设置 `allowed_domains` 限制访问范围
- **文件权限**：Agent 只能访问指定目录
- **敏感数据**：避免在任务描述中暴露敏感信息

### 功能限制
- **复杂交互**：某些复杂的 JavaScript 交互可能不支持
- **CAPTCHA**：无法自动处理验证码
- **动态内容**：高度动态的页面可能需要特殊处理

## 故障排除

### 常见错误
1. **API 密钥错误**：检查环境变量设置
2. **网络连接失败**：检查网络和防火墙设置
3. **元素找不到**：调整等待时间或元素定位方式
4. **文件权限错误**：检查目录权限和路径配置

### 调试技巧
1. **启用调试日志**：设置 `BROWSER_USE_LOGGING_LEVEL=debug`
2. **查看浏览器窗口**：设置 `headless=False`
3. **保存截图**：使用 `take_screenshot()` 分析页面状态
4. **分步执行**：将复杂任务拆分为多个简单任务

## 进阶功能

### 自定义 Actions
```python
@controller.registry.action("自定义操作")
async def custom_action(param1: str, param2: int):
    # 实现自定义逻辑
    result = do_something(param1, param2)
    return ActionResult(extracted_content=result)
```

### 多 Agent 协作
```python
# 创建多个 Agent 处理不同任务
agent1 = Agent(task="数据收集", llm=llm)
agent2 = Agent(task="数据分析", llm=llm)

# 串行执行
history1 = await agent1.run()
history2 = await agent2.run()
```

### 云端同步
```python
# 启用云端功能
agent = Agent(
    task="...",
    llm=llm,
    cloud_sync=CloudSync()
)
```

## 总结

Browser-Use 提供了强大的 AI 驱动浏览器自动化能力，通过合理配置和最佳实践，可以高效完成各种网页操作任务。关键是要理解其工作原理，合理设置安全限制，并优化任务描述以获得最佳效果。 