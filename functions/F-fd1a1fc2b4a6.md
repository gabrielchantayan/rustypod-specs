<!-- Trusted provenance: F-fd1a1fc2b4a6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd1a1fc2b4a6

## Purpose

F-fd1a1fc2b4a6 conditionally replaces a pointer-like value held in a caller-supplied integer slot. If the existing value denotes a different, non-null object, the function performs an indirect call selected through that existing object before storing the replacement.

The established behavior is a comparison, optional indirect call, and conditional slot update. The function does not itself establish that the indirect call destroys an object, releases ownership, decrements a reference count, or frees storage. Those are possible interpretations, not demonstrated effects.

Equality suppresses both the indirect call and the store. A null existing value suppresses the indirect call but does not suppress replacement with a different value.

## Interface

**Inputs**

- **Storage slot:** A pointer to an integer cell containing the encoded existing pointer. The function reads this cell on every invocation. It writes the cell only on a replacement path that reaches the final assignment.
- **Incoming value:** A pointer value that is compared with the decoded existing pointer and, when replacement proceeds, converted to the slot’s integer representation.

**Output**

There is no explicit return value. Observable results consist of slot accesses, any indirect-call effects, and whether control returns normally.

**Representation and widths**

The supplied representation uses an integer cell for pointer storage rather than a pointer-typed cell. It also interprets the first integer in the existing object as an address-like value. Correct operation therefore depends on the target’s integer and pointer representations.

The evidence establishes a numeric displacement of **4** when selecting the indirect-call entry. Because this displacement is added to an integer-valued address before conversion to a pointer, it is not scaled by the size of an integer.

The excerpt does not independently establish:

- The width or signed representation of the integer type.
- The width of data pointers or callable addresses.
- Whether pointer-to-integer conversion retains every relevant pointer bit.
- Whether a callable entry occupies four bytes.
- The overflow behavior of the address calculation.

A conventional 32-bit address model is consistent with the representation, but remains an inference. The literal displacement of four must not be generalized to “one pointer-width” without additional target evidence.

**Preconditions**

The storage slot must support the initial integer read. If the function reaches its assignment, that slot must still support an integer write.

For a distinct, non-null existing value, the value must identify storage from which the dispatch information can be read, and the resulting entry must identify a callable target under the applicable ABI. The address calculation and pointer conversions must also be valid for the target.

The incoming value is not dereferenced by this function. No validity check is performed on the object, if any, that it denotes.

## Behavior

### Initial observation and comparison

The function first reads the integer stored in the supplied slot and interprets that value as a pointer. This observed value determines the equality branch and, if required, the indirect-call target.

It compares the decoded existing pointer with the incoming pointer.

If they compare equal, the function returns immediately:

- No object dispatch information is read.
- No indirect call is made.
- No assignment to the storage slot occurs.

This includes the case in which both values are null. Consequently, equality is a true no-write path, not an assignment of an unchanged value.

The comparison is between pointer values after decoding the slot. It is not a comparison of the contents of the referenced objects. Two distinct pointers to objects with identical contents still take the replacement path.

### Replacement when the existing value is null

If the existing pointer is null and differs from the incoming pointer, the function skips dispatch and stores the incoming pointer’s integer conversion into the supplied slot.

This path does not inspect an object through either pointer. Subject to representation and slot-validity requirements, installing a non-null value into an empty slot requires no object access.

### Replacement when the existing value is non-null

If the existing pointer is non-null and differs from the incoming pointer, the function performs these observable operations in order:

1. Reads an integer at the beginning of the existing object.
2. Uses that integer as an address-like base and adds the numeric displacement four.
3. Reads a callable address from the resulting location.
4. Calls that address.
5. If the call returns normally, stores the incoming pointer’s integer conversion into the original storage slot.

This establishes a two-stage dispatch layout: an address-like word at the start of the existing object leads to a separate entry location at displacement four. It does not establish the layout of any remaining object fields or dispatch entries.

No explicit arguments are visible at the indirect-call site. The evidence does not establish an implicit receiver, register-based argument convention, or the called routine’s actual signature. In particular, it does not justify specifying that the existing object is passed as a conventional receiver argument.

No return value from the indirect call is examined. Normal return leads to the final slot assignment regardless of any value the called routine may produce.

### Ordering across the indirect call

The replacement store occurs **after** the indirect call returns. The function itself does not install the incoming value before calling out.

Immediately before the call, this function has not changed the slot. However, dispatch reads may have observable effects on unusual storage, and the called routine may independently read or change the slot.

After normal return, the function assigns the incoming value without rechecking the slot’s current contents. A callback-side change to that slot does not cancel or redirect the final assignment. If the slot remains valid, the assignment overwrites its then-current integer value.

If the indirect call does not return normally, the final assignment is not reached through the shown control flow. There is no visible recovery, rollback, or alternate completion path.

## Side effects

**Direct effects**

- One initial integer read from the supplied slot.
- On a distinct, non-null existing value, reads used to obtain and invoke the indirect-call entry.
- On either replacement path that completes normally, one integer assignment to the supplied slot.
- No direct dereference of the incoming pointer.

**Indirect effects**

The selected routine can have effects not bounded by this excerpt. It may mutate objects, modify the slot, change global state, allocate or release storage, reenter this function, or terminate control flow. None of those effects is guaranteed.

The function cannot safely be described as preserving globals across the call. It also cannot be described as acquiring ownership of the incoming value or releasing ownership of the existing value.

The existing object might cease to exist during the call without affecting any later direct read from that object by this function: the shown path performs no further object read after the call. In contrast, the storage slot must remain writable for the final assignment. If the call invalidates that slot, normal return alone is insufficient for a valid completion.

**Errors and synchronization**

There are no explicit error codes, null-slot checks, dispatch validation checks, or failure reports. Invalid storage, an invalid callable entry, or an unsupported pointer representation has no specified handling here.

No locking, atomic update, or concurrency protocol is shown. The read, callback, and store are separate operations; they must not be specified as a single atomic replacement.

## Dependencies

The supplied dependency map assigns an opaque identity only to F-fd1a1fc2b4a6. It does not identify the dynamically selected routine.

That dependency is therefore **documented_only**: an unnamed indirect callable selected from the existing object’s leading address-like value at numeric displacement four. Assigning it another opaque function ID would invent an identity not present in the evidence.

Its visible contract is limited to the call boundary:

- It is reached only when the observed existing pointer is non-null and unequal to the incoming pointer.
- Its entry is resolved before the replacement store.
- No explicit arguments are visible.
- Its return value is unused.
- Normal return permits the replacement store.
- Its memory ownership, global-state effects, and control-flow guarantees are unknown.

Target-specific pointer conversions, address arithmetic, data layout, and calling convention are additional prerequisites, not identified function dependencies.

## Acceptance examples

These examples are conditional acceptance criteria derived from the static behavior. They are not reports of executed tests.

### Existing and incoming values are equal

Given a valid readable slot whose decoded pointer equals the incoming pointer, the function returns without an indirect call and without writing the slot.

This criterion also applies when both values are null. The non-null equality case does not require dispatch information in the referenced object to be read.

### Empty slot receives a non-null value

Given a valid writable slot containing a decoded null pointer and a representable non-null incoming pointer, the function stores the incoming pointer’s integer representation and returns. It performs no dispatch call and does not dereference the incoming pointer.

### Non-null existing value is replaced with null

Given a valid non-null existing object, a valid dispatch entry, and a null incoming pointer, the function calls the selected routine first. If that routine returns normally and the slot remains writable, the function then stores the integer representation of the null incoming pointer.

### One non-null value replaces another

Given distinct existing and incoming pointer values, the existing value controls dispatch. The incoming object is not inspected. After normal callback return, the incoming value is stored.

No additional initialization, retention, or validation of the incoming object is required by the shown behavior.

### The callback changes the slot

Given a valid dispatch path whose callback writes a third value into the same slot and returns normally, the final assignment overwrites that third value with the incoming value, provided the slot remains valid.

There is no second equality check against the callback’s replacement.

### The callback does not return normally

If the selected routine terminates execution or otherwise does not return to the call site, the final replacement assignment is not reached. Effects already performed by the routine are not rolled back by any visible mechanism.

### Invalid dispatch information

If a distinct, non-null existing value leads to unreadable dispatch storage or a non-callable entry, the function provides no defined diagnostic or recovery behavior in the supplied evidence. Such an input cannot be accepted as a supported replacement operation.

## Uncertainties

**Inferred intent**

The pattern is consistent with a resource-holding slot that invokes an outgoing-value cleanup operation before replacement. That interpretation explains the equality guard, but neither cleanup semantics nor ownership transfer is established.

**documented_only unknowns**

- Concrete integer, data-pointer, and callable-address widths.
- Pointer conversion fidelity and null-pointer encoding.
- Address arithmetic behavior at representation limits.
- The identity and full ABI of the indirect routine.
- Any implicit arguments supplied by that ABI.
- Object ownership, reference counting, destruction, and deallocation.
- Whether the indirect routine preserves the slot or any globals.
- Permitted aliasing between the slot, existing object, and dispatch storage.
- Reentrancy safety and concurrent-access requirements.
- Platform behavior for invalid memory or invalid callable targets.

Aliasing deserves particular care: the evidence does not promise that the slot and dispatch-related storage are disjoint. The specified access order still applies, but object meaning and storage validity cannot be assumed across the callback.

## Evidence

The established claims above come from the supplied static function body: its initial slot read, pointer comparison, non-null guard, two-stage indirect dispatch, and post-call integer assignment.

The excerpt contains no implementation of the selected routine and no target ABI or data-model declaration. Destruction semantics and a 32-bit target interpretation therefore remain inferences rather than established guarantees.

No runtime tests have been performed. The acceptance examples describe conditional obligations for later verification, not observed outcomes.
