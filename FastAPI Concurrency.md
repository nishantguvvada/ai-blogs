# FastAPI Concurrency for AI Services

> Study notes based on *[The Concurrency Mistake Hiding in Every FastAPI AI Service](https://jamwithai.substack.com/p/the-concurrency-mistake-hiding-in)* by Shirin Khosravi Jam (March 17, 2026).
>
> The article's central point is simple: choose a concurrency design based on where time is spent. Waiting for a network response is an I/O problem; running local inference is a compute problem. Treating both as `async` work is a common cause of slow FastAPI services.

## The main mistake

FastAPI normally runs on an ASGI server such as Uvicorn. Its **event loop** is a single thread that coordinates many requests.

- A route written as `async def` runs directly on that event loop. It must use non-blocking operations and `await` while waiting for I/O.
- A route written as normal `def` is moved to Starlette's thread pool, keeping the event loop available. The default pool has 40 slots; once full, new work waits.

The mistake is putting a blocking operation—such as `requests.post()`, a synchronous database or file call, or local CPU/GPU inference—inside `async def`. The operation holds the event-loop thread. While it runs, other requests cannot progress, so the server behaves like a sequential server even though the route is labelled `async`.

```python
# Wrong for an external API: requests.post() blocks the event loop.
@app.get("/generate")
async def generate(prompt: str):
    response = requests.post("https://provider.example/v1/generate", json={"prompt": prompt})
    return response.json()
```

For external APIs, use an asynchronous client and await the request:

```python
import httpx

@app.get("/generate")
async def generate(prompt: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://provider.example/v1/generate",
            json={"prompt": prompt},
            timeout=60.0,
        )
    return response.json()
```

At `await`, the event loop can serve other requests until the network response arrives. The key rule is: **never call blocking code from an `async def` route unless you explicitly move that work elsewhere.**

---

## Concurrency is not parallelism

These related terms solve different problems:

- **Sequential execution:** one request completes before the next starts. This collapses quickly when each inference call takes seconds.
- **Concurrency:** several tasks are in progress together. One event-loop thread switches at `await` points while requests wait for I/O. The tasks are interleaved, not necessarily running at the same instant.
- **Parallelism:** tasks run simultaneously on separate CPU cores or processes. Separate Python processes have separate interpreters and separate GILs.

Async FastAPI provides useful concurrency for work that waits. It does not make Python compute run in parallel, and it does not make a CPU- or GPU-bound operation non-blocking.

---

## First classify the workload

### I/O-bound: use async clients

I/O-bound work mostly waits for another service or device:

- External model APIs such as OpenAI or Anthropic
- Vector-database queries
- Fetching RAG documents
- Web requests for enrichment
- Async database or cache calls

Use `async def` and async-capable libraries end to end. Each operation should yield with `await`, allowing the event loop to handle other requests during network waits. For this pattern, the FastAPI process is an efficient request coordinator.

### Compute-bound: use processes or a separate inference service

Compute-bound work actively consumes CPU or GPU rather than waiting:

- Local LLM generation
- Local embeddings with `sentence-transformers`
- CLIP/image inference
- Heavy NumPy or PyTorch work
- Tokenization when it is a substantial part of the request

Putting this work directly in `async def` blocks the event loop. Threads do not generally create CPU parallelism for Python bytecode because of the **GIL** (Global Interpreter Lock), which allows only one thread at a time to execute Python bytecode in a process. Use multiple processes for CPU-bound work, or move inference to a dedicated service.

---

## Three implementation levels

### 1. External-model service: fully async FastAPI

If the model runs at a remote provider, FastAPI should await an async SDK or `httpx` request. Many client requests can remain in flight while the event loop stays free.

Practical details:

- Set connection, read, and total timeouts.
- Reuse a long-lived client where appropriate instead of creating connections unnecessarily.
- Limit outbound work with a semaphore or a provider-aware queue; async makes it easy to start too many paid or rate-limited calls.
- Propagate cancellation when the client disconnects if the provider and SDK support it.

### 2. Local model in one application: process pool

For a smaller deployment that must run local CPU-bound inference in the application, use a `ProcessPoolExecutor` and await its result from the event loop:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from contextlib import asynccontextmanager
from fastapi import FastAPI

executor = ProcessPoolExecutor(max_workers=2)

def run_inference(prompt: str) -> str:
    # CPU-bound function executed in another process.
    return local_model_generate(prompt)

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    executor.shutdown(wait=True)

app = FastAPI(lifespan=lifespan)

@app.post("/generate")
async def generate(prompt: str):
    loop = asyncio.get_running_loop()
    text = await loop.run_in_executor(executor, run_inference, prompt)
    return {"response": text}
```

This keeps the web server responsive while inference runs in separate processes. It has important trade-offs:

- Each worker can load its own model copy. Two workers with a 2 GB model can require roughly 4 GB before serving requests.
- Arguments and results must cross process boundaries, so they need serialization.
- GPU models often have special multiprocessing and memory behavior; test the actual model stack rather than assuming a process pool will be efficient.
- The article reports lower latency under concurrent load for its example, but the exact numbers depend on hardware, model, batching, and workload.

### 3. Production serving: externalize inference

For serious local-model or GPU deployments, run inference in a dedicated service—such as vLLM, Text Generation Inference (TGI), or NVIDIA Triton—and let FastAPI call it asynchronously.

```python
@app.post("/generate")
async def generate(prompt: str):
    response = await inference_client.post(
        "http://inference:8000/v1/completions",
        json={"model": "my-model", "prompt": prompt, "max_tokens": 512},
        timeout=120.0,
    )
    return response.json()
```

This turns FastAPI's job back into I/O orchestration while the inference server owns GPU execution, model memory, scheduling, and throughput optimization. The article highlights two useful serving features:

- **Continuous batching:** new compatible requests can join an active batch instead of waiting for a fixed batch to finish. This improves throughput without forcing already-waiting users to wait for a batch to fill.
- **Paged attention:** vLLM manages the attention KV cache in smaller blocks, reducing memory fragmentation compared with reserving a large contiguous region per request. This can allow more simultaneous requests on the same GPU.

---

## Long-running work must leave the request path

Document summarization, multi-step agents, and video generation can take tens of seconds or minutes. Keeping one HTTP request open for that duration risks client timeouts and ties up resources.

For short, non-critical post-response work, FastAPI `BackgroundTasks` can return a job ID immediately and run a task after sending the response. It is not a durable distributed job system.

For jobs that need retries, priorities, multiple workers, persistence through restarts, or reliable status tracking, use a durable queue and worker system such as Celery, Arq, or Dramatiq:

1. Accept the request and create a job record.
2. Return `202 Accepted` with a `job_id`.
3. Put the job on a durable queue.
4. Have workers perform the work and save progress/results.
5. Let the client poll a status endpoint, or notify it separately.

This decouples the web request lifecycle from slow computation and gives the system a place to apply backpressure, retries, and priority rules.

---

## Two useful patterns

### Load models during application lifespan

Do not initialize a model inside each request handler. It makes cold requests pay model-startup time and can load multiple copies under concurrent traffic. FastAPI's lifespan hook runs once when the process starts and can release resources during shutdown.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

model = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global model
    model = load_model_once()
    yield
    close_model(model)

app = FastAPI(lifespan=lifespan)
```

This is appropriate when the model belongs in the FastAPI process. With multiple Uvicorn workers, each worker runs its own lifespan and may load another model copy; account for that memory use.

### Run independent guardrails and inference together

An input safety check and an inference call can start together. If the guardrail rejects the request first, cancel inference where possible. This lowers latency for safe requests but may spend some compute on requests that later fail the safety check.

```python
import asyncio

@app.post("/generate")
async def generate(prompt: str):
    guardrail = asyncio.create_task(check_input(prompt))
    inference = asyncio.create_task(call_model(prompt))

    done, _ = await asyncio.wait(
        [guardrail, inference], return_when=asyncio.FIRST_COMPLETED
    )
    if guardrail in done and not guardrail.result().safe:
        inference.cancel()
        return {"error": "Content policy violation"}

    return {"response": await inference}
```

Production code should also await or otherwise collect cancelled tasks safely, handle exceptions from either task, and ensure cancellation reaches any remote work when supported.

---

## Design checklist

1. Identify whether every major step is I/O-bound, CPU-bound, GPU-bound, or a long-running job.
2. In `async def`, use only non-blocking operations or explicitly offload blocking ones.
3. Use async SDKs/HTTP clients for remote model, retrieval, and database calls.
4. Bound expensive concurrent calls with semaphores, queues, rate limits, timeouts, and cancellation.
5. Use separate processes only when local compute needs true CPU parallelism; size them for model-memory duplication.
6. Prefer a dedicated inference server for production GPU/model serving and let FastAPI orchestrate it asynchronously.
7. Move durable long-running jobs to a queue and return a job ID rather than holding an HTTP connection open.
8. Load process-local models at startup, observe queue depth/latency/error rates/GPU memory, and test under realistic concurrent load.

The practical lesson is not “make every endpoint async.” It is: **keep the event loop free, give I/O work asynchronous clients, give compute work appropriate execution capacity, and detach long-running work from synchronous HTTP requests.**
