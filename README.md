# moco-dev-server

A complete Moco platform on your machine for building and testing workflows: one compose
file, published images only. You need Docker with Compose v2.24 or later, and Node.js 18+ for
the `moco` CLI; you do not need a checkout of the moco repo.

| Service       | Image                              | Role                                           |
|---------------|------------------------------------|------------------------------------------------|
| `moco-web`    | `mocoflow/moco-web:<version>`      | UI, docs, CLI download; proxies the API        |
| `moco-runtime` | `mocoflow/moco-runtime:<version>`  | API server + workflow worker                   |
| `moco-init`   | `mocoflow/moco-runtime:<version>`  | one-shot: generates this machine's crypto keys |
| `pgvector`    | `pgvector/pgvector:pg17`           | moco database (+ vectors for RAG)              |
| `temporal`    | `temporalio/temporal`              | workflow runtime (dev server, SQLite)          |
| `rabbitmq`    | `rabbitmq:4-management`            | debug events                                   |
| `langfuse-web`, `langfuse-worker` | `langfuse/langfuse:3`, `langfuse/langfuse-worker:3` | tracing UI; moco-runtime exports traces to it |
| `postgres`, `clickhouse`, `minio`, `redis` | (official images)  | Langfuse's own stores                          |

`moco-web` is published on `127.0.0.1:8080`, Langfuse on `127.0.0.1:3000` (and its media
store, minio, on `127.0.0.1:9090`).

## Quick start

```bash
git clone https://github.com/moco-workflow/moco-dev-server.git
cd moco-dev-server
docker compose up -d
docker compose ps          # wait until moco-runtime is "healthy" (first start: ~30s)
```

Open http://localhost:8080:

- `/ui` — workflow UI
- `/docs` — documentation
- `/tools/` — the `moco` CLI tarball
- `/apps/playground`, `/apps/traceview`, `/apps/moco-agent`

Auth is disabled: every request runs as `admin` (the seeded global administrator;
`MOCO_AUTH_DISABLED_USER_ID` picks another user), whatever credential it carries. The UIs open
straight to their dashboards and the CLI's `local` profile, its default, works without
`moco login`. Keep it on a machine nobody else can reach: anyone who can connect to port 8080
is an administrator.

To enable auth as in production, set `MOCO_AUTH_ENABLED=true` in `.env`. Every request then
needs either a Google ID token (OAuth login) or a moco API key. The UIs and the CLI's `local`
profile send the seeded guest API key, so they run as `guest` without a login.

Langfuse is at http://localhost:3000. Its first start creates a user `dev@moco.local` /
`moco-dev` and a project `moco-dev` with the API keys `pk-lf-moco-dev` / `sk-lf-moco-dev`;
moco-runtime is already configured with them, so every run's trace shows up there and in
traceview's Langfuse panel.

## The CLI

The dev server serves the CLI it was built with at http://localhost:8080/tools/ (the listing
shows the version):

```bash
curl -O http://localhost:8080/tools/moco-cli-1.0.0.tgz
npm i -g moco-cli-1.0.0.tgz
```

There is nothing to configure. The CLI's default profile, `local`, points at
http://localhost:8080/mcp and comes with a session for `admin`, so there is no
`moco profile use` or `moco login` step:

```bash
moco session show               # Working Namespace: admin
moco validate my.wfspec.yaml    # local schema check, no server needed
moco run my.wfspec.yaml         # runs on the dev server, as admin
moco publish                    # from a moco.json project; publishes into namespace admin
```

The first `moco publish` initializes the session with the server, which creates the `admin`
namespace; after that it is an ordinary publish.

To use the hosted service as well, switch profile and log in there:
`moco profile use my-moco && moco login` (`moco profile use local` switches back). The
[quick start](http://localhost:8080/docs/quick-start) walks through a first workflow on this
dev server.

## Configuration

Settings go in a `.env` file next to `docker-compose.yml`. Compose reads it, and `moco-runtime`
also loads it as environment. All of them are optional. Connection strings, crypto keys,
the JWT settings and the Langfuse URL are already set in `docker-compose.yml`, so don't put
them in `.env`.

```bash
# ── Images and ports ──
MOCO_VERSION=latest                 # image tag: latest, or a release such as 1.4.0
MOCO_HTTP_PORT=8080                 # host port
MOCO_PUBLIC_URL=http://localhost:8080   # the URL you browse to; change with MOCO_HTTP_PORT
MOCO_WORKER_COUNT=1
# TEMPORAL_VERSION=1.9.1

# ── Auth ──
# MOCO_AUTH_ENABLED=true            # default false: every request runs as admin
# MOCO_AUTH_DISABLED_USER_ID=admin

# ── LLM (openai.*, llama_index.*) — any OpenAI-compatible endpoint ──
# With `--profile ollama`:
# MOCO_LLM_DEFAULT_BASE_URL=http://ollama:11434/v1
# MOCO_LLM_DEFAULT_MODEL_NAME=qwen3:0.6b
# MOCO_LLM_DEFAULT_EMBED_MODEL_BASE_URL=http://ollama:11434/v1
# MOCO_LLM_DEFAULT_EMBED_MODEL_NAME=nomic-embed-text
# Ollama already running on your machine: http://host.docker.internal:11434/v1
# Hosted:
# MOCO_LLM_DEFAULT_BASE_URL=https://api.openai.com/v1
# MOCO_LLM_DEFAULT_MODEL_NAME=gpt-4o-mini
# OPENAI_API_KEY=...

# ── TypeSafe System One (typesafe.*) — a moco-systemone on your machine ──
# TYPESAFE_DEFAULT_BASE_URL=http://host.docker.internal:8002
# TYPESAFE_DEFAULT_MODEL=qwen3:0.6b
# TYPESAFE_DEFAULT_API_KEY=local

# ── Langfuse — on by default, with the seeded keys ──
# LANGFUSE_HTTP_PORT=3000
# LANGFUSE_PUBLIC_KEY=pk-lf-moco-dev    # Langfuse seeds these on first start only:
# LANGFUSE_SECRET_KEY=sk-lf-moco-dev    #   changing them later needs `docker compose down -v`
# MOCO_OTEL_ENABLE_LANGFUSE=false       # stop exporting traces
# MOCO_OTEL_ENABLE_LANGFUSE_OPENAI=false
# MOCO_TRACEVIEW_LANGFUSE_ENABLED=false
```

`host.docker.internal` reaches services running on your machine.

## Optional profile

```bash
docker compose --profile ollama up -d      # local LLM; pulls qwen3:0.6b + nomic-embed-text
```

Pass the same `--profile` flag to `down`.

## Upgrading

```bash
docker compose pull && docker compose up -d
```

The database schema is built into the `moco-runtime` image and created on first start. There
are no migrations. If a new image carries a different schema, `moco-runtime` refuses to start
and logs:

```
[db-bootstrap] ERROR: database 'mocodb' was built from schema '...', but this image carries schema '...'
```

To continue, reset the environment (below), or pin the previous release with `MOCO_VERSION`.

## Reset

```bash
docker compose down -v
```

This deletes the database (published wfspecs, state, secrets), Temporal history and this
machine's crypto keys. A secret encrypted with the old keys cannot be recovered.

## Troubleshooting

- `docker compose logs -f moco-runtime` shows the bootstrap, server and worker logs.
- If port 8080 is already in use, set `MOCO_HTTP_PORT` and `MOCO_PUBLIC_URL` in `.env`.
- To use `psql`, run `docker compose exec pgvector psql -U moco mocodb` (password `moco`).
