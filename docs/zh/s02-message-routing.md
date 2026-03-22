# s02: 消息路由引擎

`[ s01 > s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One router & rules is all you need"* -- 一个路由器 + 规则 = 智能分发。
> 
> **基础层**: 消息路由 -- 根据规则智能分发消息到不同处理管道。

## 问题 (Problem)

在 Gateway 服务器基础上，我们收到各种不同类型的消息（命令、关键词触发、普通聊天），但目前所有消息都走同一个处理流程。我们需要根据消息特征进行分类和路由，让不同类型的请求进入不同的处理管道。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Message  | ---> | Router      | ---> | Matching Handler |
| Type     |      |             |      |                  |
| - Command|      | [Rule 1]    |      | - /api/command   |
| - Keyword|      | [Rule 2]    |      | - keyword_match  |
| - Normal |      | [Rule 3]    |      | - normal_handler |
+----------+      +------+------+      +------------------+
                      |
                      v
              RouteRule Engine
              (channel_match/keyword_match/sender_match)
```

引入路由规则系统：RouteRule 定义匹配条件，Router 根据规则将消息分发到相应处理器。这种设计保持了核心处理管道不变，只是增加了前置的路由逻辑。

## 工作原理 (How It Works)

1. 定义路由规则基类：

```python
class RouteRule(ABC):
    """路由规则抽象基类"""
    
    @abstractmethod
    def matches(self, message: dict) -> bool:
        """判断消息是否符合此规则"""
        pass
    
    @abstractmethod
    def handle(self, message: dict) -> dict:
        """处理符合条件的消息"""
        pass
```

2. 实现具体规则：
- ChannelMatchRule: 根据渠道类型路由
- KeywordMatchRule: 根据关键词路由  
- SenderMatchRule: 根据发送者路由
- DefaultRule: 默认处理规则

3. 构建路由器：

```python
class Router:
    def __init__(self):
        self.rules = []
    
    def add_rule(self, rule: RouteRule):
        self.rules.append(rule)
    
    def route(self, message: dict) -> dict:
        for rule in self.rules:
            if rule.matches(message):
                return rule.handle(message)
        
        # 默认处理
        return {"error": "No matching rule found"}
```

4. 在主应用中集成：

```python
# 添加路由规则
router.add_rule(ChannelMatchRule("telegram", telegram_handler))
router.add_rule(KeywordMatchRule(["help", "info"], help_handler))  
router.add_rule(SenderMatchRule("@admin", admin_handler))
router.add_rule(DefaultRule(default_handler))

# 修改 API 端点使用路由器
@app.route("/api/chat", methods=["POST"])
def handle_chat():
    data = request.json
    result = router.route(data)
    return jsonify(result)
```

## 变化 (What Changed)

相比 s01，在不改变原有 HTTP 服务器架构的基础上，新增了路由机制。现在消息会先经过路由规则匹配，再分发到相应的处理函数。这使得我们可以为不同类型的请求实现专门的处理逻辑，而不会影响其他消息的处理流程。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s02_message_routing.py
```

测试不同路由规则：

```bash
# 测试渠道匹配
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello", "channel": "telegram"}'

# 测试关键词匹配  
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Can you help me?", "sender": "user1"}'

# 测试默认规则
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Just chatting", "channel": "unknown"}'
```

## 对应 OpenClaw 源码

- `src/routing/index.ts`: 路由系统入口
- `src/routing/rules.ts`: 各种路由规则定义
- `src/routing/router.ts`: 路由器实现
- `src/channels/handlers.ts`: 渠道特定处理器