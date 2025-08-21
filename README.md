# Dynamic Concurrent Pipeline in Go

An educational project implementing a **three-stage concurrent pipeline** with a **dynamically
scaled worker pool**, **graceful shutdown**, and **robust error handling**. Includes an **optional
auto-shutdown on high error rate** feature.

## Features

- Producer → Worker Pool → Collector pipeline
- **Supervisor** that auto-scales workers up/down based on queue backlog
- **Graceful shutdown** on Ctrl+C: producer stops, workers finish current job, results drained
- **Error handling**: per-job failures are logged; the system keeps going
- **Bonus**: auto-shutdown if too many errors happen within a time window
- Configurable via CLI flags

## Quick Start

```bash
# 1) Ensure Go 1.22+ is installed
go version

# 2) Run the demo (varying workload to showcase auto-scale)
go run ./cmd/dynpipe

# Useful flags (with defaults):
#   -min-workers=1 -max-workers=8 -queue-cap=100 -high-water=70 -low-water=20
#   -scale-interval=2s -producer-rate=10 -process-ms=80 -jitter-ms=40 -error-rate=0.05
#   -demo=true -err-window=10s -err-threshold=10
#
# Examples:
# Burst-heavy demo with more frequent scaling checks
go run ./cmd/dynpipe -scale-interval=1s -max-workers=12 -high-water=60 -low-water=15

# Disable demo and run at a fixed producer rate (jobs/sec)
go run ./cmd/dynpipe -demo=false -producer-rate=25

# Make errors more common and lower the auto-shutdown threshold
go run ./cmd/dynpipe -error-rate=0.20 -err-threshold=8 -err-window=5s
```

## Architecture Overview

- **Producer**: generates jobs at a configured or dynamic rate, pushing them into a buffered queue.
- **Supervisor**: periodically checks queue length and scales the **WorkerPool** between
  `min-workers` and `max-workers`. Removes or adds **one worker at a time** for stability.
- **Workers**: each worker processes one job at a time (simulated with sleep + jitter). Each worker
  listens for a **stop signal** and exits **only after** finishing the current job.
- **Collector**: receives `Result`s, logs outcomes, and emits **error events** used for the bonus
  auto-shutdown feature.
- **Graceful shutdown**: on Ctrl+C or auto-shutdown, the program cancels the root context, the
  producer stops and closes the job channel, the supervisor stops all workers, and the collector
  finishes reading results.

## Testing

Two focused tests **(run with `go test ./...`)**:
- `TestSupervisorScaleDecisions`: verifies scale-up/down decisions given fake queue lengths.
- `TestWorkerStopAfterCurrentJob`: ensures a worker completes the current job before stopping.

> Tip: Add `-race` for the race detector.

```bash
go test ./... -race -v
```

## Sample Output (abridged)

```
2025/08/21 12:00:00.123456 [info] worker 1 started
2025/08/21 12:00:02.124001 [info] steady: queue=5 workers=1
2025/08/21 12:00:04.127112 [info] scale up: queue=73 workers=2 (+1)
2025/08/21 12:00:04.130844 [info] worker 2 started
2025/08/21 12:00:05.004004 [collector] job 42 by worker 1 OK in 82ms
2025/08/21 12:00:05.105889 [collector] job 43 by worker 2 FAILED after 71ms: simulated processing error
2025/08/21 12:00:06.310222 [info] scale up: queue=85 workers=3 (+1)
2025/08/21 12:00:06.310777 [info] worker 3 started
...
2025/08/21 12:00:20.500500 [warn] error rate threshold exceeded -> triggering auto-shutdown
2025/08/21 12:00:20.500700 [main] context canceled; stopping producer and supervisor...
2025/08/21 12:00:20.500900 [info] worker 3 stopping (will finish current job)
2025/08/21 12:00:20.501100 [info] worker 2 stopping (will finish current job)
2025/08/21 12:00:20.501300 [info] worker 1 stopping (will finish current job)
2025/08/21 12:00:21.002002 [main] shutdown complete: processed=913 errors=117
```

## Design Notes

- **Why buffered channels?** The job queue’s buffered channel creates measurable back-pressure
  (via `len(jobs)`) for the supervisor.
- **One-by-one scaling** prevents thrashing. You can tune `-scale-interval` and thresholds
  for your environment.
- **Worker stop semantics**: each worker has its own cancelable context; a stop request sets a flag.
  The worker checks the flag after finishing the current job and then exits.
- **Bonus (auto-shutdown)**: the collector emits timestamps for failures; the supervisor keeps a
  simple time-window counter and triggers `CancelRoot()` if the count crosses `ErrThreshold`.

## Project Layout

```
dynpipe/
├── cmd/
│   └── dynpipe/
│       └── main.go
├── go.mod
└── internal/
    └── pipeline/
        ├── collector.go
        ├── job.go
        ├── logger.go
        ├── producer.go
        ├── supervisor.go
        ├── supervisor_test.go
        ├── worker.go
        └── worker_test.go
```

## Troubleshooting

- If you see no scaling, ensure `-queue-cap`, `-high-water`, and `-producer-rate` are tuned so the
  backlog can build.
- On Windows, Ctrl+C should work; on Unix, both Ctrl+C and `kill -TERM` are handled.
- For very fast machines, you may need higher `-producer-rate` or higher `-process-ms` to see scaling.
