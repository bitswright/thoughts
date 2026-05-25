# Raven — Feature Breakdown by SDE Level

> Raven is a Go-based distributed tracing library built iteratively, from core span primitives to a production-grade, tail-sampling, metrics-correlated observability platform.
> Structured as 14 phases across SDE 1 / 2 / 3 tiers — each phase is a self-contained milestone with clear concepts and deliverables.

---

## Table of Contents

- [SDE 1 — Foundation: Core Tracing Primitives (Phases 1–4)](#sde-1--foundation-core-tracing-primitives)
- [SDE 2 — Distributed: Cross-Process & Production Concerns (Phases 5–9)](#sde-2--distributed-cross-process--production-concerns)
- [SDE 3 — Advanced: Tail Sampling, Metrics & Ecosystem (Phases 10–14)](#sde-3--advanced-tail-sampling-metrics--ecosystem)
- [Concepts Index](#concepts-index)

---

## SDE 1 — Foundation: Core Tracing Primitives

> Build the data model and lifecycle of a trace. Everything is single-process and in-memory. The goal is to understand what a span IS, how context flows through a callstack, and how traces are exported.

---

### Phase 1 — Span & Trace Data Model

**Goal:** Define the primitive structs that represent a single unit of work. Everything else builds on this.

| Feature | Detail |
|---|---|
| `TraceID` and `SpanID` | 128-bit and 64-bit random IDs; generated via `crypto/rand`; represented as hex strings |
| `Span` struct | `name`, `traceID`, `spanID`, `parentSpanID`, `startTime`, `endTime`, `status`; use `time.Time` for timestamps |
| Span status | `OK / ERROR / UNSET` enum; `SetStatus()` method |
| Attributes map | `map[string]string` key-value metadata on spans; discuss cardinality limits |
| Events | `[]Event{name, time, attrs}` — timestamped structured logs attached to a span; think of them as structured `printf` inside a span |
| Error recording | `RecordError()` adds an event with `exception.type` and `exception.message` attributes |

**Key concepts:** `TraceID`, `SpanID`, parent-child relationship, span status

---

### Phase 2 — Tracer & Span Lifecycle

**Goal:** Give users a clean API to start and finish spans. This is the entry point developers call every day.

| Feature | Detail |
|---|---|
| `Tracer` interface | `Start(ctx, name) (context.Context, Span)` — returns a new context with the span embedded |
| `Span.End()` | Records `endTime`, marks span as finished; idempotent — second `End()` is a no-op |
| `defer span.End()` pattern | Standard usage idiom; demo with a real HTTP handler; show how defers stack for nested spans |
| `SpanKind` | `INTERNAL`, `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER` — metadata used later by exporters and sampling |

**Key concepts:** span lifecycle, `defer` pattern, `SpanKind`

---

### Phase 3 — Context Propagation (In-process)

**Goal:** Thread the active span through Go's `context.Context` so nested calls automatically become child spans. This is the most important concept at SDE 1 level.

| Feature | Detail |
|---|---|
| `SpanFromContext` / `ContextWithSpan` | Store/retrieve the active span in `ctx`; use a private context key type to avoid collisions |
| Parent-child linking | Auto-set `parentSpanID` from context; if no span in context, new span is a root (`parentSpanID = ""`) |
| Span Links | Link a span to spans from other traces; useful for batch processing where one job touches many upstream traces |
| Noop span | Zero-overhead span when tracing is off; `NoopSpan` satisfies the `Span` interface but does nothing |

**Key concepts:** `context.Context`, context key pattern, root vs child span, noop pattern

---

### Phase 4 — In-Memory Exporter & TracerProvider

**Goal:** Give spans somewhere to go. Build the global registry and a simple exporter useful for tests.

| Feature | Detail |
|---|---|
| `TracerProvider` | Factory that vends `Tracer` instances by name/version; holds the exporter chain; `Tracer("my-service", "v1.0.0")` |
| `SpanExporter` interface | `ExportSpans(ctx, spans) error` + `Shutdown(ctx) error` — everything that ships spans implements this |
| `InMemoryExporter` | Stores finished spans in a slice; essential for unit tests; expose `GetSpans()` and `Reset()` |
| `StdoutExporter` | Pretty-prints spans as JSON to stdout; useful during development; add color-coded status output |
| Global `TracerProvider` | Package-level singleton with `SetGlobal` / `GetGlobal`; discuss why global state is acceptable here (instrumentation bootstrap) |

**Key concepts:** `TracerProvider`, exporter interface, global singleton, testability

---

## SDE 2 — Distributed: Cross-Process & Production Concerns

> Extend tracing across process and network boundaries. Handle the hard problems: sampling, batching, resource attributes, and wiring into real HTTP/gRPC stacks. This is where "local toy" becomes "runs in prod".

---

### Phase 5 — W3C TraceContext Propagation (HTTP)

**Goal:** The moment a span crosses a network, you need a wire format. W3C TraceContext is the industry standard.

| Feature | Detail |
|---|---|
| `TextMapPropagator` interface | `Inject(ctx, carrier)` / `Extract(ctx, carrier)`; carrier = anything that looks like HTTP headers (`map[string]string`) |
| `traceparent` header | Encode version, traceID, spanID, flags; format: `00-{traceID}-{spanID}-{flags}`; parse and validate strictly |
| `tracestate` header | Vendor-specific key-value baggage alongside `traceparent`; ordered list; preserve unknown keys; max 512 chars total |
| Composite propagator | Chain multiple propagators (W3C + B3 + custom); Inject calls all; Extract stops at first success |
| B3 propagator (single & multi header) | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled`; single-header: `b3: {traceID}-{spanID}-{flag}`; for Zipkin-compatible systems |

**Key concepts:** W3C TraceContext, inject/extract, B3 propagation, carrier abstraction

---

### Phase 6 — Sampling

**Goal:** Tracing 100% of traffic is expensive. Sampling decides which traces are worth keeping. Getting this wrong destroys either your budget or your observability.

| Feature | Detail |
|---|---|
| `Sampler` interface | `ShouldSample(params) SamplingResult`; result = `{Decision: RecordAndSample | Drop, Attributes, TraceState}` |
| `AlwaysOn` / `AlwaysOff` samplers | Baseline implementations; `AlwaysOff` is useful for disabling tracing without code changes |
| `TraceIDRatioBased` sampler | Deterministic sampling by traceID hash; same traceID always gets same decision; `ratio=0.1` = 10% of traces |
| `ParentBased` sampler | Respect upstream sampling decision; if upstream `sampled=1`, sample this too; avoids broken partial traces |
| Remote/dynamic sampling config | Hot-reload sampling rate without restart; poll a config endpoint; atomically swap the active sampler |

**Key concepts:** head sampling, `TraceIDRatio`, `ParentBased`, sampling decision propagation

---

### Phase 7 — Batch Processor & OTLP Exporter

**Goal:** You can't send every span over the network one by one. Build a batching layer and wire it to the OpenTelemetry Protocol (OTLP), the industry standard export format.

| Feature | Detail |
|---|---|
| `BatchSpanProcessor` | Buffer finished spans, flush on `maxBatchSize` or interval; goroutine-safe queue; configurable: `maxQueueSize`, `maxExportBatchSize`, `scheduleDelay` |
| `SimpleSpanProcessor` | Synchronous, one-span-at-a-time; useful in tests or CLI tools; never use in prod |
| OTLP/HTTP exporter | POST protobuf/JSON to a collector endpoint; target Jaeger, Tempo, or any OTEL collector; handle retries with backoff |
| OTLP/gRPC exporter | Stream spans via gRPC; more efficient than HTTP for high-throughput; handle connection lifecycle |
| Graceful shutdown | `TracerProvider.Shutdown()` drains the queue; respect the deadline; log dropped spans on timeout |

**Key concepts:** `BatchSpanProcessor`, OTLP, protobuf, back-pressure, graceful shutdown

---

### Phase 8 — Resource Attributes & Instrumentation Libraries

**Goal:** Spans without context about WHO produced them are hard to query in prod. Resources tag every span with service identity. Instrumentation libraries wrap the spans you'd write by hand.

| Feature | Detail |
|---|---|
| `Resource` | Immutable set of attrs describing the process (`service.name`, `service.version`, `host.name`); attach to `TracerProvider`; propagated to every span automatically |
| OTEL Semantic Conventions | Standard attribute names for HTTP, DB, RPC; `http.method`, `http.status_code`, `db.system`, `rpc.method`; don't invent your own |
| `net/http` middleware | Server-side: extract, create span, inject into response; wrap `http.Handler`; handle 4xx vs 5xx status correctly |
| HTTP client transport | Auto-inject `traceparent` on outgoing requests; implement `http.RoundTripper`; wrap the default transport |
| Database instrumentation | Wrap `sql.DB` / redis client to emit DB spans; `db.system`, `db.statement`, `db.operation`; mask sensitive query values |

**Key concepts:** `Resource`, semantic conventions, middleware pattern, `RoundTripper`

---

### Phase 9 — Baggage & Correlation IDs

**Goal:** Baggage lets you pass key-value data across process boundaries alongside spans — useful for user IDs, feature flags, or tenant context.

| Feature | Detail |
|---|---|
| Baggage API | `Set` / `Get` / `Remove`; stored in context alongside span; W3C Baggage header: `baggage: userId=abc,env=prod` |
| Baggage sanitization | Validate key names, max value size, total size limit; enforce 8KB total; malicious values can bloat every outbound request |
| Auto-copy baggage to span attributes | Optional `BaggageSpanProcessor`; useful for correlation — see `userId` on every span without manual `SetAttr` calls |

**Key concepts:** W3C Baggage, cross-process correlation, `SpanProcessor` chain

---

## SDE 3 — Advanced: Tail Sampling, Metrics & Ecosystem

> This tier is where the library becomes a platform. Tail-based sampling, trace-metrics correlation (the "three pillars"), schema evolution, extensibility hooks, and performance at scale. These require deep understanding of distributed systems trade-offs.

---

### Phase 10 — Tail-Based Sampling

**Goal:** Head sampling is blind — you decide before seeing the trace. Tail sampling buffers the whole trace and decides after, so you can always keep errors and slow traces.

| Feature | Detail |
|---|---|
| In-memory trace buffer | Hold spans for a trace for N seconds before decision; `map[traceID][]Span`; evict on timeout or memory pressure |
| Latency-based policy | Always keep traces above P99 threshold; e.g. keep any trace > 500ms; configurable percentile |
| Error-based policy | Always keep traces with any `ERROR` span; walk span tree looking for `status=ERROR`; never sample these away |
| Composite policy evaluator | AND/OR/NOT combinators over policies; e.g. keep if `(latency > P99 OR hasError) AND NOT (endpoint="/healthz")` |
| Remote tail sampler | Send decisions from a central collector; needed when spans of one trace arrive at different collector replicas |

**Key concepts:** tail sampling, trace buffer, policy combinators, late-arrival spans

---

### Phase 11 — Trace–Metrics Correlation (RED Metrics)

**Goal:** Automatically derive service-level metrics from spans so you get tracing AND dashboards from one instrumentation call.

| Feature | Detail |
|---|---|
| `SpanMetricsProcessor` | Derive RED metrics from span data; Rate = spans/sec; Errors = error spans/sec; Duration = histogram of `span.duration` |
| Histogram bucketing | Configurable latency buckets; default: `[5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s]` |
| Exemplar injection | Embed a `traceID` in a metric data point; enables "click a spike on your dashboard → jump to the trace that caused it" |
| Prometheus exporter for span metrics | Expose `/metrics` endpoint; use `go-prometheus` SDK; labels = `service.name`, `span.name`, `status.code` |

**Key concepts:** RED metrics, exemplars, `SpanMetricsProcessor`, histogram

---

### Phase 12 — Async & Concurrent Instrumentation

**Goal:** Real services have goroutines, channels, and fan-outs. Trace work that splits across multiple goroutines and merges at the end.

| Feature | Detail |
|---|---|
| Context propagation across goroutines | Pass `ctx` explicitly, not via closure; demo the anti-pattern where closure captures `ctx` at the wrong point in time |
| Fan-out tracing | Parent span + N child spans launched concurrently; use `sync.WaitGroup`; each goroutine gets its own span with the same parent |
| Span Links for async messaging | Producer span links to consumer span; different trace trees; Links connect without merging; visible in Jaeger UI |
| Message queue instrumentation | Inject into Kafka/SQS message headers; Producer: `Inject` into `msg.Headers`; Consumer: `Extract` and start child span |

**Key concepts:** context in goroutines, fan-out pattern, Span Links, messaging propagation

---

### Phase 13 — Schema Evolution & Versioning

**Goal:** The library will be used by many services at different versions simultaneously. Span schemas must evolve without breaking old exporters or storage backends.

| Feature | Detail |
|---|---|
| Schema URL on Resource | Declare which attribute convention version this service uses; collectors can convert between schema versions; prevents attribute name drift |
| `SpanTransformer` | Rewrite attribute names during export for schema migration; e.g. `net.peer.ip → server.address` during a convention upgrade |
| Instrumentation scope versioning | Name + version on every `Tracer`; lets you see "which library version produced this span" in the backend |

**Key concepts:** schema URL, attribute migration, instrumentation scope

---

### Phase 14 — Performance, Benchmarking & Zero-Alloc Paths

**Goal:** A tracing library must have near-zero overhead on the hot path. Prove it with benchmarks and fix the alloc hotspots.

| Feature | Detail |
|---|---|
| `sync.Pool` for `Span` objects | Reuse span structs to avoid GC pressure; benchmark `allocs/op` before and after; target 0 allocs on sampled-away spans |
| Lock-free span queue | Atomic ring buffer for the batch processor; replace `sync.Mutex` queue with a lock-free MPSC queue for throughput |
| Microbenchmarks for hot paths | `Tracer.Start`, context extraction, inject/extract; use `testing.B` and `benchstat`; document baseline numbers in README |
| `pprof` integration | Profile the library itself under load; build a load generator; run `pprof`; find the real bottleneck (often JSON marshal) |
| Back-pressure and queue overflow handling | Drop vs block policy when queue is full; expose `DroppedSpans` counter as a metric |

**Key concepts:** `sync.Pool`, lock-free queue, `benchstat`, `pprof`, back-pressure

---

## Concepts Index

| Concept | First Introduced |
|---|---|
| `TraceID` / `SpanID` | Phase 1 |
| Span status (`OK / ERROR / UNSET`) | Phase 1 |
| `context.Context` propagation | Phase 3 |
| Noop span | Phase 3 |
| `TracerProvider` | Phase 4 |
| `SpanExporter` interface | Phase 4 |
| W3C TraceContext / `traceparent` | Phase 5 |
| B3 propagation | Phase 5 |
| Head sampling | Phase 6 |
| `TraceIDRatioBased` sampler | Phase 6 |
| `ParentBased` sampler | Phase 6 |
| `BatchSpanProcessor` | Phase 7 |
| OTLP (HTTP + gRPC) | Phase 7 |
| Graceful shutdown | Phase 7 |
| Resource attributes | Phase 8 |
| Semantic conventions | Phase 8 |
| `http.RoundTripper` instrumentation | Phase 8 |
| W3C Baggage | Phase 9 |
| Tail sampling | Phase 10 |
| Policy combinators | Phase 10 |
| RED metrics | Phase 11 |
| Exemplar injection | Phase 11 |
| Context in goroutines | Phase 12 |
| Span Links | Phase 12 |
| Schema URL / versioning | Phase 13 |
| `sync.Pool` | Phase 14 |
| Lock-free queue | Phase 14 |
| `pprof` + `benchstat` | Phase 14 |

---

## Phase Dependency Notes

Unlike Kivi where phases are strictly sequential (WAL before segments), some phases here are independent within a tier:

- **Phases 8 and 9** (SDE 2) — instrumentation libraries and Baggage are independent of each other; can be built in parallel
- **Phases 11 and 12** (SDE 3) — RED metrics and async instrumentation are independent; share no code
- **Phase 14** (performance hardening) — can be applied incrementally to any earlier phase once benchmarks reveal a hotspot

The ordering within each tier is a suggested learning path, not a hard dependency chain.

---

> **GitHub:** `github.com/bitswright/raven`
> **Reference:** [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/) · [W3C TraceContext](https://www.w3.org/TR/trace-context/) · [OTLP](https://opentelemetry.io/docs/specs/otlp/)
>
> *Raven — traces fly dark and fast.*
