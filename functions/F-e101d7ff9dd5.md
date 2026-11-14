<!-- Trusted provenance: F-e101d7ff9dd5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e101d7ff9dd5

## Purpose

The supplied static representation describes a procedure that performs a long, ordered sequence of indirect calls, 32-bit memory stores, byte reads, and byte stores, then enters an unconditional loop.

No application-level purpose is established. In particular, the evidence does not establish that the procedure constructs an object, serializes a message, allocates storage, or implements a recognizable protocol. Repeated scaling and fixed-offset stores are observable arithmetic patterns, not proof of a data structure.

This contract describes the recovered path **conditionally**: execution must reach each operation, and preceding indirect calls must return sufficiently normally for that operation to execute. The supplied material reports invalid instruction data and omitted unreachable regions. Consequently, the recovered path is evidence of a possible operational sequence, not proof that the original executable follows it faithfully.

There is no recovered normal return.

## Interface

### Explicit inputs

The recovered interface has three explicit inputs:

| Input | Width and interpretation | Directly evidenced use |
|---|---|---|
| Payload | 32-bit uninterpreted bit pattern | Value of the first 32-bit store |
| Selector | 32-bit value displayed as signed | Used in two early address calculations and as one stored value |
| Control | Unsigned 32-bit value | Shifted right twice by seven positions; the resulting value is passed to a dependency |

The selector is not established to be an index, pointer, object handle, or allocation size. Its two address formulas use different scaling factors.

### Implicit inputs

Execution also depends on:

- An incoming 32-bit register-derived value, called the **ambient word** below.
- An incoming control-transfer value used to derive the first indirect-call target.
- Results supplied by indirect calls.
- A secondary register result observed after one call, independently of its ordinary result.
- Memory read during the byte-writing phase.
- Any machine state consulted or modified by dependencies.

These requirements are not represented by the three explicit parameters. A portable three-argument interface cannot be inferred from this evidence alone.

### Outputs and completion

No explicit result value is established. Observable outputs consist of memory effects, dependency invocations, and continued execution.

If every recovered indirect call returns and every memory access succeeds, execution reaches an infinite loop. There is no recovered success status, failure status, termination condition, or cleanup exit.

### Arithmetic notation

The following names describe values for this contract; they do not identify implementation variables.

- \(A\): ambient word.
- \(S\): selector bit pattern.
- \(P\): payload.
- \(C\): control word.
- \(\operatorname{low8}(x)\): the least significant eight bits of \(x\).
- \(\operatorname{store32}(a,v)\): one recovered 32-bit store of \(v\) at address \(a\).

For the conditional 32-bit machine interpretation, products, left shifts, and address additions below retain their low 32 bits. This interpretation does **not** assert that overflowing signed arithmetic would be valid in a higher-level language.

Right shifts of explicitly unsigned quantities are logical. Byte loads are unsigned. Endianness and unaligned-access behavior are not established.

## Behavior

### Dependency notation

Opaque IDs below identify unresolved **call sites**, not resolved implementations. Matching behavior or nearby target expressions does not establish that two sites invoke the same function.

A five-call chain identified as `F-aNN-1` through `F-aNN-5` means:

1. The first site receives the arguments listed in the table.
2. The next four sites have no explicit arguments in the recovered representation.
3. The ordinary result of the fifth site supplies the named result, unless stated otherwise.

“No explicit arguments” does not establish that a dependency ignores registers, stack contents, globals, or other ambient state.

Three chains have a nine-call form:

- `F-a03-1` through `F-a03-9`
- `F-a05-1` through `F-a05-9`
- `F-a11-1` through `F-a11-9`
- `F-a13-1` through `F-a13-9`

For each, the first site receives the table’s initial arguments. The second site produces a 32-bit token consumed by the third. The fourth produces another token consumed by the fifth. Sites six through nine have no explicit arguments; the ninth supplies the chain result. The fifth site’s additional argument, where present, is specified below.

No intervening dependency effects may be assumed absent.

### Ordered 32-bit phase

The following rows occur in order. Each row’s memory store precedes that row’s calls, and the complete call sequence precedes the next row’s store.

| Step | Direct store before calls | Dependency calls and result |
|---|---|---|
| 1 | \(\operatorname{store32}(4S+20,P)\) | Five-call chain `F-a01-1`–`F-a01-5`; first argument \(A \gg 8\); result \(r_1\) |
| 2 | \(\operatorname{store32}(S+20,S)\) | Five-call chain `F-a02-1`–`F-a02-5`; first argument \(4r_1\); result \(r_2\) |
| 3 | \(\operatorname{store32}(16r_2+20,4r_2)\) | Nine-call chain `F-a03-1`–`F-a03-9`; first argument \(16r_2\); fifth site receives its token and \(A \gg 8\); result \(r_3\) |
| 4 | \(\operatorname{store32}(16r_2+20,4r_2)\) | Five-call chain `F-a04-1`–`F-a04-5`; first argument \(4r_3\); result \(r_4\) |
| 5 | \(\operatorname{store32}(16r_4+20,4r_4)\) | Nine-call chain `F-a05-1`–`F-a05-9`; first arguments \(4r_4,8A\); fifth site receives its token and \(A \gg 8\); result \(r_5\) |
| 6 | \(\operatorname{store32}(8A+20,4r_4)\) | Five-call chain `F-a06-1`–`F-a06-5`; first argument \(4r_5\); result \(r_6\) |
| 7 | \(\operatorname{store32}(16r_6+20,4r_6)\) | Five-call chain `F-a07-1`–`F-a07-5`; first arguments \(8A,8A,A \gg 8,C \gg 14\); result \(r_7\) |
| 8 | \(\operatorname{store32}(8A+20,8A)\) | Five-call chain `F-a08-1`–`F-a08-5`; first argument \(4r_7\); a secondary register result after the fifth call supplies \(q\) |
| 9 | \(\operatorname{store32}(2^{26}q+20,2^{24}q)\) | Five-call chain `F-a09-1`–`F-a09-5`; first argument \(2^{24}q\); result \(r_9\) |
| 10 | \(\operatorname{store32}(4r_9+20,2^{24}q)\) | Five-call chain `F-a10-1`–`F-a10-5`; first argument \(4r_9\); result \(r_{10}\) |
| 11 | \(\operatorname{store32}(16r_{10}+20,4r_{10})\) | Nine-call chain `F-a11-1`–`F-a11-9`; first argument \(4r_{10}\); fifth site receives only its token explicitly; result \(r_{11}\) |
| 12 | \(\operatorname{store32}(4r_{11}+20,4r_{10})\) | Five-call chain `F-a12-1`–`F-a12-5`; first argument \(4r_{11}\); result \(r_{12}\) |
| 13 | \(\operatorname{store32}(16r_{12}+20,4r_{12})\) | Nine-call chain `F-a13-1`–`F-a13-9`; first argument \(4r_{12}\); fifth site receives only its token explicitly; result \(r_{13}\) |
| 14 | \(\operatorname{store32}(4r_{13}+20,A \gg 16)\) | Five-call chain `F-a14-1`–`F-a14-5`; first argument \(4r_{13}\); result \(r_{14}\) |
| 15 | \(\operatorname{store32}(16r_{14}+20,4r_{14})\) | Five-call chain `F-a15-1`–`F-a15-5`; first arguments \(16r_{14},16r_{14},A \gg 15\); result \(r_{15}\) |
| 16 | \(\operatorname{store32}(16r_{14}+20,16r_{14})\) | Five-call chain `F-a16-1`–`F-a16-5`; first argument \(4r_{15}\); result \(r_{16}\) |

The repeated store in step 4 is a separate observable operation. Dependencies between steps 3 and 4 may have modified that location.

The control contribution in step 7 is \(C \gg 14\), not \(C \gg 7\): the recovered path shifts the unsigned input by seven positions twice.

The secondary result \(q\) is not established as the ordinary return value of `F-a08-5`. Its availability depends on machine-level calling behavior absent from the explicit interface.

### Byte-access phase

Define the byte-selector address

\[
B=\bigl((4r_{12}\mathbin{\&}\texttt{0x7fffffff})\gg7\bigr)+1.
\]

A **selected-byte write** means:

1. Read one unsigned byte at \(B\).
2. Add 17 to that byte’s numeric value.
3. Store one byte at the resulting address.

Thus its destination lies between 17 and 272 inclusive. The byte read is an address selector, not a full-width pointer. Every selected-byte write below performs a fresh read; a previously loaded selector is not reused.

The sequence is:

1. Perform a selected-byte write of \(\operatorname{low8}(4r_{16})\).
2. Invoke `F-a17-1` through `F-a17-7` in order:
   - First site receives \(4r_{16}\) three times.
   - Second site supplies a token to the third.
   - Fourth through seventh sites have no explicit arguments.
   - Seventh site supplies result \(r_{17}\).
3. Perform a selected-byte write of
   \[
   \operatorname{low8}((r_{16}\mathbin{\&}\texttt{0x3fffffff})\gg20).
   \]
4. Invoke five-call chain `F-a18-1`–`F-a18-5`, initially passing \(4r_{17}\), obtaining \(r_{18}\).
5. Store \(\operatorname{low8}(4r_{18})\) at address \((A\gg23)+28\).
6. Perform a selected-byte write of zero.
7. Invoke `F-a19-1` through `F-a19-7`:
   - First site receives \(A\gg23\).
   - Second has no explicit arguments.
   - Third receives \((r_{18}\mathbin{\&}\texttt{0x3fffffff})\gg6\).
   - Fourth through seventh have no explicit arguments.
   - Seventh supplies \(r_{19}\).
8. Perform another selected-byte write of zero.
9. Invoke five-call chain `F-a20-1`–`F-a20-5`, initially passing \(4r_{19}\), obtaining \(r_{20}\).

The two adjacent zero-writing stages cannot be collapsed: calls separate them, and the selector byte may change.

### Loop preparation and perpetual phase

Define

\[
L=(r_{20}\mathbin{\&}\texttt{0x3fffffff})\gg25
\]

and

\[
D=1024(4r_{18})+1.
\]

Initially, \(L\) is between 0 and 31 inclusive.

A **loop-selected write** reads one unsigned byte at \(D\), adds 17, and writes zero to that selected address. Each occurrence performs its own read.

Before entering the loop:

1. Perform one loop-selected write.
2. Invoke `F-a21-1` with \(2L\).
3. Invoke `F-a21-2` without explicit arguments; its result becomes the current loop value.

Each iteration then:

1. Invokes five-call chain `F-a22-1`–`F-a22-5`. Its first site receives the current loop value and \(L\gg8\). Its final result is \(y\).
2. Performs a loop-selected write.
3. Invokes five-call chain `F-a23-1`–`F-a23-5`. Its first site receives \(4y\). Its final result is \(z\).
4. Replaces the current loop value with \(4z\), and replaces \(L\) with \(z\ll26\).
5. Repeats unconditionally.

On the first iteration, the second argument to `F-a22-1` is necessarily zero because the initial \(L\le31\). On later iterations, under 32-bit arithmetic, that argument is \((z\mathbin{\&}63)\ll18\), using the preceding iteration’s \(z\).

## Side effects

### Memory layout and access order

The direct word stores use addresses formed from numeric inputs and dependency results, with a fixed addition of 20. No allocation base, enclosing object size, or field ownership is established.

The byte phase contains explicit byte reads. The initial word phase contains no explicit data loads, apart from state needed to establish indirect calls. Dependencies may perform arbitrary additional reads and writes.

Address formulas may alias:

- Different word-store destinations may coincide.
- Byte stores may overlap prior word stores.
- A byte store may change a later selector read.
- A selector destination may equal its own selector address.
- Dependency memory may overlap directly accessed memory.

No disjointness, atomicity, volatility, synchronization, or memory-order guarantee is established.

### Ownership and lifetime

The evidence shows no identifiable allocation, release, retain, or ownership-transfer operation. Passing a value to an unknown dependency does not establish borrowing or transfer.

Dependencies must not be assumed to preserve globals, preserve pointed-to memory, or refrain from releasing storage.

### Preconditions and errors

Following the recovered path requires executable indirect targets and accessible memory at every address actually used. Alignment requirements are platform-dependent and undocumented.

There are no recovered bounds checks, null checks, error branches, or rollback operations. A fault or non-returning dependency can prevent later operations. Earlier writes are not shown being undone.

## Dependencies

All dependency IDs above are unresolved, indirect call-site identities. Their implementations, signatures, side effects, ownership rules, and error conventions are unknown.

The first target depends on incoming control-transfer state. Later targets are represented through address-derived expressions, including one intermediate code-address value. These expressions do not establish a trustworthy public dependency graph.

For conformance to this conditional contract:

- Preserve call order and explicitly supplied argument bit patterns.
- Preserve the distinctions between ordinary results, intermediate tokens, and the secondary register result.
- Do not merge call sites merely because their recovered invocation patterns match.
- Do not assume unused ordinary results imply effect-free calls.
- Do not assume calls without explicit arguments are independent of ambient state.

Exact target resolution and authentic machine calling conventions remain `documented_only`.

## Acceptance examples

These examples are conditional deductions, not executed tests.

1. **Initial scaled and unscaled destinations.**  
   With selector 3 and payload `0x11223344`, the first direct store is a 32-bit write at address 32. If the first five calls return normally, the next direct store writes 3 at address 23. The second destination is not 32.

2. **Control-bit discard.**  
   Control values `0x00003fff` and zero both contribute zero to the fourth argument in step 7. Control `0x00004000` contributes 1. This establishes only that argument equality, not whole-execution equivalence.

3. **Repeated store remains observable.**  
   If \(r_2=2\), steps 3 and 4 each write 8 at address 52. If intervening dependencies alter address 52, step 4 writes 8 again.

4. **Selector reload matters.**  
   If the byte at \(B\) is 5 during one selected-byte write, that write targets address 22. If intervening activity changes it to 9, the next selected-byte write targets address 26.

5. **Byte truncation.**  
   If \(r_{16}=65\), the first byte-phase value is 4, because \(4\times65=260\) and only the low byte is stored.

6. **Distinct masks and shifts.**  
   If \(r_{16}=\texttt{0x3ff00000}\), the second selected-byte write stores `0xff`: the masked, shifted value is `0x3ff`, then narrowed to eight bits.

7. **Loop argument transition.**  
   The first loop’s second argument is zero. If the first iteration produces \(z=1\), the next iteration’s current value is 4 and its second argument is `0x00040000`.

8. **No finite successful completion.**  
   With all calls returning and accesses succeeding indefinitely, the recovered behavior continues looping rather than returning a status.

## Uncertainties

### Established static evidence

The supplied representation supports the explicit store widths, byte-load widths, arithmetic expressions, ordering, call-result propagation, and unconditional loop described above.

### Inferred intent

The repeated fixed-offset stores may suggest a common transformation pattern. That observation does not establish an object format or application purpose.

### `documented_only`

The following remain unresolved:

- Whether the recovered path faithfully represents executable control flow.
- Effects of invalid instruction data and omitted regions.
- Exact architecture, ABI, endianness, alignment rules, and fault behavior.
- Authentic indirect targets and dependency signatures.
- Whether apparent ambient values are stable machine values across calls.
- Valid address ranges, including the low addresses selected by bytes.
- Ownership, lifetime, concurrency, and external error handling.
- Any termination behavior outside the recovered unconditional loop.

## Evidence

This document uses only the supplied static representation and its warnings. No implementation access, disassembly validation, runtime execution, smoke checks, or tests were performed.

Arithmetic ranges and acceptance examples are deductions under the stated conditional 32-bit interpretation. They are not runtime observations. No provenance hash is asserted.
