# M-5 — Denial / Deception Strategies (slot 9) + User-Note Authoring Surface + Active Falsification Engine (per-slice plan)

**Status:** planner-staged 2026-06-07 by W-68-M5-DENIAL-STRATEGIES planner stage. Implementer slice to follow.
**Workflow:** `w-68-m5-denial-strategies`
**Goal:** `g-68-m5-denial`
**Work item to dispatch:** `wi-68-m5-impl-01`
**Drives:** Phase 17H of `MASTER_PLAN.md`. Phase 17H carries the binding decisions and slice index; this document carries full rationale, vocabulary tables, hunt-site diff sketches, and decomposition detail. When the two diverge, Phase 17H wins for binding decisions; this document wins for narrative and table detail.

**Inherits from:** Phase 16 §M-5, `.claude/plans/dossier-reframe-v2-roadmap.md` §M-5. Phase 17B (M-1 panel), Phase 17D (M-2 extractors + scaffold dataclasses), Phase 17F (M-3 scoring), Phase 17G (M-4 persistence + Predictions auto-validation) are prerequisites; all shipped by 2026-06-02. Worktree base: AP main `cfafd6a` (M-4 landed at merge `f928149`, impl `1b1a2b0`).

---

## 1. Goal (single paragraph)

Three interlocking surfaces ship in this slice:

1. **Denial / Deception slot 9 inference** — replace the `DEFERRED` stub for slot 9 in `slot_inference.py` with a real extractor that pattern-matches SCOs + analyst notes for denial / deception indicators (DGA-shaped domains, fast-flux DNS hints, decoy / sandbox-evasion keywords in notes). Slot 9 status transitions `deferred → empty / partial / filled` based on evidence; M-3's `emit_dossier_slot_filled_events` already handles the transition, so this slot becomes a real scored slot at weight 2.5 (Phase 16 §3) with zero changes to `dossier/scoring.py`.

2. **User-note authoring surface** — a chat meta-command `note <text>` (intercepted in `agent/chat.py`, parity with the existing `dossier` meta-command) plus a `create_dossier_note(text)` LLM tool in `agent/tools.py`. Both ride on the existing `WorkspaceManager.add_note()` public method (workspace.py:647) and the existing `AnalystNote` SQLAlchemy table (`models/database.py:218`). No schema change; no new model; no new persistence layer. Notes flow into the existing `_read_analyst_notes` reader and immediately become evidence for the M-4 `note_keyword_any` validation clause and the new M-5 falsification engine.

3. **Active falsification engine** — typed `FalsificationEvidence` dataclass in `dossier/predictions.py` (mirroring `ExpectedEvidence`) plus a new `falsify_predictions(predictions, new_scos, new_notes, hunt_count)` function that transitions `pending → falsified` when either (a) typed contradiction evidence appears or (b) the prediction has been pending for N or more hunts (`stale_after_n_hunts` window, default null = no staleness rule). Falsification persists via the existing `_predictions_log` sentinel row (M-4 storage authority). On transition, a new `dossier_prediction_falsified` ScoreEvent at points=0 is emitted (DEC-M4-PRED-006: confirmation=+4, falsification=0, no negative-points events). A manual-override LLM tool `falsify_dossier_prediction(prediction_id, reason)` lets the analyst mark a prediction wrong without waiting for auto-falsify evidence.

After M-5, the dossier puzzle has 8 of 9 slots fillable from automated inference (Targeting remains DEFERRED — its inference path requires user-supplied victim-industry context, deferred to a future slice), user notes are first-class evidence the falsification engine can act on, and predictions have a complete lifecycle (`pending → validated` OR `pending → falsified`) rather than the M-4 "predictions accumulate forever as pending" pattern.

**Out-of-scope (explicit, deferred):**
- **Targeting slot 5 inference** — its evidence is industry / geography / victim selection, which currently has no automated extractor. Stays DEFERRED until a future slice introduces a user-supplied victim profile or pulls industry context from SCOs that AP modules do not currently surface.
- **Dossier-aware auto-pivot policy** — M-6 owns.
- **Reports / celebrations / badges narrative upgrades** — M-7 owns.
- **Richer denial extractor vocabulary** — M-5 ships a deliberately small denial vocabulary (DGA shape, fast-flux hint, decoy/evasion note keywords). Multi-stage TTP cross-reference, sandbox-detection IOCs from SCO extensions, registrar-rotation behavior, etc., land in M-7 or later if the M-5 vocabulary proves too narrow against real workspaces.
- **Cross-hunt prediction revalidation** — M-5 keeps validation + falsification at current-hunt scope (DEC-M4-PRED-003 preserved). A `revalidate_all_predictions(workspace)` tool that re-scans the full workspace history is out of scope; if needed it lands as a small standalone slice after M-5.
- **No new SQLite tables, no `models/database.py` changes, no new SQLAlchemy model** — DEC-M5-NOTE-001 binds notes to the existing `AnalystNote` table + existing `add_note()` API; DEC-M5-FALSIFY-001 binds the falsified-predictions sentinel storage to the existing `_predictions_log` sentinel row (no second sentinel action needed).
- **No negative-points ScoreEvent emission** — DEC-M4-PRED-006 stays canon. The falsification event fires at points=0; the "reckless guessing should cost score" intuition is M-7 narrative-feedback territory.
- **No richer `ExpectedEvidence` vocabulary** — DEC-M4-PRED-002 v1.0 vocabulary stays frozen (`sco_type, value_regex, asn_in, note_keyword_any`). `FalsificationEvidence` is a NEW dataclass with its own (closely-mirrored) vocabulary.

---

## 2. Architecture

### 2.1 Layering authority — extend three existing modules, no new modules

```
+------------------------------------------------------------+
|  Caller: agent/tools.py::run_module +                      |
|          core/console.py::_execute_hunt                    |
|                                                            |
|  M-4 wiring (UNCHANGED in M-5):                            |
|    1. pre_dossier = load_dossier_state() or default        |
|    2. predictions_log = load_predictions_log()             |
|    3. notes_before = _read_analyst_notes()  (AnalystNote)  |
|    4. store_stix_objects(...)                              |
|    5. per_ioc_events = ScoringEngine.score_results(...)    |
|    6. fresh_post = infer_dossier_state_full(scos, runs,    |
|                       notes_before)                        |
|       [M-5: slot 9 (Denial) now returns real status]       |
|    7. post_dossier = apply_predictions_overlay(            |
|                       fresh_post, predictions_log)         |
|    8. dossier_events = emit_dossier_slot_filled_events(    |
|                          pre_dossier, post_dossier)        |
|       [M-5: slot 9 transitions now emit at +2 points]      |
|    9. validation_results = validate_predictions(...)       |
|       prediction_events = [emit_..._validated_event(...)   |
|                            for confirmed]                  |
|                                                            |
|  M-5 NEW STEPS:                                            |
|   10. falsification_results =                              |
|         falsify_predictions(predictions_log,               |
|                             new_scos_this_hunt,            |
|                             notes_before,                  |
|                             current_hunt_index)            |
|   11. falsification_events = [                             |
|         emit_dossier_prediction_falsified_event(p)         |
|         for p, fr in zip(predictions_log,                  |
|                          falsification_results)            |
|         if fr.falsified                                    |
|       ]                                                    |
|   12. all_dossier_events = dossier_events                  |
|         + prediction_events + falsification_events         |
|       workspace_mgr.store_score_events(all_dossier_events) |
|   13. updated_predictions = mark_confirmed_or_falsified(   |
|         predictions_log, validation_results,               |
|         falsification_results)                             |
|       save_predictions_log(workspace_mgr,                  |
|                            updated_predictions)            |
|   14. save_dossier_state(workspace_mgr, post_dossier)      |
+------------------------------------------------------------+
```

**Three existing modules are EXTENDED (additive only):**

- `dossier/slot_inference.py` — extend with the `_extract_denial(scos, notes)` helper and wire it into `infer_dossier_state_full(...)`. Slot 9 is removed from the always-DEFERRED set. **`infer_dossier_state_full` signature does NOT change** — slot 9 evidence comes from existing `scos` and `notes` parameters. M-2's PredictionRecord / DenialStrategyRecord scaffold dataclasses in `slots.py` stay BYTEWISE UNCHANGED (the M-2 contract preserved per DEC-M2-DOSSIER-004).
- `dossier/predictions.py` — extend with the `FalsificationEvidence` dataclass, the `FalsificationResult` dataclass, the `falsify_predictions(predictions, new_scos, new_notes, hunt_count)` function, the `mark_confirmed_or_falsified(...)` updater (extends or supersedes M-4's `mark_confirmed`), and a `manual_falsify(predictions, prediction_id, reason)` helper for the LLM tool.
- `dossier/scoring.py` — extend with the `emit_dossier_prediction_falsified_event(prediction, reason)` helper. M-3 byte-identical otherwise.

**Two existing modules gain NEW callers but their public surface is UNCHANGED:**

- `core/workspace.py` — slot 9 + falsification do not add new reserved actions (notes ride on the existing `AnalystNote` table; falsified-prediction state rides on the existing `_predictions_log` sentinel). `_RESERVED_ACTIONS` stays at the three M-4 entries. **The `add_note()` method (line 647) already exists** and is the canonical authority for note persistence — M-5 binds to it.
- `dossier/state.py` — `apply_predictions_overlay(state, predictions)` now sees `validated` AND `falsified` predictions in its argument list. Update its inline doc to reflect that falsified entries count as "concluded" predictions; the rule for slot 8 Predictions status stays "0 entries → EMPTY; ≥2 validated → FILLED; otherwise PARTIAL". `falsified` entries do NOT push toward FILLED (the slot rewards correct predictions, not concluded predictions); they DO keep the slot at PARTIAL rather than dropping back to EMPTY when the only entries are stale-falsified ones — by the existing rule `len(predictions) >= 1 → PARTIAL`. This is a doc-only clarification — no code change to `apply_predictions_overlay`.

**Two existing modules gain NEW wiring at the hunt site (additive only):**

- `agent/tools.py` — `run_module()` gains the steps 10–13 above; `create_tools()` gains two new tool schemas (`create_dossier_note`, `falsify_dossier_prediction`); `execute_tool()` dispatches them; two new `_execute_*` helpers (`_execute_create_dossier_note`, `_execute_falsify_dossier_prediction`); the existing `_DOSSIER_ACTIONS` filter at line 700 is widened from a 2-tuple to a 3-tuple (`"dossier_prediction_falsified"` added). The existing `create_dossier_prediction` tool gains one optional field: `falsification_evidence` (dict matching `FalsificationEvidence` shape), so analysts can author a prediction together with its disconfirmation criteria in one tool call. This is a non-breaking schema addition (the new field is optional with default null).
- `core/console.py` — `_execute_hunt` mirror gains steps 10–13. The cmd2 REPL does NOT gain a `do_note` or `do_dossier_note` command in M-5; the chat meta-command + LLM tool surface is enough for the M-5 user story. **DEC-M5-NOTE-002 (CLI-verb shape)** records this trade-off explicitly.

**Modules that are BYTEWISE UNCHANGED in M-5:**

- `dossier/slots.py` — M-1/M-2/M-4 byte-identical. `DenialStrategyRecord` scaffold preserved. `SLOT_WEIGHTS` unchanged. (DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 + DEC-M2-DOSSIER-004 preserved.)
- `dossier/panel.py` — M-1 byte-identical.
- `dossier/state.py` — M-4 byte-identical (doc-only update permitted; if the implementer keeps the doc the same, the file is byte-identical).
- `gamification/scoring.py` — M-3 byte-identical (no new `DEFAULT_RULES` row).
- `gamification/celebrations.py` — F63 byte-identical.
- `core/streak.py` — F62 byte-identical.
- `core/pivot_policy.py` + `core/event_bus.py` — F60 byte-identical.
- `core/workspace.py` — F59 byte-identical (no new public method, no new reserved action, no schema change).
- `models/database.py` — DEC-DB-002 byte-identical (no schema migration).

### 2.2 Note storage authority — existing `AnalystNote` table (DEC-M5-NOTE-001)

**Critical planner finding:** the dispatch context's instruction to use the F63 sentinel-row pattern for user notes is wrong for this codebase. Inspection shows:

- `AnalystNote` SQLAlchemy model exists in `models/database.py:218`.
- `WorkspaceManager.add_note(content, stix_object_id=None)` exists as a public method (workspace.py:647), already used by `core/console.py` (via the report flow) and exercised by tests.
- `_read_analyst_notes(workspace_mgr)` is the canonical reader; it lives in three call-site modules (`core/console.py:101`, `core/report.py:348`, `agent/tools.py:224`) and queries the `AnalystNote` table directly.
- `WorkspaceManager.get_workspace_stats()` already counts `AnalystNote.id` rows (workspace.py:749), and badge / motivation extractor logic already consumes them.

The sentinel-row pattern (F63 / DEC-M4-PERSIST-001) is the correct authority when no table exists for the data type. Notes have an existing table. Adding a parallel `_dossier_user_note` sentinel action would:

1. Split the note authority between two storage locations (Sacred Practice 12 violation).
2. Break `_read_analyst_notes()` callers, which would now miss sentinel-stored notes — silently degrading motivation extraction, M-4 `note_keyword_any` validation, the report's analyst-notes section, and badge / workspace-stat counts.
3. Force every future note reader to query BOTH the table AND the sentinel row.

**M-5 binds user-note persistence to the existing `AnalystNote` + `add_note()` authority.** Both the chat `note <text>` meta-command and the `create_dossier_note(text)` LLM tool call `workspace_mgr.add_note(text)`. No new reserved action, no second authority, no `core/workspace.py` change. The note immediately becomes visible to every existing `_read_analyst_notes` caller (motivation extraction, M-4 prediction validation `note_keyword_any` clauses, the new M-5 denial extractor, the new M-5 falsification engine).

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **existing `AnalystNote` table + `add_note()` method** (recommended) | `workspace_mgr.add_note(text)`; `_read_analyst_notes()` reads as today. | **accepted** | Single authority preserved. Zero schema change. Zero workspace.py change. Notes immediately visible to motivation extractor, M-4 prediction `note_keyword_any` matching, M-5 denial extractor, M-5 falsification engine. Mirrors how `cmd2 hint` / `report` flows already use the table. |
| (b) NEW `_dossier_user_note` sentinel action (dispatch context's suggestion) | F63-pattern sentinel rows in `score_events`; per-note row keyed by indicator+points=0. | **rejected** | Parallel authority violates Sacred Practice 12; silently breaks `_read_analyst_notes` callers; requires updating motivation extractor, M-4 validation, report.py, workspace stats — net larger blast radius than reusing the existing table. The sentinel-row pattern is the correct authority when no table exists; the F63 example was a single-row metadata sentinel, not a per-record store. |
| (c) NEW `dossier_user_note` table via `models/database.py` | Typed SQLAlchemy model + new workspace helper. | **rejected** | Violates DEC-DB-002 "no migrations" v1 discipline. Also redundant with (a) — `AnalystNote` already exists for exactly this purpose. |

**Honest acknowledgement:** the dispatch context's intent was "no schema change," which DEC-M5-NOTE-001 honors trivially because the existing `AnalystNote` table already covers the surface. The dispatch context's specific suggestion ("piggyback on the F63 sentinel-row pattern") is overridden here by the planner's discovery that the table already exists. No second F63 application is needed.

### 2.3 Denial / Deception slot 9 vocabulary (DEC-M5-DENIAL-001)

M-5 ships a deliberately small slot-9 extractor vocabulary. The extractor returns a `SlotState` for `DossierSlotName.DENIAL` analogous to M-2's existing extractors. Three evidence categories drive the slot:

| category | source | match rule | example |
|----------|--------|------------|---------|
| **DGA-shaped domain** | SCO type `domain-name` | label length ≥ 12 AND consonant-to-vowel ratio ≥ 3 (no recognizable word fragments) | `xqzpfwbkdmrl.example.org` |
| **Fast-flux / decoy infrastructure hint** | SCO type `ipv4-addr` or `ipv6-addr` carrying x_ap extension `x_ap_dns_ttl` with value ≤ 60 seconds | TTL ≤ 60 sec from any AP module | `1.2.3.4` with `x_ap_dns_ttl: 30` |
| **Denial / evasion keyword in analyst note** | `AnalystNote.content` (case-insensitive) | substring match against `_DENIAL_KEYWORDS` frozenset | `"actor uses sandbox-aware loader, evasion via decoy domains"` |

`_DENIAL_KEYWORDS` (M-5 v1.0; deliberately small, mirrors `_MOTIVATION_CATEGORIES` shape):

```python
_DENIAL_KEYWORDS: frozenset[str] = frozenset({
    "decoy", "deception", "evasion", "evasive",
    "sandbox", "sandbox-aware", "sandbox-evasion",
    "obfuscation", "obfuscated",
    "anti-analysis", "anti-vm", "anti-sandbox",
    "dga", "fast-flux", "fast flux", "flux",
    "domain generation", "domain-generation",
    "honeypot",  # actor's awareness of defender deception
})
```

**Status thresholds (DEC-M5-DENIAL-002):**
- **EMPTY**: 0 evidence items from any category.
- **PARTIAL**: ≥ 1 evidence item from any single category.
- **FILLED**: ≥ 1 evidence item from at least 2 distinct categories (cross-category corroboration — same threshold shape as M-1's "distinct types ≥ 2 → FILLED" rule for DEC-M1-DOSSIER-INFERENCE-STATUS-001).

**DGA shape detector (DEC-M5-DENIAL-003):** the implementer ships a tiny pure-function helper `_is_dga_shaped(label: str) -> bool` with the consonant-to-vowel rule above. This is a deliberately conservative MVP detector — it intentionally misses dictionary-word DGAs and intentionally catches some legitimate-but-cryptic domains (e.g., long base32-encoded subdomain labels). The unit tests document both classes of error. Richer DGA detection (n-gram entropy, frequency analysis) lands in a future slice if the v1.0 detector proves too noisy on real workspaces.

**Fast-flux TTL detector:** reads `sco.get("x_ap_dns_ttl")` if present (not currently set by any AP module, but the field name reserves the surface for M-7 / M-8 modules that may surface DNS TTL data). If no SCO carries the field, the fast-flux category contributes zero evidence — slot 9 still functions via the DGA + keyword categories. The detector is forward-compatible without requiring any module change in M-5.

### 2.4 FalsificationEvidence vocabulary v1.0 (DEC-M5-FALSIFY-002)

M-5 ships a typed `FalsificationEvidence` dataclass that mirrors M-4's `ExpectedEvidence` but expresses the contrapositive — "this evidence contradicts the prediction." All non-None fields are ANDed. At least one field must be non-None (loud `ValueError` otherwise).

```python
@dataclass
class FalsificationEvidence:
    """Typed contradiction pattern for prediction falsification (DEC-M5-FALSIFY-002).

    All non-None fields are ANDed together; empty FalsificationEvidence is rejected.
    Mirrors the ExpectedEvidence vocabulary but reads in the negative sense.
    """
    negative_sco_type: str | None = None
    """If set, the appearance of a SCO of this type counts as contradicting evidence
    (e.g., 'autonomous-system' with negative_asn_in below: actor used a different ASN
    than predicted)."""

    negative_value_regex: str | None = None
    """If set, an SCO's primary value matching this regex falsifies the prediction
    (e.g., '.*\\.cn$' falsifies a 'pivot to .ru' prediction)."""

    negative_asn_in: list[int] | None = None
    """For ipv4-addr / ipv6-addr / autonomous-system: appearance of an ASN in this list
    falsifies the prediction."""

    contradiction_keyword_any: list[str] | None = None
    """At least one of these substrings appearing in an analyst note falsifies the
    prediction (e.g., note 'actor disappeared, no .ru pivot observed' falsifies a
    'pivot to .ru' prediction)."""

    stale_after_n_hunts: int | None = None
    """If set, a still-pending prediction is auto-falsified once the workspace has
    completed N or more hunts since the prediction was created. None = no temporal
    rule. Counted against module_runs row count at falsify-time (DEC-M5-FALSIFY-003).

    Sentinel exception: this field MAY be the only non-None field of FalsificationEvidence
    (a pure temporal-window rule with no contradiction-evidence criteria is valid)."""
```

**Cross-hunt scope (DEC-M5-FALSIFY-004):** M-4's validation uses current-hunt evidence only (DEC-M4-PRED-003). M-5 falsification keeps the same current-hunt scope for the contradiction-evidence categories (`negative_*` and `contradiction_keyword_any`). The `stale_after_n_hunts` temporal rule is the ONLY cross-hunt signal — it does not scan historical SCOs / notes; it only counts `workspace_mgr.get_module_runs()` row count and falsifies when (current_hunt_count - prediction_creation_hunt_count) >= stale_after_n_hunts. The prediction's creation-time hunt count is captured at `create_prediction` time in a NEW `created_at_hunt_count: int` field on `PersistedPrediction` (additive non-breaking — new field defaults to 0 for legacy entries; the falsifier treats 0 as "creation hunt count unknown — skip stale check"). This keeps M-5 strictly current-hunt for evidence matching while supporting the user-facing "auto-falsify this if not confirmed in N hunts" feature without a workspace-wide rescan.

### 2.5 Falsification event + score (DEC-M5-FALSIFY-005)

A new `emit_dossier_prediction_falsified_event(prediction: PersistedPrediction, reason: str) -> dict` helper in `dossier/scoring.py`. Event shape:

```python
{
    "action": "dossier_prediction_falsified",
    "points": 0,                            # DEC-M4-PRED-006 binding: no negative-points events
    "indicator": prediction.prediction_id,  # e.g. "pred-3f19d55c"
    "rule_description": f"Dossier prediction falsified: {reason}",  # plain ASCII; F64-clean
}
```

`points=0` is the canon from DEC-M4-PRED-006. M-5 explicitly does not relitigate the negative-points question — the M-7 narrative-feedback path remains the documented surface for "reckless guessing should cost score" content. The event still flows through `store_score_events` so:

- F62 streak chain sees it as a zero-points event (does NOT reset the streak; treats it as a no-op for streak math identical to how M-4's prediction-validated and slot-fill events flow).
- F63 milestone catch-up sees the same `post_total` it would see without the event.
- F64 panel separation: `_DOSSIER_ACTIONS` filter widens to include `"dossier_prediction_falsified"` so the event text is stripped from the LLM-facing `summary` (the panel renders it directly).

### 2.6 Manual override LLM tool — `falsify_dossier_prediction` (DEC-M5-FALSIFY-006)

NEW LLM tool registered in `agent/tools.py::create_tools()` schema list + dispatched in `execute_tool()`. Implementation in a new `_execute_falsify_dossier_prediction(ctx, prediction_id, reason)` helper. F64-clean: returns structured JSON text only.

```json
{
  "type": "function",
  "function": {
    "name": "falsify_dossier_prediction",
    "description": "Mark a pending dossier prediction as falsified with a reason. Use this when you have analyst judgment that the prediction was wrong (e.g., actor pivoted elsewhere than predicted) but no machine-matchable contradiction evidence is available. The prediction transitions pending->falsified; a dossier_prediction_falsified score event fires at +0 points (no deduction). Idempotent: already-falsified or already-validated predictions are no-ops.",
    "parameters": {
      "type": "object",
      "properties": {
        "prediction_id": {
          "type": "string",
          "description": "The PersistedPrediction.prediction_id to falsify, e.g. 'pred-3f19d55c'."
        },
        "reason": {
          "type": "string",
          "description": "Plain-text explanation of why the prediction is wrong. Stored in the score event rule_description and persisted to the predictions log."
        }
      },
      "required": ["prediction_id", "reason"]
    }
  }
}
```

The handler:
1. Loads the predictions log.
2. Finds the entry by `prediction_id` (returns JSON error if missing).
3. If `status != "pending"`, returns JSON `{"prediction_id": ..., "status": "<current>", "message": "already concluded; no-op"}` (idempotent).
4. Otherwise: updates `status="falsified"`, fills `validated_at` (reused as the conclusion timestamp), persists the updated log via `save_predictions_log`.
5. Emits `emit_dossier_prediction_falsified_event(prediction, reason)` and stores it via `store_score_events`.
6. Returns JSON `{"prediction_id": ..., "status": "falsified", "reason": "...", "message": "..."}`.

The handler is the only path that emits a falsified event outside the hunt-site auto-falsification loop. The hunt-site loop runs every hunt; the manual override fires whenever the LLM (or via direct tool call) decides a prediction is wrong.

### 2.7 LLM-tool surface: `create_dossier_note` (DEC-M5-NOTE-003)

NEW LLM tool registered in `agent/tools.py::create_tools()`. Implementation in a new `_execute_create_dossier_note(ctx, text)` helper. F64-clean: returns structured JSON text only.

```json
{
  "type": "function",
  "function": {
    "name": "create_dossier_note",
    "description": "Author an analyst note about the threat actor. Notes are stored in the workspace and become evidence for dossier slot inference (especially Motivation and Denial slots) and prediction validation/falsification (note_keyword_any and contradiction_keyword_any clauses). Use this to record observations the user shares in chat that should be part of the dossier (motivations, suspected tactics, OPSEC observations).",
    "parameters": {
      "type": "object",
      "properties": {
        "text": {
          "type": "string",
          "description": "Free-text note content. Will be stored verbatim in the AnalystNote table and visible to the dossier inference engine."
        }
      },
      "required": ["text"]
    }
  }
}
```

The handler:
1. Validates `text` is non-empty (loud `ValueError` otherwise).
2. Calls `ctx.workspace_mgr.add_note(text)` — the existing public method.
3. Returns JSON `{"status": "saved", "message": f"Note saved to dossier evidence ({len(text)} chars)."}`.

No `stix_object_id` argument in v1.0 — the existing `add_note()` signature supports it (`add_note(content, stix_object_id=None)`), but the LLM tool surface stays minimal in M-5. Future slices may extend the tool to link notes to specific SCOs.

### 2.8 Chat meta-command: `note <text>` (DEC-M5-NOTE-002)

The chat meta-command interceptor in `agent/chat.py` (already handles `dossier`, `export`, `help`, etc.) gains one new branch:

```python
# Note meta-command — DEC-M5-NOTE-002: local handler, no LLM dispatch.
# Persists the text via the existing workspace_mgr.add_note() API. Visible
# immediately to motivation extractor, dossier denial extractor, and the
# M-5 falsification engine via the existing _read_analyst_notes reader.
if lower.startswith("note ") or lower == "note":
    note_text = stripped[len("note"):].strip()
    if not note_text:
        console.print("[dim]Usage: note <text>[/dim]")
        continue
    try:
        runner.ctx.workspace_mgr.add_note(note_text)
        console.print(f"[green]Note saved.[/green] ({len(note_text)} chars)")
    except Exception as e:
        handle_error(e, console, runner, config_mgr)
    continue
```

The help table gains one row:

```python
help_table.add_row(
    "note",
    "note <text>",
    "Save an analyst note (visible to dossier denial extractor + prediction validation/falsification)",
)
```

**No cmd2 `do_note` is added to `core/console.py`** — DEC-M5-NOTE-002. The cmd2 REPL is a power-user surface (ADR-010); adding a `do_note` command is a backlog item that does not block M-5's "first-class user-note authoring surface" goal. The chat meta-command + LLM tool is the user-facing surface for v1; cmd2 parity is a small future slice if requested.

### 2.9 PersistedPrediction schema extension (DEC-M5-FALSIFY-007)

M-4's `PersistedPrediction` dataclass gains one optional new field at the end:

```python
@dataclass
class PersistedPrediction:
    # ... M-4 fields unchanged ...
    falsification_evidence: FalsificationEvidence | None = None
    """If set, M-5 falsification engine uses this to detect contradiction evidence.
    Default None means the prediction is never auto-falsified (only the stale_after_n_hunts
    field's presence enables temporal-window auto-falsify). M-5 NEW (DEC-M5-FALSIFY-007)."""

    created_at_hunt_count: int = 0
    """Module-run count at prediction creation time. Used by the stale_after_n_hunts
    temporal-window rule. M-5 NEW. Defaults to 0 for legacy M-4 entries so the
    falsifier treats them as 'unknown creation hunt count — skip stale check'
    (DEC-M5-FALSIFY-004)."""
```

**JSON serialization (DEC-M5-FALSIFY-008):** the JSON envelope schema is bumped from `"schema_version": 1` to `"schema_version": 2`. The deserializer accepts both 1 and 2; v1 envelopes deserialize with `falsification_evidence=None` and `created_at_hunt_count=0`. The serializer always emits v2. A new v2-loud-failure path catches v3+ envelopes the way M-4's v1 path catches v0 / v2. This is the M-4 "schema_version=1 round-trip + schema_version=2 raises" pattern, simply bumped one version. M-4's `_dossier_state_snapshot` envelope stays at v1 — only the `_predictions_log` envelope advances.

The `create_prediction(slot, text, expected_evidence_dict, falsification_evidence_dict=None)` factory function in `dossier/predictions.py` gains the optional `falsification_evidence_dict` parameter. When supplied:
- The dict must satisfy the `FalsificationEvidence` shape.
- If all fields are None, the factory raises `ValueError("falsification_evidence must have at least one non-None field if supplied")`.
- The `PersistedPrediction.created_at_hunt_count` is computed from `workspace_mgr.get_module_runs()` row count at creation time — this requires the factory to receive a `WorkspaceManager`, which it currently does not. **Two implementation options:** (a) move the `created_at_hunt_count` capture out of `create_prediction` and into `_execute_create_dossier_prediction` (the only legitimate caller from production code); (b) extend `create_prediction` to accept an optional `hunt_count` arg defaulting to 0. The plan picks **(a)** — keeps `create_prediction` pure-function and lets the workspace coupling stay in the agent-tools call site. The implementer notes this choice in the source @decision annotation.

The `create_dossier_prediction` LLM tool's JSON schema gains the optional `falsification_evidence` property mirroring the `expected_evidence` shape:

```json
"falsification_evidence": {
  "type": "object",
  "description": "Optional typed contradiction criteria. When supplied, M-5 auto-falsifies the prediction if current-hunt evidence matches. All non-null fields are ANDed.",
  "properties": {
    "negative_sco_type": {"type": "string"},
    "negative_value_regex": {"type": "string"},
    "negative_asn_in": {"type": "array", "items": {"type": "integer"}},
    "contradiction_keyword_any": {"type": "array", "items": {"type": "string"}},
    "stale_after_n_hunts": {"type": "integer", "minimum": 1}
  }
}
```

This is added to the existing tool's `properties` block and is NOT added to `required` — it stays optional.

---

## 3. Removal targets (no parallel-authority residue)

- **The DEFERRED-only path for slot 9 in `slot_inference.py`** is replaced by the M-5 extractor. After the change, slot 9 is in the same "active extractor" class as Identity / TTPs / Infrastructure / Timing / Capability / Motivation. The DEFERRED set shrinks from `{TARGETING, PREDICTIONS, DENIAL}` to `{TARGETING}` (Predictions is overlaid by M-4; Denial is now real). The implementer removes `DossierSlotName.DENIAL` from the `deferred_names` list at slot_inference.py:335. No new module is created.
- **M-4's `mark_confirmed` updater** is superseded by M-5's `mark_confirmed_or_falsified` — `mark_confirmed` is removed (or kept as a thin wrapper that calls the new updater with an empty falsification list) to prevent two parallel "update prediction lifecycle" paths. The plan picks **removal with a thin wrapper preserved** for backward compatibility with any tests that still import `mark_confirmed`; the wrapper is a 3-line passthrough that calls `mark_confirmed_or_falsified(predictions, results, [])`. The wrapper is annotated `# DEPRECATED M-5: use mark_confirmed_or_falsified` and the implementer files a backlog issue to remove the wrapper in a future cleanup slice.

---

## 4. Implementation note: Predictions-slot overlay sees falsified entries

`apply_predictions_overlay` (M-4, `dossier/state.py`) currently computes Predictions-slot status from the predictions list:

- 0 entries → EMPTY
- ≥ 2 validated → FILLED
- otherwise → PARTIAL

After M-5, the predictions list will contain `falsified` entries in addition to `pending` and `validated`. The existing rules still apply correctly: `falsified` entries do NOT count toward `validated_count` (so they do not push the slot toward FILLED), but they DO satisfy `len(predictions) >= 1` (so the slot stays PARTIAL rather than EMPTY when the only entries are stale-falsified ones). This is the intended behavior — the Predictions slot tracks how much analytic engagement with predictions has happened, weighted by correctness. No code change to `apply_predictions_overlay`; the implementer adds an explicit test case to `test_dossier_state.py` that asserts `[falsified, falsified] → PARTIAL` (1 distinct slot-status test case).

---

## 5. The load-bearing acceptance test

The compound integration test that proves M-5 ships is in three stages, exercised end-to-end:

**Stage A — slot 9 inference:**
1. Fresh workspace, no SCOs, no notes. Render `dossier` panel — slot 9 status is EMPTY (no longer DEFERRED).
2. Add an analyst note via `note actor uses sandbox-aware loader, evasion via decoy domains`. Render `dossier` panel — slot 9 status is PARTIAL (1 keyword category hit).
3. Run a module that stores a DGA-shaped domain (e.g., `xqzpfwbkdmrl.example.org`). Render `dossier` panel — slot 9 status is FILLED (2 distinct categories: DGA shape + note keyword). One `dossier_slot_filled` event fires at +2 points (slot weight 2.5 floored).

**Stage B — auto-falsification:**
1. Author a prediction via `create_dossier_prediction` with `expected_evidence={"value_regex": ".*\\.ru$"}` AND `falsification_evidence={"contradiction_keyword_any": [".cn", "china"]}`. Prediction `pred-XXXXXXXX` persisted with `status=pending`.
2. Add an analyst note: `note actor pivoted to .cn infrastructure not .ru as expected`. Run any hunt (even a no-op). The falsification engine matches the contradiction keyword. Prediction transitions `pending → falsified`. One `dossier_prediction_falsified` event fires at +0 points (DEC-M4-PRED-006). Predictions log persists the updated status.

**Stage C — manual override + `ap chat` restart:**
1. Author a second prediction (pure stale rule): `create_dossier_prediction(text="...", expected_evidence={"sco_type": "x509-certificate"}, falsification_evidence={"stale_after_n_hunts": 1})`. Note `created_at_hunt_count` is captured at creation.
2. Run one hunt that does not produce an x509-certificate SCO. Verify the prediction is NOT auto-falsified yet (1 hunt only, threshold is 1 → falsifies on hunt 2 or later, depending on counter semantics — the implementer documents the exact ≥/> semantics in the test).
3. Run a second hunt — the stale rule fires. Prediction auto-falsifies.
4. Author a third prediction with no `falsification_evidence`. Call `falsify_dossier_prediction(prediction_id="pred-XXXXXXXX", reason="actor abandoned campaign")`. Prediction transitions `pending → falsified` via manual override.
5. Quit `ap chat`. Restart `ap chat` in the same workspace.
6. Run `dossier` panel — slot 8 Predictions status reflects the mix (0 validated → not FILLED; ≥ 1 entry → PARTIAL).
7. `get_total_score()` includes the slot 9 fill events from Stage A and zero from the falsification events. F62 streak chain is unbroken. F63 milestone catch-up sees the unchanged total.

This is the "M-5 ships" acceptance test; it is mandatory in the Evaluation Contract (§7).

---

## 6. Invariant preservation matrix

| invariant | scope | M-5 check |
|-----------|-------|-----------|
| F59 (workspace single authority for SCO persistence) | `core/workspace.py` | BYTEWISE UNCHANGED. M-5 reuses the existing `AnalystNote` table + `add_note()` public method + `_RESERVED_ACTIONS` (M-4 three entries unchanged). No schema change, no new public method, no new reserved action. Test gate: `test_workspace.py` regression asserts `_RESERVED_ACTIONS` is still the M-4 three-entry frozenset. |
| F60 (auto-pivot policy + event bus invariants) | `core/pivot_policy.py`, `core/event_bus.py` | BYTEWISE UNCHANGED. No new event-bus subscriber. M-5 falsification is a pure function called inline from hunt sites, mirroring M-4's validation. |
| F62 (StreakManager single authority; `streak_continued` semantics) | `core/streak.py`, F62 tests | BYTEWISE UNCHANGED. Falsification events at +0 points flow through `store_score_events` as ordinary score events; F62 streak logic continues to see them as zero-points events that neither reset nor extend the streak. Test gate: F62 regression test asserts a falsified event in a hunt does NOT break an active streak. |
| F63 (milestone catch-up + sentinel-row pattern) | `gamification/celebrations.py` | UNCHANGED. M-5 events at +0 points do not change `post_total`; milestone catch-up math sees identical totals. Test gate: integration test asserts a hunt that fires a falsification event does NOT cross a milestone purely due to the event. |
| F64 (de-duplicate LLM narration vs Rich panel) | `agent/tools.py::_DOSSIER_ACTIONS` filter | M-5 widens the filter from 2-tuple to 3-tuple: `{"dossier_slot_filled", "dossier_prediction_validated", "dossier_prediction_falsified"}`. Test gate: integration test asserts falsified-event text absent from LLM tool `result["summary"]`. |
| Sacred Practice 12 (one authority per operational fact) | new + existing | Slot 9 inference owned by `dossier/slot_inference.py`. FalsificationEvidence + falsification engine owned by `dossier/predictions.py`. Falsification event shape owned by `dossier/scoring.py`. User-note persistence owned by `WorkspaceManager.add_note()` + `AnalystNote` table (existing — single authority preserved per DEC-M5-NOTE-001). No fact has two owners. |
| DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 | `dossier/slots.py` | BYTEWISE UNCHANGED. Slot 9 (Denial) weight stays 2.5; the M-5 extractor reads it via `SLOT_WEIGHTS[DossierSlotName.DENIAL]`. |
| DEC-M2-DOSSIER-004 (PredictionRecord + DenialStrategyRecord scaffolds) | `dossier/slots.py` | BYTEWISE UNCHANGED. M-5 uses the richer `PersistedPrediction` shape in `dossier/predictions.py` (M-4 contract); `DenialStrategyRecord` stays as the M-2 scaffold — M-5 does NOT add a per-strategy persistence layer (it derives slot 9 status from SCO + note evidence directly, not from a per-strategy table). The scaffold dataclass remains available as an import surface for a future slice that adds the per-strategy persistence layer. |
| DEC-M3-DOSSIER-001..005 (M-3 scoring authority) | `dossier/scoring.py` | EXTENDED (additive only). `emit_dossier_slot_filled_events` UNCHANGED. `emit_dossier_prediction_validated_event` UNCHANGED. NEW `emit_dossier_prediction_falsified_event(prediction, reason)` helper at module bottom; same shape as the M-3 validated helper. |
| DEC-M4-PERSIST-001..003 (M-4 persistence + JSON envelope) | `dossier/state.py`, `dossier/predictions.py` | EXTENDED (additive only). `_dossier_state_snapshot` envelope stays at `schema_version=1`. `_predictions_log` envelope advances to `schema_version=2`; v1 envelopes still deserialize (legacy-compatible); v2 envelopes always serialize. Loud failure on v3+ (DEC-M4-PERSIST-003 pattern preserved, just bumped one version). |
| DEC-M4-PRED-002 (ExpectedEvidence v1.0 vocabulary) | `dossier/predictions.py` | FROZEN. `ExpectedEvidence` dataclass byte-identical. M-5 adds the parallel `FalsificationEvidence` dataclass; the two have separate vocabularies. |
| DEC-M4-PRED-003 (validation scope = current-hunt) | `dossier/predictions.py` | PRESERVED. M-4's `validate_predictions` byte-identical. M-5's `falsify_predictions` keeps current-hunt scope for evidence matching; the `stale_after_n_hunts` rule is a per-prediction temporal counter, not a workspace-wide rescan. |
| DEC-M4-PRED-005 (active falsification is M-5's responsibility) | M-5 plan + impl | SATISFIED. This is the slice that fulfills the deferred responsibility. The M-4 deferral text in `predictions.py:32-36` should be updated to note "Active falsification implemented in M-5 (DEC-M5-FALSIFY-001..008)" — the implementer may either replace the comment or leave it as an honest historical record of the deferral. |
| DEC-M4-PRED-006 (no negative-points events) | `dossier/scoring.py` | PRESERVED. Falsification event fires at `points=0`. M-5 explicitly does not relitigate this; the planner records it in DEC-M5-FALSIFY-005 as inherited canon. |

---

## 7. Evaluation Contract (9-key, ~35–45 tests)

**required_tests:**

The implementer ships ~35–45 tests across these files. Counts are minimums.

- `tests/test_dossier_slot_inference.py` **(EXTEND, ~8 tests)**:
  - slot 9 returns EMPTY on a workspace with no DGA domains and no denial-keyword notes (1)
  - slot 9 returns PARTIAL when only DGA-shape evidence present (1)
  - slot 9 returns PARTIAL when only note-keyword evidence present (1)
  - slot 9 returns PARTIAL when only fast-flux TTL evidence present (forward-compatible test — uses a constructed SCO with `x_ap_dns_ttl=30`) (1)
  - slot 9 returns FILLED when DGA-shape + note-keyword (2 distinct categories) (1)
  - slot 9 contributing_types reflects the actual categories hit (e.g. `frozenset({"dga", "note_keyword"})`) (1)
  - `_is_dga_shaped` returns True for `xqzpfwbkdmrl` and False for `mail.google.com` / `paypal.com` / `bobby-hill.kingofthehill.org` (1)
  - `deferred_names` in `infer_dossier_state_full` is now `[TARGETING]` only — regression that confirms slot 9 left the deferred set (1)

- `tests/test_dossier_predictions.py` **(EXTEND, ~12 tests)**:
  - FalsificationEvidence empty (all fields None) creation via factory raises ValueError (1)
  - FalsificationEvidence with only `stale_after_n_hunts` set is accepted (sentinel exception per DEC-M5-FALSIFY-002) (1)
  - falsify_predictions: empty list → empty results (1)
  - falsify_predictions: `negative_value_regex` match → falsified (1)
  - falsify_predictions: `negative_sco_type` match → falsified (1)
  - falsify_predictions: `negative_asn_in` match → falsified (1)
  - falsify_predictions: `contradiction_keyword_any` match → falsified (1)
  - falsify_predictions: `stale_after_n_hunts` rule fires at exactly the threshold hunt count (boundary case documented) (1)
  - falsify_predictions: `stale_after_n_hunts` rule does NOT fire below threshold (1)
  - falsify_predictions: already-validated prediction skipped (idempotency) (1)
  - falsify_predictions: already-falsified prediction skipped (idempotency) (1)
  - `mark_confirmed_or_falsified`: mixed list (1 confirmed + 1 falsified + 1 still-pending) updates correctly (1)

- `tests/test_dossier_predictions_serialization.py` **(NEW or extension to existing, ~4 tests)**:
  - PersistedPrediction v1 envelope deserializes with `falsification_evidence=None` and `created_at_hunt_count=0` (1)
  - PersistedPrediction v2 envelope round-trips with falsification_evidence + created_at_hunt_count populated (1)
  - schema_version=3 raises loud RuntimeError (mirrors M-4 schema_version=2-raises pattern, bumped one version) (1)
  - Serializer always emits v2 (regression — implementer must NOT silently write v1 when falsification_evidence is None) (1)

- `tests/test_dossier_scoring.py` **(EXTEND, ~3 tests)**:
  - `emit_dossier_prediction_falsified_event` returns dict with action="dossier_prediction_falsified", points=0, indicator=prediction_id, rule_description plain ASCII (1)
  - falsified event fires alongside slot-fill events in the same hunt (1)
  - slot 9 Denial transition (empty → partial → filled) emits at +2 points per transition via the M-3 `emit_dossier_slot_filled_events` path (no scoring.py change needed; this regression confirms the unchanged M-3 emitter handles the unblocked slot correctly) (1)

- `tests/test_dossier_state.py` **(EXTEND, ~3 tests)**:
  - `apply_predictions_overlay` with `[falsified]` only → PARTIAL (NOT EMPTY; ≥1 entry threshold) (1)
  - `apply_predictions_overlay` with `[falsified, falsified]` → PARTIAL (no validated entries; not FILLED) (1)
  - `apply_predictions_overlay` with `[validated, validated, falsified]` → FILLED (validated count ≥ 2; falsified does not block) (1)

- `tests/test_agent_tools.py` **(EXTEND, ~6 tests)**:
  - `create_dossier_note` tool schema present in `create_tools()` output (1)
  - `create_dossier_note` execution path calls `workspace_mgr.add_note(text)` and returns success JSON; the note is visible via `_read_analyst_notes` immediately afterward (1)
  - `falsify_dossier_prediction` tool schema present (1)
  - `falsify_dossier_prediction` execution: pending prediction → falsified + event fires + persistence updated (1)
  - `falsify_dossier_prediction` execution: already-falsified prediction → idempotent no-op (JSON message documents the no-op) (1)
  - `create_dossier_prediction` schema now includes the optional `falsification_evidence` property (1)

- `tests/test_agent_tools.py` **(EXTEND, hunt-site, ~3 tests)**:
  - hunt with a persisted prediction carrying `falsification_evidence` and matching contradiction evidence in the current hunt → `dossier_prediction_falsified` event fires + status transitions to "falsified" + persists (1)
  - F64 gate: `dossier_prediction_falsified` event text absent from `result["summary"]` (1)
  - hunt with a persisted prediction carrying `stale_after_n_hunts=1` and hunt_count past threshold → auto-falsifies (1)

- `tests/test_chat_meta.py` (or `test_chat_dossier_metacommand.py` extension) **(EXTEND, ~3 tests)**:
  - `note <text>` meta-command calls `workspace_mgr.add_note(text)` (1)
  - `note` alone prints usage hint without crashing (1)
  - `note <text>` then `dossier` shows slot 9 (or motivation) status reflecting the new note (compound) (1)

- `tests/test_dossier_persistence_integration.py` **(EXTEND, ~3 tests)**:
  - The §5 Stage B compound: prediction with `falsification_evidence` + contradiction note → auto-falsifies; persists; survives reload (1)
  - The §5 Stage C compound: stale-rule auto-falsify after N hunts; manual override; persistence across `ap chat` restart (1)
  - F62 regression: a falsification event in a hunt does NOT break the active streak chain (1)

- `tests/test_dossier_get_state_tool.py` **(EXTEND, ~1 test)**:
  - `get_dossier_state` returned JSON now shows slot 9 with real status (empty/partial/filled), no longer always "deferred" (1)

**Total: ~45 new + extended tests.** Full suite green: ≥ 2178 passed (M-4 baseline) + ~45 new M-5 tests, minus any duplicate skips. Implementer must report the actual pre/post test counts in the readiness summary.

**required_evidence:**
- Full pytest output green for the worktree.
- `git diff main -- src/adversary_pursuit/core/workspace.py` is empty (no workspace changes — F59 invariant preserved BYTEWISE).
- `git diff main -- src/adversary_pursuit/models/database.py` is empty (DEC-DB-002 + DEC-M5-NOTE-001 preserved).
- `git diff main -- src/adversary_pursuit/dossier/slots.py` is empty (DEC-M2-DOSSIER-004 preserved).
- `git diff main -- src/adversary_pursuit/dossier/panel.py` is empty.
- `git diff main -- src/adversary_pursuit/dossier/state.py` is empty OR limited to doc-only comment updates (no API change). If non-empty, the implementer pastes the diff for reviewer confirmation that it is doc-only.
- `git diff main -- src/adversary_pursuit/gamification/scoring.py` is empty.
- `git diff main -- src/adversary_pursuit/gamification/celebrations.py` is empty.
- `git diff main -- src/adversary_pursuit/core/streak.py` is empty.
- `git diff main -- src/adversary_pursuit/core/pivot_policy.py` is empty.
- `git diff main -- src/adversary_pursuit/core/event_bus.py` is empty.
- Demo trace (or test transcript) showing the §5 three-stage acceptance scenario: slot 9 transitions from EMPTY → FILLED via DGA + note; auto-falsification via contradiction keyword; manual override; stale-rule auto-falsify; persistence across `ap chat` restart.

**required_authority_invariants:**
- F59: `core/workspace.py` BYTEWISE UNCHANGED. `_RESERVED_ACTIONS` stays at M-4 three entries. No schema migration, no new public method, no new SQLAlchemy model. M-5 reuses the existing `AnalystNote` table + `add_note()` API per DEC-M5-NOTE-001.
- F60: `core/pivot_policy.py` + `core/event_bus.py` BYTEWISE UNCHANGED; no new bus subscriber.
- F62: `core/streak.py` BYTEWISE UNCHANGED; falsification events at +0 points neither reset nor extend the streak; F62 regression test included.
- F63: `gamification/celebrations.py` UNCHANGED; +0-point falsification events do not affect milestone catch-up; integration test asserts the math.
- F64: `_DOSSIER_ACTIONS` filter widens to 3-tuple `{"dossier_slot_filled", "dossier_prediction_validated", "dossier_prediction_falsified"}` in `agent/tools.py`; integration test asserts falsified event text absent from LLM summary.
- Sacred Practice 12: per the §6 invariant matrix. Critical: M-5 reuses `AnalystNote` rather than creating a parallel `_dossier_user_note` sentinel authority (DEC-M5-NOTE-001).
- DEC-M1-SLOTS-WEIGHT-AUTHORITY-001: `SLOT_WEIGHTS` UNCHANGED (slot 9 stays at 2.5).
- DEC-M2-DOSSIER-004: `dossier/slots.py` BYTEWISE UNCHANGED. M-5 does NOT add per-strategy persistence; `DenialStrategyRecord` scaffold preserved as the import contract for a future per-strategy slice if needed.
- DEC-M3-DOSSIER-001..005: `dossier/scoring.py` extended additively (new `emit_dossier_prediction_falsified_event` helper); the M-3 emitters are byte-identical.
- DEC-M4-PERSIST-001: storage authority preserved — `_predictions_log` sentinel row still the predictions-persistence authority; falsified-prediction state rides on the existing row (no new reserved action).
- DEC-M4-PERSIST-003: schema versioning preserved — `_predictions_log` envelope advances v1 → v2 (loud-failure handshake preserved; v3+ raises RuntimeError).
- DEC-M4-PRED-002: `ExpectedEvidence` v1.0 vocabulary FROZEN.
- DEC-M4-PRED-003: validation scope stays current-hunt-only; falsification keeps the same scope (the stale rule is a per-prediction counter, not a workspace rescan).
- DEC-M4-PRED-006: confirmation = +4, falsification = +0 (no negative-points events) — explicitly inherited as canon by DEC-M5-FALSIFY-005.

**required_integration_points:**
- `dossier/slot_inference.py` (EXTEND: `_extract_denial(scos, notes)` helper + `_is_dga_shaped(label)` helper + wire slot 9 into `infer_dossier_state_full`; remove `DENIAL` from `deferred_names`).
- `dossier/predictions.py` (EXTEND: `FalsificationEvidence` dataclass + `FalsificationResult` dataclass + `falsify_predictions(predictions, new_scos, new_notes, hunt_count)` + `mark_confirmed_or_falsified(...)` + thin-wrapper `mark_confirmed` for back-compat + extend `PersistedPrediction` with `falsification_evidence` + `created_at_hunt_count` + extend `create_prediction` to accept optional `falsification_evidence_dict` + bump `_SCHEMA_VERSION` to 2 + extend `_deserialize_predictions` to accept both v1 and v2 envelopes + extend `_serialize_predictions` to always emit v2).
- `dossier/scoring.py` (EXTEND: NEW `emit_dossier_prediction_falsified_event(prediction, reason)` helper at module bottom; M-3 emitters byte-identical).
- `dossier/__init__.py` (export new symbols: `FalsificationEvidence`, `FalsificationResult`, `falsify_predictions`, `mark_confirmed_or_falsified`, `emit_dossier_prediction_falsified_event`).
- `agent/tools.py` (extend `run_module` hunt-site wiring per §2.1 steps 10–13; register `create_dossier_note` + `falsify_dossier_prediction` LLM tools; extend `create_dossier_prediction` tool schema with optional `falsification_evidence`; widen `_DOSSIER_ACTIONS` filter to 3-tuple; new `_execute_create_dossier_note` + `_execute_falsify_dossier_prediction` helpers; `_execute_create_dossier_prediction` captures `created_at_hunt_count` from `ctx.workspace_mgr.get_module_runs()`).
- `core/console.py` (extend `_execute_hunt` hunt-site wiring mirror for falsification engine — same shape as `agent/tools.py` changes; widen the analogous `_DOSSIER_ACTIONS`-like filter if one exists, otherwise no change to console.py's filter site).
- `agent/chat.py` (add `note <text>` meta-command branch + help-table row).

**forbidden_shortcuts:**
- NO new SQLAlchemy table / model for user notes; reuse `AnalystNote` per DEC-M5-NOTE-001.
- NO new `_RESERVED_ACTIONS` entry for user notes (the existing `AnalystNote` table is the authority).
- NO new `_RESERVED_ACTIONS` entry for falsified-prediction state (rides on the existing `_predictions_log` sentinel — same row, updated payload).
- NO modification of `core/workspace.py` — it stays BYTEWISE UNCHANGED in M-5 (F59 invariant preserved).
- NO modification of `models/database.py` — DEC-DB-002 + DEC-M5-NOTE-001 preserved.
- NO modification of `dossier/slots.py` (DEC-M2-DOSSIER-004 preserved; M-5 does NOT add per-strategy persistence).
- NO modification of `dossier/panel.py` (M-1 byte-identical).
- NO modification of `gamification/scoring.py`, `gamification/celebrations.py`, `core/streak.py`, `core/pivot_policy.py`, `core/event_bus.py`.
- NO Rich markup in dossier event text or in any new LLM tool output (F64).
- NO negative-points ScoreEvent emission (DEC-M4-PRED-006).
- NO extension of `infer_dossier_state_full(...)` signature (slot 9 evidence comes from existing `scos` + `notes` parameters).
- NO extension of `ExpectedEvidence` vocabulary (DEC-M4-PRED-002 frozen). `FalsificationEvidence` is the new vocabulary surface.
- NO cmd2 `do_note` in `core/console.py` (DEC-M5-NOTE-002 — chat-meta + LLM tool only).
- NO cross-hunt rescan for falsification evidence (DEC-M5-FALSIFY-004 — current-hunt evidence + per-prediction temporal counter only).
- NO double-persist of falsification events.
- NO refactor of `tools.py` / `console.py` / `chat.py` beyond the documented wiring + new LLM tool registration + new meta-command branch.

**rollback_boundary:** single feature branch revertible as one merge commit. Revert restores M-4 byte state; removes the M-5 dossier/predictions.py + dossier/slot_inference.py + dossier/scoring.py + agent/tools.py + core/console.py + agent/chat.py + dossier/__init__.py edits; restores M-4 `_predictions_log` envelope `schema_version=1` serializer. Workspaces written by M-5 will have `_predictions_log` rows at `schema_version=2`; after revert, M-4's deserializer raises `RuntimeError` ("schema_version=2 newer than runtime schema_version=1; data was written by a different AP version"). This is the documented loud-failure behavior per DEC-M4-PERSIST-003 — the user receives a clear message rather than silent corruption. Documented manual mitigation: one-line SQL `DELETE FROM score_events WHERE action = '_predictions_log';` to discard the predictions log after revert. Historical `AnalystNote` rows persist after revert (no impact — the table is unchanged). No schema migrations, no settings changes, `streak.json` untouched.

**acceptance_notes:** the implementer should treat M-5 as a "fill in the deferred surfaces M-4 deliberately left for me" slice rather than as a new authority. Most M-5 work is additive extensions to M-4 modules. The single biggest risk is silently breaking the M-4 `_predictions_log` JSON contract by emitting v2 envelopes without preserving v1-read capability — the implementer must include the v1-deserialize regression test in the first pytest run and verify it passes before declaring readiness.

**ready_for_guardian_definition:**
- All required_tests green; full suite green ≥ M-4 baseline (2178) + new M-5 tests.
- All forbidden-file `git diff main` outputs empty (paste each verifying the file is byte-identical), EXCEPT `dossier/state.py` which MAY have doc-only edits (paste the diff for reviewer confirmation if non-empty).
- `core/workspace.py` diff is empty (M-5 BYTEWISE invariant; this is stronger than M-4's narrow-edit clause).
- `models/database.py` diff is empty.
- `dossier/slots.py` diff is empty.
- Phase 17H appended to `MASTER_PLAN.md` AND committed in the same commit as source (AP #74 orphan-prevention; M-3 / M-4 demonstrated the pattern works).
- Phase 17G status flipped: `in-progress` → `completed (landed 2026-06-02, merge f928149, impl 1b1a2b0)`. M-4 closeout drift fixed in this commit.
- "Active Phase Pointer" tail-line updated from `W-68-M4-PERSISTENT-DOSSIER` to `W-68-M5-DENIAL-STRATEGIES`.
- Plan Status table gains a Phase 17H row.
- `dossier/__init__.py` exports the M-5 public symbols listed under required_integration_points; no surprise additions.
- Implementer commit message follows `feat(dossier):` Phase 17 prefix, references `#68` + `DEC-M5-DENIAL-001..003` + `DEC-M5-NOTE-001..003` + `DEC-M5-FALSIFY-001..008`.

---

## 8. Scope Manifest

**Allowed / Required (the implementer MUST touch these):**
- `src/adversary_pursuit/dossier/slot_inference.py` (EXTEND: slot 9 extractor + DGA helper + slot 9 leaves the deferred set)
- `src/adversary_pursuit/dossier/predictions.py` (EXTEND: FalsificationEvidence dataclass + FalsificationResult + falsify_predictions + mark_confirmed_or_falsified + PersistedPrediction schema bump + v2 serializer)
- `src/adversary_pursuit/dossier/scoring.py` (EXTEND: emit_dossier_prediction_falsified_event helper; M-3 emitters byte-identical)
- `src/adversary_pursuit/dossier/__init__.py` (export new symbols)
- `src/adversary_pursuit/agent/tools.py` (hunt-site wiring per §2.1; new `create_dossier_note` + `falsify_dossier_prediction` LLM tools + dispatchers; extend `create_dossier_prediction` schema; widen `_DOSSIER_ACTIONS` filter to 3-tuple; capture `created_at_hunt_count` in `_execute_create_dossier_prediction`)
- `src/adversary_pursuit/core/console.py` (hunt-site wiring mirror — falsification engine call sequence; no new cmd2 commands)
- `src/adversary_pursuit/agent/chat.py` (NEW `note <text>` meta-command branch + help-table row)
- `tests/test_dossier_slot_inference.py` (extend — slot 9 extractor coverage)
- `tests/test_dossier_predictions.py` (extend — FalsificationEvidence + falsify_predictions + mark_confirmed_or_falsified coverage)
- `tests/test_dossier_predictions_serialization.py` **(NEW)** OR extension to an existing file — JSON envelope v1/v2 round-trip + schema_version=3 raises
- `tests/test_dossier_scoring.py` (extend — falsified event + slot 9 transition emission)
- `tests/test_dossier_state.py` (extend — apply_predictions_overlay with falsified entries)
- `tests/test_agent_tools.py` (extend — new tools + hunt-site falsification + F64 gate)
- `tests/test_chat_dossier_metacommand.py` (extend — `note <text>` meta-command coverage)
- `tests/test_dossier_persistence_integration.py` (extend — Stage B + Stage C of §5 acceptance test)
- `tests/test_dossier_get_state_tool.py` (extend — slot 9 now real)
- `MASTER_PLAN.md` — Phase 17H section + Phase 17G status flip + Plan Status table row + "Active Phase Pointer" tail-line update. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention).**

**Forbidden (preserved authorities):**
- `src/adversary_pursuit/core/workspace.py` (F59 — BYTEWISE UNCHANGED in M-5; stronger than M-4's narrow-edit clause)
- `src/adversary_pursuit/models/database.py` (DEC-DB-002 + DEC-M5-NOTE-001 — no schema change; no new model)
- `src/adversary_pursuit/dossier/slots.py` (M-1/M-2/M-4 byte-identical — `SLOT_WEIGHTS` + `PredictionRecord` + `DenialStrategyRecord` scaffolds preserved)
- `src/adversary_pursuit/dossier/panel.py` (M-1 byte-identical)
- `src/adversary_pursuit/dossier/state.py` (M-4 byte-identical OR doc-only edits — no API change, no new function, no new persistence)
- `src/adversary_pursuit/gamification/scoring.py` (M-3 byte-identical — no new `DEFAULT_RULES` row for falsification events)
- `src/adversary_pursuit/gamification/celebrations.py` (F63 invariant)
- `src/adversary_pursuit/core/streak.py` (F62 invariant)
- `src/adversary_pursuit/core/pivot_policy.py` (F60 invariant)
- `src/adversary_pursuit/core/event_bus.py` (F60 invariant — no new bus subscriber)
- `src/adversary_pursuit/gamification/modes.py`, `src/adversary_pursuit/agent/runner.py` (C-1/C-2 territory; F64 panel separation)
- `src/adversary_pursuit/modules/**` (no module changes)
- `pyproject.toml`, hooks, settings, `CLAUDE.md`, `agents/`, `runtime/`

**Expected state authorities touched:**
- workspace SQLite `score_events` table (read + sentinel-row payload update via existing `save_predictions_log`; no schema change; no new reserved action)
- workspace SQLite `notes` table (write via existing `WorkspaceManager.add_note()` public method; no schema change; no new public method on workspace.py)
- in-memory `DossierState` (read at hunt start from persisted snapshot; written at hunt end via fresh inference + overlay — M-4 flow unchanged)
- in-memory `list[PersistedPrediction]` (read at hunt start; mutated for confirmations AND falsifications; written at hunt end)

---

## 9. Decision Log (Phase 17H / M-5 binding)

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M5-DENIAL-001** | Slot 9 (Denial / Deception) v1.0 extractor vocabulary is three evidence categories: (a) DGA-shaped domain names (consonant-to-vowel ratio ≥ 3 AND label length ≥ 12); (b) fast-flux infrastructure hints (`x_ap_dns_ttl ≤ 60` extension on ipv4/ipv6 SCOs — forward-compatible reserve, no current AP module surfaces it); (c) denial / evasion keywords in analyst notes via a closed `_DENIAL_KEYWORDS` frozenset. The extractor is implemented in `dossier/slot_inference.py` as a pure function `_extract_denial(scos, notes)`, parallel to the existing `_extract_motivation` helper. `DossierSlotName.DENIAL` is removed from the always-DEFERRED set; the slot now returns real `empty / partial / filled` status. | Smallest vocabulary that covers the issue #68 user story ("confusing / denying / discouraging further attack progress") without becoming a TTP-classification engine. Mirrors the M-2 motivation extractor's "closed keyword frozenset + simple threshold" shape, which has been operational since 2026-05-29 and has caused zero false-positive complaints. Richer detection (sandbox-detection IOCs from SCO extensions, multi-stage TTP cross-reference, registrar-rotation behavior) is deliberately deferred to M-7 or a later slice if the v1.0 vocabulary proves too narrow against real workspaces. |
| **DEC-M5-DENIAL-002** | Slot 9 status thresholds: EMPTY = 0 evidence items; PARTIAL = ≥ 1 evidence item in any single category; FILLED = ≥ 1 item across ≥ 2 distinct categories. The "cross-category corroboration" threshold mirrors DEC-M1-DOSSIER-INFERENCE-STATUS-001 (≥ 2 distinct types → FILLED) and the M-2 motivation extractor's "≥ 2 categories → FILLED" rule. | Single-category evidence (e.g., one DGA domain with no corroborating note) is suggestive but not analytically sufficient — denial / deception claims require evidence from multiple angles to graduate from "interesting noise" to "this actor uses denial tactics." The threshold is intentionally conservative so the slot is testable with synthetic fixtures, parallel to M-1's mapping discipline. |
| **DEC-M5-DENIAL-003** | DGA shape detector is a deterministic pure-function helper `_is_dga_shaped(label) -> bool` with the rule `len(label) >= 12 AND consonant_count / max(vowel_count, 1) >= 3`. Misses dictionary-word DGAs by design; catches some legitimate-but-cryptic domains (e.g., long base32-encoded subdomains) by design. The unit tests document both classes of error. | Pure-function detector keeps the slot extractor read-only and side-effect-free (Sacred Practice 12 + DEC-M1-DOSSIER-001 inference-authority preservation). Avoids pulling in n-gram entropy / frequency-analysis libraries (no new dependencies). Conservative MVP behavior is fine for v1.0 because the cross-category corroboration rule (DEC-M5-DENIAL-002) prevents false-positive FILLED status from DGA-only noise. |
| **DEC-M5-NOTE-001** | User-note authoring persistence authority is the existing `AnalystNote` SQLAlchemy table (`models/database.py:218`) + the existing `WorkspaceManager.add_note(content, stix_object_id=None)` public method (workspace.py:647). Both the chat `note <text>` meta-command and the `create_dossier_note(text)` LLM tool call `add_note()` directly. NO new SQLAlchemy table, NO new `_RESERVED_ACTIONS` entry, NO `core/workspace.py` change. The dispatch context's instruction to piggyback on the F63 sentinel-row pattern for user notes is explicitly overridden by this DEC — the dispatch context did not catch that `AnalystNote` already exists and is the canonical authority for notes. | Sacred Practice 12 violation rejected: the F63 sentinel-row pattern is correct when no table exists for the data type; `AnalystNote` already exists and is already the canonical authority (used by motivation extractor, M-4 prediction validation `note_keyword_any`, report.py analyst-notes section, workspace stats, badge counts). Adding a parallel `_dossier_user_note` sentinel action would silently break every existing `_read_analyst_notes` caller and would require updating motivation extractor + M-4 validation + report.py + workspace stats to read from two sources. Reusing the existing table is the smaller, safer change. **The dispatch context's "no schema change" intent is honored trivially because the table already exists.** |
| **DEC-M5-NOTE-002** | User-note authoring surface in M-5 is the chat meta-command `note <text>` (in `agent/chat.py`) + the `create_dossier_note(text)` LLM tool (in `agent/tools.py`). NO cmd2 `do_note` command is added to `core/console.py`. | cmd2 REPL is a power-user surface per ADR-010; v1's user-facing front door is `ap chat`. Adding cmd2 parity is a small future slice if requested. Keeping the M-5 scope to the chat surface honors the "Simple Task Fast Path" boundary — M-5 already touches 8+ files, and a cmd2 `do_note` adds another file and another integration surface for negligible product value. |
| **DEC-M5-NOTE-003** | LLM tool `create_dossier_note(text)` v1.0 schema accepts only the `text` field. The existing `WorkspaceManager.add_note(content, stix_object_id=None)` supports linking a note to a specific SCO, but the LLM tool surface omits the `stix_object_id` parameter in M-5. | Minimal surface for v1.0. Most analyst notes are not directly linked to a specific SCO — they are observations about the actor as a whole. Adding the `stix_object_id` field to the LLM tool schema adds prompt-engineering surface (the LLM must learn when to populate it) without unlocking a measured user need. A future slice may extend the tool to accept the optional argument if specific-SCO linkage proves useful. |
| **DEC-M5-FALSIFY-001** | Active falsification engine ships as a new `falsify_predictions(predictions, new_scos, new_notes, hunt_count) -> list[FalsificationResult]` function in `dossier/predictions.py`, parallel to M-4's `validate_predictions`. Falsified-prediction state rides on the existing `_predictions_log` sentinel row (M-4 storage authority) — no new `_RESERVED_ACTIONS` entry, no second sentinel action. | Mirrors the M-4 validation engine shape so the implementer and reviewers can navigate it by analogy. Reusing the existing sentinel row avoids the parallel-authority bug that would result from a second `_predictions_falsified_log` sentinel — falsification state is part of the prediction's lifecycle, so it belongs in the same persisted record. |
| **DEC-M5-FALSIFY-002** | `FalsificationEvidence` vocabulary v1.0 mirrors `ExpectedEvidence` in the negative sense plus a temporal-window field: `negative_sco_type, negative_value_regex, negative_asn_in, contradiction_keyword_any, stale_after_n_hunts`. All non-None evidence fields are ANDed (same as `ExpectedEvidence`). Empty (all-None) FalsificationEvidence is rejected by the factory with loud `ValueError` (Sacred Practice 5). Sentinel exception: `stale_after_n_hunts` alone is a valid configuration (pure temporal rule with no contradiction-evidence criteria). | Smallest vocabulary covering the M-5 user story (auto-falsify on `.cn` when predicted `.ru`; auto-falsify on contradiction keyword in note; auto-falsify after N hunts of no confirmation). Typed dataclass over freeform dict keeps the LLM tool schema honest. Strict mirror of the `ExpectedEvidence` shape makes the implementer's job mechanical. The stale-rule sentinel exception is the one shape deviation from `ExpectedEvidence` — it lets predictions carry "just give up after N hunts" without forcing a contradiction-evidence shape. |
| **DEC-M5-FALSIFY-003** | The `stale_after_n_hunts` temporal rule is computed against the workspace's `get_module_runs()` row count, not wall-clock time. The PersistedPrediction's `created_at_hunt_count` field (NEW in M-5 schema v2) captures the run count at creation time. The falsifier transitions a pending prediction to `falsified` when `(current_hunt_count - prediction.created_at_hunt_count) >= stale_after_n_hunts`. Legacy M-4 entries with `created_at_hunt_count=0` (default for missing field) are skipped by the stale-rule path — the falsifier treats `0` as "creation hunt count unknown — skip stale check." | Workspace-relative counters are robust to clock skew, time-zone changes, and `ap chat` idle periods. Wall-clock rules would require the user to keep AP running continuously for the stale rule to fire. The "skip stale check on `created_at_hunt_count=0`" path lets M-5 deploy against M-4-authored workspaces without retroactively falsifying old predictions on first M-5 hunt. |
| **DEC-M5-FALSIFY-004** | Falsification evidence scope = **current-hunt evidence only** (mirrors DEC-M4-PRED-003). The contradiction-evidence categories (`negative_*`, `contradiction_keyword_any`) match against current-hunt SCOs and notes. The temporal `stale_after_n_hunts` rule is the ONLY cross-hunt signal — it counts `get_module_runs()` row count, not a workspace-wide rescan of historical SCOs / notes. | Honors DEC-M4-PRED-003. A workspace-wide rescan for contradiction evidence would be expensive (full SCO + note enumeration on every hunt) and would create surprising behavior (a note added 5 hunts ago could falsify a prediction created today). Restricting to current-hunt evidence keeps the engine predictable and per-hunt-bounded. A `revalidate_all_predictions(workspace)` repair tool that does a workspace-wide rescan can land as a separate small slice if needed. |
| **DEC-M5-FALSIFY-005** | Falsification ScoreEvent: `action="dossier_prediction_falsified"`, `points=0`, `indicator=prediction_id`, `rule_description=f"Dossier prediction falsified: {reason}"`. The `points=0` value is the binding canon from DEC-M4-PRED-006; M-5 explicitly does NOT relitigate the negative-points question. The event flows through `store_score_events` so F62 streak chain sees it as a no-op zero-points event (neither resets nor extends the streak), and F63 milestone catch-up sees the unchanged total. The new event action is added to the F64 `_DOSSIER_ACTIONS` filter (widened to 3-tuple) so the event text is stripped from LLM-facing summary text. | Inheriting DEC-M4-PRED-006 as canon keeps M-5 in scope (negative-points changes would require streak/milestone math changes — large blast radius). The "reckless guessing should cost score" intuition is the documented M-7 narrative-feedback surface; M-5 does not relitigate it. The event still flows through `store_score_events` so the persisted history reflects the falsification (visible in workspace audit trails) without affecting score totals. |
| **DEC-M5-FALSIFY-006** | Manual override LLM tool `falsify_dossier_prediction(prediction_id, reason)` accepts a prediction id + a plain-text reason. Idempotent: already-falsified or already-validated predictions return a no-op JSON message rather than raising. Persists via the same `save_predictions_log` path used by auto-falsification. | Analysts need a way to mark predictions wrong without authoring contradiction evidence in advance — sometimes you just know an actor pivoted before any matchable IOC surfaces. Idempotency is non-negotiable for LLM-driven tool calls (the agent may retry on transient failure). The reason field flows into both the score event `rule_description` and the persisted PersistedPrediction's `validated_at` timestamp (reused as the conclusion timestamp). |
| **DEC-M5-FALSIFY-007** | `PersistedPrediction` dataclass gains two optional fields at the end: `falsification_evidence: FalsificationEvidence | None = None` and `created_at_hunt_count: int = 0`. Both are non-breaking additions (default values cover legacy M-4 entries). | Schema-additive extension preserves M-4's persistence contract while unlocking the M-5 falsification surface. The default values make M-4-authored workspaces deserialize cleanly into M-5 dataclasses without an upgrade migration. |
| **DEC-M5-FALSIFY-008** | The `_predictions_log` JSON envelope schema version bumps from `1` to `2`. The deserializer accepts both v1 and v2 envelopes (v1 deserializes with the M-5 fields at their default values). The serializer always emits v2. v3+ envelopes raise loud `RuntimeError` (DEC-M4-PERSIST-003 pattern, one version bump). The `_dossier_state_snapshot` envelope stays at `schema_version=1` (M-5 does not change DossierState shape). | Loud-failure handshake pattern preserved (Sacred Practice 5). Single envelope advances so the implementer only updates one serializer + one deserializer. v1-read capability is the regression that protects M-4-authored workspaces from breaking on first M-5 hunt — the implementer must include the v1-deserialize test in the test matrix. |

---

## 10. Open question for the user (none)

No user-decision boundary is required to start the implementer. The dispatch context's "user notes via F63 sentinel-row pattern" instruction is overridden by DEC-M5-NOTE-001 because the existing `AnalystNote` table is the correct authority — this is a planner-layer correction of a dispatch context that did not inspect the codebase first. The planner's call honors the dispatch context's underlying intent ("no schema change") with the simpler implementation ("reuse the existing table"). If the implementer surfaces an unforeseen blast-radius (e.g., the M-5 schema_version=2 bump triggers a regression in a currently-green M-4 test), the implementer halts and reports — that is a planner re-stage trigger, not an in-flight design call.

---

## 11. Subsequent Workflow Cue

After M-5 lands, the recommended next workflow is **M-6 — Dossier-Aware Auto-Pivot Policy** per `.claude/plans/dossier-reframe-v2-roadmap.md` §M-6. M-6 extends F60's 3-gate pivot policy with a fourth "would this pivot fill an empty high-value slot?" input; the dossier state authority (M-4) and the now-real slot 9 (M-5) give M-6 the data it needs. M-6 is independent of M-7 (Reports / Celebrations / Badges Dossier-Aware Upgrade) — both depend on M-1..M-5 and can land in either order. M-8 (Cleanup, Closeout, and Novel-Method Achievement) blocks on M-7 only.

C-3 (Philosophy + Bureaucratese modes — `sun_tzu`, `bruce_lee`, `bureaucrat`) remains independent of the dossier roadmap (DEC-30-CHARACTER-V2-007) and may land in any wave.

The Targeting slot 5 inference path remains DEFERRED after M-5. A future slice (not currently scheduled — likely M-7 or M-8) will introduce either a user-supplied victim-industry profile or a victim-industry extractor from SCO data that AP modules surface in a future update. The planner that opens that slice records the trigger criteria as `DEC-MX-TARGETING-001`.
