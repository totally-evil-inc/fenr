# Fenr

Bun-only monorepo: Turborepo + TanStack Start + shadcn/Tailwind v4.

## Stack

- **Bun** — runtime, package manager, script runner, test runner (no Node)
- **Turborepo** — `apps/web` (TanStack Start) + `packages/ui` (`@workspace/ui`)
- **shadcn** components with Tailwind CSS v4, icons via **hugeicons**
- **TanStack Query** for data fetching, **Zustand** for global state, **TanStack Form + Zod** for forms
- **Biome** for lint/format

## Getting started

```sh
bun install
bun run dev        # http://localhost:3000
```

## Commands

| Command          | What it does                                    |
| ---------------- | ----------------------------------------------- |
| `bun install`    | Install dependencies (single root lockfile)      |
| `bun run dev`    | Dev server (turbo → `bun --bun vite dev`)        |
| `bun run build`  | Production build (`dist/client` + SSR handler)   |
| `bun run start`  | Production server (`apps/web/server.ts`, Bun)    |
| `bun run test`   | Run tests with `bun test`                        |
| `bun run check`  | Biome check (fix mode) + typecheck               |
| `bun run format` | Format everything with Biome                     |

See [AGENTS.md](./AGENTS.md) for the conventions enforced in this repo.
