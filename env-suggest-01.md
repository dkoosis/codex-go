0) Project goals Codex cares about

One-command bootstrap (no human guesswork).

Deterministic, offline-friendly tests (no network unless explicitly mocked).

Clear entry points (Make/Just/Mage tasks with self-describing names).

Pinned tool versions (linters/formatters) so results don’t drift.

Small, realistic fixtures (<500 rows aligns with your usual datasets).

Readable logs (structured) and predictable errors (consistent wrapping).

1) Repo layout (simple, boring, obvious)
.
├─ cmd/
│  └─ lynx/                 # main package(s)
│     └─ main.go
├─ internal/                # app code (not importable by others)
│  ├─ crawl/
│  ├─ review/
│  └─ platform/
│     ├─ log/               # slog setup
│     └─ xerr/              # error helpers (cockroachdb/errors)
├─ pkg/                     # optional: stable public packages
├─ testdata/                # fixtures used in tests (small & deterministic)
├─ scripts/                 # shell helpers called by Make/Just
│  ├─ bootstrap.sh
│  └─ ensure_tools.sh
├─ .editorconfig
├─ .gitattributes
├─ .gitignore
├─ go.mod / go.sum
├─ Makefile  (or Justfile / Magefile.go)
├─ .golangci.yml
├─ .env.example             # safe defaults; no secrets
├─ README.md                # Quickstart at top
└─ CONTRIBUTING.md          # 1-page “how to run everything”

2) One-command bootstrap

Make one obvious entry point. Pick one of these and commit to it:

Option A: make

Makefile

SHELL := /bin/bash
.DEFAULT_GOAL := help

GO_VERSION := 1.23
TOOLS_BIN := ./bin

help: ## List targets
	@awk 'BEGIN {FS":.*##"} /^[a-zA-Z0-9_%-]+:.*?##/ {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)

bootstrap: ## Install tools & pre-commit hooks
	@scripts/bootstrap.sh

check: lint test vet vuln ## All code checks

lint: ## Run linters
	@$(TOOLS_BIN)/golangci-lint run

test: ## Unit tests with coverage
	@go test ./... -count=1 -race -coverprofile=coverage.out
	@go tool cover -func=coverage.out | tail -n 1

test_html: ## Open coverage HTML
	@go tool cover -html=coverage.out -o coverage.html

vet: ## go vet
	@go vet ./...

fmt: ## gofumpt & goimports
	@$(TOOLS_BIN)/gofumpt -l -w .
	@$(TOOLS_BIN)/goimports -w .

vuln: ## govulncheck
	@$(TOOLS_BIN)/govulncheck ./...

tidy: ## go mod tidy
	@go mod tidy


scripts/bootstrap.sh

#!/usr/bin/env bash
set -euo pipefail

# 1) Verify Go
REQ_GO="${REQ_GO:-1.23}"
INSTALLED_GO="$(go version | awk '{print $3}' | sed 's/go//')"
echo "Go version: $INSTALLED_GO (required: $REQ_GO)"

# 2) Create local bin
mkdir -p bin

# 3) Install pinned tool versions
./scripts/ensure_tools.sh

# 4) Pre-commit hooks (optional)
if command -v pre-commit >/dev/null 2>&1; then
  pre-commit install
fi

echo "Bootstrap complete."


scripts/ensure_tools.sh

#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."

BIN=./bin

# Pin tool versions so Codex gets predictable results
GOLANGCI=v1.59.1
GOFUMPT=v0.6.0
GOIMPORTS=v0.20.0
GOVULN=v1.1.3

GOBIN="$BIN" go install github.com/golangci/golangci-lint/cmd/golangci-lint@${GOLANGCI}
GOBIN="$BIN" go install mvdan.cc/gofumpt@${GOFUMPT}
GOBIN="$BIN" go install golang.org/x/tools/cmd/goimports@${GOIMPORTS}
GOBIN="$BIN" go install golang.org/x/vuln/cmd/govulncheck@${GOVULN}

echo "Tools installed in $BIN:"
ls -1 "$BIN"


With this, make bootstrap, then make check is all Codex needs.

(If you prefer Just:)

# Justfile
set shell := ["bash", "-c"]
default: help
help:    @just --list

bootstrap:  bash scripts/bootstrap.sh
check:      lint test vet vuln
lint:       ./bin/golangci-lint run
test:       go test ./... -count=1 -race -coverprofile=coverage.out
vet:        go vet ./...
vuln:       ./bin/govulncheck ./...
fmt:        ./bin/gofumpt -l -w . && ./bin/goimports -w .

3) Pin Go toolchain & editor behavior

go.mod

module github.com/you/lynx

go 1.23

toolchain go1.23.0


.editorconfig

root = true
[*]
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8
indent_style = tab


.gitattributes

* text=auto eol=lf

4) Linters & formatters (fast, opinionated)

.golangci.yml

run:
  timeout: 3m
  tests: true
  allow-parallel-runners: true

linters-settings:
  gofumpt:
    extra-rules: true
  revive:
    confidence: 0.8

linters:
  enable:
    - govet
    - staticcheck
    - gofumpt
    - goimports
    - revive
    - errcheck
    - ineffassign
    - typecheck
    - gosimple
    - unused
    - gocyclo

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gocyclo

5) Logging & errors (so Codex’s debug output is useful)

Structured logging via log/slog:

package log

import (
  "log/slog"
  "os"
)

func New() *slog.Logger {
  h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
  return slog.New(h)
}


Error wrapping via github.com/cockroachdb/errors:

package xerr

import "github.com/cockroachdb/errors"

func Wrap(err error, msg string) error {
  if err == nil { return nil }
  return errors.Wrap(err, msg)
}
func New(msg string) error { return errors.New(msg) }


Use these consistently so stack context shows up in CI logs.

6) Tests Codex can actually run

No network by default: guard with -short or env flag.

Fixtures: keep in testdata/ (tiny, but real).

Golden files for snapshot-like assertions:

func TestRender(t *testing.T) {
  got := renderSomething()
  want, _ := os.ReadFile("testdata/render.golden")
  if string(got) != string(want) { t.Fatal("mismatch") }
}


Race detector (-race) in default test target.

7) CI sample (GitHub Actions)
name: ci
on: [push, pull_request]
jobs:
  go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23.x' }
      - run: make bootstrap
      - run: make check

8) Dev ergonomics (so an agent doesn’t get stuck)

README.md (first 10 lines are gold):

## Quickstart
1) `make bootstrap`
2) `make check`
3) `make run` (if you add it) or `go run ./cmd/lynx`

## Common tasks
- Format: `make fmt`
- Lint:   `make lint`
- Test:   `make test`


CONTRIBUTING.md: a single page with expectations: commit style, how to write tests, error/logging conventions.

.env.example: document all env vars; code tolerates absence (defaults).

9) Optional: Magefile instead of Make

If you want everything in Go (nice for cross-platform):

// +build mage
package main

import (
  "fmt"
  "os"
  "os/exec"
)

func Bootstrap() error {
  cmds := [][]string{
    {"bash", "scripts/ensure_tools.sh"},
  }
  for _, c := range cmds {
    cmd := exec.Command(c[0], c[1:]...)
    cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
    if err := cmd.Run(); err != nil { return err }
  }
  fmt.Println("Bootstrap complete")
  return nil
}


Then: go run mage.go Bootstrap (or use mage binary).

10) Guardrails that help Codex succeed

Fail fast: return non-zero on any script step; no silent || true.

Pin versions: linter/formatter/vuln-check tools as shown.

Deterministic outputs: avoid time.Now() in tests—inject a clock.

No interactive prompts by default; add --yes / --no-color.

Small PRs: CI runs in <2–3 minutes.

11) Nice-to-have (if you touch web bits)

If you later add a tiny React UI: keep it in /ui with its own package.json and a separate make ui-bootstrap target so Go checks aren’t blocked by Node issues.

Don’t require Playwright/Chromium for unit tests; hide behind an integration flag.

12) Starter go.mod
module github.com/you/lynx

go 1.23
toolchain go1.23.0

require (
	github.com/cockroachdb/errors v1.11.1
)

TL;DR: What Codex will run
git clone <repo>
cd repo
make bootstrap
make check


That sequence should: install pinned tools → lint → vet → test (race, coverage) → vuln scan, with zero surprises.

If you want, I can generate the Makefile, scripts, .golangci.yml, and scaffolding files for your current repo and name things “lynx”.