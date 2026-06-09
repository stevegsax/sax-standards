# ops/ — the firewall

Everything here is **not** shipped in the wheel and is **never imported by `src/`**.

- `scripts/` — one-off + operational scripts (PEP 723 inline deps, run via `uv run`).
- `deploy/` — terraform, systemd, compose, nginx, certs.
- `notes/` — dev plans, handoffs, scratch.
- `data/` — sample inputs, analyst SQL.

Routing rule: if a file isn't shipped in the wheel, it lives here, not at the repo root.
