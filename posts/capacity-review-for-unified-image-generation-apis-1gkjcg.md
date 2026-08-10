# Capacity Review for Unified Image Generation APIs: One Key, Multiple AI Models

**Short answer:** The simplest text-to-image API is the one that gives the application a small, testable contract and gives the platform team a credible exit path; a shared credential and access to multiple AI models reduce integration work, but neither feature proves that image generation will meet an SLO. **Choose on lifecycle semantics, observable output, capacity controls, and portability before counting models.**

I learned the boundary during a production launch: an image request returned HTTP 200, our worker marked the job complete, and the expected image side effect was absent. An editor found the empty asset slot 6 hours later. No alert fired because the service-level indicator measured transport success rather than a usable image. The model wasn't the useful unit of analysis. The end-to-end contract was.

That incident is bounded evidence, not a universal benchmark. It exposed one invariant that applies to any gateway or direct integration: success means the intended consumer can retrieve and validate the image, not merely that an upstream request completed. Everything below follows from that distinction.

## How should one key span multiple AI models in a text-to-image API?

A unified image generation API should normalize the parts that the application must reason about consistently: authentication, request identity, lifecycle, cancellation, result location, error classification, and output metadata. It should not pretend that every model has the same controls. A common envelope can carry a prompt, a model selector, requested output constraints, and an idempotency token while still allowing the adapter to reject a parameter that the selected model cannot honor.

One key is an operational convenience. It reduces credential distribution and rotation paths, which is useful, but it also concentrates blast radius unless scope, tenant separation, and revocation are designed deliberately. I want to know whether a credential can be constrained by environment and workload, how rotation affects in-flight jobs, and which audit record ties a caller to a request. A demo that generates an attractive image answers none of those questions.

The common contract needs explicit states such as accepted, running, succeeded, failed, and cancelled. The exact labels can vary. The transitions can't be vague. A synchronous interface may collapse the first two states, while an asynchronous interface may expose polling or completion events, but a terminal success still needs a non-empty result, a declared media type, and enough metadata for the consumer to decide whether it received what it requested. Unsupported options should fail before dispatch or be discoverable through capabilities; silently discarding them makes routing behavior impossible to test.

Keep the abstraction narrow.

This is where claims about OpenAI, Claude, Gemini, or any alternative tend to distract the review. A model name is evidence about a route target, not evidence that prompts, safety outcomes, seeds, aspect constraints, or edit operations are portable. I'm not sure a portability claim means much until the same contract suite runs against at least two adapters and the team documents every intentional difference. That test, plus a review of the native specifications for the exact image models under consideration, would resolve the uncertainty.

## Test the result, not the request

The preventative code path belongs at the adapter boundary. It should preserve one logical request identity across retries, apply a deadline to the whole operation, reject non-successful transport responses, constrain the amount of data read, and validate the declared content type and decoded image. The caller supplies the endpoint because a generic example should not invent a commercial route.

```go
package imageport

import (
	"bytes"
	"context"
	"fmt"
	"image"
	_ "image/jpeg"
	_ "image/png"
	"io"
	"net/http"
	"strings"
)

const maxImageBytes = 25 << 20

type Request struct {
	Prompt         string
	Model          string
	IdempotencyKey string
}

func Generate(ctx context.Context, client *http.Client, endpoint, token string, in Request) ([]byte, error) {
	body := []byte(fmt.Sprintf(
		`{"prompt":%q,"model":%q}`,
		in.Prompt,
		in.Model,
	))
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
	if err != nil {
		return nil, fmt.Errorf("create image request: %w", err)
	}
	req.Header.Set("Authorization", "Bearer "+token)
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Idempotency-Key", in.IdempotencyKey)

	resp, err := client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("send image request: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return nil, fmt.Errorf("image request returned %s", resp.Status)
	}

	mediaType := strings.ToLower(resp.Header.Get("Content-Type"))
	if !strings.HasPrefix(mediaType, "image/png") && !strings.HasPrefix(mediaType, "image/jpeg") {
		return nil, fmt.Errorf("unexpected content type %q", mediaType)
	}
	data, err := io.ReadAll(io.LimitReader(resp.Body, maxImageBytes+1))
	if err != nil {
		return nil, fmt.Errorf("read image: %w", err)
	}
	if len(data) == 0 || len(data) > maxImageBytes {
		return nil, fmt.Errorf("image size %d is outside the accepted range", len(data))
	}
	if _, _, err := image.DecodeConfig(bytes.NewReader(data)); err != nil {
		return nil, fmt.Errorf("decode image metadata: %w", err)
	}
	return data, nil
}
```

Short code. Hard boundary.

This example assumes a synchronous response containing image bytes, so it is a contract shape rather than a claim about any named service. For an asynchronous API, the same validation runs after a terminal completion event or poll response, and the durable record maps the application's asset ID to the upstream job ID. A production implementation should serialize JSON with `encoding/json` rather than assemble it with formatting; the omitted schema is where an adapter must translate its verified upstream contract.

Contract tests should cover duplicate submission, cancellation, malformed image data, an unsupported media type, a deadline, and a successful result that the downstream decoder can open. They should also establish the documented behavior of idempotency. Don't infer that repeating a POST is harmless. Preserve the logical request key, bound attempts, add jitter to retry timing, and retry only error classes that the selected interface documents as transient.

Observability follows the same boundary. Record request and job identifiers, model selection, state transitions, queue time, generation time, output validation, and outcome class. Avoid copying prompts into general logs because user text can be sensitive. The alert should fire on user-visible impact: a terminal job without a validated asset, or an end-to-end latency objective at risk. Upstream response time is diagnostic data, not the service promise.

## Capacity planning is part of API selection

Image workloads punish averages. Editorial imports, campaign launches, and regeneration batches create bursts, while image size and model choice can change service time, so a capacity review needs an arrival-rate distribution, service-time distribution by routing class, maximum useful completion age, and the concurrency or quota behavior of every dependency. Arrival rate multiplied by mean service time is a starting estimate for required concurrency, not a capacity guarantee; tail latency, retries, and recovery headroom still need explicit treatment.

My preferred SLI is accepted-to-validated-image latency. The clock starts when the application accepts work and stops only after the intended consumer can access a decoded image. Queue delay counts. Storage handoff counts. If the product has a deadline, one context should carry that budget through admission, dispatch, generation, validation, and persistence rather than granting a fresh timeout at every hop.

Backpressure must be visible. An unbounded queue converts overload into stale work, and stale creative output is often equivalent to failed work because the publishing window has passed. Define an admission threshold, expose a retryable overload category only when retrying later is meaningful, and track queue age alongside queue depth. Then load-test the adapter against a controlled fake implementation, where latency, cancellation, malformed output, and capacity exhaustion can be produced deterministically without spending model capacity or confusing application defects with upstream behavior.

Capacity isolation also changes the one-key calculation. Shared credentials and shared quotas may be acceptable for a single trusted workload; they are a poor default when one tenant's batch can consume another tenant's interactive budget. The review should ask for per-workload limits, fairness behavior, and emergency admission controls. If the interface cannot provide the isolation the SLO requires, separate accounts or a self-managed scheduler may be the clearer architecture even though they add operational work.

## Buy, build, or integrate directly?

Platform decisions get slippery when every option is described as flexible, so I force the ownership boundary into a table. The relevant cost is not a hypothetical per-image saving. It is engineering delivery, compatibility work, security review, observability, incident response, and the opportunity cost of carrying that system on the roadmap.

| Approach | Team owns | Good fit | Limitation |
|---|---|---|---|
| Managed gateway | Application policy, contract tests, vendor review, exit plan | A small platform team that values one control surface | Common semantics may lag provider-native controls |
| Self-managed gateway | Authentication, routing, queues, retries, upgrades, telemetry, on-call | Workloads with unusual isolation or scheduling policy | The platform team owns the full operational lifecycle |
| Direct adapters | Separate credentials, clients, error maps, quotas, and dashboards | A product coupled to native model behavior | Integration and review work grows with each provider |

Lock-in is measurable here: count the proprietary request fields exposed to application code, lifecycle states embedded in workflows, stored URLs with provider-specific access rules, and routing policies that cannot be exported. A thin internal port limits that surface. The application depends on a small domain contract, adapters translate verified native behavior, and contract tests make replacement work visible before a procurement deadline.

The catch is real: a lowest-common-denominator gateway is not suitable when provider-native controls are the product. A creative application that must expose new model parameters immediately should stick with direct adapters and normalize only lifecycle, identity, and telemetry. Strict data-location, tenant-isolation, or audit requirements can also rule out an intermediary unless documented controls satisfy the review. Conversely, self-hosting is hard to justify when the team cannot staff the pager and has no routing policy that requires owning the control plane.

There is no universal winner. I would approve the option whose contract tests pass, capacity limits match the workload, security boundary survives review, and exit exercise produces a believable estimate. Your mileage may vary because on-call capacity and native-feature demand are organization-specific, but those variables should be written down rather than hidden behind a model catalog.

## Ship evidence, not a logo matrix

The go-live gate should require a passing adapter contract suite, a load test at the planned peak with stated headroom, dashboards for end-to-end success and latency, an alert tied to user impact, and a runbook for stopping admission and draining in-flight work without duplicating jobs. Deployment needs a canary and rollback plan that defines what happens to active requests. Security review needs credential scope and rotation, prompt-retention policy, output access, auditability, and tenant boundaries.

Practice the path.

The final selection is therefore modest: choose the smallest image-generation contract that can prove a usable result, withstand the expected queue, and be replaced at a cost the team has actually estimated. One credential and multiple models can reduce friction, but the SLO and ownership boundary decide whether the API is simple in production.

## References

These sources cover adjacent retrieval components and do not specify image-generation API behavior; the exact native image-model specifications still need review during vendor evaluation.

- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- pgvector, Postgres vector similarity extension: https://github.com/pgvector/pgvector
