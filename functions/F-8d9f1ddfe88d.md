<!-- Trusted provenance: F-8d9f1ddfe88d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8d9f1ddfe88d

## Purpose

F-8d9f1ddfe88d conditionally copies source bytes into a destination while maintaining a 32-bit budget determined by two dependencies. Each iteration obtains a value from a cursor-processing dependency, passes that value twice to a second dependency, and interprets the second dependency’s 64-bit result as a contribution and a continuation indicator.

A source span is copied only when its contribution is **strictly less than** the remaining budget. If its contribution is greater than or equal to that budget, the function instead writes one zero byte at the current destination position and returns the accumulated contribution plus one.

**Inferred intent:** this may be a bounded copy governed by a measure other than byte count. The available evidence does not establish an encoding, character model, display-width model, or unit for that measure. The budget is not established as a destination-buffer capacity.

## Interface

### Inputs

The function accepts three arguments:

| Argument | Established interpretation |
|---|---|
| Destination | Pointer used for sequential one-byte writes. Passed by value; no final destination pointer is returned. |
| Source | Pointer used both to initialize a dependency-controlled cursor and to identify the next source byte to copy. Passed by value; no final source pointer is returned. |
| Budget | Unsigned 32-bit value used in comparisons and subtraction. Zero causes an immediate return. |

No explicit destination size, source length, error-output location, or ownership flag is supplied.

The implementation keeps two distinct source positions:

- A **copy position**, initially equal to the supplied source pointer.
- A **processing cursor**, also initially equal to the supplied source pointer, whose address is passed to F-d3922c7fe0ec.

The processing cursor identifies the endpoint of a candidate span. The copy position advances toward that endpoint only after the candidate is accepted.

### Output

The result is a signed 32-bit integer.

Its directly established meanings are:

- With an initial budget of zero: zero.
- After accepted spans followed by a zero continuation indicator: the sum of accepted contributions.
- After a candidate fails the strict budget comparison: the sum of previously accepted contributions plus one.

This result is not established as the number of source bytes consumed or destination bytes written. The rejection path adds one for its zero-byte write, but accepted spans contribute the dependency-reported measure rather than their byte lengths.

### Dependency result layouts

F-d3922c7fe0ec receives the address of the processing cursor and returns a 32-bit value. The caller does not otherwise interpret that value.

F-ff169115f82a receives two identical copies of that value and returns a 64-bit quantity, interpreted as:

| Bit range | Caller’s interpretation |
|---|---|
| 0–31 | Unsigned 32-bit contribution |
| 32–63 | Continuation indicator, tested only for zero versus nonzero |

The upper half is viewed as signed for the zero test, but its sign does not affect the branch: every nonzero upper-half pattern means continuation.

## Behavior

### Initial state and zero-budget case

The function initializes its accumulated result to zero and its processing cursor to the supplied source pointer.

Before calling either dependency or copying any bytes, it checks the remaining budget. If that budget is zero, it returns the accumulated result immediately. For the initial call, that result is zero.

This path performs no source-byte read or destination-byte write through the supplied pointers and invokes neither dependency. It does not append a zero byte.

### Candidate processing

For each iteration beginning with a nonzero remaining budget:

1. F-d3922c7fe0ec is called with the address of the processing cursor.
2. Its returned 32-bit value is supplied as both arguments to F-ff169115f82a.
3. The returned 64-bit quantity is split into the contribution and continuation indicator described above.
4. The remaining budget is compared with the contribution using an unsigned 32-bit comparison.

Both dependency calls occur before the budget decision for that candidate. Consequently, rejecting a candidate does not undo dependency side effects or cursor movement.

The caller uses the processing cursor’s value after the dependency calls when deciding where copying stops. It does not independently validate that cursor.

### Candidate rejection

If the remaining budget is less than or equal to the contribution:

- No bytes from that candidate span are copied by this function.
- One zero byte is written at the current destination position.
- The function returns the prior accumulated contribution plus one.
- The continuation indicator does not affect this outcome.

Equality is therefore a rejection condition. A candidate whose contribution exactly matches the remaining budget is not accepted.

The zero byte overwrites the byte at the next destination position. It is not written at an offset derived directly from the budget.

### Candidate acceptance

If the contribution is strictly less than the remaining budget, the function copies bytes from the copy position up to, but not including, the processing cursor.

For each byte copied, the observable order is:

- Read one byte from the current source position.
- Write that byte to the current destination position.
- Advance each position by one byte.

This is a forward byte-by-byte operation. There is no established staging buffer or whole-span snapshot.

Once the copy position equals the processing cursor:

- The contribution is added to the accumulated result.
- The contribution is subtracted from the remaining unsigned budget.
- The continuation indicator is examined.

If the indicator is zero, the function returns the updated accumulated result immediately. It does not append a separate zero byte.

If the indicator is nonzero, processing repeats using the updated positions and remaining budget.

### Consequences of the strict comparison

For every accepted candidate, subtracting the contribution leaves a strictly positive remaining budget. This subtraction cannot underflow on the accepted path.

As a result, after a positive initial budget, ordinary accepted iterations cannot make the budget become zero. Termination instead requires a rejected candidate, an accepted candidate with a zero continuation indicator, or behavior outside the normal defined execution of the visible operations or dependencies.

A contribution of zero is accepted whenever the remaining budget is positive. Such a candidate may copy an empty or nonempty span, leaves the accumulated result and budget unchanged, and either terminates or continues according to the upper half of the dependency result.

Repeated zero-contribution candidates with nonzero continuation indicators need not terminate.

### Copying and cursor edge cases

If the processing cursor equals the current copy position, an accepted candidate copies no bytes. Its contribution is still applied, and its continuation indicator is still tested.

The copy loop checks pointer equality, not a source bound or an ordering relation. Normal completion therefore requires the processing cursor to be reachable by forward one-byte advancement through valid memory.

A cursor behind the copy position, outside the relevant source storage, or otherwise unreachable by valid forward traversal is not handled as a reported input error.

The function does not inspect copied bytes for a zero value. A zero byte may be copied as ordinary span content. Whether a dependency recognizes such a byte as a terminator is not established here.

## Side effects

### Destination writes

The function directly performs only:

- Accepted-span byte writes, in increasing destination-address order.
- A single zero-byte write on candidate rejection.

There is no unconditional destination terminator. A normal return caused by a zero continuation indicator may leave the output without an appended zero byte.

The budget does not establish a bound on the number of bytes written. The byte length of an accepted span and its contribution are independently determined by the cursor and dependency result.

### Source reads

Direct source-byte reads occur only while copying accepted spans. Candidate rejection occurs before any direct copy read for that candidate.

F-d3922c7fe0ec may itself read source memory or perform other operations. Its behavior is outside the established caller contract.

### Aliasing and overlap

The function performs no overlap check. With overlapping source and destination storage, earlier writes can affect later reads. Behavior follows the forward read-then-write sequence, not a guaranteed overlap-preserving copy.

Such writes may also affect bytes examined by a dependency in a later iteration. No snapshot of future source contents is established.

### Ownership and external state

The visible caller performs no explicit allocation, release, or ownership transfer. It does not establish that either dependency avoids allocation, release, global-state mutation, or other external effects.

The address supplied to F-d3922c7fe0ec refers to a cursor stored within this invocation. No supported retention period beyond the call is established.

Source and destination argument values are not returned in updated form. Their underlying storage can nevertheless be affected by direct writes, aliasing, or dependency behavior.

## Dependencies

### F-d3922c7fe0ec

**Established interface:** receives the address of the processing cursor; returns a 32-bit value.

Its cursor update determines the endpoint of the candidate span. Its return value becomes both arguments to F-ff169115f82a.

**documented_only:** decoding rules, cursor advancement, memory access requirements, malformed-input behavior, global effects, retained references, ownership effects, and failure signaling.

### F-ff169115f82a

**Established interface:** receives two equal 32-bit argument values; returns a 64-bit quantity.

The caller interprets its lower half as a contribution and its upper half as a zero/nonzero continuation indicator.

**documented_only:** meaning of the contribution, units, valid argument domain, reasons for passing the same value twice, external effects, and error signaling.

No branch in F-8d9f1ddfe88d explicitly identifies a dependency result as an error. All returned bit patterns are handled through the contribution comparison and continuation test.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid readable source spans, adequate writable destination storage, ordinary dependency returns, and no arithmetic overflow.

1. **Zero initial budget.**  
   With any supplied pointer values and budget zero, the function returns zero without calling either dependency or writing a destination byte. It does not establish that the destination contains an empty string.

2. **Exact-budget candidate rejection.**  
   With budget five, if the first candidate has contribution five, the function writes a zero byte at the initial destination position and returns one. It copies none of that candidate’s span, even if its continuation indicator is zero.

3. **Oversized candidate after an accepted span.**  
   Start with budget seven. A first candidate contributes two, has a nonzero continuation indicator, and identifies a three-byte span. Those three bytes are copied, leaving budget five. If the next candidate contributes six, its span is not copied; a zero byte is written after the three copied bytes, and the return value is three.

4. **Accepted terminal candidate without appended termination.**  
   With budget eight, a candidate contributing three, identifying a two-byte span, and having a zero continuation indicator causes those two bytes to be copied and returns three. No additional destination byte is written.

5. **Zero contribution with nonempty span.**  
   With positive budget, a candidate contributing zero and identifying a four-byte span is accepted. If its continuation indicator is zero, the four bytes are copied and the function returns zero. A zero return therefore does not imply no destination writes.

6. **Empty span with positive contribution.**  
   With budget six, an unchanged cursor, contribution two, and a zero continuation indicator produce no copy writes and a return value of two.

7. **Zero contribution with continuation.**  
   With positive budget, unchanged cursor, zero contribution, and nonzero continuation indicator, another iteration occurs without direct destination writes or budget reduction. If dependencies repeatedly produce that condition, termination is not guaranteed.

8. **Forward overlap.**  
   If an accepted span has overlapping storage with its destination beginning one byte later, a write can replace the next source byte before that byte is read. Acceptance requires the observed output to follow this sequential behavior, not the original contents of the entire span.

## Uncertainties

- **Measure semantics — documented_only.** The contribution’s unit is unknown; it must not be equated with copied bytes.
- **Input format — documented_only.** No character encoding, record format, or source-termination rule is established.
- **Dependency safety — documented_only.** Valid cursor movement, memory bounds, global preservation, ownership behavior, and error behavior are unknown.
- **Pointer representation — documented_only.** Bytewise traversal is visible, but a pointer width need not be assumed.
- **Signed-result overflow — documented_only.** Contributions are unsigned 32-bit values, while the accumulator and return are signed 32-bit. Accepted contributions can exceed the signed range, and the rejection-path increment can also cross that range. A portable wrapping result or checked-error result is not established.
- **Fault behavior — documented_only.** Invalid reads, invalid writes, unreachable cursor endpoints, or dependency failures have no visible recovery path. Partial writes may already have occurred.
- **Textual intent — inferred only.** The rejection zero byte resembles output termination, but successful continuation-based return does not independently guarantee terminated text.

## Evidence

The supplied static function body establishes the initial zero-budget return, dependency-call order, 32-bit and 64-bit result handling, unsigned strict budget gate, forward byte-copy order, accumulator and budget updates, continuation decision, and rejection-path zero write.

The opaque dependency mapping establishes the identities F-d3922c7fe0ec and F-ff169115f82a. Their implementations are not available as evidence for this contract.

No runtime tests have been performed. Acceptance examples describe conditional observable behavior derived from the supplied static evidence; they are not execution reports.
