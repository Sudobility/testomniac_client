# Testomniac Client

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

API client SDK for Testomniac with TanStack Query hooks.

**Package**: `@sudobility/testomniac_client` (BUSL-1.1). Published to **GitHub Packages** (`npm.pkg.github.com`), not public npm — `.npmrc` maps the `@sudobility` scope there and install needs a GitHub `read:packages` token.

## Tech Stack

- **Language**: TypeScript (strict mode)
- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm for installing dependencies)
- **Build**: TypeScript compiler (ESM)
- **Test**: Vitest
- **Data Fetching**: TanStack Query 5

## Project Structure

```
src/
├── index.ts                          # Root barrel: re-exports network/, hooks/, utils/, + types from types.ts
├── types.ts                          # Response interfaces (RunSummary, RunStructure, GraphView...),
│                                     #   FirebaseIdToken, ScriptKind, DEFAULT_GC_TIME. (NO query keys here.)
├── network/
│   ├── TestomniacClient.ts           # HTTP client class (~100 async methods, ~1480 lines) + createTestomniacClient
│   ├── TestomniacClient.test.ts      # Client tests
│   └── index.ts                      # exports TestomniacClient, createTestomniacClient
├── hooks/                            # ~100 React hooks (one file each: use*.ts)
│   ├── index.ts                      # Barrel export (all hooks + queryKeys + STALE_TIMES)
│   ├── query-keys.ts                 # queryKeys factory, createQueryKey, getServiceKeys, QueryKey type
│   ├── query-config.ts               # STALE_TIMES (per-resource stale times)
│   ├── __tests__/query-keys.test.ts
│   ├── useRun.ts                      # Query: single run details
│   ├── useSubmitScan.ts              # Mutation: submit scan (public endpoint)
│   └── ...
└── utils/
    ├── starter-helpers.ts            # createAuthHeaders, createHeaders, buildUrl, handleApiError
    ├── testomniac-helpers.ts         # buildArtifactUrl, buildRunStreamUrl
    ├── *.test.ts
    └── index.ts
```

Everything is re-exported from the package root — there is **no `/hooks` subpath export** (no `exports` map in package.json).

## Commands

```bash
bun run build          # Build ESM (tsc -p tsconfig.build.json → dist/)
bun run build:watch    # Watch mode build
bun run clean          # Remove dist/
bun run test:unit      # Run Vitest once  (NOT `bun test` — that runs Bun's own runner, not Vitest)
bun run test:watch     # Vitest watch mode
bun run typecheck      # tsc --noEmit
bun run lint           # eslint src/   (lint:fix to autofix)
bun run format         # prettier --write src/
bun run verify         # typecheck + lint + test:unit + build (run before commit)
bun run prepublishOnly # safety check + clean + build (runs on publish)
```

Dependencies install with **bun** only (`bun install`); do not use npm/yarn/pnpm.

## Key Concepts

### TestomniacClient

HTTP client class. Constructor is **positional**: `new TestomniacClient(networkClient, baseUrl)` (also `createTestomniacClient(networkClient, baseUrl)`). Uses dependency injection via the `NetworkClient` interface from `@sudobility/types` — no direct fetch calls. ~100 async methods covering all API endpoints. Conventions:
- Authenticated methods take the Firebase ID `token` as the **first** arg, e.g. `getTestRun(token, runId)`. Public methods (e.g. `submitDiscoveryRun(data)`) take no token.
- Every method returns a `BaseResponse<T>` envelope; unwrap with `.data`.
- `baseUrl` has any trailing slash stripped in the constructor.

### Hooks (~100)

Thin wrappers over TanStack Query. **Positional args, NOT a config object:**
- Query hooks: `(networkClient, baseUrl, token, ...params, options?)`. They return the raw `UseQueryResult<BaseResponse<T>>` (`{ data, isLoading, error, refetch }`) — **no field renaming, no `?? []`/`?? null` defaults**. Unwrap payload via `data?.data`. `enabled` and other options go in the trailing `options` arg (hooks typically set `enabled: !!token && !!id` internally).
- Mutation hooks: `(networkClient, baseUrl)`, returning the raw `UseMutationResult` (`{ mutateAsync, isPending, error, reset }`).
- `useSubmitScan` / `usePublicScanSubmission` are the public (no-auth) path, hitting `POST /api/v1/scan`.

### queryKeys (cache key factory)

Exported as `queryKeys` (from `src/hooks/query-keys.ts`) — **not** `QUERY_KEYS` (that older name was removed). Hierarchical, rooted at `['testomniac']`, e.g. `queryKeys.testomniac.run(runId)` → `['testomniac','run',runId]`. `queryKeys.testomniac.all()` invalidates everything. Also: `createQueryKey`, `getServiceKeys`, `QueryKey` type.

### Cache Settings

- `STALE_TIMES` (`src/hooks/query-config.ts`): per-resource stale times — 10 min structural (entities/products/runners/personas/scenarios), 5 min discovery (surfaces/interactions/pages), 2 min run/finding, 1 min dashboard. There is **no `DEFAULT_STALE_TIME`**.
- `DEFAULT_GC_TIME` (`src/types.ts`): 30 minutes.

## Peer Dependencies

- `react` (>=18)
- `@tanstack/react-query` (>=5)
- `@sudobility/types` — NetworkClient interface, BaseResponse

## Related Projects

- **testomniac_types** — Shared type definitions; this project imports all API request/response types
- **testomniac_api** — Backend server that this client SDK communicates with over HTTP
- **testomniac_lib** — Business logic library that consumes this client's hooks and `TestomniacClient` class
- **testomniac_app** — Web frontend that uses this client transitively via testomniac_lib

Dependency injection is central: `NetworkClient` interface is provided by the consumer, allowing different fetch implementations per platform (web vs React Native vs Chrome extension).

## Coding Patterns

- Use the `queryKeys` factory (`src/hooks/query-keys.ts`) for all query keys — never hand-write key tuples.
- Construct `TestomniacClient` positionally `(networkClient, baseUrl)`; never use `fetch` directly in this package — all HTTP goes through the injected `networkClient`.
- `FirebaseIdToken` is a branded `string` alias (`src/types.ts`) for auth tokens; auth methods/hooks take it first.
- New hooks follow the one-file-per-hook pattern in `src/hooks/` and must be added to `src/hooks/index.ts`; new public types added to `src/index.ts`.
- Request/response types come from `@sudobility/testomniac_types`; only SDK-local shapes live in `src/types.ts`.

## Gotchas

- Hooks take **positional** args and return **raw** TanStack results — do not assume a config object or renamed/defaulted return fields (older starter-template docs were wrong about this).
- `getTestRun(token, runId)` etc. — token comes **first** on authenticated client methods; easy to swap with the id.
- Run tests with `bun run test:unit`, not `bun test` (the latter invokes Bun's native runner, which is not what this repo uses).
- `useSubmitScan` / `usePublicScanSubmission` hit the public `/api/v1/scan` endpoint (no auth); all other hooks require a Firebase token.
- `queryKeys` must stay in sync with API route changes.
- Published package — breaking changes require a version bump (`package.json`) and coordination with consumers (`testomniac_lib`, apps).
- `plans/IMPROVEMENTS.md` is stale starter-template content (refers to `StarterClient`/`getHistories` that don't exist here) — ignore it.

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
