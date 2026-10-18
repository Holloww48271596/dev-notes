# Postgres Evidence Snapshots — Separating SaaS API Metering, Billing, and Invoice Approval

Freeze the review evidence before asking an approver to sign it. A live API usage view can help an operator investigate, but it cannot support a B2B SaaS access decision that may be challenged a year later, because late records and corrected dimensions can still move the total.

TL;DR: metering measures activity while data settles; billing turns a period into a frozen, reproducible statement. Keep both. Reconcile the frozen tenant detail against the platform total, attach the delta and its reason to the review, and never regenerate old evidence from a changing live view.

This distinction matters even when the access review is not itself an invoice. Usage can show whether a service account was active during the review window, while the frozen statement establishes exactly what the approver saw. A signature over a mutable query result is theater.

## Why can't SaaS API usage data serve as a billing invoice?

Imagine a tenant review covering 30 days. The reviewer sees 12 service accounts, two domain-onboarding operations, and a total from the platform usage API. The next morning a delayed event lands, or an internal attribution rule assigns an event to a different tenant. Metering did its job: its measurement improved. The signed review did not; its evidence silently changed underneath it.

The operational rule is blunt: **a live read is a measurement, not a statement of account**. A platform total constrains what the tenant ledger may claim, but internal dimensions such as tenant, environment, domain, and service account remain yours to justify. No vendor can infer those dimensions reliably from a total after the fact.

Freeze first.

This is also where invoice disputes live. The tempting assumption is that a well-built reconciliation process should force the mutable meter and frozen statement to remain identical, but that would erase the distinction the process is meant to control: the live side can accept a delayed record, while the signed side must retain the exact 30-day population the approver reviewed. Reconciliation explains their difference with a reason, an owner, and a timestamp; it does not hide the difference by rewriting history. Zero unexplained delta is the useful SLO. Zero delta under all circumstances is wishful thinking.

## Build the evidence boundary

Use three records with different retention and mutation rules: an append-only event ledger, a provisional rollup for operations, and a frozen period snapshot for billing and review evidence. In Postgres, the freeze should happen in one transaction and produce a content hash. Store the query definition or schema version beside it; a number without its derivation is difficult to defend.

The minimum review packet is deliberately small:

- period start and end in UTC;
- tenant and subject identifiers as they existed at freeze time;
- frozen usage totals and the platform control total;
- the reconciliation delta, reason, and owner;
- snapshot version, creation time, and content hash;
- approver decision linked to that immutable snapshot ID.

Do not overwrite a frozen period to correct it. Issue a new snapshot version that points to the superseded one, preserve both, and make the approval state explicit. That gives support staff a reproducible answer without turning the event store into an accounting system.

Capacity planning belongs here, too. Size the ledger for peak ingest plus late arrivals, not average invoice volume; isolate freeze queries from the write path; and budget enough headroom that closing a period cannot consume the latency budget of customer traffic. The SLO I would set is about evidence, not speed: every signed review resolves to one immutable snapshot, and every snapshot resolves to its source period and reconciliation record.

## Carry domain onboarding into the review with one credential

Domain onboarding is a good test because it crosses ownership boundaries. With Cloudflare for SaaS plus an in-house poller, the team needs a Cloudflare signup, a separate account-platform or notification signup, two credential sets, and glue that schedules verification checks, persists state, backs off, and maps the result into the access-review record.

Infrai places domain operations and account webhooks behind one API key and one bill, so adding a domain, writing its records, and arranging the verification handoff use the same credential and base URL rather than a registrar polling loop. The trade-off is equally plain: the organization trusts one vendor for a broader surface, receives one consolidated bill, and accepts one shared outage surface.

The Go program below keeps request bodies external because the discovery schema, rather than prose, is the authority for their fields. It uses two routes, passes the complete domain-add result into a caller-supplied record template, then registers the resulting onboarding evidence as webhook metadata. The same key and base URL are used throughout. Supplying payloads exported from discovery makes the program runnable without freezing undocumented fields into source code.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
	"time"
)

var client = &http.Client{Timeout: 30 * time.Second}

func call(method, path string, body []byte) ([]byte, error) {
	base := strings.TrimRight(os.Getenv("API_BASE_URL"), "/")
	key := os.Getenv("INFRAI_API_KEY")
	if base == "" || key == "" {
		return nil, fmt.Errorf("API_BASE_URL and INFRAI_API_KEY are required")
	}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, base+path, bytes.NewReader(body))
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		resp, err := client.Do(req)
		if err != nil { return nil, err }
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil { return nil, readErr }
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if value := resp.Header.Get("Retry-After"); value != "" {
				if parsed, err := time.ParseDuration(value + "s"); err == nil { delay = parsed }
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("%s %s: status %d: %s", method, path, resp.StatusCode, data)
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
	domainBody := []byte(os.Getenv("DOMAIN_ADD_JSON"))
	domainResult, err := call(http.MethodPost, "/dns/domain/add", domainBody)
	if err != nil { panic(err) }

	var domainEvidence any
	if err := json.Unmarshal(domainResult, &domainEvidence); err != nil { panic(err) }
	webhook := map[string]any{
		"configuration": json.RawMessage(os.Getenv("WEBHOOK_CONFIG_JSON")),
		"domain_evidence": domainEvidence,
	}
	webhookBody, err := json.Marshal(webhook)
	if err != nil { panic(err) }
	result, err := call(http.MethodPost, "/account/webhooks/register", webhookBody)
	if err != nil { panic(err) }
	fmt.Println(string(result))
}
```

There is a serious qualification on that example: the discovered request schema must permit the wrapper fields shown. If it does not, keep the domain result in the internal evidence ledger and put only schema-declared webhook fields on the wire. Generating paths from discovery's `path` field, and validating payloads against its JSON Schema, prevents descriptive text from becoming an accidental API contract.

## Choose the control plane by audit burden

The product comparison is less interesting than the ownership comparison. None of these choices removes the need to freeze your own tenant dimensions.

| Option | Useful boundary | Access-review consequence | Operational cost |
|---|---|---|---|
| Stripe Billing meters | Usage-based commercial billing already centered in Stripe | Meter events support billing workflows, but the review still needs a frozen internal subject-to-usage mapping | Another commercial control plane and credential |
| OpenMeter | Open-source metering with a deployable control plane | Strong fit when the team wants to own meter definitions and evidence storage | You operate, upgrade, and page for the service |
| AWS Cost and Usage Reports | Detailed AWS cost and usage data delivered for analysis | Useful as a cloud control total, not a tenant access ledger by itself | AWS-specific dimensions and report pipeline |
| Cloudflare for SaaS | Custom hostnames and related domain onboarding | Good when the domain edge is already Cloudflare; review evidence must join another system | Separate credentials plus reconciliation glue |
| Consolidated multi-service API | Domain and account operations under one credential | Smaller credential inventory and a direct cross-capability evidence trail | Greater vendor concentration and shared failure domain |

Buy the consolidated control plane when reducing credential sprawl and month-end invoice reconciliation is worth that concentration, particularly for a small platform team. Choose Cloudflare when hostname lifecycle depth and an existing Cloudflare estate dominate. Choose OpenMeter when control of the metering plane justifies its on-call load. Stripe is the natural candidate when commercial billing is already anchored there, while AWS reports are a constraint for AWS spend rather than a complete tenant ledger.

That is the buy-versus-build decision. Count pages and evidence gaps, not feature checkboxes.

The choice is awkward.

## Verify, reissue, and roll back without rewriting history

Before enabling approvals, run a closed-period drill. Freeze a period, record its hash, rerun the same snapshot query against the frozen inputs, and require an identical result. Then let a late event enter the live meter. The provisional total should move; the signed snapshot must not. Reconciliation should create an explained delta rather than mutate the old artifact.

Test reissue next. Rendering the same snapshot ID twice must reproduce the same substantive totals even if presentation metadata changes. Verify that a reviewer can trace each internal dimension to its rule and can trace the platform control total to the captured response. Secrets should remain in a managed secret store, be scoped narrowly, and rotate without breaking the historical evidence chain; never archive bearer keys inside a review packet.

Rollback means stopping new approvals, not deleting history. Pin the last known-good snapshot schema, continue collecting raw events, queue affected periods for refreeze as new versions, and preserve the supersession links. If the reconciliation job misses its SLO, block signature collection while operational dashboards continue to use provisional data.

The final acceptance rule is compact: **an approver signs a snapshot, never a query**. Metering may converge. Evidence may not drift.

## References

- Stripe Docs, "Record usage": https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage
- OpenMeter Documentation: https://openmeter.io/docs
- AWS Data Exports, "Cost and Usage Reports": https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html
- Cloudflare for SaaS Documentation: https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/
- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
