---
title: "Performance"
weight: 10
description: "Head-to-head against FFTW, pocketfft and gonum, correctness-gated."
---
`go-fft` is benchmarked head-to-head against the implementations that matter, on
the same machine, with the same inputs and sizes:

* **FFTW 3.3.10**, built from source with the CPU's SIMD and called from C, the
  gold standard, single-threaded with a reused `FFTW_MEASURE` plan.
* **pocketfft** via **`numpy.fft` 2.5.3** and **`scipy.fft` 1.18.1**,
  single-threaded.
* **[`gonum/dsp/fourier`](https://pkg.go.dev/gonum.org/v1/gonum/dsp/fourier)
  v0.16.0**, the other pure-Go (`CGO_ENABLED=0`) FFT.

Correctness is gated first: every go-fft transform must match `numpy.fft` within
`rtol=1e-9, atol=1e-7` before any timing is reported.

> **Correctness, not speed:** v0.25.0 to v0.26.2 return wrong N-D results
> (`FFTN`, `FFT2`, `PlanN` and their inverses), silently, on Intel CPUs with
> AVX-512, when an axis other than the last has a length 9·p (153, 171, 207, …).
> Fixed in v0.26.3; use v0.26.3 or later. See the
> [v0.26.3 release notes](https://github.com/go-fft/fft/releases/tag/v0.26.3).

The numbers below describe **v0.28.6** on Zen 3, **v0.28.11** on Cascade Lake
and **v0.28.7**'s code on Neoverse-N1. Each host's numbers are the newest parity runs available. v0.24.0 (Round 37) changed
only Haswell's and arm64's paths. v0.25.0 (Round 35), v0.26.3 (the N-D
correctness fix) and v0.27.0 (Round 39) changed only Intel AVX-512's. v0.26.0
(Round 36) changed only paths taken where the CPU is not Intel's. v0.26.1,
v0.26.2 and v0.26.4 changed no library code. Measured on three GCC Compile
Farm hosts with go1.27.1, every row on one pinned core, all on 2026-10-11:

* **Neoverse-N1**: re-measured on 2026-11-01 at the code of v0.28.7 (two pinned
  runs; Round 40's column, the v0.26.1 tag, is the previous one);
* **Cascade Lake**: re-measured on 2026-11-07 at the v0.28.11 tag (four runs:
  default, FMA, FMA, default; the previous column was Round 39's, v0.27.0);
* **Zen 3**: re-measured on 2026-10-31 at the v0.28.6 tag (four runs: default,
  FMA, FMA, default; the default column uses the two default runs;
  [`amd64-zen3-epyc7773x-20261031-v0.28.6/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/amd64-zen3-epyc7773x-20261031-v0.28.6/)).

Cascade Lake's host had been under a one-minute load of 5–11 on 8 cores until
2026-11-01 and was measured once it fell to 0.00. Apple M4 Max was measured against FFTW for the first time on 2026-11-02
(v0.28.9; see the last bullet of "Honest read"). What those runs do not
contain is said under each host.

FFTW's own times move by up to 47% between runs on one host, so no single run is
quoted. go-fft's time is the median of the three repetitions in each run, then
the mean of the two runs named under each host; FFTW's is the median over the
round's runs: four on Cascade Lake and Zen 3 (each of those rounds ran main,
branch, branch, main), two on Neoverse-N1 (Round 40 ran the tag twice); numpy
and scipy are the mean of the same two runs as go-fft. The tables are computed
from the raw files by
[`current-numbers-v0.27.0/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/current-numbers-v0.27.0/), which also lists
each run's own value and the command that produced each table. The raw runs are in
[`benchmarks/results/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/), and the dated optimization
rounds (what was tried, kept or dropped, and why) are in
[BENCHMARKS.md](https://github.com/go-fft/fft/blob/main/BENCHMARKS.md). Reproduce them with `benchmarks/run.sh`
(`benchmarks/remote/` for a Linux host without Go).

**v0.28.0** changes no default path: default outputs are bit-identical to
v0.27.x, so the tables below stand for it. It adds the opt-in FMA plans, whose
column was measured separately (next section).

## Opt-in FMA (`Options.FMA`, v0.28.0), measured

Round 42 (2026-10-18/19, BENCHMARKS.md), the branch that became v0.28.0, two
default and two FMA parity runs per host (`GOFFT_FMA=1`), go-fft time ÷ FFTW
time:

| row | Zen 3 default | Zen 3 FMA | Cascade Lake default | Cascade Lake FMA |
|:--|--:|--:|--:|--:|
| complex 256 | 1.102 | 1.063 | **1.048** | **0.962** |
| complex 1,024 | 1.129 | 1.096 | 1.189 | **0.939** |
| complex 4,096 | **0.893** | **0.890** | **0.957** | **0.945** |
| RFFT 1,024 | **1.033** | **0.967** | **0.790** | **0.736** |
| RFFT 4,096 | 1.060 | **1.045** | **1.032** | **0.940** |
| rows ≤ 1.05, of 24 | 19 | 20 | 17 | 18 |

In process against the default plan on 41 rows the FMA plans were never slower
beyond noise (lowest 0.990 Cascade Lake, 0.991 Zen 3); where fusing lost
(Cascade Lake 4,096–16,384 points) they run the default kernels. Accuracy
against long-double scipy: 0.76–1.08× the default's error. Haswell and other
amd64 CPUs were not measured. These are separate runs from the default tables
below, so their default columns differ from them by the hosts' run-to-run
spread.

Cascade Lake re-measured on 2026-11-07 at the v0.28.11 tag (the same
procedure; the table of its section below holds the default column): with FMA
it reads 19 of 24 within 5% and faster on 15, complex 1,024 at 1.026 (1.24
default) and complex 256 at 0.986 (1.065).

## Method

* **One pinned core** (`taskset` to a single core, so `GOMAXPROCS` is 1),
  the 2-D rows included: go-fft's multicore N-D path has one thread here, like
  FFTW, numpy and scipy.
* **Steady state**: every library reuses its plan, and go-fft writes into a
  reused slice (`Plan`, `RealPlan`, and `PlanN` for 2-D). The 2-D ratios are
  `PlanN` against FFTW, not the allocating `FFT2` call, which the per-run
  reports in the raw data also give, with GFLOP/s.
* ns/op, lower is better. The ratio is go-fft ÷ FFTW (≤ 1.05 = parity, which
  includes rows slower than FFTW by at most 5%; every such row is named).

## How go-fft transforms a length

* **Lengths whose prime factors are all ≤ 13**: an iterative mixed-radix
  **Stockham** engine, with radix-8/4/2/3/5/7 passes plus a general radix-p pass.
  On amd64 its radix-2/3/4/5/8 passes run as **AVX2** kernels, and as **AVX-512**
  kernels for powers of two of 256 points or more. Both are bit-identical to the
  Go passes.
* **A prime whose N−1 is 7-smooth**: **Rader's algorithm**. **Any other length**:
  **Bluestein's chirp-z**. Both convolve on the Stockham engine.
* **Powers of two on loong64, s390x and amd64 without AVX2, and above 65536
  on riscv64**: an
  iterative, cache-blocked radix-4 kernel.
* On arm64 and the other non-amd64 targets the passes are Go code, compiled to
  scalar instructions. gc does not vectorize; it does fuse multiply-adds.

## The three hosts at a glance

go-fft time ÷ FFTW time; below 1 means go-fft is faster. Bold: at or above FFTW,
which by the project's convention includes rows up to 1.05, slower than FFTW by
at most 5%. "Faster" below means strictly below 1. Three decimals where two
would hide which side of 1 or 1.05 a row is on.

| | Zen 3 | Neoverse-N1 | Cascade Lake |
|:--|--:|--:|--:|
| at or above FFTW (≤ 1.05), of 24 | 18 | 24 | 15 |
| of which faster than FFTW | 16 | 24 | 11 |
| at or above numpy.fft, in both runs, of 24 | 24 | 24 | 24 |
| at or above scipy.fft, in both runs, of 24 | 24 | 24 | 24 |
| faster than gonum, in both runs, of 19 (1-D) | 19 | 19 | 19 |

The 24 rows are the complex 1-D, real 1-D and 2-D tables below (11 + 8 + 5); the
IRFFT rows are not counted, as in every round.

## AMD EPYC 7773X (Zen 3), AVX2

cfarm420, a 128-thread VM shared with other users (1-minute load 6.2–7.8 during
the parity runs, from five other users' single-thread jobs; the pinned core
was idle); FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. The v0.28.6
tag, 2026-10-31, four runs in the order default, FMA, FMA, default: go-fft is
the mean of the two default runs, FFTW the median of all four.

- **Within 5% but slower than FFTW:** RFFT 1,024 (1.030) and RFFT 1,080
  (1.014).
- **Slower by more:** complex 256 (1.07), 1,024 (1.13), 1,080 (1.10); RFFT 256
  (1.075), 4,096 (1.061), 1,920 (1.057).
- **Against Round 36's column (v0.26.0):** four rows moved by more than 5% in
  ratio: complex 65,536 (0.87 → 0.92), RFFT 1,000 (0.89 → 0.98), RFFT 1,920
  (0.991 → 1.057, now over 1.05) and 2-D 512² (0.991 → 0.94). The host was
  busier than in Round 36, so no release is blamed.
- **Complex 1,024 stays 1.08–1.13× FFTW** on Zen 3. Rounds 43 and 44 (branch
  `perf-zen1024-fmapipes`, not merged) show an IPC deficit of the twiddled
  radix-32 pass, not instruction count, placement or arithmetic.
- **With `Options{FMA: true}`** the 2026-10-31 FMA runs count 18 of 24 within
  5% and 16 faster (complex 1,920 at 1.052, RFFT 4,096 at 1.053 and RFFT 256 at
  1.060 just over the line); a second pair of FMA runs on 2026-11-01 counts 20
  of 24 and 17 faster (RFFT 4,096 at 1.063 still out). Round 42 counted 20.
  FMA on Zen 3 is 18–20 of 24; three rows sit within 0.01 of the line.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 306 | 287 | 1.07 | 6,714 | 5,203 |
| complex 1,024 | 1,734 | 1,540 | 1.13 | 12,796 | 9,461 |
| complex 4,096 | 9,415 | 10,216 | **0.92** | 44,761 | 33,293 |
| complex 65,536 | 269,300 | 293,325 | **0.92** | 1,960,558 | 912,471 |
| complex 1,048,576 | 5,570,234 | 11,208,976 | **0.50** | 22,308,717 | 13,750,842 |
| complex 1,000 (2³·5³) | 2,012 | 2,078 | **0.97** | 13,580 | 9,972 |
| complex 1,080 (2³·3³·5) | 2,508 | 2,279 | 1.10 | 14,309 | 10,981 |
| complex 1,920 (2⁷·3·5) | 3,844 | 3,957 | **0.97** | 21,680 | 16,124 |
| complex 1,009 (prime, Rader) | 12,930 | 20,646 | **0.63** | 60,774 | 38,362 |
| complex 1,296 (2⁴·3⁴) | 2,760 | 2,968 | **0.93** | 16,149 | 12,521 |
| complex 10,007 (prime, Bluestein) | 170,723 | 210,119 | **0.81** | 635,910 | 465,736 |
| RFFT 256 | 241 | 224 | 1.075 | 5,963 | 5,252 |
| RFFT 1,024 | 1,016 | 986 | **1.030** | 9,459 | 8,236 |
| RFFT 4,096 | 5,596 | 5,272 | 1.061 | 24,916 | 22,241 |
| RFFT 65,536 | 135,192 | 153,248 | **0.88** | 399,361 | 489,046 |
| RFFT 1,048,576 | 2,874,362 | 3,645,428 | **0.79** | 8,488,329 | 9,984,001 |
| RFFT 1,000 (2³·5³) | 1,214 | 1,234 | **0.984** | 10,159 | 8,654 |
| RFFT 1,080 (2³·3³·5) | 1,380 | 1,362 | **1.014** | 10,768 | 8,921 |
| RFFT 1,920 (2⁷·3·5) | 2,452 | 2,320 | 1.057 | 14,208 | 11,972 |
| 2-D 64×64 | 9,711 | 10,931 | **0.89** | 46,781 | 32,792 |
| 2-D 128×128 | 50,370 | 62,974 | **0.80** | 144,272 | 127,209 |
| 2-D 256×256 | 267,577 | 286,266 | **0.94** | 693,329 | 453,742 |
| 2-D 512×512 | 1,232,522 | 1,312,183 | **0.94** | 2,982,556 | 1,894,462 |
| 2-D 1024×1024 | 6,005,178 | 6,388,344 | **0.94** | 14,059,597 | 9,368,322 |

## Neoverse-N1 (arm64)

cfarm424, 64 cores, shared (1-minute load 2.3–3.3 during the runs, from other
users' jobs; core 3, pinned); FFTW 3.3.10 built from source with NEON. Runs
`out1-def` and `out2-def` of 2026-11-01, the code of the v0.28.7 tag;
FFTW over the same two runs
([`arm64-neoverse-n1-20261101-v0.28.7/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/arm64-neoverse-n1-20261101-v0.28.7/)).

- **Against Round 40's column (v0.26.1):** three rows moved by more than 5%,
  all faster: 2-D 256² 0.54 → 0.47, 512² 0.48 → 0.45, 1024² 0.42 → 0.39.
  Every other row is within 3%. arm64's code has not changed since v0.24.0.
- **Faster than FFTW on all 24 rows.** None is within 5% and still slower. The
  closest are RFFT 256 (0.992) and RFFT 1,080 (0.987), 1.049 and 1.039 on
  v0.23.1 (Round 37's main runs): Round 37's NEON untangle loads with `VLD1`
  and `VZIP` instead of `VLD2` and reads a pre-split twiddle table.
- **Since Round 37's table (v0.24.0):** Bluestein's 10,007 went 0.72 → 0.69.
  v0.26.0 moved its three pointwise products to one function, on arm64 the
  same scalar loop behind a call, and an A/B measured v0.26.1 2% faster than
  v0.25.0 there. 2-D 256² went 0.50 → 0.54 with no change to its code on arm64
  since v0.24.0; one binary ran it at 610–705 µs between processes (likely
  allocation placement, not proved). Every other row is within 0.01 of Round
  37's.
- **FFTW's variance:** its own 10,007 moved 5.9% between the two runs (586 and
  620 µs).

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 1,004 | 1,148 | **0.88** | 9,487 | 7,600 |
| complex 1,024 | 4,863 | 6,319 | **0.77** | 18,801 | 14,520 |
| complex 4,096 | 23,706 | 40,850 | **0.58** | 59,480 | 47,690 |
| complex 65,536 | 715,579 | 1,220,558 | **0.59** | 2,141,397 | 1,441,820 |
| complex 1,048,576 | 19,907,818 | 43,262,535 | **0.46** | 43,917,151 | 30,965,815 |
| complex 1,000 (2³·5³) | 5,988 | 7,152 | **0.84** | 19,233 | 15,098 |
| complex 1,080 (2³·3³·5) | 6,812 | 7,665 | **0.89** | 21,581 | 16,618 |
| complex 1,920 (2⁷·3·5) | 11,832 | 13,495 | **0.88** | 32,899 | 25,909 |
| complex 1,009 (prime, Rader) | 21,450 | 48,014 | **0.45** | 96,145 | 56,961 |
| complex 1,296 (2⁴·3⁴) | 8,612 | 11,311 | **0.76** | 24,768 | 19,483 |
| complex 10,007 (prime, Bluestein) | 403,106 | 595,830 | **0.68** | 1,024,600 | 756,475 |
| RFFT 256 | 712 | 718 | **0.992** | 8,656 | 7,579 |
| RFFT 1,024 | 2,941 | 4,200 | **0.70** | 14,229 | 12,267 |
| RFFT 4,096 | 14,056 | 19,872 | **0.71** | 37,088 | 32,520 |
| RFFT 65,536 | 382,180 | 544,310 | **0.70** | 729,117 | 740,659 |
| RFFT 1,048,576 | 9,986,061 | 17,292,621 | **0.58** | 20,728,276 | 18,539,546 |
| RFFT 1,000 (2³·5³) | 3,660 | 3,940 | **0.93** | 15,389 | 12,689 |
| RFFT 1,080 (2³·3³·5) | 4,146 | 4,200 | **0.987** | 15,961 | 13,979 |
| RFFT 1,920 (2⁷·3·5) | 6,769 | 7,309 | **0.93** | 21,779 | 18,883 |
| 2-D 64×64 | 24,297 | 30,897 | **0.79** | 70,567 | 49,901 |
| 2-D 128×128 | 125,156 | 191,820 | **0.65** | 253,284 | 199,196 |
| 2-D 256×256 | 608,100 | 1,303,268 | **0.47** | 1,135,754 | 925,971 |
| 2-D 512×512 | 3,121,416 | 6,985,524 | **0.45** | 5,666,574 | 4,091,785 |
| 2-D 1024×1024 | 17,485,117 | 44,525,113 | **0.39** | 26,170,156 | 20,891,855 |

## AMD Zen 4 (Ryzen 9 7950X), AVX-512, with hardware counters

A bare-metal Ryzen 9 7950X (Zen 4, 16 cores), the first x86 host here with a
readable PMU (`perf stat`, AMD IBS); measured 2026-11-08, one pinned core (its
SMT sibling idle), complex transforms only, interleaved rounds, median of
seven. FFTW 3.3.10 built from source twice, with and without AVX-512 (its
1,024 plan is `t1fv_32_avx512` + `n2fv_32_avx2`, or `t1fv_32_avx2` +
`n2fv_32_avx2`). No numpy on the host, so this is not a full sweep. Raw files
and counters:
[`amd64-zen4-fractal-20261108/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/amd64-zen4-fractal-20261108/).

go-fft time ÷ FFTW time (below 1 is faster). v0.28.11 ran the Cascade Lake route
on this CPU; v0.28.12 runs Zen 3's up to 65,536 points:

| complex | v0.28.11 ÷ FFTW AVX2 build | v0.28.12 ÷ FFTW AVX2 build | v0.28.12 ÷ FFTW AVX-512 build |
|--:|--:|--:|--:|
| 256 | 1.43 | 1.13 | 1.29 |
| 1,024 | 1.31 | 1.14 | 1.31 |
| 4,096 | 1.00 | **0.93** | 1.06 |
| 65,536 | **0.88** | **0.87** | **0.98** |
| 1,048,576 | 1.07 | 1.07 | 1.07 |

Why: a Zen 4 core runs a 512-bit operation as two 256-bit halves. At complex
1,024 the AVX-512 passes retired 13,922 instructions in 5,961 cycles (IPC
2.34); the AVX2 ones with the split layout 18,437 in 5,166 (IPC 3.57), FFTW's
AVX2 plan 17,588 in 4,471 (IPC 3.93). The wide passes save instructions, not
pipe time, and stalled the floating-point scheduler 3.6 times as often (2,947
against 809 token stalls per transform); IBS put 77% of their samples in the
final radix-8 pass. The 4,039 64-byte-split loads per transform that looked
like misalignment are every 512-bit load, and aligning the twiddle tables
changed nothing. v0.28.12 / v0.28.11 on this CPU: complex 256 1.26, 1,024
1.15, 4,096 1.07, 16,384 1.13, 1,000 1.13, 1,296 1.12; above 65,536 points
the AVX-512 route stays, 3–12% faster there, and nothing changes. What is left
at 1,024 is FFTW's IPC (1.14× = 1.05× instructions × 1.10× IPC, with twice the
L1D fills from L2), as on Zen 3.

Round 51 (v0.28.14, 2026-11-08, load 0.3–0.9) asked why the AVX-512 route
stalls. IBS finds no single stalling instruction: the final pass is flat, a
third of it the 24 `VINSERTF32X4` that gather four blocks into a register, each
one uop on FP1/2 at one per cycle on Zen 4 (uops.info). Rewriting the gather
(masked loads 1,185 ns against 1,103, 256-bit halves 1,075, no gather at all
1,034) moves the transform by at most 6%, so it is not the cause; the AVX2
route's split layout is. The 256-bit-halves kernel lost 3–4.6% at 256 to 1,024
points on Cascade Lake, so it is not shipped. What is shipped: on a Zen 4,
65,536, 131,072 and 262,144 points take the 512-bit split layout, Cascade Lake's
route (bit-identical), 1.065, 1.034 and 1.026 times faster than before; every
other power of two from 256 to 2^20 ties or loses on it and keeps its route.
Against FFTW's AVX-512 build, the fastest on this CPU: 65,536 0.94 to 0.88,
131,072 0.92 to 0.90, 262,144 0.86 to 0.84; 256 (1.26) and 1,024 (1.30) do not
move. Raw files:
[`amd64-zen4-fractal-20261108-round51/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/amd64-zen4-fractal-20261108-round51/).

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, a VM shared with other users; measured 2026-11-07 at the
v0.28.11 tag (a clean clone), one-minute load 0.00 at the start and 1.0 from the
pinned core during the runs. Four pinned runs, default, FMA, FMA, default (the
default column uses runs 1 and 4; run 3 was redone because another user's job
took the load to 9.4); FFTW 3.3.10 built from source with AVX-512 codelets, the
median of the four runs. A control run of Round 42's binary straight after
reads 14 of 24 within 5% with the same complex 1,024 (1.12) and 65,536 (1.14),
so the moves below are the host's, not code. Raw files:
[`amd64-cascadelake-20261107-v0.28.11/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/amd64-cascadelake-20261107-v0.28.11/).

- **Within 5% of FFTW: 15 of 24, faster on 11** (14 in the v0.27.0 column).
  All five 2-D rows stay at or below 1.018 (64×64 1.018, the others
  0.65–0.96).
- **Within 5% but slower than FFTW:** RFFT 1,920 (1.008), 2-D 64×64 (1.018),
  RFFT 4,096 (1.032), RFFT 1,080 (1.045).
- **Slower by more:** complex 256 (1.065), 1,024 (1.24), 65,536 (1.17), 1,000
  (1.11), 1,080 (1.14), 1,920 (1.11), 1,296 (1.072); RFFT 65,536 (1.09) and
  RFFT 1,048,576 (1.125).
- **With `Options{FMA: true}` (runs 2 and 3): 19 of 24, faster on 15.** Complex
  1,024 reads 1.026, 256 0.986; out are complex 65,536 (1.19), 1,000 (1.105),
  1,080 (1.14), 1,920 (1.14) and RFFT 65,536 (1.07).
- **Noise:** go-fft's complex 65,536 read 444 and 504 µs in the two default
  runs, 2-D 512² 2.17 and 2.80 ms; FFTW's own 2-D 512² ran 2.64–4.14 ms and 1024²
  22.5–24.9 ms. Read these rows as ±10%.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 399 | 374 | 1.065 | 8,850 | 6,578 |
| complex 1,024 | 2,641 | 2,129 | 1.24 | 17,333 | 12,144 |
| complex 4,096 | 12,656 | 13,140 | **0.963** | 61,346 | 48,126 |
| complex 65,536 | 474,034 | 404,291 | 1.17 | 2,086,657 | 1,416,087 |
| complex 1,048,576 | 20,755,918 | 27,048,907 | **0.77** | 53,428,242 | 42,194,059 |
| complex 1,000 (2³·5³) | 3,078 | 2,777 | 1.11 | 17,198 | 14,258 |
| complex 1,080 (2³·3³·5) | 3,476 | 3,053 | 1.14 | 20,572 | 13,706 |
| complex 1,920 (2⁷·3·5) | 5,622 | 5,053 | 1.11 | 31,575 | 23,770 |
| complex 1,009 (prime, Rader) | 18,547 | 30,890 | **0.60** | 83,732 | 57,070 |
| complex 1,296 (2⁴·3⁴) | 4,048 | 3,774 | 1.072 | 22,313 | 16,961 |
| complex 10,007 (prime, Bluestein) | 343,164 | 371,724 | **0.92** | 902,132 | 735,910 |
| RFFT 256 | 353 | 370 | **0.954** | 6,770 | 7,275 |
| RFFT 1,024 | 1,236 | 1,536 | **0.80** | 12,555 | 11,131 |
| RFFT 4,096 | 8,121 | 7,869 | **1.032** | 33,400 | 30,245 |
| RFFT 65,536 | 214,101 | 195,981 | 1.09 | 596,802 | 698,256 |
| RFFT 1,048,576 | 11,333,654 | 10,077,543 | 1.125 | 28,772,182 | 21,774,541 |
| RFFT 1,000 (2³·5³) | 1,836 | 1,904 | **0.964** | 13,561 | 11,482 |
| RFFT 1,080 (2³·3³·5) | 2,128 | 2,035 | **1.045** | 13,597 | 12,017 |
| RFFT 1,920 (2⁷·3·5) | 3,514 | 3,485 | **1.008** | 18,289 | 16,686 |
| 2-D 64×64 | 16,134 | 15,846 | **1.018** | 57,074 | 47,336 |
| 2-D 128×128 | 77,899 | 81,351 | **0.958** | 212,400 | 184,477 |
| 2-D 256×256 | 457,326 | 485,604 | **0.942** | 928,720 | 772,354 |
| 2-D 512×512 | 2,483,704 | 3,063,681 | **0.81** | 6,214,008 | 4,371,614 |
| 2-D 1024×1024 | 15,363,670 | 23,482,140 | **0.65** | 29,929,652 | 27,037,158 |

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 on all 24 rows, composites and 2-D included (1,296: 0.76×,
    2-D 128²: 0.65×), the closest being RFFT 256 (0.992) and RFFT 1,080 (0.987);
  * Zen 3 on 16 of 24: complex from 4,096 points (0.50–0.92×), the composites
    1,000, 1,920 and 1,296 (0.93–0.97×), Bluestein's 10,007 (0.81×), RFFT
    1,000 (0.984×), RFFT from 65,536 (0.79–0.88×) and every 2-D row
    (0.80–0.94×);
  * Cascade Lake on 11 of 24: 2-D 128² to 1024² (0.65–0.96×), complex 4,096
    and 2^20 (0.963×, 0.77×), Bluestein's 10,007 (0.92×), and RFFT 256, 1,024
    and 1,000 (0.80–0.964×);
  * primes on every host: Rader's 1,009 in 0.45–0.61× FFTW's time.
* **Where FFTW leads:**
  * small powers of two on amd64: complex 256 at 1.07× on Zen 3 (1.065× on
    Cascade Lake), complex 1,024 at 1.13× and 1.24×; on Zen 3 RFFT 256
    (1.075×) and 4,096 (1.061×). For complex 256 and 1,024 and RFFT 4,096,
    FFTW's 32- and 64-point codelets need two passes. Round 38 built a
    bit-identical radix-32 pass for amd64; it lost to split radix 8 wherever it
    was timed and was not shipped (v0.26.4);
  * complex 1,080 on Zen 3 (1.10×) and RFFT 1,920 (1.057×);
  * on Cascade Lake, the composites 1,000–1,920 (1.07–1.14×), complex and
    RFFT 65,536 (1.17×, 1.09×) and RFFT 2^20 (1.125×).
* **Changed since the previous version of this page (v0.26.0):**
  * Neoverse-N1 (re-measured 2026-11-01): still 24 of 24 faster; 2-D 256²
    0.54 → 0.47, 512² 0.48 → 0.45, 1024² 0.42 → 0.39.
  * Zen 3 (re-measured at v0.28.6, 2026-10-31): 18 of 24 within 5% (faster on
    16, was 17); with `Options{FMA: true}` 18 to 20 of 24 over two passes; complex 65,536 0.87 → 0.92, RFFT 1,000 0.89 → 0.98, RFFT
    1,920 0.991 → 1.057, 2-D 512² 0.991 → 0.94. Complex 1,024 stays 1.13×,
    an IPC deficit of the twiddled radix-32 pass (Rounds 43–44, branch
    `perf-zen1024-fmapipes`, not merged).
  * Cascade Lake (re-measured 2026-11-07 at v0.28.11): 15 of 24 within 5%,
    faster on 11 (was 14); with `Options{FMA: true}` 19 of 24, faster on 15.
    Complex 1,024 1.19 → 1.24 (FMA 1.026), 65,536 1.08 → 1.17, RFFT 1,080
    0.995 → 1.045, 2-D 64² 0.94 → 1.018, 512² 0.88 → 0.81, 1024² 0.83 →
    0.65. A control run of Round 42's binary shows the same pattern.
  * Cascade Lake (v0.27.0): 2-D 64² 1.19 → 0.94, 128² 1.027 → 0.95, 256² 0.97
    → 0.95, 512² 1.20 → 0.88, 1024² 0.80 → 0.83. Of those, 1024²'s two
    numbers are against FFTW medians of different rounds; against v0.26.1 in
    the same runs it went 1.025 → 0.834. The previous page put 512² down to
    allocation placement; Round 39 found it was not.
  * Neoverse-N1 (re-measured at v0.26.1): Bluestein's 10,007 0.72 → 0.69,
    2-D 256² 0.50 → 0.54.
* **Against numpy and scipy:** go-fft is at or above both on every row, in every
  run, on all three hosts (0.04–0.68 of numpy's time, 0.04–0.86 of scipy's).
* **Against gonum:** go-fft is faster on every row measured (19/19 on each host).
* **Not in these tables:** Haswell (Intel AVX2-only) has no FFTW on its host. At
  v0.21.0 it runs complex 512–2^18 1.03–1.21× and the composites 1000–6000
  1.07–1.21× faster than v0.19.1 (Round 29); v0.24.0 runs 2-D 64×1024 … 1024²
  1.04–1.06× and complex 6,144 and 10,240 1.20× faster than v0.23.1 (Round 37).
  Round 37 also measured that v0.22.x cost Haswell's RFFT 256 about 1.5%;
  v0.25.0's tail jump removes the call responsible, but has not been
  re-measured on Haswell. No loong64 host could be reached.
* **Apple M4 Max (first measurement against FFTW, 2026-11-02, v0.28.9):**
  FFTW 3.3.11 from the pkgx bottle rebuilt with NEON (control: 207 `_neon`
  symbols among 751 `_fftw_` ones in the dylib), linked statically from C;
  the other hosts use 3.3.10 built from source. go-fft ran GOMAXPROCS=1 on an
  unpinned core (macOS has no `taskset`) on a shared machine, each pass gated
  on a one-minute load below 8; two passes and a repeat of the first.
  **16 of 24 rows within 5% of FFTW, faster on 12**. Behind it on complex
  65,536 (1.27×), RFFT 256 (1.18×), 65,536 (1.18×) and 2²⁰ (1.14×), and 2-D
  64² to 1024² except 512² (1.18–1.28×); ahead on the composites (0.86–0.94×),
  the primes (0.57, 0.85) and complex 2²⁰ (0.87×). At or above numpy.fft on
  24 of 24 rows and scipy.fft on 23 (2-D 256², 1.22). The repeat moves four
  large rows by 13–16%, so rows near the line are not settled. The arm64
  kernels were tuned on Neoverse-N1; these are open defects on the M4. Raw
  data: [`arm64-m4-20261102-v0.28.9/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/arm64-m4-20261102-v0.28.9/).
* **Apple M4 Max at v0.28.11 (2026-11-02):** the complex 65,536 gap was the
  576-byte gap between the destination and the scratch buffer, which costs
  Apple Silicon a fifth of the time from 65,536 points (a false store-to-load
  dependence). It is 4,032 bytes on darwin/arm64 since v0.28.11, and the row
  went from 1.27× to about **1.07–1.09×** of FFTW, still unsettled (the
  drift control moves such rows by 13–16%). Complex 2²⁰ is **0.80–0.86×**.
  The RFFT 256, 65,536 and 2²⁰ rows and the 2-D rows are unchanged. FFTW
  runs one leaf codelet per axis there (`n1fv_64`, `n1fv_128`,
  `hc2cfdftv_32`), while go-fft's NEON passes stop at radix 8, so it makes
  more passes and more loads and stores; a radix-16/32/64 NEON pass is the
  open lever. Load at measurement 5.4–7.5. Data:
  [`arm64-m4-gap-20261102/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/arm64-m4-gap-20261102/).
* **Single precision:** float32 takes 0.46–0.83× the float64 time on Zen 3 and
  0.58–0.73× on Neoverse-N1. FFTW's single-precision library is not built on
  the hosts, so there is no float32 comparison with it.
