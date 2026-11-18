# Job status and progress

`GET /api/jobs/<id>` returns the state of a background job (`embed`, `extract`,
`silent_witness`, `verify_tx`).

```json
{
  "ok": true,
  "job": {
    "id": 12,
    "type": "embed",
    "status": "running",
    "result": null,
    "created_at": "2026-10-18T16:00:00+00:00",
    "heartbeat_at": "2026-10-18T16:00:30+00:00",
    "progress": { "phase": "embedding", "percent": 42.5 }
  }
}
```

## `progress`

| Value | Meaning |
| --- | --- |
| `null` | No progress reported yet (pending, or a job type that does not report). Never an error. |
| `{"phase": string, "percent": number}` | Current step and its completion, 0-100, one decimal. |
| `{"phase": string, "percent": null}` | Current step has no measurable fraction. |

Phases reported today:

- `embed`: `hashing_source` -> `embedding` (frames written / total frames; `percent`
  is `null` when ffprobe reports no frame count) -> `hashing_output` -> `recording`.
- `silent_witness`: `generating_proof` (the proof runs as one external command, so
  only the phase name is reported).

Processors call `db.report_job_progress(job_id, phase, percent)` as often as they
like; the value is stored at most once per second per job
(`JOB_PROGRESS_MIN_INTERVAL_SECONDS`), and only while the job is `running`.
Heartbeats extend the lease only and never overwrite progress.

## Worker recovery plan (read-only)

After an unclean shutdown, `build_recovery_plan()` (also exposed as
`worker_recovery_plan`, `plan_recovery`, `get_recovery_plan`, ...) inspects the
in-memory job queue and on-disk job artifacts and returns a read-only plan
listing abandoned leases and partial artifacts with explicit uncertainty.
It never mutates job state or deletes files. Each plan carries a bounded
`actions` list (default 10, configurable via `max_actions`).

```python
from worker import build_recovery_plan

plan = build_recovery_plan(now=..., max_actions=5)
assert plan["read_only"] is True
assert len(plan["actions"]) <= 5
```

## Claim generations (stale-worker fencing)

Every lease and every retryable release (`fail_job(..., is_fatal=False)`, used
for example when the shutdown grace period expires) increments the job's
internal `_claim_generation`. The worker passes the generation it leased to
`complete_job`, `fail_job`, `report_job_progress` and `heartbeat_job`
(`generation=`); a write from an older generation is ignored, returns `False`
and logs `job_claim_stale`. This keeps a worker that lost its claim from
overwriting a newer result, progress or failure state. Calls without
`generation` behave as before. The field is internal and never part of the
response above.

## What the response never contains

The job's request `payload` (filenames, metadata, proof secrets) and internal
underscore-prefixed fields (trace links, throttle state) are stripped by
`public_job_view` before the job is returned. Progress values are phase names
and numbers only and carry no payload data.

## Failure diagnostics

When a job fails, the exception text and traceback frames are sanitized before
they reach the logger or the job-failure record:

- Private filesystem paths are replaced with `<path>`.
- Secret-shaped values (tokens, keys, passwords, `Bearer` credentials) are
  replaced with `<redacted>`.
- Each diagnostic string is capped at a fixed byte budget and truncated with a
  `…[truncated]` marker when it exceeds it.

Sanitized diagnostics are stored in the job's `result` error fields; the raw
exception is never persisted.
