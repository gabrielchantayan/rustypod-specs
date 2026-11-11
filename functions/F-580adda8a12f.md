<!-- Trusted provenance: F-580adda8a12f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-580adda8a12f

## Purpose

The established behavior is an ordered sequence of dependency calls, memory updates through a zero-extended 16-bit address, arithmetic derived from an unsigned byte, and construction of a 20-byte record at a dependency-supplied address.

No application-level purpose is established. The evidence does not establish allocation, string handling, rendering, ownership transfer, or a particular object type.

Under the represented widths and ordinary returning-call semantics, every possible selector byte leads to the first explicit return. A subsequent region requires a negative count, whereas the preceding calculation produces only values from 99 through 609. That region is described as **documented_only**, not as a reachable fallback or error path.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit use in the represented body. |
| Second input | 32 bits | No explicit use in the represented body. |
| Third input | 32 bits | Address arithmetic, a complete word store, and truncation for a halfword store. Represented as signed. |

The lack of explicit uses for the first two inputs does not establish that dependencies cannot observe incoming argument registers. The complete calling convention is unknown.

### Additional state

The represented computation also uses:

- Incoming link-related state selecting the first indirect dependency.
- Two incoming 32-bit register-derived values, called the **first saved value** and **second saved value**.
- A global word captured on the established path.
- Two further global words in the documented_only region.
- Ordinary dependency results, paired results, and auxiliary register-derived values.

These names describe dataflow roles, not application types or preservation guarantees.

Two auxiliary values in the documented_only region have particularly unresolved provenance:

- An **unresolved byte-load component**, read after F-U09 and used in an unsigned-byte address.
- An **unresolved stored word**, read after F-U16 and written to squared-base offset 20.

The representation does not establish that these values were produced by F-U08 or F-U15, respectively, or retained across the intervening calls.

### Outputs and preconditions

There is no explicit return value. Observable outputs are dependency interactions and memory mutations.

A dependency supplies a 64-bit pair whose upper word becomes the destination address for a 20-byte record. The routine does not explicitly return that address.

Normal completion requires valid call targets, compatible calling and result conventions, and accessible storage for every executed access. No explicit validation establishes these conditions.

### Widths and arithmetic

Bytes, halfwords, words, and paired values are 8, 16, 32, and 64 bits. Addresses are represented as 32-bit quantities.

Unsigned byte and halfword loads are zero-extended. Narrow stores retain the low 8 or 16 bits. Paired results are described numerically by their low and upper 32-bit words; their memory layout depends on byte order.

Some represented arithmetic is signed, including additions, multiplication, negation, and right shifts. The evidence does not independently establish signed-overflow behavior or the handling of negative signed shifts. Formulas below describe the represented operations, not an unconditional promise of unbounded arithmetic or wrapping behavior.

The selector-derived count is exempt from these uncertainties: all its intermediate and final values are small and representable.

Byte order, alignment requirements, access atomicity, and fault behavior are unknown.

## Behavior

### Descriptor and narrow-address updates

The routine invokes **F-U01**, selected using incoming link-related state. No explicit arguments are represented.

Its 32-bit result is treated as a descriptor address. An unsigned halfword is read at descriptor offset 12 and zero-extended to form the **narrow base**, between 0 and 65,535.

The following stores occur in order:

1. Store the low byte of the first saved value at third input plus narrow base.
2. Store the complete third input at narrow-base offset 0.
3. Store the first saved value at narrow-base offset 4.
4. Store the second saved value at narrow-base offset 8.
5. Store the low halfword of the third input at descriptor offset 20.

The narrow base is the halfword’s numerical value. It is not a full-width descriptor-relative pointer, and no upper address bits are taken from the descriptor address.

The locations may overlap. Later stores replace overlapping bytes written earlier; no overlap check is represented.

### Global capture and selector acquisition

The routine calls **F-U02** with no explicit arguments represented. An auxiliary 32-bit value read after this call is treated as unsigned and logically shifted right by one bit.

After F-U02 and before F-U03, the routine reads a global word and retains it as the **captured base**.

It calls **F-U03** with the shifted auxiliary value. The argument lies between 0 and 2,147,483,647.

F-U03 supplies a 64-bit pair. Its low word is retained for arithmetic; its upper word supplies an address. An unsigned byte at offset 13 from that address is loaded as the **selector**.

Later uses of the captured base refer to the value read before F-U03. They do not imply another global read or require F-U03 to preserve that global.

### Marker store and arithmetic call

The routine derives:

- **Predecessor address:** captured base minus 1.
- **Companion address:** captured base plus 108.
- **Intermediate count:** twice the selector minus 1.

It stores the 32-bit value 51 at captured base plus selector minus 2. This is a four-byte store, not a byte store. Its starting address may precede the captured base and may be unaligned.

It then calls **F-U04** with twice the result of subtracting 54 from F-U03’s low word. The represented arithmetic operates on 32-bit values; overflow-sensitive cases remain subject to the arithmetic qualification above.

After F-U04 returns, the count is increased by 100:

**Count = 2 × selector + 99.**

The selector ranges from 0 through 255, so the count ranges from 99 through 609. The subsequent signed nonnegative test therefore succeeds for every selector.

### Record construction and return

The routine calls **F-c260788e6b26** with the explicit argument 115. The dependency supplies a 64-bit pair whose upper word becomes the **record address**.

Writes occur in this order:

| Record offset | Width | Value |
|---:|---:|---|
| 0 | 64 bits | The complete returned pair |
| 8 | 32 bits | Count |
| 12 | 32 bits | Companion address |
| 16 | 32 bits | Predecessor address |

The destination spans 20 bytes. Numerically, the pair’s upper word is also the address used for the record. Byte order determines where that word appears within the initial eight-byte store.

The routine returns immediately after the offset-16 store. There is no explicit null check, capacity check, status check, alternative initialization, or release.

### Documented_only region: entry and initial operations

The remaining region requires a negative count. No possible selector byte satisfies that condition under the represented calculation. The following operations are established within the region, but execution through the established entry is not.

The region first:

1. Reads a word at captured-base offset 20.
2. Forms the **offset-24 address**, captured base plus 24.
3. Reads a word at captured-base offset 16 and adds 100.
4. Calls **F-U05** with twice the predecessor address, a word loaded at captured-base offset 12, and the increased offset-16 value.
5. Calls **F-U06** with the earlier offset-20 word plus 105, the offset-24 address, and the increased offset-16 value minus 76.

The offset-12 read is used for F-U05 after the earlier offset-20 and offset-16 reads. F-U05 intervenes before F-U06.

Three further values derive from predecessor address plus 110:

- **Derived base:** that sum shifted right by 26 using the represented signed shift.
- **Doubled shifted value:** that sum shifted right by 13, then multiplied by 2.
- **Adjusted derived value:** doubled shifted value minus 61.

Negative-operand shifts and overflow-sensitive arithmetic remain unresolved.

A word at captured-base offset 124 is read as the **captured payload**. The routine computes the 32-bit square of the offset-24 address, called the **squared base**, and performs these word stores:

| Address | Value |
|---|---|
| Derived base + 52 | Offset-24 address |
| Derived base + 56 | Adjusted derived value |
| Derived base + 60 | Captured payload |

The squared base subsequently serves as an address base. Its calculation is subject to the unresolved signed-overflow behavior.

### Documented_only region: global capture and narrow accesses

The routine reads a separate global value and calls **F-943b4f706744** with that value and 67.

Immediately after that call returns, it reads another global word as the **later captured base**. This read precedes F-U07 and all subsequent calls.

The next operations are:

1. Call **F-U07**, with no explicit arguments represented.
2. Store the low halfword of the doubled shifted value at adjusted derived value plus an auxiliary component read after F-U07.
3. Call **F-U08**, with no explicit arguments represented.
4. Read a signed byte at later captured base plus doubled shifted value.
5. Sign-extend that byte to a 32-bit argument and call **F-U09**.
6. Read an unsigned byte at doubled shifted value plus the unresolved byte-load component.

The signed-byte address uses the global value captured before F-U07. There is no fresh read of that global after F-U07 or F-U08.

For the unsigned-byte load, the auxiliary component is read after F-U09. The representation supplies no assignment or saved-value relationship establishing its producer. In particular, this contract does **not** attribute it to F-U08 or assert preservation across F-U09. Adjacency to F-U09 also does not establish F-U09 as its producer.

The loaded unsigned byte is retained both as an 8-bit value and as a zero-extended 32-bit value. That numerical byte value later serves directly as an address component.

### Documented_only region: squared-base stores

The region continues in order:

1. Call **F-U10**.
2. Store the doubled shifted value at squared-base offset 52.
3. Call **F-U11**, obtaining an ordinary 32-bit result.
4. Read a word at the zero-extended byte value plus 20.
5. Call **F-U12** with F-U11’s result and that loaded word.
6. Read a word at the zero-extended byte value plus 4.
7. Call **F-U13** with that word.
8. Store the captured payload at squared-base offset 100.
9. Call **F-U14**.
10. Store the doubled shifted value again at squared-base offset 52.
11. Call **F-U15**.
12. Store the adjusted derived value at squared-base offset 36.
13. Call **F-U16**.
14. Store the unresolved stored word at squared-base offset 20.

The last store is a 32-bit write. Its value is read after F-U16, but no assignment or saved-value relationship establishes its producer. It is not established as an F-U15 result retained across F-U16, nor is it established as a result produced by F-U16.

The two offset-52 stores remain separate observable operations, with intervening calls and memory operations.

### Documented_only region: final accesses and transfer

The remaining sequence is:

1. Call **F-U17**.
2. Call **F-U18**.
3. Read an unsigned halfword at doubled shifted value plus an auxiliary component read after F-U18.
4. Call **F-U19**, obtaining a 64-bit pair.
5. Store the earlier unsigned byte at loaded halfword plus that byte’s zero-extended value.
6. Read a halfword at the sum of F-U19’s low and upper words.
7. Call **F-U20** with the low word, upper word, 32-bit negation of the low word, and the loaded halfword.
8. Call **F-U21**.
9. Call **F-U22**.
10. Perform the unresolved indirect transfer **F-U23**.

The final byte store uses the byte loaded after F-U09. Its value is not the loaded halfword or a fragment of either paired-result word.

The terminal target, complete argument convention, and continuation behavior are unknown. This region is not established as cleanup or recovery.

## Side effects

### Memory and ordering

The established path explicitly performs:

- One byte store.
- Three word stores through the narrow base.
- One descriptor halfword store.
- One four-byte marker store.
- One eight-byte record store followed by three record word stores.

Its explicit reads are the descriptor halfword, the global supplying the captured base, and the selector byte.

The documented_only region adds the accesses described above, including the post-F-943b4f706744 global capture, both squared-base offset-52 stores, the unresolved-value offset-20 store, and the final byte store.

Ordering matters when locations alias. Dependencies may observe preceding stores, change memory before subsequent reads, fail, or not return. No general preservation of globals or auxiliary register state is assumed.

Access width does not establish atomicity. No synchronization is represented.

### Ownership and errors

No allocation or ownership policy is established. F-c260788e6b26 is not established as an allocator, and its destination is not established as fresh, private, or uniquely owned.

No explicit release, error result, or recovery mechanism appears on the established path. Invalid addresses, inadequate storage, unsupported alignment, incompatible conventions, or nonreturning dependencies can prevent completion. The evidence does not establish a particular trap, exception, or reporting policy.

## Dependencies

F-U identifiers are document-local opaque call-boundary IDs, not resolved published identities.

| ID | Established interaction |
|---|---|
| F-U01 | Supplies the descriptor address. |
| F-U02 | Followed by an auxiliary-word use and the captured-base global read. |
| F-U03 | Receives the logically halved auxiliary word; supplies a paired result. |
| F-U04 | Receives the doubled, adjusted low word from F-U03. |
| F-c260788e6b26 | Receives 115; supplies the pair used for record contents and destination. |
| F-U05–F-U06 | Receive the three-argument combinations specified above. |
| F-943b4f706744 | Receives a global-derived value and 67; precedes the later-base global capture. |
| F-U07 | Followed by an auxiliary-component use for a halfword-store address. |
| F-U08 | No explicit arguments or ordinary result use represented; not established as producer of the later unsigned-byte address component. |
| F-U09 | Receives the sign-extended byte; followed by the unresolved byte-load component use. |
| F-U10 | No explicit arguments represented. |
| F-U11 | Supplies an ordinary word result. |
| F-U12 | Receives that result and the word loaded at byte value plus 20. |
| F-U13 | Receives the word loaded at byte value plus 4. |
| F-U14 | No explicit arguments represented. |
| F-U15 | No explicit arguments represented; not established as producer of the later offset-20 store value. |
| F-U16 | Precedes the unresolved-word store; producing that word is not established. |
| F-U17 | No explicit arguments represented. |
| F-U18 | Followed by an auxiliary-component use for a halfword-load address. |
| F-U19 | Supplies the final paired result. |
| F-U20 | Receives the pair’s two words, negated low word, and loaded halfword. |
| F-U21–F-U22 | No explicit arguments represented. |
| F-U23 | Unresolved terminal indirect transfer. |

“No explicit arguments represented” does not establish a zero-argument machine interface. Auxiliary-result conventions and dependency memory effects remain unknown.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes returning dependencies and valid storage for the relevant accesses.

### Selector boundaries

With selector 0, the marker store starts at captured base minus 2, and record offset 8 receives 99.

With selector 255, the marker store starts at captured base plus 253, and record offset 8 receives 609.

Both take the same return path. No selector byte reaches the documented_only region.

### Narrow base and overlap

If the descriptor halfword is 4,660, the three initial word stores begin at 4,660, 4,664, and 4,668.

If the third input is zero, the initial byte store and first word store begin at the same address. The word store follows and overwrites that byte. Its resulting byte value depends on byte order.

### Arithmetic dependency and record

If F-U03’s low word is 54, F-U04 receives zero. If it is 55, F-U04 receives 2.

If F-c260788e6b26 returns a pair whose upper word identifies a writable 20-byte region, that region receives the pair, count, companion address, and predecessor address in the specified order. This expectation does not imply allocation.

### Documented_only capture and unresolved values

These examples describe the later fragment, not a reachable selector-driven execution:

- If F-U07 changes the global supplying the later captured base, the subsequent signed-byte address still uses the value captured before F-U07.
- If the unresolved byte-load component has value 1,000 and the doubled shifted value is 20, the unsigned-byte load targets 1,020. A value attributed only to F-U08 is insufficient to establish this address.
- If the unresolved stored word has bit pattern 37 when read after F-U16, squared-base offset 20 receives 37. A value attributed only to F-U15 does not determine that store.
- If the retained unsigned byte is 7 and the later unsigned halfword is 300, the final byte store targets address 307 and stores 7.

## Uncertainties

**Established static evidence:** Access widths, constants, explicit read/write and call ordering, global capture points, record layout, byte-derived count range, and the resulting nonnegative branch.

**Inferred intent:** The 20-byte destination can be described as a record, and base-relative accesses may concern structured storage. Neither interpretation establishes an application type or ownership rule.

**documented_only:** Execution and operational purpose of the later region. Its visible operations are described despite its unsatisfiable entry condition.

**Unresolved auxiliary provenance:** The producer of the byte-load component read after F-U09 and the producer of the word read after F-U16. No established saved-value relationship connects them to F-U08 or F-U15. Cross-call preservation for those proposed relationships is likewise unresolved.

**Other unknowns:** Complete calling convention, incoming saved-value provenance, auxiliary-result ABI, dependency semantics, byte order, alignment, overflow-sensitive behavior, negative signed shifts, concurrency, fault policy, and terminal-transfer continuation.

The later region must not be presented as a reachable fallback without additional evidence changing the control-flow or width interpretation.

## Evidence

This contract rests on static inspection of the supplied function representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent proof.

The established return follows from the unsigned-byte selector, count formula 2 × selector + 99, and subsequent nonnegative test. The record layout follows from the eight-byte store and word stores at offsets 8, 12, and 16.

The later global capture appears after F-943b4f706744 and before F-U07. Conversely, the representation does not establish producer assignments or saved-value relationships for the auxiliary component used after F-U09 or the word stored after F-U16. Their producers and proposed cross-call preservation remain unresolved.

No execution, runtime memory inspection, dependency implementation review, or behavioral testing was performed. Trusted provenance is supplied by the publishing system, not independently authenticated by this document.
