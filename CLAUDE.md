# entity_client - AI Development Guide

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.
> Do not use feature branches for code changes — stay on the current branch.

## Overview

React client library for entity/organization management: an HTTP API client
(`EntityClient`) plus TanStack Query v5 hooks for CRUD on entities, members,
invitations, and API keys. Also ships a `CurrentEntityProvider` context for
workspace selection with per-user `localStorage` persistence.

- **Package**: `@sudobility/entity_client` (published to **GitHub Packages**, not npmjs)
- **License**: BUSL-1.1 · **Package manager**: Bun · **Module format**: ESM (`"type": "module"`)
- **Build output**: `dist/` (ESM with `.d.ts` declarations); treat `package.json` as source of truth for versions.

## Project Structure

```
src/
├── index.ts                        # Public barrel exports (all hooks, client, types)
├── hooks/
│   ├── index.ts                    # Hook barrel exports
│   ├── entityUser.tsx              # EntityUserProvider + useEntityUserId (user-scoped cache)
│   ├── useEntities.ts              # Entity CRUD hooks + entityKeys
│   ├── useEntityMembers.ts         # Member management hooks + memberKeys
│   ├── useInvitations.ts           # Invitation hooks + invitationKeys
│   ├── useEntityApiKeys.ts         # API key hooks + apiKeyKeys
│   ├── useCurrentEntity.tsx        # CurrentEntityProvider + useCurrentEntity(Optional)
│   └── useCurrentEntity.test.tsx   # Context / account-switch tests (vitest)
├── network/
│   ├── index.ts                    # Network barrel exports
│   ├── EntityClient.ts             # HTTP API client class
│   └── EntityClient.test.ts        # Unit tests (vitest)
└── types/
    ├── index.ts                    # Type barrel exports
    ├── errors.ts                   # EntityErrorCode enum + EntityApiError
    └── api-keys.ts                 # API key wire types (not yet in @sudobility/types)
```

## Public API

The full export surface (every hook, factory, and type) is documented in
`README.md`. Non-obvious notes only:

- **`EntityClient`** wraps an injected `NetworkClient` (from `@sudobility/types`),
  not `fetch` directly. Config: `{ baseUrl, networkClient }`.
- All client methods return **`BaseResponse<T>`** (from `@sudobility/types`):
  `{ success: boolean; data?: T; error?: string; timestamp: string }`. There is
  no local `ApiResponse` type.
- **API key wire types** (`EntityApiKey`, `CreatedEntityApiKey`,
  `CreateApiKeyRequest`, `UpdateApiKeyRequest`) live locally in
  `src/types/api-keys.ts`, not in `@sudobility/types` — move them there once a
  types release carries them. Request bodies use snake_case (`key_name`, `is_active`).
- **`EntityErrorCode` / `EntityApiError`** are local (`src/types/errors.ts`).

## Development Commands

```bash
bun run build          # Compile to dist/ (tsconfig.build.json) + scripts/fix-esm-imports.mjs
bun run build:watch    # Watch-mode compilation
bun run clean          # Remove dist/
bun run typecheck      # bunx tsc --noEmit
bun run lint           # ESLint on src/ (flat config: eslint.config.js)
bun run lint:fix       # ESLint with auto-fix
bun run format         # Prettier on src/**/*.{ts,tsx,js,jsx,json}
bun run test:unit      # vitest run (single pass) — NOTE: there is no `test` alias
bun run test:watch     # vitest (watch mode)
```

- `build` runs `tsc` then `scripts/fix-esm-imports.mjs`, which rewrites emitted
  relative imports to add `.js` extensions (required for Node ESM resolution).
- `prepublishOnly` auto-runs `clean` + `build`. CI publishes to GitHub Packages
  on push to `main`/`develop` via `.github/workflows/ci-cd.yml` (reusable workflow
  `johnqh/workflows/.github/workflows/unified-cicd.yml`, `publish-target: github`).

## Architecture / Patterns

### Dependency injection via client instance
Hooks take an `EntityClient` as their first argument (no global singleton), so
tests and multiple API environments are straightforward.

### Query key factory — every key starts with the user id (critical gotcha)
Each domain has a key factory, and **every key begins with the signed-in user id**:

```
entityKeys.detail(uid, slug)          // ['entities', uid, 'detail', slug]
memberKeys.list(uid, slug)            // [...entityKeys.detail, 'members', 'list']
apiKeyKeys.list(uid, slug)            // [...entityKeys.detail, 'api-keys', 'list']
invitationKeys.myList(uid)            // ['invitations', uid, 'my']
invitationKeys.entityList(uid, slug)  // [...entityKeys.detail, 'invitations']
```

Why: the cache was once keyed only by what was fetched. After a sign-out +
new-account sign-in in the same tab, the old account's workspace list survived,
the app sent the stale workspace id as `X-Entity-Id`, and the server refused
every request (including the corrective one). Clearing cache on user change
races the first render; keying by user makes the wrong answer unreachable.

Hooks read the user via `useEntityUserId()` from the nearest `EntityUserProvider`.
`CurrentEntityProvider` renders one internally, so apps using it need nothing
else. An app calling hooks **without** either provider shares a single `null`
user's cache entry. `useEntities` also accepts an explicit `userId` option
(used by `CurrentEntityProvider`, which sits above its own `EntityUserProvider`).
`useCurrentEntity.test.tsx` guards this: switching accounts never offers the
previous account's workspace.

### Automatic cache invalidation
Mutations invalidate related queries in `onSuccess`: entity create/delete → list;
update → detail + list; member/API-key mutations → that entity's list; accept
invitation → my-invitations + entity list.

### CurrentEntityProvider selection priority
1. Explicitly selected slug (`selectEntity()`)
2. Persisted slug from `localStorage`, per user (key `currentEntitySlug:<uid>`;
   missing/throwing storage, e.g. React Native or private windows, is skipped)
3. Personal entity (`entityType === EntityType.PERSONAL`)
4. First entity in the list

On user change it drops the previous user's cached entities/invitations and
restores the new user's remembered selection.

### Conditional query disabling
Hooks with a nullable `slug` disable the query when `slug` is null: `enabled: false`
plus a `['disabled']` placeholder query key, avoiding wasted API calls.

### Error handling
Client methods never throw; on failure they return
`{ success: false, error, timestamp }`. Hooks check `response.success` and throw
so TanStack Query can surface the error.

## Adding a new entity-scoped endpoint
1. Add the method to `EntityClient` (`src/network/EntityClient.ts`) + tests.
2. Add/extend a hook in `src/hooks/` with `useQuery`/`useMutation`.
3. Add keys to the relevant factory (first arg is always `userId`).
4. Wire cache invalidation in `onSuccess`.
5. Export from `src/hooks/index.ts` and `src/index.ts`.

## TypeScript config
Target ES2020, module ESNext with `bundler` resolution, `jsx: react-jsx`, strict
mode (`strict`, `noUnusedLocals`, `noUnusedParameters`). `tsconfig.build.json`
extends the base and excludes test files and `__tests__/`.

## Peer dependencies
`react` (^18 || ^19), `@tanstack/react-query` (^5), `@sudobility/types` — all
provided by the host app.
