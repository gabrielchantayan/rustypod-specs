<!-- Trusted provenance: F-ae88d44d12dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae88d44d12dc

## Purpose

The supplied static representation shows a sequence of indirect calls, address calculations, byte stores, word stores, and a small number of explicit memory reads. It does not establish a domain-level purpose.

The observable operation is best described as **an ordered, externally dependent memory-mutation sequence**. Values obtained from unresolved callees and implicit entry state become addresses, stored data, and arguments to later calls.

No successful return path is established. The supplied representation ends at an invalid-instruction boundary after its last visible indirect call. This is not evidence of an intentional application-level termination protocol.

This contract distinguishes:

- **Established static evidence:** operations and ordering present in the supplied representation.
- **Conditional interpretation:** behavior under a 32-bit bit-vector execution model with valid call targets and accessible memory.
- **Documented-only unknowns:** calling convention, dependency behavior, actual instruction validity, ownership, and behavior beyond the visible boundary.

## Interface

### Explicit inputs

The apparent interface accepts two 32-bit arguments.

- **First argument:** no explicit use is visible. This does not prove that it is unobservable to callees through the underlying calling convention.
- **Second argument:** supplied as the sole displayed argument to the first indirect call.

No type, encoding, ownership, nullability, or validation requirements are established for either argument.

### Implicit inputs

The visible sequence also consumes three entry-state quantities not established by the apparent argument list:

- **Entry word \(U\):** a 32-bit value used as data and as the source of logical right shifts.
- **Entry base \(B\):** a 32-bit address-like value used for an early byte store.
- **Entry control value \(K\):** used to derive the first indirect call target by adding `0xFEC`.

A later operation consumes a secondary register result observed after one dependency call. Its ABI meaning and definedness are unknown.

These are genuine prerequisites of the displayed behavior. They must not silently be replaced with zero, inferred from the explicit arguments, or treated as initialized local storage.

### Outputs

The apparent interface has no ordinary return value. No normal completion is visible.

Observable outputs include:

- Arguments and ordering of indirect calls.
- Explicit memory reads and writes.
- Any additional effects of unresolved callees.
- Whatever termination, fault, or continuation occurs at the invalid-instruction boundary.

### Notation and arithmetic

For the behavioral trace below:

- `F-u0001` through `F-u0119` are opaque, document-local identifiers for successive unresolved call sites. They do not assert distinct underlying functions.
- \(r_n\) is the 32-bit result used from call `F-uNNNN`.
- For calls displaying a 64-bit result, \(\ell_n\) and \(h_n\) are its low and high 32-bit halves.
- \(W_8[a]\) and \(W_{32}[a]\) denote an explicit byte or 32-bit store at address \(a\).
- \(R_8[a]\) and \(R_{32}[a]\) denote explicit unsigned byte or 32-bit reads.
- \(\operatorname{low8}(x)\) retains the low eight bits.
- Right shifts in the trace are logical shifts of unsigned 32-bit quantities.

Address additions, multiplications, and left shifts below use a **conditional 32-bit bit-vector interpretation**, retaining the low 32 bits. The representation contains signed intermediates as well as unsigned ones; it is not sufficient to establish portable signed-overflow behavior or independently confirm the underlying instruction semantics.

The 64-bit result notation specifies bit halves, not a memory layout or a particular register-pair ABI.

## Behavior

### Execution prerequisites and ordering

The visible path contains no conditional branches, loops, explicit validation, or recovery paths. Its progression nevertheless depends on every preceding call returning and every memory operation completing.

Each listed call interval denotes individual calls in ascending order. Calls described as having “no displayed arguments” may still depend on machine state. Calls whose results are not used may still have observable effects.

All listed reads and writes occur in the stated order relative to calls. Unknown callees may access or alter memory between them; consequently, repeated reads must remain separate observations.

### 1. Entry calls and first word store

1. Call `F-u0001` with the second explicit argument. Its target is derived from \(K+\texttt{0xFEC}\).
2. Call `F-u0002` with no displayed arguments and retain \(r_2\).
3. Call `F-u0003` with \(r_2\), \(U \mathbin{\gg} 2\), and the same shifted value again.
4. Call `F-u0004` through `F-u0006`, each with no displayed arguments.
5. Call `F-u0007` with no displayed arguments and retain \(r_7\).
6. Store \(U\) into \(W_{32}[4r_7+\texttt{0x14}]\).
7. Call `F-u0008` with \(4r_7\).

No bounds, alignment, or address-validity check precedes this store.

### 2. Two further word-address constructions

1. Call `F-u0009` through `F-u0011` with no displayed arguments.
2. Call `F-u0012` with no displayed arguments and retain \(r_{12}\).
3. Store \(4r_{12}\) into \(W_{32}[16r_{12}+\texttt{0x14}]\).
4. Define retained quantities:
   - \(M=r_{12}\mathbin{\&}\texttt{0x3FFFFFFF}\)
   - \(Q=M\mathbin{\gg}2\)
5. Call `F-u0013` with \(M\).
6. Call `F-u0014` through `F-u0016` with no displayed arguments.
7. Call `F-u0017` with no displayed arguments and retain \(r_{17}\).
8. Store \(M\) into \(W_{32}[4r_{17}+\texttt{0x14}]\).
9. Call `F-u0018` with \(4r_{17}\).

The first store uses the unmasked result. Masking does not retroactively constrain its address.

### 3. Byte stores and replacement of a word field

1. Call `F-u0019` through `F-u0021` with no displayed arguments.
2. Call `F-u0022` with no displayed arguments and retain \(r_{22}\).
3. Define \(T=8r_{17}\).
4. Store \(\operatorname{low8}(T)\) into \(W_8[B+1]\).
5. Store \(4r_{22}\) into \(W_{32}[T+\texttt{0x34}]\).
6. Call `F-u0023` with \(4r_{22}\).
7. Call `F-u0024` with no displayed arguments and retain \(r_{24}\).
8. Define \(N=r_{22}\mathbin{\&}\texttt{0x3FFFFFFF}\).
9. Call `F-u0025` with \(r_{24}\).
10. Call `F-u0026` through `F-u0028` with no displayed arguments.
11. Call `F-u0029` with no displayed arguments and retain its 64-bit result.
12. Store \(\operatorname{low8}(T)\) into \(W_8[N+1]\).
13. Store \(h_{29}\) into \(W_{32}[T+\texttt{0x34}]\).
14. Call `F-u0030` with \(4\ell_{29}\).

The second word store has the same computed destination as the earlier store in this stage, but intervening dependencies may observe or change that memory.

### 4. Low-address and retained-base updates

1. Call `F-u0031` through `F-u0033` with no displayed arguments.
2. Call `F-u0034` with no displayed arguments and retain \(r_{34}\).
3. Define \(V=16r_{17}\).
4. Store \(\operatorname{low8}(V)\) into \(W_8[(N\mathbin{\gg}20)+\texttt{0x11}]\).
5. Store \(\operatorname{low8}(4r_{12})\) into \(W_8[V+\texttt{0x0D}]\).
6. Call `F-u0035` with \(4r_{34}\) twice.
7. Call `F-u0036` with no displayed arguments and retain \(r_{36}\).
8. Call `F-u0037` with \(r_{36}\) and \(r_{36}\mathbin{\gg}3\).
9. Call `F-u0038` through `F-u0040` with no displayed arguments.
10. Call `F-u0041` with no displayed arguments and retain \(r_{41}\).
11. Store \(\operatorname{low8}(V)\) at absolute byte address `0x11`.
12. Again store \(\operatorname{low8}(4r_{12})\) into \(W_8[V+\texttt{0x0D}]\).
13. Call `F-u0042` with \(4r_{41}\).
14. Call `F-u0043` through `F-u0045` with no displayed arguments.
15. Call `F-u0046` with no displayed arguments and retain \(r_{46}\).
16. Store \(4r_{12}\) into \(W_{32}[N+\texttt{0x14}]\).
17. Store \(\operatorname{low8}(2N)\) into \(W_8[N+\texttt{0x0D}]\).

The absolute store is distinct from the earlier computed-address store, even when their addresses happen to coincide.

### 5. Byte-indexed stores and a saved byte

Define:

- \(P=32r_{46}\)
- \(J=(r_{46}\mathbin{\&}\texttt{0x07FFFFFF})\mathbin{\gg}17\)

Then:

1. Call `F-u0047` and `F-u0048` with no displayed arguments.
2. Call `F-u0049` with no displayed arguments and retain \(r_{49}\).
3. Store \(128r_{46}\) into \(W_{32}[P+\texttt{0x14}]\).
4. Store \(\operatorname{low8}(4r_{49})\) into \(W_8[P+\texttt{0x0D}]\).
5. Call `F-u0050` with \(4r_{49}\).
6. Call `F-u0051` through `F-u0054` with no displayed arguments.
7. Store \(\operatorname{low8}(2Q)\) into \(W_8[J+7]\).
8. Read \(R_8[P+1]\), add `0x11` to the unsigned byte value, and store \(\operatorname{low8}(J)\) at that resulting address.
9. Call `F-u0055` with \(r_{49}\mathbin{\ll}21\) three times.
10. Call `F-u0056` with no displayed arguments and retain \(r_{56}\).
11. Call `F-u0057` with \(r_{56}\).
12. Call `F-u0058` through `F-u0060` with no displayed arguments.
13. Call `F-u0061` with no displayed arguments and retain \(r_{61}\).
14. Read \(R_8[P+1]\) again and retain this second byte as \(b\).
15. Store zero into \(W_8[b+\texttt{0x11}]\).
16. Call `F-u0062` with \(4r_{61}\).
17. Call `F-u0063` through `F-u0066` with no displayed arguments.
18. Store \(\operatorname{low8}(r_{56}\mathbin{\gg}3)\) into \(W_8[P+\texttt{0x0D}]\).
19. Store \(b\) at absolute byte address `0x0D`.

The two reads of \(P+1\) need not return the same byte.

### 6. Doubled-byte base and word-linked accesses

Define \(D=2b\).

1. Call `F-u0067` with \(D\), `0x98`, and `0x260`.
2. Call `F-u0068` with no displayed arguments.
3. Call `F-u0069` with the observed secondary register quantity shifted right by three.
4. Call `F-u0070` through `F-u0072` with no displayed arguments.
5. Call `F-u0073` with no displayed arguments and retain \(r_{73}\).
6. Call `F-u0074` with \(4r_{73}\), retaining \(r_{74}\).
7. Store \(\operatorname{low8}(r_{56}\mathbin{\gg}3)\) into \(W_8[D+\texttt{0x0D}]\).
8. Store `0xC0` at absolute byte address `0x0D`.
9. Call `F-u0075` with \(4r_{74}\).
10. Call `F-u0076` through `F-u0078` with no displayed arguments.
11. Call `F-u0079` with no displayed arguments and retain \(r_{79}\).
12. Store \(4r_{79}\) into \(W_{32}[D+\texttt{0x24}]\).
13. Read \(R_{32}[D+\texttt{0x74}]\), retaining the word as \(A\).
14. Call `F-u0080` with \(A\), \(2A\), \(32r_{79}\), and \((A\mathbin{\&}\texttt{0x7FFFFFFF})\mathbin{\gg}2\).
15. Call `F-u0081` through `F-u0084` with no displayed arguments.
16. Store \(32r_{79}\) into \(W_{32}[A+\texttt{0x24}]\).
17. Read \(R_{32}[A+\texttt{0x74}]\) and pass that word to `F-u0085`.

The second word read is through the first read’s value. It is not necessarily another read of the same object.

### 7. Wide results and further scaled stores

1. Call `F-u0086` through `F-u0088` with no displayed arguments.
2. Call `F-u0089` with no displayed arguments and retain its 64-bit result.
3. Store \(r_{56}\mathbin{\gg}3\) into \(W_{32}[4h_{89}+\texttt{0x14}]\).
4. Call `F-u0090` with \(4\ell_{89}\) three times, followed by \(h_{89}\mathbin{\gg}3\).
5. Call `F-u0091` with no displayed arguments and retain its 64-bit result.
6. Call `F-u0092` with \(\ell_{91}\), \(h_{91}\), \(\ell_{91}\), and \(h_{91}\mathbin{\gg}3\).
7. Call `F-u0093` through `F-u0095` with no displayed arguments.
8. Call `F-u0096` with no displayed arguments and retain \(r_{96}\).
9. Store \(4\ell_{89}\) into \(W_{32}[4r_{96}+\texttt{0x14}]\).
10. Call `F-u0097` with \(4r_{96}\).
11. Call `F-u0098` through `F-u0100` with no displayed arguments.
12. Call `F-u0101` with no displayed arguments and retain \(r_{101}\).

### 8. Final byte-indexed updates and visible termination

Define \(X=4r_{101}\).

1. Read \(R_8[A+1]\), then store \(\operatorname{low8}(X)\) at the byte-valued result plus `0x11`.
2. Call `F-u0102` and `F-u0103` with no displayed arguments.
3. Call `F-u0104` with no displayed arguments and retain \(r_{104}\).
4. Read \(R_8[A+1]\) again, then store \(\operatorname{low8}((r_{101}\mathbin{\&}\texttt{0x3FFFFFFF})\mathbin{\gg}20)\) at this new byte-valued result plus `0x11`.
5. Call `F-u0105` with \(4r_{104}\).
6. Call `F-u0106` through `F-u0108` with no displayed arguments.
7. Call `F-u0109` with no displayed arguments and retain \(r_{109}\).
8. Read \(R_8[1024X+1]\), then store zero at the byte-valued result plus `0x11`.
9. Call `F-u0110` with \(8X\), \(8X\), \(X\), and \(4r_{109}\).
10. Call `F-u0111` through `F-u0113` with no displayed arguments.
11. Call `F-u0114` with no displayed arguments and retain \(r_{114}\).
12. Read \(R_8[1024X+1]\) again, then store zero at the byte-valued result plus `0x11`.
13. Call `F-u0115` with \(4r_{114}\).
14. Call `F-u0116` through `F-u0119` with no displayed arguments.
15. Reach the displayed invalid-instruction boundary.

Neither pair of repeated byte reads may be collapsed on the assumption that intervening dependencies preserve memory.

## Side effects

### Memory layout and access widths

The representation establishes access widths and offsets, not object types or allocation layouts.

- Byte accesses use one byte.
- Word accesses use 32 bits.
- Byte values used as addresses are zero-extended, then increased by `0x11`; their destinations range from `0x11` through `0x110`.
- \(J\) ranges from 0 through 1023. Thus \(J+7\) ranges from 7 through 1030.
- \((N\mathbin{\gg}20)+\texttt{0x11}\) ranges from `0x11` through `0x410`.
- \(D\) is an even integer from 0 through 510. Its derived word-access addresses are not necessarily four-byte aligned.

No byte order is established for word accesses. No packing, field ownership, or non-aliasing guarantee is established.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or lifetime boundary is identifiable. Calls may allocate, release, retain, or invalidate memory; none of those behaviors may be assumed absent.

Computed addresses can alias each other, absolute low addresses, or memory used by dependencies. Intermediate stores remain observable even when later stores overwrite the same location.

### Errors

There are no visible status checks or rollback operations. A call that fails to return, an invalid target, an inaccessible address, or an unsupported unaligned access may prevent the remaining sequence.

The representation does not establish an error code, exception type, recoverable failure mechanism, or guaranteed final memory state.

## Dependencies

`F-u0001` through `F-u0119` denote 119 visible indirect call occurrences. Their true identities and possible aliasing are unknown.

Most targets are represented as address-like anchors plus a displacement. This does not establish that the destinations contain valid callable instructions. Displayed argument lists are evidence of the reconstructed calls, not independently verified ABI signatures.

No dependency is known to:

- Preserve globals or pointed-to storage.
- Preserve undisplayed registers.
- Return normally.
- Own or release an argument.
- Be pure, idempotent, or safe to omit.

The call-result widths and secondary register observation are therefore conditional interface evidence, not confirmed external contracts.

## Acceptance examples

These examples specify conditional expectations; they are not test results.

1. **Entry forwarding:** Given \(U=15\), if the first three calls complete, `F-u0003` receives \(r_2\), 3, and 3. The discarded low two bits are not rounded upward.

2. **Mask timing:** Given \(r_{12}=\texttt{0x40000003}\), the conditional bit-vector interpretation gives a first word-store address of `0x44` and stored value 12. Subsequently \(M=3\) and \(Q=0\). Applying the mask before that first address calculation would not match the trace.

3. **Changed byte between reads:** If the first read of \(P+1\) returns 2 and the second returns 9, the first byte-indexed store targets `0x13`, while the later zero store targets `0x1A`. The saved byte is 9 and \(D=18\).

4. **Byte truncation:** If \(r_{56}=\texttt{0x00000808}\), then \(r_{56}\mathbin{\gg}3=\texttt{0x101}\). Its byte stores write `0x01`; the later 32-bit store derived from it writes `0x00000101`.

5. **Repeated clearing is not redundant:** If intervening dependencies change the byte read from \(1024X+1\), the two final zero stores may target different addresses. A matching implementation preserves both reads and both stores.

6. **Unaligned access:** If \(b=1\), then \(D=2\), making \(D+\texttt{0x24}=38\). Completion is conditional on the execution environment supporting that 32-bit store; the contract does not add an alignment correction.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and intended data structures.
- Whether the apparent entry point and reconstructed control flow are valid.
- Actual calling convention, target identities, and register preservation.
- Definedness and provenance of implicit entry state.
- ABI interpretation of 64-bit results and the secondary register result.
- Endianness, alignment rules, address-space mappings, and memory permissions.
- Exact overflow and shift semantics of the underlying instructions.
- Dependency effects, allocation behavior, ownership, and failure modes.
- Behavior at or beyond the invalid-instruction boundary.
- Whether any normal return path exists outside the supplied representation.

No runtime tests have been performed.

## Evidence

The sole evidence is the supplied static representation. It supports the displayed call sequence, explicit memory operations, operand widths, masks, shifts, and ordering described above.

It also explicitly reports invalid instruction data and truncation of control flow. Those limitations prevent treating the representation as a verified executable specification.

The contract therefore preserves the visible operations without assigning unsupported application meaning, inventing dependency guarantees, or claiming runtime validation.
