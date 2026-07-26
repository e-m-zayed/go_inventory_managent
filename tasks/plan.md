# Plan

## Approach

The project is decomposed into 11 phases. Each phase is a vertical slice that produces working, testable software on its own and unlocks the next. Phases are ordered by dependency: nothing in phase N requires code from phase N+1, and most of phase N+1 depends on phase N.

Within a phase, tasks are ordered by dependency too. Independent tasks within a phase could in principle run as a parallel wave (one agent per task, separate worktrees), but the default is sequential execution — the simplest safe path. Parallelism is only worth it inside phases 2, 3, 4 where multiple independent entities can be scaffolded at once; the task notes flag those.

The decomposition follows two rules from the spec-driven-development skill:
- Each task is completable in a single focused session and touches ≤ ~5 files.
- Each task ends with a runnable verification command, and the commit body carries the evidence.

## Phase Overview

| Phase | What it delivers | Depends on | Verification gate |
|---|---|---|---|
| 00 — Scaffold | Go module, mage, ent (with `intercept` feature flag), sqlite via ent, htmx, daisyUI, i18n bundle, mirrored dashboard, required-env guard for `INVENTORY_SESSION_SECRET` | nothing | `mage clean build && mage test`; `/en/` LTR, `/ar/` RTL; app exits non-zero when secret unset |
| 01 — Auth & RBAC + Tenant Foundation | Stdlib sessions, login/logout, Role+Permission, persisted locale, profile page, `Tenant` schema + `TenantMixin` + fail-closed interceptor + tenant middleware, root tenant + admin seed, tenant-isolation service test | 00 | login flow works; root redirect honors saved locale; cross-tenant query test passes; `mage test` |
| 02 — Catalog | ProductCategory tree, Product CRUD, ProductParameter, Tag, Attachment (all tenant-scoped via mixin) | 01 | create/list/edit/delete a product in browser; i18n strings present; cross-tenant isolation holds |
| 03 — Stock & Locations | Location tree, StockItem, immutable StockMovement ledger, status machine, FEFO, adjust/move/count (all tenant-scoped) | 02 | move stock between locations; ledger row count matches; cross-tenant isolation holds |
| 04 — Procurement | Company roles, supplier pricing, PurchaseOrder + lines, GoodsReceipt, Putaway, Invoice, reorder report (all tenant-scoped) | 03 | receive a PO → stock increases; movement audit trail correct; isolation holds |
| 05 — Outbound | Requisition, Picklist (FEFO), Picking, Shipment (pack/ship), Return (all tenant-scoped) | 03, 04 | pick + ship an order → stock decreases; return restocks; isolation holds |
| 06 — Manufacturing | BOM + lines, BuildOrder, consume→produce, BOM CSV import (all tenant-scoped) | 03 | complete a build → component stock down, assembly stock up; isolation holds |
| 07 — Asset Tracking | Serialized Asset, checkout/return, License + seats, Depreciation, asset QR (all tenant-scoped) | 02, 03 | checkout an asset to a user; return it; seat counts correct; isolation holds |
| 08 — Operations & QR + Barcode Scan | Cold chain fields, CycleCount, StockTransfer bulk, RecallLot, gozxing-based QR/barcode generation + scan route, browser-side frame capture (getUserMedia + htmx upload, server decodes) | 03, 07 | QR scans to the right entity; camera-scan page uploads a frame and lands on the entity; cycle count variance report correct |
| 09 — Reports & Export | Dashboard widgets, consumption/stockout reports, CSV + JSON export (all tenant-scoped automatically via interceptor) | 03–08 | export endpoint returns valid CSV and JSON scoped to the caller's tenant; dashboard renders |
| 10 — Postgres Migration | Dialect swap, versioned migrations, `mage migrate`, v0.1.0 tag | all | app runs against Postgres; `mage migrate` clean; full test suite green including tenant isolation |
| 11 — Tenant Admin | Create + list tenants, create tenant admins, switch active tenant for super-admin users, per-tenant feature flags stub | 01, 10 | super-admin can create a second tenant and its admin; that admin logs in and sees only their tenant's data |

## Implementation Order

```
00 ── 01 ── 02 ── 03 ──┬── 04 ── 05
                      │
                      ├── 06
                      │
                      ├── 07 ── 08
                      │
                      └── 09
                          
                      10 (after 00–09)
                      11 (after 01 + 10)
```

- 00 enables ent's `intercept` feature flag so 01 can register the tenant interceptor.
- 01 introduces the `Tenant` schema, `TenantMixin`, fail-closed interceptor, and tenant middleware. Every later phase embeds the mixin in new schemas without re-deriving the isolation mechanism.
- 02 depends on 01 (auth + tenant gates every CRUD page).
- 03 depends on 02 (stock references products).
- 04, 05, 06, 07 all depend on 03 (movements + stock).
- 07 depends on 02 (assets reference products) and 03 (checkout/return writes movements).
- 08 depends on 03 and 07 (QR for assets + stock + locations) and adds browser-side barcode scanning.
- 09 depends on everything it reports on (03–08); tenant scoping is automatic via the interceptor.
- 10 depends on the full feature set being frozen.
- 11 depends on 01 (tenant foundation) and 10 (prod DB), since per-tenant admin tooling is the last thing to harden before release.

## Risks & Mitigations

- **Tenant isolation silently broken.** Mitigation: a fail-closed interceptor (no tenant in context → query returns nothing) is registered once at startup; a service test in Phase 01 opens two tenants and asserts cross-tenant reads return zero rows and cross-tenant mutations are rejected. Every later phase re-runs the same isolation test in its gate.
- **Ent schema churn late in the project.** Mitigation: phases 02–03 lock the core entities; later phases add entities and edges but avoid editing earlier schemas unless an explicit spec update is approved.
- **RTL mirroring silently breaking.** Mitigation: every phase's verification gate includes a manual `/en` vs `/ar` side-by-side check; the spec's "logical CSS only" rule is a hard boundary.
- **i18n keys drifting from code.** Mitigation: `mage i18n` extracts IDs and merges into both locale files at the end of each phase; a phase is not done until both TOML files contain every key used.
- **Barcode scanning browser variance.** Mitigation: scanning is server-side via gozxing — the browser only captures a frame and uploads it. Works in every browser that supports `getUserMedia` + file upload; no feature detection, no vendored JS decoder.
- **Ent SQLite vs Postgres behavioral differences.** Mitigation: keep all DB access behind Ent (no dialect-specific code) until phase 10; phase 10 is dedicated to the dialect swap so surprises surface there, not mid-project. The tenant interceptor's `WhereP` predicate is dialect-agnostic.
- **Context bloat on long runs.** Mitigation: one task per session; the agent re-reads SPEC.md + tasks/todo.md at the start of each task rather than carrying the whole project in context.

## Parallelism Opportunities

Default is sequential. If parallel execution is desired:
- Phase 02: ProductCategory, ProductParameter, Tag, Attachment can be scaffolded in parallel once Product exists. Each new schema must embed `TenantMixin` — the parallel agents share the mixin file, so serialize the first entity of each wave and parallelize the rest.
- Phase 04: Supplier/Manufacturer (Company), Invoice, and reorder report can proceed in parallel once PO + POLine exist.
- Phase 07: License/seats, Depreciation, and Asset QR can proceed in parallel once Asset exists.
- Phase 08: QR generator, scan route, and the browser-side scan page are mutually independent and can be parallelized.

Each parallel task is scoped to a distinct entity so worktrees won't collide on the same files.

## Verification Checkpoints

After every phase:
1. `mage clean build` — exit 0.
2. `mage test` — green, including the tenant-isolation test from Phase 01 (re-run every phase).
3. Manual: log in as seeded admin, exercise the phase's primary workflow in `/en/` and `/ar/`, confirm mirror layout and translated strings.
4. Commit phase gate with evidence in the body; push.
5. Update `tasks/todo.md` checkboxes for the completed phase.