# M-8 — Cleanup, Closeout, and Novel-Method Achievement (per-slice plan)

**Status:** planner-staged 2026-06-09 by W-68-M8-CLEANUP-NOVELTY planner stage. Implementer slice `wi-68-m8-impl-01` to follow.
**Workflow:** `w-68-m8-cleanup-novelty`
**Goal:** `g-68-m8-cleanup`
**Work item to dispatch:** `wi-68-m8-impl-01`
**Drives:** Phase 17K of `MASTER_PLAN.md`. Phase 17K carries the binding decisions and slice index; this document carries full rationale, design tables, decomposition detail, and acceptance-test choreography. When the two diverge, Phase 17K wins for binding decisions; this document wins for narrative.

**Inherits from:** Phase 16 §M-8, `.claude/plans/dossier-reframe-v2-roadmap.md` §M-8, DEC-68-DOSSIER-REFRAME-008 (one-release deprecation runway for the classic shim). Phase 17B (M-1) through Phase 17J (M-7) are all prerequisites and have all landed on `main`. Worktree base: AP main at merge `55aa1fe` (M-7 merge head; impl `1127144`). M-8 closes the v0.3.x dossier roadmap.

---

## 1. Goal (single paragraph)

M-8 closes the v0.3.x dossier roadmap with two co-shipped sub-slices that share a single feature branch: **(A) classic-shim removal** — DEC-68-DOSSIER-REFRAME-008's one-release deprecation runway expires at M-8 so this slice deletes `core/report.py`, the three classic LLM tools (`start_report_interview` / `answer_report_question` / `generate_report`), the `--style {dossier,classic}` flag everywhere it appears (cmd2 `do_report`, chat `report` meta-command, `generate_dossier_report` tool parameter), the `_invoke_classic` shim in `core/dossier_report.py`, the `tests/fixtures/v1_classic_report.md` fixture, and `tests/test_classic_style_regression.py`; after M-8 there is exactly one report renderer (`core/dossier_report.py`) and exactly one report-tool entry (`generate_dossier_report`); LLM tool count 31 → 28. **(B) novel-method achievement layer** — NEW `dossier/novelty.py` pure-function module hashes `(slot_name, evidence_extractor_name, sco_type_set)` tuples at hunt time, compares against a global SQLite cache at `~/.ap/dossier_novelty.sqlite` (cross-workspace by design — a single user's analytic novelty accumulates across workspaces), emits a new `dossier_novelty_recognized` ScoreEvent at points=1 on first occurrence per workspace+hash, persists the hash in the global cache on first observation across all workspaces, widens `_DOSSIER_ACTIONS` F64 filter to 4-tuple, and unlocks one new dossier-aware badge (`badge-pioneer`, RARE, threshold 1). Cache creation is lazy and opt-out via `AP_NO_NOVELTY=1` env var (default ON; cache file does not exist until the first novel observation). No new LLM tool — novelty is structural detection at the hunt-emission site, not LLM narration. Preserves M-1..M-7 invariants by construction: `core/workspace.py` / `models/database.py` / every `dossier/*.py` EXCEPT new `novelty.py` / every existing `gamification/*.py` EXCEPT extending `dossier_badges.py` and `badges.py` enum are BYTEWISE UNCHANGED. `_DOSSIER_ACTIONS` widening (3-tuple → 4-tuple) is the F64-compliant pattern M-5 already established (DEC-M5-FALSIFY-005 widened 2-tuple → 3-tuple).

**Out-of-scope (explicit, deferred):**

- **No tiered novelty badges (3/10/25).** A single unlock badge at threshold 1 (`badge-pioneer`, RARE) per minimal-codebase principle. Tiered escalation can land as a future slice when actual play-pattern data shows users care.
- **No `--style` deprecation emitter.** M-7's release-cycle window WAS the deprecation. M-8 deletes the flag and the classic path entirely. No warning, no fallback, no migration message. cmd2 `do_report` and chat `report` meta-command no longer parse `--style` at all.
- **No novelty cache config toggle.** Opt-out is via the `AP_NO_NOVELTY` env var only. No `core/config.py` field. No GUI/cmd2 toggle. Minimal-codebase principle: today there is no user case to tune the novelty cache from config; promotion can land as a future slice if a use case surfaces.
- **No `core/workspace.py` modification.** BYTEWISE UNCHANGED (matches M-5/M-6/M-7 discipline). Novelty events are persisted via the existing `workspace_mgr.store_score_events([...])` API — the same call site M-3/M-4/M-5 score events flow through.
- **No `models/database.py` modification.** DEC-DB-002 preserved (no schema migration). The novelty cache is a NEW separate global SQLite database file at `~/.ap/dossier_novelty.sqlite`, NOT a new table in the workspace SQLite. The two databases own distinct domains (workspace = per-workspace facts; novelty cache = cross-workspace analytic-method registry).
- **No `dossier/scoring.py`, `dossier/predictions.py`, `dossier/state.py`, `dossier/slot_inference.py`, `dossier/panel.py`, `dossier/slots.py` modification.** M-8 is read-only on the existing dossier surfaces. The new `novelty.py` module imports from `dossier/slots.py` and `dossier/slot_inference.py` but does not modify them.
- **No `gamification/celebrations.py` modification.** F63 invariant. ASCII-art celebrations for the `dossier_novelty_recognized` event flow through the existing `CelebrationEngine.celebrate(total)` path unchanged. No new milestone, no new ASCII art.
- **No `gamification/dossier_celebrations.py` modification.** M-7 narration policy preserved verbatim. The novelty event is excluded from narration by construction (its action is not in `("dossier_slot_filled", "dossier_prediction_validated")` per DEC-M7-CELEB-005 eligibility check) — no narration code change required.
- **No new LLM tool.** Tool count change is REDUCTION: 31 → 28 (removing the three classic tools, no additions). The `generate_dossier_report` tool's `style` parameter is REMOVED — the tool becomes parameterless (the dossier renderer is the only path).
- **No new ScoreEvent subtype beyond `dossier_novelty_recognized`.** Falsified-prediction +0 canon (DEC-M4-PRED-006) preserved.
- **No retroactive novelty scan.** Novelty detection runs only on live hunt-emission events. Pre-M-8 score events stored in workspaces are not scanned for novelty (mirrors M-7's "no catch-up narration" discipline + F63 quiet-start migration — DEC-63-MIGRATION-001).
- **No cross-user novelty sharing.** The cache at `~/.ap/dossier_novelty.sqlite` is the local user's file. Federation/sharing remains a v1 Non-Goal. M-9 (Crowdsourced Dossier Comparison) is the named future surface for cross-user comparison if scheduled.
- **No `core/console.py` modification beyond `--style` parser removal.** cmd2 `do_report` parser is simplified by deleting the `--style` branch; `_report_show` loses its `style` parameter; `_report_generate` loses its `style` parameter and the `classic` branch — the function calls `generate_dossier_report` unconditionally.
- **No `agent/chat.py` modification beyond `--style` parser removal.** chat `report` meta-command parser is simplified by deleting the `--style` branch; the `classic` interview-table render path is deleted (the `report` bare command renders the dossier report unconditionally).

---

## 2. Architecture

### 2.1 Layering authority — two sub-slices, one NEW module, deletions everywhere else

```
+--------------------------------------------------------------------------+
|  Sub-slice A: Classic-shim REMOVAL (DEC-68-DOSSIER-REFRAME-008 closeout) |
|                                                                          |
|  DELETE: src/adversary_pursuit/core/report.py                            |
|          (entire v1 ReportGenerator + INTERVIEW_QUESTIONS module)        |
|                                                                          |
|  DELETE: tests/fixtures/v1_classic_report.md                             |
|  DELETE: tests/test_classic_style_regression.py                          |
|                                                                          |
|  EDIT (REMOVE classic shim wiring):                                      |
|    src/adversary_pursuit/core/dossier_report.py                          |
|      - remove `_invoke_classic` function (lines 178-206)                 |
|      - remove ReportGenerator type-hint imports                          |
|                                                                          |
|    src/adversary_pursuit/agent/tools.py                                  |
|      - remove `start_report_interview` tool entry (create_tools list)    |
|      - remove `answer_report_question` tool entry                        |
|      - remove `generate_report` tool entry                               |
|      - remove `style` parameter from `generate_dossier_report` tool     |
|      - remove `_execute_start_report_interview` function                |
|      - remove `_execute_answer_report_question` function                |
|      - remove `_execute_generate_report` function                       |
|      - remove `_execute_generate_dossier_report`'s `style` parameter    |
|        and the `classic` branch (function body becomes ~3 lines)        |
|      - remove `from adversary_pursuit.core.report import ReportGenerator`|
|      - remove tool dispatch rows for the three deleted tools           |
|      - update `ToolContext.report_generator` field: REMOVE              |
|                                                                          |
|    src/adversary_pursuit/core/console.py                                 |
|      - remove `--style` parser in `do_report` (lines 1053-1075)         |
|      - remove `style` parameter from `_report_generate` / `_report_show`|
|      - remove the `if style == "classic":` branch in `_report_generate` |
|      - remove the `_get_report_generator` method                       |
|      - remove `_report_interview` method (no longer reachable)         |
|      - remove `self._report_generator` attribute initialisation        |
|      - remove `from adversary_pursuit.core.report import ReportGenerator`|
|      - simplify `do_report` to just `[generate|show]` subcommands      |
|                                                                          |
|    src/adversary_pursuit/agent/chat.py                                   |
|      - remove `--style` parser in `report` meta-command                 |
|      - remove `report_style` variable + filtered_rest_tokens loop      |
|      - remove the `classic` branch in `report generate` dispatch       |
|      - remove the `report answer <idx>` sub-handler (interview-only)   |
|      - remove the classic-fallback render path in bare `report`        |
|      - remove `_execute_answer_report_question` import                 |
|      - remove `_execute_generate_report` import                        |
|      - remove `_execute_start_report_interview` import                 |
|      - remove `self._report_generator` attribute                       |
|      - simplify so `report` and `report generate` both render dossier  |
|                                                                          |
|    src/adversary_pursuit/agent/tools.py::ToolContext                    |
|      - remove `report_generator: ReportGenerator | None = None` field  |
|                                                                          |
|  Result: tool count 31 -> 28 (removed: start_report_interview,         |
|          answer_report_question, generate_report). One report tool     |
|          remains: generate_dossier_report (parameterless).             |
|                                                                          |
|  UPDATE test files:                                                     |
|    tests/test_dossier_report.py        — remove any --style classic    |
|    tests/test_agent_tools.py           — update tool-count to 28       |
|                                          remove tests targeting the     |
|                                          three deleted tools / shim     |
|    tests/test_report.py                — DELETE (entire v1 path test)  |
|    tests/test_chat_report_metacommand.py — remove --style classic       |
|                                          tests; keep dossier-default    |
|                                          tests as-is                    |
|    tests/test_dossier_celebrations.py  — no change (narration policy    |
|                                          untouched; novelty event is    |
|                                          ineligible by construction)    |
|    tests/test_dossier_badges.py        — extend with badge-pioneer     |
|                                          fires-once test                |
|    tests/test_badges.py                — extend BadgeMetric/list size   |
|                                          assertions to +1 entry        |
|                                                                          |
+--------------------------------------------------------------------------+
+--------------------------------------------------------------------------+
|  Sub-slice B: Novel-method achievement (issue #68 bonus-space ask)      |
|                                                                          |
|  NEW MODULE: src/adversary_pursuit/dossier/novelty.py                    |
|                                                                          |
|    HASH AUTHORITY (DEC-M8-NOVELTY-002):                                  |
|      def compute_novelty_hash(                                           |
|          slot: DossierSlotName,                                          |
|          extractor_name: str,                                            |
|          sco_types: frozenset[str],                                      |
|      ) -> str:                                                           |
|          # 64-char SHA-256 hex of                                        |
|          # f"{slot.value}|{extractor_name}|{','.join(sorted(sco_types))}"|
|                                                                          |
|    CACHE AUTHORITY (DEC-M8-NOVELTY-003):                                 |
|      NoveltyCache class (~80 lines):                                     |
|        __init__(path: Path | None = None) — defaults to                  |
|                                            ~/.ap/dossier_novelty.sqlite  |
|        is_known(hash: str) -> bool                                       |
|        record(hash: str, slot: str, extractor: str,                     |
|               ordering_sig: str) -> None                                 |
|        close() -> None                                                   |
|        OPENS lazily — sqlite3.connect with check_same_thread=False;     |
|        CREATES schema on first write:                                    |
|          CREATE TABLE IF NOT EXISTS novelty_hashes (                    |
|            hash TEXT PRIMARY KEY,                                       |
|            slot TEXT NOT NULL,                                          |
|            extractor TEXT NOT NULL,                                     |
|            ordering_sig TEXT NOT NULL,                                  |
|            first_seen_at TEXT NOT NULL,                                 |
|            workspace_count INTEGER NOT NULL DEFAULT 1                   |
|          )                                                              |
|        WRITES are PRIMARY KEY-deduped (INSERT OR IGNORE);               |
|        no UPDATE path in M-8 (workspace_count stays at 1 until a       |
|        future slice promotes a multi-workspace counter).                |
|                                                                          |
|    DETECTOR AUTHORITY (DEC-M8-NOVELTY-005):                             |
|      def detect_novelty(                                                |
|          slot: DossierSlotName,                                          |
|          extractor_name: str,                                            |
|          sco_types: Iterable[str],                                       |
|          cache: NoveltyCache,                                            |
|      ) -> bool:                                                          |
|          # Returns True iff:                                            |
|          #   - hash not present in cache, AND                           |
|          #   - AP_NO_NOVELTY env var is unset / falsy                   |
|          # Side effect on True: cache.record(hash, ...)                 |
|          # Side effect on False (already known): None                   |
|                                                                          |
|    EVENT EMITTER:                                                       |
|      def emit_dossier_novelty_recognized_event(                         |
|          slot: DossierSlotName,                                          |
|          extractor_name: str,                                            |
|          sco_types: frozenset[str],                                      |
|      ) -> dict:                                                          |
|          # Returns score event dict ready for                           |
|          # workspace_mgr.store_score_events()                           |
|          # action: "dossier_novelty_recognized"                         |
|          # points: 1   (DEC-M8-NOVELTY-006)                             |
|          # indicator: slot.value                                        |
|          # rule_description: f"Novel slot-fill method:                  |
|          #     {slot_display} via {extractor_name}"                     |
|                                                                          |
|    OPT-OUT (DEC-M8-NOVELTY-008):                                        |
|      def novelty_enabled() -> bool:                                     |
|          return not os.environ.get("AP_NO_NOVELTY")                     |
|                                                                          |
|  EXTEND src/adversary_pursuit/agent/tools.py::_execute_run_module:      |
|    After the existing dossier_slot_filled / prediction events block    |
|    and BEFORE the narration loop, iterate the dossier_events that      |
|    fired this hunt; for each event whose extractor + sco_types tuple   |
|    is novel (per NoveltyCache), call                                   |
|    emit_dossier_novelty_recognized_event() and APPEND to               |
|    `all_dossier_events`. Persist together via the existing             |
|    store_score_events() call (DEC-M8-NOVELTY-007).                     |
|                                                                          |
|  EXTEND src/adversary_pursuit/agent/tools.py::_DOSSIER_ACTIONS:         |
|    Widen the frozenset from 3-tuple to 4-tuple (F64 invariant):        |
|      frozenset({                                                        |
|        "dossier_slot_filled",                                           |
|        "dossier_prediction_validated",                                  |
|        "dossier_prediction_falsified",                                  |
|        "dossier_novelty_recognized",   <-- M-8 NEW                     |
|      })                                                                 |
|    (DEC-M8-NOVELTY-009 — same pattern M-5 used for                     |
|     dossier_prediction_falsified per DEC-M5-FALSIFY-005.)              |
|                                                                          |
|  EXTEND src/adversary_pursuit/gamification/dossier_badges.py:           |
|    Append one new entry to DOSSIER_BADGES:                              |
|      Badge(                                                             |
|        id="badge-pioneer",                                              |
|        name="Pioneer",                                                  |
|        description=("Discover a novel slot-fill method — a            |
|                     (slot, evidence-extractor, SCO-type-set) tuple    |
|                     no prior hunt has produced."),                      |
|        rarity=BadgeRarity.RARE,                                         |
|        metric=BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED,                   |
|        threshold=1,                                                     |
|      )                                                                  |
|    Extend build_dossier_stats() to read the                             |
|    dossier_novelty_recognized count from workspace_mgr.get_recent_scores|
|    and emit `dossier_novelty_recognized` stat key (DEC-M8-NOVELTY-010).|
|                                                                          |
|  EXTEND src/adversary_pursuit/gamification/badges.py::BadgeMetric:      |
|    Add one new enum member:                                             |
|      DOSSIER_NOVELTY_RECOGNIZED = "dossier_novelty_recognized"          |
|    (DEC-M8-NOVELTY-010, mirrors M-7's BadgeMetric extension pattern)    |
|                                                                          |
|  NEW test file: tests/test_dossier_novelty.py                           |
|    Hash determinism + cache round-trip + first-occurrence detection +   |
|    second-occurrence not-novel + cross-workspace cache survival +       |
|    AP_NO_NOVELTY opt-out + lazy cache file creation + sqlite3 file path|
|    + emit event shape + F64 markup-free rule_description.               |
|                                                                          |
+--------------------------------------------------------------------------+
```

### 2.2 Storage authority decision — global SQLite at `~/.ap/dossier_novelty.sqlite` (DEC-M8-NOVELTY-001)

The roadmap (§M-8) names `~/.ap/dossier_novelty.sqlite` as the cache location. The planner stage evaluated three options:

- **Option A — separate global SQLite at `~/.ap/dossier_novelty.sqlite`**. Pro: matches roadmap intent; novelty accumulates across workspaces (a user pursuing different actors gets credit for genuinely new method combinations); the cross-workspace surface is the entire point of the feature. Con: second SQLite authority on disk.
- **Option B — workspace-local table in `models/database.py`**. Pro: single persistence authority. Con: novelty is workspace-scoped — every new workspace re-discovers the same "novelties"; defeats the whole point of the achievement.
- **Option C — flat JSON/SQLite file at user-config dir (XDG-resolved)**. Pro: minimal — no schema, no migration. Con: hand-rolled persistence; concurrent-write race; no `INSERT OR IGNORE` semantic.

**Choice: A.** The roadmap's intent is cross-workspace recognition; B contradicts it; C trades schema robustness for code minimalism in a feature that requires PRIMARY KEY dedup. The "second DB authority" concern is mitigated because (1) `~/.ap/` is already an established cross-workspace state directory (used by `core/config.py::_DEFAULT_CONFIG_DIR`, `core/workspace.py::_DEFAULT_WORKSPACE_DIR`, `core/pivot_policy.py` allowlist/denylist files); (2) the novelty cache owns a distinct domain (cross-workspace analytic-method registry) that does NOT overlap with workspace-local data; (3) the two databases never need joins or migrations together. Sacred Practice 12 ("Single Source of Truth") is honoured by domain separation: workspace SQLite owns workspace facts; novelty cache owns the global novelty registry; neither is the authority for the other's domain.

**Cache file path resolution** uses the same `Path.home() / ".ap" / "dossier_novelty.sqlite"` pattern `core/config.py::_DEFAULT_CONFIG_DIR` and `core/pivot_policy.py` already established. The path is overridable via constructor injection for test isolation (`NoveltyCache(path=tmp_path / "novelty.sqlite")`).

### 2.3 Hash inputs — `(slot, extractor, sco_type_set)` with SCO-type SET ordering signature (DEC-M8-NOVELTY-002)

The roadmap names the hash inputs as `(slot, evidence-extractor, ordering)` and identifies three candidate interpretations of "ordering signature":

- **Order in which slots filled within a hunt.** Permissive — many novel orderings exist; novelty fires often; achievement becomes noise.
- **Set of SCO-types that fed each slot.** Strict — same (slot, extractor) with the same SCO inputs is the SAME analytic method; novelty fires only when the inputs themselves are new. **Chosen.**
- **Multi-hunt sequence of slot fills.** Strictest — almost no two hunts replicate; novelty accumulates with session length but loses semantic meaning.

**Choice: SCO-type set per slot.** The middle option is the semantic match for "novel method" — an analyst who first fills Identity via Whois + crt.sh (SCO types: `{ipv4-addr, domain-name, x509-certificate}`) is doing a recognisably distinct analytic move from one who fills Identity via Shodan + OTX (SCO types: `{ipv4-addr, autonomous-system, threat-actor}`). The set vocabulary is the STIX SCO type vocabulary already in use throughout the codebase (`stix2.v21` types referenced in `models/database.py` and `dossier/slot_inference.py`).

**Hash function:**
```
sha256(f"{slot.value}|{extractor_name}|{','.join(sorted(sco_types))}").hexdigest()
```
The sorted join makes the hash a SET-ordering signature (input order does not matter). The `|` separator is unambiguous (cannot appear in slot values or sorted SCO-type lists). 64-char hex is the natural PRIMARY KEY type.

**Extractor name** is the function name in `dossier/slot_inference.py` that produced the slot's status (e.g., `_extract_identity`, `_extract_capability`). The detector reads the extractor name from a per-slot metadata trail attached to the `DossierState` post-snapshot. Since `slot_inference.py` is BYTEWISE UNCHANGED (M-1..M-7 invariant), the extractor name is inferred at the detection site from the (slot, transition) pair using a constant `_SLOT_EXTRACTOR_NAMES` map in `novelty.py`. Decoupling extractor identity from `slot_inference.py` keeps that file untouched and gives M-8 a clean dependency boundary.

### 2.4 Novelty score event weight — points=1 (DEC-M8-NOVELTY-006)

The dispatch context proposed 1.5 (between IOC baseline 1.0 and Infrastructure 2.0). The planner reviews this and chooses **points=1**:

- All dossier-roadmap scoring events emit `int` points (M-3 floor-cast `int(SLOT_WEIGHTS[slot])`). Introducing a float-weight event would break the integer-only ScoreEvent contract.
- Per-IOC scoring at baseline 1 is the established "small, recognisable, recurrent" tier. Novelty fits that tier: a single hunt may produce multiple novel events; small flat weight prevents score inflation as the global cache grows.
- The novelty PRESTIGE surface is the badge unlock (`badge-pioneer`, RARE), not the score points. The score event exists primarily to drive badge stats and to flow through F62/F63/F64 — its point value is secondary.

The +1 weight matches M-3's per-IOC baseline (`MODULE_RUN_SCORED` retuned to `initial=minimum=1` per DEC-M3-DOSSIER-004), so the runtime contour stays predictable.

### 2.5 One novelty badge — `badge-pioneer` at threshold 1 (DEC-M8-NOVELTY-010)

The dispatch context offered 1 vs tiered (3/10/25). The planner chooses **1 simple unlock badge**:

- Per minimal-codebase principle. A single threshold-1 badge is the smallest abstraction matching the achievement need (recognise that ANY novel method was discovered).
- The M-7 dossier badges already exercise both single-threshold (Identity First, Deception Spotter, Pioneer) and 3-threshold (Predictor, Skeptic) patterns. Adding a tiered Pioneer would duplicate the Predictor/Skeptic shape without a new analytic signal.
- Tiered escalation can land as a future slice when actual telemetry shows users care about the 25-novelty horizon. Today there is no data.

**Badge spec:**
- `id="badge-pioneer"`, `name="Pioneer"`, `rarity=BadgeRarity.RARE`, `metric=BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED`, `threshold=1`.
- Stat-key: `dossier_novelty_recognized` (integer count of `dossier_novelty_recognized` events in the workspace's score_events table).

RARE matches "directed achievement" tier — same as Identity First and Deception Spotter. LEGENDARY would over-elevate a single-event achievement; UNCOMMON would under-recognise the genuine analytic insight.

### 2.6 Opt-out — `AP_NO_NOVELTY` env var (DEC-M8-NOVELTY-008)

The codebase already uses `AP_NO_BANNER` as a disable-by-truthy pattern (`core/console.py:240`, `agent/banner.py:128`, `agent/banner.py:201`). Novelty mirrors that exactly:

- **Default behavior**: ON. Novelty detection runs on every hunt's dossier events.
- **Opt-out**: set `AP_NO_NOVELTY=1` (or any non-empty value) in the environment. `novelty_enabled()` returns False; the detection block in `_execute_run_module` skips both detection and emission.
- **Lazy cache creation**: the `~/.ap/dossier_novelty.sqlite` file is created on the first INSERT — the file does not exist for users who never hit a novel observation or who set the opt-out before their first hunt.
- **No GUI surface**: no cmd2 toggle, no chat meta-command, no config.toml field. Minimal-codebase principle; promotion path is "add when a user case surfaces."

### 2.7 Emission ordering and `_DOSSIER_ACTIONS` widening (DEC-M8-NOVELTY-007 / DEC-M8-NOVELTY-009)

The existing emission sequence in `agent/tools.py::_execute_run_module` (M-7 baseline) is:

```
1. score_results events       (per-IOC discovery)
2. dossier_slot_filled events (M-3)
3. dossier_prediction_validated events (M-4 confirmations)
4. dossier_prediction_falsified events (M-5)
5. (M-7) narration loop over all_dossier_events (F64 sidecar)
```

M-8 inserts novelty detection **between steps 4 and 5**:

```
1. score_results events
2. dossier_slot_filled events
3. dossier_prediction_validated events
4. dossier_prediction_falsified events
4b. NOVELTY DETECTION + dossier_novelty_recognized emission (M-8 NEW)
5. (M-7) narration loop over all_dossier_events
6. workspace_mgr.store_score_events(all_dossier_events) — one call
```

Narration eligibility (M-7 DEC-M7-CELEB-005) restricts narration to `dossier_slot_filled` and `dossier_prediction_validated`. The novelty event's action `dossier_novelty_recognized` is NOT in that set, so step 5 silently skips it — no narration code change required.

The F64 panel-separation filter (`_DOSSIER_ACTIONS` in `agent/tools.py`) widens from 3-tuple to 4-tuple (DEC-M8-NOVELTY-009). This is the same pattern M-5 used (DEC-M5-FALSIFY-005 widened 2→3); the test suite asserts the filter and the summary suppression behavior for the new action.

### 2.8 Tool catalog change — 31 → 28 (DEC-M8-CLEANUP-002)

The three classic tools removed:

| Removed tool | Replacement | Rationale |
|--------------|-------------|-----------|
| `start_report_interview` | `generate_dossier_report` (no interview required) | DEC-68-DOSSIER-REFRAME-008 deprecation runway expires; dossier report is the sole renderer. |
| `answer_report_question` | (none — no analyst interview path) | Interview is the v1 surface; deleted with the v1 path. |
| `generate_report` | `generate_dossier_report` (parameterless) | Same. The `style` parameter on `generate_dossier_report` is also removed (no style choice exists post-M-8). |

Final tool count: 28. The `tests/test_agent_tools.py` count assertion is updated from 31 → 28.

### 2.9 Module deletion safety — `core/report.py` is read by zero non-deleted call sites (DEC-M8-CLEANUP-003)

Pre-M-8 `core/report.py` callers:

| Caller | Status after M-8 |
|--------|------------------|
| `core/dossier_report.py::_invoke_classic` | REMOVED (function body deleted) |
| `core/console.py` (`do_report`, `_get_report_generator`, `_report_interview`, `_report_generate` classic branch) | REMOVED |
| `agent/chat.py` (`report` meta-command classic branch + `report answer` handler) | REMOVED |
| `agent/tools.py` (`_execute_start_report_interview`, `_execute_answer_report_question`, `_execute_generate_report` + the three tool definitions + the dispatch rows + `ToolContext.report_generator` field) | REMOVED |
| `tests/test_classic_style_regression.py` | DELETED (entire file) |
| `tests/test_report.py` | DELETED (entire file — v1 path tests) |
| `tests/fixtures/v1_classic_report.md` | DELETED |

The implementer audits with `grep -rn "ReportGenerator\|core.report\|core\.report\|start_report_interview\|answer_report_question\|generate_report" src/ tests/` BEFORE the deletion and AFTER the deletion. After the deletion, the only matches must be in the M-8 plan/MASTER_PLAN.md narrative — zero source references.

### 2.10 F64 invariant — narration / panel-separation preserved (DEC-M8-NOVELTY-009)

The novelty event flows through the same `_DOSSIER_ACTIONS` filter as M-3..M-5 events:

- `_DOSSIER_ACTIONS` widens to include `"dossier_novelty_recognized"` → the summary suppression test (existing M-3/M-5 tests) extends with the new action.
- Narration eligibility (`dossier_celebrations.is_high_weight_event`) returns False for `dossier_novelty_recognized` (action not in `{"dossier_slot_filled", "dossier_prediction_validated"}`). The narration loop silently skips it. No `dossier_celebrations.py` modification needed.
- The novelty event still flows through `runner.last_celebrations` indirectly: the existing ASCII-art block in `_execute_run_module` (line 555-558) computes `art = self.celebration.celebrate(total)` where `total` includes the novelty event's +1 points; the celebration ASCII appears as part of the same hunt's celebration string. No additional narration.

DEC-64-LLM-PANEL-SEPARATION-001 preserved: novelty score event text is suppressed from the LLM-facing `summary` (via `_DOSSIER_ACTIONS` widening); the celebration sidecar surfaces it through the existing ASCII pathway.

### 2.11 Cache schema and round-trip semantics (DEC-M8-NOVELTY-004)

The `~/.ap/dossier_novelty.sqlite` schema is one table:

```sql
CREATE TABLE IF NOT EXISTS novelty_hashes (
  hash TEXT PRIMARY KEY,           -- 64-char SHA-256 hex
  slot TEXT NOT NULL,              -- DossierSlotName.value
  extractor TEXT NOT NULL,         -- function name from _SLOT_EXTRACTOR_NAMES
  ordering_sig TEXT NOT NULL,      -- ','.join(sorted(sco_types))
  first_seen_at TEXT NOT NULL,     -- ISO-8601 UTC
  workspace_count INTEGER NOT NULL DEFAULT 1
);
```

Operations:

- **Lookup**: `SELECT 1 FROM novelty_hashes WHERE hash = ?` — returns row if known.
- **Insert**: `INSERT OR IGNORE INTO novelty_hashes (hash, slot, extractor, ordering_sig, first_seen_at, workspace_count) VALUES (?, ?, ?, ?, ?, 1)`.

The `workspace_count` column is reserved for a future slice that may promote it to a multi-workspace counter (currently it stays at 1 — `INSERT OR IGNORE` does not increment; future code would add an UPDATE branch). Schema is forward-compatible.

Concurrent access from multiple AP processes (one user with `ap chat` open and an `ap` cmd2 session in parallel) is safe via SQLite's default journal mode. Connections are opened with `sqlite3.connect(path, check_same_thread=False, isolation_level=None)` (autocommit) so each write commits immediately.

---

## 3. Removal targets and the explicit cleanup checklist (Sacred Practice 12)

After M-8, exactly one report renderer exists in the codebase. After M-8, exactly one report-generating LLM tool exists. After M-8, no `--style` flag exists anywhere.

| Target | Type | M-8 action | Verification |
|--------|------|-----------|--------------|
| `src/adversary_pursuit/core/report.py` | file | DELETE | `ls src/adversary_pursuit/core/report.py` returns "No such file" |
| `core/dossier_report.py::_invoke_classic` | function | DELETE | `grep -n "_invoke_classic" src/adversary_pursuit/core/dossier_report.py` empty |
| `core/console.py::_get_report_generator` | method | DELETE | `grep -n "_get_report_generator" src/adversary_pursuit/core/console.py` empty |
| `core/console.py::_report_interview` | method | DELETE | `grep -n "_report_interview" src/adversary_pursuit/core/console.py` empty |
| `core/console.py::_report_generator` (attribute) | attribute | DELETE | `grep -n "_report_generator" src/adversary_pursuit/core/console.py` empty |
| `core/console.py::do_report` `--style` parser | code block | DELETE | `grep -n "\\-\\-style" src/adversary_pursuit/core/console.py` empty |
| `core/console.py::_report_generate(style=)` | parameter | REMOVE | function signature is `_report_generate(self)` |
| `core/console.py::_report_show(style=)` | parameter | REMOVE | function signature is `_report_show(self)` |
| `agent/chat.py` `--style` parser block | code block | DELETE | `grep -n "report_style\|\\-\\-style" src/adversary_pursuit/agent/chat.py` empty |
| `agent/chat.py` classic-branch render | code block | DELETE | `grep -n "_execute_generate_report\b\|_execute_start_report_interview\|_execute_answer_report_question" src/adversary_pursuit/agent/chat.py` empty |
| `agent/chat.py` `self._report_generator` | attribute | DELETE | `grep -n "_report_generator" src/adversary_pursuit/agent/chat.py` empty |
| `agent/tools.py::start_report_interview` tool entry | tool definition | DELETE | `grep -n "start_report_interview" src/adversary_pursuit/agent/tools.py` empty |
| `agent/tools.py::answer_report_question` tool entry | tool definition | DELETE | `grep -n "answer_report_question" src/adversary_pursuit/agent/tools.py` empty |
| `agent/tools.py::generate_report` tool entry | tool definition | DELETE | `grep -n '"generate_report"' src/adversary_pursuit/agent/tools.py` empty |
| `agent/tools.py::_execute_start_report_interview` | function | DELETE | `grep -n "_execute_start_report_interview" src/adversary_pursuit/agent/tools.py` empty |
| `agent/tools.py::_execute_answer_report_question` | function | DELETE | `grep -n "_execute_answer_report_question" src/adversary_pursuit/agent/tools.py` empty |
| `agent/tools.py::_execute_generate_report` | function | DELETE | `grep -n "_execute_generate_report\b" src/adversary_pursuit/agent/tools.py` returns only the renamed dossier dispatcher |
| `agent/tools.py::_execute_generate_dossier_report::style` parameter | parameter | REMOVE | function signature is `_execute_generate_dossier_report(ctx)` |
| `agent/tools.py::ToolContext.report_generator` | dataclass field | REMOVE | `grep -n "report_generator" src/adversary_pursuit/agent/tools.py` empty |
| `generate_dossier_report` tool definition `style` parameter | tool schema | REMOVE | tool `parameters` is `{"type": "object", "properties": {}, "required": []}` |
| `agent/tools.py::from adversary_pursuit.core.report import ReportGenerator` | import | DELETE | `grep -n "from adversary_pursuit.core.report" src/adversary_pursuit/agent/tools.py` empty |
| `core/console.py::from adversary_pursuit.core.report import ReportGenerator` | import | DELETE | `grep -n "from adversary_pursuit.core.report" src/adversary_pursuit/core/console.py` empty |
| `agent/chat.py::from adversary_pursuit.core.report import ReportGenerator` | import | DELETE | `grep -n "from adversary_pursuit.core.report" src/adversary_pursuit/agent/chat.py` empty |
| `tests/fixtures/v1_classic_report.md` | file | DELETE | `ls tests/fixtures/v1_classic_report.md` returns "No such file" |
| `tests/test_classic_style_regression.py` | file | DELETE | `ls tests/test_classic_style_regression.py` returns "No such file" |
| `tests/test_report.py` | file | DELETE | `ls tests/test_report.py` returns "No such file" |
| `tests/test_chat_report_metacommand.py` (`--style classic` tests only) | tests | REMOVE | `grep -n "classic\|--style" tests/test_chat_report_metacommand.py` empty |
| `tests/test_agent_tools.py` tool-count assertions | assertion | UPDATE | `assert len(tools) == 28` (was 31) |
| `tests/test_dossier_report.py` (`--style classic` tests only) | tests | REMOVE | no test stages call classic path |

Cleanup is verified by an audit script the implementer runs after the deletion: `grep -rn "ReportGenerator\|start_report_interview\|answer_report_question\|--style\|_invoke_classic" src/ tests/` MUST return no matches in source code (matches in `.claude/plans/` or `MASTER_PLAN.md` historical narrative are allowed).

---

## 4. The load-bearing acceptance tests

### Stage A — classic-shim removal (audit + import surface)

`tests/test_m8_cleanup_audit.py` (NEW):
1. **`test_classic_report_file_deleted`** — `(Path("src/adversary_pursuit/core/report.py")).exists() is False`.
2. **`test_classic_fixture_deleted`** — `(Path("tests/fixtures/v1_classic_report.md")).exists() is False`.
3. **`test_classic_regression_test_deleted`** — `(Path("tests/test_classic_style_regression.py")).exists() is False`.
4. **`test_v1_report_test_deleted`** — `(Path("tests/test_report.py")).exists() is False`.
5. **`test_no_reportgenerator_import_in_src`** — recursive grep of `src/adversary_pursuit/` for `ReportGenerator` returns 0 matches.
6. **`test_no_style_flag_in_src`** — recursive grep for `--style` in source returns 0 matches.
7. **`test_no_classic_tool_names_in_tools_py`** — `start_report_interview`, `answer_report_question`, `generate_report` strings absent from `agent/tools.py` source.
8. **`test_tool_count_is_28`** — `create_tools(ctx)` returns 28 entries.
9. **`test_generate_dossier_report_parameterless`** — the tool's `parameters` schema has empty `properties` and empty `required`.
10. **`test_execute_generate_dossier_report_no_style_param`** — call signature is `(ctx,)` (TypeError if `style=` passed).

### Stage B — novelty hash + cache (`tests/test_dossier_novelty.py` — NEW)

1. **`test_compute_novelty_hash_deterministic`** — same inputs → same 64-char hex; reordered `sco_types` set → same hash; different extractor → different hash; different slot → different hash.
2. **`test_novelty_cache_lazy_file_creation`** — `NoveltyCache(path=tmp_path / "novelty.sqlite")` does NOT create the file; first `record(...)` call DOES create it.
3. **`test_novelty_cache_round_trip`** — `record(hash, "identity", "_extract_identity", "ipv4-addr,domain-name")` then `is_known(hash) is True`.
4. **`test_novelty_cache_dedup`** — two `record()` calls with the same hash leave one row (verified via `SELECT COUNT(*) FROM novelty_hashes`).
5. **`test_novelty_cache_schema`** — `PRAGMA table_info(novelty_hashes)` returns the 6-column shape (hash, slot, extractor, ordering_sig, first_seen_at, workspace_count).
6. **`test_detect_novelty_first_occurrence_returns_true`** — fresh cache + first detection → True + cache row written.
7. **`test_detect_novelty_second_occurrence_returns_false`** — same inputs to a populated cache → False + no second cache row.
8. **`test_detect_novelty_respects_opt_out`** — `monkeypatch.setenv("AP_NO_NOVELTY", "1")` → `detect_novelty(...)` returns False regardless of cache state; no cache write.
9. **`test_novelty_enabled_truthy_values`** — empty / unset env var → enabled; `"1"` / `"true"` / `"on"` / any non-empty value → disabled.
10. **`test_emit_dossier_novelty_recognized_event_shape`** — returned dict has keys `action="dossier_novelty_recognized"`, `points=1`, `indicator=<slot_value>`, `rule_description` plain ASCII non-empty.
11. **`test_rule_description_has_no_rich_markup`** — F64: `"["`, `"]"`, `"{"`, `"}"` not in `rule_description`.
12. **`test_cache_default_path_is_user_home`** — `NoveltyCache().path == Path.home() / ".ap" / "dossier_novelty.sqlite"` (without instantiating SQLite).

### Stage C — integration in `_execute_run_module` (`tests/test_agent_tools.py` extension)

1. **`test_run_module_emits_novelty_event_on_first_novel_slot_fill`** — set up a fresh workspace + monkeypatched `NoveltyCache` pointing at `tmp_path`; run a module that triggers a `dossier_slot_filled` event; assert one `dossier_novelty_recognized` event appears in `result["score_events"]`.
2. **`test_run_module_no_novelty_event_on_repeat_slot_fill`** — run the same module twice with state-reset between hunts; second hunt produces no `dossier_novelty_recognized` event (cache hit).
3. **`test_run_module_dossier_actions_widened_to_4_tuple`** — assert `_DOSSIER_ACTIONS` is `frozenset({"dossier_slot_filled", "dossier_prediction_validated", "dossier_prediction_falsified", "dossier_novelty_recognized"})` and the LLM-facing summary contains no novelty event text.
4. **`test_run_module_ap_no_novelty_disables_detection`** — `monkeypatch.setenv("AP_NO_NOVELTY", "1")` → no novelty events emitted even on first-time slot fill.
5. **`test_run_module_novelty_event_not_narrated`** — when narration runs, the novelty event is silently skipped (no `runner.narrate` call for it).

### Stage D — badge unlock (`tests/test_dossier_badges.py` extension)

1. **`test_badge_pioneer_fires_on_first_novelty_recognized`** — workspace stats with `dossier_novelty_recognized: 1` → `badge-pioneer` in `BadgeManager.check_all()` newly-earned set.
2. **`test_badge_pioneer_idempotent`** — second `check_all()` call (already-awarded set populated) → empty newly-earned set.
3. **`test_dossier_badges_list_has_six_entries`** — `DOSSIER_BADGES` length 6 (M-7's 5 + M-8's Pioneer).
4. **`test_default_badges_list_has_16_entries`** — `_DEFAULT_BADGES` length 16 (existing 10 + M-7's 5 + M-8's 1).
5. **`test_badge_metric_has_dossier_novelty_recognized`** — `BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED.value == "dossier_novelty_recognized"`.
6. **`test_build_dossier_stats_returns_novelty_count`** — populated workspace + 3 `dossier_novelty_recognized` score events → stats dict has `dossier_novelty_recognized: 3`.

### Stage E — manual demo evidence (`tmp/evidence-m8-cleanup-novelty/`)

Manual sanity-check trace (recorded to `tmp/evidence-m8-cleanup-novelty/`):
1. **Demo 1 — classic shim is gone**: from a fresh `ap chat` session, attempt `report --style classic generate` → response is "unknown subcommand" or equivalent (no classic path); `report generate` produces the dossier report.
2. **Demo 2 — novelty fires on a new method**: fresh workspace (`AP_NO_NOVELTY` unset, `~/.ap/dossier_novelty.sqlite` absent); run `hunt for 1.1.1.1` triggering a slot-fill via a new (slot, extractor, sco_types) tuple; observe `dossier_novelty_recognized` in score events; observe `~/.ap/dossier_novelty.sqlite` was created; observe `badge-pioneer` unlocked.
3. **Demo 3 — opt-out**: `AP_NO_NOVELTY=1 ap chat`; run the same hunt; observe NO `dossier_novelty_recognized` event; observe `~/.ap/dossier_novelty.sqlite` not modified.
4. **Demo 4 — cache file presence**: `sqlite3 ~/.ap/dossier_novelty.sqlite "SELECT * FROM novelty_hashes"` shows the recorded hash row.

Implementer captures terminal output + `git diff main --stat` to `tmp/evidence-m8-cleanup-novelty/`.

---

## 5. F64 invariance — re-stated and tested

DEC-64-LLM-PANEL-SEPARATION-001 is preserved:

- `_DOSSIER_ACTIONS` widens to 4-tuple (M-3's pattern, M-5's pattern). Test asserts `dossier_novelty_recognized` is filtered out of LLM-facing summary text.
- Novelty events flow through `runner.last_celebrations` indirectly via the existing celebration ASCII pipeline (points are summed into `total`; ASCII art is computed for the total; no new sidecar field).
- Novelty events are NEVER narrated (M-7 `is_high_weight_event` returns False for `dossier_novelty_recognized`; no `dossier_celebrations.py` modification).
- `rule_description` field is plain ASCII (no Rich markup) — test asserts.
- No new `BadgeMetric` enum member breaks DEC-BADGE-003 stats-dict contract (new member `DOSSIER_NOVELTY_RECOGNIZED = "dossier_novelty_recognized"` follows the same shape M-7 established).

---

## 6. Evaluation Contract

See per-Phase 17K section for the legal-key JSON shape; this is the narrative form.

- **required_tests:** ~25 new + extended tests across:
  - `tests/test_m8_cleanup_audit.py` (NEW, ~10 tests): Stage A coverage — every deletion + import-surface assertion.
  - `tests/test_dossier_novelty.py` (NEW, ~12 tests): Stage B coverage — hash function + cache + detector + opt-out + emitter.
  - `tests/test_agent_tools.py` (extend, ~5 tests): Stage C coverage — `_execute_run_module` integration + `_DOSSIER_ACTIONS` widening + tool-count update + parameterless `generate_dossier_report` dispatcher.
  - `tests/test_dossier_badges.py` (extend, ~6 tests): Stage D coverage — Pioneer badge + lists size + `BadgeMetric` extension + stats key.
  - `tests/test_badges.py` (extend, ~2 tests): `BadgeMetric` enum has `DOSSIER_NOVELTY_RECOGNIZED`; `_DEFAULT_BADGES` list length 16.
  - `tests/test_dossier_report.py` (extend, ~2 tests): style-flag-related tests pruned; remaining tests assert dossier report still renders identically to its M-7 byte output.
  - `tests/test_chat_report_metacommand.py` (extend, ~3 tests): bare `report` and `report generate` produce dossier report; no `report --style classic` parsing; no interview path.

  Full suite green ≥ M-7 baseline minus the deleted tests (deleted: `test_classic_style_regression.py` + `test_report.py` + any `--style classic` tests that were in `test_dossier_report.py` / `test_chat_report_metacommand.py`) + new M-8 tests.

- **required_evidence:**
  - Full `pytest -q` green.
  - `ls src/adversary_pursuit/core/report.py` → No such file.
  - `ls tests/fixtures/v1_classic_report.md` → No such file.
  - `ls tests/test_classic_style_regression.py` → No such file.
  - `ls tests/test_report.py` → No such file.
  - `grep -rn "ReportGenerator\|start_report_interview\|answer_report_question\|_invoke_classic\|\\-\\-style" src/ tests/` → no matches outside this plan / MASTER_PLAN narrative.
  - `git diff main -- src/adversary_pursuit/core/workspace.py` empty.
  - `git diff main -- src/adversary_pursuit/models/database.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/state.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/predictions.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/scoring.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/slot_inference.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/slots.py` empty.
  - `git diff main -- src/adversary_pursuit/dossier/panel.py` empty.
  - `git diff main -- src/adversary_pursuit/core/pivot_policy.py` empty.
  - `git diff main -- src/adversary_pursuit/core/event_bus.py` empty.
  - `git diff main -- src/adversary_pursuit/core/dossier_pivot.py` empty.
  - `git diff main -- src/adversary_pursuit/core/streak.py` empty.
  - `git diff main -- src/adversary_pursuit/core/config.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/scoring.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/modes.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/hints.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/challenges.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/celebrations.py` empty.
  - `git diff main -- src/adversary_pursuit/gamification/dossier_celebrations.py` empty.
  - Tool-count audit at exactly 28.
  - Demo evidence under `tmp/evidence-m8-cleanup-novelty/` showing §4 Stage A/B/C/D/E acceptance.

- **required_authority_invariants:**
  - F59 (`core/workspace.py` BYTEWISE UNCHANGED).
  - F60 (`core/pivot_policy.py` / `core/event_bus.py` / `core/dossier_pivot.py` BYTEWISE UNCHANGED).
  - F62 (`core/streak.py` BYTEWISE UNCHANGED; new `dossier_novelty_recognized` action joins the established `_DOSSIER_ACTIONS` filter without changing streak semantics).
  - F63 (`gamification/celebrations.py` BYTEWISE UNCHANGED; no new milestone; no new ASCII art).
  - F64 (`_DOSSIER_ACTIONS` widening to 4-tuple is the M-5-established pattern; DEC-64-LLM-PANEL-SEPARATION-001 preserved — novelty event suppressed from LLM-facing summary; novelty event NEVER narrated; `rule_description` is plain ASCII).
  - Sacred Practice 12 (single source of truth: novelty cache = `dossier/novelty.py::NoveltyCache`; novelty hash = `dossier/novelty.py::compute_novelty_hash`; novelty detector = `dossier/novelty.py::detect_novelty`; novelty emitter = `dossier/novelty.py::emit_dossier_novelty_recognized_event`; novelty badge = `gamification/dossier_badges.py::DOSSIER_BADGES` (extended); novelty cache lives at `~/.ap/dossier_novelty.sqlite` — distinct domain from workspace SQLite; after M-8, exactly ONE report renderer = `core/dossier_report.py`; exactly ONE report-tool entry = `generate_dossier_report`).
  - DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 (read-only consumer).
  - DEC-M3-DOSSIER-001..005 (untouched).
  - DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006 (untouched; falsification=+0 canon preserved).
  - DEC-M5-DENIAL-001..003 + DEC-M5-NOTE-001..003 + DEC-M5-FALSIFY-001..008 (untouched; `_DOSSIER_ACTIONS` widening pattern inherited).
  - DEC-M6-PIVOT-001..009 (untouched).
  - DEC-M7-REPORT-001 (style flag): RETIRED / REMOVED at M-8 (one-release deprecation runway expired).
  - DEC-M7-REPORT-002 (`core/dossier_report.py` separation): PRESERVED — file remains; `_invoke_classic` shim function deleted (its sibling target `core/report.py` is also deleted in the same commit).
  - DEC-M7-REPORT-003 (classic regression fixture): RETIRED / REMOVED at M-8.
  - DEC-M7-REPORT-004 (dossier-report section composition): PRESERVED.
  - DEC-M7-REPORT-005 (LLM tool count 30 → 31): SUPERSEDED — M-8 tool count is 31 → 28 (three classic tools removed; no additions); the `style` parameter on `generate_dossier_report` is also removed.
  - DEC-M7-CELEB-001..007 (narration policy): PRESERVED; novelty event is ineligible by construction (action filter excludes it).
  - DEC-M7-BADGE-001..007 (existing 5 dossier badges byte-identical): PRESERVED; M-8 appends one new badge (DEC-M8-NOVELTY-010).
  - DEC-68-DOSSIER-REFRAME-006 (issue #32 absorbed at M-7): PRESERVED.
  - DEC-68-DOSSIER-REFRAME-008 (one-release deprecation runway for classic shim): HONOURED — runway expires at M-8, classic shim removed in this slice.
  - DEC-BADGE-001..003 (badge stats-dict contract + metric enum pattern + stateless manager): PRESERVED.
  - DEC-CELEBRATION-001 (four-level ASCII art): PRESERVED.
  - DEC-63-MILESTONE-CATCHUP-001 (milestone catch-up): UNTOUCHED.
  - DEC-AGENT-REPORT-001..N (v1 interview tools): RETIRED / REMOVED at M-8.
  - DEC-REPORT-001..003 (v1 ReportGenerator): RETIRED / REMOVED at M-8.

- **required_integration_points:**
  - NEW `src/adversary_pursuit/dossier/novelty.py` — novelty hash + cache + detector + emitter + opt-out helper.
  - EXTEND `src/adversary_pursuit/dossier/__init__.py` — export the four public names (`compute_novelty_hash`, `NoveltyCache`, `detect_novelty`, `emit_dossier_novelty_recognized_event`).
  - EXTEND `src/adversary_pursuit/agent/tools.py` — insert novelty detection block in `_execute_run_module` between dossier-events emission and narration loop; widen `_DOSSIER_ACTIONS` to 4-tuple; remove three classic tool definitions; remove three classic dispatch rows; remove three classic `_execute_*` functions; remove `report_generator` from `ToolContext`; remove `core.report` import; remove `style` parameter from `_execute_generate_dossier_report` + the corresponding tool schema.
  - EXTEND `src/adversary_pursuit/gamification/dossier_badges.py` — append `badge-pioneer` to `DOSSIER_BADGES`; extend `build_dossier_stats` to compute `dossier_novelty_recognized` count from workspace scores.
  - EXTEND `src/adversary_pursuit/gamification/badges.py` — add `BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED` enum member.
  - EDIT `src/adversary_pursuit/core/dossier_report.py` — remove `_invoke_classic` function; remove `ReportGenerator` type-hint imports.
  - EDIT `src/adversary_pursuit/core/console.py` — delete `--style` parser; simplify `do_report`; delete `_get_report_generator` / `_report_interview` / `_report_generator` attribute; remove `style` parameter from `_report_generate` / `_report_show`; remove `core.report` import.
  - EDIT `src/adversary_pursuit/agent/chat.py` — delete `--style` parser; delete `report answer` handler; delete classic-fallback render path; remove `_report_generator` attribute; remove three `_execute_*` imports.
  - DELETE `src/adversary_pursuit/core/report.py`.
  - DELETE `tests/fixtures/v1_classic_report.md`.
  - DELETE `tests/test_classic_style_regression.py`.
  - DELETE `tests/test_report.py`.
  - NEW `tests/test_dossier_novelty.py`.
  - NEW `tests/test_m8_cleanup_audit.py`.
  - EXTEND `tests/test_agent_tools.py` — tool count 28; novelty integration tests.
  - EXTEND `tests/test_dossier_badges.py` — Pioneer + sizes.
  - EXTEND `tests/test_badges.py` — `BadgeMetric` + `_DEFAULT_BADGES` size.
  - EXTEND `tests/test_dossier_report.py` — remove style-flag tests; assert dossier renderer remains.
  - EXTEND `tests/test_chat_report_metacommand.py` — remove `--style classic` tests; assert dossier-default behavior.

- **forbidden_shortcuts:**
  - no `core/workspace.py` modification (F59).
  - no `models/database.py` modification (DEC-DB-002 preserved; novelty cache is a separate database file).
  - no `dossier/state.py` / `dossier/predictions.py` / `dossier/scoring.py` / `dossier/slot_inference.py` / `dossier/panel.py` / `dossier/slots.py` modification.
  - no `core/pivot_policy.py` / `core/event_bus.py` / `core/dossier_pivot.py` modification.
  - no `core/config.py` modification (env-var opt-out, no config field).
  - no `core/streak.py` modification.
  - no `gamification/scoring.py` / `gamification/modes.py` / `gamification/hints.py` / `gamification/challenges.py` / `gamification/celebrations.py` / `gamification/dossier_celebrations.py` modification.
  - no new LLM tool (the M-8 tool delta is REMOVAL only: -3 classic tools, 0 additions; `generate_dossier_report` is preserved but loses the `style` parameter).
  - no `--style` parser anywhere (it was a one-release shim and the release is over).
  - no `_invoke_classic` shim wiring anywhere.
  - no SQLAlchemy / ORM layer over the novelty cache (raw `sqlite3` only — minimal-codebase principle; the cache schema is one table and does not need ORM lifecycle hooks).
  - no LLM call from `dossier/novelty.py` (novelty is structural detection, not narration).
  - no Rich markup in novelty event `rule_description` (F64).
  - no narration of `dossier_novelty_recognized` events (M-7 eligibility filter excludes them).
  - no second per-hunt `load_dossier_state` call (reuse the already-loaded `pre_dossier` / `post_dossier` snapshots).
  - no retroactive novelty scan of pre-M-8 score events (mirrors F63 quiet-start migration discipline; novelty detection runs ONLY on live hunt-emission events).
  - no cross-workspace federation / sharing of novelty hashes beyond the single user's `~/.ap/dossier_novelty.sqlite` file (v1 Non-Goal "Federation between AP instances" continues to bind; M-9 is the named future surface for any cross-user surface).
  - no `core/config.py` `novelty_*` field (env-var-only opt-out per minimal-codebase principle).
  - no GUI / cmd2 / chat meta-command toggle for novelty (env-var-only opt-out).
  - no novelty event score weight > 1 (DEC-M8-NOVELTY-006; integer-only ScoreEvent contract).
  - no tiered Pioneer badge (DEC-M8-NOVELTY-010; one threshold-1 badge suffices for M-8; tiered escalation is a future slice).
  - no UPDATE branch in `NoveltyCache.record` (PRIMARY KEY dedup via `INSERT OR IGNORE`; `workspace_count` stays at 1 in M-8; multi-workspace counter is a future surface).
  - no removal of any existing 15 badges (10 v1 + 5 M-7 dossier); no rename; no threshold change.
  - no `pyproject.toml` change (no new runtime dependency — `sqlite3` is stdlib).
  - no parallel report renderer (after M-8 exactly one report code path exists: `core/dossier_report.py::generate_dossier_report` invoked by `generate_dossier_report` LLM tool and by cmd2 `do_report` + chat `report` meta-command).
  - no `ToolContext.report_generator` field (the field exists only for the deleted interview path; removing the field with the deleted code prevents future implementers from re-introducing the interview).

- **rollback_boundary:**
  Single feature branch revertible as one merge commit. Revert restores:
  - `src/adversary_pursuit/core/report.py` (entire v1 module).
  - `src/adversary_pursuit/core/dossier_report.py::_invoke_classic` function + `ReportGenerator` imports.
  - `src/adversary_pursuit/core/console.py` `--style` parser, `_get_report_generator`, `_report_interview`, `_report_generator` attribute, classic branch in `_report_generate`, `style` param on `_report_show` / `_report_generate`, `core.report` import.
  - `src/adversary_pursuit/agent/chat.py` `--style` parser, classic-branch render, `report answer` handler, `_report_generator` attribute, three `_execute_*` imports.
  - `src/adversary_pursuit/agent/tools.py` three classic tool definitions + three `_execute_*` functions + three dispatch rows + `ToolContext.report_generator` field + `style` param + classic branch in `_execute_generate_dossier_report` + `core.report` import.
  - `tests/fixtures/v1_classic_report.md`.
  - `tests/test_classic_style_regression.py`.
  - `tests/test_report.py`.
  - Revert removes `src/adversary_pursuit/dossier/novelty.py`, the `BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED` enum member, the `badge-pioneer` entry in `DOSSIER_BADGES`, the novelty event handling in `_execute_run_module`, the `_DOSSIER_ACTIONS` widening (4-tuple → 3-tuple), the `dossier/__init__.py` novelty exports, `tests/test_dossier_novelty.py`, `tests/test_m8_cleanup_audit.py`, and the M-8-specific extensions to badge tests.
  - M-8 ships NO new workspace schema, NO new workspace persistence, NO new event-bus subscriber. The new `~/.ap/dossier_novelty.sqlite` file is a USER-LOCAL FILE outside the repo / outside any workspace; revert does NOT touch it (the file persists on disk; future code that reads it will succeed or skip based on the runtime state). Removing the file is a manual cleanup the user does if desired.
  - Tool count after revert: 31 (M-7 baseline restored).

- **ready_for_guardian_definition:**
  - All required_tests green; full suite green ≥ (M-7 baseline − deleted tests) + new M-8 tests.
  - Forbidden-file `git diff main` outputs empty (paste each in the §6 list above).
  - `ls` checks for deleted files all return "No such file" (paste output).
  - `grep -rn "ReportGenerator\|start_report_interview\|answer_report_question\|_invoke_classic\|\\-\\-style" src/ tests/` returns no matches outside `.claude/plans/` / `MASTER_PLAN.md`.
  - Tool count audit at exactly 28.
  - `~/.ap/dossier_novelty.sqlite` round-trip demonstrated in Stage E demo evidence (file created on first novel observation; `sqlite3 ... "SELECT *"` returns the row).
  - `_DOSSIER_ACTIONS` widening tested: includes `dossier_novelty_recognized`; LLM-facing summary contains no novelty text.
  - **Phase 17K appended to MASTER_PLAN.md AND committed in the same commit as source by the IMPLEMENTER** (AP #74 orphan-prevention).
  - Phase 17I status flipped in-progress → completed in the same commit with M-6's merge `1e5e09d` + impl `aa9cec8`.
  - Phase 17J status flipped in-progress → completed in the same commit with M-7's merge `55aa1fe` + impl `1127144`.
  - Active Phase Pointer tail-line re-pointed from `W-68-M7-REPORTS-CELEBRATIONS` to `W-68-M8-CLEANUP-NOVELTY`.
  - Implementer commit message follows `feat(dossier-m8):` prefix and references `#68` + `DEC-M8-CLEANUP-001..004` + `DEC-M8-NOVELTY-001..010`.

---

## 7. Scope Manifest (full)

See `tmp/m8-scope.json` for the canonical CLI-key JSON shape.

**Allowed / Required (the implementer MUST touch these):**

- `src/adversary_pursuit/dossier/novelty.py` (NEW — hash + cache + detector + emitter + opt-out helper).
- `src/adversary_pursuit/dossier/__init__.py` (EXTEND — export `compute_novelty_hash`, `NoveltyCache`, `detect_novelty`, `emit_dossier_novelty_recognized_event`).
- `src/adversary_pursuit/agent/tools.py` (EXTEND + REMOVE — novelty detection in `_execute_run_module`; `_DOSSIER_ACTIONS` widening; remove three classic tool definitions; remove three dispatch rows; remove three `_execute_*` functions; remove `ToolContext.report_generator`; remove `core.report` import; remove `style` param + classic branch from `_execute_generate_dossier_report`).
- `src/adversary_pursuit/gamification/dossier_badges.py` (EXTEND — append `badge-pioneer` to `DOSSIER_BADGES`; extend `build_dossier_stats`).
- `src/adversary_pursuit/gamification/badges.py` (EXTEND — add `BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED` enum member).
- `src/adversary_pursuit/core/dossier_report.py` (EDIT — remove `_invoke_classic`; remove `ReportGenerator` imports).
- `src/adversary_pursuit/core/console.py` (EDIT — remove `--style` parser; simplify `do_report`; remove `_get_report_generator` / `_report_interview` / `_report_generator` attr; remove `style` param on `_report_*` methods; remove `core.report` import).
- `src/adversary_pursuit/agent/chat.py` (EDIT — remove `--style` parser; remove `report answer` handler; remove classic-fallback render; remove `_report_generator` attribute; remove three `_execute_*` imports).
- `src/adversary_pursuit/core/report.py` (DELETE — entire file).
- `tests/fixtures/v1_classic_report.md` (DELETE).
- `tests/test_classic_style_regression.py` (DELETE).
- `tests/test_report.py` (DELETE).
- `tests/test_dossier_novelty.py` (NEW — Stage B).
- `tests/test_m8_cleanup_audit.py` (NEW — Stage A).
- `tests/test_agent_tools.py` (EXTEND — Stage C + tool count 28 + parameterless dispatcher).
- `tests/test_dossier_badges.py` (EXTEND — Stage D + Pioneer).
- `tests/test_badges.py` (EXTEND — BadgeMetric + sizes).
- `tests/test_dossier_report.py` (EXTEND — remove style-flag tests; keep dossier-default tests).
- `tests/test_chat_report_metacommand.py` (EXTEND — remove `--style classic` tests; keep dossier-default tests).
- `MASTER_PLAN.md` — Phase 17K section authored by the planner stage; Phase 17I + Phase 17J status flips to completed (with M-6 and M-7 SHAs filled in); Plan Status table row added; Active Phase Pointer tail-line re-pointed. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention).**
- `.claude/plans/dossier-m8-cleanup-novelty.md` — THIS FILE. Planner stage commits it (staged-not-committed; implementer commits as part of the source commit per AP #74).
- `tmp/m8-scope.json` — canonical scope JSON for runtime scope-sync.
- `tmp/evidence-m8-cleanup-novelty/` (NEW directory — implementer captures Stage E demo evidence here).

**Forbidden (preserved authorities):**

- `src/adversary_pursuit/core/workspace.py` (F59 BYTEWISE UNCHANGED).
- `src/adversary_pursuit/models/database.py` (DEC-DB-002 preserved; novelty cache is a separate database file at `~/.ap/dossier_novelty.sqlite`, NOT a workspace table).
- `src/adversary_pursuit/core/pivot_policy.py` (F60 invariant).
- `src/adversary_pursuit/core/event_bus.py` (F60 invariant).
- `src/adversary_pursuit/core/dossier_pivot.py` (M-6 byte-identical).
- `src/adversary_pursuit/core/config.py` (no novelty config field; env-var-only opt-out).
- `src/adversary_pursuit/core/streak.py` (F62 invariant).
- `src/adversary_pursuit/dossier/slots.py` (DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 preserved).
- `src/adversary_pursuit/dossier/state.py` (M-4 byte-identical).
- `src/adversary_pursuit/dossier/predictions.py` (M-5 byte-identical).
- `src/adversary_pursuit/dossier/scoring.py` (M-5 byte-identical; no new event subtype in this file — the novelty emitter lives in `novelty.py`).
- `src/adversary_pursuit/dossier/slot_inference.py` (M-5 byte-identical).
- `src/adversary_pursuit/dossier/panel.py` (M-1 byte-identical).
- `src/adversary_pursuit/gamification/scoring.py` (M-3 byte-identical).
- `src/adversary_pursuit/gamification/celebrations.py` (F63 invariant).
- `src/adversary_pursuit/gamification/dossier_celebrations.py` (M-7 byte-identical; novelty event is ineligible by construction).
- `src/adversary_pursuit/gamification/modes.py`, `gamification/hints.py`, `gamification/challenges.py` (no surface changes).
- `src/adversary_pursuit/modules/**` (no module changes).
- `pyproject.toml` (no new runtime dependency — `sqlite3` is stdlib).
- `CLAUDE.md`, `AGENTS.md`, `settings.json`, hooks, `runtime/`, `agents/` (no governance / harness changes).

**Authority domains touched:**

- `dossier_novelty_cache` (NEW — `~/.ap/dossier_novelty.sqlite` cross-workspace SQLite, owner = `dossier/novelty.py::NoveltyCache`).
- `dossier_novelty_hash` (NEW — `dossier/novelty.py::compute_novelty_hash` is sole authority for (slot, extractor, sco_type_set) → 64-char SHA-256 hex).
- `dossier_novelty_detector` (NEW — `dossier/novelty.py::detect_novelty` is sole authority for novelty determination + env-var opt-out).
- `dossier_novelty_event` (NEW — `dossier/novelty.py::emit_dossier_novelty_recognized_event` is sole authority for `dossier_novelty_recognized` ScoreEvent shape).
- `dossier_novelty_opt_out_env_var` (NEW — `AP_NO_NOVELTY` env var, owner = `dossier/novelty.py::novelty_enabled`).
- `dossier_badges_catalog` (EXTENDED — `DOSSIER_BADGES` grows from 5 → 6; Pioneer appended).
- `llm_tool_catalog` (REDUCED — 31 → 28; three classic tools removed; `generate_dossier_report` parameterless).
- `report_renderer_authority` (UNIFIED — `core/dossier_report.py::generate_dossier_report` is now the sole report renderer; `core/report.py` deleted).

---

## 8. Decision Log

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M8-CLEANUP-001** | The classic-shim removal and the novelty-achievement sub-slice ship in ONE feature branch with ONE merge commit. They are not split into two slices despite being orthogonal. | The two halves share the same `agent/tools.py::_execute_run_module` integration site and the same `gamification/dossier_badges.py` + `gamification/badges.py::BadgeMetric` extension surface. Splitting them would require two separate rebases through the same files and would leave a transient state where `_DOSSIER_ACTIONS` widens (M-8 novelty) before the classic surfaces shrink — two windows of inconsistency. One combined slice keeps the cleanup and the new achievement co-shipped, mirrors M-7's three-sub-slice pattern, and minimises the merge surface. |
| **DEC-M8-CLEANUP-002** | The `--style {dossier,classic}` flag is REMOVED ENTIRELY at M-8 — not converted to a deprecation emitter. cmd2 `do_report` and chat `report` meta-command no longer parse `--style` at all. | DEC-68-DOSSIER-REFRAME-008 named M-8 as the removal point; one release cycle (v0.2.x) was the deprecation runway. Converting the flag to a warning-and-fallback would create a permanent deprecation surface that no user has asked for. Minimal-codebase principle: delete the abstraction together with its target. The `report --style classic generate` form will produce an "unknown subcommand" message — that is the desired user-feedback path. |
| **DEC-M8-CLEANUP-003** | `core/report.py` is DELETED as a whole file, not gradually trimmed. The three classic LLM tools (`start_report_interview` / `answer_report_question` / `generate_report`) are DELETED together with the module they invoke. `tests/test_report.py` and `tests/test_classic_style_regression.py` are DELETED together with the code they test. | Whole-file deletion gives a clean `git diff` shape and a clean removal trail. Gradual trimming leaves dead code in successive commits, complicates rebase, and contradicts Sacred Practice 12's "remove superseded paths" disposition. The deletion is reversible via one revert if needed. |
| **DEC-M8-CLEANUP-004** | The `style` parameter on `generate_dossier_report` LLM tool is REMOVED at M-8. The tool becomes parameterless. `_execute_generate_dossier_report(ctx)` has no `style` argument. | The `style` parameter existed solely to route to the classic shim. With the classic shim deleted, the parameter is dead code. Removing it keeps the tool surface minimal and forces the LLM tool catalog to a clean post-M-8 shape (28 tools, one of which is `generate_dossier_report`). |
| **DEC-M8-NOVELTY-001** | The novelty cache lives at `~/.ap/dossier_novelty.sqlite` — a separate global SQLite database file outside any workspace. It is NOT a table in the workspace SQLite. | The roadmap's intent is cross-workspace novelty recognition. A workspace-local table would defeat the achievement (every new workspace would re-discover the same "novelties"). The `~/.ap/` path is already established for cross-workspace state (`core/config.py`, `core/pivot_policy.py` allowlist/denylist). The two databases own distinct domains (workspace facts vs cross-workspace novelty registry); neither is the authority for the other; Sacred Practice 12 is honoured by domain separation. Schema is one table with `hash` as PRIMARY KEY for `INSERT OR IGNORE` dedup. |
| **DEC-M8-NOVELTY-002** | The novelty hash inputs are `(slot, extractor_name, sorted(sco_types))`. The hash function is `sha256(f"{slot.value}|{extractor_name}|{','.join(sorted(sco_types))}").hexdigest()`. | The SCO-type SET (sorted, deduped) is the semantic match for "novel method": the same (slot, extractor) with the same SCO inputs is the same analytic move; different inputs make it new. Order in which slots fill within a hunt was rejected (too permissive — fires too often). Multi-hunt sequence was rejected (almost no two hunts replicate; loses semantic meaning). The `|` separator is unambiguous (cannot appear in slot values or sorted SCO-type lists). 64-char SHA-256 hex is the natural PRIMARY KEY type. |
| **DEC-M8-NOVELTY-003** | `NoveltyCache` is a thin class around raw `sqlite3` — no SQLAlchemy ORM, no Pydantic validation, no migration framework. Cache file is lazily created on first write. Schema is `IF NOT EXISTS`-bootstrapped on first `record()` call. | Minimal-codebase principle: a one-table cache does not need ORM lifecycle hooks. Raw `sqlite3` with `INSERT OR IGNORE` for PRIMARY KEY dedup is the smallest abstraction. Lazy file creation means users who opt out (`AP_NO_NOVELTY=1`) never accumulate any cache file. The `IF NOT EXISTS` schema bootstrap means the implementer ships no migration script. |
| **DEC-M8-NOVELTY-004** | Cache schema is one table `novelty_hashes(hash PRIMARY KEY, slot, extractor, ordering_sig, first_seen_at, workspace_count INT DEFAULT 1)`. `workspace_count` stays at 1 in M-8 — no UPDATE branch. | Forward-compatible shape: a future slice can promote `workspace_count` to a real counter by adding an UPDATE branch in `record()` without a schema migration. M-8 ships the column at default 1 for shape only — the multi-workspace counter semantic is deferred. PRIMARY KEY on `hash` is the dedup mechanism. `first_seen_at` is ISO-8601 UTC string (matches existing AP timestamp style; no SQLite `DATETIME` type). |
| **DEC-M8-NOVELTY-005** | `detect_novelty(slot, extractor_name, sco_types, cache)` returns True on first occurrence (and writes to cache), False on repeat (no write). The detector is the sole authority for the "is this novel" question. `dossier/scoring.py` calls into `novelty.py`, not the other way around. | One authority per operational fact (Sacred Practice 12): "is this slot-fill method novel?" has exactly one answer, computed by `detect_novelty`. The function is pure-effect: deterministic decision + at most one side-effect write per True path. The `dossier/scoring.py` boundary is preserved by routing all novelty logic through `novelty.py` — `scoring.py` is BYTEWISE UNCHANGED. |
| **DEC-M8-NOVELTY-006** | The `dossier_novelty_recognized` ScoreEvent emits `points=1`. Integer-only. | The integer-only ScoreEvent contract (M-3 floor-cast `int(SLOT_WEIGHTS[slot])`) is preserved — a float weight would break the contract. Per-IOC baseline of 1 (DEC-M3-DOSSIER-004 retune) sets the established "small, recognisable, recurrent" tier; novelty fits that tier. The prestige surface for novelty is the badge unlock (RARE), not the score points. Score events sum into `total` and drive existing celebration ASCII; no special-case scoring math. |
| **DEC-M8-NOVELTY-007** | Novelty detection runs in `_execute_run_module` BETWEEN dossier-event emission (steps 1-4) and the M-7 narration loop (step 5). All events including novelty are persisted in one `store_score_events` call. | Insertion order matters because the narration loop iterates `events` and the novelty event must be in that list for the F64 filter test to assert its suppression. Persisting in one call (not two) preserves M-4/M-5/M-6 transactional discipline (one workspace write per hunt). Narration eligibility filter (`is_high_weight_event`) naturally excludes the novelty event by action-name mismatch — no `dossier_celebrations.py` modification. |
| **DEC-M8-NOVELTY-008** | Opt-out is via `AP_NO_NOVELTY` environment variable (any truthy value disables). No `core/config.py` field. No GUI/cmd2/chat toggle. Default behavior is ON. | Mirrors `AP_NO_BANNER` pattern already used in `core/console.py:240`. Minimal-codebase principle: today there is no user case to expose the toggle in config; adding a field bloats the TOML surface and the doc speculatively. Env-var-only opt-out keeps the M-8 footprint minimal. Lazy cache file creation means opt-out users never accumulate a cache file. Promotion to config can land as a single-DEC future slice if a tuning case surfaces. |
| **DEC-M8-NOVELTY-009** | `_DOSSIER_ACTIONS` in `agent/tools.py` widens from 3-tuple to 4-tuple to include `"dossier_novelty_recognized"`. The F64 suppression test extends with the new action. | Same pattern M-5 used (DEC-M5-FALSIFY-005 widened 2-tuple → 3-tuple). The widening is the F64-compliant pattern: novelty event text is suppressed from LLM-facing `summary` (DEC-64-LLM-PANEL-SEPARATION-001 preserved) while the event still flows through `result["score_events"]` for LLM reasoning, badge-stats computation, and celebration ASCII summing. |
| **DEC-M8-NOVELTY-010** | One new badge `badge-pioneer` (Pioneer, RARE). Metric `DOSSIER_NOVELTY_RECOGNIZED`. Threshold 1. Fires on the first `dossier_novelty_recognized` event in the workspace. NEW `BadgeMetric.DOSSIER_NOVELTY_RECOGNIZED` enum member. `DOSSIER_BADGES` grows from 5 → 6 entries. `_DEFAULT_BADGES` grows from 15 → 16. | Minimal-codebase principle: one threshold-1 badge is the smallest abstraction matching the achievement need (recognise that ANY novel method was discovered). Tiered Pioneer (e.g., 3/10/25) was considered and rejected for M-8 — no telemetry shows users care about a 25-novelty horizon today. RARE matches "directed achievement" tier (Identity First, Deception Spotter); LEGENDARY would over-elevate a single-event achievement; UNCOMMON would under-recognise the genuine analytic insight. Future slices can add tiered Pioneer if play-pattern data motivates it. |

---

## 9. Subsequent workflow cue

M-8 closes the v0.3.x dossier roadmap. After M-8 lands:

- **M-9 (Crowdsourced Dossier Comparison + Public Actor Library)** is the next scheduled roadmap surface per DEC-68-DOSSIER-REFRAME-009 (v0.3.0+; out of M-8 critical path; STIX-bundle-based dossier export/import + comparison metric + opt-in public library). M-9 reads naturally from the M-4 persistent `DossierState` + M-5 `PersistedPrediction` log + the M-8 novelty cache. The novelty cache becomes a candidate input for "compare your method library to the community library" if M-9 schedules it.
- **C-3 (Philosophy + Bureaucratese modes)** remains independent of the dossier roadmap per DEC-68-DOSSIER-REFRAME-004 and may schedule in parallel with M-9 whenever the user prioritises it.
- **C-4 (mastery_level hook)** — Phase 17 deferred slice — remains independent and unscheduled.

No M-9 work is dispatched as part of M-8. M-8's `PLAN_VERDICT: next_work_item` points at `wi-68-m8-impl-01` (the implementer slice authored by THIS planner stage); after that implementer slice lands, the orchestrator's autonomous-continuation decision will be either `goal_complete` (v0.3.x dossier roadmap closed; user picks the next product direction — M-9 vs C-3 vs runtime-hygiene backlog) or `needs_user_decision` (mutually exclusive product paths).

---

## 10. Risks and open follow-ups

**Risks:**

- **Cache file location collision.** If a user already has `~/.ap/dossier_novelty.sqlite` from an unrelated tool (extremely unlikely — the file name is AP-specific), the first M-8 write could find an incompatible schema. Mitigated by `CREATE TABLE IF NOT EXISTS` and explicit column-shape check in `NoveltyCache.__init__` (PRAGMA table_info; if non-matching, raise a clear error message — implementer authors that guard).
- **Hash collision.** SHA-256 collision space is 2^256; the cache will not grow large enough in practice to make collision a real concern. The cost of a collision is "a genuinely novel method is wrongly flagged as already seen" — a false negative on the badge. Not a correctness risk for any other system.
- **Tool-count assertions in tests.** Multiple tests assert `len(tools) == 31` (M-7 baseline). Implementer must grep `tests/` for `== 31` and update all matches to `== 28`. Audit at deletion time.
- **Cross-platform path semantics.** `Path.home() / ".ap" / "dossier_novelty.sqlite"` resolves to different OS paths (`~/.ap/...` on POSIX, `C:\Users\<name>\.ap\...` on Windows). The `pathlib.Path` API normalises this — no special-case code needed. Lazy file creation calls `path.parent.mkdir(parents=True, exist_ok=True)` before the first `sqlite3.connect`.
- **Tests calling `~/.ap/dossier_novelty.sqlite` directly.** Test isolation requires `NoveltyCache(path=tmp_path / "novelty.sqlite")` injection. Implementer must NOT write to the user's real cache during test runs. The `pytest` `tmp_path` fixture pattern is the cleanest isolation.
- **Concurrent writes from multiple AP processes.** Two `ap chat` sessions writing to the cache simultaneously rely on SQLite's default journal mode (WAL or rollback) for write serialisation. `INSERT OR IGNORE` makes the write idempotent under race. Acceptable.
- **`AP_NO_NOVELTY` truthy semantics.** Empty string / unset env var → enabled; any non-empty string → disabled. Implementer should test `"0"` and `"false"` to confirm — these are non-empty strings and therefore DISABLE novelty. This matches `AP_NO_BANNER` semantics. Documented in the docstring.

**Open follow-ups (out-of-scope for M-8):**

- **Tiered Pioneer badge** — deferred per DEC-M8-NOVELTY-010. Add when play-pattern telemetry shows users care about the 25-novelty horizon.
- **`workspace_count` multi-workspace counter** — deferred per DEC-M8-NOVELTY-004. Schema column is reserved; an UPDATE branch in `NoveltyCache.record` can promote it in a future single-DEC slice.
- **Config-file novelty toggle** — deferred per DEC-M8-NOVELTY-008. Promotion path: add `core/config.py::NoveltyConfig` if a user case surfaces.
- **`generate_dossier_report` LLM tool returning novelty section** — out of scope; the dossier report already renders dossier slots + predictions + analyst notes; adding a novelty section would touch `core/dossier_report.py` beyond the `_invoke_classic` deletion. Schedule as a future slice if users want the novelty trail visible in reports.
- **Catch-up novelty scan of pre-M-8 score events** — explicitly out. Mirrors F63 quiet-start migration discipline. Novelty detection runs only on live hunt-emission events.
- **Cross-user federation of novelty hashes** — explicitly out. v1 Non-Goal "Federation between AP instances" continues to bind. M-9 is the named future surface for any cross-user mechanic.
- **Novelty cache cleanup / TTL** — out. The cache is append-only in M-8; rows live forever. A future slice can add a TTL field or a cleanup CLI if disk-growth becomes a concern.

---

## 11. Notes for the implementer

- **Worktree base.** This planner-staged content sits on `feature/68-m8-cleanup-novelty` at AP main `55aa1fe` (M-7 merge head). No rebase needed; M-7 has already landed.
- **No source code on the orchestrator's behalf.** This file is planner-staged content. The implementer reads it, the implementer commits the source against it, the implementer flips the Phase 17K status from in-progress to completed in MASTER_PLAN.md when committing source (AP #74 orphan-prevention).
- **Planner artifacts are staged-not-committed.** The planner stage runs `git add` on `.claude/plans/dossier-m8-cleanup-novelty.md`, `MASTER_PLAN.md`, and `tmp/m8-scope.json` but does NOT commit (planner role lacks `can_commit_feature_branch`). The implementer commits these together with the source changes in one commit.
- **Cleanup ordering.** Suggested order to minimise transient broken-state windows during the implementer's edit session:
  1. Add `novelty.py` + extend `__init__.py` exports + extend `BadgeMetric` enum + extend `DOSSIER_BADGES`.
  2. Wire novelty detection into `_execute_run_module`; widen `_DOSSIER_ACTIONS` to 4-tuple.
  3. Author the new tests (`test_dossier_novelty.py`, `test_m8_cleanup_audit.py`, badge extensions).
  4. Run the new tests; confirm green. (Both halves of the slice exercise: novelty works, audit tests pass with the classic shim STILL present — the audit tests will fail on the deletion step.)
  5. Edit `core/dossier_report.py` to remove `_invoke_classic`.
  6. Edit `agent/tools.py` to remove the three classic tools + dispatch rows + `_execute_*` functions + `ToolContext.report_generator` + `style` parameter + `core.report` import.
  7. Edit `core/console.py` to remove `--style` parser + `_get_report_generator` / `_report_interview` + classic branch in `_report_generate` + `style` params + `core.report` import.
  8. Edit `agent/chat.py` to remove `--style` parser + `report answer` handler + classic-fallback render + `_report_generator` attribute + three `_execute_*` imports.
  9. Delete `core/report.py`, `tests/fixtures/v1_classic_report.md`, `tests/test_classic_style_regression.py`, `tests/test_report.py`.
  10. Update `tests/test_dossier_report.py`, `tests/test_chat_report_metacommand.py`, `tests/test_agent_tools.py` to remove style-flag tests and update tool count to 28.
  11. Run the full test suite; confirm green.
  12. Capture Stage E demo evidence under `tmp/evidence-m8-cleanup-novelty/`.
  13. `git add MASTER_PLAN.md .claude/plans/dossier-m8-cleanup-novelty.md tmp/m8-scope.json` (the planner-staged trio) together with all source/test changes.
  14. Commit with `feat(dossier-m8): cleanup + novel-method achievement (#68)` referencing the DEC range `DEC-M8-CLEANUP-001..004 / DEC-M8-NOVELTY-001..010`.
- **MASTER_PLAN.md edits the implementer owns.** Phase 17K status flip from in-progress → completed; Phase 17I status flip from in-progress → completed with M-6 SHAs (merge `1e5e09d` / impl `aa9cec8`); Phase 17J status flip from in-progress → completed with M-7 SHAs (merge `55aa1fe` / impl `1127144`); Plan Status table row for `W-68-M8-CLEANUP-NOVELTY`; Active Phase Pointer tail-line re-pointed.
- **Active Phase Pointer.** Re-point the tail-line in MASTER_PLAN.md from `W-68-M7-REPORTS-CELEBRATIONS` to `W-68-M8-CLEANUP-NOVELTY`. The tail-line position requirement (last `**Phase ...` boldline) holds — keep the section at the end of the doc.
- **Commit message prefix.** `feat(dossier-m8):` is the canonical prefix. Body references `#68` and the DEC range `DEC-M8-CLEANUP-001..004` + `DEC-M8-NOVELTY-001..010`.
- **Audit script.** After the deletion, run:
  ```bash
  grep -rn "ReportGenerator\|start_report_interview\|answer_report_question\|_invoke_classic\|\\-\\-style" src/ tests/
  ```
  Expected result: no matches (matches in `.claude/plans/` or `MASTER_PLAN.md` are OK). Paste the empty output into the guardian-readiness evidence.
- **Cache isolation in tests.** Always inject `NoveltyCache(path=tmp_path / "novelty.sqlite")` in tests. Do NOT write to `~/.ap/dossier_novelty.sqlite` during pytest runs (the user's real cache must not be polluted by test runs). Use `monkeypatch.setattr` to substitute the default path if needed.
- **Test count expectation.** Net delta: roughly +25 new tests (Stages A-D) − ~15 deleted classic tests (entire `test_report.py` + entire `test_classic_style_regression.py` + selected style tests from `test_dossier_report.py` / `test_chat_report_metacommand.py`) = ~+10 net new tests. Full suite passes ≥ (M-7 baseline − deletions) + new M-8 tests.
