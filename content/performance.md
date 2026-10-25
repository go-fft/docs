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

The numbers below describe **v0.27.0**. Each host's numbers are the newest
parity runs of the code its paths run in v0.27.0. v0.24.0 (Round 37) changed
only Haswell's and arm64's paths. v0.25.0 (Round 35), v0.26.3 (the N-D
correctness fix) and v0.27.0 (Round 39) changed only Intel AVX-512's. v0.26.0
(Round 36) changed only paths taken where the CPU is not Intel's. v0.26.1,
v0.26.2 and v0.26.4 changed no library code. Measured on three GCC Compile
Farm hosts with go1.27.1, every row on one pinned core, all on 2026-10-11:

* **Neoverse-N1**: Round 40, the v0.26.1 tag (arm64 has not changed since);
* **Cascade Lake**: Round 39, the branch that became v0.27.0;
* **Zen 3**: Round 36, the branch that became v0.26.0.

What these runs do not contain is said under each host: Round 39's parity
binary predates its rebase onto v0.26.3, whose fix touches no row here, and
Round 36's parity was taken on the v0.23.1 base.

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
| of which faster than FFTW | 17 | 24 | 14 |
| at or above numpy.fft, in both runs, of 24 | 24 | 24 | 24 |
| at or above scipy.fft, in both runs, of 24 | 24 | 24 | 24 |
| faster than gonum, in both runs, of 19 (1-D) | 19 | 19 | 19 |

The 24 rows are the complex 1-D, real 1-D and 2-D tables below (11 + 8 + 5); the
IRFFT rows are not counted, as in every round.

## AMD EPYC 7773X (Zen 3), AVX2

cfarm420, a 128-thread VM shared with other users (1-minute load 1.9–2.3 during
the parity runs); FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. Runs
`par2-br` and `par3-br` of Round 36; FFTW over `par1-main`, `par2-br`,
`par3-br`, `par4-main` (main is v0.23.1).

- **Within 5% but slower than FFTW:** RFFT 1,024 (1.047).
- **Slower by more:** complex 256 (1.10), 1,024 (1.13), 1,080 (1.08); RFFT 256
  (1.051), 4,096 (1.09), 1,080 (1.06).
- **Not in these runs:** they were taken on the v0.23.1 base, before Round 36
  was rebased onto v0.25.0 (its A/Bs were redone on v0.25.0). The rebase brings
  Zen 3 one change of its own: v0.25.0's tail jump makes the real-FFT untangle
  one call instead of two on every CPU without AVX-512, a call Round 37
  measured at about 1.5% of RFFT 256 on Haswell. Round 38 re-took Zen 3's
  parity at v0.26.1, with the tail jump: its RFFT rows moved by −4.2% to +7.4%
  in ratio and the complex rows the jump does not touch by −2.3% to +7.2%, so
  the jump's effect is inside that measurement's spread (20 of 24 within 5%,
  15 faster, under loads up to 3.2). That is a cross-check only; the table
  stays Round 36's.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 307 | 279 | 1.10 | 6,733 | 5,417 |
| complex 1,024 | 1,760 | 1,553 | 1.13 | 12,897 | 9,729 |
| complex 4,096 | 9,582 | 10,707 | **0.89** | 44,570 | 33,320 |
| complex 65,536 | 257,943 | 297,492 | **0.87** | 1,889,202 | 926,177 |
| complex 1,048,576 | 6,008,336 | 11,885,313 | **0.51** | 39,631,045 | 13,845,480 |
| complex 1,000 (2³·5³) | 2,062 | 2,091 | **0.986** | 13,601 | 10,033 |
| complex 1,080 (2³·3³·5) | 2,477 | 2,288 | 1.08 | 14,399 | 10,891 |
| complex 1,920 (2⁷·3·5) | 3,918 | 4,003 | **0.98** | 21,654 | 16,335 |
| complex 1,009 (prime, Rader) | 12,551 | 20,431 | **0.61** | 61,005 | 38,647 |
| complex 1,296 (2⁴·3⁴) | 2,790 | 3,051 | **0.91** | 16,193 | 12,495 |
| complex 10,007 (prime, Bluestein) | 167,374 | 203,877 | **0.82** | 644,482 | 474,608 |
| RFFT 256 | 237 | 226 | 1.051 | 6,082 | 5,369 |
| RFFT 1,024 | 1,009 | 964 | **1.047** | 9,484 | 8,317 |
| RFFT 4,096 | 5,722 | 5,271 | 1.09 | 24,946 | 22,350 |
| RFFT 65,536 | 138,438 | 155,626 | **0.89** | 416,831 | 500,548 |
| RFFT 1,048,576 | 2,954,656 | 3,768,595 | **0.78** | 8,387,307 | 10,298,716 |
| RFFT 1,000 (2³·5³) | 1,159 | 1,300 | **0.89** | 10,201 | 8,669 |
| RFFT 1,080 (2³·3³·5) | 1,412 | 1,331 | 1.06 | 10,899 | 9,123 |
| RFFT 1,920 (2⁷·3·5) | 2,423 | 2,445 | **0.991** | 14,332 | 12,130 |
| 2-D 64×64 | 9,799 | 10,958 | **0.89** | 48,025 | 33,946 |
| 2-D 128×128 | 51,955 | 62,622 | **0.83** | 147,828 | 125,980 |
| 2-D 256×256 | 265,382 | 295,281 | **0.90** | 711,655 | 456,853 |
| 2-D 512×512 | 1,306,942 | 1,318,229 | **0.991** | 2,902,803 | 1,870,872 |
| 2-D 1024×1024 | 6,370,542 | 6,647,458 | **0.96** | 13,657,415 | 9,738,945 |

## Neoverse-N1 (arm64)

cfarm424, 64 cores, used by the round alone (load 0.04–1.09); FFTW 3.3.10 built
from source with NEON. Runs `r1` and `r2` of Round 40, both the v0.26.1 tag;
FFTW over the same two runs.

- **Faster than FFTW on all 24 rows.** None is within 5% and still slower. The
  closest are RFFT 256 (0.998) and RFFT 1,080 (0.994), 1.049 and 1.039 on
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
| complex 256 | 1,012 | 1,149 | **0.88** | 9,522 | 7,642 |
| complex 1,024 | 4,899 | 6,327 | **0.77** | 18,809 | 14,538 |
| complex 4,096 | 24,007 | 40,950 | **0.59** | 60,107 | 48,093 |
| complex 65,536 | 726,467 | 1,252,182 | **0.58** | 2,243,676 | 1,509,643 |
| complex 1,048,576 | 21,166,422 | 45,054,665 | **0.47** | 46,540,259 | 31,873,243 |
| complex 1,000 (2³·5³) | 6,076 | 7,168 | **0.85** | 19,204 | 15,192 |
| complex 1,080 (2³·3³·5) | 6,884 | 7,667 | **0.90** | 21,733 | 16,765 |
| complex 1,920 (2⁷·3·5) | 11,994 | 13,540 | **0.89** | 33,027 | 26,127 |
| complex 1,009 (prime, Rader) | 21,636 | 48,140 | **0.45** | 96,663 | 57,253 |
| complex 1,296 (2⁴·3⁴) | 8,752 | 11,370 | **0.77** | 25,008 | 19,610 |
| complex 10,007 (prime, Bluestein) | 415,486 | 603,267 | **0.69** | 1,023,385 | 760,683 |
| RFFT 256 | 719 | 721 | **0.998** | 8,640 | 7,854 |
| RFFT 1,024 | 2,940 | 4,204 | **0.70** | 14,332 | 12,564 |
| RFFT 4,096 | 14,314 | 19,864 | **0.72** | 37,314 | 33,231 |
| RFFT 65,536 | 392,310 | 544,361 | **0.72** | 737,580 | 760,281 |
| RFFT 1,048,576 | 10,468,752 | 18,834,911 | **0.56** | 19,618,025 | 18,495,787 |
| RFFT 1,000 (2³·5³) | 3,740 | 3,967 | **0.94** | 15,170 | 12,591 |
| RFFT 1,080 (2³·3³·5) | 4,183 | 4,207 | **0.994** | 15,894 | 13,925 |
| RFFT 1,920 (2⁷·3·5) | 6,800 | 7,337 | **0.93** | 21,741 | 18,966 |
| 2-D 64×64 | 24,535 | 31,011 | **0.79** | 70,712 | 50,410 |
| 2-D 128×128 | 128,074 | 194,975 | **0.66** | 259,414 | 200,828 |
| 2-D 256×256 | 684,228 | 1,272,719 | **0.54** | 1,092,138 | 894,998 |
| 2-D 512×512 | 3,295,962 | 6,854,409 | **0.48** | 6,507,507 | 4,320,211 |
| 2-D 1024×1024 | 18,348,440 | 44,114,803 | **0.42** | 27,336,933 | 22,074,923 |

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
  * Neoverse-N1 on all 24 rows, composites and 2-D included (1,296: 0.77×,
    2-D 128²: 0.66×), the closest being RFFT 256 (0.998) and RFFT 1,080 (0.994);
  * Zen 3 on 17 of 24: complex from 4,096 points (0.51–0.89×), the composites
    1,000, 1,920 and 1,296 (0.91–0.99×), Bluestein's 10,007 (0.82×), RFFT
    1,000 and 1,920 (0.89×, 0.991×), RFFT from 65,536 (0.78–0.89×) and every
    2-D row (0.83–0.99×);
  * Cascade Lake on 14 of 24: every 2-D row (0.83–0.95×), complex 4,096 and
    2^20 (0.94×, 0.70×), Bluestein's 10,007 (0.94×), and RFFT 256, 1,024,
    1,000, 1,080 and 1,920 (0.78–0.995×);
  * primes on every host: Rader's 1,009 in 0.45–0.61× FFTW's time.
* **Where FFTW leads:**
  * small powers of two on amd64: complex 256 at 1.10× on Zen 3 (1.052× on
    Cascade Lake), complex 1,024 at 1.13× and 1.19×; on Zen 3 RFFT 256
    (1.051×) and 4,096 (1.09×). For complex 256 and 1,024 and RFFT 4,096,
    FFTW's 32- and 64-point codelets need two passes. Round 38 built a
    bit-identical radix-32 pass for amd64; it lost to split radix 8 wherever it
    was timed and was not shipped (v0.26.4);
  * complex and RFFT 1,080 on Zen 3 (1.08×, 1.06×);
  * on Cascade Lake, the composites 1,000–1,920 (1.10–1.13×) and complex and
    RFFT 65,536 (1.08×, 1.09×).
* **Changed since the previous version of this page (v0.26.0):**
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
