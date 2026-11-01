# One API Key, Multiple LLM Providers: Gateway Routing for Text Classification Review

Short answer: put the gateway behind a stable review contract, then validate every provider response before it can affect a checkout change. A single credential can simplify rotation and routing, but it does not make OpenAI, Claude, and Gemini interchangeable. The engineering problem is preserving a typed finding when a provider declines, truncates, changes its schema behavior, or returns a syntactically valid response with the wrong meaning.

I treat this as a bounded production scenario: a pull request changes coupon and payment code, an e-commerce service sends the diff and selected repository context for classification, and the reviewer needs findings that downstream policy can consume. The output is not a paragraph for a human to interpret later. It is a record with severity, file, line, category, explanation, and confidence. That distinction drives the design.

## Can one API key coordinate multiple LLM providers safely?

A gateway can unify authentication, request accounting, timeout policy, and a provider-neutral entry point. It cannot unify model behavior. Structured-output interfaces across OpenAI, Anthropic, and Google expose different controls for tools, schemas, refusals, truncation, and safety, so each still needs an adapter boundary.

For a review pipeline, I would expose one internal operation, `ClassifyReview`, and keep provider details below it. The operation owns the contract, request ID, deadline, and validation result. Routing chooses a candidate based on measured quality and capacity, not on a permanent claim that one model is cheapest.

The useful comparison is therefore operational:

| Boundary | What a gateway may standardize | What the application must still own |
| --- | --- | --- |
| Credential | One application credential and rotation path | Provider authorization, scopes, and audit mapping |
| Request | Common messages, timeout, and request ID | Context limits, prompt assembly, and redaction |
| Response | Decode attempt and normalized error class | Schema validation, semantic checks, and retry safety |
| Routing | Candidate selection and fallback policy | Evaluation set, SLO budget, and release approval |

A hosted aggregator can shorten integration work. Self-hosting can give the platform team more control over logs, data residency, and routing policy. Both still leave the application responsible for proving that `severity: "low"` means the same thing after a provider switch. That is the handoff I would put on the critical path.

## The failure is usually a valid response

The dangerous case is not always a 5xx. A response can parse as JSON and still be unusable: a finding can name a line outside the diff, classify a security issue as a style issue, or omit a required remediation because the model treated a repository comment as an instruction. A retry only repeats that ambiguity.

That is the trap.

I use three gates. First, decode the envelope and reject unknown or missing required fields. Second, apply semantic checks: line numbers must belong to the submitted diff, severity must be one of the policy values, and a finding marked actionable must contain an explanation. Third, run a small deterministic consistency check against the review request, such as requiring the returned language and repository identifier to match the request metadata.

The adapter should return a typed failure, not a half-filled review. Here is the core shape in Go; the provider client is deliberately abstract so the contract can be tested without a commercial endpoint.

```go
package review

import (
	"context"
	"encoding/json"
	"fmt"
)

type Finding struct {
	File        string `json:"file"`
	Line        int    `json:"line"`
	Severity    string `json:"severity"`
	Category    string `json:"category"`
	Explanation string `json:"explanation"`
	Actionable  bool   `json:"actionable"`
}

type ReviewResult struct {
	Repository string    `json:"repository"`
	Findings   []Finding `json:"findings"`
}

func ValidateResult(raw []byte, repository string, changedLines map[string]map[int]bool) (ReviewResult, error) {
	var result ReviewResult
	if err := json.Unmarshal(raw, &result); err != nil {
		return ReviewResult{}, fmt.Errorf("decode review result: %w", err)
	}
	if result.Repository != repository {
		return ReviewResult{}, fmt.Errorf("repository mismatch: got %q", result.Repository)
	}
	for _, finding := range result.Findings {
		if finding.File == "" || finding.Explanation == "" {
			return ReviewResult{}, fmt.Errorf("finding has required fields missing")
		}
		if finding.Severity != "low" && finding.Severity != "medium" && finding.Severity != "high" {
			return ReviewResult{}, fmt.Errorf("invalid severity %q", finding.Severity)
		}
		if finding.Actionable && finding.Line < 1 {
			return ReviewResult{}, fmt.Errorf("actionable finding has no valid line")
		}
		if lines, ok := changedLines[finding.File]; ok && !lines[finding.Line] {
			return ReviewResult{}, fmt.Errorf("line %d is outside the submitted diff", finding.Line)
		}
	}
	return result, nil
}

func ClassifyReview(ctx context.Context, provider Provider, request ReviewRequest) (ReviewResult, error) {
	raw, err := provider.Generate(ctx, request)
	if err != nil {
		return ReviewResult{}, err
	}
	return ValidateResult(raw, request.Repository, request.ChangedLines)
}
```

The `changedLines` check is intentionally strict for actionable findings. Some teams may want file-level findings or comments on nearby context; that is a policy choice, and it belongs in the contract rather than appearing accidentally during fallback.

## How should fallback spend its error budget?

Fallback is a reliability mechanism, not a quality strategy. I set an end-to-end SLO for review completion and a separate correctness SLO for accepted structured results. A provider timeout consumes latency budget. A schema violation consumes correctness budget and should be visible even when the second candidate succeeds. Otherwise the dashboard reports availability while the review queue quietly changes shape.

The candidate table needs more than price. I would record schema-pass rate, semantic-pass rate on a fixed evaluation set, p95 latency, timeout rate, refusal rate, and context-limit failures. A lower request price does not compensate for a higher rate of invalid findings if every invalid result triggers another call and a human re-review.

A simple policy is to try one candidate, then one fallback only when the failure class is retryable or the result fails validation. Do not retry a request rejected for policy or an oversized input without changing the input. Keep the original request ID and add an attempt number so an auditor can distinguish a fallback from a duplicate review. Cap the total deadline; a checkout merge queue cannot wait for an unbounded chain of providers. I initially treated a second provider as extra availability, but the more important question is whether its output can be judged by the same contract without hiding the first failure.

I also keep model selection out of pull-request business logic. A routing layer can compare OpenAI, Anthropic Claude, and Google Gemini as candidates, but each candidate must pass the same golden cases and contract tests. A gateway-style interface can put several upstream models behind one API surface; that convenience still does not remove the need to inspect the normalized response and its upstream metadata. The relevant difference is control placement: a managed gateway carries part of the integration and routing burden, while a self-hosted adapter carries more operational work and gives the platform team direct control of the policy path.

The limitation is important: this design is a poor fit when the organization cannot send repository context to a shared service, when provider-specific controls are required for every request, or when the team cannot operate a validation and audit path. In those cases, direct, separately governed adapters are the clearer boundary, even though they duplicate some plumbing.

## What should the rollout measure?

Start with shadow classification on historical, redacted diffs. Compare normalized findings with an adjudicated set, and classify disagreements by missing finding, false positive, wrong category, invalid location, and malformed output. Keep the set versioned. A prompt edit or provider change should produce a new evaluation result before it reaches merge protection.

During the first live phase, let the existing review path decide and use the new result for measurement. Then gate only a narrow category, such as high-severity payment or authorization findings, after the semantic-pass SLO has held for a defined observation window. This is slower than switching every repository at once, but it gives the on-call engineer a rollback that is a routing change rather than a code rewrite.

Log hashes and classifications, not raw secrets or unrestricted source context. Propagate the request ID, provider candidate, attempt number, validation outcome, latency, and token accounting needed for capacity planning. Retain enough evidence to explain a finding, while applying repository and customer-data retention rules before central logging.

The decision rule is straightforward: use a gateway when centralized credentials, routing, and accounting reduce work that your team does not need to own; use direct adapters when data controls or provider-specific behavior are first-order requirements; in either case, keep the review contract and validators in your service. The gateway is an integration boundary. It is not the authority on whether a code-review finding is correct.

## Sources

- https://platform.openai.com/docs/guides/function-calling
- https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- https://ai.google.dev/gemini-api/docs/structured-output
- https://openrouter.ai/docs
- https://datatracker.ietf.org/doc/html/rfc9457
