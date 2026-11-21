<div align="center">

<img src="./assets/logo.svg" width="96" height="96" alt="Red Queen Ark logo" />

# Red Queen Ark

**An AI assistant for business, family, and personal life.**

One assistant, one memory, one set of settings — in a terminal, in a browser, or on a server you host yourself.

[Documentation](https://rqark.docs.ansoncar.com) · [Introduction](https://rqark.docs.ansoncar.com/introduction/) · [Quick Start](https://rqark.docs.ansoncar.com/quick-start/)

</div>

---

## What this is

Red Queen Ark is meant to hold more than one part of your life: meetings, bookkeeping, customer replies, groceries, schedules. It is not a coding tool with extra features — memory matters more than code intelligence, continuity more than speed, and a record of what was done more than a clever answer.

It runs where you put it. The same engine serves a terminal client, a browser client, and a service you control. Everything stays in `~/.rqark/` until you connect a data cloud; after that, sessions, agents and settings follow you across devices — hosted by you, if you prefer.

## How the parts fit

```
rqark-tui  ─┐
            ├──▶  rqark-app  ──▶  rqark-core  ──▶  model provider
web chat  ──┘         │
                      └──▶  rqark-data-cloud   (accounts, sessions, agents, memory)
```

| Layer | Repository | Role |
| --- | --- | --- |
| Engine | `rqark-core` | Stateless: the turn loop, tool calls, live prompts, one streaming event interface |
| Runtime | `rqark-app` | Sessions that resume, providers that fail over, agents, memory |
| Service | `rqark-agent-cloud` | Chat WebSocket, speech, scheduled tasks, sandboxed workspace |
| Data | `rqark-data-cloud` | Accounts, groups, API keys and stored data — the source of truth for identity |
| Interfaces | `rqark-tui`, web clients | Terminal and browser fronts for the same system |

## Repositories

| Repository | Language | What it is |
| --- | --- | --- |
| [rqark-core](https://github.com/Red-Queen-Ark/rqark-core) | Python | A stateless agent engine. The `openai` package gives you a client; this adds the loop on top of it — same protocol, same streaming, nothing replaced. |
| [rqark-app](https://github.com/Red-Queen-Ark/rqark-app) | Python | Session, provider and agent management on top of `rqark-core`, with an optional data cloud for memory, todos and settings. |
| [rqark-agent-cloud](https://github.com/Red-Queen-Ark/rqark-agent-cloud) | Python | The HTTP and WebSocket service over `rqark-app`: chat, ASR, voice, tasks, sandboxed workspace. |
| [rqark-data-cloud](https://github.com/Red-Queen-Ark/rqark-data-cloud) | Go | Accounts, groups, API keys and per-account data. Gin, GORM, Postgres, Redis. |
| [rqark-tui](https://github.com/Red-Queen-Ark/rqark-tui) | Python | Terminal client built on Textual: one session per project directory, agents from `~/.rqark/agents/`. |
| [rqark-sandbox](https://github.com/Red-Queen-Ark/rqark-sandbox) | Python | The isolated environment an agent runs in when it is not on your own machine. |

## Quick start

One compose file brings up the whole stack: the data cloud with its Postgres and Redis, the agent cloud, the task worker, and the web console. Every image is published on GHCR, so nothing is built locally.

```bash
mkdir rqark && cd rqark
curl -fsSLO https://raw.githubusercontent.com/Red-Queen-Ark/.github/main/docker-compose.yml
docker compose up -d
```

Then open <http://localhost:7778>. The first account is created by the data cloud itself:

```bash
docker compose exec data-cloud createsuperuser
```

The compose file as it stands:

```yaml
services:

  cloud-web-ui:
    image: ghcr.io/red-queen-ark/rqark-cloud-web-ui:0.27.2
    container_name: rqark-cloud-web-ui
    environment:
      - DATA_CLOUD_URL=http://data-cloud:7302
      - AGENT_CLOUD_URL=http://agent-cloud:8000
    ports:
      - "7778:80"
    depends_on:
      - data-cloud
      - agent-cloud
    networks:
      - rqark-project
    restart: unless-stopped

  agent-cloud:
    image: ghcr.io/red-queen-ark/rqark-agent-cloud:0.17.4
    container_name: rqark-agent-cloud
    environment:
      - DATA_CLOUD_URL=http://data-cloud:7302
      - REDIS_SERVER=redis
      - REDIS_PORT=6379
      - SANDBOX_IMAGE=${SANDBOX_IMAGE:-ubuntu:24.04}
      - WORKSPACE_MAX_BYTES=${WORKSPACE_MAX_BYTES:-104857600}
      - WORKSPACE_START_TIMEOUT=${WORKSPACE_START_TIMEOUT:-30}
      - WORKSPACE_SHELL_MAX_BYTES=${WORKSPACE_SHELL_MAX_BYTES:-8388608}
      - IMAGE_MAX_BYTES=${IMAGE_MAX_BYTES:-5242880}
      - IMAGE_MAX_PER_MESSAGE=${IMAGE_MAX_PER_MESSAGE:-4}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    mem_limit: ${MEM_LIMIT_AGENT:-4g}
    ports:
      - "8000:8000"
    depends_on:
      - data-cloud
    stop_grace_period: 90s
    networks:
      - rqark-project
    restart: unless-stopped

  task:
    image: ghcr.io/red-queen-ark/rqark-task:0.1.5
    container_name: rqark-task
    environment:
      - DATA_CLOUD_URL=http://data-cloud:7302
      - REDIS_SERVER=redis
      - REDIS_PORT=6379
      - INTERNAL_TOKEN=8380e319e3855bd1513fb4c7b1224e04c0f420b924a98515ad2d87bbd87c55ca
      - RQARK_MEMORY_AUTO_WRITE=${RQARK_MEMORY_AUTO_WRITE:-0}
      - TASK_TICK_SECONDS=${TASK_TICK_SECONDS:-30}
      - TASK_CONCURRENCY=${TASK_CONCURRENCY:-1}
      - TASK_LEADER_TTL=${TASK_LEADER_TTL:-60}
      - TASK_RUN_TIMEOUT=${TASK_RUN_TIMEOUT:-600}
      - TASK_STREAM_MAX_CHARS=${TASK_STREAM_MAX_CHARS:-200000}
      - SANDBOX_ENABLED=${SANDBOX_ENABLED:-true}
      - SANDBOX_IMAGE=${SANDBOX_IMAGE:-ubuntu:24.04}
      - WORKSPACE_MAX_BYTES=${WORKSPACE_MAX_BYTES:-104857600}
      - WORKSPACE_START_TIMEOUT=${WORKSPACE_START_TIMEOUT:-30}
      - WORKSPACE_SHELL_MAX_BYTES=${WORKSPACE_SHELL_MAX_BYTES:-8388608}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    mem_limit: ${MEM_LIMIT_TASK:-2g}
    depends_on:
      - data-cloud
    stop_grace_period: 90s
    networks:
      - rqark-project
    restart: unless-stopped

  data-cloud:
    image: ghcr.io/red-queen-ark/rqark-data-cloud:0.18.3
    container_name: rqark-data-cloud
    environment:
      - PORT=7302
      - POSTGRES_SERVER=pgsql
      - POSTGRES_PORT=5432
      - POSTGRES_DB=app
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin
      - REDIS_SERVER=redis
      - REDIS_PORT=6379
      - SECRET_KEY=86296d2f95b5fb789f44ef62049e47847a260a4f63fe4c8889003a1fd6b1eff0
      - INTERNAL_TOKEN=8380e319e3855bd1513fb4c7b1224e04c0f420b924a98515ad2d87bbd87c55ca
    ports:
      - "7302:7302"
    depends_on:
      - pgsql
      - redis
    networks:
      - rqark-project
    restart: unless-stopped

  redis:
    image: redis:6
    container_name: rqark-redis
    volumes:
      - redis_data:/data
    networks:
      - rqark-project
    restart: unless-stopped

  pgsql:
    image: postgres:16
    container_name: rqark-pgsql
    environment:
      - PGDATA=/var/lib/postgresql/data/pgdata
      - POSTGRES_DB=app
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=admin
    volumes:
      - db_data:/var/lib/postgresql/data/pgdata
    networks:
      - rqark-project
    restart: unless-stopped

volumes:
  db_data:
    name: rqark-data-cloud-db-data
  redis_data:
    name: rqark-data-redis-data

networks:
  rqark-project:
    name: rqark-project
```

The values in this file are for a local trial, not a deployment: `SECRET_KEY` and `INTERNAL_TOKEN` sign sessions and the calls between the services, and the database password is `admin`. Replace all three before the stack is reachable from anywhere but your own machine, and keep the two `INTERNAL_TOKEN` entries identical. Both the agent cloud and the task worker mount `docker.sock` because the sandbox runs as a sibling container next to them.

## Design rules the code keeps

- **Local by default.** Without a cloud service, every local feature works and no data leaves the machine.
- **One system, not one app.** Interfaces own no data; `rqark-data-cloud` does.
- **It does not stop at the first error.** Provider failure, context overflow and long tool calls are handled by switching, compacting and retrying.
- **It shows its work.** Thinking, tool calls, results and the plan stream into the interface as they happen.
- **It is honest about what it does not know.** Fetched pages are graded by source tier.

## Contributing

Each repository carries its own `CONTRIBUTING.md` and `AGENTS.md`; read those before opening a change. Everything here is MIT licensed.
