# Project Reckoning: Adversary Pursuit — Tactical Hygiene Wave, Strategic Direction Frozen

**Date:** 2026-06-29
**Source:** /Users/jarocki/src/ap/MASTER_PLAN.md (3479 lines)
**Project age:** ~85 days of active development since plan filed 2026-04-05 (preceded by 3.5 years of dormant 2022 vision README)
**Maturity tier:** **Mature** (30+ phase rows, 7 new phases since prior reckoning, 305 unique DEC-IDs across 59 annotated files, 2,736 tests collected, `pyproject.toml` still `v0.1.0`)
**Initiatives:** Plan Status table lists 37 phase rows; 7 new phases (17O→17U) since 2026-06-09; current pointer = Phase 17U landed 2026-06-24
**Decisions:** 305 unique DEC-IDs in code (+18 from prior reckoning's 287), 59 annotated source files (+2 from 57); `DECISIONS.md` regenerated 2026-06-10 22:31 — now 19 days behind code again
**Predecessor:** `2026-06-09-reckoning.md` (verdict: on course; named seven What-to-Do-Next items including release-cut v0.4.x and roadmap selection)

---

## I. The Core

Adversary Pursuit's irreducible essence has not changed in 20 days — and the gravitational test of "did the project honor its 06-09 verdict?" is exactly what this reckoning needs to answer. The prior reckoning's claim was that the project had moved from "drifting constructively" to **on course** because the strategic loop closed: #68 was named, ratified, decomposed, executed in 14 days, and shipped via M-1..M-9. The verdict was earned by demonstrated velocity on a strategic axis.

Twenty days later, the picture has inverted. The project has shipped **7 new phases** (17O Error Routing, 17P Workspace Clear + db_status, 17Q Boot Banner Redesign, 17R REPL Revival, 17S Hunt Config Init Fix, 17T Module Credential Resolver, 17U Fixture Path Hardcode Fix) — every one of them tactical: hygiene, bug-fix, or UX polish. **Zero of the 4 strategic options the prior reckoning surfaced under What-to-Do-Next item #4 were acted on.** The release-discipline cut (v0.4.x) was filed as issue #82 and remains OPEN. The MCP migration epic (#65, #66, #67) sits untouched at 37 days latency. The runtime-hygiene backlog (#49–#55, #58) was overridden again. The cross-workspace authority registry (item #5) was not written.

What this means about the project's soul: the founding tension between rigor and playfulness is intact, but a second tension has surfaced and the project's recent behavior is biased to one side of it — the tension between **strategic direction-setting** and **tactical defect-clearing**. In the 14-day window before the prior reckoning, the project executed two parallel strategic roadmaps to closure. In the 20-day window since, the project has executed seven tactical patches and zero strategic moves. **The reckoning loop that the 06-09 verdict celebrated did not run again.** The strategic loop is not broken — the dossier and character roadmaps remain genuinely closed — but it is *unscheduled*.

## II. The Origin

The Original Intent is unchanged and quoted verbatim from MASTER_PLAN.md line 5-7:

> "Adversary Pursuit is a gamified framework for hunting, pivoting, and discovery of actor infrastructure, indicators, and TTPs. 'Taking maximum advantage of every mistake, and celebrating with epic memes.' Goals: Make it fun. Gamify. Different modes. Graph of pursuit progress. Teaching moments at dead ends. Meme/DALL-E generator for celebrations. Final report generation (interview-based). Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development."

Re-reading this at the 2026-06-29 vantage point produces a fresh observation: **all six load-bearing commitments are at the same fulfillment state they were at the 06-09 reckoning.** Gamification is fulfilled. Multiple modes is closed. Graph of pursuit progress is shipped. Teaching moments at dead ends is partially fulfilled. ASCII celebrations is fulfilled (DALL-E remains a deliberate Non-Goal). Crowdsourcing is architecturally tractable. Nothing on this list moved in this window. That is not failure — these are *closed* commitments, not stalled ones — but it sharpens the question: **what does the project's recent labor serve?** The answer is honest and worth naming: it serves *the path to v0.4.x ship*, which itself remains unscheduled.

The Phase 17Q boot banner redesign deserves a careful note. It consumed two deep-research artifacts (`DeepResearch_AP_AdversaryPursuit_Logo_2026-06-17/`, `DeepResearch_NewLogo_2026-06-17/`) and a real implementation slice (pyfiglet dependency added, ANSI shadow wordmark + crosshair reticle). Is a logo deep-research and banner redesign a faithful pursuit of the Original Intent? The "Make it fun" principle gives it cover. But a banner redesign at the moment the project has 6 open issues blocking a v0.4.x release-discipline cut is also exactly the shape of *avoidance work* — the work you do to feel productive without facing the actual decision the project needs you to face.

## III. The Journey

### Timeline (since the prior reckoning, 2026-06-09)

| Date | Phase | Status | Key Decisions | Outcome |
|------|-------|--------|---------------|---------|
| 2026-06-10 | DECISIONS.md hygiene | merge `3cf14a7` | (#72 fix) | NEW `scripts/regen_decisions.py`; DECISIONS.md regenerated to 1,251 lines |
| 2026-06-11 | 17O Error Routing | merge `474a8a6`, impl `30f6b00` | DEC-ERROR-ROUTING-001..007 | `httpx.HTTPStatusError` 401/403/429/5xx routed through ErrorInterpreter; `ap chat` tool dispatch is the 3rd surface |
| 2026-06-11 | 17P Workspace Clear + db_status | merge `724413a`, impl `c4795b9` | DEC-WORKSPACE-DB-001..007 | `WorkspaceManager.clear()` + chat `workspace` subcommand parity + enhanced `db_status` |
| **2026-06-12 → 2026-06-18** | **(dark 7 days)** | no commits | — | First sustained gap since v1 ship |
| 2026-06-19 | 17Q Boot Banner Redesign | merge `f366fb2`, impl `675db7a` | DEC-AGENT-BANNER-001..002 | pyfiglet ANSI shadow wordmark + crosshair reticle; 2 deep-research artifacts consumed |
| 2026-06-19 | 17R REPL Revival | merge `58557a9`, impl `c161ee6` | DEC-CONSOLE-001..004 + DEC-PLUGIN-001..002 + DEC-IOC-TYPES-001 | 8 cmd2 REPL defects fixed (Rich output silently dropped; fuzzy `use`; `hunt <ioc>`; personas off prompt); NEW `core/ioc_types.py` |
| **2026-06-20 → 2026-06-22** | **(dark 3 days)** | no commits | — | Second gap |
| 2026-06-23 | 17S Hunt Config Init Fix | merge `3608461`, impl `926339b` | DEC-HUNT-INIT-001 | AP #97 — `hunt <ioc>` was calling `module.initialize(Config)` instead of `module.initialize(ConfigManager)`; regression from 17R |
| 2026-06-23 | 17T Module Credentials Resolver | merge `f117b3d`, impl `fb5b102` | DEC-MODULE-CREDS-SHARED-001 | AP #98 — shared `core/module_credentials.py` resolver; chat-vs-REPL credential path unified |
| 2026-06-24 | 17U Fixture Path Hardcode Fix | merge `99c53f7`, impl `d17ec96` | (test fix) | AP #84 — 4 invariant tests hardcoded the M-9 worktree path that was cleaned up; replaced with repo-root-relative `_REPO_ROOT` |
| **2026-06-25 → 2026-06-29** | **(dark 5 days, current)** | no commits | — | Third gap; longest of the three; current state |

### Decision Density

In the 20-day window from the prior reckoning to today:

- **21 commits to `main`** (per `git log --since 2026-06-09 | wc -l`) — versus 47 commits in the prior 14-day window. **Velocity halved.**
- **Active days: 5** (2026-06-10, 06-11, 06-19, 06-23, 06-24). **Dark days: 15.**
- **~18 new DEC-IDs in code** (305 vs 287). **Plan-side new DEC families:** DEC-ERROR-ROUTING (7), DEC-WORKSPACE-DB (7), DEC-AGENT-BANNER (2), DEC-CONSOLE (4 updates), DEC-PLUGIN (2 extensions), DEC-IOC-TYPES (1), DEC-HUNT-INIT (1), DEC-MODULE-CREDS-SHARED (1) = ~25 new in-plan DEC entries. The rate dropped from ~7 new DEC-IDs/day in the dossier sprint to ~1.25/day in this window.
- **7 new phase sections authored** (17O, 17P, 17Q, 17R, 17S, 17T, 17U). All tactical.
- **Two roadmaps remain closed** (v0.3.x dossier; v2 character). **No new roadmap opened in this window.**

That is roughly **one phase per ~2.8 days** in this window — versus one per ~0.8 days in the prior window. The 3.5× slowdown is not the whole story. The work-shape changed: from "9 slices defending one product reframe" to "7 patches across 6 unrelated surfaces."

### Inflection Points

1. **2026-06-10 (`3cf14a7` — DECISIONS.md hygiene).** First action on the prior reckoning's What-to-Do-Next list. `scripts/regen_decisions.py` is new — issue #72 closed. **Disciplined response to a named confront item.** This sets up a pattern that the rest of the window only partially honors.

2. **2026-06-11 (Phases 17O + 17P in one day).** Two slices in one day at typical AP cadence. The Active Phase Pointer is the surprise here: the M-9 closeout pointer (Confront #1 in the prior reckoning) was *also* updated as part of this work. Two more confront-items closed.

3. **2026-06-12 → 2026-06-18 (first dark week).** First sustained 7-day pause without commits since v1 ship. No issue or commit explains it. Holidays? Burnout? Strategic pause? The plan does not say. The plan and runtime have no field for "no work this week and here's why."

4. **2026-06-19 (Phases 17Q + 17R in one day after the dark week).** The return-from-dark wave. Phase 17Q is the banner redesign that consumed two deep-research artifacts; Phase 17R is the REPL revival that surfaced 8 defects in the cmd2 surface. **Both fix defects that have existed since v1, not new product capabilities.** The character of the work has shifted from "build forward" to "fix backwards."

5. **2026-06-23 (Phase 17R produces a regression that needs Phase 17S immediately).** AP #97 was filed *the day before its fix*: 17R's `_hunt_ioc` fleet-dispatch path passed `self.config` (Pydantic dataclass) instead of `self.config_mgr` (`ConfigManager`) to `module.initialize()`. **Every `hunt <ioc>` invocation crashed.** This is the first regression-chain in the project's life that surfaced within 96 hours of its causing slice. The reviewer caught it (verdict `ready_for_guardian` at impl `926339b`), but only after the implementer had finished — meaning the tests in 17R did not cover the real-world hunt path.

6. **2026-06-24 (Phase 17T immediately produces Phase 17U).** AP #98 (`ConfigManager.get()` 2-arg incompatibility) and AP #84 (test fixtures hardcoded the M-9 worktree path that was cleaned up) landed in the same day. **The 17U finding is particularly important:** when the M-9 worktree was cleaned up between 06-09 reckoning and now, four invariant tests in `tests/test_invariants_f59.py` and `tests/test_invariants_f64.py` started failing because they hardcoded `cwd=/Users/jarocki/src/ap/.worktrees/feature-68-m9-crowdsourced-dossiers`. **Worktree-cleanup discipline interacted with test discipline in an unforeseen way.** Confront #3 from 06-09 (uncleaned worktrees) has shifted: cleanup happened — and surfaced this bug. Net good, but it shows the cost of letting worktrees survive into shipped tests.

7. **2026-06-25 → 2026-06-29 (current 5-day dark stretch).** The third gap and the longest. The plan's Active Phase Pointer still names Phase 17U (landed) as the most recent work. There is no in-flight slice. **The project is currently between roadmaps with no scheduled next move.** The 06-09 reckoning's "next direction: release-discipline cut v0.4.x" remains the consensus answer in every Active Phase Pointer line ("Phase 17O/P/Q/R/S/T/U is a tactical hygiene insert that does not displace the strategic direction") — but the strategic direction itself has not been picked up.

### Plan vs. Reality

**Tight coupling (sustained / improved):**
- DECISIONS.md regeneration was repaired (`scripts/regen_decisions.py` ships, issue #72 closed, 1,251-line registry generated 2026-06-10). The prior reckoning's 6-week stale gap became a tooling fix in 24 hours after the reckoning landed.
- The Active Phase Pointer is healthy (line 3443 names Phase 17U as latest landed, dated 2026-06-24). The pointer-drift problem from prior reckoning #1 has been mechanically domesticated: every closeout commit updates the pointer in the same merge.
- Worktree cleanup discipline has recovered (15 worktrees → 3 worktrees, 2 of which are stale and 1 is the live cwd). The collapse confronted in prior #3 was addressed.
- 305 unique DEC-IDs in code; the `@decision` annotation discipline held across all 7 phases.

**Drifts (some persistent, some new):**
- **`DECISIONS.md` is freshly stale again.** Regenerated 2026-06-10; last code commit 2026-06-24. The 19-day gap is forgivable but the script exists and was not re-run after Phase 17S/T/U. **The repair was treated as a one-shot, not a contract.** Either `regen_decisions.py` belongs in a Guardian-landing hook or in CI, or `DECISIONS.md` is going to silently slide back to 6+ weeks stale by mid-July.
- **2 stale worktrees on disk** (`feature-ap76-gitignore-audit`, `feature-error-routing-2026-06-11`). The error-routing worktree is post-merge (17O landed 06-11) — it should have been cleaned at landing time. The gitignore-audit worktree is for issue #76 which has been OPEN since 2026-05-30 (30 days). Worktree-cleanup discipline is *better* than prior reckoning but not yet *automatic*.
- **Three dark stretches totaling 15 of 20 days.** No tracked rationale; no issue named "paused"; no Active Phase Pointer notation acknowledging the gap. The plan has no schema for "deliberate pause" vs "drift" — they look identical from outside.
- **Issue #82 (Release v0.4.x cut) was filed 2026-06-11 (good — captured the prior reckoning's #6) but is OPEN-unscheduled (bad — same pattern as #72 was for 13 days before tooling fixed it).** Eighteen days open. The pattern from prior reckoning is recurring at a slower cadence: file the issue, do not schedule it, work on something else.
- **MASTER_PLAN.md has not been touched since 2026-06-24** (`stat` shows mtime Jun 24 10:15:43). Five days since planner-level updates. This is normal for a between-slices state, but combined with the 5-day dark stretch it means no planner activity for the same window.

**New runtime/orchestrator surface debt (genuinely new in this window):**
A class of issues not present in prior reckonings: hook-and-runtime bugs filed against the orchestration system itself.
- **#85** (Guardian SubagentStop "Cannot route" when lease released + branch deleted before stop fires) — filed 2026-06-19. **OPEN.**
- **#89** (Guardian SubagentStop routing loop) — filed 2026-06-19. **OPEN.**
- **#90** (Guardian terminal-cleanup completions cannot route) — filed 2026-06-19. **OPEN.**
- **#96** (Hook-vs-CLI workflow_id resolver split) — filed 2026-06-23. **OPEN.**
- **#100** (Harness eval-state race) — filed 2026-06-27. **OPEN.**
- **#101** (cc-policy lease release returns {released:false} without state change) — filed 2026-06-27. **OPEN.**
- **#83** (Hooks UX: bulk-scoped approval grants for worktree triage) — filed 2026-06-11. **OPEN.**

Seven open orchestrator/hook bugs filed by the user's own orchestrator workflows. These do not appear in MASTER_PLAN.md at all — they're tracked entirely in GitHub. **This is a new category of debt the project did not have at 06-09.** The harness is being exercised harder than the runtime can keep up with.

## IV. Evolution Assessment

### Intent Alignment: **Strong (sustained but with a tactical bias)**

The product surface continues to honor the Original Intent. The intent-alignment story is *unchanged* since 06-09. What is new in this window is a question that the 06-09 verdict did not have to answer: **is the project still moving toward the Original Intent, or is it moving sideways through hygiene work?**

| Principle | Honored? | Evidence (new in this window) |
|-----------|----------|-------------------------------|
| Fun is a first-class design constraint | **Yes (deepened)** | Phase 17Q boot banner redesign — pyfiglet ANSI shadow wordmark + crosshair reticle motif + version + IOC count. The user invested two deep-research artifacts in getting the AP "feel" right. Banner is fun. |
| Metasploit UX is the interaction model | **Yes (defended actively)** | Phase 17R restored 8 cmd2 REPL defects that had silently degraded the `ap` power-user surface since v1: Rich output dropped to StringIO, no `hunt <ioc>` fleet dispatch, persona strings in `_execute_hunt`. The cmd2 REPL was the *original* MLP and it's now better than at v1 ship. |
| STIX 2.1 is the lingua franca | **Yes (untouched)** | No STIX work in this window. M-9's export remains the production surface. |
| Modules are pure data producers | **Yes (defended)** | Phase 17T extracted shared `core/module_credentials.py` resolver. The chat-vs-REPL split where chat had a correct resolver and REPL had a broken one is now unified — Sacred Practice 12 enforced post-hoc. |
| Playfulness and rigor are not opposites | **Yes (proven again)** | Phase 17Q banner redesign (playful) shipped under typed budgets — pyfiglet pre-rendered at import (≤500ms boot budget preserved per DEC-AGENT-BANNER-001 invariant); width fallback for narrow tmux. Fun under constraint. |

### Constructive Expansions

In this window, the expansions are smaller and tactically scoped. They are all genuine product improvements, but none of them expand the project's product *boundary*:

- **`core/ioc_types.py` + `hunt <ioc>` fleet dispatch (Phase 17R).** New IoC-type detector (regex-ordered: URL → IPv4 → IPv6 → SHA256 → SHA1 → MD5 → Email → Domain) + `accepts: tuple` declared on all 15 modules + `PluginManager.modules_accepting(ioc_type)` router. **First fleet-dispatch primitive in AP.** Powerful: `hunt 1.2.3.4` now runs every module that accepts IPv4 in parallel. This is closer to "Metasploit + CTFd" than the original `use → set → run` flow because it's *one shot*. Healthy expansion.
- **`core/module_credentials.py` (Phase 17T).** Extracted-and-unified credential resolver per Sacred Practice 12. Was a hidden divergence between chat-side and REPL-side; now one authority. The chat side (`agent/tools.py::_resolve_module_credentials`) had been correct since the agent path was built; REPL was passing the wrong object. **Single-authority discipline applied at a discovered duplication site.** This is exactly what Sacred Practice 12 is for.
- **`WorkspaceManager.clear()` (Phase 17P).** Zeroes rows from 6 ORM models for the active or named workspace; preserves the SQLite file and schema; loud-fail post-clear assertion. Closes the user-visible "I want to start this investigation over" gap with no schema change (DEC-DB-002 preserved). The agent does NOT get a `clear_workspace` LLM tool (DEC-WORKSPACE-DB notes "agent must NOT have unconfirmed wipe capability") — **deliberate capability-restriction discipline at the LLM tool boundary**. Healthy.

### Scope Drift — **Mild scope drift in tactical direction, not product direction**

The candidate scope drift to name: **the project's recent slices are drifting from product expansion to platform hygiene.** None of 17O/P/Q/R/S/T/U add new product surface to AP. They:
- Phase 17O — closed a Phase 10 (W-FRIENDLY-ERRORS) coverage gap.
- Phase 17P — added UI surface parity between cmd2 and chat for an existing capability.
- Phase 17Q — replaced an ASCII banner with a different ASCII banner.
- Phase 17R — fixed 8 cmd2 REPL defects that existed since v1.
- Phase 17S — fixed a regression from Phase 17R.
- Phase 17T — extracted a duplicated function into a shared module.
- Phase 17U — fixed a test that hardcoded a cleaned-up worktree path.

**Every slice is justified.** Every slice has a reviewer-verdict and a Decision Log. But *zero* of them serve the next strategic vector. The 06-09 reckoning closed by saying "next direction: release-discipline cut v0.4.x" and **every Active Phase Pointer since has dutifully repeated** "Phase 17X is a tactical insert that does not displace the strategic direction" — a refrain that, repeated 7 times, becomes a self-deception. *Tactical inserts that do not displace strategic direction* are how strategic direction gets displaced.

### Non-Goal Violations — **None**

A scan against v1 Non-Goals confirms zero violations in this window:
- Web/GUI: not built.
- Mobile: not built.
- Jupyter: not built.
- Federation: not built.
- Cloud/VM: not built.
- AI-classification: not built.
- 3D rendering, character sheets, real-time collab: not built.
- DALL-E celebrations: still ASCII. (Phase 17Q is pyfiglet ANSI art, which is ASCII-character-class — not AI-generated.)

The v1 Non-Goals discipline has held across 30+ phases. This is now extraordinarily demonstrated discipline.

### Abandoned Threads

- **MCP migration epic (#65)** — filed 2026-05-23. **37 days latent.** Zero in-plan presence. The 06-09 reckoning named this as the next big strategic question. Not picked up.
- **#66 Honeylabs MCP integration** — 37 days latent. Not picked up.
- **#67 go-roast MCP integration** — 37 days latent. Not picked up.
- **#82 Release v0.4.x cut** — filed 2026-06-11 in response to 06-09 reckoning Confront #6. **18 days latent.** Not picked up. The release-discipline cut is the *most prescriptive* unaddressed recommendation from the prior reckoning.
- **#58 Meta: drain runtime hygiene backlog before v2 planning** — 42 days OPEN. Override-by-practice persists.
- **#33 (PyPI release for v2)**, **#28 (CTI knowledge base for RAG)**, **#27 (HEF/Analysis/Persona)**, **#26 (Conversational chat console)** — all still OPEN, all still in the same state as prior reckoning. The 06-09 Confront #7 recommended closing or commenting on these. Not done.
- **#76 (.gitignore enhancement + audit untracked carry-forward)** — 30 days OPEN. Has a worktree (`feature-ap76-gitignore-audit`) provisioned but no in-flight slice. **Provisioned but not picked up — a new failure mode.**
- **#77, #78, #79, #80, #81** — M-5/M-6/M-7 follow-ups filed 2026-06-08 during dossier-roadmap closeout. All OPEN, all still latent. The 06-09 reckoning did not name these explicitly; they're worth recognizing as evidence the dossier-roadmap left tracked follow-up debt.

**Pattern:** "File the issue, don't schedule it" continues to be the project's standard response to a deferred concern. The prior reckoning had three such issues (#65, #72, #58); this reckoning has at least eight (#65, #66, #67, #76, #77, #78, #79, #80, #81, #82, #85, #89, #90, #96, #100, #101). **The latent-promises pile is growing faster than the project can drain it.**

## V. Decision Quality

### Coherence: **Strong (sustained)**

Decision coherence is intact. The 7 new phases each ship with per-slice plan + DEC table + Scope Manifest + Evaluation Contract. The AP #74 orphan-prevention pattern continues to hold (planner amendments co-shipped in implementer commits). Sacred Practice 12 was *actively enforced post-hoc* in Phase 17T when the chat-vs-REPL credential-resolver divergence was discovered — that's discipline maturing, not weakening.

- **Cross-DEC reference density (Phase 17T example):** DEC-MODULE-CREDS-SHARED-001 cites DEC-AGENT-SERVICE-NAME-MAP-001 + DEC-AGENT-TOOLS-003 + DEC-HUNT-INIT-001 + Sacred Practice 12. Four upstream decision families at a small tactical slice.
- **Decision-by-negation discipline:** Phase 17P names DEC-WORKSPACE-DB-001 ("manager method has no `confirm_token` parameter"), DEC-WORKSPACE-DB-002 ("sentinel rows in score_events ARE cleared by side effect — intentional"), and the implicit "no `clear_workspace` LLM tool" decision. Boundaries are still being explicitly drawn at slice authoring time.
- **Removal-targets discipline:** No major removals in this window. The M-8 deprecation-runway pattern (Sacred Practice 12 at deletion time) has not been re-exercised. That's not a regression — there was nothing to delete — but the pattern's reusability is unproven for a second cycle.

### Notable Decision Chains

- **The cmd2-REPL revival chain**: DEC-CONSOLE-001 (Rich output to stdout) → DEC-CONSOLE-004 (plain `ap>` prompt) → DEC-PLUGIN-001..002 (fuzzy `use` + `modules_accepting`) → DEC-IOC-TYPES-001 (regex-ordered IoC detection) → DEC-HUNT-INIT-001 (shared `_initialize_module` helper) → DEC-MODULE-CREDS-SHARED-001 (extracted shared resolver). **Six DEC families across Phases 17R/17S/17T defending the cmd2 surface.** A coherent micro-roadmap that retroactively recognizes the REPL as a v2-grade surface deserving the same single-authority discipline as the dossier package.
- **The error-routing chain**: DEC-ERROR-INTERPRETER-008 (Phase 10 contract — "no Python traceback ever reaches the user without going through the interpreter") → DEC-ERROR-ROUTING-001..007 (Phase 17O extension to chat tool dispatch). Two-phase chain that took 4 weeks to close the universal-coverage gap. Coherent, slow, faithful.

### Decision Gaps

- **No strategic-direction decision was made in this window.** The 06-09 What-to-Do-Next item #4 ("Choose the next roadmap") had five candidate options. Twenty days later, the choice has not been made. **This is the largest unmade decision in the project's life right now.** It is not stuck — the project is still functioning — but no Decision Log entry says "we deferred the next-roadmap decision because [X]." The deferral is invisible.
- **No "no-pause-without-rationale" decision.** The 15 dark days of 20 are unrecorded. A small "pause log" in MASTER_PLAN.md ("2026-06-12..18: user holiday; 2026-06-20..22: pre-banner research; 2026-06-25..29: ???") would acknowledge what the timeline data already says.
- **No release-discipline decision.** The plan still ships as `version = "0.1.0"` while production code includes the M-1..M-9 dossier roadmap, the C-1..C-4 character roadmap, and 7 more tactical slices. **`pip install adversary-pursuit` returns v0.1.0 from 2026-05-19 — 41 days behind.** Issue #82 captures the gap; no DEC-RELEASE-* family has been authored.

### Traceability

Quantitatively healthy:
- 305 unique DEC-IDs in code (was 287). +18 in 20 days (~0.9/day; down from 8.5/day in the dossier window).
- 59 annotated source files (was 57). +2 (the `core/ioc_types.py` and `core/module_credentials.py` new files).
- `DECISIONS.md` registry has been refreshed once (2026-06-10, generated by new `scripts/regen_decisions.py`). It is currently 19 days behind code (acceptable) but the regeneration script is not in any hook or CI.

**The plan-side and code-side coupling held through the slowdown. That is the structural sign that the discipline survived velocity loss.**

## VI. Project Health

| Indicator | Rating | Evidence |
|-----------|--------|----------|
| **Vitality** | **Steady (degraded from "Thriving")** | 21 commits in 20 days (vs 47 in prior 14 days). Five active days, fifteen dark days. The pulse is real but slower; the silences are noticeable. No declared cause. |
| **Focus** | **Diffuse (degraded from "Sharp")** | Seven phases across six unrelated surfaces (error routing, workspace db, banner, REPL, hunt init, credentials, fixtures). No shared upstream goal. The dossier roadmap's focus mechanism (M-1..M-9 sized to one merge each) is not present; these are isolated patches. |
| **Momentum** | **Decelerating** | Phase cadence is now ~1 per 2.8 days (vs 0.8 days in prior window). Three dark stretches. No in-flight slice as of 2026-06-29. **Current state: between roadmaps with no scheduled next move.** |
| **Coherence** | **Strong (sustained)** | DEC chains span 6 phases coherently (cmd2 REPL revival arc). Per-slice plans + DEC tables + Scope Manifests still 100% atomic with merge. AP #74 orphan-prevention pattern held through every closeout. Sacred Practice 12 was enforced post-hoc in 17T. |
| **Sustainability** | **Watchpoint shifted — strategic direction debt** | The infrastructure-of-tooling debt named at 06-09 (cross-workspace authorities, MCP migration, runtime hygiene) was joined by a new debt: **the post-roadmap drift**. When two roadmaps close and the next one is not picked up, the project goes into hygiene mode by default. Hygiene mode is sustainable as a pause; as a steady state it is product stagnation with high coherence — the most insidious kind. |

## VII. Trajectory

### Current Vector

The project has, in this window, oscillated between two vectors and committed to neither.

**Vector A (hygiene + UX polish):** Continue what 17O→17U did — find user-reported defects in the existing surface, fix them with tight DEC discipline, ship one slice every 2–3 days. This vector is sustainable, low-risk, and produces visible quality improvement. It is what the project actually did.

**Vector B (strategic continuation):** Pick one of the 06-09 reckoning's five options (M-10+ dossier follow-ons, MCP migration, runtime hygiene drain, release-discipline cut, federation) and execute it through a multi-slice roadmap. This vector is higher-risk, higher-payoff, and is what the project explicitly *did not do*.

The plan continues to assert that Vector B is what's next ("Phase 17X is a tactical hygiene insert that does not displace the strategic direction") — but **the strategic direction has not advanced.** That assertion has appeared in 7 consecutive Active Phase Pointer lines without producing the strategic decision it points at.

### Projected Destination

If Vector A continues unchanged for 3 months:

- **The REPL and chat surface will become extremely polished.** Every defect filed will get a planner+implementer+reviewer+guardian chain and a DEC table. The 305 DEC-IDs in code could grow to 400+ without any new product surface.
- **The latent-promises pile will keep growing.** #65, #66, #67, #82, #76, #77–#81, #85, #89–#101 — at the current rate, 1–2 new issues filed per week with 0 resolved per week.
- **`pip install adversary-pursuit` will keep returning v0.1.0.** As the actual codebase diverges further from v0.1.0, the released artifact becomes more misleading. At some point a new user installs v0.1.0 and finds it does not match the README — which has been updated for the dossier surface — and files a bug. That is when the release-discipline debt becomes visible to outside parties.
- **Dark-day clusters will continue.** Without a "pause log" or scheduled cadence, the 3-out-of-20 active-day ratio is the new normal until the next named strategic push.

If Vector B is picked up in the next week:

- The most prescriptive option is **release-discipline cut v0.4.x (option d from 06-09)**. The pipeline exists (`.github/workflows/release.yml` shipped at v0.1.0). The artifacts exist. The version-bump is one line in `pyproject.toml`. The decision is "what semver number captures what we shipped?" — and the answer is a small DEC family.
- The most *strategic* option is **MCP migration (option b)**. Issue #65 has been latent for 37 days. The dossier surface is now a candidate consumer of MCP. Picking #65 up would re-engage the strategic-roadmap muscle the project used during M-1..M-9.

### Intent-Trajectory Gap

The gap between the Original Intent and the current trajectory is **slightly larger than at 06-09**, but for a different reason than before. At 06-09 the gap was small because the project had just closed two major roadmaps that advanced the Intent. The gap is larger now not because the project has moved away from the Intent — it has not — but because **the project has stopped moving toward the Intent** in this window. The 6 load-bearing Intent commitments are at the same fulfillment state they were 20 days ago. That is the gap: 20 days of work and no Intent-fulfillment delta.

The Original Intent's crowdsourcing axis (commitment 6) is the test case. At 06-09 it was "architecturally tractable" (M-9 had shipped the file format). It is still "architecturally tractable" today. The next step — promoting the v1 Non-Goal "Federation" to a v2 Goal, or building a registry/share layer, or shipping a discoverable library — has not been taken. The architecturally-tractable state was meant to be a step *toward* federation; it has become a *destination* by default.

## VIII. The Reckoning

### Verdict: **Drifting constructively → drifting tactically**

The verdict changes from the prior reckoning's "on course" to **drifting constructively**. The drift is real, but the direction of the drift is into hygiene improvement, not into anti-product or anti-Intent territory. The project shipped 7 useful, well-scoped, well-tested, well-decisioned slices in 20 days. Every slice serves a real user concern (the user filed every one of #84, #97, #98 personally during their own use). The cmd2 REPL is better than at v1 ship. The error-coverage surface is universal. The banner is on-brand. The chat surface has command parity with cmd2. The novelty cache is working. **None of this is bad work.**

But: **the project has not made a strategic decision in 20 days.** The 06-09 reckoning's most prescriptive recommendation (item #4 — choose the next roadmap) is not advanced. The 06-09 reckoning's second-most-prescriptive recommendation (item #6 — codify the Active Phase Pointer as Guardian-landing concern) is *also* not done; it was handled informally by each closeout commit, which works but is not the structural fix the 06-09 reckoning called for. The project's *strategic clock* has been stopped since 06-09 even as its *tactical clock* has run.

The verdict downgrade from "on course" to "drifting constructively" reflects this. The drift is genuinely *constructive* — the codebase is healthier than it was — but it is also genuinely *drift*: the named strategic direction is not the actual direction. A project that says "v0.4.x release is next" and then patches 7 surfaces over 20 days without versioning anything is, by its own definition, off-direction.

This is what "drifting constructively" looks like in a mature project: the discipline holds, the code improves, the decisions are coherent, the tests pass, the worktrees are cleaned — and the strategic vector quietly idles. The 06-09 reckoning loop closed once at full strength; the loop has not re-engaged.

### What to Celebrate

- **DECISIONS.md is fixed.** `scripts/regen_decisions.py` ships (commit `3cf14a7`, 2026-06-10), issue #72 closed, 1,251-line registry regenerated 30 hours after the 06-09 reckoning landed. **Fastest-ever response to a named confront item.** The mechanism works.
- **The Active Phase Pointer is mechanically domesticated.** Every closeout commit (`ffc3cd4`, `df88836`, `8ac958b`, `b8a75af`, `105459e`, `d8c49f3`) updates the pointer line in the same merge as the slice it documents. Confront #1 of the 06-09 reckoning is solved.
- **Worktree-cleanup discipline recovered.** 15 worktrees → 3 worktrees (1 cwd, 2 stale). The collapse confronted in 06-09 #3 is largely addressed. The 2 stale ones (`feature-ap76-gitignore-audit`, `feature-error-routing-2026-06-11`) are a small forward debt, not the systemic 14-worktree backlog.
- **The cmd2 REPL surface got a serious revival.** Phase 17R fixed 8 defects — including Rich output being silently dropped into a StringIO buffer since v1 ship. The Metasploit-UX principle is *better* honored now than at v1.
- **Sacred Practice 12 was enforced post-hoc.** Phase 17T extracted a shared `core/module_credentials.py` resolver when a chat-vs-REPL credential-resolver divergence was discovered. **Discovering and unifying duplicate authorities is the hardest version of Sacred Practice 12 to maintain.** It happened.
- **Capability-restriction discipline at the LLM tool boundary held.** Phase 17P added `WorkspaceManager.clear()` and explicitly *did not* expose it as an LLM tool (per DEC-WORKSPACE-DB notes "agent must NOT have unconfirmed wipe capability"). The agent's tool surface is being curated, not just grown.
- **`hunt <ioc>` is real.** Phase 17R added the first fleet-dispatch primitive in AP — `hunt 1.2.3.4` runs every module that accepts IPv4 in parallel. This is closer to "Metasploit + CTFd" than the original `use → set → run` flow.

### What to Confront

1. **The strategic clock has been stopped for 20 days.** The 06-09 reckoning named 5 candidate next-roadmaps (M-10+ dossier follow-ons, MCP migration #65, runtime hygiene drain, release-discipline cut v0.4.x, federation). Twenty days later, **zero of them are scheduled.** Every Active Phase Pointer line since has repeated the refrain "Phase 17X is a tactical hygiene insert that does not displace the strategic direction" — but **the strategic direction has not advanced.** This is the most important finding in this reckoning.

2. **The latent-promises pile is growing, not shrinking.** Issues OPEN-unscheduled at this reckoning include #26, #27, #28, #33 (April), #58 (May 18, override-by-practice continues), #65, #66, #67 (May 23, MCP migration epic still untouched at 37 days), #76 (worktree provisioned but no slice), #77–#81 (M-5/M-6/M-7 follow-ups from dossier roadmap), #82 (release-discipline, 18 days from filing, captured but unscheduled), #83 (hooks UX), #85, #89, #90, #96, #100, #101 (orchestrator/runtime bugs). **At least 18 issues OPEN-unscheduled.** Prior reckoning had ~5. The "file the issue, don't schedule it" pattern is now the project's default response to discovered concern.

3. **Three dark-day clusters totaling 15 of 20 days are unrecorded.** 2026-06-12..18 (7 days), 06-20..22 (3 days), 06-25..29 (5 days, current). The plan has no schema for "deliberate pause" vs "unintended drift" — they look identical from outside. **If these were planned (holiday, research, reflection), the plan should say so. If they were drift, the project should know.** Neither answer is available right now.

4. **DECISIONS.md regeneration is a one-shot, not a contract.** `scripts/regen_decisions.py` was authored 2026-06-10 to fix issue #72 (the 6-week stale state). It has not been run since. Code has gained 18 DEC-IDs in the 19 days since. **Without a Guardian-landing hook or CI step, DECISIONS.md will silently slide back to 6+ weeks stale by mid-July** — the exact problem the script was written to solve. This is the prior reckoning's #2 finding *recurring at a smaller cadence*, not solved.

5. **The v0.4.x release-discipline gap is the most prescriptive un-acted-on recommendation.** The 06-09 reckoning's What-to-Do-Next #4-option-d names this. Issue #82 captures it. The mechanics are trivial: bump `pyproject.toml`, write a CHANGELOG, tag, push. The blocker is *not technical* — it is the unmade decision of "what semver number captures what we shipped between v0.1.0 and now?" The project shipped two major roadmaps (dossier M-1..M-9 + character C-1..C-4) on `version = "0.1.0"` lineage. **`pip install adversary-pursuit` returns v0.1.0 from 2026-05-19 — now 41 days behind production.** A first-time user who follows the README finds it does not match the artifact.

6. **A new debt category has appeared: orchestrator/runtime instability.** Issues #85, #89, #90, #91 (closed), #92 (closed), #93 (closed), #94 (closed), #95 (closed), #96, #100, #101 — at least 7 OPEN orchestrator/Guardian/hook bugs filed during this window. **This debt did not exist at 06-09.** It is not AP product code — it is the harness/runtime the project depends on to ship AP. The orchestrator chain that produced M-1..M-9 has started to surface racing, routing, and lease-cleanup bugs. Whether this is *harness drift* (the runtime changed under the project), *load surfacing latent bugs* (more sessions = more races discovered), or both — the project has not characterized it.

7. **Phase 17R produced a regression that Phase 17S had to fix within 96 hours — the first regression-chain in AP's life.** AP #97 (`hunt <ioc>` initializer passes wrong `Config` object) was a direct consequence of Phase 17R's new `_hunt_ioc` fleet-dispatch path. The tests in 17R did not exercise the real-world hunt path; the user discovered the regression in their own session. **The dossier-roadmap velocity was earned by tight test coverage of the M-x slices.** The hygiene-wave velocity is *not* earning the same coverage — and is producing chained-regression slices that did not exist during the dossier window. This is a leading indicator that quality discipline is degrading under tactical-slice rate.

8. **Phase 17Q boot banner redesign consumed two deep-research artifacts.** `DeepResearch_AP_AdversaryPursuit_Logo_2026-06-17/` + `DeepResearch_NewLogo_2026-06-17/`. A banner redesign is justifiable; **two deep-research sessions on logos when the project has 18 unscheduled issues including a stalled v0.4.x release-discipline cut is harder to justify.** This is what "avoidance work" looks like in a disciplined project: the work is real, the discipline is high, the priority is wrong.

### What to Do Next

1. **(Big decision — owner: user, blocked on user-only adjudication)** **Pick the next roadmap.** This was 06-09's #4 and is *unchanged* and *more urgent* 20 days later. The five options from 06-09 are all still on the table:
   - **(a) M-10+ dossier-axis follow-ons** (PII redaction, ingest-priors writer per DEC-M9-IMPORT-READONLY-001, multi-actor bundles, federation registry).
   - **(b) MCP migration epic #65** (vendor-neutral MCPs so non-`ap chat` LLMs can consume AP's tool surface; also resolves #66 Honeylabs + #67 go-roast).
   - **(c) Runtime hygiene backlog drain** (#49–#55, #58, #70, #71, #75, #76 + the new #85/#89/#90/#96/#100/#101 orchestrator-bug surface).
   - **(d) Release-discipline cut v0.4.x** (#82; closes the most concrete and lowest-risk gap; *strongly recommended as the next slice because it can be done in one day*).
   - **(e) Crowdsourcing federation** (promote v1 Non-Goal; build registry/share layer on M-9's local library).
   These remain mutually exclusive *in priority*. The reckoning loop demonstrated it can close one of these per ~14 days. The user is still the only one who can resolve direction. **Invoke `/reckoning operationalize` or `/decide` to convert this to a structured choice — do not file another tactical hygiene slice until this is picked.**

2. **(Small fix, 30 min)** **Cut v0.4.x.** This is the cheapest move from option-d above and is *separable* from option-1's broader strategic choice. Bump `pyproject.toml` `version = "0.4.0"`, write a CHANGELOG section covering the M-1..M-9 + C-1..C-4 + 17O→17U work, tag `v0.4.0`, push via `release.yml`. Close issue #82. **This is the cleanest single action that would re-engage the strategic clock.** It does not commit to a next roadmap; it just stops the "released artifact lags reality" drift.

3. **(Small fix, 5 min)** **Add `regen_decisions.py` to a Guardian-landing hook or CI.** Without this, `DECISIONS.md` silently slides back to 6+ weeks stale. The script exists; it just needs to be wired into the closeout flow. A line in the Guardian-landing Evaluation Contract ("run `scripts/regen_decisions.py` and commit any diff") would close this category permanently.

4. **(Small fix, 15 min)** **Worktree triage on the 2 stale ones.** `feature-error-routing-2026-06-11` is post-merge (17O landed 06-11) and should have been cleaned at landing time — bookkeeping miss. `feature-ap76-gitignore-audit` exists for issue #76 (30 days OPEN) but no slice is in flight; either land #76 as its own slice or remove the worktree.

5. **(Medium reflection, ~1 hour)** **Name what happened in the dark days.** The plan should either:
   - Add a "Pause Log" section to MASTER_PLAN.md acknowledging deliberate gaps (holidays, research time, reflection), OR
   - Acknowledge that the gaps were drift and name the cause (unscheduled time, unclear priority, fatigue), OR
   - Decide that pauses are private and not plan-tracked — but make that an explicit decision (DEC-PAUSE-001).
   The 15 dark days in this window cannot stay unexplained without setting up the next reckoning to find the same gap.

6. **(Medium triage, ~1 hour)** **Triage the orchestrator/runtime debt (#85, #89, #90, #96, #100, #101, #83).** These are not AP product code — they're harness debt. Three options:
   - File a roadmap-class slice ("Phase 18 — Orchestrator Stability") that drains them as a wave.
   - Acknowledge them as "runtime instability the project tolerates" and stop expecting them to drain organically.
   - Migrate to a more stable orchestration surface if one exists.
   The current pattern (file issue, ship AP work) cannot resolve a 7+ open orchestrator-bug pile.

7. **(Small fix, ~30 min)** **Close or comment-update the long-tail OPEN issues from prior reckoning #7.** #26 (cmd2 → chat console) — Phase 17P + 17R closed most of the gap; either close with cross-reference or scope a follow-on. #27 (HEF/Analysis/Persona) — partially done by C-1..C-4; close-with-cross-ref or scope follow-on. #28 (RAG knowledge base) — still zero presence, still no successor; schedule or formally retire. #33 (v2 PyPI docs) — re-scope for whether v0.4.x returns to PyPI. #58 (drain hygiene before v2) — close or re-affirm. **All five of these were on the 06-09 reckoning's What-to-Do-Next #7. None were addressed.** That itself is a 06-09 confront item recurring.

8. **(Reflection — for the planner)** **Codify a "next-roadmap-or-pause" decision boundary.** After two consecutive tactical-hygiene slices without a strategic vector advance, the project should be required to either (a) declare a new roadmap explicitly with a DEC-ROADMAP-* family or (b) declare a strategic pause explicitly. The current "Phase 17X is a tactical insert that does not displace the strategic direction" refrain is a known antipattern after appearing 7 times.

---

## Reckoning Delta (vs. 2026-06-09 reckoning)

| Dimension | 2026-06-09 | 2026-06-29 | Direction |
|-----------|-----------|-----------|-----------|
| Verdict | on course | **drifting constructively** | downgraded (strategic clock stopped for 20 days) |
| Maturity tier | Mature (29 phases, 287 in-code DECs) | **Mature (37 phases, 305 in-code DECs across 59 files)** | sustained |
| Intent alignment | Strong (strengthened) | **Strong (sustained, tactical bias)** | unchanged in surface, slowed in advance |
| Decision coherence | Strong (visible at deletion time) | **Strong (sustained; Sacred Practice 12 enforced post-hoc)** | maintained |
| LLM tool count | 30 (post-M-9) | 30 (unchanged) | flat — no new product-surface tools |
| Module count | 15 + 11 dossier | 15 + 11 dossier (unchanged; +2 core/* helpers) | +2 helper files in `core/` |
| Personas with `llm_profile` | 6 | 6 (unchanged) | C-roadmap remains closed |
| Cross-workspace authorities | 3 (`config.toml` + `dossier_novelty.sqlite` + `dossier_library/`) | **3 (no new authorities; no registry yet)** | unchanged — registry still not authored |
| Worktrees on disk | 15 | **3 (1 cwd, 2 stale)** | improved |
| `DECISIONS.md` staleness | 42 days | **19 days behind code, regeneration script exists but not in CI** | tooling fixed; freshness drift will recur |
| `pyproject.toml` version | v0.1.0 (41 days stale) | **v0.1.0 (still — 41 days stale, captured as #82 OPEN)** | unchanged |
| Open GitHub issues | ~15 OPEN | **~30 OPEN (≥18 OPEN-unscheduled)** | growing latent-promises pile |
| Active days / total | 14/14 (100%) | **5/20 (25%)** | 4× decline in active-day density |
| Phases shipped | 18 (in 14 days) | **7 (in 20 days), all tactical hygiene** | velocity halved, work-shape shifted |

### Resolved Findings (Prior Reckoning → Now)

- **Confront #1 (Active Phase Pointer stale within hours of landing)** → **RESOLVED.** Every closeout commit since 06-10 updates the pointer line atomically. Pattern held through 7 consecutive landings.
- **Confront #2 (`DECISIONS.md` 6 weeks stale, #72 OPEN 13 days)** → **MECHANICALLY RESOLVED, BUT NOT INSTITUTIONALLY**. `scripts/regen_decisions.py` shipped 2026-06-10. Registry regenerated. Issue #72 closed. **But** the script has not been re-run since — so freshness will silently drift again.
- **Confront #3 (15 worktrees on disk, 14 landed-but-uncleaned)** → **MOSTLY RESOLVED**. Down to 3 (1 cwd, 2 stale). The named discipline has been honored for most landings.
- **Confront #4 (MCP migration epic #65 latent 17 days)** → **WORSENED**. Now 37 days latent. Still zero in-plan presence. #66 and #67 same. The asymmetry against the dossier roadmap absorption is sharper.
- **Confront #5 (cross-workspace authorities accumulating without registry)** → **UNCHANGED**. Still 3 authorities (`config.toml`, `dossier_novelty.sqlite`, `dossier_library/`); no `AUTHORITIES.md` registry. Not yet a fourth, but the recommendation was to author the registry *before* the fourth, not after.
- **Confront #6 (No release-discipline decision since v0.1.0)** → **CAPTURED-NOT-ACTED**. Issue #82 filed 2026-06-11 (good — accountability). 18 days OPEN-unscheduled (bad — same pattern as #72 was for 13 days). The release-cut itself is one CHANGELOG + version bump away.
- **Confront #7 (#58 "drain runtime hygiene" override-by-practice)** → **UNCHANGED**. Issue #58 still OPEN at 42 days. Override-by-practice continues.

### New Findings (Not Present in Prior Reckoning)

- **The strategic clock has been stopped for 20 days.** This is the most important new finding. Seven consecutive tactical-hygiene phases shipped under the consistent refrain "does not displace the strategic direction" — without the strategic direction advancing.
- **Three dark-day clusters totaling 15-of-20 unrecorded days.** First sustained dark stretches since v1 ship. No causal record.
- **A new debt category appeared: orchestrator/runtime instability.** At least 7 OPEN harness/Guardian/hook bugs (#85, #89, #90, #96, #100, #101, #83). Did not exist at 06-09.
- **The first regression-chain in AP's life.** Phase 17R produced AP #97 that Phase 17S had to fix within 96 hours. Tests did not exercise the real-world hunt path. Leading indicator that tactical-slice rate is outrunning test discipline.
- **Avoidance-work signal: two deep-research artifacts consumed on logo/banner work while #82 release-cut is unscheduled.** The discipline is high; the priority is wrong.
- **`hunt <ioc>` is a new product capability** that snuck in via Phase 17R "REPL revival" — the first fleet-dispatch primitive in AP. Not framed as a strategic move but it *is* one. This is the closest the window comes to product expansion.

### Persistent Findings (Flagged Then, Still True)

- **Future Self promises pattern continues.** The 06-09 reckoning's recommendation to add a "Latent Promises" section with explicit retire-or-schedule deadlines was not done. The pile grew from ~5 to ~18.
- **The cross-workspace authority registry recommendation was not addressed.** Three authorities, no registry. Recommendation #5 from 06-09.
- **Long-tail issues #26, #27, #28, #33, #58 remain OPEN.** None were closed or comment-updated despite 06-09 What-to-Do-Next #7 naming them.
- **The single-authority chain is intact.** Sacred Practice 12 was actively enforced in Phase 17T (chat-vs-REPL credential resolver unified). The discipline survived velocity loss.
- **The principles (1–5) remain intact** with no new violations across 7 additional phases.
