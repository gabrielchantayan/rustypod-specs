<!-- Trusted provenance: F-cdce10c413a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdce10c413a0

## Purpose

F-cdce10c413a0 conditionally scans an indexed sequence associated with a caller-supplied object. It skips entries whose first word is zero. At the first entry whose first word is nonzero, it invokes F-e0a692eba8b7 with an address four bytes beyond that word’s value, then invokes F-feee9156a257 with the original value.

The established behavior is **first-nonzero selection**, not processing every nonzero entry. The scan returns normally if its initial gate is disabled or if all entries within the captured bound are zero.

The final dependency is annotated as not returning. That annotation supports an expected terminal transfer of control, but does not establish whether the dependency terminates execution, raises an exception, transfers control elsewhere, or performs some other nonreturning operation.

## Interface

### Input

The function accepts one pointer to an object represented as an array of integer-sized words.

The recovered representation uses 32-bit integer words. Under that layout, the function accesses:

| Position | Byte offset | Role |
|---|---:|---|
| Word 0 | 0 | Base address used to locate an indirect callable |
| Word 1 | 4 | Signed scan bound, captured once |
| Word 4 | 16 | Gate, tested after conversion to a character-sized value |

Other object fields are not directly accessed by the supplied function.

The gate depends only on the character-sized portion retained by the conversion, not on whether the entire word is zero. Under an eight-bit character representation, this is a test of the low eight bits. Character signedness does not change whether that retained value is zero. The supplied material does not independently establish the target’s character width or the physical byte order of the object.

### Indirect callable

For every inspected index, the function:

- Reads the dispatch-base value from word 0 of the input object.
- Reads a callable address at byte offset 64 from that base.
- Invokes that callable with the original object pointer and the current signed integer index.
- Treats the result as a pointer to an integer word and immediately reads that word.

The callable’s opaque dependency ID is unavailable. Its role here is an indexed lookup; its broader contract is unknown.

The returned pointer must be readable for the immediately following word access. A null result is not treated as an empty entry: it would be dereferenced.

### Output and completion

There is no explicit return value.

Normal return is established for:

- A disabled gate.
- An enabled gate with a signed bound of zero or less.
- An enabled gate for which every inspected entry word is zero.

For a selected nonzero entry, control passes through F-e0a692eba8b7 and then, if that call returns, to F-feee9156a257. The latter call is marked nonreturning in the supplied evidence.

### Preconditions

For defined execution along a reached path:

- The input pointer must permit the required field reads.
- Every reached indirect-call target must be callable with the observed arguments and result convention.
- Every returned entry pointer must permit a word read.
- A selected nonzero value and its four-byte adjustment must be acceptable to the dependencies that receive them.

The function performs no explicit validation of these conditions.

## Behavior

### Initial gate and bound

The first semantic field access is the gate at word 4. If its character-sized conversion is zero, the function returns without reading the bound, performing indexed lookup, or calling either named dependency.

If the gate is enabled, the function reads word 1 once and retains that signed value as the scan bound. It begins with index zero.

A nonpositive bound causes immediate normal return after this setup. A positive bound permits inspection of indices from zero through one less than the captured bound.

The gate is not checked again. The bound is not refreshed after a dependency call.

### Indexed inspection

Before every lookup, the current index is compared with the captured bound using a signed comparison. When the index has reached the bound, the function returns.

For an in-range index, the function resolves the indirect callable through the current word-0 value of the input object. This resolution occurs separately for each iteration; the callable address is not retained across iterations.

The lookup result is dereferenced once to obtain the candidate word.

If the candidate word is zero:

- Neither named dependency is invoked for that entry.
- The index increases by one.
- The next iteration begins with the bound comparison.

If the candidate word is nonzero:

- The scan stops immediately.
- No later index is inspected.
- The selected word is retained for the two subsequent calls.

Selection is based on nonzero value, not positivity. A word with its sign bit set is therefore selected just as any other nonzero word is.

### Selected-entry calls

For the first selected word, the function invokes F-e0a692eba8b7 with that value adjusted by four bytes.

If F-e0a692eba8b7 returns, the function then invokes F-feee9156a257 with the original selected value.

The function does not reread the selected entry between these calls. Consequently, a normal mutation of the entry storage by F-e0a692eba8b7 does not replace the argument subsequently passed to F-feee9156a257.

There is no explicit branch based on a result from F-e0a692eba8b7, and no explicit result from it is consumed. If it does not return normally, the second call is not reached.

No continuation after a returning F-feee9156a257 is established by the supplied representation.

### Integer and address arithmetic

The index and bound are represented as signed 32-bit integers. For a positive bound, the counter increases only after inspecting a zero entry at an index strictly below that bound. Even at the maximum positive signed bound, the final increment reaches the bound without exceeding the signed range. The loop itself therefore does not require signed counter overflow.

The selected-value adjustment is different: the evidence expresses addition of four to the selected integer value before passing it onward. It does not establish a checked addition or an explicit overflow branch.

For values whose four-byte adjustment would cross the relevant representable or address boundary, behavior is **documented_only**. A portable contract must not invent either trapping behavior or modulo wraparound from this representation alone.

## Side effects

### Direct accesses

The function explicitly performs reads of:

1. The gate word.
2. The bound word, only when the gate is enabled.
3. The dispatch-base word for each inspected index.
4. The callable address at offset 64 from each such base.
5. The first word at each returned entry pointer.

It contains no explicit store to the input object, an entry, or global state.

This ordering describes the semantic accesses established by the supplied representation. It does not establish volatile, atomic, concurrent, or memory-mapped-I/O guarantees.

### Dependency effects

The indirect lookup and both named dependencies may have side effects not visible here. Their effects on globals, the input object, entry storage, allocation state, and control flow are unknown.

In particular:

- A lookup may change the object’s dispatch base. A later iteration resolves the callable from the then-current base.
- A lookup may change the object’s stored bound, but this function continues to use its previously captured bound.
- A lookup may change the gate, but this function does not reevaluate it.
- F-e0a692eba8b7 may alter memory before F-feee9156a257 is reached; the second call still receives the retained original selected value.

These observations do not imply that arbitrary mutations are safe. Each later access still requires valid memory and a valid callable.

### Ownership and lifetime

No ownership transfer, allocation, deallocation, reference-count adjustment, or destructor semantics can be established from the function alone.

The four-byte adjustment may suggest a distinction between a leading word and subsequent storage. That is inferred structural intent, not proof of a header, payload, object type, or allocation layout.

Calling F-feee9156a257 with the original value does not by itself prove that the value is freed or consumed. Likewise, F-e0a692eba8b7 cannot be assumed to preserve or release any storage.

## Dependencies

| Dependency | Established interaction | Unknown contract |
|---|---|---|
| Indirect indexed callable; opaque ID unavailable | Resolved at dispatch-base offset 64; receives the object pointer and current index; returns an address whose first word is read | Error signaling, side effects, result ownership, lifetime, and valid-index requirements |
| F-e0a692eba8b7 | Called once for the first nonzero entry, with the selected value adjusted by four bytes | Meaning of the argument, ownership effects, global effects, and whether it always returns |
| F-feee9156a257 | Called after F-e0a692eba8b7 returns, with the unadjusted selected value | Nature of its nonreturning behavior, ownership effects, and error semantics |

There is no explicit local recovery path for an invalid pointer, invalid dispatch target, lookup failure, or dependency failure.

## Acceptance examples

These are conditional behavioral expectations derived from static evidence, not executed tests. Each assumes valid accesses, ordinary sequential execution, and dependencies that satisfy the stated scenario.

### Disabled gate

Given a gate whose retained character-sized value is zero, the function returns without consulting the bound or performing any lookup.

Under an eight-bit character model, a gate word of 256 is disabled even though the complete word is nonzero.

### Nonpositive bound

Given an enabled gate and a captured bound of zero or a negative value, the function returns without an indirect lookup or either named dependency call.

### All entries empty

Given an enabled gate, a bound of three, and lookup results whose first words are zero at indices 0, 1, and 2:

- Lookups occur in that order.
- Neither named dependency is called.
- The function returns after the scan reaches index 3.

### First nonzero entry

Given an enabled gate, a bound of four, and candidate words of zero at index 0 and 4096 at index 1:

- Only indices 0 and 1 are inspected.
- F-e0a692eba8b7 receives 4100.
- If it returns, F-feee9156a257 receives 4096.
- Indices 2 and 3 are not inspected.

### Bound mutation during lookup

Given an initially captured bound of three, suppose the lookup at index 0 returns an entry containing zero and changes the object’s stored bound to zero.

The function still proceeds to index 1 using the captured bound. This expectation requires the modified object to remain valid for subsequent accesses.

### Dispatch mutation during lookup

Suppose the first lookup returns a zero entry and changes word 0 to another valid dispatch base.

The next lookup is resolved through that new base. The initial callable is not reused merely because it was used for index 0.

### Entry mutation after selection

Suppose the selected word is 4096 and F-e0a692eba8b7 changes the entry’s stored word to zero before returning.

F-feee9156a257 still receives 4096, because the function does not reread the entry.

### Null lookup result

A null lookup result is not an accepted representation of an empty entry. The next operation attempts to read a word through that result; no explicit null-handling behavior is established.

## Uncertainties

The following remain **documented_only**:

- Exact machine ABI, pointer representation, character width, and byte order beyond the recovered word-based layout.
- Concrete types and semantic meaning of the input object and selected entries.
- Whether the dispatch structure follows any particular object-system convention.
- All dependency contracts beyond the observed call shapes.
- Ownership and lifetime effects of every dependency.
- Selected-value addition behavior at integer or address boundaries.
- The actual reason F-feee9156a257 does not return.
- Behavior if that dependency unexpectedly returns.
- Fault handling, exception propagation, concurrency, and reentrancy guarantees.

Inferred intent is limited to a gated search for the first nonempty entry followed by two ordered operations on related addresses. No stronger interpretation—such as destruction, cleanup, release, or fatal error handling—is established.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes the gate, the once-captured signed bound, ascending indexed lookup, zero-skipping behavior, first-nonzero selection, four-byte argument adjustment, ordered dependency calls, and the nonreturn annotation on the final call.

No dependency implementations, runtime traces, ABI documentation, ownership declarations, or error contracts were supplied. No runtime tests have been performed.
