# Baize

[中文](./README.md) | [English](./README.en.md)

Baize is an AI gateway for multi-model calls. It brings model access, protocol conversion, traffic scheduling, usage accounting, and diagnostic logs behind one entry point, allowing applications to keep familiar APIs while the gateway handles provider differences and upstream volatility.

Demo: [https://baize.cloudshift.cn](https://baize.cloudshift.cn)

> [!IMPORTANT]
> Baize is intended only for lawful and authorized AI gateway, organization-level authentication, multi-model management, usage analytics, cost accounting, and private deployment scenarios.
>
> Users must lawfully obtain upstream API keys, accounts, model services, and interface permissions, and must comply with upstream terms of service and applicable laws.
>
> When operating this project as a public generative AI service or API resale service, users are responsible for all required filing, licensing, content safety, real-name verification, log retention, tax, payment, and upstream authorization obligations in their jurisdiction.

## Why Baize

Connecting one model is not difficult. Keeping multiple models, accounts, and teams running reliably over time is. Protocol differences, upstream rate limits, traffic contention, incident diagnosis, and cost accounting eventually become an application maintenance burden.

Baize keeps these concerns in the gateway layer:

- **Unified access**: applications do not bind directly to providers; the gateway handles protocol adaptation and endpoint selection.
- **Reliable operation**: queues, limits, retries, degradation, heartbeats, and circuit breakers contain failures.
- **Adaptive routing**: channels are selected using live load, latency, cost, and error rates, with the decision reasons recorded.
- **Accurate accounting**: reservations, settlements, refunds, user prices, channel costs, and usage details are recorded separately.
- **Incremental scaling**: start with one process, then split the control plane and data plane when needed.

> **Baize aims to use clear boundaries to avoid solving recurring problems through tightly coupled patches.**

## Design Tradeoffs

Baize does not give every channel its own relay flow. Shared behavior stays in the gateway, while adaptors only declare provider differences.

| Area | Common approach | Baize |
| --- | --- | --- |
| Forwarding | Primarily OpenAI-compatible proxying with growing provider branches | Passthrough for matching protocols; explicit conversion when protocols differ |
| Adaptors | Own URLs, auth, conversion, transport, and responses | Declare only name, auth, heartbeat, and endpoints |
| Routing | Primarily priority, weight, and retries | Combine load, latency, cost, error rate, heartbeat, and breaker state |
| Health | Temporary failures mutate channel configuration | Separate configuration from runtime state, with automatic degradation and recovery |
| Streaming | Each channel handles SSE independently | The gateway owns streaming, safety checks, logs, and usage aggregation |
| Deployment | Console and relay always share one process | Support both all-in-one and control-plane / data-plane deployments |

## Core Capabilities

- 🌐 **Multiple channels and protocols**: supports OpenAI, Azure OpenAI, Anthropic, Gemini, Vertex AI, AWS Bedrock, DashScope, Doubao, DeepSeek, and other channels, with OpenAI Chat / Responses, Anthropic Messages, Gemini, Embeddings, Images, Audio, Rerank, and asynchronous task entry points.
- 🔁 **Passthrough and conversion**: matching protocols are passed through first; cross-protocol conversion follows explicit entry-point and endpoint relationships.
- 🧩 **Declarative adaptors**: channels declare endpoints, authentication, heartbeat behavior, and necessary provider differences without owning the full relay lifecycle.
- 🚦 **Traffic and scheduling**: request priorities, lanes, user/channel concurrency and rate limits, and runtime-aware adaptive routing.
- 🧠 **Context optimization**: when a request is too long and optimization is enabled for the token, Baize can shorten common expressions, merge duplicate content, and truncate oversized text to reduce unnecessary context and token usage.
- 💓 **Health and resilience**: heartbeat detection, runtime degradation, circuit breakers, retries, and recovery handle upstream volatility without polluting persistent configuration.
- 💰 **Cost and accounting**: separates user pricing from channel costs, with reservation, settlement, refund, and detailed usage records.
- 🧾 **Audit and diagnostics**: records requests, usage, latency, upstream errors, and the reasons channels were selected, skipped, or degraded.
- 🏗️ **Flexible deployment**: supports all-in-one mode as well as separate `pilot` control-plane and `proxy` data-plane deployments.

## Quick Start

### From Source

Requirements:

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

Open `http://localhost:3000` and initialize the root account from the setup page. The current version does not rely on a fixed default password.

### Local Docker Build

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

If `SQL_DSN` is not set during source development, Baize uses SQLite by default. Linux packages use bundled PostgreSQL by default, while container deployments must configure an external database. External PostgreSQL and Redis are recommended for production or multi-instance deployments.

### Docker Compose

`docker-compose.yml` is a local build deployment template. Before going online, change at least:

- PostgreSQL credentials and `SQL_DSN`
- data and log volume paths

```bash
docker compose up -d --build
```

### Helm

On Kubernetes, Helm can deploy the `pilot` control plane and `proxy` data plane separately. The Chart does not deploy all-in-one mode, a database, or Redis. See [helm/README.md](helm/README.md) for configuration and installation examples.

### Linux Packages

Requires `pnpm`, `nfpm`, and `curl`.

```bash
bash scripts/build-linux-packages.sh
```

Packages are written to `dist/packages` for `baize`, `baize-pilot`, and `baize-proxy` on `amd64` / `arm64`, both `deb` and `rpm`.

## API Usage

After adding channels and tokens in the console:

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

Anthropic Messages and Gemini-compatible entry points are also available when the configured model, channel, and token permissions allow them.

## Common Environment Variables

| Variable | Description |
| --- | --- |
| `PORT` | listen port, default `3000` |
| `HOST` | listen host, default `0.0.0.0` |
| `SQL_DSN` | database connection string; PostgreSQL is recommended for production |
| `REDIS_CONN_STRING` | Redis connection string; recommended for multi-instance deployments |
| `LOG_DIR` | log directory, default `./logs` |
| `CONFIG_SYNC_INTERVAL_SECONDS` | configuration sync interval |
| `CONFIG_TOKEN_INVALID_TTL_SECONDS` | invalid token cache duration, default `300` seconds |
| `CRITICAL_TOKEN_LIMIT` | invalid token attempt limit per IP, default `60`; set to `0` or lower to disable |
| `CRITICAL_TOKEN_LIMIT_DURATION_SECONDS` | invalid token attempt window, default `600` seconds |
| `CHANNEL_REQUEST_TIMEOUT` | upstream request timeout in seconds |
| `CHANNEL_PROXY` | global upstream proxy |
| `CHANNEL_TEST_PROMPT` | prompt used by manual channel tests |
| `CHANNEL_TEST_USER_AGENT` | User-Agent used by channel tests and heartbeats |
| `CHANNEL_HEARTBEAT_INTERVAL_SECONDS` | channel heartbeat interval |
| `CHANNEL_HEARTBEAT_TIMEOUT_SECONDS` | channel heartbeat timeout |
| `CHANNEL_DIAGNOSTICS_RETENTION_SECONDS` | channel diagnostic event retention |
| `DEMO_MODE` | demo mode; disables mutations and relay requests |

See [config/config.go](./config/config.go) and [.env.example](./.env.example) for more options.

## Deployment Modes

The default is one process, which is enough for local and small private deployments. At larger scale, split it into:

- `baize`: all-in-one mode with console and relay in one process.
- `baize-pilot --cp`: control plane for the console, configuration, billing, and background jobs.
- `baize-proxy --dp`: data plane for relay traffic, routing, auth, and runtime state.

For multi-instance deployments, use external PostgreSQL and configure Redis for config sync and runtime cache.

## Architecture Overview

![Baize AI gateway architecture overview](./docs/ai-gateway-overview.png)

## Development

```bash
go test ./...

cd web
pnpm install
pnpm run dev
```

Key directories:

- `channel/wireapi`: protocol, endpoint, request, response, and error models.
- `channel/adaptor`: channel capability declarations, auth, and heartbeat requests.
- `channel/service`: relay request/response handling, billing, logs, and runtime state.
- `routes` / `router`: HTTP API and console endpoints.
- `web`: Vue / Element Plus console.

## Contributing

Issues, design discussions, and pull requests are welcome. Before contributing, read [relay architecture boundaries](./docs/relay-architecture-boundaries.md) to avoid turning channel differences into another large interface.

Development guidelines:

- When adding a channel, prefer declaring `Endpoints()`, auth, and necessary inlet / outlet conversion instead of putting a full relay flow inside the adaptor.
- Protocol conversion, billing, audit, and security checks should reuse the shared gateway pipeline where possible.
- Changes touching relay flow, billing, diagnostics, or runtime routing should include tests or clear verification notes.
- Distribution changes for docs, Docker, or Linux packages must preserve `LICENSE`, `NOTICE`, `THIRD_PARTY_NOTICES.md`, and `web/LICENSE`.

## License

Baize is licensed under the [GNU Affero General Public License v3.0](./LICENSE), with the AGPLv3 Section 7 additional terms listed in [NOTICE](./NOTICE).

Modified versions that present a user interface must preserve Baize attribution and the original project link in a prominent about, legal, footer, or attribution location:

```text
Baize is an open-source AI gateway derived from one-api.
https://github.com/notveil/baize
```

Third-party dependency notices and frontend base project notices are documented in [NOTICE](./NOTICE), [THIRD_PARTY_NOTICES.md](./THIRD_PARTY_NOTICES.md), and [web/LICENSE](./web/LICENSE).

If your organization cannot comply with AGPLv3 network-service obligations, contact the project maintainers for commercial licensing. Regardless of the authorization model, notices for the frontend base project and third-party dependencies must still be preserved.
