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

`core/contracts/` is pure pydantic + `Final` + `Protocol`, so the inward-only import rule makes
it automatically Temporal-sandbox-safe (ORM lives only in `db/`, which `core/` may not import).

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
