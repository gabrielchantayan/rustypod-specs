<!-- Trusted provenance: F-80f1d6a7eae4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-80f1d6a7eae4

## Purpose

F-80f1d6a7eae4 performs an ordered sequence of indirect calls using an object address and two separately captured 32-bit payload values, then clears one byte of the object.

The payloads come from different global storage locations. Their bit patterns may be equal: the routine neither compares their contents nor requires them to differ.

A signed starting index determines whether indexed call pairs precede an unconditional final pair. The indexed calls receive identifiers derived from the index. The final calls receive identifiers read from two additional global locations.

An apparent purpose is to apply two payloads to selected identifiers and then clear an object status byte. This is **inferred intent** only. The meanings of the payloads, identifiers, and cleared byte are not established.

## Interface

### Inputs

**Object address**

The first argument identifies an object whose initial machine word supplies a dispatch-table address. Each indirect call selects the callable stored at byte offset `0x58` in the table resolved for that call.

The routine retains the original object address as the first argument to every callback and as the base address for its final byte write.

**Starting index**

The second argument is a signed 32-bit integer in the supplied representation. Its comparison against 6 is signed.

There is no visible lower-bound check. Negative starting indices enter the indexed portion rather than being rejected or clamped.

### Output

There is no explicit return value. Results from the indirect calls, if any, are ignored.

Normal completion includes the trailing byte clear. The routine exposes no separate success or error indicator.

### Data roles and layout

| Designation | Representation and use |
|---|---|
| Payload A | A 32-bit value read first from one global location; supplied to the first call of every pair |
| Payload B | A 32-bit value read second from another global location; supplied to the second call of every pair |
| Final identifier A | A global identifier value read for the first final call |
| Final identifier B | A global identifier value read for the second final call |
| Dispatch-table reference | The initial machine word at the object address |
| Dispatch entry | The callable reference at table byte offset `0x58` |
| Completion byte | The single byte at object byte offset `0x66` |

The representation is consistent with a 32-bit target: integer and payload operations are 32-bit, and the object-address conversion used for the final write is 32-bit-oriented. This does not establish a complete binary calling convention or portable layout for other targets.

The final identifiers are treated as 32-bit values in this representation. Their signedness and application-level interpretation are not established.

The offsets refer to different storage bases: `0x58` is relative to the dispatch table; `0x66` is relative to the object.

“Payload A” and “Payload B” identify source and call position, not unequal values or different resource identities. Equal payload values do not merge or suppress calls.

### Preconditions for normal completion

The following accesses must remain valid when they occur:

- Reads of the two payload globals.
- Reads of the two final-identifier globals.
- Repeated reads of the object’s dispatch-table reference.
- Repeated reads of the selected table entry.
- Invocation of each selected callable with the three supplied arguments.
- The final one-byte write through the original object address.

The object must remain usable across callbacks, including after the last callback returns. No visible validation or lifetime protection enforces these conditions.

## Behavior

### Entry-time payload capture

Before any indirect call, the routine reads Payload A and then Payload B from their separate global locations.

These two captured 32-bit values supply every subsequent call in the invocation. The routine does not explicitly reload either payload after callbacks begin.

The captures are separate reads, not an established atomic snapshot. The source does not compare the captured values, require inequality, or branch on equality.

A callback that changes either payload global therefore does not change the captured value passed by later calls in the same invocation. If a payload represents an address, this statement concerns only the captured address bits; it says nothing about changes to referenced memory.

### Indexed pairs

If the starting index is less than 6 under signed comparison, each index from that starting value through 5 produces two calls in this order:

| Call within pair | First argument | Second argument | Third argument |
|---|---|---|---|
| First | Original object address | Captured Payload A | Current index plus `0x7685` |
| Second | Original object address | Captured Payload B | Current index plus `0x768b` |

Both calls for one index precede the calls for the next index. The index advances by one after each completed pair.

Callback return values do not affect the visible loop condition or call sequence. A callback must nevertheless return normally for execution to reach the next operation.

When the starting index is 6 or greater, no indexed pair executes. The initial payload reads still occur, followed by the final pair.

### Unconditional final pair

After the indexed portion, including when it is empty, the routine makes two further calls:

1. Original object address, captured Payload A, Final identifier A.
2. Original object address, captured Payload B, Final identifier B.

Final identifier A is read for the first final call after all indexed calls have returned. Final identifier B is read for the second final call after the first final call has returned.

Unlike the payloads, the final identifiers are not captured at entry. Earlier callbacks may therefore change the global values later used as final identifiers.

The supplied source does not establish a complete ordering among independent operand reads used to prepare a single call. In particular, the ordering of a final-identifier read relative to dispatch resolution for that same call remains unspecified here. The ordering between completed calls is established.

### Dispatch resolution

Each call resolves its target separately through the original object:

- The object’s current initial machine word selects a table.
- The entry at that table’s byte offset `0x58` selects the callable.

A callback may change the object’s table reference or the table entry, causing a later call to reach another callable. The routine does not visibly cache one target for the invocation.

The original object address remains the first argument even if its contents change. The routine does not itself visibly replace the object’s table reference or dispatch entry.

### Final byte write

After the second final callback returns normally, the routine writes zero to the single byte at object offset `0x66`.

This write is not performed before the callbacks, between pairs, or as visible failure cleanup. It does not itself clear adjacent bytes or a larger field.

If callbacks change this byte, the routine’s final store still replaces its value with zero on normal completion.

### Integer boundaries and call counts

For a starting index below 6, the mathematical number of indexed pairs is:

**6 − starting index**

For a starting index of 6 or greater, the number is zero.

The total call count on normal completion is:

**2 × indexed-pair count + 2**

These are mathematical descriptions of the trace, not evidence that the routine stores a count in any particular width.

For the minimum signed 32-bit starting value, normal completion requires 2,147,483,654 indexed pairs and 4,294,967,310 total calls. A 32-bit unsigned trace counter cannot represent that total.

The visible index arithmetic does not require signed overflow:

- Every executing index is at most 5.
- Incrementing the last executing index produces 6.
- Adding either positive identifier offset to any executing signed 32-bit index remains within the signed 32-bit range.

For a starting index of 6 or greater, the index is not incremented by the skipped loop. In particular, the maximum signed 32-bit input does not require an overflowing increment.

### Failure and interrupted execution

There is no visible error return, validation branch, retry, recovery path, or cleanup handler.

Invalid memory, an invalid callable, a callback that does not return, or an abnormal transfer of control may prevent later calls and the final byte store. The exact failure mechanism is environment-dependent and remains **documented_only**.

## Side effects

The routine’s explicit direct write is the zero-valued, one-byte store at object offset `0x66`.

Its other externally visible actions are reads and indirect calls. The effects of those calls are unresolved. Nothing in the supplied body establishes that a callback:

- Preserves globals, object contents, or dispatch structures.
- Preserves memory referenced by a payload.
- Allocates, retains, releases, or consumes a resource.
- Leaves external state unchanged.
- Returns normally.

There is no visible allocation, deallocation, reference-count operation, or explicit ownership transfer performed directly by this routine. Passing a payload does not establish whether the callee borrows, retains, consumes, or ignores it.

Object lifetime matters even for the final callback: the routine writes through the original object address after that callback returns. A callback that invalidates the object can therefore make the trailing write invalid.

The body supplies no explicit synchronization or atomicity guarantee. Concurrent modification of globals, object fields, or dispatch tables is not specified beyond the single-invocation ordering described above.

## Dependencies

### F-000000000001 — unresolved indirect callable

This is a contract-local opaque label for the dependency selected through dispatch-table byte offset `0x58`. It is not an established identity of a concrete function. Different calls may select different concrete targets.

Each invocation supplies:

1. The original object address.
2. One captured 32-bit payload value.
3. One 32-bit identifier value.

Any returned result is ignored.

The dependency’s complete calling convention, accepted identifier domain, parameter semantics, ownership rules, side effects, and failure behavior are unresolved. In particular, the contract does not assume that it preserves globals or manages memory in any particular way.

The four global locations are data dependencies. Their initialization and application-level relationships are unknown. There is no visible direct call to a separately identified helper.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not execution results. **No runtime tests have been performed.** Unless an example states otherwise, memory remains valid and callbacks return normally.

### Starting index 6 or greater

For starting index 6, 7, or the maximum signed 32-bit value:

- No indexed calls occur.
- The first final call receives Payload A and Final identifier A.
- The second receives Payload B and Final identifier B.
- The completion byte is cleared after the second call returns.

Both payload captures still occur.

### Starting index 5

Exactly four calls occur:

1. Payload A with identifier `0x768a`.
2. Payload B with identifier `0x7690`.
3. Payload A with Final identifier A.
4. Payload B with Final identifier B.

The completion byte is cleared after the fourth call returns.

### Starting index 0

Six indexed pairs precede the final pair, producing fourteen calls.

The first indexed pair uses identifiers `0x7685` and `0x768b`. The last uses `0x768a` and `0x7690`. Payload A precedes Payload B within every pair.

### Starting index −1

Seven indexed pairs precede the final pair, producing sixteen calls.

The first pair uses identifiers `0x7684` and `0x768a`. Negative input is not clamped to zero.

### Equal payload values

Suppose both payload locations contain the same 32-bit pattern when read, and the starting index is 5.

Four calls still occur in the order specified above. Every call receives that same pattern as its second argument. The two indexed calls still have their separately derived identifiers; the final calls still read their respective identifier globals.

No equality-based optimization, call suppression, or special failure is supported by the source.

### Payload storage changed during a callback

Suppose the first callback replaces both payload globals with other values.

All later calls in the invocation continue to receive the initially captured payloads. The new global contents do not replace those captured values.

This criterion does not require any memory referenced by the payloads to remain unchanged.

### Final identifiers changed during callbacks

If an indexed callback changes Final identifier A’s global storage, the first final call uses the value present when that identifier is subsequently read.

If the first final callback changes Final identifier B’s storage, the second final call uses the subsequently read value. An implementation that captured both final identifiers at entry would not preserve this behavior.

### Dispatch changed during a callback

If a callback replaces the object’s table reference with another valid table, the next call resolves its target through that new table.

Likewise, changing the selected entry in a retained table can affect the next call. Caching the first selected callable for all calls would not preserve the visible behavior.

### Completion-byte observation and interruption

If the completion byte starts nonzero and no callback changes it, the last callback still observes the pre-clear value. The routine clears it only after that callback returns.

If a callback never returns, execution does not reach the trailing store. This does not establish any particular value for the byte, because callbacks may themselves have modified it.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of payloads, identifiers, and the completion byte.
- Whether the dispatch structure is a language-level method table or another mechanism.
- Whether payloads represent addresses, handles, masks, or other values.
- Whether equal payload patterns denote equal resources.
- The concrete callable identities and their behavior.
- Global initialization and ordinary mutation patterns.
- Ownership and lifetime obligations beyond validity of subsequent accesses.
- Full binary calling convention and target layout beyond the supplied 32-bit-oriented representation.
- Ordering of independent operand reads within preparation of one call.
- Behavior under concurrent mutation, asynchronous interruption, faults, or abnormal callback exits.

Separate payload locations establish separate reads, not unequal contents. The fixed difference between indexed identifier offsets does not establish a semantic relationship between their domains. The trailing zero store does not establish that the byte means “enabled,” “dirty,” “initialized,” or any other named state.

## Evidence

The established behavior comes from the supplied function body: two ordered entry-time payload reads; a signed comparison against 6; two ordered indirect calls for each executing index; two unconditional final calls; repeated dispatch selection through table offset `0x58`; and a trailing one-byte zero store at object offset `0x66`.

The body contains no payload comparison or equality-dependent branch. Consequently, distinct global locations support separate capture roles but do not support a requirement that the captured values differ.

Call counts, arithmetic bounds, and conditional callback-mutation examples are deductions from the visible control and data flow. They are not runtime observations.

The proposed application-level purpose is inferred intent only. No callee implementation, global declaration set, object-layout definition, or runtime evidence establishes the unresolved semantics.
