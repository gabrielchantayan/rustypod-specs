<!-- Trusted provenance: F-bec4078f6fcf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bec4078f6fcf

## Purpose

F-bec4078f6fcf resolves two numeric selectors through a context-referenced mapping table, selects the first range descriptor containing the resulting key, and searches a dependency-provided buffer for that key. An exact match produces an unsigned payload plus a signed descriptor adjustment.

The context establishes a pointer relationship to the mapping table, not ownership. Ownership of the table, descriptors, handle-associated storage, and buffer remains unspecified.

Zero return does not distinguish a match from selector rejection, descriptor exhaustion, or a buffer-search miss. Nonzero results from either of two preparation dependencies are propagated.

A compact indexed lookup is inferred intent. Conventional binary-search behavior, valid record alignment, buffer ownership, and cleanup semantics are not established.

## Interface

### Arguments and result

The interface has four arguments:

1. **Context address:** a 32-bit address identifying the fields described below.
2. **First selector:** an unsigned 32-bit integer.
3. **Second selector:** an unsigned 32-bit integer.
4. **Output address:** writable storage for two consecutive 32-bit words.

The result is a 32-bit integer:

- Zero on selector rejection, exhaustion of the descriptor list, or completion of the buffer-search path.
- The exact nonzero result of F-e40a3ceddab2 or F-d036abe1e536 when that dependency returns nonzero.

There is no independent match indicator. The output address is initialized before selector validation, so it must be writable even for rejected selectors.

### Context fields

Offsets are byte offsets from the context address. These are accessed fields, not a complete object definition.

| Offset | Representation | Observed role |
|---|---|---|
| `0x68` | 32-bit address or handle | Captured for dependency calls and a later buffer-address read |
| `0x120` | Embedded storage address | The address of this location is passed to F-e40a3ceddab2 |
| `0x198` | Unsigned 32-bit word | Inclusive upper bound for normalized selectors |
| `0x1A0` | 32-bit address | Mapping-table base |
| `0x1A8` | 32-bit address | Descriptor-list head |

A 32-bit buffer address is later read at offset `0x20` from the captured handle.

### Selector records and arithmetic

Each selector is normalized independently: zero remains zero; any nonzero selector is reduced by one. Inputs zero and one therefore both normalize to zero.

Each normalized index is accepted when it is no greater than the unsigned context bound. Equality passes; this function does not treat the bound as an exclusive count.

The displacement of each selected mapping record is:

\[
\text{displacement}=(\text{normalized index}\times16)\bmod2^{32}.
\]

The multiplication wraps to 32 bits **before** the displacement is used to address the table. Only the first 32-bit word at the resulting record address is read.

Consequently, “16-byte record spacing” does not imply an unbounded mathematical array offset. A normalized index of `0x10000000`, if accepted by the bound, produces displacement zero. The selector-bound comparison does not prevent this multiplication overflow or establish that the addressed record is valid.

### Descriptor fields

| Offset | Representation | Observed role |
|---|---|---|
| `0x00` | 32-bit address | Next descriptor; zero terminates traversal |
| `0x04` | Unsigned byte | Quantity used in preparation and search arithmetic |
| `0x05` | Unsigned byte | Key-width and payload-width flags |
| `0x06` | Signed 16-bit integer | Adjustment added to a matched payload |
| `0x08` | Unsigned 32-bit word | Preparation multiplier and search stride |
| `0x0C` | 32-bit word | Argument to F-e40a3ceddab2 |
| `0x10` | Unsigned 32-bit word | Inclusive lower key bound |
| `0x14` | Unsigned 32-bit word | Inclusive upper key bound |

“Quantity” and “stride” describe use within this function, not established producer-side meanings.

## Behavior

### Initialization and rejection

The function performs these actions in order:

1. Writes zero to the first output word.
2. Normalizes the first selector.
3. Writes zero to the second output word.
4. Normalizes the second selector.
5. Checks the first normalized index against the context bound.
6. Checks the second normalized index only if the first comparison passes.

Either failed comparison returns zero before mapping-table reads, descriptor traversal, or dependency calls. The supplied representation accesses the bound in each reached comparison; preservation between those accesses is not independently established.

There is no null-output check. If output storage overlaps context fields, initialization can change the metadata subsequently read. No overlap protection is visible.

### Mapping and key construction

For accepted selectors, the function obtains the descriptor-list head, reads the second selector’s mapping word, captures the handle, and reads the first selector’s mapping word.

Each mapping read uses its own normalized index and wrapping 32-bit displacement calculation. The table-base field is referenced for each selected word.

The packed unsigned 32-bit key contains:

- The first mapping word’s low 16 bits in key bits 31–16.
- The second mapping word’s low 16 bits in key bits 15–0.

The second word is explicitly masked to 16 bits. The first word’s upper half is discarded by the 32-bit left shift.

Mapping reads and handle capture occur before the traversal checks whether the captured list head is zero. An empty descriptor list therefore does not eliminate those earlier storage requirements.

### Descriptor selection

Descriptors are visited in linked-list order. A descriptor qualifies when its unsigned lower bound is no greater than the key and its unsigned upper bound is no less than the key.

The first qualifying descriptor is selected. Later descriptors are not examined, even if their ranges overlap the selected range.

A zero head or a zero next address reached during traversal produces a zero return without dependency calls. No cycle detection is visible. A cyclic list without a qualifying descriptor may not terminate. A descriptor whose lower bound exceeds its upper bound cannot satisfy both comparisons.

### Preparation dependencies

For the selected descriptor, F-e40a3ceddab2 receives, in argument order:

1. The captured handle.
2. The descriptor word at `0x0C`.
3. The retained low 16 bits of the second mapping word, zero-extended to 32 bits.
4. The address of the context’s embedded storage at `0x120`.
5. The original output address.

A nonzero result is returned immediately. Output is not restored, and none of the remaining dependencies is called.

After a zero result, the descriptor quantity byte and stride word are read. Their product, reduced modulo \(2^{32}\), is passed to F-d036abe1e536 together with the captured handle. A nonzero result is returned immediately, without calling either remaining dependency.

Following a zero result from F-d036abe1e536, the quantity and stride are read again and retained for search setup. F-22fec3f2a1c8 receives the retained quantity, zero-extended to 32 bits.

The flags byte is read after F-22fec3f2a1c8 returns. The buffer address is also fetched afterward, from offset `0x20` of the captured handle.

These read boundaries are significant:

- Preparation size and search setup can use different quantity or stride values if a dependency changes the descriptor.
- Changes by F-22fec3f2a1c8 do not replace the quantity and stride already retained.
- Flags and the buffer address can reflect changes made by that dependency.
- The selected descriptor address, packed key, mapped second component, and captured handle are not recomputed after dependency calls.

No ordering beyond the supplied expression structure is established for the individual operand loads within a compound expression.

### Search setup and probe sequence

Let:

- \(q\) be the retained unsigned quantity byte.
- \(s\) be the retained unsigned 32-bit stride.
- \(h\) be the 32-bit result of F-22fec3f2a1c8, used by its bit pattern in the arithmetic.
- \(L=(s h)\bmod2^{32}\).
- \(d=(q-h)\bmod2^{32}\).

The search begins with the fetched buffer address as its current base and \(L\) as its unsigned byte span. The test on \(d\) distinguishes zero from nonzero; it does not test whether the subtraction is positive.

If \(d\) is nonzero, an initial probe is made at displacement \((s d)\bmod2^{32}\) from the base:

- Equality proceeds directly to payload extraction.
- A probe key below the target changes the current base to the address immediately after that probe’s key bytes.
- A probe key above the target leaves the base unchanged.

The smaller-key update skips the key bytes only. It does not explicitly skip the payload or advance by a whole stride.

If the initial probe does not match, or if \(d\) is zero, the remaining search compares the stride with the current span. While the unsigned stride is smaller than the span:

- The span is shifted right by one bit.
- A probe is made at that byte displacement from the current base.
- Equality proceeds to payload extraction.
- A smaller probe key moves the base to the probe’s key address.
- A larger probe key leaves the base unchanged.

After this loop, one final comparison is made at the current base.

This is not a contract for conventional element-index binary search. The span is halved in bytes, alignment is not corrected, and the initial smaller-key base update differs from later updates.

Zero quantity, zero stride, or zero span does not by itself suppress all buffer reads. With zero stride, both products used for displacements are zero, the loop is skipped, and a base-key comparison remains reachable; an initial base probe can occur as well.

### Key decoding

Only flag bits zero and one affect decoding.

When flag bit zero is clear, the key occupies two bytes:

- The first byte supplies bits 23–16.
- The second byte supplies bits 7–0.
- All other bits are zero.

Thus bytes `12 34` represent `0x00120034`, not `0x00001234`. The payload begins two bytes after the key address.

When flag bit zero is set, the key occupies four bytes, interpreted most significant byte first. The payload begins four bytes after the key address.

All key comparisons use unsigned 32-bit values. The byte assembly establishes numeric significance, but does not establish a chronological order for individual byte loads inside one expression.

### Match result and completion

On a match:

- If flag bit one is clear, one unsigned payload byte is read.
- If flag bit one is set, two payload bytes form an unsigned 16-bit value, most significant byte first.
- The descriptor adjustment is read as a signed 16-bit integer.
- Payload plus adjustment is written to the first output word.

The mathematical result lies between −32,768 and 98,302 inclusive, or between −32,768 and 33,022 for a one-byte payload. It fits a signed 32-bit integer; a negative result is stored in the corresponding 32-bit representation.

A failed key comparison does not itself cause payload extraction or a result write.

After either a match or completed search miss, F-9787e15c337d is called with the captured handle. Any return value is ignored, and this function returns zero.

A match-result write precedes that final call. Neither output word is reasserted afterward.

## Side effects

The visible body explicitly writes:

- Zero to output word zero at entry.
- Zero to output word one during initialization.
- The adjusted payload to output word zero on a match.

It does not explicitly write output word one again. Nevertheless, its final value is not guaranteed to be zero: F-e40a3ceddab2 receives the output address, and dependency effects are unknown. A miss likewise does not guarantee that output word zero remains zero.

Dependencies may affect reachable context, descriptor, handle, buffer, output, or global state. No preservation guarantee is available. If a dependency invalidates a retained descriptor or handle, later accesses have no local lifetime protection.

The visible body contains no explicit allocation, deallocation, or ownership transfer. Passing an address does not establish transfer or retention rights. F-9787e15c337d may have cleanup-related intent, but neither release behavior nor ownership is established. Its omission from early error paths is not sufficient evidence of either a leak or correct cleanup.

Aliasing can affect behavior beyond initialization. For example, a match-result write overlapping handle-associated storage occurs before the final dependency call. No alias validation or concurrency protection is visible.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-e40a3ceddab2 | Receives five arguments described above; nonzero result is propagated | Preparation meaning, output effects, ownership, retention, global changes, error meanings |
| F-d036abe1e536 | Receives captured handle and wrapping 32-bit quantity-times-stride product; nonzero result is propagated | Storage acquisition, validation, mutation, ownership, error meanings |
| F-22fec3f2a1c8 | Receives zero-extended quantity; its 32-bit result determines search arithmetic | Mathematical meaning, result bounds, side effects |
| F-9787e15c337d | Receives captured handle once after completed search, including a miss; result ignored | Cleanup, release, mutation, and failure behavior |

The search structure suggests a partitioning role for F-22fec3f2a1c8. No power-of-two property, nonnegative interpretation, or bound relative to the quantity is established.

## Acceptance examples

These are conditional specification cases, not executed tests. Referenced storage must be valid, and dependencies must return normally with the stated effects.

1. **Selector aliases.** With stable metadata, selectors zero and one both select displacement zero. Selector two selects displacement 16.

2. **Inclusive bound.** With bound three, selector four normalizes to three and passes. Selector five normalizes to four and fails. Rejection occurs after both output initialization writes and before any dependency call.

3. **Wrapping record displacement.** With bound at least `0x10000000`, selector `0x10000001` normalizes to `0x10000000` and passes. Its record displacement is zero, not `0x100000000`. With the same table base, it addresses the same mapping word as normalized index zero. This applies independently to either selector.

4. **Mapping truncation.** First and second mapping words `0xABCD1234` and `0xEF005678` produce key `0x12345678`.

5. **Empty descriptor list.** With valid context and mapping storage but a zero list head, mapping reads and handle capture still occur. The function then returns zero without dependency calls.

6. **Overlapping ranges.** If two consecutive descriptors both contain the key, the first is selected. Equality with either range endpoint qualifies.

7. **First preparation failure.** If F-e40a3ceddab2 returns seven, seven is returned. No later dependency is called, and that dependency’s output changes are not restored.

8. **Second preparation failure.** If F-e40a3ceddab2 succeeds and F-d036abe1e536 returns negative four, negative four is returned. Neither F-22fec3f2a1c8 nor F-9787e15c337d is called.

9. **Sparse two-byte key.** With flag bit zero clear, reached key bytes `12 34` match `0x00120034`, not `0x00001234`.

10. **Four-byte match and adjustment.** With both low flag bits set, reached key bytes `12 34 56 78` match `0x12345678`. Payload bytes `01 02` yield 258. Adjustment −3 causes 255 to be written before F-9787e15c337d is called.

11. **Miss after preparation.** If every performed comparison misses, F-9787e15c337d is still called and the result is zero. Both output words remain zero only if dependencies and other permitted overlapping effects leave them unchanged.

12. **Zero stride.** With successful preparation and valid buffer storage, zero stride does not mean an empty search. The span-halving loop is skipped, but a key comparison at the buffer base is reached.

## Uncertainties

The following remain **documented_only**:

- Full object definitions, calling convention, native byte order of word and signed-halfword loads, alignment requirements, and valid address ranges.
- Ownership and lifetimes of the mapping table, descriptors, captured handle, buffer, and embedded storage.
- Valid selector bounds and whether producers intentionally permit wrapping record displacements.
- Table extent, descriptor-list acyclicity, buffer extent, and protection against invalid probes.
- Producer guarantees concerning sorted keys, stride meaning, record boundaries, or safe arithmetic.
- Bounds and mathematical meaning of the result from F-22fec3f2a1c8.
- Dependency mutation, retention, allocation, invalidation, and release behavior.
- Application-level error meanings and output validity following dependency failure.
- Concurrency guarantees and permitted overlap among output, context, descriptors, and buffer.

The visible body does not independently check probe bounds or detect descriptor cycles. It has no general null checks for context, output, mapping storage, handle, or fetched buffer. Safe execution requires external address-validity and lifetime guarantees; successful selector comparisons alone do not provide them.

## Evidence

Established behavior is derived from static inspection of the supplied representation: access widths and offsets, output-write order, selector normalization, unsigned comparisons, wrapping 32-bit selector-record multiplication, packed-key construction, descriptor traversal, dependency arguments, search transitions, decoding, and return propagation.

“Mapping,” “descriptor,” “preparation,” and “stride” are descriptive labels, not evidence of a broader format or subsystem contract. Compact lookup and partitioning are inferred intent only.

No runtime tests have been performed. Dynamic memory safety, dependency behavior, ownership, and correctness against an external data format remain unverified.
