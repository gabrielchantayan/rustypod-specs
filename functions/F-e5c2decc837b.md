<!-- Trusted provenance: F-e5c2decc837b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5c2decc837b

## Purpose

F-e5c2decc837b conditionally replaces an object-associated resource, derives dimensions and stride from extracted geometry, and invokes a context-provided callback with resource metadata and geometry references.

The operation is not transactional. An existing handle can be submitted to a dependency before replacement succeeds. Metadata writes precede allocation, and callback failure does not cause an explicit resource release or rollback. Unknown dependency effects prevent guarantees that earlier field values persist.

Bitmap-like resource preparation is inferred intent. The established behavior is the access sequence, arithmetic, calls, and conditional writes described below. Resource representation and ownership rules are not independently established.

All behavior is derived from static evidence. No runtime tests have been performed.

## Interface

The function accepts four arguments:

1. **Context reference:** addresses a structure containing a manager value, policy, identity, callback argument, and callback address.
2. **Object reference:** addresses mutable metadata, embedded geometry, and a reference to a separate flags structure.
3. **Selector:** a 32-bit unsigned value. The policy gate tests equality with 2; format selection separately tests bit 1.
4. **Optional translation reference:** null or a reference to two readable 32-bit words supplied as translation arguments.

The return value is a 32-bit status:

- **6** on initial identity mismatch.
- **19** when identities match but the policy gate rejects processing.
- Otherwise, the current value of a stack-resident status word at return.

On the allocation-failure path, that word contains the allocation-reported status when tested. On the callback path, the callback’s return value is assigned to it, but a subsequent dependency call can potentially modify the exposed status storage. Consequently, neither the final result nor the final success writes are unconditionally determined by the callback’s return value.

### Accessed context layout

| Byte offset | Width | Role |
|---|---:|---|
| +0x08 | 32 bits | Manager value captured for resource calls |
| +0x14 | 32 bits | Policy compared against a selected global value |
| +0x18 | 32 bits | Identity used by the initial gate |
| +0x3c | 32 bits | First callback argument |
| +0x40 | 32-bit address | Indirect callback target |

### Accessed object layout

| Byte offset | Width | Role |
|---|---:|---|
| +0x48 | 32 bits | Initial identity; overwritten on final success |
| +0x4c | 32 bits | Height |
| +0x50 | 32 bits | Width |
| +0x54 | 32 bits | Stride |
| +0x58 | 32 bits | Resource handle |
| +0x5c | 16 bits | Written with 0x0100 in one format branch |
| +0x5e | 8 bits | Format discriminator |
| +0x64 | 32 bits | Final horizontal coordinate |
| +0x68 | 32 bits | Final vertical coordinate |
| +0x6c | Embedded storage | Geometry passed by reference |
| +0x9c | 32-bit reference | Addresses a structure whose flags word is at +0x04 |

The geometry’s size and internal layout are unknown.

### Preconditions and exposed storage

Every reached memory access requires valid storage. Neither primary reference nor the flags reference is null-checked. A non-null translation reference must provide both words. The callback address must be callable with the observed interface.

F-960c2eabcfd8 receives the address of four consecutive 32-bit endpoint words. All four must be supplied before this function consumes them. F-6dd003795db7 receives the address of a 32-bit status word that this function has not initialized on the allocation path; allocation must write it before the first status test.

These addresses expose live stack storage. No supplied dependency contract forbids retaining them and modifying that storage during later calls while the invocation remains active. Such modifications affect subsequent rereads described below. They do not justify access after the storage’s lifetime ends.

No explicit validation covers endpoint ordering, dimensions, stride, resource capacity, handle validity, or arithmetic overflow.

## Behavior

### 1. Identity and policy gates

The function compares object +0x48 against context +0x18 as 32-bit values. If unequal, it sets the status to 6 and returns. No dependency or callback is invoked, and no direct object write occurs.

If identities match, global reads occur in this order:

1. Read the policy global associated with selector 2, unconditionally.
2. If the selector is not exactly 2, read the other policy global and replace the selected value with this second read.
3. Read the context policy at +0x14 and compare it with the selected value.

Equality produces status 19 and skips all remaining processing.

The non-2 path therefore reads **both** policy globals, not merely the ultimately selected one. Each value is the value obtained by its particular read; stability between reads is not assumed.

Selector equality and the later bit test are distinct. Selectors 2 and 3 use different policy-selection paths but the same format branch.

### 2. Translation, extraction, and alignment

The embedded geometry address is object +0x6c.

If the optional translation reference is non-null, F-074238b57126 receives the geometry address and the two supplied words as its second and third arguments. This call precedes extraction. The argument mapping is established; a relative evaluation order for the two input loads is not separately established.

F-960c2eabcfd8 then receives the geometry address and the address of four consecutive endpoint words. In storage order, this contract calls them:

1. First horizontal endpoint.
2. First vertical endpoint.
3. Second horizontal endpoint.
4. Second vertical endpoint.

Interpreting these as lower and upper geometric bounds is inferred rather than validated.

The function updates the endpoint storage in that order:

- Clear the low six bits of the first horizontal endpoint.
- Clear the low six bits of the first vertical endpoint.
- Add 63 modulo \(2^{32}\) to the second horizontal endpoint, then clear its low six bits.
- Apply the same addition and masking to the second vertical endpoint.

For ordinary ordered coordinates without wrapping, this aligns the first endpoints downward and the second endpoints upward to multiples of 64.

Width is obtained by subtracting the aligned first horizontal endpoint from the aligned second horizontal endpoint modulo \(2^{32}\), interpreting the difference as signed 32-bit, and arithmetically shifting right by six. Height uses the corresponding vertical endpoints.

These computations produce width and height values before any resource dependency is called. Later endpoint-storage changes do not cause width and height to be recomputed. Endpoint addition and subtraction can wrap; negative dimensions are not rejected.

### 3. Existing-resource processing

After dimensions are computed, the function captures the 32-bit manager value at context +0x08. This captured value is used for both resource calls even if later calls alter the context field.

The function follows the current reference at object +0x9c and tests bit 0 of its flags word at +0x04.

If set:

1. F-bf7242c98d06 receives the captured manager and the current handle at object +0x58.
2. After the call returns, this function writes zero to object +0x58.
3. It accesses the flags reference and flags contents again and clears bit 0, preserving the other bits obtained by that read.

The post-call flag update is not guaranteed to target the same referenced structure or preserve the same other bits observed before the call. The dependency may change reachable state.

A zero old handle is not specially excluded. If bit 0 is clear, the release call, zero-handle write, and flag-clearing operation are skipped.

### 4. Format and metadata writes

Selector bit 1 determines the format branch.

**Bit 1 clear:**

- Write byte 2 at object +0x5e.
- Derive stride by adding 3 to the width’s 32-bit bit pattern modulo \(2^{32}\), then clearing the low two bits.
- Write the 16-bit value 0x0100 at object +0x5c.

For ordinary nonnegative widths without addition overflow, stride is width rounded upward to a multiple of four.

**Bit 1 set:**

- Write byte 1 at object +0x5e.
- Derive stride by adding 15 to the width’s 32-bit bit pattern modulo \(2^{32}\), logically shifting right by four, then shifting left by one.
- Perform no direct write to +0x5c.

For ordinary nonnegative widths without overflow, this gives two bytes per group of up to sixteen horizontal units. The encoding represented by those units is unknown.

After either branch, this function writes:

1. Stride at +0x54.
2. Width at +0x50.
3. Height at +0x4c.

These writes precede allocation. They are established write events, not guarantees about field contents after unknown calls. This function contains no explicit rollback of them.

### 5. Allocation and first status test

F-6dd003795db7 receives six arguments:

1. Captured manager.
2. Computed height.
3. Zero.
4. Computed stride.
5. Zero.
6. Address of the status word.

Its 32-bit return value is written to object +0x58 after the call returns and before the status test. Thus this direct handle write occurs even when allocation reports failure.

If the current status word is nonzero, the function returns through its common return path. It performs no callback, normalization translation, restoration translation, or final identity/coordinate writes on that path.

Allocation can modify reachable state, including previously exposed endpoint storage. Its effects on object fields are unknown. Failure therefore establishes that earlier writes occurred, not that all their values remain afterward.

### 6. Flag update, normalization, and callback

If the first status test observes zero, the function rereads the current flags reference and sets bit 0 in its flags word.

It then calls F-074238b57126 with the geometry address and the modular 32-bit negations of the **then-current** first horizontal and first vertical endpoint words.

These words were aligned earlier but may have changed through retained references during intervening calls. The normalization arguments are not guaranteed to equal the negations of the endpoints used to compute dimensions.

After normalization returns, a temporary three-word callback descriptor is prepared:

| Descriptor offset | Width | Value |
|---|---:|---|
| +0 | 32 bits | Reference to object +0x4c |
| +4 | 32 bits | Reference to object +0x6c |
| +8 | 32 bits | 1 if the current format byte equals 2; otherwise 0 |

The format byte is reread after normalization. It need not still reflect the selector if an intervening dependency changed it. The observed descriptor-store order is +8, then +0, then +4.

The callback target comes from context +0x40. It receives the current value at context +0x3c and the descriptor address. These context fields are not snapshots from entry.

The callback’s 32-bit return value is assigned to the status word. This assignment replaces any status-storage changes made during earlier calls, including the callback itself before it returns.

The descriptor contains references into the original object, not metadata or geometry copies. Its storage is temporary; retention is not prohibited by a supplied callback contract, but continued validity after this invocation is not established.

### 7. Restoration, final status test, and success writes

After the callback returns normally and its result is stored, F-074238b57126 is called again. Its arguments are the geometry address and the **then-current** first horizontal and first vertical endpoint words, without negation.

This call occurs regardless of the callback’s returned status. Its arguments are reread after the callback; they need not match the values used for normalization. Calling this “restoration” describes inferred intent, not guaranteed inversion. It also does not explicitly undo the optional entry translation.

After this call returns, the function tests the current status word again. Because allocation exposed its address, the restoration dependency may have modified it. Therefore:

- A zero callback return does not by itself guarantee final success writes.
- A nonzero callback return does not by itself exclude those writes.
- The final returned status is the current status word, not an unconditional saved callback result.

If this second test observes zero, the function performs these writes in order:

1. Read the success-identity global and write it to object +0x48.
2. Reread the first horizontal endpoint, interpret it as signed 32-bit, arithmetically shift right by six, and write the result at +0x64.
3. Reread the **second vertical endpoint**, apply the same signed shift, and write the result at +0x68.

These endpoint reads occur after restoration. No further alignment is performed. The vertical result is neither the first vertical endpoint nor necessarily the computed height.

If the second test observes nonzero, all three final writes are skipped. No explicit resource release or rollback follows.

## Side effects

Direct writes can affect handle, flags, format metadata, dimensions, stride, identity, coordinates, and temporary stack storage.

Dependency and callback effects are additional and unknown. They may change context, object fields, referenced structures, globals, exposed endpoint words, or exposed status storage. Earlier writes alone do not establish persistent post-call values.

Resource ownership is only partially evidenced:

- Flag bit 0 gates submission of the old handle to F-bf7242c98d06.
- Allocation’s returned word is stored regardless of allocation status.
- Allocation status zero causes this function to set flag bit 0 before normalization.
- No explicit cleanup of the replacement handle follows callback failure or a nonzero post-restoration status.

“Release,” “allocation,” and “ownership flag” are inferred roles, not supplied ownership guarantees.

No explicit synchronization, exception handling, or recovery from invalid references is visible. Post-callback restoration is established only when control returns normally.

## Dependencies

| Opaque ID | Observed interaction | Unresolved contract |
|---|---|---|
| F-074238b57126 | Geometry reference and two 32-bit words; optional entry call and calls surrounding callback | Translation semantics, mutation scope, failures, retained-reference access, inversion |
| F-960c2eabcfd8 | Geometry reference and address of four endpoint words | Endpoint meaning, output guarantees, retention and later modification of storage |
| F-bf7242c98d06 | Captured manager and current old handle | Release semantics, ownership transfer, preservation of reachable state |
| F-6dd003795db7 | Manager, height, zero, stride, zero, status address; returns 32-bit handle word | Allocation layout, error domain, handle validity, status-address retention |

The indirect callback has no supplied opaque function ID. Its established interface is a context-associated 32-bit value and the temporary descriptor reference. Its return is stored as status before restoration; the post-restoration status controls final success writes.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Identity mismatch:** Unequal identity fields produce 6 without policy-global reads, dependencies, callback, or direct object writes.

2. **Non-2 policy path:** With matching identities and selector 3, the selector-2 policy global is read first, then the other policy global. The latter read supplies the comparison value. Equality with context policy produces 19.

3. **Ordinary dimensions:** Extraction outputs horizontal endpoints 65 and 129 and vertical endpoints 130 and 193. Alignment produces 64, 128, 192, and 256 in storage order; width and height are both 2.

4. **Format writes:** With those dimensions and passing gates, selector 0 causes format byte 2, stride 4, and the 0x0100 write. Selector 2 causes format byte 1 and stride 2, without a direct +0x5c write. These describe this function’s writes, not preservation across later calls.

5. **Preserved-storage success:** For the preceding endpoints, assume relevant dependencies preserve endpoint storage after alignment, preserve required object state, allocation writes zero status, callback returns zero, and restoration preserves that status. Normalization uses negative 64 and negative 128; restoration uses positive 64 and positive 128. Final coordinate writes are 1 and 4, and the result is zero.

6. **Changed endpoint storage:** After the same dimensions are computed, suppose allocation changes the first horizontal endpoint word to 320 while reporting success. If no later call changes it before normalization, normalization uses negative 320 horizontally; width remains the previously computed 2.

7. **Allocation failure:** If flag bit 0 was set, the old handle is submitted before allocation. Metadata writes have occurred, allocation’s return is then stored as the handle, and a nonzero status skips the callback. Earlier metadata contents are not guaranteed to survive allocation.

8. **Status changed during restoration:** If callback returns 7 but restoration changes the exposed status word to zero, final success writes occur and zero is returned, assuming no subsequent status modification. Conversely, callback zero followed by restoration setting status 7 skips those writes and returns 7.

9. **Endpoint overflow:** A second endpoint of 0xfffffff0 becomes zero after adding 63 modulo \(2^{32}\) and clearing six low bits. No overflow rejection occurs.

## Uncertainties

- **documented_only:** Geometry representation, units, endpoint semantics, and intended coordinate meaning.
- **documented_only:** Policy-global and success-global values, mutability, and concurrency behavior.
- **documented_only:** Dependency ownership, allocation sizing, error meanings, and zero-handle validity.
- **documented_only:** Retained-address behavior and mutations of live endpoint, status, or descriptor storage.
- **documented_only:** Callback mutation permissions, reference retention, and nonlocal failure behavior.
- **documented_only:** Aliasing restrictions and validity requirements beyond the accessed storage.
- **documented_only:** Downstream meaning of negative dimensions, wrapped arithmetic, and altered endpoints.

Resource preparation and inverse-translation restoration remain inferred intent.

## Evidence

Established behavior comes from the supplied static function representation: field widths and offsets, ordered policy-global loads, conditional branches, modular arithmetic, signed shifts, exposed stack addresses, dependency arguments, callback descriptor construction, status rereads, and direct write order.

The representation uses 32-bit address and integer words with explicit 16-bit and 8-bit metadata stores. Broader platform guarantees and dependency implementations were not supplied.

No implementation access, dependency inspection, or runtime testing was performed. Acceptance examples are conditional deductions from static evidence, not observed execution.
