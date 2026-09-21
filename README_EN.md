# GoFEC

[![Go Reference](https://pkg.go.dev/badge/github.com/nyarime/gofec.svg)](https://pkg.go.dev/github.com/nyarime/gofec)

Pure Go Forward Error Correction (FEC) erasure coding library with SIMD hardware acceleration. No CGO.

GoFEC provides two erasure code implementations — **RaptorQ fountain code** and **LDPC** — with AVX2/NEON assembly-accelerated GF(256) arithmetic.

Package home: https://github.com/Nyarime/GoFEC

Godoc: https://pkg.go.dev/github.com/nyarime/gofec

[中文](README.md)

## Installation

```bash
go get github.com/nyarime/gofec
```

Go Modules recommended.

## Features

| Codec | Description | Status |
|-------|-------------|--------|
| RaptorQ | RFC 6330 fountain code, unlimited repair symbols | ✅ |
| LDPC | PEG matrix + Belief Propagation decoding | ✅ |
| GF(256) | Log/Exp tables + SIMD vectorization | ✅ |
| XOR | AVX2/NEON hardware acceleration | ✅ |

### SIMD Acceleration Stack

GoFEC auto-detects CPU features at runtime and selects the optimal instruction set:

| Platform | XOR | GF(256) MulAdd |
|----------|-----|----------------|
| amd64 AVX512+GFNI | 32B/op | 64B/op (VGF2P8AFFINEQB) |
| amd64 AVX2 | 32B/op (VPXOR) | 32B/op (VPSHUFB) |
| amd64 SSE2 | 16B/op | Scalar |
| arm64 NEON | 16B/op (VEOR) | 16B/op (VTBL) |
| Generic | 8B/op (uint64) | Scalar Log/Exp |

## Usage

### RaptorQ Fountain Code

RaptorQ is a rateless erasure code — it generates **unlimited** repair symbols from source data. The receiver can recover the original data from **any combination** of symbols, as long as it receives slightly more than the number of source symbols.

This property makes it ideal for scenarios with unpredictable packet loss (mobile networks, satellite links).

```go
import "github.com/nyarime/gofec/raptorq"

// Create codec: 16 source symbols, 128 bytes each
codec := raptorq.New(16, 128)

// Encode: source data → source symbols + 8 repair symbols
symbols := codec.Encode(data, 8)

// Simulate network: some symbols are lost...
// As long as ≥16 symbols received (any combination), recovery is possible

// Decode
decoded, err := codec.Decode(receivedSymbols, len(data))
if err != nil {
    log.Fatal("not enough repair symbols:", err)
}
```

**Parameter selection:**
- `K` (source symbols): Higher = better efficiency, higher latency. Recommended: 8–64.
- `T` (symbol size): Usually tied to network MTU. Recommended: 64–1024 bytes.
- Repair symbols: More = better fault tolerance. Suggest at least `K * 0.5`.

### LDPC

LDPC is a fixed-rate block code with extremely fast encode/decode, ideal for low-latency scenarios with known loss rates.

```go
import "github.com/nyarime/gofec/ldpc"

// Create codec: 10 data shards, 4 parity shards, PEG density 0.3
codec := ldpc.New(10, 4, 0.3)

// Encode
encoded := codec.Encode(dataShards)

// Recover after partial loss
err := codec.Decode(partialShards)
```

### RaptorQ vs LDPC: When to Use Which?

| Scenario | Recommendation | Reason |
|----------|---------------|--------|
| Unknown loss rate | RaptorQ | Fountain code adapts to any loss |
| Real-time video/gaming | LDPC | BP decode latency is minimal |
| File transfer/distribution | RaptorQ | No need to predict loss |
| Satellite/high-loss links | RaptorQ | Just send more repair symbols |
| Low loss + low latency | LDPC | Fixed small overhead, fast decode |

**TL;DR:** LDPC = faster & leaner (known loss). RaptorQ = more flexible & robust (unknown loss).

## Performance

Tested on Intel Broadwell, Go 1.25, Linux amd64.

### RaptorQ Encoding

| Data Size | Latency | Throughput | Allocations |
|-----------|---------|------------|-------------|
| 512B | 2.4μs | 213 MB/s | 14 allocs |
| 2KB | 5.0μs | 400 MB/s | 27 allocs |
| 8KB | 13.5μs | 593 MB/s | 53 allocs |
| 32KB | 41μs | 780 MB/s | 103 allocs |

### RaptorQ Loss Recovery

| Lost Blocks | 8 source + N*3 repair | Status |
|-------------|----------------------|--------|
| 1–6 | ✅ | Full recovery |

### LDPC

| Operation | Latency |
|-----------|---------|
| Encode | 18μs/op |
| Decode | 20μs/op |

### SIMD Benchmarks

| Operation | Throughput | Zero Alloc |
|-----------|-----------|------------|
| XOR (AVX2, 4KB) | 31.6 GB/s | ✅ |
| GF(256) MulAdd (AVX2, 1KB) | 15.0 GB/s | ✅ |
| GF(256) Mul (scalar) | 607M ops/s | ✅ |

## Architecture

```
GoFEC/
├── codec.go                  # Unified Codec interface
├── raptorq/                  # RaptorQ fountain code (RFC 6330)
│   ├── raptorq.go            # Encoder/Decoder (LT + Gaussian elimination)
│   └── xor.go                # SIMD XOR bridge
├── ldpc/                     # LDPC codes
│   ├── ldpc.go               # Belief Propagation decoder
│   ├── peg.go                # PEG matrix construction
│   └── xor.go                # SIMD XOR bridge
└── internal/
    ├── xor/                  # Hardware-accelerated XOR
    │   ├── xor_amd64.s       # AVX2 assembly
    │   ├── xor_arm64.s       # NEON assembly
    │   └── xor_generic.go    # Generic fallback
    └── gf256/                # GF(2^8) finite field
        ├── gf.go             # Log/Exp tables
        ├── tables.go         # Precomputed split tables (8KB)
        ├── mulAdd_amd64.s    # AVX2 VPSHUFB
        ├── mulAdd_gfni_amd64.s  # AVX512 GFNI
        ├── mulAdd_arm64.s    # NEON VTBL
        ├── cpu_amd64.go      # CPU feature detection
        └── cpu_arm64.go      # CPU feature detection
```

## Roadmap

- [x] v1.0 — RaptorQ + LDPC + AVX2 + NEON
- [ ] v1.x — AVX512 GFNI runtime acceleration (3x GF speedup)
- [ ] v2.0 — GF(2^16) + Leopard-RS (65536 shards)

## Used By

GoFEC is a **standalone FEC library** in the Nyarime stack — NYA pulls it via
`go get` instead of baking erasure coding into the archive format. The same
pattern is planned for standalone codec repos (see ecosystem below).

| Project | Role |
| --- | --- |
| [NYA](https://github.com/nyarime/nya) | Archive + FEC + distribution (`nya augment`, embedded FEC; Leopard-RS / Hybrid / RaptorQ via GoFEC) |
| [NRUP](https://github.com/Nyarime/NRUP) | Nyarime Reliable UDP Protocol (transport FEC) |

### Nyarime library ecosystem (direction)

| Library | Status | Notes |
| --- | --- | --- |
| **GoFEC** (this repo) | ✅ standalone | Pure Go FEC, SIMD, no cgo |
| **nyazstd** | 📋 planned split | In-house RFC 8878 zstd encoder/decoder; today inside `nya`, target `go get` like GoFEC |
| **nyalzma2** | ⏳ polish first, then open | In-house raw LZMA2 streams; **no separate repo until library-quality** (ratio/maturity still chasing xz/7z — see NYA [SPEC-CODECS](https://github.com/nyarime/nya/blob/main/SPEC-CODECS.md)) |
| **nya** | ✅ | Archive container + FEC orchestration + distribution; depends on libs above |

Encoder upgrades do **not** change NYA on-disk payload formats for a given
CompressionID; splitting repos is for reuse and testing, not format churn.

## Acknowledgments

- [klauspost/reedsolomon](https://github.com/klauspost/reedsolomon) — The gold standard Reed-Solomon implementation in Go. GoFEC's AVX2 SIMD acceleration and GF(256) VPSHUFB lookup table design are deeply inspired by this exceptional library. Thank you Klaus Post for your outstanding contributions to the Go ecosystem.
- [google/gofountain](https://github.com/google/gofountain) — Pure Go fountain code reference implementation.

## License

Apache License 2.0
