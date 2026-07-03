# Mini CPU Profiler

A low-level CPU profiler written in C, measuring function execution time in **CPU cycles** (not wall-clock time) using the `RDTSC` / `RDTSCP` instructions.

> **Status**: work in progress.

## Motivation

Modern CPUs execute instructions **out-of-order** and speculatively. Standard timers (`clock()`, `gettimeofday`, even `clock_gettime`) measure wall-clock time, which is coarse and doesn't reflect what the CPU pipeline is actually doing at the instruction level, a single cache miss or branch misprediction can be invisible at that resolution.

Reading the CPU's own Time Stamp Counter (`RDTSC`) gives cycle-accurate measurements, which is the level of precision needed to observe cache misses, TLB misses, or branch mispredictions directly rather than inferring them indirectly.

## How it works

### 1. Serialized cycle counting (`start_tsc` / `stop_tsc`)

`RDTSC` alone isn't reliable for microbenchmarking, because out-of-order execution can let surrounding instructions run before or after the actual `RDTSC` read, skewing the measurement.

- **`start_tsc()`**: issues `lfence` (load fence) before `rdtsc`, blocking speculative execution from crossing the fence, ensuring nothing from *before* the measurement leaks in.
- **`stop_tsc()`**: uses `rdtscp` instead of `rdtsc`. `rdtscp` is itself partially serializing (it waits for all prior instructions to complete before reading the counter), followed by an `lfence` to prevent instructions *after* the read from being reordered earlier.

Both return the 64-bit counter reassembled from the `EDX:EAX` register pair returned by the instruction.

### 2. Measuring a function once (`measure_once`)

Wraps a call to `fn()` between `start_tsc()` and `stop_tsc()`, returning the raw cycle delta for a single execution.

### 3. Measuring a function N times (`measure_n`)

A single measurement is noisy, interrupts, context switches, or thermal throttling can spike an individual reading. `measure_n`:
1. Runs `fn()` once as a **warm-up** (so the function's code/data are already in cache before any measurement starts, avoiding a cold-cache penalty skewing the first sample).
2. Runs `measure_once(fn)` `n` times, storing each result.
3. Sorts the results with `qsort` and returns the **median**, more robust to outliers than a mean would be.

## Build & run

```bash
gcc -O2 -o profiler main.c profiler.c
./profiler
```

Example output (values will vary by CPU/system):
```
Measure Once: XXXX cycles
Measure N: XXXX
```

## What this demonstrates

- Direct use of CPU instructions (`RDTSC`/`RDTSCP`) via inline Assembly, including understanding *why* naive `RDTSC` usage is unreliable and how instruction serialization (`lfence`) fixes it.
- Awareness of out-of-order execution and its practical impact on measurement, not just using a timer, but understanding what could make that timer lie.
- Statistically sound benchmarking practice: warm-up pass + median over multiple samples, rather than trusting a single reading.
- Clean separation between the measurement engine (`profiler.c`/`.h`) and the benchmarked code (`main.c`), via a function-pointer interface (`bench_fn`).

## Known limitations

- **No CPU affinity / pinning.** The TSC can behave differently across cores on some systems, and the OS scheduler is free to migrate the thread mid-benchmark. Without pinning (e.g. `sched_setaffinity`), measurements can be affected by which core the thread happens to run on.
- **No control over frequency scaling or interrupts.** CPU frequency scaling (turbo boost, power-saving states) and unrelated interrupts can still affect individual samples, even with the median mitigating the worst outliers.
- **`test_fn` in `main.c` is a placeholder** (an empty `volatile` loop), not yet used to benchmark something architecturally interesting like a cache miss or a branch misprediction.
- **No CSV export yet** results are currently printed to stdout only, so runs can't be compared over time or plotted (see Future Work).

## Future Work

- Warm-up mode as a configurable option (currently a single fixed warm-up call).
- Additional benchmark targets: pointer chasing, dependency/latency chains, branch misprediction patterns.
- CSV export for plotting results across configurations, similar to the [microarch-pointer-chasing](https://github.com/AledMikuinIt/microarch-pointer-chasing) project.
