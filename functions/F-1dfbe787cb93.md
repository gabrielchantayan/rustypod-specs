<!-- Trusted provenance: F-1dfbe787cb93; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1dfbe787cb93

## Purpose

F-1dfbe787cb93 coordinates an initial preparation operation, two calls to the same computation dependency, two adjustments to recovered local storage, and a final delegated operation.

The established behavior is the call sequence and data flow described below. The supplied static representation does not establish the application-level meaning of the object, the computations, the local adjustments, or the final operation. Terms such as “object,” “preparation,” and “adjustment” describe their structural roles, not a verified domain model.

This contract describes normal execution when the required memory accesses and dependency calls complete. It does not establish behavior for invalid addresses, arithmetic overflow, dependency failure, or other abnormal execution.

## Interface

### Inputs

The function receives two arguments:

| Input | Established representation | Use |
|---|---|---|
| Object reference | A recovered 32-bit integer used as an address | Passed to the preparation and final dependencies; used as the base of three four-byte field accesses |
| Forwarded value | An opaque four-byte value | Passed unchanged to the final dependency |

The address-bearing first argument is not established as a portable integer handle. Its use requires addressable memory at the field locations described below.

The second argument has no established interpretation as a pointer, signed integer, enumeration, or ownership-bearing resource. This function performs no visible arithmetic or dereference on it.

### Object layout accessed by this function

Offsets are byte offsets from the object reference:

| Offset | Access width | Access pattern |
|---|---:|---|
| `0x38` | Four bytes | Read separately for each of the two computation calls |
| `0xAC` | Four bytes | Read for the final call |
| `0xB0` | Four bytes | Read for the final call |

These accesses do not establish the object’s complete size, field types, packing, alignment requirements, or endianness. They establish only the referenced locations and widths.

### Global inputs

Two global values participate:

- **Global A:** a four-byte value captured after the preparation call, and also read directly for the first computation call.
- **Global B:** a value supplied as the third argument of the second computation call. Its declaration and exact access width are not provided in the supplied representation.

The saved copy of Global A and the subsequent direct read of Global A are distinct operations. Their equality must not be assumed if intervening activity can change the global.

### Outputs

The function has no explicit return value.

Its externally visible effects, if any, arise through dependencies. The visible body also performs two arithmetic updates to local storage. Whether those updates affect the final dependency depends on the actual storage layout and the dependencies’ memory-access behavior.

There is no explicit success indicator, error code, output-length result, or ownership-transfer result.

## Behavior

### 1. Preparation

The function first calls F-960363c07b32 with:

1. The object reference.
2. The address of a local region represented as four bytes.

No initialization of that four-byte region is visible before the call.

The preparation dependency’s return value, if it has one, is not used in the supplied representation. Execution continues without a visible status check.

The actual amount of storage that F-960363c07b32 may access is unknown. The four-byte recovered declaration is not evidence that the dependency accesses only four bytes.

### 2. Capture Global A

After the preparation call returns, the function reads Global A into a local four-byte saved value.

This saved value is retained for the second computation call. It is not refreshed after the first computation call.

Because the capture occurs after preparation, a change made to Global A by F-960363c07b32 can affect the captured value.

### 3. First computation

The function calls F-5710a8628cba with:

1. The four-byte value read from object offset `0x38`.
2. A direct read of Global A.
3. The constant `0x3080`, numerically 12,416.

The returned value is represented as a 32-bit signed integer and retained for a later local addition.

The first call’s second argument is not expressed as the previously saved copy. Therefore, the contract preserves two distinct observations of Global A: the earlier capture and the read used for this call.

The relative evaluation order of the object-field read and the direct global read is not established. Both supply arguments to this call and occur after the earlier capture in the supplied representation.

### 4. Second computation

After the first computation call returns, the function calls F-5710a8628cba again with:

1. A new four-byte read from object offset `0x38`.
2. The saved copy of Global A captured before the first computation call.
3. Global B as observed while preparing this second call.

The returned value is represented as a 32-bit signed integer and retained for a later local subtraction.

The object field at `0x38` is not cached across the two calls. If the first computation changes that field, the second call can receive the changed value.

Likewise, Global B is not visibly captured before the first computation. A change to Global B during the first call can therefore affect the second call.

In contrast, a change to Global A during the first call does not replace the saved second argument used by the second call.

### 5. Local arithmetic updates

Both computation calls finish before either visible arithmetic update occurs.

The function then:

- Adds the first computation result to one recovered local signed integer.
- Subtracts the second computation result from another recovered local signed integer.

The addition precedes the subtraction.

No explicit initialization of either integer is visible in the supplied function body. Their usable initial values, if any, may depend on memory written by a dependency or on limitations of the recovered storage representation. Neither source of initialization is established.

Recovered stack-offset information suggests that the two adjusted integers begin four and twelve bytes after the start of the four-byte local region passed to preparation. This supports a possible interpretation of a larger shared local record, but does not establish that interpretation as a valid layout contract.

The arithmetic is represented using signed 32-bit integers. The supplied evidence does not establish overflow handling. In particular, it does not justify specifying modulo-\(2^{32}\) wrapping, saturation, trapping, or rejection.

### 6. Final delegated operation

After both local arithmetic updates, the function calls F-3f2c2a170c13 with:

1. The original object reference.
2. A four-byte read from object offset `0xAC`.
3. A four-byte read from object offset `0xB0`.
4. The original forwarded value.
5. The address of the same local region passed to preparation.

The two object fields are read at this stage, rather than captured before the earlier calls. Changes made by earlier dependencies can therefore affect the values supplied here.

The relative order of the reads at `0xAC` and `0xB0` is not established. Both are argument reads for the final call.

The final dependency’s return value, if any, is ignored. If that call returns normally, this function returns without a value.

## Side effects

### Direct effects

The visible body directly:

- Reads object fields and globals as described above.
- Retains temporary values locally.
- Updates two recovered local integer locations.
- Passes the same local-region address to the preparation and final dependencies.

There are no visible direct writes to the object fields or global values. That observation does not imply that they remain unchanged: each dependency may have side effects not supplied here.

### Dependency-visible state

No purity or global-preservation guarantee is available for any dependency.

Consequently:

- Preparation may alter values read later.
- The first computation may alter the object field reread by the second computation.
- Either computation may alter object fields read by the final call.
- Changes to Global A after its capture do not alter the saved copy used for the second computation.

The apparent adjacency of local storage may permit dependency writes to affect the subsequent arithmetic or final operation. Whether such access is valid is unknown.

### Ownership and lifetime

The function visibly allocates no heap memory and explicitly releases no resource.

The local-region pointer refers to storage associated with this invocation. No lifetime beyond the invocation is established. Whether either recipient retains that pointer, copies its contents, or treats it as a larger structure is unknown.

Passing the object reference, its field values, or the forwarded value does not itself establish an ownership transfer. No dependency may be assumed to preserve or release memory merely from its position in this sequence.

### Errors and abnormal completion

There is no visible validation of:

- The object reference.
- Field-address accessibility or alignment.
- Global values.
- Dependency return values.
- Initialization of the adjusted local integers.
- Arithmetic range.

There is also no visible retry, fallback, rollback, or cleanup branch.

If a dependency does not return normally, the remaining sequence is not established to execute. Handling of faults, exceptions, nonlocal exits, or other failure mechanisms is outside the supplied evidence.

## Dependencies

| Opaque ID | Role here | Arguments supplied | Result treatment |
|---|---|---|---|
| F-960363c07b32 | Initial preparation | Object reference; local-region address | No result consumed |
| F-5710a8628cba | Two ordered computations | Object field, global-derived value, and constant or Global B | First result added locally; second result subtracted locally |
| F-3f2c2a170c13 | Final delegated operation | Object reference; two object fields; forwarded value; local-region address | No result consumed |

These roles do not establish the dependencies’ full contracts. Their memory-access extents, ownership rules, global effects, return-value meanings, and failure behavior remain unspecified.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of runtime tests.

### Ordered dependency calls

If all calls and intervening accesses complete normally, the dependency invocation order must be:

1. F-960363c07b32.
2. F-5710a8628cba with third argument 12,416.
3. F-5710a8628cba with third argument Global B.
4. F-3f2c2a170c13.

Moving the local addition ahead of the second computation would not preserve the visible sequence.

### Object field changes between computations

Suppose offset `0x38` initially contains value \(P\), and the first computation changes it to \(Q\).

The first computation receives \(P\). The second receives \(Q\), provided no other intervening action changes the field. Reusing \(P\) for both calls would not preserve the separate reads.

### Global capture survives a dependency change

Suppose Global A contains \(A\) when captured and when read for the first computation. Suppose that computation changes Global A to \(B\).

The second computation still receives \(A\) as its second argument. Substituting a fresh read yielding \(B\) would change the specified data flow.

### Conditional local arithmetic

Suppose valid initialization has established the two adjusted local integers as 100 and 80. Suppose the computation results are 7 and 9, and all values are representable as signed 32-bit integers.

The visible updates produce 107 and 71, respectively. This example does not establish how initialization occurred or prove that the final dependency reads those locations.

### Late reads for the final call

Suppose the earlier dependencies change object offsets `0xAC` and `0xB0` to \(R\) and \(S\).

The final dependency receives \(R\) and \(S\), assuming they remain unchanged during argument preparation. Values captured before the earlier calls would not satisfy this behavior.

### No inferred error short-circuit

If F-5710a8628cba returns zero normally, no visible branch interprets it as failure. The sequence continues, and the corresponding arithmetic adjustment uses zero. A special failure meaning for another returned value is not established.

## Uncertainties

The following are **documented_only**, not implementation assumptions:

- The application-level purpose of the operation.
- The complete object layout and field meanings.
- The target address model, alignment rules, and endianness.
- Global B’s exact width and interpretation.
- The true size and layout of the local region shared with dependencies.
- Whether preparation initializes either adjusted integer.
- Whether the final dependency observes the adjusted locations.
- Signed-overflow behavior and the validity of any uninitialized local read.
- Dependency memory ownership, pointer retention, and global side effects.
- Concurrent or asynchronous modification behavior.
- Failure propagation and abnormal-exit semantics.

A plausible inferred intent is to prepare local state, compute two adjustments from an object field, apply them, and submit the resulting state. That interpretation is not established strongly enough to replace the explicit read and call sequence.

## Evidence

The supplied static representation establishes the dependency order, argument sources, explicit four-byte accesses, constant 12,416, saved-global reuse, repeated object-field read, local arithmetic order, and absence of an explicit return value.

Recovered type and stack-layout information supports discussion of 32-bit signed arithmetic and nearby local storage, but does not independently establish the original aggregate type, valid initialization, or dependency access bounds.

No dependency bodies, global declarations, caller contracts, or runtime observations were supplied. No runtime tests have been performed.
