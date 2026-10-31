# AI System Design Interview Questions and Answers

> Study guide based on *[System Design for AI Engineers: 7 patterns you should know in your interviews](https://jamwithai.substack.com/p/system-design-for-ai-engineers-7)* by Shirin Khosravi Jam (April 9, 2026).
>
> The article gives short interview prompts. This guide turns them into fuller, plain-language answers. The expanded answers are study notes based on the article, not quotes from its author.

## A simple way to frame an AI system

In an interview, start outside the system and move inward:

1. A request enters through an **API gateway**.
2. **Rate limits** stop one user from using all the budget or capacity.
3. A **cache** avoids doing the same expensive work again.
4. A **queue** holds background work until a worker can process it.
5. A **circuit breaker** stops one broken dependency from breaking everything else.
6. A **load balancer** sends work across several model servers.
7. **Auto scaling** adds or removes servers as demand changes.

For every part, explain three things: what can fail, how users are protected, and how cost is controlled.

---

## 1. Walk me through how a request flows through your AI system.

**Answer:**

I would begin with an **API gateway**, which is the front door of the system. It checks who is making the request, verifies that it is valid, applies limits, and sends it to the right internal service. This means clients do not need to know where the embedding service, search system, or model servers live.

For a RAG application—one that searches company documents before answering—the normal flow is:

1. The gateway checks the user's identity and limits. It can reject requests that are too large or that exceed the user's plan.
2. The app checks a cache. If it already has a safe answer or embedding for the same request, it can reuse it.
3. If needed, it creates an **embedding**: a numeric representation of the question that makes similar meanings easier to find.
4. It searches the vector database for relevant documents. The search must respect the user's access rights.
5. It builds a prompt from the user's question, the approved documents, and clear instructions. It keeps the prompt within a token budget so cost and response time stay predictable.
6. A model server generates the answer. The system uses time limits and a backup provider or model if that is part of the product design.
7. Before returning the answer, the app checks the format, applies safety rules, attaches citations where appropriate, and records useful operational data.

I would also mention that each step needs logging, tracing, and time limits. That makes it possible to find where a slow or failed request went wrong. The key interview point is to start with the gateway and show the whole path, not jump straight to the model.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** Login checks, document search, a model provider, or an output check can be slow or unavailable. A very large prompt can also exceed a limit.
- **How users are protected:** Give each step a time limit, show a clear error or a safe fallback when a dependency fails, and never return documents the user is not allowed to see. Include only citations that were actually retrieved.
- **How cost is controlled:** Reject oversized requests early, reuse safe cached work, set a token budget for the prompt and answer, and avoid calling a model when a cheaper rule or cached result is enough.

---

## 2. How would you prevent a denial-of-wallet attack on an inference API?

**Answer:**

A **denial-of-wallet attack** is when someone deliberately—or accidentally—causes a company to spend a lot of money on AI requests. For example, they may send thousands of huge prompts, ask for very long answers, or keep retrying failed requests. The service may still be available, but the bill can grow very quickly.

The first defense is **rate limiting** at the API gateway, before the request reaches an expensive model or GPU. I would limit both:

- **Requests per minute:** how often a user can call the API.
- **Tokens per minute:** how much text a user can send and receive. This matters because one long request can cost far more than many short ones.

I would also add:

- A maximum input size and a maximum answer length.
- Spending budgets for each user, project, and organization, with alerts before the limit is reached.
- Separate limits for live user requests and background jobs.
- A safe retry policy. Retries should be limited and delayed; they should not create an endless paid loop.
- Detection for unusual traffic spikes, repeated failures, or abnormally large prompts.
- Caching for repeated work and queues for work that does not need an immediate answer.

When a caller reaches a limit, the API should return a clear `429 Too Many Requests` response and say when to retry. If product rules allow it, the service might offer a smaller model or a shorter answer instead. The goal is to protect both availability and cost without surprising legitimate users.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** A limit can be configured too tightly, a customer's legitimate traffic can spike, or a retrying client can keep sending failed requests.
- **How users are protected:** Use fair limits per user or organization rather than one global limit, return a useful retry time, and alert support teams before normal customers are blocked.
- **How cost is controlled:** Enforce request, token, and spend caps before model calls; stop runaway retries; and use cheaper fallbacks only when they meet the product's quality rules.

---

## 3. Your embedding API receives the same queries repeatedly. How do you optimize it?

**Answer:**

I would put a **cache** in front of the embedding service. A cache stores a previous result so the system does not repeat the same expensive calculation.

Before looking in the cache, normalize harmless differences in the text, such as extra spaces. Use the cleaned text plus the embedding-model version as the cache key. Including the model version is important: an embedding from an old model may not work correctly after the model changes.

There are two useful kinds of caching:

- **Exact-match caching:** reuse a result only when the input is exactly the same after cleanup. This is simple and safe.
- **Semantic caching:** reuse a result for a very similar question. For example, “How do I reset my password?” and “I forgot my password—what now?” may be close enough. This saves more money, but needs careful testing so it does not return the wrong result.

For a semantic cache, I would choose the similarity threshold using real evaluation data, not intuition. I would also keep customer or tenant information in the cache key. A result based on one customer's private data must never be returned to another customer.

Useful metrics are cache-hit rate, saved cost, lower response time, and cases where a semantic match was not good enough.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** A cache can return an old result, a semantic match can be similar but still wrong, or an old embedding can remain after the model changes.
- **How users are protected:** Keep private data separated in cache keys, set expiration times, test semantic-match thresholds, and fall back to a fresh calculation when confidence is too low.
- **How cost is controlled:** Exact-match caching removes duplicate embedding calls, while carefully tested semantic caching can reduce more repeated work without lowering answer quality too far.

---

## 4. You need to summarize 50,000 documents. How would you architect it?

**Answer:**

I would not write one loop that sends 50,000 requests to a model and waits for each one. That would be slow, hard to recover, and likely to hit model-provider limits.

Instead, I would use a **message queue** and a pool of workers.

1. Store each document and its job status in a database.
2. Split long documents into manageable pieces when necessary.
3. Put one task per document or chunk into a durable queue, such as SQS, RabbitMQ, or Kafka.
4. Workers take tasks from the queue at a controlled pace, call the model, check the output, and save the result.
5. If a call fails briefly, retry it a small number of times with a delay that grows between attempts.
6. Move tasks that keep failing into a dead-letter queue, which is a separate place for human review.
7. Show progress, failures, and retry counts to operators and users.

A queue separates the part that creates work from the part that does work. This absorbs traffic spikes and lets workers process at a rate the model service can handle.

For very long documents, I would use hierarchical summarization: summarize each chunk, then summarize those smaller summaries. I would save progress after each step so the system can resume after a worker crash or provider outage. The workers can scale up when the queue grows and scale down when it is empty.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** A worker can crash, a model call can time out, the queue can grow faster than workers can process it, or a document can be processed twice after a retry.
- **How users are protected:** Save job progress, show clear job status, use a dead-letter queue for repeated failures, and make each task safe to retry without producing duplicate results.
- **How cost is controlled:** Set a worker-concurrency limit, scale workers from queue depth, process long documents in chunks, and stop or review jobs that repeatedly fail instead of retrying forever.

---

## 5. What happens when your vector database goes down in the middle of a query?

**Answer:**

The system should fail fast and prevent one broken service from causing a wider outage. I would place a **circuit breaker** around calls to the vector database.

A circuit breaker is like an electrical safety switch:

- **Closed:** requests flow normally while the system watches error rates and response time.
- **Open:** if too many searches fail or time out, the system stops sending new requests to the vector database for a short period. It returns a fast fallback instead of making every user wait.
- **Half-open:** after a short wait, the system tries a small number of test requests. If they succeed, normal traffic resumes. If not, the circuit stays open.

The fallback depends on how important the retrieved information is:

- For a regulated, factual, or high-risk product, return a clear message that search is temporarily unavailable.
- If it is safe, return a clearly marked cached answer that may be old.
- A model-only answer may be acceptable for some products, but it must say that no documents were retrieved. It must not invent citations.

I would set short time limits, use only a few retries, monitor the dependency, and alert the team. The important point is to protect the rest of the system from a growing backlog of requests waiting on a service that is already down.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** The vector database may be down, slow, overloaded, or return incomplete results.
- **How users are protected:** The circuit breaker produces a fast, honest fallback instead of a hanging page; permissions still apply to cached data; and the interface clearly says when no sources were retrieved.
- **How cost is controlled:** Failing fast avoids paying for model calls built on missing context, prevents wasted retries, and stops waiting requests from using more connections and compute.

---

## 6. How would you handle 10,000 concurrent requests to an LLM API?

**Answer:**

First, I would decide what the system can safely accept. The gateway would authenticate callers, apply request and token limits, set maximum prompt and answer sizes, and give priority to important traffic. If the system is full, it is better to delay or reject some requests clearly than to make every request slow or failing.

Next, I would place a **load balancer** in front of several model servers. It checks server health and sends traffic away from unhealthy machines.

### Why simple round-robin is not enough for inference

**Round-robin** sends requests in a fixed turn: server A, then B, then C, then back to A. It is useful when requests are roughly the same size and take roughly the same time. For example, it can work well for simple web requests that all do similar work.

It is often insufficient for LLM inference because AI requests are not equal:

- One request may have a 50-token prompt; another may contain a 100-page document.
- One user may ask for a two-sentence answer; another may request a long report.
- Some requests stream for a long time or call tools; others finish quickly.
- A GPU may be busy holding a large request in memory even when another server is almost free.

With round-robin, a busy server can receive another heavy request simply because it is "its turn," while an idle server receives nothing. This creates uneven queues and worse response time.

For model serving, I would prefer **load-aware routing**: send a request to the healthy server with the smallest queue, the fewest active requests, or the most available GPU memory. I would measure queue length, response time, token throughput, active requests, and GPU memory pressure.

Other practical choices are:

- Batch compatible requests together, if the serving system supports it.
- Keep short, interactive requests separate from long or batch jobs.
- Stream the answer so users see the first tokens earlier.
- Use a queue with backpressure when capacity is full.
- Cancel work when the user disconnects.
- Add warmed-up servers before expected demand peaks.
- Use asynchronous programming for network calls and streaming so one waiting request does not block the server from handling other requests. Pair it with a **semaphore**, which is a simple counter that allows only a fixed number of expensive model calls to run at once. Async code improves throughput, but without a semaphore it can start too many model calls at the same time and overload GPUs, connection pools, or a provider's rate limit.

The best answer makes clear that protecting the system during overload is part of normal design, not an afterthought.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** A GPU server can become unhealthy, queues can grow, a client can disconnect during streaming, or too many asynchronous tasks can start at once.
- **How users are protected:** Health checks route around bad servers; queues and a semaphore keep the service responsive; users receive a clear delay or `429` response instead of a silent timeout; and disconnected work is cancelled where possible.
- **How cost is controlled:** Load-aware routing uses existing capacity better, batching reduces wasted GPU time, the semaphore caps expensive parallel work, and cancellation stops generating tokens that nobody will read.

---

## 7. Your LLM API gets 10× traffic at 9 AM and almost none at 3 AM. How do you manage cost?

**Answer:**

I would use **auto scaling**, which adds servers when demand rises and removes them when demand falls. For AI workloads, I would not use CPU use alone as the main signal. A model server may show low CPU use while its GPU memory is nearly full and requests are waiting.

Better signals include:

- Number of requests waiting in the queue.
- Time taken by the slowest 5% of requests.
- Number of tokens generated per second.
- GPU use and GPU-memory use.
- Number of active requests on each model server.

Because the 9 AM peak is predictable, I would start extra model servers shortly before it. This is better than waiting for the queue to grow because loading a model onto a GPU can take time. I would keep a small number of servers warm for fast responses, then scale down only after low demand lasts for a while.

I would also control cost by:

- Sending simple tasks to a smaller or cheaper model when testing shows the quality is good enough.
- Caching repeated embeddings and safe repeated answers.
- Running large background jobs during quieter periods when possible.
- Keeping live customer traffic separate from interruptible background work.
- Tracking cost by customer, request, input text, output text, and completed task.

The cheapest setup is not always the best setup. If too few servers cause long queues, timeouts, and retries, the service becomes worse for users and may cost more overall.

### What can fail, how users are protected, and how cost is controlled

- **What can fail:** Scaling can happen too late, a newly started server may not be ready in time, or a metric can cause the system to scale up and down too often.
- **How users are protected:** Keep a small warm baseline, start extra capacity before predictable peaks, and use safe scale-down delays so active requests are not interrupted.
- **How cost is controlled:** Scale on GPU and queue signals, remove unused capacity after a stable quiet period, route suitable work to smaller models, and track cost so the team can see which customers and features drive spend.

---

## Quick interview checklist

When answering an AI system-design question, cover these points in plain language:

1. **Gateway:** who can call the system, where the request goes, and what is rejected early.
2. **Rate limits:** how cost and capacity are protected; mention both request count and text/token limits.
3. **Cache:** what repeated work can be reused, when entries expire, and how private data stays separated.
4. **Queue:** which jobs can run in the background and how failures are retried safely.
5. **Circuit breaker:** what happens when a dependency fails and what fallback the user sees.
6. **Load balancing:** why requests have unequal cost and how traffic is sent to the least busy healthy model server.
7. **Auto scaling:** which AI-specific signals add or remove capacity, and how predictable peaks are prepared for.

Finally, mention monitoring, user access rules, data separation, time limits, safe retries, honest fallback behavior, and cost tracking. These show that you are designing a production service, not only a model demo.
