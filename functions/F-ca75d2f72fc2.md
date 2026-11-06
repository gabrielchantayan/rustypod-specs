<!-- Trusted provenance: F-ca75d2f72fc2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca75d2f72fc2

## Purpose

F-ca75d2f72fc2 conditionally processes an address, examines an eight-byte preceding header, updates seven 32-bit accounting destinations relative to a captured context, and invokes dependencies in a defined order.

The operations are consistent with allocation-release accounting. This is **inferred intent**, not an established ownership contract. The supplied representation does not establish whether any dependency releases memory, invalidates the input, or provides synchronization.

A zero input returns immediately. A nonzero input initiates preparation, context capture, header validation, accounting updates, and final dependency calls.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Input and return

The function accepts one integer-valued argument used as an address in an apparent 32-bit address model. Denote that address by **P**.

- **P = 0:** supported no-operation input.
- **P ≠ 0:** explicitly input-relative memory accesses address an eight-byte header immediately preceding P.
- No explicit result value is supplied.

Normal completion of the nonzero path requires the accessed memory to be usable and each dependency on that path to return. These are conditions for completion, not checks implemented by the function.

The explicitly input-relative accesses target the preceding header. This does **not** mean the function cannot directly access bytes at or after P. Its context-relative accounting destinations are not checked for overlap with the header or payload. For example, a captured context equal to P makes the final accounting destination P + `0x008`.

### Header layout

| Address relative to P | Width | Established use |
|---|---:|---|
| −8 | 32 bits | Validation word and source of masked accounting amounts |
| −4 | 16 bits, unsigned | First accounting index |
| −2 | 16 bits, unsigned | Second accounting index |

Both header indices are zero-extended before multiplication by four. Their individual scaled displacements range from zero through `0x3fffc`.

Byte order, alignment requirements, accessible allocation extent, and valid application-level index ranges are not established.

### Context-relative destinations

Let **C** be the context base captured after the first dependency returns. The following are address formulas, not declarations of distinct fields or arrays.

| Destination | Width | Direct update |
|---|---:|---|
| C + `0x224` + 4 × first header index | 32 bits | Subtract retained masked amount |
| C + `0x30c` | 32 bits | Subtract the same amount |
| C + `0x0e0` + 4 × second header index | 32 bits | Subtract the same amount |
| C + `0x220` | 32 bits | Subtract the same amount |
| C + `0x310` + 4 × result of F-09dcb55e8e0a | 32 bits | Decrement by one |
| C + `0x390` | 32 bits | Decrement by one |
| C + `0x008` | 32 bits | Subtract a freshly read masked amount |

No visible bounds or disjointness checks protect these destinations. They can coincide with one another, overlap the preceding header, or fall at or after P unless an external invariant excludes those cases.

### Outcomes and errors

The visible outcomes are:

- Immediate normal return for zero input.
- Normal return after the nonzero path completes.
- Invocation of F-a817d8d58c10 when validation rejects the header; this dependency is marked non-returning.
- Faults or other outcomes arising from invalid accesses, arithmetic interpretation, or dependency behavior.

There is no visible error return, local exception recovery, or rollback.

## Behavior

### Zero-input path

If P is zero, the function returns before any dependency invocation, context-source read, header read, or accounting write.

This describes the visible function body, not activity performed by its caller or surrounding runtime.

### Preparation and context capture

For a nonzero input, the first visible action is a call to F-2fd8826557a4 with no explicit arguments.

After that call returns, the function reads the global context source and captures C. It uses this captured value for subsequent direct context-relative accesses and explicit context arguments.

Consequences:

- Changes made by F-2fd8826557a4 can affect which context is captured.
- Later changes to the global context source do not themselves replace C.
- Capturing C does not freeze the memory it addresses.
- No validation of C is visible.

F-2fd8826557a4 is not established to preserve the header, globals, or context memory, nor is it established to return normally.

### Header validation

After capturing C, the function checks the 32-bit word at P − 8.

The supplied representation expresses a short-circuit check:

- If the first validation read is zero, validation rejects immediately.
- Otherwise, a subsequent read checks bit zero; a set bit rejects the header.

Thus the representation does not use one retained header snapshot for both validation conditions. On the zero-first-read path, the second validation read is skipped.

Rejection invokes F-a817d8d58c10 without explicit arguments. This dependency is annotated as non-returning. Before that invocation, preparation and context capture have occurred, but no direct accounting write has occurred.

For a stable header, validation accepts exactly the nonzero, even 32-bit words. Acceptance does not imply a positive masked amount.

If the header changes between validation reads, the two observations can disagree. In particular, a nonzero first observation followed by zero on the bit check does not trigger either rejection condition. No coherent-snapshot or concurrent-access guarantee is established.

### First accounting phase

After successful validation, the function calls F-c0fc6d033caa with C.

Only after this dependency returns does the function obtain the indices and amount used in the first accounting phase. In the visible sequence:

1. It reads the unsigned 16-bit index at P − 4 and determines the first indexed destination.
2. It reads the 32-bit word at P − 8 and masks it with `0x3ffffffc`.
3. It subtracts that retained amount from the first indexed destination.
4. It subtracts the same amount from C + `0x30c`.
5. It reads the unsigned 16-bit index at P − 2 and determines the second indexed destination.
6. It subtracts the retained amount from that destination.
7. It subtracts the retained amount from C + `0x220`.

Each destination update reads the destination’s current 32-bit value and writes the subtraction result.

The mask retains bits 2 through 29 and excludes bits 0, 1, 30, and 31. The resulting amount is a multiple of four, between zero and `0x3ffffffc` inclusive.

The same retained amount supplies all four subtractions. Later changes to the header do not change that retained amount. Conversely, validation does not preserve the header value or either index for this phase: F-c0fc6d033caa runs before their accounting reads.

The second index is read after the first two accounting writes. If either write overlaps that index, the later index read can observe the changed bytes.

No visible underflow check, saturation, or failure branch surrounds these updates.

### Count changes and final amount update

After the four initial subtractions, the function calls F-09dcb55e8e0a without explicit arguments.

Its returned integer determines the destination C + `0x310` + four times that result. The function then:

- Reads and writes that destination, decrementing it by one.
- Reads and writes C + `0x390`, decrementing it by one.
- Updates C + `0x008` by subtracting a newly read header word masked with `0x3ffffffc`.

The final amount is not the retained amount from the first accounting phase. The header is read again after F-09dcb55e8e0a and both count updates. Dependency effects or aliasing accounting writes can therefore make the final amount differ.

No revalidation occurs before this final amount is used.

The supplied representation does not establish the relative evaluation order of the final header read and the destination read at C + `0x008`. Both contribute to the ensuing subtraction and write, which follow the two count decrements.

There is no visible range check on the dependency result and no zero check before either decrement.

### Final dependency calls

After all seven direct accounting updates, the function invokes:

1. F-cc6c9b4a0f68 with C and P − 8.
2. F-e586bff64fb7 with C.

The second argument to F-cc6c9b4a0f68 is the header address, not P. No direct header access follows that call in the visible body.

If both calls return, the function returns normally.

### Arithmetic and aliasing

All seven accounting stores are 32 bits wide. No explicit overflow or underflow handling is present.

Under ordinary 32-bit machine arithmetic, subtraction results wrap modulo 2³²; decrementing zero would store `0xffffffff`. However, some expressions in the supplied representation use signed integer types. Exact machine-level overflow behavior is not independently established, and the representation should not be treated as a portable-language signed-overflow contract.

The header-index multiplication fits within 32 bits: its maximum is `0x3fffc`. Adding that displacement and a fixed offset to C can still overflow the address model. Arithmetic involving the result of F-09dcb55e8e0a can also involve negative values or overflow. No protection is visible; the valid domain remains undocumented.

The accounting updates are sequential, not simultaneous. When destinations coincide, a later update operates on the value then present. When destinations overlap header bytes, earlier accounting writes can affect subsequent header or index reads.

## Side effects

### Direct memory effects

On a normally completing nonzero path, the visible body performs seven 32-bit accounting writes:

- Four subtractions using one retained masked amount.
- Two decrements.
- One subtraction using a fresh masked amount.

No write is explicitly addressed as a header-relative or payload-relative operation. Nevertheless, the accounting writes themselves can modify header or payload bytes if their computed destinations overlap those regions. There is no unconditional guarantee that the header or bytes at or after P remain unchanged.

On the zero-input path, none of these accesses occurs.

### Dependency effects and ownership

The dependencies may have additional effects not established by the supplied representation. They cannot be assumed to:

- Preserve globals, context contents, or the header.
- Acquire or release synchronization.
- Own, retain, release, or invalidate memory.
- Return normally, except as a condition for describing subsequent operations.

No independent allocation or deallocation operation is identifiable in this function beyond its dependency calls. Whether the input remains usable after completion is **documented_only**.

### Partial progress and concurrency

No rollback is visible. If a later access faults or a dependency does not return, prior writes are not locally restored.

The representation establishes neither atomic read-modify-write operations nor an atomic accounting transaction. It also establishes no safe concurrent-access protocol. Repeated reads must not be replaced in this contract by an immutable conceptual header.

## Dependencies

| Opaque function ID | Explicit inputs | Established role |
|---|---|---|
| F-2fd8826557a4 | None | Runs first on the nonzero path, before context capture |
| F-a817d8d58c10 | None | Invoked on validation rejection; marked non-returning |
| F-c0fc6d033caa | Captured context C | Runs after validation and before accounting index and amount reads |
| F-09dcb55e8e0a | None | Runs after four subtractions; integer result selects a decrement destination |
| F-cc6c9b4a0f68 | C; header address P − 8 | Runs after all direct accounting writes |
| F-e586bff64fb7 | C | Final visible dependency call |

Normal nonzero-path invocation order is:

F-2fd8826557a4 → F-c0fc6d033caa → F-09dcb55e8e0a → F-cc6c9b4a0f68 → F-e586bff64fb7.

These positions and explicit arguments do not establish dependency internals, synchronization, ownership, or preservation guarantees.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

### Zero input

Given P = 0, the function returns without calling a listed dependency or reading the context source or header.

### Stable rejected header

Given a nonzero input and a stable zero header word, preparation runs, C is captured, and F-a817d8d58c10 is invoked. No direct accounting write precedes that invocation.

A stable header word of `0x00000005` also causes rejection because its low bit is set.

### Ordinary stable accounting

Assume all accessed memory is usable, accounting destinations are mutually disjoint and disjoint from the header, dependencies return without changing relevant memory, and the result index is valid.

For a stable header word of `0x00000042`:

- Validation succeeds.
- The retained amount is `0x40`, or 64.
- Each of the first four destinations decreases by 64.
- Both count destinations decrease by one.
- C + `0x008` decreases by 64.

These decreases presume values sufficient to avoid underflow; otherwise the arithmetic qualification above applies.

### Accepted zero amount

Under the same assumptions, a stable header word of `0x40000000` is nonzero and even, but masks to zero.

The five subtraction writes still occur, leaving their destination values numerically unchanged. Both count decrements and all remaining dependencies still occur. This is not the zero-input path.

### Different amounts between phases

Suppose the first accounting amount read produces 64, and F-09dcb55e8e0a changes the header so that its final masked amount is 128.

Absent other relevant effects or aliasing:

- The first four subtractions use 64.
- The subtraction at C + `0x008` uses 128.
- No second validation intervenes.

### Coincident accounting destinations

A first header index of 58 selects C + `0x30c`, because `0x224` + 4 × 58 equals `0x30c`.

With retained amount 64 and no intervening relevant effects, that destination undergoes two successive subtractions of 64. It must not be treated as receiving only one subtraction.

### Accounting write into the payload region

Assume C = P, accessible destinations, stable header `0x00000042`, both header indices zero, a result index of zero, and dependencies that return without relevant mutations.

The final subtraction targets P + `0x008`. If that 32-bit payload-region value initially equals 256, it becomes 192. The input-relative reads still target only the preceding header; that fact does not prevent a context-relative write into the payload region.

### Accounting write into the header

Assume C = P − `0x10`, both header indices zero, a result index of zero, accessible destinations, and dependencies without relevant mutations.

The final destination C + `0x008` is P − 8, the header word itself. With an initial stable header of `0x00000042`, no earlier accounting destination in this example overlaps the header. The final update reads the header amount as 64 and changes the header word from 66 to 2.

F-cc6c9b4a0f68 consequently receives the address of a header already modified by a direct accounting write.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention, byte order, alignment, and machine arithmetic semantics.
- Required memory extents and permitted address ranges.
- Valid application-level ranges for either header index and the dependency-returned index.
- Semantic units, names, and invariants of the accounting destinations.
- Meanings of header bits beyond their observed validation and masking uses.
- Any external prohibition on accounting-to-accounting, accounting-to-header, or accounting-to-payload overlap.
- Dependency effects on memory, globals, ownership, synchronization, and failure behavior.
- Whether successful completion releases or invalidates any allocation.
- The actual failure mechanism behind the non-returning rejection dependency.
- Exact executed memory-read behavior beyond the operations expressed by the supplied static representation.

## Evidence

**Established static evidence:** The supplied representation supports the zero-input return, preparation-before-capture ordering, validation conditions, header access widths, mask, accounting address formulas, ordered seven writes, and final dependency arguments. No visible check establishes non-aliasing between those destinations and the header or payload.

**Inferred intent:** The preceding header, subtractive accounting, and count decrements are consistent with allocation-release bookkeeping. This interpretation does not establish ownership transfer or memory release.

**Not established:** Runtime validity, synchronization, actual dependency behavior, memory lifetime, and external layout invariants. No runtime tests have been performed, and no execution results are claimed.
