<!-- Trusted provenance: F-4c6f3fb02f8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c6f3fb02f8b

## Purpose

F-4c6f3fb02f8b routes an input record through callbacks associated with a context. It optionally invokes a preliminary dependency, selects and invokes one of two primary callbacks, and conditionally invokes a final callback. Any invoked stage returning a nonzero result terminates processing immediately, with that result returned unchanged.

The established behavior is a short-circuit dispatch sequence. Interpreting the record as an event, the context as an object, or the final callback as a notification is inferred intent, not an established domain contract. Numeric record classifications and callback-table offsets determine behavior without requiring those interpretations.

Changes made by a returning dependency or callback can affect subsequent routing. The function reads relevant record and context fields at the stage where they are needed rather than retaining an entry-time snapshot.

## Interface

### Inputs

The function accepts two arguments:

- **Context reference:** A reference to storage whose first field supplies a callback-table address. A separate context field at byte offset `0x50` participates in a conditional comparison.
- **Record address:** An address identifying storage whose classification field is at byte offset `0x04`. Depending on that classification, the field at byte offset `0x08` is used either as a nested-record address or as a comparison value.

The supplied representation uses 32-bit integer fields and a 32-bit address-bearing model. The layouts below describe that model; they are not a portable declaration for systems with different integer or pointer widths.

### Relevant layouts

| Storage | Byte offset | Interpretation |
|---|---:|---|
| Context | `0x00` | Callback-table address |
| Context | `0x50` | 32-bit value compared with the record’s field at `0x08` |
| Record | `0x04` | 32-bit classification |
| Record | `0x08` | Nested-record address for classification `0x16`; comparison value for classification `0x0F` |
| Nested record | `0x04` | 32-bit classification |
| Callback table | `0x0E0` | Conditional final callback address |
| Callback table | `0x10C` | Primary callback address for one specific nested classification |
| Callback table | `0x110` | Default primary callback address |

No meaning is established for unlisted fields. The record’s field at `0x08` does not have one universal interpretation: its use depends on the classification read at the relevant stage.

### Output

The return value is a 32-bit integer:

- The first nonzero result from an invoked dependency or callback is returned unchanged.
- Otherwise, the function returns zero.

There is no normalization to a Boolean result and no distinction between positive and negative nonzero results. The meaning of zero or any particular nonzero value is not established beyond its effect on control flow.

### Preconditions

For ordinary execution, each address actually accessed must identify readable storage of the required size. Each selected callback-table entry must identify a callable target compatible with receiving the original context reference and record address and returning a 32-bit integer.

These requirements are conditional:

- The preliminary dependency is invoked only for the initial classification `0x15`.
- The nested address is dereferenced only when the primary-selection classification is `0x16`.
- The context field at `0x50` is read only when the post-primary classification is `0x0F`.
- The final callback entry is read only when that classification and the subsequent equality comparison both qualify.

The function performs no explicit null, bounds, accessibility, or callback-validity checks. An invalid address on an accessed path has no defined error-return behavior in the supplied evidence.

## Behavior

### 1. Initial classification and preliminary stage

The function first reads the 32-bit classification at record offset `0x04`.

If that value is `0x15`, it calls F-8dcd89a8dd58 with the original context reference and record address.

- A nonzero result is returned immediately. No primary or final callback is selected or invoked afterward.
- A zero result permits processing to continue.

For any other initial classification, this dependency is skipped.

After the preliminary stage is skipped or returns zero, the function reads the record classification again. This second read governs primary callback selection. It is not replaced by the value from the initial read.

Consequently, if F-8dcd89a8dd58 changes the record classification and returns zero, the changed classification affects the next stage.

### 2. Primary callback selection

The function selects exactly one primary callback if execution reaches this stage and the required accesses succeed.

Selection proceeds from the newly read classification:

- If it is not `0x16`, the function selects the callback-table entry at `0x110`.
- If it is `0x16`, the function reads the address-bearing field at record offset `0x08`, then reads the 32-bit classification at offset `0x04` within that nested storage.
  - A nested classification of `0x2A` selects callback-table entry `0x10C`.
  - Any other nested classification selects callback-table entry `0x110`.

The nested record is not inspected for classifications other than `0x16`. There is no recursive traversal: this function performs one conditional nested-classification read.

After determining which entry is required, the function reads the callback-table address from context offset `0x00`, then obtains the callback address from the selected table entry. Thus, table selection occurs after any preliminary dependency call and after the record reads used to choose the entry.

The chosen callback is invoked with the original context reference and original record address. The nested record address is not substituted as an argument.

### 3. Primary result handling

If the primary callback returns nonzero, that exact result is returned immediately. The function does not perform its post-primary record classification read or invoke the final callback.

If the primary callback returns zero, processing continues.

The primary callback can therefore suppress all later processing by returning any nonzero 32-bit value, independently of the record classification.

### 4. Post-primary eligibility check

After a zero primary result, the function reads the record classification again from offset `0x04`.

If this value is not `0x0F`, the function returns zero without reading the record’s comparison field or the context’s field at `0x50` for this check.

If the classification is `0x0F`, the function:

1. Reads the 32-bit value at record offset `0x08`.
2. Reads the 32-bit value at context offset `0x50`.
3. Compares those values for equality.

This is an equality comparison, not an ordering comparison. The supplied behavior does not establish a domain meaning for either value.

A mismatch causes the function to return zero without looking up or invoking the final callback.

Because these reads occur after the primary callback returns, changes made by that callback to the classification or either compared field can affect final-stage eligibility.

### 5. Conditional final callback

If the post-primary classification is `0x0F` and the two compared values are equal, the function reads the callback-table address from context offset `0x00` again. It then reads the callback address at table offset `0x0E0`.

This is a fresh table lookup, not reuse of the table address used for primary dispatch. A primary callback that changes the context’s table address and returns zero can therefore affect the final callback target.

The final callback receives the original context reference and record address.

- A nonzero result is returned unchanged.
- A zero result leads to a zero return from F-4c6f3fb02f8b.

The function performs no further explicit record or context field reads after this callback returns.

### Read ordering and arithmetic boundaries

The established logical read order is:

1. Initial record classification.
2. Optional preliminary dependency invocation.
3. Record classification for primary selection.
4. Conditional nested-record address and nested classification.
5. Context callback-table address and selected primary entry.
6. Primary callback invocation.
7. On a zero primary result, record classification for final eligibility.
8. If that classification is `0x0F`, record comparison value, then context comparison value.
9. On equality, context callback-table address again, then final callback entry.
10. Final callback invocation.

All numeric classifications are exact comparisons: `0x15`, `0x16`, `0x2A`, and `0x0F` are not ranges or masks.

The function uses fixed-offset address calculations. It contains no explicit overflow checks, address-wrap handling, or recovery for invalid computed addresses. The source-level sequence does not establish hardware memory-ordering guarantees, atomicity, or safe concurrent mutation.

## Side effects

The function itself contains no explicit writes to the supplied record, context, callback table, or nested storage. It contains no explicit allocation, deallocation, copying, ownership transfer, or global-state update.

Its invoked dependency and callbacks may have side effects. Their implementations are not supplied, so this contract does not assume that they:

- Preserve record, context, table, or global state.
- Preserve ownership or lifetime of any referenced storage.
- Avoid allocation or release.
- Return normally.
- Leave the context’s callback-table address unchanged.

Later reads remain dependent on valid storage. If a callee invalidates storage that this function subsequently accesses, there is no visible local protection or recovery mechanism.

If a callee changes state but returns nonzero, the immediate return prevents this function’s later stages; it does not roll back the callee’s changes. No transactional behavior is established.

## Dependencies

| Dependency | Invocation condition | Arguments | Result handling |
|---|---|---|---|
| F-8dcd89a8dd58 | Initial record classification equals `0x15` | Original context reference and record address | Nonzero returned immediately; zero continues |
| Dynamically selected target at table offset `0x10C` | Primary-selection classification is `0x16` and nested classification is `0x2A` | Original context reference and record address | Nonzero returned immediately; zero permits final-stage checking |
| Dynamically selected target at table offset `0x110` | All other primary-selection cases | Original context reference and record address | Same primary-result handling |
| Dynamically selected target at table offset `0x0E0` | Primary result is zero, post-primary classification is `0x0F`, and compared fields are equal | Original context reference and record address | Nonzero returned unchanged; otherwise zero |

The dynamic targets have no established opaque function identities in the supplied dependency map. Their table-entry locations identify dispatch positions, not independent semantic guarantees.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not reports of executed tests.

1. **Preliminary short-circuit.**  
   Given an initial classification of `0x15`, if F-8dcd89a8dd58 returns `-7`, the function returns `-7`. Neither primary callback nor the final callback is invoked.

2. **Preliminary mutation affects routing.**  
   Given an initial classification of `0x15`, suppose F-8dcd89a8dd58 returns zero after changing the classification to `0x16` and installing a valid nested address whose classification is `0x2A`. The primary callback is selected from table offset `0x10C`, using the context’s table address read after that dependency returns.

3. **Specific nested match.**  
   Given a primary-selection classification of `0x16` and nested classification `0x2A`, the primary callback comes from `0x10C`. It receives the outer record address, not the nested address.

4. **Nested mismatch uses default entry.**  
   Given a primary-selection classification of `0x16` and any readable nested classification other than `0x2A`, the primary callback comes from `0x110`.

5. **Irrelevant nested storage is not accessed.**  
   Given a primary-selection classification other than `0x16`, an unusable value at record offset `0x08` does not itself require a nested dereference. The default primary entry is selected. Other accesses required by the path must still be valid.

6. **Primary nonzero result suppresses final processing.**  
   If the primary callback returns `23`, the function returns `23`, even if the record would otherwise qualify for the final callback.

7. **Matching final-stage values.**  
   If the primary callback returns zero, the subsequently read classification is `0x0F`, and the record and context comparison values are equal, the final callback is invoked. If it returns `9`, the function returns `9`.

8. **Final-stage mismatch.**  
   If the primary callback returns zero and the post-primary classification is `0x0F`, but the compared values differ, the final callback is not invoked and the function returns zero.

9. **Primary mutation enables final processing.**  
   If the primary callback returns zero after changing the record classification to `0x0F` and making the compared values equal, final processing is eligible regardless of the earlier classification.

10. **Primary mutation changes the final table.**  
    If the primary callback replaces the context’s table address, returns zero, and leaves the final-stage conditions satisfied, the final target is read from offset `0x0E0` in the replacement table.

11. **All reached stages return zero.**  
    If every invoked stage returns zero and all required accesses succeed, the function returns zero, whether the final callback is skipped or invoked.

## Uncertainties

The following remain **documented_only**:

- The semantic names of the record classifications, comparison fields, and callback roles.
- The success, failure, handled, or continuation meanings associated with return values.
- The implementations and side effects of F-8dcd89a8dd58 and all dynamically selected targets.
- Ownership, lifetime, and mutability rules for the supplied storage.
- The exact calling convention, full surrounding layouts, alignment requirements, and runtime address-space constraints.
- Whether concurrent or reentrant changes are supported.
- Behavior following invalid memory access, an invalid callback target, or a callee that does not return normally.
- Whether the represented 32-bit types and address-bearing fields exactly match the original binary interface.

The fixed offsets and 32-bit operations are established by the supplied representation. A portable ABI or a stronger memory-safety contract is not.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping. The representation establishes the branch conditions, fixed-offset reads, callback selection, argument forwarding, rereads after calls, and exact nonzero-result propagation described above.

“Preliminary,” “primary,” and “final” name stages by execution order. They do not assert undocumented domain meanings.

No implementation access, runtime execution, memory tracing, or tests were performed. The acceptance examples specify expected behavior conditional on valid accesses and the stated callee behavior; they are not empirical verification.
