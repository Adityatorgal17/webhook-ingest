## What was broken

The first problem was the dedupe path. We were checking whether an `event_id` existed and then inserting the row. That is a race, plain and simple. If two requests hit at the same time, both can see the same state and both can insert. That is how duplicate rows started showing up, and why the call counts drifted higher than they should.

The second issue was the stats cache. It had a mutex, but the actual write path never used it. So multiple goroutines could update the same account at the same time and lose increments. That is why the in-memory totals and the durable totals could get out of sync.

The third thing was the recording job. It was launched with the same request context as the webhook handler. Once the request finished, or the process restarted during deploy, that context got canceled and the job got dropped. That matches the symptom where calls came in but the recordings were never marked processed.

The fourth issue was partial writes. The service was writing to `events`, then `calls`, then `account_stats` in separate steps. If one of the later writes failed, the earlier ones stayed committed. That left the system in a partially updated state, which is exactly the kind of thing that makes retries dangerous.

## Why I fixed it this way

I made Postgres the source of truth for deduplication. The `event_id` column is unique, and the insert uses `ON CONFLICT (event_id) DO NOTHING`. That means the first successful write wins, and any duplicate or retried webhook becomes a no-op.

That is a much safer approach than doing a raw `SELECT` and then insert in application code. The check and the insert are not atomic, so it can still race. The database-level uniqueness is the point where the system actually guarantees correctness.

I also locked the cache properly and moved the background recording work onto a fresh context so it does not die when the HTTP request is done. These were not cleanup changes; they were fixes for real correctness issues.

## Why this is better than the alternatives

Redis could have been used for a dedupe key, but that would have been weaker. It adds more moving parts, it needs expiry logic, and it is not the durable source of truth. A pure app-level check is not enough because it still has the race.

The DB uniqueness check is the cleanest option because it enforces the right behavior where the data actually lives. It keeps retries safe without adding extra state machines or fragile timing assumptions.

## If this needed to handle a much bigger load

If we were pushing 10,000 webhooks per second, I would keep the same correctness model but move the slow work out of the request path. I would still keep the unique `event_id` constraint as the idempotency boundary, but I would move recording processing onto a durable queue and worker pool. That keeps the webhook endpoint fast and makes the async work retryable without tying it to an HTTP request lifecycle.

The main idea stays the same: correctness should be enforced at the transactional boundary, not in a best-effort application check.
