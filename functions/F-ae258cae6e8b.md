<!-- Trusted provenance: F-ae258cae6e8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae258cae6e8b

## Purpose

F-ae258cae6e8b conditionally processes an object containing an embedded linked-list entry. An enabled object undergoes two preliminary dependency calls and receives a global snapshot. If its membership-like field is then zero, the function performs additional preparation, optionally computes a key and inserts the embedded entry into a global list, and performs finalization followed by a conditional indirect call.

Deferred scheduling or timeout registration is inferred intent. The evidence is a duration-like field, multiplication by 1,000, a value-producing dependency, and comparison-controlled insertion. The source does not establish time units, chronological ordering, synchronization, or the purpose of the final indirect call.

The established contract is the access and call sequence below. No explicit return value is supplied.

## Interface

### Arguments and result

| Position | Visible representation | Use |
|---|---|---|
| First | 32-bit object address | Base for object accesses; passed to F-a7af65ac69dc. |
| Second | 32-bit opaque value | No explicit use in the supplied body. |
| Third | 32-bit opaque value | Passed as the third argument of every F-0d70d6f9010c call. |
| Fourth | 32-bit opaque value | Passed as the fourth argument of every F-0d70d6f9010c call. |

Each comparison call also receives a fifth 32-bit machine value whose origin is unresolved. It is not established as zero, an explicit argument, or a valid pointer.

The visible representation uses 32-bit addresses, links, and integer words. Whether the argument list completely describes the underlying calling convention remains unknown.

There is no explicit success or failure result. The first argument must designate storage valid for every access reached on the selected path. The body performs no object-address validation.

### Object and entry layout

Offsets are byte offsets from the object address.

| Offset | Access width | Established role |
|---|---:|---|
| `0x518` | 8 bits | Enable gate; zero returns immediately. |
| `0x858` | 32 bits | Embedded entry’s next link. The embedded entry starts here. |
| `0x85C` | 32 bits | Signed duration-like input to key calculation. |
| `0x860` | 32 bits | Computed key; its address is a comparison operand. |
| `0x870` | 32 bits | Copy of a configuration-like global value. |
| `0x874` | 32 bits | Membership-like field; zero permits processing, and insertion writes exactly one. |
| `0x878` | 32 bits | Copy of a snapshot global value. |

Other entries encountered during traversal have a 32-bit next link at entry offset `0x00`. The first comparison argument is the address at entry offset `0x08`. That is an address calculation, not an explicit read of the comparison payload by this function. The comparator’s payload access width is not established here.

For this object’s embedded entry, entry offset `0x08` corresponds to object offset `0x860`.

### Global roles and pointer levels

The body references global locations serving these roles:

- **Snapshot global:** supplies the value written to object offset `0x878`.
- **Resource-address global:** supplies the argument to F-338d16f9ce72. A later, fresh read supplies an address through which a status-object pointer is obtained.
- **List-slot-address global:** contains the address of the writable slot holding the list head.
- **Configuration global:** supplies the value written to object offset `0x870`.
- **Finalization-argument global:** supplies the argument to F-5bcae2eff9a7.
- **Indirect-target global:** supplies the final indirect call target.

The list-slot-address global and the head slot it designates are separate access levels: reading the former captures an address; dereferencing that captured address reads the current head entry.

Distinct source references do not establish that their reachable storage is disjoint from object fields, list entries, or other storage. No general non-aliasing guarantee is available.

## Behavior

### Enable gate

The first explicit access reads the byte at object offset `0x518`.

If it is zero, the function returns immediately. This path contains no dependency calls, global accesses, or other explicit object accesses.

Any nonzero byte enters the enabled path. The enable byte is not explicitly read again.

### Enabled-path preparation

The enabled path proceeds in this order:

1. Call F-a7af65ac69dc with the object address.
2. Form the embedded-entry address at object offset `0x858`.
3. Call F-936a8728139d with no explicit arguments.
4. Read the snapshot global and write its 32-bit value to object offset `0x878`.
5. Read the 32-bit membership field at object offset `0x874`.

The membership decision therefore observes the value present after both dependencies and the snapshot write. Neither dependency is established to preserve the object, global state, or storage validity.

If membership is nonzero, the function returns. The snapshot write has already occurred, but no subsequent setup, entry preparation, insertion, finalization, or indirect call is explicit on this branch.

### Zero-membership preparation

If membership is zero:

1. Read the resource-address global and pass its value to F-338d16f9ce72.
2. Call F-add44d4e9aaa with the embedded-entry address.
3. Read the 32-bit signed duration-like field at object offset `0x85C`.

The duration decision uses the value after F-add44d4e9aaa returns. A value observed before that call is insufficient to predict the branch without an additional preservation guarantee.

The membership field is not explicitly checked again. Changes to it during subsequent dependency calls do not introduce another membership gate.

A zero duration skips key calculation and list insertion, but still reaches finalization and the possible indirect call.

### Nonzero-duration preparation and global-read order

For a nonzero duration, the observable sequence is:

1. Call F-unresolved-01, the unidentified value-producing dependency, with no explicit arguments.
2. After that call returns, read the list-slot-address global and retain the address it supplies.
3. Calculate the retained duration multiplied by 1,000, plus the dependency’s 32-bit result, and write the resulting 32-bit key to object offset `0x860`.
4. Read the configuration global and write its 32-bit value to object offset `0x870`.
5. Dereference the previously retained head-slot address to obtain the current head entry.

The head-slot **address capture precedes both object writes**. The **head-entry read follows both writes**. These are distinct accesses and must not be combined into a single later “read the head” operation.

This ordering remains significant without non-aliasing guarantees. An intervening write could affect the global holding the slot address, the storage at the retained slot address, or the configuration source. A change to the list-slot-address global after capture does not itself select a different slot for this traversal. Conversely, the eventual head-entry read observes the retained slot’s contents after the intervening writes.

The duration used in the calculation is retained from the read before F-unresolved-01. That dependency may modify the duration field, but the body does not reread it for this calculation.

The visible arithmetic operands and stored result are 32-bit integers. Negative nonzero durations take this path. No rejection, saturation, overflow check, wider stored result, or carry field is shown. Behavior when the calculation exceeds the signed 32-bit range is not established; modular wraparound must not be asserted from this representation alone.

The multiplier is exactly 1,000. Its interpretation as a unit conversion is inferred, not established.

### Traversal

Traversal starts with the head obtained from the retained slot and no predecessor.

For each non-null current entry, F-0d70d6f9010c receives:

1. The address eight bytes into the current entry.
2. The address of object offset `0x860`.
3. The third explicit argument.
4. The fourth explicit argument.
5. The unresolved fifth machine value.

Only zero versus nonzero in the result affects control flow:

- **Zero:** read the current entry’s next link after the comparator returns, remember the current entry as predecessor, and continue at the read link.
- **Nonzero:** stop at the current entry without an explicit next-link read for advancement.

Traversal also stops when the current entry is null. No comparator call is made for null.

The next-link read occurs after the comparison, so comparator-induced link changes can affect traversal. The comparator also receives an address to the new object’s key rather than a captured key value; repeated calls are not guaranteed to observe an unchanged key.

No sortedness rule, numeric comparison direction, or equality placement can be established without the comparator’s contract. The visible placement rule is: insert before the first entry producing a nonzero result, or after all traversed entries if no such result occurs.

### Insertion and write order

At the stopping point, the function performs these writes in order:

1. With no predecessor, write the embedded-entry address to the retained head slot. Otherwise, write that address to the remembered predecessor’s next-link field.
2. Write exactly `1` to object offset `0x874`.
3. Write the stopping current-entry pointer to object offset `0x858`.

A null stopping pointer produces a null next link.

The incoming link is published before membership is set and before the new entry’s next link is written. No atomic publication, memory barrier, or exclusion guarantee is established.

The function retains the head-slot address captured before the key and configuration writes; it does not explicitly reload that address during insertion. Comparator-induced changes to the list-slot-address global therefore do not redirect a head insertion to a newly designated slot.

### Finalization and conditional indirect call

Both zero-duration processing and completed insertion continue as follows:

1. Read the finalization-argument global and call F-5bcae2eff9a7 with that value.
2. Read the resource-address global again.
3. Dereference the newly read address to obtain a status-object pointer.
4. If that pointer is non-null, read its first 32-bit word.
5. If both the status-object pointer and its first word are nonzero, invoke F-unresolved-02 through the indirect-target global, with no explicit arguments.
6. Return.

The resource address used here is a fresh read after F-5bcae2eff9a7, not a reuse of the earlier argument to F-338d16f9ce72.

The status-object null check protects the first-word read only. It does not protect the preceding dereference through the resource address. That address must already designate readable pointer storage.

No explicit null or validity check protects the indirect call target once the status gate succeeds.

## Side effects

The body explicitly writes:

- Object offset `0x878` on every enabled path reaching snapshot assignment.
- Object offsets `0x860` and `0x870` on the nonzero-duration path.
- Object offsets `0x874` and `0x858` during insertion.
- Either the retained head slot or a predecessor’s next link during insertion.

These are access locations, not guarantees about disjoint storage or final values under aliasing.

Dependencies may introduce additional writes, ownership changes, or other effects. Their placement does not establish locking, unlocking, allocation, release, or preservation of globals. No direct allocation or release appears in this body.

The inserted entry resides inside the supplied object. The object and all accessed linked storage must remain valid throughout the applicable accesses. Continued list use also requires a sufficient lifetime, but ownership transfer, removal, and release responsibilities are undocumented.

There is no explicit error result, failure branch for setup or finalization, exception handler, or rollback. If a dependency does not return normally, subsequent behavior is not established.

No cycle or duplicate-entry detection is visible. Zero membership does not prove absence from the list. Cycles, stale pointers, duplicate insertion, or inconsistent membership can cause nontermination, invalid accesses, or structural corruption; no recovery is specified.

## Dependencies

F-unresolved-01 and F-unresolved-02 are document-local opaque identifiers for unresolved call sites, not resolved identities supplied by the dependency map.

| Opaque function ID | Contract established at this call site |
|---|---|
| F-a7af65ac69dc | Called with the object address on the enabled path. Result unused. |
| F-936a8728139d | Called with no explicit arguments before the snapshot read. Result unused. |
| F-338d16f9ce72 | Called after zero membership, using a read of the resource-address global. Result unused. |
| F-add44d4e9aaa | Called with the embedded-entry address before duration is read. Result unused. |
| F-unresolved-01 | Called only for nonzero duration; supplies the 32-bit arithmetic input. |
| F-0d70d6f9010c | Called for visited non-null entries; zero advances, nonzero stops. |
| F-5bcae2eff9a7 | Called after zero-duration processing or insertion; precedes fresh status accesses. Result unused. |
| F-unresolved-02 | Indirectly invoked only when the status pointer and its first word are nonzero. Result unused. |

The supplied body does not establish these dependencies’ implementations, memory ownership rules, preservation guarantees, or full calling conventions.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, they assume valid, suitably non-overlapping storage, normally returning dependencies, and no dependency mutations.

### Disabled object

With a zero enable byte, the function returns without dependency calls or explicit writes.

### Enabled, already marked present

With a nonzero enable byte and nonzero membership at the actual membership read:

- F-a7af65ac69dc precedes F-936a8728139d.
- The snapshot is copied to offset `0x878`.
- No setup, insertion, finalization, or indirect call follows.

### Zero duration after preparation

With zero membership and a duration that is zero after F-add44d4e9aaa:

- F-338d16f9ce72 precedes F-add44d4e9aaa.
- F-unresolved-01 and the comparator are not called.
- The list-slot-address global is not explicitly read.
- No explicit key, configuration, membership, or next-link write occurs.
- Finalization and the status gate still occur.

### Retained head-slot address

Suppose the list-slot-address global supplies address A at capture.

The function retains A before writing offsets `0x860` and `0x870`, then reads the head through A after those writes. If an aliasing write changes the address global to B, that change alone does not redirect the later head read to B. If an intervening write instead changes the contents of A, the later read observes those contents.

This example asserts access order, not that any particular alias configuration is otherwise valid.

### Empty list

With nonzero duration and a null head when the retained slot is dereferenced:

- No comparator call occurs.
- The retained head slot receives the embedded-entry address.
- Membership receives `1`.
- The new next link receives null.

Those three insertion writes occur in that order.

### Stop at the head

With a non-null head and a nonzero first comparison result, the function inserts before that entry. It does not explicitly read that entry’s next link for traversal.

### Advance once, then stop

If the first comparison returns zero and the second returns nonzero:

- The first entry’s next link is read after its comparison.
- That entry becomes the predecessor.
- Its next link receives the new entry address.
- Membership receives `1`.
- The new entry’s next link receives the second entry’s address.

### Representable arithmetic

A retained duration of `2` and a value-producing result of `5,000` produce a stored key of `7,000`.

A retained duration of `-1` and a result of `5,000` produce `4,000`. Negative duration alone does not suppress insertion.

Neither example establishes out-of-range arithmetic behavior.

### Final indirect-call gate

After F-5bcae2eff9a7:

- A null status-object pointer suppresses F-unresolved-02.
- A non-null pointer with a zero first word suppresses it.
- A non-null pointer with a nonzero first word causes the indirect invocation, provided the target is callable.

## Uncertainties

The following remain **documented_only**:

- Duration and key meanings and units.
- Comparator ordering, tie policy, payload layout, and handling of out-of-range keys.
- Origin and validity requirements of the fifth comparator argument.
- Resolved identities and broader contracts of F-unresolved-01 and F-unresolved-02.
- Arithmetic behavior outside signed 32-bit range.
- Actual alias relationships among globals, object fields, head-slot storage, and entries.
- Synchronization, reentrancy, and dependency-induced mutations.
- Allocation, release, ownership transfer, removal, and lifetime rules.
- Complete ABI and calling-convention details.
- Validation requirements beyond the accesses visible in this body.

The scheduling interpretation is inferred intent and creates no additional acceptance requirement.

## Evidence

The supplied static body establishes field offsets, explicit access widths, branch conditions, dependency-call order, traversal continuation rules, insertion write order, and the final indirect-call gate.

In particular, it establishes a read of the list-slot-address global after the value-producing call and before the key write; the configuration read and write follow that key write; only then is the retained slot dereferenced for the head entry.

The dependency map supplies six resolved dependency identities used above. It supplies no implementations or preservation guarantees for those dependencies. The two unresolved call sites retain document-local identifiers.

No runtime tests have been performed. The acceptance examples are conditional static consequences, not observed execution results.
