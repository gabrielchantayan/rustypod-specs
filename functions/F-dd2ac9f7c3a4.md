<!-- Trusted provenance: F-dd2ac9f7c3a4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd2ac9f7c3a4

## Purpose

F-dd2ac9f7c3a4 performs two byte-sized writes interleaved with an ordered sequence of fourteen indirect calls. The writes derive their addresses from an unsigned byte read through an implicit entry-state base. Their values come from different portions of the same explicit 32-bit input.

The routine also forwards selected dependency results into subsequent calls. One early result is passed onward unchanged as a 32-bit quantity; two later results are shifted left by two before forwarding.

No application-level purpose is established. The available representation ends at an invalid-instruction boundary, so it does not establish a successful return, a deliberate halt, or the complete behavior of the underlying executable.

This contract describes the visible static sequence. Every later operation is conditional on earlier memory accesses and calls completing and control continuing as represented.

## Interface

### Explicit input

The routine has one explicit unsigned 32-bit input, referred to here as the **input word**.

Its visible uses are:

- The low eight bits supply the first byte write.
- The full word supplies both the first and second arguments of the first indirect call.
- A left shift by two supplies that call’s third argument.
- Bits 22 through 29 supply the second byte write.

No range check, null check, or other validation of the input word is visible.

### Implicit entry state

The visible behavior additionally depends on:

- An **entry base**, carried in an incoming machine register but not represented as an explicit parameter.
- An **entry link value**, likewise obtained from incoming machine state.
- Readable memory at the entry base plus one.
- Writable memory at an address selected from a byte read at that location.
- Executable indirect-call targets and whatever state those targets require.

The supplied representation uses 32-bit integer and pointer-like quantities. The exact calling convention, register-preservation rules, instruction-set mode, and address-wrap behavior are not established.

The entry base is needed for two separate reads. The second read must not be replaced by reuse of the first read: intervening dependencies may change memory.

### Outputs

No explicit result is assigned to the caller. The represented interface has no return value, and no ordinary return is visible.

The observable outputs established by the body are:

- Two attempted one-byte writes.
- Fourteen ordered indirect-call attempts with the arguments described below.
- Forwarding of three dependency results.
- Arrival at a boundary beyond which the supplied representation does not establish valid control flow.

This is not evidence that the real routine always terminates at that boundary.

### Memory layout and address selection

The only directly described data location relative to the entry base is:

| Location | Access | Interpretation |
|---|---|---|
| Entry base plus one | Read, one byte, twice | Unsigned address selector |

Each selector is zero-extended and increased by hexadecimal `0x11`, or decimal 17. The result is used directly as a byte address.

Thus a selector from 0 through 255 selects an address from 17 through 272 inclusive. The selector is not multiplied by a stride, and the selected write address is not relative to the entry base.

Whether these low numerical addresses identify ordinary memory, mapped device registers, inaccessible memory, or something else is unknown.

## Behavior

### Dependency naming

The dependency IDs below are opaque, document-local identifiers for the fourteen call sites. They do not establish globally assigned identities, distinct implementations, or non-aliasing between targets.

### Ordered sequence

1. **Read the first selector.**  
   Read one unsigned byte from the entry base plus one.

2. **Perform the first write.**  
   Add 17 to that selector and attempt to write the low eight bits of the input word to the resulting address.

3. **Invoke F-000000000001.**  
   The target is derived from the incoming link value using a fixed displacement of hexadecimal `0xFEC`. Supply three arguments, in order:
   - The input word.
   - The same input word.
   - The input word shifted left by two.

   The third argument is a 32-bit unsigned result: bits shifted beyond bit 31 are discarded. No result from this call is visibly consumed.

4. **Invoke F-000000000002.**  
   Supply no explicit arguments. Capture its result as a 32-bit quantity.

5. **Invoke F-000000000003.**  
   Supply the captured result of F-000000000002 as its sole explicit argument. No intervening transformation of that result is visible.

6. **Invoke F-000000000004.**  
   Supply no explicit arguments.

7. **Invoke F-000000000005.**  
   Supply no explicit arguments.

8. **Invoke F-000000000006.**  
   Supply no explicit arguments.

9. **Invoke F-000000000007.**  
   Supply no explicit arguments. Capture its result as a 32-bit signed integer in the supplied representation.

10. **Read the second selector.**  
    Read one unsigned byte from the entry base plus one again. This occurs after F-000000000007 has returned, not before the preceding call sequence.

11. **Perform the second write.**  
    Add 17 to the second selector and attempt to write one byte whose value is the input word shifted right by 22, then reduced to its low eight bits.

    Because the input word is unsigned, the right shift is logical. The written value therefore consists of original input bits 22 through 29. Original bits 30 and 31 do not contribute to the stored byte.

12. **Invoke F-000000000008.**  
    Supply the result of F-000000000007 shifted left by two as the sole explicit argument. The signed-shift qualification below applies.

13. **Invoke F-000000000009.**  
    Supply no explicit arguments.

14. **Invoke F-00000000000a.**  
    Supply no explicit arguments.

15. **Invoke F-00000000000b.**  
    Supply no explicit arguments.

16. **Invoke F-00000000000c.**  
    Supply no explicit arguments. Capture its result as a 32-bit signed integer in the supplied representation.

17. **Invoke F-00000000000d.**  
    Supply two explicit arguments, in order:
    - The result of F-00000000000c shifted left by two, subject to the signed-shift qualification.
    - The second selector, zero-extended to an unsigned 32-bit quantity.

    The second argument is the selector captured at step 10. It is not a third read of entry-base memory, and it is not the selected write address.

18. **Invoke F-00000000000e.**  
    Supply no explicit arguments.

19. **Reach the unresolved control-flow boundary.**  
    The supplied representation stops at an invalid-instruction indication. No subsequent operation or normal return is established.

### Integer-width qualifications

The input-word shift in the first call is explicitly unsigned and 32-bit. For example, shifting `0x40000000` left by two produces zero after 32-bit truncation.

The two shifts of dependency results differ: those results are represented as signed 32-bit integers. The visible evidence establishes an attempted left shift by two, but does not independently establish behavior for negative inputs or values whose mathematical product by four exceeds the signed 32-bit range.

For nonnegative dependency results through `0x1FFFFFFF` inclusive, the mathematical product by four is representable as a signed 32-bit integer. Within that range, the forwarded value is unambiguous. Outside it, exact executable behavior is documented_only pending instruction-level evidence; this contract does not silently impose unsigned wraparound.

### Ordering and retained state

The first write precedes every dependency invocation. The second read and write occur after the seventh call and before the eighth.

The second selector remains the visible source for the second argument of F-00000000000d, even though five calls occur between capturing it and that invocation. Changes to the selector memory during those calls do not imply another direct read by this routine.

These statements describe the represented data flow. They do not assert that unknown dependencies preserve arbitrary registers, globals, or memory.

## Side effects

### Direct memory effects

The body visibly attempts exactly two one-byte writes. Either write may replace an existing byte at its selected address.

The addresses can be equal or different:

- Equal selectors select the same address.
- Different selectors select different addresses.
- The two reads need not return the same selector.
- An earlier write or dependency action may affect memory used by the later read.

If the selected write address aliases the selector location, the first write itself can change the value later read as the second selector. No non-aliasing precondition is established.

The source model describes one-byte accesses. Physical bus transaction widths, volatile semantics, and memory-ordering guarantees are not established.

### Indirect effects

All fourteen dependencies may have additional side effects. Their bodies are unavailable, so the contract does not restrict them to pure computation or assume they preserve memory.

In particular, dependency activity may affect:

- The second selector read.
- The contents of either write destination.
- Global state.
- Control flow and whether subsequent operations are reached.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, or lifetime management is visible.

The routine does not visibly release the entry-base storage or the selected byte destinations. That does not establish that dependencies leave such storage allocated or valid.

The integer arguments must not be interpreted as owned pointers without further evidence.

### Errors and interruption

No local error reporting, status check, recovery path, or cleanup path is visible.

If an access or call does not complete, the subsequent sequence is not guaranteed. There is no visible rollback of a completed first write when a later operation fails.

The final invalid-instruction indication is a limitation of the supplied control-flow representation. It is not sufficient evidence for a specified exception, process exit, or intentional hardware halt.

## Dependencies

| Opaque ID | Explicit inputs | Result visibly consumed |
|---|---|---|
| F-000000000001 | Input word twice; unsigned 32-bit input word shifted left by two | None |
| F-000000000002 | None | One 32-bit quantity, forwarded to F-000000000003 |
| F-000000000003 | Result of F-000000000002 | None |
| F-000000000004 | None | None |
| F-000000000005 | None | None |
| F-000000000006 | None | None |
| F-000000000007 | None | Signed 32-bit result, shifted for F-000000000008 |
| F-000000000008 | Shifted result of F-000000000007 | None |
| F-000000000009 | None | None |
| F-00000000000a | None | None |
| F-00000000000b | None | None |
| F-00000000000c | None | Signed 32-bit result, shifted for F-00000000000d |
| F-00000000000d | Shifted result of F-00000000000c; zero-extended second selector | None |
| F-00000000000e | None | None |

“None” under inputs means no explicit arguments appear in the supplied call expression. It does not establish the absence of implicit machine-state inputs. “None” under consumed results does not establish that a dependency has no return value.

The first target depends on entry link state. The remaining targets are represented through fixed code-location anchors with a common displacement of hexadecimal `0xFFE`. Their real identities, validity, and calling conventions remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes valid accesses, returning dependencies, and continued execution through the relevant steps.

### Zero input and stable selector

Given an input word of zero and both selector reads returning 5:

- The first write stores zero at address 22.
- F-000000000001 receives three zero arguments.
- The second write also stores zero at address 22.
- F-00000000000d receives 5 as its second argument.

The dependency sequence must still occur; a zero input does not visibly bypass it.

### Distinct low byte and shifted byte

Given input word `0x004000AB`:

- The first write value is `0xAB`.
- The third argument to F-000000000001 is `0x010002AC`.
- The second write value is `0x01`.

The two write values must not be treated as interchangeable extractions.

### Unsigned truncation in the first call

Given input word `0x40000000`:

- F-000000000001 receives `0x40000000` as each of its first two arguments.
- Its third argument is zero.
- Both direct write values are zero.

A wider mathematical result must not be forwarded as the third argument.

### Selector changes between reads

Given a first selector of 2 and a second selector of 255:

- The first write targets address 19.
- The second write targets address 272.
- F-00000000000d receives 255, not 2 or 272, as its second argument.

If later dependencies change selector memory again, that alone does not change the retained argument.

### Representable dependency-result shifts

If F-000000000007 returns 3, F-000000000008 receives 12.

If F-00000000000c returns 7, the first argument to F-00000000000d is 28.

These examples do not resolve negative or overflowing signed shifts.

### Failure after the first write

If the first write completes but F-000000000001 does not return, no later direct write is established. The visible body provides no rollback of the first write.

## Uncertainties

The following are documented_only:

- The application-level purpose of the routine.
- Validity and physical meaning of the low numerical write addresses.
- Actual target identities and whether multiple call sites reach the same implementation.
- Complete calling conventions and implicit inputs.
- Dependency side effects, ownership behavior, and global-state preservation.
- Whether every dependency returns normally.
- Exact machine behavior for negative or overflowing signed left shifts.
- Whether the incoming base remains usable across calls under the real execution convention.
- Volatile, device, concurrency, and physical access-order semantics.
- Whether the invalid-instruction boundary reflects executable behavior, missing decoding context, or an incorrect control-flow reconstruction.
- Behavior beyond that boundary.

No inferred initialization, device-control, dispatch, or shutdown purpose is adopted as a requirement.

## Evidence

**Established static evidence:** The supplied representation contains one explicit unsigned input, two selector-byte reads, two one-byte writes, fourteen ordered indirect calls, three consumed dependency results, and an unresolved terminal control-flow boundary. It exposes the shift counts, address offsets, argument order, and repeated selector read described above.

**Inferred intent:** The ordered calls may form a larger stateful sequence, but no semantic role for that sequence is established. This inference creates no additional requirements.

**Evidence limitation:** The representation carries warnings about invalid instruction data and truncated control flow. Its apparent types and call structure therefore describe available static evidence, not independently verified machine behavior.

No runtime tests have been performed. The acceptance examples are derived conditions for review, not reports of observed execution.
