<!-- Trusted provenance: F-bc5c8bebc935; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc5c8bebc935

## Purpose

F-bc5c8bebc935 performs an ordered sequence of indirect calls and computed-address memory accesses. The visible body contains thirteen four-byte writes, two one-byte writes, **three unsigned four-byte read expressions**, and two unsigned-byte reads. Two word-read expressions supply arguments to F-ext-021; the third occurs after F-ext-050 and supplies a value retained for a later write.

The function combines its fourth explicit input, an implicit incoming value, dependency results, and memory contents. No application-level purpose is established. In particular, the address calculations do not establish objects, allocated buffers, devices, tables, or ownership relationships.

The visible path ends at an invalid-instruction boundary rather than an ordinary return. This contract describes the visible execution prefix, conditional on preceding calls returning and memory accesses completing. It does not establish the behavior of any execution beyond that boundary.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First input | 32 bits, otherwise uninterpreted | Passed to F-ext-001. |
| Second input | 32 bits, otherwise uninterpreted | No explicit reference in the visible body. |
| Third input | 32 bits, otherwise uninterpreted | No explicit reference in the visible body. |
| Fourth input, denoted \(P\) | Unsigned 32 bits | Used in address calculations, stored values, and dependency arguments. |

The unused-input observations concern only the visible expressions. They do not establish that the second and third inputs are irrelevant to the actual calling convention or to state consumed by indirect dependencies.

### Implicit inputs

The visible body also depends on:

- Incoming control state used to compute the target of F-ext-001.
- An incoming 32-bit register value, denoted \(V\), used by the first direct write.
- Selected dependency results.
- Memory at computed addresses.
- Any additional register, stack, global, or external state consumed by unresolved dependencies.

Neither incoming control state nor \(V\) is initialized within the visible body. Their meaning and validity requirements are unknown. The represented use of \(V\) after several calls does not independently establish the actual machine-level register-preservation rules.

### Output and termination

No ordinary return value or normal return path is established.

After F-ext-084, the supplied representation reaches an invalid-instruction termination marker. Whether this corresponds to a runtime trap, an incorrectly recovered boundary, or some other continuation is **documented_only**.

### Preconditions for completing the visible prefix

Completion requires that:

- Every indirect target be executable and compatible with the actual calling convention.
- Every dependency before a later event return normally.
- Each four-byte address support the specified read or write.
- Each byte address support the specified read or write.
- Arithmetic, address formation, alignment, and integer conversions have a concrete interpretation consistent with the recovered operations.

No visible check enforces these conditions. They are traversal requirements, not guarantees supplied by the function.

## Behavior

### Arithmetic and memory notation

The following notation describes observable relationships, not recovered object types:

- \(Q(x)\): the low 32 bits of \(4x\).
- \(H(x)\): the low 32 bits of \(16x\).
- \(W[a]\): a four-byte memory access at numeric address \(a\).
- \(B[a]\): a one-byte memory access at numeric address \(a\).
- \(R_1\) through \(R_{15}\): selected dependency results identified below.

Unsigned 32-bit multiplication and left shifts discard bits above bit 31. Logical right shifts introduce zero bits. A byte store retains only the low eight bits of its value.

Several selected dependency results are represented as signed 32-bit integers. For them, \(Q\), \(H\), and any stated modular results describe a **conditional 32-bit machine-bitvector interpretation**. They do not establish defined signed-overflow or signed-left-shift semantics. Address addition also remains subject to the unresolved target address model.

All numeric offsets below are byte offsets. A four-byte access covers four consecutive bytes beginning at the stated address; no field layout beyond these access widths and offsets is established. Endianness is unknown.

### Ordered dependency groups

The 84 visible call sites are assigned opaque IDs F-ext-001 through F-ext-084 in execution order. These IDs distinguish call sites, not necessarily distinct runtime targets.

Each row below describes five consecutive calls. The first call receives the displayed explicit arguments; the following four have no explicit arguments shown. The final call in the row supplies the selected result, where applicable.

| Calls | First call’s explicit arguments | Selected final result |
|---|---|---|
| F-ext-001–F-ext-005 | First input | \(R_1\) |
| F-ext-006–F-ext-010 | \(Q(R_1)\) | No explicit later use |
| F-ext-011–F-ext-015 | \(P\times2^{26}\), reduced to 32 bits | \(R_2\) |
| F-ext-016–F-ext-020 | \(Q(R_2)\) | \(R_3\) |
| F-ext-021–F-ext-025 | Three arguments specified below | \(R_4\) |
| F-ext-026–F-ext-030 | \(Q(R_4)\) | \(R_5\) |
| F-ext-031–F-ext-035 | \(Q(R_5), Q(R_5), P\ll26\) | \(R_6\) |
| F-ext-036–F-ext-040 | \(Q(R_6)\) | \(R_7\) |
| F-ext-041–F-ext-045 | \(Q(R_7)\) | \(R_8\) |
| F-ext-046–F-ext-050 | \(Q(R_8)\) | \(R_9\) |
| F-ext-051–F-ext-055 | \(Q(R_9)\) | \(R_{10}\) |
| F-ext-056–F-ext-060 | \(Q(R_{10})\) | \(R_{11}\) |
| F-ext-061–F-ext-065 | Three arguments specified below | \(R_{12}\) |
| F-ext-066–F-ext-070 | \(Q(R_{12})\) | \(R_{13}\) |

“No explicit arguments shown” does not mean that a dependency consumes no inputs. Additional machine-state inputs and dependency side effects remain unresolved.

### Four-byte writes and reads

The following direct accesses occur between the call groups. Row order is execution order. Each listed write follows the identified call and precedes the next call.

| Boundary | Direct memory effect |
|---|---|
| After F-ext-005 | Store \(V\) in \(W[Q(R_1)+20]\). |
| After F-ext-010 | Store the low 32 bits of \(P\times2^{26}\) in \(W[(P\times2^{28})+20]\). The unsigned product used in the address is also reduced to 32 bits. |
| After F-ext-015 | Store \((P\mathbin{\&}\texttt{0x1fffffff})\gg9\) in \(W[Q(R_2)+20]\). |
| After F-ext-020 | Store \(Q(R_3)\) in \(W[H(R_3)+20]\). |
| After F-ext-025 | Store the previously computed \(Q(R_3)\) in \(W[Q(R_4)+20]\). |
| After F-ext-030 | Store \(Q(R_5)\) in \(W[H(R_5)+20]\). |
| After F-ext-035 | Store the previously computed \(Q(R_5)\) in \(W[Q(R_6)+20]\). |
| After F-ext-040 | Store \(Q(R_7)\) in \(W[H(R_7)+20]\). |
| After F-ext-045 | Store the previously computed \(Q(R_7)\) in \(W[Q(R_8)+20]\). |
| After F-ext-050 | Store \(Q(R_9)\) in \(W[H(R_9)+20]\), then read an unsigned word from \(W[Q(R_9)+20]\) and retain it. |
| After F-ext-055 | Store the retained word logically shifted right by 12 in \(W[Q(R_{10})+20]\). |
| After F-ext-060 | Store \(Q(R_{11})\) in \(W[H(R_{11})+20]\). |
| After F-ext-065 | Store the previously computed \(Q(R_{11})\) in \(W[Q(R_{12})+20]\). |

The address scale used for a write is not necessarily the scale used for a subsequent read. In particular, \(H(R_3)+20\) and \(Q(R_3)+20\) are distinct expressions, as are the corresponding expressions involving \(R_9\).

Between the write after F-ext-020 and F-ext-021, two unsigned four-byte read expressions access \(W[Q(R_3)+20]\). F-ext-021 receives, in argument-position order:

1. One read value logically shifted right by 12.
2. \(Q(R_3)\).
3. The other read value shifted left by 2 at unsigned 32-bit width.

Argument-position order does not establish read-evaluation order. The supplied representation does not prove that the two expressions become two distinct machine loads, nor that they observe one atomic snapshot. If they produce different observations, the first and third arguments need not be derived from the same word.

The third word-read expression is the read after F-ext-050. Its value is retained across F-ext-051 through F-ext-055. The subsequent write uses that retained value; no replacement read is shown.

F-ext-061 receives:

1. \((Q(R_{10})\mathbin{\&}63)\ll14\).
2. \(Q(R_{11})\).
3. \(R_{10}\ll30\), subject to the signed-shift uncertainty.

The first argument depends on the scaled result \(R_{10}\), not on a word read from memory. Under the conditional bitvector model it is a multiple of \(2^{16}\), between zero and \(\texttt{0x000f0000}\), because the lowest two bits of \(Q(R_{10})\) are zero.

### Byte accesses and remaining calls

After F-ext-070, the function reads an unsigned byte from the address formed by \(R_{10}\times2^{28}+1\). This multiplication originates from a signed-result expression, so overflowing cases require the conditional arithmetic interpretation described above.

The byte’s numeric value becomes the destination-address component for a one-byte write. If the read yields \(S\), the write stores the low byte of \(R_{13}\ll2\) at \(B[S+17]\). The destination therefore lies numerically between 17 and 272 inclusive. It is not formed from a full-width pointer read.

The subsequent call sequence is:

| Call | Explicit arguments and result use |
|---|---|
| F-ext-071 | \(R_{13}\ll2,\ R_{13}\ll2,\ R_{13}\ll4\), with shifts originating from an unsigned 32-bit result. |
| F-ext-072 | No explicit arguments shown; retain its 32-bit result. |
| F-ext-073 | The retained result from F-ext-072. |
| F-ext-074–F-ext-077 | No explicit arguments shown; F-ext-077 supplies \(R_{14}\). |

After F-ext-077, the function reads the unsigned byte at \(R_{10}\times2^{28}+1\) again. It does not substitute the earlier observation.

Let this second byte value be \(T\). The function stores the low byte of

\[
(R_{13}\mathbin{\&}\texttt{0x3fffffff})\gg20
\]

at \(B[T+17]\).

The remaining calls are:

| Call | Explicit arguments and result use |
|---|---|
| F-ext-078 | \(R_{14}\ll2\). |
| F-ext-079–F-ext-082 | No explicit arguments shown; F-ext-082 supplies \(R_{15}\). |
| F-ext-083 | \(R_{15}\ll2\), followed by the retained \(T\), widened to an unsigned 32-bit value. |
| F-ext-084 | No explicit arguments shown. |

The shifts involving \(R_{14}\) and \(R_{15}\) originate from signed-result expressions. Their behavior for negative values or overflow is not established independently of the target arithmetic.

The visible path then reaches the invalid-instruction boundary.

## Side effects

### Direct effects and errors

The visible body contains:

- Thirteen four-byte writes.
- Two one-byte writes.
- Three unsigned four-byte read expressions: two before F-ext-021 and one after F-ext-050.
- Two separate unsigned-byte reads.
- Eighty-four indirect calls.

These counts describe the supplied static representation, not a measured number of executed machine instructions.

All effects are prefix-conditional. A fault, trap, exception, or nonreturning dependency can prevent later events. The body provides no visible bounds check, null check, dependency-result validation, recoverable error return, rollback, or cleanup branch.

No application-specific error codes or failure-state guarantees are established.

### Ordering and aliasing

Computed addresses are not established as disjoint. Exact overlap or partial overlap between accesses must not be excluded.

Consequently:

- An earlier write can affect a later read.
- Unknown dependencies can modify memory between direct accesses.
- The first byte write can affect the second byte read if its destination overlaps the byte-read address.
- The word retained after F-ext-050 is not refreshed after intervening calls.
- The second byte value \(T\) remains the represented argument to F-ext-083 despite intervening dependencies.

The represented retained-value relationships do not independently prove how unknown callees interact with registers or storage in the actual calling convention.

No atomicity, synchronization, volatility, or memory-barrier semantics are established.

### Ownership and resources

No direct allocation or release operation is identifiable. Dependencies may nevertheless allocate, release, retain, or otherwise change resource state.

Neither passing a scaled result nor writing to a computed address establishes ownership transfer, borrowing, lifetime extension, or permission to access that address. These matters remain **documented_only**.

## Dependencies

F-ext-001 through F-ext-084 are unresolved indirect call sites. Different IDs may resolve to the same target, but the evidence does not justify merging them.

F-ext-001 derives its target from incoming control state. Most later target expressions use a repeated fixed-displacement pattern. F-ext-071 uses a different displacement from the surrounding calls and must not be assumed to invoke the same dependency.

No dependency contract establishes:

- Complete inputs or calling convention.
- Preservation of registers, globals, shared memory, or external state.
- Valid result ranges.
- Whether arguments represent numbers, addresses, handles, or something else.
- Ownership, allocation, retention, or release behavior.
- Whether a dependency returns, throws, traps, or transfers control elsewhere.

Calls remain observable contract events even when their returned values have no explicit later use.

## Acceptance examples

These examples are conditional consequences of the static representation, not executed tests. Each assumes execution reaches the stated event and that the relevant accesses are valid.

1. **Zero fourth input.**  
   With \(P=0\), the write after F-ext-010 stores zero at numeric address 20. The value written after F-ext-015 is also zero. Zero does not bypass calls or writes.

2. **Unsigned multiplication truncation.**  
   With \(P=\texttt{0x00000040}\), both \(P\times2^{26}\) and \(P\times2^{28}\) reduce to zero at unsigned 32-bit width. The write after F-ext-010 therefore stores zero at address 20.

3. **Masked extraction.**  
   With \(P=\texttt{0xffffffff}\), the value written after F-ext-015 is \(\texttt{0x000fffff}\). Bits 29–31 are removed before the right shift.

4. **Different write and read locations.**  
   With \(R_3=1\), the write after F-ext-020 stores 4 at address 36. Both word-read expressions for F-ext-021 use address 24. Retrieving the just-written value is not guaranteed.

5. **Independent argument-read observations.**  
   If the word observation used for F-ext-021’s first argument is \(\texttt{0x00001000}\), that argument is 1. If its third-argument observation is instead \(\texttt{0x00000003}\), that argument is 12. This example does not prescribe which read occurs first.

6. **Retained third word read.**  
   If the word read after F-ext-050 yields \(\texttt{0xabcdef12}\), the write after F-ext-055 carries \(\texttt{0x000abcde}\). Later changes to the original read address do not replace the represented retained value.

7. **Byte-store truncation.**  
   With \(R_{13}=\texttt{0x12345678}\), the first byte write carries \(\texttt{0xe0}\); the second carries \(\texttt{0x23}\). Neither preserves the entire shifted value.

8. **Two byte observations.**  
   If the first byte read yields 5 and the second yields 9, the byte-write destinations are 22 and 26 respectively. F-ext-083 receives 9 as its second explicit argument, regardless of a later change to the byte-read location.

9. **Faulting prefix.**  
   If the write after F-ext-005 faults, later calls and writes are not guaranteed. No local recovery or rollback is established.

## Uncertainties

The following remain **documented_only**:

- Accuracy and completeness of the recovered control flow in the presence of invalid instruction data.
- Runtime meaning of the terminal invalid-instruction marker.
- Exact architecture, address width, endianness, alignment rules, and calling convention.
- Signed multiplication, signed left shifts, and unsigned-to-signed conversions outside representable ranges.
- Actual dependency targets, complete interfaces, and side effects.
- Machine-level realization and relative order of the two word-read expressions before F-ext-021.
- Validity of computed addresses, including low numeric byte-write destinations.
- Memory layout beyond the explicit access widths, scales, and offsets.
- Ownership, resource lifetimes, and preservation of state across dependencies.

Repeated scaled addresses may suggest indexed state transformation. That is inferred intent only, not an established purpose or an additional acceptance requirement.

## Evidence

Established static evidence is limited to the supplied body:

- Four explicit 32-bit inputs, with the first and fourth visibly referenced.
- Incoming control state and an implicit incoming 32-bit register value.
- Eighty-four ordered indirect call sites and their displayed arguments.
- Thirteen four-byte writes and two byte writes.
- Three unsigned four-byte read expressions, including the later retained read.
- Two separate unsigned-byte reads.
- The displayed arithmetic, masks, shifts, and invalid-instruction boundary.

The arithmetic qualifications and unresolved dependency behavior are retained rather than replaced with assumed contracts. No runtime tests, target execution, dependency inspection, or independent implementation verification have been performed.
