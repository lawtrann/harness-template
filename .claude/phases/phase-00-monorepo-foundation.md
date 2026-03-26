# Phase 0: Monorepo Foundation

## Prerequisites
> No prior phases — this is the first phase.
> Starting from a bare repo with harness tooling in `.claude/`.

## Decisions
- **Go 1.24** for all Go modules, **Python 3.12** for Alembic migrations
- **Pre-install all pinned deps** in `pkg/goshared` so downstream modules inherit them
- **Homebrew-based `scripts/setup.sh`** for developer tooling (Buf, sqlc, SOPS, age, air)
- **Single DB, schema-per-service** — schemas are `casdoor`, `mastermgmt`, `eureka` in DB `platform`
- **Module layout** — `pkg/goshared` is a shared library; `app/mastermgmt` and `app/eureka` import it via `replace` directive
> See .claude/specs/00-architecture-decisions.md for rationale behind these decisions

## Tasks in this phase
| ID | Description | Size | File |
|----|-------------|------|------|
| 0.1 | Root scaffold — .gitignore, README.md, root Makefile | small | Makefile |
| 0.2 | pkg/goshared Go 1.24 module with all pinned deps | medium | pkg/goshared/go.mod |
| 0.3 | app/mastermgmt Go module — imports goshared, scaffold cmd + internal | medium | app/mastermgmt/go.mod |
| 0.4 | app/eureka Go module — imports goshared, scaffold cmd + internal | medium | app/eureka/go.mod |
| 0.5 | Alembic migration setup for both apps (Python 3.12) | medium | app/mastermgmt/migration/alembic.ini |
| 0.6 | Proto dirs with buf.yaml + buf.gen.yaml | small | app/mastermgmt/proto/buf.yaml |
| 0.7 | Deployment tree with local env config placeholders | small | deployment/mastermgmt/local/config.yaml |
| 0.8 | scripts/setup.sh — Homebrew installer for Buf, sqlc, SOPS, age, air | medium | scripts/setup.sh |
| 0.9 | SOPS setup — age key, .sops.yaml, encrypt secrets | medium | deployment/mastermgmt/local/.sops.yaml |
| 0.10 | app/web placeholder | small | app/web/README.md |

## Key patterns

### Go module with replace directive (app → goshared)
```go
// app/mastermgmt/go.mod
module github.com/<org>/platform/app/mastermgmt

go 1.24

require (
    github.com/<org>/platform/pkg/goshared v0.0.0
)

replace github.com/<org>/platform/pkg/goshared => ../../pkg/goshared
```

### Pinned deps for goshared (from architecture doc)
```
connectrpc.com/connect v1.19.1
connectrpc.com/validate v0.3.0
github.com/jackc/pgx/v5 v5.8.0
go.uber.org/fx v1.23.0
github.com/spf13/viper v1.21.0
github.com/lestrrat-go/jwx/v3 (latest v3)
github.com/minio/minio-go/v7 v7.0.83
github.com/twmb/franz-go v1.18.0
github.com/stretchr/testify v1.10.0
google.golang.org/protobuf v1.36.5
```

### Alembic init structure
```
app/mastermgmt/migration/
├── alembic.ini
├── pyproject.toml
└── alembic/
    ├── env.py
    ├── script.py.mako
    └── versions/
```

### SOPS + age encryption
```yaml
# deployment/mastermgmt/local/.sops.yaml
creation_rules:
  - path_regex: secrets\.yaml$
    age: <age-public-key>
```

## Verification
- `cd pkg/goshared && go build ./...` — shared module compiles
- `cd app/mastermgmt && go build ./...` — mastermgmt compiles with goshared
- `cd app/eureka && go build ./...` — eureka compiles with goshared
- `cd app/mastermgmt/migration && alembic check` — Alembic configured
- `bash scripts/setup.sh --dry-run` — setup script is valid
- `sops -d deployment/mastermgmt/local/secrets.yaml > /dev/null` — SOPS decryption works

## Dependencies
- Requires: None (first phase)
- Blocks: Phase 1 (Casdoor setup needs docker-compose + deployment config), Phase 2 (goshared internals need the module from 0.2)
