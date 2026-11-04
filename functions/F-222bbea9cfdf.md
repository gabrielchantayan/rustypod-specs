<!-- Trusted provenance: F-222bbea9cfdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-222bbea9cfdf

## Purpose

F-222bbea9cfdf performs a gated action on a supplied object, decrements a 32-bit object field, and conditionally invokes an indirect dependency.

The gate is inspected after an unconditional dependency call. A zero gate enables a marker write, a selection-dependent action, and a three-argument dependency call. A nonzero gate skips that entire region. Both paths then decrement the field and inspect an optional pointer.

Two distinct program-data locations participate in calls:

- One supplies a value used as the gated call’s third argument.
- Another supplies the target of the final indirect invocation.

Their values are obtained at their respective use points, not established as entry-time snapshots. Earlier dependencies cannot be assumed to preserve either location.

An interpretation such as “perform an action once, then release a resource” is plausible but unproven. The evidence does not establish that the gate is permanent, the decrement field is a reference count, or any dependency acquires or releases a lock.

## Interface

### Inputs

| Input | Established use |
|---|---|
| Object address | Address-bearing 32-bit input used for object-relative accesses. It is also passed unchanged to F-938a7ce5ab19 on one branch. |
| Selection value | A 32-bit integer compared with zero. On the applicable branch, its full 32-bit value is stored in the object. |

Any nonzero selection value chooses the storage branch, including negative values. No narrower permitted range is established.

The supplied representation uses a 32-bit address model. Its use of integer address arithmetic does not establish portable behavior for address overflow, invalid addresses, or misalignment.

### Object-relative layout

Offsets are in bytes from the object address.

| Offset | Width | Established role |
|---|---:|---|
| +0x04 | 32-bit pointer field | Its address is passed to the initial dependency. Its pointer value is read later for the final checks. |
| +0x0C | 32 bits | Read, decremented by one, and written back. |
| +0x10 | 8 bits | Gate: zero enables the gated region; any nonzero value suppresses it. |
| +0x14 | 32 bits | Supplies the gated call’s first argument. |
| +0x18 | 32 bits | Supplies the gated call’s second argument. |
| +0x20 | 32 bits | Receives the selection value on the nonzero-selection branch of the gated region. |

The last listed field occupies offsets +0x20 through +0x23. No meaning is established for intervening or unlisted bytes.

When the pointer read from +0x04 is nonnull, the function reads a 32-bit word at its destination. That word is tested for nonzero; it is not established as a callable address.

### Program-data inputs

Two separate program-data references are visible:

| Reference | Use | Acquisition point |
|---|---|---|
| Gated-argument storage | Supplies the value passed as the gated call’s third argument. | During preparation of that call, after the selected action completes. |
| Indirect-target storage | Supplies the target value for the final indirect invocation. | After the decrement and successful pointer and pointed-word checks. |

The first reference supplies its stored **value**, not the address of its storage. Its concrete type, width, and meaning are not established.

The second reference supplies a callable target value. Its exact representation and invocation convention remain unresolved. It is distinct from both the gated-argument storage and the word reached through the object’s pointer.

### Output and preconditions

There is no explicit return value or status result. Normal completion returns control either without the final indirect invocation or after that invocation returns.

Normal progression requires:

- Every reached object access to be valid for its width and read/write direction.
- F-4a19e3742733 to accept the address of the +0x04 field.
- Any nonnull pointer examined at the end to designate a readable 32-bit word.
- The required program-data storage to be readable when used.
- Any invoked indirect target to support the invocation performed.
- Earlier dependencies to leave storage needed later valid.

The function does not explicitly check the object address for null. It provides no local recovery for invalid storage, arithmetic-address failures, or an unusable indirect target.

## Behavior

### 1. Initial dependency

The first represented external action is a call to F-4a19e3742733 with the address of the object’s +0x04 field.

This is an address-of-field argument, not a load of the pointer stored there.

The function reads the gate only after this dependency returns. Consequently, changes made by the dependency can affect the gate decision, subsequent object-field reads, the program-data argument, and the eventual indirect target. No preservation of object contents, program-data values, ownership, or storage lifetime is established.

This call occurs even if the gate was already nonzero on entry.

### 2. Gate decision and marker write

The function next reads the byte at +0x10.

If it is nonzero, the function skips all of the following:

- Writing one to the gate.
- Calling F-938a7ce5ab19.
- Storing the selection value at +0x20.
- Preparing and making the gated call to F-c90e5d01a577, including obtaining its third argument.

Execution then proceeds to the decrement.

If the gate is zero, the function writes the one-byte value one to +0x10 before performing either selection-dependent action.

This order establishes that the marker write precedes those actions. It does not establish atomicity, mutual exclusion, or lasting one-time execution. Dependencies may change the marker, and no synchronization contract is supplied.

### 3. Selection-dependent action

Within the gated region, exactly one of two actions is selected:

- With a zero selection value, F-938a7ce5ab19 receives the original object address.
- With a nonzero selection value, that value is written as 32 bits at +0x20. F-938a7ce5ab19 is not called.

The function itself does not clear or otherwise write +0x20 on the zero-selection path.

F-938a7ce5ab19 has an unresolved effect contract. It may affect fields read later or either program-data location. Continued safe execution requires later-used storage to remain valid.

The gate is not rechecked after this action. If F-938a7ce5ab19 changes the gate and returns normally, execution still proceeds to the gated call.

### 4. Gated call and current program-data argument

After the selected action completes, the function prepares a call to F-c90e5d01a577 with:

1. The 32-bit value read from +0x14.
2. The 32-bit value read from +0x18.
3. The value read from the gated-argument program-data storage.

All three argument acquisitions belong to preparation of this call, after the selected action. In particular, the third argument is neither the address of its program-data storage nor an established constant captured on entry.

Earlier dependencies cannot be assumed to preserve that storage. If F-4a19e3742733 or F-938a7ce5ab19 changes it, the later read obtains the value present when the argument is acquired.

The evidence does **not** establish a relative evaluation order among:

- The +0x14 field read.
- The +0x18 field read.
- The gated-argument program-data read.

Their argument positions are established; their mutual read order is not. This matters if storage can change during preparation or has additional access semantics not supplied here.

Likewise, the two object-field arguments reflect their values when read after the selected action, rather than guaranteed entry values.

F-c90e5d01a577 is called once on each normally progressing invocation that observed a zero gate after the initial dependency. Its result is not explicitly used.

### 5. Decrement

After either skipping or completing the gated region, the function reads the 32-bit field at +0x0C, subtracts one, and writes the 32-bit result back.

This operation is independent of:

- Whether the gate was zero.
- Which selection branch was taken.
- Whether the field was positive, zero, or negative.
- Whether the final pointer is null.
- Whether the resulting value is zero.

The input to the decrement is the value present at this point. A preceding dependency may have changed it.

The represented subtraction uses a signed integer expression. Ordinary decrements whose mathematical result fits in signed 32 bits are established. Behavior when decrementing the minimum signed 32-bit value is **documented_only**; wraparound is not guaranteed by the supplied evidence.

There is no explicit guard against decrementing zero or a negative value.

### 6. Pointer checks and late indirect-target acquisition

After the decrement write, the function reads the pointer at +0x04.

- If it is null, no pointed-to word is read and no final invocation occurs.
- If it is nonnull, the function reads its first 32-bit word.
- If that word is zero, no final invocation occurs.
- If that word is nonzero, the function obtains the target value from the separate indirect-target program-data storage and performs the represented indirect invocation.

The target acquisition is conditional and late: it occurs only after the decrement and successful nonnull-pointer/nonzero-word checks. The target value must not be treated as captured before earlier dependencies. If a preceding dependency changes the target-bearing storage, the final acquisition uses its then-current value.

Three sources must remain distinct:

1. The object field at +0x04 supplies the pointer.
2. The word reached through that pointer supplies a nonzero condition.
3. The indirect-target program-data storage supplies the invocation target.

The gated call’s third-argument storage is a fourth, separate source. The evidence establishes no equivalence between it and the target-bearing storage.

The checked pointer and word are not explicitly reread before the invocation. The decrement result is not part of the invocation condition.

No explicit arguments are represented at this invocation site. The complete calling convention, implicit inputs, actual target identity, and exact machine-level transfer behavior remain unresolved.

## Side effects

### Direct writes

The function’s direct writes occur in this order when their branches are reached:

1. One byte containing one at +0x10, only when the gate read was zero.
2. The 32-bit nonzero selection value at +0x20, only on that selection branch.
3. The decremented 32-bit value at +0x0C on every normally progressing path.

When F-c90e5d01a577 is called, its return precedes the decrement write.

The visible body does not directly write through the final pointer or to either program-data location.

### Dependency effects and ownership

Dependencies may introduce additional effects. Their contracts are not supplied, so the function does not establish that:

- The gate remains one at return.
- A stored selection value remains unchanged.
- The decrement field retains its entry value until subtraction.
- Either program-data location remains unchanged across calls.
- Object storage or pointed-to storage remains allocated.
- Any call acquires, transfers, preserves, or releases ownership.

No explicit allocation or deallocation appears in the represented body. This does not constrain allocation or deallocation inside dependencies.

### Errors and partial progress

There is no explicit error return, rollback, or local recovery path.

If a dependency fails to return normally, subsequent operations are not established to occur. Earlier writes may already have happened. For example, the marker is written before either gated dependency, while the decrement occurs only after those calls return.

An enabled final invocation occurs after the decrement has already been written. An invalid target therefore has no locally specified mechanism for undoing that write.

## Dependencies

| Dependency | Condition | Visible inputs | Unresolved aspects |
|---|---|---|---|
| F-4a19e3742733 | Unconditional first call | Address of object field +0x04 | Effects, ownership, synchronization, and termination behavior. |
| F-938a7ce5ab19 | Gate read is zero and selection value is zero | Original object address | Effects on object state, program data, lifetime, and termination. |
| F-c90e5d01a577 | Gate read is zero, after the selected action | Two 32-bit object-field values and the then-read program-data value | Argument semantics, third-argument width, effects, and termination. |

The final indirect dependency has no supplied opaque function ID. Its target comes from the late-read indirect-target storage. Its identity and complete contract remain **documented_only**; it is not equated with any listed dependency.

## Acceptance examples

These are conditional static acceptance criteria, not observed executions. Unless an example states otherwise, storage is valid, dependencies return normally, and dependencies do not change relevant values.

### Nonzero gate

Given a gate of 2, decrement field of 7, and null final pointer:

- F-4a19e3742733 is called.
- The gated region is skipped, including the third-argument program-data read.
- The gate remains 2.
- The decrement field becomes 6.
- No pointed-to word or indirect target is acquired.

Any nonzero gate has this branch behavior.

### Zero gate and zero selection

Given a zero gate, zero selection, decrement field of 1, and null final pointer:

- The initial dependency runs before the gate read.
- The marker is written to one.
- F-938a7ce5ab19 receives the object address.
- The three gated-call arguments are obtained after that dependency returns.
- F-c90e5d01a577 is called.
- The decrement field becomes zero.
- No indirect target is acquired.

The function itself does not write +0x20.

### Negative selection

Given a zero gate and selection value −3, the function writes one to the gate, stores the 32-bit representation of −3 at +0x20, skips F-938a7ce5ab19, and calls F-c90e5d01a577 before decrementing.

### Initial dependency changes the gate

If F-4a19e3742733 changes an entry-time zero gate to nonzero, the gated region is skipped. If it changes a nonzero gate to zero, the gated region runs. Entry state alone does not determine the branch.

### Selected dependency changes argument sources

Suppose F-938a7ce5ab19 changes both object argument fields and replaces the gated-argument program-data value before returning. F-c90e5d01a577 receives the subsequently read values from all three sources.

This criterion does not prescribe which of the three reads occurs first. The third argument is the stored value, not the storage address or its earlier contents.

### Earlier dependency replaces the indirect target

Suppose F-c90e5d01a577 replaces the indirect-target program-data value with another valid target and returns. If the subsequent pointer and word checks succeed, the final invocation uses the replacement target.

The gated call’s already-obtained third argument does not determine that target. If either final check fails, the target-bearing storage is not read for an invocation.

### Pointer and decrement independence

After the decrement, a nonnull pointer to a zero word suppresses the final invocation. A nonnull pointer to any nonzero 32-bit word enables target acquisition and invocation.

These conditions apply whether the decrement produced zero or another in-range value.

### Zero before subtraction

If +0x0C contains zero immediately before subtraction, it becomes −1 under ordinary signed 32-bit arithmetic. No error is explicitly reported. Decrementing the minimum signed 32-bit value has no guaranteed result under this contract.

## Uncertainties

The following remain **documented_only**:

- Object type, semantic field names, and permitted field ranges.
- Whether the gate denotes initialization, notification, completion, or another state.
- Whether the decremented field is a count.
- Signed-boundary overflow behavior.
- The gated third argument’s concrete width, type, and meaning.
- The indirect target’s identity, calling convention, implicit inputs, and precise transfer behavior.
- Dependency effects on object state, program data, ownership, and memory lifetime.
- Thread safety, reentrancy, atomicity, and externally concurrent mutations.
- Alignment requirements and behavior for invalid or overflowing addresses.
- Nonlocal failure and nonreturning-call behavior.

The possible “first-use action followed by release” interpretation is inferred intent only, not an acceptance requirement.

## Evidence

Established claims derive from the supplied static representation: access widths, offsets, branch conditions, direct writes, argument positions, and ordering across calls and later accesses.

The gated third argument is represented as a program-data value obtained during gated-call preparation. The final target is represented by a separate program-data reference used only on the successful final-check path. Neither is established as an entry-time constant, and no dependency preservation guarantee is supplied.

No relative evaluation order among the gated call’s three argument reads is established. Semantic naming, ownership interpretations, and unresolved indirect-invocation details are not promoted into requirements.

No runtime tests, smoke checks, or execution observations have been performed. The acceptance examples are conditional consequences of the static evidence.
