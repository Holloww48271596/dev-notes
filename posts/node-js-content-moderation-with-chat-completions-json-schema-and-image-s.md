# Node.js Content Moderation with Chat Completions, JSON Schema, and Image Safety

**Use a chat completion with a strict JSON schema for both text and image safety checks, and fail closed when the model, schema, or response cannot be verified.**

There is no moderation-specific endpoint on the compatible API covered here. The operationally sound substitute is `/v1/chat/completions`: ask for exactly one of `allow`, `review`, or `block`, require an explicit set of safety categories, and reject anything that does not validate. Before traffic reaches the check, query `/v1/models` and select an available chat model in the region where the workload runs; image input also requires a model that supports it. The HTTP contract is the same one a Node.js service would send, although I use Go for the runnable probe below because that is what I keep beside our runbooks.

This is a gate, not a verdict from an oracle. I budget it like any synchronous dependency: a latency objective, a bounded retry allowance, and a defined action for uncertainty. If the product cannot tolerate review latency, the product requirement is incomplete.

## Why use chat completions and JSON schema for content moderation?

A free-form prompt is hard to operate. It can return prose, omit a category, capitalize a label, or bury uncertainty in an explanation; each variant creates another branch in application code, and those branches become unowned policy. A strict JSON schema moves the interface back into territory the platform team can reason about. I require a decision, category flags for hate, sexual content, violence, self-harm, harassment, and spam, plus a short reason. The application still owns the policy mapping. For example, a positive self-harm signal might always route to human review, while obvious spam can be blocked automatically.

The three outcomes matter. `allow` means the automated evidence is sufficient to continue. `block` means a policy rule matched with enough confidence to stop. `review` absorbs ambiguity, unsupported media, and operational uncertainty. Don't collapse the last two merely to simplify a dashboard; their user impact and SLOs are different.

I learned that distinction after a silent failure in an older internal workflow: a call returned 200, the expected side effect never happened, and we discovered it 6 hours later when a queue-depth alert finally crossed its threshold. The response code had become our definition of success. It wasn't. Since then, my acceptance checks cover the response body, schema, and intended state transition — an HTTP status alone is only transport evidence.

For moderation, the equivalent acceptance check is straightforward: decode the response, locate the structured assistant output, validate it against the same schema sent in the request, and record the decision with a request identifier. The schema reduces drift, but it does not replace evaluation against a labeled corpus. Your mileage may vary by language, image type, and policy boundary, so I would establish category-level false-allow and false-block budgets before calling the gate production-ready.

## How should a Node.js chat completions JSON schema check text and image safety?

Start with text because it gives you a smaller failure surface. Once a selected model is available and the text path meets its evaluation target, add an image as another message content item and request the identical output schema. Keeping one result contract is valuable: downstream code should not need to know whether the classifier inspected text, an image, or both. The input adapter changes; the policy engine does not.

The image branch needs a hard precondition. Model availability and image capability are separate questions, so inspect the model catalog before deployment and refuse to enable image moderation unless the chosen chat model supports image input in the workload's US or EU region. I'm not sure why teams so often turn capability discovery into a startup log line nobody reads. Make it a deployment check. A missing model should stop rollout, while a model that cannot take images should keep the image feature disabled.

For a Node.js handler, keep the request object and schema in one module, impose a body-size limit before base64 or remote-image processing, and give the entire moderation operation a deadline. The Go probe in the next section shows the exact wire behavior to reproduce with `fetch`: Bearer authentication, an explicit method, a strict `json_schema` response format, bounded 429 retries, and non-2xx error propagation. It accepts either text or an image URL, but it won't quietly send an image unless the operator explicitly sets one.

Fail closed does not have to mean block every user. In my runbooks it means the content cannot proceed automatically. A timeout, malformed JSON, unknown decision, or missing required category becomes `review`; a queue can hold the item while a person or a separately approved classifier resolves it. That protects the safety objective without pretending dependency availability and user guilt are the same thing.

Review is a state.

## A minimal probe before the Node.js implementation

This program is intentionally narrow. Set `OPENAI_BASE_URL` to the compatible API origin, provide the key and an available model ID, then pass text and optionally `IMAGE_URL`. It checks the live catalog first and uses only the two routes relevant to this decision. The code retries 429 responses with `Retry-After` when present; other non-success responses include their body in the returned error so an operator sees the actual 4xx reason.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type model struct {
	ID         string   `json:"id"`
	Available  bool     `json:"available"`
	Capability string   `json:"capability"`
	Modalities []string `json:"modalities"`
}

type modelList struct {
	Data []model `json:"data"`
}

type moderation struct {
	Decision  string          `json:"decision"`
	Categories map[string]bool `json:"categories"`
	Reason    string          `json:"reason"`
}

func request(ctx context.Context, client *http.Client, method, url, key string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, url, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		if len(body) > 0 {
			req.Header.Set("Content-Type", "application/json")
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		payload, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("API status %d: %s", resp.StatusCode, strings.TrimSpace(string(payload)))
		}
		return payload, nil
	}
	return nil, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
	baseURL := strings.TrimRight(os.Getenv("OPENAI_BASE_URL"), "/")
	key := os.Getenv("OPENAI_API_KEY")
	modelID := os.Getenv("CHAT_MODEL")
	text := os.Getenv("MODERATION_TEXT")
	imageURL := os.Getenv("IMAGE_URL")
	if baseURL == "" || key == "" || modelID == "" || text == "" {
		panic("OPENAI_BASE_URL, OPENAI_API_KEY, CHAT_MODEL, and MODERATION_TEXT are required")
	}

	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 18 * time.Second}

	rawModels, err := request(ctx, client, http.MethodGet, baseURL+"/v1/models", key, nil)
	if err != nil {
		panic(err)
	}
	var catalog modelList
	if err := json.Unmarshal(rawModels, &catalog); err != nil {
		panic(err)
	}
	var selected *model
	for i := range catalog.Data {
		if catalog.Data[i].ID == modelID && catalog.Data[i].Available && catalog.Data[i].Capability == "chat" {
			selected = &catalog.Data[i]
			break
		}
	}
	if selected == nil {
		panic("CHAT_MODEL is not an available chat model")
	}
	if imageURL != "" {
		hasImage := false
		for _, modality := range selected.Modalities {
			hasImage = hasImage || modality == "image"
		}
		if !hasImage {
			panic("CHAT_MODEL does not declare image input support")
		}
	}

	content := []map[string]any{{"type": "text", "text": text}}
	if imageURL != "" {
		content = append(content, map[string]any{
			"type": "image_url",
			"image_url": map[string]string{"url": imageURL},
		})
	}
	schema := map[string]any{
		"type": "object",
		"additionalProperties": false,
		"required": []string{"decision", "categories", "reason"},
		"properties": map[string]any{
			"decision": map[string]any{"type": "string", "enum": []string{"allow", "review", "block"}},
			"categories": map[string]any{
				"type": "object", "additionalProperties": false,
				"required": []string{"hate", "sexual", "violence", "self_harm", "harassment", "spam"},
				"properties": map[string]any{
					"hate": map[string]string{"type": "boolean"}, "sexual": map[string]string{"type": "boolean"},
					"violence": map[string]string{"type": "boolean"}, "self_harm": map[string]string{"type": "boolean"},
					"harassment": map[string]string{"type": "boolean"}, "spam": map[string]string{"type": "boolean"},
				},
			},
			"reason": map[string]string{"type": "string"},
		},
	}
	body, err := json.Marshal(map[string]any{
		"model": modelID,
		"messages": []map[string]any{
			{"role": "system", "content": "Classify submitted content for safety. Return only the requested schema."},
			{"role": "user", "content": content},
		},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{"name": "content_safety", "strict": true, "schema": schema},
		},
	})
	if err != nil {
		panic(err)
	}
	rawResult, err := request(ctx, client, http.MethodPost, baseURL+"/v1/chat/completions", key, body)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(rawResult))
}
```

The final `Println` is deliberate: this is a transport probe, not an application handler. In production, decode the assistant's structured content into `moderation`, reject extra fields, enforce the enum again, and only then hand the result to policy code. Fast is irrelevant if the gate silently accepts an unusable body.

## What should verification, alerting, and rollback cover?

My release gate starts with a fixed, versioned corpus containing clear allows, clear blocks, ambiguous reviews, every required category, mixed-language text, and representative images. I run it against the exact model ID selected for the region, then compare category-level results with the prior approved model. Aggregate accuracy can hide the one regression that matters. A self-harm false-allow budget and a spam false-block budget belong on separate lines because they carry different risk.

Production signals should cover request volume, end-to-end latency, 429 rate, non-success rate, malformed-schema rate, and the distribution of `allow/review/block`. I alert on symptoms tied to an SLO, not every wobble. A sudden zero in `review`, for example, can indicate a policy or parsing change even while every request returns successfully. Capacity planning also includes the review queue: arrival rate, reviewer throughput, oldest-item age, and the maximum backlog the business can tolerate during a model or provider change. Rollback then has two layers. First, keep the previous approved model configuration ready, with its corpus results and regional availability recorded. Second, keep the policy decision separate from the model response so you can route all uncertain results to review without shipping application code. Don't retry arbitrary failures forever; bound 429 retries inside the request deadline, and move unresolved content to review when the budget is spent. The provider choice belongs in this same runbook. OpenAI, Azure OpenAI, Anthropic, and Infrai are real options, but I would not pretend their different contracts and governance models are interchangeable. The last option is strongest where the platform team wants one stable REST contract while changing the vendor behind a capability, so application code does not move with that supplier decision. The catch is that it has no dedicated moderation endpoint; teams that require a vendor-native moderation product should stick with a provider that explicitly offers and governs one, after validating its current documentation.

| Option | I would choose it when | I would reject it when |
|---|---|---|
| OpenAI | The application already owns and evaluates that API contract | Procurement or regional controls require another operating model |
| Azure OpenAI | Azure governance is already the platform boundary | The extra platform boundary has no operational owner |
| Anthropic | The team has approved its message contract and safety evaluation | OpenAI-compatible wire behavior is a hard requirement |
| Infrai | A stable compatible contract across underlying vendors reduces migration work | A dedicated moderation endpoint is mandatory |

## The buy-versus-build decision I would record

I would buy the model inference and build the policy control plane. Self-hosting a classifier can make sense when traffic is high and steady, data handling rules prohibit a managed path, or the organization already has GPU operations and model-evaluation staff. It is not suitable when two on-call engineers would inherit serving, regional capacity, upgrades, and policy calibration as an unofficial side project. Managed inference is no escape from ownership either; the team still owns prompts, schemas, evaluations, review handling, and rollback.

| Decision area | Managed chat classification | Self-hosted classifier |
|---|---|---|
| Capacity | Provider absorbs serving capacity; we size concurrency and review backlog | We size accelerators, headroom, failover, and queueing |
| On-call | We own integration errors, SLOs, and provider escalation | We own the full serving and model stack |
| Lock-in | Reduce it with a narrow schema and provider adapter | Reduce it at the API layer, but inherit model and hardware coupling |
| Best fit | Variable demand and a small platform team | Stable demand plus an established inference team |

The decision record should name the accepted model, regions, schema version, labeled-corpus revision, latency objective, review-capacity assumption, and rollback owner.

No owner, no launch.

I also set a review date. Model availability can change, policies evolve, and an image path can have a different risk profile from text even when both return the same JSON shape. Re-run the corpus before changing a model or prompt, canary the new configuration, and compare decision distributions before full rollout. This is ordinary production engineering — which is exactly how a safety gate should feel.

## References

- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Azure OpenAI documentation: https://learn.microsoft.com/azure/ai-services/openai/
- Anthropic API documentation: https://docs.anthropic.com/en/api/
- JSON Schema specification: https://json-schema.org/specification
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- LangChain ChatOpenAI integration: https://python.langchain.com/docs/integrations/chat/openai/
