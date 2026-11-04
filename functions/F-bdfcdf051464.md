<!-- Trusted provenance: F-bdfcdf051464; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdfcdf051464

## Purpose

F-bdfcdf051464 reads selected flags from byte-addressed storage and returns an unsigned mask in a different flag layout. Most selected input bits contribute independently. Two groups collapse multiple input flags into shared output contributions:

- Either of offset 1 bits 6 and 7 sets output bits 3 and 4 together.
- Either of offset 1 bits 0 and 1 sets output bit 11.

Flag extraction and remapping are established by the supplied body. The meanings of the flags, the surrounding record format, and the intended consumers of the result are not established.

The body contains no explicit validation, writes to external storage, allocation, or dependency calls.

## Interface

### Input and layout

The function accepts one pointer to byte-addressed storage. There is no length argument.

All offsets below are zero-based byte offsets from that pointer. The body explicitly accesses offsets **0, 1, 3, and 5**. It contains no explicit access to offsets 2 or 4, or beyond offset 5.

A sufficient ordinary-use precondition is an accessible object containing at least six bytes, with the relevant storage readable throughout execution. This is a sufficient storage requirement, not evidence that the underlying record is exactly six bytes long.

The body performs no explicit null, bounds, alignment, or accessibility checks. No defined invalid-input result is established.

The input is examined through byte accesses and masks, not interpreted as a multibyte integer. The mapping therefore does not require a byte-order conversion. No wider-load alignment requirement is expressed in the supplied body.

### Output and widths

The return value is an unsigned integer mask with these established value properties:

- Only bits 0 through 11 can be set.
- Higher result bits, if present in the return type, are zero.
- The minimum result is `0x000`.
- The maximum result is `0xFFF`, or 4095.
- Output bits 3 and 4 are always equal.

The definition of the unsigned return-type alias is absent. Its exact width and ABI representation remain **documented_only**; a specific width such as 32 bits is not established.

The mask calculation uses bit selection and bitwise union. There is no arithmetic addition, multiplication, or shifting. All possible results fit within 12 value bits, so no overflow or truncation is needed to explain the established mapping.

The formal definition of the input byte type is also absent. The contract describes the selected low-order bits and byte offsets shown by the supplied body; target-specific type and addressable-unit details remain unresolved.

### Ownership and lifetime

Input ownership is not established. The supplied body does not write through, retain, allocate, or release the input pointer or storage.

The pointer may designate caller-owned, borrowed, or externally owned storage; the body does not distinguish these cases. The relevant storage must remain valid for the reads. This accessibility requirement does not establish ownership or transfer it.

No pointer or reference to the input is returned. The result is a value independent of the input storage after the reads complete.

## Behavior

### Complete mapping for stable input

When the relevant bytes remain unchanged throughout execution, the result is the bitwise union of the following contributions. A false condition contributes zero.

| Input offset | Selected condition | Output contribution |
|---|---|---:|
| 0 | Bit 6 is set; mask `0x40` | `0x004` |
| 1 | Bit 4 is set; mask `0x10` | `0x020` |
| 1 | Bit 5 is set; mask `0x20` | `0x040` |
| 0 | Bit 0 is set; mask `0x01` | `0x080` |
| 0 | Bit 1 is set; mask `0x02` | `0x100` |
| 1 | Either bit 6 or bit 7 is set; mask `0xC0` | `0x018` |
| 3 | Bit 0 is set; mask `0x01` | `0x001` |
| 3 | Bit 1 is set; mask `0x02` | `0x002` |
| 5 | Bit 1 is set; mask `0x02` | `0x200` |
| 1 | Bit 2 is set; mask `0x04` | `0x400` |
| 1 | Bit 1 is set; mask `0x02` | `0x800` |
| 1 | Bit 0 is set; mask `0x01` | `0x800` |

The two conditions contributing `0x800` do not accumulate numerically. Either condition, or both together, contributes exactly `0x800`, never `0x1000`.

Likewise, the `0xC0` selection at offset 1 is a nonzero test of the combined mask. Either selected bit suffices to contribute `0x018`; both together still contribute only `0x018`. No other condition contributes to output bits 3 or 4, so those bits cannot differ.

There is no normal early return based on any flag value. In particular, zero values at offsets 0 and 1 do not bypass the later reads.

### Relevant and ignored bits

For stable input, the relevant masks are:

| Offset | Relevant mask |
|---|---:|
| 0 | `0x43` |
| 1 | `0xF7` |
| 3 | `0x03` |
| 5 | `0x02` |

Other bits at those offsets do not affect the returned value. Offset 1 bit 3 is ignored.

Changing only offsets 2 or 4 does not affect the established result. No explicit access to either offset appears in the body. This does not establish the widths of physical machine loads after compilation.

The mapping is information-losing. A result containing `0x018` does not identify which of offset 1 bits 6 and 7 was set. A result containing `0x800` does not identify which of offset 1 bits 0 and 1 was set. Ignored bits cannot be recovered from the result.

### Access order and retained values

The explicit source-level input access sequence is:

| Access position | Offset | Use |
|---|---:|---|
| First | 0 | Retained for all decisions concerning offset 0 |
| Second | 1 | Retained for all decisions concerning offset 1 |
| Third | 3 | Tests bit 0 |
| Fourth | 3 | Tests bit 1 |
| Fifth | 5 | Tests bit 1 |

The offset 0 and offset 1 values are retained before the later input reads. Contributions based on offset 0 and on offset 1 bits 4 through 7 are determined before the offset 3 accesses. The remaining contributions from the retained offset 1 value—bits 2, 1, and 0—are incorporated after the offset 5 access.

Offset 3 occurs in two separate read expressions. The body does not explicitly retain a single offset 3 value for both tests. That distinction is static evidence, not a promise that two physical reads will occur.

These observations describe the supplied body's access structure. The input is not shown as volatile, and no compiler, target, or memory model is supplied. Consequently, the contract does not promise a particular emitted load sequence, transaction count, atomic snapshot, or behavior under concurrent mutation.

### Errors and exceptional conditions

There is no explicit error value, diagnostic, exception mechanism, or recovery path.

Null, dangling, insufficiently accessible, or otherwise invalid storage does not satisfy the ordinary-use precondition. The supplied body establishes no defined result for those cases. In particular, a zero result is an ordinary flag result, not an established indication of failure.

## Side effects

For valid, stable ordinary memory, the only established externally visible result is the returned mask.

The supplied body contains no explicit:

- Writes to input or other external storage.
- Global or static state accesses.
- Allocation or release of memory.
- I/O.
- Calls to other functions.
- Retention or publication of the input pointer.

The internal accumulation of result bits does not alter the input.

Reads can themselves have effects for special memory, but the supplied evidence does not establish device-register behavior, read-to-clear semantics, synchronization, or atomicity. The function must not be treated on this evidence alone as a safe device-access primitive or a coherent snapshot operation.

## Dependencies

No callable dependencies appear in the supplied body. The supplied dependency map is empty, so no additional opaque function IDs apply.

Environmental prerequisites remain: readable input storage, definitions of the byte and unsigned return types, and execution semantics for the accesses. There is no unknown callee whose global-state preservation, pointer retention, or memory ownership behavior needs to be assumed.

## Acceptance examples

These are conditional expectations derived from static inspection, not executed tests. Each tuple lists hexadecimal byte values at offsets 0 through 5. All examples assume valid, stable ordinary memory and the byte interpretation described above.

| Input bytes | Expected result | Property illustrated |
|---|---:|---|
| `00 00 00 00 00 00` | `0x000` | No selected input flags |
| `40 00 00 00 00 00` | `0x004` | Offset 0 bit 6 maps to output bit 2 |
| `03 00 00 00 00 00` | `0x180` | Offset 0 bits 0 and 1 contribute independently |
| `00 30 00 00 00 00` | `0x060` | Offset 1 bits 4 and 5 contribute independently |
| `00 40 00 00 00 00` | `0x018` | Offset 1 bit 6 sets both grouped output bits |
| `00 80 00 00 00 00` | `0x018` | Offset 1 bit 7 produces the same contribution |
| `00 C0 00 00 00 00` | `0x018` | Both grouped input bits add no further contribution |
| `00 01 00 00 00 00` | `0x800` | Offset 1 bit 0 sets the shared output bit |
| `00 02 00 00 00 00` | `0x800` | Offset 1 bit 1 sets that same output bit |
| `00 03 00 00 00 00` | `0x800` | Shared output is not counted twice |
| `00 04 00 00 00 00` | `0x400` | Offset 1 bit 2 has a separate contribution |
| `00 00 00 03 00 02` | `0x203` | Offset 3 and offset 5 contributions combine |
| `BC 08 FF FC FF FD` | `0x000` | Ignored bits and unexamined bytes do not contribute |
| `43 F7 00 03 00 02` | `0xFFF` | Maximum result |
| `FF FF FF FF FF FF` | `0xFFF` | Additional ignored bits do not exceed the maximum |

Additional conditional acceptance properties:

- Changing only ignored bits leaves the result unchanged.
- No stable input produces output bit 3 without bit 4, or bit 4 without bit 3.
- All-zero selected flags produce zero regardless of ignored bits.
- No valid flag combination is rejected through a separate error path.
- Borrowed readable storage produces the same mapping as other readable storage; the body performs no ownership-dependent operation.

## Uncertainties

### Established static evidence

The supplied body establishes the accessed offsets, selected masks, output contributions, bitwise union, retained offset 0 and offset 1 values, and two separate offset 3 read expressions. It also establishes the absence of explicit writes and callable dependencies.

It does **not** establish an initial owner of the input storage.

### Inferred intent

The function appears intended to normalize selected flags into another layout. This interpretation does not establish flag names, domain meanings, ownership rules, or a complete containing-record definition.

### documented_only

The following remain unresolved:

- Exact definitions and ABI widths of the input byte and unsigned return types.
- Target addressable-unit properties.
- Compiled load widths, physical read count, and machine-level ordering.
- Behavior under concurrent mutation or asynchronous hardware changes.
- Special-memory read effects and synchronization requirements.
- Defined behavior for invalid or inaccessible input.
- Initial ownership and broader lifetime obligations of the storage.
- Additional layout or validity requirements imposed by callers.

The source-level access sequence does not resolve these machine-level or environmental questions.

## Evidence

This contract is based solely on static inspection of the supplied body for F-bdfcdf051464 and the supplied empty dependency map.

The mapping, result bounds, grouped-bit invariant, ignored-bit properties, and acceptance examples are deductions from the explicit accesses, masks, and bitwise combinations. The ownership statement is limited to operations absent from the body; it does not infer who owns the storage.

No runtime tests, implementation execution, ABI inspection, or machine-code verification have been performed.
