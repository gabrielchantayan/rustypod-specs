<!-- Trusted provenance: F-d6a62f9f11ee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6a62f9f11ee

## Purpose

F-d6a62f9f11ee reports whether either of two selected bits is present in the combined contents of three memory words obtained through shared state.

On the ordinary path, it reads one 32-bit unsigned word at each of three byte offsets—0, 32, and 64—from a base address obtained through a shared address slot. It combines the words with bitwise inclusive OR and returns whether the result has any bit in common with the mask 0x0000000C.

The two selected bits are bit 2 and bit 3, counting the least significant bit as bit 0. Either bit is sufficient; both need not be set.

If the initial shared base address is zero, the function calls F-a817d8d58c10 with the integer argument zero. The supplied static evidence marks that call as non-returning. Its actual termination mechanism and other effects are not supplied.

The apparent intent is a predicate over three regularly spaced entries. That interpretation is inferred, not an established description of the surrounding data structure.

## Interface

**Explicit inputs:** None.

**Implicit inputs:**

- A shared address slot containing a base address.
- The memory readable through the base addresses obtained from that slot.
- The behavior of F-a817d8d58c10 if the initial base-address check observes zero.

**Ordinary output:** A Boolean value:

- **True:** At least one of the three observed words contains bit 2 or bit 3.
- **False:** Neither selected bit occurs in any of the three observed words.

The evidence establishes a Boolean result, not its physical calling-convention representation. Register placement, Boolean storage size, and any ABI-specific extension of the result are unspecified.

**Word layout:** Each sampled value is a 32-bit unsigned word under the supplied static type model. Each occupies four bytes. The starting addresses are separated by 32 bytes; this is not a scan of three adjacent words.

Only those three four-byte regions are explicitly read as data. The function does not explicitly examine the intervening bytes. The record type, meaning of the remaining bytes, and total allocation size are unknown.

**Address widths:** The supplied material does not establish an independent pointer-width or address-space contract. The arithmetic offsets are small and exact: 0, 32, and 64. Validity of the resulting addresses remains a caller or environment precondition.

**Ordinary-path preconditions:**

- The shared address slot must be readable on every access.
- Each base address observed during sampling must support a readable 32-bit access at that iteration’s offset.
- Required alignment and memory-access rules must be satisfied by the execution environment.
- If the intended interpretation is “three fields of one object,” the base address and relevant object lifetime must remain suitable throughout the operation.

An initial nonzero base address alone does not establish these conditions.

## Behavior

### Initial check and exceptional branch

The first explicit shared-state read obtains the base-address value for comparison with zero.

If that value is zero, the function invokes F-a817d8d58c10 with one argument whose value is zero. No sampled data word is read before this call.

The supplied static annotation identifies the call as non-returning. Under that interpretation, this branch produces no Boolean return from F-d6a62f9f11ee. It must not be described as returning false, skipping the scan, or recovering with an empty result.

The underlying reason for non-return is unknown. Process termination, exception propagation, control transfer, or another mechanism cannot be selected from the available evidence.

### Sampling sequence

When the initial check observes a nonzero value, the function performs exactly three data-word reads, in increasing offset order:

1. It obtains the current base address from the shared slot and reads the 32-bit unsigned word at offset 0.
2. It obtains the current base address again and reads the 32-bit unsigned word at offset 32.
3. It obtains the current base address again and reads the 32-bit unsigned word at offset 64.

Thus, the supplied representation contains four explicit reads of the shared address slot on the ordinary path: one for the initial check and one for each data-word address.

The checked base address is not retained as a stable snapshot for the entire scan. If the slot changes, the three words can come from different base addresses. The function contains no explicit synchronization, snapshot validation, or retry.

The sampled words are incorporated into an unsigned 32-bit accumulator initially equal to zero. Combination is by bitwise inclusive OR. It is not addition, exclusive OR, intersection, or a count of matching bits.

All three reads occur before the final predicate is returned. Finding a selected bit in the first or second word does not cause an early successful return. Later accesses therefore still matter even when an earlier word already determines that the mathematical predicate will be true.

### Result calculation

After the third word is incorporated, the function tests the combined value against 0x0000000C.

Bits outside that mask do not affect the returned Boolean. For example, bit 0, bit 4, and bit 31 are irrelevant to the result even though their containing words are read and combined.

There is no requirement that the same selected bit appear in multiple words. There is also no requirement that both selected bits appear together in any word.

### Arithmetic boundaries

The loop performs three iterations. Its offset calculation produces only 0, 32, and 64, so the shown signed integer arithmetic does not approach overflow.

The OR operation preserves a 32-bit unsigned bit pattern and has no arithmetic carry or signed-overflow condition.

Address addition is a separate concern. A nonzero base can still be invalid, too near an address-space boundary, insufficiently aligned, or associated with too little readable storage. The function supplies no explicit protection against those cases.

## Side effects

On the ordinary path, the supplied body contains no explicit writes to shared state or to the sampled storage. Its visible operations are reads, local computation, and returning a Boolean.

This establishes only the absence of explicit memory writes in the shown body. It does not establish that every possible backing memory region is free of read-triggered effects. Memory-mapped device behavior and concurrent observers are outside the supplied evidence.

On the zero-base path, F-a817d8d58c10 may have effects not described here. The caller supplies no explicit restoration or cleanup around that call. No claim is made that the dependency preserves globals, leaves allocations intact, releases resources, or preserves the shared address slot.

No ownership transfer is explicit on the ordinary path. The function neither allocates nor explicitly releases the sampled storage. It does not return a pointer or retain a sampled address in an explicitly written shared location.

Invalid reads have no explicit local error conversion. The function does not return a separate error code, substitute zero for an unreadable word, or skip an inaccessible entry.

## Dependencies

### F-a817d8d58c10

**Invocation condition:** The initial read of the shared base-address slot yields zero.

**Argument:** One integer value equal to zero.

**Return expectation:** The supplied static annotation marks the call as non-returning.

**Established ordering:** The call follows the initial zero check and precedes any sampled data-word read.

**Unknown contract:** Argument width, termination mechanism, global-state changes, ownership effects, cleanup behavior, and diagnostics are not established.

If this dependency unexpectedly returns, the shown control structure has no explicit guard preventing continuation into the sampling loop. The safety and supported behavior of that situation are not established. In particular, it is not justified to assume either that the base remains zero or that the dependency repairs it.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Except where noted, assume a stable nonzero base and valid readable words at all three offsets.

| Words at offsets 0, 32, and 64 | Expected result | Reason |
|---|---|---|
| 0x00000000, 0x00000000, 0x00000000 | False | Neither selected bit is present. |
| 0x00000004, 0x00000000, 0x00000000 | True | Bit 2 in the first word is sufficient. |
| 0x00000000, 0x00000008, 0x00000000 | True | Bit 3 in the second word is sufficient. |
| 0x00000000, 0x00000000, 0x0000000C | True | Both selected bits in the third word satisfy the predicate. |
| 0x00000004, 0x00000008, 0x00000000 | True | Selected bits may occur in different words. |
| 0x80000001, 0x00000010, 0x00000002 | False | All observed set bits lie outside the mask. |

Additional boundary expectations:

- **Initial zero base:** F-a817d8d58c10 is called with zero before any sampled word is read. Under the annotated non-returning contract, no Boolean result follows.
- **Early match with invalid later storage:** A matching first word does not bypass the second or third read. Successful return is not guaranteed when a later access is invalid.
- **Changing shared base:** If the initial check sees a nonzero address, but subsequent slot reads produce different addresses, sampling uses each address observed for its respective offset. A single-object snapshot is not guaranteed.
- **Later zero base:** A zero observed during sampling is not subjected to the initial zero-check branch again. The function proceeds toward the corresponding word access without an explicit null-address check.

## Uncertainties

**documented_only — Type recovery and ABI:** The 32-bit word description follows the supplied static type model. No independent machine instructions, ABI documentation, or execution observations were provided.

**documented_only — Memory representation:** Endianness, alignment rules, pointer width, mapped address ranges, and the enclosing object layout are not established. Numeric word examples assume the listed values are those obtained by the word reads.

**documented_only — Concurrency:** Repeated slot reads are explicit in the supplied representation. Atomicity, volatility, cross-thread visibility, and whether the underlying language permits concurrent mutation are unknown.

**documented_only — Dependency behavior:** The non-returning annotation is static evidence, not a runtime observation. Effects of F-a817d8d58c10 remain unspecified beyond its invocation condition and argument.

**Inferred intent:** The 32-byte stride suggests three entries with a relevant word at the same relative position. No record schema or semantic meaning for the selected bits is established.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established static evidence includes the initial zero check, conditional dependency call, three ordered 32-bit word reads at 32-byte spacing, repeated base-address reads, inclusive-OR aggregation, final mask test, and absence of explicit shared-memory writes in the shown body.

No implementation access, runtime tests, fault experiments, or dependency inspection have been performed. The acceptance examples express conditional consequences of the static evidence, not observed execution results.
