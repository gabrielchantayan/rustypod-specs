<!-- Trusted provenance: F-d262b07bab3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d262b07bab3c

## Purpose

F-d262b07bab3c performs an ordered sequence of byte writes and dependency calls, followed by a bounded search for a nonzero reference and a conditional indirect call.

Its established behavior is to:

- Write zero to destination byte offset 0.
- Call F-13ef3c7b2c3d with an externally supplied value and integer zero.
- For a zero selector, obtain an object from F-cd96e43d3119, invoke an operation selected through that object with integer zero, and then write one to destination byte offset 3.
- Examine at most two 32-bit table entries, separately evaluating the external table-base source for each examined entry.
- If an entry is nonzero, use that retained entry value as a record address and dispatch through its offset-16 object field. That field is referenced separately for callable lookup and for the receiver argument.

Initialization and activation are possible interpretations, not established semantics. The function does not report whether any dependency succeeded.

## Interface

### Inputs

| Input | Representation | Established use |
|---|---|---|
| Destination | Address supporting byte writes | Receives zero at byte offset 0; conditionally receives one at byte offset 3. |
| Selector | 32-bit integer | Compared with zero. Zero enables the optional object operation and offset-3 write; every nonzero value skips them. |

The function does not directly read destination contents. It does not obtain the search table or either dispatched object from a destination field.

Direct destination-access preconditions are:

- Destination offset 0 must be writable whenever execution begins.
- Destination offset 3 must be writable if the selector is zero and execution reaches that write.
- No direct access is made to destination offsets 1 or 2.
- For a nonzero selector, there is no direct access to destination offset 3.

A writable four-byte region covers both possible destination writes, but does not establish validity of external references or satisfy unknown dependency requirements. The direct writes are byte-sized and do not themselves require wider alignment.

There is no explicit destination null check, bounds check, or alignment check.

### Output

The function has no defined return value.

The results of F-13ef3c7b2c3d and both indirect operations are ignored. The result of F-cd96e43d3119 is used as an object address.

Normal return occurs after the applicable calls and accesses complete. Invalid accesses or dependencies that fail to return normally can prevent completion; no recovery behavior is established.

### External storage and layouts

The following labels describe external storage roles rather than application-level identities:

- **G-1:** supplies the first argument to F-13ef3c7b2c3d.
- **G-2:** supplies the base used to address each examined search-table entry.

The supplied representation uses 32-bit integer words for table entries and object-reference traversals. Offsets below are byte offsets.

| Referenced entity | Required accessible layout |
|---|---|
| Search table | A 32-bit entry at offset 0; if that entry is zero, a 32-bit entry at offset 4 from the separately evaluated second base. |
| Object returned by F-cd96e43d3119 | A dispatch-table reference in its first 32-bit word. |
| Dispatch table for that returned object | A callable reference at offset 52. |
| Record selected by the search | An object-reference field at offset 16, referenced separately for callable lookup and receiver evaluation. |
| Object used for final callable lookup | A dispatch-table reference in its first 32-bit word. |
| Dispatch table for final callable lookup | A callable reference at offset 20. |

A native layout with wider pointers is not interchangeable with this representation merely because the conceptual objects are similar.

## Behavior

### Initial write and unconditional call

The function first writes one byte of zero at destination offset 0.

It then calls F-13ef3c7b2c3d with the value supplied by G-1 and integer zero. This call occurs for every selector value, and its result does not control subsequent execution.

The destination write precedes evaluation of G-1 for this call. If destination offset 0 aliases storage contributing to G-1, the write can affect the argument supplied. There is no established non-aliasing guarantee.

Nothing establishes that F-13ef3c7b2c3d preserves destination bytes, G-2, table entries, objects, or other global state. Subsequent actions use the state available after the call.

### Optional operation for a zero selector

When the selector is zero, the function calls F-cd96e43d3119 with no explicit arguments. Its result is retained as the object address for this optional operation.

Without checking that result for validity, the function follows the object’s first 32-bit word to a dispatch table and selects the callable at byte offset 52. This unresolved operation is designated F-unknown-1.

F-unknown-1 receives:

- The object returned by F-cd96e43d3119 as receiver.
- Integer zero.

After F-unknown-1 returns, the function writes one byte of value one at destination offset 3.

Consequently:

- Both calls precede the offset-3 write.
- If either call does not return normally, the later write is not reached.
- The offset-3 write does not restore offset 0 or any other destination byte that a dependency changed.

For every nonzero selector, the function skips F-cd96e43d3119, F-unknown-1, and the offset-3 write. Positive and negative nonzero selectors have the same branch behavior.

### Bounded search and table-base evaluation

The search begins only after the unconditional call and any zero-selector work have completed.

For the first entry, the function evaluates G-2 as a table base and reads the 32-bit entry at offset 0. If that entry is nonzero, it is selected and no second entry is examined.

If the first entry is zero, the function separately evaluates G-2 again and reads the 32-bit entry at offset 4 from that second base. The search then ends regardless of the second entry’s value.

| First entry value | Second entry value | Result |
|---|---|---|
| Nonzero | Not evaluated | Select the first value. |
| Zero | Nonzero | Select the second value. |
| Zero | Zero | Select nothing. |

The two entry addresses therefore need not be relative to the same base if G-2 differs between its evaluations. In particular, the second examination uses offset 4 from the later base; it does not restart at offset 0 of that base.

These are separate **source-level evaluations** of G-2. They do not establish guaranteed physical load counts, volatile semantics, synchronization, or a coherent snapshot. The evidence does not justify replacing them with a single captured base in circumstances where that changes observable behavior, nor does it specify what concurrent changes are guaranteed to be observed.

There is no third entry examination. Entry displacement is four bytes, and the only entry indices used for addressing are zero and one. The bounded index arithmetic does not approach 32-bit integer overflow.

A selected entry value is retained for the final traversal. The function does not reread the selected table slot to confirm it.

### Final dispatch through a selected record

When both examined entries are zero, the function returns without a final indirect call.

Otherwise, the retained nonzero entry value is treated as a record address. The record’s offset-16 object-reference field participates in two separate source-level evaluations:

- For callable lookup, that field supplies an object address. The object’s first word supplies a dispatch-table address, and the callable is selected at dispatch-table offset 20.
- For the receiver argument, the same record field is evaluated separately.

The unresolved callable role is designated F-unknown-2. Its arguments are the separately evaluated receiver and integer one.

The supplied call expression does **not** establish an ordering between callable lookup and receiver-argument evaluation. This contract therefore requires neither “lookup first” nor “receiver first.”

If the offset-16 field remains unchanged across both evaluations, the receiver is the same object used for callable lookup. If the evaluations yield different values, that identity is not established. The source does not contain a comparison or retry that would reconcile them.

As with G-2, repeated source-level field references do not prove separate physical memory loads or establish concurrent-observation guarantees.

There is no explicit validation of the selected record, either evaluation of its object field, the dispatch-table reference, or the callable reference. A nonzero entry initiates traversal; it does not certify validity. An invalid first nonzero entry does not cause fallback to the second entry.

After the final indirect call returns, the function performs no further direct writes and returns normally.

## Side effects

### Direct writes

The function directly writes only:

1. Zero to destination byte offset 0, before any dependency call.
2. One to destination byte offset 3, only on the zero-selector path and after F-unknown-1 returns.

Neither write clears a complete four-byte integer or initializes the intervening bytes. Replacing either with a wider write would add behavior not established here.

The values need not remain present at return. Later dependency calls may modify destination storage through aliasing or other access paths.

### Ordering and state exposure

The established call sequence is:

1. F-13ef3c7b2c3d.
2. For a zero selector, F-cd96e43d3119 and then F-unknown-1.
3. After the search, F-unknown-2 if a nonzero entry was selected.

The optional offset-3 write occurs between F-unknown-1 and the search. Earlier calls can affect state consulted later; no dependency is assumed to preserve globals or referenced objects.

Within the final call expression, the relative evaluation order of callable lookup and receiver construction remains unspecified by the supplied evidence.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership transfer is visible in this function.

Destination storage remains caller-supplied. The function uses externally obtained references without visible lifetime protection. The surrounding system must supply the validity and lifetime guarantees needed for whichever accesses occur.

These observations do not constrain dependency ownership behavior. A dependency may allocate, retain, release, or otherwise affect referenced storage; its contract is unknown here.

### Errors and arithmetic boundaries

There is no explicit error result, validation failure branch, rollback, or alternate traversal after an invalid reference.

The address calculations use the supplied 32-bit representation and fixed byte offsets. No overflow, wraparound, or range checks are visible. Successful access requires those calculations to designate suitable storage in the execution environment; behavior for invalid resulting addresses is documented_only.

The evidence does not specify the fault, exception, termination, or nonlocal-exit mechanism for failed accesses or calls.

## Dependencies

| Opaque function ID | Established call-site contract |
|---|---|
| F-13ef3c7b2c3d | Called once after the initial byte write, with G-1’s supplied value and integer zero. Result ignored. |
| F-cd96e43d3119 | Called only for a zero selector, with no explicit arguments. Result used as an object address. |
| F-unknown-1 | Unresolved callable selected at dispatch-table offset 52 through the returned object. Receives that retained object and integer zero. Result ignored. |
| F-unknown-2 | Unresolved callable selected at dispatch-table offset 20 through one evaluation of the selected record’s offset-16 field. Receives a separate evaluation of that field and integer one. Result ignored. |

F-unknown-1 and F-unknown-2 identify unresolved call roles, not verified concrete functions. Their targets may vary with object state, and they are not established to be distinct.

No dependency is assumed to preserve globals, preserve destination contents, or own or release memory.

## Acceptance examples

These are conditional expectations, not executed tests. Except where varied explicitly, references are valid, calls return normally, and relevant storage remains unchanged during its use.

### Nonzero selector and first-entry selection

Given selector 7 and a nonzero first entry:

- Destination offset 0 receives zero before F-13ef3c7b2c3d.
- The unconditional call receives G-1’s supplied value and zero.
- The optional object operation and offset-3 write are skipped.
- G-2 is evaluated for the first entry; there is no second-entry base evaluation.
- The first entry’s retained value identifies the selected record.
- With a stable offset-16 field, F-unknown-2 is selected through, and receives, that field’s object.
- Integer one is the final call’s other argument.

Selector −1 has the same branch behavior.

### Zero selector and second-entry selection

Given selector zero, a zero first entry, and a nonzero second entry:

- The initial write and unconditional call occur first.
- F-cd96e43d3119 supplies the object for F-unknown-1.
- F-unknown-1 receives that object and zero.
- Destination offset 3 then receives one.
- G-2 is evaluated for the offset-0 entry and separately for the offset-4 entry.
- The second entry’s value supplies the record for final dispatch.

### Different table bases between examinations

Suppose the first G-2 evaluation supplies base A and the entry at A plus 0 reads as zero. If the second evaluation supplies base B, the next entry examination uses B plus 4.

Under conditions where these evaluations observe different bases, using A plus 4 instead would not satisfy the described addressing behavior. The example does not assert that unsynchronized changes must be observable or that two physical base loads must occur.

### No selected entry

If both examined values are zero, F-unknown-2 is not called and no third entry is examined. Any zero-selector work has already occurred independently of this outcome.

### Different object-field evaluation results

Suppose the final callable-lookup evaluation of the selected record’s offset-16 field yields object A, while its receiver evaluation yields object B, and both evaluations and the resulting call are valid.

The callable is selected through A, while B is supplied as receiver. This describes the consequence of those evaluation results, not a guaranteed observation of concurrent mutation or an ordering between the evaluations. Assuming a single retained object for both roles would omit this case.

### Earlier state changes and aliasing

If an earlier dependency changes G-2 or a table entry before the search, the search evaluates the resulting state rather than a snapshot taken before the call sequence.

Likewise, if destination offset 0 aliases table storage, the initial byte write precedes table examination and may affect the entry subsequently obtained.

### Failure before the conditional write

If F-unknown-1 does not return normally, destination offset 3 has not yet received this function’s write of one. The initial write and completed dependency effects have no explicit rollback.

### Invalid selected record

If the first entry is nonzero but invalid as a record address, traversal is attempted without validation. The second entry is not used as a fallback. The resulting failure mechanism remains documented_only.

## Uncertainties

The following are documented_only:

- G-1’s declared type, complete storage width, and application meaning.
- The lifecycle, ownership, and application meaning of G-2, its entries, and referenced records.
- Volatility, synchronization, concurrent modification, and physical load behavior.
- Whether changing external state can produce differing table bases or differing final object-field values in a particular execution environment.
- The relative evaluation order of final callable lookup and receiver construction.
- The meanings of destination bytes 0 and 3.
- Concrete indirect targets and full dependency contracts.
- Dependency effects on globals, destination storage, lifetime, ownership, and reentrancy.
- Calling-convention details beyond the represented receiver and integer arguments.
- Address validity rules, alignment requirements for word accesses, and consequences of invalid address arithmetic.
- Behavior of regions absent from the supplied representation.

Initialization, activation, registration, and notification remain inferred intent, not acceptance requirements.

## Evidence

This contract is based on the supplied static function representation and the mapping of the two directly identified dependencies to opaque function IDs.

Established static evidence includes the byte-write widths and offsets, zero-selector condition, call sequence, two-entry search bound, four-byte entry spacing, separate table-base expressions for successive examinations, record offset 16, dispatch offsets 52 and 20, and integer arguments zero and one.

The final call contains separate references to the record’s offset-16 field for callable lookup and receiver supply. It does not establish their relative evaluation order or require their results to match when the field changes.

The 32-bit layout follows the supplied word accesses and reference representation. Repeated expressions are described at source level, without asserting physical load counts or synchronization.

Notices concerning omitted unreachable regions establish no additional behavior. No runtime tests have been performed.
