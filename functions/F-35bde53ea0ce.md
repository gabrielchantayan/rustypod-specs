<!-- Trusted provenance: F-35bde53ea0ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-35bde53ea0ce

## Purpose

F-35bde53ea0ce performs an ordered sequence of indirect calls and memory accesses, then enters an unconditional loop containing further indirect calls and byte stores.

The static evidence does not establish an application-level purpose. In particular, it does not establish that the accessed addresses represent objects, arrays, devices, allocated storage, or communication interfaces.

The nominal interface has one 32-bit argument and no return value. Execution also depends on additional entry-register state, unresolved call results, and memory contents. The visible path has no normal return and no explicit error-reporting mechanism.

This contract describes the supplied static representation. Reported instruction-decoding and control-flow defects limit confidence that it completely represents the underlying executable.

## Interface

### Explicit input

The explicit argument is represented as a signed 32-bit integer. Let **P** denote its value and associated 32-bit bit pattern.

The first visible memory operation stores P as a 32-bit value at the address calculated as \(4P+20\). No validation precedes this store.

### Additional inputs

The representation consumes entry state outside the nominal argument list:

- **S**: an unsigned 32-bit register value used in address calculations, stored values, and call arguments.
- **L**: a link-register-derived value used to calculate the first indirect target.
- **E**: a secondary 32-bit register result exposed after an early call sequence. Its producer and calling-convention meaning are not established.

P alone does not determine execution. Ordinary results from unresolved calls and bytes read from memory supply further inputs as execution progresses.

### Widths and arithmetic

This contract uses:

- \(W(x)\): the low 32 bits of x.
- \(B(x)\): the low 8 bits of x.
- Logical right shifts for explicitly unsigned quantities.

Expressions written with W describe the conventional 32-bit modular interpretation. They do not resolve operations represented as signed multiplication, signed addition, or signed left shift. For negative operands or overflow in those operations, portable arithmetic behavior is **documented_only**.

This qualification applies to address formation and call arguments as well as stored values. In particular, the repeated multiplication used to form a selector address and the signed shifts used in the loop require machine-level confirmation.

The visible memory access widths are:

- 32-bit stores.
- Unsigned 8-bit reads.
- 8-bit stores.

Byte order for 32-bit stores, address-space width, alignment requirements, memory permissions, and fault semantics are not established.

### Outputs, preconditions, and termination

There is no visible ordinary return value.

Observable outputs comprise direct memory writes, indirect call targets and arguments, and any additional effects of those calls.

Successful progress through any prefix requires that its memory accesses and control transfers be valid and that its calls return. These requirements are environmental preconditions, not checks performed by this function.

If every call returns and every access completes, execution remains in the final loop indefinitely. There is no visible local recovery for a fault, invalid target, or nonreturning dependency.

## Behavior

### Dependency notation

**F-U01** through **F-U14** identify pre-loop call sequences. **F-U15** and **F-U16** identify recurring loop sequences. **F-U17** and **F-U18** identify the two individual transition calls immediately before the loop.

These are opaque, document-local dependency labels, not established function identities. A sequence contains multiple unresolved call sites; its calls need not reach the same function.

A **five-call sequence** consists of:

- An initial call with the stated explicit arguments.
- Four subsequent calls with no explicit arguments shown.
- A final ordinary result supplied by the fifth call, where that result is consumed.

“No explicit arguments shown” does not establish that a call consumes no machine-register inputs.

Let **R1** through **R16** denote the final ordinary results of the correspondingly numbered sequences where used. These names describe result bit patterns, not application-level types.

### Initial word-write phase

The initial store precedes every dependency call.

Define:

\[
A=W(8S),\qquad H=S\mathbin{\mathrm{>>}}8,\qquad J=S\mathbin{\mathrm{>>}}16.
\]

The first four call sequences and their surrounding stores have the following observable order:

| Position | Required operation |
|---|---|
| Entry | Store P as 32 bits at \(4P+20\). |
| F-U01 | Five calls; the initial call receives A. Its target is calculated from \(L+4094\). |
| After F-U01 | Store A as 32 bits at \(W(A+20)\). |
| F-U02 | Five calls; the initial call receives \(W(4R1)\). |
| After F-U02 | Consume secondary result E; define \(T=W(E\times2^{24})\), and store T at \(W(E\times2^{26}+20)\). |
| F-U03 | Five calls; the initial call receives T. |
| After F-U03 | Store T at \(W(4R3+20)\). |
| F-U04 | Five calls; the initial call receives \(W(4R3)\). |
| After F-U04 | Define \(V=W(4R4)\), and store V at \(W(16R4+20)\). |

All stores in this table are 32-bit stores. F-U02’s ordinary final result is not visibly consumed; E is a separate register-derived result.

No bounds, null-address, alignment, permission, or target-validity check is visible.

### Forwarding sequences and remaining word stores

F-U05 contains nine calls. Its initial call receives V. The second call supplies an ordinary 32-bit result to the third call. The fourth supplies another ordinary 32-bit result to the fifth. Calls six through nine show no explicit arguments. R5 is the ninth call’s result.

After F-U05, V is stored as 32 bits at \(W(4R5+20)\).

F-U06 is a five-call sequence whose initial call receives \(W(4R5)\). Define \(Q=W(4R6)\). After the sequence, Q is stored as 32 bits at \(W(16R6+20)\).

F-U07 has the same nine-call forwarding shape as F-U05, but its initial argument is Q. Repetition of this shape does not establish equivalent targets or behavior. After F-U07, J is stored as 32 bits at \(W(4R7+20)\).

F-U08 is a five-call sequence whose initial call receives \(W(4R7)\). Define:

\[
D=W(16R8),\qquad
K=(Q\mathbin{\mathrm{AND}}0x7fffffff)\mathbin{\mathrm{>>}}7.
\]

After F-U08, the function stores \(W(4R8)\) as 32 bits at \(W(D+20)\).

F-U09 is a five-call sequence whose initial call receives three explicit arguments, in order:

1. D.
2. D.
3. \(H\mathbin{\mathrm{>>}}7\), equivalently \(S\mathbin{\mathrm{>>}}15\).

After F-U09, D is stored as 32 bits at \(W(D+20)\). This repeats the preceding calculated destination, but not necessarily its stored value. The intervening dependencies may also have modified that location.

F-U10 is a five-call sequence whose initial call receives \(W(4R9)\). Its final result R10 is treated as unsigned in the subsequent calculations.

### Selector-indirected byte stores

A selector-indirected store has two distinct memory effects:

- Read one unsigned byte from the specified selector address.
- Add 17 to that byte’s numeric value and use the sum as the destination of an 8-bit store.

Thus the destination is between 17 and 272 inclusive. The selector byte is not itself the destination value. Every occurrence requires a fresh read; values cannot be reused across occurrences merely because the selector address is unchanged.

Immediately after F-U10, the function reads the selector at \(K+1\) and stores \(B(4R10)\) at the selected destination.

F-U11 then performs seven calls:

- The first receives \(W(4R10)\) three times.
- The second produces an ordinary result passed to the third.
- The remaining four show no explicit arguments.

R11 is the seventh call’s result. One target in this sequence is formed relative to a code-address reference rather than an integer-like temporary; its identity remains unresolved.

After F-U11, a fresh selector read at \(K+1\) determines the destination of a store containing:

\[
B\left((R10\mathbin{\mathrm{AND}}0x3fffffff)\mathbin{\mathrm{>>}}20\right).
\]

F-U12 is a five-call sequence whose initial call receives \(W(4R11)\). Its final result R12 is treated as unsigned.

After F-U12, two stores occur in this order:

- A direct byte store of \(B(4R12)\) at \((J\mathbin{\mathrm{>>}}7)+28\), equivalently \((S\mathbin{\mathrm{>>}}23)+28\).
- A fresh selector read at \(K+1\), followed by a zero-byte store at the selected destination.

The direct destination is between 28 and 539 inclusive. It may alias the selector location or another accessed location. Consequently, its placement before the following selector read is significant.

F-U13 contains seven calls. The first receives \(J\mathbin{\mathrm{>>}}7\). The second shows no explicit arguments. The third receives:

\[
(R12\mathbin{\mathrm{AND}}0x3fffffff)\mathbin{\mathrm{>>}}6.
\]

The remaining four show no explicit arguments. R13 is the seventh call’s result.

After F-U13, another fresh selector read at \(K+1\) determines the destination of a zero-byte store.

F-U14 is a five-call sequence whose initial call receives \(W(4R13)\). Its final result R14 is treated as unsigned.

### Transition into the loop

Define the initial loop state:

\[
C=(R14\mathbin{\mathrm{AND}}0x3fffffff)\mathbin{\mathrm{>>}}25,
\]

\[
M=W(W(4R12)\times1024+1).
\]

Under modular arithmetic, \(M=W(R12\times4096+1)\). The represented multiplication used for this address includes a signed intermediate, so the equivalence is conditional on the stated machine interpretation.

Initial C lies between 0 and 31 inclusive.

A fresh selector read from M determines the destination of a zero-byte store. This store precedes both transition calls:

- F-U17 receives \(2C\).
- F-U18 shows no explicit arguments. Its ordinary result becomes the initial carried first argument for F-U15.

The first target of F-U15 also has entry-specific state. Let **\(B_{\mathrm{entry}}\)** denote the fixed target-base reference installed before F-U18, and **\(B_{\mathrm{repeat}}\)** the different fixed reference installed during F-U16. These labels are descriptive contract notation, not dependency identities.

### Unconditional loop and carried target selection

Each iteration performs F-U15, one selector-indirected zero-byte store, and F-U16, in that order.

F-U15 is a five-call sequence. Its initial call receives:

- The carried first argument.
- \(C\mathbin{\mathrm{>>}}8\).

Its initial call target depends on the iteration:

| Iteration | Initial F-U15 target expression |
|---|---|
| First iteration | \(B_{\mathrm{entry}}+4094\) |
| Every subsequent iteration | \(B_{\mathrm{repeat}}+4094\) |

The two bases are distinct fixed references in the representation. They identify distinct unresolved target expressions, not proven distinct function identities. Equivalent behavior, different behavior, or address aliasing cannot be assumed.

After F-U15 returns R15, the function freshly reads the selector at M and stores zero at the selected destination.

F-U16 is a five-call sequence whose initial call receives \(W(4R15)\). During this sequence, before its fifth call, the target-base state used by the next F-U15 initial call is assigned \(B_{\mathrm{repeat}}\). That assignment occurs on every completed passage through F-U16; it does not restore \(B_{\mathrm{entry}}\).

R16 supplies the next carried first argument, \(W(4R16)\), and the next C, interpreted as \(W(R16\times2^{26})\).

The first iteration’s second explicit argument is necessarily zero because initial C is at most 31. Under 32-bit modular interpretation, later iterations receive:

\[
(R16\mathbin{\mathrm{AND}}63)\times2^{18}
\]

as their second explicit argument, using the previous iteration’s result.

M remains the selector-address expression derived from R12; the visible loop does not replace it with a new result-derived address.

There is no visible exit condition, iteration bound, explicit delay, or error branch.

## Side effects

Before entering the loop, the visible body performs:

- **Ten 32-bit stores.**
- **Six byte stores:** four selected through fresh reads at \(K+1\), one direct store derived from S, and one selected through a fresh read at M.
- **Five selector-byte reads:** four at \(K+1\) and one at M.

Each completed loop iteration adds one selector-byte read at M and one selected zero-byte store. A partially completed iteration may already have performed that store before a later call prevents further progress.

The loop also carries argument state and call-target state. Its initial F-U15 target expression changes after the first passage through F-U16 and remains the recurring expression thereafter.

Reads and writes may alias. A write may affect a later selector, a dependency’s inputs, or other accessible state. No non-aliasing guarantee is established.

Unknown callees may perform additional memory access, synchronization, I/O, allocation, release, or nonlocal control transfer. Their effects are not bounded by the visible accesses.

No allocation or release is explicit in this body. Ownership transfer, retained references, object lifetimes, and release responsibilities are **documented_only**. Passing an integer-shaped argument does not establish whether a dependency treats it as a scalar, address, handle, or something else.

No explicit status reporting, logging, rollback, or fault handling is visible.

## Dependencies

All indirect targets remain unresolved.

F-U01 through F-U18 identify call sequences or individual calls for ordering purposes only. The dependency mapping establishes the identity of F-35bde53ea0ce, not its callees.

The relevant target-selection distinctions are:

- F-U01’s first target depends on L.
- Other targets use fixed reference-and-offset expressions, including a code-address-relative expression within F-U11.
- F-U15’s first target uses \(B_{\mathrm{entry}}+4094\) only on first entry.
- F-U16 installs the different base used by \(B_{\mathrm{repeat}}+4094\) for every subsequent F-U15 entry.

No dependency is assumed to preserve globals or memory, return normally, validate addresses, acquire ownership, or release storage. The visible use of local carried state does not establish a complete machine calling convention.

Resolving target identities, implicit arguments, secondary results, and register-preservation rules requires additional evidence.

## Acceptance examples

These are conditional consequences of static evidence. No runtime tests have been performed.

### First observable write

If P is 3 and the initial access is valid, the function stores the 32-bit value 3 at address 32 before invoking any dependency.

If that access fails, successful continuation is not promised.

### Zero additional entry state

If S is zero:

- A, H, and J are zero.
- F-U01’s first explicit argument is zero.
- The following word store writes zero at address 20.
- F-U09’s third explicit argument is zero.
- The direct byte store after F-U12 targets address 28.

Dependency results still determine other addresses and values.

### Fresh selector reads and aliasing

If one selector read returns 5, that occurrence writes to address 22. If a later read returns 9, its destination is 26.

In particular, if the direct store after F-U12 changes the byte at \(K+1\), the immediately following selector-indirected store must use the resulting byte, not a value retained from an earlier read.

### Byte truncation

If R10 is 65, the first selected byte store writes 4 because \(4\times65=260\) and \(B(260)=4\).

The later selected store based on R10 uses the masked right-shift expression independently.

### Pre-loop store accounting

If execution reaches F-U17 after all preceding accesses complete, six byte stores have occurred: four through \(K+1\), one direct, and one through M.

The first loop iteration adds another byte store only after completing F-U15 and its selector read.

### Loop arguments and target transition

If R14 has bits 25 through 29 set, initial C is 31. F-U17 receives 62, and the first F-U15 call receives zero as its second explicit argument.

That first F-U15 call uses \(B_{\mathrm{entry}}+4094\).

If the first iteration completes with R16 equal to 1, the next F-U15 call receives first argument 4 and second argument 262144 under modular interpretation. Its target uses \(B_{\mathrm{repeat}}+4094\), not the entry expression. Later iterations continue using the recurring expression.

### No normal completion

If every dependency returns and every access completes, a normal return after any finite number of iterations contradicts the visible unconditional loop.

## Uncertainties

- **Control-flow fidelity — documented_only.** The representation reports bad instruction data and removal of unreachable regions. Missing or inaccurately represented behavior remains possible.
- **Application intent — unknown.** No higher-level operation is established.
- **Calling convention — documented_only.** Extra entry registers, implicit arguments, secondary results, and register preservation require confirmation.
- **Target identity — documented_only.** The entry and recurring F-U15 target expressions differ, but their resolved function identities and behavioral relationship are unknown.
- **Arithmetic — partially established.** Unsigned masks and logical shifts are explicit. Signed overflow, signed shifts, address wrapping, and pointer interpretation require platform confirmation.
- **Memory model — unknown.** Endianness, mapped regions, alignment, device semantics, and concurrency guarantees are absent.
- **Dependency behavior and ownership — unknown.** Side effects, failure modes, termination, and memory responsibilities are not supplied.
- **Fault response — unknown.** No local handling is visible; platform behavior is unspecified.
- **Runtime behavior — unverified.** No runtime tests have been performed.

## Evidence

Established static evidence comprises the nominal interface, additional register inputs, access widths, ordered indirect calls, result forwarding, explicit memory accesses, arithmetic expressions, and unconditional loop.

The pre-loop byte-store count follows from four selector-indirected stores using \(K+1\), one direct byte store, and one selector-indirected store using M.

The loop-target transition follows from the first F-U15 call consuming target-base state established before loop entry, and F-U16 replacing that state with a different fixed reference before the next iteration. Subsequent iterations repeat that replacement.

Numerical ranges, modular equivalences, and acceptance examples are deductions subject to the stated arithmetic and execution conditions. They are not runtime observations.

No application intent or dependency ownership policy is inferred from the address patterns alone.
