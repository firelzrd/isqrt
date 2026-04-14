# isqrt: Fast Integer Square Root

`isqrt` is a C99 header-only library that computes exact `floor(sqrt(v))` for unsigned integers, significantly faster than the traditional loop-based approach (e.g., Linux kernel's `int_sqrt`).

It is designed for performance-critical environments such as the Linux kernel, embedded systems, and real-time applications where floating-point is unavailable or prohibitively expensive.

## Key Features

- **Exact**: Returns `floor(sqrt(v))`, bit-for-bit identical to `int_sqrt`.
- **Fast**: 2x to 8x faster than `int_sqrt`, depending on bit-width.
- **Trivial Decode**: `h * h` (a single integer multiply). No dedicated decoder needed.
- **Zero Dependencies**: Header-only, no external libraries. Only requires `<stdint.h>` and GCC/Clang `__builtin_clz`.
- **Minimal Memory**: 768 bytes of lookup tables total.

## Performance

Measured with `gcc -O2` on x86-64 (cycles per call, best of 7 rounds):

| Function | int_sqrt | isqrt | Speedup |
| :--- | ---: | ---: | ---: |
| `isqrt8(uint8_t)` | N/A | **~1 cy** | (pure LUT) |
| `isqrt16(uint16_t)` | ~15 cy | **~7 cy** | **x2.2** |
| `isqrt32(uint32_t)` | ~80 cy | **~10 cy** | **x8.0** |
| `isqrt64(uint64_t)` | ~115 cy | **~38 cy** | **x3.0** |

The advantage grows with bit-width because `int_sqrt` is loop-based (8/16/32 iterations), while `isqrt` uses a constant-time initial estimate followed by Newton-Raphson convergence.

## Quick Start

```c
#include "isqrt.h"

uint8_t  s = isqrt8(200);       // 14
uint16_t a = isqrt16(10000);    // 100
uint32_t b = isqrt32(1000000);  // 1000
uint64_t c = isqrt64(1000000000000ULL);  // 1000000

// Decode is just h * h
uint32_t v_approx = b * b;     // 1000000
```

## How It Works

Each bit-width uses the optimal strategy for its range:

### 8-bit: Full LUT (~1 cy, 0 divisions)

A 256-byte table stores exact `floor(sqrt(i))` for all 256 possible inputs. A single indexed memory load — no computation at all.

### 16-bit: Zone Encoding + Delta LUT (~7 cy, 0 divisions)

A CLZ-based zone encoding produces a fast initial estimate. A 256-byte delta correction table adjusts it to within +/-1 of `floor(sqrt(v))`. A single multiply-compare fixup resolves the final result exactly.

### 32-bit: CLZ + Sqrt LUT + Newton-Raphson x1 (~10 cy, 1 division)

A 128-entry sqrt lookup table (256 bytes), indexed by exponent parity and top 6 mantissa bits, provides an initial estimate with ~0.8% accuracy. One Newton-Raphson step (`y = (y + v/y) / 2`) converges to within +/-3, which the fixup resolves.

### 64-bit: CLZ + Sqrt LUT + Newton-Raphson x2 (~38 cy, 2 divisions)

Same LUT as 32-bit (shared 256 bytes). Two Newton-Raphson steps provide full 32-bit output precision. The fixup resolves the final +/-1.

## API

```c
uint8_t  isqrt8(uint8_t v);    // floor(sqrt(v)) for 8-bit
uint16_t isqrt16(uint16_t v);  // floor(sqrt(v)) for 16-bit
uint32_t isqrt32(uint32_t v);  // floor(sqrt(v)) for 32-bit
uint64_t isqrt64(uint64_t v);  // floor(sqrt(v)) for 64-bit
```

All functions return exact `floor(sqrt(v))`. Input and output share the same type.

## Platform Dependencies

Requires GCC/Clang for:
- `__builtin_clz()` (32-bit count leading zeros)
- `__builtin_clzll()` (64-bit count leading zeros)

For MSVC, substitute `_BitScanReverse` / `_BitScanReverse64`.

## License

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 2 of the License, or (at your option) any later version.

Copyright (C) 2026 Masahito Suzuki.
