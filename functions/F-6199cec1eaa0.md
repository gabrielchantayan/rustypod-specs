<!-- Trusted provenance: F-6199cec1eaa0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6199cec1eaa0

## Purpose

F-6199cec1eaa0 conditionally invokes an action on a receiver-associated target, using a descriptor and, when necessary, an ordered search over receiver-provided entries.

The established behavior is a selection procedure:

- Attempt a preferred-entry comparison when the descriptor’s signed 32-bit field at byte offset 12 is positive.
- Otherwise, or after that comparison fails, query the descriptor through a dependency.
- If that query returns nonzero, invoke the action with the 32-bit value `0xFFFFFFFF`.
- If the query returns zero, search entries in ascending index order and invoke the action for the first comparison that returns exactly zero.
- If the search finds no match, return without invoking the action.

“Descriptor,” “entry,” “comparison,” and “action” are descriptive labels, not established application-level types. In particular, the supplied evidence does not establish that the descriptor contains a string, that entries represent names, or that the final action changes a selection.

## Interface

**Inputs**

1. **Receiver:** a pointer to an object whose first word supplies an indirect-method table.
2. **Descriptor address:** a 32-bit address-bearing argument used to access fields at byte offsets 4 and 12.

The supplied representation establishes a 32-bit model for the relevant integers, words, and address calculations. It does not establish byte order.

**Receiver layout used directly**

| Byte offset | Required use |
|---|---|
| 0 | Method-table pointer, read to dispatch indirect calls. |
| 64 | A 32-bit word passed as the first argument to F-9b939bfbd36e when an action is selected. |

Two indirect-method slots are used:

| Method-table byte offset | Observed use |
|---|---|
| 280 (`0x118`) | Called with the receiver; produces a 32-bit unsigned search bound. |
| 332 (`0x14C`) | Produces an entry-related result in caller-provided temporary storage. Search calls explicitly include the receiver and an unsigned index. The preferred-entry call has an unresolved argument-list discrepancy, described below. |

**Descriptor layout used directly**

| Byte offset | Required use |
|---|---|
| 4 | Address supplied to F-b9acea4b06ad as a source. The source object’s extent and format are unknown. |
| 12 | A signed 32-bit field governing the initial branch. On successful preferred-entry comparison, this location is read again as a 32-bit unsigned action argument. |

These uses do not establish that the descriptor consists only of these fields, or that the object beginning at offset 4 ends before offset 12.

**Output**

The function has no specified return value. Its externally relevant results are calls to dependencies and indirect methods, possible effects of those calls, and zero or one direct invocation of F-9b939bfbd36e during a normally completing invocation.

**Preconditions**

The receiver, descriptor, method table, selected method entries, and helper-visible objects must be valid for the accesses and calls actually reached. No null, bounds, object-type, or address-overflow validation is visible.

The preferred-entry path additionally requires resolution of the indirect method’s calling convention. The supplied representation alone does not establish a complete argument list for that call.

## Behavior

### 1. Establish temporary state and choose the initial path

The function first calls F-b02b795a8319 with the address of a four-byte temporary region.

Only after that call returns does it read the signed 32-bit descriptor field at offset 12:

- A value greater than zero enters the preferred-entry path.
- Zero or any negative signed value enters the fallback path.

Consequently, the branch is based on the field’s value after F-b02b795a8319 returns, not necessarily its value at function entry. No guarantee that dependencies preserve descriptor or global state is available.

### 2. Preferred-entry path

The function dispatches through method-table slot 332 to produce an entry-related result in temporary storage.

The supplied representation explicitly shows a result-storage address and the receiver at this call, but no explicit index argument. Search-path calls to the same slot do include an index. The contract therefore does not assign a particular index or preferred-entry rule to this first call.

After the indirect call returns, the function performs this ordered sequence:

1. Call F-46f0b3cb316e with an eight-byte destination temporary and the address of an eight-byte portion of the entry result.
2. Copy an adjacent four-byte portion of the entry result into another four-byte temporary.
3. Call F-ee5919ca4be6 on the entry-result temporary.
4. Call F-b9acea4b06ad to construct or populate an eight-byte temporary from the eight-byte destination established in step 1.
5. Call F-44362605ab79 on that new temporary and retain its 32-bit result.
6. Call F-b9acea4b06ad to construct or populate a second eight-byte temporary from the descriptor address plus 4.
7. Call F-4a902fecdea3 with the descriptor-derived temporary and the retained 32-bit result.
8. Call F-289ec4311c64 on the descriptor-derived temporary.
9. Call F-289ec4311c64 on the entry-derived temporary.

The comparison outcome is determined by the unsigned 32-bit return from F-4a902fecdea3:

- Exactly zero is accepted.
- Every nonzero value is rejected.

The supplied arithmetic computes a 32-bit subtraction from one, but an unsigned greater-than-one check forces all results above one into rejection. Thus, `0xFFFFFFFF`, for example, is rejected; wraparound in the intermediate subtraction does not make it a match.

On acceptance:

1. Read the descriptor field at offset 12 again, now as an unsigned 32-bit value.
2. Read the receiver word at offset 64.
3. Call F-9b939bfbd36e with those two values, receiver word first.
4. Proceed to final cleanup.

The descriptor field is not captured for later use when the initial branch is taken. Intervening dependencies may change it. The action argument can therefore differ from the initially observed positive value, including becoming zero or a value with its high bit set.

On rejection, proceed to the fallback path. There is no direct action call for the rejected preferred-entry attempt.

### 3. Fallback descriptor query

The fallback path is reached either directly from the initial signed-field test or after preferred-entry rejection.

It performs the following sequence:

1. Call F-b9acea4b06ad with an eight-byte destination temporary and the descriptor address plus 4.
2. Call F-6d70e45e29c9 on that temporary.
3. Retain its result.
4. Call F-289ec4311c64 on the temporary.
5. Choose the next path according to whether the retained result is zero.

If the result is nonzero:

- Read the receiver word at offset 64 after the temporary’s cleanup call.
- Call F-9b939bfbd36e with that word and `0xFFFFFFFF`.
- Proceed to final cleanup.

The specific nonzero query value does not alter the second action argument. The sentinel’s application meaning is unknown; the evidence establishes its bit pattern, not whether it means “none,” “clear,” “invalid,” or another state.

If the query result is zero, proceed to indexed search.

### 4. Indexed search

The function invokes method-table slot 280 with the receiver and captures its 32-bit unsigned result as the search bound.

This method is called once for the search. The bound is not refreshed after entry retrieval or dependency calls.

Indices are considered in ascending unsigned order, starting at zero and continuing while the index is less than the captured bound. For each index:

1. Invoke method-table slot 332 with result storage, the receiver, and the current index.
2. Perform the entry extraction, four-byte copy, entry-result cleanup, two temporary constructions, token extraction, comparison, and two cleanup calls described for the preferred-entry path.
3. Accept only a comparison result of exactly zero.

The descriptor-derived temporary is reconstructed for every candidate. Neither the descriptor source nor the entry-derived comparison value is cached across iterations.

On the first accepted comparison:

- Read the receiver word at offset 64 after both comparison-temporary cleanup calls.
- Call F-9b939bfbd36e with that word and the matching index.
- Stop searching and proceed to final cleanup.

If every examined comparison is nonzero, the function completes without directly calling F-9b939bfbd36e.

A bound of zero causes no entry retrieval calls. For a bound of `0xFFFFFFFF`, the greatest examined index is `0xFFFFFFFE`. Incrementing that index produces `0xFFFFFFFF`, which fails the next less-than test; this loop does not require unsigned wraparound to terminate.

### 5. Final cleanup

Every visible normal completion path calls F-ee5919ca4be6 on the four-byte temporary originally passed to F-b02b795a8319, then returns.

This cleanup follows any direct action call. No explicit error-handling or exceptional-unwinding mechanism is established by the supplied representation.

## Side effects

The function contains no explicit writes to receiver or descriptor fields. It does write caller-local temporary storage, including an explicit four-byte copy for each retrieved entry.

External mutation remains possible through every dependency and indirect method. In particular:

- Receiver fields and the method-table pointer are not guaranteed stable across calls.
- The search bound is captured once, but subsequent entry dispatches resolve through the receiver’s then-accessed method table.
- The descriptor field used for the preferred action argument is reread after comparison and cleanup.
- The receiver word used for an action is read after the cleanup calls preceding that action.

No rollback is visible. If earlier calls mutate state and a later comparison rejects the candidate, those earlier effects are not explicitly reversed.

Temporary addresses are passed to dependencies, but allocation, ownership transfer, reference counting, retention, and deallocation rules are not established. The cleanup-like roles described here identify call placement, not independently verified memory-management semantics.

There is no separate visible cleanup of the eight-byte extraction destination or its adjacent four-byte copy before reuse or return. Whether they require cleanup, are covered by another object, or contain non-owning data is unknown.

## Dependencies

| Opaque ID or slot | Established use |
|---|---|
| F-b02b795a8319 | Called first with four-byte temporary storage. |
| F-ee5919ca4be6 | Called on each retrieved entry result before comparison-temporary construction, and on the initial four-byte temporary at normal exit. |
| F-46f0b3cb316e | Receives an eight-byte destination and an eight-byte entry-result source region. |
| F-b9acea4b06ad | Populates eight-byte temporaries from an entry-derived region or descriptor offset 4. |
| F-44362605ab79 | Produces a 32-bit value from the entry-derived temporary. |
| F-4a902fecdea3 | Produces the unsigned 32-bit result whose zero value accepts a candidate. |
| F-6d70e45e29c9 | Produces the fallback query result; zero searches, nonzero selects the sentinel action. |
| F-289ec4311c64 | Called after temporary use, in the ordering specified above. |
| F-9b939bfbd36e | Receives the receiver-associated word and the selected field value, index, or sentinel. Its return value is not used. |
| Indirect slot 280 | Produces the captured unsigned search bound. |
| Indirect slot 332 | Produces entry-related temporary data; its preferred-path argument contract is unresolved. |

None of these dependencies is established as pure, nonthrowing, non-reentrant, or ownership-preserving.

## Acceptance examples

These examples are conditional specifications, not observed executions. They assume the relevant calls return normally and supply the stated results.

1. **Nonpositive field, nonzero fallback query.**  
   With an initial signed field of zero and a fallback query result of 7, no entry-count or entry-retrieval call occurs. The action receives `0xFFFFFFFF` as its second argument. The query temporary is cleaned before the receiver-associated word is read.

2. **Negative field uses fallback.**  
   A field bit pattern of `0xFFFFFFFF` is signed-negative for the initial test. It does not trigger the preferred-entry attempt.

3. **Empty search.**  
   If the fallback query returns zero and the count method returns zero, there are no indexed retrievals and no direct action call. Final cleanup still occurs.

4. **First matching index wins.**  
   Given a captured bound of 4 and comparison results 2, 0, 0 for successive candidates, only indices 0 and 1 are retrieved. The action’s second argument is 1; index 2 is never examined.

5. **All nonzero comparison results reject.**  
   Results of 1, 2, and `0xFFFFFFFF` all reject. If those cover the complete search, no direct action occurs.

6. **Preferred match bypasses fallback.**  
   Conditional on the unresolved preferred retrieval being valid, an initially positive field and a preferred comparison result of zero bypass the fallback query and count method. The action receives the field value reread after comparison cleanup.

7. **Preferred rejection can repeat work.**  
   A failed preferred comparison does not exclude any indexed candidate from the fallback search. If fallback search reaches the same underlying entry, it is processed again.

8. **Mutation changes the preferred action argument.**  
   If the initial field is 3, the preferred comparison accepts, and an intervening dependency changes the field to `0x80000000`, that later bit pattern is passed to the action. There is no visible positivity recheck.

## Uncertainties

**Documented_only — preferred retrieval signature.** The preferred and indexed calls to slot 332 expose different argument lists. The missing preferred-path index cannot be reconstructed from this evidence.

**Documented_only — concrete types and layout.** Temporary-region sizes and address relationships are visible, but complete object boundaries, alignment requirements, and semantic field types are not.

**Documented_only — ownership and failures.** The dependencies’ allocation, retention, release, exception, fault, and error-reporting behavior is unavailable. Cleanup is established only for visible normally returning paths.

**Inferred intent.** The overall structure resembles preferred selection followed by descriptor-based search, with a special sentinel action. That interpretation must not replace the exact zero/nonzero decisions or imply a particular application domain.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. It records explicit field accesses, call order, branch conditions, temporary sizes, and 32-bit arithmetic.

No dependency implementations, runtime traces, or external type definitions were supplied. No runtime tests have been performed. The acceptance examples express consequences of the visible control flow under stated dependency outcomes; they are not execution evidence.
