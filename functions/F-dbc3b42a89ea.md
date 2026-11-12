<!-- Trusted provenance: F-dbc3b42a89ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dbc3b42a89ea

## Purpose

The available static evidence describes a routine that interleaves indirect calls with calculated memory writes. Its application-level purpose is not established.

The visible behavior consists of:

- Forwarding a single explicit 32-bit input to an indirect dependency.
- Obtaining 32-bit results from later dependencies and using them to calculate memory addresses, stored values, and subsequent call arguments.
- Performing eight word-width stores and four byte-width stores.
- Reading four unsigned bytes that select the destinations of the byte stores.
- Reaching a terminal boundary where instruction recovery failed.

This is a contract for the **visible recovered prefix**, not a claim that the original routine successfully completes that prefix or terminates there. The evidence explicitly reports invalid instruction data and overlapping global symbols. Those limitations materially affect confidence in the recovered control flow and memory interpretation.

No allocator, serializer, device protocol, object model, or other higher-level role can be established from the supplied material.

## Interface

### Explicit input and output

The explicit input is one opaque 32-bit value, called **the input word** below. Its signedness, encoding, and validity constraints are unspecified. The routine forwards its bit pattern to the first visible dependency without an explicit transformation.

The recovered interface declares no return value. No ordinary return is visible. Successful return, a return status, and output through a conventional result register are therefore not established.

### Ambient inputs

The recovered expressions also depend on values not supplied through the explicit parameter:

| Name used in this contract | Observed width and role |
|---|---|
| Ambient word A | Unsigned 32-bit quantity used to derive a word-store address and value |
| Ambient word B | Unsigned 32-bit quantity used in logical right shifts and later address calculations |
| Ambient payload | Opaque 32-bit value written to three calculated destinations |
| Ambient call base | Address-related value used to calculate the first indirect call target |

These names describe operands in the recovered representation. They do not establish a public calling convention, legitimate additional parameters, or preservation across dependency calls.

For compactness, **Q** denotes ambient word B logically shifted right by three bits. Q is an unsigned 29-bit quantity.

### Memory and execution preconditions

Following the entire visible prefix requires:

- Every indirect call target to be executable under the actual platform’s calling convention.
- Every dependency used along the path to return normally.
- Every calculated read and write to be permitted for its access width.
- The ambient operands and dependency results to be meaningful under the actual machine’s address arithmetic.

There are no visible checks enforcing these conditions.

Alignment requirements, address-space mapping, pointer provenance, endianness, concurrency rules, and memory-mapped device semantics are unknown. In particular, several byte destinations are numerically very low addresses. Their validity must not be assumed.

### Arithmetic interpretation

The directly established width rules are:

- Opaque words and unsigned arithmetic operands are 32 bits.
- Unsigned right shifts insert zero bits.
- Selector reads are unsigned 8-bit reads.
- Byte stores retain only the low eight bits of their supplied value.
- Word stores occupy four bytes; their byte order is not established.

Some call results are represented as signed 32-bit integers and then multiplied or left-shifted. The evidence does not establish a portable rule for negative operands, signed overflow, or overflowing address calculations. Examples below avoid those cases unless explicitly stated otherwise.

Unsigned 32-bit arithmetic, including the scaled Q calculation and unsigned result shifts, uses the low 32 bits of the result. Applying this rule to every signed address expression would be an additional assumption, not an established contract.

## Behavior

### Dependency notation and ordering

The indirect calls are assigned opaque, site-scoped IDs **F-D001** through **F-D068**, in their visible execution order.

These IDs identify call occurrences only. They do not assert that the targets are distinct functions, identify their implementation, or imply shared behavior.

A result named **result 007**, for example, means the 32-bit value returned at F-D007. “No explicit arguments” means no arguments appear in the recovered call expression; it does not prove the underlying machine call has no ambient inputs.

The following phases describe observable ordering, not executable pseudocode. Unless a dependency fails to return or a memory access faults, each phase precedes the next.

### 1. Initial calls and first two word stores

The routine first invokes:

1. F-D001 with the input word.
2. F-D002 with no explicit arguments.
3. F-D003 with Q.
4. F-D004, F-D005, and F-D006, each with no explicit arguments.
5. F-D007 with no explicit arguments, retaining result 007.

It then performs two word stores, in this order:

- At address **4 × result 007 + 68**, it writes the ambient payload.
- Let **T** be ambient word A logically shifted right by 22 bits. At address **T + 36**, it writes T as a 32-bit unsigned value.

T is between 0 and 1,023 inclusive. Consequently, the second store’s starting address is between 36 and 1,059 inclusive. This does not imply that the destination is mapped, aligned, or safe.

After both stores, F-D008 receives **4 × result 007**.

### 2. Two further payload stores

F-D009 through F-D012 are invoked with no explicit arguments. Result 012 is used to form the sole explicit argument to F-D013: **result 012 shifted left by two bits**.

The routine retains result 013 and writes the ambient payload as a word at:

**4 × result 013 + 20**

F-D014 then receives four explicit arguments, in this order:

1. Result 013.
2. Result 013 again.
3. Result 013 again.
4. Q.

F-D015 through F-D018 follow, each with no explicit arguments. After obtaining result 018, the routine writes the ambient payload as a word at:

**4 × result 018 + 20**

F-D019 then receives **4 × result 018**.

The three payload stores are distinct operations. The evidence does not establish that they address distinct storage or that intervening dependencies leave earlier destinations unchanged.

### 3. Scaled-result word stores

F-D020 through F-D023 are invoked with no explicit arguments. The routine retains result 023 and forms the scaled value:

**S = 4 × result 023**

It writes S as a word at:

**16 × result 023 + 20**

F-D024 receives S.

F-D025 through F-D028 are then invoked with no explicit arguments. After obtaining result 028, the routine writes the previously formed S as a word at:

**4 × result 028 + 20**

F-D029 receives **4 × result 028**.

F-D030 through F-D033 follow with no explicit arguments. The routine writes **4 × result 033** as a word at:

**16 × result 033 + 20**

F-D034 then receives two explicit arguments:

1. **4 × result 033**
2. **16 × result 033**

The second store in this phase uses the scaled result from F-D023, not a scaled version of result 028. This retained-data relationship is part of the visible behavior.

### 4. Word store derived from ambient word B

F-D035 is invoked with no explicit arguments. Its opaque 32-bit result becomes the first argument to F-D036. The second argument to F-D036 is ambient word B logically shifted right by six bits.

F-D037 through F-D040 are invoked with no explicit arguments. The routine then writes Q as a 32-bit unsigned value at:

**4 × result 040 + 20**

F-D041 receives **4 × result 040**.

This is the eighth and final explicit word store in the visible prefix.

### 5. First selector-based byte store

F-D042 through F-D045 are invoked with no explicit arguments. Result 045 is an unsigned 32-bit quantity retained for two byte-value calculations.

A selector-source base is formed as the low 32 bits of:

**Q × 524,288**

The routine reads one unsigned byte at **selector-source base + 1**. If the byte read is **s₁**, it writes the low eight bits of **result 045 shifted left by two bits** at:

**s₁ + 17**

The selector-derived destination is therefore between 17 and 272 inclusive.

F-D046 then receives the unsigned 32-bit value **result 045 shifted left by two bits**. Its argument is not restricted to the eight bits written to memory.

### 6. Second selector-based byte store

F-D047 is invoked with no explicit arguments. F-D048 receives its opaque result.

F-D049 through F-D052 are invoked with no explicit arguments. Result 052 is retained for a later argument, but it is not the source of the next stored byte.

The routine rereads one unsigned byte at **selector-source base + 1**. If this read produces **s₂**, it writes at **s₂ + 17** the low eight bits of:

**(result 045 AND 0x3FFFFFFF) logically shifted right by 20 bits**

Equivalently, the stored byte contains bits 20 through 27 of result 045. The mask removes bits 30 and 31 before the shift; truncation to a byte also discards shifted bits above bit 7.

F-D053 then receives **result 052 shifted left by two bits**.

The selector is read again rather than reused. Intervening dependencies or earlier writes can therefore affect the second destination.

### 7. Third selector-based byte store

F-D054 through F-D057 are invoked with no explicit arguments. Result 057 is an unsigned 32-bit quantity retained for this and the final byte store.

The routine reads another unsigned byte at **selector-source base + 1**. Calling that value **s₃**, it writes the low eight bits of **4 × result 057** at:

**s₃ + 17**

F-D058 then receives s₃ zero-extended to 32 bits.

The passed selector is the value already read for this store. No additional selector read is visible between the store and F-D058.

### 8. Fourth selector-based byte store and terminal boundary

F-D059 is invoked with no explicit arguments. F-D060 receives its opaque result.

F-D061 through F-D064 follow with no explicit arguments. Result 064 is retained for a later call argument.

The final selector read uses a different source address:

**the low 32 bits of 8 × result 057, plus 1**

If this read yields **s₄**, the routine writes at **s₄ + 17** the low eight bits of:

**(result 057 AND 0x3FFFFFFF) logically shifted right by 20 bits**

Thus, this byte contains bits 20 through 27 of result 057.

F-D065 receives **result 064 shifted left by two bits**. F-D066, F-D067, and F-D068 are then invoked with no explicit arguments.

The supplied representation ends at an invalid-instruction boundary. It provides no trustworthy subsequent behavior or ordinary return path.

## Side effects

### Explicit memory effects

If execution reaches every operation, the visible prefix contains:

- Eight four-byte stores.
- Four one-byte reads used as destination selectors.
- Four one-byte stores.

These counts exclude instruction fetches, implicit machine effects, and all reads or writes performed inside dependencies.

The explicit stores are not transactional. No rollback, compensating action, or restoration appears. A later failure can leave earlier stores observable.

### Aliasing and ordering

No non-aliasing guarantee is established. Word destinations may overlap one another, selector sources, or byte destinations. Overlapping accesses must retain the visible ordering above.

In particular:

- A word store may alter bytes read later as selectors.
- A byte store may alter a selector source used by a later read.
- Dependencies between reads may change selectors or other memory.
- The first three selector reads use the same calculated source address but need not return equal values.
- The fourth selector source is derived from result 057, not directly from Q.

A final-memory-only comparison is insufficient to characterize the routine where intermediate writes or selector reads are externally observable.

### Ownership and errors

No explicit allocation, release, ownership transfer, resource acquisition, or resource cleanup is visible outside the dependencies.

Passing a value to a dependency does not establish whether it is an address, handle, scalar, borrowed reference, or owned resource. Dependency ownership behavior remains unknown.

No explicit validation, status-based branching, retry, or error reporting appears. Faults, exceptions, nonreturning dependencies, and invalid targets can prevent later operations. Their handling is not shown.

## Dependencies

| Opaque dependency sites | Established interface use |
|---|---|
| F-D001 | Receives the input word; target depends on the ambient call base |
| F-D003 | Receives Q |
| F-D007, F-D012, F-D013, F-D018 | Return values participate in payload-store addresses or subsequent arguments |
| F-D023, F-D028, F-D033 | Return values participate in scaled-result stores |
| F-D035, F-D047, F-D059 | Produce opaque words forwarded to a following dependency |
| F-D040 | Produces the address operand for the Q word store |
| F-D045, F-D057 | Produce unsigned words used in byte-value and address calculations |
| F-D052, F-D064 | Produce values shifted for subsequent call arguments |
| F-D008, F-D019, F-D024, F-D029, F-D041 | Receive scaled address-related values |
| F-D014, F-D034, F-D036 | Receive the multiple arguments specified in the behavior section |
| F-D046, F-D048, F-D053, F-D058, F-D060, F-D065 | Receive the byte-phase values specified above |
| All remaining sites | Have no explicit arguments in the recovered expressions; their results are not explicitly consumed |

No dependency implementation is supplied. None can be assumed pure, nonthrowing, memory-preserving, global-preserving, or ownership-neutral. An ignored return value is not evidence of an ignored side effect.

The visible prefix contains no explicit conditional branches or loops. That observation does not constrain control flow inside dependencies.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results.

1. **Input forwarding.** If the input word is 0xA5A50123 and F-D001 is reached, its explicit argument is that same 32-bit pattern. No visible masking or shifting precedes it.

2. **Ambient unsigned shift.** If ambient word A is 0xFFC00000, T is 1,023. The second explicit word store writes 1,023 at starting address 1,059. This example establishes arithmetic only; it does not establish that an unaligned store there is permitted.

3. **Retained scaled result.** If result 023 is 3 and result 028 is 9, the corresponding stores write 12 at address 68 and then 12 at address 56. The latter value is not 36.

4. **Low-byte truncation.** If result 045 is 0x12345678 and the first selector is 5, the first byte store writes 0xE0 at address 22. F-D046 receives 0x48D159E0, not merely 0xE0.

5. **Selector reread.** If the first selector read returns 5 and the second returns 9, the first two byte destinations are 22 and 26. Reusing the first selector for both stores would contradict the visible read order.

6. **High-field extraction.** If result 045 is 0x12345678, the second stored byte is 0x23. Result 052 can affect the argument to F-D053 but does not supply this byte.

7. **Changed final selector source.** If result 057 is 4, the final selector is read at address 33. That read does not use the earlier Q-derived source unless the addresses happen to coincide.

8. **Interrupted execution.** If F-D028 does not return, the earlier store of S may already be observable, but the subsequent store addressed using result 028 is not reached in the visible sequence. No rollback is shown.

## Uncertainties

The following remain **documented_only**:

- The routine’s application-level purpose.
- Whether the recovered instruction sequence accurately represents executable original behavior.
- The actual calling convention and validity of the ambient operands.
- Dependency target identity, aliasing between targets, and dependency behavior.
- Whether dependency calls preserve any relevant registers, globals, memory, or resources.
- Memory layout, byte order, alignment policy, and validity of low-address accesses.
- Machine-level behavior for signed shifts, signed overflow, and overflowing signed address expressions.
- Whether the calculated accesses refer to ordinary RAM, device registers, or another address space.
- Behavior after the final visible dependency call.
- Whether the invalid-instruction boundary represents a real trap, incorrect decoding, embedded data, or another recovery failure.

A serialization or packed-field interpretation is plausible from the shifts and byte extraction, but it is inferred intent only and is not part of this contract.

## Evidence

Established static evidence comes solely from the supplied recovered representation: the declared widths, explicit call operands, consumed call results, memory-access widths, arithmetic expressions, visible ordering, and instruction-recovery warnings.

The dependency IDs in this document are analytical labels, not recovered symbols or verified target identities.

No implementation inspection beyond the supplied material, runtime execution, dependency validation, or memory-map verification has been performed. The acceptance examples are arithmetic consequences of the visible expressions under their stated conditions.
