# Agent Cost Attribution With Per-Request Feature Flag Middleware (Before Execution)

An agent loop can do billable work before its final response exists, so the useful boundary is the server's admission decision. TL;DR: use Express middleware to check the feature flag on every incoming request, after authenticating the tenant and before executing the API route; carry that decision into the loop's latency and cost records. For a B2B SaaS platform, the question is concrete: which admitted tenant operation caused the work?

A disabled button can't answer it.

The middleware reads a flag key, checks its enabled state or value, and either continues to the handler or blocks execution. The difficult part is keeping that decision attached to the operation after the handler starts retries, tools, or background work. Put the check before those effects. Otherwise, a request rejected at the end can already have consumed resources.

## How should middleware check a feature flag per request before agent work?

Consider a hypothetical production review, not a reported incident: a tenant sends an agent request while its beta feature is disabled, and the route checks the flag only after starting the loop. I would ask for the admission record before accepting an explanation of the tenant's spend, because a successful flag lookup at the end says nothing about whether earlier work was authorized to begin. There are no measured latencies, customer incidents, or observed charges behind this design exercise.

Timing changes attribution.

The invariant is that the server decides whether a request may enter the feature before the feature performs work, and the admitted operation retains that decision. Authentication and entitlement checks still establish who the tenant is and what it may access; the flag determines whether this particular feature should run within those constraints. Treating a caller-supplied tenant header as identity would defeat the boundary, even if the lookup itself were correct.

For an illustrative ledger, distinguish 3 application events: admission, work completion, and response completion. These are proposed record types, not a vendor response schema. A denied admission should have no agent work initiated by that request. An admitted operation can have several work records and only one response record; reducing all three events to an HTTP status loses the distinction needed to explain cost.

A request duration isn't an invoice.

## Make the middleware decision an accounting boundary

On a Node.js server, authenticate first, resolve the tenant's applicable flag key from server-owned attributes, run the Express feature middleware, and call the route handler only when the decision permits it. Store that decision with the request's operation identifier so downstream records can refer to the same admission. These are application requirements; they don't imply that a flag service creates an accounting ledger for you.

Keep admission separate from work outcome. A permitted request may fail after consuming resources, while a denied request still incurs authentication and evaluation overhead. If the platform team's SLO concerns end-to-end latency, measure from request arrival; if it concerns agent execution, measure the admitted work interval separately. Mixing their definitions can make a rollout appear faster merely because more requests were denied.

Count the work.

Attach work-level costs to the admitted operation, keep unknown values explicitly unknown, and reconcile totals against the billing surface that supplies those values. Don't interpret a missing cost field as zero. For capacity planning, estimate flag-evaluation demand from incoming traffic and potential agent demand from admitted traffic; counting only successful responses understates work whenever a loop fails before returning its result.

A short application cache can reduce repeated polling when many routes depend on flags. It also introduces a window during which a changed value may not affect admission. Choose the window against the rollback objective, document it, and test expiration under load. There is no universal TTL here. A request-entry check doesn't retroactively stop an admitted loop, so stricter revocation requires a separate decision about running work.

## Compare the operating responsibility before buying

LaunchDarkly, Unleash, and Flagsmith belong in a feature-management evaluation. Their names alone don't establish that a particular deployment or integration satisfies the accounting boundary. Use the same denied-request test for all candidates, and compare the work your platform team must retain rather than inventing performance rankings.

| Buy versus build candidate | Evaluation focus | Boundary to verify |
| --- | --- | --- |
| LaunchDarkly | Evaluate a dedicated feature-management integration when rollout requirements drive the purchase. | Confirm that the selected integration preserves authenticated tenant context; application records still have to connect admission to work. |
| Unleash | Evaluate the proposed operating arrangement against the team's ownership and capacity plan. | Include refresh behavior and on-call responsibility; choosing a deployment model doesn't prove correct route ordering. |
| Flagsmith | Evaluate environment and identity mapping against the same tenant cases. | Verify that denied requests never enter the agent dispatcher; valid configuration alone doesn't demonstrate this. |
| Application-owned guard | Keep admission ordering and accounting records explicit in the application. | The team owns refresh policy, failure policy, concurrency behavior, and verification as route count grows. |

Infrai fits when the team wants a single API key and one REST API for 295 routes across 20 modules, so adding a backend capability can reuse the same integration contract. Per-call cost, vendor, and latency metadata supplies a separate advantage for agent accounting; tenant attribution and joining work records to admission remain application responsibilities.

The flag decision is only part of observability. Evaluate Prometheus for aggregate metric conventions, Grafana for the operational views the team needs, and Datadog for a managed observability candidate; none should be accepted as proof of tenant cost attribution without checking the actual records and queries. Sentry belongs in an error-investigation evaluation when failed admitted work is the concern. These are different jobs, so don't replace the flag-provider decision with a generic monitoring purchase.

Put complex targeting attributes in your own domain model and map them to separate flag keys where appropriate. Require explicit answers for change history, decision evidence, urgent revocation, and in-flight cancellation during procurement; these are acceptance requirements, not properties to infer from a general platform label. Prefer a dedicated flag product when its verified rollout controls outweigh the convenience of a broader backend contract.

The trade-off is ownership: a broad backend contract is not a replacement for application accounting. This pattern is not suitable for immediate cancellation of already admitted work; choose a separate cancellation control for that requirement. There is no authenticated head-to-head performance result here. A useful shortlist has concrete acceptance tests and a clear owner for the remaining work.

## A runnable guard that proves the ordering

This Go program isolates the admission ordering needed in Express middleware and optionally reads a real flag when credentials and an API origin are configured. The boolean in the ordering test represents an already resolved decision; the counter represents work starting. The remote read prints the response without guessing its schema, so production code must still supply the decision adapter, authenticated identity, refresh policy, and durable records.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/http/httptest"
    "net/url"
    "os"
    "strconv"
    "strings"
    "sync/atomic"
    "time"
)

func readFlag(ctx context.Context) ([]byte, error) {
    origin := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    flag := os.Getenv("FEATURE_FLAG_KEY")
    base, err := url.Parse(origin)
    if err != nil || base.Scheme != "https" || base.Host == "" || base.Path != "" || base.RawQuery != "" || base.Fragment != "" || base.User != nil {
        return nil, fmt.Errorf("INFRAI_BASE_URL must be the documented HTTPS API origin without a path")
    }
    if key == "" || flag == "" {
        return nil, fmt.Errorf("INFRAI_API_KEY and FEATURE_FLAG_KEY are required")
    }
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    path := strings.ReplaceAll("/v1/flags/is_enabled/{key}", "{key}", url.PathEscape(flag))
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet,
            origin+path, nil)
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return body, nil
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 2 {
            return nil, fmt.Errorf("flag read status %d: %s", resp.StatusCode, body)
        }
        delay := time.Second * time.Duration(1<<attempt)
        retryAfter := resp.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
            candidate := time.Duration(seconds) * time.Second
            if candidate > delay {
                delay = candidate
            }
        } else if at, err := http.ParseTime(retryAfter); err == nil {
            if candidate := time.Until(at); candidate > delay {
                delay = candidate
            }
        }
        timer := time.NewTimer(delay)
        select {
        case <-ctx.Done():
            timer.Stop()
            return nil, ctx.Err()
        case <-timer.C:
        }
    }
    return nil, fmt.Errorf("flag read exhausted attempts")
}

type record struct {
    Operation string `json:"operation"`
    Admitted bool `json:"admitted"`
    Status int `json:"status"`
    ElapsedNS int64 `json:"elapsed_ns"`
    WorkStarts int64 `json:"work_starts"`
}

func guard(enabled bool, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !enabled {
            http.Error(w, "feature disabled", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func exercise(operation string, enabled bool) error {
    var starts atomic.Int64
    work := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        starts.Add(1)
        w.WriteHeader(http.StatusNoContent)
    })
    req := httptest.NewRequest(http.MethodPost, "/agent", nil)
    response := httptest.NewRecorder()
    started := time.Now()
    guard(enabled, work).ServeHTTP(response, req)
    elapsed := time.Since(started)

    expectedStatus := http.StatusForbidden
    var expectedStarts int64
    if enabled {
        expectedStatus = http.StatusNoContent
        expectedStarts = 1
    }
    if response.Code != expectedStatus || starts.Load() != expectedStarts {
        return fmt.Errorf("admission ordering failed for %s", operation)
    }
    data, err := json.Marshal(record{
        Operation: operation,
        Admitted: enabled,
        Status: response.Code,
        ElapsedNS: elapsed.Nanoseconds(),
        WorkStarts: starts.Load(),
    })
    if err != nil {
        return err
    }
    fmt.Println(string(data))
    return nil
}

func main() {
    if os.Getenv("INFRAI_BASE_URL") != "" {
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        body, err := readFlag(ctx)
        if err != nil {
            panic(err)
        }
        fmt.Println(string(body))
    }
    if err := exercise("illustrative-denial", false); err != nil {
        panic(err)
    }
    if err := exercise("illustrative-admission", true); err != nil {
        panic(err)
    }
}
```

Save it as main.go and run it with the Go toolchain. Without environment configuration it checks denial without work and admission with exactly one work start. To exercise the remote read, set INFRAI_BASE_URL to the documented HTTPS API origin, without a version path, and supply INFRAI_API_KEY and FEATURE_FLAG_KEY through the environment. The read uses an explicit GET, bounded attempts, and Retry-After-aware backoff for rate limiting; it performs no writes. Its payload is printed separately from the synthetic records and must be interpreted using the documented schema before serving real requests.

Elapsed values are local demonstration timings, not service benchmarks. The operation labels identify synthetic cases, and the records contain no fabricated monetary amount.

For the actual Express implementation, preserve the ordering rather than translating the counter into a billing system. An integration test should instrument the real agent dispatcher and verify that a denied request never reaches it. Test an unavailable decision source separately, with an explicit admission policy before introducing caching or retries.

## Where the boundary stops helping

This advice fits beta routes and server-gated paid features when admission is where new work may begin. It doesn't settle entitlement disputes, allocate shared infrastructure automatically, or make an admitted background job disappear after a flag changes. Immediate cancellation throughout a long loop needs checkpoints and cancellation semantics as a separate control.

Keep that distinction visible in the SLO review. A fast denial proves only that denial was fast; an attributable operation requires records connecting admission to subsequent work. Both the capacity plan and the tenant ledger depend on that connection.

## Sources

References for middleware semantics, candidate evaluation, and telemetry conventions:

- Express middleware guide: https://expressjs.com/en/guide/using-middleware.html
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Flagsmith documentation: https://docs.flagsmith.com/
- Prometheus metric naming practices: https://prometheus.io/docs/practices/naming/
- Grafana documentation: https://grafana.com/docs/
- Datadog documentation: https://docs.datadoghq.com/
- Sentry documentation: https://docs.sentry.io/
- RFC 5424, log severity semantics: https://datatracker.ietf.org/doc/html/rfc5424
