<!-- Trusted provenance: F-fea7d74d537b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fea7d74d537b

## Purpose

F-fea7d74d537b returns a 32-bit value selected from an indirect record or calculated from fields reached through its input object.

The established behavior has two mutually exclusive paths:

- If an indirect-record reference is nonzero, return that record’s value at byte offset `0x04`, substituting a global value when the record’s value is zero.
- Otherwise, inspect a secondary record. Return zero unless its calculation base is nonzero and its signed comparison boundary exceeds an input-associated numeric value. When both conditions hold, return the calculation base plus 24 times that numeric value plus 7.

The meaning of these records and values is not established. Descriptions such as “indirect record,” “calculation base,” and “boundary” identify their roles in this contract, not their application-level types.

## Interface

### Input

The function accepts one 32-bit argument used as the address of an input object. It is not treated as an optional reference: the function attempts to read a field from that address before performing any null check.

The following input-object fields are relevant:

| Byte offset | Width | Role |
|---|---:|---|
| `0x08` | 32 bits | Numeric operand for comparison and calculation on the secondary path |
| `0x0C` | 32 bits | Address of a secondary record |
| `0x14` | 32 bits | Address of an indirect record; zero selects the secondary path |

Although the value at `0x08` is represented with a pointer type in the supplied material, its observable uses here are numeric. The function does not dereference it.

The indirect record has one relevant field:

| Byte offset | Width | Role |
|---|---:|---|
| `0x04` | 32 bits | Preferred return value; zero requests the global fallback |

The secondary record has two relevant fields:

| Byte offset | Width | Role |
|---|---:|---|
| `0x30` | 32 bits | Calculation base; zero disables the calculation |
| `0x34` | 32 bits | Signed comparison boundary; read only when the calculation base is nonzero |

A separate global location contains a 32-bit fallback value. Its address, initialization rules, and application-level meaning are not established.

### Output

The function returns a 32-bit integer. Depending on the selected path, the return value is:

- The nonzero preferred value from the indirect record.
- The global fallback value, including zero if that is what the global contains.
- Zero because the secondary calculation is disabled or its comparison fails.
- The calculated result when the secondary-path conditions succeed.

There is no separate status output. Zero is not uniquely an error indication: several ordinary paths can return it.

### Preconditions

For normal completion, each address actually dereferenced must identify readable storage of the required width. This requirement is path-dependent.

A nonzero address is not validated for readability. The supplied material establishes no alignment recovery, bounds checking, exception translation, or invalid-reference handling.

The record names above do not imply allocation or ownership. No requirement that these objects were allocated together is established.

## Behavior

### 1. Select the path

The first observable data access reads the 32-bit indirect-record reference at input offset `0x14`.

If this first read is nonzero, the indirect-record path is selected. Otherwise, the secondary path is selected.

The function does not combine the paths. In particular, a zero value inside an indirect record does not cause the secondary record to be examined.

### 2. Indirect-record path

After selecting this path, the function reads the reference at input offset `0x14` again. It then reads the 32-bit preferred value at byte offset `0x04` from the address obtained by that second read.

Consequently, the address used for the preferred-value access is not necessarily the address observed by the initial path-selection read.

If the preferred value is nonzero, that value is returned directly. Its sign does not affect selection: any nonzero 32-bit value qualifies.

If the preferred value is zero, the function reads the global fallback value and returns it. There is no second test of that global value and no further fallback.

On this path:

- The numeric operand at input offset `0x08` is not read.
- The secondary-record reference at input offset `0x0C` is not read.
- Neither secondary-record field is read.
- The global fallback is read only if the preferred value is zero.

If the input reference changes between its two reads, the second value controls the dereference. In particular, the first read being nonzero does not establish that the second read is nonzero. The function supplies no synchronization or consistency check to prevent this situation.

### 3. Secondary path: initial reads

If the first read of input offset `0x14` is zero, the function proceeds in this order:

1. Read the 32-bit numeric operand at input offset `0x08`.
2. Read the secondary-record address at input offset `0x0C`.
3. Read the 32-bit calculation base at secondary-record offset `0x30`.

The numeric operand is read even when the calculation base subsequently proves to be zero. It is not dereferenced.

The secondary-record address is dereferenced without a preceding nonzero check.

### 4. Zero calculation base

If the calculation base is zero, the function returns zero.

In this case, it does not read the secondary-record reference a second time and does not access the boundary at offset `0x34`.

Thus, readability of the boundary field is not required by this path. Readability of the calculation-base field remains required.

### 5. Nonzero calculation base

If the calculation base is nonzero, the function reads the secondary-record reference at input offset `0x0C` again. It then reads the 32-bit boundary at offset `0x34` from the address obtained by this second reference read.

The calculation base and boundary therefore need not come from the same record if the input reference changes between reads. The earlier calculation-base value remains the value used in the eventual calculation.

The boundary and numeric operand are compared as signed 32-bit quantities. The calculation proceeds only when the boundary is strictly greater than the numeric operand.

This is not an unsigned address-order comparison. The supplied condition accounts for overflow in its intermediate 32-bit subtraction when deciding signed ordering. Equality is explicitly excluded.

If the boundary is less than or equal to the numeric operand under signed 32-bit ordering, the function returns zero.

### 6. Calculation and return

When the calculation base is nonzero and the signed boundary is greater than the numeric operand, the returned quantity is:

\[
\text{calculation base} + 24 \times \text{numeric operand} + 7
\]

The boundary is used only to authorize this calculation. It is not added to the result and does not determine the multiplier.

The numeric operand is used as its stored numeric value, not as a displacement relative to the boundary or to the secondary-record address. There is no established nonnegativity constraint on it.

For values whose multiplication and additions are representable as signed 32-bit operations, this formula specifies the result exactly.

Overflow behavior for the result expression is **documented_only**. The supplied representation uses 32-bit signed arithmetic but does not establish an explicit overflow contract. Low-32-bit wraparound is a plausible target-level interpretation, not an independently established guarantee of this specification.

## Side effects

The supplied function contains no explicit writes to the input object, either referenced record, or the global fallback location. It does not allocate, free, retain, or transfer ownership of memory.

Its observable memory activity consists of the conditional reads described above. It contains no explicit calls to other functions.

The absence of explicit writes does not establish that the reads are side-effect-free in every environment. Access to memory-mapped devices, concurrently modified storage, or inaccessible memory may have effects outside the ordinary readable-memory model. Those environments are not specified.

There is no established atomic snapshot across fields or repeated reference reads. Callers requiring coherent values must provide suitable stability or synchronization; this function does not visibly provide it.

No diagnostic output, error object, retry, or recovery action appears. The consequences of an invalid access are **documented_only**, not a specified return value.

## Dependencies

No function dependencies are visible in the supplied material. No additional opaque function IDs are therefore required.

The function depends on:

- Readable storage for the input fields reached on the selected path.
- Readable indirect-record storage when that path is selected.
- Readable secondary-record storage when the secondary path is selected.
- The global fallback location when the indirect record’s preferred value is zero.
- A 32-bit value and address model consistent with the supplied operations.

No allocator, destructor, callback, or external ownership convention is established.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not runtime test results. Unless an example says otherwise, all accessed addresses are valid and fields remain stable throughout the call.

### Preferred value bypasses the secondary path

Given a nonzero indirect-record reference and a preferred value of 91, the result is 91.

Neither the global fallback nor any secondary-path field is read. A negative nonzero preferred value would likewise be returned directly.

### Zero preferred value selects the global

Given a nonzero indirect-record reference, a preferred value of zero, and a global fallback of −6, the result is −6.

The secondary path remains unvisited. If the global fallback were zero, the result would instead be zero without further processing.

### Zero calculation base avoids the boundary read

Given a zero indirect-record reference and a secondary calculation base of zero, the result is zero regardless of the numeric operand.

The numeric operand and calculation base are still read. The boundary is not read, so its value does not affect this case.

### Strictly greater boundary permits calculation

Given:

- A zero indirect-record reference.
- A numeric operand of 3.
- A calculation base of 100.
- A boundary of 4.

The result is 179: the signed comparison succeeds, and \(100 + 24 \times 3 + 7 = 179\).

### Equality rejects calculation

With the same values except that the boundary is 3, the result is zero.

A nonzero calculation base does not override the strict comparison.

### Negative operands remain numeric operands

Given a numeric operand of −2, a calculation base of 100, and a boundary of −1, the result is 59.

The signed comparison succeeds because −1 is greater than −2. The calculation uses −2 directly.

### Signed ordering differs from unsigned ordering

Given a numeric operand of zero, a nonzero calculation base, and a boundary with bit pattern `0xFFFFFFFF`, the result is zero. That boundary represents −1 under signed 32-bit interpretation.

Conversely, a boundary of 2,147,483,647 compares greater than an operand of −2,147,483,648 despite overflow in a naive 32-bit subtraction. The calculation branch is selected, but its resulting arithmetic exceeds the exact, non-overflow contract described above.

### Repeated references are not a snapshot

If the indirect-record reference changes from one valid nonzero record address to another between its two reads, the preferred value comes from the second address.

Similarly, on the secondary path, a calculation base may be obtained from one record and the boundary from another if the secondary-record reference changes between reads.

These examples establish the visible read dependency, not a guarantee about any particular concurrent execution model.

## Uncertainties

The following remain **documented_only**:

- The application-level purpose and units of the returned value.
- Whether any field is intended to represent an index, count, address, handle, or encoded value.
- The meaning of the multiplier 24 and additive constant 7.
- The global fallback’s initialization, mutability, and ownership.
- Result-arithmetic behavior when signed 32-bit multiplication or addition overflows.
- Address-formation behavior when adding a field offset exceeds the address range.
- Byte order and alignment requirements.
- Fault behavior for invalid or inaccessible storage.
- Concurrency guarantees, atomicity, and permitted mutation during execution.
- Behavior omitted from the supplied representation.

The pointer-like representation of the numeric operand is not evidence that this function accesses an object through it. Likewise, the address-like shape of a returned value does not establish that callers may dereference it.

## Evidence

The established claims come solely from static inspection of the supplied function representation: its 32-bit loads, byte offsets, branch conditions, repeated reference reads, signed overflow-aware comparison, arithmetic expression, and returns.

The role labels and the interpretation that the function selects or constructs an application value are explanatory inferences. They do not add validation, ownership, or semantic guarantees.

The supplied material includes notices that unreachable regions were removed. Their contents are unavailable and are not reconstructed here.

No implementation was executed, no runtime tests were performed, and no external dependency behavior was verified.
