<!-- Trusted provenance: F-9125b6fce2ff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9125b6fce2ff

## Purpose

F-9125b6fce2ff performs an ordered sequence of 24 indirect calls, one four-byte memory write, two one-byte reads, and two one-byte writes. Selected call results supply later arguments and determine write addresses. A byte at an entry-derived address is read twice; each observed value selects a byte-write destination.

The supplied representation ends at an invalid-instruction boundary, not an ordinary return. It does not establish a complete, normally returning operation.

No application-level purpose is established. In particular, the calls, address calculations, and scaled results do not establish allocation, parsing, device access, or table initialization.

This contract distinguishes:

- **Established static evidence:** operations, types, and ordering visible in the supplied representation.
- **Inferred intent:** interpretations that are not behavioral requirements.
- **Unknowns:** behavior requiring target-language conventions, an execution environment, or dependency implementations.

All acceptance examples are conditional. No runtime tests have been performed.

## Interface

### Entry inputs

The displayed interface has no explicit parameters and no declared return value. Its body nevertheless consumes three entry-carried values:

| Contract name | Role |
|---|---|
| Entry word | Unsigned 32-bit quantity used both as a bit field and in address formation. |
| Payload word | Uninterpreted four-byte value written to memory. |
| Entry continuation | Signed integer-like value used to construct the first indirect call target. |

These are descriptive contract names, not a recovered public calling convention. The evidence does not establish how a caller supplies them, which registers must be preserved, or whether this is a valid standalone entry point.

The entry word supplies:

- Its value shifted right by 12 as the first call’s explicit argument.
- Its value plus one as the numeric basis of both byte-read addresses.

The addition is represented as unsigned arithmetic. Under the represented 32-bit model, it wraps modulo \(2^{32}\) before conversion to a byte address. The validity and meaning of that resulting address remain environment-dependent.

### Outputs

There is no established normal return value. Observable behavior within the represented sequence consists of:

- Indirect calls and their explicit arguments.
- A four-byte payload write.
- Two byte reads and two byte writes.
- Any effects of the indirect callees.
- Reaching the invalid-instruction boundary, if execution proceeds that far.

A void declaration does not establish normal completion.

### Widths and conversion boundaries

The represented arithmetic uses unsigned 32-bit words, signed integer results, unsigned eight-bit selectors, and four-byte forwarded data. Signed calculations are described using the represented 32-bit integer model; an authoritative target ABI is not supplied.

Three distinct operations must not be conflated:

1. Shifting an unsigned control word left by two produces an unsigned 32-bit result, reduced modulo \(2^{32}\).
2. That result is assigned to a signed integer temporary.
3. The signed temporary subsequently supplies a one-byte conversion and two call arguments.

If the unsigned shift result is at most `0x7FFFFFFF`, its signed conversion preserves its numeric value under the represented 32-bit model. If it exceeds that value, the supplied material does **not** establish the conversion result or whether conversion can interrupt execution. A bit-preserving conversion to a negative signed value is a possible target convention, not an unconditional requirement.

The byte stores use conversions to a one-byte character type. The source establishes their width, but does not independently establish plain-character signedness or out-of-range character-conversion rules. Byte-pattern examples below use ordinary low-eight-bit narrowing where stated.

The four-byte payload has no established encoding, byte order, or alignment guarantee.

### Execution prerequisites

Following the entire visible sequence requires every preceding call to return, every target to be executable, every explicit memory access to be permitted, and all required arithmetic and conversions to have continuing behavior.

The function contains no visible checks enforcing those prerequisites.

## Behavior

### Initial calls and payload write

F-000000000001 receives the entry word shifted right by 12. Its target is formed from the entry continuation plus 4,094.

F-000000000002 through F-000000000005 then execute in order. No explicit arguments are shown for those calls. F-000000000005 supplies a signed result, designated the **first index**.

The payload word is written as four bytes to the address formed by multiplying the first index by four and adding 20. No separate table base is present in that expression. Treating it as an ordinary table access would be inferred intent.

After the write, F-000000000006 receives the first index multiplied by four. Thus the payload write precedes that call.

F-000000000007 through F-000000000009 follow. F-00000000000a then supplies an unsigned 32-bit **control word**, denoted \(U\).

Signed multiplication and destination formation have no visible overflow checks. Negative or overflowing signed-index cases are not assigned unconditional address semantics by this contract.

### Unsigned shift followed by signed conversion

Before the first byte read, the function computes:

\[
Q=(U\ll2)\bmod 2^{32}
\]

This is an unsigned calculation. It then converts \(Q\) to a signed integer, designated \(S\).

For \(Q\leq\texttt{0x7FFFFFFF}\), \(S=Q\) under the represented integer model.

For larger \(Q\), the conversion is **documented_only**: the supplied representation does not determine the target’s conversion rule. In particular, the contract does not unconditionally identify \(S\) with either the unsigned number \(Q\) or a signed reinterpretation of its bits.

If conversion does not continue normally, the following read, write, and calls are not guaranteed to occur.

### First selector read and byte write

After obtaining \(S\), the function reads an unsigned byte at the entry-word-plus-one address. Call the observed value the **first selector**.

It adds 17 to that selector and uses the result as the byte-write address. Since the selector is unsigned and eight bits wide, the numeric destination lies between 17 and 272 inclusive. This is not an offset from the entry word.

The stored value is obtained by converting **\(S\)** to the one-byte character type. It is not specified as a direct conversion of \(Q\).

If the target uses both:

- A bit-preserving unsigned-to-signed conversion when \(Q\) is outside the signed range, and
- Low-eight-bit narrowing for the character conversion,

then the stored byte is:

\[
(4U)\bmod256
\]

For in-range \(Q\), only the character-narrowing convention is needed for that byte-pattern formula. Without the applicable conversion semantics, the formula is not an unconditional result.

### Three-argument call and forwarding

After the first byte write, F-00000000000b receives these explicit arguments, in order:

1. The signed temporary \(S\).
2. The same signed temporary \(S\).
3. The unsigned expression \((U\ll4)\bmod2^{32}\).

The first two arguments are equal because the same converted value is reused. They are not independent conversions, nor are they unconditionally unsigned modular results. The third argument remains an unsigned shift expression and does not pass through \(S\).

F-00000000000c next returns an uninterpreted four-byte value. F-00000000000d receives that value directly.

F-00000000000e through F-000000000010 then execute. F-000000000011 supplies a signed **second index**.

### Second selector read and byte write

The function now reads the byte at the entry-word-plus-one address again. This is a distinct read after the intervening calls.

Call the observed value the **second selector**. It is retained both for selecting the second byte-write address and for a later call argument.

The second destination is the second selector plus 17. The value converted for storage is obtained by clearing control-word bits 30 and 31 and shifting right by 20:

\[
T=(U\mathbin{\&}\texttt{0x3FFFFFFF})\gg20
\]

\(T\) is an unsigned value from 0 through 1,023. It does not depend on \(S\), the first selector, or the second index.

With low-eight-bit character narrowing, the second stored byte is:

\[
T\bmod256=(U\gg20)\bmod256
\]

That byte contains control-word bits 20 through 27. Bits 28 and 29 can survive in \(T\), but are discarded by low-byte narrowing; bits 30 and 31 were cleared earlier.

The second read must not be replaced by reuse of the first selector. An intervening callee may modify the source byte, and the first byte write may itself alias that address.

### Final calls and boundary

After the second byte write:

1. F-000000000012 receives the second index shifted left by two.
2. F-000000000013 through F-000000000015 execute.
3. F-000000000016 supplies a signed **third index**.
4. F-000000000017 receives the third index shifted left by two and the second selector widened to an unsigned integer.
5. F-000000000018 executes.

The selector argument is the previously captured unsigned byte, in the range 0–255. It is not reread after the intervening calls.

The signed left shifts are distinct from the earlier unsigned control-word shifts. Negative operands and signed overflow remain unresolved.

The representation then ends at an invalid-instruction boundary. It establishes no normal return, error result, or recovery procedure beyond that point.

## Side effects

### Explicit memory-access order

| Order | Access | Address and value |
|---|---|---|
| 1 | Four-byte write | Payload word at four times the first index, plus 20. |
| 2 | One-byte read | Entry word plus one, with represented unsigned address arithmetic. |
| 3 | One-byte write | First selector plus 17; character conversion of \(S\). |
| 4 | One-byte read | The same entry-derived address, read again. |
| 5 | One-byte write | Second selector plus 17; character conversion of \(T\). |

The conversion producing \(S\) occurs before access 2. The table lists explicit data-memory accesses, not all intervening computation or callee effects.

Aliasing is not excluded:

- The payload write may overlap either later selector read.
- A byte-write destination may equal the selector-read address.
- Both byte writes may target the same address.
- Callees may access or modify any of these locations.

Consequently, neither selector read nor either write may be eliminated merely because the address expressions appear related.

### Ownership and external state

No explicit allocation, deallocation, acquisition, or release appears in the visible body. This does not establish the absence of such effects in callees.

The forwarded four-byte value has no established ownership meaning. Passing it does not prove transfer, borrowing, retention, or release.

No dependency is assumed to preserve globals, memory, resources, or unknown implicit inputs. Reuse of scalar values in the representation does not establish an authoritative register-preservation convention.

### Errors and partial execution

There is no visible error-status branch, retry, rollback, or cleanup path.

A nonreturning callee, invalid target, failed memory access, problematic conversion, or invalid arithmetic may prevent later operations. The representation does not define the resulting exception mechanism or any external recovery.

Earlier effects are not visibly undone.

## Dependencies

These opaque IDs identify call sites, not recovered implementation identities. Different IDs may resolve to the same runtime target. “None shown” refers only to explicit arguments.

| Dependency | Explicit data use |
|---|---|
| F-000000000001 | Entry-derived target; receives entry word shifted right by 12. |
| F-000000000002 | None shown. |
| F-000000000003 | None shown. |
| F-000000000004 | None shown. |
| F-000000000005 | Returns the first signed index. |
| F-000000000006 | Receives the first index multiplied by four, after the payload write. |
| F-000000000007 | None shown. |
| F-000000000008 | None shown. |
| F-000000000009 | None shown. |
| F-00000000000a | Returns unsigned control word \(U\). |
| F-00000000000b | Receives \(S\), \(S\), and unsigned \(U\) shifted left by four. |
| F-00000000000c | Returns an uninterpreted four-byte value. |
| F-00000000000d | Receives that forwarded value. |
| F-00000000000e | None shown. |
| F-00000000000f | None shown. |
| F-000000000010 | None shown. |
| F-000000000011 | Returns the second signed index. |
| F-000000000012 | Receives the second index shifted left by two. |
| F-000000000013 | None shown. |
| F-000000000014 | None shown. |
| F-000000000015 | None shown. |
| F-000000000016 | Returns the third signed index. |
| F-000000000017 | Receives the third index shifted left by two and the widened second selector. |
| F-000000000018 | Final visible call; none shown. |

Except for the first entry-derived target, targets are constructed from embedded code-location-like values and fixed displacements. The displacement is 4,094 except at F-00000000000b, where it is 4,076.

Target validity, instruction-set state, dependency semantics, implicit arguments, and ownership rules are unknown.

## Acceptance examples

All examples assume valid targets and accesses, returning preceding calls, and arithmetic that permits reaching the described operation. Byte-pattern expectations additionally assume low-eight-bit character narrowing.

### Payload ordering

If the first index is 3, the payload write targets numeric address 32. F-000000000006 subsequently receives 12.

The write must precede that call. This does not establish that address 32 is writable.

### Control word with an in-range shifted result

For \(U=\texttt{0x12345678}\):

- \(Q=\texttt{0x48D159E0}\), within the signed range.
- \(S\) therefore has that positive numeric value.
- The first stored byte is `0xE0`.
- F-00000000000b receives signed \(S\), signed \(S\), and unsigned `0x23456780`.
- The second stored byte is `0x23`.

No out-of-range unsigned-to-signed conversion is needed in this example.

### Signed-conversion boundary

For \(U=\texttt{0x1FFFFFFF}\), \(Q=\texttt{0x7FFFFFFC}\), so \(S\) is representable and positive.

For \(U=\texttt{0x20000000}\), \(Q=\texttt{0x80000000}\). The unsigned result is established, but \(S\) is not determined by the supplied material. Consequently, the first two arguments of F-00000000000b and the first stored byte cannot be specified unconditionally.

If execution reaches that call, its third unsigned argument is `0x00000000`, independently of the unresolved signed conversion.

### All-bits-set control word

For \(U=\texttt{0xFFFFFFFF}\), the unsigned shift result is `0xFFFFFFFC`.

Under a target convention that preserves those bits when converting to signed 32-bit form:

- \(S=-4\).
- The first stored byte is `0xFC`.
- F-00000000000b receives signed arguments −4 and −4, followed by unsigned `0xFFFFFFF0`.

Without that conversion convention, the first byte and first two arguments are unresolved, and continuation past the conversion is not guaranteed.

If execution reaches the second write, its low-byte result is `0xFF`; that calculation does not depend on \(S\).

### Different or coincident selectors

If the first selector is 7 and the second is 200, the byte writes target addresses 24 and 217 respectively. F-000000000017 later receives 200 as its second explicit argument.

If both selectors are 7, both writes target address 24. The later write uses the second-write value. Final memory contents are not guaranteed because subsequent callees may modify the location.

### Interrupted sequence

If F-00000000000a does not return, neither selector read nor either byte write is reached. The payload write has already occurred, with no visible rollback.

## Uncertainties

The following remain **documented_only**:

- **Instruction validity:** Warnings identify bad instruction data and truncated control flow. The representation is not proof of a valid executable function.
- **Entry and ABI:** Entry provenance, actual calling convention, integer ABI, and register-preservation rules are not independently established.
- **Unsigned-to-signed conversion:** Out-of-range conversion of \(Q\) to \(S\) is unresolved. This affects the first byte-write conversion, the first two arguments of F-00000000000b, and possibly continuation.
- **Character conversion:** Plain-character signedness and out-of-range conversion behavior are not independently established. Low-byte examples state their narrowing assumption.
- **Signed arithmetic:** Index multiplication, signed left shifts, and signed target/address additions may encounter negative values or overflow.
- **Addresses:** Pointer conversion, executable mappings, alignment, and low-address accessibility are unknown.
- **Dependencies:** Semantics, implicit inputs, errors, global effects, resource effects, and ownership are unknown.
- **Termination:** The invalid-instruction boundary does not establish a particular exception, exit, or return.
- **Concurrency:** No synchronization or atomicity guarantee is established.
- **Purpose:** Higher-level interpretations remain inferred intent, not requirements.

## Evidence

This contract rests solely on static inspection of the supplied representation.

The representation supports the ordered 24 indirect calls, one four-byte write, two byte reads, two byte writes, result forwarding, and final invalid-instruction boundary. It also explicitly distinguishes the unsigned left-shift calculation from its assignment to a signed integer, and shows that signed temporary supplying the first byte conversion and the first two arguments of the subsequent three-argument call.

No dependency bodies, authoritative ABI, memory map, target conversion specification, or runtime observations were supplied. No runtime tests have been performed.
