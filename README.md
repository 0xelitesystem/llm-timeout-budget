# llm-timeout-budget

Model your worst-case LLM request end to end, check it against every timeout between your code and the API, and find out whether streaming saves you or does nothing at all.

## Live demo

https://0xelitesystem.github.io/llm-timeout-budget/

## Features

- **The three-kind timeout taxonomy**, which is the core idea. Every layer between your process and the provider is one of three things, and the difference decides everything:
  - **Idle** - reset by every byte received. Streaming defeats this kind, and only this kind.
  - **Total** - wall clock from the start of the request, reset by nothing. Streaming buys exactly zero against it.
  - **Execution** - counts compute or invocation time and ignores the distinction entirely. Streaming is irrelevant.

  A three-column comparison makes it visible that "just use streaming" is the wrong answer in two of the three rows.
- **A request model** built from figures you enter: input tokens, visible output tokens, reasoning tokens, observed time to first token, observed output tokens per second, and a queue allowance, with p50 / p95 / worst-case multipliers you control. The arithmetic is shown line by line and rendered as a segmented timeline.
- **An editable timeout stack.** Add, remove, reorder or edit any layer. Each row carries a kind, a value, a unit, a note on where it is configured, and a flag for whether a retry loop above it makes its budget cumulative.
- **A per-row verdict** of PASS, FAIL, FAIL UNLESS STREAMING or UNKNOWN, evaluated twice - once assuming a non-streaming request and once assuming a streaming one - plus a headline naming which layer kills the request first, at what elapsed second, and what the observable symptom is at that layer.
- **A heartbeat sub-check.** For idle rows, if the modelled time to first token alone exceeds the limit, the request dies before a single content byte arrives even with streaming enabled.
- **Retry multiplication.** A second timeline stacks the full multi-attempt budget against the layers whose budgets accumulate, with the crossing point marked and the attempt number called out.
- **A unit converter and a unit-mismatch warning** for the single most common instance of that bug.
- **Copyable output**: a plain-language verdict paragraph for a ticket, a markdown summary of the whole stack, and a what-to-change list ordered by effect.
- Dark by default with a persisted light/dark toggle, responsive to 360px, keyboard accessible, no external dependencies.

## How it works

Everything runs in your browser. Open `index.html` and it works; there is no build step, no package manifest and no network traffic.

The core is a set of pure functions that take plain values and return plain result objects - `computeRequestModel`, `evaluateLayer`, `evaluateStack`, `effectiveDeadline`, `rankRemediations`, `buildVerdictText`, `buildMarkdown`. The rendering layer only reads their output.

The duration model is `time to first token + (tokens / tokens per second) + queue allowance`, with each of the three observed figures scaled by a multiplier you supply. Reasoning tokens are handled explicitly: you tell the tool whether the thinking window sits inside your observed time to first token, is added on top of it, or streams as it is produced, so nothing is counted twice.

Each layer is then evaluated against the modelled duration in both modes. A non-streaming request delivers nothing until the end, so every kind fails once the duration passes the limit. A streaming request resets an idle timer on every byte, so an idle row fails only when the silent window itself is longer than the limit - which is exactly what the heartbeat sub-check reports.

### This tool measures nothing

Time-to-first-token and tokens-per-second are user-entered, from your own observation of your own route. The tool sends no requests, needs no API key, and has no backend. Every prefilled number is labelled `illustrative placeholder - replace with your own measurement`.

There are deliberately no benchmark tables here: no per-model tokens-per-second figures, no latency comparison between providers or models, and no claim about which model is faster. Publishing throughput numbers the author has not measured would be fabricated benchmarking, so this tool does not do it.

Every infrastructure default is an editable field pre-filled with a value labelled `commonly seen default - confirm in your own config`, with a link to that vendor's documentation. All of them are configurable and vary by plan tier. The dated vendor reference table is kept visually separate from the tool's own arithmetic, and the arithmetic never reads it, so a stale row there can never make correct maths look wrong. The tool computes only against the numbers in your stack table.

The output is labelled a planning estimate, not a guarantee. Blank or unknown stack values render as UNKNOWN, never as PASS.

## The four client-library traps

These are properties of the libraries themselves rather than claims about any vendor's infrastructure, which is why each carries the package and version it was read in. Library behaviour changes, so re-check them against the version you have installed.

**(a) The timeout you set is per chunk, not a deadline.** Observed in `requests` 2.34.2 and `httpx` 0.28.1, confirmed 2026-08-07. The requests documentation defines its read timeout as "the number of seconds that the client will wait *between* bytes sent from the server" and states that "neither the connect nor read timeouts are wall clock". The httpx documentation defines its read timeout as "the maximum duration to wait for a chunk of data to be received". Neither library documents a total-wall-clock option at all. Fix: track a monotonic clock at the loop level and bail explicitly, or wrap the call in an async timeout. The page ships a copyable snippet of the loop-level deadline pattern.

**(b) The unit is not the same in every language.** Observed in `anthropic` 0.121.0 (Python), `@anthropic-ai/sdk` 0.116.0 (TypeScript), `anthropic` 1.61.0 (Ruby gem), `anthropic-sdk-go` v1.62.0, `com.anthropic:anthropic-java` 2.52.0 and `Anthropic` 12.40.0 (NuGet), confirmed 2026-08-07. All document a 10 minute default - the Go docs scope theirs to non-streaming Messages requests and give other requests no default timeout at all - and each expresses it differently: Python and Ruby take seconds, TypeScript takes milliseconds, Go takes a `time.Duration` (and it is a per-retry timeout), Java takes a `Duration`, C# a `TimeSpan`. A value copied between languages is off by a factor of a thousand.

**(c) The SDK quietly moves its own default.** Observed in `@anthropic-ai/sdk` 0.116.0, `com.anthropic:anthropic-java` 2.52.0 and `anthropic` 0.121.0 (Python), confirmed 2026-08-07. The TypeScript SDK documents that for a large `max_tokens` on a non-streaming request the default is computed as `(60 * 60 * maxTokens) / 128000` seconds, floored at ten minutes and reaching up to sixty. The Java SDK documents the mirror image for streaming requests, with its non-streaming default scaling from a thirty second minimum to a ten minute maximum. The Python SDK refuses rather than hangs: it throws a `ValueError` if a non-streaming request is expected to take longer than approximately ten minutes.

**(d) Timeouts are retried, so multiply.** Observed in all six SDK versions listed above, confirmed 2026-08-07. Each documents that connection errors, 408, 409, 429 and 5xx are retried automatically, two times by default, and the Python, TypeScript and Ruby docs state explicitly that requests which time out are among them. Worst-case wall clock is therefore the timeout times three, and that product is usually what crosses the infrastructure ceiling.

## Sample walkthrough

Press **Load sample** for a synthetic long agentic coding request - 45,000 input tokens, 8,000 visible output tokens, 12,000 reasoning tokens, 95 s time to first token, 22 tokens per second, worst-case multipliers - modelling out to roughly 12 minutes. Three faults are planted in the stack and they fail in sequence:

1. **The SDK client timeout reads `600` in a milliseconds field.** The request dies at 0.6 seconds. The intent was obviously 600 seconds, and the unit warning says so.
2. **A reverse-proxy idle timeout of 60 seconds.** It fails a non-streaming request outright, passes a streaming one in principle, and still fails the heartbeat sub-check, because the 190 s silent window is longer than 60 s. Streaming alone is not enough when the model thinks silently.
3. **A serverless total execution ceiling of 15 minutes.** It passes comfortably on a single attempt and fails once retry multiplication is applied: 3 attempts of about 12 minutes is about 36 minutes.

Fixing them in order walks the whole lesson, and the retry step makes the point that the last surviving layer is the one that gets you. The sample also surfaces a fourth finding on the way through: even the 600 **seconds** the developer meant is shorter than the 12 minute request.

The sample is labelled synthetic. It is a plausible scenario with deliberate faults, not a measurement of anything.

## Sources

Every external fact on the page was fetched from primary documentation and confirmed on **2026-08-07**. Package versions were read from PyPI, npm, RubyGems, the Go module proxy and NuGet on the same date.

Library behaviour:

- requests timeout semantics - https://requests.readthedocs.io/en/latest/user/advanced/#timeouts
- httpx timeout semantics - https://www.python-httpx.org/advanced/timeouts/
- Anthropic Python SDK - https://platform.claude.com/docs/en/api/sdks/python
- Anthropic TypeScript SDK - https://platform.claude.com/docs/en/api/sdks/typescript
- Anthropic Ruby SDK - https://platform.claude.com/docs/en/api/sdks/ruby
- Anthropic Go SDK - https://platform.claude.com/docs/en/api/sdks/go
- Anthropic Java SDK - https://platform.claude.com/docs/en/api/sdks/java
- Anthropic C# SDK - https://platform.claude.com/docs/en/api/sdks/csharp

Infrastructure defaults, all editable in the tool and all configurable in reality:

- nginx `proxy_read_timeout` (default `60s`) - https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout
- nginx `proxy_buffering` (default `on`) - https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering
- AWS Application Load Balancer `idle_timeout.timeout_seconds` (default 60 seconds) - https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html
- AWS Lambda function timeout quota (900 seconds) - https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html
- Cloudflare error 524 Proxy Read Timeout (125 seconds; Enterprise can raise it to 6000) - https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/

A note on that last one, because it is the trap the taxonomy is for. "Proxy Read Timeout" reads like an idle timer, but Cloudflare's page does not state whether the countdown restarts when the origin sends bytes, and it does say that requests exceeding 125 seconds, "for example, streaming", need the value raised. So the tool does not label it idle, and the edge layer defaults to total. When a vendor names a timeout without documenting its reset behaviour, the safe assumption is the one that does not depend on streaming saving you.

On heartbeats: server-sent events allow comment frames, and a stream that emits them keeps an idle timer alive during a long silent window. Whether a given provider emits them, and whether every proxy in the path forwards them rather than buffering them away, is for you to confirm for your own stack. This tool makes no claim either way about any particular provider.

## Related tools

- [retry-backoff-calculator](https://0xelitesystem.github.io/retry-backoff-calculator/) - compute the delay between attempts there and bring the worst-case wait here as the backoff input. This tool deliberately does not re-implement a backoff table.
- [llm-stream-inspector](https://0xelitesystem.github.io/llm-stream-inspector/) - the postmortem sibling. That one diagnoses a stream that already died from its raw event dump; this one is the preflight that tells you which layer would have killed it.
- [idempotency-and-safe-retries-reference](https://github.com/0xelitesystem/idempotency-and-safe-retries-reference) - before you retry a timed-out request that may already have been processed.
- [rate-limit-tester](https://0xelitesystem.github.io/rate-limit-tester/) - for empirically probing a live endpoint, which this tool deliberately does not do.
- [http-status-codes-reference](https://github.com/0xelitesystem/http-status-codes-reference) - for reading the 504 you got.
- [curl-command-builder](https://0xelitesystem.github.io/curl-command-builder/) - for reproducing the request by hand.

## Privacy

Everything runs locally in your browser. The page makes no network requests of any kind: no analytics, no telemetry, no fonts, no external dependencies of any sort. Nothing you type leaves your machine, and nothing is stored anywhere except your light/dark theme preference in `localStorage`. There is no API key field because the tool never calls an API.

## License

MIT. See [LICENSE](LICENSE).

## More

- https://0xelitesystem.github.io/
- https://elitesystem.ai
