<!-- Trusted provenance: F-b3bca516b42e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3bca516b42e

## Purpose

The supplied static representation describes an ordered sequence of indirect calls, seven coprocessor-transfer operations, and a final 16-bit memory write, followed by an invalid-instruction termination marker.

A higher-level purpose is not established. The representation contains an explicit warning about bad instruction data and ends where control-flow recovery was truncated. Consequently, this contract distinguishes:

- **Established static evidence:** operations and data dependencies explicitly represented in the supplied material.
- **Conditional behavioral conclusions:** consequences of interpreting the represented word operations as fixed-width 32-bit operations.
- **Unknown behavior:** actual dependency implementations, architectural interpretation, execution validity, and behavior beyond the truncated endpoint.

This is a source-facing contract for the represented path, not a claim that the path is executable as recovered. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has three arguments:

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the represented body. |
| Second argument | 32 bits | No explicit use in the represented body. |
| Third argument | 32 bits, displayed as signed | Shifted left by 26 bit positions; the resulting word is passed to the first dependency call. |

Absence of an explicit use does not establish that the first two arguments are unobservable to callees. Actual calling conventions and implicit register inputs are not provided.

### Implicit inputs

The representation also depends on entry machine state:

- An entry register value used as an indirect-call address base.
- Another entry register value shifted left by 21 positions and passed to that first call.
- Five entry coprocessor-related values used as operands of the represented coprocessor transfers.
- Whatever additional machine state the indirect callees consume.

These implicit inputs are not specified by the three-argument interface. A caller cannot establish successful execution from those three arguments alone.

### Output and completion

The represented function has no ordinary return value. No normal return appears on the supplied path.

After the final explicit memory write, the representation invokes an invalid-instruction termination marker. This establishes the endpoint of the recovered representation, not a portable exception type, process exit status, or guaranteed physical instruction sequence.

Dependency calls can themselves fail, trap, diverge, or transfer control elsewhere. The supplied material establishes no recovery behavior for those cases.

### Arithmetic notation

For the behavioral description below:

- All ordinary words are treated as 32-bit bit patterns.
- A left shift retains the low 32 bits and discards overflow.
- A 64-bit result has a low word and a high word, each 32 bits.
- `word(value)` denotes the low 32 bits of a value.

This bit-vector interpretation is necessary to describe the apparent machine operations without importing language-level guarantees about signed left shifts. Several shifts are displayed on signed values; their portable source-language semantics are not established.

## Behavior

### Dependency identifiers

The opaque IDs F-000000000001 through F-000000000032 identify successive call sites in this contract. They do not assert that every site resolves to a different implementation. No dependency implementation or externally assigned dependency identity was supplied.

The following derived values describe dataflow, not source variable names:

- **Input word:** `word(third argument × 2²⁶)`.
- **Environment word:** `word(entry register value × 2²¹)`.
- **Result X:** the 32-bit result of F-000000000015.
- **Scaled X:** `word(Result X × 16)`.
- **Result Y:** the 32-bit result of F-000000000019.
- **Result A:** the 64-bit result of F-000000000021.
- **Result B:** the 64-bit result of F-000000000027.
- **Result C:** the 64-bit result of F-000000000028.

No dependency result is validated before the represented use.

### Initial call sequence and first transfers

The visible sequence begins as follows:

1. Invoke F-000000000001 with four explicit word arguments: Environment word, zero, Input word, zero.
2. Invoke F-000000000002.
3. Invoke F-000000000003.
4. Invoke F-000000000004.
5. Invoke F-000000000005.
6. Invoke F-000000000006.
7. Invoke F-000000000007 and retain its 32-bit result.
8. Perform the first coprocessor transfer using that result.
9. Invoke F-000000000008.
10. Invoke F-000000000009.
11. Invoke F-000000000010.
12. Perform the second coprocessor transfer using `word(Input word × 2²⁴)`.
13. Invoke F-000000000011 with that same shifted word and zero.
14. Invoke F-000000000012.
15. Invoke F-000000000013 and retain its 32-bit result.
16. Perform the third coprocessor transfer using that result.
17. Invoke F-000000000014.

Calls described without arguments have no explicit arguments in the supplied representation. This is not evidence that their actual machine interfaces have no inputs.

Under the 32-bit interpretation, Input word contains possible nonzero bits only in positions 26 through 31. Therefore, shifting it left another 24 positions produces zero. Both the second transfer’s payload and the first explicit argument to F-000000000011 are consequently zero.

Only the low six bits of the third argument affect Input word. Only the low eleven bits of the environmental register value affect Environment word.

### Result X and dependent operations

Next:

1. Invoke F-000000000015 with three explicit zeros and retain Result X.
2. Perform the fourth coprocessor transfer using `word(Result X × 2)`.
3. Form Scaled X.
4. Invoke F-000000000016 with:
   - `word(Result X × 2)`;
   - `word((Input word AND 0x007fffff) × 16)`;
   - Scaled X;
   - zero.
5. Invoke F-000000000017.
6. Invoke F-000000000018.
7. Perform the fifth coprocessor transfer with a zero payload.
8. Form an intermediate word by masking Scaled X with three and shifting the result left by 16 positions.
9. Perform the sixth coprocessor transfer with a zero payload.
10. Invoke F-000000000019 with one explicit zero and retain Result Y.
11. Perform the seventh coprocessor transfer using `word(Scaled X × 2⁸)`.
12. Invoke F-000000000020 with:
    - `word(Scaled X × 2⁸)`;
    - `word(Scaled X × 2¹²)`;
    - `word(Result Y × 2⁹)`.

Two simplifications follow from the represented widths:

- Input word has no bits in the mask `0x007fffff`, so the second argument to F-000000000016 is zero.
- Scaled X has four zero low bits. Masking it with three gives zero, so the intermediate word formed at step 8 is zero.

These conclusions do not require any particular return value from F-000000000015.

### Wide-result processing

The sequence continues:

1. Invoke F-000000000021 and retain Result A.
2. Invoke F-000000000022 with:
   - Result A’s low word;
   - Result A’s high word;
   - the low word shifted left by one position, truncated to 32 bits.
3. Invoke F-000000000023 with the intermediate word shifted left by eight positions. This argument is zero under the stated arithmetic model.
4. Pass the 32-bit result of F-000000000023 to F-000000000024.
5. Pass the 32-bit result of F-000000000024 to F-000000000025.
6. Invoke F-000000000026.
7. Invoke F-000000000027 and retain Result B.
8. Invoke F-000000000028 with:
   - Result B’s low word;
   - Result B’s high word;
   - `word(Result B’s high word × 2¹¹)`.
9. Retain the 64-bit result of F-000000000028 as Result C.

The low-word/high-word argument order is explicit. It must not be reversed merely because of a platform’s memory endianness; the representation describes separate argument values, not a memory serialization.

### Final calls and memory write

After obtaining Result C:

1. Extract the unsigned value `Result C >> 41`. Its range is zero through `0x007fffff`.
2. Shift the earlier zero intermediate word left by 25 positions; it remains zero.
3. Invoke F-000000000029 with Result C’s low word.
4. Invoke F-000000000030 and retain its 32-bit result.
5. Invoke F-000000000031 with that result and zero.
6. Invoke F-000000000032.
7. Compute the explicit store address.
8. Write the 16-bit value `0x00cd` to that address.
9. Reach the invalid-instruction termination marker.

The address calculation includes a signed right shift, but its operand is nonnegative: the previously extracted value is at most `0x007fffff`. Thus, under the stated model, signed and unsigned right shifts give the same result here.

Let **N** be the top twelve bits of Result C, interpreted as an unsigned integer:

**N = Result C >> 52**

The final address is:

**N + floor(N / 4)**

This is a raw address, not an offset from an explicit object or caller-provided buffer. Its range is zero through 5118 (`0x13fe`). The store accesses two bytes beginning at that address.

## Side effects

### Coprocessor transfers

Seven coprocessor-transfer operations occur, interleaved with calls as specified above. Their displayed selector fields and payloads are:

| Transfer | Displayed selector triple | Payload | Additional represented operands |
|---|---|---|---|
| 1 | `(0, 1, 7)` | Result of F-000000000007 | First and fifth entry coprocessor-related values |
| 2 | `(0, 5, 7)` | Zero under 32-bit interpretation | Second and fifth values |
| 3 | `(0, 6, 7)` | Result of F-000000000013 | Third and fifth values |
| 4 | `(0, 5, 7)` | `word(Result X × 2)` | Fourth and fifth values |
| 5 | `(0, 7, 7)` | Zero | Third and fifth values |
| 6 | `(0, 0, 7)` | Zero | Fifth value used twice |
| 7 | `(0, 2, 7)` | `word(Scaled X × 2⁸)` | Third and fifth values |

The architecture-specific meanings of these fields are unknown. They must not be labeled as cache maintenance, barriers, device commands, or other specific operations without additional evidence.

Their order is established by the representation. Whether they complete, trap, or impose memory-ordering effects is not established.

### Memory effects

The only explicit ordinary memory write in the body is the final 16-bit store.

It is not guarded by a null check, bounds check, alignment check, or access-permission check. Depending on Result C and the execution environment, it may target address zero, another low address, or an unaligned address.

The byte order of `0x00cd` is unspecified. On a little-endian target, the bytes would be `cd 00`; on a big-endian target, they would be `00 cd`.

Dependencies may perform additional reads or writes, including changes to globals or memory used by later dependencies. Their effects are unknown. The explicit call order must therefore be preserved; the supplied evidence does not justify treating apparently unused calls as removable.

### Ownership and resource state

No explicit allocation, release, ownership transfer, reference-count operation, or resource cleanup is visible in the body.

This does not establish that dependencies allocate nothing, release nothing, or preserve ownership. Likewise, none of the word arguments can safely be classified as a pointer, handle, or plain numeric value solely from the displayed call expressions.

The final store is not associated with an identified owned object or valid buffer extent.

## Dependencies

All 32 call sites are indirect. The first target depends on entry machine state. Subsequent targets are represented through arithmetic involving fixed address-like bases, including some function-like anchors.

The evidence does not provide:

- Callable implementations.
- Verified function boundaries.
- Calling conventions.
- Complete parameter lists.
- Failure contracts.
- Memory ownership rules.
- Global-state preservation guarantees.
- Whether any two call sites resolve to the same target.

Result-bearing sites have the following represented widths:

| Sites | Represented result |
|---|---|
| F-000000000007, F-000000000013, F-000000000015, F-000000000019 | 32 bits |
| F-000000000021, F-000000000027, F-000000000028 | 64 bits |
| F-000000000023, F-000000000024, F-000000000030 | 32 bits |

Other call results are not explicitly consumed. That does not prove those dependencies return no value.

The order and argument relationships in the Behavior section are the established dependency contract. Successful continuation additionally requires each reached target to be executable and to return compatibly with the represented path.

## Acceptance examples

These examples are conditional static acceptance criteria, not observed executions. They assume the represented instructions execute, each earlier dependency returns, and fixed-width arithmetic behaves as described.

### Third-argument bit selection

For third arguments whose low six bits match, the explicit Input word passed to F-000000000001 must match.

Examples:

- Third argument zero produces Input word zero.
- Third argument one produces `0x04000000`.
- Third argument 63 produces `0xfc000000`.
- Third argument 64 produces zero.

In every case, the second coprocessor payload and the first argument to F-000000000011 are zero.

These examples do not establish that complete executions are equivalent: dependencies may observe implicit state or other arguments.

### Result X truncation

If F-000000000015 returns `0x80000001`:

- The fourth coprocessor payload is `0x00000002`.
- Scaled X is `0x00000010`.
- F-000000000016 receives `0x00000002`, zero, `0x00000010`, zero.
- The seventh coprocessor payload is `0x00001000`.
- The first two arguments to F-000000000020 are `0x00001000` and `0x00010000`.

The intermediate masked word remains zero.

### Wide-result ordering

If Result A has high word `0x11223344` and low word `0x80000001`, F-000000000022 receives, in order:

- `0x80000001`;
- `0x11223344`;
- `0x00000002`.

### Final address boundaries

If Result C’s top twelve bits are:

| Top twelve bits | Final store address |
|---:|---:|
| 0 | 0 |
| 1 | 1 |
| 3 | 3 |
| 4 | 5 |
| 4095 | 5118 (`0x13fe`) |

For a fixed top-twelve-bit value, changing the lower 52 bits does not change the computed address. Such changes can still alter the argument to F-000000000029 and thereby affect dependency behavior.

A successful store writes `0x00cd` as a 16-bit value. An inaccessible or unsupported unaligned destination may prevent the endpoint from being reached; no recovery path is represented.

### Ordering constraint

If F-000000000032 does not return, the final explicit store is not reached through the represented sequence. If an earlier coprocessor transfer traps, later calls and the final store likewise are not reached through that sequence.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation corresponds to valid executable instructions throughout.
- Whether the recovered interface and signedness are accurate.
- The real architecture, execution mode, endianness, and alignment rules.
- The identities and contracts of all indirect dependencies.
- The semantic meaning of the coprocessor operations.
- The validity and accessibility of the final low-address destination.
- The exact runtime consequence of the truncated invalid-instruction endpoint.
- Any externally meaningful purpose for the sequence.

An initialization or hardware-control purpose is conceivable from the coprocessor operations, but it is not established and is not part of this contract.

The fixed-width arithmetic conclusions are conditional on the represented 32-bit dataflow being faithful. They do not repair or validate the warned-about instruction stream.

## Evidence

The supplied static material establishes a single represented path containing:

- Three explicit 32-bit parameters.
- Entry-state register dependencies.
- Thirty-two indirect call sites.
- Seven coprocessor-transfer operations.
- Three consumed 64-bit dependency results.
- One explicit 16-bit ordinary memory store.
- An invalid-instruction endpoint with truncated control-flow recovery.

The zero-valued arguments, shift truncations, and final address range follow algebraically from that representation under the stated bit-vector model.

No dependency bodies, verified runtime trace, architectural specification, or execution results were supplied. No runtime tests have been performed.
