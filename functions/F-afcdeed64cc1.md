<!-- Trusted provenance: F-afcdeed64cc1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afcdeed64cc1

## Purpose

F-afcdeed64cc1 is a state-mutating routine whose application-level purpose is not established. The supplied static representation describes dependency invocations interleaved with indirect byte writes, one 16-bit write, and numerous 32-bit reads and writes. Dependency results and incoming machine-state words supply both addresses and stored values.

The evidence does not establish an allocator, serializer, decoder, renderer, or collection operation. Repeated scaling and bit extraction may suggest an encoded representation, but that interpretation is not a behavioral requirement.

This contract distinguishes:

- **Established static evidence:** accesses, widths, transformations, and ordering explicitly represented in the supplied material.
- **Inferred intent:** interpretations that do not impose acceptance requirements.
- **documented_only:** unresolved behavior that must not be replaced with an assumed implementation.

The representation includes warnings about invalid instruction data, omitted unreachable material, and overlapping global symbols. Its terminal path is truncated. Consequently, the requirements below are conditional on the represented operations being reached and executable; they do not establish that a valid executable traverses the entire sequence.

## Interface

### Explicit inputs

The represented interface accepts four 32-bit arguments:

| Position | Interpretation | Visible use |
|---|---|---|
| First | Uninterpreted 32-bit quantity | No explicit use is represented. |
| Second | Uninterpreted 32-bit quantity | Supplies both the first and second arguments of the initial invocation. |
| Third | Signed 32-bit quantity | Supplies the third initial invocation argument; later participates in a left shift by 25. |
| Fourth | Signed 32-bit quantity | Left-shifted by 3 to supply the fourth initial invocation argument. |

The initial invocation therefore receives the second input twice, the third input, and the shifted fourth input, in that argument-position order.

The apparent absence of an explicit use of the first input does not prove independence from its incoming machine register. Function boundaries and the calling convention are not independently validated.

### Implicit inputs

Additional inputs include:

- An incoming return-link value used to derive the initial invocation target.
- Incoming machine-state words used as addresses and operands.
- Memory reachable through incoming words and dependency results.
- An unsigned byte at address 0x00000001.
- A 32-bit global word at address 0x00000028.
- An unnamed 32-bit global whose address is not supplied.
- Primary, secondary, and paired result words associated with dependencies.

Secondary result words are used even where the represented invocation has no ordinary declared return value. Their calling-convention meaning is documented_only.

The unnamed global has two distinct source-level references in the late sequence: one captures its value, and the other independently supplies a multiplication by four. These are separate inputs to subsequent dataflow, not one established snapshot.

### Outputs and prerequisites

No explicit return value is declared. Observable outputs consist of memory accesses, dependency invocations, and dependency effects. Normal completion is not established.

Each reached memory access requires a valid location for its width and direction. Each reached invocation requires an executable target and a compatible calling convention. No visible null, range, alignment, ownership, or target-validity check establishes these prerequisites.

## Behavior

### Widths and arithmetic

The represented scalar and address arithmetic is predominantly 32-bit. Byte reads produce unsigned values from 0 through 255. Byte stores retain the low 8 bits; the halfword store retains the low 16 bits. Word accesses are 32-bit.

Unsigned right shifts are logical. Several left shifts, multiplications, and address expressions use signed intermediates. Negative operands and signed overflow cannot safely be specified using portable signed-arithmetic semantics. Examples involving low-word truncation are conditional on that target interpretation.

Address calculations must not be silently widened to unlimited precision. Conversely, target wraparound is not independently verified.

Visible transformations include:

- Logical right shifts extracting portions of 32-bit words.
- Left shifts and multiplication by powers of two.
- Masks retaining 6, 22, 29, 30, or 31 low bits.
- Subtraction without a visible underflow check.
- Separation of represented 64-bit results into low and high 32-bit words.

A mask, shift, and narrowing store are distinct transformations. For example, retaining 30 low bits and then shifting right by 20 can produce a 10-bit value; a subsequent byte store discards its upper two bits.

One late byte value comes from a logical right shift of a represented 64-bit result by 54. This corresponds to the high 32-bit word shifted right by 22, followed by byte truncation.

### Initial sequencing

The represented body is linear, without an explicit input-dependent branch or loop. Dependencies may nevertheless branch, fail, or fail to return.

The initial invocation precedes every visible memory write. Several further invocations follow, including one receiving an incoming unsigned word shifted right by 10.

The first visible memory mutation reads a byte one byte beyond an incoming implicit address. That byte becomes a numeric destination component: the routine writes at the selector value plus 17. The stored byte comes from another incoming unsigned word shifted right by 22.

This is not a write 17 bytes beyond the original implicit address.

Later invocations receive transformed words, repeated arguments, zeros, and earlier results. Invocations represented without explicit arguments must not be assumed independent of machine state.

### Selector-directed byte writes

A recurring operation reads an unsigned selector byte and writes one byte at the numeric address equal to that selector plus 17. Its destination is therefore between 0x11 and 0x110 inclusive.

The representation does not establish an object base associated with these destinations. They are direct low-address accesses, not proven object-relative fields.

Selector sources include:

- An implicit address plus 1.
- A dependency-derived word scaled by 4, 8, 16, 64, 1,024, 4,096, or a larger power of two, plus 1.
- A word shifted right by 9, 11, 12, or 21, plus 1.
- A masked-and-shifted word plus 1.
- Address 0x00000001.

Stored bytes include zero, low bytes of scaled results, captured word bytes, and shifted bit slices.

Each represented selector reference occurs at its own point in the sequence. Dependencies or earlier writes may change the selected memory. Repeated selector accesses must not be replaced with one entry-time observation.

Some selectors are retained and passed to later invocations. Others are referenced again after intervening calls. Retention of a selector value does not imply preservation of its source memory.

### Word-access families and captured values

Many word accesses use a scaled quantity plus 20. Scaling by 4 and scaling by 16 are distinct address families; one does not generally initialize the other.

Early word operations include:

- Storing zero at a secondary-result-derived address scaled by 4 and increased by 20.
- Storing a secondary result at a primary-result-derived address scaled by 4 and increased by 20.
- Storing a four-times-scaled result at an address formed by scaling that result by 16 and adding 20.
- Reading a word through the differently scaled, four-times-derived address.
- Storing a right-shifted portion of that captured word and using another portion to select a further word read.

Later chains repeat analogous address relationships with different captured values. Their repetition does not establish a common record type or valid index range.

A dependency result is written as a 32-bit word at address 0x0000007c before an invocation receives that same captured result twice.

#### Global word at address 0x00000028

This global participates in three distinct stages:

1. Its value is captured as a 32-bit scalar.
2. The immediately following invocation has two separate argument expressions, each referencing the global at address 0x28.
3. After that invocation and a further invocation without explicit arguments, the earlier captured scalar is stored at an address formed from a secondary result scaled by 4 and increased by 20.

The two argument expressions reference the global, not the previously captured scalar. No evaluation order between those two expressions is established. The source-level references do not independently establish a machine-load count or whether their observed values must agree.

Changes to address 0x28 during intervening dependencies do not replace the scalar already captured for the later store. Conversely, the capture does not authorize substituting that scalar for either global-reference argument.

#### Two observations of the unnamed global

Near a late word-store chain, a dependency result is first stored at an address derived from a secondary result scaled by 4 and increased by 20. The representation then:

- Captures the unnamed 32-bit global.
- Separately references that global again and scales this second observation by four.
- Proceeds to the next invocation, which receives the preceding secondary result.

Both global references precede that next invocation, and the capture precedes the separately scaled reference. They are not one established observation. Their machine-load count, memory attributes, and equality are not independently known.

The two resulting dataflows remain distinct:

- **Captured observation:** later masked to retain 30 low bits and logically shifted right by 10 to supply an invocation argument.
- **Separately scaled observation:** supplies a selector-read address increased by 1. After intervening invocations, that address is used for another selector reference. The same retained scaled quantity also undergoes a left shift by 26; the resulting address supplies two separate word-reference argument expressions at offset 28. A further multiplication by 1,024 supplies later selector addresses increased by 1 and subsequent invocation operands.

The offset-28 argument expressions have no established evaluation order relative to each other. They are not a proven single shared read.

Dependencies between these uses may modify the unnamed global or selected memory. The represented computations nevertheless use the respective retained scalars unless an explicit fresh reference is shown.

Under a conditional 32-bit low-word interpretation, left-shifting a word by 26 and then multiplying by 1,024 discards all remaining bits. That conditional arithmetic fact does not establish valid signed-overflow behavior or justify erasing the preceding address transformations and word references.

### Paired results and indirect layouts

Several results are represented as 64-bit containers whose low and high 32-bit portions are consumed separately. This establishes two-word dataflow, not necessarily an actual 64-bit numeric return convention.

One high result word supplies a base for consecutive 32-bit reads at offsets 8 and 12. The offset-12 word later supplies a selector source at offset 1; the low byte of the captured offset-8 word is written through that selector. The two word captures precede intervening invocations; the selector access occurs later.

Other represented relationships include:

| Relationship | Observable use |
|---|---|
| Earlier low result plus 4 | Source of a captured 32-bit word whose low byte is later written through a selector. |
| Low result plus 12 | Passed repeatedly to an invocation; a later word read uses four bytes beyond this retained adjusted address. |
| High result plus 28 | Destination of a byte containing the low result’s low byte. |
| Transformed address plus 28 | Source of two word-reference invocation arguments in the unnamed-global chain. |

The two offset-28 cases are not established as the same object or field.

### Zero-derived and repeated writes

A retained value explicitly set to zero later produces:

- A 32-bit store at address 0 through scaling by 8.
- A separate 16-bit store at address 0 through masking and right shifting that retained zero.
- A byte destination whose scaled-zero contribution vanishes, leaving a location derived from another result.

The word and halfword stores are separate events with intervening accesses and invocations.

A late sequence writes two 32-bit values consecutively to an address formed from a shifted earlier result plus 52: first the zero-extended retained selector byte, then twice that selector. There is no intervening invocation.

Another pair writes the same four-times-scaled result twice at a secondary-result-derived address plus 52. The representation contains two write events even though the values agree.

### Later word chains and termination

The final portion contains repeated stores into both four-times-scaled and sixteen-times-scaled address families, often followed by invocations consuming the four-times-scaled quantity.

One invocation contains two separate references to a word at the four-times-scaled address plus 20: one supplies a right-shifted argument, and another supplies a left-shifted argument. These are not established as a single captured word. Elsewhere, a word at that address family is explicitly captured before intervening calls and later stored after a right shift by 12.

The terminal section again alternates invocations and selector-directed byte writes. The supplied representation ends at an invalid-instruction termination marker.

No normal return, success result, cleanup, rollback, or error translation is visible. The marker does not independently identify an actual processor exception or prove that execution reaches it.

## Side effects

### Memory and aliasing

Visible accesses include 8-bit, 16-bit, and 32-bit writes and 8-bit and 32-bit reads. No atomicity, synchronization, or transaction boundary is established.

Computed addresses may alias fixed-address globals, other computed locations, later selector sources, or dependency-accessed storage. Byte and halfword writes may overlap wider words. Their effect on enclosing values depends on unestablished endianness and memory attributes.

Repeated accesses remain distinct in the represented sequence. Their visibility to an external observer depends on whether the locations are ordinary memory, device memory, or another memory class.

### Ownership and lifetime

No allocation, release, ownership transfer, reference counting, or lifetime guarantee is established.

Passing a word to F-23325cd5e50c or another dependency does not prove that it is a pointer or that ownership changes. Reuse after a call does not prove that the dependency preserves referenced storage, globals, or registers.

### Failure and partial state

No visible validation or recovery path handles invalid accesses, incompatible targets, dependency failure, or arithmetic problems. Earlier writes and dependency effects may remain if a later operation fails.

The routine does not establish all-or-nothing behavior.

## Dependencies

Mapped direct invocations occur in this relative order; other invocations intervene:

| Dependency | Explicit argument relationship |
|---|---|
| F-23325cd5e50c | One captured 32-bit result. |
| F-bbb54b5f036d | The same captured word twice. |
| F-f80b3ac065fc | Low result word, high result word, low word again, low word again. |
| F-de15789688a8 | The same captured word twice. |
| F-ea83f9a17404 | The same captured word twice. |
| F-674e75a953e8 | The same captured word twice. |

Numerous additional invocations use computed targets. Some depend on incoming machine state; others are formed relative to represented instruction positions. Two additional direct targets have no supplied opaque IDs.

Their identities and contracts are documented_only. Similar target calculations do not justify combining invocations into a single dependency. No global preservation, memory ownership, error convention, or result interpretation is assumed.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes that the relevant operations are reached and their accesses and invocations are valid.

1. **Initial argument positions.** With second input 7, third input 9, and fourth input 3, the initial invocation receives 7, 7, 9, and 24.

2. **Selector destination.** A selector value of 0x80 directs its byte write to 0x91, not to the selector-source address plus 17.

3. **Byte narrowing.** Logically shifting 0xffffffff right by 22 produces 0x3ff; a byte store writes 0xff.

4. **Mask before shift.** Retaining 30 low bits of 0xffffffff and shifting right by 20 produces 0x3ff. A following byte store writes 0xff.

5. **Different scaling.** For a result of 2, sixteen-times scaling plus 20 selects address 52; four-times scaling plus 20 selects address 28. A store to the former does not establish the latter’s contents.

6. **Captured global versus invocation operands.** If the capture from address 0x28 yields 0x12345678, the later computed-address store uses 0x12345678 despite an intervening dependency changing that global. The intervening invocation must still be described as having two global-reference operands, not two uses of the captured scalar.

7. **Independent unnamed-global observations.** If the first observation yields 0x00123400 and the separate scaling observation yields 3, the masked-and-shifted argument is 0x48d, while the retained scaled quantity is 12. Its associated selector source is address 13. This example does not assert that differing observations occur in ordinary stable memory.

8. **Fresh selector content.** If a dependency changes the byte at the unnamed-global-derived selector source between its two represented references, the later write uses the later selector. The retained address does not imply retained byte content.

9. **Doubled selector word writes.** A selector of 200 produces consecutive 32-bit stores of 200 and 400 at the relevant offset-52 destination. The second value is not byte-truncated.

10. **Identical repeated stores.** The repeated identical word-store pair represents two access events, conditional on both executing. Whether an observer distinguishes them depends on memory semantics.

11. **Zero-address failure.** If address 0 is inaccessible, the zero-derived word store cannot be treated as a successful write elsewhere. No recovery path is supplied.

## Uncertainties

The following remain documented_only:

- Application purpose, recovered boundary, and actual calling convention.
- Meaning of secondary and paired results.
- Validity and reachability of computed invocation targets.
- Signed-overflow behavior and actual address-wrap semantics.
- Endianness, alignment, memory attributes, and access atomicity.
- Whether low addresses are valid storage, special-purpose regions, or faults.
- Machine-load counts and evaluation order for repeated argument expressions.
- Equality of separate global observations and possible asynchronous mutation.
- Dependency effects on globals, storage, ownership, and lifetime.
- Executability of the represented linear path.
- Actual terminal behavior and normal-return paths outside the supplied material.

A possible encoded-state interpretation remains inferred intent only; it establishes no layout, tag scheme, or ownership rule.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. Prior prose and reviewer statements are not independent evidence.

The representation supports explicit widths, argument-position relationships, captured-versus-fresh dataflow, address transformations, repeated accesses, and represented ordering. In particular, it distinguishes the capture at address 0x28 from the following invocation’s two global-reference operands, and distinguishes the unnamed global’s capture from its separate scaling reference.

Source-level references do not independently prove machine-load counts. Warning annotations limit confidence in execution and control flow rather than establishing runtime outcomes.

No runtime tests, execution traces, dependency inspection, or independent calling-convention validation have been performed. Acceptance examples are conditional observations, not verification results.
