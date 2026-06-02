# M-3 — Dossier Scoring + Score Event Re-Tune (per-slice plan)

**Status:** planner-staged 2026-06-01 by W-68-M3-DOSSIER-SCORING planner stage. Implementer slice to follow.
**Workflow:** `w-68-m3-dossier-scoring`
**Goal:** `g-68-m3-dossier-scoring`
**Work item to dispatch:** `wi-68-m3-impl-01`
**Drives:** Phase 17F of `MASTER_PLAN.md`. Phase 17F carries the binding decisions and the slice index; this document carries the full rationale, event-emission semantics, re-tune table, hook-site diff sketches, and decomposition detail. When the two diverge, Phase 17F wins for binding decisions; this document wins for narrative and table detail.

**Inherits from:** Phase 16 §M-3, `.claude/plans/dossier-reframe-v2-roadmap.md` §4 (DEC-68-DOSSIER-REFRAME-002 option c), §3 (slot weights), §M-3 (re-tune mandate). Phase 17B (M-1 panel) + Phase 17D (M-2 extractors) are prerequisites; both shipped 2026-05-28/29.

---

## 1. Goal (single paragraph)

Wire **dossier slot completion** into the score economy per DEC-68-DOSSIER-REFRAME-002 (option c). When a hunt fills a dossier slot (Identity / TTPs / Infrastructure / Timing / Capability / Motivation status moves `empty → partial` or `partial → filled`, per the 9-slot weights from Phase 16 §3 — Identity=5.0, Predictions=4.0, Capability=3.5, TTPs=3.0, Motivation=3.0, Targeting=2.5, Denial=2.5, Infrastructure=2.0, Timing=2.0, baseline IOC=1.0), a `dossier_slot_filled` `ScoreEvent` fires with the weighted point value. The user sees `Identity slot filled +5 points!` (or similar) instead of just `IP found +1`. After M-3, the dossier IS the score economy's center of gravity: per-IOC events drop to baseline 1.0, and slot weights dominate.

**Out-of-scope (deferred):**
- Persistent dossier state / `dossier_prediction` SQLite tables — **M-4** owns.
- Auto-validation of `DossierPredictionValidated` from later evidence — **M-4** owns (M-3 scaffolds the event shape only).
- Denial / Deception strategies slot fills — **M-5** owns the user-note authoring surface.
- Dossier-aware auto-pivot policy budget formula — **M-6** owns.
- Reports / celebrations / badges narrative upgrades — **M-7** owns.

---

## 2. Architecture

### 2.1 Layering authority (DEC-68-DOSSIER-REFRAME-002 honored)

```
+-----------------------------------------------------------+
|  Caller: agent/tools.py::run_module + core/console.py::   |
|          _execute_hunt                                    |
|   1. pre = infer_dossier_state_full(scos_before, runs_b,  |
|             notes_b)                                      |
|   2. store_stix_objects(results, ...)                     |
|   3. ScoringEngine.score_results(results, stats)  (1.0)   |
|   4. post = infer_dossier_state_full(scos_after, runs_a,  |
|             notes_a)                                      |
|   5. dossier_events = dossier/scoring.py                  |
|         .emit_dossier_slot_filled_events(pre, post)       |
|   6. workspace_mgr.store_score_events(per_ioc_events +    |
|             dossier_events)                               |
|   7. existing F62/F63 streak/milestone flow on            |
|         workspace.get_total_score() now sees the higher   |
|         total                                             |
+-----------------------------------------------------------+
```

- `dossier/scoring.py` is **a new pure function module**. It owns one question: *given a `pre` and `post` `DossierState`, which slot transitions occurred and what score events do they imply?* It does NOT subscribe to events, mutate workspace, or emit anything directly — it returns `list[dict]` for the caller to persist via the existing `workspace_mgr.store_score_events(...)` path.
- `gamification/scoring.py` `ScoringEngine` is unchanged in *behavior*. Only the per-IOC `DEFAULT_RULES` constants are re-tuned (see §4).
- `core/workspace.py` is UNTOUCHED. The existing `store_score_events(list[dict], module_run_id=None) -> int` accepts the new event dicts as-is — same `{action, points, indicator, rule_description}` shape.
- `core/streak.py` is UNTOUCHED. F62 streak invariants preserved.
- `dossier/slot_inference.py` is UNTOUCHED (M-2 byte-identical). M-3 only READS `infer_dossier_state_full()`'s output at two snapshot points per hunt.
- `dossier/slots.py` is UNTOUCHED. `SLOT_WEIGHTS` is the single weight authority — M-3 reads it; the constants themselves don't change.

### 2.2 Why pure function, not subscriber

Two architecturally simpler alternatives were rejected:

**Rejected (a) — `Dossier` aggregator subscribes to `event_bus`.** `event_bus.py` is F60 territory (pivot policy). Adding a new subscriber there would (i) violate the M-3 forbidden list, (ii) couple scoring to F60 cascade behavior (a dry-run cascade would skip emissions), (iii) introduce a parallel event-emission path. DEC-68-DOSSIER-REFRAME-002 explicitly chose "layer over scoring" not "extend event bus."

**Rejected (b) — `ScoringEngine.score_results` itself computes pre/post and emits dossier events.** Forces `ScoringEngine` to take `DossierState` inputs (foreign concern) AND to read workspace state (it currently takes `workspace_stats: dict[str, int]` only — does not know about SCOs, module_runs, or notes). Would also break the F62 streak path where `make_streak_continued_event` is a standalone helper. Keeping `ScoringEngine` ignorant of dossier preserves Sacred Practice 12.

**Selected (c) — pure-function aggregator, caller wires snapshots.** `dossier/scoring.py::emit_dossier_slot_filled_events(pre, post)` is a deterministic transformation. The two callers (`run_module`, `_execute_hunt`) already own the `before-hunt / after-hunt` boundary — they are the natural snapshot sites. The function returns events; the caller persists them through the established `store_score_events` API. No new authority, no new subscriber, no new mutator.

---

## 3. Event Vocabulary (M-3 additions)

### 3.1 `dossier_slot_filled` (PRIMARY)

Fires when a slot's `SlotStatus` transitions upward between `pre` and `post`:

| from | to | event fires |
|------|-----|-------------|
| `empty` | `partial` | YES |
| `empty` | `filled` | YES (single event; skip-step) |
| `partial` | `filled` | YES |
| `filled` | (any) | NO (idempotent — already filled) |
| `partial` | `partial` | NO (no transition) |
| `empty` | `empty` | NO (no transition) |
| `*` | `deferred` | NO (DEFERRED is a milestone marker, not a real status; cannot transition INTO it) |
| `deferred` | `partial`/`filled` | NO in M-3 — only Predictions/Denial are deferred-only in M-2 (slot inference for them lands in M-4/M-5). If M-3 sees a `deferred → real` transition, it logs a debug message and skips (defensive — guards against future inference changes). |

**Event dict shape** (persisted via `workspace_mgr.store_score_events(...)`):

```python
{
    "action": "dossier_slot_filled",                # canonical action key
    "points": int(weight),                          # e.g. 5 for Identity, 3 for TTPs
    "indicator": "<slot_value>",                    # e.g. "identity", "ttps" (DossierSlotName value)
    "rule_description": "Dossier slot filled: <Slot Display Name> (<from> -> <to>)",
}
```

**Points calculation:**
- A single transition (`empty → partial` OR `partial → filled`) awards `int(SLOT_WEIGHTS[slot])` points.
- A skip-step transition (`empty → filled` in one hunt) **emits ONE event** and awards `int(SLOT_WEIGHTS[slot])` points (NOT 2× — the user crossed two thresholds, but the slot is filled once; double-counting would let a single high-volume hunt double-bill).
- `int()` is the floor (Identity=5.0 → 5; Capability=3.5 → 3). Weights stay floats in `SLOT_WEIGHTS` for future M-4+ confidence-multiplier work, but `ScoreEvent.points` is an integer column. The floor is honest and stable across F62/F63 integer arithmetic.

**Idempotency:** the transition detector IS the idempotency mechanism. A slot that's already `filled` cannot transition upward; therefore no event fires. No external "already-fired" cache, no SQLite "dossier_slot_filled_events" table.

**Emission order within a hunt:** dossier events fire AFTER per-IOC `score_results` events but BEFORE `streak_continued` (F62). This means:
1. Per-IOC events (re-tuned baseline) → `score_events` table
2. Dossier events (high-weight) → `score_events` table
3. Streak event (if any) → `score_events` table

This ordering keeps `workspace.get_total_score()` monotonically increasing across the hunt and lets the F63 milestone-catch-up read post-total once at the end (existing `last_milestone_announced` sentinel + `check_milestones(post_total, last_id)` logic UNCHANGED).

### 3.2 `dossier_prediction_validated` (SCAFFOLD ONLY — M-4 plugs in)

Per DEC-68-DOSSIER-REFRAME-002, the v2 vision includes prediction-validated as a high-value event. But the Predictions slot is `DEFERRED` in M-2 (DEC-M2-DOSSIER-004) and lands real inference at M-4 (persistent state). M-3 scaffolds:

- The event shape:
  ```python
  {
      "action": "dossier_prediction_validated",
      "points": int(SLOT_WEIGHTS[DossierSlotName.PREDICTIONS]),  # 4
      "indicator": "<prediction_id_or_text_hash>",                # M-4 will use a real prediction_id
      "rule_description": "Dossier prediction validated by later evidence",
  }
  ```
- A pure function: `emit_dossier_prediction_validated_event(prediction: PredictionRecord) -> dict` (returns the event dict above; not called anywhere in M-3 — M-4 plumbs it in).
- A test scaffold asserting the function exists, returns the documented shape, and uses the documented weight.

**M-3 forbidden:** no caller wires this in (no transitions exist because Predictions slot remains `DEFERRED`). Any attempt to fire it during a real M-3 hunt is a planner gap.

### 3.3 Existing per-IOC events (re-tuned, not removed)

`new_ip`, `new_domain`, `new_url`, `new_email`, `adversary_mistake`, `deception_uncovered`, `adversary_linked`, `new_tool`, `campaign_described` remain in `DEFAULT_RULES`. Their action keys, dict shape, and emission site (`ScoringEngine.score_results`) are UNCHANGED. Only the `initial` / `minimum` / `decay` constants are re-tuned (§4).

`streak_continued` (F62/F63) is UNCHANGED — DEC-63-STREAK-SCORE-001 step-decay (10/5/2) preserved. Streak rewards engagement time-relevance, not analytic value; the M-3 re-tune does not touch it.

---

## 4. Per-IOC Score Event Re-Tune Table

Per DEC-68-DOSSIER-REFRAME-002: "re-tune per-IOC `MODULE_RUN_SCORED` to baseline weight 1.0." AP's scoring engine does not have a literal `MODULE_RUN_SCORED` enum (action keys are strings in `ScoringRule.action`); the equivalent is to re-tune `DEFAULT_RULES` so that **`initial == minimum == 1`** for the per-IOC SCO-mapped types. The parabolic decay floor and ceiling collapse to 1; every per-IOC event is worth exactly 1 point regardless of solve_count.

This is the *one-time* re-tune the roadmap §4 names ("Score-event weight retuning is required: old per-IOC events drop to weight 1.0 while new slot-fill events scale up to 2.0–5.0. This is a *one-time* re-tune at the M-3 slice, not an ongoing dual-surface").

| action | v1 initial | v1 minimum | v1 decay | M-3 initial | M-3 minimum | M-3 decay | rationale |
|--------|-----------|-----------|---------|-------------|-------------|-----------|-----------|
| `new_ip` | 100 | 10 | 10 | **1** | **1** | 10 | per-IOC baseline; dossier weight dominates analytic value |
| `new_domain` | 100 | 10 | 10 | **1** | **1** | 10 | per-IOC baseline |
| `new_url` | 50 | 5 | 10 | **1** | **1** | 10 | per-IOC baseline |
| `new_email` | 50 | 5 | 10 | **1** | **1** | 10 | per-IOC baseline |
| `adversary_mistake` | 10 | 5 | 5 | **1** | **1** | 5 | per-IOC baseline; "mistake" semantics now live in Identity/TTPs slot |
| `deception_uncovered` | 200 | 50 | 5 | **1** | **1** | 5 | per-IOC baseline; deception detection now scored via slot evidence |
| `adversary_linked` | 500 | 100 | 3 | **1** | **1** | 3 | per-IOC baseline; *attribution* now scored via Identity slot fill |
| `new_tool` | 500 | 100 | 3 | **1** | **1** | 3 | per-IOC baseline; *tool discovery* now scored via TTPs / Capability slot |
| `campaign_described` | 1000 | 200 | 2 | **1** | **1** | 2 | per-IOC baseline; *campaign* now scored via Identity + Targeting + Motivation slots |
| `streak_continued` | n/a (helper) | n/a | n/a | **UNCHANGED** | **UNCHANGED** | **UNCHANGED** | F62/F63 time-relevance, not analytic value |

**User impact:** Existing workspaces created pre-M-3 will see "lower" per-IOC scores going forward. Their HISTORICAL `score_events` rows are unchanged (the integer column is fixed when written), so `get_total_score()` continues to reflect the v1 scoring rate for old events. NEW per-IOC events under M-3 are worth 1 each. Slot fills add 2–5 each. The net effect for an active investigation that fills several dossier slots is **higher total score**, not lower — the slot fills more than offset the per-IOC drop because the dossier slot weights (5,4,3.5,3,3,2.5,2.5,2,2) are roughly an order of magnitude higher than the new per-IOC floor.

**No backward-compat shim.** Per DEC-68-DOSSIER-REFRAME-002 explicitly: "No flag retained for legacy scoring. No `AP_DOSSIER_DISABLE=1`, no `--no-dossier` CLI flag." The re-tune lands as a one-time constant change.

**Decay constants kept (not collapsed to 1):** `decay` is the parabolic shape parameter. With `initial == minimum`, the formula `((min - init) / decay^2) * count^2 + init = init` always returns `init`; decay becomes mathematically inert. Keeping the existing `decay` values rather than zeroing them makes the re-tune diff smaller and clearer (only the integer ceiling/floor change) and lets future planners reverse the re-tune cleanly if v2 product judgment ever needs it.

---

## 5. Caller Wiring Sites (hook-site diff sketch)

### 5.1 `src/adversary_pursuit/agent/tools.py::run_module` (around line 382-409)

**Current (M-2 byte):**
```python
results = asyncio.run(mod.hunt(target, options or {}))
count = self.workspace_mgr.store_stix_objects(results, module_path, target, ...)
pre_total = self.workspace_mgr.get_total_score()
stats = self.workspace_mgr.get_stix_type_counts()
events = self.scoring.score_results(results, stats)
total = self.scoring.total_score(events)
if events:
    self.workspace_mgr.store_score_events(events)
```

**M-3 inserts (pseudocode — implementer authors the real diff):**
```python
# Capture pre-hunt dossier state BEFORE storing the new SCOs
scos_before = self.workspace_mgr.get_stix_objects()
runs_before = self.workspace_mgr.get_module_runs()
notes_before = _read_analyst_notes(self.workspace_mgr)  # direct-engine helper, see §5.3
pre_dossier = infer_dossier_state_full(scos_before, module_runs=runs_before, notes=notes_before)

results = asyncio.run(mod.hunt(target, options or {}))
count = self.workspace_mgr.store_stix_objects(results, module_path, target, ...)
pre_total = self.workspace_mgr.get_total_score()
stats = self.workspace_mgr.get_stix_type_counts()
events = self.scoring.score_results(results, stats)         # NOTE: events now baseline 1.0 per §4
total = self.scoring.total_score(events)

# Capture post-hunt dossier state AFTER storing the new SCOs
scos_after = self.workspace_mgr.get_stix_objects()
runs_after = self.workspace_mgr.get_module_runs()
# notes haven't changed during the hunt (modules don't write notes)
post_dossier = infer_dossier_state_full(scos_after, module_runs=runs_after, notes=notes_before)

# Emit dossier slot-fill events (M-3 NEW)
dossier_events = emit_dossier_slot_filled_events(pre_dossier, post_dossier)
if dossier_events:
    self.workspace_mgr.store_score_events(dossier_events)
    events = list(events) + dossier_events
    total += sum(e["points"] for e in dossier_events)

if events:
    self.workspace_mgr.store_score_events(events)
    # (events is the per-IOC list ONLY; dossier_events already persisted above —
    #  the implementer MUST ensure no double-persist; see Evaluation Contract §7)
```

**Implementer note — DO NOT double-persist.** The wiring above is one possible reorganization; the implementer may choose to (a) persist per-IOC events first and dossier events second in two `store_score_events` calls, OR (b) persist a single combined list once. Either is acceptable as long as every event lands in `score_events` exactly once and is reflected in `total`. The test gate `test_dossier_event_persisted_exactly_once` enforces this.

### 5.2 `src/adversary_pursuit/core/console.py::_execute_hunt` (around line 430-489)

Mirror of §5.1. The cmd2 console path has its own `scoring_engine.score_results(...)` call and `workspace_mgr.store_score_events(...)` call. Apply the same pre/post snapshot pattern. Per the F62 invariant test, BOTH callers must behave identically on score-event emission ordering.

### 5.3 Notes access helper (private to caller files)

Both `tools.py` and `console.py` need `notes: list[dict]` for `infer_dossier_state_full()`. Per DEC-M2-MOTIVATION-001, the canonical pattern is the direct-engine query in `core/report.py` lines 348-369:

```python
def _read_analyst_notes(workspace_mgr) -> list[dict]:
    """Read analyst notes via direct-engine query (DEC-M2-MOTIVATION-001 pattern).

    Mirrors core/report.py:348-369. workspace.py adds no accessor (F59 invariant).
    Returns empty list on any error (Motivation slot then renders EMPTY — safe default).
    """
    try:
        from sqlalchemy import select
        from sqlalchemy.orm import Session
        from adversary_pursuit.models.database import AnalystNote
        with Session(workspace_mgr._engine) as session:
            rows = session.scalars(select(AnalystNote).order_by(AnalystNote.id)).all()
            return [{"content": r.content} for r in rows]
    except Exception:
        return []
```

This helper lives in `agent/tools.py` AND `core/console.py` (small duplicate function; DRY refactor into a shared module would require touching either `dossier/` or `core/`, both of which need careful scope review and add risk for a 1-slice gain). If the implementer wants a DRY refactor, it must be a follow-up issue (see backlog).

**Forbidden:** adding a `workspace_mgr.get_analyst_notes()` accessor — DEC-M2-MOTIVATION-001 explicitly forbids it (F59 invariant).

---

## 6. F62 / F63 / F64 Invariant Preservation

### F62 — Streak (DEC-62-STREAK-001 .. 007)

- `~/.ap/streak.json` is UNTOUCHED. `StreakManager` write authority preserved.
- `streak_continued` event step-decay (10/5/2 per DEC-63-STREAK-SCORE-001) UNCHANGED.
- Streak emission site is UNCHANGED (`tools.py` line ~564, `console.py` line ~525) — fires after badge/challenge checks per existing F62 wiring.
- Required test: `test_dossier_events_do_not_touch_streak_json` reads `streak.json` byte content before/after a hunt that fills a slot and asserts byte-identical (when no streak transition).
- Required test: `test_streak_continued_unchanged_under_m3` runs the existing F62 streak test fixture under M-3 wiring and asserts emit semantics unchanged.

### F63 — Milestone catch-up (DEC-63-MILESTONE-CATCHUP-001 .. DEC-63-MIGRATION-001)

- `check_milestones(post_total, last_id)` continues to use `last_milestone_announced` sentinel + cross-threshold detection.
- Dossier events ADD to `post_total` (via `store_score_events` → `get_total_score`). They DO NOT bypass milestones; they trigger them when the new total crosses a threshold.
- Required test: `test_dossier_event_can_trigger_milestone` — synthetic fixture where a single Identity-slot fill (5 points) crosses the 5-point or 10-point milestone (whichever is the next configured threshold per `CelebrationEngine.milestone_list()`).
- Required test: `test_milestone_seed_unchanged_with_dossier_events` — quiet-start migration (`last_id is None and pre_total > 0`) still seeds from `pre_total` (NOT `post_total` — DEC-63-MIGRATION-001 invariant).

### F64 — LLM / Rich panel separation (DEC-64-LLM-PANEL-SEPARATION-001)

- Dossier slot-fill events ARE included in the `events` list returned to the LLM as `score_events` (so the agent can reason about them, e.g., to plan the next pivot).
- Dossier slot-fill text DOES NOT appear in the `summary` string returned to the LLM. The summary mirrors today's existing pattern (lists per-IOC actions). New text like "Identity slot filled +5 points!" is FORBIDDEN in `summary` — the Rich panel surface owns gamification narration.
- Required test: `test_dossier_event_not_in_llm_summary` — feed a slot-fill scenario through `run_module`, assert no occurrence of substrings `"slot filled"`, `"dossier_slot_filled"`, slot display names ("Identity", "TTPs", etc.) in the returned `summary`.
- Required test: `test_dossier_event_is_in_score_events_sidecar` — same scenario, assert the event IS in `result["score_events"]`.

---

## 7. Evaluation Contract (binding for `wi-68-m3-impl-01`)

### Required tests (~28–32 tests total)

#### A. `tests/test_dossier_scoring.py` (NEW; ~16 tests)
Core unit tests for `dossier/scoring.py::emit_dossier_slot_filled_events`:

1. `test_empty_to_partial_identity_emits_one_event` — Identity slot transitions `empty → partial`; one event with `points == 5`, `indicator == "identity"`, `action == "dossier_slot_filled"`.
2. `test_empty_to_partial_ttps_emits_one_event` — TTPs slot, points == 3.
3. `test_empty_to_partial_infrastructure` — Infrastructure, points == 2.
4. `test_empty_to_partial_timing` — Timing, points == 2.
5. `test_empty_to_partial_capability` — Capability, points == 3 (int(3.5) floor).
6. `test_empty_to_partial_motivation` — Motivation, points == 3.
7. `test_partial_to_filled_identity` — Identity `partial → filled`, points == 5 (one event per transition, single threshold).
8. `test_skip_step_empty_to_filled_identity_one_event` — Identity `empty → filled` in one hunt; ONE event (not two), points == 5.
9. `test_filled_to_filled_no_event` — idempotency: slot already filled; zero events.
10. `test_partial_to_partial_no_event` — no transition; zero events.
11. `test_empty_to_empty_no_event` — no transition; zero events.
12. `test_deferred_target_no_event` — Predictions/Denial slots stay DEFERRED in M-3 inference; no transitions fire.
13. `test_deferred_to_real_status_skipped_with_debug_log` — defensive: if a future inference change causes `deferred → partial`, the function logs and skips (no event).
14. `test_multiple_slot_transitions_in_one_hunt` — `pre` has 3 empty slots; `post` has 2 of them partial and 1 filled (skip-step); function returns exactly 3 events with the right slot indicators.
15. `test_event_dict_shape_contract` — every returned event has exactly the 4 documented keys (`action`, `points`, `indicator`, `rule_description`) and types (str, int, str, str).
16. `test_emit_prediction_validated_scaffold_exists` — `emit_dossier_prediction_validated_event(prediction)` is importable, returns the documented shape with `action == "dossier_prediction_validated"`, `points == 4`, `rule_description` non-empty. (SCAFFOLD GATE — DEC-M3-DOSSIER-005.)

#### B. `tests/test_dossier_slot_inference.py` (extend; ~2 tests)
Transition-readiness tests (slot inference itself unchanged — these tests assert the M-2 inference output IS comparable in the way M-3 caller wiring requires):

17. `test_dossier_state_equality_on_identical_inputs` — calling `infer_dossier_state_full` twice with identical inputs returns slots that are `==` (frozen dataclass equality). M-3 relies on this for transition detection.
18. `test_dossier_state_inequality_after_sco_addition` — adding one new identity-class SCO to the input flips the Identity slot's `status` field, so `pre.slots[IDENTITY].status != post.slots[IDENTITY].status` is detectable.

#### C. `tests/test_scoring.py` (extend; ~5 tests)
Per-IOC re-tune assertions:

19. `test_new_ip_initial_is_one_post_m3` — `next(r for r in DEFAULT_RULES if r.action == "new_ip").initial == 1`.
20. `test_new_ip_minimum_is_one_post_m3` — `.minimum == 1`.
21. `test_all_per_ioc_rules_initial_one` — iterate `DEFAULT_RULES`; assert every action key from §4 row 1–9 has `initial == 1 and minimum == 1`.
22. `test_decay_constants_preserved_post_m3` — assert decay constants UNCHANGED per the table (e.g., `new_ip.decay == 10`).
23. `test_streak_continued_unchanged_post_m3` — `streak_continued_points(1) == 10`, `streak_continued_points(8) == 5`, `streak_continued_points(31) == 2` (F62/F63 step-decay UNCHANGED).

#### D. `tests/test_streak.py` (extend; ~2 tests — regression)
F62 invariants under M-3:

24. `test_streak_json_byte_identical_under_dossier_event_emission` — simulate a dossier slot-fill event emission via `dossier/scoring.py`; assert `~/.ap/streak.json` content is byte-identical before and after (dossier emission MUST NOT touch streak file).
25. `test_streak_continued_emits_after_dossier_in_combined_hunt` — full hunt fixture that fills both an Identity slot AND triggers `streak.incremented == True`; assert the `score_events` table contains both `dossier_slot_filled` and `streak_continued` rows; order may vary but both are present.

#### E. `tests/test_agent_tools.py` AND `tests/test_chat_dossier_metacommand.py` AND/OR a new `tests/test_dossier_scoring_integration.py` (~5 tests; compound)

26. `test_run_module_emits_dossier_slot_filled_for_identity_evidence` — fixture: empty workspace, run a mock module that emits one `email-addr` SCO → assert `result["score_events"]` contains a `dossier_slot_filled` event with `indicator == "identity"` and `points == 5`.
27. `test_run_module_emits_baseline_per_ioc_under_m3` — same fixture; assert per-IOC event for the SCO type has `points == 1`.
28. `test_run_module_total_points_sums_per_ioc_and_dossier` — assert `result["total_points"] == per_ioc_total + dossier_total`.
29. `test_dossier_event_not_in_llm_summary` — assert `result["summary"]` contains NEITHER `"slot filled"` NOR `"dossier_slot_filled"` NOR any of the 9 slot display names (Identity / TTPs / Infrastructure / Timing / Capability / Motivation / Predictions / Denial / Targeting).
30. `test_dossier_event_is_in_score_events_sidecar` — same scenario; `result["score_events"]` HAS the dossier event.

#### F. F63 milestone gate (`tests/test_milestones.py` if exists, else extend `test_chat_dossier_metacommand.py` or `test_agent_tools.py`; ~1 test)

31. `test_dossier_event_can_trigger_milestone` — pre-existing workspace at total_score = (milestone_threshold − 3); fire an Identity slot fill (+5); assert `check_milestones` returns the next milestone (post_total crossed threshold). Quiet-start migration (DEC-63-MIGRATION-001) honored.

### Required evidence (live output)

- Full pytest suite: `pytest -q tests/` → all green (1900+ tests; current baseline is 1984 from C-2 closeout).
- `git diff main -- src/adversary_pursuit/core/workspace.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/dossier/slot_inference.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/dossier/slots.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/dossier/panel.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/gamification/celebrations.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/core/streak.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/core/pivot_policy.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/core/event_bus.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/gamification/modes.py` → exactly empty.
- `git diff main -- src/adversary_pursuit/agent/runner.py` → exactly empty.
- Demo trace (compound integration): a single hunt against a mock identity module shows in `result["score_events"]` exactly one `new_email` event (points=1) AND one `dossier_slot_filled` event (indicator=identity, points=5); `result["total_points"] == 6`; `result["summary"]` contains the IOC line but NOT the slot-fill line.

### Required authority invariants

- **F59** (DEC-59-STIX-PROVENANCE-001): `core/workspace.py` byte-identical against main. Direct-engine `AnalystNote` query helper lives in caller, not in workspace.
- **F60** (DEC-60-PIVOT-POLICY-001..007): `pivot_policy.py` and `event_bus.py` byte-identical. Dossier emission does NOT subscribe to event bus.
- **F62** (DEC-62-STREAK-001..007): `core/streak.py` byte-identical. `streak.json` byte-identical when no streak transition occurs. `streak_continued` event semantics unchanged.
- **F63** (DEC-63-*): `gamification/celebrations.py` byte-identical. Milestone seed-from-`pre_total` quiet-start migration honored.
- **F64** (DEC-64-LLM-PANEL-SEPARATION-001): dossier event text absent from LLM `summary`; present in `score_events` sidecar.
- **Sacred Practice 12**: `dossier/scoring.py` is the sole emitter authority for `dossier_slot_filled` events. `ScoringEngine` is the sole emitter authority for per-IOC events. Workspace is the sole persistence authority for `score_events`. No two modules own the same operational fact.
- **DEC-M1-SLOTS-WEIGHT-AUTHORITY-001**: `SLOT_WEIGHTS` constants in `dossier/slots.py` UNCHANGED. M-3 reads them; does not redefine.

### Required integration points

- `dossier/scoring.py` (NEW pure-function module): `emit_dossier_slot_filled_events(pre, post)`, `emit_dossier_prediction_validated_event(prediction)`.
- `dossier/__init__.py`: export new symbols.
- `gamification/scoring.py`: `DEFAULT_RULES` constant re-tune ONLY (no engine logic edits).
- `agent/tools.py::run_module`: pre/post snapshot + emit + persist wiring per §5.1.
- `core/console.py::_execute_hunt`: same pattern per §5.2.

### Forbidden shortcuts (NO drift permitted)

- NO env-var bypass (`AP_DOSSIER_DISABLE`, `AP_LEGACY_SCORING`, etc.) — DEC-68-DOSSIER-REFRAME-002 explicit.
- NO "old scoring fallback" flag, CLI option, or config knob.
- NO new ScoreEvent emission outside the `store_score_events` path.
- NO new event-bus subscriber (F60 invariant).
- NO mutation of `dossier/slot_inference.py`, `dossier/slots.py`, `dossier/panel.py` — M-2 byte-identical.
- NO modification of `core/workspace.py` — F59 / DEC-68 invariant. The direct-engine `AnalystNote` helper lives in caller files.
- NO modification of `core/streak.py` / `gamification/celebrations.py` / `gamification/modes.py` / `agent/runner.py`.
- NO new SQLite tables — M-4 owns persistence.
- NO auto-validation logic for `dossier_prediction_validated` — M-4 owns. M-3 ships the function and scaffold test only.
- NO Rich markup in dossier event text — F64 (`rule_description` is plain ASCII).
- NO double-persist of dossier events to `score_events` table.
- NO refactor of `tools.py` or `console.py` beyond the snapshot + emit wiring (no opportunistic cleanup; the slice is large enough).

### Rollback boundary

Single feature branch revertible as one merge commit. Reverting:
- Restores per-IOC `DEFAULT_RULES` constants (v1 values).
- Removes `dossier/scoring.py` and the `dossier/__init__.py` re-exports.
- Restores `tools.py::run_module` and `console.py::_execute_hunt` to byte-identical M-2 state.
- Score events ALREADY persisted under M-3 (`dossier_slot_filled` rows) remain in `score_events` table — they are historical data; their `action` string is `"dossier_slot_filled"` which under v1 is an unknown action that simply renders as itself in the `score` command. No corrupt state; just historical events with an unfamiliar action label.
- No schema migrations, no settings changes, no per-user file changes (streak.json untouched).

### Ready-for-guardian definition

- All 28–32 tests in §7.A–F green.
- Full suite green (≥1984 passed, 1 skipped — matching baseline; +M-3 new tests added).
- Forbidden file `git diff main` is empty for every entry in the forbidden list.
- `Phase 17F` section appended to `MASTER_PLAN.md` AND committed in the same commit as source.
- `dossier/__init__.py` exports the two new symbols (no surprise additions beyond `emit_dossier_slot_filled_events`, `emit_dossier_prediction_validated_event`).
- Implementer-authored commit message follows the existing `feat(dossier):` Phase 17 prefix and explicitly references `#68` and `DEC-M3-DOSSIER-001..005`.

---

## 8. Scope Manifest (binding)

### Allowed / Required (the implementer MUST touch these)

- `src/adversary_pursuit/dossier/scoring.py` **(NEW)**
- `src/adversary_pursuit/dossier/__init__.py` (export new symbols)
- `src/adversary_pursuit/gamification/scoring.py` (per-IOC re-tune ONLY — `DEFAULT_RULES` constants)
- `src/adversary_pursuit/agent/tools.py` (`run_module` snapshot + emit wiring per §5.1; private `_read_analyst_notes` helper)
- `src/adversary_pursuit/core/console.py` (`_execute_hunt` same pattern; private `_read_analyst_notes` helper)
- `tests/test_dossier_scoring.py` **(NEW)**
- `tests/test_dossier_slot_inference.py` (extend — transition-readiness tests only)
- `tests/test_scoring.py` (extend — re-tune assertions)
- `tests/test_streak.py` (extend — F62 invariants under M-3)
- `tests/test_agent_tools.py` (extend — compound integration)
- `MASTER_PLAN.md` (**Phase 17F** section, appended after Phase 17E; implementer commits in same commit as source)

### Forbidden (the implementer MUST NOT touch these — F* invariants)

- `src/adversary_pursuit/core/workspace.py` — **F59 + DEC-68 invariant**
- `src/adversary_pursuit/core/streak.py` — F62 invariant
- `src/adversary_pursuit/core/pivot_policy.py` — F60 invariant
- `src/adversary_pursuit/core/event_bus.py` — F60 invariant
- `src/adversary_pursuit/dossier/slot_inference.py` — M-2 byte-identical
- `src/adversary_pursuit/dossier/slots.py` — M-1/M-2 byte-identical (SLOT_WEIGHTS authority preserved)
- `src/adversary_pursuit/dossier/panel.py` — M-1 byte-identical
- `src/adversary_pursuit/gamification/celebrations.py` — F63 milestone announce
- `src/adversary_pursuit/gamification/modes.py` — C-1/C-2 territory
- `src/adversary_pursuit/agent/runner.py` — C-1/C-2 territory
- `src/adversary_pursuit/agent/chat.py` — F64 invariant
- `src/adversary_pursuit/models/**` — schema unchanged
- `src/adversary_pursuit/modules/**` — Principle 4 (modules emit no scoring)
- `pyproject.toml` — no new deps
- `CLAUDE.md`, `AGENTS.md`, `settings.json`, `hooks/**`, `agents/**`, `.claude/**` (other than this plan file), `runtime/**` — constitution-level guard

### State authorities touched

- **dossier_score_event_emission** (NEW, owned by `dossier/scoring.py`) — pure function; no I/O.
- **per_ioc_score_rule_constants** (existing, owned by `gamification/scoring.py::DEFAULT_RULES`) — values re-tuned; ownership preserved.
- **score_events_table** (existing, owned by `core/workspace.py::store_score_events`) — written via the existing API; no schema change.
- **dossier_inference_state_snapshot** (existing, owned by `dossier/slot_inference.py`) — read at two snapshot points per hunt; not mutated.

---

## 9. Decision Log (binding for M-3; verbatim into Phase 17F)

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M3-DOSSIER-001** | New file `src/adversary_pursuit/dossier/scoring.py` containing `emit_dossier_slot_filled_events(pre: DossierState, post: DossierState) -> list[dict]` as a pure function. No I/O, no subscriber, no workspace mutation. Caller wires the pre/post snapshots and persists the returned events via the existing `workspace_mgr.store_score_events(...)` API. | DEC-68-DOSSIER-REFRAME-002 chose option (c) "layer over scoring." Pure function honors that layering: the new file is *the* dossier-event-emission authority, callers integrate without changing scoring-engine semantics or workspace persistence semantics. Rejects two architecturally simpler alternatives (event-bus subscriber; ScoringEngine-internal computation) — see §2.2. |
| **DEC-M3-DOSSIER-002** | Caller wiring lives in `agent/tools.py::run_module` AND `core/console.py::_execute_hunt`. Both capture `pre_dossier` BEFORE `store_stix_objects`, capture `post_dossier` AFTER, compute the diff via `dossier/scoring.py::emit_dossier_slot_filled_events`, persist the events via the existing `store_score_events` API, and include the events in the existing `events` list returned to the LLM as `score_events`. | These are the two existing per-hunt site authorities — both already own the "after hunt" boundary (per-IOC `score_results`, badge/challenge checks, streak update). Adding the dossier snapshot at the same site preserves the single-site-per-hunt pattern (Sacred Practice 12). No new orchestration layer, no new dispatcher. |
| **DEC-M3-DOSSIER-003** | `ScoringEngine` (in `gamification/scoring.py`) is unchanged in *behavior*; only the per-IOC `DEFAULT_RULES` constants are re-tuned (§4). `dossier/scoring.py` is an EVENT EMITTER consumed by the existing scoring path — NOT a parallel scorer. The `score_events` table is the single persistence authority. | Sacred Practice 12: the question "what is a scoreable event in AP?" still has one owner (the score_events table, via the store_score_events API). The question "given a slot state diff, what dossier events does it imply?" gets a new explicit owner (`dossier/scoring.py`). Two distinct questions, one authority each. |
| **DEC-M3-DOSSIER-004** | Per-IOC `DEFAULT_RULES` are re-tuned to `initial == minimum == 1` for all 9 SCO-mapped action keys (`new_ip`, `new_domain`, `new_url`, `new_email`, `adversary_mistake`, `deception_uncovered`, `adversary_linked`, `new_tool`, `campaign_described`). `decay` constants preserved (mathematically inert under `initial == minimum`, but kept so the re-tune diff is minimal and reversible). `streak_continued` (F62/F63) is UNCHANGED. | DEC-68-DOSSIER-REFRAME-002 mandates baseline 1.0 for per-IOC events so slot weights (2–5) dominate. AP scoring stores integers; the closest honest mapping of "weight 1.0 baseline" is `initial == minimum == 1`, which collapses the parabolic decay to a constant 1 regardless of solve_count. Preserving `decay` keeps the diff small and the rollback clean. |
| **DEC-M3-DOSSIER-005** | `dossier_prediction_validated` event subtype is **scaffolded** in M-3 (event shape defined, helper function `emit_dossier_prediction_validated_event(prediction)` ships and is tested for shape contract) but **NOT emitted** during any M-3 hunt. M-4 (persistent dossier state) plugs in the auto-validation logic when real prediction records exist. | Per DEC-M2-DOSSIER-004, the Predictions slot remains DEFERRED until M-4. Without persistent prediction records, no validation transitions occur in M-3. Scaffolding the shape + helper now (a) gives M-4 a stable contract to target, (b) prevents future implementers from inventing incompatible event keys, (c) is testable without M-4 persistence. The DEC-68-DOSSIER-REFRAME-007 falsified-prediction-score-deduction question remains explicitly deferred to M-4 (M-3 ships zero negative-score logic). |

---

## 10. Out-of-Scope (deferred to later slices)

- **Persistent dossier state** — M-4 owns. No `dossier_slot`, `dossier_evidence_link`, or `dossier_prediction` SQLite tables in M-3.
- **Falsified-prediction score deduction** (DEC-68-DOSSIER-REFRAME-007 open question) — M-4 owns. M-3 ships zero negative-score logic.
- **`DossierEvidenceConfidenceUpgraded` event** (roadmap §M-3 mentions but is gated on per-slot confidence inference depth that M-2 didn't ship; M-4/M-7 will revisit when persistent confidence histograms exist).
- **Denial / Deception slot fill events** — M-5 owns the authoring surface. M-3 emits no events for slot 9.
- **Dossier-aware auto-pivot policy budget** — M-6 owns.
- **Reports / celebrations / badges narrative upgrades** — M-7 owns.
- **`dossier/scoring.py` having its own confidence-multiplier** — deferred. M-3 uses `int(SLOT_WEIGHTS[slot])` flat. M-4/M-7 may introduce a confidence multiplier when the workspace has real per-slot confidence values.

---

## 11. Cross-References

- **MASTER_PLAN.md Phase 17F** — the binding planner section that cites this document.
- **`.claude/plans/dossier-reframe-v2-roadmap.md` §M-3** — strategic roadmap definition.
- **MASTER_PLAN.md Phase 17D** — M-2 closeout; orphan-prevention precedent + AP #74 lesson.
- **`src/adversary_pursuit/dossier/slot_inference.py`** — M-2 byte-identical; M-3 only READS its output.
- **`src/adversary_pursuit/dossier/slots.py`** — M-1/M-2 byte-identical; `SLOT_WEIGHTS` authority.
- **`src/adversary_pursuit/gamification/scoring.py`** — M-3 re-tunes `DEFAULT_RULES` constants only.
- **`src/adversary_pursuit/core/workspace.py`** — UNTOUCHED; `store_score_events` API consumed by M-3.
- **`src/adversary_pursuit/core/report.py`** — DEC-M2-MOTIVATION-001 direct-engine `AnalystNote` query pattern (lines 348-369); M-3 mirrors in caller files.
- **DEC-68-DOSSIER-REFRAME-002** — selected scoring authority resolution (option c).
- **DEC-68-DOSSIER-REFRAME-007** — falsified-prediction-deduction question deferred to M-4 (NOT M-3 — re-routed per M-3 scope discipline).
- **DEC-M2-DOSSIER-004** — Predictions/Denial scaffold-only invariant preserved.
- **DEC-62-STREAK-001..007 / DEC-63-* / DEC-64-LLM-PANEL-SEPARATION-001** — all preserved; M-3 emits no behavior change at any of these surfaces.

---

## 12. Subsequent Workflow Cue

After M-3 lands, the recommended next workflow is **M-4 — Persistent Dossier State + Predictions Log** per `.claude/plans/dossier-reframe-v2-roadmap.md` §M-4. M-4 introduces SQLite tables (`dossier_slot`, `dossier_evidence_link`, `dossier_prediction`), migrates the M-1/M-2/M-3 in-memory inference to persistent state, and plugs the `dossier_prediction_validated` emitter (scaffolded by M-3 per DEC-M3-DOSSIER-005) into real validation/falsification rules. M-4 also resolves DEC-68-DOSSIER-REFRAME-007 (whether falsified predictions deduct score).

C-3 (Philosophy + Bureaucratese modes) remains independent (DEC-30-CHARACTER-V2-007) and may land in the same wave or independently.
