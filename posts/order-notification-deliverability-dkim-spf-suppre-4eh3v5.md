# Order Notification Deliverability: DKIM, SPF, Suppression, Bounces, and Complaints

An accepted order-notification request is not a delivery result, and a template that can change between attempts makes the evidence harder to trust. **Short answer:** assign every notification an immutable template version and sending identity, verify DKIM, SPF, and domain state before release, then apply bounce and complaint feedback to an authoritative suppression check immediately before each send.

That choice puts template ownership at the center of troubleshooting. The owner is accountable for the rendered message, headers, domain selection, rollback, and the evidence retained for replay; the transport remains accountable for acceptance and later feedback. Mixing those responsibilities produces a familiar incident pattern: the API call looks healthy, an order email never reaches its intended recipient, and three teams inspect three different versions of “the same” message.

I would bound that production scenario tightly. Take one `order.confirmed` event, one pseudonymous recipient, and one attempt ID. Start the clock when the event became eligible, stop it at a terminal classification, and refuse to substitute aggregate delivery charts for the missing per-attempt evidence. This isn't a claim that one trace explains the whole incident. It is the fastest way to discover which boundary has stopped producing trustworthy facts.

The invariant is stricter than “the API returned success”: one event must resolve to one recorded template version, one sending identity, one fresh recipient-policy decision, and one terminal outcome. Everything else is supporting evidence.

## Template governance starts at release review

There are three common ownership arrangements, and none wins everywhere. Central platform ownership gives one team control over the sending identities, rendering contract, and rollback path, but ordinary copy changes can queue behind shared review. Domain-team ownership keeps templates beside the order schemas they consume, but every team must implement the same versioning, recipient-policy, and telemetry requirements. A managed template boundary can reduce renderer operations while moving version history, release control, and replay evidence outside the application boundary.

The useful review question is not “where is the HTML stored?” Ask who can answer, during an incident, which immutable version rendered the event, which identity signed it, what changed since the last release, and how to reproduce the render without editing production state. If answering requires screenshots from several consoles and an unversioned text field, the ownership model has already failed the operability test — even if messages usually arrive.

| Ownership model | Strong fit | On-call and capacity cost | Main limitation |
|---|---|---|---|
| Platform-owned templates | Uniform policy and identity controls matter most | One team sizes rendering and owns rollback | Product changes share a central review queue |
| Domain-owned templates | Copy and event schemas change together | Each team owns tests, peaks, and telemetry | Governance can drift without enforced contracts |
| Managed template boundary | The team wants less renderer operation | External limits and evidence enter SLO planning | Replay and migration depend on exported artifacts |

Template ownership also defines the safe signing boundary. Record the final rendered artifact or a protected digest before handoff, along with the immutable version that produced it. If a component can mutate headers or content after that record is made, include that component in the trace; otherwise responders may repeatedly inspect DNS while the evidence mismatch actually came from a later transformation.

Release gates should exercise the real template contract with non-customer recipients, verify the intended sending identity, and confirm that required attempt metadata reaches observability. They should also prove rollback. A deployment that can publish a template but cannot quickly restore its predecessor transfers change speed into on-call risk.

Troubleshooting should follow the notification's state transitions in order, even when the incident arrived through a customer complaint. Confirm that the business event exists and has a stable identifier. Confirm that the intended notification policy selected email. Recover the template version and its owner, the envelope and visible sender domains, the recipient-policy result, the transport attempt ID, and any later feedback. If a link is missing, stop there; a downstream chart cannot reconstruct an upstream decision that was never recorded. For an e-commerce platform, the difference between an order receipt and a back-in-stock alert matters. Their deadlines, retry policies, and user consequences need not be equal. I'm not sure a single availability target can represent both honestly; the product owner has to define each promised window, after which the platform can set SLOs for queue age, render latency, transport acceptance, and feedback-processing lag. Until that decision exists, a green monthly average can conceal a receipt that arrived after the customer contacted support. Keep the state vocabulary narrow. An ineligible recipient is a policy outcome. A render rejection is a template or schema outcome. A domain that has not passed the release gate is a configuration outcome. Transport acceptance is an intermediate outcome. A bounce or complaint is asynchronous feedback. These classes need separate counters and owners because combining them into `delivery_failed` makes the number easy to graph and nearly useless to act on. Capacity planning belongs in the same trace. Peak order traffic can increase queue age, which widens the interval between an early recipient check and the actual attempt. Feedback-consumer lag can leave an invalid recipient eligible longer than intended. Daily volume does not expose either risk, so alert on the age of the oldest eligible notification and the age of the oldest unprocessed feedback item, with thresholds derived from the notification SLO rather than from a convenient round number.

Start there.

Keep full addresses and message bodies out of broad operational logs. A durable incident record usually needs the event ID, recipient pseudonym, template version, template owner, sending identity, policy decision time, attempt ID, and terminal class. Access to rendered content should be narrower and auditable.

## How should an event notifications API troubleshoot DKIM, SPF, bounces, and complaints?

Treat authentication, transport, and recipient policy as separate checkpoints. DKIM associates a signed message with a domain and allows a verifier to validate that signature under the mechanism defined by RFC 6376. Record the signing domain and selector used for the attempt, but do not treat the presence of a signature as proof of inbox placement. Likewise, retain the SPF and domain-verification observations produced by the deployment check without pretending they explain a later complaint or an invalid mailbox.

The practical investigation is a decision tree, not a retry loop:

1. If the event or template version is missing, investigate production and rendering ownership before transport.
2. If the sending identity did not pass its release verification, stop the rollout and repair that configuration before generating more attempts.
3. If the recipient was already suppressed at the final policy check, classify the notification as intentionally blocked rather than failed.
4. If transport accepted the attempt, wait for and correlate asynchronous feedback instead of declaring delivery from the synchronous API response.
5. If bounce or complaint feedback arrives, apply it idempotently and make the updated policy visible to later attempts.

Order matters.

Suppose event `ord_7F31` becomes eligible at 10:00:00, feedback from an older campaign suppresses the same recipient at 10:00:02, and a delayed worker wakes at 10:00:05. A policy snapshot attached to the order event is stale by the time it matters. A fresh lookup immediately before transport prevents the later attempt. Those timestamps are an illustrative race, not measured performance, but they give a concurrency test a precise shape.

Don't automatically turn an email bounce into an SMS notification. SMS has its own API behavior, consent rules, status evidence, templates, and operational limits; the existence of an SMS transport does not authorize a cross-channel send. A product rule must decide whether fallback is allowed before an incident occurs.

## A state machine turns feedback into send policy

Bounce and complaint ingestion should be idempotent because callbacks can be repeated, reordered, or processed by more than one consumer. The store must preserve enough provenance to explain why a recipient is suppressed, while the send worker needs a small, dependable decision surface. It doesn't need to know the feedback provider's entire payload.

The Go example keeps provider parsing outside the policy core. It rejects incomplete records, ignores duplicate feedback IDs, prevents older feedback from replacing newer state, and requires every attempt to ask again just before transport. The in-memory implementation is deliberately a contract example; production durability, uniqueness, and concurrency guarantees belong in the selected datastore.

```go
package suppression

import (
	"context"
	"errors"
	"sync"
	"time"
)

type Reason string

const (
	Bounce    Reason = "bounce"
	Complaint Reason = "complaint"
)

type Feedback struct {
	ID         string
	Recipient  string
	Reason     Reason
	OccurredAt time.Time
}

type State struct {
	Suppressed bool
	Reason     Reason
	ChangedAt  time.Time
	FeedbackID string
}

type Store interface {
	Apply(context.Context, Feedback) error
	Get(context.Context, string) (State, error)
}

type MemoryStore struct {
	mu     sync.RWMutex
	state  map[string]State
	seenID map[string]struct{}
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		state:  make(map[string]State),
		seenID: make(map[string]struct{}),
	}
}

func (s *MemoryStore) Apply(_ context.Context, f Feedback) error {
	if f.ID == "" || f.Recipient == "" || f.OccurredAt.IsZero() {
		return errors.New("incomplete feedback")
	}
	if f.Reason != Bounce && f.Reason != Complaint {
		return errors.New("unknown feedback reason")
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	if _, exists := s.seenID[f.ID]; exists {
		return nil
	}
	s.seenID[f.ID] = struct{}{}

	current := s.state[f.Recipient]
	if current.Suppressed && !f.OccurredAt.After(current.ChangedAt) {
		return nil
	}
	s.state[f.Recipient] = State{
		Suppressed: true,
		Reason:     f.Reason,
		ChangedAt:  f.OccurredAt,
		FeedbackID: f.ID,
	}
	return nil
}

func (s *MemoryStore) Get(_ context.Context, recipient string) (State, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.state[recipient], nil
}

func EligibleBeforeSend(ctx context.Context, store Store, recipient string) error {
	state, err := store.Get(ctx, recipient)
	if err != nil {
		return err
	}
	if state.Suppressed {
		return errors.New("recipient suppressed")
	}
	return nil
}
```

Test the orderings that invalidate optimistic dashboards: the same feedback ID twice, older feedback after newer feedback, concurrent consumers, a worker waking after suppression changes, and a retry after an earlier attempt. Test template selection separately using a fixed event and immutable version. An attempt record is audit evidence, not permission for the next retry; each attempt requires a new policy read.

There is still a race between the last policy read and transport handoff. Don't claim atomicity unless the policy store and transport participate in a protocol that provides it. Record both times, bound the interval, and make processing idempotent. Honest limits beat a diagram with imaginary guarantees.

## Migration rehearsal reveals the real exit cost

Build the policy core when suppression semantics are a product rule, several transports must share them, and the team can own durable storage, privacy controls, callback normalization, and a staffed SLO. Buy more of the pipeline when reducing parser maintenance and renderer on-call work matters more than controlling every internal representation. A hybrid is often reasonable: external transports at the edge, an internal canonical feedback record, and one authoritative eligibility decision close to the send path.

The catch is operational ownership. A small team without datastore expertise should not self-host the suppression authority merely to avoid dependency lock-in; the resulting backup, migration, capacity, and paging burden can exceed the control gained. Conversely, a regulated workflow that requires locally governed audit retention may find an externally owned template history unsuitable. Stick with domain-owned templates when event-schema and copy releases must move together, and stick with platform ownership when consistent authentication and recipient policy outweigh local release speed.

Before choosing, run a rollback exercise and a feedback-lag exercise. Export immutable template artifacts. Estimate peak render work from the event burst, not the daily average. Decide who pages when feedback age threatens the notification SLO. Then verify that a transport can be replaced without rewriting the business event contract; reducing migration scope is more defensible than assuming migration will never happen.

This advice does not apply unchanged to purely internal mail with a controlled recipient directory, nor to a system where templates cannot affect identity, headers, or recipient policy. Even there, acceptance remains different from a terminal outcome, but the appropriate evidence and suppression rules may be much smaller. Your mileage may vary because the missing input is organizational: who can actually roll back a template and restore trustworthy state at 02:00?

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://www.twilio.com/docs/sms
