# Go Inventory & Warehouse Management System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a thorough, easy-to-use inventory and warehouse management web app in Go covering catalog, stock, procurement, outbound, manufacturing, asset tracking, and operations — bi-directional localized UI with strict git discipline.

**Architecture:** Single local Go binary serving HTML over `net/http`. Ent ORM owns all DB access (no raw SQL, no direct driver calls). HTMX drives partial UI updates server-rendered via `html/template`. daisyUI v5 + Tailwind provide styling with logical CSS for true RTL/LTR mirroring. Locale is path-prefixed (`/{lang}/...`) and persisted per-user in the database. magefile orchestrates build/run/test/migrate/css/i18n targets. SQLite now via Ent dialect; swap to Postgres dialect as final step.

**Tech Stack:**
- Go stdlib only (net/http, html/template, crypto, encoding, etc.)
- `entgo.io/ent` — sole DB layer
- `github.com/donseba/go-htmx` — htmx Go helpers
- `github.com/nicksnyder/go-i18n/v2` + TOML — i18n
- `github.com/magefile/mage` — build system
- Tailwind CSS v4 + daisyUI v5 — styling (compiled via standalone Tailwind CLI invoked from mage)
- SQLite (Ent dialect) for dev → Postgres (Ent dialect) for prod (final step)

## Global Constraints

- **Module path:** `github.com/e-m-zayed/go_inventory_managent`
- **Go version:** 1.22+ (uses `net/http.ServeMux` pattern routing, stdlib only)
- **No external libs** beyond: `entgo.io/ent`, `github.com/donseba/go-htmx`, `github.com/nicksnyder/go-i18n/v2`, `github.com/magefile/mage`, `github.com/BurntSushi/toml` (i18n TOML loader), and their transitive deps. No web framework (no chi/gin/echo). No auth lib (stdlib session). No password lib beyond `crypto/*`.
- **DB layer:** Ent only. Never import a SQL driver directly. Never write raw SQL. Migrations via `client.Schema.Create(ctx)` (auto-migrate) until versioned migrations explicitly added.
- **Locale:** URL path prefix `/{lang}/...`. Registry: `en`, `ar` (extensible). Root `/` → redirect to default (`en`) unless user has saved locale preference, then their saved locale. Locale preference persisted in DB per-user (User.Locale column).
- **Mirrored UI:** `dir` attribute on `<html>` set from locale. **Logical CSS properties only** for layout-bearing styles — `ms-* me-* ps-* pe-* start-* end-* text-start text-end float-start float-end border-s border-e inset-inline-*`. Zero `ml- mr- left right` for layout. Directional icons flipped via `[dir=rtl] .icon-fwd { transform: scaleX(-1) }`. daisyUI v5 components used as-is (dir-aware). `/en/` and `/ar/` pages must render as exact visual mirror — verified each phase.
- **Git discipline:** Every task = at least one commit. Mitchell Hashimoto style messages: imperative, lowercase summary line ≤72 chars, blank line, body explaining why. One logical change per commit. No `--amend` unless asked. Phase boundary = `mage build` clean before commit.
- **Build verification:** After every phase, run `mage clean build` and confirm exit 0 before final commit of that phase.
- **Localization files:** `locales/active.en.toml` (baseline), `locales/active.ar.toml` (parallel). `mage i18n` runs `goi18n extract` then `goi18n merge`. Never hardcode user-facing English in templates — always `{{T "key"}}`.
- **QR codes:** Stdlib-only SVG generation (no external QR lib). QR for products, stock items, locations.

---

## File Structure (target, built up across phases)

```
go_inventory_managent/
├── AGENTS.md                      # project rules (committed)
├── README.md                      # setup + run (added P0)
├── .gitignore                     # Go + node + build artifacts
├── magefile.go                    # all build targets
├── go.mod
├── go.sum
├── cmd/inventory/main.go          # entrypoint
├── ent/
│   ├── generate.go                # //go:generate go run entgo.io/ent/cmd/ent generate ./schema
│   └── schema/                    # one .go per entity (added per phase)
├── internal/
│   ├── server/
│   │   ├── server.go             # Server struct, wiring, ListenAndServe
│   │   ├── router.go             # mux setup, route registration
│   │   ├── locale.go             # locale registry, dir map, locale mw
│   │   ├── auth.go               # session mw, login/logout handlers
│   │   ├── htmxmw.go             # go-htmx middleware
│   │   └── render.go             # template render helpers
│   ├── handler/
│   │   ├── auth.go
│   │   ├── dashboard.go
│   │   ├── catalog.go            # P2: product, category, parameter, tag, attachment
│   │   ├── stock.go             # P3
│   │   ├── procurement.go       # P4
│   │   ├── outbound.go          # P5
│   │   ├── build.go             # P6
│   │   ├── asset.go             # P7
│   │   └── operations.go        # P8
│   ├── service/
│   │   └── *.go                  # business logic per domain, ent-backed
│   ├── templ/
│   │   ├── layout.html          # <html dir={{.Dir}} lang={{.Lang}}>
│   │   ├── funcs.go             # T(), path(), switchLang(), dir(), currentLang()
│   │   ├── pages/               # one .html per page
│   │   └── partials/            # htmx fragments
│   ├── i18n/
│   │   ├── bundle.go            # bundle init, LoadMessageFileFS
│   │   └── locales.go          # registry, DefaultLocale, DirFor(lang)
│   ├── config/
│   │   └── config.go           # env-driven (os.Getenv) config struct
│   ├── db/
│   │   └── db.go               # ent.Open wrapper, dialect from config
│   └── qrcode/
│       └── qrcode.go           # stdlib QR SVG generator (P8)
├── locales/
│   ├── active.en.toml
│   └── active.ar.toml
├── web/
│   ├── css/input.css           # @import "tailwindcss"; @plugin "daisyui";
│   ├── css/app.css             # compiled (gitignored)
│   └── vendor/htmx.min.js      # vendored htmx (P0)
└── docs/plans/
    └── 2026-07-26-go-inventory-management.md  # this file
```

---

## Phase / Task Breakdown

Each phase ends with `mage clean build` (exit 0) + phase-summary commit. Each task is independently testable. Git commit per task. Tasks ordered by dependency.

---

### Phase 0 — Project Scaffold & Foundation

**Phase goal:** Go module, mage, ent, sqlite via ent, htmx, daisyUI, i18n with `/en` + `/ar` mirrored dashboard, QR stub. Foundation for all later phases.

#### Task P0.1: Repo init + module + gitignore

**Files:**
- Create: `.gitignore`
- Create: `go.mod` (via `go mod init`)
- Modify: `AGENTS.md` (already present — verify content)

- [ ] **Step 1: Initialize Go module**

```bash
cd "/Users/amz/Documents/Dev/Non code/go_inventory_managent"
go mod init github.com/e-m-zayed/go_inventory_managent
```
Expected: `go.mod` created with `module github.com/e-m-zayed/go_inventory_managent` and `go 1.22`.

- [ ] **Step 2: Write `.gitignore`**

Create `.gitignore`:
```
# Go
/bin/
/dist/
*.exe
*.test
*.out
*.prof

# Build artifacts
/web/css/app.css

# Editor
.vscode/
.idea/
*.swp
.DS_Store

# Env
.env
.env.local

# DB
*.sqlite
*.sqlite-journal
*.db
```

- [ ] **Step 3: Verify AGENTS.md exists with project rules**

```bash
test -f AGENTS.md && head -5 AGENTS.md
```
Expected: first lines of AGENTS.md visible.

- [ ] **Step 4: Commit**

```bash
git add .gitignore go.mod AGENTS.md
git commit -m "init: go module and gitignore

establish go module github.com/e-m-zayed/go_inventory_managent
and baseline ignore rules for build artifacts, editor files,
env, and sqlite dev databases."
```

#### Task P0.2: Magefile — build targets

**Files:**
- Create: `magefile.go`

**Interfaces:**
- Produces: mage targets `build`, `run`, `clean`, `tidy` — used by every later task.

- [ ] **Step 1: Install mage**

```bash
go install github.com/magefile/mage@latest
which mage
```
Expected: mage binary path printed (e.g. `$HOME/go/bin/mage`).

- [ ] **Step 2: Write `magefile.go`**

Create `magefile.go`:
```go
//go:build mage

package main

import (
	"os"
	"path/filepath"

	"github.com/magefile/mage/mg"
	"github.com/magefile/mage/sh"
)

var (
	binaryName = "inventory"
	buildDir   = "bin"
)

// Build compiles the inventory binary into ./bin.
func Build() error {
	if err := os.MkdirAll(buildDir, 0o755); err != nil {
		return err
	}
	return sh.RunV("go", "build", "-o", filepath.Join(buildDir, binaryName), "./cmd/inventory")
}

// Run builds and starts the server.
func Run() error {
	mg.Deps(Build)
	return sh.RunV(filepath.Join(buildDir, binaryName))
}

// Tidy runs go mod tidy.
func Tidy() error {
	return sh.RunV("go", "mod", "tidy")
}

// Clean removes the build directory.
func Clean() error {
	return os.RemoveAll(buildDir)
}
```

- [ ] **Step 3: Add mage dependency**

```bash
go get github.com/magefile/mage
```

- [ ] **Step 4: Commit**

```bash
git add magefile.go go.mod go.sum
git commit -m "build: add magefile with build/run/clean/tidy targets

mage drives all build/run/test/migrate/css/i18n tasks per AGENTS.md.
starting with the four core targets; later tasks add ent, css, i18n,
migrate, test, and check-rtl targets."
```

#### Task P0.3: Ent setup + first schema (User with Locale)

**Files:**
- Create: `ent/generate.go`
- Create: `ent/schema/user.go`
- Modify: `magefile.go` (add `Ent` target)
- Modify: `go.mod` (ent dep)

**Interfaces:**
- Produces: `ent.User` client + `User` schema with fields `name`, `email`, `password_hash`, `locale` (default "en"). Used by P0.5 (auth) and P0.7 (locale persistence).

- [ ] **Step 1: Add ent dependency and init**

```bash
go get entgo.io/ent
go run -mod=mod entgo.io/ent/cmd/ent new User
```
Expected: `ent/schema/user.go` created with stub `User` schema.

- [ ] **Step 2: Write `ent/generate.go`**

Create `ent/generate.go`:
```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

- [ ] **Step 3: Write `ent/schema/user.go`**

Replace `ent/schema/user.go` with:
```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").NotEmpty(),
		field.String("email").Unique().NotEmpty(),
		field.String("password_hash").Sensitive(),
		field.String("locale").Default("en"),
		field.Time("created_at").Default(time.Now).Immutable(),
		field.Time("updated_at").Default(time.Now).UpdateDefault(time.Now),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return nil
}
```
Add `"time"` import.

- [ ] **Step 4: Generate ent code**

```bash
go generate ./ent
```
Expected: `ent/` directory populated with generated Go (client.go, user.go, etc.).

- [ ] **Step 5: Add `Ent` target to magefile**

Append to `magefile.go` before the closing:
```go
// Ent runs ent code generation.
func Ent() error {
	return sh.RunV("go", "generate", "./ent")
}
```

- [ ] **Step 6: Verify build still clean**

```bash
go mod tidy
mage clean build
```
Expected: build succeeds (compiles even though `cmd/inventory` doesn't exist yet — create a placeholder main next task). If build fails because main missing, defer this check to P0.4.

- [ ] **Step 7: Commit**

```bash
git add ent/ magefile.go go.mod go.sum
git commit -m "ent: scaffold ent orm and User schema

User carries name, email, password_hash, and locale fields.
locale defaults to en so every user has a persisted language
preference usable by the locale middleware for root redirect and
for setting dir on the html element."
```

#### Task P0.4: DB layer + main entrypoint

**Files:**
- Create: `internal/config/config.go`
- Create: `internal/db/db.go`
- Create: `cmd/inventory/main.go`

**Interfaces:**
- Produces: `config.Load()` → `*Config{Addr, DBURL, DefaultLocale}`; `db.Open(ctx, url)` → `*ent.Client` with auto-migrate.
- Consumes: `ent.User` from P0.3.

- [ ] **Step 1: Write `internal/config/config.go`**

```go
package config

import "os"

// Config holds runtime configuration loaded from environment.
type Config struct {
	Addr           string // HTTP listen address
	DBURL          string // Ent database URL (sqlite dev default)
	DefaultLocale  string // fallback locale when none specified
}

// Load reads configuration from environment with sensible defaults.
func Load() *Config {
	return &Config{
		Addr:          envOr("INVENTORY_ADDR", ":8080"),
		DBURL:         envOr("INVENTORY_DB", "file:inventory.db?cache=shared&_fk=1"),
		DefaultLocale: envOr("INVENTORY_DEFAULT_LOCALE", "en"),
	}
}

func envOr(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

- [ ] **Step 2: Write `internal/db/db.go`**

```go
package db

import (
	"context"
	"fmt"

	"entgo.io/ent/dialect"
	"github.com/e-m-zayed/go_inventory_managent/ent"

	// sqlite dialect driver (ent dialect abstraction; no direct driver use)
	_ "github.com/mattn/go-sqlite3"
)

// Open creates an ent client and runs auto-migration against the given
// sqlite database URL. Caller must defer client.Close().
func Open(ctx context.Context, url string) (*ent.Client, error) {
	client, err := ent.Open(dialect.SQLite, url)
	if err != nil {
		return nil, fmt.Errorf("open ent sqlite: %w", err)
	}
	if err := client.Schema.Create(ctx); err != nil {
		_ = client.Close()
		return nil, fmt.Errorf("auto migrate: %w", err)
	}
	return client, nil
}
```
Note: `github.com/mattn/go-sqlite3` is the cgo sqlite driver registered via Ent's `dialect.SQLite`. This is the only driver import — accessed only through `ent.Open`. No raw SQL anywhere.

- [ ] **Step 3: Add sqlite driver dep**

```bash
go get github.com/mattn/go-sqlite3
```

- [ ] **Step 4: Write `cmd/inventory/main.go`**

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/e-m-zayed/go_inventory_managent/internal/config"
	"github.com/e-m-zayed/go_inventory_managent/internal/db"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	cfg := config.Load()

	client, err := db.Open(ctx, cfg.DBURL)
	if err != nil {
		log.Fatalf("db: %v", err)
	}
	defer client.Close()

	log.Printf("inventory: db opened at %s", cfg.DBURL)
	log.Printf("inventory: ready (no server yet) on %s", cfg.Addr)

	<-ctx.Done()
	log.Println("inventory: shutting down")
}
```

- [ ] **Step 5: Build and verify**

```bash
go mod tidy
mage clean build
./bin/inventory &
PID=$!
sleep 1
kill $PID
```
Expected: server logs "db opened" and "ready (no server yet)" then exits on signal.

- [ ] **Step 6: Commit**

```bash
git add internal/ cmd/ go.mod go.sum
git commit -m "db: ent sqlite layer and entrypoint

config is env-driven with sqlite dev default. main opens the
ent client, runs auto-migrate, and waits for signal. next task
adds the http server and locale middleware."
```

#### Task P0.5: i18n bundle + locale registry

**Files:**
- Create: `internal/i18n/locales.go`
- Create: `internal/i18n/bundle.go`
- Create: `locales/active.en.toml`
- Create: `locales/active.ar.toml`
- Modify: `magefile.go` (add `I18n` target)
- Modify: `go.mod` (i18n + toml deps)

**Interfaces:**
- Produces: `i18n.Supported` ([]string), `i18n.DirFor(lang)` ("ltr"/"rtl"), `i18n.NewBundle()` → `*i18n.Bundle`, `i18n.Localizer(bundle, lang)` → `*i18n.Localizer`.
- Consumes: none.

- [ ] **Step 1: Add i18n deps**

```bash
go get github.com/nicksnyder/go-i18n/v2/i18n
go get github.com/BurntSushi/toml
go install github.com/nicksnyder/go-i18n/v2/goi18n@latest
```

- [ ] **Step 2: Write `internal/i18n/locales.go`**

```go
package i18n

// Supported lists all locale codes the app understands.
// Add new locales here; everything else derives from this slice.
var Supported = []string{"en", "ar"}

// DefaultLocale is used when no preference is known.
const DefaultLocale = "en"

// IsSupported reports whether the given code is a registered locale.
func IsSupported(lang string) bool {
	for _, s := range Supported {
		if s == lang {
			return true
		}
	}
	return false
}

// DirFor returns the text direction ("ltr" or "rtl") for a locale.
func DirFor(lang string) string {
	switch lang {
	case "ar", "he", "fa", "ur":
		return "rtl"
	default:
		return "ltr"
	}
}
```

- [ ] **Step 3: Write `internal/i18n/bundle.go`**

```go
package i18n

import (
	"embed"
	"fmt"

	"github.com/BurntSushi/toml"
	"github.com/nicksnyder/go-i18n/v2/i18n"
	"golang.org/x/text/language"
)

//go:embed locales/*.toml
var localeFS embed.FS

// NewBundle creates a bundle pre-loaded with all active.*.toml files
// embedded under internal/i18n/locales.
func NewBundle() *i18n.Bundle {
	bundle := i18n.NewBundle(language.English)
	bundle.RegisterUnmarshalFunc("toml", toml.Unmarshal)
	for _, lang := range Supported {
		path := fmt.Sprintf("locales/active.%s.toml", lang)
		if err := bundle.LoadMessageFileFS(localeFS, path); err != nil {
			// Missing active file is fine for stub; log later.
			continue
		}
	}
	return bundle
}

// Localizer returns a localizer for the given language tag.
func Localizer(bundle *i18n.Bundle, lang string) *i18n.Localizer {
	return i18n.NewLocalizer(bundle, lang)
}
```

Wait — embed path must be relative to the file. Adjust: locale files live at repo `locales/`, not under `internal/i18n/`. Move embed. Create `internal/i18n/locales/` and copy files there. Step 4 handles that.

- [ ] **Step 4: Move locale files under embed path**

```bash
mkdir -p internal/i18n/locales
```

Create `internal/i18n/locales/active.en.toml`:
```toml
[NavDashboard]
other = "Dashboard"

[AppTitle]
other = "Inventory Management"

[Welcome]
other = "Welcome to the inventory system."
```

Create `internal/i18n/locales/active.ar.toml`:
```toml
[NavDashboard]
other = "لوحة التحكم"

[AppTitle]
other = "إدارة المخزون"

[Welcome]
other = "مرحبًا بك في نظام المخزون."
```

Also keep top-level `locales/` copies for `goi18n extract`/`merge` workflow (the active files are the source of truth; the embedded copies are built from them by the mage `I18n` target via copy). Simpler: keep ONLY embedded path. Delete top-level `locales/` idea. `goi18n extract` writes to `internal/i18n/locales/active.en.toml`.

Update `bundle.go` embed path: `//go:embed locales/*.toml` (relative to bundle.go at `internal/i18n/bundle.go` → embeds `internal/i18n/locales/*.toml`). Correct.

- [ ] **Step 5: Add `I18n` mage target**

Append to `magefile.go`:
```go
// I18n extracts message IDs from source into the english active file.
func I18n() error {
	if err := sh.RunV("goi18n", "extract", "-outdir", "internal/i18n/locales", "./..."); err != nil {
		return err
	}
	return sh.RunV("goi18n", "merge", "-outdir", "internal/i18n/locales", "internal/i18n/locales/active.en.toml", "internal/i18n/locales/active.ar.toml")
}
```

- [ ] **Step 6: Verify build**

```bash
go mod tidy
mage clean build
```
Expected: clean build.

- [ ] **Step 7: Commit**

```bash
git add internal/i18n magefile.go go.mod go.sum
git commit -m "i18n: bundle loader, locale registry, en/ar stubs

supported locales and dir map live in one place. bundle is built
from embedded active.*.toml so the binary ships with translations.
mage i18n extracts ids and merges into both locale files."
```

#### Task P0.6: Tailwind + daisyUI CSS pipeline

**Files:**
- Create: `web/css/input.css`
- Create: `web/css/.gitignore` (ignore app.css)
- Modify: `magefile.go` (add `CSS` target)
- Create: `web/vendor/htmx.min.js` (vendored)

**Interfaces:**
- Produces: `mage css` compiles `web/css/app.css` (gitignored). Templates reference `/static/css/app.css`.
- Consumes: none.

- [ ] **Step 1: Install Tailwind CLI standalone**

Per Tailwind v4 docs, the standalone CLI avoids Node dependency. Document in README later. Download into project `bin/tailwindcss` is NOT committed (gitignored). For repeatable build, `mage CSS` downloads if missing.

Append to `magefile.go`:
```go
// CSS compiles Tailwind + daisyUI into web/css/app.css.
func CSS() error {
	tw := filepath.Join(buildDir, "tailwindcss")
	if _, err := os.Stat(tw); os.IsNotExist(err) {
		log.Println("downloading tailwindcss standalone cli")
		if err := sh.RunV("curl", "-sSL", "-o", tw,
			"https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-macos-arm64"); err != nil {
			return err
		}
		if err := os.Chmod(tw, 0o755); err != nil {
			return err
		}
	}
	return sh.RunV(tw, "-i", "web/css/input.css", "-o", "web/css/app.css", "--minify")
}
```
Add `"log"` import to magefile.

- [ ] **Step 2: Add npm deps for daisyui (Tailwind v4 plugin via CSS)**

daisyUI v5 is added as a CSS plugin (`@plugin "daisyui"`), not via tailwind.config.js. The standalone Tailwind CLI v4 supports `@plugin` resolution through npm packages. So we need a minimal `package.json` + `node_modules` for daisyui only. Add `node_modules/` to `.gitignore`.

Create `package.json`:
```json
{
  "name": "go-inventory-managent-css",
  "private": true,
  "dependencies": {
    "daisyui": "^5.0.0"
  }
}
```

Update `.gitignore` (append):
```
# Node (CSS tooling only)
node_modules/
package-lock.json
```

- [ ] **Step 3: Write `web/css/input.css`**

```css
@import "tailwindcss";
@plugin "daisyui" {
  themes: light --default, dark --prefersdark;
}
```

- [ ] **Step 4: Install daisyui**

```bash
npm install
```

- [ ] **Step 5: Vendor htmx**

```bash
mkdir -p web/vendor
curl -sSL -o web/vendor/htmx.min.js https://unpkg.com/htmx.org@2.0.4/dist/htmx.min.js
```

- [ ] **Step 6: Build CSS**

```bash
mage css
```
Expected: `web/css/app.css` exists, minified.

- [ ] **Step 7: Commit**

```bash
git add web/css/input.css web/vendor/htmx.min.js package.json magefile.go .gitignore
git commit -m "css: tailwind v4 + daisyui v5 pipeline, vendor htmx

mage css downloads tailwind standalone cli on first run, then
compiles input.css (daisyui plugin) into minified app.css.
daisyui installed via package.json for plugin resolution.
htmx vendored so the app has no runtime cdn dependency."
```

#### Task P0.7: HTTP server + locale middleware + static files

**Files:**
- Create: `internal/server/server.go`
- Create: `internal/server/router.go`
- Create: `internal/server/locale.go`
- Create: `internal/server/htmxmw.go`
- Create: `internal/server/render.go`
- Create: `internal/templ/layout.html`
- Create: `internal/templ/funcs.go`
- Create: `internal/templ/pages/dashboard.html`
- Modify: `cmd/inventory/main.go`

**Interfaces:**
- Produces: `server.New(cfg, client, bundle)` → `*Server`; `(*Server).ListenAndServe()`; locale middleware injects `Localizer`, `Lang`, `Dir` into request context.
- Consumes: `config.Config`, `*ent.Client`, `*i18n.Bundle` from P0.4/P0.5.

- [ ] **Step 1: Write `internal/server/locale.go`**

```go
package server

import (
	"context"
	"net/http"
	"strings"

	"github.com/e-m-zayed/go_inventory_managent/ent"
	"github.com/e-m-zayed/go_inventory_managent/internal/i18n"
	"github.com/nicksnyder/go-i18n/v2/i18n"
)

type ctxKey string

const (
	ctxLang     ctxKey = "lang"
	ctxDir      ctxKey = "dir"
	ctxLocalize ctxKey = "localizer"
)

// LocaleMiddleware extracts {lang} from the path, validates against the
// registry, sets dir + localizer in context, and strips the prefix for
// downstream handlers (via r.PathValue).
func LocaleMiddleware(bundle *i18n.Bundle, client *ent.Client, defaultLocale string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		lang := r.PathValue("lang")
		if !i18n.IsSupported(lang) {
			http.NotFound(w, r)
			return
		}
		loc := i18n.Localizer(bundle, lang)
		ctx := context.WithValue(r.Context(), ctxLang, lang)
		ctx = context.WithValue(ctx, ctxDir, i18n.DirFor(lang))
		ctx = context.WithValue(ctx, ctxLocalize, loc)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// LangFromCtx returns the locale code stored in the request context.
func LangFromCtx(ctx context.Context) string {
	if v, ok := ctx.Value(ctxLang).(string); ok {
		return v
	}
	return i18n.DefaultLocale
}

// DirFromCtx returns "ltr" or "rtl" from the request context.
func DirFromCtx(ctx context.Context) string {
	if v, ok := ctx.Value(ctxDir).(string); ok {
		return v
	}
	return "ltr"
}

// LocalizerFromCtx returns the per-request localizer.
func LocalizerFromCtx(ctx context.Context) *i18n.Localizer {
	if v, ok := ctx.Value(ctxLocalize).(*i18n.Localizer); ok {
		return v
	}
	return nil
}

// RootRedirect redirects "/" to the user's saved locale if logged in,
// otherwise to the default locale.
func RootRedirect(client *ent.Client, defaultLocale string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// P0.7: no auth yet; redirect to default. P1 will read user locale.
		http.Redirect(w, r, "/"+defaultLocale+"/", http.StatusFound)
	}
}

// switchLangPath returns the same path with the locale prefix replaced.
func switchLangPath(r *http.Request, target string) string {
	rest := strings.TrimPrefix(r.URL.Path, "/"+r.PathValue("lang"))
	return "/" + target + rest
}
```

- [ ] **Step 2: Write `internal/server/htmxmw.go`**

```go
package server

import (
	"context"
	"net/http"

	"github.com/donseba/go-htmx"
)

type htmxCtxKey struct{}

// HtmxMiddleware stores the htmx service and per-request handler in
// context so handlers can call h.IsHxRequest() etc. without re-init.
func HtmxMiddleware(svc *htmx.HTMX, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := svc.NewHandler(w, r)
		ctx := context.WithValue(r.Context(), htmxCtxKey{}, h)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// HtmxFromCtx returns the per-request htmx handler.
func HtmxFromCtx(ctx context.Context) *htmx.Handler {
	if v, ok := ctx.Value(htmxCtxKey{}).(*htmx.Handler); ok {
		return v
	}
	return nil
}
```

- [ ] **Step 3: Write `internal/templ/funcs.go`**

```go
package templ

import (
	"html/template"
	"path"
	"strings"

	"github.com/e-m-zayed/go_inventory_managent/internal/i18n"
	"github.com/e-m-zayed/go_inventory_managent/internal/server"
	"github.com/nicksnyder/go-i18n/v2/i18n"
)

// FuncMap returns the template function map used by every template.
func FuncMap() template.FuncMap {
	return template.FuncMap{
		"T": func(loc *i18n.Localizer, key string) string {
			msg, err := loc.Localize(&i18n.LocalizeConfig{MessageID: key})
			if err != nil {
				return key
			}
			return msg
		},
		"path": func(lang, name string, parts ...string) string {
			return "/" + path.Join(append([]string{lang, name}, parts...)...)
		},
		"switchLang": func(r interface{ PathValue(string) string }, target string) string {
			current := r.PathValue("lang")
			rest := strings.TrimPrefix(strings.TrimPrefix(pathValue(r, "lang"), ""), "/"+current)
			return "/" + target + rest
		},
		"dir": server.DirFromCtx,
		"currentLang": server.LangFromCtx,
	}
}

// pathValue is a tiny shim so we don't import *http.Request in templ.
func pathValue(r interface{ PathValue(string) string }, key string) string {
	return r.PathValue(key)
}
```

Simplify: drop the `switchLang` shim, build it directly in templates using `path`. Rewrite funcs.go without switchLang to avoid interface gymnastics:
```go
package templ

import (
	"html/template"

	"github.com/e-m-zayed/go_inventory_managent/internal/server"
	"github.com/nicksnyder/go-i18n/v2/i18n"
)

// FuncMap returns the template function map used by every template.
func FuncMap() template.FuncMap {
	return template.FuncMap{
		"T": func(loc *i18n.Localizer, key string) string {
			msg, err := loc.Localize(&i18n.LocalizeConfig{MessageID: key})
			if err != nil {
				return key
			}
			return msg
		},
		"dir":      server.DirFromCtx,
		"lang":     server.LangFromCtx,
		"localize": server.LocalizerFromCtx,
	}
}
```
Templates use `{{T (localize .) "NavDashboard"}}`, `{{lang .}}`, `{{dir .}}`. The `.` passed is `*http.Request` whose context holds everything.

- [ ] **Step 4: Write `internal/templ/layout.html`**

```html
<!DOCTYPE html>
<html lang="{{lang .Request}}" dir="{{dir .Request}}" data-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{T (localize .Request) "AppTitle"}}</title>
  <link rel="stylesheet" href="/static/css/app.css">
  <script src="/static/vendor/htmx.min.js"></script>
</head>
<body class="min-h-screen bg-base-200 text-base-content">
  <header class="navbar bg-base-100 shadow">
    <div class="flex-1">
      <a class="btn btn-ghost text-xl" href="{{path "/"}}" hx-boost="true">
        {{T (localize .Request) "AppTitle"}}
      </a>
    </div>
    <nav class="flex-none">
      <ul class="menu menu-horizontal">
        <li><a href="{{path "/dashboard"}}">{{T (localize .Request) "NavDashboard"}}</a></li>
      </ul>
      <div class="ps-2">
        <a class="btn btn-ghost btn-sm" href="/en{{requestPath .Request}}">EN</a>
        <a class="btn btn-ghost btn-sm" href="/ar{{requestPath .Request}}">AR</a>
      </div>
    </nav>
  </header>
  <main class="container mx-auto p-4">
    {{template "content" .}}
  </main>
</body>
</html>
```
Add helper funcs `path` (returns `/lang/rest`), `requestPath` (returns `r.URL.Path` stripped of `/{lang}` prefix). Update funcs.go:
```go
"path": func(r interface{ PathValue(string) string; URL interface{ Path string } }, rest string) string {
	// build /<lang>/<rest>
	...
},
"requestPath": func(r *http.Request) string {
	return strings.TrimPrefix(r.URL.Path, "/"+r.PathValue("lang"))
},
```
To keep types clean, accept `*http.Request` directly in funcs and pass `.Request` from template data.

Final funcs.go (clean version):
```go
package templ

import (
	"html/template"
	"net/http"
	"strings"

	"github.com/e-m-zayed/go_inventory_managent/internal/server"
	"github.com/nicksnyder/go-i18n/v2/i18n"
)

// FuncMap returns the template function map used by every template.
func FuncMap() template.FuncMap {
	return template.FuncMap{
		"T": func(loc *i18n.Localizer, key string) string {
			msg, err := loc.Localize(&i18n.LocalizeConfig{MessageID: key})
			if err != nil {
				return key
			}
			return msg
		},
		"dir":         server.DirFromCtx,
		"lang":        server.LangFromCtx,
		"localize":    server.LocalizerFromCtx,
		"requestPath": requestPath,
		"path":        pathFor,
	}
}

func requestPath(r *http.Request) string {
	return strings.TrimPrefix(r.URL.Path, "/"+r.PathValue("lang"))
}

func pathFor(r *http.Request, rest string) string {
	lang := r.PathValue("lang")
	return "/" + lang + rest
}
```

- [ ] **Step 5: Write `internal/templ/pages/dashboard.html`**

```html
{{define "content"}}
<div class="card bg-base-100 shadow">
  <div class="card-body">
    <h1 class="text-2xl font-semibold">{{T (localize .Request) "NavDashboard"}}</h1>
    <p>{{T (localize .Request) "Welcome"}}</p>
    <p class="mt-2 text-sm opacity-70">dir = {{dir .Request}} · lang = {{lang .Request}}</p>
  </div>
</div>
{{end}}
```

- [ ] **Step 6: Write `internal/server/render.go`**

```go
package server

import (
	"html/template"
	"io/fs"
	"net/http"
	"path"

	"github.com/e-m-zayed/go_inventory_managent/internal/templ"
)

// Renderer holds parsed templates. Templates live under internal/templ.
type Renderer struct {
	layout   *template.Template
	pages    *template.Template
}

// NewRenderer parses the layout and all page templates.
func NewRenderer(templateFS fs.FS) (*Renderer, error) {
	funcs := templ.FuncMap()
	layoutBytes, err := fs.ReadFile(templateFS, "layout.html")
	if err != nil {
		return nil, err
	}
	layoutTmpl, err := template.New("layout").Funcs(funcs).Parse(string(layoutBytes))
	if err != nil {
		return nil, err
	}
	pagesTmpl := template.Must(layoutTmpl.Clone())
	entries, err := fs.ReadDir(templateFS, "pages")
	if err != nil {
		return nil, err
	}
	for _, e := range entries {
		b, err := fs.ReadFile(templateFS, path.Join("pages", e.Name()))
		if err != nil {
			return nil, err
		}
		if _, err := pagesTmpl.Parse(string(b)); err != nil {
			return nil, err
		}
	}
	return &Renderer{layout: layoutTmpl, pages: pagesTmpl}, nil
}

// PageData is passed to every page template.
type PageData struct {
	Request *http.Request
	// fields added by handlers later (entities, form, errors)
}

// Render writes the named page into the layout using PageData.
func (r *Renderer) Render(w http.ResponseWriter, req *http.Request, page string) error {
	return r.pages.ExecuteTemplate(w, "layout", &PageData{Request: req})
}
```

- [ ] **Step 7: Write `internal/server/server.go` and `router.go`**

`internal/server/server.go`:
```go
package server

import (
	"net/http"

	"github.com/donseba/go-htmx"
	"github.com/e-m-zayed/go_inventory_managent/ent"
	"github.com/e-m-zayed/go_inventory_managent/internal/config"
	"github.com/nicksnyder/go-i18n/v2/i18n"
)

// Server wires config, db client, i18n bundle, htmx, and renderer.
type Server struct {
	cfg     *config.Config
	client  *ent.Client
	bundle  *i18n.Bundle
	htmx    *htmx.HTMX
	renderer *Renderer
}

func New(cfg *config.Config, client *ent.Client, bundle *i18n.Bundle, renderer *Renderer) *Server {
	return &Server{
		cfg:      cfg,
		client:   client,
		bundle:   bundle,
		htmx:     htmx.New(),
		renderer: renderer,
	}
}

func (s *Server) ListenAndServe() error {
	return http.ListenAndServe(s.cfg.Addr, s.routes())
}
```

`internal/server/router.go`:
```go
package server

import (
	"embed"
	"io/fs"
	"net/http"

	"github.com/e-m-zayed/go_inventory_managent/internal/i18n"
)

//go:embed static
var staticFS embed.FS

//go:embed templates
var templateFS embed.FS

func (s *Server) routes() http.Handler {
	mux := http.NewServeMux()

	// static files (css, js, images)
	staticSub, _ := fs.Sub(staticFS, "static")
	mux.Handle("GET /static/", http.StripPrefix("/static/", http.FileServer(http.FS(staticSub))))

	// root redirect
	mux.HandleFunc("GET /", s.rootRedirect())

	// locale-prefixed routes — LocaleMiddleware wraps a sub-mux
	subMux := http.NewServeMux()
	subMux.HandleFunc("GET /", s.dashboard)
	subMux.HandleFunc("GET /dashboard", s.dashboard)

	withLocale := LocaleMiddleware(s.bundle, s.client, s.cfg.DefaultLocale, s.htmxMiddleware(subMux))
	mux.Handle("/{lang}/", withLocale)
	mux.Handle("/{lang}/dashboard", withLocale)

	return mux
}

func (s *Server) rootRedirect() http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		http.Redirect(w, r, "/"+s.cfg.DefaultLocale+"/", http.StatusFound)
	}
}

func (s *Server) htmxMiddleware(next http.Handler) http.Handler {
	return HtmxMiddleware(s.htmx, next)
}

func (s *Server) dashboard(w http.ResponseWriter, r *http.Request) {
	if err := s.renderer.Render(w, r, "dashboard.html"); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```
Note: embed layout/pages under `internal/server/templates/` to keep embed close. Move templ files into `internal/server/templates/`. Adjust paths in step 8.

- [ ] **Step 8: Move templates into embed path**

```bash
mkdir -p internal/server/templates/pages internal/server/static/css internal/server/static/vendor
cp internal/templ/layout.html internal/server/templates/layout.html
cp internal/templ/pages/dashboard.html internal/server/templates/pages/dashboard.html
cp web/css/app.css internal/server/static/css/app.css
cp web/vendor/htmx.min.js internal/server/static/vendor/htmx.min.js
rm -rf internal/templ
```
Update `render.go` to read from embedded `templates` dir (already correct in router embed). Update `templ.FuncMap` import path stays `internal/templ` → rename to `internal/tmplfn` to avoid clash. Simpler: keep funcs in `internal/server/funcs.go` under `server` package. Delete `templ` package; move FuncMap into server package.

Rewrite: delete `internal/templ/`, create `internal/server/funcs.go`:
```go
package server

import (
	"html/template"
	"net/http"
	"strings"

	"github.com/nicksnyder/go-i18n/v2/i18n"
)

func funcMap() template.FuncMap {
	return template.FuncMap{
		"T": func(loc *i18n.Localizer, key string) string {
			msg, err := loc.Localize(&i18n.LocalizeConfig{MessageID: key})
			if err != nil {
				return key
			}
			return msg
		},
		"dir":         DirFromCtx,
		"lang":        LangFromCtx,
		"localize":    LocalizerFromCtx,
		"requestPath": requestPath,
		"path":        pathFor,
	}
}

func requestPath(r *http.Request) string {
	return strings.TrimPrefix(r.URL.Path, "/"+r.PathValue("lang"))
}

func pathFor(r *http.Request, rest string) string {
	return "/" + r.PathValue("lang") + rest
}
```
Update `render.go` to call `funcMap()`.

- [ ] **Step 9: Wire `main.go`**

Replace `cmd/inventory/main.go`:
```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/e-m-zayed/go_inventory_managent/ent"
	"github.com/e-m-zayed/go_inventory_managent/internal/config"
	"github.com/e-m-zayed/go_inventory_managent/internal/db"
	"github.com/e-m-zayed/go_inventory_managent/internal/i18n"
	"github.com/e-m-zayed/go_inventory_managent/internal/server"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	cfg := config.Load()

	client, err := db.Open(ctx, cfg.DBURL)
	if err != nil {
		log.Fatalf("db: %v", err)
	}
	defer func() { _ = client.Close() }()

	bundle := i18n.NewBundle()

	renderer, err := server.NewRenderer()
	if err != nil {
		log.Fatalf("renderer: %v", err)
	}

	srv := server.New(cfg, client, bundle, renderer)
	log.Printf("inventory: listening on %s", cfg.Addr)
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("server: %v", err)
		}
	}()
	<-ctx.Done()
	log.Println("inventory: shutting down")
}
```
Add `"net/http"` import.

- [ ] **Step 10: Build + manual smoke test**

```bash
go mod tidy
mage clean build
./bin/inventory &
PID=$!
sleep 1
curl -sI http://localhost:8080/ | head -1
curl -sI http://localhost:8080/en/dashboard | head -1
curl -sI http://localhost:8080/ar/dashboard | head -1
curl -s http://localhost:8080/en/dashboard | grep -o 'dir="[a-z]*"'
curl -s http://localhost:8080/ar/dashboard | grep -o 'dir="[a-z]*"'
kill $PID
```
Expected: 302 for root, 200 for en/ar, `dir="ltr"` for en, `dir="rtl"` for ar.

- [ ] **Step 11: Commit**

```bash
git add internal/server cmd/inventory go.mod go.sum
git commit -m "server: http server with locale middleware and mirrored layout

routes are path-prefixed by locale. middleware validates the lang
segment, sets dir + localizer in context. layout template sets
dir and lang on the html element so the page mirrors for rtl
locales. static css and htmx are embedded into the binary."
```

#### Task P0.8: Add `Test` mage target + first handler test

**Files:**
- Modify: `magefile.go`
- Create: `internal/server/server_test.go`

- [ ] **Step 1: Add Test target**

Append to `magefile.go`:
```go
// Test runs all go tests with race detector.
func Test() error {
	return sh.RunV("go", "test", "-race", "./...")
}
```

- [ ] **Step 2: Write `internal/server/server_test.go`**

```go
package server

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/e-m-zayed/go_inventory_managent/internal/i18n"
)

func TestDirFor(t *testing.T) {
	cases := map[string]string{"en": "ltr", "ar": "rtl", "he": "rtl", "fr": "ltr"}
	for lang, want := range cases {
		if got := i18n.DirFor(lang); got != want {
			t.Errorf("DirFor(%q) = %q, want %q", lang, got, want)
		}
	}
}

func TestRootRedirect(t *testing.T) {
	cfg := &config.Config{DefaultLocale: "en"}
	srv := &Server{cfg: cfg}
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	rec := httptest.NewRecorder()
	srv.rootRedirect()(rec, req)
	if rec.Code != http.StatusFound {
		t.Fatalf("status = %d, want 302", rec.Code)
	}
	if loc := rec.Header().Get("Location"); loc != "/en/" {
		t.Fatalf("location = %q, want /en/", loc)
	}
}
```
Add `"github.com/e-m-zayed/go_inventory_managent/internal/config"` import.

- [ ] **Step 3: Run tests**

```bash
mage test
```
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add magefile.go internal/server/server_test.go
git commit -m "test: dir map and root redirect tests

adds mage test target running go test -race across the tree.
first tests cover the locale direction map and the root -> default
locale redirect, the two pillars of the localization design."
```

#### Task P0.9: README + phase build verification

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md`**

```markdown
# go_inventory_managent

Inventory and warehouse management system built with Go stdlib, Ent, htmx, and daisyUI.

## Requirements

- Go 1.22+
- Node.js (only for daisyUI plugin resolution; no runtime JS)
- `mage` build tool: `go install github.com/magefile/mage@latest`
- `goi18n`: `go install github.com/nicksnyder/go-i18n/v2/goi18n@latest`

## Getting started

```bash
mage tidy        # go mod tidy
npm install      # daisyui plugin
mage css         # build web/css/app.css
mage ent         # generate ent code
mage build       # compile ./bin/inventory
./bin/inventory  # serve on :8080
```

Open http://localhost:8080/ — redirects to `/en/` (English) or switch to `/ar/` for Arabic (RTL).

## Mage targets

| target | description |
|--------|-------------|
| `build`  | compile binary into ./bin |
| `run`    | build + run |
| `test`   | go test -race ./... |
| `tidy`   | go mod tidy |
| `clean`  | remove ./bin |
| `ent`    | ent code generation |
| `css`    | compile tailwind + daisyui |
| `i18n`   | extract + merge translation files |
```

- [ ] **Step 2: Phase 0 clean build**

```bash
mage clean build
mage test
```
Expected: both exit 0.

- [ ] **Step 3: Commit phase 0 complete**

```bash
git add README.md
git commit -m "docs: readme with setup and mage target reference

phase 0 scaffold complete: go module, ent user schema, sqlite dev
db, htmx, daisyui, i18n bundle with en/ar stubs, path-prefixed
locale routing, mirrored layout, vendored htmx, mage build/test/
ent/css/i18n targets. foundation ready for phase 1 auth."
```

- [ ] **Step 4: Push**

```bash
git push -u origin main
```

**Phase 0 exit criteria met:** `/en/dashboard` renders LTR, `/ar/dashboard` renders RTL mirror, `mage clean build` + `mage test` green, pushed to remote.

---

### Phase 1 — Authentication & RBAC

**Phase goal:** Stdlib session login/logout, User.Locale persisted, Role + Permission ent schemas, protected routes, login UI mirrored. Root redirect reads user locale.

#### Task P1.1: Role + Permission ent schemas

**Files:**
- Create: `ent/schema/role.go`
- Create: `ent/schema/permission.go`
- Modify: `ent/schema/user.go` (add edges to Role)

- [ ] **Step 1: Write `ent/schema/role.go`**

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

type Role struct {
	ent.Schema
}

func (Role) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").Unique().NotEmpty(),
		field.String("description").Optional(),
	}
}

func (Role) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("users", User.Type).Ref("roles"),
		edge.To("permissions", Permission.Type),
	}
}
```

- [ ] **Step 2: Write `ent/schema/permission.go`**

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

type Permission struct {
	ent.Schema
}

func (Permission) Fields() []ent.Field {
	return []ent.Field{
		field.String("code").Unique().NotEmpty(),   // e.g. "stock.read"
		field.String("description").Optional(),
	}
}

func (Permission) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("roles", Role.Type).Ref("permissions"),
	}
}
```

- [ ] **Step 3: Add roles edge to User**

Modify `ent/schema/user.go` edges:
```go
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("roles", Role.Type),
	}
}
```
Add imports `entgo.io/ent/schema/edge`.

- [ ] **Step 4: Generate + build**

```bash
mage ent
go mod tidy
mage clean build
```

- [ ] **Step 5: Commit**

```bash
git add ent/ go.mod go.sum
git commit -m "ent: role and permission schemas, user->roles edge

rbac model: users have many roles, roles have many permissions.
permission codes are dotted scopes like stock.read used by the
authorization middleware added in the next task."
```

#### Task P1.2: Session store + password hashing (stdlib)

**Files:**
- Create: `internal/auth/session.go`
- Create: `internal/auth/password.go`

**Interfaces:**
- Produces: `auth.NewSessionStore(secret)` → `*SessionStore`; `(*SessionStore).Get(r)` → `*Session`; `(*SessionStore).Save(w, s)`; `auth.HashPassword(pw)` → string; `auth.VerifyPassword(pw, hash)` → bool.
- Consumes: `crypto/*` stdlib only.

- [ ] **Step 1: Write `internal/auth/password.go`**

```go
package auth

import (
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"golang.org/x/crypto/pbkdf2"
)

// HashPassword returns a pbkdf2 hash with a random salt, formatted as
// "salt$hash". Uses sha256, 10000 iterations (std crypto, no external dep).
func HashPassword(password string) (string, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	dk := pbkdf2.Key([]byte(password), salt, 10000, 32, sha256.New)
	return hex.EncodeToString(salt) + "$" + hex.EncodeToString(dk), nil
}

// VerifyPassword checks a password against a "salt$hash" stored value.
func VerifyPassword(password, stored string) bool {
	parts := strings.SplitN(stored, "$", 2)
	if len(parts) != 2 {
		return false
	}
	salt, err := hex.DecodeString(parts[0])
	if err != nil {
		return false
	}
	dk := pbkdf2.Key([]byte(password), salt, 10000, 32, sha256.New)
	return hex.EncodeToString(dk) == parts[1]
}
```
`golang.org/x/crypto/pbkdf2` is golang.org/x — semi-stdlib, part of the Go project. Confirm acceptable per AGENTS.md (stdlib minimal). If rejected, replace with stdlib-only HMAC-SHA256 PBKDF2 hand-rolled (≈30 lines). Use `crypto/hmac` + `crypto/sha256`. To be safe, implement PBKDF2 inline:

Replace password.go with stdlib-only PBKDF2:
```go
package auth

import (
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/binary"
	"encoding/hex"
	"strings"
)

func HashPassword(password string) (string, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	dk := pbkdf2HMAC(sha256.New, []byte(password), salt, 10000, 32)
	return hex.EncodeToString(salt) + "$" + hex.EncodeToString(dk), nil
}

func VerifyPassword(password, stored string) bool {
	parts := strings.SplitN(stored, "$", 2)
	if len(parts) != 2 {
		return false
	}
	salt, err := hex.DecodeString(parts[0])
	if err != nil {
		return false
	}
	dk := pbkdf2HMAC(sha256.New, []byte(password), salt, 10000, 32)
	return hmac.Equal(dk, must(hex.DecodeString(parts[1])))
}

// pbkdf2HMAC is a stdlib-only PBKDF2 (RFC 8018) implementation.
func pbkdf2HMAC(newHash func() func() hash.Hash, password, salt []byte, iter, keyLen int) []byte {
	prf := hmac.New(newHash, password)
	hashLen := prf.Size()
	numBlocks := (keyLen + hashLen - 1) / hashLen
	var dk []byte
	for block := 1; block <= numBlocks; block++ {
		prf.Reset()
		prf.Write(salt)
		var buf [4]byte
		binary.BigEndian.PutUint32(buf[:], uint32(block))
		prf.Write(buf[:])
		t := prf.Sum(nil)
		u := make([]byte, len(t))
		copy(u, t)
		for n := 2; n <= iter; n++ {
			prf.Reset()
			prf.Write(u)
			u = prf.Sum(u[:0])
			for i := range t {
				t[i] ^= u[i]
			}
		}
		dk = append(dk, t...)
	}
	return dk[:keyLen]
}
```
Add `"hash"` import. `must` helper removed — use explicit:
```go
want, err := hex.DecodeString(parts[1])
if err != nil { return false }
return hmac.Equal(dk, want)
```

- [ ] **Step 2: Write `internal/auth/session.go`**

Stdlib cookie session: signed HMAC-SHA256 cookie carrying user ID + locale. No external store.

```go
package auth

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
	"net/http"
	"time"
)

type Session struct {
	UserID  int
	Locale  string
	Expires time.Time
}

type SessionStore struct {
	secret []byte
	name   string
	maxAge time.Duration
}

func NewSessionStore(secret string) *SessionStore {
	return &SessionStore{secret: []byte(secret), name: "inv_session", maxAge: 12 * time.Hour}
}

func (s *SessionStore) Get(r *http.Request) (*Session, error) {
	c, err := r.Cookie(s.name)
	if err != nil {
		return nil, err
	}
	raw, err := base64.URLEncoding.DecodeString(c.Value)
	if err != nil {
		return nil, err
	}
	if len(raw) < 32 {
		return nil, errors.New("short session")
	}
	sig := raw[:32]
	payload := raw[32:]
	if !hmac.Equal(sig, s.sign(payload)) {
		return nil, errors.New("bad signature")
	}
	var sess Session
	if err := json.Unmarshal(payload, &sess); err != nil {
		return nil, err
	}
	if time.Now().After(sess.Expires) {
		return nil, errors.New("expired")
	}
	return &sess, nil
}

func (s *SessionStore) Save(w http.ResponseWriter, sess *Session) error {
	payload, err := json.Marshal(sess)
	if err != nil {
		return err
	}
	sig := s.sign(payload)
	value := base64.URLEncoding.EncodeToString(append(sig, payload...))
	http.SetCookie(w, &http.Cookie{
		Name:     s.name,
		Value:    value,
		Path:     "/",
		HttpOnly: true,
		SameSite: http.SameSiteLaxMode,
		MaxAge:   int(s.maxAge.Seconds()),
	})
	return nil
}

func (s *SessionStore) Clear(w http.ResponseWriter) {
	http.SetCookie(w, &http.Cookie{Name: s.name, Value: "", Path: "/", MaxAge: -1})
}

func (s *SessionStore) sign(payload []byte) []byte {
	mac := hmac.New(sha256.New, s.secret)
	mac.Write(payload)
	return mac.Sum(nil)
}
```

- [ ] **Step 3: Add password + session tests**

Create `internal/auth/auth_test.go`:
```go
package auth

import "testing"

func TestPasswordRoundtrip(t *testing.T) {
	hash, err := HashPassword("s3cret")
	if err != nil { t.Fatal(err) }
	if !VerifyPassword("s3cret", hash) { t.Fatal("verify should pass") }
	if VerifyPassword("wrong", hash) { t.Fatal("verify should fail for wrong pw") }
}

func TestSessionRoundtrip(t *testing.T) {
	store := NewSessionStore("test-secret")
	sess := &Session{UserID: 1, Locale: "en", Expires: time.Now().Add(time.Hour)}
	rec := httptest.NewRecorder()
	if err := store.Save(rec, sess); err != nil { t.Fatal(err) }
	req := httptest.NewRequest("GET", "/", nil)
	req.Header.Set("Cookie", rec.Header().Get("Set-Cookie"))
	got, err := store.Get(req)
	if err != nil { t.Fatal(err) }
	if got.UserID != sess.UserID { t.Fatalf("uid = %d", got.UserID) }
}
```
Add `"net/http/httptest"` and `"time"` imports.

- [ ] **Step 4: Run tests + build**

```bash
mage test
mage clean build
```

- [ ] **Step 5: Commit**

```bash
git add internal/auth
git commit -m "auth: stdlib session store and pbkdf2 password hashing

session is a signed hmac-sha256 cookie carrying uid and locale.
password hashing is a hand-rolled pbkdf2 over sha256 so no
golang.org/x/crypto dependency is needed. both have roundtrip tests."
```

#### Task P1.3: Auth middleware + login/logout handlers

**Files:**
- Create: `internal/server/auth.go`
- Modify: `internal/server/server.go` (add sessions)
- Modify: `internal/server/router.go` (login routes, protected wrapper)
- Modify: `internal/server/locale.go` (RootRedirect reads user locale)
- Create: `internal/server/templates/pages/login.html`
- Modify: `internal/server/templates/layout.html` (login/logout nav)

**Interfaces:**
- Produces: `RequireAuth` middleware; `/login` GET/POST; `/logout` POST; root redirect reads saved user locale.

- [ ] **Step 1: Write `internal/server/auth.go`**

```go
package server

import (
	"context"
	"net/http"

	"github.com/e-m-zayed/go_inventory_managent/internal/auth"
)

type ctxUser ctxKey

const ctxUserKey ctxKey = "user"

// RequireAuth redirects unauthenticated requests to the login page.
func (s *Server) RequireAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		sess, err := s.sessions.Get(r)
		if err != nil || sess == nil {
			lang := r.PathValue("lang")
			http.Redirect(w, r, "/"+lang+"/login", http.StatusFound)
			return
		}
		user, err := s.client.User.Get(r.Context(), sess.UserID)
		if err != nil {
			s.sessions.Clear(w)
			http.Redirect(w, r, "/"+r.PathValue("lang")+"/login", http.StatusFound)
			return
		}
		ctx := context.WithValue(r.Context(), ctxUserKey, user)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// UserFromCtx returns the authenticated user or nil.
func UserFromCtx(ctx context.Context) *ent.User {
	if v, ok := ctx.Value(ctxUserKey).(*ent.User); ok {
		return v
	}
	return nil
}

func (s *Server) login(w http.ResponseWriter, r *http.Request) {
	if r.Method == http.MethodGet {
		s.renderPage(w, r, "login.html", nil)
		return
	}
	email := r.FormValue("email")
	password := r.FormValue("password")
	user, err := s.client.User.Query().Where(user.EmailEQ(email)).Only(r.Context())
	if err != nil || !auth.VerifyPassword(password, user.PasswordHash) {
		http.Error(w, "invalid credentials", http.StatusUnauthorized)
		return
	}
	sess := &auth.Session{UserID: user.ID, Locale: user.Locale, Expires: time.Now().Add(12 * time.Hour)}
	if err := s.sessions.Save(w, sess); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	http.Redirect(w, r, "/"+user.Locale+"/dashboard", http.StatusFound)
}

func (s *Server) logout(w http.ResponseWriter, r *http.Request) {
	s.sessions.Clear(w)
	http.Redirect(w, r, "/"+r.PathValue("lang")+"/login", http.StatusFound)
}
```
Add imports for `ent/user`, `time`, `auth`.

- [ ] **Step 2: Update Server struct**

`internal/server/server.go` add `sessions *auth.SessionStore` field; in `New`, init `sessions: auth.NewSessionStore(cfg.SessionSecret)` where `cfg.SessionSecret` is new config field (env `INVENTORY_SESSION_SECRET`, default random 32 bytes via `crypto/rand` at startup if unset — log a warning).

- [ ] **Step 3: Update `RootRedirect` to read user locale**

```go
func (s *Server) rootRedirect() http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if sess, err := s.sessions.Get(r); err == nil && sess != nil {
			http.Redirect(w, r, "/"+sess.Locale+"/dashboard", http.StatusFound)
			return
		}
		http.Redirect(w, r, "/"+s.cfg.DefaultLocale+"/", http.StatusFound)
	}
}
```

- [ ] **Step 4: Add login routes + wrap protected routes**

In `router.go`, in the locale sub-mux:
```go
subMux.HandleFunc("GET /login", s.login)
subMux.HandleFunc("POST /login", s.login)
subMux.HandleFunc("POST /logout", s.logout)
protected := http.NewServeMux()
protected.HandleFunc("GET /", s.dashboard)
protected.HandleFunc("GET /dashboard", s.dashboard)
withLocale := LocaleMiddleware(..., s.RequireAuth(protected))
```
Login routes registered outside `RequireAuth`.

- [ ] **Step 5: Write `login.html` page**

`internal/server/templates/pages/login.html`:
```html
{{define "content"}}
<div class="flex min-h-[60vh] items-center justify-center">
  <form class="card bg-base-100 shadow w-full max-w-sm p-6" method="POST" action="{{path "/login"}}">
    <h1 class="text-xl font-semibold mb-4">{{T (localize .Request) "NavLogin"}}</h1>
    <fieldset class="fieldset">
      <label class="label" for="email">{{T (localize .Request) "FieldEmail"}}</label>
      <input id="email" name="email" type="email" class="input w-full" required>
    </fieldset>
    <fieldset class="fieldset mt-2">
      <label class="label" for="password">{{T (localize .Request) "FieldPassword"}}</label>
      <input id="password" name="password" type="password" class="input w-full" required>
    </fieldset>
    <button class="btn btn-primary w-full mt-4" type="submit">{{T (localize .Request) "ActionLogin"}}</button>
  </form>
</div>
{{end}}
```

- [ ] **Step 6: Add i18n keys for login**

Append to `internal/i18n/locales/active.en.toml`:
```toml
[NavLogin]
other = "Login"
[ActionLogin]
other = "Sign in"
[FieldEmail]
other = "Email"
[FieldPassword]
other = "Password"
[ActionLogout]
other = "Logout"
```
Append to `internal/i18n/locales/active.ar.toml`:
```toml
[NavLogin]
other = "تسجيل الدخول"
[ActionLogin]
other = "دخول"
[FieldEmail]
other = "البريد الإلكتروني"
[FieldPassword]
other = "كلمة المرور"
[ActionLogout]
other = "خروج"
```

- [ ] **Step 7: Update layout nav with login/logout**

Replace the `<nav>` block to show login when unauthenticated, logout + username when authenticated. Add a `user` func to funcMap returning `UserFromCtx(.Request)`.

- [ ] **Step 8: Seed a default admin user**

Create `internal/db/seed.go`:
```go
package db

import (
	"context"

	"github.com/e-m-zayed/go_inventory_managent/ent"
	"github.com/e-m-zayed/go_inventory_managent/internal/auth"
)

// SeedAdmin creates an admin user if none exist. Dev helper only.
func SeedAdmin(ctx context.Context, client *ent.Client) error {
	count, err := client.User.Query().Count(ctx)
	if err != nil { return err }
	if count > 0 { return nil }
	hash, err := auth.HashPassword("admin")
	if err != nil { return err }
	_, err = client.User.Create().
		SetName("Admin").SetEmail("admin@example.com").
		SetPasswordHash(hash).SetLocale("en").
		Save(ctx)
	return err
}
```
Call `db.SeedAdmin(ctx, client)` in `main.go` after open. Log "seeded admin@example.com / admin".

- [ ] **Step 9: Build + test manually**

```bash
mage clean build
./bin/inventory &
# in another shell:
curl -sI http://localhost:8080/en/dashboard   # 302 -> /en/login
curl -s -c /tmp/cj -b /tmp/cj -d "email=admin@example.com&password=admin" -o /dev/null -w "%{http_code}\n" http://localhost:8080/en/login  # 302
curl -s -b /tmp/cj -o /dev/null -w "%{http_code}\n" http://localhost:8080/en/dashboard  # 200
kill %1
```

- [ ] **Step 10: Commit**

```bash
git add internal/server internal/db internal/i18n cmd/inventory
git commit -m "auth: stdlib session login/logout with user locale

protected routes require a valid session cookie. login verifies
pbkdf2 hash and writes a signed cookie carrying uid and the user's
saved locale, which the root redirect then honors. seeded admin
user lets dev start immediately."
```

#### Task P1.4: RBAC middleware

**Files:**
- Modify: `internal/server/auth.go` (add `RequirePermission`)
- Create: `internal/auth/rbac.go` (permission code constants)
- Create: `internal/server/server_test.go` (extend with perm check test)

- [ ] **Step 1: Write `internal/auth/rbac.go`**

```go
package auth

const (
	PermStockRead   = "stock.read"
	PermStockWrite  = "stock.write"
	PermPORead      = "po.read"
	PermPOWrite     = "po.write"
	PermAdmin       = "admin"
)
```

- [ ] **Step 2: Add `RequirePermission` to auth.go**

```go
func (s *Server) RequirePermission(code string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		u := UserFromCtx(r.Context())
		if u == nil { http.Error(w, "unauthorized", http.StatusUnauthorized); return }
		ok, err := s.client.User.QueryRoles(u).QueryPermissions().Where(permission.CodeEQ(code)).Exist(r.Context())
		if err != nil || !ok {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```
Add `permission` ent import.

- [ ] **Step 3: Seed default roles + perms in `seed.go`**

```go
func SeedRBAC(ctx context.Context, client *ent.Client) error {
	perms := []string{auth.PermStockRead, auth.PermStockWrite, auth.PermPORead, auth.PermPOWrite, auth.PermAdmin}
	for _, code := range perms {
		if err := client.Permission.Create().SetCode(code).Exec(ctx); err != nil && !ent.IsConstraintError(err) {
			return err
		}
	}
	adminRole, err := client.Role.Create().SetName("admin").Save(ctx)
	if err != nil && !ent.IsConstraintError(err) { return err }
	if ent.IsConstraintError(err) {
		adminRole, err = client.Role.Query().Where(role.NameEQ("admin")).Only(ctx)
		if err != nil { return err }
	}
	allPerms, err := client.Permission.Query().All(ctx)
	if err != nil { return err }
	_, err = adminRole.Update().AddPermissions(allPerms...).Save(ctx)
	return err
}
```
Call after SeedAdmin; attach admin role to seeded user.

- [ ] **Step 4: Test**

```bash
mage test
mage clean build
```

- [ ] **Step 5: Commit**

```bash
git add internal/auth internal/server internal/db
git commit -m "rbac: permission codes and require-permission middleware

permission codes are dotted scopes in a single const block. the
middleware checks the authenticated user's roles for the required
permission. seeded admin role gets every permission. later phases
gate handlers with RequirePermission."
```

#### Task P1.5: Phase 1 build verification + user preference page

**Files:**
- Create: `internal/server/templates/pages/profile.html`
- Modify: `internal/server/auth.go` (add profile GET/POST to update locale)

- [ ] **Step 1: Add profile handler**

```go
func (s *Server) profile(w http.ResponseWriter, r *http.Request) {
	u := UserFromCtx(r.Context())
	if r.Method == http.MethodPost {
		locale := r.FormValue("locale")
		if !i18n.IsSupported(locale) { http.Error(w, "bad locale", 400); return }
		if err := s.client.User.UpdateOne(u).SetLocale(locale).Exec(r.Context()); err != nil {
			http.Error(w, err.Error(), 500); return
		}
		// refresh session locale
		sess := &auth.Session{UserID: u.ID, Locale: locale, Expires: time.Now().Add(12*time.Hour)}
		_ = s.sessions.Save(w, sess)
		http.Redirect(w, r, "/"+locale+"/profile", http.StatusFound)
		return
	}
	s.renderPage(w, r, "profile.html", map[string]any{"User": u})
}
```

- [ ] **Step 2: Write profile.html** — locale switcher form persisting to DB.

- [ ] **Step 3: Add i18n keys** `NavProfile`, `FieldLocale`, `ActionSave`.

- [ ] **Step 4: Build + test**

```bash
mage clean build
mage test
```

- [ ] **Step 5: Commit + push**

```bash
git add internal
git commit -m "auth: profile page persists locale preference in db

the user's locale lives in the users table. switching it on the
profile page updates the row and refreshes the session so the
next root redirect honors the new preference. phase 1 auth
complete: login, logout, rbac, persisted locale."
git push
```

**Phase 1 exit criteria:** login works, protected routes redirect to login, logout clears session, root redirect honors saved locale, profile page updates locale, `mage clean build` + `mage test` green, pushed.

---

### Phase 2 — Catalog (Product, Category, Parameter, Tag, Attachment)

**Phase goal:** Full CRUD for product catalog with hierarchical categories, custom parameters, tags, attachments. All htmx-driven, mirrored UI. Each entity gets: list, create, edit, delete with permission gates.

Decomposition (each task = one entity end-to-end with tests):
- **P2.1** ProductCategory schema + tree CRUD
- **P2.2** Product schema (FK to category) + CRUD list/detail
- **P2.3** ProductParameter schema + template-per-category + inline edit
- **P2.4** Tag schema (polymorphic via ent `Mixin` or join table) + tag filter
- **P2.5** Attachment schema (file upload to local `uploads/` dir) + attach to product
- **P2.6** Category move (re-parent) htmx endpoint
- **P2.7** Product list search + pagination (htmx)
- **P2.8** i18n keys for all catalog UI
- **P2.9** Phase 2 build verification + push

Each task follows the TDD pattern shown in P0/P1: write schema → generate → handler → template → test → commit. Tasks deliberately small — a single entity CRUD is one task because splitting create/edit/delete into separate tasks creates non-testable intermediate states (you can't test "create" without "list" to show it).

---

### Phase 3 — Stock & Locations (core warehouse)

**Phase goal:** Location tree, StockItem, immutable StockMovement ledger, status machine, FEFO helper, stock adjust/move/count UI.

- **P3.1** Location schema (self-referential tree) + CRUD + tree view
- **P3.2** StockItem schema (FK product + location, lot, serial, qty, expiry, status) + create + detail
- **P3.3** StockMovement schema (immutable: from, to, qty, reason, ref, user, ts) + append-only service
- **P3.4** StockMovement.Create service method (transactional: insert movement + adjust StockItem qty) with unit test
- **P3.5** Stock adjust handler (count correction) — writes movement reason "adjustment"
- **P3.6** Stock move handler (loc→loc) — two movements (debit/credit) in one tx
- **P3.7** CustomState schema (status machine: draft/in/quarantined/etc) + UI to change state with audit
- **P3.8** FEFO helper (sort stock by expiry asc) + use in stock list view
- **P3.9** Stock list with filters (product, location, lot, status) htmx
- **P3.10** i18n keys for stock UI
- **P3.11** Phase 3 build verification + push

---

### Phase 4 — Procurement

- **P4.1** Company schema (supplier/manufacturer/customer roles) + CRUD
- **P4.2** SupplierProductPrice schema (supplier × product × price break)
- **P4.3** PurchaseOrder + POLine schemas + create/list/detail
- **P4.4** PO state machine (draft → submitted → approved → received → closed)
- **P4.5** GoodsReceipt schema + receive against PO (creates StockMovement + StockItem)
- **P4.6** PutawayTask schema + assign received stock to bin location
- **P4.7** Invoice schema + match against PO/receipts
- **P4.8** Reorder report (products below min) + suggested PO
- **P4.9** i18n keys for procurement UI
- **P4.10** Phase 4 build verification + push

---

### Phase 5 — Outbound (Pick, Pack, Ship, Return)

- **P5.1** Requisition + RequisitionLine schemas (internal request) + CRUD + approval flow
- **P5.2** Picklist + PicklistLine schemas + FEFO allocation service (unit tested)
- **P5.3** Picking handler (confirm pick → reduce stock via StockMovement)
- **P5.4** Shipment schema (pack: pallet/box, carrier, tracking) + packing UI
- **P5.5** Shipping handler (mark shipped → audit)
- **P5.6** Return schema (reason, inspection, restock/dispose) + handler
- **P5.7** i18n keys for outbound UI
- **P5.8** Phase 5 build verification + push

---

### Phase 6 — Manufacturing (BOM, Build)

- **P6.1** BOM + BOMLine schemas (multi-level, parent product → component product × qty) + CRUD
- **P6.2** BuildOrder schema + state machine (draft → in_progress → complete → cancelled)
- **P6.3** Build allocate (consume component StockItems via StockMovement, produce assembly StockItem)
- **P6.4** Build complete handler (audit + close)
- **P6.5** BOM import from CSV (stdlib `encoding/csv`)
- **P6.6** i18n keys for manufacturing UI
- **P6.7** Phase 6 build verification + push

---

### Phase 7 — Asset Tracking (Snipe-IT style)

**Phase goal:** Serialized asset tracking with checkout/return, license seats, depreciation schedules.

- **P7.1** Asset schema (serialized Product, asset_tag unique, assigned_to User, status) + CRUD
- **P7.2** Checkout/Return handlers (writes StockMovement + Asset status)
- **P7.3** License schema + LicenseSeat schema + checkout/return seat
- **P7.4** Depreciation schema (method: straight-line, term months) + schedule generation
- **P7.5** Asset QR code generation (stdlib SVG) + label print view
- **P7.6** i18n keys for asset UI
- **P7.7** Phase 7 build verification + push

---

### Phase 8 — Operations & QR (Cold chain, Cycle counts, Transfers, Recalls)

- **P8.1** ColdChain fields on Product + Location (temperature zone) + compliance check service
- **P8.2** CycleCount + CycleCountLine schemas + generate suggested counts (last-count-date based)
- **P8.3** Cycle count entry + variance report
- **P8.4** StockTransfer bulk handler (multi-line bin→bin, facility→facility)
- **P8.5** RecallLot schema + quarantine handler (mark all stock of lot as quarantined)
- **P8.6** QR code generator (stdlib SVG, no external lib) used for products, stock items, locations, assets, bins
- **P8.7** QR scan landing handler (`/{lang}/scan/{code}` → resolves entity, redirects to detail)
- **P8.8** i18n keys for operations UI
- **P8.9** Phase 8 build verification + push

---

### Phase 9 — Reports, Export, Dashboard

- **P9.1** Dashboard widgets (stock value, low stock, expiring, fast movers) — server-rendered
- **P9.2** Consumption report (StockMovement aggregation by product/location/month)
- **P9.3** Stockout report + stock alerts
- **P9.4** CSV export (stdlib `encoding/csv`) for stock, movements, products
- **P9.5** JSON export endpoint (`encoding/json`) for API consumers
- **P9.6** i18n keys for reports UI
- **P9.7** Phase 9 build verification + push

---

### Phase 10 — Migrate to Postgres (final step)

**Phase goal:** Swap Ent dialect from SQLite to Postgres. Versioned migrations added so prod schema is reproducible.

- **P10.1** Add Postgres dialect driver (`entgo.io/ent/dialect/sql`) — already present via ent. Add `lib/pq` only as Ent driver registration (no raw use).
- **P10.2** Switch `INVENTORY_DB` default to `postgresql://user:pass@host/db?sslmode=disable` in config
- **P10.3** Add versioned migrations via ent (`ent migrate --target`)
- **P10.4** Add `mage migrate` target running versioned migrations
- **P10.5** Document prod setup in README
- **P10.6** Final clean build + full test suite + push
- **P10.7** Tag release `v0.1.0`

---

## Self-Review Notes

**Spec coverage:**
- Go stdlib backend ✓ (P0.2 main, all handlers stdlib)
- Ent as ORM, no raw SQL ✓ (P0.3, P0.4 — ent.Open only)
- htmx via go-htmx ✓ (P0.6 htmx vendored, P0.7 middleware)
- daisyUI ✓ (P0.6 css pipeline)
- i18n go-i18n + TOML ✓ (P0.5)
- en + ar start, extensible ✓ (P0.5 registry)
- bi-directional UI, no hardcoded dir ✓ (P0.7 layout dir attr + logical CSS constraint in every template task)
- magefile build/run/test ✓ (P0.2, P0.8)
- clean build after each phase ✓ (every phase ends with build verification task)
- local binary, no docker ✓ (P0.4 main serves directly)
- git discipline Mitchellh style ✓ (every task ends with commit; message style documented in Global Constraints)
- URL-based localization `/{lang}/...` ✓ (P0.7 routing)
- locale/user choice persisted in DB ✓ (P0.3 User.locale, P1.3 root redirect, P1.5 profile page)
- Postgres last step ✓ (Phase 10)
- Asset tracking in scope ✓ (Phase 7)
- Cold chain latest stage ✓ (P8.1)
- QR code ✓ (P8.6)
- Small iterations ✓ (every task is a single entity or single endpoint with its own test+commit cycle)
- InvenTree reference used for structure only ✓ (features synthesized with PartKeepr, Snipe-IT, OpenBoxes)
- Other refs researched ✓ (PartKeepr electronics, Snipe-IT assets/licenses, OpenBoxes WMS/pick/pack/FEFO/cycle counts/recalls/cold chain)

**Placeholder scan:** Tasks P0 and P1 are fully written with code. Tasks P2-P10 are decomposed to the task level (one entity = one task) but code is not pre-written because (a) the plan would exceed usable size and (b) each task's exact code depends on ent-generated API names that only exist after the schema task runs. The decomposition is concrete enough that a fresh agent can execute each task TDD-style. If you want any specific later task fully expanded with code before execution, name it and I'll expand that task in place.

**Type consistency:** `User.Locale` string field established P0.3, read P1.3, written P1.5 — consistent. `Server.sessions` added P1.2, used P1.3/P1.5 — consistent. `StockMovement` introduced P3.3, reused P3.4/P3.5/P3.6/P4.5/P5.3/P6.3 — consistent. Permission codes in P1.4 reused P2+ via `RequirePermission` — consistent.