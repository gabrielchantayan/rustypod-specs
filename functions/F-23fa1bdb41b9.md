<!-- Trusted provenance: F-23fa1bdb41b9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-23fa1bdb41b9

## Purpose

F-23fa1bdb41b9 updates a caller-supplied memory record. It first invokes a dependency with two writable field addresses, then derives two shifted positions, a row-based address, a stride, a word-span value, and two 32-bit boundary masks.

The arithmetic is consistent with preparing access to a horizontal interval in a packed, 32-bit-word-oriented representation. That interpretation is **inferred intent**, not an established data-format contract. The established behavior is the dependency invocation and the field accesses and calculations described below.

This function does not visibly allocate memory, release memory, return a result value, or report an error. The dependency’s implementation is unavailable; its effects and ownership behavior remain unknown.

## Interface

### Calling interface

The function accepts one argument representing the address of a mutable record. The supplied representation uses a 32-bit argument and 32-bit memory accesses.

There is no explicit return value. Results are communicated through writes to the record and through any effects of the dependency.

Let **R** denote the incoming record address. Offsets below are byte offsets from R. Field names are descriptive labels, not recovered public identifiers.

| Offset | Width | Role in this function |
|---|---:|---|
| +0x04 | Address of a field | Passed as the dependency’s second argument |
| +0x08 | Address of a field | Passed as the dependency’s third argument |
| +0x0C | 32 bits | Row multiplier used in address calculation |
| +0x10 | 32 bits | First position before shifting |
| +0x18 | 32 bits | Second position before shifting |
| +0x1C | 32 bits | Address of a descriptor |
| +0x20 | 32 bits | Derived address output |
| +0x24 | 32 bits | Copied stride output |
| +0x28 | 32 bits | Derived word-span output |
| +0x2C | 32 bits | Shifted first-position output |
| +0x30 | 32 bits | Shifted second-position output |
| +0x34 | 32 bits | First boundary-mask output |
| +0x38 | 32 bits | Second boundary-mask output |

Passing addresses at +0x04 and +0x08 does not establish the size or type of the objects the dependency accesses there.

The descriptor has these observed fields:

| Descriptor offset | Width | Role |
|---|---:|---|
| +0x00 | 32 bits | Base value used in the derived address |
| +0x04 | 32 bits | Stride |
| +0x08 | 32 bits | Value passed as the dependency’s first argument |
| +0x0C | 32 bits | Shift control; only its low eight bits are selected |

The descriptor used to supply the dependency’s first argument need not be the descriptor used afterward: the record’s descriptor-address field is read again after the call.

### Preconditions and validity

For ordinary completion, all addresses actually accessed must be valid for their respective reads or writes. This includes the descriptor reached before the dependency call and the descriptor reached afterward.

No explicit checks establish:

- A non-null record or descriptor address.
- Sufficient accessible storage.
- Alignment.
- Valid shift counts.
- Ordered or nonnegative positions.
- A valid row index.
- Absence of arithmetic or address overflow.
- Whether the calculated address refers to accessible memory.

The function stores the calculated address but does not visibly dereference it.

The supplied representation suggests a 32-bit address environment. It does not establish byte order or the exact fault behavior of invalid accesses.

## Behavior

### 1. Invoke the dependency before deriving outputs

Before any of this function’s explicit output-field stores, it:

1. Reads the descriptor address from R +0x1C.
2. Reads the 32-bit value at that descriptor’s +0x08 field.
3. Calls **F-4e9bdab369e8** with four arguments:
   - That 32-bit value.
   - The address R +0x04.
   - The address R +0x08.
   - The scalar value 1.

Any return value is unused.

The dependency receives addresses within the record. Its behavior is not available, so it cannot be assumed to modify only those fields, preserve the descriptor address, preserve globals, or leave other reachable memory unchanged.

If the dependency does not return normally, none of the subsequent behavior is established.

### 2. Reload the descriptor and calculate shifted positions

After the dependency returns, the function reloads the descriptor address from R +0x1C.

It then calculates:

- **A**, the first-position value from R +0x10 shifted left by the low eight bits of the descriptor’s +0x0C field.
- **B**, the second-position value from R +0x18 shifted left by the low eight bits of that descriptor field.

The two calculations each reference the shift-control field. The supplied representation does not show a single shared shift-control snapshot.

Both calculations occur before either shifted-position output is stored. The function then writes:

1. A to R +0x2C.
2. B to R +0x30.

A and B are subsequently treated as unsigned 32-bit quantities for right shifts and remainder extraction.

Selecting the low eight shift-control bits does **not** explicitly restrict the count to 0–31. Behavior for selected counts of 32–255 is not resolved by the supplied evidence. No processor-specific shift-count masking is assumed.

### 3. Copy the stride and calculate an address

The function next:

1. Reads the descriptor’s 32-bit stride at +0x04.
2. Writes that value to R +0x24.
3. Calculates and writes the derived address to R +0x20.

For arithmetic whose signedness and overflow behavior are not in dispute, the address calculation is:

**descriptor base + row multiplier × stride + 4 × floor(A / 32)**

Here:

- The descriptor base is read from descriptor offset +0x00.
- The row multiplier is read from R +0x0C.
- The stride is the value already read and copied.
- Division of A by 32 is an unsigned right shift by five.
- The final factor of four converts a 32-bit-word index into a byte offset.

The stride copy occurs before the base and row reads used by the address expression. Consequently, overlapping storage can matter. For example, if the destination of the stride copy overlaps the descriptor’s base field, the subsequent address calculation can observe the copied stride as the base.

The supplied evidence does not fix the relative ordering of the separate memory reads within the address expression.

### 4. Derive the word difference and boundary information

Using the retained A and B values, define:

- **QA** = floor(A / 32).
- **QB** = floor(B / 32).
- **RA** = A modulo 32.
- **RB** = B modulo 32.
- **D** = the 32-bit difference QB − QA.

The subtraction is unsigned in the supplied representation before its result is assigned to a signed-width temporary. If QB is less than QA, no rejection or clamping is performed. The exact signed interpretation of an out-of-range conversion is not established as a portable language-level rule.

The first boundary mask is:

**0xFFFFFFFF logically shifted right by RA**

Its shift count is always 0–31. Thus:

- RA = 0 produces 0xFFFFFFFF.
- RA = 1 produces 0x7FFFFFFF.
- RA = 31 produces 0x00000001.

The second boundary mask depends on RB:

- If RB = 0, the mask is 0xFFFFFFFF.
- Otherwise, its intended 32-bit pattern has the highest RB bits set and the remaining bits clear.

For nonzero RB, that pattern is expressed in the supplied representation by left-shifting signed −1 by 32 − RB. This is not a portable signed-left-shift definition. The all-ones high-bit mask is the evident bit-level intent; language-independent shift semantics are an uncertainty.

The final word-span value is:

- D when RB is nonzero.
- D − 1 when RB is zero.

That decrement applies even when D is zero. There is no visible empty-interval exception.

### 5. Store span and masks in the observed order

After the address output, the remaining explicit store order is:

1. Write D to R +0x28.
2. Write the first boundary mask to R +0x34.
3. If RB = 0, overwrite R +0x28 with D − 1.
4. Write the second boundary mask to R +0x38.

For a second position exactly on a 32-bit boundary, the span field is therefore written twice. This intermediate store is part of the observed sequence, even though ordinary final-state inspection sees only the adjusted result.

The function then returns without an explicit result value.

## Side effects

### Direct record changes

On normal completion, the function directly writes all of these 32-bit fields:

- +0x20: derived address.
- +0x24: copied stride.
- +0x28: span, conditionally written twice.
- +0x2C and +0x30: shifted positions.
- +0x34 and +0x38: boundary masks.

The direct write order is:

**+0x2C → +0x30 → +0x24 → +0x20 → +0x28 → +0x34 → conditional +0x28 → +0x38**

This sequence follows the dependency call. It is not an atomic record update and provides no synchronization guarantee.

### Aliasing and retained values

No non-overlap requirement is checked.

The descriptor address is retained after its post-call reload, but the entire descriptor is not snapshotted. Writes to the record may affect later descriptor reads if their storage overlaps.

In particular:

- The shifted-position stores precede the stride read.
- The stride-output store precedes the descriptor-base read.
- A and B remain retained values for the later span and mask calculations.
- The address expression uses the retained stride, rather than rereading it after its output store.

The behavior therefore must not be replaced by an assumed “read every input first, then write every output” model when overlapping storage is relevant.

### Ownership and errors

No explicit ownership transfer, allocation, deallocation, or reference-count operation appears in this function.

That observation does not establish what F-4e9bdab369e8 does with its arguments or other state.

There is no visible status result, validation branch, recovery path, or rollback. A failure after some stores may leave partial updates; no stronger failure guarantee is established.

## Dependencies

### F-4e9bdab369e8

**Invocation:** once, before the derived-field calculations.

**Arguments:** a 32-bit descriptor value, two addresses within the record, and the scalar 1.

**Result handling:** any result is ignored.

**Established caller behavior:** the descriptor address is reloaded after the call, and all derived calculations use the post-call memory state as subsequently read.

**Unknown dependency behavior:** argument interpretation, read and write extents, global-state changes, allocation, ownership, retained addresses, errors, and normal-return guarantees.

No additional function calls are visible.

## Acceptance examples

These examples are conditional specifications, not runtime observations. They assume:

- The dependency returns normally.
- The stated inputs describe memory after that return.
- Record and descriptor storage do not overlap.
- Memory accesses are valid and stable.
- Shift counts are within 0–31.
- Relevant arithmetic is representable, except where a boundary case explicitly discusses a 32-bit pattern.
- The second-mask shift has the intended 32-bit bitwise meaning described above.

### Interval inside one word

Given first position 3, second position 10, shift control 0, base 0x1000, stride 16, and row multiplier 2:

| Output | Expected value |
|---|---:|
| Shifted first position | 3 |
| Shifted second position | 10 |
| Copied stride | 16 |
| Derived address | 0x1020 |
| Final span | 0 |
| First mask | 0x1FFFFFFF |
| Second mask | 0xFFC00000 |

No combined single-word mask is stored. The two boundary masks remain separate outputs.

### Second position exactly on a boundary

Given first position 5 and second position 32, with shift control 0:

- QA = 0 and QB = 1.
- The span field is first written as 1, then overwritten as 0.
- The first mask is 0x07FFFFFF.
- The second mask is 0xFFFFFFFF.

This demonstrates the special decrement for an aligned second position.

### Scaling before word selection

Given first position 17, second position 40, shift control 1, base 0x2000, stride 64, and row multiplier 3:

| Output | Expected value |
|---|---:|
| Shifted first position | 34 |
| Shifted second position | 80 |
| Derived address | 0x20C4 |
| Final span | 1 |
| First mask | 0x3FFFFFFF |
| Second mask | 0xFFFF0000 |

The address uses the word index of the shifted first position, not the original position.

### Equal aligned positions

Given both positions equal to 32 and shift control 0:

- Both shifted outputs are 32.
- The first span write is zero.
- The aligned-end adjustment produces the final 32-bit pattern 0xFFFFFFFF.
- Both masks are 0xFFFFFFFF.

The function does not normalize this case to a zero span or an empty result.

### Dependency changes the descriptor address

If F-4e9bdab369e8 changes R +0x1C and returns normally, subsequent calculations use the reloaded descriptor address. The dependency’s first argument still came from the descriptor reached before the call.

### Out-of-range selected shift count

If the low eight bits of the shift-control field equal 32, the source shows no rejection branch. A specific shifted-position result is **documented_only**: the available evidence does not establish the execution-level shift semantics for that count.

## Uncertainties

### Established static evidence

The supplied representation establishes the field widths and offsets, dependency arguments, post-call descriptor reload, visible calculation structure, branch condition, and explicit store order.

It also establishes the absence of a visible check or error-return path in this function. It does not establish that callers always satisfy the needed preconditions.

### Inferred intent

The division into 32-bit words, four-byte word displacement, and opposing boundary masks suggest packed-bit interval preparation.

The aligned-second-position decrement suggests an exclusive endpoint convention. Neither interpretation proves the caller-visible domain model.

### Documented-only unknowns

The following remain unresolved:

- Exact execution semantics for oversized shift counts.
- Portable semantics of left-shifting negative signed values.
- Signed overflow and out-of-range signed-conversion behavior.
- Address-overflow behavior.
- Endianness, alignment requirements, and invalid-access fault behavior.
- Dependency effects and ownership rules.
- Whether negative or reversed inputs are valid caller inputs.
- Concurrent access, volatile memory, and memory-mapped-device semantics.
- Whether downstream consumers combine masks or interpret the span in a particular way.

No unknown is resolved by assuming conventional graphics behavior or a particular processor.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-23fa1bdb41b9 and the supplied opaque dependency mapping.

No implementation of F-4e9bdab369e8, caller contract, executable behavior, or external data-layout specification was available. No runtime tests were performed. The acceptance examples are arithmetic consequences under their stated conditions, not reported execution results.
