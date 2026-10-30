# AI System Design Interview Questions and Answers

> Study guide based on *[System Design for AI Engineers: 7 patterns you should know in your interviews](https://jamwithai.substack.com/p/system-design-for-ai-engineers-7)* by Shirin Khosravi Jam (April 9, 2026).
> The post gives short interview signals; the answers below are expanded, practical study answers synthesized from the article's patterns. They are not quoted answers from the author.

## 1. Walk me through how a request flows through your AI system.

**Answer:**

Start at an **API gateway**. It terminates TLS, authenticates the caller, validates the request, applies tenant-level quotas, and routes the request to the correct capability or model provider. It should enforce both request-per-minute and token-per-minute limits before expensive work begins.

For a RAG request, the application then:

1. Normalizes the request and checks exact or semantic caches.
2. Creates or retrieves a query embedding.
3. Queries the vector store, applying authorization filters to retrieved documents.
4. Builds a bounded prompt with retrieved context, instructions, and token-budget controls.
5. Sends the prompt to a model-serving tier through a provider abstraction that supports timeouts, retries, and fallback.
6. Applies output validation, safety checks, citations, and observability before streaming or returning the response.

Every boundary needs tracing, structured logs, latency/error metrics, and a clear timeout budget. The gateway is the right place to start the explanation because it protects the downstream model and hides internal service topology from clients.

---

## 2. How would you prevent a denial-of-wallet attack on an inference API?

**Answer:**

Use **token-aware rate limiting** and spend controls, not only IP-based request limits. An API request may vary from a tiny embedding call to a long-context generation, so track both requests per minute (RPM) and tokens per minute (TPM), per tenant and per API key.

A robust design includes:

- A gateway limit before requests reach GPUs or an external model provider.
- Maximum input-token, output-token, and context-window limits.
- Per-user, per-project, and organization budgets with alerts and hard caps.
- Separate quotas for interactive traffic and background/batch workloads.
- Idempotency keys to avoid charging twice for retried writes or jobs.
- Detection of anomalous prompt sizes, traffic bursts, and repeated failures.
- Caching for repeated work and queueing for non-urgent work.

When a limit is exceeded, return a clear `429` response with a retry hint or degrade to a cheaper model/cached answer if the product permits it. Do not allow unlimited retries to turn a transient failure into an expensive loop.

---

## 3. Your embedding API receives the same queries repeatedly. How do you optimize it?

**Answer:**

Put a **cache** in front of embedding generation. Normalize text first—for example, trim whitespace and standardize harmless formatting—then hash the normalized input plus the embedding-model version. Cache the resulting vector with a TTL and invalidate it whenever the embedding model, preprocessing logic, or relevant corpus version changes.

Use two levels where needed:

- **Exact-match cache:** fast, safe, and ideal for identical text.
- **Semantic cache:** serves a prior result only when a new query is sufficiently similar and the use case tolerates approximation.

For semantic caching, set a similarity threshold using offline evaluation rather than guesswork. It can reduce cost and latency but risks returning an answer for a meaningfully different query. Record cache-hit rate, latency saved, false-hit incidents, and cost avoided. Keep authorization boundaries in the cache key; never serve a cached tenant-specific result to another tenant.

---

## 4. You need to summarize 50,000 documents. How would you architect it?

**Answer:**

Use an **asynchronous, queue-backed worker system**, not a synchronous loop issuing 50,000 model requests.

1. Store document metadata and job state in a durable database.
2. Split documents into appropriately sized chunks and publish work items to a durable queue such as SQS, RabbitMQ, or Kafka.
3. Run a worker pool that pulls tasks at a controlled concurrency, calls the model, validates output, and writes results to durable storage.
4. Use idempotency keys so retries do not create duplicate summaries or duplicate billing.
5. Send transient failures through bounded retries with exponential backoff; move exhausted items to a dead-letter queue for inspection.
6. Autoscale workers from queue depth, job age, GPU utilization, and provider rate-limit headroom.
7. Expose progress, error counts, and retry status to operators and users.

For very long documents, apply hierarchical summarization: summarize chunks, then summarize the chunk summaries. Use checkpoints so the job can resume after a worker or provider outage.

---

## 5. What happens when your vector database goes down mid-query?

**Answer:**

The system should fail quickly and avoid turning one dependency failure into a platform outage. Put a **circuit breaker** around vector-store calls.

- **Closed:** normal requests pass through while latency and error rate are measured.
- **Open:** after a configured error/timeout threshold, new calls fail immediately instead of waiting for a dependency that is already unhealthy.
- **Half-open:** after a cooldown, allow a small number of probes. Close the circuit only after successful probes; otherwise reopen it.

Choose the fallback by product risk:

- Return a clear “retrieval is temporarily unavailable” response for high-accuracy or regulated use cases.
- Serve a previously cached, explicitly stale answer when that is safe.
- Allow a model-only response only if it is labeled as having no retrieved sources and product policy permits it.

Use tight timeouts, bounded retries, bulkheads/connection-pool limits, health checks, alerts, and trace IDs. Do not silently invent citations when retrieval failed.

---

## 6. How would you handle 10,000 concurrent requests to an LLM API?

**Answer:**

Start with admission control: authenticate callers, apply RPM/TPM quotas, cap prompt and completion sizes, and prioritize traffic classes. Then place a load balancer in front of multiple serving replicas and route around unhealthy nodes.

Inference needs **load-aware balancing**, not simple round-robin. Requests vary greatly by input length, output length, model, streaming behavior, and tool use. Use metrics such as active sequences, queue depth, KV-cache/GPU-memory pressure, token throughput, and latency percentiles to make routing decisions.

For the serving tier:

- Use continuous batching and request scheduling where supported.
- Separate small/interactive work from long-context or batch work.
- Stream tokens to reduce perceived latency.
- Use queues with backpressure when capacity is exceeded.
- Autoscale replicas and warm capacity ahead of expected peaks.
- Enforce timeouts, cancellation, and client-disconnect cleanup.

Design for overload explicitly: some callers should receive a controlled `429` or delayed-job response rather than causing universal high latency or an outage.

---

## 7. Your LLM API gets 10× traffic at 9 AM and almost none at 3 AM. How do you manage cost?

**Answer:**

Use **GPU-aware autoscaling** with a combination of predictive and reactive control. Reactive scaling can use inference queue depth, p95 latency, token throughput, GPU utilization, GPU-memory utilization, and active sequences. CPU utilization alone is a poor signal for GPU inference.

Predictable daily demand should be handled with scheduled or forecast-based pre-scaling because model loading and GPU provisioning have cold-start delays. Keep a small warm baseline for latency-sensitive traffic, then scale down safely after a sustained low-utilization window.

Additional cost controls include:

- Route simple tasks to smaller or cheaper models after quality evaluation.
- Cache repeated embeddings and safe repeated responses.
- Run batch work during lower-demand periods when possible.
- Use separate capacity pools for premium interactive traffic and interruptible/background jobs.
- Track cost per tenant, request, input token, output token, and successful task.

Autoscaling should protect service-level objectives first. A cheaper fleet that is permanently saturated creates longer queues, timeouts, retries, and eventually higher total cost.

---

## Quick Interview Framework

For most AI-system design interviews, describe the path from the outside in:

1. **Gateway:** authentication, routing, request validation, quotas, and provider failover.
2. **Rate limiting:** cost and availability protection using RPM and TPM.
3. **Caching:** reduce repeated expensive work; include safe invalidation and tenant isolation.
4. **Queues:** move bulk, slow, and retryable work out of synchronous request paths.
5. **Circuit breakers:** contain dependency failures and define honest fallbacks.
6. **Load balancing:** distribute variable-cost inference using health and load signals.
7. **Autoscaling:** scale with GPU-aware metrics, forecasts, and cold-start awareness.

Across all layers, discuss observability, authorization, data isolation, retries, idempotency, degradation behavior, and an explicit cost budget.
