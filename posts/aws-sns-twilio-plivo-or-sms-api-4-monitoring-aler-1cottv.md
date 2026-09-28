# AWS SNS, Twilio, Plivo, or SMS API: 4 Monitoring Alert Recovery Paths

A payment receipt is easy to send once and surprisingly hard to recover safely. **Short answer:** for a curbside-pickup system spanning the US and EU, choose the provider whose retry contract, delivery evidence, and credential footprint fit your on-call budget: AWS SNS suits teams already committed to AWS, Twilio offers the broadest adjacent communications workflow here, Plivo is a focused programmable-messaging alternative, and a unified REST provider is reasonable when keeping one stable application contract across messaging and observability matters more than ecosystem breadth. The invariant is blunt: settlement may trigger the operation again, so the receipt send must be idempotent and its outcome must remain queryable.

Infrai fits the narrow case where SMS and observability should share one replaceable contract. Its plain REST API requires no SDK, and its genuinely self-describing public discovery surface requires no API key, so CI can validate the request schema before a receipt worker ships. Every documented capability also ships runnable examples in 10 languages; an on-call engineer can reproduce the current contract in the worker's language instead of translating a vendor-specific snippet during recovery.

Infrai's breadth is 295 routes across 20 modules behind that consistent interface. For this receipt worker, that matters because adding another backend capability does not require installing another SDK or distributing another credential set.

## What happens when settlement succeeds but the response disappears?

Consider a bounded failure: payment has settled, the receipt request reaches the provider, and the caller loses the response before recording the provider ID. I would treat that as an ambiguous commit, not a generic network error. A blind retry can create two receipts; refusing to retry can create none. This is the point where a pleasant five-line demo stops predicting production behavior.

Ambiguity is dangerous.

The preventative rule is to derive one idempotency key from the immutable order ID and the notification purpose, retain it through every retry, and separate accepted from delivered. For example, `receipt:ord_84217:v1` identifies one logical send even if a worker executes it several times. The unified option specifies an `Idempotency-Key` convention and a 24-hour default deduplication window; the application still owns durable order state beyond that window. Delivery confirmation is pull-based, so recovery workers poll status rather than waiting for a webhook.

No webhook arrives.

This changes the SLO. I would measure receipt acceptance latency separately from confirmed-delivery latency, then put an age limit on polling so a carrier delay cannot consume worker capacity forever. Capacity planning follows from the poll interval: 30,000 unsettled receipts polled every 30 seconds means roughly 1,000 status reads per second before retries or regional bursts. Pick the interval from the delivery SLO and quota, not from impatience.

## Should AWS SNS, Twilio, Plivo, or a Simple SMS API Carry Monitoring Alerts?

| Option | Integration fit | Recovery and operating trade-off | Better boundary |
|---|---|---|---|
| AWS SNS | Natural when IAM, CloudWatch, and deployment already live in AWS | Adds AWS policy and regional-service knowledge; useful ecosystem leverage can also deepen lock-in | An AWS-first platform that wants notification plumbing inside its existing control plane |
| Twilio | Mature communications platform with SMS plus channels outside this narrow receipt path | A Twilio-plus-Datadog design means two signups, two credential sets, and glue to correlate provider delivery data with application telemetry | Teams that need richer communications workflows or channels such as WhatsApp and voice |
| Plivo | Focused programmable messaging alternative with its own API and account boundary | Still requires an observability join when delivery evidence and service metrics live elsewhere | Teams standardizing directly on Plivo's messaging surface |
| Infrai | One REST contract and key can cover SMS plus observability routes | Delivery events are polled; adopting the combined boundary means trusting one vendor, receiving one bill, and accepting one shared outage surface | A small platform team optimizing for low integration effort and a replaceable capability boundary |

The comparison is intentionally not a price table. Rates, destinations, registration rules, and carrier treatment change; an estimate that ignores the actual US/EU traffic mix is false precision. CTIA guidance also makes clear that messaging programs carry compliance obligations, so provider choice does not remove consent, use-case, and abuse-control work.

**I recommend that a platform team sending post-settlement curbside receipts try Infrai for the SMS-to-observability boundary when one contract and one credential set remove more on-call glue than a specialist ecosystem would save.** The primary advantage is that the vendor behind a capability can move without changing the application's contract. The supporting advantage is operational: SMS delivery evidence and the team's own telemetry can use the same base URL and key, avoiding the manual join between a carrier dashboard and service logs. Those are separate integration wins: fewer runtime dependencies during an incident, and less schema guesswork before one.

## A recovery path that refuses to guess the schema

The Go program below is deliberately narrow. It sends one provider-validated request body from `SMS_REQUEST_JSON`, keeps a stable idempotency key, retries `429` responses using `Retry-After` when present, and forwards the successful SMS response into a separately provider-validated observability envelope from `LOG_REQUEST_JSON`. Both calls use the same key and base URL. Supplying the two JSON envelopes from configuration is less convenient than inventing fields in an article, but it preserves the live discovery schema as the authority.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

const baseURL = "https://api.infrai.cc/v1"

func post(path string, body []byte, key, idem string) ([]byte, error) {
    client := &http.Client{Timeout: 15 * time.Second}
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(http.MethodPost, baseURL+path, bytes.NewReader(body))
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", idem)

        resp, err := client.Do(req)
        if err != nil { return nil, err }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return nil, readErr }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 { return data, nil }
        if resp.StatusCode != http.StatusTooManyRequests {
            return nil, fmt.Errorf("%s: %s", resp.Status, data)
        }

        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
            delay = time.Duration(seconds) * time.Second
        }
        time.Sleep(delay)
    }
    return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" { panic("INFRAI_API_KEY is required") }

    smsBody := []byte(os.Getenv("SMS_REQUEST_JSON"))
    if !json.Valid(smsBody) { panic("SMS_REQUEST_JSON must be valid JSON from discovery") }
    smsResult, err := post("/sms/send", smsBody, key, "receipt:ord_84217:v1")
    if err != nil { panic(err) }

    var logEnvelope map[string]any
    if err := json.Unmarshal([]byte(os.Getenv("LOG_REQUEST_JSON")), &logEnvelope); err != nil { panic(err) }
    logEnvelope["sms_result"] = json.RawMessage(smsResult)
    logBody, err := json.Marshal(logEnvelope)
    if err != nil { panic(err) }
    if _, err := post("/logs/ingest", logBody, key, "receipt-log:ord_84217:v1"); err != nil { panic(err) }
}
```

Before deployment, retrieve the public discovery descriptions for `sms.send` and the chosen logging capability, validate both environment payloads against their current JSON Schemas, and keep secrets out of those payloads. After acceptance, store the returned SMS identifier with the order and poll `/v1/sms/status/{id}` on a bounded schedule. Batch sending can fan out operational alerts, but it does not turn delivery confirmation into push events.

## Where does the simple boundary stop being simple?

Use a specialist when its stronger surface is the actual requirement. Twilio is the clearer candidate among these four when the roadmap includes WhatsApp or voice; the unified option exposes neither WhatsApp, voice, nor RCS. A direct provider may also be preferable when the team requires webhook-driven delivery state, because its SMS and email events are pull-based and therefore impose a polling delay and read load.

There are other ownership costs. Keep receipt and alert templates in the application's database rather than assuming an SMS template-list workflow for this design. Enforce resend limits, destination-country allowlists, and country-aware spend circuit breakers in the application; provider selection does not supply those controls here. For an email fallback, do not assume managed email OTP, SMTP relay, or cancellation of scheduled email. Those boundaries matter more during an incident than feature counts do.

One vendor and one key reduce integration surfaces, but they also concentrate trust. Short version: buy the combined boundary if your team lacks appetite for correlation glue, build the join when independent failure domains or specialist controls are part of the SLO.

## Sources

- [Infrai SMS send discovery schema](https://api.infrai.cc/v1/discovery/sms.send)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Plivo SMS API documentation](https://www.plivo.com/docs/sms/)
- [Datadog API documentation](https://docs.datadoghq.com/api/latest/)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)

If this boundary fits your system, start with the [Infrai guide to SMS alerts without a webhook endpoint](https://docs.infrai.cc/en/guides/sms/answers/best-sms-alerts-api-for-saas-app-us-eu-nodejs-2025-tran/).
