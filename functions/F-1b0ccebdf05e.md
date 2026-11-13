<!-- Trusted provenance: F-1b0ccebdf05e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b0ccebdf05e

## Purpose

F-1b0ccebdf05e mutates memory through incoming addresses, follows links whose contents may change during execution, performs arithmetic-addressed writes, and invokes an unresolved indirect dependency with three arguments.

The established behavior is an ordered memory-access trace. The representation does not establish a list operation, object lifecycle, buffer transformation, or other application-level purpose. Link offsets, marker values, and self-address stores suggest structured-state manipulation; that interpretation remains inferred intent.

No return value is represented. If the indirect dependency returns, the represented function returns without a result. Whether that dependency actually returns is unknown.

## Interface

### Inputs and incoming state

| Role | Represented width | Use |
|---|---:|---|
| First explicit input | 8 bits | Byte stored at A + `0x1c`. |
| Second explicit input | 32 bits | Full-width stores and low-byte stores. |
| Third explicit input | 32 bits | Stored at C + `0x54`. |
| Fourth explicit input | 8 bits | Stored at a subsequently loaded address + `0x11`. |
| A, B, C | 32-bit address-bearing values | Incoming bases not represented as explicit parameters. |
| D | 32 bits | Incoming value stored both whole and narrowed. |

A, B, C, and D are documentation roles, not source identifiers. Their calling convention, initialization, and relationships to explicit inputs are unknown.

Three external storage locations supply 32-bit values, named **X**, **Y**, and **Z** here. Their storage identities and mutability are not established. They are read at separate specified points, not captured together on entry.

### Memory model and notation

Offsets are in bytes. Offset `0x64` is decimal 100; it is not `0x74`.

Unless explicitly described as a byte or halfword access, every memory read or write below is 32 bits. A halfword is 16 bits. “Low byte” and “low halfword” mean the least significant eight and sixteen bits respectively.

Named captures retain the value obtained at the specified read. Reusing a captured address does not reread the location from which it came. Conversely, separately identified reads remain separate even when they address the same location.

Addresses and arithmetic are represented using 32-bit quantities. Actual overflow behavior, address validity, byte order, alignment requirements, and fault handling are not established. In particular, the signedness of the byte read near the end requires a target convention; both interpretations are documented below.

### Preconditions and ownership

Completion requires every reached address to permit its indicated access. This includes addresses formed by subtraction, doubling, adding two captured values, and adding marker constants to a byte-derived value.

No validation of null addresses, region bounds, alignment, lifetime, ownership, or link consistency is represented. Different documentation roles may designate overlapping memory.

No allocation or release is represented. Memory mutation alone establishes neither ownership transfer nor permission to release any region.

## Behavior

### Ordering rule

Within each phase, the following prose gives memory-access order. The phases also occur in order. Capturing or narrowing an already loaded value introduces no additional memory access.

Earlier writes may change subsequent reads through aliasing. The contract therefore cannot be reduced to final field values or traversal of an entry-time snapshot. Repeated stores, overlapping stores, and read-and-write-back operations remain part of the represented trace. This does not establish volatile semantics, atomicity, or synchronization.

### 1. Incoming-state propagation

The initial byte writes occur at B + `9` using D, A + `0x1c` using the first input, B + `0x11` using the second input, and C + `0x11` using B, in that order.

The function captures **Initial payload** from A + `0x50`. It then writes D at B + `0x44`, the second input at B + `0x64`, and the second input at B + `0x34`. After writing the second input’s low byte at A + `0x11`, it captures **Initial marker** from B + `0x74`.

Initial payload is then written at A + `0x10`, followed by the third input at C + `0x54`. Initial payload’s low byte is written at A itself, followed by B’s low byte at A + `9`.

The address **Primary** is captured from B + `0x14`. Crucially, the next write places A’s low byte at C + `0x11`; only afterward is Primary written at A + `0x44`. Initial payload’s low byte is written again at A itself.

The function then captures **Back-reference** from C + `0x74` and **Secondary** from C + `0x14`, in that order, before writing A’s own address at A + `0x54`.

The earlier byte write at A + `0x11` overlaps the later full-width write at A + `0x10`. Neither can be omitted merely because a later access replaces its bytes.

### 2. First marker-100 mutations

An address read from Secondary + `4` identifies the first destination. That destination receives Initial marker’s low byte at `0x11`, then byte 100 at `0x10`. Primary + `0x54` receives full-width 100 **before** the first destination receives full-width 100 at `0x60`.

The current address at Primary + `0x74` is captured, Primary + `0x74` is replaced with Secondary, and the captured destination receives 100 at `0x30`.

The function captures **Early link** from Secondary + `0x14`. Early link + `9` receives Initial marker’s low byte, and Primary + `0x54` receives Primary itself. Next, the address Back-reference minus `0x20` receives the value Back-reference minus `0x74`.

An address is read from Early link + `0x14`. After the fourth input is written as a byte at Secondary + `0x11`, **Link image** is read from Secondary + `0x14`. Its low byte is written at the just-reached address + `0x15`.

Secondary + `0x14` is read again, yielding **Reached destination**. This capture need not equal Early link or Link image.

### 3. Intermediate links and marker `0x74`

The first value read from Primary + `0x64`, **Marker capture A**, is immediately written back there. Reached destination + `0x0c` receives byte `0x73`, followed by Marker capture A’s low byte at Primary + `0x15`.

The next accesses have this exact relative order:

1. Link image is written at Secondary + `0x34`.
2. **Early reached link** is read from Reached destination + `0x74`.
3. Reached destination’s own address is written at its `0x54`.
4. Primary’s low byte is written at Secondary + `0x0d`.
5. Link image is written at Secondary + `0x54`.
6. Reached destination is written at Secondary + `0x14`.
7. **Marker capture B** is read from Primary + `0x64`.

Reached destination + `0x10` then receives byte `0x74`; Primary + `0x54` receives full-width `0x74`; Reached destination + `0x10` receives full-width `0x74`.

A new read of Reached destination + `0x74` captures **Later reached link**. Secondary + `0x54` is replaced with Secondary itself, and Reached destination + `0x30` receives `0x74`.

The next writes place Marker capture A’s low byte at Primary + `1`, Marker capture B’s low byte at Marker capture A + `9`, and Marker capture B at Primary + `0x74`. Primary + `0x11` then receives byte `0x74`.

After reading **Reached payload** from Reached destination + `0x60`, the function replaces Primary + `0x11` with Marker capture A’s low byte, writes Marker capture B at Primary + `0x34`, and writes Reached payload’s low byte at Reached destination + `0x10`.

Early reached link and Later reached link are separate captures. Their equality is not guaranteed.

### 4. Marker `0x66` and replacement of Primary + `0x44`

The function reads **Replacement destination** from Primary + `0x14`. It writes Early reached link’s low byte at Secondary + `0x11`, Secondary at Primary + `0x44`, and Marker capture B’s low byte at Primary + `0x11`.

It captures **Pre-marker link** from Primary + `0x74`, then writes Secondary’s low byte at Primary + `0x0d`.

Replacement destination + `0x10` receives full-width `0x66`. Primary + `0x44` receives Secondary again, followed by byte `0x66` at Replacement destination + `0x10`, full-width `0x66` at Primary + `0x54`, and full-width `0x66` at Replacement destination + `0x60`.

A fresh Primary + `0x74` read captures **Marker destination**. Primary + `0x74` is replaced with Secondary, Marker destination + `0x40` receives `0x66`, and Primary + `0x0d` receives Pre-marker link’s low byte.

After capturing **Marker payload** from Marker destination + `0x44`, the function writes Primary at Primary + `0x54`, then Later reached link at Later reached link + `0x54`.

An address read from Marker destination + `0x14` identifies the next byte destination. Secondary + `0x11` receives Marker payload’s low byte. A new read from Secondary + `0x14` supplies the low byte written at that destination + `0x15`.

Secondary + `0x14` is then read again, yielding **Traversal base**. The function reads **First nearby capture** from Primary + `0x64`, writes `0x73` at Traversal base + `0x20`, and reads **Second nearby capture** from Primary + `0x64`.

Second nearby capture’s low byte is written at Traversal base + `9`. A read from Traversal base + `0x74` yields **Further destination**.

Further destination receives Primary’s low byte at `0x0d`. Its `0x54` is read and written back unchanged, and its offset `9` receives byte `0x74`. **Only then** does Primary + `0x44` receive First nearby capture. Further destination + `0x34` subsequently receives Primary.

This replacement of Primary + `0x44` supersedes the earlier stores of Secondary unless later aliasing changes it. A subsequent traversal explicitly rereads that field.

### 5. Late propagation through mutable links

An address read from Further destination + `4` yields **Continuation**. Continuation + `0x24` receives Second nearby capture, and Primary + `0x19` receives First nearby capture’s low byte.

The next reads capture **Retained link** from Continuation + `0x74`, **Replacement payload** from Continuation + `4`, and **Current head** from Primary + `0x74`, in that order.

Current head + `0x14` is read to obtain a destination whose `0x54` receives Retained link. Current head + `4` is then read to obtain another destination.

Primary + `0x0d` receives Second nearby capture’s low byte twice consecutively. Primary + `0x64` receives Replacement payload, and the last destination + `0x30` receives `0x74`.

Current head + `4` is read again. Reading offset `0x74` through that newly obtained address yields **Late head**. A new read from Primary + `0x64` supplies **Published byte**, its low byte; Primary + `0x1d` receives that byte.

The function next reads Primary + `0x44`, then reads offset `0x14` through the result, capturing **Replacement link**. This traversal uses the current field contents, not an assumed earlier Secondary value.

Late head + `0x34` receives Retained link, and Primary + `0x14` receives Replacement link. After **Saved primary payload** is read from Primary + `0x54`, Replacement link + `9` receives Primary’s low byte.

The function reads Late head + `4`, then offset `0x74` through that result, obtaining **Byte carrier**. Byte carrier + `0x15` receives Retained link’s low byte. Primary + `0x54` receives Primary, Primary + `0x11` receives Late head’s low byte, and Byte carrier + `0x11` receives Saved primary payload’s low byte.

**Carrier payload** is read from Byte carrier + `0x64`. Primary + `9` receives Published byte. A fresh read of Late head + `4` captures **Byte destination**.

Byte destination + `9` receives Primary’s low byte. After **Byte payload** is read from Byte destination + `0x50`, Byte destination + `0x11` receives Carrier payload’s low byte, followed by another write of Primary’s low byte at Byte destination + `9`.

Primary + `0x14` receives Carrier payload; Primary + `0x0d` receives Byte destination’s low byte; Byte destination itself receives Byte payload’s low byte.

The function reads **Tail source** from Late head + `0x74` and **Late parent** from Late head + `0x14`, before writing Byte destination’s own address at its `0x54`.

A read from Late parent + `4` yields **Final marker destination**. Its `0x10` receives byte 100, Primary + `0x54` receives full-width 100, and Final marker destination + `0x40` receives full-width 100.

Finally, Primary + `0x0d` receives Carrier payload’s low byte, Late parent + `9` receives Final marker destination’s low byte, Primary + `0x24` receives Carrier payload, and Late parent + `0x11` receives Primary’s low byte.

### 6. Arithmetic-address tail

X is read first. A subsequent read from Primary + `0x74` captures **Tail link**.

Tail link’s low halfword is written at Tail link plus Primary. X’s low halfword is then written at twice Final marker destination.

One byte is read at Tail source plus Primary and converted to a 32-bit unsigned quantity, **Tail byte value**. The conversion’s result depends on whether the source byte is interpreted as signed.

Tail link’s low halfword is written at X plus Tail link, followed by a second identical halfword write at Tail link plus Primary. The full-width constant `0x45` is then written at Tail byte value plus `0x45`.

The next reads obtain Y, Z, and **Final field** from Primary + `0x44`, in that order. The remaining writes are:

- Byte `0x44` at Tail byte value plus `0x44`.
- Z at twice Y.
- Y’s low halfword at Retained link plus Y.

The indirect dependency then receives three 32-bit arguments, in order:

1. Tail byte value bitwise-OR `0x44`.
2. Y.
3. Final field.

The first argument uses OR, not addition. Retained link in the final halfword destination is the earlier capture from Continuation + `0x74`, not Tail link.

## Side effects

The represented effects are memory reads, ordered memory writes, and the indirect invocation. Writes include overlapping widths, self-address publication, repeated identical stores, and unchanged-value write-back.

There is no represented rollback or recovery path. A fault can leave earlier stores completed; the representation supplies no error result or partial-completion guarantee. Visibility and fault granularity remain machine-dependent.

Thread safety, reentrancy, preservation of global state, and behavior under concurrent modification are not established. The final dependency may perform additional mutations, transfer ownership, release memory, or fail to return; none of those behaviors may be assumed absent.

## Dependencies

**F-unknown-indirect** is a documentation-only opaque identifier for the unresolved final dependency. It is not a recovered identity.

Its three arguments are specified above. Its calling convention, return behavior, memory effects, global-state preservation, ownership rules, and error behavior are **documented_only** unknowns.

No other ordinary function invocation is represented. X, Y, and Z are external-state dependencies rather than evidence of additional calls.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Narrowing:** A second input of `0x12345678` produces byte stores of `0x78` and full-width stores of `0x12345678` at the specified points. Later aliases may replace them.

2. **Initial ordering:** After Primary is captured, the byte write at C + `0x11` must precede the full-width write at A + `0x44`. If those accesses overlap, their order remains significant.

3. **First marker order:** The first destination receives its two byte writes before Primary + `0x54` receives 100. Its full-width `0x60` write follows that Primary write.

4. **Intermediate capture:** The read of Reached destination + `0x74` precedes its self-address store at `0x54` and the later writes to Secondary. An alias-sensitive model must not move that read past those writes.

5. **Distinct nearby captures:** If the intervening Traversal base + `0x20` write changes Primary + `0x64`, First nearby capture and Second nearby capture may differ. Primary + `0x44` receives the first; Traversal base + `9` receives the second’s low byte.

6. **Late field traversal:** The late read of Primary + `0x44` follows its replacement with First nearby capture and all intervening accesses. It must use the value present at that later read.

7. **High-bit byte conversion:** If the tail byte is `0x80`, a signed-byte interpretation produces Tail byte value `0xffffff80` and first indirect argument `0xffffffc4`. An unsigned-byte interpretation produces `0x00000080` and `0x000000c4`. Selecting either outcome requires the missing signedness convention.

8. **OR versus addition:** A tail byte of `0x44` produces first argument `0x44`, not `0x88`, under either byte interpretation.

9. **Repeated writes:** Matching only the final contents is insufficient when an implementation omits either consecutive Primary + `0x0d` byte write or the repeated Tail link-plus-Primary halfword write.

10. **Invalid destination:** If twice Y is unwritable, successful completion is not promised. No represented validation precedes that write.

## Uncertainties

The following remain **documented_only**:

- Completeness and accuracy of the represented function boundary, signature, and machine-state inputs.
- Actual ABI and address representation.
- Semantic types, extents, lifetimes, and ownership of referenced regions.
- Byte order, alignment restrictions, arithmetic overflow, and fault semantics.
- Signedness of the tail byte interpretation, particularly for values `0x80`–`0xff`.
- Whether memory is ordinary, shared, device-backed, externally modified, or subject to special access rules.
- Storage identities and stability of X, Y, and Z.
- Identity and behavior of F-unknown-indirect.
- Intended meanings of constants, repeated stores, and self-address publication.

No stable final object invariant is established: later writes and aliases can invalidate apparently meaningful intermediate layouts.

## Evidence

The evidence is the supplied static representation: access widths, fixed offsets, separately occurring reads, captured-value dependencies, narrowing conversions, arithmetic destinations, external reads, and the unresolved indirect invocation.

This supports the ordered trace described above. It does not independently establish executable address validity, target byte signedness, ABI correctness, ownership, or application intent.

No runtime tests have been performed. Acceptance examples describe conditional consequences of the static evidence, not observed execution.
