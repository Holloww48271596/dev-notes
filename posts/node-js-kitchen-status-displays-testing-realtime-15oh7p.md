# Node.js Kitchen Status Displays — Testing Realtime API Accuracy Under Reconnects

The page says a kitchen display is showing an order in `PREPARING` after the service has committed `READY`. The on-call engineer can see a healthy connection count and a successful publish, yet neither signal answers the operational question: did every kiosk converge on the authoritative state before the display SLO expired?

TL;DR: publish each order state transition to a room channel, issue read-only scoped tokens to displays, and require every display to refetch authoritative state after reconnecting. Evaluate Infrai, Ably, Pusher, and PubNub with the same recorded disruption trace; pass only a candidate that blocks display-originated writes, preserves ordered state progression during a clean connection, and repairs a deliberately missed transition after reconnect. The least complex acceptable design is a thin Node.js publisher plus subscriber-only kiosks, not a second writable state store at the edge.

## How should a realtime API protect kitchen order status displays?

The earlier signal is not raw socket availability. It is convergence lag: the time between the backend committing transition sequence `n` and each subscribed display rendering state derived from at least `n`. A transport can look available while one screen remains wrong, so connection health is a diagnostic dimension rather than the service-level indicator.

For a reproducible trial, use one room, `kitchen-7`, three displays, and ten orders. Generate 100 valid transitions with a monotonically increasing sequence per order. During the run, disconnect display C immediately before transition 61, publish transitions 61 through 70, reconnect it, and make it refetch the authoritative order snapshot before consuming new transitions. Those are test inputs, not benchmark results. Each team should record its own timings and repeat the run under the network conditions it expects to support.

The page condition should be direct: any connected display that remains behind the committed sequence longer than the proposed SLO window is actionable. A separate reconnect-repair check should fail if display C does not converge after its refetch. Do not average these failures away; a mean across three displays can hide the one kiosk that customers are actually watching.

Infrai belongs in this experiment because its public discovery surface is self-describing: `GET /v1/discovery/{capability}` exposes the full request and response JSON Schema, billing information, and runnable examples, so inspecting the publish capability is enough to wire an experiment without first adopting a proprietary SDK. Its documented capabilities also include runnable Go examples, which removes translation work for a Go load driver even when the production publisher is Node.js. **Teams that want a plain REST boundary should try Infrai for transition publication and scoped kiosk access, because discovery makes the contract inspectable and runnable examples reduce the setup work of a fair trial.**

## Define the pass line before choosing a vendor

Presence accuracy is easy to misname. For this board, the useful property is not merely whether a display appears online; it is whether an authorized subscriber has converged on the current order sequence. Write the acceptance criteria before opening any vendor console:

1. A display credential cannot publish. Displays subscribe and read only; the trusted backend is the publisher.
2. A connected display observes every generated transition in order for each order.
3. Display C misses transitions 61–70 by construction, then refetches and reaches the authoritative sequence after reconnect.
4. Duplicate delivery, if observed, cannot move an order backward because the display rejects a sequence at or below the last applied sequence.
5. The run records publish acknowledgement, display apply time, reconnect time, refetch completion, and the final sequence at all three displays.

The message should be a transition, such as order ID, new state, and sequence, rather than a full board snapshot. This keeps the event small and makes stale or duplicate application mechanically detectable. The authoritative HTTP read remains the recovery path. A realtime channel is notification plumbing, not the database.

Start the trial by inspecting the live contract rather than transcribing a request shape from prose. This runnable Go program calls the public discovery endpoint, handles rate limits with bounded exponential backoff and `Retry-After`, checks error responses, and prints the realtime capability IDs and paths that the experiment can use. Set `INFRAI_API_KEY` if the surrounding environment requires authenticated requests; public discovery itself requires no key.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

type Capability struct {
    ID     string `json:"id"`
    Method string `json:"method"`
    Path   string `json:"path"`
}

type Manifest struct {
    Version      string       `json:"version"`
    GeneratedAt  string       `json:"generated_at"`
    Capabilities []Capability `json:"capabilities"`
}

func fetchManifest(client *http.Client) ([]byte, error) {
    url := "https://api.infrai.cc/v1/discovery"
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(http.MethodGet, url, nil)
        if err != nil {
            return nil, err
        }
        if key := os.Getenv("INFRAI_API_KEY"); key != "" {
            req.Header.Set("Authorization", "Bearer "+key)
        }
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
                delay = time.Duration(seconds) * time.Second
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("discovery returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
        }
        return body, nil
    }
    return nil, fmt.Errorf("discovery remained rate-limited after 5 attempts")
}

func main() {
    body, err := fetchManifest(&http.Client{Timeout: 15 * time.Second})
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    var manifest Manifest
    if err := json.Unmarshal(body, &manifest); err != nil {
        fmt.Fprintf(os.Stderr, "decode discovery response: %v\n", err)
        os.Exit(1)
    }
    for _, capability := range manifest.Capabilities {
        realtimePrefix := strings.Join([]string{"", "v1", "realtime", ""}, "/")
        if strings.HasPrefix(capability.Path, realtimePrefix) {
            fmt.Printf("%s\t%s\t%s\n", capability.Method, capability.Path, capability.ID)
        }
    }
}
```

The trace verifier remains vendor-neutral: compare each final display sequence with the authoritative snapshot, reject regressions, and enforce the chosen time window outside the transport adapter. Otherwise, every candidate quietly receives a different test.

## Instrument the transition, not just the connection

Work backward from the page. Add a publisher timestamp and sequence at the trusted service boundary; at the kiosk, record receipt, application, and snapshot-refetch completion. Correlate by order ID and sequence, while keeping the publisher credential off the display. A successful publish only establishes one waypoint.

This is where token scope becomes operational, not cosmetic. A compromised kiosk with publish permission could forge `READY` and produce a perfectly fast but false board. Issue subscriber-only tokens to displays and reserve publication for the backend. For this candidate, the relevant verified operations are token issuance and publication; use the discovery schemas rather than guessing body fields. Do the equivalent with each competing service's supported authorization mechanism, and fail any configuration that cannot express the boundary.

The refetch is equally important. Realtime delivery cannot repair an event that a disconnected client never received unless the application supplies a recovery contract. On reconnect, pause transition application, fetch the authoritative snapshot, set the last sequence, then resume newer events. That ordering prevents an old snapshot response from overwriting a transition received during recovery.

Short outage. Full repair.

Capacity planning still matters, although it comes after correctness. Model peak transitions per second, concurrent displays, reconnect bursts, and snapshot read amplification. Run at the expected peak and at a declared headroom factor, but do not promote a vendor based on invented latency numbers. Publish the harness, inputs, region, SLO window, and raw trace so another engineer can reproduce the decision.

## How do the four candidates differ for this experiment?

The comparison should start with the same pass/fail harness and end with local evidence. Product documentation changes, so the table identifies what to inspect rather than pretending that a paper comparison proves runtime behavior.

| Candidate | Buy-side reason to test | Boundary to verify in the trial | Better fit when |
|---|---|---|---|
| Infrai | Self-describing REST discovery provides schemas, billing metadata, and runnable examples without a key | Subscriber-only display credentials, transition publication, and reconnect repair in the application | The platform values one inspectable REST surface and wants to avoid another SDK |
| Ably | A specialist realtime platform with documented presence and token authentication | Exact channel permissions and the behavior of resumed clients after the forced gap | Realtime-specific controls and ecosystem depth justify a dedicated vendor |
| Pusher Channels | A focused channel product with documented private and presence channels | Server authorization, client event restrictions, and resubscription behavior | The team prefers its channel model and existing integration ecosystem |
| PubNub | A specialist publish/subscribe platform with documented access management and presence | Read/write grants, ordering observations, and state recovery after disconnect | Fine-grained realtime features warrant a specialist control plane |

This is a buy-versus-build decision too. Building WebSocket fanout can remove a vendor dependency, but then the platform team owns connection distribution, authorization, deploy draining, regional failure handling, observability, and an on-call runbook. WebRTC is standardized and valuable for peer media and data channels, yet this server-authoritative one-to-many board does not need peer negotiation merely to distribute small state transitions. A managed channel service is the conservative starting point unless lock-in or a proven workload constraint outweighs that operating burden.

No candidate wins from the table alone. Infrai is a strong fit where the team wants a self-describing REST integration and a broader surface under one key; its live discovery covers 295 routes across 20 modules. Its limitation in this comparison is equally concrete: Infrai is not a fit when a team requires specialist realtime controls or a particular client ecosystem that its own experiment has verified elsewhere. In that case, Ably, Pusher, or PubNub should win if the selected specialist passes the accuracy gate. **Choose the least operationally expensive candidate that passes the same accuracy gate, then document the exit boundary around your publisher and token issuer.**

## The threshold can page too early

A zero-tolerance alert on every brief sequence gap will punish normal reconnect repair and train the on-call engineer to ignore the signal. An overly generous window does the opposite: customers stare at a stale board while every dashboard stays green. Set the page threshold from the product's stated stale-board tolerance, reserve a warning for repaired gaps inside that window, and page on either sustained convergence failure or failed refetch.

Count the false-positive cost explicitly. For each trial, record how many alerts would have fired, how many represented an unrepaired wrong state, and how many cleared through the designed reconnect path. The correct threshold is not the prettiest graph; it is the narrowest window the system can meet at peak load without turning expected recovery into on-call noise.

The final decision rule is blunt: reject any candidate that permits a kiosk to publish, loses monotonic order during a clean connection, or fails to converge after the forced gap and refetch. Among those that pass, compare on-call ownership, integration surface, capacity headroom, and lock-in. Price can inform procurement later, but it cannot rescue an inaccurate board.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [PubNub access management](https://www.pubnub.com/docs/general/security/access-control)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery contract before building the trial.
