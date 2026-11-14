<!-- Trusted provenance: F-9c06dacc9aa3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c06dacc9aa3

## Purpose

F-9c06dacc9aa3 performs a fixed, dependency-heavy sequence of indirect calls and interleaved memory mutations. Its directly visible effects include byte-field updates, word-field updates, a halfword clear, two writes to a shared global location, and a final call through address zero.

No application-level purpose is established. In particular, the evidence does not establish that the routine initializes an object, serializes a structure, processes a supervisor request, or manages a collection. Such descriptions would require knowledge of the indirect callees and the memory map.

This contract describes the visible sequence conditional on execution reaching each operation. It does not guarantee normal completion: any earlier dependency or memory access may prevent later effects.

## Interface

### Explicit inputs and return

The visible interface has four 32-bit arguments:

| Input | Established use |
|---|---|
| First argument | No direct use is visible. |
| Second argument | Passed to the first indirect dependency. |
| Third argument | No direct use is visible. |
| Fourth argument | Treated as unsigned; its bits 23–31 are passed to an early dependency. |

No explicit return value is established. A return follows the final call through address zero, but that return is reachable only if the call returns normally.

“Unused” here means unused by the visible body. It does not establish that argument registers, caller state, or other entry context are invisible to unknown dependencies.

### Additional entry state

The routine depends on state not represented by the four arguments:

- A link-register-derived value used to form the first call target.
- Three register-derived values used for addressing or bit extraction.
- A secondary result-register value captured after an early dependency.
- A late secondary result-register value interpreted as a byte pointer.
- A 32-bit global value later used as an address base.
- A three-byte global region used to construct the final global write.

These are prerequisites, not optional inputs with defaults. Their origins and validity are unknown.

### Width and layout conventions

The visible accesses distinguish:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Wide dependency result: 64 bits, split into low and high 32-bit halves.

A byte store from a wider quantity retains its low eight bits. A 64-bit result’s halves are selected numerically; that split does not by itself establish memory endianness.

For compactness, this contract uses mathematical names for retained values and address formulas. These are descriptive labels, not source identifiers.

Several operations scale a 32-bit result by four. Where the supplied representation expresses a signed shift or signed multiplication, overflow semantics are not established independently. The acceptance examples below use values for which those operations are unambiguous. A modulo-\(2^{32}\) interpretation for all such operations requires confirmation of the underlying execution semantics.

## Behavior

### Ordering and notation

The visible path contains no conditional branch or loop. Its operations follow one ordered path, subject to dependency behavior and access failures.

Dependencies are identified below by editorial opaque call-site IDs **F-U001** through **F-U114**. These IDs distinguish occurrences, not proven function identities. Different occurrences may resolve to the same function.

A range described as “no explicit arguments” means exactly that the supplied call expression shows no arguments. It does not imply that the callee has no register, global, or memory inputs.

### 1. Initial calls and first byte update

F-U001 receives the second explicit argument. F-U002 follows with no explicit arguments. F-U003 receives the fourth argument shifted logically right by 23 bits, yielding a value from 0 through 511.

F-U004 through F-U007 then execute with no explicit arguments. Retain F-U007’s 32-bit result as \(A\).

Before forwarding \(4A\), the routine:

1. Reads one byte at the entry-state address base plus 1.
2. Zero-extends that byte to obtain \(b\).
3. Writes the low eight bits of a separate entry-state unsigned value shifted right by 22 to address \(b+17\).

The destination is therefore in the numeric address range 17–272. This is an address calculation, not evidence of a particular mapped object.

F-U008 receives \(4A\). F-U009 through F-U012 have no explicit arguments; retain F-U012’s result as \(B\). F-U013 receives \(4B\) twice. F-U014 returns a 32-bit value \(C\); F-U015 receives \(C\); F-U016 follows with no explicit arguments.

### 2. First indexed clear and retained secondary result

Let \(R\) be the entry-state unsigned value used for the next indexed lookup. The routine reads one byte at \((R \mathbin{\mathrm{logical\ right\ shift}} 9)+1\), zero-extends it, and writes zero to that byte value plus 17.

It then captures a secondary result-register value as \(S\). F-U017 receives \(S\). F-U018 returns a 32-bit value \(D\). F-U019 receives \(D\) and \(C\) logically shifted right by 23.

F-U020 through F-U023 have no explicit arguments. Retain F-U023’s result as \(E\).

The routine now repeats the byte lookup using \(R\), retaining this newly read byte as \(T\), and writes zero to address \(T+17\). The lookup is a fresh read after intervening calls; it must not be replaced with the earlier byte value.

F-U024 receives \(4E\). F-U025 through F-U028 have no explicit arguments; retain F-U028’s result as \(F\). F-U029 receives \(4F\) twice.

F-U030 returns a 32-bit value \(G\). The routine stores \(G\) to shared global region **G-A** before calling F-U031 with \(G\). F-U032 follows with no explicit arguments.

### 3. Byte projection and wide-result forwarding

Using the retained byte \(T\), the routine reads a word at \(T+116\) and writes its low byte to \(T+1\).

F-U033 receives \(S\) shifted left by 11. F-U034 returns a 64-bit value. F-U035 receives, in order:

- Its low 32-bit half.
- Its high 32-bit half.
- \(G\) logically shifted right by 23.

F-U036 through F-U039 have no explicit arguments. Define \(H\) as four times F-U039’s 32-bit result.

The routine reads a word at \(H+116\), then writes its low byte to \(H+1\). F-U040 receives \(H\).

F-U041 through F-U044 have no explicit arguments. Let \(J\) be F-U044’s result and let \(K\) be its 32-bit left shift by two.

The word at \(H+116\) is read again, and its low byte is again written to \(H+1\). This second read occurs after intervening dependencies and is independently observable. It cannot be replaced with the first loaded word without additional guarantees.

F-U045 receives four arguments, in order: \(S\) shifted left by 12, \(K\), \(K\), and \(J\) shifted left by 18.

### 4. Retained word and two destination layouts

F-U046 returns a word forwarded to F-U047. F-U048 returns another word forwarded to F-U049.

F-U050 through F-U053 have no explicit arguments. Define \(L\) as four times F-U053’s result.

The routine reads a word at \(L+116\), retaining the complete value as \(W\). It writes the low byte of \(W\) to \(L+1\), then calls F-U054 with \(L\).

F-U055 through F-U058 have no explicit arguments. If F-U058 returns \(M\), the routine stores \(W\) at address \(16M+20\).

F-U059 receives the result of logically shifting \(K\) right by two and then shifting left by 16. The right shift is significant: it does not recover high bits already discarded when \(K\) was formed.

F-U060 returns a word forwarded to F-U061. F-U062 returns a word forwarded to F-U063.

F-U064 through F-U067 have no explicit arguments. If F-U067 returns \(N\), the routine stores the same retained word \(W\) at \(4N+20\). F-U068 then receives \(4N\).

These stores use distinct scaling rules: the first uses a 16-byte stride and the second a four-byte stride.

### 5. Further word-field updates

F-U069 through F-U072 have no explicit arguments. Define \(P\) as F-U072’s result shifted left by two.

The routine stores \(P\) at the address obtained by clearing the bottom two bits of \(K\) and adding 20. F-U073 then receives \(P\) twice.

F-U074 returns a word forwarded to F-U075. F-U076 returns a word forwarded to F-U077.

F-U078 through F-U081 have no explicit arguments. If F-U081 returns \(Q\), the routine stores a zero word at \(4Q+20\). F-U082 then receives \(4Q\).

F-U083 through F-U086 have no explicit arguments. F-U086 supplies a 64-bit result. Let its high half be \(U\), and let \(V\) be its low half shifted left by two.

The routine:

- Stores \(V\) at \(4U+20\).
- Reads global **G-B**, retaining its value as address base \(Z\).
- Calls F-U087 with \(V\), \(U\), \(V\), and \(V\), in that order.

The global read occurs before F-U087. A later modification of G-B by a dependency does not change the retained base \(Z\).

### 6. Global-based word capture and byte projections

F-U088 through F-U091 have no explicit arguments. If F-U091 returns \(X\), the routine stores \(V\) at \(4X+20\).

It then reads a word at \(Z+20\), retaining that word as \(Y\), and calls F-U092 with \(4X\).

F-U093 through F-U096 have no explicit arguments. F-U096 supplies a 64-bit result with low half \(I\) and high half \(O\).

Next, the routine reads one byte at \((W \mathbin{\mathrm{logical\ right\ shift}} 23)+1\). If that byte is \(d\), it writes the low byte of \(Y\) to \(d+17\).

F-U097 receives \(O\), \(O\), \(O\), and \(I\) shifted left by two.

F-U098 returns a word forwarded to F-U099. F-U100 through F-U103 have no explicit arguments. Retain F-U103’s result as \(E_2\).

The routine reads one byte at:

\[
((I \mathbin{\&} \text{0x3fffffff}) \mathbin{\mathrm{logical\ right\ shift}} 21)+1.
\]

If this byte is \(e\), the routine writes the low eight bits of \(Y\) logically shifted right by 22 to \(e+17\). This selects bits 22–29 of \(Y\).

F-U104 receives \(E_2\) shifted left by two.

### 7. Final dependency sequence and terminal writes

F-U105 through F-U108 have no explicit arguments. F-U109 receives F-U108’s result shifted left by two, twice.

F-U110 has no explicit arguments. F-U111 returns a word passed to F-U112. F-U113 returns a byte value \(q\).

After F-U113, the routine performs these effects in order:

1. Writes zero through the late secondary-result byte pointer.
2. Reads the three-byte region **G-C** and writes G-A with a 32-bit value consisting of those three bytes in bits 8–31 and \(q\) in bits 0–7.
3. Writes a zero halfword at address \((I \mathbin{\&} \text{0x3fffffff})\) logically shifted right by 7.
4. Calls F-U114 through address zero with three zero arguments.
5. Returns, if that call returns.

The halfword destination ranges numerically from 0 through 8,388,607. It is not necessarily even. No explicit alignment correction or validation precedes the store.

## Side effects

### Memory and globals

The accesses above are direct effects of this body. Unknown dependencies may perform additional reads, writes, I/O, or control transfers.

G-A is written twice directly: first with \(G\), later with the assembled value incorporating \(q\) and G-C. Intermediate calls may read or overwrite it. The first write is not guaranteed to remain visible until the second.

Repeated addresses may alias one another, globals, or memory touched by dependencies. No disjointness is established. An implementation must preserve the stated order even where an optimization would appear to remove a repeated read or store.

The offsets 1, 17, 20, and 116 establish address relationships, not a complete structure layout. Allocation sizes, field types beyond access widths, and object identities remain unknown.

### Ownership and errors

No explicit allocation, release, ownership transfer, reference-count operation, or lifetime boundary is established. A returned word must not be assumed to designate newly owned storage.

There is no visible validation, error return, retry, rollback, or cleanup path. A dependency failure or invalid access may leave earlier mutations in place. The behavior of faults, traps, nonreturning calls, and address zero is environment-dependent.

## Dependencies

F-U001–F-U113 are unresolved indirect call occurrences. Their target identities and contracts are **documented_only**. The first target also depends on entry link-register state.

F-U114 is the call through numeric address zero. Its existence is established; whether it executes valid code, traps, resets execution, or has another platform-defined effect is unknown.

No dependency is assumed to preserve globals, pointed-to memory, or unspecified register state. The contract retains only values explicitly captured by the visible body. Wide and secondary-register results require calling-convention confirmation before they can be treated as ordinary public return values.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **Fourth-argument extraction.** If the fourth argument is 0xFF800000 and execution reaches F-U003, its explicit argument is 511. For 0x007FFFFF, it is zero.

2. **Byte-indexed clear.** If the relevant lookup returns byte 200, the subsequent clear writes one zero byte to numeric address 217. It does not clear a word or write at the lookup address itself.

3. **Fresh lookup after calls.** If the first \(R\)-based lookup returns 3 and a dependency changes the lookup byte to 9, the two clears target addresses 20 and 26 respectively. Reusing the first lookup fails this criterion.

4. **Full-word retention.** If the word captured as \(W\) is 0xA1B2C3D4, the immediate byte projection writes 0xD4, while both later \(W\)-based word stores retain 0xA1B2C3D4, absent an earlier interruption.

5. **Different strides.** If both \(M\) and \(N\) are 3, their word destinations are 68 and 32 respectively. Treating both destinations as the same array layout fails this criterion.

6. **Repeated source read.** If \(H+116\) contains 0x11223344 for the first read and an intervening dependency changes it to 0x55667788, the two byte writes to \(H+1\) are 0x44 and 0x88.

7. **Captured global base.** If G-B contains address 0x1000 when read, a subsequent change of G-B to 0x2000 does not redirect the later word read from 0x1014 to 0x2014.

8. **Terminal ordering.** If F-U113 returns normally, the secondary-pointer byte clear precedes the final G-A write, which precedes the halfword clear, which precedes F-U114. A fault at the halfword clear does not imply rollback of either earlier write.

## Uncertainties

The following remain **documented_only**:

- Application intent and externally meaningful data types.
- Actual indirect target identities and whether call occurrences alias.
- Calling conventions, including secondary-register and 64-bit results.
- Validity and origin of implicit entry state.
- Memory mapping, endianness, access permissions, and alignment rules.
- Overflow behavior of operations represented with signed operands.
- Overlap among globals and smaller memory regions.
- Dependency preservation, ownership, failure, and nonreturning behavior.
- The execution meaning of the final address-zero call.

No ordinary-memory assumption is justified for the small numeric addresses. Equally, the evidence does not establish that they are erroneous: they may have platform-specific meanings.

## Evidence

**Established static evidence:** the supplied representation supports the explicit argument uses, ordered indirect call occurrences, retained-result relationships, access widths, offsets, bit extractions, direct global accesses, and terminal call described here.

**Inferred intent:** repeated offsets and scaled addresses suggest field-oriented storage or indexing. This inference is not used to assign object types, ownership, or an application purpose.

**Evidence limitations:** only the supplied static representation was available. No implementation inspection beyond that material, execution, runtime test, memory-map validation, or calling-convention validation has been performed. The conditional examples are specification checks, not observed results.
