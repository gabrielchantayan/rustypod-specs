<!-- Trusted provenance: F-afeb873e18d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afeb873e18d1

## Purpose

F-afeb873e18d1 performs two ordered stages of selective zeroing around two dependency calls. Before either call, it saves an incoming 32-bit register value in a four-byte stack-backed temporary.

The first dependency supplies an address-like result used for three 32-bit zero stores. The second dependency receives an address sixteen bytes beyond that result and a reference to the temporary. Its return value supplies the destination for four further 32-bit zero stores and one 8-bit zero store.

This is not established whole-object initialization. The temporary and the two destination regions are not checked for overlap. Their overlap can affect both the value presented to the second dependency and the final memory contents.

An initialization or construction role is **inferred intent**. Object identities, field meanings, allocation boundaries, and lifecycle requirements are not established.

## Interface

### Inputs

The supplied interface declares no ordinary parameters. The body nevertheless consumes the incoming 32-bit representation associated with architectural register `r3`.

That representation is saved before either dependency call. Its semantic meaning is unknown: it could represent a number, flags, an address, a handle, or another bit pattern.

The complete calling convention is not supplied. Additional implicit inputs, including global state, remain **documented_only**.

### Output

The function has no declared return value and supplies no explicit success or failure status.

Its established effects consist of:

- A four-byte write saving the incoming register representation.
- Three 32-bit zero stores based on the first dependency’s result.
- Four 32-bit zero stores and one 8-bit zero store based on the second dependency’s result.
- Whatever effects the dependencies perform.

A value remaining in a machine register does not establish a caller-visible return-value contract.

### Contract notation and widths

This contract uses three address labels:

| Label | Meaning |
|---|---|
| A | The address-like result returned by F-f4fd698ba468. |
| B | The address-like result returned by F-b872376ec5c5. |
| T | The address of the four-byte stack-backed temporary. |

These are specification labels, not asserted object types. Offsets are in bytes.

The explicit zero stores have widths of either 32 bits, occupying four bytes, or 8 bits, occupying one byte. The saved input occupies four bytes.

The address-like results are carried through an integer-valued quantity in the supplied representation. The evidence does not independently establish the complete target address model, pointer provenance rules, integer carrier width, or address-overflow behavior. Calculations such as A + 16 are therefore conditional on meaningful address arithmetic under the applicable execution model. Neither modular wrapping nor a particular overflow failure is specified here.

### Preconditions for ordinary completion

The explicit writes require writable storage at these locations:

| Basis | Required span, inclusive | Use |
|---|---|---|
| T | T through T + 3 | Initial save of the incoming representation. |
| A | A + 4 through A + 15 | First zeroing stage. |
| B | B + 4 through B + 20 | Second zeroing stage. |

These spans describe accesses, not minimum allocation sizes or independent allocations.

The second dependency also receives A + 16. The caller constructs that address-like value without reading memory there. Any requirement that the location be readable, writable, or part of a particular object belongs to the unresolved dependency contract.

Ordinary completion additionally requires both dependencies to return normally and all address calculations and accesses to be permitted. There are no visible alignment, null, range, or overlap checks. Alignment restrictions and invalid-access handling remain **documented_only**.

## Behavior

### 1. Save the incoming representation

The function first writes the incoming 32-bit register representation to the four-byte temporary at T.

This is a memory write, not merely a register-preservation convention. It must be included when reasoning about locations changed by the function. In particular, a location outside both zeroing spans can still change if it overlaps T through T + 3.

The later reference argument designates this storage, not the live architectural register. Subsequent register changes alone do not update the saved representation.

The initial save does not guarantee that the original input remains intact until the second call. The first dependency has unknown memory effects, and the first zeroing stage may overlap the temporary.

### 2. Obtain the first destination

The function calls F-f4fd698ba468 with no explicit arguments shown.

On normal return, its result supplies A. No visible validation intervenes before the caller uses A for stores.

The absence of a check does not establish that every result is valid or that any particular result is a failure sentinel.

### 3. Perform the first zeroing stage

The caller performs these stores in order:

| Order | Destination | Width | Written value |
|---|---|---|---|
| 1 | A + 4 | 32 bits | Zero |
| 2 | A + 8 | 32 bits | Zero |
| 3 | A + 12 | 32 bits | Zero |

Against ordinary writable memory, successful completion clears the twelve consecutive bytes from A + 4 through A + 15.

These operations do not themselves write A through A + 3 or A + 16. That access boundary is not a preservation guarantee: the initial save, dependency effects, or the later B-relative stores may reach those locations through aliasing.

If any first-stage destination overlaps the temporary, the corresponding saved-input bytes are zeroed before the second dependency is called.

### 4. Invoke the second dependency

After all three first-stage stores, the function calls F-b872376ec5c5 with two explicit arguments, in this order:

1. A + 16.
2. The address T.

The first argument is an address calculation, not a value loaded from A + 16.

The second argument is a reference to four bytes of temporary storage. The dependency receives access to their then-current contents. Those contents need not equal the original incoming representation if an earlier operation changed them.

The dependency’s interpretation of either argument is unknown. It may inspect or modify the referenced temporary, subject to its unresolved behavior. The caller performs no subsequent explicit read of the temporary and makes no decision based on its contents.

On normal return, the dependency’s result supplies B. The caller does not establish that B equals A, A + 16, T, or any other particular address.

### 5. Perform the second zeroing stage

The caller performs these stores in order:

| Order | Destination | Width | Written value |
|---|---|---|---|
| 1 | B + 4 | 32 bits | Zero |
| 2 | B + 8 | 32 bits | Zero |
| 3 | B + 12 | 32 bits | Zero |
| 4 | B + 16 | 32 bits | Zero |
| 5 | B + 20 | 8 bits | Zero |

Against ordinary writable memory, successful completion clears seventeen consecutive bytes from B + 4 through B + 20.

The last store writes exactly one byte. It does not itself write B + 21 through B + 23. Those three bytes may nevertheless have changed through the initial temporary write, first-stage zeroing, or dependency effects.

The function then returns without explicit status handling or resource cleanup.

### Ordering and partial execution

The established order is:

1. Save the incoming representation at T.
2. Call F-f4fd698ba468.
3. Perform the three A-relative stores.
4. Call F-b872376ec5c5.
5. Perform the five B-relative stores.
6. Return.

Ordering matters independently of final contents. Combining the zeroing stages into a later operation would remove the first-stage writes visible before the second dependency call.

There is no visible rollback. A dependency that does not return, or an unsuccessful access, can prevent later operations while leaving earlier effects in place. No particular fault-reporting, exception, or recovery mechanism is established.

## Side effects

### Explicit reads and writes

The destination-zeroing operations comprise seven 32-bit stores and one 8-bit store: 29 bytes of write operations. Including the initial four-byte save gives 33 bytes of explicit data writes counted by operation width, not necessarily 33 distinct bytes.

No explicit read from either destination region is shown. The incoming register representation is consumed for the initial save. Dependency reads and writes are unresolved.

For ordinary byte-addressable memory, the specified zero stores clear their covered bytes regardless of byte order. Their widths remain significant: a word store and several byte stores need not be interchangeable for access-sensitive storage. Applicability to device memory remains **documented_only**.

### Aliasing and preservation

Three explicit write spans must be considered:

- T through T + 3.
- A + 4 through A + 15.
- B + 4 through B + 20.

No disjointness among these spans is established. Overlap does not suppress any specified operation.

Relative to values present before the initial save, a location can be guaranteed unchanged by the explicit body writes only if it lies outside **all three spans**. A complete preservation claim also requires that neither dependency nor another relevant actor modify it.

Alternatively, a preservation comparison may start immediately after the initial save. With that baseline, exclusion from the two later zeroing spans and absence of subsequent external or dependency writes are sufficient for the described explicit operations.

The first-stage zeros are not unconditionally preserved until return: F-b872376ec5c5 may overwrite them. Conversely, locations outside the second-stage span may already have changed through the initial save or first stage.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or reference-count operation appears in the caller. This does not establish that either dependency avoids those actions.

The temporary is stack-backed storage associated with this invocation. The caller supplies no persistent replacement and establishes no valid post-invocation use of its address.

Whether F-b872376ec5c5 retains T is unknown. Retention would not by itself make later dereferencing valid.

No synchronization, memory barrier, inter-thread visibility rule, or atomicity guarantee is established.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-f4fd698ba468 | Called once if reached, after the initial save and before the first zeroing stage; no explicit arguments shown; result supplies A. | Implicit inputs, result validity, memory and global effects, allocation, release, ownership, and failure behavior. |
| F-b872376ec5c5 | Called once if reached, after the three A-relative stores; receives A + 16 and T; result supplies B. | Argument interpretation, temporary mutation or retention, relationship of B to its arguments, memory and global effects, ownership, and failure behavior. |

Neither dependency may be assumed to preserve globals, the temporary, earlier destination values, or memory ownership. Subsequent caller operations require the relevant dependency to return normally.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless stated otherwise, assume meaningful address arithmetic, ordinary writable memory, normal dependency returns, no dependency writes to the observed locations, and no concurrent modifications.

### Disjoint zeroing spans and an outside sentinel

If the A-relative and B-relative zeroing spans are disjoint, both spans are zero at completion under the stated assumptions. The second dependency is called only after the A-relative span has been cleared.

A sentinel retains its pre-invocation value only if it lies outside both zeroing spans **and** outside T through T + 3. Excluding only the zeroing spans is insufficient.

If the sentinel’s baseline is instead recorded immediately after the initial save, exclusion from the two zeroing spans suffices for preservation across the remaining described operations.

### Identical returned addresses

If B equals A, the caller first clears offsets 4 through 15, invokes the second dependency, and then clears offsets 4 through 20.

An implementation that performs only the final zeroing would not preserve the required call-relative write order.

### Returned address independent of the first argument

If the second dependency receives A + 16 but returns a different valid B, the final five stores use B.

If B happens to equal A + 16, the final stage clears A + 20 through A + 36. That relationship is conditional on the returned value; it is not imposed by the caller.

### Saved input altered by first-stage aliasing

If T equals A + 4, the first 32-bit zero store clears the complete temporary. The second dependency then receives T pointing to four zero bytes, assuming the first dependency and other actors have not introduced additional changes.

Thus, even without dependency writes, the reference need not expose the original incoming pattern.

### Final-byte boundary and first-stage overlap

If A equals B + 16, the first stage clears B + 20 through B + 31.

Consequently, B + 21 through B + 23 are zero under the stated assumptions, even though the last store itself touches only B + 20.

### Final-byte boundary and temporary overlap

Suppose B equals T − 20, the address calculation is valid, and B + 21 through B + 23 lie outside the A-relative zeroing span.

Those three bytes are T + 1 through T + 3. The initial save writes them. Their pre-invocation values are therefore not preserved merely because neither zeroing stage nor either dependency changes them.

Under the stated assumptions, their final values are the corresponding bytes of the saved incoming representation. The mapping from bit positions to these addresses depends on byte order, which is not established here.

### Preservation beyond the final byte

B + 21 through B + 23 retain their pre-invocation values if all three bytes are outside both the A-relative span and T through T + 3, and neither dependency nor another actor modifies them. The B-relative stores do not reach them.

If their baseline is taken after the initial save, the temporary-span exclusion is unnecessary for that comparison.

### Invalid destination

If either returned address is unsuitable for its required stores, the caller attempts the accesses without a visible guard. No defined error return, recovery path, or rollback may be required from this evidence.

## Uncertainties

The following remain **documented_only**:

- The incoming representation’s semantic meaning and the complete ABI.
- Additional implicit inputs or outputs.
- Object types, field meanings, allocation boundaries, and address provenance.
- The integer address carrier’s width and address-overflow rules.
- Alignment restrictions and invalid-access behavior.
- Whether either result is a base address or an interior location.
- Actual aliasing among A, B, T, globals, and other storage.
- Dependency effects on memory, globals, ownership, and lifetime.
- Reference retention by the second dependency.
- Failure representations, unwinding, and abnormal-return behavior.
- Concurrency, atomicity, and access-sensitive memory semantics.

Initialization remains inferred intent, not a basis for resolving these unknowns.

## Evidence

The supplied static body establishes the initial four-byte save, dependency call order, argument construction, destination offsets, zero-store widths, zero values, and absence of explicit caller-side validation or status handling.

The aliasing conclusions follow from those accesses. In particular, when B equals T − 20, B + 21 through B + 23 coincide with T + 1 through T + 3. Preservation from a pre-save baseline must therefore account for the temporary write as well as both zeroing stages.

No dependency implementations, authoritative platform rules, or runtime observations are supplied. No runtime tests have been performed.
