<!-- Trusted provenance: F-c9b5d17d8d1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9b5d17d8d1e

## Purpose

F-c9b5d17d8d1e conditionally subtracts two values obtained through a linked record from four caller-supplied integer slots.

The four slots are adjusted in alternating pairs: the first and third receive the first subtraction; the second and fourth receive the second subtraction. Two early-return conditions leave the destination untouched: a nonzero field in the primary record, or a zero link in that record.

A possible interpretation is conversion of a four-component coordinate representation into a linked record’s reference frame. That interpretation is inferred intent, not an established requirement. The supplied evidence does not identify the slots as rectangle boundaries, coordinates, or any other specific data type.

## Interface

The function accepts two arguments:

- **Primary record address.** An address used to read a control field and, conditionally, a link to another record.
- **Destination address.** An address identifying four consecutive writable integer slots. On the active path, their existing contents are read before being replaced.

The function has no explicit return value. Its established result is the conditional in-place modification of the destination.

### Representation and widths

The supplied representation uses integer accesses and a link held in an integer-sized field. Its apparent memory model is 32-bit: four-byte integer slots and a four-byte encoded address. The layouts below describe that model. The evidence does not independently establish the target ABI, address-space rules, alignment requirements, or byte order.

| Region | Byte offset | Meaning in this contract |
|---|---:|---|
| Primary record | `0x80` | Integer control field; any nonzero value disables adjustment |
| Primary record | `0x78` | Integer-encoded linked-record address; zero disables adjustment |
| Linked record | `0x80` | First subtraction value |
| Linked record | `0x84` | Second subtraction value |
| Destination | `0x00` | First integer slot |
| Destination | `0x04` | Second integer slot |
| Destination | `0x08` | Third integer slot |
| Destination | `0x0C` | Fourth integer slot |

No intervening fields are examined. In particular, the function does not copy or validate either record as a whole.

### Preconditions

The required accessible memory depends on the path taken:

- Every invocation that completes normally must be able to read the primary record’s control field.
- If that field is zero, the primary record’s link field must also be readable.
- If the link is nonzero, both subtraction fields in the linked record must be readable.
- Only on that active path must all four destination slots be readable and writable.

No explicit check rejects a zero primary address, a zero destination address, an invalid nonzero link, misalignment, or an insufficient allocation. Early return can avoid some otherwise-invalid accesses, but does not validate those addresses for later use.

For unambiguous ordinary-integer results, the arithmetic must remain representable in the applicable integer width. Overflow behavior is not fully established; see **Uncertainties**.

## Behavior

### 1. Test the control field

The function first reads the integer at primary-record offset `0x80`.

If the value is nonzero, the function returns immediately. On that path:

- The link field is not read.
- Neither linked-record subtraction field is read.
- No destination slot is read or written.

The test is for zero versus nonzero, not for a particular flag value. Negative values and positive values both disable adjustment.

### 2. Test the link

If the control field is zero, the function reads the field at primary-record offset `0x78` and uses it as the linked-record address.

If that value is zero, the function returns immediately. On that path:

- Neither subtraction field is read.
- No destination slot is read or written.
- No fallback adjustment is performed.

A nonzero value is used directly. There is no established range, ownership, or record-type check.

### 3. Capture both subtraction values

For a nonzero link, the function reads:

1. The first subtraction value from linked-record offset `0x80`.
2. The second subtraction value from linked-record offset `0x84`.

Both values are obtained before the first destination update. Each is captured once and reused for its two corresponding slots.

This matters if source fields and destination storage overlap: a destination write does not cause either subtraction value to be fetched again.

### 4. Update the destination in order

Let the captured subtraction values be \(a\) and \(b\). In the ordinary nonoverlapping case, let the four destination values before adjustment be \(d_0,d_1,d_2,d_3\).

The resulting values are:

\[
(d_0-a,\ d_1-b,\ d_2-a,\ d_3-b)
\]

The established source-level update order is:

1. Read the first slot and write its adjusted value.
2. Read the second slot and write its adjusted value.
3. Read the third slot and write its adjusted value.
4. Read the fourth slot and write its adjusted value.

The destination is not first copied into a four-element snapshot. Each slot’s prior value is read when that slot is processed.

The function performs no normalization, sorting, clipping, saturation, or relationship check between the slots. Negative subtraction values increase the corresponding destination values when the arithmetic is representable.

### Completion and interruption

After the fourth update, the function returns without producing a status value.

No rollback mechanism is present. If the execution environment interrupts an active invocation after one or more stores, earlier stores may already have taken effect. Exact fault and recovery behavior belongs to the runtime environment and is not established here.

## Side effects

### Reads

On the active path, the source-level sequence is:

- Primary control-field read.
- Primary link-field read.
- First linked-record value read.
- Second linked-record value read.
- Four destination read-and-update operations, in slot order.

The control field and link are not rechecked after the linked-record values have been captured.

These are source-level sequencing observations. They do not establish hardware transaction boundaries, externally observable bus ordering, or synchronization semantics.

### Writes

The only explicit writes are to the four destination slots on the active path. The source describes an update for every slot even if its subtraction value is zero; whether an unchanged-value write remains observable in generated execution is not established.

Neither record is an explicit write target. However, destination aliasing can make these writes modify storage that also represents primary-record or linked-record fields. “Read-only record access” therefore does not imply that the records remain unchanged for every possible memory layout.

### Ownership and lifetime

There is no explicit allocation, deallocation, ownership transfer, or retention of either argument. No external routine is called.

The caller must provide memory with suitable lifetime and access permissions for the path that executes. The evidence does not establish language-level borrowing rules or whether every physically possible overlap is valid under the original program’s type and aliasing rules.

### Concurrency

No lock, atomic operation, version check, or retry is shown. The two subtraction values are separate reads, not an established atomic snapshot. A concurrent change could therefore prevent them from representing one coherent record state.

No concurrent-mutation guarantee is specified.

## Dependencies

No dependency calls are present in F-c9b5d17d8d1e. The supplied dependency information identifies no additional opaque function IDs used by this operation.

Its behavior depends on:

- Address interpretation and memory-access rules.
- The integer width and signed-arithmetic model.
- Validity of the caller-provided storage.
- The execution environment’s treatment of invalid accesses and arithmetic overflow.

There is no evidence of a logging, validation, allocation, or error-reporting dependency.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not runtime test results.

### Nonzero control field

Given a readable primary control field containing `1`, `-1`, or any other nonzero value:

- The invocation returns without accessing the link or destination.
- The destination is unchanged by this function.
- A usable linked record is not required for this path.

A destination address need not be dereferenced on this path. This does not establish that arbitrary invalid argument values are acceptable under an unknown calling environment.

### Zero control field and zero link

Given a zero control field and a readable link field containing zero:

- The invocation returns after reading the link.
- No linked-record or destination access occurs.
- No destination value changes.

### Ordinary positive subtraction

Given a zero control field, a valid nonzero link, captured values \(a=3\) and \(b=5\), and nonoverlapping destination contents:

\[
(10,20,30,40)
\]

normal completion produces:

\[
(7,15,27,35)
\]

This example assumes all accesses are valid and all arithmetic is representable.

### Negative and zero subtraction values

Given captured values \(a=-4\), \(b=0\), and destination contents:

\[
(2,-3,8,9)
\]

normal completion produces:

\[
(6,-3,12,9)
\]

The second and fourth slots have unchanged numerical values. The source nevertheless specifies their update operations.

### Destination overlaps a captured source field

Suppose an otherwise valid layout makes a destination store overwrite the linked record’s first subtraction field.

Both subtraction values must already have been read before that store. The third-slot adjustment uses the captured first value, not a newly fetched value from the overwritten field.

This expectation is conditional on the overlap being valid under the actual execution model.

### Unrepresentable arithmetic

If a captured subtraction value is the minimum signed value of the apparent 32-bit model, negating it is not representable in that model. Likewise, adjusting a slot can exceed the signed range.

For such inputs, acceptance must not require a particular wrapped result without additional evidence. The supplied material does not establish a portable overflow outcome.

## Uncertainties

The following remain **documented_only**:

- **Exact ABI and width confirmation.** Four-byte integers and integer-encoded addresses are apparent from the representation, not independently confirmed by architecture metadata.
- **Overflow semantics.** The displayed operation uses negation followed by addition. Signed negation or addition may overflow. Portable signed arithmetic does not guarantee wrapping, while actual machine behavior is not supplied.
- **Address arithmetic.** Behavior when record-field address calculation exceeds the valid address range is unknown.
- **Byte order and alignment.** Neither is established.
- **Memory classification.** There is no evidence that the records are volatile, memory-mapped, atomic, or synchronized.
- **Aliasing validity.** Capture-before-write behavior is visible, but the original language’s permitted overlapping layouts are unknown.
- **Fault behavior.** No function-level error return or recovery is shown; the environment’s response to invalid access is unspecified.
- **Domain meaning.** Coordinate translation is plausible but unconfirmed. No slot ordering beyond alternating subtraction is established.

These unknowns do not weaken the established zero/nonzero gating or the active-path update sequence for valid, representable executions.

## Evidence

This contract is based solely on the supplied static representation of F-c9b5d17d8d1e. That evidence establishes:

- Two ordered early-return conditions.
- Reads at the stated record offsets.
- Capture of both subtraction values before destination writes.
- Four ordered in-place adjustments using alternating captured values.
- No explicit return value, dependency call, allocation, release, or error-reporting operation.

No implementation was executed, no runtime tests were performed, and no external implementation or ABI documentation was inspected. Domain intent and platform-dependent outcomes remain separated from the established static behavior.
