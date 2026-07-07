# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository context

This directory (`06-api-rest-orm`) is one exercise in a larger Go web course
(`ejercicios-goweb`). It is a standalone Go module — each numbered exercise
directory in the parent repo has its own independent `go.mod` and is not
meant to import from siblings.

## Commands

```bash
go build ./...        # build
go run main.go         # run the API server (listens on :3000)
go vet ./...           # static checks
```

There are no automated tests in this exercise.

The server requires a running MySQL instance matching the DSN in
[db/database.go](db/database.go) (`root:1234@tcp(localhost:3306)/goweb_db`).
The `User` table is not auto-migrated on startup — `models.MigrarUser()` is
commented out in [main.go](main.go); uncomment it once (or run it manually)
to create the schema before first use.

## Architecture

Minimal REST API over a single `User` resource, built on `gorilla/mux` for
routing and `gorm` (MySQL driver) as the ORM. Three-layer structure:

- **[db/database.go](db/database.go)** — opens the GORM connection once at
  package-init time via a package-level `var Database = func() {...}()`.
  Every other package imports `gorm/db` and calls `db.Database` directly
  (no dependency injection / connection passed through handlers).
- **[models/users.go](models/users.go)** — the `User` struct (GORM model)
  and `Users` slice alias, plus `MigrarUser()` for schema migration.
- **[handlers/handlers.go](handlers/handlers.go)** — one handler function
  per route, following the standard `func(http.ResponseWriter, *http.Request)`
  signature. `getUserById` is a shared helper that reads the `{id}` mux var
  and does a `db.Database.First(...)` lookup; all single-record handlers
  (Get/Update/Delete) call through it.
- **[handlers/response.go](handlers/response.go)** — `sendData` /
  `sendError` are the only response helpers; all handlers funnel their
  output through these instead of writing to `http.ResponseWriter` directly.
- **[main.go](main.go)** — wires routes to handlers. Note that `POST
  /api/user/` and `GET /api/user/` share the same path (disambiguated by
  HTTP method via `.Methods(...)`), while single-record routes use
  `/api/user/{id:[0-9]+}`.

### Request flow

`main.go` (route) → `handlers.*` (decode/validate request, call `getUserById`
when an `{id}` is present) → `db.Database` (GORM call against MySQL) →
`handlers.sendData`/`sendError` (JSON response).
