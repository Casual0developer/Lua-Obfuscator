# LuaVeil performance pass

Research and tuning completed over roughly 20 minutes on October 24, 2026.

## What changed

- Fast access mode preserves literal property names and literal table keys. This keeps Luau's field caches, global imports, builtin fastcalls, and table construction optimizations available. It exposes these key names; turn it off for full key encoding.
- Encoded strings referenced inside loops can be cached in function-local variables, with a limit of eight per function and explicit register headroom. A shuffled-pool index issue found by the new safety tests was corrected before delivery.
- The strict-local workaround now prefers colder, less-captured, less-mutated bindings for packing, instead of selecting declarations in source order. Loop references receive additional weight.
- Decoder arithmetic/feedback dispatch runs once per layer rather than once per byte. Output characters are built in batches of 96 bytes. The portable fallback uses `table.unpack or unpack`.
- Browser-side generation reuses encoders, memoizes repeated literals and keys, batches name/salt randomness, uses compact byte signatures, and computes modular inverses directly.
- Generated helper names are scrambled before packed records are inserted, avoiding repeated parsing of the entire encoded payload.
- Compact now selects one layer, fast access, no junk, no padding, and no integer masks. Balanced keeps three layers with fast access. Large uses four layers and full key encoding. Other controls remain adjustable.
- The result page shows the selected access mode, build time, preserved field accesses, and cached loop constants.

The compiler behavior behind the fast-access changes is documented in [Luau's performance guide](https://luau.org/performance/): predictable constant keys enable inline caching, recognizable builtin calls enable fastcalls, and local functions remain eligible for compiler optimization. The decoder batching uses the documented [Luau standard library](https://luau.org/library/).

## Measurements

Times below are milliseconds per invocation. The harness uses the official Luau CLI runtime, seven rotating-order rounds in each of three independent processes, and medians. Every benchmark verifies its result against the original. Compilation is excluded from these runtime timings.

The before/current comparison uses three encoding layers without junk or padding. Before uses full key encoding; current enables fast access. This is an explicit speed/readability tradeoff, not identical protection settings.

| Workload | Original | Previous obfuscator | Updated obfuscator | Speedup over previous |
| --- | ---: | ---: | ---: | ---: |
| 1M table-field reads | 3.509 | 8.486 | 3.495 | 2.43x |
| 500K iterations of builtin arithmetic calls | 5.418 | 18.399 | 5.399 | 3.41x |
| 500K method calls | 8.729 | 11.567 | 8.945 | 1.29x |
| 500K string-byte calls | 5.803 | 16.092 | 5.562 | 2.89x |
| 1M numeric-loop iterations | 6.986 | 6.997 | 6.991 | About equal |
| Hot loop in a chunk with 220 locals | 3.131 | 7.647 | 3.103 | 2.46x |
| Initialize 493 encoded constants | 0.0022 | 4.706 | 3.891 | 1.21x |
| Initialize a 128 KiB string | 0.00055 | 54.073 | 43.292 | 1.25x |

Generation of the complete supplied script with three layers, 32 junk blocks, and 512 KiB padding improved from a median 311.6 ms to 265.0 ms, approximately 15%. That test includes validation and code generation. Different machines, randomized records, garbage collection, and runtime conditions can change these results.

The updated hot paths are mostly close to the original. The cached-string case measured about 4% faster than the original in this microbenchmark, while the method-call case remained about 2.5% slower. Small differences should not be treated as universal speed gains. Encoding still adds startup work. No general obfuscator can promise to make every input faster than its original.

## Delivered script and validation

`obfuscated.lua` was regenerated from the latest supplied attachment using Compact: one encoding layer, fast access, 409 unique encoded strings, 58 loop-constant caches, 52 packed locals, no junk, no padding, and no integer masking. The file is approximately 180 KB and retains the exact three supplied watermark comments.

It passes official Luau compilation at O0, O1, O2, and O1 with local-register reuse disabled. Its maximum declared-local count is 171. The Roblox script was compiled, not executed in Roblox.

Additional validation includes 60 performance-safety checks, three focused O0 compiler checks, the existing 178 behavior/byte comparisons, 31 limit assertions, and the worker/output-page checks. Coverage includes both access modes, all feedback modes, repeated loops, shadowed builtins, metatables, closures, constant-cache limits, batch boundaries, and parameter-heavy functions.

Client-side recovery limits remain unchanged: the decoder and parameters are shipped with the output. The exact recovery findings remain in `AUDIT.md`.
