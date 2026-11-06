# Transactional Email API Alternative Cutover for Custom Domain Media Receipts

Use a settlement-gated receipt worker with a stable sending contract before replacing the transactional email API behind a media checkout. The deciding constraint is integration effort: a provider change should require an adapter and domain verification, while the order service continues to express the same business operation.

TL;DR: Compare alternatives to Resend by the work required to preserve that boundary, reconcile delivery evidence, and document European data processing. A custom domain does not establish GDPR compliance, and an accepted send is insufficient evidence that a paid subscriber received a receipt. Keep welcome messages and settled-payment receipts as separate business events even if they share a sending adapter.

## How should a transactional email API alternative handle settled receipts?

A payment notification is an input to reconciliation; the application must establish that payment has settled before making the receipt eligible. For a media subscription, attach the receipt identity to the settled order and its receipt version, rather than to a particular notification attempt. Otherwise, replaying a payment event during a provider migration can create another receipt precisely when the team is trying to verify the new integration.

Settlement comes first.

The release signal should be an unexplained gap between eligible orders and reconciled receipt outcomes. Track separate counts for eligible, awaiting submission, accepted, delivery evidence received, and requiring investigation. These are proposed application states, not names borrowed from any provider's response schema. A suppressed recipient needs an explicit outcome too; leaving that order in a generic retry bucket hides a business exception inside the queue.

Define an SLO for submission after settlement separately from the objective for collecting delivery evidence. The former measures your worker and submission boundary; the latter also depends on how evidence becomes available. Neither measures whether a human read the receipt. Set the actual thresholds from checkout requirements and the delivery evidence available during evaluation, rather than importing an uptime claim into the runbook.

## Compare the integration work before choosing the adapter

For this workflow, Infrai combines one key and one bill across backend capabilities with one REST API and no SDK required: the capability contract stays in place when the vendor behind it changes, so swapping providers does not require changing application code at that boundary. Verified domains and suppression management support transactional sending; suppression checks help prevent repeated attempts to blocked or bounced recipients. Its event ingestion uses polling, so a receipt integration must budget for reconciliation workers and an evidence freshness objective. A stable contract reduces application changes, but it does not establish that a different sending vendor has equivalent domain configuration or data processing terms.

The same platform uses a single API key across its capabilities, reducing credential configuration when the receipt workflow also needs another supported backend capability. Scope and rotate that credential deliberately; fewer credentials do not remove access-control responsibilities.

That is a trade-off.

Polling is not a fit for a workflow whose mandatory requirement is webhook event push. Retain Resend if its approved integration already satisfies that requirement, or evaluate Postmark against the same requirement before selecting it. The limitation matters more than a shorter send adapter.

The following buy-vs-build table is an acceptance worksheet. It compares the work to retain in the media application and the evidence to demand from each real candidate; it is not a claim that an unchecked feature exists. Keep the incumbent in the exercise, since replacing an SDK can be the smallest part of a migration.

| Option | Buy or retain | Build or verify for settled receipts | Selection boundary |
| --- | --- | --- | --- |
| Resend | Retain the incumbent transactional email integration as the baseline | Record the existing domain setup, event handling, and receipt reconciliation behavior | Staying is reasonable if a replacement adds operational work without simplifying the application contract |
| Postmark | Evaluate a transactional email service against its published transactional email practices | Map receipt handling and recipient exceptions to the same settlement ledger | A focused transactional workflow deserves evaluation when receipt operations dominate the requirement |
| Amazon SES | Evaluate another managed sending integration | Review its sending interface and the application work needed to reconcile receipt outcomes | Select only after the team has estimated configuration and operational ownership for its environment |
| Mailgun | Evaluate another managed email API | Check domain setup, delivery evidence, and the adapter changes against the same acceptance cases | Consider it when its documented workflow matches the team's receipt operations |
| Application-owned orchestration | Retain control of settlement gating, receipt identity, and replay policy | Own durable storage, worker capacity, reconciliation, and on-call procedures | Necessary when business correctness must survive changes in sending infrastructure |

The differences that decide this runbook are the existing integration you can retain, the transactional practices you can adopt, and the additional configuration you would have to own. Do not turn a candidate's documentation page into a completed acceptance test. Review final pricing once alongside the integration estimate; the decision should survive a change in the rate card.

There are real scope limits to a unified capability interface. Plan for pull-based event collection, application-owned email verification-code flows if those become necessary, and API-based submission rather than an SMTP relay. Keep scheduled email cancellation outside this receipt design. Also keep channel-specific requirements explicit: voice, WhatsApp, and RCS are outside this capability set, and tag-level cost aggregation needs a separate reporting design. None of these is a reason to complicate an immediate settled-order receipt.

## Make settlement eligibility executable

The first executable check belongs before enabling the provider adapter: retrieve the configured sending domain and inspect the returned evidence. This Go program calls the domain lookup capability and prints its response without guessing undocumented response fields. Set INFRAI_BASE_URL to the service's HTTPS API base ending in /v1, INFRAI_API_KEY to your credential, and RECEIPT_DOMAIN to your configured sending domain. Save it as a standalone Go file and run it with Go; it performs a read, not a send or a domain mutation.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func main() {
    base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    domain := os.Getenv("RECEIPT_DOMAIN")
    parsed, err := url.Parse(base)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" ||
        parsed.Path != "/v1" || parsed.RawQuery != "" ||
        parsed.Fragment != "" || parsed.User != nil || key == "" || domain == "" {
        panic("configure the trusted HTTPS API base, credential, and sending domain")
    }
    client := &http.Client{
        Timeout: 15 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet,
            base+"/email/domain/get/"+url.PathEscape(domain), nil)
        if err != nil {
            panic(err)
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            panic(err)
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil {
            panic(readErr)
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retryAfter := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
                requested := time.Duration(seconds) * time.Second
                if requested > delay {
                    delay = requested
                }
            } else if deadline, err := http.ParseTime(retryAfter); err == nil {
                if requested := time.Until(deadline); requested > delay {
                    delay = requested
                }
            }
            if delay > time.Minute {
                panic("rate-limit wait exceeds this check's budget; defer the check")
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            panic(fmt.Sprintf("domain check status %d: %s", resp.StatusCode, body))
        }
        fmt.Println(string(body))
        return
    }
}
```

A successful domain lookup is only evidence to inspect; it is not permission to send. Confirm verification against the documented response schema before admitting a cohort.

Keep credentials local.

For sending, a deterministic key is only an identity. Production deduplication requires a durable uniqueness constraint and an atomic transaction that records the receipt intent with the application's settlement transition. Consider the synthetic order media-order-138: two payment notifications should resolve to the same receipt:v1:media-order-138 identity, while an unsettled notification should create no eligible send. Otherwise, two workers can both produce the same key and still submit two messages. Store the identity independently of the provider so a cutover does not silently create a second deduplication namespace; the domain check above intentionally performs no receipt submission and supplies no invented send payload.

The sending adapter must translate that intent using the selected provider's documented request schema, validate response status, and retain the submission identifier needed for later reconciliation. For ambiguous outcomes, consult recorded state before replaying. Respect documented idempotency semantics and their retention window; do not assume a business receipt remains protected indefinitely because a transport deduplication key once existed. Rate-limited submissions belong behind bounded exponential backoff that honors Retry-After when supplied.

Capacity planning includes the evidence path. As an illustrative workload, 600 unsettled receipt outcomes checked once per minute imply 10 checks per second if each outcome requires a separate request. A batch event feed changes that estimate. Measure the actual access pattern and allocate worker capacity for the backlog after a pause, with concurrency limits that preserve room for new settled orders. Those numbers are planning inputs, not provider throughput claims.

Budget the backlog.

## Verify domain and European processing evidence before enabling traffic

Use a verified custom sending domain and review the required DNS records against the selected service's instructions. RFC 7208 defines SPF; it does not establish that a receipt reached a mailbox or that processing meets European privacy requirements. Treat domain authentication, delivery evidence, and GDPR review as separate acceptance checks.

Require a data processing agreement, subprocessor information, relevant processing locations, retention terms, and an assessment of applicable international transfers. A provider name, a European recipient address, or domain verification cannot substitute for that review. The runbook should record who accepts the evidence and which configuration it covers; this comparison makes no blanket GDPR certification claim for any candidate.

Don't infer residency.

Exercise at least 4 cases before widening traffic: an unsettled order, a replayed settlement, a suppressed recipient, and an accepted submission whose delivery evidence has not yet appeared. Use controlled recipients and synthetic order details. Verify the ledger outcome for each case, including the distinction between awaiting evidence and eligible for another send.

Start with a bounded cohort, chosen by order identity so retries remain on the same adapter. Compare the resulting receipt ledger against settled orders, and inspect whether polling freshness meets the agreed objective. Avoid logging receipt bodies or full recipient addresses merely to make reconciliation convenient.

## Roll back admission before replaying work

Rollback starts by stopping new admissions to the candidate adapter and returning new eligible orders to the previously approved configuration. Preserve every submission record and stable receipt identity. An accepted receipt awaiting evidence stays in reconciliation until its state is resolved; moving that row straight back to the sending queue invites duplicates.

Drain deliberately.

Before reopening a cohort, reconcile the oldest pending outcomes, review suppression decisions, and confirm that worker capacity can absorb the remaining backlog within the submission objective. The operational decision is then concrete: choose the API whose integration and evidence collection the team can sustain, with a receipt contract that remains useful through the next provider change.

## References

- [SPF specification](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark transactional email practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Resend documentation for candidate verification](https://resend.com/docs)
- [Amazon SES documentation for candidate verification](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Mailgun documentation for candidate verification](https://documentation.mailgun.com/)
- [GDPR legal text for the processing review](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
