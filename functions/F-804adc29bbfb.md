<!-- Trusted provenance: F-804adc29bbfb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-804adc29bbfb

## Purpose

F-804adc29bbfb conditionally creates a 16-byte record and inserts it at the head of a chain associated with an object supplied by a dependency.

The record contains one link word followed by three caller-supplied payload words. A failed storage request produces a false result without attempting to obtain the object or update its chain. A successful storage request leads to record initialization and head insertion, provided the subsequent dependency call and memory accesses complete normally.

The boolean result reports whether the storage dependency returned a non-null value. It does not independently establish that the record remains allocated, that the resulting chain is valid, or that another component has assumed ownership.

This contract describes the supplied static evidence. No runtime tests have been performed.

## Interface

### Inputs

The function accepts three ordered, 32-bit values:

| Input | Width | Destination in the new record |
|---|---:|---:|
| First payload | 32 bits | Byte offset 4 |
| Second payload | 32 bits | Byte offset 8 |
| Third payload | 32 bits | Byte offset 12 |

The inputs are opaque words. The visible behavior does not interpret them as numbers, addresses, handles, flags, or ownership-bearing references. It preserves their 32-bit representations in the corresponding fields.

There are no visible restrictions on the payload values. Zero, repeated values, and values with the high bit set follow the same path as other inputs.

### Output

The function returns a boolean:

- **False:** F-d33f6d597e5d returned a null value.
- **True:** F-d33f6d597e5d returned a non-null value, and execution reached the return normally.

There is no visible error-code output or direct return of the new record’s address.

A non-null storage result is not sufficient to guarantee a normal true return if a later dependency does not return, or if a later memory access fails.

### Record layout

The requested extent is exactly 16 bytes. The visible record has four consecutive 32-bit fields:

| Byte offset | Width | Meaning after normal completion |
|---:|---:|---|
| 0 | 32 bits | Previous value of the selected object’s head field |
| 4 | 32 bits | First payload |
| 8 | 32 bits | Second payload |
| 12 | 32 bits | Third payload |

There is no visible padding between these fields and no additional field initialized by this function.

The link and head operations use a 32-bit representation in the supplied evidence. This contract therefore does not describe a native 64-bit-pointer variant. Byte order is not established.

### Preconditions for a normal successful execution

If the storage dependency returns a non-null value, the later operations require:

- The returned address to support the three payload writes and the link-field write.
- That storage to remain usable across the call to F-187ab9ba4eb6.
- F-187ab9ba4eb6 to return an address for which the field at byte offset 56 can be read and written.
- The new record’s address to be representable in the head field under the observed 32-bit representation.

The function does not visibly validate these conditions. Alignment requirements and behavior on invalid addresses depend on the execution environment and are not established here.

## Behavior

### Storage request and failure branch

The first visible operation is a call to F-d33f6d597e5d with the size value 16.

If that call returns null:

- No payload field is written.
- No link field is written.
- F-187ab9ba4eb6 is not called.
- No object head field is read or written by this function.
- The function returns false.

These statements concern this function’s explicit operations. They do not imply that F-d33f6d597e5d itself has no side effects when returning null.

There is no visible retry, alternative storage request, or recovery operation.

### Payload initialization

If the storage result is non-null, the function writes the payload fields in this order:

1. First payload at byte offset 4.
2. Second payload at byte offset 8.
3. Third payload at byte offset 12.

The link field at byte offset 0 has not yet been initialized by this function at that point.

No previous contents of the payload fields are read. The function does not clear the entire allocation before writing these fields.

### Selection of the object

Only after all three payload writes does the function call F-187ab9ba4eb6. No explicit arguments are supplied in the visible call.

The dependency’s result supplies the base address used for the head-field access. The function does not visibly check that result for null or any other invalid value.

The timing of this call matters. The target object is obtained after allocation and payload initialization, not before them. The evidence does not establish that repeated calls select the same object, nor that this dependency merely reads state.

Because the dependency is unknown, it must not be assumed to preserve globals, leave the new allocation untouched, or preserve allocation lifetime. The visible function does not rewrite the payload fields after the dependency returns.

### Head insertion

After F-187ab9ba4eb6 returns normally, the function:

1. Reads the 32-bit head field at byte offset 56 from the returned base address.
2. Writes the value just read into the new record’s field at byte offset 0.
3. Writes the new record’s address into that same object head field.

The previous head is therefore sampled after the object-selection dependency has completed. If that dependency changes the selected object’s head, the subsequent read observes the post-call value, subject to any external interference.

The previous head is copied as an opaque 32-bit value. It is not checked, dereferenced, traversed, or modified by this function. A zero previous head is handled by copying zero into the new record’s link field.

Publication through the object’s head field occurs after the link-field write. Under stable, non-overlapping storage and without intervening interference, normal completion leaves the new record as the head and the former head as its successor.

### Completion

After the insertion operations, the function returns true because its retained storage result is non-null.

There is no separate success test of the object, link value, or final head contents. No read-back confirms that the writes persisted.

### Arithmetic and widths

The storage request uses the fixed value 16. Payloads do not participate in address or size arithmetic.

The visible field offsets are 0, 4, 8, and 12 bytes within the record, and 56 bytes within the object. Each record access has a width of four bytes, so the final payload occupies bytes 12 through 15 of the requested extent.

The object address is used with the fixed offset 56. No bounds or address-overflow check is visible. The evidence does not establish portable behavior if that address calculation is invalid.

There is no signed comparison or arithmetic transformation of a payload. Payload interpretation as signed or unsigned does not affect the visible copying behavior.

## Side effects

### Explicit memory effects

On a normally completing non-null path, the function explicitly performs:

- Three 32-bit payload writes to the new record.
- One 32-bit read of the selected object’s head.
- One 32-bit write to the new record’s link field.
- One head-field write publishing the new record’s address.

The function does not explicitly read payload contents, inspect older records, or update the former head record.

On the null-result path, it performs none of these explicit memory accesses.

### Ownership and lifetime

The non-null path retains the storage result long enough to initialize and publish it. There is no explicit release operation in either branch.

Linking the record into the selected object makes its address available through that object under ordinary storage assumptions. This suggests retention for later use, but it does not establish an ownership-transfer policy.

The following are unknown:

- Whether the storage result denotes fresh, exclusive allocation.
- Which component is responsible for eventual release.
- Whether either dependency retains or releases storage.
- Whether payload words represent resources requiring separate lifetime management.
- Whether the selected object outlives the record.

The absence of a visible release must not be generalized into a claim that the unknown dependencies never release memory.

### Concurrency and partial state

No locking, atomic read-modify-write, or explicit synchronization is visible.

The head read and head write are separate operations. Under concurrent modification, this sequence is not established to preserve every insertion. Individual access atomicity and memory-ordering guarantees are also not established.

If execution is interrupted or fails after a non-null storage result, partial initialization may already have occurred. No rollback is visible. In particular, the payload writes precede object selection, and the link write precedes publication.

## Dependencies

| Opaque function ID | Visible use | Established contribution | Not established |
|---|---|---|---|
| F-d33f6d597e5d | Called once with size 16 | Returns the value tested for null and used as the record address | Allocation mechanism, initialization, alignment, exclusivity, failure side effects, lifetime guarantees |
| F-187ab9ba4eb6 | Called once only after a non-null storage result and all payload writes | Returns the base used for the head field at byte offset 56 | Object identity, validity, global effects, ownership effects, thread behavior, preservation of prior memory |

Neither dependency’s internal behavior is supplied. Their roles above describe how their results are consumed, not complete dependency contracts.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Null storage result

Given that F-d33f6d597e5d returns null for the request of 16 bytes:

- The result is false.
- F-187ab9ba4eb6 is not called.
- This function performs no record writes or object head accesses.

Any side effects of the failed storage request remain outside this example’s guarantees.

### Insertion with an empty head

Given usable, non-overlapping record and object storage, no interfering dependency effects, and an object head initially equal to zero:

- Payloads 17, 34, and 51 are stored at offsets 4, 8, and 12.
- The new link field becomes zero.
- The object head becomes the new record’s address.
- The result is true.

### Insertion with an existing head

Given the same validity and stability conditions, but with an existing nonzero head:

- The existing head value is copied into the new link field.
- The object head is replaced with the new record’s address.
- No access to the record designated by the existing head is required.
- The result is true.

This remains the visible behavior even if the existing head value is not a valid record address, provided the stated accesses themselves remain valid.

### Opaque payload preservation

Given payload bit patterns 0x00000000, 0xFFFFFFFF, and 0x80000000, and no intervening mutation of the record:

- Those exact 32-bit patterns occupy offsets 4, 8, and 12.
- None is rejected, dereferenced, normalized, or used to choose a branch.

### Head changed during object selection

Given that F-187ab9ba4eb6 returns a valid object after changing its head:

- The new link receives the head value present when the subsequent head read occurs.
- The function does not use a head value captured before the dependency call.

### Invalid object result

Given a non-null storage result followed by an unusable object result:

- The three payload writes have already occurred if execution reached the object-selection call normally.
- No explicit validation or false-return branch handles the unusable result.
- A particular fault, termination mode, or recovery behavior is not specified by the evidence.

## Uncertainties

### Established static evidence

The supplied body establishes the fixed allocation request, null-result branch, payload-write order, conditional dependency call, head-field offset, link-before-publication order, and boolean return condition.

### Inferred intent

The layout and update sequence are consistent with prepending a record to a singly linked chain. Calling the storage request an allocation and the selected field a chain head is a structural interpretation supported by their use.

That interpretation does not establish a public container API, a payload schema, or an ownership policy.

### documented_only

The following remain unresolved:

- Exact calling convention, exported type declarations, and machine-level boolean return encoding.
- Byte order and access-alignment rules.
- Dependency side effects and guarantees.
- Whether record and object storage can alias.
- Whether globals or storage lifetime can change during either dependency call.
- Thread safety, reentrancy, and concurrent-reader guarantees.
- Fault behavior for invalid pointers or invalid address arithmetic.
- Eventual record removal and release.
- Semantic meaning of the three payloads.

The ordinary head-insertion postcondition assumes suitable non-overlapping storage. If the head field aliases a record field, the stated write sequence still describes the visible operations, but later writes may overwrite earlier field values.

## Evidence

The evidence consists solely of the supplied static function body and the supplied mapping to opaque function IDs.

It supports the explicit operations and ordering described here. It does not include dependency implementations, runtime traces, allocator documentation, object definitions, concurrency contracts, or lifetime documentation.

No runtime tests have been performed. The acceptance examples specify conditional behavior for review; they are not observations of an executed implementation.
