# GoFEC

[![Go Reference](https://pkg.go.dev/badge/github.com/nyarime/gofec.svg)](https://pkg.go.dev/github.com/nyarime/gofec)

纯Go前向纠错 (FEC) 纠删码库，SIMD硬件加速，无CGO依赖。

GoFEC提供两种纠删码实现——**RaptorQ喷泉码**和**LDPC低密度校验码**，配合AVX2/NEON汇编加速的GF(256)有限域运算。

项目主页: https://github.com/Nyarime/GoFEC

Godoc: https://pkg.go.dev/github.com/nyarime/gofec

[English](README_EN.md)

## 安装

```bash
go get github.com/nyarime/gofec
```

推荐使用Go Modules。

## 特性

| 编码 | 描述 | 状态 |
|------|------|------|
| RaptorQ | RFC 6330 喷泉码，无限修复符号 | ✅ |
| LDPC | PEG矩阵 + BP迭代解码 | ✅ |
| GF(256) | Log/Exp查表 + SIMD向量化 | ✅ |
| XOR加速 | AVX2/NEON硬件加速 | ✅ |

### SIMD加速栈

GoFEC在运行时自动检测CPU特性，选择最优指令集：

| 平台 | XOR | GF(256) MulAdd |
|------|-----|----------------|
| amd64 AVX512+GFNI | 32B/op | 64B/op (VGF2P8AFFINEQB) |
| amd64 AVX2 | 32B/op (VPXOR) | 32B/op (VPSHUFB) |
| amd64 SSE2 | 16B/op | 标量 |
| arm64 NEON | 16B/op (VEOR) | 16B/op (VTBL) |
| 通用 | 8B/op (uint64) | 标量 Log/Exp |

## 使用

### RaptorQ喷泉码

RaptorQ是一种无速率(rateless)纠删码——可以从源数据生成**无限个**修复符号。接收端只要收到**略多于源符号数量**的任意符号组合，就能恢复原始数据。

这种特性在丢包率不可预知的场景(如移动网络、卫星链路)中非常有用。

```go
import "github.com/nyarime/gofec/raptorq"

// 创建编解码器: 16个源符号, 每个128字节
codec := raptorq.New(16, 128)

// 编码: 源数据 → 源符号 + 8个修复符号
symbols := codec.Encode(data, 8)

// 模拟网络传输: 丢失部分符号...
// 只要收到≥16个符号(任意组合), 就能恢复

// 解码
decoded, err := codec.Decode(receivedSymbols, len(data))
if err != nil {
    log.Fatal("修复符号不足:", err)
}
```

**关键参数选择:**
- `K` (源符号数): 越大编码效率越高，但延迟也越高。推荐8~64。
- `T` (符号大小): 通常与网络MTU相关。推荐64~1024字节。
- 修复符号数: 越多容错越强。建议至少`K * 0.5`。

### LDPC低密度校验码

LDPC是一种固定码率的分组码，编解码速度极快，适合已知丢包率的低延迟场景。

```go
import "github.com/nyarime/gofec/ldpc"

// 创建编解码器: 10个数据块, 4个校验块, PEG矩阵密度0.3
codec := ldpc.New(10, 4, 0.3)

// 编码: 数据块 → 数据块 + 校验块
encoded := codec.Encode(dataShards)

// 丢失部分块后恢复
err := codec.Decode(partialShards)
```

### RaptorQ vs LDPC: 如何选择？

| 场景 | 推荐 | 理由 |
|------|------|------|
| 丢包率未知 | RaptorQ | 喷泉码天生适应任意丢包率 |
| 实时视频/游戏 | LDPC | BP解码延迟极低 |
| 文件传输/分发 | RaptorQ | 不用提前知道丢多少包 |
| 卫星/高丢包链路 | RaptorQ | 多发修复符号即可 |
| 低丢包+低延迟 | LDPC | 固定少量冗余，解码快 |

**一句话总结:** LDPC更快更省(已知丢包)，RaptorQ更灵活更鲁棒(未知丢包)。

## 性能

测试环境: Intel Broadwell, Go 1.25, Linux amd64

### RaptorQ编码

| 数据大小 | 延迟 | 吞吐 | 内存分配 |
|----------|------|------|----------|
| 512B | 2.4μs | 213 MB/s | 14 allocs |
| 2KB | 5.0μs | 400 MB/s | 27 allocs |
| 8KB | 13.5μs | 593 MB/s | 53 allocs |
| 32KB | 41μs | 780 MB/s | 103 allocs |

### RaptorQ丢失恢复

| 丢失块数 | 8源+N*3修复 | 状态 |
|----------|-------------|------|
| 1~6 | ✅ | 全部恢复成功 |

### LDPC编解码

| 操作 | 延迟 |
|------|------|
| 编码 | 18μs/op |
| 解码 | 20μs/op |

### SIMD加速基准

| 操作 | 吞吐 | 零分配 |
|------|------|--------|
| XOR (AVX2, 4KB) | 31.6 GB/s | ✅ |
| GF(256) MulAdd (AVX2, 1KB) | 15.0 GB/s | ✅ |
| GF(256) Mul (标量) | 607M ops/s | ✅ |

## 架构

```
GoFEC/
├── codec.go                  # 统一Codec接口
├── raptorq/                  # RaptorQ喷泉码 (RFC 6330)
│   ├── raptorq.go            # 编解码器 (LT编码+高斯消元)
│   └── xor.go                # SIMD XOR桥接
├── ldpc/                     # LDPC低密度校验码
│   ├── ldpc.go               # BP迭代解码
│   ├── peg.go                # PEG矩阵构造
│   └── xor.go                # SIMD XOR桥接
└── internal/
    ├── xor/                  # 硬件加速XOR
    │   ├── xor_amd64.s       # AVX2汇编
    │   ├── xor_arm64.s       # NEON汇编
    │   └── xor_generic.go    # 通用fallback
    └── gf256/                # GF(2^8)有限域
        ├── gf.go             # Log/Exp查表
        ├── tables.go         # 预计算Split Tables (8KB)
        ├── mulAdd_amd64.s    # AVX2 VPSHUFB
        ├── mulAdd_gfni_amd64.s  # AVX512 GFNI
        ├── mulAdd_arm64.s    # NEON VTBL
        ├── cpu_amd64.go      # CPU特性检测
        └── cpu_arm64.go      # CPU特性检测
```

## 路线图

- [x] v1.0 — RaptorQ + LDPC + AVX2 + NEON
- [x] v1.x — AVX512 GFNI运行时加速 (3倍GF提速)
- [x] v2.0 — GF(2^16) 公开包 + Leopard-RS 高层 API (65536分片) + RaptorQ 喷泉扩展


## Benchmark

AMD EPYC 9654 (AVX512 GFNI) 实测:

| 操作 | 数据大小 | 吞吐 |
|------|---------|------|
| GF(256) MulAdd (GFNI) | 4KB | **87 GB/s** |
| GF(256) MulAdd (GFNI) | 1KB | **68 GB/s** |
| RaptorQ Encode | 32KB | **2.06 GB/s** |
| RaptorQ Encode | 2KB | **1.36 GB/s** |
| LDPC Encode | 2KB | **286 MB/s** |
| LDPC Decode | 2KB | **263 MB/s** |

SIMD 自动调度: AVX512 GFNI > AVX2 > SSE4.1 > NEON > scalar

## 被引用

GoFEC 是 Nyarime 自研栈里的**独立 FEC 库**——NYA 归档通过 `go get` 引用它，而不是把纠删码锁死在归档格式里。同类思路也适用于未来的独立编解码库（见下方生态）。

| 项目 | 用途 |
| --- | --- |
| [NYA](https://github.com/nyarime/nya) | 归档 + FEC + 分发（`nya augment`、内嵌 FEC；Leopard-RS / Hybrid / RaptorQ 等经 GoFEC） |
| [NRUP](https://github.com/Nyarime/NRUP) | Nyarime Reliable UDP Protocol（传输层 FEC） |

### Nyarime 库生态（方向）

| 库 | 状态 | 说明 |
| --- | --- | --- |
| **GoFEC**（本仓库） | ✅ 独立开源 | 纯 Go FEC，SIMD，无 cgo |
| **nyazstd** | 📋 计划拆库 | 自研 RFC 8878 Zstd 编码器/解码器；目前在 `nya` 包内，目标像 GoFEC 一样可 `go get` 单独用 |
| **nyalzma2** | ⏳ 先打磨，再开源 | 自研 raw LZMA2 流；**未达对外库质量前不单独发仓库**（比率/成熟度仍追 xz/7z，见 NYA [SPEC-CODECS](https://github.com/nyarime/nya/blob/main/SPEC-CODECS.md)） |
| **nya** | ✅ | 归档容器 + FEC 编排 + 分发；引用上述库而非内嵌一切 |

编解码器改进**不改变** NYA 盘上 `CompressionID` 对应的 payload 格式；拆库是为了复用与测试，不是为了换格式。

## 致谢

- [klauspost/reedsolomon](https://github.com/klauspost/reedsolomon) — Go生态中最优秀的Reed-Solomon实现。GoFEC的AVX2 SIMD加速策略和GF(256) VPSHUFB查表设计深受其启发。感谢Klaus Post为Go社区做出的卓越贡献。
- [google/gofountain](https://github.com/google/gofountain) — 纯Go喷泉码参考实现。

## 许可证

Apache License 2.0

