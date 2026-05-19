# Auth Session

A reusable, self-hosted **session-based authentication service** built with FastAPI and PostgreSQL. Drop it into any project that needs register / login / logout with email confirmation out of the box.

---

## Features

- **Register & login** — create an account, receive a confirmation e-mail, then log in
- **Email confirmation** — account stays inactive until the user clicks the link (24 h expiry)
- **Session management** — `HttpOnly` + `Secure` cookies, server-side session storage, 30-day TTL
- **Logout** — invalidates the session server-side and clears the cookie
- **Current user endpoint** — returns profile + all active sessions
- **Email delivery** — Resend API (Gmail SMTP kept as fallback)
- **Rate limiting** — register (5/min), login & confirm (10/min)
- **Security headers** — `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy`
- **Background tasks** — Celery workers powered by RabbitMQ

---

## Tech stack

| Layer | Technology |
|---|---|
| API | FastAPI |
| Database | PostgreSQL 17, SQLAlchemy 2.0 async, Alembic |
| Task queue | Celery 5, RabbitMQ 3.13 |
| Cache / results | Redis 7 |
| Email | Resend API |
| Rate limiting | slowapi |
| Runtime | Python 3.14, UV |
| Containers | Docker Compose |

---

## API endpoints

| Method | Path | Auth | Description |
|---|---|:---:|---|
| `GET` | `/` | — | Health check |
| `POST` | `/auth/register` | — | Create account, send confirmation e-mail |
| `GET` | `/auth/confirm-email?token=…` | — | Activate account |
| `POST` | `/auth/login` | — | Login → sets `session_token` cookie |
| `POST` | `/auth/logout` | ✓ | Invalidate session, clear cookie |
| `GET` | `/users/get-me` | ✓ | Current user + active sessions |

---

## Installation

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- [UV](https://docs.astral.sh/uv/getting-started/installation/) (for local development)

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd auth-session
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in the required values:

```env
# PostgreSQL
POSTGRES_USER=postgres_user
POSTGRES_PASSWORD=your_password
POSTGRES_DB=postgres_db
DATABASE_URL=postgresql+asyncpg://postgres_user:your_password@postgres:5432/postgres_db

# Resend (https://resend.com — free tier available)
RESEND_API_KEY=re_your_api_key_here
SENDER_EMAIL=you@yourdomain.com

# App
BASE_URL=http://localhost:8000
ORIGINS=["http://localhost:3000"]
```

> `SENDER_EMAIL` must belong to a domain verified in your Resend account.
> For quick testing you can use `onboarding@resend.dev`.

### 3. Run with Docker Compose

```bash
# Build and start all services (API, Celery worker, Postgres, RabbitMQ, Redis)
docker compose up --build

# Run the database migration (first time only)
docker compose --profile tools run --rm migrate
```

The API is now available at **http://localhost:8000**.
Interactive docs: **http://localhost:8000/docs**

### 4. Stop

```bash
docker compose down

# Also remove persistent volumes (wipes the database)
docker compose down -v
```

---

## Local development (without Docker)

```bash
# Install dependencies
uv sync --group dev

# Start infrastructure (Postgres, RabbitMQ, Redis)
docker compose up postgres rabbitmq redis -d

# Run the migration
uv run alembic upgrade head

# Start the API
uv run uvicorn src.main:app --reload

# In a second terminal — start the Celery worker
uv run celery -A src.celery_app.celery_main:celery_app worker --loglevel=info
```

---

## Project structure

```
auth-session/
├── src/
│   ├── main.py                  # App factory, middleware, router registration
│   ├── logger.py                # Logging config
│   ├── core/
│   │   ├── config.py            # Env-var loader
│   │   ├── database.py          # Async SQLAlchemy engine + session
│   │   ├── security.py          # hash_token utility
│   │   └── limiter.py           # slowapi Limiter instance
│   ├── auth/                    # Register · login · logout · confirm-email
│   ├── user/                    # User model, schemas, profile endpoint
│   ├── session/                 # Session model and lifecycle
│   └── celery_app/              # Celery config + email tasks
├── alembic/                     # Database migrations
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

---

## License

[MIT](LICENSE)
