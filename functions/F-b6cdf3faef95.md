<!-- Trusted provenance: F-b6cdf3faef95; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b6cdf3faef95

## Purpose

F-b6cdf3faef95 dispatches an operation through a caller-supplied indirect function. Its choice of dispatch path depends on a byte in an operation descriptor.

Two behaviors are established:

- A descriptor discriminator equal to 10 selects a descriptor-owned callback. The function ignores that callback’s return value and subsequently returns a 32-bit field from the request object.
- Every other discriminator selects one of two request-owned callbacks. The function supplies either a temporary pointer slot or a slot within an auxiliary object, then returns the callback’s 32-bit result.

The semantic purpose of the objects and discriminator values is unknown. Descriptions such as “root,” “request,” “descriptor,” and “auxiliary object” below identify structural roles, not established application-level types.

## Interface

### Inputs and result

The function has four 32-bit input parameters and a 32-bit result.

| Input | Role established by observed accesses |
|---|---|
| First input | Address of a root object containing an auxiliary-object pointer and several fallback regions. |
| Second input | Address of a request object containing a result field and two indirect-call targets. |
| Third input | Address of an operation descriptor containing two selector bytes and an indirect-call target. |
| Fourth input | A 32-bit value copied into temporary local storage on the discriminator-10 path. It is not an explicit argument to either indirect call. |

The result is an uninterpreted 32-bit value. Its signedness, encoding, and relationship to success or failure are not established.

The supplied representation uses 32-bit pointer values and 32-bit word accesses. All offsets below are byte offsets. Byte comparisons concern exact values and do not depend on whether the underlying character type is signed.

### Required object layout

Only the following fields or addresses are relevant to this function:

| Object | Offset | Access or use |
|---|---:|---|
| Root | `0x20C` | Read a 32-bit auxiliary-object pointer. |
| Root | `0x84` | Form a fallback address; do not directly read the addressed data. |
| Root | `0xA4` | Form another fallback address; do not directly read the addressed data. |
| Root | `0x1C8` | Form another fallback address; do not directly read the addressed data. |
| Request | `0x0C` | Read a 32-bit result after the discriminator-10 callback returns. |
| Request | `0x40` | Read an indirect-call target on the ordinary selector path. |
| Request | `0x44` | Read an indirect-call target when the discriminator is 8 or 9. |
| Descriptor | `0x04` | Read a one-byte category on the non-10 path. |
| Descriptor | `0x05` | Read a one-byte dispatch discriminator. |
| Descriptor | `0x08` | Read an indirect-call target on the discriminator-10 path. |
| Auxiliary object | `0x00` | Read a 32-bit word for categories 2, 3, and 4 when the auxiliary pointer is nonzero. |
| Auxiliary object | `0x90` | Form the address of a slot for category 2. |
| Auxiliary object | `0xD4` | Form the address of a slot for category 3. |
| Auxiliary object | `0x11C` | Form the address of a slot for category 4. |

The three auxiliary slot offsets are byte offsets derived from 32-bit-word indexing: 36 words, 53 words, and 71 words respectively. They are not offsets of 36, 53, or 71 bytes.

No minimum size for any complete object can be inferred independently of the chosen path and the behavior of its callback. For example, this function forms an auxiliary slot address but does not itself read the slot’s contents.

### Preconditions

For a defined execution:

- The root’s auxiliary-pointer field and the descriptor’s discriminator byte must be readable.
- Every additional field reached by the selected path must be readable.
- The selected indirect-call target must be executable and compatible with the five-argument invocation described below.
- Objects used after a callback returns must remain valid for those later accesses.
- Address calculations must yield valid addresses for the target environment.

The function performs no visible null checks on the root, request, or descriptor addresses. A zero auxiliary pointer is explicitly supported on the non-10 path. On the discriminator-10 path, that pointer is simply forwarded and may also be zero.

No validation, recovery, or explicit error return is established for invalid pointers, unsuitable call targets, or callback failures.

## Behavior

### Initial read and dispatch decision

The function first reads the auxiliary-object pointer from root offset `0x20C`. This read occurs even when the descriptor later selects the discriminator-10 path or an unrecognized category.

It then reads the byte at descriptor offset `0x05`.

The initially loaded auxiliary pointer is retained for use by the selected path. The function does not reload that pointer field later.

### Discriminator equal to 10

When the initial discriminator byte is 10:

1. The fourth input is copied into temporary local storage.
2. The descriptor-owned indirect target is obtained from descriptor offset `0x08`.
3. That target is invoked with five arguments, in this positional order:

   | Position | Value |
   |---:|---|
   | 1 | Root address |
   | 2 | Request address |
   | 3 | Function-pointer value read from descriptor offset `0x08` |
   | 4 | Previously loaded auxiliary-object pointer |
   | 5 | Descriptor address |

4. After the callback returns normally, the function reads the 32-bit word at request offset `0x0C`.
5. That word becomes the function’s result.

The callback’s own return value does not determine the outer result.

The descriptor’s function-pointer field appears both as the call-target source and as the source of the third argument. The supplied representation does not establish a separate evaluation order between those two uses or a reliable number of machine-level loads. With a stable descriptor field, the third argument is the same function-pointer value as the selected target.

The post-callback timing of the request-result read is significant: if the callback changes request offset `0x0C` and leaves it valid, the changed value is returned.

This branch does not read the descriptor’s category byte, select a request-owned callback, or read the auxiliary object’s first word.

The temporary copy of the fourth input is not explicitly passed by value or by address. No ordinary interface-level dependency on that input is established beyond this local assignment.

### Every other discriminator

When the initial discriminator is not 10, the function reads the category byte at descriptor offset `0x04`.

It prepares two callback arguments:

- A slot address.
- A 32-bit context word.

The selection is:

| Category | Auxiliary pointer | Slot address supplied to callback | Context word |
|---:|---|---|---|
| 2 | Zero | Address of a temporary local slot containing root address plus `0x84` | Zero |
| 2 | Nonzero | Auxiliary address plus `0x90` | Word read from auxiliary offset `0x00` |
| 3 | Zero | Address of a temporary local slot containing root address plus `0xA4` | Zero |
| 3 | Nonzero | Auxiliary address plus `0xD4` | Word read from auxiliary offset `0x00` |
| 4 | Zero | Address of a temporary local slot containing root address plus `0x1C8` | Zero |
| 4 | Nonzero | Auxiliary address plus `0x11C` | Word read from auxiliary offset `0x00` |
| Any other byte value | Either | Address of a temporary local slot containing root address plus `0x84` | Zero |

For categories 2, 3, and 4 with a nonzero auxiliary pointer, the function reads the auxiliary object’s first 32-bit word before selecting the request-owned callback.

For all other categories, a nonzero auxiliary pointer does not change the slot or context-word selection. In particular, the function does not read that auxiliary object’s first word on these category paths.

The fallback argument is the **address of a temporary slot containing an address**. It is not the fallback root-region address itself. Likewise, the auxiliary argument is the address of the selected slot, not its loaded contents.

The function then consults the discriminator byte at descriptor offset `0x05` again:

- A value of 8 selects the target at request offset `0x44`.
- Otherwise, a comparison with 9 determines whether to select request offset `0x44`.
- Other values select request offset `0x40`.

The represented condition short-circuits after recognizing 8. It therefore does not require the second comparison in that case. The supplied material does not establish exact physical load counts under compilation or concurrent mutation.

The selected request-owned target is invoked with:

| Position | Value |
|---:|---|
| 1 | Request address |
| 2 | Descriptor address |
| 3 | Selected slot address |
| 4 | Selected 32-bit context word |
| 5 | Zero |

Its 32-bit result is returned directly. This path does not subsequently read request offset `0x0C`.

### Stability and arithmetic boundaries

The initial discriminator test and the later callback-selection tests are separate observations in the supplied representation. If descriptor contents can change between them, one immutable “operation type” cannot safely be assumed. No synchronization or atomic snapshot is established.

All address offsets are fixed byte additions. No bounds checks or overflow checks are visible. The contract requires valid resulting addresses rather than assigning portable behavior to address wraparound, invalid pointer arithmetic, or inaccessible memory.

## Side effects

The function’s visible direct writes are confined to temporary local storage. It does not directly assign to fields in the root, request, descriptor, or auxiliary objects.

That does **not** establish that those objects remain unchanged:

- The discriminator-10 callback receives the root, request, descriptor, and auxiliary addresses.
- The other callbacks receive the request, descriptor, and selected slot address.
- Any callback may also affect independently accessible state.

The callbacks’ mutation, allocation, release, I/O, reentrancy, and global-state behavior are unknown. The function supplies no established ownership-transfer or preservation guarantee.

For an auxiliary slot, a callback write through the third argument would target storage within the auxiliary object. For a fallback slot, a write through that argument would target the caller function’s temporary local slot. The function does not explicitly consume that slot’s contents after the callback returns.

The temporary slot’s lifetime ends when F-b6cdf3faef95 returns. No persistent use of its address is supported by the observed implementation. Whether any callback retains it is unknown, not evidence of a valid longer lifetime.

If a callback does not return normally, no normal-return result is established.

## Dependencies

No concrete callee identity is supplied for the indirect targets. The following opaque IDs are document-local dependency designators, not claims of globally assigned identities:

| Opaque ID | Resolution | Invocation contract used here |
|---|---|---|
| F-indirect-01 | Descriptor offset `0x08` | Five arguments: root, request, descriptor-owned function-pointer value, auxiliary pointer, descriptor. Its return value is ignored. |
| F-indirect-02 | Request offset `0x40` | Five arguments: request, descriptor, slot address, context word, zero. Its 32-bit result is forwarded. |
| F-indirect-03 | Request offset `0x44` | Same argument arrangement and result forwarding as F-indirect-02. |

These are dispatch roles. Two or all three fields may contain the same target address; distinct identities are not established.

No assumptions are made that these dependencies preserve global state, preserve the objects passed to them, or own or release any supplied storage.

## Acceptance examples

These examples are conditional consequences of the static behavior. They are not runtime test results.

### Descriptor-owned dispatch returns the post-call request field

Given a readable root and descriptor, discriminator 10, and a valid F-indirect-01 target:

- The callback receives the five arguments specified for that branch.
- If it returns a value of `0x11223344` but leaves request offset `0x0C` containing `0xAABBCCDD`, the outer result is `0xAABBCCDD`.
- If it changes that request field before returning, the outer result comes from the changed field.

Acceptance does not depend on the callback’s return value.

### Null auxiliary pointer preserves the extra indirection

Given category 3, a zero auxiliary pointer, discriminator 8, and a valid F-indirect-03:

- The third callback argument addresses a temporary 32-bit slot.
- That slot contains root address plus `0xA4`.
- The fourth and fifth callback arguments are zero.
- The result is the 32-bit value returned by F-indirect-03.

Passing root address plus `0xA4` directly as the third argument would not match this behavior.

### Nonzero auxiliary pointer selects an embedded slot

Given category 4, a nonzero valid auxiliary pointer, and a stable discriminator other than 8, 9, or 10:

- F-indirect-02 is selected.
- Its third argument is auxiliary address plus `0x11C`.
- Its fourth argument is the word read from auxiliary offset `0x00`.
- The function need not read the contents at auxiliary offset `0x11C` itself.

### Unrecognized category ignores auxiliary contents

Given category 255, a nonzero auxiliary pointer, and discriminator 9:

- F-indirect-03 is selected.
- Its slot argument addresses a temporary slot containing root address plus `0x84`.
- Its context word is zero.
- No dereference of the auxiliary object’s first word is required by this path.

The root’s auxiliary-pointer field is still read initially.

### Category does not affect discriminator-10 dispatch

Given discriminator 10, changing only the category byte does not change the established branch behavior, because that byte is not read on this path.

### Fourth input has no explicit callback-argument role

For otherwise identical inputs and stable memory, changing only the fourth input does not change any explicitly supplied callback argument or return-value selection. This statement does not assert identical behavior for callbacks that inspect undocumented execution state.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of discriminator values 8, 9, and 10.
- Meanings of categories 2, 3, and 4.
- Semantic types and required capacities of fallback regions and auxiliary slots.
- Meaning of the auxiliary object’s first word.
- Actual callback implementations and their ownership, mutation, and error conventions.
- Whether the temporary fourth-input assignment has significance outside the represented explicit interface.
- Exact instruction-level read counts, alignment requirements, byte order, and calling-convention details beyond the represented 32-bit interface.
- Behavior under concurrent modification, invalid addresses, incompatible callbacks, or callbacks that invalidate storage needed afterward.

Dispatch or operation-handling intent is an inference from the structural pattern. It is not an established domain classification.

## Evidence

The supplied static representation establishes the field offsets, access widths, branch conditions, callback argument positions, temporary-slot indirection, and return-value selection described above.

The contract distinguishes address formation from memory reads and distinguishes direct writes from possible callback effects. It does not derive callback guarantees from their invocation.

No runtime tests, execution traces, or independent implementation observations have been performed. Acceptance examples describe conditional obligations supported by static evidence, not empirically verified outcomes.
