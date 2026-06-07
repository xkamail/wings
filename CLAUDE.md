# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Pterodactyl Wings — the server control plane daemon for the Pterodactyl game server panel. It exposes an HTTP API (plus websockets and a built-in SFTP server) for managing game server lifecycles inside Docker containers. Linux-only at runtime (builds target GOOS=linux); most code still compiles and tests on macOS.

## Commands

```bash
# Build (linux amd64 + arm64, output in build/)
make build

# Local debug build + run (requires root and a config.yml)
make debug

# Remote-debuggable build via delve on :2345
make rmdebug

# Tests — CI runs both of these
go test $(go list ./...)
CGO_ENABLED=1 go test -race $(go list ./...)

# Single package / single test
go test ./server/...
go test ./internal/ufs -run TestWalkDir
```

CI (`.github/workflows/push.yaml`) builds with `CGO_ENABLED=0` against Go 1.24.x and 1.25.x. The default/PR branch is `develop`.

## Architecture

The daemon boots in `cmd/root.go`: loads `config.yml` (`config/` — global singleton accessed via `config.Get()`), connects to the Panel, fetches all servers, initializes them, then starts the HTTP API, SFTP server, and cron jobs.

Core layers, from outside in:

- **`remote/`** — HTTP client for the Panel API (`remote.Client` interface). Wings is a Panel satellite: server definitions, install scripts, backup status, activity logs all sync through this client. Servers are *configured* by the Panel; Wings holds a local cached state (`server/configuration.go`, persisted states in `states.json`).
- **`router/`** — gin-based HTTP API. Routes in `router.go`; auth via token middleware (`router/middleware`, `router/tokens` for signed JWT one-time tokens used for downloads/websockets). `router/websocket` streams console output/stats to clients and handles power commands.
- **`server/`** — the heart. `server.Server` couples a Panel configuration, an `environment.ProcessEnvironment`, a `filesystem.Filesystem`, and an `events.Bus`. `manager.go` holds the in-memory collection of all servers. Key flows: `power.go` (start/stop/kill with a `system.Locker` to serialize power actions), `install.go` (runs egg install scripts in a throwaway container), `crash.go` (crash detection/auto-restart), `listeners.go` (reacts to environment events, forwards state to the Panel), `server/backup/` (local & S3 backups), `server/transfer/` (server transfers between nodes).
- **`environment/`** — `ProcessEnvironment` interface (state, power, command exec, resource stats); `environment/docker/` is the only implementation, managing the container lifecycle via the Docker SDK.
- **`server/filesystem/`** — sandboxed file operations rooted at the server data dir, enforcing disk-limit quotas. Built on **`internal/ufs/`**, a path-traversal-safe filesystem layer using `openat2`-style resolution; all server file access must go through it (never `os.*` against server data paths).
- **`parser/`** — egg-defined config-file rewriting (yaml/json/ini/xml/properties/regex `file` parsers) applied to server files before boot.
- **`sftp/`** — SFTP server authenticating against the Panel API; writes activity events.
- **`events/`, `system/`** — pub/sub bus used by server+environment; `system/` has concurrency primitives (`Locker`, `SinkPool` for console multiplexing, rate limiting).
- **`internal/cron/`** — periodic shipping of activity logs and SFTP events to the Panel, backed by **`internal/database/`** (local SQLite via GORM, models in `internal/models/`).

Event flow: docker environment emits state/stats/console onto its `events.Bus` → `server/listeners.go` translates to server events, crash handling, and Panel sync → websocket/router subscribers stream to clients.

## Conventions

- Errors use `emperror.dev/errors` (wrapping with stack traces); logging uses `github.com/apex/log` with structured fields.
- Tests use `github.com/franela/goblin` in existing suites; match the style of the package you're editing.
- Imports are grouped stdlib / external / `github.com/pterodactyl/wings/...`.
