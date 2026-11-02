<!-- Trusted provenance: F-5c1bc43ac1f8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5c1bc43ac1f8

## Purpose

F-5c1bc43ac1f8 coordinates two ordered decisions about a caller-supplied object and selects one of three completion paths:

- Return the 32-bit value 2 when the first decision reports a nonzero result.
- Return a 32-bit value read from object offset 28 when the first decision reports zero and the second reports zero.
- Copy a 32-bit value from object offset 32 to a caller-supplied output location, then return zero, when the first decision reports zero and the second reports nonzero.

Before either decision, the function gives a dependency access to a local 32-bit slot initialized from the fourth argument. Every explicit completion path subsequently passes the slot’s current value to another dependency.

These are structural behaviors established by the supplied static evidence. The application-level meanings of the decisions, object fields, slot value, and return values are not established. In particular, the evidence does not justify naming either decision a validation check, calling the final dependency a destructor, or interpreting every nonzero return as an error.

## Interface

The interface has four arguments and a 32-bit return value.

| Position | Contract role | Established representation |
|---|---|---|
| First argument | Object reference supplied to the initial dependency and both decision dependencies; also the base for two possible field reads | An integer-shaped address in the supplied representation |
| Second argument | Destination for the explicit output write on one branch only | Pointer-shaped argument designating a 32-bit storage location |
| Third argument | Unused by the visible implementation | 32-bit value |
| Fourth argument | Initial contents of a local slot that is passed by address to the initial dependency | 32-bit value |
| Return | Branch-selected result | 32-bit value |

### Object layout used by this function

Only the following object-relative storage is directly accessed:

| Byte offset from the first argument | Width | Access condition |
|---|---:|---|
| 28, equivalent to hexadecimal 0x1c | 4 bytes | Read if the first decision returns zero and the second returns zero |
| 32, equivalent to hexadecimal 0x20 | 4 bytes | Read if the first decision returns zero and the second returns nonzero |

No direct access to other object offsets is established. The dependencies may access additional state, but their implementations are not supplied.

The two fields are treated as 32-bit values without visible range checks, masking, arithmetic, or application-level interpretation. Byte order, alignment requirements, and the complete object layout are not established by this function alone.

### Preconditions

For normal completion:

- Each invoked dependency must accept the arguments supplied to it and return normally.
- The local slot must remain usable for the visible accesses throughout the invocation.
- If execution selects the offset-28 path, the corresponding four-byte object field must be readable.
- If execution selects the offset-32 path, that four-byte object field must be readable and the output location must support a four-byte write.
- The address calculations must designate valid storage under the actual execution environment.

These conditions do not establish that a null or otherwise invalid first argument is acceptable on paths without a direct field read: the dependencies receive that argument first and may use it.

The second argument is not visibly dereferenced on the first two completion paths. That fact alone does not establish a general null-output guarantee, because dependency behavior and indirect access through shared state remain unknown.

No visible constraint is imposed on the third argument. Changing only that argument does not change the visible operations described here, assuming the same surrounding state and dependency behavior.

## Behavior

### Entry and initial dependency call

The function first initializes a local 32-bit slot with the fourth argument.

It then invokes F-3794f888f18b with:

1. The address of that local slot.
2. The first argument.

This call precedes both decision calls and every direct object-field access.

Passing the slot by address permits F-3794f888f18b to replace its contents. The later final dependency receives the slot’s value as read at that later point, not necessarily the fourth argument’s original value. The visible caller neither checks a result from this initial call nor provides a fallback if the slot contains an unsuitable value.

The initial call may also change the object or other accessible state. No preservation guarantee is available.

### First decision and immediate completion

After the initial dependency returns, the function invokes F-867a1fcaaaab with the first argument.

Its result is tested for equality to zero. Every nonzero result selects the same branch; there is no visible distinction between positive and negative nonzero results.

If the result is nonzero:

- F-56e2874b0016 is not invoked.
- Neither of the two object fields is directly read by this function.
- The output location is not directly written by this function.
- F-fd58bea73663 is invoked with the current value of the local slot.
- If that invocation returns normally, the function returns the 32-bit value 2.

The value returned by the first decision is not itself propagated to the caller.

### Second decision

If the first decision returns zero, the function invokes F-56e2874b0016 with the first argument.

This call occurs after F-867a1fcaaaab returns. Any state changes performed by the earlier calls therefore precede the second decision.

The second result is also tested only for equality to zero. Nonzero results all select the same output-producing branch.

### Second decision returns zero

If the second decision returns zero, the function:

- Reads four bytes from object offset 28 as its selected return value.
- Invokes F-fd58bea73663 with the current value of the local slot.
- Returns the previously selected 32-bit value if the final dependency returns normally.

There is no direct write to the caller’s output location on this path.

The field is read after both decision calls and before F-fd58bea73663. Consequently, changes to the field made by an earlier dependency can affect the selected result. A later change to the object field by F-fd58bea73663 does not trigger a reread of that field in the visible function.

No restriction is imposed on the selected field value. It may be zero, 2, or any other 32-bit pattern. The return value alone therefore need not uniquely identify the path taken.

### Second decision returns nonzero

If the second decision returns nonzero, the function:

- Reads four bytes from object offset 32.
- Writes that 32-bit value to the location designated by the second argument.
- Invokes F-fd58bea73663 with the current value of the local slot.
- Returns the 32-bit value zero if the final dependency returns normally.

The source field is read before the destination write. The output write occurs before the final dependency call.

There is no visible output initialization before this branch. In particular, the other branches do not explicitly clear the output.

The visible write does not establish that the output still contains that value when the function returns. F-fd58bea73663 runs afterward and has no established memory-preservation contract.

### Completion boundaries

Each explicit completion path invokes F-fd58bea73663 once. This does not establish invocation after an earlier dependency fails to return, transfers control elsewhere, or encounters a fault.

No exception handler, retry, compensating write, or rollback appears in the supplied evidence.

## Side effects

### Direct effects

The direct effects established for this function are:

- Initialization of a local four-byte slot from the fourth argument.
- Exposure of that slot’s address to F-3794f888f18b.
- One conditional four-byte read from either object offset 28 or object offset 32.
- One conditional four-byte write to the second argument’s destination.
- The ordered dependency calls described above.

The function contains no visible direct write to the object through the first argument. The output destination may nevertheless alias object storage, so an output write can also constitute an object modification.

### Aliasing and observation order

No explicit non-aliasing requirement is shown for the object and output destination.

Where the memory model permits a particular alias:

- The offset-32 value is obtained before the output write.
- If the output designates the same four-byte field, the function writes back the value it just read.
- If the output designates the offset-28 field, the output-producing branch overwrites that field without first reading it as a return value.
- The final dependency runs after the output write and may therefore observe the changed storage if it has access to it.

Partially overlapping accesses require valid memory semantics and alignment for the actual platform; this evidence does not supply a portable overlap guarantee.

### Ownership and lifetime

The local slot’s lifetime is limited to this invocation. Its address is supplied to F-3794f888f18b, but no retention or ownership contract for that dependency is available.

The visible function does not directly allocate memory, release memory, or transfer ownership of either caller-supplied address. Such actions cannot be excluded inside dependencies.

The pairing of an initial slot-address call with a final slot-value call suggests a setup/finalization relationship. That is inferred intent, not an established resource-management guarantee.

## Dependencies

| Opaque ID | Visible inputs | Use of result | Invocation condition |
|---|---|---|---|
| F-3794f888f18b | Address of initialized local 32-bit slot; first argument | No result is visibly consumed | First dependency call |
| F-867a1fcaaaab | First argument | Zero/nonzero selects whether evaluation continues | After the initial call returns |
| F-56e2874b0016 | First argument | Zero/nonzero selects field-return versus output-write path | Only when the first decision returns zero |
| F-fd58bea73663 | Current 32-bit slot value | No result is visibly consumed | Once on each explicit completion path |

No dependency is known to preserve global state, the object, the output destination, or other reachable memory.

Because the initial dependency receives the local slot’s address, the evidence also does not establish that the slot becomes immutable after that dependency returns. The final dependency’s argument is the value available when its call is made.

## Acceptance examples

These examples specify conditional observations, not executed tests. Unless stated otherwise, all invoked dependencies return normally and do not introduce additional relevant memory changes.

### First decision short-circuits

Given a first-decision result of any nonzero value:

- The second decision is not called.
- No direct read occurs at object offset 28 or 32.
- No direct output write occurs.
- The final dependency receives the current slot value.
- The result is 2, regardless of the particular nonzero decision result.

### Field-derived return

Given a first-decision result of zero, a second-decision result of zero, and an offset-28 field containing the 32-bit value 9:

- The field is read after the second decision.
- The output destination receives no direct write.
- The final dependency is called after the read.
- The returned value is 9.

If the offset-28 field instead contains zero, this path returns zero without performing the output write.

### Output-producing completion

Given a first-decision result of zero, a nonzero second-decision result, and an offset-32 field containing the 32-bit pattern 0x89ABCDEF:

- That pattern is read and written as four bytes to the output destination.
- The final dependency runs after the write.
- The function returns zero.

The pattern is not narrowed or converted to a Boolean value.

### Replaced slot value

Given a fourth argument containing one 32-bit value, if F-3794f888f18b replaces the slot with a different value and no subsequent action changes it:

- F-fd58bea73663 receives the replacement value.
- It does not receive the original fourth argument merely because that argument initialized the slot.

### Field mutation before and after selection

On the field-return path, if the second decision changes the offset-28 field before returning, the subsequent field read observes the resulting value.

If the final dependency then changes that field again, the visible function does not reread it; its selected return value comes from the earlier read.

### Output overwritten by the final dependency

On the output-producing path, the direct output write must precede the final dependency call. If that dependency subsequently overwrites the destination, the final memory contents reflect its later action. The contract does not promise preservation of the earlier write.

## Uncertainties

The following are **documented_only**:

- The complete ABI, exact pointer width, calling convention, byte order, and alignment rules.
- The application-level object type and meanings of offsets 28 and 32.
- Whether zero and nonzero decision results denote success, failure, readiness, or another condition.
- Whether return value 2 is an error code, status, or ordinary result.
- The role and validity requirements of the fourth argument and local slot.
- Whether the initial and final dependencies implement acquisition/release, context switching, locking, or another protocol.
- Dependency side effects, ownership rules, memory retention, fault behavior, and nonlocal control transfers.
- Concurrency and reentrancy guarantees. No snapshot or atomicity contract is established.
- Behavior for invalid addresses or address calculations outside valid object storage.

The supplied representation uses a signed integer-shaped first argument for address calculations. The field displacements are byte offsets 28 and 32; the evidence does not establish a signed-overflow policy or authorize assuming modular address wraparound.

## Evidence

Established static evidence comprises the supplied function body and the supplied mapping to opaque dependency IDs. It supports the branch structure, four-byte data widths, field offsets, call arguments, and relative ordering described above.

The setup/finalization interpretation is inferred intent only. Dependency internals and application-level semantics remain unknown.

No runtime tests have been performed. The acceptance examples are conditional review criteria, not execution results.
