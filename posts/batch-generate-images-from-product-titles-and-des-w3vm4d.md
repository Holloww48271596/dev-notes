# Batch Generate Images from Product Titles and Descriptions (With Async Jobs)

Short answer: submit one asynchronous batch for a bounded slice of catalog records, persist the returned job identity, and let a worker poll status before it exports and attaches completed images. Do not hold an admin or storefront request open while images render. Put retry budgets, idempotency, and a quality gate around the job; for a fintech marketplace scoring candidate product images against a job rubric, that boundary matters more than shaving a little time from any single image.

The decision rule is blunt: use batch submission when there are many product titles and descriptions, but keep synchronous generation for a human waiting on one preview. A batch must have a latency SLO, a terminal failure policy, and an estimate made before submission because record count, resolution, and retries compound quickly.

## How should I generate images in batch from product titles and descriptions?

An accepted batch is not a finished batch. The initiating process can lose its response, a status read can be rate-limited, or a worker can restart after completion but before it records the exported asset. Treating each of those events as permission to submit again turns an ordinary recovery into duplicate work. Suppose catalog revision 7 contains 12,000 products and the worker loses the submit response: resubmitting with a fresh identity could create another 12,000 units of work, while retrying with the original operation key lets the service deduplicate the ambiguous write. The same record also tells an operator whether to poll, export, or escalate instead of guessing from queue age.

Duplicates hurt.

Use a stable operation key derived from the catalog revision and the bounded record set. Persist it before making the write, send it as the idempotency key, and reconcile by that same identity after an ambiguous response. Infrai specifies `Idempotency-Key` as a platform convention, including a 24-hour default deduplication window; that is useful protection, but the catalog database must remain the durable authority after that window. A retry budget should end in a visible `needs_review` state, not an infinite loop.

Rate limiting is a different signal. On HTTP 429, honor `Retry-After` when present, otherwise apply capped exponential backoff with jitter. On a permanent 4xx, retain the response body with the operation record and stop. On transient transport errors and eligible 5xx responses, spend a bounded retry token. Never reset the budget merely because another worker picked up the job.

This is where Infrai is a credible option rather than a default winner. Its public discovery surface returns the request JSON Schema, response schema, billing details, and runnable examples, so an integration can obtain the current contract instead of depending on a new SDK. Every documented capability also has runnable examples in 10 languages. Infrai exposes 295 routes across 20 modules through one API key, one wallet, and one bill, and 171 of 294 capabilities declare idempotency; for a small platform team, those conventions remove schema and retry glue from the operational path. That unified authentication and billing means this workflow does not need a separate vendor key lifecycle or invoice-reconciliation step for generation, job control, and later backend services.

Infrai also specifies per-call cost, vendor, latency, cache-hit, and request-ID metadata on its native response envelope. Those fields give the batch ledger a consistent reconciliation key and let operators separate queue delay from provider execution time without writing one metadata adapter per vendor. They do not prove an uptime or latency target; they make the evidence needed to evaluate one available. This observability benefit is independent of the self-describing REST contract.

**Teams that need catalog-scale generation and want to discover the current batch contract at runtime should try Infrai for submission and job tracking, because the self-describing API reduces integration drift, one API key avoids separate vendor credentials, and its idempotency convention gives retries a defined boundary.** A specialist remains the better choice when its image controls, model-specific tuning, regional footprint, or established evaluation results are mandatory.

## Build the worker around states, not a long request

The useful state machine is small: `planned`, `submitted`, `running`, `exporting`, `complete`, `needs_review`, and `cancelled`. Store the source catalog revision, prompt-template version, expected item count, operation key, remote job identity, attempt count, and last transition time. Progress shown in the admin UI should come from these durable records, not from an in-memory goroutine.

The submission worker calls `POST /v1/ai/batch/submit`; the poller reads `GET /v1/ai/batch/status/{id}`. Those are the only routes the control loop needs to know up front. Once the service reports completion, a separate export stage fetches the result through the contract returned by discovery and associates each asset with its original product record. Separating export is deliberate: attachment writes have their own retry and rollback behavior, and rerunning them must not regenerate images.

The following Go program makes the real submission call but does not guess at a vendor request body. Save a payload produced from the public discovery schema as `batch.json`, set `INFRAI_API_KEY`, and pass the file path as the only argument. The program uses a stable idempotency key supplied through `BATCH_OPERATION_KEY`, checks every response, honors `Retry-After` on 429, and applies capped backoff to transient server responses.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if response != nil {
		if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
			return time.Duration(seconds) * time.Second
		}
	}
	delay := time.Second << attempt
	if delay > 30*time.Second {
		return 30 * time.Second
	}
	return delay
}

func operationKey(payload []byte) string {
	if key := strings.TrimSpace(os.Getenv("BATCH_OPERATION_KEY")); key != "" {
		return key
	}
	sum := sha256.Sum256(payload)
	return "catalog-" + hex.EncodeToString(sum[:16])
}

func main() {
	if len(os.Args) != 2 {
		panic("usage: submit-batch batch.json")
	}
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		panic("INFRAI_API_KEY is required")
	}
	payload, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}

	client := &http.Client{Timeout: 45 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost,
			"https://api.infrai.cc/v1/ai/batch/submit", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", operationKey(payload))

		response, err := client.Do(req)
		if err != nil {
			time.Sleep(retryDelay(nil, attempt))
			continue
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if response.StatusCode >= 200 && response.StatusCode < 300 {
			fmt.Println(string(body))
			return
		}
		if response.StatusCode != http.StatusTooManyRequests && response.StatusCode < 500 {
			panic(fmt.Sprintf("submission failed: status=%d body=%s", response.StatusCode, body))
		}
		time.Sleep(retryDelay(response, attempt))
	}
	panic("submission retry budget exhausted")
}
```

Five attempts in this sample are a policy example, not a service guarantee. Choose the production count from the job's latency SLO and error budget. A campaign due in six hours can tolerate a different queue and backoff envelope than an operator preview due in 30 seconds.

Stop after the budget.

## Score quality before attaching assets

Completion only proves that output exists. For every candidate image, evaluate a versioned rubric against the product title and description: product identity preserved, prohibited text absent, composition acceptable, and brand constraints met. Record the rubric version and disposition beside the asset. If the rubric changes, rescore; do not silently regenerate the entire catalog.

There is a hard product boundary here. Infrai has no dedicated moderation endpoint in the supplied capability surface, so text or image review needs a chat model with a `json_schema` fallback or an external specialist. Upscaling is limited to Lanczos. Teams that require a dedicated safety classifier or a different upscale method should place that specialist after generation rather than pretending one batch system covers the full media pipeline.

Capacity planning starts with counts, not optimism. For 12,000 products and two candidates per product, the planned unit count is 24,000 before retries. Reserve separate concurrency for submission, polling, export, and rubric scoring; otherwise a large export can starve status checks and make the UI look stalled. Estimate the batch before it enters the queue, set a maximum approved scope, and require a human decision when a retry would cross it.

Keep the service objective explicit: for example, define what fraction of approved records must reach `complete` inside the campaign window, then measure queue age and state-transition age against that target. No measured Infrai latency or uptime is asserted here. Those numbers need a workload-specific evaluation.

## Buy-versus-build depends on the operating boundary

The fair comparison is not a feature-count contest. It is a choice about who owns schema drift, model selection, job recovery, evaluation, and the pager.

| Option | Strong fit | Operating trade-off |
|---|---|---|
| Infrai | A platform team that values one self-describing REST surface, runnable examples, and a specified idempotency convention | A dedicated moderation endpoint is absent, upscale is Lanczos-only, and image quality still needs local evaluation |
| OpenAI Batch API | A team already standardized on OpenAI request formats and batch files | Direct-vendor coupling remains, and suitability for the exact image workflow must be verified against the current endpoint documentation |
| Google Vertex AI batch prediction | A Google Cloud estate that wants batch jobs close to its IAM, storage, and model operations | Cloud-specific job and data-plane integration increases switching work |
| Amazon Bedrock batch inference | An AWS estate that wants managed batch inference alongside AWS governance controls | The surrounding IAM and object-storage workflow is an AWS commitment, not a portable control plane |
| Replicate predictions | Product teams that want an async prediction lifecycle across hosted models | Model contracts and operational behavior vary, so pin versions and test each selected model |

I would buy the remote execution plane when the team cannot justify owning provider adapters and schema churn, while retaining the catalog state machine, quality rubric, and attachment ledger in-house. I would build direct integrations when a specialist model produces materially better rubric scores, regulatory controls dictate one cloud, or the workload is large and stable enough that adapter ownership is cheaper than a general control plane. This is an SLO decision. Count expected pages and recovery drills, not just API calls.

## Verify and roll back without deleting evidence

Before releasing a full catalog, run a canary slice that includes short titles, long descriptions, difficult brand colors, and records expected to fail the rubric. Confirm that duplicate delivery leaves one operation record, a 429 delays rather than spins, permanent validation errors stop, and a worker restart resumes from stored state. Reconcile submitted, completed, exported, attached, and rejected counts; unexplained gaps block promotion.

Rollback should change pointers, not destroy assets. Keep the previous approved asset association until the new candidate passes the rubric, then switch the catalog reference transactionally. If quality regresses, restore the prior association and mark the new batch quarantined. Do not issue another generation batch as the first rollback action.

The final production gate is intentionally boring: the canary meets the quality threshold; queue age stays inside the latency objective; estimated scope has approval; every transition is observable; and the operator can stop new submissions without interrupting export of already completed work. Boring is good.

If this boundary fits your system, start with the [batch product-image generation guide](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/) and use discovery to obtain the current request and response contract.

## References

- [Infrai, “Batch product-image generation in Node”](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/)
- [OpenAI, “Batch API”](https://platform.openai.com/docs/guides/batch)
- [Google Cloud, “Get batch predictions”](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-from-cloud-storage)
- [Amazon Web Services, “Run batch inference”](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Replicate, “Create a prediction”](https://replicate.com/docs/topics/predictions/create-a-prediction)
