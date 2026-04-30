# Project Reckoning: Adversary Pursuit — Interface Model Correction

**Date:** 2026-04-29
**Source:** /Users/jarocki/src/ap/MASTER_PLAN.md
**Trigger:** User correction to the prior planner's reconciliation — "the interface needs to be an agentic AI chat as I described and requested to be planned."
**Predecessor:** 2026-04-06-reckoning.md (Foundation tier; original 24-issue plan; cmd2 framed as primary UX)
**Maturity tier:** Active (5 of 6 phases landed; Phase 5 PyPI publish + Phase 6 agent parity remain)

---

## The Correction

The original 2026-04-05 plan declared the v1 user-facing interface to be a Metasploit-like cmd2 REPL (issue #2: "the heart of the application"). Phases 1-4 were built on that premise and shipped. In Phase 5, smolagents/litellm support arrived as #25 — initially framed (in this plan, in the README, and in the per-file `@decision` annotations) as an *optional* `[agent]` extra, "v2," or "Conversational CTI" alongside the "Classic CLI."

The prior planner's 2026-04-28 reconciliation correctly identified what code had landed but **mis-interpreted the user's intent**: it framed #25's v1 inclusion as an open scope decision with three options (a/b/c). The user has now clarified that the v1 vision is the agentic chat as the primary user-facing interface — not as an extra, not as v2, not as an experimental opt-in.

The cmd2 REPL still ships and is still useful, but its role has shifted: it is the power-user "manual transmission" surface, not the front door.

## What changed in the plan

- **Original Intent / Context / Solution / Interface Model.** Added a "Solution (v1, revised)" paragraph and a new "Interface Model (Revised 2026-04-29)" subsection naming the agent as primary. Both layers' user journeys are documented; the shared module catalog + workspace + scoring substrate is called out so it's clear *both* interfaces sit on top of the same Phase 1-4 work.
- **v1 Non-Goals.** The "Machine-assisted features" exclusion is preserved, but a narrow carve-out is added for LLM-driven tool selection over the AP module catalog. Auto-classification, TTP clustering, automated narrative reports, and AI-image celebrations remain out of scope.
- **Plan Status table.** Reframed and renumbered. The new Phase 4 in the user-facing ordering is "Agentic Chat Interface (#25 + W-AGENT-*) — in-progress (primary v1 interface)." A note clarifies that the per-phase Decision Log narrative below preserves the original Phase 1-5 numbering for traceability and appends the new agent work as "Phase 6" so legacy phase headings don't get renumbered.
- **Phase 1 Foundation Decision Log.** Added a reframing note that the cmd2 console (#2) is now supporting infrastructure under ADR-010, not the primary UX. The DEC-CONSOLE-* decisions remain accurate facts about what was built.
- **Retired sections.** The "v1 Scope Decisions Pending → SD-1 (smolagents a/b/c)" framing is gone, replaced by the new Phase 6 section. The `@decision` annotation gap discussion (formerly SD-2) is preserved verbatim as a sub-section under Phase 6 because it remains informational and unchanged.
- **New Phase 6 section.** Documents what landed (`agent/` package, 9 tools, scoring + workspace + 7 modules wired, 41 unit tests in `test_agent_tools.py`) and what is missing for v1-quality parity with the cmd2 console (celebrations, badges, hints, character-mode UI, autopivot, challenges, graph/export, report-generation, plus 3 missing modules: VirusTotal, Censys, PassiveTotal). Includes a Gamification ↔ Agent Interface mapping table and an honest "Reality Check" subsection.
- **Next Work Items.** Replaced the retired W-SCOPE-25 with a 9-item W-AGENT-* lineup, each scoped to a single Guardian-bound work item with a target file list. Recommended next item: **W-AGENT-MODULES-VT-CENSYS-PT** (smallest, lowest-risk, restores catalog parity), then **W-AGENT-CELEBRATIONS** (highest signal-to-effort gamification gap). W-V1-PYPI-VERIFY survives unchanged. W-COVERAGE-METRIC is deferred as cosmetic.
- **ADR-010 added.** "Agentic AI chat is the v1 primary user-facing interface; cmd2 REPL is a supporting power-user surface." Rationale captures the architectural cheapness of the shift (modules + gamification primitives are already cleanly separated from the console).
- **Implementation Order + MLP.** Added Phase 6 to the implementation order. Revised MLP from "console + 3 modules + scoring" to "agentic chat + 3 modules + scoring + at least one visible gamification signal in the chat path." Status against the revised MLP: 2 of 3 components are met; **W-AGENT-CELEBRATIONS sits on the MLP critical path**.

## Honest gap report

The agent's dispatch + scoring + workspace core is solid (9 tools, 41 unit tests, clean tool/runner separation). The gap to v1 is gamification parity:

| Touchpoint            | cmd2  | agent  | Work item                 |
|-----------------------|-------|--------|---------------------------|
| Scoring               | wired | wired  | (none)                    |
| Workspace             | wired | wired  | (none)                    |
| Character Modes       | wired | partial| W-AGENT-MODES             |
| Celebrations          | wired | none   | W-AGENT-CELEBRATIONS      |
| Badges                | wired | none   | W-AGENT-BADGES            |
| Hints                 | wired | none   | W-AGENT-HINTS             |
| Auto-Pivot/Event Bus  | wired | none   | W-AGENT-AUTOPIVOT         |
| Challenges            | wired | none   | W-AGENT-CHALLENGES        |
| Graph + Export        | wired | none   | W-AGENT-GRAPH-EXPORT      |
| Report Generation     | wired | none   | W-AGENT-REPORT            |
| Module coverage       | 10    | 7      | W-AGENT-MODULES-VT-CENSYS-PT |

This is the kind of gap that's tractable because the underlying engines (`CelebrationEngine`, `BadgeManager`, `HintProvider`, `ModeManager`, event bus) were always architecturally separated from the cmd2 console. The console *uses* them; it doesn't *own* them. Wiring them into the agent path is plumbing, not redesign.

## What did not change

- The principles (1-5) are intact. "Fun is a first-class design constraint" is unchanged and is exactly what motivates the W-AGENT-CELEBRATIONS / W-AGENT-BADGES / W-AGENT-HINTS work.
- The STIX 2.1 internal data model (ADR-005) is unchanged.
- The module Protocol contract (ADR-004, DEC-MODULE-001/002) is unchanged. The agent consumes modules through the same `PursuitModule.hunt()` interface the cmd2 console uses; that's what made this correction architecturally cheap.
- The CI/CD pipeline (#24) and the PyPI publish path are unchanged. W-V1-PYPI-VERIFY remains a real v1 boundary.
- Phase 1-4 Decision Logs are preserved verbatim. They describe what was built and why. None of them are wrong under the corrected interface model — they're just describing a layer that is no longer the front door.

## Recommended next step

Dispatch **W-AGENT-MODULES-VT-CENSYS-PT** as the first guardian-bound implementer slice. It's small (3 tool definitions + 3 `_MODULE_MAP` entries + tests), has no blockers, restores catalog parity (10 modules in cmd2 vs. currently 7 in agent), and de-risks the bigger gamification work that will share the same per-tool-call hook point in `run_module`.
