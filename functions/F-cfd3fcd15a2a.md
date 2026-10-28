<!-- Trusted provenance: F-cfd3fcd15a2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfd3fcd15a2a

## Purpose

F-cfd3fcd15a2a is represented as a low-level operation that updates two global storage locations, performs a calculated 16-bit memory write, and then reaches a boundary beyond which execution is not reliably described.

The supplied static representation supports an ordered sequence of three writes. It does not establish a complete executable routine, a successful return path, or a defined terminal action. In particular, the terminal diagnostic must not be interpreted as proof that the target deliberately halts, raises a specific exception, or invokes an ordinary callable dependency.

Names associated with the first two storage locations suggest data-abort and reset-related roles. Those roles are inferred intent, not established operational semantics. This contract therefore describes storage updates rather than claiming that the routine clears an architectural exception or requests a system reset.

No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has three explicit arguments, each occupying 32 bits.

| Input | Represented use |
|---|---|
| First argument | No use is visible in the available fragment. |
| Second argument | No use is visible in the available fragment. |
| Third argument | Supplies the value assigned to the second global storage location. |

The first two arguments have no visible influence on the represented writes. This is a statement about the available fragment, not proof that they are irrelevant to the actual executable or any unavailable continuation.

The third argument is represented as an uninterpreted 32-bit quantity. Nothing establishes that it is a pointer, an enumeration, a Boolean value, or an owned resource.

### Additional incoming state

The calculated memory write depends on an unsigned 32-bit value present in an incoming register that is not modeled as one of the three explicit arguments.

This value is a necessary additional input to the visible behavior. The available representation does not establish:

- How the caller supplies it.
- Whether a calling convention requires it to be initialized.
- Whether it has a defined value at every entry.
- Whether it represents an address, packed fields, or unrelated machine state.

For the arithmetic descriptions below, let \(U\) denote this unsigned 32-bit incoming value. This notation describes the contract; it does not identify a source variable.

### Outputs

The represented interface has no return value. Its visible outputs are memory effects.

There is no established normal-return behavior. Reaching the end of the reliable fragment must not be treated as successful completion.

### Preconditions for observing all three writes

To observe the complete represented write sequence:

- Both global destinations must permit their respective accesses.
- The calculated destination must permit a 16-bit write.
- Execution must reach each operation without an intervening exception, reset, protection failure, or other transfer of control.
- The supplied representation must correspond to executable behavior through those operations.

These are conditions for the described observation, not evidence that the function validates them. No validation is visible.

The exact calling convention, global storage widths, memory map, and fault-handling environment are unspecified.

## Behavior

### 1. Update the first global storage location

The first represented operation assigns numeric zero to a global location associated by name with data-abort state.

This operation precedes both the second global assignment and the calculated 16-bit store in the supplied representation.

The destination’s declared width is not supplied. Consequently, the evidence does not establish:

- The number of bytes written.
- The access width used by executable instructions.
- Whether the write covers an entire independently meaningful object.
- Which overlapping storage views, if any, observe a change.

The assigned value is zero, but a byte-for-byte description of the affected region requires the missing storage layout.

### 2. Update the second global storage location

The next represented operation assigns the third explicit argument to a global location associated by name with reset state.

The input has a width of 32 bits. The destination’s width is not established. If it is a 32-bit scalar with ordinary assignment behavior, the resulting stored value is the third argument’s 32-bit bit pattern. If its width differs, the exact conversion and affected byte range require additional type and layout evidence.

There is no visible conditional branch between the two global assignments. In the supplied representation, the second assignment is attempted after the first, provided execution continues.

The available information does not establish that the two global destinations are disjoint. They must not be modeled as independent state fields without layout evidence.

### 3. Calculate the destination for the 16-bit write

The destination is formed from the upper ten bits of \(U\):

\[
A = \left\lfloor \frac{U}{2^{22}} \right\rfloor \times 256.
\]

The shift is unsigned and logical. Its result is between 0 and 1023 inclusive. Multiplication by 256 therefore produces an address value between 0 and 261,888 inclusive, or between hexadecimal `0x00000000` and `0x0003FF00`.

Every calculated destination is a multiple of 256. Thus every destination is also aligned to a two-byte boundary.

There is no overflow in this calculation within 32-bit unsigned arithmetic: the maximum result is well below \(2^{32}\).

The address calculation uses no supplied base address. It is represented as a direct conversion of the calculated integer to a memory destination. Whether those numeric addresses identify usable memory, devices, aliases, or inaccessible regions depends on the execution environment.

In particular, a zero destination is possible and is not rejected by any visible check.

### 4. Write the low 16 bits of the incoming value

The value supplied to the calculated store is:

\[
V = U \bmod 2^{16}.
\]

The represented operation is a 16-bit store of that bit pattern at address \(A\).

Although the intermediate target type is represented as a signed short, this contract describes the low-16-bit store shown by the fragment. A pattern whose top bit is set must not be replaced by its absolute value, clamped to a signed range, or expanded into a wider store.

The destination and payload use different portions of \(U\):

- Bits 31 through 22 select the destination.
- Bits 15 through 0 supply the stored pattern.
- Bits 21 through 16 do not affect either the calculated destination or the stored pattern in the visible fragment.

No bounds check, permission check, null-address check, or destination read is visible before this store.

The target’s byte order is not established. The contract specifies the 16-bit pattern and destination address, not which of the two individual byte addresses receives its most significant byte.

### 5. Reach the unsupported continuation boundary

After the calculated store, the supplied representation ends at an invalid-instruction diagnostic.

The evidence does not establish what the actual processor does next. Possible outcomes must not be selected without additional evidence. A normal return, deliberate halt, architectural trap, execution of differently decoded instructions, and continuation into unavailable code are all outside the established contract.

The diagnostic is not an additional proven memory operation or an ordinary dependency call.

## Side effects

### Represented order

The supported operation order is:

1. Assign zero to the first global destination.
2. Assign the third argument to the second global destination.
3. Store the low 16 bits of the additional incoming value at the calculated address.
4. Reach the boundary of reliable static description.

This is the order shown by the supplied representation. Exact machine-level access decomposition, bus visibility, compiler ordering guarantees, and inter-thread synchronization semantics are not established.

### Aliasing and final state

The write sequence does not imply three independent final values.

The global destinations may overlap other storage views. Their relationship to one another and to the calculated destination is unknown. If destinations overlap, a later write can alter bytes affected by an earlier one.

Accordingly, a consumer must not conclude that the first global destination still contains zero after the entire sequence without establishing non-overlap or otherwise analyzing the relevant layout.

### Partial execution and errors

No error result, recovery branch, rollback, retry, or cleanup action is visible.

If a write faults, earlier completed writes may remain observable. There is no evidence of transactional behavior.

The handling of access faults is external to this fragment. In particular, a fault at the calculated address does not establish that either preceding global update is undone.

### Ownership and resources

No allocation, deallocation, resource acquisition, or ownership transfer is established.

The calculated write does not demonstrate that the function owns the target memory. Likewise, writing the third argument into global storage does not demonstrate ownership of any object that its bit pattern might identify.

No preservation guarantee is established for machine state or memory beyond the explicitly described effects.

## Dependencies

No ordinary callable dependency with an established opaque function ID appears in the reliable portion of the supplied representation.

The terminal invalid-instruction notation is a diagnostic boundary, not a proven callable dependency.

The behavior nevertheless relies on external conditions:

- The calling convention and incoming register state.
- The layout and accessibility of the global destinations.
- The address interpretation and permissions of the target memory map.
- The target’s byte order.
- Fault behavior and any handlers.
- The actual instruction stream beyond the reliable fragment.

These are environmental prerequisites or unknowns, not identified functions. No assumptions about their memory preservation, ownership behavior, or recovery behavior are made.

## Acceptance examples

These examples are conditional static acceptance criteria, not recorded executions. Unless stated otherwise, they assume that the represented operations execute, the accesses are permitted, and no overlapping destination changes the observation being checked.

### Zero incoming value

For \(U =\) `0x00000000`:

- The calculated address is `0x00000000`.
- The stored 16-bit pattern is `0x0000`.
- The two global assignments precede the calculated store.

This case must not be interpreted as a skipped write merely because the destination is zero. Whether that access succeeds is environment-dependent.

### Destination selection boundary

For \(U =\) `0x003FFFFF`:

- The calculated address is `0x00000000`.
- The stored pattern is `0xFFFF`.

For \(U =\) `0x00400000`:

- The calculated address is `0x00000100`.
- The stored pattern is `0x0000`.

These values distinguish the 22-bit shift from a byte extraction or a shift by another amount.

### Maximum incoming value

For \(U =\) `0xFFFFFFFF`:

- The calculated address is `0x0003FF00`.
- The stored pattern is `0xFFFF`.
- The address calculation does not wrap.

The represented operation is a 16-bit store, not a store of the full 32-bit incoming value.

### Irrelevant middle bits

The values `0x00801234` and `0x00BF1234` both select address `0x00000200` and store pattern `0x1234`.

Their differences are confined to bits that do not participate in either visible calculation. Under identical relevant memory conditions, the calculated stores therefore match.

### Third-argument handling

If the third argument is `0x89ABCDEF`, the second global assignment uses that value independently of the calculated store’s payload.

An exact 32-bit final-value expectation for that global is valid only if its destination is established as 32 bits and no later overlapping write changes it.

### Failure at the calculated destination

If both global assignments complete and the calculated store then faults, acceptance requires preserving the possibility that both earlier updates remain visible.

Returning an ordinary failure status or rolling back those updates is not established behavior.

## Uncertainties

The following are **documented_only**:

- The executable correctness of the supplied reconstruction.
- The machine instruction sequence underlying the represented operations.
- The global destinations’ widths, addresses, and overlap relationships.
- The byte order and address-space mapping.
- The origin and calling-convention status of the additional incoming value.
- Architectural fault behavior and any subsequent handler effects.
- Behavior beyond the invalid-instruction boundary.
- Whether reset-associated or data-abort-associated storage has hardware-visible effects.
- Any actual runtime outcome, including successful completion.

The reset and data-abort interpretations are inferred from naming only. Neither establishes a hardware action.

## Evidence

The supplied static fragment directly supports three ordered assignment expressions: a zero assignment to global storage, an assignment from the third 32-bit argument to another global destination, and a calculated 16-bit store using an additional unsigned 32-bit incoming value.

The address range, 256-byte alignment, payload truncation, and unused middle-bit range follow from that represented arithmetic.

The supplied annotations report problematic instruction decoding and overlapping global symbols. These are reasons to limit confidence, not independent proof of runtime behavior or exact storage layout.

No executable inspection, runtime observation, or tests have been performed. The acceptance examples express consequences of the supplied representation under their stated conditions; they are not claims of measured behavior.
