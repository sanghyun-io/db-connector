# MySQL Export Privilege Fallback Design

## Goal

Let a MySQL user finish an Export without `FLUSH_TABLES`/`RELOAD` or
`BACKUP_ADMIN` while keeping the existing strict parallel snapshot as the
preferred path.

## User Flow

When the selected parallel Export is denied at either backup-lock step,
TunnelForge stops that attempt before table data is written and shows:

> 병렬 Export에 필요한 권한이 없습니다. 권한 없이 단일 연결로 안전하게
> Export할 수 있지만 병렬 처리는 사용되지 않습니다.

The dialog offers exactly three actions:

1. `단일 연결로 계속` — recommended; retry the same Export immediately.
2. `권한 설정 안내` — show the exact required privileges and retry guidance.
3. `취소` — leave the Export stopped.

TunnelForge never grants privileges and never silently downgrades.

## Rust Core Contract

`dump.run` accepts an explicit MySQL snapshot mode:

- `parallel_strict` (default): the current shared-snapshot protocol using the
  brief global read lock and instance backup lock.
- `single_connection`: one read-only `REPEATABLE READ` transaction started
  with `WITH CONSISTENT SNAPSHOT`; no global or backup lock is requested.

Access denial at `FLUSH TABLES WITH READ LOCK` or
`LOCK INSTANCE FOR BACKUP` returns a stable machine-readable error marker and
the applicable privilege name. Other errors, including real lock timeouts,
remain ordinary failures and do not trigger the privilege dialog.

The single-connection mode:

- uses one fixed MySQL connection for all table data reads regardless of the
  originally selected worker count;
- records a distinct snapshot policy in the manifest/result;
- accepts only InnoDB tables;
- compares the selected schema before and after data extraction and fails the
  Export if concurrent DDL changes its structure.

## Python and UI Contract

The Python exporter forwards snapshot mode without parsing raw MySQL text.
The worker returns the stable Rust marker to the dialog. The dialog recognizes
only that marker, asks the user, and on continuation starts a fresh worker for
the same schema, table selection, compression, and output directory with
`single_connection`.

The continuation attempt is visibly labeled as a one-connection compatibility
Export. It does not show the privilege prompt again.

## Failure and Cleanup

- The initial denied attempt produces no table data.
- Reusing its TunnelForge-owned output directory is allowed.
- A failed fallback remains a normal Export failure with its real reason.
- Non-InnoDB selections are rejected with the table/engine list.
- Cancellation behavior remains unchanged.

## Testing

- Rust unit tests classify only access-denial codes as missing capability.
- Rust protocol/live coverage verifies one fixed transaction, distinct
  manifest metadata, no backup-lock SQL, and DDL-drift failure.
- Python exporter tests verify payload forwarding and marker preservation.
- PyQt tests verify the three actions, recommended continuation, exact retry
  arguments, no silent fallback, and no repeated prompt.
- Existing strict parallel snapshot tests remain unchanged.
