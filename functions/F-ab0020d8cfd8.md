<!-- Trusted provenance: F-ab0020d8cfd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab0020d8cfd8

## Purpose

F-ab0020d8cfd8 represents a fixed sequence of 140 external-call occurrences, 17 coprocessor transfers, and a final 16-bit memory write. The supplied representation ends at an invalid-instruction marker; it does not establish a normal return.

No application-level purpose is established. Hardware initialization, successful completion, and recovery from dependency errors are not supported interpretations.

This contract distinguishes:

- **Established static evidence:** represented operation order, argument and result dependencies, widths, target equalities, transfer operands, and the final store.
- **Conditional arithmetic deductions:** reductions that follow when word operations truncate to 32 bits.
- **`documented_only` unknowns:** executable validity, actual calling conventions, dependency behavior, hardware interpretation, ownership, and endpoint behavior.

Reaching each operation requires control flow to reach it without an earlier nonreturning call, trap, or other interruption. A dependency that returns normally with a failure-indicating value does not, merely by returning that value, prevent subsequent operations. No runtime tests have been performed.

## Interface

### Explicit and implicit inputs

The represented interface declares no explicit parameters and no return value. Its behavior nevertheless depends on:

- An entry link-register value used to calculate the first call target.
- A 32-bit incoming general-register value, designated \(G\), whose low byte contributes to an early argument.
- Eight represented 32-bit coprocessor-context inputs, designated \(K_0\) through \(K_7\).
- Ordinary results and auxiliary register outputs from dependencies.

Different context labels identify different represented input sources, not necessarily different numerical values.

S001–S140 identify call **occurrences** in represented order. They are not function identities and do not imply distinct targets.

Consumed ordinary results are represented as 32-bit words except at S022, S026, S047, S058, S059, S071, S072, S105, S129, S135, and S136, where they are 64-bit values. A 64-bit value’s low and high halves mean numerical bits 0–31 and 32–63. This division does not establish physical register order or memory byte order.

Five additional 32-bit auxiliary outputs are consumed after S034, S048, S065, S077, and S090. They are not interchangeable with ordinary results and are not established as zero. Whether they are valid outputs under an actual calling convention remains unknown.

### Arithmetic and output interpretation

Unless stated otherwise, arithmetic deductions use truncating 32-bit word operations. A left shift discards bits above bit 31; consecutive shifts do not create a wider intermediate.

Some represented intermediates are signed. Simplifications involving their left shifts describe conditional machine-word behavior, not portable signed-language arithmetic. The signed right shift before S103 has an unresolved sign-extension interpretation, described below.

No status result is represented for the function. The final explicit memory output is a 16-bit write of numeric value 205, hexadecimal 00CD.

### Execution requirements

No input validation is represented. Continuing through the sequence requires callable targets, supported coprocessor operations, and uninterrupted control flow. Completing the final store additionally requires its computed destination to permit a 16-bit write.

Address validity, instruction support, privilege, mapping, permissions, alignment, implicit-input validity, and calling-convention compatibility are unchecked requirements, not guarantees supplied by this function.

## Behavior

### Observable ordering

Calls occur in ascending S-number order. Coprocessor transfers occur at the insertion points listed under **Side effects**. The final memory write follows S140.

Calls with no explicit arguments or no consumed ordinary result remain potentially effectful. They must not be omitted, coalesced, or reordered on the basis that their results are unused.

There is no explicit ordinary-memory load in the representation. Dependencies and coprocessor operations may nevertheless read memory, globals, or device state.

The value relationships below describe the supplied representation. They do not establish physical register-read timing or imply that unknown dependencies preserve globals, registers, or hardware state.

### Entry and initial processing

S001 calls a target calculated as the entry link-register value plus 2,422 bytes. Address-width overflow, execution-state interpretation, and target validity remain unresolved.

S002 receives four arguments: zero, zero, zero, and the low byte of \(G\) shifted left by 22. Under word semantics, its fourth argument can occupy only bits 22–29. S002’s word result is passed to S003; S003’s word result supplies T01.

F-ea490577ce35 occurs at S009.

S010 produces a word passed to S011 in four forms: unchanged, shifted left by 17, zero, and shifted left by 27. S014 produces the payload for T02.

No argument meaning such as pointer, status, length, or ownership is established by these operations.

### First paired results and auxiliary output

S022 produces a 64-bit value. Its low half passes through the word-result chain S023–S025, with S025 supplying T03. Its high half is retained for later processing through S030–S032.

S026 independently produces a 64-bit value. S027 receives its low half and its high half shifted left by 17.

S030 receives the S022 high half shifted left first by 19 and then by nine. Under truncating word semantics, this is a shift left by 28. S030’s result passes through S031 to S032.

S033 produces a word supplied to S034 alongside two zeros. After S034, T04 transfers a locally derived zero. S035 then receives zero, the auxiliary output associated with S034 shifted left by seven, zero, and zero. The second argument is not statically zero.

S039 receives zero and produces a word passed to S040 with another zero. S044 produces a word passed to S045; S045’s result is supplied to S046 both unchanged and shifted left by 20.

### Middle paired-result processing

S047 produces a 64-bit value retained across several calls:

- S048 receives its low half.
- S049 receives that low half shifted left by 14, the auxiliary output associated with S048, and the high half shifted left first by nine and then by 14.
- S050 receives the same high-half-derived expression.
- S052 receives zero, zero, the low 18 bits of the S047 low half shifted left by four, and zero.

The high-half expression supplied to S049 and S050 equals a truncating word shift left by 23. S052’s masked field is distinct from the unmasked shift supplied to S049.

S053 produces the payload for T05.

S058 produces another 64-bit value. T06 transfers its low half before S059. S059 receives that low half and the high half shifted left by 20, then produces a new 64-bit value. S060 receives the new low half, new high half, and new high half shifted left by 12.

### Retained seed, local zeros, and auxiliary outputs

S064 produces a word with two uses. The unchanged word is passed to F-0e02e4c13579 at S065. A copy shifted left by 11 is retained before that call. S066 receives zero, the auxiliary output associated with S065, and zero.

S071 produces a 64-bit result. S072 receives its low half shifted left by four, its unchanged high half, and the retained S064-derived value shifted left another 17. Under word semantics, the third argument equals the original S064 word shifted left by 28.

S072 produces a 64-bit result whose low half, high half, and a locally zero-derived third argument feed S073. S073’s word result is passed to S074.

S076 receives two zeros and produces a word passed to S077. S078 consumes the auxiliary output associated with S077, receiving zero, that auxiliary output, zero, and zero. Its first zero follows from shifting the already shift-by-11 S064-derived word left another 31 bits. The remaining zeros derive from explicitly established local zero values.

S078’s word result supplies T07 and is then passed to S079. Further word-result chains run through S080–S082 and S086–S087. S088 receives zero.

The auxiliary output associated with S090 becomes S091’s middle argument, between two zeros.

These represented retained values do not establish preservation guarantees for particular physical registers or external state.

### Truncated signed field and later transfers

S094 produces a word retained for S095 and S103. S095 receives the word unchanged, zero, and the word shifted left by five.

S103 receives zero and that retained shift-left-five intermediate shifted right by five. The second argument is not generally the original S094 word:

- With arithmetic right shifting, it sign-extends the original low 27 bits.
- With logical right shifting, it zero-extends those bits.

The intermediate is represented as signed, but executable instruction semantics have not been validated. S103’s word result is passed to S104.

Intervening word-result chains include S096–S097, S099–S100, and S101–S102. F-3de949054bdd occurs at S098.

S105 produces a 64-bit value. Its low half supplies T08; its high half shifted left by one is passed to S106.

S107 produces a word used by two separate, consecutive transfers, T09 and T10, before S108.

S108 produces a word retained after shifting left by 27. S109 receives zero, zero, that shifted word, and zero. Shifting the retained word left another 24 bits yields zero under word semantics. This zero supplies T12 and S119’s first argument; S119’s second argument is also zero.

S115 and S121 independently produce the payloads for T11 and T13. Intervening calls remain ordered and potentially effectful even where their results are unused.

### Final argument construction

S123 receives three zeros and produces a word with several later uses. Its shift-left-one form supplies T14 and S124’s first argument. S124 additionally receives zero, the S123 word shifted left by four, and zero.

S124’s second argument is zero because it masks the low 23 bits of the retained S108-derived shift-by-27 word before shifting left by four. The mask removes every potentially set bit.

The S123 shift-left-four value is retained. Two zero-payload transfers follow S126. S127 then receives zero. Before S128, T17 transfers the retained value shifted left another eight bits.

S128 receives the S123 word shifted left by 12, the S123 word shifted left by 16, and the S127 result shifted left by nine.

S129 produces a 64-bit value. S130 receives its low half, high half, and low half shifted left by one.

The retained S123 shift-left-four value has four clear low bits. Masking it with three therefore produces zero. Further represented shifts preserve that zero, supplying S131’s argument and later S139’s second argument. S131’s word result passes through S132 to S133. S138 independently produces S139’s first argument.

### Final destination and endpoint

S135 produces a 64-bit value. S136 receives its low half, high half, and high half shifted left by 11, then produces another 64-bit value.

Before S137, the unsigned S136 result is logically shifted right by 41 and narrowed to 32 bits. The retained quantity is at most 23 bits wide. S137 separately receives the result’s low half.

After S140, the retained quantity is shifted right by 11. Although represented using a signed intermediate at this point, it is nonnegative, so arithmetic and logical right shifts agree.

Let \(q\) be the resulting 12-bit quantity: bits 52–63 of the S136 result. The destination address is

\[
a=q+\left\lfloor q/4\right\rfloor,\qquad 0\leq q\leq4095.
\]

No allocation base or buffer pointer is added. Generated addresses range from zero to 5,118 inclusive, with gaps.

The function attempts a 16-bit write of 00CD at that destination. The representation then ends at an invalid-instruction marker. It establishes neither a normal return nor a particular exception, signal, or successful status.

## Side effects

### Coprocessor transfers

Each transfer has three numerical selectors, a 32-bit payload, and two context operands. Every final context operand is \(K_0\).

“After” means after the indicated call and before the next call. Where two transfers share an insertion point, table order is significant.

| Transfer | After | Selectors | Payload | Context operands |
|---|---|---|---|---|
| T01 | S003 | \(0,0,7\) | S003 word result | \(K_1,K_0\) |
| T02 | S014 | \(0,4,7\) | S014 word result | \(K_1,K_0\) |
| T03 | S025 | \(0,1,7\) | S025 word result | \(K_2,K_0\) |
| T04 | S034 | \(0,6,7\) | Locally derived zero | \(K_3,K_0\) |
| T05 | S053 | \(0,5,7\) | S053 word result | \(K_0,K_0\) |
| T06 | S058 | \(0,4,7\) | S058 low half | \(K_4,K_0\) |
| T07 | S078 | \(0,6,7\) | S078 word result | \(K_5,K_0\) |
| T08 | S105 | \(0,6,7\) | S105 low half | \(K_5,K_0\) |
| T09 | S107, first | \(0,3,7\) | S107 word result | \(K_3,K_0\) |
| T10 | S107, second | \(0,3,7\) | S107 word result | \(K_0,K_0\) |
| T11 | S115 | \(0,1,7\) | S115 word result | \(K_1,K_0\) |
| T12 | S118 | \(0,5,7\) | S108-derived zero | \(K_6,K_0\) |
| T13 | S121 | \(0,6,7\) | S121 word result | \(K_7,K_0\) |
| T14 | S123 | \(0,5,7\) | S123 word shifted left by one | \(K_2,K_0\) |
| T15 | S126, first | \(0,7,7\) | Zero | \(K_7,K_0\) |
| T16 | S126, second | \(0,0,7\) | Zero | \(K_0,K_0\) |
| T17 | S127 | \(0,2,7\) | S123 word shifted left by 12 | \(K_7,K_0\) |

Context-sharing relationships are established; their hardware meanings are not. T09 and T10 remain separate ordered operations even if their context operands happen to have equal values.

### Memory, ownership, and errors

The final halfword store is the only explicit ordinary-memory write. Its byte order, alignment legality, mapping, and possible device significance are unknown. Zero and odd destination addresses are reachable; no null or alignment guard is represented.

No allocation, release, ownership transfer, reference counting, or cleanup obligation is established. Word-sized values are not classified as owned pointers merely because their bit patterns could encode addresses.

Unknown dependencies may read or modify globals, retain arguments, allocate or release resources, alter execution state, trap, or fail to return. No ownership or preservation contract is inferred for them.

There is no represented status check, conditional recovery, retry, rollback, or cleanup path. A normally returning dependency result is used or ignored according to the represented dependencies; it is not checked for a failure indication. Such an indication does not by itself suppress later calls, transfers, or the final store. When consumed, its bits can still affect later arguments or the destination address.

A nonreturning call, trap, or other interruption of control flow prevents subsequent operations from being reached. Earlier effects need not be undone.

## Dependencies

Three dependencies have supplied opaque identities:

| Dependency | Established occurrence and use | Unresolved contract |
|---|---|---|
| F-ea490577ce35 | S009; no explicit arguments shown | Implicit inputs, effects, preservation, termination |
| F-0e02e4c13579 | S065; receives S064’s word; an associated auxiliary output is consumed afterward | Argument meaning, auxiliary-output ABI, ownership, effects |
| F-3de949054bdd | S098; no explicit arguments shown | Implicit inputs, effects, preservation, termination |

Constant target expressions establish seven additional same-target relationships:

| Same-target occurrences | Represented usage |
|---|---|
| S003 / S022 | S003 has one word argument and a consumed 32-bit result; S022 has no explicit arguments and a consumed 64-bit result |
| S044 / S057 | Neither has explicit arguments; only S044 has a consumed ordinary result |
| S052 / S099 | S052 has four explicit arguments; S099 has none shown and produces a consumed word |
| S070 / S108 | Neither has explicit arguments; only S108 has a consumed ordinary result |
| S107 / S135 | Neither has explicit arguments; consumed result widths are 32 and 64 bits respectively |
| S116 / S126 | Neither has explicit arguments or a consumed ordinary result |
| S132 / S137 | Each has one word argument; only S132 has a consumed ordinary result |

These equalities do not establish actual signatures. Differing represented result widths or argument lists remain ABI questions; they do not justify treating an established common target as different functions.

No further opaque dependency identities are supplied here. Other call occurrences must not be presumed to have distinct targets.

## Acceptance examples

These are conditional static deductions, not executed tests.

1. **Entry-byte isolation.** Changing only bits above bit seven of \(G\) leaves S002’s fourth argument unchanged. A low byte of 255 produces 3FC00000.

2. **Word-width boundary.** If the S022 high half is 16, successive word shifts left by 19 and nine produce zero for S030. A widened intermediate would not represent the same arithmetic.

3. **Masked field.** If the S047 low half is FFFFFFFF, S052’s third argument is 003FFFF0.

4. **Auxiliary output.** If the auxiliary output associated with S034 is one, S035 receives zero, 128, zero, and zero. Replacing its second argument with zero would discard an established dependency.

5. **Signed-shift distinction.** If S094 produces 04000000, its shift-left-five intermediate is 80000000. S103’s second argument is FC000000 with arithmetic right shifting or 04000000 with logical right shifting. Actual instruction interpretation remains `documented_only`.

6. **Common target, differing interface representation.** S003 and S022 retain their target equality despite their different consumed result widths. Neither a common 32-bit nor a common 64-bit signature is established.

7. **Common target, differing argument lists.** S052 and S099 target the same destination. Their differing explicit argument lists do not establish that S052’s arguments are ignored.

8. **Transfer ordering.** T09 precedes T10, and both precede S108. Equal context values do not authorize coalescing these transfers.

9. **Returning failure indication.** If S054 returns normally with a value that its own contract interprets as failure, this function does not inspect that value. Provided control flow remains uninterrupted, S055 and subsequent operations are still reached.

10. **Nonreturning dependency.** If S054 does not return, S055–S140, T06–T17, and the final store are not reached. T05 has already occurred; no rollback is represented.

11. **Address boundaries.** For \(q=0,3,4,4095\), destination addresses are respectively \(0,3,5,5118\). There is no base addition, null guard, or rounding-up division.

12. **Store interruption.** Computing destination 5,118 does not establish writable memory there. If the attempted store faults, that interruption occurs after the preceding represented calls and transfers; no invalid-instruction endpoint is then established as reached.

## Uncertainties

The following remain `documented_only`:

- Whether the representation corresponds to valid executable instructions.
- Actual calling conventions, implicit inputs, and auxiliary-output validity.
- Whether represented 64-bit results are genuine paired returns.
- Actual signatures where a common target has differing represented arguments or result widths.
- Machine interpretation of signed operations and overflow in the first target calculation.
- Dependency semantics, including failure encodings, ownership, preservation, and termination.
- Coprocessor selection, synchronization, privilege requirements, and faults.
- Physical register-read timing across calls.
- Final-store mapping, alignment legality, byte order, and external significance.
- The actual outcome at the invalid-instruction endpoint.

The seven target equalities, transfer insertion points, context-sharing relationships, and absence of represented dependency-result checks are established static facts, not uncertainties about the supplied representation.

## Evidence

The supplied static representation establishes 140 call occurrences, five consumed auxiliary outputs, eight context inputs, 17 coprocessor transfers, the described paired-result dependencies, one final 16-bit store, and an invalid-instruction endpoint.

Evaluation of constant target expressions establishes the seven same-target pairs. Those equalities do not depend on assumptions about dependency behavior.

Zero reductions, field widths, and destination bounds are deductions conditional on the stated word-operation semantics. The represented sequence contains no result-based failure branch: interruption of control flow, rather than a normally returned failure-indicating value alone, is what prevents subsequent operations from being reached.

No dependency bodies, validated instruction stream, runtime tests, or hardware observations were available.
