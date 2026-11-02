<!-- Trusted provenance: F-ecee49e1480c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ecee49e1480c

## Purpose

F-ecee49e1480c performs four ordered indirect calls through an object’s dispatch table and returns a 16-bit value assembled from the low bytes of two invocation-local 32-bit storage locations.

The first and third calls receive the same caller-supplied 32-bit argument but different writable local addresses. Between those calls, the function invokes another dispatch-table entry with an object-associated value and the constant `0x80`. After assembling the return value, it invokes that entry again with the object-associated value and zero.

The established behavior is this call sequence and byte assembly. Interpreting the calls as reads, mode selection, hardware access, or a transaction is inferred intent, not established behavior.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Representation in the supplied model | Role |
|---|---|---|
| Object | Pointer to an object whose first word supplies a dispatch-table base | Receiver for every indirect call and source of dispatch information |
| Selector | 32-bit value | Passed unchanged to the first and third indirect calls |
| High-byte seed | 32-bit value | Initializes the local storage passed to the third indirect call |
| Low-byte seed | 32-bit value | Initializes the local storage passed to the first indirect call |

The selector and both seeds have no established signed interpretation. All 32 bits of each seed initially populate its corresponding local storage location.

The supplied address expressions use a 32-bit object-word model. Dispatch-table offsets are byte offsets, not table-entry numbers. Object element index `0x14` corresponds to byte offset `0x50` under that model. The machine calling convention and complete pointer ABI are not supplied.

### Output

On normal completion, the function returns a 16-bit value:

- Bits 0–7 are the low eight bits of the low-byte local at the assembly point.
- Bits 8–15 are the low eight bits of the high-byte local at that same point.
- Bits 8–31 of either local do not contribute.

Equivalently, for the two 32-bit values observed at assembly time:

**result = 256 × (high-byte local modulo 256) + (low-byte local modulo 256).**

This operation describes numeric bit placement, not a memory serialization format. It does not establish the platform’s byte order.

### Required object layout

The function directly relies on these locations:

| Location | Required use |
|---|---|
| Object offset `0x00` | Read as the dispatch-table base |
| Dispatch-table offset `0x9c` | Read as an indirect call target for the first and third calls |
| Dispatch-table offset `0xa4` | Read as an indirect call target for the second and fourth calls |
| Object offset `0x50` | Read as the second argument of the second and fourth calls |

No other object fields are directly accessed by the visible function. Indirect callees may access additional fields or unrelated state.

### Preconditions

For ordinary completion:

- The object must remain valid for all required object reads.
- Each dispatch-table base read must identify accessible entries at the offsets used at that point.
- Each selected target must be callable with the supplied arguments.
- Each indirect call must return normally.
- Indirect callees must use any supplied local-storage addresses compatibly with their extent and lifetime.

No null checks, layout checks, target checks, or explicit error-return path are visible.

## Behavior

### 1. Initialize invocation-local storage

Before the first indirect call, two separate 32-bit local storage locations are initialized:

- The high-byte local receives the third argument.
- The low-byte local receives the fourth argument.

These are initialized values, not uninitialized output buffers. If a callee leaves a location untouched, that location retains its prior value unless some other permitted access changes it.

The initializations precede all four indirect calls. No externally observable distinction between their initialization order is established.

### 2. Invoke the first writable-local operation

The function reads the object’s dispatch-table base, selects the entry at byte offset `0x9c`, and calls that target with:

1. The original object pointer.
2. The selector.
3. The address of the low-byte local.

The first call therefore exposes the local initialized from the fourth argument.

Its return value, if any, is not consumed. The supplied source does not establish whether the target reads the local, writes it, does both, or ignores it. In particular, it does not establish that the target must replace all 32 bits.

### 3. Invoke the intermediate control operation

After the first call returns, the function obtains the dispatch-table target at byte offset `0xa4` and reads the object-associated value at byte offset `0x50`.

It calls the selected target with:

1. The original object pointer.
2. The value read from object offset `0x50`.
3. The 32-bit constant `0x80`.

The return value is not consumed.

This call is ordered after the first call and before the third. The visible function does not specify the relative evaluation order of the independent reads used to prepare this call. In particular, it does not establish whether the object-associated value is read before or after the dispatch information.

### 4. Invoke the second writable-local operation

After the intermediate call returns, the function again obtains a dispatch-table base from the object and selects the entry at byte offset `0x9c`.

It calls that target with:

1. The original object pointer.
2. The same selector supplied to the first call.
3. The address of the high-byte local.

This local was initialized from the third argument.

The target is resolved again rather than established once for both writable-local calls. If an earlier dependency changes the object’s dispatch-table base or the relevant entry, the third call can reach a different target from the first call.

Its return value is not consumed.

### 5. Assemble the return value

After the third call returns, the function takes the low eight bits of each local and combines them into the 16-bit result described in the interface.

This occurs before the fourth call.

The low-byte local is not sampled immediately after the first call. If its address has been retained and it is validly modified during a later dependency call before assembly, that later value contributes to the result.

Only the low eight bits of each local matter. A 32-bit value with low byte `0xff`, for example, contributes `0xff` regardless of its upper 24 bits or any signed interpretation.

### 6. Invoke the final control operation

After assembling the result, the function resolves the entry at dispatch-table offset `0xa4` again and reads the object-associated value at offset `0x50` again.

It calls the selected target with:

1. The original object pointer.
2. The newly read object-associated value.
3. The 32-bit value zero.

The return value is not consumed.

Neither the target nor the object-associated argument is required to match the corresponding value used by the intermediate control call. Earlier dependencies may have changed either one.

### 7. Return

If the fourth call returns normally, the function returns the already assembled 16-bit result.

Later changes to the two local values during the fourth call do not cause the function to recompute that result. This statement assumes valid execution; arbitrary corruption of execution state is outside the established contract.

## Side effects

### Direct effects

The visible function:

- Initializes two 32-bit local storage locations.
- Passes their addresses to separate indirect calls.
- Reads the object’s dispatch-table base for each call.
- Reads the applicable dispatch-table entry for each call.
- Reads object offset `0x50` separately for the second and fourth calls.
- Makes four indirect calls in the specified order.
- Computes a 16-bit return value between the third and fourth calls.

It contains no explicit direct write to the object or dispatch table.

### Dependency effects

No preservation guarantees for globals, object fields, dispatch tables, external resources, or previously exposed local storage are supplied.

The function itself performs no explicit allocation, deallocation, ownership transfer, or reference-count operation. That does not establish that its dependencies avoid such operations.

The local addresses are borrowed addresses into the current invocation. Passing them does not establish ownership transfer. Although a dependency could retain one, it would not remain a valid address to this invocation’s local storage after the function returns.

The final zero-valued call is unconditional on the normal execution path, but it is not proven cleanup:

- It is not reached if an earlier call fails to return normally.
- It may resolve to a different target from the intermediate control call.
- It may receive a different object-associated argument.
- Its effect is unknown.

Consequently, there is no established guarantee that the function restores an initial mode, releases a resource, or reverses the intermediate call.

### Errors and abnormal completion

There is no visible validation, error translation, retry, or conditional recovery.

Invalid object memory, invalid dispatch information, incompatible targets, or dependency failures have no defined function-level recovery behavior in the supplied evidence. If a dependency does not return normally, subsequent calls and the final return are not established to occur.

## Dependencies

The following opaque IDs designate unresolved dependency roles for this contract. They are not claims that concrete target identities have been recovered.

| Opaque ID | Resolution and use | Established interface |
|---|---|---|
| F-000000000001 | Dispatch-table byte offset `0x9c`, resolved separately for calls one and three | Object pointer, 32-bit selector, address of a 32-bit local |
| F-000000000002 | Dispatch-table byte offset `0xa4`, resolved separately for calls two and four | Object pointer, object-associated 32-bit value, 32-bit control value |

A role may resolve to different concrete functions during one invocation. Conversely, distinct entries could contain the same target address. Neither equality nor inequality of the targets is guaranteed.

No dependency return value affects the visible function’s control flow or returned bits. Dependency side effects can affect later target resolution, later object-associated reads, and local values sampled for assembly.

## Acceptance examples

These examples are conditional predictions from the static evidence, not executed tests.

### Unmodified seeds

Assume all calls return normally and no dependency changes either local.

With:

- High-byte seed: `0x12345678`
- Low-byte seed: `0xabcdef90`

the returned value must be `0x7890`.

Upper seed bits are discarded only when the result is assembled; the initialized locals initially contain the complete 32-bit seeds.

### Both locals replaced

Assume the first call writes `0x11223344` to its supplied local and the third writes `0xaabbccdd` to its supplied local. Assume neither value changes again before assembly.

The returned value must be `0xdd44`.

The two control calls must still occur between the writable-local calls and after assembly, respectively.

### Earlier local changed later

Assume the first call writes a value whose low byte is `0x11` and retains the local address. During the third call, that retained address is validly used to change its low byte to `0x22`. The high-byte local has low byte `0x33` at assembly.

The returned value must be `0x3322`, not `0x3311`.

### Object-associated value changes

Assume object offset `0x50` yields value A for the second call and is subsequently changed so that the fourth call reads value B.

The second control call receives A with `0x80`. The fourth receives B with zero. The function does not preserve A for reuse.

### Dispatch changes

Assume the first dependency changes the object’s dispatch-table base and leaves a valid replacement table.

Subsequent calls resolve through the table available at their own resolution points. Reusing a target cached before the first call would not match the visible behavior.

### Final call modifies exposed storage

Assume assembly produces `0x5678`. During the fourth call, a dependency validly changes one or both exposed locals through retained addresses.

If that call returns normally without invalidating execution state, the function still returns `0x5678`; no second assembly occurs.

### A dependency does not return

Assume the third call does not return normally.

The function does not reach the visible assembly operation or fourth call on that execution path. A successful return value and a final zero-valued control operation are not guaranteed.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the selector.
- The meaning of the object-associated value at offset `0x50`.
- The meanings of control values `0x80` and zero.
- The concrete identities and full contracts of the indirect targets.
- Whether the writable-local operations read, replace, or partially update their supplied storage.
- Whether dependencies retain addresses or mutate object, table, global, or external state.
- The complete object layout, calling convention, and pointer ABI.
- Thread-safety, reentrancy, synchronization, and behavior under concurrent mutation.
- Any error signaling through ignored dependency returns or external state.
- Whether the final control operation restores anything.

An interpretation in which the function combines two byte observations around a control-state change is plausible intent. It is not required by the visible evidence.

## Evidence

**Established static evidence:** two 32-bit local initializations; four ordered indirect calls at dispatch-table offsets `0x9c`, `0xa4`, `0x9c`, and `0xa4`; object-associated reads at offset `0x50` for the second and fourth calls; control arguments `0x80` and zero; and assembly of a 16-bit result before the final call.

**Inferred intent:** the local-address calls may obtain byte-oriented observations, and the other calls may change a control state. Neither interpretation is needed to specify the observable sequence.

**Unknowns:** dependency semantics, environmental behavior, complete ABI details, and abnormal-completion behavior beyond the absence of visible recovery.

No runtime tests have been performed.
