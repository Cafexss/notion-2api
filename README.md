# 🚀 notion-2api: 将 Notion AI 转换为私有 OpenAI API

![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)
![Stars](https://img.shields.io/github/stars/lzA6/notion-2api?style=social)
![Issues](https://img.shields.io/github/issues/lzA6/notion-2api)
![Forks](https://img.shields.io/github/forks/lzA6/notion-2api)



## 📂 项目结构

```
notion-2api/
├── .env                    # 环境配置文件（生产环境）
├── .env.example            # 环境配置模板
├── Dockerfile              # Docker 镜像构建文件
├── docker-compose.yml      # Docker Compose 编排文件
├── main.py                 # FastAPI 应用主入口
├── nginx.conf              # Nginx 反向代理配置
├── requirements.txt        # Python 依赖列表
└── app/
    ├── core/
    │   ├── __init__.py
    │   └── config.py       # Pydantic 配置模型
    ├── providers/
    │   ├── __init__.py
    │   ├── base_provider.py # 抽象基类
    │   └── notion_provider.py # Notion API 交互核心
    └── utils/
        └── sse_utils.py    # Server-Sent Events 工具
```

---

## 🤔 工作原理

`notion-2api` 充当了 OpenAI API 和 Notion AI 之间的翻译官，将两种不同的协议进行无缝转换。

### 架构流程图

```mermaid
graph TD
    A[客户端应用] --> B[Nginx 反向代理]
    B --> C[FastAPI 应用]
    C --> D[认证中间件]
    D --> E[Notion Provider]
    E --> F[Cloudscraper]
    F --> G[Notion AI API]
    G --> H[NDJSON 流解析]
    H --> I[SSE 格式转换]
    I --> C
    C --> B
    B --> A
```

### 数据流详解

1. **请求接收**: 客户端发送标准的 OpenAI API 格式请求
2. **协议转换**: FastAPI 将请求转换为 Notion AI 能理解的格式
3. **会话管理**: 创建 Notion 对话线程并维持会话状态
4. **流式处理**: 实时解析 Notion 返回的 NDJSON 数据流
5. **格式转换**: 将解析后的数据转换为 OpenAI 兼容的 SSE 格式
6. **响应返回**: 通过流式响应将数据实时返回给客户端

---

## 🛠️ 技术栈

| 技术组件 | 版本/类型 | 用途说明 |
|---------|-----------|----------|
| **FastAPI** | 0.104+ | 现代化异步 Web 框架，提供高性能 API 服务 |
| **Uvicorn** | 0.24+ | ASGI 服务器，用于运行 FastAPI 应用 |
| **Cloudscraper** | 最新版 | 绕过 Cloudflare 防护，确保稳定连接 |
| **Pydantic** | 2.0+ | 数据验证和设置管理 |
| **Docker** | 最新版 | 容器化部署和环境隔离 |
| **Nginx** | 最新版 | 反向代理和负载均衡 |
| **Python** | 3.8+ | 主要编程语言 |

---

## 📖 使用教程

### 环境准备

1. **安装 Docker**: 访问 [Docker 官网](https://www.docker.com/get-started) 下载并安装
2. **安装 Git**: 用于克隆代码仓库

### 获取 Notion 凭证

这是最关键的一步，请仔细按照以下步骤操作：


### 配置环境变量

1. **克隆项目**:
   ```bash
   git clone https://github.com/lzA6/notion-2api.git
   cd notion-2api
   ```

2. **创建配置文件**:
   ```bash
   cp .env.example .env
   ```

3. **编辑 `.env` 文件**:
   ```env
   # 安全配置（可选）
   API_MASTER_KEY=your_secret_key_here
   
   # 部署配置
   NGINX_PORT=8088
   
   # Notion 凭证（必需）
   NOTION_COOKIE="你的token_v2值"
   NOTION_SPACE_ID="你的Space ID"
   NOTION_USER_ID="你的User ID"
   NOTION_USER_NAME="你的名字"
   NOTION_USER_EMAIL="your_email@example.com"
   ```

### 启动服务

```bash
docker-compose up -d --build
```

### 测试 API

```bash
curl http://localhost:8088/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_secret_key_here" \
  -d '{
    "model": "claude-sonnet-4.5",
    "messages": [{"role": "user", "content": "你好，请介绍一下你自己"}],
    "stream": true
  }'
```

如果一切正常，你将看到来自 Notion AI 的流式响应！

---

## 🧠 源码解析

### 核心模块架构

```mermaid
graph TB
    A[main.py] --> B[Config Manager]
    A --> C[Auth Middleware]
    A --> D[Route Handlers]
    D --> E[Notion Provider]
    E --> F[Session Management]
    E --> G[Stream Parser]
    E --> H[Response Formatter]
    G --> I[NDJSON Parser]
    H --> J[SSE Generator]
```

### `main.py` - 应用入口

负责初始化 FastAPI 应用、配置中间件和路由：

```python
@app.post("/v1/chat/completions")
async def chat_completions(
    request: ChatCompletionRequest,
    auth: bool = Depends(verify_api_key)
):
    """处理聊天补全请求，支持流式和非流式响应"""
```

### `app/core/config.py` - 配置管理

使用 Pydantic 进行类型安全的配置管理：

```python
class Settings(BaseSettings):
    """应用配置模型，自动从环境变量加载"""
    API_MASTER_KEY: str = "1"
    NGINX_PORT: int = 8088
    NOTION_COOKIE: str
    NOTION_SPACE_ID: str
    NOTION_USER_ID: str
```

### `app/providers/notion_provider.py` - 核心逻辑

实现与 Notion AI 的交互：

- **会话预热**: 初始化时访问 Notion 以获取有效 Cookie
- **线程管理**: 为每个对话创建独立的 Notion 线程
- **流式解析**: 实时解析 Notion 的 NDJSON 响应流
- **数据清洗**: 清理响应中的冗余信息和格式标记

### `app/utils/sse_utils.py` - 流式响应

将数据转换为 Server-Sent Events 格式：

```python
def create_sse_data(data: dict) -> str:
    """将数据转换为 SSE 格式"""
    return f"data: {json.dumps(data)}\n\n"
```

---

## 📊 项目评估

### ✅ 已完成功能



### 🌟 核心优势

1. **生态兼容性**: 无缝接入现有 OpenAI 生态工具

3. **隐私保护**: 私有化部署，数据完全自主控制
4. **技术价值**: 学习现代 Web 开发和 API 设计的优秀范例

---

## 🗺️ 未来规划

### 🎯 短期目标

- [ ] 支持非流式响应 (`stream=false`)
- [ ] 增强错误处理和用户反馈
- [ ] 改进 token 自动刷新机制
- [ ] 动态模型列表获取
- [ ] 更详细的运行日志

### 💡 长期愿景

- [ ] 多账户负载均衡
- [ ] Function Calling 支持
- [ ] Web 管理界面
- [ ] 插件化架构
- [ ] 使用量统计和限制

---

## 🤖 AI 开发者指南

### 项目理解要点



- 优先修复稳定性相关问题
- 改进错误处理和用户反馈
- 添加单元测试和集成测试
- 优化文档和示例代码

---

## 💖 贡献指南

我们欢迎各种形式的贡献：

- 🐛 **问题反馈**: 提交 [Issue](https://github.com/lzA6/notion-2api/issues)
- 🔧 **代码贡献**: 创建 [Pull Request](https://github.com/lzA6/notion-2api/pulls)
- 💡 **想法分享**: 参与 [Discussions](https://github.com/lzA6/notion-2api/discussions)
- ⭐ **项目支持**: 给项目点赞和分享

每一个贡献，无论大小，都是对开源世界的宝贵礼物。

---

**让我们一起，用代码创造更多可能性！** 🚀
