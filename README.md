# @sudobility/testomniac_client

API client SDK for the Testomniac testing platform with TanStack Query hooks.

## Installation

This package is published to **GitHub Packages** (not the public npm registry).
The repo's `.npmrc` already points the `@sudobility` scope at
`https://npm.pkg.github.com`; consumers need the same scope mapping plus a
GitHub token with `read:packages` to install it.

```bash
bun add @sudobility/testomniac_client
```

Peer dependencies (provided by the consumer, not bundled):

```bash
bun add react @tanstack/react-query @sudobility/types
```

## Usage

All public exports come from the package root — there is **no `/hooks`
subpath** export.

```ts
import {
  TestomniacClient,
  createTestomniacClient,
  useRun,
  useSubmitScan,
  usePublicScanSubmission,
  queryKeys,
} from '@sudobility/testomniac_client';

// Create the HTTP client. Constructor is POSITIONAL: (networkClient, baseUrl).
// `networkClient` implements the NetworkClient interface from @sudobility/types
// (dependency injection — the SDK never calls fetch itself).
const client = new TestomniacClient(networkClient, baseUrl);
// or: const client = createTestomniacClient(networkClient, baseUrl);

// Call a method directly. Methods take the auth token FIRST, then params.
const run = await client.getTestRun(token, runId); // -> BaseResponse<TestRunResponse>

// In a React component -- query hook. Args are POSITIONAL, not a config object.
// Hooks return the raw TanStack Query result (data/isLoading/error/refetch).
const { data, isLoading, error } = useRun(networkClient, baseUrl, token, runId);
const testRun = data?.data; // unwrap BaseResponse

// Mutation hook. useSubmitScan hits the public /api/v1/scan endpoint (no auth).
const mutation = useSubmitScan(networkClient, baseUrl);
await mutation.mutateAsync({ url: 'https://example.com' });

// usePublicScanSubmission wraps useSubmitScan with friendly error handling:
const { submitPublicScan, isSubmitting, error: scanError } =
  usePublicScanSubmission(networkClient, baseUrl);
const testRunId = await submitPublicScan({ url: 'https://example.com', email });
```

## API

### TestomniacClient

HTTP client class constructed **positionally** as `new TestomniacClient(networkClient, baseUrl)` (or via the `createTestomniacClient(networkClient, baseUrl)` factory). Uses dependency injection via the `NetworkClient` interface from `@sudobility/types` -- no direct fetch calls. Provides ~100 async methods covering all API endpoints: users, entities, products, runners, test environments, test runs, findings, personas, test scenarios, test surfaces, test interactions, credentials, schedules, scan settings, navigation graph/planning, and more. Authenticated methods take the Firebase ID `token` as their **first** argument, e.g. `getTestRun(token, runId)`; public methods (e.g. `submitDiscoveryRun(data)`) take no token. Every method returns a `BaseResponse<T>` envelope.

### Hooks (~100)

Hooks are thin wrappers over TanStack Query. They take **positional arguments** — typically `(networkClient, baseUrl, token, ...params, options?)` for queries and `(networkClient, baseUrl)` for mutations — not a config object.

**Query hooks** return the raw `UseQueryResult<BaseResponse<T>>` (`{ data, isLoading, error, refetch, ... }`). Unwrap the API payload with `data?.data`. Pass TanStack options (including `enabled`) via the trailing `options` argument.

**Mutation hooks** return the raw `UseMutationResult` (`{ mutateAsync, mutate, isPending, error, reset, ... }`).

The hook names below are a catalog; consult the function signature in `src/hooks/<name>.ts` for the exact argument order.

#### Runs & Summaries

- `useRun(config)` -- Single run details
- `useRunSummary(config)` -- Run summary with finding counts and expertise breakdown
- `useRunLiveDashboard(config)` -- Live dashboard data for in-progress runs
- `useRunStructure(config)` -- Run structure (pages, interactions hierarchy)
- `useRunNavigationMap(config)` -- Site navigation map for graph visualization

#### Pages & Page States

- `useRunPages(config)` -- Pages discovered in a run
- `useRunPagesSummary(config)` -- Summary of all pages in a run
- `useRunPageSummary(config)` -- Detailed summary for a single page
- `usePageStates(config)` -- Page states for a given page
- `usePageStateItems(config)` -- Actionable items within a page state
- `usePageStateScaffolds(config)` -- Scaffolds detected in a page state
- `useHtmlElement(config)` -- HTML element details

#### Test Surfaces & Bundles

- `useRunnerTestSurfaces(config)` -- Test surfaces for a runner
- `useEnvironmentTestSurfaces(config)` -- Test surfaces for an environment
- `useTestSurfaceChildSurfaces(config)` -- Child surfaces of a test surface
- `useTestSurfaceTestInteractions(config)` -- Interactions for a test surface
- `useRunnerTestSurfaceBundles(config)` -- Test surface bundles
- `useBundleSurfaces(config)` -- Surfaces in a bundle
- `useBundleInteractions(config)` -- Interactions in a bundle
- `useBundleScenarios(config)` -- Scenarios in a bundle
- `useCreateTestSurfaceBundle(config)` -- Mutation: create bundle
- `useUpdateTestSurfaceBundle(config)` -- Mutation: update bundle
- `useDeleteTestSurfaceBundle(config)` -- Mutation: delete bundle

#### Test Interactions & Test Runs

- `useRunTestInteractions(config)` -- Test interactions for a run
- `useRunnerTestInteractions(config)` -- Test interactions for a runner
- `useEnvironmentTestInteractions(config)` -- Test interactions for an environment
- `useTestInteractionActions(config)` -- Actions within a test interaction
- `useTestInteractionRun(config)` -- Single test interaction run
- `useCreateTestInteractionRun(config)` -- Mutation: create interaction run
- `useRunTestRuns(config)` -- Test runs for a run
- `useRunnerTestRuns(config)` -- Test runs for a runner
- `useTestRunFindings(config)` -- Findings for a specific test run

#### Findings & Patterns

- `useRunFindings(config)` -- Findings for a run
- `useRunnerFindings(config)` -- Findings across a runner
- `useRunPatterns(config)` -- UI patterns detected in a run
- `useRunScaffolds(config)` -- Scaffolds detected in a run
- `useRunnerScaffolds(config)` -- Scaffolds across a runner

#### Personas

- `useRunPersonas(config)` -- Personas for a run
- `useRunnerPersonas(config)` -- Personas for a runner
- `useProductPersonas(config)` -- Personas for a product
- `usePersonaUseCases(config)` -- Use cases for a persona
- `useUseCaseInputValues(config)` -- Input values for a use case
- `useCreatePersona(config)` -- Mutation: create persona
- `useUpdatePersona(config)` -- Mutation: update persona
- `useDeletePersona(config)` -- Mutation: delete persona
- `useDetectPersonas(config)` -- Mutation: AI-detect personas

#### Test Scenarios & Sequences

- `useRunnerTestScenarios(config)` -- Test scenarios for a runner
- `useTestScenarioSequences(config)` -- Sequences in a test scenario
- `useTestScenarioSequenceTestInteractions(config)` -- Interactions in a sequence
- `useTestScenarioSequenceRuns(config)` -- Runs of a sequence
- `useCreateTestScenario(config)` -- Mutation: create scenario
- `useUpdateTestScenario(config)` -- Mutation: update scenario
- `useDeleteTestScenario(config)` -- Mutation: delete scenario
- `useDetectTestScenarios(config)` -- Mutation: AI-detect scenarios
- `useGenerateSequence(config)` -- Mutation: AI-generate sequence steps

#### Schedules & Credentials

- `useRunnerSchedules(config)` -- Schedules for a runner
- `useCreateTestSchedule(config)` -- Mutation: create schedule
- `useEntityCredentials(config)` -- Credentials for an entity
- `useCreateEntityCredential(config)` -- Mutation: create credential
- `useUpdateEntityCredential(config)` -- Mutation: update credential
- `useDeleteEntityCredential(config)` -- Mutation: delete credential

#### Entities, Products & Runners

- `useEntities(config)` -- User's entities
- `useEntityProducts(config)` -- Products for an entity
- `useProduct(config)` -- Single product details
- `useProductRuns(config)` -- Runs for a product
- `useProductRunners(config)` -- Runners for a product
- `useProductEnvironments(config)` -- Environments for a product
- `useRunner(config)` -- Single runner details
- `useRunnerScans(config)` -- Scans for a runner
- `useRunnerPages(config)` -- Pages across a runner
- `useRunnerPageStates(config)` -- Page states across a runner
- `useEnvironmentPages(config)` -- Pages for an environment

#### Scanning

- `useSubmitScan(networkClient, baseUrl)` -- Mutation: submit a scan to the public `/api/v1/scan` endpoint (no auth required)
- `usePublicScanSubmission(networkClient, baseUrl)` -- Higher-level wrapper returning `{ submitPublicScan, isSubmitting, error, resetError }`

### queryKeys

Type-safe cache key factory for TanStack Query (exported as `queryKeys`, not `QUERY_KEYS`). Keys are namespaced under `'testomniac'` and structured hierarchically, e.g. `queryKeys.testomniac.run(runId)` produces `['testomniac', 'run', runId]`. Use `queryKeys.testomniac.all()` for bulk invalidation. Also exported: `createQueryKey`, `getServiceKeys`, and the `QueryKey` type (all from `src/hooks/query-keys.ts`).

### Cache Settings

- `STALE_TIMES` (from `src/hooks/query-config.ts`) -- per-resource stale times used by the hooks: structural/reference data 10 min (`ENTITY`, `PRODUCT`, `RUNNER`, `PERSONA`, `SCENARIO`, ...), discovery artifacts 5 min (`SURFACE`, `INTERACTION`, `PAGE`, ...), run/finding data 2 min (`RUN`, `FINDING`), live dashboard 1 min (`DASHBOARD`).
- `DEFAULT_GC_TIME` (from `src/types.ts`) -- 30 minutes.

There is no `DEFAULT_STALE_TIME` constant.

### Utilities

- `createAuthHeaders(token)` -- Builds `Authorization: Bearer` header object
- `createHeaders(token?)` -- Builds request headers (optionally authorized)
- `buildUrl(base, path)` -- URL construction helper
- `handleApiError(error)` -- Standardized API error handling
- `buildArtifactUrl(...)` / `buildRunStreamUrl(...)` -- Testomniac-specific URL builders (`src/utils/testomniac-helpers.ts`)

## Development

```bash
bun run build          # Build ESM (tsc -p tsconfig.build.json -> dist/)
bun run build:watch    # Watch mode build
bun run clean          # Remove dist/
bun run test:unit      # Run Vitest tests once (NOT `bun test`, which uses Bun's own runner)
bun run test:watch     # Vitest watch mode
bun run typecheck      # TypeScript check (tsc --noEmit)
bun run lint           # ESLint over src/
bun run format         # Prettier write over src/
bun run verify         # typecheck + lint + test:unit + build (use before commit)
```

## Related Packages

- **testomniac_types** -- Shared type definitions (imported for all API request/response types)
- **testomniac_api** -- Backend server this client communicates with
- **testomniac_lib** -- Business logic library that wraps this client's hooks
- **testomniac_app** -- Web frontend (consumes transitively via testomniac_lib)
- **testomniac_app_rn** -- React Native app (consumes transitively via testomniac_lib)

## License

BUSL-1.1
