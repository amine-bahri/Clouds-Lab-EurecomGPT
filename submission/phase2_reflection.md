# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->574.706<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.024<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->444630.2<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12690.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5468<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2307.415 ms, 2 threads: 2016.666 ms, 4 threads: 2894.594 ms, 8 threads: 2662.488 ms. Fastest: 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Both versions do the same math, but the naive one runs it as about 131,000 small Python loop iterations, each calling NumPy on a 64-number vector. The actual multiplications are tiny, so most of the 574.7 ms is Python overhead. The vectorized version gives whole matrices to the BLAS library, which runs compiled code with SIMD instructions that handle several numbers at once. That took 1.02 ms, about 561x faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The fastest was 2 threads (2016.7 ms), but that is only 13% better than 1 thread (2307.4 ms). Colab gives 2 vCPUs, but lscpu shows they are two hyperthreads of the same physical core, so they share the same hardware and can't really double the work. With 4 and 8 threads there are more threads than CPUs, so they fight over the core and wait for each other, and it got slower (2894.6 and 2662.5 ms).
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Training is mostly big matrix multiplications, which can be split into lots of small independent calculations, and a GPU has thousands of cores to do them at the same time. With the same 100 steps it took 444,630 ms on the CPU and 12,690.7 ms on the GPU, about 35x faster. A task that would not benefit much is something small and sequential, like my naive attention loop, because the GPU can't speed up Python running one small step at a time.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment stopped for a different reason. With threads, the limit was hardware: Colab only has one real core, so 2 threads gave just 13% (2307 to 2017 ms), and going further made it worse because the threads spent their time competing and coordinating: 4 threads was 44% slower than 2. Vectorization was not really a parallelism limit, the 561x mostly came from removing Python overhead and using SIMD inside one core. The GPU gave the biggest gain (35x), but our model is small, so each step is many small operations and the GPU is not fully used. Some parts also stay sequential, like the Python code and waiting for the loss at every step. That is Amdahl's law: that part does not get faster with more hardware. Moving data was not a problem here because each batch is small. One caveat: the CPU run used 1 thread, so the fair speed-up is closer to 31x.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would rerun the CPU vs GPU test with a bigger batch size. If the GPU speed-up gets much bigger, it means the GPU was not getting enough work with our small model. If it stays around 35x, the limit is something else.
<!--/answer-->
