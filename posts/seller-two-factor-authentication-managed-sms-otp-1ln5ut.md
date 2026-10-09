# Seller Two-Factor Authentication: Managed SMS OTP Beats Self-Hosting Until Audit Control Dominates

**TL;DR:** For a marketplace that must notify a seller about a new order, I would start with a managed SMS challenge rather than operate the SMS delivery path, but I would keep challenge state, throttling, audit records, and recovery codes in the application boundary. Build the delivery path only when recovery policy, regional routing, or measured reliability requirements exceed what the managed boundary can prove. The login endpoint is the easy part; bounded retries, single-use state, and an auditable recovery path are the actual system.

This is a conditional choice, not a blanket endorsement. A managed challenge removes carrier integration from the first on-call rotation, while an owned challenge service gives the team more control and also makes it responsible for capacity, routing, abuse, and delivery evidence. For a seller trying to open a new-order notification, the relevant SLO is successful, safe access before the order workflow becomes stale, not the provider's API acceptance rate.

## How should NestJS handle SMS OTP two-factor authentication?

I use a deliberately bounded incident scenario in design reviews: a seller receives an order notification, signs in from a new device, requests an SMS code twice because the first message is delayed, and then reaches for a recovery code. Nothing in that sequence is exotic. It still exposes four independent failure domains: message delivery, replay prevention, throttle consistency, and account recovery. Now add two application replicas. The first request creates challenge A, a resend reaches the other replica before shared state is visible, and challenge B is issued; the seller enters A after B has become authoritative, tries again, and consumes the attempt budget without ever making a typo. A support operator sees two accepted message requests but cannot tell which challenge was valid because the messaging receipt and login record use unrelated identifiers. This is the failure I want a design to make impossible, rather than a reason to raise the retry limit after deployment.

Two replicas are enough.

The dangerous design treats “SMS accepted” as “seller authenticated.” Those events are not equivalent. Queue acceptance can be evidence that a delivery attempt began; it cannot prove that a handset received a message, that the intended person read it, or that the subsequent code verification was legitimate. NIST also treats use of the public switched telephone network for out-of-band authentication as restricted, and asks verifiers to consider risks such as SIM change and number porting. That boundary belongs in the threat model, even when SMS remains a practical factor for the user population.

Short delays are normal enough that a resend button cannot mean “mint unlimited fresh codes.” The application should preserve one active challenge window, enforce budgets against both the account and a broader abuse key, and make the state transition atomic. Otherwise two replicas can each observe remaining allowance and both issue a challenge. The race is small. Its blast radius is not.

The invariant is straightforward: one challenge has one terminal outcome, and every attempt that could affect access leaves a correlated security event. Delivery attempts, verification failures, successful consumption, expiry, throttle denial, and recovery-code use need stable event names and a challenge identifier. Never put the OTP or a recovery secret into that log.

No secret enters the log.

## Managed challenge or owned delivery?

The choice should be made against the team's reliability budget, not a feature checklist. “Managed” below means an external service owns code delivery and possibly verification. “Owned” means the application mints and verifies challenges and directly integrates the messaging path; it does not imply running a cellular network.

The pager decides.

| Decision area | Managed challenge | Owned delivery and verification |
|---|---|---|
| Initial on-call surface | Carrier routing and delivery integration stay outside the application team | The team owns routing, retries, provider failover, and delivery telemetry |
| Challenge control | Provider semantics constrain expiry, resend, and evidence | Application can enforce one state machine across SMS and recovery |
| Audit evidence | Must be reconciled with provider callbacks and local login events | Can originate from one transaction boundary, while delivery still needs external evidence |
| Lock-in pressure | Challenge identifiers and callback semantics can leak inward | Messaging adapters are portable, but operations and abuse controls become local obligations |
| Capacity planning | Plan request ceilings and degraded-provider behavior | Plan issuance, verification, queues, callback ingestion, storage, and regional routing |
| Exit condition | Good while the service can meet the documented control and evidence requirements | Justified when measured gaps matter more than the additional on-call load |

My default is the managed column because the first platform milestone should reduce the number of failure domains the on-call team must diagnose at 03:00. I still keep a provider-neutral application record. That record prevents a vendor callback schema from becoming the authentication model and provides a clean migration boundary if delivery evidence or regional coverage stops meeting the service objective.

There is a firm exception. If regulation or internal policy requires a recovery workflow and audit chain that the managed challenge cannot represent, owning the state machine may be less risky than inventing reconciliation around a black box. The same is true after real traffic shows that routing controls are material to the seller-access SLO. Evidence should trigger that move; architectural taste should not.

I would not build it earlier.

## A preventative state transition

The following Go example is intentionally focused on the verification boundary rather than an HTTP framework. A NestJS controller should be thin for the same reason: parsing transport input and mapping the authenticated principal are separate from consuming a challenge. The durable operation is an atomic store transition that checks expiry, attempt allowance, code proof, and prior consumption together.

```go
package auth

import (
	"context"
	"crypto/sha256"
	"crypto/subtle"
	"errors"
	"time"
)

var (
	ErrDenied  = errors.New("challenge denied")
	ErrExpired = errors.New("challenge expired")
)

type Challenge struct {
	ID          string
	SellerID    string
	CodeDigest  [32]byte
	ExpiresAt   time.Time
	Attempts    int
	MaxAttempts int
	ConsumedAt  *time.Time
}

type AuditEvent struct {
	Name        string
	ChallengeID string
	SellerID    string
	OccurredAt  time.Time
}

type Store interface {
	// Consume runs fn under the store's transaction or compare-and-swap boundary.
	Consume(ctx context.Context, id string, fn func(*Challenge) error) error
	AppendAudit(ctx context.Context, event AuditEvent) error
}

type Verifier struct {
	Store Store
	Now   func() time.Time
}

func (v Verifier) Verify(ctx context.Context, challengeID, submitted string) error {
	now := v.Now()
	err := v.Store.Consume(ctx, challengeID, func(c *Challenge) error {
		if c.ConsumedAt != nil || c.Attempts >= c.MaxAttempts {
			return ErrDenied
		}
		if !now.Before(c.ExpiresAt) {
			return ErrExpired
		}

		c.Attempts++
		got := sha256.Sum256([]byte(submitted))
		if subtle.ConstantTimeCompare(got[:], c.CodeDigest[:]) != 1 {
			return ErrDenied
		}

		c.ConsumedAt = &now
		return nil
	})

	name := "sms_challenge_verified"
	if err != nil {
		name = "sms_challenge_denied"
	}
	_ = v.Store.AppendAudit(ctx, AuditEvent{
		Name: name, ChallengeID: challengeID, OccurredAt: now,
	})
	return err
}
```

This snippet omits issuance on purpose. Production issuance needs an unpredictable code generated with a cryptographically secure random source, a short server-enforced lifetime, and a stored verifier rather than plaintext. OWASP recommends short lifetimes, single use, strict attempt limits, invalidation after success, and no long-term plaintext storage for OTPs. A bare fast hash is also insufficient protection if the database is copied, because a six-digit search space is small; bind the verifier to a server-held secret or use an equivalent construction designed for this threat.

The audit append in the compact example is best-effort, which is not strong enough for a security evidence requirement. In the deployed design, write the challenge transition and an outbox record in the same database transaction, then publish the audit event asynchronously with an idempotency key. This avoids granting access while silently losing the evidence event. The consumer must tolerate duplicates.

Capacity planning starts with the abuse case. If legitimate peak order traffic produces `L` login starts per second, the issuance path is not sized at `L`; it is bounded by an explicit abuse budget, queue age objective, store write capacity, and the maximum callback backlog the team can drain after an outage. I want alerts on verification success ratio, throttle denials, p95 challenge age at success, callback age, and recovery-code use. Provider acceptance belongs on the dashboard, but it is a dependency signal rather than the access SLI.

## Recovery codes are another factor, not a bypass

Recovery codes should be generated from a cryptographically secure source, displayed once, stored as protected verifiers, and consumed atomically. Each code is single use. A successful recovery must revoke that code and emit a high-signal audit event; depending on policy, it may also invalidate outstanding SMS challenges and notify the account through an independent channel.

Do not let support staff read or mint a replacement code as an informal escape hatch. Account recovery is usually the path with fewer signals and higher social-engineering pressure, so its authorization rules, review evidence, and notification behavior deserve their own threat model. NIST's account-recovery guidance describes saved recovery codes as a recovery method and sets requirements for how they are stored and used.

There is also a user-facing reliability trade-off. Invalidating every recovery code after one use limits replay, but it creates a dead end if the user loses the remaining set and the phone number is unavailable. The answer is a defined re-enrollment ceremony with stronger evidence, not a permanent master code. Keep it boring.

## Release gates and the boundary of this advice

Before rollout, test two verification requests racing against the same challenge, a resend racing with expiry, duplicate delivery callbacks, an audit consumer replay, clock skew at the expiry boundary, and exhaustion of both account-level and network-level budgets. Tests should assert the final database state and emitted event set, not merely an HTTP status. Logs must demonstrate correlation without containing submitted codes, phone numbers in full, or recovery material.

Deploy the state transition behind a small cohort and compare access outcomes across regions and carriers without weakening the global abuse ceiling. A rollback may disable new issuance, but it must not resurrect consumed challenges. Runbooks need a degraded mode for delayed SMS and a separate procedure for recovery review; “increase retries” is not a degraded mode because it can amplify both cost and abuse during a provider incident.

This recommendation does not apply when SMS is prohibited by the system's assurance policy, when the user population cannot reliably receive it, or when phishing-resistant authentication is required. NIST identifies WebAuthn/FIDO2 as an example of phishing-resistant authentication, while manually entered OTP methods are not phishing-resistant. In those cases, change the factor rather than polishing the SMS pipeline.

For the marketplace case, the decision remains asymmetric: buy the delivery capability first, own the authorization evidence from day one, and bring more of the path in-house only after measured SLO gaps or control requirements justify another service on the pager.

## Sources

- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc6238
- https://www.w3.org/TR/webauthn-3/
