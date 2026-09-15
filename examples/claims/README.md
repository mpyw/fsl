# Insurance claim processing — scaled FSL corpus (S / M / L / XL)

A multi-tier example corpus modeling **insurance benefit claim processing**: intake (受付),
assessment (査定), payment (支払), ledger posting (台帳記帳), return-for-correction (差戻し),
withdrawal (取下げ), statute-of-limitations lapse (時効), and re-assessment (再査定).
Each tier extends the previous one in both structure and scale; S is the minimal complete
requirements-layer form, and M/L/XL add design, refinement chains, saga, and external actors.

## Bounded-domain policy (required)

All tiers use **finite domains only** in kernel/requirements/design specs: no unbounded
`Int` in `state`, struct fields, or action parameters. Range types (`0..N`), enums, and
`Map`/`Seq` with bounded keys/capacity carry every quantity (days, ledger counters,
amounts). This keeps RSS growth attributable to spec depth rather than an infinite state
space.

`dbsystem` files follow the finite schema-window model (`schema 0..K`); column types use
the dialect’s `Int`/`Text`/`Bool` names over bounded rollout snapshots, not kernel state.

**This example’s domains are intentionally small.** Scaling them up hits
[#1041](https://github.com/ymm-oss/fsl/issues/1041): `fslc check` on inline
`implements` can reach 28–30 GB RSS. That is an fslc defect, not a spec error.
The n4 and n8 scaling snapshots are not stored in this repository. The required
whole-tree check populations include `specs/`, `examples/`, and
`rust/fslc/tests/fixtures/`; no repository location can hold an input that
exceeds 8 GiB locally without risking the same OOM in a required check. The
reproduction procedure is maintained in [issue #1041](https://github.com/ymm-oss/fsl/issues/1041), following the source-string precedent in
`rust/fslc/tests/issue_697_all_properties_memory.rs`.

Scale axes (monotonic S → XL): **state count × actions × invariants × verification depth**.

## Authoring procedure (required)

**.fsl を1ファイル書き終えたら、次のファイルへ進む前に
`~/.local/bin/fslc check <そのファイル>` を実行し、exit=0（negative なら宣言どおりの
非0と kind 一致）を確認する。** commit 前ではなく**ファイルごと**。新しい検査は作らず
既存 `fslc check` gate のみ。報告時は書いたファイルごとの exit code を列挙する
（「全部通りました」のみは未完了）。

## Delimiter convention (comma-separated brace lists)

Kernel/requirements/design blocks with **multiple elements on separate lines** use
**comma separators between elements**, matching `specs/cart_v1.fsl:6-9` (`stock:` line
ends with `,` before the next binding). Single-line forms (`struct Claim { a: T, b: U }`,
inline `enum { A, B, C }`) already use commas. `init { }` bodies are semicolon-free
statement sequences (same as cart_v1) — not comma lists.

### Delimiter audit (S/M, 2026-09-15)

Reference: `specs/cart_v1.fsl` (`state`), `examples/layers/return_impl_refines.fsl`
(`enum abstraction`), `examples/db/safe_add_nullable_column.fsl` (`reads` lists).

| File | Block | Lines | Precedent | Judgment |
|---|---|---:|---|---|
| `S/claims_S.fsl` | `state {` | 22–26 | cart_v1 `state` | **fixed** (commas after `claim:`, `paid:`) |
| `S/claims_S.fsl` | `enum CSt` | 15–18 | inline enum | OK (commas) |
| `S/claims_S.fsl` | `struct Claim` | 20 | inline struct | OK (comma) |
| `S/claims_S.fsl` | `init {` | 28–32 | cart_v1 `init` | N/A (statements) |
| `S/negative/pay_before_assessment.fsl` | `state {` | 15–18 | cart_v1 | **fixed** |
| `M/claims_M.fsl` | `state {` | 20–24 | cart_v1 | **fixed** |
| `M/claims_M.fsl` | `struct Claim` | 18 | inline | OK |
| `M/claims_M_design.fsl` | `state {` | 51–55 | cart_v1 | **fixed** |
| `M/claims_M_design.fsl` | `enum abstraction` | 7–17 | return_impl_refines | N/A (row syntax) |
| `M/claims_M_design.fsl` | `map … = Claim {` | 18–22 | return_impl_refines `map sys` | OK (field commas) |
| `M/negative/design_pay_bypass.fsl` | `state {` | 35–39 | cart_v1 | **fixed** |
| `M/negative/design_pay_bypass.fsl` | `map … = Claim {` | 9–18 | return_impl_refines | OK |
| `M/claims_ledger_db.fsl` | `reads`/`writes` | 24–35 | safe_add_nullable_column | OK (comma lists) |
| `M/claims_ledger_db.fsl` | `table`/`column` | 7–16 | db dialect | N/A (semicolon rows) |

Parse defect class: multiline `state {` missing inter-element commas (`expected ','`
at next binding). Fixed in commit `842ec09f`; re-verified on asset sha256
`fdfccc01…e488`.

## Scale table

Measured with `wc -l`, hand-counted actions/properties/invariants, and finite state-space
formulas below. Verification depth in examples: **6**. Binary: `fslc 4.5.0` sha256
`fdfccc01…e488`.

| Tier | Files | Lines | State vars | **State count** | Actions | Invariants | Properties | Structure |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| S | 2 | 217 | 3 | **11,664** | 9 | 2 | 9 | `requirements` |
| M | 4 | 594 | 6 | **373,248** | 18 | 4 | 12 | S + `amount` + `dbsystem` + design |
| L | 9 | 743 | 6 | **373,248** | 20 | 5 | 15 | M + business + refinements + saga + n2 scaling baseline |
| XL | — | — | — | — | — | — | — | *(pending)* |

### State-count formulas

Treat each `Map<ClaimId, …>` slot independently; multiply global scalars at the end.

**S** (`claims_S.fsl`):

```text
|Claim| = |CSt| × |Day| = 9 × 4 = 36
|ClaimId| = 2
state_count = |Claim|^|ClaimId| × |LedgerCount|^2
            = 36^2 × 3 × 3
            = 11,664
```

**M requirements** (`claims_M.fsl`):

```text
|Claim| = |CSt| × |Day| × |Amount| = 9 × 4 × 4 = 144
state_count_req = 144^2 × 3 × 3 = 186,624
```

**M design** (`claims_M_design.fsl`):

```text
|DClaim| = |DSt| × |Day| × |Amount| = 9 × 4 × 4 = 144
state_count_design = 144^2 × 3 × 3 = 186,624
```

**M tier total** (kernel specs summed; `claims_ledger_db.fsl` is a finite schema-window
compatibility model, not a kernel `Map`+counter state space):

```text
state_count = state_count_req + state_count_design = 373,248
```

Product check at depth 6: S = 11,664 × 9 × 1 × 6 = 629,856; M = 373,248 × 18 × 3 × 6 = 121,173,504 (each axis strictly larger than S).

**L tier** reuses the M bounded kernel (`Claim = 0..1`, `Amount = 0..3`, …) for the main
chain files; state count matches M requirements + design above. The checked example snapshot is
`claims_L_requirements.fsl` itself (`Claim = 0..1`; this is also the chain's requirements
input, not a separate baseline file — see `L/README.md`). To reproduce the scaling inputs, make
a copy of that file and change its sole `type Claim` bound:

```text
claims_L_requirements.fsl  type Claim = 0..1   (baseline, this file as-is)
copy with `type Claim = 0..3`                  (n4 reproduction)
copy with `type Claim = 0..7`                  (n8 reproduction)
```

The copied n4/n8 inputs are not retained in `specs/`, `examples/`, or
`rust/fslc/tests/fixtures/`, because each is a required whole-tree check
population. See issue #1041 for the materialized-source reproduction procedure;
do not commit the high-memory copies.

## Verify warnings (deadlock)

`verify` may report a `deadlock` warning when every claim reaches a **terminal business
state** (`Posted`, `Withdrawn`, or `Lapsed`) and no further action is enabled. That is
intentional completion, not a spec defect — the warning points at those terminal stops.
Do **not** suppress it with `--deadlock ignore` (see issue #998).

## Mutate calibration

Binary: `fslc 4.5.0` sha256 `fdfccc01…e488`. Depth 6, `--max-mutants 400`. Verify no cap-drop
two ways: absence of a `mutant cap … dropped` line in `notes`, **and** `summary.total < 400`
(a cap can truncate silently — the `notes` line alone is not sufficient). Current values
(2026-09-15, current spec content):

| spec | total | killed | survived | kill_rate |
|---|---:|---:|---:|---:|
| S | 331 | 299 | 32 | 0.9033 |
| M | 342 | 304 | 38 | 0.8889 |

Both rows were reproduced independently by two runs (mem-ex, orchestrator) with identical
total/killed/survived/kill_rate and identical op breakdown. The caps differed: S used
`--max-mutants 400` both times; M used 400 (mem-ex) and 2000 (orchestrator) — since 2000
comfortably exceeds any plausible builtin-mutant count for this spec, the two runs agreeing is
additional evidence that neither cap silently truncated the enumeration.

Always raise `--max-mutants` until `notes` has **no** `mutant cap … dropped` line, and report
the full `notes` array beside `summary.kill_rate` — the default cap of 200 truncates the spec's
tail constructs (`withdraw` / `advance_day` / `lapse`) and reports a head-only, misleadingly
higher rate.

### Which properties kill which mutant groups

- `forbidden FB-1..18` reject `withdraw` / `lapse` / `advance_day` once a claim has moved past
  the pre-payment stages (`Approved`, `Paid`, `Posted`, `Withdrawn`, `Lapsed`). These kill the
  `enum_constant_swap` mutants that would otherwise let a stage guard admit a post-payment stage.
- `acceptance AC-3` / `AC-4` witness that withdrawal and statute lapse remain reachable from
  `Assessment`.
- `acceptance AC-5..10` witness that withdrawal and statute lapse remain reachable from the three
  other pre-payment stages (`Intake`, `Returned`, `Reassessment`) that `AC-3`/`AC-4` do not cover.
  A `forbidden` rule cannot substitute for this: it rejects an already-forbidden transition, it
  cannot prove a still-allowed one stays enabled — that needs a positive witness. Calibrated by
  deletion on `S/claims_S.fsl` (each round restored and confirmed by an empty `git diff` before
  the next):
  - Removing `AC-5` returns exactly 3 `enum_constant_swap` mutants
    (`withdraw requires #1 Intake->{Assessment,Returned,Reassessment}`) from `killed` to
    `survived` (kill_rate 0.9033 → 0.8943); `AC-6`/`AC-7` return the same 3-mutant shape for
    `Returned`/`Reassessment`.
  - Removing `AC-8`, `AC-9`, or `AC-10` each returns 6 mutants, not 3 — this is an observation,
    not yet a decomposed mechanism. **What is confirmed:** 3 of the 6 are the `lapse requires #1`
    stage-guard swaps for that stage (the same shape as AC-5..7); the other 3 include at least one
    `advance_day requires #1 Lapsed->…` swap. **What is not confirmed:** why the count is 6 rather
    than the 3+1=4 that would match the withdraw-side shape, and which oracle besides the removed
    acceptance was covering the other mutants in that set. Do not read "6" as a validated
    mechanism until that gap is closed.

### Survivor breakdown (per-mutant, from reading `S/claims_S.fsl`; not re-verified by mutate re-run)

Exact op counts, both tiers (`fslc mutate <spec> --depth 6 --max-mutants 400`, current spec
content):

| op | S | M |
|---|---:|---:|
| `enum_constant_swap` | 11 | 11 |
| `fair_remove` | 9 | 9 |
| `type_bound_lo_minus1` | 3 | 4 |
| `type_bound_hi_plus1` | 3 | 4 |
| `requires_remove` | 2 | 3 |
| `integer_literal_minus1` | 2 | 3 |
| `integer_literal_plus1` | 2 | 3 |
| `assignment_remove` | 0 | 1 |
| **total** | **32** | **38** |

`enum_constant_swap` is the largest group in both tiers, not `fair_remove`. Per-target
classification for S (all 32 accounted for; M shares the same `enum_constant_swap` targets plus
an `amount`-related set analyzed separately below):

- **(B) equivalent, domain-floor/ceiling argument** — `type_bound_lo_minus1`/`type_bound_hi_plus1`
  on `ClaimId`/`Day`/`LedgerCount` (6): the extended value is outside every declared range and no
  `init`/action ever produces or requires it.
- **(B) equivalent, fairness has no depth-6 safety effect** — `fair_remove` on all nine actions
  (9): `mutate` exercises safety oracles (acceptance/forbidden/invariant), not liveness; fairness
  only affects `verify`'s fairness-driven search.
- **(B) equivalent, verified by exhaustive guard/invariant check** — `enum_constant_swap` on
  `receive assignment Assessment->Reassessment` and `resubmit assignment Reassessment->Assessment`
  (2): every guard and invariant in this spec that mentions `Assessment` also mentions
  `Reassessment` in the same clause (`assess`, `return_to_claimant`, `withdraw`, `lapse`,
  `advance_day`); nothing distinguishes a claim sitting in one versus the other.
- **(B) equivalent, domain-size argument** — `requires_remove` and `integer_literal_plus1` on
  `pay requires #2` / `post_ledger requires #2` (4): the guard is `paid < 2` / `posted < 2` and
  `ClaimId = 0..1` caps the number of claims at 2, so the count can never exceed 2 regardless of
  whether the cap reads `< 2`, is absent, or reads `< 3`.
- **(A) property gap** — `enum_constant_swap` on `return_to_claimant requires #1
  Reassessment->{Intake,Assessment,Returned,Approved,Withdrawn,Lapsed}` (6): `return_to_claimant`
  has no `forbidden` rule at all in this spec (unlike `withdraw`/`lapse`, which have FB-1/5/6/7/
  8/11/12/15/16/17). Two of the eight possible swaps (`->Paid`, `->Posted`) are killed because they
  trip `LedgerConsistent`; the other six are not caught by anything.
- **(A) property gap** — `enum_constant_swap` on `advance_day requires #1
  Lapsed->{Paid,Posted,Withdrawn}` (3): this swap re-admits `Lapsed` claims to `advance_day`.
  `FB-9` exercises exactly this trace (advance a lapsed claim) but is over-determined: by the time
  the trace reaches that step, `days == LAPSE_AFTER`, so the separate `days < LAPSE_AFTER` guard
  already rejects the call — `FB-9` passes for a reason unrelated to the stage-exclusion clause it
  was meant to cover.
- **(A) property gap** — `integer_literal_minus1` on `pay requires #2` / `post_ledger requires #2`
  (2): tightens `paid < 2` to `paid < 1` (and `posted` likewise). Unlike the domain-size argument
  above, this bound *is* distinguishable — it would reject the second of two claims being paid in
  one trace — but no acceptance or forbidden case pays or posts both `ClaimId`s in the same trace,
  so nothing exercises the difference.

This reasoning is static (read from the guards/invariants and the mutate tool's own `survived`
verdict), not confirmed by adding a new oracle and re-running mutate to watch the mutant flip to
`killed`. Treat the (A) items as findings to fix or file, not as already-fixed.

M's `enum_constant_swap` survivors are the identical 11 targets with the same classification
(the `return_to_claimant`/`advance_day` guards are unchanged from S). M's additional survivor
targets (`type_bound_*` on `Amount`, `receive requires #2` = `a > 0`, `init assignment`,
`assignment_remove | receive assignment`) have **not** been individually classified — the
`amount` field is written by `receive` but not read by any invariant/acceptance/forbidden in
`claims_M.fsl` (it exists for the design/ledger layers), so several of these are plausibly
equivalent-at-this-layer, but that has not been checked mutant-by-mutant the way S was. List them
as **未分類 (unclassified)**, not equivalent.

## Commands (S tier)

```bash
fslc check examples/claims/S/claims_S.fsl
fslc verify examples/claims/S/claims_S.fsl --depth 6
fslc check examples/claims/S/negative/pay_before_assessment.fsl
fslc mutate examples/claims/S/claims_S.fsl --depth 6 --max-mutants 400
# inspect stdout JSON: summary.kill_rate AND notes[] (no cap-drop line)
```

## Commands (M tier)

```bash
fslc check examples/claims/M/claims_M.fsl
fslc verify examples/claims/M/claims_M.fsl --depth 6
fslc check examples/claims/M/claims_ledger_db.fsl
fslc check examples/claims/M/claims_M_design.fsl
fslc check examples/claims/M/negative/design_pay_bypass.fsl
fslc mutate examples/claims/M/claims_M.fsl --depth 6 --max-mutants 400
fslc mutate examples/claims/M/claims_M_design.fsl --depth 6 --max-mutants 400
# inspect stdout JSON: summary.kill_rate AND notes[] (no cap-drop line)
```

| spec | total | killed | survived | kill_rate |
|---|---:|---:|---:|---:|
| `claims_M.fsl` | 342 | 304 | 38 | 0.8889 |
| `claims_M_design.fsl` | 341 | 281 | 60 | 0.8240 |

`claims_ledger_db.fsl` is a `dbsystem` document, not a `mutate`-covered dialect: `fslc mutate
examples/claims/M/claims_ledger_db.fsl` returns exit 2, `result:"error"`,
`kind:"semantics"`, `message:"mutate expects a spec-like FSL file"` (reproduced directly, not
just cited). This matches `rust/fslc/src/main.rs:10925-10930`, the `mutate` command's dialect
match: it special-cases `Spec`/`Business`/`Requirements`/`Compose`/`Domain` surface documents and
falls through to this same `error_output("semantics", "mutate expects a spec-like FSL file")` for
every other `SurfaceDocument` variant, `Db` (dbsystem) included. No design document states this
as an intended scope boundary — `docs/DESIGN-mutate.md` does not mention `dbsystem` at all, and
the only prior scope expansion on record (issue #727) added `Domain` support without addressing
`Db`. This is not filed as a defect: the rejection is fail-closed with a located diagnostic, not
an accepted-but-meaningless construct (the class AGENTS.md's mutation/vacuity rule is about), so
it is recorded here as current behavior rather than an open issue.

## Commands (L tier)

```bash
fslc check examples/claims/L/claims_L_business.fsl
fslc check examples/claims/L/claims_L_requirements.fsl
fslc check examples/claims/L/claims_L_design.fsl
fslc check examples/claims/L/claims_L_saga.fsl
fslc check examples/claims/L/negative/design_pay_bypass.fsl
fslc chain examples/claims/L/fsl-project.toml   # business/requirements/design depth=6
fslc mutate examples/claims/L/claims_L_business.fsl --depth 6 --max-mutants 400
# inspect stdout JSON: summary.kill_rate AND notes[] (no cap-drop line)
```

Design→requirements refinement lives in `claims_L_design_refines_requirements.fsl`
(external mapping, not `fslc check` on its own). Requirements→business uses inline
`implements` in `claims_L_requirements.fsl` plus `claims_L_requirements_refines_business.fsl`
for chain documentation.

### L-tier mutate

| spec | total | killed | survived | kill_rate | oracle |
|---|---:|---:|---:|---:|---|
| `claims_L_requirements.fsl` | 342 | 276 | 66 | 0.8070 | invariant ×2, forbidden ×5, acceptance ×4 |
| `claims_L_design.fsl` | 341 | 153 | 188 | **0.4487** | invariant ×2, acceptance ×1 |
| `claims_L_business.fsl` | 188 | 39 | 149 | **0.2074** | `reachable` ×1 |
| `claims_L_saga.fsl` | 110 | 2 | 108 | 0.0182 | `saga` ×1 (`domain` dialect — see the ~10% scope note above) |

`claims_L_business.fsl`'s low kill_rate is not hidden and is not a policy violation (the
existing "below ~10% is hollow" convention above does not trigger at 0.2074), but it is worth
explaining: this file carries exactly **one** property, `reachable CanPost`, and zero
`invariant`/`forbidden`/`acceptance` blocks — by design, it is a coarse kernel meant to be a
`fslc refine`/`fslc chain` target for `claims_L_requirements.fsl`, not a spec meant to carry its
own dense mutation coverage. All 149 survivors trace to that single structural cause — an
existential reachability witness (`exists c: Claim { stage[c] == BPosted }`) tolerates most
individual guard/assignment mutations, since either `ClaimId` instance reaching `BPosted` by any
surviving path satisfies it — grouped by action and op:

| group (action × op) | survivors |
|---|---:|
| `withdraw`/`lapse` stage-guard `enum_constant_swap` (3-way disjunction, 8 targets each) | 56 |
| `return_case`/`resubmit` stage-guard `enum_constant_swap` (single-literal guard, 7 targets each) | 28 |
| `intake`/`approve`/`pay`/`post` stage-guard `enum_constant_swap` (single-literal guard) | 24 |
| `init` assignment `enum_constant_swap` | 5 |
| `equality_operator_flip`, one per action (all 8 actions) | 12 |
| `requires_remove`, one per action (all 8 actions) | 8 |
| `requires_negate`, one per action (all 8 actions) | 8 |
| `assignment_remove` (`return_case`, `resubmit`, `withdraw`, `lapse`) | 4 |
| `type_bound_*` on `Claim` (floor/ceiling) | 4 |
| **total** | **149** |

(This grouping is coarser than S's individual per-mutant classification; per the request,
group-level is sufficient here.)

This tier's actual property assurance for the business layer is the refinement checks below
(business is the abstraction `claims_L_requirements.fsl` refines), not this file's own mutate
score. **This "structural, not an oracle gap" reading is this corpus's own analysis** (backed by
the mutate re-run above and by reading the spec's single `reachable` property); **it has not been
independently confirmed by the orchestrator the way the ~10%-domain-dialect finding was.**
**Do not add invariants/forbidden/acceptance to this file to raise the number** — the properties
already live at the requirements layer (FB-1..18/AC-1..10 in the S/M/L requirements files), and
duplicating them here would be an unverified-until-calibrated addition of exactly the kind the
project is currently declining to make.

`claims_L_design.fsl`'s 0.4487 is thinner than `claims_L_requirements.fsl`'s 0.8070 — 2
invariants and 1 acceptance versus 2 invariants, 5 forbidden, and 4 acceptance. That comparison
(design carries less of its own oracle density than requirements, with the difference made up by
`fslc refine`/`fslc chain` showing design refines requirements) is this corpus's own reading of
the numbers, not something the orchestrator independently verified. No oracle was added here
either.

**Mutate reporting (required):** raise `--max-mutants` until `notes` contains **no**
`mutant cap … dropped` line (default 200 truncates late actions such as `lapse`;
`docs/LANGUAGE.md` states kill-rate is not a completeness measure). Report **both**
`summary.kill_rate` **and** the full `notes` array — rate-only reports are incomplete.
**No cap drop does not mean every declaration was mutated** — those are separate claims.

Also aggregate `mutants[].target` against the spec’s declared **types, consts, init, and
actions** (per `docs/DESIGN-mutate.md` §2; **invariants and acceptances are oracles, not
mutation targets** — do not treat their absence from `target` as a gap). List any declared
type/const/init/action that never appears in `target` (write **0 件** when none).
Mutate kill-rate below ~10% indicates a hollow spec (existing convention); do not weaken
invariants to chase green mutate output. **This ~10% convention applies to the spec-like
dialects (`requirements`/`business`/`design`), not to `domain`.** Three `domain`-dialect
examples already merged to `main` all sit below 10% (`fslc mutate <file> --depth 6
--max-mutants 2000`, cap-drop-free, spot-checked here against `order_fulfillment_saga.fsl`):

| spec | total | killed | survived | kill_rate | oracle |
|---|---:|---:|---:|---:|---|
| `examples/domain/order_fulfillment_saga.fsl` | 226 | 3 | 223 | 0.0133 | `saga` ×1, invariant ×0 |
| `examples/domain/order_functional_ddd.fsl` | 57 | 1 | 56 | 0.0175 | invariant ×1 |
| `examples/domain/order_async_effect.fsl` | 252 | 21 | 231 | 0.0833 | invariant ×1 |
| `examples/claims/L/claims_L_saga.fsl` (this corpus) | 110 | 2 | 108 | 0.0182 | `saga` ×1, invariant ×0 |

`claims_L_saga.fsl`'s 0.0182 is *higher* than the nearest precedent
(`order_fulfillment_saga.fsl`, 0.0133) with the same oracle shape (one `saga`, zero
`invariant`), not an outlier below it. `mutate`'s `Domain` support was added by issue #727
specifically to give domain specs *a* self-check channel; domain correctness is carried by a
different check (domain findings/effects, `fslc domain` commands), not by acceptance/forbidden
density the way `requirements` is. **No oracle was added to `claims_L_saga.fsl` to raise this
number** — there was no gap to close once the precedent was measured.
