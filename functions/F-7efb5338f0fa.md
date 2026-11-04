<!-- Trusted provenance: F-7efb5338f0fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7efb5338f0fa

## Purpose

F-7efb5338f0fa conditionally consumes two bytes using a cursor and boundary held in a caller-supplied state object. When its comparison succeeds, it reads the later byte first, reads the earlier byte second, advances the stored cursor by two, and returns a 32-bit value with the earlier byte shifted left by eight bits. When the comparison fails, it returns zero and writes the working cursor back unchanged.

The operation resembles a two-byte big-endian read, but that description is incomplete:

- The earlier byte is read as a plain character, whose signedness is not established.
- The boundary comparison is unsigned, but the preceding cursor-plus-one addition is signed.
- A failed comparison still causes a cursor-field write.
- A diagnostic dependency may run before the working cursor and boundary are read.

This contract distinguishes established static behavior from inferred intent and unresolved execution properties. No runtime tests have been performed.

## Interface

### Input and object layout

The function takes one argument represented as a signed integer and uses it as an object address. The supplied representation indicates 32-bit integer and address values.

The directly accessed object fields are:

| Offset from object base | Width | Interpretation |
|---|---:|---|
| 0x20 | 4 bytes | Cursor, loaded and stored through a signed integer representation |
| 0x24 | 4 bytes | Boundary, loaded as an unsigned integer |

These offsets and widths describe the relevant storage, not the complete object. Total object size, alignment, allocation origin, lifetime, and other fields are not established.

A nonzero object address does not establish that either field is accessible. Likewise, a nonzero cursor does not establish that its target bytes are readable.

### Result

The result is a 32-bit unsigned integer.

Subject to valid accesses and defined arithmetic:

| Decision | Result | Cursor-field write |
|---|---|---|
| Unsigned cursor-plus-one candidate is strictly below the boundary | Combination of the two loaded bytes, with plain-character signedness affecting the earlier byte | Working cursor plus two |
| Candidate is not strictly below the boundary | Zero | Working cursor unchanged |

The function supplies no separate success indicator. Consuming two zero bytes and rejecting the consumption both produce zero; their cursor effects differ.

### Preconditions for ordinary defined use

The caller must provide storage that remains valid for all accesses actually reached:

- The object must support the relevant field reads and the cursor-field write.
- On a successful comparison, both source-byte addresses must be readable.
- Signed arithmetic used for field addressing, candidate formation, cursor advancement, and byte addressing must be valid under the execution model.
- Any diagnostic dependency that returns must leave the subsequent accesses possible if normal completion is expected.

A nonzero initial object address and preliminary cursor avoid the diagnostic call. They are not comprehensive memory-safety checks.

No synchronization, atomicity, or protection against concurrent mutation is established.

### Ownership

No allocation, deallocation, ownership transfer, or retained reference is directly visible in this function. It reads and updates supplied storage in place.

The diagnostic dependency's ownership behavior is unknown. Its invocation does not establish that the object or source storage remains alive afterward.

## Behavior

### Preliminary validation observation

If the supplied object address is nonzero, the function first reads the cursor field at offset 0x20. If the object address is zero, that preliminary field read is skipped.

F-540af0155702 is invoked when either:

- The supplied object address is zero; or
- The preliminary cursor read yields zero.

A nonzero object address with a nonzero preliminary cursor bypasses this dependency.

The preliminary cursor is used only for this diagnostic decision. It is not the authoritative cursor for subsequent consumption: the function reads the cursor field again after the diagnostic branch, including when no diagnostic call occurred.

### Diagnostic arguments and continuation

The diagnostic call receives three arguments:

1. A fixed diagnostic reference whose visible naming suggests assertion-message formatting intent.
2. A value obtained from one global data location.
3. A value obtained from another global data location.

The two global values are read on the diagnostic path to supply arguments. They are not established as immutable constants. Their widths, types, contents, and mutability are not supplied. No evaluation order between those two argument reads is established.

The fixed diagnostic reference must be distinguished from the two values read from global storage. The body does not establish what the dependency reads through that reference or how it uses any argument.

There is no explicit local return or second validation after the call. Its return value is not used.

Consequently:

- If F-540af0155702 does not return, this path produces no normal result from F-7efb5338f0fa.
- If it returns, execution continues using the original object address.
- A zero object address leads to a subsequent field-access attempt through that address; the diagnostic call does not itself provide a safe fallback.
- A nonzero object with an initially zero cursor may have a different cursor after the dependency returns. The subsequent read governs consumption.
- The dependency may change other memory or invalidate storage. No preservation or lifetime guarantee is established.

### Working field reads

After the diagnostic branch, the function reads:

1. The four-byte cursor field at offset 0x20.
2. The four-byte unsigned boundary field at offset 0x24.

These observations define the **working cursor** and **sampled boundary** for the rest of the operation.

The boundary is sampled after the cursor. The function does not obtain an atomic pair, reread the boundary later, or use the preliminary cursor to override the working cursor.

The comparison is evaluated at multiple points in the body, but those evaluations use the same captured candidate and sampled boundary. They do not introduce additional object-field reads.

### Candidate formation and comparison

The function adds one to the working cursor using signed integer arithmetic. The result is then assigned to a 32-bit unsigned value.

The distinction between addition and conversion is material:

- If the signed addition is defined, conversion produces the corresponding unsigned value.
- A defined negative result converts to its corresponding value in the unsigned 32-bit range.
- If the signed addition overflows, the supplied representation does not establish the outcome. Assignment to an unsigned destination does not retroactively make the addition unsigned or guarantee wraparound.

The function compares the unsigned candidate with the sampled unsigned boundary using strict less-than. Equality rejects consumption.

For ordinary nonwrapping addresses, this resembles checking that the second of two bytes is below an exclusive end address. That interpretation is inferred intent. The established rule is the signed addition, unsigned conversion, and strict unsigned comparison.

The function does not independently establish that:

- The working cursor is below the boundary.
- Either source address belongs to a valid allocation.
- The two bytes and boundary belong to the same buffer.
- Signed arithmetic cannot overflow.
- The comparison is sufficient to make the subsequent accesses safe.

### Successful consumption

If the comparison succeeds, the function advances its working cursor by two using signed integer arithmetic.

It then loads:

1. An unsigned byte at one byte before the advanced cursor.
2. A plain character at two bytes before the advanced cursor.

When the arithmetic is defined, these are respectively the original working cursor plus one and the original working cursor. Thus the later byte is read first, although it occupies the low-byte position in the result.

Both loaded values are converted to the 32-bit unsigned result representation. The function then writes the advanced cursor to object offset 0x20.

After that cursor-field store, it forms the result by shifting the converted earlier-byte value left by eight bits and combining it with the unsigned later-byte value using bitwise OR.

There is no final mask restricting the result to 16 bits.

The signed two-byte advance has its own overflow uncertainty, separate from candidate formation. A defined cursor-plus-one calculation does not establish that cursor-plus-two is also defined.

### Rejected consumption

If the comparison fails:

- Neither source byte is loaded.
- The working cursor is written back to offset 0x20 without advancement.
- The result is zero.

This is an unchanged-value store, not an absence of writes. Its distinction can matter for instrumented storage, concurrent access, or memory with special access semantics, none of which are otherwise specified here.

### Byte and result semantics

The later-byte load is explicitly unsigned and contributes a value from 0 through 255 in the represented eight-bit-byte model.

The earlier-byte load uses a plain character:

- With unsigned plain-character interpretation, the result is the conventional 16-bit combination, represented in 32 bits.
- With signed eight-bit plain-character interpretation, an earlier byte from 0x80 through 0xFF becomes negative before conversion to unsigned. That conversion supplies upper bits that remain relevant after the eight-bit left shift.

For example, earlier byte 0x80 and later byte 0x01 yield:

| Plain-character interpretation | Result |
|---|---:|
| Unsigned | 0x00008001 |
| Signed, eight-bit | 0xFFFF8001 |

The signedness uncertainty concerns the earlier-byte load, not the explicitly unsigned later-byte load.

## Side effects

On a normally completed path without a diagnostic call, the direct access sequence is:

1. Preliminary cursor-field read.
2. Working cursor-field read.
3. Boundary-field read.
4. If the comparison succeeds, later-byte read followed by earlier-byte read.
5. Cursor-field write.

If the diagnostic path is taken, the two global argument values are read and F-540af0155702 is invoked before the working field reads. No relative order between the two global reads is established. A zero object address suppresses only the preliminary cursor-field read.

The cursor field is the only object field directly written. There are no separate stores through the source-byte load addresses.

That does not guarantee that source storage remains unchanged. If either consumed byte overlaps the cursor field, the cursor-field store can modify the same storage. Both successful-path byte loads precede that store, so the returned value uses the previously loaded values rather than bytes reloaded after the update.

Effects of invalid memory accesses, special memory mappings, or failed stores are environment-dependent. No explicit recovery is present.

The dependency may introduce additional memory, global-state, ownership, diagnostic, or control-flow effects. Those effects are not bounded by this function's direct access list.

## Dependencies

### F-540af0155702

| Property | Contract |
|---|---|
| Invocation condition | Zero object address or zero preliminary cursor |
| First argument | Fixed diagnostic reference; assertion-style intent is inferred |
| Second and third arguments | Values read from two distinct global data locations |
| Global argument widths and contents | Not established |
| Ordering of the two global argument reads | Not established |
| Return value | Unused |
| Return behavior | Unknown |
| Memory and global preservation | Not established |
| Ownership and storage release | Not established |

If the dependency returns, the function rereads the cursor and then reads the boundary. No assumption that the dependency preserves either field is necessary or justified.

There are no other explicit function dependencies in the supplied body.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except where stated otherwise, assume stable valid storage, a writable state object, no overlapping source and cursor storage, and defined signed arithmetic.

### Two available bytes

Given a nonzero working cursor of 0x1000, boundary 0x1002, and earlier/later bytes 0x12 and 0x34:

- Signed cursor-plus-one produces 0x1001.
- The unsigned comparison succeeds.
- Address 0x1001 is read before address 0x1000.
- The cursor field is written with 0x1002.
- The result is 0x00001234 under either plain-character signedness.

### Equality at the candidate

Given working cursor 0x1000 and boundary 0x1001:

- Candidate and boundary are equal, so consumption is rejected.
- Neither source byte is read.
- The cursor field is written with 0x1000.
- The result is zero.

### Cursor equal to boundary

Given working cursor and boundary both equal to 0x1000:

- Candidate 0x1001 is not below the boundary.
- The cursor is written back unchanged.
- No source bytes are read.
- The result is zero.

### Zero result after successful consumption

Given working cursor 0x1000, boundary 0x1002, and two zero bytes:

- Consumption succeeds.
- The cursor advances to 0x1002.
- The result is zero.

A caller cannot distinguish this result from rejection using the return value alone.

### Earlier byte with its high bit set

Given sufficient range and earlier/later bytes 0xFF and 0x7E:

- Both bytes are loaded before the cursor update.
- The cursor advances by two.
- The result is 0x0000FF7E with unsigned plain characters.
- The result is 0xFFFFFF7E with signed eight-bit plain characters.

A unique expected numerical result requires resolving plain-character signedness.

### Returning diagnostic dependency

Given a nonzero object whose preliminary cursor is zero:

- Both global diagnostic argument values are read, in an unspecified relative order.
- F-540af0155702 is invoked with the diagnostic reference and those values.
- If it returns, the cursor and boundary are read afterward.
- If those later fields describe valid storage, they determine subsequent consumption.

This example does not assert that the dependency repairs the object or preserves its lifetime.

### Null object

Given a zero object address:

- No preliminary cursor-field read occurs.
- The diagnostic argument reads and dependency invocation occur if argument evaluation completes.
- If the dependency returns, a cursor-field access through the original zero object address is attempted.

A clean zero return is not established.

### Signed arithmetic boundary

Given a working cursor equal to the largest signed 32-bit integer, candidate formation requires a signed addition outside that type's range. No wrapped candidate or normal result is specified by this contract.

Given a working cursor one below that maximum, candidate formation can be defined while a successful two-byte advance overflows. A boundary comparison that succeeds does not resolve that separate uncertainty.

## Uncertainties

The following remain **documented_only**:

- Plain-character signedness and results affected by high-bit earlier bytes.
- The diagnostic reference's exact contents and the dependency's interpretation of it.
- The widths, types, contents, and mutability of the two global argument values.
- The relative evaluation order of those global reads.
- F-540af0155702's return, termination, exception, diagnostic, memory, global-state, and ownership behavior.
- Execution behavior when signed arithmetic overflows, including cursor-plus-one and cursor-plus-two.
- Validity of integer-based object and byte addressing outside ordinary address ranges.
- Object alignment, allocation, lifetime, and buffer ownership.
- Concurrency, atomicity, volatile or memory-mapped access behavior, and invalid-access handling.

The exclusive-end boundary interpretation and two-byte decoding purpose are inferred intent. Neither overrides the established arithmetic, comparison, access order, or signedness uncertainty.

## Evidence

Established evidence is limited to the supplied function body and opaque dependency mapping. It supports:

- Conditional preliminary cursor observation.
- Diagnostic invocation with one reference and two values read from global data locations.
- A fresh cursor read followed by a boundary read.
- Signed cursor-plus-one arithmetic before unsigned assignment and comparison.
- Conditional signed cursor advancement by two.
- Later-byte-first loading, with unsigned later-byte and plain-character earlier-byte interpretation.
- A cursor-field store on both comparison outcomes.
- Successful-path byte loads before that store and result assembly afterward.
- No separate source-address stores and no final 16-bit result mask.

No runtime tests, dependency inspection, or independent execution observations have been performed.
