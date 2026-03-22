# s05: Agent 系统

`[ s01 > s02 > s03 > s04 > s05 ] s06 > s07 | s08 > s09 > s10 > s11 > s12`

> *"One agent & tools is all you need"* -- 一个代理 + 工具 = 智能交互。
> 
> **智能层**: Agent 系统 -- 赋予 AI 使用工具的能力。

## 问题 (Problem)

在 LLM Provider 抽象基础上，我们只能让 AI 生成文本回复，但无法让它执行实际操作（如读取文件、运行命令、调用 API）。AI 缺乏与外部世界互动的能力，限制了其功能性。我们需要一种机制让 AI 能够使用工具来完成复杂的任务。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| User     |      | AgentScope  |      | Tool Execution   |
| Query    | ---> | (Container) | ---> | Pipeline         |
|          |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |                    |
                      v                    v
              +--------------------+   +--------------+
              | ToolCatalog        |   | agentic_loop |
              | - register_tools() |   | (main loop)  |
              | - find_tool()      |   |              |
              +--------------------+   +--------------+

Tool Definition Pattern:
- name: 工具名称
- description: 功能描述  
- parameters: 参数定义
- func: 执行函数
```

引入 Agent 系统：通过 ToolCatalog 管理可用工具，agentic_loop 实现模型与工具的交互循环。这种设计保持了核心处理管道不变，只是增加了工具执行能力。

## 工作原理 (How It Works)

1. 定义工具结构：

```python
class ToolDefinition:
    """工具定义"""
    def __init__(self, name: str, description: str, parameters: dict, func: callable):
        self.name = name
        self.description = description
        self.parameters = parameters
        self.func = func
```

2. 构建工具目录：

```python
class ToolCatalog:
    def __init__(self):
        self.tools = {}
    
    def register_tool(self, tool_def: ToolDefinition):
        self.tools[tool_def.name] = tool_def
    
    def get_tool(self, name: str) -> ToolDefinition:
        return self.tools.get(name)
    
    def list_tools(self) -> list:
        return list(self.tools.keys())
```

3. 实现内置工具：
- bash: 运行 shell 命令
- read_file: 读取文件内容
- write_file: 写入文件内容
- list_directory: 列出目录内容

4. 构建 Agent Scope 容器：

```python
class AgentScope:
    def __init__(self):
        self.tool_catalog = ToolCatalog()
        self._setup_built_in_tools()
    
    def _setup_built_in_tools(self):
        # 注册内置工具
        self.tool_catalog.register_tool(ToolDefinition(
            name="bash",
            description="Execute shell commands",
            parameters={"command": {"type": "string"}},
            func=self._run_bash
        ))
        # ... 其他内置工具
```

5. 实现 Agent 循环：

```python
async def agentic_loop(query: str, provider_manager: ProviderManager) -> str:
    messages = [{"role": "user", "content": query}]
    
    while True:
        response = await provider_manager.chat_with_default(messages)
        
        if response.get("finish_reason") != "tool_calls":
            # 没有工具调用，返回结果
            return response["content"]
        
        # 处理工具调用
        tool_calls = response.get("tool_calls", [])
        results = []
        
        for call in tool_calls:
            tool_name = call["name"]
            args = call["arguments"]
            
            tool = agent_scope.tool_catalog.get_tool(tool_name)
            if tool:
                result = tool.func(**args)
                results.append({
                    "tool_call_id": call["id"],
                    "result": result
                })
        
        # 将工具结果加入消息历史
        messages.append({"role": "tool_results", "content": results})
```

## 变化 (What Changed)

相比 s04，在不改变原有 HTTP 服务器、路由、渠道和提供商架构的基础上，新增了 Agent 系统。现在 AI 不仅可以生成文本，还能通过工具与外部环境交互。Agent 循环实现了持续的工具使用直到任务完成。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s05_agent_system.py
```

测试工具调用功能：

```bash
# 测试 bash 工具
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What files are in the current directory?", "use_agent": true}'
```

## 对应 OpenClaw 源码

- `src/agents/index.ts`: Agent 系统入口
- `src/agents/runner.ts`: Agent 执行器
- `src/agents/tools.ts`: 工具系统
- `src/agents/types.ts`: Agent 类型定义
- `src/agents/tool-catalog.ts`: 工具目录管理
- `src/agents/pi-embedded-runner/loop.ts`: Agent 循环实现