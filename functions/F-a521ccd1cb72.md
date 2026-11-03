<!-- Trusted provenance: F-a521ccd1cb72; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a521ccd1cb72

## Purpose

F-a521ccd1cb72 copies four 32-bit values from an input object into a caller-supplied output region. It then calls F-ffe0d92c57b7 and conditionally adjusts the output using two values obtained through the returned object.

The adjustment is pairwise: one value is subtracted from output positions 0 and 2, and another from positions 1 and 3.

**Inferred intent:** the four values may describe two coordinate pairs or rectangle endpoints, with the conditional subtraction translating them relative to another object. Neither the geometric interpretation nor the meaning of the condition is established. The behavioral contract is therefore expressed in terms of offsets and word positions.

## Interface

### Inputs

The function receives two arguments:

| Argument | Role | Required accessible region |
|---|---|---|
| Input-object address | Identifies the object containing the initial four values | Four 32-bit reads at byte offsets `0x80`, `0x84`, `0x88`, and `0x8C` |
| Output address | Identifies caller-provided storage | Four consecutive 32-bit positions, starting at byte offset zero |

The supplied representation uses integer-valued addresses for the input object and for addresses obtained during execution. The accesses establish a 32-bit word layout, but do not independently establish every property of the execution environment, including its native pointer width or byte order.

For a conventional contiguous object, the input reads reach through byte offset `0x8F`, and the output occupies 16 bytes. Accessibility of these particular locations is required; this does not imply that the function accesses every preceding byte.

### Output and return value

There is no explicit return value. Results are communicated through the output region.

The function always attempts to populate all four output positions before calling F-ffe0d92c57b7. Whether those values remain unchanged afterward depends on:

- The effects of that dependency.
- The condition and link fields subsequently read.
- Whether the optional arithmetic completes without a memory or arithmetic fault.

No status code reports whether the adjustment occurred.

### Preconditions

Successful completion requires:

- Valid addresses for every read and write actually reached.
- Appropriate alignment, or an execution environment that permits the accesses used.
- A usable returned-object address from F-ffe0d92c57b7.
- A usable linked-object address whenever that link is nonzero and the adjustment branch is taken.
- Arithmetic behavior defined by the applicable execution model for the values involved.

The function performs no explicit validation of these requirements. In particular, it does not check either argument for null, and it does not check the dependency’s return value for null before accessing a field through it.

## Behavior

### Initial transfer

The initial input reads occur in this order:

1. The 32-bit value at input offset `0x84`.
2. The 32-bit value at input offset `0x88`.
3. The 32-bit value at input offset `0x8C`.
4. The 32-bit value at input offset `0x80`.

The output writes then occur in increasing output-position order:

| Output position | Byte offset | Initial source |
|---|---:|---|
| 0 | `0x00` | Input offset `0x80` |
| 1 | `0x04` | Input offset `0x84` |
| 2 | `0x08` | Input offset `0x88` |
| 3 | `0x0C` | Input offset `0x8C` |

All four input values have been read before the first output write. Consequently, overlap between the input fields and the output region does not, by itself, cause an early output write to replace an initial value that has not yet been captured.

This is a property of the initial transfer only. It does not imply unrestricted alias safety for later dependency-visible objects.

There is no atomic four-word snapshot or atomic four-word write. Concurrent mutation could affect the individually observed reads, and another observer could see a partially written output.

### Dependency call and branch selection

After completing the initial four output writes, the function calls F-ffe0d92c57b7 with no explicit arguments.

The returned value is treated as an object address. The function first reads the 32-bit field at returned-object offset `0x80`.

- If this field is nonzero, the function returns immediately.
- If this field is zero, the function reads the address-valued field at returned-object offset `0x78`.

The link at offset `0x78` is not read when the field at offset `0x80` is nonzero.

If the link is zero, the function returns without performing the adjustment. If the link is nonzero, execution continues through that linked object.

The dependency’s return value itself has no analogous zero check. A zero return does not select a deliberate no-adjustment path; it leads to an attempted field access through that value.

The two early-return cases mean “no further output writes by this function.” They do not establish that the output still contains the initial copy: F-ffe0d92c57b7 may have changed the output or related state.

### Adjustment values

For a nonzero link, the function reads:

1. A 32-bit adjustment value at linked-object offset `0x80`.
2. A second 32-bit adjustment value at linked-object offset `0x84`.

Both values are captured before any adjustment write to the output.

For a conventional contiguous linked object, these reads reach through byte offset `0x87`. The returned object’s condition-field read reaches through offset `0x83`.

The same numeric offset, `0x80`, serves different observable roles in the two objects: a branch condition in the returned object, and an adjustment value in the linked object. No shared semantic meaning is established.

### Output adjustment

The function adjusts output positions in this order:

| Order | Output position | Operation |
|---:|---:|---|
| 1 | 0 | Subtract the first adjustment value |
| 2 | 1 | Subtract the second adjustment value |
| 3 | 2 | Subtract the first adjustment value |
| 4 | 3 | Subtract the second adjustment value |

Each operation reads the corresponding output position at that time and writes its adjusted value back. The function does not reconstruct the output from the original input values after the dependency call.

If the output values immediately before adjustment are \(q_0,q_1,q_2,q_3\), and the adjustment values are \(a,b\), the ordinary mathematical result is:

\[
(q_0-a,\ q_1-b,\ q_2-a,\ q_3-b)
\]

This expression describes the result only where the relevant arithmetic is defined. It also presumes no interfering mutation during those output accesses.

Because both adjustment values are captured before output adjustment begins, overlapping output writes cannot change the already captured adjustment amounts. However, initial output writes or the dependency call may already have changed the objects used for branch selection or adjustment.

### Widths and arithmetic boundaries

The represented data operations use signed 32-bit integer values. There is no visible widening, saturation, clamping, or explicit overflow check.

The arithmetic is expressed as addition of a negated adjustment value. This matters at the signed minimum:

- Negating `−2,147,483,648` is not representable as a signed 32-bit integer.
- A mathematically out-of-range result is likewise not representable as a signed 32-bit integer.
- Even a mathematically representable subtraction does not, by itself, establish defined evaluation when the intermediate negation is out of range.

The supplied material does not establish whether the executable wraps modulo \(2^{32}\), traps, or has some other behavior at these boundaries. Unconditional acceptance criteria must not assume wraparound.

## Side effects

### Direct memory effects

The function directly performs:

- Four initial input reads.
- Four initial output writes.
- One call to F-ffe0d92c57b7.
- A returned-object condition read.
- A conditional link read.
- Two conditional linked-object reads.
- Four conditional output read-modify-write operations.

No direct allocation, release, I/O, or named global-variable access is visible.

This does not constrain indirect effects of F-ffe0d92c57b7. The dependency may access globals, mutate reachable memory, allocate, release resources, or fail; none of those behaviors is resolved here.

### Ownership and lifetime

The output is caller-provided storage. This function contains no explicit operation that allocates it, replaces its address, or releases it.

There is no visible ownership transfer for the input object, returned object, or linked object. Nevertheless, the dependency cannot be assumed to preserve their lifetimes. Successful continuation requires every subsequently accessed object to remain valid after the dependency returns.

### Aliasing and partial completion

The function does not reject overlapping addresses.

Aliasing can affect more than the copied values. For example, initial output writes may alter memory that later becomes the returned object’s condition field, link field, or linked-object adjustment fields. Branching and adjustment then use the values actually read after the dependency call, not an earlier snapshot.

There is no rollback mechanism. If execution fails after some writes, the completed writes are not visibly undone. A failure during adjustment can leave some output positions adjusted and others unadjusted.

## Dependencies

### F-ffe0d92c57b7

**Established call contract at this call site:**

- Called once on the normal path after the four initial output writes.
- Receives no explicit arguments.
- Returns a value used as an address.
- The caller immediately attempts a 32-bit read at returned-address offset `0x80`.
- If that field is zero, the caller also reads the address-valued field at offset `0x78`.

**Not established:**

- How the returned object is selected.
- Whether the return is always valid or nonzero.
- Whether the returned object or its link aliases the input or output.
- Whether the dependency modifies output memory or other relevant state.
- Whether it preserves object lifetimes.
- Whether it can fail or terminate execution.

No stronger dependency contract is assumed.

## Acceptance examples

These examples are conditional behavioral criteria, not observed runtime results. Unless stated otherwise, they assume valid memory, stable relevant state, no dependency mutation of the output, and arithmetic without overflow or an unrepresentable negation.

### Nonzero condition

Input fields at offsets `0x80` through `0x8C` contain:

\[
(10,\ 20,\ 30,\ 40)
\]

If F-ffe0d92c57b7 returns a valid object whose field at offset `0x80` is nonzero, the final output is:

\[
(10,\ 20,\ 30,\ 40)
\]

The link field is not read. Any nonzero condition value selects this path, including a negative value.

### Zero condition with zero link

With the same input, a zero condition and zero link leave the output at:

\[
(10,\ 20,\ 30,\ 40)
\]

No linked-object adjustment fields are read.

### Positive adjustment

If the condition is zero, the link is valid and nonzero, and the linked adjustment values are `3` and `7`, the final output is:

\[
(7,\ 13,\ 27,\ 33)
\]

### Negative adjustment

For input values:

\[
(-5,\ 12,\ 15,\ 22)
\]

and adjustment values `−4` and `2`, the result is:

\[
(-1,\ 10,\ 19,\ 20)
\]

Negative adjustment values therefore increase the corresponding output values when arithmetic is defined.

### Dependency-modified output

Suppose the initial copy produces \((10,20,30,40)\), but the dependency changes the output to \((100,200,300,400)\). If the adjustment branch is taken with values `3` and `7`, the result is:

\[
(97,\ 193,\ 297,\ 393)
\]

The adjustment uses current output values, not retained originals.

### Overlapping initial copy

If the output overlaps the input fields, the initial transfer must still use the four values captured before the first output write. Later behavior remains dependent on whether those writes changed dependency-visible fields or other relevant memory.

### Arithmetic boundary

An adjustment value of `−2,147,483,648` does not have an unconditional expected numeric result under this contract. A wraparound result is acceptable only if separately established for the actual execution environment.

## Uncertainties

The following remain **documented_only**:

- Exact pointer width, byte order, alignment rules, and address-arithmetic behavior.
- Overflow behavior and treatment of unrepresentable negation.
- Semantic names and units for the four copied values and two adjustments.
- Meaning of the returned object’s condition and link.
- Full behavior and lifetime guarantees of F-ffe0d92c57b7.
- Concurrency guarantees and synchronization requirements.
- Fault behavior for invalid addresses or inaccessible memory.
- Intended restrictions on aliasing.

The coordinate-translation interpretation is inferred intent, not an additional requirement.

## Evidence

The supplied static function body establishes the memory offsets, four-value transfer, call placement, branch conditions, conditional read order, and pairwise adjustment structure described above.

The supplied dependency mapping establishes F-ffe0d92c57b7 as the opaque identifier for the called dependency. Its implementation was not supplied.

No runtime tests have been performed. The acceptance examples are derived conditional expectations, not execution evidence.
