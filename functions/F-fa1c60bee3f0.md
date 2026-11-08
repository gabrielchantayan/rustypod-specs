<!-- Trusted provenance: F-fa1c60bee3f0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa1c60bee3f0

## Purpose

F-fa1c60bee3f0 decodes an identifier and length header from a byte sequence. It reports the identifier’s numeric tag and class, the decoded content length, and flags describing constructed form, indefinite length, and errors.

The behavior is consistent with a BER-style tag-length header decoder. That protocol interpretation is inferred intent, not a guarantee of standards compliance. In particular, the function accepts some nonminimal encodings and has an unusual boundary condition for long-form lengths.

The function advances the caller’s cursor past the header only on its successful parsing path, including the path that reports insufficient content bytes. It does not consume the content.

## Interface

The interface has five arguments, in this order:

| Argument | Role |
|---|---|
| Cursor cell | Mutable cell containing the address of the first header byte |
| Length output | Destination for the decoded content length |
| Tag output | Destination for the decoded tag number |
| Class output | Destination for the identifier’s class bits |
| Available count | Signed count of bytes available starting at the supplied cursor |

The return value is a 32-bit unsigned flag word.

The supplied representation uses 8-bit bytes, 32-bit unsigned outputs, and a cursor address held in a 32-bit integer-sized cell. The available count is a signed 32-bit quantity. These are properties of the supplied representation, not a portable interface declaration.

### Output representations

- **Tag:** an unsigned numeric value, without the identifier’s class or constructed bits.
- **Class:** the first identifier byte masked with `0xC0`. Values are therefore `0x00`, `0x40`, `0x80`, or `0xC0`; they are not shifted down.
- **Length:** a value below `0x80000000` for accepted definite lengths, or zero for indefinite length.
- **Cursor:** the byte immediately following the parsed length field. It is not advanced over content.

Return flags are:

| Bit | Meaning |
|---|---|
| `0x01` | Indefinite-length marker accepted |
| `0x20` | Constructed bit copied from the first identifier byte |
| `0x80` | Error indication |

An ordinary successful definite-length header returns either `0x00` or `0x20`. An accepted indefinite-length header returns either `0x01` or `0x21`. A content-shortage result adds `0x80` to the otherwise applicable flags.

A header-parsing failure returns exactly `0x80`, without preserving the constructed bit.

### Preconditions and ownership

For defined ordinary use:

- The cursor cell must be readable and writable.
- The output cells must be writable whenever execution reaches their assignments.
- The input bytes actually accessed must be readable.
- The available count should be nonnegative.
- Address and count arithmetic must remain representable in the signed arithmetic used by the supplied representation.
- The input and argument cells should remain stable except for the function’s documented writes and dependency effects.

The implementation does not validate pointers, allocate memory, copy the content, or explicitly release caller-owned storage.

Distinct, nonoverlapping input and output storage is the straightforward usage model. The implementation does not enforce it. Aliasing can affect both later reads and the final bounds decision; it must not be treated as harmless.

## Behavior

### 1. Entry and first identifier byte

An available count of zero immediately takes the header-failure path. No input byte is read and no output cell is explicitly written before the diagnostic dependency is called.

Any nonzero count, including a negative count, enters identifier decoding. There is no initial rejection of all negative counts.

The function establishes a local cursor one byte after the supplied starting address and reads the first identifier byte. It extracts:

- The constructed flag from bit `0x20`.
- The initial tag value from bits `0x1F`.
- The class, subsequently written to its output, from bits `0xC0`.

The caller’s cursor cell is not yet updated.

### 2. Tag decoding

If the low five bits are not `0x1F`, they are the complete tag number.

If they are `0x1F`, subsequent bytes encode the tag in seven-bit groups:

- The high bit marks continuation.
- The low seven bits contribute the next group.
- Groups are accumulated most-significant first.
- A byte with its high bit clear terminates the tag.

The tag accumulator uses 32-bit unsigned arithmetic.

For a continuation byte, decoding may continue only if another byte remains in the header budget and the accumulated value is below `0x01000000`. Otherwise, the function takes the header-failure path. This check occurs after incorporating that continuation byte.

Under ordinary nonnegative-count execution, the guard keeps any subsequently accepted terminal accumulation below `0x80000000`. It does not impose a fixed byte-count limit: sufficiently many leading zero groups can pass while bytes remain available.

A terminal tag byte does not itself apply the continuation threshold. For example, an accumulated value of `0x00FFFFFF` may be followed by terminal group `0x7F`, yielding tag `0x7FFFFFFF`.

The function does not require a minimal encoding:

- Extended-tag form may encode a value below 31.
- Leading zero groups are not rejected merely for being redundant.
- A zero tag value is not independently rejected.

A missing terminal byte is a header failure.

### 3. Early output writes

After the tag is complete, at least one byte must remain for the initial length byte. If none remains, the function fails before explicitly writing the tag or class outputs.

Otherwise, it writes:

1. The tag output.
2. The class output.

These writes precede length validation. A later length failure therefore does not roll them back.

### 4. Length decoding

The function reads the first length byte after those two output writes.

#### Indefinite length

A first length byte of `0x80` is accepted directly:

- The decoded length is zero.
- The indefinite-length flag is set.
- The local cursor advances past that byte.

No terminator search occurs. The function does not require the constructed bit to be set, inspect content, or find an end-of-content marker.

#### Short definite length

A first length byte with its high bit clear directly encodes a length from 0 through 127. The local cursor advances past that byte.

No content byte must be readable for header decoding itself. A declared length greater than the remaining available count is handled by the later content-shortage check.

#### Long definite length

For a first length byte with its high bit set, other than `0x80`, the low seven bits specify the number of subsequent length bytes.

Only counts from one through four can reach acceptance. Counts of five through 127 produce a header failure.

Length bytes are accumulated most-significant first in eight-bit groups. Leading zero bytes are accepted. A value that could have used short form is not rejected for using long form.

The resulting length must be less than `0x80000000`. Thus, a four-byte value of `0x7FFFFFFF` can pass this magnitude check, while `0x80000000` cannot.

**Long-form boundary condition:** the available count must include at least one byte beyond all encoded length bytes. Having exactly enough bytes for the complete long-form header is not sufficient.

That extra byte is not read by this function. Its presence in the supplied count permits the length loop to recognize completion. This condition applies even when the decoded length is zero.

### 5. Length publication and content-shortage check

After accepting the length encoding, the function writes the length output.

It then computes the remaining content capacity using:

- The original available count.
- The local cursor after the header.
- The cursor address currently stored in the caller’s cursor cell.

For distinct, stable argument cells, this is the available count minus the number of header bytes consumed.

The comparison is signed. Accepted decoded lengths are below `0x80000000`, so they are nonnegative when interpreted by that comparison.

If the remaining capacity is less than the decoded length:

1. The diagnostic dependency is called with the content-shortage argument tuple.
2. The error flag is added.
3. Execution still proceeds to publish the post-header cursor.

This path is not a header-parsing failure. The function returns the constructed and indefinite flags applicable to the parsed header, together with the error flag.

If capacity is sufficient, no diagnostic is issued by this check.

For ordinary nonnegative input with distinct cells, indefinite length has decoded length zero and therefore does not itself trigger a content-shortage diagnostic.

### 6. Cursor publication and return

On an accepted header, including content shortage, the function writes the local post-header address to the cursor cell and returns the flag word.

It never advances that cursor over the decoded content length.

On a header-parsing failure, it calls the diagnostic dependency and returns exactly `0x80`. There is no explicit cursor-cell assignment on that path.

## Side effects

### Explicit write order

For accepted headers, the explicit write order is:

1. Tag output.
2. Class output.
3. Length output.
4. Optional diagnostic call for content shortage.
5. Cursor cell.

For a length-parsing failure after tag decoding:

1. Tag output.
2. Class output.
3. Diagnostic call.

The length output and cursor cell are not explicitly assigned on that failure path.

Earlier failures can reach the diagnostic without explicitly assigning any output.

These are not transactional updates. A caller must not assume that an error return leaves every output unchanged.

### Aliasing and intervening mutation

The cursor’s initial value is read to establish the local next-byte address and again to obtain the first identifier byte. Later, the cursor cell is read for the remaining-capacity calculation; that calculation does not necessarily use an immutable entry-time copy.

Consequences include:

- A tag, class, or length output that overlaps the cursor cell can change the base address used in the capacity calculation.
- An output that overlaps unread input bytes can change subsequent decoding.
- Overlapping output cells receive assignments in the order above.
- The diagnostic dependency can affect caller-visible state before the function finishes.

After a content-shortage diagnostic returns, the function still explicitly publishes its saved post-header cursor. It does not rewrite the tag, class, or length outputs after that call.

No broader guarantees about dependency-induced memory changes, global-state preservation, or ownership are established.

## Dependencies

### F-4fc89ec41a6d

The function invokes this dependency with one of two five-value tuples:

| Condition | Arguments, in order |
|---|---|
| Header-parsing failure | `0x0D`, `0x72`, `0x7B`, `0`, `0` |
| Accepted header with insufficient content capacity | `0x0D`, `0x72`, `0x9B`, `0`, `0` |

Its return value is not used.

The call sites support interpreting it as a diagnostic or error-reporting dependency. Its actual reporting mechanism, global-state effects, memory effects, ownership behavior, and whether it always returns are unknown.

Descriptions of actions after these calls are conditional on the dependency returning normally.

## Acceptance examples

All examples below are static acceptance criteria, not executed tests. Unless specified otherwise, they assume distinct writable argument cells, stable input, an available count equal to the displayed byte count, representable address arithmetic, and a normally returning dependency. Bytes are hexadecimal.

| Input | Expected observable behavior |
|---|---|
| Empty input, count zero | Header-failure diagnostic; return `0x80`; no explicit output assignments. |
| `04` | Missing length byte; header-failure diagnostic; no explicit tag or class assignment. |
| `04 00` | Tag 4, class `0x00`, length 0; cursor advances two bytes; return `0x00`. |
| `A3 00` | Tag 3, class `0x80`, length 0; cursor advances two bytes; return `0x20`. |
| `04 03 AA BB CC` | Length 3; cursor advances only two bytes; return `0x00`. Content remains unconsumed. |
| `04 03 AA` | Tag, class, and length are written; content-shortage diagnostic; cursor advances two bytes; return `0x80`. |
| `24 03 AA` | Same shortage, with constructed bit retained; return `0xA0`. |
| `04 80` | Indefinite length accepted despite primitive identifier; length 0; cursor advances two bytes; return `0x01`. |
| `24 80` | Indefinite constructed header; return `0x21`; no terminator scan. |
| `1F 01 00` | Extended form for tag 1 accepted; length 0; cursor advances three bytes. |
| `1F 80 01 00` | Redundant leading zero tag group accepted; tag 1; cursor advances four bytes. |
| `1F 81` | Unterminated extended tag; header-failure diagnostic; no explicit tag, class, length, or cursor assignment. |
| `04 81 01` | Long length ends exactly at the available boundary; header failure, not the content-shortage path. Tag and class have already been written. |
| `04 81 01 AA` | Long-form length 1 accepted; cursor advances three bytes; return `0x00`. |
| `04 82 00 00` | Header failure despite a complete zero-valued long length, because no extra byte remains in the count. |
| `04 82 00 00 AA` | Nonminimal long-form zero accepted; cursor advances four bytes and leaves `AA` unconsumed. |
| `04 85` | Unsupported five-byte length count; header-failure diagnostic after tag and class writes. |
| `04 84 80 00 00 00 AA` | Decoded length reaches `0x80000000`; header failure after tag and class writes. |

An additional tag-boundary criterion is that continuation groups producing accumulated value `0x01000000` must fail rather than proceed to another tag byte. Groups producing `0x00FFFFFF` may continue, subject to available bytes.

## Uncertainties

### Established static behavior

The byte layouts, output order, flag values, two diagnostic call patterns, content-shortage continuation, and long-form extra-byte condition are supported by the supplied function representation.

### Inferred intent

The identifier class, constructed flag, extended tag, and length formats strongly suggest a BER-family header parser. That interpretation does not establish compliance with BER, DER, or any larger protocol.

The content-shortage path appears intended to return a usable decoded header while flagging incomplete content. Only its concrete writes and return flags are established.

### documented_only

- The exact callable ABI, alignment requirements, and externally declared types are not independently available.
- Behavior involving signed overflow, address wraparound, negative counts, or unrepresentable pointer differences is not established as a supported contract.
- Invalid memory access behavior is not established.
- Concurrent mutation and arbitrary storage aliasing are not guaranteed safe.
- The diagnostic dependency’s implementation, global effects, ownership effects, and nonreturning behavior are unknown.
- No surrounding caller contract establishes how callers distinguish the two error classes beyond flags, diagnostics, and output state.
- No runtime tests have been performed.

## Evidence

This contract is based solely on static analysis of the supplied representation of F-fa1c60bee3f0 and the supplied opaque dependency mapping.

The supplied representation establishes:

- One-byte identifier decomposition and seven-bit extended-tag accumulation.
- Early tag and class writes before length validation.
- Separate indefinite, short definite, and long definite length paths.
- A four-byte maximum for long-form lengths and rejection at `0x80000000`.
- The long-form requirement for an additional counted byte beyond the length field.
- Distinct diagnostic arguments for malformed headers and insufficient content.
- Cursor publication after accepted headers, including content shortage.
- No content traversal or explicit allocation or release.

No independent source declaration, dependency implementation, protocol document, or runtime observation was available.
