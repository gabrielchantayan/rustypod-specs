<!-- Trusted provenance: F-ecd712a1deb9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ecd712a1deb9

## Purpose

F-ecd712a1deb9 dispatches an event-like record through callbacks associated with a receiver-like object. Its observable behavior consists of:

- An optional preliminary notification for one event category and payload value.
- An optional early callback for another payload value.
- Exactly one of two main callbacks, unless the early callback returns a nonzero result.
- An optional final callback when the current event identifies a value matching a receiver field.
- Immediate propagation of a nonzero result from any callback whose result is inspected.

These descriptions are structural. The meanings of the event categories, payload values, receiver fields, and callback roles are not established.

A central requirement is that event classification is repeated between callback stages. A callback may change memory used by later decisions. The contract therefore does not treat the event, receiver fields, or dispatch-table pointer as immutable snapshots.

## Interface

### Inputs

The function accepts two machine-level arguments:

1. **Receiver address.** An address used to access a receiver object, including its dispatch-table pointer and two other fields.
2. **Event address.** An address used to access an event record and, for one category, a referenced record.

The supplied representation uses 32-bit integer values and integer-held addresses. The layouts below describe that apparent 32-bit target model. They are not a portable host-language ABI specification. Pointer representation, byte order, alignment requirements, and calling convention require external confirmation.

All offsets below are byte offsets.

### Receiver layout used by this function

| Offset | Accessed representation | Role established by the function |
|---|---|---|
| `0x00` | Address-sized field in the apparent 32-bit model | Dispatch-table address |
| `0x50` | 32-bit integer | Comparison value used by the final callback condition |
| `0x10C` | 32-bit integer, subsequently converted to a character-width argument | Preliminary notification argument |

The field at receiver offset `0x10C` is distinct from the callback slot at dispatch-table offset `0x10C`. Their equal numeric offsets do not imply any shared storage or meaning.

### Event layout used by this function

| Offset | Accessed representation | Role established by the function |
|---|---|---|
| `0x04` | 32-bit integer | Event category |
| `0x08` | 32-bit value | Referenced-record address for category `0x16`; comparison value for category `0x0F` |

For category `0x16`, the referenced record is accessed as follows:

| Offset | Accessed representation | Role established by the function |
|---|---|---|
| `0x04` | 32-bit integer | Payload selector |

No access to event offset `0x00` is established.

### Callback interface

The indirect callbacks are invoked with the receiver address and event address. Their results are inspected as 32-bit integers.

The direct dependency F-4fd087f2325e receives:

- The receiver address.
- A character-width conversion of the 32-bit value read at receiver offset `0x10C`.

The direct dependency’s result, if it has one at the ABI level, is not inspected by this function.

### Output

The function returns a 32-bit integer:

- The exact nonzero result of the first result-bearing callback that returns nonzero; or
- Zero if every invoked result-bearing callback returns zero.

Negative results are nonzero and therefore terminate dispatch just as positive nonzero results do. No normalization into a Boolean, success code, or error category is established.

## Behavior

### 1. Preliminary notification decision

The function first reads the event category at offset `0x04`.

If the category is not `0x16`, this stage does not dereference the referenced-record field.

If the category is `0x16`, the function reads the address at event offset `0x08`, then reads the selector at offset `0x04` within that referenced record. The supplied representation tests whether subtracting `0x2500` from this selector produces `2`.

For ordinary, defined arithmetic, selector `0x2502` satisfies this test. Under that condition:

1. The 32-bit receiver field at offset `0x10C` is read.
2. That value is converted to a character-width argument.
3. F-4fd087f2325e is invoked with the receiver and converted value.

There is no result-based early return from this direct call.

The representation explicitly includes the subtraction. Its behavior for signed values that would overflow a 32-bit signed subtraction is not independently established. A universal wrapping-arithmetic rule must not be inferred solely from this representation.

The character conversion does not establish whether values with their high character bit set arrive as positive or negative values. Character width and signedness are ABI-dependent unknowns. On a conventional 8-bit-character target, the conversion concerns the low eight bits, but the signed interpretation still needs confirmation.

### 2. Early callback decision

After the preliminary stage, including after F-4fd087f2325e returns if called, the function reads the event category again.

If that current category is `0x16`, it reads the current referenced-record address and the current selector at referenced-record offset `0x04`.

If the current selector is zero:

1. The receiver’s dispatch-table pointer is obtained.
2. The callback address at dispatch-table offset `0x124` is obtained.
3. That callback is invoked with the receiver and event addresses.
4. A nonzero callback result is returned immediately.

If the callback returns zero, processing continues. If the category or selector does not match, this callback is skipped.

This stage uses fresh reads. Changes made by F-4fd087f2325e can therefore cause or prevent this callback, including by changing the event category, referenced-record address, selector, or receiver dispatch table.

### 3. Main callback selection and invocation

Unless the early callback has returned a nonzero result, the event category is read again.

The callback choice is:

- Dispatch-table offset `0x10C` if the current category is `0x16` and the currently referenced selector is `0x2A`.
- Dispatch-table offset `0x110` in every other case.

For category `0x16`, the referenced-record address is read before its selector is read. For all other categories, this decision does not require dereferencing the event’s offset-`0x08` value.

Exactly one of these two callbacks is invoked at this stage. It receives the receiver and event addresses.

A nonzero result is returned immediately. A zero result permits the final stage.

The callback choice is based on state at this stage, not the state used for earlier decisions. In particular, a zero-returning callback from table offset `0x124` may change which main callback is selected.

The dispatch-table pointer used to obtain the selected callback is not established as a cached value from an earlier callback invocation.

### 4. Final callback decision

After the selected main callback returns zero, the event category is read again.

If the current category is not `0x0F`, no final callback is invoked and the function returns zero.

If the current category is `0x0F`:

1. The 32-bit value at event offset `0x08` is read.
2. The 32-bit receiver field at offset `0x50` is read.
3. Those values are compared for equality.

If they are unequal, the function returns zero.

If they are equal, the current receiver dispatch table supplies the callback at offset `0xE0`. That callback is invoked with the receiver and event addresses.

A nonzero result from this callback is returned unchanged. A zero result leads to the function’s final zero return.

The main callback can therefore enable or disable this final callback by changing the event category, the event comparison value, the receiver comparison field, or the receiver’s dispatch-table pointer.

### Overall sequencing and termination

The observable call sequence is constrained to this order:

1. Optional F-4fd087f2325e invocation.
2. Optional callback through table offset `0x124`.
3. One callback through table offset `0x10C` or `0x110`.
4. Optional callback through table offset `0xE0`.

A nonzero result at a result-bearing stage prevents all subsequent stages and their associated reads.

If all invoked dependencies return normally, at least one main callback is invoked unless the callback at offset `0x124` returns nonzero. At most four dependency invocations occur.

Within a stage, the table describes relevant dereferences and their data dependencies. It does not establish an additional observable ordering between independent expression evaluations beyond what the supplied representation supports.

## Side effects

### Direct effects

No explicit write to receiver, event, referenced-record, dispatch-table, global, or heap storage is present in the supplied function.

There is no explicit allocation, deallocation, ownership transfer, reference-count update, or resource release.

The direct observable actions are memory reads, dependency calls, and the returned result.

### Effects through dependencies

Dependencies may modify memory or external state. Their implementations are unavailable, so this contract does not assume that they:

- Preserve receiver or event contents.
- Preserve the dispatch-table pointer.
- Preserve globals.
- Retain, acquire, or release ownership.
- Avoid reentrant calls.
- Return normally.
- Leave input allocations alive.

Later reads are conditional on earlier calls returning and on the referenced storage still being valid.

No ownership change is established merely because an address is passed to a dependency. Conversely, the lack of an explicit release in this function does not prove that dependencies cannot release memory.

### Validity and error handling

There are no explicit null checks, bounds checks, dispatch-target checks, or recovery paths for invalid addresses.

The initial event-category field must be readable. Additional storage must be valid when its branch is reached:

- A category-`0x16` stage requires a readable referenced-record address and selector.
- A reached indirect invocation requires a valid receiver table pointer, readable callback slot, and callable target.
- The preliminary notification requires the receiver field at `0x10C`.
- The category-`0x0F` final decision requires the receiver field at `0x50`.

An invalid address or callback target is not converted into a documented integer error result. Its outcome is outside the established contract.

No concurrency or atomicity guarantees are established. Callback-driven changes are supported by the repeated reads; that fact does not establish safe behavior under unsynchronized concurrent mutation.

## Dependencies

### Identified direct dependency

**F-4fd087f2325e**

Called only when the initial category is `0x16` and the selector satisfies the preliminary arithmetic condition.

Arguments are the receiver and the converted receiver field at offset `0x10C`. Its result is ignored. Its state effects, ownership behavior, and failure behavior are unknown.

### Dynamically resolved dependencies

Opaque function identities for the indirect targets were not supplied. They remain **documented_only** dependencies identified by dispatch-table location, not invented function IDs.

| Dispatch-table offset | Invocation condition | Result handling |
|---|---|---|
| `0x124` | Current category `0x16`, current selector zero | Nonzero returned immediately |
| `0x10C` | Main-stage category `0x16`, current selector `0x2A` | Nonzero returned immediately |
| `0x110` | All other main-stage classifications | Nonzero returned immediately |
| `0xE0` | Final-stage category `0x0F`, event value equals receiver field | Nonzero returned immediately |

These locations do not prove distinct function identities. Multiple slots may resolve to the same function. Different invocations may resolve to different targets if dependencies change the table pointer or table contents.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, all storage is valid, calls return normally, and dependencies do not mutate relevant state.

1. **Ordinary unmatched category.**  
   For category `0x07`, invoke the main callback at table offset `0x110`. If it returns zero, return zero. Do not dereference event offset `0x08` as a referenced-record address.

2. **Preliminary notification.**  
   For initial category `0x16` and selector `0x2502`, invoke F-4fd087f2325e with the converted receiver field. With unchanged event state, skip table offset `0x124`, invoke table offset `0x110`, and propagate its nonzero result or continue after zero.

3. **Early termination.**  
   For category `0x16` and selector zero, invoke table offset `0x124`. If it returns `-3`, return `-3` without selecting a main callback or reading fields for the final comparison.

4. **Early callback declines.**  
   For category `0x16` and selector zero, if table offset `0x124` returns zero without mutation, invoke table offset `0x110`. Do not invoke table offset `0x10C`.

5. **Special main selection.**  
   For category `0x16` and selector `0x2A`, invoke table offset `0x10C`, not `0x110`. A nonzero result ends processing immediately.

6. **Matching final condition.**  
   For category `0x0F`, invoke table offset `0x110` first. If it returns zero and the event value equals the receiver field at `0x50`, invoke table offset `0xE0`. Return its result, including zero.

7. **Unequal final values.**  
   For category `0x0F`, if the main callback returns zero and the comparison values differ, skip table offset `0xE0` and return zero.

8. **Mutation after preliminary notification.**  
   Begin with category `0x16` and selector `0x2502`. If F-4fd087f2325e changes the selector to zero, the subsequent fresh classification must invoke table offset `0x124`.

9. **Mutation after early callback.**  
   Begin with category `0x16` and selector zero. If the callback at offset `0x124` changes the selector to `0x2A` and returns zero, the main callback must come from offset `0x10C`.

10. **Mutation after main callback.**  
    If the main callback changes the event category to `0x0F`, makes the comparison values equal, and returns zero, invoke the final callback. The event’s initial category does not suppress this newly satisfied condition.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI, pointer width confirmation, endianness, and alignment rules.
- Character width, character signedness, and argument-extension behavior.
- Behavior of the preliminary subtraction outside its defined signed-arithmetic range.
- Semantic names for event categories and payload selectors.
- Identities and full contracts of indirect dependencies.
- Dependency effects on globals, memory lifetime, ownership, exceptions, and reentrancy.
- Whether callers enforce validity, synchronization, or lifetime preconditions.
- Whether nonzero callback results signify handling, errors, or another application-specific condition.

“Inferred intent” is limited to describing the function as an event dispatcher with short-circuiting callback results. That description does not establish application semantics or memory-ownership guarantees.

## Evidence

Established static evidence comprises the supplied function representation and the supplied mapping for F-ecd712a1deb9 and F-4fd087f2325e.

That evidence supports the accessed offsets, comparison constants, conditional dereferences, callback sequence, repeated classifications, lack of explicit memory writes, and result-propagation behavior described above.

No dependency implementation, caller contract, runtime trace, or independent ABI description was supplied. No runtime tests have been performed.
