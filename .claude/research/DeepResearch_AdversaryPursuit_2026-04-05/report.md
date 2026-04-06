# Deep Research Report: Adversary Pursuit -- Gamified CTI/OSINT Hunting Framework

## Provider Status
| Provider | Status | Time | Notes |
|----------|--------|------|-------|
| OpenAI | FAILED | 0.5s | HTTPError: HTTP 400: Bad Request |
| Perplexity | SKIPPED | -- | No API key configured |
| Gemini | OK | 215s | deep-research-pro-preview-12-2025 |

**WARNING:** OpenAI (o3-deep-research) failed with HTTP 400. Perplexity was not configured. Only Gemini returned a report. WebSearch was used to supplement and cross-validate Gemini's findings across all 5 research areas.

## Executive Summary

Gemini's deep research produced a comprehensive 35,000-character architectural blueprint for "Adversary Pursuit," covering all five requested research areas in substantial depth. The core recommendation is to build the CLI on **cmd2 + Rich** (not Textual) for a Metasploit-like interactive console, use **importlib.metadata entry points + typing.Protocol** for the plugin system, adopt **STIX 2.1** as the internal data model (following OpenCTI's pattern), implement **CTFd-style parabolic decay scoring** for gamification, and draw architectural patterns from **IntelOwl** (analyzer microservices), **SpiderFoot** (pub/sub OSINT automation), and **TheHive** (case management). WebSearch cross-validation confirmed these recommendations align with current (2025-2026) best practices and community consensus.

Because only one deep research provider succeeded, confidence levels are assessed as "single-provider + web-corroborated" rather than multi-model consensus. However, the Gemini report is exceptionally detailed with 42 cited sources and covers all requested topics comprehensively.

---

## Individual Model Reports

### Gemini (deep-research-pro) -- 215s

Gemini produced a structured architectural blueprint organized into 7 major sections with 34 subsections. Key findings:

**CLI Framework (Section 1):** Evaluated cmd, cmd2, Textual, Rich, and prompt_toolkit. Concluded that cmd2 is the optimal foundation for a Metasploit-like REPL, noting it provides "batteries-included" tab completion, history, aliases, macros, and built-in scripting. Textual was explicitly rejected for REPL use cases -- community consensus indicates it lacks robust standard terminal I/O support. Rich serves as the rendering engine via cmd2's native Rich Console base classes (`Cmd2BaseConsole`, `Cmd2GeneralConsole`).

**Metasploit Architecture (Section 2):** Proposed a module namespace hierarchy (`modules/osint/`, `modules/cti/`, `modules/pivoting/`) mirroring Metasploit's exploit/payload/auxiliary structure. Commands replicate msfconsole syntax: `use`, `search`, `show options`, `set`, `run`. Workspaces use SQLite/PostgreSQL via ORM to isolate investigation data. Sessions represent persistent connections to intelligence streams (MISP WebSocket, SpiderFoot tasks).

**Plugin System (Section 3):** Strongly advocated against `pkgutil` directory scanning as an "anti-pattern." Recommended `importlib.metadata` entry points declared in `pyproject.toml` under the `adversary_pursuit.modules` group. Plugin contracts defined via `typing.Protocol` (structural subtyping) with a `PursuitModule` protocol requiring `initialize()` and `hunt()` methods. This ensures side-effect-free loading and strict isolation of import errors.

**CTFd Gamification (Section 4):** Detailed implementation of dynamic scoring using CTFd's parabolic decay formula: `value = ((minimum - initial) / decay^2) * solve_count^2 + initial`. Challenges are contextualized as intelligence requirements (find an APT IP, execute a multi-step OSINT pivot). Includes a hint economy where unlocking hints costs points, with balance protection against negative scores. Badges awarded for behavioral milestones.

**TI Platform Integration (Section 5):** Drew lessons from five platforms -- OpenCTI (STIX 2.1 graph-based data model), TheHive (case management with observables and timelines), IntelOwl (analyzer/connector/playbook architecture), SpiderFoot (publisher/subscriber event bus for cascading OSINT), and Maltego (transform-based pivoting stored as graph relationships).

**OSINT API Sources (Section 6):** Cataloged APIs across four categories: vulnerability/attack surface (Shodan, Censys, VulnCheck), IP/domain/DNS (VirusTotal, AbuseIPDB, PassiveTotal/RiskIQ, Spamhaus), data breach/social (HIBP, Sherlock), and threat actor/IOC aggregators (AlienVault OTX, Ransomwatch).

**Implementation Plan (Section 7):** Provided a complete directory structure using `src/` layout with `pyproject.toml`, organized into `core/`, `gamification/`, `models/`, and `base_plugins/` packages. Included working code examples for the APConsole class, plugin discovery, and gamification event hooks.

### OpenAI (o3-deep-research) -- FAILED
OpenAI returned HTTP 400 (Bad Request) after 0.5 seconds. The query may have exceeded length limits or hit a model configuration issue. No report was produced.

### WebSearch Supplementation

To compensate for the missing OpenAI perspective, web searches were conducted across all 5 research areas. Key supplementary findings:

- **cmd2 validation:** PyPI confirms cmd2 works with Python 3.10+ on Windows/macOS/Linux, uses prompt_toolkit internally, and is specifically designed for interactive CLI applications. Click (38.7% market share) and Typer dominate for non-interactive CLIs but are not suitable for stateful REPL consoles.
- **CTFd validation:** GitHub and CTFd docs confirm the dynamic scoring architecture with parabolic decay, challenge type plugins via Python + Nunjucks, and unlockable hint systems as described by Gemini.
- **Plugin systems:** Python Packaging User Guide (2025-2026 editions) confirms `importlib.metadata.entry_points()` as the recommended approach. A January 2026 article specifically covers entry-point-based plugin loading for Python 3.10+.
- **TI platforms:** Cosive's 2025 MISP vs OpenCTI comparison confirms MISP uses its own data model with STIX/TAXII export, while OpenCTI uses STIX 2.1 natively. Wiz's 2026 article corroborates the platform landscape. IntelOwl documentation confirms the analyzer/connector/playbook architecture.
- **OSINT APIs:** Multiple curated lists (BrewedIntel, awesome-osint) confirm the API landscape. Mihari (github.com/ninoseki/mihari) is a notable existing OSINT query aggregator integrating Censys, Shodan, URLScan, and VirusTotal.

---

## Comparative Assessment

Since only Gemini succeeded, all findings are tagged `[unique-gemini]` with supplementary `[web-corroborated]` tags where WebSearch confirmed the claims.

### Points of Agreement (Web-Corroborated)

These Gemini findings were independently confirmed by web searches:

1. **`[unique-gemini]` `[web-corroborated]` cmd2 is the right foundation for a Metasploit-like Python CLI.** Both Gemini and web sources agree that cmd2 provides the stateful REPL experience needed, with native tab completion, history, and prompt_toolkit integration. Textual is not suitable for this use case.

2. **`[unique-gemini]` `[web-corroborated]` importlib.metadata entry points are the modern standard for Python plugin discovery.** Both the Python Packaging User Guide and recent (Jan 2026) articles confirm this approach over directory scanning with pkgutil.

3. **`[unique-gemini]` `[web-corroborated]` CTFd's parabolic decay scoring is well-documented and implementable.** CTFd's GitHub and documentation confirm the exact formula and three-variable model (initial, decay, minimum) described by Gemini.

4. **`[unique-gemini]` `[web-corroborated]` STIX 2.1 via OpenCTI patterns is the right data model.** Cosive's 2025 comparison and OpenCTI's own documentation confirm that STIX 2.1 provides the graph-based CTI model needed, with SDOs, SCOs, and SROs.

5. **`[unique-gemini]` `[web-corroborated]` IntelOwl's analyzer/connector/playbook architecture is the reference model for modular API integration.** IntelOwl documentation confirms the three-tier approach (analyzers query external APIs, connectors export to platforms, playbooks chain operations).

6. **`[unique-gemini]` `[web-corroborated]` SpiderFoot's pub/sub event bus enables cascading OSINT automation.** SpiderFoot's GitHub (200+ modules) confirms the publisher/subscriber model where discovered artifacts trigger subscribed modules.

### Unique Insights (Gemini Only, Not Web-Validated)

These findings come solely from Gemini and lack independent confirmation:

1. **`[unique-gemini]` cmd2 supports Rich Consoles natively via Cmd2BaseConsole.** Gemini cites specific class names (`Cmd2BaseConsole`, `Cmd2GeneralConsole`, `Cmd2ExceptionConsole`) for Rich integration. This appears to reference newer cmd2 API features.

2. **`[unique-gemini]` Sessions as persistent intelligence stream connections.** The recontextualization of Metasploit's "session" concept from exploitation shells to long-running OSINT data feeds (MISP WebSocket, SpiderFoot correlations) is a novel architectural proposal.

3. **`[unique-gemini]` Gamification event hook architecture.** The pattern of intercepting module results before datastore persistence to check against active challenges is a specific design proposal not found in existing frameworks.

4. **`[unique-gemini]` typing.Protocol for plugin contracts over ABCs.** While structural subtyping is well-known, the specific recommendation to use Protocol over Abstract Base Classes for plugin contracts in this context is a Gemini-specific design choice.

5. **`[unique-gemini]` Graph State memory for CLI-based pivoting.** Storing Maltego-style transform results as SROs in memory and rendering text-based relationship trees is a creative adaptation of visual link analysis to a terminal interface.

### Confidence Assessment

| Finding | Gemini | WebSearch | Confidence |
|---------|--------|-----------|------------|
| cmd2 + Rich as CLI stack | Detailed | Confirmed | High |
| importlib.metadata entry points for plugins | Detailed | Confirmed | High |
| typing.Protocol for plugin contracts | Detailed | Partially confirmed | Medium-High |
| CTFd parabolic decay scoring | Detailed | Confirmed | High |
| STIX 2.1 as internal data model | Detailed | Confirmed | High |
| IntelOwl analyzer/connector/playbook pattern | Detailed | Confirmed | High |
| SpiderFoot pub/sub for auto-pivoting | Detailed | Confirmed | High |
| TheHive case management pattern | Detailed | Confirmed | High |
| Metasploit namespace/workspace/session mapping | Detailed | Partially confirmed | Medium-High |
| Specific code examples (APConsole, etc.) | Detailed | Not validated | Medium |
| Gamification event hook design | Detailed | Not validated | Medium |
| Hint economy with balance protection | Detailed | Confirmed via CTFd docs | High |

### Source Quality Assessment

| Provider | Citations | Report Length | Depth |
|----------|-----------|-------------|-------|
| Gemini | 42 sources (via Vertex AI redirects) | ~35,800 chars (~5,500 words) | Exceptionally deep: covers all 5 areas with code examples, formulas, and directory structures |
| OpenAI | 0 (failed) | 0 | N/A |
| WebSearch | 30+ sources across 4 queries | Supplementary | Broad corroboration across all topic areas |

**Note on Gemini citations:** All 42 citations use Google Vertex AI Search redirect URLs rather than direct source URLs. Of these, 37 redirects returned errors during validation (tokens expired), and 5 were reachable. The underlying sources are identifiable from domain hints in the report body (securedebug.com, imperva.com, readthedocs.io, plainenglish.io, github.com, ctfd.io, opencti.io, etc.).

---

## Key Architectural Recommendations Summary

Based on Gemini's research and web corroboration, the recommended architecture for Adversary Pursuit v1:

### Tech Stack
| Component | Technology | Rationale |
|-----------|-----------|-----------|
| CLI Core | cmd2 | Stateful REPL, tab completion, scripting, prompt_toolkit |
| Rendering | Rich | Tables, syntax highlighting, progress bars, panels |
| Plugin Discovery | importlib.metadata entry_points | Modern, explicit, side-effect free |
| Plugin Contracts | typing.Protocol | Lightweight structural subtyping |
| Data Model | STIX 2.1 (SDO/SCO/SRO) | Industry standard, OpenCTI compatible |
| Storage | SQLite (v1) / PostgreSQL (future) | Workspace isolation per investigation |
| Scoring | Parabolic decay (CTFd model) | Self-balancing difficulty valuation |
| Async OSINT | asyncio queues (event bus) | SpiderFoot-style cascading pivots |
| Package Format | pyproject.toml, src/ layout | Modern Python packaging standards |

### Module Hierarchy
```
modules/
  osint/     -- Public API queries (Shodan, WHOIS, DNS)
  cti/       -- TI platform queries (VirusTotal, MISP, OpenCTI)
  pivoting/  -- Multi-step transforms (Maltego-style)
```

### Prioritized OSINT API Integrations (v1)
1. **VirusTotal** -- File/URL/IP reputation (most comprehensive single source)
2. **Shodan** -- Internet-connected device intelligence
3. **AbuseIPDB** -- Community-driven IP reputation
4. **URLScan.io** -- URL analysis and screenshot capture
5. **HaveIBeenPwned** -- Data breach checking
6. **AlienVault OTX** -- Community threat intelligence feeds
7. **Censys** -- Certificate and host scanning
8. **PassiveTotal/RiskIQ** -- Passive DNS and WHOIS history

---

## References

### Verified Web Sources (from supplementary WebSearch)
[1] cmd2 -- PyPI -- https://pypi.org/project/cmd2/
[2] 10+ Best Python CLI Libraries (Jan 2026) -- https://medium.com/@wilson79/10-best-python-cli-libraries-for-developers-picking-the-right-one-for-your-project-cefb0bd41df1
[3] CTFd GitHub Repository -- https://github.com/CTFd/CTFd
[4] CTFd Dynamic Value Challenges -- https://github.com/CTFd/DynamicValueChallenge
[5] CTFd Challenge Type Plugins Documentation -- https://docs.ctfd.io/docs/plugins/challenge-types/
[6] CTFd Challenge Levels Documentation -- https://docs.ctfd.io/events/challenge-levels/
[7] Python Packaging: Creating and Discovering Plugins -- https://packaging.python.org/en/latest/guides/creating-and-discovering-plugins/
[8] How to Build Plugin Systems in Python (Jan 2026) -- https://oneuptime.com/blog/post/2026-01-30-python-plugin-systems/view
[9] Plugin Architecture in Python (DEV Community) -- https://dev.to/charlesw001/plugin-architecture-in-python-jla
[10] Understanding Plugin Architecture for Python Packages (PyCon India 2025) -- https://cfp.in.pycon.org/2025/talk/U3TRT9/
[11] MISP vs. OpenCTI: Updated 2025 Guide (Cosive) -- https://www.cosive.com/misp-vs-opencti
[12] OpenCTI GitHub Repository -- https://github.com/OpenCTI-Platform/opencti
[13] IntelOwl Project Documentation -- https://intelowlproject.github.io/
[14] Top Threat Intelligence Tools for 2026 (Wiz) -- https://www.wiz.io/academy/threat-intel/the-top-oss-threat-intelligence-tools
[15] Top 5 Threat Intelligence Platforms of 2026 -- https://guptadeepak.com/top-5-threat-intelligence-platforms-of-recorded-future-mandiant-crowdstrike-flashpoint-and-misp-compared/
[16] Mihari: OSINT Query Aggregator -- https://github.com/ninoseki/mihari
[17] BrewedIntel Threat Intel Resources -- https://github.com/BrewedIntel/threat-intel-resources
[18] Complete OSINT Toolkit 2026 Edition -- https://blog.cyberhawkconsultancy.org/2026/01/complete-osint-toolkit-for-threat.html
[19] Awesome CLI Frameworks (GitHub) -- https://github.com/shadawck/awesome-cli-frameworks
[20] MISP Project -- https://www.misp-project.org/

### Gemini Source Domains (from report body, redirect URLs expired)
The following domains were cited by Gemini's deep research (42 citations total). Direct URLs are not available due to Vertex AI redirect token expiration:
- securedebug.com (Metasploit architecture)
- imperva.com (Metasploit reference)
- readthedocs.io (cmd2 documentation)
- plainenglish.io (cmd2 overview)
- github.com (cmd2, Textual, Rich, CTFd, OpenCTI, IntelOwl, SpiderFoot, awesome-osint, awesome-threat-intelligence)
- mteke.com (Rich documentation)
- hackingtutorials.org (Metasploit syntax)
- netragard.com (Metasploit reference)
- medium.com (plugin architecture articles, OSINT articles)
- rapid7.com (Metasploit documentation)
- python.org (importlib.metadata, entry_points documentation)
- polito.it (gamification research)
- diva-portal.org (gamification research)
- nih.gov (CTF/gamification study)
- ctfd.io (dynamic scoring, hints, challenge documentation)
- helpnetsecurity.com (OpenCTI coverage)
- sredevops.org (OpenCTI architecture)
- dogesec.com (STIX 2.1 documentation)
- silentpush.com (CTI analysis)
- opencti.io (OpenCTI documentation)
- mintlify.com (TheHive documentation)
- oneuptime.com (TheHive reference)
- sourceforge.net (IntelOwl)
- loginsoft.com (IntelOwl analyzers)
- jyu.fi (IntelOwl research)
- github.io (IntelOwl documentation)
- shadowdragon.io (SpiderFoot reference)
- threatintel.academy (OSINT API sources)
