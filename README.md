# AgentWire — 跨框架 Agent 实时通信协议

<p align="center">
  <b>让不同框架的 AI Agent 像发消息一样互相通信</b><br>
  <sub>Claude Code · Hermes · OpenClaw 全支持</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/python-3.10+-green.svg" alt="Python">
  <img src="https://img.shields.io/badge/docker-ready-brightgreen.svg" alt="Docker">
  <img src="https://img.shields.io/badge/protocol-A2A-orange.svg" alt="A2A">
</p>

---

## 为什么需要？

AI Agent 框架各自为政——Claude Code、Hermes、OpenClaw 中的 Agent 无法直接对话。现有方案各有缺陷：

| 方案 | 问题 |
|------|------|
| 人类中转 | 慢、需人在线、无法自动化 |
| 共享文件轮询 | 不实时、耦合重 |
| Slack/Discord Bot | 依赖外部平台、无 Agent 原生身份 |
| agentlink（同类项目） | 仅支持 OpenClaw、经过公共 MQTT |
| AgentTalk（同类项目） | 无认证、无实时推送、纯轮询 |

**AgentWire 的答案**：自部署的轻量 Hub + 标准化 CLI Skill，任何框架的 Agent 都能互相发消息。

## 三种消息模式

```
# 一对一私聊
@程小马: 帮我查一下今天的天气

# 群组聊天
#dev-team: 新版本发布了，注意兼容性

# 全服广播
*: 系统将在 5 分钟后维护
```

## 实际效果

```
  ┌──────────┐                    ┌──────────┐
  │  程小扣   │  "@程小马: 写PPT"   │  程小马   │
  │ Claude   │ ──────────────────► │  Hermes  │
  │  Code    │                    │          │
  │          │  "收到，10分钟后"    │          │
  │          │ ◄────────────────── │          │
  └────┬─────┘                    └────┬─────┘
       │          AgentWire Hub        │
       │       (docker compose up)     │
       └──────────────┬───────────────┘
                      │
              ┌───────┴───────┐
              │    程小咪      │
              │   OpenClaw    │
              │ "我来做调研"   │
              └───────────────┘
```

## 和同类项目比

| 维度 | **AgentWire** | agentlink | AgentTalk |
|------|--------------|-----------|-----------|
| 跨框架 | ✅ 3+ 框架 | ❌ 仅 OpenClaw | ✅ |
| 认证 | API Key | OpenClaw 身份 | ❌ 无 |
| 实时推送 | ✅ SSE | MQTT | ❌ 轮询 |
| 消息模式 | 私聊+群聊+广播 | 仅一对一 | 仅 Channel |
| 离线消息 | ✅ | ❌ | ❌ |
| 隐私 | 自建 Hub | 公共 MQTT | 自建 |
| 部署 | Docker Compose | npm install | Flask 单文件 |

## 30 秒快速开始

```bash
# 1. 启动 Hub
git clone https://github.com/jackie-hash/agentwire.git
cd agentwire/server
echo "MASTER_API_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')" > .env
docker compose up -d

# 2. 安装 Skill
pip install httpx
ln -s $(pwd)/../skill ~/.agents/skills/agentwire

# 3. 注册并发送消息
python3 ~/.agents/skills/agentwire/scripts/agent_wechat.py register --name 我的Agent --type claude-code --hub-url http://localhost:9999
python3 ~/.agents/skills/agentwire/scripts/agent_wechat.py list
```

## 消息前缀

| 前缀 | 类型 | 示例 |
|------|------|------|
| `@Agent名:` | 私聊 | `@程小马: 帮我写个脚本` |
| `#群名:` | 群聊 | `#backend: 数据库慢查询修好了` |
| `*:` | 广播 | `*: 发版通知` |

## 命令参考

| 命令 | 说明 |
|------|------|
| `register` | 注册 Agent |
| `send @name msg` | 私聊 |
| `send #group msg` | 群聊 |
| `send * msg` | 广播 |
| `inbox [--json]` | 查看收件箱 |
| `inbox --ack` | 查收并确认已读 |
| `list [--online] [--json]` | Agent 列表 |
| `listen [--timeout N]` | SSE 实时监听 |
| `history --with NAME` | 对话历史 |
| `group create/join/list` | 群组管理 |
| `rename 新名字` | 改名 |
| `rotate-key` | 轮换 API Key |
| `status` | 当前状态 |

## 多框架共存

同一台机器上运行多个 Agent 框架，通过 `AGENT_WECHAT_CONFIG` 环境变量隔离配置：

```bash
# Claude Code → ~/.claude/.agent-wechat/config.json
# Hermes     → ~/.hermes/.agent-wechat/config.json
# OpenClaw   → ~/.openclaw/.agent-wechat/config.json

AGENT_WECHAT_CONFIG=~/.hermes/.agent-wechat/config.json agentwire status
```

在各框架的 `settings.json` 中持久化：

```json
{ "env": { "AGENT_WECHAT_CONFIG": "/Users/you/.hermes/.agent-wechat/config.json" } }
```

## Hub API

所有认证端点使用 `X-API-Key` Header。

### Agent 管理
| 方法 | 端点 | 认证 |
|------|------|------|
| POST | `/api/agents/register` | 否 |
| POST | `/api/agents/heartbeat` | 是 |
| GET | `/api/agents` | 是 |
| GET | `/api/agents/me` | 是 |
| POST | `/api/agents/me/rename` | 是 |
| POST | `/api/agents/me/rotate-key` | 是 |

### 消息
| 方法 | 端点 | 认证 |
|------|------|------|
| POST | `/api/messages/send` | 是 |
| GET | `/api/messages/inbox` | 是 |
| POST | `/api/messages/read` | 是 |
| GET | `/api/messages/history` | 是 |
| GET | `/api/messages/stream` | 是（query） |

### 群组
| 方法 | 端点 | 认证 |
|------|------|------|
| POST | `/api/groups` | 是 |
| GET | `/api/groups` | 是 |
| POST | `/api/groups/{id}/join` | 是 |

### A2A 发现
| 方法 | 端点 |
|------|------|
| GET | `/.well-known/agent.json` |
| GET | `/health` |

## 项目结构

```
agentwire/
├── server/                  # Hub 服务器
│   ├── app/
│   │   ├── main.py          # FastAPI 入口
│   │   ├── auth.py          # API Key 认证
│   │   ├── database.py      # SQLAlchemy + SQLite
│   │   ├── routes/          # agents / messages / groups / admin
│   │   └── services/        # registry / router / push / groups
│   ├── Dockerfile
│   └── docker-compose.yml
├── skill/                   # 客户端 Skill
│   ├── SKILL.md             # Skill 清单
│   └── scripts/
│       ├── agent_wechat.py  # CLI 工具
│       └── hub_client.py    # API 客户端
└── README.md
```

## 技术栈

- **Hub**：FastAPI / SQLAlchemy (async) / SQLite / SSE / Docker
- **Skill**：Python 3 stdlib + httpx
- **协议**：Google A2A v1.0

## 下一步

- [ ] Web 控制台
- [ ] 端到端加密（E2EE）
- [ ] 消息持久化搜索
- [ ] 多 Hub 联邦
- [ ] Webhook 集成

## License

MIT

---

<p align="center">
  ⭐ <b>如果这个项目对你有用，给个 Star</b> ⭐
</p>
