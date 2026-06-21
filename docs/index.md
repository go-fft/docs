# go-fft documentation

**A pure-Go (no cgo) FFT library** — the `numpy.fft` / `scipy.fft` equivalent for
Go. It computes the discrete Fourier transform of complex and real signals of
**any length**, with no dependency on the native FFTW3 C library.

Ruby has no cgo-free FFT (every option wraps FFTW3); `gonum/dsp/fourier` is pure
Go but its optimized assembly is amd64-only. go-fft is a fully portable scalar
core with **go-asmgen SIMD kernels on four of Go's six 64-bit targets**, grown
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

Power-of-two lengths use radix-2 Cooley–Tukey; arbitrary lengths use Bluestein's
chirp-z transform. Normalization, bin layout and frequency conventions follow
`numpy.fft`.

## SIMD & architectures

The pointwise complex-multiply kernel is bit-identical across a portable scalar
path and **go-asmgen SIMD** on four targets — amd64 (SSE2), arm64 (NEON),
riscv64 (RVV, hardware-validated) and s390x (vector facility, big-endian). The
remaining two 64-bit targets, loong64 and ppc64le, run the validated scalar path
(the Go assembler lacks the vector-double ops they would need).

## Where to go next

- [Roadmap (phases)](roadmap.md) — the phased plan and what ships today.
- [Performance](performance.md) — honest benchmarks versus pocketfft / FFTW.

Source: [github.com/go-fft/fft](https://github.com/go-fft/fft) · the transform is
also exposed to Ruby through [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)'s
`FFT` module.
