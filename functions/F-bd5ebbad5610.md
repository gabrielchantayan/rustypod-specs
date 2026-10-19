<!-- Trusted provenance: F-bd5ebbad5610; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd5ebbad5610

## Purpose

F-bd5ebbad5610 writes the low 16 bits of its second input into two byte locations associated with a global base value, then returns a 32-bit zero on normal completion.

In the ordinary case where the global base remains stable, the destinations are offsets 10 and 11 from that base. Offset 10 receives input bits 0–7; offset 11 receives input bits 8–15. This gives the two-byte field a least-significant-byte-first layout independently of the machine’s native multi-byte byte order.

The supplied static representation establishes the byte writes, their order, and the normal return value. It does not establish the meaning of the containing object, the role of this two-byte field, or any synchronization protocol.

## Interface

### Inputs

| Input | Width represented | Established use |
|---|---:|---|
| First positional input | 32 bits | Not referenced by the function body. |
| Second positional input | 32 bits | Supplies the two byte values written to memory. |

The inputs have no supplied semantic type beyond their four-byte representation. In particular, the second input is not established as a length, identifier, status, or signed quantity.

The function also depends on a global storage location containing an address-like base value. That global is an implicit input, not a positional argument. Its value determines the destination addresses.

### Output

Normal completion returns a four-byte value whose value and bit pattern are zero.

The return is unconditional along the shown normal execution path. It is not a reported result of address validation, memory availability, or a separate operation. No alternate error return is shown.

### Destination layout

For a stable base value, the relevant layout is:

| Byte offset from base | Written value |
|---:|---|
| 10 | Low eight bits of the second input |
| 11 | Bits 8–15 of the second input |

Offsets are byte offsets. The writes are individual byte writes, not an established single 16-bit write.

No destination byte is read to compute its replacement. The function does not preserve or merge selected bits of either destination byte; each byte receives the full corresponding eight-bit value.

### Preconditions for ordinary completion

The global base storage must be readable. The two effective destination addresses used during the call must be writable for one byte each, and their address calculations must be valid in the execution environment.

An object providing writable bytes through offset 11 is sufficient when the base is stable and addresses the beginning of that object. The function does not check object size, pointer validity, permissions, or lifetime.

No byte-order precondition is needed to explain the explicit byte layout. No alignment check appears in the supplied body. Platform-specific addressing and access requirements remain undocumented.

## Behavior

### Address observations and write order

The supplied static representation distinguishes a captured base from the base used to address the first write:

1. The global base value is read and retained.
2. The first destination address is formed using another appearance of the global base, at byte offset 10. The low byte of the second input is written there.
3. The second destination address is formed from the retained base, at byte offset 11. The next byte of the second input is written there.
4. The function returns zero.

Thus, the base retained before either write determines the second destination. The first destination is expressed using the global base again rather than exclusively using that retained value.

When the base is stable, these observations identify adjacent bytes of the same region. The supplied representation does not justify extending that guarantee to an environment where the global can change between observations.

This is a source-facing ordering description, not evidence of a particular machine instruction sequence. In particular, the representation does not establish whether an actual build performs two physical global loads, whether a global load is volatile, or what another execution context can observe.

### Value extraction

Only the low 16 bits of the second input affect the written data.

The first written byte is the input truncated to eight bits. The second is obtained by interpreting the input as an unsigned 32-bit value, shifting it right by eight bit positions, and retaining the low eight bits.

Consequently:

- Bits 0–7 determine the first byte.
- Bits 8–15 determine the second byte.
- Bits 16–31 do not affect either stored byte.
- An interpretation of the input as negative does not cause sign extension in the explicitly unsigned shift.

The byte values are described as raw eight-bit patterns. Whether a surrounding type system would display a byte as signed or unsigned does not change the intended bit selection visible in this representation.

### Address arithmetic

The only shown destination displacements are 10 and 11 bytes. There is no input-dependent displacement, indexing, multiplication, or bounds calculation.

The supplied body uses a four-byte integer-like temporary for the retained base. This supports describing the representation as using a 32-bit base quantity; it does not establish a portable rule for address overflow or signed arithmetic overflow. Ordinary behavior should therefore be specified for valid destination calculations rather than by inventing wraparound semantics near an address-space boundary.

### Unused first input

Changing only the first input does not change any operation explicitly shown in the body.

This is a property of the function body, not a claim that every surrounding calling convention accepts arbitrary argument encodings or that caller-side evaluation has no effects.

## Side effects

On normal completion, the function performs two explicit byte-write operations in order: low byte first, next byte second. It does not explicitly assign to the global base storage.

The shown body performs no allocation, deallocation, resource acquisition, resource release, callback, or external function call. It does not transfer ownership of either destination or of any object containing them. Storage validity and lifetime remain obligations of the surrounding environment.

The two writes are not established as an atomic pair. No lock, fence, transaction, rollback, or synchronization action is shown. The contract therefore does not promise that another observer can see only the complete old value or complete new value.

There is also no explicit guarantee that the destination memory is ordinary non-aliased storage:

- A destination might overlap the storage holding the global base.
- Either destination might have platform-specific write effects.
- Different base observations might produce destinations that are not adjacent or even distinct.

If such aliasing occurs, the shown order still matters. In particular, the second destination is based on the value retained before the first write, not on an explicit re-read after that write.

For ordinary writable memory with a stable base, bytes outside offsets 10 and 11 are not explicitly written by this function. That statement does not rule out secondary effects of special memory mappings or external activity.

No failure handling is present. If an access faults or otherwise prevents execution from continuing, the supplied body does not establish a return value, recovery action, or rollback. If the first write has completed before the second fails, there is no shown mechanism to undo the first.

## Dependencies

No callable dependencies are present in the supplied body. There are therefore no dependency function IDs to list.

The non-call dependencies are:

- Readable global storage containing the base value.
- Writable storage at the two effective destination addresses.
- Execution-environment rules for integer-to-address interpretation and byte stores.

No callee behavior is needed to explain this function. No assumptions about another function preserving globals or owning memory are introduced.

## Acceptance examples

These examples are conditional deductions from the supplied static representation. They are not runtime test results.

Unless otherwise stated, assume a stable base, ordinary writable memory, valid address arithmetic, and normal completion.

| Second input, as a 32-bit pattern | Byte at offset 10 after the call | Byte at offset 11 after the call | Return |
|---|---|---|---|
| `0x00000000` | `0x00` | `0x00` | `0x00000000` |
| `0x00001234` | `0x34` | `0x12` | `0x00000000` |
| `0xABCD1234` | `0x34` | `0x12` | `0x00000000` |
| `0x00000100` | `0x00` | `0x01` | `0x00000000` |
| `0xFFFF8000` | `0x00` | `0x80` | `0x00000000` |
| `0xFFFFFFFF` | `0xFF` | `0xFF` | `0x00000000` |

Additional conditional criteria:

- **Replacement, not merging:** If the two destination bytes initially contain `0xAA` and `0x55`, an input of `0x00001234` replaces them with `0x34` and `0x12`.
- **Upper-bit independence:** Two second-input patterns that agree in their low 16 bits produce the same two written byte values.
- **First-input independence:** With the same second input and equivalent global and memory conditions, varying only the first input does not alter the explicit writes or normal return.
- **Write ordering:** Under an observation mechanism that exposes the two byte stores in their represented order, the low-byte write precedes the next-byte write. A single atomic 16-bit update is not an acceptance requirement.
- **Changing base:** If the retained base is A and the global value used for the first destination is B, the represented destinations are B plus 10 for the first write and A plus 11 for the second. This is conditional on the execution environment permitting and exposing those distinct observations.
- **Invalid second destination:** If the first write completes but the second cannot, a zero return and restoration of the first byte are not guaranteed.

## Uncertainties

The following are **documented_only**:

- The semantic purpose of the first input and of the two-byte destination field.
- The declaration, initialization, and ownership of the global base storage.
- The full destination-object layout and allocation size.
- Exact ABI details beyond the four-byte input and return representations shown.
- Native pointer width and precise address-overflow behavior.
- The signedness of the byte type and its implementation-specific conversion rules; the contract describes the visible eight-bit storage patterns.
- Whether the global is volatile, concurrently modified, or subject to special memory semantics.
- Whether the destinations alias the global storage or other significant state.
- Physical load and store instructions, atomicity, and cross-context visibility.
- Fault behavior and any recovery imposed outside this function.

An inferred intent is to update a two-byte, least-significant-byte-first field in a globally selected object. That interpretation is plausible but not required to explain the established operations.

## Evidence

The evidence is the single supplied static function body. It shows two four-byte positional inputs, one unused input, a retained global base value, two byte assignments at offsets 10 and 11, an unsigned right shift by eight for the second stored value, and a constant four-byte zero return.

The body shows no conditional branch, loop, validation, external call, allocation, or explicit synchronization.

All behavioral conclusions above are static deductions subject to the stated environmental conditions. No runtime tests have been performed.
