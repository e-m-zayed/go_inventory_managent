# Tasks

> Format per spec-driven-development skill: `- [ ] Task` + `Acceptance:` + `Verify:` + `Files:`.
> Update status in place as tasks complete. Keep closed tasks in the file for traceability.
> Per-task rule: search the codebase before assuming something isn't implemented; no stubs.

---

## Phase 00 — Scaffold

- [ ] Task 00.1: Initialize Go module + gitignore
  - Acceptance: `go.mod` declares `module github.com/e-m-zayed/go_inventory_managent`; `.gitignore` excludes macOS/Go/node/db artifacts; AGENTS.md present.
  - Verify: `go mod verify` exit 0; `git status` clean after first commit.
  - Files: `go.mod`, `.gitignore`, `AGENTS.md`

- [ ] Task 00.2: Add magefile with core targets
  - Acceptance: `mage build`, `mage run`, `mage clean`, `mage tidy` exist and run without error (build may no-op until main.go exists — accept "nothing to build" as success for this task).
  - Verify: `mage -l` lists the four targets; `mage clean` exits 0.
  - Files: `magefile.go`, `go.mod`, `go.sum`

- [ ] Task 00.3: Scaffold Ent + User schema with locale field
  - Acceptance: `ent/schema/user.go` defines User with name, email (unique), password_hash (sensitive), locale (default "en"), created_at, updated_at; `mage ent` generates client code; `ent/generate.go` carries the `//go:generate` directive.
  - Verify: `mage ent` exit 0; `go build ./ent/...` exit 0.
  - Files: `ent/generate.go`, `ent/schema/user.go`, `ent/**` (generated), `magefile.go`, `go.mod`, `go.sum`

- [ ] Task 00.4: DB layer + entrypoint
  - Acceptance: `internal/config.Load()` reads env with sqlite default; `internal/db.Open` opens ent sqlite client and runs auto-migrate; `cmd/inventory/main.go` opens DB, waits for signal, closes cleanly.
  - Verify: `mage clean build` exit 0; run binary, see "db opened" log, send SIGINT, see "shutting down".
  - Files: `internal/config/config.go`, `internal/db/db.go`, `cmd/inventory/main.go`, `go.mod`, `go.sum`

- [ ] Task 00.5: i18n bundle + locale registry
  - Acceptance: `internal/i18n` exposes Supported locales (`en`, `ar`), `DirFor(lang)` (ar→rtl else ltr), `NewBundle()` loading embedded `active.en.toml` + `active.ar.toml`, and `Localizer(bundle, lang)`; both TOML files contain `AppTitle`, `NavDashboard`, `Welcome`.
  - Verify: `go test ./internal/i18n/...` passes a DirFor table test; `mage clean build` exit 0.
  - Files: `internal/i18n/locales.go`, `internal/i18n/bundle.go`, `internal/i18n/locales/active.en.toml`, `internal/i18n/locales/active.ar.toml`, `magefile.go`, `go.mod`, `go.sum`

- [ ] Task 00.6: Tailwind + daisyUI CSS pipeline + vendor htmx
  - Acceptance: `web/css/input.css` imports tailwind and adds the daisyUI plugin; `mage css` compiles `web/css/app.css` (gitignored); `web/vendor/htmx.min.js` vendored; `package.json` declares daisyui only.
  - Verify: `mage css` exit 0; `web/css/app.css` exists and is non-empty; `test -s web/vendor/htmx.min.js`.
  - Files: `web/css/input.css`, `web/vendor/htmx.min.js`, `package.json`, `.gitignore`, `magefile.go`

- [ ] Task 00.7: HTTP server + locale middleware + mirrored dashboard
  - Acceptance: `internal/server` wires router with `/{lang}/` prefix; LocaleMiddleware validates lang, sets dir+localizer in context; root `/` redirects to default locale; dashboard template sets `<html dir lang>`; static CSS + htmx served from embedded FS.
  - Verify: `mage clean build`; run; `curl -sI /` → 302 to `/en/`; `curl -s /en/dashboard` contains `dir="ltr"`; `curl -s /ar/dashboard` contains `dir="rtl"` and Arabic strings.
  - Files: `internal/server/{server,router,locale,htmxmw,render,funcs}.go`, `internal/server/templates/layout.html`, `internal/server/templates/pages/dashboard.html`, `internal/server/static/{css/app.css,vendor/htmx.min.js}`, `cmd/inventory/main.go`

- [ ] Task 00.8: Add `mage test` target + first middleware tests
  - Acceptance: `mage test` runs `go test -race ./...`; tests cover `DirFor` map and root redirect.
  - Verify: `mage test` exit 0.
  - Files: `magefile.go`, `internal/server/server_test.go` (or split i18n_test.go + server_test.go)

- [ ] Task 00.9: README + phase 00 gate
  - Acceptance: README documents requirements, setup, mage targets; `mage clean build && mage test` green; manual `/en` vs `/ar` mirror check passes.
  - Verify: both commands exit 0; commit body records the manual mirror check.
  - Files: `README.md`

---

## Phase 01 — Auth & RBAC

- [ ] Task 01.1: Role + Permission ent schemas + User.roles edge
  - Acceptance: `ent/schema/role.go` (name unique, description, edge to permissions + users), `ent/schema/permission.go` (code unique, description, edge to roles); User gains `edge.To("roles", Role.Type)`; `mage ent` regenerates.
  - Verify: `mage ent` exit 0; `mage clean build` exit 0.
  - Files: `ent/schema/role.go`, `ent/schema/permission.go`, `ent/schema/user.go`, `ent/**` (generated)

- [ ] Task 01.2: Stdlib session store + stdlib PBKDF2 password hashing
  - Acceptance: `internal/auth.SessionStore` signs/verifies HMAC-SHA256 cookies carrying uid+locale+expiry; `internal/auth.HashPassword`/`VerifyPassword` use a stdlib-only PBKDF2 over SHA-256; roundtrip tests pass.
  - Verify: `go test -race ./internal/auth/...` exit 0.
  - Files: `internal/auth/session.go`, `internal/auth/password.go`, `internal/auth/auth_test.go`

- [ ] Task 01.3: Auth middleware + login/logout handlers + persisted locale root redirect
  - Acceptance: `Server.RequireAuth` redirects unauthenticated to `/{lang}/login`; `GET/POST /{lang}/login` validates credentials, writes session, redirects to user's locale dashboard; `POST /{lang}/logout` clears session; root redirect reads session locale when present.
  - Verify: manual curl flow — protected route 302→login; POST login 302→dashboard; GET dashboard 200 with cookie; POST logout 302→login.
  - Files: `internal/server/auth.go`, `internal/server/server.go`, `internal/server/router.go`, `internal/server/locale.go`, `internal/server/templates/pages/login.html`, `internal/server/templates/layout.html`, `internal/i18n/locales/active.{en,ar}.toml`

- [ ] Task 01.4: Seed admin user + default RBAC roles/perms
  - Acceptance: `internal/db` seed helpers create an `admin` user (env-configurable password, default `admin`) and `admin` role with every permission code; `main.go` calls seeds at startup; log line reports seeded credentials.
  - Verify: run binary; `curl` login as `admin@example.com`/`admin` → 302 to dashboard.
  - Files: `internal/db/seed.go`, `internal/auth/rbac.go` (permission code constants), `cmd/inventory/main.go`

- [ ] Task 01.5: `RequirePermission` middleware + RBAC tests
  - Acceptance: `Server.RequirePermission(code, next)` checks the user's roles for the permission; returns 403 when missing; service test covers both granted and denied cases against in-memory ent sqlite.
  - Verify: `mage test` exit 0.
  - Files: `internal/server/auth.go`, `internal/server/auth_test.go` (or rbac_test.go)

- [ ] Task 01.6: Profile page that persists locale preference in DB
  - Acceptance: `GET /{lang}/profile` shows current locale; `POST /{lang}/profile` updates `users.locale`, refreshes session locale, redirects to the new-locale profile; layout nav shows login/logout conditionally.
  - Verify: switch locale on profile → root redirect honors new locale after re-login; both TOML files updated with new keys.
  - Files: `internal/server/auth.go`, `internal/server/templates/pages/profile.html`, `internal/server/templates/layout.html`, `internal/i18n/locales/active.{en,ar}.toml`

- [ ] Task 01.7: Phase 01 gate
  - Acceptance: `mage clean build && mage test` green; login + persisted-locale redirect verified in both `/en` and `/ar`.
  - Verify: both commands exit 0; commit body records manual verification.
  - Files: `tasks/todo.md` (checkbox update), any test/seed fixes

---

## Phase 02 — Catalog

- [ ] Task 02.1: ProductCategory self-referential tree + CRUD
  - Acceptance: `ProductCategory` ent schema (name, optional parent edge to self); handlers for list (tree view), create, edit, delete, re-parent; permission-gated; htmx partials for inline edit.
  - Verify: `mage test` for category service; manual create + re-parent in `/en` and `/ar`.
  - Files: `ent/schema/product_category.go`, `internal/server/catalog.go`, `internal/service/category.go`, `internal/server/templates/pages/category*.html`, `internal/i18n/locales/active.{en,ar}.toml`

- [ ] Task 02.2: Product schema + list + detail + create/edit/delete
  - Acceptance: `Product` ent schema (sku unique, name, description, category edge, uom, reorder min/max, default location optional); full CRUD UI with permission gates; list shows category + on-hand count placeholder (filled in P03).
  - Verify: `mage test`; manual CRUD roundtrip both locales.
  - Files: `ent/schema/product.go`, `internal/server/catalog.go`, `internal/service/product.go`, `internal/server/templates/pages/product*.html`, locales

- [ ] Task 02.3: ProductParameter + per-category parameter templates
  - Acceptance: `ProductParameter` schema (key, value, type: text/num/date/list, product edge, category template edge); inline edit on product detail; list filter by parameter value.
  - Verify: `mage test`; manual add parameter to product via detail page.
  - Files: `ent/schema/product_parameter.go`, `internal/server/catalog.go`, `internal/service/parameter.go`, templates, locales

- [ ] Task 02.4: Tag (polymorphic) + filter by tag
  - Acceptance: `Tag` schema + polymorphic join to Product (ent mixin or join table); tag chips on product list; click filters list.
  - Verify: `mage test`; manual tag + filter.
  - Files: `ent/schema/tag.go`, `ent/schema/tag_assignment.go` (or mixin), `internal/server/catalog.go`, templates, locales

- [ ] Task 02.5: Attachment (file upload) to product
  - Acceptance: `Attachment` schema (filename, mime, path, product edge); upload to local `uploads/` (gitignored); download handler; delete removes file + row in one tx.
  - Verify: `mage test`; manual upload + download a small file; confirm `uploads/` ignored.
  - Files: `ent/schema/attachment.go`, `internal/server/catalog.go`, `internal/service/attachment.go`, `.gitignore`, templates, locales

- [ ] Task 02.6: Product list search + pagination (htmx)
  - Acceptance: list supports `?q=` text search across sku/name and `?page=` pagination via htmx partial swaps; pagination controls mirrored in RTL.
  - Verify: `mage test` for pagination service; manual paginate in `/en` and `/ar`, confirm mirror.
  - Files: `internal/server/catalog.go`, `internal/service/product.go`, `internal/server/templates/pages/product_list.html`, locales

- [ ] Task 02.7: Phase 02 gate
  - Acceptance: `mage clean build && mage test` green; catalog CRUD verified both locales; `mage i18n` extracts + merges all new keys.
  - Verify: commands exit 0; both TOML files have all keys used by catalog templates.
  - Files: `tasks/todo.md`, locales

---

## Phase 03 — Stock & Locations

- [ ] Task 03.1: Location self-referential tree + CRUD + tree view
  - Acceptance: `Location` schema (name, type: facility/zone/bin, parent edge to self); CRUD UI; tree view partial; permission gates.
  - Verify: `mage test`; manual create facility→zone→bin.
  - Files: `ent/schema/location.go`, `internal/server/stock.go`, `internal/service/location.go`, templates, locales

- [ ] Task 03.2: StockItem schema + create + detail
  - Acceptance: `StockItem` schema (product edge, location edge, lot optional, serial optional, quantity, expiry optional, status, received date); create + detail UI.
  - Verify: `mage test`; manual create stock item.
  - Files: `ent/schema/stock_item.go`, `internal/server/stock.go`, `internal/service/stock.go`, templates, locales

- [ ] Task 03.3: StockMovement immutable ledger + append-only service
  - Acceptance: `StockMovement` schema (from-loc, to-loc, quantity, reason, ref-doc, user edge, timestamp, immutable); `service.Movement.Create` enforces append-only (no update/delete) inside an ent tx that also adjusts the target StockItem quantity; unit tests cover debit, credit, and concurrent tx.
  - Verify: `go test -race ./internal/service/...` exit 0; ledger row count increments, never mutates.
  - Files: `ent/schema/stock_movement.go`, `internal/service/movement.go`, `internal/service/movement_test.go`

- [ ] Task 03.4: Stock adjust handler (count correction)
  - Acceptance: `POST /{lang}/stock/{id}/adjust` writes a StockMovement reason "adjustment" and updates StockItem qty in one tx; htmx partial refreshes the detail row.
  - Verify: `mage test`; manual adjust + movement history visible.
  - Files: `internal/server/stock.go`, `internal/service/stock.go`, templates, locales

- [ ] Task 03.5: Stock move handler (loc→loc, two movements one tx)
  - Acceptance: `POST /{lang}/stock/{id}/move` writes debit at source + credit at target in one tx; UI pickers for target location; quantity validated against on-hand.
  - Verify: `mage test` for move tx; manual move between two bins.
  - Files: `internal/server/stock.go`, `internal/service/stock.go`, templates, locales

- [ ] Task 03.6: CustomState schema + status transitions with audit
  - Acceptance: `CustomState` schema (entity type, name, key); status transition rules in service; every status change writes a StockMovement reason "state-change"; UI to change state with permission gate.
  - Verify: `mage test` for transition rules (allowed + rejected).
  - Files: `ent/schema/custom_state.go`, `internal/service/stock.go`, templates, locales

- [ ] Task 03.7: FEFO helper + use in stock list view
  - Acceptance: `service.Stock.FEFO` returns stock for a product sorted by expiry asc (nulls last); product detail "available stock" uses it; tests cover mixed null/expiry ordering.
  - Verify: `go test ./internal/service/...` for FEFO; manual view product with mixed lots.
  - Files: `internal/service/stock.go`, `internal/service/stock_test.go`, `internal/server/stock.go`, templates

- [ ] Task 03.8: Stock list filters (product, location, lot, status) htmx
  - Acceptance: list supports filters via query params, htmx partial swap; filters mirrored in RTL.
  - Verify: `mage test`; manual filter in both locales.
  - Files: `internal/server/stock.go`, `internal/service/stock.go`, templates, locales

- [ ] Task 03.9: Phase 03 gate
  - Acceptance: `mage clean build && mage test` green; stock move + adjust + FEFO verified both locales; `mage i18n` merges keys.
  - Verify: commands exit 0; both TOML files complete.
  - Files: `tasks/todo.md`, locales

---

## Phase 04 — Procurement

- [ ] Task 04.1: Company schema (supplier/manufacturer/customer roles) + CRUD
  - Acceptance: `Company` schema (name, roles: many bool flags or a roles join — pick the simpler at impl time, document in commit), contact fields; CRUD UI.
  - Verify: `mage test`; manual create supplier + manufacturer + customer.
  - Files: `ent/schema/company.go`, `internal/server/procurement.go`, `internal/service/company.go`, templates, locales

- [ ] Task 04.2: SupplierProductPrice (supplier × product × price break) schema + UI
  - Acceptance: schema + create/edit on product detail "suppliers" tab; price break qty tiers.
  - Verify: `mage test`; manual add supplier price to a product.
  - Files: `ent/schema/supplier_product_price.go`, `internal/server/procurement.go`, `internal/service/pricing.go`, templates, locales

- [ ] Task 04.3: PurchaseOrder + POLine schemas + create/list/detail
  - Acceptance: `PurchaseOrder` (supplier edge, status, dates, total), `POLine` (po edge, product edge, qty, unit price); create/list/detail UI; permission gate.
  - Verify: `mage test`; manual raise a PO with 3 lines.
  - Files: `ent/schema/purchase_order.go`, `ent/schema/po_line.go`, `internal/server/procurement.go`, `internal/service/po.go`, templates, locales

- [ ] Task 04.4: PO state machine (draft → submitted → approved → received → closed)
  - Acceptance: transitions in service with rules; each transition audited (POEvent or movement reason); UI buttons for allowed transitions only.
  - Verify: `mage test` for transitions (allowed + rejected).
  - Files: `internal/service/po.go`, `internal/server/procurement.go`, templates, locales

- [ ] Task 04.5: GoodsReceipt + receive against PO (creates StockMovement + StockItem)
  - Acceptance: `GoodsReceipt` schema + receiving UI per PO line; receiving writes a StockMovement (reason "receipt") and creates/updates StockItem in one tx; partial receiving supported.
  - Verify: `mage test` for receipt tx; manual receive a PO line → stock on-hand increases.
  - Files: `ent/schema/goods_receipt.go`, `internal/service/receipt.go`, `internal/server/procurement.go`, templates, locales

- [ ] Task 04.6: PutawayTask + assign received stock to bin location
  - Acceptance: `PutawayTask` schema (receipt edge, target bin, status); UI to assign + confirm putaway; confirmation writes a StockMovement "putaway".
  - Verify: `mage test`; manual putaway flow.
  - Files: `ent/schema/putaway_task.go`, `internal/service/putaway.go`, `internal/server/procurement.go`, templates, locales

- [ ] Task 04.7: Invoice schema + match against PO/receipts
  - Acceptance: `Invoice` schema (supplier, number, lines matching POLine/receipt); create + match UI; status flags.
  - Verify: `mage test`; manual create invoice matched to a received PO.
  - Files: `ent/schema/invoice.go`, `ent/schema/invoice_line.go`, `internal/service/invoice.go`, `internal/server/procurement.go`, templates, locales

- [ ] Task 04.8: Reorder report + suggested PO
  - Acceptance: report lists products below reorder min per location; "create suggested PO" button generates a draft PO with computed qty.
  - Verify: `mage test` for reorder logic; manual generate suggested PO.
  - Files: `internal/service/report.go`, `internal/server/procurement.go`, templates, locales

- [ ] Task 04.9: Phase 04 gate
  - Acceptance: `mage clean build && mage test` green; full PO→receive→putaway→invoice flow verified both locales.
  - Verify: commands exit 0; both TOML files complete.
  - Files: `tasks/todo.md`, locales

---

## Phase 05 — Outbound

- [ ] Task 05.1: Requisition + RequisitionLine + CRUD + approval flow
- [ ] Task 05.2: Picklist + PicklistLine schemas + FEFO allocation service (unit tested)
- [ ] Task 05.3: Picking handler (confirm pick → reduce stock via StockMovement)
- [ ] Task 05.4: Shipment schema (pack: pallet/box, carrier, tracking) + packing UI
- [ ] Task 05.5: Shipping handler (mark shipped → audit)
- [ ] Task 05.6: Return schema + handler (reason, inspection, restock/dispose)
- [ ] Task 05.7: Phase 05 gate

(Fuller Acceptance/Verify/Files lines filled in when phase 05 starts; the entity scope above is the deliverable contract.)

---

## Phase 06 — Manufacturing

- [ ] Task 06.1: BOM + BOMLine (multi-level) + CRUD
- [ ] Task 06.2: BuildOrder schema + state machine
- [ ] Task 06.3: Build allocate (consume components → produce assembly, StockMovement tx)
- [ ] Task 06.4: Build complete handler + audit
- [ ] Task 06.5: BOM import from CSV (stdlib `encoding/csv`)
- [ ] Task 06.6: Phase 06 gate

---

## Phase 07 — Asset Tracking

- [ ] Task 07.1: Asset schema (serialized, asset_tag unique, assigned_to, status) + CRUD
- [ ] Task 07.2: Checkout/Return handlers (StockMovement + Asset status)
- [ ] Task 07.3: License + LicenseSeat + checkout/return seat
- [ ] Task 07.4: Depreciation schema + schedule generation
- [ ] Task 07.5: Asset QR (stdlib SVG) + label print view
- [ ] Task 07.6: Phase 07 gate

---

## Phase 08 — Operations & QR

- [ ] Task 08.1: Cold chain fields on Product + Location + compliance check service
- [ ] Task 08.2: CycleCount + CycleCountLine + suggested-count generation
- [ ] Task 08.3: Cycle count entry + variance report
- [ ] Task 08.4: StockTransfer bulk (multi-line bin→bin, facility→facility)
- [ ] Task 08.5: RecallLot + quarantine handler
- [ ] Task 08.6: QR generator (stdlib SVG) for products, stock, locations, assets, bins
- [ ] Task 08.7: QR scan landing route `/{lang}/scan/{code}` → resolve + redirect
- [ ] Task 08.8: Phase 08 gate

---

## Phase 09 — Reports & Export

- [ ] Task 09.1: Dashboard widgets (stock value, low stock, expiring, fast movers)
- [ ] Task 09.2: Consumption report (StockMovement aggregation by product/location/month)
- [ ] Task 09.3: Stockout report + stock alerts
- [ ] Task 09.4: CSV export (stdlib `encoding/csv`) for stock, movements, products
- [ ] Task 09.5: JSON export endpoint (`encoding/json`)
- [ ] Task 09.6: Phase 09 gate

---

## Phase 10 — Postgres Migration (final)

- [ ] Task 10.1: Add Postgres dialect driver via ent (no direct driver use in app code)
- [ ] Task 10.2: Switch `INVENTORY_DB` default to a Postgres URL; document env in README
- [ ] Task 10.3: Generate versioned migrations (`ent migrate --target` or equivalent)
- [ ] Task 10.4: `mage migrate` target running versioned migrations
- [ ] Task 10.5: Final clean build + full test suite against Postgres
- [ ] Task 10.6: Tag release `v0.1.0`; push tag

---

## Notes for the executing agent

- Start each task session by re-reading `SPEC.md` (the 6-area contract) + the specific task line in this file. Do not load the whole project history into context.
- Search the codebase (grep/glob) before assuming something isn't implemented — ripgrep is non-deterministic and prior tasks may have left partial work.
- No stubs. A task is done only when its `Verify:` command passes against a real implementation.
- One commit per task. Commit body includes the verification evidence (command + exit code or output snippet). Reference the task ID in the body (e.g. `Task 00.1`).
- After each phase's gate task, push to remote.
- If a task reveals a spec gap, update `SPEC.md` first, then implement.