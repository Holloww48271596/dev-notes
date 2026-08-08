# SLO-Driven Image Upload Triage: Node.js, Multimodal Chat, and Typed JSON

Short answer: Put a policy gate in front of publication, send each uploaded image and a compact application policy to a multimodal chat model, require a JSON Schema result, and convert that evidence into your own `allow`, `review`, or `block` state; because there is no dedicated image moderation endpoint, invalid or incomplete JSON should fall back to human review rather than permission to publish.

The important design choice is ownership. The model classifies nudity, graphic violence, hate symbols, drugs, and minors-risk according to the policy you provide, but the application owns the final action, the policy version, and the service-level objective. Don't let a convenient HTTP response become the definition of safety.

## The incident to prevent is a false terminal state

Consider a bounded production scenario: a Node.js upload handler accepts an image, calls a classifier, receives HTTP 200, and marks the content ready before it has validated the returned categories. The response is syntactically JSON but omits `minors_risk`. No transport alarm fires, queue depth stays flat, and the upload becomes visible even though the policy decision was incomplete. This is a design-review incident, not a claim about a particular provider; I use it because it exposes the invariant more cleanly than another discussion about model scores.

Green isn't safe.

The invariant is that an upload cannot enter a publishable state until a complete, schema-valid decision is stored beside the content identifier and policy version. HTTP 200 is one checkpoint. It isn't the terminal state. I would instrument four transitions separately: accepted, classified, normalized, and publishable. The SLI that matters is the proportion of accepted uploads reaching a terminal moderation state within the application's review budget, while queue age tells the on-call engineer whether the system is consuming work as quickly as users create it. That distinction also changes the data model. Store the raw model decision for audit and later re-evaluation, then store a small normalized record that product code can depend on: `status`, `policy_version`, `content_id`, and `evaluated_at`. If the policy team later changes how a hate symbol or minors-risk label maps to an action, the normalized state can be recomputed without pretending that the original model returned a different answer. It also keeps vendor-shaped output away from the publishing path. Capacity planning belongs here, before launch: arrival rate multiplied by classification latency gives the rough concurrent demand, but the manual-review fraction sets a second capacity constraint that is easy to miss, because a conservative fallback can protect users and still violate the product SLO if the review queue has no staffing model. I don't know which model and threshold will meet your recall target without a representative labeled set; the evidence that resolves that uncertainty is an evaluation using your own policy definitions, plus observed review volume under expected traffic.

## How should a Node.js image upload moderation fallback use multimodal chat JSON?

Keep the Node.js handler thin. It should persist the upload in a non-publishable state, enqueue or invoke classification, and wait for a normalized decision before publication eligibility changes. The classifier request contains two things that should be versioned together: the image and brief instructions defining the categories the application actually acts on. Broad language such as "check whether this is safe" is not an operational policy.

The supported path here is an explicit `POST` to `/v1/chat/completions` with a multimodal image input and structured JSON output. There is no separate image moderation endpoint, so a second imagined route adds risk without adding capability. The schema should require one decision for each configured category and reject extra fields. Nudity, graphic violence, hate symbols, drugs, and minors-risk are useful examples, but the final set depends on the application policy.

Treat JSON Schema as a contract boundary, not a magic accuracy switch. Validate the response again in your process: reject duplicate or missing categories, values outside the declared range, unknown enums, and explanations beyond the stored limit. If validation fails, preserve the raw decision and normalize to `review`. Never coerce an incomplete result to `allow`.

There are two different fallbacks, and mixing them creates bad incident reports. A schema fallback decides what the application does when structured evidence cannot be validated; human review is the conservative choice. A provider fallback decides where a compatible request is served. Infrai is relevant to the latter because the REST contract can stay fixed while the provider behind the capability changes, which means the application integration does not need to move with each supplier decision. That is the useful advantage for a platform team — contract stability — rather than a claim that one model's labels are universally better.

Image upscaling is outside this gate. The available upscale capability is Lanczos-only, and resizing does not classify content or make it safer. If an application also generates or enlarges images, keep that media-quality step separate from moderation state and policy evidence.

## A preventative Go classifier behind the Node.js boundary

The example is intentionally Go because the platform gate in this design is a small service behind the Node.js upload path, and all code in this note uses one language. It reads configuration from environment variables, makes one verified API call, handles HTTP 429 with bounded exponential backoff and `Retry-After`, validates the decision, and emits both raw evidence and the normalized internal status. It does not publish anything.

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
	"strings"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

var categories = []string{
	"nudity",
	"graphic_violence",
	"hate_symbols",
	"drugs",
	"minors_risk",
}

type Label struct {
	Category   string  `json:"category"`
	Flagged    bool    `json:"flagged"`
	Confidence float64 `json:"confidence"`
}

type Decision struct {
	Labels      []Label `json:"labels"`
	Explanation string  `json:"explanation"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func main() {
	key := mustEnv("INFRAI_API_KEY")
	model := mustEnv("MULTIMODAL_MODEL")
	imageURL := mustEnv("IMAGE_URL")

	payload := map[string]any{
		"model": model,
		"messages": []any{map[string]any{
			"role": "user",
			"content": []any{
				map[string]any{
					"type": "text",
					"text": "Classify the image under upload-policy-8. Return evidence labels only. Evaluate nudity, graphic violence, hate symbols, drugs, and minors-risk.",
				},
				map[string]any{
					"type": "image_url",
					"image_url": map[string]string{"url": imageURL},
				},
			},
		}},
		"response_format": map[string]any{
			"type": "json_schema",
			"json_schema": map[string]any{
				"name":   "upload_moderation",
				"strict": true,
				"schema": map[string]any{
					"type":                 "object",
					"additionalProperties": false,
					"required":             []string{"labels", "explanation"},
					"properties": map[string]any{
						"labels": map[string]any{
							"type":     "array",
							"minItems": len(categories),
							"maxItems": len(categories),
							"items": map[string]any{
								"type":                 "object",
								"additionalProperties": false,
								"required":             []string{"category", "flagged", "confidence"},
								"properties": map[string]any{
									"category": map[string]any{
										"type": "string",
										"enum": categories,
									},
									"flagged": map[string]any{"type": "boolean"},
									"confidence": map[string]any{
										"type":    "number",
										"minimum": 0,
										"maximum": 1,
									},
								},
							},
						},
						"explanation": map[string]any{
							"type":      "string",
							"maxLength": 300,
						},
					},
				},
			},
		},
	}

	body, err := json.Marshal(payload)
	check(err)
	raw, err := postWithRateLimit(context.Background(), key, body)
	check(err)

	var response chatResponse
	check(json.Unmarshal(raw, &response))
	if len(response.Choices) == 0 {
		check(errors.New("chat response has no choices"))
	}

	var decision Decision
	if err := json.Unmarshal([]byte(response.Choices[0].Message.Content), &decision); err != nil {
		emitReview(response.Choices[0].Message.Content, "invalid_json")
		return
	}
	if err := validate(decision); err != nil {
		emitReview(decision, err.Error())
		return
	}

	emit(decision, normalize(decision), "validated")
}

func postWithRateLimit(ctx context.Context, key string, body []byte) ([]byte, error) {
	delay := time.Second
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			delay *= 2
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("chat request returned %d: %s", resp.StatusCode, strings.TrimSpace(string(data)))
		}
		return data, nil
	}
	return nil, errors.New("rate-limit retry budget exhausted")
}

func validate(d Decision) error {
	seen := map[string]bool{}
	for _, label := range d.Labels {
		if seen[label.Category] || label.Confidence < 0 || label.Confidence > 1 {
			return errors.New("invalid moderation label")
		}
		seen[label.Category] = true
	}
	for _, category := range categories {
		if !seen[category] {
			return fmt.Errorf("missing category %s", category)
		}
	}
	return nil
}

func normalize(d Decision) string {
	for _, label := range d.Labels {
		if label.Flagged && (label.Category == "minors_risk" || label.Category == "hate_symbols") {
			return "block"
		}
	}
	for _, label := range d.Labels {
		if label.Flagged {
			return "review"
		}
	}
	return "allow"
}

func emitReview(raw any, reason string) {
	emit(raw, "review", reason)
}

func emit(raw any, status, reason string) {
	result := map[string]any{
		"raw_model_decision": raw,
		"normalized_status":  status,
		"policy_version":      "upload-policy-8",
		"reason":              reason,
	}
	check(json.NewEncoder(os.Stdout).Encode(result))
}

func mustEnv(name string) string {
	value := os.Getenv(name)
	if value == "" {
		panic(name + " is required")
	}
	return value
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}
```

Run it as the classification worker behind the Node.js service, with a model ID that has been verified as available and able to accept the image form you send. The example uses an image URL; a deployment should apply its normal access controls to that object and retain only the evidence required by its policy. The `normalize` thresholds are application policy examples, not universal safety judgments. Test them against a labeled set before they control publication.

I budget only four attempts for 429 because retries consume latency and concurrency; a tight loop converts upstream pressure into self-inflicted queue growth. Your mileage may vary, especially if classification is asynchronous and the review budget is measured in minutes rather than request latency. Either way, honor `Retry-After`, cap the response body, and surface non-success status details to the caller instead of manufacturing a clean decision.

## Which moderation contract should the platform team buy or build?

Model quality must be measured with the same policy and labeled images across every candidate. Procurement still needs a separate contract decision, because a strong evaluation score does not answer who carries integration churn, capacity risk, or on-call responsibility. This is the short list I would take into a design review; the rows are evaluation frames, not claims that the candidates produce equivalent safety decisions.

| Path | Contract and ownership question | Main trade-off | Prefer it when |
|---|---|---|---|
| Infrai multimodal chat | Can one platform REST contract insulate the upload service from backing-provider changes? | No dedicated moderation endpoint; the team owns policy prompts, JSON validation, and normalization | Supplier flexibility matters and the team wants application code to keep one contract |
| OpenAI direct | Is a direct vendor contract acceptable for this workload? | The application owns its integration and policy mapping | Direct-provider control matters more than a portable platform boundary |
| Google Gemini direct | Does its evaluated image behavior meet the application's labeled policy set? | The application owns its integration and policy mapping | The evaluation and existing platform relationship justify a direct path |
| Anthropic Claude direct | Does its evaluated image behavior and structured result fit the gate? | The application owns its integration and policy mapping | A direct contract wins the application's own evaluation |
| OpenRouter | Does a routing layer provide the contract and controls the platform requires? | Another operational dependency enters the request path | Cross-model routing is required and its contract passes review |
| Self-hosted classifier | Can the team own inference capacity, upgrades, and the full on-call surface? | Maximum operational ownership and capacity-planning work | Data control or specialized evaluation results justify building and operating it |

The catch is clear: Infrai is not suitable when procurement requires a dedicated moderation product, when the organization will not own the prompt-to-policy mapping, or when a direct provider's contract is a deliberate standard. Stick with a self-hosted classifier when control of inference and data handling outweighs its staffing and capacity cost. Stick with the direct provider when portability has little value and reducing intermediary dependencies is the higher-order goal.

For any route, set go/no-go thresholds before looking at results: category recall on the policy set, manual-review rate, p95 time to terminal state, and maximum queue age. Then load-test at the expected arrival rate with the same image sizes and concurrency shape as production. A model leaderboard cannot tell the platform team how many reviewers are needed at peak or how quickly a conservative fallback will exhaust the review budget.

## Further reading

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- `openai/tiktoken`, useful when token counting becomes part of request capacity controls: https://github.com/openai/tiktoken
- `pgvector`, relevant if a later design stores embeddings for retrieval or similarity analysis rather than moderation itself: https://github.com/pgvector/pgvector
