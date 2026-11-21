<!-- Trusted provenance: F-a1e9da25fcf4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1e9da25fcf4

## Purpose

F-a1e9da25fcf4 performs a long sequence of dependency calls, byte-selected writes, word-state updates, and one halfword write. Its effects depend on the explicit input, incoming ambient state, dependency results, and memory read during execution.

No application-level purpose is established. Allocation, serialization, initialization, reference counting, and cleanup are not supported descriptions. Masks and shifts suggest encoded-state manipulation, but that is **inferred intent**, not a guaranteed semantic interpretation.

The supplied static representation reports incomplete instruction recovery, omitted unreachable material, and overlapping global symbols. It ends without a recovered normal return. This contract therefore describes the represented execution prefix conditionally, not a complete successful operation.

## Interface

### Explicit input

The visible interface accepts one opaque **32-bit word**. The first dependency receives it unchanged.

Its semantic domain is unknown. It is not established as a pointer, handle, size, flag set, or ownership-bearing reference. No explicit validation of this input is represented.

### Implicit inputs

The represented operation also consumes:

- Incoming control state used to derive the first call destination.
- Two incoming 32-bit ambient words without visible initialization in this operation.
- Primary and secondary results associated with dependency calls.
- Bytes at computed selector addresses.
- Words at computed addresses.
- A byte at numerical address **0x00000001**, through multiple distinct references.
- A word at numerical address **0x00000028**, through three distinct references.
- A global word whose numerical location is unresolved, through two sequential references.

These memory references occur at distinct execution points. They do not constitute an entry snapshot, and repeated references do not establish equal values.

Numerical address **0x0000007c is output storage**, not an established fixed-address memory input. The representation explicitly writes a captured dependency result there; it contains no explicit read naming that address. Unknown dependencies might read it, and computed addresses might alias it, but neither possibility establishes an additional direct fixed-address read.

### Outputs and completion

The visible interface declares no return value. Its established outputs are dependency invocations and memory effects.

Some calls provide a represented 32-bit result. Others provide a two-word carrier:

- **Low word:** bits 0–31.
- **High word:** bits 32–63.

This division does not establish whether the dependency semantically returns a 64-bit scalar, two machine results, or a calling-convention artifact. Secondary results used outside such carriers also lack a complete calling-convention contract.

No output-buffer parameter, capacity parameter, or output-length result is established.

Normal return, trapping, nonreturning transfer, and continuation through unrecovered instructions remain **documented_only**.

### Preconditions and ownership

For any represented prefix to complete, every reached dependency destination must support execution and every reached memory access must be supported at its represented width.

No visible guard establishes address validity, alignment, capacity, null exclusion, or safe arithmetic bounds. Numerical address zero and other low addresses are used explicitly.

No allocation, release, retention, ownership transfer, or lifetime guarantee is established. Dependencies must not be assumed to preserve globals or to acquire or release passed values.

## Behavior

### Ordering and observation model

The recovered body presents a sequential trace without a visible conditional branch or loop. This does not constrain control flow inside dependencies or beyond the recovered prefix.

The following distinctions are part of the represented behavior:

- Separate statements retain their relative order.
- A selector read precedes its associated selected write.
- Intervening calls remain boundaries at which dependency behavior is unknown.
- A captured operand is not refreshed merely because memory changes later.
- A represented reread is not replaced by a previously captured value.
- Consecutive stores remain separate effects, including identical stores.

**Evaluation order between separate argument expressions of the same call is not established.** Distinct memory references in those expressions must remain distinguishable without imposing an unsupported argument-evaluation order.

These are static representation-level requirements. The evidence does not establish which accesses are externally observable under ordinary-memory optimization, device semantics, concurrency, or the actual unrecovered instruction sequence.

### Widths, layout, and arithmetic

Principal words and address calculations are represented at 32-bit width.

- Byte reads are unsigned, yielding values from 0 through 255.
- Byte stores retain the low eight bits of their source.
- The halfword store retains the low 16 bits.
- Word accesses have width 32 bits.
- Unsigned right shifts are logical.
- Unsigned 32-bit arithmetic retains the low 32 bits where applicable.

Masks retain the low 31, 30, 29, 22, or six bits at different points. Later narrowing may discard additional bits. For example, masking to 30 bits, shifting right by 20, and storing a byte retains original bits 20–27.

Some address and value expressions pass through signed intermediates, signed products, subtractions, or signed left shifts. Their overflow behavior cannot be generalized to modular arithmetic without instruction-level evidence.

Offsets such as 4, 8, 12, 20, 28, and 52 are numerical byte offsets. They do not establish one common structure layout. Byte order, unaligned-access behavior, and memory-map validity remain **documented_only**.

### Entry and early selector updates

The first dependency destination is derived from incoming control state, and that dependency receives the explicit input unchanged. Several calls precede the first direct memory access. One receives an incoming ambient word logically shifted right by 11.

The first represented memory update reads an unsigned byte one byte beyond an ambient address. It then stores a byte at the selector’s numerical value plus 17. The stored value comes from the other ambient word after clearing its top bit and shifting right by 21.

Consequently, this selected destination lies between numerical addresses **17 and 272 inclusive**. It is not established as an offset within the object containing the selector.

Later calls consume dependency results shifted left by two. One also receives the previously captured selector.

An early scalar is explicitly zero. A subsequent selector address derived by multiplying it by 1,024 and adding one is therefore numerical address 1. The byte is read at that later point, and zero is stored at its value plus 17. Intervening dependencies may have changed that byte.

### Recurring selector-selected writes

Many later updates read a selector byte at a computed address and write one byte at that selector value plus 17.

Selector addresses include retained values plus one, scaled values plus one, and masked or shifted values plus one. Represented scales include 2, 4, 8, 16, 64, 1,024, 4,096, and larger powers of two.

Stored sources include:

- Zero.
- A word logically shifted right by 22.
- A word masked to 30 bits and shifted right by 20.
- A word masked to 22 bits and shifted right by 12.
- Four times a captured word.
- The low byte of a captured word read.
- A fragment taken from the high portion of a two-word carrier.

These patterns establish neither a shared base pointer nor a stable selector table. Every represented selector read obtains its own value. Calls between updates prevent any assumption that selector storage remains unchanged.

### Early word-state changes

F-23325cd5e50c receives one captured 32-bit dependency result.

Later, a captured dependency result is stored as a **32-bit word at numerical address 0x0000007c**. The following call receives that same captured result twice. The word store precedes the call; the arguments do not require rereading the output storage.

A subsequent word read uses an address supplied through a secondary dependency result, plus 20. The captured word contributes to later call arguments and eventually to a word store after a left shift by 11.

### Three reads at numerical address 0x00000028

The representation distinguishes three word references:

1. An initial read whose value is retained.
2. A separate read in one argument expression of a later dependency call.
3. Another separate read in another argument expression of that same call.

The retained read precedes both call-argument reads. The relative evaluation order of the argument expressions is unresolved.

The two arguments are not specified as copies of the retained value. Their equality is also not established merely because they reference the same address.

After that dependency and another intervening call, a word store writes the initially retained value to an address formed from four times a secondary result, plus 20. This store does not reread 0x00000028. A later change to that global does not replace the captured store operand.

### Composite results and direct byte destinations

One two-word result supplies a retained low word and a high word used as an address base. A word is read at the high-word base plus eight, followed by another word at that base plus 12. Both reads precede the next pair of calls.

The second captured word later supplies a selector-read address after adding one. The selected write stores the low byte of the first captured word.

Elsewhere, a word is read four bytes beyond a retained result address. Its low byte is later written through a selector-derived destination after intervening calls.

Another two-word result causes a direct byte store at **high word plus 28**, using the low byte of the low word. This destination is not selected through the selector-plus-17 mechanism.

### Two address-1 reads used directly as call arguments

A distinct unresolved dependency call receives **two separate direct byte reads of numerical address 0x00000001**.

This occurs after a selector-selected update whose source is a retained word masked to 30 bits and shifted right by 20. Calls intervene between that update and the two-read call. After the two-read call, another unresolved call occurs before the next selector-derived zero store.

Each argument expression independently references the byte at address 1. The represented byte values are unsigned, in the range 0–255; the full calling-convention treatment remains unresolved.

The contract does not establish:

- Which argument expression is evaluated first.
- That the two reads obtain equal values.
- That either value equals a previously captured selector.
- That one captured read can replace both argument references.

The next zero store uses a selector address formed from a retained low result word multiplied by 2²⁵, then increased by one. It is a separate selector access, not reuse of either call-argument byte. The signed intermediate arithmetic in this address expression retains the stated uncertainty.

### Zero-derived and overlapping accesses

Several explicitly zero values remain zero through represented shifts and scaling until reassigned.

One sequence writes a dependency result as a word at eight times an explicitly zero value: **numerical address zero**. Later, a halfword store uses a masked and shifted address derived from the same zero value: also **numerical address zero**.

Calls and other accesses separate these stores. If they reach the same ordinary storage, the halfword store overlaps part of the earlier word store. Byte order determines which byte positions are affected. Intervening dependency effects prevent an unconditional final-content claim.

A later direct byte destination combines four times that zero value with a retained masked-and-shifted value. It is distinct from the selector-selected destinations.

No visible null guard, rollback, or recovery accompanies these accesses.

### Repeated stores and differing address scales

A later pair of word stores targets the same address formed from a retained unsigned word shifted right by nine, plus 52. The first writes a captured selector widened to a word; the second writes twice that selector. No call intervenes.

Another pair writes the same dependency result shifted left by two twice to an address formed from a secondary result plus 52. Both stores remain represented effects.

Subsequent word stores use several distinct address forms, including four times a result plus 20 and sixteen times a result plus 20. Larger scales also occur. These forms must not be normalized into one field address.

One later call contains two separate word-read argument expressions at a common computed address. One read value is shifted right by 12; the other is shifted left by two. Their evaluation order and equality are unresolved. A preceding store uses a different address scale and therefore does not generally supply those reads.

### Two reads of the unresolved-location global

Following a word store at four times a secondary result plus 20, the operation reads the unresolved-location global twice in sequence:

- The first read supplies a retained word.
- The second supplies the operand for multiplication by four, producing a separately retained scaled value.

No dependency call is represented between the reads. That alone does not establish memory stability or equal results.

The first captured word is later masked to 30 bits, shifted right by 10, and passed to a dependency.

The separately scaled value supplies a selector-read address after adding one. After intervening calls, that selector address is read again. The scaled value also feeds later shift and scaling calculations used in additional addresses.

The selector reread does not reread the global. The first captured global word is not reconstructed from the second read. Signed arithmetic in the subsequent scaling chain remains unresolved where overflow is possible.

### Final represented portion

F-674e75a953e8 receives the same captured word twice before further updates. The remaining trace includes additional word stores, selector reads, and selected byte writes.

The final represented memory effect is a selector-selected byte store. Further unresolved calls follow it. No normal epilogue or successful return is recovered.

## Side effects

Established direct-effect categories are:

- Dependency invocations.
- Byte and word reads at fixed and computed addresses.
- Two distinct address-1 byte reads used directly as arguments of one call.
- Three word reads at 0x00000028.
- Two sequential reads of the unresolved-location global.
- Byte, halfword, and word stores.
- A word write to output storage at 0x0000007c.
- Word and halfword writes at address zero.
- Potentially overlapping accesses and consecutive duplicate stores.

The affected address set cannot be bounded from the explicit argument alone. Ambient values, dependency results, and mutable memory contribute to addresses.

No atomicity, synchronization, rollback, idempotence, reentrancy, or thread-safety guarantee is established. Earlier effects may remain after a later failing call or access. No explicit error result, exception translation, compensation, or cleanup path is represented.

## Dependencies

Mapped dependencies occur in this relative order, with unresolved calls and memory effects between them:

| Opaque function ID | Visible argument relationship |
|---|---|
| F-23325cd5e50c | One captured 32-bit result. |
| F-bbb54b5f036d | The same captured 32-bit result twice. |
| F-f80b3ac065fc | Low and high words of a two-word carrier, followed by two repetitions of its low word. |
| F-de15789688a8 | The same captured 32-bit result twice. |
| F-ea83f9a17404 | The same captured word twice. |
| F-674e75a953e8 | The same captured word twice. |

Their purposes, memory effects, failure behavior, ownership rules, and complete calling conventions remain **documented_only**.

Other call sites lack supplied opaque identities. They include control-state-derived destinations, computed destinations, and other explicitly represented targets. No semantic identities are assigned to them.

In particular, the dependency receiving two direct reads of 0x00000028 and the dependency receiving two direct byte reads of address 1 are unresolved. Neither call has the captured-value repetition relationship shown for mapped dependencies above.

## Acceptance examples

These are conditional static examples, not executed tests.

### Selector destination and narrowing

If a selector read yields 0x20, its selected destination is numerical address 0x31.

If a stored source is 0xffc00000 logically shifted right by 22, the intermediate value is 0x3ff, but the byte written is 0xff.

### Output storage at 0x0000007c

If the relevant dependency result is 0x12345678, the operation writes that word to 0x0000007c before calling the next dependency with 0x12345678 twice.

No prior value at 0x0000007c is needed for this fixed-address operation. Describing it as read-modify-write would add unsupported behavior.

### Independent address-1 argument reads

Suppose the argument expression for the first argument obtains 3 and the expression for the second obtains 9. The call receives those respective values, regardless of which expression was evaluated first.

This scenario is conditional on memory behavior permitting distinct results. It does not establish such behavior. If both reads obtain 7, that is also compatible; equality does not eliminate either represented read.

### Three reads with different roles

Suppose the retained read at 0x00000028 yields 0x11223344, while the two argument expressions obtain 0x55667788 and 0x99aabbcc.

The later word store uses 0x11223344. The call arguments use their own reads. Their evaluation order is unresolved.

A dependency mutation after capture does not refresh the retained store operand.

### Independent unresolved-global captures

If the first unresolved-global read yields 0x00000c00 and the second yields 3, the first value’s mask-and-right-shift-by-10 computation produces 3.

The second value produces a separately scaled value of 12 and an associated selector address of 13. Deriving that address from the first read would conflate independent inputs.

### Repeated selector access

If a selector location initially yields 3, the associated selected destination is 20. If an intervening dependency changes the location to 9, a later reread selects address 26.

### Consecutive stores

For selector 7, the offset-52 pair writes 7 and then 14 to the same word destination. Keeping only the final ordinary-memory value omits a represented store event.

### Unsigned scaling and failure

For unsigned 32-bit value 0x40000001, multiplication by four retains 0x00000004. This does not settle signed-expression overflow elsewhere.

If execution fails after the write to 0x0000007c, no restoration of that storage or earlier effects is established. Returning after the last recovered call is not an established completion behavior.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and valid semantic input domain.
- Ambient-state provenance and complete calling convention.
- Unresolved dependency identities and behavior.
- Semantic interpretation of two-word carriers and secondary results.
- Numerical location of the unresolved global.
- Stability, volatility, and synchronization of repeated memory reads.
- Evaluation order between separate call-argument expressions.
- Byte order, alignment, memory-map validity, and device semantics.
- Signed overflow and signed left-shift behavior.
- Ownership, lifetime, allocation, retention, and release.
- Dependency access to or preservation of globals.
- Aliasing between computed and fixed numerical addresses.
- Omitted control flow, terminal behavior, and complete failure behavior.

## Evidence

This contract is based on the supplied static representation and opaque dependency map. **No runtime tests, call tracing, or memory observations have been performed.**

Established static evidence includes represented access widths, distinct memory references, capture-before-use relationships, masks, shifts, repeated captured arguments, and statement-level sequencing.

The representation specifically distinguishes output storage at 0x0000007c from fixed-address inputs, two address-1 byte references used directly as call arguments, three word references at 0x00000028, and two sequential reads of the unresolved-location global.

Encoded-state manipulation is inferred intent only. Instruction completeness, recovered type accuracy, calling conventions, and terminal behavior remain unresolved; no complete executable or normal-completion contract is claimed.
