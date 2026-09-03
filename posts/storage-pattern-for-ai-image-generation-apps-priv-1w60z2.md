# Storage Pattern for AI Image Generation Apps: Private Originals, Thumbnails, Variants

Short answer: put each tenant's immutable image files in private object storage, keep the restore catalog in a database, and make large-file throughput a measured property of the upload, copy, and download paths. Originals, thumbnails, and generated variants should have different retention decisions, while signed links should be issued only after the application has authorized the request.

That pattern is the least complicated starting point for a fintech service that stores AI-generated images and must restore a selected per-tenant backup snapshot. The important boundary is not a clever key name. It is the separation between durable bytes, searchable backup state, and temporary work. It keeps a restore from turning into an application-wide scan, and it gives capacity planning something more useful than a single bucket-total chart.

The catch is that private object storage is not a backup policy by itself. It does not decide which snapshot is authoritative, prove that a restore is complete, or make a slow large-object transfer meet an SLO. For regulated records, immutable retention and provider-native replication controls may outweigh a portable storage abstraction; for a small image gallery with no restore requirement, the backup catalog and verification machinery described here may be unnecessary.

## The restore contract comes before the bucket

The first design question is not which storage API to call. It is what a selected snapshot must prove when a tenant asks for restoration.

For a fintech application, I write that contract as a manifest containing the tenant, snapshot, object key, size, checksum, retention class, and completion state. A restore is complete only after every manifest entry has been fetched or copied, verified, and recorded as complete. If the manifest is missing, the system can have every byte and still be unable to explain which bytes belong to the requested snapshot.

Measure it.

The failure mode is easy to miss in a demo: a generator writes an original, three large variants, and a handful of thumbnails; the database transaction is interrupted; then a retry starts from the beginning while an operator sees only total storage growth. A bounded restore worker instead reads the manifest, reserves a limited number of transfers, persists one result per object, and resumes after a process restart. The design does not assume that the slowest object has the average size, that a regional network path has yesterday's throughput, or that a successful copy means a correct copy. It makes each assumption testable, which is the useful part of the architecture.

That is also the incident lesson I would carry into capacity planning: restore throughput is a workflow property, not a bucket property. Track time to first usable file and time to complete the selected snapshot, alongside bytes per second, retry count, and concurrency. A single aggregate bandwidth number is not an SLO.

## What should an AI image generation app store as originals, thumbnails, and variants?

Use object keys that make the retention class explicit and put tenant identity in every durable record:

| Class | Example key shape | Operational meaning |
| --- | --- | --- |
| Original | `tenant/{tenant_id}/original/{asset_id}` | Source material that a restore must preserve |
| Variant | `tenant/{tenant_id}/variant/{asset_id}/{variant_id}` | A reproducible or accepted generated result |
| Thumbnail | `tenant/{tenant_id}/thumbnail/{asset_id}/{size}` | A derived read optimization |
| Snapshot manifest | `tenant/{tenant_id}/snapshot/{snapshot_id}/manifest.json` | A database-referenced inventory of the selected backup |
| Temporary | `tenant/{tenant_id}/work/{job_id}/{artifact_id}` | Intermediate output with a separate cleanup policy |

The database should own the facts needed to answer product and recovery questions: tenant, asset, snapshot, object key, content type, byte count, checksum, generation state, and retention class. The bucket holds bytes and their storage metadata. It should not become a slow gallery index merely because listing keys is convenient.

This layout also makes a failed generation understandable. A worker may write an original-derived intermediate, produce several sizes, and then lose its lease before the database commit. The work prefix is allowed to contain abandoned objects; the publish transaction is what makes a variant visible. A cleanup job can select old work records, while a lifecycle rule provides a second line of defense for objects that no longer have an application owner. The two mechanisms have different purposes, so I would not promise a deletion SLO from a lifecycle rule alone.

Thumbnails are the easy place to make a bad retention decision. If they can be regenerated from an original or an accepted variant, they are cache-like and can be rebuilt after restore. If the product permits an edited thumbnail that cannot be recreated, it is a durable variant and needs the same backup treatment as the source it represents. Name the difference in the schema; do not ask an operator to infer it from a filename.

## How do private links and large-file transfers affect restore throughput?

Signed links solve authorization at delivery time. They do not solve transfer capacity. The application should authorize the tenant and snapshot first, then return a short-lived link for the exact object or a controlled download stream. The link must not be the database's identity for the object, and a client should not be able to turn a signed link for one tenant into a path for another tenant.

For large originals and backup manifests, measure the whole path: object size, upload duration, download duration, retry count, concurrency, and time from restore request to the first usable file. A useful throughput test uses representative large files, multiple tenants, and the same parallelism limits planned for production. Tiny thumbnails can make a transfer dashboard look healthy while a selected snapshot containing multi-gigabyte originals misses its recovery target.

The restore coordinator should work from a manifest, verify the tenant and snapshot state, and copy or fetch objects in bounded batches. It should record each completed object idempotently, so a retry resumes from known progress rather than restarting every large file. Keep the concurrency limit explicit: too little parallelism leaves bandwidth unused; too much turns retries and memory pressure into a new incident.

Here is the part I would keep in the application rather than hide inside a storage helper. It validates the retention boundary, tracks per-object restore state, and leaves the provider-specific transfer implementation behind a small interface:

```go
package restore

import (
	"context"
	"fmt"
	"path"
	"strings"
)

type Object struct {
	Key      string
	Size     int64
	Checksum string
}

type Store interface {
	Copy(ctx context.Context, sourceKey, destinationKey string) error
	SignedReadLink(ctx context.Context, key string) (string, error)
}

func durableKey(tenantID, class, assetID, suffix string) (string, error) {
	if tenantID == "" || assetID == "" || strings.ContainsAny(tenantID+assetID, "/\\") {
		return "", fmt.Errorf("invalid tenant or asset id")
	}
	switch class {
	case "original", "variant", "thumbnail":
	default:
		return "", fmt.Errorf("class %q is not durable", class)
	}
	if suffix == "" || path.IsAbs(suffix) || strings.Contains(suffix, "..") {
		return "", fmt.Errorf("invalid object suffix")
	}
	return path.Join("tenant", tenantID, class, assetID, suffix), nil
}

func restoreOne(ctx context.Context, store Store, tenantID, snapshotID string, obj Object) error {
	destination, err := durableKey(tenantID, "variant", snapshotID, obj.Key)
	if err != nil {
		return err
	}
	if err := store.Copy(ctx, obj.Key, destination); err != nil {
		return fmt.Errorf("copy %q: %w", obj.Key, err)
	}
	return nil
}
```

The example deliberately does not pretend that every provider has the same copy semantics, checksum behavior, or signed-link API. Those details belong in the `Store` implementation and in a restore test against the chosen backend. I would also reject an object whose observed size or checksum differs from the manifest, then mark the object complete only after that verification.

One production lesson is boring but expensive: never let a restore worker use an unbounded list operation as its work queue. List from the manifest, bound concurrent transfers, and make the database state transition explicit. A 429, a dropped connection, or a process restart should delay one object, not erase the operator's understanding of the entire snapshot. Your mileage may vary by region and object-size distribution, so the SLO needs measurements from the deployment that will actually serve the tenants.

## What lifecycle and snapshot rules should the system enforce?

Retention is a policy table, not a cleanup script. I would make the decision visible before implementation:

| Data | Delete condition | Restore consequence |
| --- | --- | --- |
| Original | Tenant retention policy expires and legal holds permit deletion | Cannot be regenerated; keep through the backup window |
| Accepted variant | Asset and snapshot retention expire | Rebuild only if the generation inputs are retained and reproducible |
| Thumbnail | Rebuildable source exists and cache retention expires | Restore the source, then regenerate |
| Work artifact | Age exceeds the abandoned-job threshold | No product-visible consequence |
| Snapshot manifest | Snapshot is superseded and its retention window expires | Selected restore is no longer available |

Object lifecycle management is useful for age-based transitions and expiration, but its scope and timing must be matched to the recovery policy. AWS documents lifecycle management for objects in S3, and Google documents storage behavior and controls for Cloud Storage; the engineering point is portable even when the rule syntax is not. Treat lifecycle as an asynchronous policy engine. Use an application scheduler when the deletion objective is tighter than the storage service's lifecycle behavior can guarantee.

Before deleting a snapshot, verify that its manifest is not selected by an active restore, that no legal hold applies, and that the database has recorded the retention decision. For tenant isolation, cleanup must select by an exact tenant-owned prefix or manifest set, not by a broad substring. A deletion metric should include candidate count, successful count, skipped count, and age of the oldest remaining eligible object.

The failure mode worth testing is a partial restore followed by cleanup. A restore should pin the snapshot or otherwise prevent its objects from becoming deletion candidates until verification completes. Then inject a worker restart halfway through a large transfer, retry the same object, and confirm that the recorded state is idempotent. That test catches more than a happy-path upload test does.

## Which storage approach fits the recovery objective?

The buy-versus-build choice is really a control-boundary choice:

| Approach | Strength | Trade-off |
| --- | --- | --- |
| Direct object-storage integration | Exposes the selected provider's transfer, retention, and recovery controls | Couples the application and on-call knowledge to that provider |
| Shared storage interface | Keeps tenant, manifest, signing, and restore logic stable across backends | The common interface must not erase features required by the SLO |
| Self-hosted object storage | Gives the team direct control over placement and operations | Makes capacity, replication, upgrades, and 24-hour response part of the team's workload |

I would choose a direct integration when version history, object lock, replication, or a provider-specific large-object path is a hard requirement. I would use a shared interface when the interchangeable contract is intentionally narrow: put, copy, read, sign, inspect, and delete, with explicit limits around consistency and retention. I would not self-host merely to avoid a vendor bill if the team cannot also own capacity planning, failure testing, and recovery operations.

The recommended pattern is therefore conditional. It fits a private multi-tenant image service whose originals and accepted variants need controlled delivery and whose restore process can be measured from a manifest. It is not suitable when public anonymous delivery, WORM retention, provider-native cross-region recovery, or a deletion guarantee finer than the storage lifecycle mechanism is the actual requirement. Stick with the direct provider controls when those guarantees are part of the contract; keep the application-level key and manifest model either way.

## Further reading

- [Amazon S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
