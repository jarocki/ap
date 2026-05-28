# Character System v2 — LLM Personas Roadmap

**Status:** strategic scoping (plan-only; no implementation in this slice).
**Workflow:** `w-30-character-v2-scoping`
**Authored:** 2026-05-27 by planner stage of W-30-CHARACTER-V2-SCOPING.
**Source issue:** [#30 Upgrade character modes from static templates to LLM personality profiles](https://github.com/jarocki/ap/issues/30).
**Drives:** the Phase 17 section of `MASTER_PLAN.md`. MASTER_PLAN carries the binding decisions and the slice index; this document carries the full rationale, per-mode disposition tables, profile-schema detail, and decomposition. When the two diverge, MASTER_PLAN wins for binding decisions; this document wins for narrative and schema detail. Any binding update must edit both atomically.

**Companion document:** `.claude/plans/dossier-reframe-v2-roadmap.md` (W-68-DOSSIER-REFRAME-SCOPING). DEC-68-DOSSIER-REFRAME-004 ratified #30 as orthogonal to #68 — player persona (#30) and target persona (the dossier itself) answer different questions. This roadmap honors that orthogonality.

---

## 0. Document Scope and Non-Scope

**This document IS:**
- The strategic scoping artifact for issue #30.
- The per-mode disposition record for the 10 F62-cleaned character modes.
- The personality-profile schema v1.0 (data shape + LLM injection mechanism).
- The decomposition of character v2 into an MVP slice + follow-on slices.
- The decision log for `DEC-30-CHARACTER-V2-001..007`.
- The sequencing record relative to AP #68's M-1..M-9 dossier roadmap.

**This document is NOT:**
- An implementation plan. No source files are touched by this slice.
- A schema commitment beyond v1.0 — the MVP slice (C-1 below) may refine the profile fields before the first implementer touches code.
- A schedule. Slice ordering is recommended; user product judgment may re-order within the constraints called out in §6.
- A supersession of any prior Decision Log row. Prior DEC-IDs continue to bind unless this document explicitly retires them with a successor DEC-ID. In particular, **DEC-MODE-001/002**, **DEC-62-KILL-DOC-LIES-001**, **DEC-AGENT-CHAT-002**, **DEC-AGENT-MODES-001**, **DEC-64-LLM-PANEL-SEPARATION-001**, and **DEC-68-DOSSIER-REFRAME-004** all continue to bind.

---

## 1. The Brief (verbatim)

From issue #30 body, kept verbatim because it is the canonical statement of v2 character intent:

> "Upgrade character modes from static templates to LLM personality profiles.
>
> ### What to build
> - Phase 3 system prompts per character (Borderlands/Fallout RPG style)
> - Dynamic personality responding to investigation context
> - Character-specific tool preferences
> - RPG-style level progression
>
> See `.claude/plans/shimmying-yawning-shamir.md` Phase C"

**Referenced-but-not-found:** `.claude/plans/shimmying-yawning-shamir.md` is cited in the issue body but does not exist in the repository (confirmed 2026-05-27 by W-68-DOSSIER-REFRAME-SCOPING planner search and re-confirmed by this planner). This scoping derives entirely from the issue body + the existing AP character code (`gamification/modes.py`, `agent/chat.py`, `agent/runner.py`, `agent/tools.py`) + the landed AP #68 dossier reframe roadmap.

---

## 2. Strategic Ratification — The "Borderlands/Fallout RPG-style" Framing

The brief asks for "Borderlands/Fallout RPG style" personas. Three framings were considered. Option (c) is selected.

### Option (a) — Adopt Borderlands/Fallout aesthetic literally

Replace the 10 F62 modes with net-new Borderlands/Fallout-archetypal personas (e.g., a snarky Claptrap-style assistant, a stoic Vault Hunter, a Pip-Boy professional).

**Pros:** Cohesive aesthetic; strong genre identity; the brief gets exactly what its words say.
**Cons:**
- F62 (W-62-STREAK-AND-HONEST-MODES, landed 2026-05-26 — *yesterday* at the moment of this planning) just spent a workflow cleanup pass on these 10 modes — deleted `hint_style`, rewrote dishonest personality strings, wired `run_fail` as the single authority for failure voice, deleted the parallel `_MODE_TITLE_FLAVORS` dict. Throwing the modes out one day later wastes that cleanup and creates the exact "addition without subtraction" drift Sacred Practice 12 warns against.
- The existing 10 modes are intentionally varied across registers (Sun Tzu strategic, Bobby Hill chaotic, bureaucrat dry, Columbo investigative). Collapsing to a single genre (Borderlands/Fallout) narrows the comedic surface and erases personas that fit specific user-mood states the current catalog serves.
- The brief's mention of "Borderlands/Fallout" reads as **a style reference for the *voice quality*** ("snarky, irreverent, RPG-archetypal, willing to break the fourth wall") rather than as a literal IP-aesthetic mandate. Treating it as the former preserves the catalog; treating it as the latter wastes recent work.

**Verdict:** rejected.

### Option (b) — Replace with professional analyst archetypes (Mossad / GCHQ / Mandiant)

Replace the irreverent personas with professional-grade analyst voices.

**Pros:** Better alignment with the dossier-reframe (#68) product center; matches the "Threat Hunter expert pressure" that drove F59/F60 hardening.
**Cons:**
- Eliminates the "fun is a first-class design constraint" axis that v1 explicitly preserved (see MASTER_PLAN MLP definition: "at least one visible gamification signal in the chat path"). The character-mode catalog *is* one of those signals.
- A professional-only catalog removes the user's choice to dial register up or down by mood — which is the original *value* the 10-mode catalog provides.
- Confuses orthogonal axes: persona (player UX flavor) vs analytic rigor (which is owned by the dossier slot system + STIX provenance + auto-pivot policy). Replacing personas with "more professional" voices implies persona drives rigor; per DEC-68-DOSSIER-REFRAME-004 (player persona orthogonal to target persona), it does not.

**Verdict:** rejected.

### Option (c) — Upgrade existing modes with LLM personality profiles; preserve F62 catalog

Keep all 10 F62-cleaned modes. Layer a v2 *personality profile* per mode that:

1. Captures the persona's **voice quality** in a structured way the LLM can lean on (tone, register, idioms, fourth-wall stance, signature phrases, dialect cadence).
2. Injects via the **existing** `AgentRunner.set_character` site (`runner.py:278-295`) — extend the prompt-construction logic, don't add a parallel persona surface.
3. Preserves the F62 static surfaces (`greeting`, `run_success`, `run_fail`, `score_celebration`, `personality`) as the **Rich-panel voice** of the persona. The LLM profile is the *conversational* voice of the same persona; both fields point at the same character.
4. Honors the brief's "Borderlands/Fallout style" as a *voice-quality recommendation* — applied non-uniformly across the catalog (full_troll naturally inherits the snarky-irreverent registry; bureaucrat does not; sun_tzu stays gnomic; ninja stays minimal).

**Pros:**
- Reuses the cleanup F62 just landed instead of throwing it out.
- Single integration site (`runner._default_system_prompt` + `set_character`). No parallel persona authority.
- Preserves all v1 invariants: ModeManager state machine, F62 single-authority for `run_fail`, F64 panel/LLM separation.
- Brief's intent (LLM personalities that respond dynamically to context) is satisfied without IP cosplay or persona deletion.
- Each mode upgrades independently — incremental landing, reversible at the per-mode boundary.

**Cons:**
- The persona prompt grows; chat-path token usage increases per turn. Mitigated by §3's bounded profile budget.
- Tone-consistency between the static Rich strings (`run_success`, `score_celebration`) and the LLM persona must be authored carefully per mode. Mitigated by per-mode acceptance tests in C-1's Evaluation Contract.

**Verdict:** selected. **DEC-30-CHARACTER-V2-001.**

---

## 3. Personality Profile Schema v1.0

### 3.1 Mechanical question — how does the personality reach the LLM?

Three injection mechanisms were considered. Option (a) is selected.

**Option (a) — System-prompt fragment injected by `set_character`.** Extend `AgentRunner.set_character` to compose the persona's profile into the system prompt that fronts every LLM turn.
- *Pros:* Single integration site (the v1 site at `runner.py:278-295`). Single LLM call per turn. No new latency. Token budget bounded by §3.4. Profile is data — easy to test, mutate, and refine.
- *Cons:* Persona consumes top-of-context tokens — paid once per turn, mitigated by §3.4 budget.

**Option (b) — Sidecar agent that re-narrates tool outputs in persona voice.** A second LLM call after the main turn rewrites tool result narration through the persona.
- *Pros:* Strong persona consistency; main agent stays "professional" for tool calls.
- *Cons:* Doubles LLM round-trips; violates F60 token-budget discipline; creates a parallel agent path that competes with the sidecar pattern already used for celebrations/badges/challenges; F64's panel-authority would have to extend to a third surface. **Rejected for parallel-authority risk.**

**Option (c) — Response post-processor that rewrites the final response in persona voice.** Single extra LLM call after the main turn.
- *Pros:* Persona consistency without changing main-agent behavior.
- *Cons:* Same parallel-call cost as (b); same F60 budget violation; risk of post-processor altering tool-correctness narration. **Rejected.**

**Verdict (a):** system-prompt fragment via the existing `set_character` site. **DEC-30-CHARACTER-V2-003.**

### 3.2 Profile data shape (v1.0)

Add a single new field to `CharacterMode` (frozen dataclass per DEC-MODE-001 — adding a field is a compatible extension): `llm_profile: LLMPersonaProfile | None`.

When `llm_profile is None`, the mode falls back to the F62 behavior (just the one-line `personality` field is prepended). When present, `set_character` composes the structured profile into the system prompt.

`LLMPersonaProfile` is a new frozen dataclass with these fields (v1.0 — refinable in C-1 within ±2 fields):

| Field | Type | Definition | Token budget (recommended) |
|-------|------|------------|----------------------------|
| `voice_summary` | `str` | One-sentence summary of the persona's voice. Mirrors `personality` (and may match it verbatim for modes where the F62 personality string is already a strong voice summary). | ≤ 20 tokens |
| `tone_registers` | `tuple[str, ...]` | 2–4 register words describing the tonal palette ("snarky", "gnomic", "deadpan", "stoic", "wry", "chaotic", "minimal", "philosophical"). | ≤ 10 tokens |
| `signature_phrases` | `tuple[str, ...]` | 2–5 catch-phrases the persona uses at high frequency ("Just one more thing…", "That's my purse!", "Be water, my friend"). Quoted as data; persona may improvise around them. | ≤ 30 tokens |
| `fourth_wall_stance` | `Literal["in_character", "winking", "meta_aware"]` | Whether the persona acknowledges being an LLM/tool ("meta_aware"), occasionally winks ("winking"), or stays strictly in-character ("in_character"). Default per-mode authored, not user-selectable. | ≤ 5 tokens (enum) |
| `dialect_cadence` | `str` | Sentence-rhythm note ("clipped one-liners", "rambling drunk diction", "1970s detective trailing-off"). | ≤ 20 tokens |
| `context_hooks` | `tuple[str, ...]` | 1–3 *guidance pointers* on how the persona should respond to investigation context ("escalating victim count → escalating wry concern"; "rare finding → understated awe with a Sun Tzu quote"). Empty tuple is allowed. | ≤ 40 tokens |
| `tool_preferences` | `tuple[str, ...]` or `()` | 0–3 tool-name hints the persona has voice-affinity for ("crt.sh feels like Bruce Lee's flowing rivers"). Pure flavor; **does not bias tool selection** (see §5.3 non-shortcut). | ≤ 20 tokens |
| `forbidden_voice` | `tuple[str, ...]` | 0–3 voice patterns the persona MUST NOT do (e.g., bureaucrat MUST NOT use slang; ninja MUST NOT chatter). Negative guidance. | ≤ 20 tokens |

**Total per-mode token budget:** ≤ 165 tokens of prompt overhead. At full chat-context size this is < 0.2% of a typical context window, well within F60's budget discipline.

### 3.3 LLM injection — the `set_character` composer

The v1 composition at `runner.py:292-294`:

```python
self.system_prompt = (
    f"Character mode: {mode.name}\n{mode.personality}\n\n" + self._default_system_prompt()
)
```

The v2 composition (target shape — implementer authors exact text in C-1):

```
Character mode: {mode.name}
Voice: {profile.voice_summary}
Tone: {", ".join(profile.tone_registers)}
Cadence: {profile.dialect_cadence}
Stance: {profile.fourth_wall_stance}
Signature phrases (use sparingly, not every turn): {profile.signature_phrases}
Investigation-context hooks: {profile.context_hooks}
Forbidden voice patterns: {profile.forbidden_voice}

{default_system_prompt}
```

**When `llm_profile is None`:** unchanged from v1 (back-compat for KEEP_STATIC modes — see §4).

**Token budget enforcement:** C-1 implementer slice authors a unit test that asserts `set_character` output size in tokens for each upgraded mode is within budget. No runtime cap; the cap is a planning constraint enforced at test time, similar to F60's policy-budget tests.

### 3.4 Confidence and refinement window

The schema above is binding for C-1 with the explicit exception that C-1 may refine `LLMPersonaProfile` by ±2 fields before the first implementer touches code. Further refinement (post-C-1) requires a planner re-stage and a successor DEC-ID. Mirrors DEC-68-DOSSIER-REFRAME-010's discipline.

---

## 4. Per-Mode Disposition (the 10 F62-Cleaned Modes)

For each of the 10 modes in `gamification/modes.py`, a disposition: **UPGRADE** (gets `llm_profile`), **KEEP_STATIC** (the persona is intentionally minimal/professional — no LLM profile), or **RETIRE** (the mode no longer earns its slot in v2).

| Mode | F62 personality (verbatim) | v2 disposition | Reason |
|------|----------------------------|----------------|--------|
| `default` | Standard analyst mode — neutral tone | **KEEP_STATIC** | The neutral baseline. Purpose of this mode is to be *the persona-free option*. Adding an LLM profile would defeat its purpose. Stays as the "no flavor" choice. |
| `ninja` | Minimal output, silent and concise messaging | **KEEP_STATIC** | The mode's purpose is *less output, not more characterful output*. An LLM profile would push it toward verbose persona-as-content; the user picks ninja exactly to escape that. Stays minimal. |
| `full_troll` | Maximum memes, loud taunt messages | **UPGRADE (high priority)** | This is the strongest fit for the brief's "Borderlands/Fallout snarky-irreverent" framing. C-1's MVP slice upgrades this mode. |
| `drunken_master` | Rambling tipsy energy, unpredictable commentary | **UPGRADE** | Strong voice-quality persona that benefits dramatically from LLM dynamism. *Rambling tipsy* is hard to do well with static templates; LLM does it natively. |
| `sun_tzu` | Strategic Sun Tzu quotes for every action | **UPGRADE** | LLM can pull contextually-appropriate quotes from a wider pool than static templates allow. High signal-to-effort upgrade. |
| `chuck_norris` | Unstoppable confidence, Chuck Norris facts as flavor | **UPGRADE** | Chuck Norris facts are a well-defined corpus; LLM is well-suited to generate context-appropriate facts dynamically. |
| `bureaucrat` | Office Space vibes, everything is a TPS report | **UPGRADE** | Heavy idiom load (form numbers, policy sections, in-triplicate phrasing) — LLM extends that naturally without explicit form-name authoring. |
| `bobby_hill` | "That's my purse!" energy, King of the Hill flavor | **UPGRADE** | Strong signature phrase + chaotic energy — LLM extends well past the 4-line static catalog. |
| `bruce_lee` | Bruce Lee philosophy, flow-state zen commentary | **UPGRADE** | Parallel to sun_tzu — LLM enriches the philosophy quoting beyond a static template's limits. |
| `columbo` | "Just one more thing..." investigative prompts | **UPGRADE (priority for v2)** | The investigative-detective persona is the most *AP-thematically-aligned* of the catalog. A dossier-reframe-aware Columbo who actually asks "just one more thing…" when an Identity slot is partial would be a perfect connector between the persona system and the dossier system (see §6 sequencing note re: M-7). |

**RETIRE count:** zero. Every mode earns its v2 slot, either as static (default, ninja — the "minimum-flavor" anchors of the catalog) or as upgradeable.

**Upgrade count:** 8 of 10. C-1 MVP upgrades **one** mode (`full_troll` recommended — see §6); the remaining 7 are sequenced across C-2/C-3 in priority order.

**KEEP_STATIC ≠ second-class.** The two KEEP_STATIC modes are kept *because* they earn their slots — they serve user-mood states (no flavor; minimal flavor) that the LLM-upgraded modes cannot serve without contradicting themselves.

**DEC:** **DEC-30-CHARACTER-V2-002.**

---

## 5. RPG-Level-Progression and F62/F64 Invariant Preservation

### 5.1 RPG-level progression — the interpretation question

Issue #30's brief includes "RPG-style level progression" as a character-system feature. Issue #31 (RPG gamification v2 with XP, levels, skill trees, loot, quests) was **retired** by DEC-68-DOSSIER-REFRAME-005 as superseded by the dossier reframe. Two interpretations of #30's progression bullet are possible:

**Interpretation (i) — Same retirement applies.** #30's "RPG-style level progression" was meant as part of #31's broader RPG frame, and DEC-68-DOSSIER-REFRAME-005 retires it.

**Interpretation (ii) — Orthogonal.** #30's level progression is *character-mode mastery* (e.g., unlock more sophisticated persona behaviors after the user has spent N sessions in that mode), not score-based XP grind.

**Decision:** Interpretation **(ii) is partially adopted, narrowly scoped, and deferred**.

- The brief's RPG-progression bullet IS retired in its "skill trees / XP / loot / quest" form — those are exactly what DEC-68-DOSSIER-REFRAME-005 retired and they would re-import the activity-volume reward frame the dossier reframe is designed to replace.
- A *narrow* form of persona mastery is preserved as a **future hook**: a `mastery_level: int` field on `LLMPersonaProfile` (0–N) that, in C-4 or later, may unlock *deeper* persona behaviors (richer signature-phrase pool, more context_hooks, more elaborate dialect_cadence). The unlock mechanic is **session-count** or **dossier-completion-count for that mode**, not score-grinding.
- **Crucially:** persona mastery does NOT bias scoring, NOT alter `run_fail` voice (F62 single-authority), and NOT change tool selection. It only expands the LLM profile's expressive range.
- **C-1 does NOT implement mastery.** The MVP ships with all upgraded modes at mastery level 0 (their base profile). Mastery-level-1+ profiles are authored in C-4 (or omitted entirely if user product judgment retires the hook at that slice).

This narrows interpretation (ii) enough that it cannot re-introduce the XP-grind frame. The single piece of "level progression" that survives is **per-mode expressive depth**, which is unambiguously a UX-flavor axis, not a scoring axis.

**DEC:** **DEC-30-CHARACTER-V2-004.**

### 5.2 F62 mode authority preservation

F62 (W-62-STREAK-AND-HONEST-MODES, landed `28c97c5`/`92d6f64` 2026-05-26) established:

- **`StreakManager`** in `core/streak.py` is the sole streak authority. Persona has no streak fields and MUST NOT acquire any. Mastery (if implemented) keys off session count or completion count, NOT streak state.
- **`mode.run_fail`** is the single authority for mode-flavored failure voice. Wired at `agent/tools.py:1622-1628` (Rich-stripped before embedding in the LLM-facing error return) and at the cmd2 `console.py` exception path. Persona LLM profile MUST NOT leak failure-voice text into the LLM summary — failure voice is a Rich-panel concern.
- **`hint_style`** was deleted as a doc-lie (DEC-62-KILL-DOC-LIES-001). v2 personas MUST NOT re-introduce a `hint_style` field. Mode-flavored hint headers stay in the existing chat-path subtitle (`chat.py:268-275, 290-296`), which is sourced from `mode.name` — that pattern is preserved.

**The F62 contract for v2:** the `LLMPersonaProfile` is **strictly additive** to the existing CharacterMode dataclass. The five Rich-panel-voice fields (`prompt_prefix`, `greeting`, `run_success`, `run_fail`, `score_celebration`) and the `personality` summary remain authoritative for their respective surfaces. The LLM profile is consulted only by `AgentRunner.set_character` and is invisible to the Rich-panel path.

**DEC:** **DEC-30-CHARACTER-V2-005.**

### 5.3 F64 LLM/Rich-panel separation preservation

DEC-64-LLM-PANEL-SEPARATION-001 (landed 2026-05-26) established that gamification panels (celebrations, badges, challenges) are rendered to the user via Rich panels (`chat.py:638-680`), **not** parsed from the LLM summary. The LLM is told what fired; the Rich panel renders the surface.

**For v2 personas:**

- The persona text generated by the LLM (inside `chat()`-loop responses) is the **chat content** — naturally Rich-rendered as Markdown.
- The persona text MUST NOT smuggle gamification events into its prose ("congrats, you just earned 25 points!" — the panel owns that surface). C-1's Evaluation Contract includes a unit test that asserts the LLM does not narrate point totals when an `llm_profile` is active.
- The `run_fail`-flavored exception path (`tools.py:1622-1628`) continues to be Rich-stripped before embedding in the LLM-facing error string. Persona voice in the *response* is preserved; persona voice in the *tool-failure error embedding* stays terse and plain-text-safe.
- **Tool preferences (§3.2 `tool_preferences`)** are persona flavor only. They MUST NOT bias the LLM's tool-selection behavior in a way that changes which IOCs get enriched. The system-prompt fragment for `tool_preferences` is phrased as voice affinity ("Bruce Lee feels the flow of crt.sh"), NOT as instruction ("prefer crt.sh"). C-1's Evaluation Contract includes a test that swaps the active persona over a synthetic chat sequence and asserts the tool-call sequence is unchanged. **This is the most important forbidden shortcut in the v2 design.**

**DEC:** **DEC-30-CHARACTER-V2-005 (same DEC; F62 + F64 are jointly preserved).**

### 5.4 F60 auto-pivot policy invariance

F60's auto-pivot policy engine (`core/pivot_policy.py`, DEC-60-PIVOT-POLICY-001..007) is a security gate, not a UX layer. v2 personas MUST NOT influence pivot decisions. The pivot policy reads STIX provenance + confidence + budget — it does not read the active mode and never will. No test is needed because the policy code has no input wire to the persona surface; the DEC is preserved by virtue of architectural disconnection.

---

## 6. Decomposition — MVP Slice + Follow-On Slices

Four slices. C-1 is the MVP (smallest valuable shipping unit; ≤ 2 weeks of implementer work; deliverable as v0.2.x). C-2 through C-4 are sequenced by user-visibility × blast-radius (low blast-radius first when user-visibility ties).

### C-1 — MVP: First Upgraded Mode (recommended `full_troll`)

**Scope:**
- Add `LLMPersonaProfile` frozen dataclass to `gamification/modes.py` (per §3.2 schema).
- Extend `CharacterMode` with `llm_profile: LLMPersonaProfile | None = None` (default None — back-compat).
- Author the v2 profile for **one** mode (`full_troll` recommended — strongest fit for Borderlands/Fallout snark; lowest dossier-coupling risk; highest comedic visibility for demo).
- Extend `AgentRunner.set_character` (`runner.py:278-295`) to compose the profile into the system prompt per §3.3 when `llm_profile is not None`; preserve the v1 path verbatim when None.
- All other 9 modes ship at `llm_profile=None` — they continue to behave exactly as F62.
- Add `mode <name> compare` chat meta-command (optional polish — surfaces the v1 vs v2 voice diff to the user; out-of-scope acceptable if it inflates the slice).

**Why this is the MVP:**
- Validates the schema (`LLMPersonaProfile` survives contact with a real persona + a real LLM + the real `set_character` site) before any other mode commits to it.
- Validates the F62/F64 invariant tests against a real upgrade — if the persona leaks gamification narration or biases tool selection, C-1's tests catch it before 7 more modes inherit the bug.
- Reversible: if the schema needs refinement, only one profile rewrites.
- Single user-visible demo: `mode full_troll` then `mode default`, ask the same question, see the voice change in the chat response (not just the Rich panel headers).

**Slice size:** Small-to-Medium. ≤ 2 weeks implementer effort.

**Out of scope for C-1:**
- The other 7 UPGRADE modes (drunken_master, sun_tzu, chuck_norris, bureaucrat, bobby_hill, bruce_lee, columbo).
- `mastery_level` mechanics (deferred to C-4).
- Any cmd2 console persona-prompt changes — cmd2 path remains the F62 Rich-panel-voice path. The agent path is the only v2 surface.
- Any new gamification events.
- Any change to `mode.run_fail` wiring.

**Acceptance criteria (to be hardened by the implementer slice's Evaluation Contract):**
- `mode full_troll` then a normal chat question produces a response whose voice register matches the profile (`snarky`, `irreverent`, `meta_aware` stance, signature phrases used sparingly).
- `mode default` then the same question produces the v1 neutral-analyst voice (proving the profile is not bleeding into the global LLM behavior).
- Per-mode token-budget unit test for `set_character` asserts the system prompt under `full_troll` is within budget.
- F62 invariant test: `mode.run_fail` is still the sole authority for failure voice at the tool-error embedding site; `LLMPersonaProfile` is not consulted there.
- F64 invariant test: synthetic chat sequence with persona-on vs persona-off produces identical tool-call sequences (persona does not bias selection).
- F64 invariant test: persona response does NOT contain literal point-total strings ("+25 points", "+5 pts") when celebrations fire; the Rich panel remains the sole gamification-narration surface.
- Snapshot test: `mode <name> list` table renders unchanged from F62 (no schema change visible to that surface).

### C-2 — Upgrade Tier 1: Voice-Driven Modes (3 modes)

**Scope:** Author `LLMPersonaProfile` for `drunken_master`, `bobby_hill`, `chuck_norris` — the three modes whose v2 value is mostly *voice extension* of an already-strong static voice. Each is a small additive change: one new profile per mode, no code change beyond the data.

**Why second:** All three have well-defined corpus-affinity (drunken diction; "that's my purse!" chaos; Chuck Norris fact register) that the LLM extends naturally. They share C-1's integration site exactly; no new code paths are introduced. Lowest blast radius after C-1.

**Slice size:** Medium. Three profiles + tests that mirror C-1's invariant suite for each.

**Removal targets:** none — purely additive on top of C-1.

### C-3 — Upgrade Tier 2: Philosophy-Quoting Modes (2 modes) + bureaucrat

**Scope:** Author `LLMPersonaProfile` for `sun_tzu`, `bruce_lee`, `bureaucrat`. These three modes share a property: their voice is *idiom-heavy with a coherent semantic register* (strategic quotes; flow-state zen; corporate-form bureaucratese). The LLM profile extends each beyond its static catalog.

**Why third:** Slightly higher authoring risk than C-2 (quoting Sun Tzu and Bruce Lee accurately requires a curated phrase pool; bureaucrat requires consistent form-numbering across an LLM turn). C-2 lands first to lock the schema before idiom-heavy authoring stretches it.

**Slice size:** Medium. Three profiles + tests.

**Removal targets:** none.

### C-4 — Upgrade Tier 3: `columbo` + Optional Persona-Mastery Hook

**Scope:**
- Author `LLMPersonaProfile` for `columbo` — sequenced last because Columbo's "just one more thing…" investigative voice is the **strongest candidate for dossier-coupling experiments** (see §6.5). Landing it last lets it be authored *after* AP #68 M-1 (dossier visualization panel) lands if user product judgment elects to thread persona-aware "just one more thing…" suggestions when an Identity slot is partial.
- Optionally implement the `mastery_level` hook from §5.1 — a single new field on `LLMPersonaProfile` (`mastery_level: int = 0`) and a per-mode "mastery-level-1" profile variant for two or three modes (recommended: `full_troll` and `columbo`). Counter source: a new lightweight `core/persona_mastery.py` that tracks per-mode session count (workspace SQLite). **C-4 is the slice where the mastery question is decided** — implementer may also recommend retiring the hook entirely if the user product judgment elects.

**Why fourth:**
- Columbo's investigative voice is the **bridge** between the persona system and the dossier system. Landing it after dossier M-1 lets the C-4 planner author profile `context_hooks` that reference real dossier slot state (e.g., "Identity slot at low confidence → 'just one more thing… have we checked the WHOIS?'"). If columbo lands before M-1, the `context_hooks` stay generic.
- Mastery is the most speculative piece of the v2 scope. Deferring it to C-4 lets the prior slices' usage patterns inform whether mastery is worth implementing at all.

**Slice size:** Medium (columbo profile alone) to Large (columbo + mastery hook + per-mode mastery-level-1 profiles).

**Removal targets:** none. If mastery is retired at C-4, the `mastery_level` field is never introduced — there's nothing to remove.

### 6.5 Sequencing Relative to AP #68 (M-1..M-9)

The brief asks for an explicit sequencing decision relative to AP #68's dossier roadmap. Options:

- **(α) C-1 lands before M-1.** Character v2 ships first; the v0.2.0 release is character-themed. Dossier MVP follows.
- **(β) C-1 lands in parallel with M-1.** Different agent (or same agent in different worktrees); v0.2.0 carries both.
- **(γ) C-1 lands after M-1 but before M-7.** Dossier MVP first; persona upgrade follows; M-7 (LLM-narrated celebrations) leans on persona profile when it lands.
- **(δ) C-1 lands after M-7.** Dossier roadmap completes the analytic-value layer; persona upgrade is the last v2 surface.

**Decision:** **(β) parallel with M-1 for C-1; sequencing-aware for C-2..C-4.**

- C-1 has **zero dependency** on dossier state. It touches `gamification/modes.py`, `agent/runner.py`, no other code paths. M-1 touches `agent/chat.py` (new `dossier` meta-command), new LLM tool `get_dossier_state`, and read-side aggregation logic. The two slices share `chat.py` only as a *file-level* coincidence; their edits do not collide (M-1 adds a new meta-command branch; C-1 does not edit `chat.py` at all under its scope manifest).
- C-1's MVP visibility is **complementary** to M-1's MVP visibility: a user demonstration of v0.2.0 that shows *both* "the dossier panel updates as you investigate" *and* "the chat voice is recognizably Borderlands-snark in full_troll mode" is a stronger product story than either alone.
- C-2 and C-3 may land before, during, or after M-1..M-3 — they are purely additive on the C-1 surface and have zero dossier dependency.
- **C-4 SHOULD land after M-4 (dossier persistent state)** if the `context_hooks` for `columbo` are to reference real dossier slot state. C-4 may land earlier with generic `context_hooks`; the planner that opens C-4 makes the call.
- **M-7 SHOULD land after C-1** if the LLM-narrated-celebrations slice wants to lean on `LLMPersonaProfile` for voice consistency. M-7 may land earlier with non-persona-aware narration; the M-7 planner makes the call.

**No critical-path conflict.** C-1 ↔ M-1 are independent; C-4 prefers post-M-4; M-7 prefers post-C-1. Both roadmaps can sequence to satisfy all preferences:

```
v0.2.0:  C-1 + M-1                  (parallel, independent)
v0.2.x:  C-2 + M-2 + M-3            (parallel, independent)
v0.3.x:  C-3 + M-4 + M-5 + M-6      (parallel, independent)
v0.3.x:  C-4 + M-7                  (C-4 prefers post-M-4; M-7 prefers post-C-1; both already true)
v0.3.x:  M-8
v0.3.0+: M-9
```

The orchestrator may schedule C-1 and M-1 to the same v0.2.0 wave or stagger them. Either is consistent with this scoping.

**DEC:** **DEC-30-CHARACTER-V2-007.**

---

## 7. Decision Log (binding)

These DEC-IDs are binding for the W-30-CHARACTER-V2-SCOPING workflow and bind subsequent implementer slices C-1 through C-4. They are also written into MASTER_PLAN.md §17 Decision Log.

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-30-CHARACTER-V2-001** | The "Borderlands/Fallout RPG style" brief is interpreted as a *voice-quality recommendation* applied non-uniformly across the existing 10 F62-cleaned modes, NOT as a literal IP-aesthetic mandate that replaces the catalog. Option (c) over (a) replace-with-genre and (b) replace-with-professional. | F62 just landed 10 honest modes one day ago; throwing them out wastes that cleanup. The 10-mode catalog serves user-mood states a single-genre catalog cannot. The brief's literal Borderlands/Fallout words fit best as the voice quality of the snarky-irreverent modes (`full_troll` especially), not as a catalog-wide aesthetic. |
| **DEC-30-CHARACTER-V2-002** | Per-mode disposition: 8 of 10 modes UPGRADE with LLM profiles; 2 (default, ninja) KEEP_STATIC; 0 RETIRE. KEEP_STATIC choices are intentional — those modes serve the "no flavor" and "minimal flavor" user-mood anchors that LLM-upgraded modes cannot serve without contradicting themselves. | Each disposition justified in §4 disposition table. KEEP_STATIC ≠ second-class — those modes earn their slots by purpose, not by adoption of the v2 mechanism. |
| **DEC-30-CHARACTER-V2-003** | Personality profile schema v1.0 (§3.2, 8 fields) injects via the existing `AgentRunner.set_character` site (`runner.py:278-295`) as a system-prompt fragment. Reject sidecar-agent (option b) and response post-processor (option c) as parallel-authority surfaces that violate F60 token-budget discipline. Per-mode token budget ≤ 165 tokens. | Single integration site honors Sacred Practice 12. No additional LLM round-trips. Token budget bounded and test-enforced. CharacterMode dataclass is extended (compatible — DEC-MODE-001 frozen-dataclass discipline preserved), not replaced. |
| **DEC-30-CHARACTER-V2-004** | "RPG-style level progression" from the issue body is partially adopted: the XP-grind / skill-tree / loot / quest forms are **retired** (already retired by DEC-68-DOSSIER-REFRAME-005 superseding #31). A narrow `mastery_level: int` hook on `LLMPersonaProfile` is **deferred to C-4** as an optional future expressive-depth axis keyed off session count or per-mode dossier-completion count, NOT off score-grinding. C-4 planner may retire the hook entirely. | Re-introducing XP grind would directly contradict the dossier reframe (#68) and Sacred Practice 12's parallel-authority warning. Per-persona expressive depth is unambiguously a UX-flavor axis, not a scoring axis, and is bounded enough that it cannot drift into score-grinding territory. Deferring to C-4 lets the prior slices' usage patterns inform the decision. |
| **DEC-30-CHARACTER-V2-005** | F62 + F64 invariants are jointly preserved: `mode.run_fail` remains the sole authority for failure voice (`tools.py:1622-1628` wiring untouched); StreakManager remains the sole streak authority (persona has no streak fields); `hint_style` is not re-introduced; gamification-event narration stays on the Rich-panel surface (LLM persona text MUST NOT smuggle "+N points" strings); `tool_preferences` profile field is voice-affinity only and MUST NOT bias tool selection (C-1 invariant test verifies). F60 auto-pivot policy is architecturally disconnected from the persona surface. | The v2 personas are **strictly additive** to the F62 CharacterMode surface — the existing Rich-panel-voice fields and the existing single-authority wirings stay exactly as they are. The most important *forbidden shortcut* is the `tool_preferences` field becoming a tool-selection bias; C-1's Evaluation Contract includes the persona-swap-tool-call-identical test as a hard gate. |
| **DEC-30-CHARACTER-V2-006** | C-1 is the MVP: one upgraded mode (`full_troll` recommended) + the `LLMPersonaProfile` dataclass + the extended `set_character` composer + the invariant test suite (F62 single-authority for run_fail; F64 panel-separation; tool-call-identity under persona swap; per-mode token budget). Target release v0.2.x. ≤ 2 weeks implementer effort. The other 9 modes ship at `llm_profile=None` and behave exactly as F62 until C-2/C-3/C-4. | MVP validates the schema and the invariant tests against one real persona before 7 others inherit any latent bug. Smallest unit of demonstrable user-visible v2 value; reversible at the per-mode boundary. |
| **DEC-30-CHARACTER-V2-007** | Sequencing relative to AP #68: C-1 lands **parallel with M-1** (zero dependency between them; complementary v0.2.0 product story). C-2/C-3 may land any time; they are additive on the C-1 surface. C-4 prefers post-M-4 (so `columbo`'s `context_hooks` can reference real dossier slot state). M-7 prefers post-C-1 (so the LLM-narrated celebrations can lean on `LLMPersonaProfile` for voice consistency). All four preferences are simultaneously satisfiable per the §6.5 schedule table. | No critical-path conflict. The two roadmaps share `agent/chat.py` only as a file-level coincidence; their edits do not collide under C-1's Scope Manifest (M-1 adds a meta-command branch; C-1 does not edit chat.py). |

---

## 8. Out-of-Scope (planner asserts; implementer slices honor)

- **No source code changes in W-30-CHARACTER-V2-SCOPING.** This workflow is plan-only. The deliverable is the plan itself plus the MASTER_PLAN.md §17 amendment.
- **No new modes.** The 10 F62-cleaned modes are the v2 catalog. New persona ideas get a fresh issue and a fresh planner pass.
- **No cmd2 console persona-prompt changes.** v2 persona profiles are an `ap chat` (agentic) surface only. The cmd2 path remains the F62 Rich-panel-voice path. Discussed but rejected: extending the cmd2 prompt to also surface profile voice — the cmd2 path doesn't have an LLM, so the analogy breaks.
- **No new gamification events.** v2 personas are pure presentation flavor. They never emit `ScoreEvent`s, never earn badges, never trigger challenges. The persona is not itself a gameable surface.
- **No persona-bound tool restrictions.** `tool_preferences` is voice flavor only. Bureaucrat mode does NOT actually require Form TPS-001 before crt.sh enrichment.
- **No persona persistence beyond v1's session-default.** The active mode at session start is the v1 default ("default"). Persona mastery (if implemented in C-4) introduces a single new column-or-table for session-count tracking; no other persistence change.
- **No federation, no real-time multi-user, no DALL-E, no web/GUI.** v1 Non-Goals continue to bind.
- **No MCP-migration (#65) dependency.** Orthogonal.
- **No dossier dependency for C-1, C-2, C-3.** C-4 *prefers* post-M-4 but is not blocked by it.

---

## 9. Cross-References

- **MASTER_PLAN.md Phase 17** — the binding planner section that cites this document.
- **Issue #30** — source product directive (https://github.com/jarocki/ap/issues/30).
- **AP #68 dossier reframe** — `.claude/plans/dossier-reframe-v2-roadmap.md` + MASTER_PLAN Phase 16. DEC-68-DOSSIER-REFRAME-004 ratifies #30 as orthogonal.
- **F62 honest modes** — MASTER_PLAN Phase 12B; `src/adversary_pursuit/gamification/modes.py`; DEC-MODE-001/002, DEC-62-KILL-DOC-LIES-001.
- **F64 LLM/Rich-panel separation** — MASTER_PLAN Phase 13; DEC-64-LLM-PANEL-SEPARATION-001.
- **F60 auto-pivot policy** — MASTER_PLAN Phase 12; DEC-60-PIVOT-POLICY-001..007 (architecturally disconnected from persona surface; preserved by disconnection).
- **AgentRunner integration site** — `src/adversary_pursuit/agent/runner.py:278-295` (`set_character`) — the sole v2 integration point.
- **F62 run_fail wiring** — `src/adversary_pursuit/agent/tools.py:1622-1628` (Rich-stripped `mode.run_fail` embedded in LLM-facing error return); cmd2 path at `src/adversary_pursuit/console.py` (preserved unchanged).
- **DEC-AGENT-CHAT-002** — chat-path `mode` meta-command routing; preserved by v2 (the meta-command surface does not change).
- **Sacred Practice 12** (single authority per operational fact) — bound throughout; v2 personas are an *additive* layer over the F62 CharacterMode surface with one integration site.

**Referenced-but-not-found:** Issue #30 cites `.claude/plans/shimmying-yawning-shamir.md` Phase C. That file does not exist in the repository (confirmed 2026-05-27 by W-68 planner; re-confirmed by this planner). This scoping document does NOT depend on `shimmying-yawning-shamir.md` content; all decisions derive from the issue body and the existing AP character code.
