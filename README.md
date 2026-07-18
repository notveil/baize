# Baize

[中文](./README.md) | [English](./README.en.md)

Baize 是面向多模型调用的 AI 网关。它将模型接入、协议转换、流量调度、用量结算和诊断日志收敛到统一入口，让应用继续使用熟悉的 API，同时由网关处理供应商差异和上游波动。

演示地址：[https://baize.cloudshift.cn](https://baize.cloudshift.cn)

> [!IMPORTANT]
> Baize 仅面向合法、授权的 AI 网关、组织级鉴权、多模型管理、用量分析、成本核算和私有部署场景。
>
> 使用者必须合法取得上游 API Key、账号、模型服务和接口权限，并遵守上游服务条款以及所在司法辖区的法律法规。
>
> 如果将本项目作为面向公众的生成式 AI 服务或 API 转售服务运营，使用者应自行完成备案、许可、内容安全、实名、日志留存、税务、支付和上游授权等合规义务。

## 为什么需要 Baize

接入单个模型并不难，难的是让多个模型、账号和团队长期稳定地运行。协议差异、上游限流、流量竞争、故障排查和成本核算，最终都会成为业务侧的维护负担。

Baize 将这些变化收敛到网关层：

- **统一调用**：应用不直接绑定供应商，由网关完成协议适配和端点选择。
- **稳定运行**：通过队列、限流、重试、降级、心跳和熔断控制故障影响。
- **智能调度**：结合实时负载、延迟、成本和错误率选择渠道，并记录决策原因。
- **准确计费**：分开记录预扣、结算、退款、用户价格、渠道成本和使用明细。
- **平滑扩展**：既可单体部署，也可按需拆分控制面和数据面。

> **Baize 试图用正确的边界，避免再次用错误的方式解决这些问题。**

## 设计取舍

Baize 不把每个渠道做成一套独立的中转流程，而是把公共能力收回网关，只让适配器声明供应商差异。

| 维度 | 常见做法 | Baize |
| --- | --- | --- |
| 转发 | 以 OpenAI 兼容转发为主，不断增加渠道特判 | 同协议优先透传，跨协议时显式转换 |
| 适配器 | 同时负责 URL、鉴权、转换、发送和响应 | 只声明名称、鉴权、心跳和端点 |
| 调度 | 主要依赖优先级、权重和重试 | 结合负载、延迟、成本、错误率、心跳和熔断 |
| 健康状态 | 临时故障直接改变渠道配置 | 配置态与运行态分离，自动降级和恢复 |
| 流式响应 | 各渠道分别处理 SSE | 网关统一写流、安全检查、日志和用量聚合 |
| 部署 | 管理后台与转发固定在同一进程 | 支持单体，也支持控制面 / 数据面拆分 |

## 核心能力

- 🌐 **多渠道与多协议**：支持 OpenAI、Azure OpenAI、Anthropic、Gemini、Vertex AI、AWS Bedrock、DashScope、Doubao、DeepSeek 等渠道，以及 OpenAI Chat / Responses、Anthropic Messages、Gemini、Embedding、图片、音频、Rerank 和异步任务入口。
- 🔁 **协议透传与转换**：同协议请求优先透传；跨协议转换通过明确的入口和端点关系完成。
- 🧩 **声明式适配器**：渠道只声明端点、鉴权、心跳和必要的供应商差异，不接管完整中转生命周期。
- 🚦 **流量与调度**：支持请求优先级、泳道、用户和渠道级并发 / 速率限制，以及基于运行状态的自适应选路。
- 🧠 **上下文优化**：请求过长且令牌启用优化时，可缩短常见表达、合并重复内容并截断超长文本，减少无效上下文和 Token 消耗。
- 💓 **健康与容错**：通过心跳、运行时降级、熔断、重试和恢复处理上游波动，不让临时故障污染持久配置。
- 💰 **成本与账务**：用户价格与渠道成本分离，支持预扣、结算、退款和使用明细。
- 🧾 **审计与诊断**：记录请求、用量、响应时间、上游错误以及渠道选择、跳过和降级原因。
- 🏗️ **灵活部署**：支持单体运行，也支持 `pilot` 控制面和 `proxy` 数据面分开部署。

## 快速开始

### 从源码运行

依赖：

- Go 1.26.5+
- Node.js 20+
- pnpm 9+

```bash
git clone git@github.com:notveil/baize.git
cd baize

cd web
pnpm install
pnpm run build
rm -rf build
cp -R dist build

cd ..
go run ./cmd --port 3000 --log-dir ./logs
```

打开 `http://localhost:3000`，按页面提示初始化 root 账号。当前版本不依赖固定默认密码。

### 本地 Docker 构建

```bash
docker build -t baize:local .

docker run --name baize -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -e LOG_DIR=/app/logs \
  -v "$PWD/data:/data" \
  -v "$PWD/logs:/app/logs" \
  baize:local
```

源码开发未设置 `SQL_DSN` 时默认使用 SQLite；Linux 安装包默认使用内置 PostgreSQL；容器部署必须配置外部数据库。生产或多实例部署建议使用外部 PostgreSQL，并配置 Redis 作为运行时缓存和配置同步组件。

### 使用 docker compose

仓库提供 `docker-compose.yml` 作为本地构建部署模板。上线前至少修改：

- PostgreSQL 用户、密码和 `SQL_DSN`。
- 数据和日志挂载路径。

```bash
docker compose up -d --build
```

### 使用 Helm

Kubernetes 上可通过 Helm 分别部署 `pilot` 控制面和 `proxy` 数据面；Chart 不部署 all-in-one、数据库或 Redis。配置和安装示例见 [helm/README.md](helm/README.md)。

### 构建 Linux 安装包

需要本机已有 `pnpm`、`nfpm` 和 `curl`。

```bash
bash scripts/build-linux-packages.sh
```

产物输出到：

```text
dist/packages
```

会生成 `baize`、`baize-pilot`、`baize-proxy` 的 `amd64` / `arm64`，`deb` / `rpm` 包。

## API 使用

在控制台添加渠道和令牌后，可以把 Baize 当作 OpenAI 兼容 API Base 使用：

```bash
export OPENAI_API_KEY="sk-your-baize-token"
export OPENAI_BASE_URL="http://localhost:3000/v1"

curl "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

也可以使用 Anthropic Messages 或 Gemini 兼容入口，具体取决于你在控制台配置的模型、渠道和令牌权限。

## 常用环境变量

| 变量 | 说明 |
| --- | --- |
| `PORT` | 监听端口，默认 `3000` |
| `HOST` | 监听地址，默认 `0.0.0.0` |
| `SQL_DSN` | 数据库连接串；生产建议使用 PostgreSQL |
| `REDIS_CONN_STRING` | Redis 连接串；多实例和配置同步建议启用 |
| `LOG_DIR` | 日志目录，默认 `./logs` |
| `CONFIG_SYNC_INTERVAL_SECONDS` | 配置同步间隔 |
| `CONFIG_TOKEN_INVALID_TTL_SECONDS` | 无效令牌缓存时间，默认 `300` 秒 |
| `CRITICAL_TOKEN_LIMIT` | 单 IP 无效令牌尝试上限，默认 `60` 次；设为 `0` 或负数关闭限制 |
| `CRITICAL_TOKEN_LIMIT_DURATION_SECONDS` | 无效令牌统计窗口，默认 `600` 秒 |
| `CHANNEL_REQUEST_TIMEOUT` | 上游渠道请求超时，单位秒 |
| `CHANNEL_PROXY` | 全局渠道代理 |
| `CHANNEL_TEST_PROMPT` | 测试渠道使用的提示词 |
| `CHANNEL_TEST_USER_AGENT` | 测试渠道和心跳检测使用的 User-Agent |
| `CHANNEL_HEARTBEAT_INTERVAL_SECONDS` | 渠道心跳间隔 |
| `CHANNEL_HEARTBEAT_TIMEOUT_SECONDS` | 渠道心跳超时 |
| `CHANNEL_DIAGNOSTICS_RETENTION_SECONDS` | 渠道诊断事件保留时间 |
| `DEMO_MODE` | 演示模式，禁止提交修改和中转请求 |

更多配置可以参考 [config/config.go](./config/config.go) 和 [.env.example](./.env.example)。

## 部署模式

默认直接运行一个进程，适合本地和小规模私有部署。规模上来后可以拆成：

- `baize`：单体模式，管理后台和中转同进程。
- `baize-pilot --cp`：控制面，负责管理后台、配置、账务和后台任务。
- `baize-proxy --dp`：数据面，负责中转流量、调度、鉴权和运行态。

多实例部署建议使用外部 PostgreSQL，并配置 Redis 做配置同步和运行态缓存。

## 架构总览

![Baize AI 网关架构总览](./docs/ai-gateway-overview.png)

## 开发

```bash
go test ./...

cd web
pnpm install
pnpm run dev
```

常用目录：

- `channel/wireapi`：协议、端点、请求响应和错误模型。
- `channel/adaptor`：渠道适配器能力声明、鉴权和心跳请求。
- `channel/service`：中转请求、响应、计费、日志和运行时状态。
- `routes` / `router`：HTTP API 和控制台接口。
- `web`：Vue / Element Plus 控制台。

## 贡献

欢迎提交 issue、设计讨论和 pull request。贡献前建议先阅读 [中转架构边界](./docs/relay-architecture-boundaries.md)，避免把渠道差异写成新的大接口。

开发约定：

- 新增渠道时优先声明 `Endpoints()`、鉴权和必要的请求 / 响应转换，避免把完整中转流程塞进适配器。
- 协议转换、计费、审计、安全检查应尽量复用网关统一链路。
- 涉及请求链路、账务、诊断或运行态调度的改动，需要补充对应测试或说明验证方式。
- 文档、Docker、Linux 包等分发变更需要保留 `LICENSE`、`NOTICE`、`THIRD_PARTY_NOTICES.md` 和 `web/LICENSE`。

## 许可证

Baize 采用 [GNU Affero General Public License v3.0](./LICENSE) 授权，并适用 [NOTICE](./NOTICE) 中列明的 AGPLv3 Section 7 附加条款。

修改版本在提供用户界面时，应在显著的关于、法律、页脚或归属位置保留 Baize 项目归属和原项目链接：

```text
Baize is an open-source AI gateway derived from one-api.
https://github.com/notveil/baize
```

第三方依赖声明和前端基础项目声明见 [NOTICE](./NOTICE)、[THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md) 和 [web/LICENSE](./web/LICENSE)。

如果你的组织无法满足 AGPLv3 的网络服务开源义务，可以联系项目维护者获取商业授权。无论采用哪种授权方式，仍需保留前端基础项目和第三方依赖的许可声明。
