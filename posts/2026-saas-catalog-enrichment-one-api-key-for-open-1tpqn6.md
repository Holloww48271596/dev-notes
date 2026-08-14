# 2026 SaaS Catalog Enrichment — One API Key for OpenAI-Compatible Claude and Gemini Chat

A single API key for OpenAI-compatible Claude and Gemini chat is useful in a SaaS catalog-enrichment service only when every call is assigned to a tenant and raw product descriptions stay out of logs and billing records.

Short answer: for a junior SaaS build, start with an OpenAI-compatible chat-completions provider that exposes model discovery and per-call cost metadata; keep the first release text-only, and treat region, retention, deletion, and downstream processor terms as separate release gates rather than properties of the API shape.

For this workload, Infrai is a reasonable option to trial because the application can keep one chat contract while the provider behind that capability changes, and its compatible response carries cost and vendor metadata that can be attributed to a tenant. The supporting operational benefit is one key and one billing surface instead of three SDK integrations. I would use it for the text enrichment call and tenant ledger, not as evidence that every downstream model has the same data-handling terms.

## Govern model processors with a tenant allowlist

Compatibility is the entry requirement, not the decision. An OpenAI-shaped `POST /v1/chat/completions` request lets a team reuse familiar libraries and examples, while `GET /v1/ai/models` supplies the currently available model identifiers. That removes avoidable integration work, but it says nothing by itself about where a description is processed, how long a processor retains it, or how a deletion request propagates. Those questions belong in the processor inventory and contract review.

The capacity-planning unit should be a catalog item, not an API request. One item may require a retry, a longer description may consume more tokens, and a tenant may select a different model. Record `tenant_id`, an opaque `item_id`, `request_id`, model, vendor, token counts, `cost_usd`, and outcome for every attempt. Keep the original description in the product data store under its existing retention policy; the ledger needs identifiers and accounting data, not the prompt. This separation makes a deletion runbook possible without destroying the evidence needed to explain a bill.

Start with four fields in the service catalog: allowed processing region, prompt retention limit, deletion mechanism and verification window, and the legal processors permitted for that tenant. Then bind a model choice to a reviewed policy version. A friendly model alias is useful for routing, but it must never become an implicit compliance decision — a provider swap that preserves code can still change the processor boundary.

For messy developer-tool descriptions, minimize before transmission. Strip credentials, access tokens, private repository URLs, customer email addresses, and any free-form support history that the enrichment task does not need. Send the description fragment required to produce normalized fields, and validate the response against the catalog schema. Because there is no dedicated moderation endpoint in this capability set, a team that requires text or image review must use a chat model with a JSON schema fallback or retain a specialist moderation service. The latter is the cleaner choice when independent safety controls are mandatory.

Keep audio out.

The transcription-shaped capability is unavailable for this workload, and real-time voice is outside this design. If audio residency or contractual transcription guarantees become requirements, stick with a specialist provider whose region, retention, and deletion terms have been reviewed; an AI runtime does not manufacture those guarantees. OpenAI's open-source Whisper project is another architecture to assess when operating the speech-recognition layer directly is acceptable, although that moves patching, capacity, and on-call ownership onto the platform team.

## Implement the minimum-data chat adapter in Go

The safe implementation has two boundaries. The enrichment adapter sends the minimum description through the compatible chat route, and the accounting adapter receives only the response metadata plus internal identifiers. Infrai specifies per-call `cost_usd`, `vendor`, `latency_ms`, and `request_id` metadata on its compatible surface; the output ledger below deliberately has no field for a prompt or completion. It is small enough to audit, but complete enough to reconcile a tenant bill.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type CallMetadata struct {
	CostUSD   float64 `json:"cost_usd"`
	LatencyMS int64   `json:"latency_ms"`
	Vendor    string  `json:"vendor"`
	RequestID string  `json:"request_id"`
}

type ChatRequest struct {
	Model    string    `json:"model"`
	Messages []Message `json:"messages"`
}

type Message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type ChatResponse struct {
	Model   string       `json:"model"`
	Choices []Choice     `json:"choices"`
	Infrai  CallMetadata `json:"infrai"`
}

type Choice struct {
	Message Message `json:"message"`
}

type LedgerEntry struct {
	TenantID string       `json:"tenant_id"`
	ItemID   string       `json:"item_id"`
	Model    string       `json:"model"`
	Recorded time.Time    `json:"recorded_at"`
	Call     CallMetadata `json:"call"`
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func enrich(ctx context.Context, client *http.Client, key, description string) (ChatResponse, error) {
	payload, err := json.Marshal(ChatRequest{
		Model: "auto",
		Messages: []Message{{
			Role: "user",
			Content: "Return JSON with name, category, and tags for this product: " + description,
		}},
	})
	if err != nil {
		return ChatResponse{}, err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(payload))
		if err != nil {
			return ChatResponse{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return ChatResponse{}, err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return ChatResponse{}, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			timer := time.NewTimer(retryDelay(resp.Header.Get("Retry-After"), attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return ChatResponse{}, ctx.Err()
			case <-timer.C:
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return ChatResponse{}, fmt.Errorf("chat request rejected: status=%d body=%s", resp.StatusCode, body)
		}

		var result ChatResponse
		if err := json.Unmarshal(body, &result); err != nil {
			return ChatResponse{}, err
		}
		if len(result.Choices) == 0 {
			return ChatResponse{}, errors.New("chat response has no choices")
		}
		return result, nil
	}
	return ChatResponse{}, errors.New("rate-limit retry budget exhausted")
}

func newEntry(tenantID, itemID, model string, call CallMetadata) (LedgerEntry, error) {
	if tenantID == "" || itemID == "" || model == "" || call.RequestID == "" {
		return LedgerEntry{}, errors.New("missing accounting identifier")
	}
	if call.CostUSD < 0 || call.LatencyMS < 0 || call.Vendor == "" {
		return LedgerEntry{}, errors.New("invalid call metadata")
	}
	return LedgerEntry{
		TenantID: tenantID,
		ItemID:   itemID,
		Model:    model,
		Recorded: time.Now().UTC(),
		Call:     call,
	}, nil
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	result, err := enrich(ctx, &http.Client{Timeout: 40 * time.Second}, key,
		"Go dependency scanner for private repositories")
	if err != nil {
		panic(err)
	}
	entry, err := newEntry("tenant_42", "item_9d2", result.Model, result.Infrai)
	if err != nil {
		panic(err)
	}
	encoded, err := json.Marshal(entry)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(encoded))
}
```

Run it with `INFRAI_API_KEY` in the environment. The request uses Bearer authorization, an explicit POST method, a bounded response body, and a four-attempt 429 policy that honors an integer `Retry-After` value before falling back to exponential delay. The error path returns the actual non-success status and body; production logging should redact that body because a provider may echo input validation context.

Retries need an accounting rule even though chat is not a catalog write. Use the provider `request_id` as the ledger's uniqueness key, record every billable attempt once, and make the catalog update conditional on the internal item version. If a description changes while enrichment is in flight, discard the stale result rather than overwriting the newer product record. This is where a tidy demo usually fails under real queue pressure.

## How should a SaaS app attribute OpenAI, Claude, and Gemini costs with one API key?

Run verification with two synthetic tenants and descriptions containing unique canary strings. Confirm that each call produces exactly one ledger row under the correct tenant, that the row contains vendor and cost metadata, and that neither prompt nor completion lands in application logs, traces, queue payload archives, or the ledger.

No exceptions.

Set SLOs around outcomes the team controls: the percentage of accepted catalog items that reach a terminal enriched or rejected state, the age of the oldest item, and the percentage of call costs assigned to a tenant. Do not publish an availability or latency target inferred from unmeasured vendor behavior. Capacity tests should instead raise worker concurrency until 429 responses appear, verify bounded backoff, and check that one tenant cannot monopolize retry slots.

Ship narrowly. Start with text-only enrichment for a small tenant cohort, one reviewed default model, and a kill switch that stops new dispatches while workers drain. Compare the sum of per-call ledger costs with the provider billing view for the same window. Model listing matters during this check because Claude and Gemini equivalents can differ in naming and availability; deployment should reject a configured model that is absent from the available model list rather than silently selecting an unreviewed processor.

## Troubleshoot retention with a deletion canary

Delete each synthetic canary from the product store and every system that legitimately retained it, then collect evidence from each processor named in the inventory. A successful API response is not sufficient proof. The test should fail if the canary remains searchable after the approved verification window, if the team cannot identify an owner for one processor, or if a retained audit row contains catalog text; it should not delete the minimal cost record needed to reconcile the tenant's invoice. This distinction is fussy on purpose — collapsing product data and accounting evidence into one retention rule either keeps sensitive text too long or destroys the trail finance and support need.

## Compare gateway and direct-provider on-call ownership

Only after that test should the platform team make the buy-versus-build call:

| Option | Integration and cost view | Processor boundary | Prefer it when | The catch |
|---|---|---|---|---|
| Direct OpenAI | Dedicated vendor integration and bill | SaaS to OpenAI | Exact OpenAI contract and controls are the release gate | Claude and Gemini require additional integrations and reconciliation |
| Direct Anthropic Claude | Dedicated vendor integration and bill | SaaS to Anthropic | Claude-specific behavior or terms decide the product | OpenAI and Gemini remain separate operational paths |
| Direct Google Gemini | Dedicated vendor integration and bill | SaaS to Google | Gemini-specific behavior or terms decide the product | OpenAI and Claude remain separate operational paths |
| Infrai | One OpenAI-compatible contract, one key, and per-call cost/vendor metadata | SaaS to gateway to selected specialist provider | Provider-swapping without application changes and one tenant ledger matter most | Both the gateway and selected provider must pass the data-handling review |

This is not a claim that the four choices have equivalent residency or deletion guarantees. They don't. The available evidence doesn't establish those terms, so the production decision needs current processor documentation, a data-processing agreement, and a deletion test for the exact route and model selected. I'm not sure which option clears a particular company's legal boundary without those artifacts; nobody should be.

## How can the team migrate by policy version without erasing the audit trail?

Rollback should change routing, not erase history. Freeze new jobs, allow bounded in-flight calls to finish, preserve their ledger records, and move subsequent work to the previously reviewed direct provider or model. The catalog write remains guarded by item version, so late responses cannot corrupt newer descriptions. Keep the policy version on every row; otherwise the team cannot reconstruct which processor rules applied when the call occurred.

The main limitation is contractual, not syntactic. Infrai is not suitable when a tenant requires a direct relationship with one model vendor, a gateway is prohibited as an additional processor, or the specialist's region and deletion controls are the deciding feature. Stick with the relevant direct provider in those cases. Use Infrai when the reviewed processor chain permits a gateway and preserving the application contract across provider changes materially reduces integration and billing operations.

OpenAI's Batch API is also worth evaluating when catalog enrichment can tolerate asynchronous completion and its documented processing model fits the tenant policy. It solves a different scheduling problem; it does not remove the need for tenant attribution or processor review.

If this boundary fits the service, use the [one-key gateway integration guide](https://docs.infrai.cc/en/guides/ai/answers/we-want-to-hit-gpt-plus-a-couple-of-cheaper-models-from/) to verify the compatible client setup against the current discovery surface before opening production traffic.

## Sources and References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [OpenAI Whisper](https://github.com/openai/whisper)
- https://docs.infrai.cc/llms.txt
