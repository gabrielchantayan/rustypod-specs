<!-- Trusted provenance: F-a6233adaeba1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a6233adaeba1

## Purpose

F-a6233adaeba1 executes an indefinitely repeated sequence of indirect dependency calls, narrow memory accesses, address calculations, and state updates. Each iteration includes a software breakpoint. There is no represented normal return or loop-exit condition.

The supplied static representation establishes the ordering of these operations, but not their application-level purpose. An interpretation such as initialization, device servicing, or record processing is **documented_only**: the available evidence does not establish any of those roles.

Execution depends on inherited machine state, global pointer values, dependency-produced register values, and accessible memory at several computed addresses. This is not established as a conventional, safely callable, parameterless API.

## Interface

### Entry and exit

- No explicit parameters or return value are declared.
- Entry nevertheless consumes an inherited link-register value, another inherited register value, and a machine base whose identity is unresolved.
- Two global 32-bit values are used as address bases.
- Several dependencies supply results through ordinary return values or additional machine registers.
- If every operation completes and the breakpoint permits continuation, execution repeats indefinitely.
- A dependency failure, inaccessible address, breakpoint exception, or other machine fault may prevent further progress. No recovery behavior is represented.

The apparent parameterless declaration does not establish that callers may omit the required machine-state setup.

### Contract notation

The following names describe evidence-bearing values, not public symbols:

| Name | Meaning |
|---|---|
| **B** | Unresolved machine base used for several offset accesses; reduced by 244 bytes during each completed iteration. |
| **L** | Inherited link-register value used to derive the first dependency target on the first iteration. |
| **R** | Inherited register value used in a late byte-store address. No assignment to it is represented. |
| **G1**, **G2** | Two global 32-bit values read as address bases at different points in the iteration. |
| **A**, **D**, **E** | Successive 64-bit dependency results, each split into low and high 32-bit halves. |
| **P** | Address base obtained by reading G1 early in the iteration. |
| **Q** | Address base loaded from the current B plus 800 bytes. |
| **H** | Zero-extended 16-bit value read from address 0x21c48994. |
| **T** | Constant derived address value 0x21c4. |
| **N** | Address base read as a 32-bit word at H plus 68 bytes. |
| **J** | A dependency result used as a four-byte-scaled offset. |
| **K** | Address base read as a 32-bit word at T plus 84 bytes. |
| **U** | Zero-extended 16-bit value read at K plus 52 bytes. |
| **V** | Zero-extended 16-bit value read at E’s high half plus 20 bytes. |

F-dep01 through F-dep37 identify dependency **call sites** in execution order. These opaque identifiers do not assert distinct underlying functions or known signatures.

### Widths and layout

The representation distinguishes:

- 8-bit byte reads and writes.
- 16-bit halfword reads and writes.
- 32-bit word reads, writes, address components, and arithmetic.
- 64-bit results split at bit 32.

Offsets below are in bytes. A 16-bit value used as an address component is zero-extended, not sign-extended. A halfword store of a wider value retains its low 16 bits.

Endianness, alignment requirements, memory-region permissions, and the precise machine overflow rules are not independently established.

## Behavior

The following sequence describes one iteration. Dependency calls remain observable even when their arguments or results are not otherwise explained.

### 1. Initial calls and first result processing

1. Invoke F-dep01 through a target derived from the current link-register value plus 4094. Two explicit arguments are supplied, both zero.
2. Invoke F-dep02, then F-dep03.
3. Read G1 and retain that value as P.
4. Invoke F-dep04.
5. Invoke F-dep05 and obtain the 64-bit result A.
6. Invoke F-dep06 with two explicit 32-bit arguments:
   - A’s low half.
   - The sum of A’s high and low halves.

   The represented expression also adds an 8-bit value initialized to zero earlier in the iteration; it contributes no nonzero term.
7. Store one byte, obtained from an additional output register associated with F-dep06, at P plus 17.
8. Invoke F-dep07.
9. Store a zero byte at P plus 21.
10. Invoke F-dep08. Store its 16-bit result at offset 12 from an address supplied through an additional output register.

P is captured before F-dep04. Subsequent uses of P are not fresh reads of G1. This distinction matters if a dependency changes the global value.

### 2. Second wide result and related writes

11. Invoke F-dep09 and obtain the 64-bit result D.
12. Store a zero halfword at D’s high half plus 44.
13. Store the result of logically shifting D’s low half right by 26 at B plus 620. This is a 32-bit store; the stored value is in the range 0 through 63.
14. Establish the constant 0x21c48960 for subsequent calculations.
15. Invoke F-dep10.
16. Read the 32-bit value at B plus 800 and retain it as Q.
17. Invoke F-dep11, then F-dep12.
18. Store D’s low half shifted left by two at Q plus 84, using the represented 32-bit result.
19. Invoke F-dep13, F-dep14, F-dep15, F-dep16, and F-dep17, in that order.
20. Read the 16-bit value at 0x21c48994 and zero-extend it to form H.
21. Invoke F-dep18.
22. Read the 32-bit word at B plus 640 and pass that word as the explicit argument to F-dep19. Retain F-dep19’s result for use as J.

D is the result of F-dep09, not the earlier result A. Its low half is reused after intervening dependency calls.

### 3. Breakpoint and intermediate address-based operations

23. Execute a software breakpoint carrying immediate value 0x92.

All later behavior in this iteration is conditional on execution resuming beyond this breakpoint.

24. Shift J left by two.
25. Form T by logically shifting 0x21c48960 right by 18 and multiplying by four. The resulting value is 0x21c4.
26. Read a 32-bit value at H plus 68 and retain it as N.
27. Invoke F-dep20.
28. Invoke F-dep21 and retain its 32-bit result.
29. Read a 32-bit word at H plus 4. Invoke F-dep22 with the result from F-dep21 as its first explicit argument and the just-read word as its second.
30. Read G2 and retain its 32-bit value.
31. Invoke F-dep23.
32. At the address formed by adding the shifted J to the captured G2 value, store the low 16 bits of that captured G2 value.
33. Invoke F-dep24.
34. Read a 32-bit value at T plus 84 and retain it as K. With the represented constant arithmetic, this read is at address 0x2218.
35. Invoke F-dep25 and use its result as an address base.
36. Store the low 16 bits of T at that result plus 4. The stored halfword is 0x21c4.
37. Invoke F-dep26, then F-dep27.
38. Read a 16-bit value at K plus 52 and zero-extend it to form U.

The two reads relative to H occur at different times: the read at H plus 68 precedes F-dep20, while the read at H plus 4 follows F-dep21. They must not be treated as one atomic snapshot.

Likewise, G2 is captured before F-dep23. Both the destination calculation and stored halfword in step 32 use that captured value, regardless of later changes to G2.

### 4. Third wide result, base movement, and final updates

39. Invoke F-dep28 and obtain the 64-bit result E.
40. Store E’s low 32-bit half at the current B plus 600.
41. Read a 16-bit value at E’s high half plus 20 and zero-extend it to form V.
42. Form two scaled values:
   - E’s low half shifted left by six.
   - E’s high half multiplied by four.
43. Reduce B by 244 bytes. This update occurs after the store at the old B plus 600.
44. Form U multiplied by four.
45. Store E’s low half as a 32-bit word at the address formed by adding 116 to four times U.
46. Invoke F-dep29.
47. Store V as a 32-bit word at N plus 84.
48. Invoke F-dep30.
49. Store V as a 32-bit word at N plus 100.
50. Invoke F-dep31.
51. Store the low 16 bits of N at an address formed by adding:
   - E’s low half shifted left by six.
   - An additional register output associated with F-dep31.
52. Invoke F-dep32, then F-dep33.
53. Read one byte at the address formed by adding U to four times U.
54. Invoke F-dep34.
55. Store that byte at V plus R plus 25.
56. Invoke F-dep35.
57. Store E’s high half multiplied by four as a 32-bit word at N plus 68.
58. Invoke F-dep36.
59. Establish a fixed link-register-derived value for the next iteration’s first call.
60. Read a 32-bit word at the address formed by adding 68 to E’s high half multiplied by four. Pass that word as the explicit argument to F-dep37.
61. Begin another iteration.

The two stores of V are separate operations with F-dep30 between them. The source byte is read before F-dep34 and written afterward; it is not reread after that call.

The update to B persists into the next iteration in the supplied representation. No compensating increment is shown.

### Arithmetic boundaries

The address formed from U plus four times U is mathematically five times U. Since U is a zero-extended halfword, its range is 0 through 327675.

The address used for the 32-bit store at four times U plus 116 ranges from 116 through 262256. These numeric bounds do not establish that the addresses are valid or appropriately aligned.

The shift of D’s low half by two, the shift of E’s low half by six, the scaling of J, and the multiplication of E’s high half by four can lose high bits in a 32-bit machine interpretation. The static representation does not provide checked-arithmetic behavior or an overflow error path.

## Side effects

### Direct memory effects

Each iteration reaching all operations performs:

- Two byte stores relative to P.
- One dependency-result halfword store through an additional register output.
- One zero halfword store relative to D’s high half.
- Several 32-bit stores through B, Q, N, and computed numeric addresses.
- A halfword store indexed from G2.
- A halfword store relative to F-dep25’s result.
- A halfword store using an additional output from F-dep31.
- One byte transfer from a computed source address to an address involving V and R.
- A persistent decrement of the unresolved base B.

These writes are not a transaction. No rollback or cleanup is represented if a later operation faults or the breakpoint prevents continuation.

### Dependency and machine effects

All dependency calls may have additional effects not exposed in the supplied body. They must not be assumed to preserve globals, preserve memory, allocate nothing, release nothing, or return normally.

The breakpoint is an explicit machine-visible event. Its debugger, exception-handler, or platform behavior is unknown.

### Ownership

No direct allocation, release, ownership transfer, reference-count operation, or resource-lifetime protocol is established. Values used as addresses are not validated before access. Address use alone does not establish ownership or permission to release the referenced storage.

## Dependencies

The call-site identifiers support the following limited contracts:

| Call sites | Established interaction |
|---|---|
| F-dep01 | Indirect target depends on link-register state; receives two explicit zero arguments. |
| F-dep02–F-dep04 | Calls occur in order, with the G1 read between F-dep03 and F-dep04. |
| F-dep05 | Supplies 64-bit result A. |
| F-dep06 | Consumes the two A-derived words; an additional register output supplies a stored byte. |
| F-dep07 | Runs between the two byte stores relative to P. |
| F-dep08 | Supplies a halfword result and an additional address-bearing register output. |
| F-dep09 | Supplies 64-bit result D. |
| F-dep10–F-dep18 | Ordered calls interleaved with the Q read, D-derived store, and H read. |
| F-dep19 | Receives the word read at B plus 640; supplies J. |
| F-dep20–F-dep23 | Ordered post-breakpoint calls; F-dep21 supplies an explicit argument to F-dep22. |
| F-dep24–F-dep27 | Ordered calls around the K read and the halfword store through F-dep25’s result. |
| F-dep28 | Supplies 64-bit result E. |
| F-dep29–F-dep31 | Separate the N-relative stores; F-dep31 supplies an additional address component. |
| F-dep32–F-dep36 | Ordered calls surrounding the byte transfer and final N-relative word store. |
| F-dep37 | Receives a word read through the four-times-E-high address calculation. |

Calls without listed explicit arguments have no explicit arguments in the supplied representation. That does not establish the absence of implicit register inputs.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **First wide-result arguments.** If A’s low half is 3 and its high half is 5, F-dep06 receives explicit arguments 3 and 8. This example avoids overflow ambiguity.

2. **Narrow stores.** If the captured G2 value is 0x12345678, the indexed halfword store writes 0x5678. Its destination still uses the full captured G2 value plus shifted J.

3. **D-derived fields.** If D’s low half is 0xFC000001, the word stored at B plus 620 is 63. Under 32-bit truncating shift semantics, the later word stored at Q plus 84 is 4.

4. **Breakpoint suspension.** If the breakpoint suspends execution permanently, the earlier calls and writes have occurred, but the read at H plus 68 and every subsequent operation have not.

5. **Base-update timing.** If B is initially 0x1000 at step 40, the E-low store targets 0x1258. B subsequently becomes 0x0F0C before the next iteration, assuming the represented 32-bit address arithmetic.

6. **Zero index.** If U is zero, the E-low word store targets address 116 and the later byte read targets address zero. No zero-value special case or null-address rejection is represented.

7. **Intervening mutation.** If F-dep30 changes memory at N plus 84, that change is not prevented by the earlier V store. The subsequent direct store still writes V at N plus 100.

8. **Fault after partial progress.** If the late byte destination is inaccessible, all preceding completed operations remain observable. No rollback is specified.

## Uncertainties

The following remain **documented_only**:

- The application-level purpose and validity of the address calculations.
- The identity and ABI role of B.
- Required entry values for L and R.
- Dependency signatures, hidden inputs, register preservation, side effects, failure modes, and ownership rules.
- Whether multiple call sites resolve to the same underlying function.
- Platform treatment of the software breakpoint.
- Endianness, alignment enforcement, and memory-map validity.
- Exact overflow and signed-shift semantics behind the low-level representation.
- Whether the apparent perpetual B decrement accurately reflects executable stack or register behavior.
- Whether the declared parameterless, non-returning structure fully represents the original callable boundary.

No assumption that unknown dependencies preserve globals or release memory is part of this contract.

## Evidence

This contract is based solely on the supplied static function representation: access widths, explicit offsets, result splitting, call ordering, breakpoint placement, base update, and unconditional repetition.

“Established” here means established by that representation, not validated against execution. No implementation access, dependency-body inspection, runtime tests, or machine observations have been performed. No provenance hash is asserted.
