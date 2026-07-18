# Benchmarking & Hardware Validation

This turned into a proper investigation rather than a single measurement — worth
documenting the process, not just the final number, since it's a good example of
not trusting one benchmark without corroborating it a second way.

## Baseline: raw model speed

```bash
ollama run qwen2.5:7b --verbose "<prompt>"
```

| Run | Prompt tokens | Output tokens | Eval rate  |
|-----|---------------|---------------|------------|
| 1   | 37            | 731           | 6.47 tok/s |
| 2   | 36            | 474           | 6.84 tok/s |
| 3   | 38            | 32            | 6.88 tok/s |

Consistent ~6.5-6.9 tok/s regardless of output length — a stable, repeatable
number, not a fluke.

This came in lower than an initial estimate of 12-18 tok/s. That estimate was
wrong because it assumed a higher core count than the Xeon Gold 5222 actually
has (**4 cores / 8 threads** — a frequency-optimized, low-core SKU, not a typical
"Gold" tier part). Correction noted here rather than left standing.

## Ruling out thread count and power settings

Tested `OLLAMA_NUM_THREAD` at 8, 4, 2, and 1 — **eval rate stayed at ~6.6-6.8 tok/s
in every case**, including single-threaded. This ruled out thread starvation as
the bottleneck; the workload saturates available memory bandwidth with very few
threads.

Also tested iLO **Power Regulator Mode** (Dynamic Power Savings → Static High
Performance) — no measurable effect on inference speed. Confirmed **Power Capping**
disabled via ROM option — not the cause either.

## Memory bandwidth: STREAM, and where it misled

```bash
sysbench memory ...
gcc -O3 -march=native -fopenmp -DSTREAM_ARRAY_SIZE=100000000 stream.c -o stream_native
OMP_NUM_THREADS=<N> ./stream_native
```

| Threads | STREAM Triad    |
|---------|-----------------|
| 8       | ~48.3 GB/s      |
| 4       | ~47.5 GB/s      |
| 1       | ~16.4-17.7 GB/s |

Bandwidth saturates almost immediately (4 threads ≈ 8 threads) — consistent with
CPU inference being memory-bandwidth-bound generally.

**The 1-thread number initially looked contradictory**: if inference needed ~48GB/s
to hit 6.8 tok/s, the 1-thread case (16.4GB/s available) should have dropped speed
to roughly a third. It didn't — single-threaded `ollama run` stayed at ~6.55 tok/s.

Root cause: **the STREAM binary was never actually running AVX-512 code**, despite
`-march=native`. Confirmed directly:

```bash
objdump -d stream_native | grep -c 'zmm'   # → 0
```

GCC defaults to 256-bit vector width even when AVX-512 is available and requested
via `-march=native`; it requires an explicit override (`-mprefer-vector-width=512`)
to actually emit `zmm`-register instructions. A scalar/narrow-vector single thread
can't keep enough memory requests in flight to reflect the CPU's real single-core
bandwidth, so the STREAM 1-thread number understated what llama.cpp's hand-tuned
AVX-512/VNNI kernels can actually achieve per thread. STREAM was the flawed
measurement, not Ollama.

## Ground truth: measuring real DRAM traffic during actual inference

Rather than trust a synthetic proxy, measured live memory-controller traffic
during an actual `ollama run` with Intel PCM:

```bash
git clone --recursive https://github.com/intel/pcm.git && cd pcm
mkdir build && cd build && cmake .. && make -j4
sudo ./pcm-memory 1
```

**Result: 29.2 GB/s sustained System Memory Throughput during real generation.**

Cross-check: 29.2 GB/s ÷ 6.6-6.8 tok/s ≈ **4.3-4.5 GB per token** — matches almost
exactly the Q4_K_M weight file size for this model (4.36 GiB). Confirms the
number cleanly: **this is a genuine, hardware-grounded memory-bandwidth ceiling**,
not a misconfiguration.

## Power draw and thermal behavior

Full-load system power stayed flat around 150-161W across multiple different
workloads (Ollama inference, default `stress-ng --cpu`), with fans barely
spinning (9-20%) and CPU temps maxing at 40°C. Initially looked suspicious —
turned out to be expected:

- **Xeon Gold 5222 TDP is 105W** for the whole package. Add ~30-45W for RAM,
  spun-up drives, board, and PSU losses, and 150-160W at full CPU load is
  exactly the expected number, not a sign of an artificial cap.
- Default `stress-ng --cpu` cycles through many stress methods, most of which
  don't heavily exercise AVX-512 — so it under-reported real achievable power
  draw. Forcing a vector-heavy method showed the difference:

  ```bash
  stress-ng --cpu 8 --cpu-method matrixprod --timeout 60s --metrics-brief
  ```

  This drew **74W** (package) — confirmed via `objdump` that the installed
  `stress-ng` binary does contain real AVX-512 instructions (11,418 `zmm`
  references), ruling out a compiler-flag issue like the STREAM case. This is
  Intel's documented automatic AVX-512 frequency/power throttling doing its job —
  the chip self-regulates below rated TDP under sustained heavy-vector load by
  design. TDP is a theoretical ceiling; ~74W is closer to the real sustained
  number for genuinely heavy vector work on this part.

## Conclusion

**6.6-6.8 tok/s is the validated, real ceiling for Qwen 2.5 7B (Q4_K_M) on this
CPU**, confirmed two independent ways (thread-count invariance + direct DRAM
traffic measurement matching the per-token weight-size math). Nothing about
BIOS power settings, thread count, or CPU utilization changes this number — it's
a genuine memory-bandwidth wall, not a misconfiguration.

## Session-level cost: system prompt overhead

Separately from raw model speed, Hermes Agent's first message in a new session
pays a real, measurable tax: with 18 tools enabled, the injected system prompt
(persona + full tool definitions) ran to **~11,900 tokens**, taking **~9 minutes**
of prompt-processing time before the first reply appeared — because prefill
(processing many tokens as a batch) is compute-bound rather than
bandwidth-bound, this showed as high CPU utilization with low DRAM bandwidth
(~1.7GB/s), which looked wrong until the prefill/decode distinction was worked
through. This cost is paid once per session (cached afterward — a second
message answered in ~3.3 seconds) but recurs on every new session start, and
also raises steady-state generation cost for the life of the session, since
every generated token attends back over the full cached context.

**Action taken:** trimmed enabled tools from 18 down to 6, directly shrinking
this baseline. Re-benchmarking with the smaller tool set is the next step.
