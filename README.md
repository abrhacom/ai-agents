# Abrha / Parspack public API — agents kits

This repository helps AI agents call Abrha and Parspack cloud **public APIs** faster and with fewer failed requests.

It is not the server codebase. It is a set of agent-oriented guides: how to authenticate, which endpoints exist, which traps a naive client will hit, and the shortest path to a working call. Point an agent at a service directory instead of asking it to reverse-engineer the API from scratch.

## Layout

Each top-level directory is **one service**. An agent working on that product should open that directory and stay there.

```
.
├── README.md                 ← you are here: pick a service below
├── <service>/                ← one cloud service
│   └── AGENTS.md             ← how to call that service's public API
└── …
```

Put a new service in its own directory. Do not mix endpoints from two products in one kit.

## Services

Every service kit is the file `<service>/AGENTS.md`. These are the kits that exist today:

| Service | Kit to open | What the API does | Base URL |
|---|---|---|---|
| **IaaS** (Cloud Server / VM) | [`iaas/AGENTS.md`](iaas/AGENTS.md) | Create and manage VMs: regions, sizes, images, VPCs, firewalls, SSH keys, actions | `https://my.abrha.net/cserver/api/public/v1/` (same API on `https://my.parspack.com/cserver`) |
| **CDN** (CDN and DNS) | [`cdn/AGENTS.md`](cdn/AGENTS.md) | Order zones, manage DNS records, cache, firewall, SSL, load balancers, page/origin/transform rules, bulk lists | `https://my.parspack.com/cdnapi/` |

Examples of how to pick a kit:

- "Create a server with 4 GB RAM in Tehran" → VM work → open `iaas/AGENTS.md`.
- "Add an `A` record for `example.com` and purge its cache" → zone/DNS work → open `cdn/AGENTS.md`.
- Anything else → there is no kit for it yet; do not guess endpoints from another service's kit.

Each kit also tells you where its machine-readable OpenAPI spec is and lists the places where the spec is known to be wrong.

## How an agent should start

1. Read this `README.md` and pick the service from the table above.
2. Open that service's `AGENTS.md` (for example `iaas/AGENTS.md` or `cdn/AGENTS.md`) before generating any request.
3. Load the OpenAPI spec that the kit points to, then follow the kit's call order and its list of spec defects.
