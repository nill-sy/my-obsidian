---
title: Sub2API 私人中转部署方案
date: 2026-09-20
status: local-running
tags:
  - AI基础设施
  - Sub2API
  - Codex
  - DeepSeek
  - 私人部署
---

# Sub2API 私人中转部署方案

> [!summary] 目标
> 建立一个仅供个人使用的统一模型入口。Codex 始终连接同一个 Sub2API 地址，由 Sub2API 管理 OpenAI/Codex 订阅账号，并为以后接入 DeepSeek、Gemini、Claude、Grok 或其他 API 渠道预留扩展空间。

## 1. 方案结论

采用分阶段部署：

1. 先在 Mac 本机使用 Docker Compose 部署 Sub2API，启用单用户的 `Simple Mode`。
2. 第一阶段只接入一个自己的 GPT Plus/Codex OAuth 账号，验证 Codex Responses、流式输出、工具调用、长对话和额度显示。
3. 验证稳定后，再添加第二个 GPT 账号。对同一会话启用粘性调度，避免在两个账号之间频繁切换。
4. 再接入 DeepSeek 等正规 API Key 渠道，并为每类模型建立独立分组和模型别名。
5. 只有出现多设备访问或需要全天在线时，才迁移到私人 VPS。

不将 Sub2API 作为公共服务，不启用充值、支付、注册和多人分发功能。OAuth Token、数据库密码、JWT 密钥及下游 API Key 均不得写入 Obsidian 或 Git。

## 2. 目标架构

```text
Codex Desktop / Codex CLI / IDE 扩展
                 │
                 │ 固定 Base URL + Sub2API Key
                 ▼
       Sub2API（本机或私人 VPS）
          │          │          │
          │          │          └── 后续其他模型
          │          └── DeepSeek API Key
          └── OpenAI OAuth 订阅账号组
                 ├── GPT 账号 A（主）
                 └── GPT 账号 B（备用）
```

Sub2API 是“订阅账号/API Key → 统一 API”的网关。客户端使用 Sub2API 生成的下游 Key，Sub2API 负责账号鉴权、调度、计量、限流和请求转发。

## 3. 为什么采用 Sub2API

Sub2API 更适合管理订阅账号池：

- 支持 OAuth 与 API Key 上游账号。
- 支持 OpenAI Responses 接口，适配 Codex。
- 支持多账号调度、粘性会话、并发限制和速率限制。
- 可以生成固定的下游 API Key，客户端不用反复退出和登录不同账号。
- 提供管理后台、调用统计和订阅额度观察。
- 后续可将其他模型接到统一入口。

它与 New API 的主要区别是：New API 更侧重聚合正规 API 渠道；Sub2API 的核心场景是把 AI 产品订阅账号的额度包装并分发为 API。

## 4. 重要边界

### 4.1 对话连续性

两个 GPT 账号并不会共享 ChatGPT 云端对话。Sub2API 只能让 Codex 客户端始终使用同一入口，并通过粘性会话让一条对话尽量固定在同一个上游账号。

切换上游账号时，以下服务端状态可能无法完全迁移：

- `previous_response_id`
- 加密推理内容
- Prompt Cache
- 工具调用状态
- 服务端上下文压缩结果

因此，账号 B 应作为额度耗尽或故障时的备用账号，而不应对一条长对话进行轮询负载均衡。

### 4.2 账号与服务条款

Sub2API 项目自身明确提示：将订阅账号额度包装或分发成 API 可能违反部分上游服务商的服务条款，并可能造成账号封禁、服务中断或数据丢失。即使只供个人使用，也不能把这种方式视为 OpenAI 官方支持的标准接入方式。

### 4.3 功能完整性

纯文本、代码和基础工具调用可能接近原生 Codex，但以下功能不能预设为完全等价：

- Codex Cloud
- ChatGPT 工作空间功能
- OAuth 插件和连接器
- 图片生成与附件回传
- Web Search
- Fast/Speed 模式展示
- 新模型目录的即时同步
- Responses WebSocket 与复杂工具历史续接

## 5. 部署阶段

## 阶段 A：本机试运行

### 5.1 运行位置

建议部署目录：

```text
/Users/zhd/IdeaProjects/sub2api-deploy
```

数据目录与 Obsidian 仓库分开，避免数据库、Token 和日志被 Obsidian Git 同步。

### 5.2 组件

Sub2API 的 Docker Compose 部署包含：

- Sub2API
- PostgreSQL 15+
- Redis 7+

### 5.2.1 Docker 在本项目中的作用

Docker 负责把 Sub2API 及其依赖放在相互隔离、可重复启动的 Linux 容器中：

- `sub2api` 容器运行网关后端和管理页面。
- `postgres` 容器保存用户、账号、配置与调用记录等结构化数据。
- `redis` 容器提供缓存、限流和任务状态。
- Docker Compose 统一定义三个容器的版本、启动顺序、健康检查、内部网络、端口和数据挂载。
- Colima 在 macOS 上提供 Linux 虚拟机与 Docker Engine；Docker CLI 和 Compose 负责向它发送管理命令。

容器可以删除和重建，持久数据通过目录挂载保存在 `/Users/zhd/IdeaProjects/sub2api-deploy/deploy` 下。停止或重建容器不会清除这些数据；只有删除 `data`、`postgres_data`、`redis_data` 才会造成数据丢失。

个人使用启用：

```text
RUN_MODE=simple
```

生产式启动时还需要：

```text
SIMPLE_MODE_CONFIRM=true
```

Simple Mode 用于隐藏 SaaS、支付和复杂计费功能，减少个人部署中的无关配置。

### 5.3 安装原则

1. 从官方 GitHub 仓库克隆源码或下载明确的 Release。
2. 固定一个经过验证的版本，不直接长期跟随 `latest`。
3. 使用 `docker-compose.local.yml`，将数据保存在明确的本地目录中。
4. 自动生成强随机的 PostgreSQL 密码、JWT Secret 和 TOTP 加密密钥。
5. `.env` 权限设置为仅当前用户可读，且不得提交 Git。
6. 初期只监听 `127.0.0.1:8080`，不开放局域网和公网。

### 5.4 本机资源规划

个人低并发试运行的规划值：

| 项目 | 建议 |
|---|---:|
| CPU | 2 核可用资源 |
| 内存 | 4 GB 可用资源 |
| 磁盘 | 20–30 GB 独立余量 |
| 日志保留 | 7 天 |
| 数据库备份 | 每日一次，保留 7 份 |

这些是规划值，实际资源取决于长上下文长度、并发数和日志详细程度。

## 阶段 B：接入第一个 GPT 账号

1. 在 Sub2API 管理后台添加 OpenAI OAuth 账号。
2. 通过浏览器完成官方登录与授权。
3. 测试账号连接状态和可用模型。
4. 建立 `openai-codex-primary` 分组。
5. 只加入账号 A，启用粘性会话。
6. 创建一个仅供自己使用的 Sub2API API Key。
7. 限制这个 Key 只能访问需要的 GPT/Codex 模型。

不得将 OpenAI OAuth Token、`auth.json` 或 Sub2API 下游 Key 复制到笔记、Git、工单或聊天中。

## 阶段 C：配置 Codex

当前采用独立 Profile，文件为 `~/.codex/sub2api.config.toml`。这样保留原生 OpenAI 登录作为默认配置，通过 `--profile sub2api` 显式切换到中转站：

```toml
model = "gpt-6-astra"
model_provider = "sub2api"
model_reasoning_effort = "low"

[model_providers.sub2api]
name = "Private Sub2API"
base_url = "http://127.0.0.1:8080/v1"
wire_api = "responses"

[model_providers.sub2api.auth]
command = "/usr/bin/security"
args = ["find-generic-password", "-a", "zhd", "-s", "codex-sub2api-api-key", "-w"]
timeout_ms = 5000
refresh_interval_ms = 0
```

Sub2API 下游 Key 保存在 macOS 登录钥匙串的 `codex-sub2api-api-key` 项中，不以明文写入配置。启动命令：

```bash
codex --profile sub2api
```

最小链路测试命令：

```bash
codex --profile sub2api exec --skip-git-repo-check 'Reply with exactly: SUB2API_OK'
```

2026-09-20 已完成测试，返回 `SUB2API_OK`。该测试覆盖钥匙串取 Key、Sub2API 鉴权、OpenAI OAuth 上游选择和 Responses API 返回。

## 阶段 D：添加第二个 GPT 账号

账号 A 通过至少 7 天的稳定性验证后再添加账号 B：

1. 账号 A 设为主账号。
2. 账号 B 设为备用账号。
3. 同一会话使用稳定的 `session_id` 或网关支持的会话亲和字段。
4. 正常情况下不在每次请求间轮换账号。
5. 只有账号 A 达到额度、被限流或临时不可用时才切换 B。
6. 切换后运行一次长对话与工具调用回归测试。

购买第二个账号主要提升总额度与可用性，不能合并两个账号的云端聊天历史。

## 阶段 E：接入 DeepSeek 等模型

DeepSeek 等模型优先使用供应商正式提供的 API Key：

1. 在网关中建立独立渠道。
2. 使用独立模型分组，例如 `deepseek-code`。
3. 不把 DeepSeek 与 GPT 账号放进同一个随机负载均衡组。
4. 为不同模型设置清晰的别名，避免名称相同但实际模型不同。
5. 分别验证流式输出、上下文长度和工具调用。

如果 Sub2API 当前版本对目标模型或协议适配不足，可以：

- 让 Sub2API 连接一个兼容 OpenAI API 的 DeepSeek 上游；或
- 保留 Sub2API 管理订阅账号，另用 New API/LiteLLM 管理普通 API 模型；或
- 通过 MCP 提供 `ask_deepseek` 一类的辅助模型工具。

不建议未经测试就把 Codex 的 Responses 请求转换为 DeepSeek 的 Chat Completions。复杂工具调用、`developer` 角色、流式事件和推理字段可能无法完整转换。

## 阶段 F：需要时迁移私人 VPS

只有满足以下任一条件时迁移：

- 多台设备需要访问。
- Mac 关机时仍需服务。
- 需要稳定域名和 HTTPS。

VPS 部署要求：

1. Linux 服务器，Docker Compose v2。
2. Sub2API、PostgreSQL、Redis 只放在私有 Docker 网络。
3. 仅反向代理的 443 端口对外开放。
4. 使用 Caddy 或 Nginx 配置有效 HTTPS 证书。
5. 管理后台通过 Tailscale、IP 白名单或额外身份验证限制访问。
6. PostgreSQL 和 Redis 端口禁止暴露到公网。
7. 防火墙只开放 SSH 和 HTTPS；SSH 使用密钥认证。

## 6. 安全基线

- 开启管理员双因素认证。
- 为不同客户端生成不同的 Sub2API Key。
- 下游 Key 只授权必要模型和额度。
- 禁止公开注册、支付和自助充值。
- 默认关闭完整提示词和响应正文日志；只保留时间、模型、Token、费用与错误码。
- 日志保留 7–30 天并自动轮转。
- 所有对外流量使用 HTTPS。
- PostgreSQL、Redis 和管理接口不直接暴露公网。
- `.env`、数据库备份和 OAuth Token 均加密保存。
- 每次升级前备份数据库、配置和版本号。
- 不使用未固定版本的自动升级；升级后执行验收测试。
- 定期撤销不再使用的下游 Key 和 OAuth 授权。

## 7. 备份与恢复

需要备份：

- PostgreSQL 数据库
- Sub2API 配置文件
- `.env` 的加密副本
- 当前镜像/Release 版本号
- 反向代理配置（VPS 阶段）

不建议备份到普通 Git 仓库：

- OAuth Token
- 下游 API Key
- PostgreSQL 原始数据目录
- Redis 数据
- 未脱敏的调用日志

恢复测试至少验证：

1. 新建空目录。
2. 恢复配置和数据库。
3. 启动全部容器。
4. 登录管理后台。
5. 验证账号、分组和 Key 存在。
6. 发送一次无敏感数据的测试请求。

## 8. 验收清单

### 基础服务

- [ ] 容器重启后自动恢复。
- [ ] PostgreSQL 与 Redis 没有暴露公网端口。
- [ ] 管理员账号启用了 2FA。
- [ ] `.env` 权限正确且未被 Git 跟踪。
- [ ] 日志中不记录 OAuth Token 和完整 API Key。

### Codex 兼容性

- [ ] Codex 可以读取模型目录。
- [ ] `POST /v1/responses` 普通请求成功。
- [ ] SSE 流式输出正常结束。
- [ ] Shell/文件工具调用成功。
- [ ] 连续 10 轮对话不丢失工具历史。
- [ ] 大上下文请求不出现网关截断。
- [ ] Codex 重启后可以恢复本地任务。
- [ ] 图片、搜索和插件等非文本功能逐项验证，不预设可用。

### 多账号调度

- [ ] 同一会话连续请求固定使用账号 A。
- [ ] 新会话可以按策略选择账号。
- [ ] 账号 A 限流后能够切换账号 B。
- [ ] 切换后不会把不同会话内容串在一起。
- [ ] 账号恢复后不会造成反复来回切换。

### DeepSeek 等其他模型

- [ ] 模型名称准确映射。
- [ ] 普通流式对话成功。
- [ ] 工具调用经过单独测试。
- [ ] 费用与上游账单基本一致。
- [ ] 不同模型不会共享不兼容的服务端会话状态。

## 9. 运维节奏

### 每周

- 检查账号可用性、额度和异常错误。
- 检查磁盘、数据库和日志增长。
- 检查 Sub2API Release 与安全修复，但不自动升级。

### 每月

- 验证一次备份可恢复性。
- 轮换长期不用的下游 API Key。
- 检查模型映射和价格表。
- 清理过期调用记录。

### 每次升级

1. 记录当前版本和镜像摘要。
2. 备份 PostgreSQL、配置和 `.env`。
3. 阅读 Release Notes 和已知问题。
4. 升级测试实例。
5. 完成 Responses、流式、工具调用和长对话测试。
6. 再升级正式实例；失败时回滚原版本和数据库。

## 10. 推荐实施顺序

```text
本机 Docker 环境检查
  → 部署 Sub2API Simple Mode
  → 安全配置与备份
  → 接入一个 GPT OAuth 账号
  → 配置一个下游 Key
  → 配置 Codex
  → 完成兼容性验收
  → 稳定运行 7 天
  → 添加第二个 GPT 备用账号
  → 添加 DeepSeek API 渠道
  → 根据需要迁移私人 VPS
```

## 11. 当前决策记录

| 决策 | 选择 | 原因 |
|---|---|---|
| 网关 | Sub2API | 需要管理订阅账号，并为多模型扩展预留入口 |
| 初始部署 | Mac 本机 | 成本低、凭据不离开本机、便于验证 |
| 运行模式 | Simple Mode | 个人使用，不需要支付和 SaaS 功能 |
| 数据库 | PostgreSQL + Redis | Sub2API 官方部署结构 |
| GPT 调度 | 主账号 + 备用账号 + 粘性会话 | 减少长对话跨账号状态损失 |
| DeepSeek 接入 | 独立 API 渠道和模型组 | 避免与 GPT 会话和协议混合 |
| 公网开放 | 暂不开放 | 降低攻击面和凭据泄露风险 |

## 12. 本机实际部署记录

> [!success] 2026-09-20 已完成
> Sub2API 已在本机启动，应用、PostgreSQL 和 Redis 三个容器均通过健康检查。管理页面只监听本机地址，不对局域网或公网开放。

### 当前环境

| 项目 | 当前值 |
|---|---|
| 部署目录 | `/Users/zhd/IdeaProjects/sub2api-deploy` |
| 容器运行环境 | Colima 0.10.3，2 CPU / 4 GB 内存 / 30 GB 磁盘 |
| Docker CLI | 29.8.1 |
| Compose | docker-compose 5.5.1 |
| 本机 Compose 文件 | `deploy/docker-compose.personal.yml`（已从官方文件分离） |
| 运行模式 | Simple Mode |
| 管理地址 | `http://127.0.0.1:8080` |
| 管理员邮箱 | `admin@sub2api.local` |
| 对外暴露 | 无，仅监听 `127.0.0.1` |
| 数据目录 | `deploy/data`、`deploy/postgres_data`、`deploy/redis_data` |

本项目使用 Colima 提供 Docker 兼容运行环境，不依赖 Docker Desktop。执行命令前应确认 Docker 上下文为 `colima`，避免误连到其他本机 Docker Engine。

### 第一次登录

浏览器打开：

```text
http://127.0.0.1:8080
```

管理员密码已按本次要求在本机初始化，并保存在权限为 `600` 的 `.env` 文件中。明文密码不写入 Obsidian 或 Git。当前服务只监听 `127.0.0.1`；以后开放到局域网、Tailscale 或公网前，必须更换强密码并开启双因素认证。

### 日常命令

```bash
# 启动容器运行环境
colima start

# 启动 Sub2API
cd /Users/zhd/IdeaProjects/sub2api-deploy/deploy
docker-compose -f docker-compose.personal.yml up -d

# 查看状态
docker-compose -f docker-compose.personal.yml ps

# 查看应用日志
docker-compose -f docker-compose.personal.yml logs -f sub2api

# 停止 Sub2API，保留全部数据
docker-compose -f docker-compose.personal.yml down

# 停止 Colima
colima stop
```

### 已验证项目

- [x] 本机容器运行环境启动成功。
- [x] PostgreSQL、Redis、Sub2API 均为 `healthy`。
- [x] 数据库初始化和管理员创建成功。
- [x] Simple Mode 生效。
- [x] 服务仅监听 `127.0.0.1:8080`。
- [x] 容器 DNS 可解析 GitHub 与 OpenAI 域名。
- [x] 模型价格表已通过网络同步，从内置 203 个更新为 239 个模型。
- [x] 首次登录管理后台。
- [ ] 开启管理员双因素认证。
- [x] 接入第一个 OpenAI OAuth 账号。
- [x] 创建 Codex 专用下游 API Key，并保存到 macOS 钥匙串。
- [x] 创建 `sub2api` Profile 并完成 Responses 最小链路测试。
- [x] 将 Codex 用户级默认 Provider 设为 Sub2API，并完成桌面端同配置链路测试。

### Codex 桌面端使用 Sub2API

> [!info] 2026-09-21 已停用
> 实际使用后决定不再通过 Sub2API 在 ChatGPT 与 DeepSeek 之间切换。Codex 用户级配置已恢复为默认 OpenAI Provider；Sub2API 容器和 Colima 已停止，部署文件、数据库与钥匙串条目暂时保留。

以下配置仅作为历史记录和恢复参考。Codex 桌面端、Codex CLI 和 IDE 扩展共享用户级配置 `~/.codex/config.toml`；此前使用的配置为：

```toml
model = "gpt-6-astra"
model_provider = "sub2api"

[model_providers.sub2api]
name = "Private Sub2API"
base_url = "http://127.0.0.1:8080/v1"
wire_api = "responses"
```

API Key 不写入配置文件，由 Codex 在运行时通过 `/usr/bin/security` 从 macOS 钥匙串服务 `codex-sub2api-api-key` 读取。原始配置备份在：

```text
~/.codex/config.toml.before-sub2api-desktop-20260920
```

需要恢复中转时：

1. 保证 Colima 和 Sub2API 容器正在运行。
2. 完全退出并重新打开 Codex 桌面端。
3. 新建一个任务；新任务会通过 `http://127.0.0.1:8080/v1` 请求 Sub2API。
4. 无需退出桌面端中的 ChatGPT 账号；登录状态仍可供桌面功能使用。

2026-09-20 曾用该配置完成最小请求，Codex 输出 `provider: sub2api`，并成功返回 `DESKTOP_SUB2API_OK`。

历史上可用以下 Profile 临时绕过中转、直接使用 OpenAI：

```bash
codex --profile openai-direct
```

该回退文件只保存 Provider 和模型设置，不包含凭据。

### 下一步操作

1. 完全退出并重新打开 Codex 桌面端，新任务直接使用 OpenAI。
2. DeepSeek 在其官方客户端或单独支持 DeepSeek 的客户端中使用，避免把 Codex 的 Provider 切换与多模型聊天混在一起。
3. 暂不删除 `/Users/zhd/IdeaProjects/sub2api-deploy`、Docker 数据卷和钥匙串条目，观察一段时间后再决定是否彻底清理。

## 13. 参考资料

- [Sub2API GitHub](https://github.com/Wei-Shaw/sub2api)
- [Sub2API 中文 README](https://github.com/Wei-Shaw/sub2api/blob/main/README_CN.md)
- [OpenAI Codex 身份验证](https://developers.openai.com/zh-Hans/docs/auth)
- [OpenAI Codex 自定义模型提供商](https://developers.openai.com/zh-Hans/docs/config-file/config-advanced)
- [OpenAI Codex MCP](https://developers.openai.com/zh-Hans/docs/extend/mcp)
