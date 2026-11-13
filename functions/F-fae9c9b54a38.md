<!-- Trusted provenance: F-fae9c9b54a38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fae9c9b54a38

## Purpose

The routine performs an ordered sequence of indirect calls, calculated 32-bit memory stores, and byte accesses, followed by an unconditional repeating call-and-store sequence.

No application-level purpose is established. The supplied representation does not justify interpreting the operations as allocation, initialization, serialization, device configuration, or conversion.

The represented path has no normal return. Continued execution requires each preceding call to return and each memory operation to complete. The supplied warnings about invalid instruction data and removed unreachable regions limit confidence that this representation matches executable control flow.

This contract separates:

- **Established static evidence:** operations, widths, ordering, and target-expression changes visible in the supplied representation.
- **Inferred intent:** no application-level intent is assumed.
- **Documented-only unknowns:** unresolved dependency contracts, machine-level interface details, memory validity, and executable behavior not established by the representation.

## Interface

### Explicit inputs and result

| Input | Apparent width | Visible use |
|---|---:|---|
| First input | 32 bits | Supplied to the first indirect call. |
| Second input | 32 bits | No explicit use in the visible body. |
| Third input | Unsigned 32 bits | Logically shifted right by seven and supplied to a later call. |

The apparent interface returns no value. No return operation is visible. Neither the apparent signature nor the absence of a visible use establishes the complete machine-level calling convention.

### Additional incoming state

Execution also consumes incoming state not represented as explicit parameters:

- An address-like base for the initial store.
- A 32-bit value for that store.
- An unsigned 32-bit quantity, called **E** below.
- A control-address quantity used to derive the first indirect-call target.

The body later consumes a secondary register output, called **W** below, after D03. Its producing call or machine event is unresolved; it is not shown as an ordinary return value.

The caller must provide whatever machine state and accessible memory the actual executable requires. The representation supplies no validation mechanism for those prerequisites.

### Contract notation and arithmetic

The following labels describe values, not implementation identifiers:

- **S8:** E logically shifted right by eight.
- **S16:** E logically shifted right by sixteen.
- **Q1–Q17:** selected call results, introduced in execution order.
- **B:** the unsigned 32-bit quantity four times Q6, with its highest bit cleared, then logically shifted right by seven.
- **C:** the address formed by shifting Q12 left by two, multiplying the represented result by 0x400, and adding one.
- **N:** Q14 with its highest two bits cleared, logically shifted right by twenty-five.

Apparent ordinary call results and word stores are 32 bits. Byte reads produce unsigned values from 0 through 255. Byte stores retain eight bits.

Unsigned right shifts have their ordinary logical meaning. Several other expressions use signed-looking multiplication or left shifts. Their displayed arithmetic is recorded here without assuming that overflow or negative operands have defined high-level-language semantics. Any example relying on wraparound explicitly assumes a 32-bit bit-vector model.

Calculated addresses are not established allocations, object references, indices, or lengths. Pointer width and address-conversion rules remain unresolved.

## Behavior

### Dependency-group convention

**D01–D18** identify eighteen ordered dependency groups. A reference such as **D06.3** is an opaque function ID for a call site within its group, not a resolved symbol or proof of a distinct callee.

Unless specified otherwise, a **five-call group** consists of:

- One call with the listed explicit arguments.
- Four subsequent calls with no explicit arguments shown.
- A selected result from the fifth call, where one is identified.

Calls without displayed arguments may still consume machine state. No dependency is assumed to preserve globals, memory, resources, or otherwise unmentioned registers.

### Initial word-store sequence

The first explicit memory effect is a 32-bit store of the additional incoming value at the additional incoming base plus 0x14. It precedes D01.

The subsequent order is:

| Group | Explicit arguments to its first call | Selected result | Word store immediately after the group |
|---|---|---|---|
| D01 | First input | Q1 | Four times Q1 at sixteen times Q1 plus 0x14 |
| D02 | Eight times E; eight times E again; S8; third input logically shifted right by seven | Q2 | Eight times E at eight times E plus 0x14 |
| D03 | Q2 shifted left by two | Secondary output W is consumed afterward | W scaled by 2²⁴ at W scaled by 2²⁶ plus 0x14 |
| D04 | W scaled by 2²⁴ | Q3 | W scaled by 2²⁴ at four times Q3 plus 0x14 |
| D05 | Four times Q3 | Q4 | Four times Q4 at sixteen times Q4 plus 0x14 |

The W-derived value used after D04 is the value calculated before D04. The contract does not require D04 to reproduce W or preserve its original machine register.

### Bridge groups and remaining word stores

D06 contains nine calls. Its first receives four times Q4. The second call produces a 32-bit result supplied to the third call. The fourth produces another result supplied to the fifth. Four further calls have no displayed arguments; the ninth produces Q5.

D06 is followed by a 32-bit store of four times Q4 at four times Q5 plus 0x14.

D07 is a five-call group beginning with four times Q5. Its fifth call produces Q6. The body forms the represented unsigned 32-bit quantity four times Q6 and stores it at sixteen times Q6 plus 0x14.

D08 has the same nine-call argument/result arrangement as D06, but starts with four times Q6 and ends with Q7. Similar argument patterns do not establish identical callees. D08 is followed by a store of S16 at four times Q7 plus 0x14.

D09 is a five-call group beginning with four times Q7 and ending with Q8. It is followed by a store of four times Q8 at sixteen times Q8 plus 0x14.

D10 is a five-call group whose first call receives:

1. Sixteen times Q8.
2. Sixteen times Q8 again.
3. S8 logically shifted right by seven.

Its fifth call produces Q9. After D10, the routine stores sixteen times Q8 at sixteen times Q8 plus 0x14.

Thus the stores immediately after D09 and D10 use the same represented destination but different value expressions, with D10’s calls intervening. They must not be collapsed into a single memory effect.

D11 is a five-call group beginning with Q9 shifted left by two. Its final call produces unsigned 32-bit Q10. Byte accesses begin after D11.

### Byte accesses and intervening calls

B ranges from zero through 0x00FFFFFF. Its associated byte-read address, B plus one, ranges from 1 through 0x01000000.

Each data-dependent byte store below first reads an unsigned byte and then writes at that byte’s numeric value plus 0x11. Those destinations range from 0x11 through 0x110. No separate table base is established.

The byte-access sequence is:

| Position | Read or destination calculation | Stored byte |
|---|---|---|
| After D11 | Read at B plus one; use the byte plus 0x11 as destination | Low byte of Q10 shifted left by two |
| After D12 | Fresh read at B plus one; use the byte plus 0x11 | Low eight bits of Q10 after clearing its highest two bits and logically shifting right by twenty |
| After D13, first store | Direct destination: S16 logically shifted right by seven, plus 0x1C | Low byte of Q12 shifted left by two |
| After D13, second store | Fresh read at B plus one; use the byte plus 0x11 | Zero |
| After D14 | Fresh read at B plus one; use the byte plus 0x11 | Zero |
| After D15 | Read at C; use the byte plus 0x11 | Zero |

The direct destination after D13 equals E logically shifted right by twenty-three, plus 0x1C. It ranges from 0x1C through 0x21B.

The groups separating these accesses are:

- **D12:** Seven calls. The first receives Q10 shifted left by two as each of three arguments. The second produces a result passed to the third. Four further calls have no displayed arguments. The seventh produces Q11.
- **D13:** Five calls beginning with Q11 shifted left by two and ending with unsigned Q12.
- **D14:** Seven calls. The first receives S16 logically shifted right by seven. The second has no displayed arguments. The third receives Q12 with its highest two bits cleared and logically shifted right by six. Four further calls have no displayed arguments. The seventh produces Q13.
- **D15:** Five calls beginning with Q13 shifted left by two and ending with unsigned Q14.

After D15, N is formed from Q14. Its range is zero through thirty-one. The following byte read uses **C, derived from Q12**, not an address derived from Q14 or N.

Every listed byte read is fresh. Earlier stores may alias later read locations, and intervening dependencies may change memory. Retaining an earlier loaded byte instead of performing a later read would not preserve the represented behavior.

### Loop preparation and target transition

After the byte clear using C, D16 performs two calls:

- The first receives N shifted left by one.
- The second has no displayed arguments and produces Q15.

The loop initially carries Q15 and N.

There is also a distinct **call-target state transition**. The first call of D17 uses one represented code-position-derived target on entry to the loop and a different represented target on every subsequent iteration:

- **D17.1.entry:** Derived from the code-position value established for D16’s second call.
- **D17.1.repeat:** Derived from the code-position value established for D18’s final call.

The final portion of D18 changes the code-position value subsequently used to derive D17’s first target. Both derivations use the same displayed additive offset, but their code-position bases differ.

These are distinct represented target expressions. Their ultimate callee identities and effects remain unresolved. This transition must be preserved even if the two targets eventually prove to reach equivalent behavior.

### Unconditional repeating sequence

Each iteration has the following observable order:

1. D17 performs five calls. Its first receives the carried call result and the carried unsigned quantity logically shifted right by eight. Its fifth produces Q16.
2. A fresh unsigned byte is read at C, and zero is stored at that byte’s value plus 0x11.
3. D18 performs five calls, beginning with Q16 shifted left by two. Its fifth produces Q17.
4. The next carried call result becomes Q17 shifted left by two.
5. The next carried unsigned quantity becomes the represented 32-bit result of Q17 shifted left by twenty-six.

D17’s first target follows the entry/repeat distinction above; its remaining four call sites do not acquire that distinction. D18’s final portion establishes the target-selection state for the next iteration.

The first D17 call receives zero as its second argument because N is at most thirty-one.

Under 32-bit bit-vector semantics, subsequent second arguments retain the lowest six bits of the preceding Q17, positioned at bits eighteen through twenty-three. They are not necessarily zero.

C remains derived from Q12 throughout the visible loop. The byte read at C is repeated every iteration, so the zero-store destination can change.

No conditional exit, successful return, or explicit failure branch is visible.

## Side effects

### Memory

Before the first byte store, the body performs **eleven explicit 32-bit stores**:

- The initial incoming-value store.
- One store after each of D01 through D10.

D11 contributes calls but no additional word store.

Before loop entry, the body performs six explicit byte stores and five explicit byte reads. Each loop iteration adds one byte read and one byte zero-store. Dependency memory effects are additional and unresolved.

No explicit word loads appear outside dependency execution. No bounds, null, alignment, or address-validity checks are visible.

Offset 0x14 establishes only the displayed displacement of the word stores; it does not establish a containing structure. Endianness, memory mapping, access privileges, and ordinary-memory versus device-memory interpretation are unknown.

### Ownership and external state

No allocation, release, ownership transfer, or resource lifetime is established by the visible body. Argument passing alone does not establish pointer interpretation or ownership.

Dependencies may affect globals, shared memory, registers, devices, or resources. Their preservation, allocation, release, and re-entry behavior is unknown.

### Errors and completion

There is no explicit error result, diagnostic, recovery path, or normal return.

An invalid call target, invalid memory access, or dependency that does not return may prevent later effects. The representation does not establish a particular exception, trap, status, or recovery mechanism.

## Dependencies

D01–D18 denote call-site groups, not eighteen confirmed distinct functions. Individual opaque call-site IDs do not imply unique callee identity.

Established dependency distinctions include:

- The first call of D01 derives its target from incoming control-address state.
- Later targets use represented code-position-derived arithmetic.
- D06 and D08 have similar call/result bridges but are not established to invoke identical functions.
- D12 and D14 each contain seven calls with different argument arrangements.
- D16 contains two calls.
- D17.1.entry and D17.1.repeat use different represented targets, selected by loop-entry versus post-D18 target state.

All dependency contracts remain unresolved. Numeric target construction does not establish a supported API, valid executable entry point, or reliable calling convention.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes preceding calls return and preceding accesses complete.

1. **Initial ordering.** With incoming base 0x1000 and incoming stored value 0x12345678, the first explicit memory effect is a 32-bit store of 0x12345678 at 0x1014, before D01.

2. **Unsigned decomposition.** With E equal to 0x12345678 and the third input equal to 0x80000000, D02’s third and fourth arguments are 0x00123456 and 0x01000000. Under 32-bit wrapping arithmetic, its first two arguments are both 0x91A2B3C0.

3. **Fresh reads and truncation.** With Q10 equal to 0x12345679 and the first byte read yielding 0x20, the first byte store writes 0xE4 at 0x31. If the read after D12 yields 0x21, that later store writes 0x23 at 0x32. Reusing the earlier destination would violate the read sequence.

4. **Direct destination boundary.** With E equal to 0xFFFFFFFF, the direct byte destination after D13 is 0x21B. Its value is the low byte of Q12 shifted left by two.

5. **Word-store boundary.** If execution reaches the first byte store, eleven explicit word stores have preceded it. D11 does not increase this count.

6. **First loop arguments and target.** The first D17 call receives Q15 and zero, using D17.1.entry. If another iteration begins, its first D17 call uses D17.1.repeat rather than the entry target.

7. **Later loop arguments.** Under 32-bit bit-vector semantics, Q17 equal to 1 produces next-iteration arguments 4 and 0x00040000 for D17’s first call.

8. **Continued execution.** If every dependency returns and accesses remain valid indefinitely, the represented routine continues the loop rather than returning.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the recovered instruction stream and control flow, given the supplied warnings.
- Whether the apparent three-input interface is the actual callable interface.
- Provenance and validity of additional incoming state.
- The producer and machine-level meaning of W.
- Callee identities, contracts, side effects, and resource behavior.
- Definitive signedness, overflow, and left-shift semantics where the representation is signed-looking.
- Pointer width, address conversions, alignment, endianness, and mapped-address validity.
- Whether low-address writes are intentional memory operations, device accesses, or representation artifacts.
- Executable loop behavior, faults, termination, and ownership.

These unknowns do not erase the represented distinction between the first and subsequent D17 targets. That distinction is established statically even though the targets’ meanings remain unknown.

## Evidence

The supplied static representation supports the call ordering, argument transformations, eleven pre-byte word stores, byte-access ordering, repeated reads, loop-carried values, and absence of a visible loop exit.

The loop-target distinction follows from the different code-position values reaching D17’s first target calculation: one established during D16, the other established during D18 and carried into subsequent iterations.

Unsigned range deductions follow from the displayed widths, masks, and logical shifts. Wrapping-arithmetic examples are conditional on a 32-bit bit-vector interpretation.

No tools, runtime tests, dependency resolution, executable inspection, or independent control-flow validation have been performed.
