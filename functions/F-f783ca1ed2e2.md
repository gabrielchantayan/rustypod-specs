<!-- Trusted provenance: F-f783ca1ed2e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f783ca1ed2e2

## Purpose

F-f783ca1ed2e2 conditionally scans an object’s indexed entries. When scanning is enabled, it inspects entries in ascending index order and stops at the first entry whose leading integer is nonzero. On finding such an entry, it calls F-8ad439e5daa6, then F-feee9156a257.

If scanning is disabled, the captured entry count is nonpositive, or every inspected entry has a zero leading integer, the function returns without making those two dependency calls.

The established behavior is an enabled, ordered search followed by a two-call action. Interpreting that action as an assertion failure, shutdown, exception, or other particular error mechanism would require evidence not supplied here. The final dependency call is annotated as non-returning in the supplied static representation; its underlying implementation is unavailable.

## Interface

### Input

The function accepts one object address. It has no explicit count, mode, callback, or output argument.

The supplied representation accesses this object as an array of integer-sized words:

| Object location | Observed role |
|---|---|
| Word 0 | Address-like value used as the base for locating a callable slot |
| Word 1 | Signed entry count, captured once before scanning |
| Word 4 | Enable value, tested after conversion to a character-width value |

On the apparent 32-bit integer layout, these locations begin at byte offsets 0, 4, and 16 respectively. The callable slot lies at byte offset 64 from the address-like value obtained from word 0.

These offsets describe the apparent target layout, not a portable host-language structure. Exact pointer width, callable representation, alignment rules, and character-conversion semantics require the target ABI.

The callback selected through the slot receives:

1. The original object address.
2. The current signed integer index.

Its result is treated as an address of an integer. The function immediately reads that integer and compares it with zero.

### Output and completion

There is no explicit result value and no caller-provided output buffer.

Ordinary return occurs when:

- The character-width enable test produces zero.
- Scanning is enabled but the captured signed count is zero or negative.
- Scanning is enabled, the captured count is positive, and every returned entry inspected within that count has a zero leading integer.

A nonzero entry selects the dependency-call path instead of ordinary scan completion. F-8ad439e5daa6 is called first. F-feee9156a257 is reached only if the first call returns normally.

### Preconditions

Requirements depend on the path actually taken:

- The initial object access must permit reading word 4.
- If the enable test is nonzero, word 1 must also be readable.
- If the captured count is positive, each reached callback dispatch must be valid: word 0 must be readable, its address-like value must support access to the callable slot at byte offset 64, and that slot must identify a callable target under the applicable ABI.
- Every callback invocation that returns normally must return an address from which the leading integer can be read immediately.

The function supplies no null checks, bounds checks, callback-result validation, or recovery for violations of these requirements. Invalid accesses are outside the established valid-input contract; they are not defined error returns.

## Behavior

### Enable gate

The first observed object read is word 4. Its value is converted to a character-width value, and that converted value is compared with zero.

If the converted value is zero, the function returns immediately. On this path it does not read the count, access the dispatch slot, invoke the callback, or call either named dependency.

This is not established as a full-width “word 4 equals zero” test. A nonzero full-width value can potentially convert to a zero character-width value, depending on the target’s conversion rules. Under a confirmed low-eight-bit conversion model, for example, a word value of 256 would disable scanning. That example is conditional, not a claim that the target conversion model has been independently established.

The enable value is not read again during this invocation.

### Count capture and loop bounds

When scanning is enabled, word 1 is read once and retained as the scan bound. The index starts at zero.

Before every callback invocation, the function checks whether the captured count is less than or equal to the current index using a signed comparison. If so, it returns.

Consequently:

- A negative count produces no callback invocation.
- A zero count produces no callback invocation.
- A positive count permits inspection of indices zero through one less than that captured count.
- Later changes to the object’s count field do not change this invocation’s loop bound.

The signed interpretation matters. A stored bit pattern interpreted as a negative integer does not request a large unsigned traversal.

For a signed integer type of a given width, the index arithmetic does not inherently overflow on the all-zero path. The index increments only after processing an index strictly less than the positive captured count. At the largest representable positive count, the final increment reaches that count exactly; the next bounds check returns before another increment.

### Per-entry dispatch and inspection

For each permitted index, the function:

1. Obtains the address-like value from object word 0.
2. Obtains the callable target from the slot at byte offset 64 from that value.
3. Invokes the target with the original object address and the current index.
4. Reads the leading integer through the returned address.
5. Chooses whether to advance or take the dependency-call path based on whether that integer is zero.

These are ordered behavioral stages, not a guarantee about individual processor instruction ordering.

If the inspected integer is zero, the index increases by one and scanning continues. If it is nonzero, scanning stops immediately. Positive and negative nonzero values have the same effect.

Only the leading integer of each callback result is inspected. No other entry fields are established by the supplied evidence. Callback results need not be distinct: the function neither checks uniqueness nor compares returned addresses.

Dispatch resolution occurs within the loop rather than being captured once before scanning. A mutation affecting object word 0 or the callable slot can therefore affect a later invocation. In contrast, changes to the enable field and count do not alter the already captured control decisions.

### First nonzero entry

After the first nonzero leading integer is observed:

- No later index is inspected.
- F-8ad439e5daa6 is invoked with no explicit arguments in the supplied representation.
- If that call returns normally, F-feee9156a257 is invoked, also with no explicit arguments.
- There is no explicit entry release or object-field update between those calls.

The supplied representation marks F-feee9156a257 as non-returning. Its actual termination mechanism is unknown. This contract does not invent an exit status, diagnostic message, exception type, or behavior if that annotation is inaccurate.

## Side effects

### Direct effects

The established function body reads object fields, resolves and invokes callbacks, reads callback-returned integers, and invokes the two named dependencies on the nonzero path.

It contains no explicit writes to object fields, returned entry storage, or global storage. Index and count bookkeeping are internal to the invocation.

There is no explicit allocation, deallocation, ownership transfer, entry retention, or release operation.

### Effects through calls

The callback and named dependencies may have effects not visible in the supplied body. Their implementations are unavailable. They must not be assumed to:

- Preserve global state.
- Preserve the object or its fields.
- Leave other entries unchanged.
- Avoid allocation or deallocation.
- Return normally.
- Be safe under reentrancy or concurrent access.

In particular, the callback result is dereferenced after the callback returns, so its storage must remain readable for that access. The function does not explicitly release it afterward. This absence does not establish who owns it or whether a callee performs its own memory management.

If F-8ad439e5daa6 changes global state or frees storage, those effects occur before the subsequent call to F-feee9156a257. No preservation guarantee between the two calls is supported.

### Mutability and concurrency

The function contains no explicit synchronization. The count is captured, while dispatch-related storage is consulted during each reached iteration.

Any claim about concurrent mutation requires external memory-model and synchronization guarantees. The scan does not establish a consistent snapshot of the object or entries.

## Dependencies

| Dependency | Established use | Unknown behavior |
|---|---|---|
| Dynamically selected callback; no opaque function ID supplied | Resolved through the byte-offset-64 slot; receives the object address and index; returns an address whose leading integer is read | Target identity, valid index domain, ownership, mutations, failure behavior, and return guarantees |
| F-8ad439e5daa6 | Called once after the first observed nonzero entry, with no explicit arguments | Purpose, outputs, state changes, memory effects, and whether it returns |
| F-feee9156a257 | Called after F-8ad439e5daa6 returns normally; annotated as non-returning | Actual termination mechanism, diagnostics, state changes, and accuracy of the non-return annotation |

The dispatch mechanism resembles an object-associated method table, but that interpretation is inferred intent, not a required object model.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They have not been executed.

### Disabled scan

Given an object whose word 4 converts to character-width zero:

- The function returns.
- The count is not read.
- No callback or named dependency is invoked.

A full-width nonzero enable value belongs to this case only if its conversion produces zero.

### Enabled scan with nonpositive count

Given a nonzero converted enable value and a captured count of zero or a negative signed value:

- The count is read after the enable field.
- No callback dispatch occurs.
- The function returns without invoking either named dependency.

### All inspected entries are zero

Given an enabled object with captured count three, valid callback dispatches, and returned leading integers zero, zero, and zero:

- Callback indices are zero, one, and two, in that order.
- Each returned leading integer is read before the next callback invocation.
- Index three is not passed to the callback.
- The function returns without invoking either named dependency.

### Nonzero first entry

Given an enabled object with a positive captured count and a nonzero leading integer at index zero:

- Only index zero is inspected.
- F-8ad439e5daa6 is called afterward.
- F-feee9156a257 is called only if F-8ad439e5daa6 returns normally.

A negative leading integer satisfies the nonzero condition.

### Nonzero later entry

Given a captured count of four and leading integers zero, zero, nonzero for the first three callback results:

- Indices zero, one, and two are inspected.
- Index three is not inspected.
- The dependency-call sequence follows the nonzero observation.

### Count changes during scanning

Given an initially captured count of three, suppose the callback at index zero changes the object’s count field to zero but returns a valid zero-valued entry.

If subsequent calls return normally and provide valid zero-valued entries, indices one and two are still inspected. The updated count field is not reread.

### Dispatch changes during scanning

Given a captured count of two, suppose the first callback validly changes the object’s dispatch-related storage and returns a zero-valued entry.

The second iteration resolves dispatch again through the then-accessed storage. The function does not require reuse of the first callback target.

### Invalid callback result

If a reached callback returns an address that cannot support the immediate integer read, no defined recovery or error result is established. The acceptance contract requires a readable result rather than a guessed failure mode.

## Uncertainties

The following remain **documented_only**:

- Exact target integer, pointer, and character widths and ABI details. The supplied layout strongly suggests 32-bit integer words, but that does not independently establish every machine property.
- Character narrowing semantics for values outside the character type’s range.
- The object’s complete layout and the semantic names of its fields.
- Whether the callable slot is part of a virtual-method table or another dispatch structure.
- The callback’s implementation, ownership rules, and permissible side effects.
- The meaning of an entry’s leading integer beyond the zero/nonzero distinction.
- The purposes and effects of F-8ad439e5daa6 and F-feee9156a257.
- The behavior if the final dependency returns despite its annotation.
- Thread-safety, reentrancy, and invalid-memory behavior.

An interpretation such as “reject an object containing active entries” may be plausible, but is not established and is not part of the required behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence covers the character-width enable gate, single signed-count capture, ascending indexed dispatch, immediate leading-integer reads, first-nonzero short-circuit, ordered dependency calls, and absence of explicit storage writes or ownership operations in the body.

Layout and semantic interpretations are qualified where they depend on the target ABI or unavailable callees. No runtime tests, execution traces, dependency implementations, or independent ABI verification have been provided or performed.
