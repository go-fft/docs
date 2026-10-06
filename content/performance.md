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

The numbers below were measured on 2026-10-06 on three GCC Compile Farm hosts,
with go1.27.1, by the newest parity run on each (BENCHMARKS.md Rounds 23–24):
v0.14.0 on Zen 3, v0.15.0 on Neoverse-N1 and Cascade Lake. v0.16.0 changes only
float32, so they describe v0.16.x. Every row ran on one pinned core, the 2-D
rows included. The raw runs are in
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

cfarm420, 128 threads, load average 2.5–3.6; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. go-fft v0.14.0 (Round 23; v0.15.0 and v0.16.0 change nothing for float64 on AMD), every row on one pinned core.

- **vs FFTW (native C, gold standard)**: 13/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 351 (29.2) | 274 (37.4) | 6,798 (1.5) | 5,573 (1.8) | 5,598 (1.8) | 1.28× | lags FFTW 1.28× |
| 1,024 (2¹⁰) | 1,715 (29.9) | 1,550 (33.0) | 13,337 (3.8) | 9,832 (5.2) | 29,314 (1.7) | 1.11× | lags FFTW 1.11× |
| 4,096 (2¹²) | 9,327 (26.3) | 10,312 (23.8) | 44,564 (5.5) | 34,791 (7.1) | 137,085 (1.8) | 0.90× | **≥ parity** |
| 65,536 (2¹⁶) | 254,383 (20.6) | 292,489 (17.9) | 1,966,543 (2.7) | 924,424 (5.7) | 2,907,079 (1.8) | 0.87× | **≥ parity** |
| 1,048,576 (2²⁰) | 5,467,427 (19.2) | 11,255,953 (9.3) | 28,335,980 (3.7) | 14,391,151 (7.3) | 61,447,493 (1.7) | 0.49× | **≥ parity** |
| 1,000 (2³·5³) | 2,392 (20.8) | 2,026 (24.6) | 14,151 (3.5) | 10,346 (4.8) | 30,323 (1.6) | 1.18× | lags FFTW 1.18× |
| 1,080 (2³·3³·5) | 2,824 (19.3) | 2,227 (24.4) | 14,238 (3.8) | 10,869 (5.0) | 35,426 (1.5) | 1.27× | lags FFTW 1.27× |
| 1,920 (2⁷·3·5) | 4,461 (23.5) | 3,863 (27.1) | 22,105 (4.7) | 16,094 (6.5) | 61,995 (1.7) | 1.15× | lags FFTW 1.15× |
| 1,009 (prime) | 12,938 (3.9) | 20,119 (2.5) | 60,505 (0.8) | 39,112 (1.3) | 1,130,438 (0.0) | 0.64× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 3,588 (18.7) | 2,907 (23.0) | 16,513 (4.1) | 12,282 (5.5) | 42,402 (1.6) | 1.23× | lags FFTW 1.23× |
| 10,007 (prime) | 212,798 (3.1) | 203,640 (3.3) | 631,061 (1.1) | 467,700 (1.4) | 118,186,667 (0.0) | 1.04× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 317 (16.2) | 226 (22.6) | 6,066 (0.8) | 5,286 (1.0) | 2,551 (2.0) | 1.40× | lags FFTW 1.40× |
| 1,024 (2¹⁰) | 1,115 (23.0) | 958 (26.7) | 9,365 (2.7) | 8,180 (3.1) | 13,173 (1.9) | 1.16× | lags FFTW 1.16× |
| 4,096 (2¹²) | 5,834 (21.1) | 5,698 (21.6) | 25,308 (4.9) | 22,060 (5.6) | 60,797 (2.0) | 1.02× | **≥ parity** |
| 65,536 (2¹⁶) | 139,285 (18.8) | 148,963 (17.6) | 404,328 (6.5) | 492,580 (5.3) | 1,330,379 (2.0) | 0.94× | **≥ parity** |
| 1,048,576 (2²⁰) | 2,982,926 (17.6) | 3,711,035 (14.1) | 8,424,702 (6.2) | 10,133,481 (5.2) | 27,002,137 (1.9) | 0.80× | **≥ parity** |
| 1,000 (2³·5³) | 1,484 (16.8) | 1,260 (19.8) | 10,421 (2.4) | 8,769 (2.8) | 13,456 (1.9) | 1.18× | lags FFTW 1.18× |
| 1,080 (2³·3³·5) | 1,708 (15.9) | 1,308 (20.8) | 11,045 (2.5) | 8,962 (3.0) | 16,049 (1.7) | 1.31× | lags FFTW 1.31× |
| 1,920 (2⁷·3·5) | 2,746 (19.1) | 2,308 (22.7) | 14,106 (3.7) | 11,945 (4.4) | 27,518 (1.9) | 1.19× | lags FFTW 1.19× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 310 (16.5) | 282 (18.2) | 1.10× | lags FFTW 1.10× |
| 1,024 (2¹⁰) | 1,189 (21.5) | 1,185 (21.6) | 1.00× | **≥ parity** |
| 4,096 (2¹²) | 6,033 (20.4) | 6,081 (20.2) | 0.99× | **≥ parity** |
| 65,536 (2¹⁶) | 139,832 (18.7) | 168,007 (15.6) | 0.83× | **≥ parity** |
| 1,048,576 (2²⁰) | 2,911,271 (18.0) | 4,204,482 (12.5) | 0.69× | **≥ parity** |
| 1,000 (2³·5³) | 1,569 (15.9) | 1,444 (17.2) | 1.09× | lags FFTW 1.09× |
| 1,080 (2³·3³·5) | 1,723 (15.8) | 1,574 (17.3) | 1.09× | lags FFTW 1.09× |
| 1,920 (2⁷·3·5) | 2,837 (18.5) | 2,743 (19.1) | 1.03× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 10,921 (22.5) | 14,672 (16.8) | 11,596 (21.2) | 47,923 (5.1) | 33,681 (7.3) | 0.94× | **≥ parity** |
| 128x128 | 52,146 (22.0) | 64,307 (17.8) | 61,560 (18.6) | 145,907 (7.9) | 131,501 (8.7) | 0.85× | **≥ parity** |
| 256x256 | 274,582 (19.1) | 321,293 (16.3) | 283,960 (18.5) | 607,283 (8.6) | 458,078 (11.4) | 0.97× | **≥ parity** |
| 512x512 | 1,236,579 (19.1) | 1,425,699 (16.5) | 1,373,258 (17.2) | 2,924,952 (8.1) | 1,903,332 (12.4) | 0.90× | **≥ parity** |
| 1024x1024 | 5,811,776 (18.0) | 7,042,133 (14.9) | 6,681,744 (15.7) | 13,445,512 (7.8) | 9,391,654 (11.2) | 0.87× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 256 (2⁸): 1.40×
- real 1,080 (2³·3³·5): 1.31×
- complex 256 (2⁸): 1.28×
- complex 1,080 (2³·3³·5): 1.27×
- complex 1,296 (2⁴·3⁴): 1.23×
- real 1,920 (2⁷·3·5): 1.19×
- complex 1,000 (2³·5³): 1.18×
- real 1,000 (2³·5³): 1.18×
- real 1,024 (2¹⁰): 1.16×
- complex 1,920 (2⁷·3·5): 1.15×
- complex 1,024 (2¹⁰): 1.11×

## Neoverse-N1 (arm64)

cfarm424, 64 cores, load average < 1.3; FFTW 3.3.10 built from source with NEON. go-fft v0.15.0 (Round 24), every row on one pinned core, the 2-D rows included.

- **vs FFTW (native C, gold standard)**: 21/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,109 (9.2) | 1,181 (8.7) | 10,589 (1.0) | 8,278 (1.2) | 5,813 (1.8) | 0.94× | **≥ parity** |
| 1,024 (2¹⁰) | 5,230 (9.8) | 6,597 (7.8) | 20,239 (2.5) | 15,242 (3.4) | 30,328 (1.7) | 0.79× | **≥ parity** |
| 4,096 (2¹²) | 27,547 (8.9) | 42,183 (5.8) | 61,460 (4.0) | 48,379 (5.1) | 145,857 (1.7) | 0.65× | **≥ parity** |
| 65,536 (2¹⁶) | 734,890 (7.1) | 1,432,597 (3.7) | 2,419,369 (2.2) | 1,508,256 (3.5) | 3,297,895 (1.6) | 0.51× | **≥ parity** |
| 1,048,576 (2²⁰) | 22,678,198 (4.6) | 48,199,943 (2.2) | 45,490,557 (2.3) | 31,628,638 (3.3) | 76,727,370 (1.4) | 0.47× | **≥ parity** |
| 1,000 (2³·5³) | 5,997 (8.3) | 7,353 (6.8) | 19,105 (2.6) | 15,024 (3.3) | 33,712 (1.5) | 0.82× | **≥ parity** |
| 1,080 (2³·3³·5) | 6,811 (8.0) | 7,800 (7.0) | 21,611 (2.5) | 16,596 (3.3) | 41,427 (1.3) | 0.87× | **≥ parity** |
| 1,920 (2⁷·3·5) | 11,867 (8.8) | 13,736 (7.6) | 32,667 (3.2) | 25,799 (4.1) | 69,919 (1.5) | 0.86× | **≥ parity** |
| 1,009 (prime) | 21,361 (2.4) | 48,538 (1.0) | 96,233 (0.5) | 57,954 (0.9) | 2,225,385 (0.0) | 0.44× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 8,722 (7.7) | 11,408 (5.9) | 25,400 (2.6) | 20,205 (3.3) | 48,493 (1.4) | 0.76× | **≥ parity** |
| 10,007 (prime) | 424,271 (1.6) | 606,176 (1.1) | 1,042,818 (0.6) | 773,434 (0.9) | 216,646,738 (0.0) | 0.70× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 823 (6.2) | 724 (7.1) | 9,186 (0.6) | 8,036 (0.6) | 2,993 (1.7) | 1.14× | lags FFTW 1.14× |
| 1,024 (2¹⁰) | 3,544 (7.2) | 4,248 (6.0) | 14,347 (1.8) | 12,383 (2.1) | 15,231 (1.7) | 0.83× | **≥ parity** |
| 4,096 (2¹²) | 16,460 (7.5) | 19,877 (6.2) | 37,335 (3.3) | 33,013 (3.7) | 68,627 (1.8) | 0.83× | **≥ parity** |
| 65,536 (2¹⁶) | 420,017 (6.2) | 575,112 (4.6) | 757,750 (3.5) | 781,033 (3.4) | 1,804,297 (1.5) | 0.73× | **≥ parity** |
| 1,048,576 (2²⁰) | 11,758,743 (4.5) | 22,331,136 (2.3) | 21,774,284 (2.4) | 19,411,334 (2.7) | 45,496,208 (1.2) | 0.53× | **≥ parity** |
| 1,000 (2³·5³) | 4,115 (6.1) | 3,942 (6.3) | 15,526 (1.6) | 12,550 (2.0) | 15,433 (1.6) | 1.04× | **≥ parity** |
| 1,080 (2³·3³·5) | 4,611 (5.9) | 4,199 (6.5) | 15,913 (1.7) | 14,069 (1.9) | 17,846 (1.5) | 1.10× | lags FFTW 1.10× |
| 1,920 (2⁷·3·5) | 7,588 (6.9) | 7,330 (7.1) | 22,306 (2.3) | 18,932 (2.8) | 31,108 (1.7) | 1.04× | **≥ parity** |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 953 (5.4) | 809 (6.3) | 1.18× | lags FFTW 1.18× |
| 1,024 (2¹⁰) | 3,955 (6.5) | 4,438 (5.8) | 0.89× | **≥ parity** |
| 4,096 (2¹²) | 17,798 (6.9) | 21,160 (5.8) | 0.84× | **≥ parity** |
| 65,536 (2¹⁶) | 447,801 (5.9) | 648,285 (4.0) | 0.69× | **≥ parity** |
| 1,048,576 (2²⁰) | 13,229,668 (4.0) | 27,572,530 (1.9) | 0.48× | **≥ parity** |
| 1,000 (2³·5³) | 4,468 (5.6) | 4,323 (5.8) | 1.03× | **≥ parity** |
| 1,080 (2³·3³·5) | 5,004 (5.4) | 4,504 (6.0) | 1.11× | lags FFTW 1.11× |
| 1,920 (2⁷·3·5) | 8,213 (6.4) | 7,895 (6.6) | 1.04× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 35,015 (7.0) | 42,207 (5.8) | 31,621 (7.8) | 69,870 (3.5) | 50,219 (4.9) | 1.11× | lags FFTW 1.11× |
| 128x128 | 142,043 (8.1) | 174,782 (6.6) | 200,040 (5.7) | 250,180 (4.6) | 200,016 (5.7) | 0.71× | **≥ parity** |
| 256x256 | 760,399 (6.9) | 877,469 (6.0) | 1,505,370 (3.5) | 1,029,281 (5.1) | 846,996 (6.2) | 0.51× | **≥ parity** |
| 512x512 | 3,709,414 (6.4) | 4,613,456 (5.1) | 9,019,644 (2.6) | 6,339,961 (3.7) | 4,249,940 (5.6) | 0.41× | **≥ parity** |
| 1024x1024 | 20,004,028 (5.2) | 20,965,761 (5.0) | 50,986,421 (2.1) | 27,137,605 (3.9) | 23,309,490 (4.5) | 0.39× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 256 (2⁸): 1.14×
- 2-D 64x64: 1.11×
- real 1,080 (2³·3³·5): 1.10×

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, load average < 1.3; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/AVX-512/FMA (AVX-512 codelets). go-fft v0.15.0 (Round 24), every row on one pinned core. Sizes beyond 2^19 are noisy on this host (RFFT 2^20 read 0.92× FFTW in Round 20, 1.41× here).

- **vs FFTW (native C, gold standard)**: 6/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 499 (20.5) | 372 (27.5) | 8,693 (1.2) | 6,511 (1.6) | 5,524 (1.9) | 1.34× | lags FFTW 1.34× |
| 1,024 (2¹⁰) | 2,762 (18.5) | 2,167 (23.6) | 16,203 (3.2) | 12,103 (4.2) | 28,566 (1.8) | 1.27× | lags FFTW 1.27× |
| 4,096 (2¹²) | 13,196 (18.6) | 13,275 (18.5) | 64,818 (3.8) | 50,567 (4.9) | 133,808 (1.8) | 0.99× | **≥ parity** |
| 65,536 (2¹⁶) | 463,761 (11.3) | 406,334 (12.9) | 2,034,918 (2.6) | 1,407,701 (3.7) | 3,697,282 (1.4) | 1.14× | lags FFTW 1.14× |
| 1,048,576 (2²⁰) | 22,406,480 (4.7) | 26,949,749 (3.9) | 52,935,115 (2.0) | 42,810,735 (2.4) | 100,291,249 (1.0) | 0.83× | **≥ parity** |
| 1,000 (2³·5³) | 3,299 (15.1) | 2,780 (17.9) | 16,854 (3.0) | 14,259 (3.5) | 30,088 (1.7) | 1.19× | lags FFTW 1.19× |
| 1,080 (2³·3³·5) | 4,195 (13.0) | 2,949 (18.5) | 20,177 (2.7) | 13,432 (4.1) | 35,073 (1.6) | 1.42× | lags FFTW 1.42× |
| 1,920 (2⁷·3·5) | 6,394 (16.4) | 4,937 (21.2) | 31,386 (3.3) | 22,368 (4.7) | 62,258 (1.7) | 1.30× | lags FFTW 1.30× |
| 1,009 (prime) | 18,766 (2.7) | 30,211 (1.7) | 81,157 (0.6) | 57,164 (0.9) | 1,473,616 (0.0) | 0.62× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 4,447 (15.1) | 3,638 (18.4) | 20,819 (3.2) | 15,833 (4.2) | 41,942 (1.6) | 1.22× | lags FFTW 1.22× |
| 10,007 (prime) | 336,074 (2.0) | 362,651 (1.8) | 960,391 (0.7) | 729,061 (0.9) | 145,645,925 (0.0) | 0.93× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 481 (10.6) | 341 (15.0) | 6,541 (0.8) | 6,896 (0.7) | 2,665 (1.9) | 1.41× | lags FFTW 1.41× |
| 1,024 (2¹⁰) | 1,776 (14.4) | 1,524 (16.8) | 12,217 (2.1) | 10,850 (2.4) | 13,471 (1.9) | 1.17× | lags FFTW 1.17× |
| 4,096 (2¹²) | 9,163 (13.4) | 7,434 (16.5) | 32,595 (3.8) | 30,861 (4.0) | 61,024 (2.0) | 1.23× | lags FFTW 1.23× |
| 65,536 (2¹⁶) | 248,412 (10.6) | 203,216 (12.9) | 628,835 (4.2) | 690,553 (3.8) | 1,456,763 (1.8) | 1.22× | lags FFTW 1.22× |
| 1,048,576 (2²⁰) | 13,202,533 (4.0) | 9,377,926 (5.6) | 27,994,073 (1.9) | 22,042,025 (2.4) | 47,707,965 (1.1) | 1.41× | lags FFTW 1.41× |
| 1,000 (2³·5³) | 2,224 (11.2) | 1,864 (13.4) | 14,046 (1.8) | 12,074 (2.1) | 16,457 (1.5) | 1.19× | lags FFTW 1.19× |
| 1,080 (2³·3³·5) | 2,437 (11.2) | 2,165 (12.6) | 13,392 (2.0) | 12,422 (2.2) | 19,633 (1.4) | 1.13× | lags FFTW 1.13× |
| 1,920 (2⁷·3·5) | 3,987 (13.1) | 3,354 (15.6) | 18,182 (2.9) | 17,265 (3.0) | 33,097 (1.6) | 1.19× | lags FFTW 1.19× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 495 (10.3) | 499 (10.3) | 0.99× | **≥ parity** |
| 1,024 (2¹⁰) | 1,767 (14.5) | 1,874 (13.7) | 0.94× | **≥ parity** |
| 4,096 (2¹²) | 9,201 (13.4) | 8,240 (14.9) | 1.12× | lags FFTW 1.12× |
| 65,536 (2¹⁶) | 254,929 (10.3) | 217,122 (12.1) | 1.17× | lags FFTW 1.17× |
| 1,048,576 (2²⁰) | 12,685,175 (4.1) | 10,078,087 (5.2) | 1.26× | lags FFTW 1.26× |
| 1,000 (2³·5³) | 2,273 (11.0) | 2,204 (11.3) | 1.03× | **≥ parity** |
| 1,080 (2³·3³·5) | 2,359 (11.5) | 2,262 (12.0) | 1.04× | **≥ parity** |
| 1,920 (2⁷·3·5) | 4,043 (12.9) | 3,961 (13.2) | 1.02× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 17,747 (13.8) | 30,662 (8.0) | 14,876 (16.5) | 54,454 (4.5) | 46,046 (5.3) | 1.19× | lags FFTW 1.19× |
| 128x128 | 88,788 (12.9) | 145,390 (7.9) | 75,492 (15.2) | 209,632 (5.5) | 171,678 (6.7) | 1.18× | lags FFTW 1.18× |
| 256x256 | 458,918 (11.4) | 728,443 (7.2) | 474,552 (11.0) | 874,517 (6.0) | 757,019 (6.9) | 0.97× | **≥ parity** |
| 512x512 | 3,127,451 (7.5) | 5,366,330 (4.4) | 2,945,041 (8.0) | 6,600,314 (3.6) | 4,332,727 (5.4) | 1.06× | lags FFTW 1.06× |
| 1024x1024 | 20,187,824 (5.2) | 21,233,777 (4.9) | 26,051,359 (4.0) | 30,900,113 (3.4) | 27,850,079 (3.8) | 0.77× | **≥ parity** |

**Rows behind FFTW, worst first:**

- complex 1,080 (2³·3³·5): 1.42×
- real 256 (2⁸): 1.41×
- real 1,048,576 (2²⁰): 1.41×
- complex 256 (2⁸): 1.34×
- complex 1,920 (2⁷·3·5): 1.30×
- complex 1,024 (2¹⁰): 1.27×
- real 4,096 (2¹²): 1.23×
- real 65,536 (2¹⁶): 1.22×
- complex 1,296 (2⁴·3⁴): 1.22×
- real 1,000 (2³·5³): 1.19×
- 2-D 64x64: 1.19×
- real 1,920 (2⁷·3·5): 1.19×
- complex 1,000 (2³·5³): 1.19×
- 2-D 128x128: 1.18×
- real 1,024 (2¹⁰): 1.17×
- complex 65,536 (2¹⁶): 1.14×
- real 1,080 (2³·3³·5): 1.13×
- 2-D 512x512: 1.06×

## Honest read

* **Where go-fft leads FFTW:**
  * Neoverse-N1 almost everywhere: 21 of 24 rows, from 256 points (0.94×) to
    2^20 (0.47×), composites included (1000: 0.82×, 1296: 0.76×);
  * Zen 3 from 4096 points (0.90×) and the large sizes (2^20: 0.49×), RFFT 4096
    (1.02×), 2-D 128² and 1024² on one core (0.85–0.87×);
  * primes on every host: Rader's 1009 in 0.44–0.64× FFTW's time.
* **Where FFTW leads:**
  * small and mid sizes on Intel (Cascade Lake 256–1024: 1.27–1.34×) and 256
    on Zen 3 (1.28×);
  * composites on amd64 (1000, 1296: 1.18–1.23×);
  * Cascade Lake's large real transforms, whose numbers are noisy on that host.
* **Against numpy and scipy:** go-fft is at or above both on every row on all
  three hosts.
* **Against gonum:** go-fft is faster on every row measured (19/19 on each
  host), by the widest margin on primes.
* **Single precision** (Rounds 22, 25): float32 transforms take 0.46–0.83× the
  float64 time on Zen 3, depending on the kind. FFTW's single-precision library
  is not built on the hosts, so there is no float32 comparison with it.
