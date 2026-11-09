<!-- Trusted provenance: F-f693ab5cbdb6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f693ab5cbdb6

## Purpose

F-f693ab5cbdb6 performs an initial memory read and write, conditionally executes an overflow-controlled sequence, invokes several unresolved dependencies, and enters a mandatory loop. If that loop terminates, it calls F-29405a9c1b17, performs two further writes, and passes a shifted result to F-880399f1f568.

An application-level purpose is not established. The supplied static representation includes implicit machine-state inputs, unresolved call targets, overlapping-global warnings, and truncated control flow at a bad-instruction marker. It does not establish a conventional, fully recovered procedure or a successful return path.

This contract distinguishes:

- **Established static evidence:** accesses, conversions, calls, and ordering represented in the supplied material.
- **Conditional interpretation:** arithmetic conclusions requiring an explicitly stated machine model.
- **Inferred intent:** possible application meaning, not required behavior.
- **Documented-only unknowns:** matters the supplied evidence does not resolve.

No runtime tests have been performed.

## Interface

### Explicit inputs

The recovered interface contains nine positional inputs.

| Input | Recovered width | Represented use |
|---|---:|---|
| A1 | 16 bits | Stored at byte offset 0x32 from A4. |
| A2 | Signed 32 bits | Added to A3 to address the initial unsigned 16-bit read. |
| A3 | Signed 32 bits | Initial address-like base; replaced on the optional branch and used in later addressing. |
| A4 | Signed 32 bits | Base of the initial 16-bit write. |
| A5 | 32 bits, interpretation unspecified | Passed unchanged to F-29405a9c1b17. |
| A6 | Unsigned 32 bits | Its highest four bits become an argument to F-29405a9c1b17. |
| A7 | 32 bits, interpretation unspecified | No explicit use is represented. |
| A8 | Signed 32 bits | Determines the destination and value of the final 16-bit write. |
| A9 | Signed 32 bits | Determines the destination of the final 32-bit write. |

These widths describe the recovered representation, not a verified source-language interface or calling convention.

### Implicit inputs

Execution also depends on:

- An incoming overflow condition.
- An incoming continuation-like value used in indirect target arithmetic.
- An incoming register-like value used to derive the first loop write address unless replaced by the optional branch.
- An additional result-register value produced by each loop dependency call and used as an address base.
- Fixed-storage contents read at specific points.
- Memory reached through computed addresses and a loaded pointer.

The nine explicit inputs are insufficient to reproduce the behavior.

### Outputs, errors, and termination

No explicit return value is represented. Observable outputs are memory writes, dependency calls, and subsequent control transfer.

The visible body contains no explicit validation, error result, exception handling, allocation, or release operation. An inaccessible address or a dependency that does not return can prevent later operations from occurring.

After F-880399f1f568, the representation ends at a bad-instruction marker. Whether execution traps, continues into unrecovered instructions, or behaves otherwise is **documented_only**.

### Memory and arithmetic preconditions

Every reached access requires storage accessible at its represented width. Computed addresses can be very low; later loop writes can target address 0x1E.

Pointer-sized loads are represented as 32-bit loads. All offsets below are byte offsets. Byte order, alignment requirements, pointer provenance, address wrapping, and invalid-access behavior are unresolved.

Several expressions use signed 32-bit arithmetic or shifts. Where this contract gives wrapped results, they are conditional on a 32-bit modular machine-arithmetic model. Negative signed right shifts, overflowing signed operations, and oversized shift counts are not assigned portable language-level behavior.

## Behavior

### 1. Initial access sequence

The first represented operation reads an unsigned 16-bit value from A3 plus A2. The next operation writes A1 as 16 bits at A4 plus 0x32.

The read precedes the write, including when the two accesses overlap. The read remains part of the represented path even if its value is not subsequently used.

An outgoing 32-bit argument is initially set to 0x0856F1E0. The numerical constant is established; interpreting it as an address or another kind of token is not.

### 2. Optional overflow-controlled sequence

The incoming overflow condition determines whether this sequence executes. The representation does not derive that condition from the initial read or write.

When the condition is clear, this section is skipped. When it is set:

1. The initial unsigned 16-bit value is promoted to signed 32-bit arithmetic, reduced by 60, and right-shifted by eight. The result is stored as an unsigned 32-bit intermediate.
2. F-U01 is called with 0x0856F1E0.
3. F-U02 is called, yielding a represented 32-bit result.
4. Adding 0x31 to that result and multiplying by two produces a temporary address-related value.
5. A 16-bit zero is written at that temporary value plus 0x002CFFB0.
6. A signed 16-bit value is read at twice the temporary value minus 0x50—that is, twice the entire quantity obtained after subtracting 0x50.
7. That signed read is sign-extended to 32 bits, converted to unsigned 32 bits, and logically shifted right by 29.
8. The unsigned intermediate from step 1 is reinterpreted as a **signed 32-bit value** and right-shifted by a count equal to the low eight bits of that same unsigned intermediate. The result replaces the register-like value used for the first loop write.
9. F-U03 is called with three arguments described below.
10. F-U04 is called.
11. Three reads of one fixed global are represented, followed by the resulting store and base update as detailed below.
12. The common-path outgoing argument becomes zero.

For step 1, the subtraction ranges from −60 through 65,475. Under arithmetic signed right shift, the shifted result ranges from −1 through 255. Conversion to unsigned preserves its 32-bit bit pattern under the represented conversion.

Step 8 is a signed shift, not an unsigned logical shift. Its count can be 32 or greater. Both the treatment of negative signed operands and the treatment of oversized counts remain **documented_only**. In particular, a count must not silently be reduced modulo 32.

Step 7 produces either zero or seven: zero for a nonnegative signed 16-bit read, seven for a negative read. F-U03 consequently receives:

- First argument: zero or 0x0001C000.
- Second argument: zero or 0x0000E000.
- Third argument: a 32-bit value read from a separate fixed global when preparing this call.

The first two arguments use the same sign classification.

#### Three separate observations of the base-related global

After F-U04 returns, the represented fixed-global sequence is:

- First, read the base-related global as 32 bits and capture that observation for the replacement of A3.
- Subsequently, read that same global twice more as part of a 16-bit store:
  - One observation supplies the destination base, to which ten bytes are added.
  - The other observation supplies the low 16 bits to be stored.
- The relative evaluation order of those two store-associated reads is not established.
- Both store-associated reads precede the store itself.
- After the store, the replacement A3 is formed by shifting the **first captured observation** left by eight.

These are three represented reads, not one shared snapshot. Neither the store destination nor its stored value is guaranteed to match the first observation, or each other.

The optional branch therefore contains four fixed-global reads: one for F-U03’s third argument and three of the base-related global. A separate fixed-global read occurs after F-29405a9c1b17 on the terminating path.

### 3. Common-path calls and pointer-derived arguments

The **current base** is the original A3 if the optional branch was skipped. Otherwise, it is the first captured base-related global observation shifted left by eight, subject to the arithmetic qualifications above.

F-U05 is called with:

- 0x0856F1E0 if the optional branch was skipped.
- Zero if it was taken.

F-U06 follows.

The function then loads a 32-bit pointer from current base plus 0x40. It reads a signed 16-bit value through that pointer and adds the sign-extended value to the pointer’s signed numerical representation.

F-U07 receives two derived arguments:

1. A value obtained by signed-right-shifting that sum by 31, converting to unsigned, logically shifting right by 20, then shifting left by 24. Under modular addition and arithmetic signed right shift, the result is zero for a nonnegative sum and 0xFF000000 for a negative sum.
2. The pointer’s signed 32-bit numerical value right-shifted by 28. Under arithmetic signed right shift, this ranges from −8 through 7.

The pointer load precedes the signed 16-bit dereference, which precedes F-U07. No object-layout checks or pointer-range validation are represented.

### 4. Mandatory loop

The loop executes at least once. Each iteration begins with a 16-bit zero write and then calls F-U08.

For the first iteration, the write address is formed by shifting the incoming or replacement register-like value left by nine, multiplying by 0x1000, and adding 0x1E. Under 32-bit modular arithmetic, this is that value multiplied by 2²¹, plus 0x1E.

The first F-U08 argument is zero. Each later F-U08 argument is the zero-extended 16-bit value retained from the preceding iteration.

After each F-U08 call, the represented sequence is:

1. Reset the arithmetic value controlling the next iteration’s zero-write address to zero.
2. Read an unsigned 16-bit value from a computed address and retain it.
3. Read an unsigned 16-bit value at offset 0x14 from the address supplied by F-U08’s additional result register.
4. Double this first offset-0x14 observation to obtain the next index.
5. Read unsigned 16-bit storage at that same offset again.
6. Continue if the second observation is zero; exit otherwise.

The two offset-0x14 reads are separate. The first supplies the doubled index; the second alone controls termination. The representation does not justify merging them or assuming equal values.

Under 32-bit modular arithmetic, the retained-value read in step 2 uses:

- First iteration: current base multiplied by 2²⁷, plus 0x1C.
- Later iterations: eight times the preceding iteration’s first offset-0x14 observation, plus 0x1C.

Every iteration after the first begins with its zero write at address 0x1E.

There is no represented iteration bound, timeout, or error exit. If every second offset-0x14 observation is zero, and intervening operations continue, the loop never reaches the subsequent named dependencies.

### 5. Post-loop call and final writes

After loop exit, one unsigned byte is read at the retained 16-bit value plus 0x1A. That base is a zero-extended 16-bit number, not a full-width pointer loaded from memory.

F-29405a9c1b17 receives:

1. A5 without an explicit transformation.
2. A6 logically shifted right by 28, an unsigned value from zero through 15.
3. Twice the first offset-0x14 observation from the final iteration, ranging from zero through 131,070.
4. The byte just read, with a value from zero through 255.

The third argument can be zero even though the loop has terminated, because termination uses the second observation.

If F-29405a9c1b17 returns, the function then:

1. Reads a 32-bit value from another fixed global.
2. Writes the low 16 bits of A8 multiplied by eight at address A8 multiplied by eight plus 0x34.
3. Writes the captured fixed-global value as 32 bits at address A9 multiplied by eight plus 0x74.
4. Calls F-880399f1f568 with the 32-bit result of F-29405a9c1b17 shifted left by 20.

Under modular 32-bit shifting, only the low 12 result bits affect the final argument. The signed left-shift notation does not independently establish overflow semantics.

The bad-instruction marker follows the final call.

## Side effects

Directly represented writes occur in this order, subject to reaching each operation:

| Condition | Destination | Width | Stored value |
|---|---|---:|---|
| Always | A4 + 0x32 | 16 bits | A1 |
| Optional branch | Twice the adjusted F-U02 result, plus 0x002CFFB0 | 16 bits | Zero |
| Optional branch, after F-U04 | Store-destination observation of the base-related global + 10 | 16 bits | Low 16 bits of a separate store-value observation of that global |
| Each loop iteration | Register-derived address first; 0x1E thereafter | 16 bits | Zero |
| After F-29405a9c1b17 returns | A8 × 8 + 0x34 | 16 bits | Low 16 bits of A8 × 8 |
| Immediately afterward | A9 × 8 + 0x74 | 32 bits | Captured post-call fixed-global value |

The optional global-based store does not use the first captured observation by requirement. That first observation instead supplies the later replacement base.

Destinations may alias inputs, fixed storage, each other, or dependency-visible memory. The optional store could therefore affect storage later accessed, including the global itself, without changing the already captured replacement-base observation.

Unknown dependencies may modify globals or pointed-to memory, retain or release resources, fail to return, or transfer control elsewhere. No preservation or ownership guarantee is inferred from their call positions.

No direct allocation, deallocation, or ownership transfer is established in this body.

## Dependencies

All labels below are opaque function IDs. Unresolved labels identify call sites, not proven distinct implementations.

| Dependency | Role |
|---|---|
| F-U01 | First optional call; receives 0x0856F1E0. |
| F-U02 | Second optional call; supplies the result used in address arithmetic. |
| F-U03 | Optional three-argument call using sign-derived values and a fixed-global read. |
| F-U04 | Final optional call before the three base-related global reads. |
| F-U05 | Common-path call receiving the branch-selected argument. |
| F-U06 | Common-path call before pointer lookup. |
| F-U07 | Common-path call with two pointer-derived arithmetic arguments. |
| F-U08 | Repeated loop call; supplies an additional result-register address base. |
| F-29405a9c1b17 | Post-loop four-argument call supplying the result later shifted. |
| F-880399f1f568 | Final call receiving that shifted result. |

Several indirect targets are formed by adding 0xFFE to incoming or internally selected continuation-like values. Target validity and calling conventions are unresolved. Calls shown without explicit arguments do not establish an absence of implicit inputs.

A fixed-global read after a dependency observes the post-call state. No dependency is assumed to preserve globals or memory.

## Acceptance examples

These are conditional criteria, not observed executions. Arithmetic examples assume 32-bit modular behavior where needed.

1. **Initial aliasing:** If A3 + A2 equals A4 + 0x32, the initial read observes storage before A1 is written there.

2. **Optional branch skipped:** With the incoming overflow condition clear, F-U01 through F-U04 and both optional writes are absent. F-U05 receives 0x0856F1E0, and the current base remains A3.

3. **Optional sign classification:** A signed 16-bit read of −1 produces F-U03 arguments 0x0001C000 and 0x0000E000. A read of zero produces two zero arguments.

4. **Independent global observations:** Suppose the first base-related global read yields 0x1000, the destination-associated read yields 0x2000, and the stored-value read yields 0x3456. The optional store writes 0x3456 at 0x200A, while the replacement base becomes 0x00100000. This does not require an ordering between the two store-associated reads.

5. **Signed shift with oversized count:** If the initial 16-bit read is zero, arithmetic right shift in the first optional calculation yields −1, stored as 0xFFFFFFFF. The later shift reinterprets this as signed −1 and uses count 255. Its result remains documented_only; neither logical shifting nor count masking is an acceptance requirement.

6. **Mandatory first iteration:** A nonzero value already present at offset 0x14 does not bypass the first loop write or F-U08 call.

7. **Termination observations disagree:** If the first offset-0x14 read is zero and the second is one, the loop exits and F-29405a9c1b17 receives zero as its third argument.

8. **Continuation observations disagree:** If the first read is three and the second is zero, the next iteration writes zero at 0x1E and performs its retained-value read at 0x34.

9. **Highest-bit extraction:** A6 equal to 0xA1234567 supplies ten as the second argument to F-29405a9c1b17.

10. **Final writes:** With A8 equal to two and A9 equal to three, the final writes store 16 as 16 bits at 0x44, then the captured fixed-global value as 32 bits at 0x8C.

11. **Final shift:** A result of 0x00000123 from F-29405a9c1b17 yields final argument 0x12300000 under modular shifting.

12. **Nontermination:** If every loop-condition read is zero and no intervening operation stops execution, neither resolved post-loop dependency is reached.

## Uncertainties

The following remain **documented_only**:

- Executable control flow around the bad-instruction regions and any normal return.
- Original ABI, signature completeness, and register-preservation rules.
- Validity and provenance of implicit machine-state inputs.
- Exact signed-overflow, negative signed-shift, and oversized-count behavior.
- Byte order, alignment, address wrapping, and invalid-access behavior.
- Relative evaluation order of the optional store’s two global reads.
- Whether repeated reads observe equal or changing contents.
- Fixed-global contents, mutability, and overlapping-storage interpretation.
- Unresolved dependency identities, errors, ownership, and side effects.
- Any application meaning for the data structures or numerical constants.

Initialization, dispatch, or device-processing interpretations would be inferred intent only.

## Evidence

This contract uses the supplied static representation and opaque dependency mapping. No implementation access, independent disassembly, or runtime testing was available.

Established evidence includes access widths, byte offsets, explicit call ordering, the loop’s duplicated condition-location reads, and the three distinct reads of the optional base-related global. The signed reinterpretation before the variable right shift is also explicit.

The ordering between the optional store’s two global reads is not established. Modular arithmetic simplifications and arithmetic-right-shift examples are conditional interpretations, not runtime findings.
