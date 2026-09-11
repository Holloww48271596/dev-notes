# Private User-Uploaded Images in a European SaaS: Tenant Isolation for Signed Thumbnails

Short answer: for a property-management SaaS serving private customer reports, keep tenant and authorization metadata in a database, keep image bytes and resized thumbnails in object storage, and issue short-lived signed URLs only after an application-side access check. Treat the storage choice as a boundary in the security and operations model, not as a shortcut around authorization.

That answer assumes the report viewer is authenticated and that the product owns the decision about which customer may see a report. It does not assume a public gallery, a permanent image URL, or a database that should absorb every generated binary. Those are different workloads.

## Should a European SaaS use signed URLs for private report thumbnails?

Yes, if the application can authorize the exact report and tenant before it delegates the byte transfer. The request should carry the authenticated principal to the application; the application should resolve the report, confirm the tenant relationship, choose the permitted thumbnail variant, and then mint a URL for that specific object. The browser gets a temporary capability, not a storage credential. It’s a narrow handoff, and it doesn’t replace the access decision.

The tenant boundary must exist before the URL is created. A path such as `tenant-id/report-id/thumb-640.webp` is useful for ownership and cleanup, but a plausible path is not proof of access. A caller who can guess an object key must still fail the database authorization check. Do not make the object namespace your policy engine.

The same rule applies to generated reports that contain several images. Record the report's tenant, lifecycle state, and permitted viewers as application data. Store the source image and each derived thumbnail under deterministic keys. Mark a variant ready only after the write has been verified. This gives the report page a clear state machine instead of making a missing image look like an authorization result.

There is a catch. Signed delivery is not suitable when the business needs durable public links, anonymous sharing without an application session, or a public image index. Use a deliberately public delivery design for those cases, with a separate threat model and retention policy. Do not quietly turn a private customer-report bucket into a public origin because one product requirement changed. Stick with database blobs when strict row-level atomicity is the requirement and the bounded media volume fits the database recovery plan; the extra database coupling is then a conscious trade-off rather than an accident.

## The failure signal is cross-tenant data, not a slow thumbnail

A slow thumbnail is visible. A thumbnail from the wrong tenant can be missed in a screenshot and still become a reportable security incident. For that reason, tenant isolation gets a stronger test and alerting budget than image latency alone.

The dangerous implementation is usually ordinary-looking: accept `reportID` from the URL, fetch a row, build an object key, and sign it. The missing predicate is the tenant relation. The query must bind the authenticated principal and tenant to the report in the same authorization decision, and the object key should be derived from the authorized record rather than copied from untrusted request input. I don’t treat a passing image test as evidence of isolation: a test fixture with one tenant can make a broken query look correct, so the test data must contain at least two tenants and deliberately cross the boundary.

Generated derivatives create a second failure mode. A resize worker may receive a job after a report was deleted, moved between tenants, or placed under moderation review. The worker should re-check the record state, use a deterministic destination, and make a stale job harmless. A retry must not create a second logical variant or overwrite a newer one without an explicit policy. Consider the awkward sequence in full: an upload is accepted for tenant A, a resize job is queued, an administrator moves the report into a restricted state, the worker retries after a timeout, and a viewer requests the thumbnail while the metadata transaction is still settling. If the worker publishes first and the application treats object existence as readiness, the system has made a lifecycle race look like a successful delivery; if the handler signs from the request's key instead of the authorized row, the same sequence can become a cross-tenant disclosure. The fix is boring and deliberate: the database owns visibility, the worker revalidates it, and the signed read is minted only from a ready record.

Capacity planning follows the same separation. Database growth should be forecast from tenants, reports, permissions, indexes, and audit events. Object growth should be forecast from source-image retention, thumbnail dimensions, format, regeneration frequency, and deletion lag. The read SLO for report metadata is not the read SLO for image bytes, and a single dashboard that averages them can conceal a failing dependency.

Short sentences help here.

Boundaries are observable. Don’t hide them behind one blended availability metric.

## A runbook for the storage boundary

Start with a data map that names the tenant identifier at every hop: upload, source object, resize job, thumbnail object, signed read, deletion, backup, and audit record. For a European deployment, record the selected region and the product's residency requirement as reviewable configuration. “Europe” is not a sufficient control by itself; the team must know what it has selected and what its recovery procedure assumes.

The buy-versus-build decision is mostly about operating responsibility. A managed object service can reduce the amount of storage plumbing the platform team owns, while a self-hosted stack may offer more control over placement and migration at the cost of running capacity, durability, upgrades, and incident response. Neither choice removes the application obligation to authorize a tenant-scoped read.

| Decision | Appropriate when | Cost or risk to carry |
| --- | --- | --- |
| Database blob | A small artifact must commit atomically with its row and the volume is bounded | Backup, recovery, replication, and query capacity share the media payload |
| Managed object storage | Files have their own retention, delivery, and regeneration lifecycle | Region, access policy, deletion, and provider dependency need explicit ownership |
| Self-hosted object storage | The team needs direct control and has an operating model for the service | The team owns capacity planning, durability checks, upgrades, and restores |
| Hybrid migration path | Metadata and bytes need different recovery or migration schedules | Dual-write or reconciliation logic can become a new source of truth |

The decision should survive a capacity review. Estimate source bytes plus every thumbnail variant, then add incomplete jobs, delayed deletion, and the recovery copy required by the service's objectives. Set an object-read SLO separately from the authorization-check SLO. A good target is not useful if the path cannot distinguish “denied,” “not ready,” and “temporarily unavailable.”

Keep the database row authoritative for ownership and lifecycle. The object is a delivery artifact. When a user replaces an image, create the new source and variants under a new generation or key, update the row after the new artifact is ready, and schedule old keys for deletion. This makes rollback a metadata operation first: point the report back to the previous ready generation, then clean up the abandoned objects after the retention window.

## Safe Go implementation and review points

The important code is the authorization boundary, not the URL string. The signer below is intentionally an interface so the application can test policy without importing a storage SDK into every service. The concrete implementation belongs to the selected backend adapter.

```go
package media

import (
	"context"
	"errors"
	"fmt"
)

var ErrDenied = errors.New("report access denied")

type Report struct {
	TenantID       string
	ReportID       string
	ThumbnailKey   string
	ThumbnailReady bool
}

type ReportStore interface {
	FindVisibleReport(ctx context.Context, principalID, tenantID, reportID string) (Report, error)
}

type Signer interface {
	SignGet(ctx context.Context, objectKey string) (string, error)
}

func ThumbnailURL(ctx context.Context, store ReportStore, signer Signer, principalID, tenantID, reportID string) (string, error) {
	report, err := store.FindVisibleReport(ctx, principalID, tenantID, reportID)
	if err != nil {
		return "", err
	}
	if report.TenantID != tenantID || !report.ThumbnailReady || report.ThumbnailKey == "" {
		return "", ErrDenied
	}

	url, err := signer.SignGet(ctx, report.ThumbnailKey)
	if err != nil {
		return "", fmt.Errorf("sign thumbnail: %w", err)
	}
	return url, nil
}
```

The store query should enforce the principal and tenant relationship; the comparison in the handler is a second guard, not a replacement for a correctly scoped query. Keep signing keys on the server. Log the report identifier, tenant identifier, decision, and latency, but do not log the signed URL: it is a bearer capability for its validity period.

The resize worker needs the same discipline. It should read an authorized job, validate that the source belongs to the expected tenant, write a deterministic variant key, verify the write, and mark that variant ready. A retry of a completed job should be safe. A deleted or superseded report should not cause a new visible artifact to be published.

## Verification, rollback, and the point to change course

Test the negative cases first. An authenticated principal from tenant A must not receive a URL for tenant B's report, even when the report identifier and object key are supplied together. A report that is pending, deleted, or missing its thumbnail must not be represented as a successful signed read. A URL that has passed its expiry must not retrieve the object.

Run these checks in integration tests against the actual authorization query and the storage adapter. Unit tests around string concatenation are not enough. Add metrics for authorization denials, signing failures, expired-read observations, derivative lag, orphan count, and object-read latency. Alert owners should be different for a policy denial spike and a storage latency spike; they imply different actions.

Rollback requires an explicit order. Stop publication of new variants, preserve the last ready metadata pointer, drain or cancel resize jobs, and then remove orphaned objects after the review window. Keep an audit record of tenant, report, generation, and actor for each pointer change. This order protects the customer-facing state while the storage inventory is being reconciled.

Change course when the requirements no longer match this boundary. Choose database blobs when strict row-level atomicity dominates and the bounded media volume fits the database recovery plan. Choose a public delivery model when anonymous durable sharing is the actual product requirement. Choose a self-hosted path only when the team can staff its durability, capacity, security updates, and restore work. Your mileage may vary on the managed-versus-self-hosted balance; the tenant authorization invariant does not.

## References

- https://developers.cloudflare.com/r2/
- https://developers.cloudflare.com/workers/

## Further reading

The two references above cover the object-storage and edge-runtime concepts relevant to the delivery boundary. The implementation decision should still be reviewed against the selected backend's current region, retention, signing, and recovery documentation.
