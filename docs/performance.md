# Performance

`go-fft` is benchmarked head-to-head against the implementations that matter:

* **FFTW** — native **fftw-3.3.11** (Homebrew arm64 bottle, NEON, linked from C) —
  the absolute C gold standard, single-threaded.
* **pocketfft** via **`numpy.fft` 2.5.0** and **`scipy.fft` 1.18.0** — the other
  hand-tuned C reference, single-threaded.
* **[`gonum.org/v1/gonum/dsp/fourier`](https://pkg.go.dev/gonum.org/v1/gonum/dsp/fourier)**
  v0.16.0 — the fairest comparison: the other pure-Go (`CGO_ENABLED=0`) FFT for Go.
  go-fft beats it at **every** size.

Numerical correctness is gated first: every go-fft transform is checked against
`numpy.fft` within `rtol=1e-9, atol=1e-7` before any timing is reported, so every
number below is for a *correct* transform.

> The full methodology, every size, GFLOP/s, the before→after tables for each
> optimization round, and the per-op action items live in
> **[BENCHMARKS.md](https://github.com/go-fft/fft/blob/main/BENCHMARKS.md)** in the
> library repo. Reproduce the whole sweep with `benchmarks/run.sh` (go-fft + gonum
> via `go test -bench`, native FFTW via a C harness, numpy/scipy via Python;
> correctness-gated; gonum is isolated in a separate `benchmarks/` module so the
> library stays dependency-free).

## Method

* **Machine**: Apple M4 Max (16-core: 12P+4E), macOS 26.5, arm64.
* **Toolchains**: go1.26.4 darwin/arm64; native FFTW fftw-3.3.11; numpy 2.5.0 /
  scipy 1.18.0 (pocketfft); pyfftw 0.15.1; gonum v0.16.0.
* **Single-threaded** for the apples-to-apples core comparison: FFTW planned with
  `threads=1`; numpy/scipy pinned to one thread (`OMP_NUM_THREADS=1` …,
  `workers=1`); Go benchmarks are single-goroutine for 1-D. The 2-D rows are the
  one place go-fft uses its multicore path (the others are all 1-core).
* **Plan reuse / steady state**: go-fft via its cached `Plan` API
  (`NewPlan(n).FFT`, `NewRealPlan(n).RFFT`); gonum via its reused object; FFTW via
  a reused `FFTW_MEASURE` plan; scipy via its internal plan cache. Each number is
  the **steady-state transform** cost, not planning.
* ns/op (lower is better); ratio = go-fft ÷ FFTW (≤ 1.05 = parity). GFLOP/s uses
  the standard `5·N·log2(N)` convention (real rfft counted at half; 2-D at total
  points).

## Algorithm

`go-fft` uses a **split-radix** kernel for powers of two (≈⅓ fewer real multiplies
than radix-4), **mixed-radix Cooley–Tukey** for the other highly-composite lengths
(radix-2/3/4/5 straight-line butterflies plus a general radix-p kernel for small
primes), **Rader's algorithm** for primes (from N=700) and **Bluestein's chirp-z**
otherwise, with all twiddle factors cached per length.

## Complex 1-D FFT (`complex128`)

ns/op (GFLOP/s). Ratio = go-fft ÷ FFTW (lower is better; ≤ 1.05 = parity).

| N | go-fft | FFTW | numpy.fft | scipy.fft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 716 (14.3) | 419 (24.4) | 2 828 | 2 302 | 3 272 | 1.71× | lags FFTW 1.71× |
| 1 024 (2¹⁰) | 3 395 (15.1) | 2 127 (24.1) | 5 678 | 4 470 | 20 066 | 1.60× | lags FFTW 1.60× |
| 4 096 (2¹²) | 16 383 (15.0) | 11 064 (22.2) | 20 359 | 16 148 | 80 164 | 1.48× | lags FFTW 1.48× |
| 65 536 (2¹⁶) | 377 784 (13.9) | 322 850 (16.2) | 554 088 | 499 500 | 1 901 025 | 1.17× | lags FFTW 1.17× |
| 1 048 576 (2²⁰) | 13 355 742 (7.9) | 8 171 156 (12.8) | 11 044 400 | 8 040 680 | 41 139 865 | 1.63× | lags FFTW 1.63× |
| 1 000 (2³·5³) | 5 850 (8.5) | 2 478 (20.1) | 6 075 | 4 961 | 16 541 | 2.36× | lags FFTW 2.36× |
| 1 080 (2³·3³·5) | 6 802 (8.0) | 2 466 (22.1) | 6 799 | 5 266 | 21 183 | 2.76× | lags FFTW 2.76× |
| 1 920 (2⁷·3·5) | 12 516 (8.4) | 4 340 (24.1) | 11 349 | 9 059 | 36 810 | 2.88× | lags FFTW 2.88× |
| 1 009 (prime) | 17 214 (2.9) | 13 668 (3.7) | 31 560 | 19 004 | 563 199 | 1.26× | lags FFTW 1.26× |
| 1 296 (2⁴·3⁴) | 9 818 (6.8) | 2 998 (22.4) | 8 063 | 6 364 | 25 328 | 3.28× | lags FFTW 3.28× |
| 10 007 (prime) | 369 217 (1.8) | 171 553 (3.9) | 321 309 | 276 031 | 75 674 509 | 2.15× | lags FFTW 2.15× |

## Real 1-D RFFT (`float64` → `complex128`, N/2+1 bins)

| N | go-fft | FFTW | numpy.rfft | scipy.rfft | gonum | go/FFTW | verdict |
|---:|---:|---:|---:|---:|---:|---:|:--|
| 256 (2⁸) | 525 (9.8) | 220 (23.3) | 2 432 | 2 422 | 1 882 | 2.39× | lags FFTW 2.39× |
| 1 024 (2¹⁰) | 2 289 (11.2) | 1 000 (25.6) | 4 180 | 3 726 | 8 998 | 2.29× | lags FFTW 2.29× |
| 4 096 (2¹²) | 10 320 (11.9) | 4 890 (25.1) | 11 742 | 10 416 | 39 264 | 2.11× | lags FFTW 2.11× |
| 65 536 (2¹⁶) | 230 167 (11.4) | 143 040 (18.3) | 230 189 | 254 964 | 896 439 | 1.61× | lags FFTW 1.61× |
| 1 048 576 (2²⁰) | 4 465 650 (11.7) | 2 958 859 (17.7) | 4 392 782 | 6 365 402 | 18 259 818 | 1.51× | lags FFTW 1.51× |
| 1 000 (2³·5³) | 3 144 (7.9) | 1 176 (21.2) | 4 289 | 3 663 | 8 604 | 2.67× | lags FFTW 2.67× |
| 1 080 (2³·3³·5) | 3 616 (7.5) | 1 131 (24.1) | 4 426 | 4 196 | 9 933 | 3.20× | lags FFTW 3.20× |
| 1 920 (2⁷·3·5) | 6 555 (8.0) | 2 189 (23.9) | 6 188 | 5 657 | 17 156 | 2.99× | lags FFTW 2.99× |

The forward r2c packs N reals into an N/2-point complex FFT and untangles. The
**pack/untangle round** made that pass allocation-free (a pooled working set,
0 alloc/op steady-state on the power-of-two path) and branchless/vectorizable,
cutting the forward mid-range time **1.5–1.65×** and the FFTW gap from **~3.3–3.9×
down to ~2.1–2.4×**. The **fused-pack round** then folded the real-pair packing
into the iterative kernel's bit-reversal gather, removing one O(N) pass and the
intermediate buffer (~1.2–1.7× faster on the pack/gather portion at N=256…4096,
bit-identical output). A full real split-radix kernel (Sorensen RVFFT) was
implemented, validated bit-exact, benchmarked, and **reverted** (a 2–10×
regression — a scalar real-split-radix abandons the SIMD, cache-friendly complex
engine the packed path reuses). The residual forward gap is FFTW's
codelet-scheduled Hermitian real kernel.

## Real inverse 1-D IRFFT (`complex128` N/2+1 bins → `float64`)

The c2r inverse now mirrors the forward packing: it reverses the untangle to
recover the N/2-point packed spectrum, runs **one N/2-point inverse complex FFT**,
and unpacks — half the arithmetic and memory traffic of the previous full
length-N conjugate-symmetric inverse. ns/op (GFLOP/s). Ratio = go-fft ÷ FFTW.

| N | go-fft | FFTW (c2r) | go/FFTW | verdict |
|---:|---:|---:|---:|:--|
| 256 (2⁸) | 797 (6.4) | 253 (20.2) | 3.15× | lags FFTW 3.15× |
| 1 024 (2¹⁰) | 3 517 (7.3) | 1 184 (21.6) | 2.97× | lags FFTW 2.97× |
| 4 096 (2¹²) | 16 749 (7.3) | 5 526 (22.2) | 3.03× | lags FFTW 3.03× |
| 65 536 (2¹⁶) | 351 731 (7.5) | 155 132 (16.9) | 2.27× | lags FFTW 2.27× |
| 1 048 576 (2²⁰) | 5 062 003 (10.4) | 3 329 422 (15.7) | 1.52× | lags FFTW 1.52× |
| 1 000 (2³·5³) | 4 659 (5.3) | 1 220 (20.4) | 3.82× | lags FFTW 3.82× |
| 1 080 (2³·3³·5) | 4 770 (5.7) | 1 341 (20.3) | 3.56× | lags FFTW 3.56× |
| 1 920 (2⁷·3·5) | 8 786 (5.5) | 2 253 (23.0) | 3.90× | lags FFTW 3.90× |

The **real-inverse round** replaced the old full-length conjugate-symmetric
inverse (≈2× the work the real symmetry requires) with this half-length packed
inverse, which **roughly halves the c2r time at every size** (1.74–2.66×, the
real-symmetry 2× realized) and cut the FFTW c2r gap from **~3.6–8.3× down to
~1.5–3.9×**. Correctness is held: `IRFFT(RFFT(x), len(x)) ≈ x` round-trips, the
DC/Nyquist bins reconstruct exactly, and odd N still routes to the full
conjugate-mirror inverse.

## 2-D complex FFT2 (`complex128`, multicore)

go-fft fans the independent 1-D row/column transforms across goroutines above a
work-size threshold; FFTW / numpy / scipy are single-threaded here. This is where
a pure-Go library decisively beats single-threaded C. ns/op (GFLOP/s).

| shape | go-fft | FFTW | numpy.fft2 | scipy.fft2 | go/FFTW | verdict |
|:--|---:|---:|---:|---:|---:|:--|
| 64×64 | 29 022 (8.5) | 9 064 (27.1) | 19 895 | 14 005 | 3.20× | lags FFTW 3.20× (below parallel threshold) |
| 128×128 | 78 984 (14.5) | 56 858 (20.2) | 76 859 | 64 707 | 1.39× | lags FFTW 1.39× |
| 256×256 | 208 437 (25.2) | 271 098 (19.3) | 359 158 | 207 601 | 0.77× | **≥ parity** |
| 512×512 | 633 920 (37.2) | 1 247 215 (18.9) | 1 697 966 | 964 884 | 0.51× | **beats all** |
| 1024×1024 | 2 784 623 (37.7) | 6 301 688 (16.6) | 9 211 551 | 5 315 022 | 0.44× | **beats all** |

## Plan / setup cost (built once, then amortized)

Steady-state transforms above reuse a plan. This is the one-time construction
cost (ns). go-fft and gonum build twiddle tables in Go; FFTW's `FFTW_MEASURE`
*times trial transforms* to pick codelets, so its planning is orders of magnitude
more expensive — the price of its steady-state speed, paid back only across many
reuses.

| N | go-fft NewPlan | gonum NewCmplxFFT | FFTW FFTW_MEASURE |
|---:|---:|---:|---:|
| 256 | 3 762 | 2 473 | 1 685 000 |
| 1 024 | 14 808 | 9 285 | 3 434 000 |
| 4 096 | 60 228 | 32 714 | 11 604 000 |
| 65 536 | 1 094 262 | 464 499 | 353 280 000 |
| 1 048 576 | 19 298 901 | 6 429 831 | 3 860 636 000 |
| 1 000 | 7 184 | 8 926 | 5 765 000 |
| 1 080 | 7 460 | 9 917 | 14 182 000 |
| 1 920 | 13 310 | 17 266 | 25 377 000 |
| 1 009 | 34 376 | 10 674 | 33 948 000 |
| 1 296 | 8 928 | 11 556 | 11 156 000 |
| 10 007 | 993 365 | 100 654 | 189 451 000 |

## Honest read

**vs gonum** (the fair pure-Go, CGO=0 peer): go-fft is faster at **every** size
measured — typically 3–5× on composite N and an order of magnitude on primes
(gonum falls back to a naive Bluestein with no Rader path; ~33× faster at
N=1009 up to ~205× faster at N=10007). go-fft is the fastest pure-Go FFT here.

**vs pocketfft** (numpy/scipy): go-fft wins the small-N rows outright (the Python
FFI tax dominates pocketfft's C kernel there) and is competitive-to-winning at
large 1-D and large 2-D; the residual losses are the mid-range single-core bands.

**vs FFTW** (the C gold standard): go-fft **wins outright** at the large 2-D
multicore shapes (256×256 at parity, 512×512 and 1024×1024 beating all), and is
within ~1.3–2.2× of FFTW on the large-prime rows (1009 at 1.26×, 10007 at
2.15×) — vs gonum's ~30× gap at the same sizes.
FFTW still leads the single-core power-of-two and smooth-composite mid-range — it
has hand-written SIMD codelets and a dedicated Hermitian real kernel. go-fft's
butterflies are now SIMD too (routed SSE2 on amd64 — **1.34–1.43× faster** than
the scalar stage there; gc-autovectorized NEON on arm64), but FFTW's codelet
generator and r2c real kernel remain ahead on one core. **The framing is honest:
mixed-radix + plans beats gonum everywhere and is comparable to FFTW at scale.**

## Lagging ops — root cause + action items

1. **Power-of-two & smooth-composite mid-range (256 … 4096, 1000/1080/1296/1920).**
   *Root cause*: scalar Go butterflies vs FFTW's hand-written NEON SIMD codelets.
   *Action taken (SIMD-butterfly round)*: the radix-2/radix-4 butterfly inner
   loops were lifted into stage-granularity kernels behind a stable seam, with
   go-asmgen generating a routed **SSE2 stage kernel on amd64** (measured
   **1.34–1.43× faster** than the scalar stage there). On arm64/s390x the Go
   assembler exposes vector float only as the FMA family (no vector
   `VFADD`/`VFSUB`), so a hand kernel only **ties** the gc autovectorizer, which
   already extracts the NEON throughput from the simple loop — so off amd64 the
   hot path stays the autovectorized Go loop. The remaining gap to FFTW on arm64
   is its real-FFT kernel and codelet scheduling (items 2–3), not raw complex-mul
   SIMD.
2. **Real mid-range (1024 … 65536).** *Root cause*: go-fft packs a real signal
   into a half-length complex FFT and untangles once; FFTW runs a dedicated real
   (r2c) kernel exploiting Hermitian symmetry at every stage (~2× less
   arithmetic). *Action taken*: the **real-inverse round** packed the c2r inverse
   symmetrically (reverse-untangle → N/2-point inverse FFT → unpack), roughly
   halving the c2r time at every size and cutting the FFTW c2r gap from
   **~3.6–8.3× to ~1.5–3.9×**; the **pack/untangle round** made the forward r2c
   pass allocation-free and branchless, cutting the forward mid-range
   **1.5–1.65×** and the FFTW gap from **~3.3–3.9× to ~2.1–2.4×**; the
   **fused-pack round** banked a further ~1.2–1.7× on the pack/gather portion.
   The deferred native real split-radix kernel (Sorensen RVFFT) was implemented,
   validated bit-exact, benchmarked, and reverted (2–10× regression). The residual
   needs a *SIMD, cache-blocked* real kernel, not a scalar real-FFT port.
3. **Large primes (1009, 10007).** *Root cause*: Rader/Bluestein convolve at
   length ≈N−1 on the recursive mixed-radix engine (~1.5× recursion tax); FFTW
   convolves at exactly N−1 with codelet-fused mixed-radix. go-fft is already
   **2–2.5× faster than gonum** here and within ~1.6× of FFTW. *Action*: an
   iterative mixed-radix engine for the smooth convolution lengths (a substantial
   new engine, lower priority than items 1–2).
4. **Small 2-D (64×64, 128×128).** *Root cause*: below the parallel threshold, so
   they run the serial per-line path while FFTW uses fused 2-D codelets. *Action*:
   SIMD (item 1) lifts the per-line 1-D cost; the multicore path already makes
   go-fft win at 512×512 and 1024×1024.

> The unifying lever is **item 1 (go-asmgen SIMD butterflies)**: it attacks the
> pow2/composite mid-range, the per-line cost inside 2-D, and feeds the real and
> prime paths (which both convolve via complex FFTs).
