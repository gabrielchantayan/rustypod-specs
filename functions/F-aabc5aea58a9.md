<!-- Trusted provenance: F-aabc5aea58a9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aabc5aea58a9

## Purpose

F-aabc5aea58a9 performs an entry-state-dependent loop, calls unresolved dependencies, writes through computed and returned addresses, and conditionally constructs a 20-byte destination before returning. Its application-level purpose is not established.

The supplied static representation supports an ordered memory-and-call contract, not a recovered object model. In particular, a returned address does not establish allocation, ownership transfer, or a release obligation.

This contract distinguishes:

- **Established static evidence:** operations and data dependencies displayed in the supplied representation.
- **Conditional arithmetic conclusions:** consequences of interpreting word arithmetic as 32-bit wrapping arithmetic and displayed signed shifts as arithmetic shifts.
- **documented_only behavior:** a structurally present later region that the displayed unsigned-byte branch cannot reach, plus unresolved interface and machine-semantics details.

The representation reports invalid instruction data and incomplete control-flow recovery. Its displayed behavior therefore cannot establish complete original-executable behavior. No runtime tests have been performed.

## Interface

### Explicit input and additional entry state

The displayed interface accepts one 32-bit integer and declares no return value.

The routine also consumes entry state not initialized within the displayed body:

| Entry state | Displayed use |
|---|---|
| 32-bit cursor | Base for opening stores; advanced by 74 each iteration |
| 32-bit accumulator | Advanced by 212 each iteration; controls loop continuation |
| 32-bit comparison value | Supplies an opening halfword store and a subtraction-overflow predicate |
| Return-address-related anchor | Selects the first indirect call target by adding 4,040 |

These roles do not establish a calling convention. The explicit integer alone is insufficient to describe a valid invocation.

Later indirect targets depend on displayed control-flow anchors. One target changes according to whether the optional call occurred. Their complete interfaces and identities are unknown.

### Outputs and access requirements

No ordinary return value is established. Observable effects include:

- Ordered byte, halfword, word, and wide memory accesses.
- Dependency invocations.
- Normal return on the displayed nonnegative branch.
- Two displayed invalid-instruction termination points.
- An unresolved terminal indirect transfer in the documented-only region.

The displayed widths are 8 bits for bytes, 16 bits for halfwords, 32 bits for words and address calculations, and 64 bits for wide results and the first final-destination store. Narrow stores retain the corresponding low bits.

All reached addresses must permit the indicated access for execution to continue normally. The routine shows no address validation, extent checks, or null checks. Alignment requirements, byte order, permissions, memory-mapped effects, and precise fault behavior remain unknown.

### Arithmetic interpretation

Unless separately qualified, arithmetic conclusions below use low-32-bit wrapping addition, subtraction, multiplication, negation, and left shift. Signed comparisons interpret those bits as signed 32-bit values. Displayed arithmetic right shifts propagate the sign bit.

This interpretation is not a claim about language-level signed-overflow rules or exact original instruction semantics. Signed-overflow predicates describe whether the mathematical signed result exceeds the signed 32-bit range; they are not unsigned carry or borrow tests.

## Behavior

### Opening loop

At least one iteration executes. Each iteration:

1. Retains the current cursor as its iteration base.
2. Advances the cursor by 74.
3. Advances the accumulator by 212.
4. Shifts the explicit input left by two bits.
5. Writes the low 16 bits of the advanced cursor at iteration-base offset 28.
6. Writes the low 16 bits of the entry comparison value at that same address.

The two halfword stores are distinct ordered accesses. In ordinary memory, if both complete and nothing intervenes, the second replaces the first. That final-value observation does not permit removing the first access from an observable-access contract.

The loop continues when either:

- The advanced accumulator is zero; or
- Its signed-negative predicate differs from the signed-overflow predicate for the entry comparison value minus 67.

The comparison value is unchanged by the displayed loop, so that overflow predicate is constant.

Under the wrapping interpretation, adding 212 traverses a cycle of \(2^{30}\) accumulator values. That cycle contains nonzero values of both signs. Therefore the continuation condition eventually becomes false, within at most one complete cycle, provided the accesses complete and execution follows the displayed flow. An access fault can prevent exit; an infinite opening loop is not supported under these conditions.

### Initial result and optional call

After loop exit, an indirect dependency receives four displayed arguments: the repeatedly shifted input, followed by three copies of the final iteration base.

From its 32-bit result, the routine derives:

- An adjusted result by subtracting 81.
- A doubled adjusted result.
- The adjusted result arithmetically shifted right by 13.

It saves the signed-subtraction-overflow predicate for the **advanced cursor minus 62**. It then writes the low byte of the shifted result at final iteration-base offset 25, followed by offset 17.

The shifted result’s sign determines whether the optional call executes. The low 16 bits of the doubled adjusted result are retained independently for later stores.

If the shifted result is negative:

1. An unresolved dependency receives the adjusted result plus 80 and the constant 50.
2. Its result replaces the adjusted result used by the next address-producing call.
3. The saved overflow predicate is replaced by signed-addition overflow for the displayed stack-relative address at offset 700 plus 91.
4. The displayed loop-condition flag becomes false.
5. If the displayed stack-relative address at offset 791 is signed-negative, execution reaches an invalid-instruction endpoint.

The optional dependency executes at most once. Returning from it does not cause another optional call, because the controlling flag has been cleared.

The retained doubled-result halfword is not replaced by the optional dependency’s result. The earlier invalid-instruction endpoint occurs after the optional call and both byte stores, but before the next address-producing call.

If the optional path is not selected, the loop-condition flag is already false and the saved overflow predicate remains that for advanced cursor minus 62.

The stack origin and validity of the stack-derived addresses are unresolved. The two stack expressions remain separate displayed checks; neither is assumed valid or nonnegative.

### Address-producing calls and late termination guard

The next indirect dependency receives:

- The current adjusted result, possibly replaced by the optional dependency.
- A reference to fixed data whose contents are not supplied.

Its 32-bit returned value is used as an address. The routine writes that value’s low 16 bits at returned-address offset 4.

The following operations occur in order:

1. An unresolved call with no displayed explicit arguments.
2. A byte store at final iteration-base offset 21, containing the low byte of the retained doubled-result halfword.
3. Another unresolved call with no displayed explicit arguments, obtaining a new destination base.
4. Evaluation of the late invalid-instruction guard.

At the guard, the displayed loop-condition flag is false. Execution terminates when that flag differs from the saved overflow predicate: equivalently, when the saved predicate is true.

- Without the optional call, the predicate concerns advanced cursor minus 62. It overflows for signed cursor values from −2,147,483,648 through −2,147,483,587 inclusive.
- With the optional call, it concerns stack-relative address at offset 700 plus 91. It overflows when the address before addition lies from 2,147,483,557 through 2,147,483,647 inclusive.

Reaching this guard after the optional call also requires surviving the earlier stack-address sign check. The representation does not establish that the two stack-related conditions can be varied independently.

If the late guard passes, the routine writes:

1. The retained halfword at new-destination offset 20.
2. Its low byte at new-destination offset 26.
3. The constant 71 to global storage at numeric address 162.

The width of the last assignment is not established by the supplied declaration information.

### Global capture and indexed word store

After that global assignment, the routine reads a global pointer-like word. Its retained value is called the **captured base** below.

An unresolved dependency receives 29 and returns a displayed 64-bit value. The routine uses the high 32 bits as an address and reads an unsigned byte at offset 13. Call this value the **captured byte**; its range is 0–255.

Using retained values, the routine:

- Forms captured base minus 1.
- Writes the 32-bit constant 51 at captured base plus captured byte minus 2.
- Forms captured base plus 108.
- Calls another unresolved dependency with twice the quantity obtained by subtracting 54 from the wide result’s low 32 bits.

The indexed write is a four-byte access. Its address can be unaligned; no element type or array bounds are established.

After the call, the branch value is twice the captured byte plus 99. Its exact range is 99–609, so it is always positive as a signed 32-bit value and cannot overflow.

This conclusion uses captured values, not fresh global reads. It imposes no global-preservation requirement on the intervening dependency. Whether the representation accurately models live machine values across calls remains unresolved.

### Normal-return destination

On the necessarily nonnegative branch, F-c260788e6b26 receives 115. Its displayed 64-bit result supplies both a destination address through its high 32 bits and the complete value of the first store.

The routine performs these writes in order:

| Destination offset | Width | Stored value |
|---|---:|---|
| 0 | 64 bits | Complete returned wide value |
| 8 | 32 bits | Twice the captured byte plus 99 |
| 12 | 32 bits | Captured base plus 108 |
| 16 | 32 bits | Captured base minus 1 |

It then returns without a declared result.

The destination requires 20 writable bytes for these accesses to complete. There is no displayed validation, rollback, or explicit release. “Destination” describes its use, not its ownership or allocation status.

### Structurally present later region — documented_only

The alternative branch cannot be reached through the displayed unsigned-byte calculation. The following behavior is documented_only, not an ordinary-path requirement. Reachability would require an unrepresented transfer or different actual semantics.

Its initial ordered accesses and calls are:

1. Read a word at captured-base offset 20.
2. Read a word at captured-base offset 16 and add 100.
3. Read a word at captured-base offset 12 for an unresolved three-argument call.

That call receives, in argument order:

- **\(2 \times (\text{captured base} - 1)\)**.
- The word read at offset 12.
- The adjusted offset-16 word.

The first argument subtracts 1 **before** doubling, retaining the low 32 bits at each arithmetic step. It is not \(2 \times \text{captured base} - 1\).

A subsequent call receives the offset-20 word plus 105, captured base plus 24, and the adjusted offset-16 word minus 76.

For the remaining description, define these descriptive quantities under the stated arithmetic interpretation:

- **Shift base:** captured base plus 109, arithmetically shifted right by 26.
- **Doubled shift:** twice the arithmetic-right-shift-by-13 result of captured base plus 109.
- **Adjusted shift:** doubled shift minus 61.
- **Square base:** the low 32 bits of the square of captured base plus 24.

The routine reads a word at captured-base offset 124, then writes words at shift-base offsets 52, 56, and 60. Their values, respectively, are captured base plus 24, adjusted shift, and the offset-124 word.

Only after those stores does it read a global value for the first argument to F-943b4f706744; the second argument is 67. This read can observe preceding aliasing stores or earlier dependency effects.

After F-943b4f706744 returns, a separate global word read supplies a later byte-access base. That read must not be merged with the earlier argument read.

The remaining displayed order is:

1. Make an unresolved call; store the low 16 bits of doubled shift at adjusted shift plus an auxiliary register output.
2. Make another call; read a byte at doubled shift plus the separately read global base, extend it according to the displayed signed interpretation, and pass it to another call.
3. Read an unsigned byte at doubled shift plus an auxiliary output associated with the preceding call sequence. Retain this byte as the **small address value**.
4. Make a call; write doubled shift at square-base offset 52.
5. Make a value-producing call; pass its result and a word read at small-address-value offset 20 to another call.
6. Read a word at small-address-value offset 4 and pass it to another call.
7. Write the retained offset-124 word at square-base offset 100.
8. Make a call; write doubled shift again at square-base offset 52.
9. Make a call; write adjusted shift at square-base offset 36.
10. Make a call; write an auxiliary register output at square-base offset 20.
11. Make two more calls; read an unsigned halfword at doubled shift plus an auxiliary output.
12. Make a wide-result call.
13. Store the retained small-address byte at the address equal to that unsigned halfword plus the small address value.
14. Read a halfword at the sum of the wide result’s low and high 32-bit halves. Pass the low half, high half, negated low half, and read halfword to another call.
15. Make two further calls, then perform an unresolved indirect transfer represented as a call before return.

No valid object extent follows from these computed addresses. In particular, the unsigned byte is used as a numerically small address base for word reads.

## Side effects

Effects are not transactional. Stores and dependency effects preceding a fault or invalid-instruction endpoint need not be undone.

All memory regions may alias unless an external contract proves otherwise. A direct store can affect a later global read. Conversely, a captured value remains distinct from subsequent contents of its source storage under the displayed dataflow.

Dependencies may mutate globals, invalidate memory, allocate or release resources, invoke callbacks, fault, or transfer control nonlocally. None is assumed to preserve globals, preserve memory lifetimes, or return normally.

No explicit deallocation is displayed. Neither that absence nor use of returned addresses establishes ownership.

There is no established synchronization, concurrency-safety, or atomicity guarantee. Displayed access widths do not prove hardware-atomic accesses.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-c260788e6b26 | Receives 115 on the displayed normal-return path. Its 64-bit result supplies the destination address and first stored value. |
| F-943b4f706744 | Receives a global value read after three documented-only word stores, followed by 67. A separate global word is read after its return. |

Other calls have no supplied opaque identity. Their argument and result roles are described above without inventing identities.

“No displayed explicit arguments” does not establish a zero-argument machine interface. Several documented-only operations consume auxiliary register outputs whose contracts are unavailable. The final indirect transfer’s target and return behavior are unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where termination is the expected outcome, they require completed accesses, normally returning dependencies, and the stated arithmetic interpretation.

1. **Opening overwrite:** A completed iteration performs two halfword stores at iteration-base offset 28. Without intervening mutation, the second leaves the low 16 bits of the entry comparison value.

2. **Finite opening loop:** With completing accesses and the displayed constant comparison value, wrapping accumulator updates cannot sustain the opening loop indefinitely.

3. **Late termination without the optional call:** If the advanced cursor is −2,147,483,648 and the optional path is not selected, cursor-minus-62 overflows. The routine reaches the late invalid-instruction endpoint after the returned-address halfword store and iteration-base offset-21 byte store, but before new-destination offsets 20 and 26.

4. **Earlier optional termination:** If the optional call returns and the displayed stack-relative address at offset 791 is signed-negative, termination occurs before the next address-producing call.

5. **Retained halfword:** Changing the optional call’s result can change the following dependency argument. It does not replace the halfword derived from doubling the original adjusted result.

6. **Byte boundaries:** Captured byte 0 selects the indexed word-store address captured base minus 2 and final offset-8 value 99. Byte 255 selects captured base plus 253 and final value 609. Both select normal destination construction.

7. **Global mutation after capture:** An intervening dependency may change the pointer-like global. The final fields nevertheless use its previously captured value under the displayed dataflow.

8. **Correct documented-only grouping:** If the captured base is 10 and the later region is hypothetically reached, the first three-argument call receives 18 as its first argument, not 19. If the captured base is zero, that argument is 4,294,967,294 as a word bit pattern, equivalent to signed −2.

9. **Later global-read ordering:** If one of the three shift-base stores aliases the global supplying F-943b4f706744’s first argument, the argument read can observe that store. A dependency mutation can separately affect the global word read after the call.

## Uncertainties

**Established static evidence:** The representation displays the listed widths, access and call ordering, constants, captured-value relationships, grouped subtraction-before-doubling expression, overflow predicates, and destination layout.

**Conditional conclusions:** Opening-loop finiteness, modular address expressions, and the positive byte-derived branch range depend on the stated arithmetic interpretation and displayed dataflow.

**Inferred intent:** Terms such as “destination construction” describe layout and use only. No application semantics, resource type, or allocation protocol is inferred.

**documented_only unresolved matters:**

- Actual calling convention and hidden entry-state requirements.
- Exact instruction, flag, shift, and signedness semantics where recovery is unreliable.
- Stack origin and validity of stack-derived address interpretations.
- Completeness of recovered control flow and termination behavior.
- Width of the global assignment at numeric address 162.
- Fixed-data contents, initial global values, and unidentified dependency contracts.
- Address validity, alignment, byte order, lifetime, and ownership.
- Auxiliary register-output associations and preservation rules.
- Actual executability of the later region.
- Target and return behavior of the final indirect transfer.

## Evidence

This contract is derived from the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

The supplied representation reports invalid instruction data, removed unreachable regions, overlapping global storage symbols, and an unrecovered multiway transfer. These limitations constrain claims about the original interface and executable reachability.

The corrected documented-only argument follows the displayed dependency: captured base minus 1 is retained before that value is doubled.

No implementation access, runtime observations, smoke executions, or tests were performed. The acceptance examples state conditional static expectations, not observed execution results.
