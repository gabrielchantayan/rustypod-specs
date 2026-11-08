<!-- Trusted provenance: F-2c79ceb84ee9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c79ceb84ee9

## Purpose

F-2c79ceb84ee9 selects and executes one of several ordered action sequences. Selection depends on two direct-call predicates, a dispatch-based predicate, a shared 32-bit field, and, on one path, a further dispatch-based predicate.

The established behavior is orchestration: calling dependencies, obtaining objects, resolving dispatch entries, forwarding values, and preparing a small local argument. Every normally completed path returns the 32-bit value 1. That result does not report which action occurred or whether an underlying operation succeeded.

A fixed string reference suggests an association with a now-playing controller. This is inferred intent only. The available evidence does not establish playback, navigation, display changes, controller ownership, or another user-visible effect.

## Interface

### Invocation and return

- No explicit input parameters are shown.
- The return value is 32 bits.
- Every path reaching the final return produces 1.
- Dependency results do not replace or modify that final value.
- Normal completion requires every invoked dependency and indirect target to return normally. Faults, exceptions, nonlocal transfers, and nonreturning calls are outside the established completion behavior.

### Implicit inputs and representation

The function consumes:

- Results from F-f2db0ea1d7cc and F-cf9ff6ecb14d.
- Objects returned by F-b7af80e829ec, F-be45cd43f92d, and F-76b1c50b12f7.
- The first word of each dereferenced object and entries in the dispatch table identified by that word.
- A shared pointer chain leading to a 32-bit field at byte offset 0x3F8.
- A separate global 32-bit value.
- One fixed string reference, passed twice in each invocation of F-a3642b19087c.

The supplied representation uses 32-bit integers, object pointers, dispatch-table addresses, and forwarded scalar values. Dispatch offsets are byte offsets, not entry numbers. This contract does not establish equivalent behavior under a different pointer width or layout.

Zero/nonzero decisions do not require a result to equal 1. Any nonzero 32-bit pattern selects the nonzero branch, including a pattern interpreted as negative under signed interpretation.

### Preconditions

Every executed object dereference requires readable object storage. Its first word must identify a readable dispatch table, and each selected entry must identify a callable target compatible with the shown arguments.

The shared pointer chain and field must be readable when their branch is reached. The fixed string reference and local temporary must be usable according to the requirements of the callees receiving them; those requirements are not supplied.

There are no early null checks protecting the returned objects before their initial dispatch-table accesses. In particular, the later null check on the object returned by F-be45cd43f92d occurs only after that object has already been dereferenced for several calls. It does not make a null provider result safe.

## Behavior

### Ordered direct-call decisions

Execution begins by calling F-93eb550dc612, followed by F-f2db0ea1d7cc.

If F-f2db0ea1d7cc returns nonzero, the function calls F-93eb550dc612 again, calls F-186eb8c5ea0f, and returns 1. No second predicate or dispatch-based decision is reached.

If F-f2db0ea1d7cc returns zero, the function instead calls F-93eb550dc612 again and then F-cf9ff6ecb14d.

If F-cf9ff6ecb14d returns nonzero, the function calls F-93eb550dc612 a third time, calls F-ffb10ad289b0, and returns 1.

Only when both direct predicates return zero does execution proceed to dispatch-based selection. The first predicate therefore has priority over the second. Results from F-93eb550dc612 are not consumed by the displayed function; its effects remain unknown.

### Initial dispatch-based decision

After both direct predicates return zero, the function calls F-b7af80e829ec. It uses the returned object’s first 32-bit word to resolve dispatch-table entry 0x120 and invokes that target.

No explicit arguments, including an explicit receiver, are shown for entry 0x120. Its result is interpreted as a 32-bit zero/nonzero predicate.

Later occurrences of F-b7af80e829ec are separate calls. The function does not reuse this initial result for those later operations. Neither object identity nor dispatch-table identity is assumed to remain constant between provider calls.

### Entry 0x120 returns zero

The function performs the following sequence:

1. Call F-d260cfbdb3c8.
2. Call F-ef0964bd9b8a and retain its 32-bit result.
3. Call F-a3642b19087c with that result, the fixed string reference, and the same string reference again.
4. Retain the 32-bit result of F-a3642b19087c.
5. Call F-76b1c50b12f7.
6. Resolve entry 0x04 from the returned object and invoke it with the object as the first explicit argument and the retained result as the second.
7. Call F-b7af80e829ec again.
8. Resolve entry 0xAC from this newly returned object and invoke it with no explicit arguments.
9. Return 1.

The two string arguments are identical references, not merely independently supplied strings with equal contents.

The result of F-a3642b19087c is forwarded without an intervening validity, null, or error check. Its concrete meaning is not established.

This path does not directly read the shared field at offset 0x3F8 or the separate global value used for the local temporary.

### Entry 0x120 returns nonzero

The function follows the shared pointer chain and reads the 32-bit field at byte offset 0x3F8 from the resulting object address.

If the field is zero, execution returns 1 without calling F-be45cd43f92d or entering either fixed-string sequence.

If the field is nonzero, execution calls F-be45cd43f92d with the integer argument 0 and retains the returned object. It then performs these indirect calls, in order:

| Dispatch-table byte offset | Explicit arguments | Result use |
|---|---|---|
| 0x38 | Retained object | Not consumed |
| 0xD8 | Retained object | Not consumed |
| 0x5C | Retained object, 1, 0 | Not consumed |
| 0x108 | Retained object | Tested for zero/nonzero |

For each call, the target is resolved through the retained object’s first word. Earlier calls must not be assumed to preserve that word or the selected table entries. The retained object pointer is reused, but dispatch resolution is repeated.

If entry 0x108 returns zero, execution returns 1. There is no temporary submission, no entry-0x1C call, no entry-0x28 call, and no fixed-string sequence on this path.

### Entry 0x108 returns nonzero

The function reads the separate global 32-bit value into local storage, then writes a one-byte zero immediately following that four-byte value in the displayed local layout.

The argument storage therefore has:

- Relative bytes 0 through 3: the copied 32-bit value.
- Relative byte 4: zero.

No larger initialized structure, padding contents, string encoding, or permitted callee read extent is established. The evidence does not independently establish byte order.

After these writes, the function calls F-b7af80e829ec and resolves entry 0x20 from the returned object. It invokes that target with seven explicit arguments, in order:

1. The newly returned object.
2. The address of the leading four-byte local value.
3. The 32-bit bit pattern 0xFFFFFFFF.
4. 0.
5. 3.
6. 3.
7. The retained object previously returned by F-be45cd43f92d.

The bit pattern 0xFFFFFFFF is all ones at 32-bit width. It is −1 under signed interpretation and 4,294,967,295 under unsigned interpretation. The evidence does not establish a length, timeout, index, or sentinel interpretation.

After entry 0x20 returns, execution explicitly tests the retained F-be45cd43f92d object pointer for null. If nonnull, it resolves that object’s entry 0x1C and invokes it with the object as the explicit argument. If null, only that indirect call is skipped.

This null check is distinct from the four call-result predicates. Its placement does not protect the earlier dereferences of the same retained object. No ordinary successful path from an initially null provider result is established.

Execution then:

1. Calls F-b7af80e829ec again.
2. Resolves entry 0x28 from the newly returned object.
3. Invokes entry 0x28 with that object and 0.
4. Calls F-d260cfbdb3c8.
5. Calls F-ef0964bd9b8a.
6. Calls F-a3642b19087c with the preceding result and the same fixed string reference twice.
7. Calls F-76b1c50b12f7.
8. Invokes the returned object’s entry 0x04 with that object and the result from F-a3642b19087c.
9. Returns 1.

Entry 0xAC is absent from this path.

## Side effects

### Observable reads and writes

Direct reads include object first words, selected dispatch entries, the conditional shared pointer chain and field, and the conditional separate global value.

Direct writes shown by the function are confined to the local four-byte copy and following zero byte. No explicit write to a global location or object field is shown.

These statements do not constrain dependency effects. A direct or indirect callee may change shared storage, object contents, dispatch tables, or external resources. No such preservation guarantee is supplied.

The shared field is read only after entry 0x120 returns nonzero. The separate global value is read only after entry 0x108 returns nonzero. Neither is an entry-time snapshot; preceding calls may affect the value eventually observed.

The shared field is tested once on the shown path. Its later value does not cause another direct check within this function.

### Ownership and lifetime

No direct allocation or deallocation is shown. Returned objects and forwarded 32-bit values have no established ownership classification.

Entry 0x1C is not proven to release an object. Entry 0x20 is not proven to retain, borrow, consume, or transfer ownership of its object argument. The absence of entry 0x1C when entry 0x108 returns zero is established control flow, not proof of either correct ownership or a leak.

The temporary address passed to entry 0x20 identifies local storage belonging to this invocation. Its lifetime does not extend beyond the invocation. Whether the callee copies the data, uses it synchronously, or retains the address is unknown.

### Error handling and decision coverage

There is no explicit error return, retry, rollback, or exception handler.

Four call results control zero/nonzero decisions:

- F-f2db0ea1d7cc.
- F-cf9ff6ecb14d.
- Dispatch entry 0x120.
- Dispatch entry 0x108.

Execution also tests the shared 32-bit field and explicitly checks the retained F-be45cd43f92d object pointer for null before entry 0x1C.

Other results are not uniformly ignored or merely forwarded. Provider results supply objects that are dereferenced and used for dispatch; F-be45cd43f92d’s result is also retained, passed onward, and null-checked. F-ef0964bd9b8a and F-a3642b19087c supply forwarded values. Results with no displayed consumer do not influence this function’s explicit decisions, although their callees may have side effects.

None of these tests is independently established as an error test. A zero or nonzero value has only the branch meaning described above.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-93eb550dc612 | Called before each reached direct predicate and again before either direct priority action; result unused. |
| F-f2db0ea1d7cc | First 32-bit zero/nonzero predicate. |
| F-186eb8c5ea0f | Action selected when the first predicate is nonzero. |
| F-cf9ff6ecb14d | Second 32-bit zero/nonzero predicate, reached only after the first returns zero. |
| F-ffb10ad289b0 | Action selected when the second predicate is nonzero. |
| F-b7af80e829ec | Supplies separately obtained objects for entries 0x120, 0xAC, 0x20, and 0x28. |
| F-be45cd43f92d | Receives 0 and supplies the retained object used for entries 0x38, 0xD8, 0x5C, 0x108, and conditionally 0x1C. |
| F-d260cfbdb3c8 | Begins each fixed-string sequence. |
| F-ef0964bd9b8a | Supplies the first argument to F-a3642b19087c. |
| F-a3642b19087c | Receives a prior result and two identical string references; returns a forwarded 32-bit value. |
| F-76b1c50b12f7 | Supplies the object whose entry 0x04 receives that forwarded value. |

No opaque IDs are supplied for the indirect targets. Dispatch offsets identify call roles without establishing target identity, semantics, or ownership behavior.

## Acceptance examples

These are conditional acceptance examples, not executed tests. Each assumes readable required storage, compatible call targets, and normal completion of invoked dependencies.

1. **First predicate nonzero.**  
   The call order is F-93eb550dc612, F-f2db0ea1d7cc, F-93eb550dc612, F-186eb8c5ea0f. F-cf9ff6ecb14d is not called. The return is 1. A nonzero value other than 1 selects the same path.

2. **First predicate zero, second nonzero.**  
   F-93eb550dc612 occurs three times. F-ffb10ad289b0 is selected. No dispatch-based decision is reached, and the return is 1.

3. **Both direct predicates zero; entry 0x120 zero.**  
   The fixed-string sequence completes through entry 0x04, followed by a fresh F-b7af80e829ec result and entry 0xAC. The shared field is not directly read.

4. **Entry 0x120 nonzero; shared field zero.**  
   The field read occurs, but F-be45cd43f92d and both fixed-string sequences are absent. The return remains 1.

5. **Shared field nonzero; entry 0x108 zero.**  
   Entries 0x38, 0xD8, 0x5C, and 0x108 execute in that order on the retained object. No submission through entry 0x20 or subsequent entry 0x1C occurs.

6. **Entry 0x108 nonzero; retained object valid and nonnull.**  
   The global value and following zero byte are prepared before the fresh provider call for entry 0x20. Entry 0x1C occurs after submission and before the fresh provider call for entry 0x28. The fixed-string sequence follows entry 0x28. Entry 0xAC is absent.

7. **Provider identities differ.**  
   If successive F-b7af80e829ec calls return different valid objects, each operation uses its own call’s result. Reusing the initial object for later dispatch would violate the contract.

8. **An earlier dependency changes the separate global value.**  
   If the changed value remains in place until the read following entry 0x108, that value is copied into the temporary. An entry-time cached value would not satisfy the stated ordering.

9. **F-be45cd43f92d returns null.**  
   Normal completion is not guaranteed: the function attempts dispatch-table access before reaching its explicit null check. The late check cannot be treated as general null-result handling.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the predicates, shared field, global value, and indirect methods.
- Exact fixed-string bytes and encoding.
- Concrete types and validity rules for forwarded 32-bit values.
- Ownership, reference counting, allocation, release, and retention behavior.
- Callee preservation of globals, objects, and dispatch tables.
- Complete temporary layout, byte order, and permitted read extent.
- ABI details for indirect calls without explicit receiver arguments.
- Synchronization, concurrent mutation, reentrancy, and thread safety.
- Behavior after invalid memory access, incompatible calls, exceptions, or nonreturning dependencies.
- Any media, navigation, display, persistence, or other user-visible effect.

## Evidence

**Established static evidence:** The supplied function representation supports the ordered calls, four call-result predicates, shared-field test, retained-pointer null check, dispatch offsets, explicit arguments, conditional reads, local writes, and final 32-bit return value described here.

**Inferred intent:** Symbolic labeling of the fixed string suggests a now-playing controller association. That interpretation is not an acceptance requirement and does not establish dependency effects.

**Verification limit:** No runtime tests have been performed. Acceptance examples express conditional behavior supported by static evidence, not observed execution.
