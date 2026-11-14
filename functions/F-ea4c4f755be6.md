<!-- Trusted provenance: F-ea4c4f755be6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea4c4f755be6

## Purpose

F-ea4c4f755be6 describes a long, apparently straight-line sequence of indirect dependency calls, transfers into coprocessor state, transformations of 32-bit and 64-bit results, and a final 16-bit memory store. The supplied representation ends at an invalid-instruction condition rather than a normal return.

The application-level purpose is **documented_only**. The evidence does not establish that the routine initializes a device, decodes data, updates a particular object, or implements a recognizable protocol. Coprocessor operations suggest interaction with architectural state, but their meaning is unknown.

This contract separates:

- **Established static evidence:** operations, explicit data dependencies, and ordering visible in the supplied representation.
- **Inferred interpretation:** fixed-width bit-vector arithmetic consistent with the represented widths.
- **Unknown behavior:** dependency effects, architectural interpretation, actual reachability, and behavior beyond the invalid-instruction boundary.

All execution expectations below are conditional on the recovered control flow being accurate and on each preceding operation completing and returning where required.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits, otherwise untyped | Passed unchanged to the first indirect dependency. |
| Second argument | 32 bits, otherwise untyped | No explicit use is visible. |
| Third argument | 32 bits, represented as signed | Shifted left by 11 bit positions and passed as one argument to an early indirect dependency. |

No explicit input is established as a pointer, buffer, length, handle, or owned resource. In particular, the first argument’s passage to a dependency does not establish its interpretation or accessibility requirements.

The absence of a visible use of the second argument does not establish that its incoming machine register is irrelevant to all dependencies. The recovered call signatures and calling convention are incomplete.

### Implicit execution inputs

The routine also depends on ambient machine state:

- An incoming link or return-address register participates in computing the first indirect call target.
- Several coprocessor transfers contain incoming architectural-register values as operands.
- Several calls consume apparent secondary register outputs whose producers and contractual validity are not established.
- Indirect targets must resolve to executable operations with compatible calling behavior for execution to continue.

These are not additional well-defined API parameters. They are unresolved execution prerequisites.

### Outputs and completion

The represented interface has no return value. No successful return is visible.

The last represented actions are:

1. A 16-bit store of the value `0x00CD` to a computed address.
2. An invalid-instruction termination marker.

A caller cannot rely on normal completion, a status code, or a recoverable error result from this evidence.

### Width conventions

The represented scalar working values are 32 bits. Some indirect calls supply 64-bit results, subsequently separated into a low 32-bit word and a high 32-bit word.

For the arithmetic discussion below, a left shift means a 32-bit bit-vector shift with bits beyond bit 31 discarded, unless a 64-bit extraction is explicitly stated. This is an inferred machine-level interpretation, not a claim that every signed expression in the supplied representation has portable language-level semantics.

The evidence does not establish memory endianness, pointer provenance, or alignment policy.

## Behavior

### Execution structure

The visible body contains no conditional branch or loop. Its recovered path is a single ordered sequence.

That does not make the routine unconditionally executable. Any dependency, indirect transfer, coprocessor operation, or memory access may fault, fail to return, or alter execution in ways not described here. Later effects occur only if execution reaches them.

Calls whose results are not used explicitly remain observable dependency invocations. They cannot be removed merely because the local representation discards their results.

### Entry and early argument processing

The first indirect dependency receives the first explicit argument. Its 32-bit result is transferred into coprocessor state before subsequent calls.

Several indirect calls then precede the direct invocation of **F-ea490577ce35**.

A later indirect call produces a 32-bit result. Another call receives four arguments derived from that result and the third explicit input:

- The result unchanged.
- The result shifted left by 17.
- The third input shifted left by 11.
- The result shifted left by 27.

There is no visible range check, saturation, validation, or error branch around these transformations.

Further calls produce a value for a second coprocessor transfer.

### Early wide-result processing

An indirect dependency produces a 64-bit result. Its low word passes through three successive result-transforming calls; the final 32-bit value enters coprocessor state.

The high word of that same wide result is retained separately. It is shifted left by 19 and later by another 9 positions before being supplied to a call. Under the 32-bit interpretation, this preserves only the original high word’s lowest four bits, positioned in bits 28 through 31.

An intervening 64-bit result supplies:

- Its low word unchanged.
- Its high word shifted left by 17.

Later calls occur before the retained high-word-derived argument is consumed. The transformed return value from that consumption passes through another call and then into a further dependency.

### Zero-valued arguments and secondary register outputs

The routine explicitly introduces a zero working value. A dependency receives a preceding result followed by two zero arguments.

Subsequent operations include:

- A coprocessor transfer carrying zero.
- A four-argument call with zero in its first, third, and fourth positions.
- A second argument obtained by shifting an apparent secondary register output left by 7.

That secondary value is not established as zero and is not interchangeable with the explicit zeros.

Another dependency receives zero and returns a value that is passed to a later call alongside zero. Additional calls then produce a 32-bit value supplied both unchanged and shifted left by 20.

### Masked fields from wide results

A later 64-bit result is consumed in several ways:

- Its low word is passed unchanged to a dependency.
- Its low word shifted left by 14 is passed to another dependency.
- Its high word is shifted left by 9 and subsequently by 14 more positions.
- The resulting high-word-derived value is used in two calls.
- One call also receives an apparent secondary register output.
- Another receives four arguments: zero, zero, the low word masked to 18 bits and shifted left by 4, and zero.

The masked low-word argument ranges from zero through `0x003FFFF0`, inclusive, in multiples of 16.

A subsequent returned value enters coprocessor state. Later, the low word of another wide result enters coprocessor state before that same wide result is consumed by a dependency together with its high word shifted left by 20.

The resulting 64-bit value then supplies three arguments to another call: its low word, its high word, and its high word shifted left by 12.

### Middle direct dependency and retained values

Following additional calls, a 32-bit result is retained in two forms: unchanged and shifted left by 11. The unchanged form is supplied to **F-0e02e4c13579**.

Later:

- A call receives zero, an apparent secondary register output, and zero.
- A wide result is consumed with its low word shifted left by 4, its high word unchanged, and the retained shifted value shifted left by another 17 positions.
- Under the 32-bit interpretation, that last argument is the earlier 32-bit result shifted left by a total of 28.

Another working value is explicitly initialized to zero. Its right-shifted and subsequently left-shifted derivatives therefore remain zero. These derivatives are passed through several later argument positions.

One call receives four arguments consisting of:

- The retained value shifted left by a further 31 positions.
- Another apparent secondary register output.
- A zero-derived value.
- Another zero-derived value.

Because the retained value already resulted from an 11-position left shift, the first argument here is zero under the fixed-width interpretation. The secondary register argument remains unknown.

The returned value enters coprocessor state and is then supplied to another dependency.

### Later scalar processing and the third direct dependency

A further group of calls includes a call receiving zero and another receiving zero, a secondary register output, and zero.

A later 32-bit result is passed to a three-argument dependency as:

- The result unchanged.
- A zero-derived value.
- The result shifted left by 5.

After more indirect calls, **F-3de949054bdd** is invoked.

Subsequent processing uses the retained shift-by-5 value shifted right by 5. The representation treats this as a signed value. Under a conventional arithmetic-right-shift interpretation, it is not a general inverse of the preceding left shift: high bits discarded by the left shift are not recovered, and sign extension may occur. The exact signed-shift semantics are **documented_only**.

Another wide result supplies its low word to coprocessor state and its high word shifted left by 1 to a dependency. A later 32-bit result is transferred twice into coprocessor state, with distinct architectural operands and no intervening dependency call.

### Final field construction

A returned 32-bit value is shifted left by 27 and retained. A following dependency receives zero, zero, that shifted value, and zero.

Later uses of the retained value include:

- A further left shift by 24, which is zero under 32-bit truncation.
- Masking to the low 23 bits and shifting left by 4, which is also zero because the earlier shift by 27 left those low 23 bits clear.

The zero produced by the first operation is transferred into coprocessor state and passed to a dependency alongside zero.

Another dependency receives three explicit zero arguments and returns a new 32-bit value. This result supplies:

- Its value shifted left by 1 to coprocessor state.
- A four-argument dependency call containing that shift-by-1 value, the zero masked-field result described above, the new result shifted left by 4, and zero.

The shift-by-4 result is retained. Its low two bits are necessarily clear, so masking it with `3` produces zero. Subsequent shifts of that masked quantity remain zero.

Two explicit zero-valued coprocessor transfers occur before a dependency called with zero returns another value. A later coprocessor transfer carries the retained shift-by-4 value shifted left by another 8 positions.

A three-argument dependency then receives:

- The retained shift-by-4 value shifted left by 8.
- That same retained value shifted left by 12.
- The result of the zero-argument-value call shifted left by 9.

Another wide result supplies its low word, high word, and low word shifted left by 1 to a dependency.

Further result-transforming calls occur, including one whose explicit input is the zero derived from the low-two-bit mask.

### Terminal address and store

Near the end, a dependency produces a 64-bit result. Another dependency consumes its low word, high word, and high word shifted left by 11, producing a new 64-bit result.

That latter result determines the eventual store address:

1. Its bits 41 through 63 form an unsigned 23-bit quantity.
2. Shifting that quantity right by 11 yields a 12-bit quantity, ranging from 0 through 4095.
3. The destination address is that 12-bit quantity plus its integer quotient by 4.

Consequently, the represented destination address lies between `0x00000000` and `0x000013FE`, inclusive. Not every address within that interval is attainable.

The low word of the same wide result is supplied to a dependency before the store. A further dependency returns a value consumed by another call alongside a zero-valued argument. One final indirect call occurs before the write.

The routine then attempts to write the 16-bit value `0x00CD` at the computed address. No bounds check, null check, alignment adjustment, or visible validation precedes the write.

The invalid-instruction termination marker follows the store in the recovered order.

## Side effects

### Coprocessor state

Seventeen coprocessor-transfer operations are visible. In order, their payloads are:

1. The first indirect call’s result.
2. A later indirect result.
3. The result of the early three-call transformation chain.
4. Zero.
5. A later indirect result.
6. The low word of a wide result.
7. A middle indirect result.
8. The low word of a later wide result.
9. A later scalar result.
10. The same scalar result as the preceding transfer, with different architectural operands.
11. A later indirect result.
12. Zero derived from the retained shift-by-27 value.
13. A later indirect result.
14. A returned scalar shifted left by 1.
15. Explicit zero.
16. Explicit zero.
17. The retained shift-by-4 result shifted left by another 8 positions.

All are represented as transfers to coprocessor number zero. Their selector fields and incoming architectural operands vary. This evidence does not identify the affected hardware facilities, synchronization effects, privilege requirements, or trapping conditions.

### Memory and ownership

The final 16-bit store is the only explicit data-memory access visible in the body. Dependencies may perform arbitrary additional reads or writes.

No allocation, release, ownership transfer, or cleanup protocol is established. A dependency result must not be classified as owned memory merely because it is forwarded elsewhere.

No rollback is visible. If an operation fails after earlier calls or coprocessor transfers, those prior effects are not shown being undone.

## Dependencies

| Opaque ID | Established relationship |
|---|---|
| **F-ea490577ce35** | Directly invoked once in the early portion, without visible explicit arguments or a consumed return value. |
| **F-0e02e4c13579** | Directly invoked once with a 32-bit result previously returned by an indirect dependency. |
| **F-3de949054bdd** | Directly invoked once in the later portion, without visible explicit arguments or a consumed return value. |

The remaining dependencies are indirect calls without supplied opaque identities. Their targets are represented using code-location-relative calculations, except for the first target, which depends on incoming link-register state.

Their signatures, memory effects, register effects, failure behavior, and ownership rules are unknown. A call represented with no explicit arguments is not proven independent of ambient machine state. No dependency is assumed to preserve globals, architectural state, or memory.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

- **Third-input truncation:** If the third input is `0x00200001`, the early shift-by-11 argument is `0x00000800` under 32-bit truncation. It is not a widened mathematical product.

- **High-word narrowing:** If the first retained wide result has high word `0x00000011`, the argument formed by successive shifts of 19 and 9 positions is `0x10000000`. Only the original low four bits contribute.

- **18-bit mask:** If the relevant wide result’s low word is `0xFFFFFFFF`, the masked-and-shifted argument is `0x003FFFF0`, regardless of the discarded upper 14 bits.

- **Retained top-bit field:** If the value shifted left by 27 is nonzero, its later shift left by 24 is nevertheless zero under the fixed-width interpretation. Its low-23-bit mask is also zero.

- **Terminal address zero:** If bits 52 through 63 of the final address-producing wide result are all zero, the attempted store address is zero. No visible guard excludes that destination.

- **Terminal address boundary:** If those 12 bits are all one, the address is 4095 plus 1023, or `0x13FE`.

- **Potential misalignment:** If those 12 bits encode one, the attempted store address is one. Acceptance must not assume that every destination is halfword-aligned or that such a store succeeds.

- **Earlier failure:** If a dependency does not return, subsequent coprocessor transfers and the terminal store are not required to occur.

- **Terminal fault:** If the store faults, the invalid-instruction boundary need not be reached. If the store succeeds, the evidence still provides no normal-return behavior.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered instruction stream and straight-line control flow are accurate.
- The application-level purpose.
- The complete calling convention and true signatures of indirect dependencies.
- The provenance of apparent secondary register outputs.
- Coprocessor operand meanings, privilege requirements, and fault behavior.
- Exact signed-shift semantics.
- Dependency preservation or modification of globals and architectural state.
- Memory mapping, endianness, and alignment behavior at the terminal destination.
- Whether the invalid-instruction marker denotes executed invalid data, a decoding boundary, or incorrectly recovered control flow.
- Any error handling, cleanup, or continuation outside the supplied representation.

These uncertainties prevent treating this document as proof of executable equivalence.

## Evidence

This contract is based solely on the supplied static representation and dependency-ID mapping. No implementation access, runtime execution, tests, dependency inspection, or independent instruction verification was available.

The explicit argument uses, call-result flow, coprocessor-transfer order, direct dependency order, terminal store width, and lack of a visible normal return are supported by that representation. Arithmetic reductions and address bounds are derived under the stated fixed-width interpretation.

The representation includes warnings about invalid instruction data and truncated control flow. Those warnings materially limit confidence in actual reachability and architectural behavior. No snapshot identity or independently verified provenance is asserted here.
