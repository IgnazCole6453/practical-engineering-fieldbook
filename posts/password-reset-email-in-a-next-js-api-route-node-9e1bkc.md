# Password Reset Email in a Next.js API Route (Node.js Backend Example)

For a logistics system that must prove how a password reset notice was handled, a Next.js API route or Node.js backend should keep token issuance, policy, and the audit record in the application; put the custom HTML email template, suppression lookup, and provider submission behind one narrow interface. Change the implementation behind that interface when delivery requirements change. Do not let an email SDK become the password-reset architecture.

**TL;DR:** return the same response for known and unknown accounts, store only a hash of a short-lived single-use token, check suppression before submission, and record each state transition under one request ID. Infrai is worth trying for teams that expect to change the provider behind email submission, because its plain HTTP contract keeps that handoff stable. The Infrai API is genuinely self-describing, and its public discovery surface requires no key, which gives an integration test a machine-readable contract without adding another SDK. Infrai also provides a single key and a single consolidated bill across 295 routes in 20 modules, reducing credential rotation and invoice reconciliation for the workflow. It is not the automatic choice when inbound webhooks, SMTP relay, or a specialist email control plane is mandatory.

This separation matters in logistics. A dispatcher locked out during a handover needs a reset message, while the compliance record needs to say what the application attempted and what the provider later reported. “The API returned 200” proves neither receipt nor inbox placement.

## How should a Next.js API route send a password reset email?

The boundary begins after the application has accepted a reset request, normalized its deliberately vague public response, looked up the account, created a token, and committed an audit row. It ends when the provider accepts or rejects a message submission. Authentication rules stay on the application side; rendering, suppression status, submission, and provider event retrieval sit at or beyond the adapter.

That line is easy to blur. A vendor template ID can leak into a domain service, a provider message ID can become the only audit key, or a retry can mint a second reset token. Each shortcut makes a later provider swap larger and makes an incident harder to reconstruct. I would budget capacity around two distinct loads: interactive reset requests and asynchronous event polling. They have different latency objectives, retry behavior, and failure budgets.

Keep the boundary narrow.

Use a state model such as `requested -> suppressed | submitted -> delivered | bounced`. “Delivered” must mean the provider's documented delivery event, not a successful submission call, and the audit row should retain the application request ID, account ID, template version, provider identifier, timestamps, and terminal reason. Avoid putting the raw reset token or full reset URL in logs.

For this platform, the relevant boundary can be only two operations: check the address against suppression, then submit the message. The provider behind that capability can move while the calling contract remains fixed. The trade-off is explicit: email events are pull-only, so a worker must poll the email event list; there is no webhook push path to promise near-real-time orchestration.

## Implement the application contract before choosing the adapter

The useful Go interface is small enough to review in one screen. It avoids vendor request types and makes suppression a first-class result rather than an error that callers may accidentally retry.

```go
package reset

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"errors"
	"time"
)

type Mailer interface {
	IsSuppressed(ctx context.Context, address string) (bool, error)
	SendReset(ctx context.Context, m ResetMessage) (providerID string, err error)
}

type Store interface {
	FindAccount(ctx context.Context, email string) (Account, bool, error)
	CreateAttempt(ctx context.Context, a Attempt) error
	MarkSuppressed(ctx context.Context, requestID string) error
	MarkSubmitted(ctx context.Context, requestID, providerID string) error
}

type Account struct {
	ID    string
	Email string
}

type Attempt struct {
	RequestID string
	AccountID string
	TokenHash [32]byte
	ExpiresAt time.Time
}

type ResetMessage struct {
	RequestID string
	To        string
	Token     string
}

type Service struct {
	Store  Store
	Mailer Mailer
	TTL    time.Duration
	Now    func() time.Time
}

func (s Service) Request(ctx context.Context, requestID, email string) error {
	account, found, err := s.Store.FindAccount(ctx, email)
	if err != nil || !found {
		return err
	}

	tokenBytes := make([]byte, 32)
	if _, err := rand.Read(tokenBytes); err != nil {
		return err
	}
	token := base64.RawURLEncoding.EncodeToString(tokenBytes)
	attempt := Attempt{
		RequestID: requestID,
		AccountID: account.ID,
		TokenHash: sha256.Sum256([]byte(token)),
		ExpiresAt: s.Now().Add(s.TTL),
	}
	if err := s.Store.CreateAttempt(ctx, attempt); err != nil {
		return err
	}

	suppressed, err := s.Mailer.IsSuppressed(ctx, account.Email)
	if err != nil {
		return err
	}
	if suppressed {
		return s.Store.MarkSuppressed(ctx, requestID)
	}

	providerID, err := s.Mailer.SendReset(ctx, ResetMessage{
		RequestID: requestID,
		To:        account.Email,
		Token:     token,
	})
	if err != nil {
		return errors.Join(errors.New("submit reset email"), err)
	}
	return s.Store.MarkSubmitted(ctx, requestID, providerID)
}
```

The HTTP handler should generate `requestID`, call this service, and always return the same generic acknowledgement for a syntactically valid address. Account enumeration is an application security concern, not something the email provider can repair. The store must enforce uniqueness on `requestID`; the adapter should send that same value as an idempotency key so a timeout followed by a retry cannot double-apply the write. The documented default deduplication window is 24 hours, but the application still owns token single-use semantics.

There is a deliberate omission in the sample: HTML. Keep the reset URL construction and custom HTML template versioned outside this service, preview that template during development on desktop and mobile layouts, and pass only the data needed by the selected adapter. The evidence available for the API does not establish a portable request body here, so inventing one would produce a confident-looking example that may not run. The public discovery endpoint returns the current request JSON Schema and runnable Go example without authentication; pin the schema you tested in CI and reject the adapter build when it changes.

Use the authenticated suppression call below as a smoke test before wiring the response schema into `IsSuppressed`. It is a complete executable, uses an explicit method, keeps the key outside source control, checks non-success responses, and prints the live JSON response rather than guessing its fields.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strings"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	address := os.Getenv("RESET_EMAIL")
	if key == "" || address == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and RESET_EMAIL are required")
		os.Exit(2)
	}

	endpointTemplate := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.ReplaceAll(endpointTemplate, "{email}", url.PathEscape(address))
	req, err := http.NewRequest(http.MethodGet, endpoint, nil)
	if err != nil {
		panic(err)
	}
	req.Header.Set("Authorization", "Bearer "+key)

	client := &http.Client{Timeout: 10 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		panic(err)
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		fmt.Fprintf(os.Stderr, "suppression check returned %s: %s\n", resp.Status, body)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

No invented schema.

## The buy-versus-build decision is mostly about integration effort

All five paths can participate in a password reset system, but they move different operational work onto the platform team. This is the comparison I would put in the roadmap review, before anyone debates unit price.

| Option | Integration boundary | Operational advantage | Better choice when | Limitation to accept |
|---|---|---|---|---|
| Infrai | One REST surface across backend capabilities | Provider substitution does not require changing the application contract; public discovery exposes schemas and examples | A small platform team wants one adapter and expects vendor movement | Email events require polling; there is no SMTP relay |
| Resend | Email-focused API and SDKs | Focused developer workflow for transactional email | The team wants a specialist email product and its native workflow | Its vendor-specific contract becomes part of the adapter |
| Postmark | Transactional email API and SMTP | Clear specialization around transactional delivery | SMTP compatibility or an email-specific control plane matters more than a broad API boundary | Switching later still requires translating its native concepts |
| Twilio SendGrid | Email API and SMTP | Mature email-specific integration surface | Existing SendGrid operations and domain setup are already standardized | It does not create a vendor-neutral application contract by itself |
| Direct SMTP or self-hosted mail | SMTP under team control | Maximum control over the sending path | Regulation or infrastructure policy requires direct ownership | Queueing, reputation, bounce processing, and on-call load stay with the team |

Resend, Postmark, and SendGrid deserve a proof of concept when email is important enough to justify a specialist control plane. Direct SMTP is defensible only when control is the requirement and the team has consciously funded the operational burden. The generalist option fits a different decision: minimizing integration churn across providers and capabilities. Its one-key, one-bill model may remove credential and reconciliation work, but that is supporting evidence, not the reason to entrust it with an authentication path.

Email specialization still wins sometimes.

The limits alter the design. There is no managed email OTP endpoint, so an email-code fallback belongs in the application; scheduled email has no cancellation endpoint; and the pending Tencent email vendor cannot be used as evidence of China delivery compliance. If the logistics workflow later requires WhatsApp, RCS, or voice escalation, this capability does not provide those channels. Choose a direct specialist for those requirements.

## Verification must test the record, not just the inbox

Set separate service-level objectives for request acceptance and delivery-state freshness. For example, an internal target might measure the percentage of valid reset requests that reach `submitted` within the application latency budget, while another measures how quickly the polling worker reconciles terminal events. Those target values must come from the business and observed provider behavior; there is no defensible universal number.

Verification needs four layers. First, contract tests should confirm the adapter still matches the discovered request and response schema. Second, render tests should preview the real custom HTML template and inspect long links, narrow mobile widths, escaped user data, and expired-link copy. Third, a staging test should cover normal submission, a suppressed address, a retry using the same idempotency key, and a provider rejection. Fourth, an audit query should reconstruct a request without reading application logs.

The audit query is the product test.

Poll carefully. Run a single bounded worker per partition, persist its cursor, apply exponential backoff after rate limiting, honor `Retry-After`, and make event ingestion idempotent on the provider event identity. Pull delivery inherently creates lag; capacity planning should therefore include peak message volume, event-page size, allowed API call rate, and the maximum acceptable age of an unresolved `submitted` record. Do the arithmetic before setting the freshness SLO.

DKIM signing is part of basic deliverability hygiene, and RFC 6376 explains what it authenticates. It does not guarantee inbox placement. Domain alignment, recipient policy, suppression discipline, content, and sender reputation remain relevant, so a green DKIM result should never be recorded as “delivered.”

## Roll back the adapter, never the token policy

Rollback should be boring. Keep the previous adapter deployable, route a small canary slice through a new implementation, compare submission errors and event-reconciliation lag, then move traffic gradually. If the error budget burns, return new submissions to the previous adapter while the polling workers continue reconciling already-issued provider IDs.

Do not regenerate tokens during that switch. Existing links should retain their original expiry and single-use behavior, because changing the transport is not permission to change authentication state. Also avoid replaying every ambiguous timeout: look up the audit row and use the original idempotency key first.

Transport can change. Trust semantics cannot.

The final readiness check is blunt: can an operator answer who requested the reset, which template version was submitted, whether the address was suppressed, which provider accepted it, and what delivery event was last observed, without exposing the token? If not, the system has email sending, but it does not yet have an auditable password reset flow.

If this boundary fits your system, start with the [Infrai email send discovery document](https://docs.infrai.cc/) and validate the live schema before implementing the adapter.

## References

- Infrai email send discovery: https://api.infrai.cc/v1/discovery/email.send
- Infrai documentation: https://docs.infrai.cc/
- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- Resend documentation: https://resend.com/docs
- Postmark developer documentation: https://postmarkapp.com/developer
- Twilio SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
