<!-- Trusted provenance: F-ddffece45a05; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ddffece45a05

## Purpose

F-ddffece45a05 describes an ordered sequence of indirect calls, byte accesses, and 32-bit memory accesses. The supplied static evidence does not establish a domain operation, data structure, protocol, or successful completion condition.

The represented behavior includes selector-mediated writes to low addresses, extraction of bytes from arguments and returned values, propagation of 32-bit words, and address calculations involving scaled results. The path ends at an invalid-instruction boundary rather than a demonstrated ordinary return.

This contract describes the represented path. It does not establish that instruction recovery, the function boundary, or the apparent calling convention accurately describes executable behavior.

## Interface

### Explicit inputs

| Position | Represented width | Directly established use |
|---|---:|---|
| First | Unsigned 32 bits | Supplies two byte values and repeated arguments to the first indirect call. |
| Second | 32 bits, unspecified interpretation | No direct use shown. |
| Third | 32 bits, unspecified interpretation | No direct use shown. |
| Fourth | Unsigned 32 bits | Supplies write addresses, selector-read addresses, and masked or scaled values. |

No direct use of the second and third positions does not prove that their incoming machine-register contents are irrelevant to dependencies with incomplete signatures.

Additional inputs include an incoming address-like register, an incoming link-related register, memory contents, indirect-call targets, and incompletely specified post-call registers.

### Results and preconditions

No explicit return value, success indicator, recoverable error result, or ordinary-return guarantee is established.

Following the whole represented path requires every encountered memory access to be permitted and every dependency to return normally. No checks establish address validity, alignment, object bounds, or dependency availability. These are execution preconditions, not validations performed by this function.

### Descriptive notation

The following labels identify retained values, not implementation identifiers:

- **A**: first explicit input.
- **D**: fourth explicit input.
- **R**: incoming address-like register.
- **W(x)**: low 32 bits of an arithmetic result.
- **B(x)**: low eight bits of a value.
- **Selector write through x**: read an unsigned byte at `W(x + 1)`, then write the specified byte at that byte’s zero-extended value plus `0x11`.

A selector write therefore has two ordered memory effects: a byte read followed by a byte write. Its destination is between `0x11` and `0x110`, inclusive; it is not necessarily within an object addressed by x.

Unsigned 32-bit arithmetic is modulo \(2^{32}\). Byte stores retain eight bits; word stores retain 32 bits. Expressions below involving represented signed intermediates describe their 32-bit bit-pattern interpretation conditionally. Signed overflow, signed left shifts, and conversion into signed storage remain **documented_only** where executable semantics are not established.

## Behavior

### Ordering and dependency boundaries

The displayed path has no conditional branches, loops, validation failures, or early returns. A fault or non-returning dependency can nevertheless prevent any later operation.

All memory accesses occur in the order described. Calls between accesses are observable boundaries:

- Later reads observe memory after intervening dependencies.
- Repeated selector reads are distinct, even when their address expressions match.
- Retained values remain inputs to later operations; newly read memory must not be substituted for them.
- No dependency is assumed to preserve globals or arbitrary memory.
- Aliasing is unrestricted. Earlier writes can affect later reads.
- Calls displayed without arguments have unspecified argument contracts, not established zero-argument interfaces.

A “five-call group” below means one call with the described arguments followed by four calls with unspecified displayed arguments. When its final result is used, that result belongs to the fifth call. These groups do not imply common dependency identity or behavior.

### Initial input-derived writes

The first selector write uses R and stores `B(A)`.

A five-call group follows. Its first call receives A twice and R shifted left by two. The first target is derived from the incoming link-related register plus `0xffe`. The fifth call supplies a result.

A fresh selector read through R then selects a store of `B(A >> 22)`, extracting bits 22–29 of A. The retained result, shifted left by two, becomes the first argument of another five-call group. Let **P** be that group’s final 32-bit result.

Next, in order:

- Store `B(P << 2)` at `W(D + 0x0f)`.
- Read the selector through `W(D × 0x400)` and store the same byte at its selected destination.

Define **S = (D AND 0x003fffff) << 8**. S occupies bits 8–29 and is divisible by 256.

Three calls follow. A fresh selector read through `W(D × 0x400)` then selects a store of `B((P AND 0x3fffffff) >> 20)`, extracting bits 20–27 of P.

The last result from those three calls, shifted left by two, starts another five-call group. Afterwards, a selector write through `W(S × 0x80000)` stores zero.

The unsigned arithmetic is:

| Expression | Equivalent 32-bit value | Possible values |
|---|---|---|
| `W(S × 0x80000)` | `W(D << 27)` | 32 values, determined by D’s low five bits |
| `W(S × 0x100000)` | `W(D << 28)` | 16 values, determined by D’s low four bits |

Neither expression is generally zero. For D equal to one, they are `0x08000000` and `0x10000000`, respectively.

### Retained selectors and word-to-byte transfers

A five-call group receives `W(S << 3)` twice, S, and zero in its first call. Its final result is retained.

The selector byte at `W(S × 0x80000 + 1)` is read again and retained as **T**. Zero is written at `T + 0x11`.

Another five-call group starts with the preceding result shifted left by two. After it returns, a 32-bit word is read at `T + 0x74`, and its low byte is written at `T + 1`.

Define **Scaled = W(S × 0x100000)**, using the multiplier `0x100000`, or 1,048,576. A call receives Scaled, S, `(S AND 0x1fff) << 17`, and T.

The next call supplies a represented 64-bit result. A following call receives its low 32-bit half, high 32-bit half, repeated low half, and `(S AND 0x0fff) << 18`. Four more calls follow. Their final result, multiplied by four, becomes a base for a word read at offset `0x74` and a low-byte write at offset one.

A five-call group receives that base in its first call. A selector write through Scaled then stores zero.

Under the unsigned interpretation, this selector read occurs at `W(D << 28) + 1`, not generally at address `0x1`.

A call next consumes an incompletely specified post-call register. The following call supplies a retained base, **U**. A five-call group receives U in its first call and supplies another result.

The selector through Scaled is then read afresh. Its destination receives the high byte of Scaled:

\[
B(\mathrm{Scaled} \mathbin{\mathrm{>>}} 24)
= (D \mathbin{\mathrm{AND}} 0x0f) \ll 4.
\]

This is distinct from the earlier zero-valued write. Only when D’s low four bits are zero do both Scaled-based selector reads use address `0x1` and the later stored byte equal zero. Even then, intervening calls can change the selector and therefore the destination.

### Indirect word reads and propagated fields

The preceding group’s final result, shifted left by two, starts another five-call group. Its final result, shifted left by two, becomes **V**.

The function reads a 32-bit word at `W(U + 4)`, retaining it as **Q**, then performs a zero-valued selector write through Q.

A five-call group receives V three times. Afterwards, a byte is read at `(Q >> 2) + 1` and retained as **E**; zero is written at `E + 0x11`.

Another five-call group starts with its preceding result shifted left by two. Then, in order:

1. Read a 32-bit word at `W(V + 4)`, retaining **L**.
2. Read a 32-bit word at `E + 0x74`, retaining **M**.
3. Write `B(M)` at `E + 1`.

A call receives M, two copies of V shifted left by 19, and the latest result shifted left by two. Another call supplies a result that is passed onward together with `M >> 2`. Define **H = L >> 22**.

After four further calls, their final result multiplied by four becomes another transfer base. A word is read at its offset `0x74`; its low byte is written at offset one.

That full word is retained across a five-call group whose first call receives the transfer base. It is then stored at `W(4 × Z + 0x14)`, where **Z** is an incompletely specified post-call register.

A call receives Z. The next call’s result is passed to another call with two copies of `(H AND 0x1ff) << 21`. Four further calls supply a result selecting a word store at its four-times-scaled address plus `0x14`. The stored value is `W(H << 23)`; only H’s low nine bits can survive.

### Scaled stores and overlapping updates

A five-call group starts with the preceding selected result multiplied by four. Let **J** be its final result, and retain:

- **K = W(J × 4)**.
- **N = J AND 0x3fffffff**.
- **G = N >> 2**.

K is stored as a word at `W(J × 0x10 + 0x14)`.

A five-call group receives N first and returns **I**. N is stored at `W(I × 4 + 0x14)`. Another five-call group receives `W(I × 4)` first and returns **O**.

Define **X = W(I × 8)**. The next direct effects are a byte store of `B(X)` at `W(M + 1)`, followed by a word store of `W(O << 2)` at `W(X + 0x34)`.

A call receives `W(O << 2)`. The next call’s result is passed to a further call, followed by four calls whose final result is represented as 64 bits. Retain **Y = O AND 0x3fffffff**.

The function then writes `B(X)` at `Y + 1` and the returned high half at `W(X + 0x34)`. This word store uses the same computed destination as the earlier O-derived word store; intervening dependency effects remain possible.

A five-call group starts with the returned low half shifted left by two. Define **F = W(I × 0x10)**. Afterwards:

- Write `B(F)` directly at `((O AND 0x3fffffff) >> 20) + 0x11`.
- Write `B(K)` at `W(F + 0x0d)`.

The first destination uses bit extraction directly, without reading a selector.

A call receives two copies of the latest result shifted left by two. Its successor returns a value passed onward both unchanged and shifted right by three. Four more calls follow. The function then writes `B(F)` at absolute address `0x11` and again writes `B(K)` at `W(F + 0x0d)`.

### Additional byte fields and absolute address `0x0d`

A five-call group starts with the preceding result shifted left by two and returns **C**.

The function stores K at `Y + 0x14` and `B(Y << 1)` at `Y + 0x0d`. Retain:

- **Base32 = W(C × 0x20)**.
- **Value128 = W(C << 7)**.
- **Small = (C AND 0x07ffffff) >> 17**, ranging from zero through `0x3ff`.

Three calls supply a result. Value128 is stored at `W(Base32 + 0x14)`, and that result’s low byte after a left shift by two is stored at `W(Base32 + 0x0d)`.

A five-call group starts with the same shifted result. Afterwards, `B(G << 1)` is written at `Small + 7`, followed by a selector write through Base32 containing `B(Small)`.

A call receives three copies of the earlier three-call result shifted left by 21. The next call returns **Next**. A five-call group receives Next first.

The selector at `W(Base32 + 1)` is then read and retained as **Sel**; zero is written at `Sel + 0x11`. Another five-call group starts with the latest result shifted left by two.

The function writes `B(Next >> 3)` at `W(Base32 + 0x0d)`, then Sel at absolute address `0x0d`. A call receives `2 × Sel`, `0x98`, and `0x260`. The doubled selector ranges from zero through 510.

After another call, a dependency receives an incompletely specified unsigned post-call register shifted right by three. Four calls supply a result, which is shifted left by two and passed to another dependency.

Before the next call, the function writes `B(Next >> 3)` at `2 × Sel + 0x0d`, then writes `0xc0` at absolute address `0x0d`. When Sel is zero, these destinations coincide and the latter store immediately replaces the former byte.

### Final propagation and selector writes

A five-call group starts with the preceding dependency result shifted left by two and returns **Last**.

The function stores `W(Last << 2)` at `2 × Sel + 0x24`, reads a word at `2 × Sel + 0x74` as **Root**, and retains **Payload = W(Last << 5)**.

A five-call group receives Root, `W(Root << 1)`, Payload, and `(Root AND 0x7fffffff) >> 2` first. Afterwards, Payload is stored at `W(Root + 0x24)`.

A word is read at `W(Root + 0x74)` and passed to a five-call group. Its final represented 64-bit result supplies a high half selecting a word store at its four-times-scaled address plus `0x14`. The stored value is `Next >> 3`. Its low half, shifted left by two, is retained.

A call receives that retained value three times and the high half shifted right by three. The next call returns another represented 64-bit value. A five-call group receives its low half, high half, repeated low half, and high half shifted right by three. Its final result selects a word store of the retained scaled low half at its four-times-scaled address plus `0x14`.

Another five-call group starts with that selected result multiplied by four and returns **Final**. Define **Final4 = W(Final << 2)**.

The closing sequence is:

- Selector write through Root containing `B(Final4)`.
- Three calls.
- Fresh selector write through Root containing `B((Final AND 0x3fffffff) >> 20)`.
- Five-call group starting with the preceding result shifted left by two.
- Zero-valued selector write through `W(Final4 × 0x400)`.
- Five-call group receiving `W(Final4 << 3)` twice, Final4, and the preceding result shifted left by two.
- Fresh zero-valued selector write through `W(Final4 × 0x400)`.
- Five-call group starting with the preceding result shifted left by two.

The representation then reaches its invalid-instruction boundary. No ordinary return is established.

## Side effects

Direct effects comprise byte reads and writes, 32-bit reads and writes, and indirect calls. Repeated offsets do not establish a shared record layout.

Absolute addresses `0x0d` and `0x11` are written. Selector destinations occupy low addresses. Whether these addresses denote RAM, device mappings, inaccessible memory, or recovery artifacts is unknown.

No direct allocation or release operation is identified. Dependencies may nevertheless allocate, release, retain, or invalidate memory. Ownership transfer, lifetime rules, synchronization, reentrancy, and thread safety are **documented_only**.

No rollback is shown. Earlier writes and dependency effects may remain after a fault or non-returning call.

## Dependencies

The only supplied opaque function identity is F-ddffece45a05. No opaque identities or contracts are available for its indirect dependencies; their identities remain **documented_only** rather than being invented.

Established dependency interactions include repeated arguments, scaled addresses, retained words, represented 64-bit results split into 32-bit halves, and additional post-call registers with incomplete contracts.

Displayed target expressions use incoming machine state or embedded address-like values with offsets. No dependency may be assigned familiar library semantics or assumed to preserve globals, maintain memory validity, transfer ownership, release resources, or return normally.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume the relevant accesses are valid and preceding dependencies return.

1. **Input-byte extraction.** For A equal to `0xabc12345`, the first selector store contains `0x45`; the later A-derived store contains `0xaf`. Each uses its own selector read.

2. **Changed selector.** If successive selector reads return 3 and 9, their destinations are `0x14` and `0x1a`. An intervening dependency can cause this difference.

3. **Result-byte extraction.** For P equal to `0x12345678`, the two P-derived byte values are `0xe0` and `0x23`.

4. **Nonzero scaled addresses.** For D equal to one, the S-derived selector bases are `0x08000000` and `0x10000000`. Their reads occur at `0x08000001` and `0x10000001`. At the latter base, the first write contains zero; the later fresh-selector write contains `0x10`.

5. **Different retained bit ranges.** For D equal to `0x10`, the `0x80000` multiplier yields `0x80000000`, while the `0x100000` multiplier yields zero. Only the latter segment therefore reads its selector at address `0x1` and later writes a zero high byte.

6. **Selector reuse is limited.** T remains the byte used for the subsequent accesses at `T + 0x74` and `T + 1`; it is not a request to reread the earlier selector location.

7. **Absolute-byte alias.** For Sel equal to zero, the store at `2 × Sel + 0x0d` is immediately followed by `0xc0` at the same address.

8. **Interrupted execution.** A non-returning dependency prevents later accesses. No represented mechanism reverses earlier effects.

9. **Terminal boundary.** Reaching the final boundary does not satisfy an ordinary-return or success-status contract.

## Uncertainties

The following remain **documented_only**:

- Instruction-recovery reliability and the apparent function boundary.
- Actual calling convention and complete indirect-call signatures.
- Meaning of represented paired returns and additional post-call registers.
- Executable handling of signed overflow, signed shifts, and signed conversions.
- Address validity, alignment, byte order, access atomicity, and fault behavior.
- Domain purpose, object layouts, ownership, and resource lifetimes.
- Aliasing with dependency-accessible memory or call-target state.
- Whether the terminal boundary represents a real trap, unrecovered instructions, or non-executable data.

Repeated field-like updates suggest structured state manipulation, but this is inferred intent only. It does not establish a particular object model or application purpose.

## Evidence

Established static evidence consists of the supplied access widths, unsigned argument types, arithmetic expressions, ordered memory effects, indirect-call boundaries, retained values, and explicit control-flow warnings.

The scaled-S identities and high-byte extraction follow from 32-bit unsigned arithmetic. They do not validate recovered control flow or imply that computed addresses are usable.

Only the supplied static representation was considered. No implementation access, dependency inspection, runtime execution, or runtime tests have been performed.
