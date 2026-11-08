<!-- Trusted provenance: F-adbf5c334ed5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-adbf5c334ed5

## Purpose

F-adbf5c334ed5 describes a low-level sequence of dependency calls, three coprocessor-directed operations, and a final 16-bit memory write. Its explicit input is transformed into several call arguments. Later dependency results supply additional arguments and determine the destination of the final write.

The recovered path ends at an invalid-instruction boundary. It does not establish normal completion, a recoverable error protocol, or a meaningful return value.

**Established static evidence:** The supplied representation contains one straight-line path, with no explicit conditional branch or loop. It shows seventeen dependency calls, three coprocessor-directed operations, and one direct memory store.

**Inferred intent:** The sequence may perform platform-specific coordination or device control. The coprocessor operations and low-valued final destination support that possibility, but neither purpose nor address-space interpretation is established.

**Scope:** This contract describes the represented path and its data dependencies. It does not establish that the representation accurately recovers a valid executable function. Execution reaching any later operation is conditional on all earlier operations returning or completing.

## Interface

### Explicit input

The function accepts one integer input, denoted **X**.

The supplied representation supports a 32-bit interpretation of this input and its ordinary integer intermediates. For the behavioral calculations below:

- A **word** is 32 bits.
- A **double word** is 64 bits.
- A **halfword** is 16 bits.
- A left shift of a word retains its low 32 bits.
- Extracting the upper or lower word of a double word preserves that word’s bit pattern.

These rules describe the apparent machine-width dataflow. They do not impose the language-level semantics of signed shifts from the supplied rendering. A trustworthy instruction decode is unavailable.

### Implicit entry context

The explicit argument is not sufficient to determine all behavior. The representation also consumes:

- An incoming link-associated value used to form the first call target.
- An incoming general-register value used by the first coprocessor operation.
- Two incoming coprocessor-context values used in the coprocessor operations.

Call these hidden inputs **L**, **G**, **CA**, and **CB**, respectively. Their register provenance is visible in the representation; their application-level meaning, validity requirements, and initialization protocol are not established.

The first dependency target is formed by adding a fixed displacement of `0xDF0` to **L**. The contract does not infer that **L** is an ordinary return address or that this target is valid.

### Outputs

No usable return value is established. The represented signature has no return value, and the visible path ends at an invalid-instruction boundary rather than an ordinary return.

Externally observable effects may include:

- The seventeen dependency invocations and their arguments.
- Three coprocessor-directed operations.
- All effects performed by the dependencies.
- A final attempt to write the halfword value `0x00CD`.
- Abnormal termination, trapping, or another environment-specific consequence at the invalid-instruction boundary.

### Preconditions for reaching the final write

Reaching the final write requires every preceding call and coprocessor operation to complete and continue along the represented path. Each call target must be executable under the actual calling convention.

For the final store to complete, its computed destination must permit the represented halfword write. The source does not establish mapping, permissions, alignment support, or whether that destination names ordinary memory or a device region.

No additional input validation is visible.

## Behavior

### Arithmetic invariants

Define **S** as the 32-bit result of shifting **X** left by three bits.

Two important consequences follow:

1. The low three bits of **S** are zero. Therefore, masking **S** with `3` produces zero.
2. Every later value derived solely by left-shifting that masked result is also zero.

Consequently, the visible calls that receive those later values receive zero, regardless of **X**, under the stated 32-bit interpretation.

The input-derived arguments used elsewhere are:

- **S** shifted left by eight bits: equivalently, **X** shifted left by eleven bits, retaining 32 bits.
- **S** shifted left by twelve bits: equivalently, **X** shifted left by fifteen bits, retaining 32 bits.

No checked arithmetic, saturation, or overflow error is represented.

### Ordered interaction sequence

The following order is the order shown by the supplied representation. Dependencies are assigned opaque identifiers in the Dependencies section. A call described as having “no explicit arguments” has no arguments shown in the representation; this is not proof that it has no implicit inputs.

1. **Compute the initial shifted input.**  
   Form **S** from **X** as described above.

2. **Invoke F-01.**  
   Pass **X** to the entry-context-derived target. Its result, if any, is not explicitly consumed.

3. **Invoke F-02.**  
   No explicit arguments are shown. Its result is not explicitly consumed.

4. **Invoke F-03.**  
   No explicit arguments are shown. Its result is not explicitly consumed.

5. **Perform the first coprocessor-directed operation.**  
   The represented selector positions are `0`, `7`, and `7`. The transferred word is **G** shifted left by twenty-four bits. Under the 32-bit model, only the low byte of **G** contributes. The remaining represented context operands are **CA** and **CB**.

6. **Form the zero-valued masked intermediate.**  
   Mask **S** with `3`, then shift the result left by sixteen bits. The result is zero.

7. **Perform the second coprocessor-directed operation.**  
   The represented selector positions are `0`, `0`, and `7`. The transferred word is zero. Both remaining represented context operands are **CB**.

8. **Invoke F-04.**  
   Pass zero. Retain its word-sized result, denoted **R**.

9. **Perform the third coprocessor-directed operation.**  
   The represented selector positions are `0`, `2`, and `7`. The transferred word is **S** shifted left by eight bits. The remaining represented context operands are **CA** and **CB**.

10. **Invoke F-05.**  
    Pass, in order:
    - **S** shifted left by eight bits.
    - **S** shifted left by twelve bits.
    - **R** shifted left by nine bits.

    Any result is not explicitly consumed.

11. **Invoke F-06.**  
    No explicit arguments are shown. Retain its double-word result, denoted **A**.

12. **Invoke F-07.**  
    Pass, in order:
    - The low word of **A**.
    - The high word of **A**.
    - The low word of **A** shifted left by one bit.

    Any result is not explicitly consumed.

13. **Invoke F-08.**  
    Pass zero. Retain its word-sized result.

14. **Invoke F-09.**  
    Pass the result of F-08. Retain the word-sized result of F-09.

15. **Invoke F-10.**  
    Pass the result of F-09. Any result is not explicitly consumed.

16. **Invoke F-11.**  
    No explicit arguments are shown. Any result is not explicitly consumed.

17. **Invoke F-12.**  
    No explicit arguments are shown. Retain its double-word result, denoted **B**.

18. **Invoke F-13.**  
    Pass, in order:
    - The low word of **B**.
    - The high word of **B**.
    - The high word of **B** shifted left by eleven bits.

    Retain its double-word result, denoted **C**.

19. **Derive the final destination components.**  
    Shift **C** right logically by forty-one bits and retain the resulting word. This intermediate lies between zero and `0x7FFFFF`, inclusive.

    A separate intermediate derived from the earlier masked input is shifted left by twenty-five bits. It remains zero.

20. **Invoke F-14.**  
    Pass the low word of **C**. Any result is not explicitly consumed.

21. **Invoke F-15.**  
    No explicit arguments are shown. Retain its word-sized result.

22. **Invoke F-16.**  
    Pass the result of F-15 followed by zero. Any result is not explicitly consumed.

23. **Invoke F-17.**  
    No explicit arguments are shown. Any result is not explicitly consumed.

24. **Attempt the final halfword store.**  
    Compute the destination as described below, then write `0x00CD`.

25. **Reach the invalid-instruction boundary.**  
    No ordinary return or subsequent valid operation is established.

### Final destination calculation

The destination is determined solely by the upper twelve bits of **C**, among the explicit arithmetic inputs to the address expression.

Let **Q** be **C** shifted right logically by fifty-two bits. Thus **Q** ranges from zero through `4095`.

The destination’s numeric address is:

**Q + floor(Q / 4)**

This simplification is valid because the earlier forty-one-bit right shift yields a nonnegative 23-bit value. Its subsequent signed right shift by eleven bits therefore agrees with an unsigned right shift and yields **Q**.

Consequences:

- The destination ranges from `0` through `5118`, inclusive.
- The address arithmetic does not overflow a 32-bit word.
- The destination is not guaranteed to be even.
- No visible null-address, bounds, permission, or alignment check precedes the write.
- Changing the lower fifty-two bits of **C**, while keeping its upper twelve bits fixed, does not change the destination.
- The low word of **C** remains observable as the argument to F-14, even though it does not determine the destination.

The store is represented as one 16-bit write. Byte order, bus transaction width, atomicity, and treatment of misaligned destinations are not established.

## Side effects

### Direct effects

The visible function directly requests three coprocessor operations and one halfword memory store. It does not explicitly read the destination before writing it.

No direct allocation, deallocation, ownership transfer, lock operation, or ordinary global-data access is shown. This absence does not establish that the dependencies lack those effects.

The final store occurs after F-17 returns. It is not an early initialization write and is not conditional on any explicitly checked success result.

### Dependency effects and ownership

No dependency implementation is supplied. Each dependency may read or modify memory, change global state, affect hardware, allocate or release resources, fail, or not return.

Word-sized results are passed onward as opaque bit patterns. Their use does not establish that they are pointers, handles, status codes, owned objects, or borrowed objects.

In particular:

- Passing the result of F-08 to F-09 does not establish an ownership transfer.
- Passing the result of F-09 to F-10 does not establish cleanup.
- Passing the result of F-15 to F-16 does not establish acquisition or release.

The represented dataflow retains certain intermediate results across later calls. Its feasibility depends on the actual calling convention and instruction sequence. This contract does not independently assume that unknown callees preserve globals or arbitrary machine state.

### Errors and interruption

No explicit error-result test, retry, rollback, or cleanup path is visible. Results that feed later operations are used without a shown validity check.

A failed call, faulting coprocessor operation, invalid store, or nonreturning dependency may prevent every later operation. No function-specific recovery behavior is established.

The invalid-instruction boundary is distinct from successful completion. The evidence does not establish whether it represents intentional trapping, damaged input, or an incorrect recovery of executable instructions.

## Dependencies

The following opaque IDs label the seventeen call sites for this contract. They do not assert globally verified function identities or guarantee that distinct call sites resolve to distinct implementations.

| ID | Explicit arguments | Result explicitly consumed |
|---|---|---|
| F-01 | Input word **X** | None |
| F-02 | None shown | None |
| F-03 | None shown | None |
| F-04 | Zero | Word **R** |
| F-05 | Two input-derived words; **R** shifted left by nine | None |
| F-06 | None shown | Double word **A** |
| F-07 | Low word of **A**; high word of **A**; doubled low word | None |
| F-08 | Zero | Word |
| F-09 | Result of F-08 | Word |
| F-10 | Result of F-09 | None |
| F-11 | None shown | None |
| F-12 | None shown | Double word **B** |
| F-13 | Low word of **B**; high word of **B**; shifted high word | Double word **C** |
| F-14 | Low word of **C** | None |
| F-15 | None shown | Word |
| F-16 | Result of F-15; zero | None |
| F-17 | None shown | None |

Except for F-01’s entry-context-derived target, the representation forms call targets from embedded code-location anchors and fixed displacements. Their validity and semantic identities are unverified.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes the stated word-width model and that preceding operations complete.

### Zero input

For **X = 0**, both input-derived arguments to F-05 are zero, and the third coprocessor operation transfers zero.

The third argument to F-05 remains dependent on **R**. Zero input therefore does not imply that all three arguments are zero.

### Single-bit input

For **X = 1**:

- The third coprocessor operation transfers `0x00000800`.
- F-05 receives `0x00000800` and `0x00008000` as its first two arguments.
- F-08 receives zero.
- F-16 receives zero as its second argument.

The last two properties hold for every input, not just this example.

### Word-width truncation

For **X = 0x00200000**, shifting left by eleven bits discards the only set bit under 32-bit truncation. Both input-derived arguments to F-05 are zero.

An interpretation that retains wider intermediate arithmetic without the represented word truncation would not satisfy this model.

### Double-word argument extraction

If F-06 returns **A = 0x1234567880000001**, F-07 receives:

- `0x80000001`
- `0x12345678`
- `0x00000002`

The third argument demonstrates 32-bit truncation rather than saturation.

### Destination boundaries and alignment

If F-13 returns a value whose upper twelve bits are:

| Upper twelve bits | Final numeric destination |
|---|---:|
| `0` | `0` |
| `1` | `1` |
| `3` | `3` |
| `4` | `5` |
| `4095` | `5118` |

If the destination permits the operation, the attempted value is always the halfword `0x00CD`.

An odd destination is not rejected by any visible check. Whether it succeeds is environment-dependent.

### Interrupted execution

If F-17 does not return, the final store is not reached on the represented path. If the final store faults, the representation establishes no recovery path that continues to the invalid-instruction boundary.

## Uncertainties

The following are **documented_only**:

- Whether the supplied representation corresponds to a valid executable function.
- The exact instruction set, calling convention, pointer width, and return-value transport.
- The architectural meaning and ordering guarantees of the coprocessor operations.
- Whether machine-level barriers or synchronization effects exist beyond those visible here.
- The identities, implementations, side effects, failure behavior, and ownership rules of all dependencies.
- Whether apparent no-argument calls consume hidden register or global inputs.
- The destination address space, mapping, permissions, byte order, alignment rules, and write atomicity.
- Whether the final invalid-instruction boundary is intentional.
- Whether valid alternate paths exist outside the supplied representation.

These unknowns must not be replaced with assumptions of success, harmlessness, memory ownership, or normal return.

## Evidence

This contract is based solely on the supplied static representation. That material explicitly warns about invalid instruction data and truncated control flow, limiting confidence in its recovery.

The strongest evidence supports the represented operation order, argument dataflow, 32-bit and 64-bit value handling, zero-valued masked intermediates, and final 16-bit store expression. The simplified destination range follows arithmetically from those widths and shifts.

No runtime tests, executable inspection, dependency inspection, hardware observation, or external provenance verification have been performed.
