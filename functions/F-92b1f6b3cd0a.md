<!-- Trusted provenance: F-92b1f6b3cd0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92b1f6b3cd0a

## Purpose

F-92b1f6b3cd0a performs two ordered attribute lookups, checks the low byte of each resulting value, and conditionally writes two caller-provided output slots.

The lookup keys are `CHARSET_REGISTRY` and `CHARSET_ENCODING`, in that order. Acceptance requires both lookups to return zero and both inspected low bytes to equal one. On acceptance, the function writes the supplied payload to the first output slot and the saved address of that first slot to the second output slot.

The attribute names suggest charset-related intent. The supplied body does not establish what the value one means, what object the lookup target represents, or why the output values have this arrangement. Those interpretations are not part of the established contract.

## Interface

The interface has four inputs and one return value:

| Input | Role | Representation and use |
|---|---|---|
| Lookup target | First argument to both dependency calls | A 32-bit unsigned value, forwarded unchanged |
| First output address | Destination of the first accepted-result write | Address of a four-byte writable slot; its address value is also saved for the second write |
| Second output address | Destination of the second accepted-result write | Address of a four-byte writable slot; its address value also initializes the second lookup’s temporary storage |
| Payload | Value written to the first output slot on acceptance | An uninterpreted four-byte value |

The return value is a 32-bit integer status:

- Zero means both dependency calls returned zero and both low-byte checks passed, assuming execution completed normally.
- A nonzero dependency status is propagated from the first failing lookup.
- The function itself produces status six when both lookups return zero but either low-byte check fails.

A returned six does not uniquely identify a low-byte mismatch: a dependency may itself return six.

### Output layout

Each successful-path output store is four bytes wide. The first receives the payload. The second receives the saved first output address, using the pointer representation carried by the supplied body’s four-byte store.

The supplied material does not independently specify the complete ABI, pointer model, byte order, or alignment rules. It does establish four-byte output slots and a pointer-valued second output. A binding must preserve that representation rather than substitute a wider pointer store.

### Preconditions

For the accepted-result path to complete normally:

- Both output addresses must permit their respective four-byte writes.
- Their alignment and accessibility must satisfy the execution environment.
- Any overlap between them must be valid for both ordered stores.
- The lookup target and dependency arguments must satisfy F-3147708f5aba’s requirements, which are not supplied.

There is no explicit null-address, bounds, alignment, target-validity, or payload-validity check.

The enclosing function does not directly dereference the caller’s output addresses before acceptance. Consequently, an early-return path can avoid those direct stores even if the addresses would not support them. This does not establish that arbitrary addresses are safe: the dependency’s behavior and external effects are unknown.

## Behavior

### Lookup preparation

Before either lookup, the function establishes temporary state containing:

- The lookup target’s 32-bit value, in storage offered to the first lookup.
- A saved copy of the first output address.
- The second output address, in storage offered to the second lookup.
- A saved copy of the four-byte payload.

The lookup-output temporaries are therefore initialized values, not guaranteed zero-filled storage. If a dependency reports success without fully replacing its supplied temporary, the remaining initialized bits can affect the subsequent checks.

The two output addresses retain distinct roles. Replacing the temporary initialized from the second output address does not redirect the eventual second output store; that store still uses the original second output address.

### First lookup

The function calls F-3147708f5aba with:

1. The original lookup target.
2. The key `CHARSET_REGISTRY`.
3. The address of the temporary initialized with the lookup target.

If this call returns a nonzero status, the function returns that status immediately.

On this path:

- There is no `CHARSET_ENCODING` lookup.
- Neither low-byte check is performed.
- Neither caller-output store is performed directly by this function.

A zero status permits the second lookup. It does not yet cause the registry value to be checked.

### Second lookup

The function next calls F-3147708f5aba with:

1. The original lookup target, not the first lookup’s returned temporary value.
2. The key `CHARSET_ENCODING`.
3. The address of the temporary initialized with the second output address.

This call occurs whenever the first lookup returns zero, even if the first lookup’s result will subsequently fail the registry low-byte check.

If the second lookup returns a nonzero status, that status is returned immediately. The function does not perform either low-byte check or either direct caller-output store on this path.

### Low-byte checks

Only after both dependency calls return zero does the function examine the lookup results.

The registry temporary is reduced to its least significant eight bits. Those bits must represent the numeric value one.

If the registry check passes, the encoding temporary’s value is converted to a 32-bit unsigned representation and reduced to its least significant eight bits. Those bits must also represent one.

Consequences include:

- A result need not equal one as a complete word.
- Upper bits do not cause rejection when the low byte is one.
- The encoding low-byte reduction and comparison are skipped when the registry low byte is not one.
- Skipping the encoding check does not skip the encoding lookup; that call has already occurred.
- There is no signed range check, string comparison, or full-word equality requirement.

The mask is numeric. It identifies the least significant eight bits, not a separately specified byte position in memory.

If either required comparison fails, the function returns six without directly storing to either caller-output slot.

### Accepted-result writes

When both low-byte checks pass, the function performs these stores in order:

1. Write the saved four-byte payload through the original first output address.
2. Write the saved first output address through the original second output address.

The second store writes an address value. It does not copy the current contents of the first output slot.

The function then returns zero, retaining the successful status from the second dependency call.

No further lookup, allocation, release, or explicit rollback is shown.

### Aliasing and ordering

The two stores are ordered, not transactional.

If both output addresses identify the same four-byte slot, the payload is written first and then overwritten by the saved first output address. The final slot value is therefore that address representation, assuming both stores complete normally.

If the slots partially overlap and the environment permits such accesses, the second store overwrites the overlapping bytes from the first store. The resulting byte pattern depends on the addresses and byte order.

If the first store completes but the second store faults, no rollback is shown. The function does not establish an error-return convention for memory faults.

## Side effects

The established direct external effects are:

- One dependency call on every normally entered execution.
- A second dependency call only when the first returns zero.
- Two ordered four-byte caller-memory writes only when both dependency statuses are zero and both low-byte checks pass.

The supplied body contains no explicit allocation, deallocation, ownership-transfer operation, global assignment, or I/O operation. These observations do not constrain effects performed inside F-3147708f5aba.

In particular, a nonzero return does **not** establish that caller-visible state is unchanged. A dependency could have changed state before returning. The narrower guarantee is that this function does not execute its two explicit output stores on a rejection or dependency-error path.

The final output stores occur after both dependency calls, so they can overwrite earlier dependency effects on overlapping caller memory.

### Ownership and lifetime

The output slots are used as caller-supplied storage. This function does not explicitly release either slot or the payload.

The address written to the second slot is the caller’s first output address, not the address of an internal temporary. That write alone does not establish ownership transfer, reference counting, or permission to release the referenced storage.

Both dependency calls receive addresses of temporary storage belonging to this invocation. Whether F-3147708f5aba retains such addresses is unknown. The supplied body provides no mechanism to extend their lifetime beyond the invocation.

## Dependencies

### F-3147708f5aba

The function uses this dependency twice, conditionally and in a fixed order.

| Call | Target argument | Key | Supplied output storage initially contains |
|---|---|---|---|
| First | Original lookup target | `CHARSET_REGISTRY` | Original lookup target |
| Second | Original lookup target | `CHARSET_ENCODING` | Original second output address |

Established behavior at the call boundary:

- Zero is treated as lookup success.
- Any nonzero return stops further processing and is returned unchanged.
- The first call can modify the temporary later used for the registry check.
- The second call can modify the temporary later used for the encoding check.
- The second call is omitted after a first-call failure.

The dependency’s output-write width, completeness of writes, interpretation of keys, and status meanings are not supplied. A successful status must not be assumed to imply a complete output write without its own contract.

Likewise, nothing here establishes that the dependency preserves globals, avoids caller-memory changes, allocates or releases nothing, or refrains from retaining pointers.

## Acceptance examples

These examples are conditional expectations derived from the supplied body, not runtime observations. Unless stated otherwise, both dependency calls return zero, their supplied output storage contains the stated values afterward, memory accesses are valid, and there are no additional dependency effects.

### Both low bytes equal one

Suppose the registry result is `0x00000001`, the encoding result is `0x00000001`, and the payload is `0x12345678`.

Expected behavior:

- Both lookups occur in the specified order.
- The first output slot receives `0x12345678`.
- The second output slot receives the saved first output address.
- The return status is zero.

### Upper bits are nonzero

Suppose the registry result is `0xABCDEF01` and the encoding result’s 32-bit representation is `0x76543201`.

Both values pass because their low bytes equal one. The upper bits do not prevent the two output writes or the zero return.

### Registry mismatch

Suppose the registry result is `0x00000100` and the encoding result is `0x00000001`.

Both lookups still occur. The registry low byte is zero, so the function returns six without performing its two explicit output stores.

### Encoding mismatch

Suppose the registry result is `0x00000001` and the encoding result’s 32-bit representation is `0x00000102`.

The registry check passes, but the encoding low byte is two. The function returns six without performing its two explicit output stores.

### First lookup failure

Suppose the first lookup returns nine.

The function returns nine. It does not invoke the second lookup, inspect either result’s low byte, or execute its explicit output stores.

### Second lookup failure takes precedence over mismatch

Suppose the first lookup returns zero and leaves a registry result whose low byte is not one. The second lookup returns eleven.

The function returns eleven, not six. The registry check occurs only after a successful second lookup.

### Identical output addresses

Suppose both output addresses identify the same valid four-byte slot and both checks pass.

The slot receives the payload first, followed by its own saved address representation. The final value is that address representation, and the function returns zero.

### Success without replacement

Suppose both dependency calls return zero without modifying either supplied temporary.

The registry check then uses the low byte of the original lookup target. If that passes, the encoding check uses the low byte of the original second output address’s 32-bit representation.

Acceptance depends on both of those bytes being one. This illustrates the effect of initialized temporary storage; it does not establish that the real dependency permits successful calls without writes.

## Uncertainties

The following matters are **documented_only**:

- **Attribute semantics:** The meaning of one for either key is unknown.
- **Complete ABI:** Pointer width, calling convention, alignment, and byte order are not independently specified.
- **Dependency writes:** The width and completeness of writes through lookup-output addresses are unknown.
- **Dependency side effects:** Global changes, caller-memory changes, pointer retention, allocation, and release behavior are unknown.
- **Status vocabulary:** No dependency-specific interpretation of nonzero statuses is supplied.
- **Fault behavior:** Invalid-memory faults, exceptions, signals, and recovery behavior are outside the shown return-status handling.
- **Concurrency and reentrancy:** No synchronization or shared-state guarantees are established.
- **Ownership interpretation:** The address-valued output does not establish a higher-level ownership protocol.

The charset-related interpretation is inferred intent. The ordered calls, status propagation, low-byte conditions, and output-store sequence are established static behavior.

## Evidence

This contract is based solely on static examination of the supplied function body and dependency mapping.

The supplied body establishes:

- Initial values for both lookup temporaries.
- The exact key order and reuse of the original lookup target.
- Short-circuit propagation of nonzero dependency statuses.
- Deferral of both low-byte checks until both lookups succeed.
- Eight-bit masking and comparison against one.
- Locally generated status six for a failed value check.
- Two ordered four-byte output stores on acceptance.
- A zero return after those stores complete normally.

No dependency implementation, complete ABI specification, or runtime evidence was supplied. No runtime tests have been performed.
