# Project Reckoning: Adversary Pursuit — Two Roadmaps Closed, Original Intent Within Reach

**Date:** 2026-06-09
**Source:** /Users/jarocki/src/ap/MASTER_PLAN.md (3199 lines)
**Project age:** ~65 days of active development since plan filed 2026-04-05 (preceded by 3.5 years of dormant 2022 vision README)
**Maturity tier:** **Mature** (29 phase rows, 21+ closed phases since v1, 32 DEC-M9-* in plan, 287 unique DEC-IDs in code across 57 annotated files, **two parallel post-v1 roadmaps closed in 14 days**: v0.3.x dossier (M-1→M-9) and v2 character (C-1→C-4))
**Initiatives:** 21+ phases marked completed; Phase 17N M-9 merged today at `9cff5b0` though the plan still shows it as "reviewer + guardian pending"
**Decisions:** 287 unique DEC-IDs in code (`grep` count), 100+ DEC-* in plan per-phase tables; `DECISIONS.md` registry remains stale at 2026-04-28 (now ~6 weeks behind)
**Predecessor:** `2026-05-26-reckoning.md` (verdict: drifting constructively; flagged closeout-amend lag, DECISIONS.md staleness, #68 as the most important unmade decision, crowdsourcing axis as latent Future Self promise)

---

## I. The Core

Adversary Pursuit's irreducible essence is unchanged since the prior reckoning: a gamified threat-hunting framework where "fun is a first-class design constraint" and the implicit philosophy is **"the agent observes, the engines own."** What is new on 2026-06-09 is that the project has finally answered the question the 2026-05-26 reckoning named as its most important unmade decision. The Threat Actor Dossier reframe (issue #68) — flagged then as a "candidate, not confirmed" scope-drift question — has not just been answered, it has been *built*. Nine implementation slices (M-1 through M-9) executed in fourteen days have moved AP's center of gravity from "indicator-graph expansion" to "dossier-piece completion weighted by importance and rarity." The Original Intent's words "standardize hunting and pursuit techniques" now have a concrete artifact behind them: a 9-slot dossier schema with per-slot weights, persistent state via the F63 sentinel-row pattern, an active falsification engine, a denial-strategies extractor, dossier-aware auto-pivot ranking, dossier-themed celebrations and badges, novel-method recognition with a cross-workspace hash cache, and now — landed today — a STIX 2.1 export / import / comparison pipeline plus an opt-in local actor library.

The founding tension between rigor and playfulness has not just held; it has *deepened*. The new dossier surface ships STIX 2.1 round-trip-validated bundles (DEC-M9-STIX-MAPPING-001/002) **and** five new dossier-themed badges (Full Dossier / Identity First / Predictor / Skeptic / Deception Spotter) **and** a Pioneer badge for novel hunting methods. The new persona Columbo carries non-empty `context_hooks` referencing real dossier slot vocabulary — the character system has become semantically coupled to the analytic substrate, in exactly the way the Original Intent's "different modes" + "teaching moments at dead ends" implied. The Sacred Practice 12 single-authority chain has held across every M-x slice: M-5 made `core/workspace.py` bytewise-unchanged stronger than M-4's narrow-edit clause; M-6 made `PivotPolicy.evaluate` bytewise-unchanged; M-7/M-8/M-9 each enumerate their bytewise-unchanged surfaces explicitly in their DEC tables. The pattern has become muscle memory.

What makes THIS project THIS project — the refusal of the tradeoff between playfulness and rigor — is now demonstrably ratified by what the project shipped after asking itself "are we drifting?" in late May. The answer was: the drift candidate was the Original Intent re-asserting itself, not a deviation from it. AP responded by metabolizing #68 into product, not by deferring it. That is exactly the loop a healthy living document should produce.

## II. The Origin

Quoted verbatim from the Original Intent (2026-04-05 filing, drawn from a 2022 dormant vision README):

> "Adversary Pursuit is a gamified framework for hunting, pivoting, and discovery of actor infrastructure, indicators, and TTPs. 'Taking maximum advantage of every mistake, and celebrating with epic memes.' Goals: Make it fun. Gamify. Different modes. Graph of pursuit progress. Teaching moments at dead ends. Meme/DALL-E generator for celebrations. Final report generation (interview-based). Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development."

Re-reading the Original Intent at this moment in the project's life produces a striking observation that the 2026-05-26 reckoning hinted at but couldn't yet prove: the dossier reframe (#68) is closer to the Original Intent's voice than the indicator-graph framing that v1 shipped with. "Standardize hunting and pursuit techniques" reads as the dossier-puzzle's structural commitment: a finite slot schema (9 slots, versioned, with weights) that says "this is what a complete picture of an adversary looks like." The April-2026 plan filed v1 under a different reading of those words — modules, scoring, pivoting — and that reading was correct enough to ship a working product. But the 2022 wording always had a second reading available, and the May-2026 Threat Hunter advisories surfaced enough analytic-credibility pressure that the second reading became the planner's actual answer.

Three commitments from the Original Intent have moved this window:

- **"Different modes"** — was a static catalog of 10 character modes at the start of this window. Is now (post-C-4 closure today) a 10-mode catalog where 6 modes carry LLM-driven `LLMPersonaProfile` voice composition with a frozen schema and persona-swap tool-call-identity hard gates, and 1 of those 6 (Columbo) carries dossier-aware `context_hooks` that read M-4 slot vocabulary. Personality is now a function of analytic state, not a skin.
- **"Final report generation (interview-based)"** — was the literal v1 implementation in `core/report.py` at the start of this window. Was preserved verbatim through one M-7 release cycle as `--style classic`. Was deleted in M-8 (DEC-68-DOSSIER-REFRAME-008 deprecation runway expiry) and is now exactly one renderer: `core/dossier_report.py` actor-dossier report. The Original Intent's "interview" framing has been superseded by the dossier-puzzle framing. This is a real product-philosophy shift that the plan documented honestly.
- **"Crowdsource pursuit, competition and ranking for career development"** — was the Future Self promise the 2026-05-26 reckoning named as latent for 7+ weeks. Is now (today, M-9) architecturally tractable: STIX 2.1 dossier export, read-only import, slot-by-slot comparison weighted by `SLOT_WEIGHTS`, and a local opt-in `~/.ap/dossier_library/` library. **The crowdsourcing axis has not been built; it has been made buildable.** The library is local + opt-in by design (DEC-M9-LIBRARY-OPTIN-001 / `AP_DOSSIER_PUBLISH=on` consent gate; v1 Non-Goal "Federation" continues to bind). But the artifacts a future federation would publish — the per-actor dossier bundles — exist now. This is the right shape for the crowdsourcing axis to take its first step: a portable file format the user explicitly moves themselves.

## III. The Journey

### Timeline (since the prior reckoning, 2026-05-26)

| Date | Phase | Status | Key Decisions | Outcome |
|------|-------|--------|---------------|---------|
| 2026-05-26 | 12B/13/14 closeouts (F62/F64/F61) | landed earlier same day | DEC-62/64/61-* | Streak + dedup + 4 keyless hunters captured in-plan |
| 2026-05-27 | 12C (W-63) milestone catch-up | merge `8778af3` | DEC-63-* (F63) | streak_continued ScoreEvent landed |
| 2026-05-27 | 16 (#68 reframe scoping) | merge `b2b846a` | DEC-68-DOSSIER-REFRAME-001..010 | Dossier-puzzle ratified as v2 center; M-1..M-9 decomposed |
| 2026-05-27 | 17 (#30 character v2 scoping) | merge `fe4c0b1` | DEC-30-CHARACTER-V2-001..007 | LLMPersonaProfile schema v1.0; C-1..C-4 decomposed |
| 2026-05-28 | 17B (M-1 dossier panel) | merge `486a5ad` | DEC-M1-DOSSIER-001..004 | First dossier package; Rich panel + 3 inferred slots |
| 2026-05-28 | 17C (C-1 full_troll) | merge `e49e70b` | DEC-C1-FULLTROLL-001..005 | First LLM persona; F62/F64 invariant gates |
| 2026-05-29 | 17D (M-2 slot extractors) | merge `11b3fd3` | DEC-M2-DOSSIER-001..005 | 4 real extractors; `get_dossier_state` LLM tool |
| 2026-05-29 | 17E (C-2 ninja) | merge `f8bded8` | DEC-C2-NINJA-001..003 | Ninja disposition flipped KEEP_STATIC → UPGRADE |
| 2026-05-29 | Plan-drift fix (#74) | merge `de08b4b` | (none) | M-1/C-1/M-2/C-2 phase sections harvested into plan |
| 2026-06-01 | 17F (M-3 dossier scoring) | merge `2809b13` | DEC-M3-DOSSIER-001..005 | `dossier/scoring.py`; per-IOC rules re-tuned to 1/1 |
| 2026-06-02 | 17G (M-4 persistent dossier) | merge `f928149` | DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006 | `dossier/state.py` + `dossier/predictions.py`; F63 sentinel-row storage |
| 2026-06-07 | 17H (M-5 denial + falsify + notes) | merge `e29e8b1` | DEC-M5-DENIAL/NOTE/FALSIFY-001..008 | Slot 9 extractor + `AnalystNote` authoring + active falsification |
| 2026-06-08 | 17I (M-6 dossier-aware pivot) | merge `1e5e09d` | DEC-M6-PIVOT-001..009 | `core/dossier_pivot.py` ranker over F60 gates |
| 2026-06-08 | 17J (M-7 reports + celebrations + badges) | merge `55aa1fe` | DEC-M7-REPORT/CELEB/BADGE-001..* | `core/dossier_report.py` + LLM narration + 5 dossier badges |
| 2026-06-09 | 17K (M-8 cleanup + novelty) | merge `16acaa3` | DEC-M8-CLEANUP-001..004 + DEC-M8-NOVELTY-001..010 | Classic-shim removed; Pioneer badge + novelty cache |
| 2026-06-09 | 17L (C-3 sun_tzu + bruce_lee + bureaucrat) | merge `3f33a5b` | DEC-C3-PHILOSOPHY-001..006 | Three philosophy/bureaucracy personas |
| 2026-06-09 | 17M (C-4 columbo + roadmap closure) | merge `9a6a550` | DEC-C4-COLUMBO-001..006 + 101..104 | Columbo + dossier-aware `context_hooks` + tier-1 KEEP_STATIC + mastery_level retired; **v2 character roadmap CLOSED** |
| **2026-06-09** | **17N (M-9 crowdsourced dossier)** | **merge `9cff5b0` (today)** | DEC-M9-* (32 in plan, 15 unique in code) | **STIX export + read-only import + comparison + local opt-in library; v0.4.x dossier surface open; original-intent crowdsourcing axis architecturally tractable** |

### Decision Density

In the 14-day window from the prior reckoning to today:

- **47 commits to `main`** (per `git log --since 2026-05-26 | wc -l`).
- **~100 new DEC-IDs** across MASTER_PLAN.md per-phase tables (DEC-68-DOSSIER-REFRAME-001..010 + DEC-30-CHARACTER-V2-001..007 + DEC-M1..M-9-* + DEC-C1..C-4-*).
- **287 unique DEC-IDs in code** (versus ~168 at the prior reckoning) — a 70% growth in 14 days. Code-level decision annotation has accelerated *faster* than the in-plan rate, which is the opposite of the closeout-amend-lag direction.
- **18 phase sections authored or closed** since 2026-05-26 (Phase 12C through Phase 17N).
- **Two parallel roadmaps closed in the same calendar week**: v0.3.x dossier (M-1→M-9) and v2 character (C-1→C-4).

That is roughly **one phase per ~0.8 days** in the M-1 through M-9 / C-1 through C-4 windows — the post-v1 acceleration the 2026-05-26 reckoning flagged as a watchpoint has not slowed down; it has compounded. But unlike the May watchpoint, the doc-update cadence has *also* compounded: every M-x and C-x slice ships with its per-slice plan at `.claude/plans/dossier-m<n>-<topic>.md` or `.claude/plans/character-c<n>-<topic>.md`, a Decision Log table, a Scope Manifest summary, an Evaluation Contract summary, and an Out-of-Scope list. The closeout amendments are happening — sometimes co-shipped in the implementer commit itself per the AP #74 orphan-prevention pattern that the plan-drift fix established on 2026-05-29.

### Inflection Points

1. **2026-05-27 (Phase 16 + Phase 17 strategic scoping).** Issue #68 ratified as v2's product center. The "candidate scope drift" of the prior reckoning resolved into "intentional product evolution." This is the single largest evolution event in the project since v1 ship. *Proactive* — driven by the prior reckoning's explicit naming.
2. **2026-05-28 / 2026-05-29 (parallel M-1/C-1 + M-2/C-2 waves).** First proof the dossier-roadmap and character-roadmap could ship in parallel without coupling. This validates the architectural separation that ADR-010 and the F-* invariant chain have been protecting — modules + gamification engines + dossier package + character profiles are four orthogonal authorities.
3. **2026-05-29 (plan-drift fix `de08b4b`, AP #74).** First explicit named-and-systemic closeout-amend-lag remediation. Established the AP #74 orphan-prevention pattern (planner amendments co-shipped in implementer commit). This pattern was honored consistently through M-3..M-9 / C-3..C-4.
4. **2026-06-02 (M-4 persistent dossier via F63 sentinel-row pattern, `f928149`).** First demonstration that a major new product capability (persistent DossierState + Predictions Log auto-validation) could be added **without** a SQLAlchemy schema change. F63's sentinel-row authority is now the canonical extension point for "I need to persist a new kind of thing without inventing a parallel storage authority." This is doctrine becoming reusable.
5. **2026-06-08 (M-7 LLM narration via `AgentRunner.narrate`).** First time the LLM is invited into the *gamification* path (celebration narration), not just the tool-dispatch path. Done with strict guardrails: threshold ≥ 2.5 slot weight, 80-token cap, 3-narrations/hunt budget, silent runtime fallback to ASCII, loud-fail in tests. The "fun is a first-class design constraint" principle has matured: fun is now LLM-generated when warranted and ASCII-art otherwise, with the threshold itself a typed budget.
6. **2026-06-09 (today, three landings in one day).** M-8 closes v0.3.x dossier roadmap (16acaa3). C-3 lands philosophy + bureaucratese personas (3f33a5b). C-4 closes the v2 character roadmap (9a6a550). M-9 (today, just merged at 9cff5b0) opens the v0.4.x dossier surface and resolves the 7-week-latency call on the Original Intent crowdsourcing axis. **Two roadmaps closed and a third opened in 24 hours.** The post-v1 acceleration is no longer the watchpoint — its compounding *speed* is.
7. **The non-event:** Issue #65 (MCP migration epic) and #58 (drain runtime hygiene backlog before v2 planning) — flagged in the 2026-05-26 reckoning as the *other* unmade decisions — remain unmade. Issues #65, #66, #67 (MCP/Honeylabs/go-roast integrations) are filed but unscoped. The runtime hygiene backlog (#49/#50/#51) is explicitly deferred as "opportunistic" in the M-9 plan. This is a deliberate prioritization — dossier and character first, infrastructure-of-tooling second — but it has not been re-examined since v1.

### Plan vs. Reality

The plan-vs-reality coupling that was the project's superpower at the prior reckoning has **bifurcated**: the dossier and character roadmaps have *excellent* in-plan/in-code traceability, while two other surfaces have degraded:

**Tight coupling (improved):**
- 18 phase sections authored with per-slice DEC tables, Scope Manifests, Evaluation Contracts.
- Per-slice planning artifacts at `.claude/plans/dossier-m<n>-*.md` and `.claude/plans/character-c<n>-*.md` co-shipped with each merge.
- 287 unique DEC-IDs in code; M-9 alone added 15 new DEC-M9-* annotations matching the 32 in-plan DEC-M9-* entries.
- AP #74 orphan-prevention pattern reliably co-shipping planner amendments in implementer commits.

**Drifts (some persistent, one new):**
- **`DECISIONS.md` is now ~6 weeks stale** (last self-update 2026-04-28; file mtime 2026-05-25). The prior reckoning flagged this at 28 days behind; it has now grown to ~42 days. **Issue #72 was filed** ("stop.sh DECISIONS.md regeneration is silently broken") — that's accountability — but the issue has remained OPEN for 13 days without remediation. The fix is not blocked; it is unscheduled.
- **The Active Phase Pointer at MASTER_PLAN.md:3199 is wrong.** It says "M-9 implementer landed @ `7cc801b`; reviewer + guardian pending" — but `git log -3` shows the reviewer ran, the closeout commit `cbb738f` landed, and the Guardian merge `9cff5b0` completed today. The pointer line is stale by hours. This is the same class of closeout-amend lag the prior reckoning flagged, recurring at the *fastest* cadence the project has ever sustained.
- **15 worktrees on disk** (versus 5 at the prior reckoning). Of those 15, at least 9 represent already-landed work (`feature-30-c1-full-troll-profile`, `feature-30-c2-ninja-profile`, `feature-30-c3-philosophy-bureaucrat`, `feature-30-c4-columbo`, `feature-59-stix-provenance`, `feature-60-auto-pivot-policy`, `feature-61-keyless-hunters`, `feature-62-streak-and-honest-modes`, `feature-64-dedup-llm-narration`, `feature-68-m1-dossier-panel`, `feature-68-m5-denial-strategies`, `feature-68-m6-dossier-pivot`, `feature-68-m7-reports-celebrations`, `feature-68-m8-cleanup-novelty`, `feature-68-m9-crowdsourced-dossiers`). The 2026-05-26 reckoning flagged "five concurrent worktrees with three already-merged" as Confront item 5. The number tripled. None of the prior cleanup happened, and 10 more uncleaned worktrees joined them. **Disk cleanup discipline has effectively collapsed in this window.**
- **Issue #74 (MASTER_PLAN.md missing Phase 17B/17C cross-references)** was *re-filed* by the orchestrator on 2026-05-29 as a one-off remediation issue, then closed by the plan-drift fix the same day. That the lag had to be remediated as a one-off issue (not absorbed into the next phase's planner work) suggests the lag is being treated as exceptional rather than systemic.

The two drifts together name the same underlying pattern: **the project's plan-amend discipline operates at the per-slice level (excellently) but lags at the cross-slice authority registry level**. Per-slice DEC tables are healthy; `DECISIONS.md` and the Active Phase Pointer (both cross-slice authorities) are stale.

## IV. Evolution Assessment

### Intent Alignment: **Strong (strengthened since prior reckoning)**

The dossier reframe has moved AP closer to its Original Intent, not further from it. Every Principle still traces to in-code artifacts, and three Principles have *new* and stronger evidence in this window:

| Principle | Honored? | Evidence (new in this window) |
|-----------|----------|-------------------------------|
| Fun is a first-class design constraint | **Yes (deepened)** | M-7 LLM celebration narration with strict typed budget — fun is now generative when warranted, ASCII otherwise. 5 new dossier badges (M-7) + Pioneer badge (M-8) = the achievement registry grew 50% in this window. Columbo persona (C-4) is dossier-aware — voice reacts to analytic state. |
| Metasploit UX is the interaction model | **Yes** | `use → set → run` flow unchanged. New `dossier`/`do_dossier` meta-command + subcommand router (M-9 DEC-M9-CHAT-METACMD-001) extends the meta-command pattern without breaking it. |
| STIX 2.1 is the lingua franca | **Yes (extended)** | M-9 dossier export uses `core/graph.py::export_stix_bundle` python-stix2 round-trip authority (DEC-59-STIX-PROVENANCE-005). Slot→STIX mapping (DEC-M9-STIX-MAPPING-001) uses native fields where semantics match (`threat-actor.aliases` / `resource_level` / `roles`) and `x_ap_*` custom props for the rest. STIX 2.1 is now the wire format for the crowdsourcing axis, not just internal storage. |
| Modules are pure data producers | **Yes, defended actively** | M-4/M-5/M-6/M-7/M-8/M-9 all explicitly enumerate `core/workspace.py` bytewise-unchanged invariants in DEC tables. The single-authority discipline is now operational doctrine at *every* planner-stage. |
| Playfulness and rigor are not opposites | **Yes (proven repeatedly)** | C-1..C-4 ship LLM-personality voice composition (playful) under persona-swap-tool-call-identity hard gates (rigor). M-9 ships STIX 2.1 export (rigor) plus optional library opt-in keyed off env var (rigor) — and the same merge week shipped Columbo (playful). |

### Constructive Expansions

Every expansion in this window serves the founding vision and was driven by either the Original Intent or principles-doctrine pressure:

- **Dossier-puzzle reframe (M-1..M-9, Phases 17B/D/F/G/H/I/J/K/N).** The largest single expansion in AP's life. Reads the Original Intent's "standardize hunting and pursuit techniques" as a structural commitment to a slot schema. Honors the prior reckoning's call to adjudicate #68. Architecturally clean — uses F63 sentinel-row pattern for storage (no schema change), python-stix2 round-trip for export, opt-in env var for federation.
- **LLM persona voice (C-1..C-4, Phases 17C/E/L/M).** Reads the Original Intent's "different modes" as personality-is-a-feature, not a skin. Frozen schema (`LLMPersonaProfile`), ≤165 tokens/mode budget, persona-swap tool-call-identity hard gate, fourth-wall-stance discipline. C-4 introduces dossier-aware `context_hooks` — character voice now coupled to analytic substrate, exactly as the Original Intent's "teaching moments at dead ends" implied.
- **LLM celebration narration (M-7 `AgentRunner.narrate`).** Reads "fun is a first-class design constraint" as "fun should be authored by the most expressive author available when the moment warrants it" — but under typed budgets so the discipline doesn't drift into LLM-spam.
- **Cross-workspace novelty hash cache (M-8 `~/.ap/dossier_novelty.sqlite`).** The first cross-workspace persistent authority in AP (alongside `~/.ap/config.toml`). Detects novel `(slot, extractor, sorted(sco_types))` triples globally. This is the *infrastructure* for the Original Intent's "career development" axis — a per-user hunting-method ledger. Pioneer badge (RARE, threshold 1) is its first consumer.
- **Local opt-in actor library (M-9 `~/.ap/dossier_library/`).** The crowdsourcing axis's first concrete artifact. Local-only, opt-in via `AP_DOSSIER_PUBLISH=on`, path-override via `AP_DOSSIER_LIBRARY`. Honors v1 Non-Goal "Federation" while making the federation step (if ever taken) a file-transport question rather than a re-architecture question.

### Scope Drift — **None confirmed in this window**

The candidate scope drift from the prior reckoning (#68 Dossier reframe) has been resolved as **intentional product evolution**, not drift. Issue #68 is now CLOSED on GitHub (closed 2026-06-09 after M-9 merge). The 2026-05-26 reckoning's Confront item 3 is fully addressed.

What *could* be scope drift but isn't yet:
- **Cross-workspace authorities are accumulating** — `~/.ap/config.toml` (since v1), `~/.ap/dossier_novelty.sqlite` (M-8), `~/.ap/dossier_library/` (M-9). Three is a pattern. There is no `~/.ap/AUTHORITY_INDEX.md` registry — if a fourth gets added without one, this becomes drift. Not yet.

### Non-Goal Violations — **None**

A scan against v1 Non-Goals shows no violations:
- Web/GUI: not built.
- Mobile: not built.
- Jupyter: not built.
- **Federation: not built** — M-9 opt-in library is the *opposite* of federation; it requires the user to manually move files, which is exactly the "no network upload" invariant the Non-Goal asserts. DEC-M9-LIBRARY-OPTIN-001 surfaces the privacy contract at the consent boundary. The line was respected.
- Cloud/VM: not built.
- AI-classification: M-9 does not auto-classify campaigns or cluster TTPs; comparison is mechanical (per-slot diff, weighted completion, prediction-validation ratio). DEC-M9-PRIVACY-001 makes the no-redaction position explicit, which forces the user to think about classification rather than letting AP think for them.
- 3D rendering, character sheets, real-time collab: not built.
- DALL-E celebrations: still ASCII art. M-7 LLM narration is text-only.

The v1 Non-Goals discipline has held across two parallel roadmaps and 18 phase closeouts. This is now demonstrated discipline, not aspiration.

### Abandoned Threads

- **MCP migration epic (#65)** — filed 2026-05-23, still OPEN, zero in-plan presence. The prior reckoning flagged this; nothing has moved. Issues #66 and #67 (Honeylabs MCP, go-roast MCP integrations) are also OPEN with zero in-plan presence. **All three remain abandoned-or-deferred** for 17 days now.
- **Issue #28 (CTI knowledge base for RAG)**, **#27 (Three-phase prompt system HEF/Analysis/Persona architecture)**, **#26 (Conversational chat console)** — three issues from late April that predate the v2 strategic scoping. Issue #26 was effectively absorbed by ADR-010 (chat is primary UI) and could be closed with a cross-reference. Issue #27's "Persona architecture" is at least partially fulfilled by C-1..C-4. Issue #28 (RAG) has zero in-plan presence and no successor disposition.
- **Issue #33 (Documentation and PyPI release for v2)** — filed 2026-04-27, marked `phase-5`, still OPEN. The v1 GitHub-Releases pivot (`02fed4d`) retired the v1 PyPI question; whether v2 returns to PyPI is unscoped. The labeling is now stale (it's not phase-5; it's a forward-looking question).
- **Runtime hygiene backlog (#49/#50/#51/#52/#53/#54/#55/#58 + #70/#71/#75/#76)** — 12 open issues. The M-9 plan explicitly defers them as "opportunistic." Issue #58 (meta-issue to drain them before v2 planning) was filed 2026-05-18 in the spirit of "deal with this before we plan v2" — but v2 (dossier roadmap) was planned and shipped without draining them. The principle that filed #58 was overridden by practice. **This is not a fault — dossier-first was the right call — but #58 should be re-evaluated or closed.**

## V. Decision Quality

### Coherence: **Strong (sustained)**

Decision quality has continued to improve. The Phase 17B..17N DEC tables show patterns the prior reckoning credited as healthy and they've grown sharper:

- **Cross-DEC reference density:** M-9's DEC table cites DEC-68-DOSSIER-REFRAME-001..010 (upstream scoping), DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 (M-1), DEC-M2..M-8-* (every intermediate slice), DEC-59-STIX-PROVENANCE-005 (the round-trip authority it builds on), DEC-DB-002 (no schema change), and F59/F60/F62/F63/F64 invariants. **One DEC table cites 14 upstream decision families.** That's how doctrine becomes durable.
- **Bytewise-unchanged enumeration:** M-9 lists `core/workspace.py` / `models/database.py` / `models/stix.py` / every existing `dossier/*.py` EXCEPT `__init__.py` / every `gamification/*.py` / `agent/runner.py` / `pyproject.toml` as bytewise unchanged. Five+ surfaces explicitly enumerated as untouched per slice has become the F-style invariant boundary.
- **Removal-targets discipline:** M-8 (DEC-M8-CLEANUP-001..004) executes the deprecation-runway expiry that was scheduled at M-7 (DEC-68-DOSSIER-REFRAME-008): deletes `core/report.py` + three classic LLM tools + the `--style` parser + classic tests + the `ToolContext.report_generator` field. Single-authority discipline is now visible at *deletion* time, not just creation time.
- **Anti-pattern enumeration:** M-9 names DEC-M9-NO-NEW-BADGE-001 / DEC-M9-NO-WORKSPACE-EDIT-001 / DEC-M9-NO-EVENT-001 / DEC-M9-COMBINED-SLICE-001 / DEC-M9-IMPORT-READONLY-001 / DEC-M9-PRIVACY-001 — six explicit "NO" decisions that close known-anti-pattern doors before they're opened. Decision-by-negation discipline is now standard.

### Notable Decision Chains

- **The dossier-puzzle chain**: DEC-68-DOSSIER-REFRAME-001 (slot schema v1.0) → DEC-M1-DOSSIER-001..004 (panel + read-only inference) → DEC-M2-DOSSIER-001..005 (extractors) → DEC-M3-DOSSIER-001..005 (scoring) → DEC-M4-PERSIST-001..003 + DEC-M4-PRED-001..006 (persistence + predictions) → DEC-M5-DENIAL/NOTE/FALSIFY-* (denial + falsification) → DEC-M6-PIVOT-001..009 (dossier-aware pivot) → DEC-M7-REPORT/CELEB/BADGE-* (output surfaces) → DEC-M8-CLEANUP/NOVELTY-* (cleanup + novelty) → DEC-M9-* (crowdsourcing axis). **80+ DEC families linked across 9 slices defending one product reframe.** This is the largest coherent chain in the project's life.
- **The character-persona chain**: DEC-30-CHARACTER-V2-001..007 (scoping) → DEC-C1-FULLTROLL-001..005 (schema + first profile + persona-swap hard gate) → DEC-C2-NINJA-001..003 (disposition flip; supersession discipline) → DEC-C3-PHILOSOPHY-001..006 (single-file-source slice pattern) → DEC-C4-COLUMBO-001..006 + 101..104 (dossier-aware `context_hooks` + tier-1 KEEP_STATIC reclassification + `mastery_level` permanent retirement). 30+ DEC families across 4 slices closing a roadmap with explicit roadmap-closure decisions (101-104). Coherent.
- **The single-authority chain (continues from prior reckoning)**: ADR-005 → DEC-WS-* → DEC-STIX-* → DEC-59-STIX-PROVENANCE-* → DEC-61-MODULES-EMIT-NO-PROVENANCE-001 → DEC-M3-DOSSIER-001 → DEC-M4-PERSIST-001 → DEC-M5-NOTE-001 → DEC-M9-NO-WORKSPACE-EDIT-001 (M-9 makes `core/workspace.py` bytewise-unchanged the *strongest* version yet by adding zero read helpers). Six new links in 14 days.

### Decision Gaps

- **`DECISIONS.md` registry is 6 weeks stale.** 287 in-code DEC annotations vs ~48 rows in the registry. Issue #72 is open and unaddressed for 13 days. This is the prior reckoning's gap, *grown*.
- **No cross-workspace authority registry.** `~/.ap/config.toml` + `~/.ap/dossier_novelty.sqlite` + `~/.ap/dossier_library/` are three persistent cross-workspace authorities. No single document enumerates them. M-8 and M-9 each named theirs in their respective DEC tables, but a future Implementer asking "what cross-workspace state does AP own?" has no consolidated authority to read.
- **Active Phase Pointer is stale within hours of landing.** M-9 merged at `9cff5b0` today and the pointer line still says "implementer landed @ `7cc801b`; reviewer + guardian pending." The closeout-amend cadence is *fast* but not yet atomic with merge.

### Traceability

Quantitatively *stronger* than the prior reckoning despite the growth:
- 287 unique DEC-IDs in code (was 168). +70% in 14 days.
- 57 annotated source files (was 43). +33% in 14 days.
- ~100 new in-plan DEC-* entries.
- All M-9 DEC-IDs in the plan map to either source annotations or are deliberate planner-only governance decisions (e.g., DEC-M9-COMBINED-SLICE-001 is a process decision, not a source-code decision).

The plan-side and code-side are growing in lockstep. The *registry* (`DECISIONS.md`) is the laggard, and that's a tooling problem, not a discipline problem.

## VI. Project Health

| Indicator | Rating | Evidence |
|-----------|--------|----------|
| **Vitality** | **Thriving (sustained)** | 47 commits in 14 days; 18 phase sections authored/closed; two roadmaps closed in 24 hours; M-9 (largest new-surface slice in the project's life with 4 co-shipped sub-capabilities) merged today. |
| **Focus** | **Sharp (improved)** | The roadmap structure (M-1..M-9, C-1..C-4) is the focus mechanism. Every slice is sized to one merge; per-slice plans + Scope Manifests + Evaluation Contracts are the load-bearing focus discipline. Despite 15 worktrees on disk, only one workflow is in flight at a time per roadmap. |
| **Momentum** | **Accelerating, sustainably** | Phase cadence is now ~1 per 0.8 days through the M-x/C-x window. Unlike the prior reckoning's "accelerating with closeout lag" concern, the AP #74 orphan-prevention pattern has kept per-slice documentation atomic with merge. Per-slice docs are healthy. *Cross-slice* docs (`DECISIONS.md`, Active Phase Pointer, worktree cleanup) lag. |
| **Coherence** | **Strong (deepened)** | DEC chains span 9 slices defending one product reframe. Cross-slice references are routine. Sacred Practice 12 single-authority cited in every M-x and C-x DEC table. Doctrine has become reflex. |
| **Sustainability** | **Sustainable but watchpoint shifted** | Solo-developer + Claude Code throughput is steady at ~3 phases/week. The risk is no longer "doc absorption lag" (that's been domesticated at the per-slice level); it's now: (a) cross-workspace authority sprawl without registry, (b) worktree-cleanup-discipline collapse, (c) the unscoped MCP/runtime-hygiene/PyPI backlogs accumulating while v2 product surfaces sprint. The watchpoint is **infrastructure-of-tooling debt**, not product debt. |

## VII. Trajectory

### Current Vector

The project has shifted vectors in this window — twice.

From the May 2026-05-26 reckoning to roughly 2026-05-27, the vector was **"make AP credible to professional threat hunters"** (the Threat Hunter advisory phase chain). From 2026-05-27 through today, the vector is **"build the dossier-puzzle as the analytic substrate the Original Intent always implied"** (the M-1..M-9 + C-1..C-4 paired roadmaps). The first vector was reactive (responding to external expert critique); the second vector is *proactive* (responding to the prior reckoning's named question about #68).

What is striking about the second vector is that it has been *executed*. Most "make a strategic decision" calls in a project's life leave the strategic decision pending while incremental work continues around it. This window's strategic decision (#68 dossier reframe) was named, ratified, decomposed, and built. **The reckoning loop closed.** That is unusually disciplined behavior for any project, let alone a solo-developer one.

### Projected Destination

If the M-1..M-9 + C-1..C-4 closure rhythm continues unchanged for 3 months:

- **The cross-workspace authority surface will grow.** `~/.ap/` will accumulate registries, caches, and library directories. Without a registry, this becomes a future cleanup or migration headache.
- **The MCP migration epic (#65)** will either get planner-absorbed (becoming the next M-class roadmap as v0.5.x / v3) or will quietly die. The dossier roadmap proved that a planner-absorbed big-question gets executed; an unabsorbed big-question waits.
- **The crowdsourcing axis itself** will face the federation decision. M-9 made federation buildable but not built. v0.4.x can ship "publish to your own library + share via filesystem"; the question "do AP instances publish to a shared registry?" is a v1 Non-Goal today and will require user adjudication to lift.
- **A v0.4.x or v1.x release** is likely within weeks. The current `pyproject.toml` is still at the v0.1.0 lineage. Whether the M-1..M-9 + C-1..C-4 closure warrants a v0.2.0/v0.3.0/v1.0.x bump is an unmade release-discipline decision.
- **Runtime hygiene backlog (#49..#55, #58, #70, #71, #75, #76)** continues to defer. If the post-v1 acceleration continues without infrastructure attention, the orchestrator/Guardian quality-of-life debt compounds invisibly.

### Intent-Trajectory Gap

The gap between the Original Intent and the current trajectory is **smaller than at any prior reckoning**.

The Original Intent's six load-bearing commitments are now in this state:
1. **Gamification non-negotiable** — fulfilled (scoring + 16 badges + 10 modes + dossier celebrations + novelty Pioneer badge).
2. **Multiple modes** — fulfilled (6 LLM-driven personas + 4 static; Columbo carries dossier-aware context_hooks).
3. **Graph of pursuit progress** — fulfilled (graph + GEXF + STIX bundle export + M-9 dossier-bundle export).
4. **Teaching moments at dead ends** — partially fulfilled (hints + friendly errors); the dossier panel itself functions as a "what do I know about this actor" pedagogical surface.
5. **Memes / celebrations** — fulfilled in ASCII; DALL-E remains a v1 Non-Goal (deliberately).
6. **Crowdsourcing / competition / ranking / career development** — **architecturally tractable as of today**. M-9 ships the file format. The cross-workspace novelty cache (M-8) ships the per-user method ledger. The Pioneer badge ships the recognition. The federation step is a deliberate user-adjudicated next decision.

Commitment 6 has moved from "100% latent for 7+ weeks" (prior reckoning) to "architecturally addressed in the local layer." This is the single largest evolution this project has executed.

## VIII. The Reckoning

### Verdict: **On course**

The verdict changes from the prior reckoning's "drifting constructively" to **on course**. The reasoning: the prior reckoning's "drifting" rating named three unmade decisions (#68 dossier, #65 MCP, crowdsourcing axis), and the drift was about the *non-decision*. The project has now made the #68 decision and built it through M-1..M-9. It has made the C-1..C-4 character decision and built it through to closure. It has made the crowdsourcing axis architecturally tractable via M-9. The drift was the absence of those decisions; the decisions are now made and shipped. The Original Intent's words are visible in code in ways they weren't six weeks ago.

The remaining unmade decisions (#65 MCP migration, runtime hygiene backlog) are now isolated to **infrastructure-of-tooling**, not product direction. That is a much smaller class of drift than the May version. It is real, and Section VIII names it, but it does not warrant a "drifting" verdict for a project that has, in 14 days, ratified its v2 product center, executed its v2 roadmap, closed two parallel implementation tracks, and resolved a 7-week-latency Future Self promise.

This is what "on course" looks like for a Mature-tier project: the strategic loop closed under its own discipline. The reckoning surfaced #68 on 2026-05-26; the planner ratified #68 on 2026-05-27; the implementation chain executed M-1..M-9 over 14 days; the merge that closes the roadmap is today's `9cff5b0`. Project → reckoning → plan → implementation → ship → next reckoning. The cycle ran exactly once at full strength, and it worked.

### What to Celebrate

- **The reckoning loop closed.** The prior reckoning named #68 as "the most important unmade decision in the project." Today (14 days later), #68 is CLOSED on GitHub, the dossier-puzzle is shipped through M-1..M-9, and the v0.4.x dossier surface is open. The reckoning function does what it claims to do.
- **Two parallel roadmaps closed in 24 hours.** v0.3.x dossier (M-8 `16acaa3`) + v2 character (C-4 `9a6a550`), with M-9 (`9cff5b0`) opening v0.4.x in the same calendar day. Roadmap-closure discipline at this cadence is extraordinary for a solo-developer + Claude Code workflow.
- **Sacred Practice 12 single-authority is now visible at deletion time.** M-8's classic-shim removal executes the deprecation runway DEC-68-DOSSIER-REFRAME-008 scheduled at M-7. The discipline is now operational at both creation *and* retirement of authorities. That is mature single-authority hygiene.
- **F63 sentinel-row pattern proved generalizable.** M-4 (`dossier/state.py`), M-5 (Predictions Log persistence in same row family), M-9 (no new storage tier needed for the library because it's filesystem-native). Three slices reused F63 without inventing a parallel storage authority. The pattern has earned its place.
- **The character system became semantically coupled to the analytic substrate.** Columbo (C-4) carries `context_hooks` referencing `DossierSlotName` + `SlotStatus` enum values. Voice now responds to analytic state. The Original Intent's "different modes" + "teaching moments" implications cross-fertilized for the first time.
- **The Original Intent's crowdsourcing axis is architecturally tractable.** M-9 ships the file format, the consent gate, the local library, and the comparison engine. The federation step is now a *user-product* decision, not an *architectural-reachability* decision.
- **AP #74 orphan-prevention pattern.** The plan-drift fix on 2026-05-29 produced a reusable pattern (implementer commits include planner amendments in the same commit). It held through M-3 → M-9 and C-3 → C-4. Per-slice documentation lag is solved.

### What to Confront

1. **The Active Phase Pointer is stale within hours of landing.** M-9 merged at `9cff5b0` today. The pointer line at MASTER_PLAN.md:3199 still says "implementer landed @ `7cc801b`; reviewer + guardian pending." This is the recurring closeout-amend lag from the prior reckoning, surfacing at the *fastest cadence ever*. The AP #74 pattern handles per-slice phase sections atomically, but the cross-cutting pointer line is not in any slice's scope. **It needs an owner.**

2. **`DECISIONS.md` is now ~6 weeks stale and issue #72 has been open for 13 days.** The prior reckoning flagged this at 28 days stale; it's grown to 42 days. Issue #72 was filed (good — accountability) but unscheduled (bad — accountability without action). The registry mismatch between code (287 unique DEC-IDs) and registry (~48 rows) is now a 6× gap. A Future Implementer asking "what decisions own this surface?" *will* be misled.

3. **15 worktrees on disk, 14 of them on landed work.** The prior reckoning's Confront item 5 ("five concurrent worktrees with three already-landed") was Confront-then-ignored: 10 additional uncleaned worktrees joined them in 14 days. The `Branch and Worktree Cleanup` discipline in CLAUDE.md is being skipped systematically. This is not a product problem yet, but it is a *trust-in-discipline* problem — when CLAUDE.md says "after a successful push/landing and an idle, clean worktree: switch the checkout back... delete the local task branch... remove the task worktree" and 14 worktrees sit uncleaned, the rule is informally being treated as advisory.

4. **The MCP migration epic (#65) has been latent for 17 days.** Issues #65, #66, #67 (MCP/Honeylabs/go-roast integrations) remain OPEN with zero in-plan presence. The prior reckoning flagged #65 as a "candidate scope-direction question." The dossier roadmap (#68) was *also* a candidate in May; #68 got planner-absorbed and built. #65 did not. The asymmetry is real — and #65 is the architectural question (vendor-neutral MCP modules) that would let *other* LLM agents consume AP's tool surface, which directly serves the Original Intent's "crowdsource pursuit" axis the project just made architecturally tractable. **Whether #65 is the next roadmap, deferred indefinitely, or formally retired is itself an unmade decision.**

5. **Cross-workspace authorities are accumulating without a registry.** `~/.ap/config.toml` + `~/.ap/dossier_novelty.sqlite` (M-8) + `~/.ap/dossier_library/` (M-9). Three persistent cross-workspace state authorities now exist. A fourth without a registry-of-authorities would be drift. Today is the day to file an issue capturing the registry, not after the fourth.

6. **No release-discipline decision since v0.1.0 stable.** The project has shipped a v2 product reframe (dossier roadmap), a v2 character roadmap, four major persona profiles, an active falsification engine, a STIX 2.1 dossier wire format, a cross-workspace novelty system, and an opt-in actor library — *all on v0.1.0 lineage*. The `pyproject.toml` version has not moved. There is no v0.2.0 / v0.3.0 / v0.4.0 / v1.x cut. A user running `pip install adversary-pursuit` today gets v0.1.0 and never sees the dossier-puzzle. **The product has evolved; the released artifact has not.** This is not a fault — pre-1.0 projects routinely defer version bumps — but it is now a decision that needs naming.

7. **Issue #58 ("drain runtime hygiene backlog before v2 planning") was overridden by practice, not by decision.** Filed 2026-05-18 as a meta-issue saying "do this before v2 planning." v2 planning (and execution) happened. #58's spirit was overridden, but #58 itself remains OPEN. The principle that filed it should either be acted on or formally retired. Today (post-roadmap-closure) is a natural moment to ask: is #58 still operative, or did the dossier roadmap supersede it?

### What to Do Next

1. **(Small fix, ~5 min)** Amend MASTER_PLAN.md Active Phase Pointer to reflect M-9 merge `9cff5b0` and re-point to the *next* canonical work — which is itself a decision (see #4 below). The pointer line is mechanically broken at line 3199 right now.

2. **(Small fix, ~30 min + investigation)** Move issue #72 (`stop.sh DECISIONS.md regeneration silently broken`) from OPEN-unscheduled to OWNED-scheduled. Pick one of: (a) repair the hook and regenerate `DECISIONS.md` to current (287 DEC-IDs), (b) declare manual regeneration as the model and run it now, or (c) retire `DECISIONS.md` as a documented format and replace with a registry generated on-demand from per-phase DEC tables. The right answer is *some* answer; what cannot stand is the 6-week silent stale state.

3. **(Small fix, ~15 min)** Worktree triage. The 14 uncleaned landed worktrees on disk are doing nothing but consuming disk and adding noise to `git worktree list`. Per CLAUDE.md `Branch and Worktree Cleanup` discipline: for each merged feature branch (F59/F60/F61/F62/F64/M-1..M-8/C-1..C-3 worktrees), verify clean, then `git worktree remove <path>` + `git branch -D <branch>`. M-9 itself can stay until guardian-final-cleanup; the other 14 are stale. Single batch operation.

4. **(Big decision — invoke `/decide` or `/reckoning operationalize`)** Choose the next roadmap. The dossier and character roadmaps are closed. The candidates:
   - **(a) M-10+ dossier-axis follow-ons** — PII redaction layer, ingest-priors writer (the deferred DEC-M9-IMPORT-READONLY-001 follow-on), multi-actor bundles, federation registry.
   - **(b) MCP migration epic (#65)** — vendor-neutral MCP modules so non-`ap chat` LLMs can consume AP's tool surface; would also pull in #66 (Honeylabs) and #67 (go-roast) as concrete first integrations.
   - **(c) Runtime hygiene backlog drain** (#49..#55, #58, #70, #71, #75, #76) — the infrastructure-of-tooling debt that has been deferred since v1.
   - **(d) Release-discipline cut** — version-bump v0.1.0 → v0.4.0 (or v0.2.0 / v0.3.0 with explicit retroactive mapping) so the world can install what the project has built.
   - **(e) Crowdsourcing federation** — promote the v1 Non-Goal "Federation" to v2 Goal; build the registry/share-layer on top of M-9's local library.
   These are not mutually exclusive long-term but they are mutually exclusive in *what the planner optimizes for next*. The reckoning loop just demonstrated it can close one of these per ~14 days at this cadence. The user is the only one who can resolve direction.

5. **(Medium docs slice, ~1 hour)** File and stage a cross-workspace authority registry. Add `~/.ap/AUTHORITIES.md` (or a section in MASTER_PLAN.md) enumerating `~/.ap/config.toml` (since v1), `~/.ap/debug.log` (Phase 10), `~/.ap/dossier_novelty.sqlite` (M-8), `~/.ap/dossier_library/<actor_identifier>.json` (M-9). For each: owner module, schema version, opt-in/required, retention policy, removal procedure. Three authorities can fit in heads; four needs a registry; M-9's library is the fourth (counting `debug.log`). **The right moment to add this is before the fifth.**

6. **(Reflection — for the planner, not an implementer)** Codify the Active Phase Pointer as a Guardian-landing concern, not a planner-stage concern. The pointer line at MASTER_PLAN.md:3199 needs to be updated atomically with `git merge` of any roadmap-closing or roadmap-opening slice. The AP #74 orphan-prevention pattern handles per-slice phase sections but does not handle the cross-cutting pointer. A small Guardian-landing hook (or an explicit step in the Guardian-land Evaluation Contract) would close this.

7. **(Optional, ~30 min)** Close or comment-update the stale issues #26, #27, #28, #33, #58. #26 (cmd2 → chat console) is effectively done by ADR-010; close with cross-reference. #27 (HEF/Analysis/Persona) is partially done by C-1..C-4; either close-with-cross-ref or scope a follow-on. #28 (RAG knowledge base) has zero presence and no successor — either schedule or formally retire. #33 (v2 PyPI docs) needs re-scoping for whether v2 returns to PyPI. #58 (drain runtime hygiene before v2) was overridden by practice — either close or re-affirm. All five are housekeeping that the dossier-roadmap sprint understandably bypassed.

---

## Reckoning Delta (vs. 2026-05-26 reckoning)

The prior reckoning was the project's first full seven-dimensional analysis. The comparison below tracks what moved between then and now.

| Dimension | 2026-05-26 | 2026-06-09 | Direction |
|-----------|-----------|-----------|-----------|
| Verdict | drifting constructively | **on course** | improved (the named drift was about unmade decisions; those decisions are now made and built) |
| Maturity tier | Mature (9 closed, 168 in-code DECs) | **Mature (sustained; 287 in-code DECs across 57 files; 18 new phase sections)** | deepened |
| Intent alignment | Strong | **Strong (strengthened)** | three Principles have stronger new evidence in this window |
| Decision coherence | Strong (Sacred Practice 12 reflexive) | **Strong (visible at deletion time)** | doctrine has matured from creation-time hygiene to creation+retirement-time hygiene |
| Module count | 15 | 15 (no new modules in this window) | unchanged — the focus shifted to dossier substrate |
| LLM tool count | 21 (at v1 ship) | **30** (post-M-9) — was 28 (post-M-8), grew 28→30 via M-9 export/compare | +9 from v1 |
| Dossier package | did not exist | **11 files** (`__init__.py` + comparison + export + import_ + novelty + panel + predictions + scoring + slot_inference + slots + state) | new product surface |
| LLM personas with `llm_profile` | 0 | **6** (full_troll, ninja, sun_tzu, bruce_lee, bureaucrat, columbo) | character system v2 closed |
| Cross-workspace authorities | 1 (`~/.ap/config.toml`) | **3** (`config.toml` + `dossier_novelty.sqlite` + `dossier_library/`) | +2 in 14 days; registry needed |
| Worktrees on disk | 5 | **15** | tripled; cleanup discipline collapsed |
| `DECISIONS.md` staleness | 28 days | **~42 days** | grew; issue #72 filed but unaddressed |

### Resolved Findings (Prior Reckoning → Now)

- **Confront #1 (Phases 11/12/13/14 status text stale → closeout-amend bypass)** → **partially resolved**. The AP #74 orphan-prevention pattern handles per-slice closeout amendments atomically with merge. **However**, the cross-cutting Active Phase Pointer still drifts. Mechanism solved; surface area incomplete.
- **Confront #2 (`DECISIONS.md` silently stale)** → **flagged, not resolved**. Issue #72 was filed (accountability captured). The registry remains stale and now grew from 28 → 42 days behind.
- **Confront #3 (Issue #68 the most important unmade decision)** → **FULLY RESOLVED.** #68 was ratified Phase 16, decomposed into M-1..M-9, executed in 14 days, closed on GitHub today. The reckoning loop closed.
- **Confront #4 (Crowdsourcing axis 100% latent)** → **architecturally addressed.** M-9 ships the file format, local opt-in library, and comparison engine. The federation step remains a deliberate v1 Non-Goal but is now a *product decision*, not an *architectural-reachability* problem.
- **Confront #5 (5 concurrent worktrees, 3 landed-but-uncleaned)** → **WORSENED.** 15 worktrees, 14 on landed work. The named discipline was not followed.
- **Confront #6 (Phase 7 "organic" bypass precedent → needs Guardian gate for plan amendments)** → **mechanism mostly solved by AP #74**. Per-slice plan amendments now ride in implementer commits. Active Phase Pointer remains the gap.

### New Findings (Not Present in Prior Reckoning)

- **The Original Intent's crowdsourcing axis is architecturally tractable.** This is a major new positive finding — it was the latent Future Self promise of the prior reckoning, and is now an addressable v2 question rather than a "what would we even build?" question.
- **Cross-workspace authority sprawl** (3 in `~/.ap/`, no registry). New class of risk — was 1 authority at the prior reckoning; tripled in 14 days. Registry needed before fourth.
- **Worktree cleanup discipline has effectively collapsed.** Was a warning; is now demonstrated lapse.
- **The character system is now semantically coupled to the dossier substrate.** Columbo's `context_hooks` reference `DossierSlotName` enum values. This is a new design-time coupling between two formerly orthogonal authorities; it was made on purpose and bound by DEC-C4-COLUMBO-103 (string-literal convention, no schema refinement), but it is a new architectural fact.
- **LLM is now in the gamification path** (M-7 `AgentRunner.narrate`). New trust surface — LLM-generated celebration text under typed budget. Anti-pattern doors (token cap, per-hunt budget, silent runtime fallback, loud-fail in tests) are closed by construction, but the surface itself is new.
- **The v0.1.0 release is conceptually stale.** What the project has shipped on `main` is a v2-product-reframed system; what `pip install adversary-pursuit` returns is v0.1.0 from May. Release-discipline decision pending.
- **MCP migration epic (#65) remains the next big strategic question.** Was a candidate at the prior reckoning alongside #68. #68 was absorbed; #65 was not. The asymmetry is now sharper because dossier substrate exists and could be MCP-exposed.

### Persistent Findings (Flagged Then, Still True)

- **The interface-pivot worked because modules + gamification engines were architecturally separated from the console.** Still true. The dossier package and character profiles add two more orthogonal authorities; the separation has held.
- **The principles (1–5) are intact.** Still true. New evidence in this window strengthens 1, 3, and 5.
- **Future Self promises pattern.** The crowdsourcing axis was the named example at the prior reckoning. It got addressed. The next named examples (MCP migration #65, runtime hygiene backlog #49..#58, PyPI v2 release #33) remain latent. The pattern needs a discipline: a "Latent Promises" section in MASTER_PLAN.md with explicit retire-or-schedule deadlines.
