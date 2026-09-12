# Tenant-Isolated Avatar Delivery: Public URLs, Private Objects, and Expiring Signed Access

Short answer: treat an avatar's object key as durable data and treat every signed download URL as a temporary credential. For a customer-support application, the safest default is a private bucket with an application-controlled read path, tenant authorization before signing, and a refresh policy that turns expiry into an ordinary cache event rather than a user-visible outage. A permanent anonymous URL belongs behind a deliberately public delivery boundary, not inside a private object-store contract.

## Start with the tenant boundary, not the image tag

The failure usually appears in the browser as a broken `<img>`: the database contains a `public_url`, the page requests it, and the request cannot retrieve the avatar. That symptom is ambiguous. The object may be private, the URL may be an expired signed credential, or the browser may be enforcing a cross-origin policy that a server-side probe never exercised.

In a support system, the first question is not “can this URL be opened?” It is “which tenant is allowed to ask for this object?” Keep that decision in the application. Store a tenant identifier, an immutable object key, content type, and a revision or upload timestamp. Do not store a signed URL as the avatar's identity: its query parameters are authorization material with a lifetime, while the object key is what your restore and replacement workflows need.

A useful key shape might be `tenants/acme/users/42/avatar/7f3c.png`, but the prefix is not an authorization system. Every read still needs a server-side check that the authenticated subject can view tenant `acme`, followed by a check that the selected key belongs to that tenant. This matters during support impersonation, exports, and restore jobs, where a valid session can otherwise become a confused deputy.

## Why do avatar public URLs fail when private objects and signed access expire?

These are three different contracts:

| What the client has | What it means | Correct response |
|---|---|---|
| A stable public URL | Anonymous readers are allowed by policy | Operate it like public media, with cache and abuse controls |
| A signed download URL | A time-limited reader has been authorized | Mint a fresh URL from the durable key when it expires |
| A private object key | An internal reference with no browser access by itself | Resolve it through the application's authorization path |

Expiry does not imply that the avatar disappeared. It says that a particular bearer credential has passed its validity window. A browser can also hold an expired URL in an image cache while the object remains perfectly readable through a new authorization decision.

CORS is a separate browser rule. A request made by the application server can succeed while a browser request from the web application's origin is rejected because the response does not permit that origin. The browser's origin, method, and requested headers must be tested in the real deployment; a command-line download alone cannot establish that the `<img>` workflow is valid. The MDN CORS guidance describes this browser-enforced exchange and its preflight behavior.

For private avatars, I prefer an application-owned endpoint that checks the session and either streams the object or redirects to a freshly minted signed URL. The endpoint is the stable application contract. The signed URL is an implementation detail that can change on every request.

Keep it private.

## Implement a bounded read and refresh path

The request path should have a small state machine: authorize, resolve the current key, obtain a short-lived download credential, and return the image. If a download credential expires during a page load, retry the resolution once. After that, return a placeholder and emit a metric with tenant, object revision, and failure class, without putting the browser into an unbounded retry loop.

This Go example keeps the durable reference separate from the temporary URL. The storage implementation is intentionally an interface so the policy can be tested without coupling tenant authorization to a particular object-store SDK.

```go
package avatar

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"time"
)

var ErrExpiredDownload = errors.New("signed download expired")

type AvatarRef struct {
	TenantID string
	ObjectKey string
	Revision string
}

type Authorizer interface {
	CanViewAvatar(ctx context.Context, subjectID, tenantID, userID string) bool
}

type ObjectReader interface {
	SignedGet(ctx context.Context, tenantID, objectKey string, lifetime time.Duration) (string, error)
}

type Handler struct {
	Auth Authorizer
	Store ObjectReader
	URLLifetime time.Duration
}

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	subjectID := r.Header.Get("X-Subject-ID")
	tenantID := r.PathValue("tenant")
	userID := r.PathValue("user")
	if subjectID == "" || tenantID == "" || userID == "" || !h.Auth.CanViewAvatar(r.Context(), subjectID, tenantID, userID) {
		http.Error(w, "not found", http.StatusNotFound)
		return
	}

	ref, err := loadAvatarRef(r.Context(), tenantID, userID)
	if err != nil || ref.TenantID != tenantID {
		http.Error(w, "not found", http.StatusNotFound)
		return
	}

	downloadURL, err := h.Store.SignedGet(r.Context(), ref.TenantID, ref.ObjectKey, h.URLLifetime)
	if err != nil {
		http.Error(w, "avatar unavailable", http.StatusServiceUnavailable)
		return
	}

	http.Redirect(w, r, downloadURL, http.StatusFound)
}

func loadAvatarRef(ctx context.Context, tenantID, userID string) (AvatarRef, error) {
	// Replace this boundary with the transactional repository used by the service.
	if tenantID == "" || userID == "" {
		return AvatarRef{}, errors.New("missing avatar identity")
	}
	return AvatarRef{TenantID: tenantID, ObjectKey: fmt.Sprintf("tenants/%s/users/%s/avatar/current", tenantID, userID), Revision: "current"}, nil
}
```

The example's `X-Subject-ID` is a placeholder for the service's authenticated identity; production code should take that identity from its established session or token middleware. The important invariant is that the tenant comes from the authorized route and repository record, never from a client-supplied object key that is accepted without validation.

Keep the URL lifetime long enough for the page's normal loading and caching behavior, but short enough that a leaked URL is not a durable access grant. I'm not sure one duration fits every support console, mobile client, and CDN policy; your mileage may vary. Measure refresh rate, signed-download failures, image latency, and authorization denials before changing it.

## Verify isolation, expiry, and restore before rollout

A restore test is more than “the object exists.” Create two tenants with same-named users and deliberately similar avatar filenames. Verify that tenant A can read only its own revision, that tenant B receives the same externally safe not-found response for tenant A's key, and that a support operator's elevated workflow is separately audited. Then replace an avatar, confirm the database pointer moves to the new immutable key only after a successful read, and verify that the old key follows the retention policy rather than becoming an accidental public backup. For a more useful failure exercise, revoke the test subject's tenant membership between authorization and download, repeat the request with an old signed URL, and inspect both the audit record and the object access log; the expected result is that the stable application path makes a new authorization decision, while the old credential remains bounded by its expiry and cannot be treated as proof of current membership. Run the same sequence during a restore, when records and objects are changing together, because that is where an apparently harmless cached URL can hide a cross-tenant reference or point at the wrong revision. Keep the test data disposable, retain the event correlation ID, and make the assertion about tenant and revision explicit in the test output rather than inferring it from a successful HTTP status.

Exercise the browser origin as well as the storage API. Load the avatar from the deployed web origin, allow a signed URL to expire, refresh once, and inspect the response headers and cache behavior. A passing server probe is not enough if the browser's CORS or redirect handling differs. Record these checks as a release test, not as a one-time debugging session.

For capacity planning, size the application read path for peak avatar resolutions, not merely for upload volume. A page with 50 tickets can produce 50 resolutions, and a reconnecting support console can repeat them. Cache the authorized result only within a clearly defined tenant and subject scope, coalesce concurrent refreshes for the same `(tenant, user, revision)` key, and put a ceiling on signing work per request. The SLO should state both image delivery latency and the rate of authorization or refresh failures; one aggregate “storage healthy” metric hides the failure users actually see.

Rollback should be pointer-based. Preserve the prior object key until the new object has passed content validation and a read check, then change the reference in one transaction. If validation fails, keep serving the previous revision. This is easier to reason about than overwriting a key and trying to reconstruct which bytes were present when a support case was opened.

## Choose the boundary that matches the audience

The catch is that a private, application-controlled path is not suitable for anonymous public profiles, email clients that cannot renew credentials, or static pages that must be crawled without a session. In those cases, use a separately governed public-media or edge-delivery boundary with explicit abuse controls, cache policy, and an SLO. Stick with the private path when tenant authorization matters more than permanent hotlinkability.

The buy-versus-build decision is operational, not a contest of URL formats:

| Boundary | Strength | Cost or limitation | Use it when |
|---|---|---|---|
| Application-mediated private read | Central tenant policy, audit, and restore control | Adds application traffic and signing work | Avatars are shown to authenticated support users |
| Direct signed object read | Low application bandwidth and a simple transfer path | The browser must handle expiry, redirects, and origin policy | Clients can refresh credentials predictably |
| Public media delivery layer | Stable anonymous access and cache-friendly distribution | Requires separate publishing, abuse, and invalidation controls | Profiles or embeds must work without a session |
| Self-hosted object storage | Control over placement, identity, and retention | Your team owns upgrades, durability design, replication, and on-call | Those responsibilities are part of the platform mandate |

Do not choose a public URL because a database column named `public_url` already exists. Choose it only after the threat model, tenant isolation test, retention requirement, and audience are explicit. A URL is a delivery mechanism; it is not proof that the caller should see the bytes.

## References

- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
