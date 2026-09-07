# Tenant-Scoped Property Images: Node.js Express React Uploads to Private Storage

Short answer: keep the bucket private, have Node.js and Express authorize an immutable object key for the tenant, let React upload the bytes with a short-lived presigned PUT, and issue a separate signed download URL only after checking the tenant-scoped record. That is the right default for property images and tenant exports; it is a poor fit when the application cannot control browser CORS, needs permanent public URLs, or requires storage-native retention guarantees that the chosen service does not provide.

The first rule is identity before bytes. A property-management application should never infer tenant ownership from an object key supplied by the browser. The server creates the key, records the tenant and intended object in its database, and treats the upload as pending until the workflow verifies it. The object store is the data plane. Express is the control plane.

## How should Node.js and Express authorize React uploads to private storage?

The browser sends an upload intent, not the file, to Express. The request should carry the property or listing identifier, declared media type, and a client idempotency token. Express authenticates the user, resolves the tenant from server-side session data, checks the size and type policy, allocates an opaque key, writes a pending row, and returns a presigned PUT URL. React then sends the image directly to object storage. The application tier does not carry the image stream.

That order matters for a tenant-scoped export. An export job should select rows by tenant id in the database, read only the object keys attached to those rows, and write its result under a new key owned by the same tenant. A prefix such as `tenant-a/` is useful for operators, but it is not an authorization check; the database relationship is the check.

Here is the small part I want reviewed in a pull request: key allocation. It deliberately has no storage SDK in it, because the authorization invariant should survive a provider change.

```go
package storagepolicy

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"strings"
)

type UploadIntent struct {
	TenantID   string
	PropertyID string
	MediaType  string
}

type PendingObject struct {
	TenantID  string
	ObjectKey string
	MediaType string
	State     string
}

func AllocateImage(intent UploadIntent) (PendingObject, error) {
	if intent.TenantID == "" || intent.PropertyID == "" {
		return PendingObject{}, fmt.Errorf("tenant and property are required")
	}
	if !strings.HasPrefix(intent.MediaType, "image/") {
		return PendingObject{}, fmt.Errorf("media type is not an image")
	}

	var nonce [16]byte
	if _, err := rand.Read(nonce[:]); err != nil {
		return PendingObject{}, fmt.Errorf("allocate object key: %w", err)
	}

	return PendingObject{
		TenantID:  intent.TenantID,
		ObjectKey: "images/" + intent.TenantID + "/" + hex.EncodeToString(nonce[:]),
		MediaType: intent.MediaType,
		State:     "pending",
	}, nil
}
```

The real handler persists the returned object key before asking a presigner for a URL. It must bind the signed request to that key and the intended method, and it should make the URL expiry long enough for the measured upload tail rather than for an entire workday. A retry reuses the same pending record. It does not allocate a new object merely because the browser did not receive a response.

Short path. Good boundary.

The key never comes from React.

No public fallback.

That one rule also makes retries explainable. If the browser loses its connection after storage accepts the bytes, Express can safely reissue authority for the same pending key, while a new user action can receive a new attempt key. The export worker can then distinguish an abandoned first attempt from a replacement image instead of guessing from timestamps or filenames. I want that distinction visible in logs and in the database state: intent accepted, capability issued, transfer observed, object verified, record ready, and object retired. Each transition has an owner, which gives the on-call engineer a bounded place to look when an export is missing an image.

## The incident lesson is tenant identity, not file bytes

The bounded production scenario I design around is a leasing agent replacing a listing photo while a tenant export is being assembled. The browser has two legitimate requests in flight, but only one should be allowed to change the listing's current image. If the client can choose the final key, a stale tab can overwrite a newer image; if the export trusts a client-supplied prefix, it can collect another tenant's objects. I have seen teams solve the visible symptom by increasing the Express tier. That treats a data-model error as a capacity problem.

The invariant is simpler: immutable upload-attempt keys, tenant ownership in an application record, and an explicit state transition from pending to ready. The export reads ready records through a tenant-filtered query. A reconciliation worker handles objects whose browser completion event was lost. It can compare the pending table with storage inventory or object headers, then either confirm the record or apply the application's deletion policy. Listing by prefix alone cannot answer which image belongs to which property, who authorized it, or whether an export already included it.

Capacity planning follows the same boundary. Track signing requests, concurrent browser transfers, bytes in flight, pending-object age, export duration, and failed authorization separately. They are different SLOs. A signing endpoint can be healthy while a slow upload leaves a growing pending queue; an export can be slow while the control plane is accepting requests normally. Alerting on one blended request latency hides the failure mode the on-call engineer needs to isolate.

The browser also needs a precise CORS contract. The storage origin must allow the deployed React origin, the PUT method, and every request header the browser sends. A preflight failure is a browser policy decision, not evidence that the application authorization endpoint is unavailable. MDN's CORS guide is the right baseline for testing that distinction. Test the production origin, not only localhost, and test the actual headers emitted by the React client.

## What should a signed download prove before an export is released?

A signed download URL is a bearer capability. It proves possession of the URL for its lifetime; it does not decide whether the requester is allowed to see a property image. Express must perform that decision first, using the authenticated tenant and the database row, then request a fresh signed read for the exact object key. Never accept a key from the browser and skip the database lookup because the key appears to contain a tenant name.

For a tenant-scoped export, authorization is a set operation. Select exportable rows for one tenant, materialize the selected keys, and record the export's tenant and source snapshot. The download endpoint checks that record before issuing a URL. Keep the URL lifetime bounded to the actual handoff, and avoid placing it in durable application logs or analytics events; query parameters are easy to copy.

The export itself needs a clear consistency decision. If images may change during generation, record whether the export is a point-in-time selection or a best-effort current view. If the product promises a point-in-time result, the database needs a snapshot or immutable source references; object naming alone cannot provide that promise. This is where a queue, transaction, or dedicated export worker earns its keep.

Object lifecycle is part of the design, not housekeeping after launch. Abandoned pending images, expired download artifacts, and temporary export parts need an explicit deletion deadline. The storage lifecycle documentation cited below describes lifecycle management, but the application still owns the meaning of “safe to delete”: it must know whether an object is pending, referenced by a ready record, or part of an audit requirement.

## Where does this design stop being a good fit?

The catch is operational ownership. A direct browser upload is unsuitable when the team cannot configure or validate CORS for every production origin, when a compliance workflow requires immutable retention, or when strict compare-and-swap writes are central to correctness and the object layer does not expose the needed conditional semantics. In those cases, use a storage system and control plane whose retention and concurrency contracts are explicit, or put a dedicated application service in front of it.

It is also the wrong shape for permanent public media. Signed URLs expire, and a private bucket intentionally makes link sharing an authorization workflow. For public listing thumbnails, a separate publication path can create derivative media with a different access policy; do not weaken the private source bucket to make that delivery path convenient.

Your mileage may vary on expiry windows. The right value comes from observed upload duration, retry behavior, and the risk of a leaked bearer URL, not from a universal number. I would start with a staging test that measures the high-percentile transfer time, then add a bounded retry allowance and verify that a retry cannot change tenant or object identity.

## A buy-versus-build review for the control plane

The decision table I use with a platform team is intentionally unglamorous. It weighs on-call load and lock-in against the storage contract the product actually needs.

| Approach | Strong fit | Boundary to verify |
|---|---|---|
| Managed object storage with a thin Express signer | Teams that want direct browser transfer and a small application data plane | CORS administration, signed URL semantics, lifecycle behavior, and tenant audit evidence |
| Self-hosted S3-compatible storage | Teams that need control over placement and operations | On-call coverage, upgrades, replication, durability evidence, and egress capacity |
| Application-proxied uploads | Small files or workflows requiring byte-level inspection before storage | Express connection capacity, memory and backpressure, bandwidth cost, and larger blast radius |
| Dedicated media service | Transformation, moderation, or delivery is the product concern | Tenant isolation, export semantics, retention, and portability of original objects |

Do not choose a managed service solely because its first invoice looks attractive, and do not choose self-hosting solely to avoid a provider name in the architecture diagram. Put the decision through a failure exercise: revoke a session, replay an upload intent, lose the completion event, generate an export during an image replacement, and attempt a cross-tenant read. The design that makes those tests boring is the one worth operating.

For this property-management scenario, I would ship the private direct-upload pattern when the database is authoritative for tenant ownership, CORS is tested in deployment, and reconciliation plus deletion are observable. I would keep a provider-neutral adapter around signing and object metadata, but I would not pretend that adapter makes retention, replication, or public delivery equivalent across systems. Those are product and compliance decisions, not interface trivia.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
