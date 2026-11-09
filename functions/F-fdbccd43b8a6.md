<!-- Trusted provenance: F-fdbccd43b8a6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdbccd43b8a6

## Purpose

F-fdbccd43b8a6 constructs an array of token pointers while modifying a byte string in place. A second byte string specifies delimiter values. A final plus byte in that specification enables consumption of consecutive delimiters; that final plus is a control marker rather than a delimiter entry.

The function resets the output token count on every invocation. Successful finalization appends a null pointer after the counted entries. Early successful returns do not perform this finalization.

The supplied length limits where a token may **start**, not how far token scanning or delimiter consumption may proceed.

The tokenizer interpretation is inferred intent. The comparisons, accesses, mutations, and dependency-call ordering below are established by the supplied static body. The table-preparation operation, storage-management dependency, and shared replacement target have unresolved semantics.

## Interface

### Arguments and representation

The function accepts four arguments:

1. **Output descriptor:** writable storage containing three consecutive 32-bit fields:
   - Offset 0: output pointer-array address.
   - Offset 4: capacity.
   - Offset 8: token count.
2. **Delimiter specification:** a pointer to a zero-terminated byte sequence, or null.
3. **Source:** a pointer to mutable, zero-terminated bytes.
4. **Length:** a signed 32-bit value added to the initial source pointer to establish the token-start endpoint.

Pointer-array entries occupy four bytes. The descriptor address field and shared replacement pointer also have four-byte representations in the supplied view. Bytes are interpreted as unsigned values from 0 through 255; matching is not character-, locale-, or encoding-aware.

The token count excludes the final null pointer. Successful finalization requires room for that terminator as well as all counted entries.

### Return value

The signed 32-bit result is:

- **0:** no reported error, including early exits without a newly terminated array.
- **6:** length is nonzero, the initial source byte is nonzero, and the delimiter specification is null or begins with zero.
- **A nonzero status from F-9453bbba4fe8:** propagated when a storage-management call fails.

A zero return does not by itself establish that tokens were produced, that pointer-array storage was written, or that finalization occurred.

### Preconditions

For useful operation without invalid accesses:

- The descriptor must be writable even when length is zero.
- With nonzero length, the initial source byte must be readable.
- An examined delimiter specification must be readable through its terminating zero.
- Token scanning and delimiter consumption must have readable source storage beyond every position they inspect; overwritten delimiter positions must be writable.
- Output storage must support every performed four-byte slot write.
- The shared replacement global must be readable whenever the access rules below require it, including successful finalization with no pending trailing token.
- Descriptor contents, arithmetic, addressing, and dependency behavior must support the accesses described.
- Aliasing must not invalidate later reads or writes.

The function does not validate these conditions. Length is not a comprehensive memory bound. No checked overflow or general post-dependency capacity validation is visible.

## Behavior

### Initial reset and eligibility

The first visible memory operation writes zero to the descriptor’s token count.

If length is zero, the function returns zero without reading source bytes or delimiter bytes, without reading the shared replacement global, and without writing a pointer-array terminator.

Otherwise, it reads the first source byte. If that byte is zero, it returns zero without inspecting the delimiter specification or performing finalization.

Only for a nonzero length and initially nonempty source does it validate the delimiter specification. A null pointer or initial zero byte produces return value 6. These early paths leave existing pointer-array entries untouched.

### Delimiter-table construction

Eligible input causes a call to an external operation with a temporary 32-byte region and the size 32. The supplied mapping does not identify that operation with an opaque function ID.

The function then processes the delimiter specification as unsigned bytes. A value selects one of 32 table bytes using its upper five bits and one of eight membership bits using its lower three bits.

For each nonzero specification byte:

- A plus byte, value 43, causes a read of the following specification byte.
- If that following byte is zero, consecutive-delimiter mode is enabled and this plus is not inserted into the table.
- Otherwise, the current byte’s membership bit is set.
- Processing advances to the next specification byte.

Repeated delimiter entries only set already-set bits. A plus before the final position is a literal delimiter. Thus two plus bytes followed by zero both insert plus as a delimiter and enable consecutive-delimiter mode.

The preparation operation’s effect on the temporary region is unknown. Static evidence establishes which membership bits are set, not that all other bits begin clear. Interpretation as matching *only* the specified delimiters depends on the preparation operation clearing the region.

### Token-start admission

After table construction, the function adds length to the initial source pointer to form an endpoint.

A token is admitted only if the current source pointer compares below that endpoint and the current source byte is nonzero. The pointer comparison precedes the source-byte read in this condition.

There is no explicit rejection of negative lengths. Negative lengths, address wraparound, and out-of-object endpoints must not be assigned portable behavior from this evidence alone.

If this loop admits no tokens, the eligible-input path still proceeds to finalization.

### Boundary scanning

From each admitted token start, scanning advances until it reads either zero or a byte whose membership bit is set.

This scan does not consult the length-derived endpoint. An admitted token can extend beyond that endpoint, and its terminating delimiter can be overwritten beyond it.

A delimiter at the token start creates an empty-token position; leading delimiters are not simply skipped.

### Capacity and token recording

Before recording a token, the function compares the descriptor’s current count and capacity for equality.

If equal, it calls F-9453bbba4fe8 with the descriptor and a requested slot count of count plus one. A nonzero result returns immediately, before this token’s count increment, pointer-slot write, or boundary mutation. Earlier work remains.

The test is equality, not a greater-than-or-equal check. Malformed descriptors with count exceeding capacity are not generally corrected.

After any successful capacity call, token recording uses the descriptor’s then-current fields. The earlier array address and count are not assumed to survive the call unchanged.

Recording has this observable order:

1. Read the current count and increment the descriptor count.
2. Select the token pointer.
3. Write the selected pointer to the four-byte array slot indexed by the pre-increment count.

For an ordinary nonempty token, the selected pointer is its source start. No shared replacement-global read is performed for that loop record.

For an empty-token position, the shared replacement global is read **after the count increment and before the pointer-slot write**. Its loaded value supplies that entry. The visible selection condition is that the boundary address compares at or below the token start; for valid forward scanning, the empty case has equal addresses.

The count increment and pointer write are separate operations, with no atomicity guarantee.

### Source mutation and trailing-token state

After recording the token pointer, the function processes its boundary.

Without consecutive-delimiter mode:

- A zero boundary byte is not changed.
- A nonzero boundary delimiter is replaced with zero, and the current position advances one byte.

With consecutive-delimiter mode:

- Each consecutive nonzero delimiter byte is replaced with zero.
- The position advances after each replacement.
- Consumption stops at zero or a byte not recognized as a delimiter.

The delimiter-run scan is not bounded by the token-start endpoint.

A trailing empty token becomes pending precisely when processing advanced beyond the original boundary and the resulting current byte is zero. The comparison establishing advancement occurs before that zero-byte check. Therefore, a read after advancement can occur at or beyond the endpoint.

This state is updated for every processed token. Finalization uses the last such result; it does not independently inspect an otherwise unprocessed suffix for trailing delimiters.

### Finalization and shared-global access

Finalization computes the count plus a possible pending trailing entry and compares its unsigned 32-bit interpretation with the unsigned 32-bit capacity.

If that sum is strictly below capacity, no final capacity call is made. Otherwise, F-9453bbba4fe8 receives a requested slot count of count plus pending trailing entry plus one.

A failed final capacity call prevents all remaining finalization operations: no finalization-stage shared-global read, no appended trailing entry, and no final null-pointer write.

On either successful route—capacity already sufficient or a successful capacity call—the following ordering applies:

1. **Read the shared replacement global unconditionally.**
2. If a trailing entry is pending, read the current descriptor count, increment it, and write the loaded replacement pointer into the corresponding slot.
3. Write a four-byte zero into the slot indexed by the resulting descriptor count.

The unconditional read in step 1 occurs even when no trailing entry is pending. It also occurs if eligible input reaches successful finalization without admitting any token.

When a trailing entry is pending, its count read and increment occur **after** this global read. This differs from an empty-token record inside the loop, where the count increment precedes the global read.

These are separate global reads. No replacement value is cached across token records or across F-9453bbba4fe8 calls. A successful dependency may change the global before a later read; the function does not establish preservation of that value.

After successful finalization, the function returns zero.

## Side effects

### Descriptor and pointer array

The count is always reset first. Each recorded loop token increments it before its slot is written. A pending trailing entry is appended only during successful finalization.

Early no-work returns and delimiter-validation errors do not clear old pointer-array entries. A caller cannot treat those entries as newly produced output merely because the return value is zero.

A capacity failure can expose a partial result:

- Previously incremented counts and written entries remain.
- Previously overwritten delimiters remain zero.
- A pending trailing entry may be absent.
- The array need not be null-terminated.
- Additional dependency effects remain unknown.

No rollback is visible.

### Source and delimiter specification

Direct source writes replace recognized delimiters with zero after recording the corresponding token pointer. Token payload bytes are not directly overwritten.

Consecutive-delimiter mode may zero several delimiter bytes for one recorded token. It does not suppress the initial empty-token entry when an admitted token starts on a delimiter.

The function performs no direct writes to the delimiter specification. Overlap with other writable storage can nevertheless affect later observations; behavior must follow the actual access order rather than assuming independent buffers.

### Shared replacement global

Direct reads occur:

- Once per empty-token record inside the loop, after that record’s count increment and before its slot write.
- Once on every successful finalization path, after the capacity decision and any successful capacity call, before any optional trailing-entry count read or write.

A nonempty loop token does not require a loop-stage global read. Early returns do not reach the finalization read. A failed capacity call prevents the subsequent read at that stage, but does not undo earlier reads.

The function does not directly write this global.

### Ownership and lifetime

The function stores source pointers and replacement-pointer values; it does not copy token payloads.

Source-backed entries depend on the lifetime and subsequent contents of the caller’s source storage. The replacement target’s bytes, lifetime, mutability, and release obligations are unknown. Its interpretation as an empty string is inferred intent, not established content.

No direct allocation or deallocation is visible outside dependencies. Their ownership behavior must not be inferred.

## Dependencies

### F-9453bbba4fe8

Observed interface:

- Inputs: descriptor and a 32-bit requested slot count.
- Result: 32-bit status, zero permitting continuation and nonzero propagated as failure.

Invocation points:

- Before a loop token record when count equals capacity.
- During finalization when count plus a pending trailing entry is not unsigned-strictly-below capacity.

Inferred intent: pointer-array capacity management.

Its allocation policy, ownership effects, supported capacities, overflow handling, and effects on globals or other reachable memory are **documented_only**. Successful status does not establish preservation of the descriptor, source, shared replacement global, or any other accessible state.

### Table-preparation operation

The observed call supplies a temporary 32-byte region and size 32. No opaque function ID or implementation is supplied for it.

Clearing the region is plausible intent, but its semantics and other memory effects are **documented_only**.

### Shared replacement global

This is a data dependency, not a supplied callable dependency. It provides four-byte pointer values under the exact read conditions described above.

Its initial value, target contents, lifetime, and susceptibility to dependency-induced changes are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests. Unless an example states otherwise, assume valid disjoint storage, representable arithmetic, a cleared delimiter table, sufficient capacity, and a replacement pointer denoting a valid empty string. Dependencies must preserve the source and other state required by each example, except for their stated capacity-management effects. Length admits all discussed token starts unless specified.

1. **Zero length:** count becomes zero and the return is zero. Source, delimiter specification, and shared replacement global are not read. No array terminator is written.

2. **Initially empty source, nonzero length:** count becomes zero and the return is zero even with a null delimiter pointer. Neither finalization nor its unconditional global read occurs.

3. **Initially nonempty source, null or empty delimiter specification:** count becomes zero and return value is 6. No direct source mutation, token-slot write, or replacement-global read occurs.

4. **Source `red,blue`, specification `,`:** two source-backed entries result, the comma becomes zero, and a null pointer follows. Neither loop record reads the replacement global. Successful finalization nevertheless reads it once, even though no trailing entry uses the value.

5. **Source `,red`, specification `,`:** the first count increment precedes a replacement-global read and the first slot write. The second entry points to `red`. Successful finalization performs another replacement-global read without appending a trailing entry.

6. **Source `red,,blue`, specification `,`:** entries represent `red`, an empty position, and `blue`. Both commas become zero. The empty record reads the global after its count increment; finalization reads it separately.

7. **Source `red,,blue`, specification `,+`:** entries represent `red` and `blue`. Both commas are consumed during the first boundary’s delimiter run. Finalization still reads the replacement global.

8. **Source `,,red`, specification `,+`:** an initial replacement entry remains, followed by `red`. Collapsing the leading run does not remove the already admitted empty position.

9. **Source `red,,`, specification `,+`:** entries represent `red` and one trailing replacement entry. Both commas become zero. The finalization global read occurs before reading and incrementing the count for that trailing entry.

10. **Source `,`, specification `,`:** one replacement entry is recorded inside the loop and another during finalization. The two entries come from separate global reads, not necessarily equal pointer values.

11. **Source `red+blue`, specification `+`:** the final plus is only a mode marker. With a cleared table, the whole source is one token and its plus remains unchanged. Finalization still reads the replacement global.

12. **Source `red++blue`, specification `++`:** plus is a delimiter and consecutive-delimiter mode is enabled. The two entries represent `red` and `blue`.

13. **Source `red,blue`, specification `,`, length 1:** scanning from the admitted initial position reaches and zeroes the comma. The start of `blue` is not admitted. Finalization terminates the one-entry array and reads the replacement global despite having no trailing entry.

14. **Global changes during final growth:** for source `,`, suppose the loop records replacement value A, then the final capacity call succeeds and changes the global to B while preserving the descriptor and source state needed here. Finalization reads B after that call. The resulting two entries contain A and B respectively.

15. **Final capacity failure:** existing token entries and source mutations remain. The exact nonzero status is returned. The finalization global read, pending trailing-entry append, and null-pointer write do not occur.

## Uncertainties

- **documented_only:** whether table preparation clears all membership bits.
- **documented_only:** the shared replacement target’s bytes, lifetime, ownership, and mutability.
- **documented_only:** dependency effects on the replacement global, descriptor, source, and other reachable memory.
- **documented_only:** behavior for invalid addresses, disruptive overlap, negative lengths, wrapped addresses, count overflow, and malformed capacity values.
- **Inferred intent:** a growable token-pointer array with a shared empty-string representation.

Count increments, capacity requests, and slot addressing use 32-bit representations in the supplied view. The final capacity comparison explicitly uses unsigned interpretations; the earlier check uses equality. No safe overflow policy can be inferred.

The global-read ordering is established, not uncertain. Whether the loaded values remain equal across reads is unresolved.

## Evidence

This contract is based solely on the supplied static function body and dependency-ID mapping.

Static evidence establishes the descriptor layout, four-byte slots, reset and validation order, delimiter membership operations, terminal-plus handling, unbounded inner scans, capacity-call conditions, count and slot-write order, source mutation, trailing-entry decision, final terminator write, and returned statuses.

It also establishes separate replacement-global reads: conditional reads for empty loop records after count increments, and an unconditional read on every successful finalization path after any required successful capacity call and before optional trailing-entry count operations.

The supplied evidence does not establish dependency implementations or replacement-target contents. No runtime tests have been performed.
