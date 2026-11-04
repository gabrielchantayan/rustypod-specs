<!-- Trusted provenance: F-8761ef6eefb4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8761ef6eefb4

## Purpose

The recovered operation performs a byte store, captures two 32-bit values from memory, and conditionally writes a five-word record to an address derived from its second explicit input. A state-dependent loop separates the initial memory accesses from the record writes. For many arithmetic states, that loop has no exit in the supplied representation.

The visible path ends at an invalid-instruction boundary rather than a normal return. The evidence does not establish a useful higher-level purpose, such as object initialization, serialization, or device control. Those interpretations must not be assumed.

This contract describes the supplied static representation. It does not establish that the representation faithfully recovers an executable function: the supplied evidence explicitly reports problematic instruction decoding and truncated control flow.

## Interface

### Explicit inputs

The visible interface has four input slots:

| Input | Visible width and interpretation | Visible use |
|---|---|---|
| First | 32-bit value, with no established signedness | Not used in the recovered body |
| Second | 32-bit unsigned value; also reinterpreted as signed for one address calculation | Selects a byte-read address and determines the record destination |
| Third | 32-bit value, with no established signedness | Not used in the recovered body |
| Fourth | 32-bit signed value; its bit pattern also participates in an address calculation and byte truncation | Determines the initial byte-store address, stored byte, and part of the loop state |

The first and third inputs are unused only within the visible representation. No conclusion is established about their role in a correct decoding or in surrounding execution.

### Additional incoming state

A further 32-bit machine-state value is consumed without a visible assignment or explicit parameter slot. This contract calls it the **hidden incoming word**. It affects the loop and the first and fifth record words.

Consequently, the four explicit inputs alone do not determine the visible result.

The operation also consumes:

- A byte at one of four low numeric addresses.
- A 32-bit word from a fixed global location whose address and external role are not provided.
- A 32-bit word from an address selected by the byte read.

### Output and return behavior

No ordinary return value is established. The recovered interface is void, but the visible path does not reach a normal return:

- One class of arithmetic states remains in the loop indefinitely in the supplied representation.
- The other class proceeds through five word stores and reaches an invalid-instruction termination boundary.

The observable outputs are memory mutations and the terminal execution outcome, not a returned record or status code.

### Execution prerequisites

Completing the visible memory operations requires readable and writable mappings at the computed addresses, suitable access permissions, and any alignment required by the target.

No pointer validity checks, bounds checks, alignment checks, or error conversion are visible. Memory faults may interrupt the operation before later accesses occur. The evidence does not define fault delivery, recovery, or rollback.

## Behavior

### Numeric conventions and limits of interpretation

The representation distinguishes 8-bit byte accesses and 32-bit word accesses. Address calculations use 32-bit values.

For the arithmetic descriptions below:

- A **logical right shift** inserts zero bits.
- An **arithmetic right shift** propagates the sign bit.
- A **low byte** is the least significant eight bits.
- The **signed subtraction overflow predicate** reports whether subtracting the signed value 14 from a signed 32-bit intermediate would fall outside the signed 32-bit range.

The bit-level results and acceptance ranges below use a 32-bit wrapping machine-arithmetic interpretation. That interpretation is consistent with the widths and explicit overflow predicate in the recovered representation. It is not independently verified machine behavior. In particular, the representation does not by itself resolve language-level signed-overflow issues or prove the accuracy of the recovered instructions.

### Derived record destination

The second input is reinterpreted as a signed 32-bit value and shifted arithmetically right by 13 bits. The resulting 32-bit pattern is used as the base address for the five-word record.

This is not equivalent to an unsigned division by 8,192 for inputs whose high bit is set.

Under the arithmetic-shift interpretation, possible base-address patterns are:

- `0x00000000` through `0x0003FFFF` for second inputs with a clear high bit.
- `0xFFFC0000` through `0xFFFFFFFF` for second inputs with a set high bit.

No alignment is imposed. Adjacent second-input values can produce the same destination; some destinations are unaligned for 32-bit accesses.

### Arithmetic state

The fourth input is shifted arithmetically right by 29 bits. Its signed result is therefore between −4 and 3, inclusive.

The hidden incoming word is subtracted from that result to form a 32-bit intermediate. A second subtraction removes 14 from the intermediate to form the **candidate word**.

The signed interpretation of the intermediate is used by the overflow predicate for the subtraction of 14. The candidate word is otherwise used as an unsigned bit pattern for its right shift and eventual store.

### Initial memory accesses

The following access order is visible before the loop:

1. **Read one byte.**  
   Shift the second input logically right by 30 bits, producing an index from 0 through 3. Read one byte at numeric address `0x18 + index`. Thus, the possible read addresses are exactly `0x18`, `0x19`, `0x1A`, and `0x1B`.

2. **Write one byte.**  
   Write the low byte of the fourth input to the address formed by adding `0x1D` to that input’s 32-bit bit pattern.

3. **Read the fixed global word.**  
   Capture a 32-bit value from the fixed global location.

4. **Read the byte-selected word.**  
   Zero-extend the byte captured in step 1, add `0x4C`, and read a 32-bit word starting at that address. The possible starting addresses range from `0x4C` through `0x14B`, inclusive.

The byte-selected word address is not scaled by the word width. Every byte value selects a different starting address, including unaligned addresses.

The captured selector byte is read before the byte store. If those addresses alias, the later store does not change the selector already captured.

Both 32-bit reads occur after the byte store. If that store overlaps either word’s storage, the subsequently captured value may reflect the store, subject to the target’s byte order and memory semantics.

### Loop and exit condition

The loop repeatedly derives a **shifted candidate** by shifting the candidate word logically right by 15 bits.

It continues only when both conditions hold:

- The shifted candidate is nonzero.
- Subtracting 14 from the signed intermediate does not report signed overflow.

No visible loop operation changes the candidate word, the intermediate, or any other input to this condition. There are no visible memory accesses in the loop body.

Accordingly, under the recovered control flow, the loop either exits on its first condition check or continues indefinitely. It is not a visible polling loop: no external memory location is re-read to permit progress.

Under the 32-bit arithmetic interpretation, first-check exit occurs in either of two cases:

- The candidate word is between `0x00000000` and `0x00007FFF`, inclusive. Equivalently, the signed intermediate is between 14 and 32,781, inclusive.
- The signed intermediate is between −2,147,483,648 and −2,147,483,635, inclusive. Subtracting 14 then reports signed overflow, and the candidate bit pattern lies between `0x7FFFFFF2` and `0x7FFFFFFF`.

For all other intermediate values, the shown loop does not exit.

### Five-word record

If the loop exits, the operation performs five 32-bit stores in increasing offset order:

| Byte offset from record base | Stored value |
|---|---|
| 0 | Candidate word |
| 4 | Previously captured byte-selected word |
| 8 | Previously captured fixed global word |
| 12 | Record base address, represented as a 32-bit word |
| 16 | Shifted candidate shifted left by 6 bits |

The record spans 20 bytes when all stores complete. Its byte order is not established.

For the ordinary exit case, where the candidate is below `0x8000`, the shifted candidate is zero, so the word at offset 16 is zero.

For the signed-overflow exit case, the shifted candidate is `0xFFFF`, so the word at offset 16 is `0x003FFFC0`.

The two captured memory values are not re-read during record construction. If record stores overwrite their original storage, later fields still use the captured values.

### Terminal boundary

After the fifth word store, the representation reaches an invalid-instruction termination marker. No normal return, subsequent access, cleanup, or recovery path is visible.

This establishes the end of the recovered path, not a portable promise that execution exits a process, raises a particular exception, or invokes a real callable routine.

## Side effects

### Memory and state

The visible effects are:

- One byte store before the loop.
- Up to five ordered word stores after loop exit.
- An execution state that either remains in the shown loop or reaches the invalid-instruction boundary, assuming earlier accesses succeed.

The writes are not shown as transactional. A fault during record construction may leave only a prefix of the five fields written. There is no visible rollback.

Aliasing is unrestricted. In particular:

- The initial byte store may overlap either later word-read region.
- The record may overlap the earlier read regions or the initial byte-store address.
- Address arithmetic near the end of the 32-bit address space raises wrapping and access-boundary questions not resolved by the supplied evidence.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, reference-count operation, or memory-lifetime guarantee is visible.

The numeric destination and source addresses must not be treated as newly allocated or owned memory. Successful stores would establish only that bytes were written, not that a valid higher-level object was constructed.

### Errors and concurrency

No returned error value or explicit validation failure path is shown.

The evidence does not specify atomicity, volatile behavior, synchronization, memory ordering, or concurrent access guarantees. The access sequence above describes the recovered single-execution order; it does not establish a cross-thread visibility contract.

## Dependencies

No ordinary callable dependency with an assigned opaque function ID is present in the supplied dependency map.

The operation depends on these external resources or execution properties:

| Dependency | Required role | Unestablished properties |
|---|---|---|
| Hidden incoming word | Supplies arithmetic state | Origin, initialization, calling convention, valid range |
| Four-address byte region | Supplies the selector byte | Mapping, mutability, device behavior |
| Byte-selected word region | Supplies the record’s second word | Alignment support, ownership, byte order |
| Fixed global word | Supplies the record’s third word | Exact location, meaning, concurrent updates |
| Record destination mapping | Receives five words | Capacity, permissions, lifetime |
| Execution environment | Handles access faults and terminal invalid instruction | Exception behavior, continuation, recovery |

The terminal marker is not evidence of a normal dependency call. No global-preservation or memory-release behavior may be attributed to an unknown handler.

## Acceptance examples

These are conditional static expectations, not runtime test results. They assume faithful recovered control flow, 32-bit arithmetic as described, valid memory mappings, and successful accesses unless stated otherwise.

### Ordinary record write

Suppose:

- The second input is `0x00200000`.
- The fourth input is `0x00001000`.
- The hidden incoming word is `0xFFFFFFF2`, interpreted as −14.
- The selector byte at `0x18` is `0x04`.
- The word captured at `0x50` is a value called **A**.
- The fixed global word captured after the byte store is a value called **B**.

Then:

- The record base is `0x100`.
- A zero byte is written at `0x101D`.
- The arithmetic intermediate is 14 and the candidate is zero.
- The loop exits on its first check.
- The five words at offsets 0, 4, 8, 12, and 16 are respectively zero, **A**, **B**, `0x100`, and zero.
- Execution then reaches the invalid-instruction boundary.

### Upper ordinary-exit boundary

If the intermediate is 32,781, the candidate is `0x7FFF`. The loop exits, the first record word is `0x7FFF`, and the fifth is zero.

If the intermediate instead is 32,782, the candidate is `0x8000`. The shifted candidate is one, signed subtraction overflow is false, and the shown loop continues indefinitely. The earlier byte store and both word reads have already occurred; no record word is written.

### Negative candidate without subtraction overflow

If the intermediate is 13, the candidate bit pattern is `0xFFFFFFFF`. Its logical right shift is nonzero, and subtracting 14 does not overflow the signed range. The loop does not exit.

A negative-looking candidate does not, by itself, select the overflow exit.

### Signed-overflow exit

If the intermediate is −2,147,483,648, subtracting 14 reports signed overflow. Under wrapping arithmetic, the candidate is `0x7FFFFFF2`.

The loop exits despite its nonzero shifted candidate. The first record word is `0x7FFFFFF2`, and the fifth is `0x003FFFC0`.

### Selector aliasing

If the initial byte store targets the same address as the selector-byte read, the already captured selector determines the later word-read address. The overwritten byte is not read again for selection.

### Access failure

If the byte-selected word read faults, the initial byte store may already be visible. The loop and record stores have not yet been reached. No rollback or returned error is established.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered entry point, instructions, and control flow are accurate.
- The source and valid values of the hidden incoming word.
- Exact machine behavior for overflowing arithmetic and wrapping address calculations.
- The target’s byte order, alignment requirements, and memory map.
- The identity and role of the fixed global word.
- Whether low numeric addresses denote ordinary memory, special hardware, or inaccessible storage.
- Fault handling and the execution outcome at the invalid-instruction boundary.
- Any intended object type or broader application purpose.

The arithmetic exit ranges are derived from the recovered expressions under stated bit-width assumptions. They are not runtime-verified guarantees.

## Evidence

Established static evidence consists of the supplied four-input representation, its additional incoming state use, byte and word access widths, address expressions, captured-value ordering, invariant loop condition, five ordered record stores, and terminal invalid-instruction marker.

Inferred intent is limited to describing the five stores as a record write. No semantic object type, ownership model, or successful initialization protocol is established.

The supplied decoding warnings materially limit confidence in the recovered behavior. No implementation execution, runtime tests, or independent instruction-level validation has been performed.
