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

## Design rules the code keeps

- **Local by default.** Without a cloud service, every local feature works and no data leaves the machine.
- **One system, not one app.** Interfaces own no data; `rqark-data-cloud` does.
- **It does not stop at the first error.** Provider failure, context overflow and long tool calls are handled by switching, compacting and retrying.
- **It shows its work.** Thinking, tool calls, results and the plan stream into the interface as they happen.
- **It is honest about what it does not know.** Fetched pages are graded by source tier.

## Contributing

Each repository carries its own `CONTRIBUTING.md` and `AGENTS.md`; read those before opening a change. Everything here is MIT licensed.
