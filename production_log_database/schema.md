# Production Log Analysis DB Schema (PostgreSQL)

This container stores user/role access control, uploads, analysis runs, parsed events and signatures, findings, recommendations, and final structured reports.

**Important:** Per container standards, schema DDL is applied via `psql -c "SQL"` statements (one at a time). This file is documentation of what is currently created in the database.

## Extensions

- `pgcrypto` for `gen_random_uuid()`
- `citext` for case-insensitive emails

## Tables

### `roles`
RBAC roles.

Columns:
- `id uuid PK`
- `name text UNIQUE NOT NULL`
- `description text`
- `created_at timestamptz NOT NULL DEFAULT now()`

### `users`
Application users.

Columns:
- `id uuid PK`
- `email citext UNIQUE NOT NULL`
- `password_hash text NOT NULL`
- `display_name text`
- `is_active boolean NOT NULL DEFAULT true`
- `created_at timestamptz NOT NULL DEFAULT now()`
- `updated_at timestamptz NOT NULL DEFAULT now()` (maintained via trigger)
- `last_login_at timestamptz`

### `user_roles`
Many-to-many between users and roles.

Columns:
- `user_id uuid FK -> users(id) ON DELETE CASCADE`
- `role_id uuid FK -> roles(id) ON DELETE CASCADE`
- `granted_at timestamptz NOT NULL DEFAULT now()`
- `granted_by uuid NULL FK -> users(id) ON DELETE SET NULL`
- `PRIMARY KEY (user_id, role_id)`

### `uploads`
Uploaded log artifacts.

Columns:
- `id uuid PK`
- `user_id uuid NULL FK -> users(id) ON DELETE SET NULL`
- `original_filename text NOT NULL`
- `storage_path text` (path/URI used by backend storage subsystem)
- `content_type text`
- `file_size_bytes bigint`
- `file_sha256 text` (optional dedupe/integrity)
- `received_at timestamptz NOT NULL DEFAULT now()`
- `source_system text`
- `environment text`
- `metadata jsonb NOT NULL DEFAULT '{}'::jsonb`

Indexes:
- `idx_uploads_received_at (received_at)` for time-range filtering.

### `analysis_runs`
Each analysis execution performed on an upload.

Columns:
- `id uuid PK`
- `upload_id uuid NOT NULL FK -> uploads(id) ON DELETE CASCADE`
- `triggered_by uuid NULL FK -> users(id) ON DELETE SET NULL`
- `status text NOT NULL` (e.g., queued/running/succeeded/failed)
- `started_at timestamptz NOT NULL DEFAULT now()`
- `finished_at timestamptz`
- `tool_version text`
- `parser_name text`
- `parameters jsonb NOT NULL DEFAULT '{}'::jsonb`
- `summary jsonb NOT NULL DEFAULT '{}'::jsonb`
- `error_message text`

Indexes:
- `idx_analysis_runs_upload_started_at (upload_id, started_at DESC)`
- `idx_analysis_runs_status_started_at (status, started_at DESC)`

### `event_signatures`
Canonical signatures/patterns used to group similar events.

Columns:
- `id uuid PK`
- `signature_key text UNIQUE NOT NULL` (stable external identifier)
- `title text NOT NULL`
- `category text`
- `severity text`
- `description text`
- `pattern jsonb NOT NULL DEFAULT '{}'::jsonb` (pattern definition: regex, fields, etc.)
- `created_at timestamptz NOT NULL DEFAULT now()`
- `updated_at timestamptz NOT NULL DEFAULT now()` (maintained via trigger)

### `parsed_events`
Normalized, queryable events parsed from a log file for a given run.

Columns:
- `id uuid PK`
- `run_id uuid NOT NULL FK -> analysis_runs(id) ON DELETE CASCADE`
- `upload_id uuid NOT NULL FK -> uploads(id) ON DELETE CASCADE`
- `event_ts timestamptz` (timestamp from log line; can be NULL if unknown)
- `ingested_at timestamptz NOT NULL DEFAULT now()`
- `severity text`
- `facility text`
- `component text`
- `host text`
- `service text`
- `pid int`
- `request_id text`
- `message text`
- `raw_line text`
- `structured jsonb NOT NULL DEFAULT '{}'::jsonb` (parsed structured fields)
- `signature_id uuid NULL FK -> event_signatures(id) ON DELETE SET NULL`
- `signature_confidence real`
- `line_number int`
- `byte_offset bigint`

Indexes (time-range + run-based):
- `idx_parsed_events_run_ts (run_id, event_ts)`
- `idx_parsed_events_upload_ts (upload_id, event_ts)`
- `idx_parsed_events_signature (signature_id, event_ts)`

### `findings`
Detected issues/insights produced by a run (errors, anomalies, root causes).

Columns:
- `id uuid PK`
- `run_id uuid NOT NULL FK -> analysis_runs(id) ON DELETE CASCADE`
- `upload_id uuid NOT NULL FK -> uploads(id) ON DELETE CASCADE`
- `severity text NOT NULL`
- `title text NOT NULL`
- `description text`
- `root_cause text`
- `evidence jsonb NOT NULL DEFAULT '{}'::jsonb`
- `related_signature_id uuid NULL FK -> event_signatures(id) ON DELETE SET NULL`
- `first_seen_at timestamptz`
- `last_seen_at timestamptz`
- `occurrences int NOT NULL DEFAULT 1`
- `created_at timestamptz NOT NULL DEFAULT now()`

Indexes:
- `idx_findings_run_severity (run_id, severity)`
- `idx_findings_time_range (first_seen_at, last_seen_at)`

### `recommendations`
Recommended troubleshooting steps linked to a finding.

Columns:
- `id uuid PK`
- `finding_id uuid NOT NULL FK -> findings(id) ON DELETE CASCADE`
- `run_id uuid NOT NULL FK -> analysis_runs(id) ON DELETE CASCADE`
- `upload_id uuid NOT NULL FK -> uploads(id) ON DELETE CASCADE`
- `priority int NOT NULL DEFAULT 3` (1 highest, larger is lower)
- `title text NOT NULL`
- `steps jsonb NOT NULL DEFAULT '[]'::jsonb` (array of step objects/strings)
- `reference_links jsonb NOT NULL DEFAULT '[]'::jsonb` (renamed from reserved word `references`)
- `created_at timestamptz NOT NULL DEFAULT now()`

Indexes:
- `idx_recommendations_finding (finding_id)`

### `structured_reports`
Final structured report artifact (JSON) for a run.

Columns:
- `id uuid PK`
- `run_id uuid NOT NULL UNIQUE FK -> analysis_runs(id) ON DELETE CASCADE`
- `upload_id uuid NOT NULL FK -> uploads(id) ON DELETE CASCADE`
- `created_at timestamptz NOT NULL DEFAULT now()`
- `report_version text`
- `schema_name text`
- `report jsonb NOT NULL`

Indexes:
- `idx_structured_reports_upload_created_at (upload_id, created_at DESC)`

## Triggers

A shared trigger function keeps `updated_at` current on updates:

- `trg_users_updated_at` on `users`
- `trg_event_signatures_updated_at` on `event_signatures`

## Seed data

Default roles are seeded (idempotently):
- `admin`
- `analyst`
- `viewer`
