<!-- Trusted provenance: F-40ee893ce7ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-40ee893ce7ab

## Purpose

F-40ee893ce7ab initializes selected fields in caller-supplied storage, conditionally performs dependency calls or an indirect lookup, writes a final byte, and returns the original destination pointer.

The established contract concerns fixed-width memory accesses, their ordering, and conditional calls. A constructor-like purpose is **inferred intent**, not an established object model. The supplied evidence does not establish field meanings, a complete object size, ownership rules, or lifecycle obligations.

Initialization values are not unconditional return-state guarantees. Dependencies run after initialization on nonzero-selector paths, and their effects on accessible memory and globals are unknown.

## Interface

### Inputs

**Destination pointer**

The destination identifies writable caller-supplied storage. Direct destination accesses cover offsets 0 through 20, inclusive:

- Four-byte accesses start at offsets 0, 4, 8, 12, and 16.
- One-byte accesses occur at offset 20.

Valid storage and any platform-required alignment are preconditions. No pointer, capacity, or alignment validation is visible. The function does not directly allocate this storage.

These accesses do not establish a complete allocation size or type. In particular, no direct access to offsets 21 through 23 is shown.

**Selector**

The selector is a 32-bit integer. Its value is stored in the four-byte field at destination offset 4.

Zero bypasses all dependency calls and the conditional indirect lookup. Every nonzero value enters the conditional path; negative values are not excluded. No range check, arithmetic transformation, or normalization is visible.

**Final byte**

The final input is a one-byte value. It is written to destination offset 20 after the conditional work.

There is no visible Boolean conversion. Values such as 0x00, 0x01, and 0xE2 remain distinct byte values.

### Result

On normal completion, the function returns the original destination pointer. It does not return a dependency result or a replacement allocation.

There is no separate success indicator, error code, or explicit failure return.

### Destination layout

| Byte offset | Access width | Direct treatment |
|---|---:|---|
| 0 | 4 bytes | Initialized from an external four-byte datum |
| 4 | 4 bytes | Initialized from the selector; reread on one conditional path |
| 8 | 4 bytes | Initialized to zero |
| 12 | 4 bytes | Initialized to zero; overwritten by the indirect lookup on one path |
| 16 | 4 bytes | Initialized to zero |
| 20 | 1 byte | Initialized to zero; overwritten by the final-byte input at completion |

The table describes accesses, not semantic field names or a recovered structure definition. Four-byte values are specified as values; their in-memory byte order is not independently established here.

## Behavior

### Initialization and branch selection

Initialization is ordered. The function first reads the external four-byte leading datum and stores its value at destination offset 0. It then stores the selector at offset 4, followed by four-byte zeros at offsets 8, 12, and 16, in that order. A one-byte zero at offset 20 completes initialization.

These are distinct accesses. The evidence does not establish an atomic initialization operation, synchronization, or an indivisible publication event.

The decision to enter the conditional work uses the selector input. It does not explicitly reread destination offset 4 for this decision.

Aliasing can matter even during initialization. For example, destination writes must not be assumed independent of external data locations unless the caller or platform provides that guarantee. The leading external value is obtained before the subsequent initialization stores.

### Zero-selector path

A zero selector bypasses both function dependencies and every stage of the conditional lookup.

The only remaining direct action before return is the final-byte write at offset 20.

With valid, independent storage and no external interference, the normally completed destination contains:

- The initially read leading value at offset 0.
- Four-byte zeros at offsets 4, 8, 12, and 16.
- The supplied final byte at offset 20.

### Nonzero selector: first dependency result

For a nonzero selector, the function invokes F-875f8fbbb5eb. No explicit argument is shown for this first invocation.

The returned integer is tested for equality with zero:

- Zero selects the indirect-lookup path.
- Any nonzero result selects the second-call-and-apply path.

No further distinction based on the first result’s sign or magnitude is visible.

The first invocation occurs after all initialization stores. Its preservation properties are unknown. It may affect accessible destination memory, external data used by the lookup, or other state. Neither initialized fields nor external lookup inputs may be assumed unchanged across this call.

### Zero first result: complete indirect lookup

This path performs a three-stage lookup **after the first dependency returns**:

1. It obtains an external address-bearing datum.
2. It dereferences the address supplied by that datum to obtain a base value.
3. It reads a four-byte value at byte displacement 48 from that base.

The retrieved four-byte value is then stored at destination offset 12.

The address-bearing datum and the base obtained through it are distinct lookup stages. The function does not merely read an externally stored base and immediately access its offset-48 field.

The ordering has observable consequences:

- The external address-bearing datum is obtained after the dependency call, not captured before it.
- The intermediate dereference uses the address obtained at that stage.
- The terminal four-byte read uses the base obtained by the intermediate dereference.
- The destination-offset-12 store occurs after the terminal read.

The first dependency may change the external address-bearing datum, the memory reached by its address, or the eventual offset-48 source field. The lookup follows the state available at its respective reads. Constancy or immutability of the address-bearing datum is not established.

Aliasing is likewise not excluded at either address-resolution stage. The external datum, the intermediate storage, and the terminal source region must not be assumed disjoint from the destination or from each other. Earlier destination writes or dependency effects can therefore affect the chain before it is consumed. The function does not visibly capture the whole chain as one atomic snapshot.

Normal completion on this path requires all lookup stages to be valid:

- The external address-bearing datum must be obtainable.
- Its supplied address must support the intermediate dereference needed to obtain the base.
- The resulting base, with byte displacement 48, must identify a readable four-byte region.
- Destination offset 12 must remain writable for the resulting store.

No null check, bounds check, address-range check, or recovery path is visible for these operations. The pointer-width representation and intermediate access width are not independently specified by the supplied evidence; the terminal payload read is explicitly four bytes.

This path does not make a second invocation of F-875f8fbbb5eb and does not call F-b7ada62a0646.

### Nonzero first result: second call and application

On a nonzero first result, the function reads the current four-byte destination field at offset 4 and supplies that value as the explicit argument to a second invocation of F-875f8fbbb5eb.

This is a fresh field read after the first dependency call. It is not necessarily the original selector. If the first call changed the field, the second call receives the changed value.

The second call’s four-byte result is passed to F-b7ada62a0646 together with the destination pointer. There is no visible validity test or zero-result rejection between those calls. A zero second result is forwarded just as a nonzero result is.

Any return value from F-b7ada62a0646 is unused by this function. Its destination modifications, global effects, and ownership effects are not established.

None of the conditional lookup stages described above is performed on this path.

### Common completion

After the selected path completes normally, the function writes the final-byte input to destination offset 20 and returns the original destination pointer.

The final write follows every dependency invocation made by this function. If a dependency changed offset 20 but left the destination valid and writable, this write replaces that byte.

Other fields are not generally restored after dependency execution. Their initialized values can be replaced by dependency effects, aliasing effects, or, at offset 12, the direct lookup-result store.

## Side effects

### Direct writes

The function directly writes only the destination fields listed in the interface:

- One initialization write to each four-byte field at offsets 0, 4, 8, 12, and 16.
- An additional four-byte write at offset 12 on the zero-first-result path.
- Two one-byte writes at offset 20 on every normally completing path.

It does not visibly clear an entire allocation, write trailing padding, allocate memory, or release memory.

### Direct reads

The direct read inventory is:

- The external four-byte datum used to initialize offset 0.
- On the nonzero-first-result path, the current four-byte destination field at offset 4.
- On the zero-first-result path, in order:
  - The external address-bearing datum.
  - The intermediate dereference through the address it supplies, obtaining the base.
  - The four-byte payload at that base plus 48 bytes.

The last three items are separate stages after the first dependency call. Treating the first two as a single base-pointer read would omit an observable indirection and a distinct validity requirement.

### Ownership and lifetime

Caller-supplied storage is used and its original address is returned. The direct operations establish no ownership transfer, reference-count change, or release obligation.

Dependency ownership behavior is unknown. They must not be assumed to preserve or release memory. Normal completion requires the destination to remain valid for subsequent direct accesses, especially the final byte write.

### Failure and partial completion

No explicit rollback, cleanup, or error-return path is shown.

An invalid destination can prevent completion during initialization. An invalid intermediate lookup address can prevent completion before the terminal offset-48 read. An invalid terminal source can prevent completion after the intermediate dereference. A dependency may also fail to return normally.

The evidence does not specify the platform manifestation of invalid accesses. It establishes neither a recoverable error response nor guaranteed completion.

Earlier writes and dependency effects are not visibly undone. The final byte write and pointer return are guaranteed only for normal completion.

### Concurrency and aliasing

No synchronization or non-aliasing contract is visible. The access order does not establish thread safety or a coherent snapshot of the external lookup chain.

In particular, separate observations of the address-bearing datum, intermediate base storage, and terminal payload are not an atomic read of one stable object.

## Dependencies

| Opaque function ID | Observed use | Unknown properties |
|---|---|---|
| F-875f8fbbb5eb | Called once for every nonzero selector with no explicit argument shown. A zero first result selects the lookup. A nonzero first result causes a second invocation with the current destination-offset-4 value; that invocation supplies a four-byte result. | Formal signature, explanation of the two invocation forms, memory and global effects, allocation or release behavior, failure behavior |
| F-b7ada62a0646 | Called only after a nonzero first result. Receives the destination pointer and the second call’s four-byte result. Its return value is unused. | Meaning of the supplied value, destination changes, global preservation, ownership behavior, failure behavior |

The observed invocation forms do not justify inventing an implicit default argument for the first call or assigning undocumented application roles to either call.

The leading datum and the complete three-stage lookup are data dependencies. Their application meanings and mutability policies are not established.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not observed executions. Unless stated otherwise, they assume valid accesses, normal dependency returns, and no unmentioned interference.

### Zero selector

With an external leading value of 0x12345678, selector zero, final byte 0xA5, and independent destination storage:

- Offset 0 receives 0x12345678.
- Offsets 4, 8, 12, and 16 contain four-byte zeros at completion.
- Offset 20 contains 0xA5.
- Neither dependency is called.
- No stage of the conditional lookup is read.
- The returned pointer equals the supplied destination pointer.

### Complete lookup after a zero first result

With selector 7 and a zero first result, suppose the external address-bearing datum supplies address A, the intermediate dereference at A supplies base B, and the four-byte field at B plus 48 contains 0x89ABCDEF.

The function obtains A after the dependency returns, dereferences A to obtain B, reads the four-byte payload at B plus 48, and writes 0x89ABCDEF to destination offset 12. The final byte is written afterward.

F-875f8fbbb5eb is called once; F-b7ada62a0646 is not called.

### Mutation of either address-resolution stage

If the first dependency changes the external address-bearing datum to supply address C before returning zero, the lookup dereferences C rather than relying on the datum’s pre-call value.

Separately, if the datum continues to supply address A but the dependency changes the base stored at A from B to D, the terminal read uses D plus 48.

These cases require the resulting chain to be valid. They do not assume either stage is immutable.

### Mutation of the stored selector

Suppose the selector input is nonzero, and the first dependency changes destination offset 4 to the four-byte value 9 before returning nonzero.

The second invocation receives 9, not necessarily the original selector. Its result is passed to F-b7ada62a0646. The conditional lookup is skipped.

### Zero second result and final-byte replacement

If the first result is nonzero and the second result is four-byte zero, F-b7ada62a0646 still receives zero.

If that dependency writes 0x11 at offset 20 and returns normally, while the final-byte input is 0xE2, the subsequent direct write leaves 0xE2 there. Other dependency modifications are not generally reset.

### Invalid intermediate dereference

If the first result is zero but the external datum supplies an address that cannot support the intermediate dereference, normal completion is not guaranteed, even if some intended terminal payload would otherwise be readable.

No error result or restoration of initialized fields is promised.

### Invalid terminal payload

If the intermediate dereference succeeds but the resulting base plus 48 does not support a four-byte read, normal completion is not guaranteed. The already performed initialization, dependency effects, and earlier lookup reads are not rolled back.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the selector, leading value, initialized fields, final byte, and lookup payload.
- Complete destination type, allocation size, alignment requirements, and byte order.
- Address representation, pointer widths, and the precise width of the intermediate base-fetch access.
- Storage classification, constancy, or immutability of the external address-bearing datum.
- Whether any destination, external datum, intermediate storage, or terminal payload locations alias.
- Formal dependency signatures and the calling-convention explanation for the two observed invocation forms.
- Dependency mutation, reentrancy, allocation, release, and abnormal-termination behavior.
- Stronger caller-imposed lifetime or synchronization requirements.

No selector arithmetic is visible. Relevant integer behavior consists of 32-bit selector storage, four-byte payload transfers, one-byte final storage, and zero/nonzero branching. The lookup uses a fixed byte displacement of 48. Checked address arithmetic, wraparound handling, and pointer-range validation are not established.

A constructor-like interpretation remains inferred intent only.

## Evidence

Established static evidence consists of the supplied function body’s access widths, destination offsets, initialization order, selector branch, dependency-call order, complete indirect lookup, final byte write, and return of the original destination pointer.

The lookup evidence supports three ordered stages: obtaining the external address-bearing datum, dereferencing its address to obtain the base, and reading four bytes at base plus 48. It does not establish that either address-resolution stage is immutable or independent of destination storage.

Dependency implementations, independent type definitions, and platform fault behavior were not established. No runtime tests or fault experiments have been performed. All acceptance examples are conditional consequences of the static evidence, not execution reports.
