# MySQL Export Privilege Fallback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Continue a denied MySQL parallel Export through an explicit one-connection consistent-snapshot fallback chosen by the user.

**Architecture:** Rust Core owns both snapshot protocols and emits a stable privilege-required marker only for server access-denial codes at the two privileged statements. Python forwards an explicit snapshot mode, while the PyQt dialog owns the three-action user decision and starts a fresh worker for the same Export in one-connection mode.

**Tech Stack:** Rust 2021, `mysql` crate 26, JSONL Rust Core protocol, Python 3.9+, PyQt6, pytest.

## Global Constraints

- Keep `parallel_strict` as the default and never silently downgrade.
- `single_connection` uses exactly one read-only InnoDB consistent-snapshot connection and requests no global/backup lock.
- Preserve Rust Core ownership of all DB operations.
- Do not expose credentials, host details, or raw unbounded server diagnostics in new UI copy.
- Preserve current cancellation and TunnelForge-owned output-directory safety.

---

### Task 1: Rust privilege classification and snapshot mode parsing

**Files:**
- Modify: `migration_core/src/dump.rs`
- Test: `migration_core/src/dump.rs`

**Interfaces:**
- Produces: `MYSQL_PARALLEL_SNAPSHOT_PRIVILEGE_REQUIRED` stable marker.
- Produces: `MysqlSnapshotMode::{ParallelStrict, SingleConnection}` parsed from `payload.mysql_snapshot_mode`.
- Produces: `mysql_parallel_privilege_error(&mysql::Error, &str) -> Option<String>`.

- [ ] **Step 1: Write failing Rust tests**

Add unit tests that construct `mysql::Error::MySqlError` values with literal
codes 1045, 1142, 1205, and 1227. Assert that 1045/1142/1227 return a marker
containing the supplied privilege and that 1205 returns `None`. Add request
fixtures proving the default mode is `ParallelStrict`, `single_connection`
parses, and an unknown value is rejected.

- [ ] **Step 2: Run the tests and verify RED**

Run:

```powershell
cargo test --manifest-path migration_core\Cargo.toml mysql_snapshot_mode --lib
cargo test --manifest-path migration_core\Cargo.toml mysql_parallel_privilege --lib
```

Expected: compilation/test failure because the enum, parser field, and helper
do not exist.

- [ ] **Step 3: Implement minimal classification and parsing**

Define the marker constant and enum in `dump.rs`. Extend `DumpRunOptions` with
`mysql_snapshot_mode`. Parse only `parallel_strict` and `single_connection`,
defaulting to `parallel_strict`. Match `mysql::Error::MySqlError(error)` and
classify only codes 1045, 1142, and 1227.

- [ ] **Step 4: Run the focused tests and verify GREEN**

Run the two Task 1 commands and require zero failures.

### Task 2: Rust one-connection snapshot execution

**Files:**
- Modify: `migration_core/src/dump.rs`
- Modify: `migration_core/src/adapters.rs`
- Test: `migration_core/src/dump.rs`
- Test: `migration_core/src/adapters.rs`

**Interfaces:**
- Consumes: `MysqlSnapshotMode`.
- Produces: `MysqlSharedSnapshot::acquire_single_connection(&Endpoint)`.
- Produces: manifest/result policy `mysql_single_connection_consistent_snapshot`.

- [ ] **Step 1: Write failing execution-contract tests**

Add tests proving the one-connection SQL list contains only:

```text
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
START TRANSACTION WITH CONSISTENT SNAPSHOT, READ ONLY
```

Add metadata tests asserting `single_connection` produces
`mysql_single_connection_consistent_snapshot`, `strict_export=true`, and no
warnings. Add a schema comparison test showing a changed table definition is
reported as DDL drift.

- [ ] **Step 2: Run the tests and verify RED**

Run:

```powershell
cargo test --manifest-path migration_core\Cargo.toml mysql_single_connection --lib
cargo test --manifest-path migration_core\Cargo.toml mysql_schema_drift --lib
```

Expected: failure because the fallback acquisition, metadata policy, and drift
helper do not exist.

- [ ] **Step 3: Implement the minimal one-connection path**

Make the snapshot coordinator optional. Add one-connection acquisition that
opens one worker and starts the existing read-only transaction SQL without
executing either privileged statement. In `dump_run`, force effective worker
count to one for this mode, validate InnoDB using that worker, reuse
`dump_tables_global_mysql`, re-inspect and compare the selected normalized
schema after extraction, and emit the distinct policy.

At the strict lock calls, convert only classified access-denial errors into:

```text
MYSQL_PARALLEL_SNAPSHOT_PRIVILEGE_REQUIRED:<privilege>:<sanitized server error>
```

Keep actual timeout and other failures on their existing paths.

- [ ] **Step 4: Run focused and Rust regression tests**

Run:

```powershell
cargo test --manifest-path migration_core\Cargo.toml mysql_ --lib
cargo test --manifest-path migration_core\Cargo.toml --lib
```

Require zero failures.

### Task 3: Python payload and worker propagation

**Files:**
- Modify: `src/exporters/rust_dump_exporter.py`
- Modify: `src/ui/workers/rust_dump_worker.py`
- Test: `tests/test_rust_dump_exporter.py`
- Test: `tests/test_rust_dump_worker.py`

**Interfaces:**
- Produces: optional `mysql_snapshot_mode: str = "parallel_strict"` on both Export methods.
- Produces: `is_mysql_parallel_snapshot_privilege_error(message: object) -> bool`.

- [ ] **Step 1: Write failing Python tests**

Add exporter tests asserting `single_connection` is forwarded as
`payload["mysql_snapshot_mode"]`, forces `payload["threads"] == 1`, and the
success message names a one-connection snapshot. Add worker tests asserting
the worker forwards its `mysql_snapshot_mode` kwarg. Add a literal marker test
that accepts the stable marker and rejects a normal `ERROR 1045` connection
failure without the marker.

- [ ] **Step 2: Run the tests and verify RED**

Run:

```powershell
.\.venv\Scripts\python.exe -m pytest tests\test_rust_dump_exporter.py tests\test_rust_dump_worker.py -q -k "snapshot_mode or privilege_error"
```

Expected: signature/assertion failures because the mode and marker helper do
not exist.

- [ ] **Step 3: Implement minimal propagation**

Add the optional mode at the end of public method signatures, normalize it to
the two accepted literals, force one thread only for `single_connection`, and
include the payload field. Preserve the Rust marker in the returned failure
message. Forward the worker kwarg by keyword.

- [ ] **Step 4: Run focused Python tests and verify GREEN**

Run the Task 3 pytest command and require zero failures.

### Task 4: PyQt three-action fallback flow

**Files:**
- Modify: `src/ui/dialogs/db_export_dialog.py`
- Test: `tests/test_db_export_dialog.py`
- Test: `tests/test_i18n.py`

**Interfaces:**
- Consumes: `is_mysql_parallel_snapshot_privilege_error`.
- Produces: `_create_mysql_privilege_dialog() -> QMessageBox`.
- Produces: `_retry_with_single_connection() -> None`.

- [ ] **Step 1: Write failing UI tests**

Add an offscreen dialog test asserting the custom message box has buttons
`단일 연결로 계속`, `권한 설정 안내`, and `취소`, with continuation as the
default. Add `on_finished` tests that replace the prompt method with literal
actions and assert:

- continuation starts a new worker with `threads=1` and
  `mysql_snapshot_mode="single_connection"`;
- guidance shows the exact two privilege groups and does not start a worker;
- cancel does not start a worker;
- unmarked errors retain the existing warning/reporting path;
- a fallback failure never prompts again.

- [ ] **Step 2: Run the UI tests and verify RED**

Run:

```powershell
.\.venv\Scripts\python.exe -m pytest tests\test_db_export_dialog.py -q -k "privilege or single_connection"
```

Expected: failure because the dialog factory, action handling, and retry method
do not exist.

- [ ] **Step 3: Implement the minimal dialog flow**

Track whether the current attempt is the fallback. Extract existing worker
signal wiring into one helper. Before generic failure handling, detect only the
stable marker, show the custom dialog, and either start the one-connection
worker, show concise DBA guidance, or stop. Do not submit an anonymous error
report for this expected capability branch.

- [ ] **Step 4: Run focused UI and translation tests**

Run:

```powershell
.\.venv\Scripts\python.exe -m pytest tests\test_db_export_dialog.py tests\test_i18n.py -q
```

Require zero failures.

### Task 5: Status and regression verification

**Files:**
- Modify: `docs/current_status.md`
- Modify: `tests/test_current_status_docs.py`

**Interfaces:**
- Consumes: all completed behavior.
- Produces: TF-STATUS-096 status and fresh verification evidence.

- [ ] **Step 1: Update canonical status**

Record the implementation, RED/GREEN commands, focused/full results, remaining
live-provider verification, and use `fixed_pending_full_verify` unless the
complete Python/Rust gates and a representative live MySQL fallback pass in
this session.

- [ ] **Step 2: Run final gates**

Run:

```powershell
.\.venv\Scripts\python.exe -m pytest tests\test_rust_dump_exporter.py tests\test_rust_dump_worker.py tests\test_db_export_dialog.py tests\test_i18n.py tests\test_current_status_docs.py -q
cargo test --manifest-path migration_core\Cargo.toml
cargo build --manifest-path migration_core\Cargo.toml --release
git diff --check
```

Report exact pass/fail counts. Do not claim live managed-provider success
without running against that provider.
