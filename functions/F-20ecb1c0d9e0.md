<!-- Trusted provenance: F-20ecb1c0d9e0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-20ecb1c0d9e0

## Purpose

F-20ecb1c0d9e0 answers whether an integer stored at a selected memory location is nonzero.

For an invocation with valid selection arithmetic and a permitted memory access, the required result is:

- **False** when the selected integer is zero.
- **True** when the selected integer is nonzero, including negative values.

The result classifies stored data. It does not indicate whether an address exists, whether an index is within an allocation, or whether an access is safe. In particular, false is not an established error result.

**Established static evidence:** the operation selects a location using two signed-integer inputs, reads an integer there, and returns a Boolean classification.

**Inferred intent:** the location spacing is consistent with testing an entry in regularly arranged integer storage. This does not establish an array, an element count, or a valid index range.

## Interface

### Inputs

The operation accepts two signed-integer values of the same type:

| Input | Contract role |
|---|---|
| Origin | A numeric reference location used to select the storage to inspect. |
| Index | A signed displacement selector, with each step corresponding to four numeric address units. |

The origin is an integer-valued input rather than a pointer-typed input. Its ability to carry an address depends on the target environment. The available evidence does not establish equal integer and pointer widths.

The interface provides no storage extent, element count, ownership descriptor, synchronization mechanism, or error destination.

An index of zero selects the origin. Positive and negative indices select locations on opposite sides of it, subject to valid arithmetic and address interpretation. Negative indices are not explicitly rejected.

### Result

The semantic output is Boolean. No selected address, reference, original integer value, or diagnostic accompanies it.

The result discards all distinctions among nonzero integers. Their signs, magnitudes, and bit patterns do not produce different successful outcomes.

The physical return width, register placement, calling convention, and externally visible Boolean encoding are **documented_only**. A Boolean semantic result does not establish a particular ABI representation.

### Integer widths and storage layout

Both inputs and the integer read from memory use the same represented signed-integer type. Its exact bit width, range, representation, storage size, and alignment are not established by the supplied evidence.

The selection interval is four numeric address units. That interval is independent of the accessed integer’s unknown storage size.

If the environment is byte-addressed and uses the ordinary numeric interpretation of addresses, successive index values select starting locations four bytes apart. Under that additional assumption:

- Four-byte integers occupy adjacent selected regions.
- Smaller integers can leave gaps between selected regions.
- Larger integers can make selected regions overlap.

These are conditional layout consequences, not evidence that the stored integers are four bytes wide or that all selected regions are valid objects.

## Behavior

### Location selection

The index specifies a displacement from the origin in four-unit steps. There is no additional header offset, descriptor lookup, sentinel handling, or traversal through intervening entries.

Selection uses the represented signed-integer domain. Two arithmetic requirements matter separately:

1. The index’s scaled displacement must be valid in that domain.
2. Combining that displacement with the origin must also be valid in that domain.

A mathematically representable final location does not compensate for an invalid intermediate displacement. The evidence provides no explicit widening, saturation, range check, or portable wraparound policy.

The resulting numeric location must admit a suitable address interpretation before the normal-result contract applies. A number’s representability in the input type does not establish that it denotes accessible storage.

Only the selected location needs to support the required read. For a nonzero index, the origin itself need not contain a readable integer. Conversely, a readable origin establishes nothing about the readability of other selected locations.

### Value classification

The operation obtains the integer at the selected location and distinguishes zero from all nonzero values.

The predicate is not a positivity test. It does not accept only one as true, inspect a particular bit, count set bits, or treat a negative value as an error.

Nearby storage does not contribute to the classification. A nonzero integer at the origin cannot make a selected zero produce true, and a zero at the origin cannot make a separately selected nonzero integer produce false.

The evidence supports one logical integer read. It does not establish the number of machine instructions, memory transactions, or addressable units involved in realizing that read.

### Observable ordering

The selected address depends on both inputs. The memory observation depends on that selection, and the returned Boolean depends on the observed integer.

No explicit writes, dependency calls, or additional data reads are present. There is therefore no read/write sequence involving other objects to preserve or describe.

This dependency order does not establish a synchronization protocol, a memory barrier, or a guarantee about speculative activity. It also does not require a separately visible hardware read for every invocation where the execution environment permits ordinary optimization.

### Preconditions for a normal result

The zero-versus-nonzero guarantee requires all of the following:

- Valid signed arithmetic throughout location selection.
- A supported interpretation of the selected number as an address.
- Alignment sufficient for the accessed integer type.
- Accessible storage covering the entire integer object.
- A live object and a permitted access under the applicable execution rules.
- A stored representation valid for the accessed type.
- A concurrency arrangement that does not invalidate the read.

These are caller and environment obligations. F-20ecb1c0d9e0 does not explicitly validate them.

Readability of the first addressable unit is insufficient if the complete integer occupies more than one unit. Likewise, numerical proximity to a valid allocation does not establish that the selected location belongs to it.

The operation places no independently established requirement that the origin identify the first element of an allocation. Any such allocation-level convention would have to come from surrounding usage.

### Errors and abnormal completion

There is no explicit error channel, fallback address, substitute value, or recovery path.

When arithmetic or access requirements fail, neither Boolean result is guaranteed. In particular:

- An unusable address does not have an established false result.
- An out-of-range index does not have an established rejection result.
- Arithmetic overflow does not have an established wrapped-location result.
- An incomplete integer object does not have an established partial-read result.

The exact consequences depend on the applicable language and execution environment. Fault delivery, termination, external recovery, and other invalid-execution consequences remain **documented_only**.

## Side effects

The selected integer is explicitly read, not written. The operation does not clear it, normalize it, or record that it was observed.

There is no explicit allocation, release, callback, global assignment, address retention, or ownership transfer. The result is a Boolean rather than a reference into the selected storage.

The caller must arrange for suitable storage to remain available during the read. The evidence does not establish who allocated that storage, who may later release it, or how its lifetime is managed outside this operation.

The selected address may refer to global storage. The absence of a named global does not mean that global memory cannot be observed.

No atomic or volatile access qualification is expressed. Accordingly, the contract does not establish:

- An indivisible read.
- Synchronization with concurrent writers.
- A particular observation under an invalid data race.
- Suitability for device-register access.
- A requirement to bypass caches or other permitted optimizations.

A read from special-purpose storage may itself have environment-defined effects. The absence of an explicit write is not a universal guarantee of external harmlessness.

## Dependencies

No dependency function calls are visible. There are no dependency function IDs to enumerate and no unknown callees whose global preservation or memory ownership must be assumed.

The operation nevertheless relies on environmental rules for:

| Area | Required interpretation |
|---|---|
| Signed integers | Width, range, valid representations, and arithmetic behavior. |
| Numeric addresses | Mapping from the selected integer value to an address. |
| Object access | Integer size, alignment, lifetime, permissions, and access legality. |
| Execution | Concurrency rules and abnormal-completion behavior. |
| Interface boundary | Argument passing and physical Boolean return representation. |

These are environmental dependencies, not services explicitly invoked by F-20ecb1c0d9e0.

## Acceptance examples

These examples are conditional requirements derived from static evidence. They are not observed executions.

Unless an example deliberately violates a precondition, assume valid arithmetic, a supported address interpretation, and a complete, suitably aligned, readable integer object at the selected location.

### Zero and nonzero values

| Selected integer | Required result |
|---|---|
| Zero | False |
| Seven | True |
| Negative one | True |

An implementation that returns true only for positive integers would not satisfy this contract. Neither would one that recognizes only the value one.

If the selected integer is zero, nonzero neighboring integers do not change the required false result.

### Selection at the origin

With an index of zero, the integer at the origin determines the result.

This does not exempt the origin from access requirements. No special treatment of a zero-valued origin, null-like address, or otherwise unusable location is established.

### Positive displacement

Assume byte-addressed storage and the ordinary numeric address interpretation.

An origin of 4,096 with an index of three selects address 4,108. If the integer at 4,108 is negative one, the required result is true, even if the integer at 4,096 is zero.

No read of the intervening entries is required by the represented behavior.

### Negative displacement

Under the same environmental assumptions, an origin of 4,096 with an index of negative one selects address 4,092.

If that location contains a valid readable zero integer, the required result is false. Rejecting this invocation solely because the index is negative would add a restriction absent from the evidence.

The storage at 4,096 need not itself be readable when the selected object at 4,092 satisfies all requirements.

### Complete-object access

Assume the accessed integer occupies four bytes. If only the first byte at the selected location is readable, the normal Boolean result is not guaranteed.

The small result domain does not reduce the input access to a one-byte flag read.

### Intermediate arithmetic limit

Assume a 32-bit signed domain ranging from negative 2,147,483,648 through positive 2,147,483,647.

An index of 536,870,912 requires a displacement of 2,147,483,648. That displacement exceeds the assumed positive limit.

An origin of negative four would bring the final mathematical location back within range, but it does not make the intermediate displacement valid. No normal Boolean result or portable wrapped-address behavior is established for this invocation.

### Final arithmetic limit

Under the same 32-bit assumption, an origin of 2,147,483,644 and an index of one require a final numeric location of 2,147,483,648.

The four-unit displacement is representable, but the final selection is not. Valid displacement scaling alone is insufficient.

### Unreadable selected storage

Suppose selection arithmetic is valid but the selected address cannot support the required integer read.

Neither true nor false is a required outcome. Treating this case as false would introduce an error-handling policy not supported by the evidence.

## Uncertainties

The following remain **documented_only**:

- Exact input and stored-integer widths, ranges, representations, and sizes.
- Pointer width and the input type’s address-carrying capacity.
- Addressable-unit size and byte order.
- Integer-to-address conversion rules.
- Alignment requirements and consequences of misalignment.
- Allocation boundaries and intended index limits.
- External ownership and storage-lifetime arrangements.
- Required synchronization and permitted concurrent access.
- Physical read granularity, atomicity, and special-memory behavior.
- Consequences of invalid arithmetic or invalid memory access.
- Fault delivery and recovery arrangements.
- Calling convention and physical Boolean representation.

Regular four-unit spacing suggests table-like usage, but that interpretation remains inferred intent. It supplies no additional bounds, object-size, ownership, or safety guarantee.

## Evidence

**Established static evidence:** two signed-integer inputs of the same represented type; numeric selection at four-unit intervals; a selected memory access using that integer type; and a Boolean zero-versus-nonzero result. No explicit validation, writes, function calls, allocation, release, or recovery handling are present.

**Inferred intent:** inspecting an entry in regularly spaced integer storage. The contract does not require that interpretation.

**Unresolved:** target-specific widths, address conversion, storage layout, access legality, and invalid-execution consequences require evidence beyond the supplied representation.

No runtime tests have been performed. The acceptance examples express conditional behavior, not execution evidence.
