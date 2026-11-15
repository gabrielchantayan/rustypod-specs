<!-- Trusted provenance: F-4c4c4a3cbd0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c4c4a3cbd0a

## Purpose

The available static evidence establishes a long, straight-line sequence of indirect calls, computed-address reads, byte writes, and 32-bit writes. It does not establish a reliable application-level purpose.

The function repeatedly scales values returned by dependencies, uses those scaled values as addresses or address components, and writes both derived values and zeros. Some byte writes select their destination through another byte read. These operations resemble manipulation of encoded references or packed state, but that interpretation is **inferred intent**, not a supported interface guarantee.

The visible path ends at an invalid-instruction boundary. Successful return, completion of a larger operation, and any intended recovery behavior are **documented_only**.

This contract describes the supplied static representation, conditional on execution reaching each operation and on its arithmetic and call representation matching the executable. It does not establish that the entire path is executable.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use is visible. |
| Second argument | 32 bits | Passed to the first indirect dependency call. |
| Third argument | 32 bits | No explicit use is visible. |
| Fourth argument | Unsigned 32 bits | Logically shifted right by three; the result becomes the first visible 32-bit store. |

Unused explicit arguments must not be assumed irrelevant to an undocumented calling convention or incorrectly recovered signature.

### Additional entry state

The visible behavior also depends on:

- An incoming link-like machine value used to calculate the first call target.
- An incoming register-like value used as an address for two early byte reads and as a shifted dependency argument.
- Memory at all computed read and write addresses.
- Results and other machine-state effects of indirect calls.
- A later secondary machine result that is not represented as an ordinary explicit return value.

These requirements prevent specification of this function solely as a transformation of its four declared arguments.

### Outputs and completion

No ordinary return value is established. Observable outputs are memory changes, dependency interactions, and the eventual execution outcome.

The visible representation terminates at an invalid-instruction boundary rather than an ordinary return. Whether actual execution traps, transfers control elsewhere, or differs because the representation is incomplete remains unknown.

### Arithmetic and layout notation

The following notation describes observable values, not implementation identifiers:

- \(W(x)\): the low 32 bits of \(x\).
- \(B(x)\): the low eight bits of \(x\).
- \(L(x)=W(4x)\).
- \(T(x)=W(16x)\).
- \(H(x)=B((x\mathbin{\&}\texttt{0x3fffffff})\mathbin{\gg}20)\).
- \(Q(x)=(x\mathbin{\&}\texttt{0x3fffffff})\mathbin{\gg}1\).
- \(R8(a)\): an unsigned byte read at address \(a\).
- \(R32(a)\): a 32-bit read at address \(a\).

Derived addresses below use the apparent 32-bit machine arithmetic. This interpretation does not establish portable signed-overflow behavior. Several expressions are represented using signed intermediate types; executable-level arithmetic and fault behavior remain unverified.

A byte-selected destination has address \(R8(a)+\texttt{0x11}\). It is not an access to a field at offset `0x11` within the object at \(a\). The selector is zero-extended, so these destinations lie between `0x11` and `0x110`.

No endianness, alignment tolerance, object boundaries, or structure type is established.

## Behavior

### Execution and dependency ordering

The represented path has no visible conditional branches or loops. Every later event is conditional on all preceding calls and memory accesses completing.

Most regions contain five consecutive indirect calls: a leading call with the explicitly shown arguments, followed by four calls with no explicitly shown arguments. The last call frequently supplies a value used in the next memory operation. “No explicitly shown arguments” does not establish an empty machine-level argument state.

The descriptions below preserve the ordering of explicit memory effects and significant dependency arguments. Intervening dependencies may read or change memory. Consequently, separated writes must not be collapsed, and separated reads must not be treated as interchangeable.

### Initial state propagation

1. The first dependency receives the second explicit argument. Four further indirect calls follow. Let \(a\) denote the final result of this group.

   A 32-bit write stores the fourth argument shifted logically right by three at address \(W(L(a)+\texttt{0x14})\).

2. Another five-call group begins with argument \(L(a)\). Let \(b\) denote its final result, and let \(p=L(b)\).

   The function reads a byte at the incoming register-derived address plus one. Let its value be \(s_1\). It writes \(B(p)\) at \(s_1+\texttt{0x11}\).

3. A five-call group begins with three arguments: \(s_1\gg3\), \(p\), and the incoming register-derived value shifted left by two. Let \(c\) denote its final result.

   The selector byte is read again from the original incoming address plus one. Let this second value be \(s\). The function writes \(H(b)\) at \(s+\texttt{0x11}\).

   The two selector reads are distinct. Calls between them may change the selector.

4. A five-call group begins with \(L(c)\). After it, a 32-bit write stores \(p\) at \(W(4s+\texttt{0x14})\).

5. A five-call group begins with four arguments: \(s,s,s,s\gg1\). Let \(d\) denote its final result.

   A 32-bit write stores \(p\) at \(W(L(d)+\texttt{0x14})\).

The retained selector \(s\), rather than a later reread of that byte, drives several subsequent operations.

### Scaled-result propagation

The next operations form several distinct producer/store stages:

| Stage | Leading dependency argument | Final result | Subsequent explicit write |
|---|---|---|---|
| First | \(L(d)\) | \(e\) | Store \(L(e)\) at \(W(T(e)+\texttt{0x14})\). |
| Second | \(L(e)\) | \(f\) | Store \(L(e)\) at \(W(L(f)+\texttt{0x14})\). |
| Third | \(L(f)\) | \(g\) | Store \(L(g)\) at \(W(T(g)+\texttt{0x14})\). |

Each row contains a five-call group. All listed stores are 32-bit.

An additional indirect call receives \(L(g)\). A following no-explicit-argument call produces a 32-bit value that is passed into another five-call group. Let that group’s final result be \(h\).

The function stores \(Q(L(g))\) at \(W(L(h)+\texttt{0x14})\). The mask precedes the right shift; this is not an unrestricted division of the original result.

A five-call group beginning with \(L(h)\) produces \(i\). The function stores \(L(i)\) at \(W(T(i)+\texttt{0x14})\).

It then reads a 32-bit value at \(W(L(i)+\texttt{0x14})\), shifts that read value logically right by three, and supplies it to another five-call group. Let the final result be \(j\). The function stores \(L(i)\) at \(W(L(j)+\texttt{0x14})\).

The read address uses \(L(i)\), whereas the immediately preceding store address uses \(T(i)\). They must not be assumed to identify the same location.

### Selector placement and byte clearing

A five-call group beginning with \(L(j)\) produces \(k\). The following effects occur in order:

1. Store \(s\), as a 32-bit value, at
   \[
   W((s\gg3)\times\texttt{0x80000}+\texttt{0x24}).
   \]

2. Define \(t=T(k)\) and \(q=Q(k)\). A five-call group begins with arguments \(t,L(k)\).

3. Store \(s\), as a 32-bit value, at \(W(t+\texttt{0x24})\).

4. Read a 32-bit value at \(W(t+\texttt{0x74})\) and pass it into a five-call group. Let \(m\) denote the final result.

5. Read the byte at \(W(t+1)\). Write \(H(k)\) to the byte-selected destination.

6. An indirect call receives \(L(m)\). A following no-explicit-argument call supplies a value to another five-call group. Let its final result be \(n\).

7. Reread the byte at \(W(t+1)\). Write zero to this newly selected byte destination.

8. A five-call group begins with \(L(n)\).

9. Write a zero byte at \(W(q+\texttt{0x0f})\).

10. Define \(u=W(q\times\texttt{0x400})\). Read the byte at \(W(u+1)\), and write zero to its selected byte destination.

The earlier nonzero byte write and subsequent zero write are separate observable events. Their destination selectors are reread, and they need not designate the same byte.

### Clearing word state

The next dependency receives four arguments: three copies of \(W(m\ll23)\), followed by \(W(m\ll25)\). A following no-explicit-argument call supplies a value to a five-call group. Let its final result be \(o\).

The subsequent clearing sequence is:

| Order | Observable effect |
|---:|---|
| 1 | Read a byte at \(W(u+1)\); retain it as \(z\), and write zero at \(z+\texttt{0x11}\). |
| 2 | Execute a five-call group beginning with \(L(o)\). |
| 3 | Store a 32-bit zero at \(z+\texttt{0x44}\). |
| 4 | Store a 32-bit zero at absolute address `0x24`. |

The selector \(z\) is retained across the dependency group. The word-clear address does not multiply \(z\) by four.

Next, a dependency receives \(0,\texttt{0x98},u,\texttt{0x98}\). Its following result-producing call feeds another dependency, whose following result-producing call feeds a five-call group. Let the group’s final result be \(v\).

The function clears a 32-bit word at \(W(L(v)+\texttt{0x44})\), then clears the 32-bit word at absolute address `0x24` again.

A five-call group beginning with \(L(v)\) produces a represented 64-bit result. Treat its low and high 32-bit halves as \(\ell\) and \(r\), respectively.

- A dependency receives \(L(\ell)\).
- A 32-bit zero is stored at \(W(2r+\texttt{0x44})\).
- A 32-bit zero is stored at absolute address `0x24`.

The next call receives zero, a secondary machine result associated with the preceding dependency, and that secondary result shifted left by two. Its width is represented as 32 bits; its semantic meaning is unknown.

After another no-explicit-argument call, a five-call group begins with
\[
W((q\mathbin{\&}\texttt{0x3fffff})\ll7).
\]
Let its final result be \(w\). The function clears the 32-bit word at \(W(L(w)+\texttt{0x44})\), then clears the word at absolute address `0x24` for a fourth time.

These four writes to absolute address `0x24` are not redundant by contract: dependencies intervene, and other computed addresses may alias it.

### Final word propagation

A five-call group beginning with \(L(w)\) produces \(x\). A further dependency receives \(L(x)\) and returns \(y\). The function clears the 32-bit word at \(W(L(y)+\texttt{0x14})\).

Define
\[
M=W((q\mathbin{\&}\texttt{0x3fffff})\ll7).
\]

The remaining word operations are:

| Leading dependency arguments | Final result | Subsequent 32-bit write |
|---|---|---|
| \(y,y,y,M\) | \(a_1\) | Zero at \(W(L(a_1)+\texttt{0x14})\). |
| \(L(a_1)\) | \(a_2\) | \(L(a_2)\) at \(W(T(a_2)+\texttt{0x14})\). |
| \(L(a_2)\) | \(a_3\) | \(L(a_2)\) at \(W(L(a_3)+\texttt{0x14})\). |
| \(L(a_3)\) | \(a_4\) | \(L(a_4)\) at \(W(T(a_4)+\texttt{0x14})\). |

Each row contains a five-call group.

A dependency then receives \(L(a_4),T(a_4)\). A following no-explicit-argument call supplies a value to a five-call group whose other leading argument is
\[
W((q\mathbin{\&}\texttt{0x3fffff})\ll4).
\]
Let its final result be \(a_5\). The function stores \(M\) at \(W(L(a_5)+\texttt{0x14})\).

### Final byte propagation and visible endpoint

A five-call group beginning with \(L(a_5)\) produces \(a_6\). Define
\[
v_b=W(q\times\texttt{0x4000000}).
\]

The remaining explicit byte operations are:

1. Read the selector at \(W(v_b+1)\); write \(B(L(a_6))\) to its selected destination.
2. Call a dependency with \(L(a_6)\). A following result-producing call feeds a five-call group, whose final result is \(a_7\).
3. Reread the selector at \(W(v_b+1)\); write \(H(a_6)\) to its selected destination.
4. A five-call group beginning with \(L(a_7)\) produces \(a_8\).
5. Reread the selector at \(W(v_b+1)\); retain this byte and write \(B(4a_8)\) to its selected destination.
6. Pass the retained selector, zero-extended to 32 bits, to a dependency. A following result-producing call feeds a five-call group, whose final result is \(a_9\).
7. Read a selector at \(W(8a_8+1)\); write \(H(a_8)\) to its selected destination.
8. Execute five more indirect calls, beginning with argument \(L(a_9)\).

The last selector read uses \(8a_8\), not \(v_b\). The visible representation then reaches the invalid-instruction boundary.

## Side effects

The explicit memory footprint includes:

- Byte writes at addresses chosen by unsigned byte selectors.
- Zero-byte writes at a derived address with offset `0x0f`.
- 32-bit stores at computed addresses with offsets `0x14`, `0x24`, and `0x44`.
- Four explicit 32-bit zero stores to absolute address `0x24`.
- Byte selector reads and two explicit non-selector 32-bit reads.

These offsets do not establish a structure layout. Address expressions may alias one another, and dependency effects may introduce further overlap.

No allocation, release, ownership transfer, reference-counting rule, or lifetime guarantee is established. In particular, a dependency result used as an address must not be presumed to designate newly allocated or owned memory.

There is no visible validation, rollback, error-status test, or cleanup branch. If a call or memory access fails, no recovery is established. Earlier writes may already be observable.

## Dependencies

- **F-4c4c4a3cbd0a-D001**: the entry-state-dependent first indirect call. Its target is calculated from incoming machine state; it receives the second explicit argument.
- **F-4c4c4a3cbd0a-D002**: the remaining indirect-call sites, considered as an unresolved dependency set.

D002 is an opaque label for a set of call sites, not a claim that all sites invoke the same function. No independently resolved dependency identities were supplied.

The evidence represents:

- Calls with zero through four explicit arguments.
- Consumed 32-bit results.
- One consumed 64-bit result split into two halves.
- One consumed secondary machine result.

Dependency semantics, failure behavior, ownership, memory effects, global preservation, and non-returning behavior are **documented_only**. Every post-call read must use memory as it exists after that call. Retained scalar values in the specification do not imply preservation of any particular physical register.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Fourth-argument truncation.** If the fourth argument is `0x0000000f`, the first explicit word store contains `1`. If it is `0xffffffff`, that store contains `0x1fffffff`. The destination still depends on the first call group’s final result.

2. **Byte truncation.** If \(b=\texttt{0x00000041}\), then \(L(b)=\texttt{0x104}\). The first selected byte write stores `0x04`, while later word propagation retains `0x00000104`.

3. **Selector reread.** If the first early selector read returns `2` and the second returns `7`, the corresponding byte writes target `0x13` and `0x18`. Reusing the first selector for both is not equivalent.

4. **Mask before byte conversion.** If \(b=\texttt{0xffffffff}\), the masked right shift used by \(H\) yields `0x3ff`, and the stored byte is `0xff`.

5. **Retained selector across calls.** If \(z=3\), the later selector-derived word clear targets `0x47`, even if an intervening dependency changes the byte from which \(z\) was read.

6. **Distinct scaled locations.** If \(i=1\), the scaled-result store targets `0x24`, while the following explicit word read targets `0x18`. The read cannot generally be replaced with the preceding stored value.

7. **Repeated absolute clears.** If a dependency modifies absolute address `0x24` between two explicit clears, the later clear still occurs. Combining the four clears into one would change the observable interaction sequence.

8. **Split-result addressing.** If the represented 64-bit result has low half `3` and high half `5`, the next dependency receives `12`; the associated word clear targets `0x4e`. These effects use different halves.

9. **Final destination selection.** If \(a_8=2\), the final byte-selector read occurs at address `0x11`, regardless of the earlier value of \(v_b\), absent coincidental address equality.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied straight-line representation correctly captures executable control flow.
- The true calling convention, complete parameter list, and ordinary return behavior.
- The meaning of the incoming machine-state values.
- Valid address ranges, mapping of low addresses, alignment requirements, and byte order.
- Whether reads and writes concern ordinary memory, device state, runtime-managed storage, or incorrectly represented operations.
- Exact arithmetic behavior where signed intermediate types appear.
- Dependency identities, argument interpretation, errors, and memory ownership.
- Whether the apparent endpoint is an actual invalid instruction or an artifact of incomplete static recovery.
- Any application-level invariant or successful-operation criterion beyond the listed conditional effects.

The absence of visible checks is not evidence that arbitrary inputs or addresses are valid.

## Evidence

Established static evidence consists solely of the supplied representation: argument widths, explicit arithmetic, call ordering, computed memory accesses, repeated stores, the represented split return, and the truncated endpoint.

The supplied warning information reduces confidence in recovered control flow and indicates possible overlap among named memory locations. This contract therefore describes accesses by width and address expression rather than treating names as proof of independent objects.

The interpretation as encoded-reference or packed-state manipulation is inferred intent only. No implementation access, independent disassembly, dependency inspection, or runtime verification supports a stronger claim.

**No runtime tests have been performed.**
