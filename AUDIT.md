# LuaVeil improvement and recovery audit

Updated October 24, 2026. The tool remains an offline, three-page HTML application. Supplied Lua is parsed in a worker; it is never executed by the website.

## Recovery limits confirmed by a test

The encoded constants remain recoverable. An audit test extracted the emitted decoder initializer and recovered a sentinel string without running the original program body. The decoder and all parameters needed to reverse the layers are shipped with the output. Runtime inspection can also observe decoded strings and values.

Renaming discards the original local names, but program behavior, public API names, and decoded constants can still be studied. Formatting can restore readable layout. Integer arithmetic masks can be folded. The feedback layers are reversible obfuscation, not a standard cryptographic cipher or a confidentiality boundary. Additional layers do not remove this recovery path.

The new checksum catches damaged records. It is not a signature, keyed authentication, license enforcement, or protection against someone removing the check. Symbol padding and junk calculations provide visual bulk and can be removed.

## Implemented changes

- Constant records now choose among three arithmetic arrangements and three feedback modes per layer, with independent random parameters and optional reversal. Decoder locals and parameters are randomized too.
- Record validation rejects malformed packed data, invalid layer metadata, and checksum mismatches with a clear error.
- Supported nonnegative integer literals through 2,147,483,647 use randomized, exact hexadecimal arithmetic. Floating-point and larger literals retain their original representation. This avoids rounding and integer/float type changes.
- Named function declarations can become assignments with encoded property names. Implicit method parameters are tracked and renamed. Calls using `:` retain their normal behavior, including Roblox namecalls.
- Large Luau functions each receive their own local-packing plan. Grouped declarations, missing initializers, later function assignment, shadowing, closures, and multi-return initialization preserve their binding relationships. An overflowed source can be repaired when enough locals can be packed safely.
- Output is checked against the strict 200-local ceiling for every function. The normal packing target is 170 for headroom; a function with many parameters can use the real 200-local limit.
- Single-line output uses tokens instead of global whitespace replacement. It now works with string encoding disabled, preserving string bytes and embedded newlines through escaped literals.
- Source comments are removed by default in multiline output as well. Comment preservation remains selectable for multiline output.
- Compact, Balanced, and Large presets configure layers, junk, and padding. Number masking and comment removal have dedicated controls. Output shows the number of masked integers, packed locals, and rewritten function declarations.
- Output-size enforcement measures UTF-8 bytes as well as character count. Existing worker cancellation, timeout, storage fallback, safe text rendering, and offline content policies remain in place.

## Concrete defects corrected

1. A packed local used as the base of `function object:method()` could produce an invalid indexed function declaration. Function assignments now support indexed targets and preserve the implicit receiver.
2. The previous local-packing workaround applied only to the outer chunk. Nested factories now receive separate state tables and strict output checks.
3. Valid parameter-heavy functions could be rejected by an overly conservative headroom target. The language's actual limit remains 200.
4. A multiline source comment could retain original names or other information in the output. Comment removal is now independent of layout.
5. Malformed constant data could fail later with confusing arithmetic errors. Metadata and decoded-byte checks now reject tested corruption immediately.
6. The Luau compatibility parser shortened binary and underscored numeric ranges. Number masking could replace only the first character. The complete original ranges and enclosing ranges are now restored, with malformed normalized numerals rejected.

## Validation

- 80 focused behavior comparisons cover function and method declarations, receivers, nested factories, packed grouped/forward declarations, multi-return initialization, mutable closures, comment handling, string bytes, exact numbers, Luau numeric syntax, and all nine arithmetic/feedback combinations at one and four layers.
- 98 additional byte/behavior comparisons cover the supported parser modes, encoding layers, all 256 byte values, Unicode/escape handling, and literal boundaries.
- 12 delimiter cases and four padding sizes passed. Two deliberately corrupted records were rejected.
- 31 compiler-limit assertions, 22 existing compiler accept/reject cases, and four additional synthetic runtime comparisons passed.
- 18 focused official Luau compiler checks passed, including optimization disabled, strict register-reuse flags, and a 200-parameter boundary case.
- 17 worker/editor checks and 15 output/help checks passed, including unsafe HTML rendered as text, storage fallback, presets, new controls, and processing the complete attachment.
- The earlier full-key build used 673 unique encoded constants, 977 masked integers, 52 packed locals, 32 junk blocks, 512 KiB padding, and a maximum of 170 declared active locals in any generated function. It passed official Luau compilation at O0, O1, O2, and O1 with local-register reuse disabled. The subsequent performance pass and current Compact artifact are documented in [PERFORMANCE.md](PERFORMANCE.md).

The full Roblox script was compiled only. Roblox execution and executor-specific runtime behavior remain unverified. Browser interaction checks used a DOM and real worker test harness; this pass did not inspect the live browser's pixels.

## Performance and remaining constraints

Constants are decoded once, then read from a cached pool. More layers and validation add startup work. On this machine, a synthetic 493-constant case with three layers took about 5.1 ms per initialization, measured as the best of three batches of 30 runs using the official Luau runtime. This is not a Roblox performance guarantee. The Compact preset reduces layers, junk, and padding.

The parser supports a subset of Luau. Type annotations, interpolated strings, if expressions, and Lua 5.4 attributes remain unsupported. Local/upvalue checks do not model every temporary-register or compiler implementation limit. A full VM transformation is not included. Debug-based code can observe changed names, bindings, or function declaration form.
