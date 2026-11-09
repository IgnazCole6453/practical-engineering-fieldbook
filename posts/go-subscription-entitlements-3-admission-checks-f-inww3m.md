# Go Subscription Entitlements: 3 Admission Checks for SaaS Plan Limits

Read subscription entitlements programmatically, then gate each property-management workload on a current decision and an atomic budget reservation before allowing it to create billable work. A plan tier alone is too weak an authorization record to explain who could spend, under which subscription terms, at the moment the work was admitted.

TL;DR: A Node.js SaaS backend should read plan tier and subscription entitlements programmatically through a server-side adapter, then pass a versioned entitlement decision to its spending gate. Keep the tier for explanation; enforce explicit capabilities, applicable quotas, and a workload budget. The three checks below are entitlement freshness, workload authorization, and atomic reservation. They are a proposed operating design, not claims about a subscription service's API.

For property management, consider a proposed tenant-notice workload that processes documents for several buildings. Its service identity needs a spending boundary that an auditor can reconstruct before the invoice arrives. OWASP's secrets-management guidance supports the credential side of that requirement through least privilege, lifecycle management, and auditing [1]. The budget protocol requires additional application design; credential hygiene does not supply it.

## How should a plan tier read enforce subscription entitlements?

A display label such as a subscription tier can tell an operator what an account purchased, but the gate needs a more precise answer: may this principal run this workload, for this account, under this entitlement revision? Define that answer explicitly. In the proposed design, the subscription adapter returns the account identity, tier, entitlement revision, allowed capability, quota period, and applicable account ceiling; a separate policy decision binds the authenticated principal to a permitted workload and its local ceiling. No client-supplied plan string participates in authorization.

That distinction matters during a plan change. Consider a hypothetical account moving to a smaller allowance while document jobs remain queued: a worker that remembers only the old tier cannot explain whether a later admission used the old allowance intentionally or merely read stale state. Record the effective entitlement revision on each admission and define how a change takes effect. A downgrade should constrain new reservations under the new policy; handling previously authorized work requires an explicit commitment policy. For tenant notices, that policy must answer whether an already reserved batch may finish, whether unstarted chunks require a fresh admission, and which revision explains each outcome. Without those answers, an audit log can faithfully preserve a decision that nobody can interpret. Treat revision transitions as part of the admission protocol and test them alongside concurrency, rather than leaving their meaning to whichever worker happens to finish first.

Preserve the evidence.

For a reviewable example, suppose a test account has a period ceiling of 100,000 internal budget units and the tenant-notice workload has a ceiling of 20,000. These are invented test inputs, not currency, measured usage, or a published plan. A request that reserves 600 units needs both ceilings checked against consumption plus outstanding reservations in the same period. Scope the workload ceiling to the account and workload together. Otherwise an apparently narrow workload control can become a route around the account boundary.

## Detect stale authority before it admits more work

The operational signal is a mismatch between the authority recorded at admission and the authority currently effective for that account. Track entitlement age, rejected revision changes, reservation failures, and the number of admissions lacking a complete audit record. A proposed safety SLO is that every admitted billable operation has a durable reservation and an attributable authorization decision. Choose a separate availability objective for successful admissions; mixing policy denials into service failures obscures both.

A cache needs a stated freshness policy. For this example, use a maximum age of 60 seconds, reject snapshots dated in the future, and stop new billable admissions when the adapter cannot obtain an acceptable decision. That interval is a design assumption to test against the business's revocation requirements. If access must stop immediately, an age check alone is insufficient: admission must also validate a current revision or revocation fence at the authoritative transaction boundary.

Do not silently extend freshness during an outage.

The concurrency trap is less visible. Two workers can read the same remaining allowance, both pass a local check, and collectively exceed the ceiling; increasing the refresh frequency cannot make that read-then-spend sequence atomic. The ledger must serialize competing reservations across the account and workload boundaries. Include queued and running reservations in capacity planning, since a burst of admitted work consumes commitment capacity before any completion event arrives.

## Implement three checks at the spending boundary

Keep the Node.js application responsible for authenticating its caller and resolving the subscription through a server-side adapter. The Go example below defines the admission boundary, using an internal schema rather than an external API response. The trusted policy layer constructs the decision, including the maximum permitted snapshot age; the worker cannot select its own ceiling or freshness window.

```go
package admission

import (
	"context"
	"errors"
	"time"
)

type Decision struct {
	Account, Principal, Workload string
	Tier, Revision, Period      string
	Allowed                     bool
	ObservedAt                  time.Time
	MaxAge                      time.Duration
	AccountCap, WorkloadCap      int64
}

type Reservation struct {
	Decision Decision
	Key      string
	Units    int64
}

type Ledger interface {
	// Atomically validate the revision and reserve against both caps.
	// Persist the decision and reservation before returning success.
	// Bind Key to Account, Principal, Workload, Period, and Units.
	// An identical retry returns its existing ID; a conflict is rejected.
	Reserve(context.Context, Reservation) (string, error)
}

func Admit(ctx context.Context, ledger Ledger, d Decision,
	principal, workload, key string, units int64, now time.Time,
) (string, error) {
	if d.MaxAge <= 0 || d.ObservedAt.IsZero() ||
		d.ObservedAt.After(now) || now.Sub(d.ObservedAt) >= d.MaxAge {
		return "", errors.New("entitlement decision is not fresh")
	}
	if !d.Allowed || d.Account == "" || d.Principal == "" ||
		d.Workload == "" || d.Tier == "" || d.Revision == "" ||
		d.Period == "" || d.Principal != principal ||
		d.Workload != workload {
		return "", errors.New("workload is not authorized")
	}
	if key == "" || units <= 0 || d.AccountCap <= 0 ||
		d.WorkloadCap <= 0 || units > d.AccountCap ||
		units > d.WorkloadCap {
		return "", errors.New("invalid reservation")
	}
	return ledger.Reserve(ctx, Reservation{
		Decision: d,
		Key:      key,
		Units:    units,
	})
}
```

The interface carries the hardest requirement. Its implementation must atomically check the effective revision, compare both counters with their ceilings, and persist the reservation and audit evidence; the function's input validation cannot provide those guarantees. Use overflow-safe comparisons, such as checking requested units against remaining capacity after validating the counters. On a ledger timeout, treat the result as unknown and retry with the same key. Starting the job with a fresh key would defeat duplicate protection.

There is also a boundary around the word “spend.” A reservation caps exposure only if its units conservatively bound the downstream operation's maximum liability and every billable path requires admission. When that bound is unavailable, constrain the operation into bounded chunks or obtain an enforceable downstream limit before claiming a hard cap. Reconcile actual consumption against reservations, retain the authorization record, and release unused capacity only after the operation's outcome is known. An expired lease alone is insufficient evidence that external work stopped.

## Decide who operates the ledger and its evidence

Buy versus build should turn on demonstrable admission semantics and audit export. A feature named “budget” leaves too much unanswered.

| Decision criterion | Managed control boundary | Self-operated control boundary |
| --- | --- | --- |
| Authorization evidence | Require exportable decisions with principal, workload, period, and revision | Design the event schema, retention, and restricted read access |
| Concurrent admission | Verify that enforcement reserves capacity before execution | Implement and test atomic account and workload reservations |
| On-call responsibility | Establish what happens when the remote decision service is unavailable | Own storage availability, contention, recovery, and reconciliation |
| Exit and maintenance | Verify that reservation history and policy state can be exported | Maintain schema migrations and a stable adapter contract |

This is an evaluation framework, not an assertion that either option supplies those guarantees. A managed service may reduce operating work while introducing dependency and export constraints; a self-operated ledger gives the team direct control while adding a critical component to its on-call surface. Estimate peak admission rate, outstanding commitments, audit retention, and recovery work before choosing. Unit pricing cannot answer those questions.

The trade-off is operational ownership. This design is not suitable for a workflow whose maximum billable liability cannot be bounded before execution; use bounded operations or an enforceable downstream ceiling first. Its limitation is that reading subscription entitlements cannot itself stop an independently authorized billing path. Don't call the result a hard spending cap until that path is controlled.

Keep subscription credentials outside worker payloads. Apply least privilege to the adapter, separate entitlement reads from billing administration where supported, and record credential lifecycle events without putting secret values into logs. These controls follow the scope of OWASP's guidance [1]; they complement the proposed admission protocol.

## Verify enforcement, then roll back without reopening spending

Before deployment, race two 600-unit reservations against 1,000 remaining units in a test ledger. Exactly one should succeed. Retry its key and expect the same reservation identifier without another debit; change the amount under that key and expect rejection. Test the account ceiling with several workloads, since a workload-only concurrency test can miss the shared-account failure.

Also test stale and future-dated decisions, a revision change during admission, and a lost response after commit. Recover the committed reservation after that lost response. Verify that its audit evidence contains the authenticated principal, workload, effective revision, period, reserved amount, and outcome, while excluding credentials and tenant document contents.

Roll out in observation mode first to compare proposed decisions with existing admissions, while retaining the existing enforced boundary. Observation does not cap spending. Enable enforcement only after every billable path uses the gate and reconciliation can explain outstanding reservations. Provision for the measured admission rate and ledger contention, with enough headroom for retries during recovery; no fixed capacity number is defensible without that measurement.

For rollback, stop admitting new work, preserve existing commitments, and restore the previous verified adapter or gate version. Avoid falling back to a hardcoded tier map. Resume only when decisions, reservation state, and effective policy revisions reconcile. The useful operational outcome is an auditable spending boundary that remains enforceable when subscriptions change and requests race.

## References

1. OWASP, Secrets Management Cheat Sheet — least privilege, auditing, and secret lifecycle guidance: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
