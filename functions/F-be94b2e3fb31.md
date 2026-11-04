<!-- Trusted provenance: F-be94b2e3fb31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be94b2e3fb31

## Purpose

F-be94b2e3fb31 chooses between a receiver-directed operation and a conditional update to a separate state object. The receiver-directed operation has priority. The state-update path is considered only when the receiver’s query results do not select that operation.

The Boolean result identifies a control-flow outcome, not independently verified application success:

- **True:** the receiver-directed operation was selected and returned normally, or the state-update eligibility check succeeded and all required subsequent calls returned normally.
- **False:** the receiver-directed operation was not selected and the state-update eligibility check failed.

The function does not examine an operation result to establish success. It also does not revalidate the state after the dependent operations.

“Receiver,” “position,” “bound,” and “advancement” are descriptive roles used in this contract. Their application-level meanings are not established by the available evidence.

## Interface

### Inputs and result

There are no explicit parameters. Effective inputs include dependency results, receiver dispatch tables, and mutable object fields.

The result is Boolean. Its external calling convention and physical return-value encoding are not established. There is no separate visible error result, and false is not a general failure indicator.

Ordinary completion requires every accessed object, dispatch table, and target to be usable at the time of access. These requirements are preconditions, not validations performed by the function.

### Selected receiver

F-b7af80e829ec supplies an address. When that address is non-null, the selected receiver is located 20 bytes before it. When the supplied address is null, the receiver remains null.

The selected receiver contains a dispatch-table address at byte offset zero. Three table entries are relevant:

| Table byte offset | Behavioral role |
|---|---|
| 0x1A4 | First selection query |
| 0x1A0 | Second selection query |
| 0x0FC | Receiver-directed operation |

Each target receives the selected receiver. Query results are represented as signed integers but are classified solely as zero or nonzero. The receiver-directed operation’s result, if any, is not used.

A null result from F-b7af80e829ec is not a request to use the state-update path. The function still attempts the first receiver dispatch. Safe handling of that null address is not established.

### State object

F-af667771d6ce supplies the state address when the receiver-directed operation is not selected. The function retains that address for the remainder of this path.

The relevant object layout is:

| Byte offset | Representation and use |
|---|---|
| 0x3C | Four-byte signed bound, read for eligibility |
| 0x40 | Four-byte signed position, read for eligibility and conditionally updated |
| 0x44 | Address within the object, passed to F-047d22db5c73 |
| 0x1FC | Address within the object, passed to F-7d964f24a044 |

The last two arguments are object-relative addresses. They are not pointer values loaded from fields at those offsets.

F-7d964f24a044 supplies a further receiver. Its offset-zero field identifies a dispatch table, and the entry at table byte offset 0x284 is invoked. The represented argument order for that operation is:

1. An address of invocation-local temporary storage.
2. The receiver supplied by F-7d964f24a044.
3. A freshly read four-byte value from the state’s position field.

The temporary occupies 20 bytes. Its semantic type, internal layout, and alignment requirements are not established.

### Width and validity constraints

The supplied representation models the relevant integer fields and address-bearing accesses as four bytes. Position and bound are compared as signed 32-bit values. These observations do not establish a complete external ABI, a portable object layout, or a language-level pointer model.

For ordinary completion, the receiver adjustment and state-relative addresses must identify usable storage in the executing environment. The function supplies no visible protection against invalid addresses, insufficient object extent, misalignment, expired objects, or invalid dispatch targets.

## Behavior

### Receiver selection takes priority

The first query is always attempted after obtaining and adjusting the selected receiver. The second query is conditional on a zero first result.

| First query result | Second query result | Selected outcome |
|---|---|---|
| Nonzero | Not queried | Evaluate state-update eligibility |
| Zero | Zero | Evaluate state-update eligibility |
| Zero | Nonzero | Perform receiver-directed operation |

A negative first result therefore prevents the second query. A negative second result selects the receiver-directed operation when the first result was zero.

On normal return from the selected receiver-directed operation, the function returns true. It does not itself call F-af667771d6ce or any of the state-update dependencies on that path. This restriction concerns calls made by F-be94b2e3fb31; it does not constrain calls or state access inside a dependency.

The receiver address is retained across the selection queries and operation. The dispatch table is consulted for the individual calls rather than established as an immutable snapshot. A query may therefore alter information used by a later dispatch. Such mutation is not itself prevented or interpreted by this function.

### State-update eligibility

If the receiver-directed operation is not selected, the function obtains the state address from F-af667771d6ce.

The eligibility decision uses a saved position value and the bound reduced by one. The position read precedes the bound read. The comparison is signed and strict: the saved position must be less than one below the bound.

Equality is insufficient. There is no nonnegative-value check, clamping, unsigned reinterpretation, or separate validation that the fields describe a valid application state.

A failed comparison produces false. On that outcome, the function makes no explicit position write and does not invoke F-7d964f24a044, the later receiver operation, F-047d22db5c73, F-c1466ea05103, or F-0a49ebff0483.

False does not imply absence of side effects. The selection queries and F-af667771d6ce have already executed and may have changed state.

### Accepted state update

Successful eligibility authorizes a four-byte write that increases the saved position by one. That write occurs before F-7d964f24a044 is called with the address at state offset 0x1FC.

The position supplied to the later receiver operation is not the saved eligibility operand. It is read again after F-7d964f24a044 returns. Consequently, a mutation of the position by that dependency changes the argument received by the operation at table offset 0x284.

The required normal-return ordering is:

| Earlier event | Required later event |
|---|---|
| Position update | Entry to F-7d964f24a044 |
| Return from F-7d964f24a044 | Fresh position read used by the later receiver operation |
| Completion of the later receiver operation | Entry to F-047d22db5c73 |
| Return from F-047d22db5c73 | Entry to F-c1466ea05103 |
| Return from F-c1466ea05103 | Entry to F-0a49ebff0483 |

The later receiver operation receives the temporary address described in the interface. F-047d22db5c73 then receives the state-relative address at 0x44 followed by that same temporary address. F-c1466ea05103 receives the temporary address. F-0a49ebff0483 receives the retained state base.

These relationships do not establish every ordering within preparation of an indirect call. In particular, the evidence does not fix the relative order of all dispatch-table reads and argument reads.

If the dependent calls return normally, the result remains true because eligibility succeeded. There is no second eligibility check and no final verification of position or bound. Later mutation of those fields does not, by itself, change the saved outcome.

### Arithmetic boundaries

Position and bound range from −2,147,483,648 through 2,147,483,647.

For any bound other than −2,147,483,648, subtracting one is representable. Under that condition, successful eligibility also makes the position increment representable. At the largest bound, the greatest eligible position is 2,147,483,645; its updated value is 2,147,483,646.

Negative values are processed by the same signed rule. For example, a position of −2 is eligible with a bound of zero. This acceptance does not establish that the subsequent dependencies can meaningfully process a negative position.

A bound of −2,147,483,648 makes the subtraction exceed the signed range. The supplied representation does not settle whether the executing artifact wraps, traps, or has behavior affected by source-language overflow rules. This boundary is **documented_only**: this contract assigns neither a Boolean result nor a resulting position.

## Side effects

### Direct storage changes

The only explicit nonlocal data write established in this function is the four-byte position update at state offset 0x40. It occurs only after successful eligibility.

The temporary storage has no visible initialization before its address is supplied to the later receiver operation. Its size does not establish that every byte is written, that it contains a valid object afterward, or that the first recipient avoids reading uninitialized contents.

F-047d22db5c73 and F-c1466ea05103 receive the same temporary address. The storage is local to the invocation; this function provides no lifetime extension after the invocation ends. Whether a dependency retains the address is unknown.

### Dependency effects and ownership

No dependency is assumed to preserve globals, receiver contents, dispatch tables, state fields, or object lifetimes. Retaining an address locally does not keep the referenced object alive.

The function contains no explicit allocation or release of the selected receiver, state object, or later receiver. This observation does not establish whether dependencies allocate, release, retain, or transfer ownership.

The temporary’s progression through several operations is consistent with a produced value followed by transfer and cleanup. That interpretation is **inferred intent**, not an established ownership contract. In particular:

- The later receiver operation is not established to use a particular hidden-result convention.
- F-047d22db5c73 is not established to copy rather than move, consume, mutate, or retain the temporary.
- F-c1466ea05103 is not established to destroy an object or release memory.
- F-0a49ebff0483 is not established to validate or commit the updated state.

### Errors and partial completion

The function provides no visible conversion of invalid memory access, unusable dispatch targets, or dependency failures into false.

There is no explicit rollback of the position update. A dependency may fail or exit nonlocally after the update has occurred. The visible body does not guarantee that subsequent calls, including F-c1466ea05103 and F-0a49ebff0483, execute after an earlier abnormal exit.

The established sequence describes normal returns. Exceptional cleanup, recovery, and error propagation remain **documented_only**.

## Dependencies

Named dependencies are identified only by their opaque function IDs.

| Dependency | Established interaction |
|---|---|
| F-b7af80e829ec | Supplies the address from which the selected receiver is derived |
| F-af667771d6ce | Supplies the state address after receiver selection declines the preferred operation |
| F-7d964f24a044 | Receives the address at state offset 0x1FC and supplies the later receiver |
| F-047d22db5c73 | Receives the address at state offset 0x44 and the temporary address |
| F-c1466ea05103 | Receives the temporary after F-047d22db5c73 returns |
| F-0a49ebff0483 | Receives the retained state address after F-c1466ea05103 returns |

The indirect dependencies are identified by receiver and dispatch-table slot: 0x1A4, 0x1A0, and 0x0FC on the selected receiver, and 0x284 on the later receiver. These identify interface locations, not concrete implementations; no opaque function IDs for their possible targets are supplied.

For every dependency, application semantics, ownership behavior, validity guarantees, mutations, and abnormal-exit behavior remain unresolved unless explicitly described above.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Except where an unsupported condition is the subject, they assume usable objects and dispatch targets, representable arithmetic, and normal dependency returns.

### Receiver-directed outcome

If the first query returns zero and the second returns −7, the receiver-directed operation is selected. Normal return from that operation produces true. F-af667771d6ce is not called by this function.

If the first query returns −7, the second query is omitted. The state-update path determines the result instead.

If both queries return zero, both queries occur, but the receiver-directed operation does not. State-update eligibility is evaluated.

### Eligible position

With position 2 and bound 5 when the state fields are read, eligibility succeeds. The position is written as 3 before F-7d964f24a044 is entered.

If that dependency leaves the position unchanged, the later receiver operation receives the four-byte value 3. The remaining dependent operations follow in the specified order, and normal completion returns true.

### Ineligible boundary

With position 4 and bound 5, eligibility fails because the position equals one below the bound. The result is false, with no explicit position write and no state-update follow-on operations.

This does not require the earlier queries or F-af667771d6ce to have been side-effect-free.

### Signed negative values

With position −2 and bound zero, eligibility succeeds and the position is written as −1. The state-update dependencies are then invoked. Their ability to accept the negative argument remains unresolved.

### Mutation after the update

Suppose eligibility succeeds and F-7d964f24a044 changes the position to 9 before returning a usable receiver. The later receiver operation receives 9, not necessarily the value previously written by F-be94b2e3fb31.

No renewed bound check occurs. Normal completion still returns true.

### Dispatch mutation

Suppose the first query returns zero but changes the receiver’s dispatch-table address. The second query uses the dispatch information available for that later call; it is not required to use the original table.

This example assumes the retained receiver and replacement dispatch information remain usable.

### Abnormal completion after advancement

If the later receiver operation exits nonlocally, the earlier position write has already occurred. The function supplies no visible rollback and no guarantee that the remaining temporary-related or final state calls execute.

### Unsupported conditions

A null result from F-b7af80e829ec leads to an attempted dispatch through a null receiver, not a defined false result or a safe state-update selection.

A minimum signed 32-bit bound has no assigned outcome because subtraction behavior at that boundary remains unresolved.

## Uncertainties

The following are **documented_only**:

- Application meanings of the receivers, queries, operations, position, and bound.
- The structural reason for the 20-byte receiver adjustment.
- Full object layouts, allocation origins, alignment requirements, and external ABI.
- Ownership transfer, retention, and lifetime guarantees.
- The temporary’s representation, initialization coverage, and interpretation.
- Concrete implementations behind the indirect dispatch slots.
- Dependency effects on globals, fields, dispatch tables, and object validity.
- Error signaling, exceptions, nonlocal exits, and exceptional cleanup.
- Minimum-bound arithmetic and invalid-address behavior.
- Thread safety, synchronization, and reentrancy.

These unknowns limit claims about safety and application success. They do not replace the established branch conditions and normal-return sequencing.

## Evidence

**Established static evidence:** the parameterless Boolean interface; conditional receiver adjustment; dispatch locations; zero/nonzero short-circuit selection; relevant state offsets and four-byte field accesses; signed eligibility comparison; conditional position update; fresh later position read; 20-byte local temporary; dependency argument relationships; and normal-return ordering.

**Inferred intent:** receiver-directed work, position advancement, temporary production, transfer, and cleanup. These descriptions do not supply missing dependency contracts.

**Unknown:** application semantics, dependency implementations, ownership, exceptional behavior, and execution details not settled by the supplied representation.

This contract is based on static evidence only. No runtime execution or runtime tests have been performed.
