# Spec: go_inventory_managent

## Objective

Build a thorough, easy-to-use inventory and warehouse management web application for small-to-large operations. The app covers the full domain: product catalog, stock and locations, procurement, outbound (pick/pack/ship), manufacturing (BOM/build), serialized asset tracking, operations (cycle counts, transfers, recalls, cold chain), and reporting.

The primary user is a warehouse or inventory operator who needs fast, browser-based access from any device. Ease of use is the priority regardless of operation scale. The app must be usable in any language from day one, with Arabic (RTL) treated as a first-class peer to English (LTR).

Success looks like: a single binary runs locally, opens in a browser, lets a user log in, manage parts and stock across locations, raise and receive purchase orders, pick and ship, build assemblies, track serialized assets, and read reports — all in their chosen language with the layout correctly mirrored.

## Tech Stack

- **Language:** Go 1.22+ (standard library only for application code)
- **ORM:** Ent (`entgo.io/ent`) — the sole database layer; no raw SQL, no direct driver imports
- **HTTP:** stdlib `net/http` (Go 1.22 pattern routing); no web framework
- **UI interactivity:** HTMX, integrated via `github.com/donseba/go-htmx`
- **Styling:** Tailwind CSS v4 + daisyUI v5 (compiled via standalone Tailwind CLI invoked from mage)
- **i18n:** `github.com/nicksnyder/go-i18n/v2` with TOML message files
- **Build system:** magefile (`github.com/magefile/mage`)
- **Templating:** stdlib `html/template`
- **DB (dev):** SQLite via Ent's SQLite dialect
- **DB (prod, final phase):** Postgres via Ent's Postgres dialect — dialect swap is the last phase
- **QR codes:** stdlib-only SVG generator (no external QR library)
- **Auth:** stdlib signed-cookie sessions + stdlib PBKDF2 password hashing

No external libraries beyond the ones listed above and their transitive dependencies.

## Commands

All build/run/test/migrate work flows through mage so the surface is identical for humans and agents.

```
Build:          mage build
Run:            mage run
Test:           mage test
Clean:          mage clean
Tidy:           mage tidy
Ent codegen:    mage ent
CSS build:      mage css
i18n extract:   mage i18n
Migrate (P10):  mage migrate
Phase gate:     mage clean build && mage test
```

Per-phase exit gate: `mage clean build` must exit 0 before the phase's final commit. `mage test` must be green.

## Project Structure

```
go_inventory_managent/
├── AGENTS.md                 # project rules (committed, read-only source of truth)
├── SPEC.md                   # this file
├── tasks/
│   ├── plan.md               # phase decomposition + dependency order
│   └── todo.md               # discrete, checkbox-tracked tasks
├── magefile.go               # all build targets
├── cmd/inventory/main.go     # entrypoint
├── ent/
│   ├── generate.go           # go:generate
│   └── schema/               # one .go per entity, added per phase
├── internal/
│   ├── server/               # router, middleware (locale, auth, htmx), handlers, embedded templates + static
│   ├── service/              # business logic per domain (ent-backed)
│   ├── auth/                 # session store, password hashing, rbac constants
│   ├── i18n/                 # bundle loader, locale registry, embedded active.*.toml
│   ├── db/                   # ent client open + seed helpers
│   ├── config/               # env-driven config
│   └── qrcode/               # stdlib QR SVG generator (P8)
├── web/
│   ├── css/input.css         # tailwind + daisyui plugin source
│   └── vendor/htmx.min.js    # vendored
└── docs/                     # reference notes only
```

## Code Style

- Follow idiomatic Go: package layout, error returns, short variable names at narrow scope, interfaces defined at point of use.
- One schema file per Ent entity. Schemas describe fields and edges; business logic lives in `internal/service`.
- Handlers in `internal/server` are thin: parse request, call service, render template. No business logic in handlers.
- Templates use logical CSS properties only (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`, `text-start`, `text-end`, `float-start`, `float-end`, `inset-inline-*`). Never `ml-*`, `mr-*`, `left`, `right` for layout-bearing elements. Directional icons flipped via `[dir=rtl]` selectors. daisyUI v5 components are dir-aware and used as-is.
- All user-facing strings go through the `T` template function backed by i18n message IDs. No hardcoded English in templates.
- Internal links built via template helpers that inject the current locale prefix (`/en/parts`, `/ar/parts`). No hardcoded locale in markup.
- Commits follow Mitchell Hashimoto style: imperative summary line ≤72 chars, blank line, body explaining *why*. One logical change per commit. No `--amend` unless requested.

## Testing Strategy

- **Framework:** Go stdlib `testing` + table-driven tests.
- **Locations:** `_test.go` files live next to the code under test (Go convention).
- **Levels:**
  - Unit: pure functions (i18n dir map, password hashing, FEFO sort, QR encoder) — fast, no DB.
  - Service: business logic against an in-memory Ent SQLite client (`ent.Open(sqlite, "file:...?mode=memory")`) — covers tx boundaries, movement ledger, allocation.
  - Handler: `httptest.NewRequest` + `httptest.NewRecorder` against the locale/auth middleware and route handlers.
- **Coverage:** Every service function and every middleware has at least one test. Handlers tested for status code + redirect target, not full HTML.
- **Per task:** each task's `Verify:` field names a concrete command the agent runs (e.g. `mage test`, `curl -sI /en/dashboard`). Evidence (exit code, output snippet) goes in the commit body.
- **No stubs:** a task is not done until its verification passes against a real implementation. Placeholder implementations are forbidden.

## Boundaries

- **Always do:**
  - Run `mage clean build && mage test` before the phase-final commit.
  - Persist user locale preference in the `users.locale` column; never in a client-side cookie alone.
  - Set `dir` and `lang` on the `<html>` element from the request locale for every page render.
  - Use logical CSS properties in every template (enforces mirroring).
  - One commit per task; reference the task in the commit body.
  - Search the codebase before assuming something isn't implemented.
- **Ask first:**
  - Any DB schema change after a phase has shipped (update spec first).
  - Adding any external dependency not in the tech stack list.
  - Changing the module path or repo URL.
  - Swapping the Ent dialect (P10 owns this).
- **Never do:**
  - Raw SQL or direct driver imports — Ent is the only DB layer.
  - Hardcode `dir="ltr"` or `dir="rtl"` in templates — always server-driven.
  - Use physical-direction CSS properties (`ml-*`, `mr-*`, `left`, `right`) for layout-bearing elements.
  - Hardcode user-facing English in templates — always `{{T "key"}}`.
  - Commit secrets, `.env`, `*.sqlite`, `node_modules/`, or build artifacts.
  - Edit vendor directories.
  - Remove failing tests without approval.
  - Implement placeholder/stub implementations — full implementations only.

## Success Criteria

- `mage clean build && mage test` exits 0 after every phase.
- `/en/dashboard` renders LTR; `/ar/dashboard` renders the exact visual mirror (verified manually each phase, headless-automated when feasible).
- Root `/` redirects to the authenticated user's persisted locale, or to the default locale (`en`) when unauthenticated.
- Every protected route redirects to `/{lang}/login` when unauthenticated.
- Every domain phase (catalog, stock, procurement, outbound, manufacturing, assets, operations, reports) has end-to-end CRUD or workflow working in the browser with English + Arabic strings.
- A seeded admin user (`admin@example.com` / `admin`) can log in and exercise every feature.
- All user-facing strings exist in both `active.en.toml` and `active.ar.toml`.
- Final phase: app runs against Postgres with versioned migrations; `mage migrate` succeeds; all tests green.
- Released as `v0.1.0` git tag.

## Open Questions

1. **Session secret source:** default to a random 32-byte value generated at startup (logged once) when `INVENTORY_SESSION_SECRET` env is unset. Acceptable, or require explicit env on first run?
2. **File uploads (attachments):** store on local filesystem under `uploads/` (gitignored) keyed by ent attachment row. Acceptable for v0.1.0, or defer attachments entirely until an object store is chosen?
3. **Barcode scanning UI:** P8 ships QR *generation*. Scanning (camera input) is browser-side via the device's native `getUserMedia` + a stdlib-free JS decoder, or out of scope for v0.1.0? (Recommend: out of scope; users scan with any app and hit the `/scan/{code}` landing route.)
4. **Multi-tenant:** single-tenant for v0.1.0 (one installation = one warehouse org). Multi-tenant (org column + row-level filtering) deferred. Confirm.
5. **Currency:** single-currency per installation for v0.1.0 (config-driven), multi-currency + exchange rates deferred. Confirm.
6. **Reports rendering:** server-rendered HTML tables only (no chart library). Charts deferred. Confirm.