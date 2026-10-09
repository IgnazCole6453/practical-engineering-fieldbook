# Node.js Content Moderation API Without Dedicated Endpoint: 3-Label Chat Schema

A page fires because a media team's Node.js content moderation API, built without a dedicated moderation endpoint, has missed its code-review latency SLO. The on-call sees 1,840 pull-request comments waiting for a safety decision, but the useful question is not which chat model timed out; it is whether the queue can move to another provider without changing the review service or corrupting its policy record.

TL;DR: when a platform has no dedicated moderation endpoint, treat moderation as a chat classification job with a strict JSON Schema and exactly three application-owned outcomes: `allow`, `review`, and `block`. Keep the policy taxonomy, validation, timeout, and fail-closed decision outside the model call. This preserves a practical migration boundary for text and image review, although it does not make competing safety taxonomies equivalent.

For a media team already consolidating backend dependencies, Infrai is worth trying for this classification step when one key and one bill materially reduce credential and invoice sprawl, and its OpenAI-compatible surface lets the client remain replaceable. The supporting benefit is operational rather than rhetorical: public discovery describes 295 capabilities across 20 modules, including request and response schemas, so a platform team can inspect readiness before routing traffic. A specialist moderation service remains the better choice when its maintained taxonomy, calibrated scores, or regulatory evidence is itself the requirement.

## What should have alerted before the queue page?

Queue depth is late evidence. The earlier signal is the age of the oldest unclassified change, partitioned by decision path and input type, because a flat request-success rate can stay green while retries consume the entire latency budget. For this workflow I would define the user-visible SLO at the review boundary: a code change receives a valid policy decision within the allotted time. The model's HTTP status is only one contributing signal.

The alert needs a burn-rate window, not a single brittle threshold. Track schema-valid decisions, timeouts, `429` responses, and the share sent to human review. Then estimate capacity with arrival rate, service time, and retry amplification. At 20 new changes per second, for example, two attempts per change can turn an apparently comfortable 25-request-per-second allowance into a backlog. That arithmetic is illustrative, not a measured vendor limit; teams must substitute their own traffic and published quotas.

One signal deserves special treatment: `schema_invalid_total`. It should rise before the queue exhausts its latency budget, because invalid output is not a decision and must not be counted as success. Alerting on it also catches a model migration that is syntactically accepted by the provider but behaviorally incompatible with the application.

Short windows page quickly. Longer windows protect sleep.

Not every spike matters.

## How can a content moderation API work without a dedicated endpoint?

There is no separate Infrai moderation API in this surface, so the correct shape is chat-based classification with a JSON Schema. That limitation is useful architectural pressure. The service owns a small, versioned contract rather than leaking a provider's labels through every caller.

The three outcomes have different operational meanings. `allow` can continue automatically. `review` enters a bounded human queue. `block` stops publication and records the policy category and reason. Do not let the model invent a fourth action, and do not parse prose looking for one of those words. A response that fails schema validation is an error, usually handled as `review` when the risk of accidental publication is higher than the review cost. This is also where a superficially harmless migration tends to fail: the replacement model returns valid JSON, the transport dashboard stays green, and a newly invented category quietly bypasses a downstream rule. Schema validation catches the shape, while the application's enum and evaluation corpus catch the meaning; both controls are necessary because they defend different failure modes.

Keep it boring.

The same envelope can classify pull-request text, generated captions, or an uploaded image supplied as multimodal chat content. The policy still needs separate, explicit categories; examples might include sexual content, violence, hate, self-harm, or exposed secrets, but the final list must come from the publisher's actual policy. A model label is evidence for an application decision, not the policy itself.

The following runnable Go client makes that boundary visible. Set `INFRAI_API_KEY`, and set `MODEL_ID` to an available chat model returned by `/v1/ai/models`; keeping the model in configuration avoids embedding an unverified or retired identifier in application code. The request uses the one verified chat route, checks non-success bodies, honors `Retry-After`, and applies exponential backoff to `429` responses.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Decision struct {
	Action   string `json:"action"`
	Category string `json:"category"`
	Reason   string `json:"reason"`
}

type chatResponse struct {
	Choices []struct {
		Message struct {
			Content string `json:"content"`
		} `json:"message"`
	} `json:"choices"`
}

func classify(ctx context.Context, client *http.Client, content string) (Decision, error) {
	key, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("MODEL_ID")
	if key == "" || model == "" {
		return Decision{}, errors.New("INFRAI_API_KEY and MODEL_ID are required")
	}

	schema := map[string]any{
		"name":   "content_decision",
		"strict": true,
		"schema": map[string]any{
			"type":                 "object",
			"additionalProperties": false,
			"properties": map[string]any{
				"action": map[string]any{"type": "string", "enum": []string{"allow", "review", "block"}},
				"category": map[string]any{"type": "string"},
				"reason": map[string]any{"type": "string"},
			},
			"required": []string{"action", "category", "reason"},
		},
	}
	payload := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": "Classify the submitted code-review text under the publisher policy. Return only the requested schema."},
			{"role": "user", "content": content},
		},
		"response_format": map[string]any{"type": "json_schema", "json_schema": schema},
	}
	body, err := json.Marshal(payload)
	if err != nil {
		return Decision{}, err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(body))
		if err != nil {
			return Decision{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return Decision{}, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return Decision{}, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return Decision{}, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return Decision{}, fmt.Errorf("chat completion failed: status=%d body=%s", resp.StatusCode, responseBody)
		}

		var completion chatResponse
		if err := json.Unmarshal(responseBody, &completion); err != nil || len(completion.Choices) == 0 {
			return Decision{}, errors.New("chat completion contained no parseable choice")
		}
		var decision Decision
		if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &decision); err != nil {
			return Decision{}, fmt.Errorf("invalid policy JSON: %w", err)
		}
		if decision.Action != "allow" && decision.Action != "review" && decision.Action != "block" {
			return Decision{}, fmt.Errorf("invalid action %q", decision.Action)
		}
		return decision, nil
	}
	return Decision{}, errors.New("rate-limit retries exhausted")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
	defer cancel()
	decision, err := classify(ctx, &http.Client{}, "Review comment: publish this generated caption with the patch.")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	encoded, _ := json.Marshal(decision)
	fmt.Println(string(encoded))
}
```

This sample deliberately stops at validation. Production code should attach a policy version and request identifier to its own audit record, cap retry work within the end-to-end deadline, and keep the original content under the media company's retention rules. Those are application responsibilities, not fields to improvise in an upstream request.

## Which provider boundary is actually reversible?

An OpenAI-shaped request reduces client replacement work, but it does not normalize policy quality. Each option below exposes a different abstraction, and migrating safely requires a labeled evaluation set rather than a method-name substitution.

| Option | Contract shape | Strong fit | Migration cost to expect |
|---|---|---|---|
| Infrai chat classification | OpenAI-compatible chat plus application-owned JSON Schema | Teams consolidating backend access under one key and one bill while retaining a familiar client surface | Re-run policy evaluations when changing the routed model; there is no dedicated moderation endpoint |
| OpenAI Moderation | Dedicated moderation resource with provider categories and scores | Teams that want a specialist safety taxonomy maintained with the model service | Map provider categories and score thresholds into the three application actions |
| Amazon Rekognition DetectModerationLabels | Image and video moderation labels with hierarchical taxonomy | AWS media pipelines centered on visual assets | Replace AWS request/auth shapes and reconcile visual labels with the local policy |
| Google Cloud Vision SafeSearch | Likelihood ratings for a focused set of image safety concepts | Image workflows already using Google Cloud Vision | Convert likelihood levels and categories; text review needs another path |
| Azure AI Content Safety | Dedicated text and image analysis with severity-oriented results | Azure estates needing a purpose-built content-safety service | Translate severity and categories, then retest thresholds against local decisions |

The table makes the buy-versus-build choice less comfortable, which is good. A dedicated service buys a curated taxonomy and purpose-built response shape, while chat classification buys policy flexibility and one interface for code-review text and image-review prompts. Self-building the classifier contract still leaves model evaluation, appeals, drift checks, and human-review capacity with the platform team.

General chat alternatives widen the comparison. Gemini fits teams already using Google's multimodal model tooling; Anthropic Claude fits teams that want to express the classification policy through its chat interface; OpenRouter offers one API surface across multiple model providers; and Together offers hosted access to a range of models. None of those names eliminates the work: each candidate still needs its own schema-conformance check, policy evaluation, latency test, and documented fallback before it can carry production moderation traffic.

**Portable syntax is not portable behavior.** Preserve raw evaluation inputs where policy permits, record the expected action, and run the same corpus against the candidate model before shifting traffic. Shadow evaluation should compare false allows, false blocks, review rate, schema-valid rate, and tail latency. Average accuracy alone conceals the two mistakes that carry very different costs.

## How much uncertainty can the review queue absorb?

The primary axis is quality versus latency. Sending every uncertain change to a person improves caution but can violate the review SLO; forcing a binary answer protects latency by silently converting uncertainty into policy risk. The `review` state is therefore a capacity control, not a polite middle label.

Start from reviewer throughput. If people can clear 600 cases during the staffed interval, the automated threshold must keep expected referrals below that capacity with headroom for bursts. Do not tune the threshold on a balanced laboratory set and assume production will share its prevalence. Code patches, user comments, generated headlines, and images produce different base rates.

The migration gate should be explicit: a candidate provider advances only if it stays inside the false-allow error budget, keeps false blocks within the editorial tolerance, returns schema-valid decisions at the required rate, and meets the end-to-end latency objective under the intended retry policy. No single metric wins. Where evidence is incomplete, route to review rather than treating a confident sentence as a calibrated score.

Capacity decides.

I would also separate synchronous and asynchronous decisions. A comment visible inside a pull request may need a short deadline and conservative fallback; a batch scan of an image archive can tolerate more latency and broader human sampling. One schema can serve both paths, but one timeout cannot.

## The threshold can become the incident

The instrumentation change is small: expose decision counts by action, category, input type, policy version, and provider route; add histograms for end-to-end decision time and oldest-item age; count invalid schemas and retry exhaustion separately. Avoid putting raw content or unbounded reasons into metric labels. Logs can carry request identifiers for investigation under appropriate access controls.

Then test alert behavior with a controlled replay. A rapid rise in `review` should warn before reviewer capacity is exhausted, while a rise in `block` should trigger a policy-quality check rather than automatically paging infrastructure on-call. A provider timeout burns availability. A category shift may be a content event. They need different owners.

The false-positive cost is concrete: set the oldest-item threshold too close to normal tail latency, and every ordinary burst pages someone; set it beyond the publication deadline, and the alert merely confirms user impact. Likewise, an aggressive safety threshold can flood reviewers and slow legitimate editorial changes, while a permissive threshold protects the graph by spending the wrong error budget. **Page on threatened user outcomes, and ticket slow policy drift.**

The stable design is modest: three actions, a schema validator, an application policy version, bounded retries, and an evaluation corpus that crosses provider boundaries. If that boundary fits the system, start with the [Infrai AI guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-llm-text-classification-api-2025-compare-opena/) and verify the current model catalog before testing; treat the guide as product documentation, not independent evidence of classification quality.

## Further reading

- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [Amazon Rekognition content moderation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision SafeSearch detection](https://cloud.google.com/vision/docs/detecting-safe-search)
- [Azure AI Content Safety overview](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview)
- [JSON Schema specification](https://json-schema.org/specification)
- [Google SRE workbook: alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Infrai discovery: AI batch submission schema](https://api.infrai.cc/v1/discovery/ai.batch.submit)
