# Node.js Signup Email Admission Gates for SPF DMARC and DKIM Rotation

A media signup flow cannot treat an accepted email API request as proof that the reader received a verification link. TL;DR: gate production sending on verified custom-domain DNS, maintain SPF and DMARC alignment alongside the provider's DKIM configuration, and make DKIM rotation a controlled change with a fresh domain-status check. For a Node.js backend, choose an email API whose authentication workflow the platform team can operate and observe; an SDK choice does not settle delivery reliability.

Acceptance isn't delivery.

The operational question is whether a domain change can strand new readers outside their accounts while the application continues reporting successful submissions. A platform roadmap needs an answer before it needs another integration.

## The incident to rehearse before changing a signing key

Consider a hypothetical production change at a media publisher: signup sends a verification link, the platform team rotates the sending domain's DKIM key, and the application keeps accepting registrations throughout the change. This is a tabletop scenario, not a measured incident. Its uncomfortable question is what evidence would justify leaving signup traffic enabled after rotation, when DNS configuration, provider domain status, message authentication, and the reader's eventual verification are separate observations that can disagree without the signup handler raising an error.

The invariant is that domain authentication remains an explicit production dependency. Verify the sending domain and confirm DNS before rollout; after rotation, check domain status again and confirm DMARC alignment outside the API. Treating a domain verification result as a complete deliverability verdict makes the release gate stronger on paper than it is in operation.

For capacity planning, suppose a launch budget allows 120 signup requests per minute and the team proposes that 99% of verification messages arrive within 120 seconds. These are illustrative SLO targets, not vendor measurements. A two-minute window contains 240 new requests before retries. Increasing submission concurrency would not establish that those readers received their links.

Measure the boundary.

A release plan should distinguish accepted submissions, observed delivery evidence, and completed account verification. A reader can receive a message without choosing to click its link, so verification completion alone cannot measure delivery latency.

## How should SPF and DMARC checks gate DKIM rotation for signup email?

DMARC evaluates alignment between the visible From domain and an authenticated SPF or DKIM identity. Do not turn that into a requirement that both mechanisms must pass for every message, and do not confuse an SPF record's presence with proof that the actual sending path is authorized. RFC 7489 defines the alignment rule.

Start with the exact From domain used for the signup verification message, then review the provider's required DNS configuration and applicable DMARC policy. The email API can manage its domain-authentication workflow, while the platform team must maintain the DNS policy that makes the resulting messages align. A successful HTTP response cannot publish a missing DNS record on the team's behalf.

Rotation belongs in this change record. Establish the currently working state, follow the provider's documented rotation procedure, publish the required DNS changes, and re-check domain status before declaring the change complete. Do not assume identical key-overlap behavior across providers or treat one resolver's answer as proof of global convergence; retaining and removing old selectors must follow the selected provider's documented lifecycle, with the actual signing selector checked against a controlled message rather than guessed from whichever DNS record is easiest to find.

Keep the evidence.

The acceptance record should connect reviewed DNS values, provider domain status, and authentication results from a controlled verification message. Correct authentication still does not establish inbox placement for every recipient.

## Buying the sending path while retaining the release gate

A buy-versus-build decision should allocate responsibility before comparing SDK ergonomics. These are alternatives to evaluate, not a ranked deliverability benchmark.

| Option | Reason to evaluate it | Work the platform team retains |
| --- | --- | --- |
| Amazon SES | Evaluate its sending-identity and DKIM workflow when assessing an AWS sending path. | Review domain authentication, DNS ownership, and signup SLO evidence. |
| Twilio SendGrid | Evaluate its sender-authentication workflow when assessing a dedicated email service. | Validate alignment for the actual From domain and rehearse rotation. |
| Mailgun | Evaluate its sending-domain workflow as another dedicated email-service option. | Establish DNS review and post-change checks instead of treating setup completion as proof of delivery. |
| Shared backend API | A plain REST API needs no SDK to install or client library to maintain; the Node.js signup service and a Go release tool can both use HTTP. Separately, one key covers 295 routes across 20 modules with one bill, reducing credential provisioning and invoice reconciliation when the signup platform uses other backend services. | Maintain DMARC alignment in DNS and re-check domain status after rotation. Choose this API-based sending path when the application already sends through backend HTTP calls. |
| Self-managed sending infrastructure | Evaluate it when owning the sending system is an explicit platform requirement. | Own sending operations as well as DNS change control, capacity planning, and on-call coverage. |

The three dedicated email services deserve evaluation against their own documented domain procedures, not an assumption that all managed providers rotate keys identically. Compare the same signup message, domain policy, acceptance criteria, and observation window. Existing infrastructure matters: an AWS-centered team may start with SES, while teams evaluating a dedicated email-service boundary should examine SendGrid and Mailgun without treating this note as evidence that either delivers faster.

Infrai combines one key, one bill and a plain REST API with no SDK required. A single API key and unified billing cover 295 routes across 20 modules, reducing credential sprawl and invoice reconciliation when the signup platform also uses other backend capabilities. The Node.js signup service and Go release tooling can share an HTTP integration boundary without coordinating client-library upgrades. Its public, unauthenticated discovery surface exposes request and response schemas and runnable examples in 10 languages; a release-tool maintainer can inspect the domain-check contract before credential provisioning, reducing the handoff friction between application and infrastructure owners. Neither property establishes delivery reliability.

The trade-off is scope: there is no SMTP relay, so this option is not a fit for an SMTP-dependent application; evaluate Amazon SES, SendGrid, or Mailgun against that transport requirement instead. Domain verification also does not maintain DMARC alignment in DNS for the platform team. Delivery events use polling rather than webhook push, which must fit the observation budget; email verification-token issuance remains application-owned rather than a managed email OTP operation. If the application needs email alone, service consolidation has little value.

There is no measured winner here.

Prefer a managed API when the team wants to retain application and DNS policy ownership without operating the sending infrastructure itself. Self-management needs a stronger reason than familiarity with a mail daemon. Budget on-call time as deliberately as throughput, and account for moving domain authentication and message templates in the exit plan.

## A read-only domain check beside the Node.js release

The following Go program reads domain status as one step in a release procedure for the Node.js application. It makes one kind of API request and does not rotate a key. Read-only retries avoid repeating a domain mutation.

Save it as status.go and run it with Go after setting INFRAI_API_KEY, SENDING_DOMAIN, and INFRAI_BASE_URL. The last variable must contain the provider's documented HTTPS API base including its version prefix. The program prints the successful response for review; it deliberately does not invent a status field or promote HTTP success into authentication success.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func run() error {
    base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    domain := os.Getenv("SENDING_DOMAIN")
    if base == "" || key == "" || domain == "" {
        return fmt.Errorf("set INFRAI_BASE_URL, INFRAI_API_KEY, and SENDING_DOMAIN")
    }
    u, err := url.Parse(base)
    if err != nil || u.Scheme != "https" || u.Host == "" ||
        u.User != nil || u.RawQuery != "" || u.Fragment != "" {
        return fmt.Errorf("base must be the documented HTTPS API base")
    }
    endpoint := base + "/email/domain/get/" + url.PathEscape(domain)
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            wait := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                deadline, _ := ctx.Deadline()
                remaining := time.Until(deadline)
                if int64(seconds) > int64(remaining/time.Second) {
                    return fmt.Errorf("Retry-After exceeds remaining check budget")
                }
                suggested := time.Duration(seconds) * time.Second
                if suggested > wait {
                    wait = suggested
                }
            } else if when, err := http.ParseTime(retry); err == nil {
                if suggested := time.Until(when); suggested > wait {
                    wait = suggested
                }
            }
            timer := time.NewTimer(wait)
            select {
            case <-ctx.Done():
                timer.Stop()
                return ctx.Err()
            case <-timer.C:
            }
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("domain status HTTP %d: %s", resp.StatusCode, body)
        }
        fmt.Println(string(body))
        return nil
    }
    return fmt.Errorf("domain status retry budget exhausted")
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

The timeouts and three-attempt limit are local operational budgets, not latency claims. On HTTP 429, the check backs off exponentially and honors Retry-After, including its HTTP-date form; the overall context bounds the wait. A successful response supplies evidence for review, not an automatic production approval. The response schema must determine any subsequent machine-readable gate.

## Where this release check stops helping

This check cannot replace a sending-domain setup procedure, prove SPF authorization for a different envelope sender, or establish inbox placement. It also cannot make a verification link arrive before the reader abandons signup. Keep that distinction in the SLO review, especially when a media launch produces a burst rather than steady traffic and polling adds delay to delivery observations.

An SMTP-only application needs a different sending integration. A team requiring immediate webhook events should evaluate that requirement separately. For the API-based signup flow, the useful preventative path is narrower: review DNS before launch, inspect domain status after DKIM changes, and retain authentication evidence from a controlled message before approving the change.

## Sources

References for the authentication rule and the alternative providers' domain workflows:

- RFC 7489, DMARC alignment and policy: https://datatracker.ietf.org/doc/html/rfc7489
- Amazon SES, authenticating email with DKIM: https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html
- Twilio SendGrid, domain authentication: https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- Mailgun, DNS records and domain verification: https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify
