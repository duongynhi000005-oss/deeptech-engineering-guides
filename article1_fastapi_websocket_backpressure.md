# Fixing Memory Leaks and Slow-Consumer Backpressure in FastAPI WebSockets (With Pytest Suite)

**Tags:** `python`, `fastapi`, `asyncio`, `websockets`

Real-time AI voice pipelines, live transcription gateways, and LLM token streamers share a deceptively dangerous failure mode in Python `asyncio` and FastAPI: **silent background task leaks combined with unbounded per-client outbound queues**.

Everything works smoothly in local testing with a single browser tab. Then production traffic arrives:
1. A mobile client enters a flaky 4G tunnel; its TCP receive window stalls while your server keeps generating 50 PCM16 audio frames or LLM tokens per second.
2. Unawaited or fire-and-forget `asyncio.create_task()` calls accumulate in memory, or raise unhandled exceptions that get garbage-collected without triggering cleanup.
3. Within hours, worker RSS memory spikes to multi-gigabytes and the container is killed by the Linux OOM killer.

In this guide, we will build two zero-dependency production primitives—a **Fault-Isolated `TaskSupervisor`** and a **Bounded `WebSocketSessionGateway` with `drop_oldest` backpressure**—and verify both using `pytest`.

> **Want the complete drop-in starter kit?** Grab the full [**Production FastAPI + WebSocket Real-Time Audio/LLM Streaming & Task Supervisor Kit ($5)**](https://ko-fi.com/nhi741323) or check out our [**Deep-Tech Freelance & Crypto Tax OS ($5)**](https://ko-fi.com/s/6f070d3e68) on Ko-fi (also available on [GitHub Releases](https://github.com/duongynhi000005-oss/fastapi-websocket-realtime-ai-starter-kit/releases/tag/v1.0.0)).

---

## Pitfall #1: Fire-and-Forget `asyncio.create_task()` Leaks

In many FastAPI WebSocket endpoints, developers spawn background coroutines for webhook delivery, database persistence, or STT chunk forwarding like this:

```python
# ❌ ANTI-PATTERN: Untracked task reference + unbounded concurrency
asyncio.create_task(forward_audio_chunk(session_id, pcm_bytes))
```

This creates three production bugs:
1. **Garbage-Collected Task References:** The Python docs explicitly warn that the event loop only keeps *weak references* to tasks. A task not referenced elsewhere can be garbage-collected mid-execution.
2. **Unbounded Concurrency Storms:** If 500 WebSocket clients send 20 chunks/second during a downstream database hiccup, 10,000 coroutines pile up simultaneously.
3. **Swallowed Exceptions:** When a background task crashes, nobody records the failure context or retries with exponential backoff.

### Solution: A Fault-Isolated `TaskSupervisor`

Let's build a `TaskSupervisor` that enforces an `asyncio.Semaphore` concurrency ceiling, removes completed tasks deterministically via `add_done_callback`, retries transient failures with exponential backoff, and records permanent failures in a bounded `DeadLetterEvent` ring buffer:

```python
from __future__ import annotations
import asyncio
import time
from dataclasses import dataclass
from typing import Any, Awaitable, Callable, Dict, List, Optional


@dataclass(frozen=True)
class SupervisionPolicy:
    max_retries: int = 2
    base_delay_sec: float = 0.05
    max_delay_sec: float = 1.0
    timeout_sec: Optional[float] = 5.0


@dataclass(frozen=True)
class DeadLetterEvent:
    task_name: str
    error_type: str
    error_message: str
    attempts: int
    timestamp: float


class TaskSupervisor:
    def __init__(self, max_concurrency: int = 64, max_dead_letters: int = 256) -> None:
        self._semaphore = asyncio.Semaphore(max_concurrency)
        self._active_tasks: Dict[str, asyncio.Task[Any]] = {}
        self._dead_letters: List[DeadLetterEvent] = []
        self._max_dead_letters = max_dead_letters
        self._completed_count = 0
        self._failed_count = 0

    def spawn(
        self,
        name: str,
        coro_factory: Callable[[], Awaitable[Any]],
        policy: Optional[SupervisionPolicy] = None,
    ) -> asyncio.Task[Any]:
        effective = policy or SupervisionPolicy()
        task = asyncio.create_task(self._run(name, coro_factory, effective), name=name)
        self._active_tasks[name] = task
        task.add_done_callback(lambda t, k=name: self._active_tasks.pop(k, None))
        return task

    async def _run(
        self,
        name: str,
        coro_factory: Callable[[], Awaitable[Any]],
        policy: SupervisionPolicy,
    ) -> Any:
        max_attempts = max(1, policy.max_retries + 1)
        last_exc: Optional[BaseException] = None
        for attempt in range(1, max_attempts + 1):
            try:
                async with self._semaphore:
                    coro = coro_factory()
                    res = (
                        await asyncio.wait_for(coro, timeout=policy.timeout_sec)
                        if policy.timeout_sec
                        else await coro
                    )
                self._completed_count += 1
                return res
            except asyncio.CancelledError:
                raise
            except Exception as exc:
                last_exc = exc
                if attempt < max_attempts:
                    await asyncio.sleep(
                        min(policy.max_delay_sec, policy.base_delay_sec * (2 ** (attempt - 1)))
                    )
        self._failed_count += 1
        if last_exc is not None:
            self._dead_letters.append(
                DeadLetterEvent(name, type(last_exc).__name__, str(last_exc), max_attempts, time.time())
            )
            if len(self._dead_letters) > self._max_dead_letters:
                self._dead_letters.pop(0)
        return None
```

---

## Pitfall #2: Slow WebSocket Consumers Exhausting RAM

When streaming real-time audio or partial LLM tokens, calling `await websocket.send_bytes(frame)` directly inside your producer loop couples your upstream pipeline speed to the slowest client's network socket.

Instead, every session should own a **bounded `asyncio.Queue(maxsize=N)`** with a deterministic eviction policy:
- **`drop_oldest` (Ideal for real-time audio / live telemetry):** When the queue is full, evict the oldest stale frame immediately and enqueue the fresh frame so latency stays near zero.
- **`reject` (Ideal for strict command streams):** Reject new frames when the buffer is full and increment a `dropped_frames` counter.

```python
from dataclasses import dataclass, field
from typing import Literal

BackpressureMode = Literal["drop_oldest", "reject"]


@dataclass
class SessionMetrics:
    session_id: str
    enqueued_frames: int = 0
    delivered_frames: int = 0
    dropped_frames: int = 0
    last_activity_ts: float = field(default_factory=time.time)


class WebSocketSessionGateway:
    def __init__(
        self,
        max_queue_size: int = 64,
        backpressure_mode: BackpressureMode = "drop_oldest",
    ) -> None:
        self._max_queue_size = max_queue_size
        self._mode = backpressure_mode
        self._queues: Dict[str, asyncio.Queue[Any]] = {}
        self._metrics: Dict[str, SessionMetrics] = {}

    def register_session(self, session_id: str) -> SessionMetrics:
        self._queues[session_id] = asyncio.Queue(maxsize=self._max_queue_size)
        self._metrics[session_id] = SessionMetrics(session_id=session_id)
        return self._metrics[session_id]

    def push_frame(self, session_id: str, frame: Any) -> bool:
        q = self._queues[session_id]
        m = self._metrics[session_id]
        if q.full():
            if self._mode == "reject":
                m.dropped_frames += 1
                return False
            q.get_nowait()
            q.task_done()
            m.dropped_frames += 1
        q.put_nowait(frame)
        m.enqueued_frames += 1
        return True
```

---

## Verifying Backpressure & Recovery with `pytest`

You can verify that a slow consumer with `max_queue_size=3` deterministically retains only the 3 newest frames (`pcm_chunk_2`, `pcm_chunk_3`, `pcm_chunk_4`) without blocking the event loop:

```python
async def test_ws_gateway_drop_oldest_backpressure() -> None:
    gw = WebSocketSessionGateway(max_queue_size=3, backpressure_mode="drop_oldest")
    gw.register_session("sess-1")
    for i in range(5):
        assert gw.push_frame("sess-1", f"pcm_chunk_{i}") is True

    metrics = gw._metrics["sess-1"]
    assert metrics.enqueued_frames == 5
    assert metrics.dropped_frames == 2
```

---

## Download the Full Production Boilerplate

- **Ko-fi Shop ($5 Instant Download):** [Production FastAPI + WebSocket Real-Time Audio/LLM Kit](https://ko-fi.com/nhi741323)
- **Bonus Tool ($5):** [Deep-Tech Freelance, Bounty & Crypto FIFO/LIFO Tax OS (Excel + Python CLI)](https://ko-fi.com/s/6f070d3e68)
- **GitHub Repository & Release `v1.0.0`:** [duongynhi000005-oss/fastapi-websocket-realtime-ai-starter-kit](https://github.com/duongynhi000005-oss/fastapi-websocket-realtime-ai-starter-kit)
