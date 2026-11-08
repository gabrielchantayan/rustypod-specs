<!-- Trusted provenance: F-5cdddd9b4e5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5cdddd9b4e5e

## Purpose

F-5cdddd9b4e5e evaluates a sequence of selector-and-value constraints against a caller-supplied record. It returns a 32-bit result representing either acceptance, `1`, or rejection, `0`.

Acceptance requires reaching a terminating selector without any preceding constraint rejecting the record. A null constraint-list pointer is accepted immediately. Rejection occurs at the first failed constraint, unsupported selector, or unrecognized selector.

The constraints are not uniformly interpreted as minimum requirements. Depending on the selector, the function requires one of four relationships:

- The record field is at least the requested signed value.
- The record field is at most the requested signed value.
- The record field has exactly the requested 32-bit value.
- The record field and requested mask share at least one set bit.

**Inferred intent:** this is a record-filtering predicate. The source does not establish the application-level meanings of the record fields, selectors, or terminating value.

## Interface

### Inputs

The function has two pointer inputs:

| Input | Role |
|---|---|
| Record pointer | Addresses a record containing fields at fixed four-byte offsets. |
| Constraint-list pointer | Addresses a sequence of selector/value pairs followed by a terminating selector, or is null. |

The supplied source represents the record and list as arrays of integer words. This contract describes the apparent 32-bit word layout: each word occupies four bytes, and each ordinary constraint occupies eight bytes. The source does not independently establish the target ABI or byte order; those remain platform prerequisites.

The pointers are not declared read-only, but the function body contains no writes through either pointer.

### Constraint-list layout

Each nonterminating entry consists of:

| Relative byte offset | Contents |
|---|---|
| `+0` | 32-bit selector |
| `+4` | 32-bit requested value or mask |

After reading an ordinary entry, traversal advances by two words, or eight bytes.

Termination uses a selector whose value equals a value obtained from an external global location. The actual terminating value is not supplied. It must not be assumed to be zero, any particular recognized selector, or any conventional API terminator.

A terminating entry requires only its selector word to be readable. Its following word is not read by the shown function.

There is no list-length argument, count field, or traversal limit.

### Record layout

Record fields are addressed by fixed offsets from the record pointer. The largest referenced offset is `+88`, covering bytes `88–91` under the four-byte-word layout. A fully accessible record therefore spans at least 92 bytes.

The function does not read the whole record. It reads only the field selected by each reached, supported constraint. No record field is needed for a null list, an immediately terminating list, or an entry that rejects without selecting a field.

### Result

The only explicit return values are:

| Result | Meaning |
|---|---|
| `1` | Null list, or termination reached after all preceding constraints passed |
| `0` | A reached constraint rejected |

There is no explicit error result distinguishing malformed input, unknown selectors, unsupported selectors, or ordinary constraint failure. Invalid memory accesses are not converted into a defined return value.

## Behavior

### Entry and termination

The function first checks whether the constraint-list pointer is null. If it is, the function returns `1` without reading the record, list contents, or terminating-value global.

For a non-null list, each entry begins with a selector read. That selector is compared with the externally stored terminating value. If they match, the function returns `1` immediately:

- The entry’s requested-value word is not read.
- No record field is read for that entry.
- The selector is not dispatched as a constraint.

Consequently, termination takes precedence over all selector-specific behavior. If the external terminating value equals a selector otherwise described as unsupported, unknown, or supported, that selector terminates the list instead of rejecting or evaluating a field.

The source places the terminating-value comparison inside traversal. It does not establish that the global value is immutable or that a single value is captured for the whole invocation. With stable global storage, the list uses one consistent terminator. Behavior under concurrent mutation is not established.

### Ordinary entry processing

If the selector is not the terminator, the function reads the following requested-value word and advances the traversal position by eight bytes. It then dispatches the selector.

This order matters for memory requirements: even an unsupported or unknown nonterminating selector reaches a requested-value read before rejection. A caller cannot rely on such a selector to reject safely when its second word is inaccessible.

Supported selectors read the corresponding record field and apply the relationship below. A passing constraint continues at the next entry. A failing constraint returns `0` immediately, without inspecting later entries.

### Selector relationships

In the table, **record** means the field at the stated byte offset, and **request** means the entry’s second word.

| Selector | Record byte offset | Passing condition |
|---|---:|---|
| `0x3020` | `0` | Signed record ≤ signed request |
| `0x3021` | `16` | Signed record ≥ signed request |
| `0x3022` | `12` | Signed record ≥ signed request |
| `0x3023` | `8` | Signed record ≥ signed request |
| `0x3024` | `4` | Signed record ≥ signed request |
| `0x3025` | `28` | Signed record ≤ signed request |
| `0x3026` | `68` | Signed record ≤ signed request |
| `0x3027` | `20` | Signed record ≥ signed request |
| `0x3028` | `24` | Exact 32-bit equality |
| `0x3029` | `32` | Exact 32-bit equality |
| `0x302A` | None | Always rejects |
| `0x302B` | None | Always rejects |
| `0x302C` | None | Always rejects |
| `0x302D` | `48` | Exact 32-bit equality |
| `0x302E` | None | Always rejects |
| `0x302F` | `56` | Exact 32-bit equality |
| `0x3030` | None | Always rejects |
| `0x3031` | `64` | Signed record ≤ signed request |
| `0x3032` | `60` | Signed record ≤ signed request |
| `0x3033` | `72` | Bitwise intersection with request is nonzero |
| `0x3034` | `76` | Exact 32-bit equality |
| `0x3035` | `88` | Exact 32-bit equality |
| `0x3036` | `84` | Exact 32-bit equality |
| `0x3037` | `80` | Exact 32-bit equality |

Every other nonterminating selector rejects.

The “always rejects” entries are conditional on not matching the external terminator. They do not read a record field.

### Signed comparisons and widths

The ordering comparisons interpret both operands as signed integers. Although the requested word is initially held as an unsigned quantity in the supplied representation, it is explicitly treated as signed for these comparisons.

Under the apparent 32-bit two’s-complement interpretation:

- `0xFFFFFFFF` represents `−1` in ordering comparisons.
- `0x80000000` represents `−2147483648`.
- `0x7FFFFFFF` represents `2147483647`.

High-bit-set requests must therefore not be treated as large positive unsigned thresholds.

The threshold comparisons are inclusive. Equal record and request values pass both the “at least” and “at most” relationships.

Exact-match selectors compare all 32 bits. Signedness does not change which bit patterns are equal.

For `0x3033`, passing requires at least one common set bit. It does not require all requested bits to be present and does not require equality. A zero requested mask always rejects. A zero record mask also always rejects.

There is no threshold addition, subtraction, scaling, normalization, or clamping. The relevant arithmetic is pointer advancement through the list; exhaustion or overflow is not checked.

### Composition and repeated entries

Constraints are evaluated in list order and combined conjunctively: every reached ordinary entry must pass before termination can produce acceptance.

Repeated selectors are evaluated repeatedly. The function does not deduplicate constraints, retain earlier field values, or resolve contradictory requests. With stable inputs, incompatible repeated constraints cause rejection when the first unsatisfied entry is reached.

An empty list consists of a terminating selector at its first word and returns `1`.

A readable but unterminated list has no guaranteed successful completion. Traversal continues until rejection, a later terminating selector, or behavior outside the valid-memory preconditions.

## Side effects

### Reads

The source-level access sequence is:

1. Check the list pointer for null.
2. For each reached entry, read its selector and compare it with the external terminating value.
3. If nonterminating, read its requested-value word.
4. For a supported selector, read its selected record field.
5. Continue or return according to the result.

These are logical source-level accesses. The supplied text does not establish machine-instruction ordering, speculative accesses, memory barriers, or volatile semantics.

Early returns prevent subsequent source-level list and record reads. Acceptance or rejection does not require examining the remainder of the list.

### Writes and ownership

The function body contains no writes to:

- The record.
- The constraint list.
- The terminating-value global.
- Any other external storage.

Traversal advances a local pointer only; it does not update a caller-owned cursor.

There are no shown allocations, deallocations, ownership transfers, resource acquisitions, or callbacks. The caller remains responsible for the input storage. The function does not retain an input pointer in visible external state.

### Preconditions and errors

For defined evaluation under the intended layout:

- Each reached selector word must be readable with suitable alignment.
- Each reached nonterminating entry must also have a readable requested-value word.
- Each supported entry must have its selected record field readable.
- The external terminating-value storage must be readable when a non-null list is processed.
- Pointer traversal must remain within the applicable platform’s valid address and object rules.

The record pointer is not checked for null. A null record can avoid dereference on paths that require no record field, but it is not generally valid for a list containing supported constraints.

No diagnostics, exceptions, error-object writes, or recovery paths are shown.

## Dependencies

No function calls are present in the supplied body. There are therefore no callable dependencies to identify by opaque function ID.

The function depends on one external data value: the terminating selector. Its value, initialization, storage width independent of the surrounding representation, mutability, and synchronization contract are not supplied.

The function also depends on the platform’s integer representation, pointer semantics, alignment requirements, and readable input storage. No preservation or ownership behavior of an unknown callee is assumed.

## Acceptance examples

These examples are conditional static expectations, not observed runtime results. Unless stated otherwise, assume four-byte words, 32-bit two’s-complement signed comparisons, valid readable storage, stable inputs, and a terminating value distinct from the selectors being evaluated.

1. **Null list:** A null constraint-list pointer returns `1`, regardless of the record pointer. No record access is required.

2. **Immediate termination:** If the first selector equals the external terminating value, the function returns `1`. The next word need not be readable for this path.

3. **Inclusive lower bound:** For `0x3021`, a record value of `8` at byte offset `16` passes requests of `7` and `8`, but rejects a request of `9`.

4. **Inclusive upper bound:** For `0x3020`, a record value of `8` at byte offset `0` passes requests of `8` and `9`, but rejects a request of `7`.

5. **Negative request:** For `0x3021`, a record value of zero passes request `0xFFFFFFFF`, because the request is interpreted as `−1`. For `0x3020`, the same field value rejects that request.

6. **Exact bit-pattern equality:** For `0x3028`, field `0xFFFFFFFF` at byte offset `24` passes request `0xFFFFFFFF` and rejects request zero.

7. **Any-bit overlap:** For `0x3033`, record mask `0x00000002` and request `0x00000006` pass. Only one requested bit is present, which is sufficient. Request `0x00000004` rejects.

8. **Zero mask:** A reached, nonterminating `0x3033` entry with request zero rejects for every record-mask value.

9. **Unsupported selector:** A nonterminating `0x302A` entry returns `0` after its requested-value word is read. It does not require a record-field read.

10. **Terminator precedence:** If the external terminating value is `0x302A`, encountering selector `0x302A` returns `1` without reading its following word. Its ordinary rejection behavior is bypassed.

11. **Short-circuit rejection:** If the first constraint fails, a later inaccessible entry is not reached by the source-level traversal.

12. **All constraints pass:** Several passing entries followed by the terminating selector return `1`. Passing entries without a reached terminator do not independently establish acceptance.

## Uncertainties

The following are **documented_only** limitations:

- The actual terminating-selector value and its application-level meaning are unknown.
- Field names, units, valid application ranges, and selector semantics beyond the demonstrated comparisons are unknown.
- The target ABI and byte order are not independently supplied. Four-byte offsets and signed boundary examples use the apparent 32-bit representation.
- The treatment of out-of-range unsigned-to-signed conversions depends on the underlying language and target interpretation; the examples use two’s-complement bit-pattern interpretation.
- Concurrent modification, memory-mapped input, volatile access requirements, and synchronization guarantees are unspecified.
- Behavior after invalid accesses, invalid pointer arithmetic, or address-space exhaustion is outside the established normal contract.
- Aliasing between inputs and the external global is not prohibited by the shown body, but no broader aliasing or concurrency contract is supplied.
- Machine-level read ordering and optimization effects cannot be established from this representation alone.

No runtime tests have been performed.

## Evidence

**Established static evidence:** the supplied body contains a null-list acceptance path, a per-entry external-terminator comparison, two-word ordinary-entry traversal, the selector relationships listed above, immediate rejection paths, and explicit returns of only `0` or `1`. It contains no function calls or external writes.

**Inferred intent:** the function acts as a conjunction-based record filter. This interpretation does not establish application-specific field meanings or ownership conventions.

**Unknowns:** platform details, the external terminator’s value and lifecycle, concurrency guarantees, and runtime behavior outside valid-memory preconditions are not resolved by the supplied source.
