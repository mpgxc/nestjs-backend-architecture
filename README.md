# nestjs-backend-architecture

A [Claude Code](https://claude.com/claude-code) skill with pragmatic architecture and code-style conventions for TypeScript/NestJS backend services.

It picks the lightest architecture level that fits the problem and generates only what that level needs:

| Level | When |
|-------|------|
| 0 — CRUD | No invariants, no integration |
| 1 — Business rules | Invariants, state transitions, limits |
| 2 — Integration / publishing | Vendor calls, events consumed by other services |
| 3 — Distributed flow | Cross-service flows with compensation |

Covers `contexts/` layout, use-cases, CQS with query services, the Result pattern with a global interceptor/filter, abstract-class ports, DDD, outbox, SQS consumers, idempotency, cron workers, vendor ACL, circuit breaker, logging, REST/RFC 9457, transactions, testing and project bootstrap.

## Install

```
/plugin marketplace add mpgxc/nestjs-backend-architecture
/plugin install nestjs-backend-architecture@mpgxc
```

The skill triggers on its own for NestJS code writing, structuring, review or refactor tasks.

## Layout

```
.claude-plugin/          plugin and marketplace manifests
skills/nestjs-backend-architecture/
  SKILL.md               entry point: levels, defaults, orientation
  references/            detailed guides, loaded on demand
evals.json               eval prompts and assertions for the skill
```

## License

MIT
