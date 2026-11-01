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

The numbers below describe **v0.28.6** on Zen 3 and **v0.27.0** on the other
two hosts. Each host's numbers are the newest parity runs available. v0.24.0 (Round 37) changed
only Haswell's and arm64's paths. v0.25.0 (Round 35), v0.26.3 (the N-D
correctness fix) and v0.27.0 (Round 39) changed only Intel AVX-512's. v0.26.0
(Round 36) changed only paths taken where the CPU is not Intel's. v0.26.1,
v0.26.2 and v0.26.4 changed no library code. Measured on three GCC Compile
Farm hosts with go1.27.1, every row on one pinned core, all on 2026-10-11:

* **Neoverse-N1**: re-measured on 2026-11-01 at the code of v0.28.7 (two pinned
  runs; Round 40's column, the v0.26.1 tag, is the previous one);
* **Cascade Lake**: Round 39, the branch that became v0.27.0;
* **Zen 3**: re-measured on 2026-10-31 at the v0.28.6 tag (four runs: default,
  FMA, FMA, default; the default column uses the two default runs;
  [`amd64-zen3-epyc7773x-20261031-v0.28.6/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/amd64-zen3-epyc7773x-20261031-v0.28.6/)).

Cascade Lake was not re-measured: its host was under a one-minute load of 5–11
on 8 cores, above the load its earlier runs were taken at, so its column is the
2026-10-11 one. Apple M4 was not measured against FFTW either: the pkgx FFTW
bottle has no NEON codelets (see BENCHMARKS.md). What those runs do not
contain is said under each host: Round 39's parity binary predates its rebase
onto v0.26.3, whose fix touches no row here.

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
| of which faster than FFTW | 16 | 24 | 14 |
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

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, used by the round alone (load 0.26–1.02 during the parity
runs); FFTW 3.3.10 built from source with AVX-512 codelets. Runs `p-br1` and
`p-br2` of Round 39; FFTW over `p-main1`, `p-br1`, `p-br2`, `p-main2` (main is
v0.26.1).

- **All five 2-D rows are faster than FFTW** (0.83–0.95), and 14 of the 24 rows
  (10 in v0.26.0). Round 39 widened the column strips of large N-D axes and
  put the 2-D rows on the stack-frame kernels on Intel too.
- **Within 5% but slower than FFTW:** RFFT 4,096 (1.049).
- **Slower by more:** complex 256 (1.052), 1,024 (1.19), 65,536 (1.08), 1,000
  (1.10), 1,080 (1.13), 1,920 (1.11), 1,296 (1.10); RFFT 65,536 (1.09) and
  RFFT 1,048,576 (1.09).
- **RFFT 1,048,576 is noise:** one run read 7.97 ms and the other 10.49, and
  FFTW's four runs read 6.6–9.3 ms. Round 39 does not touch that path (its A/B
  reads 1.021, 0.980–1.093).
- **FFTW's variance:** its own 2-D 1024² ran 15.9–23.3 ms over the four runs,
  so go-fft's 0.83 there is against the median (0.90 against FFTW's fastest
  run).
- **Not in these runs:** Round 39 was rebased onto v0.26.3 after these runs.
  v0.26.3's fix changes only N-D axes of length 9·p, which no row here has.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 389 | 370 | 1.052 | 8,804 | 6,505 |
| complex 1,024 | 2,537 | 2,134 | 1.19 | 16,355 | 12,153 |
| complex 4,096 | 12,370 | 13,122 | **0.94** | 60,933 | 44,471 |
| complex 65,536 | 436,546 | 405,732 | 1.08 | 1,887,061 | 1,291,823 |
| complex 1,048,576 | 18,192,579 | 25,810,678 | **0.70** | 47,883,344 | 41,127,411 |
| complex 1,000 (2³·5³) | 3,023 | 2,750 | 1.10 | 16,996 | 13,965 |
| complex 1,080 (2³·3³·5) | 3,422 | 3,030 | 1.13 | 20,461 | 13,235 |
| complex 1,920 (2⁷·3·5) | 5,484 | 4,932 | 1.11 | 31,505 | 22,961 |
| complex 1,009 (prime, Rader) | 18,037 | 30,362 | **0.59** | 80,496 | 56,581 |
| complex 1,296 (2⁴·3⁴) | 3,965 | 3,595 | 1.10 | 20,789 | 16,529 |
| complex 10,007 (prime, Bluestein) | 339,064 | 360,983 | **0.94** | 896,852 | 724,882 |
| RFFT 256 | 343 | 365 | **0.94** | 6,581 | 7,075 |
| RFFT 1,024 | 1,190 | 1,515 | **0.78** | 12,984 | 10,385 |
| RFFT 4,096 | 7,876 | 7,506 | **1.049** | 32,895 | 31,569 |
| RFFT 65,536 | 205,210 | 187,492 | 1.09 | 594,791 | 689,933 |
| RFFT 1,048,576 | 9,226,644 | 8,452,834 | 1.09 | 19,769,657 | 20,641,798 |
| RFFT 1,000 (2³·5³) | 1,778 | 1,865 | **0.95** | 13,575 | 11,927 |
| RFFT 1,080 (2³·3³·5) | 2,061 | 2,072 | **0.995** | 13,554 | 11,673 |
| RFFT 1,920 (2⁷·3·5) | 3,397 | 3,489 | **0.97** | 18,157 | 16,632 |
| 2-D 64×64 | 15,949 | 17,027 | **0.94** | 57,622 | 45,179 |
| 2-D 128×128 | 76,616 | 81,010 | **0.95** | 204,093 | 182,495 |
| 2-D 256×256 | 447,658 | 471,570 | **0.95** | 889,079 | 758,109 |
| 2-D 512×512 | 2,100,370 | 2,379,883 | **0.88** | 5,071,545 | 3,569,429 |
| 2-D 1024×1024 | 14,216,236 | 17,053,614 | **0.83** | 26,710,309 | 24,662,554 |

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 on all 24 rows, composites and 2-D included (1,296: 0.76×,
    2-D 128²: 0.65×), the closest being RFFT 256 (0.992) and RFFT 1,080 (0.987);
  * Zen 3 on 16 of 24: complex from 4,096 points (0.50–0.92×), the composites
    1,000, 1,920 and 1,296 (0.93–0.97×), Bluestein's 10,007 (0.81×), RFFT
    1,000 (0.984×), RFFT from 65,536 (0.79–0.88×) and every 2-D row
    (0.80–0.94×);
  * Cascade Lake on 14 of 24: every 2-D row (0.83–0.95×), complex 4,096 and
    2^20 (0.94×, 0.70×), Bluestein's 10,007 (0.94×), and RFFT 256, 1,024,
    1,000, 1,080 and 1,920 (0.78–0.995×);
  * primes on every host: Rader's 1,009 in 0.45–0.61× FFTW's time.
* **Where FFTW leads:**
  * small powers of two on amd64: complex 256 at 1.07× on Zen 3 (1.052× on
    Cascade Lake), complex 1,024 at 1.13× and 1.19×; on Zen 3 RFFT 256
    (1.075×) and 4,096 (1.061×). For complex 256 and 1,024 and RFFT 4,096,
    FFTW's 32- and 64-point codelets need two passes. Round 38 built a
    bit-identical radix-32 pass for amd64; it lost to split radix 8 wherever it
    was timed and was not shipped (v0.26.4);
  * complex 1,080 on Zen 3 (1.10×) and RFFT 1,920 (1.057×);
  * on Cascade Lake, the composites 1,000–1,920 (1.10–1.13×) and complex and
    RFFT 65,536 (1.08×, 1.09×).
* **Changed since the previous version of this page (v0.26.0):**
  * Neoverse-N1 (re-measured 2026-11-01): still 24 of 24 faster; 2-D 256²
    0.54 → 0.47, 512² 0.48 → 0.45, 1024² 0.42 → 0.39.
  * Zen 3 (re-measured at v0.28.6, 2026-10-31): 18 of 24 within 5% (faster on
    16, was 17); with `Options{FMA: true}` 18 to 20 of 24 over two passes; complex 65,536 0.87 → 0.92, RFFT 1,000 0.89 → 0.98, RFFT
    1,920 0.991 → 1.057, 2-D 512² 0.991 → 0.94. Complex 1,024 stays 1.13×,
    an IPC deficit of the twiddled radix-32 pass (Rounds 43–44, branch
    `perf-zen1024-fmapipes`, not merged).
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
  re-measured on Haswell. Apple M4 has not been measured with the current
  engines, and no loong64 host could be reached.
* **Single precision:** float32 takes 0.46–0.83× the float64 time on Zen 3 and
  0.58–0.73× on Neoverse-N1. FFTW's single-precision library is not built on
  the hosts, so there is no float32 comparison with it.
