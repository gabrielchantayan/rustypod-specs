<!-- Trusted provenance: F-eb768adac21a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb768adac21a

## Purpose

F-eb768adac21a is a stateful, non-returning routine that repeatedly interleaves indirect dependency calls with byte and 32-bit memory accesses.

The supplied static representation establishes an initialization sequence followed by an unconditional loop. It does not establish an application-level purpose, a conventional object model, or the meaning of the affected memory. In particular, the accesses must not be characterized as allocation, device control, serialization, or reference management without additional evidence.

The principal observable behavior is the order of dependency invocations, the values explicitly supplied to them, and the memory accesses between those invocations. Dependencies are opaque and may themselves have effects not represented here.

## Interface

### Invocation and completion

The displayed interface has no declared arguments and no declared return value. Nevertheless, its behavior depends on incoming machine state:

- An incoming control/link value determines the first indirect call target.
- An incoming byte supplies the first explicit memory write.
- An incoming 32-bit base supplies the address of the first byte-selector read.
- Two further incoming 32-bit values supply an early bit-field value and a shifted dependency argument.

These are implicit inputs, not evidence of a stable public calling convention. Their register placement and the legitimacy of an ordinary function call are **documented_only**.

There is no reachable normal return in the supplied representation. After initialization, the loop executes at least once if control reaches it and then repeats without an explicit exit condition. A dependency that fails to return, a memory fault, or another external interruption can prevent further progress.

### Data widths and descriptive notation

The following descriptive terms identify values by their role, not by implementation identifiers:

- **Word**: a 32-bit quantity.
- **Byte**: an 8-bit quantity.
- **Pair result**: a displayed 64-bit result, consumed as low and high 32-bit halves.
- **Selector base**: an address whose byte at offset 1 is read.
- **Selected-byte write**: read one unsigned byte at `selector base + 1`, then write one byte at the absolute address `read byte + 17`.
- **Selected-word write**: write a word at the absolute address `previously read selector byte + 20`.

Thus a selected-byte write addresses the range 17 through 272. A selected-word write starts in the range 20 through 275. Neither operation adds the selector base to the final destination.

For readable arithmetic below, “scaled by” denotes multiplication; “shifted” denotes a left shift unless explicitly described as a right shift. Address constants and multipliers are decimal.

The displayed computations use 32-bit operands except where a pair result is explicitly shown. For machine-word interpretation, left shifts and multiplication retain only the low 32 bits. Some displayed operands are signed, however, so this is not a claim that every expression has defined portable-language semantics. Exact overflow and address-formation behavior require the underlying instruction semantics.

Unsigned masks and right shifts are established where explicitly described. Conversion to a byte retains the low eight bits.

### Preconditions

Successful continued execution requires:

- Every reached indirect target to be executable under the actual calling convention.
- Each dependency to return whenever subsequent behavior is expected.
- Every computed memory address to permit the required read or write.
- The environment to support the alignment and access widths actually used.
- Incoming implicit state to be valid for the sequence.

No local validation enforces these conditions.

## Behavior

### Dependency-sequence notation

Dependency sites are assigned opaque IDs here because no trusted external dependency identities are supplied.

- F-D000 is the first, incoming-control-dependent call.
- F-D001 through F-D032 denote the ordered call groups described below.
- An individual call within a group is identified by its position, for example F-D004.2.

These IDs distinguish observable call sites; they do not assert that different sites resolve to different implementations.

A **five-call chain** below means one call with the stated explicit arguments, followed by four calls with no explicit arguments shown. Its reported result comes from the fifth call. A **four-call chain** means one explicitly described call followed by three calls with no explicit arguments shown.

“No explicit arguments shown” does not guarantee an empty machine-level argument interface. Unmodeled registers and other implicit state may still be visible to the callee.

### Initialization

1. Invoke F-D000 without explicit arguments. Its target depends on the incoming control/link value.
2. Invoke four further calls, F-D001.1 through F-D001.4, without explicit arguments. Retain the word result of the fourth.
3. Perform a selected-byte write through the incoming selector base, storing the incoming byte.
4. Invoke the five-call chain F-D002, passing the retained word shifted by 2 to its first call. Retain its final word result as the first **pass seed**.

The loop starts only after this sequence completes. There are no explicit memory accesses before the first five calls, although those dependencies may access memory themselves.

### Repeating pass

The following operations occur in order in each pass. Values described as retained remain logically available across intervening calls in the supplied representation; this does not establish that arbitrary dependencies preserve their underlying machine locations.

#### 1. Initial selected-byte update and pair exchange

Derive a **scaled seed** by shifting the pass seed by 2.

Perform a selected-byte write through the current selector base. The stored byte is obtained by masking the current field value with hexadecimal `3fffffff`, logically shifting right by 20, and retaining eight bits. Consequently the stored value reflects original bits 20–27, not all ten bits remaining after the mask and shift.

F-D003 consists of these seven calls:

1. Call with the current shift-source value shifted by 12.
2. Call without explicit arguments; retain its word result.
3. Call with that result and the scaled seed shifted by 28.
4. Call without explicit arguments; retain its pair result.
5. Call with the low and high halves of that pair.
6. Call without explicit arguments.
7. Call without explicit arguments; retain its word result.

Then perform a selected-byte write through the current selector base, storing zero.

#### 2. First word propagation

Run F-D004 as a five-call chain, initially passing the preceding word result shifted by 2. Shift the chain’s final result by 2 to obtain the **first propagated word**.

Write that word at `scaled seed × 4096 + 20`.

Run F-D005 as a five-call chain whose first call receives, in order:

- `scaled seed × 1024`;
- the same value again;
- `pass seed shifted by 14`;
- the first propagated word.

Write the first propagated word at `F-D005 final result × 4 + 20`.

Run F-D006 as a five-call chain, initially passing `F-D005 final result × 4`. Derive the **second propagated word** as `F-D006 final result × 4`, and write it at `F-D006 final result × 16 + 20`.

Run F-D007 as a five-call chain, passing the second propagated word three times to its first call. Write that same word at `F-D007 final result × 4 + 20`.

Run F-D008 as a five-call chain, initially passing `F-D007 final result × 4`.

Derive:

- The **third propagated word** as `F-D008 final result × 4`.
- The **high-bit base** as `F-D008 final result × 1073741824`.

Write the third propagated word at `F-D008 final result × 16 + 20`.

Run F-D009 as a five-call chain, passing the third propagated word three times. Write it at `F-D009 final result × 4 + 20`. Preserve this final result as the **saved index**.

#### 3. Repeated byte clears and indexed word read

Run F-D010 as a five-call chain, initially passing `saved index × 4`. Retain its final result as the **lookup selector**.

Write zero first at `high-bit base + 17`, then at `high-bit base + 1`.

F-D011 has seven calls:

1. Call with the lookup selector shifted by 2.
2. Call without explicit arguments; retain its word result.
3. Call with that result.
4–7. Four calls without explicit arguments.

Retain the seventh call’s word result. Again write zero first at `high-bit base + 17`, then at `high-bit base + 1`.

Run F-D012 as a five-call chain, initially passing that retained result shifted by 2. Preserve its final result as the **later index**, and derive the **later index scale** as `later index × 4`.

Write `saved index shifted by 31` at `later index × 16 + 20`.

Only after that write, read a word at:

`((lookup selector masked with hexadecimal 3fffffff) logically shifted right by 20) + 20`.

This read starts at an address between 20 and 1043 inclusive. Its address is not scaled to a word boundary; alignment therefore cannot be presumed.

#### 4. Field extraction and selector reuse

F-D013 consists of a call with the later index scale, followed by a call without explicit arguments. Retain the second call’s result as the **saved transfer value**.

Run F-D014 as a five-call chain. Its first argument is the word just read, logically shifted right by 22 and then shifted left by 28. Under 32-bit truncation, only original bits 22–25 contribute.

Write a zero word at `F-D014 final result × 4 + 20`.

Run F-D015 as a five-call chain, initially passing `F-D014 final result × 4`. Retain its final result as the **field source**.

Read the selector byte at `high-bit base + 1`, then write the low byte of `field source shifted by 2` at `selector byte + 17`.

F-D016 consists of:

1. A call with the field source shifted by 2.
2. A call without explicit arguments; retain its result.
3. A call with that result and the field source shifted by 30.
4–7. Four calls without explicit arguments.

Read the selector byte at `high-bit base + 1` again. Preserve this newly read byte separately. Write at its value plus 17 the low byte of the field source masked with hexadecimal `3fffffff` and logically shifted right by 20.

Run F-D017 as a five-call chain, initially passing `F-D016 final result shifted by 2`. Ignore its displayed final result.

Write the saved transfer value shifted by 2 at the **preserved selector byte plus 20**. This address uses the byte read before F-D017, not a fresh selector read.

Then read a separate global word. Its address, overlapping-symbol layout, and application meaning are not established here.

#### 5. Global-derived byte update and second propagation

Run F-D018 as a five-call chain, initially passing the later index scale. Its final result is a pair. Derive the **pair-derived scale** by shifting its low half by 2.

Perform a selected-byte write through the high-bit base, storing the low byte of the global word logically shifted right by 22.

F-D019 has seven calls:

1. Call with the high half of the F-D018 pair.
2. Call without explicit arguments; retain its result.
3. Call with that result and the pair-derived scale shifted by 28.
4–7. Four calls without explicit arguments.

Perform a selected-byte write through the high-bit base, storing zero.

Run F-D020 as a five-call chain, initially passing `F-D019 final result shifted by 2`.

Run F-D021 as a five-call chain, passing `F-D020 final result shifted by 2` twice. Shift its final result by 2 to obtain the **second propagation value**.

Write that value at `pair-derived scale × 4096 + 20`.

Run F-D022 as a five-call chain, initially passing the pair-derived scale shifted by 14. Write the second propagation value at `F-D022 final result × 4 + 20`.

Run F-D023 as a five-call chain, initially passing `F-D022 final result × 4`. Its final result is a pair.

#### 6. Final selected-byte updates and loop-carried state

Derive the **next scale** by shifting the low half of the F-D023 pair by 2.

Perform a selected-byte write using `pair-derived scale × 16384` as selector base. Store the low byte of the second propagation value masked with hexadecimal `3fffffff` and logically shifted right by 20.

F-D024 has seven calls:

1. Call with the high half of the F-D023 pair.
2. Call without explicit arguments; retain its result.
3. Call with that result alone.
4. Call without explicit arguments; retain its pair result.
5. Call with that pair’s low and high halves.
6–7. Calls without explicit arguments.

Perform a selected-byte write using `next scale × 268435456` as selector base, storing zero.

Run F-D025 as a five-call chain, initially passing `F-D024 final result shifted by 2`. Preserve its final result as the **next transfer index**.

Set the next/current selector base to `next scale × 4`. Perform a selected-byte write through it, storing zero.

F-D026 has seven calls:

1. Call with the next scale shifted by 10.
2. Call without explicit arguments; retain its result.
3. Call with that result and the saved transfer value shifted by 28.
4–7. Calls without explicit arguments.

Perform another selected-byte write through the current selector base, storing zero.

Run F-D027 as a five-call chain, initially passing `F-D026 final result shifted by 2`. Its displayed final result is not used.

Derive the **next field value** as `next transfer index shifted by 14`. Write it at `(next transfer index shifted by 2) × 4096 + 20`.

F-D028 consists of:

1. A call with `(next transfer index shifted by 2) shifted by 14`.
2. A call without explicit arguments.
3. A call with the next field value shifted by 28 and an additional, insufficiently established secondary call result.
4–7. Calls without explicit arguments.

Write the next field value at `F-D028 final result × 4 + 20`.

Run F-D029 as a five-call chain, initially passing `F-D028 final result × 4`. Its final word result becomes the next pass seed.

Repeat the pass with:

- The updated selector base.
- The next field value as the current field value.
- The next field value also as the current shift-source value.

There is no intervening termination test.

## Side effects

The routine explicitly performs byte reads, byte writes, word reads, and word writes. Accesses may overlap, including across widths. Their order must not be collapsed merely because some writes store zero or appear repetitive.

Dependencies execute between many accesses. A dependency may change a later selector, the global word, or any other shared state. In particular:

- Repeated selector reads are independent observations.
- A byte selector retained across calls remains distinct from a later reread.
- The separate global read occurs after the saved-transfer word write.
- Word and byte destinations may alias source locations or each other.

No explicit allocation, release, ownership transfer, synchronization, rollback, or error-reporting operation is established. Passing a numeric value to a dependency does not establish whether it is a pointer or whether ownership changes.

## Dependencies

F-D000 and F-D001 through F-D029 are opaque external interactions. F-D030 through F-D032 are unused identifiers in this document and imply no additional calls.

Most call targets are represented as fixed control-relative calculations; several use different offsets, and F-D000 depends on incoming control state. Arithmetic similarity does not establish semantic identity.

The dependencies’ signatures, side effects, failure behavior, memory ownership, and preservation of globals or implicit state are unknown. Pair results are established only at the sites explicitly identified above. The additional secondary result passed within F-D028 lacks a sufficiently established producer and is **documented_only**.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Initial selector indirection.** If the incoming selector byte is 7, initialization writes the incoming byte at address 24. It does not write at the incoming selector base plus 24.

2. **Masked byte truncation.** If the current field value is hexadecimal `3ff00000`, the first pass’s selected-byte value is 255. The ten-bit shifted intermediate is truncated to eight bits.

3. **Preserved selector.** If the selector read before F-D017 yields 9, and F-D017 subsequently changes the selector memory to 12, the saved-transfer word write still starts at address 29.

4. **Unaligned lookup.** If the lookup selector is hexadecimal `00100000`, the indexed word read starts at address 21. Acceptance requires support for that access or an environment-specific fault; silently rounding to address 20 changes behavior.

5. **High-bit base reduction.** Under 32-bit modular arithmetic, the high-bit base is determined solely by the low two bits of the F-D008 result. A result of 4 produces base zero.

6. **Repeated clearing.** Even if the first pair of zero writes leaves both destinations zero, the second pair remains observable and must occur after F-D011 returns.

7. **Dependency non-return.** If an initialization dependency never returns, no later explicit write is required. If a loop dependency never returns, already completed accesses are not rolled back.

8. **Loop continuation.** If all dependencies return and all memory accesses succeed, finishing F-D029 causes another pass rather than a normal function return.

## Uncertainties

- **documented_only:** Exact calling convention and implicit input locations.
- **documented_only:** Instruction-level overflow, signed shifts, pointer formation, endianness, and alignment behavior.
- **documented_only:** Whether low numeric addresses are mapped memory, special hardware, or artifacts of an incomplete representation.
- **documented_only:** The global word’s address and overlap with other storage.
- **documented_only:** Dependency identity, aliasing between call targets, hidden arguments, clobbers, ownership effects, and failure semantics.
- **documented_only:** The secondary result consumed within F-D028.
- **Inferred intent:** Repeated scales and offsets suggest structured state manipulation, but no record schema or application role is established.
- **Representation limitation:** An unreachable-region warning and overlapping-global warning accompany the supplied material. They reduce confidence in completeness and typing; they do not establish additional reachable behavior.

## Evidence

This contract is based solely on the supplied static representation. Established evidence consists of the visible invocation order, explicit operand widths, masks and shifts, memory-access expressions, result reuse, initialization boundary, and unconditional loop.

The mathematical descriptions are a behavioral organization of that evidence, not a claim of recovered high-level design. Dependency behavior and machine details remain qualified where the supplied material does not establish them.

No implementation access, runtime execution, or tests were used. No snapshot hash or independent provenance claim is asserted.
