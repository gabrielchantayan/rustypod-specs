<!-- Trusted provenance: F-aa5e874b1ad0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa5e874b1ad0

## Purpose

F-aa5e874b1ad0 records a selector byte, performs unconditional control-word accesses, and then either invokes a selector-specific dependency or returns an unsupported-selector result.

Selectors zero and one are accepted. Acceptance is determined from the complete 32-bit selector, not from the byte recorded in memory. Every other selector returns failure only after the selector-byte write and the initial control-word operations.

The initial near-word operation has a significant address-selection property: its read address and write address come from separate evaluations of a global address value. They do not use the previously captured control-region address. If that global changes between evaluations, the operation can read one word and write a masked copy into another. Describing it unconditionally as clearing a bit in one word would therefore be inaccurate.

The apparent intent is to select between two control configurations. Hardware meanings, lifecycle semantics, and the purpose of the dependency calls are not established.

## Interface

The interface has two inputs and one 32-bit result.

| Item | Established contract |
|---|---|
| First input | An opaque 32-bit value. Forwarded unchanged to F-04e0b56d7624 only for selector one. Not otherwise interpreted or dereferenced directly by this function. |
| Second input | A 32-bit integer selector. Its complete value controls selection; its low eight bits are stored before selection. |
| Accepted-selector result | `0x00000000`, provided execution reaches the selected dependency and that dependency returns normally. |
| Unsupported-selector result | `0xFFFFFFFF`, after the unconditional memory operations complete. |

There is no explicit output pointer. Observable outputs include the result, memory accesses and mutations, and any effects of a dependency invocation.

Two global address-bearing locations participate:

- **Selector-storage global:** supplies the base for the selector-byte destination.
- **Control-region global:** supplies a captured base and, separately, the two address evaluations used by the initial near-word operation.

The relevant layout is:

| Access | Address derivation | Width |
|---|---|---:|
| Selector-byte destination | Selector-storage base plus three bytes | 8 bits |
| Initial near-word read | Independently evaluated control-region global plus eight bytes | 32 bits |
| Initial near-word write | Separately evaluated control-region global plus eight bytes | 32 bits |
| Later near-word accesses | Captured control-region base plus eight bytes | 32 bits |
| All distant-word accesses | Captured control-region base minus `0x000FFFFC` bytes | 32 bits |

The distant-word displacement is negative 1,048,572 bytes, not negative 1,048,576 bytes. Relative to one captured base, the near and distant word starting addresses are `0x00100004` bytes apart.

The supplied representation establishes 32-bit data-word operations and an eight-bit selector store. It does not independently establish a target address-space model, byte order, alignment rules, or behavior when address arithmetic exceeds the valid address range.

The function performs no explicit storage-validity checks. Ordinary completion requires the accesses actually selected during execution to be valid. Accepted selectors additionally require valid later near-word accesses and a normally returning dependency.

## Behavior

### Unconditional entry operations

The function first writes the selector’s low eight bits to the selector-storage base plus three bytes. This happens for accepted and unsupported selectors alike.

Next, it evaluates the control-region global and captures that address value. Later distant-word operations and branch-specific near-word operations use this captured value.

The initial near-word operation then:

- Obtains a 32-bit word using an address formed from an evaluation of the control-region global plus eight bytes.
- Clears bit 4 of that read value using mask `0xFFFFFFEF`.
- Writes the resulting 32-bit value using an address formed from a separate evaluation of the control-region global plus eight bytes.

Neither address uses the captured base.

The supplied representation does not establish the relative evaluation order of the read-address and write-address expressions. In particular, it does not establish that the write-address global evaluation occurs after the word read. The word value must be obtained before the resulting value can be stored, but this does not determine when the destination address is evaluated.

Consequently:

- With a stable control-region global, the initial near-word operation reads and writes the same word.
- If the global changes between its two address evaluations, the read and write addresses can differ.
- The captured base can differ from either independently evaluated base.
- When the addresses differ, the operation writes a masked copy of the source word into the destination word. It does not preserve the destination word’s previous unmasked bits.
- Subject to successful storage, the value written has bit 4 clear. This alone does not establish the resulting value of the captured near word.

After the initial near-word operation, the function reads the distant word relative to the captured base, clears bit 1 using mask `0xFFFFFFFD`, and writes the result to that same captured-base location.

These operations precede the selector branch. There is no early rejection of unsupported selectors and no rollback.

### Selector zero

For a full selector value of zero, the function performs the following additional operations in order:

1. Reads the near word relative to the captured base, sets bit 4 using mask `0x00000010`, and writes the result to the same location.
2. Reads the distant word relative to the captured base, sets bit 1 using mask `0x00000002`, and writes the result to the same location.
3. Invokes F-d5aab148f638 with no explicit arguments.
4. Returns `0x00000000` if the dependency returns normally.

The branch-specific operations use the captured base even if the control-region global has changed.

With stable addresses, nonoverlapping ordinary storage, and no intervening mutation, this path clears and then sets both selected bits before the call. Without those conditions, the stronger established rule is the ordered sequence of accesses and masks, not a claim that both clears affected the same words later updated.

Dependency effects are unknown. The bit values immediately before the call are not unconditional guarantees about memory at function return.

### Selector one

For a full selector value of one, the function:

1. Reads the near word relative to the captured base and writes that same 32-bit value back to the same location.
2. Reads the distant word relative to the captured base, sets bit 1, and writes the result to the same location.
3. Invokes F-04e0b56d7624 with the original first input unchanged.
4. Returns `0x00000000` if the dependency returns normally.

The near-word self-write is explicitly present in the supplied representation. It is not an absent operation merely because the written value equals the value read.

With a stable control-region global, nonoverlapping ordinary storage, and no intervening mutation, near-word bit 4 is clear and distant-word bit 1 is set immediately before the call.

If the initial near-word write used a different base, this branch does not independently clear bit 4 of the captured near word. It reads and rewrites whatever value that location supplies at that time.

### Unsupported selectors

For every full selector value other than zero or one, the function returns `0xFFFFFFFF` after the unconditional operations.

Before that result:

- The selector byte has been written.
- The initial near-word source has been read.
- A value with bit 4 cleared has been written to the independently selected near-word destination.
- The distant word relative to the captured base has undergone its bit-1 clear.
- Neither dependency has been invoked.

There is no restoration of prior contents.

Under stable-address, nonoverlapping ordinary-memory conditions, this reduces to clearing bit 4 of the near word and bit 1 of the distant word. Without those conditions, the initial near-word operation must retain its separate-source-and-destination description.

Selectors 256 and 257 illustrate the distinction between recording and acceptance: they record bytes `0x00` and `0x01`, respectively, but both take the unsupported-selector path.

## Side effects

At the supplied representation level, every path contains:

- One eight-bit selector store.
- One control-region address capture.
- Two separate control-region global evaluations for the initial near-word operation.
- One 32-bit initial near-word read and one 32-bit initial near-word write.
- One 32-bit distant-word read and write.

Each accepted path additionally contains two 32-bit reads, two 32-bit writes, and one dependency invocation.

The established high-level sequence is:

1. Selector-byte store.
2. Control-region address capture.
3. Initial near-word operation, with the internal address-evaluation order unresolved.
4. Initial captured-base distant-word clear.
5. Branch-specific captured-base near-word operation, if accepted.
6. Branch-specific captured-base distant-word set, if accepted.
7. Selected dependency invocation, if accepted.
8. Result return, if reached.

These are source-level access requirements, not guarantees of particular bus transactions. Atomicity, synchronization, memory barriers, volatile behavior, and device-register semantics are not established.

Aliasing can affect later observations. For example, the selector-byte store precedes address capture and word reads. If that store overlaps relevant address storage or control storage, later operations may observe its effects. Likewise, a word write overlapping a global address-bearing location could influence subsequent evaluations. The supplied evidence does not establish that such overlaps occur or that they are excluded.

No allocation, release, or ownership transfer is directly visible. The first input is opaque; forwarding it does not prove that it is a pointer, handle, or owned resource.

Neither dependency’s result is used. A normal dependency return is followed by zero regardless of any returned value. No explicit dependency-error translation, recovery, or post-call restoration is present. A dependency that does not return prevents the ordinary success return.

Neither dependency may be assumed to preserve global addresses, control words, the selector byte, unrelated state, or resources associated with the first input.

## Dependencies

| Opaque function ID | Invocation condition | Explicit inputs | Established handling |
|---|---|---|---|
| F-d5aab148f638 | Full selector equals zero | None | Invoked after the selector-zero word operations; any result is ignored |
| F-04e0b56d7624 | Full selector equals one | Original first input, unchanged | Invoked after the selector-one word operations; any result is ignored |

Both dependency contracts are otherwise **documented_only**: memory effects, ownership behavior, resource release, global preservation, failure signaling, and termination behavior are unresolved.

Neither dependency validates unsupported selectors.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

Unless an example states otherwise, assume valid ordinary-memory storage, stable global addresses, no relevant overlap, and no concurrent or intervening mutation. Values at dependency entry precede any dependency effects.

### Zero selects the set/set configuration

Given both control words initially contain `0x00000000` and the selector is zero:

- The selector byte becomes `0x00`.
- Both initial word operations write zero.
- The captured near word becomes `0x00000010`.
- The captured distant word becomes `0x00000002`.
- F-d5aab148f638 is called.
- A normal dependency return is followed by result `0x00000000`.

Post-call memory values remain unspecified without a dependency contract.

### One retains the near clear and sets the distant bit

Given both words initially contain `0xFFFFFFFF` and the selector is one:

- The selector byte becomes `0x01`.
- The initial near-word write produces `0xFFFFFFEF`.
- The initial distant-word write produces `0xFFFFFFFD`.
- The near-word self-write reads and writes `0xFFFFFFEF`.
- The distant-word set produces `0xFFFFFFFF`.
- F-04e0b56d7624 receives the exact first input.
- A normal dependency return is followed by zero.

### Unsupported selection still mutates storage

Given both words initially contain `0xFFFFFFFF` and the selector is two:

- The selector byte becomes `0x02`.
- The near word becomes `0xFFFFFFEF`.
- The distant word becomes `0xFFFFFFFD`.
- Neither dependency is called.
- The result is `0xFFFFFFFF`.

Rejecting this selector before the memory operations would not match the established behavior.

### Byte truncation does not grant acceptance

For selector 256, the recorded byte is `0x00`; for selector 257, it is `0x01`. In both cases, the initial word operations occur, neither dependency is called, and the result is `0xFFFFFFFF`.

### Different initial read and write bases copy masked source data

Suppose the initial near-word read-address evaluation selects a word containing `0x12345678`, while its write-address evaluation selects a distinct word containing `0xAAAAAAAA`.

The initial operation writes `0x12345668` to the destination. It does not produce `0xAAAAAA A A` with only bit 4 cleared, nor otherwise preserve the destination’s prior unrelated bits. The source remains unchanged by this particular write if the locations do not overlap.

This example is conditional on those address-evaluation outcomes. It does not prescribe which global evaluation occurs first or establish a supported concurrency model.

### A captured near word can escape the initial clear

Suppose the captured base identifies one region, while both initial near-word address evaluations identify a different, nonoverlapping region. Let the captured near word initially contain `0x00000010`, and select one.

The initial near-word operation acts on the other region. The branch-specific self-write reads and writes `0x00000010` at the captured near word. Therefore, bit 4 of that captured word is still set immediately before F-04e0b56d7624, assuming no other effects change it.

This demonstrates why selector one does not unconditionally guarantee a cleared captured near-word bit.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the selector, control bits, and dependencies.
- The relative evaluation order of the two global-derived address expressions in the initial near-word operation.
- Whether changing global values between evaluations has defined behavior in the actual execution environment.
- Storage validity, lifetime, alignment, allocation extent, and aliasing.
- Address width, address-overflow behavior, byte order, and fault behavior.
- Whether addressed storage is ordinary memory or device-backed storage.
- How the represented reads and writes map to physical transactions.
- Concurrency, interrupt, signal, and reentrant behavior.
- All dependency effects beyond invocation condition and explicit argument forwarding.
- The operational significance of the selector-one self-write.

Each mask preserves unmasked bits of the value actually read. For the initial near-word operation, that is not necessarily preservation of the destination’s previous bits. For later operations, the read and write addresses derive from the same captured base, but intervening mutations and device effects remain unresolved.

## Evidence

This contract uses the supplied static function representation and opaque dependency mapping. Embedded commentary, the prior draft, and reviewer feedback are not independent proof of behavior.

Established static evidence includes the selector-byte store, address capture, separate global-derived initial near-word address expressions, 32-bit masks, captured-base later accesses, full-selector comparisons, dependency selection, unchanged first-input forwarding, and the two result bit patterns.

The two-configuration interpretation is inferred intent, not an established hardware contract.

No dependency implementation, target memory map, runtime trace, or execution-environment contract was supplied. No runtime tests have been performed. All acceptance examples are conditional static consequences rather than observed results.
