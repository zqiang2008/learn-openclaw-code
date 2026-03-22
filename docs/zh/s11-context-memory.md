# s11: 上下文引擎与记忆

`[ s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 > s11 ] s12`

> *"One memory & context is all you need"* -- 一个记忆 + 上下文 = 智能感知。
> 
> **智能层**: 上下文记忆系统 -- 长期记忆与上下文管理。

## 问题 (Problem)

在分层配置管理系统基础上，我们需要实现更高级的记忆功能：不仅要有短期的会话记忆（s07 中实现的），还需要长期记忆、上下文压缩、知识库检索等功能。当前的会话系统只保存原始消息历史，随着对话增长会变得冗长且低效。AI 需要能够访问过去的知识和经验，而不仅仅是最近的消息。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Context  |      | Memory      |      | Intelligent    |
| Data     | ---> | Management  | ---> | Reasoning      |
| (Past    |      | System      |      |                  |
| Events)  |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Memory Lifecycle  |
              | - store           |
              | - retrieve        |
              | - compact         |
              | - expire          |
              +-------------------+
              
Context Engine Pattern:
- 结构化存储记忆片段
- 支持语义搜索和关联
- 实现记忆压缩算法
- 提供上下文注入机制
```

引入上下文记忆系统：MemoryEntry 封装记忆单元，MemoryStore 管理存储后端，ContextCompactor 实现记忆压缩，MemoryManager 统一协调所有记忆操作。这种设计保持了核心处理管道不变，只是增加了长期记忆能力。

## 工作原理 (How It Works)

1. 定义记忆条目结构：

```python
class MemoryEntry:
    """记忆条目，封装单个记忆单元"""
    def __init__(self, id_: str, content: str, metadata: dict = None, timestamp: float = None, importance: float = 1.0):
        self.id = id_
        self.content = content
        self.metadata = metadata or {}
        self.timestamp = timestamp or time.time()
        self.importance = importance  # 重要性评分 (0.0-1.0)
        self.embedding = None  # 可选的向量嵌入用于相似度搜索
```

2. 构建存储抽象：

```python
class MemoryStore(ABC):
    """记忆存储抽象基类"""
    
    @abstractmethod
    async def save_memory(self, entry: MemoryEntry):
        """保存记忆"""
        pass
    
    @abstractmethod
    async def load_memory(self, id_: str) -> MemoryEntry:
        """加载记忆"""
        pass
    
    @abstractmethod
    async def search_memories(self, query: str, limit: int = 10) -> list[MemoryEntry]:
        """搜索相关记忆"""
        pass
    
    @abstractmethod
    async def delete_memory(self, id_: str):
        """删除记忆"""
        pass
```

3. 实现具体存储：
- FileMemoryStore: 基于文件的持久化存储
- VectorMemoryStore: 基于向量数据库的语义搜索存储
- RedisMemoryStore: 基于 Redis 的高性能存储

4. 构建上下文压缩器：

```python
class ContextCompactor:
    """上下文压缩器，将大量记忆压缩为摘要"""
    
    def __init__(self, provider_manager: ProviderManager):
        self.provider_manager = provider_manager
    
    async def compact_context(self, memories: list[MemoryEntry], max_tokens: int = 2000) -> str:
        """将多个记忆条目压缩为简洁摘要"""
        if len(memories) == 0:
            return ""
        
        # 按重要性和时间排序
        sorted_memories = sorted(
            memories, 
            key=lambda x: (x.importance, x.timestamp), 
            reverse=True
        )
        
        # 分批压缩
        batch_size = 5
        summaries = []
        
        for i in range(0, len(sorted_memories), batch_size):
            batch = sorted_memories[i:i+batch_size]
            batch_text = "\n".join([f"- {m.content}" for m in batch])
            
            # 使用 AI 压缩批次
            prompt = f"""
            请将以下相关信息压缩为一句简短的摘要：
            
            {batch_text}
            
            摘要:"""
            
            summary = await self.provider_manager.generate_with_provider("openai", prompt)
            summaries.append(summary.strip())
        
        # 合并最终摘要
        final_summary = " ".join(summaries[:max_tokens//10])  # 控制长度
        return final_summary
```

5. 构建记忆管理器：

```python
class MemoryManager:
    def __init__(self, store: MemoryStore, compactor: ContextCompactor):
        self.store = store
        self.compactor = compactor
        self.cache = {}  # 内存缓存
    
    async def add_memory(self, content: str, session_id: str = None, importance: float = 1.0):
        """添加新记忆"""
        import uuid
        entry = MemoryEntry(
            id_=str(uuid.uuid4()),
            content=content,
            metadata={"session_id": session_id},
            importance=importance
        )
        
        await self.store.save_memory(entry)
        self.cache[entry.id] = entry
        
        print(f"[memory] Added: {content[:50]}...")
    
    async def get_relevant_memories(self, query: str, session_id: str = None, limit: int = 5) -> list[MemoryEntry]:
        """获取相关记忆"""
        # 添加会话限制条件
        search_query = query
        if session_id:
            search_query += f" (in session {session_id})"
        
        results = await self.store.search_memories(search_query, limit)
        
        # 过滤特定会话的记忆（如果指定了）
        if session_id:
            results = [r for r in results if r.metadata.get("session_id") == session_id]
        
        return results
    
    async def compress_and_store_context(self, session_id: str):
        """压缩并存储会话上下文"""
        # 获取会话的所有记忆
        all_memories = await self.store.search_memories(f"in session {session_id}", limit=100)
        
        # 压缩为摘要
        compressed = await self.compactor.compact_context(all_memories)
        
        if compressed:
            # 存储压缩后的上下文作为新的记忆
            await self.add_memory(
                content=f"Session {session_id} context summary: {compressed}",
                session_id=session_id,
                importance=0.8
            )
```

6. 在主应用中集成：

```python
# 初始化记忆系统
memory_manager = MemoryManager(FileMemoryStore("./memories"), context_compactor)

async def process_message_with_memory(message: dict) -> dict:
    user_msg = message["text"]
    session_id = message.get("user_id", "default")
    
    # 检索相关记忆
    relevant_memories = await memory_manager.get_relevant_memories(user_msg, session_id)
    
    # 准备带记忆的提示词
    memory_context = "\n".join([f"Remember: {m.content}" for m in relevant_memories])
    
    full_prompt = [
        {"role": "system", "content": f"You are a helpful assistant.\n\n{memory_context}"},
        {"role": "user", "content": user_msg}
    ]
    
    # 调用 AI 并获取响应
    response = await provider_manager.chat_with_default(full_prompt)
    ai_response = response["choices"][0]["message"]["content"]
    
    # 存储这次交互到记忆
    await memory_manager.add_memory(f"User asked: {user_msg}", session_id, importance=0.5)
    await memory_manager.add_memory(f"Assistant replied: {ai_response}", session_id, importance=0.5)
    
    return {"reply": ai_response}
```

## 变化 (What Changed)

相比 s10，在不改变原有 HTTP 服务器、路由、渠道、提供商、Agent、管道、会话管理和插件架构的基础上，新增了上下文记忆系统。现在系统不仅能维护短期会话记忆，还能存储和检索长期记忆，并通过压缩算法管理大量历史信息。这使得 AI 能够更好地利用过去的交互经验。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s11_context_memory.py
```

测试记忆功能：

```bash
# 第一次对话 - 创建记忆
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "My favorite color is blue and I live in New York", "user_id": "user_abc"}'

# 后续对话 - AI 应该记住之前的信息
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Remind me what my favorite color is?", "user_id": "user_abc"}'
```

## 对应 OpenClaw 源码

- `src/memory/index.ts`: 记忆系统入口
- `src/memory/types.ts`: 记忆相关类型定义
- `src/memory/store.ts`: 记忆存储接口
- `src/memory/engine.ts`: 记忆引擎实现
- `src/context-engine/index.ts`: 上下文引擎入口
- `src/context-engine/compactor.ts`: 上下文压缩器
- `src/memory/manager.ts`: 记忆管理器