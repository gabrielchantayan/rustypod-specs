<!-- Trusted provenance: F-f02cd183abcb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f02cd183abcb

## Purpose

The established behavior is an ordered sequence of indirect calls, integer transformations, byte and word memory accesses, and updates to shared storage. The routine ends by attempting a call to address zero with three zero-valued arguments.

No application-level purpose is established. The repeated scaling of call results by four and accesses at offsets `0x14` and `0x74` could reflect an encoded object representation, but that interpretation is not part of this contract.

This specification describes the supplied static representation. It does not establish that the routine is executable under an ordinary process memory model, that its indirect targets return normally, or that the final call reaches a valid handler.

## Interface

**Explicit inputs**

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit read in the visible body |
| Second argument | 32 bits | No explicit read in the visible body |
| Third argument | Unsigned 32 bits | Its upper nine bits are written to a computed word location |

The first two arguments are not proven irrelevant to the machine-level invocation: the recovered call signatures and calling convention are incomplete.

**Additional entry and call-produced state**

The routine also depends on:

- An entry link/control value used to compute its first indirect-call target.
- Two entry register values, designated **E5** and **E6**, treated as 32-bit values.
- Three auxiliary values exposed after indirect calls: two word-sized values and one byte pointer. Their precise producing calls and ABI status are unresolved.
- Several ordinary 32-bit call results.
- Three results represented as 64-bit quantities, subsequently separated into low and high 32-bit halves.
- Two shared word-sized storage locations and an overlapping three-byte region.

These labels describe values and storage, not public parameter names or inferred data types.

**Output**

The visible interface returns no value. There is no explicit status or error result.

Normal return is conditional on every preceding access and call completing, including the call to address zero. The listing contains a return after that call; it does not prove that execution can reach it.

**Memory and arithmetic notation**

For the behavior below:

- A **byte read** obtains an unsigned 8-bit value.
- A **word read or write** accesses 32 bits.
- A **halfword write** accesses 16 bits.
- “Low byte” means the least significant eight bits of a value, independent of memory endianness.
- Unsigned right shifts insert zero bits.
- “Scaled by four” denotes the visible multiplication or left shift by two.
- **Slot(x)** denotes the address obtained by scaling `x` by four and adding `0x14`. It does not assert a valid array or object.
- **Byte destination(x)** denotes the address obtained by reading one byte at address `x` and adding `0x11`.

The representation mixes signed integer expressions with pointer arithmetic. A 32-bit wrapping interpretation is appropriate only if confirmed for the target machine. It is not a portable-language guarantee for signed overflow or signed left shifts.

## Behavior

The following phases are ordered. An unknown call may change memory, including memory accessed by later phases. No intervening calls may be removed, reordered, or assumed harmless merely because their explicit results are unused.

### 1. Establish the first word and byte updates

The routine makes an argument-free indirect call through a target computed from the entry link/control value. It forwards that call’s word-sized result to another indirect call.

Four further argument-free calls follow. The last supplies an index. The routine writes the third explicit argument shifted right by 23 to Slot(index).

Immediately afterward, it reads a word at:

- address `0x14 + (E5 shifted right by 22)`.

This is an addition of the shifted value, not a four-byte array index.

The routine calls another dependency with the first index scaled by four, then makes four argument-free calls.

It then:

1. Computes E6 shifted right by 22.
2. Shifts the previously read word right by 22; designate this retained value **A**.
3. Reads a byte at address `E6 + 1`.
4. Writes A’s low byte to the resulting Byte destination.

A is a ten-bit value before byte truncation. The written byte therefore represents bits 22–29 of the earlier word, not its complete upper ten bits.

### 2. Issue a repeated-argument call and clear a byte

An indirect call receives one auxiliary word repeated in its first three argument positions and `E6 shifted right by 22` in its fourth position.

One argument-free call follows. The next call receives:

- `(E6 shifted right by 22) shifted right by 23`.

Under the represented unsigned 32-bit arithmetic, this argument is always zero.

Four argument-free calls follow, with the last supplying another index. Before forwarding that index, the routine:

1. Reads a byte at address `4 × A + 1`.
2. Writes zero to the resulting Byte destination.

It then calls a dependency with the new index scaled by four.

### 3. Produce a retained word and perform the first sign-selected clear

Four argument-free calls follow. The last result is scaled by four and supplied twice to another call.

An argument-free call supplies a word, designated **B**. B is passed to a dependency, followed by one argument-free call.

The routine next derives a selector from E6’s sign bit:

- Selector `1` when E6 interpreted as signed 32-bit is nonnegative.
- Selector `2` when that interpretation is negative.

It reads a byte at that selector address and writes zero to the resulting Byte destination.

This sign extraction relies on the represented arithmetic right shift.

An auxiliary word is captured for later use; designate it **H**. A dependency receives H three times.

### 4. Repeat the sign-selected clear and publish another word

An argument-free call supplies a word. The next call receives that word and `B shifted right by 23`.

Four argument-free calls follow, with the last returning an index.

The routine rereads the byte at selector address `1` or `2`, using the same sign rule as before. Designate this newly read byte **C**. It writes zero at `C + 0x11`.

C is not necessarily equal to the byte obtained during the earlier sign-selected read: intervening calls or aliasing may have changed memory.

The routine passes the latest index scaled by four to a dependency. Four argument-free calls follow; the last result, scaled by four, is supplied twice to another call.

The next argument-free call supplies a word, designated **D**. The routine:

1. Writes D to shared word storage **G1**.
2. Calls a dependency with D.
3. Makes one argument-free call.
4. Reads a word at `C + 0x74`.
5. Writes that word’s low byte at `C + 1`.

### 5. Perform repeated word-to-byte transfers

A dependency receives H shifted left by 11.

An argument-free call returns a 64-bit quantity. The next call receives its low 32-bit half, its high 32-bit half, and `D shifted right by 23`, in that order.

Four argument-free calls follow. The last supplies an index whose four-scaled value is retained as **P**.

The routine reads a word at `P + 0x74`, writes its low byte at `P + 1`, and calls a dependency with P.

Four further argument-free calls follow. The final result is designated **J**, and **U** is J shifted left by two.

The routine again reads the word at `P + 0x74` and writes its low byte at `P + 1`. This is a fresh read, not reuse of the earlier word.

A dependency then receives, in order:

- H shifted left by 12;
- U;
- U again;
- J shifted left by 18.

### 6. Retain and distribute a word

An argument-free call returns a word that is forwarded to another call. A further argument-free call returns another word, also forwarded.

Four argument-free calls follow. The last supplies an index whose four-scaled value is **Q**.

The routine:

1. Reads a word at `Q + 0x74`; designate it **W**.
2. Writes W’s low byte at `Q + 1`.
3. Calls a dependency with Q.
4. Makes four argument-free calls, obtaining an index from the last.
5. Writes W at address `16 × index + 0x14`.

This last address uses a scale of sixteen, not four.

A dependency receives `(U shifted right by 2) shifted left by 16`.

An argument-free result is forwarded to another call; a second argument-free result is forwarded to another call. Four argument-free calls then produce an index.

The routine writes W to Slot(that index), then calls a dependency with that index scaled by four.

### 7. Write a scaled link value

Four argument-free calls produce another index. Its four-scaled value is designated **L**.

The routine writes L at address:

- `(U AND 0xfffffffc) + 0x14`.

U was formed by a two-bit left shift, so its low two bits are already zero under a 32-bit bit-vector interpretation. The visible mask remains part of the address computation.

A dependency receives L twice.

### 8. Clear a word and distribute a paired result

An argument-free result is forwarded to another call. A second argument-free result is forwarded to another call. Four argument-free calls then produce an index.

The routine writes a zero word to Slot(index), and calls a dependency with that index scaled by four.

Four argument-free calls follow. The last returns a 64-bit quantity:

- Its low 32-bit half is scaled by four, producing **V**.
- Its high 32-bit half is used as an index.

The routine writes V to Slot(the high-half index).

It then reads shared word storage **G2**, retaining that value as **N**, before calling a dependency with these four arguments:

1. V;
2. The high-half index;
3. V;
4. V.

Four argument-free calls follow. The final index selects another Slot, to which V is written.

Only then does the routine read a word at `N + 0x14`; designate it **T**. Consequently, N is a previously captured address basis, while T reflects memory after the intervening calls and write.

A dependency receives the most recent index scaled by four.

### 9. Perform two late indirect byte writes

Four argument-free calls follow. The last returns a 64-bit quantity whose low half is designated **R** and high half **S**.

The routine:

1. Reads a byte at `(W shifted right by 23) + 1`.
2. Writes T’s low byte to the resulting Byte destination.
3. Calls a dependency with S repeated three times and R shifted left by two as the fourth argument.

An argument-free result is forwarded to another call. Four argument-free calls follow, yielding another index.

Before forwarding that index, the routine:

1. Clears R’s top two bits and shifts the result right by 21.
2. Adds one and reads a byte at that address.
3. Writes the low byte of `T shifted right by 22` to the resulting Byte destination.

It then calls a dependency with the latest index scaled by four.

### 10. Finish with shared-state replacement and a zero-address call

Four argument-free calls produce an index. Its four-scaled value is passed twice to another call.

Two argument-free calls follow. The second result is forwarded to another call, after which a final dependency returns an 8-bit value.

The routine then performs these operations in order:

1. Writes zero through an auxiliary byte pointer exposed by the call sequence.
2. Forms a 32-bit value whose low byte is the final dependency’s 8-bit result and whose upper 24 bits come from a separate three-byte storage region, **G3**.
3. Writes that constructed value to G1.
4. Writes a zero halfword at address `(R AND 0x3fffffff) shifted right by 7`.
5. Calls address zero with three zero arguments.
6. Returns, if that call returns.

The auxiliary pointer is not established as an ordinary return value. Its validity, source, and aliasing relationships are unknown.

## Side effects

The direct writes comprise byte clears, low-byte transfers, word stores, one halfword clear, and two writes to G1. Indirect dependencies may have additional effects.

Important observable properties include:

- Byte destinations are computed from fresh byte reads where specified.
- A word-to-byte transfer reads all 32 bits before retaining the low eight.
- The two transfers involving P are separate reads and writes, with calls between them.
- G2 is read before several calls; the word addressed through its captured value is read later.
- G1 is first assigned D and later replaced by a constructed word.
- The auxiliary-pointer clear occurs before G3 supplies the final upper 24 bits.
- No direct allocation, release, ownership transfer, or resource cleanup is established.

Distinct expressions do not establish distinct storage. Low-address accesses, computed stores, G1, G2, G3, and dependency-owned memory may alias.

## Dependencies

No resolved dependency identities are provided beyond F-f02cd183abcb itself. The following opaque identifiers distinguish unresolved dependency categories; they do not assert that all calls within a category reach the same function.

| Opaque ID | Role | Established boundary |
|---|---|---|
| F-000000000001 | Entry-relative indirect target | Called without explicit arguments; supplies a word-sized result |
| F-000000000002 | Remaining computed indirect targets | Numerous ordered calls; explicit arguments and consumed results are described above |
| F-000000000003 | Address-zero target | Invoked last with three zero arguments |

The targets within F-000000000002 are computed using several distinct displacements. Their concrete function identities, equivalence, calling conventions, and validity are unresolved.

No dependency is assumed to preserve shared memory, retain or release memory, validate an index, or return normally. “Argument-free” means no explicit arguments are shown; it does not establish that the target ignores register or global state.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Third-argument extraction.** If execution reaches the first word store with third argument `0xffffffff`, the stored word is `0x000001ff`. With third argument `0x007fffff`, it is zero. The address still depends on the preceding call result.

2. **Ten-bit extraction followed by byte truncation.** If the early word read produces `0xffffffff`, A is `0x3ff`, but the first indirect byte write stores `0xff`.

3. **Forced-zero argument.** For every unsigned 32-bit E6, shifting it right by 22 and then by 23 produces zero. A replacement that passes an unshifted E6-derived value changes the call boundary.

4. **Sign-selected address.** E6 equal to `0x7fffffff` selects byte address `1`; E6 equal to `0x80000000` selects byte address `2`. These are byte-read addresses, not final write destinations.

5. **Fresh selector read.** If an intervening dependency changes the selected byte from `0x20` to `0x30`, the later sign-selected clear targets `0x41`, not `0x31`.

6. **Word-to-byte transfer.** If a specified word read yields `0x123456ab`, the corresponding byte write stores `0xab`. A byte read from the word’s starting address is not generally equivalent without an established endianness.

7. **Different slot scale.** If the index used for the sixteen-scaled store is `3`, W is written at address `0x44`. A four-scaled store would incorrectly target `0x20`.

8. **Repeated read visibility.** If a dependency changes the word at `P + 0x74` between the two transfers, the second write must reflect the later word.

9. **Final address range.** For 32-bit R, the final halfword address lies between zero and `0x007fffff`. It can be odd; no alignment correction is visible.

10. **Final shared value.** If the final returned byte is `0x5a`, G1’s low byte becomes `0x5a`. Its upper 24 bits come from G3, not necessarily from G1’s previous value.

11. **Failure before completion.** If any dependency does not return or an access traps, later effects are not established. No rollback or recovery path is visible.

## Uncertainties

**documented_only**

- The routine’s domain purpose and any logical object layout.
- The correctness and completeness of recovered signatures.
- The machine-level origin and preservation of auxiliary register outputs.
- Concrete indirect-target identities and target equivalence.
- Whether low addresses and address zero are intentionally mapped.
- Signed-overflow, pointer-wrap, and signed-left-shift semantics.
- Alignment requirements and behavior on unaligned accesses.
- Memory endianness and the physical interpretation of G3’s three-byte region.
- Aliasing among shared storage and computed addresses.
- Dependency errors, exceptions, nonlocal control flow, and memory ownership.
- Whether any complete invocation reaches the final return.

The supplied warning establishes that some shared symbols overlap smaller storage regions. It does not establish every overlap or justify treating the affected regions as independent variables.

## Evidence

**Established static evidence:** The supplied representation shows three explicit 32-bit parameters, ordered indirect calls, the described integer operations and memory accesses, a void return interface, and a final call to address zero.

**Inferred intent:** Four-scaled values and repeated field offsets suggest structured or encoded storage. That inference does not establish object validity, types, ownership, or application semantics.

**Verification boundary:** No runtime tests have been performed. No dependency implementation, executable memory map, ABI documentation, or independent instruction-level validation was supplied. The acceptance examples describe conditional behavior derived from the visible operations, not observed execution.
