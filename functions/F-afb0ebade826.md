<!-- Trusted provenance: F-afb0ebade826; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afb0ebade826

## Purpose

F-afb0ebade826 exposes a machine-state-dependent operation that writes a fixed sequence of nine 32-bit words, conditionally performs coprocessor operations, invokes external functionality, and either reads a 64-bit return value or reaches an invalid-instruction termination boundary.

**Established static evidence:** Its behavior depends on substantially more than the four explicit arguments. Incoming condition flags, additional register-held values, pointer-like values, and coprocessor state influence memory accesses and control flow.

**Inferred intent:** The nine-word output resembles a partial execution-state record. The evidence does not establish that it is a context save, exception frame, diagnostic record, or other particular structure. Consumers must not rely on any such interpretation.

**Evidence limit:** The supplied representation explicitly reports invalid instruction data and truncated control flow. This contract describes the represented behavior; it does not establish that the representation corresponds to a valid, conventionally callable routine.

## Interface

### Explicit inputs

The represented interface has four positional arguments:

| Input | Width | Observed use |
|---|---:|---|
| First argument | 32 bits | Written to the output record at byte offset 4. |
| Second argument | 32 bits | Written to the output record at byte offset 8. |
| Third argument | 32 bits | No explicit use is visible. |
| Fourth argument | 32 bits, represented as signed | Supplies the base for a conditional coprocessor load at displacement 76 bytes. |

The third argument’s lack of an explicit use does not prove that its underlying machine state is irrelevant to external calls.

### Additional incoming state

The represented operation also consumes:

- Zero, negative, overflow, and carry condition indicators.
- An initial 32-bit destination address.
- An incoming source pointer, also treated numerically as a 32-bit value.
- Incoming values for the first and second saved payload words.
- Another 32-bit payload value, also used as an address base on one terminal path.
- An incoming target pointer.
- A final 32-bit payload value.
- A separate address base used for the possible 64-bit result read.
- Coprocessor state or selectors used by the operations listed below.

For clarity, this contract calls the incoming source address the **source base**, and the address used for the nine output writes the **record base**. These are descriptive interface terms, not declared parameters.

### Output

The represented normal result is 64 bits. On the only visibly returning branch, the result is read from memory at a separate incoming address base plus 4 bytes.

The record writes do not themselves determine that result. Aliasing can connect them to the result memory, but no such relationship is required by the evidence.

### Preconditions

For a particular execution to complete the corresponding access:

- Every selected ordinary-memory read and write must address accessible storage.
- The four-word source area must be readable when overflow is clear.
- Each of the nine record destinations must be writable.
- The target pointer must be writable when the later negative-conditioned store executes.
- The result location must support an eight-byte read when the return branch is reached.
- Selected coprocessor operations must be valid in the active execution environment.

No explicit bounds checks, null checks, alignment checks, or recoverable error returns are visible. Required alignment, memory endianness, privilege level, and fault handling are not established.

## Behavior

### 1. Select the record base

When the incoming zero condition is set, the record base becomes the 32-bit numeric source address XORed with `0x00001f00`.

When zero is clear, the incoming destination address remains the record base.

This is a bitwise transformation, not addition or subtraction. No allocation, capacity check, or initialization of the selected destination is visible.

### 2. Select the values to be recorded

The pointer-valued payload initially equals the source base.

When overflow is clear, four consecutive 32-bit words are read from the source area:

| Source byte offset | Selected use |
|---:|---|
| 0 | First saved payload word |
| 4 | Second saved payload word |
| 8 | Pointer-valued payload |
| 12 | Target pointer |

These reads appear before all nine record writes in the supplied representation.

When overflow is set, these four source reads do not occur. The first saved payload, second saved payload, and target pointer retain their incoming values. The pointer-valued payload remains the source base itself.

The source area is not shown being modified during this selection step.

### 3. Write the record

Nine 32-bit writes appear in the following order:

| Record byte offset | Value written |
|---:|---|
| 4 | First explicit argument |
| 8 | Second explicit argument |
| 12 | Selected first saved payload |
| 16 | Additional incoming payload/address-base value |
| 20 | Selected second saved payload |
| 24 | Selected pointer-valued payload |
| 28 | Selected target pointer |
| 32 | Final incoming payload value |
| 36 | Literal `0x08a049c8` |

The write footprint is 36 bytes spanning offsets 4 through 39 inclusive. No write to offsets 0 through 3 is visible.

The literal at offset 36 is established as a stored value. Its meaning as a continuation address, code location, or identifier is not established.

These writes occur before the first represented invalid-instruction termination check. A path that cannot return can therefore still modify the record.

### 4. Perform initial coprocessor operations

After the record writes:

1. If overflow is clear, a coprocessor load is represented using coprocessor number 11, selector 10, and the fourth explicit argument plus 76 bytes.
2. If the negative and overflow indicators are equal, a coprocessor function is represented with coprocessor number 7, operation selectors 13 and 3, and three additional coprocessor operands identified by selectors 12, 3, and 9.

The exact architectural interpretation of these operands is not available. In particular, the evidence does not establish the transfer size or ordinary-memory footprint of the coprocessor load.

### 5. Apply the first termination condition

If zero is set, or negative differs from overflow, the represented execution reaches an invalid-instruction boundary.

This is not a normal return and does not expose an error-code convention. Whether the actual environment traps, faults, stops execution, or transfers control elsewhere is unknown.

Consequently, continuation beyond this boundary requires, in the represented predicate model:

- Zero clear; and
- Negative equal to overflow.

The equality-conditioned coprocessor function precedes this check. It can therefore occur even when zero subsequently forces termination.

### 6. Execute the continuing path

For executions continuing past the first boundary:

1. If negative is set, a coprocessor load is represented using coprocessor number 9, selector 11, and address literal `0x08a049dc`.
2. F-3daf2e987f4c is invoked.
3. Under the represented condition “zero clear and negative equal to overflow,” an unresolved external callable is invoked.
4. If negative is set, the selected first saved payload is written as one 32-bit word through the selected target pointer.
5. Otherwise, a coprocessor function is represented using coprocessor number 8, operation selectors 5 and 7, and operands identified by selectors 15, 13, and 1.

The condition for the unresolved invocation matches the condition required to pass the earlier boundary. In the supplied representation, that invocation is therefore selected on every continuing path. Its reachability still depends on preceding operations and F-3daf2e987f4c returning normally.

No claim is made that either external call preserves globals, memory, coprocessor state, or machine flags.

### 7. Return or reach a later invalid-instruction boundary

The final represented decision is:

- If overflow is clear and carry is clear, read eight bytes at the result-address base plus 4 and return that 64-bit value.
- If overflow is clear and carry is set:
  - Perform a coprocessor store when negative is set, using coprocessor number 2, selector 11, and the additional payload/address-base value minus 672 bytes.
  - Reach an invalid-instruction boundary.
- If overflow is set, reach an invalid-instruction boundary.

Under a consistent interpretation of the condition indicators throughout this representation, the normal-return combination is:

- Zero clear;
- Negative clear;
- Overflow clear;
- Carry clear.

Under that same interpretation, the late coprocessor store is unreachable: passing the first boundary with overflow clear requires negative clear, whereas that store requires negative set. Its presence must be retained as evidence of the represented structure, not promoted into a proven executable path.

## Side effects

### Ordinary memory

Visible ordinary-memory effects include:

- Four conditional 32-bit source reads.
- Nine ordered 32-bit record writes.
- One conditional later 32-bit target write.
- One conditional final 64-bit result read.

No disjointness requirement is established between the source area, record, target, and result area.

The initial four source values are selected before the record writes. Thus, if source and record overlap, the represented record uses the values already read rather than rereading source words between record stores.

The later target write can overwrite the record, source area, result area, or unrelated memory if the selected pointer aliases those locations. External calls can also alter memory between the record writes and the later accesses.

These ordering statements describe the supplied representation. They do not establish hardware atomicity, barriers, concurrency safety, or architectural ordering guarantees.

### Coprocessor and external effects

Coprocessor operations may have state or memory effects not expressible from the supplied operands alone. External calls may introduce additional reads, writes, exceptions, or non-returning behavior.

### Ownership

The visible operation does not explicitly allocate or release memory, and it does not establish an ownership-transfer contract.

Pointers are used as addresses or recorded as 32-bit values. That use does not prove retained ownership, borrowed lifetime, or permission to release the referenced storage. Ownership effects of dependencies remain unknown.

### Partial completion and errors

There is no visible rollback. A later fault, external failure, or invalid-instruction boundary can occur after earlier record writes have taken effect.

No structured status result, exception mapping, or recovery protocol is established.

## Dependencies

### F-3daf2e987f4c

This dependency is invoked after the first invalid-instruction boundary has been passed and after the negative-conditioned coprocessor load, if selected.

No explicit argument list or consumed return value is visible. Its actual machine-state inputs, memory effects, ownership effects, flag behavior, and return guarantees are unknown.

### Unresolved external callable

A second invocation follows F-3daf2e987f4c under the represented continuation predicate. No opaque dependency ID is provided for this target.

Its arguments, return value, memory effects, ownership behavior, and execution guarantees are documented_only. No functional identity is inferred.

### Execution environment

Coprocessor facilities and invalid-instruction handling are necessary environmental dependencies. Their architecture-specific behavior is not supplied.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results. They assume required accesses and coprocessor operations succeed, external calls return when reached, and the represented condition indicators retain a consistent meaning.

### Normal return with source-loaded payloads

Given zero clear, negative clear, overflow clear, and carry clear:

- Read the four source words before writing the record.
- Preserve the incoming record base.
- Write all nine record words at offsets 4 through 36.
- Select the coprocessor load based on the fourth argument plus 76.
- Select the equality-conditioned coprocessor function.
- Pass the first termination check.
- Invoke F-3daf2e987f4c and then the unresolved callable.
- Select the non-negative coprocessor function rather than the target-pointer write.
- Return the eight-byte value read from the result-address base plus 4.

The returned value is the value at the time of that final read, including any intervening memory changes.

### Zero-triggered termination after record creation

Given zero set, negative clear, and overflow clear:

- Derive the record base by XORing the source address with `0x00001f00`.
- Read the four source words and write the record.
- Select both initial coprocessor operations.
- Reach the first invalid-instruction boundary.
- Do not reach either external invocation or the final result read.

An implementation that skips the record writes merely because this path cannot return would not match the represented behavior.

### Negative/overflow mismatch

Given zero clear, negative set, and overflow clear:

- Load the four source words and write the record.
- Select the initial overflow-clear coprocessor load.
- Do not select the equality-conditioned coprocessor function.
- Reach the first invalid-instruction boundary.
- Do not reach the later negative-conditioned coprocessor load or target write.

### Continuing overflow-set path

Given zero clear, negative set, and overflow set:

- Do not read the four-word source area.
- Record incoming payload values, the source base as pointer-valued payload, and the incoming target pointer.
- Skip the initial overflow-clear coprocessor load.
- Select the equality-conditioned coprocessor function and pass the first boundary.
- Select the later negative-conditioned coprocessor load.
- Invoke both external callables, subject to normal return.
- Write the selected first saved payload through the target pointer.
- Reach the final invalid-instruction boundary without returning a 64-bit value.

### Carry-triggered final termination

Given zero clear, negative clear, overflow clear, and carry set:

- Follow the normal continuing path through the external invocations and non-negative coprocessor function.
- Do not read the final 64-bit result.
- Do not select the negative-conditioned coprocessor store.
- Reach the later invalid-instruction boundary.

## Uncertainties

The following are documented_only:

- Whether this entry is a valid function boundary or executable instruction sequence.
- The true calling convention and availability of the additional incoming state.
- Whether represented condition indicators correspond to stable entry values or imperfectly recovered machine-state dependencies.
- The effects of external calls on globals, memory, flags, ownership, and control flow.
- Coprocessor instruction validity, transfer widths, privilege requirements, and exceptions.
- Endianness, alignment requirements, and address-space interpretation.
- Arithmetic behavior at address boundaries. The fourth-argument addition and late base subtraction do not establish a portable overflow rule.
- The meaning of the stored address-like literal.
- Runtime behavior at the invalid-instruction boundaries.

The representation uses 32-bit address-bearing values and an eight-byte return read. A replacement must not silently widen record fields or substitute a 32-bit result read. Exact address-wrap semantics require additional architectural evidence.

## Evidence

This contract is based solely on the supplied static representation, including its invalid-instruction warning and truncated control-flow boundaries.

Established observations include the explicit argument widths, conditional source reads, nine-word record layout, represented ordering, branch predicates, external invocations, coprocessor operation operands, and conditional 64-bit result read.

The partial-state-record interpretation is inferred intent only. No runtime tests, instruction-level validation, dependency inspection, or independent execution observations have been performed.
