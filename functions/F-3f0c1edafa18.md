<!-- Trusted provenance: F-3f0c1edafa18; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f0c1edafa18

## Purpose

F-3f0c1edafa18 performs one of two actions on a supplied object:

- For a zero-valued control argument, it invokes an object-associated operation with the fixed value 16.
- For a nonzero control argument, it selects one of three endpoint configurations, obtains a value through two dependencies, optionally writes a 16-bit marker into another dependency-provided object, delivers the resulting value through an object-associated operation, and invokes a final dependency.

Every explicitly shown normal return is the 32-bit value 1. This value does not distinguish the two paths or report whether dependencies achieved their intended effects.

The established behavior is the dispatch, selection, data access, marker write, and dependency ordering described below. Any interpretation as resource selection, UI activity, activation, or success reporting is inferred intent rather than an established contract.

## Interface

### Inputs

**Object reference**

A pointer-sized argument referring to an object whose observed representation uses 32-bit fields and addresses.

The function accesses these locations conditionally:

| Location | Observed interpretation | Required on |
|---|---|---|
| Object offset +0x00 | Address of an indirect-call table | Both paths, at the point of dispatch |
| Table offset +0xB8 | Indirect-call target receiving the object reference and 16 | Zero-control path |
| Object offset +0xB0 | Address of a secondary structure | Nonzero-control path when the selector byte is zero |
| Secondary structure offset +0x9C | 32-bit word whose low eight numerical bits select a configuration | Same condition |
| Table offset +0xD0 | Indirect-call target receiving the object reference and a derived value | Nonzero-control path |

These offsets describe accessed locations, not complete structure definitions. No total object size, table size, or secondary-structure size is established.

**Control argument**

A 32-bit integer tested for equality with zero.

- Zero selects the short dispatch path.
- Every nonzero bit pattern selects the longer path.
- On the longer path, the original control value is passed unchanged to F-876f4b90d114.

The visible function does not otherwise classify positive, negative, or individual nonzero control values.

### Output

A 32-bit return value equal to 1, provided execution reaches a shown normal return.

Indirect-call return values, if any, are ignored. The returns of F-5179423132f8 and F-876f4b90d114 are also ignored. No visible return-status check converts a dependency failure into another result.

### External state

The function accesses:

- An address-bearing external location used to reach a selector byte.
- Three external 32-bit endpoint sources, called the **normal endpoint**, **alternate endpoint**, and **override endpoint** in this contract.
- State reachable through the input object.
- State returned by F-fdd8589eca2d.

These descriptive names do not establish the application-level meanings of the stored values.

### Preconditions

The function contains no visible null-pointer, bounds, accessibility, or call-target validation.

For a particular execution to complete normally:

- Every memory location reached on that execution must be readable or writable as required.
- Each selected indirect-call target must be callable with the observed arguments.
- Dependencies must accept the supplied arguments and return normally.
- Addresses used for multi-byte accesses must satisfy the execution environment’s access requirements.

These requirements are path-specific. The zero-control path does not visibly require the selector state, secondary structure, endpoint sources, or marker destination to be usable.

## Behavior

### Zero-control path

When the control argument equals zero:

1. The function obtains the call-table address from object offset +0x00.
2. It obtains the target stored at table offset +0xB8.
3. It invokes that target with the original object reference and the integer value 16.
4. If the invocation returns normally, the function returns 1.

No direct call to any named dependency occurs on this path. In particular, the function does not visibly inspect the selector byte, read an endpoint, access object offset +0xB0, perform the marker write, or call F-876f4b90d114.

The invoked target may itself have arbitrary effects; the absence of those operations in this function does not constrain that target’s implementation.

### Nonzero-control path: initial operation and selection

For every nonzero control argument, F-5179423132f8 is called first with no explicit arguments.

Only after that call returns does the function inspect the selector byte through the external address-bearing location. Consequently, changes made by F-5179423132f8 to that state can affect the selected branch. The selector is not established as an entry-time snapshot.

The selector determines the next step:

- A nonzero selector selects the override configuration.
- A zero selector causes inspection of the secondary structure.

For the latter case, the function reads the address at object offset +0xB0 and then reads a 32-bit word at secondary-structure offset +0x9C. It tests that word after masking with 255:

- Low eight numerical bits equal to zero select the normal configuration.
- Any nonzero value in those bits selects the alternate configuration.

Bits 8 through 31 do not affect this decision. The access is nevertheless a 32-bit read, not merely a byte read. Interpreting the tested numerical bits as a particular addressed byte would require an endianness assumption.

### Deriving the dispatched value

All three configurations first call F-ef0964bd9b8a with no explicit arguments.

Its 32-bit result becomes the first argument of F-1958950ee503. The remaining arguments are selected as follows:

| Configuration | Second argument | Third argument | Fourth argument |
|---|---|---|---|
| Normal | Normal endpoint minus 1 | Normal endpoint | 0 |
| Alternate | Alternate endpoint minus 3 | Alternate endpoint | 0 |
| Override | Override endpoint minus 2 | Override endpoint | 0 |

The table expresses the relationship between endpoint source and argument calculation. It does not guarantee a coherent snapshot across repeated reads of that source.

For the normal and alternate configurations, the supplied representation places these operations after F-ef0964bd9b8a returns:

1. Read the selected endpoint and form the subtraction-derived argument.
2. Read the selected endpoint for the third argument.
3. Call F-1958950ee503.

Thus, F-ef0964bd9b8a may affect endpoint values subsequently observed. Repeated endpoint reads must not be replaced by an assumed immutable value without an additional stability precondition.

For the override configuration, both endpoint-derived expressions are evaluated after F-ef0964bd9b8a and before F-1958950ee503 receives its arguments. The supplied representation does not establish the relative evaluation order of those two expressions.

The endpoint values and subtraction operands are shown as 32-bit quantities. Ordinary values with representable subtraction results have the relationships shown in the table. At arithmetic boundaries, the supplied material does not independently establish a language-level overflow guarantee. In particular, the contract does not assume saturation, wider intermediate arithmetic, or validation of endpoint values.

F-1958950ee503 returns the 32-bit value eventually supplied to the object-associated operation at table offset +0xD0. This function does not visibly inspect that result before using it.

### Marker write for a zero selector

Both configurations selected under a zero selector—normal and alternate—perform an additional operation after F-1958950ee503 returns:

1. Call F-fdd8589eca2d with no explicit arguments.
2. Treat its result as a base address.
3. Write the 16-bit value 1 at base offset +0x80.

The write is exactly a 16-bit store in the supplied representation. It is not shown as a 32-bit assignment, a masked update, or a read-modify-write operation.

No prior read of that destination is visible. No address validation is visible. The function does not explicitly write adjacent bytes.

The override configuration skips both F-fdd8589eca2d and this marker write. That does not prove the marker remains unchanged: other dependencies may affect the same storage.

### Final dispatch and trailing operation

After value derivation and, when applicable, the marker write, the function:

1. Reads the call-table address from object offset +0x00.
2. Reads the target at table offset +0xD0.
3. Invokes that target with the original object reference and the 32-bit result from F-1958950ee503.
4. Calls F-876f4b90d114 with the original object reference and original nonzero control argument.
5. Returns 1 if all preceding operations return normally.

The +0xD0 target is obtained late, after the preceding dependencies and optional marker write. The function does not establish or retain an entry-time target for this dispatch. Earlier dependencies therefore cannot be assumed to preserve the table reference or target used here.

The trailing dependency runs after the indirect call returns. If that call does not return normally, the visible sequence does not reach F-876f4b90d114.

## Side effects

### Directly established effects

The only explicit ordinary data-memory write shown in the function is the 16-bit store of 1 at offset +0x80 from the address returned by F-fdd8589eca2d. It occurs only on the nonzero-control, zero-selector path.

The function also makes the calls described above. Their effects are part of the observable execution, but their internal behavior is not supplied.

### State preservation

The function provides no established guarantee that any dependency preserves:

- The selector or endpoint sources.
- The input object or its call table.
- The secondary structure.
- The marker destination.
- Other external state.

The sequencing contract matters because later reads can observe effects of earlier calls. The function does not visibly recheck the selector or secondary-word condition after making the corresponding branch decision.

### Ownership and lifetime

No direct allocation, deallocation, retain, release, or ownership transfer is visible.

The following remain unspecified:

- Ownership of the input object.
- Ownership and lifetime of the secondary structure.
- Whether the value returned by F-ef0964bd9b8a denotes storage or another kind of handle.
- Whether F-1958950ee503 consumes, borrows, transforms, or retains its first argument.
- Ownership and lifetime of the address returned by F-fdd8589eca2d.
- Whether the final indirect call retains or consumes its second argument.

The absence of a visible release does not establish a leak. The absence of a visible allocation does not establish that dependencies allocate nothing.

### Errors

There is no visible local error branch, retry, rollback, or cleanup path.

Invalid memory, an invalid call target, a dependency that fails to return, or a dependency-specific failure mechanism can prevent normal completion. The supplied material does not establish how such events are represented or handled by the surrounding environment.

## Dependencies

| Opaque ID or dispatch location | Observed role |
|---|---|
| F-5179423132f8 | First operation on the nonzero-control path; no explicit arguments; return ignored |
| F-ef0964bd9b8a | Supplies the first argument to F-1958950ee503; called once in every selected nonzero-control configuration |
| F-1958950ee503 | Receives four arguments and supplies the value for the +0xD0 dispatch |
| F-fdd8589eca2d | Supplies the base address for the 16-bit marker write; called only when the selector is zero |
| F-876f4b90d114 | Trailing operation after the +0xD0 dispatch; receives the original two inputs |
| Table slot +0xB8 | Unknown indirect target used only for zero control |
| Table slot +0xD0 | Unknown indirect target used only for nonzero control |

No opaque identity or internal contract is available for either indirect target.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless otherwise stated, they assume valid memory, normally returning dependencies, stable endpoint values during their reads, and representable subtraction results.

### Zero control bypasses selection

Given a control value of zero:

- The +0xB8 target receives the object reference and 16.
- None of the named dependencies is directly invoked by this function.
- No selector, endpoint, secondary-word, or marker access is performed directly.
- The normal return value is 1.

The selector and endpoint contents are irrelevant to this path.

### Normal configuration

Given nonzero control, a selector byte of zero after F-5179423132f8, a secondary word of 0xABCD1200, and a stable normal endpoint of 100:

- The high bits of the secondary word do not select the alternate configuration.
- F-1958950ee503 receives the result of F-ef0964bd9b8a, then 99, 100, and 0.
- The marker is written as a 16-bit value of 1 after that dependency returns.
- The +0xD0 dispatch follows the write.
- F-876f4b90d114 follows the dispatch.

### Alternate configuration

Given nonzero control, a zero selector, a secondary word of 0x00000080, and a stable alternate endpoint of 100:

- F-1958950ee503 receives the derived arguments 97, 100, and 0.
- The normal endpoint is not selected.
- The same marker-write and trailing-call ordering as the normal configuration applies.

### Override configuration

Given nonzero control, a nonzero selector, and a stable override endpoint of 100:

- The function does not read the secondary-structure address or secondary word.
- F-1958950ee503 receives the derived arguments 98, 100, and 0.
- F-fdd8589eca2d is not called by this function.
- No explicit marker write occurs.
- The +0xD0 dispatch and F-876f4b90d114 still occur in that order.

### Earlier dependencies affect later observations

If F-5179423132f8 changes the selector from zero to nonzero, selection follows the value read after that call.

If F-ef0964bd9b8a changes the selected endpoint, subsequent endpoint reads observe state after that call rather than a value captured at function entry.

### Nonreturning dispatch

If the selected +0xD0 target does not return normally, F-876f4b90d114 and the explicit return of 1 are not reached through the shown sequence.

## Uncertainties

The following are **documented_only**, not resolved by assumptions:

- Application-level meaning of the control argument, endpoint sources, selector, marker, and derived value.
- Full types and layouts of the participating objects.
- Internal behavior and ownership contracts of all dependencies and indirect targets.
- Relative evaluation order of the override endpoint expressions.
- Arithmetic-boundary behavior beyond the shown 32-bit operations.
- Atomicity, synchronization, volatility, and permissible concurrent mutation.
- Error propagation, unwinding, and recovery behavior.
- Whether external callers interpret the constant return value as success.

Possible intent: the function selects a value, records a condition on two branches, delivers the value, and performs a trailing operation. That interpretation adds no guarantees beyond the established sequence.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes the two top-level paths, conditional structure accesses, low-eight-bit test, dependency identities and arguments, access widths, offsets, explicit marker store, late indirect dispatch, and constant normal return.

No dependency implementations, call sites, complete type definitions, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are conditional consequences of the visible behavior, not claims of observed execution.
