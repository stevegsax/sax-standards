# Migrating an existing repo onto the standard

## In-scope fleet

| Repo | Components | Effort | Notes |
|---|---|---|---|
| `forge-contracts` | pure contracts pkg | trivial | **Do first** — becomes the git-tag-published keystone |
| `sax-llm` | provider lib | LOW | git-tag-dep-ify + light FC/IS tidy |
| `forge-ocr` | Temporal + DB | LOW | already close |
| `pbook` | Temporal + DB | LOW | inject the provider instead of the module global |
| `forge` | Temporal + CLI | MODERATE | split the `workflows.py`/`cli.py`/`models.py` god-files |
| `annotation_manager` | web + DB + **Celery→Temporal** | MODERATE–HIGH | standard migration **plus** queue conversion |
| `epub-reader` | web + DB | LOW / optional | grandfather as the `web/`+`db/` reference impl |

**Order:** `forge-contracts` → the three LOW forge-ecosystem repos → `forge` → `annotation_manager`
→ `epub-reader` opportunistically.

The standard is **Temporal-only** — there is no `tasks/`/Celery module. `annotation_manager`'s
Celery → Temporal conversion (tasks → `@activity.defn`; chains/groups/chords → workflow
orchestration; beat schedules → Temporal Schedules) is the one genuinely non-mechanical step, so
it is sequenced last.

## Phases (each independently shippable, CI-checkable)

0. **Cheap mechanical wins, every repo:** add the single `Settings` class and replace scattered
   `os.environ`; pin 3.14 + `.python-version`; drop `from __future__ import annotations`; collapse
   `AGENTS.md`/`GEMINI.md` into one `CLAUDE.md`; wire `mypy --strict` + `ruff` CI gates; commit
   `uv.lock`; delete committed `dist/`/generated docs; pick one migration system.
1. **Firewall + gate:** create `ops/`, move root clutter in; gitignore `.generated/`; add
   `importlinter.ini` and turn it on in CI — its initial failures are the Phase 2–3 work-list.
2. **Extract the pure core** (expensive, judgment-heavy): pull the functions already marked pure
   in docstrings into `core/<domain>/`; split god-modules along the FC/IS line. Do it behind
   characterization tests; keep tests green per domain.
3. **Regroup the shell by kind:** move flat modules into `db/`, `workflows/`, `activities/`,
   `clients/`, `web/`; mirror `activities/` filenames to `workflows/`; convert import-time Temporal
   registration to side-effect-free `workflows()`/`activities()` discovery.
4. **Contracts seam:** fold in-repo contracts into `core/contracts/` (Tier 1); for the `forge`
   ecosystem, publish `forge-contracts` by git tag (see `contracts-and-deps.md`).
5. **Tests:** split flat `tests/` into `unit/integration/e2e`; add the `conftest` safety fixtures.

## "Done" is the gate passing, not the shape looking right

The realistic failure mode is a half-migrated fleet where Phase 0–1 lands everywhere and Phase
2–3 stalls, so reviewers can no longer trust the structure. A repo is either **`just lint`
(import-linter) green** or **explicitly grandfathered** (`epub-reader`) — no long-lived in-between.
