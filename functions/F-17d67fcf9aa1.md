<!-- Trusted provenance: F-17d67fcf9aa1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-17d67fcf9aa1

## Purpose

F-17d67fcf9aa1 performs one of two dependent operations using a caller-supplied context and an opaque 32-bit value.

A classification dependency selects the path:

- A result of exactly zero selects two lookups followed by a combining operation.
- Any nonzero result selects traversal of state reached through the context, with one processing call and one advancement call for each iteration accepted by a comparison dependency.

The function has no explicit return value. Its externally meaningful results are dependency calls and any memory or external-state changes those dependencies perform.

These are established structural behaviors. Interpreting the paths as scalar handling versus collection traversal is inferred intent, not a guaranteed semantic distinction.

## Interface

### Inputs

The function takes two arguments:

| Argument | Representation | Established use |
|---|---|---|
| Context | 32-bit integer used as an address | Supplies a selector, addresses a selected record, and is passed to F-e8bbeba61a78. |
| Opaque value | Uninterpreted 32-bit quantity | Passed to F-9316cc774c14 and, on the zero-result path, to two calls of F-b4a9589b2ea9. |

The opaque value is not directly dereferenced by this function. Dependencies may interpret it as a value, handle, address, or another representation; their contracts are unavailable.

### Output

There is no explicit returned value or status. Normal completion means that the selected sequence of dependency calls has returned and, for the traversal path, that the comparison dependency has eventually returned exactly one.

No direct error-reporting mechanism is established.

### Context-relative layout

The visible accesses establish the following addressing relationships:

- A 32-bit selector is read at context offset 84 bytes.
- The selected-record address is the context address plus the selector multiplied by 28 bytes.
- On the traversal path, a 32-bit address-bearing word is read at offset 16 bytes within that selected record.
- A 32-bit cursor value is initially read at offset 8 bytes from the address obtained through that word.

The selected-record calculation does not include an additional array-header displacement. The zero-result path passes the selected-record address itself to a dependency.

The selector is represented as a signed 32-bit integer. Multiplication by 28 and the address additions require valid results under the actual execution model. The supplied material does not establish portable overflow behavior. This contract therefore does not promise wrapping, saturation, or rejection for out-of-range selectors or addresses.

### Memory preconditions

For defined normal execution:

- Every directly accessed context word must be readable when its access occurs.
- On the traversal path, the address-bearing record word must support the subsequent read at offset 8.
- Values and addresses supplied to dependencies must satisfy those dependencies’ unknown preconditions.
- Any arithmetic used to form an address must produce an address meaningful to the execution environment.

There is no visible null check, bounds check, selector validation, or pointer validation. A zero context is not a special supported case merely because it is zero.

## Behavior

### Initial classification

The first dependency call is F-9316cc774c14 with the opaque value.

No context field is directly read before this call. Consequently, any context changes made by this dependency can affect the subsequent path.

The returned integer is compared with zero:

- Exactly zero selects the lookup-and-combine path.
- Every other value, including a negative value, selects the traversal path.

There is no further local interpretation of this classification result.

### Zero-result path: lookup and combine

The observable sequence is:

1. Read the selector at context offset 84.
2. Calculate the selected-record address using a stride of 28 bytes.
3. Call F-b4a9589b2ea9 with that address as its first argument and the opaque value as its second argument.
4. Retain the 32-bit result of that call.
5. Call F-e8bbeba61a78 with the context.
6. Call F-b4a9589b2ea9 again, using the result from F-e8bbeba61a78 as its first argument and the original opaque value as its second argument.
7. Call F-4fcc06b56813 with the second lookup result first and the first lookup result second.
8. Return after that call completes.

Argument order in the final operation is significant: the lookup based on F-e8bbeba61a78 supplies the first argument, while the selected-record lookup supplies the second.

The selected-record address is calculated before F-e8bbeba61a78 runs. There is no later selector reread on this path. If a dependency changes the selector, the already obtained first lookup result is nevertheless retained for the combining call.

This function does not inspect the two lookup results for nullness, equality, failure indicators, or other properties. It forwards both results to F-4fcc06b56813.

### Nonzero-result path: initialization

The traversal path begins by calling F-e8bbeba61a78 with the context. Its 32-bit result is retained as the second argument for every subsequent processing call on this invocation.

Only after that dependency returns does the function begin reading traversal state from the context.

Initialization performs two distinct selected-record resolutions:

1. Read the selector, calculate the selected-record address, and read the address-bearing word at record offset 16.
2. Read a 32-bit cursor value at offset 8 from the address just obtained.
3. Read the selector again, recalculate the selected-record address, and reread the address-bearing word at record offset 16.
4. Retain this second address-bearing value as the anchor.
5. Copy the anchor and initial cursor into two further local words.

The two record-word reads are separate accesses, not one cached read. The cursor derives from the first record-word read; the retained anchor derives from the second.

The established local state consists of four consecutive 32-bit words in this order:

| Relative offset | Initial content |
|---|---|
| 0 | Anchor from the second record-word read |
| 4 | Cursor obtained through the first record-word read |
| 8 | Copy of the anchor |
| 12 | Copy of the cursor |

The comparison dependency receives the address of the word at offset 4 as its first argument and the address of the word at offset 0 as its second argument. The advancement dependency receives the address of the word at offset 4.

These are addresses into local storage. The dependency contracts are not available, so the extra copied words cannot safely be characterized as unused, nor can the addressed state be assumed immutable across calls.

### Nonzero-result path: iteration

Before processing any item, the function calls F-eac694802fc1 with the two local-state addresses described above.

The stopping condition is exact:

- A return value of exactly one ends traversal and returns from this function.
- Any other return value enters the iteration body.

For each entered iteration:

1. Read the current cursor word after the comparison call has completed.
2. Add 16 bytes to that value.
3. Call F-657c4a32ea89 with:
   - the address of an eight-byte local scratch area;
   - the retained result of F-e8bbeba61a78;
   - the cursor-derived address.
4. Call F-357c4c54c6ea with the address of the cursor word.
5. Repeat the comparison.

The processing call precedes the advancement call. There is no advancement before the first comparison.

The comparison may affect local state through the addresses it receives. Accordingly, the processing address is derived from the cursor value present after comparison, not necessarily from the initially loaded cursor or the preceding advancement result.

The return values of F-657c4a32ea89 and F-357c4c54c6ea are not used for local branching. In particular, a failure-like return from either dependency does not visibly stop traversal.

The function has no local iteration limit and no independent progress check. Termination depends on a comparison eventually returning exactly one, or on a dependency otherwise preventing normal continuation.

### Scratch storage

The same eight-byte local scratch area is passed to every processing call.

The function does not explicitly initialize that area before its first use, reset it between iterations, or inspect its contents afterward. It also performs no explicit finalization of that area on exit.

This establishes only the caller-side behavior. It does not establish that the processing dependency ignores incoming contents, constructs an object, transfers ownership, or requires no cleanup.

## Side effects

### Direct effects

The function directly reads context-associated memory and initializes its local traversal state. It makes no explicit store into caller-owned context or selected-record memory.

That does not make the function read-only. Dependencies receive context-derived values, the context itself, or addresses of local storage and may perform additional side effects.

On the traversal path, dependency calls can observe and potentially modify local state through the supplied addresses. The processing dependency also receives the scratch-area address.

### Ordering and state visibility

The following ordering is established:

- Classification precedes all direct context reads.
- On the zero-result path, the selected-record lookup precedes context resolution.
- On the traversal path, context resolution precedes traversal-state reads.
- The first cursor read precedes the second record-pointer read.
- Each comparison precedes its corresponding processing call.
- Each processing call precedes its corresponding advancement call.
- Advancement precedes the next comparison.

There is no established transaction, rollback, lock, or snapshot mechanism. Changes made by dependencies remain relevant to later reads and calls. Global-state preservation must not be assumed.

### Ownership and lifetime

No direct allocation or deallocation is visible.

The function retains lookup results and the context-resolution result as 32-bit values. Their ownership and validity requirements are unknown.

Addresses of local state and scratch storage are supplied to dependencies. Those storage regions exist for the duration of this invocation; retaining their addresses beyond that lifetime would require a dependency contract not established here.

## Dependencies

| Opaque ID | Role established at this call site |
|---|---|
| F-9316cc774c14 | Classifies the opaque value; zero versus nonzero selects the path. |
| F-b4a9589b2ea9 | Performs each of the two lookups on the zero-result path. |
| F-e8bbeba61a78 | Resolves a 32-bit value from the context, once on either path. |
| F-4fcc06b56813 | Receives the two lookup results, in second-lookup/first-lookup order. |
| F-eac694802fc1 | Receives two local-state addresses; exactly one terminates traversal. |
| F-657c4a32ea89 | Receives scratch storage, the retained context-resolution result, and the current cursor plus 16 bytes. |
| F-357c4c54c6ea | Receives the cursor-state address after each processing call. |

These descriptions specify call-site roles, not complete dependency semantics. No dependency is assumed to preserve globals, preserve addressed state, allocate memory, release memory, or report errors in a particular way.

## Acceptance examples

The following are conditional static acceptance examples, not observed runtime results.

### Zero classification preserves combining order

Assume classification returns zero and the selector is 2. Provided address arithmetic is valid, the first lookup receives the context address plus 56 bytes.

If that lookup returns value A, context resolution returns value B, and the second lookup returns value C, the combining dependency must receive C first and A second. Traversal dependencies must not be called.

### Nonzero classification includes negative values

If classification returns negative one, the traversal path is selected. It is not treated as a local error or as equivalent to zero.

### Immediate traversal termination

Assume initialization succeeds and the first comparison returns exactly one.

The function must return without calling the processing or advancement dependencies. Context resolution and both initialization record-word reads have already occurred.

### Comparison zero does not terminate

Assume the first comparison returns zero, the first advancement completes, and the next comparison returns one.

Exactly one processing call and one advancement call occur, in that order. Treating zero as the traversal stopping result would violate the established behavior.

### Comparison mutation affects processing

Assume the comparison changes the current cursor word to address X and returns a value other than one.

Provided X plus 16 is valid, the subsequent processing call receives X plus 16 as its third argument. It must not use a cursor value cached before that comparison.

### Two initialization reads can disagree

Assume the first selected-record resolution yields address A, while the second yields address B.

The initial cursor is read from A plus 8, while the anchor is B. The function does not reconcile the difference or repeat initialization. Whether such disagreement is safe depends on the environment and dependency contracts.

### Scratch reuse

Assume two iterations are entered. Both processing calls receive the same scratch-area address. No explicit clearing or finalization occurs between them.

## Uncertainties

The following are documented_only:

- The semantic meaning of the context, selector, selected record, opaque value, anchor, and cursor.
- The actual object layouts beyond the directly accessed offsets and local-word arrangement.
- Dependency preconditions, side effects, error conventions, ownership rules, and access extents.
- Whether the comparison or advancement dependencies modify more than the first addressed word.
- Whether the scratch area represents an output object, temporary workspace, or another structure.
- The meaning of the cursor-plus-16 address.
- Behavior under arithmetic overflow, invalid addresses, concurrent mutation, or dependency failure.
- Whether traversal is guaranteed to terminate.
- Exception, nonlocal-exit, synchronization, and asynchronous execution behavior.

The repeated stride, anchor/cursor initialization, comparison, and advancement suggest record-oriented iteration. That interpretation is inferred intent only.

## Evidence

This contract is based exclusively on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence includes the branch predicates, argument widths, accessed offsets, record stride, repeated reads, dependency-call order, local-state copies, exact traversal stopping value, and absence of explicit local validation or cleanup.

Dependency implementations and authoritative type definitions were not supplied. No runtime tests have been performed. The acceptance examples describe required behavior under stated conditions; they are not test results.
