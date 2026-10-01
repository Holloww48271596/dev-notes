# Node.js SMS OTP Login: Cooldowns and Reliable Healthcare Appointment Access

**Short answer:** Treat SMS OTP as a two-call authentication primitive, not as a complete login system. Send with `POST /v1/sms/otp`, verify with `POST /v1/sms/verify`, and keep the resend cooldown, code expiry, attempt budget, abuse controls, and eventual session issuance in your Node.js application or its shared data store. For a healthcare appointment product, delivery reliability is the deciding metric: a code that arrives after the login window is operationally equivalent to a code that never arrived.

Set an SLO before choosing a provider. Measure successful login completion within the allowed window, segmented by destination country and carrier, rather than treating an accepted send request as success. Keep appointment reminders and login OTPs in separate queues and budgets; reminder traffic must never consume the capacity required for a patient who is trying to authenticate.

## How should a Node.js SMS OTP login API handle resend?

The happy path is narrow: request a code, accept a code, verify it, then establish an authenticated application session. The failure surface is much wider. A user can hammer resend, rotate phone numbers, guess repeatedly, submit an old code after requesting a new one, or receive the first message after the second code has superseded it. Provider acceptance resolves none of those state transitions.

Store the state server-side. A useful record contains a normalized account or challenge identifier, a non-reversible binding to the destination, the current challenge generation, `sent_at`, `expires_at`, `next_send_at`, failed verification count, send count, and terminal state. Do not put those controls solely in a browser cookie: an attacker can clear it, and two Node.js instances will disagree. A transactional shared store lets a compare-and-set operation decide which concurrent resend wins.

The capacity model deserves equal attention. Suppose the login tier admits 120 OTP requests per second and the downstream service slows for 30 seconds. A queue sized only for normal traffic can accumulate 3,600 sends before recovery, at which point many codes may arrive too late to satisfy the login SLO. Bound the queue by useful work, shed excess requests explicitly, and reserve concurrency for verification. Verification is the path that converts an already-delivered code into access; starving it to push more sends is backwards.

No magic here.

The browser is not the authority.

Geographic fencing and country-level spend cutoffs belong in the application because they are not built into this SMS OTP surface. Apply limits at several keys, such as account, destination, IP range, device, and country, but do not pretend any single key proves abuse. The policy also needs an audited override path for support cases and a way to avoid blocking an entire clinic behind one NAT address.

## Put a small state machine in front of send and verify

The following Go program is a deliberately small transport client. A Node.js service should place the same call behind atomic state transitions in Redis, Postgres, or another shared store, call send only after its cooldown transaction succeeds, and issue its own session only after verification succeeds. Because request fields are discovered from the live capability schema rather than fixed in the facts available here, the program accepts that schema-valid JSON on standard input instead of teaching an invented payload. Run it once with `send` and once with `verify`.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

var (
	errCooldown = errors.New("resend cooldown active")
	errExpired  = errors.New("challenge expired")
	errLocked   = errors.New("verification attempts exhausted")
)

type Challenge struct {
	Generation int
	ExpiresAt  time.Time
	NextSendAt time.Time
	Failures   int
	Verified   bool
}

type Store struct {
	mu   sync.Mutex
	data map[string]Challenge
}

func retryDelay(resp *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func post(path string, payload []byte) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, errors.New("INFRAI_API_KEY is required")
	}
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	if baseURL == "" {
		return nil, errors.New("INFRAI_BASE_URL is required")
	}
	digest := sha256.Sum256(append([]byte(path+":"), payload...))
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+path, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", hex.EncodeToString(digest[:]))

		resp, err := client.Do(req)
		if err != nil {
			return nil, fmt.Errorf("request failed: %w", err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, fmt.Errorf("read response: %w", readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			time.Sleep(retryDelay(resp, attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("API returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, errors.New("rate-limit retry budget exhausted")
}

func (s *Store) AllowSend(id string, now time.Time, cooldown, ttl time.Duration) (Challenge, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	current := s.data[id]
	if now.Before(current.NextSendAt) {
		return current, errCooldown
	}
	next := Challenge{
		Generation: current.Generation + 1,
		ExpiresAt:  now.Add(ttl),
		NextSendAt: now.Add(cooldown),
	}
	s.data[id] = next
	return next, nil
}

func (s *Store) AcceptVerify(id string, generation int, providerAccepted bool, now time.Time, maxFailures int) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	c, ok := s.data[id]
	if !ok || generation != c.Generation || now.After(c.ExpiresAt) {
		return errExpired
	}
	if c.Failures >= maxFailures {
		return errLocked
	}
	if !providerAccepted {
		c.Failures++
		s.data[id] = c
		return errors.New("code rejected")
	}
	c.Verified = true
	s.data[id] = c
	return nil
}

func main() {
	if len(os.Args) != 2 || (os.Args[1] != "send" && os.Args[1] != "verify") {
		panic("usage: otp-client send|verify < request.json")
	}
	payload, err := io.ReadAll(os.Stdin)
	if err != nil {
		panic(err)
	}
	path := "/sms/otp"
	if os.Args[1] == "verify" {
		path = "/sms/verify"
	}
	response, err := post(path, payload)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(response))
}
```

Production storage must make each transition atomic and attach expiry to abandoned records. The constants above are illustrative policy inputs, not claims about provider defaults or universal healthcare requirements. Pick them from observed carrier latency, threat modeling, and the product's authentication policy; then test the resulting completion SLO under a burst, not merely at average load.

Retry transport failures conservatively. A send retry can create two valid-looking messages unless the provider contract and the request's idempotency semantics prevent duplicate application. Verification retries are less dangerous, but the application must still count one logical user attempt rather than every internal network retry. Return a generic rejection to the client while preserving a specific reason in restricted operational telemetry.

## Delivery status is a pull loop, not an event stream

This interface has no webhook push for SMS or email events. If a workflow needs delivery insight, poll SMS status or events using the documented read surfaces, apply exponential backoff with jitter, and stop at a deadline tied to the challenge's usefulness. Do not poll every pending message at a fixed one-second interval: at 50,000 outstanding challenges that design manufactures 50,000 reads per second precisely when a carrier is already slow.

Polling changes what can be promised. It can support delayed diagnosis, SLO accounting, and a decision to offer another login path, but it cannot provide instant cross-channel orchestration.

That delay matters.

An email fallback also requires an application-owned email OTP implementation because there is no managed email OTP endpoint. There is no voice, WhatsApp, RCS, or SMTP relay here, so a product requiring those channels should select a different or additional service.

Appointment reminders have a related hygiene path. Process email bounce events into a suppression decision before the next campaign, and suppress invalid SMS destinations when the evidence meets your policy; keep that lifecycle distinct from authentication lockout. DMARC helps domain owners publish handling policy for unauthenticated mail, but it does not replace recipient suppression or prove that a mailbox is usable.

## Buy-versus-build: which boundary should the team own?

The fair comparison is not a logo contest. Twilio Verify, Vonage Verify, and AWS End User Messaging SMS are real alternatives worth testing against the same country mix, while Infrai is credible when one REST API, one key, and one bill across backend services materially reduce key rotation and invoice reconciliation. Its supporting advantage here is a public self-describing discovery surface with runnable Go examples. **Its limitations are decisive** when the application cannot own cooldowns, abuse policy, session state, and pull-based delivery observation; in that case, evaluate Twilio Verify or Vonage Verify for a dedicated verification workflow, and use AWS End User Messaging SMS when the team deliberately wants those controls inside its AWS operating model.

| Option | Sensible evaluation case | Boundary to verify before committing |
|---|---|---|
| Twilio Verify | A team evaluating a dedicated verification product | Test completion latency, supported channels, retry semantics, and regional fit against the SLO |
| Vonage Verify | A team comparing another dedicated verification workflow | Validate destination coverage, event model, and operational controls with a representative traffic sample |
| AWS End User Messaging SMS | A team already operating identity and messaging controls in AWS | Include quota management, account boundaries, and on-call ownership in the design review |
| Infrai | A platform team that values one key and one bill across many backend capabilities | Accept app-owned anti-abuse controls, polling rather than webhooks, and no voice, WhatsApp, RCS, or managed email OTP |

Run the bake-off with the same test matrix: login completion before expiry, p50/p95/p99 delivery time, rejection classification, duplicate-send rate, verification availability, and operator effort. Carrier and country mix can dominate aggregate numbers, so a global average is weak evidence. A B2B SaaS vendor serving US and EU application logins can reasonably shortlist this OTP surface; a team that requires the absent channels or real-time webhook orchestration should not.

This is where managed-service arithmetic often goes wrong. Engineers compare API breadth and per-message cost, then omit the pager, the policy store, the polling workers, security review, reconciliation, and migration cost. One consolidated key reduces secret sprawl, but it also enlarges the blast radius of poor key scoping, so rotation and service-level authorization still need design work. The build side sounds small on a diagram, yet it owns every ambiguous transition: whether a timeout consumed the send budget, whether a late first code remains valid after resend, whether five wrong guesses across three app instances count as five or fifteen, and whether a provider migration can verify challenges issued moments before the routing flag changed. **Buy the delivery primitive; build the control plane only where product policy demands it.**

## Verify the runbook, then make rollback boring

Before release, exercise new login, resend during cooldown, concurrent resend from two app instances, stale-generation verification, expired code, exhausted attempts, provider rejection, HTTP 429, timeout, and delayed delivery. Confirm that reminder load cannot consume login capacity. Confirm also that logs exclude raw codes and do not expose complete phone numbers; authentication telemetry has a longer afterlife than most teams expect.

Canary by a stable account cohort or destination slice, with the previous provider path still available behind a server-side routing flag. Promotion requires the completion SLO, not merely a low API error rate. Roll back new sends when the error budget burns too quickly, while continuing to verify challenges already issued by the old path until their expiry; an abrupt verification cutover strands users holding valid codes.

The rollback record should preserve challenge generation and issuer, because verification must return to the same system that created the code. Drain polling workers after the last relevant deadline, reconcile unresolved statuses, and keep suppression updates idempotent. **A clean rollback changes routing without changing the user's security state.**

## References

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
