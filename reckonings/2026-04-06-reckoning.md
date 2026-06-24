# Project Reckoning: Adversary Pursuit

**Date:** 2026-04-06
**Source:** /Users/jarocki/src/ap/MASTER_PLAN.md
**Project age:** ~3.5 years from initial commit (2022-11-24); 1 day from active planning (2026-04-05)
**Maturity tier:** Foundation
**Initiatives:** 5 phases planned (24 issues), 0 completed, 0 parked
**Decisions:** 9 Architecture Decision Records (ADR-001 through ADR-009)

---

## I. The Core

Adversary Pursuit is, at its irreducible essence, the conviction that threat intelligence work should be *fun* -- that the act of tracking adversary infrastructure, pivoting through indicators, and mapping campaigns is inherently game-like and should be treated as such. The project doesn't just add gamification to OSINT; it reframes the entire discipline as a pursuit game where every adversary mistake is a point scored and every discovery is a celebration.

The founding tension is between two worlds that rarely speak to each other: the structured, procedural rigor of CTI analysis and the dopamine-driven engagement loops of gaming and CTF competitions. Most CTI tools optimize for analytical correctness. Most gamification is bolted on as an afterthought. AP's bet is that building the game mechanics *into the core architecture* -- scoring that responds to module outputs, character modes that reshape the interface, challenges as intelligence requirements with verifiable flags -- will produce something neither world has achieved alone: analysts who stay engaged because the work itself rewards them.

The implicit philosophy embedded in the design is that CTI/OSINT is currently too fragmented and too joyless. The README's original vision ("Taking maximum advantage of every mistake, and celebrating with epic memes") carries an irreverence that the MASTER_PLAN.md has faithfully preserved. The project believes that playfulness and rigor are not opposites -- that a tool can be simultaneously serious in its analytical capabilities and absurd in its celebration of them (Bobby Hill mode, Drunken Master chaos pivoting). This is not a contradiction; it is the soul of the project.

## II. The Origin

The Original Intent, verbatim from MASTER_PLAN.md:

> Adversary Pursuit is a gamified framework for hunting, pivoting, and discovery of actor infrastructure, indicators, and TTPs. "Taking maximum advantage of every mistake, and celebrating with epic memes."
>
> Goals: Make it fun. Gamify. Different modes. Graph of pursuit progress. Teaching moments at dead ends. Meme/DALL-E generator for celebrations. Final report generation (interview-based). Standardize hunting and pursuit techniques, crowdsource pursuit, competition and ranking for career development.
>
> Interface should feel like a combination of Metasploit and CTFd. Move straight to v1 multi-platform Python. Reference CTI and OSINT awesome lists for data sources. Priority integrations: VirusTotal, Shodan, PassiveTotal, Censys, URLScan, HaveIBeenPwned, AbuseIPDB, AlienVault OTX, plus Maltego-style transforms and OSINT Tool aggregators.

This was written to distill and formalize ideas that first appeared in the README.md (committed 2022-11-24), which was rawer and more expansive -- including v0 Jupyter, v3 web app, v4 mobile, federation, cloud hosting, 3D character printing, and machine-assisted features. The Original Intent in the MASTER_PLAN.md represents a *conscious scoping* of that broader dream into a concrete v1 target: multi-platform Python CLI, skip Jupyter, focus on the core integration+gamification loop.

Key assumptions embedded in the Original Intent:
1. That a CLI interface (Metasploit-like) is the right medium for CTI analysts -- not a web GUI, not a notebook
2. That the module ecosystem (OSINT/CTI APIs) is the primary source of value, and gamification makes that value *sticky*
3. That Python is the natural language for this domain (analyst tooling, API integrations, security community)
4. That existing platforms (IntelOwl, SpiderFoot, TheHive) are reference architectures to learn from, not competitors to replicate

The scoping decision to skip v0 (Jupyter) and go straight to v1 (multi-platform Python) is notable. It trades prototyping speed for architectural commitment. This is a bet that the vision is clear enough to build properly from the start.

## III. The Journey

### Timeline

| Period | Event | Key Decisions | Outcome |
|--------|-------|---------------|---------|
| 2022-11-24 | Initial commit: LICENSE, README, .gitignore | Project created as a vision document | Dormant for 3+ years |
| 2022-11-24 | "Added overview" + "minor" commits | README with full vision | Raw idea captured |
| 2026-04-05 | Deep research (Gemini) conducted | Architecture stack validated | ADR-001 through ADR-009 |
| 2026-04-05 | MASTER_PLAN.md written | 5 phases, 24 issues, 9 ADRs | Full plan in place |
| 2026-04-06 | GitHub issues #1-#5 created | Phase 1 tracked | Ready for implementation |

### Decision Density

All 9 decisions were made on a single day (2026-04-05), derived from a deep research session. This is characteristic of a "big bang" planning event rather than iterative discovery. The decisions are well-rationalized and research-backed, but they have not been tested against implementation reality.

Decision density: 9 decisions / 1 day = concentrated. This is expected for a Foundation-tier project emerging from a research+planning phase. The quality risk is not the density itself, but that all decisions were made simultaneously without the feedback loop of building anything.

### Inflection Points

1. **The 3.5-year dormancy (2022-11-24 to 2026-04-05).** The project existed as a README -- a vision document -- for over three years before formal planning began. This is the single most significant fact about the project's history. It means the idea has had time to mature in the user's mind, but it also means there's pent-up ambition and scope that could overwhelm execution.

2. **The research-to-plan event (2026-04-05).** A deep research session (Gemini, with web corroboration) produced architectural validation, which was immediately translated into a comprehensive MASTER_PLAN.md. This was proactive -- choosing architecture based on evidence rather than personal preference -- and it produced defensible decisions. But it was also a single session with a single research provider (OpenAI failed, Perplexity was not configured).

### Plan vs. Reality

**Reality check:** There is no code. Zero Python files exist. The project consists of:
- README.md (original vision, 2022)
- MASTER_PLAN.md (comprehensive plan, 2026-04-05)
- Deep research artifacts (.claude/research/)
- 5 GitHub issues (Phase 1 only, created 2026-04-06)
- No src/ directory, no tests/, no pyproject.toml

The plan describes a detailed directory structure, dependency list, and module contracts, but none of it has been instantiated. The project is entirely aspirational at this point. This is appropriate for a Foundation-tier project that has just completed planning, but it means every architectural decision is untested.

GitHub issues exist only for Phase 1 (issues #1-#5). Phases 2-5 (issues #6-#24) are described in the plan but not yet tracked as issues. This is a gap -- the plan references issue numbers that don't exist yet.

## IV. Evolution Assessment

### Intent Alignment: Strong

The MASTER_PLAN.md faithfully translates the README's raw vision into an implementable architecture. Every major element from the original README appears in the plan:

- "Make it fun. Gamify." -> Phase 3: Gamification Engine (scoring, challenges, modes, badges)
- "Different modes" -> Issue #16: Character Modes (all 9 modes preserved from README)
- "Graph of pursuit progress" -> Issue #20: Graph State & Visualization
- "Meme/DALL-E generator" -> Issue #22: Celebration System
- "Final report generation (interview-based)" -> Issue #21: Report Generation (exact same questions)
- "Metasploit and CTFd" -> cmd2 REPL + CTFd scoring (ADR-001, ADR-008)
- All 8 priority API integrations -> Issues #6-#13

The plan consciously excludes elements from the README's broader vision that belong to later versions: Jupyter (v0), web application (v3), mobile (v4), federation, cloud hosting, 3D printing, machine-assisted features. This is disciplined scoping, not intent drift.

### Principle Adherence

The MASTER_PLAN.md does not have a formal "Principles" section, which is itself a finding. However, implicit principles can be extracted from the design:

| Implicit Principle | Honored in Plan? | Evidence |
|-------------------|-----------------|----------|
| Fun first | Yes | Gamification is Phase 3 (not an afterthought), character modes are core |
| Metasploit-like UX | Yes | cmd2 REPL, module namespace, use/set/run workflow (ADR-001) |
| Industry-standard data | Yes | STIX 2.1 as internal model (ADR-005), OpenCTI compatibility |
| Modular and extensible | Yes | Plugin system via entry points (ADR-003), Protocol contracts (ADR-004) |
| Open and free | Yes | LICENSE present from initial commit |

### Constructive Expansions

The MASTER_PLAN.md added architectural depth not present in the README:
- **Event bus for auto-pivoting** (Phase 4, ADR-007): The README mentioned "next pivot suggestion" but the plan formalized this as a SpiderFoot-pattern pub/sub event bus. This is a constructive expansion that enables cascading discovery.
- **STIX 2.1 data model** (ADR-005): The README didn't specify a data model. Adopting STIX 2.1 gives the project interoperability with the broader CTI ecosystem. Well-motivated.
- **Workspace isolation** (Issue #4): Metasploit's workspace concept, adapted for investigation isolation. Not in the README, but a natural extension of the CLI pattern.

### Scope Drift

None detected. The MASTER_PLAN.md is actually *more focused* than the README, having deliberately excluded v0/v3/v4 scope and aspirational features like federation and 3D printing.

### Non-Goal Violations

Not applicable. No non-goals have been formally declared in the plan (another finding -- the plan lacks explicit non-goals at the initiative level).

### Abandoned Threads

Several README ideas are not addressed in the MASTER_PLAN.md and are effectively deferred to future versions without being explicitly parked:
- Federation
- Cloud/VM hosting (Docker/Kubernetes)
- Playbooks (mentioned in README, not formalized as an issue)
- Machine-assisted features (feature identification, behavior summarization, TTP clustering)
- Character sheets, backstories, 3D rendering
- MS Paint and .stl graphics

These aren't abandoned -- they belong to v2+ -- but they aren't tracked anywhere. They exist only in the README.

## V. Decision Quality

### Coherence: Strong

The 9 ADRs form a coherent architectural stack where each decision reinforces the others:

- ADR-001 (cmd2) + ADR-002 (Rich) = the presentation layer
- ADR-003 (entry points) + ADR-004 (Protocol) = the plugin system
- ADR-005 (STIX 2.1) + ADR-006 (SQLite) + ADR-008 (SQLAlchemy) = the data layer
- ADR-007 (asyncio event bus) = the automation layer
- ADR-008 (parabolic decay) = the gamification model
- ADR-009 (httpx) = the networking layer

No decisions contradict each other. The async story is consistent: asyncio event bus (ADR-007) + httpx for async HTTP (ADR-009). The data story is consistent: STIX 2.1 objects (ADR-005) stored in SQLite (ADR-006) via SQLAlchemy (ADR-008 references ORM). The plugin story is consistent: entry points (ADR-003) load Protocol-conforming modules (ADR-004) that return STIX objects.

### Notable Decision Chains

The strongest chain is ADR-001 -> ADR-002 -> ADR-004: choosing cmd2 over Textual specifically because Textual lacks REPL support, then choosing Rich for rendering (which cmd2 natively supports), then choosing Protocol for module contracts (lightweight structural subtyping that works naturally with cmd2's command dispatch). Each decision narrows the next choice space.

### Decision Gaps

1. **No decision on testing strategy.** The Verification section mentions pytest, but there's no ADR on testing philosophy (unit vs. integration vs. e2e balance, mocking strategy for API-dependent modules, test data management).

2. **No decision on error handling.** API integrations will fail (rate limits, auth errors, network issues). How modules handle and report failures is an architectural concern that deserves an ADR.

3. **No decision on API key management security.** The config section mentions "Sensitive values stored with file permissions (0600)" but doesn't address keyring integration, encrypted storage, or credential rotation. For a tool that stores API keys to 8+ services, this deserves more thought.

4. **No decision on versioning/migration strategy.** SQLite schema will evolve. The plan mentions Alembic for migrations but doesn't formalize this as a decision.

5. **No decision on offline/degraded operation.** What happens when APIs are unreachable? Can the tool still function with cached data? This matters for a tool analysts might use in restricted environments.

### Traceability

No @decision annotations exist in code (no code exists). ADR-IDs in the plan use a simple `ADR-NNN` scheme without cross-references between them. Traceability is nascent -- appropriate for Foundation tier, but should be established as code is written.

## VI. Project Health

| Indicator | Rating | Evidence |
|-----------|--------|----------|
| Vitality | Active | MASTER_PLAN.md created 2026-04-05, issues filed 2026-04-06, deep research conducted -- project is actively being bootstrapped after 3.5 years dormant |
| Focus | Sharp | Single clear v1 target, conscious scoping from broader README vision, 5-phase linear implementation plan |
| Momentum | Starting | No code yet, but planning artifacts are comprehensive and GitHub tracking has begun. The critical question is whether coding begins promptly. |
| Coherence | Strong | 9 ADRs form an interlocking stack, plan faithfully translates original vision, research validates architecture choices |
| Sustainability | Unknown | No implementation data exists to assess pace. The plan's scope (24 issues across 5 phases) is ambitious for a solo project. Phase 2 alone has 8 API integrations. |

## VII. Trajectory

### Current Vector

The project is moving from vision to plan to implementation readiness. The last 24 hours represent a burst of planning energy: deep research, MASTER_PLAN.md creation, GitHub issue filing. The vector points toward Phase 1 implementation (scaffolding, console, plugins, data model, config).

### Projected Destination

If current patterns continue -- meaning the planning rigor translates to implementation discipline -- the project becomes a functional CLI with a Metasploit-like REPL, a handful of OSINT module integrations, and a basic scoring system within 2-3 months. If the planning phase extends further without code, the project risks becoming a beautifully documented idea that never ships.

The 3.5-year dormancy is the elephant in the room. The idea survived -- it was clearly compelling enough to return to -- but the gap between vision and execution is a pattern to be aware of. The MASTER_PLAN.md and deep research represent a serious commitment to breaking that pattern. Whether it succeeds depends entirely on whether Issue #1 (scaffolding) gets implemented this week.

### Intent-Trajectory Gap

**Small.** The plan is tightly aligned with the original intent. The gap is not between vision and plan -- it's between plan and reality. Zero lines of code exist. The trajectory is pointed in the right direction, but the journey hasn't started.

## VIII. The Reckoning

### Verdict: On course

The verdict is "on course" with a qualification: the course has been charted but not yet sailed. This project has done something right that many projects fail at -- it took a raw, expansive, 3.5-year-old vision and translated it through rigorous research into a coherent, implementable architecture. The MASTER_PLAN.md faithfully honors the README's original intent while making disciplined scoping decisions. The 9 ADRs form an interlocking stack where each decision reinforces the others. The plan is phase-gated, linearly ordered, and maps cleanly to GitHub issues.

But "on course" for a Foundation-tier project means the foundation is sound, not that the building stands. The plan is untested against implementation reality. Every architectural decision was made on paper, validated by research but not by code. The project's biggest risk is not architectural -- the research is solid -- but motivational and scoping. Can a solo developer sustain the energy to implement 24 issues across 5 phases? Will the first collision with implementation reality (API quirks, cmd2 limitations, STIX 2.1 complexity) cause a course correction or a stall?

The project has a clear soul. The founding tension -- rigorous CTI analysis meets gaming engagement -- is preserved from the 2022 README through the 2026 plan without dilution. The character modes, the scoring system, the celebration mechanics -- these aren't afterthoughts bolted onto an OSINT tool. They're co-equal architectural citizens alongside the module system and data model. That's rare, and it's the project's greatest strength.

### What to Celebrate

1. **The scoping discipline.** The README dreamed of federation, mobile apps, 3D character printing, and Kubernetes deployment. The MASTER_PLAN.md cut all of that cleanly and focused on a v1 CLI. This is the hardest thing in project planning -- saying no to your own ideas -- and it was done well.

2. **Research-backed architecture.** ADR-001 (cmd2 over Textual) wasn't a gut call -- it came from a deep research session that evaluated alternatives with specific rationale. ADR-005 (STIX 2.1) wasn't assumed -- it was validated against the CTI platform landscape. Every major decision has a "why not the alternative" rationale. Future Implementers will understand not just what was chosen, but why.

3. **Identity preservation across 3.5 years.** The original README's personality -- "Taking maximum advantage of every mistake, and celebrating with epic memes" -- survives intact in the plan. Bobby Hill mode, Drunken Master chaos pivoting, and "Full Troll" mode are still there. The project didn't lose its irreverence when it got serious about architecture.

4. **The module contract design.** `PursuitModule` as a `typing.Protocol` with `initialize()` and `hunt()` methods returning STIX 2.1 observables is elegant. It creates a clean boundary: modules are pure data producers, the console is the orchestrator, the gamification engine is a side-effect observer. This separation of concerns will pay dividends when third-party modules arrive.

### What to Confront

1. **The 3.5-year gap is the project's biggest risk factor, and the plan doesn't acknowledge it.** The idea was compelling enough to write down in 2022 but not compelling enough to build for 3 years. What changed? What will sustain momentum this time? The MASTER_PLAN.md reads as if the project was just created, but it wasn't -- it has a history of dormancy. A "Lessons Learned" or "Why Now" section would be honest and useful for future reference.

2. **The plan has no explicit non-goals or scope boundaries per phase.** Each phase describes what will be built but not what will be explicitly excluded. For a project whose greatest planning achievement was scope discipline (cutting v0/v3/v4), the plan should carry that discipline forward into each phase. What does Phase 1 explicitly NOT include? What happens when a tempting feature appears during module development (Phase 2)?

3. **Sustainability is unaddressed.** 24 issues across 5 phases for what appears to be a solo developer. Phase 2 alone requires implementing 8 API integrations, each with its own auth model, rate limits, response schemas, and error modes. The plan presents these as roughly equal effort items, but in reality, some (VirusTotal v3 with its complex analysis endpoints) are significantly more complex than others (AbuseIPDB with its simple REST API). There is no effort estimation or prioritization within phases.

4. **Issues #6-#24 don't exist in GitHub.** The plan references 24 issues, but only #1-#5 have been created. This means Phases 2-5 are planned in the MASTER_PLAN.md but not tracked where work actually gets done. If implementation begins, the plan and the issue tracker will immediately diverge. Issues should be created (even as stubs) for all phases, or the plan should note that only Phase 1 is tracked.

5. **No formal Principles section.** The project clearly has principles -- fun first, Metasploit-like UX, industry-standard data, modular extensibility -- but they're implicit, spread across the Context, Architecture Decisions, and README. Stating them explicitly would give future decisions a documented anchor. When the first hard architectural trade-off appears (performance vs. fun, simplicity vs. STIX compliance), explicit principles tell you which way to lean.

6. **The deep research had a single provider.** OpenAI failed, Perplexity wasn't configured. Gemini produced excellent results, corroborated by web search, but the architecture is validated by one AI research model rather than multi-model consensus. The research report honestly acknowledges this, which is good. But for the most consequential decisions (cmd2 as the CLI foundation, STIX 2.1 as the data model), a second deep research run with all providers working would strengthen confidence.

### What to Do Next

1. **Implement Issue #1 (Project Scaffolding) immediately.** The plan is complete enough to begin. The single most important thing for this project is to have code -- a pyproject.toml, a src/ directory, a passing test. Every day without code reinforces the 3.5-year dormancy pattern. Break the pattern now.

2. **Add explicit Principles and Non-Goals sections to MASTER_PLAN.md.** Extract the implicit principles into a formal section: "Fun is a first-class design constraint," "Metasploit UX patterns are the interaction model," "STIX 2.1 is the lingua franca," "Modules are pure data producers." Add phase-level non-goals, especially for Phase 1 (e.g., "Phase 1 does NOT include any API integrations, gamification scoring, or character modes").

3. **Create GitHub issues for all 24 planned items.** Even if Phases 2-5 are stub issues with just a title and reference back to MASTER_PLAN.md. This ensures the single source of truth for task tracking is GitHub, not just the plan document. Label them by phase for filtering.

4. **Add a "Why Now" section to the plan.** Acknowledge the 3.5-year gap and document what changed. This serves two purposes: it's honest history for Future Implementers, and it creates a motivational anchor the developer can return to when energy wanes.

5. **Prioritize within Phase 2.** Not all 8 API integrations are equal. AbuseIPDB and AlienVault OTX have free tiers and simple APIs -- implement those first. VirusTotal and PassiveTotal have complex auth and rate limiting -- save those for later. Add effort estimates or at least a priority ordering within the phase.
