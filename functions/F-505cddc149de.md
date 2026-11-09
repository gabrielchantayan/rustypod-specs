<!-- Trusted provenance: F-505cddc149de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-505cddc149de

## Purpose

F-505cddc149de conditionally processes an item associated with a selector and a supplied subject. Its visible behavior comprises an initial subject callback, an optional completion gate, item lookup, position preparation, a four-word copy, object creation/configuration calls, registration calls, and a final completion-byte write.

Selectors 0, 1, 2, and 4 each have an associated completion byte. A nonzero byte suppresses processing **after** the initial callback. Other selectors bypass that gate and receive no direct completion-byte write.

“Position,” “creation,” “configuration,” and “registration” describe roles suggested by the dataflow. Their subsystem meanings are **inferred intent**, not independently established guarantees.

## Interface

### Inputs and output

The supplied representation has five arguments, each occupying a 32-bit slot:

| Input | Established use |
|---|---|
| Context | Address containing completion bytes and an embedded target used by a later dependency. |
| Subject | Address of an object whose first word supplies a dispatch-table address. |
| Selector | 32-bit integer passed unchanged to lookup and compared with 0, 1, 2, and 4. |
| Displacement control | 32-bit integer tested for equality to zero when choosing the displacement value. |
| Mode control | 32-bit integer tested for equality to zero when choosing a configuration value. |

Neither control is restricted to Boolean encodings: every nonzero value takes its respective nonzero branch.

No explicit return value or separate success indicator is established. Normal return occurs after a completion-gate rejection, a zero lookup result, or the full processing sequence. There is no visible error-reporting interface.

Pointer-bearing values and handles occupy 32-bit slots in this representation. The exact calling convention is **documented_only**.

### Accessible object layouts

Offsets are byte offsets and identify only the fields accessed here.

| Object | Offset | Access |
|---|---|---|
| Context | `+0x58` | Address of an embedded target passed onward; no direct read of its contents here. |
| Context | `+0x98` | One-byte completion state for selector 0. |
| Context | `+0x99` | One-byte completion state for selector 1. |
| Context | `+0x9a` | One-byte completion state for selector 2. |
| Context | `+0x9b` | One-byte completion state for selector 4. |
| Subject | `+0x00` | Read a 32-bit dispatch-table address for each dispatched call. |
| Subject dispatch table | `+0xa0` | Initial method entry. |
| Subject dispatch table | `+0x5c` | Later method entry returning a source object. |
| F-38aa1bb9efd0 result | `+0xe8` | Read one 32-bit lookup argument. |
| Nonzero lookup result | `+0x30`, `+0x34` | Read two 32-bit position components. |
| Nonzero lookup result | `+0xc4`, `+0xc8`, `+0xcc`, `+0xd0` | Write four 32-bit words. |
| Later subject-method result | `+0x80`, `+0x84`, `+0x88`, `+0x8c` | Read four 32-bit source words. |
| F-2698159acdb1 result | `+0x04` | Read one 32-bit handle. |

Three separate static storage locations also supply 32-bit values:

- **Displacement static:** read during position preparation, unconditionally once a nonzero item has been obtained.
- **Creation-argument static:** read for the fourth argument to F-2698159acdb1.
- **Finalization-argument static:** read for the third argument to F-aa408aa1a280.

Their numeric contents are unavailable. They are separate reads at separate stages, not values established as captured together at entry.

### Preconditions and failure boundaries

The function performs no visible validation of context, subject, dispatch tables, or method entries. Each must support its reached accesses and calls.

Only the lookup result receives an explicit zero check. There is no zero check before dereferencing the result of F-38aa1bb9efd0, the later subject method, or F-2698159acdb1. The result of F-8157a180d901 is passed onward without a local zero check.

The static locations must also be readable when reached. In particular, zero displacement control does **not** eliminate the displacement-static read.

Invalid accesses, dependency failures, and nonlocal exits have no locally specified recovery behavior.

## Behavior

### Initial callback and completion gate

The function first obtains the subject’s dispatch-table address and invokes the entry at `+0xa0` with the subject and the 32-bit value 1. No completion byte is inspected before that call.

After the callback returns, the selector determines the gate:

| Selector | Byte inspected |
|---|---|
| 0 | Context `+0x98` |
| 1 | Context `+0x99` |
| 2 | Context `+0x9a` |
| 4 | Context `+0x9b` |
| Any other value | None |

A recognized selector returns immediately if its byte is nonzero; the byte need not equal 1. An unrecognized selector proceeds without reading any of these bytes for the gate.

The gate observes memory after the initial callback. If that callback changes the relevant byte, its changed value governs the decision.

### Service acquisition and item lookup

If the gate permits processing:

1. F-7a89f2f7fd9a is called without arguments. Its 32-bit result is retained for two later calls.
2. F-38aa1bb9efd0 is called without arguments.
3. The 32-bit word at `+0xe8` of its result is read.
4. F-db754b1ab3c7 receives that word and the selector.

A zero lookup result causes normal return. This path has already executed the initial subject callback and the two preceding dependency calls, but does not reach position reads, static displacement access, copying, creation, or the direct completion write.

### Position capture and unconditional displacement-static read

For a nonzero item, the established read order is:

1. Read the item’s 32-bit word at `+0x30`.
2. Read the item’s 32-bit word at `+0x34`.
3. Read the 32-bit displacement static into temporary storage.
4. Test displacement control for equality to zero.

The third operation occurs on **both** control paths. Its execution is distinct from whether its value contributes to the addition:

| Displacement control | Static read occurs? | Value used as displacement |
|---|---|---|
| Zero | Yes | `0x140`, decimal 320, replacing the captured static value |
| Nonzero | Yes | The value captured from the static |

There is no dependency call between these position reads and the displacement-static read. The static read is not deferred until a nonzero-control branch.

The function prepares two distinct contiguous two-word temporaries:

- An adjusted pair containing the captured first component plus the selected displacement, followed by the captured second component.
- An original-position pair containing the captured first and second components.

F-a19ace379809 receives the item and the address of the adjusted pair.

The addition has a 32-bit stored result. No saturation, bounds check, or explicit overflow branch is visible. The supplied representation does not establish a portable signed-overflow rule for a mathematical result outside the signed 32-bit range; such cases remain **documented_only**, rather than being assigned a guessed wraparound policy.

The call establishes the supplied pair, not that the dependency necessarily updates the item’s position or consumes the pair synchronously. The original-position pair is prepared before this call and passed to a later dependency; no unknown callee is assumed unable to alter reachable memory.

### Subject-derived four-word copy

After F-a19ace379809 returns, the subject’s dispatch-table address is read again. The entry at `+0x5c` is invoked with the subject alone. The initial table address is not visibly reused.

From the returned source object, the access order is:

1. Read `+0x84`.
2. Read `+0x88`.
3. Read `+0x8c`.
4. Read `+0x80` and write that word to item `+0xc4`.
5. Write the captured `+0x84` word to item `+0xc8`.
6. Write the captured `+0x88` word to item `+0xcc`.
7. Write the captured `+0x8c` word to item `+0xd0`.

Every access is 32 bits. The destination receives the corresponding four source words, but the transfer is not established as atomic.

The capture-before-store ordering matters when the objects overlap. In particular, the last three destination stores use words captured before any destination store; they do not reread their source fields after earlier writes.

### Associated object and captured handle

F-2698159acdb1 receives six arguments in this order:

1. Context.
2. Zero.
3. Item.
4. The 32-bit creation-argument static.
5. Zero.
6. One.

That static is accessed at this stage, after the four-word copy. Earlier dependencies are not assumed to preserve its contents.

The returned object’s word at `+0x04` is then read and retained as a handle. Next, F-a8c06eac92c0 receives:

1. Context.
2. The returned object.
3. Item.
4. Address of the original-position pair.
5. Zero.

The handle read precedes this call. If the dependency changes the object’s `+0x04` field, later arguments still derive from the captured handle; the function does not reread that field.

### Storage request and configuration

Mode control and selector choose a configuration value:

| Condition | Value |
|---|---|
| Mode control is zero | 6 |
| Mode control is nonzero and selector is 1 | 0 |
| Mode control is nonzero and selector is not 1 | 9 |

On each branch, F-8157a180d901 is called exactly once with `0x24`, decimal 36. Allocation intent is inferred; the visible contract is the argument and subsequent use of its result.

F-3a954974c1bb then receives:

1. The F-8157a180d901 result.
2. Captured handle.
3. Subject.
4. Item.
5. Selected configuration value.

No result from F-3a954974c1bb is visibly consumed.

### Final calls and completion write

The remaining calls occur in order:

1. F-4b539170a43a receives the retained F-7a89f2f7fd9a result and the object returned by F-2698159acdb1.
2. F-aa408aa1a280 receives that retained result, the captured handle, and the 32-bit finalization-argument static read at this stage.
3. A temporary 32-bit slot is initialized with the captured handle. F-7e55b35f8edc receives the address of context `+0x58` and the address of that slot.

The finalization static is not established as preserved across earlier calls, including F-4b539170a43a.

After these calls return, the function writes the one-byte value 1 to the completion byte selected by 0, 1, 2, or 4. Other selectors cause no direct completion-byte write.

There is no second gate check before the write. Return from the dependencies, rather than an inspected success result, is what permits reaching it. The function performs no visible rollback of earlier actions.

## Side effects

Direct object writes are limited to the four 32-bit item fields and, on a recognized-selector completion path, one context byte. Temporary position and handle storage is also written.

The unconditional displacement-static read is part of the established memory-access sequence even when zero control causes its value to be discarded. This observation does not establish that the storage is volatile, hardware-backed, or externally observable in any particular runtime.

Every dependency and dispatched method may have additional effects not described by this body. No preservation of globals, context fields, dispatch tables, item fields, or allocation state is assumed.

No explicit release, destruction, or ownership-transfer operation is established. The absence of a local release proves neither a leak nor ownership transfer. The behavior of the configuration and registration dependencies remains unknown.

Temporary addresses are supplied to three dependencies: the adjusted position to F-a19ace379809, the original position to F-a8c06eac92c0, and a handle slot to F-7e55b35f8edc. Their storage is not established as valid after this invocation. Copying, retention, or asynchronous use by callees is unresolved.

No locking or atomic completion protocol is visible. Since the byte is written only at the end, the gate alone does not prevent reentrant processing before completion.

## Dependencies

| Opaque dependency | Established interaction |
|---|---|
| F-7a89f2f7fd9a | Supplies a retained 32-bit value for two later calls. |
| F-38aa1bb9efd0 | Supplies an object whose `+0xe8` word is read. |
| F-db754b1ab3c7 | Receives that word and selector; zero terminates processing. |
| F-a19ace379809 | Receives item and adjusted-position address. |
| F-2698159acdb1 | Receives six arguments and supplies an object containing the handle word. |
| F-a8c06eac92c0 | Receives context, associated object, item, original-position address, and zero. |
| F-8157a180d901 | Receives 36; its unchecked result is passed onward. |
| F-3a954974c1bb | Receives that result, handle, subject, item, and configuration value. |
| F-4b539170a43a | Receives retained service value and associated object. |
| F-aa408aa1a280 | Receives retained service value, handle, and a static value. |
| F-7e55b35f8edc | Receives embedded-target address and temporary-handle address. |

Opaque IDs for the two dynamically selected method targets are unavailable. Their established identities are limited to subject-table entries `+0xa0` and `+0x5c`, their arguments, and the latter’s result use.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, all reached accesses are valid, dependencies return normally, and no interfering mutations occur.

1. **Nonzero completion byte.** With selector 2 and byte `+0x9a` equal to 7 when inspected, the initial subject callback runs, then processing returns. Neither lookup nor the displacement-static read occurs.

2. **Callback changes the gate.** With selector 0, an initially clear byte changed to nonzero by the first callback causes the subsequent gate to return.

3. **Lookup miss.** A zero lookup result follows the initial callback, service acquisition, and lookup preparation. It prevents both item-position reads and the displacement-static read, as well as all later processing.

4. **Zero control still reads the static.** Given item position `(100, 25)` and displacement control zero, the function reads both components, then reads the displacement static, then replaces the captured displacement with 320. The adjusted pair supplied to F-a19ace379809 is `(420, 25)`. Even if the static contains a different value, that value does not contribute to this addition.

5. **Nonzero control uses the captured static.** If displacement control is nonzero and the displacement-static read yields 12, position `(100, 25)` produces adjusted pair `(112, 25)`. This example is conditional on the read yielding 12; it does not identify the actual static contents.

6. **Original position remains distinct.** Under the noninterference assumption, the zero-control example later supplies original pair `(100, 25)` to F-a8c06eac92c0, not `(420, 25)`.

7. **Overlapping copy.** If item `+0xc4` aliases source `+0x84`, the first destination store may overwrite that source field. Item `+0xc8` nevertheless receives the value captured from source `+0x84` before the overwrite.

8. **Mode precedence.** Selector 1 with mode control zero selects 6. Selector 1 with any nonzero mode control selects 0. Selector 4 with nonzero mode control selects 9.

9. **Unrecognized selector.** Selector 3 bypasses completion-byte inspection. A successful lookup can reach every later call, but no completion byte is directly written.

10. **Handle capture.** If the associated object contains handle H when read and F-a8c06eac92c0 later changes that field to J, subsequent handle arguments use H rather than rereading J.

11. **Late completion.** Selector 4 reaches a direct write of 1 to `+0x9b` only after the three final calls return. No success result from those calls is inspected. A nonlocal exit before that write does not establish completion or rollback.

## Uncertainties

The following remain **documented_only**:

- Numeric contents and semantic meanings of all three static values.
- Exact calling convention, concrete machine arithmetic, and out-of-range signed-addition behavior.
- Semantic types of the four-word block, position components, retained service value, and handle.
- Callee mutations of globals and aliased objects.
- Allocation, construction, registration, failure, and ownership semantics.
- Resource lifetime and destruction obligations.
- Whether temporary-address arguments are consumed synchronously.
- Exception, fault, and other nonlocal-exit behavior.
- Thread safety and external serialization.

The displacement static’s value is unknown; its unconditional read after the two position reads is established. These are separate conclusions.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. Established claims concern visible accesses, widths, branch conditions, argument values, call order, direct writes, and termination paths.

The position sequence explicitly reads two item words and then the displacement static before testing displacement control. Zero control replaces the captured value; it does not bypass that read.

Role labels express inferred intent only. Unavailable callee behavior and arithmetic details remain unresolved rather than being supplied from convention.

No runtime execution or runtime tests have been performed. Acceptance examples specify conditional expected behavior, not observed results.
