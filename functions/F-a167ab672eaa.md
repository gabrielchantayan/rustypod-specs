<!-- Trusted provenance: F-a167ab672eaa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a167ab672eaa

## Purpose

F-a167ab672eaa is represented as a nonterminating sequence of indirect calls, byte accesses, and 32-bit word accesses. It repeatedly derives addresses and stored values from entry state and dependency results, then feeds selected results into the next iteration.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish that the accessed locations are ordinary objects, device registers, allocator metadata, or any other named data structure. The repeated offsets and scaling operations are insufficient to assign those meanings.

The contract below describes the visible access and call sequence. It does not assume that unresolved dependencies preserve memory, preserve implicit machine state, return normally, or leave earlier writes intact.

## Interface

### Explicit inputs

The displayed interface has four arguments:

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | Passed to the first indirect call of each iteration. Replaced near the end of the iteration by a dependency result. |
| Second argument | 32 bits | Passed alongside the first argument. Later replaced by a dependency result, used in a word store, and shifted before the next iteration. |
| Third argument | 32 bits | No visible use. |
| Fourth argument | 32 bits | Used in an early word-store address and a subsequent call argument. Replaced near the end of each iteration. |

“No visible use” does not establish anything about an external ABI or caller obligations beyond the supplied representation.

### Implicit inputs

The first iteration also depends on state not supplied through the four displayed arguments:

- An 8-bit value used by the first byte store.
- A 32-bit address-like value used to locate a selector byte.
- A 32-bit value used in shifts and word stores.
- A link-register-like value used to select the first indirect-call target.
- An additional register result forwarded to an indirect call without a displayed ordinary assignment.
- A global 32-bit value read partway through each iteration.
- Memory and executable targets reached through the computed addresses.

These are genuine prerequisites to predicting the represented execution. Their initialization and external calling convention are **documented_only**.

### Outputs and completion

There is no represented normal return, return value, or termination condition. Each completed iteration enters another iteration.

Observable outputs consist of:

- Indirect dependency invocations and their explicitly represented arguments.
- Ordered byte and word stores.
- Reads that select later addresses or values.
- Carried state for the next iteration.

A dependency that does not return, an invalid instruction target, or an invalid memory access may prevent further progress. No local recovery path is displayed.

### Width and arithmetic conventions

The representation explicitly distinguishes 8-bit accesses, 32-bit accesses, and several 64-bit dependency results.

For the arithmetic descriptions below:

- “Low byte” means the least significant eight bits.
- “Low word” and “high word” mean the lower and upper 32-bit halves of a represented 64-bit result.
- Right shifts of unsigned quantities are logical.
- Address arithmetic and left shifts are described using their low 32 bits, conditional on a 32-bit machine interpretation.

The last convention is an interpretation of the represented machine operations, not a claim that equivalent signed arithmetic in a high-level language is defined. Address-space width, alignment behavior, and endianness are not independently established.

## Behavior

### Dependency notation

All dependencies are unresolved indirect calls. Their target identities are not provided by the private dependency map.

To identify call sites without inventing semantic names, this contract uses opaque IDs of the form **F-D01-01**: the first number identifies a phase below, and the second identifies the call’s ordinal position within that phase. Distinct IDs identify call sites, not necessarily distinct target functions.

A call described as having “no displayed arguments” may still consume implicit machine state. Its effects are not assumed empty.

### Ordered iteration

The following phases occur in order, provided every preceding call and access completes. Symbols used here describe values, not object types or ownership.

#### 1. Initial selector write and scaled word write

Let the iteration’s implicit byte, selector-base, and numeric inputs be \(e\), \(S\), and \(R\). Let the fourth argument be \(Q\).

1. F-D01-01 receives the first and second arguments. Its target is derived from the incoming link-register-like value.
2. F-D01-02 through F-D01-05 have no displayed arguments. The last supplies a 32-bit result \(a\).
3. Read one byte at \(S+1\), zero-extend it, add 17, and store \(e\) at that address.
4. F-D01-06 receives \(a\ll2\).
5. F-D01-07 through F-D01-10 have no displayed arguments.
6. Store the 32-bit value \(R\ll12\) at address \(Q\times4096+20\).
7. F-D01-11 receives \(Q\ll14\).
8. F-D01-12 has no displayed arguments.
9. F-D01-13 receives \((R\ll12)\ll28\) and the additionally represented register result.

Under 32-bit truncation, the first argument of F-D01-13 is zero: the combined shift discards all original bits. The second argument’s provenance is unresolved.

#### 2. Second word write and selector update

1. F-D02-01 through F-D02-04 have no displayed arguments. The last supplies \(b\).
2. Store \(R\ll12\) at \(4b+20\).
3. F-D02-05 receives \(4b\).
4. F-D02-06 through F-D02-09 have no displayed arguments. The last supplies \(c\). Retain \(C=4c\).
5. Read a fresh selector byte at \(S+1\). At that selector plus 17, store the low byte of
   \[
   ((R\ll12)\mathbin{\&}\text{0x3fffffff})\gg20.
   \]
6. F-D02-10 receives \(R\ll24\).
7. F-D02-11 has no displayed arguments and supplies a word forwarded to F-D02-12.
8. F-D02-12 receives that word and \(C\ll28\).
9. F-D02-13 has no displayed arguments and supplies a 64-bit result.
10. F-D02-14 receives its low and high words separately.
11. F-D02-15 and F-D02-16 have no displayed arguments; the last supplies \(d\).
12. Read the selector at \(S+1\) again and store zero at selector plus 17.
13. F-D02-17 receives \(d\ll2\).

Each selector read is independent. Intervening dependencies may change the selected destination.

#### 3. Word propagation

1. F-D03-01 through F-D03-04 have no displayed arguments; the last supplies \(h\).
2. Retain \(H=4h\), and store \(H\) at \(4096C+20\).
3. F-D03-05 receives four words: \(1024C\), \(1024C\), \(c\ll14\), and \(H\).
4. F-D03-06 through F-D03-09 have no displayed arguments; the last supplies \(j\).
5. Store \(H\) at \(4j+20\).
6. F-D03-10 receives \(4j\).

The first two arguments of F-D03-05 are equal as represented. Their equality does not imply anything about what the callee does with them.

#### 4. Two scaled-value propagation stages

1. F-D04-01 through F-D04-04 have no displayed arguments; the last supplies \(k\).
2. Retain \(K=4k\). Store \(K\) at \(16k+20\).
3. F-D04-05 receives \(K\) three times.
4. F-D04-06 through F-D04-09 have no displayed arguments; the last supplies \(m\).
5. Store \(K\) at \(4m+20\).
6. F-D04-10 receives \(4m\).
7. F-D04-11 through F-D04-14 have no displayed arguments; the last supplies \(n\).
8. Retain \(N=4n\) and \(B=n\times2^{30}\). Store \(N\) at \(16n+20\).
9. F-D04-15 receives \(N\) three times.
10. F-D04-16 through F-D04-19 have no displayed arguments; the last supplies \(p\).
11. Store \(N\) at \(4p+20\).
12. F-D04-20 receives \(4p\).

Under the 32-bit interpretation, \(B\) depends only on the low two bits of \(n\).

#### 5. Repeated byte clearing and indexed word read

1. F-D05-01 through F-D05-04 have no displayed arguments; the last supplies an unsigned word \(u\).
2. Store zero at \(B+17\), then zero at \(B+1\).
3. F-D05-05 receives \(u\ll2\).
4. F-D05-06 has no displayed arguments and supplies a word.
5. F-D05-07 receives that word.
6. F-D05-08 through F-D05-11 have no displayed arguments; the last supplies \(q\).
7. Store zero at \(B+17\), then zero at \(B+1\), again.
8. F-D05-12 receives \(q\ll2\).
9. F-D05-13 through F-D05-16 have no displayed arguments; the last supplies \(r\).
10. Retain \(T=4r\). Store \(p\ll31\) at \(16r+20\).
11. Read a 32-bit word \(V\) at
    \[
    ((u\mathbin{\&}\text{0x3fffffff})\gg20)+20.
    \]
12. F-D05-17 receives \(T\).
13. F-D05-18 has no displayed arguments and supplies a new second-argument value \(P\).
14. F-D05-19 receives \((V\gg22)\ll28\).

The indexed read’s computed address ranges from 20 through 1043 before any machine-specific access restrictions. It is not necessarily word-aligned.

#### 6. Zero word store and selector-based byte updates

1. F-D06-01 through F-D06-04 have no displayed arguments; the last supplies \(s\).
2. Store a zero word at \(4s+20\).
3. F-D06-05 receives \(4s\).
4. F-D06-06 through F-D06-09 have no displayed arguments; the last supplies an unsigned word \(w\).
5. Read a selector byte at \(B+1\). Store the low byte of \(w\ll2\) at selector plus 17.
6. F-D06-10 receives \(w\ll2\).
7. F-D06-11 has no displayed arguments and supplies a word.
8. F-D06-12 receives that word and \(w\ll30\).
9. F-D06-13 through F-D06-16 have no displayed arguments; the last supplies \(t\).
10. Read and retain a fresh selector byte \(z\) from \(B+1\).
11. Store the low byte of \((w\mathbin{\&}\text{0x3fffffff})\gg20\) at \(z+17\).
12. F-D06-17 receives \(t\ll2\).
13. F-D06-18 through F-D06-21 have no displayed arguments.
14. Store \(P\ll2\) at \(z+20\).
15. Read and retain the global 32-bit value \(G\).

The word-store address uses retained selector \(z\), not a new selector read after the intervening calls.

#### 7. Global-derived byte write and selector clearing

1. F-D07-01 receives \(T\).
2. F-D07-02 through F-D07-05 have no displayed arguments; the last supplies a 64-bit result with low word \(l\) and high word \(h_1\).
3. Retain \(L=4l\).
4. Read a selector byte at \(B+1\). Store the low byte of \(G\gg22\) at selector plus 17.
5. F-D07-06 receives \(h_1\).
6. F-D07-07 has no displayed arguments and supplies a word.
7. F-D07-08 receives that word and \(L\ll28\).
8. F-D07-09 through F-D07-12 have no displayed arguments; the last supplies \(v\).
9. Read a fresh selector byte at \(B+1\), and store zero at selector plus 17.
10. F-D07-13 receives \(v\ll2\).
11. F-D07-14 through F-D07-17 have no displayed arguments; the last supplies \(x\).
12. F-D07-18 receives \(x\ll2\) twice.

#### 8. Final word propagation

1. F-D08-01 through F-D08-04 have no displayed arguments; the last supplies \(y\).
2. Retain \(Y=4y\). Store \(Y\) at \(4096L+20\).
3. F-D08-05 receives \(L\ll14\).
4. F-D08-06 through F-D08-09 have no displayed arguments; the last supplies \(a_1\).
5. Store \(Y\) at \(4a_1+20\).
6. F-D08-10 receives \(4a_1\).
7. F-D08-11 through F-D08-14 have no displayed arguments; the last supplies a 64-bit result with low word \(b_1\) and high word \(b_2\).
8. Retain \(J=4b_1\).
9. Read a selector byte at \(16384L+1\). Store the low byte of \((Y\mathbin{\&}\text{0x3fffffff})\gg20\) at selector plus 17.
10. F-D08-15 receives \(b_2\).
11. F-D08-16 has no displayed arguments and supplies a word.
12. Retain \(D=J\times2^{28}\). F-D08-17 receives the supplied word.
13. F-D08-18 has no displayed arguments and supplies a 64-bit result.
14. F-D08-19 receives its low and high words separately.
15. F-D08-20 and F-D08-21 have no displayed arguments; the last supplies \(c_1\).
16. Read a selector byte at \(D+1\), and store zero at selector plus 17.
17. F-D08-22 receives \(c_1\ll2\).

#### 9. Next-iteration state

1. F-D09-01 through F-D09-04 have no displayed arguments; the last supplies \(d_1\).
2. The next implicit numeric input becomes \(4d_1\).
3. The next selector-base input becomes \(4J\).
4. Read a selector byte at \(4J+1\), and store zero at selector plus 17.
5. The next fourth argument becomes \(4d_1\).
6. F-D09-05 receives \(J\ll10\).
7. F-D09-06 has no displayed arguments and supplies the next first argument.
8. The next second argument becomes \(P\ll28\).
9. The next implicit byte input becomes zero.
10. Execution repeats, with the first call target derived from an internally established continuation value rather than the original entry value.

There is no represented delay, exit test, or iteration bound.

## Side effects

The direct memory effects are byte stores, 32-bit stores, selector-byte reads, one explicitly indexed word read, and one global word read per completed iteration.

Selector-based byte destinations lie between addresses 17 and 272 because a selector is an unsigned byte. The selector is an address component, not a displayed full-width pointer.

The two repeated clearing pairs in phase 5 are separate observable stores. They cannot be collapsed on the assumption that the intervening calls do nothing. Likewise, clearing \(B+1\) does not guarantee later selector reads return zero: dependencies may modify that location.

No allocation, release, ownership transfer, synchronization, bounds check, or explicit error report is established. The function does not visibly acquire ownership of the memory it accesses. Dependencies may have additional effects, including allocation or release, but none can be assigned from this evidence.

Earlier effects remain performed if a later call fails or does not return. There is no displayed rollback.

## Dependencies

The F-D identifiers above cover the represented indirect-call sites. Their contracts are unknown:

- Target validity and instruction-state requirements are **documented_only**.
- Displayed argument widths and consumed result widths are described above.
- Additional arguments, register inputs, global effects, and memory effects are **documented_only**.
- Normal return, exception behavior, reentrancy, and ownership effects are **documented_only**.
- Distinct sites may resolve to the same target; equal-looking call sequences do not establish target equivalence.

Nothing in this contract permits removing a call merely because its displayed result is unused.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial byte selection.** If the first selector read returns 7, the first direct byte store targets address 24 and writes the incoming implicit byte. A later selector read returning 9 targets address 26 instead.

2. **Early word store.** Under 32-bit arithmetic, if \(Q=2\) and \(R=3\), phase 1 stores 12288 at address 8212. Its later doubly shifted argument is zero.

3. **Narrowed byte field.** If \(w=\text{0x12345678}\), phase 6’s first byte store writes `0xe0`; its second byte store writes `0x23`. These are low-byte truncations of different expressions.

4. **Small indexed read.** If \(u=\text{0xffffffff}\), phase 5 reads its word at address 1043. Whether that unaligned access succeeds is not established.

5. **Retained selector.** If phase 6 captures \(z=8\), its later word store targets address 28 even if a dependency subsequently changes the selector byte.

6. **Wrapped high-bit scaling.** Under 32-bit arithmetic, \(n=5\) gives \(B=\text{0x40000000}\), not an unbounded mathematical product.

7. **No implicit completion.** If all accesses succeed and all dependencies return, finishing phase 9 begins another iteration rather than returning.

## Uncertainties

### Established static evidence

The representation establishes the displayed argument uses, access widths, relative operation order, unconditional repetition, and next-iteration assignments.

### Inferred intent

Repeated shifts, masks, offsets, and field-sized stores suggest packed-state manipulation. That interpretation is not strong enough to name a format, object layout, or subsystem.

### Documented-only limitations

- Actual instruction semantics and faithful recovery of the displayed control flow.
- Initial implicit-register values and the extra forwarded register result.
- Executable validity of computed call targets.
- Memory mapping, permissions, alignment handling, and byte order.
- Meaning and overlap of the global storage.
- Callee preservation of globals, memory, and machine state.
- Whether any dependency intentionally terminates execution.
- Whether apparent low-address accesses are valid in the deployment environment.

These unknowns prevent an unconditional claim that the represented function can execute successfully from an ordinary caller-provided state.

## Evidence

This contract is based solely on the supplied static representation. The dependency map identifies F-a167ab672eaa itself but provides no resolved dependency contracts.

The supplied annotations report removed unreachable material and overlapping global symbols. Those annotations do not establish additional reachable behavior or a stable global layout.

No implementation access, runtime execution, smoke checks, or tests have been performed. The acceptance examples are arithmetic and ordering consequences of the representation, conditional on the stated machine interpretation.
