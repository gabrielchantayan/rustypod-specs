<!-- Trusted provenance: F-8e0ff172ea1d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8e0ff172ea1d

## Purpose

F-8e0ff172ea1d performs an ordered sequence of indirect invocations, computed-address 32-bit stores, unsigned byte reads, and byte stores. The represented path ends in an unconditional loop containing further invocations and byte writes.

No application-level purpose is established. The evidence does not identify a record format, conventional object, device protocol, allocator, or ownership operation. Repeated address displacements alone do not establish fields or object boundaries.

This contract describes the supplied static representation. It does not guarantee that actual execution reaches the loop: an invocation may not return, a memory access may fail, and warnings about invalid instruction data and removed unreachable regions limit confidence in the recovered control flow.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | Supplied to the first indirect invocation. |
| Second argument | 32 bits | No explicit use in the represented body. |
| Third argument | Unsigned 32 bits | Logically shifted right by seven and supplied to an early invocation. |

The absence of an explicit use of the second argument does not prove that its incoming machine state is irrelevant. The recovered interface and implicit calling convention remain unresolved.

### Additional entry state

The represented execution also consumes:

- An incoming address-like base for the initial word store.
- An incoming 32-bit value for that store.
- An incoming unsigned 32-bit quantity used for scaling and extraction.
- An incoming control-transfer value used to form the first indirect target.

A later invocation sequence additionally supplies a secondary-register output used in two scaled expressions. Its meaning and calling convention are unknown.

For this contract:

- **Seed** denotes the incoming unsigned 32-bit quantity.
- **Scaled seed** denotes eight times the seed, as represented in the 32-bit computation.
- **Middle portion** denotes the seed logically shifted right by eight.
- **Upper portion** denotes the seed logically shifted right by sixteen.

These are descriptive names, not established fields.

### Output, completion, and errors

The represented interface has no return value and no normal return path.

If every reached invocation returns and every access succeeds, execution enters an unconditional loop. No success status, error status, validation failure, timeout, exception translation, or recovery path is represented.

A fault or nonreturning dependency can stop progression through the described sequence. That possibility is not an explicit error-handling contract.

### Preconditions

Traversal requires each reached indirect target to support the attempted control transfer and each reached memory address to support the specified access width. The body does not validate either condition.

No buffer extent, object type, alignment guarantee, or ownership token is supplied. Address validity, alignment behavior, and byte order depend on the execution environment.

## Behavior

### Widths and invocation notation

Word stores are 32 bits. Byte reads produce unsigned values from 0 through 255. Byte stores retain the low eight bits of the supplied value.

Unsigned right shifts are logical. Unsigned 32-bit operations have their represented width; however, several address and scaling expressions involve signed intermediates. The supplied representation does not establish a complete machine-level interpretation of signed overflow, signed left shifts, or conversion to addresses. Examples requiring wraparound are therefore conditional on low-32-bit machine arithmetic.

A **five-invocation chain** means an ordered group of five indirect invocations:

- The first receives the explicitly described arguments.
- The next four have no explicit arguments in the representation.
- Where specified, the fifth invocation supplies the chain’s retained result.

This is descriptive shorthand, not an identity or behavioral equivalence claim. “No explicit arguments” does not mean “no inputs”: dependencies may consume registers, memory, globals, or other implicit state. Each invocation remains observable even when its result is unused.

### Initial word stores

The first visible action writes the incoming 32-bit stored value at the incoming base plus 20 bytes. It precedes every represented invocation.

A five-invocation chain then begins with the first explicit argument. Its final result determines a word store whose destination is sixteen times that result plus 20 bytes and whose value is four times that result.

The next five-invocation chain receives four explicit arguments, in this order:

1. Scaled seed.
2. Scaled seed again.
3. Middle portion.
4. Third argument logically shifted right by seven.

After the chain returns, the operation writes the scaled seed as a word at scaled seed plus 20 bytes.

Another five-invocation chain begins with four times the preceding chain’s final result. After this chain, a secondary-register output determines a word store:

- Destination: secondary output multiplied by \(2^{26}\), plus 20 bytes.
- Value: secondary output multiplied by \(2^{24}\).

The evidence does not establish an independent contract for that secondary output.

### Result-dependent word stores and forwarding

The value formed by multiplication by \(2^{24}\) becomes the first argument of another five-invocation chain. The chain’s final result selects a destination at four times that result plus 20 bytes. The stored word remains the value derived from the secondary output.

A following five-invocation chain begins with four times that destination-selecting result. Define the **first scaled result** as four times this new chain’s final result. The operation stores the first scaled result at sixteen times the unscaled result plus 20 bytes.

A distinct forwarding sequence follows. Its first invocation receives the first scaled result. A second invocation produces a result that is passed to a third invocation. A fourth invocation produces another result, which begins a five-invocation chain. The chain’s final result selects a destination at four times that result plus 20 bytes; the stored word is the first scaled result.

Another five-invocation chain begins with four times this destination-selecting result. Define the **second scaled result** as four times its final result. The operation writes the second scaled result at sixteen times the unscaled result plus 20 bytes.

A second forwarding sequence has the same explicit argument/result relationships as the first, but begins with the second scaled result. Its fourth invocation’s result begins another five-invocation chain. This chain selects a word destination at four times its final result plus 20 bytes. The value written is the upper portion of the seed.

The next five-invocation chain begins with four times that destination-selecting result. It causes a word store with:

- Destination: sixteen times its final result, plus 20 bytes.
- Value: four times its final result.

The destination before adding 20 is called the **working base**.

Separately, the **selector base** is derived by clearing bit 31 of the second scaled result and logically shifting right by seven. Its represented range is 0 through \(2^{24}-1\), inclusive.

### Transition from word stores to byte stores

A five-invocation chain receives three explicit arguments:

- Working base.
- Working base again.
- Middle portion logically shifted right by seven.

After the chain returns, a word store writes the working base at working base plus 20 bytes. This has the same represented destination as the immediately preceding word store, but a different value expression. Dependencies intervene between the two stores, so they are not interchangeable or redundant.

Another five-invocation chain begins with four times the preceding chain’s final result. Its final unsigned result is the **first byte-producing result**.

The operation reads an unsigned byte at selector base plus one. It adds 17 to that byte value and writes the low byte of four times the first byte-producing result at the resulting address.

Thus, the selected destination is numerically between 17 and 272 inclusive. The operation uses the byte’s value as an absolute address component; no conventional pointer-table interpretation is established.

An invocation next receives four times the first byte-producing result in each of three explicit argument positions. A second invocation produces a result that begins a five-invocation chain.

After that chain, the operation freshly reads the byte at selector base plus one. At this newly read byte’s value plus 17, it stores the low eight bits of the first byte-producing result after clearing bits 30 and 31 and logically shifting right by 20.

The selector is not cached across these writes. Intervening dependencies or aliasing memory updates may cause the two selected destinations to differ.

### Byte clearing before the loop

A five-invocation chain begins with four times the immediately preceding chain’s final result. Its final unsigned result is the **second byte-producing result**.

The next direct memory accesses occur in this order:

1. A byte store at **the upper portion logically shifted right by seven, plus 28 bytes**. The stored value is the low byte of four times the second byte-producing result.
2. A fresh unsigned byte read at selector base plus one.
3. A zero-byte store at the freshly read byte’s value plus 17.

Because the upper portion is the seed logically shifted right by sixteen, the first destination is exactly the seed logically shifted right by twenty-three, plus 28. Its range is 28 through 539 inclusive. It is **not** the unshifted upper portion plus 28.

This ordering matters if the first destination aliases the selector address: the byte read follows the first store and can observe its effect.

An invocation then receives the upper portion logically shifted right by seven, equivalently the seed logically shifted right by twenty-three. Another invocation follows without explicit arguments.

A five-invocation chain next begins with the second byte-producing result masked to its low 30 bits and logically shifted right by six. After the chain, the operation again reads the selector byte at selector base plus one and writes zero at the read byte’s value plus 17.

Another five-invocation chain begins with four times the preceding chain’s final result. Its final unsigned result is masked to its low 30 bits and logically shifted right by twenty-five. The resulting **initial loop quantity** is between 0 and 31 inclusive.

A **separate selector address** is formed from four times the second byte-producing result, multiplied by 1024, plus one. This calculation involves a signed intermediate in the representation; its machine-address interpretation remains qualified by the arithmetic uncertainties.

The operation reads an unsigned byte at that separate selector address and writes zero at the read byte’s value plus 17.

An invocation receives twice the initial loop quantity. A following invocation produces the initial **loop-carried value**.

### Unconditional loop

Each represented iteration contains the following ordered phases:

- A five-invocation chain begins with the current loop-carried value and the current loop quantity logically shifted right by eight.
- A fresh unsigned byte is read at the separate selector address. Zero is written at that byte’s value plus 17.
- A second five-invocation chain begins with four times the first chain’s final result.
- The loop-carried value becomes four times the second chain’s final result.
- The loop quantity becomes that same final result shifted left by twenty-six.

No termination condition is represented.

On the first iteration, the second explicit argument of the loop-head chain is necessarily zero: the initial loop quantity is at most 31.

Under low-32-bit shift semantics, later values of that argument depend only on the low six bits of the preceding iteration’s final result, with eighteen low zero bits.

The target-forming state at the loop head changes during the body. The first iteration’s head target is not represented identically to subsequent iterations’ head target. A contract treating all loop-head invocations as one fixed dependency is unsupported.

## Side effects

Established direct effects are ordered word stores, unsigned byte reads, byte stores, indirect control transfers, and potentially unbounded repetition.

The body does not explicitly read the prior contents of its word-store destinations. Unknown dependencies may read or modify those locations.

No allocation, deallocation, reference-count change, ownership transfer, resource acquisition, or cleanup is established. Dependencies must not be assumed to preserve globals or to acquire, retain, or release memory.

Aliasing is not excluded. In particular:

- A word or byte store may change memory later read as a selector.
- A selector-derived destination may coincide with another accessed location.
- Dependencies may modify selector memory between direct reads.
- A dependency may observe an earlier store before a later store replaces it.

Repeated zero writes cannot be collapsed into one final write. Their selected addresses may differ, and invocations intervene.

The recurring displacements of 20, one, 17, and 28 bytes establish address arithmetic only, not a memory layout with named fields. No atomicity, synchronization, or memory-mapped-device semantics is established.

## Dependencies

The only supplied opaque function identity is **F-8e0ff172ea1d**.

All indirect dependencies are **documented_only unresolved dependencies**. No opaque function IDs are supplied for them; assigning additional identities would invent information. Their invocation positions and explicit data relationships are described above without exposing private source symbols.

Unresolved dependency properties include:

- Identity and whether different sites resolve to the same implementation.
- Calling convention and implicit inputs.
- Return interpretation beyond the represented uses.
- Secondary-register output behavior.
- Memory and global-state effects.
- Ownership or lifetime effects.
- Whether an invocation returns, blocks, faults, or transfers control elsewhere.

No result being unused makes its producing invocation removable.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes that execution reaches the relevant point, preceding dependencies return as required, and the relevant accesses are valid. Wraparound-dependent examples additionally require low-32-bit machine arithmetic.

1. **Initial action.** An incoming base of 4096 and stored value of 7 produce a 32-bit write of 7 at address 4116 before any represented invocation.

2. **Early extraction.** With seed 4660 and third argument 511, the early four-argument invocation receives 37280, 37280, 18, and 3. The upper portion is zero.

3. **Result-selected word store.** If the first five-invocation chain’s final result is 3, the following direct word store writes 12 at address 68.

4. **Fresh selector values.** A selector read yielding 5 selects address 22. A later read yielding 9 selects address 26. Reusing the first destination would not preserve the represented behavior.

5. **Byte truncation.** A first byte-producing result of 65 gives a first selected byte value of 4, because four times 65 is 260.

6. **Extracted-byte truncation.** A first byte-producing result of \(2^{28}\), masked to 30 bits and shifted right by twenty, yields 256. The associated byte store writes zero after truncation.

7. **Corrected pre-loop destination.** With seed \(2^{23}\), the upper portion is 128. Shifting that upper portion right by seven gives one, so the first byte store in “Byte clearing before the loop” targets address 29—not address 156. Its value is the low byte of four times the second byte-producing result.

8. **Destination boundaries.** Seeds from zero through \(2^{23}-1\) select address 28 for that store. The maximum unsigned 32-bit seed selects address 539. These boundaries follow from shifting the seed right by twenty-three before adding 28.

9. **Store-before-read aliasing.** If that pre-loop byte destination equals selector base plus one, the following selector read occurs after the store. If the stored byte is 12 and no other effect intervenes, the ensuing zero write targets address 29.

10. **Loop entry and noncompletion.** The first loop-head chain receives zero as its second explicit argument. If every subsequent invocation returns and access succeeds indefinitely, execution continues without a normal return.

## Uncertainties

### Established static evidence

The supplied representation establishes the explicit argument uses, arithmetic relationships, direct access widths, invocation ordering, repeated selector reads, state updates, and unconditional loop described here.

The corrected pre-loop destination follows two unsigned right shifts: sixteen bits to form the upper portion, then seven more bits before adding 28.

### Inferred intent

No domain-level intent is adopted. Repeated invocation and store patterns may suggest structured processing, but that inference does not establish a higher-level functional contract.

### Documented-only unknowns

The following remain **documented_only**, rather than guessed:

- Whether the entry boundary and recovered three-argument interface are complete.
- Whether invalid instruction data or removed regions hide reachable behavior.
- Exact signed-overflow, signed-shift, address-width, alignment, and byte-order rules.
- Validity of low numerical addresses.
- Whether accessed memory overlaps code, globals, device registers, or other accessed regions.
- Implicit dependency inputs, outputs, and preservation rules.
- Dependency ownership, memory, and global-state effects.
- Whether actual executable behavior matches the represented nontermination.

## Evidence

This contract uses only the supplied static representation and opaque identity mapping. The prior draft and feedback are not independent evidence.

The supplied representation shows 32-bit stores, unsigned byte reads, byte stores, computed indirect invocations, result-dependent addresses, and an unconditional loop. It also carries warnings about invalid instruction data and removed unreachable regions.

For the corrected destination, the value retained from the seed is its unsigned upper sixteen-bit portion. The later byte-store address shifts that retained value right by seven before adding 28. This supports both the seed-right-shift-by-twenty-three equivalence and the inclusive destination range of 28 through 539.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed execution results.
