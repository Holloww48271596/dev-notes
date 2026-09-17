# Rotate Production API Keys Without Downtime in Node.js: Grace-Period Secret-Store Deploys

## Short answer

Short answer: rotate a production API key with two active versions, a measured grace period, and a secret-store version check in every Node.js process. For a healthtech access review, the deciding metric is the blast radius of one credential: scope each key to one deploy target, prove adoption before revocation, and keep the old version usable only for the overlap window.

That is the answer I would sign. A credential is not an identity record, and a green deployment is not proof that every worker loaded the new secret.

During an access review, I once treated a successful secret-store read as adoption evidence. The dashboard showed 100% reads, yet one Node.js worker had cached its credential in a module-level client. It kept sending requests with the previous key after the rollout. No outage followed, but the review could not answer the important question: which workloads could one leaked key still reach?

The invariant is stricter than “the deploy finished.” Every instance must report the secret version it has loaded, the process start time, and its target scope. The controller publishes the new version, waits for those observations, holds the old version through a bounded grace period, and only then revokes it. A provider authentication response can cause one refresh and one retry; a timeout cannot, because a timeout is not evidence that the credential is invalid.

Short signals beat optimistic dashboards.

## How should Node.js rotate production API keys with a grace period and secret store?

Use three timestamps: `publish_at`, `retire_at`, and the maximum local cache age. Set the overlap longer than cache age plus rollout delay, queue delay, and the time needed for regional secret replication. Measure those values from telemetry instead of copying a vendor default. I'm not sure any fixed “24 hours” is safe for every healthtech workload; your mileage may vary when a batch worker can sleep overnight.

The secret record needs an immutable version and a small state machine. In phase one, publish the new key while the old key remains valid. In phase two, roll instances and record their observed version. In phase three, revoke the old key only after the grace deadline and the adoption check both pass. If either check is missing, stop the rotation and page the owner; do not guess.

Here is the core decision path in Go. It is deliberately independent of a particular secret product.

```go
package rotation

import "time"

type Observation struct {
	Version   string
	Observed  time.Time
	StartedAt time.Time
}

func SafeToRevoke(now, retireAt, publishAt time.Time, maxCacheAge time.Duration, o Observation, expected string) bool {
	if expected == "" || o.Version != expected {
		return false
	}
	if publishAt.IsZero() || retireAt.Before(publishAt) || now.Before(retireAt) {
		return false
	}
	return !o.Observed.IsZero() && now.Sub(o.Observed) >= maxCacheAge
}
```

The production path should also reject a rotation record whose `retire_at` is earlier than `publish_at`, and it should use a monotonic deadline for local retries while storing audit timestamps in UTC. Keep authentication retries bounded at one per request. Never log the key; log only a version, workload identifier, and response class.

## What does a deploy prove, and what does it leave unproven?

A rolling deploy proves that the orchestrator replaced some containers. It does not prove that a warm process discarded a cached client, that a delayed queue item can still authenticate during overlap, or that a second region saw the new secret. Those are separate assertions and belong in the access-review evidence.

Test the boundaries with a fake store and provider stub: a process started before publication, a restart during overlap, a refresh delayed past cache age, duplicate deploy events arriving out of order, and a queued request signed with the old version. Assert that each event keeps the same workload identity and that the audit record contains one key version per attempt. Test the edges.

For a healthtech review, join access logs to an immutable workload ID, not to the API key. A shared key cannot tell you which service touched a patient-data endpoint, so split that identity before attempting rotation. Set an SLO for rotation evidence, such as “99% of instances report the target version within the rollout budget,” then alert on the remaining 1% before the revoke job runs.

## Buy, build, and the boundary of this advice

| Approach | What it controls well | Failure boundary | Use it when |
| --- | --- | --- | --- |
| Two-key overlap | No planned downtime and an auditable handoff | Requires version telemetry and timed revocation | Rolling production deploys are required |
| Per-request lookup | Fast propagation after publication | Secret-store latency enters the request path | Low-volume control-plane calls |
| Local cache with refresh | Predictable latency and bounded store load | Stale credentials persist until cache expiry | High-throughput Node.js services |
| Brokered credential exchange | Keeps raw keys out of workloads | Adds a service and another SLO | Many teams need centrally enforced scope |

The catch is operational weight: overlap means two live credentials, a revoke controller, and evidence retention. This is not suitable when the upstream system permits exactly one active key and offers no overlap semantics; use a drained maintenance window or a credential broker there. Stick with immediate replacement when downtime is explicitly budgeted and reconciliation is already part of the runbook.

The choice is about blast radius, not a headline price. A narrowly scoped key that expires predictably can be safer than a cheaper shared credential, even when both pass a basic connectivity test.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://nodejs.org/api/process.html
- https://nodejs.org/api/timers.html
