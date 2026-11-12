<!-- Trusted provenance: F-6b36adf26ce8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b36adf26ce8

## Purpose

F-6b36adf26ce8 exposes a recovered execution path consisting of indirect calls, calculated word stores, byte-selected byte stores, and an unconditional repeating phase.

No application-level purpose is established. The supplied operations do not justify classifying the function as initialization, decoding, allocation, device control, or manipulation of a known object layout. In particular, calculated low addresses must not be replaced with assumed object-relative accesses.

This contract distinguishes:

- **Established static evidence:** explicit operations, widths, data dependencies, and ordering visible in the supplied representation.
- **Conditional arithmetic interpretation:** consequences of treating calculated values and addresses as 32-bit machine quantities.
- **documented_only:** behavior not established by the supplied material.

The representation includes warnings about invalid instruction data and removal of unreachable regions. Consequently, the contract describes the visible recovered path, not a verified reconstruction of every executable path.

## Interface

### Explicit inputs

| Argument position | Visible representation | Established use |
|---|---|---|
| First | 32-bit value without an established semantic type | No explicit use in the recovered body |
| Second | Signed 32-bit integer, denoted \(S\) | Determines the first word-store address and payload |
| Third | Unsigned 32-bit integer, denoted \(U\) | Supplies shifted values for calls, a word payload, and a byte-store address |

The first argument’s absence from explicit expressions does not establish that its incoming machine state is irrelevant to indirect calls. Actual callee prototypes and calling-convention requirements are unknown.

An additional implicit input is the incoming return-link state. The first indirect call target depends on that state plus a fixed displacement. The three explicit arguments therefore do not completely determine execution.

### Outputs and termination

No explicit return value or normal return is established.

The visible path ends in unconditional repetition. If every reached indirect call returns normally and every explicit memory access succeeds, the function continues repeating rather than returning.

A callee may fail to return or otherwise divert execution. The representation does not establish how exceptions, faults, external termination, or invalid call targets behave.

### Widths and notation

The following mathematical notation describes conditional 32-bit behavior:

- \(\operatorname{word}(x)\): the low 32 bits of \(x\).
- \(\operatorname{byte}(x)\): the low 8 bits of \(x\).
- \(\operatorname{shr}(x,n)\): logical right shift of an unsigned 32-bit value by \(n\) bits.

Explicit word stores write 32 bits. Explicit selector reads and byte stores access 8 bits. A selector byte is interpreted as an unsigned value from 0 through 255.

Some supplied expressions involve signed multiplication or signed left shift. Their portable language-level behavior is not established for negative or overflowing operands. Formulas using \(\operatorname{word}\) describe the conditional machine interpretation, not a claim that those signed expressions have defined portable semantics.

Address width, byte order, word alignment requirements, memory mapping, and fault behavior remain **documented_only**.

## Behavior

### Call-group convention

A **five-call group** consists of an initial call with the stated explicit arguments, followed by four calls with no explicit arguments. The fifth call’s 32-bit result is retained when a result is named below.

“No explicit arguments” describes only the supplied representation. It does not establish a zero-argument callee prototype or the absence of meaningful incoming register state.

Call groups describe invocation order, not reusable dependency identities. Distinct groups need not invoke the same targets.

### Initial word-store sequence

The following mathematical values describe retained results and derived quantities:

\[
P=\operatorname{word}(S\,2^{24}),\qquad
A=\operatorname{word}(S\,2^{26}+20),\qquad
H=\operatorname{shr}(U,8).
\]

There are **eight explicit 32-bit stores before byte processing begins**. Their order and the intervening calls are:

| Store | Intervening activity before this store | Destination | Payload |
|---|---|---|---|
| 1 | None | \(A\) | \(P\) |
| 2 | Five-call group beginning with \(P\), retaining \(R_1\) | \(\operatorname{word}(4R_1+20)\) | \(P\) |
| 3 | Five-call group beginning with \(\operatorname{word}(4R_1)\), retaining \(R_2\) | \(\operatorname{word}(16R_2+20)\) | \(B=\operatorname{word}(4R_2)\) |
| 4 | First forwarding sequence and a five-call group, described below, retaining \(R_3\) | \(\operatorname{word}(4R_3+20)\) | \(B\) |
| 5 | Five-call group beginning with \(\operatorname{word}(4R_3)\), retaining \(R_4\) | \(\operatorname{word}(16R_4+20)\) | \(C=\operatorname{word}(4R_4)\) |
| 6 | Second forwarding sequence and a five-call group, described below, retaining \(R_5\) | \(\operatorname{word}(4R_5+20)\) | \(H\) |
| 7 | Five-call group beginning with \(\operatorname{word}(4R_5)\), retaining \(R_6\) | \(\operatorname{word}(D+20)\), where \(D=\operatorname{word}(16R_6)\) | \(\operatorname{word}(4R_6)\) |
| 8 | Five-call group beginning with three arguments \(D,D,\operatorname{shr}(U,7)\), retaining \(R_7\) | \(\operatorname{word}(D+20)\) | \(D\) |

The first forwarding sequence, between stores 3 and 4, contains four calls:

- A call receiving \(B\).
- A call with no explicit arguments whose result is retained.
- A call receiving that retained result.
- Another call with no explicit arguments whose result is retained.

That fourth result becomes the initial argument of the five-call group producing \(R_3\).

The second forwarding sequence, between stores 5 and 6, has the same argument/result arrangement, but begins with \(C\). Its fourth result becomes the initial argument of the five-call group producing \(R_5\).

Store 1 has no preceding visible validation or indirect call. The shifted input \(H\) is computed after that store in the recovered sequence.

Stores 7 and 8 target the same calculated address but have different payload expressions. They are separated by five calls. Both are observable effects; the later write does not justify eliminating the earlier one.

### Selector-based byte processing

Define the first selector index:

\[
T=\operatorname{shr}(C\mathbin{\&}\text{0x7fffffff},7).
\]

The first selector address is \(T+1\).

Each selector operation reads one unsigned byte and uses that byte’s numeric value plus 17 as the destination address. Thus, every selector-derived destination lies between 17 and 272 inclusive. This is a byte-to-address calculation, not a pointer-sized load.

After store 8, a five-call group beginning with \(\operatorname{word}(4R_7)\) produces an unsigned result \(E\). Define:

\[
F=\operatorname{word}(4E).
\]

The first byte operation reads the selector at \(T+1\) and writes \(\operatorname{byte}(F)\) to the selected destination.

Next, a call receives three identical explicit arguments \(F,F,F\). A following call has no explicit arguments and retains its result. That result starts another five-call group, whose final result is \(R_8\).

After those seven calls, the selector at \(T+1\) is read again. The selected destination receives:

\[
\operatorname{byte}\!\left(
\operatorname{shr}(E\mathbin{\&}\text{0x3fffffff},20)
\right).
\]

The two selector reads are distinct. A callee or an earlier write may change selector memory, so the destinations need not match.

### Input-derived byte store and clearing

A five-call group beginning with \(\operatorname{word}(4R_8)\) produces unsigned result \(G\). Define:

\[
J=\operatorname{word}(4G),\qquad
K=\operatorname{shr}(H,7).
\]

A direct byte store writes \(\operatorname{byte}(J)\) to \(K+28\).

Immediately afterward, the function reads the selector at \(T+1\) and clears the byte at the selected address. The direct store precedes this read. If \(K+28=T+1\), that store can change which byte is cleared.

The next call receives \(K\). A second call has no explicit arguments; its result has no explicit subsequent use. A five-call group then begins with:

\[
\operatorname{shr}(G\mathbin{\&}\text{0x3fffffff},6).
\]

Its final result is \(R_9\). After this group, another fresh read at \(T+1\) selects a destination byte to clear.

A further five-call group begins with \(\operatorname{word}(4R_9)\) and produces unsigned result \(L\). Define:

\[
Q=\operatorname{shr}(L\mathbin{\&}\text{0x3fffffff},25),
\qquad
V=\operatorname{word}(1024J+1).
\]

The function reads one byte at \(V\), then clears the byte at the address equal to the read value plus 17.

The complete pre-repetition byte-access sequence is therefore:

| Order | Explicit read | Explicit write |
|---|---|---|
| 1 | Selector at \(T+1\) | \(\operatorname{byte}(F)\) at selected destination |
| 2 | Fresh selector at \(T+1\) | Truncated shifted value derived from \(E\) |
| 3 | None | \(\operatorname{byte}(J)\) at \(K+28\) |
| 4 | Fresh selector at \(T+1\) | Zero at selected destination |
| 5 | Fresh selector at \(T+1\) | Zero at selected destination |
| 6 | Selector at \(V\) | Zero at selected destination |

Intervening calls occur as described above. This table does not make the accesses adjacent or independent.

### Entry to repetition

After the first clearing operation selected through \(V\), a call receives \(2Q\). A following call has no explicit arguments and supplies the initial carried value for the repeating phase.

At this point \(Q\) is between 0 and 31 inclusive. Consequently, the second explicit argument of the first iteration’s initial call is zero.

### Repeating phase

Every completed iteration contains:

- A five-call group whose initial call receives the carried value and \(\operatorname{shr}(Q,8)\). Its final result is retained for the next group.
- A fresh byte read at \(V\), followed by a zero-byte store at the read value plus 17.
- A five-call group beginning with four times the retained result, subject to the conditional 32-bit interpretation.

If the second group’s final result is \(N\), the next carried value is \(\operatorname{word}(4N)\), and the next \(Q\) is \(\operatorname{word}(N\,2^{26})\).

There is no visible termination condition.

After an iteration, the next iteration’s second explicit argument is conditionally:

\[
(N\mathbin{\&}63)\,2^{18}.
\]

The explicit calculation of \(V\) does not change during repetition. Its memory contents may change.

The initial indirect call of the first iteration and the corresponding call reached through the back edge use different visible target expressions. Their dependency identities must not be assumed equal merely because both begin the same repeated call pattern.

## Side effects

Before repetition, the visible body performs:

- **Eight 32-bit stores**, in the order specified above.
- Five explicit selector-byte reads: four at \(T+1\), then one at \(V\).
- Six explicit byte stores: three calculated payload bytes and three zero bytes.
- Indirect calls interleaved with those accesses.

Each completed iteration adds ten indirect calls, one selector-byte read at \(V\), and one zero-byte store to a selector-derived destination.

These counts exclude any memory effects performed by callees.

There is no explicit allocation, release, ownership transfer, locking, validation, error report, rollback, or recovery path. Their absence from the visible body does not establish their absence from callees.

No ownership guarantee is established for memory addressed by the function. In particular, passing a calculated value to an indirect call does not prove that the callee receives a pointer, borrows memory, takes ownership, or preserves any global state.

All reached accesses require usable addresses and suitable access permissions. Calculated addresses may alias one another, selector storage, or externally visible memory. Acceptance must preserve access order and fresh reads rather than treating the sequence as independent field assignments.

A failure can leave earlier writes and callee effects observable. No transactional behavior is shown.

## Dependencies

The supplied mapping identifies only **F-6b36adf26ce8**. It provides no opaque function IDs or contracts for the indirect callees. Accordingly, no additional dependency identities are assigned here.

Established target information is limited to:

- The first call target depends on incoming return-link state plus a fixed displacement.
- Later targets are calculated from embedded address-like constants or a code-location reference.
- Calls visibly carry zero, one, two, or three explicit arguments.
- Selected 32-bit call results feed later arguments, memory addresses, payloads, or loop state.

The actual callee signatures, target validity, instruction-set execution mode, hidden inputs, state preservation, ownership behavior, error behavior, and return guarantees are **documented_only**.

Nothing establishes that similar invocation patterns refer to one shared dependency or that distinct target expressions necessarily identify different executable functions.

## Acceptance examples

These are conditional static deductions, not runtime results. Each assumes execution reaches the relevant operation, required accesses succeed, and the stated 32-bit interpretation applies.

### Initial scaling and low addresses

For \(S=1\), the first word store writes \(\text{0x01000000}\) at \(\text{0x04000014}\).

For \(S=0\), it writes zero at address 20. No visible check rejects that address.

### Unsigned input

For \(U=\text{0xffffffff}\):

- \(H=\text{0x00ffffff}\).
- The three-argument group between word stores 7 and 8 receives \(\text{0x01ffffff}\) as its third argument.
- \(K=\text{0x0001ffff}\).
- The direct byte-store destination is \(\text{0x0002001b}\).

These right shifts do not sign-extend.

### Eight distinct word-store events

If execution reaches byte processing, all eight listed word-store events have occurred, even if some destinations alias or some payloads are equal.

For \(R_6=3\), \(D=48\). Store 7 writes 12 at address 68. Five calls then occur, and store 8 writes 48 at address 68. An intervening callee may observe the first value.

### Fresh selector reads

If the first selector read returns 5, its destination is address 22.

If an intervening call changes that selector byte to 9, the next selector-derived write targets address 26. Reusing the earlier byte would not preserve the visible behavior.

### Byte truncation

For \(E=\text{0x12345678}\), the first selected payload is \(\text{0xe0}\). The later selected payload derived from \(E\) is \(\text{0x23}\).

Both writes have width one byte, not one word.

### Direct-store aliasing

If \(K+28=T+1\) and \(\operatorname{byte}(J)=9\), the direct byte store changes the selector to 9. Without intervening modification, the immediately following selector read therefore selects address 26 for clearing.

### Loop transition

The first iteration’s second explicit argument is zero.

If an iteration ends with \(N=1\), the next carried value is 4 and the next second argument is \(\text{0x00040000}\).

If it ends with \(N=64\), the next carried value is 256 and the next second argument is zero.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered path faithfully represents executable instructions.
- The effect of invalid-instruction warnings and omitted unreachable regions on completeness.
- Additional entries, exits, or effects absent from the supplied representation.
- Actual address width, alignment rules, byte order, memory mapping, and fault handling.
- Exact machine behavior corresponding to signed multiplication and signed left shift.
- Indirect-callee identities, signatures, hidden inputs, global effects, ownership effects, and return behavior.
- Calling-convention state beyond the visible arguments and return-link dependence.
- Concurrent or asynchronous modification, memory ordering, and device-memory semantics.
- Intended data format, application-level purpose, and ownership model.

Repeated patterns may suggest structured manipulation, but they do not establish a structure layout. Unconditional repetition establishes the visible path’s continuation, not universal non-return across unknown callee behavior or unrecovered execution paths.

## Evidence

This contract is based on the supplied static representation’s visible argument widths, arithmetic, access widths, call ordering, retained results, and unconditional repetition.

The eight word stores are independently accounted for in order. In particular, the two writes at \(\operatorname{word}(D+20)\) remain separate events with a five-call group between them.

Comments, address-like constants, the prior draft, and reviewer feedback are not treated as authority for purpose or runtime behavior. Recovery warnings limit confidence in completeness.

No implementation access, runtime execution, smoke run, or tests have been performed. Acceptance examples are conditional deductions, not observed execution results.
