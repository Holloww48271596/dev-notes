# Custom Failure Metrics Dashboard: 3 Cohort Signals for Email Alerts

The useful page says which experiment cohort is failing, how much completed work is affected, and who owns the response. For an edtech service comparing tenant cohorts, the least complex starting point is a custom failure counter, a dashboard showing the matching attempt count, and a small poller that evaluates the same window before sending an email. A page that says only “imports failed” leaves the on-call engineer doing the attribution work while the experiment continues.

TL;DR: Report failures and attempts together, query a short window, and evaluate the failure ratio separately for each bounded cohort. Add attributed operating cost as the third signal for the experiment review. Use the dashboard for inspection and a separate notification component for email delivery; page on an SLO threat, and route cost differences to the experiment owner.

The incident described here is a design exercise, not a production report. All cohort counts and thresholds below are illustrative inputs. They are useful for checking arithmetic and routing behavior; they establish no vendor performance, uptime, or savings claim.

## How should a custom metrics dashboard poll failure counts and send email alerts?

Start at the alert. An import failure email should identify the cohort, the evaluated window, failed and attempted operations, the threshold crossed, and the responsible team. The dashboard should let the recipient inspect those same numbers. Otherwise an alert and its investigation can disagree merely because one uses a different interval or denominator, and the next person on call inherits an argument about queries instead of a decision about learner impact.

Count completed attempts.

Consider two deliberately small example rows: a control cohort has 100 attempts and 2 failures, while a treatment cohort has 200 attempts and 12 failures. The corresponding failure ratios are 2% and 6%. Comparing raw failures would exaggerate the difference because the treatment cohort processed twice as much work. Even the ratio needs context: differences in tenant workload can make a cohort comparison misleading without proving that the experiment caused the failures.

The earlier signal is the cohort ratio rising against its own service objective while enough completed attempts exist to make the result actionable. A rising failure count alone cannot distinguish deterioration from traffic growth. A missing completion is different again: a job that never runs contributes no failure event, so use a heartbeat monitor such as Healthchecks for that detection path rather than interpreting an empty failure window as success.

For the experiment review, put attributed operating cost beside attempts and failures. Distinguish cost per attempt from cost per successful completion. Retries can increase work even when the final outcome succeeds, so decide whether the experiment owns retry work before anyone reads the comparison. An allocation rule is part of the experiment contract, not a rounding choice made after results arrive.

## Make the instrumentation carry the attribution

Record a custom counter such as `import_failed` at the terminal failure boundary, and record the corresponding completed attempt exactly once at that same boundary. A retry is additional work, but it must not silently become another terminal outcome for the same logical import. Keep retry work in a separate accounting dimension or record so the operational success ratio and the cost analysis answer their respective questions.

This boundary matters more than the chart styling. If one code path records failures before retry exhaustion while another records attempts after completion, the numerator and denominator describe different populations. Parsing mixed application logs for each alert makes that contract harder to inspect; explicit counters provide a shared operational definition for the dashboard and polling logic.

Use bounded cohort values, such as control and treatment, and keep tenant-level billing records outside the metric label set. A capacity review should count the combinations of metric names, cohort values, operations, and outcomes before approving another dimension. Prometheus's instrumentation guidance is a useful reference for the cardinality consequences. A tenant identifier that seems harmless during an experiment can multiply series as adoption grows, precisely when the team has less appetite for telemetry work.

Attribution needs evidence.

Infrai offers 295 routes across 20 modules under one key, so a team that needs metrics and adjacent backend capabilities can add another capability through a consistent API contract; its per-call cost, vendor, and latency metadata also provides inputs for an operating-cost ledger, although that metadata is not itself a tenant-cohort allocation policy. Treat metric reporting and querying, notification delivery, and heartbeat monitoring as separate responsibilities in the design. The poller owns threshold evaluation and hands a notification to the chosen delivery service.

Keep query construction behind a small adapter. Validate the returned window and cohort coverage against known inputs before enabling pages, and keep the dashboard and poller on the same interpretation of those results. An alert should never depend on an assumed filter parameter or an inferred response field.

## A small evaluator makes the email decision reviewable

The application can emit counters from Node.js, but the focused example below uses Go to make the evaluation and email boundary independently runnable. It operates on the illustrative cohort window above; it does not pretend to be a vendor query client. Replace the fixture with an adapter to a verified query response when integrating it. The deliberate separation keeps unverified API schemas out of a copyable example.

The example uses a 5% window threshold and a minimum of 100 completed attempts. Neither number is a recommended production SLO. It sends one email for the treatment row when an internal SMTP relay is configured through `SMTP_ADDR`, `ALERT_FROM`, and `ALERT_TO`; otherwise it prints the email for inspection. The relay must permit delivery from this process. This example supplies no relay credentials and isn't a recurring polling daemon.

There are two explicit execution modes. With `METRICS_ORIGIN` set to the provider's HTTPS API origin and `INFRAI_API_KEY` supplied through the environment, it queries metrics with no assumed filters, prints the response, and exits. Without that origin it evaluates the local fixture. The live response never masquerades as the illustrative cohort dataset: map a verified response into completed windows before joining the two paths.

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net"
    "net/http"
    "net/smtp"
    "os"
    "strconv"
    "strings"
    "time"
)

type Window struct {
    Cohort   string
    Attempts int
    Failures int
}

func main() {
    if origin := os.Getenv("METRICS_ORIGIN"); origin != "" {
        if err := queryMetrics(origin); err != nil {
            log.Fatal(err)
        }
        return
    }
    windows := []Window{
        {Cohort: "control", Attempts: 100, Failures: 2},
        {Cohort: "treatment", Attempts: 200, Failures: 12},
    }
    const minimumAttempts = 100
    const threshold = 0.05
    var breaches []string
    for _, w := range windows {
        if w.Attempts < 0 || w.Failures < 0 || w.Failures > w.Attempts {
            log.Fatal("invalid completed-attempt window")
        }
        if w.Attempts < minimumAttempts {
            continue
        }
        ratio := float64(w.Failures) / float64(w.Attempts)
        if ratio >= threshold {
            breaches = append(breaches, fmt.Sprintf(
                "%s: %d/%d failed (%.1f%%), threshold %.1f%%",
                w.Cohort, w.Failures, w.Attempts, ratio*100, threshold*100,
            ))
        }
    }
    if len(breaches) == 0 {
        fmt.Println("No threshold breach in the example window.")
        return
    }
    body := "Illustrative cohort window; owner: platform team\r\n" +
        strings.Join(breaches, "\r\n")
    relay := os.Getenv("SMTP_ADDR")
    if relay == "" {
        fmt.Println(body)
        return
    }
    if _, _, err := net.SplitHostPort(relay); err != nil {
        log.Fatal(err)
    }
    from, to := os.Getenv("ALERT_FROM"), os.Getenv("ALERT_TO")
    for _, address := range []string{from, to} {
        if address == "" || strings.ContainsAny(address, "\r\n") {
            log.Fatal("set valid ALERT_FROM and ALERT_TO mailbox addresses")
        }
    }
    message := "From: " + from + "\r\nTo: " + to +
        "\r\nSubject: Cohort import failure threshold crossed\r\n\r\n" + body
    if err := smtp.SendMail(relay, nil, from, []string{to}, []byte(message)); err != nil {
        log.Fatal(err)
    }
}

func queryMetrics(origin string) error {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" || !strings.HasPrefix(origin, "https://") {
        return fmt.Errorf("set an HTTPS METRICS_ORIGIN and INFRAI_API_KEY")
    }
    client := &http.Client{
        Timeout: 15 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet,
            strings.TrimRight(origin, "/")+"/v1/metrics/query", nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        data, readErr := io.ReadAll(io.LimitReader(resp.Body, 4<<20))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                if specified := time.Duration(seconds) * time.Second; specified > delay {
                    delay = specified
                }
            } else if until, err := http.ParseTime(retry); err == nil {
                if specified := time.Until(until); specified > delay {
                    delay = specified
                }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("metrics query status %d: %s", resp.StatusCode, data)
        }
        fmt.Println(string(data))
        return nil
    }
    return fmt.Errorf("metrics query retry budget exhausted")
}
```

Save this as `main.go` and run it with `go run main.go`. With no relay configured, the output exposes the treatment cohort's 12 failures out of 200 attempts. That is the decision the dashboard should explain.

Production polling needs persistent notification state. Identify an evaluated window and cohort, record the alert transition, and suppress repeated sends while the same condition remains active. Preserve a recovery transition too. If delivery has an uncertain outcome, reconcile it using the notification service's delivery semantics before retrying; blindly rerunning a poll can turn one sustained breach into a stream of duplicate email. Missing query results should produce an explicit data-quality state rather than a zero failure count.

## Buy versus build around the operating obligation

The question is which team will own collection, window evaluation, delivery state, and capacity growth after the experiment becomes permanent. Compare actual obligations before comparing a dashboard screenshot. The following is a selection framework, not a benchmark or a claim that these products are interchangeable.

| Option | Reason to evaluate it | Boundary to settle before adoption |
| --- | --- | --- |
| Prometheus | A metrics-centered implementation where the team wants direct control of instrumentation and alert rules | Budget ownership for collection, retention, cardinality, and the alert-delivery configuration |
| Grafana Cloud | A managed observability option when operating the metrics backend competes with the platform roadmap | Verify cohort queries, notification routing, retention needs, and usage growth against the proposed instrumentation |
| Datadog | A managed monitoring option to compare when the organization wants monitoring and alert investigation in an established service | Test the same cohort dataset and assess configuration portability and operating commitments |
| Custom dashboard and polling code | A narrow experiment with explicit counters and a small notification state machine | Assign ongoing ownership of polling, deduplication, data gaps, and delivery verification |

Prometheus, Grafana Cloud, and Datadog deserve the same acceptance test: can the proposed setup explain why the treatment row alerts and the control row does not, while preserving the attribution boundary? Do not infer a winner from the existence of an alerting feature. A managed service can remove backend administration while leaving metric design and SLO ownership with the platform team; building the evaluator keeps the decision logic visible but adds code that somebody must operate.

Keep an exit path. Store the cohort definition, counter semantics, and cost-allocation rule in the application contract so a backend change does not redefine the experiment.

## Charge false positives to the experiment too

A threshold that pages on every small fluctuation consumes on-call capacity and can encourage an unnecessary rollback; a minimum-volume gate reduces that noise but delays detection for smaller cohorts. Choose the trade-off against the learner-facing SLO and test low-volume windows explicitly. The illustrative 100-attempt gate would leave a cohort with 99 completed attempts unevaluated even if its failures deserved investigation. That should be a visible state on the dashboard, not a green check.

For this edtech experiment, the actionable decision is to evaluate failure ratios by bounded cohort, retain a separate ledger for attributable work, and route the service-impact alert to the responder while the experiment owner reviews cost differences. Review false pages alongside useful pages. If the threshold interrupts people without changing a decision, its operating cost belongs in the comparison as surely as another retry does.

## Further reading

References for instrumentation and evaluating the operating alternatives:

- Prometheus instrumentation practices: https://prometheus.io/docs/practices/instrumentation/
- Prometheus alerting overview: https://prometheus.io/docs/alerting/latest/overview/
- Grafana Cloud documentation: https://grafana.com/docs/grafana-cloud/
- Datadog monitor documentation: https://docs.datadoghq.com/monitors/
- Healthchecks documentation: https://healthchecks.io/docs/
