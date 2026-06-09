# C-3 — Philosophy + Bureaucratese Modes (sun_tzu + bruce_lee + bureaucrat) — per-slice plan

**Status:** planner-staged 2026-06-09 by W-30-C3-PHILOSOPHY-BUREAUCRAT planner stage. Implementer slice `wi-30-c3-impl-01` to follow.
**Workflow:** `w-30-c3-philosophy-bureaucrat`
**Goal:** `g-30-c3-philosophy`
**Work item to dispatch:** `wi-30-c3-impl-01`
**Drives:** Phase 17L of `MASTER_PLAN.md`. Phase 17L carries the binding decisions and slice index; this document carries the full rationale, per-mode profile content, voice-affinity rationale, and acceptance-test choreography. When the two diverge, Phase 17L wins for binding decisions; this document wins for narrative.

**Inherits from:** Phase 17 (W-30-CHARACTER-V2-SCOPING; `.claude/plans/character-v2-roadmap.md`), Phase 17C (C-1 MVP; W-30-C1-FULL-TROLL-PROFILE; `LLMPersonaProfile` dataclass + `set_character` composer + `full_troll` profile), Phase 17E (C-2; W-30-C2-NINJA-PROFILE; `ninja` profile + mirror test suites + KEEP_STATIC → UPGRADE supersession for ninja).
**Worktree base:** AP main at merge `16acaa3` (M-8 merge head; impl `6c87a53`). C-3 has zero dependency on M-8 or any dossier slice; the worktree happens to be cut from M-8's merge because that is current `main`. C-3 is a `gamification/modes.py` data-only extension; it does not touch any dossier surface.

---

## 1. Goal (single paragraph)

C-3 adds three `LLMPersonaProfile` entries to `src/adversary_pursuit/gamification/modes.py` — one each for `sun_tzu`, `bruce_lee`, and `bureaucrat` — completing the C-3 slice of the v2 character roadmap. The profiles are pure data extensions on top of the C-1-frozen `LLMPersonaProfile` dataclass (8 fields, ≤ 165 tokens per mode). No code path changes: `runner.set_character` already detects `mode.llm_profile is not None` and composes the structured profile into the system prompt (the C-1 `if` branch fires for every C-3 profile by construction). After C-3 lands, 4 of 10 modes carry `llm_profile` entries (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat — wait, that's 5; see §2 disposition table); the remaining 5 modes (default + drunken_master + chuck_norris + bobby_hill + columbo) continue to ship at `llm_profile=None` until C-4 closes the catalog. Tests mirror the C-2 ninja suites verbatim: per-profile content assertions, ≤ 165-token budget gate, persona-swap-tool-call-identity hard gate extended over the three new modes (DEC-C1-FULLTROLL-004), F62 `run_fail` single-authority invariants, F64 panel-separation guards.

**Out-of-scope (explicit, deferred):**

- **No `agent/runner.py` modification.** C-1 already built the integration site (`runner.py:342-401`); C-2 verified it handles any non-None profile (`runner.py` was byte-identical in C-2 — DEC-C2-NINJA-002). C-3 inherits that wiring; the `llm_profile is not None` branch fires for each of the three new profiles by construction. `runner.py` MUST be byte-identical post-C-3 (same as C-2's invariant). Verified via the cleanup-audit grep test (§5.3).
- **No schema refinement.** DEC-30-CHARACTER-V2-003's ±2-field refinement window closed at C-1. The schema is frozen at the 8 fields C-1 + C-2 ship today (`voice_summary`, `tone_registers`, `signature_phrases`, `fourth_wall_stance`, `dialect_cadence`, `context_hooks`, `tool_preferences`, `forbidden_voice`). C-3 MUST NOT add, remove, rename, or retype any field. `mastery_level` field MUST NOT appear (deferred to C-4 per DEC-30-CHARACTER-V2-004 — the existing `test_mastery_level_not_present` gate at `tests/test_character_v2.py:161-166` enforces this).
- **No new `CharacterMode` fields.** `CharacterMode.llm_profile: LLMPersonaProfile | None = None` is the v2 schema and remains unchanged. C-3 only sets `llm_profile=...` on three existing dict entries inside `DEFAULT_MODES`.
- **No edit to `core/streak.py`.** F62 streak-authority invariant; the `test_streak_manager_module_not_imported_by_modes_module` gate at `tests/test_character_v2.py:693-708` enforces it (and grows naturally — modes.py adds three more profile authors, but no streak-related symbol).
- **No edit to `agent/tools.py`.** F62 `run_fail` wiring authority; the `test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline` gate enforces that `LLMPersonaProfile` is not referenced and `llm_profile` is not referenced anywhere in `tools.py`.
- **No edit to gamification scoring surfaces.** `gamification/scoring.py`, `gamification/celebrations.py`, `gamification/dossier_celebrations.py`, `gamification/hints.py`, `gamification/challenges.py`, `gamification/badges.py`, `gamification/dossier_badges.py` BYTEWISE UNCHANGED. The persona is not a gameable surface (DEC-30-CHARACTER-V2-005 §8 — no new gamification events).
- **No new LLM tool.** Tool count stays at 28 (post-M-8 floor). The persona is system-prompt data, not a tool surface.
- **No new module.** DEC-30-CHARACTER-V2-003 mandates `set_character` as the single integration site. C-3 extends only `gamification/modes.py`'s `DEFAULT_MODES` dict.
- **No dossier-package modification.** C-3 has zero dossier coupling. `dossier/*.py` BYTEWISE UNCHANGED. The roadmap's §6.5 reservation that columbo's `context_hooks` may reference dossier slot state lands at C-4, not C-3.
- **No `context_hooks` referencing dossier slot state.** Per DEC-C1-FULLTROLL-005 / DEC-C2-NINJA-001 pattern, C-3 ships `context_hooks=()` for all three new profiles (deferred to C-4 alongside the columbo dossier-aware hook decision). **See §3.4 for the C-3 decision to keep `context_hooks=()` empty rather than seed generic non-dossier hooks** — the dispatch context invited C-3 to optionally seed 1–2 generic hooks per mode; we reject that invitation to preserve a clean C-4 surface and avoid two-stage authoring drift.
- **No `core/console.py` / `agent/chat.py` modification.** cmd2 path stays on the F62 Rich-panel-voice surface (DEC-C1-FULLTROLL-003 / roadmap §8). The agent path is the only v2 surface and is wired through `set_character` (no `chat.py` or `console.py` edit required because C-3 is data-only).
- **No mid-test persona swap that asserts streak counters.** The dispatch context invited a persona-swap streak-corruption test (starting in sun_tzu and switching to bureaucrat); we reject that test as duplicate coverage of the existing F62 invariant test pattern. The existing `test_streak_manager_module_not_imported_by_modes_module` gate already mechanically prevents `gamification/modes.py` from acquiring any streak coupling, which is the actual invariant. A persona-swap test that explicitly drives `StreakManager` would force `tests/test_character_v2.py` to import streak machinery — exactly what F62 forbids the modes module from doing. The invariant is preserved by architectural disconnection, not by adversarial test.
- **No edit to `core/workspace.py` / `models/database.py` / `core/event_bus.py` / `core/pivot_policy.py` / `core/dossier_pivot.py` / `core/dossier_report.py` / `dossier/novelty.py` (M-8) / `dossier/*` / `agent/chat.py` / `core/console.py`.** Scope manifest forbids all of these — see §4.

---

## 2. Architecture

### 2.1 Layering authority — data-only extension at a single integration site

```
+----------------------------------------------------------------------+
|  Sole edit site: src/adversary_pursuit/gamification/modes.py         |
|                                                                      |
|  EDIT (extend DEFAULT_MODES dict — three entries gain llm_profile):  |
|    "sun_tzu":    CharacterMode(... , llm_profile=LLMPersonaProfile(  |
|                    voice_summary=..., tone_registers=...,            |
|                    signature_phrases=..., fourth_wall_stance=...,    |
|                    dialect_cadence=..., context_hooks=(),            |
|                    tool_preferences=..., forbidden_voice=...))       |
|    "bruce_lee": CharacterMode(... , llm_profile=LLMPersonaProfile(...|
|    "bureaucrat":CharacterMode(... , llm_profile=LLMPersonaProfile(...|
|                                                                      |
|  ADD module docstring @decision entries: DEC-C3-PHILOSOPHY-001..005  |
|                                                                      |
|  All other DEFAULT_MODES entries BYTEWISE UNCHANGED:                 |
|    default, drunken_master, chuck_norris, bobby_hill, columbo         |
|     (continue to ship at llm_profile=None — F62 behavior preserved)  |
+----------------------------------------------------------------------+
                            |
                            v
+----------------------------------------------------------------------+
|  Sole consumption site: AgentRunner.set_character (runner.py:342-401)|
|  BYTEWISE UNCHANGED. C-1's `if mode.llm_profile is not None:` branch |
|  already handles every non-None profile.                             |
+----------------------------------------------------------------------+

+----------------------------------------------------------------------+
|  Test extensions: tests/test_character_v2.py                         |
|                                                                      |
|  EXTEND test_llm_profile_default_is_none_for_all_modes               |
|    upgraded_modes: {"full_troll", "ninja"} ->                        |
|                    {"full_troll", "ninja", "sun_tzu", "bruce_lee",   |
|                     "bureaucrat"}                                    |
|                                                                      |
|  ADD: TestSunTzuProfileContent (mirrors TestNinjaProfileContent)     |
|  ADD: TestBruceLeeProfileContent (mirrors TestNinjaProfileContent)   |
|  ADD: TestBureaucratProfileContent (mirrors TestNinjaProfileContent) |
|                                                                      |
|  ADD: TestSunTzuPersonaSwapHardGates                                 |
|       (mirrors TestNinjaPersonaSwapHardGates)                        |
|  ADD: TestBruceLeePersonaSwapHardGates                               |
|  ADD: TestBureaucratPersonaSwapHardGates                             |
|                                                                      |
|  ADD: TestSunTzuF64PanelSeparation                                   |
|       (mirrors TestNinjaF64PanelSeparation)                          |
|  ADD: TestBruceLeeF64PanelSeparation                                 |
|  ADD: TestBureaucratF64PanelSeparation                               |
|                                                                      |
|  EXISTING tests that grow naturally without rewrite:                 |
|    test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline  |
|    test_run_fail_field_still_consumed_at_tools_py_1622_1628          |
|    test_streak_manager_module_not_imported_by_modes_module           |
|    test_hint_style_not_reintroduced                                  |
|    test_mastery_level_not_present                                    |
|    test_default_mode_keeps_static (default stays None — C-4 closes)  |
|    test_set_character_drunken_master_uses_v1_composition_verbatim    |
|      (drunken_master remains the v1-composition carrier through C-3) |
+----------------------------------------------------------------------+
```

### 2.2 Why this slice is one file + one test file

C-3 is a data-only extension on top of two prior implementer slices (C-1 + C-2). The dataclass shape, the injection wiring, the v1/v2 branching logic, the per-mode token-budget gate, the persona-swap hard gate, the F62/F64 invariants, and the test patterns are all already implemented and shipping on `main`. C-3's work is:

1. Author three `LLMPersonaProfile` entries with voice content that matches each mode's existing static voice (Sun Tzu strategic gnomic; Bruce Lee flow-state water; bureaucrat dry corporate-form).
2. Update one test fixture (`upgraded_modes` set in `test_llm_profile_default_is_none_for_all_modes`) and add three test classes per mode (content + persona-swap + F64) following the C-2 ninja template verbatim.

Everything else is leveraged from existing code. The slice's total LOC is dominated by the three profile content blocks (≈ 35-45 LOC per profile in the dict; ≈ 105-135 LOC total in `modes.py`) plus the three test class triples (≈ 80-100 LOC per mode; ≈ 240-300 LOC total in `test_character_v2.py`). All other production code paths are read-only inputs to the slice.

### 2.3 Why no `runner.py` edit (Sacred Practice 12 / DEC-C2-NINJA-002 inheritance)

The C-1 implementer authored the system-prompt composition template in `set_character` to be field-driven, not mode-name-driven. When a new mode acquires an `llm_profile`, no `runner.py` change is needed — the existing `if mode.llm_profile is not None:` branch fires for any non-None profile and composes the same template with the new mode's fields. C-2 verified this empirically: `runner.py` was byte-identical between C-1 ship and C-2 ship. C-3 inherits that property; the slice's Evaluation Contract asserts `runner.py` is byte-identical post-C-3 (same as C-2).

This is the load-bearing reason C-3 is data-only. If C-1's composition template had been mode-name-keyed (e.g. `if mode.name == "full_troll": ... elif mode.name == "ninja": ...`), every new mode would require a `runner.py` edit and each subsequent C-slice would carry parallel-authority risk. The C-1 implementer's field-driven design is what makes C-2/C-3/C-4 each a pure-data slice.

### 2.4 State-authority map

C-3 touches one runtime state domain: **persona profile authority** (the catalog of `LLMPersonaProfile` instances bound to `CharacterMode` entries in `DEFAULT_MODES`).

| State domain | Canonical authority (post-C-3) | C-3 mutation |
|---|---|---|
| Persona profile catalog | `gamification/modes.py::DEFAULT_MODES` | 3 dict entries gain `llm_profile=LLMPersonaProfile(...)` |
| Persona injection composer | `agent/runner.py::AgentRunner.set_character` | BYTEWISE UNCHANGED (inherits C-1) |
| Persona schema | `gamification/modes.py::LLMPersonaProfile` (frozen dataclass) | BYTEWISE UNCHANGED (inherits C-1) |
| Streak authority | `core/streak.py::StreakManager` | UNCHANGED (F62 invariant; modes.py never imports streak) |
| Run-fail voice authority | `gamification/modes.py::CharacterMode.run_fail` (data) + `agent/tools.py` (consumer) | UNCHANGED (F62 invariant; persona profile does not touch run_fail wiring) |
| Rich-panel gamification narration | `gamification/celebrations.py` + `gamification/dossier_celebrations.py` | UNCHANGED (F64 invariant; persona profile does not narrate point totals) |
| LLM tool catalog | `agent/tools.py::create_tools` | UNCHANGED (tool count stays at 28) |
| Tool selection bias | "Owned by LLM weighed against tool descriptions; persona MUST NOT bias" | NOT TOUCHED (`tool_preferences` is voice-affinity only — DEC-30-CHARACTER-V2-005; persona-swap-tool-call-identity test enforces) |

No new state domains. No new authorities. No migration needed (the field is optional with default None; existing serialized data structures are unaffected — `CharacterMode` is a frozen dataclass and is constructed in code, not deserialized).

---

## 3. Per-profile content authoring (binding)

This section is the authoritative reference for the three profile content blocks. The implementer copies the field values verbatim from §3.1–§3.3. Any deviation requires a planner re-stage and a successor DEC-ID. Token budgets are estimated using the 4-chars-per-token heuristic the existing C-1/C-2 token-budget test uses (`tests/test_character_v2.py::_rough_token_count`).

### 3.1 `sun_tzu` profile (DEC-C3-PHILOSOPHY-001)

**Voice anchor:** Strategic Sun Tzu quotes for every action (F62 personality).
**Static Rich-panel template carriers:** `'"Know thy enemy and know thyself."'` (greeting), `'"Opportunities multiply as they are seized."'` (run_success), `'"In the midst of chaos, there is also opportunity."'` (run_fail), `'"Supreme excellence."'` (score_celebration).
**v2 extension intent:** LLM pulls context-appropriate Art of War quotes from a wider pool than the static catalog allows. Strategist register — oblique, gnomic, second-person guidance.

```python
llm_profile=LLMPersonaProfile(
    voice_summary=(
        "Strategist who frames every observation through Art of War —"
        " oblique, patient, second-person guidance."
    ),
    tone_registers=("gnomic", "strategic", "patient", "oblique"),
    signature_phrases=(
        "Know thy enemy.",
        "Opportunities multiply as they are seized.",
        "Victory is reserved for those who pay the price.",
        "In the midst of chaos, opportunity.",
        "Supreme excellence is subduing without fighting.",
    ),
    # opaque: sun_tzu is the role — no LLM/tool acknowledgement.
    # Mirrors ninja DEC-C2-NINJA-001 stance choice for in-character personas.
    fourth_wall_stance="opaque",
    dialect_cadence=(
        "Short aphoristic lines; quote-then-application;"
        " second-person 'you' for tactical guidance; no modern slang."
    ),
    # context_hooks: empty per DEC-C3-PHILOSOPHY-005 — deferred to C-4 alongside
    # columbo's dossier-aware hook decision. Mirrors DEC-C1-FULLTROLL-005 + DEC-C2-NINJA-001.
    context_hooks=(),
    # tool_preferences: voice-affinity ONLY — phrased as strategist's framing,
    # never as selection instruction. Persona-swap-tool-call-identity test
    # gates this invariant (DEC-30-CHARACTER-V2-005).
    tool_preferences=(
        "crt.sh: reconnaissance of the enemy's terrain before engagement",
        "VirusTotal: the verdict of many spies, weighed with discernment",
    ),
    # forbidden_voice: F64 panel-separation guard + voice-register guards
    # preventing drift toward modern-snark personas.
    forbidden_voice=(
        "never narrate point totals — the Rich panel owns scoring",
        "never use modern slang, memes, or exclamations",
        "never quote sources other than Sun Tzu's Art of War",
    ),
)
```

**Token budget estimate (4-chars-per-token heuristic):**

| Field | Char count (approx) | Token estimate |
|---|---|---|
| voice_summary | ~105 | ~26 |
| tone_registers (4 entries joined) | ~38 | ~10 |
| signature_phrases (5 entries joined) | ~170 | ~43 |
| fourth_wall_stance | 6 | ~2 |
| dialect_cadence | ~110 | ~28 |
| context_hooks | 0 | 0 |
| tool_preferences (2 entries joined) | ~115 | ~29 |
| forbidden_voice (3 entries joined) | ~150 | ~38 |
| **Total (sum + 7 join spaces)** | **~700** | **~175** |

Token estimate exceeds 165 by ~10 — implementer MUST trim during authoring. Concrete trim path (apply in order until ≤ 165):

1. Drop the 5th signature_phrase ("Supreme excellence is subduing without fighting.") — saves ~12 tokens.
2. If still over: shorten voice_summary to "Strategist who frames observations through Art of War — oblique, patient guidance." (saves ~6 tokens).
3. If still over: drop 2nd tool_preference — saves ~16 tokens.

The implementer MUST iterate trim → run `tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_token_budget` → repeat until green. The exact final wording is the implementer's authorship within the trim-path constraints; the content-assertion tests (§5.1) bind the *semantic* content, not the exact strings.

### 3.2 `bruce_lee` profile (DEC-C3-PHILOSOPHY-002)

**Voice anchor:** Bruce Lee philosophy, flow-state zen commentary (F62 personality).
**Static Rich-panel template carriers:** `'"Be water, my friend."'` (greeting), `'"I fear not the man who has practiced 10,000 kicks once."'` (run_success), `'"Don\'t fear failure."'` (run_fail), `"Flow state!"` (score_celebration).
**v2 extension intent:** LLM enriches the flow-state philosophy beyond static templates — water/formless metaphors; adaptive iteration; movement-as-investigation framing.

```python
llm_profile=LLMPersonaProfile(
    voice_summary=(
        "Flow-state philosopher: water metaphors for investigation;"
        " adapts to data shape; movement before form."
    ),
    tone_registers=("zen", "flowing", "focused", "philosophical"),
    signature_phrases=(
        "Be water, my friend.",
        "Don't fear failure.",
        "Take what is useful, discard what is not.",
        "10,000 kicks, once.",
        "Empty your mind — formless, shapeless.",
    ),
    fourth_wall_stance="opaque",
    dialect_cadence=(
        "Short declarative sentences; nature-metaphor framing;"
        " second-person 'you' for guidance; pauses where Western prose would rush."
    ),
    # context_hooks: empty per DEC-C3-PHILOSOPHY-005 — same deferral as sun_tzu.
    context_hooks=(),
    # tool_preferences: voice-affinity ONLY — flow-state framing of recon surfaces.
    # Phrased so persona-swap-tool-call-identity test passes.
    tool_preferences=(
        "crt.sh: the river of certificate history flowing past",
        "DNS resolution: each query a ripple in the network's surface",
    ),
    forbidden_voice=(
        "never narrate point totals — the Rich panel owns scoring",
        "never use sarcasm, snark, or exclamation-driven hype",
        "never quote philosophers other than Bruce Lee",
    ),
)
```

**Token budget estimate (4-chars-per-token heuristic):**

| Field | Char count (approx) | Token estimate |
|---|---|---|
| voice_summary | ~105 | ~26 |
| tone_registers | ~32 | ~8 |
| signature_phrases (5 entries joined) | ~150 | ~38 |
| fourth_wall_stance | 6 | ~2 |
| dialect_cadence | ~135 | ~34 |
| context_hooks | 0 | 0 |
| tool_preferences (2 entries joined) | ~115 | ~29 |
| forbidden_voice (3 entries joined) | ~135 | ~34 |
| **Total** | **~680** | **~171** |

Token estimate exceeds 165 by ~6 — implementer trim path:

1. Drop the 5th signature_phrase ("Empty your mind — formless, shapeless.") — saves ~12 tokens.
2. If still over: shorten dialect_cadence to "Short declarative sentences; nature-metaphor framing; second-person 'you' for guidance." (saves ~10 tokens).

### 3.3 `bureaucrat` profile (DEC-C3-PHILOSOPHY-003)

**Voice anchor:** Office Space vibes, everything is a TPS report (F62 personality).
**Static Rich-panel template carriers:** `"Please sign form TPS-001 before proceeding. In triplicate."` (greeting), `"Results filed under Form IR-7734."` (run_success), `"Your request has been denied. Please submit Form ERR-404 to the help desk."` (run_fail), `"Per Policy §4.2.1, you have been awarded +{points} compliance points."` (score_celebration).
**v2 extension intent:** Heavy idiom load — form numbers, policy section numbers, in-triplicate phrasing. LLM extends without the static-template author having to enumerate every form-name combination.

```python
llm_profile=LLMPersonaProfile(
    voice_summary=(
        "Office-Space-grade compliance officer: every observation"
        " is a form filing; every conclusion cites a policy section."
    ),
    tone_registers=("dry-corporate", "procedural", "deadpan", "officious"),
    signature_phrases=(
        "Per Policy §4.2.1, ...",
        "Please file under Form IR-7734.",
        "In triplicate, naturally.",
        "Submit Form ERR-404 to the help desk.",
        "Routing this to Compliance for review.",
    ),
    # opaque: bureaucrat is the role — the persona is the form-filer,
    # never the LLM/tool. Mirrors ninja/sun_tzu/bruce_lee stance.
    fourth_wall_stance="opaque",
    dialect_cadence=(
        "Sentences open with policy citation or form number;"
        " passive voice preferred; no contractions; section-numbered lists where natural."
    ),
    context_hooks=(),
    # tool_preferences: voice-affinity ONLY — bureaucratic framing of recon surfaces.
    # Intentionally less colorful than sun_tzu/bruce_lee — bureaucrat is the
    # "intentional gray voice" of the catalog. Persona-swap test gates the
    # tool-selection-bias invariant.
    tool_preferences=(
        "crt.sh: Form CT-3 (Certificate Transparency Submission, public registry)",
        "WHOIS: Form WH-1 (Domain Registration Disclosure, see Appendix A)",
    ),
    forbidden_voice=(
        "never narrate point totals — the Rich panel owns scoring",
        "never use slang, contractions, or exclamation marks",
        "never break character to acknowledge the persona is bureaucratic",
    ),
)
```

**Token budget estimate (4-chars-per-token heuristic):**

| Field | Char count (approx) | Token estimate |
|---|---|---|
| voice_summary | ~120 | ~30 |
| tone_registers | ~46 | ~12 |
| signature_phrases (5 entries joined) | ~170 | ~43 |
| fourth_wall_stance | 6 | ~2 |
| dialect_cadence | ~145 | ~36 |
| context_hooks | 0 | 0 |
| tool_preferences (2 entries joined) | ~140 | ~35 |
| forbidden_voice (3 entries joined) | ~165 | ~41 |
| **Total** | **~800** | **~200** |

Token estimate exceeds 165 by ~35 — bureaucrat is the most over-budget of the three because the corporate-form voice naturally inflates wording. Implementer trim path:

1. Drop the 5th signature_phrase ("Routing this to Compliance for review.") — saves ~10 tokens.
2. Drop the 3rd forbidden_voice entry ("never break character to acknowledge the persona is bureaucratic") — the opaque stance already covers this. Saves ~16 tokens.
3. Shorten dialect_cadence to: "Sentences open with policy citation or form number; passive voice preferred; no contractions." Saves ~10 tokens.

After all three trims, estimated tokens ~164. The implementer iterates with `_rough_token_count` until the budget test passes. The content-assertion tests (§5.1) bind the *semantic* content — at minimum: a TPS/IR/CT/WH-style form number AND a `§` policy-section reference AND no exclamation marks.

### 3.4 `context_hooks=()` decision (DEC-C3-PHILOSOPHY-005)

The dispatch context explicitly invited C-3 to optionally seed 1–2 generic non-dossier `context_hooks` per mode. **We reject that invitation.** All three C-3 profiles ship with `context_hooks=()`, mirroring DEC-C1-FULLTROLL-005 + DEC-C2-NINJA-001.

**Rationale (4 points):**

1. **C-4 owns the `context_hooks` design surface.** The roadmap §6 explicitly reserves `context_hooks` as the dossier-coupling experiment for `columbo` — C-4's voice IS the investigative-detective register that motivates dossier-aware hooks. Seeding generic hooks in C-3 would force C-4 to either inherit the generic-hook design (constraining C-4's freedom to author dossier-aware hooks fresh) or to retire C-3's hooks (creating two-stage authoring drift). Empty tuple preserves a clean C-4 surface.
2. **Generic non-dossier hooks are an unfalsifiable design space.** A hook like "escalating victim count → escalating gnomic concern" sounds reasonable but has no observable behavior to test (the LLM may or may not honor it; there's no mechanical gate that proves the hook matters). The hook becomes prose-in-data — exactly the doc-lie pattern F62 deleted. Until there is dossier slot state to bind hooks to (post-M-1 onward, but only relevant for the columbo persona in C-4), empty is the honest default.
3. **C-1 and C-2 both shipped `context_hooks=()` and the chat UX is acceptable.** Field testing of the full_troll and ninja personas through M-1..M-8's lifetime did not surface a user complaint that the persona "fails to respond to investigation context". The static voice + the structured profile (voice_summary, tone_registers, dialect_cadence) carries the persona register well enough without `context_hooks`. C-3 inherits that confidence.
4. **Token budget pressure.** All three C-3 profiles are at or over the 165-token budget in their full-content drafts (§3.1–§3.3). Adding 1–2 hooks per mode would inflate every profile by ~30-40 tokens, forcing more aggressive trims elsewhere and reducing the semantic richness of the fields that DO carry the persona (voice_summary, signature_phrases, dialect_cadence).

**DEC:** **DEC-C3-PHILOSOPHY-005.** `context_hooks=()` for all three C-3 profiles. C-4 may revisit (and most likely will, for `columbo`).

### 3.5 `tool_preferences` voice-affinity discipline (DEC-C3-PHILOSOPHY-004)

Each C-3 profile carries 2 `tool_preferences` entries (one strategic-recon surface, one verdict-corroboration surface). All entries MUST satisfy the C-1 invariant: voice-affinity language ONLY, never selection instruction. Reference matrix:

| Persona | Recon-surface affinity | Verdict-surface affinity |
|---|---|---|
| `sun_tzu` | "crt.sh: reconnaissance of the enemy's terrain before engagement" | "VirusTotal: the verdict of many spies, weighed with discernment" |
| `bruce_lee` | "crt.sh: the river of certificate history flowing past" | "DNS resolution: each query a ripple in the network's surface" |
| `bureaucrat` | "crt.sh: Form CT-3 (Certificate Transparency Submission, public registry)" | "WHOIS: Form WH-1 (Domain Registration Disclosure, see Appendix A)" |

**Voice-affinity discipline (verbatim from DEC-30-CHARACTER-V2-005 / DEC-C1-FULLTROLL-001 / DEC-C2-NINJA-001):**

- ✅ Affinity language: "feels like", "the river of", "the verdict of", "Form X", "the enemy's terrain"
- ❌ Selection instruction: "prefer X", "use X", "must use X", "always X first", "X is required"

The existing `test_*_profile_tool_preferences_content` test in `test_character_v2.py` (lines 260-281 for full_troll, 842-862 for ninja) is the pattern C-3 mirrors. The pattern blocks `prefer ` prefix and `must use` substring matches case-insensitively. C-3's three new mirrored tests apply the same checks to each persona's `tool_preferences`.

**The hard gate — extended over three modes.** `TestSunTzuPersonaSwapHardGates::test_sun_tzu_swap_preserves_tool_call_identity`, `TestBruceLeePersonaSwapHardGates::test_bruce_lee_swap_preserves_tool_call_identity`, and `TestBureaucratPersonaSwapHardGates::test_bureaucrat_swap_preserves_tool_call_identity` each drive a `chat()` turn under their own persona vs `default` with a deterministic mock LLM and assert tool-call sequence equality. This is the mechanical proof that `tool_preferences` is voice-affinity, not selection bias. Same mock LLM + execute_tool harness as the ninja test (`tests/test_character_v2.py:917-1020`).

**DEC:** **DEC-C3-PHILOSOPHY-004.**

### 3.6 `fourth_wall_stance="opaque"` for all three C-3 personas (DEC-C3-PHILOSOPHY-006)

All three C-3 personas use `fourth_wall_stance="opaque"`. The schema-as-data is intentionally permissive here (the C-1 dataclass uses `str` not `Literal`; the `# Literal["in_character", "winking", "meta_aware"]` doctring comment is advisory). The C-2 ninja profile already established `"opaque"` as a valid stance value the test suite accepts (`test_ninja_profile_fourth_wall_stance` asserts `== "opaque"`).

**Rationale:** Each C-3 persona is *the role*, not a meta-aware narrator of being an LLM playing the role. Sun Tzu is the strategist; Bruce Lee is the philosopher; the bureaucrat is the compliance officer. None of them benefits from breaking character to acknowledge the persona is an LLM artifact — doing so would cheapen the voice register. `meta_aware` is the right stance for `full_troll` (which is built on irony) and is the wrong stance for any of these three.

**DEC:** **DEC-C3-PHILOSOPHY-006.**

---

## 4. Scope Manifest (binding — implementer enforces)

Persisted at `tmp/c3-scope.json` and registered into runtime via `cc-policy workflow scope-sync w-30-c3-philosophy-bureaucrat --work-item-id wi-30-c3-impl-01 --scope-file tmp/c3-scope.json` before the implementer dispatch. The planner stage in this work item (wi-30-c3-planning) ALSO writes the scope JSON file at the planner site; the guardian-provision stage runs `scope-sync` to bind it to the implementer work item (wi-30-c3-impl-01) at provision time.

### 4.1 Allowed paths (implementer may edit)

```
src/adversary_pursuit/gamification/modes.py   # 3 dict entries gain llm_profile + DEC-C3-PHILOSOPHY-001..006 @decision blocks
tests/test_character_v2.py                    # 9 new test classes + 1 fixture update
MASTER_PLAN.md                                # Phase 17L append + Phase 17K closeout + Plan Status row + active phase pointer
.claude/plans/character-c3-philosophy-bureaucrat.md  # this document (may be edited if implementer surfaces narrative gaps)
tmp/c3-scope.json                             # scope manifest (already authored by planner; implementer touch only if scope-sync recovery needed)
tmp/evidence-c3-philosophy-bureaucrat/        # implementer demo evidence dir (Stage E)
```

### 4.2 Required paths (implementer MUST edit at least once)

```
src/adversary_pursuit/gamification/modes.py
tests/test_character_v2.py
MASTER_PLAN.md
```

The implementer slice is incomplete if any required path is unmodified at landing.

### 4.3 Forbidden paths (implementer MUST NOT touch — denial routes back to planner)

```
src/adversary_pursuit/agent/runner.py             # C-1 already built; BYTEWISE UNCHANGED through C-3 (DEC-C2-NINJA-002 inheritance)
src/adversary_pursuit/agent/tools.py              # F62 run_fail single-authority site
src/adversary_pursuit/agent/chat.py               # cmd2/agent chat surface; persona is system-prompt data not chat-surface code
src/adversary_pursuit/core/console.py             # cmd2 path stays on F62 Rich-panel-voice surface
src/adversary_pursuit/core/streak.py              # F62 streak single-authority
src/adversary_pursuit/core/workspace.py           # workspace authority — no persona persistence in C-3
src/adversary_pursuit/core/event_bus.py           # M-3..M-6 dossier event surface — orthogonal
src/adversary_pursuit/core/pivot_policy.py        # F60 auto-pivot policy — architecturally disconnected
src/adversary_pursuit/core/dossier_pivot.py       # M-6 surface — orthogonal
src/adversary_pursuit/core/dossier_report.py      # M-7 surface — orthogonal
src/adversary_pursuit/core/config.py              # no persona config in C-3
src/adversary_pursuit/models/database.py          # DEC-DB-002 no schema migration
src/adversary_pursuit/dossier/                    # all dossier modules — orthogonal to C-3
src/adversary_pursuit/gamification/scoring.py     # ScoringEngine — orthogonal
src/adversary_pursuit/gamification/celebrations.py        # F64 panel-narration surface
src/adversary_pursuit/gamification/dossier_celebrations.py # M-7 narration surface
src/adversary_pursuit/gamification/hints.py       # F62 doc-lie cleanup site
src/adversary_pursuit/gamification/challenges.py  # gamification surface
src/adversary_pursuit/gamification/badges.py      # M-7/M-8 badge surface
src/adversary_pursuit/gamification/dossier_badges.py # M-7/M-8 dossier-badge surface
src/adversary_pursuit/modules/                    # CTI/OSINT modules — orthogonal
pyproject.toml                                    # no dep changes; tool count stays at 28
CLAUDE.md                                         # constitution
AGENTS.md                                         # constitution
settings.json                                     # constitution
hooks/HOOKS.md                                    # constitution
runtime/                                          # constitution
agents/                                           # constitution
```

### 4.4 State authorities touched

```json
["persona_profile_catalog", "persona_voice_affinity_text"]
```

These are descriptive labels for the two new authority surfaces the C-3 data extension owns: the per-mode persona catalog (`DEFAULT_MODES["sun_tzu"|"bruce_lee"|"bureaucrat"].llm_profile`) and the tool-preference voice-affinity text body. No existing authority is mutated or replaced; both are extensions of the C-1-defined `LLMPersonaProfile` authority surface. The labels are emitted into runtime via the `authority_domains` field of the scope JSON (see `tmp/c3-scope.json`).

---

## 5. Evaluation Contract (binding — reviewer + guardian-land enforce)

Persisted into runtime via `cc-policy workflow work-item-set ... --evaluation-json ...` at scope-sync time. The implementer is dispatched with this contract verbatim in the dispatch payload (AGENTS.md / Sacred Practice 6). Reviewer asserts every required_* element; Guardian-land treats reviewer's `REVIEW_VERDICT=ready_for_guardian` as the readiness gate.

The 9 legal keys are bound below. Each key is a JSON array of strings except `rollback_boundary`, `acceptance_notes`, and `ready_for_guardian_definition` which are JSON strings (DEC-CLAUDEX-EVAL-CONTRACT-SCHEMA-PARITY-001).

### 5.1 `required_tests`

```
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_has_llm_profile
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_voice_summary_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_tone_registers_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_signature_phrases_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_fourth_wall_stance
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_dialect_cadence_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_context_hooks_empty
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_tool_preferences_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_forbidden_voice_content
tests/test_character_v2.py::TestSunTzuProfileContent::test_sun_tzu_profile_token_budget
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_has_llm_profile
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_voice_summary_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_tone_registers_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_signature_phrases_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_fourth_wall_stance
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_dialect_cadence_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_context_hooks_empty
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_tool_preferences_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_forbidden_voice_content
tests/test_character_v2.py::TestBruceLeeProfileContent::test_bruce_lee_profile_token_budget
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_has_llm_profile
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_voice_summary_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_tone_registers_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_signature_phrases_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_fourth_wall_stance
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_dialect_cadence_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_context_hooks_empty
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_tool_preferences_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_forbidden_voice_content
tests/test_character_v2.py::TestBureaucratProfileContent::test_bureaucrat_profile_token_budget
tests/test_character_v2.py::TestSunTzuPersonaSwapHardGates::test_sun_tzu_swap_preserves_tool_call_identity
tests/test_character_v2.py::TestBruceLeePersonaSwapHardGates::test_bruce_lee_swap_preserves_tool_call_identity
tests/test_character_v2.py::TestBureaucratPersonaSwapHardGates::test_bureaucrat_swap_preserves_tool_call_identity
tests/test_character_v2.py::TestSunTzuF64PanelSeparation::test_sun_tzu_persona_text_not_present_in_tool_result_summary
tests/test_character_v2.py::TestSunTzuF64PanelSeparation::test_sun_tzu_does_not_smuggle_point_totals
tests/test_character_v2.py::TestBruceLeeF64PanelSeparation::test_bruce_lee_persona_text_not_present_in_tool_result_summary
tests/test_character_v2.py::TestBruceLeeF64PanelSeparation::test_bruce_lee_does_not_smuggle_point_totals
tests/test_character_v2.py::TestBureaucratF64PanelSeparation::test_bureaucrat_persona_text_not_present_in_tool_result_summary
tests/test_character_v2.py::TestBureaucratF64PanelSeparation::test_bureaucrat_does_not_smuggle_point_totals
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_llm_profile_default_is_none_for_all_modes
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_mastery_level_not_present
tests/test_character_v2.py::TestCharacterModeLlmProfileField::test_default_mode_keeps_static
tests/test_character_v2.py::TestSetCharacterIntegration::test_set_character_drunken_master_uses_v1_composition_verbatim
tests/test_character_v2.py::TestF62AuthorityInvariants::test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline
tests/test_character_v2.py::TestF62AuthorityInvariants::test_streak_manager_module_not_imported_by_modes_module
tests/test_character_v2.py
tests/
```

The trailing `tests/test_character_v2.py` entry asserts the full file is green (counts all suites). The trailing `tests/` entry asserts the full project suite is green at the M-8 baseline plus the new C-3 tests (≥ 2027 / 2027 pass post-C-3, based on M-8 baseline 1984 + ≈ 40 new C-3 tests; reviewer pastes exact count).

### 5.2 `required_evidence`

```
git diff main -- src/adversary_pursuit/gamification/modes.py | head -300
git diff main -- tests/test_character_v2.py | head -600
git diff main -- src/adversary_pursuit/agent/runner.py
# (must be empty — runner.py BYTEWISE UNCHANGED inherits DEC-C2-NINJA-002)
git diff main -- src/adversary_pursuit/agent/tools.py
# (must be empty)
git diff main -- src/adversary_pursuit/core/streak.py
# (must be empty)
git diff main -- src/adversary_pursuit/gamification/celebrations.py src/adversary_pursuit/gamification/dossier_celebrations.py src/adversary_pursuit/gamification/scoring.py src/adversary_pursuit/gamification/hints.py src/adversary_pursuit/gamification/challenges.py src/adversary_pursuit/gamification/badges.py src/adversary_pursuit/gamification/dossier_badges.py
# (must all be empty)
git diff main -- src/adversary_pursuit/dossier/
# (must be empty)
pytest tests/test_character_v2.py -v
pytest tests/ -q
tmp/evidence-c3-philosophy-bureaucrat/sun_tzu_token_budget.txt
tmp/evidence-c3-philosophy-bureaucrat/bruce_lee_token_budget.txt
tmp/evidence-c3-philosophy-bureaucrat/bureaucrat_token_budget.txt
tmp/evidence-c3-philosophy-bureaucrat/tool_count_audit.txt
```

The four token-budget evidence files contain the `_rough_token_count` output for each of the three new profiles plus an audit confirming `len(create_tools(...))` still returns 28 (unchanged from M-8). Reviewer pastes the actual content as part of the readiness verdict.

### 5.3 `required_real_path_checks`

```
ap chat then mode sun_tzu then ask "what should I do about evil.example.com?" — response opens with a Sun Tzu quote
ap chat then mode bruce_lee then ask "I'm not getting anywhere on this hunt" — response uses water/flow metaphor
ap chat then mode bureaucrat then ask "what are next steps?" — response cites a Form number and a Policy section
grep -n 'LLMPersonaProfile' src/adversary_pursuit/agent/runner.py — must show only existing imports / set_character branch (no new ref)
grep -rn 'llm_profile' src/adversary_pursuit/agent/ src/adversary_pursuit/core/ — must show only existing references (runner.py:set_character)
grep -c 'LLMPersonaProfile(' src/adversary_pursuit/gamification/modes.py — must equal 5 (full_troll + ninja + sun_tzu + bruce_lee + bureaucrat)
python -c "from adversary_pursuit.gamification.modes import DEFAULT_MODES; print([n for n,m in DEFAULT_MODES.items() if m.llm_profile is not None])" — must print exactly ['ninja','full_troll','sun_tzu','bruce_lee','bureaucrat'] (dict order; exact set membership matters, order is informational)
python -c "from adversary_pursuit.agent.tools import create_tools; from adversary_pursuit.agent.tools import ToolContext; import pathlib; ctx=ToolContext(config_dir=pathlib.Path('/tmp/c3-test-config'),workspace_dir=pathlib.Path('/tmp/c3-test-workspace')); print(len(create_tools(ctx)))" — must print 28
```

The three `ap chat` checks are demonstration evidence the implementer captures into `tmp/evidence-c3-philosophy-bureaucrat/chat-<persona>.txt`. The reviewer's job is to verify the persona register is recognizable in the response, not to grade the exact prose — the LLM is non-deterministic and the test gate is the deterministic mock-LLM persona-swap-tool-call-identity invariant.

### 5.4 `required_authority_invariants`

```
F62 — mode.run_fail remains the sole authority for failure voice; persona profile MUST NOT touch run_fail wiring (test_run_fail_wiring_in_tools_remains_byte_identical_to_baseline)
F62 — StreakManager in core/streak.py remains sole streak authority; persona profile MUST NOT acquire streak fields (test_streak_manager_module_not_imported_by_modes_module)
F62 — hint_style MUST NOT be re-introduced (test_hint_style_not_reintroduced)
F64 — gamification panels remain the sole narration surface for point totals; persona LLM text MUST NOT smuggle point/pts/score strings (test_*_does_not_smuggle_point_totals for sun_tzu / bruce_lee / bureaucrat)
F64 — persona signature phrases MUST NOT leak into LLM-facing tool result summary (test_*_persona_text_not_present_in_tool_result_summary for each new persona)
DEC-30-CHARACTER-V2-003 — schema frozen at 8 fields, ≤ 165 tokens/mode; mastery_level field MUST NOT appear (test_mastery_level_not_present)
DEC-30-CHARACTER-V2-005 — tool_preferences voice-affinity ONLY; persona-swap-tool-call-identity must hold for sun_tzu/bruce_lee/bureaucrat each vs default (test_*_swap_preserves_tool_call_identity)
DEC-C2-NINJA-002 inheritance — runner.py BYTEWISE UNCHANGED through C-3 (git diff empty)
DEC-30-CHARACTER-V2-002 supersession discipline — disposition table now reads: 5 UPGRADE post-C-3 (full_troll C-1; ninja C-2; sun_tzu/bruce_lee/bureaucrat C-3); 5 not-yet-upgraded (default KEEP_STATIC permanent; drunken_master/chuck_norris/bobby_hill/columbo deferred to C-4). The 8-UPGRADE/2-KEEP-STATIC count in DEC-30-CHARACTER-V2-002 is the v2 catalog target, not the per-slice state.
Sacred Practice 12 — persona is a single-authority extension at one integration site (set_character); no sidecar agent, no post-processor, no parallel persona surface added in C-3
```

### 5.5 `required_integration_points`

```
gamification/modes.py — DEFAULT_MODES dict; 3 entries gain llm_profile=LLMPersonaProfile(...)
agent/runner.py:set_character — inherits C-1 composer; BYTEWISE UNCHANGED post-C-3
tests/test_character_v2.py — TestCharacterModeLlmProfileField fixture grows (upgraded_modes set adds 3); new test classes mirror ninja
MASTER_PLAN.md — Phase 17K closeout with merge 16acaa3 / impl 6c87a53; Phase 17L appended for C-3; Plan Status table row added; Active Phase Pointer re-pointed
.claude/plans/character-c3-philosophy-bureaucrat.md — this document
tmp/c3-scope.json — registered via cc-policy workflow scope-sync at planner-stage close + provision-stage sync
```

### 5.6 `forbidden_shortcuts`

```
DO NOT edit agent/runner.py — inheritance from C-2 (DEC-C2-NINJA-002 byte-identity discipline)
DO NOT edit agent/tools.py — F62 run_fail wiring authority
DO NOT add a new module for persona authoring — DEC-30-CHARACTER-V2-003 mandates set_character as single integration site
DO NOT add a new LLM tool — tool count stays at 28 (post-M-8 floor); persona is system-prompt data, not a tool surface
DO NOT seed non-empty context_hooks for any C-3 persona — DEC-C3-PHILOSOPHY-005 binds context_hooks=() for sun_tzu/bruce_lee/bureaucrat; C-4 owns the dossier-aware hook surface
DO NOT introduce a mastery_level field on LLMPersonaProfile — deferred to C-4 per DEC-30-CHARACTER-V2-004
DO NOT phrase any tool_preferences entry as selection instruction ("prefer X", "use X", "must use X", "always X") — voice-affinity language only (DEC-30-CHARACTER-V2-005 / DEC-C3-PHILOSOPHY-004)
DO NOT remove or rename any field of LLMPersonaProfile — schema is frozen at C-1's 8 fields
DO NOT add a "personality v2 vs v1 compare" cmd2 path — cmd2 surface stays on F62 Rich-panel-voice (roadmap §8)
DO NOT bypass the token-budget test by changing _rough_token_count — the 4-chars-per-token heuristic is the C-1/C-2 baseline; trim profile content instead
DO NOT pre-stage MASTER_PLAN.md amendment in a separate commit — Phase 17L + Phase 17K closeout amendment commits in the SAME implementer commit as source per AP #74 orphan-prevention
DO NOT modify any dossier package file — C-3 has zero dossier coupling
DO NOT modify gamification/scoring.py, celebrations.py, dossier_celebrations.py, hints.py, challenges.py, badges.py, dossier_badges.py — persona is not a gameable surface (DEC-30-CHARACTER-V2-005)
DO NOT add a streak-corruption persona-swap test that drives StreakManager from test_character_v2.py — that would force the test module to import streak machinery; the F62 invariant is preserved by architectural disconnection (test_streak_manager_module_not_imported_by_modes_module)
DO NOT amend ninja's or full_troll's profile content — C-3 only ADDS three new profiles; pre-existing profiles BYTEWISE UNCHANGED
```

### 5.7 `rollback_boundary`

```
Single-commit revert via `git revert <impl-sha>` restores M-8 (16acaa3) state byte-for-byte:
  - DEFAULT_MODES[sun_tzu|bruce_lee|bureaucrat].llm_profile reverts to None (= F62 behavior verbatim)
  - tests/test_character_v2.py reverts to the C-2-shipped 45-test suite
  - MASTER_PLAN.md Phase 17K row reverts to in-progress, Phase 17L row removed, Active Phase Pointer reverts to W-68-M8 line
  - All other production code paths unchanged (runner.py + tools.py + dossier/* + gamification/* were never edited)
  - tool count unchanged (28; the revert doesn't touch tools.py at all)
  - No DB migration to roll back (no models/database.py edit)
  - No new global file to clean up (no novelty-cache analog; persona is process-local state in memory)
  - No deps to remove (no pyproject.toml edit)
Rollback safety is identical to C-2's: a pure-data revert with no side effects in workspace files, no migration state, and no external system to reconcile.
```

### 5.8 `acceptance_notes`

```
- The dispatch context's "persona swap mid-session does not corrupt streak counters" requirement is satisfied by architectural disconnection. modes.py never imports streak; F62 invariant test test_streak_manager_module_not_imported_by_modes_module enforces this mechanically. We do NOT add a runtime test that drives StreakManager from test_character_v2.py because that would force the test module to import the streak surface, violating the architectural disconnection principle the F62 invariant exists to protect.
- The dispatch context's "extend persona swap test from C-2" requirement is satisfied by adding three new TestPersonaSwapHardGates classes (one per new persona), each driving the same deterministic mock-LLM harness ninja uses. The C-1 invariant (tool_preferences voice-affinity only) is mechanically enforced by these tests for sun_tzu/bruce_lee/bureaucrat each vs default.
- The dispatch context's "tokenizer choice" requirement is decided: we use the existing C-1/C-2 _rough_token_count helper (4-chars-per-token conservative BPE proxy). Rationale: consistency with the two prior C-slices' budget test. Introducing tiktoken in C-3 would add a runtime dep, force C-1 + C-2 to adopt it for parity, and create a transitive dep on OpenAI's tokenizer model. The 4-chars heuristic is conservative (overestimates token count) so passing the budget means real tokens are also under budget.
- The dispatch context's "decide context_hooks for sun_tzu/bruce_lee/bureaucrat" requirement is decided: empty per DEC-C3-PHILOSOPHY-005. See §3.4 for the 4-point rationale.
- The dispatch context's "decide tool_preferences exact entries" requirement is decided: see §3.5 voice-affinity reference matrix.
- The dispatch context's "refinement window" requirement is acknowledged: closed at C-1 + C-2 ship. C-3 does NOT refine the schema.
- Phase 17K closeout (M-8 merge 16acaa3 / impl 6c87a53) is C-3's responsibility per dispatch context. The MASTER_PLAN.md row for Phase 17K and the Plan Status table row for W-68-M8-CLEANUP-NOVELTY both flip to "completed" with SHAs in the SAME implementer commit as source. AP #74 orphan-prevention pattern.
- Active Phase Pointer line is re-pointed from W-68-M8-CLEANUP-NOVELTY to W-30-C3-PHILOSOPHY-BUREAUCRAT in the same commit.
- The implementer commit message follows the C-2 pattern: `feat(character-v2): C-3 sun_tzu + bruce_lee + bureaucrat LLMPersonaProfile` with body referencing #30, DEC-C3-PHILOSOPHY-001..006, and the M-8 inheritance note (worktree based on 16acaa3 with zero dossier coupling).
```

### 5.9 `ready_for_guardian_definition`

```
All required_tests are green (every line in §5.1 passes when executed verbatim against the implementer's head SHA).

The full test suite is green (`pytest tests/ -q` reports zero failures; total test count ≥ M-8 baseline + new C-3 tests; reviewer pastes exact pass/total count).

Every git-diff entry in §5.2 is captured verbatim in the reviewer verdict: the four "must be empty" diffs are confirmed empty (paste each); the modes.py + test_character_v2.py diffs are bounded by §4.1 allowed paths.

Every real-path check in §5.3 is captured: the three `ap chat` demonstration captures are in `tmp/evidence-c3-philosophy-bureaucrat/`; the grep + python audits paste their actual output.

Every authority invariant in §5.4 is verified by its named test (or by architectural disconnection where named).

Every integration point in §5.5 has the named-edit verified by diff.

No forbidden shortcut in §5.6 is taken.

MASTER_PLAN.md has been edited in the SAME commit as source: Phase 17L appended with binding decisions; Phase 17K row flipped to completed with M-8 SHAs; Plan Status table row for W-30-C3-PHILOSOPHY-BUREAUCRAT added; Plan Status table row for W-68-M8-CLEANUP-NOVELTY flipped to completed with SHAs; Active Phase Pointer re-pointed to W-30-C3-PHILOSOPHY-BUREAUCRAT.

Implementer commit message follows `feat(character-v2):` prefix and references #30 + the DEC range `DEC-C3-PHILOSOPHY-001..006`.

tmp/c3-scope.json registered into runtime via `cc-policy workflow scope-sync w-30-c3-philosophy-bureaucrat --work-item-id wi-30-c3-impl-01 --scope-file tmp/c3-scope.json` before implementer dispatch and unchanged at landing time.

The reviewer verdict is `REVIEW_VERDICT=ready_for_guardian` at the current HEAD SHA after all above are confirmed.
```

---

## 6. Execution plan (implementer choreography)

Five stages mirroring the C-2 ninja plan structure. Each stage is a unit of progress the implementer can demonstrate to the reviewer if questioned mid-slice.

### Stage A — modes.py profile entries (~45 min)

1. Open `src/adversary_pursuit/gamification/modes.py`.
2. Locate the three target dict entries: `"sun_tzu"` (line 292), `"bruce_lee"` (line 328), `"bureaucrat"` (line 310).
3. Add the three `llm_profile=LLMPersonaProfile(...)` blocks per §3.1, §3.2, §3.3 — copy the verbatim Python literal from each section; if a profile estimates over 165 tokens at first draft, apply the named trim path until `_rough_token_count` ≤ 165.
4. Add module-level `@decision DEC-C3-PHILOSOPHY-001..006` annotations to the module docstring (extend the existing block — do NOT replace prior C-1/C-2 annotations).
5. Run `pytest tests/test_character_v2.py -q` — most existing tests should pass; the new ones don't exist yet, expect ~3 new failures (token budget, content, no-llm-profile-set fixtures).

### Stage B — tests/test_character_v2.py extensions (~75 min)

1. Update the `upgraded_modes` set in `test_llm_profile_default_is_none_for_all_modes` (line 144) from `{"full_troll", "ninja"}` to `{"full_troll", "ninja", "sun_tzu", "bruce_lee", "bureaucrat"}`.
2. Add three new test classes — copy `TestNinjaProfileContent` (lines 757-909) verbatim three times, then substitute persona name in fixture, fixture lookup key, voice-anchor word list in `test_*_profile_voice_summary_content`, and signature_phrases canonical list in `test_*_profile_signature_phrases_content` per the §3.1/§3.2/§3.3 content:
   - `TestSunTzuProfileContent` — voice_summary anchors `("strategist", "gnomic", "oblique", "patient", "art of war", "sun tzu")`; signature canonical includes `"know thy"` or `"opportunities multiply"`; fourth_wall_stance asserts `== "opaque"`; tool_preferences asserts ≥ 1 entry and forbids `prefer ` prefix + `must use` substring; forbidden_voice asserts F64 point-narration guard AND modern-slang guard.
   - `TestBruceLeeProfileContent` — voice_summary anchors `("flow", "water", "philosophical", "zen", "movement", "bruce lee")`; signature canonical includes `"water"` or `"don't fear"` or `"10,000 kicks"`; fourth_wall_stance asserts `== "opaque"`; F64 + voice-register guards (no sarcasm/snark).
   - `TestBureaucratProfileContent` — voice_summary anchors `("compliance", "form", "policy", "officer", "procedural", "corporate", "bureaucrat")`; signature canonical includes `"per policy"` or `"form"` or `"triplicate"`; fourth_wall_stance asserts `== "opaque"`; F64 + no-slang/no-contractions/no-exclamation guards.
3. Add three new persona-swap classes — copy `TestNinjaPersonaSwapHardGates` (lines 917-1020) verbatim three times, substituting persona name in `_run_chat_with_mode` and in the test method name (`test_sun_tzu_swap_preserves_tool_call_identity` etc.). The mock-LLM and execute_tool patterns are byte-identical to the ninja test.
4. Add three new F64 panel-separation classes — copy `TestNinjaF64PanelSeparation` (lines 1028+) verbatim three times, substituting persona name.
5. Run `pytest tests/test_character_v2.py -v` — expect zero failures.
6. Run `pytest tests/ -q` — expect zero failures (full suite green).

### Stage C — MASTER_PLAN.md amendments (~30 min)

1. Open `MASTER_PLAN.md`.
2. **Phase 17K closeout** — flip line 117's Status from `in-progress (planner-staged 2026-06-09...)` to `completed (2026-06-09, merge \`16acaa3\`, impl \`6c87a53\`)`; do not modify the rationale body of that row.
3. **Phase 17L append** — under `## Plan Status` after the Phase 17K row, insert a new line for `Phase 17L — Character v2 — C-3 — Philosophy + Bureaucratese (W-30-C3-PHILOSOPHY-BUREAUCRAT)` with `**Status:** completed (2026-06-09, merge \`<TBD-guardian-merge>\`, impl \`<TBD-impl>\`)` and a rationale body summarizing the three new profiles + DEC-C3-PHILOSOPHY-001..006 binding + the M-8 worktree base note + the test-count delta. Mirror the C-2 row's prose style (line 111).
4. **Aggregate paragraph** — update the `**Aggregate (reconciled 2026-06-09...)`** line 119 to acknowledge Phase 17K + Phase 17L landing: append after the existing M-8 sentence: `Phase 17K closed M-8 cleanup + cross-workspace novelty (2026-06-09, merge \`16acaa3\`, impl \`6c87a53\`). Phase 17L closed C-3 (Philosophy + Bureaucratese — sun_tzu/bruce_lee/bureaucrat LLMPersonaProfile) on 2026-06-09. 28 phases landed. After C-3 lands: v2 character roadmap has 5 UPGRADE personas live (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat); C-4 (columbo + optional mastery_level hook) is the remaining character-v2 slice.`
5. **Plan Status workflow row table** — at the existing workflow row table near line 2826 (where W-30-C2-NINJA-PROFILE is listed), add a new row: `| W-30-C3-PHILOSOPHY-BUREAUCRAT | Character v2 C-3: sun_tzu + bruce_lee + bureaucrat LLMPersonaProfile (philosophy-heavy + bureaucratic-heavy idiom personas). Single-file source slice (`gamification/modes.py` only — three dict entries gain `llm_profile`). `runner.py` BYTEWISE UNCHANGED inherits DEC-C2-NINJA-002. DEC-C3-PHILOSOPHY-001..006 binding. See Phase 17L + `.claude/plans/character-c3-philosophy-bureaucrat.md`. | source + tests | \`<TBD-merge>\` (merge) / \`<TBD-impl>\` (impl) | completed |`. Simultaneously flip the W-68-M8-CLEANUP-NOVELTY row's last-three columns to `\`16acaa3\` (merge) / \`6c87a53\` (impl) | completed`.
6. **Phase 17L section body** — append a new `## Phase 17L: ...` section near line 2150 (after the existing Phase 17E ninja section, before Phase 17F dossier scoring), mirroring the structure of `## Phase 17E` (lines 2101+). Include: workflow header, source paragraph, binding decision summary, DEC-C3-PHILOSOPHY-001..006 table, work-item summary table, evaluation contract summary, scope manifest summary, file list summary, ready-for-guardian definition. Body content harvested from this per-slice plan §3, §4, §5 — but condensed (MASTER_PLAN wins for binding decisions; this plan wins for narrative).
7. **Active Phase Pointer** — replace lines 2926-2928 with: `**Phase Active (2026-06-09 — C-3 landed; v2 character roadmap one slice from closeout):** \`W-30-C3-PHILOSOPHY-BUREAUCRAT\` (Phase 17L — Character v2 C-3, sun_tzu + bruce_lee + bureaucrat LLMPersonaProfile). Landed 2026-06-09 (merge \`<TBD>\`, impl \`<TBD>\`). After C-3 lands: 5 of 10 modes have LLMPersonaProfile (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat); remaining 5 are default (KEEP_STATIC permanent) + drunken_master, chuck_norris, bobby_hill (deferred to C-4 tier 1 if scheduled) + columbo (C-4 priority, dossier-aware context_hooks experiment). C-4 is the next scheduled v2 character slice; M-9 (Crowdsourced Dossier Comparison) is the next scheduled dossier slice. Canonical chain \`planner → guardian (provision) → implementer → reviewer → guardian (land)\`. This pointer line is positioned as the last \`**Phase ...\` boldline in the document so \`~/.claude/hooks/context-lib.sh:88\` \`get_plan_status()\` tail-grep on \`^#.*phase|^**Phase\` resolves to current work.`
8. Run `pytest tests/ -q` — expect zero failures (MASTER_PLAN edits are docs-only and do not affect test count).

### Stage D — scope/evaluation registration (planner-stage close — already done by planner)

The planner stage authors `tmp/c3-scope.json` and `cc-policy workflow work-item-set wi-30-c3-impl-01 ... --evaluation-json $(cat tmp/c3-evaluation.json)` and `cc-policy workflow scope-sync ...`. The implementer inherits these without re-running unless drift is detected. If drift is detected (e.g., `cc-policy workflow scope-get w-30-c3-philosophy-bureaucrat` returns different content than `tmp/c3-scope.json`), re-run scope-sync.

### Stage E — demo evidence capture (~15 min)

1. `ap chat` then `mode sun_tzu` then prompt "what should I do about evil.example.com?" — capture full stdout to `tmp/evidence-c3-philosophy-bureaucrat/chat-sun_tzu.txt`.
2. `ap chat` then `mode bruce_lee` then prompt "I'm not getting anywhere on this hunt" — capture to `chat-bruce_lee.txt`.
3. `ap chat` then `mode bureaucrat` then prompt "what are next steps?" — capture to `chat-bureaucrat.txt`.
4. Run the three token-budget probes per §5.2:
   ```bash
   python -c "
   from adversary_pursuit.gamification.modes import DEFAULT_MODES
   from tests.test_character_v2 import _rough_token_count
   for n in ('sun_tzu','bruce_lee','bureaucrat'):
       p = DEFAULT_MODES[n].llm_profile
       text = ' '.join([p.voice_summary,' '.join(p.tone_registers),
                        ' '.join(p.signature_phrases),p.fourth_wall_stance,
                        p.dialect_cadence,' '.join(p.context_hooks),
                        ' '.join(p.tool_preferences),' '.join(p.forbidden_voice)])
       print(f'{n}: ~{_rough_token_count(text)} tokens (budget 165)')
   " | tee tmp/evidence-c3-philosophy-bureaucrat/token_budgets.txt
   ```
5. Run tool-count audit:
   ```bash
   python -c "
   from pathlib import Path
   from adversary_pursuit.agent.tools import ToolContext, create_tools
   ctx = ToolContext(config_dir=Path('/tmp/c3-test-config'),
                     workspace_dir=Path('/tmp/c3-test-workspace'))
   print(f'tool count: {len(create_tools(ctx))} (expected: 28)')
   " | tee tmp/evidence-c3-philosophy-bureaucrat/tool_count_audit.txt
   ```

### Stage F — implementer commit (single commit per AP #74 pattern)

```bash
git -C /Users/jarocki/src/ap/.worktrees/feature-30-c3-philosophy-bureaucrat add \
    src/adversary_pursuit/gamification/modes.py \
    tests/test_character_v2.py \
    MASTER_PLAN.md \
    .claude/plans/character-c3-philosophy-bureaucrat.md \
    tmp/c3-scope.json \
    tmp/evidence-c3-philosophy-bureaucrat/

git -C /Users/jarocki/src/ap/.worktrees/feature-30-c3-philosophy-bureaucrat commit -m "$(cat <<'EOF'
feat(character-v2): C-3 sun_tzu + bruce_lee + bureaucrat LLMPersonaProfile

C-3 completes the philosophy + bureaucratese tier of the v2 character
roadmap. Three new LLMPersonaProfile entries in gamification/modes.py;
no other source code touched. runner.py BYTEWISE UNCHANGED inherits
DEC-C2-NINJA-002 (set_character composer is field-driven, fires for
any non-None profile).

Decisions: DEC-C3-PHILOSOPHY-001..006

- DEC-C3-PHILOSOPHY-001: sun_tzu profile (gnomic-strategist register)
- DEC-C3-PHILOSOPHY-002: bruce_lee profile (zen-flow register)
- DEC-C3-PHILOSOPHY-003: bureaucrat profile (dry-corporate register)
- DEC-C3-PHILOSOPHY-004: tool_preferences voice-affinity matrix
- DEC-C3-PHILOSOPHY-005: context_hooks=() for all three (C-4 owns dossier-aware hooks)
- DEC-C3-PHILOSOPHY-006: fourth_wall_stance="opaque" for all three

Tests: 3 x (10 content + 1 swap + 2 panel-sep) = 39 new tests in
tests/test_character_v2.py mirroring TestNinja* suites verbatim with
persona-specific content assertions. Full suite green. Tool count 28
unchanged.

Phase 17K closeout: M-8 cleanup + cross-workspace novelty landed at
merge 16acaa3 / impl 6c87a53 (2026-06-09). Phase 17K Status flipped
to completed in this commit. Phase 17L appended for C-3. Plan Status
table row for W-30-C3-PHILOSOPHY-BUREAUCRAT added. Active Phase
Pointer re-pointed.

Inherits: Phase 17 / DEC-30-CHARACTER-V2-001..007 (scoping);
          Phase 17C / DEC-C1-FULLTROLL-001..005 (C-1 dataclass + composer);
          Phase 17E / DEC-C2-NINJA-001..003 (C-2 ninja + test mirror pattern).

Refs: #30
EOF
)"
```

The planner cannot commit (capability: can_emit_dispatch_transition + can_set_control_config + can_write_governance — no can_commit_feature_branch). Planner stages via `git add` only; implementer is dispatched separately to commit.

---

## 7. Verification (reviewer choreography)

The reviewer's job is to execute every line of §5 against the implementer's head SHA and paste the actual output into the readiness verdict.

1. `git diff main --name-only` — confirm the only changed files match §4.1 (allowed_paths).
2. For each "must be empty" diff in §5.2, run the command and paste the (empty) output.
3. For each non-empty diff in §5.2, run the command and confirm content is bounded by §4.1.
4. Run `pytest tests/test_character_v2.py -v` and `pytest tests/ -q` — paste exact pass/fail counts.
5. For each `required_real_path_check` in §5.3, run the command and paste the output.
6. For each `required_authority_invariant` in §5.4, confirm the named test passed in step 4.
7. For each `forbidden_shortcut` in §5.6, confirm by inspection that the shortcut was not taken.
8. Confirm Stage E demo evidence files exist at `tmp/evidence-c3-philosophy-bureaucrat/` and contain non-trivial content.
9. Confirm implementer commit message follows §6 Stage F template (substring match on `feat(character-v2): C-3 sun_tzu` and `Refs: #30` and `DEC-C3-PHILOSOPHY-001..006`).
10. If all 9 above are green, emit `REVIEW_VERDICT=ready_for_guardian` at the implementer head SHA.

---

## 8. Decision Log (binding for w-30-c3-philosophy-bureaucrat)

These DEC-IDs bind the W-30-C3-PHILOSOPHY-BUREAUCRAT workflow. They are written into MASTER_PLAN.md §Phase 17L Decision Log in the implementer commit. They do NOT supersede any prior DEC-ID; they extend the C-1/C-2 schema and authoring patterns to three new personas.

| DEC ID | Decision | Rationale |
|---|---|---|
| **DEC-C3-PHILOSOPHY-001** | `sun_tzu`'s `LLMPersonaProfile` content is authored per §3.1 (strategist register, Art of War signature phrases, opaque fourth-wall stance, voice-affinity tool_preferences referencing reconnaissance/verdict-from-many-spies framing). Implementer follows the named token-trim path in §3.1 until `_rough_token_count` ≤ 165. | The static-template Sun Tzu register (`personality="Strategic Sun Tzu quotes for every action"` + four Art of War quote templates) sets the voice: oblique, patient, second-person. The LLM profile extends that without inventing a new persona — strategist framing of recon surfaces; quote-then-application cadence; no modern slang. Per-mode token budget ≤ 165 is the C-1/C-2 baseline. **Refines disposition table DEC-30-CHARACTER-V2-002:** sun_tzu was always UPGRADE — this DEC binds the concrete profile content. |
| **DEC-C3-PHILOSOPHY-002** | `bruce_lee`'s `LLMPersonaProfile` content is authored per §3.2 (zen-philosophical register, water/flow signature phrases, opaque fourth-wall stance, voice-affinity tool_preferences referencing river/ripple flow framing). Implementer follows the named token-trim path in §3.2. | The static-template Bruce Lee register (`personality="Bruce Lee philosophy, flow-state zen commentary"` + flowing-water + 10,000-kicks-once templates) sets the voice: water metaphors, adaptive iteration, movement before form. The LLM profile extends without inventing — nature-metaphor framing of recon surfaces; second-person guidance; pauses where Western prose would rush. |
| **DEC-C3-PHILOSOPHY-003** | `bureaucrat`'s `LLMPersonaProfile` content is authored per §3.3 (dry-corporate register, Form/Policy signature phrases, opaque fourth-wall stance, voice-affinity tool_preferences referencing Form CT-3/WH-1 framing). Implementer follows the named token-trim path in §3.3 — bureaucrat is the most over-budget of the three at first draft and requires the longest trim list. | The static-template bureaucrat register (`personality="Office Space vibes, everything is a TPS report"` + TPS-001 + IR-7734 + Policy §4.2.1 + ERR-404 templates) sets the voice: corporate-form citations, passive voice, procedural deadpan. The LLM profile extends without enumerating every Form number — LLM generates contextually-appropriate Form-N entries. The "no contractions / no exclamation" forbidden_voice entries preserve the dry register against drift toward modern-snark personas. |
| **DEC-C3-PHILOSOPHY-004** | All three C-3 `tool_preferences` entries are voice-affinity language ONLY (per §3.5 reference matrix): never selection instruction. The existing C-1 test pattern (forbid `prefer ` prefix + `must use` substring; require ≥ 1 known CTI tool reference) is mirrored in C-3's three new content tests. The HARD GATE is `TestSunTzuPersonaSwapHardGates::test_sun_tzu_swap_preserves_tool_call_identity` + the two parallel tests for bruce_lee/bureaucrat — each asserts tool-call sequence equality under the persona vs default using the same deterministic mock-LLM harness as the ninja test. | The most important forbidden shortcut in the v2 design (DEC-30-CHARACTER-V2-005). Without three new persona-swap hard gates, `tool_preferences` could drift into selection-biasing prose mode-by-mode without being caught. With them, every C-3 persona is mechanically gated against tool-call divergence at test time. The bureaucrat case is particularly important — Form CT-3 framing could plausibly bias the LLM toward crt.sh; the test proves it does not (or surfaces a fix). |
| **DEC-C3-PHILOSOPHY-005** | All three C-3 profiles ship with `context_hooks=()`. Generic non-dossier hooks are explicitly rejected (see §3.4 rationale). C-4 owns the `context_hooks` design surface for dossier-aware hooks (the `columbo` voice motivates them; the slot state to bind to is now landed M-1..M-8). | Empty tuple preserves a clean C-4 surface, avoids two-stage authoring drift, and prevents prose-in-data hook entries that have no mechanical gate to validate. The pattern matches DEC-C1-FULLTROLL-005 + DEC-C2-NINJA-001 (both empty for the same reason). Token-budget pressure also pushes against non-empty hooks for all three C-3 personas. |
| **DEC-C3-PHILOSOPHY-006** | All three C-3 profiles use `fourth_wall_stance="opaque"`. The schema as authored in C-1 uses `str` not `Literal`, with the doctring comment listing `in_character` / `winking` / `meta_aware` as the C-1-era recommendations; C-2 established `opaque` as a valid additional value for in-character-only personas. C-3 inherits `opaque` for all three. | sun_tzu is the strategist, not a meta-aware narrator of being an LLM playing the strategist. Same for bruce_lee (the philosopher) and bureaucrat (the compliance officer). Each persona's voice register depends on staying in-character — `meta_aware` would cheapen sun_tzu's gnomic register, break bruce_lee's flow-state register, and contradict bureaucrat's procedural deadpan. The `opaque` stance is the right semantic match for all three. C-1's `meta_aware` for `full_troll` remains the right choice for that persona because the full_troll voice IS irony — it benefits from breaking the fourth wall. |

---

## 9. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Token budget overrun at first draft | High (all three estimate over 165) | Low (named trim paths in §3.1–§3.3) | Implementer iterates trim → test until green; the named trim path is exhaustive enough to bring each profile under budget |
| Persona-swap test reveals tool-selection bias | Low (C-1 pattern proven; C-2 mirror passed) | Medium (would force tool_preferences re-author) | If a C-3 persona-swap test fails, the implementer iterates tool_preferences wording (replace "Form CT-3" with "Form CT-Z — public registry filing" etc.) until the LLM does not bias selection. The mock-LLM is deterministic so failure is reproducible; the fix is content-only |
| Bureaucrat voice drifts toward exclamation/snark during authoring | Medium (the static-template bureaucrat already uses `"In triplicate."` which is droll; LLM may amplify) | Low | forbidden_voice entry "never use slang, contractions, or exclamation marks" is the prompt-level guard; the F64 test (`test_bureaucrat_does_not_smuggle_point_totals`) is the data-level guard. Reviewer flags any exclamation in the Stage E `chat-bureaucrat.txt` capture |
| MASTER_PLAN.md edit collides with parallel main update | Low (no parallel C/M slice in flight) | Medium (rebase + reauthor Phase 17L numbering) | Worktree is current at M-8 merge; no parallel implementer workflow is active per `cc-policy lease summary`. If a parallel slice lands while C-3 is in-flight, the implementer rebases and re-numbers Phase 17L → Phase 17M etc. |
| Active Phase Pointer regex breaks if Phase 17L heading style deviates | Low | Low (status board reads stale until line is re-pointed) | Implementer mirrors the exact `**Phase Active (YYYY-MM-DD — ...):**` boldline format used by Phase 17K's pointer; the `get_plan_status()` regex `^#.*phase|^**Phase` matches the leading `**Phase` literal |
| Implementer skips Phase 17K closeout (M-8 merge SHA flip) and lands C-3 with Phase 17K still showing in-progress | Medium (the dispatch context explicitly assigns Phase 17K closeout to C-3, which is unusual) | Medium (status board stays inconsistent for v0.3.x roadmap) | §5.9 ready_for_guardian definition + §6 Stage C step 2 + §6 Stage F commit message all explicitly name Phase 17K closeout as part of C-3's scope. Reviewer pastes `grep "Phase 17K" MASTER_PLAN.md` output in the verdict to prove the SHA flip happened |
| Test count regression mistaken for failure | Low | Low (false alarm) | The C-3 slice ADDS tests; total count should grow by ~39. Reviewer notes the M-8 baseline (1984) and the C-3 expected total (≥ 2023). Exact count varies depending on what M-8's actual landed test count was — reviewer pastes both numbers |

---

## 10. Open follow-ups (recorded; not part of C-3 scope)

These are explicitly OUT of C-3 scope and recorded here so a future planner can pick them up cleanly without duplicating analysis.

1. **C-4 — `columbo` LLMPersonaProfile + dossier-aware `context_hooks` experiment.** The last remaining v2 character slice. C-4 owns:
   - `columbo` profile content (investigative-detective register; "just one more thing…" signature anchor; voice-affinity tool_preferences referencing investigation framing; `context_hooks` that reference real dossier slot state).
   - Optional `mastery_level` field on `LLMPersonaProfile` (DEC-30-CHARACTER-V2-004 deferral; C-4 planner decides whether to implement or retire).
   - Optional `drunken_master` + `chuck_norris` + `bobby_hill` profiles if user product judgment elects to close the catalog in one slice instead of spreading across C-4 + future slices. Per the roadmap §6, these three are the "voice-driven modes" tier that C-2 originally targeted as a 3-profile batch; C-2 narrowed to ninja only. C-4 may bundle them or defer.
   - After C-4: 6 to 10 personas have LLMPersonaProfile; v2 character roadmap closes; `default` is the only permanent KEEP_STATIC.

2. **Optional persona-mode-compare meta-command.** Out-of-scope per DEC-30-CHARACTER-V2-006 (roadmap §6 C-1 "optional polish"). If C-4 declines to add it, a future polish slice can surface a `mode compare <a> <b>` meta-command that runs the same prompt under two personas back-to-back and displays the voice diff. Low priority; not on any roadmap critical path.

3. **Persona persistence across sessions.** Out-of-scope per Phase 17 §8. Today the active mode at session start is always `default` (cmd2 ModeManager initializer line 369). A future slice could add a workspace-or-config-level last-active-mode preference. Touches `core/config.py` + `core/console.py` + chat startup; not in C-3 scope.

4. **Persona-aware celebration narration.** M-7's narration policy (DEC-M7-CELEB-001..007) already uses `AgentRunner.narrate()` which reuses the active persona system prompt. So persona voice already flows through dossier-celebration narration without further wiring. No follow-up needed. (Recorded here to prevent a future planner from re-discovering this surface as "missing".)

5. **Persona-aware report rendering.** M-7's `core/dossier_report.py` is a Rich-only renderer (no LLM); it does not consume persona voice. M-9 (Crowdsourced Dossier Comparison) may add LLM-narrated report sections; if so, the persona profile naturally flows through via `runner.narrate()`. Not C-3's responsibility.

---

## 11. Cross-references

- **Issue #30** — source product directive (https://github.com/jarocki/ap/issues/30).
- **`.claude/plans/character-v2-roadmap.md`** — strategic scoping (Phase 17). §3 personality schema, §4 per-mode disposition, §6 C-1..C-4 decomposition, §6.5 sequencing relative to #68.
- **MASTER_PLAN.md §Phase 17** — strategic scoping landing; DEC-30-CHARACTER-V2-001..007.
- **MASTER_PLAN.md §Phase 17C** — C-1 MVP landing; DEC-C1-FULLTROLL-001..005; `LLMPersonaProfile` dataclass + `set_character` composer + `full_troll` profile.
- **MASTER_PLAN.md §Phase 17E** — C-2 landing; DEC-C2-NINJA-001..003; `ninja` profile + KEEP_STATIC → UPGRADE supersession + test mirror pattern.
- **MASTER_PLAN.md §Phase 17K** — M-8 cleanup + cross-workspace novelty (CLOSES v0.3.x dossier roadmap). C-3 closes the Status flip in the same commit as source.
- **src/adversary_pursuit/gamification/modes.py** — sole edit site. `LLMPersonaProfile` frozen dataclass + `CharacterMode.llm_profile` field + `DEFAULT_MODES` dict + `ModeManager`.
- **src/adversary_pursuit/agent/runner.py:342-401** — sole consumption site (`set_character`). BYTEWISE UNCHANGED through C-3 (inherits DEC-C2-NINJA-002).
- **tests/test_character_v2.py** — mirror test pattern. C-3 adds 9 new test classes mirroring C-2's `TestNinja*` triple (content + persona-swap + F64-panel-separation).
- **DEC-30-CHARACTER-V2-001..007** — Phase 17 binding decisions; C-3 extends without superseding.
- **DEC-C1-FULLTROLL-001..005** — Phase 17C binding decisions; C-3 inherits schema + composer + test patterns.
- **DEC-C2-NINJA-001..003** — Phase 17E binding decisions; C-3 inherits the multi-mode-extension authoring pattern + the `opaque` fourth-wall stance value + the mirror-test-class pattern.
- **DEC-CLAUDEX-EVAL-CONTRACT-SCHEMA-PARITY-001** — runtime constraint: Evaluation Contract has exactly 9 legal keys; unknown keys raise at decode time. §5 honors this exactly.
- **AP #74** — orphan-prevention pattern: implementer commits MASTER_PLAN.md amendment in the SAME commit as source. §6 Stage C + Stage F honor this.
- **Sacred Practice 6** — Evaluation Contract is mandatory for any source task that may reach Guardian. §5 is the binding contract for C-3.
- **Sacred Practice 12** — single authority per operational fact. §2.4 state-authority map honors this. §3.4 `context_hooks=()` decision rationale honors this (no parallel hook surface in C-3 vs C-4).

---

**End of per-slice plan.**
