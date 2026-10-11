# Node.js Password Reset: Own Email API Templates, Reserve SMS OTP

Short answer: for a B2B SaaS password-reset flow, keep the email template in your application repository and make an emailed, short-lived link the default; reserve SMS OTP for an explicit escalation path. This is the simpler operational choice when users already have a verified email address and the team can own link issuance, template review, and delivery telemetry. SMS is the better primary path only when a verified phone number is the durable account identifier or access to email is predictably unavailable.

The page fires at 02:13: `recovery_completion_ratio` has fallen below its objective. The on-call sees enough successful API submissions to make the provider look healthy, yet users are opening support tickets because no reset arrives. That gap is the decision in miniature: an accepted request is not a recovered account, and channel simplicity must be measured at the user outcome rather than at the send call.

## How can password reset email API and SMS OTP pages detect user harm?

A send-success alert watches the easiest boundary and misses the longer transaction. For email, the path includes token creation, queueing, template rendering, message acceptance, mailbox delivery, link opening, token validation, and password replacement. SMS swaps mailbox delivery and link opening for carrier delivery and code entry, but it also adds phone-number quality, message segmentation, and a second account identifier to govern. Neither channel deserves an SLO merely because its API returned success.

Work backward from the recovery completion event. The earlier signal should have been a widening gap between `recovery_requested` and `recovery_completed`, sliced by channel and region, with enough delay to respect the short expiry window. A useful page is tied to customer harm: a sustained burn of the completion SLO, not one transient queue spike. Queue age, render failures, delivery outcomes, token-expired outcomes, and completion latency remain diagnostics. They can open a ticket before they justify waking someone.

This distinction matters because the first instinct is often to add SMS as a universal fallback. That creates two delivery systems while leaving the missing end-to-end signal untouched. More paths can hide the symptom and enlarge the on-call surface at the same time.

Stop there.

## The 02:13 trace points to template ownership

A password-reset message is security-sensitive application behavior, not marketing content. Keeping the email template beside the code gives the team reviewable changes, deterministic variables, localization tests, and one deployment trail for the link contract. The delivery service transports the rendered message; it does not become the source of truth for wording, expiry disclosure, or the destination URL.

The trade-off is real, and the recommended path has a clear limitation. Application ownership means the platform team must maintain rendering, escaping, plain-text and HTML variants, localization coverage, and compatibility tests; it must also keep token policy synchronized with what the message says, prevent a stale deployment from emitting the wrong expiry text, and make template revisions visible in telemetry without logging message content. Handing templates to a messaging service can reduce application code, but now a console edit can drift from the release that generates its variables, so the apparent convenience moves change control outside the normal review trail. Repository-owned email is not suitable when users cannot reliably reach their registered inbox, when phone possession is the established account proof, or when the team cannot operate domain authentication and delivery diagnostics. In those cases, choose SMS OTP instead, with verified phone-number lifecycle controls and explicit code-attempt policy. For a small team, fewer lines of code are not automatically fewer operational boundaries; the capacity plan has to count each queue, callback stream, dashboard, policy surface, and responder skill, because all of them consume attention during an incident even when message volume is low.

| Decision | Repository-owned email link | Service-owned SMS OTP |
|---|---|---|
| Primary identifier | Existing verified email | Verified phone number |
| Template change control | Code review and release history | External configuration workflow |
| Payload pressure | Room for context and a clear expiry notice | Character encoding and segment count affect transport |
| Recovery proof | Single-use link token | Short code plus attempt controls |
| On-call surface | Mail delivery, link, and token stages | Carrier delivery, code entry, and phone-number lifecycle |
| Best fit | Email is already the account identity | Phone is the durable identity or email access is unavailable |

The table points one way for the stated system: own the email template and keep SMS exceptional. It does not claim email delivery is easy. DKIM exists because a receiving system needs a domain-level mechanism to validate that a message was authorized and was not altered in transit; signing and authentication belong in the production readiness checklist, not in a post-launch deliverability scramble.

Email has a boundary.

## Measure the recovery cohort before transport health

Emit stable, low-cardinality events at state transitions. Do not put an email address, phone number, reset token, or template body in labels or logs. A random flow identifier can connect events inside the retention window, while the security audit record should capture policy-relevant actions separately from operational metrics.

The following Go sketch shows the boundary even if the surrounding web service is Node.js. The point is the event contract: channel, stage, result, region, template revision, and elapsed time are bounded dimensions; user secrets are absent.

```go
package recovery

import (
	"context"
	"time"
)

type Event struct {
	FlowID          string
	Channel         string
	Stage           string
	Result          string
	Region          string
	TemplateVersion string
	Elapsed         time.Duration
}

type Recorder interface {
	Record(context.Context, Event) error
}

func RecordCompletion(ctx context.Context, r Recorder, flowID, channel, region, templateVersion string, started time.Time) error {
	return r.Record(ctx, Event{
		FlowID:          flowID,
		Channel:         channel,
		Stage:           "recovery_completed",
		Result:          "success",
		Region:          region,
		TemplateVersion: templateVersion,
		Elapsed:         time.Since(started),
	})
}
```

A counter alone is insufficient. Track a cohort from request to completion so delayed successes land against the correct requests, then monitor completion ratio and latency distributions by channel and broad region. Keep template version as a bounded release identifier; a free-form template name eventually becomes a cardinality incident.

Test the same state machine before deployment: one active token per intended policy, single-use consumption, expiry rejection, resend behavior, generic responses that do not reveal account existence, and idempotent handling of repeated delivery callbacks. The message renderer needs snapshot or golden-file coverage for both formats, plus checks that every supported locale receives the same security meaning. These are application properties, so outsourcing transport does not outsource them.

## SMS segmentation changes the failure drill

Price per message is too narrow for this choice. Capacity planning should start with peak reset requests, resend amplification, queue drain time, provider or carrier delay, and the number of engineers able to diagnose each path. A secondary channel costs more than its traffic: it needs its own dashboards, runbooks, abuse controls, data-retention decisions, regional policy review, and failure exercises.

SMS also has a payload constraint that email does not share in the same form. GSM-7 messages can fit 160 characters in one segment, while UCS-2 messages can fit 70; longer messages are segmented, with reduced per-segment capacity. A localized OTP message that looks short on screen may therefore travel as multiple segments after one character changes the encoding. Treat rendered encoding and segment count as pre-send telemetry, and test every locale rather than assuming English behavior carries over.

Set budgets in units the on-call can act on: queue age relative to token lifetime, request-to-completion latency, and completion failure ratio. Keep vendor acceptance rates as supporting indicators. If a channel cannot expose enough delivery state to separate a transport delay from an application rejection, account for that uncertainty when setting the objective instead of manufacturing precision.

## Email should lose under three conditions

Ship the repository-owned email path behind a channel policy, not scattered conditional calls. Start with synthetic rendering and token tests, then exercise queue backlog, delayed delivery, duplicate callbacks, expired links, and regional slices in staging. In production, compare request cohorts with completed resets and support contacts; do not declare success from accepted sends.

Choose SMS as the primary recovery method only when the phone number is already verified, maintained through its lifecycle, and materially more available than email for the target users. Otherwise, choose the owned email template and short-lived link, with SMS reserved for a separately reviewed escalation policy. That conclusion is less glamorous than a multichannel abstraction, but it leaves one authoritative template contract and one default path for the team to operate.

The alert threshold is the last engineering choice, not a footnote. Set it too loose and users report the incident first. Set it too tight and normal regional delay wakes the on-call, repeated false pages teach responders to distrust the signal, and the SLO stops guiding action. Calibrate it with completion cohorts over the full expiry window, then page on sustained error-budget burn while routing narrow transport anomalies to non-paging diagnostics.

## Further reading

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio, SMS character limits and segmentation (GSM-7/UCS-2): https://www.twilio.com/docs/glossary/what-sms-character-limit
