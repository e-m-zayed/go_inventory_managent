# Spec: go_inventory_managent

## Objective

Build a thorough, easy-to-use, **multi-tenant** inventory and warehouse management web application for small-to-large operations. One installation hosts multiple organizations; every domain entity is tenant-scoped and isolated at the data layer. The app covers the full domain: product catalog, stock and locations, procurement, outbound (pick/pack/ship), manufacturing (BOM/build), serialized asset tracking, operations (cycle counts, transfers, recalls, cold chain), reporting, and browser-side barcode scanning.

The primary user is a warehouse or inventory operator who needs fast, browser-based access from any device. Ease of use is the priority regardless of operation scale. The app must be usable in any language from day one, with Arabic (RTL) treated as a first-class peer to English (LTR).

Success looks like: a single binary runs locally, opens in a browser, lets a user log in, manage parts and stock across locations within their tenant, raise and receive purchase orders, pick and ship, build assemblies, track serialized assets, scan barcodes in the browser, and read reports — all in their chosen language with the layout correctly mirrored. Users in tenant A can never see or touch tenant B's data.

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
- **QR/barcode generation + scanning:** `github.com/makiuchi-d/gozxing` (pure-Go ZXing port — handles both encoding and decoding across QR, Code128, Code39/93, EAN-8/13, Codabar, ITF, DataMatrix, Aztec). No vendored JS, no stdlib SVG hand-roll.
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
│   ├── generate.go           # //go:generate with --feature intercept,schema/snapshot
│   └── schema/               # one .go per entity; TenantMixin embedded in every domain entity
├── internal/
│   ├── server/               # router, middleware (locale, auth, tenant, htmx), handlers, embedded templates + static
│   ├── service/              # business logic per domain (ent-backed, tenant-aware via context)
│   ├── auth/                 # session store, password hashing, rbac constants
│   ├── tenant/               # TenantMixin definition, tenant context helpers, fail-closed interceptor registration
│   ├── i18n/                 # bundle loader, locale registry, embedded active.*.toml
│   ├── db/                   # ent client open + seed helpers (seeds root tenant + admin)
│   ├── config/               # env-driven config (INVENTORY_SESSION_SECRET required, INVENTORY_DB, INVENTORY_ADDR, INVENTORY_DEFAULT_LOCALE, INVENTORY_CURRENCY)
│   └── barcode/              # gozxing-based generation + decoding helpers (P8)
├── web/
│   ├── css/input.css         # tailwind + daisyui plugin source
│   └── vendor/
│       └── htmx.min.js       # vendored
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
  - Embed `TenantMixin` in every domain ent schema (everything except `Tenant`, `User`-tenant-edge, `Permission` codes). Never write a domain entity without it.
  - Register the fail-closed tenant interceptor on the ent client at startup; every query/mutation passes through it.
  - Carry `tenantID` in the session cookie and inject the tenant into request context via middleware before any handler runs.
- **Ask first:**
  - Any DB schema change after a phase has shipped (update spec first).
  - Adding any external dependency not in the tech stack list.
  - Changing the module path or repo URL.
  - Swapping the Ent dialect (P10 owns this).
  - Writing code that intentionally bypasses the tenant interceptor (admin tooling only; document why in the commit).
- **Never do:**
  - Raw SQL or direct driver imports — Ent is the only DB layer.
  - Hardcode `dir="ltr"` or `dir="rtl"` in templates — always server-driven.
  - Use physical-direction CSS properties (`ml-*`, `mr-*`, `left`, `right`) for layout-bearing elements.
  - Hardcode user-facing English in templates — always `{{T "key"}}`.
  - Commit secrets, `.env`, `*.sqlite`, `node_modules/`, or build artifacts.
  - Edit vendor directories.
  - Remove failing tests without approval.
  - Implement placeholder/stub implementations — full implementations only.
  - Start the app without `INVENTORY_SESSION_SECRET` set. Fail fast, log a clear message, exit non-zero.
  - Add a domain ent schema without the `TenantMixin`.

## Success Criteria

- `mage clean build && mage test` exits 0 after every phase.
- `/en/dashboard` renders LTR; `/ar/dashboard` renders the exact visual mirror (verified manually each phase, headless-automated when feasible).
- Root `/` redirects to the authenticated user's persisted locale, or to the default locale (`en`) when unauthenticated.
- Every protected route redirects to `/{lang}/login` when unauthenticated.
- The app refuses to start when `INVENTORY_SESSION_SECRET` is unset.
- Every domain entity is tenant-scoped: a query in tenant A's request context returns zero rows from tenant B. Verified by a service test that opens two tenants and asserts cross-tenant read/mutation attempts return no rows / are rejected.
- Every domain phase (catalog, stock, procurement, outbound, manufacturing, assets, operations, reports) has end-to-end CRUD or workflow working in the browser with English + Arabic strings.
- Barcode + QR generation and scanning works: opening the scan page grants camera access, captures a frame, uploads it via htmx, the server decodes it with gozxing, and the browser follows an HX-Redirect to the matching entity's detail page. Label views render QR PNGs generated server-side via gozxing.
- A seeded admin user (`admin@example.com` / `admin`) belonging to the root tenant can log in and exercise every feature.
- All user-facing strings exist in both `active.en.toml` and `active.ar.toml`.
- Final phase: app runs against Postgres with versioned migrations; `mage migrate` succeeds; all tests green.
- Released as `v0.1.0` git tag.

## Decisions (from review)

1. **Session secret:** explicit env var `INVENTORY_SESSION_SECRET` is required. App refuses to start when unset. No runtime-generated fallback.
2. **File uploads (attachments):** local filesystem under `uploads/` (gitignored), path stored on the ent Attachment row. Chosen for v0.1.0; object-store abstraction deferred.
3. **Multi-tenant:** required for v0.1.0. One installation hosts multiple organizations; every domain entity is tenant-scoped.
   - Mechanism: a `Tenant` ent schema + a `TenantMixin` (ent.Mixin) that adds a `tenant` edge to every domain entity. Ent's `intercept` feature flag is enabled in `ent/generate.go`; a fail-closed `intercept.TraverseFunc` injects a `tenant_id` predicate into every read/write so no query can cross tenant boundaries unless the context explicitly opts out (admin tooling only).
   - `User` has a `tenant` edge; the session cookie carries `tenantID`; an HTTP middleware injects the tenant into request context; the interceptor reads it.
   - Default tenant: a single root org is seeded at first run; the seeded admin belongs to it. Creating additional tenants + admin users is an admin UI task (Phase 11 — Tenant Admin, see tasks/plan.md).
   - No cross-tenant queries in normal request flows. Reports and exports inherit the tenant filter automatically.
4. **Currency:** single currency per installation for v0.1.0, driven by an env/config value. Multi-currency + FX rates (Frankfurter v2) + `go-money` are out of scope for v0.1.0 and will land in a later minor release.
5. **Reports rendering:** server-rendered HTML tables only. No chart library. Charts deferred.
6. **Barcode + QR generation and scanning:** in scope for v0.1.0 (Phase 08). Handled by a single Go package, `github.com/makiuchi-d/gozxing`, on the server.
   - **Generation:** the server renders QR (and other 1D/2D formats as needed) to a `*image.RGBA` via gozxing's writers, encodes to PNG with stdlib `image/png`, and streams it back for product/stock/location/asset/bin label views.
   - **Scanning:** the browser captures a single frame from `getUserMedia`, uploads it via htmx (`hx-post` with the image bytes), the server decodes it with gozxing's `qrcode.NewQRCodeReader().Decode(...)` (or `MultiFormatReader` for mixed formats), resolves the resulting code string against the entity lookup, and returns an HX-Redirect to the entity's detail page. No browser-side decoder, no feature detection, no vendored JS. Works in every browser that supports `getUserMedia` + file upload.

## Open Questions

None remaining. All six original questions are resolved above.
