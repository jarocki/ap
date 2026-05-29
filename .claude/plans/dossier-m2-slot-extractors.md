# M-2 Plan: Per-Module Slot Extractors + `get_dossier_state` LLM Tool

**Status:** planner-authored (per-slice plan; binding for the W-68-M2-SLOT-EXTRACTORS workflow).
**Workflow:** `w-68-m2-slot-extractors`
**Goal:** `g-68-m2-slot-extractors`
**Planner work item:** `wi-68-m2-planner`
**Implementer work item (proposed):** `wi-68-m2-impl-01`
**Worktree:** `/Users/jarocki/src/ap/.worktrees/feature-68-m2-slot-extractors`
**Branch:** `feature/68-m2-slot-extractors` (from AP main `e49e70b`)
**Authored:** 2026-05-28
**Drives:** M-2 slice of the Phase 16 dossier roadmap. Does NOT amend MASTER_PLAN.md (per orchestrator instruction: Phase 17X tracking is deferred to AP #74 doc closeout to avoid stacking more orphaned-planner-content carry-forward).
**Supersession:** none. All Phase 16 / Phase 17 DEC-IDs continue to bind. F59 / F60 / F62 / F63 / F64 invariants preserved.

---

## 0. Scope & Non-Scope (planner asserts; implementer honors)

### IN scope
- Extend `src/adversary_pursuit/dossier/slot_inference.py` with 4 real extractors (Timing, Targeting, Capability, Motivation) and 2 scaffold-only extractors (Predictions, Denial).
- Extend `src/adversary_pursuit/dossier/slots.py` with **no schema vocabulary change** but with the additional metadata the new extractors need (extended `SLOT_EVIDENCE_TYPES` for Targeting; new `MOTIVATION_TAG_VOCABULARY` constant; new `M2_ACTIVE_SLOTS` constant; `M1_ACTIVE_SLOTS` preserved as a historical marker).
- Add the `get_dossier_state` LLM tool in `src/adversary_pursuit/agent/tools.py` per DEC-M1-DOSSIER-004 (the LLM tool M-1 deferred).
- New / extended tests under `tests/`:
  - `tests/test_dossier_slot_inference.py` (extend with M-2 cases)
  - `tests/test_dossier_get_state_tool.py` (NEW)
  - `tests/test_agent_tools.py` (extend with `get_dossier_state` registration + dispatch)
- A new infer entrypoint `infer_dossier_state_full(scos, module_runs=None, notes=None)` that accepts the additional inputs the M-2 extractors need; the legacy `infer_dossier_state(scos)` keeps working unchanged (M-1 contract preserved for the existing chat meta-command site).

### OUT of scope (DO NOT touch)
- `src/adversary_pursuit/core/workspace.py` — F59 / DEC-59-STIX-PROVENANCE-001 authority. The Motivation extractor reads `AnalystNote` rows via the same direct-engine pattern `core/report.py` already uses; no new `workspace.py` accessor is added in M-2.
- `src/adversary_pursuit/core/pivot_policy.py`, `src/adversary_pursuit/core/event_bus.py` — F60 authority.
- `src/adversary_pursuit/core/streak.py`, `src/adversary_pursuit/gamification/scoring.py`, `src/adversary_pursuit/gamification/celebrations.py` — F62 / F63 / F64.
- `src/adversary_pursuit/gamification/modes.py`, `src/adversary_pursuit/agent/runner.py` — C-1 / C-2 territory.
- `src/adversary_pursuit/models/**`, `src/adversary_pursuit/modules/**` — modules emit no provenance per DEC-61-MODULES-EMIT-NO-PROVENANCE-001; M-2 reads only.
- `MASTER_PLAN.md` — defer Phase 17X closeout to AP #74 future doc slice.
- `src/adversary_pursuit/dossier/panel.py` — *byte-identical* in M-2. The panel already renders all 9 slots (M-1 landed `_DEFERRED_MILESTONE`/`_SLOT_ORDER` for the 9-slot puzzle); the extractor wiring change in M-2 must let the existing panel render the new real statuses with zero panel-code edits. (Exception: if the implementer discovers a panel-side bug surfaced by M-2 real-status rows, raise a separate planner re-stage; do not in-place edit.)
- New SCOs / new modules / new STIX writes / new persistence tables. M-4 owns the persistence layer.

### Critical scope boundaries codified
The Scope Manifest in §3.b below is the runtime authority. The orchestrator MUST run `cc-policy workflow scope-sync` with the manifest before dispatching the implementer.

---

## 1. Problem Decomposition

**Problem.** M-1 landed the 9-slot dossier panel with real inference for 3 of 9 slots (Identity, TTPs, Infrastructure) and `DEFERRED` placeholders for the other 6 (Timing, Targeting, Capability, Motivation, Predictions, Denial). The user-visible puzzle metaphor only delivers ⅓ of its promise — the panel currently shows 6 dimmed rows with `M-2 / M-4 / M-5` milestone labels. The agent also cannot reason about dossier state because M-1 deferred the LLM tool (DEC-M1-DOSSIER-004).

**Goal.** After M-2:
- The dossier panel renders **real status** for at least 4 of the 6 currently-deferred slots (Timing, Targeting, Capability, Motivation). Predictions and Denial remain deferred as *scaffold-only* per §2 below — but their deferral marker in M-2 names a clearer milestone (M-4 / M-5) rather than re-using the generic M-2 placeholder.
- The agent has a `get_dossier_state` LLM tool that returns a structured dict the LLM can reason about ("which slot to push next?").
- All Phase 16 / Phase 17 invariants preserved (F59 / F60 / F62 / F63 / F64).
- M-1 panel + meta-command rendering continues to work unchanged (regression test).

**Non-goals.** (Explicit exclusions with rationale.)
- **No auto-inference for Predictions / Denial in M-2.** Predictions need persistent state across sessions (M-4 owns); Denial needs a user-note surface (M-5 owns). M-2 ships *typed scaffolding* (the `PredictionRecord` / `DenialStrategyRecord` value objects) so M-4 / M-5 can plug into them, but the M-2 extractors return `SlotStatus.DEFERRED` for both. Rationale: §2.5 below.
- **No scoring changes.** M-3 owns. M-2 does not emit new `ScoreEvent`s, does not change weights, does not register new event subtypes.
- **No new workspace tables.** M-4 owns persistence. M-2 reads existing `stix_objects`, `module_runs`, `notes`.
- **No panel CSS / display order changes.** Panel is byte-identical (§0 OUT scope).
- **No removal of `M1_ACTIVE_SLOTS`.** It stays as a historical marker so future readers can see the M-1 / M-2 expansion progression. Replaced as the inference driver by a new `M2_ACTIVE_SLOTS` constant.

**Unknowns / ambiguities resolved by this plan.**
- *Motivation extractor data source.* The dispatch spec said "derive from analyst-note tagging (workspace notes already exist)" but the workspace has no `get_analyst_notes()` reader and the spec forbids touching `workspace.py`. **Resolution:** use the same direct-engine `AnalystNote` query pattern `core/report.py` already uses (lines 348-369). The Motivation extractor receives a `notes: list[dict]` parameter (pre-fetched by the caller), parallel to how it receives `scos: list[dict]` today. The caller (chat meta-command site + LLM tool) fetches notes via the engine-direct pattern. **No `workspace.py` edit.**
- *Capability extractor "not observed" count.* See DEC-M2-DOSSIER-003 below.
- *Timing cluster threshold.* See DEC-M2-DOSSIER-002.

**Dominant constraints.**
- F59: must not write `x_ap_*` provenance.
- F60: must not emit `ScoreEvent`s and must not register new event-bus subscriptions.
- F62 / F63: streak / milestone math is byte-identical (M-2 does not touch `gamification/scoring.py`).
- F64: the new LLM tool must NOT leak dossier panel text into the LLM-facing summary. DEC-M2-DOSSIER-005 codifies this.
- Sacred Practice 12: `dossier/` remains the sole inference authority. The `get_dossier_state` tool delegates to `dossier.slot_inference` — it does NOT re-infer in `agent/tools.py`.

---

## 2. Architecture Design

### 2.1 State-Authority Map (new and existing)

| Operational fact | Authority module | Read or Write |
|---|---|---|
| SCO list for the active workspace | `core/workspace.py::get_stix_objects()` | M-2 reads |
| Module run history (for Timing + Capability) | `core/workspace.py::get_module_runs()` | M-2 reads |
| Analyst notes (for Motivation) | `models/database.py::AnalystNote` via direct SQLAlchemy engine (pattern from `core/report.py`) | M-2 reads |
| `x_ap_*` provenance fields on SCOs | `core/workspace.py::store_stix_objects()` | M-2 must not write; may read `x_ap_fetched_at` |
| Dossier slot inference | `dossier/slot_inference.py` (existing + extended) | M-2 owns |
| Dossier slot vocabulary + weights + evidence-type table | `dossier/slots.py` (existing + extended) | M-2 owns |
| Dossier panel rendering | `dossier/panel.py` (existing, byte-identical) | M-2 reads only |
| `get_dossier_state` tool surface | `agent/tools.py` (extended) | M-2 owns |
| Module subscription list (for Capability "expected modules") | `core/event_bus.py::DEFAULT_SUBSCRIPTIONS` | M-2 reads (does not modify) |
| Scoring / score events | `gamification/scoring.py` | M-2 must not touch |
| Streak math | `core/streak.py` | M-2 must not touch |
| Pivot policy | `core/pivot_policy.py` | M-2 must not touch |

### 2.2 Extractor architecture — single-call-per-slot vs aggregator

**Decision: DEC-M2-DOSSIER-001 — single inference call per slot, dispatched by `infer_dossier_state_full()`.**

Two options considered:

**Option A — Per-slot extractor functions, dispatched by a top-level `infer_dossier_state_full(...)`.**
Each slot has a pure function `_infer_<slot>_slot(scos, module_runs, notes) -> SlotState`. The dispatcher calls each in turn and returns `DossierState`. Trivially unit-testable; each test calls one extractor in isolation.

**Option B — Aggregator that walks the SCO list once and updates all slot accumulators in a single pass.**
Single-pass efficiency for very large workspaces; cleaner code if many slots share the same inputs. Harder to test in isolation; one buggy slot can mask another.

**Selected: A.** The 9 dossier slots have *different input shapes* (Identity uses SCO types; Timing uses `x_ap_fetched_at` clustering; Capability uses module-run counts; Motivation uses analyst-note tagging). A single-pass aggregator would either duplicate the per-slot logic anyway (one branch per slot inside the loop) or force every slot into the same input shape, which is wrong. Per-slot extractors map 1:1 to per-slot unit tests, which is exactly the Evaluation Contract shape M-2 needs. The dispatcher is a thin function (~20 lines) that fans out and assembles results. Performance is fine — the workspaces we ship for are O(10²) SCOs not O(10⁶).

### 2.3 `infer_dossier_state_full()` signature (binding)

```python
def infer_dossier_state_full(
    scos: list[dict],
    module_runs: list[dict] | None = None,
    notes: list[dict] | None = None,
) -> DossierState:
    """Full M-2 inference: all 9 slots populated from workspace facts.

    Parameters
    ----------
    scos:
        List of plain STIX SCO dicts as returned by
        ``WorkspaceManager.get_stix_objects()``. Required.
    module_runs:
        Optional list of module-run dicts as returned by
        ``WorkspaceManager.get_module_runs()``. Each dict has keys:
        ``module_name``, ``target``, ``timestamp``, ``result_count``.
        When ``None`` or empty, Timing and Capability slots return
        ``SlotStatus.EMPTY`` instead of inferring.
    notes:
        Optional list of analyst-note dicts. Each dict has keys:
        ``id``, ``content``, ``stix_object_id`` (nullable), ``created_at``.
        When ``None`` or empty, Motivation slot returns ``SlotStatus.EMPTY``.

    Returns
    -------
    DossierState
        Immutable snapshot of all 9 slot fill states.
    """
```

Why three parameters, not one workspace handle:
- Pure-function discipline (DEC-M1-DOSSIER-001) is preserved — no I/O, no SQLAlchemy session inside `dossier/`.
- Caller (chat meta-command + LLM tool dispatcher) owns the I/O fan-out; they call three workspace readers and pass the three lists in.
- Tests can supply synthetic dicts for each parameter independently — same pattern as the M-1 SCO-only tests.

The legacy `infer_dossier_state(scos)` is kept as a thin wrapper that calls `infer_dossier_state_full(scos, module_runs=None, notes=None)`. This preserves the M-1 chat meta-command site (currently calls `infer_dossier_state(raw_objects)`) without forcing the implementer to edit `chat.py` in M-2. The implementer MAY upgrade the chat site to call `infer_dossier_state_full(...)` so the new slots render for real, but the API back-compat shim guarantees no breaking change to any caller.

### 2.4 Per-extractor design (the 6 deferred slots)

#### Slot 4 — Timing / Behavioral (real inference)

**Inputs:** `scos` (for `x_ap_fetched_at` on each SCO) and `module_runs` (for `timestamp` on each module execution).

**Algorithm.**
1. Collect all timestamps from `scos[*].x_ap_fetched_at` and `module_runs[*].timestamp`. Skip SCOs without provenance (M-1 vintage SCOs may lack it; legacy storage now defaults it but pre-storage runs may not have it).
2. Bucket timestamps by **hour-of-day (UTC)** into 24 buckets and by **day-of-week** into 7 buckets.
3. If total event count < 5 → `SlotStatus.EMPTY` (matches roadmap §3 "Low = <5 events").
4. If total event count between 5 and 9 → `SlotStatus.PARTIAL` (matches "Medium = 5-10").
5. If total event count ≥ 10 AND at least one hour-bucket holds ≥ 25% of events (clustering signal) → `SlotStatus.FILLED` (matches "High = ≥10 events clustering to a timezone").
6. Otherwise → `SlotStatus.PARTIAL`.
7. `evidence_count = total timestamps observed`.
8. `contributing_types = frozenset({"x_ap_fetched_at", "module_runs"})` for tests to assert on.

**DEC-M2-DOSSIER-002.** The cluster threshold for the "Filled" jump is **a single hour-of-day bucket holding ≥25% of all events**. Rationale: roadmap §3 calls "clustering to a timezone or weekday distribution" the High-confidence signal. 25% of all events in one of 24 buckets is ~6× uniform — a strong working-hours signal — without requiring more sophisticated statistics that M-2 should not pull in. M-3 may refine. M-2 does NOT implement weekday clustering (kept as a TODO marker in code with explicit DEC-M2-DOSSIER-002 reference).

#### Slot 5 — Targeting Profile (real inference)

**Inputs:** `scos` (specifically `identity` SDOs with `sectors` / `countries` and `location` SDOs; today the only consistent producer is `osint/censys_host` via its `x_location_country` and `x_autonomous_system.country_code` fields on `ipv4-addr` SCOs).

**Algorithm.**
1. Collect (country_code, sector_label) tuples from:
   - `identity` SDO `sectors` / `countries` arrays (canonical STIX path, if any module ever emits them).
   - `ipv4-addr` SCOs with `x_location_country` (Censys path; documented in `censys_host.py` line 363).
   - `ipv4-addr` SCOs with `x_autonomous_system.country_code` (Censys path; line 349).
2. If total observations < 1 → `SlotStatus.EMPTY`.
3. If total < 3 → `SlotStatus.PARTIAL`.
4. If ≥ 3 observations AND at least 2 share a country OR sector → `SlotStatus.FILLED` (matches roadmap "High = ≥3 victims share sector + region").
5. Otherwise → `SlotStatus.PARTIAL`.
6. `evidence_count = total observation tuples`.

**Extended `SLOT_EVIDENCE_TYPES`.** Add `"identity"` and `"location"` (STIX SDO type strings) → `[DossierSlotName.TARGETING]`. Note: these are SDO types, not SCO types — `SLOT_EVIDENCE_TYPES` becomes a broader "STIX type → slot" map. Rename the constant for clarity to `SLOT_STIX_TYPE_MAP` and keep `SLOT_EVIDENCE_TYPES` as a deprecated alias for one slice (M-2 → M-3) before removal in M-3. Implementer note: this is a *naming* change with explicit alias-and-remove discipline (Sacred Practice 12 — the alias is *temporary, with a named removal milestone*, not a permanent fallback).

(If the implementer decides the alias-and-remove pattern over-complicates the M-2 slice, the alternative is to keep the name `SLOT_EVIDENCE_TYPES` and add a docstring update saying "now covers SDO types too." Implementer's call — DEC-M2-DOSSIER-001 grants extractor-shape latitude. The rename is a *recommendation* not a binding decision.)

#### Slot 6 — Capability Ceiling (real inference)

**Inputs:** `module_runs` + the static `DEFAULT_SUBSCRIPTIONS` map from `core/event_bus.py` (READ ONLY — `core/event_bus.py` is in the "do not touch" list; M-2 imports the constant and reads it).

**Algorithm.**
1. Build the set of "expected modules" = the union of all keys in `DEFAULT_SUBSCRIPTIONS` (12 modules today: abuseipdb, shodan_ip, dns_resolve, whois_lookup, otx, hibp, urlscan, greynoise, urlhaus, threatfox, malwarebazaar, crtsh).
2. Build the set of "observed modules" = `{run["module_name"] for run in module_runs}`.
3. `not_observed = expected_modules - observed_modules`. This is the **"tool families the actor didn't surface" inference** — the capability *ceiling* signal.
4. If `observed_modules` is empty → `SlotStatus.EMPTY` (no inference possible).
5. If `len(observed_modules) < 3` → `SlotStatus.PARTIAL` (insufficient breadth to ceiling-infer).
6. If `len(observed_modules) ≥ 3` AND `len(not_observed) ≥ 3` → `SlotStatus.FILLED` (we've seen enough breadth + identified enough gaps to call it a ceiling).
7. Otherwise → `SlotStatus.PARTIAL`.
8. `evidence_count = len(observed_modules) + len(not_observed)` (the *total signal*, observed + absence-of-evidence).
9. `contributing_types = frozenset({"module_run", "module_absence"})` — symbolic; tests assert these literals.

**DEC-M2-DOSSIER-003.** "Not observed" is computed against `DEFAULT_SUBSCRIPTIONS` keys *as-of-M-2-landing* (12 modules). Rationale: this is the canonical authority for "which modules are currently part of the AP arsenal" — the same source the auto-pivot engine uses. M-2 does NOT walk `PluginManager.load_plugins()` because that would couple capability inference to plugin discovery (a more complex, less deterministic surface). When new modules are added in future slices, `DEFAULT_SUBSCRIPTIONS` is updated alongside, and the capability inference automatically picks up the broader expected set. **Forbidden shortcut:** the extractor MUST NOT hardcode "[12]" anywhere — it must compute `len(DEFAULT_SUBSCRIPTIONS)` at call time.

#### Slot 7 — Motivation Indicators (real inference)

**Inputs:** `notes` (list of `AnalystNote` dicts as fetched via the engine-direct pattern from `core/report.py`).

**Algorithm.**
1. Build a small **motivation tag vocabulary** in `dossier/slots.py` (new `MOTIVATION_TAG_VOCABULARY` constant):
   ```python
   MOTIVATION_TAG_VOCABULARY: dict[str, str] = {
       "financial":   "financial",
       "ransom":      "financial",
       "extortion":   "financial",
       "hacktivist":  "hacktivist",
       "political":   "hacktivist",
       "nation-state":"nation-state",
       "apt":         "nation-state",
       "espionage":   "nation-state",
       "ego":         "ego",
       "vandal":      "ego",
   }
   ```
   Each key is a substring the extractor case-insensitively searches in each note's `content` field. The value is the canonical motivation bucket.
2. For each note, run the substring search. A note may contribute to multiple buckets (a note saying "ransom payment hacktivist front" contributes to both `financial` and `hacktivist`).
3. Count buckets that received ≥1 contribution.
4. If `total contributing notes == 0` → `SlotStatus.EMPTY`.
5. If exactly 1 bucket has signal → `SlotStatus.PARTIAL` (one motivation classification; roadmap §3 "Low = inferred from a single victim type").
6. If ≥2 buckets have signal → `SlotStatus.FILLED` (multiple independent motivation signals).
7. `evidence_count = total contributing notes`.
8. `contributing_types = frozenset(<canonical bucket names that hit>)`.

**Forbidden shortcut:** no LLM-driven motivation classification in M-2. M-2 is deterministic substring matching only. LLM-grounded motivation reasoning is a sequel slice (likely M-3 or M-7).

**Implementer scope note for the caller.** The Motivation extractor needs analyst notes pre-fetched. The caller (chat meta-command site + LLM tool dispatcher) uses the same pattern `core/report.py` lines 360-369 already uses:
```python
from adversary_pursuit.models.database import AnalystNote
from sqlalchemy.orm import Session
with Session(workspace_mgr._engine) as session:
    rows = session.execute(select(AnalystNote).order_by(AnalystNote.id)).scalars().all()
    notes = [{"id": r.id, "content": r.content,
              "stix_object_id": r.stix_object_id,
              "created_at": r.created_at} for r in rows]
```
**Do NOT add `get_analyst_notes()` to `workspace.py` in M-2** — that file is in the no-touch list. Adding a *pure reader* might look harmless, but the orchestrator instruction is explicit, and the `report.py` precedent shows the engine-direct pattern is already accepted convention in the codebase. M-3 / M-4 may choose to promote it; M-2 does not.

#### Slot 8 — Predictions Log (scaffold-only)

**Inputs:** none (no persistence yet).

**Algorithm.**
1. Always return `SlotStatus.DEFERRED`, `evidence_count=0`.
2. The deferral marker (`_DEFERRED_MILESTONE` in `panel.py`) for Predictions is already `"M-4"` — panel renders correctly without change.

**Scaffolding in `dossier/slot_inference.py`:** add a `PredictionRecord` typed dataclass:
```python
@dataclass(frozen=True)
class PredictionRecord:
    """Scaffolding for M-4. M-2 does NOT produce these; M-4 persistence layer will.

    Defined here so the M-4 implementer slice does not have to invent the shape
    from scratch — the dossier package owns the slot vocabulary AND the slot
    record shapes (Sacred Practice 12).
    """
    prediction_id: str
    content: str
    created_at: str  # ISO 8601 UTC
    status: str  # "pending" | "validated" | "falsified"
    evidence_ids: tuple[str, ...] = ()
```

This is not exported in `__all__` for M-2 (no caller can use it yet). It is documented and tested by import-existence assertion only (1 test). The M-4 implementer will import it directly.

**DEC-M2-DOSSIER-004.** Predictions and Denial slots ship as scaffold-only in M-2. The deferral marker continues to point to M-4 (Predictions) and M-5 (Denial). The typed scaffolding (`PredictionRecord`, `DenialStrategyRecord`) is added in M-2 so that downstream milestones have a stable record shape to target. No auto-inference. Rationale: Predictions need *persisted state across sessions* (M-4 owns), Denial needs a *user-note authoring surface* (M-5 owns). Doing either in M-2 would either build the wrong shape (no persistence = lost predictions) or leak into the M-5 user-note surface (which has not been scoped yet).

#### Slot 9 — Denial / Deception (scaffold-only)

**Inputs:** none.

**Algorithm.** Same as Predictions — always returns `SlotStatus.DEFERRED`, `evidence_count=0`. Deferral marker stays `"M-5"`.

**Scaffolding:** add `DenialStrategyRecord` typed dataclass in `dossier/slot_inference.py`:
```python
@dataclass(frozen=True)
class DenialStrategyRecord:
    """Scaffolding for M-5. M-2 does NOT produce these; M-5 user-note surface will."""
    strategy_id: str
    content: str
    linked_evidence_ids: tuple[str, ...] = ()
    created_at: str = ""  # ISO 8601 UTC
```

Same test discipline as Predictions: import-existence + shape assertion only.

### 2.5 Why scaffold-only for Predictions / Denial (DEC-M2-DOSSIER-004 expanded)

Three options considered:

| Option | M-2 effort | Risk |
|---|---|---|
| A: scaffold-only (selected) | small | none — no behavioral change, panel already says "M-4 / M-5" |
| B: in-memory only (lost on session restart) | medium | high — users will set predictions and lose them; UX trap |
| C: full persistence in M-2 | large | high — pulls M-4 into M-2, fights the slice boundary |

Selected: A. The dispatch spec explicitly said "scaffold-only" — this DEC ratifies and records the rationale so a future implementer doesn't re-litigate.

### 2.6 `get_dossier_state` LLM tool surface (DEC-M2-DOSSIER-005)

**Tool registration in `agent/tools.py::create_tools()`:**
```python
{
    "type": "function",
    "function": {
        "name": "get_dossier_state",
        "description": (
            "Read-only inspection of the current Threat Actor Dossier. "
            "Returns a structured snapshot of all 9 dossier slots — "
            "their fill status (empty/partial/filled/deferred), evidence count, "
            "fill percentage, and importance weight. Use this to decide which "
            "slot to push on next: prioritise empty/partial high-weight slots."
        ),
        "parameters": {"type": "object", "properties": {}},
    },
},
```

**Dispatch branch in `execute_tool()`:**
```python
if tool_name == "get_dossier_state":
    return _execute_get_dossier_state(ctx), None, [], []
```

**Helper `_execute_get_dossier_state(ctx) -> str`:**
1. Fetch SCOs: `scos = ctx.workspace_mgr.get_stix_objects()`.
2. Fetch module runs: `runs = ctx.workspace_mgr.get_module_runs()`.
3. Fetch notes via the engine-direct `AnalystNote` pattern from `report.py` (see §2.4 Motivation).
4. Call `infer_dossier_state_full(scos, module_runs=runs, notes=notes)`.
5. Convert `DossierState` to a JSON-serialisable dict:
   ```python
   {
       "slots": {
           "identity":      {"status": "filled",   "evidence_count": 7, "fill_percentage": 100, "weight": 5.0},
           "ttps":          {"status": "partial",  "evidence_count": 1, "fill_percentage": 50,  "weight": 3.0},
           "infrastructure":{"status": "filled",   "evidence_count": 4, "fill_percentage": 100, "weight": 2.0},
           "timing":        {"status": "partial",  "evidence_count": 6, "fill_percentage": 50,  "weight": 2.0},
           "targeting":     {"status": "empty",    "evidence_count": 0, "fill_percentage": 0,   "weight": 2.5},
           "capability":    {"status": "filled",   "evidence_count": 10,"fill_percentage": 100, "weight": 3.5},
           "motivation":    {"status": "empty",    "evidence_count": 0, "fill_percentage": 0,   "weight": 3.0},
           "predictions":   {"status": "deferred", "evidence_count": 0, "fill_percentage": 0,   "weight": 4.0},
           "denial":        {"status": "deferred", "evidence_count": 0, "fill_percentage": 0,   "weight": 2.5},
       },
       "total_sco_count": 12,
       "summary": "3 filled, 1 partial, 3 empty, 2 deferred (of 9 slots)"
   }
   ```
6. Return `json.dumps(snapshot, indent=2)` as the tool summary string.

**`fill_percentage` mapping:**
- `empty` → 0
- `partial` → 50
- `filled` → 100
- `deferred` → 0

**DEC-M2-DOSSIER-005 — F64 surface discipline.**
The `summary` string returned to the LLM is **the typed JSON dict only**. It does NOT include:
- Rich markup (no `[bold]`, no `[/cyan]`).
- Panel-rendered text (no `_status_symbol()`, no `_SLOT_DISPLAY_NAME` human-readable labels — the LLM gets the canonical enum string keys: `"identity"`, `"ttps"`, etc.).
- Gamification narration (no "you filled 3 slots!" — the typed dict speaks for itself; the LLM decides what to say to the user).

The panel rendering surface (`dossier/panel.py`) is the *user-facing* presentation; the JSON dict is the *LLM-facing* representation. Same underlying `DossierState`, two presentations, no leakage between them. This is the F64 "LLM vs Panel separation" invariant applied to the dossier package.

**Forbidden shortcut:** the tool MUST NOT call `dossier.panel.render()` and embed the resulting `Panel.renderable` text in the summary. Each presentation owns its own format.

### 2.7 Alternatives gate — caller fan-out vs in-package I/O

Considered putting the workspace I/O (notes / module_runs fetch) inside a new `dossier/io.py` helper. **Rejected.** It would violate DEC-M1-DOSSIER-001's "pure function, no I/O" stance for the dossier package. The two callers (chat meta-command + LLM tool dispatcher) duplicating ~5 lines of fetch code is fine — the duplication is bounded (two sites) and the alternative is worse (a new I/O authority inside the inference package that has to learn about SQLAlchemy sessions and workspace lifecycle).

If at M-3 a third caller appears, the implementer can refactor a `_collect_workspace_inputs(workspace_mgr) -> tuple[scos, runs, notes]` helper into either `dossier/` (if I/O has matured) or `agent/` (parallel to `ToolContext`). M-2 does not preempt that decision.

---

## 3. Wave Decomposition

### 3.a Work items

| W-ID | Title | Weight | Gate | Deps | Integration touchpoints |
|---|---|---|---|---|---|
| `wi-68-m2-impl-01` | M-2 per-module extractors + `get_dossier_state` LLM tool | M (single bounded implementer slice) | review → guardian(land) | none | `dossier/slot_inference.py`, `dossier/slots.py`, `agent/tools.py`, three test files |

**Wave width:** 1. M-2 is a single bounded implementer slice. Decomposition into smaller slices was considered (one per slot) but rejected — the slots share the dispatcher and the test scaffolding, and shipping the 6 slot inferences in 6 sub-slices would create 6 review cycles for what is fundamentally one architectural change (add real inference for the deferred slots + the LLM tool).

**Critical path:** `planner` → `guardian (provision)` → `implementer` (wi-68-m2-impl-01) → `reviewer` → `guardian (land)`.

### 3.b Scope Manifest (binding — orchestrator MUST `cc-policy workflow scope-sync` before dispatch)

**Allowed (the implementer may touch any of these):**
- `src/adversary_pursuit/dossier/slot_inference.py`
- `src/adversary_pursuit/dossier/slots.py`
- `src/adversary_pursuit/dossier/__init__.py` (export new symbols)
- `src/adversary_pursuit/agent/tools.py`
- `src/adversary_pursuit/agent/chat.py` (OPTIONAL — only to upgrade the `dossier` meta-command call site to use `infer_dossier_state_full(...)` so the new slots render for real; **the rest of `chat.py` is forbidden**, see §3.b "forbidden" below)
- `tests/test_dossier_slot_inference.py` (extend)
- `tests/test_dossier_slots.py` (extend if vocabulary additions warrant)
- `tests/test_dossier_panel.py` (extend with the regression test in §3.c)
- `tests/test_dossier_get_state_tool.py` (NEW)
- `tests/test_agent_tools.py` (extend for tool registration + dispatch)

**Required (the implementer MUST modify these for the slice to be complete):**
- `src/adversary_pursuit/dossier/slot_inference.py` — add the 6 extractors + `infer_dossier_state_full()` + the 2 scaffolding dataclasses (`PredictionRecord`, `DenialStrategyRecord`).
- `src/adversary_pursuit/dossier/slots.py` — add `M2_ACTIVE_SLOTS` constant, extend `SLOT_EVIDENCE_TYPES` for Targeting, add `MOTIVATION_TAG_VOCABULARY`.
- `src/adversary_pursuit/agent/tools.py` — add `get_dossier_state` to `create_tools()` and the `execute_tool` dispatch branch + the `_execute_get_dossier_state` helper.
- `tests/test_dossier_slot_inference.py` — add the M-2 extractor tests (per §4.a).
- `tests/test_dossier_get_state_tool.py` — NEW file (per §4.a).
- `tests/test_agent_tools.py` — add the tool-registration + dispatch tests (per §4.a).

**Forbidden touch points (MUST NOT change; reviewer enforces byte-identity if needed):**
- `src/adversary_pursuit/core/workspace.py` — F59 / DEC-59-STIX-PROVENANCE-001 authority.
- `src/adversary_pursuit/core/pivot_policy.py` — F60.
- `src/adversary_pursuit/core/event_bus.py` — F60 (imports the `DEFAULT_SUBSCRIPTIONS` constant; no edits).
- `src/adversary_pursuit/core/streak.py` — F62.
- `src/adversary_pursuit/gamification/scoring.py` — F62 / F63.
- `src/adversary_pursuit/gamification/celebrations.py` — F63.
- `src/adversary_pursuit/gamification/modes.py` — C-1 / C-2.
- `src/adversary_pursuit/agent/runner.py` — C-2.
- `src/adversary_pursuit/models/**`.
- `src/adversary_pursuit/modules/**`.
- `src/adversary_pursuit/dossier/panel.py` — byte-identical in M-2 (see §0 OUT scope).
- `src/adversary_pursuit/agent/chat.py` outside of the single optional `dossier` meta-command call site upgrade. The implementer may change ONLY the two lines:
  ```python
  state = infer_dossier_state(raw_objects)
  ```
  to:
  ```python
  state = infer_dossier_state_full(raw_objects, module_runs=..., notes=...)
  ```
  plus the matching import. Any other `chat.py` edit is out of scope.
- `MASTER_PLAN.md` — deferred to AP #74.
- `.claude/plans/dossier-reframe-v2-roadmap.md` — strategic doc; not touched by M-2 implementer.

**State authorities touched (read-only):** `stix_objects` (read), `module_runs` (read), `notes` (read). **Authorities written by this slice:** none. M-2 is a read-and-render slice; no persistence change.

---

## 3c. Evaluation Contract (binding — reviewer enforces; guardian verifies before land)

### 3.c.1 Required tests (≥30 tests; explicit per-test list)

**File: `tests/test_dossier_slot_inference.py` (extend; ≥18 new tests)**

Per-slot extractor tests (4 new tests per real-inference slot × 4 slots = 16):
1. `test_timing_slot_empty_when_no_module_runs` — `module_runs=None` → EMPTY.
2. `test_timing_slot_partial_with_few_events` — 5 timestamps clustered to one hour → PARTIAL.
3. `test_timing_slot_filled_with_clustered_events` — 12 timestamps, ≥25% in one hour bucket → FILLED.
4. `test_timing_slot_partial_with_uniform_distribution` — 12 timestamps spread evenly → PARTIAL (no cluster).
5. `test_targeting_slot_empty_with_no_location_evidence` — only `email-addr` SCOs → EMPTY.
6. `test_targeting_slot_partial_with_single_country_signal` — 1 `ipv4-addr` with `x_location_country` → PARTIAL.
7. `test_targeting_slot_filled_with_clustered_victims` — 3 `ipv4-addr` SCOs sharing `x_location_country` → FILLED.
8. `test_targeting_slot_reads_censys_x_fields_without_writing` — provenance pass-through invariant.
9. `test_capability_slot_empty_with_no_module_runs` — no runs → EMPTY.
10. `test_capability_slot_partial_with_two_modules` — 2 distinct modules → PARTIAL.
11. `test_capability_slot_filled_with_breadth_and_gaps` — 4 distinct modules observed, ≥3 in DEFAULT_SUBSCRIPTIONS unobserved → FILLED.
12. `test_capability_slot_reads_default_subscriptions_dynamically` — assert `len(DEFAULT_SUBSCRIPTIONS)` is consulted at call time (not hardcoded).
13. `test_motivation_slot_empty_with_no_notes` — `notes=None` → EMPTY.
14. `test_motivation_slot_partial_with_single_bucket` — 2 notes both tagged "financial" → PARTIAL.
15. `test_motivation_slot_filled_with_multiple_buckets` — notes tagged "financial" AND "hacktivist" → FILLED.
16. `test_motivation_slot_case_insensitive_substring_match` — note "RANSOM extorted from victims" hits the `ransom` substring → financial bucket.

Scaffold tests (2 new tests per scaffold slot × 2 slots = 4):
17. `test_predictions_slot_deferred_always` — even with notes+runs+SCOs, Predictions stays DEFERRED.
18. `test_denial_slot_deferred_always` — even with all inputs supplied, Denial stays DEFERRED.
19. `test_prediction_record_dataclass_shape` — `PredictionRecord` exists and has the binding shape (§2.4 Slot 8).
20. `test_denial_strategy_record_dataclass_shape` — `DenialStrategyRecord` exists and has the binding shape (§2.4 Slot 9).

Integration / regression tests (≥4):
21. `test_infer_dossier_state_full_returns_all_9_slots` — every `DossierSlotName` member present in `state.slots`.
22. `test_infer_dossier_state_legacy_wrapper_unchanged` — `infer_dossier_state(scos)` returns the same M-1 result for an M-1 fixture (no regression).
23. `test_infer_dossier_state_full_pure_function_no_mutation` — input lists unchanged after call.
24. `test_infer_dossier_state_full_no_x_ap_writes` — F59 invariant: no `x_ap_*` keys added to any input dict.

**File: `tests/test_dossier_get_state_tool.py` (NEW; ≥8 tests)**

25. `test_get_dossier_state_tool_registered` — `"get_dossier_state"` appears in `create_tools(ctx)` output.
26. `test_get_dossier_state_tool_schema_no_params` — schema has empty `properties`.
27. `test_get_dossier_state_returns_valid_json` — `execute_tool(ctx, "get_dossier_state", {})` returns a JSON-parseable string.
28. `test_get_dossier_state_snapshot_has_9_slots` — JSON has `"slots"` with all 9 keys.
29. `test_get_dossier_state_fill_percentage_mapping` — EMPTY→0, PARTIAL→50, FILLED→100, DEFERRED→0.
30. `test_get_dossier_state_includes_weight_field` — every slot dict has `"weight"` matching `SLOT_WEIGHTS`.
31. `test_get_dossier_state_no_rich_markup_in_summary` — F64 invariant: no `[...]` brackets in returned string (DEC-M2-DOSSIER-005).
32. `test_get_dossier_state_reads_notes_via_engine_direct_pattern` — verifies the helper fetches notes (uses an in-memory workspace with `add_note()` and asserts Motivation bucket populated in output).
33. `test_get_dossier_state_tuple_shape` — return value is `(summary, None, [], [])` per `execute_tool` contract.
34. `test_get_dossier_state_does_not_render_panel` — assert `dossier.panel.render` is not in the call path (introspect or use monkeypatch to fail-on-import).

**File: `tests/test_agent_tools.py` (extend; ≥2 tests)**

35. `test_create_tools_includes_get_dossier_state` — tool count increases by 1 from current.
36. `test_execute_tool_dispatch_get_dossier_state` — unknown branch removed, dispatch returns non-error for the new name.

**File: `tests/test_dossier_panel.py` (extend; ≥2 tests — regression)**

37. `test_panel_renders_with_m2_full_state` — call `render(infer_dossier_state_full(...))` with non-trivial inputs → panel constructs without error; the panel byte-identity rule is preserved (panel code unchanged).
38. `test_panel_renders_with_m1_legacy_state` — call `render(infer_dossier_state(scos))` with M-1-vintage inputs → identical to M-1 baseline.

**Total: ≥34 tests.** Target ≥30 per dispatch spec; this contract exceeds it intentionally so the implementer has headroom if a couple of tests fold together.

### 3.c.2 Required real-path checks (production sequence)

1. **`ap chat dossier` meta-command end-to-end.** After implementation, manually run `uv run ap chat` (or equivalent), execute one module to populate SCOs (e.g. `dns_resolve example.com`), then type `dossier`. The panel must render with at least Identity, TTPs, Infrastructure showing real status AND at least one of Timing / Capability showing real status (the optional `chat.py` upgrade path). Capture the panel output to a file under `tmp/` as evidence. **If the implementer chose not to upgrade the chat call site:** real-path check is "the panel renders identical to M-1 baseline" and the LLM tool path carries the M-2 evidence (see check 2 below).
2. **`get_dossier_state` LLM tool live path.** Construct a `ToolContext` against a tmp workspace, seed it with 3-5 SCOs + 1 note, call `execute_tool(ctx, "get_dossier_state", {})`, parse the returned JSON, assert all 9 slot keys, capture the output to `tmp/`.
3. **F64 surface check.** Grep the `_execute_get_dossier_state` helper for `[bold`, `[cyan`, `_SLOT_DISPLAY_NAME`, or `render(` — must return zero matches. (Test 31 covers automated assertion; this is the manual sanity check.)

### 3.c.3 Required authority invariants

| Invariant | Source | M-2 enforcement |
|---|---|---|
| F59 — `workspace.store_stix_objects()` is sole `x_ap_*` authority | DEC-59-STIX-PROVENANCE-001 | M-2 reads `x_ap_fetched_at` (Timing) and `x_ap_*` fields on Censys SCOs (Targeting); never writes. Test 8, 24. |
| F60 — `PivotPolicy` is sole gate authority | DEC-60-PIVOT-POLICY-001 | M-2 reads `DEFAULT_SUBSCRIPTIONS` (Capability); never edits `core/event_bus.py` or `core/pivot_policy.py`. Test 12. |
| F62 — `StreakManager` is sole streak authority | DEC-62-STREAK-007 | M-2 does not touch `core/streak.py` or `gamification/scoring.py`. Forbidden-touch list enforces. |
| F63 — milestone catch-up math | DEC-63-MILESTONE-* | Same as F62 — no touch. |
| F64 — LLM tool summary ≠ panel text | DEC-64-LLM-PANEL-SEPARATION-001, DEC-M2-DOSSIER-005 | Test 31, 34. |
| Sacred Practice 12 — single inference authority | DEC-M1-DOSSIER-001 | Test 34 verifies `agent/tools.py` does not re-implement inference; it delegates to `dossier.slot_inference`. |

### 3.c.4 Required integration points (must still work after the slice)

1. M-1 `dossier` chat meta-command continues to render the panel without error.
2. The Rich panel renders cleanly across all 9 slots whether the slots show real M-2 status or remain on M-1 status (the optional chat upgrade decides which).
3. Every other LLM tool in `create_tools()` continues to register and dispatch (test 35 implicitly verifies — the count check ensures no tool was accidentally removed).
4. Full pytest green: `uv run pytest -q` must pass with zero failures and zero errors.
5. Lint clean: `uv run ruff check src/ tests/` zero errors.
6. Type check (if the project runs one): no new mypy / pyright errors introduced in the touched files.

### 3.c.5 Forbidden shortcuts (explicit; reviewer rejects on detection)

- **NO `workspace.py` edit** — even a "harmless" `get_analyst_notes()` reader. Use the engine-direct pattern from `report.py`.
- **NO LLM call inside any extractor** — extractors are pure functions over typed inputs; deterministic substring matching for Motivation is the upper bound on inference sophistication in M-2.
- **NO new SCOs / new STIX writes / new modules** — M-2 is read-side only.
- **NO `dossier.panel.render()` call inside the LLM tool path** — F64 violation.
- **NO hardcoded `len(DEFAULT_SUBSCRIPTIONS)` integer literal** — the Capability extractor must compute at call time so future module additions automatically expand the expected set.
- **NO `MASTER_PLAN.md` edit** — deferred to AP #74.
- **NO scope-file omission** — orchestrator MUST `cc-policy workflow scope-sync` with this manifest before dispatching the implementer (per CLAUDE.md ClauDEX Contract Injection rules).
- **NO removal of `M1_ACTIVE_SLOTS`** — keep as historical marker per §0 IN scope.
- **NO env-var bypass** for any M-2 feature. No `AP_DOSSIER_M2_DISABLE`. No fallback to M-1 if M-2 extractors fail — they should never fail on well-formed input, and on malformed input they return EMPTY (graceful) rather than raising.

### 3.c.6 Ready-for-guardian definition (the exact conditions under which reviewer may declare readiness)

The reviewer may emit `REVIEW_VERDICT: ready_for_guardian` only when ALL of the following are true on the current head SHA:

1. All ≥34 tests listed in §3.c.1 exist as named tests in their target files.
2. `uv run pytest -q` exits 0 with zero failures/errors on a clean run.
3. `uv run ruff check src/ tests/` exits 0.
4. The two real-path checks in §3.c.2 have been performed and the captured outputs exist under `tmp/`.
5. Spot-grep verification of the F64 surface (§3.c.2 check 3) returns zero matches.
6. Scope manifest compliance: `git diff --name-only main...HEAD` lists only files in §3.b "Allowed". No file in §3.b "Forbidden" appears in the diff.
7. The Decision Log §5 below is updated in the plan if any DEC was modified during implementation.
8. The reviewer has read the implementation diff and confirms the binding signatures in §2.3 and §2.6 match the code.

If any of (1)-(8) fails, reviewer emits `needs_changes` with the specific bullet that failed.

### 3.c.7 Rollback boundary

The M-2 slice is a single git merge into `feature/68-m2-slot-extractors`. Rollback is `git revert <merge-sha>` followed by `git push` — no DB migration, no persistence change, no on-disk format change. The slice is 100% reversible.

---

## 4. Decision Log (binding; written to MASTER_PLAN.md at the AP #74 doc closeout)

| DEC-ID | Decision | Rationale |
|---|---|---|
| **DEC-M2-DOSSIER-001** | Per-slot extractor architecture: each of the 6 new slots has its own pure-function `_infer_<slot>_slot(...)` extractor. A single dispatcher `infer_dossier_state_full(scos, module_runs, notes)` fans out to each extractor and assembles the `DossierState`. Per-slot extractors over a single-pass aggregator. | Per-slot extractors map 1:1 to per-slot unit tests, which is exactly the Evaluation Contract shape M-2 needs. Single-pass aggregator would force same-shape inputs or duplicate per-slot branching; per-slot extractors are clearer, more testable, and let each slot evolve independently in future milestones. §2.2. |
| **DEC-M2-DOSSIER-002** | Timing extractor uses `x_ap_fetched_at` + `module_runs.timestamp` and clusters by hour-of-day UTC; "FILLED" threshold is ≥10 total events AND ≥25% concentrated in one of 24 hour buckets. Weekday clustering deferred. | Matches roadmap §3 confidence ladder (Low <5 / Medium 5-10 / High ≥10 + cluster). 25% in 1 of 24 buckets is ~6× uniform — a strong working-hours signal — without pulling in scipy or full statistical clustering. M-3 may refine; M-2 ships the simple, testable threshold. §2.4 Slot 4. |
| **DEC-M2-DOSSIER-003** | Capability extractor: "expected modules" is `set(DEFAULT_SUBSCRIPTIONS.keys())` from `core/event_bus.py` evaluated at call time. "Not observed" = expected − observed (from `module_runs`). FILLED requires ≥3 observed AND ≥3 unobserved. | `DEFAULT_SUBSCRIPTIONS` is the canonical module-arsenal authority the auto-pivot engine already uses; reusing it avoids creating a parallel "module list" surface (Sacred Practice 12). The 3+3 thresholds give a meaningful breadth + ceiling-inference signal without false-positives on tiny investigations. §2.4 Slot 6. |
| **DEC-M2-DOSSIER-004** | Predictions (slot 8) and Denial (slot 9) are scaffold-only in M-2: extractors always return `SlotStatus.DEFERRED`; M-2 adds typed `PredictionRecord` / `DenialStrategyRecord` dataclasses so M-4 / M-5 have stable shapes to target. No auto-inference, no in-memory state, no persistence. | Predictions need persisted cross-session state (M-4 owns); Denial needs a user-note authoring surface (M-5 owns). Doing either in M-2 would build the wrong shape (in-memory predictions are lost; UX trap) or pull M-4/M-5 scope into M-2. Scaffolding ships now so successor slices have a stable contract. §2.5. |
| **DEC-M2-DOSSIER-005** | `get_dossier_state` LLM tool returns a typed JSON dict (`{slots: {name: {status, evidence_count, fill_percentage, weight}}, total_sco_count, summary}`) as its `summary` string. No Rich markup, no `_SLOT_DISPLAY_NAME` text, no `dossier.panel.render()` invocation in the tool path. | F64 (DEC-64-LLM-PANEL-SEPARATION-001) applied to the dossier package: the LLM-facing representation is structured data the LLM reasons about; the user-facing representation is the Rich panel. Same `DossierState` source-of-truth, two separate presentations, no double-narration risk. §2.6. |

These DECs are **binding** for the W-68-M2-SLOT-EXTRACTORS workflow. They will be promoted into MASTER_PLAN.md Phase 17X at the AP #74 doc closeout. Until then, this document is the canonical record.

---

## 5. Cross-References

- `.claude/plans/dossier-reframe-v2-roadmap.md` §5 (M-2 definition) and §7 (sequencing).
- `MASTER_PLAN.md` Phase 16 (Dossier Reframe Strategic Scoping) and Phase 17 (Character v2 Scoping — for context on the parallel C-1/C-2 work).
- `src/adversary_pursuit/dossier/{__init__.py, slots.py, slot_inference.py, panel.py}` — landed M-1 at AP main `486a5ad` (2026-05-28).
- `src/adversary_pursuit/agent/chat.py` lines 393-416 — M-1 `dossier` meta-command site (the optional chat upgrade point).
- `src/adversary_pursuit/agent/tools.py` lines 668-1480 — `create_tools()` + `execute_tool()` (extension points).
- `src/adversary_pursuit/core/report.py` lines 348-369 — the canonical engine-direct `AnalystNote` read pattern the Motivation extractor caller reuses.
- `src/adversary_pursuit/core/event_bus.py` lines 86-100 — `DEFAULT_SUBSCRIPTIONS` (Capability extractor reads).
- `src/adversary_pursuit/modules/osint/censys_host.py` lines 339-365 — `x_location_country` + `x_autonomous_system` emission (Targeting extractor reads).
- DEC-M1-DOSSIER-001..004 — M-1 binding decisions; M-2 honors all.
- DEC-59-STIX-PROVENANCE-001, DEC-60-PIVOT-POLICY-001..007, DEC-62-STREAK-*, DEC-63-MILESTONE-*, DEC-64-LLM-PANEL-SEPARATION-001 — preserved.

---

## 6. Continuation after M-2 lands

Per the Phase 16 roadmap §5 sequencing: M-2 unblocks **M-3 (Dossier Scoring + Score Event Re-Tune)**. After M-2 lands cleanly:

- The planner re-stages for M-3 with a fresh Evaluation Contract focused on `gamification/scoring.py` (which is forbidden in M-2 but **required** in M-3).
- The DEC-68-DOSSIER-REFRAME-007 deferred question (whether falsified predictions should *deduct* score) is M-3's to answer.
- M-3's Scope Manifest will allow `gamification/scoring.py` and `dossier/scoring.py` (NEW) and continue to forbid the F59 / F60 / F64 surfaces.

This planner emits `PLAN_VERDICT: next_work_item` so the orchestrator can immediately dispatch guardian-provision for `wi-68-m2-impl-01`.
