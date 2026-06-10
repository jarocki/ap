# Dossier M-9 — Crowdsourced Dossier Comparison + Public Actor Library

**Workflow:** `w-68-m9-crowdsourced-dossiers` / goal `g-68-m9-crowdsource` / work-item `wi-68-m9-impl-01`.
**Branch:** `feature/68-m9-crowdsourced-dossiers`. **Worktree:** `/Users/jarocki/src/ap/.worktrees/feature-68-m9-crowdsourced-dossiers`.
**Base:** AP main at merge `9a6a550` (C-4 closure head; impl `0d1f227`). M-8 closed at `16acaa3`; C-4 closed at `9a6a550`.
**Issue:** #68 — M-9 line per `.claude/plans/dossier-reframe-v2-roadmap.md` §7 and DEC-68-DOSSIER-REFRAME-009.

## 1. Goal (verbatim binding)

M-9 closes the v0.3.x dossier roadmap's last named open surface: **Crowdsourced Dossier Comparison + Public Actor Library** per DEC-68-DOSSIER-REFRAME-009. The slice delivers three co-shipped capabilities behind one feature branch:

1. **Dossier STIX 2.1 bundle export** — `dossier/export.py::export_dossier(workspace_mgr, actor_identifier)` returns a STIX 2.1 bundle JSON string composed of: (a) all SCOs already in the workspace (recovered from `WorkspaceManager.get_stix_objects()`), (b) all relationships already in the workspace (via the existing `core/graph.py::RelationshipGraph` build path), (c) a synthesized `threat-actor` SDO whose fields are derived from the persistent DossierState (M-4 `load_dossier_state`) — slot 1 Identity contributes `aliases`; slot 6 Capability contributes `resource_level`; slot 7 Motivation contributes `roles`; slot 3 Infrastructure summary lives under `x_ap_dossier_infrastructure`; (d) the Predictions Log (M-4 `load_predictions_log`) encoded as a custom STIX bundle extension `x_ap_predictions` attached to the threat-actor SDO, (e) Analyst Notes (M-5 `AnalystNote` rows) encoded as `x_ap_analyst_notes`, (f) per-bundle metadata `x_ap_version` / `x_ap_exported_at` / `x_ap_workspace_id` / `x_ap_dossier_schema_version`. **Read-only consumer** of M-4 / `WorkspaceManager`; no workspace mutation, no x_ap_* invention beyond the M-9-named custom extension namespace.

2. **Dossier STIX bundle import** — `dossier/import_.py::import_dossier(bundle_json)` parses a STIX 2.1 bundle string into an in-memory `ImportedDossier` dataclass. **Read-only shape; never mutates the workspace SQLite.** Validation rejects malformed bundles with `ValueError` carrying a diagnostic message (Sacred Practice 5 — loud failure). The parse path round-trips through `stix2.parse(bundle, allow_custom=True)` so spec-version regressions surface at import time.

3. **Dossier comparison** — `dossier/comparison.py::compare_dossiers(local, remote)` returns a `ComparisonReport` dataclass that captures slot-by-slot deltas, completion ratios, validated-prediction ratios, unique-slot-fills per side, and a one-line summary string. **Pure function**: no I/O, no LLM call, no workspace mutation, no `dossier/state.py` write. Single-authority for the "how do these two dossiers compare?" question (Sacred Practice 12).

4. **Public library (local opt-in)** — `dossier/export.py` also hosts three helpers — `publish_to_library`, `list_library`, `load_from_library` — that read/write static dossier files at `~/.ap/dossier_library/<actor_identifier>.json`. **No network, no upload, no federation.** The directory is created with 0o700 permissions on first publish. Publication requires `AP_DOSSIER_PUBLISH=on` env var (opt-in) so accidental disclosure is mechanically blocked.

5. **LLM tools (28 → 30)** — two new tools registered in `agent/tools.py::create_tools`:
   - `export_dossier` — wraps `export_dossier` + optional `publish_to_library`. Returns the bundle JSON string (or the library path when published).
   - `compare_dossier` — wraps `import_dossier` + `compare_dossiers`. Accepts either a local path or an `actor_identifier` resolved via `load_from_library`. Returns a plain-ASCII rendering of the `ComparisonReport`.

6. **Console + chat meta-commands** — `dossier export [<path>|--publish]` and `dossier compare <actor_identifier|path>` are added to `core/console.py::do_dossier` (or its meta-command equivalent) and the chat-meta dispatcher in `agent/chat.py`. F64-compliant plain-ASCII output (no Rich markup smuggled through LLM-facing surfaces).

After M-9 lands: the v0.4.x dossier roadmap surface opens (M-9 is the only currently-named v0.4.x slice). Tool count 28 → 30. **`core/workspace.py` BYTEWISE UNCHANGED.** **`models/database.py` BYTEWISE UNCHANGED.** Every `dossier/*.py` EXCEPT three new modules + `__init__.py` exports BYTEWISE UNCHANGED. F59 / F60 / F62 / F63 / F64 + Sacred Practice 12 invariants preserved by construction.

## 2. Why now / what unblocked this

M-8 closed the v0.3.x dossier roadmap (merge `16acaa3`). C-4 closed the v2 character roadmap (merge `9a6a550`). Both gates are green; M-9 is the only on-roadmap slice with no unresolved dependency. The persistent M-4 DossierState API (`load_dossier_state`) and the M-4 PredictionsLog API (`load_predictions_log`) supply the exact serialization inputs the M-9 export demands. The M-5 AnalystNote API supplies the notes surface. The existing `core/graph.py::export_stix_bundle` already proves the python-stix2 spec-compliance round-trip; M-9 BUILDS ON TOP of that path rather than duplicating it.

The Reckoning's 7-week-latency call (DEC-68-DOSSIER-REFRAME-009) resolves once M-9 lands.

## 3. Architecture

### 3.1 Module layout

Three NEW modules in the `dossier/` package — co-located because they share the dossier vocabulary authority. The library helpers ride inside `export.py` rather than spawning a fourth module (minimal-codebase principle; the helpers are file-system glue, not a distinct domain).

| Module | Public surface | Authority |
|---|---|---|
| `dossier/export.py` | `export_dossier(workspace_mgr, actor_identifier) -> str` ; `publish_to_library(bundle_json, actor_identifier) -> Path` ; `list_library() -> list[Path]` ; `load_from_library(actor_identifier) -> str` ; `library_root() -> Path` ; `library_publish_enabled() -> bool` | Sole authority for "serialize this workspace's dossier to a STIX 2.1 bundle" + "where do local library files live" |
| `dossier/import_.py` | `import_dossier(bundle_json) -> ImportedDossier` ; `ImportedDossier` dataclass | Sole authority for "parse this bundle into a comparable in-memory shape" |
| `dossier/comparison.py` | `compare_dossiers(local, remote) -> ComparisonReport` ; `ComparisonReport` dataclass | Sole authority for "how do these two dossiers compare" |

`dossier/__init__.py` extends `__all__` with these names; no other dossier source touched.

### 3.2 STIX 2.1 mapping table (DEC-M9-STIX-MAPPING-001)

```
DossierSlot                 →  STIX target field on threat-actor SDO
-------------------------------------------------------------------
1. IDENTITY                 →  threat-actor.aliases (list[str]; sourced from
                               distinct values of email-addr / user-account /
                               x509-certificate SCO values; deduped, sorted)
2. TTPS                     →  x_ap_dossier_ttps (custom prop on threat-actor;
                               list[{sco_type, distinct_count, evidence_count}])
3. INFRASTRUCTURE           →  x_ap_dossier_infrastructure (custom prop;
                               same shape)
4. TIMING                   →  x_ap_dossier_timing (custom prop;
                               {status, distinct_hour_count})
5. TARGETING                →  x_ap_dossier_targeting (custom prop;
                               {status, distinct_industry_count})
6. CAPABILITY               →  threat-actor.resource_level (str;
                               mapped from filled→'organization',
                               partial→'club', empty/deferred→omitted)
7. MOTIVATION               →  threat-actor.roles (list[str]; sourced from
                               slot inference output; values among
                               {'financial-gain','espionage','hacktivism',
                               'unknown'}; deferred status → roles omitted)
8. PREDICTIONS              →  x_ap_predictions (list[PredictionEnvelope];
                               see §3.3 schema)
9. DENIAL                   →  x_ap_dossier_denial (object;
                               {status, distinct_strategy_count})
```

Per-bundle metadata lives in custom bundle properties (set on the threat-actor SDO, not on the bundle root, because the python-stix2 Bundle class strips unknown top-level keys):

```
x_ap_version                — AP package version string at export time
x_ap_exported_at            — ISO-8601 UTC timestamp
x_ap_workspace_id           — workspace name (workspace_mgr.active)
x_ap_dossier_schema_version — integer "1"
x_ap_actor_identifier       — the actor_identifier argument verbatim
x_ap_analyst_notes          — list[{content: str}] (M-5 AnalystNote.content
                              ordered by id ascending)
```

### 3.3 Custom extension schemas (DEC-M9-STIX-MAPPING-002)

`x_ap_predictions` is a list of `PredictionEnvelope` JSON objects. Each envelope mirrors the on-disk `PersistedPrediction` shape one-to-one to keep import round-trip lossless:

```
{
  "prediction_id": str,
  "text": str,
  "slot": str (DossierSlotName.value),
  "status": "pending" | "validated" | "falsified",
  "created_at": str (ISO-8601 UTC),
  "validated_at": str | null,
  "validated_by_sco_id": str | null,
  "expected_evidence": {
    "sco_type": str | null,
    "value_regex": str | null,
    "asn_in": list[int] | null,
    "note_keyword_any": list[str] | null
  },
  "falsification_evidence": {...} | null,
  "created_at_hunt_count": int
}
```

`x_ap_analyst_notes` is a list of `{ "content": str }` objects. No PII redaction — content is verbatim by design.

### 3.4 STIX bundle assembly path

`export_dossier` body shape (pseudo-code):

```
1. scos = workspace_mgr.get_stix_objects()                       # F59 provenance preserved
2. graph = RelationshipGraph.build_from_workspace(workspace_mgr) # M-9 reuses #59 path
3. base_bundle = graph.export_stix_bundle()                      # DEC-59-STIX-PROVENANCE-005
   # base_bundle is a fully spec-compliant dict; objects include SCOs + SROs.
4. local_state     = load_dossier_state(workspace_mgr) or default_deferred_state()
5. local_preds     = load_predictions_log(workspace_mgr)
6. local_notes     = _get_analyst_notes(workspace_mgr)           # _get_analyst_notes
                                                                  # exists in tools.py;
                                                                  # reuse pattern, do not
                                                                  # add a workspace method
7. ta = _synthesize_threat_actor_sdo(
         actor_identifier, local_state, local_preds, local_notes
       )                                                          # custom-prop carrier
8. base_bundle["objects"].append(stix2.parse(ta_dict, allow_custom=True).serialize() ...)
   # In practice the bundle is rebuilt one more time via stix2.v21.Bundle(allow_custom=True)
   # so spec-compliance is asserted across the entire payload.
9. return json.dumps(bundle_dict, sort_keys=True)
```

Step (7) builds a `threat-actor` SDO dict and round-trips it through `stix2.parse(..., allow_custom=True)` to guarantee spec-version + id derivation. Round-trip preserves `x_ap_*` custom props verbatim because `allow_custom=True` is the same flag F59 already relies on.

### 3.5 Import shape

```python
@dataclass(frozen=True)
class ImportedDossier:
    actor_identifier: str
    slot_states: dict[DossierSlotName, SlotStatus]
    predictions: list[PersistedPrediction]
    analyst_notes: list[str]
    metadata: dict[str, str]    # x_ap_version / x_ap_exported_at / etc.
```

`import_dossier` reconstructs `slot_states` from the threat-actor SDO custom props using the inverse of the §3.2 mapping. `predictions` are rehydrated via the existing `_deserialize_predictions` helper if the on-disk schema-version line matches; otherwise the envelope schema_version 1 path is used.

### 3.6 Comparison shape

```python
@dataclass(frozen=True)
class ComparisonReport:
    actor_identifier: str
    slot_diff: dict[DossierSlotName, tuple[SlotStatus, SlotStatus]]
    completion_local: float       # see DEC-M9-COMPLETION-001
    completion_remote: float
    unique_to_local: list[DossierSlotName]
    unique_to_remote: list[DossierSlotName]
    prediction_validation_ratio_local: float
    prediction_validation_ratio_remote: float
    summary_line: str
```

`compare_dossiers` is pure — no global state read or write. The `summary_line` is plain ASCII (F64).

### 3.7 Public library layout

```
~/.ap/dossier_library/                      (mode 0o700, owner-only)
    <actor_identifier>.json                 (one file per actor; UTF-8 STIX bundle)
```

Override via `AP_DOSSIER_LIBRARY=<path>` env var (test isolation + power-user customization).

Writes require `AP_DOSSIER_PUBLISH=on`. Reads are unconditional (intended use: import as priors).

`actor_identifier` is validated `_validate_actor_identifier()`:
- non-empty
- matches `^[A-Za-z0-9._-]{1,128}$` (rejects path traversal, NUL bytes, slashes)
- duplicated in the bundle's `x_ap_actor_identifier` metadata for on-import verification

### 3.8 What is NOT touched

- `core/workspace.py` — BYTEWISE UNCHANGED (F59 sole authority for `x_ap_*` on SCOs preserved; the M-9 custom props live on the synthesized threat-actor SDO that is created in-process, never round-tripped through `store_stix_objects`).
- `models/database.py` — BYTEWISE UNCHANGED (no new tables; the library is plain JSON files).
- `dossier/state.py` / `predictions.py` / `scoring.py` / `slot_inference.py` / `panel.py` / `slots.py` / `novelty.py` / `dossier_report.py` — BYTEWISE UNCHANGED.
- `agent/runner.py` — BYTEWISE UNCHANGED (no new persona surface; no narration of comparison events).
- `gamification/*` — BYTEWISE UNCHANGED. M-9 ships **no new badge** (see §5.6 DEC-M9-DEFER-001).
- `core/{event_bus,pivot_policy,dossier_pivot,streak,report,dossier_report,config}.py` — BYTEWISE UNCHANGED.
- `_DOSSIER_ACTIONS` in `agent/tools.py` — UNCHANGED at 4-tuple. M-9 emits **no new ScoreEvent**; export/import/compare are infrastructure operations, not scored events.
- `models/stix.py` — UNCHANGED. M-9 does not add new module-side STIX helpers; the custom-property assembly lives in `dossier/export.py`.

## 4. Demo / Stage acceptance

| Stage | Demonstration | Captured to |
|---|---|---|
| **A** Tool registration | `python -c "from pathlib import Path; from adversary_pursuit.agent.tools import ToolContext, create_tools; ctx=ToolContext(config_dir=Path('/tmp/m9-test-cfg'),workspace_dir=Path('/tmp/m9-test-ws')); print(sorted([t['function']['name'] for t in create_tools(ctx)]))"` — list includes `export_dossier` AND `compare_dossier`; length == 30. | `tmp/evidence-m9-crowdsourced-comparison/tool_count_audit.txt` |
| **B** Export round-trip | Build a tmp workspace with 5 IPv4 + 2 domains + 1 prediction; call `export_dossier` → JSON; call `stix2.parse(bundle_json, allow_custom=True)` → succeeds; call `import_dossier(bundle_json)` → `ImportedDossier`; assert `imported.predictions == original_predictions`. | `tmp/evidence-m9-crowdsourced-comparison/export_roundtrip.txt` |
| **C** Comparison determinism | Two `ImportedDossier`s built from the same bundle compare to `slot_diff` entries all `(x, x)` and `completion_local == completion_remote`. | `tmp/evidence-m9-crowdsourced-comparison/comparison_idempotent.txt` |
| **D** Library opt-in | `AP_DOSSIER_PUBLISH` unset → `publish_to_library` raises `RuntimeError`; with `AP_DOSSIER_PUBLISH=on` → file written; `stat -f %Lp <dir>` → 700; `list_library()` returns the file. | `tmp/evidence-m9-crowdsourced-comparison/library_optin.txt` |
| **E** Malformed import | `import_dossier("{not a bundle}")` raises `ValueError`; `import_dossier(json.dumps({"type": "bundle"}))` raises `ValueError` (missing required fields). | `tmp/evidence-m9-crowdsourced-comparison/malformed_import.txt` |
| **F** Chat demo | `ap chat`; create a workspace; run two modules; `export_dossier`; `mode default`; `compare_dossier <actor_id>` from library — response shows slot diff lines + completion ratios. | `tmp/evidence-m9-crowdsourced-comparison/chat_export_compare.txt` |

## 5. Decision Log (binding for Phase 17N / M-9)

### Content decisions

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-M9-ACTOR-ID-001** | The `actor_identifier` is an explicit user-provided string argument. Default when omitted: `workspace_mgr.active` (workspace name). Validated against `^[A-Za-z0-9._-]{1,128}$` at the export and library boundaries. NOT auto-derived from the first `intrusion-set` SDO id (M-4 dossiers do not currently materialize `intrusion-set` SDOs; auto-derivation would couple M-9 to a future M-X intrusion-set inference slice that does not exist yet). | Explicit string + safe-default keeps the LLM tool surface trivially testable and the library file-name space mechanically safe. Filesystem-safety regex blocks path traversal at the smallest abstraction. Coupling to workspace name preserves the M-4 / M-7 workspace=case association without inventing a parallel actor-registry authority. |
| **DEC-M9-STIX-MAPPING-001** | The slot→STIX mapping table in §3.2 is binding. STIX-native fields (`aliases`, `roles`, `resource_level`) carry slots 1 / 7 / 6. All other slots and their derived counts live under `x_ap_*` custom props on the threat-actor SDO. Slot 9 Denial is stored under `x_ap_dossier_denial` (NOT in `aliases` or `description`). | Round-trip via `stix2.parse(..., allow_custom=True)` preserves both spec-compliance AND custom-prop fidelity. Using STIX-native fields where the semantics match (Identity ↔ aliases) maximizes interop with OpenCTI / MISP. Placing denial under `x_ap_dossier_denial` (not freeform description) keeps the import path deterministic. |
| **DEC-M9-STIX-MAPPING-002** | `x_ap_predictions` and `x_ap_analyst_notes` are top-level custom properties on the threat-actor SDO (not on the bundle root) and use the literal field shapes in §3.3. The `x_ap_predictions` envelope mirrors `PersistedPrediction` one-to-one (including `falsification_evidence` and `created_at_hunt_count`) so import is lossless. | python-stix2's `Bundle` class strips unknown top-level keys; SDOs carry custom props losslessly with `allow_custom=True`. One-to-one envelope shape removes ambiguity between authoring and consuming code paths. |
| **DEC-M9-COMPLETION-001** | Completion math: `filled = 1.0`, `partial = 0.5`, `empty = 0.0`, `deferred = 0.0`. `completion_local` = sum(per-slot weight × per-slot factor) / sum(SLOT_WEIGHTS) — weighted by the M-3 `SLOT_WEIGHTS` table so high-value slots dominate the metric. Same formula on the remote side. | Strict 0/1 was rejected because it discards the partial-evidence signal that M-1's panel surface already exposes. Weighting by `SLOT_WEIGHTS` ties the comparison metric to the same authority M-3 already uses for slot-fill scoring (Sacred Practice 12); a future SLOT_WEIGHTS retune flows through unchanged. `deferred = 0` because deferred is a milestone-scoping marker, not a confidence claim. |
| **DEC-M9-PRED-RATIO-001** | `prediction_validation_ratio_*` = `validated / (validated + pending + falsified)` per side. When the denominator is 0, the field is `0.0` (NOT NaN — pure ratio is undefined for empty sets; 0.0 is the conservative null). | Mirrors the F63 milestone catch-up math discipline (integer-only / safe-default-zero). The "validated-over-all-known" framing matches the roadmap §7 framing of "whose Predictions Log has the highest validated-prediction ratio". |
| **DEC-M9-PRIVACY-001** | Bundles published to the library contain raw IOCs from the user's workspace verbatim. M-9 ships NO PII redaction layer. The opt-in `AP_DOSSIER_PUBLISH=on` env var is the consent gate; the README + the `dossier export --publish` chat help text MUST surface "publishing a dossier publishes the underlying IOCs verbatim — user responsibility". | A redaction layer at M-9 would invent a parallel "trusted vs untrusted IOC" authority that contradicts F59's single-authority-for-x_ap_* invariant. Surfacing the privacy contract at the consent boundary (env var + visible help text) is the minimal honest abstraction and matches the v1 Non-Goal "Federation" framing: nothing leaves the local file system unless the user moves the file themselves. |
| **DEC-M9-IMPORT-READONLY-001** | `import_dossier` does NOT write to the workspace SQLite. It returns an `ImportedDossier` value object. Comparison is the only consumer in M-9. A future M-X slice may add an "ingest priors" path; M-9 explicitly defers it. | "Import becomes ingest" is the single largest authority risk in this surface: a write-path import would create dual authority between `WorkspaceManager.store_stix_objects` (F59 SCO authority) and the M-9 importer (DossierState authority). Holding M-9 strictly read-only preserves F59 by construction. The conflict-resolution question raised in the dispatch context resolves trivially — read-only import cannot conflict. |
| **DEC-M9-LIBRARY-LOCATION-001** | Default library root: `~/.ap/dossier_library/`. Override via `AP_DOSSIER_LIBRARY` env var (absolute path). Directory created with `0o700` permissions on first publish via `Path.mkdir(mode=0o700, parents=True, exist_ok=True)` + an explicit `os.chmod(path, 0o700)` for robustness against pre-existing directories. | Mirrors the M-8 `~/.ap/dossier_novelty.sqlite` cross-workspace location pattern. `0o700` is the established AP cross-workspace permission floor for files that contain raw IOC content. Env-var override is the test-isolation and power-user pattern already used by `AP_NO_NOVELTY` / `AP_NO_BANNER`. |
| **DEC-M9-LIBRARY-OPTIN-001** | Library WRITES require `AP_DOSSIER_PUBLISH=on` (the literal string "on", case-insensitive). Any other value or unset → `publish_to_library` raises `RuntimeError` with a remediation message. Reads are unconditional. | Two-step consent: the user must intentionally enable publication. Loud-fail rather than silent skip honors Sacred Practice 5. Reads need no gate because importing a bundle the user already has on disk is not a disclosure event. |
| **DEC-M9-CONFLICT-001** | Import is read-only (DEC-M9-IMPORT-READONLY-001), so an imported bundle whose `actor_identifier` matches a local workspace's actor produces NO conflict — the imported shape is a value object held in memory by `compare_dossier` and discarded after the response is rendered. | Eliminates the entire conflict-resolution surface raised in the dispatch context's "Required decisions" §6. Single-authority discipline: workspace SQLite is the sole DossierState authority; the importer is a comparison-only consumer. |

### Tool surface decisions

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-M9-TOOL-EXPORT-001** | NEW LLM tool `export_dossier(actor_identifier: str \| None = None, publish: bool = False) -> str`. When `publish=True`, requires `AP_DOSSIER_PUBLISH=on` and returns the library file path string; otherwise returns the bundle JSON string. F64-compliant: no Rich markup, no score-event smuggling. | One tool covers both "give me the bundle" and "publish to library" — the opt-in env-var gate enforces consent independently of the LLM's `publish=True` flag, so a hostile prompt can't bypass user consent. |
| **DEC-M9-TOOL-COMPARE-001** | NEW LLM tool `compare_dossier(source: str) -> str`. The `source` string is resolved as a file path if it contains a path separator; otherwise treated as an `actor_identifier` and resolved via `load_from_library`. Returns a plain-ASCII rendering of the `ComparisonReport`. F64-compliant. | Single argument resolves the two practical use cases (compare to library entry; compare to a manually-shared file path) without inventing a multi-tool surface. Resolution rule is deterministic. |
| **DEC-M9-TOOLCOUNT-001** | LLM tool count grows 28 → 30 (M-8 floor 28 + 2 new tools). `tests/test_agent_tools.py` assertions at lines 183, 287, 1422 (currently `== 28`) update to `== 30`. | Concrete, mechanical floor. No tools are removed by M-9; the +2 is the only delta. |

### Scope / process decisions

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-M9-NO-NEW-BADGE-001** | M-9 ships NO new badge. The "Published 5 dossiers" candidate hinted in the dispatch context is DEFERRED to a future single-DEC slice once telemetry (or user request) shows the surface is worth a badge. | Minimal-codebase principle: badge inflation creates a permanent registry-shape commitment for one bonus achievement. Defer until the gameplay loop shows traction. Mirrors DEC-M8-NOVELTY-010's "tiered Pioneer deferred" rationale. |
| **DEC-M9-NO-WORKSPACE-EDIT-001** | `core/workspace.py` is BYTEWISE UNCHANGED in M-9. M-9 reads through the existing `get_stix_objects` / `_engine` (AnalystNote session) / `get_module_runs` surfaces. No new workspace method is added. | F59 sole-authority invariant. Adding even a one-line read helper (`get_analyst_notes`) would invite a future writer-side counterpart and re-open the F59 windowing question. The duplicated 6-line query in `tools.py::_get_analyst_notes` is the existing pattern; M-9 follows it. |
| **DEC-M9-NO-EVENT-001** | M-9 emits NO new ScoreEvent. `_DOSSIER_ACTIONS` stays at 4-tuple. Export/import/compare are infrastructure operations, not scored events. The "first export" achievement question is folded into DEC-M9-NO-NEW-BADGE-001. | F63 / F64 invariants: each ScoreEvent corresponds to an in-game analytic action. Bundle export is a meta-operation (manifest the analysis), not analysis itself. |
| **DEC-M9-COMBINED-SLICE-001** | All four sub-capabilities (export, import, comparison, library) ship in ONE feature branch with ONE merge commit. They are not split. | The four halves share the dossier-vocabulary authority and the `dossier/__init__.py` extension surface. Splitting would create a transient state where `compare_dossiers` exists without an `import_dossier` consumer, or library writes exist without read-side comparison. Mirrors M-7's three-sub-slice and M-8's two-sub-slice co-ship discipline. |
| **DEC-M9-CHAT-METACMD-001** | The chat meta-command extension reuses the existing `dossier` meta-command surface in `agent/chat.py` (currently `dossier` / `show dossier`). New subcommands: `dossier export [<path>|--publish]` and `dossier compare <actor_id_or_path>`. cmd2 mirrors the same surface in `core/console.py::do_dossier` (NEW handler — currently `do_dossier` does not exist; the `dossier` keyword is handled inline by `chat.py`). | One meta-command keyword in both surfaces; subcommand router style mirrors `do_report` from M-8. Single-authority for the keyword. |

## 6. Evaluation Contract (executable)

### 6.1 Required tests (must all pass at landing)

NEW test files:
- `tests/test_dossier_export.py` (~22 tests): bundle structure (type=bundle, spec_version=2.1, objects nonempty); threat-actor SDO presence + alias derivation from Identity slot; resource_level from Capability slot; roles from Motivation slot; `x_ap_predictions` shape one-to-one with `PersistedPrediction`; `x_ap_analyst_notes` shape; per-bundle metadata (`x_ap_version` / `x_ap_exported_at` / `x_ap_workspace_id` / `x_ap_actor_identifier` / `x_ap_dossier_schema_version`); `stix2.parse(bundle_json, allow_custom=True)` round-trips; `actor_identifier=None` defaults to `workspace_mgr.active`; invalid `actor_identifier` (`"../../etc/passwd"`, empty, `"a"*200`) raises `ValueError`; `core/workspace.py` not touched (git diff guard); F59 invariant — `x_ap_*` on SCOs unchanged across export.
- `tests/test_dossier_import.py` (~14 tests): round-trip from export produces `ImportedDossier` with identical slot statuses; `predictions` list equals the source; `analyst_notes` list equals the source; malformed JSON raises `ValueError`; bundle missing `type` raises `ValueError`; bundle missing threat-actor SDO raises `ValueError`; bundle with `x_ap_dossier_schema_version` != 1 raises `RuntimeError` (loud version mismatch per DEC-M4-PERSIST-003 pattern); read-only assertion — `WorkspaceManager` SQLite mtime unchanged across import.
- `tests/test_dossier_comparison.py` (~16 tests): self-compare → all `slot_diff` are `(x, x)` and `completion_local == completion_remote`; one-slot-flip diff (`filled` vs `empty`) appears in `slot_diff`; `unique_to_local` non-empty when local has a slot the remote lacks (and vice versa); `prediction_validation_ratio_*` math correct for {0/0, 0/N, N/N, mixed}; weighted completion math under DEC-M9-COMPLETION-001 (filled/partial/empty/deferred = 1.0/0.5/0.0/0.0; weighted by `SLOT_WEIGHTS`); `summary_line` is plain ASCII (no Rich markup chars); function is pure (no env reads, no I/O).
- `tests/test_dossier_library.py` (~12 tests): default root is `~/.ap/dossier_library/`; `AP_DOSSIER_LIBRARY` override honored; `publish_to_library` without `AP_DOSSIER_PUBLISH=on` raises `RuntimeError`; with `AP_DOSSIER_PUBLISH=on` writes the file and returns the path; dir permission `0o700` after first publish; pre-existing 0o755 dir is chmod'd to 0o700; `list_library()` lists files; `load_from_library` reads the file; invalid `actor_identifier` rejected at publish AND load; `AP_DOSSIER_PUBLISH=off` (or any non-"on" value) rejected.
- `tests/test_dossier_m9_tools.py` (~10 tests): `export_dossier` tool registered; `compare_dossier` tool registered; `len(create_tools(ctx)) == 30`; OpenAI function spec for each tool validates; calling `_execute_export_dossier` returns bundle JSON; calling `_execute_compare_dossier` returns plain-ASCII report; F64 `_DOSSIER_ACTIONS` unchanged (still 4-tuple); `core/workspace.py` git diff empty (architectural disconnection assert).

EXTENDED test files:
- `tests/test_agent_tools.py` — three `len(...) == 28` sites updated to `== 30` (lines 183, 287, 1422 per current file); add two name-membership asserts (`export_dossier` AND `compare_dossier` in the tool name set); the M-8 `_DOSSIER_ACTIONS` 4-tuple assertion is REUSED unchanged.
- `tests/test_dossier_state.py` (if present) — add a regression that proves `save_dossier_state` is NOT called from any M-9 code path (architectural disconnection).

Full pytest suite (`pytest tests/ -q`) green: target ≥ (M-8 baseline + C-4 closure baseline) + ~74 new M-9 tests.

### 6.2 Required evidence (paste into reviewer's verdict)

- Full `pytest tests/ -q` output green.
- `git diff main -- src/adversary_pursuit/core/workspace.py` empty.
- `git diff main -- src/adversary_pursuit/models/database.py` empty.
- `git diff main -- src/adversary_pursuit/models/stix.py` empty.
- `git diff main -- src/adversary_pursuit/core/event_bus.py src/adversary_pursuit/core/pivot_policy.py src/adversary_pursuit/core/dossier_pivot.py src/adversary_pursuit/core/streak.py src/adversary_pursuit/core/dossier_report.py src/adversary_pursuit/core/config.py` empty.
- `git diff main -- src/adversary_pursuit/dossier/state.py src/adversary_pursuit/dossier/predictions.py src/adversary_pursuit/dossier/scoring.py src/adversary_pursuit/dossier/slot_inference.py src/adversary_pursuit/dossier/panel.py src/adversary_pursuit/dossier/slots.py src/adversary_pursuit/dossier/novelty.py` empty.
- `git diff main -- src/adversary_pursuit/gamification/` empty.
- `git diff main -- src/adversary_pursuit/agent/runner.py` empty.
- `git diff main -- pyproject.toml` empty (no new runtime dependency — `stix2` already present at `>=3.0`).
- `python -c "from pathlib import Path; from adversary_pursuit.agent.tools import ToolContext, create_tools; ctx=ToolContext(config_dir=Path('/tmp/m9-test-cfg'),workspace_dir=Path('/tmp/m9-test-ws')); names=sorted(t['function']['name'] for t in create_tools(ctx)); print(len(names)); print('export_dossier' in names, 'compare_dossier' in names)"` → prints `30`, `True True`.
- Demo capture at `tmp/evidence-m9-crowdsourced-comparison/` for every §4 Stage A-F.
- `stat -f %Lp ~/.ap/dossier_library/` (or test-temp dir) → `700` after first publish.

### 6.3 Required real-path checks

- `ap chat` then `mode default` then `dossier export` — response is a valid STIX 2.1 bundle JSON string parseable by `stix2.parse(..., allow_custom=True)`.
- `ap chat` then `dossier export --publish` with `AP_DOSSIER_PUBLISH=on` — response is the library file path; `cat <path>` is parseable.
- `ap chat` then `dossier compare <other_actor>` — response shows slot-by-slot diff lines and `completion_local` / `completion_remote` numerics.
- `grep -c "LLMPersonaProfile(" src/adversary_pursuit/gamification/modes.py` → 6 (unchanged from C-4).
- `grep -rn "_DOSSIER_ACTIONS" src/adversary_pursuit/agent/tools.py | head` shows the 4-tuple unchanged.
- `python -c "from adversary_pursuit.dossier import ImportedDossier, ComparisonReport, export_dossier, import_dossier, compare_dossiers, publish_to_library, list_library, load_from_library; print('OK')"` → prints `OK`.

### 6.4 Required authority invariants

- **F59** — `core/workspace.py` BYTEWISE UNCHANGED; `store_stix_objects` remains the sole `x_ap_*` authority for SCOs. M-9's `x_ap_*` custom props live on the synthesized threat-actor SDO, which is in-process and never round-tripped through `store_stix_objects` (architectural disconnection by named test in `test_dossier_export.py`).
- **F60** — `core/pivot_policy.py` / `event_bus.py` / `dossier_pivot.py` BYTEWISE UNCHANGED.
- **F62** — `core/streak.py` + `gamification/modes.py` BYTEWISE UNCHANGED.
- **F63** — `gamification/celebrations.py` + `gamification/scoring.py` BYTEWISE UNCHANGED; no new ScoreEvent action.
- **F64** — `_DOSSIER_ACTIONS` 4-tuple unchanged; the two new LLM tool result strings carry NO score-event narration and NO Rich markup (test in `test_dossier_m9_tools.py`).
- **DEC-M1-SLOTS-WEIGHT-AUTHORITY-001** — `SLOT_WEIGHTS` consumed read-only by `compare_dossiers` (DEC-M9-COMPLETION-001).
- **DEC-M4-PERSIST-001..003** — `load_dossier_state` / `load_predictions_log` consumed read-only.
- **DEC-M5-NOTE-001..003** — `AnalystNote` consumed via the existing `tools.py::_get_analyst_notes` query pattern (no new workspace method).
- **DEC-M8-NOVELTY-001..010** — `~/.ap/dossier_novelty.sqlite` NOT TOUCHED by M-9 (parallel domain). `dossier/novelty.py` BYTEWISE UNCHANGED.
- **Sacred Practice 12** — single authority per question: export = `dossier/export.py::export_dossier`; import = `dossier/import_.py::import_dossier`; comparison = `dossier/comparison.py::compare_dossiers`; library location = `dossier/export.py::library_root`; library opt-in = `dossier/export.py::library_publish_enabled`. No parallel implementation, no fallback.
- **v1 Non-Goal "Federation between AP instances"** continues to bind: M-9 ships ZERO network I/O, ZERO upload, ZERO sync; the library is plain local files.
- **Sacred Practice 5** — loud failure: malformed bundles raise `ValueError`; schema-version mismatch raises `RuntimeError`; opt-in violation raises `RuntimeError`; invalid `actor_identifier` raises `ValueError`. No silent skip path.

### 6.5 Required integration points

- NEW `src/adversary_pursuit/dossier/export.py`.
- NEW `src/adversary_pursuit/dossier/import_.py`.
- NEW `src/adversary_pursuit/dossier/comparison.py`.
- EXTEND `src/adversary_pursuit/dossier/__init__.py` (`__all__` + imports).
- EXTEND `src/adversary_pursuit/agent/tools.py`:
  - Two NEW `_execute_*` functions (`_execute_export_dossier`, `_execute_compare_dossier`).
  - Two NEW dict entries in `create_tools(ctx)` (export_dossier + compare_dossier).
  - NO change to `_DOSSIER_ACTIONS`.
- EXTEND `src/adversary_pursuit/agent/chat.py`: `dossier export` + `dossier compare` subcommand router in the existing `dossier` meta-command handler.
- EXTEND `src/adversary_pursuit/core/console.py`: NEW `do_dossier` cmd2 command with `export` / `compare` / `show` subcommands (the existing inline `dossier` chat-handler stays; cmd2 gets a peer surface).
- EXTEND `tests/test_agent_tools.py` (three count-update sites, two membership-asserts).
- NEW `tests/test_dossier_export.py`, `tests/test_dossier_import.py`, `tests/test_dossier_comparison.py`, `tests/test_dossier_library.py`, `tests/test_dossier_m9_tools.py`.
- NEW `tmp/evidence-m9-crowdsourced-comparison/` (Stage A-F capture).
- EDIT `MASTER_PLAN.md`: append Phase 17N (M-9); flip Phase 17M closeout with C-4 SHAs (merge `9a6a550` / impl `0d1f227`); add Plan Status table row; re-point Active Phase Pointer; update Aggregate paragraph.

### 6.6 Forbidden shortcuts

- DO NOT edit `core/workspace.py` — F59 sole `x_ap_*` authority for SCOs.
- DO NOT edit `models/database.py` — no new schema; library is plain JSON files.
- DO NOT edit `models/stix.py` — M-9 reuses python-stix2 directly (DEC-59-STIX-PROVENANCE-005 pattern).
- DO NOT edit any existing `dossier/*.py` EXCEPT `__init__.py` (exports only).
- DO NOT edit `core/dossier_report.py` — M-9 is structured-data export (JSON bundle), not narrative report. Adding bundle export to `dossier_report.py` would dual-authorize the report renderer with the bundle exporter.
- DO NOT widen `_DOSSIER_ACTIONS` — no new ScoreEvent.
- DO NOT add a new badge — DEC-M9-NO-NEW-BADGE-001.
- DO NOT make `import_dossier` write to the workspace — DEC-M9-IMPORT-READONLY-001.
- DO NOT publish to the library without the opt-in env var — DEC-M9-LIBRARY-OPTIN-001.
- DO NOT add a network/upload path — v1 Non-Goal "Federation" continues to bind.
- DO NOT redact IOCs in published bundles — DEC-M9-PRIVACY-001 explicitly defers redaction; surface the privacy contract at the consent boundary.
- DO NOT call `_synthesize_threat_actor_sdo` via `store_stix_objects` — the threat-actor SDO is in-process only; persisting it through workspace storage would reactivate F59 stripping and corrupt the bundle round-trip.
- DO NOT introduce `stix2` as a new dependency — `stix2>=3.0` is already in `pyproject.toml`.
- DO NOT add a `dossier_library` config field in `core/config.py` — env-var-only knob (mirrors M-8 / `AP_NO_NOVELTY`).
- DO NOT pre-stage `MASTER_PLAN.md` amendment in a separate commit — AP #74 orphan-prevention: amend MASTER_PLAN.md in the SAME implementer commit as source.
- DO NOT amend any existing test file outside the named extension sites in §6.5.
- DO NOT cache imported bundles to disk — the importer is in-memory only.
- DO NOT auto-derive `actor_identifier` from an `intrusion-set` SDO — DEC-M9-ACTOR-ID-001.

### 6.7 Rollback boundary

Single-commit `git revert <impl-sha>` restores the C-4 closure state (merge `9a6a550`) byte-for-byte for production source. Revert removes `dossier/export.py`, `dossier/import_.py`, `dossier/comparison.py`, the `__init__.py` exports for those names, the two LLM tool definitions + dispatch entries, the chat meta-command extension, the cmd2 `do_dossier` handler, the five new test files, the `test_agent_tools.py` count updates, and the M-9 evidence directory. Library files at `~/.ap/dossier_library/<actor_identifier>.json` are OUTSIDE the repo and persist on disk through revert (manual cleanup if desired). Tool count after revert: 28 (C-4 baseline restored). No workspace schema change, no database migration, no event-bus subscriber, no event-action.

### 6.8 Ready-for-Guardian definition

All required_tests green; full suite green at ≥ (C-4 baseline + ~74 new M-9 tests); every git-diff `MUST be empty` entry pastes empty; every Stage A-F demo capture exists in `tmp/evidence-m9-crowdsourced-comparison/`; `len(create_tools(ctx)) == 30` audited; `_DOSSIER_ACTIONS` still 4-tuple; `stat` on the library dir shows `700`; **Phase 17N appended to MASTER_PLAN.md AND committed in the same commit as source by the IMPLEMENTER** (AP #74 orphan-prevention); Phase 17M status flipped in-progress → completed with C-4 SHAs (merge `9a6a550` / impl `0d1f227`); Plan Status table row for W-68-M9-CROWDSOURCED-DOSSIERS added; Active Phase Pointer tail-line re-pointed from `W-30-C4-COLUMBO` to `W-68-M9-CROWDSOURCED-DOSSIERS`; implementer commit message follows `feat(dossier-m9):` prefix and references `#68` + the DEC range `DEC-M9-ACTOR-ID-001` / `DEC-M9-STIX-MAPPING-001..002` / `DEC-M9-COMPLETION-001` / `DEC-M9-PRED-RATIO-001` / `DEC-M9-PRIVACY-001` / `DEC-M9-IMPORT-READONLY-001` / `DEC-M9-LIBRARY-LOCATION-001` / `DEC-M9-LIBRARY-OPTIN-001` / `DEC-M9-CONFLICT-001` / `DEC-M9-TOOL-EXPORT-001` / `DEC-M9-TOOL-COMPARE-001` / `DEC-M9-TOOLCOUNT-001` / `DEC-M9-NO-NEW-BADGE-001` / `DEC-M9-NO-WORKSPACE-EDIT-001` / `DEC-M9-NO-EVENT-001` / `DEC-M9-COMBINED-SLICE-001` / `DEC-M9-CHAT-METACMD-001`.

## 7. Scope Manifest

See `tmp/m9-scope.json` (canonical CLI keys `allowed_paths` / `required_paths` / `forbidden_paths` / `authority_domains`). Summary:

### Allowed (the implementer MAY touch these)

- `src/adversary_pursuit/dossier/export.py` (NEW)
- `src/adversary_pursuit/dossier/import_.py` (NEW)
- `src/adversary_pursuit/dossier/comparison.py` (NEW)
- `src/adversary_pursuit/dossier/__init__.py` (extend `__all__` + imports)
- `src/adversary_pursuit/agent/tools.py` (extend: 2 new `_execute_*` + 2 new dict entries in `create_tools`; NO `_DOSSIER_ACTIONS` change)
- `src/adversary_pursuit/agent/chat.py` (extend: `dossier export` + `dossier compare` subcommand router)
- `src/adversary_pursuit/core/console.py` (extend: NEW `do_dossier` cmd2 command)
- `tests/test_dossier_export.py` (NEW)
- `tests/test_dossier_import.py` (NEW)
- `tests/test_dossier_comparison.py` (NEW)
- `tests/test_dossier_library.py` (NEW)
- `tests/test_dossier_m9_tools.py` (NEW)
- `tests/test_agent_tools.py` (extend: 3 count-sites 28 → 30; 2 membership asserts)
- `tmp/evidence-m9-crowdsourced-comparison/` (NEW directory)
- `tmp/m9-scope.json` (planner artifact; unchanged by implementer)
- `tmp/m9-evaluation.json` (planner artifact; unchanged by implementer)
- `MASTER_PLAN.md` (append Phase 17N + flip Phase 17M closeout + Plan Status row + Active Phase Pointer)
- `.claude/plans/dossier-m9-crowdsourced-comparison.md` (this document)

### Required (the implementer MUST touch these — slice is incomplete if any is unmodified at landing)

- All NEW files above
- `src/adversary_pursuit/dossier/__init__.py`
- `src/adversary_pursuit/agent/tools.py`
- `src/adversary_pursuit/agent/chat.py`
- `src/adversary_pursuit/core/console.py`
- `tests/test_agent_tools.py`
- `MASTER_PLAN.md`

### Forbidden (preserved authorities)

- `src/adversary_pursuit/core/workspace.py` (F59 BYTEWISE UNCHANGED)
- `src/adversary_pursuit/models/database.py` (no schema change)
- `src/adversary_pursuit/models/stix.py` (no module-side STIX helper change)
- `src/adversary_pursuit/core/event_bus.py`, `core/pivot_policy.py`, `core/dossier_pivot.py` (F60)
- `src/adversary_pursuit/core/streak.py` (F62)
- `src/adversary_pursuit/core/dossier_report.py` (M-8 sole renderer authority)
- `src/adversary_pursuit/core/report.py` (DELETED at M-8 — must not be re-introduced)
- `src/adversary_pursuit/core/config.py` (env-var-only knob)
- `src/adversary_pursuit/dossier/state.py`, `predictions.py`, `scoring.py`, `slot_inference.py`, `panel.py`, `slots.py`, `novelty.py` (M-1..M-8 byte-identical)
- `src/adversary_pursuit/gamification/` (entire package — no new badge, no event-side change)
- `src/adversary_pursuit/agent/runner.py` (DEC-C2-NINJA-002 inheritance)
- `src/adversary_pursuit/modules/`, `pyproject.toml`, `CLAUDE.md`, `AGENTS.md`, `settings.json`, `hooks/`, `runtime/`, `agents/`

### Authority domains touched (descriptive labels — NEW domains owned by M-9)

- `dossier_bundle_exporter` — `dossier/export.py::export_dossier`
- `dossier_bundle_importer` — `dossier/import_.py::import_dossier`
- `dossier_comparison` — `dossier/comparison.py::compare_dossiers`
- `dossier_library_filesystem` — `dossier/export.py::library_root` + `publish_to_library` + `list_library` + `load_from_library`
- `dossier_library_optin_env_var` — `dossier/export.py::library_publish_enabled` (consumes `AP_DOSSIER_PUBLISH`)
- `dossier_actor_identifier_validation` — `dossier/export.py::_validate_actor_identifier`
- `llm_tool_catalog` — `agent/tools.py::create_tools` (+2 tools, 28 → 30)

## 8. Out-of-Scope (deferred to later slices)

- **PII redaction in published bundles** — DEC-M9-PRIVACY-001 surfaces the privacy contract at the consent boundary; an automated redactor would invent a parallel x_ap_* authority. Promotion path: file a future single-DEC slice once user demand surfaces.
- **Network upload / federation / public registry** — v1 Non-Goal "Federation between AP instances" continues to bind. The library is local-files-only.
- **"Ingest priors" import (write-side)** — DEC-M9-IMPORT-READONLY-001 defers explicitly. A future slice may add an `ingest_dossier(bundle_json, workspace_mgr)` path; that slice owns its own conflict-resolution DEC.
- **Bundle comparison tool emitting a ScoreEvent** — DEC-M9-NO-EVENT-001 / DEC-M9-NO-NEW-BADGE-001. Promotion path: telemetry-driven follow-on.
- **`generate_dossier_report` integration with library bundles** — out of scope; the M-8 renderer reads workspace state, not bundle files.
- **GUI / cmd2 library browser** — cmd2 gets a thin `do_dossier export|compare|show` peer surface; no library-listing UI beyond `list_library()` text output.
- **Multi-actor bundle / dossier collection export** — M-9 ships one-actor-per-bundle. Multi-actor packing is a future single-DEC slice.
- **`intrusion-set` auto-derivation of `actor_identifier`** — DEC-M9-ACTOR-ID-001 defers explicitly. Promotion path: a future M-X intrusion-set inference slice.
- **OpenCTI / MISP push integration** — out of scope. v1 Non-Goal "Federation" continues to bind; bundles are interop-compatible by virtue of STIX 2.1 spec compliance (DEC-59-STIX-PROVENANCE-005), but transport is not M-9's concern.

## 9. Subsequent Workflow Cue

After M-9 lands, the dossier-roadmap pipeline carries no scheduled successor. The orchestrator's autonomous-continuation decision after M-9 lands will be either:
- `goal_complete` if the v0.4.x dossier surface is the User's named end state; OR
- `next_work_item` if a runtime-hygiene backlog item (issues #49 / #50 / #51) is unblocked and ready for canonical planner adoption; OR
- `needs_user_decision` if the User must choose between dossier-axis follow-ons (PII redaction; ingest-priors writer; multi-actor bundles) AND non-dossier directions (LLM context-budget tuning, agent-flow polish, etc.).

The M-9 plan does NOT pre-commit to one of these; it documents the decision boundary so the post-landing planner pass has explicit material to reason from.

---

End of `.claude/plans/dossier-m9-crowdsourced-comparison.md`.
