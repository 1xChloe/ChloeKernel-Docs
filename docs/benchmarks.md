## Benchmarks

Measured via `Bench.runAll` on kernel hot paths:

| Operation | Throughput | p50/op |
|---|---|---|
| TokenBucket:take | 17.0M ops/s | 58ns |
| Hooks:fire (3-validator chain) | 6.6M ops/s | 148ns |
| Bus:publish | 5.0M ops/s | 198ns |
| Signal:fire (1 handler) | 4.5M ops/s | 203ns |
| Scheduler schedule+step (production) | 2.2M ops/s | 354ns |
| Scheduler schedule+step (Studio, MicroProfiler zones) | 2.1M ops/s | 388ns |
| Serde schema encode (3 fixed fields, fast engine) | 3.5M ops/s | 285ns |
| Serde schema decode (3 fixed fields, fast engine) | 2.7M ops/s | 367ns |
| Serde struct delta encode (2 fields) | 3.7M ops/s | 270ns |
| Serde struct delta decode (2 fields) | 3.4M ops/s | 293ns |

The fast engine's same-machine A/B against the wire-engine path: fixed-schema encode 463ns to 285ns (38% faster), decode 514ns to 367ns, dynamic schemas 22-27% faster — identical bytes either way.

The full intent security pipeline (rate limit + session check + 3 validators) costs well under a microsecond per packet. The always-on hot-task profiler (per-task timing attributed to script:line) costs **~33ns per task** — origins are cached by function identity, and MicroProfiler zones run only in Studio unless a live server opts in via `Scheduler.setMicroProfilerZones(true)`.
