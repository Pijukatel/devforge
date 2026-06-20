# devforge — repo-level registry merge for domain engines (design record)

**Date:** 2026-06-20
**Status:** Designed — not yet built. Living docs to update on build:
[`README.md`](../../../README.md), [`docs/devforge-config.md`](../../devforge-config.md),
[`VENDORED.md`](../../../VENDORED.md).

This records the *why* and the *what* for letting a target repo bring its own engines into
the gated loop. For *how it works once built*, the living docs above are authoritative.

## Problem

devforge is a generic gated-loop orchestrator, but it gets used mostly inside one target
repo (`apify-mcp-server`) for MCP work. That work needs three domain-specific things in the
loop:

1. **MCP spec + MCP Apps knowledge** — so planning and review know what "correct" means.
2. **Project conventions + the oracle commands** — naming rules, `pnpm run type-check / lint
   / test:unit`, public/internal separation.
3. **End-to-end verification** — building the server and probing it with `mcpc`.

The constraint: **none of this may enter the generic devforge repo.** The MCP server lives
in a separate repo, and devforge must stay general so any project can use it. devforge is a
**shared install** in the target repo (not vendored into it).

## Key insight — the domain context already exists in the target repo

Each of the three inputs maps onto an asset that **already lives in `apify-mcp-server`**, and
onto devforge's existing structure. Nothing new is authored:

| Input | Source (already in the target repo) | devforge touchpoint |
|-------|-------------------------------------|---------------------|
| MCP spec + Apps knowledge | the **`dig`** skill — its *Available-resources* map (MCP spec/SDK + Apps spec/SDK URLs and paths) and planning discipline | fills the **`architect`** slot |
| conventions + oracle commands | root **`AGENTS.md`** (auto-loaded when devforge runs in the repo) | ambient context + the **oracle** |
| verification | the **`mcpc-tester`** agent | a gating **`final_reviewer`** |

So the feature is not "inject MCP docs." It is "let a target repo point devforge's slots at
the engines and knowledge it already maintains."

## Decisions

1. **Domain engines are selected by the target repo's own `.devforge/registry.json` +
   `.devforge/config.json`.** No MCP knowledge enters the devforge repo. This *refines* the
   earlier configurable-slots out-of-scope note ("domain skills stay in their target repo as
   `config.local.json` swaps"): the mechanism is now a repo-level **registry** merge, which is
   strictly more capable (it can introduce brand-new `uses`, not just swap an existing one).

2. **Base + repo registry merge — the one new generic primitive.** devforge carries a **base
   registry** (today's generic `uses` + `slot_roles`) that ships with the install. At startup
   it loads the base, then **shallow-merges the current repo's `.devforge/registry.json`
   `uses` over it** (repo wins on key collision). A repo registry contributes **`uses` only**;
   `slot_roles` always comes from the base (the slot→role map is fixed). A repo with no
   `registry.json` runs on the base alone. This mirrors the existing `config.json` /
   `config.local.json` shallow-merge — same idea, applied to the registry.

3. **Two-root path resolution.** A base `use`'s `engine` path resolves relative to the
   devforge **install** (the skill directory); a repo `use`'s `engine` path resolves relative
   to the **repo root** (cwd). This lets `apify-mcp-server` point `dig` at
   `.claude/skills/dig/SKILL.md` in its own tree while the generic engines keep resolving from
   the install.

4. **Minimal repo footprint + two legibility touches.** The target repo commits **only its
   deltas** (`apify-mcp-server`: just `dig` + `mcpc-tester`). To keep "only two engines here"
   from being confusing:
   - an optional **`$comment`** pointer key in the repo `registry.json` stating it is
     intentionally partial and the generic engines come from devforge's base;
   - devforge **logs the fully-resolved merged registry** (every engine + its resolved path)
     to `.devforge/progress.md` at startup, where it already records the resolved config.

   DRY without being opaque: the source of truth stays small, but every run leaves a complete,
   concrete view of exactly what is wired.

5. **Shared install, not vendored.** devforge stays a shared install; the base travels with it
   so improvements flow into every target repo automatically. Vendoring the whole tool into
   the target repo (a complete self-contained registry, zero devforge change) was considered
   and rejected: it adds re-sync friction during active devforge development and an unwanted
   generic-tool footprint inside a published npm package. The merge primitive is the cost we
   pay to avoid that. (Vendoring remains a legitimate choice for anyone who prioritizes
   self-containment over update-friction — it needs no devforge code.)

## Worked example — lives in `apify-mcp-server`, NOT in devforge

`.devforge/registry.json` (committed in the target repo — only its deltas):

```jsonc
{
  "$comment": "MCP-only engines. The generic engines come from devforge's base registry — see docs/devforge-config.md.",
  "uses": {
    "dig": {
      "roles": ["architect"],
      "engine": ".claude/skills/dig/SKILL.md",
      "scope": "follow as instruction text (NOT Skill-invoked). Use dig's exploration + planning discipline, its Available-resources map (MCP spec/SDK + MCP Apps spec/SDK, internal repo, dev servers, mcpc), and its Key conventions. SKIP dig's intent-detection, its EnterPlanMode/ExitPlanMode gate, and its GitHub-issue creation — the orchestrator owns the gate. Write .devforge/design.md only: approach, files to change, test strategy (the oracle), risks."
    },
    "mcpc-tester": {
      "roles": ["reviewer", "final_reviewer"],
      "engine": ".claude/agents/mcpc-tester.md",
      "scope": "follow as instruction text. pnpm run build, then probe the live server via mcpc per its workflow; judge implemented behavior against design.md + task.md. Do NOT run or replace the unit suite (that is the oracle). First line `VERDICT: PASS|FAIL` (PASS only with zero findings of any severity), then findings tagged blocker/major/minor/nit with actual-vs-expected."
    }
  }
}
```

`.devforge/config.json` (committed in the target repo — its wiring):

```jsonc
{
  "slots": {
    "validate":    { "use": "brainstorming", "model": "opus" },
    "architect":   { "use": "dig",           "model": "opus" },
    "implementer": { "use": "feature-dev",   "model": "opus" },
    "reviewers":       [ { "use": "staff-review",  "model": "sonnet" } ],
    "final_reviewers": [ { "use": "thermonuclear", "model": "sonnet" },
                         { "use": "code-review",   "model": "sonnet" },
                         { "use": "mcpc-tester",   "model": "sonnet" } ]
  },
  "limits": { "inner_iterations": 3, "final_review_rounds": 2 },
  "plan_mode_gate": true
}
```

### Run flow (what the user observes)

1. **Setup** — load base registry → merge the repo's `registry.json` `uses` over it → validate
   the merged registry + config (`architect`'s role ∈ `dig.roles` ✓; `mcpc-tester` allowed as
   `final_reviewer` ✓). Log the resolved config **and the fully-resolved registry** to
   `progress.md`.
2. **Validate** (`brainstorming`) — clarify scope; run the GitHub-issue staleness/claim-ledger
   check if an issue was passed. Writes `task.md` + `validation.md`. Kept generic on purpose —
   `dig` does not do the staleness check.
3. **Explore** (orchestrator) — ground in the codebase.
4. **Architect** (`dig`, scoped) — **MCP knowledge enters here.** `dig` plans using its
   resource map (graceful when a sibling repo such as `../typescript-sdk` is not checked out)
   and key conventions, writing `design.md` — without its own gate or issue creation.
5. **Design gate — STOP.** Plan-mode render / `/devforge-approve-design`.
6. **Inner loop** — `feature-dev` implements → **oracle** runs the target repo's own commands,
   read from the ambient `AGENTS.md` (`pnpm run type-check / lint / test:unit`; not integration
   — AGENTS.md marks it humans-only) → `staff-review`. Iterate to green + clean.
7. **Final review** — `thermonuclear` + `code-review` + **`mcpc-tester`** in parallel.
   mcpc-tester builds, probes the live server, emits a `VERDICT`. A `FAIL` **reopens the inner
   loop** like any other final reviewer — verification genuinely gates the merge. It sits *on
   top of* the oracle, exactly as its own description intends (unit tests stay the source of
   truth; it is the e2e layer).
8. **Pre-merge gate — STOP.** `/devforge-approve-merge` → PR.

The three inputs land where they belong: **spec/apps knowledge → `dig` at planning;
conventions + oracle commands → ambient `AGENTS.md`; verification → `mcpc-tester` as a gating
final reviewer.**

### `mcpc-tester` placement is feature-dependent

`mcpc-tester` declares `roles: ["reviewer", "final_reviewer"]`, so the target repo's
`config.json` can wire it into more than one spot depending on what a feature needs (the
per-list dedup check still allows the same `use` to appear in both lists):

- **`final_reviewers`** — a single end-to-end probe before the merge gate. The default in the
  example above; cheapest, keeps the inner loop fast.
- **`reviewers`** — a live probe *every iteration*, for features whose behavior needs
  continuous verification while the implementation converges. Its `VERDICT` gates convergence
  like any reviewer.
- **both** — when a feature wants per-iteration probing *and* a final clean pass.

Two related uses sit deliberately **outside** the slot system:

- *Probing to understand current behavior* happens at **planning time** — the `dig`/architect
  engine already probes via `mcpc`. It is not a separate slot.
- *Per-iteration deterministic ground truth* stays the **oracle** (unit tests/lint, run by the
  orchestrator). `mcpc-tester` complements the oracle as a reviewer; it never replaces it, and
  an agentic prober is never folded *into* the deterministic oracle.

## Changes in the generic devforge repo

All domain-agnostic — no `dig`, `mcpc-tester`, or "MCP" anywhere.

- **Orchestrator (`SKILL.md`):** load base registry → shallow-merge repo `registry.json`
  `uses` → two-root path resolution → validate the *resolved* registry → log the resolved
  registry to `progress.md`. Update the *Setup* and *Slot dispatch* sections accordingly.
- **Relocate the base registry to ship with the skill** so it resolves from the install
  regardless of cwd. Today's `.devforge/registry.json` (the 6 generic `uses` + `slot_roles`)
  becomes the shipped base; a target repo's `.devforge/registry.json` is always treated as
  deltas. (Exact base-file location is a plan decision.)
- **Schema / validation:** tolerate a top-level `$comment` in `registry.json`; have
  `scripts/validate_config.py` validate against the *merged* registry. Add a registry-shape
  check for repo deltas (each `use` has `roles` + `engine`; roles are known).
- **`docs/devforge-config.md`:** document base + repo-registry layering, the `$comment`
  convention, the resolved-registry log line, and a per-repo "add a domain engine" recipe.
- **Tests:** merge precedence (repo overrides base), two-root path resolution, no-repo-registry
  fallback (base only), validation against the merged registry, `$comment` tolerated.
- **Update the configurable-slots design record** (`2026-06-19-configurable-slots-design.md`)
  out-of-scope line to point here.

## Out of scope

- **Vendoring devforge into a target repo** — rejected above; remains possible for anyone who
  wants full self-containment (it needs no devforge change).
- **Authoring the `apify-mcp-server` files** — downstream work in that repo (can itself be run
  through devforge). This spec covers only the generic primitive + the worked example.
- **Changing the oracle, the file contract, the two gates, or the dispatch contract.**
- **`config.local.json` behavior** — unchanged; still the gitignored per-developer override.

## Risks / open implementation questions

- **Path resolution mechanics** — locating the install/skill directory at runtime to resolve
  base engine paths is the key implementation detail; nail it in the plan.
- **Scoped `dig`** — confirm that following `dig`'s `SKILL.md` as instruction text with the
  gate/intent-detection/issue-creation stripped still yields a complete `design.md`.
- **`mcpc-tester` preconditions** — it needs `pnpm run build` and a live server; ensure it
  degrades/escalates cleanly (a clear `FAIL` with reason) when the server cannot start, rather
  than hanging.
