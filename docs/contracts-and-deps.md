# Cross-repo dependencies & Temporal contracts

## The contracts-tier rule

Temporal cross-service contracts (workflow/activity wire DTOs, `Final` queue/namespace/signal
names, `Protocol` ports) have a different right home at each scale. The standard pre-builds the
seam (`core/contracts/`) so moving up a tier is a `git mv`, not a restructuring.

| Scale | Contracts live as | Why |
|---|---|---|
| **Same repo** (default) | `src/<pkg>/core/contracts/` subpackage | A second importer can already `import <pkg>`; a second wheel buys nothing |
| **2nd deployable, same team/cadence** | a `uv` workspace member `packages/<pkg>-contracts/` | one `uv.lock`, atomic cross-package refactors |
| **Independently-operated service, separate repo** | a **published** `<pkg>-contracts` distribution | hard boundary; communicate only via wire models + string constants |

`core/contracts/` is pure pydantic + `Final` + `Protocol`; the inward-only import rule plus the
`contracts-sandbox-safe` contract (no `asyncio`/`logging` in `core/contracts/`) keeps it
Temporal-sandbox-safe (ORM lives only in `db/`, which `core/` may not import).

Extract a wheel **only when a second _repo_ must import the contracts — not one day before.**

## Distribution on GitHub (no internal package index)

SAX has no private PyPI, and GitHub has no native Python registry. For the `forge` ecosystem
(`forge`, `forge-contracts`, `forge-ocr`, `pbook`, `sax-llm` — separate repos sharing contracts),
distribute by **git tag**. The git tag *is* the published version.

```toml
# consumer pyproject.toml — declare the dep normally, redirect the source
[project]
dependencies = ["forge-contracts", "sax-llm"]

[tool.uv.sources]
forge-contracts = { git = "https://github.com/stevegsax/forge-contracts.git", tag = "v0.3.0" }
sax-llm         = { git = "https://github.com/stevegsax/sax-llm.git", tag = "v1.2.0" }
```

Generate it canonically instead of hand-editing:

```sh
uv add "forge-contracts @ git+https://github.com/stevegsax/forge-contracts.git@v0.3.0"
```

`uv lock` pins the exact commit behind the tag into `uv.lock` → reproducible everywhere.

**Workflow:** change contracts → tag `vX.Y.Z` → bump the tag in each consumer's
`[tool.uv.sources]` → `uv lock`. With a handful of consumers and a rarely-changing package, the
explicit bump is a reviewable "we adopted the new contract."

**Do not** keep editable path deps (`{ path = "../forge-contracts" }`): they don't resolve in
CI and carry no version.

### CI auth (the only setup cost)

Private repos need read access to clone the dependency. On GitHub Actions:

```sh
git config --global url."https://x-access-token:${INTERNAL_REPO_TOKEN}@github.com/".insteadOf "https://github.com/"
```

(A fine-grained PAT or GitHub App token with read on the internal repos.) `uv` respects git's
credential config, so `uv sync` then works.

### When to graduate to a static index

Move to a serverless PEP 503 index (GitHub Pages / S3 + `uv publish`) only when consumers
multiply enough that hand-bumping tags hurts, or you want consumers to not need git access to the
source repo. Neither is true today.

## Migrating an existing repo onto git-tag deps

Converting from editable path deps to git-tag deps has three rules that, skipped, cost real
round-trips:

1. **Migrate leaves-first.** Order by the internal dependency graph: a package with no internal
   deps first, then packages that depend on it, then the root app — e.g. `contracts`/`sax-llm` →
   `pbook` (uses `sax-llm`) → `forge` (uses all three). Migrating the root first pins it to
   dependencies that are still path-sourced and possibly stale.
2. **An internal package must source its *own* internal deps via git tags too — and
   consistently.** If `forge` depends on `pbook` and `sax-llm`, and `pbook` also depends on
   `sax-llm`, the `sax-llm` spec must be identical everywhere. `{ rev = "v0.1.0" }` and
   `{ tag = "v0.1.0" }` resolve to the same commit but uv treats them as **conflicting URLs**
   (the error shows two identical-looking URLs). Fix by making the specs identical (`tag = …`
   everywhere) — not with `override-dependencies`, which only masks the inconsistency.
3. **Tag at current HEAD; never pin a stale tag.** A consumer pinned to a tag that lags the
   package's code is silently **downgraded** — you lose commits and the dependencies the newer
   code added (a stale `pbook` tag dropped `pgvector`/`psycopg` this way). Before pinning, confirm
   the tag == the code you've been developing against; if not, cut a fresh tag.

### Worked sequence (per dependency, leaves-first)

1. In the dep repo: swap its own `[tool.uv.sources]` path deps → git tags (rule 2), bump
   `[project].version`, `uv lock`, commit.
2. Tag it at that commit (`vX.Y.Z`) and push the branch **and** the tag
   (`git push && git push origin vX.Y.Z` — a plain `git push` does **not** push tags).
3. In the consumer: point `[tool.uv.sources]` at the new tag, `uv lock`, `uv sync`, and verify
   imports + that no needed transitive deps vanished.

Verify after every `uv lock` with `uv sync` and a smoke import — a resolution that *locks*
cleanly can still have dropped a transitive dependency your code needs at runtime.
