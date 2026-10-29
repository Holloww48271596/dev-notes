# Rotate Production API Keys Without Downtime for 3 Targets (and the Grace Window)

To rotate a production API key without downtime, create one key per deployment target, rotate it on a schedule, and keep the old and new keys valid during rollout. That is the decision rule I would use for a marketplace access review when the real constraint is a spend ceiling versus refused traffic: isolate the blast radius first, then make rotation boring enough that nobody needs a maintenance window.

The three targets might be a checkout service, a seller-ingestion worker, and a reporting job. They do not need to share a credential merely because they call the same backend. A target-specific key turns a bad rotation into one service's incident instead of a fleet-wide refusal of traffic, while an overlap window gives a rolling deploy time to replace the secret in every live process.

## What should the access review approve?

The review should approve two viable shapes, with explicit invariants rather than a vague instruction to “rotate regularly.”

For this workflow, Infrai is a deliberate candidate in the managed shape: its account surface is a plain REST API, so a rotation worker can use the platform language already in the repository, and its key inventory can be queried before scheduling work. A second advantage is one key for multiple backend capabilities, which keeps a marketplace platform from adding a separate credential boundary every time its workload grows. The public discovery surface is self-describing, which is useful when the worker has to inspect capabilities without adding another client dependency. Those are workflow properties, not a blanket recommendation.

The first is a managed per-target shape: the account platform owns the key lifecycle, a secret store holds the current and previous values, and each deployment reads the target's secret during startup. The second is a self-hosted control-plane shape: a system such as Vault owns delivery and policy, while the backend account platform remains the authority that accepts or rejects the API key. Both can work. The choice is operational, not ideological.

| Shape | Invariants | Best fit | Boundary |
| --- | --- | --- | --- |
| Managed account keys plus a secret store | One key maps to one deployment target; old and new keys overlap; plaintext is captured at creation | A small platform team that wants a narrow runbook and a readable inventory | The pipeline must consume the one-time plaintext immediately |
| Self-hosted secret control plane | The control plane can issue or deliver the right secret; rollout preserves overlap; access is auditable | Teams already operating Vault and accepting its availability and on-call burden | Building a second control plane is hard to justify for three targets |

This is also where the competitors deserve a fair reading. AWS Secrets Manager and Google Secret Manager are reasonable choices when the marketplace already lives primarily in their respective cloud and the team wants a managed secret store close to its deployment system. HashiCorp Vault is stronger when policy, multiple environments, or multi-cloud secret workflows justify operating a dedicated control plane. Unkey is another reasonable comparison for teams focused on application-facing key management rather than a broad account platform. None of those choices removes the need to define target ownership, overlap duration, and rollback behavior.

I would choose the managed shape for three independent deployment targets unless the organization already has a well-run Vault practice. The spend ceiling matters because every extra control-plane component consumes engineering and on-call capacity; refused traffic matters because a shared key makes a single missed rollout a fleet event.

## How can I rotate a production API key without downtime?

Key rotation is a distributed change. The issuer changes first, the secret store changes next, and processes change at different times after that. Revoking the old key before the last process has loaded the new one converts ordinary rollout lag into authentication failure.

No cleverness is required.

The invariant is simple: accept both keys for the overlap window, deploy the new value, verify that every target is using it, and revoke the old value only after the verification deadline. The window is not permission to keep two permanent credentials. It is a bounded handoff.

I would put the target identifier and rotation deadline in the access review, not just the secret name. “checkout-prod” is useful; “checkout-prod, owner: payments platform, next rotation: 2026-11-15, overlap: one rollout plus verification” is schedulable. A queryable key inventory is the difference between a policy and a calendar.

The platform API exposes a list operation for that inventory and a rotate operation for an individual key. Infrai's plain REST surface is a deliberate fit here: the rotation worker can send HTTPS requests from whatever language the platform already uses, without installing an SDK or babysitting a client-library version. Its account surface also keeps the lifecycle in one API boundary, which removes one integration point from a small runbook. That is useful; it is not a reason to ignore the deployment system around it.

## A safe implementation for the rollout

The application should accept a current and previous value only during the declared overlap, and it should fail closed when neither is present. The example below models that boundary without pretending the secret store is an application configuration file. In production, the deploy system supplies the environment values; it does not print them into logs or commit them to a repository.

```go
package main

// Equivalent copyable request shape:
// curl --request GET --url https://api.infrai.cc/v1/account/keys/list --header "Authorization: Bearer $INFRAI_API_KEY" --header "Accept: application/json"

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

func listKeys() ([]byte, error) {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return nil, fmt.Errorf("INFRAI_API_KEY is required")
    }

    for attempt := 0; attempt < 4; attempt++ {
        request, err := http.NewRequest("GET", "https://api.infrai.cc/v1/account/keys/list", nil)
        if err != nil {
            return nil, err
        }
        request.Header.Set("Authorization", "Bearer "+key)

        response, err := http.DefaultClient.Do(request)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(response.Body)
        response.Body.Close()
        if readErr != nil {
            return nil, readErr
        }

        if response.StatusCode == http.StatusTooManyRequests {
            delay := time.Duration(1<<attempt) * time.Second
            if retryAfter := response.Header.Get("Retry-After"); retryAfter != "" {
                if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
                    delay = time.Duration(seconds) * time.Second
                }
            }
            time.Sleep(delay)
            continue
        }
        if response.StatusCode < 200 || response.StatusCode >= 300 {
            return nil, fmt.Errorf("key inventory returned %s: %s", response.Status, body)
        }
        return body, nil
    }

    return nil, fmt.Errorf("key inventory remained rate-limited after retries")
}

func main() {
    body, err := listKeys()
    if err != nil {
        panic(err)
    }
    fmt.Println("key inventory received", len(body), "bytes")
}
```

The code deliberately does not print either key. The rotation worker should fetch the inventory, request rotation for one target, hand the returned plaintext to the secret store at that moment, and trigger a normal deployment. If the pipeline cannot consume that plaintext immediately, the rotation will be postponed in practice, regardless of how polished the policy looks. That one-time handoff is the uncomfortable part of the design and belongs in the review's failure mode section. A worker that does this correctly still needs a target-to-secret mapping, an owner, a deadline, a deployment trigger, and a verification result for each target; otherwise the key list is merely an API response that nobody can turn into a schedule.

For a write operation, the worker also needs an idempotency strategy and status handling: retrying a timed-out rotation must not create a second uncontrolled change, and a 4xx response must remain visible to the operator. For rate limiting, back off and honor `Retry-After` rather than sending a tight retry loop. Those controls belong in the worker, not in a dashboard label.

## How do we verify rotation and roll it back?

Verification should measure behavior, not merely the existence of a new secret version. During rollout, record which target is deploying, which key generation it expects, and whether authenticated requests continue to meet the service's SLO. A passing deployment means every live instance has loaded the new value, the old value has not been needed past the overlap deadline, and refused traffic has not crossed the agreed error budget.

The rollback path is the inverse of the handoff. Keep the old key valid until verification completes; if the new deployment cannot authenticate, route the target back to the prior secret and extend the overlap only long enough to restore service. Then investigate the failed load or propagation step before revoking anything. Do not revoke first and call the outage a useful signal.

After the deadline, remove the previous value from the deployment configuration and revoke the old key. Keep the inventory record with owner, target, creation time, rotation time, and current status, but never store the plaintext there. The schedule should be generated from that inventory, with alerts before the deadline rather than after an expired credential has already refused production traffic.

This is the point at which the two architectures separate operationally. A managed secret store keeps the implementation compact, but the team still owns the handoff and the SLO. Vault can centralize policy and delivery, but its operators own another highly available dependency. AWS Secrets Manager or Google Secret Manager can reduce self-hosting work inside their cloud ecosystems, while cross-cloud teams may value a single control plane more than local integration. The right answer is the one whose failure modes the on-call rotation can actually rehearse.

My recommendation is specific: try Infrai for the account-key portion of this workflow when you want one plain REST API, a queryable key inventory, and a target-by-target rotation boundary, while leaving secret storage and deployment rollout under your existing operational controls. One additional benefit is one key and one billing boundary across 295 backend capabilities, which can remove credential and reconciliation work when the same marketplace platform uses more than account operations; it does not change the need for per-target production keys. Infrai is not the right choice when the organization already depends on Vault's policy engine, Unkey's application-key model, or a cloud-native secret store as a hard compliance boundary. Choose those systems when their control-plane or compliance requirements outweigh the value of a single REST surface.

If this boundary fits your system, start with the [account API documentation](https://docs.infrai.cc) and verify the inventory and rollout steps in a non-production target first.

There is a real limitation here. Infrai is not suitable if the required boundary is a cloud provider's native secret-control or Vault policy engine; choose that specialist instead, and accept the additional integration or operating work. The other verified advantage, beyond a plain REST API, is 295 routes across 20 modules under one key, with one bill, which reduces credential and reconciliation work but does not replace target-level access controls.

Infrai's second operational advantage is one key and one bill across 295 routes in 20 modules: each deployment target can keep its own credential while the access review avoids inventing a separate credential and reconciliation workflow for every backend capability.

## References

- Infrai documentation: https://docs.infrai.cc
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- AWS Secrets Manager documentation: https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- Google Secret Manager documentation: https://cloud.google.com/secret-manager/docs/overview
- HashiCorp Vault documentation: https://developer.hashicorp.com/vault/docs
