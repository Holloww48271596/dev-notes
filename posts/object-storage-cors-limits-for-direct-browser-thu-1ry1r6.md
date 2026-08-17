# Object Storage CORS Limits for Direct Browser Thumbnail Uploads to Private Buckets

Short answer: keep original images in a private object store, authorize a narrowly scoped upload intent in the application, and treat browser CORS, request signing, and thumbnail processing as three separate controls. A valid signed upload request does not grant a web page permission to send it, and a successful write does not prove that an image is safe or ready for resizing.

This is a capacity and failure-domain decision before it is a JavaScript decision. Direct browser transfer can remove application bandwidth from the hot path, which matters once upload concurrency becomes a real part of the capacity plan, but it also creates a boundary where browser policy, temporary credentials, object naming, and asynchronous work can each fail independently. Keep those signals distinct.

## What causes a browser upload to a private object store to fail?

A private bucket is compatible with direct upload. The browser still needs permission from the storage origin through CORS, while the storage service needs a request that satisfies the signed authorization. Those checks answer different questions. CORS decides whether a page from a particular origin may make a cross-origin request with a method and set of headers; a presigned request bounds authority to the details that were signed, including its expiry and any signed request components. Neither replaces the other.

The first common failure happens before the body leaves the browser. A `PUT` with a non-simple header can cause an `OPTIONS` preflight, and the browser will block the follow-on request if the response does not permit the page origin, method, or requested headers. The second happens at the authorization boundary: application code signs one method, object key, content type, or expiry, then the browser sends a materially different request. A third succeeds as a write but has no thumbnail because the system equated storage receipt with processing completion.

Name the gate first.

That distinction changes incident response. A browser console report of a rejected preflight calls for comparison of the observed origin, method, and requested headers against the CORS policy. A storage authorization rejection calls for comparison of the issued intent against the actual request. An object that exists without a derivative belongs to the event, validation, queue, or resizing path. Broadening CORS cannot repair a signed-request mismatch, and opening a private namespace cannot make a decoder or worker finish.

CORS policies should therefore be deliberately narrow: known application origins, the upload methods actually used, and only the headers the browser must send. The storage policy should remain private and accept writes only through the intended authorization mechanism. Do not log a full signed URL; it is a temporary credential. Log an upload identifier, object key, expiry time, planned method, selected content type, and an outcome for each stage instead. Those fields make an SLO for upload acceptance separate from an SLO for derivative readiness.

For a useful diagnostic exercise, write down the expected request before opening browser tools: the page origin, the method, every application-supplied header, the content type, the opaque object key, and the time the intent becomes invalid. Then capture the browser's preflight and its eventual upload attempt with the same upload identifier. The comparison is intentionally boring, but it separates a policy mismatch from an authorization mismatch without changing a security control under pressure. If preflight denies a header that the signer required, there are two independent changes to review: the CORS response must permit the browser to send it, and the application contract must preserve that header exactly. If the upload succeeds but no derivative appears, stop looking at CORS; inspect the receipt-to-verification transition, the idempotency record, and the resize queue. This is also where a combined availability indicator causes trouble. A healthy uploader can coexist with delayed validation, while an unavailable signer prevents any new upload intent even when the object store and image workers are otherwise healthy. Separate stage objectives turn a vague user-facing upload failure into a bounded ownership question.

## How should direct browser thumbnail uploads use CORS, private buckets, and presigned requests?

The browser should upload an original, not a trusted thumbnail. It first asks an application endpoint for an upload intent. That endpoint authenticates the caller, applies quota and media policy, creates an opaque destination key, and returns one short-lived request plus the exact headers that must accompany it. The browser replays those values without substituting a new content type or adding an unplanned header.

After the object is received, a separate control path verifies the content and creates derivatives under service-controlled keys. Decode the uploaded file rather than relying on its filename or declared media type, enforce byte and pixel limits, and define a quarantined result for files that fail validation. The delivery layer exposes only verified derivatives under its own authorization rules. This keeps an untrusted original from becoming a browser-visible thumbnail merely because a client was able to upload bytes.

A small Go boundary makes the division of responsibility visible. The handler owns application policy and the response contract; a storage-specific signer stays behind an interface, where canonical request details belong.

```go
package upload

import (
	"encoding/json"
	"net/http"
	"path/filepath"
	"strings"
	"time"
)

type Signer interface {
	PresignPut(key, contentType string, expires time.Time) (string, http.Header, error)
}

type Handler struct {
	Signer Signer
	Now    func() time.Time
	NewKey func(extension string) string
}

type intentRequest struct {
	Filename    string `json:"filename"`
	ContentType string `json:"content_type"`
}

type intentResponse struct {
	UploadID  string      `json:"upload_id"`
	ObjectKey string      `json:"object_key"`
	URL       string      `json:"url"`
	Headers   http.Header `json:"headers"`
	ExpiresAt time.Time   `json:"expires_at"`
}

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var in intentRequest
	if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 4096)).Decode(&in); err != nil {
		http.Error(w, "invalid request", http.StatusBadRequest)
		return
	}
	if !strings.HasPrefix(in.ContentType, "image/") {
		http.Error(w, "unsupported media type", http.StatusUnsupportedMediaType)
		return
	}

	key := h.NewKey(filepath.Ext(in.Filename))
	expires := h.Now().Add(10 * time.Minute)
	url, headers, err := h.Signer.PresignPut(key, in.ContentType, expires)
	if err != nil {
		http.Error(w, "upload intent unavailable", http.StatusServiceUnavailable)
		return
	}

	response := intentResponse{key, key, url, headers, expires}
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(response)
}
```

This example does not make the browser responsible for trust. Production policy also needs authenticated identity, an allowlist that is narrower than every `image/*` subtype, quota, collision-resistant keys, and audit fields. The output contract needs to state that returned headers are mandatory, because a header inserted by an HTTP helper can change the request the signer authorized.

The processing path needs idempotency. Storage notifications and client completion notices can repeat, so model the object as a state transition such as `issued -> received -> verified -> derived`, and use compare-and-set state or an idempotency key before a resize job is released. Emit the upload ID through every stage. A single generic metric called image upload hides the important question: was the user delayed by transfer, verification, or derivative generation?

## The operational trade-offs behind the data path

Different owners.

Use direct browser-to-store transfer when originals are frequent or large enough that the application tier should not carry the data-plane bandwidth. The application still owns the control plane: identity, policy, key issuance, status, and revocation rules. Its on-call burden shifts from socket pressure to signer correctness, CORS regression coverage, object-event handling, queue depth, decoder maintenance, and trace correlation.

| Path | Fits best when | Primary operating obligation | Boundary to accept |
| --- | --- | --- | --- |
| Direct browser write to a private object store | Upload volume or object size would pressure the application tier | Intent issuance, exact request handling, verification, and lifecycle controls | Browser policy and signed-request details must remain aligned |
| Application-proxied upload | Every byte requires synchronous inspection before persistence, or clients cannot reach the storage origin | Bandwidth, connection capacity, buffering, and horizontal scaling | The application becomes the transfer bottleneck |
| Multipart upload | Originals are large and network retries are expected | Part state, retry policy, completion, and cleanup | Coordination overhead is excessive for small images |
| Managed media pipeline | The team accepts an external operating contract for less specialized work | Contract review, observability mapping, and exit planning | Feature boundaries and portability constraints can shape the design |

Multipart upload is a useful illustration of why the original and the derivative have different operating profiles. The multipart model permits independent upload and retry of parts, then requires explicit completion to assemble the object. Incomplete work must be completed or aborted so uploaded parts do not continue to consume storage. A high-resolution original may justify that coordination; a small generated thumbnail usually does not.

The catch is that direct browser upload is not suitable when policy requires application-layer inspection before storage accepts any bytes, or when the traffic is tiny enough that a separate signer and asynchronous pipeline cost more than the bandwidth they avoid. An application proxy is the clearer choice there. Keep a managed pipeline when the team cannot sustain image-decoder patching, queue operations, derivative backfills, and deletion propagation. Self-host processing only when custom codecs, locality, or portability warrants the corresponding on-call load.

## Test the failure modes before production traffic

Build the release check around observable contracts, not a happy-path screenshot. Test an allowed origin and a denied one; the exact upload method and header set; an expired intent; an object key outside the issued namespace; a repeated completion notice; an oversized or malformed image; and a partial notification delivery. The expected result should identify the gate and retain the upload ID for investigation.

Capacity planning should count originals, derivative count, validation reads, lifecycle retention, request volume, and retry amplification separately. Storage charges can include stored capacity, requests, retrieval, transfer, and management capabilities, so a stored-gigabyte estimate does not describe the workload. Put a ceiling on retry behavior through an error-budget policy, otherwise a dependency disruption can multiply work across the browser, control plane, and worker fleet.

There is no universal upload topology. The defensible design is the one that preserves private originals, grants least-privilege write intents, has bounded expiry, verifies content before exposure, makes duplicate work harmless, and tells on-call which stage missed its objective. That is a much better production contract than a bucket policy adjusted until a browser error disappears.

## References

- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
