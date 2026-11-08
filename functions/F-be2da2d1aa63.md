<!-- Trusted provenance: F-be2da2d1aa63; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be2da2d1aa63

## Purpose

F-be2da2d1aa63 transfers a four-word descriptor into a selected object, invokes two preparation operations on that object, derives an integer selection value, and passes that value to two further object operations.

The selection value initially comes from the descriptor. If the descriptor contains a nonzero search key, the function searches a signed, half-open index range for the first matching value. A match replaces the initial selection with either the matching index or that index plus a nonnegative descriptor adjustment.

This purpose describes established data flow. Interpreting the object as a particular application entity, the descriptor as a named configuration type, or the selection as a particular user-interface position would be inferred intent, not established behavior.

The function has three ordinary early-return conditions: an initial indirect operation returns zero, the second input is zero, or an initial object lookup returns zero. Two later failures invoke F-a817d8d58c10, which is represented as nonreturning.

## Interface

### Inputs

The visible interface has three inputs:

| Input | Width and representation | Established use |
|---|---|---|
| Context | 32-bit address-bearing value | Used to reach an indirect operation and passed to descriptor lookup. |
| Request | 32-bit value | Tested against zero and passed to descriptor lookup. |
| Qualifier | Opaque 32-bit word | Passed to the first descriptor lookup; replaced by zero for a fallback lookup. |

The context is not checked for zero before being dereferenced. A zero request does not prevent the initial context-based indirect operation.

The evidence uses 32-bit address and integer representations. Fields described below as signed are used in signed comparisons. Other words may be addresses, identifiers, or opaque values; their semantic types are not established.

### Output

There is no explicit return value. On ordinary completion, the observable result consists of memory writes and dependency calls. The result of the final indirect operation is not exposed by the visible interface.

### Referenced layouts

The context must support reading a 32-bit address at byte offset `0x34`. The referenced object must, in turn, support reading a dispatch-table address from byte offset zero and resolving a callable entry at dispatch-table byte offset `0xE4`.

The selected object returned by F-682adfd5a561 is accessed through this layout:

| Byte offset | Width | Role in this function |
|---|---|---|
| `0x00` | 32 bits | Dispatch-table address. |
| `0x38` | 32 bits | Input to a conditional secondary lookup. |
| `0x100` | 32 bits | Initial search index. |
| `0x104` | 32 bits | Search upper bound, reread for each loop condition. |
| `0x108` | 32 bits | Destination for descriptor word 0. |
| `0x10C` | 32 bits | Destination for descriptor word 1. |
| `0x110` | 32 bits | Destination for descriptor word 2. |
| `0x114` | 32 bits | Destination for descriptor word 3. |

The descriptor returned by F-f7490d12dfa0 must support four consecutive 32-bit word reads:

| Byte offset | Role |
|---|---|
| `0x00` | Payload used in a preparation call and copied into the selected object. |
| `0x04` | Search key; zero disables searching. |
| `0x08` | Initial selection value. |
| `0x0C` | Signed adjustment applied on a match only when nonnegative. |

These are access requirements, not evidence of complete object sizes or allocation boundaries.

### Preconditions

Every pointer dereferenced on the executed path must identify accessible storage of the required extent. Every resolved dispatch entry must be callable with the observed argument shape. The returned descriptor and selected object must remain usable across intervening calls.

The function does not establish these conditions through validation. It also does not establish that dependencies preserve descriptor contents, object fields, dispatch tables, or shared state.

## Behavior

### 1. Initial gate and selected-object lookup

The function first follows the context’s reference at byte offset `0x34`, reads that referenced object’s dispatch table, and invokes its entry at byte offset `0xE4`.

No explicit arguments are visible for this initial indirect invocation. A receiver convention or other implicit arguments cannot be established from the supplied representation.

If the invocation returns zero, the function returns immediately. Otherwise, it tests the request input. If the request is zero, it returns immediately.

Consequently, a zero request still permits all effects of the initial indirect invocation.

For a nonzero request and nonzero initial result, the function calls F-682adfd5a561 with:

- The initial indirect result.
- The constant `0x1700`.

If this lookup returns zero, the function returns without requesting a descriptor or writing the selected-object fields.

### 2. Descriptor lookup and fallback

With a nonzero selected object, the function calls F-f7490d12dfa0 with the ordered inputs:

1. Context.
2. Request.
3. Qualifier.
4. Zero.

If that call returns zero, it calls the same dependency again with context, request, zero, and zero.

The fallback is triggered by a zero result, not by the qualifier’s value. Thus, if the original qualifier is already zero and the first call fails, the second call still occurs with the same visible argument values.

If both calls return zero, the function invokes F-a817d8d58c10. The supplied static representation marks this dependency as nonreturning. No recoverable error value is returned by F-be2da2d1aa63 on that path.

### 3. Descriptor transfer

After obtaining a nonzero descriptor address, the function reads descriptor words 1, 2, and 3, in that order.

It then reads descriptor word 0 and stores it into selected-object offset `0x108`. The previously read words are stored into offsets `0x10C`, `0x110`, and `0x114`, respectively, in that order.

This is not an established atomic snapshot or atomic four-word update. The read order and write order matter if the descriptor aliases destination storage or if storage changes during the operation. In particular, words 1–3 are obtained before the first destination write, while word 0 is read immediately before its destination write.

### 4. Preparation operations

After the four stores, the function invokes the selected object’s dispatch entry at byte offset `0x15C`, passing the selected-object address.

It next invokes the selected object’s dispatch entry at byte offset `0x1A0`, passing:

1. The selected-object address.
2. A newly read descriptor word 0.

The payload supplied to this second operation is not necessarily the value previously copied to offset `0x108`: the first preparation operation may have changed descriptor storage.

The dispatch table is resolved again for subsequent indirect operations. The evidence does not support caching the original dispatch table across calls.

### 5. Initial selection and optional search

After both preparation operations, descriptor word 2 is reread to initialize the selection value. Descriptor word 1 is then read and tested against zero.

If the key is zero, the search is skipped. The initial selection proceeds directly to the final operations.

If the key is nonzero, the function reads selected-object offset `0x38` and calls F-682adfd5a561 with that value and the constant `6`.

Immediately after this lookup returns, two shared 32-bit words are read. The word used as the second shared argument to F-9dfe5f6048f1 is read first; the word used as its first shared argument is read second. These reads occur before the secondary lookup result is tested for zero.

If the secondary lookup result is zero, F-a817d8d58c10 is invoked. No final selection operation occurs on the represented nonreturning path.

Otherwise, the function reads the selected object’s field at `0x100` once to obtain the initial index. It compares that index, as signed, with the field at `0x104`. The upper-bound field is reread at each loop condition.

For each index below the current signed upper bound, F-9dfe5f6048f1 is invoked with:

- The secondary lookup result.
- The two previously captured shared words, in their argument order.
- The current index.
- Five additional carried values whose origin and meaning are unresolved.

After each invocation, descriptor word 1 is reread and compared with the dependency result. Equality selects that iteration as the first match and stops further searching.

On a match:

- The selection first becomes the matching index.
- Descriptor word 3 is read and interpreted as signed.
- If that word is nonnegative, the selection becomes the index plus the adjustment read for the addition.
- If that word is negative, the selection remains the matching index.

The supplied expression accesses the adjustment for its sign test and again for the addition on the nonnegative path. A stable adjustment is therefore a prerequisite for treating this as one fixed descriptor value.

Without a match, the selection remains the descriptor word 2 value captured before entering the search branch. If the initial index is not below the initial upper bound, no search invocation occurs.

### 6. Final operations

The selected object’s dispatch entry at byte offset `0x190` is invoked with:

1. The selected-object address.
2. The selection value.
3. The constant `1`.

Next, its dispatch entry at byte offset `0x174` is invoked with the selected-object address and the same selection value.

The first final operation may change object state or the dispatch table used to resolve the second. It does not cause the visible function to recompute the selection.

The final dispatch is represented with an indirect-control-flow recovery warning. The established observation is the dispatch target and its visible arguments; the precise underlying call-versus-tail-transfer mechanics are unresolved.

## Side effects

Established direct effects are:

- Four ordered 32-bit writes at selected-object offsets `0x108` through `0x114`.
- Reads through the context, selected object, descriptor, and dispatch tables.
- Conditional reads of two shared words.
- Calls to the dependencies and indirect operations described above.

No direct allocation, deallocation, reference-count update, or release is visible. This does not establish that dependencies perform none of those actions.

The function neither visibly releases the descriptor nor visibly releases either object returned by F-682adfd5a561. Ownership transfer, borrowing duration, and dependency-side lifetime management are documented_only.

Effects are not transactional. A later failure can occur after the descriptor fields have been written and both preparation operations have executed. No rollback is visible.

Dependency mutation matters:

- Preparation operations can affect descriptor values subsequently reread.
- The secondary lookup can affect the shared words captured after it returns.
- Search calls can affect the key reread after each call or the upper bound reread at the next condition.
- A changing upper bound can extend or shorten the search.
- Dependencies may invalidate storage; the visible function provides no protection against that.

## Dependencies

| Opaque dependency | Observed contract at this call site |
|---|---|
| F-682adfd5a561 | Called first with the initial indirect result and `0x1700`; zero causes an ordinary return. Conditionally called later with selected-object offset `0x38` and `6`; zero then causes a nonreturning failure invocation. |
| F-f7490d12dfa0 | Returns an address used as a four-word descriptor. A zero result triggers one fallback call with a zero qualifier. Two zero results trigger F-a817d8d58c10. |
| F-a817d8d58c10 | Invoked for descriptor exhaustion or failed secondary lookup. Marked nonreturning by the supplied static representation; termination mechanism and other effects are unknown. |
| F-9dfe5f6048f1 | Called once per visited search index. Its result is compared with a freshly read descriptor key. Five additional inputs are unresolved. |

Indirect dependencies are identified by dispatch-table byte offsets rather than invented function identities: `0xE4` for the initial operation, and `0x15C`, `0x1A0`, `0x190`, and `0x174` for selected-object operations.

No dependency is assumed pure, globally preserving, or responsible for releasing memory.

## Acceptance examples

These examples are conditional behavioral checks, not executed tests. Stable storage and normally returning dependencies are assumed except where stated.

1. **Zero request with a valid context.** The initial indirect operation is still invoked. After it returns, the function returns without calling F-682adfd5a561, regardless of whether that initial result is zero or nonzero.

2. **Initial selected-object lookup fails.** With a nonzero initial result and nonzero request, F-682adfd5a561 receives the initial result and `0x1700`. If it returns zero, no descriptor lookup or direct descriptor-field write follows.

3. **Fallback succeeds.** The first descriptor lookup returns zero; the zero-qualifier lookup returns a descriptor. Exactly those two descriptor calls occur, and processing uses the second result.

4. **Both descriptor calls fail.** F-a817d8d58c10 is invoked before descriptor transfer or preparation operations. If the qualifier was already zero, both attempted lookups nevertheless occur.

5. **Search disabled.** The descriptor words are `[23, 0, 12, 5]` and remain unchanged. After transfer and preparation, no secondary lookup or search call occurs. The final operations receive selection `12`, first with the additional argument `1`, then without it.

6. **Match with a negative adjustment.** The stable descriptor key is nonzero, the search range is `[4, 8)`, and results first match at index `6`. With adjustment `−1`, calls visit indices `4`, `5`, and `6`; the final selection is `6`.

7. **Match with a nonnegative adjustment.** Under the preceding conditions, adjustment `3` produces selection `9`. There is no visible clamp to the search range.

8. **Empty signed range.** With start `8` and upper bound `4`, the secondary lookup and shared-word reads still occur for a nonzero key, but no search call occurs. The previously captured initial selection is retained.

9. **Descriptor mutation during preparation.** If the first preparation operation changes descriptor word 0, the second preparation operation receives the changed value. The earlier destination write is not automatically refreshed.

10. **Secondary lookup failure.** The four writes and both preparation operations have already occurred. The two shared words are read after the failed lookup returns, then F-a817d8d58c10 is invoked. No rollback or final selection dispatch is visible.

## Uncertainties

The following remain documented_only:

- Application meanings of descriptor fields, lookup constants, and dispatch operations.
- Calling convention and implicit inputs of the initial indirect invocation.
- Origins and meanings of the five additional search inputs.
- Ownership and lifetime guarantees for returned addresses.
- Shared-word identities and their broader synchronization rules.
- Effects of every dependency on globals, descriptor storage, and object state.
- Exact failure mechanism used by F-a817d8d58c10.
- Exact control-transfer mechanics of the final indirect operation.
- Behavior under invalid addresses, concurrent mutation, or invalid dispatch entries.

Index increment and match adjustment use 32-bit integer values. No explicit overflow check, saturation, or range validation is visible. Examples above avoid overflow. Exact overflow behavior is not established by the supplied representation and must not be replaced with a claim of guaranteed wrapping or unbounded arithmetic.

## Evidence

This contract is based solely on the supplied static function representation and its opaque dependency mapping. Established claims concern visible accesses, offsets, argument values, comparisons, branch structure, and call order.

The nonreturning failure behavior and final indirect-transfer limitation are supported by annotations in that representation, not independent runtime observation. Semantic interpretations are explicitly separated from those observations.

No runtime tests have been performed. No implementation access or external verification was used.
