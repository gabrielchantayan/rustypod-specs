<!-- Trusted provenance: F-81e7cec10db8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-81e7cec10db8

## Purpose

F-81e7cec10db8 replaces a selected object’s 32-bit field with the low 13 bits of an unsigned input value.

Selection depends on a value read from shared global storage. That value serves as an anchor for selector-relative entry addressing. The selected entry supplies an object address, and the destination field lies 1,036 bytes beyond that address.

The operation replaces the entire destination field; it does not preserve any of its previous bits. On ordinary completion, the destination’s numeric value is between 0 and 8,191 inclusive.

The application-level meaning of the selector, entries, objects, and field is unknown. Calling the destination a flag set, identifier, or device setting would be inferred intent rather than established behavior.

## Interface

### Explicit inputs

| Input | Width and interpretation | Role |
|---|---|---|
| Selector | Signed 16-bit integer | Determines a byte displacement from the anchor value read from shared storage. |
| Value | Unsigned 32-bit integer | Supplies the low 13 bits written to the destination. |

These widths use the conventional 32-bit data model represented by the supplied source: 16-bit signed short, 32-bit signed and unsigned integers, and 32-bit addresses. No independent executable metadata or runtime observation confirms that model.

The selector’s representable range under this model is −32,768 through 32,767. It remains signed when used for entry selection. Negative selectors are not visibly rejected or converted into unsigned indices.

The value’s upper 19 bits do not affect the stored result.

### Implicit inputs and memory layout

The function depends on three distinct memory locations or regions:

1. **Anchor storage:** a shared global location holding the current anchor value.
2. **Entry region:** memory addressed relative to the anchor value obtained from that storage.
3. **Destination object:** memory reached through the address read from the selected entry.

The anchor storage is not the entry region. Reading the global supplies the value used as the entry-region anchor; the location of that global is not itself the base of the selector-relative lookup.

The required layout is:

| Component | Required relationship |
|---|---|
| Anchor storage | Holds the anchor value used by this invocation. |
| Selected entry | Located at the loaded anchor plus 396 times the signed selector, in bytes. |
| Entry’s leading word | A 32-bit address value identifying the destination object. |
| Destination field | A 32-bit field at byte offset 1,036 from the loaded object address. |

The 396-byte spacing is established for entry selection. It does not establish the allocated size, count, or semantic structure of the entries. Only the leading 32-bit word of the selected entry is visibly read.

The anchor value is an input obtained from shared state, not a fixed table address embedded in this contract. Different anchor values can cause identical explicit arguments to select different memory.

### Output

There is no explicit return value or status result.

On ordinary completion, the observable result is a 32-bit replacement write at the computed destination. The function does not return the previous destination value, the selected object address, or an indication of whether the memory was valid.

### Preconditions for ordinary completion

Successful execution requires:

- Readable shared storage containing a usable anchor value.
- A readable 32-bit word at the selected entry location.
- A loaded object address for which the destination at offset 1,036 supports a 32-bit write.
- Address calculations and accesses permitted by the execution environment’s arithmetic, alignment, and memory rules.

These are execution preconditions, not checks performed by the function. The visible body does not validate any of them.

## Behavior

### Anchor acquisition and entry selection

The function reads the anchor value from shared global storage. It uses that loaded value, together with the signed selector, to determine the selected entry’s address.

The selector contributes a displacement of 396 bytes per unit:

- Selector zero contributes no displacement.
- Selector one contributes 396 bytes.
- Selector negative one contributes −396 bytes.

Across the full signed 16-bit range, the displacement ranges from −12,976,128 through 12,975,732 bytes. Both endpoints fit within a signed 32-bit integer. Thus, under the stated width model, the selector multiplication itself does not overflow.

This arithmetic fact does not establish the validity of the resulting address. Adding a displacement to an anchor can still produce an unusable location or encounter arithmetic limitations. The source does not establish a portable rule for address overflow, wraparound, or out-of-region access.

There is no visible bounds check, clamping, selector normalization, default entry, or special treatment of negative selectors.

### Object lookup and destination selection

At the selected entry address, the function reads a 32-bit word used as the destination object’s address.

The function then selects the 32-bit field 1,036 bytes beyond that loaded address. The field is therefore reached by indirection through the entry; it is not inherently located inside the entry region.

No visible check confirms that the loaded address is nonzero, points to an object of sufficient size, or identifies writable memory. No object header, length, type indicator, or existing destination value is inspected before the write.

The address-bearing word is represented through a 32-bit integer access in the supplied source. Interpreting its value as an object address describes its use, not an independently established pointer type or portable pointer representation. The validity of the subsequent address arithmetic remains environment-dependent.

### Value transformation and replacement

The value is restricted to its lowest 13 bits. For an unsigned 32-bit input, this produces the same numeric result as the remainder after division by 8,192.

The entire 32-bit destination is assigned that result:

- Destination bits 0 through 12 receive the corresponding input bits.
- Destination bits 13 through 31 become zero.
- No previous destination bits are preserved.

This is not a read-modify-write operation on the existing destination value. The visible expression does not require reading that value.

Two inputs differing only in bits 13 through 31 produce the same stored result when they resolve to the same destination. An input with all 32 bits set produces 8,191, not a negative value.

### Observable memory dependency order

The established memory-access dependency chain is:

1. Read the anchor value from the shared global storage.
2. Use that loaded anchor and the selector displacement to address and read the selected entry’s 32-bit object-address word.
3. Use the loaded object address and the 1,036-byte offset to address and write the destination field.

The first read is material: the entry lookup depends on the value obtained from global storage. An account beginning only with the entry read would omit an input access.

The source does not establish an additional ordering guarantee between calculating the masked input value and resolving the destination. In particular, it does not require masking to occur before or after the anchor read or entry read.

This dependency order is not a synchronization guarantee. The supplied representation does not establish atomicity, memory barriers, inter-thread visibility order, or an exact machine-instruction sequence.

### Completion and errors

After the replacement assignment, the function returns without an explicit result.

No visible branch handles an unreadable anchor, inaccessible entry, invalid loaded address, unwritable destination, or invalid address calculation. There is no defined error return, recovery path, fallback destination, or safe no-op for these cases.

A particular signal, exception, or fault sequence cannot be promised from the supplied material. Ordinary-completion guarantees do not extend to invalid executions.

## Side effects

The visible operation consists of an anchor read, an entry-word read dependent on that anchor, and a destination write dependent on the entry word.

There is no separate visible write to the anchor storage or selected entry. However, their preservation is conditional on non-overlap: the computed destination may alias either location because the source imposes no non-aliasing condition.

For example, if the destination overlaps the anchor storage, the final write can change the anchor value available to a later invocation. The current invocation has already used its anchor read to resolve the destination. The function does not visibly restart selection or reread the anchor after writing.

No allocation, deallocation, retention, release, reference-count adjustment, callback, or external function call appears in the body. The function operates on existing memory. This does not establish who owns that memory, whether the caller must retain it, or how long it remains valid.

The numeric 32-bit result is established; its byte serialization is not. Endianness, access alignment requirements, write indivisibility, and device-memory effects are unresolved. The function provides no visible locking or other concurrency mechanism.

## Dependencies

There are no visible function-call dependencies. No additional opaque function IDs are required to describe the operation.

The function depends on:

- The externally established value in shared anchor storage.
- Memory readable at the selector-relative entry address.
- A usable object-address value in the selected entry’s leading 32-bit word.
- Writable memory at the destination field.
- The environment’s integer, address, alignment, and memory-access rules.

There is no visible mechanism for initializing, validating, owning, or extending the lifetime of these resources.

## Acceptance examples

These are conditional expectations derived from static inspection. They are not executed tests. Each successful example assumes valid memory and valid address arithmetic under the stated width model.

### Anchor-dependent selection

Suppose the shared global storage contains an anchor identifying region A. Selector zero reads the object-address word at region A’s anchor and writes the field 1,036 bytes beyond that loaded address.

If the shared storage instead contains an anchor identifying region B on a later invocation, the same selector and input value use region B’s entry. Treating the shared global’s own storage address as the entry-region anchor would not match the demonstrated lookup.

### Selector displacement

Suppose readable entries exist at the loaded anchor, 396 bytes after it, and 396 bytes before it. Their leading words identify distinct writable objects.

- Selector zero targets the object named by the entry at the anchor.
- Selector one targets the object named by the entry 396 bytes after it.
- Selector negative one targets the object named by the entry 396 bytes before it.

Each write occurs at offset 1,036 within the corresponding addressed object. Rejecting the negative selector solely because it is negative would add behavior not present in the visible operation.

### Mask boundaries

For a valid selected destination:

| Input value | Destination after ordinary completion |
|---|---:|
| 0 | 0 |
| 37 | 37 |
| 8,191 | 8,191 |
| 8,192 | 0 |
| 8,193 | 1 |
| 4,294,967,295 | 8,191 |

These outcomes distinguish low-13-bit masking from saturation or an unchanged full-width input write.

### Full-width replacement

Given a destination initially containing 4,294,967,295 and an input value of 1, ordinary completion leaves the destination equal to 1.

An outcome retaining any previous upper bits would not satisfy the replacement behavior.

### Lookup dependency

For valid, non-overlapping storage, destination selection depends on the value read from the global, followed by the object address read from the selected entry. The destination write cannot be specified independently of either loaded value.

This example does not constrain whether the input mask is calculated before, between, or after those reads.

### Invalid memory

If the global anchor storage or selected entry is unreadable, or the resulting destination is unusable, no successful result is promised. A specific error code, fallback write, or harmless return is not an acceptance requirement.

## Uncertainties

The following remain **documented_only**:

- Independent confirmation of the assumed integer and address widths.
- The origin, initialization, lifetime, and permitted mutation of the shared anchor.
- The entry region’s extent and application-valid selector range.
- The semantic meaning and actual allocated size of entries and destination objects.
- Whether null, sentinel, or otherwise unusable address values occur.
- The precise arithmetic and fault behavior for invalid address calculations.
- Alignment, endianness, atomicity, and concurrent-access guarantees.
- Whether accesses involve ordinary memory or storage with device-specific effects.
- Ownership and lifetime obligations imposed by callers.
- Whether callers enforce stronger bounds or validity conditions.

“Entry region,” “object,” and “field” describe the demonstrated addressing relationships. They do not establish an application-level data model.

## Evidence

The supplied static body establishes:

- A signed selector and an unsigned value input.
- A read of a shared global anchor value.
- Selector-relative entry addressing at a 396-byte stride.
- A 32-bit entry read supplying the destination object address.
- A destination offset of 1,036 bytes.
- Restriction of the input to 13 low bits.
- A full 32-bit destination assignment.
- Return without an explicit value.

The address dependencies establish the sequence from global-anchor read to entry-word read to destination write. They do not impose an additional ordering relationship with value masking.

No conditional validation, loop, external call, ownership operation, or explicit error handling is visible. No runtime tests have been performed. All acceptance examples are static consequences conditioned on valid memory and the stated data model.
