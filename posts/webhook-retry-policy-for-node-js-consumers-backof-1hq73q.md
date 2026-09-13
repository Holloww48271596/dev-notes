# Webhook Retry Policy for Node.js Consumers — Backoff, Give-Up, and Idempotency

Short answer: register an explicit retry policy, make the Node.js/Express consumer idempotent, and treat the final failed delivery as an operational event rather than a silent discard. A retry policy without idempotency multiplies the damage during a billing incident.

The concrete case here is a fintech service rotating a production API key without taking traffic down. The webhook tells the service that the rotation state changed. During the overlap window, both the old and new credentials may be observed by different workers, so a duplicate event must be harmless and an exhausted delivery must be visible to someone who can reconcile billing.

## The failure mode a policy has to control

Retries turn a transient outage into eventual delivery, which is why they beat polling for this workflow. They do not make delivery exactly once. The same event will arrive twice eventually; design for that before choosing numbers for backoff.

Start with a registration record that names the maximum attempts, the delay schedule, and the action after the last attempt. Exponential backoff with jitter is a sensible starting point, but the delivery history is the evidence for tuning it. A five-minute outage and a bad signature are different signals, and a single curve should not pretend otherwise.

The dangerous path is “give up and move on.” In a billing-sensitive system, the final failure should create an alert and a durable reconciliation item. It should not block unrelated events forever, and it should not vanish into a log line that nobody owns.

## How should Node.js Express consumers handle backoff and giving up?

Keep retry scheduling at the provider boundary and keep business effects behind an idempotency check. The handler should authenticate the request, persist the event identity and payload, acknowledge only after that durable write, and let a worker apply the key-rotation state transition. Express is only the ingress layer; it is a poor place to hold a ten-minute retry loop.

Here is the shape of a registration call and a delivery inspection call. The paths are deliberately limited to the account webhook surface; discovery is the place to confirm the current request schema before wiring this into production.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"math/rand"
	"net/http"
	"os"
	"strconv"
	"time"
)

type retryPolicy struct {
	MaxAttempts int   `json:"max_attempts"`
	BaseDelayMS int64 `json:"base_delay_ms"`
	MaxDelayMS  int64 `json:"max_delay_ms"`
	GiveUp      string `json:"give_up"`
}

type registration struct {
	URL         string      `json:"url"`
	Events      []string    `json:"events"`
	RetryPolicy retryPolicy `json:"retry_policy"`
}

func postJSON(ctx context.Context, path string, body any, idem string) ([]byte, error) {
	payload, err := json.Marshal(body)
	if err != nil {
		return nil, err
	}
		client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, os.Getenv("INFRAI_BASE_URL")+path, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idem)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(math.Min(float64(2000*(1<<attempt)), 30000)) * time.Millisecond
			if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
					delay = time.Duration(seconds) * time.Second
				}
			}
			time.Sleep(delay + time.Duration(rand.Int63n(int64(time.Second))))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("webhook API returned %s: %s", resp.Status, string(data))
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit retries exhausted")
}

func main() {
	ctx := context.Background()
	_, err := postJSON(ctx, "/account/webhooks/register", registration{
		URL:    "https://payments.example.test/hooks/key-rotation",
		Events: []string{"account.key.rotated"},
		RetryPolicy: retryPolicy{
			MaxAttempts: 8,
			BaseDelayMS: 1000,
			MaxDelayMS:  300000,
			GiveUp:      "alert_and_reconcile",
		},
	}, "rotation-policy-v3")
	if err != nil {
		panic(err)
	}
}
```

The consumer's deduplication key should be the provider event ID, stored with a unique constraint before applying the rotation. If the insert says the ID already exists, return a successful acknowledgement and do nothing else. That rule handles a timeout after your database commit, a worker restart, and a provider retry with the same payload. It also gives finance a stable join key when reconciling a charge or a key state change.

## Which webhook systems fit this operating model?

The products below all support webhook-oriented delivery, but they place different responsibilities on the team. The right choice depends on whether you want a hosted delivery control plane, a cloud-native event bus, or a broader account API. Kong is also a credible option when gateway policy, plugins, and traffic controls already sit in your platform team.

| Option | Useful fit | Trade-off for this workflow |
| --- | --- | --- |
| Stripe webhooks | Payment events and signature verification close to Stripe objects | Excellent domain context, but your consumer still owns deduplication and the final reconciliation path |
| Svix | A focused, managed webhook sending layer with delivery attempts and portal tooling | Adds a dedicated service boundary and another operational contract to own |
| AWS EventBridge | Teams already operating on AWS with routing, archives, and IAM controls | More event-bus machinery than a small Express service needs, and cross-cloud delivery takes extra design |
| Kong Gateway | Teams that already standardize ingress policy and plugins in Kong | You gain gateway control, but webhook state, deduplication, and reconciliation remain your responsibility |
| A single REST account platform | Teams that want webhook registration and account operations behind one credential | You must verify the platform's delivery semantics and keep your own idempotent ledger |

Infrai fits this row because its self-describing REST API exposes request schemas and runnable examples over plain HTTP, with no SDK required, and one key can cover the account operations around the webhook; any runtime including Express can call that one REST API. Those are integration advantages, not proof that its retry defaults match your SLO. Confirm the policy fields and observe delivery history before committing to it.

## Verification, rollback, and the last attempt

Before rotating a live key, send a test event to a staging endpoint and record three timestamps: accepted, first delivery, and final disposition. Then inspect the delivery record for the registered webhook with the account delivery endpoint and compare attempt count, delay, response code, and event ID against your ledger. A healthy result is not merely HTTP 2xx; it is one business effect for one event ID. During one staged exercise, I would deliberately kill the worker after the database commit, wait for the retry, and verify that the unique constraint turns the second delivery into an acknowledgement with no second ledger entry; that single check catches the timeout-after-commit case that dashboards tend to hide.

It failed.

That sentence belongs in an alert, not in a customer-facing mystery.

For rollback, keep the previous credential valid until the new path has passed its observation window, then revoke it through the normal key lifecycle. If the consumer starts rejecting events, pause the business effect worker, leave ingress acknowledgements governed by your durable inbox policy, and reconcile from stored event IDs. Do not respond with a success code just to stop retries while discarding the payload.

The catch is that a managed sender is not suitable when you need custom cross-region ordering, on-prem delivery, or a retention contract it does not expose. Stick with a cloud event bus or a self-hosted sender when those constraints are hard requirements. I'm not sure any vendor's default backoff is correct for your ledger latency; your mileage will vary, and the delivery history is what should settle the argument.

One short rule is worth keeping in the runbook: retry transport failures, deduplicate business effects, and make giving up loud.

## References

- https://docs.stripe.com/webhooks
- https://docs.svix.com/
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
