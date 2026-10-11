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

The numbers below describe **v0.26.0**. Each host's numbers are the parity runs
of the branch that became the release changing that host's paths: v0.24.0
(Round 37) changed only Haswell's and arm64's paths, v0.25.0 (Round 35) only
Intel AVX-512's, v0.26.0 (Round 36) only paths taken where the CPU is not
Intel's. Measured on three GCC Compile Farm hosts with go1.27.1, every row on
one pinned core, all on 2026-10-11:

* **Neoverse-N1**: Round 37, the branch that became v0.24.0;
* **Cascade Lake**: Round 35, the branch that became v0.25.0;
* **Zen 3**: Round 36, the branch that became v0.26.0.

What these runs do not contain is said under each host: Round 35's parity binary
predates its own last two changes, Round 36's parity was taken on the v0.23.1
base, and two later changes reach other vendors' paths without having been
re-measured against FFTW there.

FFTW's own times move by up to 48% between runs on one host, so no single run is
quoted. go-fft's time is the median of the three repetitions in each run, then
the mean of the round's two branch runs; FFTW's is the median over the round's
four runs (each round ran main, branch, branch, main); numpy and scipy are the
mean of the same two branch runs. The tables are computed from the raw files by
[`current-numbers-v0.26.0/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/current-numbers-v0.26.0/), which also lists
each run's own value and the command that produced each table. The raw runs are in
[`benchmarks/results/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/), and the dated optimization
rounds (what was tried, kept or dropped, and why) are in
[BENCHMARKS.md](https://github.com/go-fft/fft/blob/main/BENCHMARKS.md). Reproduce them with `benchmarks/run.sh`
(`benchmarks/remote/` for a Linux host without Go).

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
| of which faster than FFTW | 17 | 24 | 10 |
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
  measured at about 1.5% of RFFT 256 on Haswell. Zen 3's RFFT rows have not
  been re-measured against FFTW with it.

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

cfarm424, 64 cores, used by the round alone (load 0.1–1.1); FFTW 3.3.10 built
from source with NEON. Runs `br1` and `br2` of Round 37; FFTW over `main1`,
`br1`, `br2`, `main2` (main is v0.23.1).

- **Faster than FFTW on all 24 rows.** The closest are RFFT 256 (0.991) and RFFT
  1,080 (0.995), 1.049 and 1.039 on v0.23.1 in the same round's main runs:
  Round 37's NEON untangle loads with `VLD1` and `VZIP` instead of `VLD2`
  and reads a pre-split twiddle table.
- **Not in these runs:** v0.26.0 moved Bluestein's three pointwise products to
  one function on every architecture; on arm64 that is the same scalar loop
  behind a call, and complex 10,007 has not been re-measured on N1 since.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 1,018 | 1,152 | **0.88** | 9,515 | 7,697 |
| complex 1,024 | 4,889 | 6,321 | **0.77** | 18,710 | 14,497 |
| complex 4,096 | 23,886 | 41,103 | **0.58** | 59,574 | 47,470 |
| complex 65,536 | 719,719 | 1,232,510 | **0.58** | 2,201,182 | 1,422,954 |
| complex 1,048,576 | 20,876,114 | 44,027,499 | **0.47** | 43,668,175 | 32,037,592 |
| complex 1,000 (2³·5³) | 6,067 | 7,198 | **0.84** | 19,571 | 15,324 |
| complex 1,080 (2³·3³·5) | 6,914 | 7,741 | **0.89** | 21,736 | 16,724 |
| complex 1,920 (2⁷·3·5) | 12,014 | 13,616 | **0.88** | 32,798 | 25,913 |
| complex 1,009 (prime, Rader) | 21,724 | 48,543 | **0.45** | 97,049 | 57,817 |
| complex 1,296 (2⁴·3⁴) | 8,746 | 11,337 | **0.77** | 24,904 | 19,688 |
| complex 10,007 (prime, Bluestein) | 428,134 | 596,926 | **0.72** | 1,022,992 | 768,560 |
| RFFT 256 | 715 | 721 | **0.991** | 8,686 | 7,802 |
| RFFT 1,024 | 2,931 | 4,225 | **0.69** | 14,304 | 12,482 |
| RFFT 4,096 | 14,186 | 19,900 | **0.71** | 37,135 | 32,972 |
| RFFT 65,536 | 391,220 | 545,445 | **0.72** | 722,176 | 752,438 |
| RFFT 1,048,576 | 10,384,114 | 18,430,631 | **0.56** | 19,631,689 | 18,297,101 |
| RFFT 1,000 (2³·5³) | 3,710 | 3,961 | **0.94** | 15,197 | 12,809 |
| RFFT 1,080 (2³·3³·5) | 4,192 | 4,214 | **0.995** | 15,983 | 14,139 |
| RFFT 1,920 (2⁷·3·5) | 6,852 | 7,336 | **0.93** | 22,071 | 19,015 |
| 2-D 64×64 | 24,836 | 31,078 | **0.80** | 70,816 | 50,369 |
| 2-D 128×128 | 128,667 | 193,502 | **0.66** | 260,395 | 199,393 |
| 2-D 256×256 | 626,124 | 1,255,943 | **0.50** | 1,087,451 | 909,638 |
| 2-D 512×512 | 3,219,429 | 6,828,754 | **0.47** | 5,663,813 | 4,105,104 |
| 2-D 1024×1024 | 18,088,587 | 43,799,490 | **0.41** | 26,331,516 | 20,539,367 |

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, used by the round alone (load 0.1–1.5); FFTW 3.3.10 built
from source with AVX-512 codelets. Runs `p-br1` and `p-br2` of Round 35;
FFTW over `p-main1`, `p-br1`, `p-br2`, `p-main2` (main is v0.23.1).

- **Within 5% but slower than FFTW:** complex 256 (1.042), RFFT 4,096 (1.039),
  RFFT 1,048,576 (1.040), RFFT 1,080 (1.020), 2-D 128² (1.027).
- **Slower by more:** complex 1,024 (1.20), 65,536 (1.07), 1,000 (1.09), 1,080
  (1.13), 1,920 (1.12), 1,296 (1.07); RFFT 65,536 (1.11); 2-D 64² (1.19),
  512² (1.20).
- **2-D 512² is allocation placement, not code:** the same binary read 2.17 ms
  in one run and 3.62 ms in the other; on eight separately allocated buffer
  pairs it runs 2.19–2.60 ms against FFTW's 2.29–2.59 (Round 35).
- **FFTW's variance:** its own 2-D 1024² ran 16.3–24.1 ms over the four runs,
  so go-fft's 0.80 there is against the median (1.00 against FFTW's fastest
  run); RFFT 1,048,576 is noisy on both sides (go-fft 7.6 and 9.3 ms, FFTW
  6.9–9.6).
- **Not in these runs:** the parity binary predates Round 35's tail jump (the
  untangle as one call, neutral in process on this host) and its rebase onto
  v0.24.0, whose changes are Haswell's and arm64's.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 389 | 374 | **1.042** | 8,943 | 6,401 |
| complex 1,024 | 2,528 | 2,104 | 1.20 | 16,522 | 11,896 |
| complex 4,096 | 12,322 | 13,131 | **0.94** | 59,214 | 44,751 |
| complex 65,536 | 436,876 | 408,166 | 1.07 | 1,948,516 | 1,367,829 |
| complex 1,048,576 | 18,136,814 | 25,944,781 | **0.70** | 51,045,545 | 40,757,897 |
| complex 1,000 (2³·5³) | 3,014 | 2,762 | 1.09 | 18,201 | 14,285 |
| complex 1,080 (2³·3³·5) | 3,412 | 3,029 | 1.13 | 20,684 | 13,349 |
| complex 1,920 (2⁷·3·5) | 5,499 | 4,911 | 1.12 | 31,541 | 22,914 |
| complex 1,009 (prime, Rader) | 17,962 | 30,387 | **0.59** | 80,353 | 56,663 |
| complex 1,296 (2⁴·3⁴) | 3,938 | 3,674 | 1.07 | 20,855 | 15,617 |
| complex 10,007 (prime, Bluestein) | 333,805 | 364,257 | **0.92** | 924,500 | 726,991 |
| RFFT 256 | 350 | 361 | **0.97** | 6,616 | 6,587 |
| RFFT 1,024 | 1,210 | 1,517 | **0.80** | 13,132 | 10,285 |
| RFFT 4,096 | 8,000 | 7,696 | **1.039** | 32,711 | 31,802 |
| RFFT 65,536 | 209,828 | 188,300 | 1.11 | 617,454 | 700,037 |
| RFFT 1,048,576 | 8,465,202 | 8,142,078 | **1.040** | 18,113,770 | 21,184,363 |
| RFFT 1,000 (2³·5³) | 1,778 | 1,884 | **0.94** | 14,237 | 12,133 |
| RFFT 1,080 (2³·3³·5) | 2,069 | 2,028 | **1.020** | 13,425 | 12,057 |
| RFFT 1,920 (2⁷·3·5) | 3,412 | 3,474 | **0.98** | 18,103 | 17,288 |
| 2-D 64×64 | 17,893 | 15,084 | 1.19 | 55,232 | 45,511 |
| 2-D 128×128 | 80,268 | 78,149 | **1.027** | 210,656 | 176,963 |
| 2-D 256×256 | 470,670 | 483,294 | **0.97** | 916,699 | 759,127 |
| 2-D 512×512 | 2,897,349 | 2,421,769 | 1.20 | 5,428,501 | 3,634,033 |
| 2-D 1024×1024 | 16,277,564 | 20,427,042 | **0.80** | 30,683,333 | 26,219,812 |

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 on all 24 rows, composites and 2-D included (1,296: 0.77×,
    2-D 128²: 0.66×), the closest being RFFT 256 (0.991) and RFFT 1,080 (0.995);
  * Zen 3 on 17 of 24: complex from 4,096 points (0.51–0.89×), the composites
    1,000, 1,920 and 1,296 (0.91–0.99×), Bluestein's 10,007 (0.82×), RFFT
    1,000 and 1,920 (0.89×, 0.991×), RFFT from 65,536 (0.78–0.89×) and every
    2-D row (0.83–0.99×);
  * primes on every host: Rader's 1,009 in 0.45–0.61× FFTW's time.
* **Where FFTW leads:**
  * small powers of two on amd64: complex 256 at 1.10× on Zen 3 (1.04× on
    Cascade Lake), complex 1,024 at 1.13× and 1.20×; on Zen 3 RFFT 256
    (1.051×) and 4,096 (1.09×). For complex 256 and 1,024 and RFFT 4,096,
    FFTW's 32- and 64-point codelets need two passes, and go-fft has no
    radix-32 pass (Round 36);
  * complex and RFFT 1,080 on Zen 3 (1.08×, 1.06×);
  * on Cascade Lake, the composites 1,000–1,920 (1.07–1.13×), complex and RFFT
    65,536 (1.07×, 1.11×) and 2-D 64² (1.19×).
* **Changed since the previous version of this page (v0.23.0):**
  * Neoverse-N1: RFFT 256 1.046 → 0.991 and RFFT 1,080 1.04 → 0.995 (v0.24.0).
  * Cascade Lake: complex 256 1.27 → 1.04, complex 1,296 1.20 → 1.07, RFFT 256
    1.08 → 0.97 (v0.25.0).
  * Zen 3: Bluestein's 10,007 1.07 → 0.82, 2-D 64² 1.056 → 0.89, complex 1,000
    1.11 → 0.986 (v0.26.0).
* **Against numpy and scipy:** go-fft is at or above both on every row, in every
  run, on all three hosts (0.04–0.69 of numpy's time, 0.04–0.93 of scipy's).
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
