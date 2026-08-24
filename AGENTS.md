# AGENTS.md — Fenr

Guidance for AI coding agents working in this repository.

## Stack

| Concern          | Library                                        |
| ---------------- | ---------------------------------------------- |
| Runtime          | **Bun only** (`bun`, `bun --bun` for Vite)     |
| Monorepo         | Turborepo + Bun workspaces                     |
| App framework    | TanStack Start (`apps/web`)                    |
| UI components    | shadcn in `packages/ui` (`@workspace/ui`)      |
| Styling          | Tailwind CSS v4                                |
| Icons            | hugeicons (`@hugeicons/react` + `@hugeicons/core-free-icons`) |
| Server data      | TanStack Query                                 |
| Global state     | Zustand                                        |
| Forms            | TanStack Form + Zod                            |
| Lint / format    | Biome (strictly — never ESLint/Prettier)       |
| Tests            | `bun test`                                     |

## Hard rules

### 1. Bun toolchain only

Never invoke `npm`, `npx`, `pnpm`, `yarn`, or the `node` binary directly.
Bun is the runtime, package manager, script runner and test runner.

```sh
bun install            # install (single root bun.lock)
bun run dev            # turbo → bun --bun vite dev
bun run build          # turbo → bun --bun vite build
turbo run start        # apps/web/server.ts — custom Bun.serve production server
bun run test           # bun test
bun run check          # biome check --write + typecheck
```

- All Vite invocations must be prefixed with `bun --bun` so Vite executes on
  Bun's runtime instead of Node.
- Do not add `engines.node`, `.npmrc`, or npm/pnpm config fields to any
  package.json.
- Production serving uses `apps/web/server.ts` — a Bun-native `Bun.serve`
  server that serves `dist/client` assets and falls back to TanStack Start's
  SSR handler (`dist/server/server.js`). Do not add Nitro or Node-based
  servers; plain `vite build` already emits the correct output.

### 2. Data fetching → TanStack Query

All network requests made outside of app runtime (route loaders, components,
effects) MUST go through TanStack Query:

- Use `queryClient.ensureQueryData(...)` in route loaders for SSR prefetching;
  the router's `setupRouterSsrQueryIntegration` handles dehydration/hydration.
- Use `useSuspenseQuery` / `useQuery` in components. Never fetch with bare
  `fetch()` inside effects, and never store server data in Zustand.
- Create QueryClients via `apps/web/src/lib/query-client.ts#createQueryClient`
  (one per request on the server).

### 3. Global state → Zustand

All global state — in-memory, sessionStorage, and localStorage — lives in
Zustand stores under `apps/web/src/lib/stores/`:

- Persisted state uses the `persist` middleware (`createJSONStorage`);
  choose `localStorage` vs `sessionStorage` in the storage factory.
- Never mirror query results into stores; never use React context as a
  global-state mechanism.

### 4. Forms → TanStack Form + Zod

Every form uses TanStack Form validated by a Zod schema (Standard Schema):

- Schemas live in `apps/web/src/lib/schemas/` and are passed directly as
  validators (e.g. `validators: { onChange: schema }`). No adapters needed.
- Shared schemas are the single source of truth; derive types with
  `z.infer`.

### 5. Components & styling

- Shared UI lives in `packages/ui` (`@workspace/ui`). Add components by
  running `bunx shadcn@latest add <component>` from `apps/web/` — the CLI
  routes shared components into `packages/ui` automatically. Import them as
  e.g. `import { Button } from "@workspace/ui/components/button"`.
- `components.json` keeps `"iconLibrary": "lucide"` purely for CLI
  compatibility. **After every `shadcn add`, replace any generated lucide
  imports with hugeicons** (`HugeiconsIcon` from `@hugeicons/react` +
  icon exports from `@hugeicons/core-free-icons`) and remove the lucide
  dependency if it reappears. `lucide-react` must not end up in any
  package.json or import statement.
- Styling is Tailwind v4 (CSS-first config in `packages/ui/src/styles/globals.css`).

### 6. Code quality

- Biome only. Run `bun run check` before finishing any change; keep it green.
- Generated files are excluded from lint/format: `routeTree.gen.ts`,
  `.output/`.
- Tests live next to sources as `*.test.ts(x)` and run with plain `bun test`.

### 7. Skills

Load and follow these skills when the task matches them:

- `/vercel-react-best-practices` — React/Next.js-style performance patterns for React code
- `/logging-best-practices` — logging (wide events / canonical log lines)
- `/animation-vocabulary` — naming motion effects when discussing animations
- `/apple-design` — gesture-driven UI, springs, materials, motion feel
- `/fixing-motion-performance` — jank, layout thrashing, compositor properties
- `/improve-animations` — animation audits and improvement plans
- `/transitions-dev` — production-ready CSS transitions
- `/shadcn` — anything involving shadcn components or registries
- `/ui-ux-pro-max` — UI/UX design intelligence (styles, palettes, fonts, UX rules)
- `/emil-design-eng` — component design polish and micro-interaction details

## Styling convention

- All design tokens and component CSS live in `packages/ui/src/styles/globals.css`
  (`@layer base`, `@layer components`, `@layer utilities`).
- Components must **not** use hard-coded Tailwind values (e.g. `bg-white`,
  `text-gray-900`). Instead use CSS variable tokens mapped through
  `@theme inline` (e.g. `bg-background`, `text-foreground`, `border-border`).
- This keeps theming (light/dark mode) centralized and makes refactoring safe.
- **Custom ScrollArea**: avoid native browser scrollbars. All scrollable
  containers (chat feeds, thread lists, dropdown menus, side panels) MUST use
  `@workspace/ui/components/scroll-area` (Radix/shadcn `<ScrollArea />`).

## Working with shadcn/ui Blocks

Throughout the lifetime of this project we will periodically install `shadcn`
blocks into the repository. Do **not** treat generated block code as
production-ready architecture. Generated code is a reference implementation —
consider it raw material.

Rebuild it from scratch using the project's architectural conventions:

- Organize files and folders by feature/domain rather than mirroring the
  generated output.
- Extract reusable UI into shared components where appropriate.
- Create shell/layout components when a component is primarily responsible
  for composition.
- Separate presentation from business logic.
- Move state management, effects, and event handling into custom hooks whenever
  they improve clarity or reusability.
- Extract utility functions into appropriate utility modules.
- Remove duplication aggressively and improve naming.
- Ensure components have clear, single responsibilities.
- Follow existing project conventions before introducing new patterns.
- **Reusability**: extract UI elements (buttons, cards, empty states, dialogs,
  drawers, etc.) if they have a reasonable chance of being reused.
- **Maintainability**: favor small focused components, predictable file
  structures, minimal prop drilling, composition over configuration, and
  readability. Avoid "God components".
- **Functional equivalence**: rebuilt implementations must preserve the
  original functionality and user experience unless there is a clear
  improvement (e.g. accessibility, performance, cleaner state, types).

## Layout

```
apps/web        TanStack Start app (Bun + custom Bun.serve prod server)
packages/ui     @workspace/ui — shadcn components, Tailwind v4 styles
biome.json      root lint/format config
turbo.json      build/dev/typecheck/test tasks
```
