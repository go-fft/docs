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

The numbers below were measured on 2026-10-04 on three GCC Compile Farm hosts:
**go-fft v0.1.5** on Zen 3 and Neoverse-N1, **v0.1.7** on Cascade Lake. None of
those machines runs different code under the current release. The raw runs are in
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
* **Powers of two on riscv64, loong64, s390x, and amd64 without AVX2**: an
  iterative, cache-blocked radix-4 kernel.
* On arm64 and the other non-amd64 targets the passes are Go code, compiled to
  scalar instructions. gc does not vectorize; it does fuse multiply-adds.

## AMD EPYC 7773X (Zen 3), AVX2

cfarm420, 128 threads, load average ≈ 3–4; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/FMA.

- **vs FFTW (native C, gold standard)**: 6/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 403 (25.4) | 278 (36.9) | 6,784 (1.5) | 5,439 (1.9) | 12,300 (0.8) | 1.45× | lags FFTW 1.45× |
| 1,024 (2¹⁰) | 2,053 (24.9) | 1,525 (33.6) | 13,203 (3.9) | 9,709 (5.3) | 61,064 (0.8) | 1.35× | lags FFTW 1.35× |
| 4,096 (2¹²) | 12,967 (19.0) | 10,519 (23.4) | 48,063 (5.1) | 32,864 (7.5) | 286,577 (0.9) | 1.23× | lags FFTW 1.23× |
| 65,536 (2¹⁶) | 287,755 (18.2) | 295,993 (17.7) | 1,950,255 (2.7) | 907,537 (5.8) | 6,261,047 (0.8) | 0.97× | **≥ parity** |
| 1,048,576 (2²⁰) | 7,015,491 (14.9) | 11,150,846 (9.4) | 23,327,232 (4.5) | 13,742,973 (7.6) | 131,640,385 (0.8) | 0.63× | **≥ parity** |
| 1,000 (2³·5³) | 2,606 (19.1) | 2,048 (24.3) | 13,867 (3.6) | 10,114 (4.9) | 68,944 (0.7) | 1.27× | lags FFTW 1.27× |
| 1,080 (2³·3³·5) | 3,044 (17.9) | 2,305 (23.6) | 14,331 (3.8) | 11,853 (4.6) | 69,808 (0.8) | 1.32× | lags FFTW 1.32× |
| 1,920 (2⁷·3·5) | 5,093 (20.6) | 3,934 (26.6) | 21,847 (4.8) | 16,313 (6.4) | 129,547 (0.8) | 1.29× | lags FFTW 1.29× |
| 1,009 (prime) | 13,257 (3.8) | 20,771 (2.4) | 60,288 (0.8) | 38,431 (1.3) | 1,549,023 (0.0) | 0.64× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 4,069 (16.5) | 2,997 (22.4) | 16,314 (4.1) | 12,375 (5.4) | 83,992 (0.8) | 1.36× | lags FFTW 1.36× |
| 10,007 (prime) | 253,076 (2.6) | 203,981 (3.3) | 649,788 (1.0) | 465,714 (1.4) | 155,005,073 (0.0) | 1.24× | lags FFTW 1.24× |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 322 (15.9) | 227 (22.6) | 6,264 (0.8) | 5,461 (0.9) | 5,211 (1.0) | 1.42× | lags FFTW 1.42× |
| 1,024 (2¹⁰) | 1,276 (20.1) | 1,004 (25.5) | 9,596 (2.7) | 8,831 (2.9) | 24,897 (1.0) | 1.27× | lags FFTW 1.27× |
| 4,096 (2¹²) | 6,965 (17.6) | 5,396 (22.8) | 25,097 (4.9) | 22,256 (5.5) | 109,442 (1.1) | 1.29× | lags FFTW 1.29× |
| 65,536 (2¹⁶) | 163,344 (16.0) | 154,110 (17.0) | 408,729 (6.4) | 497,040 (5.3) | 2,308,755 (1.1) | 1.06× | lags FFTW 1.06× |
| 1,048,576 (2²⁰) | 3,651,718 (14.4) | 3,679,195 (14.2) | 8,396,119 (6.2) | 10,128,341 (5.2) | 45,926,821 (1.1) | 0.99× | **≥ parity** |
| 1,000 (2³·5³) | 1,594 (15.6) | 1,253 (19.9) | 10,347 (2.4) | 8,598 (2.9) | 27,577 (0.9) | 1.27× | lags FFTW 1.27× |
| 1,080 (2³·3³·5) | 1,827 (14.9) | 1,326 (20.5) | 10,865 (2.5) | 9,005 (3.0) | 28,149 (1.0) | 1.38× | lags FFTW 1.38× |
| 1,920 (2⁷·3·5) | 2,992 (17.5) | 2,398 (21.8) | 14,273 (3.7) | 12,149 (4.3) | 51,694 (1.0) | 1.25× | lags FFTW 1.25× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 362 (14.2) | 348 (14.7) | 1.04× | **≥ parity** |
| 1,024 (2¹⁰) | 1,281 (20.0) | 1,176 (21.8) | 1.09× | lags FFTW 1.09× |
| 4,096 (2¹²) | 7,191 (17.1) | 5,810 (21.1) | 1.24× | lags FFTW 1.24× |
| 65,536 (2¹⁶) | 167,713 (15.6) | 166,455 (15.7) | 1.01× | **≥ parity** |
| 1,048,576 (2²⁰) | 3,417,614 (15.3) | 4,105,172 (12.8) | 0.83× | **≥ parity** |
| 1,000 (2³·5³) | 1,618 (15.4) | 1,491 (16.7) | 1.09× | lags FFTW 1.09× |
| 1,080 (2³·3³·5) | 1,828 (14.9) | 1,530 (17.8) | 1.19× | lags FFTW 1.19× |
| 1,920 (2⁷·3·5) | 3,058 (17.1) | 2,712 (19.3) | 1.13× | lags FFTW 1.13× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 23,520 (10.4) | 39,639 (6.2) | 10,702 (23.0) | 50,854 (4.8) | 34,042 (7.2) | 2.20× | lags FFTW 2.20× |
| 128x128 | 129,283 (8.9) | 207,589 (5.5) | 66,104 (17.4) | 143,128 (8.0) | 127,998 (9.0) | 1.96× | lags FFTW 1.96× |
| 256x256 | 328,888 (15.9) | 571,409 (9.2) | 283,132 (18.5) | 649,841 (8.1) | 450,754 (11.6) | 1.16× | lags FFTW 1.16× |
| 512x512 | 1,094,156 (21.6) | 1,591,244 (14.8) | 1,296,141 (18.2) | 2,899,721 (8.1) | 1,934,815 (12.2) | 0.84× | **≥ parity** |
| 1024x1024 | 2,347,602 (44.7) | 5,534,835 (18.9) | 6,554,636 (16.0) | 13,416,170 (7.8) | 9,461,339 (11.1) | 0.36× | **≥ parity** |

**Rows behind FFTW, worst first:**

- 2-D 64x64: 2.20×
- 2-D 128x128: 1.96×
- complex 256 (2⁸): 1.45×
- real 256 (2⁸): 1.42×
- real 1,080 (2³·3³·5): 1.38×
- complex 1,296 (2⁴·3⁴): 1.36×
- complex 1,024 (2¹⁰): 1.35×
- complex 1,080 (2³·3³·5): 1.32×
- complex 1,920 (2⁷·3·5): 1.29×
- real 4,096 (2¹²): 1.29×
- complex 1,000 (2³·5³): 1.27×
- real 1,000 (2³·5³): 1.27×
- real 1,024 (2¹⁰): 1.27×
- real 1,920 (2⁷·3·5): 1.25×
- complex 10,007 (prime): 1.24×
- complex 4,096 (2¹²): 1.23×
- 2-D 256x256: 1.16×
- real 65,536 (2¹⁶): 1.06×

## Neoverse-N1 (arm64)

cfarm424, 64 cores, load average < 1; FFTW 3.3.10 built from source with NEON.

- **vs FFTW (native C, gold standard)**: 7/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 23/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 20/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,897 (5.4) | 1,156 (8.9) | 9,535 (1.1) | 7,770 (1.3) | 10,786 (0.9) | 1.64× | lags FFTW 1.64× |
| 1,024 (2¹⁰) | 9,009 (5.7) | 6,317 (8.1) | 18,723 (2.7) | 14,597 (3.5) | 53,320 (1.0) | 1.43× | lags FFTW 1.43× |
| 4,096 (2¹²) | 47,150 (5.2) | 40,604 (6.1) | 59,349 (4.1) | 48,159 (5.1) | 281,470 (0.9) | 1.16× | lags FFTW 1.16× |
| 65,536 (2¹⁶) | 1,126,744 (4.7) | 1,324,500 (4.0) | 2,256,821 (2.3) | 1,482,226 (3.5) | 6,469,940 (0.8) | 0.85× | **≥ parity** |
| 1,048,576 (2²⁰) | 26,566,310 (3.9) | 44,987,190 (2.3) | 44,213,908 (2.4) | 32,179,429 (3.3) | 141,620,920 (0.7) | 0.59× | **≥ parity** |
| 1,000 (2³·5³) | 11,075 (4.5) | 7,152 (7.0) | 19,142 (2.6) | 15,145 (3.3) | 55,645 (0.9) | 1.55× | lags FFTW 1.55× |
| 1,080 (2³·3³·5) | 12,755 (4.3) | 7,674 (7.1) | 22,138 (2.5) | 16,604 (3.3) | 63,788 (0.9) | 1.66× | lags FFTW 1.66× |
| 1,920 (2⁷·3·5) | 20,799 (5.0) | 13,620 (7.7) | 33,041 (3.2) | 25,894 (4.0) | 116,176 (0.9) | 1.53× | lags FFTW 1.53× |
| 1,009 (prime) | 27,628 (1.8) | 48,453 (1.0) | 96,470 (0.5) | 57,376 (0.9) | 2,344,866 (0.0) | 0.57× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 15,813 (4.2) | 11,367 (5.9) | 24,794 (2.7) | 19,689 (3.4) | 80,856 (0.8) | 1.39× | lags FFTW 1.39× |
| 10,007 (prime) | 815,537 (0.8) | 609,824 (1.1) | 1,044,754 (0.6) | 768,517 (0.9) | 223,174,406 (0.0) | 1.34× | lags FFTW 1.34× |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,239 (4.1) | 723 (7.1) | 8,817 (0.6) | 7,741 (0.7) | 6,029 (0.8) | 1.71× | lags FFTW 1.71× |
| 1,024 (2¹⁰) | 5,439 (4.7) | 4,172 (6.1) | 14,286 (1.8) | 12,407 (2.1) | 30,100 (0.9) | 1.30× | lags FFTW 1.30× |
| 4,096 (2¹²) | 24,895 (4.9) | 19,720 (6.2) | 37,944 (3.2) | 33,753 (3.6) | 133,431 (0.9) | 1.26× | lags FFTW 1.26× |
| 65,536 (2¹⁶) | 646,781 (4.1) | 539,920 (4.9) | 738,289 (3.6) | 745,045 (3.5) | 3,231,981 (0.8) | 1.20× | lags FFTW 1.20× |
| 1,048,576 (2²⁰) | 14,361,911 (3.7) | 19,042,966 (2.8) | 20,251,994 (2.6) | 18,649,949 (2.8) | 75,630,164 (0.7) | 0.75× | **≥ parity** |
| 1,000 (2³·5³) | 6,301 (4.0) | 3,947 (6.3) | 15,441 (1.6) | 12,658 (2.0) | 28,621 (0.9) | 1.60× | lags FFTW 1.60× |
| 1,080 (2³·3³·5) | 7,243 (3.8) | 4,206 (6.5) | 16,152 (1.7) | 14,197 (1.9) | 33,385 (0.8) | 1.72× | lags FFTW 1.72× |
| 1,920 (2⁷·3·5) | 12,154 (4.3) | 7,323 (7.1) | 22,217 (2.4) | 18,757 (2.8) | 60,948 (0.9) | 1.66× | lags FFTW 1.66× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 1,324 (3.9) | 812 (6.3) | 1.63× | lags FFTW 1.63× |
| 1,024 (2¹⁰) | 5,495 (4.7) | 4,486 (5.7) | 1.22× | lags FFTW 1.22× |
| 4,096 (2¹²) | 27,301 (4.5) | 21,175 (5.8) | 1.29× | lags FFTW 1.29× |
| 65,536 (2¹⁶) | 634,073 (4.1) | 651,122 (4.0) | 0.97× | **≥ parity** |
| 1,048,576 (2²⁰) | 14,554,225 (3.6) | 21,944,856 (2.4) | 0.66× | **≥ parity** |
| 1,000 (2³·5³) | 6,774 (3.7) | 4,296 (5.8) | 1.58× | lags FFTW 1.58× |
| 1,080 (2³·3³·5) | 7,564 (3.6) | 4,421 (6.2) | 1.71× | lags FFTW 1.71× |
| 1,920 (2⁷·3·5) | 13,396 (3.9) | 7,824 (6.7) | 1.71× | lags FFTW 1.71× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 63,408 (3.9) | 172,237 (1.4) | 30,873 (8.0) | 70,063 (3.5) | 49,955 (4.9) | 2.05× | lags FFTW 2.05× |
| 128x128 | 478,244 (2.4) | 546,855 (2.1) | 192,657 (6.0) | 250,649 (4.6) | 195,179 (5.9) | 2.48× | lags FFTW 2.48× |
| 256x256 | 924,311 (5.7) | 1,255,620 (4.2) | 1,309,105 (4.0) | 1,054,372 (5.0) | 863,151 (6.1) | 0.71× | **≥ parity** |
| 512x512 | 2,362,428 (10.0) | 2,929,776 (8.1) | 7,032,992 (3.4) | 5,538,421 (4.3) | 3,988,878 (5.9) | 0.34× | **≥ parity** |
| 1024x1024 | 6,309,178 (16.6) | 7,836,390 (13.4) | 43,933,250 (2.4) | 25,754,452 (4.1) | 21,814,100 (4.8) | 0.14× | **≥ parity** |

**Rows behind FFTW, worst first:**

- 2-D 128x128: 2.48×
- 2-D 64x64: 2.05×
- real 1,080 (2³·3³·5): 1.72×
- real 256 (2⁸): 1.71×
- complex 1,080 (2³·3³·5): 1.66×
- real 1,920 (2⁷·3·5): 1.66×
- complex 256 (2⁸): 1.64×
- real 1,000 (2³·5³): 1.60×
- complex 1,000 (2³·5³): 1.55×
- complex 1,920 (2⁷·3·5): 1.53×
- complex 1,024 (2¹⁰): 1.43×
- complex 1,296 (2⁴·3⁴): 1.39×
- complex 10,007 (prime): 1.34×
- real 1,024 (2¹⁰): 1.30×
- real 4,096 (2¹²): 1.26×
- real 65,536 (2¹⁶): 1.20×
- complex 4,096 (2¹²): 1.16×

## Intel Xeon (Cascade Lake), AVX-512

cfarm151, 8 vCPUs, load average < 1; FFTW 3.3.10 built from source with SSE2/AVX/AVX2/AVX-512/FMA. go-fft v0.1.7, whose code on this machine is that of v0.1.5 and v0.1.8.

- **vs FFTW (native C, gold standard)**: 5/24 ops at-or-above parity.
- **vs numpy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs scipy.fft (pocketfft)**: 24/24 ops at-or-above parity.
- **vs gonum (pure-Go peer)**: 19/19 ops at-or-above parity.

### Complex 1-D FFT

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 518 (19.8) | 364 (28.1) | 8,971 (1.1) | 6,527 (1.6) | 9,095 (1.1) | 1.42× | lags FFTW 1.42× |
| 1,024 (2¹⁰) | 2,777 (18.4) | 2,078 (24.6) | 16,468 (3.1) | 11,988 (4.3) | 46,636 (1.1) | 1.34× | lags FFTW 1.34× |
| 4,096 (2¹²) | 13,195 (18.6) | 11,925 (20.6) | 55,881 (4.4) | 43,591 (5.6) | 223,296 (1.1) | 1.11× | lags FFTW 1.11× |
| 65,536 (2¹⁶) | 630,588 (8.3) | 402,862 (13.0) | 2,010,010 (2.6) | 1,379,240 (3.8) | 5,532,786 (0.9) | 1.57× | lags FFTW 1.57× |
| 1,048,576 (2²⁰) | 30,219,146 (3.5) | 26,034,898 (4.0) | 52,323,462 (2.0) | 40,861,867 (2.6) | 137,193,076 (0.8) | 1.16× | lags FFTW 1.16× |
| 1,000 (2³·5³) | 3,940 (12.6) | 2,767 (18.0) | 17,142 (2.9) | 14,140 (3.5) | 46,444 (1.1) | 1.42× | lags FFTW 1.42× |
| 1,080 (2³·3³·5) | 4,681 (11.6) | 3,001 (18.1) | 20,430 (2.7) | 13,235 (4.1) | 56,431 (1.0) | 1.56× | lags FFTW 1.56× |
| 1,920 (2⁷·3·5) | 7,900 (13.3) | 4,918 (21.3) | 30,208 (3.5) | 21,510 (4.9) | 97,921 (1.1) | 1.61× | lags FFTW 1.61× |
| 1,009 (prime) | 19,932 (2.5) | 30,736 (1.6) | 79,351 (0.6) | 55,867 (0.9) | 1,955,698 (0.0) | 0.65× | **≥ parity** |
| 1,296 (2⁴·3⁴) | 6,369 (10.5) | 3,628 (18.5) | 20,564 (3.3) | 15,806 (4.2) | 66,231 (1.0) | 1.76× | lags FFTW 1.76× |
| 10,007 (prime) | 346,752 (1.9) | 359,673 (1.8) | 852,248 (0.8) | 728,135 (0.9) | 192,346,904 (0.0) | 0.96× | **≥ parity** |

### Real 1-D RFFT

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 494 (10.4) | 370 (13.9) | 6,517 (0.8) | 6,643 (0.8) | 4,607 (1.1) | 1.34× | lags FFTW 1.34× |
| 1,024 (2¹⁰) | 1,753 (14.6) | 1,506 (17.0) | 12,139 (2.1) | 10,123 (2.5) | 22,355 (1.1) | 1.16× | lags FFTW 1.16× |
| 4,096 (2¹²) | 9,144 (13.4) | 7,354 (16.7) | 32,419 (3.8) | 31,497 (3.9) | 97,673 (1.3) | 1.24× | lags FFTW 1.24× |
| 65,536 (2¹⁶) | 245,318 (10.7) | 194,778 (13.5) | 616,113 (4.3) | 688,198 (3.8) | 2,192,859 (1.2) | 1.26× | lags FFTW 1.26× |
| 1,048,576 (2²⁰) | 16,068,287 (3.3) | 10,403,525 (5.0) | 20,155,236 (2.6) | 21,152,562 (2.5) | 66,732,806 (0.8) | 1.54× | lags FFTW 1.54× |
| 1,000 (2³·5³) | 2,464 (10.1) | 2,016 (12.4) | 13,082 (1.9) | 11,991 (2.1) | 22,643 (1.1) | 1.22× | lags FFTW 1.22× |
| 1,080 (2³·3³·5) | 2,846 (9.6) | 2,156 (12.6) | 13,432 (2.0) | 11,119 (2.4) | 24,921 (1.1) | 1.32× | lags FFTW 1.32× |
| 1,920 (2⁷·3·5) | 4,477 (11.7) | 3,456 (15.1) | 18,370 (2.8) | 15,260 (3.4) | 45,519 (1.2) | 1.30× | lags FFTW 1.30× |

### Real inverse 1-D IRFFT

| N | go-fft | FFTW | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 520 (9.8) | 529 (9.7) | 0.98× | **≥ parity** |
| 1,024 (2¹⁰) | 1,813 (14.1) | 1,976 (13.0) | 0.92× | **≥ parity** |
| 4,096 (2¹²) | 9,263 (13.3) | 8,378 (14.7) | 1.11× | lags FFTW 1.11× |
| 65,536 (2¹⁶) | 252,618 (10.4) | 215,436 (12.2) | 1.17× | lags FFTW 1.17× |
| 1,048,576 (2²⁰) | 15,050,916 (3.5) | 9,680,486 (5.4) | 1.55× | lags FFTW 1.55× |
| 1,000 (2³·5³) | 2,421 (10.3) | 2,279 (10.9) | 1.06× | lags FFTW 1.06× |
| 1,080 (2³·3³·5) | 2,801 (9.7) | 2,329 (11.7) | 1.20× | lags FFTW 1.20× |
| 1,920 (2⁷·3·5) | 4,535 (11.5) | 3,960 (13.2) | 1.15× | lags FFTW 1.15× |

### 2-D complex FFT2

| shape | go-fft | FFT2 | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|---:|:--|
| 64x64 | 34,023 (7.2) | 48,515 (5.1) | 15,083 (16.3) | 57,151 (4.3) | 44,842 (5.5) | 2.26× | lags FFTW 2.26× |
| 128x128 | 140,357 (8.2) | 191,900 (6.0) | 84,074 (13.6) | 211,819 (5.4) | 180,727 (6.3) | 1.67× | lags FFTW 1.67× |
| 256x256 | 498,210 (10.5) | 734,341 (7.1) | 480,842 (10.9) | 945,551 (5.5) | 770,487 (6.8) | 1.04× | **≥ parity** |
| 512x512 | 1,855,475 (12.7) | 2,943,242 (8.0) | 2,627,125 (9.0) | 6,672,469 (3.5) | 4,669,821 (5.1) | 0.71× | **≥ parity** |
| 1024x1024 | 8,430,820 (12.4) | 11,388,587 (9.2) | 25,825,030 (4.1) | 30,855,494 (3.4) | 27,295,744 (3.8) | 0.33× | **≥ parity** |

**Rows behind FFTW, worst first:**

- 2-D 64x64: 2.26×
- complex 1,296 (2⁴·3⁴): 1.76×
- 2-D 128x128: 1.67×
- complex 1,920 (2⁷·3·5): 1.61×
- complex 65,536 (2¹⁶): 1.57×
- complex 1,080 (2³·3³·5): 1.56×
- real 1,048,576 (2²⁰): 1.54×
- complex 1,000 (2³·5³): 1.42×
- complex 256 (2⁸): 1.42×
- complex 1,024 (2¹⁰): 1.34×
- real 256 (2⁸): 1.34×
- real 1,080 (2³·3³·5): 1.32×
- real 1,920 (2⁷·3·5): 1.30×
- real 65,536 (2¹⁶): 1.26×
- real 4,096 (2¹²): 1.24×
- real 1,000 (2³·5³): 1.22×
- real 1,024 (2¹⁰): 1.16×
- complex 1,048,576 (2²⁰): 1.16×
- complex 4,096 (2¹²): 1.11×

## Honest read

* **Where go-fft leads FFTW:**
  * the largest 1-D transforms on Zen 3 and Neoverse-N1: complex 2^20 runs in
    0.59–0.63× FFTW's time, and 65536 in 0.85–0.97× (on Cascade Lake they trail,
    1.16–1.57×: throughput halves past 4096 points there, for a reason not yet
    established — see BENCHMARKS.md, Round 13);
  * primes whose N−1 is smooth (Rader);
  * the larger 2-D shapes, which go-fft spreads across cores and FFTW runs on one.
* **Where FFTW leads:**
  * the small and mid sizes, 256 to 4096 points, by up to 1.7×;
  * small 2-D shapes (64×64, 128×128), by 2.0–2.5×;
  * composites on arm64, where go-fft's passes are scalar Go and FFTW's are NEON.
* **Against numpy and scipy:** go-fft is at or above both on every row on Zen 3
  and on Cascade Lake.
* **Against gonum:** go-fft is faster at every size, 5–6× on Neoverse-N1,
  4.5–18× on Cascade Lake and 19–31× on Zen 3 for powers of two and composites,
  and 85–612× on primes.
