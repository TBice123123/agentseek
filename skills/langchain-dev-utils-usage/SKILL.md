---
name: langchain-dev-utils-usage
description: Expert in using langchain-dev-utils library for LangChain/LangGraph development. Use when working with chat/embedding model management, agent creation, middleware configuration, tool calling, message processing, and state graph building.
---
# langchain-dev-utils Usage Guide

## Quick Start

```python
from langchain.tools import tool
from langchain_core.messages import HumanMessage
from langchain_dev_utils.chat_models import register_model_provider, load_chat_model
from langchain_dev_utils.agents import create_agent

# Register model provider (OpenAI-compatible)
register_model_provider("vllm", "openai-compatible", base_url="http://localhost:8000/v1")

@tool
def get_current_weather(location: str) -> str:
    """Get the current weather for a specified location"""
    return f"25 degrees, {location}"

# Dynamically load model using string identifier
model = load_chat_model("vllm:qwen2.5-7b")
response = model.invoke("Hello")

# Create an agent with tools
agent = create_agent("vllm:qwen2.5-7b", tools=[get_current_weather])
response = agent.invoke({"messages": [HumanMessage(content="What's the weather like in New York today?")]})
```

## Core Modules

- **Model Management**: Unified provider registry for chat models and embeddings. Switch models via string identifiers (e.g., `"vllm:qwen2.5-7b"`). Key APIs: `register_model_provider`, `load_chat_model`, `register_embeddings_provider`, `load_embeddings`
- **OpenAI-Compatible Integration**: Built-in adapter for OpenAI-compatible APIs (vLLM, TGI, etc.). Configure base URL and model name explicitly. Key APIs: `create_openai_compatible_model`, `create_openai_compatible_embedding`
- **Message Processing**: Handle chain-of-thought concatenation, streaming responses, and message formatting. Key APIs: `format_sequence`
- **Tool Calling**: Detect tool calls, parse arguments, and implement human-in-the-loop review workflows. Key APIs: `has_tool_calling`, `parse_tool_calling`, `human_in_the_loop`
- **Agent Development**: Provides utility functions to wrap sub-agents as tools, enabling multi-agent systems under a subagents architecture: `wrap_agent_as_tool`, `wrap_all_agents_as_tool`
- **Middleware**: Provides commonly used utility middleware for planning, routing, handoffs, tool calling repair, and prompt formatting. Key APIs: `PlanMiddleware`, `ModelRouterMiddleware`, `HandoffAgentMiddleware`, `ToolCallRepairMiddleware`, `FormatPromptMiddleware`
- **State Graph Building**: Pre-built functions for constructing sequential and parallel state graphs in LangGraph. Key APIs: `create_sequential_graph`, `create_parallel_graph`

## Model Management Best Practices

Choose the appropriate approach based on your scenario:
1. If all model providers are officially supported by `init_chat_model`, use the official function directly for best compatibility.
2. If some providers are not officially supported, use `register_model_provider` to register them, then use `load_chat_model` to load.
3. If the provider offers an OpenAI-compatible API (e.g., vLLM), register with `chat_model="openai-compatible"` and use `load_chat_model`.

## Key Patterns

### OpenAI-Compatible Integration

```python
from langchain_dev_utils.chat_models.adapters import create_openai_compatible_model

ChatVLLM = create_openai_compatible_model(
    model_provider="vllm",
    base_url="http://localhost:8000/v1",
    chat_model_cls_name="ChatVLLM",
)
model = ChatVLLM(model="qwen2.5-7b")
```

### Middleware Usage

**PlanMiddleware** - Task planning, decompose complex tasks into ordered sub-tasks:

```python
from langchain_dev_utils.agents import create_agent
from langchain_dev_utils.agents.middleware import PlanMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    middleware=[
        PlanMiddleware(
            custom_plan_tool_descriptions={
                "write_plan": "用于写计划，将任务拆解为多个有序的子任务。",
                "finish_sub_plan": "用于完成子任务，更新子任务状态为已完成。",
                "read_plan": "用于查询当前的任务规划列表。"
            },
            use_read_plan_tool=True,
        )
    ],
)
```

**ModelRouterMiddleware** - Dynamically route to the most suitable model based on input:

```python
from langchain_dev_utils.agents.middleware import ModelRouterMiddleware

agent = create_agent(
    model="vllm:qwen2.5-7b",  # Placeholder, actual model selected by middleware
    tools=[get_current_time],
    middleware=[
        ModelRouterMiddleware(
            router_model="vllm:qwen2.5-7b",
            model_list=model_list,
        )
    ],
)
```

**HandoffAgentMiddleware** - Flexibly hand off tasks between multiple sub-agents:

```python
from langchain_dev_utils.agents.middleware import HandoffAgentMiddleware

agent = create_agent(
    model="vllm:qwen2.5-7b",
    middleware=[HandoffAgentMiddleware(agents_config=agent_config)],
)
```

**ToolCallRepairMiddleware** - Automatically repair invalid tool calls from LLMs:

```python
from langchain_dev_utils.agents.middleware import ToolCallRepairMiddleware

agent = create_agent(
    model="vllm:qwen2.5-7b",
    tools=[run_python_code, get_current_time],
    middleware=[ToolCallRepairMiddleware()],
)
```

### Multi-Agent with wrap_all_agents_as_tool

Wrap multiple agents into a single tool for orchestration by a main agent:

```python
from langchain_dev_utils.agents import wrap_all_agents_as_tool, create_agent

call_subagent_tool = wrap_all_agents_as_tool(
    [calendar_agent, email_agent],
    tool_name="call_subagent",
    tool_description=(
        "调用子智能体执行任务。"
        "可以使用的智能体是："
        "- calendar_agent：用于安排日历事件"
        "- email_agent：用于发送电子邮件"
    ),
)

main_agent = create_agent(
    "vllm:qwen2.5-7b",
    tools=[call_subagent_tool],
    system_prompt=(
        "你是一个有用的个人助手。"
        "你可以使用**call_subagent**工具调用子智能体执行任务。"
    ),
)
```

### Sequential/Parallel State Graph Building

```python
from langchain_dev_utils.graph import create_sequential_graph, create_parallel_graph

sequential_graph = create_sequential_graph(
    nodes=[step1, step2, step3], state_schema=AgentState
)
parallel_graph = create_parallel_graph(
    nodes=[task1, task2], state_schema=AgentState
)
```

## Usage with Context7 MCP Server
Use the `context7` mcp server:
1. Use `resolve-library-id` to identify the library.
2. Use `query-docs` to fetch documentation.
3. **Selection:** If multiple docs appear, choose the one with the latest update date and highest confidence.