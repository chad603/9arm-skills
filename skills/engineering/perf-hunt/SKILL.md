---
name: perf-hunt
description: Four-step performance discipline — define the figure of merit, get a stable measurement, attribute the cost to a layer and a resource, then change one thing and prove the delta beats the noise. The correctness sibling of debug-mantra, for the other half of systems work: "it's too slow / it regressed / it's below roofline." Trigger on /perf-hunt and proactively whenever the work is about speed — user says something is slow/slower/regressed, asks to optimize/speed up/profile/benchmark, reports a throughput/latency/bandwidth/utilization number that's below target, or hands you a perf regression to chase.
---

# Perf Hunt

Four-step discipline for any performance investigation — regression-chasing or fresh optimization. The sibling of [`debug-mantra`](../debug-mantra/SKILL.md): same shape, different failure mode. A correctness bug is *wrong output*; a perf bug is *a number that's worse than it should be*. The traps are different, so the discipline is different. Apply the four steps in order before you touch a single line.

## The four steps — apply in order

> **Perf mantra:**
> 1. **Name the number.** What metric, what workload, what target? No target → no hunt.
> 2. **Make it stable.** Measure a distribution, not a point. If noise > the effect, you can't see it yet.
> 3. **Attribute, don't guess.** Profile first. Which layer, which resource — and is it big enough to matter?
> 4. **One change, proven delta.** A/B with everything else fixed. The win is real only end-to-end, above the noise.

---

## 1. Name the number

A perf investigation without a figure of merit is a fishing trip.

- **Pick the metric.** Latency (p50 *and* p99 — they move differently), throughput (req/s, tok/s, GB/s, GFLOP/s), end-to-end (time-to-train, time-to-first-token, cost per unit). One primary metric. Name it.
- **Pin the workload.** The exact input, shape, scale, and config that produces the number. "It's slow" is not a workload; "8-GPU all-reduce, 256 MB message, NVLink" is.
- **Set the target and the stop condition.** What's "good enough" — a regression baseline to recover, a roofline fraction to reach, an SLA to clear? Know when you stop *before* you start, or you'll gold-plate a slice nobody cares about.
- **Know the theoretical ceiling.** Peak bandwidth, peak FLOPs, the roofline for this op. A measured speedup that exceeds the ceiling is a measurement bug, not a win — and the ceiling tells you how much headroom is even on the table.

## 2. Make it stable

Before optimizing anything, make the number reproducible. This is the perf analogue of `debug-mantra`'s reliable repro — and the step people skip most.

- **Warm up, then measure steady state.** Discard cold-start, JIT, cache-fill, autotune, and first-iteration outliers. Measure the regime you actually care about.
- **Control the environment.** Lock GPU/CPU clocks (no boost, no thermal throttle), isolate the machine (no co-tenant load), fix the input and seed, control cache state. Uncontrolled clocks alone can swing a number 30%.
- **Take a distribution, not a point.** N runs. Report median + spread (p99, stddev, min/max). A single run is an anecdote.
- **Establish the noise floor.** Run the *same* config twice and measure the run-to-run variance. That variance is the smallest effect you can trust. If the regression or the win you're chasing is smaller than the noise floor, **you cannot measure it yet** — reduce the noise before you change anything.

## 3. Attribute, don't guess

"I think it's slow here" is a hypothesis, not data. Profile before you touch a line.

- **Profile first, top-down.** End-to-end → which phase → which function/kernel → which resource. Use the right tool (`nsys`/`ncu`, `perf`, flamegraph, roofline plot, a tracer) — don't read the source and guess.
- **Attribute to a resource.** Is it compute-bound, memory-bandwidth-bound, latency-bound, sync/idle-bound, or host-bound? The bottleneck class dictates the fix; optimizing the wrong axis does nothing. A kernel at 40% of peak bandwidth doesn't get faster by reducing FLOPs.
- **Amdahl check — is it worth it?** Size the slice in the *end-to-end* budget. A 10× speedup on a kernel that's 3% of runtime is a 0.3% win. Find the slice that's both slow *and* large before committing.
- **Beware the instrument.** Profiler overhead and sampling skew lie. Cross-check a profiler claim against wall-clock, or against a counter. Profiling that perturbs the thing it measures isn't measuring your workload.

## 4. One change, proven delta

When a candidate fix surfaces, prove it the same way `debug-mantra` falsifies a hypothesis — adversarially.

- **Change one thing.** A/B with everything else frozen. Two changes at once and you can't attribute the delta.
- **Re-measure with step 2's rigor.** Same warmup, same clocks, same N, same distribution. A win measured loosely is not a win.
- **The win is real only if all hold:** delta > noise floor; the gain shows up in the **end-to-end** number, not just the micro-benchmark; the target workload improves; and no other workload regresses. A local speedup that doesn't move the end-to-end number is not a win — it's a story.
- **Check against the roofline.** If your "speedup" pushed past the theoretical ceiling, you measured an artifact (skipped work, cached result, dead-code-eliminated benchmark). Go find what you actually changed.
- **Keep a ledger.** Every experiment: what changed, the before/after distribution, what it ruled in or out. This is your memory across the session and your raw material for the `post-mortem`.

---

## When to invoke

- `/perf-hunt`
- "this is slow / slower / regressed / fell off a cliff"
- "optimize / speed up / make this faster / reduce latency / raise throughput"
- "profile this / benchmark this / why is this below peak"
- A reported number under target: bandwidth, latency, utilization, tok/s, time-to-train.
- Proactively when a debug or review session turns out to be about *speed*, not *correctness*.

## When NOT to use

- **Correctness bug.** Wrong output, hang, crash, race → that's [`debug-mantra`](../debug-mantra/SKILL.md). (A hang *looks* like infinite slowness but it's a correctness bug — wrong sync, not a slow kernel.)
- **No way to measure.** If you can't run the workload or get a number, this skill can't start. Say so; ask for access or a repro harness. Do not optimize by inspection.
- **The number already meets target.** If it's at goal, stop. Don't manufacture an optimization for ceremony — record the headroom and move on.

## Operating rules

- **Never optimize without a profile.** Source-reading produces hypotheses; profiles produce facts. Step 3 before step 4, always.
- **Never trust a single run.** Distribution or it didn't happen. A point estimate hides the variance that decides whether your win is real.
- **Never claim a delta smaller than the noise floor.** "5% faster" on a workload with 8% run-to-run variance is noise wearing a result's clothes.
- **Always measure end-to-end, not just the micro-benchmark.** Amdahl is undefeated. The figure of merit from step 1 is the only number that counts.
- **Lock the environment or your numbers are fiction.** Unpinned clocks, co-tenant load, and cold caches have killed more "regressions" and "wins" than real code ever did.
- **Know the roofline.** A win above the theoretical ceiling is a measurement bug. The ceiling also tells you when to stop.
- **Stop at the target.** When you hit the figure of merit, stop and record the remaining headroom for future-you. Don't chase diminishing returns nobody asked for.
- **The discipline is a constraint you carry, not advice to hand back.** Run the four steps; report findings, not the mantra.

## Worked example — all-reduce below NVLink bandwidth (JIRA-23110)

> **1. Name the number.** Figure of merit: bus bandwidth of `tadaKernel_AllReduce_f32_RING` on the standard 8-GPU NVLink node, 256 MB message, FP32. Measured 142 GB/s; the ring algorithm's ceiling on this topology is ~230 GB/s (busBw model). Target: ≥ 90% of ceiling (~207 GB/s). Stop there.
>
> **2. Make it stable.** Locked GPU clocks (`nvidia-smi -lgc`), single-tenant node, 50 iterations after a 10-iteration warmup, reported median + p99. Noise floor (same config, twice): ±1.8%. The 38% gap to ceiling is far above noise — measurable.
>
> **3. Attribute.** `ncu` showed the kernel at 61% of peak NVLink bandwidth but only 22% of compute — memory/transfer-bound, not compute-bound. `nsys` timeline showed the transfer split into 512 small chunks with per-chunk launch latency dominating; the ring was latency-bound at this chunk size, not bandwidth-bound. End-to-end check: this all-reduce is 31% of step time, so a real win here moves the workload — worth it.
>
> **4. One change, proven delta.** Hypothesis: chunk count too high → coalesce to 64 chunks, nothing else changed. Re-measured with step 2's rigor: 142 → 211 GB/s median (p99 209), a 49% gain, 6× the noise floor. End-to-end step time dropped 11%, matching the 31%-slice prediction. No regression on the small-message path (re-ran 4 MB config: unchanged). 211 GB/s is 92% of the 230 ceiling — past target, under the roofline. Stop.
>
> **Ledger:** chunk=512 → 142; chunk=256 → 178; chunk=64 → 211; chunk=32 → 207 (latency win exhausted, slight bandwidth loss). 64 is the knee. Recorded the remaining 8% headroom as not worth the complexity.

What this hunt did that a guess wouldn't: it found the bottleneck was *latency*, not *bandwidth*, before changing anything — so it didn't waste a day shaving FLOPs off a transfer-bound kernel. It proved the win end-to-end (11% step time), not just in the micro-benchmark (49% kernel). And it stopped at the knee instead of chasing the last 8%.

## Handoffs

- **Found a regression with a root cause and a validated fix?** Hand the ledger to [`post-mortem`](../post-mortem/SKILL.md) — your before/after distributions are its Validation section, your attribution is its Root cause.
- **Need to report the win upward?** Hand the result to [`management-talk`](../productivity/management-talk/SKILL.md) — it keeps the workload name and the end-to-end number, strips the kernel identifiers and chunk counts.
- **Reviewing someone else's optimization PR?** That's [`scrutinize`](./scrutinize/SKILL.md) — and the first thing to demand is the proven-delta evidence from step 4.
