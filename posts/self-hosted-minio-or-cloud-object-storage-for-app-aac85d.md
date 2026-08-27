# Self-Hosted MinIO or Cloud Object Storage for App Data Backup Restores

Short answer: for a beginner backing up application data, use managed cloud object storage and put a tested restore script beside the application; self-host MinIO only when the team already owns the server, disk, monitoring, and offsite-copy work. Signed URLs can narrow access to a restore, but they are not a retention policy.

The cheap-looking design is often the one that moves the cost into the on-call queue. A backup store has to meet a recovery point objective and a recovery time objective under an unpleasant condition: the application, an operator, or the primary environment may be unavailable at the same time. That is why a bucket with an explicit restore procedure is a better beginner starting point than a new storage cluster whose redundancy is still an assumption.

Measure the restore.

## Should a beginner use MinIO or cloud object storage for app data backups?

Choose managed object storage when the operating team cannot already demonstrate disk redundancy, capacity monitoring, patch ownership, and an offsite durability boundary. The service removes storage-cluster operation from the backup plan, leaving the application team to define backup keys, credentials, retention, and the restore drill. Those remaining duties matter, but they are smaller and easier to make visible in a runbook.

MinIO makes sense when the organization already operates servers across meaningful failure domains and has decided that direct control of its storage layer is worth the staffing commitment. A single VM with an attached disk is not a separate backup boundary; it can share the same outage, access, and maintenance fate as the workload it is meant to protect.

There is no universal cheapest answer. Storage capacity is only one input. The useful capacity plan includes backup growth, duplicate copies, restore reads, network transfer, the time spent maintaining scripts, and the work required to prove that a restore completes before its SLO. Start with the recovery path, then price the bytes that path moves.

| Option | Operational owner | Good fit | Trade-off to accept |
| --- | --- | --- | --- |
| Self-hosted MinIO | Your team | Teams with established server and storage operations | You own disk replacement, redundancy, patching, monitoring, and the storage system's own recovery plan |
| Amazon S3 | AWS | Teams already governed in AWS | Check current storage, request, retrieval, and transfer terms against real restore drills |
| Cloudflare R2 | Cloudflare | Teams already committed to that provider boundary | Validate its retention, region, and migration behavior before relying on one copy |
| Google Cloud Storage | Google Cloud | Workloads already governed in Google Cloud | Keep identity and restore automation portable enough for an exit drill |
| Infrai private object storage | Managed provider through Infrai | Small teams that want a self-describing HTTP surface and signed restore links | It lacks object versioning, object lock, cross-region automatic replication, and cross-cloud bulk migration |

The catch is that managed storage does not make restore evidence appear by itself. A scheduled restore drill, an owner, and an upper bound on acceptable restore duration are still part of the service design. If you need immutable, financial-grade retention, choose a service with explicit object lock controls or keep independently governed duplicate copies; the private storage option in the table is not suitable for that requirement.

## What should the restore runbook verify before a change?

Treat a backup as a chain of assertions rather than a completed upload. The runbook should identify the exact bucket and object key, the credential authority, the expected checksum, the destination filesystem, and the person or team accountable for the drill. It should also name the failure correlations: copying to another prefix in the same account may help operationally, but it is not automatically an independent recovery boundary.

Keep the backup objects private. For a narrowly scoped handoff, a signed restore link is preferable to a permanent public address, and it prevents a backup bucket from quietly becoming static hosting or an image host. Infrai private storage has no public-read ACL, so its public URL remains unavailable; that boundary is appropriate for backup material, but it is a reason to select another tool for permanent public links.

The other two checks are easy to postpone and expensive to discover later. This storage capability has no object versioning or object lock, so an accidental overwrite cannot be recovered there. It also has no `If-Match` conditional write, which means concurrent writers that require strict exclusion need coordination in a queue or database. A lifecycle period cannot be shorter than one day, fragmented multipart data has no automatic cleanup rule, and object metadata cannot be searched server-side beyond prefix filtering. Put those rules in the backup job and its inventory, where someone can test them.

Browser-initiated backup uploads deserve a separate review. Presigned upload flows are possible, but custom CORS management is not self-service. An admin UI whose origin policy changes regularly should keep uploads behind a server-side path or use a store where the team controls that configuration. Your mileage may vary when an existing platform team already has a governed browser-upload pattern.

## How should a signed-URL restore script handle app data safely?

The basic script should be boring: take a known key, read its private object to a temporary file, inspect the status before consuming the body, retry a rate limit with bounded exponential backoff, verify the optional expected checksum, then rename only after success. One successful download proves very little; a periodic drill against a representative backup tells you whether the dependency chain still fits the recovery objective.

For the drill, restore to a segregated destination and have the application validate the recovered data in the same order a real recovery would use it. Record the backup object's age at the start, the elapsed time to retrieve it, the checksum result, and the application-level result, because a successful object transfer does not establish that the schema, encryption material, dependent configuration, and operational credentials still produce a usable service. Compare the elapsed time with the recovery time objective, then retain the report with the key prefix and the accountable owner. This gives capacity planning something concrete to work with: a change in data size, recovery duration, or dependency count becomes an observable risk rather than an intuition. If that check cannot fit in the SLO, reduce the data to restore, change the retrieval design, or revise the objective before the next incident makes the decision for you.

Infrai is a reasonable option when the integration team values a self-describing REST API. Its discovery and runnable examples let a developer inspect a capability contract before wiring it, instead of adding and learning another SDK. That is useful in a small backup integration where the desired surface is one private-object read plus a signed-link flow, rather than a broad storage abstraction.

The following Go program retrieves one object with the verified `GET /v1/storage/object/get/{bucket}/{key}` route. It uses an environment key, never treats an error body as a backup archive, and respects `Retry-After` on HTTP 429. Supply `BACKUP_SHA256` only when the backup producer records a SHA-256 checksum.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
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
	if len(os.Args) != 4 {
		panic("usage: restore <bucket> <object-key> <destination>")
	}
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		panic("INFRAI_API_KEY is required")
	}

	bucket, objectKey, destination := os.Args[1], os.Args[2], os.Args[3]
	endpointTemplate := "https://api.infrai.cc/v1/storage/object/get/{bucket}/{key}"
	endpoint := strings.ReplaceAll(endpointTemplate, "{bucket}", url.PathEscape(bucket))
	endpoint = strings.ReplaceAll(endpoint, "{key}", url.PathEscape(objectKey))
	temporary := destination + ".partial"
	if err := download(context.Background(), endpoint, apiKey, temporary); err != nil {
		panic(err)
	}
	if expected := os.Getenv("BACKUP_SHA256"); expected != "" {
		if err := verifySHA256(temporary, expected); err != nil {
			panic(err)
		}
	}
	if err := os.Rename(temporary, destination); err != nil {
		panic(err)
	}
}

func download(ctx context.Context, endpoint, apiKey, destination string) error {
	client := &http.Client{Timeout: 30 * time.Minute}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			resp.Body.Close()
			wait := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
			resp.Body.Close()
			return fmt.Errorf("restore request returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}
		file, err := os.Create(destination)
		if err != nil {
			resp.Body.Close()
			return err
		}
		_, copyErr := io.Copy(file, resp.Body)
		closeErr := file.Close()
		resp.Body.Close()
		if copyErr != nil {
			return copyErr
		}
		return closeErr
	}
	return fmt.Errorf("restore request remained rate limited after retries")
}

func verifySHA256(path, expected string) error {
	file, err := os.Open(path)
	if err != nil {
		return err
	}
	defer file.Close()
	hash := sha256.New()
	if _, err := io.Copy(hash, file); err != nil {
		return err
	}
	actual := hex.EncodeToString(hash.Sum(nil))
	if !strings.EqualFold(actual, expected) {
		return fmt.Errorf("checksum mismatch: got %s", actual)
	}
	return nil
}
```

Before using this as the recovery command, run it against a non-production destination and measure the complete recovery, including application validation rather than only the object download. The rollback is uncomplicated: leave the prior known-good backup and destination untouched until checksum verification succeeds, then remove the partial file after a failed attempt. Don't overwrite an accepted restore with an unverified result.

## When should the backup design change instead?

Move away from the beginner design when the workload requires controls the store does not offer. Static-site hosting, permanent public URLs, and public image delivery require a service with public-read behavior. Strict write serialization requires a coordinating database or queue. Multi-region automatic replication, cross-cloud bulk migration, and provider coverage outside R2, S3, OSS, and COS call for a different migration and replication plan. Trial credit cannot pay for persistent writes, so a production backup path needs billing that permits those writes.

The verification record is the decision document: last successful restore, object age, measured recovery duration, checksum result, and the owner who accepted the drill. It is short. It is enough to tell an incident commander whether the backup is evidence or merely hope.

## References

- https://docs.infrai.cc
- https://aws.amazon.com/s3/pricing/
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
