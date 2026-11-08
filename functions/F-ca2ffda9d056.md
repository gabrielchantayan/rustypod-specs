<!-- Trusted provenance: F-ca2ffda9d056; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca2ffda9d056

## Purpose

F-ca2ffda9d056 advances a byte-oriented state machine, updates a register-like block, invokes F-4b9706cf1b41, and conditionally invokes an indirect dependency with argument 61.

Two data-transfer directions are established:

- State 10 can read one buffer byte and write its zero-extended value to a 32-bit data word.
- States 21 and 22 read a 32-bit data word and write its low eight bits to one buffer byte.

Other paths select states, issue full-word command values, and select whether common finalization sets or clears control bit 7. Common finalization also sets bit 8 of another word.

“Transfer,” “control,” “command,” and “busy” describe observed uses. Hardware operation, interrupt handling, and protocol meaning are inferred intent, not established guarantees.

## Interface

### Explicit input and result

The function accepts one 32-bit integer argument.

- Zero suppresses the final indirect invocation.
- Any nonzero value enables that invocation if execution reaches it.
- The argument has no other observed use.
- The function returns no explicit value or status.

The indirect dependency receives the integer 61. Its return value is not inspected.

### Implicit state and register layouts

A global state reference identifies storage with these accessed fields:

| Byte offset | Width | Established use |
|---|---:|---|
| 1 | 8 bits | State discriminator |
| 4 | 32 bits | Signed position |
| 8 | 32 bits | Signed limit |
| 12 | 32 bits | Buffer address representation |

A separate global register reference identifies storage with these accessed words:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Control word; bit 7 is modified |
| 4 | 32 bits | Bit 0 is inspected; command values 144 or 208 are written |
| 12 | 32 bits | Data word |
| 16 | 32 bits | Busy word; polling continues while nonzero |
| 32 | 32 bits | Word whose bit 8 is set |

Register writes are full 32-bit writes, including a data-word write whose value comes from one byte.

The limit is used in signed comparisons. It is not independently established as a buffer capacity or allocation boundary. Negative positions are not explicitly rejected.

### Access requirements and ownership

Normal completion requires valid storage for every actual access and valid callable targets. No null, alignment, capacity, or call-target checks are present.

No allocation, deallocation, reference counting, or ownership transfer is directly established. The called dependencies may have additional effects; their allocation, retention, release, and global-state behavior are unknown.

## Behavior

### Initial reference and field observations

The initial sequence is significant:

1. Read the global register reference and retain that reference for later captured-register accesses.
2. Read the global state reference and retain that reference for the final discriminator store.
3. Read the global state reference again, then read the 32-bit position at offset 4 through that reference.
4. Read the global state reference again, then read the 8-bit discriminator at offset 1 through that reference.
5. Read the global state reference again, then read the 32-bit limit at offset 8 through that reference.
6. Read the global state reference again, then read the 32-bit buffer address at offset 12 through that reference.

The four field reads are **not** established as accesses through the captured state reference. If the global reference changes between observations, the position, discriminator, limit, and buffer address can come from different objects. The object receiving the final discriminator store can be different from all of them.

The function forms saved position plus one before selecting the state-specific path, including paths that do not transfer data. The saved field values govern subsequent decisions and buffer addressing; waiting does not refresh them.

The default finalization choice is to **set** control bit 7. Only the threshold transitions from state 20 or state 21 to state 22 select **clear**.

### Reference selection rules

“Captured register reference” below means the reference obtained by the first initial read. “Fresh global reference” means a new observation of the relevant global reference at that access.

| Path or operation | Reference used |
|---|---|
| State 10: bit-0 inspection | Fresh global register reference |
| State 10: each state-specific busy evaluation | Fresh global register reference |
| State 10: command write | Fresh global register reference |
| State 10: data-word write | Captured register reference |
| State 20: bit-0 inspection | Fresh global register reference |
| State 20: each state-specific busy evaluation | Fresh global register reference |
| State 20: command write | Fresh global register reference |
| States 21 and 22: data-word read | Fresh global register reference |
| State 22: each state-specific busy evaluation and command write | Captured register reference |
| All common-finalization register accesses | Captured register reference |
| Every position-field store | Fresh global state reference |
| Final discriminator store | Captured state reference |

Thus, a globally addressed polling loop can inspect a different block on each evaluation. Its eventual zero observation does not establish that the subsequent globally addressed command write targets the block that supplied that zero.

These are access-selection rules, not a concurrency or memory-model guarantee.

### State 10

First compare the saved signed position with the saved signed limit.

**When position is less than limit**, inspect bit 0 of the word at register offset 4 through a fresh global register reference.

If that bit is clear:

1. Poll offset 16 until zero, using a fresh global register reference for every evaluation.
2. Store saved position plus one at offset 4 through a fresh global state reference.
3. Read the byte at saved buffer address plus saved position.
4. Zero-extend that byte and write the resulting 32-bit value to offset 12 through the captured register reference.
5. Retain state 10 and the bit-7 set choice.

The position store precedes the buffer read.

If the inspected bit is set:

1. Select state 11.
2. Poll offset 16 until zero, using a fresh global register reference for every evaluation.
3. Write the full value 208 to offset 4 through a fresh global register reference.
4. Retain the bit-7 set choice.

This branch does not store a position or access the buffer.

**When position is greater than or equal to limit**, select state 12, perform the same globally addressed polling, and write command 208 through a fresh global register reference. This branch does not inspect bit 0, store a position, or access the buffer. It retains the bit-7 set choice.

### State 20

Inspect bit 0 at offset 4 through a fresh global register reference.

If set, poll offset 16 until zero using a fresh global register reference for every evaluation, then write the full value 144 to offset 4 through another fresh global register reference.

After that conditional command sequence, compare saved position plus one with the saved signed limit:

- If less, select state 21 and retain the bit-7 set choice.
- Otherwise, select state 22 and choose to clear bit 7.

Neither branch stores a position or transfers a buffer byte. When the inspected bit is clear, there is no state-specific busy wait.

### State 21

Perform these accesses and decisions in order:

1. Read the full 32-bit data word at offset 12 through a fresh global register reference.
2. Store saved position plus one at offset 4 through a fresh global state reference.
3. Store the low eight bits of the saved data word at saved buffer address plus saved position.
4. Compare saved position plus two with the saved signed limit.

If position plus two is less than limit, retain state 21 and the bit-7 set choice. Otherwise, select state 22 and choose to clear bit 7.

There is no state-specific busy wait. The threshold comparison follows the buffer write and therefore does not guard it.

### State 22

Perform these actions in order:

1. Read the full 32-bit data word at offset 12 through a fresh global register reference.
2. Store saved position plus one at offset 4 through a fresh global state reference.
3. Store the low eight bits of the saved data word at saved buffer address plus saved position.
4. Select state 24.
5. Poll offset 16 through the captured register reference until zero.
6. Write the full value 144 to offset 4 through the captured register reference.

Retain the bit-7 set choice. There is no limit comparison and no busy wait before the data-word read or buffer write. The data source and command destination can be different blocks if the global register reference changed after its initial capture.

### All other states

Retain the observed discriminator and the bit-7 set choice. There is no state-specific position store, buffer transfer, or command write.

This includes states 11, 12, and 24. They have no special branch in this function. Common finalization still occurs, so these and other unrecognized states are not no-ops.

### Common finalization

After the selected state-specific path completes:

1. Store the selected discriminator byte at offset 1 through the captured state reference.
2. Poll the busy word at offset 16 through the captured register reference until zero.
3. Read the control word at offset 0 through that captured reference.
4. Set bit 7, or clear it if a state-20 or state-21 threshold selected state 22.
5. Write the modified full word back to offset 0.
6. Poll offset 16 through the captured register reference until zero.
7. Read offset 32 through the captured register reference, set bit 8, and write the full result back.
8. Invoke F-4b9706cf1b41 with no explicit arguments.
9. If the explicit input is nonzero, obtain the indirect target from its global slot and invoke it with argument 61.

The discriminator store precedes both common waits. A nonterminating first common wait can therefore leave the selected discriminator stored without either common register update or either dependency invocation.

The bit updates preserve other bits from the words actually read. They are separate reads and writes, not established atomic operations. No additional busy evaluation occurs between either read and its corresponding write.

The indirect target is obtained after F-4b9706cf1b41 returns. Its identity must not be assumed unchanged by that dependency.

### Widths and arithmetic

Positions and limits are signed 32-bit quantities in the supplied representation. Buffer addresses are represented by 32-bit values; register words are 32 bits, while discriminator and buffer accesses are eight bits.

Consequences include:

- Outgoing byte values become data-word values from 0 through 255.
- Incoming data words lose all but their low eight bits when stored to the buffer.
- Commands replace the entire destination word with 144 or 208.
- State 20 tests position plus one; state 21 tests position plus two.
- Position updates do not saturate.
- Position plus one is formed even on paths that will not store it.

Signed-overflow behavior is not established. Neither checked failure nor guaranteed wraparound is part of this contract. Address arithmetic outside its valid range is likewise unresolved. Conditional acceptance cases require representable calculations and valid addressed storage.

## Side effects

Direct writes can include:

- One discriminator-byte store on every path reaching common finalization.
- One position store on the state-10 byte-transfer path and in states 21 and 22.
- One buffer-byte store in states 21 and 22.
- One full data-word write on the state-10 byte-transfer path.
- One full command write on the applicable state-10, state-20, and state-22 paths.
- The common control-word and offset-32 read-modify-writes.

Polling performs repeated full-word reads. No timeout, retry bound, fallback, or explicit error result is present. Any required wait can prevent further progress indefinitely. Observing zero only permits the following operation; it does not establish that the word remains zero.

Reference changes can split one invocation across multiple state objects and register blocks, including during initial field collection and globally addressed polling. The captured references themselves remain the destinations for the specified captured-reference accesses.

Aliasing is not excluded. For example, a position store can affect the later buffer read if their storage overlaps. A buffer store can affect storage accessed during finalization. The saved scalar values are not refreshed in response to such writes.

Dependency effects are additional and unknown. Normal return is not guaranteed if memory access fails, a wait never completes, or a dependency fails to return.

## Dependencies

### F-4b9706cf1b41

Called once if execution completes the common register updates.

- No explicit arguments.
- No inspected return value.
- Runs after the offset-32 update.
- Runs before the optional indirect target is obtained.

Its global mutations, memory ownership behavior, hardware effects, termination, and failure behavior are documented_only.

### Unidentified indirect dependency

No opaque function ID is supplied for this target.

- Invoked only for a nonzero explicit input and only after F-4b9706cf1b41 returns.
- Target obtained from a global slot at that stage.
- Receives integer 61.
- No validity check or inspected return value.

Its identity, additional calling-convention details, effects, ownership behavior, and termination are documented_only.

## Acceptance examples

These are conditional static cases, not executed tests. Unless a case states otherwise, assume stable references, valid nonoverlapping storage, representable arithmetic, eventually terminating waits, and returning dependencies.

1. **State-10 byte transfer.** With position 2, limit 5, inspected bit 0 clear, and buffer byte 167 at position 2, position becomes 3 before data-word value 167 is written. The discriminator remains 10. Common finalization sets control bit 7 and offset-32 bit 8.

2. **State-10 alternatives.** With position 2, limit 5, and inspected bit 0 set, command 208 follows the state-specific wait and state 11 is selected. With position 5 and limit 5, state 12 is selected and command 208 follows the wait without a bit-0 inspection. Neither case transfers a byte or updates position.

3. **State-20 boundary.** With position 3 and limit 5, state 21 is selected and control bit 7 is set. With position 4 and limit 5, state 22 is selected and bit 7 is cleared. Position is unchanged. Command 144 occurs only if the bit-0 inspection returned set.

4. **State-21 thresholds and truncation.** With position 2, limit 5, and data word 305419896, position becomes 3 and buffer byte 120 is stored at position 2. State 21 remains selected because 4 is less than 5. With position 3 and limit 5, the transfer still occurs first, position becomes 4, and state 22 is selected because 5 is not less than 5.

5. **State-22 unguarded transfer.** Regardless of the saved limit, a valid addressed buffer byte receives the data word’s low eight bits. Position advances, command 144 follows the captured-register wait, and state 24 is stored. Common finalization sets bit 7.

6. **Split initial state observations.** Suppose the captured state reference identifies object A, while successive initial field-reference reads identify B, C, D, and E. Position comes from B, discriminator from C, limit from D, and buffer address from E. The final discriminator store targets A. Any position store uses a further global-reference observation.

7. **Changing register reference in state 10.** Suppose register block A was captured, bit 0 is read clear from B, and successive state-specific busy evaluations inspect C and D, with D returning zero. The byte-transfer data write still targets A. Common register finalization also targets A.

8. **Different state-22 source and destination.** If A was captured but the global reference identifies B at the data read, the received word comes from B. The later state-specific busy wait, command 144, and common register operations target A.

9. **Unknown state and callback gating.** State 99 is written back without a state-specific transfer, but common bit updates and F-4b9706cf1b41 still occur. Input zero suppresses only the indirect invocation; negative and positive nonzero inputs both enable it.

10. **Partial progress before a wait.** If the first common busy wait never observes zero, the selected discriminator has already been stored. Neither common register update nor either dependency invocation occurs.

## Uncertainties

The following remain documented_only:

- Protocol meaning of states, commands, bits, and register-like storage.
- Volatile semantics, bus behavior, read side effects, and memory ordering.
- Valid position ranges, buffer capacity, and external restrictions on negative positions.
- Signed-overflow and address-overflow behavior.
- Concurrency, reentrancy, aliasing, and reference-stability guarantees.
- Dependency effects, ownership, failures, and termination.
- Machine-level details of the final indirect transfer beyond its observed argument.

No explicit error-reporting mechanism is established. Invalid accesses, invalid targets, arithmetic outside the established range, and nonterminating waits are not converted into defined status results.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping. The prior draft and feedback are not independent evidence.

Established evidence includes the ordered initial reference reads, separate field-reference observations, path-specific reference selection, access widths, signed comparisons, polling conditions, byte extension and truncation, state choices, full-word commands, bit updates, and dependency order.

Hardware-service and transfer-direction terminology describes inferred intent only. No dependency implementation, register specification, ownership documentation, or independent runtime observation was supplied.

No runtime tests have been performed.
