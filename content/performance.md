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
with go1.27.1: **go-fft v0.8.0**'s code on Zen 3 and Cascade Lake (Round 17 of
BENCHMARKS.md), and the **NEON passes of v0.7.0** on Neoverse-N1 (Round 18).
The protocol differs on one point, stated under each host: the 2-D rows ran on
one core on the two amd64 hosts, on all cores on Neoverse-N1. The raw runs are in
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

cfarm420, 128 threads, load average 2–5; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. go-fft v0.8.0 code (Round 17), every row on one core, the 2-D rows included.

- **vs FFTW (native C, gold standard)**: 9/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 401 (25.5) | 286 (35.8) | 6,744 (1.5) | 5,335 (1.9) | 5,570 (1.8) | 1.41× | lags FFTW 1.41× |
| 1,024 (2¹⁰) | 2,057 (24.9) | 1,734 (29.5) | 12,888 (4.0) | 9,439 (5.4) | 28,806 (1.8) | 1.19× | lags FFTW 1.19× |
| 4,096 (2¹²) | 12,823 (19.2) | 10,109 (24.3) | 46,469 (5.3) | 32,964 (7.5) | 132,503 (1.9) | 1.27× | lags FFTW 1.27× |
| 65,536 (2¹⁶) | 297,569 (17.6) | 291,379 (18.0) | 1,882,640 (2.8) | 938,437 (5.6) | 2,895,850 (1.8) | 1.02× | **≥ parity** |
| 1,048,576 (2²⁰) | 6,436,742 (16.3) | 11,157,857 (9.4) | 22,740,294 (4.6) | 14,094,420 (7.4) | 59,842,748 (1.8) | 0.58× | **≥ parity** |
| 1,000 (2³·5³) | 2,391 (20.8) | 2,188 (22.8) | 13,648 (3.7) | 10,145 (4.9) | 29,777 (1.7) | 1.09× | lags FFTW 1.09× |
| 1,080 (2³·3³·5) | 2,876 (18.9) | 2,321 (23.4) | 14,100 (3.9) | 10,704 (5.1) | 35,560 (1.5) | 1.24× | lags FFTW 1.24× |
| 1,920 (2⁷·3·5) | 4,649 (22.5) | 3,818 (27.4) | 21,689 (4.8) | 16,306 (6.4) | 63,637 (1.6) | 1.22× | lags FFTW 1.22× |
| 1,009 (prime) | 12,988 (3.9) | 20,103 (2.5) | 59,440 (0.8) | 37,217 (1.4) | 1,153,476 (0.0) | 0.65× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 3,666 (18.3) | 3,080 (21.8) | 16,392 (4.1) | 12,240 (5.5) | 41,888 (1.6) | 1.19× | lags FFTW 1.19× |
| 10,007 (prime) | 252,134 (2.6) | 209,411 (3.2) | 653,129 (1.0) | 466,779 (1.4) | 121,112,274 (0.0) | 1.20× | lags FFTW 1.20× |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 316 (16.2) | 222 (23.0) | 6,055 (0.8) | 5,218 (1.0) | 2,613 (2.0) | 1.42× | lags FFTW 1.42× |
| 1,024 (2¹⁰) | 1,249 (20.5) | 1,038 (24.7) | 9,451 (2.7) | 8,293 (3.1) | 12,751 (2.0) | 1.20× | lags FFTW 1.20× |
| 4,096 (2¹²) | 6,696 (18.4) | 5,192 (23.7) | 25,466 (4.8) | 22,383 (5.5) | 61,206 (2.0) | 1.29× | lags FFTW 1.29× |
| 65,536 (2¹⁶) | 163,443 (16.0) | 168,232 (15.6) | 397,263 (6.6) | 491,324 (5.3) | 1,351,758 (1.9) | 0.97× | **≥ parity** |
| 1,048,576 (2²⁰) | 3,368,360 (15.6) | 3,621,895 (14.5) | 8,300,800 (6.3) | 10,219,383 (5.1) | 28,513,298 (1.8) | 0.93× | **≥ parity** |
| 1,000 (2³·5³) | 1,515 (16.4) | 1,232 (20.2) | 10,365 (2.4) | 8,895 (2.8) | 13,699 (1.8) | 1.23× | lags FFTW 1.23× |
| 1,080 (2³·3³·5) | 1,733 (15.7) | 1,323 (20.6) | 11,037 (2.5) | 9,282 (2.9) | 15,959 (1.7) | 1.31× | lags FFTW 1.31× |
| 1,920 (2⁷·3·5) | 2,821 (18.6) | 2,351 (22.3) | 14,697 (3.6) | 12,696 (4.1) | 28,023 (1.9) | 1.20× | lags FFTW 1.20× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 335 (15.3) | 353 (14.5) | 0.95× | **≥ parity** |
| 1,024 (2¹⁰) | 1,278 (20.0) | 1,164 (22.0) | 1.10× | lags FFTW 1.10× |
| 4,096 (2¹²) | 6,833 (18.0) | 6,146 (20.0) | 1.11× | lags FFTW 1.11× |
| 65,536 (2¹⁶) | 164,218 (16.0) | 167,106 (15.7) | 0.98× | **≥ parity** |
| 1,048,576 (2²⁰) | 3,604,579 (14.5) | 4,075,471 (12.9) | 0.88× | **≥ parity** |
| 1,000 (2³·5³) | 1,577 (15.8) | 1,423 (17.5) | 1.11× | lags FFTW 1.11× |
| 1,080 (2³·3³·5) | 1,742 (15.6) | 1,708 (15.9) | 1.02× | **≥ parity** |
| 1,920 (2⁷·3·5) | 2,932 (17.9) | 3,084 (17.0) | 0.95× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 10,818 (22.7) | 15,047 (16.3) | 12,001 (20.5) | 49,681 (4.9) | 33,642 (7.3) | 0.90× | **≥ parity** |
| 128x128 | 55,124 (20.8) | 69,689 (16.5) | 61,732 (18.6) | 142,413 (8.1) | 129,620 (8.8) | 0.89× | **≥ parity** |
| 256x256 | 285,933 (18.3) | 345,785 (15.2) | 286,703 (18.3) | 639,109 (8.2) | 454,439 (11.5) | 1.00× | **≥ parity** |
| 512x512 | 1,333,251 (17.7) | 1,493,579 (15.8) | 1,256,457 (18.8) | 2,903,627 (8.1) | 1,869,173 (12.6) | 1.06× | lags FFTW 1.06× |
| 1024x1024 | 5,869,450 (17.9) | 7,428,613 (14.1) | 6,448,914 (16.3) | 13,542,561 (7.7) | 9,748,028 (10.8) | 0.91× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 256 (2⁸): 1.42×
- complex 256 (2⁸): 1.41×
- real 1,080 (2³·3³·5): 1.31×
- real 4,096 (2¹²): 1.29×
- complex 4,096 (2¹²): 1.27×
- complex 1,080 (2³·3³·5): 1.24×
- real 1,000 (2³·5³): 1.23×
- complex 1,920 (2⁷·3·5): 1.22×
- complex 10,007 (prime): 1.20×
- real 1,024 (2¹⁰): 1.20×
- real 1,920 (2⁷·3·5): 1.20×
- complex 1,296 (2⁴·3⁴): 1.19×
- complex 1,024 (2¹⁰): 1.19×
- complex 1,000 (2³·5³): 1.09×
- 2-D 512x512: 1.06×

## Neoverse-N1 (arm64)

cfarm424, 64 cores, load average < 1.2; FFTW 3.3.10 built from source with NEON. go-fft with the NEON passes of v0.7.0 (Round 18); 1-D rows on one core, 2-D rows on all cores, before v0.8.0's column passes.

- **vs FFTW (native C, gold standard)**: 13/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 23/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 22/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,445 (7.1) | 1,151 (8.9) | 9,367 (1.1) | 7,657 (1.3) | 6,049 (1.7) | 1.26× | lags FFTW 1.26× |
| 1,024 (2¹⁰) | 7,040 (7.3) | 6,312 (8.1) | 18,917 (2.7) | 14,623 (3.5) | 30,297 (1.7) | 1.12× | lags FFTW 1.12× |
| 4,096 (2¹²) | 34,087 (7.2) | 40,865 (6.0) | 59,655 (4.1) | 47,636 (5.2) | 148,739 (1.7) | 0.83× | **≥ parity** |
| 65,536 (2¹⁶) | 815,171 (6.4) | 1,223,530 (4.3) | 2,300,408 (2.3) | 1,511,052 (3.5) | 3,566,263 (1.5) | 0.67× | **≥ parity** |
| 1,048,576 (2²⁰) | 22,505,663 (4.7) | 44,372,311 (2.4) | 48,392,375 (2.2) | 31,268,249 (3.4) | 78,471,867 (1.3) | 0.51× | **≥ parity** |
| 1,000 (2³·5³) | 7,641 (6.5) | 7,156 (7.0) | 19,322 (2.6) | 15,236 (3.3) | 33,919 (1.5) | 1.07× | lags FFTW 1.07× |
| 1,080 (2³·3³·5) | 9,145 (6.0) | 7,663 (7.1) | 21,620 (2.5) | 16,824 (3.2) | 41,417 (1.3) | 1.19× | lags FFTW 1.19× |
| 1,920 (2⁷·3·5) | 15,745 (6.7) | 13,492 (7.8) | 32,691 (3.2) | 25,986 (4.0) | 72,872 (1.4) | 1.17× | lags FFTW 1.17× |
| 1,009 (prime) | 23,356 (2.2) | 48,037 (1.0) | 96,036 (0.5) | 57,098 (0.9) | 2,321,839 (0.0) | 0.49× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 11,517 (5.8) | 11,314 (5.9) | 24,889 (2.7) | 19,788 (3.4) | 48,705 (1.4) | 1.02× | **≥ parity** |
| 10,007 (prime) | 527,937 (1.3) | 591,258 (1.1) | 1,042,148 (0.6) | 784,284 (0.8) | 227,908,718 (0.0) | 0.89× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 967 (5.3) | 719 (7.1) | 9,270 (0.6) | 8,385 (0.6) | 3,136 (1.6) | 1.34× | lags FFTW 1.34× |
| 1,024 (2¹⁰) | 4,146 (6.2) | 4,199 (6.1) | 14,658 (1.7) | 12,691 (2.0) | 15,107 (1.7) | 0.99× | **≥ parity** |
| 4,096 (2¹²) | 20,350 (6.0) | 19,800 (6.2) | 37,180 (3.3) | 33,376 (3.7) | 72,423 (1.7) | 1.03× | **≥ parity** |
| 65,536 (2¹⁶) | 483,162 (5.4) | 548,397 (4.8) | 733,056 (3.6) | 751,738 (3.5) | 1,836,265 (1.4) | 0.88× | **≥ parity** |
| 1,048,576 (2²⁰) | 12,022,117 (4.4) | 20,135,567 (2.6) | 22,317,009 (2.3) | 18,391,511 (2.9) | 44,765,172 (1.2) | 0.60× | **≥ parity** |
| 1,000 (2³·5³) | 4,799 (5.2) | 3,944 (6.3) | 15,217 (1.6) | 12,751 (2.0) | 15,581 (1.6) | 1.22× | lags FFTW 1.22× |
| 1,080 (2³·3³·5) | 5,767 (4.7) | 4,190 (6.5) | 15,963 (1.7) | 14,102 (1.9) | 18,259 (1.5) | 1.38× | lags FFTW 1.38× |
| 1,920 (2⁷·3·5) | 8,904 (5.9) | 7,303 (7.2) | 21,710 (2.4) | 19,016 (2.8) | 30,450 (1.7) | 1.22× | lags FFTW 1.22× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,051 (4.9) | 808 (6.3) | 1.30× | lags FFTW 1.30× |
| 1,024 (2¹⁰) | 4,741 (5.4) | 4,451 (5.8) | 1.07× | lags FFTW 1.07× |
| 4,096 (2¹²) | 21,752 (5.6) | 21,017 (5.8) | 1.03× | **≥ parity** |
| 65,536 (2¹⁶) | 511,944 (5.1) | 636,258 (4.1) | 0.80× | **≥ parity** |
| 1,048,576 (2²⁰) | 13,364,791 (3.9) | 25,935,733 (2.0) | 0.52× | **≥ parity** |
| 1,000 (2³·5³) | 5,258 (4.7) | 4,245 (5.9) | 1.24× | lags FFTW 1.24× |
| 1,080 (2³·3³·5) | 6,134 (4.4) | 4,418 (6.2) | 1.39× | lags FFTW 1.39× |
| 1,920 (2⁷·3·5) | 9,948 (5.3) | 7,845 (6.7) | 1.27× | lags FFTW 1.27× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 53,789 (4.6) | 134,189 (1.8) | 30,935 (7.9) | 71,528 (3.4) | 50,308 (4.9) | 1.74× | lags FFTW 1.74× |
| 128x128 | 393,855 (2.9) | 431,000 (2.7) | 197,991 (5.8) | 251,970 (4.6) | 198,322 (5.8) | 1.99× | lags FFTW 1.99× |
| 256x256 | 845,653 (6.2) | 1,017,437 (5.2) | 1,313,881 (4.0) | 1,089,263 (4.8) | 901,686 (5.8) | 0.64× | **≥ parity** |
| 512x512 | 2,075,061 (11.4) | 2,594,231 (9.1) | 7,766,802 (3.0) | 5,953,796 (4.0) | 4,250,378 (5.6) | 0.27× | **≥ parity** |
| 1024x1024 | 5,853,728 (17.9) | 7,145,597 (14.7) | 44,428,331 (2.4) | 25,670,433 (4.1) | 19,994,544 (5.2) | 0.13× | **≥ parity** |

**Rows behind FFTW, worst first:**

- 2-D 128x128: 1.99×
- 2-D 64x64: 1.74×
- real 1,080 (2³·3³·5): 1.38×
- real 256 (2⁸): 1.34×
- complex 256 (2⁸): 1.26×
- real 1,920 (2⁷·3·5): 1.22×
- real 1,000 (2³·5³): 1.22×
- complex 1,080 (2³·3³·5): 1.19×
- complex 1,920 (2⁷·3·5): 1.17×
- complex 1,024 (2¹⁰): 1.12×
- complex 1,000 (2³·5³): 1.07×

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, load average < 1.3; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/AVX-512/FMA. go-fft v0.8.0 code (Round 17), every row on one core, the 2-D rows included.

- **vs FFTW (native C, gold standard)**: 4/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 506 (20.2) | 373 (27.4) | 8,813 (1.2) | 6,566 (1.6) | 5,548 (1.8) | 1.36× | lags FFTW 1.36× |
| 1,024 (2¹⁰) | 2,765 (18.5) | 2,307 (22.2) | 16,318 (3.1) | 12,187 (4.2) | 29,198 (1.8) | 1.20× | lags FFTW 1.20× |
| 4,096 (2¹²) | 13,256 (18.5) | 13,184 (18.6) | 59,676 (4.1) | 46,152 (5.3) | 133,257 (1.8) | 1.01× | **≥ parity** |
| 65,536 (2¹⁶) | 614,506 (8.5) | 401,443 (13.1) | 2,020,579 (2.6) | 1,408,327 (3.7) | 3,780,052 (1.4) | 1.53× | lags FFTW 1.53× |
| 1,048,576 (2²⁰) | 28,575,615 (3.7) | 26,274,590 (4.0) | 52,316,517 (2.0) | 41,122,196 (2.5) | 100,772,881 (1.0) | 1.09× | lags FFTW 1.09× |
| 1,000 (2³·5³) | 3,389 (14.7) | 2,752 (18.1) | 16,880 (3.0) | 14,108 (3.5) | 30,299 (1.6) | 1.23× | lags FFTW 1.23× |
| 1,080 (2³·3³·5) | 4,141 (13.1) | 3,017 (18.0) | 20,156 (2.7) | 13,124 (4.1) | 35,269 (1.5) | 1.37× | lags FFTW 1.37× |
| 1,920 (2⁷·3·5) | 7,248 (14.4) | 4,985 (21.0) | 30,804 (3.4) | 23,407 (4.5) | 62,335 (1.7) | 1.45× | lags FFTW 1.45× |
| 1,009 (prime) | 18,738 (2.7) | 30,523 (1.6) | 82,143 (0.6) | 56,989 (0.9) | 1,447,068 (0.0) | 0.61× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 5,606 (12.0) | 3,687 (18.2) | 20,887 (3.2) | 15,877 (4.2) | 42,317 (1.6) | 1.52× | lags FFTW 1.52× |
| 10,007 (prime) | 336,791 (2.0) | 361,047 (1.8) | 960,318 (0.7) | 734,594 (0.9) | 141,361,856 (0.0) | 0.93× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 476 (10.7) | 364 (14.1) | 6,786 (0.8) | 6,938 (0.7) | 2,668 (1.9) | 1.31× | lags FFTW 1.31× |
| 1,024 (2¹⁰) | 1,753 (14.6) | 1,497 (17.1) | 13,023 (2.0) | 10,165 (2.5) | 13,664 (1.9) | 1.17× | lags FFTW 1.17× |
| 4,096 (2¹²) | 9,089 (13.5) | 7,438 (16.5) | 33,631 (3.7) | 31,403 (3.9) | 61,101 (2.0) | 1.22× | lags FFTW 1.22× |
| 65,536 (2¹⁶) | 257,281 (10.2) | 193,294 (13.6) | 607,084 (4.3) | 698,752 (3.8) | 1,449,498 (1.8) | 1.33× | lags FFTW 1.33× |
| 1,048,576 (2²⁰) | 14,507,761 (3.6) | 9,464,891 (5.5) | 33,531,421 (1.6) | 22,599,165 (2.3) | 49,884,958 (1.1) | 1.53× | lags FFTW 1.53× |
| 1,000 (2³·5³) | 2,210 (11.3) | 1,880 (13.3) | 14,024 (1.8) | 12,061 (2.1) | 14,280 (1.7) | 1.18× | lags FFTW 1.18× |
| 1,080 (2³·3³·5) | 2,563 (10.6) | 2,032 (13.4) | 13,492 (2.0) | 12,528 (2.2) | 17,076 (1.6) | 1.26× | lags FFTW 1.26× |
| 1,920 (2⁷·3·5) | 4,173 (12.5) | 3,453 (15.2) | 18,725 (2.8) | 17,297 (3.0) | 29,390 (1.8) | 1.21× | lags FFTW 1.21× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 499 (10.3) | 535 (9.6) | 0.93× | **≥ parity** |
| 1,024 (2¹⁰) | 1,809 (14.2) | 1,866 (13.7) | 0.97× | **≥ parity** |
| 4,096 (2¹²) | 9,378 (13.1) | 8,703 (14.1) | 1.08× | lags FFTW 1.08× |
| 65,536 (2¹⁶) | 261,249 (10.0) | 216,207 (12.1) | 1.21× | lags FFTW 1.21× |
| 1,048,576 (2²⁰) | 15,518,795 (3.4) | 10,319,166 (5.1) | 1.50× | lags FFTW 1.50× |
| 1,000 (2³·5³) | 2,274 (11.0) | 2,291 (10.9) | 0.99× | **≥ parity** |
| 1,080 (2³·3³·5) | 2,558 (10.6) | 2,343 (11.6) | 1.09× | lags FFTW 1.09× |
| 1,920 (2⁷·3·5) | 4,167 (12.6) | 3,947 (13.3) | 1.06× | lags FFTW 1.06× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 17,987 (13.7) | 30,679 (8.0) | 14,806 (16.6) | 55,648 (4.4) | 45,646 (5.4) | 1.21× | lags FFTW 1.21× |
| 128x128 | 86,215 (13.3) | 145,025 (7.9) | 71,124 (16.1) | 207,045 (5.5) | 181,449 (6.3) | 1.21× | lags FFTW 1.21× |
| 256x256 | 486,557 (10.8) | 762,958 (6.9) | 455,352 (11.5) | 981,202 (5.3) | 793,981 (6.6) | 1.07× | lags FFTW 1.07× |
| 512x512 | 3,956,199 (6.0) | 5,900,679 (4.0) | 2,572,734 (9.2) | 7,081,572 (3.3) | 5,223,995 (4.5) | 1.54× | lags FFTW 1.54× |
| 1024x1024 | 16,226,482 (6.5) | 19,869,204 (5.3) | 22,871,676 (4.6) | 31,646,437 (3.3) | 28,330,303 (3.7) | 0.71× | **≥ parity** |

**Rows behind FFTW, worst first:**

- 2-D 512x512: 1.54×
- real 1,048,576 (2²⁰): 1.53×
- complex 65,536 (2¹⁶): 1.53×
- complex 1,296 (2⁴·3⁴): 1.52×
- complex 1,920 (2⁷·3·5): 1.45×
- complex 1,080 (2³·3³·5): 1.37×
- complex 256 (2⁸): 1.36×
- real 65,536 (2¹⁶): 1.33×
- real 256 (2⁸): 1.31×
- real 1,080 (2³·3³·5): 1.26×
- complex 1,000 (2³·5³): 1.23×
- real 4,096 (2¹²): 1.22×
- 2-D 64x64: 1.21×
- 2-D 128x128: 1.21×
- real 1,920 (2⁷·3·5): 1.21×
- complex 1,024 (2¹⁰): 1.20×
- real 1,000 (2³·5³): 1.18×
- real 1,024 (2¹⁰): 1.17×
- complex 1,048,576 (2²⁰): 1.09×
- 2-D 256x256: 1.07×

## Honest read

* **Where go-fft leads FFTW:**
  * the largest 1-D transforms on Zen 3 and Neoverse-N1: complex 2^20 runs in
    0.51–0.58× FFTW's time (on Cascade Lake it trails, 1.09×, and 65536 at
    1.53×: throughput halves past 4096 points there, for a reason not yet
    established — see BENCHMARKS.md, Round 13);
  * primes: Rader's 1009 in 0.49–0.65× FFTW's time on all three hosts;
  * on Neoverse-N1 with NEON, from 4096 points up (complex 4096: 0.83×);
  * 2-D: 128² and 1024² on one core on Zen 3 (0.89–0.91×), and the large
    shapes spread across cores on Neoverse-N1 (1024²: 0.13×).
* **Where FFTW leads:**
  * small powers of two: 256 points in 1.26–1.41× FFTW's time;
  * smooth composites (1000, 1080, 1296, 1920): 1.07–1.52×;
  * small 2-D shapes on Neoverse-N1 (128²: 1.99×), whose parallel path is
    slower there than one core; amd64 raised that threshold in v0.8.0.
* **Against numpy and scipy:** go-fft is at or above both on every row on Zen 3
  and on Cascade Lake, and on 23 and 22 of 24 on Neoverse-N1.
* **Against gonum:** go-fft is faster on every row measured (19/19 on each
  host), by the widest margin on primes.
