<!-- Trusted provenance: F-1ab8fadffe3d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ab8fadffe3d

## Purpose

F-1ab8fadffe3d conditionally transfers one 16-bit memory value, evaluates a sequence of arithmetic gates, and enters an unconditional cycle containing two indirect calls and a software interrupt.

A second halfword store is reachable only under restrictive arithmetic and address conditions. Under the 32-bit interpretation defined below, the subsequent memory-operation blocks are unreachable.

No application-level purpose is established. The available evidence does not justify identifying this function as a parser, allocator, record builder, or conventional returning API.

This contract describes static evidence and arithmetic deductions from the supplied representation. No runtime tests have been performed.

## Interface

### Explicit inputs

The displayed interface has five positional inputs:

| Position | Width | Established role |
|---|---:|---|
| First | 32 bits | No explicit use in the displayed body. |
| Second | Unsigned 32 bits | Its low two bits determine whether the initial transfer and subsequent arithmetic gates can execute. |
| Third | 32 bits | No explicit use in the displayed body. |
| Fourth | 32 bits | No explicit use in the displayed body. |
| Fifth | 16 bits | Its value is not directly consumed as an input expression; its storage address participates in arithmetic. |

The fifth input’s storage address cannot be replaced by its numeric value without changing the displayed behavior. Its storage may also be reached indirectly by memory accesses if their computed addresses alias it.

“No explicit use” does not establish that an unknown callee cannot observe corresponding machine state.

### Additional entry state

The explicit inputs do not fully determine behavior. The representation also consumes incoming machine-state values whose origins are not established by the displayed interface.

This contract assigns semantic names to those values:

| Name | Width | Role |
|---|---:|---|
| Transfer address, \(S\) | 32-bit address | Source of the initial halfword read. |
| Offset word, \(O\) | 32 bits | Byte offset used to form the initial destination. |
| Adjustment word, \(A\) | 32 bits | Subtracted from that destination to update internal address state. |
| Comparison word, \(C\) | 32 bits | Compared with the offset through a zero-result gate. |
| Auxiliary word, \(U\) | 32 bits | Used in a carry-aware addition. |
| Entry link address, \(L\) | 32-bit address | Participates in gates and normally determines the first indirect-call target. |
| Fifth-input storage address, \(P\) | 32-bit address | Address of the fifth input’s 16-bit storage. |

These names describe functional dependencies, not additional parameters guaranteed by a calling convention.

### Outputs and completion

No return value or normal return path is displayed.

If memory accesses complete, both indirect calls return, and execution resumes after each software interrupt, the function repeats its terminal cycle indefinitely. A fault, nonreturning dependency, or external control transfer can prevent subsequent actions.

### Arithmetic and memory interpretation

The deductions in this contract use:

- 32-bit words and addresses.
- Unsigned arithmetic modulo \(2^{32}\), except when explicitly evaluating carry out of a 32-bit addition.
- Logical right shifts of unsigned quantities.
- Byte-addressed memory.
- 16-bit halfword loads and stores.
- Two-byte scaling for arithmetic on halfword pointers.

A 16-bit store retains the low 16 bits of its value.

These are assumptions about the low-level representation’s arithmetic, not assurances that arbitrary addresses or pointer operations are valid in a portable language implementation. Memory mapping, alignment constraints, and endianness are not established.

## Behavior

### 1. Initial classification

The second input is multiplied by \(2^{30}\), retaining the low 32 bits:

| Second input modulo four | Classification value |
|---:|---:|
| 0 | `0x00000000` |
| 1 | `0x40000000` |
| 2 | `0x80000000` |
| 3 | `0xC0000000` |

The initial transfer executes exactly when the second input is divisible by four, including zero.

For all other second-input values, the initial transfer and the later gated memory operations are skipped. Execution proceeds to the terminal cycle, subject to the representation’s ordinary control flow.

### 2. Initial conditional transfer

For a second input divisible by four, define the destination address as:

\[
D = S + O \pmod {2^{32}}.
\]

The displayed ordering is:

1. Compute the destination.
2. Read one 16-bit value from address \(S\).
3. Replace the internal transfer-address state with:
   \[
   S' = D - A = S + O - A \pmod {2^{32}}.
   \]
4. Write the previously read 16-bit value to address \(D\).

The internal update does not change the destination of this write. The offset is a byte offset here; it is not multiplied by two.

The transfer comprises one halfword read and one halfword write, not a repeated operation. If source and destination coincide, the read and write are still present in the displayed behavior.

No null check, bounds check, alignment repair, or recoverable error result precedes these accesses. Their successful completion is an external precondition for subsequent observable actions.

### 3. Conditions for the exceptional store

The following arithmetic gates can reach an additional halfword store only after the initial-transfer path has been taken.

On a progressing path, each intermediate result used as a shift count is zero. The relevant shifts therefore do not introduce unknown nonzero shift-count behavior into this reachability deduction. The propagated carry is also zero before the auxiliary-word addition.

The conditions reduce to:

- The second input is divisible by four.
- The entry link address \(L\) is zero.
- The fifth-input storage address satisfies \(P \mathbin{\text{shift-right}} 1 = 0\).
- The offset word equals the comparison word: \(O=C\).
- The adjustment word equals the **updated** transfer-address state: \(A=S'\).
- The auxiliary word \(U\) is zero.
- The final address-derived sum is zero modulo \(2^{32}\).

The storage-address shift permits only \(P=0\) or \(P=1\). At this point the final sum is:

\[
P + (2O \bmod 2^{32}) \pmod {2^{32}}.
\]

The doubled offset is even, including after truncation to 32 bits. Consequently, \(P=1\) cannot produce zero. The complete reduced conditions are therefore:

\[
\begin{aligned}
&\text{second input} \equiv 0 \pmod 4,\\
&L=0,\qquad P=0,\qquad U=0,\\
&O=C,\qquad O\in\{\texttt{0x00000000},\texttt{0x80000000}\},\\
&A=S+O-A\pmod {2^{32}}.
\end{aligned}
\]

The last condition is equivalently:

\[
2A=S+O\pmod {2^{32}}.
\]

It is not generally equivalent to \(A=S\). The comparison uses the address state after the initial subtraction, not the incoming transfer address.

Modular division by two is not unique: when \(S+O\) is even, the equation has two adjustment-word solutions separated by `0x80000000`; when \(S+O\) is odd, it has none.

The doubling of \(O\) is truncated before the displayed carry checks. A bit lost by that doubling is not separately propagated as carry. Thus both listed offset values remain arithmetically eligible.

These conditions are sufficient to reach the exceptional block under the stated model if the preceding operations complete. Whether an actual invocation can place the fifth input’s storage at address zero remains unknown.

### 4. Exceptional halfword store and state assignment

On this path, the representation first assigns a machine-state location a value derived from \(64P\). Since the gate requires \(P=0\), the assigned numeric value is zero under the stated model.

The architectural meaning and externally observable effect of that location are **documented_only**. It is not established to be an ordinary global, stack pointer, or owned storage object.

The block then performs an address-derived halfword store involving two displayed reads through the link address. Because \(L=0\) is required:

- One 16-bit read from address zero supplies a destination address by zero extension.
- Another 16-bit read from address zero supplies a value to which fifteen is added.
- The low 16 bits of that sum are stored at the derived destination.

The relative evaluation order of these two reads is not established. They must not be assumed to return the same value for volatile, device-backed, or concurrently changing memory.

If the destination-producing read returns \(h_d\) and the value-producing read returns \(h_v\), the store is:

- Address: the zero-extended 16-bit value \(h_d\).
- Width: 16 bits.
- Value: \((h_v+15)\bmod 2^{16}\).

For stable memory containing a halfword \(h\) at address zero, this simplifies to a halfword store of \((h+15)\bmod 2^{16}\) at byte address \(h\).

The link-derived address remains zero on this path. An internal halfword-pointer value advances by one element, changing its byte-address value from zero to two.

### 5. Why later memory blocks are unreachable

The displayed representation contains additional blocks with signed halfword loads and wider stores. They do not establish reachable record layouts or outputs under this contract’s arithmetic model.

There are two cases:

- If the exceptional block is skipped, the prerequisite condition for the following gate chain is false.
- If the exceptional block executes, the next gate produces zero, but the following gate compares the advanced pointer value, two, against zero through exclusive-or. Its result is two, not zero, and progression stops.

The subsequent memory blocks therefore cannot execute under either case. Their apparent loads, stores, and address-derived structures must not be described as ordinary reachable effects.

This is a deduction about the supplied representation, not independent verification of the underlying executable.

### 6. Terminal call-and-interrupt cycle

When execution reaches the terminal cycle, each iteration has this order:

1. Invoke **F-indirect-A**, with three displayed 32-bit argument words:
   - `0x00000000`
   - `0x00000000`
   - `0xFFFFFFA5`
2. If that invocation returns, invoke **F-indirect-B**, with no explicit arguments displayed.
3. If the second invocation returns, request software interrupt `0x5B`.
4. If execution resumes after the interrupt, replace the link-derived address with `0x083D4CC2`.
5. Repeat.

The third argument to F-indirect-A is the bit pattern `0xFFFFFFA5`, equivalent to −91 under signed 32-bit interpretation. The dependency’s intended interpretation is unknown.

The first F-indirect-A target is:

\[
L+\texttt{0xEB6}\pmod {2^{32}}.
\]

The displacement is `0xEB6` bytes because the displayed pointer displacement is scaled by a two-byte element size. On the exceptional-store path, the first target is therefore `0x00000EB6`.

After an iteration completes through interrupt resumption, subsequent displayed F-indirect-A targets are:

\[
\texttt{0x083D4CC2}+\texttt{0xEB6}
=\texttt{0x083D5B78}.
\]

F-indirect-B has the fixed displayed target `0x0857A94B`.

These are target-address calculations, not guarantees of executable mappings, valid instruction-set state, or successful calls.

## Side effects

Reachable direct effects established by the representation are:

- A conditional halfword read followed by a halfword write for the initial transfer.
- Under the exceptional conditions, a representation-specific state assignment, two displayed address-zero halfword reads, and one derived-address halfword write.
- Repeated indirect calls and software-interrupt requests, conditional on preceding operations returning or resuming.
- Persistent execution without a normal return.

The body contains no explicit allocation, deallocation, ownership transfer, or resource-release operation. No ownership obligation is established for any input or addressed memory.

These observations do not constrain the dependencies. Unknown callees and the interrupt handler may modify globals, accessible memory, devices, or ownership state.

No recoverable error status is produced. Invalid accesses or call targets can prevent later behavior; the fault mechanism, reporting, and recovery semantics are unknown.

## Dependencies

The identifiers below are opaque labels for unresolved call roles, not recovered dependency identities.

| Opaque function ID | Role | Established call information |
|---|---|---|
| F-indirect-A | First call in each terminal iteration | Address-derived target; three displayed 32-bit arguments: zero, zero, `0xFFFFFFA5`. |
| F-indirect-B | Second call in each terminal iteration | Fixed displayed target `0x0857A94B`; no explicit arguments displayed. |

Software interrupt `0x5B` is a separate external dependency, not an identified ordinary function call.

No dependency is established to return, preserve globals or accessible memory, retain ownership, or release resources. Later events remain conditional on control returning or resuming and on the displayed continuation being followed.

## Acceptance examples

These are conditional static examples, not executed tests.

### Non-transfer input

Given a second input of one:

- No initial halfword transfer occurs.
- No exceptional or later gated memory operation occurs.
- The first terminal call target is the entry link address plus `0xEB6`, modulo \(2^{32}\).
- If the first call returns, the second call follows.
- If the second call returns, software interrupt `0x5B` follows.

Successful target resolution and interrupt resumption are separate preconditions.

### Ordinary transfer

Given:

- Second input four.
- Transfer address `0x1000`.
- Offset word `0x20`.
- A readable source halfword of `0xABCD`.
- A writable destination at `0x1020`.
- A nonzero entry link address.

The function reads `0xABCD` from `0x1000` and writes it as a halfword to `0x1020`. The internal transfer-address state becomes `0x1020` minus the adjustment word, modulo \(2^{32}\).

The nonzero entry link address prevents the exceptional-store chain from progressing. Execution reaches the terminal cycle if the accesses complete.

### Wrapped destination

Given an eligible transfer, source address `0xFFFFFFF0`, and offset `0x20`, the destination is `0x00000010`.

This arithmetic result does not authorize either access. Successful completion still requires the relevant memory mappings and access rules.

### Incoming-address equality is insufficient

Given:

- Second input four.
- Transfer address `0x1000`.
- Adjustment word `0x1000`.
- Offset, comparison, auxiliary, entry link, and fifth-input storage address all zero.
- Successful initial accesses.

The initial transfer reads and writes at `0x1000`. The updated transfer-address state is zero.

The subsequent comparison requires adjustment `0x1000` to equal that updated value, zero. It fails. No exceptional store occurs, despite the incoming transfer address equaling the adjustment word.

### Sufficient exceptional-store example

Given all of the following concrete conditions:

- Second input four.
- Incoming transfer address zero.
- Adjustment word zero.
- Offset word zero.
- Comparison word zero.
- Auxiliary word zero.
- Entry link address zero.
- Fifth-input storage address actually zero.
- Successful relevant accesses.
- Stable halfword contents `0x1234` at address zero during the reads.
- Writable halfword storage at address `0x1234`.

The initial transfer reads `0x1234` from address zero and writes it back to address zero. The updated transfer-address state is zero, so it equals the adjustment word.

All exceptional-store gates pass. The exceptional operation writes halfword `0x1243` to address `0x1234`. Later gated memory blocks remain unreachable. The first terminal call target is `0x00000EB6`.

### Halfword increment wraps

Under the same sufficient exceptional conditions, but with stable address-zero contents `0xFFFF` and a permitted halfword write at address `0x0000FFFF`, the exceptional store writes `0x000E` there.

The arithmetic wraps at 16 bits. Whether that odd destination permits a halfword access is platform-dependent.

### Dependency does not return

If F-indirect-A does not return during an iteration, neither F-indirect-B nor the software interrupt is reached in that iteration. The enclosing cycle supplies no recovery from a nonreturning dependency.

## Uncertainties

The following remain **documented_only**:

- Native calling convention and origins of the additional entry-state values.
- Whether the displayed entry state is realizable for a valid invocation.
- Whether fifth-input storage can actually occupy address zero.
- Architectural meaning of the representation-specific state assignment.
- Memory mappings, access permissions, alignment requirements, endianness, and fault behavior.
- Relative evaluation order and stability of the two exceptional halfword reads.
- Dependency identities, complete signatures, effects, ownership behavior, and return guarantees.
- Software-interrupt dispatch, handler behavior, and resumption semantics.
- Fidelity of the supplied representation to the underlying executable.

Inferred intent is limited to a conditional halfword transfer followed by persistent call-and-interrupt activity. No higher-level intent is established.

## Evidence

The sole evidence is the supplied static function representation. The earlier draft and reviewer feedback are not independent evidence of behavior.

Direct static evidence establishes the displayed input widths, conditional memory accesses, arithmetic operations, state assignment, call arguments, interrupt number, and unconditional terminal cycle.

Arithmetic deductions under the stated 32-bit model establish:

- The divisibility-by-four transfer condition.
- The exceptional-store requirements, including the actual zero storage address.
- The updated-address equation \(2A=S+O\pmod {2^{32}}\).
- The two eligible offset values.
- The unreachable later memory blocks.
- The wrapped address and call-target calculations.

No dependency inspection, implementation execution, runtime observation, or test execution supports this contract.
