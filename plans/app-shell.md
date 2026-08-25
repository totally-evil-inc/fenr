# Plan — Epic #2: App shell (sidebar-03 floating navbar, profile menu, dark mode)

> Status: ready for review

**Decisions (user-confirmed):** nav = minimal (Home + placeholder Settings); drop team-switcher &
notifications popover; default theme mode = `"system"`.

## Context

Issue #2 (https://github.com/…/issues/2). The authenticated area sits behind `_protected.tsx`
(`apps/web/src/routes/_protected.tsx`) which renders nothing shell-like today. We need:

1. A sidebar app shell based on `@blocks-so/sidebar-03` (floating navbar aesthetic)
2. A user-profile menu (signed-in identity + logout)
3. Dark mode done properly for TanStack Start (no next-themes)

Per AGENTS.md, generated block code is raw material → rebuild to project conventions.
Work decomposes into small atomic blocks, each implemented → adversarially reviewed → committed.

## What I found (exploration notes)

- **Guard**: `_protected.tsx` returns `{ session }` via `beforeLoad`; shell slots inside it,
  session passed down through route context — no refetch.
- **Session type**: `Session = typeof auth.$Infer.Session` (`apps/web/src/lib/auth.ts:77`);
  `session.user.{name,email,image}` available.
- **Auth client**: `apps/web/src/lib/auth-client.ts` exports `signOut` (better-auth/react).
  Existing sign-out pattern with Sonner feedback lives in `_protected.index.tsx`.
- **Theme tokens**: `.dark { … }` palette already exists in
  `packages/ui/src/styles/globals.css` incl. full `--sidebar-*` token set mapped via
  `@theme inline`. Components just need semantic-token usage.
- **Store convention**: Zustand + `persist` + `createJSONStorage`, SSR-guarded storage factory —
  see `apps/web/src/lib/stores/demo-store.ts` (pattern to follow for theme store).
- **Root doc**: `__root.tsx` `shellComponent` renders `<html><head><HeadContent/>` — FOUC script
  goes here before `<HeadContent />`/styles settle; Toaster already mounted.
- **Block internals** (fetched from blocks.so registry): `Sidebar collapsible="icon"
  variant="floating"`; header (logo + trigger), collapsible sub-nav via Collapsible, footer slot
  (team switcher), notifications popover. Uses framer-motion + lucide → must be replaced
  (hugeicons; CSS transitions instead of framer-motion).
- **UI primitives present**: badge, button, card, input, label, separator, sonner.
  **Missing** (block deps): avatar, collapsible, dropdown-menu, sheet, tooltip, sidebar
  (+ scroll-area required by conventions, not yet installed).
- Style is `base-nova` (Base UI `render` prop pattern) — new primitives will match existing ones.
- Icons: hugeicons only; any generated lucide imports must be swapped and `lucide-react`
  removed if it reappears.

## Approach

Four atomic work blocks, each: implement → review (correctness/concurrency/quality lenses) → commit.

### Block 1 — UI primitives (foundation)

- From `apps/web/`: `bunx --bun shadcn@latest add avatar collapsible dropdown-menu sheet
  tooltip sidebar scroll-area` → shared components land in `packages/ui/src/components/`.
- Swap any generated lucide imports → hugeicons; remove lucide dep if re-added anywhere.
- Verify `bun run check` green; commit.

### Block 2 — Dark mode infrastructure

- `apps/web/src/lib/stores/theme-store.ts`: Zustand persist store, mode union
  `"light" | "dark" | "system"` (default `"system"`), versioned storage key `fenr.theme`,
  SSR-safe storage factory (demo-store pattern).
- FOUC inline blocking script in `__root.tsx` `<head>`: reads persisted mode from localStorage,
  resolves system preference, sets/removes `class="dark"` on `<html>` pre-paint. try/catch-wrapped,
  no-op on server. React must not fight the imperative class (apply outside React state /
  suppress on `<html>`).
- Small headless hook/provider that syncs store ↔ DOM class and subscribes to
  `prefers-color-scheme` while mode = `system` (listener cleanup on unmount/mode change).
- Optional polish: meta `theme-color` sync.
- Unit-test store resolution logic (`theme.test.ts`: light/dark/system/expired/corrupt values).

### Block 3 — Shell composition

- `apps/web/src/components/shell/`:
  - `app-shell.tsx` — composition root (`SidebarProvider` + sidebar + inset + outlet area)
  - `app-sidebar.tsx` — floating sidebar (`variant="floating"`, `collapsible="icon"`),
    logo header + trigger; no footer extras
  - `nav-main.tsx` — collapsible nav rendering from config
  - `nav-config.ts(x)` — routes as data: `{ id, title, icon, path, subs? }`; hugeicons icons.
    **Confirmed scope:** Home (`/`) + a Settings placeholder; collapsible sub-nav code stays
    generic so future items register in one place
- Active-route state from TanStack Router (`useMatchRoute`/router state), not pathname parsing.
- Nav list overflow → `@workspace/ui/components/scroll-area`.
- No framer-motion — collapse/expand handled by sidebar's own CSS; block's fade dropped or CSS'd.
- Block extras (team switcher footer, notifications popover) **dropped** per decision; sidebar
  header keeps logo + theme toggle slot + SidebarTrigger.
- `_protected.tsx` renders `<AppShell>` around `<Outlet/>` (guard flow untouched, redirect param
  preserved). Demo index page keeps working inside the shell.
- Mobile: sheet-based navigation via sidebar's built-in responsive behavior; verify.

### Block 4 — Profile menu + theme toggle chrome

- `components/shell/user-menu.tsx`: avatar trigger (image fallback initials from name/email),
  name + email display, logout item.
- Logout: `signOut()` from auth-client → navigate `/auth/sign-in`; Sonner success/failure
  ("Signed out" / actionable failure); stale-session edge case still navigates away.
- Theme toggle (light/dark/system) in the navbar/sidebar header using the theme store.
- Defensive: null/missing user fields fall back gracefully (email → initials).

## Files to modify

- `packages/ui/src/components/` — new: avatar, collapsible, dropdown-menu, sheet, tooltip,
  sidebar, scroll-area (shadcn-generated, hugeicons-swapped)
- `packages/ui/package.json` — new radix/base-ui deps as CLI adds them
- `apps/web/src/routes/__root.tsx` — FOUC script, theme sync provider
- `apps/web/src/lib/stores/theme-store.ts` — new (+ test)
- `apps/web/src/components/shell/*` — new shell components
- `apps/web/src/routes/_protected.tsx` — render AppShell
- `apps/web/src/routes/_protected.index.tsx` — adjust to live inside shell (drop local
  sign-out button once profile menu lands)

## Reuse

- Store pattern: `apps/web/src/lib/stores/demo-store.ts` (persist + SSR-safe storage)
- Sign-out + toast pattern: `apps/web/src/routes/_protected.index.tsx#SignOutButton`
- Session via route context: `_protected.tsx` `beforeLoad` return value
- Tokens: existing `.dark` palette + `--sidebar-*` vars in `packages/ui/src/styles/globals.css`

## Steps

- [x] Block 1: primitives installed, lucide-free, check green → review → commit
- [x] Block 2: theme store + FOUC script + system listener + tests → review → commit
- [x] Block 3: AppShell + sidebar + nav config + integration → review → commit
- [x] Block 4: profile menu + theme toggle → review → commit
- [x] Final: full acceptance pass (SSR hydration clean, prod build, grep for hard-coded colors)

## Verification

- Per block: `bun run check` + `bun test`
- Manual dev-server exercise: `bun --bun vite dev` — sign in → shell renders; toggle themes;
  reload mid-toggle (no flash); system-mode OS switch applies live; collapse sidebar; mobile width
  sheet nav; sign out from menu → redirected with toast; unauthenticated visit redirects with
  `redirect` param intact
- Prod: `bun run build` + `turbo run start`, repeat smoke checks
- Grep audit: no `lucide-react`, no hard-coded Tailwind colors in shell code, no console.log


## Outcome (2026-08-24)

- Branch `feat/issue-2-app-shell`, commits 9e7b786 / c8ac30a / e004fec.
- Blocks 2+3 landed as one atomic commit (theme files were uncommitted when shell integration completed); reviewed together.
- Verified: bun run check green; 28 tests pass; vite build green; Bun.serve SSR smoke (unauth / → 307 with redirect param; auth / renders floating sidebar shell, profile menu, theme toggle; sign-out API 200 then / → 307).
- Known P2s left open: hard-coded #ffffff initial theme-color meta bootstrap; sidebar collapse state not persisted across reloads (cookie unused); browser-side hydration-warning confirmation pending a real-browser pass.
