<!-- Trusted provenance: F-362dfede5dca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-362dfede5dca

## Purpose

F-362dfede5dca conditionally performs a guarded initialization sequence, ensures that a separate status byte is nonzero on ordinary completion, and returns a 32-bit value from global storage.

The established behavior has two distinct conditions:

- A bit in a 32-bit guard word determines whether an acquisition dependency is consulted.
- A byte at the beginning of the captured state region is changed from zero to one after the optional initialization sequence.

These conditions are not interchangeable. A nonzero status byte does not prevent initialization, and an unsuccessful guard acquisition does not prevent the status byte from being set.

**Inferred intent:** the guard operations and the three-argument dependency resemble a one-time initialization and registration protocol. That interpretation is not necessary to specify the observable behavior. The evidence does not establish thread safety, registration semantics, object ownership, or a successful-initialization guarantee.

## Interface

### Inputs

The function has no explicit parameters. Its inputs are held in global storage:

| Contract name | Role |
|---|---|
| State reference | A pointer to a region containing the status byte and guard word. |
| Result slot | A 32-bit value used as an argument to F-180384254a70 and read again for the function’s final return. |
| Registration operand A | A globally stored value supplied as the second argument to F-804adc29bbfb. |
| Registration operand B | A globally stored value supplied as the third argument to F-804adc29bbfb. |

These names are descriptive contract labels, not source-level identifiers.

The function captures the state reference at entry. Subsequent status-byte access and the final guard-operation address use that captured pointer. The initial guard test and the acquisition argument are expressed through the global state reference rather than exclusively through the captured pointer.

### State layout

Relative to the state reference:

| Byte offset | Access |
|---|---|
| 0 | One-byte status read; a one-byte write occurs only when the value read is zero. |
| 1–3 | No direct access established. |
| 4–7 | A 32-bit unsigned guard-word read beginning at offset 4. The address at offset 4 is also passed to guard dependencies. |

Only bit 0 of the directly read guard word affects the branch. Other bits do not directly control this function.

The function requires memory suitable for the accesses it actually performs. In particular:

- The captured state pointer must support a status-byte read and, when that byte is zero, a status-byte write.
- The state pointer used for the initial guard read must support a 32-bit read at offset 4.
- Each address passed to a dependency must satisfy that dependency’s requirements.

No null check, bounds check, or alignment correction is present. Exact alignment requirements, pointer width, byte order, and address-overflow behavior are not established by the supplied evidence.

### Output

The function returns a raw 32-bit value read from the result slot at the end of ordinary execution.

That returned value is not necessarily:

- The value previously supplied to F-180384254a70.
- The value returned by F-180384254a70.
- A pointer, status code, or Boolean.

Those interpretations require information not present here.

## Behavior

### Initial state capture and guard decision

The function first saves the current state reference for later use.

It then reads the 32-bit guard word at byte offset 4 through the global state reference and tests its least significant bit.

If bit 0 is set:

- F-b255b26284e6 is not called.
- The optional initialization sequence is skipped.
- Execution proceeds to the status-byte handling.

If bit 0 is clear:

- F-b255b26284e6 is called with an address at byte offset 4 from the state reference evaluated for that call.
- A zero return skips the optional initialization sequence.
- Any nonzero return enters that sequence.

The branch depends only on zero versus nonzero. There is no positive-only requirement; a negative signed result is also nonzero.

This is short-circuit behavior: the acquisition dependency is not consulted when the initial guard bit is already set.

### Optional initialization sequence

When the initial guard bit is clear and F-b255b26284e6 returns nonzero, the following calls occur in order, provided each preceding call returns normally:

1. F-180384254a70 receives the then-read value of the result slot.
2. F-804adc29bbfb receives three arguments:
   - The 32-bit return value from F-180384254a70.
   - Registration operand A.
   - Registration operand B.
3. F-d08d3ca17b81 receives the address at byte offset 4 from the state pointer captured at entry.

The return value of F-180384254a70 is carried into the first argument of F-804adc29bbfb. It is not explicitly stored in the result slot.

No return value from F-804adc29bbfb or F-d08d3ca17b81 is used to choose subsequent behavior. If either dependency reports a failure solely through an ordinary return value, this function has no visible branch that handles it.

The registration operands are obtained for the call after F-180384254a70 returns. Consequently, changes that F-180384254a70 makes to those global operands can affect the arguments supplied to F-804adc29bbfb. The relative evaluation order of the two operand reads is not established.

The final guard-operation address is based on the entry-captured state pointer. It is not recomputed from whatever state reference remains in global storage after the earlier calls.

### Status-byte handling

After the guard decision and any normally completed initialization sequence, the function reads the byte at offset 0 through the entry-captured state pointer.

- If the byte is zero, the function writes the byte value one.
- If the byte is nonzero, the function leaves it unchanged.

This is not normalization of every nonzero value to one. For example, a byte value of 2 remains 2.

The status-byte operation occurs after F-d08d3ca17b81 on the successful acquisition path. It occurs without any initialization calls on the already-set-guard path and after the acquisition call alone on the failed-acquisition path.

The byte tested is its value at this later read. A dependency that changes it before that point can therefore affect whether the write occurs.

### Final return read

After status-byte handling, the function reads and returns the result slot.

This final read is distinct from the earlier result-slot read used as the argument to F-180384254a70. Dependencies are not assumed to preserve that slot. Thus, the input to F-180384254a70 and the final returned value can differ.

The final read also follows the possible status-byte write. No non-aliasing guarantee is established for the global storage and pointed-to state, so a contract must not assume that every earlier write is independent of the final result value.

### Pointer changes and reentrancy

The captured state pointer remains the target for status-byte handling and the final guard operation even if a dependency changes the global state reference.

The acquisition address, however, comes from a separate evaluation of the global state reference. Under stable global state, it matches the final guard-operation address. If an external execution context can change that reference between its earlier evaluations, the addresses need not match.

The supplied evidence does not define synchronization or a language-level concurrency model. This address distinction describes the reference usage; it does not establish that concurrent mutation is safe or supported.

No explicit reentrancy protection is visible beyond whatever the guard dependencies implement.

## Side effects

### Direct effects

The only explicit state write by this function is a one-byte store of one at offset 0 of the captured state region, conditional on reading zero there.

The function does not explicitly:

- Write the guard word.
- Update the global state reference.
- Update the result slot.
- Allocate or free memory.
- Clear the status byte.
- Restore global values changed by dependencies.

Those effects can nevertheless occur inside dependencies or through aliasing. They must not be ruled out without additional evidence.

### Indirect effects and ownership

All four dependencies may have side effects not described here. Their behavior is not established merely by the arguments passed to them.

No ownership transfer, reference-count change, allocation lifetime, destruction responsibility, or memory-release guarantee is proven. In particular, passing the result of F-180384254a70 to F-804adc29bbfb does not establish who owns that value or any resource it may designate.

### Errors and interrupted execution

The function has no explicit error return path of its own. A zero acquisition result is a skip condition, not an early return and not a reason to omit status-byte handling.

There is no visible cleanup path for abnormal completion of a dependency. If a dependency does not return normally, the evidence does not guarantee:

- Execution of later dependencies.
- The status-byte read or write.
- The final result-slot read.
- Release or completion of a previously acquired guard.

Invalid memory access, dependency exceptions, process termination, and other abnormal transfers are not assigned additional behavior by this contract.

## Dependencies

| Opaque ID | Invocation condition | Established arguments and result use |
|---|---|---|
| F-b255b26284e6 | Initial guard bit is clear. | Receives the guard address obtained through the global state reference. Zero skips initialization; nonzero permits it. |
| F-180384254a70 | Acquisition returned nonzero. | Receives the current 32-bit result-slot value. Its 32-bit return becomes the first argument of the next dependency. |
| F-804adc29bbfb | F-180384254a70 returned normally. | Receives that return value, registration operand A, and registration operand B, in that argument order. Its return does not control this function. |
| F-d08d3ca17b81 | F-804adc29bbfb returned normally. | Receives the guard address derived from the entry-captured state pointer. Its return does not control this function. |

Names such as “acquisition” and “guard operation” describe positions in the observed protocol. They do not substitute for unavailable dependency contracts.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, stable global references, no relevant aliasing, dependencies that return normally, and no unmentioned dependency mutations.

### Guard already set; status clear

Given a guard word whose bit 0 is one and a status byte of zero:

- No dependency is called.
- The status byte becomes one.
- The function returns the result-slot value read after that write.

A guard word with additional bits set has the same branch outcome.

### Guard clear; acquisition refused

Given a guard word whose bit 0 is zero, a status byte of zero, and an acquisition result of zero:

- Only F-b255b26284e6 is called.
- The status byte still becomes one.
- The function returns the final result-slot value.

This case prevents treating the status byte as proof that initialization occurred.

### Guard clear; acquisition permitted

Given a clear guard bit and any nonzero acquisition result:

- All four dependencies are called in the order listed above.
- The output of F-180384254a70 is the first argument to F-804adc29bbfb.
- Status-byte handling follows the final guard dependency.
- The function returns the result slot, not the saved output of F-180384254a70.

### Noncanonical status value

Given a status byte of 255 when the status check occurs:

- The function performs no direct status-byte write.
- The byte is not replaced with one.

The guard branch remains independent of this byte value.

### Dependency changes the result slot

Suppose F-180384254a70 receives value R, a dependency subsequently changes the result slot to S, and nothing later changes it:

- The initialization call’s input remains R.
- The final function return is S.

### Dependency changes the state reference

Suppose the acquisition succeeds for the entry state region and F-180384254a70 changes the global state reference to another region:

- The final guard operation still receives the entry region’s address at offset 4.
- Status-byte handling still targets the entry region.
- The function does not switch those operations to the newly referenced region.

### Dependency changes the status byte

Suppose an initialization dependency changes the captured region’s status byte from zero to 7 before the final status check:

- The later check observes 7.
- The function does not overwrite it with one.

## Uncertainties

The following are **documented_only**:

- The semantic meaning and representation of the returned 32-bit value.
- The storage widths and semantic types of the two registration operands.
- The pointer width, byte order, alignment rules, and full state-region layout.
- Whether the guard protocol provides exclusion, waiting, recursion detection, memory ordering, or persistent completion state.
- Whether any dependency modifies globals, releases memory, retains arguments, or transfers ownership.
- Whether the status byte represents readiness, use, registration, or another condition.
- Whether global slots can alias each other or the state region.
- The behavior of repeated or concurrent invocations once dependency effects are included.
- The failure mechanisms and abnormal-control-flow guarantees of each dependency.

In particular, neither “initialization occurs exactly once” nor “a nonzero status byte means initialization succeeded” is established.

## Evidence

The supplied static representation establishes the no-argument interface, 32-bit return, entry pointer capture, byte offsets, unsigned 32-bit guard read, bit-zero test, zero/nonzero acquisition branch, dependency argument relationships, conditional one-byte write, and final global result read.

The one-time-initialization interpretation is inferred intent. Dependency semantics and platform properties beyond the represented accesses remain unknown.

No runtime tests, dependency inspection, or independent implementation verification have been performed.
