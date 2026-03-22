# s09: CLI 命令系统

`[ s01 > s02 > s03 > s04 > s05 > s06 > s07 > s08 > s09 ] s10 > s11 | s12`

> *"One command & argparse is all you need"* -- 一个命令 + 参数解析 = 系统控制。
> 
> **系统层**: CLI 命令系统 -- 统一的命令行接口。

## 问题 (Problem)

在插件架构基础上，我们需要提供一种方式让用户能够通过命令行与网关交互，执行各种管理任务（如启动服务、查看状态、配置设置等）。如果没有统一的命令行界面，用户需要直接操作各个组件，这既不方便也不安全。我们需要一个结构化的 CLI 系统来暴露网关的核心功能。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------------+
| User Cmd |      | CliCommand  |      | Executed         |
| Args     | ---> | Registry    | ---> | Action           |
|          |      |             |      |                  |
+----------+      +------+------+      +------------------+
                      |
                      v
              +-------------------+
              | Command Lifecycle |
              | - parse_args      |
              | - validate        |
              | - execute         |
              | - handle_error    |
              +-------------------+
              
Cli Command Pattern:
- 定义命令参数和行为
- 集中注册和管理命令
- 支持子命令和选项
- 提供帮助信息
```

引入 CLI 命令系统：CliArgument 定义参数，CliCommand 封装命令逻辑，CliRegistry 管理所有命令，CliRunner 执行命令。这种设计保持了核心处理管道不变，只是增加了命令行控制能力。

## 工作原理 (How It Works)

1. 定义命令参数结构：

```python
class CliArgument:
    """CLI 参数定义"""
    def __init__(self, name: str, type_: type, required: bool = False, default=None, help_text: str = ""):
        self.name = name
        self.type = type_
        self.required = required
        self.default = default
        self.help_text = help_text
```

2. 构建命令基类：

```python
class CliCommand(ABC):
    """CLI 命令抽象基类"""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """命令名称"""
        pass
    
    @property
    @abstractmethod
    def description(self) -> str:
        """命令描述"""
        pass
    
    @abstractmethod
    def get_arguments(self) -> list[CliArgument]:
        """获取命令参数"""
        pass
    
    @abstractmethod
    async def execute(self, args: dict) -> int:
        """执行命令，返回退出码"""
        pass
```

3. 实现命令注册表：

```python
class CliRegistry:
    """CLI 命令注册表"""
    def __init__(self):
        self.commands = {}
    
    def register_command(self, cmd: CliCommand):
        self.commands[cmd.name] = cmd
    
    def get_parser(self) -> argparse.ArgumentParser:
        parser = argparse.ArgumentParser(description="OpenClaw Gateway CLI")
        subparsers = parser.add_subparsers(dest="command", help="Available commands")
        
        for name, cmd in self.commands.items():
            subparser = subparsers.add_parser(name, help=cmd.description)
            
            for arg in cmd.get_arguments():
                kwargs = {
                    "type": arg.type,
                    "help": arg.help_text
                }
                
                if not arg.required:
                    kwargs["default"] = arg.default
                    kwargs["required"] = arg.required
                
                subparser.add_argument(f"--{arg.name}", **kwargs)
        
        return parser
```

4. 构建命令执行器：

```python
class CliRunner:
    """CLI 命令执行器"""
    def __init__(self, registry: CliRegistry):
        self.registry = registry
    
    async def run(self, argv: list[str]) -> int:
        parser = self.registry.get_parser()
        args = parser.parse_args(argv)
        
        if not args.command:
            parser.print_help()
            return 1
        
        if args.command in self.registry.commands:
            cmd = self.registry.commands[args.command]
            # 转换参数为字典
            cmd_args = {k: v for k, v in vars(args).items() 
                       if k != 'command'}
            return await cmd.execute(cmd_args)
        else:
            print(f"Unknown command: {args.command}")
            return 1
```

5. 实现具体命令：
- GatewayRunCommand: 启动网关服务器
- ConfigSetCommand: 设置配置项
- ChannelsStatusCommand: 查看渠道状态
- AgentMessageCommand: 发送代理消息

6. 在主应用中集成：

```python
# 注册内置命令
cli_registry = CliRegistry()
cli_registry.register_command(GatewayRunCommand())
cli_registry.register_command(ConfigSetCommand())
cli_registry.register_command(ChannelsStatusCommand())

# 主入口点
if __name__ == "__main__":
    import sys
    runner = CliRunner(cli_registry)
    exit_code = asyncio.run(runner.run(sys.argv[1:]))
    sys.exit(exit_code)
```

## 变化 (What Changed)

相比 s08，在不改变原有 HTTP 服务器、路由、渠道、提供商、Agent、管道、会话管理和插件架构的基础上，新增了 CLI 命令系统。现在用户可以通过命令行与网关进行交互，执行管理任务。CLI 系统提供了结构化的命令组织方式，易于扩展新功能。

## 动手试 (Try It)

运行 CLI 命令：

```bash
cd /path/to/learn-openclaw

# 查看帮助
python agents/s09_cli_command_system.py --help

# 运行特定命令
python agents/s09_cli_command_system.py gateway-run --host 127.0.0.1 --port 18789

# 查看配置
python agents/s09_cli_command_system.py config-get --key model
```

## 对应 OpenClaw 源码

- `src/cli/index.ts`: CLI 入口点
- `src/cli/types.ts`: CLI 类型定义
- `src/cli/commands/index.ts`: 命令系统入口
- `src/cli/commands/run.ts`: 网关运行命令
- `src/cli/commands/config.ts`: 配置管理命令
- `src/commands/onboard-search.ts`: 内置命令实现
- `src/commands/gateway-run.ts`: 网关运行命令