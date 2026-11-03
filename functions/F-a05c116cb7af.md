<!-- Trusted provenance: F-a05c116cb7af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a05c116cb7af

## Purpose

F-a05c116cb7af conditionally copies a requested number of bytes from a caller-supplied source into storage addressed through a mutable state object. It returns a 32-bit success indicator.

The operation has two admission conditions:

1. When the fourth argument is nonzero, dependency F-15e46ffb3077 must return zero.
2. The requested byte count must not exceed an availability value calculated from four 32-bit state fields.

Once admitted, the operation copies bytes individually and updates a state-held write position after each byte. There is no second availability check during the copy.

**Inferred intent:** The state layout and masked position update resemble a circular byte buffer. This interpretation is useful terminology, not an additional guarantee. The established behavior is the arithmetic, field access, copying, and return behavior specified below.

## Interface

### Arguments

| Argument | Meaning established by the supplied representation |
|---|---|
| State address | Address of a mutable object containing the fields listed below. Passed unchanged to F-15e46ffb3077 when that dependency is called. |
| Source address | Address from which successive bytes are read after admission. The function does not write through this argument directly. |
| Requested count | Unsigned 32-bit number of bytes requested. |
| Gate argument | A 32-bit value tested for equality with zero. Every nonzero value selects the dependency call; its magnitude has no other established significance. |

The state address is represented by a 32-bit integer used in address calculations. Storage addresses are likewise obtained from 32-bit fields. This describes the supplied representation, not a portable pointer-size guarantee for another environment.

### State layout

Offsets are byte offsets from the state address. Names below describe their roles in this function only.

| Offset | Width | Contract name | Use |
|---|---:|---|---|
| +4 | 32 bits | Storage address | Supplies the base address for each destination byte. |
| +8 | 32 bits | Occupancy adjustment | Participates in the admission calculation. |
| +12 | 32 bits | Read position | Subtracted from the write position during admission. |
| +16 | 32 bits | Write position | Participates in admission, addresses each destination byte, and is updated after every copied byte. |
| +20 | 32 bits | Mask | Participates in admission and masks each updated write position. |

No meaning is established for the object’s bytes at offsets not listed here. In particular, this function makes no direct access to the field at offset zero.

The byte order used to interpret these 32-bit fields is not established by the supplied representation.

### Return value

The return value is a 32-bit quantity:

- **1:** The gate and availability checks passed, and all requested copy iterations completed. This includes an admitted request of zero bytes.
- **0:** The dependency returned nonzero, or the requested count exceeded the calculated availability.

No other normal return value is shown. Invalid memory accesses and dependency failures outside a normal return are not converted into a defined status by this function.

### Preconditions for defined completion

The caller must provide accessible memory for every state-field access actually reached, every source byte actually read, and every destination byte actually written.

No null check, bounds check against an allocated storage extent, alignment repair, or malformed-state rejection is shown. A zero-byte request avoids the copy-loop accesses, but does not inherently avoid the dependency or admission-field accesses.

## Behavior

### 1. Optional gate

If the gate argument is zero, F-15e46ffb3077 is not called.

If the gate argument is nonzero, the function first calls F-15e46ffb3077 with the state address:

- A nonzero result causes an immediate return of 0.
- A zero result allows the availability check to proceed.

Consequently, an invoked dependency runs before this function reads the state fields for admission. If the dependency changes those fields, admission uses the state present after the call, not a preserved pre-call snapshot.

No contract for the dependency’s effects is supplied. A rejected gated request therefore does **not** imply that all memory or global state remained unchanged.

### 2. Availability calculation

Let the admission-time field values be:

- \(W\): write position;
- \(R\): read position;
- \(A\): occupancy adjustment;
- \(M\): mask;
- \(N\): requested count.

The established 32-bit calculation is:

\[
U = ((W - R + A) \bmod 2^{32}) \mathbin{\&} M
\]

\[
F = (M - U) \bmod 2^{32}
\]

Admission succeeds exactly when the unsigned comparison \(N \leq F\) is true.

Here, \(\mathbin{\&}\) means bitwise AND. These are fixed-width operations, not unbounded mathematical integer arithmetic. The addition and subtraction inside the masked expression are represented with signed integer types, while the mask and final comparison are unsigned. The operational interpretation above captures the low-32-bit target arithmetic; it does not assert that arbitrary signed overflow would be portable in a higher-level language.

Because \(U\) is a bitwise subset of \(M\), its unsigned value cannot exceed \(M\). Thus the final subtraction does not underflow under this bit-vector interpretation.

Important consequences:

- The formula uses \(M\), not \(M+1\), as the upper bound for availability.
- A request of zero passes this comparison whenever it is reached.
- No requirement that \(M\) be one less than a power of two is checked.
- No requirement that either position already lie within the mask is checked.
- The occupancy adjustment affects admission only; it is not added to destination addresses.

The supplied representation establishes the admission expression, but not a reliable left-to-right order among its individual field reads. It does not provide an atomic state snapshot.

If the unsigned comparison fails, the function returns 0 without entering the copy loop.

### 3. Byte copying

After admission, the function performs exactly \(N\) iterations if all accesses complete normally.

For each iteration:

1. It reads one byte from the current source address and writes that byte to the destination selected by the current storage-address field plus the current write-position field.
2. After that byte store, it updates the 32-bit write-position field to the low-32-bit result of adding one to its then-current value, bitwise ANDed with the then-current mask.
3. It advances its local source address by one byte.

The destination byte store precedes the write-position update. A byte value must be obtained before that value can be stored, but the supplied representation does not establish the relative order of the source-byte read and the field reads used to form its destination address.

The storage address and write position are referenced anew for each byte. The write position and mask used for the position update are referenced after the destination store. There is no established caching of these fields across iterations.

For ordinary non-overlapping storage with a stable mask, each completed iteration advances the position according to:

\[
W_{\text{next}} = ((W_{\text{current}} + 1) \bmod 2^{32}) \mathbin{\&} M
\]

This equation describes a conventional circular increment only for appropriate masks. The operation remains a bitwise mask for every other value.

The initial write position is used directly to form the first destination address. It is **not** masked before that first store. A malformed initial position can therefore select an address outside the intended storage even when admission succeeds.

### 4. Completion and rejection

After the final iteration, the function returns 1. There is no post-copy callback or second capacity check.

When admission fails, the function performs no copy-loop source reads, destination stores, or direct write-position updates.

A memory fault or other abnormal interruption during copying has no specified rollback. Earlier byte stores and write-position updates may already have occurred.

## Side effects

### Direct effects

On an admitted nonzero request, the function directly:

- Reads successive source bytes.
- Writes one destination byte per iteration.
- Writes the 32-bit write-position field once per completed iteration.

It does not explicitly update the read position, occupancy adjustment, storage-address field, or mask.

However, the destination address is not checked against the state object. If destination storage overlaps any state field, the byte stores can modify that field. Accordingly, “not explicitly updated” is not a guarantee that those bytes remain unchanged.

The incremented source address is local to the invocation. No updated source pointer is returned or written to a caller-owned pointer slot.

### Aliasing

No overlap detection or overlap-safe bulk-copy policy is shown.

If earlier destination writes overlap later source locations, later iterations read the modified bytes. The result need not match a copy from an immutable snapshot of the original source.

If destination storage overlaps the write-position field, the byte store can alter the value subsequently read for the position update. If it overlaps the mask, the update can use the altered mask. Other overlapping state fields can affect later destination addressing.

Acceptance examples below assume non-overlapping state, source, and destination memory unless stated otherwise.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, or retention of the source address is shown.

The operation uses the state’s storage address for immediate writes. This alone does not establish who owns that storage, who must release it, or how long it remains valid.

F-15e46ffb3077 may have additional effects. Its ownership behavior and effects on global state are unknown and must not be inferred from this caller.

### Concurrency

No synchronization or atomic-access mechanism is shown. Concurrent mutation of the state or participating memory is outside the established guarantees. In particular, the function does not establish a coherent admission snapshot or a single published write-position update for the entire copied sequence.

## Dependencies

### F-15e46ffb3077

**Invocation condition:** The gate argument is nonzero.

**Input:** The state address.

**Observed result use:** Zero permits the availability check; any nonzero result causes return 0.

**Established ordering:** The call completes before this function performs admission-field reads or copy-loop accesses.

**Unknown effects:** State mutation, global mutation, memory ownership changes, synchronization, callbacks, and abnormal termination are not specified.

There are no other explicit function dependencies in the supplied representation.

## Acceptance examples

These examples are conditional static predictions, not runtime test results. They assume valid accessible memory, stable non-overlapping fields and buffers, normal dependency completion where applicable, and the 32-bit arithmetic described above.

### Gate rejection

Given a nonzero gate argument and a dependency result of 7:

- The dependency receives the state address.
- The function returns 0.
- It does not perform its admission-field reads or any copy-loop accesses.
- Any effects of the dependency remain outside this caller’s specified guarantees.

This outcome also applies when the requested count is zero.

### Zero-count success

Given a zero gate argument and valid admission fields:

- A requested count of zero returns 1.
- No source byte or storage-address field is needed by the copy loop.
- No destination byte or write-position field is written.

Thus the source may be unused, but the state cannot be treated as wholly unused.

### Availability boundary

Given \(M=7\), \(W=2\), \(R=0\), and \(A=0\):

- Masked occupancy is 2.
- Availability is 5.
- A request for 5 bytes is admitted.
- A request for 6 bytes returns 0 without copying.

For the admitted request, bytes go to storage offsets 2 through 6, and the final write position is 7.

### Wrap at the mask

Given \(M=7\), \(W=7\), \(R=7\), \(A=0\), and a request for 3 bytes:

- Availability is 7.
- Destination offsets are 7, 0, and 1, in that order.
- The final write position is 2.
- Each destination byte is stored before its associated position update.

### Admission adjustment

Given \(M=7\), \(W=2\), \(R=0\), and \(A=3\):

- Masked occupancy is 5.
- Availability is 2.
- A request for 3 bytes returns 0.

Changing the adjustment changes admission without changing how an admitted byte’s destination address is formed.

### Zero mask

Given \(M=0\):

- Availability is zero regardless of the other admission values.
- A zero-byte request succeeds if the optional gate permits it.
- Every positive request is rejected before copying.

### Non-circular mask behavior

Given \(M=5\), \(W=1\), \(R=1\), \(A=0\), and a request for 2 bytes:

- Availability is 5.
- The first destination offset is 1.
- The next position is \((1+1)\mathbin{\&}5=0\).
- The second destination offset is 0.
- The final position is 1.

An implementation that substitutes remainder modulo 6 would not match this behavior.

### Dependency mutation before admission

Given a nonzero gate argument, a dependency result of zero, and a dependency that changes the mask before returning:

- Admission uses the changed mask.
- The pre-call mask is not preserved for the admission calculation.

Whether that dependency mutation is valid within the larger system is unknown.

## Uncertainties

The following are **documented_only**:

- The complete state-object definition, allocated size, field signedness outside this function, and byte order.
- Whether callers maintain power-of-two storage, masked positions, or a reserved slot.
- The semantic meaning of the occupancy adjustment and gate argument.
- All undocumented effects and requirements of F-15e46ffb3077.
- The valid address range, alignment requirements, and behavior of invalid or wrapping address calculations.
- Supported aliasing and concurrency arrangements beyond the accesses visible here.
- Whether abnormal interruptions are recoverable by the surrounding system.

The circular-buffer interpretation is **inferred intent**. It does not justify replacing bitwise masking with remainder, adding validation, normalizing the initial position, or changing the availability formula.

## Evidence

**Established static evidence:** The supplied representation contains one optional dependency call, one unsigned admission comparison, a byte-at-a-time loop with a masked 32-bit write-position update, and normal returns of 0 or 1. The specified field offsets, gate ordering, admission arithmetic, and byte-store-before-position-update ordering follow from those operations.

**Evidence limits:** The representation does not establish the dependency’s implementation, a complete object type, runtime memory validity, or exact machine-level ordering of independent expression reads.

**Runtime verification:** No runtime tests have been performed. All acceptance examples are conditional consequences of the static evidence, not observed executions.
