# s04: LLM Provider 抽象

`[ s01 > s02 > s03 > s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One provider & abstraction is all you need"* -- 一个提供者 + 抽象 = 统一 AI 接口。
> 
> **基础层**: LLM 抽象 -- 将不同 AI 服务统一到单一调用管道。

## 问题 (Problem)

在多渠道适配基础上，我们需要连接不同的 LLM 提供商（OpenAI、Anthropic、Ollama等）。每个提供商有自己的 API 格式、认证方式和响应结构。如果为每个提供商单独实现一套逻辑，会导致代码重复且难以切换。我们需要一种机制来将所有 LLM 服务统一到相同的调用管道中。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| OpenAI   |      | Provider    |      | Unified AI       |
| Anthropic| ---> | Abstraction | ---> | Processing       |
| Ollama   |      | (Interface) |      | Pipeline         |
| ...      |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              ProviderManager
              (select/configure/use)
```

引入 Provider 抽象模式：LLMProvider ABC 定义统一接口，各具体提供商继承并实现。ProviderManager 管理所有注册的提供商实例。这种设计保持了核心处理管道不变，只是增加了提供商抽象层。

## 工作原理 (How It Works)

1. 定义提供商抽象基类：

```python
class LLMProvider(ABC):
    """LLM 提供商抽象基类"""
    
    @abstractmethod
    async def generate_text(self, prompt: str, model: str = None) -> str:
        """生成文本"""
        pass
    
    @abstractmethod
    async def chat_completion(self, messages: list, model: str = None) -> dict:
        """聊天完成"""
        pass
    
    @property
    @abstractmethod
    def name(self) -> str:
        """提供商名称"""
        pass
```

2. 实现具体提供商：
- OpenAIProvider: 实现 OpenAI API 接口
- AnthropicProvider: 实现 Anthropic Claude API 接口  
- OllamaProvider: 实现本地 Ollama API 接口
- AzureProvider: 实现 Azure OpenAI 接口

3. 构建提供商管理器：

```python
class ProviderManager:
    def __init__(self):
        self.providers = {}
        self.default_provider = "openai"
    
    def register_provider(self, provider: LLMProvider):
        self.providers[provider.name] = provider
        
    def unregister_provider(self, provider_name: str):
        if provider_name in self.providers:
            del self.providers[provider_name]
    
    async def generate_with_provider(self, provider_name: str, prompt: str, model: str = None):
        if provider_name in self.providers:
            return await self.providers[provider_name].generate_text(prompt, model)
        else:
            raise ValueError(f"Provider {provider_name} not found")
    
    async def chat_with_default(self, messages: list, model: str = None):
        default = self.providers.get(self.default_provider)
        if default:
            return await default.chat_completion(messages, model)
        else:
            raise ValueError(f"Default provider {self.default_provider} not found")
```

4. 在主应用中集成：

```python
# 注册提供商
provider_manager.register_provider(OpenAIProvider(api_key=os.getenv("OPENAI_API_KEY")))
provider_manager.register_provider(AnthropicProvider(api_key=os.getenv("ANTHROPIC_API_KEY")))

# 修改消息处理器以使用提供商
async def process_message(message: dict) -> dict:
    # 使用默认提供商生成回复
    response = await provider_manager.chat_with_default([
        {"role": "system", "content": "You are a helpful assistant"},
        {"role": "user", "content": message["text"]}
    ])
    
    return {"reply": response["choices"][0]["message"]["content"]}
```

## 变化 (What Changed)

相比 s03，在不改变原有 HTTP 服务器、路由和渠道架构的基础上，新增了 LLM 提供商抽象层。现在我们可以轻松切换不同的 AI 服务而无需修改核心处理逻辑。提供商管理器提供了统一的方式来管理和操作不同的 AI 服务实例。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s04_provider_abstraction.py
```

测试不同提供商：

```bash
# 设置环境变量
export OPENAI_API_KEY="your_openai_key_here"

# 测试通过 API 调用
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Say hello in Spanish", "provider": "openai"}'
```

## 对应 OpenClaw 源码

- `src/providers/index.ts`: 提供商系统入口
- `src/providers/base.ts`: 提供商抽象基类
- `src/providers/openai/`: OpenAI 提供商实现
- `src/providers/anthropic/`: Anthropic 提供商实现
- `src/providers/ollama/`: Ollama 提供商实现
- `src/providers/provider-manager.ts`: 提供商管理器