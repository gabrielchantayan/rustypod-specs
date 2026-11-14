<!-- Trusted provenance: F-de0d0cde95bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de0d0cde95bd

## Purpose

F-de0d0cde95bd exposes a four-word entry interface and a visible execution prefix consisting of indirect dependency invocations, byte-selected writes, and transfers of derived 32-bit values.

The supplied static representation establishes arithmetic relationships and operation ordering. It does not establish an application-level purpose, a valid object model, or successful completion. In particular, it does not demonstrate object allocation, initialization of a known structure, or processing of a recognizable format.

This contract describes the represented prefix, conditional on execution reaching each operation. The representation warns of unreliable instruction data and ends at an invalid-instruction boundary. Neither a normal return nor a particular runtime response to that boundary is established.

## Interface

### Explicit inputs

| Input | Represented width | Established explicit use |
|---|---:|---|
| First argument | 32 bits | Supplied to F-101a. |
| Second argument | 32 bits | No explicit use in the visible prefix. |
| Third argument | Signed-looking 32 bits | Determines a byte-read address through multiplication by \(2^{26}\), followed by addition of 1. |
| Fourth argument | Signed-looking 32 bits | Shifted left by 10 and supplied twice to F-103c. |

The absence of an explicit use of the second argument is not a complete ABI-level noninterference guarantee. Unknown dependencies may observe register state not expressed in their displayed argument lists.

### Additional entry state

The represented behavior also depends on:

- An incoming link-related value used to construct the first indirect-call target.
- An incoming address-like value used for the first byte read.
- An incoming 8-bit value used for the first byte write.
- An additional unsigned 32-bit register-derived value used later for arguments, addressing, and byte extraction.

The producer and ABI lifetime of the last value are unresolved. This contract calls it the **auxiliary word**, meaning the value represented by that unresolved register state, not a newly inferred argument or dependency return.

The first byte read occurs after five dependency invocations. Its placement does not establish that those dependencies preserve the underlying incoming registers; the supplied representation, rather than a verified calling convention, is the basis for these additional inputs.

### Outputs and completion

There is no declared return value and no visible normal return.

If the entire represented prefix is reached, its explicit effects include:

- Five byte writes.
- Fourteen 32-bit writes.
- Five byte reads.
- Three 32-bit reads.
- The dependency invocations and explicit argument relationships described below.

Dependency-internal effects are additional and unknown. No success status, returned object, ownership result, or recoverable error protocol is established.

### Widths and address arithmetic

A byte is 8 bits; a word is 32 bits. One dependency result is represented as 64 bits and subsequently separated into low and high 32-bit halves.

For the arithmetic relationships below:

- Addition, multiplication, and left shift on words retain the low 32 bits.
- Logical right shifts insert zeros.
- A byte stored from a wider value retains only its low eight bits.
- A byte used as an address selector is zero-extended.
- All address offsets are byte offsets.

This is a bit-vector interpretation of the apparent machine-width relationships, not a claim that signed-looking overflow or left shifts have portable language semantics. Pointer interpretation, memory mapping, alignment requirements, and the reliability of the recovered arithmetic remain unresolved.

## Behavior

### Ordering and dependency notation

Dependency IDs identify opaque **call sites**, not proven distinct target functions. Different IDs may reach the same function.

Where a five-call group is specified, its invocations occur in suffix order, from “a” through “e.” Only the first invocation has the listed explicit arguments; the following four have no explicit arguments in the representation. Unless otherwise stated, the group’s named result is the final invocation’s 32-bit result.

“No explicit arguments” does not establish a complete zero-parameter signature or exclude implicit register inputs.

Each operation is conditional on all earlier dependencies returning and all earlier accesses completing. No conditional branch, loop, retry, or rollback is visible in the prefix.

### Initial selectors and scaled arguments

The initial five-call group, F-101a through F-101e, begins with the first entry argument. Its final result is the **initial scale word**.

After that group, one byte is read at the incoming address-like value plus 1. Its zero-extended value is the **initial selector**. The incoming 8-bit value is then written to numeric address \(17+\text{initial selector}\). The destination is not relative to the address from which the selector was read.

The next five-call group, F-102a through F-102e, begins with four times the initial scale word. Its final result supplies the next scale word.

Four further invocations follow:

| Dependency | Explicit arguments, in order |
|---|---|
| F-103a | Four times the next scale word; initial selector |
| F-103b | None |
| F-103c | Fourth entry argument shifted left by 10; the same shifted value |
| F-103d | None |

After F-103d, a fresh byte is read at \(2^{26}\times\text{third argument}+1\). Zero is written to numeric address 17 plus that byte.

These are two separate byte stores: the first transfers the incoming byte; the second clears a separately selected destination.

### Auxiliary-word byte transfer and split result

The five-call group F-104a through F-104e begins with three explicit arguments: the auxiliary word, the auxiliary word again, and sixteen times the auxiliary word. Its final result is retained as a scale word.

Next, a byte is read at \(16\times\text{auxiliary word}+1\). This **auxiliary selector** is retained. The low eight bits of the auxiliary word logically shifted right by 22 are written to address \(17+\text{auxiliary selector}\).

The five-call group F-105a through F-105e begins with four times the retained scale word. Its final result is used by the following sequence:

| Dependency | Explicit arguments, in order |
|---|---|
| F-106a | Auxiliary selector; four times the F-105e result |
| F-106b | None |
| F-106c | Zero; zero |
| F-106d | None |

The argument order at F-106a differs from F-103a: the selector is first here.

F-106d supplies the represented 64-bit result. Its halves are called the **low half** and **high half**. The first explicit word write stores zero at \(4\times\text{high half}+20\).

### Propagation of the split result

The five-call group F-107a through F-107e begins with the high half. Call its final result the **high-half destination word**.

The high half is stored at four times that destination word plus 20. The five-call group F-108a through F-108e then begins with four times the high-half destination word. No result from this group is explicitly consumed afterward, but its invocations and possible hidden effects remain part of the prefix.

Next, \(2^{26}\times\text{low half}\) is stored at \(2^{28}\times\text{low half}+20\).

The five-call group F-109a through F-109e begins with \(2^{26}\times\text{low half}\). At four times its final result plus 20, the function stores:

\[
(\text{low half}\mathbin{\&}\text{0x1fffffff}) \gg 9.
\]

This stored value ranges from zero through \(\text{0x000fffff}\).

The five-call group F-110a through F-110e begins with four times the F-109e result. Its final result is the **first transfer word**.

### First transfer and independent argument reads

Four times the first transfer word is stored at sixteen times that word plus 20.

Two separate 32-bit reads then occur from four times the first transfer word plus 20. F-111a receives:

1. One read’s unsigned value logically shifted right by 12.
2. Four times the first transfer word.
3. The other read’s unsigned value shifted left by 2.

The mutual evaluation order of these two reads is not established. Both precede the invocation. They are not specified as one captured value used twice and need not observe equal values if memory changes between them.

F-111b through F-111e follow without explicit arguments. Four times the first transfer word is stored at four times the F-111e result plus 20.

The five-call group F-112a through F-112e begins with four times the F-111e result. Its final result is the **second transfer word**.

### Subsequent transfers and captured memory

The next relationships comprise six word writes, interleaved with dependency groups:

| Stage | Ordered contract |
|---|---|
| Second transfer | Store four times the second transfer word at sixteen times that word plus 20. F-113a receives that scaled value twice, followed by the low half shifted left by 26. F-113b through F-113e follow without explicit arguments. Store the same scaled second transfer word at four times the F-113e result plus 20. |
| Third transfer | F-114a through F-114e form a five-call group beginning with four times the F-113e result. Its final result is the **third transfer word**. Store four times that word at sixteen times that word plus 20. F-115a through F-115e form a group beginning with four times the third transfer word. Store four times the third transfer word at four times the F-115e result plus 20. |
| Captured-memory transfer | F-116a through F-116e form a group beginning with four times the F-115e result. Its final result is the **capture-base word**. Store four times that word at sixteen times that word plus 20. Then read and retain one unsigned word from four times the capture-base word plus 20. F-117a through F-117e form a group beginning with four times the capture-base word. Its final result is the **field word**. Store the retained read logically shifted right by 12 at four times the field word plus 20. |

The read in the last stage follows its preceding store, but the two addresses use different scale factors and are not generally equal. The read cannot be replaced with the preceding stored value.

The retained read precedes F-117a. Later dependency writes do not alter that already-captured value.

### Field-derived arguments and final word transfer

The five-call group F-118a through F-118e begins with four times the field word. Its final result is the **final transfer word**.

Four times the final transfer word is stored at sixteen times that word plus 20.

F-119a receives:

1. \(((4\times\text{field word})\mathbin{\&}63)\ll14\).
2. Four times the final transfer word.
3. The field word shifted left by 30.

Under 32-bit truncation, the first argument depends only on the field word’s low four bits and is a multiple of \(2^{16}\). The third depends only on its low two bits.

F-119b through F-119e follow without explicit arguments. Four times the final transfer word is then stored at four times the F-119e result plus 20.

The five-call group F-120a through F-120e begins with four times the F-119e result. Its final result is the **final byte-source word**.

### Final selector reread and terminal calls

A byte is read at \(2^{28}\times\text{field word}+1\). The low eight bits of four times the final byte-source word are written to address 17 plus that selector.

F-121a receives four times the final byte-source word twice, followed by sixteen times that word. Its target construction uses displacement 4076 rather than the displacement 4094 used by the other represented indirect calls. This difference is established; its semantic significance is unknown.

F-121b has no explicit arguments. Its word result is passed directly to F-121c. F-121d through F-121g then execute without explicit arguments. The final result is retained as the **terminal scale word**.

After those intervening calls, the byte at \(2^{28}\times\text{field word}+1\) is read again. At address 17 plus this fresh selector, the function writes the low eight bits of:

\[
(\text{final byte-source word}\mathbin{\&}\text{0x3fffffff})\gg20.
\]

The two final selector reads cannot be merged: intervening dependencies may change the source, and aliasing may also matter.

The five-call group F-122a through F-122e begins with four times the terminal scale word. F-123a then receives four times the F-122e result followed by the latest selector. F-123b follows without explicit arguments.

The representation ends immediately afterward at an invalid-instruction boundary. It establishes no subsequent effect or normal return.

## Side effects

### Explicit memory footprint

The store totals follow from the described phases:

| Phase | Byte stores | 32-bit stores |
|---|---:|---:|
| Initial selectors | 2 | 0 |
| Auxiliary-word transfer and split result | 1 | 1 |
| Split-result propagation | 0 | 3 |
| First transfer | 0 | 2 |
| Subsequent transfers | 0 | 6 |
| Field-derived transfer | 0 | 2 |
| Final selectors | 2 | 0 |
| **Total** | **5** | **14** |

There are also five byte reads and three word reads. The word reads comprise the two distinct argument reads before F-111a and the captured read before F-117a. These totals exclude dependency-internal accesses.

Every selector-based byte destination lies between numeric addresses 17 and 272 inclusive. This is an arithmetic range, not evidence that those addresses are mapped or writable.

### Layout, aliasing, and preservation

Offsets 1, 17, and 20 and scale factors 4 and 16 are established. They do not prove structure fields, object sizes, array bounds, or pointer ownership.

No disjointness is established. Stores may alias other stores, selector sources, word-read sources, or dependency-visible state. Later reads must observe the actual state at their specified positions, not an assumed initial snapshot.

Unknown dependencies have no established global-preservation guarantee. Values explicitly retained in the representation describe dataflow, not proof of a particular physical-register preservation rule.

### Ownership and failure

No allocation, release, reference-count operation, ownership transfer, or lifetime extension is established. Neither address-like arguments nor returned words imply ownership.

There is no visible validation, status check, exception translation, or rollback. An inaccessible address, invalid call target, or nonreturning dependency may stop execution after earlier effects have occurred. Earlier stores are not undone by any visible mechanism.

Reaching the end requires all preceding accesses and invocations to complete. The function does not establish those preconditions for its caller.

## Dependencies

All dependency names are opaque local call-site IDs.

The ordinary five-call groups are F-101, F-102, F-104, F-105, and F-107 through F-120, each using suffixes “a” through “e.” The shorter sequences are F-103a through F-103d and F-106a through F-106d. The terminal sequences are F-121a through F-121g, F-122a through F-122e, and F-123a through F-123b.

Their argument and consumed-result relationships are specified above. No dependency body or semantic identity is available.

The evidence does not establish complete signatures, implicit inputs, hidden results, memory effects, allocation behavior, determinism, reentrancy, or termination. The represented 64-bit result may reflect reconstructed register state rather than a verified logical return type.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Selector zero-extension.** If the initial selector is 255 and the incoming byte is 166, the first byte store writes 166 to numeric address 272, not to an offset from the selector’s source address.

2. **Third-argument addressing.** With third argument 1, the clear-operation selector is read at \(\text{0x04000001}\). If that byte is 7, zero is written to address 24.

3. **Duplicate shifted arguments.** With fourth argument 3, F-103c receives 3072 twice.

4. **Split-result arithmetic.** If the high half is 3 and the low half is \(\text{0x12345678}\), the first word store writes zero at address 32. The later masked-and-shifted low-half store writes \(\text{0x00091a2b}\), if reached.

5. **Distinct argument reads.** If the read feeding F-111a’s first argument observes \(\text{0x12345000}\), that argument is \(\text{0x00012345}\). If the other read observes 3, the third argument is 12. This does not require either read to occur first.

6. **Captured memory.** If the captured word is \(\text{0xabcdef01}\), the later store at four times the field word plus 20 writes \(\text{0x000abcde}\), even if a dependency subsequently changes the read location.

7. **Field extraction.** If the field word is 15, F-119a’s first argument is \(\text{0x000f0000}\), and its third is \(\text{0xc0000000}\).

8. **Selector reread.** If the final selector reads yield 5 and then 9, their stores target addresses 22 and 26. The second destination must use the second observation.

9. **Different byte transformations.** If the final byte-source word is \(\text{0x12345678}\), its first final-phase byte value is \(\text{0xe0}\), and its second is \(\text{0x23}\).

10. **Complete-prefix footprint.** Reaching every represented explicit store entails five byte stores and fourteen word stores, regardless of address aliasing. These are operation counts, not counts of distinct modified locations.

11. **Partial execution.** If a dependency does not return, later explicit operations are not required to occur. Earlier effects have no visible rollback.

12. **No success return.** Reaching F-123b does not demonstrate normal completion; the next represented boundary remains unresolved.

## Uncertainties

**Established static evidence:** represented access widths, explicit dataflow, dependency ordering, store counts, separate reads, arithmetic relationships, and the terminal invalid-instruction boundary.

**Inferred intent:** repeated scaled transfers may represent state propagation. Selector-based byte stores may participate in another addressing mechanism. Neither inference establishes a domain, object layout, or valid execution environment.

**documented_only unknowns:**

- Reliability of instruction recovery and the represented control-flow prefix.
- Actual architecture, endianness, pointer model, alignment rules, and mapped addresses.
- Origins and ABI treatment of implicit entry state and the auxiliary word.
- Dependency identities, complete signatures, hidden effects, and preservation rules.
- Ownership, lifetime, concurrency, and memory-ordering requirements.
- Mutual evaluation order of the paired reads before F-111a.
- Whether any invocation can execute the whole prefix.
- Runtime behavior at the final invalid-instruction boundary.

## Evidence

This contract derives from the supplied static representation of F-de0d0cde95bd. The store totals are supported by the phase-by-phase inventory above, including both initial byte stores, the auxiliary byte store, both final byte stores, and all fourteen word stores.

The representation reports unreliable instruction data and terminates without a normal return. No dependency bodies, implementation access, architecture documentation, or runtime observations were available. No runtime tests have been performed; the acceptance examples express conditional comparisons only.
