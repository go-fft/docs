---
title: "go-fft docs"
description: "A pure-Go (CGO=0) FFT library — the numpy.fft / scipy.fft equivalent for Go."
---
**A pure-Go (no cgo) FFT library** — the `numpy.fft` / `scipy.fft` equivalent for
Go. It computes the discrete Fourier transform of complex and real signals of
**any length**, with no dependency on the native FFTW3 C library.

Ruby has no cgo-free FFT (every option wraps FFTW3); `gonum/dsp/fourier` is pure
Go but its optimized assembly is amd64-only. go-fft is a fully portable scalar
core with **go-asmgen SIMD kernels** — a bit-identical complex multiply on four
of Go's six 64-bit targets, and AVX2 butterfly stage kernels on amd64 — grown
test-first with **100% coverage** and differentially checked against `numpy.fft`.

```go
import "github.com/go-fft/fft"

x := []complex128{1, 2, 3, 4}
X := fft.FFT(x)          // forward transform
y := fft.IFFT(X)         // round-trips back to x
```

## API surface

| Area | Functions |
| --- | --- |
| Complex 1-D | `FFT`, `IFFT` |
| Real 1-D | `RFFT`, `IRFFT` |
| Multi-dimensional | `FFTN`, `IFFTN`, `FFT2`, `IFFT2`, `RFFT2`, `IRFFT2` |
| Frequency bins | `FFTFreq`, `RFFTFreq` |
| Windows | `Hann`, `Hamming`, `Blackman`, `BlackmanHarris`, `Bartlett` |
| Spectral | `PSD`, `Spectrogram` |

Powers of two use a **split-radix** kernel; other highly-composite lengths use
**mixed-radix Cooley–Tukey**; primes use **Rader's algorithm** (from N=700) and
**Bluestein's chirp-z** otherwise, with all twiddle factors cached per length.
Normalization, bin layout and frequency conventions follow `numpy.fft`.

## SIMD & architectures

There are two kinds of kernel, and they are worth separating because only one of
them is on the hot path.

**Pointwise complex multiply.** Bit-identical across a portable scalar path and
**go-asmgen SIMD** on four targets — amd64 (SSE2), arm64 (NEON), riscv64 (RVV,
hardware-validated) and s390x (vector facility, big-endian). The remaining two
64-bit targets, loong64 and ppc64le, run the validated scalar path, because the
Go assembler lacks the vector-double ops they would need. These are built and
validated but *not* routed off amd64: measured, they only tie or lose to the gc
autovectorizer there.

**Butterfly stage kernels.** A whole radix-4 or radix-2 pass — the loop over
groups and the loop over positions both — inside one call. amd64 only: SSE2
across the baseline, and **AVX2** where the CPU *and* the operating system both
support it, chosen at run time. These are routed.

| real input | SSE2 | AVX2 | time |
| ---: | ---: | ---: | ---: |
| 64 | 135.5 ns | 95.99 ns | −29.2% |
| 256 | 562.3 ns | 402.0 ns | −28.5% |
| 512 | 1,145 ns | 919.2 ns | −19.7% |
| 1,024 | 2,521 ns | 1,734 ns | −31.2% |
| 2,048 | 5,230 ns | 4,256 ns | −18.6% |
| 4,096 | 11,349 ns | 7,398 ns | −34.8% |
| 16,384 | 50,527 ns | 32,676 ns | −35.3% |

Intel Core i5-14600K, `GOAMD64=v1`, one thread, process affinity to CPU 0,
median of five 400 ms repetitions on a reusable `RealPlan`; 0 B/op and
0 allocs/op on every row. The raw runs are committed under
`benchmarks/results/amd64-avx2-20260922/` in the library repository.

⛔ The arithmetic order is preserved exactly — separately rounded multiply, add
and subtract, no FMA and no reassociation — so an AVX2 result is **bit-identical**
to the SSE2 one and to the scalar oracle. The tests assert it by running every
shape down both paths and comparing `math.Float64bits`. A faster transform that
answers differently is not the same transform.

## Where to go next

- [Roadmap (phases)](roadmap.md) — the phased plan and what ships today.
- [Performance](performance.md) — honest benchmarks versus pocketfft / FFTW.

Source: [github.com/go-fft/fft](https://github.com/go-fft/fft) · the transform is
also exposed to Ruby through [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)'s
`FFT` module.
