<!-- Trusted provenance: F-2c9ace2640db; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c9ace2640db

## Purpose

F-2c9ace2640db processes an indexed item within a mutable context. It acquires and normalizes two metric pairs, discovers item data, and selects empty, simple, or composite processing. Composite processing can recursively process component indices using the same context.

The established contract consists of explicit memory accesses, calls, branches, conversions, and returned statuses. “Item,” “metric,” “geometry,” and “component” are descriptive labels, not established application-level types.

Processing is not transactional. A failure can leave metrics, coordinates, component records, and recursive state partially updated.

## Interface

### Arguments and result

| Argument | Representation | Role |
|---|---|---|
| Context | Pointer in the observed 32-bit layout | Mutable input/output state |
| Item index | Unsigned 32-bit integer | Selects the item |
| Depth | Unsigned 32-bit integer | Checked before item-index validation |

The result is a 32-bit status:

| Status | Established condition |
|---|---|
| 0 | Successful completion |
| 0x10 | Item index is at least the owner’s unsigned item count |
| 0x14 | A nonempty item’s signed 16-bit discriminator is below −1 |
| 0x15 | Depth exceeds the unsigned 16-bit limit read at entry |
| Other nonzero values | Propagated from specified dependencies or callbacks |

Dependency-generated status meanings are **documented_only**.

### Context layout

Offsets are bytes. Fields are 32 bits unless stated otherwise.

| Offset | Observed role |
|---|---|
| 0x00 | Owner pointer |
| 0x04 | Scale-state pointer |
| 0x08 | Secondary-output pointer |
| 0x0c | Shared-state pointer |
| 0x10 | Flags |
| 0x14 | Current item index |
| 0x1c | Item-data presence field |
| 0x20 | Discriminator; only the signed low 16 bits select the representation |
| 0x24, 0x28, 0x2c, 0x30 | Geometry fields |
| 0x34, 0x38 | First origin and extent |
| 0x3c | One-time captured first extent |
| 0x40 | Initialization marker; only the byte at this address is tested and written |
| 0x44, 0x48 | Auxiliary pair A |
| 0x4c, 0x50 | Auxiliary pair B |
| 0x54 | Addend for the parsing argument |
| 0x98 | State saved during composite processing |
| 0xa0, 0xa4 | Second origin and extent |
| 0xa8, 0xac | Auxiliary pair C |
| 0xb0, 0xb4 | Auxiliary pair D |

Normalized origins are signed 16-bit values sign-extended into 32-bit fields. Normalized extents are unsigned 16-bit values zero-extended into 32-bit fields.

### Referenced state

The entry-captured owner provides:

- An unsigned 32-bit item count at 0x10.
- An unsigned 16-bit default second extent at 0x44.
- A 32-bit handle at 0x64.
- An unsigned 16-bit first-extent override at 0xe2.
- A byte selecting second-metric retrieval at 0x124.
- A 32-bit override-enable field at 0x1dc.
- Callback slots at 0x1fc, 0x200, 0x204, 0x208, and 0x20c.
- A callback-table pointer at 0x210.
- An adjustment-enable byte at 0x29c.

The depth-limit address is formed by adding the entry-captured owner address to a 32-bit value read from an unresolved global. The function reads an unsigned 16-bit limit at that resulting address. Neither the global value nor the resulting owner-relative location is established as constant.

Scale state supplies 32-bit operands at 0x30 and 0x34.

Shared state supplies signed 16-bit quantities at 0x14 and 0x16, a 32-bit component-base index at 0x2c, record pointers at 0x30 and 0x50, and a 32-bit component count at 0x4c. Records have a 0x20-byte stride, with an item index at 0, unsigned 16-bit flags at 4, and 32-bit adjustment targets at 8 and 0x0c.

### Preconditions

All accesses reached by the selected path require valid storage, callbacks, and buffer capacity. No explicit pointer validation, component bounds checks, allocation-size checks, or adjustment-buffer length checks are present.

Dependencies’ numeric requirements, mutation permissions, ownership rules, and reentrancy guarantees are **documented_only**. Their preservation of pointers, globals, flags, or counts must not be assumed.

## Behavior

### Entry validation and scale capture

The function first captures the owner and shared-state pointers from the context. It then reads the unresolved depth-offset global and the unsigned 16-bit limit at owner plus that value.

Depth strictly greater than the limit returns 0x15. Equality is accepted. Only after accepting depth does the function compare the item index against the owner’s unsigned 32-bit count. An index at least that count returns 0x10. Neither rejection performs an explicit context write.

For an accepted index, the function writes the current-index field.

If flag bit 0 is clear, it captures both scale operands from scale state. Otherwise, both captured operands are 0x00010000. Later scaling decisions reread the current flags; they do not reuse the entry decision.

### Metric acquisition

Four writable 32-bit temporary outputs start at zero. Before the first metric call, the function captures a flags value for the first-extent override decision.

It reads the table pointer from captured-owner offset 0x210, then the function pointer from table offset 0x70. The first metric call receives all five arguments, in order:

1. Captured owner.
2. Selector 0.
3. Item index.
4. Pointer to the first-origin temporary.
5. Pointer to the first-extent temporary.

Its return value does not cause an error return.

If bit 9 of the captured flags is clear, the function reads the owner’s override-enable field. When that field is nonzero, it replaces the first-extent temporary with the unsigned 16-bit value at owner offset 0xe2.

The function next reads the owner byte at 0x124:

- If zero, the second origin becomes zero and the second extent is read as unsigned 16 bits from owner offset 0x44.
- Otherwise, it reloads the table pointer from captured-owner offset 0x210 and reloads the function pointer from that table’s offset 0x70. This dispatch receives captured owner, selector 1, item index, second-origin output pointer, and second-extent output pointer. Its return is also ignored.

The selector-1 target need not be the selector-0 target. The first callback can change the table pointer or its function-pointer entry.

The function writes normalized metrics in this order: first origin, first extent, second origin, second extent. Origins retain the signed interpretation of their low 16 bits; extents retain only their low 16 bits.

If the initialization byte at context 0x40 is zero, the function first writes that byte to 1, leaving the other bytes of the containing field untouched, then writes the normalized first extent to 0x3c. A nonzero byte suppresses both writes.

### Discovery and empty items

F-64ca99439dd1 receives captured owner, item index, and a writable region beginning at context 0x1c. Its return is retained for possible parsing, not checked independently as a status.

If the presence field at 0x1c is zero, the function writes zero to geometry offsets 0x24, 0x2c, 0x28, and 0x30, in that order. It then writes A through D, first coordinate before second:

- A = (−first origin, 0).
- B = (−first origin + first extent, 0).
- C = (0, second origin).
- D = (0, second origin − second extent).

These calculations use the context’s current metric fields after discovery.

The empty path reloads the owner pointer from the context. If that owner’s adjustment-enable byte is nonzero, it captures the owner handle and calls F-bfa215d8fff9 with owner, item index, a pointer-output location, and count 4.

A nonzero result returns immediately. On success, eight consecutive 32-bit buffer entries are added to A, B, C, and D in coordinate order. F-bf7242c98d06 then receives the captured handle, not the buffer pointer.

If current flag bit 0 is clear, four F-74efb2674226 calls transform A’s first coordinate, B’s first coordinate, C’s second coordinate, and D’s second coordinate, in that order. The first two use the captured first scale; the latter two use the captured second scale. Each result is stored before the next coordinate is read for transformation.

The empty path returns zero.

### Nonempty parsing and preparation

For a nonempty item, the callback selected from captured-owner slot 0x1fc receives context, item index, and the 32-bit sum of the discovery return and current context field 0x54.

A nonzero parsing result returns directly, without the slot-0x200 callback.

On parsing success, the slot-0x204 callback receives context. If it fails, the function calls slot 0x200 with context and returns the preparation failure.

After preparation succeeds, auxiliary coordinates are written in A-through-D coordinate order:

- A = (geometry at 0x24 − first origin, 0).
- B = (geometry at 0x24 − first origin + first extent, 0).
- C = (0, second origin + geometry at 0x30).
- D = (0, second origin + geometry at 0x30 − second extent).

The signed low 16 bits of the discriminator then select the processing path.

### Simple and unsupported representations

A nonnegative discriminator selects the simple path. The slot-0x208 callback receives context.

If it fails, slot 0x200 is called and the failure is returned. If it succeeds, slot 0x200 is called before F-d692aa94b284 receives context. A nonzero result from that dependency returns immediately. Otherwise, F-3c4d89a1b06c receives the entry-captured shared-state pointer, and the function returns zero.

A discriminator below −1 produces status 0x14, calls slot 0x200, and returns that status.

Exactly −1 selects composite processing.

### Composite preparation, adjustments, and scaling

Before calling slot 0x20c with context, the function reads shared-state offset 0x16 as signed 16 bits and retains its sign-extended value as an unsigned 32-bit quantity. It also reads signed 16 bits at shared-state 0x14; that captured value has no subsequent observable use in this body.

A nonzero slot-0x20c result is followed by slot 0x200 and returned.

On success, context field 0x98 is saved, then slot 0x200 is called. Its return is ignored.

If the captured owner’s adjustment-enable byte is nonzero, the function captures its handle and calls F-bfa215d8fff9 with captured owner, item index, pointer-output location, and current component count plus four. A nonzero status returns immediately.

After success, component count, record pointer at shared-state 0x50, and base index are reread. The adjustment traversal interprets the count as signed:

- Records are visited from the base index in 0x20-byte steps.
- Each position consumes two buffer entries in the indexing scheme.
- Flag 0x0002 enables adding that position’s entries to record offsets 8 and 0x0c, in that order.
- Eight entries after the traversed component positions are added to A through D.
- F-bf7242c98d06 receives the captured handle.

For a negative signed count, no record iteration occurs and auxiliary adjustments begin at buffer entry zero. Buffer validity and the meaning of the earlier count-plus-four request remain unresolved.

Current flag bit 0 then controls the same four-coordinate scaling sequence used for empty items.

### Recursive composite expansion

If current flag 0x0400 is clear, the function captures the component count as unsigned 32 bits and captures the base index. It calls F-3c4d89a1b06c before visiting components.

The captured count and base remain the traversal bounds even if dependencies change the corresponding fields. For each position, in ascending order:

- Auxiliary coordinates are saved in the read order B first, B second, A first, A second, C first, C second, D first, D second.
- Shared-state 0x16 is captured with signed-16-bit extension.
- The component item index is read through the current pointer at shared-state 0x30.
- F-2c9ace2640db is called with the same context and depth plus one.

A recursive error returns immediately. Parent coordinates and saved field 0x98 are not restored.

After a successful child, the record pointer at 0x30 is reread and the corresponding record’s flags are read. If flag 0x0200 is clear, all eight saved coordinates are restored in A-through-D coordinate order. If set, the child’s resulting coordinates remain.

Shared-state 0x16 is then reread and sign-extended. A change from its pre-child value triggers F-59536fe4c5db with context. The captured post-child value is retained before that call; the function does not reread it afterward for the final growth decision.

After every child succeeds, saved context field 0x98 is restored. If current flag bit 1 is clear and the original unsigned-converted quantity is less than the final captured quantity, F-70768044b43b receives context and the original quantity.

With zero components, the final comparison quantity remains the original capture; no growth call occurs.

### Alternate composite output

If current flag 0x0400 is set, recursive expansion is skipped. The function calls F-3c4d89a1b06c with entry-captured shared state.

After that call returns, it reads a 32-bit value from a second unresolved global and stores it at offset 0x48 of the secondary output currently referenced by context 0x08. The global is not established as constant or preserved by F-3c4d89a1b06c. The output pointer is also obtained after that call.

This path does not restore saved context field 0x98. Successful composite processing returns zero, with no further slot-0x200 call.

## Side effects

Explicit mutations include current index, normalized metrics, the initialization byte and captured extent, geometry fields, auxiliary coordinates, selected component-record fields, and alternate-output storage.

Recursion shares the complete context. Successful children can leave non-restored fields, including current index and metrics, describing a descendant.

Callback slots are accessed through the captured owner at their respective call sites; callback identity is not guaranteed to remain stable. Dependencies can add effects between the explicit accesses described above.

No direct allocation is visible. Adjustment-buffer ownership and lifetime are **documented_only**. The handle-based F-bf7242c98d06 call is not established as buffer deallocation. An adjustment-provider failure does not trigger that call.

There is no rollback. Metric writes precede discovery and parsing failures; composite preparation precedes adjustment failures; recursive failures preserve partial child and parent state.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-64ca99439dd1 | Discovery output; return contributes to parsing input |
| F-bfa215d8fff9 | Adjustment-pointer output and checked status |
| F-bf7242c98d06 | Handle-based call after successful adjustment application |
| F-74efb2674226 | Coordinate transformation result |
| F-d692aa94b284 | Checked simple-path processing |
| F-3c4d89a1b06c | Shared-state operation at specified path boundaries |
| F-59536fe4c5db | Notification after an observed post-child quantity change |
| F-70768044b43b | Call under the final unsigned-growth condition |
| F-2c9ace2640db | Recursive processing using the shared context |

No opaque IDs are supplied for dynamically selected callbacks. Their contracts are **documented_only** by dispatch slot, arguments, ordering, and result handling. Slot-0x200 returns and the returns of F-3c4d89a1b06c, F-59536fe4c5db, F-70768044b43b, and F-bf7242c98d06 are not used as statuses here.

## Acceptance examples

These are conditional static cases, not executed tests. Except where mutation is specified, relevant callees are assumed to preserve the fields used by each example.

1. **Validation precedence.** If depth exceeds the limit read through the current depth-offset global, return 0x15 even when the index is also invalid. No current-index write occurs. Depth equal to the limit proceeds to index validation.

2. **Dynamic depth location.** If the unresolved offset global differs between invocations, each invocation reads its limit at captured owner plus the value read for that invocation; no fixed offset is required.

3. **Metric dispatch replacement.** If selector 0 replaces the owner’s table pointer, and selector 1 is enabled, selector 1 uses the replacement table’s slot 0x70. If selector 0 instead replaces that slot, selector 1 uses the replaced target.

4. **Metric narrowing.** Origin bits 0x0000ffff become −1. Extent bits 0x12345678 become 0x5678, absent an applicable override. A nonzero metric-callback return alone does not terminate processing.

5. **One-time capture.** A zero initialization byte is written to 1 before field 0x3c receives the normalized extent. A nonzero byte leaves both unchanged.

6. **Empty geometry.** Origins −3 and 5, extents 10 and 8, no adjustments, and scaling disabled produce A = (3, 0), B = (13, 0), C = (0, 5), D = (0, −3).

7. **Unsupported representation.** A nonempty discriminator whose signed low 16 bits equal −2 returns 0x14 after slot 0x200, provided earlier stages succeed.

8. **Child retention and failure.** Successful recursion restores parent coordinates when the reread record flag 0x0200 is clear and retains child coordinates when set. Recursive failure bypasses both coordinate restoration and field-0x98 restoration.

9. **Unsigned comparison.** Original signed quantity −1 converts to 0xffffffff. A final captured quantity of 1 does not satisfy unsigned growth.

10. **Post-call global output.** If F-3c4d89a1b06c changes the alternate-output global and context’s secondary-output pointer, alternate processing writes the subsequently read global value through the subsequently obtained output pointer. It performs no component recursion.

## Uncertainties

- **Inferred intent:** Metric pairs, coordinates, and scale 0x00010000 suggest fixed-point geometry. Units, rounding, saturation, and application types are unestablished.
- **documented_only:** Both global addresses and their broader mutation rules are unresolved. The observed accesses do not establish immutable depth offsets or output values.
- **documented_only:** Dependency internals, ownership, buffer capacity, global preservation, and reentrancy are unknown.
- **Arithmetic:** Depth increment is unsigned 32-bit arithmetic. Metric narrowing and signed-to-unsigned growth comparisons are explicit. Other arithmetic uses 32-bit quantities; portable signed-overflow behavior is not established.
- **Counts:** Adjustment traversal is signed; recursive traversal is unsigned. A negative count can therefore skip adjustment records yet represent a very large recursive bound. No safe malformed-input contract is established.
- **Mutation:** Entry captures and later rereads have different lifetimes. In particular, the final growth comparison uses a captured post-child value, not necessarily the shared-state value after subsequent dependency effects.

## Evidence

This contract derives solely from the supplied static body and opaque dependency mapping: explicit access widths, address formation, conversions, call arguments, dispatch reloads, branches, writes, and returns.

Application-level intent is separated from established behavior. Unresolved globals and dependency semantics remain documented unknowns.

No runtime tests, dependency inspection, or independent implementation verification have been performed.
