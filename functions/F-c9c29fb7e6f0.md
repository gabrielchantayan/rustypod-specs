<!-- Trusted provenance: F-c9c29fb7e6f0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9c29fb7e6f0

## Purpose

F-c9c29fb7e6f0 queries one bit in a byte reached through an indirectly supplied base address and an integer offset. Its observable result is Boolean: true when the selected bit intersects the loaded byte, false otherwise.

The bit selector is reduced to its low eight bits before it is used as a shift count. It is **not** reduced to a byte-relative index in the range zero through seven. Consequently, selectors whose low eight bits exceed seven require separate treatment; they do not select a bit from another byte.

This contract distinguishes directly supported behavior from behavior that depends on the underlying integer and shift model. No runtime tests have been performed.

## Interface

The function has three inputs and one Boolean result.

| Input | Supplied form | Role |
|---|---|---|
| Address holder | Pointer to an integer | Identifies the storage location from which the base address is read. |
| Offset | Signed integer | Added directly to the loaded base-address value to identify the byte to read. |
| Selector | Unsigned integer | Supplies a bit-selection value; only its low eight bits participate in the shift count. |

The address holder is not itself the byte buffer. The function first reads an integer from the address holder, then uses that integer as the base of the byte address calculation.

The offset is a byte-address displacement, not an index scaled by the size of an integer. Negative offsets are not explicitly rejected.

The result conveys only whether the byte and the generated mask have a nonzero intersection. There is no separate status result, returned address, returned byte, or output parameter.

### Widths and representation

The supplied representation identifies signed and unsigned integer types, an unsigned byte-sized load, and a Boolean return. It does not independently establish the target ABI.

The byte interpretation used here is an unsigned eight-bit quantity, ranging from 0 through 255. The selector reduction retains eight bits, giving an effective shift count from 0 through 255.

A conventional 32-bit interpretation of the integer types makes the address-holder read 32 bits wide, the offset a signed 32-bit value, and the selector an unsigned 32-bit value. Those widths are consistent with the supplied representation but are not independently verified by architecture or ABI evidence.

The address is stored as an integer rather than as a separately established pointer-width object. Whether that integer can represent every valid address is therefore an environmental constraint, not a guarantee of this function.

## Behavior

### Supported operation

For a call that completes normally, the function performs the following logical work:

1. Reads the integer stored at the address holder.
2. Forms the byte address by adding the signed offset to that value.
3. Reads one unsigned byte at the resulting address.
4. Forms a bit mask by shifting the integer value one left by the selector’s low-eight-bit value.
5. Intersects the byte’s unsigned integer value with that mask.
6. Returns true if the intersection is nonzero, and false if it is zero.

This is a description of externally relevant data flow, not an assurance that every intermediate computation has a defined outcome for every possible input.

The integer read must supply the address used by the byte read, so that dependency establishes their order. The mask calculation does not depend on either memory value. The supplied expression does not establish whether the mask is calculated before, after, or between those reads.

### Address selection

The offset is applied before the byte load. No additional displacement is derived from the selector.

In particular, a selector of eight does not advance the address to the following byte. A selector of nine does not mean bit one of the following byte. Both still refer to the same computed byte address, subject to the shift qualifications below.

The function contains no explicit bounds check, address-range check, null check, or alignment check. Callers must provide an address holder that can be read as the required integer object and an effective address from which one byte can be read.

The integer address addition and its conversion into a usable address must also be meaningful in the execution environment. The evidence does not establish wrapping, overflow detection, address truncation, or recovery behavior.

### Selector reduction

Only the selector’s low eight bits affect the effective shift count. Values with the same low eight bits therefore request the same mask calculation.

For example:

- Selector 0 and selector 256 both have effective count 0.
- Selector 7 and selector 263 both have effective count 7.
- Selector 255 has effective count 255, not 7.

The final example is important: discarding all but the low eight bits is not equivalent to limiting a selection to the eight bits of the loaded byte.

### Effective counts zero through seven

For an effective count from zero through seven, the function tests the corresponding bit of the loaded byte.

Bit zero is the least significant bit. Bit seven is the most significant bit of the eight-bit byte.

For these counts, the answer depends only on the selected bit. Other bits in the same byte do not change the result. A zero byte returns false for every count in this range. A byte with all eight bits set returns true for every count in this range.

### Larger effective counts

When an integer shift is defined and produces a single bit above bit seven, the unsigned byte cannot intersect that bit. The result is false.

That observation must not be extended into an unconditional promise for all effective counts from eight through 255. The mask is formed from the integer literal one, and its behavior depends on integer width, signed-shift rules, and the actual execution model.

Under conventional 32-bit signed-integer source semantics:

- Effective counts 8 through 30 produce positive masks above the byte’s bit range and therefore return false.
- Effective count 31 raises a signed-shift representability issue.
- Effective counts 32 through 255 meet or exceed the assumed width of the shifted integer.

The supplied expression does not resolve the latter cases. A particular machine instruction might reduce a shift count internally, but no such instruction is supplied as evidence. This contract therefore does not promise hardware-style count wrapping or any particular Boolean result for those cases.

### Reads are not conditional on a useful bit index

The expression contains no explicit early return for a selector outside the byte’s bit range. The byte is an operand of the final intersection even when a valid mask would lie above bit seven.

Accordingly, an out-of-range byte-bit selector is not a supported way to avoid accessing the supplied memory. Actual fault timing or optimized access behavior cannot be established without the executable environment.

## Side effects

The supplied body contains no explicit writes to caller memory, no global-state access, and no function calls.

Its explicit memory accesses are:

1. An integer read through the address holder.
2. A byte read through the address derived from that integer and the offset.

There is no explicit allocation, deallocation, ownership transfer, reference-count adjustment, or retention of an input address. Nothing in the supplied body releases the address holder or the byte storage. Their lifetime and accessibility must cover the relevant reads.

There is no explicit synchronization or atomic-access designation. The function does not establish a coherent snapshot across the two locations. Concurrent modification, volatile storage, memory-mapped devices, or other special memory behavior requires environmental rules not provided here.

Invalid accesses are not converted into false or another documented error result. No explicit exception handling or recovery path is present.

## Dependencies

No opaque function dependencies are visible in the supplied body. F-c9c29fb7e6f0 performs its calculation directly.

Execution nevertheless depends on environmental facilities and conventions:

- Integer widths and signed arithmetic semantics.
- Integer-to-address interpretation.
- Readability and lifetime of the two accessed memory locations.
- Shift behavior for the effective count.
- The ABI representation of the Boolean result.

These are not established callable dependencies and are not assigned invented function IDs.

The order of the integer and byte reads follows their address dependency. No claim about preservation of unrelated global state by unknown callees is needed, because no callees are present.

## Acceptance examples

These examples are conditional on valid, stable memory; meaningful address arithmetic; an unsigned eight-bit byte; and defined shift behavior for the count being exercised. They are acceptance conditions, not reported test results.

| Situation | Required result or observation |
|---|---|
| The selected byte is hexadecimal 01 and the effective count is 0. | Returns true. |
| The selected byte is hexadecimal 01 and the effective count is 1. | Returns false. |
| The selected byte is hexadecimal 80 and the effective count is 7. | Returns true. |
| The selected byte is hexadecimal 80 and the effective count is 0. | Returns false. |
| The selected byte is hexadecimal A5 and the effective count is 2. | Returns true. |
| The selected byte is hexadecimal A5 and the effective count is 1. | Returns false. |
| The selected byte is hexadecimal 00 and the effective count is between 0 and 7. | Returns false. |
| The selected byte is hexadecimal FF and the effective count is between 0 and 7. | Returns true. |

Additional boundary and addressing conditions:

- **Selector truncation:** For the same address holder, offset, and stable memory, selectors 0 and 256 produce the same result. Selectors 7 and 263 likewise produce the same result.
- **No byte traversal:** With an effective count of 8 and a defined mask containing only bit eight, the result is false even if the next byte has its least significant bit set. The next byte is not selected.
- **Offset behavior:** If the holder contains a valid base address and offset one names the next readable byte, the tested value comes from that next byte.
- **Negative offset:** If offset minus one forms a valid readable address without arithmetic ambiguity, the tested value comes from the byte immediately before the base. No explicit rejection is present.
- **Indirect address behavior:** Changing the integer stored in the holder can change which byte is read on a subsequent call. The holder’s own first byte is not the intended target merely because it contains a set bit.
- **Large shift boundary:** Effective counts 31, 32, and 255 have no unconditional expected Boolean result under this contract. Acceptance requires an independently documented execution model before assigning one.
- **Invalid memory:** No particular Boolean result is required when the address holder or computed byte address is unreadable. Returning false is not an established error contract.

## Uncertainties

The following items are **documented_only**:

- **Exact ABI widths:** The conventional 32-bit interpretation is not independently confirmed.
- **Signed shift boundary:** The supplied expression does not establish a portable result when shifting one into or beyond the signed integer boundary.
- **Machine count reduction:** No evidence establishes whether an underlying instruction masks large shift counts.
- **Address overflow and conversion:** Wraparound, truncation, and invalid-address handling are not specified.
- **Fault behavior:** The exception, signal, termination, or other response to an invalid read is unknown.
- **Special memory and concurrency:** Atomicity, ordering beyond the address dependency, and effects of simultaneous modification are not specified.
- **Physical access details:** The expression identifies logical reads, not bus transactions or guaranteed optimized instruction counts.
- **Boolean ABI encoding:** The logical outcomes are established; register width and binary return encoding are not.

The likely intent is a bit-membership query over byte-addressed storage. That intent is inferred. It does not justify replacing the observed eight-bit selector reduction with a three-bit reduction or introducing bounds checks absent from the evidence.

## Evidence

The supplied static representation establishes an integer read through the first input, addition of the second input to form a byte address, an unsigned byte load, a shift count derived from the third input’s low eight bits, a bitwise intersection, and comparison against zero.

It contains no explicit calls, writes, validation branches, allocation operations, or error-return path.

The contract above separates those observations from ABI assumptions and unresolved arithmetic behavior. No disassembly, runtime observations, external dependency contracts, or test results were supplied. No runtime tests have been performed.
