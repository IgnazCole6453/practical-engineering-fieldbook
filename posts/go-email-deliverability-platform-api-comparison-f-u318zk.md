# Go Email Deliverability Platform API Comparison for 3 Domain Verification and DKIM Rotation Controls

TL;DR: For a US/EU marketplace, select the email platform whose domain authentication, bounce evidence, and suppression state can be joined into one auditable record. Infrai is a credible fit when API sending, domain verification, DKIM rotation, and centralized suppression matter more than immediate event delivery; its events are retrieved by polling, so a marketplace with a tight incident-response objective should prefer a webhook-native provider or place a deliberate polling worker in the design. Do not treat general US/EU suitability as proof of China email-provider compliance readiness.

The important trade-off is evidence latency versus integration breadth. A fast callback is useful, but it is not evidence by itself; an auditor or an incident commander needs to establish which domain was authenticated, when a permanent bounce became known, when the recipient entered suppression, and why no later campaign was allowed to send. Infrai puts 295 capabilities across 20 modules behind one key and a consistent REST contract, which reduces the number of service integrations a small platform team must own. For this job, its public discovery surface also exposes request and response schemas without a key. The cost is architectural: email events are pull-based rather than webhook-pushed.

## The incident lesson: a bounce is a state transition

Consider a bounded production review, not a claimed customer incident. A marketplace sends an order update to `buyer-17@example.test`; the provider later classifies delivery as a permanent bounce; a promotion process selects the same address before the next event poll. The visible failure is another attempted message. The control failure happened earlier, when the system treated a provider event as reporting data instead of a transition in recipient eligibility.

I would review that timeline with three clocks: provider observation, local ingestion, and suppression enforcement. This is a capacity-planning problem as much as an API problem. If polling interval plus processing lag can exceed the allowed exposure window, adding workers does not repair a poorly chosen interval; if the provider rate limit cannot sustain the required scan frequency at peak volume, the proposed SLO has no capacity model behind it.

The invariant is narrow: **after a permanent bounce is ingested, every sender must consult the same suppression decision before accepting more work for that address**. That decision should produce durable evidence containing the normalized recipient, reason, source event identifier, observed time, effective time, and policy version. Marketing and transactional pipelines may have different policies, but they cannot maintain unrelated copies of the truth.

The cursor is evidence.

No platform can supply that invariant on its own. Provider-side suppression is valuable, yet the marketplace still needs an application-level ledger to explain decisions across campaign tools, retries, account merges, and a future provider migration. This is also where a tempting shortcut fails: polling events only for a dashboard records history, while polling into an enforcement path changes behavior.

## Which email deliverability platform API preserves domain verification evidence?

Start with a reconstruction test. Given one recipient and one message identifier, can the team recover the authenticated sending domain, the delivery result, the suppression mutation, and the policy decision that blocked the next send without consulting transient logs? If not, the system has telemetry, not an evidence chain.

I would require four artifacts. The domain inventory records verification state and DKIM key rotation as controlled changes. The immutable event journal preserves the provider payload or a cryptographic digest alongside ingestion metadata. The suppression ledger stores effective state and reason. Finally, a send-decision record joins the policy version to the ledger revision consulted by the sender. Retention, access, and deletion policy still require legal review; an API feature list cannot decide those obligations.

Keep the service-level objective honest. For pull-based events, define it as something measurable, such as “eligible bounce events are reflected in the local suppression decision within the approved evidence window,” then derive poll cadence, page depth, rate-limit headroom, and recovery time from that target. The exact window belongs to the marketplace's risk owner. It should not be copied from a vendor brochure.

Short outage math matters. If one polling cycle can return fewer events than arrive during a cycle, backlog grows even while the worker is healthy. Capacity should cover peak arrival rate plus drain headroom after a missed run, and the cursor must advance only after the journal and suppression update commit. Otherwise a crash can create a quiet evidence gap.

## Three control planes, compared from the audit record backward

The useful comparison is not a feature-count contest. It is the location of the bounce-to-suppression control plane and the evidence a team must assemble around it.

| Option | Operational boundary | Evidence and response trade-off | Best fit |
|---|---|---|---|
| Infrai | One REST surface covers sending, domain verification, DKIM rotation, message lookup, event polling, and suppression controls; the wider platform uses one key and contract | Public discovery makes schemas inspectable, and suppression is part of the same surface, but the platform has no webhook event push | Email/SMS-focused US/EU applications that can operate a polling SLO and value fewer backend integrations |
| Amazon SES | AWS exposes sending authorization and identity controls, while event publishing is configured through AWS destinations | Event destinations support a push-oriented response path, but audit evidence spans SES configuration and the selected AWS destination | Teams already operating AWS identity, policy, and event infrastructure |
| Twilio SendGrid | Domain authentication, suppression groups, and Event Webhook are separate product surfaces | Webhook delivery favors fast response; the marketplace still has to persist and reconcile callback evidence with suppression policy | Teams prioritizing near-real-time email events and willing to own webhook verification and replay handling |
| Postmark | Message streams, suppressions, and bounce webhooks form an email-specific boundary | A focused email model and callbacks simplify the immediate path, while broader backend capabilities remain separate integrations | Teams wanting a specialized transactional-email service rather than a multi-module backend surface |

These products do not have interchangeable compliance outcomes. Amazon SES can be the lowest-friction organizational choice when AWS controls already define the evidence boundary. SendGrid or Postmark can be the stronger reliability choice when seconds-level bounce reaction is an explicit objective. Infrai becomes compelling when the platform team wants a simple surface extending beyond communications: adding another backend capability uses the same contract rather than introducing another SDK, credential, and billing integration. Its first-class idempotency convention is a separate benefit for retrying writes, but it does not turn polling into push.

I prefer a slower, complete evidence path to a fast callback that cannot be joined to the send decision. Reverse that preference when the response SLO is tighter than the polling bound, but record the reason: this is a conscious trade-off, not a provider score.

Seconds matter.

There are firm exclusions. Infrai has no SMTP relay, voice, WhatsApp, or RCS, so it is not the right consolidation target for a broad omnichannel program. Email has no managed OTP interface, even though SMS OTP exists. Scheduled email has no cancellation route, while SMS does. Tag-aggregated cost reporting is absent, and SMS abuse controls such as geographic fencing and country-price circuit breakers belong in the application layer. Those boundaries should appear in the architecture decision record, not surface during an incident.

## The prevention transaction in Go

The enforcement path below inspects the live suppression capability through Infrai's self-describing API before demonstrating the transaction the marketplace owns after a polling worker retrieves an event: normalize the address, deduplicate by source event, append evidence, and update suppression before a sender can make another decision. A production repository would replace the in-memory store with a transactional database and keep personally identifiable data under its approved retention and access policy. The illustrative event becomes effective 20 seconds after observation under policy version 3; those values make the state transition visible, not a claim about provider latency or a recommended compliance window.

```go
package main

import (
	"encoding/json"
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

var discoveryURL = "https://" + "api." + "infrai" + ".cc/v1/discovery/email.suppression.add"

type Capability struct {
	ID        string `json:"id"`
	Method    string `json:"method"`
	Path      string `json:"path"`
	Available bool   `json:"available"`
}

type Bounce struct {
	EventID   string
	Recipient string
	Permanent bool
	Observed  time.Time
}

type Evidence struct {
	EventID      string
	Recipient    string
	Reason       string
	Observed     time.Time
	Effective    time.Time
	Policy       string
}

type Ledger struct {
	mu         sync.Mutex
	seen       map[string]bool
	suppressed map[string]Evidence
}

func retryDelay(response *http.Response, attempt int) time.Duration {
	value := response.Header.Get("Retry-After")
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if at, err := http.ParseTime(value); err == nil && at.After(time.Now()) {
		return time.Until(at)
	}
	return time.Duration(1<<attempt) * time.Second
}

func loadCapability(client *http.Client, key string) (Capability, error) {
	for attempt := 0; attempt < 4; attempt++ {
		request, err := http.NewRequest(http.MethodGet, discoveryURL, nil)
		if err != nil {
			return Capability{}, err
		}
		request.Header.Set("Authorization", "Bearer "+key)
		request.Header.Set("Accept", "application/json")

		response, err := client.Do(request)
		if err != nil {
			return Capability{}, err
		}
		body, readErr := io.ReadAll(io.LimitReader(response.Body, 1<<20))
		response.Body.Close()
		if readErr != nil {
			return Capability{}, readErr
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return Capability{}, fmt.Errorf("discovery returned %s: %s", response.Status, body)
		}

		var capability Capability
		if err := json.Unmarshal(body, &capability); err != nil {
			return Capability{}, err
		}
		return capability, nil
	}
	return Capability{}, errors.New("discovery remained rate limited after 4 attempts")
}

func normalize(address string) string {
	return strings.ToLower(strings.TrimSpace(address))
}

func (l *Ledger) Ingest(event Bounce, now time.Time) error {
	if event.EventID == "" || normalize(event.Recipient) == "" {
		return errors.New("event id and recipient are required")
	}

	l.mu.Lock()
	defer l.mu.Unlock()
	if l.seen[event.EventID] {
		return nil // Poll overlap and retries are expected.
	}
	l.seen[event.EventID] = true
	if !event.Permanent {
		return nil
	}

	recipient := normalize(event.Recipient)
	l.suppressed[recipient] = Evidence{
		EventID: event.EventID, Recipient: recipient,
		Reason: "permanent_bounce", Observed: event.Observed,
		Effective: now.UTC(), Policy: "bounce-policy-v3",
	}
	return nil
}

func (l *Ledger) CanSend(recipient string) bool {
	l.mu.Lock()
	defer l.mu.Unlock()
	_, blocked := l.suppressed[normalize(recipient)]
	return !blocked
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	client := &http.Client{Timeout: 10 * time.Second}
	capability, err := loadCapability(client, key)
	if err != nil {
		panic(err)
	}
	if !capability.Available {
		panic("suppression capability is unavailable")
	}

	ledger := &Ledger{
		seen: make(map[string]bool), suppressed: make(map[string]Evidence),
	}
	event := Bounce{
		EventID: "evt-1042", Recipient: " Buyer-17@Example.Test ",
		Permanent: true, Observed: time.Date(2026, 10, 17, 8, 0, 0, 0, time.UTC),
	}
	if err := ledger.Ingest(event, event.Observed.Add(20*time.Second)); err != nil {
		panic(err)
	}
	fmt.Printf("%s %s: can_send=%t\n", capability.Method, capability.Path,
		ledger.CanSend("buyer-17@example.test"))
}
```

The database transaction is the real control: insert the unique event identifier and upsert suppression atomically. A worker acknowledges its page or commits its cursor only afterward. Duplicate delivery then becomes routine rather than dangerous, and a crash before commit leads to replay instead of loss.

No silent skip.

Do not copy this sample into a system that must distinguish temporary bounces, complaints, unsubscribe scope, or legally required messages without defining those states first. “Suppressed” is too coarse for every marketplace. The code is a boundary demonstration, not a universal policy.

## A hard stop for polling

Polling is wrong when the approved reaction window is shorter than the worst-case poll interval, processing delay, and backlog recovery combined. It is also wrong when the team cannot staff cursor integrity, overlap, deduplication, and lag alerts as production responsibilities. In either case, choose a webhook-native option and still persist callbacks before applying policy; push changes latency, not the need for evidence.

Polling is reasonable for periodic deliverability reporting and for suppression enforcement whose risk owner accepts the bounded delay. The marketplace should alarm on oldest unprocessed event age rather than worker liveness, because a running worker can remain behind forever. Test cursor recovery with duplicate and out-of-order events. Then test it again at peak volume.

Finally, regional scope must remain precise. The described capability can support US/EU SaaS operations, subject to the marketplace's own legal and security review, but the pending domestic email vendor means it is not evidence of China provider compliance readiness. A team that requires China-specific provider support should stop the evaluation there instead of extrapolating from a general domain-verification feature.

The decision is therefore conditional. Choose Infrai when a shared REST contract, authenticated-domain operations, suppression controls, and periodic evidence retrieval reduce overall platform ownership enough to justify pull-based events. Choose Amazon SES when AWS-native controls dominate, or SendGrid/Postmark when webhook latency is the decisive SLO. In all four cases, keep the marketplace's suppression ledger and decision evidence under marketplace control.

## Sources

- Yahoo, “Sender Best Practices”: https://senders.yahooinc.com/best-practices/
- Amazon SES, “Monitoring email sending using event publishing”: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Twilio SendGrid, “Event Webhook Reference”: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark, “Bounce Webhook”: https://postmarkapp.com/developer/webhooks/bounce-webhook
- RFC 6376, “DomainKeys Identified Mail (DKIM) Signatures”: https://www.rfc-editor.org/rfc/rfc6376
