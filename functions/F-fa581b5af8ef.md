<!-- Trusted provenance: F-fa581b5af8ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa581b5af8ef

## Purpose

F-fa581b5af8ef performs two ordered indirect invocations through a dispatch entry associated with a caller-supplied receiver. It forwards the receiver and an opaque 32-bit argument to both invocations. The third argument differs: the first invocation receives bits 8–15 of an unsigned 32-bit input, and the second receives bits 0–7.

The established behavior is high-byte-first dispatch of the input’s low 16 bits. Bits 16–31 do not contribute to either derived argument.

A possible intent is to emit or process a two-byte quantity in high-byte-first order. That interpretation is not established: the indirect dependency’s meaning and externally visible effects are unknown. The contract therefore describes two calls, not necessarily two writes, two transmitted bytes, or a particular encoding.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Representation | Established role |
|---|---|---|
| Receiver | Pointer to storage whose initial word supplies a dispatch-table address | Used for indirect target selection and forwarded unchanged as the first argument of both invocations |
| Opaque value | 32-bit value without established signedness or semantic type | Forwarded unchanged as the second argument of both invocations |
| Packed value | Unsigned 32-bit integer | Supplies two independently extracted 8-bit quantities |

The receiver is not itself the dispatch-table address. Its initial word supplies that address. The callable entry is obtained from byte offset 164, hexadecimal `0xA4`, relative to that address.

The supplied representation uses 32-bit words. It consequently supports a 32-bit receiver-header and dispatch-entry model. A portable implementation must preserve the represented lookup behavior rather than silently substitute the native pointer width of a different execution environment.

The opaque argument is not established to be an integer quantity, handle, pointer, or owned resource. Its 32-bit representation is preserved without interpretation.

### Output

The function has no explicit result value. It does not expose either indirect invocation’s return value.

Normal completion requires both invocations to return control normally. The supplied evidence does not define an error result, exception translation, recovery mechanism, or completion status.

### Preconditions

For ordinary successful execution:

- The receiver must support the dispatch lookup used for the first invocation.
- The selected dispatch entry must designate a callable target compatible with the three supplied arguments.
- After the first invocation returns, the receiver must still support the lookup needed for the second invocation.
- The entry selected for the second invocation must also designate a compatible callable target.

These are execution preconditions, not validations performed by this function. No receiver-null check, table-null check, callable check, or explicit bounds check is shown.

## Behavior

### Argument derivation

The first derived quantity is the unsigned value represented by bits 8–15 of the packed input. Its numerical range is 0 through 255.

The second derived quantity is the unsigned value represented by bits 0–7 of the packed input. Its numerical range is also 0 through 255.

Each derived quantity is supplied as the third argument to its respective invocation. Although each contains only eight significant bits, the evidence does not establish an eight-bit parameter type or an eight-bit calling-convention slot for the dependency.

The extraction is numerical, not a read of the packed input’s byte-addressed memory representation. Consequently, the specified ordering does not depend on host memory endianness.

There is no rounding, saturation, signed-byte interpretation, or carry propagation. Bits above bit 15 are discarded for both calls. For example, inputs differing only in their upper 16 bits produce the same two derived argument values.

### First invocation

The function obtains the receiver’s dispatch-table address and selects the callable entry at byte offset `0xA4`.

It invokes that target with:

1. The original receiver pointer.
2. The original opaque 32-bit value.
3. The quantity extracted from bits 8–15 of the packed input.

The first invocation must complete normally before the second invocation occurs. There is no branch that skips this invocation based on either extracted quantity being zero.

### Second invocation

After the first invocation returns, the function again resolves the entry at offset `0xA4` through the receiver’s dispatch-table reference.

It invokes the resulting target with:

1. The original receiver pointer.
2. The original opaque 32-bit value.
3. The quantity extracted from bits 0–7 of the packed input.

The repeated lookup is behaviorally significant. The first invocation may change the dispatch-table reference stored in the receiver, change the relevant entry in a table, or otherwise affect subsequent dispatch. The evidence does not justify resolving a callable once and reusing it for both invocations.

The two invocations therefore use the same dispatch location rule, but are not guaranteed to reach the same concrete function.

The receiver pointer passed to the second invocation is the same argument value originally supplied to this function. That does not imply that the receiver’s contents remain unchanged.

### Completion and interruption

After the second invocation returns normally, this function returns without additional explicit work.

No local error handling is shown. If the first invocation does not return normally, the evidence establishes no path to the second invocation. If the second invocation does not return normally, the evidence establishes no normal return from this function.

Invalid receiver storage, an invalid table address, or an invalid callable entry is not mapped to a defined result by this contract. The resulting execution behavior depends on the environment and dependency semantics.

## Side effects

### Established direct effects

The function performs dispatch-related reads and two ordered indirect invocations. It contains no explicit assignment to receiver fields, dispatch entries, global state, or caller-provided output storage.

The established logical read order is:

1. Resolve the first dispatch target through the receiver.
2. Perform the first invocation.
3. Resolve the second dispatch target through the receiver.
4. Perform the second invocation.

This specifies dispatch dependencies and call ordering, not an exact machine-instruction sequence, atomicity guarantee, or complete order of argument evaluation.

### Dependency-mediated effects

Either invocation may produce effects not visible in the supplied body, including changes to receiver state, shared tables, global state, external resources, or memory ownership.

No preservation guarantee may be inferred for those resources. In particular:

- The first invocation need not preserve the receiver’s dispatch-table reference.
- Neither invocation is established to preserve global state.
- Neither invocation is established to retain or release any supplied resource.
- The first invocation is not established to leave the receiver valid for subsequent use.

The last point matters because the function nevertheless attempts a second dispatch after a normal first return. A dependency that invalidates the receiver without arranging for the later lookup makes the successful-execution preconditions fail.

### Ownership and allocation

No explicit allocation, deallocation, ownership transfer, reference-count adjustment, or resource acquisition is shown in this function.

Passing the receiver and opaque value to the dependencies does not establish whether either dependency borrows, retains, consumes, or releases anything. Those ownership semantics remain documented_only.

## Dependencies

**F-unresolved-1** is a contract-local opaque label for the unresolved indirect dependency reached through the dispatch entry at offset `0xA4`. It is not a recovered concrete function identity.

| Property | Contract |
|---|---|
| Resolution | Through the receiver’s initial dispatch-table reference |
| Entry location | Byte offset `0xA4` from the resolved table address |
| Invocation count | Two on a normally completing execution |
| Argument count | Three |
| Common arguments | Original receiver and original opaque 32-bit value |
| Varying argument | Bits 8–15 first; bits 0–7 second |
| Result handling | No dependency result is exposed or explicitly inspected |
| Concrete identity | Unknown; may differ between invocations |
| Effects and ownership | Unknown |

There is no established direct dependency beyond this unresolved dispatch mechanism. No assumption is made that a target reached on the first lookup has the same implementation or behavior as a target reached on the second lookup.

## Acceptance examples

These examples are conditional behavioral criteria, not reports of executed tests. Unless otherwise stated, they assume valid dispatch storage and dependencies that return normally.

### Distinct extracted values

For packed input `0x1234`, the first invocation receives third argument 18, and the second receives third argument 52.

Both invocations receive the original receiver pointer and the same opaque value.

### Upper bits ignored

Packed inputs `0x00001234` and `0xABCD1234` produce identical derived argument sequences: 18 followed by 52.

Their dependency effects are identical only if the relevant initial state and dependency behavior are otherwise identical. Equality of derived arguments alone does not guarantee equality of effects.

### Zero is still dispatched

For packed input zero, both invocations occur and both receive third argument zero. Neither invocation is omitted.

For packed input `0x0100`, the sequence is 1 followed by 0. The trailing zero still causes an invocation.

### No sign extension

For packed input `0x000080FF`, the sequence is 128 followed by 255, not negative signed-byte values.

For packed input `0xFFFFFFFF`, both derived quantities are 255.

### Dispatch mutation between calls

Suppose the receiver initially selects one valid table. During the first invocation, its dependency replaces the receiver’s table reference with a different valid table.

The second target must be selected through the updated reference at the same offset `0xA4`. Reusing the first target without a new lookup would not satisfy this contract.

The same principle applies if the first invocation changes the selected entry within the existing table.

### First invocation prevents continuation

If the first dependency does not return control normally, no second invocation is established. The function supplies no local recovery step that guarantees continuation.

## Uncertainties

The following remain documented_only:

- The receiver’s full layout, allocation size, lifecycle, and semantic type.
- The meaning and ownership of the opaque 32-bit argument.
- The concrete identities and full signatures of the indirect targets.
- The precise calling convention and machine-level argument placement.
- Whether the targets interpret the derived values as bytes, commands, characters, or another kind of quantity.
- Dependency return types and any dependency-specific failure conventions.
- Dependency effects on memory, globals, external resources, and ownership.
- Synchronization requirements and behavior under concurrent mutation.
- Environment-specific consequences of invalid pointers or callable entries.

Describing the operation as “high-byte-first” is established at the argument level. Describing it as a two-byte output operation is inferred intent only.

## Evidence

The supplied static representation establishes two sequential indirect invocations through the same receiver-relative dispatch rule. Their third arguments are selected from bits 8–15 and bits 0–7, respectively. It also establishes unchanged forwarding of the first two argument values, a repeated dispatch expression after the first invocation, and no explicit result value or local error handling.

The representation does not include the indirect targets’ implementations. Their effects, ownership rules, and preservation guarantees are therefore unknown.

No runtime tests have been performed. The acceptance examples above are derived contract criteria, not observed execution results.
