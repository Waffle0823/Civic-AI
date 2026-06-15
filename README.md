# Civic AI

A complaint management platform with AI-powered analysis. Users submit complaints through a REST API or web frontend; each complaint is published to Kafka and processed by an LLM-based pipeline that classifies categories, rates urgency, generates analysis, and creates action tasks for staff.

## Architecture

```
page (:3000)          page-admin (:3001)
     │                       │
     └──────────┬────────────┘
                │
                ▼
          core-api (:8080)
           ├── PostgreSQL  (users, complaints, categories, tasks)
           ├── Redis       (JWT session)
           └── Kafka
                    │ complaint.created
                    ▼
         complaint-analyzer ──► LLM API (external)
                    │ complaint.analyzed
                    ├──────────────────────► core-api (상태·카테고리·긴급도 업데이트)
                    ▼
            task-manager ──► LLM API (external)
                    │ complaint.tasks_ready
                    ▼
               core-api (태스크 자동 생성)
```

## Services

### core-api

Rust/Axum REST API. Handles user registration, JWT authentication, complaint submission, category management, and task tracking. Publishes `complaint.created` events to Kafka, and consumes `complaint.analyzed` (to persist AI analysis) and `complaint.tasks_ready` (to auto-create staff tasks).

| | |
|---|---|
| Port | `8080` |
| Docs | `http://localhost:8080/swagger-ui/` |
| Dependencies | PostgreSQL, Redis, Kafka |

**Endpoints**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/users` | — | Register a new user |
| `GET` | `/users` | Bearer (admin) | List all users |
| `GET` | `/users/me` | Bearer | Get current user profile |
| `GET` | `/users/{id}` | Bearer (admin) | Get user by ID |
| `PATCH` | `/users/me` | Bearer | Update current user |
| `PATCH` | `/users/{id}` | Bearer (admin) | Update user by ID |
| `DELETE` | `/users/{id}` | Bearer (admin) | Delete user by ID |
| `POST` | `/auth/login` | — | Login, returns access + refresh tokens |
| `POST` | `/auth/logout` | Bearer | Revoke access token |
| `POST` | `/auth/refresh` | — | Refresh access token |
| `GET` | `/complaints` | Bearer | List complaints (paginated, role-scoped) |
| `POST` | `/complaints` | Bearer | Submit a complaint |
| `GET` | `/complaints/{id}` | Bearer | Get complaint by ID |
| `PATCH` | `/complaints/{id}` | Bearer | Update complaint |
| `GET` | `/complaints/{id}/tasks` | Bearer | List tasks for a complaint |
| `POST` | `/complaints/{id}/tasks` | Bearer | Create a task for a complaint |
| `GET` | `/complaints/{id}/tasks/{task_id}` | Bearer | Get task by ID |
| `PATCH` | `/complaints/{id}/tasks/{task_id}` | Bearer | Update task |
| `GET` | `/categories` | Bearer | List categories |
| `POST` | `/categories` | Bearer (admin) | Create a category |
| `GET` | `/categories/{id}` | Bearer | Get category by ID |
| `PATCH` | `/categories/{id}` | Bearer (admin) | Update category |
| `DELETE` | `/categories/{id}` | Bearer (admin) | Delete category |
| `GET` | `/swagger-ui/` | — | Interactive API docs |

### complaint-analyzer

Python Kafka consumer. Reads `complaint.created` events, calls an OpenAI-compatible LLM API to classify the category, rate urgency (1–5), and generate an analysis summary in Korean. Publishes results to `complaint.analyzed`.

| | |
|---|---|
| Port | None (consumer only) |
| Dependencies | Kafka, LLM API |

### task-manager

Python Kafka consumer. Reads `complaint.analyzed` events and calls an OpenAI-compatible LLM API to generate 2–5 concrete action tasks for staff. Publishes results to `complaint.tasks_ready`, which core-api picks up to auto-create tasks in the database.

| | |
|---|---|
| Port | None (consumer only) |
| Dependencies | Kafka, LLM API |

### page

User-facing frontend (React + Vite). Allows citizens to register, log in, submit complaints, and track their status.

| | |
|---|---|
| Port | `3000` |
| Dependencies | core-api |

**Pages**

| Path | Description |
|------|-------------|
| `/login` | Login |
| `/register` | Register |
| `/dashboard` | My complaints list |
| `/complaints/new` | Submit a new complaint |
| `/complaints/:id` | Complaint detail and status |

### page-admin

Admin-facing frontend (React + Vite). Provides staff and administrators with a dashboard to manage complaints, users, categories, and tasks.

| | |
|---|---|
| Port | `3001` |
| Dependencies | core-api |

**Pages**

| Path | Description |
|------|-------------|
| `/login` | Admin login |
| `/dashboard` | Overview and statistics |
| `/complaints` | All complaints list |
| `/complaints/:id` | Complaint detail and task management |
| `/users` | User management |
| `/categories` | Category management |
| `/tasks` | Task list |

## Getting Started

### Prerequisites

- Docker and Docker Compose v2
- A running vLLM server (or any OpenAI-compatible endpoint) — see [vLLM docs](https://docs.vllm.ai)

### Run

```bash
# Point complaint-analyzer and task-manager at your LLM server
export LLM_BASE_URL=http://your-llm-host:8000/v1
export LLM_API_KEY=your-key          # omit if not required
export LLM_MODEL=Qwen/Qwen2.5-7B-Instruct

docker compose up --build
```

| Service | URL |
|---------|-----|
| User frontend | `http://localhost:3000` |
| Admin frontend | `http://localhost:3001` |
| REST API | `http://localhost:8080` |
| Swagger UI | `http://localhost:8080/swagger-ui/` |

An admin account is seeded on first startup with the credentials set via `APP_API__ADMIN__*` (default: `admin` / `Admin@123!`).

### Run frontends locally (optional)

If you prefer to run the frontends outside of Docker:

```bash
# User frontend
cd pages/page
npm install
npm run dev        # http://localhost:3000

# Admin frontend
cd pages/page-admin
npm install
npm run dev        # http://localhost:3001
```

Both dev servers proxy `/api/*` requests to `http://localhost:8080` by default.

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
    "priority": 3
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
| `APP_SERVER__JWT_EXPIRATION_SECONDS` | `3600` | Access token lifetime (seconds) |
| `APP_SERVER__JWT_REFRESH_EXPIRATION_SECONDS` | `604800` | Refresh token lifetime (seconds) |
| `APP_DATABASE__HOST` | `127.0.0.1` | PostgreSQL host |
| `APP_DATABASE__PORT` | `5432` | PostgreSQL port |
| `APP_DATABASE__USER` | `postgres` | PostgreSQL user |
| `APP_DATABASE__PASSWORD` | — | PostgreSQL password |
| `APP_DATABASE__DATABASE` | `postgres` | Database name |
| `APP_REDIS__HOST` | `127.0.0.1` | Redis host |
| `APP_REDIS__PORT` | `6379` | Redis port |
| `APP_REDIS__PASSWORD` | — | Redis password |
| `APP_KAFKA__BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka brokers |
| `APP_API__COMPLAINTS__PRIORITY_MIN` | `1` | Minimum urgency value |
| `APP_API__COMPLAINTS__PRIORITY_MAX` | `5` | Maximum urgency value |
| `APP_API__CATEGORIES__NAME_MIN_LENGTH` | `1` | Category name minimum length |
| `APP_API__CATEGORIES__NAME_MAX_LENGTH` | `100` | Category name maximum length |
| `APP_API__TASKS__TITLE_MIN_LENGTH` | `1` | Task title minimum length |
| `APP_API__TASKS__TITLE_MAX_LENGTH` | `200` | Task title maximum length |
| `APP_API__TASKS__DESCRIPTION_MAX_LENGTH` | `2000` | Task description maximum length |
| `APP_API__ADMIN__USER_NAME` | `admin` | Seeded admin username |
| `APP_API__ADMIN__EMAIL` | `admin@example.com` | Seeded admin email |
| `APP_API__ADMIN__PASSWORD` | `Admin@123!` | Seeded admin password — **change in production** |
| `RUST_LOG` | `error` | Log filter (e.g. `info`, `debug`) |

### complaint-analyzer

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_KAFKA__BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka brokers |
| `APP_KAFKA__TOPIC` | `complaint.created` | Topic to consume |
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

### task-manager

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_KAFKA__BOOTSTRAP_SERVERS` | `localhost:9092` | Kafka brokers |
| `APP_KAFKA__TOPIC` | `complaint.analyzed` | Topic to consume |
| `APP_KAFKA__OUTPUT_TOPIC` | `complaint.tasks_ready` | Topic to publish task suggestions |
| `APP_KAFKA__GROUP_ID` | `task-manager` | Consumer group |
| `APP_KAFKA__AUTO_OFFSET_RESET` | `earliest` | Offset reset policy |
| `APP_KAFKA__ENABLE_AUTO_COMMIT` | `false` | Whether to auto-commit offsets |
| `APP_KAFKA__SESSION_TIMEOUT_MS` | `30000` | Consumer session timeout |
| `APP_KAFKA__MAX_POLL_INTERVAL_MS` | `300000` | Max time between polls before rebalance |
| `APP_KAFKA__POLL_TIMEOUT_S` | `1.0` | Poll call timeout in seconds |
| `APP_LLM__BASE_URL` | `http://localhost:8000/v1` | LLM API base URL (OpenAI-compatible) |
| `APP_LLM__API_KEY` | `none` | LLM API key |
| `APP_LLM__MODEL` | `Qwen/Qwen2.5-7B-Instruct` | Model to use |
| `APP_LLM__MAX_TOKENS` | `512` | Max tokens per response |
| `APP_LLM__TEMPERATURE` | `0.1` | Sampling temperature |
