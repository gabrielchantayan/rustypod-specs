<!-- Trusted provenance: F-e3a84a950aae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3a84a950aae

## Purpose

F-e3a84a950aae performs a fixed sequence of indirect calls interleaved with byte, halfword, and word memory accesses. It forwards input-derived values, forms addresses from dependency results, propagates captured words, updates shared storage, clears selected storage, and finally attempts a call to address zero.

The established behavior is the ordered sequence of calls and memory effects. A higher-level purpose—such as initialization, allocation, or linkage management—is not established.

This contract distinguishes:

- **Established static evidence:** operations, widths, conversions, and ordering visible in the supplied representation.
- **Inferred intent:** possible interpretations that are not behavioral requirements.
- **Unknowns:** unresolved details marked `documented_only`.

No runtime tests have been performed.

## Interface

### Explicit inputs

The visible interface has four 32-bit argument slots.

| Input | Interpretation | Explicit use |
|---|---|---|
| First argument | Opaque 32-bit value | No explicit use |
| Second argument | Opaque 32-bit value | Forwarded to F-U01 |
| Third argument | Signed 32-bit integer | Shifted left by 11 and separately by 12 |
| Fourth argument | Unsigned 32-bit integer | Logically shifted right by 23 for F-U03 |

Absence of an explicit use of the first argument does not establish that dependencies cannot observe its incoming register state.

### Additional entry state

Execution also depends on:

- An incoming link-related signed 32-bit value used to form the first call target.
- An incoming signed 32-bit address used for a byte read at offset 1.
- An incoming unsigned 32-bit value whose bits 22–29 supply an early byte store.
- Shared storage and memory reached through calculated addresses.
- A pointer associated with the final byte-returning dependency, used for a byte clear.

None of these prerequisites is validated.

### Outputs and termination

There is no explicit return value or error status. The final visible action is F-U98, an attempted call to address zero with three explicit zero arguments. If it returns normally, the enclosing function returns.

A fault or nonreturning dependency prevents subsequent operations. Completed effects are not rolled back.

### Widths and arithmetic

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Addresses are formed through the displayed 32-bit integer or pointer expressions; a complete platform address model is unavailable.

Byte stores retain the low eight bits. Unsigned right shifts are logical. Splitting a 64-bit result into low and high 32-bit portions is a numeric operation, independent of memory byte order.

Signed and unsigned calculations must be distinguished:

- Signed left shifts, multiplications, **and additions** are not assigned an implicit wrapping interpretation. Their behavior outside valid, representable cases remains `documented_only`.
- Every intermediate signed operation matters. A representable product does not make a subsequent overflowing addition valid.
- Explicit unsigned 32-bit shifts and additions operate within unsigned 32-bit arithmetic. This does not retroactively make an earlier signed calculation unsigned.
- Conversion to an address does not establish validity, mapping, alignment, or permission.

In particular, F-U70’s low and high halves are each interpreted as signed 32-bit integers before their subsequent operations. The high-half pattern `FFFFFFFF` therefore denotes −1 for the displayed signed address calculation. A negative low half does not acquire a defined unsigned left-shift behavior.

## Behavior

The following phases specify observable order, conditional on every preceding operation completing. Dependency labels identify call sites, not necessarily distinct implementations.

### 1. Initial calls and selector-based write

F-U01 receives the second argument. F-U02 returns an opaque word. F-U03 receives that word followed by the fourth argument logically shifted right by 23.

F-U04, F-U05, and F-U06 follow. F-U07 then returns a signed 32-bit result.

Before the next call, the function reads a byte at the incoming signed address plus 1. This addition is subject to the signed-arithmetic qualification above. The byte is zero-extended and retained as the **early selector base**.

At early selector base plus 17, the function stores the low byte of the incoming unsigned value shifted right by 22. The destination lies in 17–272.

F-U08 receives the signed F-U07 result shifted left by two.

### 2. First shared-word update

F-U09, F-U10, and F-U11 precede F-U12. F-U13 receives two copies of the signed F-U12 result shifted left by two.

F-U14 returns an unsigned 32-bit value, the **captured control word**. The function writes it to **shared word A**, then passes the captured value to F-U15. F-U16 follows.

The function next reads a full word at early selector base plus 116 and writes that word’s low byte at early selector base plus 1.

The early selector base is retained across the calls; its original selector byte is not reread. Similarly, later uses of the captured control word use the saved value, not a fresh read of shared word A. Dependencies may have changed either shared storage or addressed memory.

### 3. Third-argument forwarding and first scaled base

F-U17 receives the signed third argument shifted left by 11.

F-U18 returns a 64-bit quantity. F-U19 receives:

1. Its low 32-bit portion converted to a signed 32-bit integer.
2. Its high 32-bit portion converted to a signed 32-bit integer.
3. The captured control word logically shifted right by 23.

F-U20, F-U21, and F-U22 precede F-U23. The signed F-U23 result multiplied by four becomes the **first scaled base**.

The function reads a word at first scaled base plus 116, writes its low byte at first scaled base plus 1, then calls F-U24 with the first scaled base.

Both address additions are signed calculations. Their validity is separate from the validity of the multiplication.

### 4. Fresh read and four-argument forwarding

F-U25, F-U26, and F-U27 precede F-U28.

The signed F-U28 result is shifted left by two, then stored as an unsigned 32-bit value called the **saved scaled selector**. Assignment to unsigned storage does not resolve invalid behavior in the preceding signed shift.

The function again reads a word at first scaled base plus 116 and writes its low byte at first scaled base plus 1. This is a fresh read after intervening dependencies.

F-U29 receives, in order:

1. The signed third argument shifted left by 12.
2. The saved scaled selector.
3. Another copy of the saved scaled selector.
4. The signed F-U28 result shifted left by 18.

F-U30 returns an opaque word passed to F-U31. F-U32 returns another opaque word passed to F-U33.

### 5. Capturing and transferring a word

F-U34, F-U35, and F-U36 precede F-U37. Its signed result multiplied by four forms a new signed base.

A full unsigned word is read at that base plus 116 and retained as the **transfer word**. Its low byte is written at that base plus 1. F-U38 then receives the base.

F-U39, F-U40, and F-U41 precede F-U42. The function writes the full transfer word at the signed address calculated by multiplying the F-U42 result by 16 and adding 20.

The multiplication and addition must each be representable. For example, a result of `07FFFFFF` produces the representable product `7FFFFFF0`, but adding 20 exceeds the signed 32-bit maximum. No wrapped destination is specified for that case.

F-U43 receives the unsigned saved scaled selector logically shifted right by two and then shifted left by 16, within 32-bit unsigned arithmetic. This expression must not be replaced by an assumed reconstruction of the original F-U28 result.

### 6. Further transfer and link installation

F-U44 returns a word passed to F-U45. F-U46 returns a word passed to F-U47. F-U48, F-U49, and F-U50 follow.

F-U51 returns a signed 32-bit value. The function writes the transfer word at four times that value plus 20, then calls F-U52 with four times the value. The address uses signed multiplication and addition.

F-U53, F-U54, and F-U55 precede F-U56. Its signed result shifted left by two becomes the **link value**.

The link value is written at the saved scaled selector, masked with `FFFFFFFC`, plus 20. Unlike the preceding signed address calculation, this destination expression uses unsigned 32-bit arithmetic. The mask is explicit even though a valid preceding shift by two produces zero low bits.

F-U57 receives two copies of the link value.

### 7. Clearing a word and installing a signed paired result

F-U58 returns a word passed to F-U59. F-U60 returns a word passed to F-U61. F-U62, F-U63, and F-U64 follow.

F-U65 returns a signed index. A full zero word is written at four times that index plus 20. F-U66 receives four times the index. Both multiplication and address addition are signed.

F-U67, F-U68, and F-U69 precede F-U70, which returns a 64-bit quantity. Its portions have distinct signed uses:

- The low 32-bit portion is converted to a signed 32-bit integer **before** being shifted left by two. The result is the **paired link value**.
- The high 32-bit portion is converted to a signed 32-bit integer, the **paired destination index**, before multiplication by four.

The function writes the paired link value at four times the paired destination index plus 20, using signed multiplication and addition.

Thus, a high-half pattern of `FFFFFFFF` supplies index −1 and, with otherwise valid operations, destination 16. Conversely, a low half interpreted as negative does not justify substituting a wrapping unsigned shift; that case remains unresolved.

After this store, the function reads **shared word B** into a signed 32-bit captured address base. F-U71 then receives the paired link value, the signed paired destination index, and two more copies of the paired link value.

### 8. Dereferencing the captured shared base

F-U72, F-U73, and F-U74 precede F-U75.

The paired link value is written at four times the signed F-U75 result plus 20. The function then reads a full unsigned word at the captured shared-word-B base plus 20, retaining it as the **late captured word**. F-U76 receives four times the F-U75 result.

These signed address additions require representable results. The shared-word-B base is not reread: intervening dependencies may alter shared word B or the memory at the captured address without changing the saved base itself.

### 9. Two byte-selected destinations

F-U77, F-U78, and F-U79 precede F-U80. Its 64-bit result supplies an unsigned **late low half** and an opaque **late high half**.

The function reads a byte at the transfer word logically shifted right by 23, plus 1. It zero-extends that selector byte and writes the low byte of the late captured word at selector plus 17.

The selector-read address lies in 1–512; the store address lies in 17–272.

F-U81 receives three copies of the late high half, followed by the unsigned late low half shifted left by two.

F-U82 returns a word passed to F-U83. F-U84, F-U85, and F-U86 precede F-U87.

The function performs a second selector operation. It masks the late low half with `3FFFFFFF`, logically shifts right by 21, adds 1, and reads a byte at that address. At that byte’s zero-extended value plus 17, it stores the low byte of the late captured word logically shifted right by 22.

Again, the selector-read address lies in 1–512 and the destination in 17–272. The stored bits are bits 22–29 of the late captured word.

F-U88 receives the signed F-U87 result shifted left by two.

### 10. Final calls, shared composition, and clears

F-U89, F-U90, and F-U91 precede F-U92. F-U93 receives two copies of the signed F-U92 result shifted left by two.

F-U94 follows. F-U95 returns a word passed to F-U96. F-U97 returns a byte; the representation also exposes an additional pointer associated with this call.

The remaining direct effects occur in this order:

1. Store a zero byte through that additional pointer.
2. Read the shared 24-bit quantity used in the final composition.
3. Write shared word A with that quantity in bits 8–31 and the F-U97 byte in bits 0–7.
4. Store a zero halfword at the unsigned late low half masked with `3FFFFFFF` and logically shifted right by seven.
5. Attempt F-U98 with three explicit zero arguments.

The halfword destination ranges from 0 through 8,388,607 and need not be aligned. The shared composition is numeric; its byte layout depends on the platform.

## Side effects

### Memory layout and access widths

The body directly performs selector-byte reads, byte stores, full-word reads and writes, a shared 24-bit read, and a halfword clear.

Offsets 1, 17, 20, and 116 are byte offsets. They establish neither object boundaries nor structure types.

A full-word read followed by truncation is required where described. It is not interchangeable with a byte read at the same address: byte order, access permissions, and memory-mapped side effects can distinguish them.

### Aliasing and ordering

No nonaliasing guarantee is established. Calculated addresses may overlap each other, shared storage, or dependency-visible memory. The supplied material also warns that shared symbols may overlap smaller symbols.

Consequently:

- Repeated offset-116 reads may return different values.
- A store may affect a later selector read or shared read.
- The additional-pointer byte clear precedes the final shared 24-bit read.
- Captured values remain distinct from later reads of their original storage.

No atomicity, synchronization, memory barriers, or volatility guarantees are established.

### Ownership and failure

No direct allocation, release, ownership transfer, or lifetime extension is visible. Passing a value to a dependency establishes no ownership contract.

There are no explicit checks for invalid addresses, overflow, misalignment, dependency failure, or nonexecutable targets. Fault handling belongs to the unresolved execution environment.

## Dependencies

F-U01 through F-U98 are opaque call-site IDs assigned by this contract. They do not assert 98 distinct function identities.

F-U01’s target depends on incoming link-related state. Other nonzero targets are formed from embedded location expressions. F-U98 targets address zero. Signed target-address additions are subject to the arithmetic qualifications above.

Dependencies without explicit argument expressions may still observe ABI-visible registers or other state. Their implementations and contracts are unavailable.

No dependency is assumed to:

- Preserve globals, memory, or undocumented registers.
- Validate, retain, allocate, or release memory.
- Return normally.
- Provide recognizable failure reporting.

The additional pointer used after F-U97 is not treated as an ordinary documented return value or as proof of ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes that preceding execution completes, necessary accesses are supported, and relevant arithmetic is valid unless the example specifically identifies an unresolved boundary.

1. **Unsigned extraction.** Fourth argument `FFFFFFFF` makes F-U03’s second explicit argument 511.

2. **Early selector.** A selector byte of 40 and incoming unsigned value `ABCDEF01` produce a byte store of `AF` at address 57.

3. **Full-word truncation.** An offset-116 read yielding `12345678` produces a corresponding byte store of `78`, regardless of the source word’s byte order.

4. **Fresh read.** If intervening dependencies change the first scaled base’s offset-116 word from `11111122` to `33333344`, its two byte stores contain `22` and `44`, respectively.

5. **Scale and addition.** If F-U42 returns 3, the transfer-word destination is 68. If it returns `07FFFFFF`, the product fits signed 32 bits but the addition does not. The contract does not prescribe a store at wrapped address `80000004`.

6. **Positive paired halves.** F-U70 low half 7 and high half 9 produce paired link value 28, stored at address 56.

7. **Signed high half.** F-U70 low half 7 and high-half pattern `FFFFFFFF` produce paired link value 28 and signed destination index −1. The store targets address 16, and F-U71’s second argument is signed −1.

8. **Negative low half.** F-U70 low-half pattern `FFFFFFFF` is interpreted as signed −1 before shifting. A paired link value of `FFFFFFFC` is not an established requirement; the signed shift remains `documented_only`.

9. **Final composition.** A final shared 24-bit value of `123456` and F-U97 byte `AB` produce shared word A value `123456AB`. The 24-bit read occurs after the pointer-based byte clear.

10. **Final halfword.** Late low half `00000100` selects address 2 for the zero-halfword store, before the zero-address call.

## Uncertainties

The following remain `documented_only`:

- Fidelity of the supplied representation to executable behavior.
- Calling convention, hidden inputs, and register preservation.
- Dependency identities, side effects, return guarantees, and ownership behavior.
- Whether 64-bit results represent integers, paired registers, or another ABI convention.
- Memory mapping, permissions, alignment support, and byte order.
- Behavior of invalid signed shifts and overflowing signed multiplications or additions, including address and call-target calculations.
- Pointer conversion and address interpretation beyond the displayed expressions.
- Exact shared-storage overlap and computed-address aliasing.
- Origin, validity, and lifetime of the pointer used after F-U97.
- Whether address zero executes, traps, or has special platform meaning.

Repeated scaled stores may suggest linkage or table manipulation. That is inferred intent only, not an established data-structure invariant.

## Evidence

This contract is based solely on the supplied static representation: declared widths, signed and unsigned conversions, call arguments, address expressions, and ordered memory accesses.

The representation contains no explicit branches, loops, validation paths, rollback, or error returns. Its apparent straight-line execution remains conditional on every preceding operation completing.

No dependency implementations, platform memory map, ABI documentation, or runtime observations were supplied. No runtime tests have been performed.
