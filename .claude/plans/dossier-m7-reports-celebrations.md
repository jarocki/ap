# M-7 — Reports / Celebrations / Badges Dossier-Aware Upgrade (per-slice plan)

**Status:** planner-staged 2026-06-08 by W-68-M7-REPORTS-CELEBRATIONS planner stage. Implementer slice `wi-68-m7-impl-01` to follow.
**Workflow:** `w-68-m7-reports-celebrations`
**Goal:** `g-68-m7-reports`
**Work item to dispatch:** `wi-68-m7-impl-01`
**Drives:** Phase 17J of `MASTER_PLAN.md`. Phase 17J carries the binding decisions and slice index; this document carries full rationale, layering diagram, threshold derivation, and decomposition detail. When the two diverge, Phase 17J wins for binding decisions; this document wins for narrative.

**Inherits from:** Phase 16 §M-7, `.claude/plans/dossier-reframe-v2-roadmap.md` §M-7. Phase 6 (v1 report — DEC-AGENT-REPORT-*, DEC-REPORT-001..003), Phase 9 (W-AGENT-CELEBRATIONS / `CelebrationEngine`), Phase 12C (F63 milestones + DEC-63-MILESTONE-CATCHUP-001), Phase 13 (F64 panel-separation sidecar / DEC-64-LLM-PANEL-SEPARATION-001), Phase 17B (M-1 dossier panel + SLOT_WEIGHTS authority), Phase 17D (M-2 `infer_dossier_state_full` + extractors), Phase 17F (M-3 dossier scoring events + `_DOSSIER_ACTIONS` F64 filter), Phase 17G (M-4 persistent `DossierState` + `PersistedPrediction` log + `load_dossier_state` / `load_predictions_log`), Phase 17H (M-5 `AnalystNote`-table user-note authoring + active falsification engine), Phase 17I (M-6 dossier-aware auto-pivot ranker) are prerequisites. M-6 is planner-staged and expected to land before M-7 implementer dispatch; the M-7 implementer rebases on whatever main contains after M-6 lands. Worktree base: AP main at merge `1e5e09d` (M-6 merge head; impl `aa9cec8`).

---

## 1. Goal (single paragraph)

Bind the three remaining gamification surfaces — reports, celebrations, and badges — to the dossier state authority that M-1..M-6 already produces. Three orthogonal but co-shipped sub-slices:

1. **Reports.** The default `report generate` / `generate_report` output becomes the **actor-dossier report** rendered from M-4 `DossierState` + M-5 `AnalystNote` rows + M-5/M-4 `PersistedPrediction` log + workspace STIX + module-run history. The v1 interview-based investigation report is preserved verbatim behind a `--style classic` flag (cmd2) / `style="classic"` parameter (chat meta-command + LLM tool) for one release cycle (v0.2.x) and is scheduled for removal in M-8 cleanup per DEC-68-DOSSIER-REFRAME-008. Output formats unchanged from v1: Markdown text (single string return / file save). No new format introduced.
2. **Celebrations.** High-weight dossier slot-fill score events (slot weight ≥ 2.5: Identity / Predictions / Capability / TTPs / Motivation / Targeting / Denial) fire **LLM-narrated celebration text** rendered through the **existing Rich panel pipeline** (`runner.last_celebrations` sidecar → `chat.py` celebration-panel loop). Routine events (slot weight < 2.5: Infrastructure / Timing, plus all per-IOC discovery events) keep the v1 ASCII-art `CelebrationEngine.celebrate()` path byte-identical. Token budget is hard-capped per narration and per hunt; LLM failure silently falls back to ASCII art at runtime and is asserted as a loud test case.
3. **Badges.** Five new **dossier-aware badges** added to `_DEFAULT_BADGES`, each keyed on a new `DossierMetric` enum value computed from `DossierState` + `PersistedPrediction` log. Existing 10 badges and their thresholds stay byte-identical (additive only). The badge-event pipeline (`WorkspaceManager.store_badge_event` + `runner.last_badges` sidecar + `chat.py` badge-panel loop) is unchanged.

Preserves M-1..M-6 + F59/F60/F61/F62/F63/F64 invariants by construction. Tool count grows by exactly one: a new `generate_dossier_report` LLM tool surfacing the dossier report (DEC-M7-REPORT-005). Existing `start_report_interview` / `answer_report_question` / `generate_report` tools stay for the classic interview shim. `core/workspace.py`, `models/database.py`, every `dossier/*.py` module, every M-3/M-4/M-5/M-6 score event subtype, and every gate of F60's `PivotPolicy` are BYTEWISE UNCHANGED.

**Out-of-scope (explicit, deferred):**

- **No new ScoreEvent subtype.** M-7 consumes the existing `dossier_slot_filled` / `dossier_prediction_validated` / `dossier_prediction_falsified` events as celebration triggers; it does not introduce a new score event.
- **No `_DOSSIER_ACTIONS` widening.** The 3-tuple frozenset stays byte-identical. LLM-narrated celebration text is rendered via Rich panel (`runner.last_celebrations`), not via the LLM-facing `summary` string — DEC-64-LLM-PANEL-SEPARATION-001 preserved.
- **No `core/workspace.py` modification.** BYTEWISE UNCHANGED (stronger than M-4's narrow-edit clause; matches M-5/M-6 discipline).
- **No `models/database.py` modification.** DEC-DB-002 preserved (no schema migration). New badges key off in-memory `DossierState` + `PersistedPrediction` JSON payload via a new `BadgeMetric.DOSSIER_*` enum + a new dossier-stats dict layered on top of the existing `WorkspaceManager.get_workspace_stats()` return.
- **No `dossier/state.py`, `dossier/predictions.py`, `dossier/scoring.py`, `dossier/slot_inference.py`, `dossier/panel.py`, `dossier/slots.py` modification.** M-7 is a read-only consumer of the dossier authority.
- **No persona-LLM swap.** Narration uses the active `AgentRunner` model + the active persona system prompt that's already wired through C-1+; M-7 does not introduce a parallel LLM client. The narration call site is `agent/runner.py::AgentRunner` (extended; see §2.4) — same `litellm.completion` shape as the chat path. No new provider, no new auth, no new API.
- **No PDF / HTML / JSON output format.** Markdown only — same as v1 (DEC-REPORT-002 preserved). PDF/HTML output is explicitly deferred to a future slice.
- **No celebrations history.** Celebrations are session-scoped (`runner.last_celebrations` clears per turn). No persistence of narration text. The score event row that triggered the celebration is the persistent record.
- **No `core/console.py` LLM narration.** The cmd2 console path does not gain LLM-narrated celebrations. The cmd2 console renders `CelebrationEngine.celebrate()` ASCII art unchanged for all events. LLM narration is an `ap chat` (agent) surface only because the chat surface owns the live LLM session; the cmd2 surface has no LLM client. The cmd2 `report` command DOES gain the dossier-default + `--style classic` shim (mirrors `ap chat report` meta-command parity).
- **No celebration narration in classic-report output.** Narration is a celebration surface, not a report surface. The report renderer reads dossier state for content; it does not render narration text.
- **No new persona-mode behavior.** The active `CharacterMode.llm_profile` is consulted only to flavor narration voice (the M-7 narration call uses the same system prompt the chat path uses, so any active profile's voice naturally bleeds through). No new persona fields, no profile authoring per M-7.
- **No backlog of pre-M-7 score events for catch-up narration.** M-7 fires narration only on **live** slot-fill events during the current hunt. Score events stored in prior sessions are not retroactively narrated (matches F63 quiet-start migration discipline — DEC-63-MIGRATION-001).

---

## 2. Architecture

### 2.1 Layering authority — three sub-slices, three small NEW modules

```
+--------------------------------------------------------------------------+
|  Sub-slice 1: Reports                                                    |
|                                                                          |
|  NEW MODULE: core/dossier_report.py                                      |
|    - generate_dossier_report(workspace_mgr, *, scoring_engine=None) -> str|
|      Reads: load_dossier_state, load_predictions_log,                    |
|             workspace_mgr.get_stix_objects, get_module_runs,             |
|             get_total_score, get_stix_type_counts, AnalystNote rows.     |
|      Returns: complete Markdown report string.                           |
|    - render_dossier_slots_section(state) -> str (private)                |
|    - render_predictions_section(predictions) -> str (private)            |
|    - render_analyst_notes_section(notes) -> str (private)                |
|                                                                          |
|  STYLE DISPATCH (single switch):                                         |
|    style="dossier" (default) → core.dossier_report.generate_dossier_report|
|    style="classic"          → ReportGenerator(workspace_mgr).generate() |
|                                  (verbatim v1 path, removed at M-8)      |
|                                                                          |
|  NEW LLM TOOL: generate_dossier_report (count: 30 → 31)                  |
|    The existing generate_report tool keeps current semantics             |
|    (classic interview path) for v0.2.x; M-8 removes it together with     |
|    the classic shim per DEC-68-DOSSIER-REFRAME-008.                      |
|                                                                          |
|  EXTEND core/console.py do_report:                                       |
|    parse --style flag; default="dossier".                                |
|                                                                          |
|  EXTEND agent/chat.py 'report' meta-command:                             |
|    recognise 'report --style classic generate'  → classic path           |
|    recognise 'report generate' / 'report'       → dossier-default        |
|                                                                          |
+--------------------------------------------------------------------------+
+--------------------------------------------------------------------------+
|  Sub-slice 2: Celebrations                                               |
|                                                                          |
|  NEW MODULE: gamification/dossier_celebrations.py                        |
|    - HIGH_WEIGHT_NARRATION_THRESHOLD: float = 2.5  (DEC-M7-CELEB-002)    |
|    - PER_NARRATION_TOKEN_CAP: int = 80              (DEC-M7-CELEB-003)   |
|    - PER_HUNT_NARRATION_BUDGET: int = 3             (DEC-M7-CELEB-004)   |
|    - HuntNarrationBudget dataclass (frozen counter per hunt)             |
|    - is_high_weight_event(event: dict) -> bool                          |
|    - build_narration_prompt(event: dict, dossier_state) -> str          |
|    - narrate_celebration(runner, event, dossier_state, budget)          |
|        -> str | None  (None on budget exhausted or failure; loud-fail   |
|         in tests via _NARRATION_FAILURE_HOOK)                          |
|                                                                          |
|  EXTEND agent/tools.py::_execute_run_module:                            |
|    after the existing self.celebration.celebrate(total) block, iterate  |
|    the dossier_slot_filled / dossier_prediction_validated events that   |
|    fired this hunt; for each whose slot weight ≥ 2.5 and budget remains,|
|    call narrate_celebration(...) and APPEND the result to celebration   |
|    string (Rich panel rendering unchanged — F64 invariant: still goes   |
|    through runner.last_celebrations sidecar).                           |
|                                                                          |
|  EXTEND agent/runner.py::AgentRunner with a narrow narration helper:    |
|    narrate(prompt: str, *, max_tokens: int) -> str | None              |
|    Single-turn, no tools, no conversation mutation, persona system      |
|    prompt reused. litellm.completion with max_tokens=max_tokens.        |
|    Loud-fail returns None on any exception; runtime path swallows None  |
|    silently and falls back to ASCII art (already produced earlier in    |
|    the celebration string). DEC-M7-CELEB-006.                           |
|                                                                          |
|  cmd2 path: core/console.py CelebrationEngine unchanged. No narration.  |
|                                                                          |
+--------------------------------------------------------------------------+
+--------------------------------------------------------------------------+
|  Sub-slice 3: Badges                                                     |
|                                                                          |
|  NEW MODULE: gamification/dossier_badges.py                             |
|    - DOSSIER_BADGES: list[Badge]  — five new entries                    |
|        badge-dossier-complete      (weight LEGENDARY)                    |
|        badge-identity-first        (weight RARE)                         |
|        badge-predictor             (weight UNCOMMON)                     |
|        badge-skeptic               (weight UNCOMMON)                     |
|        badge-deception-spotter     (weight RARE)                         |
|    - build_dossier_stats(dossier_state, predictions) -> dict            |
|        Returns:                                                          |
|          dossier_slots_filled (int)                                      |
|          dossier_identity_first (int 0/1)                                |
|          dossier_predictions_validated (int)                             |
|          dossier_predictions_falsified (int)                             |
|          dossier_denial_filled (int 0/1)                                |
|                                                                          |
|  EXTEND gamification/badges.py:                                          |
|    BadgeMetric enum gains 5 new values (DOSSIER_SLOTS_FILLED,           |
|    DOSSIER_IDENTITY_FIRST, DOSSIER_PREDICTIONS_VALIDATED,                |
|    DOSSIER_PREDICTIONS_FALSIFIED, DOSSIER_DENIAL_FILLED).               |
|    _DEFAULT_BADGES extended with the 5 entries from DOSSIER_BADGES      |
|    (imported from gamification/dossier_badges.py — single source of    |
|    truth; the dossier_badges module is the authority, _DEFAULT_BADGES  |
|    is the splice site).                                                  |
|                                                                          |
|  EXTEND agent/tools.py::_execute_run_module:                            |
|    immediately before the existing self.badge_mgr.check_all(...) call,  |
|    extend the badge_stats dict via build_dossier_stats(post_dossier,    |
|    predictions). post_dossier is already loaded for the post-hunt diff. |
|    predictions = load_predictions_log(workspace_mgr) (one extra call    |
|    per hunt at the badge check site — acceptable, same shape as M-4).   |
|                                                                          |
+--------------------------------------------------------------------------+
```

**Three NEW modules ship (one per sub-slice):**

- `src/adversary_pursuit/core/dossier_report.py` — dossier report renderer. Pure-function module; no I/O of its own (consumes `WorkspaceManager` for reads only — never writes). Public API: `generate_dossier_report(workspace_mgr, *, scoring_engine=None) -> str`.
- `src/adversary_pursuit/gamification/dossier_celebrations.py` — narration policy + per-hunt budget. Public API: `HIGH_WEIGHT_NARRATION_THRESHOLD`, `PER_NARRATION_TOKEN_CAP`, `PER_HUNT_NARRATION_BUDGET`, `HuntNarrationBudget`, `is_high_weight_event`, `build_narration_prompt`, `narrate_celebration`.
- `src/adversary_pursuit/gamification/dossier_badges.py` — five new badge specs + dossier-stats builder. Public API: `DOSSIER_BADGES`, `build_dossier_stats`.

**Three EXISTING modules gain narrow extensions:**

- `src/adversary_pursuit/gamification/badges.py` — `BadgeMetric` enum gains five new members; `_DEFAULT_BADGES` list extended with `DOSSIER_BADGES` (imported from `dossier_badges.py`). `Badge` dataclass, `BadgeManager` class, `check_award` contract, `get_workspace_stats` integration site — all byte-identical in signature and behavior. The metric-to-stat-key contract (DEC-BADGE-003) is preserved: the new metrics map to new stat keys produced by `build_dossier_stats`.
- `src/adversary_pursuit/agent/runner.py` — `AgentRunner` gains one new method `narrate(prompt: str, *, max_tokens: int) -> str | None`. No change to `chat`, `_call_llm`, `_extract_tool_calls`, `_extract_text`, `set_character`, `reset`. The `narrate` helper reuses the active system prompt (so the character voice flows through naturally) but does not mutate `self.conversation` and does not pass tools.
- `src/adversary_pursuit/agent/tools.py` — `_execute_run_module` gains:
  1. After `self.celebration.celebrate(total)` and before the milestone catch-up block: a small loop over `events` filtering on `dossier_slot_filled` / `dossier_prediction_validated`, gated on `is_high_weight_event`, calling `narrate_celebration(...)` with a per-hunt `HuntNarrationBudget`, and appending the returned narration string (when non-None) to the `celebration` string (still inside the F64 sidecar surface — not piped into `summary`).
  2. Before the existing `self.badge_mgr.check_all(...)` call: a small block computing `post_dossier = load_dossier_state(...) or default_deferred_state()`, `predictions = load_predictions_log(...) or []`, then merging `build_dossier_stats(post_dossier, predictions)` into `badge_stats`. (Note: when M-6's ranker-wiring branch has already loaded `post_dossier`, prefer reusing that variable — but M-6 only loads `pre_dossier`, so `post_dossier` is a new load. One extra `load_dossier_state` per hunt at the badge-check site. Same shape and idempotency discipline as M-4's `pre_dossier` load — acceptable.)
  3. A new public-API entry for `generate_dossier_report` LLM tool dispatcher: `_execute_generate_dossier_report(ctx, style="dossier") -> str` — when `style == "dossier"`, calls `core.dossier_report.generate_dossier_report(ctx.workspace_mgr)`; when `style == "classic"`, calls the existing `_execute_generate_report` path. `style="dossier"` is the default; `style="classic"` is the one-release-cycle shim per DEC-68-DOSSIER-REFRAME-008.

**Two EXISTING modules gain narrow CLI / meta-command parsing:**

- `src/adversary_pursuit/core/console.py` — `do_report` gains `--style {dossier,classic}` flag parsing (default `dossier`). When style=dossier, `_report_generate` and `_report_show` route through `core.dossier_report.generate_dossier_report`; when style=classic, they route through the existing `ReportGenerator` path verbatim. Output file naming and save semantics unchanged. The cmd2 console does NOT gain LLM celebrations (no LLM client at this surface).
- `src/adversary_pursuit/agent/chat.py` — the `report` meta-command parsing block grows a small `--style classic` recognizer; otherwise unchanged. Default subcommand routing (`report generate`, `report answer`, bare `report`) preserved. The dossier path uses the new tool dispatcher; the classic path uses the existing `_execute_generate_report` / `_execute_answer_report_question` / `_execute_start_report_interview` chain. Renders Markdown through the existing `Markdown` + `Panel` Rich path.

**Modules that are BYTEWISE UNCHANGED in M-7:**

- `src/adversary_pursuit/core/workspace.py` — F59 / M-4 / M-5 / M-6 BYTEWISE UNCHANGED (no new method, no new reserved action, no schema-touching code).
- `src/adversary_pursuit/models/database.py` — DEC-DB-002 BYTEWISE UNCHANGED.
- `src/adversary_pursuit/core/report.py` — v1 `ReportGenerator` class BYTEWISE UNCHANGED (it IS the classic shim). Removed by M-8 cleanup.
- `src/adversary_pursuit/gamification/celebrations.py` — F63 / DEC-63-MILESTONE-CATCHUP-001 BYTEWISE UNCHANGED. `CelebrationEngine`, `CELEBRATION_ART`, `MILESTONES`, `highest_crossed_milestone_id`, `check_milestones`, `first_blood_message`, `celebrate` — all preserved. M-7 narration extends celebration **rendering** at the call site (agent/tools.py); the v1 engine class itself is untouched.
- `src/adversary_pursuit/dossier/slots.py` — DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 preserved. `SLOT_WEIGHTS` is M-7's read-only input.
- `src/adversary_pursuit/dossier/state.py`, `dossier/predictions.py`, `dossier/scoring.py`, `dossier/slot_inference.py`, `dossier/panel.py`, `dossier/__init__.py` — all byte-identical.
- `src/adversary_pursuit/core/event_bus.py`, `core/pivot_policy.py`, `core/dossier_pivot.py`, `core/config.py`, `core/streak.py` — all byte-identical (M-7 does not touch the pivot/event-bus/config/streak surfaces).
- `src/adversary_pursuit/gamification/scoring.py`, `gamification/modes.py`, `gamification/hints.py`, `gamification/challenges.py` — all byte-identical.
- `src/adversary_pursuit/modules/**`, `pyproject.toml`, hooks, settings, `CLAUDE.md`, `AGENTS.md`, `agents/`, `runtime/` — unchanged.

### 2.2 Report style flag vocabulary — `--style {dossier,classic}` (DEC-M7-REPORT-001)

The dispatch context posed: a third style flag (e.g., `--style minimal` / `--style markdown-only`)? Per minimal-codebase principle, no.

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **two-value flag: `dossier` (default) + `classic` (deprecated shim)** (recommended) | `--style` accepts exactly two values for v0.2.x. M-8 removes `classic`, leaving `dossier` as the only style. | **accepted** | Two-value enum is the smallest abstraction that delivers the deprecation runway DEC-68-DOSSIER-REFRAME-008 commits to. A third value would require justification by evidence (which doesn't exist — no user has asked for a minimal/markdown-only style). Adding it speculatively violates the "no abstractions without need" principle on file in CLAUDE.md. |
| (b) three-value flag including `minimal` | Add a `minimal` style for short / executive-summary-only output. | **rejected** | No requesting user; speculative. Minimal output is one filter away from dossier output — when a user actually asks for it, that slice writes a `--style minimal` enum value and a 20-line section filter. Today it's dead weight. |
| (c) no flag — always dossier; v1 path removed in M-7 | Skip the deprecation runway; remove `core/report.py` in M-7. | **rejected** | DEC-68-DOSSIER-REFRAME-008 explicitly commits to a one-release-cycle deprecation. A v0.2.x user upgrading to v0.3.x is the audience for the shim. Forcing immediate removal violates the standing decision. |

The flag is parsed identically on three surfaces: cmd2 `do_report` (positional `--style classic`), chat meta-command (`report --style classic generate`), and the new `generate_dossier_report` LLM tool (`style` parameter, default `"dossier"`).

### 2.3 Dossier report module location — NEW `core/dossier_report.py` (DEC-M7-REPORT-002)

The dispatch context posed: sibling function in `core/report.py` or NEW `core/dossier_report.py` module? New module preferred.

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **NEW `core/dossier_report.py`** (recommended) | New module owns `generate_dossier_report` + private renderers for the slot / predictions / notes sections. | **accepted** | Separation of concerns: `core/report.py` owns the v1 interview-driven report; `core/dossier_report.py` owns the dossier-puzzle report. The two reports are not variants of one template — they are different shapes. Co-locating them would force every reader to mentally branch on style. Mirrors M-6's `core/dossier_pivot.py` vs `core/pivot_policy.py` separation. M-8's cleanup removes `core/report.py` outright, leaving `core/dossier_report.py` standing on its own — clean removal trail. |
| (b) sibling function in `core/report.py` | Add `generate_dossier_report` next to `ReportGenerator` in the existing file. | **rejected** | M-8 removes the v1 report path. If the dossier renderer lives in the same file, M-8 has to surgically extract it before deletion. Putting the new code in its own module from day one means M-8 deletes one file and leaves the other intact. |
| (c) new method on `ReportGenerator` | `ReportGenerator.generate_dossier(self)` plus a `style` parameter on the constructor. | **rejected** | `ReportGenerator` is wired to the interview pattern (per-question state, `set_answer`, `sections` list). Wedging the dossier path into it would force the dataclass to carry fields it doesn't need. The dossier report has no interview to capture; coupling it to `ReportGenerator` adds shape impedance. |

### 2.4 Narration call site — `AgentRunner.narrate` helper (DEC-M7-CELEB-001)

The dispatch context posed: where does the narration LLM call live? Options: inside `gamification/dossier_celebrations.py` directly (it constructs its own litellm client), or as a small helper on `AgentRunner`. Helper preferred.

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **`AgentRunner.narrate(prompt, *, max_tokens)` helper** (recommended) | `narrate` reuses the active model, the active persona system prompt, the active API key resolution, and the active litellm import. `dossier_celebrations.narrate_celebration(...)` calls `runner.narrate(prompt, max_tokens=PER_NARRATION_TOKEN_CAP)`. | **accepted** | Single source of truth for the LLM client lives in `AgentRunner` already. Mirroring its `_call_llm` shape into a parallel client in `dossier_celebrations.py` would (i) duplicate API-key resolution, (ii) force persona injection elsewhere, (iii) violate Sacred Practice 12. `narrate` is a 25-line method that calls `litellm.completion(model=self.model, messages=[{"role": "system", "content": self.system_prompt}, {"role": "user", "content": prompt}], max_tokens=max_tokens, tools=None, tool_choice=None)`. No new auth, no new provider hookup. Active persona voice flows through naturally because `self.system_prompt` was already mutated by `set_character`. |
| (b) `dossier_celebrations.py` owns its own litellm client | Module-level helper that reads config, builds its own messages, calls litellm. | **rejected** | Duplicates `_call_llm` semantics; reintroduces persona-prompt injection in a second place; doubles the API-key resolution surface area; violates Sacred Practice 12. |
| (c) shell out via `runner.chat(narration_prompt)` reusing the existing tool surface | Feed a "narrate this please" string to the chat surface and capture the response. | **rejected** | Pollutes the user's conversation history with a mid-hunt narration round-trip; the LLM would see it as part of the analytical conversation. Also triggers tool-choice arbitration with the full toolset — token waste and the wrong system signal. |

### 2.5 High-weight threshold for LLM narration — ≥ 2.5 (DEC-M7-CELEB-002)

Slot weights live at `dossier/slots.py::SLOT_WEIGHTS`:

| Slot | Weight |
|------|--------|
| Identity | 5.0 |
| Predictions | 4.0 |
| Capability | 3.5 |
| TTPs | 3.0 |
| Motivation | 3.0 |
| Targeting | 2.5 |
| Denial | 2.5 |
| Infrastructure | 2.0 |
| Timing | 2.0 |

The threshold ≥ 2.5 captures Identity / Predictions / Capability / TTPs / Motivation / Targeting / Denial (7 of 9 slots). Routine: Infrastructure / Timing (2 of 9). The cut matches the natural break in the weight distribution between the "downstream-derivable" tier (2.5) and the "baseline-above-routine" tier (2.0). It is also the same cut that distinguishes "puzzle-keystone" slots (analyst signal that meaningfully advances the dossier) from "ambient" slots (low-effort enrichment).

Per-IOC score events (action prefix `score_results_*`) are always routine — they fire at points 1 per indicator per the M-3 re-tune (DEC-M3-DOSSIER-004), well below the threshold by any reasonable transform. The narration filter operates only on `dossier_slot_filled` and `dossier_prediction_validated` events; per-IOC events are never narrated.

`dossier_prediction_falsified` events fire at **points=0** per DEC-M4-PRED-006 (no negative-points events). They are not "achievement"-shaped (they're contradiction signals) and would be tone-mismatched if rendered as a celebration. M-7's narration explicitly **excludes** `dossier_prediction_falsified` events from narration eligibility (DEC-M7-CELEB-005). The Skeptic badge fires on falsifications, but the badge-panel surface (not the celebration-panel surface) carries the recognition.

### 2.6 Per-narration token cap — 80 tokens (DEC-M7-CELEB-003)

The narration LLM call uses `max_tokens=80` (a hard ceiling enforced by litellm — the model cannot exceed it). 80 tokens is enough for 2-3 short sentences of in-character voice; small enough that 3 narrations per hunt total ≤ 240 narration tokens (cheap relative to the chat round-trip's per-message envelope). The cap is asserted in tests by mocking the litellm call and validating `max_tokens` is set. If the model returns more than 80 tokens (impossible under litellm but defensive), the runtime path truncates to 80 tokens of text and logs a debug warning.

Backing context for the choice: F60 doesn't explicitly cap per-LLM-call tokens, but DEC-AGENT-HINTS-001 and the persona profiles (DEC-30-CHARACTER-V2-006: ≤ 165 tokens per profile) operate in the same "small enough to be cheap" range. 80 is half a persona profile; appropriate for celebration narration that should be lighter than persona priming.

Future loosening: a `CelebrationConfig.per_narration_token_cap: int = 80` field is **NOT** added in M-7. The cap is a module-level constant. If empirical use shows 80 is too tight (specific personas get clipped), a future slice promotes it to config. Today the constant suffices and matches the minimal-codebase principle (DEC-M7-CELEB-007 records the deferral).

### 2.7 Per-hunt narration budget — 3 narrations per hunt (DEC-M7-CELEB-004)

A single hunt typically produces 0-2 slot-fill events (the M-3 emission rules cap upward transitions per slot per hunt; with 9 slots a worst case is 9, but realistic hunts produce 1-3). The per-hunt budget is set to **3**: enough to narrate every realistic hunt's worth of high-weight transitions while bounding worst-case cost to ~240 narration tokens regardless of how many slot fills fire. When budget is exhausted, subsequent eligible events fall back to ASCII art (the existing `CelebrationEngine.celebrate(total)` output is already in the celebration string; the budget exhaustion path simply does not append a narration string for the additional event). The `HuntNarrationBudget` is a per-call counter (per `_execute_run_module` invocation), not a per-session counter; resets each hunt.

Future loosening: same as the token cap, this is a constant. A future slice can promote it to config if needed (DEC-M7-CELEB-007).

### 2.8 LLM failure mode — silent fallback to ASCII, loud test assertion (DEC-M7-CELEB-006)

Runtime path:
1. `narrate_celebration(...)` wraps the LLM call in a broad `try/except Exception`. On any exception (HTTP failure, JSON malformation, content-policy rejection, timeout), it returns `None`.
2. `narrate_celebration(...)` also returns `None` when the model returns a string that contains Rich markup characters (`[`, `]`, `{`, `}` outside whitespace contexts), trailing whitespace exceeds 2x cap, or stripped length is zero. Validation is intentionally narrow — the goal is "didn't produce a usable narration", not "produced something that could be a vulnerability".
3. The caller in `_execute_run_module` checks for `None` and skips the append. The pre-existing ASCII celebration string is unaffected — fallback is automatic by construction.

Test path:
1. A test fixture installs `narration_failure_hook = True` (a module-level test flag in `gamification/dossier_celebrations.py::_NARRATION_TESTING_RAISE_ON_FAILURE`). When set, `narrate_celebration` re-raises any underlying exception instead of returning None. This converts the silent runtime fallback into a loud test failure (Sacred Practice 5).
2. A second test asserts that mocking `runner.narrate` to raise `RuntimeError("simulated LLM outage")` produces a celebration string that contains the ASCII art but NOT a narration line — proves the production fallback path.

### 2.9 Dossier-aware badge set (DEC-M7-BADGE-001..005)

Five new badges. Each maps to a new `BadgeMetric` value and a new stat key computed by `build_dossier_stats(post_dossier, predictions)`. Existing 10 badges and their unchanged metrics stay byte-identical (additive only — DEC-M7-BADGE-006).

| Badge ID | Name | Rarity | Metric | Threshold | Rationale |
|----------|------|--------|--------|-----------|-----------|
| `badge-dossier-complete` | Full Dossier | LEGENDARY | `DOSSIER_SLOTS_FILLED` | 9 | All 9 slots reach FILLED status. The signature M-7 puzzle-solved achievement. |
| `badge-identity-first` | Identity First | RARE | `DOSSIER_IDENTITY_FIRST` | 1 | Identity slot reaches FILLED before any other slot does. Captured by `build_dossier_stats` checking: Identity is FILLED AND every other non-Identity, non-Deferred slot is EMPTY or PARTIAL — a snapshot heuristic. The exact "before any other" temporal semantic is approximated as "Identity is FILLED while at most one other slot is also FILLED" — a single check against the snapshot. (DEC-M7-BADGE-007 records this approximation explicitly.) |
| `badge-predictor` | Predictor | UNCOMMON | `DOSSIER_PREDICTIONS_VALIDATED` | 3 | Three or more `PersistedPrediction.status == "validated"` in the workspace's predictions log. Reads M-4/M-5 persistence directly. |
| `badge-skeptic` | Skeptic | UNCOMMON | `DOSSIER_PREDICTIONS_FALSIFIED` | 3 | Three or more `PersistedPrediction.status == "falsified"`. Reads M-5 active-falsification engine output (DEC-M4-PRED-006: falsification=+0 points; the badge is the prestige signal). |
| `badge-deception-spotter` | Deception Spotter | RARE | `DOSSIER_DENIAL_FILLED` | 1 | Denial slot reaches FILLED. Recognises that the analyst surfaced credible denial / deception evidence (M-5 slot 9 inference + analyst note authoring). |

Badge IDs are stable strings; rarity assignment matches the existing palette (LEGENDARY for the apex puzzle solve, RARE for the directed achievements, UNCOMMON for the moderate-tier validation/skepticism awards). Five was the dispatch context's "4–6 range"; five hits the sweet spot — one apex, two RARE, two UNCOMMON.

`build_dossier_stats(post_dossier, predictions)` produces:

```python
{
    "dossier_slots_filled": int,           # count of slots with status == FILLED
    "dossier_identity_first": int,         # 1 if Identity FILLED with ≤1 other FILLED
    "dossier_predictions_validated": int,  # count of validated predictions
    "dossier_predictions_falsified": int,  # count of falsified predictions
    "dossier_denial_filled": int,          # 1 if Denial FILLED
}
```

This dict is merged into the existing `workspace_stats` dict before `BadgeManager.check_all(badge_stats, already_awarded)` runs. The existing 10 badges read their old keys; the new 5 read the new keys. `Badge.check_award(workspace_stats)` semantics are preserved verbatim — `workspace_stats.get(metric.value, 0) >= threshold`.

### 2.10 Classic-style regression fixture (DEC-M7-REPORT-003)

A fixture file at `tests/fixtures/v1_classic_report.md` captures the byte-exact output of the v1 `ReportGenerator` for a known fabricated workspace shape. The fixture is generated **once** as part of the M-7 implementer commit (the test that asserts the fixture passes records the generating workspace shape inline in the test docstring — anyone can regenerate it by running the helper at the bottom of the test file). The test `test_classic_style_regression.py::test_report_style_classic_byte_identical_to_v1_fixture` calls `core.dossier_report._invoke_classic(workspace_mgr)` (a one-line wrapper that calls `ReportGenerator(workspace_mgr).generate()` to make the call site explicit and grep-friendly) and asserts byte-identical match against the fixture, modulo the dynamic timestamp line which the fixture has redacted to `**Date:** {DYNAMIC_DATE}` and the test substitutes via regex.

The fixture lives in `tests/fixtures/` (new directory). The implementer creates the directory + the fixture file + the one-line regenerate-helper test in the same commit as the source. Until M-8 removes the classic shim, this regression is the M-7-binding signal that the shim has not silently rotted.

### 2.11 New LLM tool — `generate_dossier_report` (DEC-M7-REPORT-005)

Tool count goes 30 → 31. The existing `generate_report` LLM tool is preserved verbatim for the classic interview shim (`start_report_interview` + `answer_report_question` + `generate_report` form the v1 trio). `generate_dossier_report` is a new third leg that produces the dossier report without an interview.

```python
{
    "type": "function",
    "function": {
        "name": "generate_dossier_report",
        "description": (
            "Generate the dossier-style investigation report as Markdown. "
            "Renders the current threat actor dossier state (9 slots), "
            "the predictions log (pending / validated / falsified), the "
            "analyst notes, and the workspace STIX summary. "
            "Optional 'style' parameter: 'dossier' (default, M-7 dossier-puzzle report) "
            "or 'classic' (v1 interview-based report — deprecated, removed in v0.3.0). "
            "Returns the complete Markdown report as a string. "
            "No interview required — the dossier state is the source of truth."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "style": {
                    "type": "string",
                    "enum": ["dossier", "classic"],
                    "description": (
                        "Report style. 'dossier' produces the M-7 actor-dossier report "
                        "(default). 'classic' produces the v1 interview-based report "
                        "(requires prior start_report_interview + answer_report_question "
                        "calls). 'classic' is deprecated and will be removed in v0.3.0."
                    ),
                    "default": "dossier",
                },
            },
            "required": [],
        },
    },
},
```

Per minimal-codebase principle, a third style is not introduced. The new tool surfaces the dossier report; the existing tools surface the classic flow.

### 2.12 Read-paths and integration surfaces

- **Dossier state authority:** `dossier/state.py::load_dossier_state(workspace_mgr)` (M-4) and `dossier/state.py::default_deferred_state()` (M-4 fallback). M-7 reads via these.
- **Predictions log authority:** `dossier/predictions.py::load_predictions_log(workspace_mgr)` (M-4). M-7 reads via this; never writes.
- **Slot weight authority:** `dossier/slots.py::SLOT_WEIGHTS` (M-1, DEC-M1-SLOTS-WEIGHT-AUTHORITY-001). Read-only consumer.
- **Slot status enum:** `dossier/slots.py::SlotStatus`. Read-only consumer.
- **Analyst note authority:** `models/database.py::AnalystNote` + `core/workspace.py::WorkspaceManager.add_note` (M-5 DEC-M5-NOTE-001). M-7 reads the rows via the same `SQLAlchemy select(AnalystNote)` pattern the existing `ReportGenerator._generate_analyst_notes` uses; never writes.
- **Workspace stats authority:** `core/workspace.py::WorkspaceManager.get_workspace_stats` (returns the existing dict). M-7 layers `build_dossier_stats` on top of the returned dict; never modifies the workspace method.
- **Score events:** `core/workspace.py::WorkspaceManager.store_score_events` (existing). M-7 does not call it. Existing `_execute_run_module` flow stores per-IOC + dossier slot-fill + streak events as today. M-7's narration consumes the in-memory `events` list at the same call site.
- **Badge persistence:** `core/workspace.py::WorkspaceManager.store_badge_event` (existing). Unchanged.
- **Badge sidecar:** `runner.last_badges` (existing). Unchanged.
- **Celebration sidecar:** `runner.last_celebrations` (existing). M-7 narration appends to the same `celebration` string that `_execute_run_module` already returns; rendered via the same `chat.py` celebration-panel loop.
- **F64 LLM-facing summary surface:** `agent/tools.py` builds `summary` via the `_DOSSIER_ACTIONS` filter (M-3/M-5). M-7 does NOT widen `_DOSSIER_ACTIONS` and does NOT pipe narration text into `summary` — narration is panel content per DEC-64-LLM-PANEL-SEPARATION-001.

---

## 3. Removal targets (no parallel-authority residue) and M-8 removal trail

M-7 is purely additive on the source surface — there is no parallel authority to delete now. The removal trail is queued for M-8:

- **Classic-report shim:** `core/report.py` (entire file), `agent/tools.py::_execute_start_report_interview` / `_execute_answer_report_question` / `_execute_generate_report` (all three private dispatchers and their tool entries), `agent/chat.py` `report answer <idx> <text>` and `report` (no-subcommand) interview-display block, `core/console.py::do_report` interview/show subcommands, `tests/test_report.py` interview-specific tests, `tests/fixtures/v1_classic_report.md` regression fixture. M-8 removes them together with the `--style classic` flag handling on all three surfaces.
- **`ReportGenerator` re-export from `core.report`:** scheduled for the same M-8 commit.

These removals are noted in MASTER_PLAN.md Phase 17J §M-7 Out-of-Scope and re-noted as the M-8 entry-point work.

There are no shadow LLM-narration mechanisms to retire. No prior slice has shipped narration; M-7 is the first.

There are no parallel badge metrics. The five new badges are net-new IDs.

---

## 4. The load-bearing acceptance tests

Three compound integration tests, one per sub-slice. Each lives in a NEW `tests/test_*.py` file.

### Stage A — reports (`tests/test_dossier_report.py`)

1. Fresh workspace. Fabricate a fixture state: Identity=FILLED (1 SCO), TTPs=PARTIAL (2 SCOs), Infrastructure=FILLED (3 SCOs), others EMPTY. Persist via `save_dossier_state`. Save two predictions: one validated (`status="validated"`), one pending (`status="pending"`). Add two analyst notes via `WorkspaceManager.add_note`.
2. Call `generate_dossier_report(workspace_mgr)` — assert the returned string contains:
   - section header "## Dossier State"
   - text "Identity: FILLED" with the contributing-types list
   - text "TTPs: PARTIAL"
   - text "Infrastructure: FILLED"
   - text "Timing: EMPTY" (or DEFERRED, depending on fixture)
   - section header "## Predictions"
   - the validated prediction's text
   - the pending prediction's text
   - section header "## Analyst Notes"
   - both note bodies in their authored order
   - the workspace metadata header (workspace name, date, total score)
   - the existing v1-style "## Indicators of Compromise" table (the IOC table is content-shared; no reason to drop it)
3. Call `_invoke_classic(workspace_mgr)` (the classic shim wrapper). Assert the returned string is byte-identical to the redacted fixture at `tests/fixtures/v1_classic_report.md` after the dynamic-date substitution.
4. Call `_execute_generate_dossier_report(ctx, style="dossier")` — assert it returns the dossier string from step 2. Call with `style="classic"` — assert it returns the classic string from step 3.
5. Invoke the cmd2 path: simulate `report --style classic generate` and `report generate` (default dossier). Assert the file saved to the workspace dir contains the matching content.

### Stage B — celebrations (`tests/test_dossier_celebrations.py`)

1. Fixture: a `ToolContext` with `_execute_run_module` patched at the events-construction boundary so the test feeds in a fabricated `events` list: one `dossier_slot_filled` for Identity (weight 5.0) at +5 points; one `dossier_slot_filled` for Infrastructure (weight 2.0) at +2 points; one `score_results_ipv4-addr` at +1 point. Total: +8 points.
2. Mock `runner.narrate(...)` to return `"Identity locked. The mask slipped — and you saw the face beneath."` deterministic 14-token string.
3. Call `_execute_run_module` through to the celebration-construction block. Assert the returned `celebration` string contains:
   - the ASCII art produced by `CelebrationEngine.celebrate(8)` (level=small per existing rules)
   - the narration text for the Identity event (one narration; high-weight)
   - NO narration text for the Infrastructure event (routine; weight 2.0 < 2.5)
   - NO narration text for the per-IOC event (never narrated)
4. Re-run with `runner.narrate` mocked to raise `RuntimeError("simulated outage")`. Assert the celebration string contains the ASCII art and NO narration line — fallback path proven.
5. Run with three Identity-tier events queued (Identity, Predictions, Capability, TTPs — four high-weight events). Assert exactly 3 narrations appear in the celebration string (the per-hunt budget cap); the 4th high-weight event falls back to ASCII (no narration line).
6. Loud-fail test: set the module flag `_NARRATION_TESTING_RAISE_ON_FAILURE = True`, mock `runner.narrate` to raise — assert that calling `narrate_celebration(...)` raises `RuntimeError`, not silently returns None.
7. Token cap test: inspect the `litellm.completion` mock — assert `max_tokens == 80` on every narration call.
8. F64 invariance: assert the `summary` string returned by `_execute_run_module` contains NO narration text (DEC-64-LLM-PANEL-SEPARATION-001 preserved; narration is panel content).

### Stage C — badges (`tests/test_dossier_badges.py`)

1. Fresh workspace. Save a `DossierState` where all 9 slots are FILLED. Save 3 validated predictions, 0 falsified, Denial FILLED.
2. Call `build_dossier_stats(post_dossier, predictions)` — assert it returns `{"dossier_slots_filled": 9, "dossier_identity_first": 0 (Identity is FILLED but multiple others are too), "dossier_predictions_validated": 3, "dossier_predictions_falsified": 0, "dossier_denial_filled": 1}`.
3. Merge into `workspace_stats` dict; call `BadgeManager.check_all(merged_stats, already_awarded=set())`. Assert four new badges fire: `badge-dossier-complete`, `badge-predictor`, `badge-deception-spotter`, plus the existing badges that the fixture workspace meets — at minimum verify `badge-dossier-complete` is in the returned list.
4. Identity-first scenario: Identity=FILLED, all other slots EMPTY. `dossier_identity_first` returns 1; `badge-identity-first` fires.
5. Skeptic scenario: save 3 falsified predictions. `dossier_predictions_falsified` returns 3; `badge-skeptic` fires.
6. Existing-badge regression: a workspace state that meets the existing `badge-data-hoarder` threshold (1000 total_indicators) still fires `badge-data-hoarder` exactly as before. Existing badge IDs and thresholds byte-identical (DEC-M7-BADGE-006).
7. Tool count audit: count `"name": "` entries in `agent/tools.py::create_tools` returned list. Assert exactly 31.

### Stage D — integrated demo trace (manual evidence under `tmp/evidence-m7-reports-celebrations/`)

Implementer captures the following live evidence:
1. `pytest_dossier_report.txt`, `pytest_dossier_celebrations.txt`, `pytest_dossier_badges.txt` from running the new test files.
2. `report_classic_byte_match.txt`: `diff` output between live `_invoke_classic` and the fixture (empty diff modulo timestamp).
3. `narration_smoke.txt`: a real `ap chat` run against a mocked LLM that fires one Identity slot-fill event and shows the narration text in the Rich celebration panel (PTY capture or saved transcript).
4. `tool_count_31.txt`: output of `python -c "from adversary_pursuit.agent.tools import create_tools; from adversary_pursuit.agent.tools import ToolContext; ctx=...; print(len(create_tools(ctx)))"` showing 31.
5. `full_suite.txt`: `pytest -q` showing baseline ≥ M-6 + new M-7 tests, all green.
6. `ruff_clean.txt`: `ruff check src/ tests/` clean on touched files.

---

## 5. F64 invariance — re-stated and tested

DEC-64-LLM-PANEL-SEPARATION-001 is the binding constraint M-7 must not perturb. The contract:

- LLM-facing `summary` string contains findings only — no gamification narrative, no badge text, no challenge text, no celebration text.
- Rich panel surface (the `chat.py` celebration-panel / badge-panel / challenge-panel loop) renders gamification artifacts only.
- Per the sidecar pattern: `result["celebration"]`, `result["badges"]`, `result["challenges"]` are the sole conduits between the tool result and the panel renderer.

M-7 wires narration into `result["celebration"]` — the same field that already carries ASCII art and milestone messages today. Narration text rides the celebration sidecar, gets stored in `runner.last_celebrations`, and surfaces via `chat.py`'s existing celebration-panel loop. Nothing reads `summary` for narration content. The `_DOSSIER_ACTIONS` filter stays a 3-tuple (no widening).

Tests that prove the invariant:
- Stage B step 8 above asserts `"Identity locked"` (the narration text) is absent from the LLM-facing `summary`.
- A negative-test pattern: build a hunt that fires Identity narration, run through to the chat surface (with a mocked LLM), assert the LLM sees `summary` without narration text, the Rich panel sees narration text.

---

## 6. Evaluation Contract

See per-Phase 17J section for the legal-key JSON shape; this is the narrative form.

- **required_tests:** ~35 new tests across:
  - `tests/test_dossier_report.py` (NEW, ~12 tests): Stage A coverage + IOC table preserved + analyst notes preserved + predictions block + style switch + cmd2 path + chat path + classic byte-identity.
  - `tests/test_dossier_celebrations.py` (NEW, ~10 tests): Stage B coverage + threshold + token cap + budget + fallback + loud-fail flag + F64 invariance + per-IOC exclusion + falsified exclusion + persona-voice carrier (system prompt reuse).
  - `tests/test_dossier_badges.py` (NEW, ~9 tests): Stage C coverage + each new badge fires + existing badges byte-identical + tool count 31 + stats dict shape + identity-first approximation.
  - `tests/test_classic_style_regression.py` (NEW, ~2 tests): byte-identity to fixture + dynamic-date substitution.
  - `tests/test_agent_tools.py` (extend, ~3 tests): `_execute_generate_dossier_report` dispatcher + tool-count audit + narration call-site integration with mocked LLM.
  - `tests/test_chat_report_metacommand.py` (NEW or extend existing chat tests, ~3 tests): `report --style classic` parsing + bare `report generate` default routing + LLM tool tool count.
  - `tests/test_badges.py` (extend, ~2 tests): new `BadgeMetric` enum members defined + `_DEFAULT_BADGES` list has 15 entries.
  - `tests/test_report.py` (extend, ~2 tests): v1 interview path still works under classic style.

  Full suite green ≥ M-6 baseline + new M-7 tests.

- **required_evidence:** full `pytest -q` output green; `git diff main -- src/adversary_pursuit/core/workspace.py` empty; `git diff main -- src/adversary_pursuit/models/database.py` empty; `git diff main -- src/adversary_pursuit/dossier/state.py` empty; `git diff main -- src/adversary_pursuit/dossier/predictions.py` empty; `git diff main -- src/adversary_pursuit/dossier/scoring.py` empty; `git diff main -- src/adversary_pursuit/dossier/slot_inference.py` empty; `git diff main -- src/adversary_pursuit/dossier/slots.py` empty; `git diff main -- src/adversary_pursuit/dossier/panel.py` empty; `git diff main -- src/adversary_pursuit/dossier/__init__.py` empty; `git diff main -- src/adversary_pursuit/core/pivot_policy.py` empty; `git diff main -- src/adversary_pursuit/core/event_bus.py` empty; `git diff main -- src/adversary_pursuit/core/dossier_pivot.py` empty; `git diff main -- src/adversary_pursuit/core/streak.py` empty; `git diff main -- src/adversary_pursuit/core/config.py` empty; `git diff main -- src/adversary_pursuit/gamification/scoring.py` empty; `git diff main -- src/adversary_pursuit/gamification/modes.py` empty; `git diff main -- src/adversary_pursuit/gamification/hints.py` empty; `git diff main -- src/adversary_pursuit/gamification/challenges.py` empty; `git diff main -- src/adversary_pursuit/gamification/celebrations.py` empty; `git diff main -- src/adversary_pursuit/core/report.py` empty (M-7 keeps it byte-identical; M-8 deletes it); tool-count audit at exactly 31; demo trace showing §4 Stage A/B/C/D acceptance — dossier-default report, narration on high-weight event, ASCII fallback on routine event, all 5 new badges fire, classic-style byte-matches fixture.

- **required_authority_invariants:**
  - F59 (`core/workspace.py` BYTEWISE UNCHANGED).
  - F60 (`core/pivot_policy.py::PivotPolicy` BYTEWISE UNCHANGED; `event_bus.py` BYTEWISE UNCHANGED; no new gate; no new ranker).
  - F62 (`core/streak.py` BYTEWISE UNCHANGED; no new score events).
  - F63 (`celebrations.py` BYTEWISE UNCHANGED; `MILESTONES` unchanged; `check_milestones` unchanged; M-7 layers narration on top of the existing celebration string at the call site, not inside the engine).
  - F64 (`_DOSSIER_ACTIONS` filter UNCHANGED; narration rides celebration sidecar, not LLM summary; tests assert summary contains no narration text — DEC-64-LLM-PANEL-SEPARATION-001 preserved).
  - Sacred Practice 12 (single source of truth: report renderer = `core/dossier_report.py`; narration policy = `gamification/dossier_celebrations.py`; new badge specs = `gamification/dossier_badges.py`; existing 10 badges in `_DEFAULT_BADGES` byte-identical).
  - DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 (`SLOT_WEIGHTS` read-only consumer).
  - DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006 (read-only consumer of `load_dossier_state` + `load_predictions_log`).
  - DEC-M5-DENIAL-001..003 + DEC-M5-NOTE-001..003 + DEC-M5-FALSIFY-001..008 (read-only consumer of slot 9 + AnalystNote rows + falsification engine output).
  - DEC-M6-PIVOT-001..009 (ranker layer untouched — M-7 does not extend it).
  - DEC-68-DOSSIER-REFRAME-006 (M-7 absorbs issue #32; honored).
  - DEC-68-DOSSIER-REFRAME-008 (classic shim preserved one release cycle; removed at M-8).
  - DEC-BADGE-001..003 (badge stats-dict contract + metric enum pattern + stateless manager preserved).
  - DEC-CELEBRATION-001 (four-level ASCII art preserved verbatim for routine events).
  - DEC-63-MILESTONE-CATCHUP-001 (milestone catch-up path untouched).
  - DEC-REPORT-001..003 (v1 report preserved verbatim via classic shim; one-release deprecation runway).

- **required_integration_points:** `core/dossier_report.py` (NEW); `gamification/dossier_celebrations.py` (NEW); `gamification/dossier_badges.py` (NEW); `gamification/badges.py` (EXTEND: `BadgeMetric` + `_DEFAULT_BADGES`); `agent/runner.py` (EXTEND: `narrate` method only); `agent/tools.py` (EXTEND: narration loop + dossier-stats merge + new `generate_dossier_report` tool entry + new `_execute_generate_dossier_report` dispatcher); `core/console.py` (EXTEND: `do_report` `--style` parser); `agent/chat.py` (EXTEND: `report` meta-command `--style classic` recognizer); `tests/fixtures/v1_classic_report.md` (NEW); test files per §6.

- **forbidden_shortcuts:**
  - no `core/workspace.py` modification
  - no `models/database.py` modification
  - no `dossier/*.py` modification (six files: state, predictions, scoring, slot_inference, panel, slots, plus `__init__.py`)
  - no `core/pivot_policy.py` / `core/event_bus.py` / `core/dossier_pivot.py` modification
  - no `core/config.py` modification (token cap, budget, threshold all module-level constants in `dossier_celebrations.py`; no config field)
  - no `core/streak.py` modification
  - no `gamification/scoring.py` / `gamification/modes.py` / `gamification/hints.py` / `gamification/challenges.py` / `gamification/celebrations.py` modification
  - no `core/report.py` modification (it IS the classic shim; M-8 removes it)
  - no `_DOSSIER_ACTIONS` widening (narration is sidecar content, not summary content)
  - no new ScoreEvent action / subtype
  - no new event-bus subscriber
  - no parallel litellm client (narration uses `AgentRunner.narrate` helper that reuses the existing client)
  - no PDF/HTML/JSON report format
  - no new score points for narration (narration is celebration text, not a score event)
  - no narration of `dossier_prediction_falsified` events (Skeptic badge is the recognition surface, per DEC-M7-CELEB-005)
  - no narration in cmd2 console path (no LLM client at that surface)
  - no third style flag beyond `dossier` / `classic`
  - no Rich markup in narration text (validated and stripped)
  - no LLM call that mutates `runner.conversation`
  - no caching of narration across hunts (per-hunt budget object only)
  - no badge persistence change (`store_badge_event` unchanged)
  - no badge sidecar contract change (`runner.last_badges` shape unchanged)
  - no refactor beyond documented additions
  - no removal of any existing 10 badges; no rename; no threshold change

- **rollback_boundary:** single feature branch revertible as one merge commit. Revert removes `core/dossier_report.py`, `gamification/dossier_celebrations.py`, `gamification/dossier_badges.py`, the five new `BadgeMetric` enum members, the five new entries in `_DEFAULT_BADGES`, the `AgentRunner.narrate` method, the narration loop and dossier-stats merge in `_execute_run_module`, the `generate_dossier_report` tool entry and `_execute_generate_dossier_report` dispatcher, the `--style` parser additions in `do_report` and the `report` meta-command, the `tests/fixtures/v1_classic_report.md` file. Restores M-6 byte state. M-7 ships NO new persistence, NO new schema, NO new event types. Workspace TOML / config files unaffected. Badge events already stored under a new badge ID after revert decode fine because `BadgeManager.get_badge(unknown_id)` returns None; rendering side gracefully falls through (existing badge-id-unknown handling). The five new badge IDs in `_DEFAULT_BADGES` are removed by the revert so no future awarding fires for them — pre-revert awarded events stay as historical workspace records. No data migration required.

- **ready_for_guardian_definition:** all required_tests green; full suite green ≥ M-6 baseline + new M-7 tests; forbidden-file `git diff main` outputs empty (paste each); `core/workspace.py::WorkspaceManager` diff empty; `gamification/celebrations.py::CelebrationEngine` diff empty; `gamification/badges.py` diff limited to `BadgeMetric` enum extension + `_DEFAULT_BADGES` extension; `agent/runner.py` diff limited to `narrate` method; `agent/tools.py` diff limited to narration loop + dossier-stats merge + `generate_dossier_report` tool entry + `_execute_generate_dossier_report` dispatcher; `core/console.py` diff limited to `--style` parsing in `do_report`; `agent/chat.py` diff limited to `--style` parsing in `report` meta-command; tool count audit at exactly 31; **Phase 17J appended to MASTER_PLAN.md AND committed in the same commit as source by the IMPLEMENTER** (AP #74 orphan-prevention applies to source-side edits); Phase 17I status flipped in-progress → completed in the same commit as the planner-staged section, recorded with M-6 merge SHA and impl SHA filled in once M-6 lands; Active Phase Pointer tail-line re-pointed from `W-68-M6-DOSSIER-PIVOT` to `W-68-M7-REPORTS-CELEBRATIONS`; implementer commit message follows `feat(dossier-reports):` or `feat(dossier-m7):` prefix and references `#68` + `#32` + `DEC-M7-REPORT-001..005` + `DEC-M7-CELEB-001..007` + `DEC-M7-BADGE-001..007`.

---

## 7. Scope Manifest (full)

See `tmp/m7-scope.json` for the canonical CLI-key JSON shape.

**Allowed / Required (the implementer MUST touch these):**

- `src/adversary_pursuit/core/dossier_report.py` (NEW: `generate_dossier_report` + private renderers + `_invoke_classic` shim)
- `src/adversary_pursuit/gamification/dossier_celebrations.py` (NEW: thresholds + `narrate_celebration` + `HuntNarrationBudget` + `is_high_weight_event` + `build_narration_prompt`)
- `src/adversary_pursuit/gamification/dossier_badges.py` (NEW: `DOSSIER_BADGES` + `build_dossier_stats`)
- `src/adversary_pursuit/gamification/badges.py` (EXTEND, narrow: `BadgeMetric` enum gains 5 new members; `_DEFAULT_BADGES` list extended with `DOSSIER_BADGES`)
- `src/adversary_pursuit/agent/runner.py` (EXTEND, narrow: new `narrate(prompt, *, max_tokens) -> str | None` method only)
- `src/adversary_pursuit/agent/tools.py` (EXTEND: narration loop in `_execute_run_module` + dossier-stats merge before badge check + new `generate_dossier_report` tool entry in `create_tools` + new `_execute_generate_dossier_report` dispatcher + tool-dispatch routing entry)
- `src/adversary_pursuit/core/console.py` (EXTEND, narrow: `do_report` `--style {dossier,classic}` parser + `_report_generate` / `_report_show` style switch)
- `src/adversary_pursuit/agent/chat.py` (EXTEND, narrow: `report` meta-command `--style classic` recognizer)
- `tests/test_dossier_report.py` (NEW — Stage A)
- `tests/test_dossier_celebrations.py` (NEW — Stage B)
- `tests/test_dossier_badges.py` (NEW — Stage C)
- `tests/test_classic_style_regression.py` (NEW — fixture byte-identity)
- `tests/test_agent_tools.py` (extend — dispatcher + tool-count audit + narration call-site)
- `tests/test_badges.py` (extend — new enum + list size)
- `tests/test_report.py` (extend — classic-style preservation under flag)
- `tests/test_chat_report_metacommand.py` (NEW or extend if a chat-report meta-command test file exists; if not, NEW)
- `tests/fixtures/v1_classic_report.md` (NEW directory + NEW fixture file — generator helper inline in `test_classic_style_regression.py`)
- `MASTER_PLAN.md` — Phase 17J section authored by the planner stage; Phase 17I status flip from in-progress → completed (filled with M-6's merge SHA + impl SHA once M-6 lands); Plan Status table row added; Active Phase Pointer tail-line re-pointed. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention).**
- `.claude/plans/dossier-m7-reports-celebrations.md` — THIS FILE. Planner stage commits it; implementer rebases on it.
- `tmp/m7-scope.json` — canonical scope JSON for runtime scope-sync.

**Forbidden (preserved authorities):**

- `src/adversary_pursuit/core/workspace.py` (F59 BYTEWISE UNCHANGED; stronger than M-4's narrow-edit clause)
- `src/adversary_pursuit/models/database.py` (DEC-DB-002 preserved; no schema change)
- `src/adversary_pursuit/core/report.py` (BYTEWISE UNCHANGED in M-7; classic shim verbatim; M-8 deletes)
- `src/adversary_pursuit/core/pivot_policy.py` (F60 invariant)
- `src/adversary_pursuit/core/event_bus.py` (F60 invariant)
- `src/adversary_pursuit/core/dossier_pivot.py` (M-6 byte-identical)
- `src/adversary_pursuit/core/config.py` (no new config field — thresholds are module-level constants)
- `src/adversary_pursuit/core/streak.py` (F62 invariant)
- `src/adversary_pursuit/dossier/slots.py` (DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 preserved; SLOT_WEIGHTS read-only consumer)
- `src/adversary_pursuit/dossier/state.py` (M-4 byte-identical; read-only consumer of `load_dossier_state`)
- `src/adversary_pursuit/dossier/predictions.py` (M-5 byte-identical; read-only consumer of `load_predictions_log`)
- `src/adversary_pursuit/dossier/scoring.py` (M-5 byte-identical; M-7 does NOT add new event subtype)
- `src/adversary_pursuit/dossier/slot_inference.py` (M-5 byte-identical)
- `src/adversary_pursuit/dossier/panel.py` (M-1 byte-identical)
- `src/adversary_pursuit/dossier/__init__.py` (no new exports — the renderer lives in `core/`)
- `src/adversary_pursuit/gamification/scoring.py` (M-3 byte-identical)
- `src/adversary_pursuit/gamification/celebrations.py` (F63 invariant; CelebrationEngine + MILESTONES + CELEBRATION_ART byte-identical)
- `src/adversary_pursuit/gamification/modes.py` (no persona changes)
- `src/adversary_pursuit/gamification/hints.py` (no surface changes)
- `src/adversary_pursuit/gamification/challenges.py` (no surface changes)
- `src/adversary_pursuit/modules/**` (no module changes)
- `pyproject.toml`, `CLAUDE.md`, `AGENTS.md`, `settings.json`, hooks, `runtime/`, `agents/` (no governance / harness changes)

**Authority domains touched:**

- `dossier_report_in_memory` (NEW — `core/dossier_report.py` renderer)
- `dossier_narration_policy` (NEW — `gamification/dossier_celebrations.py` thresholds + budget)
- `dossier_badges_catalog` (NEW — `gamification/dossier_badges.py` + extended `_DEFAULT_BADGES`)
- `agent_runner_narration_helper` (NEW — `AgentRunner.narrate`)
- `report_style_flag` (NEW — `--style` parsing on cmd2 + chat + LLM tool)
- `llm_tool_catalog` (`generate_dossier_report` added; tool count 30 → 31)
- `classic_report_fixture` (`tests/fixtures/v1_classic_report.md`)

---

## 8. Decision Log

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M7-REPORT-001** | Report style flag is a two-value enum: `--style dossier` (default) and `--style classic` (deprecated shim, removed at M-8). No third style introduced in M-7. | Smallest abstraction matching DEC-68-DOSSIER-REFRAME-008's one-release deprecation runway. A third value would be speculative — no user has asked for minimal / markdown-only output. Minimal-codebase principle: do not introduce abstractions without need. |
| **DEC-M7-REPORT-002** | The dossier report renderer lives in NEW `core/dossier_report.py`, not in `core/report.py` and not as a method on `ReportGenerator`. | Separation of concerns: `core/report.py` is the interview-driven v1 report; `core/dossier_report.py` is the dossier-puzzle report. The two are different shapes, not variants of one template. M-8 removes `core/report.py` cleanly (whole file) while `core/dossier_report.py` stands on its own — clean removal trail. Mirrors M-6's `core/dossier_pivot.py` vs `core/pivot_policy.py` separation. |
| **DEC-M7-REPORT-003** | Classic-style regression fixture lives at `tests/fixtures/v1_classic_report.md`. Implementer generates the fixture in the same commit. The byte-comparison test redacts the dynamic-date line via regex; everything else is byte-identical. | Until M-8 deletes the classic shim, the regression must stay green. A fixture under version control is the cheapest signal that no one silently rotted the shim (e.g., via a refactor of `ReportGenerator.generate`). The test helper that regenerates the fixture lives in the same test file so any v1-path-byte-affecting change is detected immediately. |
| **DEC-M7-REPORT-004** | The IOC table, Metadata, Timeline, and Analyst Notes sections from the v1 report are PRESERVED in the dossier report (they carry workspace facts that are also dossier facts — no reason to drop them). The dossier report ADDS: Dossier State (slot status), Predictions (pending / validated / falsified), Top Findings (per-slot evidence summary). The Interview Notes section from the v1 report is OMITTED from the dossier report (no interview to capture). | Reports are user-facing artifacts; the dossier report is a superset of the workspace facts the v1 report rendered (minus the interview). Discarding the IOC table / Timeline / Analyst Notes would force users to compare two reports to see all the facts — friction without value. M-7's contribution is the dossier-aware sections, not a rewrite of the workspace-facts sections. |
| **DEC-M7-REPORT-005** | One new LLM tool: `generate_dossier_report` with optional `style` parameter (`"dossier"` default, `"classic"` legacy). Tool count 30 → 31. Existing `start_report_interview` + `answer_report_question` + `generate_report` tools preserved verbatim for the classic interview path (removed by M-8). | The dossier report needs an LLM-discoverable entry point. Adding a new tool with explicit dossier semantics is cleaner than overloading `generate_report` with a `style` parameter — LLMs reason about tools by name, and `generate_dossier_report` is the discoverable shape. The existing trio stays so the classic interview path continues to work via tools, not just via cmd2 / chat meta-commands. M-8 removes the trio together with `core/report.py`. |
| **DEC-M7-CELEB-001** | LLM-narrated celebrations call into `AgentRunner.narrate(prompt, *, max_tokens) -> str | None`, a small helper that reuses the active model, system prompt (persona-flavored), and API-key resolution. `gamification/dossier_celebrations.py` is the policy layer (threshold, budget, prompt construction, fallback) and calls `runner.narrate` for the LLM round-trip. | Single source of truth for the LLM client lives in `AgentRunner` already. A parallel client in `dossier_celebrations.py` would (i) duplicate API-key resolution, (ii) force persona injection elsewhere, (iii) violate Sacred Practice 12. The active persona's voice flows through naturally because `runner.narrate` reuses `self.system_prompt` (which `set_character` already mutated). |
| **DEC-M7-CELEB-002** | High-weight threshold for LLM narration is `slot_weight >= 2.5`. Captures Identity (5.0) / Predictions (4.0) / Capability (3.5) / TTPs (3.0) / Motivation (3.0) / Targeting (2.5) / Denial (2.5). Routine (no narration; ASCII only): Infrastructure (2.0) / Timing (2.0). Per-IOC events (action prefix `score_results_*`) and `dossier_prediction_falsified` events (DEC-M7-CELEB-005) are always routine. | The cut matches the natural break in `SLOT_WEIGHTS` between the "downstream-derivable" tier (2.5) and the "baseline-above-routine" tier (2.0). It also distinguishes puzzle-keystone slots (analyst signal that meaningfully advances the dossier) from ambient enrichment slots. Threshold lives as a module-level constant `HIGH_WEIGHT_NARRATION_THRESHOLD: float = 2.5` in `gamification/dossier_celebrations.py`. |
| **DEC-M7-CELEB-003** | Per-narration token cap is `max_tokens=80` (a hard ceiling at the litellm call). Module-level constant `PER_NARRATION_TOKEN_CAP: int = 80`. Test asserts the value reaches `litellm.completion`. | 80 tokens fits 2-3 short sentences of in-character voice. Cheap compared to the chat round-trip envelope. Aligns with the "small enough to be cheap" envelope F60 + DEC-AGENT-HINTS-001 + DEC-30-CHARACTER-V2-006 work in. Hardcoded as a constant (not config) in M-7 per the minimal-codebase principle; promotion to config is a future-slice concern (DEC-M7-CELEB-007). |
| **DEC-M7-CELEB-004** | Per-hunt narration budget is `3` narrations. Module-level constant `PER_HUNT_NARRATION_BUDGET: int = 3`. Tracked via a per-hunt `HuntNarrationBudget` dataclass counter that resets each `_execute_run_module` invocation. After exhaustion, remaining eligible events fall back to ASCII (existing celebration string suffices). | A single hunt typically fires 0–2 slot-fill events; 3 covers realistic worst cases while bounding cost. Per-hunt scope keeps the budget contract simple (no cross-session bookkeeping; no persistence). |
| **DEC-M7-CELEB-005** | Narration eligibility is `(action in {"dossier_slot_filled", "dossier_prediction_validated"}) AND (slot_weight >= 2.5)`. `dossier_prediction_falsified` events are NEVER narrated. | Falsification at points=0 (DEC-M4-PRED-006) is a contradiction signal, not an achievement. Narrating it as a celebration would be tone-mismatched. The Skeptic badge (DEC-M7-BADGE-004) is the prestige surface for falsifications. |
| **DEC-M7-CELEB-006** | LLM failure mode: silent fallback to ASCII art in runtime; loud-fail in tests via a module flag (`_NARRATION_TESTING_RAISE_ON_FAILURE`). `narrate_celebration` returns `None` on any exception / malformed output (Rich markup characters / oversize); the caller skips the append. Existing ASCII art remains in the celebration string. A second narrow validation rejects responses containing Rich markup characters (`[`, `]`, `{`, `}`) outside whitespace contexts. | Sacred Practice 5: loud failure in tests, controlled silence in runtime so a single LLM outage does not block the hunt. Rich-markup validation defends the panel surface from accidental string corruption. |
| **DEC-M7-CELEB-007** | Token cap / budget / threshold are MODULE-LEVEL CONSTANTS in `gamification/dossier_celebrations.py`, NOT fields on `CelebrationConfig` or any config submodel. No `core/config.py` modification in M-7. | Minimal-codebase principle: today there is no user / analyst case to tune these. Adding three config fields speculatively bloats the TOML surface and the doc. Promotion to config can land as a single-DEC future slice the day an actual tuning need surfaces. |
| **DEC-M7-BADGE-001** | New badge `badge-dossier-complete` (Full Dossier, LEGENDARY). Metric `DOSSIER_SLOTS_FILLED`. Threshold 9. Fires when all 9 dossier slots reach status FILLED. | The signature M-7 puzzle-solved achievement. LEGENDARY matches `badge-supreme-hunter` tier — the apex prestige slot. |
| **DEC-M7-BADGE-002** | New badge `badge-identity-first` (Identity First, RARE). Metric `DOSSIER_IDENTITY_FIRST`. Threshold 1. Fires when Identity is FILLED AND at most one other slot is FILLED. Snapshot heuristic — see DEC-M7-BADGE-007. | Recognises the "lead with attribution" play pattern: an analyst who locks Identity early is doing the most valuable work first. The snapshot heuristic is a known approximation; a true "before any other" temporal semantic would require event-time tracking (deferred — out of M-7 scope). |
| **DEC-M7-BADGE-003** | New badge `badge-predictor` (Predictor, UNCOMMON). Metric `DOSSIER_PREDICTIONS_VALIDATED`. Threshold 3. Fires when ≥3 predictions in the workspace's persistent log have status `"validated"`. | Three validated predictions is a meaningful tier — above one-off luck, below mastery. UNCOMMON matches the existing UNCOMMON tier (Pivot Master, Note Taker). |
| **DEC-M7-BADGE-004** | New badge `badge-skeptic` (Skeptic, UNCOMMON). Metric `DOSSIER_PREDICTIONS_FALSIFIED`. Threshold 3. Fires when ≥3 predictions have status `"falsified"`. | Reads M-5's active-falsification engine output. Recognises the analyst surfacing contradictions, a separate-but-equally-valuable skill from prediction. Three falsifications matches the predictor threshold for symmetry. |
| **DEC-M7-BADGE-005** | New badge `badge-deception-spotter` (Deception Spotter, RARE). Metric `DOSSIER_DENIAL_FILLED`. Threshold 1. Fires when Denial (slot 9) reaches status FILLED. | Slot 9 is the M-5 deception/denial-strategy slot. Filling it is a specific analytic achievement (DGA shape + fast-flux TTL + denial-keyword notes). RARE matches the "directed achievement" tier. |
| **DEC-M7-BADGE-006** | Existing 10 badges in `_DEFAULT_BADGES` are BYTEWISE PRESERVED. M-7 is additive only on the badge surface. | DEC-BADGE-001..003 preserved by construction; existing badge IDs, names, thresholds, and rarities unchanged. Five new badges extend the list — no rename, no threshold change, no rarity demotion. |
| **DEC-M7-BADGE-007** | Identity-first detection is a SNAPSHOT HEURISTIC: `dossier_identity_first = 1` iff Identity is FILLED AND (count of other non-Deferred slots with status FILLED) ≤ 1. The true temporal "Identity was the first slot to FILL" semantic would require event-time tracking and an Identity-first-fire badge on the event itself. M-7 uses the snapshot heuristic; the badge fires the first time `build_dossier_stats` returns 1 (idempotent via `BadgeManager` already-awarded set). | A true temporal detector would require tracking the order of slot-fill events across hunts — non-trivial state. The snapshot heuristic catches the realistic case (analyst leads with attribution) and accepts a known false-positive boundary case (analyst fills two slots in the same hunt and Identity is one of them). The approximation is explicit so future slices can promote to a real temporal detector if the false-positive rate matters. |

---

## 9. Subsequent workflow cue

After M-7 lands, the recommended next workflow is **M-8 — Cleanup, Closeout, and Novel-Method Achievement** per `.claude/plans/dossier-reframe-v2-roadmap.md` §M-8. M-8 removes the classic-report shim (`core/report.py` + the three classic LLM tools + the `--style classic` flag everywhere it appears + the `tests/fixtures/v1_classic_report.md` fixture + the byte-identity regression test) and closes the v0.3.x dossier roadmap. C-3 (Philosophy + Bureaucratese modes) remains independent and may schedule in parallel with M-8.

---

## 10. Risks and open follow-ups

**Risks:**

- **LLM cost.** Per-hunt cap of 3 narrations × 80 tokens = 240 narration tokens per hunt. Negligible against the chat round-trip envelope; explicitly bounded.
- **LLM failure rate.** Silent fallback to ASCII is the runtime path. Test path (Stage B step 4) asserts the fallback. A spike in failures would not be visible from the user surface — but the score event row that fired the narration is still persisted, so dossier-state continues to advance even if narration text is missing.
- **Fixture drift.** If a refactor of `ReportGenerator.generate()` accidentally changes the byte output of the v1 report, the byte-identity regression test catches it. The implementer commits the fixture inline with the source so future implementers see the gate.
- **Identity-first false positives.** The snapshot heuristic (DEC-M7-BADGE-007) can fire when two slots fill in the same hunt and one is Identity. Accepted boundary case; documented; promotion path to true temporal detector noted.
- **Persona-voice drift.** Because the narration call reuses the active system prompt, a persona that's been swapped mid-hunt could fire a narration with the prior voice. Mitigated by the fact that personas don't swap mid-hunt in practice (mode swap is a top-level chat meta-command). Not a v0.2.x correctness risk.

**Open follow-ups (out-of-scope for M-7):**

- **PDF / HTML output formats** — deferred to a post-M-8 slice. DEC-REPORT-002 (Markdown-first) is preserved through M-7; a future slice can layer rendering on top.
- **`CelebrationConfig.per_narration_token_cap` / `per_hunt_narration_budget` / `narration_threshold` config fields** — deferred per DEC-M7-CELEB-007. Add when a user case surfaces.
- **True temporal Identity-First detector** — deferred per DEC-M7-BADGE-007. Add when the snapshot false-positive rate matters.
- **Catch-up narration for pre-M-7 score events** — explicitly out (mirrors F63 quiet-start migration).
- **Persona-aware narration prompt templates** — DEC-M7-CELEB-001 uses the persona system prompt to flavor voice; a future slice can add per-persona prompt templates if voice quality varies.

---

## 11. Notes for the implementer

- **No source code on the orchestrator's behalf.** This file is planner-staged content. The implementer reads it, the implementer commits the source against it, the implementer flips the Phase 17J status from in-progress to completed in MASTER_PLAN.md when committing source (AP #74 orphan-prevention).
- **Honor the M-6 base.** Rebase on whatever main HEAD contains M-6's merge. M-7 does not touch M-6's surfaces; the integration site for narration is `_execute_run_module`'s celebration-construction block, which M-6 does not edit.
- **Read-paths only.** `WorkspaceManager`, `DossierState`, `PersistedPrediction`, `AnalystNote` — every reference is read. No `add_note`, no `store_score_events`, no `save_dossier_state`, no `store_badge_event` (the existing event-storage call sites are unchanged; M-7 does not introduce new ones).
- **Tool count audit.** Implementer must add a test that counts `"name": "` occurrences in the tool list returned by `create_tools(ctx)` and asserts the count is exactly 31.
- **Phase 17H status.** When the M-7 implementer commits, Phase 17H is already completed (M-5 landed at `e29e8b1` / `c5dd6bf`). Phase 17I (M-6) may be either in-progress or completed at commit time depending on landing order. If M-6 has landed, fill its merge + impl SHAs in Phase 17I's status. If M-6 is still in flight, leave Phase 17I as in-progress with the SHAs marked TBD — M-7 must not pre-emptively close M-6.
- **Active Phase Pointer.** Re-point the tail-line in MASTER_PLAN.md from `W-68-M6-DOSSIER-PIVOT` to `W-68-M7-REPORTS-CELEBRATIONS`. The tail-line position requirement (last `**Phase ...` boldline) holds — keep the section at the end of the doc.
- **Commit message prefix.** `feat(dossier-reports):` or `feat(dossier-m7):` — either is acceptable. Body references `#68` + `#32` + the DEC range `DEC-M7-REPORT-001..005 / DEC-M7-CELEB-001..007 / DEC-M7-BADGE-001..007`.
