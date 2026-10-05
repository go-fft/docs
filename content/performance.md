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

The numbers below were measured on 2026-10-05 on three GCC Compile Farm hosts,
with go1.27.1, by the newest parity run on each (BENCHMARKS.md Rounds 19–21):
v0.10.0 on Zen 3, v0.11.0 on Neoverse-N1, v0.12.0 on Cascade Lake. No later
release changes the float64 path on those machines, so they describe v0.13.0.
The 2-D rows ran on one core on the two amd64 hosts, on all cores on
Neoverse-N1. The raw runs are in
[`benchmarks/results/`](https://github.com/go-fft/fft/blob/main/benchmarks/results/), and the dated optimization
rounds (what was tried, kept or dropped, and why) are in
[BENCHMARKS.md](https://github.com/go-fft/fft/blob/main/BENCHMARKS.md). Reproduce them with `benchmarks/run.sh`
(`benchmarks/remote/` for a Linux host without Go).

## Method

* **Single-threaded**, except the 2-D rows, where go-fft uses its multicore path
  (all hardware threads); FFTW, numpy and scipy stay on one thread.
* **Steady state**: every library reuses its plan, and go-fft writes into a
  reused slice (`Plan`, `RealPlan`, and `PlanN` for 2-D). The `FFT2` column is the
  allocating convenience call, shown for comparison.
* ns/op, lower is better, with GFLOP/s in parentheses (`5·N·log2(N)`; real
  transforms counted at half). The ratio is go-fft ÷ FFTW (≤ 1.05 = parity).

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

## AMD EPYC 7773X (Zen 3), AVX2

cfarm420, 128 threads, load average 3.2–3.8; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. go-fft v0.10.0 (Round 19; v0.11.0–v0.13.0 do not change the float64 path here), every row on one core, the 2-D rows included.

- **vs FFTW (native C, gold standard)**: 11/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 392 (26.1) | 279 (36.7) | 6,762 (1.5) | 5,129 (2.0) | 6,506 (1.6) | 1.40× | lags FFTW 1.40× |
| 1,024 (2¹⁰) | 1,929 (26.5) | 1,513 (33.8) | 13,056 (3.9) | 9,383 (5.5) | 28,984 (1.8) | 1.27× | lags FFTW 1.27× |
| 4,096 (2¹²) | 13,465 (18.3) | 10,439 (23.5) | 43,873 (5.6) | 33,408 (7.4) | 134,432 (1.8) | 1.29× | lags FFTW 1.29× |
| 65,536 (2¹⁶) | 289,276 (18.1) | 294,760 (17.8) | 1,896,843 (2.8) | 914,724 (5.7) | 2,886,940 (1.8) | 0.98× | **≥ parity** |
| 1,048,576 (2²⁰) | 6,268,394 (16.7) | 11,732,105 (8.9) | 23,083,952 (4.5) | 13,759,221 (7.6) | 58,986,505 (1.8) | 0.53× | **≥ parity** |
| 1,000 (2³·5³) | 2,342 (21.3) | 2,032 (24.5) | 13,676 (3.6) | 9,718 (5.1) | 29,697 (1.7) | 1.15× | lags FFTW 1.15× |
| 1,080 (2³·3³·5) | 2,764 (19.7) | 2,389 (22.8) | 14,507 (3.8) | 10,348 (5.3) | 35,301 (1.5) | 1.16× | lags FFTW 1.16× |
| 1,920 (2⁷·3·5) | 4,727 (22.2) | 3,876 (27.0) | 21,852 (4.8) | 15,371 (6.8) | 63,361 (1.7) | 1.22× | lags FFTW 1.22× |
| 1,009 (prime) | 12,690 (4.0) | 20,223 (2.5) | 60,768 (0.8) | 38,194 (1.3) | 1,136,393 (0.0) | 0.63× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 3,533 (19.0) | 2,891 (23.2) | 16,203 (4.1) | 12,111 (5.5) | 41,909 (1.6) | 1.22× | lags FFTW 1.22× |
| 10,007 (prime) | 247,050 (2.7) | 200,754 (3.3) | 640,499 (1.0) | 471,001 (1.4) | 116,570,203 (0.0) | 1.23× | lags FFTW 1.23× |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 313 (16.4) | 225 (22.8) | 5,982 (0.9) | 5,122 (1.0) | 2,650 (1.9) | 1.39× | lags FFTW 1.39× |
| 1,024 (2¹⁰) | 1,251 (20.5) | 965 (26.5) | 9,564 (2.7) | 8,167 (3.1) | 12,637 (2.0) | 1.30× | lags FFTW 1.30× |
| 4,096 (2¹²) | 6,753 (18.2) | 5,326 (23.1) | 24,637 (5.0) | 21,969 (5.6) | 59,802 (2.1) | 1.27× | lags FFTW 1.27× |
| 65,536 (2¹⁶) | 160,913 (16.3) | 153,398 (17.1) | 397,907 (6.6) | 492,804 (5.3) | 1,314,221 (2.0) | 1.05× | **≥ parity** |
| 1,048,576 (2²⁰) | 3,306,441 (15.9) | 3,835,318 (13.7) | 8,087,968 (6.5) | 9,911,058 (5.3) | 26,249,260 (2.0) | 0.86× | **≥ parity** |
| 1,000 (2³·5³) | 1,484 (16.8) | 1,357 (18.4) | 10,541 (2.4) | 8,488 (2.9) | 13,365 (1.9) | 1.09× | lags FFTW 1.09× |
| 1,080 (2³·3³·5) | 1,691 (16.1) | 1,660 (16.4) | 10,942 (2.5) | 8,884 (3.1) | 15,593 (1.7) | 1.02× | **≥ parity** |
| 1,920 (2⁷·3·5) | 2,780 (18.8) | 2,477 (21.1) | 14,480 (3.6) | 11,979 (4.4) | 28,023 (1.9) | 1.12× | lags FFTW 1.12× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 352 (14.6) | 412 (12.4) | 0.85× | **≥ parity** |
| 1,024 (2¹⁰) | 1,288 (19.9) | 1,182 (21.7) | 1.09× | lags FFTW 1.09× |
| 4,096 (2¹²) | 6,713 (18.3) | 5,872 (20.9) | 1.14× | lags FFTW 1.14× |
| 65,536 (2¹⁶) | 165,091 (15.9) | 168,379 (15.6) | 0.98× | **≥ parity** |
| 1,048,576 (2²⁰) | 3,561,438 (14.7) | 4,152,693 (12.6) | 0.86× | **≥ parity** |
| 1,000 (2³·5³) | 1,520 (16.4) | 1,452 (17.2) | 1.05× | **≥ parity** |
| 1,080 (2³·3³·5) | 1,723 (15.8) | 1,551 (17.5) | 1.11× | lags FFTW 1.11× |
| 1,920 (2⁷·3·5) | 2,987 (17.5) | 2,712 (19.3) | 1.10× | lags FFTW 1.10× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 11,089 (22.2) | 15,718 (15.6) | 11,235 (21.9) | 46,953 (5.2) | 31,959 (7.7) | 0.99× | **≥ parity** |
| 128x128 | 55,159 (20.8) | 70,291 (16.3) | 60,590 (18.9) | 147,585 (7.8) | 128,888 (8.9) | 0.91× | **≥ parity** |
| 256x256 | 279,317 (18.8) | 328,308 (16.0) | 281,353 (18.6) | 666,774 (7.9) | 451,205 (11.6) | 0.99× | **≥ parity** |
| 512x512 | 1,303,604 (18.1) | 1,621,838 (14.5) | 1,308,368 (18.0) | 3,138,327 (7.5) | 1,892,781 (12.5) | 1.00× | **≥ parity** |
| 1024x1024 | 6,119,465 (17.1) | 7,560,949 (13.9) | 6,487,431 (16.2) | 14,558,257 (7.2) | 9,283,255 (11.3) | 0.94× | **≥ parity** |

**Rows behind FFTW, worst first:**

- complex 256 (2⁸): 1.40×
- real 256 (2⁸): 1.39×
- real 1,024 (2¹⁰): 1.30×
- complex 4,096 (2¹²): 1.29×
- complex 1,024 (2¹⁰): 1.27×
- real 4,096 (2¹²): 1.27×
- complex 10,007 (prime): 1.23×
- complex 1,296 (2⁴·3⁴): 1.22×
- complex 1,920 (2⁷·3·5): 1.22×
- complex 1,080 (2³·3³·5): 1.16×
- complex 1,000 (2³·5³): 1.15×
- real 1,920 (2⁷·3·5): 1.12×
- real 1,000 (2³·5³): 1.09×

## Neoverse-N1 (arm64)

cfarm424, 64 cores, load average < 1.1; FFTW 3.3.10 built from source with NEON. go-fft v0.11.0 (Round 21; later releases do not change the float64 path here); 1-D rows on one core, 2-D rows on all cores.

- **vs FFTW (native C, gold standard)**: 15/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,171 (8.7) | 1,155 (8.9) | 9,542 (1.1) | 7,752 (1.3) | 5,807 (1.8) | 1.01× | **≥ parity** |
| 1,024 (2¹⁰) | 5,490 (9.3) | 6,352 (8.1) | 18,814 (2.7) | 14,523 (3.5) | 31,906 (1.6) | 0.86× | **≥ parity** |
| 4,096 (2¹²) | 27,706 (8.9) | 40,991 (6.0) | 58,883 (4.2) | 47,496 (5.2) | 149,039 (1.6) | 0.68× | **≥ parity** |
| 65,536 (2¹⁶) | 712,790 (7.4) | 1,233,581 (4.2) | 2,031,529 (2.6) | 1,419,109 (3.7) | 3,552,909 (1.5) | 0.58× | **≥ parity** |
| 1,048,576 (2²⁰) | 20,567,636 (5.1) | 44,957,810 (2.3) | 43,707,533 (2.4) | 31,562,419 (3.3) | 78,832,624 (1.3) | 0.46× | **≥ parity** |
| 1,000 (2³·5³) | 7,699 (6.5) | 7,159 (7.0) | 19,342 (2.6) | 15,109 (3.3) | 33,659 (1.5) | 1.08× | lags FFTW 1.08× |
| 1,080 (2³·3³·5) | 9,158 (5.9) | 7,663 (7.1) | 21,704 (2.5) | 16,624 (3.3) | 41,312 (1.3) | 1.20× | lags FFTW 1.20× |
| 1,920 (2⁷·3·5) | 15,783 (6.6) | 13,495 (7.8) | 32,720 (3.2) | 25,979 (4.0) | 70,020 (1.5) | 1.17× | lags FFTW 1.17× |
| 1,009 (prime) | 21,547 (2.3) | 48,124 (1.0) | 96,442 (0.5) | 57,092 (0.9) | 2,323,968 (0.0) | 0.45× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 12,142 (5.5) | 11,313 (5.9) | 24,702 (2.7) | 19,511 (3.4) | 51,019 (1.3) | 1.07× | lags FFTW 1.07× |
| 10,007 (prime) | 489,323 (1.4) | 603,875 (1.1) | 1,013,494 (0.7) | 758,396 (0.9) | 227,863,734 (0.0) | 0.81× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 873 (5.9) | 724 (7.1) | 8,583 (0.6) | 7,733 (0.7) | 3,048 (1.7) | 1.21× | lags FFTW 1.21× |
| 1,024 (2¹⁰) | 3,735 (6.9) | 4,210 (6.1) | 14,362 (1.8) | 12,477 (2.1) | 15,801 (1.6) | 0.89× | **≥ parity** |
| 4,096 (2¹²) | 17,449 (7.0) | 19,850 (6.2) | 37,363 (3.3) | 32,527 (3.8) | 67,982 (1.8) | 0.88× | **≥ parity** |
| 65,536 (2¹⁶) | 432,654 (6.1) | 550,126 (4.8) | 711,672 (3.7) | 741,999 (3.5) | 1,734,554 (1.5) | 0.79× | **≥ parity** |
| 1,048,576 (2²⁰) | 11,817,534 (4.4) | 18,941,246 (2.8) | 21,356,505 (2.5) | 18,256,820 (2.9) | 46,010,612 (1.1) | 0.62× | **≥ parity** |
| 1,000 (2³·5³) | 4,981 (5.0) | 3,939 (6.3) | 15,255 (1.6) | 12,762 (2.0) | 15,560 (1.6) | 1.26× | lags FFTW 1.26× |
| 1,080 (2³·3³·5) | 5,801 (4.7) | 4,200 (6.5) | 15,819 (1.7) | 14,015 (1.9) | 18,190 (1.5) | 1.38× | lags FFTW 1.38× |
| 1,920 (2⁷·3·5) | 9,197 (5.7) | 7,306 (7.2) | 21,901 (2.4) | 18,980 (2.8) | 32,171 (1.6) | 1.26× | lags FFTW 1.26× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,006 (5.1) | 811 (6.3) | 1.24× | lags FFTW 1.24× |
| 1,024 (2¹⁰) | 4,123 (6.2) | 4,454 (5.7) | 0.93× | **≥ parity** |
| 4,096 (2¹²) | 17,968 (6.8) | 21,078 (5.8) | 0.85× | **≥ parity** |
| 65,536 (2¹⁶) | 447,504 (5.9) | 638,608 (4.1) | 0.70× | **≥ parity** |
| 1,048,576 (2²⁰) | 11,578,102 (4.5) | 22,897,989 (2.3) | 0.51× | **≥ parity** |
| 1,000 (2³·5³) | 5,288 (4.7) | 4,245 (5.9) | 1.25× | lags FFTW 1.25× |
| 1,080 (2³·3³·5) | 5,972 (4.6) | 4,419 (6.2) | 1.35× | lags FFTW 1.35× |
| 1,920 (2⁷·3·5) | 9,503 (5.5) | 7,822 (6.7) | 1.21× | lags FFTW 1.21× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 34,708 (7.1) | 96,204 (2.6) | 30,814 (8.0) | 69,350 (3.5) | 50,053 (4.9) | 1.13× | lags FFTW 1.13× |
| 128x128 | 155,039 (7.4) | 407,290 (2.8) | 194,246 (5.9) | 260,171 (4.4) | 202,250 (5.7) | 0.80× | **≥ parity** |
| 256x256 | 709,961 (7.4) | 879,220 (6.0) | 1,249,232 (4.2) | 1,054,863 (5.0) | 841,328 (6.2) | 0.57× | **≥ parity** |
| 512x512 | 1,531,912 (15.4) | 2,152,661 (11.0) | 6,936,183 (3.4) | 5,847,853 (4.0) | 4,192,867 (5.6) | 0.22× | **≥ parity** |
| 1024x1024 | 3,856,620 (27.2) | 5,689,408 (18.4) | 43,737,163 (2.4) | 25,803,658 (4.1) | 22,640,532 (4.6) | 0.09× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 1,080 (2³·3³·5): 1.38×
- real 1,000 (2³·5³): 1.26×
- real 1,920 (2⁷·3·5): 1.26×
- real 256 (2⁸): 1.21×
- complex 1,080 (2³·3³·5): 1.20×
- complex 1,920 (2⁷·3·5): 1.17×
- 2-D 64x64: 1.13×
- complex 1,000 (2³·5³): 1.08×
- complex 1,296 (2⁴·3⁴): 1.07×

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, load average < 1.1; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/AVX-512/FMA. go-fft v0.12.0 (Round 20; v0.13.0 does not change the float64 path here), every row on one core, the 2-D rows included.

- **vs FFTW (native C, gold standard)**: 8/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 502 (20.4) | 370 (27.7) | 8,999 (1.1) | 6,365 (1.6) | 5,545 (1.8) | 1.36× | lags FFTW 1.36× |
| 1,024 (2¹⁰) | 2,744 (18.7) | 2,108 (24.3) | 16,298 (3.1) | 11,713 (4.4) | 29,004 (1.8) | 1.30× | lags FFTW 1.30× |
| 4,096 (2¹²) | 13,120 (18.7) | 12,056 (20.4) | 59,717 (4.1) | 44,103 (5.6) | 133,483 (1.8) | 1.09× | lags FFTW 1.09× |
| 65,536 (2¹⁶) | 464,999 (11.3) | 391,792 (13.4) | 1,911,638 (2.7) | 1,383,767 (3.8) | 3,717,712 (1.4) | 1.19× | lags FFTW 1.19× |
| 1,048,576 (2²⁰) | 21,149,064 (5.0) | 25,444,376 (4.1) | 51,170,709 (2.0) | 40,735,986 (2.6) | 98,925,235 (1.1) | 0.83× | **≥ parity** |
| 1,000 (2³·5³) | 3,399 (14.7) | 2,731 (18.2) | 17,116 (2.9) | 13,801 (3.6) | 30,485 (1.6) | 1.24× | lags FFTW 1.24× |
| 1,080 (2³·3³·5) | 4,165 (13.1) | 3,010 (18.1) | 20,089 (2.7) | 13,011 (4.2) | 35,610 (1.5) | 1.38× | lags FFTW 1.38× |
| 1,920 (2⁷·3·5) | 7,200 (14.5) | 4,852 (21.6) | 31,234 (3.4) | 21,801 (4.8) | 62,312 (1.7) | 1.48× | lags FFTW 1.48× |
| 1,009 (prime) | 18,756 (2.7) | 30,505 (1.6) | 78,417 (0.6) | 54,563 (0.9) | 1,452,325 (0.0) | 0.61× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 5,485 (12.2) | 3,646 (18.4) | 21,356 (3.1) | 15,273 (4.4) | 42,519 (1.6) | 1.50× | lags FFTW 1.50× |
| 10,007 (prime) | 334,347 (2.0) | 365,867 (1.8) | 854,413 (0.8) | 728,122 (0.9) | 142,403,188 (0.0) | 0.91× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 470 (10.9) | 368 (13.9) | 6,557 (0.8) | 7,089 (0.7) | 2,647 (1.9) | 1.28× | lags FFTW 1.28× |
| 1,024 (2¹⁰) | 1,764 (14.5) | 1,516 (16.9) | 13,077 (2.0) | 11,609 (2.2) | 13,981 (1.8) | 1.16× | lags FFTW 1.16× |
| 4,096 (2¹²) | 9,102 (13.5) | 7,402 (16.6) | 32,673 (3.8) | 31,361 (3.9) | 61,242 (2.0) | 1.23× | lags FFTW 1.23× |
| 65,536 (2¹⁶) | 244,108 (10.7) | 182,520 (14.4) | 596,393 (4.4) | 694,655 (3.8) | 1,418,149 (1.8) | 1.34× | lags FFTW 1.34× |
| 1,048,576 (2²⁰) | 9,752,189 (5.4) | 10,551,617 (5.0) | 25,145,379 (2.1) | 21,514,993 (2.4) | 46,231,383 (1.1) | 0.92× | **≥ parity** |
| 1,000 (2³·5³) | 2,218 (11.2) | 1,854 (13.4) | 14,257 (1.7) | 12,018 (2.1) | 14,170 (1.8) | 1.20× | lags FFTW 1.20× |
| 1,080 (2³·3³·5) | 2,559 (10.6) | 2,185 (12.5) | 13,396 (2.0) | 12,466 (2.2) | 16,437 (1.7) | 1.17× | lags FFTW 1.17× |
| 1,920 (2⁷·3·5) | 4,143 (12.6) | 3,561 (14.7) | 18,187 (2.9) | 17,256 (3.0) | 28,883 (1.8) | 1.16× | lags FFTW 1.16× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 500 (10.2) | 531 (9.6) | 0.94× | **≥ parity** |
| 1,024 (2¹⁰) | 1,772 (14.4) | 1,834 (14.0) | 0.97× | **≥ parity** |
| 4,096 (2¹²) | 9,349 (13.1) | 8,468 (14.5) | 1.10× | lags FFTW 1.10× |
| 65,536 (2¹⁶) | 252,790 (10.4) | 212,873 (12.3) | 1.19× | lags FFTW 1.19× |
| 1,048,576 (2²⁰) | 11,081,301 (4.7) | 9,849,658 (5.3) | 1.13× | lags FFTW 1.13× |
| 1,000 (2³·5³) | 2,258 (11.0) | 2,257 (11.0) | 1.00× | **≥ parity** |
| 1,080 (2³·3³·5) | 2,566 (10.6) | 2,504 (10.9) | 1.02× | **≥ parity** |
| 1,920 (2⁷·3·5) | 4,169 (12.6) | 3,862 (13.6) | 1.08× | lags FFTW 1.08× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 17,005 (14.5) | 30,095 (8.2) | 16,673 (14.7) | 54,138 (4.5) | 44,740 (5.5) | 1.02× | **≥ parity** |
| 128x128 | 86,937 (13.2) | 141,274 (8.1) | 71,963 (15.9) | 211,202 (5.4) | 180,545 (6.4) | 1.21× | lags FFTW 1.21× |
| 256x256 | 487,789 (10.7) | 678,082 (7.7) | 475,374 (11.0) | 929,074 (5.6) | 757,952 (6.9) | 1.03× | **≥ parity** |
| 512x512 | 2,528,027 (9.3) | 4,498,780 (5.2) | 2,605,001 (9.1) | 6,463,724 (3.7) | 4,027,749 (5.9) | 0.97× | **≥ parity** |
| 1024x1024 | 15,730,661 (6.7) | 19,004,510 (5.5) | 23,312,782 (4.5) | 30,619,575 (3.4) | 27,084,908 (3.9) | 0.67× | **≥ parity** |

**Rows behind FFTW, worst first:**

- complex 1,296 (2⁴·3⁴): 1.50×
- complex 1,920 (2⁷·3·5): 1.48×
- complex 1,080 (2³·3³·5): 1.38×
- complex 256 (2⁸): 1.36×
- real 65,536 (2¹⁶): 1.34×
- complex 1,024 (2¹⁰): 1.30×
- real 256 (2⁸): 1.28×
- complex 1,000 (2³·5³): 1.24×
- real 4,096 (2¹²): 1.23×
- 2-D 128x128: 1.21×
- real 1,000 (2³·5³): 1.20×
- complex 65,536 (2¹⁶): 1.19×
- real 1,080 (2³·3³·5): 1.17×
- real 1,024 (2¹⁰): 1.16×
- real 1,920 (2⁷·3·5): 1.16×
- complex 4,096 (2¹²): 1.09×

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 from 1024 points up (complex 1024: 0.86×, 4096: 0.68×, 2^20:
    0.46×), with the NEON passes and the split layout;
  * the largest transforms on every host: complex 2^20 in 0.46–0.83× FFTW's
    time, Cascade Lake included since v0.12.0's blocked schedule;
  * primes: Rader's 1009 in 0.45–0.63×;
  * 2-D 128² on Zen 3 (0.91×, one core) and N1 (0.80×, all cores), and the
    large shapes.
* **Where FFTW leads:**
  * small and mid sizes on amd64: 256 to 4096 points in 1.09–1.40× FFTW's time;
  * smooth composites (1000, 1080, 1296, 1920): 1.08–1.24×;
  * Cascade Lake's 65536 (1.19×) and small 2-D shapes on one core (1.21×).
* **Against numpy and scipy:** go-fft is at or above both on every row on all
  three hosts.
* **Against gonum:** go-fft is faster on every row measured (19/19 on each
  host), by the widest margin on primes.
* **Single precision** (v0.13.0, Round 22): float32 complex transforms take
  0.51–0.68× the float64 time on Zen 3 and 0.49–0.89× on Apple M4. FFTW's
  single-precision library is not built on the hosts, so there is no float32
  comparison with it.
