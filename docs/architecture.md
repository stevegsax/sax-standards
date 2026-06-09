# SAX standard project layout

One installable `src/`-layout package per repo. A pure `core/` at the center; one shell
subpackage per **kind** of side effect; bounded-context **domains** one level down inside each
layer. Imports flow strictly inward, enforced as a CI gate by import-linter.

This is the target shape the `python-quality-checker` skill audits against.

## Two routing rules (where does a file go?)

1. **Does it do I/O?** — yes → a shell subpackage (`db/`, `web/`, `workflows/`, `activities/`,
   `clients/`, …). No → `core/`. There is no `services/`/`utils/` pile: logic has exactly one home.
2. **Is it shipped in the wheel?** — no → `ops/`. Scripts, deploy config, notes, sample data
   never sit at the repo root.

## Tiered: required core + opt-in modules

Every repo has the required skeleton. Optional modules appear **only when needed**, but always
at the **same path** when they do — so a 5-file CLI and a full Temporal+web+DB app are the same
shape minus the parts the small one omits.

- **Required:** `pyproject.toml`, `uv.lock`, `.python-version`, `importlinter.ini`, `Justfile`,
  `CLAUDE.md`, `src/<pkg>/{__init__,settings,ids,errors,cli}.py`, `src/<pkg>/core/<domain>/`,
  `tests/unit/`, `tests/conftest.py`, `docs/`, `ops/`, `.generated/`.
- **Opt-in (fixed names):** `core/contracts/`, `db/`, `workflows/`+`activities/`+`steps/`+
  `worker.py`, `web/`, `clients/`, `observability.py`, `tests/integration/`, `tests/e2e/`.

## The tree

```
myapp/                                  # repo root — ONE installable package (not a workspace)
├── pyproject.toml                      # single [project]; ruff+mypy(strict)+pytest+import-linter; hatchling → src/myapp
├── uv.lock  .python-version            # committed lock; pin 3.14
├── importlinter.ini                    # inward-only import contract, enforced as a CI GATE
├── Justfile                            # just test|lint|typecheck|up
├── CLAUDE.md  README.md                # ONE agent file (no AGENTS/GEMINI duplicates)
│
├── src/myapp/                          # the only thing in the wheel
│   ├── __init__.py  py.typed           # public facade; NO import-time I/O
│   ├── settings.py                     # the ONE pydantic-settings class; only reader of os.environ; fail-fast
│   ├── ids.py  errors.py               # stdlib-only leaves: NewType IDs; AppError hierarchy
│   │
│   ├── core/                           # FUNCTIONAL CORE — pure, sync, deterministic; no I/O/framework/async
│   │   ├── contracts/                  # (Temporal/cross-process) pure SPI — wire.py DTOs, constants.py Finals, ports.py Protocols
│   │   └── <domain>/                   # ≥1 bounded context: domain.py (frozen types) + logic.py (pure; returns X | DomainError)
│   │
│   ├── db/                             # (persistence) SQLAlchemy 2.0 — the ONLY place ORM lives
│   │   ├── engine.py base.py           #   session FACTORIES (no module singletons); one DeclarativeBase
│   │   ├── <domain>.py                 #   <Entity>Row + repo fns; mirrors core/ domains; .to_value() → frozen core type
│   │   └── migrations/                 #   Alembic vendored in-package; version_table=myapp_alembic_version
│   │
│   ├── workflows/  activities/  steps/ # (Temporal) @workflow.defn / @activity.defn shells, one file per domain (mirrored names)
│   │                                   #   __init__ exposes workflows()/activities() lists — side-effect-free discovery
│   ├── worker.py                       # (Temporal) the SINGLE registry; concatenates discovery lists; wires SandboxedRunner
│   │
│   ├── web/                            # (HTTP/UI) FastAPI — the only HTTP layer
│   │   ├── app.py  deps.py             #   create_app()+lifespan; deps = only reader of app.state; no DI container
│   │   ├── <domain>/                   #   routes.py (shell only) + presenters.py (pure) + templates/ COLOCATED
│   │   └── templates/base.html static/ #   shared base layout; vendored htmx/_hyperscript + VENDOR.md
│   │
│   ├── clients/                        # (outbound I/O) one module per external service; each implements a ports Protocol; injected
│   ├── observability.py                # (opt-in) structlog + OTel; NoOp under test
│   └── cli.py                          # Click entrypoint; web via `uvicorn myapp.web.app:app`; worker via `myapp worker`
│
├── tests/                              # mirrors src/myapp
│   ├── conftest.py                     # autouse SAFETY: refuse non-test DATABASE_URL; tmp_path XDG override; Temporal time-skip
│   ├── unit/                           # pure core — parametrize + hypothesis; no I/O, no fixtures
│   ├── integration/                    # real ephemeral Postgres / moto S3 / Temporal test server
│   └── e2e/features/                   # TestClient + CliRunner + BDD at the boundary
│
├── docs/                               # AUTHORITATIVE design only — architecture.md, decisions/ (ADRs), requirements/
├── ops/                                # THE JUNK-DRAWER FIREWALL — everything NOT in the wheel
│   ├── scripts/ (PEP 723)  deploy/ (terraform/systemd/compose)  notes/ (dev-plans, scratch)  data/ (sample/SQL)
└── .generated/                         # gitignored, disposable: Hugo/coverage output — never committed, never authoritative
```

## The import-direction contract

`core/` may not import any shell subpackage, and `core/contracts/` may import nothing impure.
This is the load-bearing invariant; `importlinter.ini` encodes it and `just lint` runs it in CI.
A new domain folder MUST be added to the contract — a meta-test treats a missing entry as a
build break, so the gate can't silently develop holes.

## Layer rules (from the house python-guidelines)

- `core/` is pure, sync, deterministic. Money is `Decimal`; value types are
  `@dataclass(frozen=True, slots=True, kw_only=True)`; expected failures are return values
  (`X | DomainError`), not exceptions.
- `settings.py` is the only reader of `os.environ`; pure code receives scalar values, never the
  whole `Settings`.
- Temporal: no import-time/side-effect registration — `worker.py` wires the lists returned by
  `workflows()` / `activities()`. Workflows import core under
  `with workflow.unsafe.imports_passed_through():`.
- `db/` is the only place ORM lives; sessions are passed as the first parameter, repos return
  frozen core values, never rows/dicts.
- Forbid a `core/shared/` or `core/common/` domain — cross-domain pure helpers live in a named
  domain or are promoted deliberately.

See `docs/contracts-and-deps.md` for the cross-repo / Temporal-contracts distribution rule.
