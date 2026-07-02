# Baize

[中文](./README.md) | [English](./README.en.md)

Baize 是面向多模型调用的 AI 网关，用统一入口管理模型渠道、协议转换、智能调度、请求队列、用量核算、成本统计和诊断日志。

演示地址：[https://baize.cloudshift.cn](https://baize.cloudshift.cn)

它把模型接入、协议转换、渠道调度、用量结算和诊断日志收敛到一个网关里。业务侧可以继续使用熟悉的 API 形态，网关侧负责处理供应商差异、上游波动、调用成本和排障线索。

> [!IMPORTANT]
> Baize 仅面向合法、授权的 AI 网关、组织级鉴权、多模型管理、用量分析、成本核算和私有部署场景。
>
> 使用者必须合法取得上游 API Key、账号、模型服务和接口权限，并遵守上游服务条款以及所在司法辖区的法律法规。
>
> 如果将本项目作为面向公众的生成式 AI 服务或 API 转售服务运营，使用者应自行完成备案、许可、内容安全、实名、日志留存、税务、支付和上游授权等合规义务。

## 为什么需要 Baize

单个模型服务接入成本不高，真正的复杂度来自多模型、多账号、多团队长期运行后的维护成本：协议差异会进入业务代码，上游限流和账号状态会影响可用性，不同业务和用户会争抢同一批模型资源，请求失败需要解释，调用成本也需要和实际账单对齐。

Baize 将这些变化收敛到网关层，让应用侧保持稳定的模型调用边界。

- 稳定调用边界：应用侧不直接绑定供应商差异，协议适配和上游端点选择由网关处理。
- 降低故障扩散：限流、超时、5xx、认证失败和余额异常会进入重试、降级、熔断或跳过流程。
- 入口流量可控：通过请求队列、优先级、泳道、用户和渠道级并发 / 速率限制，降低突发流量对关键调用的影响。
- 解释调度决策：选路参考实时负载、延迟、错误率、心跳和熔断状态，并记录选择、跳过和降级原因。
- 对齐用量成本：预扣、结算、退款、用户价格、渠道成本和使用明细分开记录。
- 支持部署演进：小规模单体运行，规模上来后可拆分控制面和数据面。

## 为什么不是另一个 one-api

one-api 解决了多渠道聚合和后台管理问题。Baize 继续保留这类能力，但重点转向 AI 网关的数据面质量：请求如何进入、协议如何转换、渠道如何选择、故障如何降级、账务如何结算、日志如何解释。

| 维度 | 常见聚合器做法 | Baize 做法 |
| --- | --- | --- |
| 转发策略 | 以 OpenAI 兼容转发为主，特殊渠道补分支 | 透传优先，需要跨协议时才转换 |
| 协议边界 | 转换逻辑容易散落在各渠道适配器 | `EntryPoint -> Endpoint -> Inlet -> Outlet` 统一建模 |
| 适配器 | 一个接口管 URL、Header、转换、发送、响应、模型列表 | 适配器只声明名称、鉴权、心跳和端点 |
| 调度 | 主要依赖优先级、权重、重试和自动禁用 | 结合 pending、EWMA 延迟、首字延迟、错误率、心跳和熔断 |
| 健康状态 | 上游失败后改渠道状态或等待人工处理 | 配置态和运行态分离，临时故障进入降级 / 熔断 / 恢复 |
| 流式响应 | 各协议处理器直接写 SSE | 网关统一写流、做逐事件安全检查、日志和用量聚合 |
| 诊断 | 知道请求失败 | 解释渠道为什么被选中、跳过、降级或熔断 |
| 部署边界 | 单体后台 + 转发 | 单体可用，也支持控制面 / 数据面拆分 |

核心取舍很简单：**渠道适配器越小，网关链路越统一，长期维护成本越低。**

## 核心能力

- 🌐 **多渠道 AI 网关**：OpenAI、Azure OpenAI、Anthropic、Gemini、Vertex AI、DashScope、Doubao、DeepSeek、Moonshot、Mistral、Minimax、Ollama、SiliconFlow、StepFun、Together AI、Cohere、Cloudflare Workers AI、腾讯混元、讯飞星火、智谱、百度千帆等。
- 🔌 **多入口 API**：OpenAI Chat Completions、OpenAI Responses、Anthropic Messages、Gemini generateContent / streamGenerateContent / Interactions、Embeddings、Images、Audio、Rerank、视频与图片异步任务。
- 🔁 **透传与协议转换**：OpenAI 兼容上游优先直连，OpenAI Chat / Responses、Anthropic Messages、Gemini 之间按显式协议矩阵转换。
- 🧩 **声明式适配器**：渠道只声明上游端点、鉴权、心跳请求和必要的请求 / 响应转换，不接管整条中转生命周期。
- 🛤️ **统一中转链路**：真实请求、测试渠道和心跳检测复用同一组端点、鉴权、请求构造、响应校验和错误分类抽象。
- 🚦 **精细化流量管理**：支持请求优先级、泳道隔离、用户和渠道级并发 / 速率限制，分别控制入口压力和上游出站能力。
- ⚖️ **自适应调度**：结合权重、pending 请求数、EWMA 延迟、首字延迟、错误率、近期失败、心跳和熔断状态选择渠道。
- 💓 **运行态健康**：使用类 Phi Accrual 的心跳判断和运行时熔断，避免一次上游抖动永久改坏渠道配置。
- 🌊 **流式响应管线**：服务端统一写 SSE，逐事件执行响应安全检查、日志采样和用量聚合，减少渠道分支绕过公共逻辑。
- 💰 **成本与账务**：用户侧价格和渠道侧成本分离，支持文本、图片、音频、视频、工具等能力类型的预扣、结算、退款和明细。
- 🔎 **上游模型发现**：按渠道声明的 `models` 端点拉取上游模型，支持保存渠道和编辑表单临时探测。
- 🧾 **审计与诊断**：请求日志、使用明细、渠道诊断事件、运行时熔断状态、响应时间和错误信息可追踪。
- 🏗️ **控制面 / 数据面拆分**：支持单体运行，也支持 `--cp` 控制面和 `--dp` 数据面模式。
- 🔐 **账号授权渠道**：支持 Codex / OpenAI OAuth 和 Gemini OAuth 等需要账号授权的渠道形态。

## 快速开始

### 从源码运行

依赖：

- Go 1.26.4+
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

未设置 `SQL_DSN` 时，服务默认使用 SQLite。本地启用内置 PostgreSQL 时会使用内置 PostgreSQL；生产或多实例部署建议使用外部 PostgreSQL，并配置 Redis 作为运行时缓存和配置同步组件。

### 使用 docker compose

仓库提供 `docker-compose.yml` 作为本地构建部署模板。上线前至少修改：

- PostgreSQL 用户、密码和 `SQL_DSN`。
- 数据和日志挂载路径。

```bash
docker compose up -d --build
```

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

```mermaid
flowchart LR
  A[客户端入口<br/>EntryPoint] --> B[入口协议识别]
  B --> C[渠道选择<br/>ResolvePlan]
  C --> D[上游端点规划]
  D --> E[入站转换<br/>Endpoint.Inlet]
  E --> F[上游请求<br/>ChannelTransport.DoRequest]
  F --> G[出站转换<br/>Endpoint.Outlet]
  G --> H[响应 / 用量 / 日志 / 计费]
```

## 设计原则

Baize 把模型调用中的入口、调度、协议、计费和诊断统一放在网关层处理，让多上游模型调用更稳定、可控、可解释：

- 运行态调度：结合配置权重、实时延迟、错误率、心跳和熔断状态选择更合适的渠道。
- 平滑健康探测：使用类 Phi Accrual 的心跳检测，降低短暂抖动造成的误判。
- 配置与运行态分离：手动配置保持稳定，临时故障通过运行时降级、熔断和恢复处理。
- 错误语义分类：对限流、超时、认证失败、余额不足、请求错误等场景采取不同策略。
- 账务事实独立：将预扣、结算、退款和统计从请求日志中拆出，便于审计和对账。
- 可解释诊断：记录渠道选择、跳过、降级、熔断和恢复原因，方便管理员定位问题。
- 可拆部署边界：支持单体运行，也支持控制面和数据面分离部署。

## 架构边界

Baize 的适配器设计围绕一个目标：新增渠道时优先声明能力，而不是复制一套完整的中转流程。

适配器保留四类职责：

```go
type Adaptor interface {
    Name() string
    Auth() wireapi.Auth
    Heartbeat() wireapi.Heartbeat
    Endpoints() []wireapi.Endpoint
}
```

这四个方法分别描述：

- `Name()`：渠道名称。
- `Auth()`：上游请求鉴权方式。
- `Heartbeat()`：测试渠道和心跳检测使用的最小请求。
- `Endpoints()`：渠道支持的上游接口。

传统适配器很容易把转换、发送、响应处理、计费和日志都写进渠道内部，最后变成一个什么都管的 `God Interface`。Baize 将公共链路收回网关：入口识别、路由选择、协议转换、HTTP 发送、响应归一、用量结算和日志审计走统一流程。适配器只保留供应商差异，例如上游地址、鉴权方式、请求格式和响应出口。

如果把协议转换逻辑塞进入口处理器或渠道适配器，后续成本会按“入口协议 × 上游协议 × 流式 / 非流式 × 工具 / 多模态”膨胀：新增一个入口协议时，每个渠道都要知道怎么接；新增一个上游协议时，又要补多条入口到上游的转换路径。更麻烦的是，日志、计费、安全检查、错误分类和 usage 聚合也会跟着协议分支分散，最后变成某些路径记录完整、某些路径漏字段，某些错误能正确分类、某些错误只能按普通失败处理。

Baize 把问题拆成两层：渠道适配器只声明“这个供应商有哪些上游端点”，协议转换只发生在 `Endpoint.Inlet` 和 `Endpoint.Outlet` 两个边界。这样新增供应商时，不需要让渠道理解所有入口协议；新增协议方向时，也不需要改每个渠道的中转流程。上游请求、流式写出、日志、计费、安全检查和错误分类仍然走同一条网关主链路，协议越多，越能避免把复杂度摊到每个适配器里。

更详细的设计说明见：

- [中转架构边界](./docs/relay-architecture-boundaries.md)
- [控制面 / 数据面拆分](./docs/control-data-plane-split.md)
- [运行时边界](./docs/runtime-boundaries.md)

## 协议转换

文本与生成类接口按“客户端入口 -> 上游端点 -> 客户端响应”的方向转换。图片、音频、视频、Rerank 等能力按渠道声明的端点直连或走各自能力链路，不放进这个文本协议转换范围。

请求路径：

```mermaid
flowchart LR
  A[客户端入口] --> B[上游端点选择]
  B --> C[请求转换]
  C --> D[上游入站转换]
```

| 客户端入口 | OpenAI Chat 上游 | OpenAI Responses 上游 | Anthropic Messages 上游 | Gemini generateContent 上游 | Gemini streamGenerateContent 上游 |
| --- | --- | --- | --- | --- | --- |
| OpenAI Chat | 直连 | 转换 | 转换 | 转换 | 流式转换 |
| OpenAI Responses | 转换 | 直连 | 转换 | 转换 | 流式转换 |
| Anthropic Messages | 转换 | 转换 | 直连 | 转换 | 流式转换 |
| Gemini generateContent | 转换 | 转换 | 转换 | 直连 | 不适用 |
| Gemini streamGenerateContent | 流式转换 | 流式转换 | 不适用 | 不适用 | 直连 |
| Gemini Interactions | 转换 | 转换 | 转换 | 不适用 | 不适用 |

普通响应路径：

```mermaid
flowchart LR
  A[上游出站响应] --> B[上游协议]
  B --> C[客户端入口协议转换]
```

| 上游端点 | OpenAI Chat 客户端 | OpenAI Responses 客户端 | Anthropic Messages 客户端 | Gemini generateContent 客户端 | Gemini Interactions 客户端 |
| --- | --- | --- | --- | --- | --- |
| OpenAI Chat | 直连 | 转换 | 转换 | 转换 | 转换 |
| OpenAI Responses | 转换 | 直连 | 转换 | 转换 | 转换 |
| Anthropic Messages | 转换 | 转换 | 直连 | 转换 | 转换 |
| Gemini generateContent | 转换 | 转换 | 转换 | 直连 | 不适用 |

流式响应路径：

```mermaid
flowchart LR
  A[上游流式响应] --> B[上游协议]
  B --> C[客户端入口协议流式转换]
```

| 上游端点 | OpenAI Chat 客户端 | OpenAI Responses 客户端 | Anthropic Messages 客户端 | Gemini streamGenerateContent 客户端 | Gemini Interactions 客户端 |
| --- | --- | --- | --- | --- | --- |
| OpenAI Chat | 直连 | 转换 | 转换 | 转换 | 转换 |
| OpenAI Responses | 转换 | 直连 | 转换 | 转换 | 转换 |
| Anthropic Messages | 转换 | 转换 | 直连 | 不适用 | 转换 |
| Gemini streamGenerateContent | 转换 | 转换 | 转换 | 直连 | 不适用 |

Gemini Interactions 目前是客户端入口形态，可以由 OpenAI Chat、OpenAI Responses 或 Anthropic Messages 上游承载，不作为非 Gemini 渠道的上游目标。

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

欢迎提交 issue、设计讨论和 pull request。贡献前建议先阅读 [架构边界](#架构边界) 和 [中转架构边界](./docs/relay-architecture-boundaries.md)，避免把渠道差异写成新的大接口。

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

本项目基于 [one-api](https://github.com/songquanpeng/one-api) 二次开发，one-api 使用 MIT 许可证。上游 MIT 版权和许可声明、第三方依赖声明、前端基础项目声明见 [NOTICE](./NOTICE)、[THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md) 和 [web/LICENSE](./web/LICENSE)。

如果你的组织无法满足 AGPLv3 的网络服务开源义务，可以联系项目维护者获取商业授权。无论采用哪种授权方式，仍需保留 one-api、前端基础项目和第三方依赖的许可声明。
