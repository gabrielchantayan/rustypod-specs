<!-- Trusted provenance: F-1deedbbf641a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1deedbbf641a

## Purpose

F-1deedbbf641a searches an indexed collection for the first eligible entry whose stored 32-bit key equals a masked form of the supplied search value.

A successful result contains the matching entry’s zero-based index with bit 31 forced to one. An unsuccessful search returns zero. The function invokes F-603a53835f04 before searching and F-9b718979506d before returning through either normal search outcome.

**Established behavior:** eligibility means a designated byte is nonzero; matching means exact equality with the search value after its lowest seven bits have been cleared.

**Inferred intent:** the representation appears to combine a success marker with an index, and the mask appears to group search values into 128-value ranges. Neither interpretation establishes additional constraints on collection contents or permitted indices.

## Interface

### Inputs

The interface has four arguments:

1. **Collection base:** a 32-bit address-bearing value locating the collection’s memory representation.
2. **Search value:** an unsigned 32-bit value. The comparison target is this value AND `0xFFFFFF80`.
3. **Unused argument:** a 32-bit value that is not referenced in the supplied function body.
4. **Initial auxiliary value:** a 32-bit value copied into a local four-byte slot before the first dependency invocation.

No meaning is established for the unused argument or the initial auxiliary value beyond these observable uses.

### Collection layout

Offsets are byte offsets relative to the collection base.

| Location | Width | Meaning in this function |
|---|---:|---|
| Base + `0x468` | 32 bits | Unsigned entry count |
| Base + `0x46C` + index × `0x3C` | 32 bits | Stored key for that entry |
| Base + `0x485` + index × `0x3C` | 8 bits | Eligibility indicator for that entry |

The stride between corresponding fields of consecutive entries is `0x3C`, or 60 bytes. The eligibility byte is 25 bytes after the corresponding key address.

These offsets identify the fields actually accessed. They do not establish the beginning, full layout, allocation size, or ownership of an enclosing object or entry.

The key is compared as a complete unsigned 32-bit quantity. The eligibility byte is tested only for zero versus nonzero; its signed interpretation does not affect that test.

### Output

The return value is unsigned and 32 bits wide:

- **No match:** `0x00000000`.
- **Match at index k:** k bitwise OR `0x80000000`.

A successful return is therefore always nonzero and always has bit 31 set.

The function does not verify that indices fit in 31 bits. Recovering the index by clearing bit 31 is valid only under an additional constraint that the matching index is below `0x80000000`. Without that constraint, different indices can produce the same result.

### Preconditions for normal completion

For execution to complete normally:

- Both dependencies must return normally.
- The memory locations actually reached by the search must be readable with the required access widths.
- Address calculations must produce valid locations in the target execution environment.
- The dependency calls must accept the supplied arguments and the address of the local four-byte slot.

The function contains no explicit null check, allocation-size check, address-range check, or dependency-status check.

## Behavior

### Entry and auxiliary processing

Before examining the collection, the function copies the fourth input into a local four-byte slot. It then calls F-603a53835f04 with:

- The address of that slot.
- Three additional arguments, each zero.

The dependency’s return value is not used to determine the search result.

The slot remains live for the subsequent call to F-9b718979506d. The caller does not explicitly reinitialize it between dependency calls. If the first dependency changes the slot, that changed content remains available to the second dependency, subject to any other effects not established here.

The search result and index are initialized to zero after the first dependency returns.

### Search order and memory reads

Entries are considered in ascending index order, beginning at zero.

At each candidate index, the function first reads the unsigned 32-bit count at base + `0x468`. If the current index is greater than or equal to that observed count, the search ends without reading that entry’s eligibility byte or key.

Otherwise, the function reads the entry’s eligibility byte:

- If the byte is zero, the entry is skipped without reading its key.
- If the byte is nonzero, the function reads the entry’s 32-bit key and compares it with the masked search value.

The comparison target is the search value with bits 0 through 6 cleared. The stored key is **not** masked.

Consequently, a stored key with any of its lowest seven bits set cannot match, even if clearing those bits would make it equal to the target.

If an eligible entry’s key matches, the function selects that index and ends the search immediately. Later entries are not examined.

If the entry is ineligible or its key differs, the index increases by one and the count check is repeated.

### Count handling

The supplied representation places the count read inside the search loop. The function does not explicitly take a single count snapshot for the entire search.

For a stable count of N:

- N equal to zero causes no entry fields to be read.
- A search without a match examines indices zero through N − 1.
- A match at index k prevents inspection of later entries and prevents the next loop count check.

If count observations change during execution, the next observed count controls whether the current index is examined. No locking, snapshot, retry, or consistency protocol is present in the supplied body.

This describes the explicit accesses in the supplied representation, not a guarantee about compiled machine-load ordering or the validity of unsynchronized concurrent modification.

### Result formation and exit

On a match, the function forms the result by forcing bit 31 of the matching index to one.

On exhaustion, the result remains zero.

Both normal outcomes converge on a call to F-9b718979506d with the address of the same local four-byte slot used for F-603a53835f04. The selected search result is then returned.

The second dependency’s return value is not used. No separate error return is established for either dependency.

### Integer and address boundaries

The index, count, masked target, and result are 32-bit unsigned quantities.

The index advances only after the count check permits an entry access. Because the count cannot exceed `0xFFFFFFFF`, an accessed index cannot exceed `0xFFFFFFFE`. An increment can therefore reach `0xFFFFFFFF`, but the next count check must terminate the search. The represented loop does not require an index wrap back to zero.

Address calculations are a separate concern. Multiplying a large index by 60 and adding the field offsets can exceed a 32-bit address range. No check prevents overflow, wraparound, or an invalid resulting address. The supplied representation does not establish portable behavior for such cases.

A numerically permitted count is therefore not sufficient to establish safe memory access.

## Side effects

### Explicit local writes

The function explicitly:

- Copies the fourth input into a four-byte local slot.
- Initializes and updates its local search index.
- Initializes and, on success, replaces its local return value.

It contains no explicit write to the collection count, entry keys, or eligibility bytes.

### Dependency effects

Both dependencies receive the address of the local slot. Their implementations are unavailable here, so their effects are unknown.

In particular, this contract does not assume that either dependency:

- Preserves global or externally reachable state.
- Leaves the collection unchanged.
- Allocates or releases memory.
- Acquires or releases a lock.
- Retains or does not retain the supplied address.
- Returns normally under all conditions.

The local slot’s address is supplied while that slot is live. No supported lifetime beyond this function’s invocation is established.

### Failure handling and ownership

The function performs no explicit allocation, deallocation, or collection-ownership transfer. The collection base is used for reads, but this alone establishes no ownership contract.

There is no explicit recovery path for invalid memory, arithmetic-related address failure, or a dependency that does not return normally. In particular, the final dependency call is reached through normal search completion; it is not established as exception-safe or fault-safe cleanup.

## Dependencies

| Opaque function ID | Invocation | Established use |
|---|---|---|
| F-603a53835f04 | Once before the search, with the local slot’s address followed by three zero arguments | Receives access to the slot initialized from the fourth input; return value is ignored |
| F-9b718979506d | Once after either a match or exhaustion, with the same slot address | Runs before the normal return; return value is ignored |

The pairing suggests auxiliary setup and teardown, but that interpretation is inferred intent only. The dependency identifiers do not establish locking, resource management, ownership, or global-state guarantees.

## Acceptance examples

These examples are conditional static expectations, not reports of executed tests. Unless stated otherwise, they assume valid addresses, stable collection contents during the search, and dependencies that return normally without changing the relevant collection fields.

### Empty collection

Given a count of zero:

- The first dependency runs.
- The count is read.
- No entry eligibility byte or key is read.
- The second dependency runs.
- The result is `0x00000000`.

### Match at index zero

Given a count of at least one, a search value of `0x123456FF`, a nonzero eligibility byte at index zero, and a stored key of `0x12345680`:

- The target is `0x12345680`.
- Index zero matches.
- The result is `0x80000000`.
- No later entry is examined.

### Ineligible key is not read

Given a count of one and a zero eligibility byte at index zero:

- The key at index zero is not read by the explicit search logic.
- The next count check ends the search.
- The result is zero.

This conditional outcome does not require the skipped key value to be meaningful or matching.

### Stored keys are not normalized

Given a count of one, an eligible entry, a search value of `0x0000017F`, and a stored key of `0x00000101`:

- The target is `0x00000100`.
- The stored key differs from that target.
- The result is zero.

Changing the stored key to `0x00000100` produces a match at index zero.

### First matching entry wins

Given eligible matching entries at indices two and five, with no earlier eligible match:

- The result is `0x80000002`.
- Index five is not examined.
- No duplicate-key error is reported.

### Search values sharing the lowest seven-bit range

Search values `0x00000200` and `0x0000027F` both produce target `0x00000200`.

Under otherwise identical stable conditions, they therefore select the same first matching entry or both return zero.

### High-index encoding limitation

If valid addressing permits a match at index `0x80000000`, the returned value is `0x80000000`, identical to the encoding for a match at index zero.

Thus success remains distinguishable from failure, but the returned value alone does not uniquely identify every possible 32-bit index.

### Count changes before a later check

If an unsuccessful examination of index zero is followed by a count observation of one:

- The current index is then one.
- The search ends without reading index one.
- The result is zero.

This example states the represented loop’s decision rule; it does not establish concurrent access as safe.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the collection, stored keys, and eligibility byte.
- The purpose of the third argument.
- The meaning and permitted contents of the fourth argument.
- The full signatures, effects, failure behavior, and ownership rules of both dependencies.
- Required object allocation size, alignment, and byte order.
- Whether the surrounding system constrains counts or indices to a smaller range.
- The target’s exact behavior for overflowing address calculations.
- Whether external mutation is permitted and, if so, what synchronization governs it.
- Whether the auxiliary dependency pairing represents locking, resource lifetime, instrumentation, or another mechanism.

No runtime tests have been performed. No concurrency, fault-handling, or dependency behavior has been experimentally verified.

## Evidence

The supplied static function representation establishes:

- Four inputs and a 32-bit unsigned return value.
- A local four-byte slot initialized from the fourth input.
- An initial dependency call receiving that slot’s address and three zero arguments.
- An unsigned count comparison before each candidate entry.
- A 60-byte stride between corresponding entry fields.
- A nonzero eligibility test before the key read.
- Exact key comparison against the search value masked by `0xFFFFFF80`.
- Ascending first-match selection.
- Zero for exhaustion and an index with bit 31 set for success.
- A shared final dependency call before both normal return outcomes.

The purpose descriptions and possible setup/teardown interpretation are explicitly inferred. Dependency internals, broader object invariants, and runtime outcomes are not established by the supplied evidence.
