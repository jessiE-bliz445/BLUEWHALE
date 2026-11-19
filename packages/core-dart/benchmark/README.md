# Dart AOT vs JIT Benchmark — Issue #449

Measures throughput and per-call latency of core Bluewhale operations in both
Dart JIT and AOT compilation modes on ARM64 mobile targets.

## Why AOT vs JIT?

Dart ships two compilation models for mobile:
- **JIT** (VM): Used during development and by `dart run`. Includes adaptive
  recompilation, so the first few hundred calls pay a compile cost.
- **AOT**: Used in release Flutter builds and compiled server binaries. All
  code is compiled to native machine code before execution, eliminating JIT
  overhead at the cost of a larger startup binary.

ARM64 is the primary architecture for modern Android and iOS devices. AOT
typically shows lower tail latency (p99) and higher peak throughput than JIT
for tight loops because there is no JIT recompilation pause. However, AOT
cannot speculate on call-site types, so certain polymorphic dispatch patterns
can be faster under JIT. This benchmark measures which effect dominates for
the Bluewhale's core hot paths.

## Operations benchmarked

| Benchmark | Hot path |
|-----------|----------|
| `muxed_encode` (id=42, canary, max) | Base32 encode + CRC-16 write |
| `muxed_decode` (id=42, canary) | Base32 decode + CRC-16 verify + BigInt extract |
| `detect` (G, M, C, lowercase, invalid) | StrKey prefix + CRC-16 dispatch |
| `extractRoutingSync` (G + various memoTypes) | Address parse + memo normalise |
| `extractRoutingSync` (M-address) | Muxed decode + warning accumulator |
| Round-trip encode → decode → route | Full pipeline |

## Running the benchmark

### JIT mode (Dart VM)

```bash
cd packages/core-dart
dart pub get
dart run benchmark/aot_vs_jit_benchmark.dart
```

### JIT mode — JSON output (for CI comparison)

```bash
dart run benchmark/aot_vs_jit_benchmark.dart --json > results-jit.json
```

### AOT mode — compile then run

```bash
# Compile to a native binary (requires Dart SDK ≥ 2.12)
dart compile exe benchmark/aot_vs_jit_benchmark.dart -o /tmp/bw_bench_aot

# Run the compiled binary
/tmp/bw_bench_aot
# or with JSON output:
/tmp/bw_bench_aot --json > results-aot.json
```

### ARM64 CI runner (GitHub Actions example)

```yaml
jobs:
  dart-aot-vs-jit-arm64:
    runs-on: ubuntu-24.04-arm  # GitHub-hosted ARM64 runner
    steps:
      - uses: actions/checkout@v4
      - uses: dart-lang/setup-dart@v1
        with:
          sdk: stable
      - name: Install dependencies
        run: dart pub get
        working-directory: packages/core-dart
      - name: Run JIT benchmark
        run: dart run benchmark/aot_vs_jit_benchmark.dart --json > /tmp/jit.json
        working-directory: packages/core-dart
      - name: Compile AOT binary
        run: dart compile exe benchmark/aot_vs_jit_benchmark.dart -o /tmp/bw_bench
        working-directory: packages/core-dart
      - name: Run AOT benchmark
        run: /tmp/bw_bench --json > /tmp/aot.json
      - name: Print comparison
        run: |
          echo "=== JIT ===" && cat /tmp/jit.json | jq '.benchmarks[] | {benchmark, throughputOpsPerSec, latency}'
          echo "=== AOT ===" && cat /tmp/aot.json | jq '.benchmarks[] | {benchmark, throughputOpsPerSec, latency}'
```

## Interpreting results

The benchmark reports per-operation:

| Field | Description |
|-------|-------------|
| `throughputOpsPerSec` | Operations completed per second |
| `latency.meanUs` | Average per-call latency in microseconds |
| `latency.p99Us` | 99th-percentile latency (tail latency) |
| `latency.maxUs` | Worst single call observed |

**Expected AOT vs JIT characteristics on ARM64:**
- AOT throughput typically 10–40% higher for small, tight loops (muxed
  encode/decode, detect) because there is no JIT warm-up overhead.
- AOT p99 latency is usually lower for the same reason.
- JIT can occasionally match or beat AOT for heavily polymorphic dispatch
  thanks to speculative inlining. The routing extraction benchmarks may
  show this effect.
- Both modes should produce **identical results** for all inputs — if they
  differ the implementation has a correctness bug triggered by the compiler.

## Output format

### Text (default)

```
════════════════════════════════════════════════════════════════════════════
  Bluewhale Dart Benchmark  —  AOT  —  ARM64
  Dart SDK: 3.4.0        OS: android
  Warm-up: 500 iters   Timed: 10000 iters per benchmark
════════════════════════════════════════════════════════════════════════════

  Mode: AOT

  muxed_encode(id=42)                       throughput:     850000 ops/s   mean:    1µs   p99:    3µs   max:   12µs
  muxed_encode(id=2^53+1 canary)            throughput:     820000 ops/s   mean:    1µs   p99:    3µs   max:   11µs
  ...
```

### JSON (--json flag)

```json
{
  "meta": {
    "mode": "AOT",
    "arch": "ARM64",
    "dartVersion": "3.4.0",
    "os": "android",
    "warmupIterations": 500,
    "timedIterations": 10000,
    "timestamp": "2025-11-01T12:00:00.000Z"
  },
  "benchmarks": [
    {
      "benchmark": "muxed_encode(id=42)",
      "iterations": 10000,
      "totalTimeMs": 11,
      "throughputOpsPerSec": 909090,
      "latency": { "meanUs": 1, "minUs": 1, "maxUs": 12, "p99Us": 3 }
    },
    ...
  ]
}
```
