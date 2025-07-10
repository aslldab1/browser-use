# Browser-Use 快速参考手册

## 快速开始

### 安装和配置
```bash
# 安装
pip install browser-use

# 设置 API 密钥
export OPENAI_API_KEY="your-api-key"

# 或创建 .env 文件
echo "OPENAI_API_KEY=your-api-key" > .env
```

### 基本使用
```python
from browser_use import Agent
from browser_use.llm import ChatOpenAI

# 创建 Agent
llm = ChatOpenAI(model='gpt-4o')
agent = Agent(
    task="访问 https://example.com 并提取标题",
    llm=llm
)

# 执行任务
history = await agent.run()
```

## 常用配置

### Agent 配置
```python
agent = Agent(
    task="任务描述",
    llm=llm,
    # 浏览器配置
    browser_session=BrowserSession(
        headless=False,  # 显示浏览器窗口
        allowed_domains=["example.com"],  # 限制域名
        viewport={"width": 1920, "height": 1080},  # 视窗大小
    ),
    # 文件系统配置
    file_system_path="/path/to/project",  # 工作目录
    # 执行配置
    max_steps=100,  # 最大步数
    max_failures=3,  # 最大失败次数
    # 日志配置
    save_conversation_path="./conversations",  # 保存对话
    generate_gif=True,  # 生成 GIF
)
```

### 环境变量配置
```bash
# 日志级别
export BROWSER_USE_LOGGING_LEVEL=debug

# 成本计算
export BROWSER_USE_CALCULATE_COST=true

# 敏感数据
export BROWSER_USE_SENSITIVE_DATA='{"username": "user", "password": "pass"}'
```

## 常用任务示例

### 1. 数据提取
```python
task = """
访问 https://news.ycombinator.com
提取前 10 个新闻标题和链接
保存到 hacker_news.txt 文件
格式：标题 | 链接
"""
```

### 2. 表单填写
```python
task = """
访问 https://example.com/contact
填写联系表单：
- 姓名：John Doe
- 邮箱：john@example.com
- 消息：Hello World
点击提交按钮
"""
```

### 3. 登录操作
```python
agent = Agent(
    task="登录到 https://example.com",
    llm=llm,
    sensitive_data={
        "username": "your-username",
        "password": "your-password"
    }
)
```

### 4. 文件下载
```python
task = """
访问 https://example.com/downloads
下载所有 PDF 文件
保存到 downloads 文件夹
"""
```

### 5. 截图和监控
```python
task = """
访问 https://example.com
等待页面加载完成
截图保存为 screenshot.png
提取页面标题和描述
"""
```

## 内置动作参考

### 导航操作
```python
# 页面导航
go_to_url(url: str, new_tab: bool = False)

# 返回上一页
go_back()

# 刷新页面
refresh_page()
```

### 元素交互
```python
# 点击元素
click_element_by_index(index: int)
click_element_by_text(text: str)
click_element_by_selector(selector: str)

# 输入文本
type_text(text: str, element_index: int = None)

# 滚动页面
scroll(direction: str = "down", amount: int = 1000)
```

### 数据提取
```python
# 提取结构化数据
extract_structured_data(
    extraction_goal: str,
    extraction_instructions: str = None
)

# 提取文本
extract_text(element_index: int = None)

# 获取页面信息
get_page_info()
```

### 文件操作
```python
# 写入文件
write_file(file_name: str, content: str)

# 追加文件
append_to_file(file_name: str, content: str)

# 读取文件
read_file(file_name: str)

# 下载文件
download_file(url: str, file_name: str = None)
```

### 高级操作
```python
# 截图
take_screenshot(file_name: str = None)

# 上传文件
upload_file(file_path: str, element_index: int)

# 拖拽操作
drag_and_drop(
    source_index: int,
    target_index: int
)

# 批量操作
multi_act(actions: list[ActionModel])
```

## 错误处理

### 常见错误和解决方案

#### 1. 元素找不到
```python
# 增加等待时间
agent = Agent(
    task="...",
    llm=llm,
    browser_session=BrowserSession(
        wait_for_timeout=10000  # 10秒等待
    )
)
```

#### 2. 网络超时
```python
# 设置超时时间
agent = Agent(
    task="...",
    llm=llm,
    browser_session=BrowserSession(
        navigation_timeout=30000  # 30秒超时
    )
)
```

#### 3. 验证码处理
```python
# 手动处理验证码
task = """
访问登录页面
填写用户名和密码
等待手动处理验证码
按 Enter 继续执行
"""
```

#### 4. 文件权限错误
```python
# 使用绝对路径
agent = Agent(
    task="...",
    llm=llm,
    file_system_path="/Users/your/project"
)
```

## 调试技巧

### 1. 启用详细日志
```python
import os
import logging

# 设置环境变量
os.environ['BROWSER_USE_LOGGING_LEVEL'] = 'debug'

# 配置日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

### 2. 查看浏览器窗口
```python
agent = Agent(
    task="...",
    llm=llm,
    browser_session=BrowserSession(headless=False)
)
```

### 3. 保存执行记录
```python
agent = Agent(
    task="...",
    llm=llm,
    save_conversation_path="./debug",
    generate_gif=True
)
```

### 4. 分步调试
```python
# 手动执行单步
await agent.step()

# 查看当前状态
print(agent.state.history.messages[-1])
```

## 性能优化

### 1. 模型选择
```python
# 快速模型（适合简单任务）
llm = ChatOpenAI(model='gpt-4o-mini')

# 高质量模型（适合复杂任务）
llm = ChatOpenAI(model='gpt-4o')
```

### 2. 关闭视觉分析
```python
agent = Agent(
    task="...",
    llm=llm,
    use_vision=False  # 提升速度
)
```

### 3. 限制操作数量
```python
agent = Agent(
    task="...",
    llm=llm,
    max_actions_per_step=3  # 每步最多3个操作
)
```

### 4. 缓存配置
```python
agent = Agent(
    task="...",
    llm=llm,
    browser_session=BrowserSession(
        cache_enabled=True,  # 启用缓存
        cache_ttl=3600  # 缓存1小时
    )
)
```

## 安全配置

### 1. 域名限制
```python
browser_session = BrowserSession(
    allowed_domains=["trusted-site.com"],
    blocked_domains=["malicious-site.com"]
)
```

### 2. 文件系统隔离
```python
agent = Agent(
    task="...",
    llm=llm,
    file_system_path="/safe/directory",
    read_only_paths=["/readonly"],
    write_allowed_paths=["/writable"]
)
```

### 3. 敏感数据处理
```python
agent = Agent(
    task="...",
    llm=llm,
    sensitive_data={
        "api_key": "***",
        "password": "***"
    },
    sensitive_data_domains=["api.example.com"]
)
```

## 自定义扩展

### 1. 自定义动作
```python
from browser_use.core.controller import Controller, ActionResult

controller = Controller()

@controller.registry.action("自定义操作")
async def custom_action(param1: str, param2: int):
    # 实现自定义逻辑
    result = await external_api_call(param1, param2)
    return ActionResult(
        extracted_content=result,
        include_in_memory=True
    )

# 使用自定义控制器
agent = Agent(
    task="...",
    llm=llm,
    controller=controller
)
```

### 2. 自定义 LLM
```python
from browser_use.llm.base import BaseChatModel

class CustomLLM(BaseChatModel):
    async def ainvoke(self, messages, output_format=None):
        # 实现自定义 LLM 调用
        response = await self._call_api(messages)
        return ChatInvokeCompletion(
            completion=response,
            usage=self._get_usage(response)
        )

# 使用自定义 LLM
llm = CustomLLM()
agent = Agent(task="...", llm=llm)
```

### 3. 自定义浏览器
```python
from browser_use.browser.browser import BrowserSession

class CustomBrowserSession(BrowserSession):
    async def custom_navigation(self, url: str):
        # 实现自定义导航逻辑
        await self.page.goto(url)
        await self.page.wait_for_load_state()

# 使用自定义浏览器
browser_session = CustomBrowserSession()
agent = Agent(
    task="...",
    llm=llm,
    browser_session=browser_session
)
```

## 监控和统计

### 1. Token 使用统计
```python
# 获取使用统计
token_summary = await agent.token_cost_service.get_usage_summary()
print(f"Total tokens: {token_summary.total_tokens}")
print(f"Total cost: ${token_summary.total_cost}")

# 获取详细统计
usage_details = await agent.token_cost_service.get_usage_details()
for detail in usage_details:
    print(f"Step {detail.step}: {detail.tokens} tokens")
```

### 2. 执行时间统计
```python
import time

start_time = time.time()
history = await agent.run()
end_time = time.time()

print(f"Execution time: {end_time - start_time:.2f} seconds")
print(f"Total steps: {len(history.messages)}")
```

### 3. 成功率统计
```python
successful_steps = sum(1 for msg in history.messages if not msg.error)
total_steps = len(history.messages)
success_rate = successful_steps / total_steps * 100

print(f"Success rate: {success_rate:.1f}%")
```

## 最佳实践

### 1. 任务描述优化
- **具体明确**：详细描述每个步骤
- **格式要求**：明确输出格式和文件路径
- **错误处理**：说明异常情况的处理方式
- **分步执行**：复杂任务拆分为多个简单任务

### 2. 安全配置
- **域名限制**：设置 `allowed_domains` 限制访问范围
- **文件隔离**：使用 `file_system_path` 限制文件访问
- **敏感数据**：通过 `sensitive_data` 安全传递凭据

### 3. 性能优化
- **模型选择**：根据任务复杂度选择合适的模型
- **功能开关**：关闭不必要的功能（如视觉分析）
- **缓存启用**：启用缓存减少重复请求

### 4. 调试和监控
- **日志记录**：启用详细日志便于调试
- **状态保存**：保存执行记录用于分析
- **性能监控**：监控 Token 使用和执行时间

## 常见问题

### Q: 如何解决文件写入到临时目录的问题？
A: 使用 `file_system_path` 参数或自定义 Action 直接写入本地路径。

### Q: 如何处理登录验证？
A: 使用 `sensitive_data` 参数传递凭据，或手动登录后保存 cookies。

### Q: 任务执行失败怎么办？
A: 检查日志，调整任务描述，增加重试次数和错误处理。

### Q: 如何提高执行效率？
A: 关闭不必要的功能（如视觉分析），优化任务描述，使用更快的模型。

### Q: 如何调试复杂任务？
A: 启用详细日志，查看浏览器窗口，分步执行，保存执行记录。

## 资源链接

- [官方文档](https://browser-use.com)
- [GitHub 仓库](https://github.com/browser-use/browser-use)
- [示例代码](https://github.com/browser-use/browser-use/tree/main/examples)
- [问题反馈](https://github.com/browser-use/browser-use/issues) 