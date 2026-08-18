## What was broken

The root cause was a race in the ingestion flow: the service checked `events` for an `event_id` and then inserted, which is a classic read-then-insert race. With concurrent retries, two requests could both see “missing” and both write a row, which created duplicate calls and inflated the `account_stats` total. The async recording job was also tied to the HTTP request context, so when the request returned the context was canceled and the work was silently abandoned, which matches the “recordings never get marked processed” symptom after deploys.

The in-memory `stats.Cache` also updated without locking, so reads could race with writes and produce inconsistent totals when the service was under load.

## Why this deduplication strategy

I used Postgres as the source of truth and made the dedupe atomic with `INSERT ... ON CONFLICT (event_id) DO NOTHING RETURNING event_id`. That guarantees that only one delivery wins for a given `event_id`, even under concurrency, while still allowing retries to return an idempotent success. Redis was available, but the stable event ID is already a uniquely indexed database key, so the database gives stronger correctness with less operational complexity than a separate redis lock or cache. Redis would be a useful optimization for short-lived dedupe windows, but not the primary correctness boundary.

## If we needed 10,000 webhooks/sec

I would keep the same database-backed idempotency boundary, but move the hot path to a transactional write model with a single `INSERT` and minimal downstream work. I would also decouple processing more aggressively: use a durable queue or worker pool for recording jobs, persist job state explicitly, and make the cache a sharded or Redis-backed aggregate rather than a single in-memory map. At that scale, I would also add observability around duplicate suppression, queue lag, and worker retries so we can distinguish true provider retries from application-level bugs.
