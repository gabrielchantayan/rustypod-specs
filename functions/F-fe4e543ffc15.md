<!-- Trusted provenance: F-fe4e543ffc15; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe4e543ffc15

## Purpose

F-fe4e543ffc15 scans fixed-size records associated with a resolved descriptor and a context. Two converted inputs determine an unsigned selection interval. A shared cursor determines which record is inspected next.

For each eligible record, the function resolves a resource value, writes it into a record, invokes object-specific operations, captures descriptor metadata, and performs a post-operation check. A successful check permits a request to transfer eight metadata bytes into a record.

The established 32-bit return values are:

- **1:** Descriptor resolution succeeded, scanning ended at the captured count or the interval stopping condition, and the completion dependency returned normally.
- **3:** The initial association was zero, descriptor resolution returned zero, or a selected record’s post-operation check returned zero.

These are control-flow meanings, not externally established application status definitions. Failure is not transactional: record writes and dependency effects may precede a return of 3, with no visible rollback.

“Resource,” “position,” and “metadata” describe how values are used here. Their application-level meanings remain unknown.

## Interface

### Arguments and result

The function accepts three 32-bit arguments:

| Argument | Interpretation |
|---|---|
| Owner address | Address used to read the association and an object pointer |
| First conversion input | Opaque value passed to F-870c4fcf7295 with the constant 10 |
| Second conversion input | Opaque value passed separately to F-870c4fcf7295 with the constant 10 |

The conversion inputs are not directly dereferenced by this function. Their valid representations depend on the conversion dependency.

The result is a 32-bit value, explicitly either 1 or 3 on the visible returning paths.

### Owner and context layout

| Structure | Byte offset | Width | Use |
|---|---:|---:|---|
| Owner | 0x08 | 32 bits | Object address |
| Owner | 0x1B0 | 32 bits | Association value |
| Context | 0x1A | 16 bits, signed | Selector for descriptor resolution |

The context selector is sign-extended to 32 bits. Its representable range is −32,768 through 32,767.

The object begins with a 32-bit indirect-call-table address. Table entries at byte offsets 0x10 and 0x14 provide operation targets.

### Descriptor and record layout

A successfully resolved descriptor supplies:

| Byte offset | Width | Use |
|---|---:|---|
| 0x10 | 8 bits | Special-path selector; compared with 3 |
| 0x68 | 32 bits | Metadata source |
| 0x6C | 32 bits | Metadata source |
| 0x70 | 32 bits | Metadata source |
| 0x74 | 32 bits | Metadata source |
| 0x7C | 32 bits | Address of a secondary structure |

A signed 32-bit record count is read at byte offset 0x04 of the secondary structure.

The record array has a stride of **0x50 bytes**:

| Record offset | Width | Use |
|---|---:|---|
| 0x00 | 32 bits | Resource-resolution input |
| 0x04 | 32 bits | Cached resource value |
| 0x08 | 32 bits | Argument to an indirect object operation |
| 0x14 | 32 bits, unsigned | Selection position |
| 0x21 | 8 bytes | Destination supplied to the transfer dependency |

The cursor dependency supplies the address of a mutable signed 32-bit cursor.

### Preconditions

Every reached memory access requires valid storage of the indicated width. Reached indirect calls require valid object, table, and target addresses. The function does not establish these conditions through bounds, capacity, alignment, or general null-address checks.

The explicit zero checks cover only the initial association, descriptor-resolution result, selected record’s cache, and post-operation result. Other zero values are not generally rejected.

Negative cursors are not rejected. Their resulting addresses require a separate validity guarantee; the function does not establish that negative indexing is supported.

## Behavior

### Resolution and scan setup

The owner’s association is read first. If zero, the function returns 3 without acquiring a context, converting either input, or notifying completion.

For a nonzero initial association:

1. F-8b2d02fa1959 is called to obtain the context.
2. After that call, the owner’s association is read again and the context’s signed 16-bit selector is read. These supply the arguments to F-a2a686313150.
3. If descriptor resolution returns zero, the function returns 3.

**The relative evaluation order of the association reread and selector read is not established.** They occur in separate arguments of the same call. Neither a selector-first nor an association-first ordering is part of this contract.

The association used for resolution need not equal the initial association: context acquisition may have changed owner state. The initial nonzero check is not repeated after acquisition.

After successful descriptor resolution, setup proceeds in this established order:

1. Convert the first conversion input through F-870c4fcf7295, with argument 10.
2. Convert the second input through the same dependency, also with argument 10.
3. Read the descriptor’s secondary-structure address and capture its signed 32-bit count.
4. Obtain the record-array base through F-02d6ae513643, passing the context.
5. Obtain the cursor address through F-28320d810e05, passing the context.

The context address, descriptor address, conversion results, count, record base, and cursor address are retained. Later changes to descriptor count storage do not refresh the captured scan limit.

### Interval arithmetic and signedness

The first conversion result is interpreted as an unsigned 32-bit lower bound. The second result contributes its 32-bit representation to the upper-bound calculation.

The upper bound is the unsigned 32-bit sum of those results, modulo \(2^{32}\). No overflow rejection or saturation is visible.

Eligibility requires both:

- Position greater than or equal to the lower bound.
- Position strictly less than the upper bound.

For a nonwrapping interval, this is the half-open interval from lower bound through, but excluding, upper bound.

A wrapped sum does not create a split interval. When the resulting upper bound is less than or equal to the lower bound, no position satisfies both eligibility comparisons. Records below the upper bound can still be skipped and advance the cursor.

Cursor-versus-count comparison is **signed 32-bit** comparison. Position comparisons are **unsigned 32-bit** comparisons. These distinctions must be preserved.

### Inspection and stopping

Each iteration first reads the shared cursor. If it is not less than the captured signed count, scanning ends.

Otherwise, that cursor selects a record at the retained base with stride 0x50. The record’s unsigned position at offset 0x14 is read.

If the position is greater than or equal to the unsigned upper bound:

- Scanning stops immediately.
- The cache is not read.
- No ordinary iteration increment occurs for that record.
- Later records are not inspected.
- Normal completion notification follows.

For a position below the upper bound:

- A position below the lower bound is skipped without reading the cache.
- A position at or above the lower bound permits processing only if its cache at offset 0x04 is zero.
- A nonzero cache skips processing.

Skipped records reach the ordinary cursor increment.

The interval stopping rule does not inspect or validate ordering. If a record at or above the upper bound precedes a later in-range record, that later record is not reached.

### Eligible-record processing

The resource-resolution input is read from offset 0x00 of the record selected by the iteration’s cursor. F-e2b9ef74c779 receives the retained context and that input.

After the dependency returns:

- Its 32-bit result is retained.
- The shared cursor is read again.
- The result is written at offset 0x04 of the record selected by this new cursor read.

A zero resource result is not rejected. It is written and used in subsequent operations.

The record supplying the resource input is therefore not necessarily the record receiving the cache write. No dependency-preservation guarantee permits treating the cursor as unchanged.

### Object operations

After the cache write, the function obtains the indirect target from slot 0x14 of the owner’s current object table. It invokes that target with these ordered argument positions:

1. The owner’s object address, read for the call.
2. The selected target value itself.
3. The field at offset 0x08 of the record selected by a fresh shared-cursor read.
4. Zero.
5. Zero.

The target is obtained before the call’s argument evaluation. The relative evaluation order of the argument expressions is not established. The target-selection object read and the object read supplied as an argument are distinct accesses.

After this call returns, the owner’s object address is read again and retained for the next operation. The descriptor’s mode byte at offset 0x10 then selects the path:

- **Mode 3:** Call slot 0x10 with the retained object address and a 1024-byte local scratch-area address. Afterward, call F-6c280d5c8cb4. Pass its result to F-594afbb97f20 together with the scratch address, retained resource result, retained resource input, and constant 3.
- **Any other mode:** Call slot 0x10 with the retained object address, retained resource result, retained resource input, and constant 2.

The scratch area is not explicitly initialized before use. This includes reuse across iterations. The indirect operation’s output extent and the downstream interpretation are not established.

The indirect operation results and the result of F-594afbb97f20 are not used to select success or failure here.

### Metadata capture and checking

After the selected object-operation path returns, descriptor fields are read in this order:

| Read order | Descriptor offset | Retained value |
|---:|---:|---|
| 1 | 0x74 | Low 16 bits |
| 2 | 0x70 | Low 16 bits |
| 3 | 0x68 | Low 16 bits |
| 4 | 0x6C | Low 16 bits |

These become four consecutive 16-bit elements in an eight-byte local block. No dependency call intervenes between these reads.

F-db9cc30e2b4e is then called with the retained descriptor address, a fresh shared-cursor value, and the retained record base.

If the result is zero, the function immediately returns 3. It does not:

- Invoke the metadata-transfer dependency.
- Perform the ordinary iteration increment.
- Invoke the completion dependency.
- Undo the cache write or prior dependency effects.

For a nonzero check result, the external transfer dependency receives:

- Destination: retained record base plus the **then-current cursor** times 0x50, plus 0x21.
- Source: the captured local metadata block.
- Size: 8.

The destination cursor is read after the check returns. Changes made by that check can therefore redirect the request. The source remains the earlier metadata snapshot.

An eight-byte copy is inferred intent. Actual transfer semantics and any return-value meaning are not established; the function does not inspect a transfer result.

### Advancement and completion

Every skipped or successfully processed iteration increments the shared cursor by one. The increment reads the cursor’s current stored value; it does not necessarily use the iteration-entry value.

A dependency may change the cursor before this increment, including during the transfer request.

No explicit increment-overflow guard is visible. Unlike the unsigned interval sum, signed cursor overflow and signed address-expression overflow do not have a safely established wraparound contract.

When scanning ends normally, F-ce1ad0b83e05 receives the retained context and constant 1. If it returns normally, the function returns 1 without inspecting its result.

## Side effects

Established direct writes are:

- The resource result to a cursor-selected record’s cache.
- The current shared cursor plus one on each completed or skipped iteration.

The transfer dependency additionally receives an eight-byte destination beginning at record offset 0x21. This is a supplied writable range, not independent evidence that eight bytes are actually copied.

Every dependency may have additional effects. No supplied contract guarantees preservation of owner fields, descriptor fields, records, cursor storage, object identity, or global state.

No explicit allocation or release is performed by this function. Caching a resource result does not establish ownership transfer. There is no visible release of an overwritten cache value or cleanup on failure.

The scratch area and metadata block have invocation-local lifetime. No permission to retain their addresses beyond the invocation is established.

Record-address arithmetic uses a signed cursor and 0x50-byte stride. Valid arithmetic and addressability are preconditions for portable interpretation; unchecked overflow is not a supported addressing mechanism.

## Dependencies

| Opaque ID or identifiable dependency | Established role |
|---|---|
| F-8b2d02fa1959 | Supplies the context |
| F-a2a686313150 | Resolves a descriptor from association and sign-extended selector |
| F-870c4fcf7295 | Converts each input, first input before second, with constant 10 |
| F-02d6ae513643 | Supplies the record base from the context |
| F-28320d810e05 | Supplies the shared cursor address from the context |
| F-e2b9ef74c779 | Resolves a resource value from context and record input |
| F-6c280d5c8cb4 | Supplies an argument for special-mode processing |
| F-594afbb97f20 | Receives special-mode scratch data and retained resource values |
| F-db9cc30e2b4e | Performs the post-operation check; zero causes return 3 |
| F-ce1ad0b83e05 | Receives normal-completion notification |
| Indirect dependency at object-table slot 0x14; opaque ID unavailable | Receives the object, target value, record argument, and two zeros |
| Indirect dependency at object-table slot 0x10; opaque ID unavailable | Receives mode-dependent arguments |
| External transfer dependency; opaque ID unavailable | Receives destination, metadata source address, and size 8 |

Dependency implementations, ownership rules, preservation guarantees, and exceptional behavior were not supplied. Missing opaque IDs are left unidentified rather than invented.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless a case specifies otherwise, addresses and arithmetic are valid, dependencies return normally, and dependencies do not alter relevant shared state.

1. **Initial association absent.** Association zero produces return 3 without context acquisition, conversions, or completion notification.

2. **Association changes during acquisition.** If context acquisition changes the association from 7 to 9, descriptor resolution receives 9. It also receives the sign-extended context selector. Neither relative ordering of those two argument reads is required.

3. **Signed selector.** A context selector with 16-bit representation 0xFFFF supplies −1 to descriptor resolution, not 65,535.

4. **Resolution failure.** A zero descriptor result produces return 3 before either conversion or record-base acquisition.

5. **Already exhausted.** Cursor equal to captured count causes no record inspection. Both conversions and both record-accessor calls have already occurred. Completion is notified, then the function returns 1.

6. **Signed count comparison.** Cursor 0 and count −1 end scanning without inspecting a record. Negative count is not interpreted as a large unsigned count.

7. **Lower-bound selection.** With converted values 10 and 5, position 9 skips without reading the cache. Position 10 with cache zero enters processing; position 10 with nonzero cache skips. Both skipped cases increment the current cursor.

8. **Upper-bound stop.** With the same converted values, position 15 stops scanning without reading its cache or performing an iteration increment. Completion follows even if a later record would have position 12.

9. **Zero width.** Converted values 10 and 0 yield upper bound 10. Positions below 10 skip; a position at or above 10 stops the scan. No record is eligible.

10. **Wrapped sum.** Converted values 4,294,967,294 and 4 yield upper bound 2. Position 4,294,967,294 stops scanning immediately. Position 1 instead skips; it is not part of a wrapped eligible range.

11. **Resolution mutates cursor.** If the eligible input comes from record 2 and resource resolution changes the cursor to 4, the resource result is written into record 4’s cache. The retained resource input still comes from record 2.

12. **Check failure.** A zero post-operation result returns 3 after the cache write and object operations, without metadata transfer, ordinary increment, or completion notification.

13. **Metadata truncation.** Descriptor fields at offsets 0x74, 0x70, 0x68, and 0x6C containing 0x12345678, 0xABCDEF01, 0x00010002, and 0xFFFF8000 produce successive 16-bit elements 0x5678, 0xEF01, 0x0002, and 0x8000. Byte encoding remains target-dependent.

14. **Check redirects destination.** If metadata has been captured and the successful check changes the cursor to 6, the transfer request targets record 6 at offset 0x21. It uses the previously captured metadata.

## Uncertainties

The following remain **documented_only**:

- Conversion representation, syntax, errors, overflow handling, and the meaning of constant 10.
- Application meanings of associations, selectors, resource values, descriptor modes, positions, and return codes.
- Dependency side effects, ownership, retention, release, and preservation guarantees.
- Whether a zero resource result is valid.
- Record ordering and actual allocated capacity.
- Scratch initialization obligations, output extent, and interpretation.
- Exact external transfer semantics.
- Native byte order of the metadata block.
- Behavior on invalid addresses, arithmetic overflow, or dependencies that do not return normally.
- Reentrancy, concurrency, and concurrent mutation guarantees.

The relative order of the two descriptor-resolution argument reads is specifically **not established**. Their common placement after context acquisition is established.

Dependencies may change cursor storage, but no particular mutation behavior is promised. Negative cursors can satisfy the signed loop condition and address before the nominal array base; this is an unchecked possibility, not a supported-input guarantee.

## Evidence

Established static evidence consists of the supplied function body’s access widths, offsets, explicit sequencing, comparisons, argument positions, direct writes, and returning paths. Where reads occur in separate call arguments, no unsupported relative evaluation order is imposed.

The interval description follows the unsigned comparisons and 32-bit unsigned addition. Resource terminology, metadata terminology, and the apparent copy purpose are interpretations rather than independently established application semantics.

No dependency implementations or runtime observations were supplied. **No runtime tests have been performed.**
