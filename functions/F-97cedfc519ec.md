<!-- Trusted provenance: F-97cedfc519ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-97cedfc519ec

## Purpose

F-97cedfc519ec conditionally notifies one dependency, then dispatches an event-like record through up to three indirect handlers associated with an object.

The dispatch order is significant:

1. An optional notification for one record subtype.
2. An optional preliminary handler for another subtype.
3. Exactly one of two primary handlers, unless the preliminary handler has already returned nonzero.
4. An optional final handler, but only after the selected primary handler returns zero.

A nonzero result from an indirect handler terminates processing and is returned unchanged. The notification dependency does not directly determine the result.

“Object,” “record,” “subtype,” and “handler” describe the observed access pattern. Their application-level meanings are inferred intent, not established semantics.

## Interface

### Inputs

The function accepts two arguments:

- **Object address:** an address used to access a dispatch-table pointer and two object fields.
- **Record address:** an address used to access a discriminator and a second word whose interpretation depends on that discriminator.

Both argument values are forwarded unchanged to every indirect handler that is invoked. The notification dependency instead receives the object address and a byte-derived value.

### Width and layout assumptions

The supplied representation uses integer-sized accesses and address-bearing integer fields. The layout below describes the apparent 32-bit target model: integer words, stored addresses, and dispatch entries occupy four bytes.

This is the model supported by the supplied access expressions, not an independently verified ABI. Byte order, alignment requirements, and the signedness of plain character values are not established.

Relevant object locations:

| Byte offset | Apparent width | Observed use |
|---|---:|---|
| 0 | 32 bits | Address of the dispatch table |
| 80 (`0x50`) | 32 bits | Comparison value used by the final-handler condition |
| 260 (`0x104`) | 32 bits | Converted to a character-width argument for the notification dependency |

Relevant record locations:

| Byte offset | Apparent width | Observed use |
|---|---:|---|
| 4 | 32 bits | Discriminator, read repeatedly |
| 8 | 32 bits | Payload address when the discriminator is 22; comparison value when it is 15 |

When the discriminator is 22, the address at record offset 8 identifies a payload with a 32-bit subtype at payload offset 4.

Relevant dispatch-table locations:

| Byte offset | Role |
|---|---|
| 292 (`0x124`) | Preliminary handler |
| 268 (`0x10C`) | Primary handler selected for subtype 42 |
| 272 (`0x110`) | Primary handler selected otherwise |
| 224 (`0xE0`) | Final handler |

These are byte offsets, not entry indices.

### Output

The result is an integer:

- The exact nonzero result of the first indirect handler that returns nonzero; or
- Zero if no invoked indirect handler returns nonzero.

The function does not normalize successful handling to 1. Negative results and other nonzero values are propagated without reinterpretation.

### Preconditions

Every memory location actually accessed must be readable and valid for the applicable target access width. Every selected indirect entry must identify a callable target compatible with the observed two-argument invocation and integer result.

There are no explicit null checks, bounds checks, table-size checks, or callable-target checks.

Validity is path-dependent. For example, a payload need not be dereferenced during a phase whose discriminator read is not 22. The object comparison field at offset 80 is read only when the late discriminator read is 15.

Dependencies must leave subsequent accesses valid if they return normally and processing continues. The function itself does not enforce that condition.

## Behavior

### 1. Optional notification

The function first reads the discriminator at record offset 4.

If it is not 22, this phase performs no payload access and makes no notification call.

If it is 22, the function:

- Reads the payload address at record offset 8.
- Reads the subtype at payload offset 4.
- Subtracts `0x2500`, or 9472, from that subtype.
- Checks whether the subtraction result is 2.

When the check succeeds, it reads the object word at offset 260, converts that word to a plain character value, and calls F-4fd087f2325e with:

- The original object address.
- The character-converted value.

For ordinary defined arithmetic, subtype `0x2502`, or 9474, satisfies this check. Under 32-bit wrapping subtraction, it is also the unique 32-bit value satisfying the check.

The supplied representation expresses signed integer subtraction. Its behavior for values whose subtraction would overflow is not independently established; the contract does not assign such cases an invented error or recovery behavior.

The notification call’s result is not tested or returned. If the call returns normally, processing continues.

### 2. Optional preliminary handler

After the notification phase, the function reads the record discriminator again. It does not reuse the initial discriminator or payload subtype.

If this new discriminator is 22, it reads the payload address again and then reads the payload subtype again.

The preliminary handler is invoked only if these new reads establish both:

- Discriminator equals 22.
- Payload subtype equals zero.

For that invocation, the function reads the dispatch-table address from object offset 0, selects the entry at table offset 292, and calls the selected target with the original object and record addresses.

If the handler returns nonzero, that exact value is returned immediately. No primary or final handler is subsequently selected or invoked.

If the handler returns zero, or if its condition was false, processing continues.

This re-reading matters: F-4fd087f2325e may have changed the record, payload, object, dispatch table, or other reachable state. No preservation guarantee for that dependency is established.

### 3. Primary-handler selection and invocation

The function reads the record discriminator again.

If it is 22, it reads the current payload address from record offset 8 and then the current subtype from payload offset 4.

It selects:

- The table entry at offset 268 when the current discriminator is 22 and the current payload subtype is 42.
- The table entry at offset 272 in every other case.

A discriminator other than 22 selects the second entry without a payload dereference in this phase.

The function reads the object’s current dispatch-table address to obtain the selected entry, then invokes that target with the original object and record addresses.

If the selected handler returns nonzero, that exact value is returned immediately. The final-handler condition is not evaluated.

If it returns zero, processing continues.

Unless the preliminary handler has already returned nonzero, one primary handler is invoked. There is no direct return-zero path that bypasses both primary entries.

### 4. Optional final handler

After a zero result from the primary handler, the function reads the record discriminator again.

If it is not 15, the function returns zero without accessing the object comparison field for this phase.

If it is 15, the function reads, in this order:

1. The word at record offset 8.
2. The word at object offset 80.

It compares these two word values for equality. This is a comparison of the stored values; neither value is dereferenced by this condition.

If they differ, the function returns zero.

If they are equal, the function reads the current dispatch-table address from object offset 0, selects the entry at table offset 224, and calls that target with the original object and record addresses.

A nonzero result is returned unchanged. A zero result leads to a zero return.

The primary handler may change the record discriminator, the record comparison word, the object comparison word, or the dispatch table before this phase. The condition and selected target reflect the later reads, not a snapshot taken at entry.

## Side effects

### Direct effects

The supplied body contains no explicit stores into the object, record, payload, dispatch table, or global memory. It contains no explicit allocation or release operation.

Its direct observable interactions consist of:

- Reading the fields described above.
- Selecting indirect targets from the object’s dispatch table.
- Calling dependencies.
- Returning an integer.

The read sequence described in this contract is the logical access sequence visible in the supplied representation. It does not establish volatile-memory semantics, synchronization guarantees, or an instruction-level ordering contract for an optimizing implementation.

### Dependency-mediated effects

Invoked dependencies may have side effects that are not described by the supplied body. Their behavior must not be assumed to preserve globals, object state, record state, payload contents, or dispatch tables.

The function repeatedly reloads relevant fields after earlier calls. Consequently, dependency mutations can change later dispatch conditions and later target selection.

The function performs no explicit rollback. Effects that have occurred before a nonzero return are not explicitly undone.

### Ownership and lifetime

No ownership transfer, retain operation, release operation, or destruction operation is visible in this function.

The object and record addresses are passed to indirect dependencies, and the object address is passed to F-4fd087f2325e. Whether those dependencies retain, release, or otherwise change the lifetime of reachable memory is unknown.

If a dependency invalidates memory that this function later accesses, there is no local validation or recovery path protecting the subsequent access.

### Errors and abnormal execution

No local error classification is present. Any nonzero indirect-handler result terminates processing, regardless of whether the application interprets it as “handled,” failure, cancellation, or another outcome.

The supplied body establishes no behavior for invalid addresses, invalid indirect targets, incompatible calling conventions, dependency non-return, or exceptional control transfer.

## Dependencies

### F-4fd087f2325e

This is the only statically identified dependency in the supplied mapping.

Its invocation is conditional on the initial notification-phase subtype check. It receives the original object address and the converted value from object offset 260.

Its internal behavior, return type, ownership effects, and global-state effects are unknown. This function does not use a result from it.

### Unresolved indirect dependencies

Four dispatch-table locations supply indirect call targets:

- Offset 292: conditional preliminary dispatch.
- Offset 268: primary dispatch for subtype 42.
- Offset 272: other primary dispatch.
- Offset 224: conditional final dispatch.

No opaque function identities are supplied for those runtime targets. The offsets identify selection locations, not distinct function identities. Multiple entries may resolve to the same target.

All observed indirect calls receive the original object and record addresses and produce an integer result used for zero-versus-nonzero control flow.

Neither the semantic meaning of these targets nor any preservation or ownership guarantee can be inferred solely from their table positions.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Ordinary discriminator

Given a record whose discriminator remains neither 22 nor 15, and a valid fallback primary entry:

- F-4fd087f2325e is not called.
- The preliminary handler is not called.
- The primary entry at offset 272 is called.
- If it returns zero, the function returns zero without calling the final handler.
- If it returns a nonzero value, that value is returned unchanged.

The function does not dereference record offset 8 as a payload address in this scenario.

### Notification subtype

Given an initial discriminator of 22 and subtype 9474:

- The notification dependency is called with the original object address and the character-converted object field.
- If the notification leaves the discriminator and subtype unchanged, the preliminary handler is skipped.
- The primary entry at offset 272 is selected because 9474 is not 42.

This example assumes the dependency returns normally and leaves all later accesses valid.

### Notification changes the next phase

Given the same initial notification condition, suppose F-4fd087f2325e changes the current payload subtype to zero while leaving the discriminator 22.

The preliminary condition must use the updated subtype and invoke the entry at offset 292. If that handler returns 7, the function returns 7 without invoking a primary handler.

### Preliminary handler changes primary selection

Given discriminator 22 and subtype zero, suppose the preliminary handler returns zero and changes the payload subtype to 42.

The primary phase must re-read the subtype and select the entry at offset 268, not the fallback entry at offset 272.

### Special primary result

Given discriminator 22 and subtype 42, with no earlier terminating result:

- The primary entry at offset 268 is invoked.
- If it returns −3, the function returns −3.
- No final-phase discriminator or comparison-field read is required after that nonzero result.

### Late matching record

Suppose the primary handler returns zero and the subsequent reads observe:

- Discriminator 15.
- Equal words at record offset 8 and object offset 80.

The final entry at offset 224 is invoked. Its nonzero result is propagated; its zero result yields zero.

### Primary handler creates a late match

Suppose the record initially has another discriminator, but the primary handler changes it to 15 and establishes equality between the two comparison words before returning zero.

The final handler is eligible. Entry-state values do not override these later observations.

### Late mismatch

If the primary handler returns zero, the late discriminator is 15, and the two comparison words differ, the final handler is not called and the function returns zero.

## Uncertainties

The following are **documented_only**:

- The independently verified target ABI, pointer width, byte order, and alignment requirements.
- The signedness of plain character conversion and the value delivered for object-field values outside its representable range.
- Signed-overflow behavior for the notification subtraction.
- Application-level meanings of the discriminator values, subtype values, object fields, and nonzero results.
- Identities and contracts of runtime-selected indirect targets.
- Whether dependencies mutate globals, retain addresses, release memory, replace dispatch tables, or alter record storage.
- Thread-safety, concurrent mutation rules, volatile access requirements, and memory-order guarantees.
- Behavior after invalid memory access, invalid dispatch, or abnormal dependency termination.

The likely intent is event routing with a special notification and a later object-specific match. That interpretation is not needed for the established dispatch contract.

## Evidence

Established static evidence consists of the supplied function body and the supplied opaque dependency mapping.

The body supports the conditional field accesses, repeated reads between calls, four dispatch-table offsets, exact nonzero-result propagation, ignored notification result, and absence of explicit memory stores or memory-management operations.

The dependency mapping identifies F-4fd087f2325e. It does not identify runtime indirect targets or establish any dependency’s side effects.

No runtime tests have been performed. The acceptance examples specify conditions under which an implementation should exhibit the stated behavior; they do not claim observed execution results.
