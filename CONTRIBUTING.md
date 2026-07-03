# Contributing to Colony

Colony is **curated**. An entry is only as valuable as it is trustworthy — a pile of unverified examples is worse than nothing, because a confidently-broken "canonical" flow teaches the wrong thing. So every entry clears the same bar.

## The curation bar

An entry (template or pattern) is accepted only if **all** hold:

1. **Verifies.** `swarm verify` passes against the current `platform-spec` (the version pinned in the entry's `platform_spec` field).
2. **Proven.** It ships at least one scenario test under `scenarios/` that exercises its behavior (deterministic; no live LLM spend). CI runs these. *(Until the scenario runner lands, `swarm verify` + a documented `swarm event publish` walkthrough is the interim proof.)*
3. **Documented.** A `README.md` that states: what it does, its inputs/outputs, required backends/tools/credentials, and how to run or compose it.
4. **Manifest.** A complete `package.yaml` — name, version, description, `platform_version`, and the `colony:` curation block (`maturity`, `category`). See [docs/authoring-guide.md](./docs/authoring-guide.md). No separate metadata file: Colony derives the catalog from `package.yaml` + the contracts.
5. **Interface (patterns only).** Patterns declare their pins (inputs/outputs) in `schema.yaml` so they're swappable/composable (see #1468).
6. **Idiomatic.** It follows current platform patterns — it's teaching material, so it should model the *right* way, not a workaround.

## Anti-rot policy

The platform dialect evolves. To keep Colony honest:

- CI runs `swarm verify` + scenarios on **every** entry, on every PR **and** on a schedule.
- A platform change that breaks an entry turns CI red. The fix is to **update or quarantine** the entry — never to weaken the check.
- Each entry pins the `platform_spec` version it targets; bumping it is a deliberate, reviewed change.

## Adding an entry

1. Create `templates/<name>/` or `patterns/<name>/` with the contract bundle, `template.yaml`/`pattern.yaml`, `README.md`, and `scenarios/`.
2. Run `swarm verify` locally against current `platform-spec`.
3. Regenerate the catalog: `scripts/build-index.sh`.
4. Open a PR. A CODEOWNER reviews for the bar above.

Maturity levels (`maturity:` field): `flagship` (showcase, exhaustively documented) · `reference` (solid, idiomatic) · `pattern` (a building block).
