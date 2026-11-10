<!-- Trusted provenance: F-b63ff4e1cea7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b63ff4e1cea7

## Purpose

F-b63ff4e1cea7 dispatches a tagged input record against a mutable receiver. Recognized routes invoke receiver operations, construct and submit an auxiliary value, query a related object, or forward input payload words. Unrecognized records reach fallback handling.

On normal completion, the function returns a 32-bit value:

- **1** after completing a recognized route.
- **0** when fallback recognizes neither supported fallback tag.

A return of 1 indicates completion of the selected dispatch sequence, not success of its dependencies. The function does not inspect dependency results for success or propagate a dependency-specific error.

Two global values control the dispatch partition. Their values are unavailable, so literal tags are recognized only under the corresponding boundary conditions. Boundary equality takes precedence over literal-tag handling.

## Interface

### Arguments and result

| Item | Established representation |
|---|---|
| Receiver | Address of a mutable object accessed through 32-bit fields and address-bearing words |
| Input record | Address of storage containing a tag and route-dependent payload |
| Return value | 32-bit value, either 0 or 1 on normal return |

The visible layout uses four-byte words and four-byte address-bearing fields. Compatibility with another pointer width is not established.

### Input-record layout

| Byte offset | Width | Use |
|---|---:|---|
| +0 | Not accessed directly | Unspecified |
| +4 | 32 bits | Dispatch tag |
| +8 | 32 bits | First payload word |
| +12 | 32 bits | Second payload word |
| +16 | 32 bits | Third payload word |

Ordered tag comparisons use signed 32-bit interpretation. Equality and payload forwarding preserve the corresponding 32-bit values.

Payload words have no established semantic type beyond their width. They may represent integers, addresses, handles, or other data.

### Receiver layout accessed directly

| Byte offset | Width | Use |
|---|---:|---|
| +0 | 32-bit address-bearing word | Receiver dispatch-table pointer |
| +132 | 32-bit address-bearing word | Optional related-object pointer |
| +192 | 32 bits | Compared with input payload at +8 |
| +952 | 32 bits | Compared with a query result and conditionally overwritten |

Receiver dispatch-table entries are accessed at byte offsets +88, +92, and +120. The related object's dispatch-table entry is accessed at byte offset +92.

### Global roles

These names identify distinct storage roles, not known meanings or numeric values:

| Role | Use |
|---|---|
| Boundary A | First equality and signed partition boundary |
| Boundary B | Equality and signed partition boundary within the below-A region |
| Entry-selector global | Read at function entry; also read afresh for selected operations |
| Descriptors P, Q, R, S | Construction inputs for `0x70001`, `0x80009`/`0x8000a`, `0x80011`, and Boundary A equality, respectively |
| Initial selector; initial argument | First receiver operation on `0x70001` |
| Refresh-selector global | Captured after the related-object query and read again for the first refresh operation |
| First refresh argument; second refresh argument | Arguments for the two refresh operations |
| Optional-flag pointer | Global pointer used to read a controlling byte |
| Optional selector; optional argument | Final conditional receiver operation on `0x70001` |
| D-first argument; D-final argument | Distinct argument globals for the two operations on `0x8000d` |
| F-first argument; F-middle argument; F-final argument | Distinct argument globals for the three operations on `0x8000f` and `0x80010` |

**D-final argument and F-final argument come from different global locations.** Their equality is not established.

### Preconditions

For the selected route, accessed storage must remain valid and readable; the cache field must also be writable when updated. Dispatch-table entries must identify compatible callable targets.

Neither top-level argument is checked for null. Construction results, dispatch-table pointers, and the optional-flag pointer are not validated. A non-null related-object pointer is checked only for non-nullness, not for validity.

The function provides no ownership or lifetime protection across calls. Continued validity of storage after a dependency returns is therefore a precondition for subsequent accesses, not a guarantee supplied by this function.

## Behavior

### Entry capture and dispatch precedence

Before reading the input tag, the function reads and retains the Entry-selector global. This entry-captured value remains the selector used by the later operations specifically identified below; subsequent global changes do not replace that captured value.

The initial tag determines the main dispatch:

1. Equality with Boundary A selects the Boundary A construction route.
2. Otherwise, a signed-less-than comparison against Boundary A selects the lower partition:
   - Equality with Boundary B invokes F-fb096c39be5b.
   - Otherwise, signed-less-than Boundary B selects the `0x0f`/`0x70001` region.
   - Otherwise, the function considers `0x80009` and `0x8000a`.
3. Otherwise, the function considers the upper numeric switch routes.
4. Unmatched cases enter fallback.

The source refers to the boundary globals at successive tests rather than explicitly retaining one snapshot of each. Descriptions such as “below both boundaries” assume those globals remain stable during dispatch. If they change between accesses, the actual sequence of equality and ordered-comparison results governs routing.

Boundary A equality precedes every literal case. Boundary B equality is considered only after entering the lower partition.

### Boundary A equality

The function performs this sequence:

1. Invoke F-483a717208e6 with the 32-bit argument 8.
2. Read descriptor S and pass it, together with the preceding result, to F-17c2f3b940d8.
3. Invoke F-8e50f265e4fa with the receiver, the constructed result, and a final argument of zero.
4. Return 1.

The descriptor is obtained after F-483a717208e6 returns. The intermediate results are not checked before being passed onward.

### Boundary B equality

When the lower partition is selected and the tag equals Boundary B, the function invokes F-fb096c39be5b with the receiver and returns 1.

Its result, if any, does not affect the visible return value.

### Region below both boundaries

With stable boundaries, the following literal routes require the initial tag to be signed-less-than both boundaries and not intercepted by equality.

#### Tag `0x0f`

The function compares input +8 with receiver +192 as 32-bit values.

- Equal: invoke F-2da1de88f247 with the receiver, then return 1.
- Unequal: enter fallback.

Neither compared field is directly written by this comparison.

#### Tag `0x70001`: construction and initial operation

The function:

1. Invokes F-483a717208e6 with 8.
2. Reads descriptor P and invokes F-17c2f3b940d8 with the preceding result and that descriptor.
3. Submits the constructed result through F-8e50f265e4fa with the receiver and zero.
4. Invokes receiver operation F-d88 with the Initial selector and Initial argument.
5. Checks the related-object pointer at receiver +132.

The initial operation uses its own selector global, not the entry-captured selector.

#### Tag `0x70001`: related-object query and refresh

If receiver +132 is null, the query and refresh processing are skipped.

If it is non-null, the function resolves the related object's F-q92 operation and invokes it. The visible call has no explicit arguments. Its result is treated as a 32-bit value.

After the query returns, the function:

1. Captures the Refresh-selector global.
2. Compares receiver +952 against the query result.

The selector capture occurs even when the comparison finds no difference.

If the cache equals the query result, neither refresh operation nor the direct cache write occurs.

If they differ:

1. Invoke receiver operation F-d88 using a **fresh read** of the Refresh-selector global and the First refresh argument.
2. Resolve F-d88 again and invoke it using the **previously captured** refresh selector and the Second refresh argument.
3. Write the retained query result to receiver +952.

The two refresh selectors need not be equal. In particular, a change to the global after capture can affect the first refresh operation without changing the selector retained for the second.

The cache comparison follows the query, so query-induced changes to receiver +952 can affect whether refresh occurs. The final cache write follows both refresh operations and uses the retained query result, not a second query. If a refresh operation changes the cache, the final direct write overwrites that change upon normal continuation.

The related-object pointer is referenced again to resolve its operation after the non-null check; the function does not establish a retained object lifetime or an immutable pointer snapshot.

#### Tag `0x70001`: final optional operation

After related-object processing, the function reads the byte addressed by the Optional-flag pointer.

- Zero: return 1.
- Nonzero: resolve receiver operation F-d88, read the Optional selector, read the Optional argument, invoke the resolved operation with the receiver and those values, then return 1.

This check is reached whether the related object was null, the cache matched, or refresh completed.

### Region below Boundary A but above Boundary B

After excluding Boundary B equality, this region recognizes `0x80009` and `0x8000a`.

For either tag:

1. Invoke F-483a717208e6 with 8.
2. Read descriptor Q and construct through F-17c2f3b940d8.
3. Invoke F-f74ca80f10de with the receiver.
4. Submit the retained constructed result through F-8e50f265e4fa with the receiver and zero.
5. Return 1.

The constructed result is retained across F-f74ca80f10de. That dependency is not assumed to preserve receiver state, globals, or object lifetimes.

Recognition of `0x8000a` uses a 32-bit subtraction of `0x80000` followed by equality with decimal 10. Together with the separate `0x80009` check, the accepted 32-bit tag patterns are exactly those two values within this dispatch region. This arithmetic does not compute a size or payload length.

Other tags enter fallback.

### Upper numeric switch routes

With stable Boundary A, these routes require the initial tag to be signed-greater-than it.

| Tag | Selected behavior |
|---|---|
| `0x8000c` | Invoke F-b8f47c5be3a1 with the receiver; return 1 |
| `0x8000d` | Perform the two-operation sequence below; return 1 |
| `0x8000f`, `0x80010` | Perform the three-operation sequence below; return 1 |
| `0x80011` | Construct with descriptor R, submit, and return 1 |
| Other values, including `0x8000e` | Enter fallback |

For `0x8000d`:

1. Invoke F-d88 with a fresh read of the Entry-selector global and D-first argument.
2. Resolve F-d88 again.
3. Read D-final argument.
4. Invoke the resolved operation with the entry-captured selector and D-final argument.

For `0x8000f` and `0x80010`:

1. Invoke F-d88 with a fresh read of the Entry-selector global and F-first argument.
2. Invoke F-d88 with the entry-captured selector and F-middle argument.
3. Resolve F-d88 again.
4. Read F-final argument.
5. Invoke the resolved operation with the entry-captured selector and F-final argument.

Each operation is resolved separately. D-final argument is not an alias established for F-final argument; they are separate global inputs.

For `0x80011`, F-483a717208e6 receives 8. After it returns, descriptor R is read and supplied with its result to F-17c2f3b940d8. The constructed result is submitted through F-8e50f265e4fa with the receiver and zero, then the function returns 1.

### Fallback

Fallback reads input +4 again:

- If this read equals `0x14`, invoke F-d120 with the receiver and the three 32-bit payload words at +8, +12, and +16, then return 1.
- Otherwise, read input +4 again. If this later read equals `0x13`, invoke F-d92 with the receiver and the two payload words at +8 and +12, then return 1.
- Otherwise, return 0.

These are separate tag reads, not tests solely against the entry tag. A changing record can therefore produce results that a stable-tag description would not predict.

The payload positions determine argument positions. The supplied evidence does not establish a finer ordering among loads within a single call expression.

## Side effects

The only direct receiver-field write established in the body is the conditional 32-bit write at +952 after both refresh operations.

All dependencies and indirect operations may have additional effects. The function supplies no general guarantee that they preserve globals, dispatch tables, receiver fields, input storage, or associated object lifetimes.

Successive receiver calls resolve their operation targets separately. A preceding call that changes the receiver's dispatch table can therefore affect a later target.

No explicit release operation appears for the receiver, record, related object, or construction results. This does not establish ownership transfer, borrowing, retention, or absence of release inside dependencies.

No synchronization, rollback, retry, or explicit exception handling is visible. If a dependency does not return normally, later operations and the final return are not reached.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-483a717208e6 | Receives 8; produces the first construction-stage result |
| F-17c2f3b940d8 | Receives that result and a route-specific descriptor; produces the submitted value |
| F-8e50f265e4fa | Receives receiver, constructed value, and zero |
| F-fb096c39be5b | Receiver call on reachable Boundary B equality |
| F-f74ca80f10de | Receiver call between construction and submission for `0x80009`/`0x8000a` |
| F-b8f47c5be3a1 | Receiver call for reachable `0x8000c` |
| F-2da1de88f247 | Receiver call for reachable matching `0x0f` |
| F-d88 | Contract-local ID for receiver operation at dispatch-table byte offset +88 |
| F-d92 | Contract-local ID for receiver operation at dispatch-table byte offset +92 |
| F-d120 | Contract-local ID for receiver operation at dispatch-table byte offset +120 |
| F-q92 | Contract-local ID for related-object operation at dispatch-table byte offset +92 |

The indirect-operation IDs label call-site roles, not fixed runtime identities. F-d92 and F-q92 are not established to be the same function.

## Acceptance examples

These are conditional static criteria, not executed tests. Dependencies must return normally and required storage must remain valid. Unless a change is explicitly described, dispatch boundaries and input tags remain stable.

1. **Boundary equality overrides a literal case.**  
   If the tag equals Boundary A, descriptor S is used and the result is submitted with zero, even when that numeric tag also matches a listed literal route.

2. **Matching and nonmatching `0x0f`.**  
   When `0x0f` reaches its lower-region route, equal words at input +8 and receiver +192 cause F-2da1de88f247 and return 1. Unequal words cause fallback, which returns 0 for the unchanged `0x0f` tag.

3. **Refresh uses distinct selector reads.**  
   On reachable `0x70001`, suppose the query result is 27 and the post-query cache is 12. If the refresh selector is captured as 5 but its fresh read supplies 9, the first refresh receives selector 9 and the second receives selector 5. Only after both return is 27 written to the cache.

4. **First refresh cannot replace the captured second selector.**  
   If both reads initially yield 5 and the first refresh changes the Refresh-selector global to 11, the second refresh still receives 5.

5. **An equal cache still follows selector capture.**  
   If the query result equals the post-query cache, the refresh selector is captured before the comparison, but no refresh operation or direct cache write follows. The optional-flag check still occurs.

6. **Null related object skips only query and refresh.**  
   Construction, submission, and the initial receiver operation still occur. No related-object operation is attempted; optional-flag processing remains.

7. **The two final-argument globals remain distinct.**  
   If D-final argument is 31 and F-final argument is 47, reachable `0x8000d` supplies 31 to its second operation. Reachable `0x8000f` and `0x80010` supply 47 to their third operation.

8. **Entry-captured selector survives callback changes.**  
   On reachable `0x8000f`, changing the Entry-selector global during the first operation does not change the selector supplied to the second or third operation.

9. **Fallback forwards exact payload words.**  
   A stable fallback tag of `0x14` forwards all three payload words to F-d120 unchanged. A stable fallback tag of `0x13` forwards the first two to F-d92. Both return 1.

10. **Fallback uses its later tag read.**  
    If the first fallback read is not `0x14`, but the second reads `0x13`, F-d92 is selected. If neither fallback comparison matches, the result is 0.

## Uncertainties

The following remain **documented_only**:

- Boundary values, their ordering, and unconditional reachability of literal routes.
- Descriptor, selector, and argument meanings and values.
- Whether the argument 8 denotes an allocation size or something else.
- Construction failure representations and dependency-specific error behavior.
- Ownership, retention, release, and lifetime effects of dependencies.
- The related-object operation's implicit arguments and full calling convention.
- Validity of replacing or destroying receiver-associated objects during callbacks.
- Defined behavior for invalid pointers or concurrent mutation.
- Fine-grained ordering of reads within expressions where the supplied evidence does not establish it.

**Inferred intent:** the function behaves as a dispatcher, and receiver +952 behaves as a cache of the related-object query result. These descriptions do not establish a broader protocol.

## Evidence

Established behavior derives solely from the supplied static function body: 32-bit field accesses, signed dispatch comparisons, separate selector reads and captures, distinct argument-global locations, dependency ordering, fallback rereads, the conditional cache write, and literal return values.

The post-query refresh-selector capture and the first refresh operation's fresh selector read are separate source accesses. The final arguments for `0x8000d` and `0x8000f`/`0x80010` likewise come from separate global locations.

No dependency implementations, global definitions, runtime traces, or independent type information were supplied. No runtime tests have been performed.
