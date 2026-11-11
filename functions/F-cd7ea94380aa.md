<!-- Trusted provenance: F-cd7ea94380aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd7ea94380aa

## Purpose

F-cd7ea94380aa dispatches a 32-bit operation selector using an object, a context value, an item reference, and an output location. Seven selectors are recognized: `0x7692` through `0x7698`.

Recognized operations delegate to an object method, construct results through dependencies, or directly write zero. Unsupported selectors return zero without a direct output write. A conditional global preparation sequence precedes every dispatch, including unsupported selectors.

The contract distinguishes direct effects from dependency effects. In particular, writing zero before a dependency call does not guarantee that the output still contains zero afterward. Dependencies may modify accessible storage through aliases, globals, or callbacks.

The operations resemble metadata or presentation queries, but that interpretation is inferred intent. The established behavior is the dispatch, memory access, dependency-call ordering, output writes, and return values described below.

## Interface

The function accepts five inputs and returns a 32-bit value.

| Input | Representation | Observed use |
|---|---|---|
| Object | Pointer to storage containing 32-bit words and an initial method-table reference | Supplies virtual dispatch, embedded lookup state, and an associated word |
| Context | 32-bit value | Forwarded to delegated operations; compared with a captured global discriminator |
| Selector | 32-bit value | Chooses the operation |
| Item reference | Opaque 32-bit value | Passed to item lookup |
| Output | Pointer to writable 32-bit storage | Receives explicit writes on applicable paths; also passed to delegated operations |

Context comparison is equality of the represented 32-bit values; signed ordering is not involved. The item reference is not locally validated or dereferenced.

### Observed layouts

Offsets are byte offsets.

| Storage | Offset | Access |
|---|---:|---|
| Object | 0 | Method-table reference for delegated operations |
| Object | 40 | A 32-bit word read by `0x7698`; the address of this region is passed by `0x7697` |
| Object | 76 | Address of embedded state passed to item lookup |
| Lookup result | 0 | 32-bit payload, read on selected kind-2 paths |
| Lookup result | 4 | One-byte kind discriminator |

The delegated object method is selected at method-table offset 100. A second object, obtained during `0x7697`, supplies a method at method-table offset 556.

The function uses local temporary storage of 8 bytes, 20 bytes, and two separate 4-byte regions. These sizes describe the storage supplied to dependencies, not the size or ownership of any indirectly referenced allocation.

### Preconditions and return interpretation

No explicit null checks, lookup-failure checks, bounds checks, or output-capacity checks are visible. Storage must support every access performed by the executed path. Dependencies must accept the supplied object regions, values, and temporary storage.

The return value is not a uniform success flag:

- Locally completed recognized operations normally return 1.
- Unsupported selectors return 0.
- The context-mismatch path of `0x7697` returns 0 after directly writing zero.
- `0x7692` and `0x7693` return the delegated method’s result unchanged.

A non-null output pointer is required whenever the function directly writes through it. Paths without a direct write do not establish that a null pointer is acceptable to dependencies.

## Behavior

### Preparation before dispatch

Every invocation begins with the following conditional sequence:

1. Read the 32-bit flag word at offset 20 in a globally referenced region.
2. If bit 0 is set, skip the preparation calls.
3. Otherwise, call F-b255b26284e6 with a global state argument.
4. If that result is zero, skip the remaining preparation calls.
5. If it is nonzero:
   - Call F-5b24edfb22c0 with the global resource argument.
   - Call F-804adc29bbfb with that result and two additional global arguments, in their established argument positions.
   - Call F-d08d3ca17b81 with the global state argument.

After this sequence, read and capture the global 32-bit context discriminator. Dispatch then proceeds. That captured value, rather than a later reread, controls the context comparison in `0x7697`.

The function does not directly set the preparation flag or record completion. A dependency may do so, but repeated invocations are not guaranteed to skip preparation.

Global arguments are read where used. They must not be treated as an entry-time snapshot: an earlier dependency may change the resource, state argument, or other global values used later. The supplied representation establishes argument positions, but not a complete machine-level ordering among independent argument reads within a single call.

### Selectors `0x7692` and `0x7693`: delegated operation

Invoke the object method at method-table offset 100 with:

- The object.
- The context.
- The original selector.
- The output pointer.

The item reference is not forwarded.

Return the method’s 32-bit result unchanged. The function performs no direct output write after the call. Output behavior, ownership effects, and error interpretation therefore depend on the delegated method.

### Selectors `0x7694` and `0x7695`: temporary-derived result

Both operations call F-ebbd9627a906 with the embedded region at object offset 76 and the item reference.

Construct an 8-byte temporary using the lookup result and object:

- `0x7694` calls F-d19f8fa90ae4.
- `0x7695` calls F-8b3b4cd10074.

Each receives the temporary’s address, the object, and the lookup result.

Then:

1. Call F-46f0b3cb316e with the current global resource argument and the temporary.
2. Call F-289ec4311c64 with the temporary.
3. Call F-44362605ab79 with the current global resource argument.
4. Directly write its 32-bit result to the output.
5. Return 1.

The resource used by the final call is read after temporary processing and finalization. It is not guaranteed to equal the earlier resource value.

No kind byte or payload is directly inspected by these branches.

### Selector `0x7696`: exclusion or selector conversion

Look up the item through F-ebbd9627a906 and read the kind byte at offset 4.

If the kind is 2:

- Directly write zero to the output.
- Return 1.
- Do not call F-9d0ecc5f6adf or F-ea9442e8ddbb.

For every other kind:

1. Call F-9d0ecc5f6adf.
2. Call F-ea9442e8ddbb with that result as its first argument and `0x7696` as its second argument.
3. Directly write the conversion result to the output.
4. Return 1.

The second conversion argument is the unchanged selector, not a separate global token.

### Selector `0x7697`: context-gated classification

First, directly write zero to the output. Then compare the context with the discriminator captured after preparation.

If they differ, return 0. No item lookup occurs.

If they match, call F-ebbd9627a906 and read the returned item’s kind byte.

- **Kind 0:** Call F-9d0ecc5f6adf, then read global token source **K0**. Pass the returned value and token to F-ea9442e8ddbb, write its result, and return 1.
- **Kind 1:** Use the same sequence with distinct global token source **K1**.
- **Kind 2:** Follow the temporary-classification path below.
- **Every other kind:** Return 1 without another direct output write.

For the last case, zero is only the function’s initial direct write. The intervening lookup may modify the output through aliases or other unknown side effects. Consequently, the established guarantee is **return 1 with no direct output write after lookup**, not “return 1 with output zero.” Zero remains the final value only if intervening effects leave that location unchanged.

#### Kind-2 temporary construction

1. Call F-7d964f24a044 with the address of the object region at offset 40.
2. Call F-ebbd9627a906 again.
3. Read the second lookup result’s 32-bit payload at offset 0.
4. Invoke the obtained object’s method at method-table offset 556, passing the 20-byte temporary’s address, that object, and the payload.

The second lookup is independent of the first. Neither its pointer nor its payload is assumed unchanged. Its kind byte is not rechecked before the payload is used.

#### Classification precedence

All classification queries receive the 20-byte temporary.

- Call F-6e482452607b first.
- If it returns nonzero, call F-15632527c6ab. A zero result selects token source **T-A** immediately.
- Otherwise, continue to F-d1b4663968a8. A nonzero result selects **T-B**.
- If that result is zero, call F-e8ce52ce5425. A nonzero result selects **T-C**.
- If that result is also zero, call F-15632527c6ab. Zero selects **T-E**; nonzero selects **T-D**.

When F-6e482452607b returns zero, the early F-15632527c6ab call is skipped. When both early queries return nonzero and classification reaches the final query, F-15632527c6ab is called again. Its later result is a separate observation, not a reused answer.

After the classification branch is determined:

1. Call F-9d0ecc5f6adf.
2. Read the selected global token source.
3. Call F-ea9442e8ddbb with the returned value and token.
4. Directly write the conversion result to the output.
5. Call F-4e292a3dd745 with the temporary.
6. Return 1.

Token selection identifies a global source; it does not capture that source’s value before F-9d0ecc5f6adf. That dependency may change the value subsequently read.

Finalization occurs after the output write. The contract therefore does not guarantee that the written result remains unchanged if finalization can reach the output storage.

### Selector `0x7698`: gated associated-value conversion

Look up the item and read its kind.

For a kind other than 2, skip wrapper construction and use the fallback below.

For kind 2:

1. Read the 32-bit word at object offset 40.
2. Call F-83300d26ff95 to construct a 4-byte wrapper from that word.
3. Call F-552123f40f1e with the wrapper and retain its zero/nonzero result.
4. Call F-561858688f94 with the wrapper.

A zero gate result selects the fallback. A nonzero gate result selects the associated-value path. Finalization of the first wrapper occurs before either path continues.

#### Associated-value path

1. Call F-ebbd9627a906 again.
2. Read the returned item’s 32-bit payload.
3. Read the object word at offset 40 again.
4. Construct a second, separate 4-byte wrapper through F-83300d26ff95.
5. Call F-1f4c3d25219d with that wrapper and the captured payload.
6. Call F-9d0ecc5f6adf.
7. Call F-ea9442e8ddbb with the latter result first and the associated-value result second.
8. Directly write the conversion result to the output.
9. Finalize the second wrapper through F-561858688f94.
10. Return 1.

The second lookup, payload read, and object-word read occur after the first wrapper’s finalization. They cannot be replaced by cached values on an assumption of unchanged state. The second lookup’s kind is not checked.

#### Fallback

Call F-9d0ecc5f6adf, then call F-ea9442e8ddbb with that result and numeric zero. Write the conversion result and return 1.

This is conversion of zero, not a direct zero output write. The resulting output may be nonzero.

### Unsupported selectors

After preparation and discriminator capture, return 0 without a direct output write. Preparation dependencies may nevertheless modify the output or other state if they can reach that storage.

## Side effects

Direct externally visible writes by this function target the 32-bit output location. Dependencies may additionally modify objects, globals, aliased output storage, or indirectly reachable state.

Important ordering boundaries are:

- `0x7697` writes zero before context comparison and lookup.
- Its unrecognized-kind branch performs no second direct write.
- Its kind-2 branch writes the converted result before temporary finalization.
- The associated-value path of `0x7698` writes before finalizing its second wrapper.
- `0x7694` and `0x7695` finalize their temporary before obtaining and writing the final result.

Aliasing may affect subsequent behavior. For example, the initial zero write in `0x7697` could change object or global storage later used by lookup. No alias-safety guarantee is established.

Constructor-like and finalizer-like call pairs do not establish heap allocation, ownership transfer, reference counting, or release obligations. No separate release of lookup results is visible. Returned output words may represent scalars, pointers, or handles; their interpretation and lifetime requirements remain unknown.

No arithmetic transformation of contexts, payloads, or conversion results is visible. The relevant numeric operations are fixed layout displacements, selector comparisons, equality tests, and a bit-0 flag test. The kind read is one byte; signedness does not change comparisons with 0, 1, and 2. No overflow-handling behavior is established.

## Dependencies

Names below are opaque identities. Roles describe only their use by this function.

| ID | Observed role |
|---|---|
| F-b255b26284e6 | Gates preparation |
| F-5b24edfb22c0 | Obtains a value from the global resource |
| F-804adc29bbfb | Receives that value and two global arguments |
| F-d08d3ca17b81 | Last preparation call |
| F-ebbd9627a906 | Item lookup using embedded object state and item reference |
| F-d19f8fa90ae4 | Constructs the `0x7694` temporary |
| F-8b3b4cd10074 | Constructs the `0x7695` temporary |
| F-46f0b3cb316e | Processes the 8-byte temporary with the global resource |
| F-289ec4311c64 | Finalizes the 8-byte temporary |
| F-44362605ab79 | Supplies the final resource-derived result |
| F-9d0ecc5f6adf | Supplies the first conversion argument |
| F-ea9442e8ddbb | Converts a token or value |
| F-7d964f24a044 | Supplies the object for the offset-556 method |
| F-6e482452607b | First classification query |
| F-15632527c6ab | Conditionally repeated classification query |
| F-d1b4663968a8 | Intermediate classification query |
| F-e8ce52ce5425 | Later classification query |
| F-4e292a3dd745 | Finalizes the 20-byte temporary |
| F-83300d26ff95 | Constructs a 4-byte wrapper |
| F-552123f40f1e | Gates associated-value processing |
| F-1f4c3d25219d | Obtains an associated value |
| F-561858688f94 | Finalizes a 4-byte wrapper |

Two virtual dependencies are identified by their method-table offsets, 100 and 556; independent opaque function identities are unavailable.

No dependency is assumed to preserve globals, preserve aliased storage, avoid callbacks, or own or release memory.

## Acceptance examples

These are conditional criteria, not executed tests. They assume valid accessible storage and normal dependency completion. Output-at-return claims additionally require that later dependency or concurrent effects do not alter the stated value.

1. **Unsupported selector:** For `0x8000`, return 0 with no direct output write. If preparation leaves an initial `0x12345678` unchanged, that value remains at return.

2. **Delegation:** For `0x7692`, if the offset-100 method returns 7, return 7 rather than normalizing it to 1.

3. **Kind-2 exclusion:** For `0x7696` and kind 2, directly write zero and return 1 without calling either conversion dependency.

4. **Selector conversion:** For `0x7696` and kind 0, the second argument to F-ea9442e8ddbb is `0x7696`. If conversion returns `0x11223344`, directly write that 32-bit value.

5. **Context mismatch:** For `0x7697` and unequal context, directly write zero, return 0, and perform no item lookup.

6. **Other kind with output preservation:** For `0x7697`, matching context, and kind 3, if lookup does not modify the output, the initial zero remains. Return 1 without conversion or another direct write.

7. **Other kind with output mutation:** Under the same selector and context conditions, suppose lookup modifies the output to `0x55667788` and returns a valid item of kind 3. Return 1 without overwriting that value. This behavior is permitted by the established branch.

8. **Immediate classification:** If the first classification query returns nonzero and the early F-15632527c6ab returns zero, select T-A and skip the remaining classification queries.

9. **Repeated observation:** If both early queries return nonzero, followed by zero from F-d1b4663968a8 and F-e8ce52ce5425, call F-15632527c6ab again. A later zero selects T-E.

10. **Token read timing:** If T-B is selected and F-9d0ecc5f6adf changes its global token source, conversion receives the subsequently read value, not an earlier snapshot.

11. **Converted fallback:** For `0x7698` and a non-kind-2 item, if conversion of numeric zero returns 9, directly write 9.

12. **Wrapper lifetime ordering:** For `0x7698`, kind 2, and a nonzero gate result, finalize the first wrapper before the second lookup; write the result before finalizing the second wrapper.

## Uncertainties

**Inferred intent:** Preparation may implement initialization or registration. Conversion may produce presentation values. Kind-2 processing may access richer item metadata. These interpretations are not required by the contract.

**documented_only:**

- Actual global values and domain meanings of selectors, kinds, tokens, payloads, and predicates.
- Dependency implementations, error conventions, and behavior on invalid inputs.
- Whether dependencies mutate output through aliases, callbacks, or shared state.
- Allocation, retention, release, and ownership semantics.
- Cleanup obligations associated with output values.
- Exception, termination, reentrancy, and abnormal-completion behavior.
- Thread safety and consistency under concurrent changes.
- Complete ABI details beyond the represented widths, storage accesses, and method offsets.
- Validity requirements for overlapping storage.
- Whether finalization changes an output already written.

No recovery path is visible for abnormal dependency completion. The normal-return sequences do not establish cleanup guarantees when a dependency fails to return normally.

## Evidence

Established static evidence is limited to the supplied source representation: recognized selectors, 32-bit values, byte-sized kind reads, offsets, conditional call sequences, repeated lookups and reads, temporary storage sizes, explicit writes, and return paths.

The distinction for other kinds in `0x7697` follows directly from ordering: zero is written, item lookup executes, the kind is read, and the branch returns 1 without another direct write. The lookup’s implementation is unavailable, so preservation of that earlier zero is not established.

Embedded names, comments, prior wording, and apparent domain conventions are not treated as authoritative semantics. No dependency implementation or runtime trace was supplied. No runtime tests have been performed; the acceptance examples are conditional specifications only.
