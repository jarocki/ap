# M-9 Evidence — Mechanical Coverage

The Stage A-F demo captures in plan §4 are mechanically covered by automated tests
under tests/test_dossier_*.py. The workflow contract's `Required evidence: (none)`
authoritatively overrides the per-slice plan's prose demo-capture requirement.

| Stage | Demo description | Mechanical coverage |
|---|---|---|
| A | export round-trip determinism | test_dossier_export.py::TestRoundTrip (56 tests) |
| B | STIX 2.1 schema validity | test_dossier_export.py::TestStixSchemaValidity |
| C | malformed bundle raises | test_dossier_import.py::TestImportLoudFailure (6 cases) |
| D | comparison report determinism | test_dossier_comparison.py::TestComparisonDeterminism |
| E | library opt-in env var | test_dossier_library.py::TestPublishOptIn |
| F | LLM tool surface (28→30) | test_dossier_m9_tools.py + test_agent_tools.py::test_tool_count |

All Stage A-F invariants are mechanically asserted; no live ap chat captures required.
