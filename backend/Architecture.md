# Architecture

Read this before adding or moving code. It tells you where things go.

## The core rule: core vs edge

The platform and infrastructure will change. Business logic must not know they exist.

- **Core** is business logic. Plain code, data, no platform or infrastructure imports. Core never fetches what it needs; it is handed it.
- **Edge** is any file that touches the platform, a library tied to it, or the outside world. Edges wire dependencies into core and translate everything foreign into our own types.

If a platform or infra change happens, only edge files change.

| Layer | Files | May import |
| --- | --- | --- |
| **Core** | `execute.ts`, `errors.ts`, `shared/*` | Plain TypeScript, `common/`, other core files in the same domain, `import type` from its own `schema.ts` |
| **Edge** | `worker.ts`, `bootstrap/`, `app/`, `config/`, `route.ts`, `routes.ts`, `queries.ts`, `infrastructure/`, `auth/` | Anything needed for the job (see Dependency rules) |
| **Contract** | `schema.ts`, `index.ts` | Zod (schema), core exports (index) |

Core must not import: `hono`, `drizzle-orm`, the Drizzle schema, anything Cloudflare, `infrastructure/`, `auth/`, `config/`, `bootstrap/`, any `queries.ts`, or another domain's files.

## Folder map

```
src/
  worker.ts            Workers entry point. Builds the platform, runs bootstrap, maps entry points to the app.
  bootstrap/           The one place where startup and entry wiring happens. Nothing imports it.
  app/                 Builds the Hono app from the routes it is given. Last-resort error backstop.
  config/              Validates settings it is given. Returns a Result.
  common/              AppError and Result types, plus ok, err, attempt. Imports nothing.
  domains/             One folder per business area.
    invoices/
      index.ts         Public API. The only file other domains may import.
      routes.ts        Composes this domain's use-case routes.
      shared/          Pure logic used by two or more use cases in this domain.
      use-cases/
        create-invoice/
          route.ts     EDGE. HTTP handler. Wires dependencies, calls execute.
          schema.ts    Zod schemas for request and response.
          execute.ts   CORE. The business logic.
          queries.ts   EDGE. Drizzle queries for this use case.
          errors.ts    CORE. Business errors for this use case.
  workflows/           Processes that connect several domains.
    routes.ts          Composes workflow routes.
    onboard-client/    Same files as a use case.
  infrastructure/
    platform/
      platform.ts      The Platform contract (database, storage, settings) and its registry. No Cloudflare.
      cloudflare/      The Cloudflare adapter. The only code that imports Cloudflare.
    database/
      db.ts            getDb(): the Drizzle client, taken from the platform.
      schema/          Drizzle tables.
      migrations/      The only place for migrations.
    http/              Hono helpers.
    integrations/      Object storage and other external services.
  auth/                Authentication and authorization.
tests/
  unit/<domain>/
  integration/<domain>/
  e2e/
```

## Naming

- Files and folders: kebab-case (`create-invoice`, `calculate-totals.ts`).
- Functions, variables, constants, Zod schemas: camelCase (`createInvoiceSchema`).
- Types: PascalCase (`type NewInvoice`, `type AppError`).
- Prefer `type` over `interface`.

## Bootstrap

Anything that must be set up at startup or at entry has one official path: a named function in its own module, called from `bootstrap/` and nowhere else. This keeps the import graph one-directional and prevents circular dependencies.

- Nothing runs on import. A module never initializes itself, registers itself, or reads state at import time. Importing is for definitions only.
- A module that needs startup work exposes a named function: `setPlatform`, `loadConfig`, `createAuth`, `createApp`.
- `bootstrap/` is the single composition root. It imports each module's startup function and each domain's `routes.ts`, and calls them in a fixed order. Nothing imports `bootstrap/` except `worker.ts` and test setup.
- Modules never call each other's startup functions. If module A needs B's output, `bootstrap/` passes it to A as an argument (`loadConfig({ settings })`, `createApp({ routes })`). So `application/` never imports domains, and `config/` never imports the platform.
- Startup order lives in one ordered list of steps. Each step returns a `Result`. The first failure stops bootstrap, and `worker.ts` responds with the `internal` error.
- Registries (the platform handle) are write-once. Only a bootstrap step sets them. Everything else reads them through a getter.

`worker.ts` builds the platform with the Cloudflare adapter, runs `bootstrap` once (cached at module level), and maps each Workers entry point (`fetch`, and later `scheduled` or `queue`) to the bootstrapped app. It contains no other wiring.

New startup need? Add a named function to the owning module and one step in `bootstrap/bootstrap.ts`. Never work around it with an import-time side effect.

## Platform access

The `Platform` contract defines the infrastructure capabilities the application needs, independent of any provider or implementation. It is not a database abstraction or an application interface.

- Cloudflare D1 is a platform-specific database implementation; Drizzle is a database access layer, not part of the platform contract.
- `infrastructure/platform/cloudflare/` adapts Cloudflare Workers infrastructure to the platform contract. It is the only place that knows about Cloudflare-specific APIs and types.
- `worker.ts` constructs the platform and passes it to `bootstrap`, where `setPlatform` registers it.
- Infrastructure consumers access platform capabilities through `getPlatform()`. Core and `route.ts` remain unaware of the platform.
- Moving to another provider means implementing the platform contract for that provider and updating `worker.ts`, without changing application logic.

## Errors: never throw

- Functions that can fail return a `Result`, defined in `common/`:

example:
```ts
type Result<T> = { ok: true; value: T } | { ok: false; error: AppError }

type AppError = {
  kind: 'validation' | 'unauthorized' | 'forbidden' | 'notFound' | 'conflict' | 'external' | 'internal'
  code: string                       // stable and unique, e.g. 'invoices.clientNotFound'
  message: string                    // safe to show to a client
  details?: Record<string, unknown>  // plain data, safe to show
  cause?: unknown                    // the original error; never sent to a client
}
```

- Handle errors where they happen: at the call that can fail, right away. Return `err(...)` or handle it. Do not catch far from the source.
- Anything that is not ours must be turned into an `AppError` at the edge that called it: D1 and Drizzle, `fetch`, external services, R2, JSON parsing. Wrap throwing calls with `attempt({ run, toError })`. Use Zod's `safeParse`. No raw `Error` or library error crosses an edge. Unknown failures become `internal` (our systems) or `external` (a third party).
- `queries.ts` maps failures it understands (such as a unique constraint) to a specific business error and everything else to `databaseError` from `infrastructure/database/`.
- Business errors live in the use case's `errors.ts` as plain functions returning an `AppError`.
- `respond` in `infrastructure/http/` is the only place that maps `kind` to an HTTP status.
- `app/` registers an `onError` handler as a last-resort backstop. It converts a stray throw into an `internal` `AppError` and responds. It is not control flow. If it fires, fix the code that threw.
- Registry getters are only reachable after bootstrap succeeded, because `worker.ts` handles no request before then.

## How dependencies reach core

`execute.ts` declares the plain types it needs. `route.ts` wires real implementations to them. Core never imports `queries.ts`, and `route.ts` never touches the database client.

```ts
// execute.ts (core)
type Deps = {
  findClient: (args: { id: string }) => Promise<Result<Client | null>>
  insertInvoice: (args: { invoice: NewInvoice }) => Promise<Result<Invoice>>
}
type ExecuteArgs = { input: CreateInvoiceInput; deps: Deps }
type ExecuteOutput = Result<Invoice>

export const execute = async ({ input, deps }: ExecuteArgs): Promise<ExecuteOutput> => {
  const found = await deps.findClient({ id: input.clientId })
  if (!found.ok) return found
  if (!found.value) return err(clientNotFound({ id: input.clientId }))
  const totals = calculateTotals({ items: input.items })
  return deps.insertInvoice({ invoice: { ...input, totals } })
}

// queries.ts (edge): gets the client from getDb() in infrastructure/database/db.ts
export const findClient = ({ id }: { id: string }) =>
  attempt({
    run: async () => {
      const rows = await getDb().select().from(clients).where(eq(clients.id, id)).limit(1)
      return rows[0] ? toClient({ row: rows[0] }) : null
    },
    toError: databaseError,
  })

// route.ts (edge)
export const createInvoiceRoute = async (c: Context) => {
  const input = await parseRequest({ c, schema: createInvoiceSchema })
  if (!input.ok) return respond({ c, result: input })
  const created = await execute({
    input: input.value,
    deps: {
      findClient: clients.findClientSummary, // from domains/clients/index.ts
      insertInvoice,                         // from this use case's queries.ts
    },
  })
  return respond({ c, result: created, status: 201 })
}
```

Rules that follow:

- Anything non-deterministic or external (database, external services, R2, current time, generated IDs) arrives as a dependency. Core does not call `fetch`, `Date.now()`, or `crypto.randomUUID()` when the result matters to logic.
- Data from another domain arrives as a dependency wired in `route.ts` from that domain's `index.ts`. Core may import another domain's pure exports directly, never anything that reads or writes data. Never import another domain's tables or queries.
- `queries.ts` has explicit parameter and return types made of plain data. Never return Drizzle rows, query builders, or D1 result objects.
- `queries.ts` owns atomic multi-statement writes (`db.batch`). Core never knows a batch happened.
- Only `queries.ts` and `infrastructure/database/` may import `drizzle-orm` or the Drizzle schema (the adapter additionally imports `drizzle-orm/d1`).

## Where does it go?

| You are adding | Put it in |
| --- | --- |
| A new business area | `src/domains/<name>/` with an `index.ts`, and add its routes to `bootstrap/bootstrap.ts` |
| A new business operation | `src/domains/<domain>/use-cases/<verb-noun>/` |
| The HTTP endpoint for it | `route.ts` in that use case, registered in the domain's `routes.ts` |
| Request or response validation | `schema.ts` in the use case |
| A database read or write | `queries.ts` in the use case |
| A business error | `errors.ts` in the use case, returning an `AppError` |
| A rule used by two or more use cases | `domains/<domain>/shared/` (if only one use case uses it, keep it in that folder) |
| A query used by two or more use cases | `domains/<domain>/shared/queries.ts` |
| Something other domains need | Export it from the domain's `index.ts` |
| A process touching several domains | `src/workflows/<verb-noun>/`, same files as a use case, registered in `workflows/routes.ts` |
| Anything that needs setup at startup or entry | A named function in its module, called from a step in `bootstrap/bootstrap.ts` |
| A table or column | `src/infrastructure/database/schema/`, then generate a migration into `src/infrastructure/database/migrations/` |
| A call to an external service or storage | `src/infrastructure/integrations/<provider>/` |
| Anything that needs a Cloudflare resource | Add it to the `Platform` type, implement it in `infrastructure/platform/cloudflare/` |
| Login or permission checks | `src/auth/` |
| An environment setting | `src/config/` |
| A test | See Testing |

## Definitions

- **Public API:** what a domain exports from `index.ts`. Nothing else is importable from outside.
- **Workflow:** a process that coordinates several domains through their public APIs. It has the same file structure as a use case and follows the same core and edge rules. Domain data comes through `index.ts`, so its `queries.ts` is usually not needed.
- **Platform:** the runtime we run on (Cloudflare today), seen only through the `Platform` contract.
- **Bootstrap:** the single composition root that runs startup steps in order and builds the app.

## Dependency rules

| From | May import | Must not import |
| --- | --- | --- |
| `worker.ts` | `infrastructure/platform/` (including the adapter), `bootstrap/`, `infrastructure/http/`, `common/` | Domains, workflows, `application/` |
| `bootstrap/` | Startup functions and routes of any module: `infrastructure/platform/platform.ts`, `config/`, `auth/`, `application/`, `domains/*/routes.ts`, `workflows/routes.ts`, `common/` | The Cloudflare adapter, private files of domains |
| `application/` | `infrastructure/http/`, `common/` | Domains, workflows, `bootstrap/` |
| `route.ts` | Its own `schema.ts`, `execute.ts`, `queries.ts`; other domains' `index.ts`; `infrastructure/http/`; `infrastructure/integrations/`; `auth/`; `common/` | `infrastructure/database/`, `infrastructure/platform/`, `bootstrap/`, other use cases |
| `execute.ts` | See the core vs edge table | Everything listed under "Core must not import" |
| A domain | Other domains' `index.ts`, `infrastructure/` (not the Cloudflare adapter), `auth/`, `common/` | Private files of other domains, `workflows/`, `bootstrap/` |
| `workflows/` | Domains' `index.ts`, `infrastructure/` (not the Cloudflare adapter), `auth/`, `common/` | Private files of any domain, `bootstrap/` |
| `infrastructure/platform/cloudflare/` | Cloudflare packages, `platform.ts`, `common/` | Domains, workflows, `bootstrap/`, the rest of `infrastructure/` |
| Rest of `infrastructure/` | Its own files, `platform.ts`, `common/` | Domains, workflows, `bootstrap/`, `config/`, anything Cloudflare |
| `config/` | `common/` | Everything else |
| `auth/` | `infrastructure/` (not the Cloudflare adapter), `common/` | Domains, workflows, `bootstrap/`, `config/` |
| `common/` | Nothing | Everything |

Only `worker.ts` and test setup may import `bootstrap/`. These rules are enforced by `pnpm check` (boundary lint). A failing check means the code is in the wrong place, not that the rule is wrong.

## Testing

- **Unit** (`tests/unit/<domain>/`): `execute.ts` and `shared/` with plain fake dependencies. No database, no HTTP. Assert on the returned `Result`, including error codes.
- **Integration** (`tests/integration/<domain>/`): `queries.ts` and `route.ts` against a real local D1 (`@cloudflare/vitest-pool-workers`). The test setup builds the platform through the Cloudflare adapter and runs `bootstrap`, like `worker.ts` does.
- **E2E** (`tests/e2e/`): a few critical flows through the whole app.

## Rules that shape the code

- Routes are written by hand. No generated routes.
- Migrations live only in `src/infrastructure/database/migrations/`. Never edit an applied migration. For risky changes: add the new shape, switch over, remove the old one in a later migration.
- Workers are stateless. Keep nothing request-specific in module-level variables. Module-level variables may hold caches only. The registries set by bootstrap are the exception: they hold bindings, not request state.
- Webhook handlers must be safe to process twice.
- Prefer one database round trip per request. Avoid per-row loops.

## Adding a feature

1. Find the most similar existing use case and read it. Copy its shape.
2. Pick the domain. If none fits, ask before creating a new one.
3. Create only the files the use case needs. A simple read may be just `route.ts` and `queries.ts`.
4. Write core first (`execute.ts` with its `Deps` type), then the edge that satisfies it.
5. If another domain needs it, export from `index.ts`. If it touches several domains, make it a workflow.
6. Add tests, then run `pnpm check`.

## Do not create

- Empty folders or files "for later".
- Global `controllers/`, `services/`, `repositories/`, or `validators/` folders.
- A repository layer over Drizzle. Queries stay next to the use case that uses them.
- Import-time side effects, or any startup path outside `bootstrap/`.
- Observability (logging, error reporting). It is out of scope for now.