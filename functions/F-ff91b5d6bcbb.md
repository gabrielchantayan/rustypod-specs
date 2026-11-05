<!-- Trusted provenance: F-ff91b5d6bcbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff91b5d6bcbb

## Purpose

F-ff91b5d6bcbb performs a conditional state transition involving a caller-supplied record and an associated object. It selects a path using a signed 32-bit selector, invokes dependencies according to that path, performs common finalization, and returns the bitwise intersection of two 32-bit words read at the end.

Established behavior consists of the explicit accesses, branch conditions, dependency arguments, and ordering described below. Application-level meanings such as “commit,” “cancel,” “release,” or “notify” are not established.

Two values come from external storage. They are read at their respective points of use, not captured at entry and not established as immutable. Preceding dependencies may change either value.

This contract describes executions with valid reached accesses and normally returning dependencies. Invalid-storage behavior, dependency failure semantics, and concurrency guarantees remain documented_only unknowns.

## Interface

### Arguments and record layout

The function receives:

- **State record:** a pointer to mutable storage.
- **Forwarded argument:** an opaque 32-bit value passed unchanged to F-00cf49e09ab2. The visible function does not otherwise inspect it.

The state record has the following visible accesses. Offsets are in bytes.

| Offset | Access width | Role |
|---|---:|---|
| `0x00` | 32 bits | Associated-object address captured before the first dependency call |
| `0x44` | 32 bits | Destination for a value read from external storage after common finalization |
| `0x70` | 32 bits | Branch selector, interpreted as signed |
| `0x74` | 32 bits | Transition word used by branch work and read again for the result |
| `0xF4` | 32 bits | Pending word, tested or passed to a dependency and conditionally cleared |
| `0xFE` | 8 bits | Byte cleared in the common tail |
| `0xFF` | 8 bits | Gate byte read only on a particular negative-selector path |

The address read from `0x00` identifies the **captured associated object**. Direct accesses through that captured address use:

| Offset | Access width | Role |
|---|---:|---|
| `0x14` | 32 bits | Destination for a conditional transition-word write |
| `0x18` | 32 bits | Mask operand for the return value |
| `0xC8` | 32 bits | First argument to F-df71e640897e |

These names describe access roles, not recovered object types or ownership declarations.

### External storage

Two distinct external-storage references appear:

- **External argument storage:** read to obtain the fifth argument to F-df71e640897e whenever that dependency is invoked. Its application-level type, contents, and storage width are not independently declared in the supplied material.
- **External finalization storage:** read to obtain the 32-bit value written to state offset `0x44` after F-00cf49e09ab2 returns.

Neither value is a demonstrated constant. No preservation guarantee applies across dependency calls. The exact point of each read is part of the behavior below.

### Result

On normal completion, the function returns an unsigned 32-bit word equal to the bitwise AND of:

- the word read from state offset `0x74`; and
- the word read from captured-associated-object offset `0x18`.

Both reads occur after common finalization and the explicit tail writes. Earlier transition-word reads do not determine the result if the word changes before the final read.

### Preconditions

All reached accesses must address usable storage. Across possible paths, direct state-record accesses extend through byte `0xFF`. This access range does not establish an allocation size.

The associated-object address captured at entry must remain usable for subsequent reached accesses, including the unconditional final read at object offset `0x18`. A dependency replacing the address in state offset `0x00` does not redirect those accesses.

External storage must be readable when its respective read is reached. Dependencies may impose additional validity, alignment, lifetime, or ownership requirements; their contracts are unavailable.

The function contains no explicit pointer, bounds, alignment, or lifetime checks.

## Behavior

### Entry capture and selector read

The function first reads the 32-bit associated-object address from state offset `0x00` and retains that address for its own later object accesses.

It then invokes F-2523efe3304d with no explicit arguments.

After that dependency returns, the function reads the selector at state offset `0x70` and interprets it as a signed 32-bit integer:

- Bit 31 set: negative-selector branch.
- Bit 31 clear: nonnegative-selector branch, including selector zero.

Consequently, F-2523efe3304d may affect the selected branch by changing the selector. It may also affect later field and external-storage reads. It cannot redirect this function’s retained object address merely by replacing state offset `0x00`.

### Negative-selector branch

The function reads the 32-bit transition word at state offset `0x74`.

If the transition word is zero, it proceeds directly to common finalization. On this path it does not read the gate byte, invoke either branch-specific dependency, or directly clear pending.

If the transition word is nonzero, the function reads the byte at state offset `0xFF`. A zero gate byte also sends execution directly to common finalization, without either branch-specific call or a direct pending clear.

If both the transition word and gate byte are nonzero, execution proceeds as follows:

1. F-a061acf353ed receives the captured associated-object address, the already-read transition word, and a 32-bit zero.
2. After that dependency returns, F-df71e640897e receives:
   - the 32-bit word read from captured-object offset `0xC8`;
   - the 32-bit all-ones value `0xFFFFFFFF`;
   - the pending word read from state offset `0xF4`;
   - the value one;
   - the value read from external argument storage during evaluation of this call’s arguments.
3. After F-df71e640897e returns, the function writes a 32-bit zero to state offset `0xF4`.
4. Execution reaches common finalization.

There is no pending-zero test on this branch. Pending zero therefore does not suppress F-df71e640897e when both gates pass.

The object-field, pending, and external-argument reads for F-df71e640897e occur after F-a061acf353ed returns. Mutations by that earlier dependency can affect all three arguments. The source does not establish their relative evaluation order.

The transition passed to F-a061acf353ed is the word already read before that call. This branch contains no direct write to object offset `0x14`.

### Nonnegative-selector branch

The function first reads pending at state offset `0xF4`.

**If pending is zero**, F-a061acf353ed receives:

- the captured associated-object address;
- the transition word read from state offset `0x74`;
- a 32-bit zero.

Execution then reaches common finalization. This path does not invoke F-df71e640897e, read its external argument, or directly clear pending afterward.

There is no transition-zero or gate-byte condition on this path. F-a061acf353ed is called even when the transition word is zero. If that dependency changes pending to a nonzero value, this path contains no later direct pending clear.

**If pending is nonzero**, execution proceeds as follows:

1. F-df71e640897e receives the object word at `0xC8`, `0xFFFFFFFF`, the pending word, one, and the value read from external argument storage during argument evaluation.
2. After that dependency returns, the function reads the transition word at state offset `0x74`.
3. It writes that transition word to captured-object offset `0x14`.
4. It writes a 32-bit zero to state offset `0xF4`.
5. Execution reaches common finalization.

This path does not invoke F-a061acf353ed.

The pending test and the pending argument are distinct source-level reads, with no intervening dependency call. No atomic snapshot or concurrency protocol is established.

The external-argument read belongs to F-df71e640897e’s argument evaluation, not to entry capture or the pending test. Its ordering relative to the independent object-field and pending reads is unspecified by the supplied expression.

The transition copied to object offset `0x14` is read after F-df71e640897e returns. A dependency mutation of that word therefore affects the copied value.

### Common finalization and return

Every path that reaches the common tail performs these operations in order:

1. Invoke F-00cf49e09ab2 with the original state-record pointer and the forwarded 32-bit argument.
2. After that dependency returns, read the 32-bit value from external finalization storage.
3. Write that value to state offset `0x44`.
4. Write an 8-bit zero to state offset `0xFE`.
5. Read the two return operands and return their 32-bit bitwise AND.

The external finalization value is not an entry-time snapshot. F-2523efe3304d, any branch-specific dependencies, and F-00cf49e09ab2 may change its storage before the read. In particular, a change made by F-00cf49e09ab2 can determine the value subsequently written to `0x44`.

Both return operands are read after the explicit tail writes. Their relative read order is not established.

The byte write at `0xFE` does not itself clear the adjacent gate byte at `0xFF`. Other writes, dependency effects, or aliasing may affect that gate.

## Side effects

### Direct writes

| Destination | Written value | Condition and ordering |
|---|---|---|
| State `0xF4`, 32 bits | Zero | After either invocation path of F-df71e640897e returns |
| Captured object `0x14`, 32 bits | Transition word read after F-df71e640897e | Only on the nonnegative-selector, nonzero-pending path; before the pending clear |
| State `0x44`, 32 bits | Value read from external finalization storage | After F-00cf49e09ab2 returns |
| State `0xFE`, 8 bits | Zero | After the state `0x44` write |

The pending clear and conditional object write precede common finalization. Without that dependency’s contract, neither write establishes an unconditional final value at its destination.

The function performs no explicit write to either external-storage reference. This does not establish that those locations remain unchanged: dependencies may modify them, and aliasing restrictions are unknown.

### Ownership and dependency effects

There is no direct allocation or release operation in the visible body. Dependencies may allocate, release, retain, or otherwise affect storage; their behavior is not established.

No ownership transfer is established for either argument, the captured object address, pending, or either externally read value. Passing a value to a dependency does not prove consumption, retention, or release.

No assumption is made that dependencies preserve globals, record contents, object contents, or lifetimes. If a dependency invalidates storage needed later, normal completion under this contract requires that the later accesses nevertheless remain valid.

### Errors and partial progress

There is no explicit validation failure, error-status branch, rollback, or retry. Dependency return values are not inspected.

If a dependency does not return normally, subsequent operations are not guaranteed. Earlier effects are not visibly undone. The source does not define an alternative error return.

## Dependencies

| Opaque ID | Invocation condition and visible inputs |
|---|---|
| F-2523efe3304d | Once after object-address capture and before selector inspection; no explicit arguments |
| F-a061acf353ed | Negative selector with nonzero transition and gate, or nonnegative selector with zero pending; receives captured object, transition word, and zero |
| F-df71e640897e | After F-a061acf353ed on the qualifying negative path, or directly on the nonnegative nonzero-pending path; receives object field, all-ones word, pending word, one, and an external-storage value read during argument evaluation |
| F-00cf49e09ab2 | Once on each normally completing path, after branch work and before the external finalization read and tail writes; receives state pointer and forwarded argument |

On the qualifying negative path, F-a061acf353ed completes before the external argument for F-df71e640897e is read. Independent argument reads within the latter call have no established relative order.

External finalization storage is read only after F-00cf49e09ab2 returns. The dependency contracts do not establish preservation of either external-storage location.

## Acceptance examples

These are conditional acceptance criteria from static evidence, not executed tests. Assume valid storage, normal dependency returns, and no relevant mutation or aliasing except where stated.

1. **Negative selector, zero transition.** With selector `0x80000000` and transition zero, neither branch-specific dependency is invoked. The gate byte is not read, and a nonzero pending word is not directly cleared. Common finalization still occurs.

2. **Negative selector, closed gate.** With selector `0xFFFFFFFF`, transition `5`, and gate zero, both branch-specific calls are skipped. There is no direct object `0x14` write.

3. **Negative selector, open gate, zero pending.** With a negative selector, transition `5`, gate `2`, and pending zero, F-a061acf353ed receives transition `5`. F-df71e640897e subsequently receives pending zero, and pending is then explicitly cleared. Any nonzero gate byte suffices.

4. **External argument changes before its read.** On the preceding path, if external argument storage initially contains value A and F-a061acf353ed changes it to value B before returning, F-df71e640897e receives B, provided no further change occurs during argument evaluation. The initial A is not retained by this function.

5. **Nonnegative selector, zero pending and transition.** With selector zero, pending zero, and transition zero, F-a061acf353ed is invoked with transition zero. The gate byte is not consulted, and external argument storage is not explicitly read by this function.

6. **Nonnegative selector, nonzero pending.** With selector `0x7FFFFFFF`, pending `9`, and transition `6`, F-df71e640897e is invoked and F-a061acf353ed is not. Object offset `0x14` receives `6`, followed by the pending clear.

7. **Transition changes before the object write.** On the preceding path, if F-df71e640897e changes transition to `11`, object offset `0x14` receives `11`.

8. **Captured object remains the target.** If F-2523efe3304d replaces state offset `0x00`, subsequent direct object accesses still use the earlier captured address, assuming it remains valid.

9. **Finalization changes external storage.** If F-00cf49e09ab2 changes external finalization storage from `0x11223344` to `0xAABBCCDD`, the subsequent state `0x44` write stores `0xAABBCCDD`, assuming no intervening mutation.

10. **Finalization affects the result.** If the final transition read yields `0x000000B6` and the final object-mask read yields `0x0000006D`, the result is `0x00000024`. This condition includes any effects of the tail writes or aliasing.

11. **Pending becomes nonzero again.** If pending is explicitly cleared and F-00cf49e09ab2 then sets it to `7`, there is no later direct pending write. A final pending value of `7` is consistent with the visible behavior.

## Uncertainties

The following remain documented_only unknowns:

- Application meanings of the fields, external values, and returned intersection.
- Dependency behavior, failures, reentrancy, and ownership rules.
- Contents, application-level types, and mutability policies of the external storage.
- The independently declared storage width of the external fifth-argument value.
- Whether dependencies actually modify either external value on any particular execution.
- Whether the all-ones argument denotes a signed sentinel, unsigned limit, or another convention.
- Allocation sizes, alignment requirements, byte order, and aliasing restrictions.
- Concurrency, atomicity, and asynchronous mutation semantics.
- Which field combinations are valid in the surrounding application.

These unknowns do not obscure the established read timing: the fifth argument is read during its call’s argument evaluation, and the 32-bit finalization value is read after F-00cf49e09ab2 returns.

## Evidence

Static evidence consists of the supplied function body’s explicit field accesses, signed selector test, conditional calls, external-storage references, writes, and final bitwise expression, interpreted using the supplied opaque dependency mapping.

The visible accesses establish 32-bit record words and address representation, byte-sized gate and tail operations, and the ordered branch and finalization behavior above. Field values undergo no visible arithmetic beyond the final bitwise AND; address calculations use the stated byte offsets.

A transition-related purpose is inferred from the conditional calls and updates. Domain semantics are not established.

No runtime tests have been performed. Acceptance examples are conditional specifications, not observed results.
