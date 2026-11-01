<!-- Trusted provenance: F-91afc569b6ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-91afc569b6ec

## Purpose

F-91afc569b6ec coordinates a context-dependent preparation operation, a delegated two-input operation, an optional notification, and conditional propagation of a 16-bit field.

Its return value is the 32-bit result of the delegated operation. Subsequent notification and field propagation do not explicitly replace that saved result.

The observable behavior is primarily sequencing. In particular:

- Preparation precedes the delegated operation.
- A nonzero second input causes a notification after the delegated operation.
- The source field is inspected after that notification, not before it.
- Conditional propagation obtains its destination through two separate dependency calls. The destination inspected for zero need not be the destination subsequently written.

These are established properties of the supplied static representation. Interpreting the fields as flags, status values, identifiers, or ownership markers would require additional evidence.

## Interface

### Inputs

The function accepts two inputs:

| Input | Represented width | Contract role |
|---|---:|---|
| Object input | 32-bit integer in the supplied representation | Forwarded unchanged to two dependencies; also used as an address base for a 16-bit field at byte offset `0xE6`. |
| Control input | 32-bit integer in the supplied representation | Forwarded unchanged to the delegated operation; independently tested for equality with zero. |

The object input is not checked for nullness, alignment, accessibility, object type, or lifetime.

The control input is a zero/nonzero selector for this function’s own conditional behavior. It is not normalized before being forwarded to the delegated operation. Consequently, different nonzero values follow the same local branches but may produce different delegated behavior.

The presented integer widths describe the supplied representation. The underlying address width, integer-to-address conversion rules, and address-overflow behavior are not independently established.

### Output

The function returns one 32-bit value: the result produced by the delegated two-input operation.

No return value from preparation, notification, or either later context lookup is substituted for that result. There is no locally defined success code, failure code, or error-to-result conversion.

### Referenced layouts

Only two field locations are established:

| Address base | Byte offset | Access width | Local use |
|---|---:|---:|---|
| Object input | `0xE6` | 16 bits | Read for a zero test; potentially read again as the value to propagate. |
| Context returned by a later lookup | `0x80` | 16 bits | Read for a zero test or written with the propagated value. |

The source zero test and destination zero test are represented as signed 16-bit reads. Signedness does not alter the zero/nonzero decision. Propagation transfers the 16-bit representation without an established arithmetic conversion.

There is no evidence for the overall size, packing, endianness, or semantics of either containing object. Valid execution requires the particular accesses reached on a given path to be supported by the execution environment.

## Behavior

### Initial lookup and preparation

The function first calls F-fdd8589eca2d without arguments.

Its returned 32-bit value is passed as the first argument to F-3e5212464798. The second argument is the original object input, and the third argument is zero.

This initial lookup result is not locally dereferenced. Its meaning to F-3e5212464798 is unknown. The result of preparation is not used to select a branch or determine the final return value.

Both calls occur regardless of the control input, provided execution reaches them and each returns normally.

### Delegated operation

After preparation returns, the function invokes the unmapped delegated dependency with the original object input and the original control input, in that order.

For this contract, that dependency is assigned the document-local opaque identifier **F-U01**. This identifier does not assert a published identity or establish that the dependency has been resolved.

The 32-bit result of F-U01 is retained for the eventual return.

No local branch interprets that result. In particular, a zero result does not skip notification or field processing, and a nonzero result does not independently enable them.

### Conditional notification

After F-U01 returns:

- If the control input is zero, F-5c5958cfd5fe is not called.
- If the control input is nonzero, F-5c5958cfd5fe is called without arguments.

Any result from F-5c5958cfd5fe is unused.

The notification occurs before the source-field zero test. Therefore, the source field examined afterward may reflect changes made by the delegated operation, the notification, or other intervening activity. This contract does not assume either dependency preserves the source object or any global state.

### Source-field inspection

After the optional notification returns, the function reads the 16-bit field at object-input address plus `0xE6` and tests whether it is zero.

This read occurs even when the control input is zero. The control input’s zero value prevents later propagation, but it does not bypass this source-field read.

If the source field is zero, propagation stops. No additional context lookup is performed.

If the source field is nonzero but the control input is zero, propagation also stops. No additional context lookup is performed.

Only a nonzero source-field observation together with a nonzero control input enables the destination check.

### Destination inspection

When both enabling conditions hold, the function calls F-fdd8589eca2d again.

It treats that call’s result as an address base and reads the 16-bit field at byte offset `0x80`.

If the destination field is nonzero, no propagation write occurs, and there is no further context lookup on this path.

If the destination field is zero, execution proceeds to the write stage.

The zero observation applies to this particular lookup result. It does not establish that any later lookup returns the same context or that the observed field remains zero.

### Propagation write

After observing a zero destination field, the function calls F-fdd8589eca2d a third time in total.

This final lookup supplies the destination address base for a 16-bit write at byte offset `0x80`. The value written is obtained by reading the object’s 16-bit field at byte offset `0xE6` again.

Consequences:

- The value written is not necessarily the value observed during the earlier source-field test.
- The context written is not necessarily the context whose field was tested for zero.
- The final destination field is not checked for zero before being overwritten.
- There is no local synchronization, compare-and-set operation, retry, or consistency check.

The final context lookup precedes the source reread and destination write. Any changes caused by that lookup can therefore affect the source value subsequently copied.

### Return and abnormal execution

After the applicable field processing completes, the function returns the retained result of F-U01.

This normal-return contract assumes every reached dependency returns normally and every reached memory access succeeds. The supplied representation contains no local recovery for an inaccessible field, invalid context result, dependency failure, exception, or nonreturning call.

## Side effects

### Explicit local memory effects

The function explicitly performs:

- One 16-bit source-field read on every path that reaches post-operation field processing.
- One additional 16-bit destination-field read when the source observation and control input are both nonzero.
- One additional 16-bit source-field read and one 16-bit destination-field write when the destination observation is zero.

The source field is not explicitly written by this function. The only explicit store is to offset `0x80` of the final lookup result.

These statements describe accesses in the supplied representation, not guaranteed bus transactions, atomic operations, or volatile-access semantics.

### Dependency effects

Preparation, delegation, notification, and context lookup may have additional side effects. Their implementations are unavailable. They must not be assumed to preserve globals, preserve object contents, return stable context identities, or leave memory ownership unchanged.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership transfer is visible locally.

Forwarding an input to a dependency does not establish whether that dependency borrows, retains, consumes, or releases it. Successful later field access requires suitable object lifetime and accessibility despite those unknown effects.

The context results used for field access likewise require accessible storage on the paths that dereference them. No local null check or lifetime extension is established.

## Dependencies

| Opaque function ID | Arguments supplied here | Result usage and established role |
|---|---|---|
| F-fdd8589eca2d | None | Called once unconditionally; up to two additional calls support destination inspection and writing. The first result is forwarded to preparation; later results are used as address bases. |
| F-3e5212464798 | Initial lookup result, object input, zero | Preparation before delegation. Its result is unused. |
| F-U01 | Object input, control input | Delegated operation. Its 32-bit result becomes this function’s return value. Published identity is unknown. |
| F-5c5958cfd5fe | None | Called after delegation exactly when the control input is nonzero. Its result is unused. |

Normal execution makes between one and three calls to F-fdd8589eca2d. Their results are independent observations; equivalence is not established.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Each assumes normal dependency returns and valid reached memory accesses.

### Zero control still permits a source read

Given a zero control input:

- Initial lookup, preparation, and delegation occur.
- Notification is skipped.
- The source field at `0xE6` is read.
- No destination lookup or propagation write occurs.
- The delegated result is returned.

A null or inaccessible object input is therefore not made safe merely by choosing zero control.

### Nonzero control with zero source

Given a nonzero control input and a source field observed as zero after notification:

- Notification occurs.
- No post-notification context lookup occurs.
- No explicit memory write occurs.
- The delegated result is returned.

An earlier nonzero source value is irrelevant if a dependency changes it to zero before the source test.

### Nonzero destination suppresses propagation

Given nonzero control, a nonzero source observation, and a nonzero field at `0x80` in the second lookup result:

- Exactly two context lookups occur in total.
- The destination field is inspected but not explicitly modified.
- No final lookup occurs.

### Stable context and source permit a copy

Given nonzero control, a source value of `0x1234` on both source reads, and a context whose destination field is zero and whose identity remains stable across the later lookups:

- Exactly three context lookups occur.
- The final 16-bit store writes `0x1234` at that context’s offset `0x80`.
- The delegated result remains the return value.

Context stability is an example condition, not a guaranteed invariant.

### Changed context redirects the write

Given a zero destination observation in one context, followed by a final lookup returning a different context:

- The write targets the final context.
- The previously inspected context receives no explicit write from this function.
- The final context’s prior field value is not checked.

### Changed source affects the propagated value

Given a nonzero initial source observation and a zero destination observation, suppose the final lookup changes the source field to zero.

The subsequent source reread obtains zero, and the function writes zero to the final destination. The earlier source test does not guarantee that a nonzero value is ultimately propagated.

### Nonzero controls are forwarded intact

Given control values `1` and `2`, both enable the same local notification and propagation conditions. F-U01 nevertheless receives the respective original value, so equal results or identical dependency effects are not required.

## Uncertainties

### Inferred intent

The destination-zero check suggests an attempt to populate an unset context field from an object field. This is inferred intent only. The repeated context lookup and source reread prevent a stronger guarantee that the function fills precisely the field it inspected, or copies precisely the value it tested.

### documented_only

The following remain unresolved:

- The published identity and semantics of F-U01.
- The meanings of the two fields and both inputs.
- Whether context lookups normally return the same object.
- Dependency error conventions and nonlocal control flow.
- Dependency effects on globals, object contents, and memory ownership.
- Platform address width, endianness, alignment requirements, and address-arithmetic behavior.
- Threading, reentrancy, synchronization, and access atomicity.
- Whether aliases exist between the source object and any returned context.

No acceptance claim assumes these unknowns away.

## Evidence

The supplied static representation establishes call sequencing, argument forwarding, zero/nonzero branches, 16-bit field accesses at offsets `0xE6` and `0x80`, the repeated context lookups, and preservation of the delegated 32-bit result for return.

The provided dependency mapping establishes the published opaque identities listed above except F-U01. That dependency’s document-local identifier supplies a reference without inventing a resolved identity.

No implementation access, dependency inspection, runtime execution, or runtime tests were performed. The acceptance examples are conditional consequences of the static representation, not observed outcomes.
