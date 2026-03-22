# s01: Gateway HTTP 服务器

`[ s01 ] s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One server & REST is all you need"* -- 一个服务器 + REST 接口 = 网关入口。
> 
> **基础层**: HTTP 服务器 -- 所有渠道消息的统一接入点。

## 问题 (Problem)

OpenClaw 需要接收来自不同渠道（WhatsApp、Telegram、Discord等）的消息，但每个渠道都有不同的 API 格式和调用方式。如果没有统一的网关，我们需要为每个渠道单独实现一套处理逻辑。

## 解决方案 (Solution)

```
+----------+      +-------------+      +------------+
| WhatsApp | ---> | HTTP Server | ---> | Message    |
| Telegram |      | (Gateway)   |      | Processing |
| Discord  |      |             |      | Pipeline   |
| ...      |      |             |      |            |
+----------+      +------+------+      +------------+
                      |
                      v
              Unified Interface
              (/api/chat, /api/send, etc.)
```

一个统一的 HTTP 服务器作为所有渠道消息的汇聚点。各渠道只需将消息发送到标准 REST 接口，后续处理流程保持一致。

## 工作原理 (How It Works)

1. 创建 Flask 应用实例：

```python
app = Flask(__name__)
```

2. 定义核心 API 端点：

```python
@app.route("/api/chat", methods=["POST"])  # 聊天接口
def handle_chat():
    data = request.json
    message = data.get("message")
    channel = data.get("channel", "unknown")
    
    # 返回模拟响应
    response = {
        "reply": f"Echo: {message}",
        "channel": channel,
        "timestamp": time.time()
    }
    return jsonify(response)
```

3. 实现其他必要端点：
- `/api/send`: 发送消息回渠道
- `/api/webhook`: 渠道 webhook 入口
- `/api/status`: 健康检查
- `/api/health`: 系统状态

4. 启动服务器：

```python
if __name__ == "__main__":
    app.run(host="127.0.0.1", port=18789, debug=False)
```

## 变化 (What Changed)

从零到有了基本的 HTTP 服务框架。虽然目前只是简单的 Echo 回复，但已经建立了完整的请求-响应管道，为后续的功能扩展打下了基础。

## 动手试 (Try It)

启动服务器：

```bash
cd /path/to/learn-openclaw
python agents/s01_gateway_server.py
```

测试接口：

```bash
curl -X POST http://127.0.0.1:18789/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello OpenClaw!", "channel": "test"}'
```

预期输出：
```json
{
  "reply": "Echo: Hello OpenClaw!",
  "channel": "test",
  "timestamp": 1234567890.123
}
```

## 对应 OpenClaw 源码

- `src/gateway/server.ts`: 主网关服务器
- `src/gateway/server-methods/`: 各种 API 方法实现
- `src/provider-web.ts`: Web 提供商接口