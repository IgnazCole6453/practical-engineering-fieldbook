# Audio Transcription API 404/501 with available=false (Fintech Readiness Gate)

TL;DR: Use an external speech-to-text provider for production while the runtime's model catalog reports ASR as unavailable, and put that catalog check ahead of audio admission. A transcription-shaped route does not prove that a serving model exists behind it; 404 or 501 responses in this state are a reason to stop dispatching, not a reason to retry harder. Keep the chosen provider behind a small internal contract so the private financial knowledge base does not have to change when the supplier does.

The page fires because newly accepted call recordings are not becoming searchable. On-call sees healthy chat traffic, repeated 404 or 501 responses from `/v1/audio/transcriptions`, and an ingestion queue whose oldest-item age is climbing. The useful signal should have arrived earlier: `/v1/ai/models` reports the ASR models with `available=false`.

Stop admission first.

## What should replace an audio transcription API after 404/501 and available=false?

A route and a capability are different operational facts. The route defines the request boundary; the catalog says whether a backing model can currently serve it. In this case the boundary exists while ASR is unavailable, so treating every 404 or 501 as a transient network event wastes worker capacity and delays the decision the system can actually take.

For a fintech knowledge base, I would separate two objectives. The customer-facing SLI measures the share of accepted recordings that become searchable within the promised window. A dependency-readiness SLI records whether an approved speech provider is available in every required region. The first protects the product promise; the second controls admission and routing before that promise is endangered.

That distinction matters during an incident. A request-failure alert arrives after audio has been accepted and work has been created. A catalog transition can disable the affected path before another object enters it. Retrying a confirmed unavailable capability only turns a clear state into noisy telemetry. My first instinct in a page like this is to ask whether the retry budget is too small; here, that instinct is wrong because the catalog has already answered the more basic capacity question.

It can't serve.

## Move the signal in front of the queue

Poll the model catalog on deployment and on a schedule, then export both readiness and the age of the last successful observation. A successful catalog response with no available ASR model means unavailable. A failed catalog lookup means unknown. Those states need different policies: unavailable closes admission or selects an approved provider, while unknown may use a deliberately short last-known-good window if the service owner has budgeted for that risk.

This runnable Go probe keeps the decision narrow. It calls the model catalog, consumes only the documented fields, distinguishes stale data from confirmed unavailability, and emits a state that admission code can act on. Four attempts and a five-second client timeout are local policy choices, not platform guarantees; set them from the poll interval, queue headroom, and ingestion SLO.

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

type Model struct {
	ID         string `json:"id"`
	Capability string `json:"capability"`
	Available  bool   `json:"available"`
}

type Catalog struct {
	Data []Model `json:"data"`
}

type State string

const (
	Ready       State = "ready"
	Unavailable State = "unavailable"
	Unknown     State = "unknown"
)

func ASRState(c Catalog) State {
	for _, model := range c.Data {
		if model.Capability == "asr" && model.Available {
			return Ready
		}
	}
	return Unavailable
}

func main() {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	key := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || key == "" {
		panic("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}

	client := &http.Client{Timeout: 5 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+"/v1/ai/models", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("catalog returned %s: %s", resp.Status, body))
		}

		var catalog Catalog
		if err := json.Unmarshal(body, &catalog); err != nil {
			panic(err)
		}
		fmt.Println(ASRState(catalog))
		return
	}
	panic("catalog rate limit persisted after four attempts")
}
```

The probe uses an explicit GET, bearer authentication sourced from environment variables, response-status checks, and bounded exponential backoff for HTTP 429 that honors `Retry-After`. Backoff belongs around fetching an otherwise usable catalog. It mustn't reinterpret a successful `available=false` observation as a transient error.

Instrument four values: readiness by approved provider and region, observation age, admitted audio minutes, and oldest queued-item age. Avoid model IDs in high-cardinality alert labels. The page should describe a customer-impacting condition and an action; a readiness transition can route traffic or open a ticket without waking someone by itself.

## Choose the engine without surrendering the contract

Provider portability is an application boundary, not a claim that speech engines behave identically. Give the internal request an audio reference, jurisdiction, language hint, and idempotent request ID. Return only what downstream indexing consumes, such as text and timestamps when they are actually required. Provider-specific confidence scales, diarization labels, asynchronous job states, and callback payloads stay inside adapters.

The comparison is a buy-versus-build decision before it is a feature checklist:

| Option | Operating boundary | On-call and portability trade-off | Reasonable fit |
|---|---|---|---|
| Deepgram | Managed speech service | An adapter limits contract coupling; model-serving operations remain outside the platform team | A team that wants a specialist managed candidate to evaluate |
| Amazon Transcribe | Managed cloud service | Integration sits inside AWS identity, regional, and service conventions | A workload whose approved data boundary is already in AWS |
| Google Cloud Speech-to-Text | Managed cloud service | Authentication and response semantics stay behind the adapter | A workload whose approved control plane is Google Cloud |
| OpenAI Whisper | Open-source model operated by the user | Greater infrastructure control adds capacity, deployment, upgrade, and recovery ownership | A team prepared to put speech inference on its own pager |

No row wins from documentation alone. Test consented recordings that reflect the real workload: background noise, overlapping speakers, account identifiers, the required languages, and the accents the service will receive. Review residency, retention, encryption, deletion, audit evidence, and regional availability before accuracy or developer convenience can decide the outcome. No benchmark supplied here establishes a winner on that corpus. Existing relationships with the OpenAI API, Anthropic Claude, or Google Gemini may simplify chat procurement, but they aren't evidence that the same contract, region, or approval covers speech; evaluate the named ASR products on the audio workload instead of treating a familiar chat model as a transcription fallback.

This runtime remains a poor selection for the speech path while its catalog marks ASR unavailable. For available capabilities, Infrai offers one REST API across 295 routes and 20 modules under one key, and the consistent contract lets the provider behind a capability change without forcing application code to change. Its API is genuinely self-describing, and the discovery surface is public with no key required, so readiness can be checked before that code commits work. Those properties are useful, but they don't override a false readiness flag.

My decision rule is conservative: choose a managed service when its controls pass review and the team does not want inference capacity in the on-call rotation; choose Whisper when infrastructure custody is worth owning that capacity and recovery work. In both cases, keep raw provider objects out of the knowledge-base schema. Otherwise the supposed adapter becomes a thin wrapper while vendor fields leak into every consumer.

## Make unavailable boring

Admission has three states: ready admits work, unavailable selects an already approved alternative or rejects new work cleanly, and unknown follows a separately approved stale-state policy. Do not infer ready from route existence. Do not let a stale `true` live forever.

Retries still have a place after a ready provider has been selected. Bound them, add jitter, honor rate-limit guidance, and carry the same idempotent request ID through every attempt. A confirmed unavailable state never enters that loop. This prevents an outage-shaped queue from consuming all worker slots while useful ingestion work waits behind it.

Capacity planning does not disappear with a managed service. Track incoming audio minutes per minute, completion throughput, the age of the oldest accepted item, and regional headroom. For self-hosted Whisper, measure concurrency and recovery using the exact model and hardware intended for production; an open-source repository is not a capacity promise.

There is a second boundary after transcription. The private knowledge base will likely chunk and embed transcript text before retrieval, but a working chat or embeddings capability is not evidence that speech input is supported. Gate each capability independently. Shared credentials and interfaces can reduce integration work, yet readiness remains specific to the operation being admitted.

## Price the false positive before setting the page

An aggressive threshold can trigger unnecessary failover, move regulated audio into another processing boundary, consume reserved regional capacity, and wake a responder who has no useful action. A loose threshold accepts recordings into a path that cannot complete. Set the observation window from the catalog polling interval, available queue headroom, and the maximum searchable delay the business has approved rather than copying a universal number from a runbook.

The alert should fire when accepted recordings are at risk of breaching the ingestion SLO. Readiness automation should act earlier without paging: close admission, select an approved adapter, and record the transition. Exercise that path, including return to the primary, because an untested fallback is inventory rather than resilience.

The practical answer is blunt: **do not send production audio to a capability whose current catalog state is unavailable**. Re-evaluate after the state becomes available, then repeat the regional, quality, capacity, and failure-injection checks before admitting traffic. The false-positive cost is real, but the threshold must never erase the difference between unknown and explicitly unavailable.

## Further reading

- [OpenAI Whisper repository](https://github.com/openai/whisper)
- [Deepgram speech-to-text documentation](https://developers.deepgram.com/docs/speech-to-text)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
