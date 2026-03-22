# s10: 分层配置管理

`[ s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 > s10 ] s11 > s12`

> *"One config & layers is all you need"* -- 一个配置 + 层次 = 统一管理。
> 
> **系统层**: 配置系统 -- 多层级配置管理。

## 问题 (Problem)

在 CLI 命令系统基础上，我们需要管理大量的配置项（如 API 密钥、服务器地址、默认模型等）。这些配置可能来自多个地方：默认值、配置文件、环境变量、命令行参数等。当前硬编码的配置方式缺乏灵活性和安全性，特别是对于敏感信息（如 API 密钥）需要特殊处理。我们需要一种分层的配置管理系统来统一管理所有配置项。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| Config   |      | Config      |      | Layered          |
| Sources  | ---> | Management  | ---> | Values           |
| (5 levels)|      | System      |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Config Hierarchy  |
              | - defaults        |
              | - project_file    |
              | - user_file       |
              | - env_vars        |
              | - overrides       |
              +-------------------+
              
Config Schema Pattern:
- 定义字段类型和验证规则
- 支持默认值和环境变量映射
- 区分公开和私密字段
- 提供配置加载顺序
```

引入分层配置系统：ConfigField 定义单个配置项，ConfigSchema 管理所有字段定义，ConfigLoader 实现五层配置加载逻辑，ConfigManager 提供运行时访问接口。这种设计保持了核心处理管道不变，只是增加了配置管理能力。

## 工作原理 (How It Works)

1. 定义配置字段结构：

```python
class ConfigField:
    """配置字段定义"""
    def __init__(self, name: str, description: str, type_: type, default_value: Any = None, 
                 env_var: str = None, secret: bool = False, validator: callable = None):
        self.name = name
        self.description = description
        self.type = type_
        self.default_value = default_value
        self.env_var = env_var  # 对应的环境变量名
        self.secret = secret    # 是否为敏感字段
        self.validator = validator  # 验证函数
```

2. 构建配置模式：

```python
class ConfigSchema:
    """配置模式，管理所有字段定义"""
    def __init__(self):
        self.fields = {}
    
    def define(self, field: ConfigField):
        """定义配置字段"""
        self.fields[field.name] = field
    
    def get_field(self, name: str) -> ConfigField:
        """获取字段定义"""
        return self.fields.get(name)
    
    def validate(self, config: dict) -> tuple[bool, str]:
        """验证配置值"""
        for key, value in config.items():
            if key in self.fields:
                field = self.fields[key]
                
                # 类型检查
                if not isinstance(value, field.type):
                    return False, f"Field {key} must be of type {field.type.__name__}"
                
                # 自定义验证
                if field.validator and not field.validator(value):
                    return False, f"Validation failed for field {key}"
        
        return True, ""
```

3. 实现配置加载器（五层覆盖逻辑）：

```python
class ConfigLoader:
    """配置加载器，实现五层配置覆盖逻辑"""
    
    @staticmethod
    def load_config(schema: ConfigSchema) -> dict:
        """按优先级加载配置"""
        config = {}
        
        # 1. 默认值 (lowest priority)
        for name, field in schema.fields.items():
            config[name] = field.default_value
        
        # 2. 项目配置文件 (project/.openclawrc)
        project_config = ConfigLoader._load_from_file("./.openclawrc")
        config.update(project_config)
        
        # 3. 用户配置文件 (~/.openclaw/config)
        user_config = ConfigLoader._load_from_file(os.path.expanduser("~/.openclaw/config"))
        config.update(user_config)
        
        # 4. 环境变量 (environment variables)
        for name, field in schema.fields.items():
            if field.env_var and os.environ.get(field.env_var):
                env_val = os.environ[field.env_var]
                # 类型转换
                if field.type == bool:
                    env_val = env_val.lower() in ('true', '1', 'yes', 'on')
                elif field.type == int:
                    env_val = int(env_val)
                elif field.type == float:
                    env_val = float(env_val)
                config[name] = env_val
        
        # 5. 覆盖值 (highest priority, from CLI args)
        override_values = {}  # 这些通常由CLI传入
        config.update(override_values)
        
        # 验证最终配置
        valid, msg = schema.validate(config)
        if not valid:
            raise ValueError(f"Configuration validation failed: {msg}")
        
        return config
```

4. 构建配置管理器：

```python
class ConfigManager:
    """配置管理器，提供运行时访问接口"""
    def __init__(self, schema: ConfigSchema, config: dict):
        self.schema = schema
        self._config = config
    
    def get(self, key: str, default: Any = None) -> Any:
        """获取配置值"""
        return self._config.get(key, default)
    
    def set(self, key: str, value: Any):
        """设置配置值（运行时临时修改）"""
        field = self.schema.get_field(key)
        if field:
            # 验证类型
            if not isinstance(value, field.type):
                raise TypeError(f"Value for {key} must be of type {field.type.__name__}")
            
            # 验证自定义规则
            if field.validator and not field.validator(value):
                raise ValueError(f"Validation failed for {key}: {value}")
        
        self._config[key] = value
    
    def to_dict(self) -> dict:
        """导出配置（过滤 secret 字段）"""
        return {
            k: v for k, v in self._config.items()
            if not (self.schema.get_field(k) and self.schema.get_field(k).secret)
        }
    
    def save_to_file(self, path: str):
        """保存配置到文件（排除 secret 字段）"""
        p = Path(path)
        p.parent.mkdir(parents=True, exist_ok=True)
        
        safe_config = {
            k: v for k, v in self._config.items()
            if not (self.schema.get_field(k) and self.schema.get_field(k).secret)
        }
        
        with open(p, "w") as f:
            if p.suffix in (".yaml", ".yml"):
                import yaml
                yaml.dump(safe_config, f, default_flow_style=False)
            else:
                json.dump(safe_config, f, indent=2)
        print(f"  [config] Saved to {path}")
```

5. 在主应用中集成：

```python
# 构建默认配置模式
def build_default_schema() -> ConfigSchema:
    schema = ConfigSchema()
    schema.define(ConfigField("host", "Gateway host", str, "127.0.0.1"))
    schema.define(ConfigField("port", "Gateway port", int, 18789, env_var="OPENCLAW_PORT"))
    schema.define(ConfigField("model", "Default model", str, "gpt-4o-mini"))
    schema.define(ConfigField("provider", "Default provider", str, "openai"))
    schema.define(ConfigField("debug", "Debug mode", bool, False))
    schema.define(ConfigField("openai_api_key", "OpenAI API Key", str, "", 
                             env_var="OPENAI_API_KEY", secret=True))
    schema.define(ConfigField("discord_token", "Discord Bot Token", str, "",
                             env_var="DISCORD_TOKEN", secret=True))
    return schema

# 初始化配置
schema = build_default_schema()
raw_config = ConfigLoader.load_config(schema)
config_manager = ConfigManager(schema, raw_config)

# 使用配置启动服务
@app.route("/api/status", methods=["GET"])
def status():
    # 显示非敏感配置
    return jsonify({
        "status": "running",
        "config": config_manager.to_dict(),  # 不包含 secret 字段
        "timestamp": time.time()
    })
```

## 变化 (What Changed)

相比 s09，在不改变原有 HTTP 服务器、路由、渠道、提供商、Agent、管道、会话管理和插件架构的基础上，新增了分层配置管理系统。现在可以灵活地从多个来源加载配置，并且能够安全地处理敏感信息。配置管理器提供了统一的接口来访问和管理所有配置项。

## 动手试 (Try It)

启动服务器并查看配置：

```bash
cd /path/to/learn-openclaw

# 设置环境变量
export OPENAI_API_KEY="your-key-here"
export OPENCLAW_PORT=18790

# 运行带配置的应用
python agents/s10_config_system.py

# 查看状态端点显示的配置（不含敏感信息）
curl http://127.0.0.1:18790/api/status
```

## 对应 OpenClaw 源码

- `src/config/index.ts`: 配置系统入口
- `src/config/schema.ts`: 配置模式定义
- `src/config/loader.ts`: 配置加载器
- `src/config/types.ts`: 配置相关类型定义
- `src/config/manager.ts`: 配置管理器
- `src/config/default.ts`: 默认配置定义