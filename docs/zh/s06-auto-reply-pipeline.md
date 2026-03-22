# s06: 自动回复管道

`[ s01 > s02 > s03 > s04 > s05 > s06 ] s07 > s08 | s09 > s10 > s11 > s12`

> *"One pipeline & stages is all you need"* -- 一个管道 + 阶段 = 流水线处理。
> 
> **智能层**: 自动回复管道 -- 多阶段消息处理流水线。

## 问题 (Problem)

在 Agent 系统基础上，我们需要实现更复杂的自动回复功能：去重、命令检测、队列管理、并发控制等。如果把这些逻辑都放在一个函数里，会导致代码复杂且难以维护。我们需要一种机制来将不同的处理步骤分解为独立的阶段，形成一条清晰的消息处理流水线。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Incoming |      | Pipeline    |      | Processed        |
| Message  | ---> | Stages      | ---> | Response         |
|          |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Stage Composition |
              | - debounce_stage  |
              | - command_stage   |
              | - agent_stage     |
              | - reply_stage     |
              +-------------------+
              
Envelope Pattern:
- 封装原始消息和元数据
- 在各阶段间传递状态
- 支持异步处理链
```

引入自动回复管道模式：使用 Envelope 封装消息，DispatchPipeline 实现多阶段处理。这种设计保持了核心处理管道不变，只是增加了分阶段处理能力。

## 工作原理 (How It Works)

1. 定义消息信封：

```python
class Envelope:
    """消息信封，封装原始消息及处理元数据"""
    def __init__(self, message: dict):
        self.original_message = message
        self.processed_data = {}
        self.timestamp = time.time()
        self.status = "pending"
        self.error = None
```

2. 构建管道基类：

```python
class PipelineStage(ABC):
    """管道阶段抽象基类"""
    
    @abstractmethod
    async def process(self, envelope: Envelope) -> Envelope:
        """处理信封"""
        pass
```

3. 实现具体处理阶段：
- InboundDebounce: 入站消重（防止短时间内重复消息）
- CommandDetection: 命令识别（检查是否为指令而非普通聊天）
- AgentExecution: Agent 执行（调用 AI 和工具）
- ReplyHandler: 回复处理（发送响应到原渠道）

4. 构建调度管道：

```python
class DispatchPipeline:
    def __init__(self):
        self.stages = []
    
    def add_stage(self, stage: PipelineStage):
        self.stages.append(stage)
    
    async def dispatch(self, message: dict) -> dict:
        envelope = Envelope(message)
        
        for stage in self.stages:
            envelope = await stage.process(envelope)
            if envelope.status == "error":
                return {"error": envelope.error}
        
        return envelope.processed_data
```

5. 在主应用中集成：

```python
# 创建并配置管道
pipeline = DispatchPipeline()
pipeline.add_stage(InboundDebounce())      # 消重
pipeline.add_stage(CommandDetection())     # 命令检测
pipeline.add_stage(AgentExecution())       # Agent 执行
pipeline.add_stage(ReplyHandler())         # 回复处理

# 修改 API 端点使用管道
@app.route("/api/chat", methods=["POST"])
async def handle_chat():
    data = request.json
    result = await pipeline.dispatch(data)
    return jsonify(result)
```

## 变化 (What Changed)

相比 s05，在不改变原有 HTTP 服务器、路由、渠道、提供商和 Agent 系统架构的基础上，新增了自动回复管道。现在消息会经过多个明确分离的处理阶段，每个阶段负责特定的任务。这种设计提高了系统的可扩展性和可维护性。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s06_auto_reply_pipeline.py
```

测试管道功能：

```bash
# 正常消息处理
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What files are in the current directory?", "channel": "test"}'

# 测试命令检测
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "/help", "channel": "test"}'
```

## 对应 OpenClaw 源码

- `src/auto-reply/index.ts`: 自动回复系统入口
- `src/auto-reply/pipeline.ts`: 管道实现
- `src/auto-reply/stages/`: 各处理阶段实现
- `src/auto-reply/types.ts`: 类型定义
- `src/auto-reply/handler.ts`: 消息处理器
- `src/auto-reply/queue.ts`: 队列管理系统