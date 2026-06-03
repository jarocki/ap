# M-4 — Persistent Dossier State + Predictions Log Auto-Validation (per-slice plan)

**Status:** planner-staged 2026-06-02 by W-68-M4-PERSISTENT-DOSSIER planner stage. Implementer slice to follow.
**Workflow:** `w-68-m4-persistent-dossier`
**Goal:** `g-68-m4-persistent-dossier`
**Work item to dispatch:** `wi-68-m4-impl-01`
**Drives:** Phase 17G of `MASTER_PLAN.md`. Phase 17G carries the binding decisions and the slice index; this document carries the full rationale, sentinel-row JSON contract, validation-engine vocabulary, hook-site diff sketches, and decomposition detail. When the two diverge, Phase 17G wins for binding decisions; this document wins for narrative and table detail.

**Inherits from:** Phase 16 §M-4, `.claude/plans/dossier-reframe-v2-roadmap.md` §M-4. Phase 17B (M-1 panel), Phase 17D (M-2 extractors + scaffold dataclasses), Phase 17F (M-3 scoring + prediction-validated scaffold) are prerequisites; all shipped by 2026-06-01. Worktree base: AP main `2809b13`.

---

## 1. Goal (single paragraph)

Make the dossier **stateful across hunts** and turn the Predictions Log into a real, scored slot. Two interlocking surfaces ship in this slice:

1. **Persistent DossierState** — a single per-workspace snapshot of the most-recent inferred dossier state, written to workspace SQLite at the end of every hunt and read at the start of the next hunt. M-3's "pre = infer_dossier_state_full(...)" becomes "pre = load_dossier_state() or default_deferred_state(); compare against fresh post-inference". State survives `ap chat` restarts.
2. **Predictions Log lifecycle** — a NEW `create_dossier_prediction` LLM tool lets the analyst (via the agent) author predictions tied to slots; predictions are persisted as `PredictionRecord` entries; subsequent hunts auto-validate them by matching new evidence against typed `expected_evidence` patterns; on confirmation, M-3's scaffolded `emit_dossier_prediction_validated_event` fires with weight=4.0 (Phase 16 §3 Predictions slot weight).

After M-4, the dossier is no longer recomputed from scratch every hunt — it has memory; predictions are first-class scored events; and the Predictions slot (slot 8) status transitions from `deferred` to real `empty`/`partial`/`filled` based on `pending`/`validated`/`falsified` ratios.

**Out-of-scope (explicit, deferred):**
- **Active falsification rules** — M-4 ships **confirmation-only** validation. The question "what evidence proves a prediction was wrong?" is harder than "what evidence confirms it?" — M-5 owns active falsification (typed `falsification_evidence` shape + per-prediction `falsification_after_n_hunts` window) per DEC-M4-PRED-005 below.
- **DEC-68-DOSSIER-REFRAME-007 falsified-prediction score deduction** — DEC-M3-DOSSIER-005 explicitly deferred this to M-4. M-4 commits the default in DEC-M4-PRED-006: confirmation = +N points; falsification = 0 points (no deduction). Deeper "should reckless guessing cost score?" question is documented as a future re-stage candidate, NOT relitigated in M-4 implementer.
- **Denial / Deception Strategies slot 9** — M-5 owns the user-note authoring surface.
- **Dossier-aware auto-pivot policy** — M-6 owns.
- **Reports / celebrations / badges narrative upgrades** — M-7 owns.
- **NO new SQLite tables and NO `models/database.py` changes** — DEC-M4-PERSIST-001 binds the storage authority to the F63 sentinel-row pattern in the existing `score_events` table. M-4 does not migrate the schema.

---

## 2. Architecture

### 2.1 Layering authority — two new pure-data modules, hunt-site wiring

```
+------------------------------------------------------------+
|  Caller: agent/tools.py::run_module +                      |
|          core/console.py::_execute_hunt                    |
|                                                            |
|  1. pre_state = dossier/state.py.load_dossier_state(       |
|                   workspace_mgr)                           |
|                 or dossier/state.py.default_deferred_state()|
|  2. predictions = dossier/predictions.py.                  |
|                   load_predictions_log(workspace_mgr)      |
|  3. store_stix_objects(results, ...)                       |
|  4. per_ioc_events = ScoringEngine.score_results(...)      |
|     workspace_mgr.store_score_events(per_ioc_events)       |
|  5. post_state = infer_dossier_state_full(                 |
|                    scos_after, runs_after, notes_after)    |
|  6. slot_events = dossier/scoring.py.                      |
|       emit_dossier_slot_filled_events(pre_state, post_state)|
|  7. validation_results = dossier/predictions.py.           |
|       validate_predictions(predictions, scos_after,        |
|                            notes_after)                    |
|     prediction_events = [                                  |
|       dossier/scoring.py.                                  |
|         emit_dossier_prediction_validated_event(p)         |
|       for p, vr in zip(predictions, validation_results)    |
|       if vr.confirmed                                      |
|     ]                                                      |
|  8. workspace_mgr.store_score_events(                      |
|       slot_events + prediction_events)                     |
|  9. dossier/state.py.save_dossier_state(                   |
|       workspace_mgr, post_state)                           |
| 10. dossier/predictions.py.save_predictions_log(           |
|       workspace_mgr,                                       |
|       _mark_confirmed(predictions, validation_results))    |
| 11. Existing F62/F63 streak/milestone flow runs on         |
|     post_total = workspace.get_total_score(); milestones   |
|     can now fire from slot OR prediction events.           |
+------------------------------------------------------------+
```

**Two NEW pure-data modules:**
- `dossier/state.py` — single authority for *"how do we persist a DossierState across hunts?"*. Owns `load_dossier_state(workspace_mgr) -> DossierState | None`, `save_dossier_state(workspace_mgr, state) -> None`, `default_deferred_state() -> DossierState`. No score-event emission; no validation logic; no LLM-tool surface.
- `dossier/predictions.py` — single authority for *"what is a Predictions Log entry's lifecycle and how do we validate predictions against new evidence?"*. Owns `load_predictions_log(workspace_mgr) -> list[PredictionRecord]`, `save_predictions_log(workspace_mgr, predictions) -> None`, `validate_predictions(predictions, new_scos, new_notes) -> list[ValidationResult]`, helper `_mark_confirmed(predictions, results) -> list[PredictionRecord]`. Pure functions over the workspace API; no direct ScoreEvent construction (that stays in `dossier/scoring.py`).

**Two existing pure-function modules are UNCHANGED:**
- `dossier/scoring.py` — M-3 byte-identical. M-4 calls its two existing helpers but does not modify them. The scaffolded `emit_dossier_prediction_validated_event(prediction)` is finally exercised by `dossier/predictions.py` callers.
- `dossier/slot_inference.py` — M-2 byte-identical. M-4 reads its output; does not extend the API. **This is load-bearing:** extending `infer_dossier_state_full(...)` to accept a persistent-state hint would create a parallel-authority bug (two ways to compute a slot's status — fresh inference vs persisted snapshot). The persistent layer lives strictly above inference: callers compare the always-fresh `infer_dossier_state_full(...)` output against the persisted snapshot for the diff, and store the fresh output as the new snapshot.
- `dossier/panel.py` — M-1 byte-identical. The panel may render persistent state once `_execute_get_dossier_state` (the LLM tool) is wired to optionally load persistent state, but the panel module itself does not change.

**`gamification/scoring.py` is UNCHANGED.** M-3's per-IOC re-tune stays. `dossier_prediction_validated` is NOT a per-IOC scoring rule — it is an event subtype emitted directly by `dossier/scoring.py::emit_dossier_prediction_validated_event` (M-3 scaffold; M-4 caller). The `DEFAULT_RULES` table does not gain a new row.

**`core/workspace.py` is UNCHANGED (F59 + DEC-68 invariant).** The persistence pattern reuses the F63 sentinel-row precedent against the existing `store_score_events` API — see §2.2.

### 2.2 Storage authority — F63 sentinel-row pattern (DEC-M4-PERSIST-001)

F63 (DEC-63-MILESTONE-CATCHUP-001, landed `8778af3` 2026-05-26) established the precedent: workspace-scoped metadata can be persisted as a *reserved-action sentinel row* in the existing `score_events` table without any schema change. F63 stores `last_milestone_id` as `action="_milestone_sentinel"`, `points=0`, `indicator=str(milestone_id)`.

M-4 extends this pattern with **two new reserved actions**:

| reserved action | what it stores | column carrying the payload | uniqueness rule |
|-----------------|----------------|-----------------------------|-----------------|
| `_dossier_state_snapshot` | one full DossierState JSON blob | `indicator` (string) | exactly one row per workspace |
| `_predictions_log` | one JSON array of PredictionRecord entries | `indicator` (string) | exactly one row per workspace |

**Both upserts follow the F63 pattern exactly:** in a single SQLAlchemy session, `select` existing rows with the reserved action, `session.delete(...)` each, then `session.add(...)` a fresh row with the new payload. `points=0` so the sentinel never affects `get_total_score()`. `indicator` is a `Column(String, nullable=True)` in SQLite with no length cap — it accepts JSON of any size we will produce.

**Why this storage authority over alternatives:**

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **F63 sentinel-row in score_events** (recommended) | Two new reserved actions; JSON payload in `indicator`. | **accepted** | Zero schema change. Mirrors landed precedent the same week. No new SQLAlchemy model. Survives `ap chat` restart automatically because workspace SQLite is the same store. F59 + DEC-68 invariant preserved (workspace.py BYTEWISE UNCHANGED). |
| (b) NEW `workspace_metadata` table via `models/database.py` | Typed key/value SQLAlchemy model + new workspace helpers. | **rejected** | Violates DEC-DB-002 "no migrations" v1 discipline. Cleaner authority but requires `core/workspace.py` + `models/database.py` edits — both forbidden in M-4 scope. Defer to a future v2 hygiene slice if sentinel-row overload causes real downstream pain. |
| (c) `~/.ap/dossiers/<workspace_id>.json` flat file | File-based per-workspace JSON dossier. | **rejected** | Parallel authority with workspace SQLite — violates Sacred Practice 12 and DEC-WS-001 single-authority-for-workspace-state. Workspace export / clone / archive flows would silently lose the dossier. |

**Honest acknowledgement of the F63 overload:** the `indicator` column is documented as "The observable value (e.g. '1.2.3.4', 'evil.com'). For display only." M-4 widens that contract: for reserved actions starting with `_`, `indicator` may carry a JSON payload that the runtime (not the display layer) interprets. This is exactly the implicit contract F63 already established — F63's `_milestone_sentinel` stores `str(milestone_id)` in `indicator`, which is not "the observable value" either. M-4 makes the implicit contract explicit by reserving the `_`-prefix convention for runtime-payload actions and documenting all three reserved actions (`_milestone_sentinel`, `_dossier_state_snapshot`, `_predictions_log`) in one place (see §2.3). Future use of `_`-prefixed action names without planner re-stage is forbidden.

`get_recent_scores()` already filters out `_milestone_sentinel` (workspace.py:521). M-4 extends that filter to the two new reserved actions so the display layer still shows only real score events. This is the one **unavoidable** workspace.py read-only-filter change — DEC-M4-PERSIST-002 carves the narrow exception.

### 2.3 Reserved-action registry (DEC-M4-PERSIST-002)

Three reserved actions exist after M-4 ships. They are documented and constant-named in `dossier/state.py` (for the two NEW ones) and `core/workspace.py` (existing `_MILESTONE_SENTINEL_ACTION`); the read-side filter in `get_recent_scores()` is widened to a frozenset:

```python
# In core/workspace.py — minimal narrow change (DEC-M4-PERSIST-002)
_RESERVED_ACTIONS: frozenset[str] = frozenset({
    "_milestone_sentinel",            # F63 — last_milestone_id
    "_dossier_state_snapshot",        # M-4 — persistent DossierState
    "_predictions_log",               # M-4 — Predictions Log entries
})

# get_recent_scores filter
.where(ScoreEvent.action.notin_(_RESERVED_ACTIONS))
```

This is the single workspace.py edit M-4 authorizes. No public-method signature changes; no new SQLAlchemy column; no new helper methods; no schema migration. The F59 + DEC-68 invariant claim becomes: *"workspace.py public surface is BYTEWISE UNCHANGED apart from the narrow `get_recent_scores` filter widening and the new `_RESERVED_ACTIONS` constant, both of which preserve the read-side semantics of every existing caller (they never saw `_milestone_sentinel` either)."* Implementer must add `test_workspace.py` regression that proves `_dossier_state_snapshot` and `_predictions_log` rows are excluded from `get_recent_scores()`.

### 2.4 JSON serialization contract (DEC-M4-PERSIST-003)

Both `DossierState` and `PredictionRecord` are frozen-style dataclasses with simple primitive fields (status enums, strings, lists). M-4 ships **two small adapter functions** in `dossier/state.py` and `dossier/predictions.py`:

- `_serialize_dossier_state(state: DossierState) -> str` — produces a stable, JSON-formatted string. Enum members serialize as their `.value` (lowercase string); dataclass fields serialize as plain dicts; `slots: dict[DossierSlotName, SlotState]` serializes as `{"identity": {...}, "ttps": {...}, ...}`.
- `_deserialize_dossier_state(payload: str) -> DossierState` — parses JSON and reconstructs the DossierState; status strings re-promoted to `SlotStatus(...)` enum; slot keys re-promoted to `DossierSlotName(...)`. Unknown slot keys raise `ValueError` (loud failure, Sacred Practice 5).
- Analogous `_serialize_predictions(...)` / `_deserialize_predictions(...)` for `list[PredictionRecord]`.

Stable JSON output: keys sorted alphabetically; `indent=None` (compact form) to minimize column size; UTF-8. All four adapters live in their owning module (state.py owns DossierState; predictions.py owns PredictionRecord); no shared "dossier_serialization.py" cross-module helper (Sacred Practice 12 — each authority owns its own serialization).

**Versioning:** the JSON envelope carries a single `"schema_version": 1` field at the top level. Future schema changes bump the version and add a translator function; mismatched versions raise a loud `RuntimeError` with the diagnostic message `"persisted dossier schema version N is newer/older than runtime schema version 1; data was written by a different AP version"`. No silent fallback (Sacred Practice 5).

### 2.5 Predictions Log lifecycle and validation engine (DEC-M4-PRED-001..006)

#### 2.5.1 PredictionRecord schema extension (DEC-M4-PRED-001)

M-2's `PredictionRecord` (in `dossier/slots.py`) currently has two fields: `text: str`, `status: str = "pending"`. **`dossier/slots.py` is BYTEWISE UNCHANGED in M-4** (it is in the forbidden list — M-2 scaffold contract). The richer schema M-4 needs lives in **a new typed wrapper** in `dossier/predictions.py`:

```python
# In dossier/predictions.py
@dataclass
class ExpectedEvidence:
    """Typed match pattern for prediction validation (DEC-M4-PRED-002).

    M-4 vocabulary (minimum viable). M-5+ may extend.
    """
    sco_type: str | None = None
    """STIX SCO type the evidence must be (e.g., 'domain-name', 'ipv4-addr'). None = any type."""

    value_regex: str | None = None
    """Python regex the SCO value must match. None = no value filter."""

    asn_in: list[int] | None = None
    """For ipv4-addr / ipv6-addr / autonomous-system: ASN must be in this list. None = no ASN filter."""

    note_keyword_any: list[str] | None = None
    """At least one of these substrings must appear in an analyst note authored after the prediction. None = no note filter."""


@dataclass
class PersistedPrediction:
    """Workspace-persisted Predictions Log entry (DEC-M4-PRED-001).

    Extends M-2's PredictionRecord shape (text + status) with the M-4 lifecycle metadata.
    This is the JSON-serialised shape in the _predictions_log sentinel row.
    """
    prediction_id: str        # stable workspace-unique id, e.g. "pred-{8-char hex}"
    text: str
    slot: str                 # one of DossierSlotName values; the slot this prediction targets
    status: str               # "pending" | "validated" | "falsified"
    expected_evidence: ExpectedEvidence
    created_at: str           # ISO-8601 UTC timestamp
    validated_at: str | None  # ISO-8601 UTC; set when status -> validated
    validated_by_sco_id: str | None  # STIX object id of the confirming SCO
```

**M-2's `PredictionRecord` stays as the scaffold dataclass.** M-4's `PersistedPrediction` is the richer, persisted version. The `dossier/predictions.py` API converts between them at the workspace boundary; the LLM tool surface accepts the M-2 shape (`text` only) plus a slot + `expected_evidence` arg and produces a `PersistedPrediction` for storage. This keeps M-2's scaffold contract honest (DEC-M2-DOSSIER-004 preserved — the M-2 dataclass is byte-identical) while letting M-4 ship the richer shape it needs.

#### 2.5.2 `expected_evidence` match vocabulary (DEC-M4-PRED-002)

M-4 ships a deliberately small, typed match vocabulary:

| field | applies to | semantics | example |
|-------|-----------|-----------|---------|
| `sco_type` | any SCO | new SCO's STIX type must equal this string | `"domain-name"` |
| `value_regex` | any SCO with a string value | Python `re.search(...)` against the SCO's primary value field | `".*\\.ru$"` |
| `asn_in` | `ipv4-addr` / `ipv6-addr` / `autonomous-system` | ASN drawn from the SCO must be in this list | `[12345, 67890]` |
| `note_keyword_any` | analyst notes | at least one substring is present in a note authored after the prediction | `["pivoted", "ransomware"]` |

**All non-None fields are ANDed together.** Empty `expected_evidence` (all fields None) is rejected by `create_dossier_prediction` with a loud `ValueError` — predictions must have at least one match criterion.

This vocabulary is intentionally constrained. Richer matching (SCO relationship hops; multi-SCO patterns; numeric thresholds on `confidence` / `count`) lands in M-5+. The current vocabulary covers the M-4 user story (*"actor will pivot to .ru infrastructure"*, *"this actor will reuse ASN 12345"*, *"next hunt will surface a ransomware-noted IP"*) without expanding to a query DSL.

#### 2.5.3 Validation semantics (DEC-M4-PRED-003)

`validate_predictions(predictions, new_scos, new_notes) -> list[ValidationResult]`:

- Iterates `predictions` in stable list order.
- For each `pending` prediction: applies every non-None `expected_evidence` field against every entry in `new_scos` (or `new_notes` for `note_keyword_any`) until one entry matches all non-None fields, or all entries have been tried.
- Skips predictions whose `status` is already `validated` or `falsified` (idempotency).
- **Scope = current-hunt evidence only.** `new_scos` is the SCO set discovered *in this hunt*, not the full workspace history. This matches the M-3 wiring (slot-fill diff is also computed per-hunt against pre/post snapshots). Cross-hunt re-validation against the entire workspace is out of scope; if needed, M-5 introduces it via a `revalidate_all_predictions(workspace)` repair tool.

`ValidationResult` is a small dataclass:

```python
@dataclass
class ValidationResult:
    prediction_id: str
    confirmed: bool
    matched_sco_id: str | None  # STIX id of the confirming SCO; None if not confirmed
    rationale: str               # plain ASCII; safe for rule_description and LLM-events sidecar
```

The caller (`run_module` / `_execute_hunt`) filters `confirmed=True` results and asks `dossier/scoring.py.emit_dossier_prediction_validated_event(...)` for each. The scaffolded M-3 helper accepts a `PredictionRecord` (M-2 scaffold shape). `dossier/predictions.py` provides a one-liner adapter `_to_m2_record(persisted: PersistedPrediction) -> PredictionRecord` that returns a `PredictionRecord(text=persisted.text, status="validated")`. This keeps M-3's scaffold helper signature untouched.

#### 2.5.4 Falsification is deferred (DEC-M4-PRED-005)

**M-4 ships zero active-falsification logic.** Predictions stay `pending` until they validate. A prediction never auto-transitions to `falsified` in M-4. This is a deliberate scope cut because:

- Falsification semantics require either (a) a per-prediction "must validate within N hunts" window, or (b) typed `falsification_evidence` patterns (the contrapositive of `expected_evidence`). Both are richer than the M-4 minimum-viable confirmation engine.
- DEC-68-DOSSIER-REFRAME-007 (do falsified predictions deduct score?) hinges on which falsification model ships. M-4 stays in scope by deferring the model.
- M-5 owns the active-falsification slice and may revisit DEC-M4-PRED-006 at that time.

A user-facing manual-override LLM tool (`falsify_dossier_prediction(prediction_id, reason)`) is also **out of scope** for M-4. If the user wants to mark a prediction wrong before M-5 lands, they can edit the workspace SQLite directly (escape hatch documented in the per-slice plan, not in user-facing docs).

#### 2.5.5 Score event on validation (DEC-M4-PRED-004)

When `validate_predictions` returns `confirmed=True`, the caller emits a `dossier_prediction_validated` ScoreEvent via the M-3 scaffolded helper. Weight is `int(SLOT_WEIGHTS[DossierSlotName.PREDICTIONS])` = **4** (Phase 16 §3). `gamification/scoring.py::DEFAULT_RULES` is UNCHANGED — this event subtype is emitted directly with `points=4`, identical to how `dossier_slot_filled` events are emitted (M-3 pattern).

`indicator` for prediction events: M-3 scaffold uses `f"prediction:{hash(prediction.text) & 0xFFFFFFFF:08x}"`. M-4 keeps that exact format (no helper signature change) — the `PersistedPrediction.prediction_id` (e.g., `"pred-3f19d55c"`) is **not** passed through M-3's helper because the helper accepts the M-2 `PredictionRecord` (text-only) by contract. The two id forms are deterministically related but not identical; that is acceptable because the indicator field is for display attribution only — `prediction_id` is the persistent identity (used in the `_predictions_log` JSON payload).

#### 2.5.6 Falsified-prediction score deduction commit (DEC-M4-PRED-006)

DEC-68-DOSSIER-REFRAME-007 + DEC-M3-DOSSIER-005 deferred the question to M-4. M-4 commits: **confirmation = +N points; falsification = 0 points (no deduction).** Rationale:

- A score deduction would have to flow through `store_score_events` with a negative `points` value. The `ScoringRule` minimum / `streak_continued` math both assume non-negative event values; introducing negative events is a non-trivial blast radius that M-4 doesn't need to ship to unblock M-5+.
- The "reckless guessing should cost score" intuition is real, but the right gate is M-7 (reports + celebrations) — analysts who falsify a prediction in their report should receive narrative feedback, not silent score deduction.
- DEC-M4-PRED-006 is the binding decision. A future re-stage may revisit it if the falsification engine in M-5 surfaces evidence that requires score-level enforcement. Until then, M-4's "zero negative events" stance is canon.

### 2.6 LLM-tool surface: `create_dossier_prediction`

NEW LLM tool registered in `agent/tools.py::create_tools()` schema list + dispatched in `execute_tool()`. Implementation in a new `_execute_create_dossier_prediction(ctx, slot, text, expected_evidence)` helper. F64-clean: returns structured JSON text (the `PersistedPrediction.prediction_id` plus a short confirmation sentence); no Rich markup. The tool result text is **not** dossier-event text, so it does not need to be added to the `_DOSSIER_ACTIONS` filter (which targets ScoreEvent action strings, not tool output).

LLM-tool JSON schema (OpenAI function-calling format, DEC-AGENT-TOOLS-002 honored):

```json
{
  "type": "function",
  "function": {
    "name": "create_dossier_prediction",
    "description": "Author a prediction about the threat actor's next move. The prediction is tied to a dossier slot and validated against future hunt evidence. On confirmation, a dossier_prediction_validated score event fires.",
    "parameters": {
      "type": "object",
      "properties": {
        "slot": {"type": "string", "enum": ["identity", "ttps", "infrastructure", "timing", "targeting", "capability", "motivation", "predictions", "denial"]},
        "text": {"type": "string", "description": "Free-text prediction statement, e.g. 'Actor will pivot to .ru infrastructure within 7 days.'"},
        "expected_evidence": {
          "type": "object",
          "properties": {
            "sco_type": {"type": "string"},
            "value_regex": {"type": "string"},
            "asn_in": {"type": "array", "items": {"type": "integer"}},
            "note_keyword_any": {"type": "array", "items": {"type": "string"}}
          }
        }
      },
      "required": ["slot", "text", "expected_evidence"]
    }
  }
}
```

`get_dossier_state` (M-2 tool) is extended in one bounded way: it now reads the persistent state if present, falling back to a fresh inference if no snapshot exists yet. This preserves the M-2 tool's API contract (no signature change; same returned JSON shape) while honoring the M-4 state authority.

---

## 3. Removal targets (no parallel-authority residue)

- M-3 wired hunt sites already call `infer_dossier_state_full(...)` twice per hunt — once for `pre`, once for `post`. **M-4 removes the `pre` call** and replaces it with `load_dossier_state(workspace_mgr) or default_deferred_state()`. The `post` call stays (M-4 still needs a fresh inference of the post-hunt state to compute the diff and persist the new snapshot). **Net effect:** one fewer `infer_dossier_state_full` call per hunt; one new `load_dossier_state` call; one new `save_dossier_state` call.
- The Predictions slot's status was returning `DEFERRED` in M-2 (DEC-M2-DOSSIER-004). M-4 makes the slot return real `empty` / `partial` / `filled` based on the persisted predictions log: 0 entries = `empty`; ≥1 `pending` entries = `partial`; ≥2 `validated` entries = `filled`. **This makes `infer_dossier_state_full` for the Predictions slot a no-op stub today** — M-4 either (a) extends `slot_inference.py` with the Predictions reader (touches a forbidden file) or (b) computes Predictions-slot status in `dossier/state.py` and merges it into the DossierState returned by `infer_dossier_state_full`. M-4 chooses (b) per Sacred Practice 12: the persistent layer owns its slot. See §4 implementation note.

---

## 4. Implementation note: Predictions-slot status overlay

`infer_dossier_state_full(...)` in M-2 returns a `DossierState` with `predictions.status = DEFERRED`. M-4 must change that to real status without modifying `slot_inference.py` (forbidden). The hunt-site wiring computes:

```python
fresh_state = infer_dossier_state_full(scos_after, runs_after, notes_after)
predictions_log = load_predictions_log(workspace_mgr)
post_state = dossier.state.apply_predictions_overlay(fresh_state, predictions_log)
```

`apply_predictions_overlay` lives in `dossier/state.py`; it takes a DossierState and a predictions list, returns a new DossierState with the `predictions` slot's status set per the rules above. `slot_inference.py` stays byte-identical. M-3's `emit_dossier_slot_filled_events` then sees real Predictions-slot transitions (`deferred → partial`, `partial → filled`) and emits `dossier_slot_filled` events for them at weight 4 — which is the same weight as `dossier_prediction_validated`. This is intentional: filling the Predictions slot (by accumulating validated predictions) and validating individual predictions are two distinct scoreable events, both worth 4 points, both flowing through the M-3 emission path.

**M-3 transition guard:** `emit_dossier_slot_filled_events` explicitly skips DEFERRED-involving transitions (per its docstring lines 87–92). M-4's overlay flips the Predictions slot away from DEFERRED on first use; that triggers a one-time `_LOG.debug("...transitioned deferred->%s; skipping")` defensive-skip the first time the overlay runs against a pre-overlay snapshot. **This is expected and correct:** the first M-4 hunt after upgrade has a `pre_state` with DEFERRED Predictions (from a workspace that has no persisted snapshot yet, defaulting via `default_deferred_state()`) and a `post_state` with real `empty`/`partial`/`filled` Predictions (from the overlay). The defensive-skip is the right behavior — the M-3 emitter declined to award a transition score for the `deferred → real` migration. M-4 implementer must add a regression test that asserts this and documents the one-time silence.

---

## 5. Snapshot survives `ap chat` restart (the load-bearing acceptance test)

The compound integration test that proves M-4 ships is:

1. Fresh workspace, no persisted snapshot.
2. `ap chat`, run one identity-flavored hunt (e.g., `cti/otx_pulses` against a hash with x509 in results). Slot Identity transitions `empty → partial`. Predictions slot transitions `deferred → empty` (overlay first-fire — no score event per §4). New snapshot persisted.
3. Author one prediction via `create_dossier_prediction(slot="infrastructure", text="actor pivots to .ru", expected_evidence={"value_regex": ".*\\.ru$"})`. Predictions slot transitions `empty → partial` (1 pending) — `dossier_slot_filled` event fires at weight 4.
4. **Quit `ap chat`** (process exits cleanly).
5. **Restart `ap chat`** in the same workspace.
6. Run a second hunt that surfaces a `.ru` domain SCO. The persisted prediction is loaded, validated (confirmed=True), and a `dossier_prediction_validated` ScoreEvent fires at weight 4.
7. The persisted DossierState now has Predictions slot `partial` (still 1 validated; not yet 2 for `filled`).
8. `dossier` meta-command renders the panel with the validated prediction visible; `get_total_score()` includes both slot-fill and prediction-validated points; F63 milestone catch-up considers the new total.

This is the "persistent state survives ap chat restart" acceptance from the dispatch context; it is **mandatory** in the Evaluation Contract (§7).

---

## 6. Invariant preservation matrix

| invariant | scope | M-4 check |
|-----------|-------|-----------|
| F59 (workspace single authority for SCO persistence) | `core/workspace.py` | One narrow widening (read filter + `_RESERVED_ACTIONS` constant) per DEC-M4-PERSIST-002. No schema change, no new public method, no new SQLAlchemy model. Test gate: `test_workspace.py` regression proving `_dossier_state_snapshot` + `_predictions_log` rows are excluded from `get_recent_scores`. |
| F60 (auto-pivot policy + event bus invariants) | `core/pivot_policy.py`, `core/event_bus.py` | BYTEWISE UNCHANGED. M-4 does not introduce a new bus subscriber. `dossier/predictions.py::validate_predictions` is a pure function called inline from hunt sites — no event-bus wiring. |
| F62 (StreakManager single authority; `streak_continued` semantics) | `core/streak.py`, F62 tests | BYTEWISE UNCHANGED. M-4's new score events flow through the existing `store_score_events` path; F62 streak logic continues to see them as ordinary points. |
| F63 (milestone catch-up + sentinel-row pattern) | `gamification/celebrations.py` | UNCHANGED. F63 milestone-catch-up sees the higher `post_total` after prediction events; an integration test asserts a single hunt can fire a milestone via a prediction-validated event. |
| F64 (de-duplicate LLM narration vs Rich panel) | `agent/tools.py::_DOSSIER_ACTIONS` filter | M-4 does NOT add a third action key. The existing filter `{"dossier_slot_filled", "dossier_prediction_validated"}` (tools.py:665) already covers both. Test gate: integration test asserts neither slot-fill nor prediction-validated text appears in LLM tool `summary`. |
| Sacred Practice 12 (one authority per operational fact) | new + existing | `dossier/state.py` is sole authority for persistent DossierState; `dossier/predictions.py` is sole authority for PredictionRecord lifecycle; `dossier/scoring.py` is sole authority for `dossier_*` ScoreEvent shape; `core/workspace.py` is sole persistence authority. No fact has two owners. |
| DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 | `dossier/slots.py` | BYTEWISE UNCHANGED. Predictions weight stays 4.0; the new prediction-validated event reads it via `SLOT_WEIGHTS[DossierSlotName.PREDICTIONS]`. |
| DEC-M2-DOSSIER-004 (PredictionRecord scaffold contract) | `dossier/slots.py::PredictionRecord` | BYTEWISE UNCHANGED. M-4's richer `PersistedPrediction` lives in `dossier/predictions.py`; the M-2 scaffold dataclass is untouched. |

---

## 7. Evaluation Contract (9-key, ~35–45 tests)

**required_tests:**

The implementer ships ~35–45 tests across these files. Counts are minimums; full coverage will land above each.

- `tests/test_dossier_state.py` **(NEW, ~12 tests)**:
  - load returns None on fresh workspace (1)
  - default_deferred_state returns valid DossierState with all 9 slots present (1)
  - save then load round-trips (1)
  - second save replaces first (sentinel uniqueness: exactly one `_dossier_state_snapshot` row after N writes) (1)
  - JSON deserializer rejects unknown slot key with loud ValueError (1)
  - JSON deserializer rejects unknown SlotStatus value with loud ValueError (1)
  - schema_version=1 round-trips; schema_version=2 raises loud RuntimeError (1)
  - apply_predictions_overlay: 0 entries → `empty` (1)
  - apply_predictions_overlay: 1 pending → `partial` (1)
  - apply_predictions_overlay: 2 validated → `filled` (1)
  - apply_predictions_overlay: mixed (1 validated + 3 pending) → `partial` (1)
  - apply_predictions_overlay: does not mutate input state (frozen dataclass discipline) (1)

- `tests/test_dossier_predictions.py` **(NEW, ~14 tests)**:
  - load returns empty list on fresh workspace (1)
  - save then load round-trips (1)
  - second save replaces first (sentinel uniqueness for `_predictions_log`) (1)
  - validate_predictions: empty list → empty results (1)
  - validate_predictions: sco_type-only match → confirmed (1)
  - validate_predictions: value_regex-only match → confirmed (1)
  - validate_predictions: asn_in match against ipv4-addr SCO with ASN in extension → confirmed (1)
  - validate_predictions: note_keyword_any match against note text → confirmed (1)
  - validate_predictions: mixed sco_type + value_regex (both must match) → confirmed (1)
  - validate_predictions: mixed sco_type + value_regex (only one matches) → not confirmed (1)
  - validate_predictions: no SCOs in new_scos → no confirmations (1)
  - validate_predictions: already-validated prediction skipped (idempotent) (1)
  - validate_predictions: already-falsified prediction skipped (M-4 never sets this; defensive guard) (1)
  - create with empty expected_evidence raises ValueError (1)

- `tests/test_dossier_scoring.py` **(EXTEND, +3 tests)**:
  - prediction-validated event fires from confirmed validation with points=4, action="dossier_prediction_validated", indicator format matches M-3 scaffold (1)
  - prediction-validated event fires alongside slot-fill events in the same hunt (1)
  - no prediction event fires when validate_predictions returns 0 confirmations (1)

- `tests/test_workspace.py` **(EXTEND, +3 tests)**:
  - `_dossier_state_snapshot` row excluded from `get_recent_scores()` (1)
  - `_predictions_log` row excluded from `get_recent_scores()` (1)
  - `_RESERVED_ACTIONS` constant covers all three reserved actions (regression guard: forces a code change if a future implementer adds a fourth without updating the constant) (1)

- `tests/test_agent_tools.py` **(EXTEND, +5 tests)**:
  - `create_dossier_prediction` tool schema present in `create_tools()` output (1)
  - `create_dossier_prediction` execution path persists a PersistedPrediction and returns the prediction_id (1)
  - hunt with a persisted prediction that matches new SCOs fires `dossier_prediction_validated` event (compound) (1)
  - F64 gate: prediction-validated event text absent from `result["summary"]` (1)
  - hunt-site `pre_state` loads from persistent snapshot, not from `infer_dossier_state_full` (verify by mock-patching `infer_dossier_state_full` to count calls — expect exactly ONE call per hunt after M-4, was TWO in M-3) (1)

- `tests/test_scoring.py` **(EXTEND, +2 tests)**:
  - F62 regression: prediction-validated event participates in streak chain (does not reset it) (1)
  - F63 regression: prediction-validated points contribute to milestone seed (1)

- `tests/test_dossier_get_state_tool.py` **(EXTEND, +1 test)**:
  - `get_dossier_state` returns persisted state when present; falls back to fresh inference when no snapshot (1)

- **NEW integration test: `tests/test_dossier_persistence_integration.py`** (~3 tests):
  - The §5 compound integration story above (1 large test or split into 3 stages) (3)

**Total: ~43 new + extended tests.** Full suite green: ≥1984 passed (matching the C-2 / M-3 baseline) plus the new M-4 tests, minus any duplicate skips. Implementer must report the actual pre/post test counts in the readiness summary.

**required_evidence:**
- Full pytest output green for the worktree.
- `git diff main -- src/adversary_pursuit/core/workspace.py` shows only the narrow DEC-M4-PERSIST-002 changes (filter widening + `_RESERVED_ACTIONS` constant). Implementer pastes the diff.
- `git diff main -- src/adversary_pursuit/dossier/scoring.py` is empty.
- `git diff main -- src/adversary_pursuit/dossier/slot_inference.py` is empty.
- `git diff main -- src/adversary_pursuit/dossier/slots.py` is empty.
- `git diff main -- src/adversary_pursuit/dossier/panel.py` is empty.
- `git diff main -- src/adversary_pursuit/gamification/scoring.py` is empty.
- `git diff main -- src/adversary_pursuit/gamification/celebrations.py` is empty.
- `git diff main -- src/adversary_pursuit/core/streak.py` is empty.
- `git diff main -- src/adversary_pursuit/core/pivot_policy.py` is empty.
- `git diff main -- src/adversary_pursuit/core/event_bus.py` is empty.
- Demo trace (or test transcript) showing the §5 restart-survival scenario: snapshot persists across two distinct `ap chat` invocations against the same workspace.

**required_authority_invariants:**
- F59: `core/workspace.py` changes limited to the narrow DEC-M4-PERSIST-002 read-side filter + `_RESERVED_ACTIONS` constant; no schema migrations; no new public method; no new model column. Test gate proves the filter excludes both new sentinel actions.
- F60: `core/pivot_policy.py` + `core/event_bus.py` BYTEWISE UNCHANGED; no new event-bus subscriber.
- F62: `core/streak.py` + `streak_continued` semantics UNCHANGED; new prediction events do not reset streaks; regression test included.
- F63: `gamification/celebrations.py` UNCHANGED; milestone catch-up integration test asserts a prediction event can trigger a milestone.
- F64: `_DOSSIER_ACTIONS` filter at `agent/tools.py:665` covers both `dossier_slot_filled` AND `dossier_prediction_validated`; integration test asserts prediction event text absent from LLM summary.
- Sacred Practice 12: per the §6 invariant matrix.
- DEC-M1-SLOTS-WEIGHT-AUTHORITY-001: `SLOT_WEIGHTS` constants in `dossier/slots.py` UNCHANGED.
- DEC-M2-DOSSIER-004: M-2's `PredictionRecord` scaffold dataclass in `dossier/slots.py` UNCHANGED; M-4's `PersistedPrediction` lives in `dossier/predictions.py`.
- DEC-M3-DOSSIER-001..005: M-3's `dossier/scoring.py` UNCHANGED; M-4 wires the scaffolded prediction-validated emitter without modifying the helper.

**required_integration_points:**
- `dossier/state.py` (NEW pure-data module — DossierState persistence + overlay).
- `dossier/predictions.py` (NEW pure-data module — PredictionRecord lifecycle + validation engine).
- `dossier/__init__.py` (export new symbols: `load_dossier_state`, `save_dossier_state`, `default_deferred_state`, `apply_predictions_overlay`, `load_predictions_log`, `save_predictions_log`, `validate_predictions`, `PersistedPrediction`, `ExpectedEvidence`, `ValidationResult`).
- `agent/tools.py` (extend `run_module` hook per §2.1 wiring; register `create_dossier_prediction` LLM tool; extend `_execute_get_dossier_state` to read persistent state with fresh-inference fallback).
- `core/console.py` (mirror `_execute_hunt` wiring; same pattern as M-3 mirror).
- `core/workspace.py` (NARROW: `_RESERVED_ACTIONS` frozenset + `get_recent_scores` filter widening per DEC-M4-PERSIST-002).

**forbidden_shortcuts:**
- NO env-var bypass of persistence.
- NO "always-re-infer" fallback flag if the snapshot is missing — fresh workspaces use `default_deferred_state()` and write a snapshot on first hunt; that is the only path.
- NO new event-bus subscriber.
- NO schema migration; NO `models/database.py` edits; NO new SQLAlchemy column or model.
- NO new public method on `WorkspaceManager`; the entire workspace surface change is the read-side filter widening and the `_RESERVED_ACTIONS` constant.
- NO modification of `dossier/scoring.py`, `dossier/slot_inference.py`, `dossier/slots.py`, `dossier/panel.py`, `gamification/scoring.py`, `gamification/celebrations.py`, `core/streak.py`, `core/pivot_policy.py`, `core/event_bus.py`.
- NO Rich markup in dossier event text or in `create_dossier_prediction` tool output (F64).
- NO active falsification logic (M-5 owns; DEC-M4-PRED-005).
- NO negative-points ScoreEvent emission (DEC-M4-PRED-006).
- NO double-persist of dossier events.
- NO extension of `infer_dossier_state_full(...)` signature to accept a persistent-state hint (parallel-authority risk).
- NO refactor of `tools.py` or `console.py` beyond the documented wiring + new LLM tool registration.

**rollback_boundary:** single feature branch revertible as one merge commit. Revert restores M-3 byte state, removes `dossier/state.py` + `dossier/predictions.py` + their re-exports, restores `tools.py` / `console.py` / `workspace.py` to M-3 byte state. Historical `_dossier_state_snapshot` and `_predictions_log` sentinel rows in `score_events` remain after revert — they are valid rows with `points=0`, so they do not affect `get_total_score()`. The revert log notes that workspaces written by M-4 will accumulate orphan sentinel rows after revert; a manual cleanup is the documented mitigation (one-line SQL `DELETE FROM score_events WHERE action LIKE '\\_dossier%' OR action = '\\_predictions_log';`). No schema migrations, no settings changes, streak.json untouched.

**ready_for_guardian_definition:**
- All required_tests green; full suite green.
- All forbidden-file `git diff main` outputs empty (paste each verifying the file is byte-identical).
- `core/workspace.py` diff limited to the DEC-M4-PERSIST-002 narrow change (paste the diff; reviewer confirms scope match).
- Phase 17G appended to `MASTER_PLAN.md` AND committed in the same commit as source (AP #74 orphan-prevention; M-3 demonstrated the pattern works at `974fa1a`).
- Phase 17F status flipped: `in-progress` → `completed (landed 2026-06-01, merge 2809b13, impl 974fa1a)`. M-3 closeout drift fixed in this commit.
- "Active Phase Pointer" tail-line updated from `W-68-M3-DOSSIER-SCORING` to `W-68-M4-PERSISTENT-DOSSIER`.
- Plan Status table gains a Phase 17G row.
- `dossier/__init__.py` exports the M-4 public symbols listed under required_integration_points; no surprise additions.
- Implementer commit message follows `feat(dossier):` Phase 17 prefix, references `#68` + `DEC-M4-PERSIST-001..003` + `DEC-M4-PRED-001..006`.

---

## 8. Scope Manifest

**Allowed / Required (the implementer MUST touch these):**
- `src/adversary_pursuit/dossier/state.py` **(NEW)**
- `src/adversary_pursuit/dossier/predictions.py` **(NEW)**
- `src/adversary_pursuit/dossier/__init__.py` (export new symbols)
- `src/adversary_pursuit/agent/tools.py` (hunt-site wiring per §2.1; new `create_dossier_prediction` LLM tool + dispatcher; bounded extension of `_execute_get_dossier_state`)
- `src/adversary_pursuit/core/console.py` (hunt-site wiring mirror)
- `src/adversary_pursuit/core/workspace.py` **(NARROW per DEC-M4-PERSIST-002 only — `_RESERVED_ACTIONS` constant + `get_recent_scores` filter widening; reviewer enforces minimal diff)**
- `tests/test_dossier_state.py` **(NEW)**
- `tests/test_dossier_predictions.py` **(NEW)**
- `tests/test_dossier_persistence_integration.py` **(NEW)**
- `tests/test_dossier_scoring.py` (extend)
- `tests/test_agent_tools.py` (extend)
- `tests/test_workspace.py` (extend)
- `tests/test_scoring.py` (extend — F62/F63 regression)
- `tests/test_dossier_get_state_tool.py` (extend — persistent-state read)
- `MASTER_PLAN.md` — Phase 17G section + Phase 17F status flip + Plan Status table row + "Active Phase Pointer" tail-line update. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention; M-3 demonstrated the pattern works).**

**Forbidden (preserved authorities):**
- `src/adversary_pursuit/dossier/scoring.py` (M-3 byte-identical)
- `src/adversary_pursuit/dossier/slot_inference.py` (M-2 byte-identical)
- `src/adversary_pursuit/dossier/slots.py` (M-1/M-2 byte-identical — `SLOT_WEIGHTS` + `PredictionRecord` scaffold preserved)
- `src/adversary_pursuit/dossier/panel.py` (M-1 byte-identical)
- `src/adversary_pursuit/gamification/scoring.py` (M-3 byte-identical — no new `DEFAULT_RULES` row for prediction events)
- `src/adversary_pursuit/gamification/celebrations.py` (F63 invariant)
- `src/adversary_pursuit/core/streak.py` (F62 invariant)
- `src/adversary_pursuit/core/pivot_policy.py` (F60 invariant)
- `src/adversary_pursuit/core/event_bus.py` (F60 invariant — no new bus subscriber)
- `src/adversary_pursuit/models/database.py` (no schema change; DEC-DB-002)
- `src/adversary_pursuit/gamification/modes.py`, `src/adversary_pursuit/agent/runner.py`, `src/adversary_pursuit/agent/chat.py` (C-1/C-2 territory; F64 panel separation)
- `src/adversary_pursuit/models/**` (apart from the above database.py forbidden, no other model edits)
- `src/adversary_pursuit/modules/**` (no module changes)
- `pyproject.toml`, hooks, settings, `CLAUDE.md`, `agents/`, `runtime/`

**Expected state authorities touched:**
- workspace SQLite `score_events` table (read + sentinel-row upsert via existing `store_score_events` + new `_RESERVED_ACTIONS` filter)
- in-memory `DossierState` (read at hunt start from persisted snapshot; written at hunt end via fresh inference + overlay)
- in-memory `list[PersistedPrediction]` (read at hunt start; mutated for confirmations; written at hunt end)

---

## 9. Decision Log (Phase 17G / M-4 binding)

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M4-PERSIST-001** | Persistent DossierState + Predictions Log storage authority is the F63 sentinel-row pattern in the existing `score_events` table. Two new reserved actions (`_dossier_state_snapshot`, `_predictions_log`) carry JSON payloads in the `indicator` column. No schema change, no new SQLAlchemy model, no new `models/database.py` edits. | Mirrors the landed F63 precedent (DEC-63-MILESTONE-CATCHUP-001, merge `8778af3`). Zero schema migration risk. Persists in the workspace SQLite store so it survives `ap chat` restart and travels with workspace export. Rejected alternatives: NEW `workspace_metadata` table (cleaner authority, but violates DEC-DB-002 "no migrations" and forbidden `models/database.py` edits) and flat `~/.ap/dossiers/<id>.json` files (parallel authority with workspace SQLite — violates Sacred Practice 12). |
| **DEC-M4-PERSIST-002** | `core/workspace.py` gains exactly two narrow changes: (a) a module-level `_RESERVED_ACTIONS` frozenset enumerating `_milestone_sentinel` + the two new M-4 reserved actions; (b) `get_recent_scores()` `.where(...)` clause widened from `ScoreEvent.action != _MILESTONE_SENTINEL_ACTION` to `ScoreEvent.action.notin_(_RESERVED_ACTIONS)`. No public-method signature change, no new column, no schema migration. F59 invariant claim becomes: workspace public surface preserves all existing caller semantics; M-4 widens an existing display filter from one action to three. | DEC-M4-PERSIST-001 picked the F63 sentinel-row pattern; that mechanically requires hiding the new sentinel rows from `get_recent_scores()` the same way F63 hides `_milestone_sentinel`. Widening F63's existing filter is the smallest honest workspace.py change that preserves caller semantics. Implementer MUST keep the diff minimal; reviewer enforces. |
| **DEC-M4-PERSIST-003** | JSON envelope for both reserved actions carries `"schema_version": 1`. Serializers live in the owning module (`dossier/state.py` for DossierState, `dossier/predictions.py` for PredictionRecord lists). Mismatched versions raise loud `RuntimeError` (Sacred Practice 5 — no silent fallback). Stable sorting: keys sorted alphabetically; compact form. | Future M-4+ schema evolution needs an explicit handshake; loud failure tells the user "you upgraded AP and your workspace pre-dates the change" rather than silently reading garbage. Per-module serializers honor Sacred Practice 12 (each authority owns its own serialization; no cross-module helper). |
| **DEC-M4-PRED-001** | Predictions Log persists `PersistedPrediction` (NEW in `dossier/predictions.py`) — a richer typed dataclass with `prediction_id`, `slot`, `expected_evidence`, `created_at`, `validated_at`, `validated_by_sco_id`. M-2's `PredictionRecord` (scaffold dataclass in `dossier/slots.py`) stays BYTEWISE UNCHANGED. `dossier/predictions.py` provides a one-way adapter `_to_m2_record(persisted: PersistedPrediction) -> PredictionRecord` that lets the M-3 scaffolded `emit_dossier_prediction_validated_event` helper accept the M-2 shape without signature change. | DEC-M2-DOSSIER-004 ratified the M-2 scaffold as the long-lived shape contract; modifying `dossier/slots.py` would break that contract. Keeping the richer M-4 shape in `dossier/predictions.py` honors Sacred Practice 12 (the persistence-layer module owns the persistence-layer schema). The adapter preserves the M-3 helper signature so M-3's scaffold is truly the contract M-4 targets — not an aspirational shape that M-4 had to renegotiate. |
| **DEC-M4-PRED-002** | `expected_evidence` validation vocabulary v1.0 is `ExpectedEvidence(sco_type, value_regex, asn_in, note_keyword_any)`. All non-None fields are ANDed. Empty `expected_evidence` is rejected by `create_dossier_prediction` with loud `ValueError`. Richer matching (multi-SCO patterns, relationship-hop queries, numeric thresholds) deferred to M-5+. | Smallest vocabulary that covers the M-4 user story (actor pivot to .ru, ASN reuse, keyword in note) without becoming a query DSL. Typed dataclass over freeform dict keeps the LLM-tool schema honest and validation predictable. |
| **DEC-M4-PRED-003** | Validation scope is **current-hunt evidence only**: `validate_predictions(predictions, new_scos, new_notes)` matches against the SCOs / notes surfaced in the current hunt, not the full workspace history. Predictions already `validated` or `falsified` are skipped (idempotency). | Matches M-3's per-hunt diff pattern; avoids accidental re-validation against unchanged history. Cross-hunt re-validation, if needed later, is a separate M-5+ tool (`revalidate_all_predictions(workspace)`). |
| **DEC-M4-PRED-004** | When validation returns `confirmed=True`, the caller fires `dossier/scoring.py::emit_dossier_prediction_validated_event(_to_m2_record(persisted))` with weight 4 (Phase 16 §3 SLOT_WEIGHTS[PREDICTIONS]). `gamification/scoring.py::DEFAULT_RULES` is UNCHANGED — prediction-validated events are emitted directly with `points=4`, mirroring M-3's `dossier_slot_filled` emission pattern. | Honors DEC-M3-DOSSIER-003 (ScoringEngine unchanged in behavior). Honors DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 (SLOT_WEIGHTS is the single weight authority). Emission pattern is symmetric with slot-fill events — both flow through `store_score_events` as ordinary score-event dicts. |
| **DEC-M4-PRED-005** | Active falsification is **out of scope for M-4**. Predictions transition from `pending` to `validated` only; M-4 ships no auto-falsify rules, no per-prediction "must validate within N hunts" window, no manual-override LLM tool. M-5 owns the falsification slice. | Falsification semantics require either a typed `falsification_evidence` shape or a temporal window; both are non-trivial design surfaces that would inflate M-4. Deferring lets M-5 design the falsification engine alongside the Denial / Deception slot work it already owns. Workspaces written by M-4 record predictions as `pending`; M-5 can falsify them retroactively without a schema change. |
| **DEC-M4-PRED-006** | DEC-68-DOSSIER-REFRAME-007 + DEC-M3-DOSSIER-005 falsified-prediction score deduction question is **committed**: confirmation = +N points (where N = SLOT_WEIGHTS[PREDICTIONS] = 4); falsification = 0 points (no deduction). M-4 ships zero negative-event logic. | Negative `points` events would have to flow through `store_score_events` and through `streak_continued` math; both currently assume non-negative event values. The "reckless guessing should cost score" intuition is real but the right surface is M-7 narrative feedback, not silent score deduction. A future re-stage may revisit this if M-5's falsification engine surfaces evidence requiring score-level enforcement; until then this DEC is canon. |

---

## 10. Open question for the user (none)

No user-decision boundary is required to start the implementer. The DEC-68-DOSSIER-REFRAME-007 deferred question is committed by DEC-M4-PRED-006 (no negative-event logic). Storage authority is committed by DEC-M4-PERSIST-001. All other design surfaces are settled by the M-3 scaffold contract or by the existing F59/F60/F62/F63/F64 invariants.

If implementation surfaces an unforeseen blast-radius (e.g., the `_RESERVED_ACTIONS` widening triggers a regression in a currently-green test elsewhere), the implementer halts and reports — that is a planner re-stage trigger, not an in-flight design call.

---

## 11. Subsequent Workflow Cue

After M-4 lands, the recommended next workflow is **M-5 — Denial / Deception Strategies (slot 9) + User-Note Surface** per `.claude/plans/dossier-reframe-v2-roadmap.md` §M-5. M-5 introduces `dossier note` meta-command + `add_dossier_strategy` LLM tool + cross-evidence linkage, and is the natural place to revisit DEC-M4-PRED-005 (active falsification) since the user-note surface it introduces could carry analyst-authored "this prediction was wrong because X" notes that feed an extended falsification engine. M-6 (dossier-aware auto-pivot) is independent of M-5 once M-4 persistence lands and may be scheduled in parallel.

C-3 (Philosophy + Bureaucratese modes) remains independent of the dossier roadmap (DEC-30-CHARACTER-V2-007) and may land in any wave.
