# s07: 会话管理

`[ s01 > s02 > s03 > s04 > s05 > s06 > s07 ] s08 > s09 | s10 > s11 > s12`

> *"One session & persistence is all you need"* -- 一个会话 + 持久化 = 状态记忆。
> 
> **智能层**: 会话管理 -- 维护对话历史与上下文状态。

## 问题 (Problem)

在自动回复管道基础上，我们需要维护跨消息的对话状态和历史记录。当前每次请求都是无状态的，AI 无法记住之前的对话内容，导致连续对话体验不佳。我们需要一种机制来存储和检索每个用户的对话历史，并支持会话级别的状态管理。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Message  |      | Session     |      | Persistent       |
| History  | ---> | Management  | ---> | State            |
| Context  |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Session Lifecycle |
              | - create_session  |
              | - append_message  |
              | - get_history     |
              | - compact_memory  |
              +-------------------+
              
Session Store Pattern:
- 内存存储用于开发测试
- 文件存储用于生产环境  
- 支持多种持久化后端
- 自动清理过期会话
```

引入会话管理系统：Session 类封装单个会话的状态，SessionStore ABC 定义存储接口，SessionManager 协调所有会话操作。这种设计保持了核心处理管道不变，只是增加了状态持久化能力。

## 工作原理 (How It Works)

1. 定义会话实体：

```python
class Session:
    """会话类，封装单个会话的状态"""
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.messages = []  # 对话历史
        self.metadata = {}  # 会话元数据
        self.created_at = time.time()
        self.last_accessed = time.time()
        
    def add_message(self, role: str, content: str):
        """添加消息到会话历史"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": time.time()
        })
        self.last_accessed = time.time()
    
    def get_recent_messages(self, n: int) -> list:
        """获取最近的n条消息"""
        return self.messages[-n:]
```

2. 构建存储抽象：

```python
class SessionStore(ABC):
    """会话存储抽象基类"""
    
    @abstractmethod
    async def save_session(self, session: Session):
        """保存会话"""
        pass
    
    @abstractmethod
    async def load_session(self, session_id: str) -> Session:
        """加载会话"""
        pass
    
    @abstractmethod
    async def delete_session(self, session_id: str):
        """删除会话"""
        pass
```

3. 实现具体存储：
- MemorySessionStore: 基于内存的临时存储（适用于开发测试）
- FileSessionStore: 基于文件的持久化存储（适用于生产环境）
- RedisSessionStore: 基于 Redis 的分布式存储（适用于集群部署）

4. 构建会话管理器：

```python
class SessionManager:
    def __init__(self, store: SessionStore):
        self.store = store
        self.active_sessions = {}
    
    async def get_or_create_session(self, session_id: str) -> Session:
        """获取现有会话或创建新会话"""
        if session_id in self.active_sessions:
            return self.active_sessions[session_id]
        
        # 尝试从存储加载
        session = await self.store.load_session(session_id)
        if not session:
            session = Session(session_id)
        
        self.active_sessions[session_id] = session
        return session
    
    async def save_session(self, session: Session):
        """保存会话到存储"""
        await self.store.save_session(session)
        # 更新活跃会话缓存
        self.active_sessions[session.session_id] = session
```

5. 在主应用中集成：

```python
# 初始化会话管理器
session_manager = SessionManager(FileSessionStore("./sessions"))

async def process_message_with_context(message: dict) -> dict:
    # 获取用户会话
    session_id = message.get("user_id", "default")
    session = await session_manager.get_or_create_session(session_id)
    
    # 添加用户消息到会话历史
    session.add_message("user", message["text"])
    
    # 准备带历史的提示词
    history = session.get_recent_messages(n=10)  # 最近10条消息
    full_prompt = [
        {"role": "system", "content": "You are a helpful assistant"},
    ] + history
    
    # 调用 AI 并获取响应
    response = await provider_manager.chat_with_default(full_prompt)
    ai_response = response["choices"][0]["message"]["content"]
    
    # 添加 AI 回复到会话历史
    session.add_message("assistant", ai_response)
    
    # 保存会话
    await session_manager.save_session(session)
    
    return {"reply": ai_response}
```

## 变化 (What Changed)

相比 s06，在不改变原有 HTTP 服务器、路由、渠道、提供商、Agent 和管道架构的基础上，新增了会话管理系统。现在系统能够维护跨消息的对话历史，提供更好的连续对话体验。会话管理器提供了灵活的存储选项以适应不同的部署需求。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s07_session_management.py
```

测试会话功能：

```bash
# 第一次对话
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "My name is Alice", "user_id": "alice_123"}'

# 后续对话（AI 应该记得之前的信息）
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What did I tell you earlier?", "user_id": "alice_123"}'
```

## 对应 OpenClaw 源码

- `src/sessions/index.ts`: 会话系统入口
- `src/sessions/session.ts`: 会话实体定义
- `src/sessions/store.ts`: 存储接口定义
- `src/sessions/manager.ts`: 会话管理器实现
- `src/sessions/types.ts`: 会话相关类型定义
- `src/gateway/server-methods/sessions.ts`: 会话相关的网关方法