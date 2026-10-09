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

The numbers below describe **v0.23.0**. Each host's newest parity round runs
v0.23.0's code on that host: v0.22.x changed only Intel paths, v0.23.0 only AMD
paths, and nothing has changed arm64 since v0.21.0. Measured on three GCC Compile
Farm hosts with go1.27.1, every row on one pinned core:

* **Zen 3**: Round 32, 2026-10-09, the branch that became v0.23.0;
* **Neoverse-N1**: Round 33, 2026-10-09, v0.21.0;
* **Cascade Lake**: Round 31, 2026-10-09, the branch that became v0.22.0.

FFTW's own times move by up to 22% between runs on one host, so no single run is
quoted. go-fft's time is the median of the three repetitions in each run, then
the mean of two runs; FFTW's is the median over every run of the round (four on
Zen 3 and Cascade Lake, two on Neoverse-N1); numpy and scipy are the mean of the
same two runs. The tables are computed from the raw files by
[`current-numbers-v0.23.0/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/current-numbers-v0.23.0/), which also lists
each run's own value. The raw runs are in
[`benchmarks/results/`](https://github.com/go-fft/fft/tree/main/benchmarks/results/), and the dated optimization
rounds (what was tried, kept or dropped, and why) are in
[BENCHMARKS.md](https://github.com/go-fft/fft/blob/main/BENCHMARKS.md). Reproduce them with `benchmarks/run.sh`
(`benchmarks/remote/` for a Linux host without Go).

## Method

* **Single-threaded**, except the 2-D rows, where go-fft uses its multicore path
  (all hardware threads); FFTW, numpy and scipy stay on one thread.
* **Steady state**: every library reuses its plan, and go-fft writes into a
  reused slice (`Plan`, `RealPlan`, and `PlanN` for 2-D). The per-run reports in
  the raw data also give the allocating `FFT2` call and GFLOP/s.
* ns/op, lower is better. The ratio is go-fft ÷ FFTW (≤ 1.05 = parity, which
  includes rows slower than FFTW by at most 5%).

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
at most 5%. "Faster" below means strictly below 1.

| | Zen 3 | Neoverse-N1 | Cascade Lake |
|:--|--:|--:|--:|
| at or above FFTW (≤ 1.05), of 24 | 12 | 24 | 11 |
| of which faster than FFTW | 9 | 22 | 10 |
| at or above numpy.fft, in both runs, of 24 | 24 | 24 | 24 |
| at or above scipy.fft, in both runs, of 24 | 24 | 24 | 24 |
| faster than gonum, in both runs, of 19 (1-D) | 19 | 19 | 19 |

The 24 rows are the complex 1-D, real 1-D and 2-D tables below (11 + 8 + 5); the
IRFFT rows are not counted, as in every round.

## AMD EPYC 7773X (Zen 3), AVX2

cfarm420, a 128-thread VM shared with other users (1-minute load 3.0–4.1 during the
parity runs); FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. Runs
`par2-br` and `par3-br` of Round 32; FFTW over `par1-main` … `par4-main`.

- **Within 5% but slower than FFTW:** complex 10,007 (1.02), RFFT 1,000 (1.02),
  2-D 256² (1.01).
- **Slower by more:** complex 256 (1.09), 1,024 (1.13), 1,000 (1.06), 1,080
  (1.09), 1,296 (1.054); RFFT 256 (1.07), 1,024 (1.08), 4,096 (1.12), 1,080
  (1.14), 1,920 (1.08); 2-D 64² (1.13); and complex 4,096 (¹).

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 306 | 280 | 1.09 | 7,026 | 5,600 |
| complex 1,024 | 1,812 | 1,598 | 1.13 | 13,448 | 9,976 |
| complex 4,096 | 15,516 ¹ | 10,320 | 1.50 ¹ | 44,068 | 34,161 |
| complex 65,536 | 271,336 | 297,433 | **0.91** | 1,916,995 | 941,529 |
| complex 1,048,576 | 6,322,353 | 11,403,185 | **0.55** | 32,030,182 | 14,687,209 |
| complex 1,000 (2³·5³) | 2,200 | 2,074 | 1.06 | 13,738 | 10,557 |
| complex 1,080 (2³·3³·5) | 2,606 | 2,383 | 1.09 | 14,583 | 10,823 |
| complex 1,920 (2⁷·3·5) | 3,862 | 3,940 | **0.98** | 23,099 | 16,470 |
| complex 1,009 (prime, Rader) | 13,331 | 20,254 | **0.66** | 61,489 | 39,326 |
| complex 1,296 (2⁴·3⁴) | 3,118 | 2,959 | 1.054 | 16,599 | 12,692 |
| complex 10,007 (prime, Bluestein) | 217,170 | 213,383 | **1.02** | 660,133 | 468,610 |
| RFFT 256 | 243 | 228 | 1.07 | 6,131 | 5,333 |
| RFFT 1,024 | 1,060 | 985 | 1.08 | 9,582 | 8,420 |
| RFFT 4,096 | 5,962 | 5,304 | 1.12 | 25,372 | 22,149 |
| RFFT 65,536 | 145,598 | 154,166 | **0.94** | 424,736 | 496,290 |
| RFFT 1,048,576 | 2,934,751 | 3,781,403 | **0.78** | 8,400,873 | 10,170,434 |
| RFFT 1,000 (2³·5³) | 1,280 | 1,260 | **1.02** | 10,675 | 8,911 |
| RFFT 1,080 (2³·3³·5) | 1,528 | 1,341 | 1.14 | 11,186 | 9,106 |
| RFFT 1,920 (2⁷·3·5) | 2,480 | 2,304 | 1.08 | 14,279 | 12,385 |
| 2-D 64×64 | 12,421 | 10,986 | 1.13 | 48,812 | 33,336 |
| 2-D 128×128 | 55,124 | 60,079 | **0.92** | 151,529 | 128,914 |
| 2-D 256×256 | 288,030 | 284,430 | **1.01** | 635,683 | 467,818 |
| 2-D 512×512 | 1,266,620 | 1,332,972 | **0.95** | 3,010,417 | 1,939,220 |
| 2-D 1024×1024 | 6,047,132 | 6,776,408 | **0.89** | 14,002,910 | 9,918,276 |

¹ Host load, not the code: the first run read complex 4,096 at 19.7–20.4 µs on
all three repetitions, the second at 10.1–10.9 µs and the two main runs of the
same round at 9.2–9.6 µs, while the load rose from 3.3 to 4.2. The second run
alone gives 1.03. The row is kept as the method computes it.

## Neoverse-N1 (arm64)

cfarm424, 64 cores, used by the round alone; FFTW 3.3.10 built from source with
NEON. Runs `parity-a` and `parity-b` of Round 33 (go-fft v0.21.0, whose
arm64 code v0.23.0 keeps).

- **Within 5% but slower than FFTW:** RFFT 256 (1.046; 1.04 and 1.05 in the two
  runs) and RFFT 1,080 (1.04). Every other row is faster.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 1,015 | 1,160 | **0.88** | 9,440 | 7,683 |
| complex 1,024 | 4,897 | 6,382 | **0.77** | 18,851 | 14,633 |
| complex 4,096 | 23,894 | 41,271 | **0.58** | 60,140 | 48,099 |
| complex 65,536 | 734,460 | 1,267,164 | **0.58** | 2,192,458 | 1,444,663 |
| complex 1,048,576 | 20,482,493 | 45,859,295 | **0.45** | 45,526,432 | 30,524,594 |
| complex 1,000 (2³·5³) | 6,114 | 7,179 | **0.85** | 19,435 | 15,830 |
| complex 1,080 (2³·3³·5) | 6,936 | 7,725 | **0.90** | 22,178 | 17,065 |
| complex 1,920 (2⁷·3·5) | 11,972 | 13,554 | **0.88** | 33,340 | 26,210 |
| complex 1,009 (prime, Rader) | 21,612 | 48,139 | **0.45** | 97,321 | 57,534 |
| complex 1,296 (2⁴·3⁴) | 8,668 | 11,413 | **0.76** | 24,879 | 19,746 |
| complex 10,007 (prime, Bluestein) | 425,244 | 630,409 | **0.68** | 1,036,692 | 758,770 |
| RFFT 256 | 753 | 720 | **1.046** | 8,718 | 7,817 |
| RFFT 1,024 | 3,123 | 4,228 | **0.74** | 14,202 | 12,506 |
| RFFT 4,096 | 14,795 | 19,946 | **0.74** | 36,984 | 32,924 |
| RFFT 65,536 | 397,248 | 557,735 | **0.71** | 722,644 | 756,900 |
| RFFT 1,048,576 | 10,207,510 | 19,139,423 | **0.53** | 20,971,907 | 18,595,691 |
| RFFT 1,000 (2³·5³) | 3,874 | 3,949 | **0.98** | 15,182 | 12,632 |
| RFFT 1,080 (2³·3³·5) | 4,370 | 4,217 | **1.04** | 15,945 | 14,092 |
| RFFT 1,920 (2⁷·3·5) | 7,135 | 7,326 | **0.97** | 21,826 | 18,904 |
| 2-D 64×64 | 24,715 | 31,180 | **0.79** | 70,788 | 50,072 |
| 2-D 128×128 | 129,414 | 196,285 | **0.66** | 253,527 | 201,876 |
| 2-D 256×256 | 625,272 | 1,323,000 | **0.47** | 1,141,482 | 939,314 |
| 2-D 512×512 | 3,256,792 | 7,169,069 | **0.45** | 5,617,647 | 4,004,770 |
| 2-D 1024×1024 | 18,542,260 | 44,847,924 | **0.41** | 26,555,379 | 20,395,201 |

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, used by the round alone (load 0.8–1.3); FFTW 3.3.10 built
from source with AVX-512 codelets. Runs `out-br1` and `out-br2` of Round 31;
FFTW over `out-main1`, `out-br1`, `out-br2`, `out-main2`.

- **Within 5% but slower than FFTW:** 2-D 64² (1.045).
- **Slower by more:** complex 256 (1.27), 1,024 (1.18), 65,536 (1.16), 1,000
  (1.11), 1,080 (1.26), 1,920 (1.11), 1,296 (1.20); RFFT 256 (1.08), 4,096
  (1.051), 65,536 (1.14), 1,048,576 (1.13); 2-D 128² (1.06), 512² (1.17).
- FFTW's 2-D times varied twofold over the four runs (1024²: 13.7–27.3 ms), and
  go-fft's 2-D 512² read 3.1 ms in one run and 2.4 ms in the other (allocation
  placement); read the 2-D rows with that margin.

| transform | go-fft | FFTW | go-fft ÷ FFTW | numpy.fft | scipy.fft |
|:--|--:|--:|--:|--:|--:|
| complex 256 | 466 | 366 | 1.27 | 8,850 | 6,448 |
| complex 1,024 | 2,494 | 2,120 | 1.18 | 16,182 | 12,694 |
| complex 4,096 | 12,232 | 13,007 | **0.94** | 57,985 | 46,010 |
| complex 65,536 | 472,172 | 406,752 | 1.16 | 1,864,806 | 1,383,708 |
| complex 1,048,576 | 21,181,875 | 26,872,067 | **0.79** | 50,144,731 | 41,553,796 |
| complex 1,000 (2³·5³) | 3,026 | 2,731 | 1.11 | 16,970 | 13,983 |
| complex 1,080 (2³·3³·5) | 3,764 | 2,999 | 1.25 | 20,415 | 13,697 |
| complex 1,920 (2⁷·3·5) | 5,490 | 4,951 | 1.11 | 31,320 | 23,715 |
| complex 1,009 (prime, Rader) | 18,710 | 30,713 | **0.61** | 81,722 | 57,009 |
| complex 1,296 (2⁴·3⁴) | 4,405 | 3,665 | 1.20 | 21,866 | 17,339 |
| complex 10,007 (prime, Bluestein) | 334,018 | 363,644 | **0.92** | 952,147 | 733,422 |
| RFFT 256 | 391 | 360 | 1.08 | 6,577 | 6,937 |
| RFFT 1,024 | 1,296 | 1,519 | **0.85** | 12,976 | 11,275 |
| RFFT 4,096 | 7,962 | 7,578 | 1.051 | 33,823 | 31,658 |
| RFFT 65,536 | 215,633 | 189,957 | 1.14 | 602,797 | 694,274 |
| RFFT 1,048,576 | 9,941,053 | 8,766,969 | 1.13 | 26,441,303 | 24,048,265 |
| RFFT 1,000 (2³·5³) | 1,860 | 1,888 | **0.98** | 13,578 | 11,803 |
| RFFT 1,080 (2³·3³·5) | 2,074 | 2,234 | **0.93** | 13,762 | 12,637 |
| RFFT 1,920 (2⁷·3·5) | 3,438 | 3,471 | **0.99** | 18,158 | 17,621 |
| 2-D 64×64 | 17,634 | 16,877 | **1.04** | 54,945 | 45,535 |
| 2-D 128×128 | 81,410 | 77,026 | 1.06 | 208,670 | 174,065 |
| 2-D 256×256 | 469,853 | 482,312 | **0.97** | 904,767 | 758,912 |
| 2-D 512×512 | 2,792,684 | 2,382,100 | 1.17 | 5,249,498 | 3,551,813 |
| 2-D 1024×1024 | 16,070,592 | 21,490,832 | **0.75** | 30,484,511 | 26,410,621 |

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 on 22 of 24 rows, composites and 2-D included (1,296: 0.76×,
    2-D 128²: 0.66×), and within 5% on the other two (RFFT 256, RFFT 1,080);
  * Zen 3 from 65,536 points (0.55–0.91×), 2-D from 128² on one core
    (0.89–1.01×), and complex 1,920 (0.98×);
  * primes on every host: Rader's 1,009 in 0.45–0.66× FFTW's time.
* **Where FFTW leads:**
  * small sizes on amd64: complex 256 at 1.09× on Zen 3 and 1.27× on Cascade
    Lake, RFFT 256 at 1.07× and 1.08×;
  * on Cascade Lake, complex 1,296 (1.20×; the split layout has no radix-3 or
    radix-12 kernel), the other composites (1.11–1.26×) and the large real
    transforms (1.13–1.14×).
* **Fixed since the previous version of this page:** 2-D 1024² on Cascade
  Lake, slowed to 0.94× of v0.18.0's speed by v0.19.0, runs 1.03× v0.18.0's
  speed since v0.22.0 (0.75× FFTW's time here).
* **Against numpy and scipy:** go-fft is at or above both on every row, in every
  run, on all three hosts.
* **Against gonum:** go-fft is faster on every row measured (19/19 on each host).
* **Not in these tables:** Haswell (Intel AVX2-only) has no FFTW on its host; at
  v0.21.0 it runs complex 512–2^18 1.03–1.21× and the composites 1000–6000
  1.07–1.21× faster than v0.19.1 (Round 29). Apple M4 has not been measured
  with the current engines, and no loong64 host could be reached.
* **Single precision:** float32 takes 0.46–0.83× the float64 time on Zen 3 and
  0.58–0.73× on Neoverse-N1. FFTW's single-precision library is not built on
  the hosts, so there is no float32 comparison with it.
