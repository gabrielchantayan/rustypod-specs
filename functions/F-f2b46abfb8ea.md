<!-- Trusted provenance: F-f2b46abfb8ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2b46abfb8ea

## Purpose

F-f2b46abfb8ea resolves a small unsigned selector into a byte-sequence pointer, then searches an object-associated pointer table for the first qualifying entry.

A table entry qualifies only when it is non-null, its first byte equals the resolved sequence’s first byte, and an external comparison dependency returns zero for the entry and the resolved sequence, in that argument order. The function returns the entry’s zero-based index. If the selector is outside the accepted range or the search completes without a qualifying entry, it returns the 32-bit value `0xFFFFFFFF`.

The established behavior is a resolver-assisted search. Interpreting it as a textual name lookup is inferred intent, not a proven contract: the resolver’s mapping and the comparison dependency’s complete semantics are unavailable.

## Interface

### Inputs

The function takes two machine-level arguments:

- **Object address:** a 32-bit address identifying storage containing the fields described below. The presented signature represents this address as a signed integer, but its observable use is as a base address.
- **Selector:** an unsigned 32-bit value. Only values from 0 through 255, inclusive, enter the lookup path.

For an out-of-range selector, the function returns the failure sentinel without dereferencing the object address. Consequently, the object’s validity is irrelevant to that path.

For an accepted selector, the caller must provide valid storage and callable dependencies for every access actually reached. The function performs no explicit validation of the object, its nested pointers, the resolver address, or the resolved pointer.

### Object-associated layout

The supplied body establishes these byte offsets and access widths:

| Location | Width | Role |
|---|---:|---|
| Object + `0x554` | 32 bits | Address of a lookup descriptor |
| Object + `0x558` | 32 bits | Unsigned search count |
| Object + `0x55C` | 32 bits | Base address of a table of 32-bit entry pointers |
| Descriptor + `0x14` | 32 bits | Indirect resolver address |
| Descriptor + `0x18` | 32 bits | Base address of a selector-to-value table |
| Selector-value table + selector × 2 | 16 bits | Value supplied to the resolver |
| Entry-pointer table + index × 4 | 32 bits | Candidate pointer |

The established layout uses four-byte addresses, two-byte selector-table elements, and four-byte candidate-pointer elements. It does not establish the full size, ownership, alignment requirements, or purpose of either object or descriptor.

The descriptor address is obtained through the object when accessing the resolver and selector-value table. The body does not create a persistent descriptor snapshot.

### Output

The return value is unsigned and 32 bits wide:

- A zero-based index for the first qualifying entry.
- `0xFFFFFFFF` when the selector is at least 256 or no entry qualifies before the search ends.

The sentinel has the same bit pattern as signed negative one, but the declared result is unsigned.

A successful result cannot equal `0xFFFFFFFF`. To inspect an entry, its index must be strictly less than an unsigned 32-bit count; no such count can exceed `0xFFFFFFFF`.

## Behavior

### Selector gating

The selector is compared as an unsigned 32-bit quantity against 256.

When it is 256 or greater, the function immediately returns `0xFFFFFFFF`. This path performs no object-field reads, selector-table reads, resolver invocation, candidate-table reads, or comparison calls.

If an external caller supplies the bit pattern associated with signed negative one, the function receives it as `0xFFFFFFFF` and rejects it through the same range check. There is no separate negative-selector case.

### Resolution

For an accepted selector, the function obtains an indirect resolver through the descriptor associated with the object. It also obtains the selector-value table through that descriptor and reads the two-byte element selected by the input.

The element offset is selector multiplied by two. Because the selector is restricted to 0–255, this offset ranges from zero through 510 and does not overflow a 32-bit quantity. The selected read covers two bytes starting at that offset.

The resolver is invoked once, with the selected 16-bit value as its sole visible argument. Its returned address becomes the resolved sequence pointer used throughout the search.

The source establishes a 16-bit load but does not establish whether the resolver’s actual parameter type is signed, unsigned, or another two-byte representation. Any calling-convention extension beyond those 16 bits is undocumented.

Both the resolver address and its argument must be obtained before the resolver is invoked. The supplied expression does not establish a portable order between the independent reads used to obtain those two operands.

Resolution occurs before the search count is checked. An accepted selector therefore invokes the resolver even when the eventual count is zero. If the resolver changes the object’s count or entry-table field, the subsequent search uses the state it reads after that call.

There is no explicit null check on the resolver’s result.

### Search and qualification

The search begins at index zero. Before each candidate visit, the current index is compared with the unsigned 32-bit count read from the object.

If the index is not less than that count, the function returns `0xFFFFFFFF`.

Otherwise, the function reads the current entry-table base from the object and obtains the four-byte pointer at the current index. It then applies these gates in order:

1. A null candidate is skipped.
2. A non-null candidate has its first byte read.
3. The first byte at the resolved pointer is read for the equality check.
4. Only if those bytes are equal is the external comparison dependency invoked.
5. A comparison result of zero causes an immediate return of the current index.

The comparison dependency receives the candidate pointer first and the resolved pointer second. A nonzero result causes the search to continue, regardless of its sign or magnitude.

The first-byte gate is exact byte equality. No case conversion, normalization, encoding conversion, or locale processing is visible in the function itself. A comparison dependency with broader equivalence rules cannot qualify a candidate whose first byte fails this preliminary gate.

The supplied expression does not establish the relative evaluation order of the two first-byte reads. It does establish that neither occurs for a null candidate, and that both precede any comparison call for that candidate.

After a skipped or nonmatching candidate, the index increases by one and the next count check occurs. Once a candidate qualifies, no later entry is inspected.

### Repeated reads and dependency effects

The count is re-read at each loop condition. The entry-table base is re-read for each visited index. The function does not visibly snapshot either field for the full search.

The resolved pointer itself is retained from the single resolver call. Its first byte is nevertheless read again for each non-null candidate. The function does not visibly cache that byte across iterations.

These distinctions matter if a dependency changes reachable state:

- A comparison call that changes the count can affect the next loop condition.
- A comparison call that changes the entry-table base can affect the next candidate lookup.
- A comparison call that changes the resolved sequence can affect a later first-byte check.
- A zero comparison result returns immediately, without another count or table-base read.

None of these observations guarantees safe concurrent mutation. Synchronization, atomicity, and behavior under data races are not established.

### Arithmetic and boundaries

The selector-table offset is bounded and cannot overflow under the accepted selector range.

The candidate-table offset uses four times a 32-bit index. Unlike the selector offset, this arithmetic is not protected by a small range check. Under the represented 32-bit address arithmetic, large indices can wrap the scaled offset or the resulting address. The function contains no overflow detection or table-capacity validation.

Normal use therefore requires a count and table layout for which every reached pointer access addresses the intended storage. The count alone does not prove that storage exists.

The index does not need to wrap during a returning search. The largest index that can enter an iteration is `0xFFFFFFFE`; at `0xFFFFFFFF`, no unsigned 32-bit count can satisfy the strict less-than condition. Dependency calls can still prevent completion by not returning.

## Side effects

The function body contains no explicit writes to the object, descriptor, selector-value table, candidate-pointer table, or sequence bytes. Its direct observable actions are reads, one conditional resolver call, zero or more comparison calls, and a return.

This does **not** establish that the overall operation is read-only. The dependencies may write memory, alter external state, retain pointers, release storage, or fail. Their implementations and ownership contracts are unavailable.

The body performs no explicit allocation, deallocation, reference-count adjustment, ownership transfer, logging, or error recording. It returns an index rather than a pointer to either sequence.

The failure sentinel reports only the two ordinary failure outcomes visible here: selector rejection and search exhaustion. There is no visible mechanism translating invalid memory access, an invalid callable address, or a dependency failure into that sentinel.

A null resolved pointer is not automatically fatal on every accepted-selector path. If the count is zero, or every visited candidate is null, the body does not dereference it. Once a non-null candidate reaches the byte-equality gate, a readable resolved address is required.

## Dependencies

Only the target function has a supplied canonical opaque ID. The following opaque labels identify dependency roles locally; they are not asserted to be externally assigned identities.

| Opaque label | Invocation and observed contract | Unknowns |
|---|---|---|
| **F-unknown-01** | Indirect resolver selected through descriptor offset `0x14`. Called once for an accepted selector, with one value read from a 16-bit table element. Its result is used as a byte-addressable pointer. | Actual parameter type, calling convention details, mapping semantics, result validity and lifetime, ownership, mutations, and failure behavior. |
| **F-unknown-02** | Comparison dependency called only after a non-null candidate passes the first-byte gate. Receives candidate first and resolved pointer second. A zero result qualifies the candidate. | Whether it performs conventional string comparison, termination requirements, extent of reads, encoding rules, mutations, pointer retention, ownership effects, and failure behavior. |

Neither dependency may be assumed to preserve object state, global state, or pointed-to storage. Neither may be assumed to allocate or release memory.

## Acceptance examples

These are conditional behavioral examples derived from static evidence. They are not runtime test results.

1. **Rejected selector bypasses object access.**  
   Given selector 256, the result is `0xFFFFFFFF`. Neither dependency is called, even if the supplied object address would be invalid to dereference.

2. **Largest accepted selector.**  
   Given selector 255 and valid reachable storage, the resolver receives the 16-bit value whose first byte is at selector-table offset 510. The selector is not reduced modulo any smaller range.

3. **Empty search still resolves.**  
   Given selector 12, a returning resolver, and a count of zero when the first loop condition is evaluated, the resolver is called once and the function then returns `0xFFFFFFFF`. No candidate pointer is read.

4. **Null entries do not trigger sequence reads.**  
   Given a stable count of two and two null candidate pointers, both slots are visited and the result is `0xFFFFFFFF`. The resolved pointer is not dereferenced by the search body, and the comparison dependency is not called.

5. **First-byte mismatch prevents comparison.**  
   Given one non-null candidate whose first byte differs from the resolved sequence’s first byte, the candidate is rejected without invoking F-unknown-02. This remains true even if that dependency would otherwise return zero for those sequences.

6. **First qualifying index wins.**  
   Given stable search state, a null entry at index zero, a first-byte mismatch at index one, and a first-byte match with comparison result zero at index two, the result is two. No later entry is inspected.

7. **Nonzero comparison is not success.**  
   Given equal first bytes but comparison result negative one, that entry is rejected. A positive nonzero result has the same rejection effect.

8. **A dependency can shorten the remaining search.**  
   Given a comparison at index zero that returns nonzero and changes the object’s count to zero, the next loop condition ends the search with `0xFFFFFFFF`, assuming the dependency returns normally and subsequent accesses remain valid.

9. **The next table lookup uses the then-current base.**  
   Given a comparison that returns nonzero after replacing the entry-table base, the next visited index is fetched from the replacement table, provided the next count check permits that visit. Earlier candidate pointers are not revisited.

## Uncertainties

The following are **documented_only** because the supplied body does not resolve them:

- The domain meaning of the selector and its 16-bit mapped value.
- Whether the resolved and candidate sequences are null-terminated strings.
- The complete equivalence relation implemented by F-unknown-02.
- Valid allocation sizes, alignment constraints, and storage lifetimes.
- The resolver’s null-result policy and whether callers normally permit null results.
- Ownership, retention, allocation, or release behavior of either dependency.
- Whether dependency-induced changes to search state are intended usage.
- Thread-safety, synchronization, reentrancy guarantees, and atomicity.
- Fault behavior for invalid addresses or unsupported callable targets.
- Exact dependency ABI details beyond the visible argument use.

A likely intent is to map an eight-bit selector into a textual value and locate the corresponding entry in an object-owned collection. That interpretation is not required for the established control flow and must not replace the narrower evidence-based contract.

## Evidence

This contract is based solely on static inspection of the supplied function body. No runtime tests have been performed.

Established evidence includes the unsigned selector threshold, the listed offsets and access widths, the single resolver call before the search, ascending index traversal, repeated count and table-base reads, null-candidate filtering, the first-byte gate, comparison argument order, zero-result acceptance, and the `0xFFFFFFFF` fallback.

Dependency semantics, ownership, allocation validity, textual interpretation, and runtime fault behavior are not established. The acceptance examples are conditional consequences of the visible behavior, not claims about an exercised implementation.
