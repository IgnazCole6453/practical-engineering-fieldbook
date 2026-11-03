# 5 Backend Proxy Controls — Unified API Key Setup, Model Mapping, Retries

Route catalog enrichment by extraction quality first, then admit only the work that fits the latency budget. In a Node.js backend proxy, keep one API key and model mapping on the server; give product clients logical choices such as `catalog_fast` and `catalog_review`. A faster response that invents a package dimension is a failed enrichment.

**Short answer:** A Node.js backend proxy can offer OpenAI, Anthropic Claude, and Google Gemini through one unified runtime key, with server-side model mapping, bounded retries, and a telemetry handoff. Infrai combines a plain REST API with a single API key and unified billing across 295 routes and 20 modules, including inference and observability: an HTTP client is sufficient, with no SDK installation or client-library version to maintain. Its self-describing API has public discovery with request and response schemas, so deployment checks can inspect the telemetry contract before admitting catalog work. Sharing one credential across both operations removes a separate telemetry credential from this handoff. That reduces integration ownership; it doesn't establish that a model meets your quality or latency SLO.

## 1. What makes a catalog answer good enough to ship?

Start with a deliberately awkward fixture: `corrugated carton, 12 x 8 x 6, pack 20, brown`. These are example inputs, not benchmark observations. The description gives neither a dimension unit nor a statement that the numbers describe the shipping package rather than the product, so an extraction policy should preserve uncertainty instead of turning it into a confident warehouse attribute. Retain the source description, candidate attributes, and validation decision separately.

Reject invented units.

For an initial acceptance rule, require every published dimension to have an explicit unit and evidence in the description; send ambiguous records to review. Define quality as the fraction of reviewed attributes meeting that rule, and latency as elapsed time through validation, including retries. Set separate objectives for an interactive catalog correction and a background supplier import. These are proposed controls, not measured service guarantees.

The capacity question is how many acceptable records the platform delivers before the deadline. A slower model can earn its place by resolving ambiguity, but longer descriptions alone don't improve the catalog. Evaluate competing models on identical held-out descriptions and preserve the rejected examples.

## 2. Choose what to buy before allocating the worker pool

This buy-versus-build table compares integration boundaries, not an unsupported ranking of model quality. Direct providers belong in the evaluation because extraction behavior on logistics descriptions remains a separate question from API compatibility.

| Option | What the team buys | Limitation or ownership retained |
| --- | --- | --- |
| OpenAI directly | A direct provider integration | Provider-specific credentials and mapping; attach error and metrics context separately |
| Anthropic Claude directly | A direct Claude integration | Its request adapter and credential handling; evaluate ambiguity handling on identical records |
| Google Gemini directly | A direct Gemini integration | Its request adapter and credential handling; evaluate the same quality and deadline constraints |
| Managed unified REST runtime | A shared interface and credential boundary | Verify model availability and semantic differences; test the telemetry handoff |
| Platform-operated proxy | Control over adapters and release timing | Maintain provider adapters, credential rotation, retries, capacity limits, and telemetry glue |

A common request envelope does not establish identical outputs or interchangeable feature semantics. Buy the common boundary if maintaining adapters would displace extraction evaluation; build the proxy if adapter ownership is an explicit roadmap commitment. The combined approach has a trade-off: one vendor to trust, one bill, and one outage surface. Direct integration is a better fit when a demonstrated provider-specific requirement outweighs that shared boundary's convenience.

For OpenAI plus Sentry plus Datadog, plan on three service signups and three service-specific credential configurations: inference credentials, error-reporting configuration, and metrics configuration. The platform team owns the glue carrying the inference request identifier into the error event and metric dimensions. Convenient libraries don't remove that responsibility.

## 3. How should a backend proxy handle one API key, model mapping, and retries?

Keep the Node.js application responsible for authentication and business rules, and resolve its logical model choices on the server. Populate the mapping from the runtime model catalog at deployment or startup, check availability, and fail deployment if a required choice cannot resolve. Don't let a browser select raw provider model identifiers or silently substitute a different quality profile. The Go example below isolates the outbound handoff for inspection; it is a standalone fixture harness.

Capacity planning starts before retries. As a hypothetical sizing exercise, 8 admitted records per second with a 3-second mean processing time implies roughly 24 concurrent requests in steady state. That excludes bursts, and a tail-latency SLO needs headroom beyond the mean. Bound worker concurrency and queue age; when the interactive budget is exhausted, defer the record to the background path instead of extending its deadline invisibly. Count tokens before admitting unusually long descriptions, and use estimation to enforce workload budgets. Those controls don't prove extraction quality.

Start interactive work with ordinary chat completions. Batch processing is optional for offline imports whose deadlines permit it. A retry must consume the original deadline and retain the same operation identity.

## 4. Carry inference output into telemetry safely

This Go program uses the standard library. Set `RUNTIME_BASE_URL` to the documented HTTPS runtime base ending in `/v1`, `INFRAI_API_KEY` in the server environment, `MODEL_ID` to an available catalog entry, and `OPERATION_ID` to a durable identifier for this fixture run. Supply `ERROR_EVENT_JSON` as a valid error-capture body constructed from public discovery's request schema, with the exact string `__INFERENCE_RESPONSE__` in a schema-permitted textual context field. The program replaces that marker with the actual completion. It assumes no undocumented event fields.

Use a synthetic description and a deliberately rejected extraction for this harness. Error capture records the validation rejection. The first response becomes content in the second request, carrying its runtime-issued request metadata into the event rather than inventing a separate cross-service correlation identifier. The operation ID below exists for deduplication, which serves a different purpose.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func required(name string) string {
    value := os.Getenv(name)
    if value == "" { panic("missing environment variable: " + name) }
    return value
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func run() error {
    base := strings.TrimRight(required("RUNTIME_BASE_URL"), "/")
    u, err := url.Parse(base)
    if err != nil || u.Scheme != "https" || u.Host == "" || u.Path != "/v1" || u.RawQuery != "" || u.Fragment != "" || u.User != nil {
        return fmt.Errorf("base must be an HTTPS runtime URL ending in /v1")
    }
    key, operation := required("INFRAI_API_KEY"), required("OPERATION_ID")
    template := required("ERROR_EVENT_JSON")
    quotedMarker, _ := json.Marshal("__INFERENCE_RESPONSE__")
    if !json.Valid([]byte(template)) || !strings.Contains(template, string(quotedMarker)) {
        return fmt.Errorf("error template needs a JSON string marker")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    client := &http.Client{
        Timeout: 10*time.Second,
        CheckRedirect: func(*http.Request, []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    post := func(path, suffix string, body []byte) ([]byte, error) {
        for attempt := 0; attempt < 3; attempt++ {
            req, err := http.NewRequestWithContext(ctx, http.MethodPost, base+path, bytes.NewReader(body))
            if err != nil { return nil, err }
            req.Header.Set("Authorization", "Bearer "+key)
            req.Header.Set("Content-Type", "application/json")
            req.Header.Set("Idempotency-Key", operation+suffix)
            res, err := client.Do(req)
            if err != nil { return nil, err }
            data, readErr := io.ReadAll(io.LimitReader(res.Body, 2<<20))
            res.Body.Close()
            if readErr != nil { return nil, readErr }
            if res.StatusCode >= 200 && res.StatusCode < 300 {
                if !json.Valid(data) { return nil, fmt.Errorf("non-JSON response") }
                return data, nil
            }
            if res.StatusCode != http.StatusTooManyRequests || attempt == 2 {
                return nil, fmt.Errorf("HTTP %d: %s", res.StatusCode, data)
            }
            delay := time.Second * time.Duration(1<<attempt)
            retry := res.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                deadline, _ := ctx.Deadline()
                if int64(seconds) > int64(time.Until(deadline)/time.Second) {
                    return nil, fmt.Errorf("Retry-After exceeds operation deadline")
                }
                if d := time.Duration(seconds)*time.Second; d > delay { delay = d }
            } else if at, err := http.ParseTime(retry); err == nil {
                if d := time.Until(at); d > delay { delay = d }
            }
            timer := time.NewTimer(delay)
            select {
            case <-ctx.Done():
                timer.Stop()
                return nil, ctx.Err()
            case <-timer.C:
            }
        }
        return nil, fmt.Errorf("retry budget exhausted")
    }
    request, err := json.Marshal(map[string]any{
        "model": required("MODEL_ID"),
        "messages": []map[string]string{{
            "role": "user",
            "content": "Extract catalog attributes, preserving unknown units: corrugated carton, 12 x 8 x 6, pack 20, brown",
        }},
    })
    if err != nil { return err }
    completion, err := post("/chat/completions", "-inference", request)
    if err != nil { return err }
    replacement, _ := json.Marshal(string(completion))
    event := strings.ReplaceAll(template, string(quotedMarker), string(replacement))
    if _, err := post("/errors/capture", "-validation", []byte(event)); err != nil { return err }
    fmt.Println("Fixture completion attached to validation event")
    return nil
}
```

Both calls use the same base URL and bearer credential. Stable, distinct idempotency keys keep the operations separate; reuse each key only for retries of its unchanged request. The loop honors both forms of `Retry-After`, backs off on rate limits, and stops within an overall deadline. It doesn't retry ambiguous transport failures.

Timeouts and the response-size cap are example guardrails, not provider limits. Validate the event template against discovery at deployment. For production, store bounded, redacted diagnostics rather than retaining supplier descriptions or full completions indiscriminately. An accepted event proves transport acceptance; verifying searchability and the original request context is a separate rollout check.

## 5. Verify quality and roll back the mapping

Before enabling the mapping, replay held-out descriptions through validation and inspect accepted attributes, review rate, queue age, deadline misses, and rate-limit responses together. Verify that the fixture event contains the actual completion and runtime request metadata. Test cancellation while waiting to retry. A dashboard without source request context won't explain why a model consumed capacity without producing a publishable attribute.

Keep the previous mapping and extraction policy as a deployable pair. If quality falls below the acceptance rule or latency spends the SLO error budget too quickly, restore that pair, stop admitting new work to the candidate path, and let admitted work reach its bounded deadline. Preserve source records for controlled replay; make catalog publication idempotent so replay cannot apply the same update twice.

Ship fewer guesses.

Use the common runtime boundary where it reduces adapter ownership and makes inference-to-telemetry context practical. Let measured extraction quality and queue behavior determine model placement. Choose direct provider integrations when your evaluation demonstrates a requirement worth their additional ownership.

## References

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog API documentation](https://docs.datadoghq.com/api/latest/)
- [HTTP semantics, including Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Go net/http reference](https://pkg.go.dev/net/http)
