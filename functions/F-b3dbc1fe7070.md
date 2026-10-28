<!-- Trusted provenance: F-b3dbc1fe7070; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3dbc1fe7070

## Purpose

F-b3dbc1fe7070 conditionally scans an indexed collection associated with an input object. It returns normally if the scan is disabled, if the captured item count is nonpositive, or if every inspected item has a zero first word. On the first inspected item whose first word is nonzero, it stops scanning and calls F-1724d515319a, followed—if that call returns—by F-feee9156a257.

The supplied static representation marks F-feee9156a257 as non-returning. Consequently, the nonzero-item path has no established normal return.

An inferred interpretation is that this routine checks an enabled collection for a disallowed or exceptional item and then enters a terminal handling path. That interpretation is not required by the contract: the meanings of the enable field, item word, and terminal action are unknown.

## Interface

**Input:** One pointer to an object containing integer-valued fields and an indirect-dispatch reference.

**Output:** No return value. Normal completion provides no explicit indication of how many items were inspected.

The recovered layout is expressed below using 32-bit integer words and byte offsets. This is the apparent target data model; the supplied fragment does not independently establish the complete ABI, address width, byte order, or calling convention.

| Object location | Observed role | Access |
|---|---|---|
| Byte offset 0 | Dispatch-base value used to locate the indexed-access callback | Read once per attempted item access |
| Byte offset 4 | Signed item-count bound | Read once after the enable check succeeds |
| Byte offset 40 | Integer field converted to a character-sized value for the enable check | Read before any count or dispatch access |

The dispatch operation uses a callable entry at byte offset 64 from the dispatch base. That entry is invoked with the original object pointer and the current index. Its result is treated as a pointer to an integer word; the first such word is read and compared with zero.

The exact pointer-slot width and call ABI are not independently established. In particular, byte offset 64 is established as a displacement, not as a universally valid slot number.

### Preconditions

The necessary readable storage depends on the path actually taken:

- The input pointer must permit the enable-field access.
- If the enable value is nonzero, the count field must also be readable.
- If the captured count is positive, each attempted dispatch must resolve to a valid callable entry.
- Each callback invocation that returns must supply a pointer from which an integer word can be read.

There is no explicit null-pointer check, dispatch validation, returned-pointer validation, or recovery from an invalid access.

The function does not acquire ownership of the object or callback results in any observable local operation. It also performs no explicit release. This does not establish that dependencies preserve ownership or refrain from releasing memory.

## Behavior

### 1. Enable decision

The function first reads the integer field at byte offset 40 and converts it to a character-sized value. If that converted value is zero, it returns immediately.

Under an 8-bit character model, this tests whether the low eight bits are zero, not whether the entire 32-bit field is zero. Thus an integer value of 256 disables the scan under that model, even though the full integer is nonzero. Whether the character type is signed or unsigned does not change the zero-versus-nonzero result under the ordinary low-eight-bit conversion model.

The fragment does not independently establish character width or all implementation-defined signed conversion rules. Those platform details remain qualifications on the low-eight-bit interpretation.

On this early-return path, the count and dispatch fields are not read by this function.

### 2. Capture the count

If the enable check succeeds, the function reads the signed count at byte offset 4 and retains that value as the loop bound.

The initial index is zero. The bound is compared with that index before any callback invocation. If the captured count is zero or negative, the function returns normally without performing an indexed access.

The count is not reread during the scan. A callback that changes the count field therefore does not change the retained bound for this invocation. This fact does not make the rest of the object immutable or safe: subsequent dispatch operations can still observe changes to other object state.

### 3. Inspect items in ascending index order

For a positive captured count, indices are attempted in ascending order, starting at zero.

For each attempted index, the function:

- Reads the current dispatch-base value from the object.
- Resolves the callable entry at byte offset 64 from that base.
- Invokes that entry with the object pointer and current index.
- Reads the first integer word through the returned pointer.
- Decides whether to continue based only on whether that word is zero.

The dispatch base and callable entry are resolved again for every attempted index. They are not captured once for the whole scan. A dependency-induced change to the object's dispatch reference can therefore affect a later iteration.

A zero item word advances the index by one. A nonzero item word ends the scan immediately. No later index is requested after a nonzero word is observed.

If every inspected item word is zero, the function returns when the index reaches the captured bound. With a stable, valid environment and a positive count of \(N\), that path makes exactly \(N\) indexed callback invocations.

### 4. Enter the nonzero-item path

After observing a nonzero item word, the function invokes F-1724d515319a with no explicit arguments.

If that invocation returns normally, it next invokes F-feee9156a257, also with no explicit arguments. The supplied representation marks this second dependency as non-returning. No normal result or subsequent action is established after that call.

The function does not explicitly pass the triggering index, returned item pointer, item word, or input object to either of these two dependencies. They may nevertheless obtain information through shared state or ABI mechanisms not visible in the fragment; such behavior is unknown.

### Integer boundaries

The count, index, and inspected item word are represented as signed integer values, with an apparent width of 32 bits.

Under a signed 32-bit model, the loop's own index increment does not overflow. An increment occurs only after inspecting an index strictly below the positive captured bound. Even with the maximum signed count, the last increment reaches the maximum signed value, after which the bound check returns without another increment.

This arithmetic property does not guarantee that the callback supports every permitted index or that the corresponding data exists.

## Side effects

The function contains no explicit store to the input object, returned item storage, or a global location. It performs no explicit allocation, deallocation, reference-count update, or ownership transfer.

Its observable effects nevertheless include:

- Reading the enable field.
- Conditionally reading and retaining the count.
- Resolving and invoking an indirect dependency for each attempted item.
- Reading one integer word from each returned pointer.
- Conditionally invoking F-1724d515319a and F-feee9156a257.

Those dependencies may mutate the object, alter shared state, allocate or release resources, perform external I/O, or fail to return. None of these possibilities is ruled out by the supplied fragment.

The read-and-call order matters. The count is captured before the first callback. Each item word is read after its callback returns. The terminal dependencies are invoked only after a nonzero item word has been read, and F-feee9156a257 is reached only if F-1724d515319a returns.

No synchronization, atomic access, or exception handling is shown. Behavior under concurrent mutation, invalid memory, or dependency failure is not defined by this fragment. No cleanup path is established for those cases.

## Dependencies

| Dependency | Invocation condition | Established interface and limitation |
|---|---|---|
| Indirect indexed-access dependency; opaque function ID unavailable | Once for each attempted index | Receives the original object pointer and the current integer index. Returns a value interpreted as a pointer to a readable integer word. Its implementation and broader effects are unknown. |
| F-1724d515319a | After the first observed nonzero item word | Receives no explicit arguments. Its return behavior, result, state changes, and ownership effects are unknown. Any result is unused here. |
| F-feee9156a257 | After F-1724d515319a returns on the nonzero-item path | Receives no explicit arguments. Marked non-returning in the supplied static representation; the mechanism and external outcome are unknown. |

The missing opaque ID for the indirect dependency is **documented_only**. Its identity is not inferred from the dispatch displacement.

## Acceptance examples

These are conditional contract examples, not runtime observations. Except where stated otherwise, they assume valid storage, an 8-bit character and signed 32-bit integer model, and callbacks that return normally with readable pointers.

1. **Disabled scan.** The enable field converts to zero. The function returns without reading the count, resolving a callback, or invoking either named dependency.

2. **Nonzero field with zero low byte.** The enable field contains 256. Under the stated character-conversion model, the function takes the disabled path. A contract that enables scanning for every nonzero full-width value would not match this behavior.

3. **Empty or negative count.** The enable check succeeds and the captured count is zero or negative. The function returns without any indexed callback invocation.

4. **All inspected words are zero.** The captured count is three. Callback invocations at indices 0, 1, and 2 each return a pointer whose first word is zero. The function returns normally after those three accesses. Neither named dependency is invoked.

5. **First item is nonzero.** The captured count is positive, and the callback result for index 0 has first word −1. The sign is immaterial: the word is nonzero. No index 1 access occurs. F-1724d515319a is invoked, followed by F-feee9156a257 if the first call returns.

6. **Later item is nonzero.** The captured count is four. Index 0 produces zero and index 1 produces a nonzero word. The callback is invoked only for indices 0 and 1 before the terminal dependency sequence begins.

7. **Count mutation does not replace the bound.** The captured count is three. During the index 0 callback, the object count is changed to zero. If all returned pointers remain valid and all inspected words are zero, indices 1 and 2 are still attempted.

8. **Dispatch mutation can affect the next access.** The index 0 callback changes the object's dispatch reference and returns an item whose first word is zero. The next iteration resolves its callback through the changed reference, rather than reusing the earlier callable entry.

9. **Invalid callback result.** A callback returns a null or otherwise unreadable pointer. The function attempts the item-word read; no checked error return or recovery is established.

10. **First terminal dependency does not return.** A nonzero item is observed, but F-1724d515319a does not return normally. F-feee9156a257 is not thereby guaranteed to execute.

## Uncertainties

The following are **documented_only**:

- The semantic identity of the object, collection, enable field, and inspected item word.
- The complete target ABI, including pointer width, character width, alignment requirements, and byte order.
- The indirect callback's opaque identity, supported index range, result lifetime, and ownership behavior.
- Whether dependencies modify globals, the object, dispatch state, or returned storage.
- The purpose and external effects of F-1724d515319a.
- Whether F-feee9156a257 terminates a process, throws, transfers control, blocks indefinitely, or has another non-returning effect.
- The outcome if the non-returning annotation is inaccurate and F-feee9156a257 actually returns.
- Behavior under concurrency, reentrancy, invalid memory, or unsupported callback indices.

The inferred exceptional-condition purpose must not be promoted into a claim about diagnostics, process termination, resource cleanup, or error reporting.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the enable conversion and early return; the single captured signed count; ascending indexed dispatch through byte displacement 64; the first-word zero test; early scan termination on a nonzero word; the ordered calls to F-1724d515319a and F-feee9156a257; and the annotation that the latter does not return.

The 32-bit word layout and 8-bit enable interpretation describe the apparent recovered target model, with the platform qualifications stated above. Semantic purpose, dependency internals, and ownership guarantees are not established.

No runtime tests have been performed.
