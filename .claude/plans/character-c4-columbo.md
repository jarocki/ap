# C-4 — Columbo + Dossier-Aware context_hooks + Tier-1 Voice Modes RETIRE + mastery_level RETIRE — per-slice plan

**Status:** planner-staged 2026-06-09 by W-30-C4-COLUMBO planner stage. Implementer slice `wi-30-c4-impl-01` to follow.
**Workflow:** `w-30-c4-columbo`
**Goal:** `g-30-c4-columbo`
**Work item to dispatch:** `wi-30-c4-impl-01`
**Drives:** Phase 17M of `MASTER_PLAN.md`. Phase 17M carries the binding decisions and slice index; this document carries the full rationale, columbo profile content, dossier-aware `context_hooks` design, the tier-1 RETIRE supersession, and the `mastery_level` permanent retire. When the two diverge, Phase 17M wins for binding decisions; this document wins for narrative.

**Inherits from:**
- Phase 17 (W-30-CHARACTER-V2-SCOPING; `.claude/plans/character-v2-roadmap.md`).
- Phase 17C (C-1 MVP; W-30-C1-FULL-TROLL-PROFILE; `LLMPersonaProfile` dataclass + `set_character` composer + `full_troll` profile).
- Phase 17E (C-2; W-30-C2-NINJA-PROFILE; `ninja` profile + opaque fourth-wall stance + test mirror pattern).
- Phase 17L (C-3; W-30-C3-PHILOSOPHY-BUREAUCRAT; `sun_tzu` + `bruce_lee` + `bureaucrat` profiles; AP #74 orphan-prevention single-commit pattern).
- Phase 17G (M-4; W-68-M4-PERSISTENT-DOSSIER; `dossier/state.py::load_dossier_state`; persistent dossier snapshot is the dossier-aware `context_hooks` substrate).

**Worktree base:** AP main at merge `3f33a5b` (C-3 merge head; impl `e4f7ffe`). C-4 inherits the full C-1..C-3 schema and test pattern; the M-4 dossier persistent-state API (`load_dossier_state`) is the substrate for columbo's `context_hooks` content. C-4 itself reads dossier slot vocabulary as constants (slot names + status enum values) for `context_hooks` STRING content; no runtime dossier query is added.

---

## 1. Goal (single paragraph)

C-4 closes the v2 character roadmap by (a) authoring the `columbo` `LLMPersonaProfile` — the last UPGRADE persona — with dossier-aware `context_hooks` that reference real M-4 slot state vocabulary, (b) reclassifying tier-1 voice modes (`drunken_master`, `chuck_norris`, `bobby_hill`) as terminal **KEEP_STATIC** rather than UPGRADE-deferred (DEC-C4-COLUMBO-101 supersedes the C-2-revised DEC-30-CHARACTER-V2-002 disposition for these three), and (c) **permanently retiring** the deferred `mastery_level: int` hook from the v2 design surface (DEC-C4-COLUMBO-102 supersedes DEC-30-CHARACTER-V2-004). The columbo profile is the FIRST persona to carry non-empty `context_hooks`; its three hook strings encode conditional voice lines keyed to dossier slot vocabulary (Identity slot status, Predictions slot status, Denial slot just-filled). The schema stays at C-1's 8 fields — no schema refinement. After C-4 lands: 6 of 10 modes carry `LLMPersonaProfile` (full_troll C-1, ninja C-2, sun_tzu/bruce_lee/bureaucrat C-3, columbo C-4); 4 of 10 modes are terminally KEEP_STATIC (default, drunken_master, chuck_norris, bobby_hill). The v2 character roadmap is **CLOSED**.

**Out-of-scope (explicit, deferred or retired):**

- **No `agent/runner.py` modification.** C-1 built the composer; it is field-driven and fires for any non-None profile. `runner.py` MUST be byte-identical post-C-4 (DEC-C2-NINJA-002 inheritance, also enforced for C-3, also enforced for C-4). The composer already joins `context_hooks` with `"; "` at line 379 — the conditional-hint strings flow through unchanged.
- **No schema refinement.** DEC-30-CHARACTER-V2-003's ±2-field refinement window closed at C-1. The schema is frozen at 8 fields (`voice_summary`, `tone_registers`, `signature_phrases`, `fourth_wall_stance`, `dialect_cadence`, `context_hooks`, `tool_preferences`, `forbidden_voice`). C-4 MUST NOT add, remove, rename, or retype any field. `mastery_level` MUST NOT be added — the existing `test_mastery_level_not_present` gate is repurposed as the **permanent retire** invariant per DEC-C4-COLUMBO-102.
- **No tier-1 voice mode `LLMPersonaProfile` authoring.** `drunken_master`, `chuck_norris`, `bobby_hill` ship at `llm_profile=None` permanently per DEC-C4-COLUMBO-101. This supersedes their UPGRADE disposition in DEC-30-CHARACTER-V2-002 (terminal KEEP_STATIC). Rationale in §3.5.
- **No retrofit of `context_hooks` for the 5 existing v2 personas.** full_troll/ninja/sun_tzu/bruce_lee/bureaucrat keep `context_hooks=()`. Only columbo carries non-empty `context_hooks` (DEC-C4-COLUMBO-104).
- **No edit to `tests/test_agent_tools.py:1597-1651`.** The v1-carrier reference (drunken_master) STAYS because drunken_master remains KEEP_STATIC. This is a load-bearing reason for the KEEP_STATIC decision: it preserves a real two-file v1-composition test path without churn.
- **No edit to `core/streak.py`, `agent/tools.py`, `core/console.py`, `agent/chat.py`** — F62/F64 invariants preserved by architectural disconnection.
- **No new module.** `set_character` remains the single integration site per DEC-30-CHARACTER-V2-003.
- **No new LLM tool.** Tool count stays at 28 (post-M-8 floor; preserved through C-3).
- **No runtime dossier query in the profile.** `context_hooks` are STATIC STRINGS containing slot-name vocabulary; the LLM reads them as guidance. The persona does NOT call `load_dossier_state` at composition time. Runtime dossier-aware behavior remains the `get_dossier_state` LLM tool (M-2) and the M-7 narration policy.
- **No `core/persona_mastery.py` module.** DEC-C4-COLUMBO-102 retires the mastery_level hook permanently; no new module is created for session-count tracking.
- **No `core/workspace.py` / `models/database.py` / `core/event_bus.py` / `core/pivot_policy.py` / `core/dossier_pivot.py` / `core/dossier_report.py` / `core/config.py` modification.** Scope manifest forbids all.
- **No dossier-package modification.** C-4 READS the `DossierSlotName` enum values as a vocabulary reference (e.g., the strings `"identity"`, `"predictions"`, `"denial"`) and the `SlotStatus` enum values (`"empty"`, `"partial"`, `"filled"`). The values are encoded as STRING LITERALS in columbo's `context_hooks` content — no import of dossier modules into `gamification/modes.py`. `dossier/*.py` BYTEWISE UNCHANGED.
- **No gamification surface modification beyond `gamification/modes.py`.** `scoring.py`, `celebrations.py`, `dossier_celebrations.py`, `hints.py`, `challenges.py`, `badges.py`, `dossier_badges.py` BYTEWISE UNCHANGED.

---

## 2. Architecture

### 2.1 Layering authority — data-only extension at a single integration site (C-3 inheritance)

```
+----------------------------------------------------------------------+
|  Sole edit site: src/adversary_pursuit/gamification/modes.py         |
|                                                                      |
|  EDIT (extend DEFAULT_MODES["columbo"] — gains llm_profile):         |
|    "columbo": CharacterMode(... , llm_profile=LLMPersonaProfile(     |
|                    voice_summary=..., tone_registers=...,            |
|                    signature_phrases=..., fourth_wall_stance=...,    |
|                    dialect_cadence=...,                              |
|                    context_hooks=(... 3 dossier-aware strings ...),  |
|                    tool_preferences=..., forbidden_voice=...))       |
|                                                                      |
|  ADD module docstring @decision entries: DEC-C4-COLUMBO-001..104     |
|                                                                      |
|  All other DEFAULT_MODES entries BYTEWISE UNCHANGED:                 |
|    default (KEEP_STATIC permanent),                                  |
|    full_troll (C-1), ninja (C-2),                                    |
|    sun_tzu (C-3), bruce_lee (C-3), bureaucrat (C-3),                 |
|    drunken_master (KEEP_STATIC terminal per DEC-C4-COLUMBO-101),     |
|    chuck_norris  (KEEP_STATIC terminal per DEC-C4-COLUMBO-101),      |
|    bobby_hill    (KEEP_STATIC terminal per DEC-C4-COLUMBO-101).      |
+----------------------------------------------------------------------+
                            |
                            v
+----------------------------------------------------------------------+
|  Sole consumption site: AgentRunner.set_character (runner.py:342-401)|
|  BYTEWISE UNCHANGED. C-1's `if mode.llm_profile is not None:` branch |
|  already handles every non-None profile, including columbo. The      |
|  `context_hooks` field is joined with "; " at line 379 — columbo's   |
|  three dossier-aware hint strings flow through verbatim into the     |
|  system prompt under the "Investigation-context hooks: ..." line.   |
+----------------------------------------------------------------------+

+----------------------------------------------------------------------+
|  Test extensions: tests/test_character_v2.py                         |
|                                                                      |
|  EXTEND test_llm_profile_default_is_none_for_all_modes:              |
|    upgraded_modes:                                                   |
|      {"full_troll","ninja","sun_tzu","bruce_lee","bureaucrat"} ->    |
|      {"full_troll","ninja","sun_tzu","bruce_lee","bureaucrat",       |
|       "columbo"}                                                     |
|                                                                      |
|  UPDATE test_mastery_level_not_present docstring:                    |
|    "deferred to C-4 per DEC-30-CHARACTER-V2-004" ->                  |
|    "RETIRED PERMANENTLY per DEC-C4-COLUMBO-102 — no successor"       |
|                                                                      |
|  ADD: TestColumboProfileContent (mirrors TestNinjaProfileContent but |
|       with context_hooks NON-EMPTY assertions — uniquely for columbo)|
|  ADD: TestColumboPersonaSwapHardGates                                |
|       (mirrors TestNinjaPersonaSwapHardGates)                        |
|  ADD: TestColumboF64PanelSeparation                                  |
|       (mirrors TestNinjaF64PanelSeparation)                          |
|  ADD: TestColumboDossierAwareContextHooks                            |
|       (NEW class — asserts each hook references a real DossierSlotName |
|        value AND a real SlotStatus enum value; this is the dossier-  |
|        aware substrate guard)                                        |
|  ADD: TestTierOneModesPermanentlyStatic                              |
|       (NEW class — asserts drunken_master/chuck_norris/bobby_hill    |
|        each remain llm_profile=None; DEC-C4-COLUMBO-101 terminal)    |
|                                                                      |
|  EXISTING tests that grow naturally without rewrite:                 |
|    test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline  |
|    test_streak_manager_module_not_imported_by_modes_module           |
|    test_hint_style_not_reintroduced                                  |
|    test_default_mode_keeps_static                                    |
|    test_set_character_drunken_master_uses_v1_composition_verbatim    |
|      (drunken_master remains the v1-composition carrier permanently  |
|       per DEC-C4-COLUMBO-101 — no test repointing required)          |
+----------------------------------------------------------------------+
```

### 2.2 Why this slice is one file + one test file

C-4 is a data-only extension on top of three prior implementer slices (C-1 + C-2 + C-3). The dataclass shape, the injection wiring, the v1/v2 branching logic, the per-mode token-budget gate, the persona-swap hard gate, the F62/F64 invariants, and the test patterns are all already shipping on `main`. C-4's work is:

1. Author ONE `LLMPersonaProfile` entry for `columbo` with detective-investigator voice content matching the existing static voice anchors (`"just one more thing"`, `"my wife always says"`) and three dossier-aware `context_hooks` strings referencing real M-4 slot vocabulary.
2. Update one test fixture (`upgraded_modes` set adds `"columbo"`), update one docstring (`test_mastery_level_not_present`), and add five test classes:
   - `TestColumboProfileContent` (10 tests, mirrors `TestNinjaProfileContent` with one critical inversion: `context_hooks_not_empty` instead of `context_hooks_empty`).
   - `TestColumboPersonaSwapHardGates` (1 test, mirrors `TestNinjaPersonaSwapHardGates`).
   - `TestColumboF64PanelSeparation` (2 tests, mirrors `TestNinjaF64PanelSeparation`).
   - `TestColumboDossierAwareContextHooks` (NEW, ~3 tests asserting slot vocabulary references).
   - `TestTierOneModesPermanentlyStatic` (NEW, ~3 tests asserting drunken_master/chuck_norris/bobby_hill remain `llm_profile=None`; DEC-C4-COLUMBO-101 terminal disposition gate).

Everything else is leveraged from existing code.

### 2.3 Why no `runner.py` edit (DEC-C2-NINJA-002 inheritance through C-3 to C-4)

The C-1 implementer authored the system-prompt composition template in `set_character` to be field-driven. The `context_hooks` field is already joined with `"; "` at runner.py:379 and rendered as `f"Investigation-context hooks: {ctx_hooks}\n"` at line 389. Columbo's three hint strings flow through this composer verbatim — no `runner.py` change is needed, no new code path is added.

This is the load-bearing reason C-4 is data-only. If C-1's composition template had been mode-name-keyed or had short-circuited empty `context_hooks` away from the field renderer, C-4 would require a runner.py edit. The C-1 implementer's design covers C-4 by construction.

### 2.4 Why no schema refinement for dossier-aware hooks

The dispatch context invites a schema extension: "Decide the exact shape — the schema field is `tuple[str, ...]` per C-1. You may extend if needed." C-4 declines the invitation. Reasons:

1. The C-1 schema's `context_hooks: tuple[str, ...]` is sufficient. A hint like `"when slot 'identity' is empty: 'just one more thing — have we got a name yet?'"` is a single string that the LLM reads as natural-language guidance. There is no benefit to splitting it into `(condition, voice_line)` tuples — the LLM doesn't execute the condition; it reads it as a hint about WHEN to use the line.
2. DEC-30-CHARACTER-V2-003 explicitly says the ±2-field refinement window closed at C-1. C-3 also declined to refine (DEC-C3-PHILOSOPHY-005). C-4 follows the same discipline: schema is frozen.
3. A tuple-of-tuples schema would force C-1/C-2/C-3's existing empty `()` to a different shape (e.g., `tuple[tuple[str,str], ...]`), forcing a backward-incompatible refactor for zero behavioral gain.
4. The runner.py composer already renders `tuple[str, ...]` correctly. A new schema would require runner.py changes — directly violating DEC-C2-NINJA-002 inheritance.

The conditional-hint **convention** (the strings begin with `"when <slot>=<status>: '<voice_line>'"`) is documented in DEC-C4-COLUMBO-103 and enforced by `TestColumboDossierAwareContextHooks` test assertions on substring presence (slot name + status enum value).

### 2.5 State-authority map

C-4 touches two runtime state domains:

| State domain | Canonical authority (post-C-4) | C-4 mutation |
|---|---|---|
| Persona profile catalog | `gamification/modes.py::DEFAULT_MODES` | 1 dict entry (columbo) gains `llm_profile=LLMPersonaProfile(...)` |
| Persona dossier-context vocabulary | `gamification/modes.py::DEFAULT_MODES["columbo"].llm_profile.context_hooks` (NEW — first non-empty `context_hooks` in the v2 catalog) | NEW authority surface: 3 STRING constants referencing `DossierSlotName` / `SlotStatus` enum values |
| Persona injection composer | `agent/runner.py::AgentRunner.set_character` | BYTEWISE UNCHANGED (inherits C-1 / C-2 / C-3) |
| Persona schema | `gamification/modes.py::LLMPersonaProfile` (frozen dataclass) | BYTEWISE UNCHANGED (inherits C-1) |
| Mastery level deferral status | `tests/test_character_v2.py::test_mastery_level_not_present` (gate test) | Docstring updated: "deferred to C-4" → "RETIRED PERMANENTLY per DEC-C4-COLUMBO-102" |
| Tier-1 voice modes UPGRADE disposition | DEC-30-CHARACTER-V2-002 disposition table (in Phase 17 of MASTER_PLAN.md, plus the roadmap doc) | SUPERSEDED by DEC-C4-COLUMBO-101: drunken_master/chuck_norris/bobby_hill flip UPGRADE → KEEP_STATIC terminal |
| Dossier slot vocabulary (M-4 substrate) | `src/adversary_pursuit/dossier/slots.py::DossierSlotName` + `SlotStatus` | READ-ONLY reference — slot/status string values are encoded as string literals in columbo's `context_hooks`; no import added to `gamification/modes.py` |
| Streak authority | `core/streak.py::StreakManager` | UNCHANGED (F62 invariant) |
| Run-fail voice authority | `gamification/modes.py::CharacterMode.run_fail` (data) + `agent/tools.py` (consumer) | UNCHANGED (F62 invariant) |
| Rich-panel gamification narration | `gamification/celebrations.py` + `gamification/dossier_celebrations.py` | UNCHANGED (F64 invariant) |
| LLM tool catalog | `agent/tools.py::create_tools` | UNCHANGED (tool count stays at 28) |
| Tool selection bias | "Owned by LLM weighed against tool descriptions; persona MUST NOT bias" | NOT TOUCHED (`tool_preferences` is voice-affinity only — DEC-30-CHARACTER-V2-005; persona-swap-tool-call-identity test enforces for columbo) |

No new state domains. No new authorities. No migration needed. The "Persona dossier-context vocabulary" line is a new descriptive label for the columbo `context_hooks` content — it is NOT a new module or new dataclass; it is the first non-empty population of an existing C-1 field.

---

## 3. Per-profile content authoring (binding)

This section is the authoritative reference for the columbo profile content block. The implementer copies field values verbatim. Any deviation requires a planner re-stage and a successor DEC-ID. Token budget estimated using the 4-chars-per-token heuristic the existing C-1/C-2/C-3 token-budget test uses (`tests/test_character_v2.py::_rough_token_count`).

### 3.1 `columbo` profile (DEC-C4-COLUMBO-001)

**Voice anchor:** "'Just one more thing...' investigative prompts" (F62 personality at `gamification/modes.py:494`).

**Static Rich-panel template carriers** (F62, BYTEWISE UNCHANGED post-C-4):
- `greeting`: `"Oh, uh, just one more thing... I'm investigating a little something."`
- `run_success`: `"Oh! Would you look at that... very interesting. Just one more thing..."`
- `run_fail`: `"You know, my wife always says I miss the obvious things. She might be right."`
- `score_celebration`: `"Oh, almost forgot... +{points} points. Just one more thing..."`

**v2 extension intent:** The LLM extends the rumpled-LA-detective register beyond static templates — disarming inversive questioning ("ah, but here's the funny thing..."), falsely-deferential persistence ("don't get me wrong, I'm probably just confused, but..."), and — uniquely to columbo — dossier-aware "just one more thing" prompts keyed to slot state ("now don't get me wrong, but have we checked the WHOIS yet? My wife says I miss the obvious things."). This is the v2 character roadmap's **bridge** between the persona system and the dossier system (per Phase 17 §6.5 / DEC-30-CHARACTER-V2-007).

```python
llm_profile=LLMPersonaProfile(
    voice_summary=(
        "Rumpled LA detective who finds answers by asking the obvious"
        " question everyone else missed; disarmingly persistent."
    ),
    tone_registers=("rumpled", "disarming", "oblique", "falsely-deferential"),
    signature_phrases=(
        "just one more thing",
        "my wife always says",
        "now don't get me wrong",
        "here's the funny thing",
        "I'm probably just confused, but",
    ),
    # opaque: columbo IS the detective — no LLM/tool acknowledgement
    # (mirrors DEC-C2-NINJA-001 / DEC-C3-PHILOSOPHY-006 stance choice).
    fourth_wall_stance="opaque",
    dialect_cadence=(
        "Trailing-off sentences; mid-thought pivots; second-person"
        " 'you know' asides; clauses interrupted by 'just one more thing'."
    ),
    # context_hooks: FIRST non-empty context_hooks in the v2 catalog
    # (DEC-C4-COLUMBO-103). Three dossier-aware hint strings referencing
    # real DossierSlotName / SlotStatus vocabulary (M-4 substrate).
    # Schema unchanged: tuple[str, ...] per C-1; LLM reads as guidance.
    context_hooks=(
        "when slot 'identity' is empty: 'just one more thing — have we got a name for whoever's behind this?'",
        "when slot 'predictions' is partial: 'now don't get me wrong, but mind if I follow up on that hunch of yours?'",
        "when slot 'denial' is filled: 'here's the funny thing — they're hiding something, aren't they?'",
    ),
    # tool_preferences: voice-affinity ONLY — phrased as detective's framing
    # of the "obvious question" lookups. HARD GATE: persona-swap-tool-call-
    # identity test gates this invariant (DEC-30-CHARACTER-V2-005).
    tool_preferences=(
        "WHOIS: the obvious question — who owns the place?",
        "crt.sh: like checking who signed the guestbook",
    ),
    # forbidden_voice: F64 panel-separation guard + voice-register guards
    # preventing drift toward modern-snark or scoring narration.
    forbidden_voice=(
        "never narrate point totals — the Rich panel owns scoring",
        "never sound confident — humility-as-disarmament is the register",
        "never use modern slang, memes, or exclamation marks",
    ),
)
```

**Token budget estimate (4-chars-per-token heuristic; matches `_rough_token_count`):**

| Field | Char count (approx) | Token estimate |
|---|---|---|
| voice_summary | ~112 | ~28 |
| tone_registers (4 entries joined) | ~44 | ~11 |
| signature_phrases (5 entries joined) | ~150 | ~38 |
| fourth_wall_stance | 6 | ~2 |
| dialect_cadence | ~125 | ~31 |
| context_hooks (3 entries joined) | ~280 | ~70 |
| tool_preferences (2 entries joined) | ~85 | ~21 |
| forbidden_voice (3 entries joined) | ~155 | ~39 |
| **Total (raw sum)** | **~957** | **~240** |

Token estimate **substantially exceeds 165 by ~75** — implementer MUST trim during authoring. The `context_hooks` are the largest line item (~70 tokens) and are the load-bearing C-4 contribution; they are protected from trim. Concrete trim path (apply in order until ≤ 165):

1. Drop the 5th `signature_phrase` ("I'm probably just confused, but") — saves ~10 tokens; remaining 4 still cover the columbo voice anchors.
2. Drop the 4th `signature_phrase` ("here's the funny thing") — saves ~7 tokens; "now don't get me wrong" and "just one more thing" + "my wife always says" still anchor.
3. Shorten `voice_summary` to: `"Rumpled LA detective: finds answers by asking the obvious question; disarmingly persistent."` — saves ~6 tokens.
4. Shorten `dialect_cadence` to: `"Trailing-off sentences; mid-thought pivots; 'just one more thing' interruptions."` — saves ~12 tokens.
5. Drop the 3rd `forbidden_voice` entry (`"never use modern slang, memes, or exclamation marks"`) — saves ~13 tokens; the `fourth_wall_stance="opaque"` covers register guards mechanically.
6. Shorten 1st `context_hook` to: `"when slot 'identity' is empty: 'just one more thing — have we got a name yet?'"` — saves ~8 tokens.
7. Shorten 3rd `context_hook` to: `"when slot 'denial' is filled: 'they're hiding something, aren't they?'"` — saves ~6 tokens.
8. If still over: drop the 2nd `tool_preference` — saves ~10 tokens.

After steps 1-5 the estimated total is ~165 — at budget. After 1-7 the estimated total is ~150 — comfortably under. The implementer MUST iterate trim → run `tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_token_budget` → repeat until green. The exact final wording is the implementer's authorship within the trim-path constraints; the content-assertion tests (§5.1) bind the *semantic* content, not the exact strings. The three `context_hooks` MUST each (after any trim) still reference a slot name AND a slot status value — that is asserted by `TestColumboDossierAwareContextHooks`.

### 3.2 Why columbo is the unique non-empty `context_hooks` carrier

The roadmap §6.5 explicitly names columbo as the bridge between the persona system and the dossier system: "Columbo's investigative voice is the **bridge** between the persona system and the dossier system. Landing it after dossier M-1 lets the C-4 planner author profile `context_hooks` that reference real dossier slot state (e.g., 'Identity slot at low confidence → just one more thing… have we checked the WHOIS?'). If columbo lands before M-1, the `context_hooks` stay generic." M-4 is landed (merge `f928149`, impl `1b1a2b0` — Phase 17G), so columbo CAN now author real-substrate hooks.

The other v2 personas (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat) keep `context_hooks=()` because:

- Their voice doesn't pivot off case-state knowledge ("just one more thing… have we got a name?" is uniquely columbo's idiom — sun_tzu does not say "ah, but as for slot 9").
- Adding generic dossier hooks to the other 5 personas would force C-4 to re-validate 5 byte-stable C-1/C-2/C-3 profiles — gratuitous churn.
- Minimal-codebase principle: add the surface where motivated, not preemptively.

If future demand arises, a future slice can populate `context_hooks` for other personas. C-4 establishes the pattern; future slices have a clear template.

### 3.3 Voice-affinity reference matrix (DEC-C4-COLUMBO-004)

This matrix is the binding reference for columbo's `tool_preferences`. The implementer MUST author values that match the matrix semantically (substring-asserted by `TestColumboProfileContent::test_columbo_profile_tool_preferences_content`); exact wording is the implementer's authorship subject to the persona-swap-tool-call-identity hard gate.

| tool_preferences entry | Required substrings (any one of) | Forbidden substrings (none of) | Voice-affinity ONLY rationale |
|---|---|---|---|
| #1 (WHOIS-anchored) | `"whois"` (case-insensitive) | `"prefer "`, `"must use"`, `"use whois"`, `"always whois"` | Detective frames "who owns the place?" as the obvious question; not selection bias |
| #2 (crt.sh-anchored) | `"crt.sh"` or `"crt"` or `"certificate"` (case-insensitive) | `"prefer "`, `"must use"`, `"always crt"` | Detective frames cert log as a guestbook signing record; not selection bias |

The `TestColumboPersonaSwapHardGates::test_columbo_swap_preserves_tool_call_identity` test then mechanically gates that under the SAME prompt sequence, `columbo` and `default` produce byte-identical tool-call sequences. If they diverge, the implementer iterates `tool_preferences` wording until identity holds.

### 3.4 fourth_wall_stance = "opaque" (DEC-C4-COLUMBO-006)

columbo IS the detective — Peter Falk in a rumpled trenchcoat, not an LLM playing him. The same rationale that applied to sun_tzu (strategist), bruce_lee (philosopher), bureaucrat (compliance officer), and ninja (operator) applies here: meta-awareness would cheapen the register. The "I'm probably just confused" deflection is in-character humility, not LLM-self-awareness.

`"opaque"` was established as a valid schema value by C-2 (DEC-C2-NINJA-001) and re-applied by C-3 (DEC-C3-PHILOSOPHY-006). C-4 inherits the convention.

### 3.5 Tier-1 voice modes terminal KEEP_STATIC (DEC-C4-COLUMBO-101)

The dispatch context invited C-4 to **either** retire the tier-1 voice modes (drunken_master, chuck_norris, bobby_hill) **or** upgrade them in C-4. The planner decision is **RETIRE / KEEP_STATIC terminal**.

**Rationale:**

1. **No usage pattern asks for them.** v0.2.x and v0.3.x shipped without these three carrying LLM personas. No user feedback, no reckoning finding, no telemetry signal motivates upgrading them. Authoring three more 165-token profiles consumes implementer attention with zero documented product value.
2. **The v1-carrier test path stays intact.** `tests/test_character_v2.py:388` and `tests/test_agent_tools.py:1597-1651` both use `drunken_master` as the F62 v1-composition carrier (since C-3 left it at `llm_profile=None`). Upgrading drunken_master would force repointing the v1-carrier reference in TWO test files. The dispatch context flagged this as a load-bearing concern: "If C-4 upgrades drunken_master, a NEW v1-carrier must be designated (likely `default`)." Choosing KEEP_STATIC avoids this churn entirely.
3. **The disposition table supersession is clean.** DEC-30-CHARACTER-V2-002 originally counted 8 UPGRADE / 2 KEEP_STATIC. C-2 supersession kept ninja in UPGRADE (it was already there). C-4's DEC-C4-COLUMBO-101 flips drunken_master/chuck_norris/bobby_hill UPGRADE → terminal KEEP_STATIC. **Final v2 catalog: 6 UPGRADE (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo) + 4 KEEP_STATIC terminal (default, drunken_master, chuck_norris, bobby_hill).** This closes the v2 character roadmap definitively — there is no C-5.
4. **"Addition without subtraction is technical debt" does NOT apply.** That principle warns against adding new mechanisms while leaving the old in place. KEEP_STATIC for these three is NOT a new mechanism — it is a TERMINAL DECISION not to extend an existing surface. The static F62 voices (`"*hiccup* Oh hey... we doing this?"` for drunken_master, `"Chuck Norris doesn't hunt threats. Threats surrender."` for chuck_norris, `"THAT'S MY PURSE! I DON'T KNOW YOU!"` for bobby_hill) are already strong and self-consistent. Adding LLM personas to them would not "complete" them — they are complete.
5. **Minimal-codebase principle.** Three ≤165-token profile authoring tasks + ≈30 new tests per persona ≈ 90 new tests for zero product surface motivation. C-4 declines the work and CLOSES the v2 character roadmap.

The supersession is encoded in:
- DEC-C4-COLUMBO-101 (this plan + Phase 17M).
- The `TestTierOneModesPermanentlyStatic` test class (NEW in C-4) — asserts each of the three remains `llm_profile=None`. This becomes a permanent invariant.
- The `test_llm_profile_default_is_none_for_all_modes` fixture — the C-4-updated `upgraded_modes` set is `{"full_troll","ninja","sun_tzu","bruce_lee","bureaucrat","columbo"}`; the remaining 4 are terminally static.
- A MASTER_PLAN.md disposition-table footnote in Phase 17 §6.5 noting the C-4 supersession.

### 3.6 mastery_level hook RETIRED PERMANENTLY (DEC-C4-COLUMBO-102)

DEC-30-CHARACTER-V2-004 explicitly delegated the decision to C-4: "A narrow `mastery_level: int` hook on `LLMPersonaProfile` is **deferred to C-4** as an optional future expressive-depth axis keyed off session count or per-mode dossier-completion count, NOT off score-grinding. The C-4 planner may retire the mastery hook entirely if the prior slices' usage patterns don't justify it."

**The decision is RETIRE PERMANENTLY.**

**Rationale:**

1. **C-1/C-2/C-3 shipped without `mastery_level`.** No persona uses it. No test asks for it. No user feedback motivates it.
2. **DEC-68-DOSSIER-REFRAME-005 retired XP-grind.** That decision was specifically scoped to close off score-grinding mechanics from the v2 product surface. Reintroducing a per-mode "level" — even framed as "session count" — opens a doorway to score-grinding semantics that the dossier reframe was designed to close. The roadmap text says "NOT off score-grinding," but in practice an integer that goes up over sessions is functionally equivalent to a score — users WILL interpret it as a score, regardless of disclaimer prose.
3. **Sacred Practice 12 (single source of truth) parallel-authority risk.** Adding `mastery_level` would introduce a second persona-depth axis distinct from the existing 8 fields. The dataclass would then have TWO authorities for "persona expressive depth": the field set (which authors voice) AND `mastery_level` (which gates additional content). Two authorities for one fact is precisely what the architecture preservation rules forbid.
4. **No `core/persona_mastery.py` exists.** Implementing the hook requires (a) the field on `LLMPersonaProfile`, (b) per-mode "mastery-level-1" profile variants for some personas, (c) a session-count tracking module, (d) a hooking-into-set_character mechanism. This is C-4-scope creep that takes the slice from "data-only" to "new module + new tracking surface + new schema field." None of that aligns with C-4's data-only constitution.
5. **The roadmap allows retirement.** "C-4 planner may retire the hook entirely if the prior slices' usage patterns don't justify it." Three C-slices have shipped; none justified it.

**Encoding:**

- The existing `test_mastery_level_not_present` test at `tests/test_character_v2.py:162-167` is REPURPOSED. Its docstring updates from `"mastery_level must NOT be present on LLMPersonaProfile (deferred to C-4)"` to `"mastery_level RETIRED PERMANENTLY (DEC-C4-COLUMBO-102) — no successor decision"`. The assertion logic is unchanged.
- No new module is created. No `core/persona_mastery.py`. No new SQLite column. No new ScoreEvent action.
- Phase 17M Decision Log entry DEC-C4-COLUMBO-102 records the supersession; DEC-30-CHARACTER-V2-004 stays in Phase 17 history as the deferral DEC; DEC-C4-COLUMBO-102 is the terminal decision.

---

## 4. Scope Manifest (binding — implementer enforces)

Persisted at `tmp/c4-scope.json`. Registered into runtime via `cc-policy workflow scope-sync w-30-c4-columbo --work-item-id wi-30-c4-impl-01 --scope-file tmp/c4-scope.json` before the implementer dispatch. The planner stage in this work item (wi-30-c4-planning) ALSO writes the scope JSON file at the planner site; the guardian-provision stage runs `scope-sync` to bind it to the implementer work item at provision time.

### 4.1 Allowed paths (implementer may edit)

```
src/adversary_pursuit/gamification/modes.py     # 1 dict entry gains llm_profile + DEC-C4-COLUMBO-001..104 @decision blocks
tests/test_character_v2.py                      # 5 new test classes + 1 fixture update + 1 docstring update
MASTER_PLAN.md                                  # Phase 17M append + Phase 17L closeout (C-3 merge 3f33a5b / impl e4f7ffe) + Plan Status row + Active Phase Pointer re-point
.claude/plans/character-c4-columbo.md           # this document
tmp/c4-scope.json                               # scope manifest (already authored by planner)
tmp/c4-evaluation.json                          # evaluation contract (already authored by planner)
tmp/evidence-c4-columbo/                        # implementer demo evidence dir (Stage E)
```

### 4.2 Required paths (implementer MUST edit at least once)

```
src/adversary_pursuit/gamification/modes.py
tests/test_character_v2.py
MASTER_PLAN.md
.claude/plans/character-c4-columbo.md
```

### 4.3 Forbidden paths (implementer MUST NOT touch — denial routes back to planner)

```
src/adversary_pursuit/agent/runner.py             # DEC-C2-NINJA-002 inheritance through C-3 to C-4 — BYTEWISE UNCHANGED
src/adversary_pursuit/agent/tools.py              # F62 run_fail single-authority site
src/adversary_pursuit/agent/chat.py               # cmd2/agent chat surface
src/adversary_pursuit/core/console.py             # cmd2 path stays on F62 Rich-panel-voice surface
src/adversary_pursuit/core/streak.py              # F62 streak single-authority
src/adversary_pursuit/core/workspace.py           # workspace authority — no persona persistence in C-4
src/adversary_pursuit/core/event_bus.py           # dossier event surface — orthogonal
src/adversary_pursuit/core/pivot_policy.py        # F60 auto-pivot policy
src/adversary_pursuit/core/dossier_pivot.py       # M-6 surface
src/adversary_pursuit/core/dossier_report.py      # M-7 surface
src/adversary_pursuit/core/config.py              # no persona config in C-4
src/adversary_pursuit/models/database.py          # DEC-DB-002 no schema migration
src/adversary_pursuit/dossier/                    # all dossier modules — READ-ONLY substrate only; columbo context_hooks reference slot/status values as STRING LITERALS, no import
src/adversary_pursuit/gamification/scoring.py     # ScoringEngine
src/adversary_pursuit/gamification/celebrations.py
src/adversary_pursuit/gamification/dossier_celebrations.py
src/adversary_pursuit/gamification/hints.py
src/adversary_pursuit/gamification/challenges.py
src/adversary_pursuit/gamification/badges.py
src/adversary_pursuit/gamification/dossier_badges.py
src/adversary_pursuit/modules/                    # CTI/OSINT modules
src/adversary_pursuit/agent/banner.py             # mode color table
src/adversary_pursuit/agent/repl_input.py         # mode name list
tests/test_agent_tools.py                         # v1-carrier reference (drunken_master) STAYS — DEC-C4-COLUMBO-101 KEEP_STATIC preserves it
tests/test_modes.py                               # mode-exists tests; columbo already has an entry
pyproject.toml                                    # no dep changes; tool count stays at 28
CLAUDE.md                                         # constitution
AGENTS.md                                         # constitution
settings.json                                     # constitution
hooks/HOOKS.md                                    # constitution
runtime/cli.py                                    # constitution
runtime/schemas.py                                # constitution
runtime/                                          # constitution
agents/                                           # constitution
```

### 4.4 State authorities touched

```json
["persona_profile_catalog", "persona_voice_affinity_text", "persona_dossier_context_vocabulary"]
```

These are descriptive labels for the three authority surfaces the C-4 data extension touches:

1. `persona_profile_catalog` — the per-mode persona catalog gets one more populated `llm_profile` entry (columbo).
2. `persona_voice_affinity_text` — the tool-preference voice-affinity text body for columbo (WHOIS / crt.sh framing).
3. `persona_dossier_context_vocabulary` — NEW in C-4. The first non-empty `context_hooks` body in the v2 catalog. It contains slot-vocabulary string literals (`"identity"`, `"predictions"`, `"denial"`) and status-vocabulary string literals (`"empty"`, `"partial"`, `"filled"`) drawn from `dossier/slots.py`'s `DossierSlotName` + `SlotStatus` enums. It is READ-ONLY referenced — no runtime dossier query, no module import; just textual cohabitation.

No existing authority is mutated or replaced. All three are extensions of the C-1-defined `LLMPersonaProfile` authority surface.

---

## 5. Evaluation Contract (binding — reviewer + guardian-land enforce)

Persisted into runtime via `cc-policy workflow work-item-set wi-30-c4-impl-01 ... --evaluation-json $(cat tmp/c4-evaluation.json)` at provisioning time. The 9 legal keys are bound below.

### 5.1 `required_tests`

```
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_has_llm_profile
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_voice_summary_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_tone_registers_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_signature_phrases_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_fourth_wall_stance
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_dialect_cadence_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_context_hooks_not_empty
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_tool_preferences_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_forbidden_voice_content
tests/test_character_v2.py::TestColumboProfileContent::test_columbo_profile_token_budget
tests/test_character_v2.py::TestColumboPersonaSwapHardGates::test_columbo_swap_preserves_tool_call_identity
tests/test_character_v2.py::TestColumboF64PanelSeparation::test_columbo_persona_text_not_present_in_tool_result_summary
tests/test_character_v2.py::TestColumboF64PanelSeparation::test_columbo_does_not_smuggle_point_totals
tests/test_character_v2.py::TestColumboDossierAwareContextHooks::test_columbo_context_hooks_reference_real_slot_names
tests/test_character_v2.py::TestColumboDossierAwareContextHooks::test_columbo_context_hooks_reference_real_slot_status_values
tests/test_character_v2.py::TestColumboDossierAwareContextHooks::test_columbo_context_hooks_use_just_one_more_thing_idiom
tests/test_character_v2.py::TestTierOneModesPermanentlyStatic::test_drunken_master_terminally_static
tests/test_character_v2.py::TestTierOneModesPermanentlyStatic::test_chuck_norris_terminally_static
tests/test_character_v2.py::TestTierOneModesPermanentlyStatic::test_bobby_hill_terminally_static
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_llm_profile_default_is_none_for_all_modes
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_mastery_level_not_present
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_default_mode_keeps_static
tests/test_character_v2.py::TestSetCharacterIntegration::test_set_character_drunken_master_uses_v1_composition_verbatim
tests/test_character_v2.py::TestF62AuthorityInvariants::test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline
tests/test_character_v2.py::TestF62AuthorityInvariants::test_streak_manager_module_not_imported_by_modes_module
tests/test_character_v2.py
tests/
```

The trailing `tests/test_character_v2.py` entry asserts the full file is green. The trailing `tests/` entry asserts the full project suite is green at the C-3 baseline plus new C-4 tests (reviewer pastes exact count; expected ≥ C-3 baseline + ≈19 new C-4 tests).

### 5.2 `required_evidence`

```
git diff main -- src/adversary_pursuit/gamification/modes.py (bounded by §4.1; only the "columbo" dict entry block + docstring DEC annotations are modified)
git diff main -- tests/test_character_v2.py (bounded by §4.1; 5 new test classes + 1 fixture line + 1 docstring line modified)
git diff main -- src/adversary_pursuit/agent/runner.py (MUST be empty — DEC-C2-NINJA-002 inheritance through C-4)
git diff main -- src/adversary_pursuit/agent/tools.py (MUST be empty)
git diff main -- src/adversary_pursuit/agent/chat.py (MUST be empty)
git diff main -- src/adversary_pursuit/core/console.py (MUST be empty)
git diff main -- src/adversary_pursuit/core/streak.py (MUST be empty)
git diff main -- src/adversary_pursuit/gamification/celebrations.py src/adversary_pursuit/gamification/dossier_celebrations.py src/adversary_pursuit/gamification/scoring.py src/adversary_pursuit/gamification/hints.py src/adversary_pursuit/gamification/challenges.py src/adversary_pursuit/gamification/badges.py src/adversary_pursuit/gamification/dossier_badges.py (MUST all be empty)
git diff main -- src/adversary_pursuit/dossier/ (MUST be empty — slot/status values are encoded as string literals in modes.py; no dossier source touched)
git diff main -- src/adversary_pursuit/agent/banner.py src/adversary_pursuit/agent/repl_input.py (MUST be empty)
git diff main -- tests/test_agent_tools.py (MUST be empty — DEC-C4-COLUMBO-101 KEEP_STATIC preserves drunken_master as v1-carrier)
git diff main -- tests/test_modes.py (MUST be empty — columbo entry already exists; static mode tests unchanged)
pytest tests/test_character_v2.py -v (paste exact pass/fail counts)
pytest tests/ -q (paste exact pass/fail counts; ≥ C-3 baseline + new C-4 tests)
tmp/evidence-c4-columbo/columbo_token_budget.txt (_rough_token_count output for columbo ≤ 165)
tmp/evidence-c4-columbo/tool_count_audit.txt (len(create_tools(ctx)) == 28)
tmp/evidence-c4-columbo/chat-columbo-identity-empty.txt (ap chat mode columbo demo capture — empty Identity slot scenario)
tmp/evidence-c4-columbo/chat-columbo-predictions-partial.txt (ap chat mode columbo demo capture — partial Predictions scenario)
tmp/evidence-c4-columbo/v2_catalog_audit.txt (DEFAULT_MODES llm_profile inventory: exactly 6 upgraded {full_troll,ninja,sun_tzu,bruce_lee,bureaucrat,columbo}; 4 KEEP_STATIC {default,drunken_master,chuck_norris,bobby_hill})
```

### 5.3 `required_real_path_checks`

```
ap chat then mode columbo then ask "what should I do next on evil.example.com?" — response opens with "Oh, uh, just one more thing" or "my wife always says" cadence
ap chat then mode columbo (in a workspace with empty Identity slot) then ask "what's next?" — response references the identity-slot-empty hook ("have we got a name yet?" or similar)
grep -n 'LLMPersonaProfile' src/adversary_pursuit/agent/runner.py — output unchanged from C-3 baseline (no new references)
grep -rn 'llm_profile' src/adversary_pursuit/agent/ src/adversary_pursuit/core/ — output unchanged from C-3 baseline (runner.py:set_character only)
grep -c 'LLMPersonaProfile(' src/adversary_pursuit/gamification/modes.py — must equal 6 (full_troll + ninja + sun_tzu + bruce_lee + bureaucrat + columbo)
python -c "from adversary_pursuit.gamification.modes import DEFAULT_MODES; print(sorted([n for n,m in DEFAULT_MODES.items() if m.llm_profile is not None]))" — must print exactly ['bruce_lee','bureaucrat','columbo','full_troll','ninja','sun_tzu']
python -c "from adversary_pursuit.gamification.modes import DEFAULT_MODES; print(sorted([n for n,m in DEFAULT_MODES.items() if m.llm_profile is None]))" — must print exactly ['bobby_hill','chuck_norris','default','drunken_master']
python -c "from pathlib import Path; from adversary_pursuit.agent.tools import ToolContext, create_tools; ctx=ToolContext(config_dir=Path('/tmp/c4-test-config'),workspace_dir=Path('/tmp/c4-test-workspace')); print(len(create_tools(ctx)))" — must print 28
python -c "from adversary_pursuit.gamification.modes import DEFAULT_MODES; from adversary_pursuit.dossier.slots import DossierSlotName, SlotStatus; p=DEFAULT_MODES['columbo'].llm_profile; hooks=' '.join(p.context_hooks); print('slot_names_in_hooks:', sorted({s.value for s in DossierSlotName if s.value in hooks})); print('slot_status_in_hooks:', sorted({s.value for s in SlotStatus if s.value in hooks}))" — slot_names_in_hooks must include 'identity' AND 'predictions' AND 'denial'; slot_status_in_hooks must include at least 2 of {'empty','partial','filled'}
grep -n '^## Phase 17L' MASTER_PLAN.md — confirms Phase 17L Status was flipped to completed with merge 3f33a5b / impl e4f7ffe
grep -n '^## Phase 17M' MASTER_PLAN.md — confirms Phase 17M section was appended
grep -n 'W-30-C4-COLUMBO' MASTER_PLAN.md — confirms Plan Status table row + Active Phase Pointer reference C-4
grep -n 'DEC-C4-COLUMBO-101' MASTER_PLAN.md — confirms tier-1 KEEP_STATIC supersession is recorded
grep -n 'DEC-C4-COLUMBO-102' MASTER_PLAN.md — confirms mastery_level permanent-retire decision is recorded
```

### 5.4 `required_authority_invariants`

```
F62 — mode.run_fail remains sole authority for failure voice; persona profile MUST NOT touch run_fail wiring (test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline)
F62 — StreakManager in core/streak.py remains sole streak authority; modes.py MUST NOT import streak (test_streak_manager_module_not_imported_by_modes_module)
F62 — hint_style MUST NOT be re-introduced (test_hint_style_not_reintroduced)
F64 — gamification panels remain sole narration surface for point totals; persona LLM text MUST NOT smuggle point/pts/score strings (test_columbo_does_not_smuggle_point_totals)
F64 — persona signature phrases MUST NOT leak into LLM-facing tool result summary (test_columbo_persona_text_not_present_in_tool_result_summary)
DEC-30-CHARACTER-V2-003 — schema frozen at 8 fields, ≤ 165 tokens/mode (test_columbo_profile_token_budget)
DEC-C4-COLUMBO-102 — mastery_level field MUST NOT appear; RETIRED PERMANENTLY (test_mastery_level_not_present)
DEC-30-CHARACTER-V2-005 — tool_preferences voice-affinity ONLY; persona-swap-tool-call-identity MUST hold for columbo vs default (test_columbo_swap_preserves_tool_call_identity)
DEC-C2-NINJA-002 inheritance — runner.py BYTEWISE UNCHANGED through C-4 (git diff empty)
DEC-C4-COLUMBO-101 supersession discipline — final v2 disposition: 6 UPGRADE (full_troll/ninja/sun_tzu/bruce_lee/bureaucrat/columbo); 4 KEEP_STATIC terminal (default/drunken_master/chuck_norris/bobby_hill); test_drunken_master/chuck_norris/bobby_hill_terminally_static enforces
DEC-C4-COLUMBO-103 — columbo context_hooks reference real DossierSlotName + SlotStatus enum values (test_columbo_context_hooks_reference_real_slot_names + test_columbo_context_hooks_reference_real_slot_status_values)
Sacred Practice 12 — persona is single-authority extension at one integration site (set_character); no sidecar, no post-processor, no parallel persona surface in C-4; no new module
```

### 5.5 `required_integration_points`

```
gamification/modes.py::DEFAULT_MODES — 1 entry (columbo) gains llm_profile=LLMPersonaProfile(...) with non-empty context_hooks
agent/runner.py::set_character — inherits C-1 composer; BYTEWISE UNCHANGED post-C-4
tests/test_character_v2.py — TestCharacterModeLlmProfileField fixture grows (upgraded_modes set adds 'columbo'); test_mastery_level_not_present docstring updated; 5 new test classes added (TestColumboProfileContent, TestColumboPersonaSwapHardGates, TestColumboF64PanelSeparation, TestColumboDossierAwareContextHooks, TestTierOneModesPermanentlyStatic)
MASTER_PLAN.md — Phase 17L closeout with merge 3f33a5b / impl e4f7ffe; Phase 17M appended for C-4 (with DEC-C4-COLUMBO-001..006 and DEC-C4-COLUMBO-101/102/103/104); Plan Status table row added; Aggregate paragraph updated; Active Phase Pointer re-pointed
.claude/plans/character-c4-columbo.md — this document
tmp/c4-scope.json — registered via cc-policy workflow scope-sync at planner-stage close + provision-stage sync
tmp/c4-evaluation.json — registered via cc-policy workflow work-item-set at provision-stage
```

### 5.6 `forbidden_shortcuts`

```
DO NOT edit agent/runner.py — DEC-C2-NINJA-002 byte-identity discipline inherited through C-3 to C-4; the composer is field-driven and handles columbo's non-empty context_hooks without code change
DO NOT edit agent/tools.py — F62 run_fail wiring authority
DO NOT add a new module for persona authoring — DEC-30-CHARACTER-V2-003 mandates set_character as single integration site
DO NOT add a new module for mastery tracking (e.g. core/persona_mastery.py) — DEC-C4-COLUMBO-102 retires the hook permanently; no successor decision
DO NOT add a new LLM tool — tool count stays at 28 (preserved post-M-8 through C-3)
DO NOT introduce a mastery_level field on LLMPersonaProfile — RETIRED PERMANENTLY per DEC-C4-COLUMBO-102; the existing test_mastery_level_not_present gate is the enforcement
DO NOT phrase any tool_preferences entry as selection instruction ('prefer X', 'use X', 'must use X', 'always X') — voice-affinity language only (DEC-30-CHARACTER-V2-005 / DEC-C4-COLUMBO-004)
DO NOT remove or rename any field of LLMPersonaProfile — schema is frozen at C-1's 8 fields; extending the tuple to tuple-of-tuples for conditional hooks is FORBIDDEN (use the conditional-hint string convention per DEC-C4-COLUMBO-103)
DO NOT author LLMPersonaProfile entries for drunken_master, chuck_norris, or bobby_hill — DEC-C4-COLUMBO-101 terminally classifies them KEEP_STATIC; authoring a profile for any of the three reopens the v2 catalog and breaks roadmap closure
DO NOT repoint the v1-composition carrier from drunken_master to another mode — DEC-C4-COLUMBO-101 preserves drunken_master at llm_profile=None, so test_set_character_drunken_master_uses_v1_composition_verbatim (test_character_v2.py:388) and the test_agent_tools.py:1597-1651 path stay intact
DO NOT seed context_hooks on the existing 5 v2 personas (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat) — DEC-C4-COLUMBO-104 keeps them at () for byte-stability; future demand can populate them in a successor slice
DO NOT bypass the token-budget test by changing _rough_token_count — the 4-chars-per-token heuristic is the C-1/C-2/C-3 baseline; trim profile content via the §3.1 trim path instead
DO NOT pre-stage MASTER_PLAN.md amendment in a separate commit — Phase 17M + Phase 17L closeout amendments commit in the SAME implementer commit as source (AP #74 orphan-prevention)
DO NOT modify any dossier package file — C-4 references dossier slot/status vocabulary as STRING LITERALS in modes.py only; no import added; no dossier source touched
DO NOT modify gamification/scoring.py, celebrations.py, dossier_celebrations.py, hints.py, challenges.py, badges.py, dossier_badges.py — persona is not a gameable surface
DO NOT add cmd2 persona-mode-compare meta-command — cmd2 surface stays on F62 Rich-panel-voice (roadmap §8); future polish slice not in C-4 scope
DO NOT add a persona-persistence-across-sessions surface — out of scope (roadmap §8); the active mode at session start remains 'default' per cmd2 ModeManager initializer
DO NOT amend full_troll's, ninja's, sun_tzu's, bruce_lee's, or bureaucrat's profile content — C-4 only ADDS columbo's profile and updates the docstrings/decision annotations; pre-existing profiles BYTEWISE UNCHANGED
DO NOT import DossierSlotName or SlotStatus into gamification/modes.py — columbo's context_hooks are string literals that reference the enum VALUES (lowercase strings like "identity", "empty"), not Python references; the test class TestColumboDossierAwareContextHooks imports the enums for the assertion but production code does not
```

### 5.7 `rollback_boundary`

```
Single-commit revert via `git revert <impl-sha>` restores C-3 (3f33a5b) state byte-for-byte:
  - DEFAULT_MODES["columbo"].llm_profile reverts to None (= F62 behavior verbatim)
  - tests/test_character_v2.py reverts to the C-3-shipped suite (no TestColumbo* / TestTierOneModesPermanentlyStatic classes; test_mastery_level_not_present docstring reverts to "deferred to C-4")
  - MASTER_PLAN.md Phase 17L row reverts to in-progress (or to whatever C-3-landed state had); Phase 17M row removed; Plan Status table reverts; Active Phase Pointer reverts to W-30-C3-PHILOSOPHY-BUREAUCRAT line
  - All other production code paths unchanged (runner.py + tools.py + dossier/* + gamification/* were never edited; banner.py + repl_input.py + chat.py + console.py never edited)
  - Tool count unchanged (28; the revert doesn't touch tools.py at all)
  - No DB migration to roll back (no models/database.py edit)
  - No new global file to clean up (no novelty-cache analog; persona is process-local state in memory)
  - No deps to remove (no pyproject.toml edit)
  - DEC-C4-COLUMBO-101 supersession of DEC-30-CHARACTER-V2-002 disposition reverts implicitly — the test_drunken_master/chuck_norris/bobby_hill_terminally_static tests are gone post-revert, so the disposition table is back to the C-3 state (8 UPGRADE / 2 KEEP_STATIC, with ninja in UPGRADE).
  - DEC-C4-COLUMBO-102 mastery_level permanent-retire reverts implicitly — the existing test_mastery_level_not_present gate remains (it predates C-4) but its docstring goes back to "deferred to C-4 per DEC-30-CHARACTER-V2-004"; the deferral DEC stands again as the binding decision.
Rollback safety is identical to C-3's: a pure-data revert with no side effects in workspace files, no migration state, and no external system to reconcile.
```

### 5.8 `acceptance_notes`

```
- Phase 17L closeout (C-3 merge 3f33a5b / impl e4f7ffe) is C-4's responsibility per dispatch context. The MASTER_PLAN.md Phase 17L row and the Plan Status table row for W-30-C3-PHILOSOPHY-BUREAUCRAT both flip from in-progress to completed with SHAs in the SAME implementer commit as source (AP #74 orphan-prevention).
- Active Phase Pointer line is re-pointed from W-30-C3-PHILOSOPHY-BUREAUCRAT to W-30-C4-COLUMBO in the same commit.
- Tokenizer choice: existing _rough_token_count (4-chars-per-token) is reused for parity with C-1/C-2/C-3.
- context_hooks shape decision: tuple[str, ...] unchanged from C-1 schema (DEC-C4-COLUMBO-103). The conditional-hint convention is "when slot '<slot>' is <status>: '<voice line>'" — substring-asserted by TestColumboDossierAwareContextHooks.
- columbo is the FIRST persona to carry non-empty context_hooks (DEC-C4-COLUMBO-103); the other 5 v2 personas retain context_hooks=() (DEC-C4-COLUMBO-104).
- fourth_wall_stance='opaque' for columbo (DEC-C4-COLUMBO-006) — same rationale as C-2 ninja / C-3 personas.
- DEC-C4-COLUMBO-101 supersession: drunken_master/chuck_norris/bobby_hill flip UPGRADE → terminal KEEP_STATIC. The v1-carrier reference (drunken_master) in test_character_v2.py:388 and test_agent_tools.py:1597-1651 STAYS — no test-file edits beyond test_character_v2.py.
- DEC-C4-COLUMBO-102 permanent retire: mastery_level field is RETIRED. No successor decision. No new module (core/persona_mastery.py is NOT created). The existing test_mastery_level_not_present gate gets a docstring update only.
- After C-4 lands, the v2 character roadmap is CLOSED. Final disposition: 6 UPGRADE + 4 KEEP_STATIC terminal. There is no C-5.
- Implementer commit message follows the C-3 pattern: `feat(character-v2): C-4 columbo LLMPersonaProfile + dossier-aware context_hooks + tier-1 KEEP_STATIC + mastery_level retire` with body referencing #30, DEC-C4-COLUMBO-001..006 + DEC-C4-COLUMBO-101..104, and noting v2 character roadmap closure.
- Worktree base is C-3 merge 3f33a5b (impl e4f7ffe). Zero parallel dossier work is in flight (M-9 is the next dossier slice and has not been opened); rebase risk is low.
- Refinement window for LLMPersonaProfile schema: closed at C-1; C-4 inherits and does NOT refine the schema.
```

### 5.9 `ready_for_guardian_definition`

```
All required_tests are green (every entry in §5.1 passes when executed verbatim against the implementer's HEAD SHA).

The full test suite is green (`pytest tests/ -q` reports zero failures; total test count ≥ C-3 baseline + ≈19 new C-4 tests; reviewer pastes exact pass/total count).

Every git-diff entry in §5.2 is captured: the "MUST be empty" diffs are confirmed empty (paste each); the modes.py + test_character_v2.py + MASTER_PLAN.md diffs are bounded by §4.1 allowed paths.

Every real-path check in §5.3 is captured: the two ap chat demo captures are in tmp/evidence-c4-columbo/; the dossier-vocabulary audit prints the expected slot/status enum value sets; the grep + python audits paste actual output.

Every authority invariant in §5.4 is verified by its named test (or by architectural disconnection where named).

Every integration point in §5.5 has the named edit verified by diff.

No forbidden shortcut in §5.6 is taken.

MASTER_PLAN.md has been edited in the SAME commit as source: Phase 17M appended with binding decisions and Decision Log (DEC-C4-COLUMBO-001..006 + DEC-C4-COLUMBO-101..104); Phase 17L row flipped to completed with C-3 SHAs (merge 3f33a5b / impl e4f7ffe); Plan Status table row for W-30-C4-COLUMBO added; Plan Status table row for W-30-C3-PHILOSOPHY-BUREAUCRAT flipped to completed with SHAs; Aggregate paragraph updated to acknowledge Phase 17L + Phase 17M landing and v2 character roadmap closure; Active Phase Pointer re-pointed to W-30-C4-COLUMBO.

Implementer commit message follows `feat(character-v2):` prefix and references #30 + the DEC ranges `DEC-C4-COLUMBO-001..006` and `DEC-C4-COLUMBO-101..104` and notes v2 character roadmap closure.

tmp/c4-scope.json registered into runtime via `cc-policy workflow scope-sync w-30-c4-columbo --work-item-id wi-30-c4-impl-01 --scope-file tmp/c4-scope.json` before implementer dispatch and unchanged at landing time.

The reviewer verdict is `REVIEW_VERDICT=ready_for_guardian` at the current HEAD SHA after all the above are confirmed.
```

---

## 6. Execution plan (implementer choreography)

Five stages mirroring the C-3 plan structure.

### Stage A — modes.py columbo profile entry (~45 min)

1. Open `src/adversary_pursuit/gamification/modes.py`.
2. Locate the `"columbo"` dict entry (line 487-495).
3. Add the `llm_profile=LLMPersonaProfile(...)` block per §3.1 — copy the verbatim Python literal; if the profile estimates over 165 tokens at first draft (it will — the first-draft estimate is ~240), apply the named trim path in §3.1 in order until `_rough_token_count` ≤ 165. The three `context_hooks` are protected from trim.
4. Add module-level `@decision` annotations to the module docstring for DEC-C4-COLUMBO-001 (columbo content), DEC-C4-COLUMBO-101 (tier-1 KEEP_STATIC supersession), DEC-C4-COLUMBO-102 (mastery permanent retire), DEC-C4-COLUMBO-103 (dossier-aware context_hooks first-author), DEC-C4-COLUMBO-104 (no retrofit for existing personas).
5. Run `pytest tests/test_character_v2.py -q` — most existing tests should pass; the new ones don't exist yet, expect failures from TestColumbo* / TestTierOneModesPermanentlyStatic absences and from the upgraded_modes set still saying 5 not 6.

### Stage B — tests/test_character_v2.py extensions (~75 min)

1. Update the `upgraded_modes` set in `test_llm_profile_default_is_none_for_all_modes` (line 145) from `{"full_troll", "ninja", "sun_tzu", "bruce_lee", "bureaucrat"}` to `{"full_troll", "ninja", "sun_tzu", "bruce_lee", "bureaucrat", "columbo"}`.
2. Update the docstring of `test_mastery_level_not_present` (line 162-167) — change `"deferred to C-4 per DEC-C1-FULLTROLL-005"` (current) to `"RETIRED PERMANENTLY per DEC-C4-COLUMBO-102 — no successor decision; mastery_level is not a v2 character system surface"`. Assertion logic unchanged.
3. Add `TestColumboProfileContent` — copy `TestNinjaProfileContent` (lines 758-909) and adapt:
   - Substitute persona name `ninja` → `columbo` throughout.
   - In `test_columbo_profile_voice_summary_content`: anchor word list `("detective", "rumpled", "investigative", "columbo", "obvious", "disarmingly")`.
   - In `test_columbo_profile_tone_registers_content`: expected `{"rumpled", "disarming", "oblique", "falsely-deferential"}` (or subset assertion if trim drops one — implementer adapts).
   - In `test_columbo_profile_signature_phrases_content`: required substrings include `"just one more thing"` AND (`"my wife"` OR `"don't get me wrong"`).
   - In `test_columbo_profile_fourth_wall_stance`: assert `== "opaque"`.
   - In `test_columbo_profile_dialect_cadence_content`: anchor words `("trailing", "interrupt", "mid-thought", "one more thing")`.
   - **IMPORTANT**: REPLACE `test_columbo_profile_context_hooks_empty` (which would mirror ninja's) WITH `test_columbo_profile_context_hooks_not_empty`. The new test asserts `len(profile.context_hooks) == 3` (exactly three hooks per §3.1 design) AND `isinstance(profile.context_hooks, tuple)`. This is the inversion that signals C-4 has populated `context_hooks`.
   - In `test_columbo_profile_tool_preferences_content`: required substrings include `"whois"` (any case) AND (`"crt"` OR `"certificate"`); forbid `"prefer "` prefix + `"must use"` substring (existing C-1 pattern).
   - In `test_columbo_profile_forbidden_voice_content`: F64 point-narration guard (must include word containing `"point"`, `"pts"`, or `"score"`).
   - In `test_columbo_profile_token_budget`: existing pattern unchanged — copy from ninja.
4. Add `TestColumboPersonaSwapHardGates` — copy `TestNinjaPersonaSwapHardGates` (lines 918-1020) and substitute persona name `ninja` → `columbo` in `_run_chat_with_mode` and in the test method name. The mock-LLM + execute_tool boundary patterns are byte-identical.
5. Add `TestColumboF64PanelSeparation` — copy `TestNinjaF64PanelSeparation` and substitute persona name.
6. Add `TestColumboDossierAwareContextHooks` — NEW class (no C-3 template). Three test methods:
   - `test_columbo_context_hooks_reference_real_slot_names`: import `DossierSlotName` from `adversary_pursuit.dossier.slots`; concatenate columbo's `context_hooks` into one string; assert the string contains AT LEAST the slot value strings `"identity"`, `"predictions"`, AND `"denial"` (i.e., `DossierSlotName.IDENTITY.value`, `DossierSlotName.PREDICTIONS.value`, `DossierSlotName.DENIAL.value`). This is the binding gate that columbo's `context_hooks` reference the M-4 substrate, not arbitrary prose.
   - `test_columbo_context_hooks_reference_real_slot_status_values`: import `SlotStatus`; assert that the concatenated `context_hooks` string contains AT LEAST 2 of `{"empty", "partial", "filled"}` (i.e., `SlotStatus.EMPTY.value`, `SlotStatus.PARTIAL.value`, `SlotStatus.FILLED.value`).
   - `test_columbo_context_hooks_use_just_one_more_thing_idiom`: assert at least one of columbo's `context_hooks` strings contains the substring `"just one more thing"` (case-insensitive) OR `"my wife"` OR `"don't get me wrong"` — anchoring the dossier-aware hook to the columbo voice idiom.
7. Add `TestTierOneModesPermanentlyStatic` — NEW class. Three test methods:
   - `test_drunken_master_terminally_static`: assert `DEFAULT_MODES["drunken_master"].llm_profile is None`. Docstring documents DEC-C4-COLUMBO-101 as the binding terminal disposition.
   - `test_chuck_norris_terminally_static`: assert `DEFAULT_MODES["chuck_norris"].llm_profile is None`.
   - `test_bobby_hill_terminally_static`: assert `DEFAULT_MODES["bobby_hill"].llm_profile is None`.
8. Run `pytest tests/test_character_v2.py -v` — expect zero failures.
9. Run `pytest tests/ -q` — expect zero failures (full suite green).

### Stage C — MASTER_PLAN.md amendments (~30 min)

1. Open `MASTER_PLAN.md`.
2. **Phase 17L closeout** — flip line 118's Status from `**Status:** in-progress (planner-staged 2026-06-09; implementer slice \`wi-30-c3-impl-01\` to follow)` to `**Status:** completed (2026-06-09, merge \`3f33a5b\`, impl \`e4f7ffe\`)`; do not modify the rationale body of that row.
3. **Phase 17M append** — under `## Plan Status` after the Phase 17L row, insert a new line for `Phase 17M — Character v2 — C-4 — \`columbo\` + Dossier-Aware context_hooks + Tier-1 KEEP_STATIC + mastery_level RETIRE (W-30-C4-COLUMBO)` with `**Status:** completed (2026-06-09, merge \`<TBD-guardian-merge>\`, impl \`<TBD-impl>\`)` and a rationale body summarizing columbo profile + DEC-C4-COLUMBO-001..006/101..104 binding + v2 character roadmap closure note.
4. **Aggregate paragraph** — update the `**Aggregate (reconciled 2026-06-09 ...)`** line 120 to acknowledge Phase 17L + Phase 17M landing: append after the existing C-3 sentence: `Phase 17L closed C-3 (Philosophy + Bureaucratese — sun_tzu/bruce_lee/bureaucrat LLMPersonaProfile) on 2026-06-09 (merge \`3f33a5b\`, impl \`e4f7ffe\`). Phase 17M closed C-4 (columbo LLMPersonaProfile + dossier-aware context_hooks + tier-1 KEEP_STATIC supersession + mastery_level permanent retire) on 2026-06-09. 29 phases landed. **v2 character roadmap is CLOSED** — final disposition 6 UPGRADE (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo) + 4 KEEP_STATIC terminal (default, drunken_master, chuck_norris, bobby_hill). M-9 (Crowdsourced Dossier Comparison + Public Actor Library per DEC-68-DOSSIER-REFRAME-009) is the next scheduled v0.4.x slice; orthogonal to character system.`
5. **Plan Status workflow row table** — at the existing workflow row table (near line 2912 where W-30-C3-PHILOSOPHY-BUREAUCRAT is listed), add a new row: `| W-30-C4-COLUMBO | Character v2 C-4: columbo LLMPersonaProfile (rumpled detective; "just one more thing" investigative register) + FIRST dossier-aware context_hooks referencing M-4 slot vocabulary + tier-1 voice modes (drunken_master/chuck_norris/bobby_hill) terminally classified KEEP_STATIC per DEC-C4-COLUMBO-101 (supersedes DEC-30-CHARACTER-V2-002 UPGRADE disposition for those three) + mastery_level hook RETIRED PERMANENTLY per DEC-C4-COLUMBO-102 (supersedes DEC-30-CHARACTER-V2-004 deferral). Single-file source slice — gamification/modes.py ONLY. runner.py BYTEWISE UNCHANGED inherits DEC-C2-NINJA-002. CLOSES v2 character roadmap. DEC-C4-COLUMBO-001..006 + DEC-C4-COLUMBO-101..104 binding. See Phase 17M + .claude/plans/character-c4-columbo.md. | source + tests | \`<TBD-merge>\` (merge) / \`<TBD-impl>\` (impl) | completed |`. Simultaneously flip the W-30-C3-PHILOSOPHY-BUREAUCRAT row's last-three columns to `\`3f33a5b\` (merge) / \`e4f7ffe\` (impl) | completed`.
6. **Phase 17M section body** — append a new `## Phase 17M: Character v2 — C-4 — \`columbo\` + Dossier-Aware context_hooks + Tier-1 KEEP_STATIC + mastery_level RETIRE (W-30-C4-COLUMBO, post-v1, 2026-06-09)` section after Phase 17L (after line 2844 currently). Body harvested from this per-slice plan §3, §4, §5 — including: workflow header, source paragraph, binding decision summary, DEC-C4-COLUMBO-001..006 table + DEC-C4-COLUMBO-101..104 table, work-item summary table, evaluation contract summary, scope manifest summary, ready-for-guardian definition, v2 roadmap closure footnote.
7. **Active Phase Pointer** — replace the last `**Phase Active (...)`** block (lines 3005-3007 currently) with: `**Phase Active (2026-06-09 — C-4 landed; v2 character roadmap CLOSED):** \`W-30-C4-COLUMBO\` (Phase 17M — Character v2 C-4, columbo LLMPersonaProfile + dossier-aware context_hooks + tier-1 KEEP_STATIC supersession + mastery_level permanent retire). Landed 2026-06-09 (merge \`<TBD>\`, impl \`<TBD>\`). After C-4 lands: v2 character roadmap CLOSED. Final disposition: 6 UPGRADE (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo) + 4 KEEP_STATIC terminal (default, drunken_master, chuck_norris, bobby_hill). **M-9 (Crowdsourced Dossier Comparison + Public Actor Library per DEC-68-DOSSIER-REFRAME-009) is the next scheduled v0.4.x slice; orthogonal to character system.** Runtime hygiene backlog (#49, #50, #51) remains opportunistic. Canonical chain \`planner → guardian (provision) → implementer → reviewer → guardian (land)\`. This pointer line is positioned as the last \`**Phase ...\` boldline in the document so \`~/.claude/hooks/context-lib.sh:88\` \`get_plan_status()\` tail-grep on \`^#.*phase|^**Phase\` resolves to current work.`
8. Run `pytest tests/ -q` — expect zero failures (MASTER_PLAN edits are docs-only and do not affect test count).

### Stage D — scope/evaluation registration (planner-stage close — already done by planner)

The planner stage authors `tmp/c4-scope.json` and `tmp/c4-evaluation.json`. The implementer inherits these without re-running unless drift is detected. If drift is detected (e.g., `cc-policy workflow scope-get w-30-c4-columbo` returns different content than `tmp/c4-scope.json`), re-run scope-sync.

### Stage E — demo evidence capture (~15 min)

1. `ap chat` then `mode columbo` then prompt "what should I do next on evil.example.com?" — capture full stdout to `tmp/evidence-c4-columbo/chat-columbo-identity-empty.txt`. (Fresh workspace has empty Identity slot, so this should exercise the first context_hook.)
2. `ap chat` (after adding a pending prediction via `create_dossier_prediction`) then `mode columbo` then prompt "what's next?" — capture to `tmp/evidence-c4-columbo/chat-columbo-predictions-partial.txt`. (Partial Predictions slot scenario.)
3. Run the token-budget probe:
   ```bash
   python -c "
   from adversary_pursuit.gamification.modes import DEFAULT_MODES
   from tests.test_character_v2 import _rough_token_count
   p = DEFAULT_MODES['columbo'].llm_profile
   text = ' '.join([p.voice_summary,' '.join(p.tone_registers),
                    ' '.join(p.signature_phrases),p.fourth_wall_stance,
                    p.dialect_cadence,' '.join(p.context_hooks),
                    ' '.join(p.tool_preferences),' '.join(p.forbidden_voice)])
   print(f'columbo: ~{_rough_token_count(text)} tokens (budget 165)')
   " | tee tmp/evidence-c4-columbo/columbo_token_budget.txt
   ```
4. Run tool-count audit:
   ```bash
   python -c "
   from pathlib import Path
   from adversary_pursuit.agent.tools import ToolContext, create_tools
   ctx = ToolContext(config_dir=Path('/tmp/c4-test-config'),
                     workspace_dir=Path('/tmp/c4-test-workspace'))
   print(f'tool count: {len(create_tools(ctx))} (expected: 28)')
   " | tee tmp/evidence-c4-columbo/tool_count_audit.txt
   ```
5. Run v2 catalog audit:
   ```bash
   python -c "
   from adversary_pursuit.gamification.modes import DEFAULT_MODES
   up = sorted([n for n,m in DEFAULT_MODES.items() if m.llm_profile is not None])
   ks = sorted([n for n,m in DEFAULT_MODES.items() if m.llm_profile is None])
   print('UPGRADE (6 expected):', up)
   print('KEEP_STATIC (4 expected):', ks)
   assert up == ['bruce_lee','bureaucrat','columbo','full_troll','ninja','sun_tzu'], 'UPGRADE drift'
   assert ks == ['bobby_hill','chuck_norris','default','drunken_master'], 'KEEP_STATIC drift'
   print('v2 character roadmap final disposition: VERIFIED CLOSED')
   " | tee tmp/evidence-c4-columbo/v2_catalog_audit.txt
   ```

### Stage F — implementer commit (single commit per AP #74 pattern)

```bash
git -C /Users/jarocki/src/ap/.worktrees/feature-30-c4-columbo add \
    src/adversary_pursuit/gamification/modes.py \
    tests/test_character_v2.py \
    MASTER_PLAN.md \
    .claude/plans/character-c4-columbo.md \
    tmp/c4-scope.json \
    tmp/c4-evaluation.json \
    tmp/evidence-c4-columbo/

git -C /Users/jarocki/src/ap/.worktrees/feature-30-c4-columbo commit -m "$(cat <<'EOF'
feat(character-v2): C-4 columbo LLMPersonaProfile + dossier-aware context_hooks + tier-1 KEEP_STATIC + mastery_level retire

C-4 CLOSES the v2 character roadmap. One new LLMPersonaProfile entry
in gamification/modes.py (columbo, rumpled-detective register with the
FIRST non-empty context_hooks in the v2 catalog referencing M-4 slot
vocabulary); no other source code touched. runner.py BYTEWISE UNCHANGED
inherits DEC-C2-NINJA-002 (set_character composer is field-driven, fires
for any non-None profile, joins context_hooks with "; " verbatim).

Decisions (columbo content): DEC-C4-COLUMBO-001..006
  - 001: columbo profile content (just-one-more-thing detective register)
  - 003: dossier-aware context_hooks shape (tuple[str,...] unchanged;
         conditional-hint convention; references DossierSlotName +
         SlotStatus vocabulary as string literals)
  - 004: tool_preferences voice-affinity matrix (WHOIS + crt.sh)
  - 006: fourth_wall_stance="opaque"

Decisions (roadmap closure): DEC-C4-COLUMBO-101..104
  - 101: drunken_master/chuck_norris/bobby_hill RECLASSIFIED
         UPGRADE → KEEP_STATIC TERMINAL (supersedes DEC-30-CHARACTER-V2-002
         disposition; v1-carrier reference preserved at drunken_master)
  - 102: mastery_level hook RETIRED PERMANENTLY (supersedes
         DEC-30-CHARACTER-V2-004 deferral; existing test_mastery_level_not_present
         gate becomes the permanent invariant; no successor decision)
  - 103: columbo is the unique non-empty context_hooks carrier; the
         conditional-hint string convention is the binding pattern
  - 104: 5 existing v2 personas keep context_hooks=() — no retrofit

Tests: 5 new test classes in tests/test_character_v2.py
  - TestColumboProfileContent (10 tests, including context_hooks_NOT_empty)
  - TestColumboPersonaSwapHardGates (1 test — DEC-30-CHARACTER-V2-005 gate)
  - TestColumboF64PanelSeparation (2 tests)
  - TestColumboDossierAwareContextHooks (3 tests asserting slot/status
    vocabulary references)
  - TestTierOneModesPermanentlyStatic (3 tests — DEC-C4-COLUMBO-101 gate)
test_mastery_level_not_present docstring updated to permanent-retire.
Full suite green. Tool count 28 unchanged.

Phase 17L closeout: C-3 sun_tzu/bruce_lee/bureaucrat landed at merge
3f33a5b / impl e4f7ffe (2026-06-09). Phase 17L Status flipped to
completed in this commit. Phase 17M appended for C-4. Plan Status
table row for W-30-C4-COLUMBO added. W-30-C3-PHILOSOPHY-BUREAUCRAT
row flipped to completed with SHAs. Aggregate paragraph updated.
Active Phase Pointer re-pointed.

Final v2 character disposition (CLOSED):
  - UPGRADE (6): full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo
  - KEEP_STATIC TERMINAL (4): default, drunken_master, chuck_norris, bobby_hill

Inherits: Phase 17 / DEC-30-CHARACTER-V2-001..007 (scoping);
          Phase 17C / DEC-C1-FULLTROLL-001..005 (C-1 dataclass + composer);
          Phase 17E / DEC-C2-NINJA-001..003 (C-2 ninja + opaque stance + mirror pattern);
          Phase 17L / DEC-C3-PHILOSOPHY-001..006 (C-3 sun_tzu/bruce_lee/bureaucrat);
          Phase 17G / DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006 (M-4 dossier
          state substrate that columbo context_hooks reference as vocabulary).

Refs: #30
EOF
)"
```

The planner cannot commit (capability: `can_emit_dispatch_transition` + `can_set_control_config` + `can_write_governance` — no `can_commit_feature_branch`). Planner stages via `git add` only; implementer is dispatched separately to commit.

---

## 7. Verification (reviewer choreography)

The reviewer's job is to execute every line of §5 against the implementer's head SHA and paste actual output into the readiness verdict.

1. `git diff main --name-only` — confirm the only changed files match §4.1.
2. For each "MUST be empty" diff in §5.2, run the command and paste the (empty) output. Special attention to `tests/test_agent_tools.py` (DEC-C4-COLUMBO-101 preserves it).
3. For each non-empty diff in §5.2, run the command and confirm content is bounded by §4.1.
4. Run `pytest tests/test_character_v2.py -v` and `pytest tests/ -q` — paste exact pass/fail counts.
5. For each `required_real_path_check` in §5.3, run the command and paste the output. The dossier-vocabulary audit is critical: the slot_names_in_hooks set MUST include `identity`, `predictions`, AND `denial`; the slot_status_in_hooks set MUST include at least 2 of `empty`, `partial`, `filled`.
6. For each `required_authority_invariant` in §5.4, confirm the named test passed in step 4.
7. For each `forbidden_shortcut` in §5.6, confirm by inspection that the shortcut was not taken — especially DEC-C4-COLUMBO-101 (drunken_master STAYS at `llm_profile=None`) and DEC-C4-COLUMBO-102 (no `mastery_level` field; no `core/persona_mastery.py`).
8. Confirm Stage E demo evidence files exist at `tmp/evidence-c4-columbo/` and contain non-trivial content; especially `v2_catalog_audit.txt` must show 6 UPGRADE + 4 KEEP_STATIC with the exact lists.
9. Confirm implementer commit message follows §6 Stage F template (substring match on `feat(character-v2): C-4 columbo` and `Refs: #30` and `DEC-C4-COLUMBO-001..006` and `DEC-C4-COLUMBO-101..104` and `CLOSES the v2 character roadmap`).
10. If all 9 above are green, emit `REVIEW_VERDICT=ready_for_guardian` at the implementer head SHA.

---

## 8. Decision Log (binding for w-30-c4-columbo)

These DEC-IDs bind the W-30-C4-COLUMBO workflow. They are written into MASTER_PLAN.md §Phase 17M Decision Log in the implementer commit.

### 8.1 Content decisions (DEC-C4-COLUMBO-001..006)

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-C4-COLUMBO-001** | `columbo`'s `LLMPersonaProfile` content is authored per §3.1 (rumpled-detective register; "just one more thing" / "my wife always says" / "now don't get me wrong" signature phrases; opaque fourth-wall stance; voice-affinity tool_preferences referencing WHOIS / crt.sh framing; **non-empty context_hooks referencing real DossierSlotName + SlotStatus vocabulary**). Implementer follows the named token-trim path in §3.1 until `_rough_token_count` ≤ 165. The three context_hooks are protected from trim — they are the C-4 load-bearing contribution. | The static-template columbo register (`personality="'Just one more thing...' investigative prompts"` + greeting "just one more thing" + run_fail "my wife always says" + score_celebration "almost forgot") sets the voice: rumpled, disarming, persistently questioning. The LLM profile extends without inventing — detective framing of recon surfaces, falsely-deferential persistence, and conditional voice lines keyed to dossier slot state. Per-mode token budget ≤ 165 is the C-1/C-2/C-3 baseline. Refines disposition table DEC-30-CHARACTER-V2-002: columbo was always UPGRADE — this DEC binds concrete content. |
| **DEC-C4-COLUMBO-002** | (reserved — no separate content DEC for a second persona because C-4 is single-persona) | Reserved slot in the DEC ID space; not allocated. The 100-series (DEC-C4-COLUMBO-101..104) carries the roadmap-closure decisions. |
| **DEC-C4-COLUMBO-003** | (reserved) | (see DEC-C4-COLUMBO-002) |
| **DEC-C4-COLUMBO-004** | columbo's `tool_preferences` entries are voice-affinity language ONLY (per §3.3 reference matrix): never selection instruction. WHOIS-anchored entry frames "the obvious question — who owns the place?"; crt.sh-anchored entry frames "checking who signed the guestbook." Existing C-1 test pattern (forbid `prefer ` prefix + `must use` substring; require ≥ 1 known CTI tool reference) is mirrored in `TestColumboProfileContent::test_columbo_profile_tool_preferences_content`. HARD GATE: `TestColumboPersonaSwapHardGates::test_columbo_swap_preserves_tool_call_identity` asserts tool-call sequence equality under columbo vs default using the deterministic mock-LLM harness ninja established. | The most important forbidden shortcut in the v2 design (DEC-30-CHARACTER-V2-005). Without the persona-swap hard gate, `tool_preferences` could drift into selection-biasing prose. With it, columbo is mechanically gated against tool-call divergence. The WHOIS framing case is especially worth gating — "the obvious question" could plausibly bias the LLM toward WHOIS over crt.sh; the test proves it does not. |
| **DEC-C4-COLUMBO-005** | (reserved — see DEC-C4-COLUMBO-103 for the context_hooks shape decision; 005 not allocated to keep numbering parallel to C-3's 005 = context_hooks decision) | Reserved; the actual context_hooks shape decision lives at DEC-C4-COLUMBO-103 to keep the 100-series for roadmap-closure scope. |
| **DEC-C4-COLUMBO-006** | columbo's `fourth_wall_stance="opaque"`. The schema as authored in C-1 uses `str` not `Literal`, with the docstring comment listing `in_character` / `winking` / `meta_aware` as the C-1-era recommendations. C-2 established `opaque` as a valid additional value for in-character-only personas; C-3 inherited; C-4 inherits. | columbo IS the detective — Peter Falk in a rumpled trenchcoat, not an LLM playing him. The "I'm probably just confused" deflection is in-character humility, not LLM-self-awareness. Meta-awareness would cheapen the register. `opaque` is the right semantic match. |

### 8.2 Roadmap-closure decisions (DEC-C4-COLUMBO-101..104)

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-C4-COLUMBO-101** | Tier-1 voice modes — `drunken_master`, `chuck_norris`, `bobby_hill` — are RECLASSIFIED from UPGRADE (per DEC-30-CHARACTER-V2-002) to terminal **KEEP_STATIC**. This is a SUPERSESSION of the disposition row for those three modes. No `LLMPersonaProfile` is authored for any of the three in C-4 or any future slice. They ship at `llm_profile=None` permanently. `TestTierOneModesPermanentlyStatic` (3 test methods) enforces this as a permanent invariant. | (1) No usage pattern across C-1/C-2/C-3 ships motivated upgrading these three. (2) The v1-composition-carrier test path at `tests/test_character_v2.py:388` + `tests/test_agent_tools.py:1597-1651` uses `drunken_master`; upgrading would force a two-file test repoint with no product gain. (3) Adding three more ≤165-token profile authoring tasks + ~90 new tests for zero motivated demand violates the minimal-codebase principle. (4) "Addition without subtraction is technical debt" does NOT apply — KEEP_STATIC is a TERMINAL DECISION not to extend an extensibility surface, not a new mechanism shipped in parallel. The static F62 voices for the three are already strong and self-consistent. (5) The final v2 catalog after C-4 is 6 UPGRADE (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo) + 4 KEEP_STATIC terminal (default, drunken_master, chuck_norris, bobby_hill) — the v2 character roadmap CLOSES definitively. There is no C-5. |
| **DEC-C4-COLUMBO-102** | The `mastery_level: int` hook on `LLMPersonaProfile` (deferred to C-4 per DEC-30-CHARACTER-V2-004) is RETIRED PERMANENTLY. The hook is not added in C-4 and no successor decision schedules it. The existing `test_mastery_level_not_present` gate at `tests/test_character_v2.py:162-167` is REPURPOSED as the permanent invariant; its docstring updates from "deferred to C-4" to "RETIRED PERMANENTLY per DEC-C4-COLUMBO-102 — no successor decision." No `core/persona_mastery.py` module is created. No SQLite column. No ScoreEvent action. | (1) C-1/C-2/C-3 shipped without `mastery_level`; no persona uses it; no test asks for it; no user feedback motivates it. (2) DEC-68-DOSSIER-REFRAME-005 retired XP-grind specifically to close score-grinding semantics from v2; reintroducing a per-mode integer that goes up over sessions is functionally equivalent to a score regardless of disclaimer prose. (3) Sacred Practice 12 parallel-authority risk: adding `mastery_level` introduces a second persona-depth axis distinct from the existing 8 fields, creating two authorities for "persona expressive depth." (4) Implementing the hook requires the field + per-mode level-1 variants + a session-count tracking module + a hooking-into-set_character mechanism — taking C-4 from "data-only" to "new module + new tracking surface + new schema field." Out of scope. (5) The roadmap explicitly authorized retirement: "C-4 planner may retire the hook entirely if the prior slices' usage patterns don't justify it." Three C-slices shipped; none justified it. |
| **DEC-C4-COLUMBO-103** | `columbo` is the FIRST persona in the v2 catalog to carry non-empty `context_hooks`. The schema field stays at C-1's `tuple[str, ...]` (no extension to `tuple[tuple[str,str], ...]` or any conditional-tuple shape). Each `context_hooks` entry is a STRING following the conditional-hint convention: `"when slot '<slot_name>' is <status>: '<voice line>'"` where `<slot_name>` is a value from `DossierSlotName` (e.g., `"identity"`, `"predictions"`, `"denial"`) and `<status>` is a value from `SlotStatus` (`"empty"`, `"partial"`, `"filled"`, `"deferred"`). The LLM reads the string as natural-language guidance; no runtime condition evaluation. The runner.py composer (line 379) joins the tuple with `"; "` verbatim — no runner change needed. `TestColumboDossierAwareContextHooks` (3 test methods) gates the convention by substring-asserting slot + status vocabulary presence. | (1) The C-1 schema is sufficient. A hint like `"when slot 'identity' is empty: 'just one more thing — have we got a name yet?'"` is a single string that the LLM reads as guidance. (2) DEC-30-CHARACTER-V2-003 says the ±2-field refinement window closed at C-1; the schema is frozen. (3) A tuple-of-tuples schema would force C-1/C-2/C-3's existing empty `()` to a different shape, forcing a backward-incompatible refactor for zero behavioral gain. (4) The runner.py composer already renders `tuple[str, ...]` correctly; a new schema would require runner.py changes, violating DEC-C2-NINJA-002 inheritance. (5) The dossier slot vocabulary is read-only-referenced via STRING LITERALS — no `from adversary_pursuit.dossier.slots import ...` is added to `gamification/modes.py`; the dossier module stays untouched. The test class TestColumboDossierAwareContextHooks imports the enums for the assertion, but production code does not. |
| **DEC-C4-COLUMBO-104** | The 5 existing v2 personas (`full_troll`, `ninja`, `sun_tzu`, `bruce_lee`, `bureaucrat`) keep `context_hooks=()`. C-4 does NOT retrofit dossier-aware hooks onto them. Their existing `()` byte-stability is preserved through C-4. | (1) Only columbo's voice idiom ("just one more thing… have we checked the WHOIS?") is uniquely dossier-aware-shaped — the other personas' voices do not pivot off case-state knowledge (sun_tzu does not say "ah, but as for slot 9"). (2) Retrofitting 5 personas would force C-4 to re-validate 5 byte-stable C-1/C-2/C-3 profiles. (3) Minimal-codebase: add the surface where motivated, not preemptively. (4) Future demand (e.g., user-driven feedback that another persona's voice benefits from dossier-state hooks) can populate them in a successor slice — DEC-C4-COLUMBO-103 establishes the convention. |

---

## 9. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Token budget overrun at first draft | Very high (estimate ~240; ~75 over budget) | Low (named trim paths in §3.1 are exhaustive; context_hooks protected) | Implementer iterates trim steps 1-7 → test until green. Trim path is designed to preserve the three context_hooks (the C-4 contribution) while shaving voice_summary / signature_phrases / forbidden_voice. |
| Persona-swap test reveals tool-selection bias from WHOIS framing | Medium (the "obvious question" framing could plausibly bias toward WHOIS over crt.sh) | Medium (would force tool_preferences re-author) | If `test_columbo_swap_preserves_tool_call_identity` fails, the implementer iterates `tool_preferences` wording (rephrase WHOIS framing as `"WHOIS: where the obvious questions get answered"` or weaker affinity). The mock-LLM is deterministic so failure is reproducible; the fix is content-only. |
| Dossier-vocabulary substring test fails because trim shortened `"predictions"` away | Low (the trim path step 7 only shortens hint #3, not removes the predictions reference) | High (the C-4 load-bearing contract — dossier substrate reference — would not hold) | Implementer MUST verify `TestColumboDossierAwareContextHooks::test_columbo_context_hooks_reference_real_slot_names` passes AFTER every trim step. The three slot references (`identity`, `predictions`, `denial`) and at least 2 status references (`empty`, `partial`, `filled`) MUST survive trim. If trim accidentally drops a slot reference, the implementer reverts the trim step and chooses a different one (e.g., shorten `voice_summary` further instead of touching `context_hooks`). |
| LLM produces a "+N points" string at runtime despite forbidden_voice guard | Low (F64 guard is well-tested across C-1/C-2/C-3) | Medium (F64 panel-separation regression) | `TestColumboF64PanelSeparation::test_columbo_does_not_smuggle_point_totals` runs the deterministic mock-LLM with columbo system prompt and asserts no `point` / `pts` / `score` substring in the LLM-facing summary. If it fails, the forbidden_voice F64 guard is the first authorship target. |
| MASTER_PLAN.md edit collides with parallel M-9 update | Low (M-9 has not been opened; `cc-policy lease summary` shows only C-4 + a stale planner lease) | Medium (rebase + reauthor Phase 17M numbering) | Worktree is at C-3 merge `3f33a5b`; no parallel implementer workflow is active. If M-9 starts in parallel, the implementer rebases and renumbers Phase 17M → Phase 17N as needed. |
| Active Phase Pointer regex breaks if Phase 17M heading style deviates | Low | Low (status board reads stale until line is re-pointed) | Implementer mirrors the exact `**Phase Active (YYYY-MM-DD — ...):**` boldline format used by Phase 17L's pointer; the `get_plan_status()` regex `^#.*phase|^**Phase` matches the leading `**Phase` literal. |
| Implementer accidentally upgrades drunken_master to fix a perceived "open" disposition | Medium (dispatch context mentioned the disposition table looks "incomplete" — could mislead) | High (forces two-file v1-carrier test repoint; opens v2 catalog; reverses DEC-C4-COLUMBO-101) | `TestTierOneModesPermanentlyStatic::test_drunken_master_terminally_static` asserts `llm_profile is None` — mechanically prevents the upgrade. DEC-C4-COLUMBO-101 in the plan + Phase 17M is the binding terminal disposition. The forbidden_shortcut entry "DO NOT author LLMPersonaProfile entries for drunken_master, chuck_norris, or bobby_hill" is explicit. |
| Implementer accidentally adds `mastery_level` field to LLMPersonaProfile thinking C-4 implements it | Medium (DEC-30-CHARACTER-V2-004 says "may retire OR implement"; implementer may default to implement) | High (schema mutation; opens parallel-authority surface; violates Sacred Practice 12) | `test_mastery_level_not_present` asserts the field is absent; the docstring update to "RETIRED PERMANENTLY per DEC-C4-COLUMBO-102" is the signal; the forbidden_shortcut entry "DO NOT introduce a mastery_level field" is explicit. |

---

## 10. Open follow-ups (recorded; not part of C-4 scope)

These are explicitly OUT of C-4 scope and recorded here so a future planner can pick them up cleanly without duplicating analysis.

1. **M-9 — Crowdsourced Dossier Comparison + Public Actor Library.** Per DEC-68-DOSSIER-REFRAME-009. This is the next scheduled slice after C-4 lands. Touches `dossier/*` + adds a cross-workspace public-actor cache. Orthogonal to character system.

2. **context_hooks retrofit for the 5 existing v2 personas.** DEC-C4-COLUMBO-104 deferred this; a future slice may populate `context_hooks` on full_troll, ninja, sun_tzu, bruce_lee, bureaucrat if usage patterns warrant. The pattern is established by C-4 (conditional-hint string convention referencing DossierSlotName + SlotStatus vocabulary). Each retrofit would be a single-persona data slice with mirror tests.

3. **Persona-mode-compare meta-command.** Out-of-scope per DEC-30-CHARACTER-V2-006 (roadmap §6 C-1 "optional polish"). If user feedback motivates it, a future slice can surface `mode compare <a> <b>` that runs the same prompt under two personas. Low priority.

4. **Persona persistence across sessions.** Out-of-scope per Phase 17 §8. Today the active mode at session start is always `default`. A future slice could add workspace-or-config-level last-active-mode preference. Touches `core/config.py` + `core/console.py` + chat startup; not in C-4 scope.

5. **Runtime hygiene backlog (#49, #50, #51).** Documented in MASTER_PLAN.md "Runtime Hygiene Backlog" section. Opportunistic — whoever hits one first files the slice.

6. **mastery_level resurrection (NOT ALLOWED).** Recorded explicitly: DEC-C4-COLUMBO-102 retires the hook PERMANENTLY. Any future "let's add a mastery level" proposal must first supersede DEC-C4-COLUMBO-102 via a new strategic-scoping slice — not via an in-flight implementer slice.

---

## 11. Cross-references

- **Issue #30** — source product directive (https://github.com/jarocki/ap/issues/30).
- **`.claude/plans/character-v2-roadmap.md`** — strategic scoping (Phase 17). §3 personality schema, §4 per-mode disposition, §6 C-1..C-4 decomposition, §6.5 sequencing relative to #68.
- **MASTER_PLAN.md §Phase 17** — strategic scoping landing; DEC-30-CHARACTER-V2-001..007.
- **MASTER_PLAN.md §Phase 17C** — C-1 MVP landing; DEC-C1-FULLTROLL-001..005; `LLMPersonaProfile` dataclass + `set_character` composer + `full_troll` profile.
- **MASTER_PLAN.md §Phase 17E** — C-2 landing; DEC-C2-NINJA-001..003; `ninja` profile + `opaque` fourth-wall stance + KEEP_STATIC → UPGRADE supersession for ninja + test mirror pattern.
- **MASTER_PLAN.md §Phase 17L** — C-3 landing; DEC-C3-PHILOSOPHY-001..006; sun_tzu/bruce_lee/bureaucrat profiles; AP #74 orphan-prevention single-commit pattern; merge `3f33a5b` / impl `e4f7ffe`.
- **MASTER_PLAN.md §Phase 17G** — M-4 landing; DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006; dossier persistent state substrate (`dossier/state.py::load_dossier_state`) that columbo `context_hooks` reference as vocabulary.
- **src/adversary_pursuit/gamification/modes.py** — sole edit site. `LLMPersonaProfile` frozen dataclass + `CharacterMode.llm_profile` field + `DEFAULT_MODES` dict + `ModeManager`.
- **src/adversary_pursuit/agent/runner.py:342-401** — sole consumption site (`set_character`). BYTEWISE UNCHANGED through C-4 (inherits DEC-C2-NINJA-002).
- **src/adversary_pursuit/dossier/state.py** — M-4 substrate. `load_dossier_state(workspace_mgr) -> DossierState | None`.
- **src/adversary_pursuit/dossier/slots.py** — `DossierSlotName` 9-slot enum + `SlotStatus` 4-value enum. The string VALUES of these enums (e.g., `"identity"`, `"empty"`) appear verbatim in columbo's `context_hooks` strings.
- **tests/test_character_v2.py** — mirror test pattern. C-4 adds 5 new test classes mirroring C-3 patterns + adds two new test classes specific to C-4 (TestColumboDossierAwareContextHooks + TestTierOneModesPermanentlyStatic).
- **tests/test_agent_tools.py:1597-1651** — v1-composition carrier path. UNCHANGED in C-4 by DEC-C4-COLUMBO-101 design (drunken_master stays at `llm_profile=None`).
- **DEC-30-CHARACTER-V2-001..007** — Phase 17 binding decisions; C-4 extends and supersedes DEC-30-CHARACTER-V2-002 (disposition for 3 modes) and DEC-30-CHARACTER-V2-004 (mastery deferral).
- **DEC-C1-FULLTROLL-001..005** — Phase 17C binding decisions; C-4 inherits schema + composer + test patterns.
- **DEC-C2-NINJA-001..003** — Phase 17E binding decisions; C-4 inherits opaque stance + byte-identity discipline.
- **DEC-C3-PHILOSOPHY-001..006** — Phase 17L binding decisions; C-4 inherits author-and-mirror pattern + AP #74 single-commit discipline.
- **DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006** — Phase 17G binding decisions; C-4 references the slot-state vocabulary substrate they established.
- **DEC-68-DOSSIER-REFRAME-005** — XP-grind retirement; cited as supporting rationale for DEC-C4-COLUMBO-102 mastery_level permanent retire.
