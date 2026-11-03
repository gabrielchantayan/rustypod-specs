<!-- Trusted provenance: F-eab17af3f0cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eab17af3f0cd

## Purpose

F-eab17af3f0cd conditionally publishes a pointer-sized value into a destination slot and returns the address of that slot.

A zero second argument causes an immediate null return. Otherwise, a dependency establishes the destination location, and metadata preceding a descriptor-associated pointer selects one of two behaviors:

- An exact all-one-bits metadata value selects a call that produces an address, publication of that address adjusted by twelve bytes, and an external operation using a subsequently reread descriptor pointer.
- Any other metadata value selects direct publication of the initially captured descriptor pointer, followed by a call concerning its preceding metadata.

The established behavior is the sequence of reads, writes, calls, and address calculations. Allocation, copying, sharing, and reference management are possible interpretations, not established guarantees.

## Interface

The visible interface has four explicit inputs and an address-valued result.

| Input | Visible representation | Established use |
|---|---|---|
| First argument | Four-byte opaque value | Not referenced by this function body. |
| Second argument | Four-byte integer | Compared with zero; passed as the first input to F-f4fd698ba468 when nonzero. |
| Third argument | Four-byte integer used as an address | Passed as the second input to F-f4fd698ba468; its byte offset +4 supplies a descriptor-pointer field. |
| Fourth argument | Four-byte opaque value | Passed to F-fe78e25e82ff on the sentinel path only. |

F-fe78e25e82ff also receives an unresolved register-derived value as its fifth input. The supplied representation does not establish that value’s origin, initialization, type, or relationship to the four explicit arguments.

### Result

When the second argument is zero, the result is null.

Otherwise, if execution completes normally, the result is the address obtained by adding four bytes to the value returned by F-f4fd698ba468. That address identifies the destination slot; it is not the value published into the slot.

A nonzero second argument does not itself guarantee a valid or non-null result. The function does not validate the dependency result or the adjusted address.

### Widths and layout

The supplied representation establishes four-byte integer values, four-byte pointer-bearing reads and writes, and byte-address adjustments. It does not independently establish the complete execution ABI or the behavior of overflowing address arithmetic.

The relevant descriptor layout is:

| Position | Access | Meaning established by use |
|---|---|---|
| Third-argument address +4 bytes | Four-byte read | Supplies the descriptor-associated pointer. |

Relative to the pointer captured from that field:

| Position | Access | Meaning established by use |
|---|---|---|
| −12 bytes | Four-byte read | Selects the branch by comparison with signed −1, the all-one-bits pattern. |
| −4 bytes | Four-byte read, sentinel path only | Supplies a value repeated in the F-fe78e25e82ff call and later supplied to the external dependency. |
| 0 bytes | Address value | Published directly on the nonsentinel path; identifies the region whose preceding metadata is initially inspected. |

The body does not inspect the four bytes between offsets −8 and −5. It does not establish a complete object-header layout.

The value at offset −4 is treated here as an opaque four-byte, amount-like value. Its signedness, units, and interpretation are unresolved.

### Preconditions for normal completion

The zero-input path does not require the descriptor or destination memory to be addressable.

For a nonzero second argument, normal completion requires the addresses actually accessed to support the relevant reads and writes:

- The descriptor field at the third-argument address plus four must be readable after F-f4fd698ba468 returns.
- The field twelve bytes before the captured descriptor pointer must be readable.
- On the sentinel path, the field four bytes before that pointer must be readable, and the descriptor field must remain readable for its later reread.
- The destination slot must support a four-byte write at publication time.
- Every invoked dependency must have its own requirements satisfied and return normally.

The function supplies no explicit bounds, alignment, capacity, ownership, null-pointer, or failure-result checks.

## Behavior

### Zero-input path

The second argument is tested before any dependency call or descriptor access.

If it is zero, the function returns null immediately. The other explicit arguments do not affect this path’s result. No listed dependency is called, and this body performs no memory read through the descriptor or explicit memory write.

Every nonzero second-argument value proceeds beyond this guard, including values represented as negative integers.

### Establishing the destination and capturing metadata

On the nonzero path, F-f4fd698ba468 receives the second and third explicit arguments, in that order.

The destination-slot address is formed by adding four bytes to its result. The function does not reject a null, invalid, or otherwise failure-shaped result.

Only after this call does the function read the descriptor-pointer field at the third-argument address plus four. Consequently, mutations performed by F-f4fd698ba468 before returning can determine which pointer is captured.

The function reads the four-byte field twelve bytes before that captured pointer. The sentinel path is selected only when this field has the exact all-one-bits pattern, represented as signed −1. Other negative values select the nonsentinel path.

The branch decision uses this captured metadata. It is not revisited after subsequent dependency calls or publication.

### Sentinel path

On the sentinel path, the function first captures the four-byte value four bytes before the initially captured descriptor pointer.

F-fe78e25e82ff then receives five positional inputs:

1. The destination-slot address.
2. The captured amount-like value.
3. The same captured amount-like value again.
4. The fourth explicit argument.
5. The unresolved register-derived value.

After that call returns, twelve bytes are added to its result. The adjusted value is written into the destination slot as a four-byte value.

The descriptor-pointer field is then read again. This reread occurs **after both the dependency call and publication into the destination slot**.

F-unmapped-external-1 subsequently receives:

1. The adjusted result from F-fe78e25e82ff.
2. The descriptor-pointer value obtained by the post-publication reread.
3. The previously captured amount-like value.

Its return value is unused. If it returns normally, F-eab17af3f0cd returns the destination-slot address.

The second descriptor read need not match the first. It can reflect changes made by F-fe78e25e82ff, changes caused by publication through an alias, or other permitted changes before that read. A descriptor value written by F-fe78e25e82ff reaches the external dependency only if it remains in the descriptor field through the later reread.

The amount-like value is not reread. It remains the value captured from four bytes before the initial pointer, even if the descriptor subsequently identifies a different region.

A zero amount-like value does not bypass either sentinel-path dependency. A zero return from F-fe78e25e82ff is not treated as failure: the visible adjustment produces the address value twelve, subject to the execution environment’s arithmetic interpretation. The function provides no assurance that this address is usable.

### Nonsentinel path

On the nonsentinel path, the initially captured descriptor pointer is written into the destination slot.

After publication, F-54e9fd6505cf receives the address twelve bytes before that same captured pointer. Its return value does not contribute to the result. If it returns normally, the destination-slot address is returned.

This path does not read the amount-like field at offset −4, call F-fe78e25e82ff, call F-unmapped-external-1, or reread the descriptor-pointer field.

Publication can change aliased storage, but it does not cause this body to recapture the pointer used for the F-54e9fd6505cf argument.

### Aliasing and observation order

No disjointness guarantee is established among the descriptor, destination slot, metadata fields, or dependency-managed memory.

In particular, if the destination slot is the descriptor-pointer field, sentinel-path publication overwrites that field before its reread. In the absence of another intervening modification, the external dependency therefore receives the newly published adjusted address as both its destination-like and source-like inputs.

This remains true even if F-fe78e25e82ff previously placed a different pointer in the descriptor field: publication follows that call and can overwrite its change.

On either branch, the final dependency is called after publication. It may therefore observe the published value through any access it possesses. It may also modify the destination slot before returning. The function neither checks nor restores the slot afterward.

These observations describe sequencing within this body. They do not establish atomic publication, synchronization, or guarantees under concurrent access.

## Side effects

Every nonzero path that reaches publication performs one explicit four-byte write into the destination slot. The zero-input path performs none.

The body’s explicit memory reads are:

- One descriptor-pointer read and one sentinel-metadata read on either nonzero path.
- One additional amount-like metadata read and one descriptor-pointer reread on the sentinel path.

Dependency effects are not bounded by those counts. The available evidence does not establish whether the dependencies allocate, release, retain, copy, synchronize, mutate globals, or access aliased memory.

There is no explicit release, rollback, status propagation, or cleanup operation in this body. This absence does not establish that dependencies perform no such operations.

Ownership of the destination slot, published pointer, preceding metadata, and any storage associated with dependency results remains unspecified. No obligation or permission to free, retain, or transfer ownership follows from this function alone.

Publication precedes the final dependency on both branches. If that dependency fails, raises an exception, or does not return, the write may already be observable. No transactional guarantee is established.

The returned address identifies the selected slot on normal completion. Its contents at return are not guaranteed to equal the value this body wrote, because subsequent dependency effects are unknown.

## Dependencies

| Opaque identifier | Established interaction | Unresolved contract |
|---|---|---|
| F-f4fd698ba468 | Called once on every nonzero path with the second and third arguments. Its result plus four bytes determines the destination slot. | Result layout, validity, failure conventions, descriptor mutations, allocation, ownership, and other effects. |
| F-fe78e25e82ff | Called only on the sentinel path with the slot address, duplicated captured metadata, fourth explicit argument, and unresolved register-derived input. Its result is adjusted by twelve bytes before publication. | Allocation semantics, capacity, initialization, failure signaling, ownership, and mutations. |
| F-54e9fd6505cf | Called only on the nonsentinel path, after publication, with the initially captured pointer minus twelve bytes. Its result is unused. | Reference management, metadata changes, release behavior, and other effects. |
| F-unmapped-external-1 | Called only on the sentinel path, after publication and descriptor reread, with the adjusted address, reread descriptor value, and captured amount-like value. Its result is unused. | Identity, transfer semantics, units, overlap support, validity requirements, and failure behavior. |

F-unmapped-external-1 is a document-local opaque identifier because no canonical function ID was supplied for that dependency.

“Allocation-shaped” and “transfer-shaped” are descriptions of argument and result usage only. They do not establish allocation or byte-copy contracts.

## Acceptance examples

These are conditional consequences of the static evidence, not executed tests. Unless stated otherwise, all accessed memory is valid, relevant arithmetic is valid, dependencies return normally, and no unspecified modification changes the stated observations.

### Immediate null return

With a zero second argument, the result is null. No dependency is called and no descriptor-associated memory is accessed by this body. An invalid third-argument address alone does not prevent this branch’s result because it is not dereferenced.

### Direct publication

With a nonzero second argument and sentinel metadata equal to zero:

- F-f4fd698ba468 runs before the descriptor-pointer read.
- The captured descriptor pointer is published into the destination slot.
- F-54e9fd6505cf receives that pointer minus twelve bytes after publication.
- Neither sentinel-path dependency runs.

If F-54e9fd6505cf leaves the slot unchanged, the returned address identifies a slot containing the captured pointer.

### Exact sentinel matching

Metadata represented as signed −2 selects direct publication. Only the exact all-one-bits pattern selects the sentinel path.

### Sentinel publication

With sentinel metadata equal to −1 and amount-like metadata equal to 24, F-fe78e25e82ff receives 24 in both repeated positions.

Its result plus twelve bytes is published before the descriptor reread. F-unmapped-external-1 receives that adjusted address, the descriptor value reread after publication, and 24.

Actual transfer of 24 bytes is not an acceptance requirement established by this body; the external dependency’s units and behavior are unknown.

### Descriptor mutation survives publication

Suppose F-fe78e25e82ff changes the descriptor pointer to a different address. The external dependency receives that address only if it remains in the descriptor field through publication and the subsequent reread.

For example, a non-overlapping destination slot and no intervening descriptor modification permit the dependency’s new descriptor value to reach the external call.

The amount-like input remains the value captured four bytes before the original descriptor pointer. It is not obtained from the replacement pointer.

### Publication overwrites descriptor mutation

Suppose the destination slot is the descriptor-pointer field, and F-fe78e25e82ff changes that field before returning.

Publication then overwrites the field with the dependency result plus twelve bytes. Without another intervening change, the external call receives this published address—not the earlier replacement pointer—as its source-like input. Its destination-like input is the same address.

The amount-like input still comes from the metadata captured before F-fe78e25e82ff was called. Whether the external dependency supports equal or overlapping addresses is unresolved.

### Zero amount-like value

With sentinel metadata equal to −1 and amount-like metadata equal to zero, F-fe78e25e82ff receives zero twice, and the external dependency later receives zero as its third input. There is no zero-amount bypass. Whether either dependency accesses memory for this value is dependency-defined.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and provenance of the unresolved fifth input to F-fe78e25e82ff.
- Complete target ABI, alignment rules, and arithmetic behavior for overflow or invalid addresses.
- Signedness, units, and valid range of the amount-like metadata.
- Dependency result validity, failure conventions, exceptions, and nonreturning behavior.
- Allocation, ownership, lifetime, retention, and release rules.
- Dependency effects on globals, descriptor fields, metadata, and destination storage.
- External-operation overlap guarantees and actual transfer behavior.
- Thread safety, synchronization, and behavior under concurrent mutation.

An inferred intent is to publish existing storage in one metadata state and obtain separate storage in the distinguished state. This interpretation does not establish sharing, reference counting, deep copying, or successful allocation.

## Evidence

Established static evidence consists of the supplied function body and dependency-ID mapping. It supports the input guard, visible widths and offsets, exact sentinel comparison, argument forwarding, publication order, post-publication descriptor reread, preserved amount-like value, and returned slot-address calculation.

The alias-sensitive acceptance conditions follow from the established order: F-fe78e25e82ff returns, publication occurs, and only then is the descriptor field reread. A descriptor change made by that dependency is therefore not independently guaranteed to reach the external call.

No dependency implementation or external API contract was supplied. No runtime tests have been performed. Dependency semantics and runtime outcomes beyond the visible operations remain unresolved.
