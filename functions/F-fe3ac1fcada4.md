<!-- Trusted provenance: F-fe3ac1fcada4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe3ac1fcada4

## Purpose

F-fe3ac1fcada4 exchanges three consecutive 32-bit storage units between two caller-supplied locations.

For disjoint regions, its net effect is an exchange of two 12-byte contents. The exchange is performed in increasing offset order, one 32-bit unit at a time. It is not established as a simultaneous exchange of two snapshots. Consequently, overlapping regions can produce a different result from an exchange based on the original contents of both regions.

The routine treats the stored values as uninterpreted 32-bit quantities. No arithmetic, comparison, byte-order conversion, or value-dependent branching is shown.

This contract describes behavior established by the supplied static representation. It does not claim runtime verification.

## Interface

**Inputs**

Two address-valued arguments, designated **A** and **B**. Each identifies the beginning of a region containing three consecutive 32-bit storage units:

| Unit | Byte offset from the argument | Width |
|---|---:|---:|
| First | 0 | 32 bits |
| Second | 4 | 32 bits |
| Third | 8 | 32 bits |

Each region therefore occupies 12 bytes, under the byte-addressed layout represented by the supplied source.

Both arguments are input/output locations: existing contents are read, and replacement contents are written. Neither argument is merely an output buffer.

**Output**

There is no explicit return value. Successful completion is represented by the writes to the supplied locations and normal return.

**Preconditions for the ordinary successful-memory contract**

- Both arguments designate live storage throughout the operation.
- All three 32-bit units at each argument are readable and writable.
- The relevant addresses satisfy the target environment’s requirements for these accesses.
- The address ranges are representable and valid in that environment.
- No concurrent actor modifies the relevant storage during the operation if the caller requires the deterministic results described below.
- The storage behaves as ordinary memory if the caller relies on the ordinary-memory examples.

The supplied representation does not establish pointer width, address-space size, an exact alignment requirement, or the handling of misaligned accesses. A 32-bit storage-unit width does not establish a 32-bit pointer width.

There is no visible non-overlap check. Identical arguments are consistent with the visible accesses. Partially overlapping arguments require special care because later reads can observe earlier writes.

## Behavior

### Ordered exchange

The routine processes offsets 0, 4, and 8 bytes, in that order.

At each offset, the established logical memory-access order is:

1. Read the 32-bit contents at A plus that offset and retain the value temporarily.
2. Read the 32-bit contents at B plus the same offset.
3. Write the value just read from B into A at that offset.
4. Write the retained value from A into B at that offset.

The complete exchange at one offset precedes the first read at the next offset.

Thus, the first unit is exchanged before the second unit is read, and the second unit is exchanged before the third unit is read. There is no visible initial read of all three units from either region.

For a normal completion, this amounts to six logical 32-bit reads and six logical 32-bit writes. These are the accesses expressed by the supplied representation, not a claim about the number or atomicity of physical bus transactions.

### Disjoint regions

When the two 12-byte regions are disjoint and ordinary memory remains stable except for this routine’s writes:

- A receives the three original units from B, in their original order.
- B receives the three original units from A, in their original order.
- The routine does not intentionally access storage outside those regions.

Every 32-bit bit pattern is handled identically. Signedness, numeric magnitude, embedded addresses, and possible higher-level object meaning do not affect the visible operation.

### Identical arguments

If A and B are the same address, each unit is read twice and written twice at the logical level.

For stable ordinary memory, the final contents are unchanged. This is not an early-return case: the supplied representation still performs the accesses.

That distinction matters for storage with externally visible read or write behavior. The unchanged final value alone is not a sufficient description for such storage.

### Partial overlap

Overlapping regions are not treated as independent snapshots. A write at an earlier offset may alter a location that will be read at a later offset.

The appropriate overlap rule is therefore the ordered exchange above, not the disjoint-region postcondition. In particular, the operation is not generally equivalent to saving both complete regions before performing any writes.

Whole-unit overlap examples appear below. For overlaps that do not preserve 32-bit unit boundaries, the representation does not supply enough target-memory detail to give a portable byte-level outcome. Such cases depend on access validity, byte order, and the semantics of overlapping machine accesses.

### Width and arithmetic

The transferred quantities are exactly 32 bits each. No widening, truncation, arithmetic on their contents, or explicit byte swapping is shown.

The only address displacements are the fixed offsets 0, 4, and 8 bytes. The routine has no caller-supplied length and no length calculation. No address-overflow validation is visible.

The temporary value preserves one 32-bit unit while the corresponding destination is overwritten. The routine does not visibly retain an entire region for later restoration.

### Completion and failure

There is no visible input validation, conditional rejection, error return, retry, rollback, or recovery path.

If an access is invalid or faults, the supplied representation does not establish the resulting exception, process behavior, or recoverability. Earlier writes may already have occurred. Callers must not assume all-or-nothing behavior.

Normal return follows the final write to B at offset 8.

## Side effects

The explicit side effects are writes to the three addressed units of A and the three addressed units of B.

For disjoint regions, these are six distinct 32-bit destinations. For overlapping regions, some destinations refer to the same underlying storage. The repeated writes remain part of the logical operation even when a value does not change.

The routine does not visibly:

- Allocate or release memory.
- Transfer ownership of either region.
- Change the argument addresses stored by the caller.
- Interpret stored values as references or invoke object-lifetime operations.
- Access a named global object.
- Perform input/output through another function.

The caller retains responsibility for the storage’s lifetime and validity. If a stored 32-bit quantity represents a resource handle or address in some external scheme, this routine only moves its bits; it does not visibly acquire, release, validate, or duplicate the referenced resource.

There is no synchronization mechanism in the supplied representation. Neither the complete operation nor an individual 32-bit access is established as atomic. Intermediate states can matter when storage is shared or externally observed.

The stated access ordering is the logical ordering evidenced by the supplied representation. Exact hardware visibility, compiler treatment of ordinary memory accesses, and memory-mapped device semantics are not established.

## Dependencies

No calls to other functions are visible. There are therefore no identified opaque function dependencies to list.

The routine depends on the execution environment for:

- Validity and lifetime of the supplied storage.
- The semantics of 32-bit reads and writes.
- Alignment requirements.
- Address calculation and address-space limits.
- Fault behavior.
- Concurrency and external-observation rules.

No unknown callee is assumed to preserve state, manage ownership, or perform cleanup, because no callee appears in the supplied representation.

## Acceptance examples

These examples are conditional specifications, not reports of executed tests. Unless otherwise stated, they assume valid, stable, ordinary memory, valid 32-bit accesses, and normal completion.

### Disjoint contents exchange

Initially:

- A contains the three units `[11, 22, 33]`.
- B contains the three units `[44, 55, 66]`.
- The regions do not overlap.

On return:

- A contains `[44, 55, 66]`.
- B contains `[11, 22, 33]`.

The values here label 32-bit contents; the operation does not depend on their numeric interpretation.

### Bit patterns remain intact

Initially:

- A contains `[0x00000000, 0xFFFFFFFF, 0x80000000]`.
- B contains `[0x12345678, 0x00000001, 0x7FFFFFFF]`.
- The regions are disjoint.

On return, each region contains the other region’s original three bit patterns, without numeric conversion or reordering.

### Same starting address

A and B identify the same three units, initially `[7, 8, 9]`.

On normal return, those units remain `[7, 8, 9]`. Acceptance also requires retaining the distinction between unchanged contents and absent accesses: the static operation still includes the reads and writes.

### Overlap by one whole unit

A four-unit backing region initially contains:

`[10, 20, 30, 40]`

A begins at the first unit, and B begins at the second unit. The final backing region is:

`[20, 30, 40, 10]`

The first exchange moves `10` into the second unit. The next exchange reads that updated unit, and the final exchange propagates the retained value into the fourth unit. This demonstrates why two original-region snapshots would give the wrong contract.

### Overlap by two whole units

A five-unit backing region initially contains:

`[10, 20, 30, 40, 50]`

A begins at the first unit, and B begins at the third unit. The final backing region is:

`[30, 40, 50, 20, 10]`

In particular, the third exchange reads the first exchange’s updated contents at A’s third unit.

### Storage outside disjoint regions

Given two valid disjoint regions surrounded by otherwise accessible ordinary storage, normal completion exchanges the six addressed units without any explicit write to surrounding storage.

### Invalid storage

If either region includes inaccessible storage, there is no specified successful result or error code. Acceptance must not require rollback or assume that contents preceding the failing access remain unchanged.

## Uncertainties

**Established static behavior:** three ordered exchanges of 32-bit units at offsets 0, 4, and 8, with no visible calls, branches, validation, or explicit return value.

**Inferred intent:** the routine likely exchanges a fixed-size, 12-byte aggregate. The representation does not establish that aggregate’s type, field meanings, or higher-level invariants.

**documented_only — target environment:** pointer width, byte order, precise alignment requirements, physical access granularity, atomicity, and memory-order guarantees are unknown.

**documented_only — unusual storage:** behavior for device registers, volatile storage, concurrent mutation, or non-unit-aligned overlap is not established beyond the visible logical access sequence.

**documented_only — failure handling:** fault delivery, faulting-access effects, and whether execution can resume after an invalid access are unknown.

**documented_only — higher-level validity:** moving these bit patterns may violate invariants of an enclosing object or resource protocol. No such protocol is supplied, and the routine visibly performs no associated validation.

## Evidence

The supplied static representation establishes two address arguments, 32-bit accesses to three consecutive units, a single-unit temporary, and an unconditional sequence of three exchanges followed by return.

The disjoint, identical-address, and whole-unit overlap outcomes above follow from that access sequence under their stated memory assumptions.

No runtime tests have been performed. No binary-level tracing, target-platform validation, or external caller contract was supplied.
