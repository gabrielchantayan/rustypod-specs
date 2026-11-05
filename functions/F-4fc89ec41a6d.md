<!-- Trusted provenance: F-4fc89ec41a6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4fc89ec41a6d

## Purpose

F-4fc89ec41a6d updates one position in a 16-slot shared structure. It advances a primary cursor, advances a secondary cursor if the two would coincide, records a packed 32-bit value and two additional 32-bit values, and conditionally invokes a dependency before clearing slot-associated state.

The cursor handling is consistent with an overwrite-on-collision ring buffer. That interpretation is inferred intent, not an established consumer contract: the supplied evidence does not identify the reader, define queue occupancy, or establish how records are consumed.

The function returns no value. Its observable results are writes to the structure and, under a specific condition, a call to F-5990a26eaaa3.

## Interface

### Inputs

The function accepts five arguments:

| Argument | Recovered representation | Established use |
|---|---|---|
| First | Signed 32-bit integer | Supplies the upper eight bits of the packed record through a left shift of 24 bits. |
| Second | Unsigned 32-bit integer | Supplies the middle twelve bits of the packed record. |
| Third | Unsigned 32-bit integer | Supplies the lower twelve bits of the packed record. |
| Fourth | Uninterpreted 32-bit value | Stored unchanged in one slot-associated array. |
| Fifth | Uninterpreted 32-bit value | Stored unchanged in another slot-associated array. |

No input is explicitly validated. The fourth and fifth arguments are copied as 32-bit values; their meaning as numbers, pointers, handles, or other data is unknown.

### Implicit structure

F-035afcc6e793 is called without explicit arguments to obtain the structure location. The recovered representation uses a 32-bit integer for that location. Subsequent accesses treat it as a base address.

The following layout is established by the accesses. Offsets are byte offsets from that base; every listed element is four bytes wide.

| Offset or range | Layout | Use in this function |
|---|---|---|
| 0x004–0x043 | 16 packed-record elements | Writes one packed value. |
| 0x044–0x083 | 16 indicator elements | Tests one element for nonzero; conditionally clears an element after a dependency call. |
| 0x084–0x0C3 | 16 flag elements | Conditionally reads one element; clears an element on normal completion. |
| 0x0C4–0x103 | 16 elements | Stores the fourth argument. |
| 0x104–0x143 | 16 elements | Stores the fifth argument. |
| 0x144–0x147 | Primary cursor | Advances and repeatedly reads it for slot selection. |
| 0x148–0x14B | Secondary cursor | Compares it with the advanced primary cursor; advances it on equality. |

For any array, slot selection adds four times the selected index to that array’s starting offset. Byte order within each 32-bit element is not established.

The field at offset zero, if any, is not accessed.

### Preconditions

For the ordinary 16-slot interpretation:

- F-035afcc6e793 must return a usable structure address.
- The structure must support the relevant four-byte reads and writes through offset 0x14B.
- Both cursors should initially be in the range 0 through 15.
- Address calculations must resolve to the intended storage.
- Execution must not encounter an invalid access or a failure inside a dependency.

The function does not check these conditions.

Stable slot targeting additionally requires that no intervening actor change the primary cursor. This includes F-5990a26eaaa3 when that dependency is invoked. The function itself does not establish that guarantee.

## Behavior

### Cursor advancement and collision handling

The first observable operation is a call to F-035afcc6e793.

Using the returned base, the function reads the primary cursor, adds one, takes the signed remainder on division by 16, and writes the result back to the primary cursor.

For valid initial cursor values:

- Values 0 through 14 advance to 1 through 15.
- Value 15 advances to 0.

The newly computed value is retained for the collision comparison and the packed-record destination.

After writing the primary cursor, the function reads the secondary cursor and compares it with the newly computed primary value. If they are equal, it reads the secondary cursor again, adds one, takes the signed remainder on division by 16, and writes that result to the secondary cursor. Otherwise, it leaves the secondary cursor unchanged.

This is an equality-based collision adjustment. There is no separate full-state test, empty-state test, count update, or error report.

The second read of the secondary cursor matters if the field can change between the comparison and update. The evidence does not establish synchronization.

### Packed-record construction

The function next writes a 32-bit packed value into the packed-record array, using the newly computed primary index retained from the advancement step.

Under 32-bit bit-vector interpretation, the stored layout is:

| Packed bits | Input contribution |
|---|---|
| 31–24 | Low eight bits of the first argument |
| 23–12 | Low twelve bits of the second argument |
| 11–0 | Low twelve bits of the third argument |

The second argument is masked with 0xFFF00FFF before a 12-bit left shift. At a 32-bit result width, the retained high portion shifts out of the result. Consequently, only the original low twelve bits contribute to the stored word.

The third argument is masked with 0xFFF. Its higher bits do not affect the packed word.

The recovered expression shifts the signed first argument left by 24 bits. Its unrestricted signed-language semantics are not established. The field layout above describes the 32-bit bit effect; acceptance examples below use nonnegative first arguments whose shift is representable.

### Additional value stores

After the packed-record write, the function:

1. Reads the primary cursor again and writes the fourth argument into the corresponding element beginning at offset 0x0C4.
2. Reads the primary cursor again and writes the fifth argument into the corresponding element beginning at offset 0x104.

These stores preserve the arguments’ 32-bit representations without an explicit transformation.

Unlike the packed-record write, these stores select their destinations through fresh cursor reads. If the primary cursor changes between operations, the three values need not land in the same logical slot.

### Conditional dependency call

The function then reads the primary cursor and uses that value to select an indicator element.

It reads the indicator and compares the full 32-bit value with zero:

- If the indicator is zero, it does not read the corresponding flag element and does not invoke F-5990a26eaaa3.
- If the indicator is nonzero, it reads the flag element for that same selected slot.

F-5990a26eaaa3 is invoked only when both conditions hold:

- The indicator was nonzero.
- Bit zero of the flag value read was set.

Other flag bits do not affect this decision. Any nonzero indicator qualifies; the indicator is not restricted to the value one.

The call supplies no explicit arguments. In particular, the indicator value, flag value, structure address, and slot index are not explicitly passed.

If the dependency returns normally, the function reads the primary cursor again and clears the indicator element selected by that new read. Thus, the cleared indicator is not necessarily the one that triggered the call.

If the call is skipped, no indicator is cleared by this invocation.

### Final flag clearing

After the conditional-call branch, the function reads the primary cursor again and writes zero to the selected flag element.

This final write occurs on every normally completing path, whether:

- The indicator was zero.
- The indicator was nonzero but the flag’s low bit was clear.
- F-5990a26eaaa3 was invoked and returned.

The function then returns without a value.

There is no established rollback. If a dependency fails to return or an access fails, earlier writes may already have occurred.

## Side effects

With valid, stable cursors and no relevant dependency mutation, the function:

- Advances the primary cursor.
- Advances the secondary cursor only on collision.
- Replaces the selected packed record.
- Replaces both additional 32-bit values.
- Clears the selected flag.
- Clears the selected indicator only after invoking F-5990a26eaaa3.

Existing packed and additional values are overwritten without being inspected. No explicit release operation is tied to those overwritten values.

The conditional dependency call cannot be classified as a deallocation, destructor, notification, or ownership transfer from this evidence alone. Its lack of explicit arguments does not establish that it is side-effect-free or independent of shared state.

The function obtains the structure address once and continues using that same base after the conditional call. It does not reacquire the structure. If the dependency invalidates that storage, subsequent accesses are unsafe; the supplied evidence provides no lifetime guarantee.

The observable sequence includes publishing the advanced cursor before writing the new record. No lock, atomic operation, memory barrier, or transaction boundary is explicit. Concurrent consumers must not infer that observing the advanced cursor guarantees observation of fully updated slot contents.

## Dependencies

### F-035afcc6e793

- Called once, before structure accesses.
- Receives no explicit arguments.
- Supplies the address used for all subsequent structure accesses.
- Allocation, initialization, lifetime, failure behavior, and shared-state effects are unknown.
- Its result is not checked for a null or invalid address.

### F-5990a26eaaa3

- Called at most once per invocation.
- Receives no explicit arguments.
- Called only after cursor adjustment and all three record-value writes.
- Requires a nonzero selected indicator and a selected flag with bit zero set.
- Its return value, if any, is not used.
- On normal return, indicator clearing precedes final flag clearing.
- Preservation of the structure, cursors, globals, and memory ownership is unknown.

No additional dependency behavior is established.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid storage, initial cursors in range, no concurrent changes, and normally returning dependencies.

### Ordinary insertion without collision

Given primary cursor 2 and secondary cursor 9, the primary cursor becomes 3 and the secondary remains 9.

With first argument 0x12, second argument 0xABC, and third argument 0xDEF, slot 3 receives packed value 0x12ABCDEF. The fourth and fifth arguments are stored unchanged in slot 3 of their respective arrays.

If slot 3’s indicator is zero, F-5990a26eaaa3 is not called. Slot 3’s flag becomes zero. Its indicator remains zero.

### Wraparound with collision

Given primary cursor 15 and secondary cursor 0, the primary cursor becomes 0. Because the new primary equals the secondary, the secondary advances to 1.

With stable state, all slot-associated operations target slot 0.

### Truncation of unsigned inputs

For second argument 0xABCDEABC and third argument 0x12345DEF, the packed contributions are respectively 0xABC and 0xDEF.

With first argument 0x12, the packed word is therefore 0x12ABCDEF, the same as when the unsigned arguments are 0xABC and 0xDEF.

### Nonzero indicator without the low flag bit

If the selected indicator is 7 and the selected flag is 0x80000002, the dependency is not called. The indicator remains 7 and the flag becomes zero.

The high set flag bit does not independently trigger the call.

### Dependency call followed by clearing

If the selected indicator is nonzero and the selected flag is 0x80000001, F-5990a26eaaa3 is called.

If it returns without changing relevant state, the selected indicator becomes zero and then the selected flag becomes zero.

### Zero indicator suppresses the flag read

If the selected indicator is zero, its flag value is not read before the branch decision, even if that flag would have bit zero set.

The final flag write still occurs. No dependency call occurs.

### Dependency changes the primary cursor

Suppose slot 4 supplies the nonzero indicator and low-bit-set flag that trigger the call. If F-5990a26eaaa3 changes the primary cursor to 6 and leaves it there, the subsequent indicator and flag clears target slot 6.

The earlier packed and additional-value stores are not repeated for slot 6. This example illustrates fresh cursor reads; it does not establish that such dependency mutation is intended or safe.

## Uncertainties

The following are documented_only:

- The semantic names of the fields and the consumer protocol.
- Whether the secondary cursor denotes a reader position, oldest entry, or another boundary.
- Whether collision represents intentional record eviction.
- The meanings and ownership of the two uninterpreted argument values.
- The meaning of the indicator and flag bits other than the observed low-bit condition.
- Whether F-5990a26eaaa3 releases memory or mutates shared state.
- Whether either dependency can fail, terminate execution, or reenter this function.
- Byte order, platform alignment requirements, and concurrency guarantees.
- Exact executable behavior for signed overflow, negative cursor values, and out-of-range signed shifts.

Signed remainder must not be treated as a general normalization into 0 through 15. For example, incrementing a sufficiently negative cursor can leave a negative remainder, leading to accesses outside the nominal array. Addition can also overflow for an invalid extreme cursor value. No validation or recovery is present.

## Evidence

Established static evidence consists of the supplied function body and dependency mapping: five 32-bit arguments, two opaque calls, fixed-offset four-byte accesses, signed cursor arithmetic, masked packing, repeated primary-cursor reads, and conditional state clearing.

The ring-buffer description is inferred from the 16-position cursor progression and collision adjustment. Ownership, synchronization, dependency semantics, and broader record meaning remain unknown.

No runtime tests have been performed.
