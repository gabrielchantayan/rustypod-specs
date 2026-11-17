<!-- Trusted provenance: F-ade6000d3f4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ade6000d3f4a

## Purpose

F-ade6000d3f4a performs an ordered sequence of indirect invocations, derives values from their results and implicit machine state, mutates fixed and computed storage locations, and finishes with an unresolved indirect transfer carrying four arguments.

The supplied static representation supports an access-and-value contract, not an application-level interpretation. It does not establish formatting, text output, initialization, collection management, or a protocol. Constants resembling character encodings remain numeric values.

This specification distinguishes:

- **Established static evidence:** represented operations, explicit access widths, constants, ordering, and retained-value dependencies.
- **Conditional deductions:** consequences of expressly stated arithmetic and storage assumptions.
- **Inferred intent:** application meaning; none is adopted as a requirement.
- **documented_only:** details not established by the supplied material.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The displayed interface has no explicit parameters and no explicit result. Execution nevertheless depends on incoming control state, an incoming machine value, indirect-invocation results, secondary machine results, global storage, and memory reached through computed addresses.

The first indirect destination depends on incoming control state plus 4094. An early invocation receives an incoming machine value shifted left by 23 positions. An empty explicit parameter list therefore does not establish independence from the caller.

Observable outputs comprise invocations, memory accesses, mutations, and the final transfer. No status code, returned object, or output-buffer convention is established. A return is represented after the final transfer; whether that transfer returns is **documented_only**.

### Widths and arithmetic

| Operation or quantity | Represented width |
|---|---:|
| Explicit byte access | 8 bits |
| Explicit halfword write | 16 bits |
| Explicit word access | 32 bits |
| Most arithmetic and address-like quantities | 32 bits |
| Selected paired invocation results | 64 bits |
| Load from global storage associated with absolute address 5 | **documented_only** |

A byte write retains the low eight bits. An explicitly unsigned byte load, when converted to a word, produces a value from 0 through 255. Paired results provide low and high 32-bit portions; some uses discard additional bits from the high portion. These extractions do not establish a record layout or complete calling convention.

**The global read associated with absolute address 5 is not an established unsigned-byte load.** Its declaration is absent. Conversion of its value to a 32-bit unsigned quantity establishes the converted value’s width, not the original load’s width or signedness.

Below, U denotes that converted, retained 32-bit value. U is not restricted to 0–255. Addresses derived from U use the converted value, without an established intervening eight-bit truncation.

For conditional deductions, the **32-bit bit-vector model** retains the low 32 bits after addition, subtraction, multiplication, and left shifts; address arithmetic is modulo \(2^{32}\). Actual signed-overflow behavior, integer-to-address conversion, and signed right shifts remain **documented_only**. No count-masking rule is assumed for excessive variable shift counts.

### Storage preconditions

Successful arrival at the final transfer requires all preceding dependencies to return and all reached accesses to succeed. Executable destinations, mappings, alignment, object extents, and lifetimes must satisfy the actual execution environment. No visible validation establishes these conditions.

Offsets describe accesses, not independent fields. Computed bases may coincide, and differently sized accesses may overlap. For the undeclared global at address 5, the complete storage extent required by its read is itself unresolved.

## Behavior

### Indirect invocation phase

A long straight-line sequence of indirect invocations precedes the explicit mutation region. After the initial incoming-state-dependent destination, most destinations are represented by fixed control-location values displaced by 4094. No visible branch or loop selects among these invocations.

Argument forms include shifted primary results, equal pairs, high portions paired with shifted low portions, zero values, masked subsets, and secondary machine results. Retained values sometimes survive intervening invocations and supply later arguments.

Similar argument forms do not establish identical destinations. An invocation displaying no arguments may consume implicit state or mutate storage. Dependencies are not assumed to preserve globals, machine state, or memory ownership.

Established extraction bounds include:

- Bits 10–19: 0–1023.
- Low 24 bits shifted right by 20: 0–15.
- Low 24 bits shifted right by 19: 0–31.
- Low 17 bits shifted right by four: 0–8191.

These bounds apply to the specified extractions, not to U. Under the conditional bit-vector model, successive left shifts by 25 and 11 discard every original bit and produce zero.

### Retained values

The following mathematical labels describe values, not implementation identifiers or object types.

| Label | Meaning |
|---|---|
| P | A dependency result multiplied by \(2^{25}\) |
| Q | An earlier dependency result shifted left by 25 |
| R | Another dependency result shifted left by eight |
| T | Low 15 bits of an already eight-position-left-shifted result, logically shifted right by six |
| L, H | Low and high 32-bit portions of the last paired result before explicit mutations |
| U | Value of the undeclared global associated with address 5, converted to 32-bit unsigned |
| K | Word sampled at U plus 84 |
| C | \(4K\) |
| A | \(4K-150\) |
| S | \(4A=16K-600\) |
| G | A retained global-derived 32-bit value sampled before the expanded stores |
| D | Twice a later linked word sample |

Potentially overflowing formulas use the conditional bit-vector model. Under that model, P and Q are multiples of \(2^{25}\); T is a multiple of four from 0 through 508.

### Opening mutations and the cleared-word invariant

The opening accesses write Q as a word at H plus 20, H’s low byte at Q plus 9, and P as a word at \(256L+84\). Q plus 9 then receives Q’s low byte. The two byte writes are distinct events.

The word at P plus 20 is cleared, followed by an explicit unsigned-byte sample at P plus 5. A zero word is written at \(2T+20\), the sampled byte is zero-extended and stored as a word at P plus 52, and the word at P plus 20 is sampled.

Under ordinary byte-addressed storage, successful accesses, bit-vector arithmetic, and no external or read-triggered mutation, **this last sample is zero**. The intervening clear can only write zero bytes. The word at P plus 52 is disjoint from the word at P plus 20, including across address wraparound. This deduction does not depend on byte order.

Another zero word write at \(2T+20\) follows. Two consecutive byte writes at \(2T+21\) receive the cleared-word sample’s low byte; both therefore write zero under those assumptions.

Next, the global associated with absolute address 5 is evaluated and converted to U. Its load width and signedness remain unresolved. After this evaluation:

- Absolute address 1 receives the low byte of \(2T\).
- U plus 21 receives T’s low byte.
- P plus 17 receives R’s low byte.
- The cleared-word sample plus 84 receives R as a word.
- P plus 17 is cleared.

Under the cleared-word assumptions, the R word write targets absolute address 84.

K is then sampled as a word at U plus 84 before that location is replaced by the retained cleared-word sample. U and K are retained values; neither is reloaded for this replacement. No 0–255 restriction on U or corresponding low-address restriction on these dependent accesses is established.

### Chained mutations

The next region follows word-derived bases at offsets 4, 100, and 116 and explicit unsigned-byte-derived bases at offset 5. Those later explicit byte loads are distinct from the undeclared global read that produced U.

The region writes T at \(2T+52\), samples a base at P plus 116, and writes twice P at that base plus 84. It also represents a self-valued word write through a base obtained from \(2T+116\). Repeated reads of this source remain separate where reads have effects or external mutation is possible; their within-expression evaluation order is not established here.

Self-valued word writes at K plus 36 and byte writes of K’s low byte at the P-derived base plus 9 occur on both sides of an R word write at that base plus 84.

The following subsequent effects have distinct order and dependencies:

- \(2T+20\) receives twice P as a word.
- Absolute address 84 receives \(2K\) as a word.
- \(2T+20\) receives K twice as separate word writes.
- \(2T+21\) receives C’s low byte twice.
- Absolute address 21 receives the low byte of \(2T\).
- C plus 84 receives R as a word.
- \(2T+52\) receives C as a word.

The fixed-address-84 write is separate from the earlier R write. Both use values already retained. The absolute-address-21 write is distinct from the computed writes at \(2T+21\), except when those addresses coincide.

Further accesses first replace C plus 84 with C and use a doubled word sampled at \(2T+116\). A byte at C plus 5 supplies an explicitly unsigned base for a word read at offset 116. Another chain uses words at C plus 100 and C plus 4.

These chains cause word writes at offsets 20, 68, and 84 and byte writes at offsets 9, 13, and 21. The byte at the C-plus-4-derived base plus 5 is read more than once, with intervening writes. Its later sample cannot unconditionally reuse its earlier value.

The later sample supplies a linked word whose double is D and whose quadruple is retained for the expanded stores. Before those stores, numeric 58 is written as a byte at twice D, as a halfword at four times D, and as a word at twice D plus 32.

After C is reduced by 150 to form A, the word at S plus 32 receives, in order:

- Numeric 32.
- Numeric 47.
- A zero-extended unsigned byte sampled at S plus 4.
- Numeric 32 again.

The byte sample follows the first two word writes. G is then sampled from global storage, and absolute address 96 receives a word containing 32.

### Expanded stores

Each represented group has four ordered effects:

1. A word write at a destination base plus 84.
2. A byte write at G plus 25 containing S’s low byte.
3. A byte write at S plus 17 containing G’s low byte.
4. A word write at G plus 52.

The first group uses \(8K\) as both destination base and final word value. Subsequent matching groups use \(8A\). The offset-84 word value is the retained quadrupled linked value, equal to \(2D\) under the conditional arithmetic model.

G is retained, not sampled per group. An aliased write to its original global storage does not retroactively change G.

The groups are explicitly expanded, not controlled by a visible loop. Repeated identical writes remain separate access events; reducing them to a final memory image is not equivalent. Exact cardinality is **documented_only** in this specification.

### Terminal samples and address 58

The terminal region retains four word-sized samples:

| Label | Source |
|---|---|
| W | Global storage represented at absolute address 169 |
| J | \(2D+116\) |
| V | D plus 20 |
| E | S plus 4 |

The represented order is W’s sample, J’s sample, a byte write at absolute address 58, V’s sample, then E’s sample. The byte at 58 receives the low eight bits of \(8A+48\).

W and J retain their earlier values even if the byte write overlaps their source storage. V and E are sampled afterward.

Ordinary byte overlap can allow the address-58 write to affect V. It cannot directly affect E under the conditional model:

\[
S+4=16K-596\equiv12\pmod {16}.
\]

E occupies residues 12–15 modulo 16; address 58 has residue 10. The exclusion holds across 32-bit wraparound. It neither authorizes moving E’s read nor excludes device, external, or nonordinary mapping effects.

### Remaining terminal effects

After these samples, S’s low byte is written at D plus 13. Numeric 108 is written at S plus 8, and G’s low byte at S plus 13 and S plus 9. D plus 84 receives D as a word, followed by a write of W’s low byte to absolute address 103.

A word at D plus 68 is sampled before address 103 is read to supply the write at absolute address 33. The retained word supplies:

- A byte at S plus 16.
- A word at G plus 84.
- The operand for a variable-count right shift, whose low byte is written at D plus 1.

The shift count is S’s low eight bits. Counts outside the supported range are not assigned masked semantics. G plus 84 is subsequently overwritten with S.

A word sampled at S plus 64 is retained across the word write of E at G plus 116, then written at S plus 16. G plus 68 receives D. A fresh address-103 read supplies S plus 9; D plus 9 receives S’s low byte; S plus 68 receives W.

A word at S plus 84 is then retained as X. Another fresh address-103 read supplies another S-plus-9 write. Intervening aliasing may change address 103, so these reads cannot unconditionally reuse W’s low byte.

X plus 13 receives J’s low byte, and D plus 84 receives V. A word at S plus 20 is retained as Y. D plus 17 receives J’s low byte, S plus 9 receives X’s low byte, and S plus 68 receives X.

### Final linked writes and transfer

A word sampled at D plus 4 provides a derived base F. A halfword containing 101 is written at \(2F\) before either terminal argument is sampled through F.

The word at F plus 100 is retained as B. X plus 17 then receives Y’s low byte, and X plus 84 receives a word containing 100. Only afterward is the word at F plus 116 retained as Z.

The remaining order is:

1. Sample a word at D plus 100 and retain it as N.
2. Write F’s low byte at D plus 9.
3. Write retained B as a word at N plus 84.
4. Write retained N as a word at \(2B\).
5. Perform the unresolved indirect transfer.

The transfer’s displayed arguments are, in order, numeric 114, B, V, and Z. Later writes cannot retroactively alter these retained arguments. The write at N plus 84 cannot be omitted or moved after the write at \(2B\).

## Side effects

Explicit computed accesses use 8-, 16-, and 32-bit widths. Significant fixed-address activity includes:

| Address | Activity |
|---|---|
| 1 | Byte write containing the low byte of \(2T\) |
| 5 | Undeclared global read; value converted to U; original width and signedness unresolved |
| 21 | Byte write containing the low byte of \(2T\) |
| 33 | Terminal write supplied by a fresh address-103 read |
| 58 | Byte write between J’s and V’s samples |
| 84 | Conditional earlier R word write; later word write containing \(2K\) |
| 96 | Word write containing 32 |
| 103 | Narrow write from W and subsequent separate reads |
| 169 | Represented word-sized global sample W |

Computed accesses may reach these locations too. The table does not promise their final contents.

Byte order, atomicity, volatility, synchronization, and device semantics are **documented_only**. Named global representations are not instruction-level confirmation of load widths where declarations are absent.

No explicit allocation, release, reference counting, or ownership transfer is identifiable. This does not restrict unresolved dependencies from performing such operations. Stored address-like values establish neither ownership nor lifetime.

No local error checking, rollback, or transactional guarantee is visible. A fault or nonreturning dependency may leave earlier effects in place. No particular exception category or graceful response is established.

## Dependencies

F-ade6000d3f4a is the only supplied opaque function identity. No supplied opaque IDs identify the initial destination, subsequent indirect destinations, or terminal destination. No dependency identities are fabricated.

For each unresolved dependency, calling convention, implicit inputs, results, global preservation, memory effects, ownership effects, failure behavior, and return behavior remain **documented_only**.

The explicit fixed-address mutations and final linked writes are operations represented within F-ade6000d3f4a, not merely possible dependency effects.

## Acceptance examples

These are conditional static examples, not executed tests.

### Global conversion without an eight-bit bound

If the undeclared global’s evaluation converts to U equal to 256, its dependent byte write targets 277, and K is sampled as a word at 340 under the bit-vector model. Replacing U with its low byte would incorrectly select addresses 21 and 84.

This example does not establish that the actual global can hold 256; its type is unresolved.

### Conditional unsigned-byte global

If an additional storage declaration establishes an unsigned eight-bit global and its sampled value is 254, U is 254. The dependent byte write targets 275, and K is sampled at 338.

The 0–255 bound is valid only under that additional declaration, not from the displayed conversion alone.

### Cleared-word sample

With ordinary byte storage, successful accesses, bit-vector arithmetic, and no external or read-triggered mutation, the word sampled at P plus 20 is zero. The following R word write therefore targets absolute address 84, independently of U’s load width.

### Distinct fixed-address writes

If retained K is 7, the later word write at absolute address 84 contains 14. It remains distinct from the earlier R write.

If T is 4, computed address \(2T+21\) is 29, while the later absolute-address-21 write stores numeric 8.

### Terminal ordering and retained values

If D plus 20 equals 58, V is sampled after the address-58 write. Its overlapping byte reflects that write absent another mutation; its full numeric value depends on byte order and other bytes.

If B is 200 and N is 500, the final word writes target 584 with value 200, then 400 with value 500. The second transfer argument remains 200 even if those writes overlap B’s source.

### Narrowing and partial execution

An explicit byte write from a value ending in hexadecimal AB stores AB. An explicit unsigned-byte sample FE, zero-extended into a word, produces numeric 254. Neither rule establishes the undeclared address-5 global’s type.

If that global’s required storage cannot be read when reached, no graceful return, diagnostic, or rollback is established. Its required read extent cannot be determined from the conversion.

## Uncertainties

The following remain **documented_only**:

- Application purpose and completeness of the displayed interface.
- Indirect destination identities and calling conventions.
- Secondary result conventions and preservation rules.
- Original load width, signedness, and storage extent of the global associated with address 5.
- Any restriction of U to a byte range.
- Instruction-level details of undeclared named-global accesses.
- Actual signed arithmetic, address conversion, and variable-shift semantics.
- Byte order, mappings, alignment, aliases, and external mutation.
- Object extents, ownership, validity, and lifetime.
- Exact enumeration of expanded store groups.
- Within-expression ordering of repeated source reads where not otherwise established.
- Terminal destination effects, return behavior, and repeat-call safety.

The opening zero invariant and E’s exclusion of ordinary overlap with address 58 are conditional deductions, not unresolved aliasing questions within their stated assumptions.

## Evidence

The supplied representation supports the explicit access widths, constants, retained-value relationships, and relative ordering described above. Prior prose, comments, and reviewer feedback are not independent behavioral evidence.

The global associated with address 5 appears as an undeclared named value converted to a 32-bit unsigned quantity. Unlike later explicit unsigned-byte dereferences, that expression does not establish its original load width or signedness. The U-plus-21 and U-plus-84 accesses depend on the converted value without a represented byte mask.

The opening clear and intervening zero-writing accesses support the conditional zero invariant. The retained quadrupling, subtraction of 150, and further quadrupling support \(S=16K-600\) and E’s residue constraint.

The terminal region separately retains B and Z, samples N, writes F’s low byte, writes B at N plus 84, then writes N at \(2B\) before the final transfer.

No dependency implementations, instruction-level confirmation, validated memory map, or runtime observations were available. No runtime tests have been performed.
