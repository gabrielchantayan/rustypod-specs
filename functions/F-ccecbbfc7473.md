<!-- Trusted provenance: F-ccecbbfc7473; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ccecbbfc7473

## Purpose

F-ccecbbfc7473 conditionally retrieves a descriptor for an unsigned selection, optionally delivers that descriptor to a supplied object, and returns a 32-bit status.

A context byte distinguishes mode 2 from other modes:

- In mode 2, ordinary nonzero selections are translated to zero-based retrieval indices. Selection 0 and the all-ones selection instead invoke F-fbe40d09d5eb, provided the selection passes the acceptance check.
- In other modes, accepted selections are forwarded unchanged to a different retrieval dependency.

Every accepted selection processed through the mode-2 branch also causes two ordered state stores. Their destinations are obtained from **two separate reads of a global reference**. The stores target the same object only if those reads yield the same reference.

Descriptor delivery is enabled by a nonzero one-byte flag. Completed delivery forces the return status to 0. Without delivery, the status remains 9 unless F-fbe40d09d5eb supplied a replacement.

Application meanings such as “current selection,” “reset,” and “notification” are not established.

## Interface

### Arguments

| Argument | Representation | Use |
|---|---|---|
| Context address | 32-bit address-like value | Supplies a readable byte at offset `0x2ec`; passed to F-fbe40d09d5eb on its applicable paths. |
| Object address | Pointer to an object whose first 32-bit word supplies a dispatch-table address | Supplies the receiver and dispatch information for indirect calls. |
| Selection | Unsigned 32-bit value | Compared with an unsigned bound, then forwarded unchanged or decremented for retrieval. |

The distinguished selection is `0xffffffff`, or 4,294,967,295. It bypasses the ordinary unsigned range comparison.

No null checks, address checks, dispatch validation, or result-capacity checks are visible. Every accessed address and called entry must be valid for the executed path. Writable state destinations need not be the same address or belong to the same object.

### Result

The function returns one 32-bit word:

- The initial status is 9.
- F-fbe40d09d5eb replaces that status with its own 32-bit result when invoked.
- A descriptor-delivery call that returns normally is followed by replacement of the status with 0.

Retrieval results and the delivery dependency’s return value are not used as status values. No symbolic interpretation of 0, 9, or helper-produced values is established.

### Retrieval-result layout

A retrieval dependency receives the beginning of temporary storage with the following visible relative layout:

| Offset | Width | Treatment by this function |
|---|---:|---|
| `0x00` | 4 bytes | Not inspected. |
| `0x04` | 1 byte | Copied as the descriptor flag. |
| `0x05`–`0x07` | 3 bytes | Not inspected. |
| `0x08` | 4 bytes | Copied to outgoing offset `0x08`. |
| `0x0c` | 4 bytes | Copied to outgoing offset `0x0c`. |

The function does not initialize this result storage before passing it to a retrieval dependency. Deterministic use therefore requires the dependency to supply every byte subsequently read.

### Outgoing descriptor layout

| Offset | Width | Source |
|---|---:|---|
| `0x00` | 4 bytes | A global datum captured before any dependency call. |
| `0x04` | 1 byte | Initially zero; replaced by the retrieval flag on retrieval paths. |
| `0x05`–`0x07` | 3 bytes | No explicit initialization is visible. |
| `0x08` | 4 bytes | Retrieval-result word at offset `0x08`, when retrieval occurs. |
| `0x0c` | 4 bytes | Retrieval-result word at offset `0x0c`, when retrieval occurs. |

These are observed temporary-storage layouts, not a proven public descriptor type. The first word, flag, payload words, and intervening bytes have no established application-level interpretation.

## Behavior

### Initialization and bound acquisition

Before any dependency call, the function reads a global 32-bit datum into the outgoing descriptor’s first word, sets its flag to zero, and initializes the return status to 9.

The captured first word is not refreshed later. A dependency changing the source global does not, by that change alone, update the outgoing descriptor.

The function then reads the context byte at offset `0x2ec`:

- If it equals 2, F-000000000001 is called with the object. Its result is represented as a signed 32-bit value, incremented by one, and assigned to the unsigned 32-bit acceptance bound.
- Otherwise, F-000000000002 is called with the object, and its 32-bit result directly supplies the unsigned bound.

The bound-producing call occurs even for selection `0xffffffff`.

For the mode-2 calculation, signed values whose increment remains representable have a defined arithmetic interpretation in the supplied representation. Assignment of a negative result to the unsigned bound gives its corresponding unsigned 32-bit value. An increment from the signed maximum is unresolved; this contract does not silently assume wrapping.

### Acceptance check

A selection is accepted if either:

- It is unsigned-less-than the bound; or
- It equals `0xffffffff`.

An ordinary selection equal to the bound is rejected.

Rejection returns 9 without a second mode read, retrieval, helper invocation, descriptor delivery, or either explicit state store. The bound dependency has already run, so rejection does not imply absence of dependency side effects.

Selection 0 is not independently exempt from this check. With a zero bound, selection 0 is rejected.

### Second mode read

For an accepted selection, the function reads the context byte at offset `0x2ec` again. This read—not the first read—chooses the processing branch.

The bound dependency may therefore cause mixed-mode behavior:

- A mode-2 bound can be followed by non-mode-2 retrieval.
- A non-mode-2 bound can be followed by mode-2 handling.

The first mode value is not reused as an immutable decision.

### Accepted mode-2 selections

If the second mode read equals 2, handling depends on the original selection.

**Selection 0 or `0xffffffff`:** F-fbe40d09d5eb receives the context address and integer value 2. Its 32-bit return replaces the current status.

This path performs no local retrieval and no local assignment of a nonzero descriptor flag. Under the visible local writes, the flag remains zero and descriptor delivery is skipped. This does not establish that the helper lacks other side effects.

**Any other accepted selection:** F-000000000003 receives the retrieval-result address, the object, and the selection minus one.

Because this branch excludes 0 and `0xffffffff`, its possible forwarded indices range from 0 through `0xfffffffd`, subject to the acceptance bound. The subtraction does not underflow.

After retrieval returns, the function copies, in order:

1. The one-byte result flag to outgoing offset `0x04`.
2. The four-byte result word at offset `0x0c` to outgoing offset `0x0c`.
3. The four-byte result word at offset `0x08` to outgoing offset `0x08`.

Both payload copies occur even when the flag is zero.

### Mode-2 reference reads and state stores

After either the helper returns or retrieval and its copies finish, every accepted mode-2 path performs the following sequence:

1. Read the global state-object reference and save that first value.
2. Read the same global reference again.
3. Using the **second** reference, store the 32-bit value `0xffffffff` at offset `0x1c`.
4. Using the **first, saved** reference, store the original 32-bit selection at offset `0x20`.

Both reference reads occur after the helper or retrieval dependency returns. Neither is a reference captured before that dependency.

The first read supplies the destination base for the **later** store. The second read supplies the destination base for the **earlier** store. There is no intervening explicit dependency call between these reads or stores.

Without an additional stability assumption, the two references need not be equal. If the first read yields reference A and the second yields reference B, the writes are to B at offset `0x1c`, then A at offset `0x20`. The function does not reload the reference for the final store.

If both reads yield the same reference, the familiar same-object interpretation applies: that object receives `0xffffffff` at offset `0x1c`, followed by the original selection at offset `0x20`.

These stores precede any descriptor delivery.

### Accepted non-mode-2 selections

If the second mode read is not 2, F-000000000004 receives the retrieval-result address, the object, and the original selection.

The all-ones selection is forwarded unchanged on this path. The function does not locally validate whether the retrieval dependency supports it.

After retrieval returns, the flag and payload words are copied in the same order described for mode-2 retrieval.

This branch performs neither of the explicit global-reference reads or state stores described above.

### Conditional descriptor delivery

After either accepted processing branch, the function checks the outgoing one-byte flag:

- Zero skips delivery.
- Any nonzero byte invokes F-000000000005 with the object and outgoing descriptor address.

The flag need not equal 1. A byte with its high bit set also satisfies the nonzero condition.

After delivery returns normally, the status is set to 0 regardless of the dependency’s return value. No later local state writes are shown.

## Side effects

### Established local effects

Depending on the path, the function:

- Reads the descriptor-prefix global datum.
- Reads the context mode once or twice.
- Reads object dispatch information for indirect calls.
- Writes temporary result-related and outgoing-descriptor storage.
- Performs two separate reads of the global state-object reference.
- Stores `0xffffffff` through the second reference at offset `0x1c`.
- Stores the original selection through the first reference at offset `0x20`.

The two state stores are the only explicit writes outside temporary storage. They are not unconditionally writes to one object.

### Observable ordering

The established sequence is:

1. Capture the outgoing descriptor’s first word and initialize local status and flag.
2. Read the initial mode.
3. Invoke the corresponding bound dependency.
4. Check acceptance.
5. If accepted, read the mode again.
6. Invoke the helper or retrieval dependency selected by that second read.
7. On retrieval paths, copy the flag, offset-`0x0c` word, then offset-`0x08` word.
8. On mode-2 paths, read and save the global reference, read it again, store through the second reference, then store through the saved first reference.
9. Deliver the descriptor if its flag is nonzero.
10. Return the resulting status.

Dependency effects are not constrained by the supplied body. No dependency is established to preserve the context, globals, object contents, dispatch table, or payload-reachable memory. Later indirect calls must not be specified as necessarily using the same dispatch-table contents as earlier calls.

The visible ordering does not establish atomicity, synchronization, or a valid concurrent-access protocol.

### Ownership and lifetime

No explicit allocation, deallocation, reference-count operation, or ownership transfer appears in this function.

The retrieval and outgoing descriptor addresses refer to temporary storage. Their being passed to dependencies does not establish permission to retain and dereference them after this function returns.

Payload words are copied as uninterpreted 32-bit quantities. They may be scalars, handles, or addresses; no ownership or release rule follows from the copies. Dependency allocation, retention, and release behavior remains unknown.

### Errors and interrupted execution

No explicit recovery is shown for invalid addresses, missing dispatch entries, incomplete retrieval output, dependency failure, or nonlocal exit.

A retrieval dependency’s returned value does not control whether its output is copied. If it returns normally without initializing the needed bytes, the function still proceeds to read those bytes.

Subsequent actions require preceding operations to complete. For example, failure during the first state store does not establish that the second store or delivery occurs. The function is not specified as transactional.

## Dependencies

These opaque IDs distinguish call sites; they do not prove that runtime implementations are distinct.

| ID | Dispatch-table byte offset | Established interaction |
|---|---:|---|
| F-000000000001 | `0x1c4` | Receives the object; supplies the mode-2 bound source before increment. |
| F-000000000002 | `0x1b4` | Receives the object; supplies the other-mode bound directly. |
| F-000000000003 | `0x1c8` | Receives result storage, object, and selection minus one for ordinary accepted mode-2 retrieval. |
| F-000000000004 | `0x1b8` | Receives result storage, object, and unchanged selection for accepted non-mode-2 retrieval. |
| F-000000000005 | `0x3c` | Receives object and outgoing descriptor address; its return value is ignored. |
| F-fbe40d09d5eb | Direct call | Receives context and 2 for accepted mode-2 selections 0 and `0xffffffff`; supplies a replacement status. |

No dependency body or complete dependency contract is available.

## Acceptance examples

These are conditional static examples, not executed tests. Required memory is valid and dependencies return normally unless stated otherwise.

1. **Ordinary rejection.** The first mode is not 2, the bound dependency returns 3, and selection is 3. The function returns 9. There is no second mode read, retrieval, helper call, delivery, or explicit state store.

2. **Direct retrieval and delivery.** Both mode reads are not 2, the bound is 3, and selection is 2. F-000000000004 receives index 2. If it supplies flag 7 and initialized payload words, delivery receives those values and the earlier captured prefix. The final status is 0.

3. **Zero retrieval flag.** Under the preceding conditions, a zero flag suppresses delivery. Both payload words are nevertheless copied, and the function returns 9.

4. **Translated retrieval with a stable reference.** Both mode reads equal 2, the bound source returns 4, and selection is 4. The bound is 5 and retrieval receives index 3. If both subsequent global-reference reads yield A, A receives `0xffffffff` at offset `0x1c`, then 4 at offset `0x20`, before any delivery.

5. **Different reference values.** On the same accepted mode-2 retrieval path, suppose the first reference read yields A and the second yields B. The function stores `0xffffffff` at B plus `0x1c`, then stores 4 at A plus `0x20`. It does not instead use B for both writes. This example is conditional; the evidence does not establish a mechanism that changes the reference between reads.

6. **Mode-2 zero selection.** Both mode reads equal 2, the computed bound is positive, and selection is 0. The helper receives the context and 2. After it returns, the two reference reads and corresponding stores occur, with the later store writing 0 through the first reference. With the flag remaining zero, the helper’s result is returned.

7. **Zero bound.** A computed bound of zero rejects selection 0 before the second mode read. Selection `0xffffffff` remains accepted.

8. **All-ones outside mode 2.** Both mode reads are not 2 and the bound is zero. Selection `0xffffffff` is forwarded unchanged to F-000000000004. Delivery depends on the supplied flag.

9. **Mode changes during bound acquisition.** The first mode read equals 2, its dependency produces a bound of 5 after increment, and changes the mode to a value other than 2. Selection 2 is retrieved unchanged through F-000000000004. No explicit state-reference reads or state stores occur.

10. **Delivery after split-destination stores.** An accepted mode-2 retrieval supplies a nonzero flag, and the two reference reads yield A then B. At entry to delivery, the store at B plus `0x1c` and then the store at A plus `0x20` have completed. Delivery may itself change memory; the function supplies no preservation guarantee.

## Uncertainties

- **documented_only — Increment overflow:** The mode-2 increment is represented as signed 32-bit arithmetic. Incrementing the signed maximum cannot be assigned a portable overflow result from this material. Low-32-bit wrapping is not established.
- **documented_only — Reference stability:** The two global-reference reads are distinct. Equality of their results, referent identity, reference lifetime, and synchronization are not guaranteed by the supplied body.
- **documented_only — Descriptor semantics:** The descriptor’s type, first-word meaning, flag meaning, and payload interpretation are unknown.
- **documented_only — Retrieval completeness:** Dependencies are not proven to initialize every subsequently read result byte.
- **documented_only — Uninitialized outgoing bytes:** Bytes `0x05`–`0x07` have no explicit initialization and must not be treated as deterministic.
- **documented_only — Dependency contracts:** Mutation, ownership, retention, allocation, release, and failure behavior remain unresolved.
- **documented_only — Concurrency:** Separate reads and stores do not establish atomicity, snapshot consistency, or safe concurrent access.
- **Inferred intent:** The mode-2 increment and index decrement suggest an extra leading selection distinct from retrieved entries. This does not establish application semantics.

## Evidence

The supplied static body establishes the widths represented here, constants, field offsets, dispatch offsets, acceptance condition, repeated mode read, retrieval-copy order, status assignments, and conditional delivery.

It also establishes the critical mode-2 sequence: first save one global-reference value, then read the global again for the offset-`0x1c` store, then use the saved first value for the offset-`0x20` store. A same-object interpretation requires an additional stability assumption.

The supplied dependency mapping identifies the direct helper as F-fbe40d09d5eb. Other dependency IDs are opaque call-site labels.

No dependency bodies, public type definitions, runtime traces, or independent behavioral documentation were supplied. No runtime tests have been performed.
