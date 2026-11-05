<!-- Trusted provenance: F-d3ecedf30842; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3ecedf30842

## Purpose

F-d3ecedf30842 processes an event-like record for an object-like receiver. It performs limited preliminary handling for two event subtypes, selects and invokes one of two receiver-supplied handlers, and conditionally invokes a third handler when the selected handler returns zero.

The established behavior is dispatch and result propagation. Meanings such as “event,” “receiver,” “subtype,” and “handler table” are descriptive interpretations of the access pattern, not independently established application semantics.

The function does not itself validate pointers, allocate memory, release memory, or construct an error result. Its observable effects include dependency calls, indirect handler calls, and reads of potentially mutable receiver and event storage.

## Interface

### Inputs

The function accepts two arguments:

- **Receiver:** an address treated as the base of a sequence of integer-width fields. Its first field supplies the address of a table containing callable entries.
- **Event:** an address represented by an integer-width input. Storage at offsets from this address supplies a category and, depending on that category, either an address of another record or a comparison value.

The supplied representation uses 32-bit integer and address-sized accesses. The layouts below describe that represented interface. They do not establish a portable interface for systems with different integer or pointer widths.

All offsets are byte offsets.

| Receiver offset | Width | Use |
|---|---:|---|
| 0x00 | 32 bits | Address of the receiver’s handler table |
| 0x50 | 32 bits | Value compared with an event field during conditional fallback |
| 0xE8 | 32 bits | Value narrowed to a character-width argument for one preliminary dependency |

| Event offset | Width | Use |
|---|---:|---|
| 0x04 | 32 bits | Event category |
| 0x08 | 32 bits | Nested-record address for category 0x16; comparison value for category 0x0F |

For an event whose category is 0x16, the function treats the event’s field at offset 0x08 as an address and reads a 32-bit subtype at offset 0x04 within that nested record.

The handler table contains callable entries at these offsets:

| Table offset | Role | Arguments | Result |
|---|---|---|---|
| 0x10C | Primary handler for subtype 0x2A | Receiver, event | Integer |
| 0x110 | Primary handler for all other dispatch cases | Receiver, event | Integer |
| 0xE0 | Conditional fallback handler | Receiver, event | Integer |

Each invoked handler receives the original two argument values, not a copied event or a replacement receiver.

### Output

The function returns an integer:

- A nonzero result from the selected primary handler is returned unchanged.
- A nonzero result from the fallback handler is returned unchanged.
- Otherwise, the function returns zero.

Negative handler results are nonzero and therefore propagate unchanged. There is no positive-only success test and no normalization to a Boolean value.

The return values of the preliminary dependencies, if any, do not contribute directly to the returned integer.

### Preconditions

Normal completion requires every access and call actually reached to be valid:

- The event category field must be readable.
- Whenever a category read produces 0x16 and the subsequent nested access is reached, the event’s nested-record address and the nested subtype field must be valid.
- The receiver’s table address and the selected callable entry must be readable, and the entry must identify a callable target compatible with the observed invocation.
- Receiver offset 0xE8 must be readable when preliminary subtype 0x2503 is handled.
- Receiver offset 0x50 must be readable when the fallback comparison is reached.
- Dependencies and handlers must leave any storage needed by later operations usable if execution is to continue normally.

The function does not check these preconditions or provide a defined recovery path when they fail.

## Behavior

### Preliminary handling

The function first reads the event category at offset 0x04.

If the category is not 0x16, it performs no preliminary nested-record read and calls no preliminary dependency.

If the category is 0x16, it reads the address at event offset 0x08 and then reads the subtype at offset 0x04 of that nested record. That subtype controls preliminary handling:

- **Subtype 0x2502:** invoke F-d3ddc3fa52af with no explicit arguments, then invoke F-1db977fb408b with no explicit arguments. The second call follows only if the first returns normally.
- **Subtype 0x2503:** read the 32-bit receiver field at offset 0xE8, convert it to a character-width value, and invoke F-2663b0669dd7 with the receiver and that converted value.
- **Any other subtype:** make no preliminary dependency call.

These alternatives are mutually exclusive for this preliminary subtype read. A dependency changing the event from subtype 0x2502 to 0x2503 does not cause the second preliminary alternative to run during the same invocation.

The character conversion is a narrowing conversion, not a Boolean conversion. On the conventional eight-bit-character interpretation, it retains the low eight bits. Whether values with the high bit set are subsequently interpreted as negative or nonnegative is not established by the supplied representation. The contract therefore does not assign a signed numeric meaning to converted values from 0x80 through 0xFF.

### Primary-handler selection

After preliminary handling, the function reads the event category again. It does not reuse the first category or subtype observation.

If this new category value is 0x16, it again reads the nested-record address from event offset 0x08 and the subtype from nested offset 0x04.

The selected table entry is:

- Offset **0x10C** when the newly observed category is 0x16 and its newly observed subtype is 0x2A.
- Offset **0x110** in every other case.

When the new category is not 0x16, selection does not dereference the event’s offset-0x08 field as a nested-record address.

The function reads the receiver’s table address and obtains the selected callable entry after preliminary handling. Thus, changes made by preliminary dependencies can affect both event-based selection and the table from which the handler is loaded.

The selected handler is invoked once with the original receiver and event arguments.

### Primary result

If the selected handler returns any nonzero integer, the function immediately returns that integer. It performs no fallback category read, fallback comparison, or fallback-handler lookup.

If the selected handler returns zero, the function proceeds to the fallback decision.

### Fallback decision and invocation

The function reads the event category again after the primary handler has returned zero.

If this latest category is not 0x0F, the function returns zero. It does not read the receiver’s comparison field or invoke the fallback handler.

If the latest category is 0x0F:

1. The function reads the 32-bit event field at offset 0x08 as a comparison value, not as a nested-record address.
2. It reads the 32-bit receiver field at offset 0x50.
3. It compares those integer-width values for equality.

If they differ, the function returns zero.

If they match, the function reads the receiver’s current table address, obtains the callable entry at offset 0xE0, and invokes that fallback handler with the original receiver and event arguments.

A nonzero fallback result is returned unchanged. A zero fallback result produces a zero return.

### Mutation-sensitive observations

The repeated reads are behaviorally significant:

- Preliminary dependencies may change the event before primary-handler selection.
- The first preliminary dependency for subtype 0x2502 may affect state observed by the second dependency.
- The primary handler may change the event category, its comparison field, the receiver’s comparison field, or the receiver’s table before the fallback decision.
- The fallback table lookup is fresh; it is not required to use the table observed for the primary call.

No preservation of globals, receiver fields, event fields, or table contents is assumed across calls. The function also provides no snapshot, synchronization, or atomicity guarantee across these separate observations.

## Side effects

### Direct effects

The visible body performs reads and calls. It contains no explicit write to receiver storage, event storage, nested-record storage, table storage, or global storage.

It performs no explicit allocation, deallocation, reference-count operation, or ownership transfer.

### Indirect effects

The preliminary dependencies and indirect handlers may have side effects. Their effects are not bounded by this function’s visible body.

In particular, the available evidence does not establish that any callee:

- Preserves global state.
- Preserves the receiver or event.
- Leaves the handler table unchanged.
- Retains, releases, or takes ownership of an argument.
- Returns normally.

The function’s continued use of the receiver and event after certain calls creates a validity requirement for normal execution; it is not proof that those calls preserve the relevant storage.

If a reached dependency or handler does not return normally, subsequent reads and calls described here are not reached. No local cleanup or recovery mechanism is visible.

## Dependencies

### Named dependencies

| Opaque ID | Invocation condition | Explicit arguments | Established ordering |
|---|---|---|---|
| F-d3ddc3fa52af | Initial category 0x16 and initial subtype 0x2502 | None | Before F-1db977fb408b and before primary selection |
| F-1db977fb408b | Same preliminary branch, after the preceding call returns | None | After F-d3ddc3fa52af and before primary selection |
| F-2663b0669dd7 | Initial category 0x16 and initial subtype 0x2503 | Receiver; character-width conversion of receiver offset 0xE8 | Before primary selection |

Their internal behavior, return conventions, global-state effects, and ownership policies are not established here.

### Indirect dependencies

The three handler-table entries are additional dependencies whose identities are not supplied.

The primary handler is selected from offsets 0x10C and 0x110. The fallback handler is selected from offset 0xE0 only after a zero primary result and a successful category-and-value match.

The visible invocation shape for each is two arguments and an integer result. No broader behavioral contract for those targets is established.

## Acceptance examples

These examples are conditional static expectations, not reports of executed tests. Each assumes valid storage, compatible callable entries, normal returns, and no unmentioned intervening mutation.

1. **Ordinary default dispatch.**  
   The category is 0x07. No preliminary dependency runs. The primary handler comes from table offset 0x110. If it returns 19, the function returns 19 without reading the category again for fallback.

2. **Special primary selection.**  
   The category remains 0x16 and the nested subtype remains 0x2A. No preliminary dependency runs. The primary handler comes from offset 0x10C. If it returns −4, the function returns −4 unchanged.

3. **Ordered preliminary calls.**  
   The initial category is 0x16 and the initial subtype is 0x2502. F-d3ddc3fa52af runs before F-1db977fb408b. If neither changes the dispatch-relevant event fields, the primary handler comes from offset 0x110.

4. **Reclassification before primary dispatch.**  
   The initial subtype is 0x2502. During the preliminary calls, the nested subtype becomes 0x2A while the category remains 0x16. The later selection uses offset 0x10C. The initial subtype does not remain authoritative.

5. **Character-width argument.**  
   The initial category is 0x16, the subtype is 0x2503, and receiver offset 0xE8 contains 0x00000123. Under an eight-bit-character interpretation, F-2663b0669dd7 receives the receiver and character value 0x23. Its return value does not directly determine this function’s result.

6. **Nonzero primary result suppresses fallback.**  
   The primary handler returns 5. Even if the event category is now 0x0F and its comparison value matches the receiver field, the function returns 5 without invoking the fallback handler.

7. **Zero primary result without fallback category.**  
   The primary handler returns zero, and the subsequent category read yields 0x16. The function returns zero without a fallback nested-record read or fallback-handler call.

8. **Fallback mismatch.**  
   The primary handler returns zero. The subsequent category is 0x0F, the event comparison value is 42, and receiver offset 0x50 contains 43. The fallback handler is not invoked; the result is zero.

9. **Fallback match and result propagation.**  
   The primary handler returns zero. The subsequent category is 0x0F and the two comparison values match. The handler at offset 0xE0 runs. A result of −7 propagates as −7; a result of zero yields zero.

10. **Primary mutation enables fallback.**  
    The event initially has category 0x16. The primary handler changes it to 0x0F, establishes matching comparison values, and returns zero. Fallback is eligible because the decision uses post-handler reads.

11. **Replacement table affects fallback.**  
    The primary handler replaces the receiver’s table address and returns zero. If the fallback conditions then hold, the fallback target is loaded from offset 0xE0 of the replacement table.

## Uncertainties

The following are **documented_only**:

- The application-level meanings of the category and subtype constants.
- Whether the receiver’s table is formally a virtual-method table or another dispatch structure.
- The identities and complete calling contracts of indirect targets.
- Character signedness, exact character width, and the numeric interpretation of high-bit character values.
- Alignment requirements, byte order, and portability beyond the represented 32-bit layout.
- Valid object lifetimes, memory ownership, and any retention or release performed by callees.
- Global state read or modified by dependencies and handlers.
- Concurrent-access rules and synchronization requirements.
- Behavior after invalid addresses, incompatible targets, or invalidated storage are encountered.
- Whether zero and nonzero results have application meanings beyond the visible dispatch and propagation rules.

Inferred intent is limited to event processing with preliminary handling, primary dispatch, and conditional fallback. That interpretation does not establish consumption, acknowledgment, cancellation, or error semantics.

## Evidence

The supplied static function representation establishes:

- The field and table offsets described above.
- The category and subtype comparisons.
- The order of preliminary dependency calls.
- Fresh event reads before primary selection and after a zero primary result.
- Selection between two primary table entries.
- The category-and-equality gate for the third table entry.
- Exact propagation of nonzero handler results.
- The absence of explicit memory writes, allocation, release, validation, or local recovery in the visible body.

The dependency mapping establishes the opaque IDs used for the three named calls. It supplies no behavioral guarantees for those dependencies.

No runtime tests have been performed. The acceptance examples are consequences of the visible control flow under their stated conditions, not empirical observations.
