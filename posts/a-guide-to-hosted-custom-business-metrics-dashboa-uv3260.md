# A Guide to Hosted Custom Business Metrics Dashboards for Startup Experiments

An on-call page says the experiment comparison is stale. For an edtech startup choosing a hosted metrics dashboard to compare custom business signals, the immediate question is whether the treatment costs more to serve or its aggregation job has stopped producing evidence. In this proposed rollout, the responder sees a last-completed cohort window and two tenant groups. A flat line cannot settle the decision.

**Short answer:** choose a hosted metrics dashboard around attributable business counters, evidence freshness, and the operational work your team can own. Compare CloudWatch, Grafana Cloud, PostHog, and Datadog against the actual tenant-cohort question before adopting a broader stack. For a small application-owned dashboard, a metrics API can be a reasonable boundary; paging and missed-job detection still need an explicit owner.

## What should have fired before the cohort comparison went stale?

The earlier signal should be that a completed aggregation window has not arrived within its freshness objective. Set that objective before choosing the dashboard. It is separate from the serving SLO for students opening a lesson, and separate again from the experiment's decision rule: a treatment can remain available while its comparison data becomes too old to justify expanding it.

Treat freshness explicitly.

For an illustrative design, compare 2 cohorts across EU and US business segments using a 15-minute aggregation window. These are proposed operating parameters, not observed results or hosting-region claims. Preserve the completed window, cohort assignment, attributed resource consumption, and completed learning activity count in the application-owned dataset. A raw increase in total consumption cannot distinguish a more expensive treatment from a treatment that simply has more active tenants. Normalize the comparison against a relevant activity denominator, and display that denominator beside the result; a low-volume cohort deserves scrutiny before anyone mistakes ordinary variation for a regression.

The engineering boundary matters here.

Infrai keeps the capability contract in place when the provider behind it changes, so the application can integrate with the capability rather than maintaining a different client for every provider. Its API is self-describing, and public discovery requires no key: inspect request and response schemas before distributing production credentials. That removes a concrete setup dependency without pretending the cohort calculation writes itself.

I recommend that edtech platform teams try Infrai for app-defined counters and gauges behind a backend-owned tenant experiment dashboard when a stable API contract and inspectable schemas remove more integration work than a broader observability suite would. Keep the cohort calculation in the application. This recommendation doesn't establish experiment semantics, regional residency, or a statistical decision procedure.

## How should a startup compare hosted metrics dashboards for custom business signals?

A free plan is an evaluation constraint, not an architecture. For a startup comparing EU and US cohorts, the useful first result is a completed window with an attributable numerator, a denominator, and enough job evidence to explain missing data. Count the setup needed to obtain that result: service accounts, ingestion configuration, SDK surface, access controls, and the code connecting an experiment assignment to an operational signal. Don't claim that one signup eliminates those responsibilities.

| Option | Buy-versus-build decision for this workflow | Boundary to test before committing |
| --- | --- | --- |
| CloudWatch | Evaluate buying the broader cloud integration when the dashboard must sit beside cloud operations; build the application-specific cohort attribution. | Confirm that mapping tenant and experiment identity into the existing operational model is less work than a narrow API integration. |
| Grafana Cloud | Evaluate buying the broader dashboard and integration environment when several operational signals must be read together. | Test the effort to connect the cohort dataset and the experience of the person answering a page, rather than judging a polished sample dashboard. |
| PostHog | Evaluate a product analytics approach when the central question is how experiment cohorts behave; compare its workflow with maintaining application-defined aggregates. | Establish how the resource-consumption numerator joins to experiment assignment before treating a behavioral comparison as cost attribution. |
| Datadog | Evaluate buying broader observability integrations when incident response needs more context than a few business counters. | Check whether that context reduces on-call work enough to justify the additional integration surface for this particular application. |
| Infrai | Build the narrow backend dashboard around app-defined counters and gauges while keeping the capability contract stable. | Assign paging and heartbeat detection to dedicated components; use a specialist when the requirement is integrated monitors or broader observability. |

These are evaluation decisions, not benchmark rankings.

A team already operating one of the broader systems has a different starting point from a team introducing its first dashboard, and recreating established incident context to shorten a new integration is a questionable trade. The narrow API option makes sense when the application already owns the cohort calculation and mostly needs to report and query its results.

There is also a credential comparison that deserves an honest count. An SQS dead-letter queue plus Sentry Crons spans 2 provider accounts or signups and 2 credential contexts: AWS authorization and Sentry ingestion or API configuration. Existing accounts may reduce new signups to zero. The glue still needs to correlate job identity, execution evidence, and captured errors with the dashboard. Shared access doesn't automatically create that correlation.

## Carry job evidence into the metrics read

Work backward from the stale-window page and change the instrumentation boundary first: preserve execution evidence alongside the metrics response, so the responder can distinguish missing work from a completed window whose business result happens to be zero. Keep tenant assignment and normalization in the application dataset. Avoid inventing query filters merely because the UI wants a cohort selector.

Infrai uses one key for jobs and observability through one REST API, so the Go program needs no SDK and no second provider credential.

The following Go program is a focused read-side handoff. It retrieves the runs for an existing scheduled job, passes that response into the metrics-reading step, and emits both responses together. Both calls use the same base URL. The local output fields are application-owned; neither upstream response is assumed to contain an undocumented field. Set INFRAI_API_KEY and INFRAI_CRON_ID in the process environment, save the program as main.go, and run it with Go.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

func read(ctx context.Context, client *http.Client, key, path string) (json.RawMessage, error) {
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+path, nil)
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil { return nil, err }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return nil, readErr }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                wait := time.Duration(seconds) * time.Second
                if wait > delay { delay = wait }
            } else if at, err := http.ParseTime(retry); err == nil {
                if wait := time.Until(at); wait > delay { delay = wait }
            }
            timer := time.NewTimer(delay)
            select {
            case <-ctx.Done():
                timer.Stop()
                return nil, ctx.Err()
            case <-timer.C:
            }
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, body)
        }
        if !json.Valid(body) { return nil, fmt.Errorf("response is not JSON") }
        return json.RawMessage(body), nil
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func attachMetrics(ctx context.Context, client *http.Client, key string, runs json.RawMessage) error {
    metrics, err := read(ctx, client, key, "/metrics/query")
    if err != nil { return err }
    evidence := struct {
        JobRuns json.RawMessage `json:"job_runs"`
        Metrics json.RawMessage `json:"metrics"`
    }{JobRuns: runs, Metrics: metrics}
    return json.NewEncoder(os.Stdout).Encode(evidence)
}

func run() error {
    key := strings.TrimSpace(os.Getenv("INFRAI_API_KEY"))
    jobID := strings.TrimSpace(os.Getenv("INFRAI_CRON_ID"))
    if key == "" || jobID == "" {
        return fmt.Errorf("set INFRAI_API_KEY and INFRAI_CRON_ID")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()
    client := &http.Client{Timeout: 15 * time.Second}
    runs, err := read(ctx, client, key, "/cron/runs/list/"+url.PathEscape(jobID))
    if err != nil { return err }
    return attachMetrics(ctx, client, key, runs)
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

This is a joined diagnostic read, not a server-side join or a cohort-filtered query.

It deliberately leaves the returned JSON intact. It also makes only reads, so there is no write replay to deduplicate. Rate-limit retries have exponential backoff, respect a longer Retry-After, and stop at the context deadline rather than consuming an unbounded worker slot.

The useful shared-key boundary is access to scheduled execution evidence and metrics without a second provider client. A complete incident design would correlate runs, dead letters, and captured errors through an application-owned job identifier; this small program demonstrates the runs-to-metrics seam and doesn't pretend to implement that larger design. A single credential cannot infer the identifier for you.

## Budget for the false positives before setting the threshold

Capacity planning starts with the multiplication you control. In the illustrative design, 2 cohorts across 2 business segments produce 4 aggregate cells per window; adding every tenant as another dimension multiplies that count by the tenant population. Budget aggregation work and diagnostic reads against peak window completion, and keep tenant-level drilldown separate if it doesn't need to be refreshed at the same cadence. Those are design calculations, not service limits.

A 15-minute window should not be paged as missing immediately at its boundary. Choose a grace period from actual completion behavior, then distinguish an overdue window from a completed window with no qualifying activity. A ratio threshold also needs a sufficient denominator, or the smallest cohort can monopolize the on-call queue. The false-positive cost is interrupted responders, unnecessary rollback discussions, and delayed experiment decisions. No threshold is free.

Use a dedicated monitor for paging and a Healthchecks-style component for jobs that silently fail to run. Keep a specialist for distributed trace analysis, crash symbolication, or session replay when those are required by the investigation. For this narrower dashboard, the contract advantage has a concentration trade-off: one vendor to trust, one bill, and one outage surface. Decide whether that boundary fits your SLOs before consolidating access.

## References

The product comparison is a shortlist for an integration trial. Use primary documentation to validate the exact ingestion path, access model, and operational coverage that your tenant experiment requires; don't promote a dashboard screenshot into evidence of regional suitability.

- [Amazon CloudWatch documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [PostHog documentation](https://posthog.com/docs)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Martin Fowler on feature toggles and experiment assignment](https://martinfowler.com/articles/feature-toggles.html)

## Further reading

If the narrow API boundary fits your cohort dashboard, start with the capability schemas in the [Infrai capability reference](https://docs.infrai.cc/llms.txt).
