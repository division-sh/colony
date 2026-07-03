# Colony — design & review

One place to review the whole idea: what Colony is, how it's structured, the metadata model, and the platform work it depends on.

---

## 1. Vision

**Colony is a curated, CI-verified library of Swarm flows** — reusable, working starting points to fork, and composable building blocks to wire in. It is the **trusted, first-party precursor to the Swarm flow marketplace**.

Three jobs at once:
1. **Kills the authoring learning curve** — "fork this working thing" beats "learn the dialect cold."
2. **Is the corpus for the authoring agent** — a flow-authoring agent needs verified examples to pattern-match against.
3. **De-risks the marketplace** — you can't open a marketplace to third parties until you can curate reusable flows internally. Colony forces portability/composition (#1450, #1468) to be real.

**Curated library ≠ marketplace.** Colony is first-party + trusted (bar = *reusable + verified*). The marketplace adds a capability-audit/trust model, sandboxing (why `logic_node`→WASM matters), versioning, and discovery. Build the curated version first; let it tell us what the trust layer needs.

**Colony ≠ conformance.** Its `scenarios/` are *smoke tests that the example still works* — the platform's semantic conformance stays in `swarm` (cataloge2e). Colony is realistic authoring examples, not a correctness oracle.

## 2. Two tiers

| Tier | Dir | What | Extra bar |
|---|---|---|---|
| **Templates** | `templates/` | Complete, end-to-end flows — *fork-and-go* | — |
| **Patterns** | `patterns/` | Small, single-concept building blocks — *composable* | declared pins in `schema.yaml` (#1468) |

The tier is carried by **directory**, not a field.

## 3. Metadata model (settled)

**One manifest, generate the rest.** `package.yaml` is the manifest; everything an entry declares is a *fact the package states about itself*. There is **no Colony-specific metadata** — no `template.yaml`, no `catalog.yaml`, no `colony:` block.

The design walked through three dead ends to get here:
- A separate `template.yaml` — dropped: it *restated* what the contracts already declare (name→`package.yaml`, interface→`schema.yaml` pins, deps→`requires`). The restatement footgun.
- A `colony:` block in `package.yaml` — dropped: the loader silently ignores unknown fields, so it'd be non-authoritative cruft in a file meant to be forked.
- A `maturity` rating (flagship/reference/pattern) — **removed entirely.** It was the *only* field that couldn't be self-asserted (a package can't credibly rate its own trust tier — forgeable, and stale when forked). Removing it removes the last reason for any registry-side authored file.

What's left:
- **`package.yaml` = all self-facts** — `name`, `version`, `author`, `description`, `platform_version`, `flows`, plus (proposed, strictly-modeled) `keywords`, `license`, `repository`, `category`.
- **`index.yaml` = generated** — package self-facts + **verification status** (green/red against current spec, and when — computed at verify time, never self-asserted) + a **derived capability surface** (tools/backends reached, from `tools.yaml`/agents).

The line: **`package.yaml` = what the package says about *itself*; the registry only *generates*** (verification, derived facts). Nothing authored is registry-judgment.

## 4. Anti-rot (the non-negotiable)

The platform dialect churns; templates rot (twitter-gen drifted in weeks). Verification is **structural**:
- CI (`.github/workflows/verify.yml`) builds `swarm` from `division-sh/swarm@master` and runs `swarm verify` on **every** entry against current `platform-spec` — on push, PR, **and daily**.
- A platform change that breaks an entry turns CI red → **fix or quarantine**, never weaken the check. The result feeds each entry's `verification` in `index.yaml`.
- TODO: add the #1252 scenario runner (`swarm test`) once it lands.
- **Depends on platform-side `platform_version` *enforcement*** (Issue A) — otherwise the compat pin is decorative and entries rot silently.

## 5. Repo structure

```
colony/
├── README.md · CONTRIBUTING.md · CODEOWNERS      # what it is + the bar + curation
├── index.yaml                                    # GENERATED catalog (feeds future `swarm add`)
├── scripts/build-index.sh                        # derives index.yaml from each package.yaml
├── .github/workflows/verify.yml                  # anti-rot CI
├── docs/authoring-guide.md · docs/design.md      # how to author + this doc
├── templates/<name>/                             # fork-and-go flows (+ scenarios/)
└── patterns/<name>/                              # composable building blocks (+ scenarios/)
```

## 6. Platform work (two issues, split)

Most of the metadata work is **platform-side, not Colony-specific** — it makes every flow package a proper, marketplace-ready manifest. Split into:

### Issue A — URGENT: enforce `platform_version`
`platform_version` is parsed in ~8 files but **enforced nowhere** (code probe found no semver-range/satisfaction check). So it's decorative — the root cause of silent dialect-drift rot. `verify`/boot must **reject when the running spec version doesn't satisfy the declared range**. Colony cannot exist without this; it gates everything else.

### Issue B — manifest hygiene: model self-facts strictly, stop silent-ignore
- Add `keywords`, `license`, `repository`, `category` as **modeled, validated** manifest fields.
- **Unknown top-level fields should warn or fail**, not be silently ignored — silent-ignore in a marketplace manifest is a footgun (you declare a license nothing reads).
- **Not** adding `requires`: it already exists as the import-boundary contract (`inputs/outputs/policy/credentials` the parent binds) and is enforced. The capability-audit surface (tools/backends reached) is **derived + surfaced**, not authored.

## 7. Consumers / related work

- **#1468 flow interfaces** — Colony derives the interface from here; do not restate.
- **#1252 scenario runner** — the smoke-proof mechanism (`scenarios/` + `swarm test`); CI runs it once it lands.
- **#1450 sealed package boundary** — makes fork/import clean; Colony is its acceptance test.
- **#1447 batch agent** / logic-node work — source of the first `batch-classify` pattern and the rebuilt flagship.

## 8. Sequencing & status

- **Now:** scaffold in place (this repo), empty of entries (no drifted content). Local-only — no GitHub remote yet.
- **Next:** file Issues A + B; as #1447 lands, add `patterns/batch-classify`; rebuild `twitter-prospecting` on #1447 + logic-node and add it as the first template (verified against fresh master).
- **Deepen** as #1450/#1468/#1252 stabilize — don't over-invest in entries before the composition primitives are solid, or they'll be rewritten.

## 9. Open questions
1. Separate GitHub repo `division-sh/colony` (mirrors npm/registry separation; needs cross-repo CI token) vs a directory in the main repo?
2. `category` as a first-class field vs just `keywords`? (Kept as a field for catalog grouping; low stakes.)
3. Where does the derived capability surface live — `swarm describe` output consumed by `build-index`, or a standalone derivation?

_Resolved: no separate metadata file; `maturity` removed; `requires` untouched (import contract exists); curation is generated, not authored._
