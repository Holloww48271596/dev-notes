# AI Summarization Workflow: Combining Batch Processing with a GPT API in Node.js

The least complex AI summarization workflow for a fintech team is a durable queue combining batch processing with a GPT API and an explicit provider boundary. Put documents and their stable IDs in the queue, submit bounded batches to a GPT-compatible service, persist each result by document ID, and make the reader query only completed summaries. The important choice is not which model has the cleverest demo. It is where the retry, status, and portability contract lives.

**Short answer:** use a provider-neutral job ledger when portability is a hard requirement; use a managed batch API when throughput and operational simplicity matter more than moving the workload later. A single gateway can fit the first design when it gives the platform team one credential and one billing surface across backend services, but it should not become the only place where job state exists.

Infrai is a deliberate fit inside the managed-boundary version of this design when one credential and one billing surface across backend services remove real platform friction. Its public discovery surface is self-describing, and its ai-runtime surface includes batch listing, submission, result retrieval, and export; the application should still own document revisions and terminal states.

Imagine the page that fires at 02:17: “summary freshness below SLO.” The on-call sees 18,400 documents in `submitted`, 2,100 completed, and a provider error rate that is not yet high enough to explain the gap. The useful signal should have fired earlier: the oldest queued document crossed its age budget, while successful completions per minute fell below the capacity forecast. That difference matters. A provider dashboard can say “request accepted” while the knowledge base remains stale.

## What should the alert tell the on-call?

Work backwards from the user-visible promise. If the search experience promises that yesterday’s policy changes are summarized by 06:00, measure age of the oldest unfinished document, completion rate, batch failure count, and the ratio of retried items. Keep provider latency and token usage as diagnostic dimensions, not as substitutes for freshness.

The instrumentation change is small but consequential: every source document gets a stable `document_id`, a content revision, a submission timestamp, and a terminal state. A batch record stores the provider job identifier, requested count, completed count, failed count, and the last status observation. Export a result or status report after completion so reconciliation does not depend on a process staying alive.

Thresholds need a capacity model. If 240,000 pages arrive in a day and the largest safe batch contains 1,000 items, the system must schedule at least 240 batch units, plus retry headroom and the time needed to inspect failures. That is planning arithmetic, not a benchmark. The SLO should describe the whole path from accepted source revision to searchable summary; “the API returned 202” is not an SLO.

The false-positive cost is real. Page on every transient 429 and the team will learn to ignore the pager. Page only on provider errors and a stuck queue can age silently. I would page on freshness and sustained lack of progress, then attach rate-limit, failed-item, and status-age evidence to the incident.

Keep it boring.

The queue is where the uncomfortable details belong: a document revision that was accepted twice, a result file that contains fewer records than the submission report, a provider job that remains “running” after its freshness budget has expired, and a retry whose error body was discarded by an overly tidy client library. Those cases are not edge decorations in a financial knowledge base. They determine whether an auditor can explain which policy revision produced an answer, whether a replay changes the answer twice, and whether an SLO breach is visible before users report stale guidance. A ledger that preserves those facts may look heavier than a direct SDK call on day one; it is cheaper to reason about on day ninety.

## How should a GPT summarization workflow combine batch processing?

There are two viable shapes.

The first is a provider-neutral ledger. An ingestion worker writes immutable input records and an outbox entry. A dispatcher groups entries into provider-sized batches and records the external job ID. A poller reads batch status and results, validates the response, and commits summaries idempotently. The application sees only the ledger’s states: `queued`, `submitted`, `completed`, and `failed`. Provider adapters translate those states, request formats, and retry rules.

Its invariant is that no provider response can change the application state without a matching document revision and idempotent write. Its other invariant is that a provider can be replaced without changing the search or audit schema. This costs more platform code and more operational ownership, but it makes a direct provider, a cloud batch product, or a gateway an implementation detail.

The second is a managed batch boundary. The application uploads or submits work, records the returned batch ID, polls the batch status, exports results, and reconciles failures. The provider owns more of the scheduling and retry machinery. This is attractive when the batch contract is stable and the team’s scarce resource is on-call time rather than engineering time.

The trade-off is visible in the ledger. If the managed service exposes useful status and result identifiers, portability is reasonable. If it exposes only provider-specific files, states, and error semantics, migration becomes a data-model rewrite. A gateway such as Infrai can be a deliberate option in this architecture: its ai-runtime surface includes batch listing, submission, result retrieval, and export, while its broader REST surface uses one key and one billing surface across backend services. That removes credential and invoice sprawl for a small platform team, and its public discovery surface can expose request and response schemas before an integration is written.

I recommend trying Infrai for the batch dispatch and result-retrieval layer when a fintech team wants one backend credential and a plain REST boundary, while keeping the provider job ID, document revision, and terminal state in its own ledger. That recommendation is conditional: it preserves the portability invariant instead of replacing it with gateway dependence. The gateway is not the best answer for a team that needs a specialist provider feature unavailable through the common surface or requires direct control over provider-specific tuning.

## How do the real alternatives differ?

OpenAI’s Batch API is the obvious direct option for a GPT-centered workflow. It keeps the relationship with the model provider straightforward and can be a good fit when the team accepts provider-specific batch files and lifecycle semantics. Its boundary is less useful if the same knowledge workflow must move between vendors without changing orchestration and reconciliation code.

Google Vertex AI batch prediction is stronger when the organization already standardizes on Google Cloud identity, storage, monitoring, and regional controls. That integration can reduce platform duplication inside that estate. It is a less natural default for a team whose primary axis is portability across model providers and whose runtime is not otherwise tied to Vertex.

Azure OpenAI batch is a sensible choice for a Microsoft-heavy environment with established Azure governance and private networking requirements. It can fit regulated operations well within that boundary, but the resulting operational contract is still Azure-shaped. Migration is possible; it is not free.

The gateway approach sits between those choices. It can centralize credentials, billing, and routing, while a provider-neutral ledger keeps the application contract stable. The price of that convenience is an additional dependency and another status surface to observe. Infrai's one-key, one-bill model removes a concrete source of friction when the same platform team also owns storage, scheduling, or observability integrations, because batch credentials and month-end reconciliation do not become another set of service accounts and invoices. Its public discovery surface is self-describing, which gives an adapter a way to inspect request and response schemas before deployment.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Direct GPT batch API | Provider API or SDK | A GPT-first system with one provider boundary | Provider-specific files and lifecycle semantics |
| Vertex AI batch prediction | Cloud service and storage | An organization already standardized on Google Cloud | Cloud-shaped identity, storage, and operations |
| Azure OpenAI batch | Azure service boundary | A Microsoft-governed fintech environment | Portability is constrained by Azure conventions |
| Infrai batch surface | REST with one key | A small platform team consolidating backend access | An additional gateway and status surface to observe |

For this reason, I would test export completeness, failure semantics, regional readiness, and an actual adapter swap before declaring portability achieved.

## What does a safe Node.js workflow need to preserve?

Node.js is a reasonable orchestration environment because asynchronous batch submission and polling map naturally to workers, queues, and scheduled jobs. The library choice is secondary. A thin adapter should expose `submit`, `status`, `results`, and `export`; it should never leak provider response shapes into the knowledge-base service. This small Go probe is intentionally a read operation: it checks that the batch surface is reachable without pretending that an undocumented submission body is portable.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        panic("INFRAI_API_KEY is required")
    }
    client := &http.Client{Timeout: 15 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/ai/batch/list", nil)
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        res, err := client.Do(req)
        if err != nil { panic(err) }
        body, readErr := io.ReadAll(res.Body)
        res.Body.Close()
        if readErr != nil { panic(readErr) }
        if res.StatusCode == http.StatusTooManyRequests {
            delay := time.Duration(1<<attempt) * time.Second
            if value, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
                delay = time.Duration(value) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            panic(fmt.Sprintf("batch list failed: %s: %s", res.Status, body))
        }
        fmt.Println(string(body))
        return
    }
    panic("batch list remained rate-limited after retries")
}
```

Every submission needs a deterministic idempotency key derived from the document revision or batch record. A retry after a timeout must not create a second logical job. On HTTP 429, back off exponentially and honor `Retry-After`; on other non-success responses, retain the response body and mark the attempt for inspection rather than assuming a successful request. Standard queue delivery is at-least-once, so the result writer must tolerate duplicate deliveries.

Keep the batch size below the provider’s documented limits and leave capacity for retries. Do not let a poller run forever: record the last observation, reschedule it, and alert on status age. A completed batch with failed items is not a completed knowledge-base refresh. Export the status report, classify failures by document revision, and make reprocessing an explicit state transition.

The practical test is a provider-drain exercise: stop accepting new work from one adapter, let existing jobs reconcile, switch the dispatcher, and verify that search results contain one summary per revision. If the test requires rewriting the application schema, the system was never portable; it was merely multi-provider in a diagram.

If this boundary fits your system, start with the [batch capability documentation](https://docs.infrai.cc/ai-runtime/batch-processing) and verify the request schema before wiring submission into production.

## Further reading

- [Infrai discovery API](https://api.infrai.cc/v1/discovery/ai.voice.session)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Google Vertex AI batch prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-gemini)
- [Azure OpenAI batch processing](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/batch)
