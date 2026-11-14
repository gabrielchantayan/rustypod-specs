<!-- Trusted provenance: F-ba5aba645a3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ba5aba645a3b

## Purpose

F-ba5aba645a3b exposes a straight-line sequence of indirect calls, byte-oriented selector accesses, and 32-bit memory updates. The supplied representation does not establish a higher-level purpose such as initialization, serialization, allocation, or device control.

The recoverable contract is therefore an **ordered interaction contract**, not a semantic description of an identified subsystem. Its distinguishing behavior includes:

- Forwarding one explicit input and several ambient machine-state values to opaque dependencies.
- Using dependency results to construct memory addresses through scaling and fixed offsets.
- Reading unsigned byte selectors and using them as absolute destinations for subsequent byte writes.
- Extracting and forwarding selected portions of 32-bit values.
- Splitting one represented 64-bit dependency result into two 32-bit components.
- Reaching a terminal bad-instruction boundary rather than a demonstrated normal return.

All behavior below is established only within the supplied static representation. That representation carries an explicit control-flow reliability warning. It is not evidence that an executable safely performs the entire sequence.

## Interface

### Explicit inputs

The represented interface has four positional inputs:

| Position | Represented width | Visible use |
|---|---:|---|
| First | 32 bits, interpretation unspecified | Passed unchanged to the first indirect call. |
| Second | 32 bits, interpretation unspecified | No explicit use shown. |
| Third | 32 bits, interpretation unspecified | No explicit use shown. |
| Fourth | 32-bit signed integer | Shifted left by 10 and supplied twice to an intermediate dependency. |

No visible use of the second and third inputs does not establish that their incoming machine registers are unobservable to opaque dependencies.

### Ambient inputs

The represented body also depends on machine state absent from the four-input interface:

- An incoming link-state value used to construct the first indirect-call target.
- An unsigned 32-bit ambient value whose upper 20 bits become the first call’s second explicit argument.
- An ambient address used for an unsigned byte read at offset 1.
- An ambient 32-bit word written to memory early in the sequence.
- A secondary machine-result value, designated **E** below, whose precise producer and calling-convention meaning are unresolved.

These are required execution inputs, not established globals or owned objects.

### Outputs and completion

The interface is represented as having no ordinary return value. Observable outputs instead consist of dependency interactions and memory effects.

No normal return is shown. The supplied representation ends at a bad-instruction termination marker after the final indirect call. This does not establish a particular hardware exception, exit status, or guaranteed termination behavior for the actual executable.

### Notation and arithmetic

For this contract:

- **F-U001** through **F-U122** identify opaque dependency invocation sites in encounter order. These are document-local opaque IDs, not recovered function identities. Different IDs need not identify different callees.
- **Rₙ** denotes the represented primary result of invocation n, when subsequently consumed.
- **L** and **H** denote, respectively, the low and high 32-bit halves of the represented 64-bit result from F-U038.
- **W(a)** denotes a 32-bit memory access at address a.
- **B(a)** denotes an unsigned 8-bit read at address a.
- A byte write retains only the low eight bits of its supplied value.

Address expressions identify absolute computed addresses. For example, a byte selected by B(a) produces a destination of B(a) + 17; it does not select an offset within the object containing a.

Unsigned arithmetic is 32-bit unless otherwise stated. Unsigned multiplication and left shifts retain the low 32 bits. Unsigned right shifts are logical. Some recovered address arithmetic and left shifts instead involve signed 32-bit values. Their overflow behavior is not established as a portable signed-language contract.

## Behavior

### Overall ordering

The represented path has 122 indirect invocations, 15 explicit 32-bit stores, and six explicit byte stores. It also contains six explicit byte reads and three explicit 32-bit read expressions.

There are no visible conditional branches, loops, validation checks, or recovery paths. This describes the supplied representation, not a guarantee about control flow inside dependencies.

The following phases specify the ordering of explicit effects. Every described continuation is conditional on preceding dependencies returning and preceding accesses completing.

### 1. Initial word publication and selector-byte updates

F-U001 receives the first explicit input and the ambient unsigned value shifted right by 12.

F-U002 through F-U005 follow without explicit arguments. Using R₅ as an index, the function writes the ambient 32-bit word to:

**W(4 × R₅ + 20).**

F-U006 then receives 4 × R₅.

F-U007 through F-U010 have no explicit arguments. After F-U010 returns, the function:

1. Reads an unsigned byte at the ambient address plus 1.
2. Writes the low byte of 4 × R₁₀ to the absolute address equal to that byte plus 17.

F-U011 receives 4 × R₁₀. F-U012 through F-U015 have no explicit arguments. The next selector transaction reads:

**B(64 × R₁₀ + 1).**

Call this selector **S₁**. The function writes to absolute address S₁ + 17 the low byte of:

**(R₁₀ AND 0x3fffffff) shifted right by 20.**

The mask-and-shift expression can produce ten significant bits before byte truncation. The actual stored byte contains original bits 20 through 27; bits 28 and 29 do not survive the byte write.

F-U016 receives 4 × R₁₅. F-U017 through F-U020 have no explicit arguments. F-U021 receives, in order:

- 4 × R₂₀;
- S₁.

S₁ is the previously captured selector, not a new memory read.

### 2. Fourth-input forwarding, zeroing, and secondary-result extraction

F-U022 has no explicit arguments. F-U023 receives the fourth explicit input shifted left by 10 in both argument positions. F-U024 has no explicit arguments.

The next direct memory action reads B(1), then writes zero to the absolute address equal to that byte plus 17. The address 1 follows from a visibly zero-valued intermediate; it is not established as a base-relative structure field.

F-U025 receives E, E, and 16 × E, in that order.

F-U026 through F-U029 have no explicit arguments. The function then reads:

**B(16 × E + 1).**

Call this selector **S₂**. It writes the low byte of E shifted right by 22 to absolute address S₂ + 17. The stored byte therefore reflects bits 22 through 29 of E, assuming the represented unsigned 32-bit interpretation.

F-U030 receives 4 × R₂₉. F-U031 through F-U034 have no explicit arguments. F-U035 receives:

- S₂;
- 4 × R₃₄.

This argument order differs from F-U021.

F-U036 has no explicit arguments. F-U037 receives two zero arguments. F-U038 has no explicit arguments and produces the represented 64-bit result split into L and H.

The function writes zero to:

**W(4 × H + 20).**

F-U039 then receives H.

### 3. Publishing and transforming the split result

F-U040 through F-U043 have no explicit arguments. The function writes H to:

**W(4 × R₄₃ + 20).**

F-U044 receives 4 × R₄₃. F-U045 through F-U048 have no explicit arguments.

The next direct store is:

- Destination: **W((L × 2²⁸) + 20)**.
- Value: **L × 2²⁶**.

These are distinct scales. Under unsigned 32-bit arithmetic, the address contribution depends only on the low four bits of L, while the value depends only on its low six bits.

F-U049 receives L × 2²⁶. F-U050 through F-U053 have no explicit arguments. The function then stores:

**(L AND 0x1fffffff) shifted right by 9**

at:

**W(4 × R₅₃ + 20).**

The stored value is in the range 0 through 0xfffff.

F-U054 receives 4 × R₅₃.

### 4. Scaled-index records and dependent forwarding

This phase repeatedly uses two different address forms:

- A result scaled by 16, plus 20, receives that same result scaled by 4.
- A later result scaled by 4, plus 20, receives the previously computed scaled value.

These forms are not interchangeable and do not establish a conventional array layout.

F-U055 through F-U058 have no explicit arguments. The function writes 4 × R₅₈ to:

**W(16 × R₅₈ + 20).**

F-U059 receives three arguments:

1. A 32-bit read from W(4 × R₅₈ + 20), shifted right by 12.
2. 4 × R₅₈.
3. A 32-bit read from W(4 × R₅₈ + 20), shifted left by 2.

The supplied expression contains two reads of that address. Their relative evaluation order is not established. They must not be assumed to produce equal values for volatile, device-backed, or concurrently modified storage. Neither read is necessarily from the location just written, because the write used a scale of 16 rather than 4.

F-U060 through F-U063 have no explicit arguments. The function writes 4 × R₅₈ to W(4 × R₆₃ + 20), then invokes F-U064 with 4 × R₆₃.

The remaining interactions in this phase are:

| Result-producing no-explicit-argument chain | Direct store after that chain | Following invocation |
|---|---|---|
| F-U065–F-U068 | W(16 × R₆₈ + 20) receives 4 × R₆₈ | F-U069 receives 4 × R₆₈, 4 × R₆₈, and L shifted left by 26 |
| F-U070–F-U073 | W(4 × R₇₃ + 20) receives 4 × R₆₈ | F-U074 receives 4 × R₇₃ |
| F-U075–F-U078 | W(16 × R₇₈ + 20) receives 4 × R₇₈ | F-U079 receives 4 × R₇₈ |
| F-U080–F-U083 | W(4 × R₈₃ + 20) receives 4 × R₇₈ | F-U084 receives 4 × R₈₃ |
| F-U085–F-U088 | W(16 × R₈₈ + 20) receives 4 × R₈₈ | A read described below precedes F-U089 |

After the last store, the function reads W(4 × R₈₈ + 20). Designate this captured 32-bit value **T**. F-U089 receives 4 × R₈₈.

T is read before F-U089 and retained for a later extraction. It is not a read deferred until the later store.

### 5. Captured-word extraction and final scaled-index sequence

F-U090 through F-U093 have no explicit arguments. The function stores T shifted right by 12 at:

**W(4 × R₉₃ + 20).**

The right shift is unsigned, so the stored value is at most 0xfffff.

F-U094 receives 4 × R₉₃. F-U095 through F-U098 have no explicit arguments. The function writes 4 × R₉₈ to:

**W(16 × R₉₈ + 20).**

F-U099 receives, in order:

1. **((4 × R₉₃) AND 63) shifted left by 14**;
2. 4 × R₉₈;
3. R₉₃ shifted left by 30.

The first argument is a scaled low-bit field, not a dereference. Its possible values are multiples of 65536 from zero through 0xf0000, under 32-bit arithmetic. The third argument retains only the low two bits of R₉₃ in the high two output positions under a machine-bit-vector interpretation; signed-shift validity remains unresolved.

F-U100 through F-U103 have no explicit arguments. The function writes 4 × R₉₈ to W(4 × R₁₀₃ + 20), then invokes F-U104 with 4 × R₁₀₃.

F-U105 through F-U108 have no explicit arguments. The next selector address is:

**R₉₃ × 2²⁸ + 1.**

The function reads a byte there and writes the low byte of 4 × R₁₀₈ to the absolute address selected by that byte plus 17.

### 6. Final selector reread and terminal boundary

F-U109 receives 4 × R₁₀₈, 4 × R₁₀₈, and 16 × R₁₀₈.

F-U110 has no explicit arguments. F-U111 receives R₁₁₀ unchanged. F-U112 through F-U115 have no explicit arguments.

The function now **rereads** the selector byte at:

**R₉₃ × 2²⁸ + 1.**

Call this fresh value **S₃**. It writes to absolute address S₃ + 17 the low byte of:

**(R₁₀₈ AND 0x3fffffff) shifted right by 20.**

The earlier selector read cannot substitute for S₃. Dependencies between the two reads may change the referenced memory, and direct-memory aliasing is not excluded.

F-U116 receives 4 × R₁₁₅. F-U117 through F-U120 have no explicit arguments. F-U121 receives 4 × R₁₂₀ followed by S₃. F-U122 has no explicit arguments.

The supplied representation then reaches its bad-instruction boundary.

## Side effects

### Memory layout and access requirements

The visible layouts are access patterns only:

- Four-byte values at computed addresses ending in an added offset of 20.
- One-byte selectors at computed addresses ending in an added offset of 1.
- One-byte writes to absolute addresses from 17 through 272, because each selector is an unsigned byte.

No object base, allocation size, field name, endianness, or memory-mapped device identity is established.

For execution to progress through the represented path, every actual call target must be executable and every computed memory access must satisfy the target environment’s mapping, permission, alignment, and access-width requirements. No visible code establishes those conditions.

The byte destination range is unusually low. It must not be silently reinterpreted as relative addressing to make the behavior appear safer.

### Visibility, aliasing, and ownership

The listed direct stores are ordered around the dependency calls as described. Dependencies may perform additional reads, writes, allocation, release, synchronization, or nonlocal control transfer; their effects are unknown.

Computed locations may alias each other, inputs, or dependency-managed storage. No disjointness guarantee is visible. A write may consequently affect a later selector read or word read.

No allocation, ownership transfer, release, lifetime extension, or reference-counting contract is established. In particular:

- A dependency result used in address arithmetic is not thereby proven to identify allocated memory.
- Passing a scaled value does not establish borrowing or transfer.
- Retaining a recovered value across a call does not establish that every relevant machine register or global is preserved by that callee.

### Errors

There is no represented status check, null check, bounds check, rollback, or cleanup path. Earlier stores are not visibly undone if a later call or access fails.

A callee may fail to return. An access may fault. The static evidence does not establish error codes, exception types, or a portable failure mode.

## Dependencies

F-U001–F-U122 are opaque invocation-site IDs. Their argument ordering and consumed results are specified above; their implementations are unavailable.

The first target is constructed from incoming link state plus 4094. Subsequent targets are constructed from embedded code-location-like values, generally with an added displacement of 4094. F-U109 uses a displacement of 4076 instead. These constructions do not establish stable library entry points, callable signatures, or distinct target identities.

“No explicit arguments” describes the recovered call expression only. It does not prove that a callee consumes no ambient registers, stack data, globals, or memory.

The represented 64-bit result at F-U038 may reflect a multi-register calling convention. Its semantic meaning and actual declared return type are unknown. E similarly requires a calling-convention interpretation that the evidence does not supply.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial word publication.** If R₅ is 3, the ambient word is 0x89abcdef, and the destination is valid, the first direct store writes that word at address 32. F-U006 subsequently receives 12.

2. **Selector addressing.** If the first selector read yields 5 and R₁₀ is 65, the corresponding byte destination is address 22 and the stored byte is 4. The destination is not the selector source address plus 17.

3. **Masked byte extraction.** If R₁₀ is 0x3ab00000, the later masked extraction produces 0x3ab before truncation and stores 0xab at S₁ + 17.

4. **Zeroing through a selector.** If B(1) yields 255, the zero-byte write targets address 272. A zero selector instead targets address 17; neither case is visibly rejected.

5. **Split-result effects.** If F-U038 supplies H = 2 and L = 1, the immediate zero-word destination is address 28. The later L-derived store targets address 0x10000014 and writes 0x04000000, provided intervening execution reaches it.

6. **Different scaling is significant.** If R₅₈ is 2, the direct store targets address 52 and contains 8. F-U059’s two read expressions access address 28, not address 52.

7. **Captured read timing.** If T is captured as 0xabcdefff, the later store based on R₉₃ writes 0x000abcde. A dependency changing T’s original memory location after capture does not, in the represented dataflow, replace the captured value.

8. **Final selector freshness.** If the final selector source first yields 2 and later yields 9, the two byte writes target addresses 19 and 26 respectively. Treating both as writes to address 19 would violate the visible reread.

9. **Failure after partial effects.** If an intermediate dependency does not return, no later explicit effects are required. No rollback of completed stores is established.

## Uncertainties

The following are **documented_only**, not acceptance assumptions:

- Whether the recovered instruction stream and function boundary are valid.
- The actual calling convention, including ambient inputs, E, and the represented 64-bit result.
- Signed overflow and signed-left-shift behavior.
- Whether memory accesses are ordinary, volatile, device-backed, or concurrently modified.
- Actual dependency signatures, identities, effects, and preservation guarantees.
- Address-space width and the executable’s precise address-calculation semantics.
- Endianness, alignment tolerance, and object boundaries.
- Ownership, allocation provenance, and memory lifetime.
- Whether the terminal marker corresponds to an executed invalid instruction, an analysis boundary, or another control-flow outcome.

An inferred intent is that repeated scale-and-offset operations publish transformed values into some structured state. That interpretation is not sufficiently supported to define named fields, objects, or subsystem semantics.

## Evidence

The supplied static representation establishes the explicit inputs, recovered dataflow, dependency-call ordering, access widths, scale factors, masks, shifts, and terminal analysis boundary described here.

It also explicitly warns that control flow encountered bad instruction data and was truncated. That warning limits confidence in the executable-level interpretation of the entire contract.

No dependency bodies, calling-convention documentation, independent disassembly, memory map, or runtime observations were provided. **No runtime tests have been performed.**
