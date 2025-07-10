# Agent 最佳实践指南

基于优秀 Agent 项目的设计经验，总结 Agent 开发的核心原则和实用技巧。

## 🎯 核心设计原则

### 1. **分层信息架构** - 让 LLM 理解完整上下文

**问题**：LLM 需要什么信息才能做出好的决策？

**解决方案**：把信息分成不同层次，每层都有明确的作用

```xml
<!-- 第一层：任务背景和目标 -->
<user_request>
访问开关管理页面，收集所有开关的分布信息，保存到文件中
</user_request>

<!-- 第二层：当前状态 -->
<browser_state>
当前URL: https://switch.alibaba-inc.com/#/switchList
页面内容: 显示开关列表，有10个开关项
可交互元素: [33]按钮"分布", [34]按钮"编辑", [35]按钮"删除"
</browser_state>

<!-- 第三层：可用动作 -->
<available_actions>
- click_element_by_index: 点击指定索引的元素
- extract_structured_data: 提取页面结构化数据
- write_file: 写入文件
- scroll: 滚动页面
</available_actions>

<!-- 第四层：约束条件 -->
<constraints>
- 只能点击有索引的元素
- 文件格式必须是 "开关名||机器数||开关值"
- 最多执行20步
</constraints>
```

**实际效果**：LLM 能清楚知道"我要做什么"、"现在在哪里"、"能做什么"、"有什么限制"

### 2. **强制推理框架** - 让 LLM 必须思考

**问题**：LLM 经常直接给出答案，不分析过程，容易出错

**解决方案**：通过 `reasoning_rules` 强制 LLM 按照固定步骤思考

#### **推理规则设计**

在系统提示词中定义详细的推理规则：

```xml
<reasoning_rules>
你必须在每个步骤的 `thinking` 块中明确且系统地进行推理。

为了成功完成 <user_request>，请展示以下推理模式：
- 分析 <agent_history> 以跟踪朝向 <user_request> 的进度和上下文。
- 分析 <agent_history> 中最新的"下一个目标"和"动作结果"，并清楚说明你之前试图实现什么。
- 分析 <agent_history>、<browser_state>、<read_state>、<file_system>、<read_state> 和截图中的所有相关项目，以理解你的状态。
- 明确判断上一个动作的成功/失败/不确定性。
- 如果 todo.md 为空且任务是多步骤的，使用文件工具在 todo.md 中生成分步计划。
- 分析 `todo.md` 来指导和跟踪你的进度。
- 如果任何 todo.md 项目已完成，在文件中标记为完成。
- 分析你是否陷入困境，例如当你重复相同的动作多次而没有进展时。然后考虑替代方法，例如滚动获取更多上下文，或使用 send_keys 直接与键盘交互，或访问不同页面。
- 分析 <read_state>，其中由于你之前的动作而显示一次性信息。推理你是否想要将这些信息保留在记忆中，并计划在适用时使用文件工具将它们写入文件。
- 如果你看到与 <user_request> 相关的信息，计划将信息保存到文件中。
- 在将数据写入文件之前，分析 <file_system> 并检查文件是否已有内容以避免覆盖。
- 决定应该存储什么简洁、可操作的上下文到记忆中，以指导未来的推理。
- 准备完成时，说明你准备调用 done 并向用户传达完成/结果。
- 在 done 之前，使用 read_file 验证打算输出给用户的文件内容。
- 始终推理 <user_request>。确保仔细分析所需的具体步骤和信息。例如，特定过滤器、特定表单字段、要搜索的特定信息。确保始终将当前轨迹与用户请求进行比较，并仔细思考这是否是用户要求的方式。
</reasoning_rules>
```

#### **强制输出格式**

```json
{
  "thinking": "让我分析一下当前情况：\n\n1. 历史回顾：我已经访问了开关列表页面，看到了10个开关\n2. 现状分析：当前在第1个开关，需要点击[33]分布按钮查看详情\n3. 目标确认：用户要求收集所有开关的分布信息\n4. 行动决策：点击分布按钮，然后提取开关信息，记录到文件中",
  
  "evaluation_previous_goal": "成功访问了开关列表页面，找到了目标元素",
  
  "memory": "已找到10个开关，当前处理第1个开关，页面状态正常",
  
  "next_goal": "点击第1个开关的分布按钮，查看其详细信息",
  
  "action": [{"click_element_by_index": {"index": 33}}]
}
```

#### **推理步骤详解**

每个 `thinking` 块必须包含以下推理步骤：

1. **历史回顾** - 分析之前的动作和结果
2. **现状分析** - 理解当前状态和上下文  
3. **目标确认** - 明确当前要达成的目标
4. **行动决策** - 基于分析决定下一步行动

#### **实际效果**：LLM 必须分析历史、现状、目标，然后才决定行动，大大减少盲目操作

### 3. **结构化输出** - 让 LLM 输出可解析的结果

**问题**：LLM 自由文本输出难以解析，无法自动化执行

**解决方案**：强制输出固定格式的 JSON

```json
{
  "thinking": "分析过程...",
  "evaluation_previous_goal": "成功/失败/不确定",
  "memory": "关键信息记录",
  "next_goal": "下一步要做什么",
  "action": [
    {"click_element_by_index": {"index": 33}},
    {"extract_structured_data": {"query": "开关分布信息", "extract_links": false}}
  ]
}
```

**实际效果**：程序可以直接解析 JSON，自动执行动作，无需人工干预

## 🔄 流程设计 - Agent 如何一步步完成任务

### 核心执行循环
```
获取状态 → LLM分析 → 执行动作 → 验证结果 → 继续循环
```

### 具体执行流程示例

**场景**：收集开关分布信息

```python
# 第1步：获取当前状态
browser_state = {
    "url": "https://switch.alibaba-inc.com/#/switchList",
    "elements": "[33]分布按钮, [34]编辑按钮, [35]删除按钮",
    "content": "开关列表页面，显示10个开关项"
}

# 第2步：LLM分析并决策
llm_response = {
    "thinking": "我看到开关列表页面，有10个开关需要处理。当前在第1个开关，应该点击分布按钮查看详情。",
    "action": [{"click_element_by_index": {"index": 33}}]
}

# 第3步：执行动作
result = await click_element_by_index(index=33)
# 结果：成功点击，页面显示开关详情弹窗

# 第4步：验证结果
if result.success:
    # 继续下一步：提取开关信息
    next_state = await get_browser_state()  # 获取新状态
    # 循环继续...
else:
    # 处理错误，重试或调整策略
```

### 实际执行示例

**任务**：收集开关信息

```
步骤1: 访问开关列表页面
├── 状态：页面加载完成，显示10个开关
├── LLM分析：找到目标页面，开始处理第1个开关
└── 动作：点击第1个开关的分布按钮

步骤2: 查看开关详情
├── 状态：弹窗显示开关分布信息
├── LLM分析：需要提取开关名、机器数、开关值
└── 动作：提取结构化数据

步骤3: 保存信息
├── 状态：已提取到开关信息
├── LLM分析：信息完整，可以保存到文件
└── 动作：写入文件

步骤4: 关闭弹窗，处理下一个
├── 状态：回到开关列表页面
├── LLM分析：第1个开关处理完成，继续第2个
└── 动作：点击第2个开关的分布按钮

... 重复直到所有开关处理完成
```

## 🛡️ 防呆机制 - 让 Agent 不犯低级错误

### 1. **状态检查** - 确保环境正常

**问题**：Agent 可能在页面还没加载完就操作，导致失败

**解决方案**：每步都检查状态

```python
async def click_element_by_index(params, browser_session):
    # 检查1：元素是否存在
    selector_map = await browser_session.get_selector_map()
    if params.index not in selector_map:
        # 强制刷新状态，可能页面刚加载
        await browser_session.get_state_summary()
        selector_map = await browser_session.get_selector_map()
        
        if params.index not in selector_map:
            return ActionResult(
                error=f"元素 {params.index} 不存在",
                success=False
            )
    
    # 检查2：页面是否加载完成
    page = await browser_session.get_current_page()
    try:
        await page.wait_for_load_state(timeout=5000)
    except:
        return ActionResult(error="页面加载超时", success=False)
    
    # 检查3：元素是否可交互
    element = await browser_session.get_dom_element_by_index(params.index)
    if not element.is_clickable():
        return ActionResult(error="元素不可点击", success=False)
    
    # 所有检查通过，执行点击
    await browser_session._click_element_node(element)
    return ActionResult(success=True)
```

### 2. **错误处理** - 失败时自动恢复

**问题**：网络错误、页面变化等导致操作失败

**解决方案**：智能重试和备选策略

```python
async def robust_click_element(index, browser_session, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = await click_element_by_index(index, browser_session)
            if result.success:
                return result
            
            # 失败原因分析
            if "元素不存在" in result.error:
                # 策略1：刷新页面重试
                await browser_session.refresh()
                await asyncio.sleep(2)
            elif "页面导航" in result.error:
                # 策略2：等待页面稳定
                await asyncio.sleep(3)
            else:
                # 策略3：尝试滚动查找元素
                await browser_session.scroll(down=True)
                
        except Exception as e:
            print(f"尝试 {attempt + 1} 失败: {e}")
            if attempt == max_retries - 1:
                return ActionResult(error=f"重试{max_retries}次后仍然失败", success=False)
    
    return ActionResult(error="未知错误", success=False)
```

### 3. **进度跟踪** - 避免重复和遗漏

**问题**：Agent 可能重复操作或遗漏某些步骤

**解决方案**：记录进度和状态

```python
class ProgressTracker:
    def __init__(self):
        self.completed_items = set()
        self.current_step = 0
        self.total_steps = 0
    
    def mark_completed(self, item_id):
        """标记项目已完成"""
        self.completed_items.add(item_id)
        print(f"✅ 完成项目: {item_id}")
    
    def is_completed(self, item_id):
        """检查项目是否已完成"""
        return item_id in self.completed_items
    
    def get_next_item(self, all_items):
        """获取下一个未完成的项目"""
        for item in all_items:
            if not self.is_completed(item.id):
                return item
        return None
    
    def update_progress(self):
        """更新进度"""
        self.current_step += 1
        progress = (self.current_step / self.total_steps) * 100
        print(f"📊 进度: {progress:.1f}% ({self.current_step}/{self.total_steps})")

# 使用示例
tracker = ProgressTracker()
tracker.total_steps = 10

# 在处理开关时
for switch in switches:
    if tracker.is_completed(switch.id):
        continue  # 跳过已完成的
    
    # 处理开关...
    tracker.mark_completed(switch.id)
    tracker.update_progress()
```

### 4. **约束检查** - 确保操作安全

**问题**：Agent 可能执行危险或无效的操作

**解决方案**：在操作前验证约束

```python
class SafetyChecker:
    def __init__(self):
        self.allowed_domains = ['switch.alibaba-inc.com']
        self.max_steps = 20
        self.sensitive_actions = ['delete', 'remove', 'drop']
    
    def check_domain_safety(self, url):
        """检查域名是否安全"""
        for domain in self.allowed_domains:
            if domain in url:
                return True
        return False
    
    def check_action_safety(self, action_name, params):
        """检查动作是否安全"""
        # 检查敏感操作
        if any(sensitive in action_name.lower() for sensitive in self.sensitive_actions):
            print(f"⚠️ 警告：检测到敏感操作 {action_name}")
            return False
        
        # 检查参数有效性
        if 'index' in params and params['index'] < 0:
            print(f"❌ 错误：无效的元素索引 {params['index']}")
            return False
        
        return True
    
    def check_step_limit(self, current_step):
        """检查步数限制"""
        if current_step >= self.max_steps:
            print(f"⏰ 达到最大步数限制 {self.max_steps}")
            return False
        return True

# 使用示例
safety = SafetyChecker()

# 在执行动作前检查
if not safety.check_domain_safety(current_url):
    return ActionResult(error="不允许访问此域名", success=False)

if not safety.check_action_safety(action_name, action_params):
    return ActionResult(error="操作被安全策略阻止", success=False)

if not safety.check_step_limit(current_step):
    return ActionResult(error="达到步数限制", success=False)
```

## 🚀 关键技巧 - 让 Agent 更智能

### 1. **多模态输入** - 让 Agent "看见"页面

**问题**：纯文本描述可能不够准确，Agent 可能误解页面布局

**解决方案**：结合截图和文本，让 Agent 真正"看见"页面

```python
def build_multimodal_message(browser_state, task):
    """构建多模态消息"""
    
    # 文本信息
    text_content = f"""
    <browser_state>
    当前URL: {browser_state.url}
    页面内容: {browser_state.page_content[:500]}...
    可交互元素: {browser_state.selector_map}
    </browser_state>
    """
    
    # 多模态内容列表
    content_parts = [
        ContentPartTextParam(text=text_content)
    ]
    
    # 添加截图（如果有）
    if browser_state.screenshot:
        content_parts.append(ContentPartTextParam(text="当前页面截图:"))
        content_parts.append(ContentPartImageParam(
            image_url=ImageURL(url=f"data:image/png;base64,{browser_state.screenshot}")
        ))
    
    return [
        SystemMessage(content=system_prompt),
        UserMessage(content=content_parts)
    ]

# 实际效果对比
# 纯文本：Agent 看到 "按钮在页面右侧"
# 多模态：Agent 看到截图，知道按钮在右上角，有红色边框
```

### 2. **记忆管理** - 让 Agent 记住重要信息

**问题**：Agent 可能忘记之前做了什么，重复操作或遗漏步骤

**解决方案**：智能记忆系统

```python
class SmartMemory:
    def __init__(self, max_items=20):
        self.short_term = []  # 短期记忆（最近几步）
        self.long_term = {}   # 长期记忆（重要信息）
        self.max_items = max_items
    
    def add_step(self, step_info):
        """添加步骤到记忆"""
        self.short_term.append(step_info)
        
        # 保持短期记忆在合理范围内
        if len(self.short_term) > self.max_items:
            # 压缩记忆：保留最重要的信息
            self.compress_memory()
    
    def remember_important(self, key, value):
        """记住重要信息"""
        self.long_term[key] = value
        print(f"💾 记住重要信息: {key} = {value}")
    
    def get_context(self, current_task):
        """获取相关上下文"""
        relevant_info = []
        
        # 从短期记忆中找相关步骤
        for step in self.short_term[-5:]:  # 最近5步
            if self.is_relevant(step, current_task):
                relevant_info.append(step)
        
        # 从长期记忆中找重要信息
        for key, value in self.long_term.items():
            if self.is_relevant_to_task(key, current_task):
                relevant_info.append(f"{key}: {value}")
        
        return "\n".join(relevant_info)
    
    def is_relevant(self, step, task):
        """判断步骤是否与任务相关"""
        # 简单的关键词匹配
        task_words = set(task.lower().split())
        step_words = set(str(step).lower().split())
        return bool(task_words & step_words)

# 使用示例
memory = SmartMemory()

# 在处理开关时
memory.add_step("点击了第1个开关的分布按钮")
memory.remember_important("开关总数", "10")
memory.remember_important("已处理开关", ["开关1", "开关2"])

# 在下一步中获取上下文
context = memory.get_context("继续处理开关")
# 输出：点击了第1个开关的分布按钮\n开关总数: 10\n已处理开关: ['开关1', '开关2']
```

### 3. **动作序列** - 让 Agent 执行复杂操作

**问题**：某些任务需要多个连续动作，但中间状态变化可能打断序列

**解决方案**：智能动作序列管理

```python
class ActionSequence:
    def __init__(self):
        self.sequence = []
        self.current_index = 0
        self.interrupted = False
    
    def add_actions(self, actions):
        """添加动作序列"""
        self.sequence.extend(actions)
    
    async def execute_sequence(self, browser_session):
        """执行动作序列"""
        results = []
        
        for i, action in enumerate(self.sequence[self.current_index:], self.current_index):
            try:
                # 执行单个动作
                result = await self.execute_action(action, browser_session)
                results.append(result)
                
                # 检查是否成功
                if not result.success:
                    print(f"❌ 动作 {i} 失败: {result.error}")
                    self.current_index = i
                    return results
                
                # 检查页面是否发生变化
                if self.page_changed(browser_session):
                    print(f"🔄 页面发生变化，序列中断")
                    self.current_index = i + 1
                    self.interrupted = True
                    return results
                
                self.current_index = i + 1
                
            except Exception as e:
                print(f"❌ 执行动作时出错: {e}")
                self.current_index = i
                return results
        
        return results
    
    async def execute_action(self, action, browser_session):
        """执行单个动作"""
        action_name = list(action.keys())[0]
        params = action[action_name]
        
        if action_name == "click_element_by_index":
            return await click_element_by_index(params, browser_session)
        elif action_name == "extract_structured_data":
            return await extract_structured_data(params, browser_session)
        # ... 其他动作
    
    def page_changed(self, browser_session):
        """检查页面是否发生变化"""
        # 检查URL、页面内容等是否发生变化
        return False  # 简化示例

# 使用示例
sequence = ActionSequence()
sequence.add_actions([
    {"click_element_by_index": {"index": 33}},
    {"extract_structured_data": {"query": "开关信息", "extract_links": False}},
    {"write_file": {"file_name": "switch_info.txt", "content": "..."}}
])

# 执行序列
results = await sequence.execute_sequence(browser_session)

# 如果序列被中断，可以继续执行
if sequence.interrupted:
    print("继续执行剩余动作...")
    remaining_results = await sequence.execute_sequence(browser_session)
```

### 4. **结果验证** - 让 Agent 确保操作成功

**问题**：Agent 可能认为操作成功，但实际上失败了

**解决方案**：多维度验证结果

```python
class ResultValidator:
    def __init__(self):
        self.validation_rules = {}
    
    def validate_click_result(self, action_result, expected_changes):
        """验证点击操作结果"""
        validations = []
        
        # 验证1：动作本身是否成功
        if not action_result.success:
            validations.append("❌ 点击动作失败")
            return False
        
        # 验证2：页面是否发生预期变化
        if "弹窗出现" in expected_changes:
            if not self.check_popup_appeared():
                validations.append("❌ 预期弹窗未出现")
                return False
        
        # 验证3：URL是否发生变化
        if "URL变化" in expected_changes:
            if not self.check_url_changed():
                validations.append("❌ URL未按预期变化")
                return False
        
        # 验证4：元素是否消失/出现
        if "元素消失" in expected_changes:
            if not self.check_element_disappeared():
                validations.append("❌ 预期元素未消失")
                return False
        
        print("✅ 所有验证通过")
        return True
    
    def validate_data_extraction(self, extracted_data, expected_fields):
        """验证数据提取结果"""
        if not extracted_data:
            print("❌ 未提取到任何数据")
            return False
        
        # 检查必需字段
        for field in expected_fields:
            if field not in extracted_data:
                print(f"❌ 缺少必需字段: {field}")
                return False
        
        # 检查数据质量
        for field, value in extracted_data.items():
            if not value or value.strip() == "":
                print(f"❌ 字段 {field} 为空")
                return False
        
        print("✅ 数据提取验证通过")
        return True
    
    def validate_file_write(self, file_path, expected_content):
        """验证文件写入结果"""
        if not os.path.exists(file_path):
            print(f"❌ 文件 {file_path} 未创建")
            return False
        
        with open(file_path, 'r') as f:
            content = f.read()
        
        if not content:
            print(f"❌ 文件 {file_path} 为空")
            return False
        
        if expected_content and expected_content not in content:
            print(f"❌ 文件内容不符合预期")
            return False
        
        print(f"✅ 文件写入验证通过: {file_path}")
        return True

# 使用示例
validator = ResultValidator()

# 验证点击结果
click_success = validator.validate_click_result(
    action_result,
    expected_changes=["弹窗出现", "元素消失"]
)

# 验证数据提取
data_valid = validator.validate_data_extraction(
    extracted_data,
    expected_fields=["开关名", "机器数", "开关值"]
)

# 验证文件写入
file_valid = validator.validate_file_write(
    "switch_info.txt",
    expected_content="开关名||机器数||开关值"
)
```

## 🔗 LLM 集成策略

### 1. **选择合适的 LLM 库**
根据项目需求选择最适合的 LLM 库：

```python
# OpenAI 官方库
from openai import OpenAI
client = OpenAI(api_key="your-key")

# Anthropic 官方库
from anthropic import Anthropic
client = Anthropic(api_key="your-key")

# LangChain 生态
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o")

# 自定义实现
class CustomLLM:
    def __init__(self, model_name: str):
        self.model_name = model_name
    
    async def ainvoke(self, messages, output_format=None):
        # 实现你的 LLM 调用逻辑
        response = await your_llm_call(messages)
        return response
```

### 2. **统一接口设计**
设计统一的 LLM 接口，便于切换不同提供商：

```python
class LLMInterface:
    """统一的 LLM 接口"""
    
    async def ainvoke(self, messages, output_format=None):
        raise NotImplementedError
    
    @property
    def model(self) -> str:
        raise NotImplementedError
    
    @property
    def provider(self) -> str:
        raise NotImplementedError

class OpenAIAdapter(LLMInterface):
    def __init__(self, client: OpenAI):
        self.client = client
    
    async def ainvoke(self, messages, output_format=None):
        # 适配 OpenAI 调用
        pass

class AnthropicAdapter(LLMInterface):
    def __init__(self, client: Anthropic):
        self.client = client
    
    async def ainvoke(self, messages, output_format=None):
        # 适配 Anthropic 调用
        pass
```

### 3. **消息格式标准化**
定义统一的消息格式，支持多模态内容：

```python
from dataclasses import dataclass
from typing import Union, List

@dataclass
class Message:
    role: str  # 'user', 'assistant', 'system'
    content: Union[str, List[dict]]  # 支持文本和多模态
    name: str = None

@dataclass
class ImageContent:
    type: str = 'image_url'
    image_url: dict

@dataclass
class TextContent:
    type: str = 'text'
    text: str
```

## 📝 实际应用案例

### 案例1：网页自动化 Agent 完整实现

#### 核心架构设计
```python
class WebAgent:
    def __init__(self, llm, browser_session):
        self.llm = llm
        self.browser_session = browser_session
        self.memory = []
        self.state = AgentState()
    
    async def run(self, task):
        while not self.is_done():
            # 1. 获取当前状态
            browser_state = await self.browser_session.get_state_summary()
            
            # 2. 构建输入消息
            messages = self.build_messages(browser_state, task)
            
            # 3. LLM 分析决策
            response = await self.llm.ainvoke(messages, self.AgentOutput)
            
            # 4. 执行动作
            result = await self.execute_actions(response.action)
            
            # 5. 更新记忆和状态
            self.update_memory(browser_state, response, result)
```

#### 状态管理实现
```python
@dataclass
class AgentState:
    """Agent 状态管理"""
    n_steps: int = 0
    last_model_output: Optional[AgentOutput] = None
    last_result: List[ActionResult] = field(default_factory=list)
    paused: bool = False
    stopped: bool = False

class BrowserStateSummary:
    """浏览器状态摘要"""
    url: str
    tabs: List[str]
    selector_map: Dict[int, str]  # 可交互元素映射
    screenshot: Optional[str] = None  # base64 截图
    page_content: str  # 页面文本内容
```

### 案例2：智能 Prompt 设计

#### 分层信息架构
```xml
<!-- 系统提示词模板 -->
<system>
你是一个 AI agent，设计用于在迭代循环中自动化浏览器任务。

<intro>
你擅长以下任务：
1. 导航复杂网站并提取精确信息
2. 自动化表单提交和交互操作
3. 收集和保存信息
4. 有效使用文件系统管理上下文
5. 在 agent 循环中高效操作
</intro>

<input>
每一步的输入包含：
1. <agent_history>: 包含之前动作和结果的时序事件流
2. <agent_state>: 当前用户请求、文件系统摘要、todo内容、步骤信息
3. <browser_state>: 当前URL、打开的标签页、可交互元素、可见页面内容
4. <browser_vision>: 带有边界框的浏览器截图
5. <read_state>: 仅在前一个动作是数据提取时显示
</input>

<reasoning_rules>
你必须明确且系统地在每个步骤的 `thinking` 块中进行推理：
- 分析 <agent_history> 以跟踪进度和上下文
- 评估最近动作的成功/失败/不确定性
- 分析所有相关项目以理解当前状态
- 如果任务多步骤且 todo.md 为空，生成分步计划
- 决定应该存储什么简洁、可操作的上下文
</reasoning_rules>

<output>
你必须始终以有效的 JSON 格式响应：
{
  "thinking": "结构化的推理块",
  "evaluation_previous_goal": "对上一个动作的一句话分析",
  "memory": "1-3句话的具体记忆",
  "next_goal": "下一个目标的一句话描述",
  "action": [{"action_name": {"params": "values"}}]
}
</output>
</system>
```

### 案例3：动作系统设计

#### 动作注册和执行
```python
class Controller:
    def __init__(self):
        self.registry = ActionRegistry()
        self._register_default_actions()
    
    def _register_default_actions(self):
        # 浏览器导航动作
        @self.registry.action('Navigate to URL')
        async def go_to_url(params: GoToUrlAction, browser_session):
            await browser_session.navigate(params.url, params.new_tab)
            return ActionResult(
                extracted_content=f"Navigated to {params.url}",
                include_in_memory=True
            )
        
        # 元素交互动作
        @self.registry.action('Click element by index')
        async def click_element(params: ClickElementAction, browser_session):
            element = await browser_session.get_dom_element_by_index(params.index)
            await browser_session._click_element_node(element)
            return ActionResult(
                extracted_content=f"Clicked element {params.index}",
                include_in_memory=True
            )
        
        # 文件系统动作
        @self.registry.action('Write content to file')
        async def write_file(file_name: str, content: str, file_system):
            await file_system.write_file(file_name, content)
            return ActionResult(
                extracted_content=f"Wrote content to {file_name}",
                include_in_memory=True
            )
```

#### 动作参数模型
```python
@dataclass
class ClickElementAction:
    index: int = Field(description="Element index to click")
    
@dataclass
class InputTextAction:
    index: int = Field(description="Element index to input text")
    text: str = Field(description="Text to input")
    
@dataclass
class GoToUrlAction:
    url: str = Field(description="URL to navigate to")
    new_tab: bool = Field(default=False, description="Open in new tab")
```

### 案例4：错误处理和防呆机制

#### 状态验证和重试
```python
async def click_element_by_index(params: ClickElementAction, browser_session):
    # 1. 状态检查
    selector_map = await browser_session.get_selector_map()
    if params.index not in selector_map:
        # 强制刷新状态
        await browser_session.get_state_summary(cache_clickable_elements_hashes=True)
        selector_map = await browser_session.get_selector_map()
        
        if params.index not in selector_map:
            return ActionResult(
                error=f"Element {params.index} does not exist",
                success=False
            )
    
    # 2. 执行动作
    try:
        element_node = await browser_session.get_dom_element_by_index(params.index)
        await browser_session._click_element_node(element_node)
        return ActionResult(
            extracted_content=f"Clicked element {params.index}",
            include_in_memory=True
        )
    except Exception as e:
        # 3. 错误处理
        if 'Execution context was destroyed' in str(e):
            # 页面导航导致上下文销毁，刷新状态
            await browser_session.get_state_summary()
            return ActionResult(
                error="Page navigated during click. Refreshed state provided.",
                success=False
            )
        else:
            return ActionResult(error=str(e), success=False)
```

### 案例5：多模态输入处理

#### 视觉和文本信息结合
```python
def build_messages(self, browser_state, task):
    """构建包含多模态信息的消息"""
    
    # 文本状态信息
    state_description = f"""
    <browser_state>
    Current URL: {browser_state.url}
    Interactive Elements: {self.format_elements(browser_state.selector_map)}
    Page Content: {browser_state.page_content[:1000]}...
    </browser_state>
    """
    
    # 多模态内容
    content_parts = [ContentPartTextParam(text=state_description)]
    
    # 添加视觉信息（截图）
    if browser_state.screenshot:
        content_parts.append(ContentPartTextParam(text="Current screenshot:"))
        content_parts.append(ContentPartImageParam(
            image_url=ImageURL(url=f"data:image/png;base64,{browser_state.screenshot}")
        ))
    
    return [
        SystemMessage(content=self.system_prompt),
        UserMessage(content=content_parts)
    ]
```

### 案例6：记忆管理和上下文优化

#### 智能记忆系统
```python
class MemoryManager:
    def __init__(self, max_history_items=40):
        self.max_history_items = max_history_items
        self.history = []
    
    def add_step(self, step_info: StepInfo):
        """添加步骤信息到记忆"""
        self.history.append(step_info)
        
        # 保持记忆在合理范围内
        if len(self.history) > self.max_history_items:
            # 保留最重要的信息
            self.history = self._compress_history()
    
    def get_relevant_context(self, current_task: str) -> str:
        """获取与当前任务相关的上下文"""
        relevant_steps = []
        
        for step in self.history[-10:]:  # 最近10步
            if self._is_relevant(step, current_task):
                relevant_steps.append(step)
        
        return self._format_context(relevant_steps)
    
    def _is_relevant(self, step: StepInfo, task: str) -> bool:
        """判断步骤是否与当前任务相关"""
        # 基于关键词匹配或语义相似度
        task_keywords = self._extract_keywords(task)
        step_keywords = self._extract_keywords(step.description)
        return bool(set(task_keywords) & set(step_keywords))
```

## 🎯 最佳实践总结

### 核心原则
1. **信息分层** - 清晰的信息架构
2. **强制推理** - 结构化思考过程
3. **状态管理** - 完整的上下文跟踪
4. **错误处理** - 健壮的容错机制
5. **结果验证** - 每步质量检查
6. **记忆优化** - 避免重复工作

### LLM 集成建议
1. **选择合适的库** - 根据项目需求选择最适合的 LLM 库
2. **统一接口设计** - 设计适配器模式，便于切换不同提供商
3. **消息格式标准化** - 定义统一的消息格式，支持多模态内容
4. **性能优化** - 减少不必要的格式转换，提高调用效率

### 案例7：登录状态管理

#### 多种登录策略实现
```python
class LoginManager:
    """登录状态管理器"""
    
    async def method1_manual_login(self):
        """方法1: 手动登录并保存 cookies"""
        browser_session = BrowserSession(
            browser_profile=BrowserProfile(
                headless=False,  # 显示浏览器，方便手动登录
                user_data_dir='~/.config/browseruse/profiles/manual_login',
                storage_state='./saved_cookies.json'  # 保存 cookies 到文件
            )
        )
        
        await browser_session.start()
        print("请手动登录到目标网站，完成后按 Enter 继续...")
        input()
        
        # 保存 cookies
        await browser_session.save_storage_state()
        return browser_session
    
    async def method2_load_saved_cookies(self):
        """方法2: 加载已保存的 cookies"""
        browser_session = BrowserSession(
            browser_profile=BrowserProfile(
                headless=True,
                storage_state='./saved_cookies.json'  # 加载 cookies
            )
        )
        return browser_session
    
    async def method3_sensitive_data(self):
        """方法3: 使用敏感数据处理登录"""
        sensitive_data = {
            'https://example.com': {
                'username': 'your_username',
                'password': 'your_password'
            }
        }
        
        browser_session = BrowserSession(
            browser_profile=BrowserProfile(
                headless=True,
                allowed_domains=['https://example.com']
            )
        )
        
        # 在 Agent 中使用敏感数据
        agent = Agent(
            task="使用提供的凭据登录到网站",
            llm=self.llm,
            browser_session=browser_session,
            sensitive_data=sensitive_data
        )
        return agent
```

### 案例8：任务执行流程

#### 完整任务执行示例
```python
async def execute_switch_analysis_task():
    """执行开关分析任务的完整流程"""
    
    # 1. 初始化组件
    llm = ChatOpenAI(model="gpt-4o")
    browser_session = BrowserSession(
        browser_profile=BrowserProfile(
            headless=True,
            storage_state='./saved_cookies.json'
        )
    )
    
    # 2. 创建 Agent
    agent = Agent(
        task="访问链接https://switch.alibaba-inc.com/#/switchList?appName=ace-guide-delivery-s&envGroup=aeOnlineGroup&unit=rg-us-east。逐个点击每个开关的分布按钮，查看开关名、开关值和机器数，并记录下来，然后关闭抽屉继续处理下一个开关。先针对第一页的所有开关处理，不需要做翻页。内容写到switch_info.txt文件中，格式是开关名||机器数||开关值，对于有多个值的开关写成多行。",
        llm=llm,
        browser_session=browser_session,
        file_system_path="/path/to/workspace",
        save_conversation_path="./conversations"
    )
    
    # 3. 执行任务
    history = await agent.run()
    
    # 4. 处理结果
    if history.success:
        print("✅ 任务完成!")
        # 读取生成的文件
        with open("switch_info.txt", "r") as f:
            switch_data = f.read()
        print(f"收集到的开关信息:\n{switch_data}")
    else:
        print("❌ 任务未完成")
    
    return history
```

### 案例9：调试和监控

#### 详细日志记录
```python
class DebugLLM:
    """带调试功能的 LLM 包装器"""
    
    def __init__(self, llm):
        self.llm = llm
    
    async def ainvoke(self, messages, output_format=None):
        # 记录输入
        print("🔍 LLM 输入:")
        for i, msg in enumerate(messages):
            print(f"消息 {i}: {msg.content[:200]}...")
        
        # 调用原始 LLM
        response = await self.llm.ainvoke(messages, output_format)
        
        # 记录输出
        print("🤖 LLM 输出:")
        print(f"思考: {response.thinking[:200]}...")
        print(f"动作: {response.action}")
        
        return response

# 使用调试 LLM
debug_llm = DebugLLM(ChatOpenAI(model="gpt-4o"))
agent = Agent(task="...", llm=debug_llm, ...)
```

### 案例10：文件系统管理

#### 智能文件操作
```python
class FileManager:
    """文件系统管理器"""
    
    def __init__(self, base_path: str):
        self.base_path = Path(base_path)
        self.todo_file = self.base_path / "todo.md"
        self.results_file = self.base_path / "results.md"
    
    async def initialize_todo(self, task: str):
        """初始化任务清单"""
        if self.task_is_multi_step(task):
            todo_content = self.generate_todo_plan(task)
            await self.write_file("todo.md", todo_content)
    
    async def update_progress(self, completed_item: str):
        """更新进度"""
        if self.todo_file.exists():
            content = await self.read_file("todo.md")
            # 标记完成的项目
            updated_content = self.mark_completed(content, completed_item)
            await self.write_file("todo.md", updated_content)
    
    async def save_results(self, data: str, format: str = "txt"):
        """保存结果数据"""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"results_{timestamp}.{format}"
        await self.write_file(filename, data)
        return filename
    
    def task_is_multi_step(self, task: str) -> bool:
        """判断是否为多步骤任务"""
        multi_step_keywords = ["逐个", "分别", "依次", "列表", "多个"]
        return any(keyword in task for keyword in multi_step_keywords)
```

### 性能优化

#### 1. 模型选择策略
```python
class ModelSelector:
    """智能模型选择器"""
    
    def select_model(self, task_complexity: str, budget: str) -> str:
        if task_complexity == "simple" and budget == "low":
            return "gpt-3.5-turbo"
        elif task_complexity == "complex" and budget == "high":
            return "gpt-4o"
        else:
            return "gpt-4o-mini"
    
    def estimate_cost(self, task: str) -> float:
        """估算任务成本"""
        # 基于任务长度和复杂度估算 token 数量
        estimated_tokens = len(task) * 2  # 粗略估算
        return estimated_tokens * 0.00001  # 假设每 token 0.00001 美元
```

#### 2. 缓存机制
```python
class LLMCache:
    """LLM 调用缓存"""
    
    def __init__(self):
        self.cache = {}
    
    async def cached_invoke(self, llm, messages, cache_key: str):
        """带缓存的 LLM 调用"""
        if cache_key in self.cache:
            print("📦 使用缓存结果")
            return self.cache[cache_key]
        
        result = await llm.ainvoke(messages)
        self.cache[cache_key] = result
        return result
    
    def generate_cache_key(self, messages) -> str:
        """生成缓存键"""
        content_hash = hashlib.md5(
            str(messages).encode()
        ).hexdigest()
        return f"llm_cache_{content_hash}"
```

#### 3. 资源管理
```python
class ResourceManager:
    """资源管理器"""
    
    def __init__(self):
        self.browser_sessions = []
        self.llm_instances = []
    
    async def cleanup(self):
        """清理所有资源"""
        # 关闭浏览器会话
        for session in self.browser_sessions:
            await session.stop()
        
        # 清理 LLM 实例
        for llm in self.llm_instances:
            if hasattr(llm, 'close'):
                await llm.close()
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.cleanup()

# 使用资源管理器
async with ResourceManager() as rm:
    browser_session = BrowserSession()
    rm.browser_sessions.append(browser_session)
    
    agent = Agent(task="...", browser_session=browser_session)
    await agent.run()
```

这样设计出来的 Agent 既能理解复杂任务，又能稳定执行，还能自我纠错，同时具备良好的扩展性和兼容性。核心价值在于学习优秀的设计模式，而不是依赖特定的实现框架。 