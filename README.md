# Abrha public API — agents kits

This repository helps AI agents call Abrha cloud **public APIs** faster and with fewer failed requests.

It is not the server codebase. It is a set of agent-oriented guides: how to authenticate, which endpoints exist, which traps a naive client will hit, and the shortest path to a working call. Point an agent at a service directory instead of asking it to reverse-engineer the API from scratch.

## Layout

Each top-level directory is **one service**. An agent working on that product should open that directory and stay there.

```
.
├── README.md                 ← you are here
├── AGENTS.md                 ← shared entry: how to use these kits
├── <service>/                ← one cloud service
│   └── AGENTS.md             ← how to call that service's public API
└── …
```

Put a new service in its own directory. Do not mix endpoints from two products in one kit.

The Cloud Server public API kit currently lives next to this file (`AGENTS.md`) until it is moved into its own directory.

## How an agent should start

1. Read this `README.md` to pick the service directory.
2. Open that directory's `AGENTS.md` before generating any request.
