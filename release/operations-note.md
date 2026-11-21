# Fixture operations note

Owner: eugenelim (repository owner and AWS sandbox owner). Recorded 2026-11-15.

## Target and prior state
- The only effect is the object `releases/current.json` in a target bucket created on 2026-11-15 for this fixture alone. No other system reads or writes it.
- The bucket policy denies every S3 action on that key to every principal except the deploy role, so only this workflow can create it.
- The bucket is unversioned and expires objects after one day.
- Before each run the key is absent, or it holds a prior run's bytes marked with that run's admission. Overwriting it loses no state another system needs.

## Recovery
- A failed, cancelled or uncertain run is reconciled before any retry. The agent observes the key and runs `github_release.py reconcile` with the run, jobs, observation, audit listing and ops record.
- `retry-allowed` means a new dispatch with a new approval. `proceed-to-observe` means the effect is applied. `pause` escalates to the owner, who either deletes the object or adds the run id to `reconciled_runs` through a reviewed pull request.
- Recovery deadline: the same working day. After one day the target lifecycle removes any leftover object anyway.

## Cost bound
- Controls allow at most 20 mutating executions per one-day window.
- Each execution makes about 40 S3 and IAM requests: the budget and recheck listings, one GET per retained ops version, the decision and ops writes, live settings reads, the payload write, observation, teardown delete and absence check, and the summary read-back. The object is at most 1,024 bytes; decision and ops records are a few kilobytes each.
- At the limit that is under 2,000 requests a day, growing by one GET per retained ops version, and well under 1 MB stored: less than USD 0.01 a day.
- The runner job is capped at 20 minutes and the AWS CLI at 3 attempts.

## Audit retention
- Decision and ops objects are deliberately kept: they are the immutable decision log and the execution count, so no lifecycle rule expires them.
- Object Lock compliance mode holds each version for one day. After that only a bucket-policy change by the owner allows deletion.
- Retention owner: eugenelim. Retirement date: 2026-12-31. On that date, or earlier under the stop rule, the owner removes the deny statements, deletes the decision and ops versions once their retention has ended, and deletes both buckets and the role.
- Lifetime bound: from 2026-11-15 to 2026-12-31 the 20-per-day cap allows at most 940 executions, so at most about 1,900 decision and ops versions of a few kilobytes each: under 5 MB stored and under 100,000 requests in total, less than USD 0.10.
- Stop rule: if the audit store holds more than 2,000 versions, the owner sets `max_executions` to 1 through a reviewed change and retires the fixture early.

## Classification
- Non-production AWS sandbox account with no real user data and no production path, attested by the owner.
