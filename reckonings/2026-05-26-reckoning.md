# Project Reckoning: Adversary Pursuit — Post-v1 Direction

**Date:** 2026-05-26
**Source:** /Users/jarocki/src/ap/MASTER_PLAN.md
**Project age:** 51 days of active development since plan was filed (2026-04-05), preceded by 3.5 years of dormant 2022 vision README
**Maturity tier:** Mature (9 closed phases, 14 phase sections, 168 `@decision` annotations across 43 files, 48 dispatched work items, 23 open GitHub issues, 71 issues total, **`v0.1.0` stable shipped 7 days ago**)
**Initiatives:** 9 phases closed, 5 in-progress worktrees active (Phases 11–14 plus a still-open F62 worktree), 23 open issues
**Decisions:** ~70 DEC-* in the plan's per-phase tables + 48 distinct DEC-IDs surfaced in `DECISIONS.md` (auto-generated from code; last updated 2026-04-28 — stale by ~28 days)
**Predecessor:** `2026-04-29-reckoning.md` (interface-model correction; recommended W-AGENT-MODULES-VT-CENSYS-PT as next step)

---

## I. The Core

Adversary Pursuit is a **gamified threat-hunting framework** whose irreducible essence is captured in two phrases that the project's founder has elevated to architectural status: *"Taking maximum advantage of every mistake, and celebrating with epic memes,"* and *"fun is a first-class design constraint."* The founding tension is between two cultures that almost never co-exist in CTI tooling: the rigor of STIX 2.1 / OpenCTI / Maltego on the one hand, and the joy of CTFd / Metasploit-banter / msfconsole tab-completion on the other. AP exists because the founder believes those are not opposites — that you can simultaneously care about spec compliance and ASCII-art celebration without compromising either.

The implicit philosophy embedded in every design choice is **"the agent observes, the engines own."** Modules are pure STIX data producers. Gamification engines (scoring, badges, hints, modes, celebrations) are stateless observers of tool-execution events. The console — first cmd2, now `ap chat` — is *not* the heart; it is one of two interchangeable presentation layers over a shared substrate of modules + workspace + scoring + event bus. This is why the 2026-04-29 interface pivot (ADR-010, from cmd2-primary to chat-primary) was "architecturally cheap": the agent slot was always there in the design, just unfilled. The same shape now visible in issue #65 (MCP migration) confirms the philosophy was right — modules really are vendor-neutral capabilities that any orchestrator should be able to consume.

What makes THIS project THIS project is the refusal to accept a tradeoff between playfulness and rigor. Bobby Hill mode and `stix2.parse()` round-trip validation live in the same codebase, both as first-class citizens. The Sacred Practice 12 single-authority discipline that's been beating in this codebase's chest for two months is the rigor side of that bargain. The fact that the v1 ship featured *Columbo mode* alongside 168 `@decision` annotations is the proof.

## II. The Origin

Quoted verbatim from the Original Intent (as filed 2026-04-05, drawn from a 2022 dormant vision README):

> "Adversary Pursuit is a gamified framework for hunting, pivoting, and discovery of actor infrastructure, indicators, and TTPs. 'Taking maximum advantage of every mistake, and celebrating with epic memes.' Goals: Make it fun. Gamify. Different modes. Graph of pursuit progress. Teaching moments at dead ends. Meme/DALL-E generator for celebrations. Final report generation (interview-based). Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development."

This Original Intent contains six load-bearing commitments:

1. **Gamification is non-negotiable.** Listed first. Reinforced via "Make it fun. Gamify."
2. **Multiple modes.** Personality is a feature, not a skin.
3. **Graph of pursuit progress.** Visualizable investigation state.
4. **Teaching moments at dead ends.** The tool helps you learn, not just produce.
5. **Memes / celebrations.** Even DALL-E is named — explicitly. (v1 explicitly deferred AI-image generation to ASCII art per the Non-Goals.)
6. **Crowdsourcing / competition / ranking.** A community dimension.

The assumptions embedded in the Original Intent are noteworthy because of which ones held and which didn't:

- **Held:** The "feel like a combination of Metasploit and CTFd" interface assumption motivated cmd2 selection (#2) and the `use → set → run → score` flow. It survived through Phase 4. (ADR-010 then re-cast it as the power-user surface, but the shape persists.)
- **Held:** "Standardize hunting" predicted the STIX 2.1 commitment (ADR-005).
- **Held:** Free-tier-first API discipline (Phase 2 ordering rationale) preserved the "easy to start" gamification ethos.
- **Re-cast:** "Combination of Metasploit and CTFd" → became `ap chat` agent in front of `ap` cmd2 REPL (ADR-010, 2026-04-29). The user's clarification ("the interface needs to be an agentic AI chat") was not in the Original Intent — it was a 2026-era opportunity surfaced by the maturity of smolagents/litellm/Ollama. The plan absorbed this gracefully because modules were already pure data producers.
- **Unfulfilled (v1, not abandoned):** Crowdsourcing, competition, ranking for career development — the *social* gamification axis. v1 ships single-player. The Non-Goals explicitly defers "federation between AP instances" and "real-time collaboration." This is a deliberate scope contraction documented in plan text.
- **Unfulfilled (recently surfaced as scope drift candidate):** "Teaching moments at dead ends" — hints exist (W-AGENT-HINTS landed) but the *teaching* framing has not crystallized. Issue #68 (Threat Actor Dossier reframe) and the Threat Hunter expert advisories (issues #59, #60) are pushing the product toward "analytic value" framing that aligns with teaching-at-dead-ends but has not been planner-reconciled with the Original Intent yet.

The constraint that shaped the vision was *solo-developer sustainability* under "Why Now": AI-assisted development changes the equation; the 24-issue plan would have been multi-person work in 2022. This constraint has held with discipline — the project has shipped v1 in ~51 days of active development with a single primary author, which is direct evidence the framing was correct.

## III. The Journey

### Timeline

| Period | Initiative | Status | Key Decisions | Outcome |
|--------|-----------|--------|---------------|---------|
| 2022-11-24 | Initial vision README | dormant 3.5 years | (none) | Idea survived latent incubation |
| 2026-04-05 | MASTER_PLAN.md filed (Phase 0–5 ordering) | scoped | ADR-001..ADR-009 | 24-issue v1 plan instantiated |
| ~2026-04-06..04-15 | Phase 0 Foundation (#1–#5) | completed | DEC-CONSOLE-*, DEC-PLUGIN-*, DEC-WS-*, DEC-STIX-*, DEC-CONFIG-* | cmd2 console + plugin + workspace + STIX + config |
| ~2026-04-15..04-25 | Phase 1 Modules (#6–#13) | completed | DEC-MODULE-OTX-*, DEC-MODULE-URLSCAN-* (etc.) | 8 priority + 2 stretch modules |
| ~2026-04-25..04-28 | Phase 2 Gamification (#14–#18) | completed | DEC-SCORING-*, DEC-CHALLENGE-*, DEC-MODE-*, DEC-BADGE-*, DEC-HINT-* | 5 gamification engines |
| ~2026-04-28..04-29 | Phase 3 Auto-Pivot (#19–#20) | completed | DEC-EVENTBUS-*, DEC-GRAPH-* | Event bus + graph + export |
| **2026-04-29** | **ADR-010 interface pivot** (reckoning 2) | scope re-cast | ADR-010 | `ap chat` is primary; cmd2 is supporting |
| 2026-04-29..05-01 | Phase 6 W-AGENT-* (10 slices) | completed | DEC-AGENT-* (8 decisions), 10 W-AGENT-* slices | Full gamification parity in agent path |
| 2026-05-01..05-03 | Phase 6 closeout + W-AGENT-DOCS | completed | (none new) | README rewritten agent-first |
| 2026-05-03..05-15 | Phase 7 organic post-polish (~12 commits) | completed off-chain | (none filed) | Setup wizard, Censys v3, URLScan auth, smoke SKIP classification, TUI polish |
| 2026-05-03 | Distribution pivot PyPI → GitHub Releases | completed | (`02fed4d`, retroactive ADR) | Reduced supply-chain surface |
| 2026-05-15 | Phase 8 W-OTX-TIMEOUT | completed | DEC-MODULE-OTX-TIMEOUT-001/002 | OTX ReadTimeout → stub SCO pattern |
| 2026-05-16 | Phase 9 W-GREYNOISE | completed | DEC-MODULE-GREYNOISE-001..003 | 11th module (noise/RIOT axis) |
| **2026-05-18** | **Phase 5 close — `v0.1.0rc1` verification** | completed | DEC-V1-RELEASE-VERIFY-001..005 | Install path verified end-to-end |
| **2026-05-19** | **Phase 5 stable — `v0.1.0` ship** | completed | DEC-V1-FINAL-SHIP-001..004 | Stable release; stale release force-replaced |
| 2026-05-22 | Phase 10 W-FRIENDLY-ERRORS | completed | DEC-ERROR-INTERPRETER-001..008 | Universal `core/error_interpreter.py` |
| 2026-05-25 | Phase 11 W-59-STIX-PROVENANCE | landed (in-plan status stale) | DEC-59-STIX-PROVENANCE-001..007 | Per-SCO provenance + `stix2.parse()` round-trip |
| 2026-05-25 | Phase 12 W-60-AUTO-PIVOT-POLICY | landed (in-plan status stale) | DEC-60-PIVOT-POLICY-001..007 | 3-gate policy engine, `max_depth` removed |
| 2026-05-25 | F62 streak + honest modes | landed (in-plan status stale) | (no per-phase section authored) | Streak mechanic + persona honesty |
| 2026-05-26 | Phase 13 W-64-DEDUP-LLM-NARRATION | landed (in-plan status stale) | DEC-64-LLM-PANEL-SEPARATION-001 | Sidecar pattern; LLM stops double-narrating |
| 2026-05-26 | Phase 14 W-61-KEYLESS-HUNTERS | landed (in-plan status stale) | DEC-61-* (6 decisions) | +4 modules (15 total): URLhaus, ThreatFox, MalwareBazaar, crt.sh |

### Decision Density

In the ~30-day window from the prior reckoning (2026-04-29) to today (2026-05-26):

- **~46 commits to `main`** (per `git log --since 2026-04-29`).
- **~70 new DEC-IDs** (counting only the per-phase planner Decision Logs visible in MASTER_PLAN.md; not counting code-side @decision annotations).
- **15 distinct work items dispatched** through the canonical planner chain, plus the Phase 7 organic 12 commits that bypassed the chain.
- **9 phase closeouts authored**: Phase 5 verify, Phase 5 stable, Phase 8, Phase 9, Phase 10, Phase 11, Phase 12, Phase 13, Phase 14.

That is roughly **1 phase per 3 days** in May, with decision density spiking late in the month (Phases 11–14 all in the last 4 days). Two interpretations: (a) the team is hitting velocity escape — Phase 5 stable ship triggered a creative burst now that the v1 release-gate constraints are lifted; (b) post-v1 the planner is dispatching too much in parallel for the orchestrator's reconciliation cadence to keep up — note that the in-plan status for Phases 11, 12, 13, and 14 still says "planner stage complete, implementer next" despite the merges already on `main` (`f4a71a3`, `60eab19`, `e460b41`, `556f873`). Both interpretations are evidence-grounded.

### Inflection Points

1. **2026-04-29 ADR-010 (Interface Pivot).** Recast the project's primary UX from cmd2 to `ap chat`. Triggered by direct user clarification. Architectural cost was low because the modules + gamification substrate was already engine-pure. This was a *proactive* shift driven by user-vision clarification, not by failure.
2. **2026-05-03 Distribution Pivot (PyPI → GitHub Releases, `02fed4d`).** A *reactive* shift surfaced during release prep — credential / trusted-publisher surface was judged too large for a solo-maintainer pre-1.0 project. Retired the W-V1-PYPI-VERIFY work item.
3. **2026-05-15 OTX timeout stub-SCO pattern.** Generalized URLScan transient-failure behavior into a cross-module pattern (now reaffirmed by GreyNoise 404 handling). A *codification* inflection — preexisting practice was promoted to declared policy.
4. **2026-05-19 v0.1.0 Stable Ship.** Removed the v1 release gate. The single largest psychological inflection in the project's life. Post-ship, the planning cadence accelerated dramatically (4 phases in 8 days).
5. **2026-05-22 Threat Hunter Advisory (issue #59 / #60 origins).** External-expert pressure surfaced two real spec / safety gaps (STIX bundle non-compliance, quota-bomb cascades). Both produced in-progress phases. This was the project's first external-input-driven planning round.
6. **2026-05-23 Issue #68 "Threat Actor Dossier reframe".** Filed but not yet planner-reconciled. Proposes recasting scoring from indicator-graph expansion to **dossier-piece completion weighted by importance and rarity.** This is a candidate *vision-shift* — it has not yet been absorbed into the plan and it changes what the project is optimizing for. (Flagged in Section VIII.)

### Plan vs. Reality

This project enjoys unusually tight plan-vs-reality coupling: 168 `@decision` annotations in code, all DEC-IDs traceable through the plan or per-component `DECISIONS.md`. Two specific drifts emerged in the latest window:

- **`DECISIONS.md` is stale.** Auto-generated by `stop.sh` from in-code annotations; last updated 2026-04-28. ~28 days of post-Phase 6 decisions (DEC-MODULE-GREYNOISE-*, DEC-MODULE-OTX-TIMEOUT-*, DEC-V1-FINAL-SHIP-*, DEC-V1-RELEASE-VERIFY-*, DEC-ERROR-INTERPRETER-*, DEC-59-*, DEC-60-*, DEC-61-*, DEC-64-*) are in `MASTER_PLAN.md` and in source `@decision` annotations, but `DECISIONS.md` does not reflect them. The stop hook either stopped firing or stopped writing.
- **In-plan Phase 11/12/13/14 status text is stale.** The merge SHAs are on `main` (verified via `git log --oneline`) but the phase sections still read "Status: in-progress (planner stage complete, implementer next)." This is the same generative drift the 2026-04-29 reckoning saw in microcosm: the plan is right about what was *planned* and wrong about what has *shipped*. Closeout amendments lagging behind merges.

Neither drift is alarming, but together they reveal that the **doc-update step at end of each canonical chain is informally being skipped**. The planner-amend-MASTER_PLAN sub-task in every Evaluation Contract is being treated as ceremonial rather than load-bearing.

## IV. Evolution Assessment

### Intent Alignment: **Strong**

The project has shipped v1 doing exactly what the Original Intent asked it to do: gamified hunting + pivoting + discovery, with modes, graphs, celebrations, hints, interview-based reports, modular OSINT/CTI sources, and STIX 2.1 as the standardization fabric. Every Principle traces to in-code artifacts:

| Principle | Honored? | Evidence |
|-----------|----------|----------|
| Fun is a first-class design constraint | **Yes** | `gamification/celebrations.py` (DEC-CELEBRATION-001/002), 10 character modes, ASCII art, "Bobby Hill" mode shipped in v1, Phase 13 (F64) literally exists to protect the *panel* gamification surface from being trampled by LLM narration. |
| Metasploit UX is the interaction model | **Yes (re-cast by ADR-010)** | The `use → set → run` flow is alive in cmd2 (`core/console.py`); the agent path renames "set" to "tool argument" but the mental model survives — tools have options, results are scored, workspaces isolate investigations. |
| STIX 2.1 is the lingua franca | **Yes, hardening in progress** | All 15 modules emit STIX 2.1 SCOs. Phase 11 (W-59) explicitly closes the spec-compliance gap (`stix2.parse()` round-trip). The Threat Hunter advisory pressure that surfaced #59 is itself evidence the principle has external validity. |
| Modules are pure data producers | **Yes, defended actively** | F59 DEC-001 reaffirms this verbatim: "workspace.store_stix_objects() is the sole authority for x_ap_* fields; modules MUST NOT emit them." F61 DEC-MODULES-EMIT-NO-PROVENANCE-001 re-reaffirms for the new modules. This principle has *teeth* — it's enforced by tests. |
| Playfulness and rigor are not opposites | **Yes** | Sacred Practice 12 (single-authority discipline) is now invoked in nearly every per-phase DEC rationale. Sacred Practice 1–12 from CLAUDE.md show up in plan text as load-bearing constraints. And the project still shipped Columbo mode. |

### Principle Adherence — strong across the board. The DEC-MODULES-EMIT-NO-PROVENANCE-001 reaffirmation in F61 is a particularly healthy signal: the planner is now treating principle-restatement as part of the Evaluation Contract for every new module slice. That is exactly the architectural-preservation discipline CLAUDE.md prescribes.

### Constructive Expansions

The project has **doubled in scope without losing focus**, mostly through expansion that was implied-but-undeclared in the Original Intent:

- **Agent path (Phase 6).** Original Intent named "different modes" and "meme generator" but not "agentic AI chat." The expansion is constructive — it serves Principle 1 ("Fun is a first-class design constraint") with a UX that was unavailable in 2022. The carve-out in v1 Non-Goals is explicit and bounded.
- **F59 provenance (Phase 11).** Original Intent named "report generation" but not forensic provenance chain. The Threat Hunter advisory surfaced that "research toy" is not where this project wants to land. Adding provenance turns single-author hobby-grade evidence into auditable downstream-consumer-ready evidence. Constructive.
- **F60 quota-aware auto-pivot (Phase 12).** Original Intent named "auto-pivoting" but not "quota-aware." Same Threat Hunter pressure — default cascades on URLScan-fronted CDN domains burned free-tier API quotas. Adding the policy engine protects the "easy to start" promise of free-tier-first ordering. Constructive.
- **F61 keyless hunters (Phase 14).** First-five-minutes UX. A fresh-install user could SKIP-wall on every module except `dns_resolve` / `whois_lookup` because every CTI source needed a key. Adding 4 keyless hunters (URLhaus, ThreatFox, MalwareBazaar, crt.sh) means the "first query returns real evidence" promise is now true. Constructive.
- **Friendly errors (Phase 10).** Original Intent named "teaching moments at dead ends" but framed it as hints, not error handling. Phase 10 reads that intent broadly: errors are dead-ends, and they should teach. `core/error_interpreter.py` is a direct fulfillment of an Original Intent commitment that had been latent for two months.

### Scope Drift — **Candidate, not confirmed**

Issue #68 (Threat Actor Dossier reframe, 2026-05-23) is a *proposal* to recast scoring from indicator-graph-expansion to dossier-piece completion. If absorbed, this is not constructive expansion — it is a **redefinition of what the product values**. The Original Intent's wording ("hunting, pivoting, discovery of actor infrastructure, indicators, and TTPs") is ambiguous between these two framings:

- Today's framing: indicators-and-TTPs are the unit of value, scoring rewards finding them and pivoting through them.
- #68's framing: the **dossier** (a coherent picture of a Threat Actor: habits, motivations, "tells," predicted next moves) is the unit of value, scoring rewards dossier-piece completion weighted by importance.

#68 is not in scope yet. It is a filed issue, not a planner stage. But the **steer-mode question** (Section VIII) is whether v2 should be a refactoring of v1 (more modules, deeper gamification, web UI) or a re-foundation (dossier-puzzle, MCP migration via issue #65, persona prediction). The current plan has no answer.

### Non-Goal Violations — **None**

A scan against v1 Non-Goals shows no violations:

- Web/GUI: not built. Issue #33 (PyPI/docs for v2) is filed but not in-plan.
- Mobile: not built.
- Jupyter: not built.
- Federation: not built.
- Cloud/VM: not built.
- AI-classification: the agent dispatches tools and presents results; it does not invent classification heuristics. F60's missing-confidence "optimistic" default explicitly avoids the agent making up confidence values it doesn't have.
- 3D rendering, character sheets, real-time collab: not built.
- DALL-E celebrations: explicit Non-Goal in v1; ASCII art celebrations only (DEC-CELEBRATION-001).

The v1 Non-Goals discipline held. This is rare and worth naming.

### Abandoned Threads

- **Original `cti/misp` stretch module** named in Phase 2 — never implemented; not filed as backlog. The MISP query path is implicitly covered today by F61's `cti/threatfox` (which is the same data shape) so the abandonment is functionally fine, but the doc still names it as a stretch goal.
- **Original `pivoting/domain_to_ip` and `pivoting/email_recon` stretch modules** — never built. Auto-pivot via the event bus (F19/#20) covers some of this in flow form; the explicit `pivoting/` namespace exists as an empty namespace package but holds no modules. This is a *deferral* rather than abandonment — but no follow-up issue has been filed.
- **Crowdsourcing / competition / ranking / career-development axis.** Original Intent named it; v1 deferred it; no issue exists for v2 absorption. (Issues #29–32 cover RPG/leveling/LLM personas/SATs but not crowdsourcing per se.)
- **Phase 7 "organic" 12 commits.** Documented retroactively in MASTER_PLAN, but the lesson — *"live-smoke regressions should be filed as discrete planner slices"* — was learned and applied (Phase 8 W-OTX-TIMEOUT went through the canonical chain). Healthy resolution.

## V. Decision Quality

### Coherence: **Strong**

Decision quality has *improved* in the post-v1 window. The Phase 11–14 DEC rationales are noticeably tighter:

- Cross-DEC references: F60 DEC-001 cites Sacred Practice 12 and CLAUDE.md "Encode authority, don't imply it" — the project's own architecture doctrine is being cited in its decision rationales. That's how doctrine becomes durable.
- Removal-targets discipline: every recent phase has an explicit "Removal targets (addition without subtraction is debt)" sub-section. F60 removes `PivotConfig.max_depth` entirely (no deprecation shim). This is the unified-implementation answer Sacred Practice 12 demands.
- Forbidden-shortcuts catalogs: F59, F60, F61, F64 all enumerate forbidden shortcuts that close known-anti-pattern doors before they get used. Build-then-regex-filter (F64), env-var bypass (F60, F64), live HTTP in unit tests (F61), parallel catalogs (F10) — these are all *named and ruled out* in the planner stage.

### Notable Decision Chains

- **The single-authority invariant chain**: ADR-005 (STIX 2.1) → DEC-WS-004 (dedup) → DEC-STIX-001/002 (deterministic ids) → DEC-59-STIX-PROVENANCE-002 (provenance does not feed id derivation) → DEC-61-MODULES-EMIT-NO-PROVENANCE-001. Five decisions across 7 weeks defending the same invariant. Coherent.
- **The transient-failure chain**: DEC-MODULE-URLSCAN-* (`5cc2be6`/`26c5b54`) → DEC-MODULE-OTX-TIMEOUT-002 (mirrors URLScan pattern) → DEC-MODULE-GREYNOISE-002 (404→unknown stub) → DEC-MODULES-EMIT-NO-PROVENANCE-001's "x_<vendor>_status" convention. Pattern crystallized into doctrine.
- **The interface-pivot chain**: Original Intent (2022) → ADR-001 (cmd2 over Textual) → DEC-CONSOLE-001..004 → #25 smolagents landing → ADR-010 (2026-04-29) → DEC-AGENT-ARCH-001..002 → 10 W-AGENT-* slices → W-AGENT-DOCS. Substantive evolution, every link cited downstream.

### Decision Gaps

- **Phase 7 12 commits have no DEC-IDs in the plan.** Each commit's `@decision` annotations exist in code (verified via `grep`), but the Phase 7 narrative table just lists merge SHAs and one-line rationales. The "lesson" was captured as policy ("file live-smoke regressions through the canonical chain") but the decisions themselves bypassed the planner.
- **F62 (streak + honest modes) has no per-phase section** in MASTER_PLAN.md. The merges are on `main` (`1d424ae`, `8b0faa2`, `e3cf5ca`), and `tests/test_streak.py` exists, but there's no Phase 12.5 / Phase 13 entry. The `tmp/evidence-62-streak-and-honest-modes` directory exists and the worktree is preserved. **This is a closeout gap** — a slice landed without amending the plan.
- **Phases 11, 12, 13, 14 status text is wrong** (still says "implementer next" though merges are landed). Closeout-amend-MASTER_PLAN sub-tasks went unfulfilled.
- **`DECISIONS.md` is 28 days stale.** The `stop.sh` regeneration step is silently no-op-ing. No issue filed.

### Traceability

168 `@decision` annotations in 43 source files maps cleanly into the 70+ plan-side DEC-IDs in most cases. The gap is in the direction of `code → plan`: the new decisions land in code annotations and per-phase plan tables, but the consolidated `DECISIONS.md` registry is the *one* surface where a future implementer would look first, and it's stale. Mechanical, not conceptual.

## VI. Project Health

| Indicator | Rating | Evidence |
|-----------|--------|----------|
| **Vitality** | **Thriving** | 46 commits since last reckoning, 9 phases closed in 28 days, 4 phases landed in last 4 days. v1 stable shipped. Multi-author-equivalent throughput from a solo developer + Claude Code. |
| **Focus** | **Moderate (with warning)** | v1 shipped clean and on-vision. Phases 10–14 each have crisp scope manifests + 9-key evaluation contracts. **But** 5 worktrees are open simultaneously (Phases 11, 12, 14 merged; F62 and F64 worktrees still dirty); the "5 worktrees" SubagentStart context line points to this. Three days from now this is a clean concurrency story or a context-collapse story depending on which way the cleanup goes. |
| **Momentum** | **Accelerating** | Phase cadence is increasing post-v1 ship. Decision density is up. Closeout-amend lag is the early warning signal that throughput is exceeding the doc-reconciliation rate. |
| **Coherence** | **Strong** | DEC chains hold across phases. Sacred Practices are cited in rationales. F61 explicitly reaffirms F59 authority. No contradictions in the Decision Log. The only coherence risk is the stale `DECISIONS.md` registry. |
| **Sustainability** | **Sustainable with watchpoints** | Solo-developer load is real but stable. The bigger risk is **plan absorption of issue #68 (Dossier reframe) and #65 (MCP migration epic)** — these are not yet planner-stage but represent two different futures. Choosing between them is sustainability work, not feature work. |

## VII. Trajectory

### Current Vector

The project is in **post-v1 hardening mode driven by external (Threat Hunter) expert pressure.** Phases 11–14 are all responding to advisory-grade pressure that surfaced after v1 ship:

- F59 (provenance) responds to "I cannot put this in an advisory. Until every result is timestamped + URL-attributed + content-hashed at the workspace layer, this is a research toy."
- F60 (auto-pivot policy) responds to "Default config is hostile to anyone with a free-tier key. I cannot recommend AP until the cascade is throttled."
- F61 (keyless hunters) responds to "fresh users hit a SKIP wall — every CTI module except dns/whois needs a key."
- F64 (de-dup LLM narration) responds to a UX-discipline boundary the user named directly: "Pick one — either the LLM gets to be the announcer, or the panel does."

The vector is: **make AP credible to professional threat hunters** — production-grade evidence chain, quota-safe defaults, first-query-returns-data UX, no double-narration UX bugs. This is a *credibility* trajectory.

### Projected Destination

If the current rhythm continues unchanged for 3 months:

- 15 → ~25–30 modules (F61 + F61b CIRCL + GitHub Issues #65/66/67 MCP integrations + the unbuilt `cti/misp` + a few more keyless sources).
- Full STIX 2.1 round-trip on all SCO types (the `file` SCO follow-up filed in F59 / F61 will close).
- A renderer for F60's `decision_log` (filed follow-up).
- Persistent quota counters in workspace SQLite (filed follow-up).
- Streak panel (filed follow-up from F64).
- Crystallization of the `tests/test_pivot_policy_integration.py`-style end-to-end "scenario" tests as a project pattern.
- Probably one or two more Threat Hunter advisories surface, each producing one phase.

But this projection has a **branch point** that's already visible: **issue #68 (Dossier reframe) + issue #65 (MCP migration epic) are not in plan.** Three months from now, either they remain idle (the project continues v1.x incremental hardening) or they get planner-absorbed (the project pivots to v2 with a re-foundation). The plan as it stands suggests the former. The user's revealed preferences via issue filing suggest the latter is on their mind.

### Intent-Trajectory Gap

The gap between the Original Intent and the current trajectory is **small but conceptually meaningful**:

- The Original Intent's "Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development" axis remains **unaddressed**. v1 ships single-player. The current trajectory is "make the single-player tool credible to professional hunters" — which is *adjacent* to the original goal but not the same as it.
- The Original Intent's "career development" framing implies a **community / multi-player / leaderboard** dimension that has zero in-plan presence. No issues filed. No backlog item. This is a Future Self promise that has quietly slipped.
- Issue #68's Dossier reframe is *closer* to the Original Intent than today's trajectory — "piecing together a picture of a Threat Actor: their habits, strengths, tells, anything that can match activity to their persona fingerprint or predict what they'll do next" reads like a direct echo of "Standardize hunting and pursuit techniques." If anything, #68 is the Original Intent's voice clarifying itself.

The gap is therefore not large in magnitude but it points to a **product-direction decision** that the planner has not yet made: is v2 a continuation of v1's UX (incremental hardening), or a partial re-foundation around the Dossier framing?

## VIII. The Reckoning

### Verdict: **drifting constructively**

The project has shipped v1 cleanly against its Original Intent, with strong principle adherence, no Non-Goal violations, and improving decision discipline. The post-v1 work has expanded beyond original scope but every expansion serves the founding vision — provenance, quota safety, keyless first-query UX, and friendly errors are all in-spirit fulfillments of "fun is a first-class design constraint" and "teaching moments at dead ends." The decision-coherence chains held, the architecture-preservation rules from CLAUDE.md are being cited in DEC rationales (the doctrine has become operational), and Sacred Practice 12 single-authority is now reflexive rather than aspirational.

What keeps this from being "on course" is the appearance of three plan-absorption gaps that the planner has not yet metabolized: (a) **issue #68's Dossier reframe** which is conceptually closer to the Original Intent than today's incremental trajectory, but is unscoped; (b) **issue #65's MCP migration epic** which would re-cast modules from in-process Python to vendor-neutral MCP servers (a v2-grade architectural shift); (c) **the crowdsourcing / competition / career-development axis** named in the Original Intent that has zero in-plan presence. None of these is a fault — they are unmade decisions about v2's center of gravity. The project is constructive *and* drifting because it is making local progress without making a v2 product-direction call.

The watchpoints are mechanical, not conceptual. Phase-closeout MASTER_PLAN-amendments are lagging behind merges. `DECISIONS.md` regeneration has silently broken (last update 2026-04-28). Five worktrees are open simultaneously and the orchestrator context line says "6 dirty." The plan-vs-reality coupling that has been this project's superpower is fraying at the doc-update step. None of this threatens the codebase; all of it threatens the doc trust that downstream Future Implementers depend on.

### What to Celebrate

- **`v0.1.0` shipped clean and on-vision** (2026-05-19, tag SHA `e669b5d`, commit `e8e9b13`). 11 modules, full gamification parity in `ap chat`, fresh-venv install verified end-to-end, stale-release force-replaced cleanly without consumer harm. v1 had a release gate and the gate closed.
- **The Sacred Practice 12 single-authority chain is now operational doctrine.** F59 reaffirms it, F60 cites it, F61 quotes it. Five DEC chains link across 7 weeks defending the same invariant. The architecture-preservation rules from CLAUDE.md have become muscle memory in the planner.
- **Zero Non-Goal violations across nine landed phases.** Web/GUI, mobile, Jupyter, federation, cloud, DALL-E, 3D rendering, character sheets, real-time collab, AI-classification — all v1 deferrals held. Rare and worth naming.
- **Threat Hunter expert pressure became Phases 11 and 12 in 3 days.** The project responded to external credibility feedback by planning real fixes, not by deferring or arguing. This is exactly the loop a credibility trajectory needs.
- **The keyless-first-query UX gap was caught before v2.** F61 ships 4 strictly-keyless hunters specifically so fresh installs return real evidence on the first query. This serves Principle 1 ("fun is a first-class design constraint") at the first-five-minutes UX layer where the principle is most fragile.

### What to Confront

1. **Closeout-amend-MASTER_PLAN sub-tasks are being skipped.** Phases 11, 12, 13, 14 all show "in-progress (planner stage complete, implementer next)" status in plan text even though their merges are on `main`. F62 (streak + honest modes) has no per-phase section at all despite landing 2026-05-25. The planner's own Evaluation Contracts name this step ("MASTER_PLAN.md amended with closeout SHA + evidence summary") as part of `ready_for_guardian` — and Guardian shipped anyway. Either the gate is informal or the gate is being bypassed.

2. **`DECISIONS.md` is silently stale.** Auto-generated by `stop.sh` from in-code `@decision` annotations; last updated 2026-04-28; ~28 days behind. The 168 in-code annotations have grown but the consolidated registry has not. A Future Implementer asking "what decisions own this surface?" will be misled. No GitHub issue exists tracking this.

3. **Issue #68 (Dossier reframe) is the most important unmade decision in the project.** Filed 2026-05-23 with substantial body content reframing scoring from indicator-graph expansion to dossier-piece completion weighted by importance. It is conceptually closer to the Original Intent than today's trajectory. The planner has not addressed it. Every day it sits unscoped is a day the project's v2 center of gravity is unset.

4. **The Original Intent's crowdsourcing / competition / career-development axis has fully fallen off.** Zero issues filed. Zero in-plan presence. The 2026-04-29 reckoning didn't surface it either. This is a *Future Self promise* that has gone latent for 7+ weeks without being explicitly retired or scheduled. CLAUDE.md's "Future Implementers rely on you" principle would say: either schedule it or formally retire it via a Non-Goal entry.

5. **Five concurrent worktrees with three already-merged.** F59, F60, F61 are landed but their feature branches and worktrees remain on disk. F62 worktree is still dirty (`MASTER_PLAN.md` modified). F64 worktree is preserved. This is the "Branch and Worktree Cleanup" gap CLAUDE.md names explicitly: *"Pushing is not cleanup by itself. After a successful push/landing and an idle, clean worktree: switch the checkout back to the long-lived base branch when needed... delete the local task branch... remove the task worktree."* Three landed worktrees have not been cleaned.

6. **Phase 7 ("organic" 12 commits, 2026-05-03..05-15) set a precedent that the plan claims to have learned from but the post-v1 cadence is at risk of repeating.** Phase 11–14 went through the canonical chain — good. But the closeout-amend lag (item 1 above) is the same family of bypass at a smaller scale. The lesson — *"file live-smoke regressions through the canonical chain so the chain owns them"* — needs an equivalent for documentation: *"plan amendments are not optional; Guardian landing is gated on them."* This is not enforced today.

### What to Do Next

1. **(Small fix, ~1 hour)** Amend MASTER_PLAN.md to mark Phases 11 (W-59), 12 (W-60), 13 (W-64), 14 (W-61) as `completed` with their merge SHAs, and author a missing Phase 12.5 / 13 / 14 section for F62 (streak + honest modes). Use `git log` + the existing closeout-section template. Single docs-only commit. Closes the most visible plan-vs-reality drift in the project.

2. **(Small fix, ~30 min + investigation)** File a GitHub issue: *"`stop.sh` `DECISIONS.md` regeneration is silently no-op-ing (last update 2026-04-28; 168 in-code @decision annotations vs 48 rows in registry)."* Investigate whether the hook is failing, the trigger is wrong, or the regeneration logic missed Phase 7+'s commits. Decide whether to fix the hook or to declare manual regeneration as the model. Either way, get `DECISIONS.md` current to today.

3. **(Big decision — invoke `/decide` or `/reckoning operationalize`)** Adjudicate issue #68 (Dossier reframe) before scheduling any more incremental v1.x phases. The decision is: is v2 a **continuation** of v1 (more modules, deeper gamification, web UI eventually, MCP migration via #65 happening orthogonally) or a **re-foundation** (dossier-puzzle scoring, persona prediction, denial-strategy generation)? These are not mutually exclusive in implementation but they are mutually exclusive in *what the planner optimizes for next*. The Original Intent's wording supports the dossier framing; the trajectory supports the continuation framing. The user is the only one who can resolve this — but the planner can present it as a structured `/decide`.

4. **(Small fix, ~10 min)** Clean up the three already-landed worktrees (F59, F60, F61): switch each back to `main`, delete the merged feature branches locally, and `git worktree remove` the directories. Verify F62 worktree's dirty `MASTER_PLAN.md` change is either landed or discarded before cleanup. (Be careful — F62 is the *only* worktree whose plan section is still missing; the dirty edit might be the planner-amend draft for that.)

5. **(Medium docs slice — sized like a planner task)** Reconcile the Original Intent's **crowdsourcing / competition / ranking / career-development** axis with the current plan. Decide: either (a) file an issue scheduling it as a future phase (and add it to "Next Work Items"), or (b) add a v1 Non-Goal entry explicitly retiring it for v1 and capturing the rationale. CLAUDE.md's "Future Implementers rely on you" principle would object to it remaining latent indefinitely.

6. **(Reflection — for the planner, not an implementer)** Codify the closeout-amend-MASTER_PLAN sub-task as a Guardian-landing gate rather than a planner-stage Evaluation Contract item. Today it's named in the Evaluation Contract but Guardian appears to land without verifying it. If the doc-update step is load-bearing for Future Implementers (it is), the gate has to be in Guardian's preflight, not the planner's contract. Otherwise the post-v1 acceleration will continue to outpace the documentation reconciliation rate, and the project's plan-vs-reality coupling — its current superpower — will erode.

---

## Reckoning Delta (vs. 2026-04-29 reckoning)

The prior reckoning was a **focused correction** (ADR-010 interface model), not a full seven-dimensional analysis. The comparison below tracks what moved between then and now.

| Dimension | 2026-04-29 | 2026-05-26 | Direction |
|-----------|-----------|-----------|-----------|
| Verdict | "interface model corrected; agent gamification gap is the v1 critical path" | **drifting constructively** | new dimension (prior reckoning was scope-specific, not whole-project) |
| Maturity tier | "Active (5 of 6 phases landed; Phase 5 + Phase 6 remain)" | **Mature** (9 phases closed, v1 stable shipped) | +1 tier |
| Intent alignment | not explicitly rated | **Strong** | new finding |
| Decision coherence | implicit-strong via the ADR-010 carve-out discipline | **Strong, and more explicit** — Sacred Practice 12 cited in DEC rationales, removal-targets discipline universal | improved |
| Module count | 10 in cmd2, 7 in agent (parity gap) | **15 in both** (W-AGENT-MODULES-VT-CENSYS-PT closed gap, then W-GREYNOISE +1, then W-61-KEYLESS-HUNTERS +4) | gap closed; capacity expanded |
| Agent gamification parity | 2 of 11 surfaces (scoring, workspace) | **11 of 11** (all W-AGENT-* slices landed) | fully closed |
| `v0.1.0` ship | "W-V1-PYPI-VERIFY remains a real v1 boundary" | **shipped stable 2026-05-19** (`v0.1.0`, tag `e669b5d`, commit `e8e9b13`); distribution pivoted PyPI→GitHub Releases | shipped, with strategic pivot |

### Resolved Findings (Prior Reckoning → Now)

- **W-AGENT-MODULES-VT-CENSYS-PT recommended → completed (`66f89dd`)**. The recommended-next-step closed.
- **W-AGENT-CELEBRATIONS (MLP critical path) → completed (`4ccc5888`)**. The signaled-highest-priority gap closed.
- **9 W-AGENT-* slices total → all completed**. Phase 6 fully closed (closeout 2026-05-01).
- **The "Honest gap report" 11-row parity table → 0 rows remain open.** Every cmd2-only surface has been mirrored into the agent path.
- **W-V1-PYPI-VERIFY** (named in prior reckoning as remaining) → **retired by the 2026-05-03 GitHub Releases pivot** (`02fed4d`), and W-V1-RELEASE-VERIFY (`cd3709a`) + W-V1-FINAL-SHIP (`e8e9b13`) replaced it cleanly.
- **`@decision` annotation gap (76% coverage)** → assessed as informational-not-blocking; no follow-up filed; coverage discussion absorbed into per-phase planning instead. Resolved by treating the metric as advisory.

### New Findings (Not Present in Prior Reckoning)

- **Threat Hunter external advisory pressure** (issues #59, #60) has become a primary driver of post-v1 phase prioritization. This was not a vector in the 2026-04-29 reckoning because v1 had not shipped yet.
- **Issue #68 (Dossier reframe)** — filed 2026-05-23, not yet planner-absorbed. Conceptually closer to Original Intent than today's trajectory. **The most important unmade decision in the project.**
- **Issue #65 (MCP migration epic)** — filed late May; represents v2-grade architectural reframe (modules → MCP servers) that has zero in-plan presence.
- **Closeout-amend-MASTER_PLAN lag.** Phases 11/12/13/14 status text reads "in-progress" though merges are landed; F62 has no phase section. New class of drift not present pre-v1.
- **`DECISIONS.md` silently stale** (last update 2026-04-28). `stop.sh` regeneration broken — no GitHub issue tracks this.
- **Five concurrent open worktrees** (3 of which are already-landed but uncleaned). New scaling-of-concurrency pattern post-v1.
- **The Original Intent's crowdsourcing axis** — explicitly named in 2022 vision, still 100% latent at v1+7 days. Not in prior reckoning's surface; surfaced here because v1 has shipped without it.

### Persistent Findings (Flagged Then, Still True)

- **The interface-pivot worked because modules + gamification engines were architecturally separated from the console.** This is still true and is now load-bearing for the *next* reframe pressures (issues #65 and #68 both depend on the same separation holding).
- **The principles (1–5) are intact.** Still true. F61 reaffirmed Principle 4 ("modules are pure data producers") explicitly via DEC-MODULES-EMIT-NO-PROVENANCE-001.
- **The "Future Self promises" pattern.** The prior reckoning didn't flag this by name, but the crowdsourcing axis and the `cti/misp` stretch module were both latent then and remain latent now. The pattern needs a named-and-managed discipline.

---

