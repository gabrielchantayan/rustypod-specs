<!-- Trusted provenance: F-91bcfddb00d3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-91bcfddb00d3

## Purpose

F-91bcfddb00d3 describes a straight-line sequence of indirect calls, coprocessor transfers, integer transformations, and a final 16-bit memory write. The supplied static representation ends at an invalid-instruction boundary rather than a normal return.

No application-level purpose is established. In particular, the coprocessor operations do not, by themselves, establish cache maintenance, synchronization, device initialization, or any other named hardware service.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Conditional arithmetic conclusions:** consequences of interpreting the represented integer operations as fixed-width, 32-bit bit-vector operations.
- **Unknown behavior:** dependency semantics, actual instruction validity, hardware effects, and execution beyond the represented boundary.

All described later operations are conditional on earlier calls returning and earlier operations completing without transferring control elsewhere.

## Interface

### Entry and inputs

The recovered interface declares no explicit parameters. It nevertheless depends on ambient machine state:

- An incoming link-register value participates in selecting the first indirect-call target.
- An incoming general-purpose register supplies a value shifted left by 30 for the fourth call.
- Incoming coprocessor-related operands appear in nine coprocessor-transfer operations.
- The execution environment supplies the validity and behavior of every indirect-call target.

Consequently, “no explicit parameters” does not mean that behavior is independent of entry state.

The declared interface is insufficient to establish the complete calling convention, register-preservation requirements, stack requirements, privilege level, or valid entry contexts.

### Widths and result notation

The following notation describes values without naming source-local storage:

| Symbol | Meaning |
|---|---|
| \(R\) | Incoming general-purpose value consumed by the fourth call |
| \(P\) | 32-bit result of dependency F-U03 |
| \(A\) | \(P\) shifted left by 27, reduced to 32 bits |
| \(B\) | 32-bit result of dependency F-U18 |
| \(V\) | \(B\) shifted left by 4, reduced to 32 bits |
| \(C\) | 32-bit result of dependency F-U22 |
| \(D\) | 64-bit result of dependency F-U24 |
| \(E\) | 64-bit result of dependency F-U30 |
| \(G\) | 64-bit result of dependency F-U31 |

For a 64-bit value, “low half” means bits 0–31 and “high half” means bits 32–63. These are numeric halves, not claims about byte order in memory.

Unless explicitly identified as 64-bit, arithmetic conclusions below use 32-bit operands and discard bits beyond bit 31. This interpretation is necessary because several left shifts are displayed on signed types; the supplied representation does not establish portable signed-language arithmetic semantics.

### Outputs and termination

There is no established ordinary return value.

The visible sequence culminates in a write of the 16-bit value `0x00CD`, followed by an invalid-instruction termination marker. The evidence does not establish whether actual execution traps, faults, stops, continues through differently decoded instructions, or reaches that boundary at all.

Dependency calls and coprocessor operations may have additional outputs or side effects that are not described by their displayed return values.

## Behavior

### 1. Initial calls and first derived argument

The function first invokes F-U01, whose target depends on incoming link-register state. No explicit arguments are displayed.

It then invokes F-U02 and uses its 32-bit result as the payload for two successive coprocessor transfers. Both transfers have the displayed selector prefix `(0, 3, 7)`. Their remaining operands differ:

1. The first uses incoming coprocessor operands labeled 6 and 15.
2. The second uses incoming operand 15 in both remaining positions.

The same captured result is supplied to both transfers. No dependency call is displayed between them.

F-U03 is then invoked. Its result establishes \(P\), from which \(A\) is formed by a 27-bit left shift.

F-U04 receives four explicitly represented 32-bit arguments, in order:

1. \(R\) shifted left by 30.
2. Zero.
3. \(A\).
4. Zero.

Under fixed-width interpretation, only the low two bits of \(R\) contribute to the first argument, and only the low five bits of \(P\) contribute to \(A\).

### 2. Intermediate calls and transfers

F-U05 through F-U09 are invoked in that order, with no explicit arguments displayed.

F-U10 is then invoked. Its 32-bit result becomes the payload of a coprocessor transfer with selector prefix `(0, 1, 7)` and remaining incoming operands 1 and 15.

F-U11, F-U12, and F-U13 follow, in order.

Next, the function performs a coprocessor transfer with selector prefix `(0, 5, 7)`, payload \(A\) shifted left by 24, and remaining incoming operands 5 and 15.

F-U14 receives that same shifted payload followed by zero.

Under the 32-bit interpretation, \(A\) has nonzero bits only in positions 27–31. Shifting it left by another 24 positions therefore produces zero. Both the transfer payload and F-U14’s first argument are consequently zero.

F-U15 is then invoked.

F-U16 supplies a 32-bit result for a coprocessor transfer with selector prefix `(0, 6, 7)` and remaining incoming operands 9 and 15.

F-U17 follows.

### 3. Second derived value and zero arguments

F-U18 receives three displayed arguments. Each derives from a value explicitly set to zero earlier, with no intervening displayed assignment to that value. The arguments are therefore zero, zero, and zero in the represented dataflow.

Its 32-bit result establishes \(B\).

A coprocessor transfer follows with:

- Selector prefix `(0, 5, 7)`.
- Payload \(B\) shifted left by one.
- Remaining incoming operands 10 and 15.

The function forms \(V\), the low 32 bits of \(B\) shifted left by four.

F-U19 receives:

1. \(B\) shifted left by one.
2. The low 23 bits of \(A\), shifted left by four.
3. \(V\).
4. Zero.

The second argument is zero under the fixed-width interpretation because \(A\)’s low 27 bits are already zero.

F-U20 and F-U21 are then invoked.

Two coprocessor transfers follow, in order:

1. Selector prefix `(0, 7, 7)`, zero payload, remaining operands 9 and 15.
2. Selector prefix `(0, 0, 7)`, zero payload, remaining operands 15 and 15.

Between their represented positions, a value is formed from the low two bits of \(V\), shifted left by 16. Since \(V\)’s low four bits are zero, this derived value is zero. It is subsequently used to produce zero arguments for F-U26 and F-U34.

### 4. Further shifts and a 64-bit result

F-U22 receives zero and returns \(C\).

A coprocessor transfer then uses selector prefix `(0, 2, 7)`, payload \(V\) shifted left by eight, and remaining operands 9 and 15.

F-U23 receives three arguments:

1. \(V\) shifted left by eight.
2. \(V\) shifted left by twelve.
3. \(C\) shifted left by nine.

Equivalently, with 32-bit truncation, the first two are \(B\) shifted left by 12 and \(B\) shifted left by 16. The transfer immediately before F-U23 uses the same bit pattern as its first argument.

F-U24 returns the 64-bit value \(D\).

F-U25 receives:

1. The low half of \(D\).
2. The high half of \(D\).
3. The low half of \(D\), shifted left by one.

No memory layout for this 64-bit value is shown.

### 5. Result chain and final calls

F-U26 receives zero. Its 32-bit result becomes the sole displayed argument to F-U27.

F-U27’s 32-bit result becomes the sole displayed argument to F-U28.

F-U29 is then invoked, followed by F-U30, which returns \(E\).

F-U31 receives:

1. The low half of \(E\).
2. The high half of \(E\).
3. The high half of \(E\), shifted left by eleven.

F-U31 returns \(G\).

F-U32 receives the low half of \(G\).

F-U33 is invoked next. Its 32-bit result becomes F-U34’s first argument; F-U34’s second argument is zero.

F-U35 is then invoked.

The high bits of \(G\) have already been captured for final address formation before F-U32. The represented dataflow does not reload \(G\) from memory after the intervening calls. This does not establish that those calls preserve global state or that the recovered calling convention is complete.

### 6. Final memory write

The final write address is derived from \(G\), not from a returned pointer expressly identified as such.

First, the function logically shifts \(G\) right by 41 bits, yielding a value no larger than \(2^{23}-1\). That value is converted to signed 32-bit form and shifted right by eleven. Because it is nonnegative and fits in signed 32-bit range, the represented signed shift introduces no sign-extension distinction here.

Let:

\[
T = G \mathbin{\gg} 52
\]

The resulting numeric address is:

\[
T + \left\lfloor T/4 \right\rfloor
\]

The function stores `0x00CD` as a 16-bit value at that address.

Since \(T\) is a 12-bit value:

- \(T\) ranges from 0 through 4095.
- The numeric destination ranges from 0 through 5118, or `0x13FE`.
- Different values of \(G\) with identical top 12 bits produce the same destination.
- The address need not be aligned for a 16-bit store.
- No base pointer, allocation result, bounds check, null check, or alignment adjustment is displayed.

After this write, the representation reaches the invalid-instruction termination marker.

## Side effects

### Explicitly represented effects

The function contains:

- Thirty-five indirect dependency calls.
- Nine coprocessor-transfer operations.
- One explicit 16-bit memory write.
- An invalid-instruction boundary at the end of the represented sequence.

Their relative ordering is specified above. No branch, loop, retry, or explicit error-recovery path is represented.

The final write is sequenced after F-U35. The address calculation’s dependence on an earlier result does not move the write ahead of that call.

### Memory and ownership

No explicit ordinary memory read is shown outside effects that may occur inside dependency calls or coprocessor operations.

No allocation, ownership transfer, release, or lifetime guarantee is established. In particular:

- Dependency results must not be assumed to denote owned memory.
- Passing a numeric result to another dependency does not establish transfer of ownership.
- Dependencies must not be assumed to preserve globals or avoid memory access.
- The final store must not be assumed to target an object owned by this function.

The representation specifies a 16-bit write but not its byte order, atomicity, volatility, bus transaction width, or fault behavior. Those require architecture and environment evidence.

## Dependencies

F-U01 through F-U35 are document-local opaque identifiers, assigned by invocation order. They are not resolved external function identities. No corresponding dependency bodies or semantic contracts were provided.

| Dependency | Displayed arguments | Consumed result |
|---|---|---|
| F-U01 | None | None |
| F-U02 | None | 32-bit coprocessor payload |
| F-U03 | None | \(P\) |
| F-U04 | Four 32-bit values | None |
| F-U05–F-U09 | None | None |
| F-U10 | None | 32-bit coprocessor payload |
| F-U11–F-U13 | None | None |
| F-U14 | Two 32-bit values | None |
| F-U15 | None | None |
| F-U16 | None | 32-bit coprocessor payload |
| F-U17 | None | None |
| F-U18 | Three zero values | \(B\) |
| F-U19 | Four 32-bit values | None |
| F-U20–F-U21 | None | None |
| F-U22 | Zero | \(C\) |
| F-U23 | Three 32-bit values | None |
| F-U24 | None | \(D\), 64 bits |
| F-U25 | Three 32-bit values | None |
| F-U26 | Zero | 32-bit result for F-U27 |
| F-U27 | F-U26 result | 32-bit result for F-U28 |
| F-U28 | F-U27 result | None |
| F-U29 | None | None |
| F-U30 | None | \(E\), 64 bits |
| F-U31 | Three 32-bit values | \(G\), 64 bits |
| F-U32 | Low half of \(G\) | None |
| F-U33 | None | 32-bit result for F-U34 |
| F-U34 | F-U33 result, zero | None |
| F-U35 | None | None |

“None” in the arguments column means no explicit arguments are displayed, not proof that the callee consumes no ambient registers or other implicit inputs. An unconsumed result likewise does not imply that the dependency returns nothing.

## Acceptance examples

These are conditional static acceptance criteria, not results of executed tests. They assume the represented path is executable, dependencies return as required, and integer operations have the fixed-width interpretation described above.

1. **Initial bit selection.** If \(P=1\), then \(A=\texttt{0x08000000}\). If \(P=33\), \(A\) is unchanged because only \(P\)’s low five bits survive. In either case, F-U14 receives zero as both arguments.

2. **Incoming register truncation.** If \(R=3\), F-U04’s first argument is `0xC0000000`. If \(R=4\), that argument is zero. Neither case changes F-U04’s second or fourth argument.

3. **Second-result propagation.** If \(B=1\), the subsequent coprocessor payload based on \(B\) is 2, and F-U19 receives `(2, 0, 16, 0)`. F-U23 later receives first and second arguments of `0x00001000` and `0x00010000`.

4. **Overflow truncation.** If \(B=\texttt{0x80000000}\), its one-bit and four-bit left shifts both reduce to zero. The corresponding F-U19 arguments are zero; no overflow error branch is represented.

5. **64-bit half ordering.** If \(D=\texttt{0x1122334455667788}\), F-U25 receives `0x55667788`, `0x11223344`, and `0xAACCEF10`, in that order. This criterion does not depend on memory endianness.

6. **Zero-address write.** If \(G\)’s top 12 bits are zero, the final destination is numeric address zero. The representation does not suppress the write. Successful completion depends on that address being writable.

7. **Unaligned destination.** If \(G\)’s top 12 bits equal 1, the destination is address 1. The representation still requests a 16-bit write. Whether it succeeds or faults is platform-dependent.

8. **Maximum destination.** If \(G\)’s top 12 bits are all ones, the destination is `0x13FE`. Changes to the remaining 52 bits do not change that address.

9. **Interrupted sequence.** If F-U20 does not return, no later transfer, call, or final write is reached through this represented sequence. There is no displayed recovery or alternate path.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation corresponds to a valid, correctly decoded instruction stream.
- Whether the selected entry point and function boundary are correct.
- The concrete identities and contracts of F-U01 through F-U35.
- The complete calling convention and implicit arguments.
- Whether repeated target expressions resolve to distinct functions.
- Coprocessor instruction semantics, privilege requirements, ordering guarantees, and faults.
- Whether incoming coprocessor operand labels denote values, encoded selectors, or reconstruction artifacts.
- Actual writable mappings for the final low numeric address.
- Endianness and handling of unaligned 16-bit stores.
- Error signaling by dependencies and behavior following a fault.
- Execution beyond the invalid-instruction boundary.

The apparent straight-line structure and unusual low-address store do not justify inventing a higher-level intent or silently correcting the representation.

## Evidence

The sole evidence is the supplied static function representation. It explicitly includes a warning about invalid instruction data and a terminal invalid-instruction marker.

The call order, argument relationships, nine coprocessor transfers, 64-bit result splitting, and final 16-bit store are directly represented. Constant-zero arguments and the simplified final address follow from the stated fixed-width arithmetic interpretation.

No dependency implementation, architecture specification, memory map, execution trace, or runtime test result was provided. No runtime tests have been performed.
