# Threat Actor Dossier Reframe — v2 Roadmap

**Status:** strategic scoping (plan-only; no implementation in this slice).
**Workflow:** `w-68-dossier-reframe-scoping`
**Authored:** 2026-05-27 by planner stage of W-68-DOSSIER-REFRAME-SCOPING.
**Source issue:** [#68 Reframe AP around Threat Actor Dossier puzzle](https://github.com/jarocki/ap/issues/68) (filed 2026-05-23).
**Drives:** the Phase 16 section of `MASTER_PLAN.md`. MASTER_PLAN carries the binding decisions and the slice index; this document carries the full rationale, slot schema, disposition tables, and decomposition detail. When the two diverge, MASTER_PLAN wins for binding decisions; this document wins for narrative and schema detail. Any binding update must edit both atomically.

---

## 0. Document Scope and Non-Scope

**This document IS:**
- The strategic scoping artifact for issue #68.
- The dossier slot schema v1.0.
- The decomposition of the reframe into an MVP slice plus follow-on slices.
- The disposition record for related open issues #29, #30, #31, #32, and the Original Intent crowdsourcing axis.
- The decision log for `DEC-68-DOSSIER-REFRAME-001..010`.

**This document is NOT:**
- An implementation plan. No source files are touched by this slice.
- A schema commitment. Slot vocabulary and weights are v1.0 — the MVP slice may refine them before the first implementer touches code.
- A schedule. Slice ordering is recommended; user product judgment may re-order without invalidating the rest.
- A supersession of any prior Decision Log row. Prior DEC-IDs continue to bind unless this document explicitly retires them with a successor DEC-ID.

---

## 1. The Reframe (verbatim, with planner annotations)

From issue #68 body, kept verbatim because it is the canonical statement of v2 product center:

> "Adversary Pursuit is **not** about pivoting through indicators to find factoids. It is about **piecing together a picture of a Threat Actor**:
> - their habits
> - their strengths
> - technology they use or are comfortable with — and the quirky way they use it
> - their motivations
> - their 'tells'
> - anything else that can match activity to their **persona fingerprint** or **predict what they'll do next**
> - or build a strategy for **confusing / denying / discouraging** further attack progress
>
> The right metaphor is a **dossier** — a **puzzle** where the pieces filled in are the actual score drivers. The more important pieces (the ones nobody else has, the ones that pin actor identity, the ones that predict the next move) are worth higher scores."

From the 2026-05-26 Project Reckoning (`reckonings/2026-05-26-reckoning.md`, Section VIII, item 3):

> "Issue #68 (Dossier reframe) is the **most important unmade decision in the project.** Filed 2026-05-23 with substantial body content reframing scoring from indicator-graph expansion to dossier-piece completion weighted by importance. It is conceptually closer to the Original Intent than today's trajectory. The planner has not addressed it. Every day it sits unscoped is a day the project's v2 center of gravity is unset."

**Planner interpretation:** the reframe is ratified (DEC-68-DOSSIER-REFRAME-001 below). The product center for v2 shifts from "indicator-graph traversal scored by per-module event volume" to "Threat Actor Dossier completion scored by piece importance, rarity, and predictive validation." The dossier is the unit of value. Indicators, TTPs, infrastructure observations, and tool-execution events become **evidence that fills dossier slots** rather than ends in themselves.

This is a re-foundation, not an incremental hardening. It re-orients scoring, gamification, reports, and the auto-pivot policy engine — all four of which already exist as v1 surfaces and must evolve coherently.

---

## 2. Strategic Ratification — Why the Dossier Metaphor Holds

The dossier-puzzle frame is the correct v2 center for five independently sufficient reasons. Any one of them would justify the reframe; all five reinforce it.

### 2.1 It is closer to the Original Intent than the v1 trajectory

The Original Intent (2022 vision README, preserved verbatim in MASTER_PLAN §1) names:

> "Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development."

"Pursuit" in the project name is not "pursuit of indicators" — it is "pursuit of an *actor*." The 2026-05-26 reckoning identifies this in Section VII: *"piecing together a picture of a Threat Actor… reads like a direct echo of 'Standardize hunting and pursuit techniques.' If anything, #68 is the Original Intent's voice clarifying itself."* The reframe is therefore not scope drift; it is overdue Original Intent clarification.

### 2.2 It serves Principle 4 ("modules are pure data producers") more cleanly than v1 scoring does

v1's `gamification/scoring.py` rewards module-call volume (parabolic decay over module-execution events; see `ScoringEngine.score_results` and the F62/F63 streak chain). That implicitly couples scoring to *which modules fire*, which is a v1 indicator-graph artifact, not a CTI value artifact. The dossier frame decouples scoring from module count: a single Shodan call that fills the "Infrastructure habits" slot with a previously-unseen banner fingerprint can outscore fifty routine VirusTotal lookups. This is *more* principle-aligned — modules remain pure data producers, the dossier becomes the analytic-value layer above them, and scoring observes the analytic-value layer instead of the tool-call layer.

### 2.3 It resolves an unstated tension in the existing gamification stack

v1 has five gamification engines (scoring, celebrations, badges, modes, hints) plus three event-bus layers (event_bus, autopivot, challenges). The F60 auto-pivot policy engine, F62 streak mechanic, F63 milestone catch-up, and F64 LLM-narration de-duplication all wrestle with the same underlying question: *"what counts as analytic progress?"* v1 answered with module-call cadence + indicator graph depth. That answer has been increasingly contorted (the F60 confidence gate, the F62 honesty constraint, the F64 panel-authority assertion all push back against rewarding mechanical activity). The dossier reframe replaces the strained answer with a coherent one: progress = dossier completion, importance-weighted.

### 2.4 It absorbs the Threat Hunter advisory pressure into the product, not just the spec

Phases 11 (F59 STIX provenance) and 12 (F60 quota-aware auto-pivot) responded to expert pressure by hardening the *evidence chain*, not by changing what the product values. The dossier reframe responds to the same pressure at the *value* layer: a professional threat hunter cares about "what do I now know about this actor?" — they do not care about "how many module calls did the tool make?" v1 has good evidence chain; v2 needs good analytic value chain. The dossier IS the analytic value chain.

### 2.5 It is architecturally cheap given v1's separation discipline

Per ADR-010 and the F59/F60/F61 single-authority chain, modules are already pure STIX 2.1 producers, workspace owns persistence, gamification observes events, and the event bus has been decoupled from the cmd2 console (DEC-EVENTBUS-002). Adding a Dossier aggregation layer that observes the same SCO stream the workspace already persists does NOT require touching modules, the event bus, or persistence semantics. The reframe rides on top of v1's separation discipline — exactly the way ADR-010 rode on top of Phase 1–4's module/console separation.

### 2.6 Non-superseded prior decisions

Ratifying the dossier reframe does NOT supersede any of these v1 invariants. They continue to bind:

- **ADR-005 (STIX 2.1 as internal data model)** — the dossier consumes STIX objects; it does not replace them.
- **DEC-59-STIX-PROVENANCE-001** (`workspace.store_stix_objects` is the sole `x_ap_*` authority) — the dossier reads provenance, never writes it.
- **DEC-WS-004 (ID dedup)** — the dossier observes deduplicated SCO state.
- **DEC-EVENTBUS-002 (opt-in event bus)** — dossier slot-fill events ride on the same opt-in bus; they do not create a parallel notification channel.
- **DEC-60-PIVOT-POLICY-001..007 (auto-pivot policy)** — the policy engine input vocabulary expands to include "would this pivot fill an empty high-value dossier slot?" but the gate structure (IOC filter + confidence gate + per-cascade budget + dry-run) is preserved.
- **Sacred Practice 12 (single authority per operational fact)** — every dossier-state-of-the-world question has exactly one owner module. See §4.
- **Principle 4 (modules are pure data producers)** — modules MUST NOT emit dossier slot values directly. They emit STIX SCOs as today; a separate Dossier extraction layer reads the SCOs.

---

## 3. Dossier Slot Schema v1.0

Refined from the issue's seven-slot draft into a v1.0 vocabulary of nine slots. The MVP slice (M-1 below) is authorized to refine this schema by ±2 slots before the first implementer touches code; further refinement requires a planner re-stage and a successor DEC-ID.

For each slot:
- **Name** and **definition**.
- **Evidence types** — which STIX SCO types or workspace fields can fill it.
- **Confidence levels** — `contested | low | medium | high`.
- **Source-attribution requirement** — which modules can supply the evidence (free-text; the M-2 slice authors the canonical mapping table).
- **Score weight** — relative weight in dossier scoring; routine baseline = 1.0. Weights are recommendations; the M-3 scoring slice authorizes weight refinement under its own Evaluation Contract.

| # | Slot | Definition | Evidence types | Confidence basis | Score weight |
|---|------|------------|----------------|------------------|--------------|
| 1 | **Identity / Attribution** | Who is this actor? Handles, infrastructure ownership claims, language artifacts, persona fingerprints linked to prior reporting. | `email-addr`, `domain-name`, `user-account`, `identity`, `x509-certificate` Subject/Issuer; report-document references; persona-fingerprint tokens. | High = independently corroborated by ≥2 evidence types from ≥2 modules. Medium = single high-trust source (e.g., signed CT log + WHOIS owner match). Low = single uncorroborated source. Contested = two sources disagree. | **5.0** (highest — Identity is the puzzle's keystone) |
| 2 | **TTPs and Tradecraft** | Preferred CVEs, loaders, C2 frameworks, payload quirks, build-environment fingerprints. | `vulnerability`, `malware`, `attack-pattern`, `tool` (STIX SDOs); file hashes via `file` SCO; binary-section patterns via `file:hashes`/`file:extensions`. | High = ATT&CK technique mapping + ≥2 corroborating samples. Medium = single technique with high-trust source. Low = inferred from infrastructure only. | **3.0** |
| 3 | **Infrastructure Habits** | Hosting preferences, registrar preferences, OAST tools, naming patterns, TLS cert reuse, banner fingerprints. | `domain-name` + WHOIS provenance, `ipv4-addr`/`ipv6-addr`, `autonomous-system`, `x509-certificate`, Shodan banner SCOs, crt.sh CT entries. | High = ≥3 independent infrastructure observations sharing a pattern. Medium = 2 observations. Low = 1. | **2.0** |
| 4 | **Timing / Behavioral** | Working hours, weekday cadence, response-to-blocking latency, campaign tempo. | Module run timestamps (`x_ap_fetched_at`), STIX `first_seen`/`last_seen` on `indicator` SDOs, sighting timestamps. | High = ≥10 events across ≥3 distinct sources clustering to a timezone or weekday distribution. Medium = 5–10 events. Low = <5. | **2.0** |
| 5 | **Targeting Profile** | Industries, geographies, victim selection logic, sector/region clustering. | `identity` SDO `sectors`/`countries`, `location` SDO, victim domain TLDs, geolocated IP clusters. | High = ≥3 victims share sector + region. Medium = ≥3 share one axis. Low = single victim or weak clustering. | **2.5** |
| 6 | **Capability Ceiling** | What this actor can and can't do; what tools they don't pivot to; observed sophistication ceiling. | Absence-of-evidence inferences from TTP slot + module SKIP/empty-result patterns; comparison to ATT&CK technique inventory. | High = explicit user-validated inference. Medium = pattern-based inference from ≥5 observations. Low = single-observation inference. Contested = expected; capability ceilings are inherently uncertain. | **3.5** (high — capability ceilings are rare and predictive) |
| 7 | **Motivation Indicators** | Financial / hacktivist / nation-state / ego — *and the evidence for the call*. | Targeting profile slot + TTP slot + Identity slot triangulation; ransom-note text; political statements; victim-payment patterns. | High = ≥2 motivation-classification signals from independent slots. Medium = 1 signal. Low = inferred from a single victim type. Contested = multiple plausible motivations remain. | **3.0** |
| 8 | **Predictions Log** | Past AP-generated predictions about this actor's next moves, marked `pending`/`validated`/`falsified`. | Workspace-stored prediction records (new persistence surface, M-4 slice). | High = ≥2 validated predictions. Medium = 1 validated. Low = pending predictions only. (Falsified predictions are recorded but contribute zero to slot weight — and may be worth negative score under M-3 if it discourages reckless guesses; deferred decision DEC-68-DOSSIER-REFRAME-007.) | **4.0** (predictive validation is the deepest analytic signal; only Identity outranks it) |
| 9 | **Denial / Deception Strategies** | Concrete countermeasures tied to this actor's tells — what to block, what to honeypot, what to spoof. | User-authored notes in workspace + agent-generated suggestions grounded in the other 8 slots. | High = ≥3 distinct countermeasures linked to specific evidence. Medium = ≥1. Low = none yet. | **2.5** |

**Slot count rationale:** the issue named 7 slots; the planner adds **Predictions Log** (slot 8) and **Denial / Deception Strategies** (slot 9) because both are named in the issue body as primary product purposes ("predict what they'll do next" and "build a strategy for confusing / denying / discouraging further attack progress") and would otherwise be homeless. Both are explicitly v2-grade — neither has v1 scaffolding.

**Confidence vocabulary:** `contested | low | medium | high`. The four-level scheme matches v1's existing `IOC_CONFIDENCE_LOW/MEDIUM/HIGH` constant ladder (see `gamification/scoring.py`) plus a new `contested` value for slots where multiple sources disagree. `contested` is NOT a fifth confidence level — it is a flag that toggles the slot's contribution to dossier score to zero until adjudicated. M-3 owns the contested-handling rules.

**Score weight rationale (relative):** Identity (5.0) and Predictions (4.0) are highest because they are the slots that distinguish AP from a generic indicator-feed aggregator. Capability ceiling (3.5) is high because capability inferences are rare and predictively valuable. TTPs (3.0) and Motivation (3.0) are mid-high because they are the analytic-value backbone. Targeting (2.5) and Denial (2.5) are mid because they are downstream-derivable. Infrastructure (2.0) and Timing (2.0) are baseline-above-routine because they are the slots most directly fed by raw module output. **Routine per-IOC lookup with no slot impact = 1.0** (the v1 baseline, retained for backward compatibility under DEC-68-DOSSIER-REFRAME-005).

---

## 4. Scoring Authority Resolution

The most consequential single decision in this reframe. Three options were considered. Option (c) is selected.

### Option (a) — Replace `scoring.py` with `dossier_scoring.py`

**Pros:** Clean, single-authority. No backward-compat baggage.
**Cons:** Breaks F62 streak chain, F63 milestone catch-up, F64 panel de-duplication, and every existing test. Forces all v1.x downstream consumers (badges, celebrations, hints, modes) to re-wire to a new event vocabulary in a single landing.
**Verdict:** rejected — violates Sacred Practice 12 only superficially (a "single authority" replacement that breaks everything is single-authority in name only — the transition itself is a parallel-authority window that lasts the entire migration).

### Option (b) — Augment `scoring.py` with new event types

Add `EventType.DOSSIER_SLOT_FILLED`, `EventType.DOSSIER_PREDICTION_VALIDATED`, etc. to the existing `ScoringEngine` vocabulary. New events fire alongside old per-IOC events.

**Pros:** Backward-compatible. Each new event type lands as a discrete slice. F62/F63/F64 invariants remain valid.
**Cons:** Creates a permanent dual-authority surface: per-IOC events from `score_results` and per-slot events from a new dossier extractor. Sacred Practice 12 warns: *"'I'll add the new way but keep the old way as a fallback' creates dual-authority bugs."* The dual surface would be permanent because per-IOC events would never be retired (they fire from module runs and the dossier extraction layer reads module runs).
**Verdict:** rejected — the dual-authority window is permanent by construction, which is exactly what Sacred Practice 12 forbids.

### Option (c) — Layer a `Dossier` aggregator over scoring; re-weight at presentation time

Introduce a new `dossier/` subpackage (new authority, not under `gamification/`) that:
1. Subscribes to the same workspace SCO stream that scoring already observes.
2. Maintains per-investigation dossier state in workspace SQLite (new tables: `dossier_slot`, `dossier_evidence_link`, `dossier_prediction`).
3. Emits a new `ScoreEvent` subtype, `DossierSlotFilled` (and `DossierPredictionValidated`, etc.), through the existing `ScoringEngine` interface. These events are weighted by slot importance × confidence × rarity per the schema above.
4. **Per-IOC `EventType.MODULE_RUN_SCORED` events continue to fire**, but their weight is reduced to baseline (1.0) and the `Dossier` aggregator owns the "did this run advance the analytic state?" question.
5. Reports, celebrations, badges, modes, hints continue to consume `ScoringEngine` events without modification. They learn about the dossier indirectly, through the new event subtype.

**Pros:**
- Single new authority (`dossier/`), single existing authority (`scoring.py`) — no dual surface for *the same question*. Each authority owns a distinct question: scoring owns "what's a scoreable event?", dossier owns "what's the analytic-state-of-the-world?"
- F62 streak, F63 milestone catch-up, F64 panel de-duplication invariants are preserved: they continue to observe `ScoreEvent`s, which is what they already do.
- Modules untouched. Workspace untouched except for new tables.
- The reframe lands incrementally: each slot can be implemented as its own slice, and downstream consumers see the new event subtype the moment it fires.
- Sacred Practice 12 honored: every operational fact has one owner. Scoring owns score-event emission. Dossier owns slot state. Workspace owns SCO persistence and `x_ap_*` provenance. No question has two owners.

**Cons:**
- The aggregator layer adds one more subscription point to the event bus. Mitigated by F60's opt-in bus discipline.
- Score-event weight retuning is required: old per-IOC events drop to weight 1.0 while new slot-fill events scale up to 2.0–5.0. This is a *one-time* re-tune at the M-3 slice, not an ongoing dual-surface.

**Verdict:** selected. DEC-68-DOSSIER-REFRAME-002.

### Removal targets (no parallel-authority residue)

When the reframe lands:
- **No flag retained for legacy scoring.** No `AP_DOSSIER_DISABLE=1`, no `--no-dossier` CLI flag. Dossier is the v2 product center; v1.x users who don't want it stay on v0.1.x.
- **No per-IOC-events deprecation shim.** Per-IOC `MODULE_RUN_SCORED` events continue to fire as today — they are not deprecated; they are repurposed as routine baseline events at weight 1.0. Their meaning narrows ("a tool ran and produced an SCO") and the dossier layer carries the analytic-value semantics. This is a *narrowing*, not a parallel surface.
- **No "old scoring view" UI.** Whatever the agent panel, cmd2 `score` command, and reports currently show, they all migrate to dossier-aware presentation when M-6 lands. No backward-compat toggle.

---

## 5. Decomposition — MVP Slice + Follow-On Slices

Eight slices total. The first (M-1) is the MVP — smallest valuable shipping unit, ≤2 weeks of implementer work, deliverable as `v0.2.0`. The remaining seven are sequenced by unblock-value × blast-radius (lowest blast radius first when unblock-value ties).

### M-1 — MVP: Dossier Visualization Panel (v0.2.0 target)

**Scope:** Add a `Dossier` panel to `ap chat` that surfaces the *currently inferred* dossier state from the workspace's existing SCO stream. **No new scoring math.** **No new tables.** Pure read-side aggregation of what the workspace already knows, surfaced via a new Rich panel + a new LLM tool (`get_dossier_state`).

**Why this is the MVP:**
- It validates the slot schema (slot vocabulary survives contact with real workspace data) before any persistence or scoring change is committed.
- It is reversible: if the schema needs refinement, the panel + tool change without touching modules, workspace, or scoring.
- It produces an immediate user-visible demo: "open `ap chat`, run a hunt, type `dossier`, see the puzzle pieces fill in as you work."
- It honors the Sacred Practice 6 "Plan before code" gate: the MVP demonstrates the metaphor works against the real data before the project commits to the slot schema as a persisted-state contract.

**Slice size:** Small-to-Medium. ≤2 weeks implementer effort.

**Out of scope for M-1:**
- No new workspace tables.
- No new score events.
- No predictions log (slot 8 — depends on M-4).
- No denial/deception slot (slot 9 — depends on M-5 user-note surface).
- The MVP shows the 7 raw-evidence slots (1–7) with their inferred state from workspace SCOs. Slots 8 and 9 are stubbed as "Coming in M-4 / M-5."

**MVP acceptance criteria (to be hardened by the implementer slice's Evaluation Contract):**
- `dossier` chat meta-command renders a Rich panel showing each slot, its inferred confidence, and the count of evidence SCOs feeding it.
- `get_dossier_state` LLM tool returns the same data as a structured dict for the agent to reason over.
- Slot inference is read-only — no workspace mutations, no new persistence.
- At least one unit test per slot demonstrates inference correctness against synthetic SCO fixtures.

### M-2 — Module-to-Slot Mapping Layer

**Scope:** Declare, per existing module (15 total at landing of W-61-KEYLESS-HUNTERS), which dossier slots its output can populate and the extraction rules. New file `src/adversary_pursuit/dossier/extractors.py` (or one extractor per module under `dossier/extractors/`). Pure functions: SCO list → slot evidence pointers.

**Why second:** Unblocks M-3 (scoring) and M-6 (presentation) — both need to know "where does this evidence come from?" but neither depends on the persistent state surface that M-4 introduces.

**Slice size:** Medium. 15 modules × small extractor each.

**Removal targets:** None — this is purely additive over the existing module surface.

### M-3 — Dossier Scoring + Score Event Re-Tune

**Scope:** Introduce `DossierSlotFilled` and `DossierEvidenceConfidenceUpgraded` `ScoreEvent` subtypes. Re-tune existing per-IOC `MODULE_RUN_SCORED` events down to weight 1.0 baseline. Implement the slot-weight × confidence × rarity scoring per §3.

**Why third:** Scoring depends on extraction (M-2) and is the load-bearing v2 product-center change. Must land before predictions (M-4) because predictions are themselves scored.

**Slice size:** Medium-Large. Touches `gamification/scoring.py` (preserves F62 streak chain + F63 milestone catch-up) + new `dossier/scoring.py` aggregator. Forbidden shortcuts: NO env-var bypass; NO "old scoring fallback" flag; NO new ScoreEvent emission outside `ScoringEngine` (single-authority).

**Removal targets:**
- The per-IOC `MODULE_RUN_SCORED` weight constants in `scoring.py` are reduced to baseline 1.0 and the F62/F63 streak math is verified to remain correct under the new weight regime. No deprecation shim.

### M-4 — Persistent Dossier State + Predictions Log (slot 8)

**Scope:** Introduce three new workspace SQLite tables: `dossier_slot`, `dossier_evidence_link`, `dossier_prediction`. Migrate M-1's read-only inference into a persistent state surface that survives across sessions. Implement the Predictions Log slot — agent-generated next-move predictions are recorded with `pending`/`validated`/`falsified` status; later evidence triggers validation/falsification.

**Why fourth:** Persistence is a one-way door (once a table exists, it has to stay or be migrated out). Must land *after* the slot schema has survived M-1 contact with real data and M-3 scoring has confirmed the weighting works.

**Slice size:** Large. New SQLAlchemy models + Alembic-free migration (per DEC-DB-001 "no Alembic v1") + Predictions Log LLM tools + validation/falsification rules.

**Removal targets:** M-1's in-memory inference is replaced by persistent state. The `get_dossier_state` tool's implementation flips from "infer from SCOs" to "read from `dossier_slot` table"; the inference logic is preserved as the *backfill* path when a workspace has SCOs but no dossier_slot rows yet.

### M-5 — Denial / Deception Strategies (slot 9) + User-Note Surface

**Scope:** Slot 9 is user-authored or agent-suggested. Introduce a `dossier note` meta-command and an LLM tool `add_dossier_strategy` that links a strategy note to specific evidence pointers from slots 1–8.

**Why fifth:** Depends on persistent state (M-4) for evidence linkage. Independent of M-6 visualization upgrades.

**Slice size:** Medium.

### M-6 — Dossier-Aware Auto-Pivot Policy

**Scope:** Extend F60's auto-pivot policy engine (`core/pivot_policy.py`) to consume dossier state. The policy's "should we pivot from indicator X?" question gains a fourth input: "would this pivot fill an empty high-value dossier slot?" The existing three-gate structure (IOC filter + confidence gate + per-cascade budget + dry-run) is preserved; the budget formula expands to give higher budget to pivots that fill higher-weighted empty slots.

**Why sixth:** Depends on M-4 persistent state (the policy needs to query slot fill state). Sequenced after M-5 because the agent must already understand the full 9-slot vocabulary before the policy engine reasons over it.

**Slice size:** Medium. Touches F60 policy code; preserves DEC-60-PIVOT-POLICY-001..007 invariants. Forbidden shortcut: NO new policy class — extend the existing one.

**Removal targets:** None — F60 invariants stay; the policy gains new inputs, not new authorities.

### M-7 — Reports, Celebrations, Badges Dossier-Aware Upgrade

**Scope:** Existing `gamification/reports.py`, `celebrations.py`, `badges.py` continue to consume `ScoreEvent`s (no surface change). They are upgraded to interpret the new `DossierSlotFilled` and `DossierPredictionValidated` event subtypes specially: reports become "actor dossier" reports keyed to the puzzle metaphor; celebrations get slot-fill-specific flavor; new badges are introduced for "first to fill all 7 raw-evidence slots," "first validated prediction," etc.

**Why seventh:** Depends on M-3 (scoring) and M-4 (predictions log). All three downstream surfaces upgrade together because the LLM narration layer (F64) requires coherent semantic content from all three or none.

**Slice size:** Large. Three subsystems; coordinate to land as one slice to preserve F64 panel-authority invariants.

**Removal targets:** The "investigation report" template is replaced by the "actor dossier report" template. v1's interview-based report (DEC-AGENT-REPORT-* in Phase 6) continues to be available as a `--style classic` flag for one release cycle, then removed in M-8 cleanup (DEC-68-DOSSIER-REFRAME-008).

### M-8 — Cleanup, Closeout, and Achievement-Mechanism for Novel Methods

**Scope:** Retire the v1 "classic" report template flag (one-release-cycle deprecation per M-7). Implement the issue #68 bonus-space ask: "the achievement system needs an explicit slot for *recognizing* a novel method when AP's agent or user executes one." This is the meta-achievement layer — not a preset badge for a known pattern, but a mechanism that flags when an investigation produced a *new* combination of slot fills that no prior session has produced. Likely a `dossier/novelty.py` module that hashes (slot, evidence-extractor, ordering) tuples and compares against a global novelty cache in `~/.ap/dossier_novelty.sqlite`.

**Why eighth:** Lowest-priority new surface; depends on the full 9-slot picture being persisted (M-4) and presentational coherence (M-7). The cleanup is the closeout — every prior slice's removal-targets discipline catches up here.

**Slice size:** Medium.

**Removal targets:** v1 "classic" report flag deleted. Any per-slot temporary scaffolding deleted.

---

### Sequencing Rationale (one-line per slice)

| Slice | Unblocks | Blocked by | Blast radius |
|-------|----------|------------|--------------|
| M-1 (MVP panel) | M-2, M-3, M-6, M-7 (validates schema) | nothing | Low (read-side only) |
| M-2 (module→slot map) | M-3, M-6 | M-1 | Low (per-module additive) |
| M-3 (dossier scoring) | M-4, M-7 | M-2 | High (scoring weights) |
| M-4 (persistent state + predictions) | M-5, M-6, M-7 | M-3 | High (new SQLite tables) |
| M-5 (denial strategies) | M-7 | M-4 | Low (user-note surface) |
| M-6 (dossier-aware auto-pivot) | M-7 | M-4 | Medium (F60 policy extension) |
| M-7 (reports/celebrations/badges upgrade) | M-8 | M-3, M-4, M-5, M-6 | Medium (three subsystems) |
| M-8 (cleanup + novelty layer) | nothing (closeout) | M-7 | Low (cleanup + new opt-in surface) |

**Critical path:** M-1 → M-2 → M-3 → M-4 → M-7 → M-8. Max width: M-5 and M-6 can land in parallel after M-4, before M-7.

---

## 6. Disposition of Related Open Issues (#29, #30, #31, #32)

Each related issue is dispositioned as **supersede**, **augment**, **sequence-within**, or **stays-independent**. Decisions are binding (DEC-IDs below).

### #29 — Structured Analysis Integration (18 SATs as agent capabilities)

**Disposition:** **Sequence-within the dossier roadmap** as the analyst-step interpretation layer that converts raw SCO evidence into dossier slot values. The 18 Structured Analytic Techniques are *exactly* the analyst-step interpretation that turns Shodan banner noise into "Infrastructure habits" slot evidence, timeline data into "Timing/behavioral" inference, etc.

**Re-aim:** #29 was originally framed as a generic agent capability layer. Re-aim it as the dossier extraction-and-confidence engine. SATs become slot-extractor enrichers — Analysis of Competing Hypotheses (ACH) becomes the M-3 confidence-resolution mechanic; Key Assumptions Check becomes a Predictions Log (slot 8) sanity gate.

**Practical:** #29's body comment ("into the bones") aligns with the M-2 (module→slot mapping) and M-3 (scoring) slices. Schedule #29 as a *sub-issue* of M-3, not as a parallel work item. The SAT library lands as `dossier/sats/` rather than as a generic agent-capabilities module.

**DEC:** DEC-68-DOSSIER-REFRAME-003.

### #30 — Character System v2 (LLM personas with RPG personality)

**Disposition:** **Stays independent** of the dossier reframe.

**Rationale:** Character mode controls the *flavor of presentation* (Bobby Hill mode says "that's my purse!"; Sun Tzu mode quotes The Art of War). It does NOT shape the dossier slot schema, the scoring formula, or the analyst-step interpretation. The persona is orthogonal: a Bobby Hill-mode user fills the same dossier slots a Columbo-mode user fills; only the chat flavor differs.

**Honored re-aim from #68's body:** issue #68 noted "currently about the *player's* agent persona. Re-scope to also cover the *target* persona model." This is honored — the **target** persona (the threat actor being dossiered) IS the central artifact of v2, but it is owned by the dossier slot schema (especially slots 1, 6, 7), NOT by the character-mode system. The dossier IS the target persona. #30's player-persona surface stays independent because they answer different questions.

**Practical:** #30 ships when it's ready. No dossier-roadmap dependency in either direction.

**DEC:** DEC-68-DOSSIER-REFRAME-004.

### #31 — RPG Gamification v2 (XP, leveling, skill trees, quests)

**Disposition:** **Retired (close as superseded by #68)**.

**Rationale:** This is the most consequential of the four dispositions. #31's framing — "transform scoring into a full RPG progression system" with XP, levels, skill trees, loot drops, quests — is the *old* "indicator-pivot rewards mechanical effort" frame writ large. RPG level-grinding is exactly the kind of reward-the-volume mechanic that the dossier reframe is designed to replace. Two examples of the friction:

- *Skill trees (OSINT/CTI/Analysis specializations).* The dossier reframe values **what you've learned about an actor**, not **which specialization branch you've grinded XP into**. A "Threat Hunter" persona who exclusively works financial-crime actors should not have an empty "Analysis" skill tree — their domain expertise is captured in their dossier corpus, not in a tree gated by activity volume.
- *Loot drops (rare findings = rare loot).* This *almost* aligns with dossier rarity weighting (§3 slot weights), but the "loot" framing implies a *transactional reward* (you got X, now use it on Y). The dossier frame says the rare finding *is itself the reward* because it fills a high-value slot. Loot is the wrong metaphor.

The single piece of #31 that survives in spirit is the **quest system (multi-step investigations)** — but it survives natively in the dossier roadmap as "fill a specific high-value empty slot for actor X" as an investigation goal. M-3 and M-7 collectively own the quest-shape replacement.

**Practical:** Close #31 with a comment pointing at the M-3/M-7 disposition and at issue #68. Do not reuse the XP/leveling/skill-tree/loot vocabulary in any v2 work item; if a future user idea echoes that vocabulary, it gets a fresh issue and a fresh planner pass.

**DEC:** DEC-68-DOSSIER-REFRAME-005. **The single most assertive call in this scoping.** Worth defending in user review because it closes an open user-filed issue.

### #32 — LLM-Enhanced Celebrations and Narrative Reports

**Disposition:** **Augment via sequence-within** the dossier roadmap as part of M-7.

**Rationale:** Celebrations and reports become dramatically more interesting under the dossier reframe — "Identity slot just upgraded from medium to high confidence — here's an LLM-generated paragraph summarizing what you now know about this actor and what evidence corroborated the identity" is exactly the kind of LLM-narration the issue #32 framing envisions. The dossier reframe gives #32 a *substrate* (slot state + evidence pointers) it didn't have under v1.

**Practical:** #32 becomes a sub-issue of M-7. The LLM celebration upgrade is gated on M-3 (so there's a `DossierSlotFilled` event to celebrate) and M-4 (so there's persistent slot state for the narrative to reference). The v1 ASCII-art celebrations remain — they fire for routine module-call events at baseline weight 1.0; LLM-narrated celebrations fire for high-weight slot-fill events. F64 LLM-narration de-duplication invariants are honored: the panel still owns the gamification surface; the LLM narration is the *content* surfaced by the panel, not a parallel surface.

**DEC:** DEC-68-DOSSIER-REFRAME-006.

### Disposition summary table

| Issue | Title | Disposition | Action | DEC |
|-------|-------|-------------|--------|-----|
| #29 | 18 SATs as agent capabilities | Sequence-within (M-2/M-3) | Add a comment re-aiming #29 to dossier extraction/confidence engine; relabel as `dossier-roadmap` | DEC-68-DOSSIER-REFRAME-003 |
| #30 | Character v2 LLM personas | Stays independent | Add a comment clarifying #30 ≠ #68 (player persona vs target persona); leave open with no dossier dependency | DEC-68-DOSSIER-REFRAME-004 |
| #31 | RPG gamification v2 (XP/levels/skill trees/loot/quests) | **Retired** | Close #31 with supersession comment pointing at M-3, M-7, and DEC-68-DOSSIER-REFRAME-005 | DEC-68-DOSSIER-REFRAME-005 |
| #32 | LLM-enhanced celebrations + reports | Augment via M-7 | Add a comment re-scoping #32 as M-7 sub-issue; gated on M-3 + M-4 | DEC-68-DOSSIER-REFRAME-006 |

---

## 7. Original Intent Crowdsourcing / Competition / Career-Development Axis

Per the 2026-05-26 Reckoning Section VIII item 4: *"The Original Intent's crowdsourcing / competition / career-development axis has fully fallen off. Zero issues filed. Zero in-plan presence. … This is a *Future Self promise* that has gone latent for 7+ weeks without being explicitly retired or scheduled."*

**Decision (DEC-68-DOSSIER-REFRAME-009): Schedule it as a v0.3.0+ slice within the dossier roadmap as M-9 (Crowdsourced Dossier Comparison + Public Actor Library).** Do NOT retire as a Non-Goal.

**Rationale:** The dossier reframe makes the crowdsourcing axis architecturally tractable in a way it was not under v1's indicator-pivot frame:

- A dossier is a *bounded, comparable artifact*. Two AP users investigating the same actor can independently produce dossiers that can be compared slot-by-slot. v1's indicator-graph snapshots are not similarly comparable.
- *Career development* in OSINT/CTI is measured by **what you've learned about which actors** — exactly what the dossier captures. A user's dossier corpus IS their analyst portfolio. This was implicit in the Original Intent and is now explicit in the dossier frame.
- *Competition* under the dossier frame is "who filled the most high-value slots for actor X first" or "whose Predictions Log has the highest validated-prediction ratio" — concrete, measurable, and grounded in real analytic work rather than mechanical activity volume (which v1's competition framing would have rewarded).
- *Crowdsourcing* under the dossier frame is "publish your dossier for actor X to a shared library; consume others' dossiers as priors when starting your own investigation." This naturally rides on STIX 2.1 (ADR-005) — dossiers are STIX bundles plus metadata.

**M-9 placement:** Scheduled post-M-8 as v0.3.0+. Not in the immediate critical path. Explicitly *scheduled*, not *latent indefinitely* (the Reckoning's named anti-pattern).

**Why not retire as a Non-Goal:** The Original Intent named it. The dossier reframe makes it tractable. v1 deferred it for solo-developer sustainability, but v2's substrate enables it without requiring real-time collaboration or federation infrastructure (those remain v1 Non-Goals). Retiring it would contradict the Reckoning's framing of the dossier reframe as "Original Intent's voice clarifying itself." Scheduling it is the action that resolves the latency.

**Practical:** File a tracker GitHub issue under label `dossier-roadmap` titled "M-9 Crowdsourced Dossier Comparison + Public Actor Library" with scope "STIX-bundle-based dossier export/import; comparison metric; opt-in public-library publication." Do not start work; this is the scheduling artifact.

---

## 8. Decision Log (binding)

These DEC-IDs are binding for the W-68-DOSSIER-REFRAME-SCOPING workflow and bind subsequent implementer slices (M-1 through M-9). They are also written into MASTER_PLAN.md §16 Decision Log.

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-68-DOSSIER-REFRAME-001** | The dossier-puzzle metaphor is ratified as v2's product center. AP's analytic unit of value shifts from indicator-graph traversal to Threat Actor Dossier completion. | Five independently sufficient reasons (§2.1–2.5): closer to Original Intent than v1 trajectory; serves Principle 4 more cleanly; resolves a latent tension in the existing gamification stack; absorbs Threat Hunter expert pressure at the *value* layer; architecturally cheap given ADR-010's separation discipline. |
| **DEC-68-DOSSIER-REFRAME-002** | Scoring authority: layer a new `dossier/` aggregator over the existing `ScoringEngine`; emit new `DossierSlotFilled` / `DossierPredictionValidated` event subtypes via the same engine; re-tune per-IOC `MODULE_RUN_SCORED` to baseline weight 1.0. Option (c) over (a) replace and (b) augment. | Single new authority + single existing authority, each owning a distinct question. Preserves F62/F63/F64 invariants. Honors Sacred Practice 12 by avoiding any permanent dual-authority window. No deprecation shim, no fallback flag. |
| **DEC-68-DOSSIER-REFRAME-003** | Issue #29 (18 SATs) is sequenced *within* the dossier roadmap as the analyst-step interpretation engine that produces slot values from raw SCO evidence. SAT library lands under `dossier/sats/`, not as a parallel agent-capability module. | SATs are exactly the analyst-step interpretation the dossier extraction layer needs. ACH becomes the M-3 confidence-resolution mechanic; Key Assumptions Check becomes the slot 8 Predictions Log sanity gate. Avoids two parallel "analyst capabilities" surfaces. |
| **DEC-68-DOSSIER-REFRAME-004** | Issue #30 (character v2 personas) stays independent. The player persona (#30) and the target persona (dossier slots 1, 6, 7) answer different questions. The dossier IS the target persona; #30 governs presentation flavor. | Orthogonality test: a Bobby Hill-mode user fills the same dossier slots a Columbo-mode user fills. Persona affects presentation, not analytic state. No dossier-roadmap dependency in either direction. |
| **DEC-68-DOSSIER-REFRAME-005** | Issue #31 (RPG gamification v2: XP, levels, skill trees, loot, quests) is **retired** as superseded by #68. Close #31 with a supersession comment pointing at M-3, M-7, and this DEC. The "quest" piece survives in spirit as "fill a high-value empty slot for actor X" under M-3/M-7; it does NOT survive as a quest *subsystem*. | RPG level-grinding rewards activity volume — exactly the v1 frame the dossier reframe replaces. Skill trees gate progression by mechanical specialization rather than by what's been learned about actors. Loot is the wrong metaphor: the rare finding *is itself* the reward because it fills a high-value slot. Keeping #31 open would create permanent friction against the dossier scoring model. |
| **DEC-68-DOSSIER-REFRAME-006** | Issue #32 (LLM-enhanced celebrations + reports) augments the dossier roadmap as M-7 sub-issue. Gated on M-3 (so there's `DossierSlotFilled` content to narrate) and M-4 (so there's persistent slot state for narrative reference). Honors F64 panel-authority invariants — LLM narration is the *content* of panel events, not a parallel surface. | The dossier reframe gives #32 a substrate it didn't have under v1's per-IOC framing. v1 ASCII-art celebrations remain for routine baseline events; LLM-narrated celebrations fire for high-weight slot-fill events. |
| **DEC-68-DOSSIER-REFRAME-007** | Predictions Log slot (slot 8): a falsified prediction contributes 0 to slot weight (not negative). Whether falsified predictions should *deduct* score is **deferred to M-3 implementer slice**. | Deferred not because the question is unimportant but because the right answer depends on whether negative-score events break F62 streak invariants or F63 milestone catch-up math. M-3 implementer evaluates and decides; planner re-stages only if M-3 surfaces a violation of an existing DEC. |
| **DEC-68-DOSSIER-REFRAME-008** | v1's interview-based investigation report (DEC-AGENT-REPORT-* under Phase 6) is replaced by the actor-dossier report at M-7. The v1 template is available via `--style classic` flag for one release cycle (v0.2.x), then removed at M-8 cleanup. | One-release-cycle deprecation is the minimum window needed for a single user to consume the change without surprise; longer windows would create the parallel-authority residue Sacred Practice 12 forbids. M-8 is the named removal point so the cleanup is not optional. |
| **DEC-68-DOSSIER-REFRAME-009** | The Original Intent crowdsourcing / competition / career-development axis is **scheduled as M-9** (Crowdsourced Dossier Comparison + Public Actor Library, v0.3.0+), NOT retired as a Non-Goal. | The dossier reframe makes the axis architecturally tractable (dossiers are STIX-bundle-comparable; career = dossier corpus; competition = slot-fill speed / validated-prediction ratio). Retiring it would contradict the Reckoning's framing of the dossier reframe as "Original Intent's voice clarifying itself." Scheduling it is the action that resolves the 7-week latency. |
| **DEC-68-DOSSIER-REFRAME-010** | The dossier slot schema v1.0 (§3, nine slots) is binding for M-1 with the explicit exception that M-1 may refine the schema by ±2 slots before the first implementer touches code. Further refinement (post-M-1) requires a planner re-stage and a successor DEC-ID. | Locking the schema before MVP contact with real data would be premature; locking it forever after MVP would be brittle. The ±2-slot window during M-1 lets the MVP validate the metaphor; the successor-DEC-ID gate ensures subsequent refinements are tracked. |

---

## 9. Out-of-Scope (planner asserts; implementer slices honor)

- **No source code changes in W-68-DOSSIER-REFRAME-SCOPING.** This workflow is plan-only. The deliverable is the plan itself plus the MASTER_PLAN.md §16 amendment.
- **No new modules.** The dossier reframe is an *aggregation* layer over existing modules; no new module sources are needed for M-1 through M-8. M-9 may incentivize new modules but does not require them.
- **No federation infrastructure** beyond STIX-bundle file export/import. v1 Non-Goal "Federation between AP instances" continues to bind for M-1 through M-8. M-9 may revisit but defaults to opt-in file-based sharing.
- **No real-time multi-user collaboration.** v1 Non-Goal continues to bind through M-9.
- **No DALL-E / AI-generated celebration images.** v1 Non-Goal continues to bind. LLM-narrated text celebrations (M-7) are explicitly text-only.
- **No web/GUI surface.** v1 Non-Goal continues to bind. M-1 visualization lands in the existing `ap chat` Rich panel surface.
- **No MCP migration (issue #65) dependency.** The dossier reframe is orthogonal to #65; both can land independently in either order. If #65 lands first, dossier slot-fill events fire from MCP-served modules just as well as from in-process modules.

---

## 10. Cross-References

- **MASTER_PLAN.md Phase 16** — the binding planner section that cites this document.
- **Issue #68** — source product directive (https://github.com/jarocki/ap/issues/68).
- **`reckonings/2026-05-26-reckoning.md`** — the reckoning that surfaced #68's urgency (Sections II, VII, VIII).
- **`src/adversary_pursuit/gamification/scoring.py`** — current scoring authority (will be re-tuned by M-3; not removed).
- **`src/adversary_pursuit/core/workspace.py`** — current SCO persistence authority (extended by M-4; not displaced).
- **`src/adversary_pursuit/core/event_bus.py`** + **`pivot_policy.py`** — F19/F60 auto-pivot subsystem (extended by M-6; F60 DEC-001..007 preserved).
- **DEC-EVENTBUS-002**, **DEC-WS-004**, **DEC-STIX-001**, **DEC-59-STIX-PROVENANCE-001..007**, **DEC-60-PIVOT-POLICY-001..007**, **DEC-62-STREAK-*** , **DEC-63-MILESTONE-***, **DEC-64-LLM-PANEL-SEPARATION-001** — all preserved; none superseded by the dossier reframe.

**Referenced-but-not-found:** The four related issues (#29, #30, #31, #32) cite `.claude/plans/shimmying-yawning-shamir.md` Phases B/C/D. That file does not exist in the repository (confirmed 2026-05-27 via planner search of both worktree and parent repo). The issue bodies' references to it are unfulfilled internal pointers. This scoping document does NOT depend on `shimmying-yawning-shamir.md` content; the four dispositions in §6 are derived from each issue's own body and from the dossier reframe in §1.
