<!-- Trusted provenance: F-bbe5d09fc9f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbe5d09fc9f5

## Purpose

F-bbe5d09fc9f5 selects one of two register-update and polling behaviors according to whether its second input is zero. It sets a mode-specific bit in a shared control word, obtains an opaque value from a dependency, and polls a condition involving two shared words. Polling ends when the register condition no longer requests continuation or a second dependency returns a nonzero result.

Every observed terminating path returns the four-byte value 1. The return value does not distinguish a register-condition exit from an exit requested by the second dependency.

The register addresses and bit tests suggest a hardware-control or synchronization operation. That interpretation is inferred intent, not an established device-level contract. The supplied evidence establishes register-access behavior but does not identify the hardware, the meaning of the bits, or the units of the literal 100 passed to a dependency.

## Interface

### Inputs

| Input | Representation in supplied evidence | Established use |
|---|---|---|
| First input | Four-byte value of unspecified interpretation | Not read or forwarded by this function. |
| Second input | Integer, interpreted here under a 32-bit target model | Compared with zero once to select a branch. Its magnitude and sign otherwise have no effect. |

The second input has two behavioral classes:

- **Zero mode:** exactly zero.
- **Nonzero mode:** every other representable value, including negative values.

The first input has no established ownership, pointer, or validity requirement because this function does not use it. Any additional calling-convention requirement is outside the supplied evidence.

### Output

The function returns a four-byte value equal to 1 whenever it returns normally.

There is no demonstrated error return, timeout code, or completion-status result. Termination is not guaranteed solely by this function: it depends on register observations and dependency behavior.

### Shared locations

This contract uses the following descriptive labels:

| Label | Address | Observed operations |
|---|---|---|
| Status word | `0x38400014` | Read |
| Control word | `0x38400804` | Read and write |

The supplied representation treats these accesses as word operations, consistent with a 32-bit target model. Exact machine access widths, alignment requirements, and volatile or memory-ordering semantics are not independently established by instruction-level evidence.

Relevant masks are:

- Status mask `0x40`: bit 6.
- Control-condition mask `0x04`: bit 2.
- Zero-mode request mask `0x100`: bit 8.
- Nonzero-mode request mask `0x80`: bit 7.

### Preconditions

For the behavior below to complete without an access fault or external interruption:

- Both shared addresses must support the accesses described.
- Both dependencies must be callable under the applicable ABI.
- Dependency calls must return whenever execution reaches them.
- Any device-specific requirements for accessing the shared words must already be satisfied.

The evidence does not establish that this function initializes the relevant device, acquires a lock, enables access, or validates those preconditions.

## Behavior

### Common structure

The function selects its mode using the second input. It does not subsequently reselect the mode based on shared state or dependency results.

In either mode, the observable sequence begins with:

1. Read the control word.
2. Write back that observed word with the mode-specific request bit set.
3. Call dependency F-000000000001 with no explicit arguments.
4. Retain its four-byte result as an opaque value.
5. Perform at least one polling iteration, provided the preceding operations return normally.

The initial control-word update is a read followed by an OR-derived write. For the value observed by that read, all bits other than the selected request bit retain their values. This does not establish an atomic bit-set operation or protection against concurrent updates.

No explicit cleanup, request-bit clearing, or restoration occurs after polling.

### Zero mode

When the second input is zero, the initial update sets control bit 8 using mask `0x100`.

Each polling iteration has this ordered behavior:

1. Read the status word and retain whether bit 6 is set.
2. Read the status word again, regardless of the result of the first read.
3. If the **first** status read had bit 6 set, read the control word.
4. Continue evaluating the loop condition only if the first status read had bit 6 set and the subsequently read control word had bit 2 set.
5. If both conditions hold, call F-000000000002 with the retained opaque value and the integer literal 100.
6. Start another polling iteration only if that dependency returns zero.

Consequently, zero-mode polling ends on the first iteration for which any of the following holds:

- The first status read has bit 6 clear.
- The first status read has bit 6 set, but the control read has bit 2 clear.
- Both tested bits are set, but F-000000000002 returns nonzero.

The second status read does not change the branch decision. Its read still occurs and may be observable if the location has read side effects.

### Nonzero mode

When the second input is nonzero, the initial update sets control bit 7 using mask `0x80`.

Each polling iteration has this ordered behavior:

1. Read the status word and retain whether bit 6 is clear.
2. Read the status word again, regardless of the result of the first read.
3. If the **first** status read had bit 6 clear, read the control word.
4. Continue evaluating the loop condition only if the first status read had bit 6 clear and the subsequently read control word had bit 2 clear.
5. If both conditions hold, call F-000000000002 with the retained opaque value and the integer literal 100.
6. Start another polling iteration only if that dependency returns zero.

Consequently, nonzero-mode polling ends on the first iteration for which any of the following holds:

- The first status read has bit 6 set.
- The first status read has bit 6 clear, but the control read has bit 2 set.
- Both tested bits are clear, but F-000000000002 returns nonzero.

As in zero mode, a change visible only in the second status read does not alter that iteration’s decision.

### Polling and termination properties

The dependency result is tested only for equality to zero. No distinction is made among different nonzero results, including negative results under a signed interpretation.

The function itself maintains no visible iteration limit and performs no explicit delay, yield, or sleep. Any such behavior inside a dependency is unknown.

A polling sequence may continue indefinitely if:

- The relevant register predicate remains true on every iteration; and
- F-000000000002 returns zero on every call.

A nonreturning dependency also prevents normal completion.

The literal 100 is passed unchanged on every call to F-000000000002. There is no established basis for describing it as milliseconds, ticks, an iteration limit, or a guaranteed deadline.

### Width and arithmetic constraints

The explicit operations are bitwise OR, bit masking, zero comparisons, and passing a constant. There is no demonstrated addition, subtraction, multiplication, or pointer arithmetic.

Under the 32-bit interpretation:

- Every mask fits in the word.
- Setting bit 7 or bit 8 introduces no arithmetic-overflow issue.
- The second input is not narrowed or transformed before its zero/nonzero classification.
- The opaque four-byte dependency result is reused without explicit modification.

The signedness of the opaque value is not established. Its bit pattern, rather than a numeric interpretation, is the relevant forwarding contract.

## Side effects

### Direct shared-state effects

The function explicitly performs one control-word write before obtaining the opaque dependency value. That write requests the selected bit be set in the value observed by the preceding read.

The function does not explicitly:

- Write the status word.
- Clear either request bit.
- Clear the other mode’s request bit.
- Reset control bit 2.
- Undo the initial write when polling ends.
- Store a separate completion or error indication.

If both request bits were already set, the initial OR operation leaves both set in the computed write value.

### Read ordering

Each entered polling iteration contains two status reads. They are not interchangeable with one cached read under the behavior represented by the supplied evidence.

A control-condition read occurs only when the first status read satisfies the selected mode’s status predicate. F-000000000002 is called only when both the status predicate and the control predicate hold.

Thus, the following ordering is significant:

- The request write precedes F-000000000001.
- F-000000000001 precedes all polling reads.
- Both status reads precede the conditional control read.
- The conditional control read precedes any corresponding F-000000000002 call.
- Another iteration starts only after F-000000000002 returns zero.

### Concurrency and external mutation

No lock, atomic read-modify-write primitive, or memory barrier is explicit in the supplied representation.

If another actor modifies the control word between the initial read and write, the computed write need not preserve that actor’s intervening changes. Whether the underlying device provides special register semantics is unknown.

Neither dependency may be assumed to preserve either shared word. In particular, the initial request bit need not remain set after F-000000000001 returns. Subsequent polling uses fresh reads rather than relying on the initial control value.

### Ownership

No allocation, release, or explicit ownership transfer is visible.

The result of F-000000000001 is treated as opaque. The evidence does not establish whether it represents a timestamp, handle, pointer, or resource. Passing it to F-000000000002 does not by itself establish ownership transfer, retention, or release.

## Dependencies

The following opaque IDs are assigned locally for this contract. They distinguish the two external call targets; they are not verified provenance identifiers.

| Dependency | Observed interface | Established role |
|---|---|---|
| F-000000000001 | No explicit arguments; returns a four-byte value | Called once after the request write. Its result is retained for conditional later calls. |
| F-000000000002 | Receives the retained value and literal 100; returns an integer result | Called only while the mode-specific register predicate holds. Zero permits another iteration; nonzero ends polling. |

For F-000000000001, the evidence does not establish:

- The meaning or validity range of its result.
- Whether it reads or writes shared state.
- Whether it creates or transfers a resource.
- Whether it can fail or fail to return.

For F-000000000002, the evidence does not establish:

- Whether it measures elapsed time.
- Whether 100 denotes a duration or another threshold.
- Whether zero means success, pending, or another state.
- Whether a nonzero result denotes an error.
- Whether it modifies global state or memory represented by the opaque value.

These unknowns do not alter the caller’s observed zero/nonzero continuation rule.

## Acceptance examples

These are conditional contract examples derived from static evidence. They are not runtime test results. Each assumes valid shared-word accesses and normally returning dependency calls unless stated otherwise.

### Zero mode exits without calling the polling dependency

Given:

- The second input is zero.
- The initial control read yields `0x00000000`.
- The first status read of the first iteration has bit 6 clear.

Expected behavior:

- The initial computed control write is `0x00000100`.
- F-000000000001 is called once.
- Two status reads occur.
- No polling control read occurs.
- F-000000000002 is not called.
- The function returns 1.

### Zero mode repeats once

Given:

- The second input is zero.
- On each of two iterations, the first status read has bit 6 set and the control read has bit 2 set.
- F-000000000002 returns zero on its first call and nonzero on its second.

Expected behavior:

- Two polling iterations occur.
- Each iteration performs two status reads and one control read.
- Both dependency calls receive the same retained opaque value and literal 100.
- The function returns 1 after the second dependency call.

### Nonzero values select the same mode

Given either a positive or negative nonzero second input, and an initial control read of `0x00000100`:

- The initial computed write is `0x00000180`.
- Bit 8 is not explicitly cleared.
- If the first polling status read has bit 6 set, a second status read still occurs.
- No polling control read or F-000000000002 call occurs in that iteration.
- The function returns 1.

### Nonzero mode exits on control bit 2

Given:

- The second input is nonzero.
- The first polling status read has bit 6 clear.
- The subsequent control read has bit 2 set.

Expected behavior:

- Two status reads occur before the control read.
- F-000000000002 is not called.
- The function returns 1.

### Conflicting status reads use the first observation

Given zero mode, a first status read with bit 6 set, and a second status read with bit 6 clear:

- A control read still occurs.
- If that control read has bit 2 set, F-000000000002 is called.

Conversely, in zero mode, a first status read with bit 6 clear ends polling after the second status read even if the second read has bit 6 set.

### Persistent continuation does not imply a bounded return

Given nonzero mode, first status reads with bit 6 consistently clear, control reads with bit 2 consistently clear, and F-000000000002 consistently returning zero:

- Polling continues.
- No finite return bound follows from the literal 100 alone.

## Uncertainties

The following items are **documented_only**:

- Exact target ABI, integer widths beyond explicitly four-byte values, and physical register access widths.
- Device identity and the hardware meanings of the tested and set bits.
- Whether reads clear flags or have other device side effects.
- Whether the control word supports ordinary storage semantics, write-one semantics, or another register protocol.
- The meaning, units, ownership implications, and failure modes of both dependencies.
- Memory-ordering guarantees, interrupt interactions, and concurrent-caller safety.
- Whether the repeated status reads reflect distinct machine accesses in the underlying executable.
- Any guarantee of termination or successful device transition.

An elapsed-time interpretation for the dependencies is plausible but unproven. The contract therefore uses their observed argument and return behavior without assigning timing semantics.

## Evidence

The established behavior comes solely from the supplied static function representation:

- A zero/nonzero branch on the second input.
- One mode-specific control-word OR update.
- One unconditional call producing an opaque four-byte value.
- Two status-word reads per represented polling iteration.
- A conditional control-word read.
- A short-circuited dependency call whose zero result permits repetition.
- A constant four-byte return value of 1 on normal termination.

No dependency bodies, machine instructions, ABI documentation, device documentation, or runtime traces were supplied. No runtime tests have been performed. The acceptance examples express consequences of the static representation, not independently observed executions.
