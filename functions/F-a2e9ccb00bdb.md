<!-- Trusted provenance: F-a2e9ccb00bdb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2e9ccb00bdb

## Purpose

F-a2e9ccb00bdb is a globally configured state machine that returns a 32-bit word. Its selectable operations prepare arithmetic state, invoke opaque dependencies, publish four words, and write individual bytes through computed addresses.

The static evidence establishes these operations and their sequencing within individual selections. It does not establish a required overall sequence. Global values control initial dispatch, dispatch adjustment, and transitions. Setup, processing, publication, and return can be reached independently, subject to the validity of the state they consume.

Allocation, transformation, and cleanup are possible interpretations, not established behavior. A pointer-taking dependency call does not establish ownership transfer or memory release.

## Interface

### Arguments and return

| Argument | Representation | Established use |
|---|---|---|
| 1 | 32-bit word | Saved on entry; selection 1 multiplies it by the sampled scale to set the shared call scalar. |
| 2 | Opaque 32-bit word | Saved on entry; no later consumption is visible. |
| 3 | 32-bit word | Saved on entry; selection 1 installs it as the first working operand. |
| 4 | Opaque 32-bit word | Saved on entry; selection 1 installs it as the first operand’s companion. |
| 5 | 32-bit word | Selection 1 installs it as the second working operand. |
| 6 | Opaque 32-bit word | Selection 1 installs it as the second operand’s companion. |
| 7 | Address of word storage | Selection 1 installs it as the publication destination. |

The return accumulator starts at zero. Selection 12 replaces it; selection 0 returns its current value.

The companion words are not used as high halves of the wide multiplications. Arguments 3–4 and 5–6 therefore do not establish two 64-bit input quantities.

### Widths and access requirements

Arithmetic on words retains the low 32 bits. Complements and bitwise combinations operate on 32 bits. Selection predicates described as sign tests inspect bit 31, rather than performing a magnitude comparison.

The two wide products in selection 17 are unsigned 64-bit products. Each begins with a multiplication that has already wrapped to 32 bits. Byte writes retain the low eight bits of their arithmetic value.

Address arithmetic uses a 32-bit representation, including conversion of a local-array address into a word. Portable pointer semantics are not established.

Reached operations require:

- Four writable publication words at byte offsets 0, 4, 8, and 12.
- Readable words at offsets `0x70` and `0x6c` from the saved address-like result.
- A writable byte at each computed byte destination.
- Validity sufficient for the opaque dependencies’ actual behavior.

No visible validation establishes bounds, alignment, lifetime, or null safety. These are conditional requirements for the operations reached, not unconditional requirements on every invocation.

## Behavior

### Entry and initialized state

Entry observes the scale first, the multiplier second, and the initial raw state later. Scale and multiplier are retained for subsequent arithmetic; these reads do not constitute an atomic configuration snapshot.

The following state starts at zero:

- Return accumulator and shared call scalar.
- Saved address-like dependency result.
- Both working operands and both companion words.
- Processing first-argument quantity and processing third argument.
- Saved scaled processing return.
- Saved multiple used by byte generation.
- Unsigned result quantity.
- Iteration-like value and index word.

Five distinct pointer states start null: publication destination, publication comparison pointer, saved comparison pointer, current indexed-write pointer, and saved byte base.

The index is the first word of a separate three-word arithmetic array. Only that word is initialized on entry. Selection 17 initializes its remaining two words.

Local result storage, separate publication source fields, and two additional pointer fields have no visible initialization. The address passed to the processing dependency identifies a declared two-word local array; the evidence does not establish that all publication sources form a contiguous four-word result structure.

### Common dispatch behavior

Before each selection, the function captures the processing third argument, both additional pointer fields, and a global pass-transition word. It then reads a dispatch offset and adds it to the current raw state with 32-bit wrapping.

Adjusted values 0–17 select explicit operations. Every other value selects the default operation. Transition assignments establish raw states; each subsequent pass reads and applies the dispatch offset again.

The two additional pointer captures occur even when the selected operation does not use them. Their missing initialization is therefore a definedness uncertainty even on an apparent immediate-return path.

Repeated global reads remain separate observations. Where an expression does not establish relative evaluation order, that order is unresolved.

### Pointer roles

| Role | Direct assignment | Use |
|---|---|---|
| Publication destination | Argument 7 in selection 1 | Four publication stores |
| Publication comparison pointer | Null on entry and in selection 1 | Copied and compared in selection 4 |
| Saved comparison pointer | Publication comparison pointer in selection 4 | Comparison in selection 5 |
| Current indexed-write pointer | Selections 11, 14, and 15 | Saved by selection 9 |
| Saved byte base | Current indexed-write pointer in selection 9 | Byte destination in selection 10 |

The two additional pointer fields are separate from these five roles. Neither comparison pointer receives a direct nonzero assignment, but preservation against opaque calls or aliased writes is not guaranteed.

### Return, setup, and transition-only operations

**Selection 0** returns the current accumulator. Common pass captures and global reads have already occurred.

**Selection 1** resets the publication comparison pointer to null, sets the shared call scalar to scale times argument 1, and installs arguments 3–6 in the working operands and companions. It then reads its transition global and installs argument 7 as the publication destination. It does not reset all arithmetic state or establish an indexed-write pointer.

**Selections 3, 8, and 16** read their respective transition globals and continue without another selection-specific effect.

### Selection 2: address-like result

F-5e11a17b9827 receives the shared call scalar multiplied by a fresh global factor, followed by a separately read global word. Relative evaluation order of those argument-related reads is unresolved.

Its returned word replaces the saved address-like result. After the call, one transition location is read twice. A zero result selects the first observation minus seven; a nonzero result selects the second observation.

This is only a zero/nonzero branch. It does not establish allocation, usable extent, ownership, or a failure convention.

### Selection 17: wide arithmetic preparation

Each working operand is first multiplied by the sampled scale at 32-bit width. That wrapped result is interpreted as unsigned and multiplied by the unsigned sampled multiplier at 64-bit width.

For the second working operand:

- The product’s high half becomes the arithmetic array’s third word.
- Its low half, multiplied by scale at word width, becomes the processing third argument.
- The array’s second word is cleared; its index word is unchanged.

For the first working operand:

- The complemented low half and the low half plus two are saved separately.
- The low half multiplied by scale becomes the processing first-argument quantity.
- The complemented high half, shifted left once at word width and with bit zero set, is saved separately.

The auxiliary saved words have no visible later consumer. A transition global is read afterward.

Without intervening indirect mutation, reaching this operation before selection 1 uses zero operands and produces zero wide products and zero processing argument quantities.

### Selection 13: processing dependency

F-23b828769638 receives:

1. Multiplier times the processing first-argument quantity.
2. The word at saved-address offset `0x70`.
3. The current processing third argument.
4. The address of local result storage.
5. The word at saved-address offset `0x6c`.

Both offset reads precede the call; their relative order is unresolved. The third argument is read for the call, whereas a separate, earlier pass capture is retained for the later shared-scalar assignment.

The returned word is multiplied by scale and saved. The unsigned result quantity becomes the complement of the bitwise OR of two wrapped products: multiplier times that saved value, and that saved value times a fresh global coefficient.

A transition location is then read twice. Bit 31 clear in the unsigned result selects the first observation plus three; bit 31 set selects the second observation. The shared call scalar is replaced by the pass-captured processing third argument.

No count, success, or failure meaning is established for the dependency’s return.

### Selection 4: publication

The publication comparison pointer is first copied into the saved comparison pointer. Four word stores follow in this order:

| Destination offset | Value |
|---|---|
| 0 | First publication source times multiplier times scale |
| 4 | Second publication source times multiplier times scale |
| 8 | Third publication source times multiplier times scale |
| 12 | Fourth publication source times scale times multiplier |

Every product wraps to 32 bits. Sources are read as publication proceeds, not snapshotted together. Earlier stores can therefore affect later source reads through aliasing.

Between the third and fourth stores, the fourth source and its companion replace the second working operand and its companion. The fourth source is read again and multiplied by scale to replace the processing first-argument quantity. Multiplying that quantity by the multiplier supplies the fourth store.

After publication, the current first additional pointer field is compared with the publication comparison pointer. Equality selects the pass-transition capture minus six; inequality selects that capture unchanged. The pointer is read after the stores, while the transition value was captured before dispatch.

Publication is not transactional. No rollback is visible.

### Pointer comparisons and pointer-taking operations

**Selection 5** compares the current second additional pointer field with the saved comparison pointer. Equality selects the first observation of its transition location plus one; inequality selects a second observation of that location.

**Selection 11** calls F-5e3309c9d083 with the current first additional pointer field. Afterward, it reads its transition global and installs that field’s pre-dispatch capture as the current indexed-write pointer.

**Selection 14** does the corresponding work for the second additional pointer field.

The pointer passed to the dependency and the pointer subsequently installed are distinct observations. The installed pointer is not reread after the call.

**Selection 15** reads its transition global, then points the current indexed-write pointer at local result storage. These three selections do not directly change the comparison pointers.

### Selections 6, 7, and 12: accumulator preparation

Selection 6 reads its next-state global, then reads another global into the unsigned result quantity.

Selection 7 reads the same next-state source, then sets the unsigned result quantity to scale times multiplier times the saved scaled processing return. This applies scale again to a value already scaled by selection 13.

Selection 12 replaces the accumulator with multiplier times the unsigned result quantity, then reads its transition global. It does not return immediately.

### Default operation and selection 9: indexed-byte preparation

The default operation sets the index to `0x80`, converts the address of the three-word arithmetic array into a 32-bit word, and multiplies it by scale to set the iteration-like value. It then reads a transition global. The iteration-like value is not established as a length.

Selection 9 computes a new shared call scalar from the current index, sampled factors, and coefficient observations. Its arithmetic contract is described through two word-valued contributions:

- The **baseline** is multiplier times scale times index.
- A **biased index quantity** is scale times index times a fresh coefficient, increased by one.
- Two distinct coefficient locations independently supply masks that are OR-combined with that biased quantity.
- Another coefficient location is observed twice. Those observations must not be merged or assumed equal.

The first contribution is the successor of a bitwise-disagreement word. One word being compared is the baseline reduced by the first masked biased quantity and by the first observation of the repeated coefficient. The other is the predecessor of the second observation of that coefficient. Each disagreement bit is one exactly where those two words differ.

The second contribution has a bit-level contract. Consider the predecessor of the baseline. Wherever that predecessor has a one bit, the contribution also has a one bit. At each remaining position, the contribution is the parity of three bits: the corresponding bits of two further coefficient observations and of a candidate word. That candidate word is a separate coefficient reduced by the baseline and increased by the second masked biased quantity.

All arithmetic and bit operations above are 32-bit. The seven coefficient locations participating in these contributions are distinct except for the explicitly repeated observation. Their relative read order within the calculation is unresolved.

The wrapped sum of the two contributions becomes the shared call scalar. Subtracting the baseline from it produces the unsigned result quantity.

Selection 9 next saves a fresh coefficient times the iteration-like value. It then reads one transition location twice: bit 31 clear in the unsigned result selects the first observation minus seven; bit 31 set selects the second observation. Finally, it copies the current indexed-write pointer into the saved byte base.

### Selection 10: byte generation

The index becomes multiplier times the sum of scale times the old index and a fresh global increment. The unsigned result quantity then becomes multiplier times scale times the updated index; this wrapped word is the byte offset.

The shared call scalar becomes a fresh global factor times the difference between the old iteration-like value and a fresh global offset. Relative order of those two global reads is unresolved.

Exactly one byte is written at saved byte base plus the wrapped offset. Its value is the low eight bits of:

\[
\bigl(
\text{scale}\times((\text{saved multiple}-1)\mathbin{\mathrm{OR}}\texttt{0x8b})
-\texttt{0x3b}+\text{old iteration-like value}
\bigr)
\times\text{new shared call scalar}\times(-9).
\]

Only participating low bytes affect the stored byte. The saved multiple starts at zero; selection 10 does not initialize it from another value.

Before the store, selection 10 prepares multiplier times the sum of the old iteration-like value and scale. After the store, scale times that prepared value replaces the iteration-like value. The operation then reads the same transition source as the default operation.

No bounded traversal, contiguous output, character encoding, or cryptographic meaning is established.

## Side effects

Visible potential effects comprise dependency calls, two saved-address-relative word reads, four ordered publication stores, individual byte stores, and updates to invocation-local state.

There is no explicit direct global assignment. Dependencies and aliased writes may nevertheless modify globals or accessible storage. Entry-sampled factors remain sampled; subsequent global observations may see changes.

No ownership transfer, release guarantee, exception handling, iteration bound, timeout, or invalid-state rejection is visible. Passing local storage does not establish permission to retain its address or extend its lifetime.

Fault behavior and dependency-exception outcomes are unresolved. Arbitrary configuration does not guarantee termination.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-5e11a17b9827 | Two word arguments; word return tested for zero and used as an address-like base | Validity, extent, allocation, failure meaning, ownership, lifetime, memory and global effects |
| F-23b828769638 | Five arguments including local storage; arithmetic word return | Writes, initialized fields, retained addresses, return meaning, ownership, memory and global effects |
| F-5e3309c9d083 | One pointer argument; no return consumed | Release behavior, null tolerance, validity requirements, memory and global effects |

No dependency is assumed to preserve globals or accessible storage, initialize undocumented fields, or release memory.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume admissible captures, valid reached accesses, the stated selections, and no unmentioned indirect mutation.

1. **Immediate return.** Selection 0 before selection 12 returns zero. No earlier dependency call or publication is required.

2. **Preparation before setup.** Selection 17 before selection 1 uses zero operands regardless of arguments 3 and 5. Both wide products and processing argument quantities become zero. The arithmetic array’s second and third words become zero; its index is unchanged.

3. **Return preparation before processing.** Selection 7 before selection 13 produces unsigned result zero. Selections 12 and 0 then return zero.

4. **Zero saved multiple.** With saved multiple zero, scale one, old iteration-like value zero, and new shared scalar one, selection 10’s parenthesized byte term is `0xc4`, representing −60 modulo 256. Multiplication by −9 gives low byte `0x1c`. Observing the store additionally requires a valid computed destination. Before any selection 9, the saved byte base remains null unless indirectly changed.

5. **Independent pointers.** Selection 15 changes the indexed-write pointer, not the publication comparison pointer. A subsequent selection 4 still compares against null unless that comparison pointer was indirectly changed.

6. **Shared-scalar replacement.** If selection 13 restores a captured processing argument of seven, selection 2 with factor three passes 21 as its first argument, regardless of an earlier argument-1 product.

7. **Processing predicate returned.** With both sampled factors one and processing return zero, selection 13 produces unsigned result `0xffffffff`. Selections 12 and 0 return that word.

8. **Byte offset returned.** With scale one, multiplier two, old index zero, and increment one, selection 10 produces index two and offset four. Selection 12 then sets the accumulator to eight.

9. **Publication aliasing.** With factors one and stable sources 10, 20, 30, and 40, publication stores those values in order. If the first store changes the second source, the second store uses its changed value.

10. **Truncation and partial effects.** Scale two and working operand `0x80000000` produce zero before widening. A fault at the third publication store has no visible rollback requirement for the first two stores.

11. **Transition timing.** Publication uses its pre-dispatch transition capture. Selection 11 reads its transition after the dependency returns but installs its pre-dispatch pointer capture.

12. **Selection 9 bit boundary.** If the baseline is zero, its predecessor has every bit set, so the second contribution is `0xffffffff` regardless of its three coefficient observations.

## Uncertainties

The following remain **documented_only**:

- Concrete configuration values, reachable sequences, and termination.
- Definedness of the two additional pointer-field captures.
- Full result-storage layout and dependency initialization guarantees.
- Pointer validity, extents, alignment, aliasing constraints, ownership, and lifetimes.
- Dependency success conventions and preservation of memory or globals.
- Intended meaning of the arithmetic predicates and generated bytes.
- Evaluation order not established by explicit sequencing.
- ABI details, endianness, concurrency guarantees, and fault behavior.

Explicitly zero-initialized operands, companions, saved processing return, and saved byte-generation multiple are established state, not unresolved initialization assumptions.

## Evidence

Established claims derive solely from the supplied static representation: explicit initialization, arithmetic widths, dispatch structure, dependency arguments, pass captures, offset reads, ordered stores, transition selection, and return handling.

Resource construction, meaningful decoding, bounded traversal, and cleanup are inferred intent only, not acceptance requirements.

Trusted provenance is supplied separately by the publishing system. No snapshot identifier is reproduced here. No runtime tests have been performed.
