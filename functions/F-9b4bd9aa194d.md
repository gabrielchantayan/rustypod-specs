<!-- Trusted provenance: F-9b4bd9aa194d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9b4bd9aa194d

## Purpose

The supplied representation describes a non-returning operation that repeatedly invokes unresolved indirect dependencies and performs byte-sized and 32-bit memory accesses at arithmetically derived addresses.

No application-level purpose is established. In particular, the evidence does not establish that these accesses implement a collection, allocator, device interface, interpreter, or serialized format. Those interpretations must not become requirements.

The observable contract is an ordered, indefinitely repeated sequence of dependency calls, memory reads, and memory writes. Dependency results supply most address components and stored values. Selected results become inputs to the following iteration.

## Interface

### Explicit inputs

The represented interface has four 32-bit input slots:

| Input | Established use |
|---|---|
| First input word | No explicit use in the supplied body. |
| Second input word | No explicit use in the supplied body. |
| Third input word | Its low four bits are shifted into bits 28–31 for a dependency argument during the first iteration. A dependency result replaces this word before later use and before the next iteration. |
| Fourth input word | Bits 20–27 supply the first indirect byte write. Near the end of each iteration, this word is replaced with a dependency result shifted left by two. |

The absence of explicit uses of the first two inputs does not prove that machine-level indirect calls cannot observe their incoming register contents.

### Implicit inputs

The first iteration also depends on:

- An incoming 32-bit arithmetic state value used to form the address of the first byte read.
- An incoming execution-context value used to form the first indirect call target.
- A readable global word.
- Memory contents at all derived read addresses.
- All dependency results and any additional return-register state consumed by subsequent calls.

These implicit inputs prevent treating the four explicit words as a complete standalone interface.

### Output and termination

No ordinary return value is established. The represented body ends in an unconditional backward repetition, with no visible normal-return path.

This does not guarantee infinite successful execution. A dependency may fail to return, or an access or call may fault. No local recovery path is shown.

### Width conventions

For the descriptions below:

- A **word** is 32 bits.
- A **byte** is eight bits.
- A two-word result has a low word and a high word.
- Byte stores retain only the low eight bits of their supplied value.
- Logical right shifts apply to unsigned words where indicated.

Address arithmetic and left shifts are described using their low-32-bit bit-vector interpretation. This is the apparent machine-level reading, not a claim that every signed expression in the supplied representation has defined portable-language behavior. Address width and overflow interpretation remain subject to the uncertainties below.

## Behavior

### Address and value notation

The following notation describes access patterns, not implementation variables:

- **Indirect byte update at base \(B\):** read one unsigned byte at \(B+1\), then write one byte at the read value plus \(0x11\).
- **Indexed word update for token \(T\):** write a word at \(4T+0x14\).
- **Wide-stride word update for token \(T\):** write a word at \(16T+0x14\).
- **Page-stride word update for token \(T\):** write a word at \((T\ll14)+0x14\).

All address calculations use the width qualification above. In particular, an indirect byte update uses a zero-extended byte as an address component. It does not load a full pointer.

Names such as “opening token” below are specification labels for captured dependency results. They are not identified external objects or source-level identifiers.

### 1. Opening calls and initial byte update

Each iteration begins with four indirect calls. The first target depends on the incoming execution context; the following targets are derived from embedded execution-location values. No explicit arguments are shown for these four calls.

The fourth call supplies a two-word result:

- Its low word becomes the **opening token**.
- Its high word is passed to the next dependency.

Before passing that high word, the function performs an indirect byte update with:

- Base: the incoming arithmetic state multiplied by \(0x1000\).
- Stored byte: bits 20–27 of the current fourth input word.

The mask shown before the right shift retains bits 0–29, but byte truncation means that bits 28–29 do not reach the store.

The next visible sequence passes the high result word to a dependency, obtains another word from a subsequent call, and forwards that word to another dependency. A further two-word result is passed onward as two separate arguments, low word first.

### 2. Opening-token byte clearing

After further dependency calls, a word result is obtained. Before passing that result shifted left by two to the next dependency, the function performs an indirect byte update storing zero at a base equal to:

\[
\text{opening token}\ll30
\]

This base follows from the composed multiplication and shift operations; it depends only on the opening token’s low two bits under the 32-bit interpretation.

A later dependency result becomes the **first page token**. The function then performs another indirect byte update storing zero, this time using base:

\[
\text{opening token}\ll4
\]

It subsequently:

1. Passes the opening token shifted left by 12 to a dependency.
2. Obtains a word from another dependency.
3. Passes that word and the current third input word shifted left by 28 to another dependency.

After another call sequence, it again performs an indirect byte update storing zero at the opening-token-shifted-left-by-four base. The byte selector is read again; the earlier selector is not reused.

The newly obtained dependency word, shifted left by two, is then passed onward.

### 3. First pair of word writes

Following additional dependency calls, the function writes:

- Address: \((\text{first page token}\ll14)+0x14\).
- Value: \(\text{first page token}\ll14\).

It next passes the first page token shifted left by 16 to a dependency. After another call, a dependency receives:

- First argument: zero under the 32-bit interpretation.
- Second argument: additional return-register state whose value is not established by the supplied representation.

The first argument is zero because it is the already-left-shifted-by-14 word shifted left by another 28 positions.

A later dependency supplies an **indexed destination token**. The function writes the same first-page value at four times this token plus \(0x14\), then calls a dependency with four times that token.

### 4. Second page token and selector update

A later call result becomes the **second page token**.

The function performs an indirect byte update at the opening-token-shifted-left-by-four base. The stored byte is bits 20–27 of the first-page value.

It then passes the first page token shifted left by 26 to a dependency. A later call supplies a word that is passed to another dependency together with the second page token shifted left by 30.

A subsequent two-word result is forwarded as low word and high word. After further calls, the opening-token-based indirect byte update is repeated with zero. The selector is freshly read.

### 5. Second pair of word writes

A later dependency supplies a **second payload token**. The function writes:

- Address: \((\text{second page token}\ll14)+0x14\).
- Value: \(\text{second payload token}\ll2\).

The next dependency receives four arguments, in order:

1. The second page token shifted left by 12.
2. That same value again.
3. The second page token shifted left by 14.
4. The stored payload value.

After further calls, an indexed destination token is obtained. The payload value is written at four times that token plus \(0x14\). A dependency is then called with four times that token.

### 6. Two wide-stride/indexed write pairs

The function performs two successive groups of the following form, using fresh dependency results for each group:

1. Obtain a **wide-stride token**.
2. Write four times that token at sixteen times that token plus \(0x14\).
3. Call a dependency with three copies of four times that token.
4. Make further dependency calls.
5. Obtain an indexed destination token.
6. Write the same saved value at four times the indexed destination token plus \(0x14\).
7. Call a dependency with four times the indexed destination token.

The two groups are distinct. Neither their tokens nor their destinations are required to match.

For the second group, retain these additional values:

- The wide-stride token shifted left by 30 becomes the **tail base**.
- The indexed destination token becomes the **tail indexed token**.

### 7. Direct byte clearing at the tail base

A later dependency supplies a **selector word**. Before passing that word shifted left by two to a dependency, the function performs two direct byte writes, in this order:

1. Zero at tail base plus \(0x11\).
2. Zero at tail base plus \(1\).

After intervening dependency calls, the same two direct zero writes occur again, in the same order. A newly returned word shifted left by two is then passed onward.

These are direct writes at tail-base-relative addresses, not indirect byte updates.

### 8. Wide-stride write, selected read, and input replacement

A later dependency supplies a new wide-stride destination token. The function writes:

- Address: sixteen times that token plus \(0x14\).
- Value: the tail indexed token shifted left by 31.

It then reads a word at:

\[
\left((\text{selector word}\mathbin{\&}0x3fffffff)\gg20\right)+0x14
\]

This read address ranges from \(0x14\) through \(0x413\) before any target-specific address interpretation. It is not scaled by four and may be unaligned.

After a dependency call using four times the latest wide-stride destination token, another dependency result replaces the current third input word. Call this replacement the **carried third word**.

The next dependency receives the selected memory word shifted right by 22 and then left by 28. Under 32-bit truncation, only bits 22–25 of the selected word survive.

### 9. Zero word and tail selector updates

After further calls, the function obtains an indexed destination token and writes a zero word at four times that token plus \(0x14\). A dependency is then called with four times that token.

A later dependency supplies a **tail selector value**. The function:

1. Reads a byte at tail base plus \(1\).
2. Writes the low byte of the tail selector value shifted left by two at the read byte plus \(0x11\).
3. Calls a dependency with the tail selector value shifted left by two.
4. Obtains another word and forwards it together with the tail selector value shifted left by 30.

After further calls, it again reads the byte at tail base plus \(1\). This second read is significant: intervening dependencies may have changed the byte.

The second read value is retained as the **saved byte selector**. The function writes bits 20–27 of the tail selector value at saved byte selector plus \(0x11\), then passes a newly obtained dependency result shifted left by two onward.

Following further calls, it writes a word at saved byte selector plus \(0x14\):

\[
\text{carried third word}\ll2
\]

The saved selector is reused for this word write even though dependencies have executed since it was read.

### 10. Global-derived byte update

Immediately after that word write, the function reads the global word.

After another sequence of calls, a two-word result supplies a **closing token** from its low word. Before forwarding its high word to a dependency, the function performs an indirect byte update using the tail base and storing the low byte of the global word shifted right by 22.

Thus the byte contains global-word bits 22–29. This differs from the masked-right-shift-by-20 extraction used elsewhere.

A later call forwards another dependency result together with the closing token shifted left by 30.

After additional calls, an indirect byte update at the tail base stores zero. A newly obtained word shifted left by two is then passed onward.

### 11. Closing writes and next-iteration state

After further calls, a dependency receives two identical arguments, each equal to a recently returned word shifted left by two.

A later result becomes the **closing payload token**. The function writes:

- Address: \((\text{closing token}\ll14)+0x14\).
- Value: \(\text{closing payload token}\ll2\).

Before the next iteration:

- Incoming arithmetic state becomes the closing token shifted left by four.
- Fourth input word becomes the closing payload token shifted left by two.
- Third input word remains the carried third word.

A dependency receives the closing token shifted left by 16. After further calls, another indexed destination token is obtained. The closing payload value is written at four times that token plus \(0x14\), and a dependency is called with four times that token.

The execution-context value used to form the next iteration’s first call target is also replaced. Execution then repeats without a visible condition.

## Side effects

The body explicitly performs byte reads, byte writes, word reads, word writes, a global-word read, and numerous indirect calls.

The following ordering requirements are material:

- Every indirect byte update reads its selector before writing the selected byte.
- Repeated updates generally re-read selectors.
- The saved byte selector in the tail sequence is deliberately reused across intervening calls.
- The two pairs of direct tail-base clears each write offset \(0x11\) before offset \(1\).
- The carried-third-word write precedes the global-word read.
- Closing state is established before repetition.

No allocation, release, reference counting, ownership transfer, locking, rollback, or explicit error reporting is established. Dependencies may have such effects, but their contracts are unknown.

Aliasing is possible. A write can overlap a later selector, a later word, or another derived destination. No snapshot isolation or independence between these addresses may be assumed.

Successful progress requires callable dependency targets and memory supporting every executed access, including potentially low and unaligned word addresses. No local validation is shown.

## Dependencies

**F-opaque** denotes the unresolved indirect dependency family in this document. It is an opaque reference category, not a claim that all call sites resolve to one function.

The supplied dependency map establishes no separate dependency identities. Assigning distinct stable function IDs, names, ownership contracts, or purposes to these targets would therefore be unsupported.

Observable dependency interactions include:

- Calls with no explicit arguments shown.
- Calls consuming individual words.
- Calls consuming two, three, or four words.
- Calls returning one word or two words.
- A call consuming additional return-register state not otherwise defined in the representation.
- Call targets derived partly from execution-context values and partly from embedded execution-location values.

Calls between memory accesses remain observable even where their results are not used explicitly. They must not be eliminated on an assumption of purity. Nor may they be assumed to preserve global state, memory, ownership, or unspecified register state.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial byte extraction.** If the fourth input word is \(0x1AB00000\), the first indirect byte update stores \(0xAB\), provided execution reaches it successfully.

2. **Byte selector interpretation.** If an indirect byte update reads selector byte \(0x7F\), its destination is address \(0x90\), not an address obtained by loading a full pointer.

3. **Fresh selector reads.** If a dependency changes the opening-token selector byte between two zero updates, the later update uses the changed byte.

4. **Saved selector reuse.** If the saved byte selector is \(0x20\), the carried-third-word store targets \(0x34\), even if a dependency subsequently changes the byte at tail base plus \(1\).

5. **Truncated call argument.** The call described after the first page-stride write receives zero as its first argument under 32-bit arithmetic, regardless of the first page token.

6. **Selected read boundary.** A selector word of zero selects word address \(0x14\). A selector word of \(0x3FF00000\) selects \(0x413\). The latter requires whatever unaligned-access behavior the execution environment supplies.

7. **Direct clear ordering.** If both direct tail-base clear pairs are reached, observers see writes at offsets \(0x11,1,0x11,1\), with dependency calls between the pairs.

8. **Cross-iteration carry.** If the closing payload token is \(3\), the next iteration’s fourth input word is \(12\). Its initial extracted byte is consequently zero.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention, instruction-set mode, address width, and signed-overflow semantics.
- Validity and identity of each indirect call target.
- Whether apparent no-argument calls consume implicit register arguments.
- The origin and meaning of the additional return-register argument.
- The first iteration’s implicit arithmetic and execution-context inputs.
- Global-word identity, lifetime, volatility, and mutability.
- Memory mapping, byte order, alignment requirements, and fault behavior.
- Whether dependencies preserve any memory, globals, or register state.
- Ownership and allocation relationships.
- Whether the represented unconditional repetition accurately captures the complete executable control flow.

The apparent repeated layout using offsets \(1\), \(0x11\), and \(0x14\) suggests structured state, but no object type or field semantics are established.

## Evidence

Established static evidence consists solely of the supplied representation: four explicit input slots, unresolved indirect calls, explicit byte and word accesses, arithmetic address formation, captured dependency results, one global-word read, and unconditional repetition.

The offset descriptions and bit extractions above are derived from those explicit operations. Modular arithmetic conclusions are conditional on the stated 32-bit interpretation.

Application intent, dependency contracts, executable validity, and runtime behavior are not independently established. No runtime tests have been performed.
