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

The numbers below were measured on 2026-10-07 on three GCC Compile Farm hosts,
with go1.27.1, by the newest round on each (BENCHMARKS.md Rounds 26–28): v0.18.0
on Zen 3, v0.17.0 on Neoverse-N1, v0.19.0 on Cascade Lake. No later release
changes the float64 path on those machines. Every row ran on one pinned core.
FFTW's own times move by up to 22% between runs on one host, so read a single
run's ratios with that margin. The raw runs are in
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

cfarm420, 128 threads, load average 1.2–3.8; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA. go-fft v0.18.0 (Round 26, run `parity-br2`; later releases change nothing for float64 on AMD), every row on one pinned core. The README and BENCHMARKS.md tables average two runs; this is one of them.

- **vs FFTW (native C, gold standard)**: 14/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 347 (29.5) | 287 (35.7) | 7,055 (1.5) | 5,473 (1.9) | 5,470 (1.9) | 1.21× | lags FFTW 1.21× |
| 1,024 (2¹⁰) | 1,727 (29.6) | 1,509 (33.9) | 13,542 (3.8) | 9,899 (5.2) | 28,251 (1.8) | 1.14× | lags FFTW 1.14× |
| 4,096 (2¹²) | 9,599 (25.6) | 11,295 (21.8) | 47,171 (5.2) | 34,540 (7.1) | 132,154 (1.9) | 0.85× | **≥ parity** |
| 65,536 (2¹⁶) | 260,897 (20.1) | 298,621 (17.6) | 1,891,459 (2.8) | 914,418 (5.7) | 2,842,938 (1.8) | 0.87× | **≥ parity** |
| 1,048,576 (2²⁰) | 5,970,248 (17.6) | 11,505,252 (9.1) | 27,424,859 (3.8) | 14,850,006 (7.1) | 59,965,099 (1.7) | 0.52× | **≥ parity** |
| 1,000 (2³·5³) | 2,182 (22.8) | 2,060 (24.2) | 14,817 (3.4) | 9,899 (5.0) | 29,897 (1.7) | 1.06× | lags FFTW 1.06× |
| 1,080 (2³·3³·5) | 2,603 (20.9) | 2,322 (23.4) | 14,157 (3.8) | 10,659 (5.1) | 35,262 (1.5) | 1.12× | lags FFTW 1.12× |
| 1,920 (2⁷·3·5) | 3,852 (27.2) | 3,947 (26.5) | 21,865 (4.8) | 16,346 (6.4) | 61,434 (1.7) | 0.98× | **≥ parity** |
| 1,009 (prime) | 13,032 (3.9) | 20,086 (2.5) | 60,741 (0.8) | 38,166 (1.3) | 1,171,675 (0.0) | 0.65× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 2,955 (22.7) | 2,912 (23.0) | 16,206 (4.1) | 12,415 (5.4) | 42,387 (1.6) | 1.01× | **≥ parity** |
| 10,007 (prime) | 210,266 (3.2) | 206,839 (3.2) | 641,946 (1.0) | 463,639 (1.4) | 128,548,175 (0.0) | 1.02× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 296 (17.3) | 227 (22.6) | 5,943 (0.9) | 5,542 (0.9) | 2,551 (2.0) | 1.30× | lags FFTW 1.30× |
| 1,024 (2¹⁰) | 1,124 (22.8) | 979 (26.1) | 9,443 (2.7) | 9,026 (2.8) | 12,648 (2.0) | 1.15× | lags FFTW 1.15× |
| 4,096 (2¹²) | 5,831 (21.1) | 5,248 (23.4) | 25,405 (4.8) | 21,905 (5.6) | 59,229 (2.1) | 1.11× | lags FFTW 1.11× |
| 65,536 (2¹⁶) | 137,990 (19.0) | 154,529 (17.0) | 402,780 (6.5) | 500,832 (5.2) | 1,307,470 (2.0) | 0.89× | **≥ parity** |
| 1,048,576 (2²⁰) | 2,893,913 (18.1) | 3,748,442 (14.0) | 8,287,442 (6.3) | 9,956,538 (5.3) | 27,908,205 (1.9) | 0.77× | **≥ parity** |
| 1,000 (2³·5³) | 1,372 (18.2) | 1,222 (20.4) | 10,186 (2.4) | 8,506 (2.9) | 14,448 (1.7) | 1.12× | lags FFTW 1.12× |
| 1,080 (2³·3³·5) | 1,596 (17.0) | 1,418 (19.2) | 10,742 (2.5) | 9,371 (2.9) | 15,744 (1.7) | 1.13× | lags FFTW 1.13× |
| 1,920 (2⁷·3·5) | 2,564 (20.4) | 2,360 (22.2) | 14,149 (3.7) | 12,189 (4.3) | 28,990 (1.8) | 1.09× | lags FFTW 1.09× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 307 (16.7) | 347 (14.8) | 0.88× | **≥ parity** |
| 1,024 (2¹⁰) | 1,148 (22.3) | 1,191 (21.5) | 0.96× | **≥ parity** |
| 4,096 (2¹²) | 5,930 (20.7) | 5,788 (21.2) | 1.02× | **≥ parity** |
| 65,536 (2¹⁶) | 140,312 (18.7) | 171,179 (15.3) | 0.82× | **≥ parity** |
| 1,048,576 (2²⁰) | 2,882,704 (18.2) | 4,126,716 (12.7) | 0.70× | **≥ parity** |
| 1,000 (2³·5³) | 1,418 (17.6) | 1,426 (17.5) | 0.99× | **≥ parity** |
| 1,080 (2³·3³·5) | 1,654 (16.4) | 1,585 (17.2) | 1.04× | **≥ parity** |
| 1,920 (2⁷·3·5) | 2,675 (19.6) | 2,662 (19.7) | 1.00× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 10,990 (22.4) | 15,450 (15.9) | 10,742 (22.9) | 50,400 (4.9) | 32,844 (7.5) | 1.02× | **≥ parity** |
| 128x128 | 52,771 (21.7) | 71,355 (16.1) | 62,073 (18.5) | 147,509 (7.8) | 127,001 (9.0) | 0.85× | **≥ parity** |
| 256x256 | 275,994 (19.0) | 316,445 (16.6) | 287,810 (18.2) | 594,783 (8.8) | 454,994 (11.5) | 0.96× | **≥ parity** |
| 512x512 | 1,254,306 (18.8) | 1,511,382 (15.6) | 1,373,084 (17.2) | 2,953,141 (8.0) | 1,916,868 (12.3) | 0.91× | **≥ parity** |
| 1024x1024 | 6,846,428 (15.3) | 8,436,663 (12.4) | 6,886,025 (15.2) | 13,316,283 (7.9) | 9,361,665 (11.2) | 0.99× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 256 (2⁸): 1.30×
- complex 256 (2⁸): 1.21×
- real 1,024 (2¹⁰): 1.15×
- complex 1,024 (2¹⁰): 1.14×
- real 1,080 (2³·3³·5): 1.13×
- real 1,000 (2³·5³): 1.12×
- complex 1,080 (2³·3³·5): 1.12×
- real 4,096 (2¹²): 1.11×
- real 1,920 (2⁷·3·5): 1.09×
- complex 1,000 (2³·5³): 1.06×

## Neoverse-N1 (arm64)

cfarm424, 64 cores, load average < 1.2; FFTW 3.3.10 built from source with NEON. go-fft v0.17.0 (Round 27, run `parity-br2`), every row on one pinned core. The README and BENCHMARKS.md tables average two runs; this is one of them.

- **vs FFTW (native C, gold standard)**: 23/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,107 (9.3) | 1,156 (8.9) | 9,673 (1.1) | 7,767 (1.3) | 5,857 (1.7) | 0.96× | **≥ parity** |
| 1,024 (2¹⁰) | 5,290 (9.7) | 6,327 (8.1) | 19,107 (2.7) | 14,683 (3.5) | 30,391 (1.7) | 0.84× | **≥ parity** |
| 4,096 (2¹²) | 27,755 (8.9) | 41,074 (6.0) | 60,620 (4.1) | 48,881 (5.0) | 146,572 (1.7) | 0.68× | **≥ parity** |
| 65,536 (2¹⁶) | 719,022 (7.3) | 1,258,770 (4.2) | 2,217,306 (2.4) | 1,454,544 (3.6) | 3,403,682 (1.5) | 0.57× | **≥ parity** |
| 1,048,576 (2²⁰) | 20,407,904 (5.1) | 44,492,925 (2.4) | 46,317,743 (2.3) | 33,048,267 (3.2) | 74,161,852 (1.4) | 0.46× | **≥ parity** |
| 1,000 (2³·5³) | 6,029 (8.3) | 7,239 (6.9) | 19,354 (2.6) | 15,371 (3.2) | 34,016 (1.5) | 0.83× | **≥ parity** |
| 1,080 (2³·3³·5) | 6,866 (7.9) | 7,771 (7.0) | 21,968 (2.5) | 16,854 (3.2) | 41,586 (1.3) | 0.88× | **≥ parity** |
| 1,920 (2⁷·3·5) | 11,958 (8.8) | 13,787 (7.6) | 33,658 (3.1) | 27,104 (3.9) | 69,794 (1.5) | 0.87× | **≥ parity** |
| 1,009 (prime) | 21,461 (2.3) | 48,543 (1.0) | 98,182 (0.5) | 57,760 (0.9) | 2,241,627 (0.0) | 0.44× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 8,655 (7.7) | 11,552 (5.8) | 25,602 (2.6) | 19,768 (3.4) | 49,528 (1.4) | 0.75× | **≥ parity** |
| 10,007 (prime) | 409,418 (1.6) | 598,675 (1.1) | 1,046,121 (0.6) | 774,515 (0.9) | 220,242,338 (0.0) | 0.68× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 771 (6.6) | 731 (7.0) | 8,681 (0.6) | 7,833 (0.7) | 3,063 (1.7) | 1.05× | lags FFTW 1.05× |
| 1,024 (2¹⁰) | 3,272 (7.8) | 4,220 (6.1) | 14,301 (1.8) | 12,433 (2.1) | 15,534 (1.6) | 0.78× | **≥ parity** |
| 4,096 (2¹²) | 15,492 (7.9) | 20,231 (6.1) | 37,528 (3.3) | 33,173 (3.7) | 68,498 (1.8) | 0.77× | **≥ parity** |
| 65,536 (2¹⁶) | 405,417 (6.5) | 558,973 (4.7) | 718,094 (3.7) | 765,605 (3.4) | 1,787,719 (1.5) | 0.73× | **≥ parity** |
| 1,048,576 (2²⁰) | 10,414,911 (5.0) | 18,816,464 (2.8) | 19,269,423 (2.7) | 18,096,695 (2.9) | 42,235,436 (1.2) | 0.55× | **≥ parity** |
| 1,000 (2³·5³) | 3,856 (6.5) | 4,019 (6.2) | 15,204 (1.6) | 12,663 (2.0) | 14,873 (1.7) | 0.96× | **≥ parity** |
| 1,080 (2³·3³·5) | 4,368 (6.2) | 4,269 (6.4) | 15,962 (1.7) | 14,088 (1.9) | 17,422 (1.6) | 1.02× | **≥ parity** |
| 1,920 (2⁷·3·5) | 7,224 (7.2) | 7,447 (7.0) | 22,100 (2.4) | 19,018 (2.8) | 30,907 (1.7) | 0.97× | **≥ parity** |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 773 (6.6) | 810 (6.3) | 0.95× | **≥ parity** |
| 1,024 (2¹⁰) | 3,375 (7.6) | 4,458 (5.7) | 0.76× | **≥ parity** |
| 4,096 (2¹²) | 15,429 (8.0) | 20,991 (5.9) | 0.74× | **≥ parity** |
| 65,536 (2¹⁶) | 388,297 (6.8) | 657,208 (4.0) | 0.59× | **≥ parity** |
| 1,048,576 (2²⁰) | 10,358,317 (5.1) | 21,323,533 (2.5) | 0.49× | **≥ parity** |
| 1,000 (2³·5³) | 3,903 (6.4) | 4,248 (5.9) | 0.92× | **≥ parity** |
| 1,080 (2³·3³·5) | 4,452 (6.1) | 4,454 (6.1) | 1.00× | **≥ parity** |
| 1,920 (2⁷·3·5) | 7,269 (7.2) | 7,816 (6.7) | 0.93× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 28,060 (8.8) | 33,318 (7.4) | 30,842 (8.0) | 72,575 (3.4) | 50,470 (4.9) | 0.91× | **≥ parity** |
| 128x128 | 127,604 (9.0) | 149,245 (7.7) | 193,020 (5.9) | 258,448 (4.4) | 198,764 (5.8) | 0.66× | **≥ parity** |
| 256x256 | 680,878 (7.7) | 787,979 (6.7) | 1,251,540 (4.2) | 1,187,653 (4.4) | 947,338 (5.5) | 0.54× | **≥ parity** |
| 512x512 | 3,486,009 (6.8) | 4,220,728 (5.6) | 6,747,951 (3.5) | 5,195,350 (4.5) | 4,060,175 (5.8) | 0.52× | **≥ parity** |
| 1024x1024 | 18,091,751 (5.8) | 19,060,544 (5.5) | 44,134,627 (2.4) | 26,344,551 (4.0) | 19,851,258 (5.3) | 0.41× | **≥ parity** |

**Rows behind FFTW, worst first:**

- real 256 (2⁸): 1.05×

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, load average < 1.3; FFTW 3.3.10 built from source with AVX-512 codelets. go-fft v0.19.0 (Round 28, run `parity-br2`), every row on one pinned core. The README and BENCHMARKS.md tables average two runs; this is one of them.

- **vs FFTW (native C, gold standard)**: 10/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 462 (22.2) | 365 (28.1) | 8,849 (1.2) | 6,352 (1.6) | 5,521 (1.9) | 1.27× | lags FFTW 1.27× |
| 1,024 (2¹⁰) | 2,493 (20.5) | 2,109 (24.3) | 16,304 (3.1) | 12,101 (4.2) | 29,043 (1.8) | 1.18× | lags FFTW 1.18× |
| 4,096 (2¹²) | 11,942 (20.6) | 12,828 (19.2) | 57,891 (4.2) | 45,768 (5.4) | 132,665 (1.9) | 0.93× | **≥ parity** |
| 65,536 (2¹⁶) | 457,875 (11.5) | 400,240 (13.1) | 1,844,171 (2.8) | 1,363,184 (3.8) | 3,549,357 (1.5) | 1.14× | lags FFTW 1.14× |
| 1,048,576 (2²⁰) | 21,571,351 (4.9) | 26,037,623 (4.0) | 50,245,144 (2.1) | 42,013,715 (2.5) | 99,546,142 (1.1) | 0.83× | **≥ parity** |
| 1,000 (2³·5³) | 3,354 (14.9) | 2,723 (18.3) | 17,014 (2.9) | 13,874 (3.6) | 30,031 (1.7) | 1.23× | lags FFTW 1.23× |
| 1,080 (2³·3³·5) | 4,192 (13.0) | 3,026 (18.0) | 20,313 (2.7) | 13,872 (3.9) | 34,940 (1.6) | 1.39× | lags FFTW 1.39× |
| 1,920 (2⁷·3·5) | 6,449 (16.2) | 4,887 (21.4) | 31,434 (3.3) | 22,226 (4.7) | 61,637 (1.7) | 1.32× | lags FFTW 1.32× |
| 1,009 (prime) | 18,736 (2.7) | 30,034 (1.7) | 80,530 (0.6) | 56,820 (0.9) | 1,428,089 (0.0) | 0.62× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 4,441 (15.1) | 3,587 (18.7) | 20,758 (3.2) | 15,650 (4.3) | 41,928 (1.6) | 1.24× | lags FFTW 1.24× |
| 10,007 (prime) | 344,044 (1.9) | 357,803 (1.9) | 878,541 (0.8) | 736,130 (0.9) | 139,737,668 (0.0) | 0.96× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 435 (11.8) | 360 (14.2) | 6,497 (0.8) | 6,487 (0.8) | 2,663 (1.9) | 1.21× | lags FFTW 1.21× |
| 1,024 (2¹⁰) | 1,673 (15.3) | 1,516 (16.9) | 13,100 (2.0) | 10,272 (2.5) | 13,181 (1.9) | 1.10× | lags FFTW 1.10× |
| 4,096 (2¹²) | 9,178 (13.4) | 7,640 (16.1) | 32,569 (3.8) | 31,441 (3.9) | 62,050 (2.0) | 1.20× | lags FFTW 1.20× |
| 65,536 (2¹⁶) | 254,541 (10.3) | 186,699 (14.0) | 592,952 (4.4) | 682,752 (3.8) | 1,438,358 (1.8) | 1.36× | lags FFTW 1.36× |
| 1,048,576 (2²⁰) | 9,763,710 (5.4) | 9,924,578 (5.3) | 17,422,845 (3.0) | 20,909,599 (2.5) | 41,952,695 (1.2) | 0.98× | **≥ parity** |
| 1,000 (2³·5³) | 2,215 (11.2) | 1,854 (13.4) | 12,692 (2.0) | 10,720 (2.3) | 14,208 (1.8) | 1.19× | lags FFTW 1.19× |
| 1,080 (2³·3³·5) | 2,373 (11.5) | 1,995 (13.6) | 13,428 (2.0) | 10,924 (2.5) | 16,482 (1.7) | 1.19× | lags FFTW 1.19× |
| 1,920 (2⁷·3·5) | 3,936 (13.3) | 3,379 (15.5) | 17,965 (2.9) | 17,509 (3.0) | 28,949 (1.8) | 1.16× | lags FFTW 1.16× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 456 (11.2) | 502 (10.2) | 0.91× | **≥ parity** |
| 1,024 (2¹⁰) | 1,737 (14.7) | 1,913 (13.4) | 0.91× | **≥ parity** |
| 4,096 (2¹²) | 9,052 (13.6) | 8,380 (14.7) | 1.08× | lags FFTW 1.08× |
| 65,536 (2¹⁶) | 248,898 (10.5) | 213,591 (12.3) | 1.17× | lags FFTW 1.17× |
| 1,048,576 (2²⁰) | 11,766,175 (4.5) | 9,201,246 (5.7) | 1.28× | lags FFTW 1.28× |
| 1,000 (2³·5³) | 2,261 (11.0) | 2,270 (11.0) | 1.00× | **≥ parity** |
| 1,080 (2³·3³·5) | 2,372 (11.5) | 2,324 (11.7) | 1.02× | **≥ parity** |
| 1,920 (2⁷·3·5) | 4,071 (12.9) | 4,039 (13.0) | 1.01× | **≥ parity** |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 17,544 (14.0) | 30,544 (8.0) | 17,140 (14.3) | 54,722 (4.5) | 45,883 (5.4) | 1.02× | **≥ parity** |
| 128x128 | 79,554 (14.4) | 136,478 (8.4) | 79,329 (14.5) | 212,902 (5.4) | 181,062 (6.3) | 1.00× | **≥ parity** |
| 256x256 | 471,750 (11.1) | 674,067 (7.8) | 452,062 (11.6) | 861,663 (6.1) | 752,781 (7.0) | 1.04× | **≥ parity** |
| 512x512 | 2,126,357 (11.1) | 4,747,621 (5.0) | 2,434,624 (9.7) | 4,328,751 (5.5) | 3,324,983 (7.1) | 0.87× | **≥ parity** |
| 1024x1024 | 15,863,499 (6.6) | 19,612,861 (5.3) | 16,905,520 (6.2) | 26,884,234 (3.9) | 27,141,627 (3.9) | 0.94× | **≥ parity** |

**Rows behind FFTW, worst first:**

- complex 1,080 (2³·3³·5): 1.39×
- real 65,536 (2¹⁶): 1.36×
- complex 1,920 (2⁷·3·5): 1.32×
- complex 256 (2⁸): 1.27×
- complex 1,296 (2⁴·3⁴): 1.24×
- complex 1,000 (2³·5³): 1.23×
- real 256 (2⁸): 1.21×
- real 4,096 (2¹²): 1.20×
- real 1,000 (2³·5³): 1.19×
- real 1,080 (2³·3³·5): 1.19×
- complex 1,024 (2¹⁰): 1.18×
- real 1,920 (2⁷·3·5): 1.16×
- complex 65,536 (2¹⁶): 1.14×
- real 1,024 (2¹⁰): 1.10×

## Honest read

(Ratios below are means of two runs per host, as in the README.)

* **Where go-fft leads FFTW:**
  * Neoverse-N1 on 23 of 24 rows, composites and 2-D included (1296: 0.76×,
    2-D 128²: 0.65×); only RFFT 256 trails (1.06×);
  * Zen 3 from 4096 points (0.84×), composites 1296 and Bluestein at parity
    (1.02×), 2-D on one core (0.83–0.96×): 15 of 24 rows;
  * primes on every host: Rader's 1009 in 0.45–0.66× FFTW's time.
* **Where FFTW leads:**
  * small sizes on amd64: 256 points at 1.24× on Zen 3 and Cascade Lake, RFFT
    256 at 1.21×;
  * on Cascade Lake, mid-size real transforms and composites (1.17–1.23×), and
    2-D 1024² regressed by 6% in v0.19.0 (cause not established).
* **Against numpy and scipy:** go-fft is at or above both on every row, in every
  run, on all three hosts.
* **Against gonum:** go-fft is faster on every row measured (19/19 on each host).
* **Single precision:** float32 takes 0.46–0.83× the float64 time on Zen 3 and
  0.58–0.73× on Neoverse-N1. FFTW's single-precision library is not built on
  the hosts, so there is no float32 comparison with it.
