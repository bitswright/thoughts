# Raven

> A distributed tracing library written in Go — built from scratch, from a single span all the way to a production-grade, tail-sampling, metrics-correlated observability platform.

Raven is not a wrapper around OpenTelemetry. It is an implementation of the ideas *behind* OpenTelemetry — built ground-up so every design decision is understood, not assumed. The goal is to deeply understand how distributed tracing works: how a unit of work is modelled, how context flows across goroutine and network boundaries, how you make sampling decisions without breaking traces, and how you derive metrics from spans without double-instrumenting your code.

---

## Table of Contents

- [What We Are Building](#what-we-are-building)
- [Core Concepts](#core-concepts)
- [Phases](#phases)
  - [Phase 1 — Span & Trace Data Model](#phase-1--span--trace-data-model)
  - [Phase 2 — Tracer & Span Lifecycle](#phase-2--tracer--span-lifecycle)
  - [Phase 3 — In-Process Context Propagation](#phase-3--in-process-context-propagation)
  - [Phase 4 — TracerProvider & Exporters](#phase-4--tracerprovider--exporters)
  - [Phase 5 — W3C TraceContext Propagation](#phase-5--w3c-tracecontext-propagation)
  - [Phase 6 — Sampling](#phase-6--sampling)
  - [Phase 7 — Batch Processor & OTLP Exporter](#phase-7--batch-processor--otlp-exporter)
  - [Phase 8 — Resource Attributes & Instrumentation Libraries](#phase-8--resource-attributes--instrumentation-libraries)
  - [Phase 9 — Baggage & Correlation IDs](#phase-9--baggage--correlation-ids)
  - [Phase 10 — Tail-Based Sampling](#phase-10--tail-based-sampling)
  - [Phase 11 — Trace–Metrics Correlation (RED Metrics)](#phase-11--tracemetrics-correlation-red-metrics)
  - [Phase 12 — Async & Concurrent Instrumentation](#phase-12--async--concurrent-instrumentation)
  - [Phase 13 — Schema Evolution & Versioning](#phase-13--schema-evolution--versioning)
  - [Phase 14 — Performance & Zero-Alloc Paths](#phase-14--performance--zero-alloc-paths)
- [Checklist](#checklist)

---

## What We Are Building

When a request enters a distributed system, it fans out — it touches a web server, calls a database, fires a message queue job, hits three downstream microservices, and eventually returns. Something goes wrong. The p99 latency spikes. Without tracing, you are blind. You have logs from five services with no way to stitch them together. You have metrics that show *something* is slow but not *where*.

Distributed tracing solves this by attaching a unique `TraceID` to a request at its entry point and propagating it across every service boundary. Each unit of work — a database call, an HTTP request, a function with meaningful latency — becomes a `Span`. All spans that share a `TraceID` form a `Trace`: a tree of causally related work you can visualise as a waterfall diagram.

Raven implements the full stack of this:

- The **data model** — what a span is, what a trace is, how parent-child relationships are encoded
- The **lifecycle API** — how developers start, annotate, and finish spans in their code
- The **context system** — how the active span is threaded through `context.Context` so child spans are created automatically
- The **propagation layer** — how a trace crosses a network via HTTP headers (W3C `traceparent`, B3)
- The **sampling layer** — how you decide which traces to keep without exploding your storage budget
- The **export pipeline** — how finished spans are buffered, batched, and shipped to a backend (Jaeger, Tempo)
- The **metrics bridge** — how you derive Rate/Error/Duration metrics from spans automatically, so one instrumentation call gives you both traces and dashboards

---

## Core Concepts

**Span** — The fundamental unit. Represents a single named operation with a start time, end time, status, and a bag of key-value attributes. Every span belongs to exactly one trace.

**TraceID** — A 128-bit random ID that is the same across every span in a request's journey. This is what lets you find all the work done for a single user request across 10 services.

**SpanID** — A 64-bit random ID unique to one span. Combined with `parentSpanID`, it encodes the tree structure of a trace.

**Parent-child relationship** — When service A calls service B, A's span becomes the parent of B's span. This is how the trace tree is built. The root span has no parent.

**Context propagation** — In Go, the active span lives inside `context.Context`. When you call `tracer.Start(ctx, "name")`, it reads the current span from `ctx` to set as parent, then stores the new span back into a derived context. This is the mechanism that makes nesting automatic.

**Inject / Extract** — When a span crosses a network, its `traceID` and `spanID` are serialised into HTTP headers (e.g. `traceparent: 00-{traceID}-{spanID}-01`). The downstream service deserialises them back into a span context — this is Extract. The upstream putting them into headers is Inject.

**Sampling** — Tracing 100% of traffic is expensive. Sampling is the decision of whether to record and export a trace. Head sampling decides at trace start. Tail sampling buffers the whole trace and decides after completion — allowing you to always keep error traces and slow traces.

**TracerProvider** — The factory that creates `Tracer` instances and holds the shared configuration: exporter chain, sampler, resource attributes. You configure it once at startup; all instrumented code just calls `tracer.Start()`.

**SpanProcessor** — Sits between a finished span and the exporter. The `BatchSpanProcessor` buffers spans in a goroutine-safe queue and flushes them in batches. You can chain multiple processors.

**SpanExporter** — The terminal: takes a batch of finished spans and ships them somewhere — stdout, in-memory (for tests), or a real backend via OTLP.

**Resource** — Immutable metadata about the process that produced spans: `service.name`, `service.version`, `host.name`. Attached to the `TracerProvider` and stamped on every exported span automatically.

**Baggage** — Key-value pairs that propagate across process boundaries alongside the trace context, via the W3C `baggage` header. Used for cross-cutting concerns like `userId` or `tenantId` that you want visible on every span without manually setting attributes everywhere.

**RED Metrics** — Rate, Errors, Duration. The three metrics every service should expose. Raven derives them automatically from span data via a `SpanMetricsProcessor`, so you get dashboards for free from your tracing instrumentation.

**Exemplars** — A mechanism to embed a `traceID` inside a metric data point. This is the link between your Prometheus dashboard and Jaeger: click a latency spike → jump directly to the trace that caused it.

---

## Phases

### Phase 1 — Span & Trace Data Model

The foundation. Everything in Raven is built on top of what we define here. A span is a named, timed operation that belongs to a trace. We define the structs, the ID types, and the status model.

**What we are building:**
- `TraceID` (128-bit) and `SpanID` (64-bit) types generated via `crypto/rand`, serialised as lowercase hex strings
- `Span` struct: `name`, `traceID`, `spanID`, `parentSpanID`, `startTime`, `endTime`, `status`, `attributes`, `events`
- `SpanStatus` enum: `Unset`, `Ok`, `Error` — with `SetStatus(status, description)` method
- `Attributes` — `map[string]string` key-value metadata; each span carries its own copy
- `Event` — a named, timestamped record attached to a span mid-flight (e.g. "cache miss", "retry attempt 2"); stored as `[]Event` on the span
- `RecordError(err)` — convenience method that appends an event with `exception.type` and `exception.message`

**Notes:**
- `TraceID` being 128-bit is a W3C requirement. `SpanID` being 64-bit is a Zipkin/B3 legacy that W3C kept.
- `parentSpanID` being an empty string (not a pointer) is a deliberate choice — it avoids nil checks everywhere and clearly signals "this is a root span".
- Attributes have cardinality implications: a key like `userId` is fine; a key like `request.body` on every span will explode your storage. We will enforce a max attribute count per span.
- Events are the tracing equivalent of structured log lines — they are timestamped and attached to the span they happened within, which is something flat logs cannot express.

---

### Phase 2 — Tracer & Span Lifecycle

Define how code actually creates and finishes spans. The `Tracer` is the object developers interact with daily.

**What we are building:**
- `Span` interface — the public API: `End()`, `SetStatus()`, `SetAttributes()`, `AddEvent()`, `RecordError()`, `SpanContext()`
- `Tracer` interface — `Start(ctx context.Context, name string, opts ...SpanOption) (context.Context, Span)`
- `Span.End()` — records `endTime`, marks span finished; idempotent (second call is a no-op)
- `SpanKind` — `Internal`, `Server`, `Client`, `Producer`, `Consumer`; set via `SpanOption` at start time
- `SpanOption` — functional options pattern: `WithSpanKind(kind)`, `WithAttributes(attrs...)`, `WithTimestamp(t)`

**Notes:**
- `Start` returning `(context.Context, Span)` is the canonical Go pattern. The context is the "updated world" with the new span embedded; callers use `defer span.End()` to ensure cleanup.
- `defer span.End()` is not just a convenience — it is the correct pattern. Without it, a `return` in the middle of a function leaks an open span with no `endTime`, which shows up as infinite-duration spans in Jaeger.
- `SpanKind` is metadata, not logic — at this phase it is just stored on the span. Later, exporters and samplers will use it (e.g. only `Server` spans get HTTP attributes auto-populated).
- Functional options (`WithSpanKind`, etc.) keep the `Start` signature stable as we add more options later — no breaking API changes.

---

### Phase 3 — In-Process Context Propagation

The mechanism that makes parent-child relationships automatic. Spans live inside `context.Context`. When you start a new span from a context that already has a span, the new span automatically becomes a child.

**What we are building:**
- `SpanFromContext(ctx) Span` — retrieves the active span from context; returns a `NoopSpan` if none
- `ContextWithSpan(ctx, span) context.Context` — stores a span into a derived context
- Private context key type (`type contextKey struct{}`) — prevents key collisions with other packages
- `NoopSpan` — implements the full `Span` interface but does nothing; zero allocations; used when tracing is disabled or no span is in context
- Auto parent-child wiring in `Tracer.Start` — reads span from ctx, sets as `parentSpanID` on the new span
- `Span Links` — a span can hold references to spans from *other* traces via `[]Link{traceID, spanID, attrs}`; used for batch jobs that process work from multiple upstream traces

**Notes:**
- The private key type is critical. If you use a plain `string` as a context key, any package can accidentally overwrite your span with `ctx = context.WithValue(ctx, "span", something)`. The private struct type is unexported and unguessable.
- `NoopSpan` is what gets returned when you call `SpanFromContext` on a context with no span. This means all code can call `span.SetAttributes(...)` without nil-checking — it just silently does nothing. This is the "safe default" design.
- Span Links are different from parent-child: they do not create a tree edge. They are a loose association — "this span is related to that other trace" — visible in Jaeger as a dashed line between two otherwise separate trace trees.

---

### Phase 4 — TracerProvider & Exporters

Give spans somewhere to go. Build the registry that holds the shared configuration and two exporters: one for tests, one for development.

**What we are building:**
- `SpanExporter` interface — `ExportSpans(ctx, []Span) error` + `Shutdown(ctx) error`
- `InMemoryExporter` — stores finished spans in a `[]Span`; exposes `GetSpans()` and `Reset()`; the backbone of all unit tests
- `StdoutExporter` — marshals spans to JSON and writes to stdout; colour-codes status (green `OK`, red `ERROR`)
- `TracerProvider` struct — holds: sampler, exporter(s), resource, and a map of named `Tracer` instances
- `TracerProvider.Tracer(name, version string) Tracer` — returns (or creates) a named tracer; name is the instrumentation library name (e.g. `"raven/http"`)
- Global `TracerProvider` — `raven.SetGlobalProvider(p)` / `raven.GlobalProvider()` package-level functions

**Notes:**
- `InMemoryExporter` is the most important exporter for correctness. Every phase from here onwards gets tested by asserting against what lands in the in-memory store: span names, parent IDs, attributes, status codes.
- The `TracerProvider` holding a map of `Tracer` instances by name allows you to see in exported data which library or component produced each span — useful when debugging "which version of my HTTP middleware is this span from?".
- The global provider is a pragmatic compromise. In a perfect world, every function would accept a `TracerProvider` argument. In practice, instrumentation code sits deep in call stacks and passing it through is not feasible. The global is set once at `main()` startup and never changed after that.

---

### Phase 5 — W3C TraceContext Propagation

When a span crosses a network boundary, its identity must travel with the request. We implement the W3C TraceContext specification — the industry standard wire format for distributed traces.

**What we are building:**
- `TextMapCarrier` interface — `Get(key) string`, `Set(key, value)`, `Keys() []string`; any map-like thing (HTTP headers, Kafka message metadata) implements this
- `TextMapPropagator` interface — `Inject(ctx, carrier)`, `Extract(ctx, carrier) context.Context`
- `W3CTraceContextPropagator` — reads/writes the `traceparent` and `tracestate` headers
  - `traceparent` format: `00-{32-hex traceID}-{16-hex spanID}-{8-bit flags}`
  - `tracestate` format: ordered `key=value` pairs, vendor-specific, max 512 bytes
- `B3Propagator` — reads/writes Zipkin-style `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` headers; also supports single-header `b3: {traceID}-{spanID}-{flag}`
- `CompositePropagator` — chains multiple propagators; `Inject` calls all; `Extract` tries each and stops at first success

**Notes:**
- The `traceparent` flags byte currently only uses bit 0: the "sampled" flag. If `01`, the upstream decided to sample this trace. If `00`, it decided not to. The downstream should respect this decision — see Phase 6.
- `tracestate` is where vendors can stash extra routing information without polluting `traceparent`. We preserve any `tracestate` values we do not understand — throwing them away would break other vendors in the chain.
- B3 support exists because a huge amount of existing infrastructure (Zipkin, older Jaeger versions, AWS X-Ray) still uses it. The `CompositePropagator` lets Raven speak both simultaneously during a migration period.
- The `TextMapCarrier` abstraction is what allows the same propagator code to work over HTTP headers, gRPC metadata, Kafka headers, or any other key-value transport.

---

### Phase 6 — Sampling

Sampling is the decision of whether to record and export a given trace. It must be made early (to avoid work), consistently (all spans in a trace must agree), and correctly (broken partial traces are worse than no traces).

**What we are building:**
- `Sampler` interface — `ShouldSample(params SamplingParameters) SamplingResult`
- `SamplingParameters` — carries `traceID`, `spanName`, `spanKind`, `attributes`, `parentSpanContext`
- `SamplingResult` — `Decision` (`Drop` | `RecordOnly` | `RecordAndSample`) + `Attributes` + `TraceState`
- `AlwaysOnSampler` — always returns `RecordAndSample`; for development and low-traffic services
- `AlwaysOffSampler` — always returns `Drop`; useful for disabling tracing via config without code changes
- `TraceIDRatioBased(ratio float64)` — hashes the `traceID` and compares against the ratio; deterministic — the same `traceID` always gets the same decision across all services
- `ParentBased(root Sampler)` — if there is a remote parent, respect its sampling decision (read from the `traceparent` sampled flag); if there is no parent (root span), delegate to the configured `root` sampler
- Remote/dynamic sampler — wraps any `Sampler`; polls a config endpoint on a background goroutine; atomically swaps the active sampler on change

**Notes:**
- `TraceIDRatioBased` being deterministic on `traceID` is essential. If service A samples 10% and service B samples 10% independently using random decisions, a trace that A keeps might be dropped by B — resulting in a partial trace with a missing subtree. By hashing the `traceID`, both services make the same decision for the same trace.
- `ParentBased` is the right default for most production systems. It means: "trust whatever the entry point decided". The entry point (the load balancer or the first microservice) runs `TraceIDRatioBased`; everything downstream defers to it.
- `RecordOnly` is a rarely used middle ground: record the span (for local diagnostic purposes) but do not export it. Useful for internal debugging builds.
- The remote sampler swap must be atomic — you cannot have a goroutine mid-way through `ShouldSample` while another swaps the sampler underneath it. Use `atomic.Value` to store the active sampler.

---

### Phase 7 — Batch Processor & OTLP Exporter

The production export pipeline. Spans cannot be sent one-by-one over the network — the overhead would be enormous. We build a batching layer and implement the OpenTelemetry Protocol (OTLP) to ship spans to real backends.

**What we are building:**
- `SpanProcessor` interface — `OnStart(ctx, span)`, `OnEnd(span)`, `Shutdown(ctx) error`, `ForceFlush(ctx) error`
- `SimpleSpanProcessor` — calls `exporter.ExportSpans` synchronously on every `OnEnd`; only for tests and CLI tools
- `BatchSpanProcessor` — the production processor:
  - Internal goroutine-safe ring buffer / channel queue
  - Flushes when `maxExportBatchSize` is reached or `scheduleDelay` elapses (whichever comes first)
  - Configurable: `maxQueueSize` (default 2048), `maxExportBatchSize` (default 512), `scheduleDelay` (default 5s), `exportTimeout`
  - On queue full: drop the span and increment a `DroppedSpans` counter (exposed as a metric)
- `OTLPHTTPExporter` — serialises spans to protobuf, POSTs to `/v1/traces` on a collector endpoint; retries with exponential backoff on 5xx; configurable headers for auth
- `OTLPGRPCExporter` — streams spans via gRPC to a collector; more efficient than HTTP for high-throughput; manages connection lifecycle (reconnect on failure)
- `TracerProvider.Shutdown(ctx)` — signals `BatchSpanProcessor` to flush remaining spans and stop; respects the context deadline; logs count of dropped spans on timeout

**Notes:**
- The `BatchSpanProcessor` is a producer-consumer problem. The application goroutines are producers (calling `OnEnd`); the background flush goroutine is the consumer. The queue is the buffer between them. The design question is: what happens when the producer is faster than the consumer? We drop, not block — blocking would introduce latency into the application's hot path, which is unacceptable for a tracing library.
- OTLP is the wire format that all modern backends (Jaeger v2, Grafana Tempo, Honeycomb, Datadog) accept. It is protobuf over HTTP or gRPC. By implementing OTLP, Raven can ship to any backend without backend-specific code.
- `ForceFlush` is called by tests and shutdown hooks to ensure all pending spans are exported before the process exits. Without it, the last few seconds of spans from a short-lived process would be silently lost.
- Exponential backoff on the OTLP exporter is important — if the collector is overloaded, hammering it with retries makes things worse. Back off to at most 30s between retries.

---

### Phase 8 — Resource Attributes & Instrumentation Libraries

Spans need to carry identity about the service that produced them. Without this, a span named `"SELECT"` tells you nothing about which of your 40 services ran that query. Resource attributes solve this. Instrumentation libraries eliminate boilerplate.

**What we are building:**
- `Resource` — an immutable `map[string]string` of process-level attributes; created once at startup and attached to the `TracerProvider`
  - Standard keys: `service.name`, `service.version`, `service.instance.id`, `host.name`, `os.type`, `process.pid`
  - Merged automatically into every exported span by the exporter
- OTEL Semantic Conventions — a `semconv` package of constants for standard attribute names (`semconv.HTTPMethod`, `semconv.DBSystem`, `semconv.RPCMethod`, etc.); we follow the published spec so our spans are queryable in standard UIs
- `raven/http` — HTTP server middleware:
  - Wraps `http.Handler`; extracts trace context from incoming headers; creates a `Server` span; sets `http.method`, `http.route`, `http.status_code`, `http.url`; injects updated context into the request
  - Sets span status to `Error` for 5xx; leaves it `Ok` for 4xx (a 404 is not a server error)
- `raven/httpclient` — HTTP client `RoundTripper`:
  - Wraps `http.DefaultTransport`; creates a `Client` span before the request; injects `traceparent` into outgoing headers; records `http.status_code` and duration on response
- `raven/sql` — wraps `database/sql`; creates a span per query; records `db.system`, `db.statement` (with sensitive values masked), `db.operation`

**Notes:**
- Resources are *process-level* — they describe the thing running, not the thing being traced. They are stamped onto every span at export time rather than stored in each span struct, which keeps per-span memory small.
- The distinction between 4xx and 5xx for HTTP server spans matters: a client sending a malformed request (400) is not your server's error. Setting `Error` status on 4xx would pollute your error rate dashboard with client mistakes.
- Masking `db.statement` values is not optional — `INSERT INTO users VALUES ('password123')` must never appear in your tracing backend. The instrumentation should capture the query structure (`INSERT INTO users VALUES (?)`) but not the values.
- `RoundTripper` wrapping is the Go idiom for HTTP client middleware. You create a struct that holds the inner transport and implements `RoundTrip(req) (*Response, error)`. This composes cleanly and does not require modifying call sites.

---

### Phase 9 — Baggage & Correlation IDs

Baggage is a mechanism to propagate arbitrary key-value data across process boundaries alongside the trace context. Unlike span attributes (which are local to one span), baggage values travel with every downstream request for the life of the trace.

**What we are building:**
- `Baggage` type — an immutable map of `Member{key, value, properties}`; created via `NewBaggage(members...)` 
- `BaggageFromContext(ctx) Baggage` and `ContextWithBaggage(ctx, baggage) context.Context`
- W3C Baggage header format — `baggage: userId=abc123,env=production,region=us-east-1`
- `BaggagePropagator` — `Inject` serialises baggage into the `baggage` header; `Extract` parses it back
- Sanitisation — validate key format (no whitespace, no `=` or `;`), max value length (4096 bytes), max total baggage size (8192 bytes)
- `BaggageSpanProcessor` — optional processor that copies all baggage entries into the current span's attributes on `OnStart`; enables seeing `userId` on every span in Jaeger without manually calling `span.SetAttributes` everywhere

**Notes:**
- Baggage travels in *every* downstream HTTP request for the life of the trace. A value of 1KB in baggage × 100 downstream calls = 100KB of extra header data. This is why the size limits are enforced strictly — unbounded baggage is a DoS vector.
- Baggage is propagated by the propagation layer, not by the tracing layer. It piggybacks on the same `TextMapPropagator` chain that propagates `traceparent`. Both are injected/extracted in the same pass over the headers.
- The `BaggageSpanProcessor` is the elegant solution to the "I want `userId` on every span" problem. Set it in baggage at the entry point once; the processor stamps it as an attribute on every span created downstream. No manual `span.SetAttribute("userId", ...)` calls scattered throughout the codebase.

---

### Phase 10 — Tail-Based Sampling

Head sampling (Phase 6) is blind — you decide at trace start, before you know if this trace will be interesting. Tail sampling buffers the entire trace and decides after it is complete. This means you can have a policy like: "keep all traces that had an error or took more than 500ms; sample everything else at 1%".

**What we are building:**
- In-memory trace buffer — `map[TraceID]*traceBuffer`; each buffer holds `[]Span` and a `lastUpdated` timestamp
- `traceBuffer` eviction — a background goroutine sweeps the map every N seconds; evicts buffers that have not received a new span in longer than `traceTimeout` (default 30s); this handles late-arriving spans
- `TailSampler` interface — `Evaluate(trace []Span) SamplingDecision`
- `LatencyPolicy(threshold time.Duration)` — keeps a trace if its root span duration exceeds the threshold
- `ErrorPolicy()` — keeps a trace if any span in the trace has `status=Error`
- `CompositePolicyEvaluator` — AND/OR/NOT combinators: keep if `(ErrorPolicy OR LatencyPolicy(500ms)) AND NOT EndpointPolicy("/healthz")`
- `EndpointPolicy(path string)` — drops traces for known noisy endpoints (health checks, readiness probes)
- Remote tail sampler — a `TailSampler` that sends the trace summary to a central decision service; needed when one trace's spans arrive at multiple collector replicas

**Notes:**
- The fundamental tension in tail sampling is memory. To make a decision on the whole trace, you must keep all spans in memory until the trace is "complete" — but traces do not signal completion. You infer it from silence (no new spans for N seconds). This means you must tune `traceTimeout` carefully: too short and you make decisions on incomplete traces; too long and your memory footprint balloons.
- This phase requires a different architecture from the `BatchSpanProcessor`. That processor is a simple queue — spans flow through in order. The tail sampler is a *stateful aggregator* — spans arrive out of order and must be correlated by `traceID` before a decision is made.
- The remote tail sampler solves a real distributed systems problem: in a horizontally-scaled collector fleet, spans from the same trace land on different collector instances. A local tail sampler on each instance can only see a fragment. The remote sampler centralises the decision. This comes at the cost of an extra network hop per trace.

---

### Phase 11 — Trace–Metrics Correlation (RED Metrics)

The holy grail of observability: automatically derive service-level metrics from span data. One instrumentation call gives you traces for debugging *and* dashboards for monitoring. No double-instrumentation.

**What we are building:**
- `SpanMetricsProcessor` — a `SpanProcessor` that, on `OnEnd`, extracts and records three metrics from each finished span:
  - `raven_requests_total` (counter) — labels: `service`, `span_name`, `status_code`; the Rate in RED
  - `raven_errors_total` (counter) — labels: `service`, `span_name`; incremented only when `status=Error`; the Errors in RED
  - `raven_duration_seconds` (histogram) — labels: `service`, `span_name`; the Duration in RED
- Histogram buckets — configurable; default: `5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s`
- Exemplar injection — on each histogram observation, embed the current `traceID` as an exemplar; this is the link from a Prometheus metric data point to the specific trace that caused it
- `PrometheusExporter` — exposes a `http.Handler` for `/metrics`; uses the `go-prometheus` client library; `SpanMetricsProcessor` registers its metrics here

**Notes:**
- RED stands for Rate, Errors, Duration. These are the three metrics that describe the health of any service that handles requests. They are the minimum viable monitoring signal.
- Exemplars are the feature that makes the metrics-to-traces link possible. In Grafana, when you hover over a latency spike on a Prometheus histogram, if exemplars are present, a "View Trace" button appears. Clicking it opens Jaeger/Tempo at the exact trace that was recorded at that moment. This is the "single pane of glass" that otherwise requires expensive commercial APM tools.
- The `SpanMetricsProcessor` is chained *before* the `BatchSpanProcessor` in the processor pipeline. Both see the same span: one derives metrics, the other exports the span. The application code does not change.

---

### Phase 12 — Async & Concurrent Instrumentation

Real services spawn goroutines, fan out across worker pools, and process messages from queues. Tracing these correctly requires explicit patterns — Go's context system does not automatically propagate across goroutine boundaries.

**What we are building:**
- Goroutine context propagation patterns — documented with examples; the `ctx` passed to `go func(ctx)` must be the current ctx at the time of spawn, not a closure-captured stale ctx
- Fan-out tracing helper — `TraceGoGroup(ctx, tracer, spans []WorkItem, fn func(ctx, item))` — creates a parent span, spawns N goroutines each with their own child span, waits for all with `sync.WaitGroup`, ends the parent span on completion
- Async span completion — `span.End()` is safe to call from a different goroutine than the one that called `Start()`; the span struct uses `sync.Mutex` internally for the end-time write
- `raven/kafka` — Kafka producer/consumer instrumentation:
  - Producer: creates a `Producer` span, calls `Inject` into `msg.Headers`, ends span on ack/error
  - Consumer: calls `Extract` from `msg.Headers`, creates a `Consumer` span as a *child* of the producer span (same trace) — or as a linked span if the consumer processes in a separate logical trace
- `raven/grpc` — gRPC interceptors (unary and streaming) for both server and client; propagates trace context via gRPC metadata

**Notes:**
- The goroutine closure anti-pattern is subtle and common: `go func() { span.SetAttributes(...) }()` looks fine, but if `span` is captured by value from the outer scope and the outer scope already called `span.End()`, the goroutine is writing to a finished span. Always pass `ctx` explicitly to goroutines, never capture spans by closure.
- The Kafka consumer creating a *child* of the producer span (same trace) vs a *linked* span (separate trace) is a design choice. Child = same trace, which gives you the full request-to-job journey in one Jaeger view. Linked = separate traces with a reference, which is better for batch jobs that process many upstream requests — you do not want 10,000 producer spans as parents of one consumer span.

---

### Phase 13 — Schema Evolution & Versioning

Raven will be used by many services simultaneously, deployed at different versions. The attribute names and conventions that spans use will change over time as the OpenTelemetry community evolves the semantic conventions spec. Schema versioning is how we manage this without breaking things.

**What we are building:**
- Schema URL on `Resource` — a URL string that declares which version of the semantic conventions this service uses (e.g. `https://opentelemetry.io/schemas/1.21.0`); stamped on every exported batch
- `SpanTransformer` — a `SpanProcessor` that rewrites attribute keys on outgoing spans to match a target schema version; e.g. `net.peer.ip` → `server.address` when migrating from OTEL semconv 1.17 to 1.21
- Instrumentation scope — every `Tracer` carries a name and version (`"raven/http"`, `"v0.3.1"`); exported in the OTLP payload; visible in Jaeger as "instrumentation library"; lets you identify which library version produced a given span in prod
- `Resource.Merge(other Resource) Resource` — merges two resource sets; later keys win; used to combine auto-detected resource (from env vars) with manually specified resource

**Notes:**
- Schema versioning only matters at scale — when you have 50 services and you upgrade the HTTP semantic conventions (renaming `http.url` to `url.full`), you cannot redeploy all 50 at once. During the rollover period, your tracing backend receives spans with both old and new attribute names. The Schema URL tells the backend which convention a given span follows, so it can query correctly for both.
- The `SpanTransformer` is a stop-gap for controlled migrations. It lets you upgrade the schema at the collector level without redeploying application code — you just add a transformer to the processor chain.

---

### Phase 14 — Performance & Zero-Alloc Paths

A tracing library is in the hot path of every request. If it is slow or allocation-heavy, it will show up in your application's flame graphs and p99 latency. This phase is about proving Raven is fast and fixing the hotspots that matter.

**What we are building:**
- `sync.Pool` for `Span` objects — reuse span structs instead of allocating a new one per `Start` call; `Reset()` the span on retrieval from pool; return it to the pool on `End`
- `sync.Pool` for `Attributes` maps — the `map[string]string` alloc is expensive on the hot path; pool pre-sized maps
- Lock-free MPSC queue for `BatchSpanProcessor` — replace the `sync.Mutex`-guarded slice with an atomic ring buffer (multi-producer, single-consumer); eliminates lock contention under high goroutine counts
- Benchmarks for hot paths — `go test -bench` covering: `Tracer.Start + End`, `SpanFromContext`, `Inject`, `Extract`; use `benchstat` to compare before/after; document baseline numbers in `BENCHMARKS.md`
- `pprof` load generator — a `cmd/loadgen` binary that hammers the tracer at configurable RPS; attach `pprof` to it; identify the real bottleneck (historically: JSON marshal in the stdout exporter, map allocs in attributes)
- Sampled-away span cost — when a trace is dropped by the sampler, `Tracer.Start` must be as cheap as possible; the `NoopSpan` path must have **0 allocs**; verify with `testing.AllocsPerRun`
- Back-pressure metrics — expose `raven_queue_size`, `raven_dropped_spans_total`, `raven_export_duration_seconds` as Prometheus metrics; these are the operational signals for the tracing infrastructure itself

**Notes:**
- The most important performance constraint is the sampled-away path. If 99% of traces are dropped, 99% of `Tracer.Start` calls must be near-free. The `NoopSpan` path should touch nothing except a context read and a sampler call. 0 allocs here is the target.
- `sync.Pool` is not a silver bullet — pooled objects must be fully reset before reuse or you will get data leaks between requests (a span from request A appearing in request B's trace). The `Reset()` method must zero every field.
- `benchstat` is the correct tool for benchmark comparison — it runs benchmarks multiple times, computes statistics, and tells you whether a change is statistically significant. Never compare benchmark numbers from a single run.
- The lock-free queue is an advanced optimisation and should only be attempted after the `sync.Mutex` version is correct and benchmarked. Premature lock-freedom is a source of subtle races.

---

## Checklist

### Phase 1 — Span & Trace Data Model

- [ ] Define `TraceID` type (128-bit, `[16]byte`); implement `NewTraceID()` using `crypto/rand`
  - [ ] Implement `String() string` — lowercase hex, 32 chars
  - [ ] Implement `IsValid() bool` — false if all zeros
- [ ] Define `SpanID` type (64-bit, `[8]byte`); implement `NewSpanID()` using `crypto/rand`
  - [ ] Implement `String() string` — lowercase hex, 16 chars
  - [ ] Implement `IsValid() bool` — false if all zeros
- [ ] Define `SpanContext` struct — `TraceID`, `SpanID`, `TraceFlags`, `TraceState`, `Remote bool`
  - [ ] Implement `IsValid() bool`
  - [ ] Implement `IsSampled() bool` — checks `TraceFlags` bit 0
- [ ] Define `SpanStatus` type and constants: `Unset`, `Ok`, `Error`
- [ ] Define `Attribute` type — `Key string`, `Value string`
- [ ] Define `Event` struct — `Name string`, `Timestamp time.Time`, `Attributes []Attribute`
- [ ] Define `Span` struct (internal, not the interface yet) with all fields
  - [ ] `name`, `spanContext`, `parentSpanID`, `spanKind`
  - [ ] `startTime`, `endTime time.Time`
  - [ ] `status SpanStatus`, `statusDescription string`
  - [ ] `attributes []Attribute`
  - [ ] `events []Event`
  - [ ] `links []Link`
  - [ ] `finished bool`, `mu sync.Mutex`
- [ ] Implement `SetStatus(status SpanStatus, description string)`
- [ ] Implement `SetAttributes(attrs ...Attribute)`
- [ ] Implement `AddEvent(name string, attrs ...Attribute)`
- [ ] Implement `RecordError(err error)` — adds event with `exception.type`, `exception.message`
- [ ] Write unit tests for ID generation, validity checks, attribute setting, event recording

---

### Phase 2 — Tracer & Span Lifecycle

- [ ] Define `Span` interface — public API
  - [ ] `End(opts ...SpanEndOption)`
  - [ ] `SetStatus(status SpanStatus, description string)`
  - [ ] `SetAttributes(attrs ...Attribute)`
  - [ ] `AddEvent(name string, attrs ...Attribute)`
  - [ ] `RecordError(err error)`
  - [ ] `SpanContext() SpanContext`
  - [ ] `IsRecording() bool`
- [ ] Define `SpanKind` type and constants: `Internal`, `Server`, `Client`, `Producer`, `Consumer`
- [ ] Define `SpanStartOption` and `SpanEndOption` functional option types
  - [ ] `WithSpanKind(kind SpanKind) SpanStartOption`
  - [ ] `WithAttributes(attrs ...Attribute) SpanStartOption`
  - [ ] `WithTimestamp(t time.Time) SpanStartOption` / `SpanEndOption`
- [ ] Define `Tracer` interface — `Start(ctx, name string, opts ...SpanStartOption) (context.Context, Span)`
- [ ] Implement `Span.End()` — record `endTime`, set `finished=true`; idempotent
- [ ] Implement `IsRecording()` — true if started and not yet ended
- [ ] Write tests: `defer span.End()` pattern, double-End is safe, nested spans have correct parent IDs

---

### Phase 3 — In-Process Context Propagation

- [ ] Define private `contextKey` type for context storage
- [ ] Implement `ContextWithSpan(ctx context.Context, span Span) context.Context`
- [ ] Implement `SpanFromContext(ctx context.Context) Span` — returns `NoopSpan` if none present
- [ ] Implement `NoopSpan` struct — satisfies `Span` interface; all methods are no-ops
  - [ ] Verify `NoopSpan` methods are inlined by the compiler (no heap allocs)
- [ ] Wire parent-child into `tracer.Start` — extract span from ctx, use its `SpanID` as `parentSpanID`
- [ ] Define `Link` struct — `SpanContext`, `Attributes []Attribute`
- [ ] Implement `WithLinks(links ...Link) SpanStartOption`
- [ ] Write tests: root span has empty parentSpanID, child span has correct parentSpanID, noop span is returned on empty context

---

### Phase 4 — TracerProvider & Exporters

- [ ] Define `SpanExporter` interface — `ExportSpans(ctx, []ReadOnlySpan) error`, `Shutdown(ctx) error`
- [ ] Define `ReadOnlySpan` interface — getter methods for all span fields (prevents mutation after export)
- [ ] Implement `InMemoryExporter`
  - [ ] Thread-safe `GetSpans() []ReadOnlySpan`
  - [ ] `Reset()`
- [ ] Implement `StdoutExporter`
  - [ ] Marshal span to JSON
  - [ ] Colour-code status in terminal output
- [ ] Define `TracerProvider` struct
  - [ ] Holds: `sampler`, `[]SpanProcessor`, `resource`, `map[string]Tracer`
- [ ] Implement `TracerProvider.Tracer(name, version string) Tracer`
- [ ] Implement global provider: `SetGlobalTracerProvider`, `GlobalTracerProvider`
- [ ] Write tests: spans flow through to InMemoryExporter, multiple named tracers are distinct, global provider is settable

---

### Phase 5 — W3C TraceContext Propagation

- [ ] Define `TextMapCarrier` interface — `Get`, `Set`, `Keys`
- [ ] Implement `MapCarrier` (wraps `map[string]string`) and `HeaderCarrier` (wraps `http.Header`)
- [ ] Define `TextMapPropagator` interface — `Inject`, `Extract`, `Fields() []string`
- [ ] Implement `W3CTraceContextPropagator`
  - [ ] `Inject` — serialise `traceID`, `spanID`, flags into `traceparent`; copy `tracestate`
  - [ ] `Extract` — parse `traceparent`; validate version, lengths, hex format; return `SpanContext`
  - [ ] Handle malformed headers gracefully — return empty `SpanContext`, do not panic
- [ ] Implement `B3Propagator` (multi-header)
  - [ ] `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled`, `X-B3-Flags`
- [ ] Implement `B3SingleHeaderPropagator`
  - [ ] `b3: {traceID}-{spanID}-{flag}`
- [ ] Implement `CompositePropagator` — inject all, extract first-success
- [ ] Set composite propagator on `TracerProvider`; expose `SetGlobalPropagator` / `GlobalPropagator`
- [ ] Write tests: round-trip inject→extract preserves all fields, malformed headers return empty context, B3 and W3C interop

---

### Phase 6 — Sampling

- [ ] Define `SamplingDecision` constants: `Drop`, `RecordOnly`, `RecordAndSample`
- [ ] Define `SamplingParameters` struct — `TraceID`, `ParentContext`, `SpanName`, `SpanKind`, `Attributes`
- [ ] Define `SamplingResult` struct — `Decision`, `Attributes`, `TraceState`
- [ ] Define `Sampler` interface — `ShouldSample(params) SamplingResult`, `Description() string`
- [ ] Implement `AlwaysOnSampler`
- [ ] Implement `AlwaysOffSampler`
- [ ] Implement `TraceIDRatioBased(ratio float64) Sampler`
  - [ ] Hash lower 8 bytes of `TraceID` using `binary.BigEndian.Uint64`
  - [ ] Compare against `uint64(ratio * math.MaxUint64)`
- [ ] Implement `ParentBased(root Sampler) Sampler`
  - [ ] If remote parent and sampled → `RecordAndSample`
  - [ ] If remote parent and not sampled → `Drop`
  - [ ] If no parent → delegate to `root`
- [ ] Implement `DynamicSampler` — wraps `atomic.Value`; expose `SetSampler(s Sampler)`
- [ ] Wire sampler into `tracer.Start` — call `ShouldSample`; if `Drop`, return `NoopSpan`
- [ ] Write tests: ratio sampler is deterministic on same traceID, ParentBased respects upstream flag, Drop returns NoopSpan

---

### Phase 7 — Batch Processor & OTLP Exporter

- [ ] Define `SpanProcessor` interface — `OnStart`, `OnEnd`, `Shutdown`, `ForceFlush`
- [ ] Implement `SimpleSpanProcessor` — calls exporter synchronously on `OnEnd`
- [ ] Implement `BatchSpanProcessor`
  - [ ] Internal buffered channel as queue
  - [ ] Background goroutine: flush on tick or `maxExportBatchSize` reached
  - [ ] `Shutdown` — signal goroutine to stop, flush remaining, wait for done
  - [ ] `ForceFlush` — drain queue synchronously within context deadline
  - [ ] Increment `droppedSpans` counter when queue is full
- [ ] Wire `[]SpanProcessor` into `TracerProvider` — call `OnStart` on span creation, `OnEnd` on `End()`
- [ ] Implement `OTLPHTTPExporter`
  - [ ] Serialise `[]ReadOnlySpan` to OTLP protobuf (`go.opentelemetry.io/proto/otlp`)
  - [ ] POST to `/v1/traces` with `Content-Type: application/x-protobuf`
  - [ ] Retry on 429 and 5xx with exponential backoff (max 3 retries, max 30s delay)
  - [ ] Configurable endpoint, headers, timeout
- [ ] Implement `OTLPGRPCExporter`
  - [ ] Connect to collector via gRPC; handle reconnection
  - [ ] Stream spans using `TraceServiceClient.Export`
- [ ] Write tests: batch flushes on size limit, flushes on timeout, dropped spans are counted, OTLP payload is valid protobuf

---

### Phase 8 — Resource Attributes & Instrumentation Libraries

- [ ] Define `Resource` struct — immutable `[]Attribute`
  - [ ] `NewResource(attrs ...Attribute) Resource`
  - [ ] `Resource.Merge(other Resource) Resource` — later keys win
- [ ] Implement auto-detection: `DetectResource()` reads `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES` env vars
- [ ] Define `semconv` package with string constants for all OTEL semantic convention attribute keys
- [ ] Implement `raven/http` server middleware
  - [ ] `NewMiddleware(tracer Tracer, propagator TextMapPropagator) func(http.Handler) http.Handler`
  - [ ] Extract trace context from request headers
  - [ ] Start `Server` span; set `http.method`, `http.route`, `http.url`, `http.scheme`
  - [ ] Record `http.status_code` on response; set `Error` status for 5xx only
  - [ ] Inject updated span context into response (for debugging)
- [ ] Implement `raven/httpclient` `RoundTripper`
  - [ ] `NewTransport(tracer Tracer, propagator TextMapPropagator, inner http.RoundTripper) http.RoundTripper`
  - [ ] Start `Client` span; inject `traceparent` into request headers
  - [ ] Record response status code and end span
- [ ] Implement `raven/sql` driver wrapper
  - [ ] Wrap `*sql.DB` methods: `QueryContext`, `ExecContext`, `PrepareContext`
  - [ ] Mask parameter values in `db.statement`
- [ ] Write integration tests: HTTP server middleware creates correct spans, client transport injects headers, spans have correct parent-child from server to client

---

### Phase 9 — Baggage & Correlation IDs

- [ ] Define `BaggageMember` struct — `Key`, `Value`, `Properties string`
- [ ] Define `Baggage` type — immutable slice of `BaggageMember`
  - [ ] `NewBaggage(members ...BaggageMember) (Baggage, error)` — validates keys and size
  - [ ] `Member(key string) (BaggageMember, bool)`
  - [ ] `Members() []BaggageMember`
- [ ] Implement `BaggageFromContext` / `ContextWithBaggage`
- [ ] Implement `BaggagePropagator`
  - [ ] `Inject` — serialise to W3C `baggage` header
  - [ ] `Extract` — parse `baggage` header; validate format; enforce size limits (8192 bytes total, 4096 per value)
  - [ ] Handle malformed values gracefully
- [ ] Add `BaggagePropagator` to the default `CompositePropagator`
- [ ] Implement `BaggageSpanProcessor`
  - [ ] On `OnStart`, read all baggage from ctx and call `span.SetAttributes` for each member
- [ ] Write tests: baggage survives round-trip inject→extract, oversized baggage is rejected, BaggageSpanProcessor stamps attributes

---

### Phase 10 — Tail-Based Sampling

- [ ] Define `TraceBuffer` struct — `map[TraceID]*traceRecord` with `sync.RWMutex`
- [ ] Define `traceRecord` — `spans []ReadOnlySpan`, `lastUpdated time.Time`
- [ ] Implement buffer eviction goroutine — sweep every `sweepInterval`; evict records older than `traceTimeout`; send evicted spans to evaluator
- [ ] Define `TailSampler` interface — `Evaluate(spans []ReadOnlySpan) SamplingDecision`
- [ ] Implement `LatencyPolicy(threshold time.Duration) TailSampler`
  - [ ] Find root span (empty parentSpanID); compute `endTime - startTime`
- [ ] Implement `ErrorPolicy() TailSampler`
  - [ ] Walk all spans; return `RecordAndSample` if any has `status=Error`
- [ ] Implement `RatioFallbackPolicy(ratio float64) TailSampler` — for traces that pass no other policy
- [ ] Implement `CompositePolicyEvaluator` with AND / OR / NOT combinators
- [ ] Implement `EndpointExclusionPolicy(paths ...string) TailSampler`
- [ ] Build `TailSamplingProcessor` — a `SpanProcessor` that buffers spans and delegates to evaluator on eviction
- [ ] Write tests: error traces are kept, health-check traces are dropped, latency threshold works, eviction fires correctly

---

### Phase 11 — Trace–Metrics Correlation (RED Metrics)

- [ ] Add `github.com/prometheus/client_golang` dependency
- [ ] Define `SpanMetricsConfig` — `Buckets []float64`, `LabelFn func(ReadOnlySpan) map[string]string`
- [ ] Implement `SpanMetricsProcessor`
  - [ ] Register `raven_requests_total` counter vec — labels: `service`, `span_name`, `status_code`
  - [ ] Register `raven_errors_total` counter vec — labels: `service`, `span_name`
  - [ ] Register `raven_duration_seconds` histogram vec — labels: `service`, `span_name`; configurable buckets
  - [ ] On `OnEnd`: observe duration, increment counters, record exemplar
- [ ] Implement Exemplar recording
  - [ ] Pass `traceID` as exemplar label on histogram `ObserveWithExemplar`
- [ ] Implement `PrometheusHTTPHandler() http.Handler` — wraps `promhttp.Handler()`
- [ ] Write tests: counters increment correctly, duration is within expected bucket, exemplar contains traceID

---

### Phase 12 — Async & Concurrent Instrumentation

- [ ] Add context propagation section to docs — goroutine spawn patterns with examples and anti-patterns
- [ ] Implement `raven.Go(ctx, tracer, name, fn func(ctx))` helper
  - [ ] Creates child span; passes ctx with span to fn in a new goroutine
- [ ] Implement `raven.GoGroup(ctx, tracer, name string, fns ...func(ctx)) Span`
  - [ ] Creates parent span; spawns each fn as goroutine with child span; waits with `sync.WaitGroup`
- [ ] Verify `Span.End()` is safe to call from a different goroutine than `Start()`
- [ ] Implement `raven/kafka` producer instrumentation
  - [ ] Wrap `sarama.SyncProducer.SendMessage`; inject trace context into `msg.Headers`
- [ ] Implement `raven/kafka` consumer instrumentation
  - [ ] Extract trace context from `msg.Headers`; create consumer span
  - [ ] Support both child-of and link-to modes via config
- [ ] Implement `raven/grpc` unary interceptors — client and server
- [ ] Implement `raven/grpc` streaming interceptors — client and server
- [ ] Write tests: fan-out spans have correct parent, goroutine span ends after parent without panicking, Kafka headers survive round-trip

---

### Phase 13 — Schema Evolution & Versioning

- [ ] Add `SchemaURL string` field to `Resource`
- [ ] Implement `Resource.WithSchemaURL(url string) Resource`
- [ ] Include `SchemaURL` in OTLP export payload (`ResourceSpans.schema_url`)
- [ ] Implement `AttributeRenameTransformer`
  - [ ] Takes `map[string]string` of `{oldKey: newKey}` renames
  - [ ] Applied as a `SpanProcessor` on `OnEnd` before export
- [ ] Implement `Resource.Merge` — combine auto-detected and manually specified resources; later keys win; merge schema URLs (warn if conflicting)
- [ ] Ensure `Tracer` name and version are exported in OTLP `InstrumentationScope`
- [ ] Write tests: schema URL appears in export, attribute rename transforms correctly, merge preserves both resource sets

---

### Phase 14 — Performance & Zero-Alloc Paths

- [ ] Implement `sync.Pool` for internal `span` structs
  - [ ] `spanPool.Get()` in `tracer.Start`; `spanPool.Put(span.reset())` in `span.End`
  - [ ] Implement `span.reset()` — zero all fields, reset slice lengths to 0
- [ ] Implement `sync.Pool` for attribute slices — pre-allocate `[]Attribute` with capacity 8
- [ ] Replace `BatchSpanProcessor` mutex queue with lock-free MPSC ring buffer
  - [ ] Power-of-2 capacity; atomic head/tail indices; CAS-based enqueue
- [ ] Write benchmark suite `bench_test.go`
  - [ ] `BenchmarkTracerStart` — allocations and ns/op for `Start + End`
  - [ ] `BenchmarkSpanFromContext` — context read cost
  - [ ] `BenchmarkInject` — W3C header serialisation
  - [ ] `BenchmarkExtract` — W3C header parsing
  - [ ] `BenchmarkDroppedSpan` — cost of a `Drop` decision (must be 0 allocs)
- [ ] Run `go test -bench=. -benchmem` and record baseline in `BENCHMARKS.md`
- [ ] Build `cmd/loadgen` — hammers tracer at configurable RPS; accepts `--rps`, `--duration`, `--pprof-port` flags
- [ ] Profile with `pprof` under load; identify and fix top 3 alloc hotspots
- [ ] Expose operational metrics via `SpanMetricsProcessor`: `raven_queue_size`, `raven_dropped_spans_total`, `raven_export_duration_seconds`
- [ ] Verify with `testing.AllocsPerRun` — sampled-away path must have 0 allocs
- [ ] Record final benchmark numbers in `BENCHMARKS.md`; add to README

---

> **GitHub:** `github.com/bitswright/raven`
> **Reference:** [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/) · [W3C TraceContext](https://www.w3.org/TR/trace-context/) · [OTLP](https://opentelemetry.io/docs/specs/otlp/)
>
> *Raven — traces fly dark and fast.*
