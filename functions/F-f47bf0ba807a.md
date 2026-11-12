<!-- Trusted provenance: F-f47bf0ba807a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f47bf0ba807a

## Purpose

F-f47bf0ba807a performs a fixed sequence of memory reads and writes, followed by an unresolved indirect control transfer. Its visible effects include repeated initialization stores, byte truncation, replacement of earlier field values, and accesses through addresses obtained from memory.

The function does not establish an application-level object model. Incoming values and subsequently captured words are used variously as addresses, numeric values, and sources of low-order bytes.

**Established static evidence:** the supplied representation establishes access widths, address expressions, captured values, and a represented ordering of operations.

**Inferred intent:** some address bases may identify related records or runtime state. This interpretation is not necessary to describe the behavior and does not establish ownership, allocation boundaries, or disjoint objects.

**Unknown:** the larger purpose, complete incoming-state convention, and final indirect target.

No runtime tests have been performed.

## Interface

### Incoming state and result

The explicit argument is **P**, a signed 32-bit value also used as an address base.

Three additional incoming values are required:

| Contract designation | Width and interpretation | Uses |
|---|---|---|
| **A** | Unsigned 32-bit | Address base, doubled address component, stored word, byte source, shift-count source |
| **B** | Signed 32-bit | Address base and byte source |
| **C** | Signed 32-bit | Address base and stored word |

These are contract designations, not implementation identifiers. The mechanism supplying A, B, and C is unresolved; P alone does not determine the function’s behavior.

No result value is explicitly produced. If the final indirect transfer returns normally, the enclosing function returns without a defined result value.

### Widths, addresses, and captured values

All offsets below are byte offsets.

- A **byte** contains 8 bits.
- A **halfword** contains 16 bits.
- A **word** contains 32 bits.
- **low8(X)** means the least significant eight bits of X.

The following labels denote captured values:

| Label | Read location | Interpretation used subsequently |
|---|---|---|
| G | 0xA9 | Opaque word and byte source |
| H | P + 0x74 | Opaque word and byte source |
| J | C + 0x14 | Opaque word |
| K | A + 0x04 | Opaque word |
| L | C + 0x44 | Signed 32-bit value |
| M | A + 0x40 | Opaque word |
| N | A + 0x54 | Signed 32-bit address-bearing value |
| Q | A + 0x14 | Opaque word and byte source |
| R | C + 0x04 | Signed 32-bit address-bearing value |
| S | R + 0x64 | Signed 32-bit address-bearing value |
| T | R + 0x74 | Opaque word |
| U | C + 0x64 | Signed 32-bit address-bearing value |

These captures occur at the points specified in Behavior, not together at entry. Later writes can affect a capture that has not yet occurred. Once captured, a value is retained independently of subsequent changes to its source location.

The directly accessed fixed addresses are:

| Address | Access |
|---|---|
| 0x21 | Byte write |
| 0x3A | Byte write |
| 0x67 | Byte write and three subsequent byte reads |
| 0xA9 | Word read |

Byte order is not established. A word access covers four consecutive bytes, and a halfword access covers two; overlap effects depend on target byte order.

### Arithmetic and preconditions

Address expressions involving unsigned A use 32-bit unsigned arithmetic. In particular, twice A and twice A plus 0x54 are reduced modulo \(2^{32}\).

Do not extend this wraparound rule to signed address calculations. Overflow behavior for calculations involving P, B, C, N, R, S, or U is **documented_only**. This includes doubling R and S.

Normal completion requires:

- Every reached memory access to be valid for its width and access direction.
- Address calculations to have supported semantics in the execution environment.
- Any required alignment to be satisfied.
- The final indirect target to be valid for the represented transfer and arguments.

The body contains no explicit null checks, bounds checks, alignment checks, or recovery paths.

## Behavior

### Repeated initialization prefix

The function begins with a fixed sequence of identical four-store rounds:

| Order within each round | Destination | Width | Stored value |
|---|---|---|---|
| First | Twice A + 0x54 | Word | P |
| Second | B + 0x19 | Byte | low8(A) |
| Third | A + 0x11 | Byte | low8(B) |
| Fourth | B + 0x34 | Word | Twice A, modulo \(2^{32}\) |

The destinations do not advance between rounds. No data-memory read or dependency invocation is represented within this prefix.

The represented prefix contains repeated stores, not merely a requirement for four final values. If destinations overlap, the listed order governs the represented overwrite sequence within each round and across successive rounds.

The numerical repetition count is not specified by this contract and remains **documented_only**. This omission does not authorize replacing the supplied prefix with one round under an execution model that observes individual stores.

### Initial captures and byte-field updates

After the prefix, G is captured from 0xA9, followed by H from P + 0x74.

The byte at 0x3A then receives:

\[
\bigl(2 \times \operatorname{low8}(A) + 0x30\bigr) \bmod 256.
\]

This describes the stored bit pattern without requiring a particular signed interpretation of an intermediate character-sized value. No printable-character or decimal-digit restriction is established.

J is captured from C + 0x14, followed by K from A + 0x04. Both captures precede the following writes:

| Order | Destination | Width | Value |
|---|---|---|---|
| 1 | C + 0x0D | Byte | low8(A) |
| 2 | A + 0x08 | Byte | 0x6C |
| 3 | A + 0x0D | Byte | low8(B) |
| 4 | A + 0x09 | Byte | low8(B) |
| 5 | C + 0x54 | Word | C |
| 6 | 0x67 | Byte | low8(G) |

These operations do not form an established transaction or atomic group.

### First fixed-byte read and shift-dependent update

L is next captured as a signed 32-bit value from C + 0x44. The function then explicitly reads the byte at 0x67 and writes the read byte to 0x21.

Between initializing 0x67 with low8(G) and this first subsequent read, the visible body performs only the read that captures L. It performs no intervening write that could alias 0x67. Thus, for ordinary stable memory without external modification or read-triggered side effects, this read obtains low8(G).

The read nevertheless remains an explicit operation. Environmental changes or memory-mapped read effects are not excluded by the supplied representation.

The next represented effects are:

| Order | Destination | Width | Value |
|---|---|---|---|
| 1 | A + 0x10 | Byte | low8(L) |
| 2 | B + 0x54 | Word | L |
| 3 | C + 0x01 | Byte | Low eight bits of the shifted result described below |
| 4 | B + 0x54 | Word | A |

For the third operation, the **full signed 32-bit L is shifted right by low8(A), and only then is the result truncated to eight bits**. Truncating L before shifting is not equivalent.

Counts 0–31 are within the represented operand width. Counts 32–255 have no deterministic result specified here. For negative L, the signed right-shift rule is also unresolved; logical shifting must not be assumed.

### Replacement of earlier stores

M is captured from A + 0x40 after the word A has been written to B + 0x54.

The next writes, in order, are:

- Word K to B + 0x74.
- Word M to A + 0x10.
- Word C to B + 0x44.

The write of M covers offsets A + 0x10 through A + 0x13. It therefore replaces the earlier byte at A + 0x10 and the prefix’s byte at A + 0x11. The individual bytes of this replacement depend on target byte order.

The function then reads 0x67 again and writes the read byte to A + 0x09. This is followed by a byte write of low8(A) to C + 0x09 and a word write of G to A + 0x44.

Unlike the first read of 0x67, this read follows intervening writes that can alias 0x67. Its value cannot unconditionally be replaced with low8(G).

### Address extraction and later captures

N is captured from A + 0x54 only after the preceding writes.

A third explicit read of 0x67 supplies another byte write to A + 0x09. The intervening writes since the previous read may again affect the value obtained.

The function then writes:

- low8(H) to N + 0x0D.
- Word J to C + 0x54.

Writing J does not establish restoration of C + 0x54 to its entry value: J was captured from C + 0x14.

Q is next captured from A + 0x14. After that capture, the represented order is:

| Destination | Width | Value |
|---|---|---|
| C + 0x11 | Byte | low8(H) |
| A + 0x09 | Byte | low8(N) |
| A + 0x44 | Word | N |

These replace earlier values at A + 0x09 and A + 0x44. Equality of successive stored values, when it occurs, does not by itself eliminate the distinction between the represented operations.

### Final address-dependent updates

R is captured from C + 0x04.

Before any further field is read through R, the function writes halfword 0x0065 to the address twice R. This is a 16-bit write, not an 8-bit or 32-bit write.

S is then captured from R + 0x64. The preceding halfword write can affect this capture if their byte ranges overlap.

Next, low8(Q) is written to N + 0x11, followed by word 100 to N + 0x54.

Only after those two writes is T captured from R + 0x74. U is subsequently captured from C + 0x64.

The final explicit writes before the indirect transfer are, in order:

| Destination | Width | Value |
|---|---|---|
| C + 0x09 | Byte | low8(R) |
| U + 0x54 | Word | S |
| Twice S | Word | U |

The signed calculations used for twice R, twice S, and signed address offsets retain the arithmetic uncertainties stated in Interface.

### Indirect transfer

The final unresolved dependency, designated **F-indirect-unknown**, receives these arguments in order:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | S |
| Third | J |
| Fourth | T |

S, J, and T are captured values, not fresh reads.

The supplied representation shows an indirect invocation followed by return. The actual target identity, transfer convention, and return behavior are unresolved. If control returns normally to the visible continuation, no further explicit memory operation is represented.

## Side effects

### Memory and aliasing

The body directly modifies addressable memory without establishing object ownership or allocation boundaries.

No disjointness is established among:

- Regions based on incoming values.
- Regions reached through captured words.
- Fixed-address storage.
- Byte, halfword, and word accesses.

Consequently, chronological reads and writes control the represented behavior. A later read may obtain bytes written earlier through a different address expression. Treating every base as a separate record would introduce an unsupported assumption.

### Observability

The accesses are not represented as volatile. Their listed order is the order in the supplied representation, not a guarantee that concurrent or hardware observers see every access or see them in that order.

Preservation of individual intermediate stores is required only under an execution model in which those stores are observable. Compiler visibility rules, concurrency semantics, memory-mapped behavior, and access atomicity remain **documented_only**.

In particular, the two stores to B + 0x54 establish an intermediate represented value L followed by A; they do not independently establish that a runtime observer can detect both.

### Ownership and failure behavior

No allocation, release, ownership transfer, synchronization, logging, or explicit error-reporting operation is visible in the body. This observation does not constrain the indirect dependency.

Invalid memory accesses or an invalid indirect transfer have no visible recovery path. Earlier writes are not explicitly rolled back. Exact fault behavior is environment-dependent; no error code, sentinel result, or all-or-nothing guarantee is established.

## Dependencies

| Opaque function ID | Relationship | Established interface |
|---|---|---|
| F-indirect-unknown | Final unresolved indirect dependency | Receives 0x72, S, J, and T, in that order |

F-indirect-unknown is a contract-local opaque designation, not an identified implementation.

Its memory effects, global-state changes, allocation or release behavior, ownership rules, and return behavior are unknown. The argument list does not establish that it preserves any previously written location or owns any address-bearing argument.

Other environmental dependencies include memory mappings, access permissions, alignment rules, byte order, integer semantics, and the convention supplying A, B, and C.

## Acceptance examples

These examples are conditional deductions from static evidence, not executed tests.

### Ordinary, non-overlapping storage

Assume all reached accesses are valid, arithmetic is defined, relevant byte ranges do not overlap except where the contract explicitly revisits a destination, and no environmental actor changes memory.

Immediately before the indirect invocation:

- A + 0x08 contains byte 0x6C.
- B + 0x54 contains word A, replacing L.
- B + 0x74 contains word K.
- C + 0x54 contains word J.
- A + 0x44 contains word N.
- N + 0x54 contains word 100.
- U + 0x54 contains word S.
- Twice S contains word U.

These are pre-invocation conditions. They remain post-return conditions only if the dependency returns without changing them.

### Shift before truncation

Let L be 0x00000180 and low8(A) be 1. Assuming no subsequent aliasing write changes C + 0x01, that location receives 0xC0:

- Shifting the full value yields 0x000000C0.
- Truncating that result yields 0xC0.

Truncating L first would produce 0x80 and then 0x40 after shifting; that result does not satisfy the represented operation.

If low8(A) is 32, no deterministic shift result is supplied. Masking the count to zero is not an established rule.

### First versus later reads of 0x67

With ordinary stable memory, the first read of 0x67 after initialization obtains low8(G), because the only intervening visible operation is the read from C + 0x44.

For later reads, if an intervening write aliases 0x67 and changes it, the newly read byte is used. Reusing low8(G) regardless of such writes would not preserve the represented behavior.

### Byte truncation

If low8(A) is 0x90, the byte written to 0x3A is 0x50.

If low8(B) is 0xE7, the prefix writes bit pattern 0xE7 to its corresponding byte destinations regardless of B’s signed interpretation.

### Intermediate overwritten value

For non-overlapping storage, the represented sequence writes L to B + 0x54 and later writes A there.

Under a model that exposes individual stores, preserving that intermediate write and its order is required. Without such a model, this contract does not require a runtime observer to see both writes or their order.

### Dependency mutation

If F-indirect-unknown changes C + 0x54, no restoration to J is required afterward. No post-invocation restoration is visible.

## Uncertainties

The following remain **documented_only**:

- How A, B, and C are supplied.
- The application meaning and valid ranges of address-bearing values.
- The numerical repetition count of the initial prefix in this contract.
- Target byte order and alignment requirements.
- Signed overflow and signed address-calculation behavior.
- Signed right shifts of negative L and shift counts outside 0–31.
- Fixed-address validity and memory-mapped access effects.
- Compiler visibility, concurrency, and atomicity guarantees.
- The final dependency’s identity, actual calling convention, effects, and return behavior.
- Any ownership or lifetime obligations imposed by that dependency.
- Any larger application purpose or object layout.

These uncertainties are not resolved by assuming conventional records, preserved globals, harmless repeated stores, or dependency ownership.

## Evidence

This contract is based on static inspection of the supplied representation of F-f47bf0ba807a. Prior descriptions and reviewer comments are not independent evidence of behavior.

The representation supports the access widths, capture locations, address expressions, truncations, represented operation order, and final argument sequence described above. In particular, it supports shifting the full signed L before byte truncation and shows no intervening body write between initialization of 0x67 and its first subsequent read.

It does not establish runtime observer visibility, dependency behavior, or environmental semantics. No runtime execution, memory trace, dependency inspection, or runtime test has been performed.
