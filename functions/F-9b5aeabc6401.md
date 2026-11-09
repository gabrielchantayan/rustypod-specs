<!-- Trusted provenance: F-9b5aeabc6401; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9b5aeabc6401

## Purpose

F-9b5aeabc6401 obtains an outer collection, visits its entries, obtains a secondary collection for each entry, and dispatches secondary items through a shared table of handler objects.

Two routing modes are observable:

- **Direct routing:** a record field supplies a handler-table index.
- **Sequence routing:** an initial routing-field value of exactly 1 triggers a scan that resolves a candidate sequence of handler indices. The last qualifying item’s result determines whether dispatch proceeds.

These terms describe call-site roles. They do not establish application semantics, ownership, allocation, or cleanup behavior inside dependencies.

The function has no explicit return value or local error-reporting interface. A null construction result stops processing. A null final candidate after a selector scan suppresses dispatch for that record, but does not suppress its per-entry completion call.

## Interface

### Arguments and result

The supplied representation exposes four 32-bit argument positions.

| Position | Established use |
|---|---|
| First | Parent-object address. Its field at offset `0x30` is read only when the original second argument is zero. |
| Second | Construction input. A nonzero value is used directly; zero selects the parent-field fallback. |
| Third | Initial value written into temporary secondary-collection storage. Replaced before ordinary per-entry use. |
| Fourth | Initial value written into temporary outer-count storage. Replaced before traversal. |

The initial third and fourth values do not determine normal traversal or dispatch: the relevant temporary storage is overwritten before those operations.

No explicit result is returned. The representation uses 32-bit integer and pointer-like values; it does not independently establish a portable calling convention or byte order.

### Referenced layouts

Only the following layout fragments are established:

| Storage | Offset or stride | Observed content |
|---|---:|---|
| Parent object | `0x30` | 32-bit fallback construction input |
| Outer entry | `0x08` | 32-bit record pointer |
| Record | `0x60` | 32-bit routing field |
| Record | `0x68` | 32-bit secondary-acquisition input |
| Secondary item | `0x00` | 32-bit first selector-stage input |
| Secondary item | `0x04` | 32-bit final selector-stage argument |
| Secondary item | `0x08` | One-byte qualification flag |
| Handler table | Four-byte stride | Handler-object pointer |
| Handler object | `0x00` | Operation-table pointer |
| Handler operation table | `0x08` | Dispatch-operation pointer |
| Candidate sequence | Four-byte stride | 32-bit handler index; zero terminates traversal |
| Outer collection | `0x00` | Operation-table pointer |
| Outer operation table | `0x04` | Final-operation pointer |

Routing values and sequence indices are represented as signed 32-bit integers. Counts and traversal counters are unsigned 32-bit values.

### Preconditions for normal completion

Every performed memory access and indirect call requires valid storage and an appropriate target. The function does not validate:

- Parent, outer-entry, record, secondary-item, or handler pointers.
- Handler-table extents or index ranges.
- Operation-table pointers or operation entries.
- Candidate-sequence bounds, lifetime, or termination.
- Whether collection accessors support all positions implied by their reported counts.

Dependencies and indirect operations must return normally for subsequent behavior to occur. These are requirements for normal completion, not checks implemented by this function.

## Behavior

### Construction and input selection

Before construction, temporary storage receives the original third and fourth arguments.

The original second argument determines whether the parent-field fallback will be used. In either mode, the following calls occur in order:

1. F-8157a180d901 receives 36.
2. F-ea223a7ae124 is called without explicit arguments.

When the original second argument is zero, the function then reads the parent’s 32-bit field at offset `0x30`, followed by a shared default-mode selector.

When the original second argument is nonzero, the function retains that argument and reads a different shared selector. It does not read the parent field in this mode.

The parent-field and selector reads therefore occur after both initial calls. Those dependencies are not assumed to preserve the referenced storage.

F-326b1fb00786 receives five arguments, in this order:

1. The result of F-8157a180d901.
2. The result of F-ea223a7ae124.
3. The selected construction input.
4. The selected shared selector.
5. Zero.

A null result ends processing. No explicit release of either preceding result appears on that path. Interpreting 36 as an allocation size is possible, but allocation semantics are not established.

### Outer traversal and accessor conversion

For a non-null constructed collection, F-d0b90560f1fd receives that collection. Its result is stored as the unsigned 32-bit outer count.

The function next captures the shared handler-table base. This read occurs even if the outer count is zero. The base is not explicitly refreshed during traversal.

Outer positions start at zero and are visited in increasing order while below the captured count. F-0ff4e60503e7 receives the collection and a position converted first to signed 16 bits, then extended to the 32-bit argument width.

| Unsigned traversal position | Accessor argument |
|---:|---:|
| 32,767 | 32,767 |
| 32,768 | −32,768 |
| 65,535 | −1 |
| 65,536 | 0 |

The conversion does not reduce the unsigned traversal bound. The same position conversion applies to secondary-item access during both scanning and dispatch.

For each returned outer entry, the function reads the record pointer at offset `0x08`. No null check separates that read from subsequent record accesses.

### Secondary acquisition

For each record:

- Its 32-bit field at offset `0x68` is read and passed to F-73d4385c28ec.
- The result replaces the temporary secondary-collection value.
- F-d0b90560f1fd is called again, and its result becomes the unsigned 32-bit secondary count.
- The candidate-sequence pointer is initialized to null.

The secondary-count call has **no explicit argument in the supplied representation**. It is not established as an ordinary explicit-argument query on the acquired secondary collection. Any implicit argument relationship remains unresolved.

The secondary count is obtained once per outer entry. Both the optional scan and the optional dispatch pass use that retained count.

### Initial routing and selector scan

The function reads the record’s routing field at offset `0x60`.

An initial value other than 1 enters the dispatch pass without scanning. The candidate remains null.

An initial value of exactly 1 causes a scan of every secondary position. F-6e0389afef8f receives the stored secondary-collection value and the signed-16-bit-derived position.

For each returned item, the byte at offset `0x08` is read. Only exactly 1 qualifies; other byte values cause no selector-processing calls for that item.

A qualifying item causes:

1. Its 32-bit word at offset `0x00` to be read and passed to F-283eda65a14c.
2. That result and the constant 5 to be passed to F-682adfd5a561.
3. The preceding result, a shared selector-resolution value, and the item’s 32-bit word at offset `0x04` to be passed to F-5710a8628cba.
4. The returned value to replace the candidate pointer.

The shared selector-resolution value and the item’s offset-`0x04` word are inputs obtained for the final stage after the first two calls. Their relative read order is not established.

Scanning does not stop after a non-null result. Every qualifying item replaces the candidate, including when its result is null. No explicit release accompanies replacement.

After the complete scan:

- A null candidate suppresses dispatch for the record.
- A non-null candidate permits dispatch.

This decision does not reread the routing field. A routing-field change during scanning does not independently override a null-candidate skip.

### Dispatch pass

Dispatch visits every secondary position using the retained count. Each item is fetched through F-6e0389afef8f again. Scan-pass item pointers are not retained for dispatch, and equality between scan-pass and dispatch-pass results is not established.

After fetching each dispatch item, the function reads the record’s routing field to choose the branch.

#### Sequence routing

When the branch-selection read yields exactly 1, sequence traversal starts at the retained candidate pointer.

For each sequence position, the function:

- Reads the current 32-bit entry to test for zero.
- Stops traversal for this item if that read yields zero.
- Otherwise reads the current entry for handler-table indexing.
- Loads a handler pointer from the captured table base using four-byte index scaling.
- Loads the handler’s operation-table pointer and the operation at offset `0x08`.
- Invokes that operation with the handler object and current secondary item.
- Advances by four bytes.

The terminator test and index use are distinct represented reads. No explicit call intervenes between them, but storage stability is not established.

Traversal restarts at the candidate’s beginning for every dispatch item. With stable sequence contents, zero terminates traversal rather than selecting handler zero. Duplicate nonzero indices cause repeated handler calls.

The routing field is not explicitly reread between handler calls within one sequence traversal. A callback changing that field does not itself switch the current item to direct routing.

#### Direct routing

When the branch-selection read yields anything other than 1, the function reads the routing field **again** to obtain the direct handler index.

The branch-selection read and direct-index read are separate accesses. No explicit call occurs between them, but the function establishes no concurrency or storage-stability guarantee.

The selected handler’s operation at operation-table offset `0x08` is invoked once with the handler and current secondary item.

Zero can select handler-table entry zero directly, subject to storage validity. If the first read yields 4 and the second yields 1, handler-table entry 1 is invoked directly; the second value does not change the already-selected branch.

### Per-entry completion and outer final operation

After each record’s scan and optional dispatch, F-5d38327d98c6 receives the **address of the temporary secondary-collection storage**, not merely its stored value.

This call occurs even when the secondary count is zero or a null candidate suppressed dispatch. The dependency can access or modify the temporary through that address. Its ownership and release behavior are not established.

After all outer entries, the function invokes the constructed collection’s operation at operation-table offset `0x04`, passing the collection itself. This also occurs when the outer count is zero.

The function then returns without an explicit result.

## Side effects

### State capture and refreshed reads

Explicit effects include dependency calls, indirect calls, object and shared-storage reads, and temporary-storage writes. No explicit assignment to record fields, handler-table entries, or sequence elements appears.

The observable distinctions are:

- Construction inputs from shared or parent storage are read after the initial calls.
- The handler-table base is captured once after the outer-count query.
- Handler pointers and operation pointers are loaded during dispatch.
- Initial routing selection is separate from per-item dispatch selection.
- Every direct dispatch performs an additional routing-index read.
- Candidate entries are read during each traversal rather than copied in advance.
- Each qualifying scan item causes a fresh shared selector-resolution input to be obtained.

Unknown callees may mutate reachable objects or shared storage. A changed shared handler-table base is not explicitly adopted after capture; changes to entries reachable through the captured base may affect subsequent dispatch.

### Ownership and failure behavior

No ownership transfer is established for construction inputs, dependency results, collections, handlers, or candidate sequences.

The per-entry address-taking call and outer indirect operation are finalization-shaped actions only. Neither proves destruction, deallocation, or reference-count adjustment. No explicit release of candidate sequences is visible, including overwritten candidates.

**There are no explicit bounds checks or local bounds-error reporting paths.** Indexed handler accesses and sequence traversal are unchecked. This does not establish that bounds errors cannot occur.

There are also no explicit retries, exception handlers, or recoverable failure results. No cleanup guarantee is established if a dependency or indirect operation does not return normally.

A routing transition can expose the null candidate left by initial direct routing. If a later dispatch branch-selection read yields 1, the function attempts sequence access through that null candidate. No guard or recovery path handles this transition.

### Arithmetic

Handler addressing uses a represented signed 32-bit index, multiplication by four, and addition to the captured base. Sequence advancement is four bytes per entry.

No checked arithmetic or valid index range is established. Negative indices, large indices, overflowing address calculations, and missing sequence terminators have no locally defined safe handling. The representation does not justify a portable guarantee that malformed calculations either wrap or trap.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-8157a180d901 | Receives 36; result supplies the first construction argument |
| F-ea223a7ae124 | Called without explicit arguments; result supplies the second construction argument |
| F-326b1fb00786 | Receives five construction arguments; returns a nullable outer collection |
| F-d0b90560f1fd | Supplies the outer count with an explicit collection argument; later supplies the secondary count without an explicit argument shown |
| F-0ff4e60503e7 | Supplies outer entries using the collection and converted position |
| F-73d4385c28ec | Supplies the secondary-collection value from the record’s offset-`0x68` field |
| F-6e0389afef8f | Supplies secondary items during scan and dispatch |
| F-283eda65a14c | First selector-processing stage for qualifying items |
| F-682adfd5a561 | Second stage, receiving the preceding result and constant 5 |
| F-5710a8628cba | Returns the replacement candidate pointer |
| F-5d38327d98c6 | Receives the secondary temporary’s address once per processed outer entry |

The handler dispatch and outer final operations have no supplied opaque IDs. Their contracts are limited to the observed table offsets, arguments, and call order. No dependency is assumed to preserve globals or to acquire, transfer, or release ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless a change is stipulated, referenced storage is valid and stable and calls return normally.

1. **Null construction result.** The two initial calls and construction call occur. No count query, traversal, per-entry completion, or outer final operation follows.

2. **Empty outer collection.** A non-null collection reports count zero. The handler-table base is read, then the outer final operation runs. No secondary acquisition occurs.

3. **Stable direct routing.** A record retains routing value 4 and has three secondary items. Each item is fetched once for dispatch. Both routing reads for each item yield 4, and handler 4 is invoked once per item. No selector scan occurs.

4. **Different direct-branch reads.** A dispatch branch-selection read yields 4, but its subsequent direct-index read yields 1. Handler-table entry 1 is invoked directly. No sequence access follows from that second value.

5. **No qualifying items.** Initial routing is 1, but no scanned qualification byte equals 1. Every position is scanned, no selector stages run, dispatch is skipped, and per-entry completion still occurs.

6. **Last candidate is null.** Two qualifying items resolve to a non-null candidate and then null. The final null suppresses the entire dispatch pass. The earlier result is not a fallback.

7. **Stable sequence with duplicates.** The final candidate contains 3, 3, 7, then zero. While dispatch routing reads remain 1, each dispatch item invokes handlers 3, 3, and 7 in order. Handler zero is not invoked.

8. **Empty sequence.** A non-null candidate begins with zero. Dispatch still fetches each secondary item, but invokes no handler while routing remains 1.

9. **Mutation between dispatch items.** A handler for the first item changes routing from 1 to 6. If both routing reads for the next item yield 6, that next item invokes handler 6 directly.

10. **Accessor-width boundary.** Position 32,768 is supplied as −32,768; position 65,536 is supplied as zero. The unsigned count still controls traversal. No local diagnostic accompanies these conversions.

11. **Direct-to-sequence transition.** Initial routing is non-1, leaving the candidate null. If a later dispatch branch-selection read yields 1, sequence access through null is attempted. Successful recovery is not established.

12. **Out-of-range handler selection.** A direct index designates storage outside the valid handler table. The function performs no explicit bounds check or local bounds-error report before the indexed access. Whether execution faults, accesses unintended storage, or has another consequence remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention and argument state for the secondary-count call.
- Complete behavior, side effects, and abnormal exits of dependencies and indirect operations.
- Meanings and values of the construction selectors and shared selector-resolution input.
- Whether 36 denotes an allocation size.
- Ownership, aliasing, release obligations, and lifetime guarantees.
- Valid count ranges, handler-index domains, and maximum sequence lengths.
- Concurrency, reentrancy, synchronization, and storage-stability guarantees.
- Relative order of the shared-selector and item-word reads used by the final selector stage.
- Consequences of invalid pointers, invalid indices, arithmetic overflow, missing terminators, or abnormal exits.
- Higher-level meanings of routing value 1 and qualification byte 1.

These unknowns are not resolved by assuming that dependencies preserve globals, return stable objects, or release memory.

## Evidence

**Established static evidence:** the supplied representation supports the field offsets, represented widths, call ordering, unsigned traversal bounds, signed-16-bit-derived accessor arguments, candidate replacement, null-dependent dispatch entry, distinct routing reads, repeated sequence accesses, unchecked indexed addressing, and completion calls described above.

**Inferred intent:** construction, collection traversal, selector resolution, handler routing, and finalization are role interpretations suggested by usage. They are not independent guarantees about opaque dependencies.

**Runtime evidence:** none. No runtime tests or execution traces have been performed or supplied.
