---
name: aptabase-analytics-report
description: Export and analyze Vibe Aptabase analytics for recent time windows or existing CSV exports, with emphasis on failure rates, affected users, app versions, and OS breakdowns. Use when Codex needs to: (1) export recent analytics with `scripts/export_analytics.py`, (2) filter an export to the exact last N hours, (3) restrict analysis to specific `app_version` values, (4) quantify the impact of the current failure-related events defined in the repository, or (5) produce a concise reliability report with fix suggestions.
---

# Aptabase Analytics Report

## Overview

Use this skill to export Aptabase telemetry from the Vibe repo, trim day-granularity exports to an exact time window, and produce a reliability report from the CSV with inline `uv` + pandas commands.

## Workflow

### 1. Confirm the event set

Read these files before interpreting the export:

- `desktop/src-tauri/src/analytics.rs`
- `desktop/src/lib/analytics.ts`

Treat those files as the source of truth for the current event set. Identify:

- lifecycle start events
- lifecycle success events
- lifecycle failure events
- startup or infrastructure failure events

Do not assume specific event names are stable across revisions.

### 2. Export enough UTC days for the requested window

`scripts/export_analytics.py` accepts day-granularity dates, `start` inclusive and `end` exclusive. To analyze the last N hours, export the full UTC days that overlap the window, then filter afterward.

For the last 48 hours:

```bash
python3 - <<'PY'
from datetime import datetime, timedelta, timezone
now = datetime.now(timezone.utc)
print((now - timedelta(hours=48)).strftime("%Y-%m-%d"))
print((now + timedelta(days=1)).strftime("%Y-%m-%d"))
PY
```

Then export:

```bash
env UV_CACHE_DIR="$PWD/.uv-cache" uv run scripts/export_analytics.py \
  --start-date YYYY-MM-DD \
  --end-date YYYY-MM-DD \
  --output scripts/analytics_last_48h_raw.csv
```

If `uv` cannot run inside the sandbox, move the cache into the workspace (`UV_CACHE_DIR`) and, if needed, request to run outside the sandbox so the script can authenticate and fetch data.

### 3. Analyze with inline pandas every time

Do not rely on a bundled analysis script. Use ad hoc inline Python with `uv` so the analysis stays easy to adapt as the repo changes.

Start with a minimal trim and version filter:

```bash
env UV_CACHE_DIR="$PWD/.uv-cache" uv run --with pandas==3.0.0 python - <<'PY'
from datetime import timedelta
import pandas as pd

df = pd.read_csv("scripts/analytics_last_48h_raw.csv")
df["timestamp"] = pd.to_datetime(df["timestamp"], utc=True)

end_time = df["timestamp"].max()
cutoff = end_time - timedelta(hours=48)
filtered = df[df["timestamp"] >= cutoff].copy()
filtered = filtered[filtered["app_version"].astype(str).isin({"3.0.14", "3.0.15"})]
filtered = filtered.sort_values("timestamp", ascending=False).reset_index(drop=True)
filtered["timestamp"] = filtered["timestamp"].dt.strftime("%Y-%m-%d %H:%M:%S")
filtered.to_csv("scripts/analytics_last_48h_versions.csv", index=False)

print(f"rows={len(filtered)}")
print(f"time_min={filtered['timestamp'].min()}")
print(f"time_max={filtered['timestamp'].max()}")
PY
```

Then run additional inline pandas passes as needed. Prefer several small targeted scripts over one large script:

- one pass for headline counts and global rates
- one pass for affected users and happy users
- one pass for OS, OS version, and app version breakdowns
- one pass for parsing `string_props` / `numeric_props` and clustering error messages

Keep iterating until the causes are clear. The skill should adapt the analysis to the actual export contents, not force one fixed taxonomy.

### 4. Interpret the report

Prioritize these metrics:

- Global raw error rate: `(current failure-class events) / all events`
- Primary workflow failure rate: `(current failure-class events) / (matching start-class events)` when the lifecycle is present
- Affected users: unique `user_id` values with any failure-class event
- Happy users: users with at least one success-class event and no failure-class events
- OS risk: failure rate and affected-user share by OS family and OS name

Separate user-input mistakes from runtime, environment, and packaging failures. A small number of users can generate many repeated input errors and distort the raw event-level error rate.

### 5. Suggest fixes from the observed buckets

Build buckets from the current data, then map the high-volume failures to code paths before proposing changes:

- `desktop/src-tauri/src/sona.rs`: spawn, load-model, and transcribe HTTP failures
- `desktop/src-tauri/src/cmd/mod.rs`: input validation and surfaced transcription errors

Read `references/error-buckets.md` for the general workflow on discovering the current event set and deriving buckets from the current repository state.

## Output Expectations

Keep the final report compact but complete:

- State the exact time range, filters, row count, and active users
- Report event counts and version counts
- Separate raw error rate from runtime-like error rate
- Give affected-user counts, happy-user counts, and user share percentages
- Break down failures by OS family, OS name, app version, and top issue buckets
- End with prioritized fix suggestions tied to likely code paths

## Resources

### references/error-buckets.md

Read this file when you need a general checklist for discovering event names, deriving buckets from the current data, and tying findings back to the right code.
