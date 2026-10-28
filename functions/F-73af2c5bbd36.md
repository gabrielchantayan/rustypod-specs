<!-- Trusted provenance: F-73af2c5bbd36; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-73af2c5bbd36

## Purpose

F-73af2c5bbd36 conditionally scans a sequence of indexed entries associated with an object. Scanning is enabled by a narrow flag derived from an object field. The scan uses an indirect operation to retrieve each entry and examines the entry’s first integer word.

If scanning is disabled, the captured entry count is nonpositive, or every examined entry has a zero first word, the function returns without invoking either named downstream dependency.

At the first entry whose first word is nonzero, the function invokes F-588f8c1c803a. If that invocation returns normally, it invokes F-feee9156a257. The latter invocation is marked as non-returning in the supplied static material.

An interpretation such as “validate entries and terminate on a violation” is plausible but **inferred intent**, not an established semantic classification. The evidence does not identify what the entries represent, what a nonzero first word means, or what the downstream operations do.

## Interface

### Input

The function accepts one object pointer and has no explicit result value.

The object must support the following accesses along the paths that reach them:

| Location | Required interpretation | Use |
|---|---|---|
| Integer slot 10 | Integer converted to a character-sized value | Enables scanning when the converted value is nonzero |
| Integer slot 1 | Signed integer | Captured upper bound for entry indices |
| Integer slot 0 | Value used as a dispatch-table address | Resolves the indexed-entry operation |
| Dispatch-table byte offset `0x40` | Callable target address | Invoked with the object pointer and current index |

The supplied representation uses integer-indexed object storage and a byte offset within the dispatch table. It does not independently establish the target ABI.

Under a 32-bit integer layout:

- Object slot 0 begins at byte offset 0.
- Object slot 1 begins at byte offset 4.
- Object slot 10 begins at byte offset 40, or `0x28`.
- The returned entry’s first integer occupies four bytes.
- The dispatch-table displacement remains `0x40` bytes, or 64 bytes.

These byte offsets for object fields are conditional on the 32-bit integer layout. Pointer width, function-pointer width, alignment requirements, and calling convention are **documented_only unknowns**.

### Indexed-entry result

Each indirect invocation supplies:

1. The original object pointer.
2. The current signed integer index.

Its result is treated as a pointer to storage whose first integer can be read immediately. No null check precedes that read.

The object pointer is not checked for null before the initial flag access. Valid storage and callable dispatch state are therefore preconditions on any path that accesses them, not conditions the function validates.

### Output and completion

Ordinary completion produces no explicit value and no explicit status code.

There are three observed normal-return paths:

- The enable field converts to zero.
- The captured count is zero or negative.
- All entries selected by the captured count have zero first words.

Finding a nonzero first word instead enters the downstream-call path. The final dependency is statically marked non-returning; the mechanism and runtime validity of that property are not established here.

## Behavior

### Enable decision

The first explicit object access is to integer slot 10. The value is converted to a character-sized value and compared with zero.

If the converted value is zero, the function returns immediately. It does not explicitly read the count field, resolve the indexed-entry operation, or invoke either named downstream dependency.

The conversion matters: this is not established as a full-width integer comparison against zero. On an ABI with an eight-bit character and ordinary low-eight-bit narrowing, a full-width nonzero value whose low eight bits are zero disables scanning. For example, the integer value 256 would disable scanning under that model.

Character width and out-of-range conversion semantics are not independently supplied. The contract therefore preserves the conversion rather than asserting a universally portable low-byte rule. Under ordinary narrowing, whether the resulting character is signed does not change the zero-versus-nonzero decision.

### Count capture and bounds

If scanning is enabled, the function reads integer slot 1 once and retains that value as the scan bound. The initial index is zero.

Before every attempted entry lookup, it compares the captured bound with the current index using a signed comparison. Scanning ends when the bound is less than or equal to the index.

Consequences:

- A zero bound causes no entry lookup.
- A negative bound causes no entry lookup.
- A positive bound permits indices from zero through one less than that bound.
- No sentinel entry is consulted beyond the captured bound.
- Later changes to the object’s count field do not change this invocation’s retained bound.

Under a 32-bit signed-integer model, even a bound of 2,147,483,647 does not require the local index increment to overflow. After a zero entry at index 2,147,483,646, the index becomes 2,147,483,647, and the next bound check ends the scan.

This arithmetic observation does not establish that an object can validly supply that many entries.

### Per-entry lookup and examination

For every admitted index, the function resolves the indirect operation through the current dispatch-table address in object slot 0 and the callable entry at byte displacement `0x40`.

The dispatch operation is resolved afresh for each lookup. The supplied representation does not cache the dispatch-table address or callable target across iterations. A dependency that changes the object’s dispatch state can therefore affect a subsequent lookup, provided that the changed state remains valid.

The indirect operation is called with the object pointer and current index. Once it returns, the function reads the first integer at the returned address.

- If that integer is zero, the function increments its index and repeats the bound check.
- If that integer is nonzero, the scan stops immediately and enters the downstream-call path.

No later index is looked up after a nonzero entry is found. No other entry fields are explicitly inspected. The sign or magnitude of a nonzero first word does not affect the branch.

The first-word read occurs after the indirect invocation returns. There is no separate validity check, null check, ownership check, or recovery branch before that read.

### Downstream-call path

Upon observing a nonzero first word:

1. F-588f8c1c803a is invoked without explicit arguments in the supplied representation.
2. If it returns normally, F-feee9156a257 is invoked without explicit arguments in that representation.

No explicit object, entry pointer, index, count, or entry value is forwarded to either dependency.

This absence of explicit arguments does not establish that either dependency lacks access to shared state. It also does not establish that F-588f8c1c803a preserves the object, entry storage, globals, or resources.

If F-588f8c1c803a does not return normally, the second invocation is not reached through the ordinary sequential path.

F-feee9156a257 is marked as non-returning. The evidence does not establish whether it exits a process, raises an exception, transfers control elsewhere, waits indefinitely, or uses some other mechanism. Behavior following an unexpected normal return from that dependency is **documented_only**, not a defined recovery case.

## Side effects

The function contains no explicit writes to object fields or entry storage. It maintains a local count snapshot and local index.

Its explicit reads, in observable logical order, are:

1. The enable field.
2. The count field, only if enabled.
3. For each admitted index, the dispatch state needed to resolve the indirect operation.
4. The returned entry’s first integer, after that invocation returns.

Dependency activity can interleave additional reads, writes, callbacks, allocation, deallocation, or external effects between these operations. Those effects are unknown and must not be treated as absent.

There is no explicit allocation, deallocation, retention, or ownership transfer in this function. In particular:

- Returned entry storage is read but is not explicitly freed here.
- No release operation is explicitly invoked after a zero entry.
- A nonzero entry does not trigger an explicit release before the downstream calls.
- No guarantee is established that dependencies leave input or returned storage alive.

The caller must provide valid object storage for each explicit read that is reached. The indirect dependency must return storage valid for the immediate first-word read. Null, dangling, misaligned, or otherwise unreadable storage is not translated into a defined status or alternate path.

No synchronization is visible. A captured count does not make the object, dispatch table, or entries immutable or thread-safe.

## Dependencies

| Dependency | Established interaction | Unestablished behavior |
|---|---|---|
| Unresolved indirect dependency; opaque function ID unavailable | Resolved through object dispatch-table offset `0x40`; called once per examined index with the object and index; result is immediately dereferenced as an integer pointer | Implementation, ownership, mutation, failure behavior, global effects, and whether different iterations select the same callable |
| F-588f8c1c803a | Called after the first observed nonzero entry word; no explicit arguments; its result is not inspected | Purpose, side effects, resource handling, and whether it returns normally |
| F-feee9156a257 | Called after F-588f8c1c803a returns normally; no explicit arguments; marked non-returning | Actual non-return mechanism, external effects, and behavior if the annotation is inaccurate |

No opaque identifier was supplied for the indirect target. Assigning it the identity or guarantees of a known function would exceed the evidence.

## Acceptance examples

These are conditional behavioral expectations derived from static evidence, not executed tests. Except where stated otherwise, they assume valid storage, normally returning entry lookups, a 32-bit integer model, and ordinary eight-bit character narrowing.

1. **Disabled object**  
   The enable field is zero. The function returns after the enable decision. The count field is not explicitly read, no entry lookup occurs, and neither named dependency is called.

2. **Nonzero full-width field that narrows to zero**  
   The enable field is 256. Under the stated narrowing model, scanning is disabled. Treating the original full-width value as a Boolean would not match this behavior.

3. **Enabled object with a negative count**  
   The enable conversion is nonzero and the count is −1. The count is captured, but index zero fails the signed bound check. No dispatch lookup or downstream call occurs.

4. **Enabled object with an empty sequence**  
   The enable conversion is nonzero and the count is zero. No indexed-entry invocation occurs.

5. **All entries clear**  
   The captured count is three. Lookups for indices 0, 1, and 2 each return storage whose first word is zero. Those indices are examined in ascending order. The function returns without either named downstream call.

6. **First match stops the scan**  
   The captured count is four. Entry 0 has first word zero; entry 1 has first word −7. Only indices 0 and 1 are looked up. F-588f8c1c803a is called next. If it returns normally, F-feee9156a257 is called. Indices 2 and 3 are not examined.

7. **Count changes during lookup**  
   The captured count is two. The index-0 dependency changes the stored count to zero and returns a zero entry. The function still attempts index 1, assuming all required storage and dispatch state remain valid.

8. **Dispatch changes during lookup**  
   The index-0 dependency changes the object’s dispatch-table address and returns a zero entry. If another iteration is admitted, its operation is resolved from the changed dispatch state rather than a previously cached target.

9. **Null lookup result**  
   An admitted lookup returns null. The function has no defined null-result branch; it attempts the first-word read. No graceful return, diagnostic, or downstream-call outcome is guaranteed by this evidence.

## Uncertainties

The following remain **documented_only**:

- Concrete integer, character, pointer, and function-pointer widths.
- Exact ABI, calling convention, alignment, and narrowing rules.
- Semantic names and meanings of object fields and entry words.
- The identity and complete contract of the indirect dependency.
- Whether dependencies mutate globals, change dispatch state, invalidate storage, or manage resources.
- Failure, exception, signal, and nonlocal-control-transfer behavior.
- The runtime validity and mechanism of the final non-return annotation.
- Behavior under concurrent mutation or invalid memory access.
- Any intended diagnostic, validation, or termination semantics beyond the observed call sequence.

## Evidence

The established behavior comes solely from the supplied static function representation: the character-converted enable check, captured signed bound, ascending indexed indirect calls, immediate first-word examinations, first-nonzero stopping condition, and ordered downstream invocations.

The 32-bit layout examples are conditional ABI interpretations. The final non-return property is supplied as an annotation, not demonstrated execution.

No runtime tests, dependency implementations, or independent ABI evidence were available. No runtime tests have been performed.
