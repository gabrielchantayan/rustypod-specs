<!-- Trusted provenance: F-fbbf9291a1d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbbf9291a1d4

## Purpose

F-fbbf9291a1d4 publishes a six-word record and then either completes through a dependency, prepares another publication, or reaches an abnormal-termination boundary. An exceptional entry path performs memory writes before reaching a separate nonreturning boundary.

The application-level purpose is unknown. “Record,” “workspace,” “cursor,” and “carried value” identify access roles, not application types or ownership.

This contract distinguishes three evidence levels:

- **Established:** directly represented interface, access widths, ordering, dependency arguments, and state relationships.
- **Conditional:** consequences requiring stated arithmetic, address-ordering, or retained-condition semantics.
- **documented_only:** represented operations whose reachability or executable meaning is unresolved.

No runtime tests have been performed.

## Interface

### Explicit and implicit inputs

The explicit interface accepts three 32-bit arguments and has a signed 32-bit result. The first two arguments have no represented use. The third is forwarded unchanged during exceptional entry processing.

The function also depends on incoming state not described by that explicit interface:

| Input role | Interpretation |
|---|---|
| Payload | Unsigned 32-bit quantity used in records, selection, and address formation |
| Entry displacement | Signed 32-bit quantity used in an initial write and read |
| Address seed | Signed 32-bit quantity used to locate initial storage and establish routing conditions |
| Destination base | Address basis for the entry field |
| Workspace | Address basis for metadata and subsequent publications |
| Link selector | Unsigned 32-bit quantity used to derive dependency targets |
| Stack context | Address basis for exceptional preparation |

The delivery convention for these inputs is unknown. They must not be treated as additional ordinary parameters without independent interface evidence.

Several dependencies provide a 64-bit result consumed as low and high 32-bit components. Depending on the call, a component serves as a scalar, signed arithmetic operand, or address. F-000000000006 additionally supplies a separately consumed signed 32-bit output; its delivery convention is unresolved.

A normal return carries a dependency-provided signed 32-bit value, except for the documented-only finalization outcome described below. No status-code interpretation is established.

### Width and execution requirements

A word is 32 bits, a halfword is 16 bits, and a byte is 8 bits. Narrow stores retain the corresponding low-order bits. All six record fields, including address-bearing fields, occupy one word each. Addresses obtained from halfword reads are zero-extended to 32 bits.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). This does not establish equivalent behavior for signed expressions. Signed overflow, negative signed shifts, out-of-range conversions to signed values, and address arithmetic require separate execution assumptions.

Every selected memory access requires usable storage, and every selected dependency target must support the represented call. There are no represented bounds, alignment, permission, or target checks. In particular, the word accesses at 0x0000005b and 0x0000005f are numerically unaligned.

These requirements are preconditions for execution, not conditions validated by the function.

## Behavior

### Entry visibility and exceptional selection

The entry field is the word at destination base plus 0x74. The externally relevant entry ordering is:

1. The entry displacement becomes visible in that field.
2. A word is captured from address seed plus entry displacement.
3. The entry field is replaced with the unsigned payload shifted left one bit.

If the capture aliases the entry field, it observes the displacement, not the replacement. Later changes to the captured location do not change the saved value.

Initial storage is addressed by twice the signed address seed. Its contents are not read before exceptional selection.

Payload bit 27 selects exceptional processing. Bit 28 by itself does not. Without exceptional selection, the initial cursor and initial secondary read base both use the payload shifted left five bits, interpreted as a signed 32-bit quantity.

### Exceptional outcome

Exceptional processing calls F-000000000001 with stack context plus 0x2a4, the initial-storage address, the third explicit argument, and the captured entry word.

F-000000000002 supplies a signed value. Subtracting one establishes the exceptional value, subject to signed-arithmetic validity. That value controls all three subsequent stores:

| Required ordering | Effect |
|---|---|
| Before F-000000000003 | Store the exceptional value as a word at 0x0000005b |
| After F-000000000003 returns | Store its low byte at exceptional value plus 0x15 |
| After that byte store | Store the exceptional value as a word at exceptional value minus 0x7a |

F-000000000003’s result does not replace the exceptional value.

A boundary represented as nonreturning follows these effects. Under that interpretation, exceptional entry neither publishes a record nor returns normally. Earlier writes are not rolled back.

A represented continuation beyond this boundary is **documented_only**. It associates the exceptional value with stack-related state at offset 0x15c and invokes F-000000000004 through a target supplied by the boundary. Whether that association entails an observable memory store is unresolved. It does not establish a write to workspace plus 0x15c.

If this continuation is admitted, the exceptional value supplies both the initial cursor and the initial secondary read base.

### Publication state and record layout

The initial publication obtains its carried values and pair from initial storage. Reads occur at offsets 4, 8, and 0, in that order:

- Offset 4 supplies the signed first carried value.
- Offset 8 supplies the unsigned second carried value.
- Offset 0 supplies the current pair’s low component.

The address 16 bytes beyond initial storage is both the initial record address and the current pair’s high component.

Every publication produces the same 24-byte layout:

| Offset | Word content |
|---|---|
| 0x00 | Current record address |
| 0x04 | Zero-extended narrow address |
| 0x08 | First carried value |
| 0x0c | Current payload |
| 0x10 | Second carried value |
| 0x14 | Sampled value |

A publication is not atomic. Its observation boundary is defined by the following order:

- Capture a signed word from 0x085718a4.
- Write the cursor to workspace plus 0x248.
- Read the narrow-address halfword from cursor plus 0x1c.
- Read the signed sampled word from secondary read base plus current record address.
- Write all six record fields in increasing offset order.

The fixed-address capture is refreshed for each publication. The cursor metadata store may affect either following read through aliasing. Record writes cannot change values already captured for that publication, but may affect later accesses or dependencies.

### Routing contract

The first routing decision uses the zero, sign, and signed-addition-overflow conditions associated with the 32-bit sum of address seed and payload. Later publications use replacement conditions established during continuation.

The **condition-driven route** requires a nonzero result condition and equality between sign and signed-overflow conditions. Every other represented combination takes the **alternative route**.

These are retained conditions in the supplied representation. Their use does not prove physical condition-state preservation across dependency calls.

### Condition-driven outcome

This route replaces the current pair through F-000000000005, using the old pair’s low component interpreted as signed. Before F-000000000006, the replacement pair’s low byte is stored at first carried value plus decimal 10.

F-000000000006 receives the replacement low component and twice the signed high component, expressed as a signed left shift. Its separately delivered signed output establishes the cursor by subtracting 0x57 and multiplying by four.

The initial inner address is a zero-extended halfword fetched from a location combining:

- Signed first carried value minus 0x5a.
- Unsigned second carried value minus one.

Their combination uses unsigned addition. The second carried value contributes to the read location; it is not added to the fetched halfword. This read precedes a word store of first carried value minus 0x5a at first carried value minus 0x16.

The inner protocol has these observable requirements:

- F-000000000007 receives 124 plus a comparison Boolean, a word read from 0x0000005f, and the cursor.
- The Boolean reports whether address 0xffffffbf is below the current inner address.
- A byte value of 0x3e is stored at 0x00000049 before F-000000000008.
- The current inner address is stored at workspace plus 0xc8 after F-000000000008 and before F-000000000009.
- Workspace plus 0x134 is selected as the prospective next inner address before F-00000000000a supplies a signed result. Selection alone does not write there.

Under flat unsigned address ordering, the initial halfword-derived address makes the Boolean false. Regardless of address ordering, an ordinary Boolean is either zero or one.

The represented dispatch to F-00000000000b requires that Boolean to exceed unsigned 0xffffff83. Neither Boolean value qualifies. That dispatch, which would pass twice F-00000000000a’s signed result and return F-00000000000b’s result, is therefore **documented_only** rather than an established reachable return.

Otherwise, F-00000000000c is called. Both possible sums, 124 and 125, have clear signed-addition overflow. Under the literal retained conditions, this makes the next outcome invalid-instruction termination.

Represented inner repetition would use cursor 0x37 and inner address workspace plus 0x134. Its reachability remains **documented_only**.

### Alternative-route preparation and completion

The alternative route forms an adjusted value by adding 0x3c to the signed fixed-address capture and assigning the result to unsigned storage. The destination’s unsigned type does not remove possible signed overflow in the addition.

Adjusted-value bit 11 selects completion when set and continuation when clear.

Both selections store the current record address at workspace plus 0x244 and invoke F-00000000000d with:

- The sampled value.
- A halfword read from narrow address plus 0x3c.
- The narrow address.
- The adjusted value logically shifted right 12 bits.

There is no cursor contribution to that halfword address. Its argument-extension convention is unknown.

For completion, the following effects follow F-00000000000d:

- The sampled value is stored as a word at sampled value plus 0x14.
- A word read from fixed-address capture plus 0x70 supplies the linked value.
- F-00000000000e receives F-00000000000d’s result and the adjusted value.

The sample-derived store precedes the linked-value read, so aliasing can affect the linked value. After F-00000000000e returns, stores occur in this order:

| Destination | Content |
|---|---|
| Payload plus 0x44 | Word containing linked value minus 0x0c |
| Linked value plus 9 | Low byte of linked value minus 0x0c |
| Linked value plus 0x68 | Payload word |
| Workspace plus 0x368 | Payload word |

F-00000000000f then receives F-00000000000e’s result, fixed-address capture plus 0x34, and a word read from workspace plus 0x15c. Its signed result is returned.

The workspace-plus-0x15c read is late: it follows F-00000000000e and all four stores. Neither its value nor the memory containing it is presumed preserved across preceding dependencies.

### Continuation and subsequent publication

When adjusted-value bit 11 is clear, F-000000000010 supplies a pair. A later word read from 0x0857183c supplies the next secondary read base.

The signed high component of that pair provides a slice through right shift by 29 bits. With arithmetic right shift, the slice ranges from −4 through 3. Continuation establishes:

| Next-publication state | Relationship |
|---|---|
| First carried value | Previous sampled value minus 0x39 |
| Second carried value | Signed pair high component multiplied by two, then increased by four and assigned to unsigned storage |
| Payload | New second carried value logically shifted right 12 bits |
| Cursor | Slice shifted left two bits |

A halfword containing the low bits of the new first carried value is stored at twice the slice plus 0x14. This address uses twice the slice, not the cursor.

Replacement routing conditions use the new payload’s zero test, a clear sign condition, and signed-subtraction overflow for new first carried value minus 0x2c. That subtraction establishes a condition; it does not replace the carried value.

Before the next publication:

- F-000000000011 receives twice the slice and previous sampled value minus 0x43.
- F-000000000012 receives workspace plus 0x3d4.
- F-000000000013 follows without represented explicit arguments.
- The link selector becomes 0x08571715.
- F-000000000014 supplies the next current pair.
- The workspace advances by 0x134 bytes.

F-000000000014’s high component supplies the next record address. F-000000000010’s high component supplies continuation arithmetic, not that address. Accordingly, the next sampled-value address combines the secondary base captured from 0x0857183c with F-000000000014’s high component.

### Documented-only finalization

After F-000000000010 and the secondary-base read, the completion condition is tested again without a represented replacement. It was clear to enter continuation. Literal retained-condition semantics therefore exclude this finalization path.

If that path is nevertheless admitted, its represented contract is:

- F-000000000015 receives a word from 0x085718e8, F-000000000010’s high component, a word from workspace plus 0x11c, and a word from fixed-address capture plus 0x70. Relative ordering among these memory-argument reads is not established.
- F-000000000016 receives the resulting pair’s components and 0x08571a5c.
- F-000000000017 through F-00000000001a follow in numeric order.
- F-00000000001a’s result supplies both the address basis and content for a halfword store at that result plus 0x16.

The represented return is F-000000000010’s saved low component, interpreted as signed and shifted left two bits. Later dependency results do not replace this return source.

## Side effects

Observable writes include entry storage, records, workspace metadata, fixed addresses, and addresses derived from values. No disjointness among these regions is established. Aliasing must respect the ordering specified above.

Dependencies may modify memory subsequently observed by this function. A retained value means the function has already captured that value; it does not mean the original memory remains unchanged.

No allocation, release, ownership transfer, or permission to retain an address is established. Passing an address does not establish who owns or releases its storage.

There is no represented rollback, synchronization, recoverable error code, or fault-recovery mechanism. Invalid accesses, nonreturning dependencies, and abnormal instructions may leave earlier writes visible. Concurrency safety, reentrancy, termination, and repeat-call safety are unknown.

## Dependencies

Opaque IDs identify call roles, not proven distinct implementations.

| IDs | Role |
|---|---|
| F-000000000001–F-000000000004 | Exceptional preparation, value acquisition, intervening call, and unresolved post-boundary call |
| F-000000000005–F-000000000006 | Pair replacement and separately delivered cursor-producing output |
| F-000000000007–F-00000000000a | Inner protocol |
| F-00000000000b | Predicate-excluded dispatch and return source |
| F-00000000000c | Call preceding inner abnormal termination |
| F-00000000000d | Shared alternative-route preparation |
| F-00000000000e–F-00000000000f | Completion and signed return |
| F-000000000010–F-000000000014 | Continuation arithmetic and next-publication preparation |
| F-000000000015–F-00000000001a | Predicate-excluded finalization |

F-000000000001’s target clears the link selector’s low two bits and adds 0x5d4. F-000000000005’s target adds 0xf74 without that masking. Both calculations use unsigned 32-bit arithmetic.

Initial and represented repeated F-000000000007 calls derive their targets differently; implementation identity is not established. F-000000000004 and F-00000000000b have unresolved target provenance.

Complete calling conventions, hidden inputs, preservation rules, ownership, retention rights, memory effects, and termination guarantees are unavailable. No represented explicit arguments does not mean no inputs.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes usable storage and targets, returning dependencies where required, and valid relevant arithmetic.

| Condition | Required consequence |
|---|---|
| The entry capture aliases destination base plus 0x74 | It observes the entry displacement before the payload-derived replacement. |
| Payload is 0x08000000 | Exceptional entry is selected; 0x10000000 alone does not select it. |
| F-000000000002 returns 0x1235 | Word 0x1234 is stored at 0x5b before F-000000000003; afterward, byte 0x34 is stored at 0x1249, then word 0x1234 at 0x11ba. |
| A publication’s narrow-address read overlaps workspace plus 0x248 | It follows the cursor store; overlap and byte order determine the observed halfword. |
| First carried value is 0x1000 and second carried value is 3 | The inner halfword is read at 0x0fa8. Contents 0x1234 yield inner address 0x1234, not 0x1236. |
| Narrow address is 0x1234 | F-00000000000d’s halfword comes from 0x1270, without an added cursor contribution. |
| Fixed-address capture is 0x7c4 on the alternative route | Adjusted value is 0x800, selecting completion. |
| Previous sample is 0x100 and F-000000000010’s high component is 0x20000000 | Slice is 1; carried values become 0xc7 and 0x40000004; payload is 0x40000; cursor is 4; the halfword destination is 0x16. |
| Next secondary base is 0x1000 and F-000000000014’s high component is 0x2000 | The next sampled word comes from 0x3000, independently of F-000000000010’s high component. |
| Either Boolean reaches the inner decision under literal retained conditions | F-00000000000b is excluded; F-00000000000c precedes abnormal termination. |
| Documented-only finalization is admitted and the saved low component is 3 | The represented return is 12, regardless of later dependency results. |

## Uncertainties

The following remain unresolved and are **documented_only** wherever execution depends on them:

- Implicit-input, pair-result, and separate-output delivery conventions.
- Physical storage of the stack-associated exceptional assignment and its relationship to workspace storage.
- Memory mapping, byte order, alignment handling, address interpretation, and pointer ordering.
- Signed overflow, signed shifts, and out-of-range signed conversions.
- Extension of F-00000000000d’s halfword argument.
- Execution beyond the exceptional nonreturning boundary.
- Physical condition-state behavior across dependency calls.
- Reachability of inner dispatch, inner repetition, and finalization.
- Dependency identity, hidden inputs, preservation, ownership, effects, and termination.
- Fault handling, concurrency, reentrancy, and repeat-call safety.

No repaired control flow, recovery behavior, or memory-management discipline is inferred.

## Evidence

**Established static evidence:** The supplied representation supports the interface widths, entry ordering, six-word publication layout, stated memory-order constraints, dependency argument relationships, continuation-state relationships, and represented return sources. It also represents abnormal-instruction boundaries and unresolved indirect control flow.

**Conditional deductions:** Payload bit 27 selects exceptional entry. An ordinary Boolean cannot satisfy the inner dispatch threshold. Both possible inner sums have clear signed overflow, producing abnormal termination under literal retained-condition semantics. Those same semantics exclude finalization because its completion condition remains clear.

**Inferred intent:** Repeated publication of one 24-byte layout suggests record-oriented processing. No application-level meaning is established.

Dependency implementations, independent calling-convention evidence, memory-map guarantees, and execution traces were not supplied. **No runtime tests have been performed.**
