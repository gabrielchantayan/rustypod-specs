<!-- Trusted provenance: F-dd82aab5ab38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd82aab5ab38

## Purpose

F-dd82aab5ab38 selects a record associated with a context and two opaque request values. It can return an immediately acceptable candidate, invoke a preparation dependency and obtain a fresh candidate, or search up to three alternative selectors.

**Established behavior:** a non-null initial candidate whose 32-bit field at byte offset 12 is nonzero is returned immediately. Otherwise, preparation is followed by a fresh lookup. A null fresh result is returned as null; a fresh result with a nonzero offset-12 field is returned directly. A fresh result with a zero offset-12 field becomes the destination for alternative searching.

On the first alternative whose offset-12 field is nonzero, the function invokes an opaque external operation with the destination, the alternative, and the numeric argument 20. After that operation returns, it clears the destination’s 32-bit field at offset 16 and returns the destination. If all three alternatives have zero predicates, it returns null.

The destination is obtained exclusively from the fresh lookup. Its address need not differ from the supplied candidate or an earlier lookup result.

**Inferred intent:** offset 12 appears to represent availability or readiness, and the external operation appears copy-like. These interpretations do not establish field semantics or the external operation’s effects.

No runtime tests have been performed.

## Interface

### Inputs

| Input | Representation supported by the evidence | Observed use |
|---|---|---|
| Context | 32-bit address-bearing value | Passed to dependencies and used to traverse a pointer chain. |
| Optional candidate | Record pointer, or null | Tested for immediate return when non-null. |
| Request value A | Opaque 32-bit value | Passed unchanged to preparation and default-selector lookups. |
| Request value B | Opaque 32-bit value | Passed unchanged to preparation and default-selector lookups. |

The supplied representation uses 32-bit words and address-bearing values. A portable calling convention, host pointer representation, and pointer-conversion rules are not established.

### Result

The result is a record pointer or null. A non-null result can be:

- The supplied candidate.
- The result of an initial default-selector lookup when no candidate was supplied.
- The result of the fresh default-selector lookup after preparation.
- That fresh result after the external operation and the offset-16 clearing write.

The fresh lookup may return the same address as either earlier candidate. “Fresh” describes a new dependency invocation, not a newly allocated or distinct record.

A non-null return does not universally guarantee that offset 12 is nonzero at return. In particular, the alternative-success path does not reread that field after invoking the external operation.

### Directly observed record layout

Offsets are measured in bytes from the record address.

| Offset | Width | Direct use |
|---|---:|---|
| 0 | 32 bits | Captured from the fresh destination before alternative searching. |
| 12 | 32 bits | Compared with zero. |
| 16 | 32 bits | Cleared after an alternative succeeds and the external operation returns. |

No direct interpretation of offsets 4 or 8 is established. The numeric argument 20 passed to the external operation does not, by itself, establish a 20-byte memory-access contract.

### Default-selector traversal

Each default-selector lookup obtains its selector through these dependent reads, in order:

1. Read a 32-bit address-bearing value at context offset 8.
2. Read another 32-bit address-bearing value at offset 20 from that address.
3. Read an 8-bit value at offset 89 from the second address.

The selector is loaded as one byte. Its semantic domain and the dependency’s parameter-extension rules are not established.

### Preconditions for normal completion

Requirements depend on the path taken:

- Any candidate whose predicate is inspected must permit a 32-bit read at offset 12.
- Each required context traversal must permit the two address-bearing reads and the final byte read.
- A destination entering alternative searching must permit a 32-bit read at offset 0.
- Every alternative result that reaches predicate inspection must permit a 32-bit read at offset 12. There is no alternative-result null check.
- Every selector-table byte actually examined must be readable.
- The destination must remain valid and writable for a 32-bit store at offset 16 after the external operation returns.
- Dependencies must return normally for execution to proceed past their calls.

Alignment requirements, address-space limits, and behavior on invalid storage are **documented_only**. The immediate-return path does not require readable context-chain or selector-table storage.

## Behavior

### Initial candidate evaluation

If the optional candidate is non-null, the function reads its 32-bit field at offset 12.

- A nonzero value causes immediate return of that exact pointer.
- A zero value leads to preparation.
- No initial default-selector lookup is performed for a supplied candidate, regardless of its predicate.

If the optional candidate is null, the function obtains the default selector through the context traversal and invokes F-4a95613e1504 with these arguments, in order:

1. Context.
2. The loaded selector byte.
3. Request value A.
4. Request value B.
5. Zero.

If the result is null, execution proceeds to preparation without reading a record predicate. If it is non-null, the function reads its 32-bit field at offset 12. A nonzero value causes immediate return; a zero value leads to preparation.

An unsuccessful initial candidate does not determine the later destination. The function obtains that destination from a separate lookup after preparation. This does not imply address inequality: the later lookup may return the same record.

### Preparation and fresh lookup

The function invokes F-8a57f113fd9c with context, request value A, and request value B. Its return value does not control subsequent behavior.

After preparation returns, the function traverses the complete context pointer chain again. It does not reuse a previously loaded selector or intermediate address. It then invokes F-4a95613e1504 with context, the newly loaded selector, request value A, request value B, and zero.

The fresh result determines the next action:

- **Null:** return null.
- **Non-null, offset-12 field nonzero:** return the fresh result.
- **Non-null, offset-12 field zero:** use the fresh result as the alternative-search destination.

Preparation may change the context chain, selector, lookup state, or earlier candidate memory. Its preservation of any of those is not assumed.

### Capturing alternative-search inputs

For a fresh destination with a zero predicate, the function performs these actions in order:

1. Read the destination’s 32-bit field at offset 0.
2. Invoke F-1cb585febd84 with that captured value.
3. Capture the dependency’s 32-bit result.
4. Read a global 32-bit address-bearing value identifying the selector-table base.

The destination’s first word and the transformation result are each captured once. Neither is recomputed for later alternatives.

The selector-table base is acquired after F-1cb585febd84 returns. Consequently, a change that dependency makes to the global base can affect this search.

### Alternative search

The function examines up to three selector-table positions: byte offsets 0, 1, and 2, in ascending order.

At each reached position, it reads the selector byte and invokes F-4a95613e1504 with:

1. Context.
2. That selector byte.
3. The captured destination word from offset 0.
4. The captured result of F-1cb585febd84.
5. Zero.

The third and fourth arguments are therefore different in origin from the request values used by default-selector lookups, even if their numeric values happen to coincide.

After each lookup returns, the function directly reads the returned address’s 32-bit field at offset 12. It does not first test the returned address for null.

- A zero predicate advances to the next position, if one remains.
- A nonzero predicate stops the search and starts the success sequence.
- Three zero predicates cause null to be returned.

Exactly three iterations occur only when neither of the first two alternatives succeeds and execution continues normally through the third. A first success prevents reads and lookups for subsequent positions.

Selector bytes are read as their positions are reached, not captured together in advance. Dependencies can therefore affect later selector values by modifying unread table bytes. The table-base address itself is captured once and is not reread during the search.

### Alternative-success sequence

For the first alternative with a nonzero predicate, the function invokes F-000000000001 with:

1. The fresh destination pointer.
2. The successful alternative pointer.
3. The numeric value 20.

F-000000000001 is an opaque identifier used here for the external dependency not identified by the supplied dependency map.

After that operation returns, the function:

1. Writes a 32-bit zero at destination offset 16.
2. Returns the destination pointer.

The external operation’s return value is ignored. There is no explicit success-status check and no reread of the destination’s offset-12 field.

The successful alternative is not substituted as the return pointer. It can nevertheless have the same address as the destination; no identity or overlap check is performed.

### Exhaustion

If all three inspected alternatives have zero predicates, the function returns null. It neither invokes F-000000000001 nor performs its explicit offset-16 clearing write on this path.

This return does not roll back preparation, lookup effects, transformation effects, or other dependency mutations.

## Side effects

### Direct reads and writes

The only explicit store established for F-dd82aab5ab38 is the 32-bit zero written at destination offset 16 after alternative success.

Direct reads can include:

- Initial and fresh candidate predicates.
- The context pointer chain and default-selector byte.
- The fresh destination’s first 32-bit word.
- The global selector-table base.
- Up to three selector-table bytes.
- The predicate of each returned alternative.

There is no directly established allocation, deallocation, reference-count change, or ownership transfer. Dependencies may perform such actions; their behavior is unknown.

### Observable ordering

The following ordering constraints are established:

- Initial candidate evaluation precedes preparation.
- Preparation precedes the fresh context traversal and lookup.
- The fresh predicate read precedes capture of the destination’s first word.
- That capture precedes F-1cb585febd84.
- F-1cb585febd84 precedes the global table-base read.
- Each reached selector-byte read precedes its corresponding lookup.
- Each alternative predicate read follows its lookup.
- The successful predicate read precedes F-000000000001.
- The offset-16 store follows that operation’s normal return.

These constraints do not imply that dependencies preserve previously read memory, globals, or record lifetimes.

### Aliasing, ownership, and lifetime

The supplied candidate, initial lookup result, fresh destination, and alternative records are not required by explicit checks to have distinct addresses. Records may also overlap. Whether the external operation supports any particular overlap is **documented_only**.

Captured scalar values remain the arguments used for alternative lookups even if a dependency changes the memory from which they originated. Conversely, later predicate and selector-byte reads observe memory at the time those reads occur.

The function provides no established ownership guarantee for returned pointers. Whether a result is borrowed, owned, retained, or subject to release is **documented_only**. Normal completion on the success path requires the destination to survive dependency calls and remain writable for the final store.

## Dependencies

| Opaque ID | Observed invocation | Boundary and unknowns |
|---|---|---|
| F-4a95613e1504 | Five-argument lookup for default and alternative selectors | Default results are null-checked; alternative results are not. Allocation, mutation, ownership, and lifetime behavior are unknown. |
| F-8a57f113fd9c | Preparation with context and both request values | Return value ignored. Subsequent selector traversal and lookup occur after it returns. |
| F-1cb585febd84 | One captured 32-bit destination word as input; 32-bit result used later | Called once on entry to alternative searching. Other effects are unknown. |
| F-000000000001 | Destination, successful alternative, and 20 | Return value ignored. Exact memory effects, length units, overlap rules, failure reporting, and ownership effects are unknown. |

No dependency is assumed to preserve globals, retain storage, release storage, or leave record contents unchanged.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Each assumes valid storage and normally returning dependencies for the path described.

1. **Supplied candidate passes immediately.**  
   A non-null supplied candidate has a nonzero offset-12 field. Return that exact pointer without dependency calls, context traversal, or direct writes.

2. **Initial lookup passes.**  
   No candidate is supplied. The initial default lookup returns a readable record with a nonzero predicate. Return it without preparation or alternative searching.

3. **A supplied zero-predicate candidate skips initial lookup.**  
   A non-null supplied candidate has a zero predicate. Invoke preparation next; do not perform a default lookup before preparation.

4. **Preparation changes the default selector.**  
   If preparation changes the context chain or selector byte, the fresh lookup uses the selector obtained by the post-preparation traversal.

5. **Fresh lookup reuses an earlier address.**  
   An earlier candidate has a zero predicate. After preparation, the fresh lookup returns that same address. That address is accepted as the fresh result and, if its newly read predicate is zero, becomes the alternative-search destination.

6. **Fresh lookup returns null.**  
   Return null without reading the destination’s first word, invoking F-1cb585febd84, or accessing the alternative-selector table.

7. **First alternative succeeds.**  
   Inspect only table offset 0. Invoke F-000000000001 with the fresh destination, the first alternative, and 20. After normal return, clear destination offset 16 and return the destination. Do not read table offsets 1 or 2.

8. **Second alternative succeeds.**  
   The first predicate is zero and the second is nonzero. Perform two alternative lookups, then the success sequence. Do not read the third selector byte.

9. **Third alternative is reached.**  
   The first two predicates are zero. If execution continues normally, inspect the third alternative. A nonzero predicate takes the success path; a zero predicate returns null. Thus three iterations do not necessarily mean exhaustion.

10. **All alternatives fail their predicates.**  
    Three readable alternative records each have a zero predicate. Return null without invoking F-000000000001 or directly clearing destination offset 16.

11. **A later table byte changes.**  
    If the first alternative lookup changes the byte at captured table-base offset 1 and the first predicate is zero, the second lookup uses the changed byte. Changing only the global table-base value does not redirect this search.

12. **An alternative lookup returns null.**  
    The function next attempts the offset-12 predicate read; it does not convert null into a clean miss. Normal completion is not specified.

13. **The external operation leaves the destination predicate zero.**  
    If the operation returns normally and the destination remains writable, the function still clears offset 16 and returns the destination. It performs no post-operation predicate validation.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the context, request values, selectors, and record fields.
- Dependency internals, state changes, ownership, allocation, and release behavior.
- Whether F-000000000001 copies memory, and whether 20 denotes bytes.
- Record sizes, table bounds beyond accessed positions, alignment, and byte order.
- Selector argument extension and the complete ABI.
- Invalid-access, exception, fault, or non-returning behavior.
- Concurrent mutation and synchronization guarantees.

The loop index remains within 0 through 3 during normal control flow; its increment does not approach a 32-bit overflow boundary. The accessed selector positions are only 0 through 2. Pointer-offset overflow handling and address validity checks are not established.

## Evidence

**Established static evidence:** the supplied representation supports the branch conditions, separate initial and fresh lookups, argument ordering, 32-bit record accesses, 8-bit selector loads, pointer-chain offsets, null-check placement, captured alternative-search inputs, early termination, numeric external-operation argument 20, and final 32-bit zero store.

The fresh lookup is the exclusive source of the alternative-search destination, but no comparison requires its address to differ from an earlier candidate. The alternative search has an upper bound of three iterations and stops at the first nonzero predicate.

**Inferred intent:** readiness, preparation, transformation, fallback selection, and copying are descriptive interpretations, not additional guarantees.

**Unknowns:** callee contracts, ownership, field semantics, runtime validity, and external-operation effects are not resolved by this evidence.

No runtime tests, dynamic traces, or independent implementation inspection have been performed.
