<!-- Trusted provenance: F-191acd2fadfa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-191acd2fadfa

## Purpose

F-191acd2fadfa selects one byte from externally referenced storage and returns that byte’s bits selected by mask 0x06.

The selector is unsigned. Values from 0 through 255 select the corresponding byte offset. Every representable selector greater than 255 selects offset zero instead. This is a fallback-to-zero rule, not modulo-256 indexing.

The established operation is a byte lookup followed by bit filtering. The storage’s application meaning is unknown. Neither retained bit has an established semantic name.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

### Input

The function receives one unsigned integer by value.

The supplied representation identifies an unsigned integer input but does not independently establish its exact ABI width. A 32-bit input is a conventional interpretation, not a verified interface fact. The behavior described here applies to values representable by the actual input type.

The function distinguishes:

- Values from 0 through 255 inclusive.
- Representable values greater than 255.

There is no signed-input case within the visible interface. Conversion of negative values, or values exceeding the input type’s range, occurs outside the established behavior of this function.

No storage address or storage length is passed as an explicit argument. The lookup base comes from an external data reference.

### Output

The supplied representation identifies a byte-sized return and a byte-sized storage read, interpreted here as unsigned 8-bit quantities. Exact ABI return conventions are not independently established.

The returned value is the selected byte filtered by mask 0x06:

| Return value | Meaning at the bit level |
|---|---|
| 0x00 | Bits 1 and 2 are both clear. |
| 0x02 | Bit 1 is set and bit 2 is clear. |
| 0x04 | Bit 1 is clear and bit 2 is set. |
| 0x06 | Bits 1 and 2 are both set. |

Bit positions are numbered from zero at the least significant bit.

The result is not shifted into the range zero through three and is not converted to a Boolean. All bits other than bits 1 and 2 are cleared.

There is no separately returned error indicator. A zero result is an ordinary filtered value, not evidence of failure.

### Storage preconditions

For a normal return, the external reference must supply a usable lookup base, and the selected address must permit the byte read.

For a particular invocation, only its selected byte must be readable according to the execution environment’s memory rules. Supporting every selector from 0 through 255 requires readable storage at all 256 corresponding offsets.

The function does not establish or check the storage extent. The actual allocation size, address-space rules, and validity of address formation remain external preconditions.

## Behavior

### Selector handling

The comparison against 255 uses the unsigned input value.

Selectors from 0 through 255 remain unchanged for lookup purposes. Every greater representable value is replaced with zero for lookup purposes.

The resulting selector is then restricted to its low eight bits when forming the offset. Because the preceding range handling has already produced a value between 0 and 255, this restriction does not change the selected offset.

The order matters:

- Input 255 selects offset 255.
- Input 256 selects offset zero.
- Input 257 also selects offset zero, not offset one.
- Input 511 selects offset zero, not offset 255.

Thus, retaining the original selector’s low eight bits is not an equivalent implementation.

Only the function’s local treatment of the by-value argument changes. No normalized selector is visibly written to caller-supplied storage.

### Lookup layout

The storage is accessed as bytes. A normalized selector of zero identifies the byte at the lookup base. A normalized selector of 255 identifies the byte 255 bytes after that base.

There is no visible header skip, element-size multiplier, second-level lookup, or multibyte value interpretation. Endianness does not affect filtering of the single selected byte.

The visible offset is bounded by 255. The function does not visibly validate whether adding that displacement produces a valid address in the target environment. Pointer width, address overflow behavior, and address-space restrictions are not established by the supplied representation.

### Read and result formation

After selector normalization, the function uses the external lookup base to identify the selected address, reads the byte there, and filters the loaded value with mask 0x06.

Bits 1 and 2 retain their loaded values. Every other bit is discarded.

For example:

- A selected byte of 0xFF produces 0x06.
- A selected byte of 0x01 produces 0x00.
- A selected byte of 0x06 produces 0x06.
- A selected byte of 0xF9 produces 0x00.

The filtering may involve an integer representation wider than a byte. Its result is nevertheless one of zero, two, four, or six, so a byte-sized return does not discard any retained bit.

### Observable access order and extent

The source-facing sequence is:

1. Determine the lookup offset from the unsigned selector.
2. Obtain the lookup base through the external data reference.
3. Read the byte at the selected address.
4. Return the filtered value.

There is one selected-byte read in the visible expression. There is no visible scan, neighboring-byte inspection, retry, or additional table lookup.

An oversized selector does not cause an early return and does not avoid the storage read. It redirects the read to offset zero.

This sequence describes the operations expressed by the supplied representation. It does not establish exact machine-instruction counts, speculative-access behavior, physical memory transaction sizes, atomicity, or memory barriers.

### Failure behavior

The function contains no visible base-validity check, storage-length check, recoverable error branch, or alternate error result.

If the required address cannot validly be read, the evidence does not establish a normal return. Whether the environment faults, traps, terminates execution, or exhibits another invalid-access consequence is unresolved.

No fallback byte is supplied when storage is invalid. The selector fallback to offset zero is independent of storage validity: offset zero must itself be readable when selected.

## Side effects

No explicit memory writes appear in the visible body. In particular, there is no visible operation that changes the external base reference or writes the selected byte.

No explicit allocation, release, or other ownership-management operation is visible. No ownership transfer is visible. These observations do **not** establish an ownership contract: the storage’s meaning and any ownership implications of accessing it remain unknown.

The byte-valued return establishes the form of the result, not whether that value has resource-related significance elsewhere. The evidence does not justify a general promise that caller ownership is preserved or that no newly owned resource can be represented or implied by the returned value.

The storage is read through its external reference on the visible lookup path; no persistent copy or cache is shown. Repeated calls with the same selector can therefore produce different results if the effective base or selected byte changes between accesses.

No synchronization is visible. The function does not establish a coherent snapshot across concurrent changes to the base reference and referenced storage. Safe publication, lifetime protection, and concurrent-update rules must be supplied by the surrounding environment, if needed.

The absence of explicit writes is not a guarantee that execution has no external side effects. The storage could have read-triggered behavior, including device-specific effects. Such effects, and any resulting ownership implications, are not characterized by this evidence.

## Dependencies

No dependency function calls are visible. Consequently, there are no callee contracts to identify, and this specification makes no assumptions about unknown callees preserving globals or managing memory.

The visible non-call dependencies are:

- An external data reference supplying the lookup base.
- A readable byte at the selected address.
- The execution environment’s address-formation and memory-access rules.

The external reference’s exact representation is unresolved. The evidence does not establish whether its declaration is an ordinary pointer, an address-valued object, or another representation that supports the displayed address calculation.

Initialization, storage extent, lifetime, ownership, and replacement policy are also unresolved. F-191acd2fadfa performs no visible operation that establishes those properties.

## Acceptance examples

These examples are conditional requirements derived from static evidence, not observed test results. Each assumes that the input is representable, the external base is usable, the selected byte is readable, and no interfering change alters the relevant access.

| Input | Relevant storage contents | Required result |
|---|---|---|
| 0 | Offset 0 contains 0xFF. | Select offset 0 and return 0x06. |
| 1 | Offset 1 contains 0x02. | Select offset 1 and return 0x02. |
| 42 | Offset 42 contains 0x04. | Select offset 42 and return 0x04. |
| 255 | Offset 255 contains 0x80. | Select offset 255 and return 0x00. |
| 256 | Offset 0 contains 0x04. | Select offset 0 and return 0x04. |
| 257 | Offset 0 contains 0x02; offset 1 contains 0x04. | Return 0x02, distinguishing fallback-to-zero from modulo-256 indexing. |
| 511 | Offset 0 contains 0x04; offset 255 contains 0x02. | Return 0x04, distinguishing fallback-to-zero from retaining the original low eight bits. |

Additional conditional acceptance properties:

- For otherwise identical accesses, changing only selected-byte bits outside mask 0x06 must not change the returned value.
- A selected byte of 0x01 returns zero despite being nonzero.
- A selected byte of 0x06 returns six, not three or one.
- Every representable selector greater than 255 selects offset zero, regardless of its original low eight bits.
- With ordinary, non-side-effecting storage and no interference, equal selected bytes produce equal results.
- An oversized selector still requires a read at offset zero. A purported implementation that returns zero without that read does not preserve the visible behavior.
- An invocation selecting offset zero does not visibly inspect offsets 1 through 255. Readability of the entire region is not established as a requirement for that invocation.

No normal-result acceptance case is specified for an invalid selected address. No ownership-preservation acceptance property follows from the absence of explicit ownership-management operations.

## Uncertainties

### Established static evidence

The visible representation establishes unsigned selector handling, fallback to zero above 255, a byte displacement restricted to eight bits, a selected-byte read, filtering by 0x06, and a byte-sized return.

It also establishes the absence of explicit function calls, memory writes, allocation, release, validation branches, and error-return paths in the visible body.

These are statements about visible operations, not complete guarantees about the environment or the consequences of reading externally supplied storage.

### Inferred intent

The lookup and mask are consistent with extracting two flags from a byte-indexed table. This is an interpretation, not an established application contract. It must not be used to assign meanings to the selector or retained bits.

A 32-bit unsigned input and an ordinary pointer-valued base are conventional interpretations. Neither is independently verified by an ABI declaration or runtime observation.

### documented_only

The following remain unresolved:

- Exact input ABI width, calling convention, and return-register conventions.
- Independent confirmation of the byte type’s width and unsigned representation.
- Pointer width, external-reference declaration, and address-space rules.
- Storage size, initialization, ownership, and lifetime.
- Whether the base reference can change during an invocation.
- Whether the storage is ordinary, immutable, concurrently mutable, or device-backed.
- Read-triggered side effects and any associated ownership implications.
- Synchronization requirements and invalid-access consequences.
- Application meanings of the selector, returned byte, and retained bits.

These unknowns must not be replaced with assumed ownership guarantees, recovery behavior, or synchronization policies.

## Evidence

The supplied static representation shows an unsigned comparison with 255, replacement of greater selectors with zero, restriction of the lookup displacement to eight bits, a byte read relative to an external base, and filtering with mask 0x06 before return.

The selected offset range, four-value result set, and distinction between fallback-to-zero and modulo-256 indexing follow directly from those operations.

The visible body contains no explicit calls or writes and no explicit ownership-management operation. This supports only the statement that no ownership transfer is visible; it does not establish that ownership is preserved or that the returned value has no ownership significance.

Exact ABI details, storage-management rules, read-side-effect semantics, and platform-specific failure behavior are not established. No runtime tests have been performed.
