

## LangChain Tool Calling

LangChain Tool Calling（工具调用） 是让大语言模型（LLM）连接并执行外部功能（如搜索、计算、API、数据库）的核心机制，是构建智能 Agent（智能体） 的基础。它让模型从 “纯文本生成” 升级为 “能思考、会动手” 的系统。

#### 核心概念

- Tool（工具）：封装了特定功能的可执行单元，包含：名称、描述、参数 Schema、执行逻辑。

- Tool Calling（工具调用）：模型自主判断是否需要工具 → 生成结构化调用指令 → 框架执行 → 结果返回模型 → 生成最终回答。

#### 关键 API（LangChain 1.0）

- **`@tool`**
  - 自动从函数签名、类型注解、docstring 生成 **JSON Schema**。
  - 函数必须有清晰的 docstring（模型靠它理解工具用途）。
- **`model.bind_tools(tools)`**
  - 把工具列表传给模型，让模型 “知道” 可用工具。
  - 支持：函数、Pydantic、BaseTool、OpenAI 格式字典。
- **`AIMessage.tool_calls`**
  - 标准化结构：`[{"id": "...", "name": "...", "args": {...}}]`。
  - 跨模型（OpenAI、Anthropic、Gemini、通义千问）统一格式。
- **`ToolMessage`**
  - 工具结果必须用 `ToolMessage` 返回，包含 `tool_call_id`。

##### 关键要点

| 必需项         | 说明                                 |
| -------------- | ------------------------------------ |
| `@tool` 装饰器 | 声明这是一个工具                     |
| **docstring**  | AI 读这个来理解工具用途 ⚠️ 非常重要！ |
| 类型注解       | 参数和返回值的类型                   |
| 返回 `str`     | 工具应该返回字符串（AI 最容易理解）  |

工具的 docstring: **AI 依赖 docstring 来理解工具！**

```
AIMessage(
    content="",
    tool_calls=[{
        "name": "get_weather",
        "args": {"city": "上海"},
        "type": "tool_call"
    }]
)
```

#### @tool 基本用法

```
@tool
def my_tool(param: str) -> str:
    """
    工具的简短描述（AI 读这个！）

    参数:
        param: 参数说明

    返回:
        返回值说明
    """
    ...


from langchain_core.tools import tool
@tool
def get_weather(city: str) -> str:
    """
    获取指定城市的天气信息

    参数:
        city: 城市名称，如"北京"、"上海"

    返回:
        天气信息字符串
    """
    # 你的实现
    return "晴天，温度 15°C"
```



#### 完整工作流（LangChain 1.0）

1. 定义工具（`@tool`）
2. 绑定工具（`model.bind_tools([...])`）
3. 用户提问 → 模型分析
4. 生成 ToolCall（结构化：`name` + `parameters`）
5. 执行工具（`tool.invoke(args)`）
6. 返回 ToolMessage 给模型
7. 模型整合结果 → 输出自然语言回答







# 问题

### 1. 什么是 LangChain Tool Calling？

Tool Calling 是让 LLM 根据用户问题**自主选择并调用外部工具**的能力，LangChain 对各家模型的函数调用做了**统一封装**，让模型能执行搜索、计算、数据库、API 等外部操作。



### 2. Tool、Function、Tool Calling 三者区别？

- **Tool**：封装好的功能单元（函数 + 描述 + 参数）
- **Function**：工具里具体执行的代码逻辑
- **Tool Calling**：模型决定调用哪个工具、传什么参数的过程



### 3. Tool Calling 工作流程？

1. 定义工具（`@tool`）
2. 把工具描述传给模型
3. 模型输出结构化调用指令（tool name + args）
4. 执行工具
5. 将结果返回给模型，生成最终回答

### 4. Tool Calling 和普通 if-else 函数调用有什么区别？

- if-else：**人工写规则**，只能匹配关键词，不理解意图
- Tool Calling：**LLM 自主理解、自主决策**，支持复杂意图、自动填参、多轮调用



### 5. LangChain Tool Calling 与模型原生 Function Calling 的区别？

- Function Calling：**模型层接口**，每家格式不一样
- Tool Calling：**LangChain 统一封装层**，一套代码适配所有模型，自动解析、异常处理、消息格式对齐



### 6. Tool Calling 和 ReAct 的关系？

- ReAct 是**思考范式**：Thought → Action → Observation
- Tool Calling 是**执行机制**：模型输出结构化调用
- 现在 Agent = **ReAct 逻辑 + Tool Calling 执行**

### 7. 一个好的 Tool 要满足什么？

- 函数描述清晰（模型靠这个判断用不用）
- 参数类型明确
- 功能单一、职责清晰
- 有错误处理
- 输入输出简单稳定

### 8. 模型是怎么知道该调用哪个 Tool 的？

靠 **Tool 的描述（docstring）+ 参数 Schema**，

模型在生成时会**选择最匹配问题的工具**。



### 9. Tool Calling 失败常见原因？

- 工具描述不清楚
- 参数格式不对
- 模型理解错误
- 工具执行报错 / 超时
- 工具太多，模型选错

### 10. Tool Calling 能解决 LLM 哪些痛点？

- 不知道实时信息
- 不会精确计算
- 不能访问内部系统 / 数据库
- 不能调用外部 API
- 只能生成文本，不能 “做事”



### 11. 哪些场景必须用 Tool Calling？

- 智能客服（查订单、改状态）
- 数据分析（读表、计算、画图）
- RAG 检索
- 系统操作（文件、API、数据库）
- 自主 Agent



### 12. 如何让 Tool Calling 更稳定？

- 工具描述精简准确
- 工具数量不要太多
- 加示例（few-shot）
- 参数加格式校验
- 异常捕获 + 重试
- 使用支持 Tool Calling 强的模型

### 13. 多工具并行调用怎么实现？

模型支持并行 tool call → LangChain 解析 → 异步 / 多线程执行 → 汇总结果。



### 14. Tool Calling 和 RAG 有什么区别 / 结合？

- RAG：**查外部知识**
- Tool Calling：**执行外部动作**
- 结合：RAG 可以做成一个 Tool，让 Agent 自主决定什么时候检索。



### 15. Agent 和 Tool Calling 的关系？

- Tool Calling 是**能力**
- Agent 是**具备自主决策 + 多轮执行的系统**
- Agent 必须依赖 Tool Calling 才能完成复杂任务。