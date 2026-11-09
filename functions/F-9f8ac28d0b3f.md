<!-- Trusted provenance: F-9f8ac28d0b3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9f8ac28d0b3f

## Purpose

F-9f8ac28d0b3f obtains an entry associated with a positive, 31-bit identifier. It either processes an entry returned by lookup or obtains entry storage, initializes selected fields, and inserts an entry into context-associated bucket storage. Successful completion publishes an entry address through a caller-provided output slot and returns zero.

The apparent intent is cached-entry acquisition with reference-like bookkeeping. “Cache,” “reference,” and “initialization” describe inferred roles, not established ownership or subsystem contracts.

The function is not visibly transactional. New-entry bookkeeping occurs before several possible failures, and no local rollback restores all earlier writes.

## Interface

### Arguments and result

| Input | Representation | Established role |
|---|---|---|
| Context | 32-bit address | Supplies configuration, status, counters, and bucket storage. |
| Identifier | Unsigned 32-bit integer | Lookup key and value initially written into a newly obtained entry. |
| Output location | Address of a writable four-byte slot | Receives zero after validation and an entry address on successful completion. |
| Mode | 32-bit integer | Zero and nonzero select behavior; no narrower range is enforced. |

The result is a 32-bit status:

- **0:** successful completion.
- **11:** identifier rejected during initial validation.
- **13:** alternative initialization rejected because the identifier exceeds a context limit.
- **Other nonzero values:** propagated dependency results or a context status value.

A result of **522**, hexadecimal **0x20A**, from F-b5b842adfa3c is accepted rather than propagated.

### Directly accessed layout

Offsets are in bytes. Addresses and pointer-bearing fields have a four-byte representation.

| Object | Offset | Width | Use |
|---|---:|---:|---|
| Context | 0x13 | 8 bits | Mode-related initialization flag. |
| Context | 0x14 | 8 bits | First-path eligibility flag. |
| Context | 0x20 | 32 bits | Status read after F-f3192f39e4f6. |
| Context | 0x3C | 32 bits | Signed value tested against zero. |
| Context | 0x40 | 32 bits | Argument to validation and alternative initialization dependencies. |
| Context | 0x48 | 32 bits | Counter incremented during new-entry setup. |
| Context | 0x50 | 32 bits | Unsigned alternative-path identifier limit. |
| Context | 0x54 | 32 bits | Argument to the byte-producing dependency. |
| Context | 0xCC | 32 bits | Value from which the bucket mask is formed. |
| Context | 0xD0 | 32-bit address | Bucket-array base, read separately for the old-head load and new-head store. |
| Entry | 0x04 | 32 bits | Identifier; cleared on one fatal path. |
| Entry | 0x08 | 32-bit address | Next-entry link. |
| Entry | 0x0C | 32-bit address | Back-link written in a linked entry. |
| Entry | 0x1C | 8 bits | Dependency-produced byte. |
| Entry | 0x1E | 8 bits | Explicitly cleared during new-entry setup. |
| Entry | 0x20 | 8 bits | Initialization-related flag. |
| Entry | 0x22 | 16 bits | Reference-like state. |
| Entry | 0x34 | 32 bits | Argument to alternative initialization. |
| Entry | 0x38 | Interior address | Passed to an optional dependency. |

An internal four-byte slot holds the lookup result and later supplies entry addresses. Its address is passed to F-bff1472e66f7 on a lookup miss. **Its incoming value at that call is zero**, not unspecified.

### Preconditions and aliasing

No local checks establish pointer validity, object extent, alignment, bucket capacity, or dependency-specific preconditions. Every reached access requires suitable memory.

Nonoverlapping output, context, entry, and bucket storage is a useful operating assumption, not an enforced condition. Aliasing can change later reads through earlier writes. In particular, entry offset 0x08 can overlap context offset 0xD0, changing the bucket base between its two reads.

Once the internal slot’s address escapes, its preservation across later calls is not established. Descriptions of a single stable “entry” require that dependencies leave that slot unchanged. Otherwise, subsequent accesses and final publication use the slot’s value at the corresponding point.

## Behavior

### 1. Identifier validation

An identifier of zero or greater than 0x7FFFFFFF causes immediate return of 11. These cases perform no dependency call, do not read context offset 0x40, and do not locally write the output slot.

For an identifier in the accepted numerical range, the function reads context offset 0x40 and calls F-870c4fcf7295 with:

1. Constant 0x40000000.
2. The four-byte value just read.

The dependency result is increased by one using unsigned 32-bit arithmetic. If the result equals the identifier, the function returns 11 without locally writing the output slot. This exclusion uses modulo-2³² addition.

After these checks succeed, the function writes zero to the output slot. Dependency effects during validation are unknown; “unchanged output” means no direct write by this function, not guaranteed preservation against dependency side effects.

### 2. Preparation and lookup

F-d113371226a7 receives the context. A nonzero result is returned immediately.

On zero, F-c1f06f89d6d7 receives the context and identifier. Its result is stored in the internal entry-address slot:

- Zero selects acquisition of a new entry.
- Nonzero selects existing-entry processing.

No local comparison verifies that a returned entry contains the requested identifier.

### 3. Acquisition and slot exposure

On a lookup miss, F-bff1472e66f7 receives the context and the address of the internal slot. The slot still contains the **zero lookup result** when the call begins; the dependency may observe that value as well as replace it.

A nonzero dependency result is returned immediately, regardless of what it wrote into the slot.

A zero result is accepted without checking the slot for a nonzero or valid address. If the dependency returns zero without changing the slot, the next visible entry access uses a zero entry base: the identifier write targets offset 0x04 from that base. No local error status substitutes for this invalid-pointer scenario.

Passing the slot’s address makes it available to the dependency. Whether that address is retained or used by subsequent dependencies is unknown. The function does not visibly protect the slot from later modification. Consequently, acquisition success alone does not establish that later entry accesses use the originally supplied address.

### 4. Initial new-entry writes

After successful acquisition, the following actions occur in order:

1. Write the identifier at entry offset 0x04.
2. Read context offset 0x54 and call F-811a25bdbe21 with that value and the identifier.
3. Store the returned byte at entry offset 0x1C.
4. Store zero at entry offset 0x1E.
5. Call F-d0d75bd453b2; explicit arguments are not established.
6. Store the 16-bit value one at entry offset 0x22.
7. Read the context counter at 0x48, increment it, and store the result.
8. Read context offset 0x3C as signed. If positive, call U-01 with the entry’s interior address at offset 0x38.
9. Call F-f3192f39e4f6 with the context.
10. Read the 32-bit context status at 0x20.

Dependency calls can separate the selection of one entry address from later entry accesses. For example, the destination of the byte store after F-811a25bdbe21 is not guaranteed to retain the address used for the earlier identifier write.

If the context status is nonzero, F-44d249c54e70 is called with the then-current entry address, and the previously read status is returned. There is no local reversal of the counter increment or prior entry writes. The cleanup-shaped dependency’s effects remain unknown.

The counter increment is a 32-bit operation with no explicit overflow check. Its original-language signed-overflow semantics are not established.

### 5. Initialization-path selection

If context status is zero, the result from F-f3192f39e4f6 controls eligibility for the first initialization path.

The comparison interprets both that result and the identifier as signed 32-bit values. Although accepted identifiers are positive, a dependency result with its high bit set compares as negative.

The first path is selected exactly when:

- The signed identifier is less than or equal to the signed dependency result.
- Context byte 0x14 is zero.
- Mode is zero, or context byte 0x13 is nonzero.

The 0x14 byte is read only when the signed comparison succeeds. The mode-related condition is considered only after that eligibility condition succeeds. Within the mode-related condition, mode zero avoids the 0x13 read.

#### First path

F-b5b842adfa3c receives the context, current entry address, and identifier.

A result of zero or 522 is accepted. For either accepted result, entry byte 0x20 is written as zero, followed by bucket insertion.

Any other nonzero result causes these actions:

1. Clear the identifier field at entry offset 0x04.
2. Call F-44d249c54e70.
3. Return the initialization status.

The supplied representation does not establish an explicit argument for this particular F-44d249c54e70 invocation. No bucket insertion is performed locally on this failure path.

#### Alternative path

The function reads context offset 0x50 and compares it with the identifier as unsigned 32-bit values.

If the limit is smaller than the identifier, F-44d249c54e70 receives the current entry address, and the function returns 13. This branch does not explicitly clear the identifier first.

Otherwise, U-02 receives the four-byte values from entry offset 0x34 and context offset 0x40. Their relative read order is not established.

After U-02 returns, entry byte 0x20 is set to:

- Zero when mode is zero.
- Zero when mode is nonzero and the then-read context byte 0x13 is nonzero.
- One otherwise.

The mode-nonzero flag read occurs after U-02; it is not a preserved value from earlier path selection. The destination entry address likewise depends on the internal slot’s value after that call.

### 6. Bucket insertion

The bucket index is the identifier bitwise-ANDed with the 32-bit result of subtracting one from context offset 0xCC. Subtraction is unsigned and wraps modulo 2³². A zero field produces mask 0xFFFFFFFF.

This is masking, not general remainder. Bucket displacement uses a four-byte stride in the supplied 32-bit address model. No local bounds or address-overflow check proves that the resulting access lies within bucket storage.

The observable insertion order is:

1. Read context offset 0xD0 to obtain a bucket base.
2. Read the old head at that base plus the indexed displacement.
3. Store the old head at entry offset 0x08.
4. **Read context offset 0xD0 again** to obtain the destination base.
5. Store the current entry address at that second base plus the same indexed displacement.
6. Read entry offset 0x08 and test it against zero.
7. If nonzero, read entry offset 0x08 again and write the current entry address at offset 0x0C of the entry designated by that read.

The second base read is not equivalent to reusing the first selected bucket address. If the step-3 store overlaps context offset 0xD0, it can redirect the step-5 store.

Likewise, step 5 can affect the later link test if its destination overlaps entry storage. Without suitable nonaliasing, the final back-link target need not be the originally loaded head.

The function does not explicitly initialize the new entry’s own offset 0x0C.

### 7. Existing-entry processing

For a nonzero lookup result:

- Mode zero causes a call to F-4c6b253a4fd3. Its explicit arguments are not established. A nonzero result is returned before local reference-like processing.
- Nonzero mode skips that call.

The function then examines the 16-bit entry field at 0x22:

- If zero, it calls F-7d670f6f009a. Explicit arguments are not established, and no local increment follows.
- If nonzero, it reads the field for increment and stores the incremented value in 16 bits.

The stored increment is width-limited: the represented value 0xFFFF becomes zero. There is no local saturation or overflow rejection. F-7d670f6f009a is selected by the pre-increment zero test, not by an increment producing zero.

This path performs no local new-entry initialization or bucket insertion.

### 8. Successful completion

After either successful branch, the function writes the current internal-slot value to the output location and returns zero.

Final publication follows local entry processing. It does not establish synchronization, atomic publication, a nonzero output guarantee independent of dependencies, or an ownership transfer.

## Side effects

Direct effects include output clearing and publication, selected entry-field initialization, context-counter increment, bucket-head and back-link writes, existing-entry state increment, and identifier clearing on one failure path.

Failure after initial validation does not imply restoration of earlier state. The output has been locally cleared before preparation, but dependencies or aliases can subsequently change it; it is not unconditionally guaranteed to remain zero on failure.

No allocation, release, ownership transfer, locking, or preservation of global state is established for dependencies. Cleanup-shaped calls do not prove deallocation. A successful address may represent owned, borrowed, or reference-held storage; the applicable lifetime contract is unavailable.

The escaped internal-slot address adds a separate lifetime uncertainty. The function exposes that address during its activation, but no supplied contract authorizes its later retention or use after return.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-870c4fcf7295 | Produces the validation-exclusion value. |
| F-d113371226a7 | Context preparation; nonzero status propagates. |
| F-c1f06f89d6d7 | Lookup; result is stored in the internal slot. |
| F-bff1472e66f7 | Receives context and the address of the zero-valued internal slot; may supply an entry address. |
| F-811a25bdbe21 | Produces the byte stored at entry offset 0x1C. |
| F-d0d75bd453b2 | Called during new-entry setup; arguments and effects unresolved. |
| F-f3192f39e4f6 | Produces a signed-comparison operand; context status is read afterward. |
| F-b5b842adfa3c | First initialization path; statuses zero and 522 accepted. |
| F-44d249c54e70 | Called on three new-entry failures; explicit arguments differ in what is established. |
| F-4c6b253a4fd3 | Existing-entry, zero-mode processing; nonzero status propagates. |
| F-7d670f6f009a | Existing-entry processing for zero 16-bit state. |
| U-01 | Optional call with entry interior address at 0x38. |
| U-02 | Alternative initialization using entry 0x34 and context 0x40 values. |

U-01 and U-02 are document-local opaque dependency IDs. Their external identities and contracts are unavailable.

## Acceptance examples

These are conditional static consequences, not executed tests. Except where varied explicitly, assume valid, nonoverlapping storage; returning dependencies; no relevant dependency-induced mutations; and preservation of the internal slot after acquisition.

1. **Immediate rejection:** Identifier zero or 0x80000000 returns 11 without dependency calls, reading context 0x40, or changing an output sentinel.

2. **Excluded identifier:** Identifier 7 with F-870c4fcf7295 returning 6 returns 11 without locally clearing output.

3. **Preparation failure:** A valid, nonexcluded identifier followed by preparation status 37 causes output clearing and return of 37, without lookup.

4. **Acquisition input:** A zero lookup result causes F-bff1472e66f7 to receive a slot containing zero. Returning nonzero propagates that status. Returning zero without updating the slot leads to an identifier write using zero as the entry base, not a locally detected acquisition error.

5. **Escaped-slot mutation:** If acquisition supplies address A but F-811a25bdbe21 subsequently changes the escaped slot to valid address B, the identifier was written at A plus 0x04, while the returned byte is stored at B plus 0x1C. A single stable entry cannot be assumed.

6. **Existing-entry increment:** Nonzero mode and existing state 4 skip F-4c6b253a4fd3, store 5, publish the entry, and return zero.

7. **Existing-entry boundary:** State 0xFFFF becomes zero without invoking F-7d670f6f009a merely because of that result. Initially zero state invokes that dependency instead of locally storing one.

8. **Accepted initialization status:** First-path status 522 clears entry byte 0x20, proceeds through insertion, and ultimately returns zero.

9. **Fatal initialization status:** First-path status 521 clears entry identifier 0x04, invokes F-44d249c54e70, and returns 521 without local insertion.

10. **Alternative-path limit failure:** Identifier 10 with alternative-path limit 9 returns 13 after the cleanup-shaped call. The earlier counter increment is not locally undone.

11. **Signed boundary:** A comparison result of 0xFFFFFFFF with zero context status forces a positive accepted identifier onto the alternative path without reading flag 0x14 for eligibility.

12. **Stable occupied bucket:** With nonoverlapping entry and bucket storage and an unchanged context bucket base, an old head is copied into new-entry 0x08, the new entry becomes head, and the old head’s 0x0C receives the new address.

13. **Aliased bucket base:** If entry 0x08 overlaps context 0xD0, storing the old head replaces the base field. The bucket-head store uses that replacement base on its second read. Validity of the resulting destination is an additional precondition.

## Uncertainties

The following remain **documented_only**:

- Concrete types, object extents, alignment, and address-validity requirements.
- Dependency allocation, release, ownership, synchronization, and global-state effects.
- Retention and modification of the escaped internal slot.
- Arguments absent from the supplied representation.
- Original-language overflow semantics of the signed context-counter increment.
- Required bucket capacity, power-of-two constraints, and address-overflow policy.
- Initialization of fields not visibly written, including the new entry’s own back-link.
- Semantic meaning of mode, status 522, and reference-like state.
- Concurrency guarantees and dependency effects on output or aliased storage.

## Evidence

Established behavior comes from static inspection of the supplied function representation and dependency mapping: explicit access widths, call arguments where shown, branch conditions, arithmetic, and ordered memory operations.

The zero incoming acquisition slot, its address exposure, and the two distinct bucket-base reads are directly visible. Preservation of that slot across later calls is not established.

Cache acquisition and reference tracking are inferred intent. Dependency ownership and lifetime behavior remain unknown.

No runtime tests, dependency execution, or ownership verification have been performed. The acceptance examples describe conditional consequences of static evidence, not observed execution.
