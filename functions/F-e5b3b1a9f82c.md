<!-- Trusted provenance: F-e5b3b1a9f82c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5b3b1a9f82c

## Purpose

F-e5b3b1a9f82c performs state-dependent memory writes, indirect calls, and recurring processing. The supplied static representation does not establish an application-level purpose, a conventional success condition, or a normal return path.

Its defensible contract is an ordered description of represented effects, conditional on memory accesses succeeding and dependencies returning. Writes include small numeric addresses, fixed locations, computed addresses, and a location associated with another function. These are actual represented accesses, not validation checks.

The apparent control flow either repeats outer processing, enters a recurring call-and-write phase, faults, encounters a breakpoint, or fails to return from a dependency. No ordinary completion is shown.

## Interface

### Explicit arguments and result

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Added to the incoming base to initialize the current address-like state. |
| Second | 32 bits | No explicit use shown. |
| Third | 32 bits | No explicit use shown. |
| Fourth | 16 bits | Written at incoming base plus 14 bytes. |

No result is declared, and no normal return is visible. Absence of explicit uses for the second and third arguments does not prove machine-level irrelevance: dependencies may observe state outside their displayed arguments.

### Additional state

The representation also consumes incoming state for:

- A working base.
- A stored 32-bit value.
- A word-oriented cursor.
- A comparison pointer.
- A return-address-like value.
- An increment for an address-like accumulator.

Several additional state roles develop during execution: a current address-like value, a working transfer address, an earlier loaded address, a retained word, and a represented 64-bit dependency result.

These roles describe mutable state, not immutable entry parameters. In particular, subsequent outer iterations use the current base, stored value, current address, cursor, and return-address-like state.

### Widths, layout, and preconditions

Explicit memory operations distinguish 8-, 16-, 32-, and 64-bit accesses. Word-pointer indexing advances by four bytes; indexing the 64-bit storage pointer by one advances by eight bytes.

The low and high halves of a represented 64-bit result are numeric 32-bit halves. This does not establish memory byte order.

Address calculations appear to use a 32-bit address model. Exact overflow and oversized-shift semantics are not fully established. Modular examples below are explicitly conditional on 32-bit wrapping arithmetic.

Execution requires each reached access to be valid for its width and the relevant execution environment. No alignment, address-map, privilege, executable-memory, exception, or dependency-preservation guarantees are supplied. Global declarations are absent, so assignments to named global locations do not independently establish access widths.

## Behavior

### Entry and outer processing

The first represented memory effect is a 16-bit write of the fourth argument at incoming base plus 14. The current address is then initialized to incoming base plus the first argument.

Each outer iteration performs the following ordered work:

1. Compute and save the predecessor cursor, four bytes before the current cursor.
2. Write the current stored 32-bit value at that predecessor.
3. Multiply the current address by eight and write that 32-bit product at the address numerically equal to the product.
4. Write a 32-bit zero at an address formed by shifting the current address left by 17, then right by 32, then adding 16.
5. Write the halfword value 0xc470 at a computed address.
6. Assign 0x8000 to the global at 0xfe238028.
7. Replace the current address with 0x0897f228.
8. Assign 0x225fc470 to the global at 0x225fc470.
9. Reset the working displacement to zero.
10. Assign 0x225f to the global at 0x0000229f.
11. Read the global at 0x00000022, shift its represented value right by 16, and write the result as a word at current cursor plus 84.
12. Conditionally invoke D-01.

The computed halfword destination in step 5 is:

- The current stored value logically shifted right by four;
- Minus sixteen times the current base;
- Minus one if bit three of the stored value is clear, otherwise minus zero;
- Plus 38 bytes.

The shift-by-32 expression in step 4 is unresolved; it must not be silently simplified to zero or assigned a particular processor’s masked-shift behavior.

D-01 is skipped exactly when the comparison pointer equals current cursor plus 20 bytes. Otherwise, its target is the current return-address-like state plus 46. The target is formed before that state is replaced with 0x0897f193.

Only the first outer iteration necessarily uses the entry-derived base, stored value, and address. Later iterations use state replacements described below. The return-address-like state is likewise not necessarily its incoming value.

### Inner processing and cursor transition

After the optional call, the represented setup:

- Resets the accumulator to zero.
- Replaces the working base with 0x0897f1f8.
- Forms a 64-bit value with high half 0x0897f1f8 and low half equal to the zero displacement.
- Forms an inner write-address value from the current address plus the displacement plus five.

Each inner iteration then:

1. Writes the inner write-address value as a word at the low half of the current 64-bit value plus 12.
2. Interprets that low half as signed 32-bit and arithmetic-right-shifts it by a count taken from its low byte.
3. Computes signed 32-bit subtraction overflow for subtracting one from the shifted result.
4. Reads a word at twice the shifted result.
5. Replaces the stored 32-bit state with 0x0897f1ec.
6. Writes the low 16 bits of the word read to the location associated with F-422afe36d401.
7. Writes 0x0897f1f8 as a word at the read value plus four.
8. Assigns the saved predecessor cursor to the current cursor.

The predecessor is computed once per outer iteration. Repeating the inner loop reassigns that same predecessor; it does not decrement the cursor again. A further four-byte decrement is introduced only when another outer iteration computes a new predecessor.

For the literal initial inner setup, the low half is zero. The function therefore attempts a word write at address 12 before a word read at address zero. The derived signed value is zero, so subtracting one does not overflow.

If the subtraction-overflow predicate is true, inner processing ends and another outer iteration begins. That outer iteration sees the replacement base 0x0897f1f8 and stored value 0x0897f1ec, subject to any effects not captured by the scalar representation.

Otherwise, the function:

1. Writes byte 0xd8 at the high-half address of the current 64-bit value.
2. Assigns a supervisor-call-associated address to the working transfer-address state. This assignment is not itself a supervisor instruction.
3. Forms D-02’s target from the current return-address-like state plus 0xe56.
4. Replaces that return-address-like state with 0x0897f1d7.
5. Invokes D-02 with explicit argument zero and captures its represented 64-bit result.
6. Adds the incoming increment to the accumulator.
7. Reads the global at 0x0897f216, converts its value to an unsigned address-like value, and replaces the earlier loaded address with it.
8. Reads a byte at D-02’s low result half plus 0x0897f1f8 and replaces the current address with that zero-extended byte.

The access width of the global read at 0x0897f216 is unresolved. Its name and unsigned conversion do not establish a 16-bit read.

A subsequent equality compares zero with the false overflow predicate required to reach D-02. In the literal represented scalar flow it succeeds. On that path, a 16-bit unsigned read at the inner write-address value plus D-02’s low half supplies the destination of a word write containing the accumulator. The destination is therefore in the numeric range 0–65,535.

Execution then enters the repeated phase with explicit argument zero. If the equality were not satisfied because the representation omitted relevant machine-state behavior, inner processing would repeat using the current 64-bit result and the same saved predecessor.

### Repeated phase: initial effects

Each entry captures the low byte of the current address. D-03 is then called with the phase’s current 32-bit argument.

Its represented 64-bit result is split into low and high halves. The low half is multiplied by 32, and that 32-bit product is written at the address equal to the product. Only afterward is the product compared, unsigned, with 20.

If the product is below 20, the continuation argument is zero and processing proceeds directly to the common continuation. The working base, accumulator, and working transfer address are not replaced by the skipped larger-product path.

Under wrapping 32-bit multiplication, only product zero can satisfy this comparison, because a multiple of 32 has its low five bits clear. This path still attempts its self-addressed word write at zero.

### Repeated phase: larger-product path

For a product at least 20, operations occur in this order:

1. Write the current base as a word at product plus 88.
2. Write the low 16 bits of the earlier loaded address at base plus 14.
3. Replace the working transfer address with the base shifted left by two.
4. Assign the captured byte to the reset-associated global.
5. Invoke D-04 with the earlier loaded address logically shifted right by two.
6. Invoke D-05 and split its represented 64-bit result.

Using D-05’s low and high halves as addresses, the next effects are:

1. Write the captured byte at twice the low-half address.
2. Write the low byte of the high half at low-half address plus eight.
3. Write D-03’s high-half address as a word at twice the current accumulator.
4. Write the complete 64-bit D-05 result at its low-half address.

These accesses may overlap. In particular, the full-width store can overwrite an earlier byte store for some addresses. Their order is part of the contract; they are not independent structure fields.

The following state replacements and writes then occur:

- Replace the accumulator with four times D-03’s high-half address.
- Replace the working base with the earlier loaded address shifted left by eight.
- Through D-03’s high-half address, write three consecutive words: the working transfer address, the replacement base, and the earlier loaded address.

A derived unsigned value is then formed by shifting the earlier loaded address, interpreted as signed 32-bit, left by 31 and right by 32, converting to unsigned, and logically shifting right by 20. The shift-by-32 semantics are unresolved.

If that derived value is zero:

1. Assign zero to the reset-associated global.
2. Add the working transfer address to the replacement accumulator.
3. Write that sum as a word through D-05’s high-half address.
4. Select continuation argument zero.

If the derived value is nonzero:

1. Write it as a word at working transfer address multiplied by 256, plus 32.
2. Select the continuation argument by interpreting the 32-bit product of working transfer address and 256 as signed, then arithmetic-right-shifting it by 29.
3. Encounter a breakpoint with immediate value four.
4. If execution resumes, replace the working base with D-05’s high-half address interpreted as signed 32-bit and arithmetic-right-shifted by 18.

The breakpoint is not an ordinary error return. Subsequent effects require resumed execution.

### Common continuation and carried state

D-06 is invoked at the branch-dependent target listed below, with the selected continuation argument. Its represented 64-bit result drives these operations:

1. Write its low 16 bits at its low-half address.
2. Multiply the current base by 64 and write that word at the address equal to the product.
3. Read a word through the earlier loaded address and retain it.
4. Write a halfword at current base plus four, containing the earlier loaded address plus four truncated to 16 bits.
5. Read a pointer-like word at D-06’s high-half address plus 80 and replace the earlier loaded address with it.
6. Invoke D-07, then D-08.
7. Replace the current address with the working transfer address retained from the preceding phase.
8. Invoke D-09 with the retained word from step 3.
9. Form the next repeated-phase argument by logically shifting that retained word right by three.
10. Replace the working transfer address with the retained word.
11. Re-enter the repeated phase.

The next captured byte consequently comes from the preceding working transfer address, not from the newly retained word. The retained word instead becomes the new working transfer address after D-09.

The earlier loaded address, base, accumulator, current address, and working transfer address carry their current values into recurrence unless explicitly replaced. No normal exit is shown.

## Side effects

Established side effects include memory writes of several widths, fixed-global assignments, dependency-associated storage modification, indirect calls, a conditional breakpoint, and cursor movement across outer iterations.

There is no established allocation, release, ownership transfer, lock discipline, rollback, or error-reporting protocol. Dependency results used as addresses are not thereby proven owned, newly allocated, writable, aligned, or persistent.

Effects before a fault, trap, or non-returning call remain observable unless the execution environment provides behavior not shown here. Later effects are conditional on all preceding operations completing.

## Dependencies

The following opaque IDs identify call sites, not necessarily distinct implementations.

| ID | Target and represented call state |
|---|---|
| D-01 | Current return-address-like state plus 0x2e; that state is replaced with 0x0897f193 before invocation. No explicit arguments shown. |
| D-02 | Current return-address-like state plus 0xe56; that state is replaced with 0x0897f1d7 before invocation. Explicit argument zero; represented 64-bit result. |
| D-03 | Fixed target 0x07f755f6. Receives the current repeated-phase argument; represented 64-bit result. |
| D-04 | Target 0x0897edf1 plus 0xae, numerically 0x0897ee9f. Receives the earlier loaded address logically shifted right by two. |
| D-05 | Code-location anchor plus 0xffe. The anchor’s authoritative identity and address are unresolved. No explicit arguments shown; represented 64-bit result. |
| D-06 | Below-20 path: 0x0897edf1 plus 0xa1e = 0x0897f80f. Zero-derived larger-product path: 0x0897ee1d plus 0xa1e = 0x0897f83b. Nonzero-derived path: 0x08790d52 plus 0xa1e = 0x08791770. Receives the selected argument; represented 64-bit result. |
| D-07 | Target 0x0897edcd plus 0x3c4 = 0x0897f191. No explicit arguments shown. |
| D-08 | Target 0x0897eddf plus 0xffe = 0x0897fddd. No explicit arguments shown. |
| D-09 | Target 0x0897ede1 plus 0x800 = 0x0897f5e1. Receives the retained word. |

F-422afe36d401 identifies the location receiving the inner halfword write. No invocation of that function is shown.

No dependency has an established memory, register, global-preservation, ownership, or error contract. “No explicit arguments shown” does not mean a proven zero-argument machine interface. The scalar state transitions above describe the supplied representation, not an independently verified machine ABI.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Entry ordering.** Given valid storage, the fourth argument’s halfword is written at incoming base plus 14 before the predecessor-cursor word write.

2. **Outer-state replacement.** If execution reaches another outer iteration after inner processing, its stored-word write uses 0x0897f1ec and its base-dependent calculation uses 0x0897f1f8 in the represented scalar flow, rather than unconditionally reusing entry values.

3. **Saved predecessor.** For an outer iteration beginning with cursor 0x1000, its saved predecessor is 0x0ffc. Multiple inner iterations assign 0x0ffc, not successively 0x0ff8 and 0x0ff4. A subsequent outer iteration computes 0x0ff8.

4. **Initial low-address effects.** With the literal initial inner setup, the word write at 12 precedes the word read at zero. Neither access may be silently omitted.

5. **Dependency-associated write.** If the inner word read yields 0x12345678, the halfword written at F-422afe36d401’s associated location is 0x5678. No call is implied.

6. **Zero product.** If D-03’s low half produces scaled value zero, zero is written at address zero before continuation argument zero is selected.

7. **Replacement accumulator.** On the larger-product path, if D-03’s high half is 0x1000 and the working transfer address is 0x2000, the replacement accumulator is 0x4000. On the zero-derived branch, 0x6000 is stored through D-05’s high-half address, assuming these operations complete without overflow.

8. **Signed continuation argument.** If the nonzero branch’s 32-bit product by 256 is 0x80000000, arithmetic right shift by 29 yields signed −4. If the breakpoint resumes and D-05’s high half is 0x80000000, the replacement base is signed −8192.

9. **Recurrence byte.** If the preceding working transfer address ends in 0xab and the retained word ends in 0xcd, the next repeated phase captures 0xab. The retained word becomes the working transfer address only after D-09.

## Uncertainties

### documented_only

Unresolved behavior includes:

- Complete calling convention and incoming machine-state validity.
- Exact oversized-shift, overflow, and signed-left-shift semantics.
- Global access widths, including the read at 0x0897f216.
- Endianness, alignment, address validity, and executable-memory behavior.
- D-05’s code-location anchor identity.
- Dependency preservation, side effects, ownership, and return behavior.
- Reachability of paths controlled by ambiguous arithmetic.
- Breakpoint resumption and exception handling.
- Any externally intended success condition.

### Inferred intent

The supervisor-call-associated address, reset-associated global, breakpoint, and code-associated writes suggest low-level control or memory manipulation. They do not establish a device protocol, initialization routine, service, or meaningful successful outcome.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping. Established evidence comprises explicit access widths, numeric constants, assignment order, target arithmetic, argument expressions, result consumption, branch predicates, and recurring control flow.

The supplied warnings report unresolved type propagation and overlapping global symbols. Consequently, symbolic names and apparent types are not independent proof of access width, ownership, or operational purpose.

No runtime tests, memory observations, instruction-level validation, or dependency implementations were available. Acceptance examples state conditional obligations of the represented behavior, not verified executions.
