<!-- Trusted provenance: F-be06f64d7eec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be06f64d7eec

## Purpose

The established behavior is a sequence of global reads, conditional memory accesses and calls, computed-address writes, and a terminal infinite loop. The supplied static representation does not establish an application-level purpose.

This contract does not interpret the function as an initializer, allocator, decoder, or object method. Zero addresses, repeated stores, and arithmetically constructed addresses are part of the visible behavior, not conditions to replace with validation or error returns.

If execution reaches the final loop, it does not return. A preceding fault, interrupt, or nonreturning dependency may prevent that loop from being reached.

## Interface

### Explicit and implicit inputs

The displayed interface has no explicit arguments and no ordinary return value. Its behavior nevertheless depends on:

- An incoming 32-bit scale operand.
- An incoming 32-bit comparison operand.
- An incoming 32-bit address base.
- An incoming link value used to construct a call target on one route.
- A word supplied by an incoming stack location at byte displacement 360.
- The word global at absolute address `0x089112d4`.
- The pointer global at absolute address `0x089110b4`.
- The byte-address-pointer global at absolute address `0x089110d4`.
- The word global at absolute address `0x0000003c`, referenced separately as a final destination base and final stored value.
- Memory reached through these inputs, intermediate loads, computed addresses, and dependency results.

These are machine-state dependencies, not evidence of a conventional public calling convention. The stack displacement identifies the supplied stack input; it does not establish an entry-time snapshot or an ordering between the later call’s argument evaluations.

### Widths and notation

The visible model uses 32-bit addresses and words, 8-bit bytes, and one call result represented as 64 bits.

Throughout this contract:

- Address addition, multiplication, subtraction, and left shifts retain the low 32 bits.
- Signed tests interpret a 32-bit pattern as two’s-complement signed.
- Right shifts of the unsigned working values are logical.
- Word stores affect four bytes; byte stores affect one byte.
- Access order in a table is execution order, not simultaneous assignment.
- Numeric offsets are bytes.

The following labels are mathematical notation, not recovered implementation identifiers:

| Label | Meaning |
|---|---|
| \(G\) | Initially sampled word global; replaced on the conditional route |
| \(P\) | Initially sampled pointer global; replaced on the conditional route |
| \(H\) | Initially sampled byte-address pointer; replaced on the conditional route |
| \(A\) | Incoming address base |
| \(M\) | Low 32 bits of the incoming scale operand multiplied by 2,048 |
| \(W\) | Working displacement, initially zero |
| \(Q\) | Working address, initially zero |
| \(C\) | Value selected before the common writes |
| \(J\) | Fixed address value `0x0891111c` |

Overlapping destinations are permitted by the visible expressions. Later writes may overwrite earlier values, but that does not erase the earlier access events.

### Preconditions and outputs

No alignment, null, range, overflow, or permissions validation is visible. Continued execution requires every reached access, interrupt, and call to complete.

There is no ordinary output-value contract. Observable effects comprise memory accesses, call targets and arguments, the conditional software interrupt, and nontermination if the final loop is reached.

## Behavior

### Entry sampling and route selection

The function samples \(G\) from `0x089112d4`, then \(P\) from `0x089110b4`. It constructs \(M\), initializes \(Q=W=0\), evaluates signed-subtraction overflow, and samples \(H\) from `0x089110d4`.

The conditional route is selected exactly when subtracting the incoming comparison operand from \(M\), as signed 32-bit values, overflows signed subtraction.

This is not unsigned borrow, ordinary less-than comparison, or overflow of the earlier multiplication. The multiplication’s retained 32-bit result is the subtraction input.

Without subtraction overflow, execution proceeds to the preliminary stores with the initially sampled \(G,P,H\), with \(W=0\), and with cursor value \(M\).

### Conditional setup

When subtraction overflows, these accesses occur in order:

| Order | Operation |
|---:|---|
| 1 | Store byte zero through the pointer supplied by global `0x089110d4`. |
| 2 | Store byte zero at \(M\). |
| 3 | Load a word from \(M\); denote it \(O\). |
| 4 | Execute software interrupt `0xde`, decimal 222. |
| 5–7 | Store \(P\) at \(A+20\), three separate times. |
| 8 | Store the initial \(G\) at \(A+60\). |
| 9–10 | Store \(O\) at \(A+84\), twice. |
| 11 | Load a word from \(O+20\); denote it \(K\). |
| 12 | Load a word from \(O+116\); denote it \(N\). |
| 13 | Load a word from the initial \(G+108\), then store its low byte at \(N+23\). |
| 14 | Store the low byte of \(O\) at the initial \(G+21\). |
| 15 | Load a byte from the initial \(G+5\); retain its zero-extended value as \(B\). |
| 16 | Load a byte from \(N+7\); its zero-extended value replaces \(G\). |

The first operation explicitly references the byte-pointer global again. The representation does not justify treating every reference to that global as one guaranteed physical load.

The word load at \(M\) follows a one-byte store there. Only one byte has been cleared; \(O\) is not thereby established as zero. Its value depends on the remaining bytes and byte order.

The function then makes these calls:

1. **F-7aee1fc5f275**, with three words:
   \[
   ((K\gg8)\ll12),\quad K,\quad ((K\gg8)\ll24).
   \]
2. **F-000000000001**, with the preceding result shifted left by 12.
3. **F-000000000002**, with the same three-word transform applied to an additional machine-register word available after the preceding call.

That additional register word is not established as a conventional return value of **F-000000000001**.

Let the result of **F-000000000002** be \(Z\). The continuing state is:

- \(P=Z\ll8\).
- \(W=Z\ll12\).
- \(H=\texttt{0x007f8000}\).
- The working link value becomes `0x08910ff9`.

If \(P\) is signed nonnegative, the preliminary stores are skipped. The common writes begin with \(Q=0\) and \(C=B\).

If \(P\) is signed negative, the preliminary stores execute with cursor value \(J=\texttt{0x0891111c}\). Thus this fixed address is observable as stored data, not merely an unresolved target or symbolic location.

The repeated sign condition on this route tests an unchanged value. An intervening temporary assignment of `0x08911110` does not become the common value \(C\): the subsequent preliminary sequence replaces it with \(W\).

### Preliminary stores

This stage is reached either without conditional setup or after conditional setup with negative \(P\).

Let \(D=M\) on the first route and \(D=J\) on the second.

| Order | Word destination | Stored value |
|---:|---|---|
| 1 | \(W+A\) | \(H\) |
| 2 | \(P+W\) | \(D\) |
| 3 | \(H+W\) | \(D\) |
| 4 | \(D+W\) | \(W\) |
| 5 | \(P+(W\gg2)\) | \(W\) |
| 6 | \(W+P\) | \(P\) |
| 7 | \(H+P\) | \(W\gg2\) |

Afterward, \(Q=P\) and \(C=W\).

Stores 2 and 6 address the same location. On the negative-\(P\) conditional route, stores 2 and 3 write the exact word `0x0891111c`, although subsequent aliasing writes may replace it.

### Common writes and initial common calls

All continuing routes execute:

| Order | Word destination | Stored value |
|---:|---|---|
| 1 | \(2P\) | \(G\) |
| 2 | \(Q+P\) | \(C\) |
| 3 | \((Q\gg2)+P\) | \(Q\gg2\) |
| 4 | \(C+W\) | \(Q\gg2\) |
| 5 | \(P+Q\) | \(P\) |
| 6 | \(2Q\) | \(P\) |
| 7 | \(P+Q\) | \(P\) |
| 8 | \(P+Q\) | \(Q\gg2\) |

The function calls **F-000000000003** with \(Q\), then **F-000000000004** with no explicit arguments. Their target constructions are specified under Dependencies.

The latter result is represented as 64 bits. Denote its low word by \(L\) and high word by \(U\). The next word stores are:

| Order | Destination | Value |
|---:|---|---|
| 1 | \(Q+(Q\gg2)\) | \(L\) |
| 2 | \(2Q\) | \(L-22\) |
| 3 | \(2(Q\gg2)\) | \(Q\gg2\) |
| 4 | \(2Q\) | \(Q\) |
| 5 | \(P+(Q\gg2)\) | \(U\) |
| 6 | \(2Q\) | \(Q\) |

Subtraction in store 2 is modulo \(2^{32}\).

Next, **F-000000000005** receives \((Q,Q)\), followed by **F-000000000006** with no explicit arguments. Denote the latter’s word result by \(R\).

### Result-dependent writes and calls

Let \(T=Q\gg2\). The next stores are:

| Order | Word destination | Stored value |
|---:|---|---|
| 1 | \(2Q\) | \(Q\) |
| 2 | \(T+Q\) | \(R\) |
| 3 | \(Q+64\) | \(T\) |
| 4 | \(2Q\) | \(Q\) |
| 5 | \(T+76\) | \(T\) |
| 6 | \(Q+76\) | \(T\) |
| 7 | \(2Q\) | \(Q\) |
| 8 | \(4R+20\) | \(R\gg2\) |
| 9 | \(R+Q\) | \(R\) |
| 10 | \(4R+20\) | \(R\gg2\) |

**F-c512cbdbd236** then receives the word loaded from \(Q+60\) and the incoming stack word at displacement 360. The \(Q+60\) read follows the preceding calls and stores; it is not an entry snapshot. The representation does not resolve the relative evaluation order of the two arguments.

Continuing operations are:

1. Store \(P\) at \(Q+(R\gg4)\).
2. Call **F-000000000007** with \((R,R\gg2)\); retain word result \(E\).
3. Store \(P\) at \(Q+(R\gg2)\).
4. Call **F-000000000008** with \((E,R\gg2)\); no result is consumed.
5. Call **F-000000000009** with \((R\gg2,R\gg2)\); retain word result \(V\).
6. Store \(Q\) at \(P+Q\).
7. Store \(V\) at \(V+24\).
8. Store \(P\) at \(Q+(R\gg4)\).
9. Store \(R\gg4\) at \((R\gg4)+24\).

**F-00000000000a** then receives, in order:
\[
R\gg4,\quad V\ll2,\quad R\gg2,\quad R\gg4.
\]
Denote its word result by \(Y\).

### Final accesses and nontermination

The function stores \(R\gg2\) at \(P\), then stores \(R\gg4\) at \(Y+60\).

The final assignment separately references the word global at absolute address `0x0000003c`:

- One use supplies the destination base, to which 24 is added.
- The other use supplies the stored word.

The representation does **not** establish a single loaded snapshot reused for both purposes. It also does not establish their physical load count or evaluation order. If the global is stable, both uses yield the same word \(x\), and the effect is a word store of \(x\) at \(x+24\). If the uses can observe different values, the contract must retain that possibility rather than force equality.

After the final store, execution enters an unconditional infinite loop. No further explicit memory access, call, or exit condition is visible.

## Side effects

- Memory accesses may alias globals, earlier destinations, dependency-accessible memory, or address zero.
- Repeated stores remain separate listed operations; they are not permission to coalesce observable accesses.
- Byte stores truncate to eight bits and do not clear adjacent bytes.
- Address calculations wrap within the stated 32-bit model without visible validation.
- The conditional interrupt precedes the conditional setup’s address-base stores. Its effects and return conditions are unknown.
- No allocation, deallocation, ownership transfer, reference-count operation, or release protocol is established.
- Address-like arguments do not establish that a dependency owns, preserves, or releases memory.
- Dependencies may alter memory or globals. This contract does not assume otherwise.
- Any fault, trap, or nonreturning dependency prevents subsequent operations. No recovery path or error result is visible.

## Dependencies

Opaque labels identify call sites, not necessarily distinct implementations. The following target constructions are established independently of the callees’ unknown behavior.

| Dependency | Target construction |
|---|---|
| **F-7aee1fc5f275** | Direct dependency identified by the supplied map |
| **F-c512cbdbd236** | Direct dependency identified by the supplied map |
| **F-000000000001** | `0x08910fd9 + 0x0ffe` |
| **F-000000000002** | `0x08910fe1 + 0x0e10` |
| **F-000000000003** | Working link value plus `0x0ff8` |
| **F-000000000004** | `0x089111cb + 0x0ffe` |
| **F-000000000005** | `0x089111cd + 0x0ffe` |
| **F-000000000006** | Address of opaque symbol **F-00000000000b** plus `0x0ffe` |
| **F-000000000007** | `0x0891132b + 0x0588` |
| **F-000000000008** | `0x0891133f + 0x0308` |
| **F-000000000009** | Address of opaque symbol **F-00000000000c** plus `0x0448` |
| **F-00000000000a** | `0x08911367 + 0x01b0` |

For **F-000000000003**, the working link value derives from incoming machine state when conditional setup is skipped. Conditional setup replaces it with `0x08910ff9`, making that route’s target `0x08910ff9 + 0x0ff8`.

**F-00000000000b** and **F-00000000000c** name two distinct symbol-referenced bases, respectively used for **F-000000000006** and **F-000000000009**. They are not additional observed calls. Their numeric base addresses are not independently established here; the known offsets remain part of the contract.

All target additions use the 32-bit address model. No target-bit normalization or instruction-set interpretation is added.

Argument order and consumed results are specified above. Internal behavior, register preservation, memory effects, termination, and exceptional behavior remain unknown. Calls without explicit arguments may still depend on registers, globals, and preceding calls.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Signed-overflow distinction.** With \(M=0\) and comparison operand 1, setup is skipped. With \(M=0\) and comparison operand `0x80000000`, subtraction overflows and setup is entered.

2. **Zero-address exposure.** Without conditional setup, \(W=0\). Preliminary store 4 writes zero at \(M\). Common store 4 writes \(P\gg2\) at address zero. Replacing these operations with input rejection would change behavior.

3. **Nonnegative conditional result.** If setup completes with \(P\) nonnegative, preliminary stores are absent, \(Q=0\), and common store 6 writes \(P\) at address zero.

4. **Fixed-address data.** If setup completes with negative \(P\), preliminary stores 2 and 3 write `0x0891111c` at \(P+W\) and `0x007f8000 + W`, respectively. Later aliasing stores may overwrite either destination.

5. **Conditional call target.** After conditional setup, **F-000000000003** targets `0x08910ff9 + 0x0ff8`. Without setup, it uses the incoming-derived link value plus the same offset.

6. **Truncation and wrap.** A byte store of a word ending in `0xab` writes byte 171. If \(L=10\), the subtraction-derived word store writes 4,294,967,284.

7. **Repeated writes.** Common stores 2, 5, 7, and 8 share destination \(P+Q\). Store 8 writes \(Q\gg2\); all earlier listed writes still occur.

8. **Stable final global.** If both final uses of global `0x3c` yield `0x1000`, the final store writes `0x1000` at `0x1018`. This example does not establish that the two uses share one load.

9. **Changing final global.** If the destination-base use yields \(a\) and the stored-value use yields \(b\), the final effect is a word store of \(b\) at \(a+24\). Whether such differing observations are possible depends on unresolved execution and memory semantics.

10. **Termination boundary.** If every preceding operation completes, the final store is followed by indefinite looping, not normal return.

## Uncertainties

### Established static evidence

The supplied representation establishes access widths, the listed ordered operations, arithmetic expressions, conditional interrupt, call-site sequence and target constructions, fixed value `0x0891111c`, separate final global references, and terminal loop.

### Inferred intent

No application-level purpose is sufficiently supported. Pointer-like use does not prove an object layout, valid allocation, or ownership relationship.

### documented_only

Unresolved matters include:

- Actual calling convention, entry validity, and register preservation.
- Callee identities beyond the supplied dependency map.
- Numeric addresses of the two opaque symbol bases.
- Byte order, alignment requirements, memory mapping, and device-memory semantics.
- Interrupt behavior and whether it returns.
- Whether the represented 64-bit result is one logical return or multiple register outputs.
- Meaning of the additional register consumed after **F-000000000001**.
- Physical load count and evaluation order for the final global’s two uses, and whether they can observe different values.
- Whether suspicious operations reflect executable machine behavior or limitations of the static representation.
- Whether any reachable execution completes all accesses and reaches the loop.

## Evidence

This contract is derived solely from the supplied static function representation and dependency map. Embedded comments, apparent names, and the prior draft are not independent evidence of intent.

No implementation access, runtime execution, tests, traces, or memory observations were available. All acceptance examples are conditional deductions, not observed outcomes.
