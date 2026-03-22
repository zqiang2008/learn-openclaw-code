# s03: 多渠道适配器

`[ s01 > s02 > s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One adapter & abstraction is all you need"* -- 一个适配器 + 抽象 = 统一接口。
> 
> **基础层**: 渠道抽象 -- 将不同通信渠道统一到单一处理管道。

## 问题 (Problem)

在消息路由基础上，我们需要支持多种通信渠道（WhatsApp、Telegram、Discord等）。每个渠道有自己的 API 格式、认证方式和消息结构。如果为每个渠道单独实现一套逻辑，会导致代码重复且难以维护。我们需要一种机制来将所有渠道统一到相同的处理管道中。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| WhatsApp |      | Channel     |      | Unified Message  |
| Telegram | ---> | Abstraction | ---> | Processing       |
| Discord  |      | (Adapter)   |      | Pipeline         |
| ...      |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              ChannelManager
              (register/unregister/send_message)
```

引入渠道适配器模式：Channel ABC 定义统一接口，各具体渠道继承并实现。ChannelManager 管理所有注册的渠道实例。这种设计保持了核心处理管道不变，只是增加了渠道适配层。

## 工作原理 (How It Works)

1. 定义渠道抽象基类：

```python
class Channel(ABC):
    """渠道抽象基类"""
    
    @abstractmethod
    async def send_message(self, message: str, recipient: str = None) -> dict:
        """发送消息"""
        pass
    
    @abstractmethod
    async def receive_message(self) -> dict:
        """接收消息"""
        pass
    
    @property
    @abstractmethod
    def name(self) -> str:
        """渠道名称"""
        pass
```

2. 实现具体渠道：
- TelegramChannel: 实现 Telegram API 接口
- DiscordChannel: 实现 Discord API 接口  
- WhatsAppChannel: 实现 WhatsApp API 接口
- SmsChannel: 实现短信接口

3. 构建渠道管理器：

```python
class ChannelManager:
    def __init__(self):
        self.channels = {}
    
    def register_channel(self, channel: Channel):
        self.channels[channel.name] = channel
        
    def unregister_channel(self, channel_name: str):
        if channel_name in self.channels:
            del self.channels[channel_name]
    
    async def send_to_channel(self, channel_name: str, message: str, recipient: str = None):
        if channel_name in self.channels:
            return await self.channels[channel_name].send_message(message, recipient)
        else:
            raise ValueError(f"Channel {channel_name} not found")
```

4. 在主应用中集成：

```python
# 注册渠道
channel_manager.register_channel(TelegramChannel(bot_token="..."))
channel_manager.register_channel(DiscordChannel(token="..."))

# 修改路由器以支持渠道特定处理
class ChannelMatchRule(RouteRule):
    def __init__(self, channel_type: str, handler_func):
        self.channel_type = channel_type
        self.handler_func = handler_func
    
    def matches(self, message: dict) -> bool:
        return message.get("channel") == self.channel_type
    
    def handle(self, message: dict) -> dict:
        # 使用渠道管理器发送响应
        asyncio.run(
            channel_manager.send_to_channel(
                message["channel"], 
                f"Processed: {message['text']}", 
                message.get("sender_id")
            )
        )
        return {"status": "sent", "to": message["channel"]}
```

## 变化 (What Changed)

相比 s02，在不改变原有 HTTP 服务器和路由架构的基础上，新增了渠道抽象层。现在我们可以轻松添加新的通信渠道而无需修改核心处理逻辑。渠道管理器提供了统一的方式来管理和操作不同的渠道实例。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s03_channel_adapters.py
```

测试多渠道功能：

```bash
# 模拟向不同渠道发送消息
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello from Telegram!", "channel": "telegram", "recipient": "@user1"}'

curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello from Discord!", "channel": "discord", "recipient": "#general"}'
```

## 对应 OpenClaw 源码

- `src/channels/index.ts`: 渠道系统入口
- `src/channels/base.ts`: 渠道抽象基类
- `src/channels/telegram/`: Telegram 渠道实现
- `src/channels/discord/`: Discord 渠道实现
- `src/channels/whatsapp/`: WhatsApp 渠道实现
- `src/channels/channel-manager.ts`: 渠道管理器