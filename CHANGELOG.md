# Changelog

## Unreleased (0.2)

Each item was reproduced by running the code before it was fixed.

- Idempotent requests whose process died no longer stay `409 in_progress` forever: after
  `limits.idem_lease_s` they are reconciled from the ledger (never re-executed) and answered with the
  recorded outcome or `409 abandoned` (`executed: no|unknown`); a `tool.abandoned` event is recorded.
- `verify --chain-only` states that it checks structure only; `--anchors FILE` checks the ledger against
  anchors held outside the data dir. `ATL_ANCHOR_FILE` moves the anchor file; the webhook result is
  recorded as an `anchor.published` event.
- `atl erase` / `POST /v1/admin/erase` delete one evidence blob and record an `evidence.erased`
  tombstone; verification accepts tombstoned blobs.
- Policy: per-param `enum`, `pattern`, `min`/`max`; per-tool `rate_per_minute`. Invalid constraints
  refuse to start.
- Admin ledger reads are recorded as `admin.read` events.
- Append cost no longer grows with ledger size (partial index for the key-rotation lookup); `verify` and
  `erase` stream the ledger. `atl anchor --all`, `atl serve --anchor-every` / `ATL_ANCHOR_EVERY_S` narrow the
  window in which tail truncation is undetectable. Opt-in `limits.per_agent_sandbox`. Dot-only ids rejected.

## v0.1.0 — 2026-10-05

First standalone release. Split out of the `phantom-runtime` monorepo into
its own repository with full commit history preserved.

### Core
- Identity → capability policy → authorization (+ separate approver,
  single-use, TTL-bound) → sandboxed execution → per-tenant hash-chained,
  HMAC'd ledger → evidence → external anchor → offline verifier.
- Sandboxed execution: subprocess driver (rlimited, unprivileged) and a
  Docker driver (`--network none --read-only --cap-drop ALL
  --no-new-privileges --pids-limit 64 --memory 256m --cpus 1
  --user 65534:65534`, pinned-digest image, fail-closed, no fallback).
- `KeyProvider` abstraction: file, env, aws-kms, and vault backends; no
  plaintext key on disk outside file mode; versioned key rotation that
  keeps old ledger entries verifiable.

### Verified
- `./verify.sh`: 27 tests + demo, green.
- Agent test harness (`harness.run_agents`): 744 scripted requests across 24
  agents, 100% matched expectation after the availability fix in
  `reports/REPORT.md`; both subprocess and Docker executors.
- Live adversarial trial with real Claude subagents (not scripted): zero
  successful privilege escalations, zero unapproved destructive actions
  across an embedded prompt-injection attack, ledger independently verified
  clean (`reports/REPORT.md` §6).
- Security scan: bandit 0 High/Medium after fixes; no tracked secrets;
  `pip-audit` has nothing to audit (stdlib only).

### Known limits
- SQLite single writer; DB-backed single-node rate limits; Python policy
  evaluator (OPA not used).
- Ledger bake-off vs. POM/SAL ledgers not run — adapters for those ledgers
  were never supplied.
- Vault and AWS KMS key providers are tested against mocks only.

### Packaging
- CI: `.github/workflows/ci.yml` (verify + Docker-executor tests across
  Python 3.9/3.12, bandit, CycloneDX SBOM for both source and container
  image).
