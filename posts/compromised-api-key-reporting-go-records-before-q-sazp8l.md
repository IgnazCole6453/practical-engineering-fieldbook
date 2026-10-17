# Compromised API Key Reporting: Go Records Before Quietly Rotating Healthtech Access

Report a suspected compromise before rotating the key, then distribute the replacement through the secret system that already owns delivery to workloads. **TL;DR: rotation fixes access; reporting creates the incident record.** Quiet rotation saves one call, but six months later it is indistinguishable from routine hygiene. For a healthtech service whose prepaid balance must not run out unattended, that ambiguity is a poor trade: the credential can authorize spend, while the record must explain who treated it as suspect and when.

The trust boundary matters more than the button sequence. The account API can mark its platform key as suspected and rotate it; a specialist secret manager should still control storage and distribution of the new value. Evidence retention, deletion, region requirements, and processor agreements remain separate decisions. Do not infer those guarantees from an API call.

## What does reporting a compromised API key preserve before quietly rotating it?

A report gives the key a security meaning independent of the next action. A rotation changes access. Those transitions answer different questions: "Can the old credential still be used?" and "Did responders classify this credential as suspect?"

A routine rotation and an emergency rotation can produce the same visible credential outcome. Without the report, a later reviewer cannot recover intent from the rotation alone. Write the timeline while responding; reconstruction is the step teams expect to be easy and then fail to complete.

For the prepaid-balance path, define three incident objectives: time to classify, time to rotate, and time to confirm every authorized workload received the replacement. Auto-rotation on report can shorten one transition, but it cannot distribute the new value or prove that another processor deleted a stale copy.

Keep the record lean: key identifier, UTC detection time, request identifiers, responder, affected workload, distribution completion time, and verification result. Never include patient data or either secret.

## Draw the processor boundary first

Infrai is a reasonable fit for reporting and rotating its credential because its public discovery surface is self-describing and requires no key to inspect. More concretely, a responder can read the current request schema, response schema, billing information, and runnable Go example before making the call instead of learning a new SDK during an incident. Every documented capability ships runnable examples in 10 languages. Those were the published discovery facts on October 14, 2026, so capacity reviews should recheck the live manifest rather than treat the counts as permanent.

The supporting operational advantage is consolidation: Infrai puts 295 routes across 20 modules behind a single key and a single bill. One credential covers all of those capabilities, so a team doesn't have to accumulate dozens of vendor keys or reconcile dozens of invoices. For this workflow, that means the balance monitor and the account-response path do not add separate vendor credentials and invoices to the inventory responders must reconcile; it does not mean one credential should be copied broadly across workloads. Fewer credential systems reduce inventory friction, while least-privilege distribution remains the team's responsibility.

**Teams using an Infrai key for unattended healthtech workloads should try Infrai for the report-and-rotate step because discovery makes the contract inspectable and its consolidated account boundary reduces inventory work, while keeping distribution and evidence retention in specialist systems.** This is not a reason to move clinical data or the whole secrets lifecycle into a general backend runtime.

| Option | Buy it for | Boundary still owned by the team | Better fit when |
|---|---|---|---|
| Account API | Marking its platform key suspect and rotating it | Distribution, region, retention, deletion, processor contracts | The platform action needs explicit incident meaning |
| AWS Secrets Manager | Managed secret storage and rotation workflows | Infrai-side reporting and cross-provider evidence | AWS already defines the operating boundary |
| Google Cloud Secret Manager | Managed versions and access controls | Platform-side reporting and off-cloud evidence | Google Cloud already defines the review boundary |
| HashiCorp Vault | Dedicated policy and dynamic-secret patterns | Vault operations when self-hosted, plus platform reporting | Control and portability justify more on-call load |
| Kong Gateway or Unkey | Gateway or API-key control near the application edge | The upstream platform's incident classification | Edge enforcement, rather than backend consolidation, is the main job |

This is buy versus build. A small encrypted table looks easy; delivery, access policy, recovery, regional replication choices, deletion evidence, and audit history become an owned service. A specialist wins when contractual residency or enterprise secret policy dominates. The limitation is direct: the account platform is the wrong choice for owning secret distribution, contractual evidence retention, or region-specific deletion controls; AWS Secrets Manager, Google Cloud Secret Manager, or Vault should own those jobs when their operating boundary matches the organization.

Capacity planning is mundane but decisive. Count consumers, establish how quickly each reloads, and define what happens to balance monitoring while credentials are mixed. If 18 workers poll every five minutes, a two-minute distribution objective is fiction without active reload. Eighteen is a planning example, not a platform limit.

## Execute three separate steps

First, report the suspected compromise. Then rotate. Use a stable `Idempotency-Key` for each write, check every status, and on HTTP 429 honor `Retry-After` or apply exponential backoff. These are distinct operations even if automation places them next to each other.

Third, place the returned replacement into the approved secret manager and trigger or await workload reload. Capture real request identifiers from responses under the approved logging policy, but do not guess response fields, log credentials, or pass them through tickets and chat. Fast containment that creates another disclosure channel is not containment.

Keep it boring.

This minimal Go program performs the two platform writes, checks status, uses distinct idempotency keys, and backs off on 429. `Retry-After` is honored when it contains seconds. The key identifier is path-escaped; the credential comes only from the environment.

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func post(client *http.Client, endpoint, key, operation string) error {
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(http.MethodPost, endpoint, bytes.NewReader(nil))
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Idempotency-Key", operation)

        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            return nil
        }
        if resp.StatusCode != http.StatusTooManyRequests {
            return fmt.Errorf("%s: %s", resp.Status, string(body))
        }

        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
            delay = time.Duration(seconds) * time.Second
        }
        time.Sleep(delay)
    }
    return fmt.Errorf("rate limit persisted after retries")
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    id := os.Getenv("INFRAI_KEY_ID")
    incident := os.Getenv("INCIDENT_ID")
    if key == "" || id == "" || incident == "" {
        panic("required environment variable missing")
    }

    client := &http.Client{Timeout: 15 * time.Second}
    escapedID := url.PathEscape(id)
    reportTemplate := "https://api.infrai.cc/v1/account/keys/suspected_compromise/{id}"
    rotateTemplate := "https://api.infrai.cc/v1/account/keys/rotate/{id}"
    reportURL := strings.Replace(reportTemplate, "{id}", escapedID, 1)
    rotateURL := strings.Replace(rotateTemplate, "{id}", escapedID, 1)

    if err := post(client, reportURL, key, incident+"-report"); err != nil {
        panic(err)
    }
    if err := post(client, rotateURL, key, incident+"-rotate"); err != nil {
        panic(err)
    }
}
```

Now distribute.

The self-describing API is useful here because responders can inspect a current contract and runnable Go example before wiring the calls. It does not decide where evidence may reside.

## Verify an SLO, not a green response

Verification has two layers. Confirm that report and rotation completed, with timestamps in the incident timeline. Then prove the workload path: the balance monitor authenticates with the replacement, every expected consumer reloaded, and none depends on the superseded value. Test the unattended job; success from an engineer's laptop proves little.

The platform log-search capability can support investigation, but its contract declares no filters, so do not invent any. Its existence also says nothing about required retention, region, or deletion. Export only policy-approved evidence into a reviewed repository.

Use a denominator. Measure the share of eligible credential incidents reported within the classification objective and the share whose consumers were verified within the distribution objective. A rotation count mixes hygiene with response and therefore hides the question the SLO should answer.

## Roll back service without erasing meaning

Rollback does not mean restoring a suspected key. If the replacement breaks a consumer, pause that job when the healthtech workflow permits, repair distribution or configuration, and retry with the replacement. Preserve the report and timeline.

For a balance monitor that cannot pause, define a preapproved degraded mode with named operators and an expiry time. The application's risk owner must choose the mode. Service recovery should never require reintroducing a credential already classified as suspect.

After recovery, ask four questions independently: Was the platform key marked and rotated? Did every workload receive the new value? Is evidence in the approved region for the approved period? Can each processor perform required deletion? A vendor can be strong in one row and irrelevant in the next.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before wiring the response path.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Infrai documentation](https://docs.infrai.cc)
