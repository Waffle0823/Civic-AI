# Civic AI

A complaint management platform with AI-powered analysis. Users submit complaints through a REST API; each complaint is published to Kafka and picked up by an LLM-based analyzer that produces sentiment scores, urgency ratings, categories, and summaries.

## Architecture

```
Client
  │
  ▼
core-api (REST, :8080)
  ├── PostgreSQL  (users, complaints)
  ├── Redis       (JWT session blacklist)
  └── Kafka ──► complaint.created topic
                        │
                        ▼
             complaint-analyzer
                        │
                        ▼
                   vLLM API (external)
```

## Services

### core-api

Rust/Actix-web REST API. Handles user registration, JWT authentication, and complaint submission. Publishes a `complaint.created` event to Kafka for every new complaint.

| | |
|---|---|
| Port | `8080` |
| Docs | `http://localhost:8080/swagger-ui/` |
| Dependencies | PostgreSQL, Redis, Kafka |

**Endpoints**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/users` | — | Register a new user |
| `POST` | `/auth/login` | — | Login, returns JWT |
| `POST` | `/auth/logout` | Bearer | Revoke JWT |
| `POST` | `/complaints` | Bearer | Submit a complaint |
| `GET` | `/swagger-ui/` | — | Interactive API docs |

### complaint-analyzer

Python Kafka consumer. Reads from the `complaint.created` topic and sends each complaint to a vLLM-backed LLM (OpenAI-compatible API) for analysis — extracting sentiment, urgency, categories, and a summary.

| | |
|---|---|
| Port | None (consumer only) |
| Dependencies | Kafka, vLLM API |

## Getting Started

### Prerequisites

- Docker and Docker Compose v2
- A running vLLM server (or any OpenAI-compatible endpoint) — see [vLLM docs](https://docs.vllm.ai)

### Run

```bash
# Point complaint-analyzer at your LLM server
export LLM_BASE_URL=http://your-llm-host:8000/v1
export LLM_API_KEY=your-key          # omit if not required
export LLM_MODEL=Qwen/Qwen2.5-7B-Instruct

docker compose up --build
```

The API is available at `http://localhost:8080`. Swagger UI at `http://localhost:8080/swagger-ui/`.

Kafka is reachable from the host at `localhost:9094` (EXTERNAL listener).

### Quick test

```bash
# Register
curl -s -X POST http://localhost:8080/users \
  -H 'Content-Type: application/json' \
  -d '{"user_name":"alice","email":"alice@example.com","password":"secret123"}'

# Login
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com","password":"secret123"}' | jq -r '.token')

# Submit a complaint
curl -s -X POST http://localhost:8080/complaints \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "Broken streetlight",
    "content": "The streetlight on Main St has been out for two weeks.",
    "categories": ["infrastructure"]
  }'
```

## Configuration

All services share the same convention: environment variables with an `APP_` prefix and `__` as a nested separator (e.g. `APP_DATABASE__HOST`). A `config.toml` file can be placed in the working directory as an alternative; environment variables take precedence.

### core-api

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_SERVER__HOST` | `127.0.0.1` | Bind address |
| `APP_SERVER__PORT` | `8080` | Bind port |
| `APP_SERVER__JWT_SECRET` | `change_me` | JWT signing secret — **change in production** |
| `APP_SERVER__JWT_EXPIRATION_SECONDS` | `3600` | Token lifetime |
| `APP_DATABASE__HOST` | `127.0.0.1` | PostgreSQL host |
| `APP_DATABASE__PORT` | `5432` | PostgreSQL port |
| `APP_DATABASE__USER` | `postgres` | PostgreSQL user |
| `APP_DATABASE__PASSWORD` | — | PostgreSQL password |
| `APP_DATABASE__DATABASE` | `postgres` | Database name |
| `APP_REDIS__HOST` | `127.0.0.1` | Redis host |
| `APP_REDIS__PORT` | `6379` | Redis port |
| `APP_REDIS__PASSWORD` | — | Redis password |
| `APP_KAFKA__BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka brokers |
| `RUST_LOG` | `error` | Log filter (e.g. `info`, `debug`) |

### complaint-analyzer

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_KAFKA__BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka brokers |
| `APP_KAFKA__TOPIC` | `complaints` | Topic to consume |
| `APP_KAFKA__OUTPUT_TOPIC` | `complaint.analyzed` | Topic to publish analysis results |
| `APP_KAFKA__GROUP_ID` | `complaint-analyzer` | Consumer group |
| `APP_KAFKA__AUTO_OFFSET_RESET` | `earliest` | Offset reset policy (`earliest`, `latest`, `none`) |
| `APP_KAFKA__ENABLE_AUTO_COMMIT` | `false` | Whether to auto-commit offsets |
| `APP_KAFKA__SESSION_TIMEOUT_MS` | `30000` | Consumer session timeout |
| `APP_KAFKA__MAX_POLL_INTERVAL_MS` | `300000` | Max time between polls before rebalance |
| `APP_KAFKA__POLL_TIMEOUT_S` | `1.0` | Poll call timeout in seconds |
| `APP_LLM__BASE_URL` | `http://localhost:8000/v1` | LLM API base URL (OpenAI-compatible) |
| `APP_LLM__API_KEY` | `none` | LLM API key |
| `APP_LLM__MODEL` | `Qwen/Qwen2.5-7B-Instruct` | Model to use |
| `APP_LLM__MAX_TOKENS` | `512` | Max tokens per response |
| `APP_LLM__TEMPERATURE` | `0.1` | Sampling temperature |
