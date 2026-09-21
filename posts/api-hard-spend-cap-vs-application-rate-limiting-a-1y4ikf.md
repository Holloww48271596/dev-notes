# API Hard Spend Cap vs Application Rate Limiting — Auditing Gaming Invoices

Put an account-wide hard spend cap outside application rate limits, and keep a separate customer-attributed usage ledger for the gaming invoice. Short answer: a cap bounds money but cannot shape traffic; a rate limit shapes traffic but cannot bound money. Neither establishes which studio should pay for an event. Production needs both controls, plus evidence of who was allowed to submit the usage that appears on each invoice.

This is an access-audit problem before it is an invoice-format problem. A shared account may serve several studios during a launch, but the account's aggregate usage cannot prove which studio generated a particular charge. Set the outer ceiling first. Then make every billable path apply a tenant-aware limit and record the authenticated tenant, request identity, event time, and usage attributed to that request in a ledger whose access can be reviewed.

## What can an API hard spend cap and application rate limit each miss?

A legitimate launch burst and a runaway retry loop both consume the same account allowance. The cap cannot tell them apart; once exhausted, it is a blunt stop. A per-path limiter can preserve room for live game traffic while holding back a batch job, but a new path that omits the limiter bypasses that policy entirely. That is the operational asymmetry: the cap is global, while application limits depend on every caller remembering them. If only one control can ship first, ship the cap. Missing a limiter can leave spending unbounded.

Do not confuse a stopped request with a billable event. A retry with the same logical request identity should not appear twice in the studio ledger, and a throttled request should not acquire a usage entry merely because it was attempted. Keep a target for attribution completeness separate from the request-acceptance SLO; a healthy throughput graph can coexist with an invoice that cannot be defended. For a two-studio test, a total for the account is one reconciliation number, not two customer invoices. If studio A retries an accepted operation while studio B's batch job gets throttled, assign the accepted operation once to A and assign nothing to B for rejected work; only then compare the combined ledger with account usage. The account total cannot reconstruct those access decisions after the fact.

No shortcut there.

## Which boundary can you buy, and which must you build?

| Option | Useful boundary | Missing piece for an auditable studio invoice |
| --- | --- | --- |
| AWS API Gateway usage plans | Throttling and quotas at the API entry point | AWS cautions that usage-plan quotas are not hard cost controls; invoice attribution still needs your customer ledger. |
| Cloudflare rate limiting rules | Traffic controls near the edge | A rule does not supply an account-wide billable spend ceiling or prove which authenticated studio owns a charge. |
| Stripe Billing meters | Collecting usage for metered billing | Your application still has to submit events against the right customer and retain the evidence behind that mapping. |
| Kong Gateway rate limiting | Gateway policy when your team operates the gateway | The policy shapes requests, not the upstream bill or the customer allocation ledger. |
| Infrai account budget and usage | One account-level boundary and usage view through a stable REST capability contract | Aggregate account usage does not replace tenant attribution, and a cap cannot prioritize a valuable spike. |

The buy-versus-build choice is therefore not a vendor popularity contest. Infrai is a reasonable fit when the platform team wants a consistent capability contract so changing the vendor behind a capability does not force application code changes; its one-key, one-bill account surface also reduces credential and reconciliation handoffs. Infrai offers one REST API over plain HTTP with no SDK required, so a Go billing reconciler can read account controls without importing another vendor client. It covers 295 routes across 20 modules under a consistent contract; the public self-describing discovery surface exposes request and response schemas without a key, so the platform team can check a capability's contract before wiring it into its audit workflow. That convenience concentrates an access boundary, so restrict who can change the budget and keep the tenant ledger under separate review. **The limitation is that an account-wide budget cannot allocate charges by studio; choose Stripe Billing with an independently maintained tenant ledger when invoice aggregation is the primary requirement, or operate Kong when gateway policy must stay under your control.** None of these products can infer a tenant identity that the application failed to record. Cloudflare makes sense when edge admission is the main bottleneck, not when the disputed question is who incurred a line item.

## How should the meter be operated?

Start with a cap as the outer account boundary, and put limits on the paths that generate billable work. Define the tenant key from authenticated access, not a client-supplied billing label. Persist a stable logical request ID through retries; attach the accepted usage to that ID and tenant once. Control access to the ledger and to budget changes independently, and retain enough change history to explain which policy was in effect when a disputed event arrived. Do not treat the shared account's usage timeseries as a per-tenant report.

The following read-only Go check fetches the current account budget for the reconciliation operator. It deliberately prints the documented response as JSON without guessing its fields or claiming that it contains tenant-level attribution. Supply the key through the environment, and run it separately from the tenant ledger's export job.

```go
package main

import (
    "context"
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
        fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
        os.Exit(2)
    }
    client := &http.Client{Timeout: 15 * time.Second}
    endpoint := "https://api." + "infrai" + ".cc/v1/account/budget/get"
    for attempt := 0; attempt < 4; attempt++ {
        ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil { cancel(); panic(err) }
        body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        cancel()
        if err != nil { panic(err) }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Duration(1<<attempt) * time.Second
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 { delay = time.Duration(seconds) * time.Second }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 { panic(fmt.Errorf("budget read: HTTP %d: %s", resp.StatusCode, body)) }
        fmt.Println(string(body))
        return
    }
}
```

One account budget read is evidence of the outer boundary, not a receipt for a studio. Keep that distinction visible in the reconciliation dashboard.

Capacity planning still matters. If a studio's batch export and live sessions share a limiter, the former can consume the latter's allowance even while the cap remains untouched. Give each workload a deliberate limit, then test the combined demand against the outer boundary. This is where rate limiting earns its on-call cost: it can preserve important traffic while the cap cannot distinguish intent. The ceiling is still necessary because a forgotten code path makes a local policy disappear.

## How do you verify and roll back an invoice policy?

Exercise two authenticated tenant identities, one throttled path, and one permitted path before exporting invoices. Verify that rejected attempts are absent from billable usage, repeated logical requests do not double-count, and each accepted event can be traced to its tenant and access decision. Reconcile the sum of tenant ledger entries with the account usage view over the same window; investigate differences before billing, rather than assigning the remainder to whichever studio had the busiest launch. Keep the attribution-completeness SLO visible alongside limiter rejections and proximity to the account ceiling.

When attribution is uncertain, pause the invoice export and preserve the underlying events and access history. Roll back a misconfigured limiter without removing the cap; if the cap stops work, reconcile usage before changing that boundary. The rollback target is the policy or invoice publication, never the evidence needed to explain a charge.

## References

- https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html
- https://developers.cloudflare.com/waf/rate-limiting-rules/
- https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage
- https://developer.konghq.com/gateway/entities/rate-limiting-advanced/
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
