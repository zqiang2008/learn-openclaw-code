# s08: 插件架构

`[ s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 ] s09 > s10 | s11 > s12`

> *"One plugin & extensibility is all you need"* -- 一个插件 + 可扩展性 = 功能拓展。
> 
> **系统层**: 插件架构 -- 动态扩展网关功能。

## 问题 (Problem)

在会话管理系统基础上，我们需要一种机制来动态扩展网关的功能，而无需修改核心代码。比如添加新的渠道、集成第三方服务、实现自定义命令等。硬编码所有可能的功能会导致系统臃肿且难以维护。我们需要一个插件系统来支持按需加载和运行额外功能。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Plugin   |      | Plugin      |      | Extended         |
| Module   | ---> | Runtime     | ---> | Functionality    |
| (.py/.ts)|      | System      |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Plugin Lifecycle  |
              | - load            |
              | - validate        |
              | - initialize      |
              | - execute         |
              +-------------------+
              
Plugin Manifest Pattern:
- 定义插件元数据
- 声明依赖关系  
- 指定入口点
- 版本兼容性检查
```

引入插件架构：PluginManifest 定义插件元数据，PluginEntry 实现具体功能，PluginLoader 负责动态加载，PluginManager 统一管理所有插件。这种设计保持了核心处理管道不变，只是增加了功能扩展能力。

## 工作原理 (How It Works)

1. 定义插件清单结构：

```python
class PluginManifest:
    """插件清单，定义插件元数据"""
    def __init__(self, name: str, version: str, description: str, entry_point: str, dependencies: list = None):
        self.name = name
        self.version = version
        self.description = description
        self.entry_point = entry_point  # 入口函数路径
        self.dependencies = dependencies or []
        self.author = ""
        self.license = ""
```

2. 构建插件基类：

```python
class PluginEntry(ABC):
    """插件条目抽象基类"""
    
    @abstractmethod
    async def initialize(self, context: dict):
        """初始化插件"""
        pass
    
    @abstractmethod
    async def execute(self, params: dict) -> dict:
        """执行插件功能"""
        pass
    
    @abstractmethod
    def get_metadata(self) -> dict:
        """获取插件元数据"""
        pass
```

3. 实现插件加载器：

```python
class PluginLoader:
    """插件加载器，负责动态加载和实例化插件"""
    
    @staticmethod
    async def load_from_file(plugin_path: str) -> tuple[PluginManifest, PluginEntry]:
        """从文件加载插件"""
        # 加载 manifest.json
        manifest_path = os.path.join(plugin_path, "manifest.json")
        with open(manifest_path, "r") as f:
            manifest_data = json.load(f)
        
        manifest = PluginManifest(**manifest_data)
        
        # 动态导入插件模块
        spec = importlib.util.spec_from_file_location(
            f"plugin_{manifest.name}", 
            os.path.join(plugin_path, "plugin.py")
        )
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        # 获取插件实例
        plugin_instance = getattr(module, "PluginImpl")()
        
        return manifest, plugin_instance
```

4. 构建插件管理器：

```python
class PluginManager:
    def __init__(self):
        self.plugins = {}
        self.manifests = {}
    
    async def register_plugin(self, plugin_path: str):
        """注册插件"""
        try:
            manifest, instance = await PluginLoader.load_from_file(plugin_path)
            
            # 验证依赖
            for dep in manifest.dependencies:
                if dep not in self.plugins:
                    raise ValueError(f"Dependency {dep} not found for plugin {manifest.name}")
            
            # 初始化插件
            await instance.initialize({
                "gateway": self.gateway_context,  # 提供对网关的访问
                "config": self.config_manager
            })
            
            self.plugins[manifest.name] = instance
            self.manifests[manifest.name] = manifest
            
            print(f"[plugin] Loaded {manifest.name} v{manifest.version}")
        except Exception as e:
            print(f"[plugin] Failed to load {plugin_path}: {str(e)}")
    
    async def execute_plugin(self, plugin_name: str, params: dict) -> dict:
        """执行指定插件"""
        if plugin_name in self.plugins:
            return await self.plugins[plugin_name].execute(params)
        else:
            raise ValueError(f"Plugin {plugin_name} not found")
    
    def list_plugins(self) -> list:
        """列出所有已加载插件"""
        return [m.name for m in self.manifests.values()]
```

5. 在主应用中集成：

```python
# 初始化插件管理器
plugin_manager = PluginManager()

# 自动扫描并加载插件
async def load_plugins():
    plugins_dir = "./plugins"
    for plugin_dir in os.listdir(plugins_dir):
        plugin_path = os.path.join(plugins_dir, plugin_dir)
        if os.path.isdir(plugin_path):
            await plugin_manager.register_plugin(plugin_path)

# 修改消息处理器以支持插件调用
async def process_message_with_plugins(message: dict) -> dict:
    # 检查是否为插件命令
    if message.get("type") == "plugin_call":
        plugin_name = message.get("plugin")
        params = message.get("params", {})
        return await plugin_manager.execute_plugin(plugin_name, params)
    
    # 否则按常规流程处理
    return await regular_process_message(message)
```

## 变化 (What Changed)

相比 s07，在不改变原有 HTTP 服务器、路由、渠道、提供商、Agent、管道和会话管理架构的基础上，新增了插件系统。现在可以动态加载和执行外部插件来扩展网关功能，而无需修改核心代码。插件管理器提供了安全的隔离环境来运行第三方代码。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s08_plugin_architecture.py
```

测试插件功能（需要先创建示例插件）：

```bash
# 创建示例插件目录
mkdir -p ./plugins/example/{manifest.json,plugin.py}

# 测试插件调用
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"type": "plugin_call", "plugin": "example", "params": {"action": "hello"}}'
```

## 对应 OpenClaw 源码

- `src/plugins/index.ts`: 插件系统入口
- `src/plugins/types.ts`: 插件类型定义
- `src/plugins/loader.ts`: 插件加载器
- `src/plugins/manager.ts`: 插件管理器
- `src/plugins/runtime.ts`: 插件运行时环境
- `src/plugin-sdk/index.ts`: 插件开发工具包