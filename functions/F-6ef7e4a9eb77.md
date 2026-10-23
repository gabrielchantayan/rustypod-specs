<!-- Trusted provenance: F-6ef7e4a9eb77; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6ef7e4a9eb77

## Purpose

F-6ef7e4a9eb77 compares two byte sequences for equality over a caller-supplied extent. It returns a four-byte scalar with value **1** when the permitted comparisons finish without a mismatch, and **0** when a compared pair differs.

This is an equality predicate, not a lexical ordering operation. A mismatch produces the same result regardless of which byte has the greater numeric value. Zero-valued bytes have no special terminating role.

The established useful contract covers nonnegative extents. Negative extents do not represent an empty comparison: they enter the comparison path, and their subsequent behavior depends on arithmetic semantics and accessible memory.

## Interface

### Inputs

The recovered interface supplies three arguments:

- **First sequence address:** a pointer to character-sized storage.
- **Second sequence address:** another pointer to character-sized storage.
- **Comparison extent:** an integer controlling how many matching byte pairs may be examined before success is returned.

Neither address carries an accompanying allocation size or ownership descriptor. The function performs no visible address, capacity, or extent validation.

The sequence element type is a character-sized unit. On an eight-bit-byte target, each comparison consumes one octet from each sequence. Character signedness does not affect equality between two stored character values interpreted under the same character type.

The extent is represented as an `int` in the supplied recovered interface. Its exact ABI width is not independently established by that spelling alone. Under a 32-bit integer interpretation, positive values range through 2,147,483,647, and negative values require the arithmetic qualifications described below. The contract must not silently reinterpret a negative extent as a conventional unsigned buffer length.

### Output

The recovered return type explicitly denotes a four-byte scalar. The only explicit return values are:

- **1:** the current remaining extent was zero at an iteration boundary.
- **0:** the two bytes read during an iteration were unequal.

No byte difference, mismatch position, or updated address is returned. There is no explicit error result.

### Preconditions for the ordinary nonnegative case

For a nonnegative extent, both addresses must support every byte read that the comparison actually reaches. Each accessed byte must be readable, and the pointer advancement associated with that access must be valid under the applicable execution model.

Providing two readable regions of at least the requested extent is a sufficient ordinary buffer precondition. Early termination can require fewer reads, but callers relying on that fact must ensure that a mismatch occurs before an otherwise invalid access.

For extent zero, the function performs no sequence-memory reads and does not advance either sequence address. Null or otherwise unusable sequence addresses therefore do not cause a dereference within this function on that path. This does not establish broader language or ABI rules about constructing and passing arbitrary pointer values.

Ownership remains with the caller. No ownership transfer is expressed by the interface.

## Behavior

### Nonnegative extents

For a positive extent, corresponding positions are compared in increasing address order, beginning at the two supplied addresses.

At each iteration, the function first tests whether the remaining extent is zero and decreases its local remaining extent by one. If the tested value was zero, it returns 1 without reading either sequence during that iteration.

Otherwise, it reads one character from the first sequence, then one character from the second sequence. Both local sequence positions advance by one character-sized unit before the equality outcome determines whether comparison continues.

An unequal pair ends the operation immediately with result 0. No later pair is read.

An equal pair permits another iteration. After the last permitted pair matches, the next iteration recognizes that the remaining extent is zero and returns 1. Consequently:

- An extent of zero returns 1 without sequence reads.
- A positive extent whose entire compared prefix matches returns 1.
- A positive extent containing a mismatch returns 0 at the first mismatching position.
- Data beyond the permitted extent does not affect the result and is not explicitly read by the supplied body.

For an extent of \(n>0\), if the first mismatch is at zero-based position \(j<n\), the function reads exactly \(j+1\) characters from each sequence. If all \(n\) positions match, it reads exactly \(n\) characters from each sequence.

These counts describe the explicit accesses in the supplied body, not speculative processor activity or transformations that an unknown compiler might perform.

### Byte interpretation and termination

The comparison is equality-only. It does not interpret characters as text, perform case conversion, decode multibyte characters, or recognize delimiters.

In particular, a matching pair of zero bytes does not terminate comparison. If the extent includes later bytes, those later bytes remain eligible for comparison. Similarly, equality at an earlier position does not compensate for a later mismatch.

No subtraction between the two byte values is used to determine the result. Character signedness therefore cannot change the result for a given pair of stored bytes merely by changing their ordering interpretation.

### Zero and negative arithmetic

The extent decrement occurs even on the iteration that observes zero. That decrement changes only local state and does not cause an additional sequence read. With ordinary signed integer arithmetic, decrementing zero to negative one is representable.

A negative initial extent does not satisfy the zero test. The function therefore attempts its first pair of reads, just as it does for a positive nonzero extent. If that pair differs, the explicit result is 0.

If successive pairs remain equal, the negative remaining extent continues decreasing. There is no visible rejection, saturation, or normalization of negative input.

Its eventual behavior is not portable from the recovered representation alone:

- Under signed arithmetic rules that do not permit overflow, continued matching can reach an arithmetic operation outside the signed type’s range. No defined continuation is established past that point.
- Under fixed-width wrapping arithmetic, the remaining extent can eventually cycle through the representable values and reach zero, provided execution survives all preceding reads and address advances.
- Under a 32-bit wrapping interpretation, an initial extent of negative one would require 4,294,967,295 matching comparison iterations before a subsequent zero check could return 1.

The wrapping case is a conditional arithmetic consequence, not evidence that such a run is valid or practical. Memory access or pointer arithmetic may fail much earlier. Negative-input completion behavior is therefore **documented_only**, except where execution reaches an explicit mismatch without encountering an earlier invalid operation.

## Side effects

The supplied body explicitly reads sequence memory but does not write to either sequence. It modifies only its local extent and local copies of the two addresses.

Advancing those local address copies does not update caller-held pointer variables. The function exposes neither its final positions nor its remaining extent.

The visible access order within a comparison iteration is:

1. Read the current character from the first sequence.
2. Read the current character from the second sequence.
3. Advance both local positions.
4. Decide whether to continue based on equality.

This is the order represented by the supplied body. It is not a guarantee about hardware bus transactions, compiler reordering, or memory-mapped device accesses. The interface does not declare volatile or atomic access.

Aliasing is not expressly prohibited. The two addresses may identify the same storage or overlapping regions without introducing an explicit write conflict, because the function contains no sequence writes. Concurrent modification remains outside the established contract: there is no synchronization or coherent-snapshot mechanism.

There are no visible allocations, releases, global accesses, callbacks, or external calls. Invalid reads have no explicit recovery path. Their consequences depend on the surrounding execution environment rather than a defined error return.

## Dependencies

No dependency invocation is visible in F-6ef7e4a9eb77.

Its result depends on the supplied extent, the character values observed at the reached addresses, and the execution model’s rules for integer arithmetic, pointer advancement, and memory access.

No external ownership, cleanup, or global-preservation behavior is assumed.

## Acceptance examples

These examples are conditional contracts derived from static inspection. They are not runtime test results. Except where noted, assume stable readable storage, valid pointer advancement, eight-bit bytes, and nonnegative extents representable by the interface.

| Inputs and condition | Required behavior |
|---|---|
| Extent zero; neither sequence address is dereferenceable | Return 1 without reading sequence memory. |
| Both sequences begin with `41 42 43`; extent three | Read three corresponding pairs and return 1. |
| First sequence begins with `41 42`; second begins with `41 43`; extent two | Read both pairs and return 0 after the second pair. |
| The first pair differs; extent is greater than one | Read only the first pair and return 0. |
| Both sequences begin with `41 00`, followed respectively by `42` and `43`; extent three | Continue past the matching zero bytes and return 0 at the third pair. |
| Both sequences match for two bytes but differ at the third; extent two | Return 1 without explicitly reading the third pair. |
| Both addresses identify the same stable region; extent four | Read four pairs from that region and return 1, assuming all accesses and advances are valid. |
| Extent negative one; the first pair differs | Return 0 if the initial decrement and both reads execute validly. |
| Extent negative one; every reached pair matches | Do not treat the extent as empty. Eventual completion is documented_only because arithmetic and memory limits are unresolved. |

An unreadable address with a positive extent has no specified Boolean error outcome. In particular, callers must not expect invalid storage to produce 0.

## Uncertainties

**Established static behavior:** zero-before-read termination, local decrementing, ordered character-pair reads, early mismatch termination, no zero-byte special case, and explicit four-byte results of 0 or 1.

**Inferred intent:** bounded byte-sequence equality. That description fits the nonnegative behavior, but does not establish the original API name, intended caller population, or a deliberate policy for negative extents.

**Documented_only unknowns:**

- Exact ABI width and signed-overflow behavior of the recovered integer parameter.
- Whether the target’s character-sized storage unit is eight bits.
- Compiler transformations and actual machine-level access ordering.
- Pointer validity and failure behavior outside ordinary readable objects.
- Any caller-side validation, synchronization, or extent restrictions.
- Whether negative extents are forbidden by an external contract unavailable here.

The function itself supplies no evidence resolving these points. In particular, absence of a negative-input check does not establish support for negative lengths.

## Evidence

The supplied static body contains a repeating zero test and local decrement, two character reads, local address advances, an equality-controlled continuation, and explicit returns of 1 and 0. It contains no dependency calls, sequence writes, allocation operations, or explicit error handling.

This contract describes those visible operations and separates them from target-dependent arithmetic and inferred intent. No runtime tests have been performed.
