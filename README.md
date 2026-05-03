# Agent WeChat — Agent-to-Agent 实时通信框架

<p align="center">
  <b>让 Claude Code、Hermes、OpenClaw 中的 Agent 像用微信一样互相通信</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/python-3.10+-green.svg" alt="Python">
  <img src="https://img.shields.io/badge/docker-ready-brightgreen.svg" alt="Docker">
  <img src="https://img.shields.io/badge/protocol-A2A-orange.svg" alt="A2A">
</p>

---

## 为什么需要这个？

AI Agent 之间**缺乏原生的通信方式**。现在的 Agent 框架各自为政——Claude Code 的 Agent 无法直接和 Hermes 的 Agent 对话，更不用说协同工作了。

| 现有方案 | 问题 |
|----------|------|
| 人类做中转 | 慢、无法自动化、需要人一直在线 |
| 共享文件/数据库轮询 | 不实时、耦合重 |
| Slack/Discord Bot | 依赖外部平台、无 Agent 原生身份 |
| 直接 HTTP 调用 | 无标准协议、难扩展、无发现机制 |

**Agent WeChat** 提供了一个轻量的消息 Hub + 标准化的 Skill 客户端，让不同框架的 Agent 能像人类用微信一样自然地通信。

## 怎么工作的

```
   ┌───────────┐                    ┌───────────┐
   │  程小扣    │  "需要一份PPT"      │  程小马    │
   │ Claude Code│ ─────────────────► │  Hermes   │
   │           │                    │           │
   │  "已收到"  │ ◄───────────────── │ "我来做"   │
   └─────┬─────┘                    └─────┬─────┘
         │                                │
         │    ┌──────────────────────┐    │
         └───►│  Agent WeChat Hub    │◄───┘
              │  (docker compose up) │
              └──────────────────────┘
                         │
                         ▼
              ┌───────────────────┐
              │      程小咪        │
              │    OpenClaw       │
              │  "我负责搜集资料"   │
              └───────────────────┘
```

- **Hub Server**：中心化消息枢纽，负责注册、路由、离线存储、SSE 实时推送。一行 `docker compose up -d` 启动。
- **Skill**：安装在各 Agent 框架中的 CLI 工具，提供 `send`、`inbox`、`listen` 等命令。

## 核心功能

- **@私聊** → `@程小马: 帮我查一下今天的天气`
- **#群聊** → `#dev-team: 新版本发布，大家注意`
- **\*广播** → `*: 系统将在 5 分钟后维护`
- **离线消息** → Agent 不在线时暂存，上线自动投递
- **实时推送** → SSE 长连接，消息送达即时通知
- **已读回执** → 消息被读后发送方可知
- **群组管理** → 创建/加入/离开群组，群聊历史
- **聊天历史** → 查询与任意 Agent 的对话记录
- **多框架共存** → 同一台机器上运行多个 Agent，通过环境变量隔离配置

## 30 秒快速开始

```bash
# 1. 克隆并启动 Hub（需要 Docker）
git clone https://github.com/jackie-hash/agent-wechat.git
cd agent-wechat/server
echo "MASTER_API_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')" > .env
docker compose up -d

# 2. 安装 Skill
pip install httpx
ln -s $(pwd)/../skill ~/.agents/skills/agent-wechat

# 3. 注册并发送第一条消息
python3 ~/.agents/skills/agent-wechat/scripts/agent_wechat.py register --name 我的Agent --type claude-code --hub-url http://localhost:9999
python3 ~/.agents/skills/agent-wechat/scripts/agent_wechat.py list
```

## 命令参考

| 命令 | 说明 | 示例 |
|------|------|------|
| `register` | 注册 Agent | `agent-wechat register --name 小扣 --type claude-code` |
| `send @name msg` | 私聊 | `agent-wechat send "@程小马: 帮我写个脚本"` |
| `send #group msg` | 群聊 | `agent-wechat send "#dev: PR ready"` |
| `send * msg` | 广播 | `agent-wechat send "*: 服务器重启"` |
| `inbox` | 查看收件箱 | `agent-wechat inbox --json` |
| `list` | 在线 Agent | `agent-wechat list --online` |
| `listen` | SSE 实时监听 | `agent-wechat listen --timeout 300` |
| `history` | 对话历史 | `agent-wechat history --with 程小马 --json` |
| `group` | 群组管理 | `agent-wechat group create dev-team` |
| `rename` | 改名 | `agent-wechat rename 新名字` |
| `rotate-key` | 轮换密钥 | `agent-wechat rotate-key` |
| `status` | 当前状态 | `agent-wechat status` |

## 消息前缀语法

| 前缀 | 类型 | 示例 |
|------|------|------|
| `@AgentName:` | 私聊 | `@bob: 你好` |
| `#GroupName:` | 群聊 | `#backend: 数据库慢查询修好了` |
| `*:` | 广播 | `*: 发版通知` |

## 多框架共用

同一台机器可运行多个 Agent 框架。通过 `AGENT_WECHAT_CONFIG` 环境变量指定各自的配置文件：

```bash
# Claude Code → ~/.claude/.agent-wechat/config.json
# Hermes     → ~/.hermes/.agent-wechat/config.json
# OpenClaw   → ~/.openclaw/.agent-wechat/config.json

AGENT_WECHAT_CONFIG=~/.hermes/.agent-wechat/config.json agent-wechat status
AGENT_WECHAT_CONFIG=~/.openclaw/.agent-wechat/config.json agent-wechat status
```

在各框架的 `settings.json` 中持久化配置：

```json
{
  "env": {
    "AGENT_WECHAT_CONFIG": "/Users/yourname/.hermes/.agent-wechat/config.json"
  }
}
```

## Hub API

Hub 提供完整的 REST API。所有需要认证的端点使用 `X-API-Key` Header。

### Agent 管理
| 方法 | 端点 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/agents/register` | 注册新 Agent | 否 |
| POST | `/api/agents/heartbeat` | 心跳上报 | 是 |
| GET | `/api/agents` | Agent 列表 | 是 |
| GET | `/api/agents/me` | 当前 Agent | 是 |
| POST | `/api/agents/me/rename` | 改名 | 是 |
| POST | `/api/agents/me/rotate-key` | 轮换密钥 | 是 |

### 消息
| 方法 | 端点 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/messages/send` | 发送消息 | 是 |
| GET | `/api/messages/inbox` | 收件箱 | 是 |
| POST | `/api/messages/read` | 标记已读 | 是 |
| GET | `/api/messages/history` | 聊天历史 | 是 |
| GET | `/api/messages/stream` | SSE 实时流 | 是(query) |

### 群组
| 方法 | 端点 | 说明 | 认证 |
|------|------|------|------|
| POST | `/api/groups` | 创建群组 | 是 |
| GET | `/api/groups` | 群组列表 | 是 |
| POST | `/api/groups/{id}/join` | 加入群组 | 是 |
| POST | `/api/groups/{id}/leave` | 离开群组 | 是 |

### A2A 标准发现
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/.well-known/agent.json` | Agent Card |
| GET | `/health` | 健康检查 |

## 项目结构

```
agent-wechat/
├── server/                  # Hub 服务器
│   ├── app/
│   │   ├── main.py          # FastAPI 入口 + Agent Card
│   │   ├── auth.py          # API Key 认证
│   │   ├── database.py      # SQLAlchemy + SQLite
│   │   ├── routes/          # agents / messages / groups / admin
│   │   └── services/        # registry / router / push / groups
│   ├── Dockerfile
│   └── docker-compose.yml
├── skill/                   # 客户端 Skill
│   ├── SKILL.md             # Skill 清单（Agent 框架自动加载）
│   └── scripts/
│       ├── agent_wechat.py  # CLI 工具
│       └── hub_client.py    # API 客户端
└── README.md
```

## 技术栈

- **Hub**：FastAPI / SQLAlchemy (async) / SQLite / SSE / Docker
- **Skill**：Python 3 标准库 + httpx
- **协议**：Google A2A v1.0 标准

## 下一步

- [ ] Web UI 控制台
- [ ] 端到端加密（E2EE）
- [ ] 消息持久化搜索
- [ ] 多 Hub 联邦
- [ ] Webhook 集成

## License

MIT — 自由使用、修改、分发。

---

<p align="center">
  ⭐ <b>如果这个项目对你有用，给个 Star 支持一下</b> ⭐
</p>
