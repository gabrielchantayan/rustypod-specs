<!-- Trusted provenance: F-ba557db7e07d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ba557db7e07d

## Purpose

The established behavior is a straight-line sequence of indirect calls, computed-address memory accesses, byte-field updates, word-field updates, two writes to a shared four-byte location, and a final indirect call to address zero.

No application-level purpose is established. Repeated scaling by four, accesses at offset 20, and byte updates through selector tables suggest manipulation of indexed records or encoded references. That interpretation is **inferred intent**, not a required data model.

This contract describes the supplied static representation. It does not establish that the representation faithfully recovers the original calling convention, executable instruction boundaries, or runtime address space.

## Interface

**Explicit inputs**

- Two opaque 32-bit values. Neither is explicitly consumed by an expression in the visible body. This does not establish that indirect callees cannot observe their incoming register contents.
- One unsigned 32-bit value, called the **input word** below.

**Additional entry state**

The body also depends on state outside the explicit parameter list:

- An incoming link value used to compute the first call target.
- An incoming 32-bit register value, later stored without an explicit assignment in this body.
- Several secondary results attributed to indirect calls but not represented as ordinary function return values.
- Mutable memory, including absolute low-address locations and two shared locations described below.

The register assignments, preservation rules, and provenance of secondary results are **documented_only**.

**Output**

The declared interface has no return value. A return is visible only after the final call to address zero. Normal completion therefore requires that call, and every earlier call and memory access, to return or complete successfully.

**Memory conventions used in this contract**

- A **byte read** or **byte write** transfers 8 bits.
- A **halfword write** transfers 16 bits.
- A **word read** or **word write** transfers 32 bits.
- A stored low byte is the least significant 8 bits of the source value.
- A split 64-bit result supplies a low 32-bit word and a high 32-bit word by numerical significance. This does not specify their in-memory byte order.
- Address arithmetic is described as it appears in the static representation. Its machine-level overflow semantics are not independently established.

Two shared locations are distinguished:

- **Shared slot A:** a location written as a 32-bit value twice. The representation warns that named global objects can overlap smaller objects at the same address.
- **Shared slot B:** a location read as a pointer-like 32-bit value.

These labels describe access sites, not independent storage ownership.

## Behavior

### Execution and ordering rules

The visible body contains no conditional branches, loops, validation paths, or explicit early returns. The operations below occur in order if execution reaches them.

Indirect calls separate many of the direct accesses. Those calls are observable dependencies, not removable padding. They may change memory subsequently read by this function. No dependency is assumed to preserve globals, selector bytes, pointed-to fields, or externally visible state.

For dependency references, this contract uses locally assigned opaque IDs:

- **F-000000000001:** the entry-relative indirect call.
- **F-000000000002:** the intervening indirect-call family whose identity and complete signature remain unresolved.
- **F-000000000003:** the late indirect call with a distinct target displacement.
- **F-000000000004:** the final call through address zero.

Repeated references to F-000000000002 do **not** establish that the underlying targets are identical.

### 1. Entry calls and initial word updates

The function first invokes F-000000000001 at the incoming link value plus 4094. Its apparent 32-bit result is passed to the next indirect call.

The input word is shifted right logically by 23 bits, producing a nine-bit value in the range 0–511. Call this the **input selector**. It remains available across subsequent calls in the represented local dataflow.

After intervening indirect calls:

1. A returned 32-bit index determines a word destination at four times that index plus 20.
2. The incoming, otherwise unassigned 32-bit register value is written there.
3. The four-times-index value is passed to the next call.
4. Further calls produce or expose a secondary 32-bit value.
5. Zero is written as a word at four times that secondary value plus 20.
6. That secondary value is passed to a subsequent call.

No range checks precede either destination calculation.

### 2. Input-selector storage and first selector-table update

Another result is passed from one indirect call to the next. Following additional calls:

1. A returned index selects a word destination at four times the index plus 20.
2. The input selector is written to that destination as a complete 32-bit word.
3. A word is then read at this separately computed address: mask the currently represented secondary unsigned result with `0x1fffffff`, shift it right by 19, and add 20.
4. The scaled destination from step 1 is passed to the next call.
5. More indirect calls occur before the word read in step 3 is used.

The masked-and-shifted address component ranges from 0 to 1023. It is not multiplied by four and need not be word-aligned.

The saved word is shifted right logically by 22, yielding a value in the range 0–1023. Call this the **first extracted value**.

The function then:

- Reads a selector byte at the input selector plus 1.
- Adds 17 to that byte’s unsigned value.
- Writes the low byte of the first extracted value to the resulting address.

The full ten-bit first extracted value is retained for later addressing; only this store truncates it to eight bits.

A subsequent call receives the same unresolved secondary 32-bit value three times and an explicit fourth argument of zero.

### 3. Three zero-byte updates

After another call, an indirect call receives an explicit zero, represented as zero shifted right by 23. Additional calls follow.

The next direct store is:

- Read a byte at four times the first extracted value plus 1.
- Write zero to the address given by that byte plus 17.

The selector-read address can range from 1 through 4093. The byte-write destination is always between 17 and 272, inclusive.

A recently returned index is scaled by four and passed to a call. After further calls, another scaled result is supplied twice to one call. A subsequent unsigned 32-bit result is passed onward and retained.

After another call:

- The byte at absolute address 1 is read.
- Zero is written at that byte’s value plus 17.

A secondary result is then saved for later shifts. Call it the **saved shift operand**. A following call receives this value three times.

After another result-producing call, its apparent result and the retained unsigned result shifted right by 23 are passed together to a dependency. More calls follow.

Then:

- The byte at absolute address 1 is read again.
- Its value is saved as the **saved small base**.
- Zero is written at the saved small base plus 17.

The two reads of address 1 are distinct. They must not be merged: intervening calls may change the byte.

### 4. First shared-slot write and byte refreshes

A returned index is scaled by four and passed to a dependency. Further calls obtain another index; its four-times value is passed twice to a later call.

The next unsigned result is:

1. Written as a full word to shared slot A.
2. Passed as an argument to the following call.

After another call, the function reads a word at the saved small base plus 116 and writes its low byte at the saved small base plus 1.

It then calls a dependency with the saved shift operand shifted left by 11.

A subsequent dependency supplies a represented 64-bit result. The next call receives:

- Its low 32-bit word.
- Its high 32-bit word.
- The value previously written to shared slot A, shifted right logically by 23.

Following more calls, a returned index is scaled by four. Call the resulting address-like value the **refresh base**.

The function:

1. Reads a word at refresh base plus 116.
2. Writes its low byte at refresh base plus 1.
3. Passes refresh base to a dependency.
4. Executes additional calls.
5. Obtains another result, scales it by four, and saves that scaled value as the **retained scaled value**.
6. Reads the word at refresh base plus 116 again.
7. Writes its low byte at refresh base plus 1 again.

The second word read is fresh. The two byte writes are not a guaranteed redundant operation because calls intervene.

A subsequent four-argument call receives, in order:

- The saved shift operand shifted left by 12.
- The retained scaled value.
- The retained scaled value again.
- The unscaled result underlying the retained scaled value, shifted left by 18.

### 5. Captured-word propagation and link-like writes

Several more calls forward apparent 32-bit results. A later returned index is scaled by four to form a new base.

The function reads a word at that base plus 116, saves it as the **captured word**, and writes its low byte at that base plus 1. It then passes the base to a dependency.

After further calls:

1. The captured word is written at sixteen times a returned index plus 20.
2. A call receives the retained scaled value shifted right logically by two and then shifted left by 16.
3. Additional result-forwarding calls occur.
4. The captured word is written again, now at four times a returned index plus 20.
5. That second scaled index is passed onward.

The two captured-word writes use different scaling factors: sixteen and four. They are not interchangeable.

Following more calls, another result is shifted left by two. That scaled result is written as a word at the retained scaled value, with its bottom two bits cleared, plus 20. The scaled result is then passed twice to a dependency.

Further calls lead to a zero word written at four times a newly returned index plus 20. The scaled index is passed onward.

A subsequent represented 64-bit result supplies:

- A low word, shifted left by two to produce the **link value**.
- A high word used as an index.

The link value is written at four times the high word plus 20.

The function then reads shared slot B and retains its value as the **saved external base**. A following call receives the link value, the high word, and the link value twice more, in that order.

After additional calls, the link value is written again at four times another returned index plus 20.

A word is then read at saved external base plus 20. Call it the **external word**. The most recent scaled index is passed to the next call.

The shared-slot-B read precedes the intervening calls; the external-word read follows them. The saved address is reused, but the pointed-to word is read only at that later point.

### 6. Final selector updates

Following additional calls, a represented 64-bit result provides a **final low word** and a **final high word**.

The function performs a selector update using the captured word:

1. Shift the captured word right logically by 23.
2. Read a byte at that result plus 1.
3. Write the low byte of the external word at the selector byte’s value plus 17.

A subsequent call receives the final high word three times and the final low word shifted left by two as its fourth argument.

After further calls, a returned result is obtained, but the next selector update instead uses the saved final low word:

1. Mask the final low word with `0x3fffffff`.
2. Shift right logically by 21.
3. Read a selector byte at that result plus 1.
4. Write the low byte of the external word shifted right logically by 22 at the selector byte’s value plus 17.

The selector-read address in this update ranges from 1 through 512.

The recently returned result is shifted left by two and passed onward. More calls obtain another result whose four-times value is passed twice to a dependency.

### 7. Terminal operations

After another call, F-000000000003 produces an apparent 32-bit result, which is forwarded to a dependency. The next call supplies an apparent byte result and an unresolved secondary pointer.

The terminal direct effects occur in this order:

1. Write zero to the byte addressed by the secondary pointer.
2. Read the three-byte quantity represented at absolute address 9.
3. Combine that quantity as the upper 24 bits with the apparent byte result as the lower eight bits.
4. Write the combined 32-bit value to shared slot A.
5. Write a zero halfword at the address formed by masking the final low word with `0x3fffffff` and shifting right logically by seven.
6. Invoke F-000000000004 at address zero with three explicit zero arguments.
7. Return only if that call returns.

The final halfword destination ranges from zero through `0x007fffff`. It is not guaranteed to be halfword-aligned.

## Side effects

Established direct effects include:

- Word writes to computed destinations with offsets of 20.
- Byte writes through byte-valued selector tables.
- Byte refreshes from the low byte of word reads at offset 116.
- Two full-word writes to shared slot A.
- One read of shared slot B.
- A final byte clear through an unresolved pointer.
- A final halfword clear.
- All effects of the indirect dependencies.

The apparent indexed records do not establish allocation boundaries, ownership, or valid object sizes. No explicit allocation, deallocation, locking, rollback, or ownership transfer is visible. Dependencies may perform any of these; their behavior is unknown.

Aliasing is unrestricted by the supplied evidence. Direct stores may change locations later used as selectors, fields, shared state, or call-dependent data. Later reads must therefore observe memory at their actual execution points.

No explicit error status is produced. Invalid addresses, inaccessible call targets, alignment faults, nonreturning callees, or dependency failures can prevent later effects. Earlier writes are not visibly undone.

## Dependencies

| Opaque ID | Established role | Unknown contract |
|---|---|---|
| F-000000000001 | First call; target derived from incoming link state plus 4094; apparent result forwarded | Entry-state validity, signature, side effects, termination |
| F-000000000002 | Ordered intervening call family; apparent word, byte, paired-word, and secondary results; explicit argument transfers described above | Individual identities, complete signatures, register preservation, memory effects, ownership |
| F-000000000003 | Late result-producing call before terminal byte clear and shared-slot update | Signature, side effects, secondary-state relationship |
| F-000000000004 | Call through address zero with three explicit zero arguments | Whether address zero is callable, behavior, returnability |

The supplied dependency map resolves only F-ba557db7e07d itself. The IDs above are local opaque references, not externally verified dependency identities.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

- **Input extraction:** With input word `0xffffffff`, the input selector is 511, its full-word store contains 511, and the first input-selected byte read occurs at address 512, provided preceding calls return normally.

- **Extraction versus truncation:** If the first extracted value is `0x3ff`, its byte store writes `0xff`, but its later selector read uses address 4093. Truncating it before address calculation would violate the visible behavior.

- **Byte-selected destination:** A selector byte of `0xff` selects byte destination 272, not a signed negative offset.

- **Fresh repeated read:** If a dependency changes the word at refresh base plus 116 between the two refreshes, the second byte write uses the new word’s low byte.

- **Distinct propagation strides:** For an index of 3, the sixteen-times-index captured-word destination is 68; the four-times-index destination is 32.

- **Saved address, later contents:** If a dependency changes shared slot B after its read, the external-word read still uses the saved external base. If it instead changes memory at that base plus 20, the later read observes the changed contents.

- **Terminal truncation:** With external word `0xffffffff`, both final selector-based byte writes store `0xff`; the second derives it from the low byte of `0x3ff`.

- **Terminal low address:** With final low word zero, the final halfword clear targets address zero. No visible guard skips that write or the subsequent call through address zero.

## Uncertainties

**Established static evidence:** explicit access widths, constants, visible dataflow, argument ordering, and the straight-line ordering described above.

**Inferred intent:** indexed records, selector tables, field refreshes, and link-like relationships. None establishes a semantic object type or valid-memory guarantee.

**Documented_only unknowns:**

- Accuracy of recovered function boundaries and indirect-call signatures.
- Provenance and lifetime of secondary register results.
- Whether nominally signed index multiplication and left shifts reflect wrapping machine operations. The representation alone does not justify portable signed-overflow semantics.
- Target byte order and alignment behavior.
- Validity of low-address memory and executable address zero.
- Exact shared-location addresses, sizes, and overlap.
- Dependency effects on memory, registers, ownership, and termination.
- Whether the first two explicit parameters are observable by dependencies.
- Any application-level success or failure criterion beyond completing the represented sequence.

## Evidence

This contract is based solely on the supplied static function representation and its global-overlap warning. No implementation access, dependency bodies, runtime observations, or runtime tests were available. Acceptance examples are arithmetic consequences of the represented operations under their stated preconditions, not evidence of successful execution.
