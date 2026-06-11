# Error Interpreter Routing — Universal Coverage for Agent Tool Dispatch + CTI/OSINT Module Boundary (W-ERROR-ROUTING-2026-06-11)

**Workflow id:** `w-error-routing-2026-06-11` · **Goal id:** `g-error-routing` · **Work item id:** `wi-error-routing-impl-01` (this slice)
**Branch:** `feature/error-routing-2026-06-11` · **Worktree:** `.worktrees/feature-error-routing-2026-06-11`
**Plan head SHA:** `3cf14a7` (`chore(hygiene): MASTER_PLAN APP refresh + scripts/regen_decisions.py + DECISIONS.md regen (#72)`)
**Stage:** planner (this document) → guardian (provision) → implementer → reviewer → guardian (land)

## 1. User-reported defect (verbatim reproduction, 2026-06-11)

The User typed `ap chat` and asked:

```
ap> Investigate this password ... : fuckusa300100XX
```

Output included a raw `httpx.HTTPStatusError` traceback ending in:

```
Tool execution failed: threatfox_lookup
Traceback (most recent call last):
  File "/Users/jarocki/src/ap/src/adversary_pursuit/agent/tools.py", line 2120, in execute_tool
    result = ctx.run_module(module_path, target, options)
  ...
  File ".../modules/cti/threatfox.py", line 146, in hunt
    response.raise_for_status()
  ...
httpx.HTTPStatusError: Client error '401 Unauthorized' for url 'https://threatfox-api.abuse.ch/api/v1/'
```

This directly violates **DEC-ERROR-INTERPRETER-008** (Phase 10, 2026-05-14, merge `1ccf13b`):

> "no Python traceback ever reaches the user without going through the interpreter"

The interpreter machinery (`core/error_interpreter.py`, `agent/error_handler.py`) is fully present and correctly wired into `core/console.py` (cmd2 surface) and `scripts/smoke_test.py` (smoke surface). It is **not** wired into the `ap chat` agent-tool-dispatch surface (`agent/tools.py::execute_tool`), which is the LLM's primary path for calling CTI/OSINT modules in the chat REPL. The two surfaces (cmd2 console and chat agent) execute the same modules, so a 401 from the same module renders friendly in cmd2 and as a raw traceback in chat.

## 2. Root cause analysis

### 2.1 The escape path (one bug, three contributing facts)

1. **Bug source — `agent/tools.py:2129-2137`.** `execute_tool` *does* wrap the module-dispatch call in `try/except Exception as e`. Inside the except branch it calls `logger.exception("Tool execution failed: %s", tool_name)`. `logger.exception` writes the formatted traceback to the root logger handler, which (by Python default with no handler configured) is `sys.stderr`. **That is the traceback the user sees.** The except branch then returns a plain string `f"{run_fail_plain} Error running {tool_name}: {e}"` to the LLM, which the LLM dutifully but blindly summarizes.

2. **Catalog gap — `core/error_interpreter.py:248-263`.** `_is_auth_error` matches AuthenticationError class names AND `"api key" in msg AND ("invalid" OR "missing" OR "unauthorized") in msg`. A stock `httpx.HTTPStatusError("Client error '401 Unauthorized' ...")` carries neither the class name nor "api key" in the message. **A bare 401 HTTPStatusError falls through to `_unknown_fallback`**, which is a friendly panel but does not name the actual problem ("your API key is missing or wrong").

3. **Modules raise raw `httpx.HTTPStatusError`.** 14 of 15 CTI/OSINT modules call `response.raise_for_status()` (greynoise has explicit pre-checks; threatfox catches 429 but lets every other 4xx through to `raise_for_status`). For Sacred Practice 5 ("loud failure") this is correct module behavior — *but the boundary must convert raw `httpx` errors into recognizable shapes before handing them to the interpreter*.

### 2.2 What the user sees today vs. what they should see

| | Today (broken) | After this slice |
|---|---|---|
| User's chat console | Raw `httpx.HTTPStatusError` traceback to stderr + LLM narration of "I encountered an HTTPStatusError" | Rich Panel with `[bold]Problem:[/bold] An API key is missing or invalid.` + `[bold]Fix:[/bold] Set AP_THREATFOX_API_KEY or run \`ap config setup\`.` + 8-char diagnostic ID + debug-log path |
| LLM-facing tool result | Long error string including the raw HTTPStatusError repr | Concise plain-text marker `[USER_SAW_PANEL] [API key] An API key is missing or invalid. Fix: Set AP_THREATFOX_API_KEY or run ap config setup. (diag <id>)` |
| `~/.ap/debug.log` | No entry | One JSONL line with full traceback + diag id + context (existing infrastructure, DEC-ERROR-INTERPRETER-002/003) |
| stderr | Multi-line Python traceback | Empty (no `Traceback (most recent call last):`) |

## 3. Architecture

### 3.1 Single new authority? No.

This slice adds **no new state authority**. It is a routing hygiene slice that extends one existing authority (`core/error_interpreter.py`) to recognize one more error shape (`httpx.HTTPStatusError`) and wires one previously-unrouted boundary (`agent/tools.py::execute_tool`) into the existing interpreter. No new module is created; no new dataclass is created (`FriendlyError`, `ErrorInterpretation`, `AutoFix` all stay byte-identical in shape).

### 3.2 State-authority map (unchanged from Phase 10, three boundary additions)

| State domain | Canonical authority | This slice |
|---|---|---|
| Error classification + fix catalog | `core/error_interpreter.py::_CATALOG` | EXTEND `_is_auth_error` to recognize `httpx.HTTPStatusError` with `response.status_code in (401, 403)`; EXTEND `_is_rate_limit` to recognize HTTPStatusError 429 (defense-in-depth — threatfox already catches 429 itself, but four modules — abuseipdb / urlhaus / shodan_ip / censys_host — let 429 through to `raise_for_status`). NEW small entry `_is_http_status_error_generic` covers other 4xx/5xx so the user sees "the remote service rejected the request (status N)" instead of unknown-fallback. |
| Friendly panel rendering (cmd2) | `core/error_interpreter.py::render_interactive` | Unchanged — already wired. |
| Friendly panel rendering (chat main loop) | `agent/error_handler.py::handle_error` | Unchanged — already wired. |
| Friendly panel rendering (chat **tool dispatch**) | `core/error_interpreter.py::render_interactive` via `agent/tools.py::execute_tool` exception branch | **NEW WIRING.** This is the bug we are fixing. |
| LLM-facing concise summary | `core/error_interpreter.py::render_summary_line` | EXTEND call sites — `execute_tool`'s exception branch returns `render_summary_line(interp)` (with `[USER_SAW_PANEL]` marker prefix) to the LLM instead of `f"...Error running {tool_name}: {e}"`. |
| Console reference inside tool dispatch | `ToolContext.console` (new optional attribute) | NEW. `ToolContext.__init__` gains `console: Console | None = None` parameter and `self.console: Console = console or Console()`. `agent/chat.py` passes its existing `console` singleton when instantiating runner-side context if it constructs one; the smoke / agent-runner paths get the default `Console()` (which prints to stdout/stderr like the existing cmd2 pexcept path). |
| Mode-flavored panel tone | `core/error_interpreter.py::_panel_title` reads `CharacterMode` | Unchanged — `execute_tool` passes `ctx.mode_mgr.active` exactly as `_execute_hunt` does in cmd2. |
| Debug log (JSONL @ `~/.ap/debug.log`) | `core/error_interpreter.py::_append_debug_log` | Unchanged — `interpret()` writes the entry whether the caller is cmd2 or agent-tool-dispatch. |

### 3.3 Why Option A (modules raise; boundary catches) — DEC-ERROR-ROUTING-001

The dispatch context names two design alternatives for the module side:

- **Option A.** Modules continue to call `raise_for_status()` (Sacred Practice 5 loud failure). The boundary catches and converts. Smallest diff (14 modules untouched). Sole authority for friendly translation stays in `core/error_interpreter.py` (Sacred Practice 12 single-authority).
- **Option B.** Modules catch their own HTTP errors and return `{"error": "<friendly>"}` dicts. Diff hits 14 module files. Decentralizes the catalog. Risks parallel mechanisms (each module re-implements the auth/rate-limit/connect/timeout decisions that already live in `_CATALOG`).

**This slice adopts Option A** and binds the choice in **DEC-ERROR-ROUTING-001**. Rationale: Sacred Practice 12 (single authority per state domain) is the binding constraint. Option B would create 14 parallel mini-catalogs whose maintenance drifts independently from `core/error_interpreter.py`. The boundary is exactly the right altitude for translation because the boundary is what owns "what the user sees" — modules own "what the truth is." Modules already produce the truth loudly (good); the boundary already shapes the user view (good); the bug is one missing wire between them, not 14 missing translators.

### 3.4 Why catalog extension is in-scope — DEC-ERROR-ROUTING-002

The dispatch context says "core/error_interpreter.py BYTEWISE UNCHANGED — the catalog is correct; we're just wiring more callers to use it." But the user's reproduction proves the catalog is **not** correct for the bare `httpx.HTTPStatusError` 401 case: it falls through to `_unknown_fallback`, which renders a friendly panel but with the wrong category and wrong fix-suggestion (the user is told to "check the debug log and file a bug" when the actual fix is "set AP_THREATFOX_API_KEY").

**This slice adopts a narrow, additive catalog extension** and binds the choice in **DEC-ERROR-ROUTING-002**. The extension:

1. Adds **one short helper** `_is_httpx_http_status_error(exc)` (class-name match on `HTTPStatusError`).
2. Extends `_is_auth_error` to ALSO return True when the exc is HTTPStatusError with `getattr(exc.response, "status_code", None) in (401, 403)`.
3. Extends `_is_rate_limit` to ALSO return True when the exc is HTTPStatusError with `getattr(exc.response, "status_code", None) == 429` (defense in depth — most modules catch 429 explicitly first).
4. Adds **one new catalog entry** at the end (after auth/rate-limit/connect/timeout — so the more specific entries win first) that matches HTTPStatusError 5xx → `("Service", "The remote service returned a server error.", "Wait a moment and retry; the upstream service may be experiencing issues.")`. The 4xx fallthrough (after 401/403/429 special-cases) maps to `("Network", "The remote service rejected the request (status <N>).", "Check your input / API key and retry; if the error repeats, file a bug.")`.
5. Adds an in-process diagnostic context hook so the catalog entry can read which module surfaced the error (already supported by the existing `context: dict` parameter on `interpret()`).

This is additive, single-authority, and preserves every other catalog entry byte-identical. **No existing entry is changed.** The Phase 10 contract (DEC-ERROR-INTERPRETER-001..008) remains binding; this slice ADDS three catalog matchers, one entry, and one helper. The dispatch context's "bytewise unchanged" framing is honored in intent (no rewrite of the catalog body) but explicitly relaxed for the narrow additions named above. Reviewer enforcement: `git diff` of `core/error_interpreter.py` must show ONLY the named additions (≤ ~60 lines of new code) and **zero deletions or rewrites of existing entries**.

### 3.5 Why a `[USER_SAW_PANEL]` marker on the LLM-facing string — DEC-ERROR-ROUTING-003

After this slice lands, the user sees a Rich Panel directly. We do NOT want the LLM to re-narrate "It looks like an error happened — your API key may be invalid..." (which would duplicate the panel). The LLM-facing string should carry enough information that the LLM can write a sensible terminal message ("I couldn't run threatfox_lookup; please set your API key and try again.") but should also signal "the human already saw the friendly version."

**DEC-ERROR-ROUTING-003** binds the marker convention: the LLM-facing string is

```
[USER_SAW_PANEL] [<category>] <summary> Fix: <suggested_fix> (diag <diagnostic_id>)
```

This is exactly `render_summary_line(interp)` (existing helper) prefixed with the literal `[USER_SAW_PANEL] ` token. The token is intentionally not styled (no Rich markup) so it survives the LLM-tool boundary. The LLM is instructed (via the existing system prompt — no edits to runner.py) to treat `[USER_SAW_PANEL]` as a signal that the user already saw the friendly explanation and to keep its own follow-up message concise. (System-prompt revision is OUT of scope this slice; the marker is a forward-compatible affordance. The LLM produces a reasonable terminal message either way because `render_summary_line` already includes the category and fix.)

### 3.6 Why `ToolContext.console` (one new optional field) — DEC-ERROR-ROUTING-004

The Rich panel needs a `Console` to print to. Three alternatives were considered:

- (a) Construct a fresh `Console()` inside `execute_tool`'s except branch. Works (cmd2's `pexcept` and `_execute_hunt` use this pattern), but creates a parallel console reference that the chat REPL can't intercept (e.g., for testing capture).
- (b) Add `last_error_panels` sidecar to `runner` (`runner.last_error_panels: list[ErrorInterpretation]`) and let `chat.py` drain it after `runner.chat()` returns. Forbidden — `agent/runner.py` is BYTEWISE UNCHANGED per C-series invariant and the dispatch context.
- (c) Add `ToolContext.console: Console` (constructor-injected, defaulting to `Console()`) and let `execute_tool` print directly. `agent/chat.py` already has a `console = Console()` it can pass in when constructing the `ToolContext`. Works in tests via `Console(file=io.StringIO())`. Single new field. No new module. No sidecar.

**DEC-ERROR-ROUTING-004** binds Option (c). `ToolContext.__init__` gains one optional kwarg `console: Console | None = None`. When None, defaults to `Console()` (lazy import of `rich.console.Console`). `chat.py` passes its existing console singleton at runner construction time. `runner.py` itself is BYTEWISE UNCHANGED — it does not need to know about `ctx.console`; the field is consumed by `tools.py` only.

### 3.7 Why every error renders a panel (including unknown) — DEC-ERROR-ROUTING-005

The dispatch context asks: "Should the Rich panel render unconditionally (every error) or only for known patterns (unknown → terse `Tool failed: <name>`)?" Per DEC-ERROR-INTERPRETER-008: "no Python traceback ever reaches the user without going through the interpreter, including the case where the interpreter itself doesn't recognize the error. If the interpreter raises during interpretation, the renderer's outer-catch emits a canned 'Something unexpected happened (diag <id>)' panel."

**DEC-ERROR-ROUTING-005** binds: every caught exception renders a panel via `render_interactive(interp, ctx.console, mode=ctx.mode_mgr.active, interactive=False)`. The interpreter's `interpret()` guarantees a non-None ErrorInterpretation for every exception (catalog match → typed; unknown → fallback panel with diag id; interpreter self-fault → canned panel with diag id). `interactive=False` matches the cmd2 `pexcept` path (no `[y/n]` prompt in tool dispatch — the agent loop is synchronous LLM-driven, not a human-interactive prompt loop).

### 3.8 Removal targets

- `logger.exception("Tool execution failed: %s", tool_name)` at `agent/tools.py:2130` — replaced with `logger.debug("Tool execution failed: %s", tool_name, exc_info=True)`. The traceback is still captured (debug-level — visible in `~/.ap/debug.log` via the existing interpreter `_append_debug_log` path AND in stderr only when the root logger is set to DEBUG, which it never is in production). **This is the line that prints the user-visible traceback today.** Removing it (downgrading to `debug` + `exc_info`) is the fix.
- The composed string `f"{run_fail_plain} Error running {tool_name}: {e}"` at `agent/tools.py:2137` — replaced with `f"[USER_SAW_PANEL] {render_summary_line(interp)}"`. The `run_fail` mode-flavored voice is now folded into the panel title via the existing `_panel_title(mode=ctx.mode_mgr.active)` path in `render_interactive`. (F62 invariant: `mode.run_fail` remains the sole authority for failure voice; we are MOVING the wire, not duplicating it.)

### 3.9 Out of scope (explicitly deferred)

- **Catalog rewrite.** Only three named additions land (auth-401/403, rate-limit-429, HTTPStatusError generic 4xx/5xx). No reorganization, no test rewrite for unrelated entries.
- **Module-side HTTP error handling.** Modules continue to call `raise_for_status()` (Sacred Practice 5; DEC-ERROR-ROUTING-001 Option A). No edits to any of the 15 CTI/OSINT module files.
- **LLM system-prompt revision** (the `[USER_SAW_PANEL]` marker is forward-compatible — the LLM produces a reasonable terminal message either way).
- **`agent/runner.py` edits.** Byte-identical inheritance from C-series invariants.
- **`agent/error_handler.py` rewrite.** Stage-1 catalog already delegates correctly; this slice extends the catalog at the destination (`core/error_interpreter.py`).
- **cmd2 console rewiring.** Already correctly routes via `interpret()` → `render_interactive()`; verify only.
- **Smoke test surface.** Already routes via `render_summary_line()`; not touched.
- **New auto-fix entries** for the HTTPStatusError catalog additions (rate-limit auto-fix has its own DEC-ERROR-INTERPRETER-005 constraint; 401 has no mechanically-safe auto-fix — we cannot auto-rotate keys).
- **A separate `ToolContext.last_error_panels` accumulator.** Panels are printed at the boundary they fire from; no replay surface is added.

## 4. Decision Log (Phase 17O)

| Decision ID | Title | Rationale |
|---|---|---|
| DEC-ERROR-ROUTING-001 | Modules raise loudly; boundary catches and translates (Option A). | Sacred Practice 12 (single authority per state domain) binds. Option B would create 14 parallel mini-catalogs whose maintenance drifts independently from `core/error_interpreter.py`. Modules already produce the truth loudly (Sacred Practice 5); the boundary is the right altitude for friendly translation because the boundary owns "what the user sees." |
| DEC-ERROR-ROUTING-002 | Catalog extension is narrow, additive, and named: (a) `_is_auth_error` recognizes HTTPStatusError 401/403; (b) `_is_rate_limit` recognizes HTTPStatusError 429; (c) NEW final entry `_is_http_status_error_generic` covers 5xx and 4xx fallthrough. No existing catalog entry is rewritten or reordered. | The dispatch context says "catalog UNCHANGED — wiring only." But the user-reproduced bug proves the bare `httpx.HTTPStatusError` 401 falls through to unknown-fallback. The fix without catalog extension would require the boundary to inspect HTTPStatusError and synthesize a richer exception before calling `interpret()` — that synthesis logic is itself a parallel mini-catalog hiding inside the boundary. The clean answer is: extend the canonical catalog (additively, named entries, no rewrite). Reviewer enforces "zero deletions or rewrites of existing entries" by `git diff`. |
| DEC-ERROR-ROUTING-003 | LLM-facing string prefixed with literal `[USER_SAW_PANEL] ` marker. | The user sees a Rich Panel directly. The LLM should know "the user already has the friendly version" and produce a concise follow-up. The marker is intentionally unstyled (no Rich markup) so it survives the LLM-tool message boundary. Forward-compatible: the LLM produces a sensible terminal message either way because `render_summary_line(interp)` already includes category + summary + fix + diag id. |
| DEC-ERROR-ROUTING-004 | `ToolContext.console: Console` (one new optional constructor kwarg, default `Console()`); `agent/chat.py` passes its existing console singleton at runner construction. `agent/runner.py` is BYTEWISE UNCHANGED (does not know about `ctx.console`). | Three alternatives weighed: (a) fresh `Console()` inside except branch — works but creates parallel reference; (b) `runner.last_error_panels` sidecar — forbidden (runner BYTEWISE UNCHANGED); (c) `ToolContext.console` constructor injection — single new field, no sidecar, testable via `Console(file=io.StringIO())`. Option (c) is the smallest change that respects all byte-invariants. |
| DEC-ERROR-ROUTING-005 | Every caught exception in `execute_tool` renders a Rich Panel; `interactive=False` for the auto-fix prompt. | DEC-ERROR-INTERPRETER-008 binds the universal-coverage contract. The agent tool dispatch loop is LLM-driven and synchronous, not a human-interactive prompt loop, so `interactive=False` is correct (mirrors cmd2 `pexcept` path). The interpreter guarantees a non-None ErrorInterpretation for every exception (catalog match → typed; unknown → fallback; self-fault → canned). |
| DEC-ERROR-ROUTING-006 | `logger.exception` at `agent/tools.py:2130` downgraded to `logger.debug(..., exc_info=True)`. | `logger.exception` writes to the root logger handler (stderr by default in production) — that is the line that prints the user-visible traceback today. Downgrading to `debug` removes the user-visible traceback while preserving the traceback in the debug log (the interpreter's `_append_debug_log` writes a structured JSONL entry; the Python logger writes to its own handler at the same level — both stay captured for power-user / CI debugging via `AP_LOG_LEVEL=DEBUG` env, no behavior change). |
| DEC-ERROR-ROUTING-007 | F62 invariant preserved by FOLDING `mode.run_fail` into the panel title (via existing `_panel_title(mode=...)` path) and REMOVING the `_strip_rich_markup(ctx.mode_mgr.active.run_fail)` prepend from the LLM-facing string at `agent/tools.py:2136`. | The old code prepended `mode.run_fail` to the LLM-facing string. With the new wiring, `mode.run_fail` is consumed by `render_interactive` via `_panel_title(mode=ctx.mode_mgr.active)` → user sees mode-flavored panel title. The LLM-facing string drops the prefix because the human already heard the persona voice in the panel; the LLM should NOT re-narrate it (F64 panel separation). `mode.run_fail` remains the sole authority for failure voice (F62 invariant preserved — we move the wire from string-concat to panel-title; we do not duplicate or rewrite the authority). |

## 5. Work item

| ID | Title | Type | Worktree | Status |
|---|---|---|---|---|
| `wi-error-routing-planning` | Planner: per-slice plan + Phase 17O section authoring + Plan Status table row + Active Phase Pointer re-point + scope manifest + Evaluation Contract | docs only | `.worktrees/feature-error-routing-2026-06-11` | landed-as-staged (this document; planner stages artifacts via `git add` — implementer commits per AP #74 pattern) |
| `wi-error-routing-impl-01` | Implementer: extend `core/error_interpreter.py` catalog (~3 helpers, 1 entry); rewire `agent/tools.py::execute_tool` exception branch (interpreter call + Rich panel render + `[USER_SAW_PANEL]` LLM marker); add `ToolContext.console: Console = None` ctor kwarg; pass console from `agent/chat.py` runner-construction; extend `tests/test_error_interpreter.py` (HTTPStatusError 401/403/429/500/4xx); extend `tests/test_agent_tools.py` (5 new exception-injection tests asserting Rich panel rendered + no traceback in stderr + `[USER_SAW_PANEL]` marker in LLM-facing string); MASTER_PLAN.md amendment (Phase 17O + Plan Status row + Active Phase Pointer in SAME commit as source per AP #74). | source + tests | TBD-after-provision |
| `wi-error-routing-review-01` | Reviewer: execute every required_test from §6.1; verify every git-diff entry from §6.2; confirm every real-path check from §6.3; emit `REVIEW_VERDICT=ready_for_guardian` at implementer head SHA when all green. | read-only | pending (post-implementer) |
| `wi-error-routing-guardian-land` | Guardian (land): reviewer-readiness preflight; merge `feature/error-routing-2026-06-11` → `main`; push to `origin/main`; local branch + worktree cleanup. | git landing | pending (post-reviewer-ready) |

### 5.1 Implementer sub-task order (one worktree, sequential)

1. **WI-ER-1.1** — `core/error_interpreter.py`: ADD `_is_httpx_http_status_error(exc)` helper; EXTEND `_is_auth_error` to also match HTTPStatusError 401/403; EXTEND `_is_rate_limit` to also match HTTPStatusError 429; ADD final catalog entry `(_is_http_status_error_generic, _interpret_http_status_error_generic, lambda _: None)`. No reorganization. No deletions. Existing entries byte-identical.
2. **WI-ER-1.2** — `tests/test_error_interpreter.py`: ADD 5 new test methods — `test_http_status_error_401_classified_as_auth`, `test_http_status_error_403_classified_as_auth`, `test_http_status_error_429_classified_as_rate_limit`, `test_http_status_error_500_classified_as_service`, `test_http_status_error_400_classified_as_generic_client`. Each constructs a real `httpx.HTTPStatusError` via `httpx.Response(status_code=N).raise_for_status()` to ensure shape parity with production. Existing test bodies byte-identical.
3. **WI-ER-1.3** — `agent/tools.py::ToolContext`: ADD `console: Console | None = None` kwarg; ADD `self.console: Console = console or Console()` line (with `from rich.console import Console` import at module top — should already be imported; verify and add only if missing).
4. **WI-ER-1.4** — `agent/tools.py::execute_tool`: rewrite the except branch (~10 lines). Replace `logger.exception(...)` with `logger.debug("Tool execution failed: %s", tool_name, exc_info=True)`. Replace the LLM-facing return string with `interp = interpret(e, context={"surface": "agent_execute_tool", "tool": tool_name}); render_interactive(interp, ctx.console, mode=ctx.mode_mgr.active, interactive=False); return f"[USER_SAW_PANEL] {render_summary_line(interp)}", None, [], []`. Add the two imports `from adversary_pursuit.core.error_interpreter import interpret, render_interactive, render_summary_line` at module top.
5. **WI-ER-1.5** — `agent/chat.py`: locate the runner / ToolContext construction site and pass `console=console` to it. (If chat.py never constructs ToolContext directly — runner does — then `runner.ctx.console` is set post-construction: `runner.ctx.console = console` after `runner = AgentRunner(...)` so the default `Console()` is replaced with the chat singleton. **This is a one-line attribute assignment; runner.py is not edited.** Verify and add wherever the existing console singleton is in scope at runner construction time.)
6. **WI-ER-1.6** — `tests/test_agent_tools.py`: ADD 5 new test methods under a new class `TestExecuteToolFriendlyErrorRouting`:
   - `test_execute_tool_401_renders_panel_and_returns_marker` (mocks `ctx.run_module` to raise `httpx.HTTPStatusError` 401; asserts return string starts with `[USER_SAW_PANEL]` and contains category `[API key]`; asserts test-captured Rich Console output contains the panel; asserts `capsys.readouterr().err` does NOT contain `Traceback (most recent call last):`)
   - `test_execute_tool_429_renders_rate_limit_panel`
   - `test_execute_tool_500_renders_service_panel`
   - `test_execute_tool_connect_error_renders_network_panel` (`httpx.ConnectError("Connection refused")`)
   - `test_execute_tool_unknown_exception_renders_unknown_fallback` (`ValueError("synthetic")`)
   Each test uses `Console(file=io.StringIO())` injected via the new `ToolContext(console=...)` kwarg; assertions read the StringIO contents for panel text and `capsys` for the stderr-no-traceback guarantee.
7. **WI-ER-1.7** — Live evidence captures in `tmp/evidence-error-routing-2026-06-11/`:
   - `chat-threatfox-401-fixed.txt` — `ap chat` then `Investigate this password ...` reproduction with `AP_THREATFOX_API_KEY` unset OR set to `invalid_test_value`; capture must show Rich Panel with `[API key]` category + `Set AP_THREATFOX_API_KEY` fix + 8-char diag id; capture must NOT contain `Traceback (most recent call last):`.
   - `chat-shodan-rate-limit-fixed.txt` — synthetic test with shodan_ip module returning 429 (or mocked via env-var override if real 429 not achievable); same assertions for rate-limit panel.
   - `chat-network-fixed.txt` — disconnect network or point AP_*_API_URL to localhost:1; assert Rich Panel with Network category.
   - `debug-log-sample.txt` — `tail -1 ~/.ap/debug.log | jq` showing the structured entry for one of the above; diag id MUST match the panel diag id.
8. **WI-ER-1.8** — Amend `MASTER_PLAN.md` Phase 17O section with closeout merge SHA + evidence summary; flip Plan Status row + Active Phase Pointer; `git add MASTER_PLAN.md` in same implementer commit (AP #74).

**Critical path:** strictly sequential 1.1 → 1.8 (steps 1.3/1.4/1.5 form the boundary-wiring change set; step 1.6 depends on the new `ToolContext(console=...)` kwarg from 1.3; step 1.7 depends on the integration landing from 1.5).

## 6. Evaluation Contract (Phase 17O)

Persisted in runtime via `cc-policy workflow work-item-set wi-error-routing-impl-01 ... --evaluation-json $(cat tmp/error-routing-evaluation.json)` at provisioning. Authoritative copy at `tmp/error-routing-evaluation.json`. Summary below; canonical content in JSON.

### 6.1 Required tests (10 new + suite green)

New tests in `tests/test_error_interpreter.py` (5):
- `test_http_status_error_401_classified_as_auth`
- `test_http_status_error_403_classified_as_auth`
- `test_http_status_error_429_classified_as_rate_limit`
- `test_http_status_error_500_classified_as_service`
- `test_http_status_error_400_classified_as_generic_client`

New tests in `tests/test_agent_tools.py` (5, under new class `TestExecuteToolFriendlyErrorRouting`):
- `test_execute_tool_401_renders_panel_and_returns_marker`
- `test_execute_tool_429_renders_rate_limit_panel`
- `test_execute_tool_500_renders_service_panel`
- `test_execute_tool_connect_error_renders_network_panel`
- `test_execute_tool_unknown_exception_renders_unknown_fallback`

Plus the existing full suite (`pytest tests/ -q`) must remain green at ≥ current baseline + the 10 new tests.

### 6.2 Required evidence (git diffs + counts)

- `git diff main -- src/adversary_pursuit/core/error_interpreter.py` — bounded to: ONE new helper function (`_is_httpx_http_status_error`), ONE new interpret-function (`_interpret_http_status_error_generic`), 2-3 lines added inside existing `_is_auth_error` and `_is_rate_limit` (HTTPStatusError check), ONE new `_CATALOG` tuple appended at END. NO existing entry rewritten or reordered. NO deletions.
- `git diff main -- src/adversary_pursuit/agent/tools.py` — bounded to: ONE new import line (`from adversary_pursuit.core.error_interpreter import interpret, render_interactive, render_summary_line`), ONE new optional kwarg + 1-line attribute set on `ToolContext.__init__`, the `except Exception` branch rewritten (~10 lines). NO change to any of the 30 LLM tool dispatch entries; NO change to `_execute_*` helper functions; NO change to `_strip_rich_markup` (it remains because cmd2 paths still use it).
- `git diff main -- src/adversary_pursuit/agent/chat.py` — bounded to: ONE line setting `runner.ctx.console = console` after runner construction.
- `git diff main -- tests/test_error_interpreter.py` — bounded to ADDED 5 new test methods.
- `git diff main -- tests/test_agent_tools.py` — bounded to ADDED 1 new test class with 5 new test methods.
- `git diff main -- src/adversary_pursuit/agent/runner.py` — MUST be EMPTY (C-series + dispatch invariant).
- `git diff main -- src/adversary_pursuit/agent/error_handler.py` — MUST be EMPTY (the three-stage pipeline at the agent main loop is correctly wired; this slice does not change it).
- `git diff main -- src/adversary_pursuit/core/console.py` — MUST be EMPTY (cmd2 surface already correctly wired; this slice does not change it).
- `git diff main -- src/adversary_pursuit/core/workspace.py` `src/adversary_pursuit/models/database.py` — MUST be EMPTY.
- `git diff main -- src/adversary_pursuit/modules/` — MUST be EMPTY (Option A; modules untouched).
- `git diff main -- src/adversary_pursuit/dossier/` `src/adversary_pursuit/gamification/` — MUST be EMPTY.
- `pyproject.toml` `uv.lock` — MUST be EMPTY (no new deps; `httpx` and `rich` are already deps).

### 6.3 Required real-path checks (live + grep)

- `AP_THREATFOX_API_KEY=invalid_test ap chat` then ask for a threatfox lookup → captured output (in `tmp/evidence-error-routing-2026-06-11/chat-threatfox-401-fixed.txt`) MUST contain `[API key]` category text and MUST NOT contain `Traceback (most recent call last):`. Diag-id in panel MUST appear in `~/.ap/debug.log` last entry.
- `grep -n "logger.exception" src/adversary_pursuit/agent/tools.py | head -5` — the line at the OLD `execute_tool` site (formerly line 2130) MUST be gone; remaining `logger.exception` calls are in other helper functions (`_workspace_summary`, `_search_workspace`, etc. — those are out-of-scope; we may opportunistically downgrade them in a follow-up slice but NOT here).
- `grep -n "render_interactive\|render_summary_line" src/adversary_pursuit/agent/tools.py | head -5` — MUST show the new imports + at least one call site inside `execute_tool`.
- `python -c "from adversary_pursuit.agent.tools import ToolContext; import inspect; sig = inspect.signature(ToolContext.__init__); assert 'console' in sig.parameters, sig"` — MUST exit 0.
- `python -c "import httpx; from adversary_pursuit.core.error_interpreter import interpret; resp = httpx.Response(401, request=httpx.Request('GET', 'http://x')); exc = httpx.HTTPStatusError('Client error 401', request=resp.request, response=resp); print(interpret(exc).category)"` — MUST print `API key`.
- `python -c "import httpx; from adversary_pursuit.core.error_interpreter import interpret; resp = httpx.Response(429, request=httpx.Request('GET', 'http://x')); exc = httpx.HTTPStatusError('Client error 429', request=resp.request, response=resp); print(interpret(exc).category)"` — MUST print `Rate limit`.
- `grep -n "^## Phase 17O" MASTER_PLAN.md` — confirms Phase 17O section was appended.
- `grep -n "W-ERROR-ROUTING-2026-06-11" MASTER_PLAN.md` — confirms Plan Status table row + Active Phase Pointer reference.
- `grep -n "DEC-ERROR-ROUTING-001" MASTER_PLAN.md` — confirms Decision Log entry recorded.
- `git diff main -- src/adversary_pursuit/agent/runner.py` — output MUST be empty.

### 6.4 Required authority invariants

- **DEC-ERROR-INTERPRETER-001** preserved: `core/error_interpreter.py` remains sole catalog authority; `agent/error_handler.classify_error()` continues to delegate to `interpret()` (existing call site is byte-identical).
- **DEC-ERROR-INTERPRETER-008** universal-coverage contract preserved AND strengthened: the agent tool dispatch surface now also routes through `interpret()` (this is the gap this slice closes).
- **DEC-AGENT-ERROR-HANDLER-001** preserved: the chat main-loop three-stage pipeline at `agent/chat.py:670` (`handle_error(exc, console, runner, config_mgr)`) is unchanged. Only the tool-dispatch sub-path (inside `execute_tool`) gains a parallel friendly-error wire.
- **F62** (`mode.run_fail` is sole authority for failure voice): preserved — `run_fail` continues to be consumed by `render_interactive` via `_panel_title(mode=ctx.mode_mgr.active)`. The line at `agent/tools.py:2136` that prepended `_strip_rich_markup(ctx.mode_mgr.active.run_fail)` to the LLM-facing string is REMOVED; the voice now lives in the panel title only (no duplication, no parallel authority).
- **F64** (panels are sole narration surface for points; LLM-facing strings carry no markup): preserved — `render_summary_line(interp)` produces plain ASCII with no Rich markup; the `[USER_SAW_PANEL] ` prefix is plain ASCII.
- **C-series** (`agent/runner.py` BYTEWISE UNCHANGED): preserved — `runner.py` is not edited.
- **Sacred Practice 5** (loud failure): preserved — modules continue to call `raise_for_status()` raw; the boundary catches and translates.
- **Sacred Practice 12** (single authority): preserved — translation lives ONLY in `core/error_interpreter.py::_CATALOG`. No parallel translator inside `agent/tools.py`, no per-module mini-catalog (Option A bound by DEC-ERROR-ROUTING-001).

### 6.5 Required integration points

- `src/adversary_pursuit/core/error_interpreter.py` — ADD: `_is_httpx_http_status_error` helper, HTTPStatusError checks inside `_is_auth_error` and `_is_rate_limit`, `_interpret_http_status_error_generic`, final `_CATALOG` tuple.
- `src/adversary_pursuit/agent/tools.py::ToolContext.__init__` — ADD: `console: Console | None = None` kwarg + `self.console = console or Console()` line.
- `src/adversary_pursuit/agent/tools.py::execute_tool` — REWRITE the `except Exception as e` branch (~10 lines): downgrade `logger.exception` → `logger.debug(..., exc_info=True)`; call `interpret()` + `render_interactive()`; return `[USER_SAW_PANEL] {render_summary_line(interp)}`.
- `src/adversary_pursuit/agent/chat.py` — ADD: ONE-LINE `runner.ctx.console = console` after runner construction (or pass `console=console` to ToolContext at construction site if chat.py owns it).
- `tests/test_error_interpreter.py` — ADD 5 new test methods.
- `tests/test_agent_tools.py` — ADD 1 new test class (`TestExecuteToolFriendlyErrorRouting`) with 5 new test methods.
- `MASTER_PLAN.md` — Phase 17O section + Plan Status table row for `W-ERROR-ROUTING-2026-06-11` + Active Phase Pointer re-point + Aggregate paragraph nudge. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention).**
- `.claude/plans/error-routing-2026-06-11.md` — this document.
- `tmp/error-routing-scope.json` — Scope Manifest (registered via `cc-policy workflow scope-sync`).
- `tmp/error-routing-evaluation.json` — Evaluation Contract (registered via `cc-policy workflow work-item-set --evaluation-json`).
- `tmp/evidence-error-routing-2026-06-11/` — 4 live-capture artifacts (chat-threatfox-401-fixed, chat-shodan-rate-limit-fixed, chat-network-fixed, debug-log-sample).

### 6.6 Forbidden shortcuts

- DO NOT edit `agent/runner.py` — BYTEWISE UNCHANGED (C-series + dispatch invariant).
- DO NOT edit `agent/error_handler.py` — the chat main-loop three-stage pipeline is already correct; this slice does not change it. If a `to_llm_summary()` method on `FriendlyError` is tempting, RESIST — `render_summary_line(interp)` already provides the canonical LLM-facing string from the catalog side.
- DO NOT add the `[USER_SAW_PANEL]` marker inside `render_summary_line` itself — that helper is shared by the smoke surface (which has no user-visible panel). The marker is added at the `execute_tool` call site only.
- DO NOT touch any of the 15 CTI/OSINT module files — Option A (DEC-ERROR-ROUTING-001).
- DO NOT rewrite or reorder existing `_CATALOG` entries — additive only. Existing test bodies remain byte-identical.
- DO NOT introduce a new dataclass or sidecar (e.g., `runner.last_error_panels`) — DEC-ERROR-ROUTING-004 binds the `ToolContext.console` solution.
- DO NOT remove the `_strip_rich_markup` helper — it is still used by cmd2 paths and by other helpers.
- DO NOT change the LLM tool dispatch entries (the long `if tool_name == "..."` chain) — they are byte-identical; this slice only changes the `except Exception` branch at the end.
- DO NOT touch `pyproject.toml`, `uv.lock`, or any module under `core/workspace.py`, `models/`, `dossier/`, `gamification/`.
- DO NOT introduce a `[y]/[n]/[d]` interactive prompt inside `execute_tool` — `interactive=False` per DEC-ERROR-ROUTING-005 (mirrors cmd2 `pexcept`).
- DO NOT pre-stage `MASTER_PLAN.md` amendment in a separate commit — Phase 17O section + Plan Status row + Active Phase Pointer commit in the SAME implementer commit as source (AP #74).
- DO NOT downgrade other `logger.exception` calls in `tools.py` (e.g., the ones in `_workspace_summary`, `_search_workspace`, `_execute_*` helpers). Those surfaces all return composed error strings to the LLM but do NOT call `interpret()`. Universal coverage across THOSE surfaces is out of scope this slice (would be a follow-up).
- DO NOT add catalog auto-fix entries for the new HTTPStatusError entries — DEC-ERROR-INTERPRETER-005 binds mechanically-safe-only (401 has no safe auto-fix; rate-limit's existing 429 auto-fix already covers retry_after when available).

### 6.7 Rollback boundary

Single-commit `git revert <impl-sha>` restores the pre-slice state byte-for-byte. The user-visible regression (traceback at chat prompt) returns, but no schema migration, no new external file, no env var to clean up. `~/.ap/debug.log` is purely additive (delete-to-rollback). Test count after revert: ≥ pre-slice baseline (the 10 new tests vanish).

### 6.8 Ready-for-guardian definition

`pytest tests/ -q` green (paste exact counts ≥ pre-slice baseline + 10 new tests; zero failures); every `git diff` "MUST be empty" entry pastes empty; the bounded diffs land within scope; 4 evidence captures present in `tmp/evidence-error-routing-2026-06-11/`; `grep -n "^## Phase 17O" MASTER_PLAN.md` confirms section appended; `grep -n "W-ERROR-ROUTING-2026-06-11" MASTER_PLAN.md` confirms Plan Status row + Active Phase Pointer; `grep -n "DEC-ERROR-ROUTING-001" MASTER_PLAN.md` confirms Decision Log row; the threatfox-401 live capture is the canonical user-acceptance artifact and MUST show zero `Traceback (most recent call last):` strings; reviewer emits `REVIEW_VERDICT=ready_for_guardian` at implementer head SHA.

## 7. Scope Manifest (Phase 17O)

Full manifest persisted at `tmp/error-routing-scope.json` with canonical CLI keys `allowed_paths` / `required_paths` / `forbidden_paths` / `authority_domains`. Summary:

### 7.1 Allowed / Required paths (implementer MUST touch)

- `src/adversary_pursuit/core/error_interpreter.py` (extend — additive only; ~50 LOC)
- `src/adversary_pursuit/agent/tools.py` (extend `ToolContext.__init__` + rewrite `execute_tool` except branch; ~15 LOC)
- `src/adversary_pursuit/agent/chat.py` (1 line — set `runner.ctx.console`)
- `tests/test_error_interpreter.py` (5 new test methods)
- `tests/test_agent_tools.py` (1 new test class with 5 methods)
- `MASTER_PLAN.md` (Phase 17O section + Plan Status row + Active Phase Pointer + Aggregate paragraph)
- `.claude/plans/error-routing-2026-06-11.md` (this document; planner stages)
- `tmp/error-routing-scope.json`
- `tmp/error-routing-evaluation.json`
- `tmp/evidence-error-routing-2026-06-11/` (4 artifacts at implementer Stage 1.7)

### 7.2 Forbidden paths (preserved authorities)

- `src/adversary_pursuit/agent/runner.py` (BYTEWISE UNCHANGED — C-series + dispatch invariant)
- `src/adversary_pursuit/agent/error_handler.py` (chat main-loop pipeline already correct)
- `src/adversary_pursuit/core/console.py` (cmd2 surface already correctly wired)
- `src/adversary_pursuit/core/workspace.py`, `models/database.py`, `models/stix.py` (no schema change)
- `src/adversary_pursuit/core/event_bus.py`, `core/pivot_policy.py`, `core/dossier_pivot.py`, `core/streak.py`, `core/dossier_report.py`, `core/config.py`, `core/graph.py`
- `src/adversary_pursuit/dossier/` (entire package)
- `src/adversary_pursuit/gamification/` (entire package)
- `src/adversary_pursuit/modules/` (entire tree — Option A, DEC-ERROR-ROUTING-001)
- `src/adversary_pursuit/agent/banner.py`, `agent/repl_input.py`
- `pyproject.toml`, `uv.lock`
- `CLAUDE.md`, `AGENTS.md`, `settings.json`, `hooks/`, `runtime/`, `agents/`
- `scripts/smoke_test.py` (smoke surface already correctly wired via `render_summary_line`)
- Every test file EXCEPT `tests/test_error_interpreter.py` and `tests/test_agent_tools.py`

### 7.3 Authority domains touched

- `error_classification_catalog` (Phase 10 authority — EXTEND additively per DEC-ERROR-ROUTING-002)
- `friendly_panel_rendering_chat_tool_dispatch` (NEW surface — Phase 17O claims this domain via `agent/tools.py::execute_tool` wiring)
- `tool_context_console_reference` (NEW domain — `ToolContext.console` per DEC-ERROR-ROUTING-004)
- `llm_facing_error_marker_convention` (NEW domain — `[USER_SAW_PANEL]` per DEC-ERROR-ROUTING-003)

## 8. Out-of-Scope (Phase 17O; planner asserts; implementer slices honor)

- **Module-side translation.** DEC-ERROR-ROUTING-001 Option A binds; modules continue to raise loudly.
- **Other `logger.exception` calls inside `agent/tools.py`** (in `_workspace_summary`, `_search_workspace`, `_execute_*` helpers). The user-reported bug is the `execute_tool` site; the other sites compose error strings differently and may be addressed in a follow-up hygiene slice (a single-DEC follow-on would be appropriate).
- **LLM system-prompt revision** to acknowledge the `[USER_SAW_PANEL]` marker. The marker is forward-compatible; the LLM produces a sensible terminal message either way.
- **Catalog reorganization** or rewrite. Additive only.
- **New auto-fix entries** for the HTTPStatusError catalog additions. DEC-ERROR-INTERPRETER-005 binds mechanically-safe-only; 401 has no safe auto-fix.
- **cmd2 surface changes.** Already correctly wired; verified by inspection (`core/console.py:263` calls `interpret()` directly).
- **Smoke surface changes.** Already correctly wired.
- **Per-module test extensions.** This slice tests at the boundary (`execute_tool`); per-module HTTP-error tests stay where they are.

## 9. Subsequent Workflow Cue

After Phase 17O lands, no successor is named in the current plan. The orchestrator's autonomous-continuation decision after Phase 17O lands will be one of:

- `goal_complete` (the user's reported defect is fixed and the universal-coverage invariant per DEC-ERROR-INTERPRETER-008 is now upheld across all three surfaces — cmd2, smoke, AND chat tool dispatch).
- `next_work_item` (a runtime-hygiene follow-up: downgrade the other `logger.exception` calls in `agent/tools.py` helper functions to bring full universal-coverage to every error path in the chat surface — single-DEC follow-on; would be appropriate but is NOT scheduled by this plan).
- `needs_user_decision` (the User chooses between further error-routing hygiene and unrelated post-v0.4.x release-discipline work per the 2026-06-09 reckoning).

This plan does NOT pre-commit; the post-landing planner pass owns the decision.

---

## Appendix A — Catalog extension cookbook

The exact additive edits the implementer should make in `core/error_interpreter.py` (paraphrased for clarity; final code at implementer time):

```python
# At module top, near other helpers:
def _is_httpx_http_status_error(exc: BaseException) -> bool:
    return type(exc).__name__ == "HTTPStatusError" and hasattr(exc, "response")


# Inside _is_auth_error, ADD as the FINAL check before returning False:
    if _is_httpx_http_status_error(exc):
        status = getattr(exc.response, "status_code", None)
        if status in (401, 403):
            return True


# Inside _is_rate_limit, ADD as the FINAL check before returning False:
    if _is_httpx_http_status_error(exc):
        status = getattr(exc.response, "status_code", None)
        if status == 429:
            return True


# NEW catalog matchers + interpret-fn (at end of file, before _CATALOG list):
def _is_http_status_error_generic(exc: BaseException) -> bool:
    return _is_httpx_http_status_error(exc)


def _interpret_http_status_error_generic(exc: BaseException) -> dict:
    status = getattr(exc.response, "status_code", None)
    if status is not None and 500 <= status < 600:
        return {
            "severity": "warn",
            "category": "Service",
            "summary": f"The remote service returned a server error (HTTP {status}).",
            "suggested_fix": "Wait a moment and retry; the upstream service may be experiencing issues.",
        }
    return {
        "severity": "error",
        "category": "Network",
        "summary": f"The remote service rejected the request (HTTP {status}).",
        "suggested_fix": "Check your input and API key; if the error repeats, file a bug.",
    }


# At end of _CATALOG list, APPEND ONE TUPLE:
_CATALOG.append(
    (_is_http_status_error_generic, _interpret_http_status_error_generic, lambda _: None)
)
```

This is the entire catalog-side change. Order matters: auth (with the new 401/403 HTTPStatusError check) is checked BEFORE this final entry, so 401/403 land in the `API key` category; 429 lands in `Rate limit`; everything else lands in `Service` (5xx) or `Network` (4xx fallthrough). Existing entries are byte-identical.

## Appendix B — execute_tool boundary cookbook

The exact rewrite at `agent/tools.py:2129-2137` (paraphrased):

```python
# OLD (lines 2129-2137):
    except Exception as e:
        logger.exception("Tool execution failed: %s", tool_name)
        run_fail_plain = _strip_rich_markup(ctx.mode_mgr.active.run_fail)
        return f"{run_fail_plain} Error running {tool_name}: {e}", None, [], []

# NEW:
    except Exception as e:
        logger.debug("Tool execution failed: %s", tool_name, exc_info=True)
        # DEC-ERROR-ROUTING-001..007: catch at boundary, render Rich panel via the
        # canonical interpreter, return [USER_SAW_PANEL] marker + summary line to
        # the LLM. mode.run_fail is consumed by _panel_title in render_interactive
        # (F62 invariant: single authority for failure voice).
        interp = interpret(
            e,
            context={"surface": "agent_execute_tool", "tool": tool_name},
        )
        render_interactive(
            interp,
            ctx.console,
            mode=ctx.mode_mgr.active,
            interactive=False,
        )
        return f"[USER_SAW_PANEL] {render_summary_line(interp)}", None, [], []
```

This is the entire boundary change. The 30 LLM tool dispatch entries above it are byte-identical.

## Appendix C — ToolContext.console cookbook

The exact additive edits at `agent/tools.py::ToolContext`:

```python
# At module top, ENSURE this import is present (verify before adding):
from rich.console import Console

# In ToolContext.__init__ signature, ADD console kwarg LAST:
def __init__(
    self,
    config_dir=None,
    workspace_dir=None,
    hints=None,
    streak_path=None,
    console=None,  # NEW: DEC-ERROR-ROUTING-004
):

# At the END of __init__ body (or near the top — order doesn't matter), ADD:
    self.console: Console = console or Console()
```

In `agent/chat.py`, AFTER the runner is constructed (somewhere around the existing `runner = AgentRunner(...)` line), ADD one line:

```python
runner.ctx.console = console  # DEC-ERROR-ROUTING-004 — propagate chat console
```

(Or, equivalently, find the construction site and pass `console=console` directly. Either approach is acceptable; the implementer chooses whichever is the cleanest one-line edit at the actual construction site in `chat.py`.)
