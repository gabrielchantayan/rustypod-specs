<!-- Trusted provenance: F-b635fbda3c01; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b635fbda3c01

## Purpose

F-b635fbda3c01 performs an ordered sequence of dependency calls, bit-field transformations, and 8-, 16-, and 32-bit memory accesses. Its effects depend on four explicit input words, implicit entry-register state, dependency results, auxiliary register outputs, and mutable memory.

No application-level purpose is established. Allocation, serialization, initialization, dispatch, and service-management interpretations are not acceptance requirements.

This contract distinguishes:

- **Established static evidence:** widths, constants, data dependencies, and ordering visible in the supplied representation.
- **Inferred intent:** possible interpretations without sufficient evidence to require them.
- **documented_only:** unresolved behavior that a replacement must not invent.

The represented caller has no explicit conditional branch or loop. Nevertheless, every later effect is conditional on preceding calls and memory accesses completing.

## Interface

### Explicit inputs and completion

The interface accepts four opaque 32-bit words. The first dependency receives all four unchanged, in their original order. No subsequent direct use of the original arguments is established.

There is no explicit return value. Normal completion is represented after a final call to numeric address zero; it requires that call, and every preceding operation, to return or complete successfully.

Dependency results include opaque words, 64-bit quantities split into low and high 32-bit halves, and byte-sized values. Additional addressing and arguments depend on auxiliary register outputs not explained by ordinary return signatures.

### Implicit inputs

Required execution context includes:

- An incoming control-address word used to derive the first call target.
- An incoming address used for early selector-byte reads.
- An incoming byte used by the first byte write.
- Readable and writable ambient memory, including low numeric addresses.
- Dependency-produced auxiliary register state.
- Accessible stack storage whose address is exposed to a dependency.

Supplying only the four explicit words does not establish an equivalent execution environment.

### Widths and address terminology

A **selected-byte update at base A** consists of an unsigned 8-bit read at A + 1, followed by an 8-bit write at numeric address 17 plus that read value. The destination is therefore within **17–272**. It is not A + 17.

A **word slot indexed by I** is the 32-bit location at 4I + 20. A **scaled record slot indexed by I** is the 32-bit location at 16I + 20. These names describe formulas, not proven arrays or object layouts.

Unsigned word shifts are logical. Narrow byte writes retain the low eight bits. Splitting a 64-bit result into numerical halves does not establish memory byte order.

Word calculations are represented at 32-bit width. Some scaling and left shifts involve signed-looking operands; their overflow semantics are **documented_only**. Bit-vector consequences below are identified where relevant rather than silently assuming unbounded, checked, or saturating arithmetic.

## Behavior

### 1. Entry and initial selector writes

The first target is derived from the incoming control address by adding 4094 and receives the four explicit arguments. A subsequent dependency supplies a 64-bit result.

The first selected-byte update uses the incoming implicit address and stores the incoming implicit byte. A following call receives, in order, the result’s high word twice, zero, and its low word shifted right by 19.

After intervening calls, the function clears a selected byte using the incoming address again, then forwards a returned word shifted left by two. The selector is freshly read; the initial selector is not retained.

Further calls forward another left-shifted result twice. An unsigned result W then supplies a retained base:

**floor(W / 2¹⁸) × 1024.**

W is passed in four argument positions. Two later selected-byte clears use this retained base, with calls between them. Another call in this region receives two copies of a left-shifted result and an explicit third argument of zero.

### 2. Low-memory input and auxiliary-state clears

Four argument expressions reference the 32-bit location at numeric address **48**. They must not be replaced by an assumed immutable configuration constant. The representation does not settle the physical number or ordering of reads among those argument expressions.

Two selected-byte clears use an earlier auxiliary output logically shifted right by nine. They occur at distinct points separated by calls.

Additional calls forward shifted results, sometimes duplicated. A later invocation receives four explicit zeros.

### 3. Initial slot construction: distinct store and load addresses

An auxiliary word is stored unchanged in the word slot indexed by that same word.

A subsequent invocation receives that auxiliary word and a freshly loaded word shifted right by 19. This load uses a different retained unsigned result W to form:

**floor(W / 2¹⁸) × 1024 + 20.**

Later, a returned index selects a word slot into which the retained auxiliary word is copied.

For the next returned index **I**, the function retains **S = 4I** and stores S at **16I + 20**. The immediately following argument load is explicitly from **S + 20**, and its loaded unsigned word is shifted right by 19. The call receives:

- S;
- the word read at S + 20, shifted right by 19;
- zero;
- S again.

The load is **not** from the word slot indexed by S, which would be 4S + 20. Nor may it generally be satisfied by the preceding store at 16I + 20.

After further calls, S is copied into another returned-index-selected word slot. No immutability of slot contents across dependencies is assumed.

### 4. Field extraction and mixed-width input

A subsequent unsigned result W supplies:

- W shifted left by two;
- its low 30 bits shifted right by 20;
- its low 30 bits shifted right by 17;
- its low 30 bits shifted right by seven.

The first two quantities supply separate selected-byte writes, narrowed to eight bits. Both use a retained auxiliary base shifted right by nine, but each rereads its selector.

A seven-argument invocation receives four copies of another returned word, the right-by-17 and right-by-seven fields, and an opaque code-derived word. The final argument is not established as an application-data pointer.

A later 64-bit result supplies a retained low word L and duplicated high-word arguments. Two selected-byte clears use the earlier right-by-seven field as base.

Subsequent operations include forwarding L shifted right by 28, clearing selected bytes using L shifted right by nine, reading an unsigned byte at 2L, and passing that byte four times. Another clear uses the byte at numeric address 1 as selector.

One argument shifts a word left by 31 after that word has already been shifted left by two. Under 32-bit bit-vector semantics this argument is zero, not a general sign flag.

### 5. Parity-dependent addressing, stack exposure, and halfword input

A later 64-bit result supplies low word L. Let E be L logically shifted right by 19. Following intervening calls and selector updates, a 32-bit read uses:

**L + (E shifted left by 31).**

Under 32-bit bit-vector arithmetic, the added term is zero or 2³¹ according to E’s low bit. The address cannot generally be simplified to L.

Further reads and writes include:

- A byte read at auxiliary address + 14, forwarded four times.
- Selected-byte clears using a retained result multiplied by 4096.
- Clears using auxiliary addresses directly and shifted right by nine.
- A call receiving the same stack-local address four times.
- A clear whose selector address depends on an extracted field multiplied by −2³¹.
- An unsigned byte read at a retained low word plus an auxiliary output.
- A 16-bit read at an auxiliary word shifted right by 19, plus that unsigned byte.

The halfword is forwarded in four argument positions. Its application meaning and any signature-dependent argument extension are unresolved.

The following 64-bit result contributes a retained low word and a high-word slot index. The function stores the preceding right-by-19 auxiliary extraction into the high-word-selected word slot, and later into another selected word slot.

The stack-address call establishes exposure of temporary storage only. Initialization, accessible extent, ownership transfer, and permission to retain the address are not established.

### 6. Offset-13 writes and scaled propagation

Using the retained low word as base B, the function writes the low byte of 2B at **B + 13**. After a word load and calls, it writes byte value **4** at B + 13.

Both writes are observable. The earlier write cannot be eliminated merely because the later write uses the same address.

This region also includes:

- A 32-bit read at 4W + 68.
- A later 32-bit read at the low 30 bits of W shifted right by 20, plus 68.
- A 32-bit store at 256J + 20 containing 4W.
- A later word-slot store containing J’s low 26 bits shifted right by 13.

Several subsequent groups store four times a returned index into its scaled record slot, then copy that scaled value into another selected word slot. Associated call arguments include shifts left by 24 and 22. These patterns do not establish an allocator or linked-record protocol.

### 7. Offset-36 state and offset-116 base replacement

For a retained scaled base B, the function first writes a zero word at **B + 36**, then reads a word at **B + 116**. The loaded word becomes a new retained base N.

A later unsigned dependency result supplies field E by shifting right by 20. After calls, E is stored at N + 36. A fresh word read at N + 116 is passed to a dependency.

Selected-byte writes use N and, later, E × 1024. Calls remain interleaved with these accesses.

Three separate byte reads at offset **21** from dependency-produced auxiliary addresses occur in this region. One byte is later stored zero-extended into a 32-bit word slot. The reads cannot be merged or assumed equal.

E is at most 4095. Consequently, E logically shifted right by 21 is zero, independent of memory changes by dependencies.

### 8. Additional fields, slots, and explicit global clearing

Another auxiliary word supplies field E from its low 29 bits shifted right by 18. The quantity **E shifted left by 13** is both forwarded and stored in a selected word slot. The stored quantity is not the unshifted field.

A later returned word supplies a selected-byte write from its left shift by two and another from its low 30 bits shifted right by 20.

Further propagation includes:

- Retaining an auxiliary word shifted left by three, then shifting it left by two, then left by two again for a later store.
- Storing a scaled word’s low 30 bits shifted right by 19.
- Fresh word-slot reads shifted right by 21.

A distinguished 32-bit global location, **state word A**, is explicitly cleared to zero.

This clear does not establish the values of later reads of A. Intervening dependencies and possible memory aliasing remain relevant.

### 9. Low-address processing and byte-derived reads

The next region combines zero-valued word-slot stores with further scaled-index propagation. A directly designated dependency receives the same returned word twice.

A retained local arithmetic quantity is explicitly zero and remains zero along the represented local data flow. Several selector-address calculations therefore reduce to numeric address 1 without assuming global preservation.

Distinct reads include:

1. A byte at a returned address + 1.
2. A byte at one plus the numerical value of that previously read byte.
3. A later byte at numeric address 1.

The second address uses an eight-bit intermediate value; it is not a full-width pointer-table lookup.

Other extractions are necessarily zero because an earlier mask and shift already bounded their inputs. A low-30-bit quantity shifted right by 19 is at most 2047; shifting that result right by 22 yields zero.

### 10. Offset-9 writes and dependent word reads

After additional calls and slot stores, the function performs:

- A selected-byte write using a returned byte shifted left by two.
- A direct byte clear at a retained scaled address multiplied by 8192, plus 9.
- A byte write at another retained base + 9 containing that base’s low byte.
- A 32-bit read at that base + 52, forwarded as an argument.

Subsequent stores are interleaved with dependent 32-bit reads involving offset 20, right shifts by 22, and halving previously loaded words. These are separate mutable-memory observations, not one immutable lookup.

A later read of state word A is shifted right by 23. That retained extraction is subsequently shifted right by 23 again, necessarily producing zero for a 32-bit unsigned input. This conclusion does not require A to have remained zero.

### 11. State word B and offset-116 narrowing

A returned word is assigned to another distinguished 32-bit global location, **state word B**, before being forwarded to a dependency.

A selector byte previously read from numeric address 1 is retained as a numerical base Q. The function reads a full word at **Q + 116**, then writes its low byte at **Q + 1**. Q is not reread for that operation.

Later operations similarly read words at scaled bases + 116 and write their low bytes at those bases + 1. One retained base undergoes this pair twice, with calls between the pairs. The two full-width reads remain distinct.

Further stores propagate an entire offset-116 word into selected slots, write a shifted returned word through a retained four-byte-aligned address + 20, and clear another selected word slot.

A fresh read of state word A is retained as an address base. After calls and stores, a word is loaded from that retained base + 20. It supplies later selected-byte writes, first narrowed directly and later shifted right by 22 before narrowing.

### 12. Terminal effects

The visible tail, after further calls:

1. Obtains a byte-sized dependency result.
2. Writes zero through a dependency-produced byte address.
3. Assigns state word B a 32-bit quantity whose low eight bits are the result byte and whose upper 24 bits come from a referenced 24-bit memory quantity associated with numeric address **9**.
4. Writes a zero halfword at the retained low word’s low 30 bits shifted right by seven.
5. Calls numeric address **zero** with three explicit zero arguments.

The 24-bit contribution is not established as the preserved upper portion of B. Its storage relationship to other references remains unresolved.

## Side effects

Direct effects comprise byte writes, one terminal halfword clear, 32-bit slot writes, and assignments to state words A and B. Dependencies may have additional, unbounded effects.

Repeated selector reads, word reads, intermediate writes, and dependency calls are observable ordering boundaries. No selector or loaded word may be treated as stable across intervening calls without an external dependency contract.

Computed locations may alias each other or the distinguished globals. Nothing establishes atomicity, synchronization, rollback, allocation, release, or ownership transfer.

No explicit null, bounds, or alignment checks are visible. Completion requires every target and computed address to be accessible with the required operation and width. Fault behavior, partial progress after faults, and recovery are **documented_only**. No status-based error protocol is established.

## Dependencies

The supplied identity map provides no authoritative opaque IDs for callees. The following are **contract-local opaque boundary handles**, not claims of resolved function identity:

- **F-dep-001:** the first, entry-control-derived invocation.
- **F-dep-002:** the directly designated invocation receiving a word twice.
- **F-dep-003:** the terminal invocation at numeric address zero.

Other dependency sites remain unresolved and must not be equated merely because they have similar argument lists. Targets are predominantly computed from code-derived values, with several distinct target offsets.

Complete signatures, auxiliary-register production, global preservation, memory effects, and ownership behavior are **documented_only**. A nonreturning dependency prevents subsequent caller operations.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Input order:** If F-dep-001 is callable, it receives the four supplied 32-bit words unchanged and in order.

2. **Selected destination:** A selector read of 255 directs the associated byte write to numeric address 272, not to a field relative to the selector’s base.

3. **Fresh selection:** If one update reads selector 3 and a later update reads 9, their destinations are 20 and 26 respectively.

4. **Distinct scaled store and load:** In Behavior §3, let I = 3, so S = 12. The store writes 12 at address 68. The following argument load reads address **32**. If the word there is 0x00180000, the call arguments are **12, 3, 0, 12**. Reading address 68 instead would incorrectly substitute the preceding stored value.

5. **Byte truncation:** A word value 65 shifted left by two supplies stored byte 4.

6. **Intermediate write:** If both offset-13 writes complete, the low byte of twice the retained base is written before the later value 4, with the intervening load and calls preserved.

7. **Shifted field storage:** If the field extracted from the low 29 bits is 3, Behavior §8 stores 24576, not 3.

8. **Global mutation:** An intervening dependency may change state word A after its clear. Later reads must observe memory at their actual read points, not a substituted constant zero.

9. **Terminal completion:** If F-dep-003 traps, no normal return follows. If it returns, and all preceding operations complete, the represented return is reachable.

## Uncertainties

The following remain **documented_only**:

- Application purpose and meanings of the fields and slots.
- Resolved dependency identities and complete calling conventions.
- Sources and preservation rules for auxiliary register outputs.
- Address-space mapping, alignment, byte order, and overflow semantics.
- Physical read multiplicity for repeated argument expressions.
- Aliasing among globals, low-memory references, and computed stores.
- Stack-storage extent, initialization, and permitted retention.
- Accessibility and meaning of address zero.
- Whether all represented operations correspond to executable behavior on the intended platform.

Encoded-field processing and structured-state propagation are possible intent, not established requirements.

## Evidence

The evidence is the supplied static representation: explicit access widths, constants, argument relationships, retained values, masks, shifts, and represented operation order.

For the corrected slot distinction, the representation separately establishes a scaled store at 16I + 20 and a subsequent load through the retained scaled value S at S + 20. No dependency contract is needed to distinguish those address expressions.

Notices about removed unreachable material and overlapping global symbols limit conclusions about complete control flow and storage identity. They do not establish missing behavior.

No dependency inspection, implementation access, runtime execution, smoke run, or tests have been performed. Acceptance examples state conditional obligations derived from static evidence, not observed outcomes.
