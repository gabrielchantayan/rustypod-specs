<!-- Trusted provenance: F-4e6cefe281de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e6cefe281de

## Purpose

F-4e6cefe281de performs a dependency-mediated selection and, when the second dependency reports zero, copies one 32-bit value from the selected record to caller-provided output storage.

The function returns a Boolean indicating whether the second dependency returned zero. It does not return the copied value directly.

The established behavior is narrower than a general lookup contract: the meanings of the input object, the supplied 32-bit value, the selected record, and the dependency’s result codes are not known. Describing them as a container, key, entry, or error code would be inferred intent rather than an established interface.

## Interface

### Inputs

The function has three arguments:

| Argument | Established representation | Role |
|---|---|---|
| Object address | A 32-bit integer used as a memory address | Passed to F-a6e1ba6965d3; later used to read a 32-bit field at byte offset `0x10` |
| Supplied value | An uninterpreted 32-bit value | Copied into local storage whose address is passed to F-a6e1ba6965d3 |
| Output address | Pointer to a 32-bit storage unit | Receives a 32-bit value only on the zero-result path |

The supplied value is a by-value argument. The first dependency receives an address to its local copy, not an explicit pointer to the caller’s original storage. The dependency can therefore modify that local copy.

The object address is represented as a 32-bit integer in the supplied material. Its use as an address supports a 32-bit-address interpretation, but the complete target ABI, address-conversion rules, and pointer representation are not independently established.

### Return value

The logical result is:

- **True** when F-c7e5bb3c4e39 returns a 32-bit integer equal to zero, provided the subsequent source read and output write complete.
- **False** when F-c7e5bb3c4e39 returns any nonzero 32-bit integer.

Both positive and negative nonzero results produce false. No distinction among those values is exposed through the Boolean result.

The physical width and calling-convention representation of the Boolean return are not established.

### Required memory

For ordinary completion, the following accesses must be valid when reached:

- A 32-bit read at the object address plus `0x10`.
- Storage and initialization sufficient for the two dependencies to operate on the local state passed to them.
- On the zero-result path, a 32-bit read at the selected address plus `0x14`.
- On that same path, a 32-bit write at the output address.

The function contains no explicit null, alignment, range, or access-permission checks. Exact alignment requirements and invalid-access behavior depend on the execution environment and are documented_only.

## Behavior

### Local state and first dependency

The function establishes local storage for:

- A 32-bit address-bearing state value.
- A 32-bit copy of the supplied value.
- A separate 32-bit field value used by the second dependency.

No initial value for the address-bearing state is established by this function before the first dependency receives its address. Valid use of that state therefore depends on the dependencies’ contracts. In particular, the wrapper itself does not supply a fallback selected address.

The first dependency, F-a6e1ba6965d3, receives these arguments in order:

1. The address of the local address-bearing state.
2. The object address.
3. The address of the local copy of the supplied value.

Any return value from this dependency is not inspected. There is no wrapper-level success test, retry, or early return based on that call.

The addresses supplied to the dependency allow it to modify the corresponding local storage. Its actual read/write requirements, other side effects, and completion behavior are unknown.

### Object-field observation

After F-a6e1ba6965d3 returns normally, the function reads exactly one 32-bit value from byte offset `0x10` relative to the object address and places it in the separate local field storage.

This ordering matters. The field is not captured before the first dependency runs. If that dependency changes the object field, the wrapper’s subsequent read observes the value then present, subject to the execution environment’s memory rules.

The supplied material establishes this object-relative access only:

| Byte offset | Access width | Wrapper use |
|---|---:|---|
| `0x10` | 32 bits | Read after the first dependency returns; pass a local copy by address to the second dependency |

This does not establish an entire object layout or the meaning of that field.

### Second dependency and result classification

The second dependency, F-c7e5bb3c4e39, receives:

1. The address of the same local address-bearing state used by the first dependency.
2. The address of the local copy of the object field read at offset `0x10`.

The second dependency therefore receives two addresses to local storage, not merely their current 32-bit contents. The wrapper does not enforce read-only access to either local.

The second dependency returns a 32-bit integer. The wrapper classifies that result solely by equality with zero:

- Zero selects the output-copy path.
- Every nonzero value skips the wrapper’s output read and write.

No additional interpretation of the result is established.

### Zero-result output path

If the second dependency returns zero, the wrapper uses the address-bearing state as it exists **after that dependency returns**.

It reads one 32-bit value at byte offset `0x14` from that selected address, then writes that value to the output address. The selected address need not be the value originally produced by the first dependency: the second dependency received the state by address and may have changed it.

The selected-record access established by the wrapper is:

| Byte offset | Access width | Wrapper use |
|---|---:|---|
| `0x14` | 32 bits | Read only after a zero result; copy to caller-provided output storage |

The source read logically precedes the output store. Consequently, if valid source and destination storage overlap, the transferred value is the value obtained by that source read before the wrapper performs its output write. Exact behavior for unaligned or otherwise invalid overlap is not established.

After the write completes, the function returns true.

### Nonzero-result path

If the second dependency returns nonzero, the wrapper does not:

- Read the selected-record field at offset `0x14`.
- Dereference the output address.
- Store a default or failure value into the output.

It returns false.

This absence of a direct wrapper write is not a guarantee that output memory remains unchanged across the entire call. Either dependency could have access to that memory through globals, aliases, or other mechanisms not shown.

### Arithmetic and widths

The two object-relative offsets are byte offsets: `0x10` is 16 bytes, and `0x14` is 20 bytes. They are not scaled indices into an array of 32-bit elements.

Both transferred fields are 32 bits wide. The wrapper does not visibly narrow, widen, sign-extend, or interpret their contents. Signedness is relevant to neither the field copies nor the zero/nonzero classification.

Address formation involves values represented as 32-bit integers. There is no visible overflow check. A portable wraparound rule, valid near-boundary addressing, and the behavior of an overflowed address calculation are not established. Ordinary use requires the reached field addresses to be valid under the target’s addressing rules.

## Side effects

### Directly established effects

In wrapper-level order, normal execution performs:

1. A call to F-a6e1ba6965d3.
2. A 32-bit read from the object field at offset `0x10`.
3. A call to F-c7e5bb3c4e39.
4. Only for a zero result, a 32-bit selected-field read followed by a 32-bit output write.

The wrapper itself shows no allocation, deallocation, resource acquisition, logging, or explicit global-state access.

The local field passed to the second dependency is a copy of the object field. Modification through that local pointer does not, by itself, write back to the object field.

### Ownership and lifetime

No ownership transfer is explicitly established for the object, the supplied value, the selected record, or output storage. The final transfer copies a 32-bit value; it does not establish duplication or transfer of any resource that value might represent.

The dependencies receive addresses into the wrapper’s local storage. Those addresses do not provide storage whose lifetime extends beyond the wrapper’s invocation. Whether either dependency retains such an address is unknown; retention must not be assumed safe.

The absence of explicit allocation or release in the wrapper does not establish that the dependencies neither allocate nor release memory. Their ownership effects remain documented_only.

### Errors and abnormal completion

The only explicit result classification is zero versus nonzero from the second dependency. The wrapper exposes no separate status for an invalid object, invalid selected address, invalid output address, or dependency failure outside that classification.

If the zero-result path encounters an invalid source read or output write, the material does not establish a normal Boolean return. A zero dependency result alone is therefore insufficient to promise that the caller receives true.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-a6e1ba6965d3 | Called first with an address to local state, the object address, and an address to a local 32-bit supplied-value copy; its return is not inspected | State initialization, interpretation of inputs, additional reads/writes, ownership effects, and failure behavior |
| F-c7e5bb3c4e39 | Called second with addresses to local state and the captured object-field copy; its 32-bit return determines the branch | Meaning of result codes, permitted state changes, selected-record validity guarantees, and other side effects |

Neither dependency is established to preserve globals, preserve unrelated memory, or leave the passed local values unchanged.

## Acceptance examples

These are conditional behavioral examples derived from static evidence, not executed tests.

### Zero result copies the selected field

Given that both dependencies complete normally, the second returns zero, and the final selected address has a readable 32-bit value `0x89ABCDEF` at offset `0x14`:

- The wrapper writes `0x89ABCDEF` to valid output storage.
- It returns true after the store completes.

The high bit being set does not change the copy or branch decision.

### Nonzero result suppresses the wrapper’s output access

Given that the second dependency returns `7`:

- The wrapper does not read the selected field.
- The wrapper does not dereference or write the output address.
- It returns false.

The same wrapper behavior applies to a return of `-1`. Any dependency-originated memory changes remain outside this example’s guarantee.

### A first-dependency object change is observed

Given that the first dependency changes the object’s field at offset `0x10` from `0x00000010` to `0x00000020` before returning:

- The wrapper subsequently captures `0x00000020`.
- The second dependency receives the address of that captured local value.

The earlier field value is not preserved by this wrapper.

### A second-dependency selection change is honored

Given that the first dependency establishes selected address A, and the second changes the local selected address to B while returning zero:

- The wrapper reads the 32-bit field at B plus `0x14`.
- It does not use A for that final field read.

Both the changed state and the zero result are relevant.

### Invalid output matters only when reached

If the second dependency returns nonzero, an invalid output address is not directly dereferenced by the wrapper.

If it returns zero, the wrapper attempts the output write after the selected-field read. The contract does not promise successful completion or a Boolean return for an invalid output destination.

## Uncertainties

The following remain documented_only:

- The semantic identities of the object, supplied value, selected record, and copied field.
- Whether the first dependency initializes all state subsequently read.
- The meanings of individual nonzero results from the second dependency.
- The complete layouts and minimum extents of participating objects.
- Target endianness, alignment requirements, Boolean ABI, and precise integer-to-address semantics.
- Dependency side effects, ownership behavior, retained pointers, and abnormal completion behavior.
- Thread safety, reentrancy, synchronization, and effects of concurrent mutation.

**Inferred intent:** the sequence is consistent with establishing a candidate, checking or advancing it, and returning a field on success. That interpretation is not required by the established contract.

## Evidence

This contract is based solely on the supplied static function body. The established observations are the two ordered dependency calls, the intervening 32-bit object-field read, the zero/nonzero decision, and the conditional 32-bit source-to-output transfer.

Dependency implementations, external interface definitions, and runtime observations were not provided. No runtime tests have been performed.
