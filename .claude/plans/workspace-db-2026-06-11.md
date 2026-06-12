# Workspace Clear + Chat Workspace Parity + Enhanced db_status (Phase 17P)

**Workflow:** `w-workspace-db-2026-06-11`
**Branch:** `feature/workspace-db-2026-06-11`
**Worktree:** `/Users/jarocki/src/ap/.worktrees/feature-workspace-db-2026-06-11`
**Planner-time HEAD:** `474a8a6` (Phase 17O closeout, error-routing landed)
**Date:** 2026-06-11
**Complexity tier:** Tier 2 (Standard) — multi-file, parallel surfaces, no schema change.

---

## 1. Problem statement

Two user-visible UX gaps surfaced on the post-17O surface:

1. **`ap chat` workspace command is anemic.** The cmd2 console (`core/console.py::do_workspace`) supports `list / create / switch / delete` subcommands. The chat surface (`agent/chat.py`) only honours `workspace <name>` and interprets the arg as a switch target. That means:
   - `workspace switch foo` tries to switch to a workspace literally named `"switch foo"` (a real bug).
   - `workspace list / create / delete` are not even parsed.
   - There is no `clear` subcommand on either surface.
2. **`do_db_status` "appears to do nothing."** Output is 4–5 rows: active workspace, workspace count, STIX object count, module run count, last run. On a fresh default workspace (every count is 0) the table reads as inert — the user reasonably concluded the command is broken. There is **no chat `db_status` meta-command at all** — parity gap.

Per the dispatch contract, scope intentionally avoids any schema change, dossier touch, gamification touch, and `agent/runner.py` touch.

### Goals (measurable)

- New `WorkspaceManager.clear(name: str | None = None)` zeroes 5 ORM-backed tables (StixObject, Relationship, ModuleRun, ScoreEvent, AnalystNote, BadgeEvent — that's 6 ORM models; relationships count separately per DEC-WORKSPACE-DB-002) for the named (or active) workspace without removing the workspace itself.
- cmd2 `workspace clear [name]` subcommand wired through `_workspace_clear()` with confirmation prompt.
- Chat `workspace` parser rewritten to feature-parity (`list / create <name> / switch <name> / delete <name> / clear [name]`) with bare `workspace` listing.
- Chat `db_status` meta-command added; renders the same enhanced table as cmd2.
- `do_db_status` enhanced: DB file path, file size on disk (humanised), workspace count, **per-table row counts** for the 6 ORM models, total score, last-run / last-note / last-badge timestamps.
- New tests cover `clear()`, cmd2 `workspace clear` (incl. confirmation cancel), chat `workspace` parity, chat `db_status`, enhanced `do_db_status`. Full suite green at ≥ pre-slice baseline + new tests.

### Non-goals

- **No schema change.** No new ORM column, no new table, no Alembic migration (DEC-DB-002 invariant). `src/adversary_pursuit/models/database.py` is BYTEWISE UNCHANGED.
- **No new LLM tool.** Chat surface is sufficient; tool catalog stays at 30 (no `agent/tools.py` edit).
- **No dossier / gamification edits.** `dossier/`, `gamification/` BYTEWISE UNCHANGED.
- **No `agent/runner.py` edit.** C-series byte-invariant preserved.
- **No `error_interpreter.py` or `error_handler.py` edit.** Phase 17O just landed; no regression.
- **No `pyproject.toml` / `uv.lock` edit.** No new dependencies — Python stdlib `os.stat` and `pathlib.Path.stat().st_size` suffice for file size.
- **No `core/config.py` edit.** Confirmation is interactive at UI surface only.

### Unknowns / ambiguities → resolved in §3

All resolved via DEC-WORKSPACE-DB-001..006 below. No user-decision boundary remains; the recommended path from the dispatch contract is adopted everywhere.

### Dominant constraints

- `dossier_state_snapshot` / `_predictions_log` / `_milestone_sentinel` sentinel rows live in `score_events`. Clearing `score_events` deletes them. **This is correct semantics by the user's brief** ("clearing DOES clear dossier state by side effect, which is correct semantics"). Document explicitly in DEC-WORKSPACE-DB-002.
- Confirmation belongs at the UI surface (cmd2 + chat) — the manager method has no `confirm_token` parameter (DEC-WORKSPACE-DB-001). Sacred Practice 5 enforced via post-clear loud assertion.

---

## 2. Architecture decisions (Phase 17P)

### State-authority map

| State domain | Canonical authority | This slice |
|---|---|---|
| Per-workspace row content (stix_objects, relationships, module_runs, score_events, badge_events, notes) | `WorkspaceManager` ORM session operations | **EXTEND** — new `clear()` method as additional authoritative mutator |
| Confirmation gate semantics | cmd2 `_workspace_clear()` + chat `_chat_workspace_clear()` helpers | **NEW WIRING** — manager has no confirm parameter (DEC-WORKSPACE-DB-001) |
| DB file path | `WorkspaceManager._db_path(name)` (private helper, **promoted** to read access via existing public `active` + `_db_path`) | **READ** — db_status calls `_db_path(active)` (treats as effectively public for status display per `_workspace_list` precedent) |
| DB file size on disk | `pathlib.Path.stat().st_size` via NEW `WorkspaceManager.get_workspace_db_size(name: str \| None = None) -> int` public helper | **NEW PUBLIC HELPER** — public API stays in workspace.py; UI code in console.py / chat.py humanises via small local helper |
| Per-table row counts (status display) | NEW `WorkspaceManager.get_workspace_table_counts() -> dict[str, int]` public helper | **NEW PUBLIC HELPER** — returns `{"stix_objects": int, "relationships": int, "module_runs": int, "score_events": int, "badge_events": int, "notes": int}` |
| Last-event timestamps (status display) | NEW `WorkspaceManager.get_last_event_timestamps() -> dict[str, datetime \| None]` public helper | **NEW PUBLIC HELPER** — returns `{"last_run": datetime\|None, "last_note": datetime\|None, "last_badge": datetime\|None, "last_score": datetime\|None}` |
| Schema (ORM models) | `models/database.py` | **READ ONLY** — BYTEWISE UNCHANGED (DEC-DB-002 invariant preserved) |

**Reuse-first principle:** `get_workspace_stats()` already returns `total_indicators`, `module_run_count`, `total_score`, `note_count`. We extend the public API with three new helpers (`get_workspace_db_size`, `get_workspace_table_counts`, `get_last_event_timestamps`) rather than overloading `get_workspace_stats` — preserves the existing badge-evaluation contract byte-for-byte while giving the status surface what it actually needs.

### Decision Log

| ID | Title | Rationale |
|---|---|---|
| `DEC-WORKSPACE-DB-001` | Confirmation lives at UI surface, NOT in `WorkspaceManager.clear()`. The manager method has no `confirm_token` parameter. | Sacred Practice 12 (single authority per domain). `WorkspaceManager` owns mutation; cmd2 / chat own user interaction. Mixing them would mean every future caller (tests, scripts, agent tools) inherits a confirmation parameter that doesn't apply. Cleaner: `clear(name=None)` is a no-argument data mutator; the UI surfaces gate it via `read_input()` / `input()` with default `N`. Tests call `clear()` directly with no token needed. |
| `DEC-WORKSPACE-DB-002` | `clear()` drops rows from 6 ORM models: `StixObject`, `Relationship`, `ModuleRun`, `ScoreEvent`, `AnalystNote`, `BadgeEvent`. Sentinel rows (`_milestone_sentinel`, `_dossier_state_snapshot`, `_predictions_log`) are **included** in the `ScoreEvent` deletion. | The user's brief explicitly affirms this semantics: "the M-4 sentinel rows live in `score_events` which is one of the cleared tables — so clearing DOES clear dossier state by side effect, which is correct semantics." Clearing a workspace's investigation data must reset dossier state — the sentinel pattern (DEC-M4-PERSIST-001, DEC-63-MILESTONE-CATCHUP-001) deliberately reused `score_events` to avoid schema migration, and the consequence is that a workspace clear is a dossier-state clear too. `Relationship` is the 6th table — the brief said "5 tables" but listed `stix_objects, relationships` separately on the next line; treating relationships as the 6th distinct ORM model is mechanically necessary (separate `__tablename__`). |
| `DEC-WORKSPACE-DB-003` | Legacy chat `workspace <name>` shorthand is **deprecated-with-warning** for one release cycle (v0.4.x → removal in v0.5.x). When chat sees `workspace foo` where `foo` is not a known subcommand keyword, it prints a yellow deprecation hint AND still performs the switch. | Per dispatch-contract recommendation (option 3 in "Required decisions"). Hard-breaking the shorthand would surprise existing muscle memory; preserving it forever encodes a parser ambiguity in the surface. One-cycle deprecation is the Code-as-Truth–honouring middle path: visible signal, no silent breakage, removal documented (`DEC-WORKSPACE-DB-003-removal` placeholder for v0.5.x). |
| `DEC-WORKSPACE-DB-004` | DB-file-size humanisation lives in a small local helper `_humanise_bytes(n: int) -> str` in `core/console.py` and is re-imported by chat. NO new module. Returns `"<512 B"`, `"4.2 KB"`, `"12.7 MB"`, `"1.3 GB"` shape. | Public API stays at the manager; UI string-format is a UI concern. One helper, two call sites. Tests verify both sides reuse the same function (no parallel rounding logic). |
| `DEC-WORKSPACE-DB-005` | Chat `db_status` meta-command renders **identical** content to cmd2 `do_db_status`. Both share an internal helper `_render_db_status_table(workspace_mgr, rich_console)` so the rendered Rich Table object is produced by a single code path. | Per dispatch-contract recommendation (option 5 in "Required decisions"). Surface parity is the whole point of this slice; a "summarised for chat" variant would re-create the parity gap one release later. Single render helper lives in `core/console.py` (or a new private module) and is imported by both call sites. |
| `DEC-WORKSPACE-DB-006` | Confirmation default is **`y/N`** (default No). Prompt text: `"Clear all data from workspace '<name>'? This cannot be undone. [y/N]: "`. Empty input or any non-`y`/`yes` (case-insensitive) cancels. On cancel, print `"Cancelled."` and return without mutation. | Loud-fail principle (Sacred Practice 5): the destructive action requires an explicit yes from the user. Default-No mirrors `rm -i`, `git push --force` confirmations, and existing convention. Tests inject input via `monkeypatch.setattr("builtins.input", ...)` / cmd2 `read_input` mock. |
| `DEC-WORKSPACE-DB-007` | `clear()` post-condition is verified by a **loud assertion** inside the method: after `session.commit()`, re-query each of the 6 tables; if any returns a non-zero count, raise `RuntimeError("Workspace clear verification failed: <table>=<count>")`. | Sacred Practice 5 (fail loudly, never silently). The user-visible action is destructive; a silent partial-clear bug would mask a real DB corruption signal. The assertion is cheap (6 scalar count queries on a freshly cleared table) and ensures correctness rather than narrating it. |

### Alternatives gate

Two reasonable approaches differ for the legacy chat-shorthand handling (DEC-WORKSPACE-DB-003). The dispatch contract already recommended option A (deprecate-and-warn) and rejected option B (hard break). No live user gate — the recommendation is adopted as DEC-WORKSPACE-DB-003 with explicit removal placeholder for v0.5.x. If the user disagrees post-landing, the removal-cycle DEC is the natural follow-up boundary.

### Research gate

No new research needed. All five touched modules (`core/workspace.py`, `core/console.py`, `agent/chat.py`, `models/database.py`, tests) were read in full at planner time. No new external dependency, no unfamiliar API, no architectural unknown. SQLAlchemy `session.query(Model).delete()` semantics (commit-required, cascade-aware) are well-understood and used elsewhere in this codebase. `pathlib.Path.stat().st_size` is Python stdlib. Rich `Table.add_row` for the additional rows is identical pattern to existing `do_db_status`.

---

## 3. Wave decomposition + Work Items

| W-ID | Title | Weight | Gate | Deps | Integration |
|---|---|---|---|---|---|
| `W-WORKSPACE-DB-PLAN` | Planner: per-slice plan, scope manifest, evaluation contract, MASTER_PLAN.md amendment | S | none (this doc) | — | `.claude/plans/workspace-db-2026-06-11.md`, `tmp/workspace-db-scope.json`, `tmp/workspace-db-evaluation.json`, `MASTER_PLAN.md` |
| `W-WORKSPACE-DB-IMPL` | Implementer: extend `core/workspace.py` (`clear`, `get_workspace_db_size`, `get_workspace_table_counts`, `get_last_event_timestamps`), extend `core/console.py` (`_workspace_clear`, enhanced `do_db_status`, `_humanise_bytes`, `_render_db_status_table`), rewrite chat `workspace` parser + add chat `db_status`, write 4 new test files, amend MASTER_PLAN.md Phase 17P SAME COMMIT (AP #74 pattern) | L | approve (Guardian land) | `W-WORKSPACE-DB-PLAN` | source + tests |
| `W-WORKSPACE-DB-REVIEW` | Reviewer: execute §6.1 required tests; verify §6.2 git-diff bounds; confirm §6.3 real-path checks; emit `REVIEW_VERDICT=ready_for_guardian` at impl head SHA when green | M | review | `W-WORKSPACE-DB-IMPL` | read-only |
| `W-WORKSPACE-DB-LAND` | Guardian (land): reviewer-readiness preflight; merge `feature/workspace-db-2026-06-11` → `main`; push to `origin/main`; local branch + worktree cleanup | S | approve (auto via Guardian) | `W-WORKSPACE-DB-REVIEW` | git landing |

**Critical path:** plan → impl → review → land. No parallel waves. Max width 1.

---

## 4. Files touched (Implementer scope)

### Allowed / required (modified)

- `src/adversary_pursuit/core/workspace.py` — ADD `clear()` (~30 LOC), `get_workspace_db_size()` (~10 LOC), `get_workspace_table_counts()` (~20 LOC), `get_last_event_timestamps()` (~25 LOC). Total ~85 LOC additive. NO existing method modified.
- `src/adversary_pursuit/core/console.py` — ADD `_workspace_clear()` helper (~25 LOC including confirmation), ADD `_humanise_bytes()` module-level helper (~12 LOC), ADD `_render_db_status_table()` private helper (~50 LOC consolidating new + old fields), REPLACE `do_db_status` body to call `_render_db_status_table()` (preserve docstring + signature), EXTEND `do_workspace` dispatcher (add `elif sub == "clear":` branch). Total ~+90 / -15 LOC net.
- `src/adversary_pursuit/agent/chat.py` — REPLACE lines 161–169 (the `workspace ` startswith branch) with a proper subcommand dispatcher that mirrors cmd2 (`list / create / switch / delete / clear` + legacy single-arg-deprecation), ADD `db_status` / `db status` meta-command branch importing `_render_db_status_table` from `core/console.py`. Total ~+80 / -8 LOC.
- `tests/test_workspace.py` — EXTEND with new `TestWorkspaceClear` class (~6 tests per §6.1), new `TestWorkspaceStatusHelpers` class (~5 tests for the 3 new helpers).
- `tests/test_console.py` — EXTEND with new `TestConsoleWorkspaceClear` class (~3 tests) and `TestConsoleDbStatusEnhanced` class (~4 tests).
- **NEW** `tests/test_chat_workspace_parity.py` — full coverage of chat workspace subcommands + chat `db_status` (~10 tests).
- `MASTER_PLAN.md` — Phase 17P section authoring + Plan Status table row + Active Phase Pointer re-point. Per AP #74 pattern: amended in the SAME implementer commit (not pre-staged).
- `.claude/plans/workspace-db-2026-06-11.md` (this doc; planner stages via `git add`).
- `tmp/workspace-db-scope.json` (planner stages via `git add`).
- `tmp/workspace-db-evaluation.json` (planner stages via `git add`).

### Forbidden (BYTEWISE UNCHANGED)

- `src/adversary_pursuit/models/database.py` — no schema change (DEC-DB-002 + dispatch contract).
- `src/adversary_pursuit/agent/runner.py` — C-series invariant + dispatch invariant.
- `src/adversary_pursuit/agent/tools.py` — no new LLM tool; surface count stays at 30.
- `src/adversary_pursuit/agent/error_handler.py` — Phase 17O contract preserved.
- `src/adversary_pursuit/core/error_interpreter.py` — Phase 17O catalog preserved.
- Entire `src/adversary_pursuit/dossier/` package — forbidden by dispatch contract.
- Entire `src/adversary_pursuit/gamification/` package — forbidden by dispatch contract.
- `src/adversary_pursuit/modules/` — out of scope.
- `pyproject.toml`, `uv.lock` — no dependency change.
- `CLAUDE.md`, `AGENTS.md`, `settings.json`, `hooks/`, `runtime/`, `agents/` — constitution-level files outside this scope.
- `scripts/smoke_test.py` — not affected (clear semantics are interactive-only).
- Every test file EXCEPT the three named above.

---

## 5. State authorities touched

- `workspace_row_mutation_clear` (NEW domain — `WorkspaceManager.clear()` is the sole authority; no module-side or console-side row deletion).
- `workspace_status_display_helpers` (NEW domain — three new public read helpers on `WorkspaceManager`).
- `workspace_clear_confirmation_gate` (NEW domain — cmd2 `_workspace_clear` + chat `_chat_workspace_clear` are the sole UI gates; no manager-side prompting).
- `chat_workspace_command_parser` (EXTEND — chat surface now matches cmd2 dispatcher shape).
- `db_status_render_helper` (NEW domain — `_render_db_status_table` is the single render path; both surfaces import).

---

## 6. Evaluation Contract (W-WORKSPACE-DB-IMPL)

Persisted at `tmp/workspace-db-evaluation.json`. Mirrored here for completeness; runtime authority is the JSON file plus `cc-policy workflow work-item-set ... --evaluation-json`.

### 6.1 Required tests (≥28 new; full suite green at ≥ baseline + new)

**`tests/test_workspace.py::TestWorkspaceClear` (6 new):**

1. `test_clear_active_empty_workspace_is_noop` — `clear()` on a fresh workspace (zero rows everywhere) returns normally, all 6 tables still have 0 rows, DB file still exists.
2. `test_clear_populated_workspace_zeros_six_tables` — seed StixObject + Relationship + ModuleRun + ScoreEvent + AnalystNote + BadgeEvent rows, call `clear()`, assert each `session.execute(select(func.count()).select_from(Model)).scalar() == 0`, assert DB file still exists at `_db_path(active)`.
3. `test_clear_named_non_active_workspace` — `WorkspaceManager.clear(name="other")` clears `other` while active workspace `current` is untouched (its row counts remain).
4. `test_clear_no_active_no_name_raises_runtime_error` — fresh `WorkspaceManager` with no `switch()` call; `clear()` raises `RuntimeError` (no auto-create — that would silently make-then-clear a workspace).
5. `test_clear_missing_named_workspace_raises_value_error` — `clear(name="does-not-exist")` raises `ValueError` matching `f"Workspace '{name}' does not exist"`.
6. `test_clear_drops_dossier_sentinel_rows` — seed `_milestone_sentinel`, `_dossier_state_snapshot`, `_predictions_log` rows in `score_events`, call `clear()`, assert all sentinel rows gone (verifies DEC-WORKSPACE-DB-002 side-effect semantics is intentional).

**`tests/test_workspace.py::TestWorkspaceStatusHelpers` (5 new):**

7. `test_get_workspace_db_size_returns_positive_int` — newly created workspace returns `> 0` bytes (SQLite header alone is ~4 KB).
8. `test_get_workspace_db_size_for_named_workspace` — `get_workspace_db_size(name="other")` returns size for non-active.
9. `test_get_workspace_table_counts_keys_complete` — returns dict with all 6 keys: `stix_objects`, `relationships`, `module_runs`, `score_events`, `badge_events`, `notes`.
10. `test_get_workspace_table_counts_after_inserts` — seed N rows in each table; assert each count matches.
11. `test_get_last_event_timestamps_returns_none_on_empty` — fresh workspace returns `{"last_run": None, "last_note": None, "last_badge": None, "last_score": None}`.

**`tests/test_console.py::TestConsoleWorkspaceClear` (3 new):**

12. `test_do_workspace_clear_no_arg_prompts_and_clears_active` — mock `read_input` to return `"y"`, populate workspace, call `do_workspace("clear")`, assert `clear()` called on active, assert "Cleared" line in output.
13. `test_do_workspace_clear_named_arg_prompts_and_clears` — mock confirm to `"y"`, call `do_workspace("clear other")`, assert `clear(name="other")` called.
14. `test_do_workspace_clear_user_cancels` — mock confirm to `""` (default No) and `"n"`, call `do_workspace("clear")`, assert `clear()` NOT called, "Cancelled." in output.

**`tests/test_console.py::TestConsoleDbStatusEnhanced` (4 new):**

15. `test_do_db_status_contains_db_path_row` — output table has a row labelled "DB file" with an absolute path matching `_db_path(active)`.
16. `test_do_db_status_contains_db_size_row` — output table has a row labelled "DB size" with a humanised size matching `_humanise_bytes()` shape.
17. `test_do_db_status_contains_per_table_counts` — output table has rows for `stix_objects`, `relationships`, `module_runs`, `score_events`, `badge_events`, `notes`.
18. `test_do_db_status_contains_last_event_rows` — populated workspace shows "Last run", "Last note", "Last badge" rows with timestamps; empty workspace shows "(none)".

**NEW `tests/test_chat_workspace_parity.py` (10 new):**

19. `test_chat_workspace_bare_lists` — input `"workspace"` triggers list rendering (Rich Table with workspace names + active marker).
20. `test_chat_workspace_list_explicit` — input `"workspace list"` same as bare.
21. `test_chat_workspace_create_calls_manager` — input `"workspace create foo"` calls `workspace_mgr.create("foo")`.
22. `test_chat_workspace_switch_calls_manager` — input `"workspace switch bar"` calls `workspace_mgr.switch("bar")`.
23. `test_chat_workspace_delete_prompts_then_calls` — input `"workspace delete baz"` prompts confirmation, then on `"y"` calls `workspace_mgr.delete("baz")`.
24. `test_chat_workspace_clear_no_arg_prompts_then_calls_active` — input `"workspace clear"` prompts, on `"y"` calls `clear()`.
25. `test_chat_workspace_clear_with_name_prompts_then_calls_named` — input `"workspace clear qux"` calls `clear(name="qux")` on confirm.
26. `test_chat_workspace_legacy_shorthand_warns_and_still_switches` — input `"workspace mybox"` (where `mybox` is not a subcommand keyword and IS a known workspace) prints yellow deprecation hint AND switches.
27. `test_chat_db_status_renders_enhanced_table` — input `"db_status"` (and `"db status"` alias) renders the same enhanced table as cmd2 `do_db_status`.
28. `test_chat_workspace_unknown_subcommand_shows_usage` — input `"workspace floop"` (no matching subcommand and not a known workspace) prints usage hint.

**Full suite:** `pytest tests/ -q` must remain green at ≥ pre-slice baseline + 28 new tests.

### 6.2 Required evidence (git-diff bounded)

`git diff main` MUST be non-empty for exactly these 7 files:

- `src/adversary_pursuit/core/workspace.py` (additive only — no existing method modified)
- `src/adversary_pursuit/core/console.py` (additive `_workspace_clear` + `_humanise_bytes` + `_render_db_status_table`; modified `do_db_status` body + `do_workspace` dispatcher)
- `src/adversary_pursuit/agent/chat.py` (rewritten `workspace` block + new `db_status` block)
- `tests/test_workspace.py` (extended with 11 new tests)
- `tests/test_console.py` (extended with 7 new tests)
- `tests/test_chat_workspace_parity.py` (NEW, 10 tests)
- `MASTER_PLAN.md` (Phase 17P section + Plan Status row + Active Phase Pointer re-point, SAME commit per AP #74)

`git diff main` MUST be EMPTY for these forbidden files (sample real-path probe checks):

- `src/adversary_pursuit/models/database.py`
- `src/adversary_pursuit/agent/runner.py`
- `src/adversary_pursuit/agent/tools.py`
- `src/adversary_pursuit/agent/error_handler.py`
- `src/adversary_pursuit/core/error_interpreter.py`
- `src/adversary_pursuit/dossier/`
- `src/adversary_pursuit/gamification/`
- `src/adversary_pursuit/modules/`
- `pyproject.toml`, `uv.lock`
- `CLAUDE.md`, `AGENTS.md`, `settings.json`
- `scripts/smoke_test.py`

### 6.3 Required real-path checks (production-sequence verifications)

1. `python -c "from adversary_pursuit.core.workspace import WorkspaceManager; assert hasattr(WorkspaceManager, 'clear')"` — public method exists.
2. `python -c "...; assert hasattr(WorkspaceManager, 'get_workspace_db_size')"` — public helper exists.
3. `python -c "...; assert hasattr(WorkspaceManager, 'get_workspace_table_counts')"` — public helper exists.
4. `python -c "...; assert hasattr(WorkspaceManager, 'get_last_event_timestamps')"` — public helper exists.
5. In-process: instantiate `WorkspaceManager(tmp_path)`, `create("t")`, `switch("t")`, `store_stix_objects([ipv4-stix2-obj], "osint/x", "1.2.3.4")`, then `clear()`, then `get_stix_objects() == []` and `get_module_runs() == []`.
6. `grep -n 'def clear' src/adversary_pursuit/core/workspace.py` — at least one hit at the public method.
7. `grep -n 'def _workspace_clear' src/adversary_pursuit/core/console.py` — at least one hit.
8. `grep -n 'workspace clear' src/adversary_pursuit/agent/chat.py` — chat dispatcher contains the clear branch.
9. `grep -n 'db_status' src/adversary_pursuit/agent/chat.py` — at least one hit (new meta-command).
10. `grep -n '_render_db_status_table' src/adversary_pursuit/core/console.py src/adversary_pursuit/agent/chat.py` — single render helper imported from console.
11. `grep -n 'DEC-WORKSPACE-DB-001\|DEC-WORKSPACE-DB-002\|DEC-WORKSPACE-DB-003' MASTER_PLAN.md` — at least 3 hits (Phase 17P section landed).
12. `grep -n 'Phase 17P' MASTER_PLAN.md` — Plan Status row + closeout section both present.
13. `git diff main --name-only | sort` — equals exactly the 7-file list in §6.2.
14. `python -c "import hashlib, pathlib; print(hashlib.sha256(pathlib.Path('src/adversary_pursuit/models/database.py').read_bytes()).hexdigest())"` — equals pre-slice SHA-256 (proves BYTEWISE UNCHANGED).
15. `python -c "import hashlib, pathlib; print(hashlib.sha256(pathlib.Path('src/adversary_pursuit/agent/runner.py').read_bytes()).hexdigest())"` — equals pre-slice SHA-256.

### 6.4 Required authority invariants

- **DEC-DB-002** (no schema migrations in v1) — `models/database.py` BYTEWISE UNCHANGED. Reviewer verifies via SHA-256 equality.
- **DEC-WS-001** (one SQLite file per workspace, no shared DB) — `clear()` operates within the active workspace's engine only; cross-workspace clear via `name=` argument re-uses the existing `_db_path`/engine pattern without holding two engines simultaneously.
- **DEC-WS-006** (`_ensure_active()` called at top of every public data method) — `clear()` honours this when `name=None`. When `name` is given, it explicitly does NOT auto-create the named workspace (matches `delete()` semantics — `ValueError` if missing).
- **F62 / F64** (mode.run_fail + panel-separation) — not touched; this slice doesn't render error panels.
- **C-series** — `agent/runner.py` BYTEWISE UNCHANGED.
- **Sacred Practice 5** (loud failures) — DEC-WORKSPACE-DB-007 post-clear assertion.
- **Sacred Practice 12** (single authority per state domain) — DEC-WORKSPACE-DB-001 (clear semantics) + DEC-WORKSPACE-DB-005 (single render helper).
- **Phase 17O contract** (DEC-ERROR-INTERPRETER-001..008 + DEC-ERROR-ROUTING-001..007) — preserved BYTEWISE; `error_interpreter.py` + `error_handler.py` + `tools.py::execute_tool` UNCHANGED.

### 6.5 Required integration points

- `core/workspace.py` — 4 new public methods (additive).
- `core/console.py` — `do_workspace` dispatcher extended (additive `elif sub == "clear":` arm), `_workspace_clear` helper NEW, `_humanise_bytes` helper NEW, `_render_db_status_table` helper NEW, `do_db_status` body replaced (delegates to render helper; docstring + signature preserved).
- `agent/chat.py` — workspace branch REPLACED with proper dispatcher, NEW `db_status` branch.
- `tests/test_workspace.py` + `tests/test_console.py` extended; NEW `tests/test_chat_workspace_parity.py`.
- `MASTER_PLAN.md` Phase 17P section + Plan Status row + Active Phase Pointer re-point — SAME implementer commit (AP #74 pattern).
- `.claude/plans/workspace-db-2026-06-11.md`, `tmp/workspace-db-scope.json`, `tmp/workspace-db-evaluation.json` — planner stages via `git add`.

### 6.6 Forbidden shortcuts

- No schema change of any kind (no new column, no new table, no new index, no Alembic).
- No new SQLAlchemy ORM model.
- No edit of `agent/runner.py` (C-series).
- No edit of `dossier/` package, ANY file.
- No edit of `gamification/` package, ANY file.
- No edit of `agent/tools.py` (LLM tool count stays at 30).
- No edit of `agent/error_handler.py` or `core/error_interpreter.py` (Phase 17O preserved).
- No `pyproject.toml` / `uv.lock` change (no new dependency — Python stdlib only).
- No pre-staged MASTER_PLAN.md commit; the Phase 17P amendment lands in the SAME implementer commit as source (AP #74).
- No silent partial clear — DEC-WORKSPACE-DB-007 post-clear assertion is mandatory.
- No `confirm_token` parameter on `WorkspaceManager.clear()` (DEC-WORKSPACE-DB-001).
- No hard break of the legacy chat shorthand — DEC-WORKSPACE-DB-003 deprecation path is mandatory.
- No summarised chat `db_status` variant — DEC-WORKSPACE-DB-005 single-render-helper is mandatory.

### 6.7 Rollback boundary

Single-commit `git revert <impl-sha>` restores pre-slice state byte-for-byte. Effects of revert:

- `WorkspaceManager.clear()` and 3 status helpers vanish (test count drops by 28 new).
- cmd2 `workspace clear` subcommand vanishes (parser falls back to "Unknown workspace subcommand").
- Chat `workspace` returns to its 161–169 single-arg-switch behaviour (the literal `workspace switch foo` bug returns).
- Chat `db_status` meta-command vanishes (parity gap returns).
- `do_db_status` output returns to the 4-row anemic table.
- MASTER_PLAN.md Phase 17P section + Plan Status row vanish; Active Phase Pointer reverts to Phase 17O closeout state.

No schema migration to undo. No new external file or env var to clean up. No new dependency to uninstall.

### 6.8 ready_for_guardian_definition

- All 28 new tests green.
- Full suite green at ≥ pre-slice baseline + 28 new (no flaky regressions).
- All 15 real-path checks in §6.3 succeed.
- Every "MUST be non-empty" diff in §6.2 is non-empty; every "MUST be empty" diff is empty.
- `models/database.py` and `agent/runner.py` SHA-256 hashes equal pre-slice values (BYTEWISE UNCHANGED proof).
- `MASTER_PLAN.md` Phase 17P section + Plan Status row + Active Phase Pointer re-point all present in the implementer commit (SAME commit per AP #74).
- Implementer commit message starts with `feat(workspace):` and references DEC-WORKSPACE-DB-001..007 + Phase 17P.
- `tmp/workspace-db-scope.json` registered into runtime via `cc-policy workflow scope-sync` (or scope-set) at planner→implementer dispatch, byte-identical at landing time.
- `tmp/workspace-db-evaluation.json` registered into runtime via `cc-policy workflow work-item-set wi-workspace-db-impl-01 ... --evaluation-json`.
- Reviewer emits `REVIEW_VERDICT=ready_for_guardian` at implementer head SHA.

---

## 7. Scope Manifest (W-WORKSPACE-DB-IMPL)

Persisted at `tmp/workspace-db-scope.json` (canonical keys `allowed_paths` / `required_paths` / `forbidden_paths` / `authority_domains`). Summary:

- **Allowed / Required paths:**
  - `src/adversary_pursuit/core/workspace.py`
  - `src/adversary_pursuit/core/console.py`
  - `src/adversary_pursuit/agent/chat.py`
  - `tests/test_workspace.py`
  - `tests/test_console.py`
  - `tests/test_chat_workspace_parity.py` (NEW)
  - `MASTER_PLAN.md`
  - `.claude/plans/workspace-db-2026-06-11.md`
  - `tmp/workspace-db-scope.json`
  - `tmp/workspace-db-evaluation.json`
- **Forbidden paths:**
  - `src/adversary_pursuit/models/database.py`
  - `src/adversary_pursuit/agent/runner.py`
  - `src/adversary_pursuit/agent/tools.py`
  - `src/adversary_pursuit/agent/error_handler.py`
  - `src/adversary_pursuit/agent/banner.py`
  - `src/adversary_pursuit/agent/provider_setup.py`
  - `src/adversary_pursuit/agent/repl_input.py`
  - `src/adversary_pursuit/core/error_interpreter.py`
  - `src/adversary_pursuit/core/config.py`
  - `src/adversary_pursuit/core/event_bus.py`
  - `src/adversary_pursuit/core/pivot_policy.py`
  - `src/adversary_pursuit/core/dossier_pivot.py`
  - `src/adversary_pursuit/core/streak.py`
  - `src/adversary_pursuit/core/dossier_report.py`
  - `src/adversary_pursuit/core/graph.py`
  - `src/adversary_pursuit/models/stix.py`
  - Entire `src/adversary_pursuit/dossier/` package
  - Entire `src/adversary_pursuit/gamification/` package
  - Entire `src/adversary_pursuit/modules/` tree
  - `pyproject.toml`, `uv.lock`
  - `CLAUDE.md`, `AGENTS.md`, `settings.json`
  - `hooks/`, `runtime/`, `agents/`
  - `scripts/smoke_test.py`
  - Every test file EXCEPT the three named above.
- **Authority domains touched:**
  - `workspace_row_mutation_clear` (NEW)
  - `workspace_status_display_helpers` (NEW)
  - `workspace_clear_confirmation_gate` (NEW)
  - `chat_workspace_command_parser` (EXTEND)
  - `db_status_render_helper` (NEW)

---

## 8. Out-of-scope (planner asserts; implementer + reviewer honour)

- **Schema migration** to add a dedicated `dossier_state` / `workspace_metadata` / `audit_log` table (deferred to a post-v1 schema-stabilisation slice; DEC-DB-002).
- **LLM tool to expose `clear`** (deliberate — chat surface is sufficient; agent should not be given the ability to wipe a workspace without a human confirmation step).
- **`workspace export` / `workspace copy` / `workspace rename`** (potential follow-ups; not in this slice).
- **`db_status --json`** output mode for scripting (potential follow-up; this slice is interactive Rich Table only).
- **Cross-workspace summary view** (`workspaces` plural — show row counts for all workspaces at once; potential follow-up).
- **Cleanup of the legacy chat shorthand** (DEC-WORKSPACE-DB-003 deprecation path — removal is a v0.5.x follow-up slice).
- **Auto-fix entry for `clear` user-error patterns** in error_interpreter catalog (DEC-ERROR-INTERPRETER-005 mechanically-safe-only invariant — clear has no safe auto-fix).

---

## 9. Subsequent Workflow Cue

After Phase 17P lands, autonomous-continuation decision will be one of:

- `goal_complete` — the two user-reported defects are fixed; no other plan-named follow-ups remain in flight at the close of 17O+17P.
- `next_work_item` — runtime-hygiene follow-ups exist (e.g. `db_status --json`; legacy-shorthand removal in v0.5.x; cross-workspace summary). The post-landing planner pass evaluates against the plan and chooses.
- `needs_user_decision` — only if the User reserves the post-17P direction choice (v0.4.x release-discipline vs. continued UX hygiene vs. v2 scoping).

This plan does NOT pre-commit; the post-landing planner pass owns the decision.
