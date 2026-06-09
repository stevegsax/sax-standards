# sax-standards

The SAX Capital standard Python project layout, and a [copier](https://copier.readthedocs.io/)
template that scaffolds a compliant repo.

- `docs/architecture.md` — the standard: layout, import-direction contract, routing rules.
- `docs/contracts-and-deps.md` — cross-repo dependency + Temporal-contracts distribution.
- `docs/migration.md` — migrating an existing repo onto the standard.
- `template/` — the copier template.

## Scaffold a new project

```sh
uvx copier copy gh:stevegsax/sax-standards my-new-project
# or, from a local clone:
uvx copier copy ./sax-standards my-new-project
```

## Update an existing project to a newer template

```sh
uvx copier update     # run inside a project previously generated from this template
```

Compliance is defined by the import-linter gate (`just lint`) passing, not by the directory
shape looking right. See `docs/architecture.md`.
